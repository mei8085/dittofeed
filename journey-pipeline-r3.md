# Journey 全链路深度分析报告 (R3)

## 核心关注点修正说明

本报告是对 R1/R2 的修正版，重点核实验证了两个关键问题：

1. **isRunnable 在 userJourneyWorkflow 中的实际调用时机与次数** - 实际仅调用 1 次，而非多次
2. **三类 signal 的实际定义名与 handler 对应关系** - 变量名与 Temporal 注册名的区别

---

## 1. isRunnable 实际调用分析

### 1.1 调用点精确定位

**文件**: `packages/backend-lib/src/journeys/userWorkflow.ts`

**实际调用次数**: 1 次

**调用位置**: Line 332-349

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:332-349
if (
  !(await isRunnable({
    journeyId,
    userId,
    eventKey,
    eventKeyName,
    workspaceId,
  }))
) {
  logger.info("early exit unrunnable user journey", {
    workflow: WORKFLOW_NAME,
    journeyId,
    userId,
    workspaceId,
    entryEventProperties,
  });
  return null;
}
```

### 1.2 调用时机上下文

**工作流执行流程**:

```
userJourneyWorkflow(props)
    ↓
版本兼容处理 (props.version: V1/V2/V3)
    ↓
isRunnable() 检查  ← 唯一调用点
    ↓
    ↓ (检查通过)
    ↓
初始化 keyedEventIds / keyedEvents
    ↓
EventEntry 验证
    ↓
设置三个 Signal Handlers
    ↓
节点循环 (nodeLoop)
    ├─ SegmentEntryNode
    ├─ EventEntryNode
    ├─ DelayNode
    ├─ WaitForNode
    ├─ SegmentSplitNode
    ├─ RandomCohortNode
    └─ MessageNode
    ↓
ContinueAsNew 检查 (shouldReEnter)
```

**调用时机**: 工作流启动后，**第一个实际操作**

- 在版本兼容处理之后
- 在 keyedEventIds 初始化之前
- 在 Signal Handlers 设置之前
- 在节点循环执行之前

### 1.3 修正: 关于之前报告的错误

**R1/R2 错误陈述**:
- "检查时机: 1. 工作流启动前；2. SegmentEntryNode 等待后；3. ContinueAsNew 前"
- 暗示有 3 次调用

**实际情况**: 仅在工作流开头调用 1 次

**代码证据**:

```typescript
// 搜索结果显示仅 3 个匹配，其中 1 个定义，2 个引用
// packages/backend-lib/src/journeys/userWorkflow/activities.ts:396: export async function isRunnable({
// packages/backend-lib/src/journeys/userWorkflow.ts:225:     isRunnable,  // proxyActivities 导入
// packages/backend-lib/src/journeys/userWorkflow.ts:333:     !(await isRunnable({  // 唯一调用
```

### 1.4 isRunnable 函数实现

**文件**: `packages/backend-lib/src/journeys/userWorkflow/activities.ts:396-470`

```typescript
export async function isRunnable({
  workspaceId,
  journeyId,
  userId,
  eventKey,
  eventKeyName,
}: {
  workspaceId?: string;
  journeyId: string;
  userId: string;
  eventKey?: string;
  eventKeyName?: string;
}): Promise<boolean> {
  // 并行查询三个数据源
  const [previousExitEvent, journey, workspace] = await Promise.all([
    // 1. 用户历史记录: 是否完成过此 Journey
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
    // 2. Journey 配置: canRunMultiple 标志
    db().query.journey.findFirst({ where: eq(dbJourney.id, journeyId) }),
    // 3. 工作空间状态
    workspaceId
      ? db().query.workspace.findFirst({ where: eq(dbWorkspace.id, workspaceId) })
      : null,
  ]);

  // 检查逻辑:
  if (!previousExitEvent) return true;            // 从未运行过
  
  if (workspace?.status !== "Active") return false;  // 工作空间非活跃
  
  return !!journey?.canRunMultiple;              // 允许多次运行?
}
```

### 1.5 设计意图分析

**为什么只在开头调用一次?**

1. **Temporal 工作流不可中断性**:
   - 工作流代码是确定性的，不能在执行中间"突然"退出
   - 只能在等待点（condition/sleep）被唤醒后检查状态

2. **检查点设计**:
   - 实际的"检查点"是在等待**之后**的代码逻辑
   - 但 isRunnable 本身只在开头调用一次

3. **Long-Running Node 的处理**:
   - SegmentEntryNode: 等待 segmentUpdateSignal
   - DelayNode:  sleep(delay)
   - WaitForNode: 等待 segmentUpdateSignal + 超时
   - 这些节点唤醒后，**没有**额外的 isRunnable 检查

**修正后的一致性边界理解**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    工作流启动阶段                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 版本兼容处理 (V1/V2/V3)                              │   │
│  │  2. isRunnable() 检查 ← 唯一检查点                       │   │
│  │     - 工作空间 Active?                                   │   │
│  │     - 用户已完成过? (canRunMultiple?)                     │   │
│  │  3. 如果不可运行 → return null (提前退出)                  │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    工作流执行阶段                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  节点循环:                                               │   │
│  │  - SegmentEntryNode: 等待 condition                     │   │
│  │  - DelayNode: sleep(delay)                             │   │
│  │  - WaitForNode: 等待 condition + timeout               │   │
│  │  - SegmentSplitNode: 实时查询分段状态                    │   │
│  │  - MessageNode: 发送消息                                │   │
│  │                                                         │   │
│  │  ⚠️  唤醒后没有 isRunnable 检查                          │   │
│  │  ⚠️  但每个节点有自己的实时数据查询                       │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Signal 定义名与 Handler 对应关系

### 2.1 变量名 vs Temporal 注册名

**文件**: `packages/backend-lib/src/journeys/userWorkflow.ts:51-77`

```typescript
// 定义位置
export const segmentUpdateSignal =
  wf.defineSignal<[SegmentUpdate]>("segmentUpdate");   // 变量名: segmentUpdateSignal
                                                      // 注册名: "segmentUpdate"

