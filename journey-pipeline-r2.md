# Journey 全链路深度分析报告 (R2)

## 1. 画布 State 到 Definition 的完整转换

### 1.1 前端画布数据模型

**State 结构** (packages/dashboard/src/components/journeys/store.ts:541-584):

```typescript
interface JourneyContent {
  journeyNodes: JourneyUiNode[];           // UI 节点数组
  journeyEdges: JourneyUiEdge[];           // 边数组
  journeyNodesIndex: Map<string, number>;  // 节点索引
  journeySelectedNodeId: string | null;    // 当前选中节点
  journeyName: string;                     // Journey 名称
  journeyDraggedComponentType: null | ...  // 拖拽中的组件类型
}
```

**节点类型** (JourneyUiNode):
- **Journey 节点** (`type: "journey"`): 真正的业务节点
  - `data.type = JourneyUiNodeType.JourneyUiNodeDefinitionProps`
  - 包含 `nodeTypeProps`: 具体节点配置
  
- **Label 节点** (`type: "label"`): 分支标签展示
  - `data.type = JourneyUiNodeType.JourneyUiNodeLabelProps`
  - 如 `"true"`, `"false"`, `"In segment"`
  
- **Empty 节点** (`type: "empty"`): 分支汇合点
  - `data.type = JourneyUiNodeType.JourneyUiNodeEmptyProps`
  - 用于分支路径的重新汇合

**边类型** (JourneyUiEdge):
- **Workflow 边** (`type: "workflow"`): 执行流
  - `data.type = JourneyUiEdgeType.JourneyUiDefinitionEdgeProps`
  
- **Placeholder 边** (`type: "placeholder"`): 分支连接
  - `data.type = JourneyUiEdgeType.JourneyUiPlaceholderEdgeProps`
  - 从分支节点到 label 节点

### 1.2 HeritageMap: 节点关系图

**构建函数**: `buildUiHeritageMap()` (store.ts:127-192)

```typescript
type HeritageMap = Map<string, {
  children: Set<string>;      // 直接子节点
  descendants: Set<string>;   // 所有后代节点
  parents: Set<string>;       // 直接父节点
  ancestors: Set<string>;     // 所有祖先节点
}>;
```

**构建算法**:
1. **初始化**: 为每个节点创建空的集合
2. **BFS 遍历**: 从每个节点出发，遍历所有可达节点
3. **填充关系**:
   - `children`: 直接连接的子节点
   - `parents`: 直接连接的父节点
   - `descendants`: 所有可达子节点（递归）
   - `ancestors`: 所有可达父节点（递归）

**示例**:
```
    A (SegmentSplit)
   / \
 true false  (Label 节点)
  |    |
  B    C   (Journey 节点)
   \  /
    D     (Empty 节点)
     |
     E

A 的 children: {true, false}
A 的 descendants: {true, false, B, C, D, E}
true 的 parents: {A}
true 的 ancestors: {A}
B 的 children: {D}
B 的 ancestors: {A, true}
```

### 1.3 分支节点的 child 关系构建

**关键函数**: `getNearestJourneyFromChildren()` (store.ts:225-259)

找到所有分支路径的**最近公共后代节点**（汇合点）：

```typescript
function getNearestJourneyFromChildren(
  nId: string,           // 分支节点 ID
  hm: HeritageMap,       // 关系图
  uiJourneyNodes: Map<...>  // UI 节点映射
): string {
  // 找到所有子路径的后代集合的交集
  // 返回 ancestor 数量最少的那个（最近的）
}
```

**算法原理**:
1. 获取分支节点的所有直接子节点
2. 遍历每个子节点的所有后代
3. 筛选出被**所有分支**共享的后代
4. 选择 `ancestors.size` 最小的（最近的）

### 1.4 State → Definition 转换

**核心函数**: `journeyDefinitionFromStateBranch()` (store.ts:693-1065)

这是一个递归函数，处理线性和分支两种情况：

#### 线性节点转换 (MessageNode, DelayNode, EntryNode)

