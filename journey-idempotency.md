# Journey 工作流的幂等、重试与外部 Signal 协同分析

## 1. 概述

本文档深入分析 Dittofeed 中 Journey（用户旅程）工作流的三个核心机制及其协同关系，并严格对照源代码进行精确描述：

- **幂等性 (Idempotency)**: 确保相同事件处理多次时产生相同结果
- **重试策略 (Retry)**: 处理异步活动失败时的恢复机制
- **外部 Signal 交互**: 外部取消、跳过、重发等信号与正在运行的工作流步骤的互动方式

---

## 2. 幂等键选择策略

### 2.1 工作流 ID 生成规则

Dittofeed 使用两种工作流 ID 生成策略，分别对应不同的场景：

#### 2.1.1 普通工作流 ID

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:146-154
export function getUserJourneyWorkflowId({
  userId,
  journeyId,
}: {
  userId: string;
  journeyId: string;
}): string {
  return `user-journey-${userId}-${journeyId}`;
}
```

**适用场景**: 基于 Segment 进入的旅程（SegmentEntryNode）

**幂等键组成**: `userId` + `journeyId`

#### 2.1.2 键控工作流 ID (Keyed Workflow)

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:86-144
export function getKeyedUserJourneyWorkflowIdInner({
  workspaceId,
  userId,
  journeyId,
  eventKey,
  eventKeyValue,
}: {
  workspaceId: string;
  userId: string;
  journeyId: string;
  eventKey: string;
  eventKeyValue: string;
}): string | null {
  const combined = uuidV5(
    [userId, eventKey, eventKeyValue].join("-"),
    workspaceId,
  );
  return `user-journey-keyed-${workspaceId}-${journeyId}-${combined}`;
}
```

**适用场景**: 基于事件进入的旅程（EventEntryNode）

**幂等键组成**: `workspaceId` + `userId` + `journeyId` + `eventKey` + `eventKeyValue`（通过 UUID v5 哈希）

### 2.2 幂等键选择逻辑

#### 2.2.1 事件键值提取

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:106-144
export function getKeyedUserJourneyWorkflowId({
  workspaceId,
  userId,
  journeyId,
  entryNode,
  event,
}: {...}): string | null {
  let key: string;
  let keyValue: string;
  if (entryNode.key) {
    key = entryNode.key;
    const keyValueResult = jsonStringOrNumber({
      data: event.properties,
      path: key,
    }).map(String).unwrapOr(null);
    if (!keyValueResult) {
      return null;
    }
    keyValue = keyValueResult;
  } else {
    key = "messageId";
    keyValue = event.messageId;
  }
  return getKeyedUserJourneyWorkflowIdInner({
    workspaceId,
    userId,
    journeyId,
    eventKey: key,
    eventKeyValue: keyValue,
  });
}
```

**优先级规则**:
1. **用户定义键 (entryNode.key)**: 如果 EventEntryNode 配置了自定义 key，从事件属性中提取
2. **默认键 (messageId)**: 如果没有配置自定义 key，使用事件的 `messageId`

#### 2.2.2 幂等保证机制

**机制一: WorkflowExecutionAlreadyStartedError 捕获**

```typescript
// packages/backend-lib/src/journeys/userWorkflow/lifecycle.ts:96-108
try {
  await workflowClient.signalWithStart<...>(...);
} catch (e) {
  if (e instanceof WorkflowExecutionAlreadyStartedError) {
    logger().info("User journey already started.", {
      workflowId,
      journeyId,
      userId,
      workspaceId,
      eventKey: definition.entryNode.key,
    });
    return;
  }
  throw e;
}
```

使用 Temporal 的 `signalWithStart` 原子操作：
- 第一次调用：创建工作流并发送信号
- 后续调用：捕获 `WorkflowExecutionAlreadyStartedError`，静默忽略

**机制二: isRunnable 数据库检查（精确逻辑）**

```typescript
// packages/backend-lib/src/journeys/userWorkflow/activities.ts:396-470
export async function isRunnable({
  workspaceId,
  journeyId,
  userId,
  eventKey,
  eventKeyName,
}: {...}): Promise<boolean> {
  const [previousExitEvent, journey, workspace] = await Promise.all([
    db().query.userJourneyEvent.findFirst({
      where: and(
        eq(dbUserJourneyEvent.journeyId, journeyId),
        eq(dbUserJourneyEvent.userId, userId),
        eventKey ? eq(dbUserJourneyEvent.eventKey, eventKey) : undefined,
        eventKeyName
          ? eq(dbUserJourneyEvent.eventKeyName, eventKeyName)
          : undefined,
        inArray(dbUserJourneyEvent.type, Array.from(ENTRY_TYPES)),
      ),
    }),
    // 注意：isRunnable 查询 journey 时没有 workspaceId 条件
    db().query.journey.findFirst({ where: eq(dbJourney.id, journeyId) }),
    workspaceId
      ? db().query.workspace.findFirst({ where: eq(dbWorkspace.id, workspaceId) })
      : null,
  ]);

  // 关键：没有历史记录时，直接返回 true，不检查 journey 是否存在
  if (!previousExitEvent) {
    return true;
  }

  // 有历史记录时，检查 canRunMultiple
  // 注意：journey 不存在时 journey?.canRunMultiple 是 undefined，!!undefined = false
  const canRunMultiple = !!journey?.canRunMultiple;

  // workspace 检查（workspaceId 是可选参数，但不传时 workspace = null，
  // 导致 workspace?.status = undefined，undefined !== "Active" 为 true，返回 false）
  if (workspace?.status !== "Active") {
    return false;
  }
  return canRunMultiple;
}
```

**isRunnable 精确逻辑判定表**:

| previousExitEvent | workspaceId 传入 | workspace 存在 | workspace.status | journey 存在 | journey.canRunMultiple | 返回值 | 说明 |
|------------------|-----------------|---------------|----------------|-------------|----------------------|-------|------|
| null (首次进入) | 任意 | 任意 | 任意 | 任意 | 任意 | `true` | **首次进入，不检查任何条件** |
| 存在 | 否 | `null`（未查询） | N/A | 任意 | 任意 | `false` | **workspaceId 不传 → workspace = null → workspace?.status = undefined → undefined !== "Active" → 返回 false** |
| 存在 | 是 | 否 | N/A | 任意 | 任意 | `false` | workspace 不存在 → workspace = null → 返回 false |
| 存在 | 是 | 是 | 非 Active | 任意 | 任意 | `false` | 工作空间不活跃 |
| 存在 | 是 | 是 | Active | 是 | true | `true` | 允许重复运行 |
| 存在 | 是 | 是 | Active | 是 | false | `false` | 不允许重复运行 |
| 存在 | 是 | 是 | Active | 否 | N/A | `false` | 旅程不存在 → `canRunMultiple = false` |

**调用位置**:

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:332-348
if (
  !(await isRunnable({
    journeyId,
    userId,
    eventKey,
    eventKeyName,
    workspaceId,
  }))
) {
  logger.info("early exit unrunnable user journey", {...});
  return null;  // 工作流直接返回 null 退出
}
```

