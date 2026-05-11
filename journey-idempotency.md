# Journey 工作流的幂等、重试与外部 Signal 协同分析

## 1. 概述

本文档深入分析 Dittofeed 中 Journey（用户旅程）工作流的三个核心机制及其协同关系：

- **幂等性 (Idempotency)**: 确保相同事件处理多次时产生相同结果
- **重试策略 (Retry)**: 处理异步活动失败时的恢复机制
- **外部 Signal 交互**: 外部取消、跳过、重发等信号与正在运行的工作流步骤的互动方式

这三个机制协同工作，确保用户旅程的可靠性、一致性和可控性。

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

**特点**: 
- 同一用户 + 同一旅程 = 同一工作流
- 保证用户在同一旅程中只有一个活跃实例

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
}: {
  workspaceId: string;
  userId: string;
  journeyId: string;
  entryNode: EventEntryNode;
  event: UserWorkflowTrackEvent;
}): string | null {
  let key: string;
  let keyValue: string;
  if (entryNode.key) {
    key = entryNode.key;
    const keyValueResult = jsonStringOrNumber({
      data: event.properties,
      path: key,
    })
      .map(String)
      .unwrapOr(null);
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

1. **用户定义键 (entryNode.key)**: 如果 EventEntryNode 配置了自定义 key，从事件属性中提取对应的值
   - 例如: `appointmentId`, `orderId` 等业务唯一标识
   
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

使用 Temporal 的 `signalWithStart` 原子操作，配合相同的工作流 ID，保证：
- 第一次调用：创建工作流并发送信号
- 后续调用：捕获 `WorkflowExecutionAlreadyStartedError`，静默忽略

**机制二: isRunnable 数据库检查**

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
    db().query.journey.findFirst({
      where: and(
        eq(dbJourney.id, journeyId),
        eq(dbJourney.workspaceId, workspaceId),
      ),
    }),
    ...
  ]);
  
  if (!journey) {
    return false;
  }
  
  if (!previousExitEvent) {
    return true;
  }
  
  const canRunMultiple = !!journey?.canRunMultiple;
  if (!canRunMultiple) {
    return false;
  }
  return canRunMultiple;
}
```

工作流启动时会检查：
- 数据库中是否存在该用户+旅程（+事件键）的历史记录
- 如果存在且旅程不允许重复运行（`canRunMultiple: false`），则工作流直接退出

**机制三: 运行时 Signal 去重**

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:444-459
wf.setHandler(trackSignal, async (event) => {
  logger.info("keyed event signal", {
    workspaceId,
    journeyId,
    userId,
    messageId: event.messageId,
  });
  if (keyedEventIds.has(event.messageId)) {
    logger.info("ignoring duplicate keyed event", {
      journeyId,
      userId,
      workspaceId,
      messageId: event.messageId,
    });
    return;
  }
  // ... 处理信号
  keyedEventIds.add(event.messageId);
});
```

工作流内部维护 `keyedEventIds` Set，防止相同事件信号被重复处理。

---

## 3. 重试策略详解

### 3.1 Temporal 活动重试配置

Dittofeed 的活动重试策略基于 Temporal SDK，通过 `proxyActivities` 的 `retry` 参数配置。

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:223-268
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

#### 3.2.1 核心旅程活动 (Core Journey Activities)

| 活动名称 | 超时设置 | 重试次数 (配置) | 说明 |
|---------|---------|---------------|------|
| `isRunnable`, `shouldReEnter`, `onNodeProcessedV2` | startToClose: 2分钟 | `defaultUserJourneyMaxAttempts` | 核心流程控制 |
| `getWorkspace` | startToClose: 2分钟 | `defaultUserJourneyMaxAttempts` | 工作空间检查 |
| `getUserPropertyDelay`, `findNextLocalizedTime` | startToClose: 2分钟 | `defaultUserJourneyMaxAttempts` | 延迟计算 |

#### 3.2.2 计算属性等待活动 (长时运行)

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

| 配置项 | 值 | 说明 |
|-------|---|------|
| startToCloseTimeout | 20分钟 | 长时运行任务 |
| heartbeatTimeout | 30秒 | 心跳检测，防止 worker 失联 |
| maximumAttempts | `waitForComputePropertiesMaxAttempts` | 可配置 |

#### 3.2.3 本地活动 (Local Activities)

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:247-268
const { getEventsById, getSegmentAssignment } = wf.proxyLocalActivities<...>({
  startToCloseTimeout: "2 minutes",
  retry: {
    maximumAttempts: defaultGetSegmentAndEventDetailsMaxAttempts,
  },
});

const { getRandomNumber } = wf.proxyLocalActivities<...>({
  startToCloseTimeout: "5 seconds",
  retry: {
    maximumAttempts: defaultUserJourneyMaxAttempts,
  },
});
```

本地活动在 worker 进程内执行，不通过 Temporal 服务调度。

#### 3.2.4 消息发送活动 (节点级配置)

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:955-960
const { sendMessageV2 } = proxyActivities<typeof activities>({
  startToCloseTimeout: "2 minutes",
  retry: {
    maximumAttempts: currentNode.retryCount ?? 3,
  },
});
```