```
当前节点 → 找下一个 Journey 节点（跳过 label/empty）→ 构建 child 关系

示例:
Entry → Delay → Message → Exit

转换后:
entryNode: { type: SegmentEntryNode, child: delayNode.id }
delayNode: { type: DelayNode, child: messageNode.id }
messageNode: { type: MessageNode, child: exitNode.id }
```

#### 分支节点转换 (SegmentSplitNode, WaitForNode, RandomCohortNode)

**步骤**:
1. **找到最近公共后代 (nfc)**: 所有分支的汇合点
2. **处理每个分支**:
   - 找到该分支的第一个 Journey 节点
   - 如果该节点不是 nfc（说明分支内有自己的子路径），递归调用
   - 递归终止条件: `terminateBefore = nfc`
3. **构建 Definition 节点**:
   - `trueChild` / `falseChild` (SegmentSplit)
   - `segmentChildren` / `timeoutChild` (WaitFor)
   - `children` (RandomCohort)

**SegmentSplit 示例**:

```
UI 结构:
    A (SegmentSplit)
   / \
 true false
  |    |
  B    C
   \  /
    D (Empty)
     |
     E

Definition 结构:
A: {
  type: SegmentSplitNode,
  variant: {
    trueChild: B.id,
    falseChild: C.id
  }
}
B: { type: MessageNode, child: E.id }   // 跳过 D，直接指向 E
C: { type: DelayNode, child: E.id }     // 跳过 D，直接指向 E
```

**递归调用过程**:
1. 处理 A，找到 nfc = E
2. 处理 true 分支，发现 B ≠ E
3. 递归处理 B，terminateBefore = E
4. B 的下一个节点是 D (Empty)，跳过
5. B 的下一个节点是 E，等于 terminateBefore，停止
6. B.child = E.id
7. 同理处理 false 分支

### 1.5 Definition → State 反向转换

**函数**: `journeyBranchToState()` (store.ts:1533-1939)

用于从数据库加载 Definition 后恢复画布状态：

1. **线性节点**: 直接创建 Journey 节点 + Workflow 边
2. **分支节点**:
   - 创建 Journey 节点
   - 为每个分支创建 Label 节点
   - 创建 Placeholder 边（分支节点 → Label）
   - 创建 Empty 节点（汇合点）
   - 递归处理每个分支的子路径
   - 创建 Workflow 边连接到汇合点

## 2. Draft 与 Definition 发布机制

### 2.1 数据模型对比

| 属性 | Draft | Definition |
|------|-------|------------|
| **结构** | UI 节点 + 边 | 执行节点 + child 关系 |
| **存储形式** | `{ nodes: [], edges: [] }` | `{ entryNode, nodes, exitNode }` |
| **节点类型** | Journey/Label/Empty | Entry/Body/Exit (业务节点) |
| **执行性** | ❌ 不可直接执行 | ✅ 可直接执行 |
| **编辑状态** | ✅ 用于未发布的编辑 | ❌ 已发布版本 |

### 2.2 状态流转

```
[创建 Journey]
      ↓
[NotStarted 状态]
      ↓
    draft = null
    definition = 初始结构
      ↓
[用户编辑画布]
      ↓
[自动保存]
      ↓
    draft = UI 状态快照
    definition 保持不变
      ↓
[点击发布]
      ↓
    draft → 转换为 definition
    draft = null (清除)
    status = Running
```

### 2.3 关键转换函数

**State → Draft**: `journeyStateToDraft()` (store.ts:1983-2000)

```typescript
export function journeyStateToDraft(
  state: JourneyStateForDraft
): JourneyDraft {
  return {
    nodes: state.journeyNodes.map((n) => ({
      id: n.id,
      data: n.data,  // 保留所有 UI 数据
    })),
    edges: state.journeyEdges.map((e) => ({
      source: e.source,
      target: e.target,
      data: e.data,   // 保留边类型信息
    })),
  };
}
```