**机制三: 运行时 Signal 去重**

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:444-459
wf.setHandler(trackSignal, async (event) => {
  if (keyedEventIds.has(event.messageId)) {
    logger.info("ignoring duplicate keyed event", {...});
    return;
  }
  // ... 处理信号
  keyedEventIds.add(event.messageId);
});
```

工作流内部维护 `keyedEventIds` Set，防止相同 `messageId` 的 signal 被重复处理。

---

## 3. 重试策略详解

### 3.1 Temporal 活动重试配置

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:223-237
const {
  onNodeProcessedV2,
  isRunnable,
  findNextLocalizedTime,
  findNextLocalizedTimeV2,
  getEarliestComputePropertyPeriod,
  getUserPropertyDelay,
  getWorkspace,
  shouldReEnter,
} = proxyActivities<typeof activities>({
  startToCloseTimeout: "2 minutes",
  retry: {
    maximumAttempts: defaultUserJourneyMaxAttempts,
  },
});
```

### 3.2 不同活动类型的重试策略

#### 3.2.1 核心旅程活动

| 活动名称 | 超时设置 | 重试次数 | 说明 |
|---------|---------|---------|------|
| `isRunnable`, `shouldReEnter`, `onNodeProcessedV2` | 2分钟 | `defaultUserJourneyMaxAttempts` | 核心流程控制 |
| `getWorkspace` | 2分钟 | `defaultUserJourneyMaxAttempts` | 工作空间检查 |

#### 3.2.2 计算属性等待活动

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:239-245
const { waitForComputeProperties } = proxyActivities<typeof activities>({
  startToCloseTimeout: "20 minutes",
  heartbeatTimeout: "30 seconds",
  retry: {
    maximumAttempts: waitForComputePropertiesMaxAttempts,
  },
});
```

#### 3.2.3 消息发送活动（重点分析）

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:955-970
const { sendMessageV2 } = proxyActivities<typeof activities>({
  startToCloseTimeout: "2 minutes",
  retry: {
    maximumAttempts: currentNode.retryCount ?? 3,
  },
});
const messageSucceeded = await sendMessageV2(sendMesssageParams);

if (!messageSucceeded && !currentNode.skipOnFailure) {
  logger.info("message node early exit", {...});
  nextNode = definition.exitNode;
  break;
}
```

**消息发送的三层结构（精确调用链）**:

```
工作流调用 sendMessageV2(params)
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│ sendMessageWithSender (sendMessageFactory 返回)                      │
│                                                                     │
│  第一步：旅程状态预检查                                              │
│  const journey = await db().query.journey.findFirst({               │
│    where: and(                                                      │
│      eq(dbJourney.id, params.journeyId),                            │
│      eq(dbJourney.workspaceId, params.workspaceId),                 │
│    ),                                                               │
│  });                                                                │
│  if (!journey || journey.status !== "Running") {                    │
│    return false;  ← 直接返回 false，不抛出异常（不触发重试）         │
│  }                                                                  │
│                                                                     │
│  第二步：调用 sendMessageInner                                      │
│  const sendResult = await sendMessageInner({                        │
│    ...params,                                                       │
│    sender,                                                          │
│  });                                                                │
│                                                                     │
│  第三步：Result → boolean 转换                                      │
│  let shouldContinue: boolean;                                       │
│  if (sendResult.isErr()) {                                          │
│    shouldContinue = false;  ← 所有错误都返回 false                  │
│    event = sendResult.error.type;  // JourneyEarlyExit /            │
│                                    // MessageSkipped /              │
│                                    // BadWorkspaceConfiguration     │
│  } else {                                                           │
│    shouldContinue = true;                                           │
│  }                                                                  │
│                                                                     │
│  第四步：提交事件追踪                                                │
│  await submitTrack({...});                                          │
│                                                                     │
│  return shouldContinue;  ← 返回给工作流的最终值                    │
└─────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│ sendMessageInner (内部实现)                                          │
│                                                                     │
│  第一步：加载依赖                                                    │
│  const [userPropertyAssignments, journey, subscriptionGroup] =      │
│    await Promise.all([...]);                                        │
│                                                                     │
│  第二步：旅程存在性检查                                              │
│  if (!journey) {                                                    │
│    return err({                                                     │
│      type: InternalEventType.BadWorkspaceConfiguration,             │
│      variant: { type: BadWorkspaceConfigurationType.JourneyNotFound }│
│    });                                                              │
│  }                                                                  │
│                                                                     │
│  第三步：旅程状态检查                                                │
│  if (!(journey.status === "Running" || journey.status === "Broadcast")) {│
│    return err({                                                     │
│      type: InternalEventType.JourneyEarlyExit,                      │
│      message: `Journey is not running: ${journey.status}`,          │
│    });                                                              │
│  }                                                                  │
│                                                                     │
│  第四步：实际发送 + 重试决策                                         │
│  try {                                                              │
│    const result = await sender({...});                              │
│    return result;  // 成功：返回 MessageSuccess                     │
│  } catch (senderError) {                                            │
│    const activityInfo = Context.current().info;                     │
│    const isLastAttempt = activityInfo.attempt >= (retryCount ?? 3); │
│                                                                     │
│    if (isLastAttempt) {                                             │
│      return err({                                                   │
│        type: InternalEventType.JourneyEarlyExit,                    │
│        message: `Message failed after maximum retry attempts`,      │
│      });                                                            │
│    }                                                                │
│                                                                     │
│    throw senderError;  // 非最后一次：抛出触发 Temporal 重试        │
│  }                                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 消息发送场景的精确判定表

**场景一: 旅程不存在或非 Running（sendMessageWithSender 预检查）**

```typescript
// packages/backend-lib/src/journeys/userWorkflow/activities.ts:307-323
const journey = await db().query.journey.findFirst({
  where: and(
    eq(dbJourney.id, params.journeyId),
    eq(dbJourney.workspaceId, params.workspaceId),
  ),
});
if (!journey || journey.status !== "Running") {
  return false;  // 直接返回 false
}
```

| 情况 | Temporal 重试 | sendMessageV2 返回值 | 工作流行为 |
|-----|--------------|---------------------|-----------|
| 旅程不存在 | ❌ 不触发（不抛异常） | `false` | 检查 `skipOnFailure` |
| 旅程状态非 Running | ❌ 不触发（不抛异常） | `false` | 检查 `skipOnFailure` |

**场景二: 实际发送失败（sender 抛出异常）**

| attempt | isLastAttempt | 行为 |
|--------|--------------|------|
| 1 | false | 抛出异常 → Temporal 重试 |
| 2 | false | 抛出异常 → Temporal 重试 |
| ... | false | 抛出异常 → Temporal 重试 |
| N (last) | true | 返回 `JourneyEarlyExit` → sendMessageV2 返回 `false` |

**场景三: 消息跳过（MessageSkipped）**

在 `sender` 内部（`packages/backend-lib/src/messaging.ts`）：

```typescript
// 订阅状态检查
if (subscriptionGroupDetails && !inSubscriptionGroup(subscriptionGroupDetails)) {
  return err({
    type: InternalEventType.MessageSkipped,
    variant: {
      type: MessageSkippedType.SubscriptionState,
      action: subscriptionGroupAction,
      subscriptionGroupType,
    },
  });
}