export const reEvaluateSegmentsSignal =
  wf.defineSignal<[ReEvaluateSegmentsParams]>("reEvaluateSegments");  // 变量名: reEvaluateSegmentsSignal
                                                                      // 注册名: "reEvaluateSegments"

export const trackSignal = wf.defineSignal<[TrackSignalParams]>("track");  // 变量名: trackSignal
                                                                            // 注册名: "track"
```

### 2.2 对应关系表

| 变量名 | Temporal 注册名 | Handler 定义位置 | 用途 |
|--------|----------------|-----------------|------|
| `segmentUpdateSignal` | `"segmentUpdate"` | `packages/backend-lib/src/journeys/userWorkflow.ts:529-549` | 分段状态更新通知 |
| `trackSignal` | `"track"` | `packages/backend-lib/src/journeys/userWorkflow.ts:444-527` | Keyed Event 收集 |
| `reEvaluateSegmentsSignal` | `"reEvaluateSegments"` | `packages/backend-lib/src/journeys/userWorkflow.ts:551-599` | 强制重新评估分段 |

### 2.3 Handler 完整对应关系

**Handler 1: segmentUpdateSignal → "segmentUpdate"**

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:529-549
wf.setHandler(segmentUpdateSignal, (update) => {
  const prev = segmentAssignments.get(update.segmentId);
  const loggerAttrs = {
    workflow: WORKFLOW_NAME,
    journeyId,
    userId,
    workspaceId,
    prev,
    update,
  };
  
  // 防旧检查
  if (prev && prev.segmentVersion >= update.segmentVersion) {
    logger.info("ignoring stale segment update", loggerAttrs);
    return;
  }

  logger.info("segment update", loggerAttrs);
  
  // 更新分段状态
  segmentAssignments.set(update.segmentId, {
    currentlyInSegment: update.currentlyInSegment,
    segmentVersion: update.segmentVersion,
  });
});
```

