# Journey 工作流的幂等、重试与外部 Signal 协同分析

## 1. 概述

本文档深入分析 Dittofeed 中 Journey（用户旅程）工作流的三个核心机制：

- **幂等性 (Idempotency)**: 确保相同事件处理多次时产生相同结果
- **重试策略 (Retry)**: 处理异步活动失败时的恢复机制
- **外部 Signal 交互**: 外部信号与正在运行的工作流步骤的互动方式

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

**机制二: isRunnable 检查**

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
    ...
  ]);
  
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

**机制三: 重复 Signal 去重**

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

## 3. Temporal 异步活动重试策略

### 3.1 重试配置概览

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

#### 3.2.4 消息发送活动

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

### 3.4 消息发送失败处理

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
    // 返回 MessageSkipped 而不是抛出
    return err({
      type: InternalEventType.JourneyEarlyExit,
      message: `Message failed after maximum retry attempts: ${senderErrorString}`,
    });
  }
  // 非最后一次尝试，重新抛出以触发重试
  throw senderError;
}
```

**策略**:
- 非最后一次失败：抛出异常 → Temporal 自动重试
- 最后一次失败：返回 `JourneyEarlyExit` 错误 → 工作流根据 `skipOnFailure` 决定是否继续

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:961-970
const messageSucceeded = await sendMessageV2(sendMesssageParams);

if (!messageSucceeded && !currentNode.skipOnFailure) {
  logger.info("message node early exit", {...});
  nextNode = definition.exitNode;
  break;
}
```

如果消息发送最终失败：
- `skipOnFailure: true` → 继续执行下一个节点
- `skipOnFailure: false` → 跳转到退出节点，结束工作流

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

## 4. 外部 Signal 与工作流步骤的互动

### 4.1 Signal 定义概览

Dittofeed 的 UserJourneyWorkflow 定义了三种信号：

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:51-77
export const segmentUpdateSignal =
  wf.defineSignal<[SegmentUpdate]>("segmentUpdate");

export const reEvaluateSegmentsSignal =
  wf.defineSignal<[ReEvaluateSegmentsParams]>("reEvaluateSegments");

export const trackSignal = wf.defineSignal<[TrackSignalParams]>("track");
```

### 4.2 信号处理机制

#### 4.2.1 trackSignal - 键控事件信号

**用途**: 向已运行的键控工作流发送额外事件

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

**与工作流步骤的互动**:

1. **WaitForNode 中**: 
   - `waitForSegmentIds` 不为 null
   - 信号触发后立即重新评估所有等待中的 segment
   - 如果满足条件，`wf.condition` 会被唤醒

2. **其他节点中**:
   - 事件被加入 `keyedEvents` / `keyedEventIds`
   - 不立即影响当前执行
   - 等待下一个需要 segment 评估的节点

#### 4.2.2 segmentUpdateSignal - Segment 变更通知

**用途**: 外部系统通知 segment 分配状态变更

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

#### 4.2.3 reEvaluateSegmentsSignal - 强制重新评估

**用途**: 强制工作流重新从数据库查询 segment 状态

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

### 4.3 Signal 发送方式

#### 4.3.1 SignalWithStart (原子操作)

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

#### 4.3.2 单独发送 Signal

```typescript
// 测试示例
await handle1.signal(trackSignal, {
  version: TrackSignalParamsVersion.V2,
  messageId: cancelledEvent.messageId,
});
```

用于向已存在的工作流发送信号。

### 4.4 长时运行节点的信号交互

#### 4.4.1 长时运行节点类型

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:200-204
const LONG_RUNNING_NODE_TYPES = new Set<JourneyNodeType>([
  JourneyNodeType.WaitForNode,
  JourneyNodeType.DelayNode,
  JourneyNodeType.SegmentEntryNode,
]);
```

#### 4.4.2 工作区状态检查

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:1100-1111
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

长时运行节点结束后，会检查工作空间状态：
- 如果工作空间不再 Active，立即退出旅程
- 用于处理工作空间被禁用/删除的情况

---

## 5. 三者协同机制总结