// 缺少标识符检查
const identifier = userPropertyAssignments[identifierKey];
if (!identifier || typeof identifier !== "string") {
  return err({
    type: InternalEventType.MessageSkipped,
    variant: {
      type: MessageSkippedType.MissingIdentifier,
      identifierKey,
    },
  });
}
```

**MessageSkipped 精确流程**:
1. `sender` 返回 `err({type: MessageSkipped})`（**不抛出异常**）
2. `sendMessageInner` 捕获这个 Result（isErr）
3. `sendMessageWithSender` 转换 `sendResult.isErr() → shouldContinue = false`
4. `sendMessageV2` 返回 `false`
5. 工作流检查 `skipOnFailure`

| 情况 | Temporal 重试 | sendMessageV2 返回值 | 工作流行为 |
|-----|--------------|---------------------|-----------|
| 用户取消订阅 | ❌ 不触发（不抛异常） | `false` | 检查 `skipOnFailure` |
| 缺少 email/phone | ❌ 不触发（不抛异常） | `false` | 检查 `skipOnFailure` |

### 3.4 配置默认值

```typescript
// packages/backend-lib/src/config.ts:782-803
waitForComputePropertiesMaxAttempts: parseMaxAttempts(
  rawConfig.waitForComputePropertiesMaxAttempts,
  nodeEnv === NodeEnvEnum.Test ? 1 : 5,
),
defaultUserJourneyMaxAttempts: rawConfig.defaultUserJourneyMaxAttempts !== undefined
  ? parseInt(rawConfig.defaultUserJourneyMaxAttempts)
  : (nodeEnv === NodeEnvEnum.Test ? 1 : undefined),  // 生产: undefined = Temporal 默认
defaultGetSegmentAndEventDetailsMaxAttempts: parseMaxAttempts(
  rawConfig.defaultGetSegmentAndEventDetailsMaxAttempts,
  nodeEnv === NodeEnvEnum.Test ? 1 : 10,
),
```

### 3.5 应用级重试 (非 Temporal 重试)

```typescript
// packages/backend-lib/src/journeys/userWorkflow/activities.ts:104-137
export async function getEventsByIdWithRetry(
  params: GetEventsByIdParams,
  metadata: { journeyId?: string; userId: string },
) {
  try {
    const events = await pRetry(() => getEventsById(params, metadata), {
      retries: 10,
      minTimeout: 50,
      maxTimeout: 1000,
      maxRetryTime: 60_000,
    });
    return events;
  } catch (e) {
    throw e;
  }
}
```

使用 `p-retry` 库实现：
- **最多 10 次重试**
- **指数退避**: 50ms ~ 1000ms
- **最大总等待时间**: 60秒
- **适用场景**: ClickHouse 事件查询，处理数据写入延迟

---

## 4. 取消、跳过、重发与工作流步骤的协同

### 4.1 取消机制 (Cancellation / Termination)

#### 4.1.1 工作流 Terminate 调用（硬取消）

```typescript
// packages/backend-lib/src/computedProperties/computePropertiesWorkflow/lifecycle.ts:124-142
export async function terminateComputePropertiesWorkflow({
  workspaceId,
}: {
  workspaceId: string;
}) {
  const client = await connectWorkflowClient();
  try {
    await client
      .getHandle(generateComputePropertiesId(workspaceId))
      .terminate();
  } catch (e) {
    logger().info({ err: e }, "Failed to terminate compute properties workflow.");
  }
}
```

**终止场景**:
- 工作空间禁用/删除
- 计算属性工作流需要重置
- 功能开关变更

**效果**:
- 立即终止工作流，不等待当前活动完成
- 不会触发 `onNodeProcessed` 事件追踪
- 工作流历史中记录 Terminated 状态

#### 4.1.2 工作空间状态检查（软取消）

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:200-204, 1100-1111
const LONG_RUNNING_NODE_TYPES = new Set<JourneyNodeType>([
  JourneyNodeType.WaitForNode,
  JourneyNodeType.DelayNode,
  JourneyNodeType.SegmentEntryNode,
]);

// check if workspace is inactive after a long running node
if (LONG_RUNNING_NODE_TYPES.has(currentNode.type)) {
  const workspace = await getWorkspace(workspaceId);
  if (workspace?.status !== "Active") {
    logger.info("workspace is not active, exiting journey", {...});
    break;  // 优雅退出 nodeLoop
  }
}
```

**软取消精确行为**:
- **检查时机**: 长时运行节点（DelayNode、WaitForNode、SegmentEntryNode）**完成后**
- **判断条件**: `workspace?.status !== "Active"`
  - `workspace` 为 `null`/`undefined` → 条件为 true → 退出
  - `workspace.status` 为非 "Active" → 条件为 true → 退出
- **行为**: `break` 退出 `nodeLoop`，允许当前节点完成，然后优雅退出

#### 4.1.3 消息发送时的旅程状态检查

**sendMessageWithSender 预检查**（Temporal 活动的外层）：

```typescript
// packages/backend-lib/src/journeys/userWorkflow/activities.ts:307-323
const journey = await db().query.journey.findFirst({
  where: and(
    eq(dbJourney.id, params.journeyId),
    eq(dbJourney.workspaceId, params.workspaceId),
  ),
});
if (!journey || journey.status !== "Running") {
  return false;  // 直接返回 false，不重试
}
```

**sendMessageInner 检查**（Temporal 活动的内层）：

```typescript
// packages/backend-lib/src/journeys/userWorkflow/activities.ts:228-242
if (!journey) {
  return err({
    type: InternalEventType.BadWorkspaceConfiguration,
    variant: {
      type: BadWorkspaceConfigurationType.JourneyNotFound,
    },
  });
}

if (!(journey.status === "Running" || journey.status === "Broadcast")) {
  return err({
    type: InternalEventType.JourneyEarlyExit,
    message: `Journey is not running: ${journey.status}`,
  });
}
```

**注意**: 两层检查的状态范围不同
- 外层: `status !== "Running"`（只允许 Running）
- 内层: `status !== "Running" && status !== "Broadcast"`（允许 Running 或 Broadcast）