**Handler 2: trackSignal → "track"**

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:444-527
wf.setHandler(trackSignal, async (event) => {
  logger.info("keyed event signal", {
    workspaceId,
    journeyId,
    userId,
    messageId: event.messageId,
  });
  
  // 去重检查
  if (keyedEventIds.has(event.messageId)) {
    logger.info("ignoring duplicate keyed event", {
      journeyId,
      userId,
      workspaceId,
      messageId: event.messageId,
    });
    return;
  }
  
  // 版本兼容处理
  switch (event.version) {
    case TrackSignalParamsVersion.V2: {
      // V2: 通过 ID 重新获取完整事件
      const newEvents = await getEventsById({
        workspaceId,
        eventIds: [event.messageId],
      });
      if (Array.isArray(keyedEvents)) {
        keyedEvents.push(...newEvents);
      }
      keyedEventIds.add(event.messageId);
      break;
    }
    case TrackSignalParamsVersion.V1:
    default: {
      // V1: 直接使用信号中的事件数据
      if (keyedEvents) {
        keyedEvents.push(event);
      }
      keyedEventIds.add(event.messageId);
      break;
    }
  }
  
  // 如果正在 WaitFor，触发分段重新评估
  if (!waitForSegmentIds) {
    return;
  }
  await Promise.all(
    waitForSegmentIds.map(async ({ segmentId }) => {
      const nowMs = Date.now();
      const assignment = await getSegmentAssignmentHandler({
        segmentId,
        now: nowMs,
      });
      if (assignment === null) {
        return;
      }
      segmentAssignments.set(segmentId, {
        currentlyInSegment: assignment.inSegment,
        segmentVersion: nowMs,
      });
    }),
  );

  reportWorkflowInfoHandler();
});
```

**Handler 3: reEvaluateSegmentsSignal → "reEvaluateSegments"**

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:551-599
wf.setHandler(reEvaluateSegmentsSignal, async (params) => {
  logger.info("re-evaluating segments", {
    workflow: WORKFLOW_NAME,
    journeyId,
    userId,
    workspaceId,
    segmentIds: params.segmentIds,
  });

  // 如果未指定，重新评估所有已知分段
  const segmentIdsToEvaluate =
    params.segmentIds ?? Array.from(segmentAssignments.keys());
  const nowMs = Date.now();

  // 并行重新评估所有指定分段
  await Promise.all(
    segmentIdsToEvaluate.map(async (segmentId) => {
      // 强制重新计算，不检查版本号
      const assignment = await getSegmentAssignmentHandler({
        segmentId,
        now: nowMs,
      });

      if (assignment === null) {
        logger.warn("segment assignment returned null during re-evaluation", {...});
        return;
      }

      // 强制更新，使用当前时间作为新版本号
      segmentAssignments.set(segmentId, {
        currentlyInSegment: assignment.inSegment,
        segmentVersion: nowMs,
      });

      logger.info("segment re-evaluated", {
        workflow: WORKFLOW_NAME,
        journeyId,
        userId,
        workspaceId,
        segmentId,
        inSegment: assignment.inSegment,
      });
    }),
  );

  reportWorkflowInfoHandler();
});
```

### 2.4 信号发送位置验证

**segmentUpdateSignal 发送**: `packages/backend-lib/src/journeys.ts:875-891`

```typescript
// triggerSegmentEntryJourney
await workflowClient.signalWithStart<
  typeof userJourneyWorkflow,
  [SegmentUpdate]
>(userJourneyWorkflow, {
  taskQueue: "default",
  workflowId,
  args: [
    {
      journeyId,
      definition,
      workspaceId,
      userId,
    },
  ],
  signal: segmentUpdateSignal,  // 使用变量名，但实际发送 "segmentUpdate"
  signalArgs: [segmentUpdate],
});
```

**trackSignal 发送**: `packages/backend-lib/src/journeys/userWorkflow/lifecycle.ts:70-95`