**Draft → State**: `journeyDraftToState()` (store.ts:2205-2263)

用于从数据库加载 draft 恢复画布状态。

**State → Definition**: `journeyDefinitionFromState()` (store.ts:1067-1117)

```typescript
export function journeyDefinitionFromState({
  state,
}: {
  state: Omit<JourneyStateForResource, "journeyName">;
}): Result<JourneyDefinition, { message: string; nodeId: string }> {
  const nodes: JourneyNode[] = [];
  const journeyNodes = buildJourneyNodeMap(state.journeyNodes);
  const hm = buildUiHeritageMap(state.journeyNodes, state.journeyEdges);

  // 从 Entry 节点开始递归转换
  const result = journeyDefinitionFromStateBranch(
    AdditionalJourneyNodeType.EntryUiNode,
    hm,
    nodes,
    journeyNodes,
    state.journeyEdges,
  );

  // 分离 entryNode, exitNode, bodyNodes
  for (const node of nodes) {
    if (node.type === SegmentEntryNode || node.type === EventEntryNode) {
      entryNode = node;
    } else if (node.type === ExitNode) {
      exitNode = node;
    } else {
      bodyNodes.push(node);
    }
  }

  return ok({ entryNode, exitNode, nodes: bodyNodes });
}
```

### 2.4 后端发布逻辑

**关键代码**: `upsertJourney()` (packages/backend-lib/src/journeys.ts:981-1095)

```typescript
// packages/backend-lib/src/journeys.ts:981-1095
export async function upsertJourney(params: UpsertJourneyResource) {
  const { id, name, definition, workspaceId, status, canRunMultiple, draft } = params;

  // === 核心逻辑: nullableDraft ===
  // 规则:
  // 1. 如果提供了 definition（发布操作），draft 置为 null
  // 2. 如果显式设置 draft = null，也清除 draft
  // 3. 否则保持传入的 draft
  const nullableDraft = definition || draft === null ? null : draft;

  // 数据库操作
  const updateResult = await tx
    .update(dbJourney)
    .set({
      name,
      definition,              // 可能有值（发布）或 undefined（不更新）
      draft: nullableDraft,     // 根据规则计算
      status,
      statusUpdatedAt,
      canRunMultiple,
    })
    .where(...)
    .returning();

  // 返回时区分状态
  let resource: SavedJourneyResource;
  if (journeyStatus === "NotStarted") {
    resource = {
      ...baseResource,
      status: journeyStatus,
      definition: journeyDefinition,  // 可以没有
    };
  } else {
    // Running/Paused/Broadcast 状态必须有 definition
    if (!journeyDefinition) {
      throw new Error("Journey status is not NotStarted but has no definition");
    }
    resource = {
      ...baseResource,
      status: journeyStatus,
      definition: journeyDefinition,
    };
  }
}
```

### 2.5 前端保存策略

**函数**: `shouldDraftBeUpdated()` (store.ts:2273-2310)

决定是否需要更新 draft：

```typescript
export function shouldDraftBeUpdated({
  draft,
  definition,
  journeyNodes,
  journeyEdges,
  journeyNodesIndex,
}): boolean {
  // 情况 1: 已有 draft，比较当前 state 与 draft
  if (draft) {
    return !deepEquals(
      journeyStateToDraft({ journeyNodes, journeyEdges }),
      draft,
    );
  }
  
  // 情况 2: 没有 draft（已发布），比较当前 state 与 definition
  if (!definition) {
    throw new Error("definition should exist if draft is undefined");
  }
  
  // 尝试将 state 转换为 definition 进行比较
  const draftFromStateResult = journeyDefinitionFromState({
    state: { journeyNodes, journeyEdges, journeyNodesIndex },
  });
  
  // 转换失败（比如节点不完整），也需要更新 draft
  if (draftFromStateResult.isErr()) {
    return true;
  }
  
  // 比较转换后的 definition 与已发布的 definition
  return !deepEquals(draftFromStateResult.value, definition);
}
```