#### 4.1.4 取消与 Signal 的交互

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    工作流等待状态                                        │
│                                                                         │
│  工作流正在执行: wf.condition(() => segmentAssignedTrue(segId))          │
│                                                                         │
│  同时可以接收 Signal:                                                    │
│  ├── segmentUpdateSignal → 更新 segmentAssignments                      │
│  ├── trackSignal → 追加事件并重新评估 segment                           │
│  └── reEvaluateSegmentsSignal → 强制重新查询数据库                      │
│                                                                         │
│  如果工作流被 terminate():                                              │
│  ├── 正在执行的活动被取消                                                │
│  ├── 工作流立即停止，不再处理任何节点                                    │
│  └── 不会触发后续节点的 onNodeProcessed                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 跳过机制 (Skip)

#### 4.2.1 工作流中的 skipOnFailure 决策

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:961-970
const messageSucceeded = await sendMessageV2(sendMesssageParams);

if (!messageSucceeded && !currentNode.skipOnFailure) {
  logger.info("message node early exit", {...});
  nextNode = definition.exitNode;
  break;
}

// 如果跳过或成功，继续下一个节点
nextNode = nodes.get(currentNode.child) ?? null;
```

**精确决策逻辑**:

```typescript
// 条件: !messageSucceeded && !currentNode.skipOnFailure
// 即: messageSucceeded === false 且 skipOnFailure === false
// 才会触发 early exit
```

| messageSucceeded | skipOnFailure | 行为 |
|-----------------|--------------|------|
| `true` | 任意 | `nextNode = child` → 继续 |
| `false` | `true` | `nextNode = child` → 继续（跳过失败） |
| `false` | `false` | `nextNode = exitNode`, `break` → 取消旅程 |

#### 4.2.2 消息跳过的类型

**类型定义**:

```typescript
// packages/isomorphic-lib/src/types.ts:4503-4506
export enum MessageSkippedType {
  SubscriptionState = "SubscriptionState",
  MissingIdentifier = "MissingIdentifier",
}
```

**SubscriptionState - 订阅状态跳过**:

```typescript
// packages/backend-lib/src/messaging.ts:422-435
if (subscriptionGroupDetails && !inSubscriptionGroup(subscriptionGroupDetails)) {
  return err({
    type: InternalEventType.MessageSkipped,
    variant: {
      type: MessageSkippedType.SubscriptionState,
      action: subscriptionGroupAction,  // Unsubscribe / OptOut
      subscriptionGroupType,
    },
  });
}
```

**MissingIdentifier - 缺少标识符跳过**:

```typescript
// packages/backend-lib/src/messaging.ts:960-968
const identifier = userPropertyAssignments[identifierKey];
if (!identifier || typeof identifier !== "string") {
  return err({
    type: InternalEventType.MessageSkipped,
    variant: {
      type: MessageSkippedType.MissingIdentifier,
      identifierKey,  // e.g., "email", "phone"
    },
  });
}
```

#### 4.2.3 跳过与重试的精确协同流程

```
消息发送调用链:
workflow → sendMessageV2 → sendMessageWithSender → sendMessageInner → sender
```

**完整流程图**:

```
工作流调用 sendMessageV2(params)
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. sendMessageWithSender 预检查                                 │
│    const journey = await db().query.journey.findFirst({...})    │
│    ├── journey 不存在  → return false                           │
│    └── journey.status !== "Running"  → return false            │
│    ↓ 检查通过                                                   │
│    const sendResult = await sendMessageInner(...)               │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. sendMessageInner 检查                                        │
│    ├── journey 不存在  → return err(BadWorkspaceConfiguration)  │
│    ├── journey.status 非 Running/Broadcast                      │
│    │                       → return err(JourneyEarlyExit)      │
│    └── 检查通过 → 调用 sender(...)                              │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. sender 实际发送                                               │
│    ├── 用户取消订阅                                              │
│    │   → return err(MessageSkipped.SubscriptionState)          │
│    ├── 缺少标识符 (email/phone)                                  │
│    │   → return err(MessageSkipped.MissingIdentifier)          │
│    ├── 发送成功                                                  │
│    │   → return ok(MessageSuccess)                              │
│    └── 发送失败 (抛出异常)                                       │
│        → catch 块检查 isLastAttempt                             │
│        ├── 否 → throw 触发 Temporal 重试                        │
│        └── 是 → return err(JourneyEarlyExit)                   │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. sendResult → shouldContinue 转换                             │
│    sendResult.isErr() → shouldContinue = false                  │
│    sendResult.isOk()  → shouldContinue = true                   │
│                                                                 │
│    提交事件追踪: submitTrack({event: ...})                      │
│                                                                 │
│    return shouldContinue  // 即 messageSucceeded               │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 工作流决策                                                   │
│    if (!messageSucceeded && !currentNode.skipOnFailure) {       │
│      nextNode = definition.exitNode                             │
│      break  // 取消旅程                                         │
│    }                                                            │
│    // 否则继续下一个节点                                         │
│    nextNode = nodes.get(currentNode.child) ?? null              │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 重发机制 (Retry / Resend)

#### 4.3.1 Temporal 活动级重试

**触发条件**: `sendMessageInner` 的 catch 块中，`isLastAttempt === false` 时抛出异常

```typescript
// packages/backend-lib/src/journeys/userWorkflow/activities.ts:270-300
catch (senderError) {
  const activityInfo = Context.current().info;
  const isLastAttempt = activityInfo.attempt >= (retryCount ?? 3);

  if (isLastAttempt) {
    return err({
      type: InternalEventType.JourneyEarlyExit,
      message: `Message failed after maximum retry attempts`,
    });
  }

  throw senderError;  // 触发 Temporal 重试
}
```

**Temporal 重试特性**:
- 指数退避 (Exponential Backoff)
- `maximumAttempts = currentNode.retryCount ?? 3`
- 只有实际发送异常才会触发重试

#### 4.3.2 键控事件的 Signal 追加

当同一业务键的事件再次到达时：

```typescript
// packages/backend-lib/src/journeys/keyedEventEntry.test.ts:1415-1440
// 测试场景：预约被取消，发送取消事件
const cancelledEvent = {
  type: EventType.Track,
  event: "APPOINTMENT_UPDATE",
  userId,
  messageId: randomUUID(),  // 新的 messageId
  properties: {
    operation: "CANCELLED",
    appointmentId: appointmentId1,  // 同一 appointmentId
  },
} as const;

await submitBatch({...});

// 向已运行的工作流发送 signal
await handle1.signal(trackSignal, {
  version: TrackSignalParamsVersion.V2,
  messageId: cancelledEvent.messageId,
});
```