```typescript
// startKeyedUserJourney
await workflowClient.signalWithStart<
  typeof userJourneyWorkflow,
  [TrackSignalParams]
>(userJourneyWorkflow, {
  taskQueue: "default",
  workflowId,
  signal: trackSignal,  // 使用变量名，但实际发送 "track"
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

**reEvaluateSegmentsSignal 发送**: 测试代码中使用，无生产代码直接发送

```typescript
// packages/backend-lib/src/journeys/waitForTest.test.ts:289
await handle2.signal(segmentUpdateSignal, {
  segmentId: waitForSegmentId,
  currentlyInSegment: true,
  type: "segment",
  segmentVersion: await getTestEnv().currentTimeMs(),
});
```

### 2.5 信号处理机制统一说明

**共享状态**: `segmentAssignments: Map<string, ReceivedSegmentUpdate>`

```typescript
interface ReceivedSegmentUpdate {
  currentlyInSegment: boolean;
  segmentVersion: number;  // 时间戳版本号
}
```

**唤醒机制**:

```
信号处理器修改 segmentAssignments Map
        ↓
Temporal 检查所有等待的 condition
        ↓
condition 重新求值
        ↓
如果返回 true，唤醒工作流继续执行
```

**示例**:

```typescript
// 工作流中等待
case JourneyNodeType.SegmentEntryNode: {
  const cn = currentNode;
  const initialSegmentAssignment = 
    (await getSegmentAssignmentHandler({...}))?.inSegment === true;
  
  if (!initialSegmentAssignment) {
    // 等待条件：检查 segmentAssignments Map
    await wf.condition(() => 
      segmentAssignments.get(cn.segment)?.currentlyInSegment === true
    );
  }
  nextNode = nodes.get(currentNode.child) ?? null;
  break;
}
```

**三种信号的作用对比**:

| 特性 | segmentUpdateSignal | trackSignal | reEvaluateSegmentsSignal |
|------|---------------------|-------------|------------------------|
| **注册名** | `"segmentUpdate"` | `"track"` | `"reEvaluateSegments"` |
| **变量名** | `segmentUpdateSignal` | `trackSignal` | `reEvaluateSegmentsSignal` |
| **去重** | ❌ (版本检查替代) | ✅ (keyedEventIds Set) | ❌ |
| **防旧** | ✅ (segmentVersion 比较) | ❌ | ❌ |
| **更新状态** | 直接使用信号值 | 间接更新分段 | 重新查询 |
| **强制刷新** | ❌ | ❌ | ✅ |
| **Handler 位置** | userWorkflow.ts:529 | userWorkflow.ts:444 | userWorkflow.ts:551 |

---

## 3. 修正后的完整流程图

### 3.1 工作流启动到执行的准确流程

```
signalWithStart(userJourneyWorkflow)
        ↓