### 5.1 协同流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    外部事件 / Signal 入口                         │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 幂等性检查层 (Idempotency Layer)                              │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ 工作流 ID 生成 (uuidV5 + 业务键)                      │     │
│     │ signalWithStart + WorkflowExecutionAlreadyStarted   │     │
│     │ isRunnable 数据库检查                                 │     │
│     └─────────────────────────────────────────────────────┘     │
└──────────────────────┬──────────────────────────────────────────┘
                       │ 检查通过
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 工作流执行层 (Workflow Execution)                             │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ 节点循环 (nodeLoop)                                  │     │
│     │ - SegmentEntryNode: 等待 segment 条件               │     │
│     │ - DelayNode: 睡眠等待                                │     │
│     │ - WaitForNode: 多条件等待 + 超时                     │     │
│     │ - MessageNode: 消息发送 + 重试                      │     │
│     └─────────────────────────────────────────────────────┘     │
└──────────────────────┬──────────────────────────────────────────┘
                       │ Signal 到达
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. Signal 处理层 (Signal Handlers)                              │
│     ┌─────────────┬──────────────┬─────────────────────┐        │
│     │ trackSignal │segmentUpdate │ reEvaluateSegments  │        │
│     │ 事件追加    │ 版本控制更新  │ 强制重新评估         │        │
│     └──────┬──────┴──────┬───────┴──────────┬──────────┘        │
│            │             │                  │                   │
│            ▼             ▼                  ▼                   │
│     ┌─────────────────────────────────────────────────┐        │
│     │ 更新 segmentAssignments / keyedEvents           │        │
│     │ 触发 wf.condition 唤醒 (如果在等待节点)          │        │
│     └─────────────────────────────────────────────────┘        │
└──────────────────────┬──────────────────────────────────────────┘
                       │ 活动调用
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 活动重试层 (Activity Retry)                                  │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ Temporal 重试 (maximumAttempts)                     │     │
│     │ - 默认重试 (defaultUserJourneyMaxAttempts)          │     │
│     │ - 节点配置 (MessageNode.retryCount)                 │     │
│     │ - 心跳超时 (waitForComputeProperties)               │     │
│     ├─────────────────────────────────────────────────────┤     │
│     │ 应用级重试 (p-retry)                                │     │
│     │ - getEventsByIdWithRetry (ClickHouse 延迟)          │     │
│     └─────────────────────────────────────────────────────┘     │
└──────────────────────┬──────────────────────────────────────────┘
                       │ 失败处理
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. 失败决策层 (Failure Decision)                                │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ skipOnFailure: true  → 继续下一节点                  │     │
│     │ skipOnFailure: false → 退出节点                      │     │
│     │ canRunMultiple: true  → continueAsNew 重复进入       │     │
│     │ canRunMultiple: false → 永久结束                     │     │
│     └─────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 典型场景分析

#### 场景一: 重复订单事件 (幂等性)

1. **事件**: `order_created` 事件到达，`orderId: "ORD001"`
2. **幂等检查**:
   - 工作流 ID: `user-journey-keyed-{workspaceId}-{journeyId}-{uuidV5(userId, "orderId", "ORD001", workspaceId)}`
   - 第一次: `signalWithStart` 创建工作流
   - 重复事件: 捕获 `WorkflowExecutionAlreadyStartedError`，忽略
3. **工作流内部**:
   - `keyedEventIds.add("message-id-001")`
   - 再次收到相同 signal → `keyedEventIds.has()` → 忽略

#### 场景二: 消息发送失败重试

1. **节点**: MessageNode 发送邮件
2. **Temporal 重试**:
   - 配置: `maximumAttempts: 3`
   - 失败 1, 2 次 → Temporal 自动重试
   - 失败第 3 次 (last attempt) → 活动返回 `JourneyEarlyExit`
3. **决策**:
   - `skipOnFailure: true` → 继续下一个节点
   - `skipOnFailure: false` → 跳转到 ExitNode

#### 场景三: WaitFor 节点 + Segment Signal