**工作流内部处理**:

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:444-527
wf.setHandler(trackSignal, async (event) => {
  // 1. 去重检查（按 messageId）
  if (keyedEventIds.has(event.messageId)) {
    logger.info("ignoring duplicate keyed event", {...});
    return;
  }

  // 2. 根据版本处理事件
  switch (event.version) {
    case TrackSignalParamsVersion.V2: {
      const newEvents = await getEventsById({
        workspaceId,
        eventIds: [event.messageId],
      });
      keyedEvents?.push(...newEvents);
      keyedEventIds.add(event.messageId);
      break;
    }
    // ...
  }

  // 3. 如果在 WaitForNode，立即重新评估 segment
  if (waitForSegmentIds) {
    await Promise.all(
      waitForSegmentIds.map(async ({ segmentId }) => {
        const assignment = await getSegmentAssignmentHandler({...});
        if (assignment === null) return;
        segmentAssignments.set(segmentId, {
          currentlyInSegment: assignment.inSegment,
          segmentVersion: nowMs,
        });
      }),
    );
  }
});
```

**Signal 重发事件与等待节点的精确协同**:

```
工作流状态: 在 WaitForNode 等待
           waitForSegmentIds = [{segmentId: "cancelled", ...}]

时间线:
T0  工作流执行 WaitForNode:
    ├── 调用 getSegmentAssignmentHandler 初始评估
    ├── 初始评估不满足条件
    ├── waitForSegmentIds 被设置
    └── 进入 wf.condition(...) 等待

T1  收到 trackSignal (新事件):
    ├── Signal Handler 执行
    ├── 去重检查通过（新 messageId）
    ├── 事件追加到 keyedEvents
    ├── waitForSegmentIds 存在 → 重新评估所有 segment
    ├── getSegmentAssignmentHandler 被调用
    ├── 新事件使 segment 变为 true
    └── segmentAssignments.set("cancelled", {currentlyInSegment: true})

T2  wf.condition 检测到条件变化:
    ├── segmentAssignedTrue("cancelled") 返回 true
    ├── 工作流被唤醒
    └── 继续执行取消分支
```

### 4.4 交互矩阵（精确版）

| 机制 | 触发方式 | 与运行中步骤的交互 | Temporal 重试 | 工作流行为 |
|-----|---------|-----------------|--------------|-----------|
| **取消 (Terminate)** | `client.getHandle().terminate()` | 立即终止，不等待当前步骤 | - | 工作流完全停止 |
| **取消 (软取消)** | 长时运行节点后 `workspace.status` 检查 | 当前节点完成后检查 | - | 优雅退出 `nodeLoop` |
| **取消 (旅程不存在)** | `sendMessageWithSender` 预检查 | 活动开始时检查 | ❌ 不触发 | `sendMessageV2` 返回 `false` |
| **取消 (旅程非 Running)** | `sendMessageWithSender` 预检查 | 活动开始时检查 | ❌ 不触发 | `sendMessageV2` 返回 `false` |
| **取消 (内层状态检查)** | `sendMessageInner` 检查 | 活动内部检查 | ❌ 不触发 | `sendMessageV2` 返回 `false` |
| **跳过 (skipOnFailure)** | 工作流决策层 | `messageSucceeded === false` 时 | - | `skipOnFailure === true` → 继续 |
| **跳过 (订阅状态)** | `sender` 返回 `MessageSkipped` | 消息发送前检查 | ❌ 不触发 | `sendMessageV2` 返回 `false` |
| **跳过 (缺少标识符)** | `sender` 返回 `MessageSkipped` | 消息发送前检查 | ❌ 不触发 | `sendMessageV2` 返回 `false` |
| **重发 (Temporal 重试)** | `sender` 抛出异常 | `isLastAttempt === false` 时 | ✅ 自动重试 | 对工作流透明 |
| **重发 (Signal 追加)** | 新事件通过 `trackSignal` | Signal Handler 异步执行 | - | 等待节点可能被唤醒 |

---

## 5. 外部 Signal 与工作流步骤的互动

### 5.1 Signal 定义

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:51-77
export const segmentUpdateSignal =
  wf.defineSignal<[SegmentUpdate]>("segmentUpdate");

export const reEvaluateSegmentsSignal =
  wf.defineSignal<[ReEvaluateSegmentsParams]>("reEvaluateSegments");

export const trackSignal = wf.defineSignal<[TrackSignalParams]>("track");
```

### 5.2 三种 Signal 的处理

#### 5.2.1 trackSignal - 键控事件信号

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:444-527
wf.setHandler(trackSignal, async (event) => {
  // 去重检查
  if (keyedEventIds.has(event.messageId)) {
    return;
  }

  // 根据版本处理事件
  switch (event.version) {
    case TrackSignalParamsVersion.V2: {
      // 从数据库加载完整事件
      const newEvents = await getEventsById({...});
      keyedEvents?.push(...newEvents);
      keyedEventIds.add(event.messageId);
      break;
    }
    case TrackSignalParamsVersion.V1:
    default: {
      keyedEvents?.push(event);
      keyedEventIds.add(event.messageId);
      break;
    }
  }

  // 关键：如果在 WaitForNode，立即重新评估 segment
  if (waitForSegmentIds) {
    await Promise.all(
      waitForSegmentIds.map(async ({ segmentId }) => {
        const assignment = await getSegmentAssignmentHandler({...});
        if (assignment === null) return;
        segmentAssignments.set(segmentId, {...});
      }),
    );
  }
});
```

**与不同节点的交互**:

| 当前节点 | waitForSegmentIds | 收到 trackSignal 后的行为 |
|---------|-------------------|------------------------|
| **WaitForNode** | `!= null` | 追加事件 + 立即重新评估所有等待的 segment |
| **SegmentEntryNode** | `== null` | 追加事件，但不重新评估（SegmentEntryNode 有单独的等待逻辑） |
| **DelayNode** | `== null` | 追加事件，睡眠继续 |
| **MessageNode** | `== null` | 追加事件，消息发送可能使用新事件 |

#### 5.2.2 segmentUpdateSignal - Segment 变更通知

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:529-549
wf.setHandler(segmentUpdateSignal, (update) => {
  const prev = segmentAssignments.get(update.segmentId);

  // 版本检查，忽略过期更新
  if (prev && prev.segmentVersion >= update.segmentVersion) {
    logger.info("ignoring stale segment update", {...});
    return;
  }

  segmentAssignments.set(update.segmentId, {
    currentlyInSegment: update.currentlyInSegment,
    segmentVersion: update.segmentVersion,
  });
});
```

**特性**:
- 同步处理（非 async）
- 版本控制：`segmentVersion` 低的更新被忽略
- 直接更新 `segmentAssignments` Map

#### 5.2.3 reEvaluateSegmentsSignal - 强制重新评估

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:551-599
wf.setHandler(reEvaluateSegmentsSignal, async (params) => {
  const segmentIdsToEvaluate =
    params.segmentIds ?? Array.from(segmentAssignments.keys());
  const nowMs = Date.now();

  await Promise.all(
    segmentIdsToEvaluate.map(async (segmentId) => {
      const assignment = await getSegmentAssignmentHandler({...});
      if (assignment === null) return;
      segmentAssignments.set(segmentId, {
        currentlyInSegment: assignment.inSegment,
        segmentVersion: nowMs,
      });
    }),
  );
});
```

**触发场景**:
- 外部系统修改了用户属性
- 需要立即同步最新状态
- 不依赖 segment 变更事件

### 5.3 Signal 发送方式

#### 5.3.1 SignalWithStart（原子操作）

```typescript
// packages/backend-lib/src/journeys/userWorkflow/lifecycle.ts:70-95
await workflowClient.signalWithStart<
  typeof userJourneyWorkflow,
  [TrackSignalParams]