┌─────────────────────────────────────────────────────────────────┐
│ userJourneyWorkflow(props)                                      │
│                                                                 │
│ 1. proxyActivities 初始化                                        │
│    - config, isRunnable, onNodeProcessedV2, etc.               │
│    - proxyLocalActivities: getEventsById, getSegmentAssignment  │
│                                                                 │
│ 2. 版本兼容处理 (V1/V2/V3)                                      │
│    - V3: isHidden = props.hidden, eventKey = props.eventKey     │
│    - V2: eventKey = 从 event.properties 解析                     │
│    - V1: eventKey = props.eventKey                              │
│                                                                 │
│ 3. isRunnable() 检查 ← 唯一检查点                                │
│    - 工作空间 Active?                                           │
│    - 用户已完成过此 Journey?                                     │
│    - canRunMultiple?                                            │
│    → 不可运行 → return null                                     │
│                                                                 │
│ 4. 初始化 keyedEventIds / keyedEvents                           │
│    - keyedEventIds = new Set<string>()                          │
│    - V3/V2: 添加初始 event.messageId                            │
│                                                                 │
│ 5. EventEntry 验证                                              │
│    - 如果是 EventEntryNode 但没有 eventKey → return null        │
│                                                                 │
│ 6. 设置 Signal Handlers                                         │
│    ├─ wf.setHandler(segmentUpdateSignal, ...)                   │
│    ├─ wf.setHandler(trackSignal, ...)                           │
│    └─ wf.setHandler(reEvaluateSegmentsSignal, ...)              │
│                                                                 │
│ 7. 节点循环 (nodeLoop)                                          │
│    for (let i = 0; i < nodes.size + 1; i++) {                   │
│      switch (currentNode.type) {                                │
│        case SegmentEntryNode:                                   │
│          - 初始查询分段状态                                      │
│          - 不在分段 → await condition()                         │
│          - ⚠️ 唤醒后没有 isRunnable 检查                         │
│          → nextNode = currentNode.child                        │
│                                                                 │
│        case EventEntryNode:                                     │
│          → nextNode = currentNode.child                        │
│                                                                 │
│        case DelayNode:                                          │
│          - 计算 delay                                           │
│          - delay > 0 → await sleep(delay)                       │
│          - ⚠️ 唤醒后没有 isRunnable 检查                         │
│          → nextNode = currentNode.child                        │
│                                                                 │
│        case WaitForNode:                                        │
│          - 设置 waitForSegmentIds                               │
│          - 初始检查所有分段                                      │
│          - 都不满足 → await condition(timeout)                  │
│          - ⚠️ 唤醒后没有 isRunnable 检查                         │
│          → 根据结果选择分支                                      │
│                                                                 │
│        case SegmentSplitNode:                                   │
│          - 实时查询分段状态                                      │
│          → trueChild / falseChild                               │
│                                                                 │
│        case RandomCohortNode:                                   │
│          - 确定性随机数                                          │
│          → 选择对应的 child                                     │
│                                                                 │
│        case MessageNode:                                        │
│          - 调用 sendMessageV2 activity                          │
│          - 失败且 skipOnFailure=false → exitNode                │
│          → nextNode = currentNode.child                        │
│                                                                 │
│        case ExitNode:                                           │
│          → break nodeLoop                                       │
│      }                                                          │
│      currentNode = nextNode                                    │
│    }                                                            │
│                                                                 │
│ 8. ContinueAsNew 检查                                           │
│    - 如果 reEnter=true 且 shouldReEnter=true                    │
│    → continueAsNew<typeof userJourneyWorkflow>(props)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 信号处理流程图