**特殊点**: 消息发送的重试次数可在节点级别配置（`currentNode.retryCount`），默认为 3 次。

### 3.3 配置默认值

```typescript
// packages/backend-lib/src/config.ts:782-803
waitForComputePropertiesMaxAttempts: parseMaxAttempts(
  rawConfig.waitForComputePropertiesMaxAttempts,
  nodeEnv === NodeEnvEnum.Test ? 1 : 5,  // 测试: 1, 生产: 5
),
defaultUserJourneyMaxAttempts: rawConfig.defaultUserJourneyMaxAttempts !== undefined 
  ? parseInt(rawConfig.defaultUserJourneyMaxAttempts)
  : (nodeEnv === NodeEnvEnum.Test ? 1 : undefined),  // 测试: 1, 生产: undefined (使用 Temporal 默认)
defaultGetSegmentAndEventDetailsMaxAttempts: parseMaxAttempts(
  rawConfig.defaultGetSegmentAndEventDetailsMaxAttempts,
  nodeEnv === NodeEnvEnum.Test ? 1 : 10,  // 测试: 1, 生产: 10
),
```

| 配置项 | 测试环境 | 生产环境 |
|-------|---------|---------|
| `waitForComputePropertiesMaxAttempts` | 1 | 5 |
| `defaultUserJourneyMaxAttempts` | 1 | undefined (Temporal 默认) |
| `defaultGetSegmentAndEventDetailsMaxAttempts` | 1 | 10 |

### 3.4 消息发送的重试与跳过协同

消息发送失败后的处理流程涉及重试和跳过机制的协同：

```typescript
// packages/backend-lib/src/journeys/userWorkflow/activities.ts:258-300
try {
  const result = await sender({...});
  return result;
} catch (senderError) {
  const activityInfo = Context.current().info;
  const isLastAttempt = activityInfo.attempt >= (retryCount ?? 3);

  if (isLastAttempt) {
    logger().error("sender failed after maximum retry attempts", {...});
    // 返回 JourneyEarlyExit 而不是抛出，让工作流决定下一步
    return err({
      type: InternalEventType.JourneyEarlyExit,
      message: `Message failed after maximum retry attempts: ${senderErrorString}`,
    });
  }
  // 非最后一次尝试，重新抛出以触发 Temporal 重试
  throw senderError;
}
```

**重试策略**:
- 非最后一次失败：抛出异常 → Temporal 自动重试（指数退避）
- 最后一次失败：返回 `JourneyEarlyExit` 错误 → 工作流根据 `skipOnFailure` 决定是否继续

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
  logger.info("message node early exit", {
    ...defaultLoggingFields,
    child: currentNode.child,
  });
  nextNode = definition.exitNode;
  break;
}
```

**跳过决策**:
- `skipOnFailure: true` → 消息失败后继续执行下一个节点（跳过失败）
- `skipOnFailure: false` → 消息失败后跳转到退出节点（取消旅程）

### 3.5 消息跳过的类型 (MessageSkipped)

除了重试失败后的跳过，还有两种消息跳过场景：

```typescript
// packages/isomorphic-lib/src/types.ts:4503-4506
export enum MessageSkippedType {
  SubscriptionState = "SubscriptionState",
  MissingIdentifier = "MissingIdentifier",
}
```

#### 3.5.1 SubscriptionState - 订阅状态跳过

```typescript
// packages/backend-lib/src/messaging.ts:422-435
if (
  subscriptionGroupDetails &&
  !inSubscriptionGroup(subscriptionGroupDetails)
) {
  const { type: subscriptionGroupType, action: subscriptionGroupAction } =
    subscriptionGroupDetails;
  return err({
    type: InternalEventType.MessageSkipped,
    variant: {
      type: MessageSkippedType.SubscriptionState,
      action: subscriptionGroupAction,
      subscriptionGroupType,
    },
  });
}
```

**场景**: 用户取消订阅 (Unsubscribe) 或被加入退订列表，消息被跳过。

#### 3.5.2 MissingIdentifier - 缺少标识符跳过

```typescript
// packages/backend-lib/src/messaging.ts:960-968
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

**场景**: 用户缺少发送消息所需的标识符（如 email、phone），消息被跳过。

### 3.6 应用级重试 (非 Temporal 重试)

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
    // ... 日志记录
    throw e;
  }
}
```

使用 `p-retry` 库实现的应用级重试：
- **最多 10 次重试**
- **指数退避**: 50ms ~ 1000ms
- **最大总等待时间**: 60秒

**适用场景**: ClickHouse 事件查询，处理数据写入延迟（写入后立即可读性延迟）

---

## 4. 取消、跳过、重发与工作流步骤的协同

### 4.1 取消机制 (Cancellation / Termination)

#### 4.1.1 工作流 Terminate 调用

Dittofeed 使用 Temporal 的 `terminate()` API 来强制终止工作流：

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
    logger().info(
      {
        err: e,
      },
      "Failed to terminate compute properties workflow.",
    );
  }
}
```

**终止场景**:
- 工作空间禁用/删除
- 计算属性工作流需要重置
- 功能开关变更