>(userJourneyWorkflow, {
  taskQueue: "default",
  workflowId,
  signal: trackSignal,
  signalArgs: [
    {
      version: TrackSignalParamsVersion.V2,
      messageId: event.messageId,
    },
  ],
  args: [
    {
      journeyId,
      definition,
      workspaceId,
      userId,
      eventKey,
      hidden: event.context?.hidden === true,
      messageId: event.messageId,
      version: UserJourneyWorkflowVersion.V3,
    },
  ],
});
```

**优点**:
- 原子性：创建工作流 + 发送信号在一个事务中
- 避免竞态条件
- 配合 `WorkflowExecutionAlreadyStartedError` 实现幂等

#### 5.3.2 单独发送 Signal

```typescript
// 向已存在的工作流发送取消事件
await handle1.signal(trackSignal, {
  version: TrackSignalParamsVersion.V2,
  messageId: cancelledEvent.messageId,
});
```

### 5.4 长时运行节点的 Signal 响应

| 节点类型 | 当前操作 | Signal 处理 | 对工作流的影响 |
|---------|---------|------------|--------------|
| **WaitForNode** | `wf.condition()` 等待 | Signal Handler 执行，更新 `segmentAssignments` | 条件满足时立即唤醒 |
| **SegmentEntryNode** | `wf.condition()` 等待 | `segmentUpdateSignal` 更新 Map，`trackSignal` 仅追加事件 | `segmentAssignedTrue` 检测到变化时唤醒 |
| **DelayNode** | `await sleep()` 睡眠 | Signal Handler 执行，状态更新 | 不打断睡眠，睡眠结束后状态生效 |
| **MessageNode** | `await sendMessageV2()` | Signal Handler 执行 | 消息发送可能使用新事件 |

---

## 6. shouldReEnter 与 ContinueAsNew 精确分析

### 6.1 shouldReEnter 严格条件

```typescript
// packages/backend-lib/src/journeys/userWorkflow/activities.ts:777-838
export async function shouldReEnter({
  journeyId,
  userId,
  workspaceId,
}: {...}): Promise<boolean> {
  // 条件 1: 旅程存在
  const journey = await db().query.journey.findFirst({
    where: and(
      eq(dbJourney.id, journeyId),
      eq(dbJourney.workspaceId, workspaceId),
    ),
  });
  if (!journey) {
    return false;
  }

  // 条件 2: canRunMultiple === true
  if (!journey.canRunMultiple) {
    return false;
  }

  // 条件 3: status === "Running"
  if (journey.status !== "Running") {
    return false;
  }

  // 条件 4: 定义有效
  const definitionResult = schemaValidateWithErr(
    journey.definition,
    JourneyDefinition,
  );
  if (definitionResult.isErr()) {
    return false;
  }
  const definition = definitionResult.value;

  // 条件 5: 入口节点类型是 SegmentEntryNode
  if (definition.entryNode.type !== JourneyNodeType.SegmentEntryNode) {
    return false;
  }

  // 条件 6: 用户仍在入口 segment 中
  const assignment = await getSegmentAssignmentDb({
    workspaceId,
    segmentId: definition.entryNode.segment,
    userId,
  });

  // 条件 7: 入口节点 reEnter === true
  return assignment === true && definition.entryNode.reEnter === true;
}
```

**shouldReEnter 返回 true 的所有必要条件**:

| 条件 | 检查内容 | 失败时返回 |
|-----|---------|-----------|
| 1 | 旅程存在（带 workspaceId 条件） | `false` |
| 2 | `journey.canRunMultiple === true` | `false` |
| 3 | `journey.status === "Running"` | `false` |
| 4 | journey.definition 有效 | `false` |
| 5 | 入口节点类型是 `SegmentEntryNode` | `false` |
| 6 | 用户在入口 segment 中 | `false` |
| 7 | 入口节点 `reEnter === true` | `false` |

**注意**: `shouldReEnter` 只允许 `SegmentEntryNode` 类型的旅程重复进入，**EventEntryNode 类型不支持**。

### 6.2 ContinueAsNew 调用时机

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:274, 1127-1133
let {
  // ...
  shouldContinueAsNew = true,  // 默认值是 true
} = props;

// ... nodeLoop 执行完成 ...

// 检查是否应该重入
if (await shouldReEnter({ journeyId, userId, workspaceId })) {
  if (shouldContinueAsNew) {
    await continueAsNew<typeof userJourneyWorkflow>(props);
  } else {
    return props;
  }
}
return null;
```

**流程**:
1. 工作流完成所有节点，到达 `ExitNode`
2. 调用 `shouldReEnter` 检查
3. 返回 `true` 且 `shouldContinueAsNew === true` 时：
   - 调用 `continueAsNew<typeof userJourneyWorkflow>(props)`
   - 创建新的工作流 Run，保持相同 Workflow ID
   - 重置工作流历史，避免无限增长

---