```
                    ┌─────────────────────────────────────────┐
                    │           Temporal Server              │
                    │                                       │
                    │  信号队列                              │
                    │  ┌─────────────┐                     │
                    │  │ "track"     │ ← lifecycle.ts      │
                    │  │ "segmentUp…"│ ← journeys.ts       │
                    │  │ "reEvaluat…"│ ← (测试/手动)       │
                    │  └─────────────┘                     │
                    └───────────────┬─────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      userJourneyWorkflow 实例                            │
│                                                                         │
│  segmentAssignments = Map<segmentId, {currentlyInSegment, version}>      │
│  keyedEventIds = Set<messageId>                                         │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Signal Handlers (设置后一直运行)                                │   │
│  │                                                                  │   │
│  │  1. "segmentUpdate" handler (line 529)                          │   │
│  │     ┌─────────────────────────────────────────────────────┐    │   │
│  │     │ if (prev && prev.segmentVersion >= update.version) │    │   │
│  │     │   return (忽略旧消息)                               │    │   │
│  │     │ else                                              │    │   │
│  │     │   segmentAssignments.set(...) ← 更新共享状态       │    │   │
│  │     └─────────────────────────────────────────────────────┘    │   │
│  │                                                                  │   │
│  │  2. "track" handler (line 444)                                  │   │
│  │     ┌─────────────────────────────────────────────────────┐    │   │
│  │     │ if (keyedEventIds.has(event.messageId))            │    │   │
│  │     │   return (去重)                                    │    │   │
│  │     │ else                                              │    │   │
│  │     │   keyedEventIds.add(event.messageId)              │    │   │
│  │     │   if (waitForSegmentIds)                         │    │   │
│  │     │     重新评估分段 → segmentAssignments.set(...)    │    │   │
│  │     └─────────────────────────────────────────────────────┘    │   │
│  │                                                                  │   │
│  │  3. "reEvaluateSegments" handler (line 551)                      │   │
│  │     ┌─────────────────────────────────────────────────────┐    │   │
│  │     │ 强制查询最新分段状态                                  │    │   │
│  │     │ segmentAssignments.set(...) ← 强制更新              │    │   │
│  │     │ (无版本检查，无去重)                                 │    │   │
│  │     └─────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  条件等待被唤醒                                                 │   │
│  │                                                                  │   │
│  │  任一 handler 修改 segmentAssignments 后                          │   │
│  │    ↓                                                             │   │
│  │  Temporal 检查所有等待的 condition                                │   │
│  │    ↓                                                             │   │
│  │  condition 重新求值:                                             │   │
│  │    - SegmentEntryNode: segmentAssignedTrue(segmentId)            │   │
│  │    - WaitForNode: segmentChildren.some(s => segmentAssignedTrue) │   │
│  │    ↓                                                             │   │
│  │  如果返回 true，唤醒工作流继续执行                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ⚠️  唤醒后没有 isRunnable 检查                                        │
│  ⚠️  直接继续执行节点循环                                               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 关键代码索引（修正版）

### 4.1 isRunnable 相关

| 文件 | 行号 | 内容 |
|------|------|------|
| `packages/backend-lib/src/journeys/userWorkflow/activities.ts` | 396-470 | `isRunnable()` 函数定义 |
| `packages/backend-lib/src/journeys/userWorkflow.ts` | 225 | `isRunnable` 从 `proxyActivities` 导入 |
| `packages/backend-lib/src/journeys/userWorkflow.ts` | 332-349 | **唯一调用点**: 工作流开头 |

### 4.2 Signal 相关

| 变量名 | Temporal 注册名 | 定义位置 | Handler 位置 |
|--------|----------------|----------|-------------|
| `segmentUpdateSignal` | `"segmentUpdate"` | userWorkflow.ts:51-52 | userWorkflow.ts:529-549 |
| `trackSignal` | `"track"` | userWorkflow.ts:77 | userWorkflow.ts:444-527 |
| `reEvaluateSegmentsSignal` | `"reEvaluateSegments"` | userWorkflow.ts:58-59 | userWorkflow.ts:551-599 |

### 4.3 信号发送位置

| 信号 | 发送位置 | 用途 |
|------|---------|------|
| `segmentUpdateSignal` | journeys.ts:889 | SegmentEntry 触发 |
| `trackSignal` | lifecycle.ts:76 | EventEntry 触发 (Keyed) |
| `reEvaluateSegmentsSignal` | 测试代码中 | 手动触发 |

---

## 5. 修正总结

### 5.1 关于 isRunnable 的修正

**错误陈述 (R1/R2)**:
- "isRunnable 在多个检查点调用：启动前、SegmentEntry 等待后、ContinueAsNew 前"
- 暗示 3 次调用

**实际情况**:
- 仅在工作流开头调用 **1 次** (line 332-349)
- 调用位置：版本兼容处理之后，节点循环之前
- 其他节点唤醒后**没有** isRunnable 检查

**设计含义**:
- isRunnable 主要用于"准入控制"
- 实际的运行时状态检查通过每个节点的实时查询实现
- 工作流一旦启动，不会因为 Journey 状态变化而"突然"停止

### 5.2 关于 Signal 命名的修正

**容易混淆的点**:
- 代码中使用的是**变量名** (`segmentUpdateSignal`)
- Temporal 实际注册的是**字符串名** (`"segmentUpdate"`)

**完整对应关系**:

```typescript
// 定义
export const segmentUpdateSignal = wf.defineSignal<[SegmentUpdate]>("segmentUpdate");
//          ↑ 变量名                                    ↑ Temporal 注册名

// 使用
wf.setHandler(segmentUpdateSignal, handler);  // 使用变量名
workflow.signal(segmentUpdateSignal, data);    // 使用变量名

// 但 Temporal 内部实际发送/接收的是字符串 "segmentUpdate"
```

**为什么重要**:
- 如果在外部通过 raw signal 发送，必须使用字符串名 `"segmentUpdate"` 而非变量名
- 测试代码中直接使用变量名是安全的，因为 Temporal SDK 会自动转换