**工作流加载策略**: `journeyResourceToState()` (store.ts:2312-...)

```typescript
export function journeyResourceToState(
  journey: SavedJourneyResource,
): JourneyStateForResource {
  // 优先加载 draft（如果存在）
  if (journey.draft) {
    return journeyDraftToState({
      ...journey,
      draft: journey.draft,
    });
  }
  
  // 否则加载 definition
  return journeyToState({
    ...journey,
    definition: journey.definition,
  });
}
```

### 2.6 完整发布流程

```
1. 用户在画布编辑
   → Zustand store 更新 (journeyNodes, journeyEdges)

2. 自动保存/手动保存
   → 检查 shouldDraftBeUpdated()
   → 如果需要，调用 API: { draft: journeyStateToDraft(state) }
   → 后端保存 draft 字段，definition 保持不变

3. 用户点击"发布"
   → 调用 journeyDefinitionFromState() 将 state 转为 definition
   → 调用 API: { definition: newDefinition }
   → 后端:
     * nullableDraft = definition ? null : draft
     * 更新 definition 字段
     * 清空 draft 字段
     * 更新状态为 Running

4. 重新加载
   → journeyResourceToState() 发现 draft 为 null
   → 使用 definition 恢复画布
```

## 3. 快照配置与实时可运行性检查的一致性边界

### 3.1 快照时机与内容

**快照时机**: 工作流启动时 (`signalWithStart`)

**代码位置**: 
- `triggerSegmentEntryJourney()` (journeys.ts:824-895)
- `triggerEventEntryJourneys()` (journeys.ts:715-809)

```typescript
await workflowClient.signalWithStart(userJourneyWorkflow, {
  taskQueue: "default",
  workflowId,
  args: [{
    journeyId,
    definition,        // <-- 快照: 从数据库读取的当前 definition
    workspaceId,
    userId,
    version: UserJourneyWorkflowVersion.V3,
    // ...
  }],
  signal: segmentUpdateSignal,
  signalArgs: [segmentUpdate],
});
```

**快照内容**: `JourneyDefinition`

```typescript
interface JourneyDefinition {
  entryNode: SegmentEntryNode | EventEntryNode;   // 入口节点（含 child）
  nodes: JourneyBodyNode[];                       // 中间节点数组
  exitNode: ExitNode;                             // 出口节点
}
```

### 3.2 运行时数据来源分类

| 数据类型 | 来源 | 时机 | 一致性保证 |
|---------|------|------|-----------|
| **节点结构** | 快照 definition | 工作流启动 | ✅ 不可变 |
| **节点配置** (延迟时间、模板ID、分段ID) | 快照 definition | 工作流启动 | ✅ 不可变 |
| **用户分段状态** | 实时查询 (getSegmentAssignment) | 节点执行时 | ⚠️ 最终一致 |
| **Journey 状态** (Running/Paused) | 实时查询 (isRunnable) | 关键节点 | ⚠️ 最终一致 |
| **用户属性** | 实时查询 (getComputedPropertyAssignment) | 消息发送时 | ⚠️ 最终一致 |
| **工作空间状态** | 实时查询 (isRunnable) | 关键节点 | ⚠️ 最终一致 |

### 3.3 可运行性检查

**函数**: `isRunnable()` (packages/backend-lib/src/journeys/userWorkflow/activities.ts:396-470)