#### 4.1.2 工作空间状态检查 (软取消)

在长时运行节点结束后，会检查工作空间状态：

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:1100-1111
const LONG_RUNNING_NODE_TYPES = new Set<JourneyNodeType>([
  JourneyNodeType.WaitForNode,
  JourneyNodeType.DelayNode,
  JourneyNodeType.SegmentEntryNode,
]);

// check if workspace is inactive after a long running node
if (LONG_RUNNING_NODE_TYPES.has(currentNode.type)) {
  const workspace = await getWorkspace(workspaceId);
  if (workspace?.status !== "Active") {
    logger.info("workspace is not active, exiting journey", {
      workspaceId,
      userId,
      journeyId,
    });
    break;
  }
}
```

**软取消机制**:
- 工作流在长时运行节点（DelayNode、WaitForNode、SegmentEntryNode）结束后检查工作空间状态
- 如果工作空间不再 Active，直接 `break` 退出 `nodeLoop`
- 允许当前节点完成，然后优雅退出

#### 4.1.3 旅程状态检查 (活动级别取消)

在消息发送等活动中，会检查旅程是否仍然运行：

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

**活动级别取消**:
- 旅程被禁用或删除后，活动返回 `JourneyEarlyExit`
- 工作流收到后决定是否继续或退出

#### 4.1.4 取消与 Signal 的交互

当工作流正在 **WaitForNode** 或 **SegmentEntryNode** 等待时：

```
┌─────────────────────────────────────────────────────────────────┐
│                    工作流等待状态                                │
│                                                                 │
│  工作流正在执行: wf.condition(() => segmentAssignedTrue(segId))  │
│                                                                 │
│  同时可以接收 Signal:                                            │
│  ├── segmentUpdateSignal → 更新 segmentAssignments              │
│  ├── trackSignal → 追加事件并重新评估 segment                   │
│  └── reEvaluateSegmentsSignal → 强制重新查询数据库              │
│                                                                 │
│  如果工作流被 terminate():                                      │
│  ├── 正在执行的活动被取消                                        │
│  ├── 工作流立即停止，不再处理任何节点                            │
│  └── 不会触发后续节点的 onNodeProcessed                          │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 跳过机制 (Skip)

#### 4.2.1 skipOnFailure - 消息发送失败后的跳过

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:961-970
const messageSucceeded = await sendMessageV2(sendMesssageParams);

if (!messageSucceeded && !currentNode.skipOnFailure) {
  logger.info("message node early exit", {
    ...defaultLoggingFields,
    child: currentNode.child,
  });
  nextNode = definition.exitNode;
  break;
}

// 如果 skipOnFailure: true，继续执行下一个节点
nextNode = nodes.get(currentNode.child) ?? null;
```

**跳过决策流程**:
```
消息发送活动调用
       │
       ▼
┌───────────────┐     否     ┌─────────────────┐
│ 消息发送成功?  │──────────▶│ 继续下一个节点  │
└───────┬───────┘           └─────────────────┘
        │
        │ 是 (失败)
        ▼
┌───────────────┐     是     ┌─────────────────┐
│ skipOnFailure?│──────────▶│ 跳过，继续下一节点│
└───────┬───────┘           └─────────────────┘
        │
        │ 否
        ▼
┌───────────────────────┐
│ nextNode = exitNode   │
│ break nodeLoop        │
│ (取消旅程)            │
└───────────────────────┘
```

#### 4.2.2 跳过与重试的协同

```typescript
// packages/backend-lib/src/journeys/userWorkflow/activities.ts:270-300
catch (senderError) {
  const activityInfo = Context.current().info;
  const isLastAttempt = activityInfo.attempt >= (retryCount ?? 3);

  if (isLastAttempt) {
    // 最后一次尝试失败，返回 JourneyEarlyExit
    return err({
      type: InternalEventType.JourneyEarlyExit,
      message: `Message failed after maximum retry attempts`,
    });
  }
  // 非最后一次，抛出异常触发 Temporal 重试
  throw senderError;
}
```

**重试与跳过的协同流程**:
```
第 N 次发送失败
       │
       ▼
┌──────────────────────┐
│ attempt >= retryCount│
└──────────┬───────────┘
           │
     ┌─────┴─────┐
     │ 否        │ 是
     ▼           ▼
┌─────────┐  ┌───────────────┐
│抛出异常 │  │ JourneyEarlyExit│
│Temporal │  │ 工作流决定    │
│重试     │  │ skipOnFailure │
└─────────┘  └───────┬───────┘
                     │
              ┌──────┴──────┐
              │ true        │ false
              ▼             ▼
         ┌─────────┐   ┌─────────┐
         │跳过继续 │   │取消旅程 │
         └─────────┘   └─────────┘
```

### 4.3 重发机制 (Retry / Resend)

#### 4.3.1 Temporal 活动级重试

消息发送使用 Temporal SDK 的重试机制：

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:955-960
const { sendMessageV2 } = proxyActivities<typeof activities>({
  startToCloseTimeout: "2 minutes",
  retry: {
    maximumAttempts: currentNode.retryCount ?? 3,
  },
});
```

