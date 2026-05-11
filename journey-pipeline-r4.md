# Journey 全链路深度分析报告 (R4)

## 核心关注点：信号证据纠错

本报告是 R3 的修正版，重点核实验证了三个关键问题：

1. **reEvaluateSegmentsSignal 发送路径** - 未发现生产发送路径
2. **segmentUpdate 发送入口** - 2 个生产发送入口（含 restartUserJourneyWorkflow）
3. **track 发送入口** - 1 个生产发送入口

---

## 1. 信号定义与 Handler 对应关系（准确版）

### 1.1 变量名 vs Temporal 注册名

**文件**: `packages/backend-lib/src/journeys/userWorkflow.ts:51-77`

```typescript
// 定义位置
export const segmentUpdateSignal =
  wf.defineSignal<[SegmentUpdate]>("segmentUpdate");   // 变量名: segmentUpdateSignal
                                                      // 注册名: "segmentUpdate"

export interface ReEvaluateSegmentsParams {
  segmentIds?: string[];
}

export const reEvaluateSegmentsSignal =
  wf.defineSignal<[ReEvaluateSegmentsParams]>("reEvaluateSegments");  // 变量名: reEvaluateSegmentsSignal
                                                                      // 注册名: "reEvaluateSegments"

export enum TrackSignalParamsVersion {
  V1 = 1,
  V2 = 2,
}

export type TrackSignalParamsV1 = UserWorkflowTrackEvent & {
  version?: TrackSignalParamsVersion.V1;
};

export interface TrackSignalParamsV2 {
  version: TrackSignalParamsVersion.V2;
  messageId: string;
}

export type TrackSignalParams = TrackSignalParamsV1 | TrackSignalParamsV2;

export const trackSignal = wf.defineSignal<[TrackSignalParams]>("track");  // 变量名: trackSignal
                                                                            // 注册名: "track"
```

### 1.2 Handler 对应关系表

| 变量名 | Temporal 注册名 | Handler 定义位置 | 用途 |
|--------|----------------|-----------------|------|
| `segmentUpdateSignal` | `"segmentUpdate"` | `packages/backend-lib/src/journeys/userWorkflow.ts:529-549` | 分段状态更新通知 |
| `trackSignal` | `"track"` | `packages/backend-lib/src/journeys/userWorkflow.ts:444-527` | Keyed Event 收集 |
| `reEvaluateSegmentsSignal` | `"reEvaluateSegments"` | `packages/backend-lib/src/journeys/userWorkflow.ts:551-599` | 强制重新评估分段 |