```typescript
export async function isRunnable({
  workspaceId,
  journeyId,
  userId,
  eventKey,
  eventKeyName,
}): Promise<boolean> {
  // 并行查询三个数据源
  const [previousExitEvent, journey, workspace] = await Promise.all([
    // 1. 用户历史记录: 是否完成过此 Journey
    db().query.userJourneyEvent.findFirst({
      where: and(
        eq(dbUserJourneyEvent.journeyId, journeyId),
        eq(dbUserJourneyEvent.userId, userId),
        eventKey ? eq(dbUserJourneyEvent.eventKey, eventKey) : undefined,
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

**检查时机**:

1. **工作流启动前**: 启动器检查
2. **SegmentEntryNode 等待后**: `packages/backend-lib/src/journeys/userWorkflow.ts:637`
3. **ContinueAsNew 前**: `packages/backend-lib/src/journeys/userWorkflow.ts:1129`

```typescript
// 示例: SegmentEntryNode 执行
case JourneyNodeType.SegmentEntryNode: {
  const cn = currentNode;
  const initialSegmentAssignment = 
    (await getSegmentAssignmentHandler({...}))?.inSegment === true;
  
  if (!initialSegmentAssignment) {
    // 等待 segmentUpdateSignal
    await wf.condition(() => segmentAssignedTrue(cn.segment));
  }
  
  // 等待完成后，检查可运行性
  if (!(await isRunnable({ journeyId, userId, workspaceId }))) {
    logger.info("early exit unrunnable user journey");
    return null;  // 直接退出
  }
  
  nextNode = nodes.get(currentNode.child) ?? null;
  break;
}
```

### 3.4 一致性边界分析

#### 强一致区域 (快照定义)

**保证**: 工作流执行路径由启动时的 definition 决定

```
时间线:
T0: definition = [Entry → Delay → Message → Exit]
T1: 工作流启动，快照 definition
T2: 用户修改 definition = [Entry → Delay → Exit]
T3: 工作流执行 Delay 节点
T4: 工作流执行 Message 节点（仍执行快照版本）
```

**后果**:
- 已启动的工作流不受后续编辑影响
- 不同时间启动的工作流可能运行不同版本

#### 弱一致区域 (实时状态)

**保证**: 最终一致，通过检查点保证安全退出

```
场景 1: Journey 被暂停
T0: 工作流启动，Running
T1: 工作流进入 SegmentEntryNode 等待
T2: 用户暂停 Journey (status = Paused)
T3: 用户进入分段，signal 唤醒工作流
T4: 工作流检查 isRunnable() → false
T5: 工作流提前退出

场景 2: 工作空间被禁用
T0: 工作流启动
T1: 工作流执行 Delay (sleep 1小时)
T2: 工作空间被禁用 (status = Inactive)
T3: sleep 结束
T4: 工作流检查 isRunnable() → false
T5: 工作流提前退出
```

#### 一致性边界图示

```
┌─────────────────────────────────────────────────────────────────┐
│                    快照配置 (强一致)                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  entryNode.child = "delay-1"                             │   │
│  │  nodes["delay-1"].child = "message-1"                    │   │
│  │  nodes["message-1"].child = "exit"                       │   │
│  │  nodes["message-1"].templateId = "template-A"            │   │
│  │                                                           │   │
│  │  ⚠️  工作流生命周期内不可变                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                 实时数据 (最终一致)                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ 用户分段状态  │  │ Journey 状态 │  │ 工作空间状态          │  │
│  │ (inSegment)  │  │ (Running)    │  │ (Active)             │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                 │                     │               │
│         └─────────────────┼─────────────────────┘               │
│                           ↓                                     │
│              isRunnable() 检查点                                 │
│                   (可能导致提前退出)                               │
└─────────────────────────────────────────────────────────────────┘
```

### 3.5 双写机制与一致性

**PostgreSQL 存储**:
- `journey` 表: definition/draft/status
- `userJourneyEvent` 表: 用户 Journey 历史

**ClickHouse 存储**:
- `internal_events` 表: `DFJourneyNodeProcessed` 事件

**写入流程**:
```typescript
// recordNodeProcessed() - packages/backend-lib/src/journeys/recordNodeProcessed.ts
// 1. 写入 PostgreSQL (用于 isRunnable 检查)
await db().insert(dbUserJourneyEvent).values({...});