**Temporal 重试特性**:
- 指数退避 (Exponential Backoff)
- 可配置 `maximumAttempts`（节点级 `retryCount` 或默认 3）
- 失败活动在 worker 侧重试，不影响工作流历史

#### 4.3.2 键控事件的 Signal 追加 (重发事件)

当同一业务键的事件再次到达时，通过 `trackSignal` 追加到已运行的工作流：

```typescript
// packages/backend-lib/src/journeys/keyedEventEntry.test.ts:1415-1440
// 测试场景：预约被取消，发送取消事件
const cancelledEvent = {
  type: EventType.Track,
  event: "APPOINTMENT_UPDATE",
  userId,
  messageId: randomUUID(),
  properties: {
    operation: "CANCELLED",
    appointmentId: appointmentId1,  // 同一 appointmentId
  },
  timestamp: new Date(cancelTime).toISOString(),
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
  // 1. 去重检查
  if (keyedEventIds.has(event.messageId)) {
    logger.info("ignoring duplicate keyed event", {...});
    return;
  }
  
  // 2. 根据版本处理事件
  switch (event.version) {
    case TrackSignalParamsVersion.V2: {
      // 从数据库加载完整事件
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
        segmentAssignments.set(segmentId, {...});
      }),
    );
  }
  reportWorkflowInfoHandler();
});
```

**Signal 重发事件与工作流步骤的协同**:

```
工作流状态: 在 WaitForNode 等待 "appointment_cancelled" segment
           (等待预约取消事件)

步骤 1: 取消事件到达
        └─▶ signalWithStart 或单独 signal 发送 trackSignal

步骤 2: Signal Handler 执行
        └─▶ 去重检查 keyedEventIds.has(messageId)
        └─▶ 追加事件到 keyedEvents
        └─▶ 添加到 keyedEventIds

步骤 3: WaitForNode 中的 segment 重新评估
        └─▶ waitForSegmentIds 不为 null (正在等待)
        └─▶ 调用 getSegmentAssignmentHandler 重新查询
        └─▶ 新事件可能使 segment 变为 true

步骤 4: wf.condition 唤醒
        └─▶ segmentAssignedTrue(segmentId) 返回 true
        └─▶ 工作流继续执行 cancel 分支
        └─▶ 发送取消通知消息
```

### 4.4 三种机制与工作流步骤的交互矩阵

| 机制 | 触发方式 | 与运行中步骤的交互 | 效果 |
|-----|---------|-----------------|------|
| **取消 (Terminate)** | `client.getHandle().terminate()` | 立即终止，不等待当前步骤 | 工作流完全停止，无清理 |
| **取消 (软取消)** | 工作空间状态检查 | 长时运行节点结束后检查 | 优雅退出，当前节点完成 |
| **取消 (活动级)** | 旅程状态检查 | 活动执行时检查 | 返回 `JourneyEarlyExit`，工作流决定 |
| **跳过 (skipOnFailure)** | 消息发送失败后 | 工作流决策层 | 失败节点跳过，继续下一节点 |
| **跳过 (订阅状态)** | 订阅检查 | 消息发送前检查 | `MessageSkipped`，工作流决定 |
| **跳过 (缺少标识符)** | 标识符检查 | 消息发送前检查 | `MessageSkipped`，工作流决定 |
| **重发 (Temporal 重试)** | 活动失败抛出 | 活动层重试 | 对工作流透明，同一次活动调用 |
| **重发 (Signal 追加)** | 新事件 signal | 工作流内部状态更新 | 追加事件，等待节点重新评估 |

---

## 5. 外部 Signal 与工作流步骤的互动详解

### 5.1 Signal 定义概览

Dittofeed 的 UserJourneyWorkflow 定义了三种信号：

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:51-77
export const segmentUpdateSignal =
  wf.defineSignal<[SegmentUpdate]>("segmentUpdate");

export const reEvaluateSegmentsSignal =
  wf.defineSignal<[ReEvaluateSegmentsParams]>("reEvaluateSegments");

export const trackSignal = wf.defineSignal<[TrackSignalParams]>("track");
```

### 5.2 三种 Signal 与工作流步骤的互动

#### 5.2.1 trackSignal - 键控事件信号

**用途**: 向已运行的键控工作流发送额外事件（如订单更新、预约取消等）

**与工作流步骤的互动**:

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:444-527
wf.setHandler(trackSignal, async (event) => {
  logger.info("keyed event signal", {...});
  
  // 去重检查
  if (keyedEventIds.has(event.messageId)) {
    logger.info("ignoring duplicate keyed event", {...});
    return;
  }
  
  // 根据版本处理事件
  switch (event.version) {
    case TrackSignalParamsVersion.V2: {
      // 从数据库加载事件详情
      const newEvents = await getEventsById({
        workspaceId,
        eventIds: [event.messageId],
      });
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
  
  // 如果正在 WaitFor 节点，立即重新评估 segment
  if (waitForSegmentIds) {
    await Promise.all(
      waitForSegmentIds.map(async ({ segmentId }) => {
        const nowMs = Date.now();
        const assignment = await getSegmentAssignmentHandler({
          segmentId,
          now: nowMs,
        });
        if (assignment === null) return;
        segmentAssignments.set(segmentId, {
          currentlyInSegment: assignment.inSegment,
          segmentVersion: nowMs,
        });
      }),
    );
  }
  reportWorkflowInfoHandler();
});
```