1. **工作流状态**: 在 WaitForNode 等待用户进入 "paid" segment
2. **外部变更**: 用户升级为付费用户
3. **Signal 发送**:
   - `segmentUpdateSignal` 到达: `{segmentId: "paid", currentlyInSegment: true, segmentVersion: 12345}`
4. **Signal 处理**:
   - `segmentAssignments.set("paid", {currentlyInSegment: true, segmentVersion: 12345})`
5. **工作流唤醒**:
   - `wf.condition(() => segmentChildren.some((s) => segmentAssignedTrue(s.segmentId)))` 返回 true
   - 工作流继续执行对应分支

#### 场景四: 工作流 ContinueAsNew (循环旅程)

1. **旅程配置**: `canRunMultiple: true`
2. **工作流结束**: 执行完所有节点
3. **检查**:
   ```typescript
   if (await shouldReEnter({ journeyId, userId, workspaceId })) {
     if (shouldContinueAsNew) {
       await continueAsNew<typeof userJourneyWorkflow>(props);
     }
   }
   ```
4. **结果**:
   - 创建新的工作流 Run（保持相同的 Workflow ID）
   - 重置工作流历史，避免历史无限增长
   - 保持幂等性（同一时间只有一个运行实例）

---

## 6. 关键文件位置

| 功能 | 文件路径 | 关键函数/配置 |
|-----|---------|-------------|
| 工作流定义 | `packages/backend-lib/src/journeys/userWorkflow.ts` | `userJourneyWorkflow`, `getKeyedUserJourneyWorkflowId` |
| Signal 定义 | `packages/backend-lib/src/journeys/userWorkflow.ts` | `trackSignal`, `segmentUpdateSignal`, `reEvaluateSegmentsSignal` |
| 工作流生命周期 | `packages/backend-lib/src/journeys/userWorkflow/lifecycle.ts` | `startKeyedUserJourney`, `signalWithStart` |
| 活动实现 | `packages/backend-lib/src/journeys/userWorkflow/activities.ts` | `isRunnable`, `shouldReEnter`, `sendMessageV2` |
| 配置 | `packages/backend-lib/src/config.ts` | `defaultUserJourneyMaxAttempts`, `waitForComputePropertiesMaxAttempts` |
| 应用级重试 | `packages/backend-lib/src/journeys/userWorkflow/activities.ts` | `getEventsByIdWithRetry` |
| 工作流入站拦截器 | `packages/backend-lib/src/temporal/workflowInboundCallsInterceptor.ts` | `DittofeedWorkflowInboundInterceptor` |
| 重试工具函数 | `packages/backend-lib/src/retry.ts` | `retryExponential` |
| 测试 | `packages/backend-lib/src/journeys/keyedEventEntry.test.ts` | 键控事件测试 |
| 测试 | `packages/backend-lib/src/journeys/reEnter.test.ts` | 重复进入测试 |

---

## 7. 总结

Dittofeed 的 Journey 工作流通过三层机制保证可靠性：

1. **幂等性**:
   - 工作流 ID = 业务键的 UUID v5 哈希
   - `signalWithStart` + `WorkflowExecutionAlreadyStartedError`
   - `isRunnable` 数据库检查
   - `keyedEventIds` 去重

2. **重试策略**:
   - Temporal SDK 重试（可配置的 `maximumAttempts`）
   - 节点级配置（MessageNode.retryCount）
   - 应用级重试（p-retry 处理 ClickHouse 延迟）
   - 心跳超时检测

3. **Signal 互动**:
   - `trackSignal`: 追加事件，支持 WaitFor 节点实时评估
   - `segmentUpdateSignal`: 版本控制的 segment 变更通知
   - `reEvaluateSegmentsSignal`: 强制重新查询数据库
   - 通过 `segmentAssignments` + `wf.condition` 实现等待节点的唤醒

这三个机制协同工作，确保：
- **一致性**: 相同事件只处理一次
- **可靠性**: 失败时自动重试
- **响应性**: 外部变更能实时影响工作流执行
- **可观测性**: 完整的日志和错误追踪