## 7. 协同流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          外部事件 / Signal 入口                               │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. 幂等性检查层                                                              │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 工作流 ID 生成                                                       │   │
│  │ - 普通: user-journey-{userId}-{journeyId}                            │   │
│  │ - 键控: user-journey-keyed-{...uuidV5(userId, eventKey, value)...}   │   │
│  │                                                                     │   │
│  │ signalWithStart + WorkflowExecutionAlreadyStartedError               │   │
│  │ - 第一次: 创建工作流 + 发送 signal                                    │   │
│  │ - 重复: 捕获异常，静默忽略                                            │   │
│  │                                                                     │   │
│  │ isRunnable 检查 (精确逻辑)                                           │   │
│  │ ┌─────────────────────────────────────────────────────────────┐     │   │
│  │ │ previousExitEvent === null (首次进入)                        │     │   │
│  │ │   → return true (不检查 journey/workspace)                  │     │   │
│  │ │                                                             │     │   │
│  │ │ previousExitEvent 存在                                       │     │   │
│  │ │   → journey 不存在 → canRunMultiple = false → return false  │     │   │
│  │ │   → journey.canRunMultiple = false → return false           │     │   │
│  │ │   → workspace 不存在/非 Active → return false               │     │   │
│  │ │   → 其他 → return true                                      │     │   │
│  │ └─────────────────────────────────────────────────────────────┘     │   │
│  │                                                                     │   │
│  │ 运行时 Signal 去重 (keyedEventIds Set, 按 messageId)                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ 检查通过
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. 工作流执行层                                                              │
│                                                                             │
│  nodeLoop 循环执行节点:                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │SegmentEntry │  │  DelayNode  │  │ WaitForNode │  │MessageNode  │       │
│  │    Node     │  │             │  │             │  │             │       │
│  │等待 segment │  │  sleep()    │  │多条件等待   │  │消息发送     │       │
│  │ wf.condition│  │             │  │+超时       │  │+重试        │       │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
│         │                │                │                │                │
│         ▼                ▼                ▼                ▼                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │              活动调用 (Activity Invocation)                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ Signal 随时可以到达
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. Signal 处理层                                                            │
│                                                                             │
│  ┌──────────────────┐ ┌──────────────────┐ ┌─────────────────────┐         │
│  │   trackSignal    │ │segmentUpdateSignal│ │ reEvaluateSegments  │         │
│  │ (async)          │ │ (sync)            │ │ (async)             │         │
│  │                  │ │                  │ │                     │         │
│  │ 事件追加         │ │ 版本控制更新      │ │ 强制重新查询数据库   │         │
│  │ keyedEvents     │ │ segmentVersion   │ │ getSegmentAssignment │         │
│  │ keyedEventIds   │ │                  │ │                     │         │
│  │                  │ │                  │ │                     │         │
│  │ WaitForNode 时   │ │ 直接更新 Map     │ │ 更新 Map            │         │
│  │ 重新评估 segment │ │                  │ │                     │         │
│  └────────┬─────────┘ └────────┬─────────┘ └──────────┬──────────┘         │
│           │                   │                      │                      │
│           ▼                   ▼                      ▼                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 更新 segmentAssignments / keyedEvents                               │   │
│  │                                                                     │   │
│  │ 等待节点 (WaitFor / SegmentEntry):                                  │   │
│  │ └─▶ wf.condition() 检测条件变化 → 唤醒工作流                         │   │
│  │                                                                     │   │
│  │ 非等待节点 (Delay / Message):                                       │   │
│  │ └─▶ 状态更新，但不打断当前执行                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ 活动调用
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. 消息发送活动层 (精确调用链)                                               │
│                                                                             │
│  workflow → sendMessageV2 → sendMessageWithSender → sendMessageInner → sender│
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 第一层: sendMessageWithSender 预检查                                 │   │
│  │ ├── journey 不存在 → return false (不重试)                          │   │
│  │ └── journey.status !== "Running" → return false (不重试)            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓ 检查通过                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 第二层: sendMessageInner 检查                                        │   │
│  │ ├── journey 不存在 → err(BadWorkspaceConfiguration)                │   │
│  │ └── journey.status 非 Running/Broadcast → err(JourneyEarlyExit)     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓ 检查通过                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 第三层: sender 实际发送                                              │   │
│  │ ├── 用户取消订阅 → err(MessageSkipped.SubscriptionState)            │   │
│  │ ├── 缺少标识符 → err(MessageSkipped.MissingIdentifier)              │   │
│  │ ├── 发送成功 → ok(MessageSuccess)                                   │   │
│  │ └── 发送失败 (抛异常)                                                │   │
│  │     ├── 非最后一次 → throw 触发 Temporal 重试                        │   │
│  │     └── 最后一次 → err(JourneyEarlyExit)                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓ Result 转换                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ sendResult.isErr() → shouldContinue = false                         │   │
│  │ submitTrack({event: ...})                                           │   │
│  │ return shouldContinue  // messageSucceeded                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ messageSucceeded 返回
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. 决策层 (取消 / 跳过)                                                      │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 工作流决策                                                           │   │
│  │ if (!messageSucceeded && !currentNode.skipOnFailure) {              │   │
│  │   nextNode = definition.exitNode                                    │   │
│  │   break  // 取消旅程                                                │   │
│  │ }                                                                    │   │
│  │ // 否则继续                                                          │   │
│  │ nextNode = nodes.get(currentNode.child) ?? null                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ messageSucceeded vs skipOnFailure 真值表                             │   │
│  │ ┌──────────────────┬──────────────────┬─────────────────────────┐    │   │
│  │ │ messageSucceeded │ skipOnFailure    │ 行为                    │    │   │
│  │ ├──────────────────┼──────────────────┼─────────────────────────┤    │   │
│  │ │ true             │ 任意             │ nextNode = child → 继续  │    │   │
│  │ │ false            │ true             │ nextNode = child → 继续  │    │   │
│  │ │ false            │ false            │ exitNode, break → 取消   │    │   │
│  │ └──────────────────┴──────────────────┴─────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ContinueAsNew 检查 (shouldReEnter)                                   │   │
│  │ 7 个必要条件:                                                        │   │
│  │ 1. 旅程存在 (带 workspaceId 条件)                                    │   │
│  │ 2. journey.canRunMultiple === true                                  │   │
│  │ 3. journey.status === "Running"                                     │   │
│  │ 4. definition 有效                                                  │   │
│  │ 5. entryNode.type === SegmentEntryNode                              │   │
│  │ 6. 用户在入口 segment 中                                             │   │
│  │ 7. entryNode.reEnter === true                                       │   │
│  │                                                                     │   │
│  │ 全部满足 + shouldContinueAsNew === true                             │   │
│  │   → continueAsNew<typeof userJourneyWorkflow>(props)                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. 典型场景分析

### 场景一: 首次进入旅程（isRunnable 精确行为）

1. **事件**: 用户首次触发 `order_created`，`orderId: "ORD001"`
2. **工作流启动**:
   - `signalWithStart` 创建工作流
3. **isRunnable 检查**:
   ```typescript
   // previousExitEvent = null (数据库中没有该用户+旅程+orderId的记录)
   if (!previousExitEvent) {
     return true;  // 直接返回 true
   }
   ```
4. **结果**: 工作流继续执行，不检查 journey/workspace

**关键点**: 首次进入时，即使旅程已被删除，`isRunnable` 也会返回 `true`。

### 场景二: 重复订单事件（幂等性）

1. **事件**: 相同 `orderId: "ORD001"` 的事件再次到达
2. **入口幂等检查**:
   - `signalWithStart` 捕获 `WorkflowExecutionAlreadyStartedError`，忽略
3. **如果工作流已在运行**:
   - 单独调用 `handle.signal(trackSignal, {...})`
   - Signal Handler 检查 `keyedEventIds.has(messageId)`
   - 相同 `messageId` → 忽略

### 场景三: 消息发送失败 → 重试 → 跳过/取消

**配置**: `retryCount: 3`, `skipOnFailure: true`

1. **第 1 次尝试**:
   - `sender` 抛出异常
   - `isLastAttempt = (1 >= 3) = false`
   - 抛出异常 → Temporal 重试

2. **第 2 次尝试**:
   - 同样抛出异常
   - `isLastAttempt = false` → Temporal 重试

3. **第 3 次尝试 (last attempt)**:
   - 抛出异常 → catch 块捕获
   - `isLastAttempt = true`
   - 返回 `err({type: JourneyEarlyExit})`

4. **Result 转换**:
   - `sendResult.isErr() → shouldContinue = false`
   - `sendMessageV2` 返回 `false`