**交互场景**:

1. **WaitForNode 中**: 
   - `waitForSegmentIds` 不为 null
   - 信号触发后立即重新评估所有等待中的 segment
   - 如果满足条件，`wf.condition` 会被唤醒

2. **DelayNode 中**:
   - 事件被加入 `keyedEvents` / `keyedEventIds`
   - 不立即影响当前睡眠
   - 等待 DelayNode 结束后，后续节点使用新事件

3. **MessageNode 中**:
   - 事件被加入 `keyedEvents`
   - 消息发送可能使用新事件的属性

#### 5.2.2 segmentUpdateSignal - Segment 变更通知

**用途**: 外部系统通知 segment 分配状态变更（如计算属性工作流检测到 segment 变化）

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:529-549
wf.setHandler(segmentUpdateSignal, (update) => {
  const prev = segmentAssignments.get(update.segmentId);
  const loggerAttrs = {...};
  
  // 版本检查，忽略过期更新
  if (prev && prev.segmentVersion >= update.segmentVersion) {
    logger.info("ignoring stale segment update", loggerAttrs);
    return;
  }

  logger.info("segment update", loggerAttrs);
  segmentAssignments.set(update.segmentId, {
    currentlyInSegment: update.currentlyInSegment,
    segmentVersion: update.segmentVersion,
  });
});
```

**特点**:
- 使用 `segmentVersion` 进行版本控制
- 旧版本信号被静默忽略
- 同步处理，不阻塞工作流执行

**与工作流步骤的互动**:

1. **SegmentEntryNode 等待中**:
   ```typescript
   // packages/backend-lib/src/journeys/userWorkflow.ts:627-648
   case JourneyNodeType.SegmentEntryNode: {
     const cn = currentNode;
     const initialSegmentAssignment =
       (
         await getSegmentAssignmentHandler({
           segmentId: cn.segment,
           now: Date.now(),
         })
       )?.inSegment === true;
     if (!initialSegmentAssignment) {
       await wf.condition(() => segmentAssignedTrue(cn.segment));
     }
     // ...
   }
   ```
   - `segmentUpdateSignal` 更新 `segmentAssignments`
   - `wf.condition` 检测到条件满足，唤醒工作流

2. **WaitForNode 等待中**:
   ```typescript
   // packages/backend-lib/src/journeys/userWorkflow.ts:783-788
   if (!satisfiedSegmentWithinTimeout) {
     satisfiedSegmentWithinTimeout = await wf.condition(
       () => segmentChildren.some((s) => segmentAssignedTrue(s.segmentId)),
       timeoutSeconds * 1000,
     );
   }
   ```
   - 任何一个 segment 变为 true 都会唤醒
   - 超时后走 timeout 分支

#### 5.2.3 reEvaluateSegmentsSignal - 强制重新评估

**用途**: 强制工作流重新从数据库查询 segment 状态（不依赖 segment 变更事件）

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:551-599
wf.setHandler(reEvaluateSegmentsSignal, async (params) => {
  logger.info("re-evaluating segments", {...});

  const segmentIdsToEvaluate =
    params.segmentIds ?? Array.from(segmentAssignments.keys());
  const nowMs = Date.now();

  await Promise.all(
    segmentIdsToEvaluate.map(async (segmentId) => {
      const assignment = await getSegmentAssignmentHandler({
        segmentId,
        now: nowMs,
      });

      if (assignment === null) {
        logger.warn("segment assignment returned null during re-evaluation", {...});
        return;
      }

      segmentAssignments.set(segmentId, {
        currentlyInSegment: assignment.inSegment,
        segmentVersion: nowMs,
      });

      logger.info("segment re-evaluated", {...});
    }),
  );

  reportWorkflowInfoHandler();
});
```

**触发场景**:
- 外部系统修改了用户属性
- 需要立即同步最新状态
- 不依赖 segment 变更事件

### 5.3 Signal 发送方式

#### 5.3.1 SignalWithStart (原子操作)

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
// 测试示例: 向已存在的工作流发送取消事件
await handle1.signal(trackSignal, {
  version: TrackSignalParamsVersion.V2,
  messageId: cancelledEvent.messageId,
});
```

用于向已存在的工作流发送信号。

### 5.4 长时运行节点的详细交互

#### 5.4.1 长时运行节点类型

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:200-204
const LONG_RUNNING_NODE_TYPES = new Set<JourneyNodeType>([
  JourneyNodeType.WaitForNode,
  JourneyNodeType.DelayNode,
  JourneyNodeType.SegmentEntryNode,
]);
```

#### 5.4.2 不同长时运行节点的 Signal 响应

| 节点类型 | 当前状态 | 收到 Signal 后的行为 |
|---------|---------|-------------------|
| **WaitForNode** | `wf.condition()` 等待中 | 立即重新评估 segment，可能唤醒工作流 |
| **DelayNode** | `await sleep()` 睡眠中 | Signal Handler 执行（更新状态），但不打断睡眠 |
| **SegmentEntryNode** | `wf.condition()` 等待中 | 检查 segment 条件，满足则唤醒 |