// 2. 写入 ClickHouse (用于统计分析)
await submitTrack({
  workspaceId,
  data: {
    type: InternalEventType.DFJourneyNodeProcessed,
    userId,
    timestamp: now.toISOString(),
    journeyId,
    nodeId,
    nodeType,
  },
});
```

**一致性保证**:
- PostgreSQL 先写，用于运行时判断
- ClickHouse 异步写，用于分析
- 通过幂等性（messageId/nodeId）防止重复

## 4. 三种 Signal 的去重、防旧与唤醒顺序

### 4.1 Signal 概览

| Signal 名称 | 触发时机 | 用途 | 处理函数 |
|------------|---------|------|---------|
| `segmentUpdateSignal` | 用户分段变化 | 更新分段状态，唤醒等待 | `setHandler(segmentUpdateSignal, ...)` |
| `trackSignal` | EventEntry 事件 | Keyed Journey 事件收集 | `setHandler(trackSignal, ...)` |
| `reEvaluateSegmentsSignal` | 手动触发 | 强制重新计算分段 | `setHandler(reEvaluateSegmentsSignal, ...)` |

### 4.2 segmentUpdateSignal: 防旧机制

**信号定义**:
```typescript
export const segmentUpdateSignal = wf.defineSignal<ReceivedSegmentUpdate>(
  "segmentUpdateSignal",
);

interface ReceivedSegmentUpdate {
  segmentId: string;
  currentlyInSegment: boolean;
  segmentVersion: number;  // ⚠️ 版本号，用于防旧
}
```

**处理逻辑** (userWorkflow.ts:529-549):

```typescript
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
  
  // === 防旧检查 ===
  // 如果已有更新，且版本号 >= 新更新的版本号，忽略
  if (prev && prev.segmentVersion >= update.segmentVersion) {
    logger.info("ignoring stale segment update", loggerAttrs);
    return;  // 直接返回，不更新
  }

  logger.info("segment update", loggerAttrs);
  
  // 更新分段状态
  segmentAssignments.set(update.segmentId, {
    currentlyInSegment: update.currentlyInSegment,
    segmentVersion: update.segmentVersion,
  });
});
```

**防旧原理**:
```
版本号 = 更新时间戳 (ms)

场景:
T0: segmentAssignments = {}
T1 (1000ms): 收到 update { segmentId: "S1", inSegment: false, version: 1000 }
    → 更新: segmentAssignments["S1"] = { inSegment: false, version: 1000 }
    
T2 (1500ms): 收到 update { segmentId: "S1", inSegment: true, version: 1500 }
    → 1500 > 1000 ✓
    → 更新: segmentAssignments["S1"] = { inSegment: true, version: 1500 }
    
T3 (1200ms): 收到 update { segmentId: "S1", inSegment: false, version: 1200 }
    → 1200 < 1500 ✗ (延迟到达的旧消息)
    → 忽略!
```

**版本号来源**:
- 分段计算完成时间戳
- 信号发送时设置

### 4.3 trackSignal: 去重机制

**信号定义**:
```typescript
export const trackSignal = wf.defineSignal<UserWorkflowTrackEvent>(
  "trackSignal",
);
```

**处理逻辑** (userWorkflow.ts:444-527):

```typescript
// 去重集合: 已处理的事件 ID
const keyedEventIds = new Set<string>();