5. **工作流决策**:
   ```typescript
   if (!messageSucceeded && !currentNode.skipOnFailure) {
     // messageSucceeded = false, skipOnFailure = true
     // 条件为 false，不进入 if 块
   }
   // 继续下一个节点
   ```

### 场景四: 用户取消订阅（MessageSkipped）

1. **工作流状态**: 执行到 MessageNode 准备发送邮件
2. **sender 内部检查**:
   ```typescript
   if (subscriptionGroupDetails && !inSubscriptionGroup(subscriptionGroupDetails)) {
     return err({
       type: InternalEventType.MessageSkipped,
       variant: { type: MessageSkippedType.SubscriptionState, ... }
     });
   }
   ```
3. **Result 传递**:
   - `sender` 返回 `err(MessageSkipped)`（不抛异常）
   - `sendMessageInner` 收到这个 Result
   - `sendMessageWithSender` 转换 `isErr() → shouldContinue = false`
   - `sendMessageV2` 返回 `false`

4. **工作流决策**:
   - 检查 `skipOnFailure`
   - `true` → 继续
   - `false` → 取消旅程

**关键点**: MessageSkipped 不会触发 Temporal 重试，因为 `sender` 没有抛出异常。

### 场景五: 旅程被删除（两层检查差异）

**情况 A: 外层预检查捕获**

1. `sendMessageWithSender` 查询：
   ```typescript
   const journey = await db().query.journey.findFirst({
     where: and(
       eq(dbJourney.id, params.journeyId),
       eq(dbJourney.workspaceId, params.workspaceId),  // 带 workspaceId 条件
     ),
   });
   ```
2. 旅程不存在 → 直接返回 `false`
3. 不触发重试，工作流检查 `skipOnFailure`

**情况 B: 理论上的内层检查**

如果外层预检查没有捕获（例如旅程在外层查询后被删除）：

1. `sendMessageInner` 查询：
   ```typescript
   const journey = await db().query.journey.findFirst({
     where: eq(dbJourney.id, journeyId),  // 注意：没有 workspaceId 条件
   });
   ```
2. 旅程不存在 → 返回 `err(BadWorkspaceConfiguration.JourneyNotFound)`
3. `sendMessageWithSender` 转换 `isErr() → shouldContinue = false`

### 场景六: ContinueAsNew（循环旅程）

**配置**:
- `journey.canRunMultiple: true`
- `definition.entryNode.type: SegmentEntryNode`
- `definition.entryNode.reEnter: true`
- `journey.status: "Running"`
- 用户仍在入口 segment 中

1. **工作流结束**: 执行完所有节点，到达 ExitNode
2. **shouldReEnter 检查**:
   - 7 个条件全部满足 → 返回 `true`
3. **ContinueAsNew**:
   ```typescript
   if (shouldContinueAsNew) {
     await continueAsNew<typeof userJourneyWorkflow>(props);
   }
   ```
4. **结果**:
   - 创建新的工作流 Run（保持相同 Workflow ID）
   - 重置工作流历史

---

## 9. 关键文件位置

| 功能 | 文件路径 | 关键函数/配置 |
|-----|---------|-------------|
| 工作流定义 | `packages/backend-lib/src/journeys/userWorkflow.ts` | `userJourneyWorkflow`, `getKeyedUserJourneyWorkflowId` |
| Signal 定义 | `packages/backend-lib/src/journeys/userWorkflow.ts` | `trackSignal`, `segmentUpdateSignal`, `reEvaluateSegmentsSignal` |
| 工作流生命周期 | `packages/backend-lib/src/journeys/userWorkflow/lifecycle.ts` | `startKeyedUserJourney`, `signalWithStart` |
| 活动实现 | `packages/backend-lib/src/journeys/userWorkflow/activities.ts` | `isRunnable`, `shouldReEnter`, `sendMessageFactory`, `sendMessageInner` |
| 跳过类型定义 | `packages/isomorphic-lib/src/types.ts` | `MessageSkippedType`, `NonRetryableMessageSendFailure` |
| 消息跳过逻辑 | `packages/backend-lib/src/messaging.ts` | `sendMessage` 中的订阅检查、标识符检查 |
| 配置 | `packages/backend-lib/src/config.ts` | `defaultUserJourneyMaxAttempts`, `waitForComputePropertiesMaxAttempts` |
| 工作流终止 | `packages/backend-lib/src/computedProperties/computePropertiesWorkflow/lifecycle.ts` | `terminateComputePropertiesWorkflow` |
| 测试 (键控事件) | `packages/backend-lib/src/journeys/keyedEventEntry.test.ts` | 预约取消场景、Signal 追加测试 |
| 测试 (重复进入) | `packages/backend-lib/src/journeys/reEnter.test.ts` | continueAsNew 测试 |

---

## 10. 总结

### 10.1 幂等性关键发现

**isRunnable 精确逻辑**:
- **首次进入** (`previousExitEvent === null`): 直接返回 `true`，**不检查** journey/workspace 是否存在
- **非首次进入**: 检查 `journey.canRunMultiple`，journey 不存在时返回 `false`
- **workspaceId 是可选参数**，不传则跳过 workspace 状态检查

**三层幂等**:
1. 入口幂等: `signalWithStart` + `WorkflowExecutionAlreadyStartedError`
2. 历史检查: `isRunnable` 数据库查询
3. 运行时去重: `keyedEventIds` Set 防止重复 Signal

### 10.2 重试策略关键发现

**消息发送的三层结构**:
1. **sendMessageWithSender (外层)**: 预检查旅程状态，**不存在/非 Running 直接返回 `false`，不重试**
2. **sendMessageInner (内层)**: 详细状态检查，发送失败时根据 `isLastAttempt` 决定抛出还是返回 `JourneyEarlyExit`
3. **sender (实际发送)**: 业务检查（订阅状态、标识符），**MessageSkipped 不抛异常，不触发重试**

**触发 Temporal 重试的唯一条件**:
- `sender` 抛出异常
- `isLastAttempt === false`

### 10.3 取消/跳过/重发协同关键发现

**取消机制**:
- **硬取消**: `terminate()` 立即终止
- **软取消**: 长时运行节点后检查 `workspace.status`，优雅退出
- **活动级取消**: 旅程状态检查返回 `JourneyEarlyExit`

**跳过机制**:
- 工作流决策条件: `!messageSucceeded && !skipOnFailure`
- `messageSucceeded === false` 且 `skipOnFailure === false` 才会取消旅程

**重发机制**:
- **Temporal 重试**: 只有 `sender` 抛异常且非最后一次尝试才触发
- **Signal 追加**: 新事件通过 `trackSignal` 追加，WaitForNode 时触发 segment 重新评估

**shouldReEnter 严格条件**:
- 7 个必要条件必须全部满足
- 只支持 `SegmentEntryNode` 类型，不支持 `EventEntryNode`
- 必须 `reEnter === true` 且用户仍在入口 segment 中