**DelayNode 中的 Signal 处理**:
```
工作流正在执行 DelayNode: await sleep(1小时)

时间线:
0:00   工作流进入 DelayNode，开始 sleep
0:30   收到 trackSignal (新事件到达)
       └─▶ Signal Handler 执行
       └─▶ 事件追加到 keyedEvents
       └─▶ 如果有 waitForSegmentIds，重新评估 segment
       └─▶ 但 sleep 仍在继续...
1:00   sleep 结束
       └─▶ 工作流继续执行后续节点
       └─▶ 后续节点可以访问新事件
```

---

## 6. 三者协同机制总结

### 6.1 协同流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          外部事件 / Signal 入口                               │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. 幂等性检查层 (Idempotency Layer)                                          │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 工作流 ID 生成 (uuidV5 + 业务键)                                      │   │
│  │ - 普通: user-journey-{userId}-{journeyId}                            │   │
│  │ - 键控: user-journey-keyed-{...uuidV5(userId, eventKey, value)...}   │   │
│  │                                                                     │   │
│  │ signalWithStart + WorkflowExecutionAlreadyStartedError               │   │
│  │ - 第一次: 创建工作流 + 发送 signal                                    │   │
│  │ - 重复: 捕获异常，静默忽略                                            │   │
│  │                                                                     │   │
│  │ isRunnable 数据库检查                                                │   │
│  │ - 查询 userJourneyEvent 历史记录                                     │   │
│  │ - canRunMultiple: false → 直接退出                                   │   │
│  │                                                                     │   │
│  │ 运行时 Signal 去重 (keyedEventIds Set)                               │   │
│  │ - 相同 messageId 的 signal 被忽略                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ 检查通过
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. 工作流执行层 (Workflow Execution)                                         │
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
│  3. Signal 处理层 (Signal Handlers)                                          │
│                                                                             │
│  ┌──────────────────┐ ┌──────────────────┐ ┌─────────────────────┐         │
│  │   trackSignal    │ │segmentUpdateSignal│ │ reEvaluateSegments  │         │
│  │                  │ │                  │ │                     │         │
│  │ 事件追加         │ │ 版本控制更新      │ │ 强制重新查询数据库   │         │
│  │ keyedEvents     │ │ segmentVersion   │ │ getSegmentAssignment │         │
│  │ keyedEventIds   │ │                  │ │                     │         │
│  └────────┬─────────┘ └────────┬─────────┘ └──────────┬──────────┘         │
│           │                   │                      │                      │
│           ▼                   ▼                      ▼                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 更新 segmentAssignments / keyedEvents                               │   │
│  │                                                                     │   │
│  │ 如果在等待节点 (WaitFor / SegmentEntry):                            │   │
│  │ └─▶ wf.condition() 检测条件变化 → 唤醒工作流                         │   │
│  │                                                                     │   │
│  │ 如果在非等待节点 (Delay / Message):                                 │   │
│  │ └─▶ 状态更新，但不打断当前执行                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ 活动失败
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. 重试层 (Retry Layer)                                                     │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Temporal 活动重试                                                   │   │
│  │ - maximumAttempts (节点级配置或默认值)                              │   │
│  │ - 指数退避                                                          │   │
│  │ - 心跳超时检测 (waitForComputeProperties)                           │   │
│  │                                                                     │   │
│  │ 非最后一次失败 → 抛出异常 → Temporal 自动重试                        │   │
│  │                                                                     │   │
│  │ 最后一次失败 → 返回 JourneyEarlyExit → 工作流决策                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 应用级重试 (p-retry) - getEventsByIdWithRetry                      │   │
│  │ - 10次重试，50ms-1000ms 指数退避                                    │   │
│  │ - 最大 60秒总等待                                                   │   │
│  │ - 处理 ClickHouse 写入延迟                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ 重试耗尽或跳过
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. 决策层 (Decision Layer) - 取消 / 跳过                                    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 消息发送失败决策                                                     │   │
│  │                                                                     │   │
│  │ JourneyEarlyExit 到达时:                                            │   │
│  │ ┌─────────────────────────────────────────────────────────────┐   │   │
│  │ │ skipOnFailure ?                                              │   │   │
│  │ ├── true  ──▶ 跳过失败，继续下一节点 (nextNode = child)         │   │   │
│  │ └── false ──▶ 取消旅程，退出    (nextNode = exitNode, break)   │   │   │
│  │ └─────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 消息跳过决策 (MessageSkipped)                                        │   │
│  │                                                                     │   │
│  │ SubscriptionState:                                                  │   │
│  │ - 用户取消订阅 → MessageSkipped → 工作流决定                        │   │
│  │                                                                     │   │
│  │ MissingIdentifier:                                                  │   │
│  │ - 缺少 email/phone → MessageSkipped → 工作流决定                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 工作流取消 (Termination)                                             │   │
│  │                                                                     │   │
│  │ 硬取消:                                                             │   │
│  │ - client.getHandle().terminate()                                    │   │
│  │ - 立即停止，不等待当前活动                                           │   │
│  │                                                                     │   │
│  │ 软取消:                                                             │   │
│  │ - 长时运行节点后检查 workspace.status                                │   │
│  │ - 非 Active 时 break 退出                                           │   │
│  │ - 当前节点完成，优雅退出                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 工作流重复进入 (ContinueAsNew)                                       │   │
│  │                                                                     │   │
│  │ shouldReEnter() 检查:                                               │   │
│  │ - journey.canRunMultiple === true                                   │   │
│  │ - 仍有新事件等待处理                                                │   │
│  │                                                                     │   │
│  │ continueAsNew<typeof userJourneyWorkflow>(props)                    │   │
│  │ - 保持相同 Workflow ID                                              │   │
│  │ - 重置工作流历史，避免无限增长                                       │   │
│  │ - 保持幂等性                                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 典型场景分析