### 1.3 Handler 完整实现

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
  
  // 防旧检查：忽略版本号旧的更新
  if (prev && prev.segmentVersion >= update.segmentVersion) {
    logger.info("ignoring stale segment update", loggerAttrs);
    return;
  }

  logger.info("segment update", loggerAttrs);
  
  // 更新分段状态到共享 Map
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
  
  // 去重检查：忽略已处理的事件
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
      const propsVersion = props.version ?? UserJourneyWorkflowVersion.V1;
      if (propsVersion !== UserJourneyWorkflowVersion.V3) {
        if (!Array.isArray(keyedEvents)) {
          logger.error(
            "keyed events not set on a workflow version that expects it to be",
            {...},
          );
          return;
        }
        const newEvents = await getEventsById({
          workspaceId,
          eventIds: [event.messageId],
        });
        if (Array.isArray(keyedEvents)) {
          keyedEvents.push(...newEvents);
        }
      }
      keyedEventIds.add(event.messageId);
      break;
    }
    case TrackSignalParamsVersion.V1:
    default: {
      // V1: 直接使用信号中的事件数据
      if (keyedEvents) {
        keyedEvents.push(event);
      } else {
        logger.error("keyed events not set", {...});
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

---

## 2. 信号发送入口详细分析

### 2.1 reEvaluateSegmentsSignal: 未发现生产发送路径

**搜索结果**:
```
packages/backend-lib/src/journeys/userWorkflow.ts:58:export const reEvaluateSegmentsSignal =
packages/backend-lib/src/journeys/userWorkflow.ts:551:  wf.setHandler(reEvaluateSegmentsSignal, async (params) => {
```

**结论**:
- ✅ 定义存在 (userWorkflow.ts:58)
- ✅ Handler 存在 (userWorkflow.ts:551)
- ❌ **未发现任何生产代码中的发送调用**

**使用场景推断**:
- 该信号设计用于**手动触发**或**调试**
- 可能通过 Temporal CLI 手动发送：
  ```bash
  tctl workflow signal --workflow_id <wf-id> --name "reEvaluateSegments" --input '{"segmentIds": ["seg-1"]}'
  ```
- 测试代码中也未发现使用

### 2.2 segmentUpdateSignal: 2 个生产发送入口

#### 发送入口 1: triggerSegmentEntryJourney

**文件**: `packages/backend-lib/src/journeys.ts:875-891`

```typescript
// triggerSegmentEntryJourney 函数
// 触发时机: 用户进入/退出分段时
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
  signal: segmentUpdateSignal,  // 发送 "segmentUpdate" 信号
  signalArgs: [segmentUpdate],
});
```

**发送上下文**:
```typescript
// 前置检查 (journeys.ts:827-857)
if (journey.definition.entryNode.type !== JourneyNodeType.SegmentEntryNode) {
  return;  // 只处理 SegmentEntryNode 类型
}

if (journey.definition.entryNode.segment !== segmentId) {
  return;  // 只处理匹配的分段
}

// 构建信号参数
const segmentUpdate: SegmentUpdate = {
  segmentId,
  currentlyInSegment: Boolean(segmentAssignment.latest_segment_value),
  segmentVersion: new Date(segmentAssignment.max_assigned_at).getTime(),
  type: "segment",
};

if (!segmentUpdate.currentlyInSegment) {
  return;  // 只在进入分段时触发
}
```

**触发场景**:
- 分段计算完成后，用户进入该分段
- Journey 类型为 SegmentEntryNode
- 分段 ID 匹配

#### 发送入口 2: restartUserJourneysActivity

**文件**: `packages/backend-lib/src/restartUserJourneyWorkflow/activities.ts:104-120`

```typescript
// restartUserJourneysActivity 函数
// 触发时机: Journey 从 Paused 恢复为 Running 时

const segmentUpdate: SegmentUpdate = {
  type: "segment",
  segmentId,
  currentlyInSegment: true,
  segmentVersion: Date.now(),  // 使用当前时间作为版本号
};

const promises: Promise<unknown>[] = page.map(({ userId }) => {
  const workflowId = getUserJourneyWorkflowId({
    journeyId,
    userId,
  });
  return workflowClient.signalWithStart<
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
    signal: segmentUpdateSignal,  // 发送 "segmentUpdate" 信号
    signalArgs: [segmentUpdate],
  });
});
await Promise.all(promises);
```

**完整调用链**:

```
用户操作: 暂停的 Journey → 恢复运行
        ↓
upsertJourney() 状态检查 (journeys.ts:1183-1200)
        ↓
条件满足?
  - status === Running (新状态)
  - journey.status === Paused (旧状态)
  - definition.entryNode.type === SegmentEntryNode
  - definition.entryNode.reEnter === true
        ↓
restartUserJourneyWorkflow() (lifecycle.ts:11-45)
        ↓
workflowClient.start(restartUserJourneysWorkflow)
        ↓
restartUserJourneysWorkflow() (restartUserJourneyWorkflow.ts:23-33)
        ↓
restartUserJourneysActivity() (activities.ts:18-126)
        ↓
分批查询用户: findRecentlyUpdatedUsersInSegment()
        ↓
对每个用户: signalWithStart + segmentUpdateSignal
```

**触发条件** (journeys.ts:1183-1198):
```typescript
if (
  status === JourneyResourceStatusEnum.Running &&
  journey.status === JourneyResourceStatusEnum.Paused &&
  journeyDefinition?.entryNode.type === JourneyNodeType.SegmentEntryNode &&
  journeyDefinition.entryNode.reEnter
) {
  await restartUserJourneyWorkflow({
    journeyId: journey.id,
    workspaceId,
    statusUpdatedAt: priorityStatusUpdatedAt,
  });
}
```

### 2.3 trackSignal: 1 个生产发送入口

#### 发送入口: startKeyedUserJourney

**文件**: `packages/backend-lib/src/journeys/userWorkflow/lifecycle.ts:70-95`

```typescript
// startKeyedUserJourney 函数
// 触发时机: EventEntryNode 类型的 Journey 收到匹配事件时

await workflowClient.signalWithStart<
  typeof userJourneyWorkflow,
  [TrackSignalParams]
>(userJourneyWorkflow, {
  taskQueue: "default",
  workflowId,
  signal: trackSignal,  // 发送 "track" 信号
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

**完整调用链**:

```
事件到达: track() API
        ↓
triggerEventEntryJourneys() (journeys.ts)
        ↓
匹配缓存的 EventEntryNode 类型 Journey
        ↓
startKeyedUserJourney() (lifecycle.ts:22-109)
        ↓
检查: definition.entryNode.type === EventEntryNode
        ↓
生成 workflowId: user-journey-keyed-<hash>
        ↓
signalWithStart + trackSignal
```

**触发条件**:
- Journey 类型为 EventEntryNode
- 事件名称匹配 (`definition.entryNode.event === eventName`)
- 工作流 ID 唯一（防止重复触发）

---

## 3. 信号发送位置汇总表

### 3.1 生产代码发送位置

| 信号 | 变量名 | 注册名 | 发送位置 | 发送方式 | 触发时机 |
|------|--------|--------|---------|---------|---------|
| **segmentUpdateSignal** | `segmentUpdateSignal` | `"segmentUpdate"` | `packages/backend-lib/src/journeys.ts:889` | `signalWithStart` | 用户进入分段时触发 SegmentEntry Journey |
| **segmentUpdateSignal** | `segmentUpdateSignal` | `"segmentUpdate"` | `packages/backend-lib/src/restartUserJourneyWorkflow/activities.ts:118` | `signalWithStart` | Journey 从 Paused 恢复为 Running 且 reEnter=true |
| **trackSignal** | `trackSignal` | `"track"` | `packages/backend-lib/src/journeys/userWorkflow/lifecycle.ts:76` | `signalWithStart` | EventEntryNode 类型 Journey 收到匹配事件 |
| **reEvaluateSegmentsSignal** | `reEvaluateSegmentsSignal` | `"reEvaluateSegments"` | **未发现生产发送路径** | - | 设计用于手动触发/调试 |

### 3.2 测试代码发送位置（参考）

| 信号 | 测试文件 | 行号 | 发送方式 |
|------|---------|------|---------|
| `segmentUpdateSignal` | `waitForTest.test.ts` | 235 | `signalWithStart` |
| `segmentUpdateSignal` | `waitForTest.test.ts` | 268 | `signalWithStart` |
| `segmentUpdateSignal` | `waitForTest.test.ts` | 289 | `handle.signal()` |
| `segmentUpdateSignal` | `reEnter.test.ts` | 182, 212, 288, 318, 406, 466 | `signalWithStart` |
| `trackSignal` | `keyedEventEntry.test.ts` | 641, 1086, 1436 | `handle.signal()` |

---

## 4. isRunnable 调用分析（确认版）

### 4.1 调用点精确定位

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

### 4.2 调用时机上下文

```
userJourneyWorkflow(props)
    ↓
1. proxyActivities 初始化
   - config, isRunnable, onNodeProcessedV2, etc.
    ↓
2. 版本兼容处理 (V1/V2/V3)
   - V3: isHidden = props.hidden, eventKey = props.eventKey
   - V2: eventKey = 从 event.properties 解析
   - V1: eventKey = props.eventKey
    ↓
3. isRunnable() 检查 ← 唯一调用点
   - 工作空间 Active?
   - 用户已完成过此 Journey?
   - canRunMultiple?
   → 不可运行 → return null
    ↓
4. 初始化 keyedEventIds / keyedEvents
    ↓
5. EventEntry 验证
    ↓
6. 设置 Signal Handlers
    ↓
7. 节点循环 (nodeLoop)
```

### 4.3 isRunnable 函数实现

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

---

## 5. 关键代码索引（准确版）

### 5.1 信号定义与 Handler

| 信号 | 定义位置 | Handler 位置 | 注册名 |
|------|---------|-------------|--------|
| `segmentUpdateSignal` | userWorkflow.ts:51-52 | userWorkflow.ts:529-549 | `"segmentUpdate"` |
| `trackSignal` | userWorkflow.ts:77 | userWorkflow.ts:444-527 | `"track"` |
| `reEvaluateSegmentsSignal` | userWorkflow.ts:58-59 | userWorkflow.ts:551-599 | `"reEvaluateSegments"` |

### 5.2 信号发送位置（生产代码）

| 信号 | 发送文件 | 行号 | 发送函数 |
|------|---------|------|---------|
| `segmentUpdateSignal` | journeys.ts | 889 | `signalWithStart` |
| `segmentUpdateSignal` | restartUserJourneyWorkflow/activities.ts | 118 | `signalWithStart` |
| `trackSignal` | journeys/userWorkflow/lifecycle.ts | 76 | `signalWithStart` |
| `reEvaluateSegmentsSignal` | **未发现** | - | - |

### 5.3 restartUserJourneyWorkflow 调用链

| 步骤 | 文件 | 行号 | 描述 |
|------|------|------|------|
| 1 | journeys.ts | 1183-1200 | 状态变更检测（Paused → Running） |
| 2 | journeys.ts | 1194 | 调用 restartUserJourneyWorkflow() |
| 3 | restartUserJourneyWorkflow/lifecycle.ts | 11-45 | 启动 restartUserJourneysWorkflow |
| 4 | restartUserJourneyWorkflow.ts | 23-33 | 工作流定义 |
| 5 | restartUserJourneyWorkflow/activities.ts | 18-126 | 分批查询用户并发送 segmentUpdateSignal |

---

## 6. 修正总结

### 6.1 reEvaluateSegmentsSignal 修正

**R3 错误陈述**:
- "reEvaluateSegmentsSignal 发送: 测试代码中使用，无生产代码直接发送"

**实际情况**:
- ✅ **未发现生产发送路径**
- 仅定义了信号和 Handler
- 设计用于手动触发（Temporal CLI）或调试场景

### 6.2 segmentUpdateSignal 发送入口修正

**R3 错误陈述**:
- "segmentUpdateSignal 发送: journeys.ts:889 - SegmentEntry 触发"

**实际情况**:
- ✅ 发送入口 1: `journeys.ts:889` - 用户进入分段时
- ✅ 发送入口 2: `restartUserJourneyWorkflow/activities.ts:118` - Journey 从 Paused 恢复时（reEnter=true）

### 6.3 trackSignal 发送入口确认

**实际情况**:
- ✅ 发送入口: `journeys/userWorkflow/lifecycle.ts:76` - EventEntry 触发
- ✅ 发送方式: `signalWithStart` 同时启动工作流并发送信号

### 6.4 发送位置表与正文一致性

本报告中所有发送位置表均已与代码实际位置一一对应：
- `segmentUpdateSignal` 的 2 个发送入口在正文和表格中一致
- `trackSignal` 的 1 个发送入口在正文和表格中一致
- `reEvaluateSegmentsSignal` 明确标记为"未发现生产发送路径"