wf.setHandler(trackSignal, async (event) => {
  logger.info("keyed event signal", {
    workspaceId,
    journeyId,
    userId,
    messageId: event.messageId,
  });
  
  // === 去重检查 ===
  if (keyedEventIds.has(event.messageId)) {
    logger.info("ignoring duplicate keyed event", {
      journeyId,
      userId,
      workspaceId,
      messageId: event.messageId,
    });
    return;  // 已处理过，忽略
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
      keyedEventIds.add(event.messageId);  // <-- 记录已处理
      break;
    }
    case TrackSignalParamsVersion.V1:
    default: {
      // V1: 直接使用信号中的事件数据
      if (keyedEvents) {
        keyedEvents.push(event);
      }
      keyedEventIds.add(event.messageId);  // <-- 记录已处理
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

**去重原理**:
```
keyedEventIds = Set<string>

场景 (Keyed EventEntryNode):
T0: keyedEventIds = {}

T1: 收到 event { messageId: "event-1", key: "user-123", ... }
    → "event-1" not in keyedEventIds ✓
    → 处理事件
    → keyedEventIds.add("event-1")
    → keyedEventIds = { "event-1" }

T2: 收到 event { messageId: "event-1", key: "user-123", ... } (重试/重复)
    → "event-1" in keyedEventIds ✗
    → 忽略!

T3: 收到 event { messageId: "event-2", key: "user-123", ... }
    → "event-2" not in keyedEventIds ✓
    → 处理事件
    → keyedEventIds.add("event-2")
    → keyedEventIds = { "event-1", "event-2" }
```

### 4.4 reEvaluateSegmentsSignal: 强制刷新

**信号定义**:
```typescript
export const reEvaluateSegmentsSignal = wf.defineSignal<{
  segmentIds?: string[];
}>("reEvaluateSegmentsSignal");
```

**处理逻辑** (userWorkflow.ts:551-599):

```typescript
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
      // === 强制重新计算 ===
      // 不检查版本号，直接查询最新状态
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
        segmentVersion: nowMs,  // <-- 新版本号
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

**特点**:
- **无防旧**: 不检查版本号，总是执行
- **无去重**: 每次收到都重新计算
- **使用场景**: 手动触发、修复不一致状态

### 4.5 唤醒机制: Signal + Condition

**工作原理**:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Temporal 工作流                          │
│                                                                 │
│  segmentAssignments (Map)                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  "segment-1": { inSegment: false, version: 1000 }        │  │
│  │  "segment-2": { inSegment: true, version: 1500 }         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           ↑                                     │
│                           │ 信号处理器更新                        │
│                           │                                     │
│  Signal Handlers                                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐ │
│  │segmentUpdateSignal│  │  trackSignal     │  │reEvaluate... │ │
│  │  (防旧)          │  │  (去重)          │  │  (强制)      │ │
│  └────────┬─────────┘  └────────┬─────────┘  └──────┬───────┘ │
│           │                     │                   │          │
│           └─────────────────────┼───────────────────┘          │
│                                 ↓                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  await wf.condition(() =>                                  │  │
│  │    segmentAssignments.get("segment-1")?.inSegment === true │  │
│  │  )                                                         │  │
│  │                                                             │  │
│  │  ⚠️  信号处理器修改 Map → condition 重新求值 → 唤醒          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**关键代码示例**:

```typescript
// 1. 定义信号
export const segmentUpdateSignal = wf.defineSignal<ReceivedSegmentUpdate>(...);

// 2. 工作流中设置处理器（修改共享状态）
wf.setHandler(segmentUpdateSignal, (update) => {
  // 修改共享 Map
  segmentAssignments.set(update.segmentId, {
    currentlyInSegment: update.currentlyInSegment,
    segmentVersion: update.segmentVersion,
  });
  // 处理器返回后，Temporal 会检查所有等待的 condition
});

// 3. 节点中等待（检查共享状态）
case JourneyNodeType.SegmentEntryNode: {
  const cn = currentNode;
  const initialSegmentAssignment = 
    (await getSegmentAssignmentHandler({...}))?.inSegment === true;
  
  if (!initialSegmentAssignment) {
    // 等待条件：分段状态变为 true
    await wf.condition(() => segmentAssignedTrue(cn.segment));
  }
  // 被唤醒后继续执行
  nextNode = nodes.get(currentNode.child) ?? null;
  break;
}

// 4. condition 检查函数
function segmentAssignedTrue(segmentId: string): boolean {
  return segmentAssignments.get(segmentId)?.currentlyInSegment === true;
}
```

### 4.6 三种 Signal 对比

| 特性 | segmentUpdateSignal | trackSignal | reEvaluateSegmentsSignal |
|------|---------------------|-------------|------------------------|
| **触发方式** | 外部发送 | 外部发送 | 外部发送/手动 |
| **去重** | ❌ (版本检查替代) | ✅ (keyedEventIds Set) | ❌ |
| **防旧** | ✅ (segmentVersion 比较) | ❌ | ❌ |
| **更新状态** | 直接使用信号值 | 间接更新分段 | 重新查询 |
| **强制刷新** | ❌ | ❌ | ✅ |
| **使用场景** | 分段变化通知 | Keyed Event 收集 | 手动修复/调试 |

### 4.7 完整唤醒时序

```
场景: SegmentEntryNode 等待用户进入分段

T0: 工作流启动
    → 初始化 segmentAssignments = {}
    
T1: 执行 SegmentEntryNode
    → getSegmentAssignmentHandler() 查询: inSegment = false
    → await wf.condition(() => segmentAssignedTrue("S1"))
    → 工作流挂起，等待信号
    
T2: 用户进入分段 (外部事件)
    → 分段计算服务: inSegment = true
    → 发送 segmentUpdateSignal:
      { segmentId: "S1", inSegment: true, version: 2000 }
    
T3: 工作流收到信号
    → 处理器执行: segmentAssignments.set("S1", {inSegment: true, version: 2000})
    → 处理器返回
    → Temporal 检查 condition: segmentAssignedTrue("S1") → true ✓
    → condition 满足，工作流唤醒
    
T4: 工作流继续执行
    → 检查 isRunnable()
    → 执行下一个节点
```

## 5. 关键代码索引

### 5.1 画布与 State 管理

| 文件 | 函数/类型 | 职责 |
|------|----------|------|
| `packages/dashboard/src/components/journeys/store.ts` | `JourneyContent` | 画布状态接口 |
| `packages/dashboard/src/components/journeys/store.ts` | `buildUiHeritageMap()` | 构建节点关系图 |
| `packages/dashboard/src/components/journeys/store.ts` | `getNearestJourneyFromChildren()` | 找分支汇合点 |
| `packages/dashboard/src/components/journeys/store.ts` | `journeyDefinitionFromStateBranch()` | 递归转换 State→Definition |
| `packages/dashboard/src/components/journeys/store.ts` | `journeyBranchToState()` | 反向转换 Definition→State |

### 5.2 Draft 与 Definition 切换

| 文件 | 函数/类型 | 职责 |
|------|----------|------|
| `packages/dashboard/src/components/journeys/store.ts` | `journeyStateToDraft()` | State→Draft 转换 |
| `packages/dashboard/src/components/journeys/store.ts` | `journeyDraftToState()` | Draft→State 转换 |
| `packages/dashboard/src/components/journeys/store.ts` | `shouldDraftBeUpdated()` | 判断是否需要保存 draft |
| `packages/dashboard/src/components/journeys/store.ts` | `journeyResourceToState()` | 加载策略 (draft 优先) |
| `packages/backend-lib/src/journeys.ts` | `upsertJourney()` | nullableDraft 逻辑 |

### 5.3 信号处理

| 文件 | 函数/类型 | 职责 |
|------|----------|------|
| `packages/backend-lib/src/journeys/userWorkflow.ts` | `segmentUpdateSignal` | 分段更新信号定义 |
| `packages/backend-lib/src/journeys/userWorkflow.ts` | `trackSignal` | 事件追踪信号定义 |
| `packages/backend-lib/src/journeys/userWorkflow.ts` | `reEvaluateSegmentsSignal` | 重新评估信号定义 |
| `packages/backend-lib/src/journeys/userWorkflow.ts` | `wf.setHandler(segmentUpdateSignal)` | 防旧处理 |
| `packages/backend-lib/src/journeys/userWorkflow.ts` | `wf.setHandler(trackSignal)` | 去重处理 |
| `packages/backend-lib/src/journeys/userWorkflow.ts` | `wf.setHandler(reEvaluateSegmentsSignal)` | 强制刷新 |
| `packages/backend-lib/src/journeys/userWorkflow/activities.ts` | `isRunnable()` | 可运行性检查 |