#### 场景一: 重复订单事件 (幂等性)

1. **事件**: `order_created` 事件到达，`orderId: "ORD001"`
2. **幂等检查**:
   - 工作流 ID: `user-journey-keyed-{workspaceId}-{journeyId}-{uuidV5(userId, "orderId", "ORD001", workspaceId)}`
   - 第一次: `signalWithStart` 创建工作流
   - 重复事件: 捕获 `WorkflowExecutionAlreadyStartedError`，忽略
3. **工作流内部**:
   - `keyedEventIds.add("message-id-001")`
   - 再次收到相同 signal → `keyedEventIds.has()` → 忽略

#### 场景二: 消息发送失败 → 重试 → 跳过/取消

1. **节点**: MessageNode 发送邮件，配置 `retryCount: 3`，`skipOnFailure: true`
2. **第 1 次发送失败**:
   - 活动抛出异常
   - Temporal 自动重试（指数退避）
3. **第 2 次发送失败**:
   - 活动抛出异常
   - Temporal 自动重试
4. **第 3 次发送失败 (last attempt)**:
   - 活动不抛出异常，返回 `JourneyEarlyExit`
   - 工作流检查 `skipOnFailure: true`
   - 跳过失败，继续执行下一个节点

**如果 skipOnFailure: false**:
   - 工作流设置 `nextNode = definition.exitNode`
   - `break nodeLoop`，取消旅程

#### 场景三: 预约取消事件 (Signal + WaitForNode)

1. **工作流状态**: 在 WaitForNode 等待用户进入 "appointment_cancelled" segment
   - 配置: 当 `APPOINTMENT_UPDATE` 事件 `operation: "CANCELLED"` 时，segment 变为 true

2. **取消事件到达**:
   - 事件: `{event: "APPOINTMENT_UPDATE", properties: {appointmentId: "APT001", operation: "CANCELLED"}}`
   - `getKeyedUserJourneyWorkflowId` 生成相同的工作流 ID（基于 `appointmentId`）

3. **Signal 发送**:
   - 捕获 `WorkflowExecutionAlreadyStartedError`
   - 或者单独调用 `handle.signal(trackSignal, {...})`

4. **Signal Handler 执行**:
   - 去重检查通过（新的 messageId）
   - 事件追加到 `keyedEvents`
   - `waitForSegmentIds` 不为 null（正在等待）
   - 调用 `getSegmentAssignmentHandler` 重新评估
   - 新事件使 segment 变为 true

5. **工作流唤醒**:
   - `wf.condition(() => segmentAssignedTrue(cancelledSegmentId))` 返回 true
   - 工作流继续执行取消分支
   - 发送取消通知邮件

#### 场景四: 用户取消订阅 (MessageSkipped)

1. **工作流状态**: 执行到 MessageNode 准备发送邮件
2. **订阅检查**:
   ```typescript
   // packages/backend-lib/src/messaging.ts:422-435
   if (subscriptionGroupDetails && !inSubscriptionGroup(subscriptionGroupDetails)) {
     return err({
       type: InternalEventType.MessageSkipped,
       variant: {
         type: MessageSkippedType.SubscriptionState,
         action: SubscriptionChange.Unsubscribe,
         ...
       },
     });
   }
   ```
3. **消息跳过**:
   - 活动返回 `MessageSkipped` 错误
   - `sendMessageV2` 返回 `false`（消息未成功发送）
4. **工作流决策**:
   - 检查 `skipOnFailure`
   - 如果 `true`: 继续下一个节点
   - 如果 `false`: 取消旅程

#### 场景五: 工作流 ContinueAsNew (循环旅程)

1. **旅程配置**: `canRunMultiple: true`
2. **工作流结束**: 执行完所有节点，到达 ExitNode
3. **重复进入检查**:
   ```typescript
   // packages/backend-lib/src/journeys/userWorkflow/activities.ts:777-810
   export async function shouldReEnter({
     journeyId,
     userId,
     workspaceId,
   }): Promise<boolean> {
     const journey = await db().query.journey.findFirst({...});
     if (!journey || !journey.canRunMultiple) {
       return false;
     }
     // 检查是否有新事件等待处理
     // ...
     return true;
   }
   ```
4. **ContinueAsNew**:
   ```typescript
   // packages/backend-lib/src/journeys/userWorkflow.ts:1127-1133
   if (await shouldReEnter({ journeyId, userId, workspaceId })) {
     if (shouldContinueAsNew) {
       await continueAsNew<typeof userJourneyWorkflow>(props);
     } else {
       return props;
     }
   }
   ```
5. **结果**:
   - 创建新的工作流 Run（保持相同的 Workflow ID）
   - 重置工作流历史，避免历史无限增长
   - 保持幂等性（同一时间只有一个运行实例）

---

## 7. 关键文件位置

| 功能 | 文件路径 | 关键函数/配置 |
|-----|---------|-------------|
| 工作流定义 | `packages/backend-lib/src/journeys/userWorkflow.ts` | `userJourneyWorkflow`, `getKeyedUserJourneyWorkflowId` |
| Signal 定义 | `packages/backend-lib/src/journeys/userWorkflow.ts` | `trackSignal`, `segmentUpdateSignal`, `reEvaluateSegmentsSignal` |
| 工作流生命周期 | `packages/backend-lib/src/journeys/userWorkflow/lifecycle.ts` | `startKeyedUserJourney`, `signalWithStart` |
| 活动实现 | `packages/backend-lib/src/journeys/userWorkflow/activities.ts` | `isRunnable`, `shouldReEnter`, `sendMessageV2`, `getEventsByIdWithRetry` |
| 跳过类型定义 | `packages/isomorphic-lib/src/types.ts` | `MessageSkippedType`, `NonRetryableMessageSendFailure` |
| 消息跳过逻辑 | `packages/backend-lib/src/messaging.ts` | `sendMessage` 中的订阅检查、标识符检查 |
| 配置 | `packages/backend-lib/src/config.ts` | `defaultUserJourneyMaxAttempts`, `waitForComputePropertiesMaxAttempts` |
| 工作流终止 | `packages/backend-lib/src/computedProperties/computePropertiesWorkflow/lifecycle.ts` | `terminateComputePropertiesWorkflow`, `resetComputePropertiesWorkflow` |
| 工作流入站拦截器 | `packages/backend-lib/src/temporal/workflowInboundCallsInterceptor.ts` | `DittofeedWorkflowInboundInterceptor` |
| 重试工具函数 | `packages/backend-lib/src/retry.ts` | `retryExponential` |
| 测试 (键控事件) | `packages/backend-lib/src/journeys/keyedEventEntry.test.ts` | 预约取消场景、Signal 追加测试 |
| 测试 (重复进入) | `packages/backend-lib/src/journeys/reEnter.test.ts` | continueAsNew 测试 |

---

## 8. 总结

Dittofeed 的 Journey 工作流通过三层机制保证可靠性，并实现了取消、跳过、重发与工作流步骤的精确协同：

### 8.1 幂等性机制

- **工作流 ID 生成**: 业务键的 UUID v5 哈希（`workspaceId` + `userId` + `journeyId` + `eventKey` + `eventKeyValue`）
- **入口幂等**: `signalWithStart` + `WorkflowExecutionAlreadyStartedError`
- **历史检查**: `isRunnable` 数据库查询，支持 `canRunMultiple` 配置
- **运行时去重**: `keyedEventIds` Set 防止重复 Signal

### 8.2 重试策略

- **Temporal 活动重试**: 可配置 `maximumAttempts`，指数退避
- **节点级配置**: `MessageNode.retryCount` 覆盖默认值
- **应用级重试**: `p-retry` 处理 ClickHouse 写入延迟（10次，50ms-1000ms）
- **最后一次尝试**: 返回 `JourneyEarlyExit` 而非抛出，让工作流决策

### 8.3 取消、跳过、重发与工作流步骤的协同

**取消 (Cancellation)**:
- **硬取消**: `client.getHandle().terminate()` 立即终止
- **软取消**: 长时运行节点后检查 `workspace.status`，优雅退出
- **活动级取消**: 旅程状态检查，返回 `JourneyEarlyExit`

**跳过 (Skip)**:
- **skipOnFailure**: 消息重试耗尽后，工作流决定跳过或取消
- **MessageSkipped.SubscriptionState**: 用户取消订阅时跳过
- **MessageSkipped.MissingIdentifier**: 缺少标识符时跳过

**重发 (Retry/Resend)**:
- **Temporal 重试**: 活动层透明重试，对工作流无感知
- **Signal 追加**: 新事件通过 `trackSignal` 追加到已运行工作流
- **WaitFor 节点响应**: Signal 触发 segment 重新评估，可能唤醒工作流

### 8.4 协同保证

这三个机制协同工作，确保：
- **一致性**: 相同事件只处理一次（幂等键 + 去重）
- **可靠性**: 失败时自动重试（Temporal + 应用级）
- **可控性**: 失败后可选择跳过或取消（`skipOnFailure`）
- **响应性**: 外部事件通过 Signal 实时影响工作流（`trackSignal` + `wf.condition`）
- **可观测性**: 完整的日志和事件追踪（`onNodeProcessedV2`, `InternalEventType`）
