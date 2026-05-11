# 人群分段 (Segment) 从查询计算到 Journey 触发机制分析

## 一、Segment 定义结构

### 1.1 核心数据结构

Segment 的定义存储在 Postgres 的 `segment` 表中，由 `SegmentDefinition` 类型描述：

```typescript
// 定义入口
interface SegmentDefinition {
  entryNode: SegmentNode;      // 入口节点
  nodes: SegmentNode[];        // 辅助节点（用于 And/Or 逻辑组合）
}
```

### 1.2 Segment 节点类型

| 节点类型 | 说明 | 示例 |
|---------|------|------|
| `Performed` | 基于事件执行次数 | "购买过至少3次商品" |
| `Trait` | 基于用户属性 | "年龄大于30岁" |
| `And` | 逻辑与 | 同时满足多个条件 |
| `Or` | 逻辑或 | 满足任意一个条件 |
| `Manual` | 手动分段 | 手动导入用户列表 |
| `RandomBucket` | 随机分桶 | "前20%用户" |
| `Email` | 邮件互动 | "打开过某封邮件" |
| `SubscriptionGroup` | 订阅组状态 | "已订阅 newsletter" |
| `KeyedPerformed` | 带键值的事件统计 | 按事件属性分组统计 |
| `Everyone` | 所有人 | 全体用户 |
| `Includes` | 数组包含 | "属性数组包含某个值" |
| `Broadcast` | 广播分段 | 已废弃 |

### 1.3 关键状态标识

每个 Segment 有状态生命周期：
- `NotStarted` - 未启动（Manual 分段默认状态）
- `Running` - 运行中
- `Paused` - 已暂停

---

## 二、Segment 编译为 ClickHouse 查询

### 2.1 整体编译流程

Segment 的编译和计算分为 **三个阶段**：

```
原始事件 (user_events_v2)
    ↓
[阶段1] computeState: 计算中间状态
    ↓
computed_property_state_v3
    ↓
[阶段2] computeAssignments: 计算最终成员
    ↓
computed_property_assignments_v2
    ↓
[阶段3] processAssignments: 处理变化，触发下游
    ↓
Journey / Integration
```

### 2.2 阶段一：State 计算

核心函数：`segmentNodeToStateSubQuery` (packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts:1767)

将 Segment 节点转换为原始事件查询，写入 `computed_property_state_v3` 表。

**状态 ID 生成机制**：
```typescript
// packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts:394
export function segmentNodeStateId(
  segment: SavedSegmentResource,
  nodeId: string,
): string | null {
  // 基于 segment.definitionUpdatedAt + nodeId 生成 UUID
  // 确保定义变化时状态 ID 也变化
  const name = `${segment.definitionUpdatedAt.toString()}:${nodeId}${versionSuffix}`;
  return uuidv5(name, segment.id);
}
```

**不同节点类型的编译示例**：

#### 2.2.1 Performed 节点（事件计数）

```sql
-- 原始事件聚合为状态
INSERT INTO computed_property_state_v3
SELECT
  ue.workspace_id,
  'segment' AS type,
  '{segment_id}' AS computed_property_id,
  '{state_id}' AS state_id,
  ue.user_or_anonymous_id,
  argMaxState('' as last_value, ue.event_time),
  uniqState(message_id as unique_value),  -- 用于计数
  truncated_event_time,
  groupArrayState('' as grouped_message_id),
  now() as computed_at
FROM user_events_v2 ue
WHERE
  workspace_id = '{workspace_id}'
  AND processing_time <= now()
  AND (event_type == 'track' AND startsWithUTF8(event, 'Purchase') AND ...)
  AND processing_time >= {period_bound}  -- 增量窗口
GROUP BY ue.workspace_id, ue.user_or_anonymous_id, ue.event_time
```

#### 2.2.2 Trait 节点（用户属性）

```sql
INSERT INTO computed_property_state_v3
SELECT
  ...
  argMaxState(JSON_VALUE(properties, '$.age') as last_value, ue.event_time),
  uniqState('' as unique_value)
FROM user_events_v2 ue
WHERE event_type == 'identify'
```

#### 2.2.3 And/Or 逻辑组合

And/Or 节点本身不直接生成查询，而是递归处理子节点：
- 编译阶段：遍历所有子节点，生成各自的 state 查询
- 赋值阶段：组合子节点的结果

### 2.3 阶段二：Assignment 计算

核心函数：`computeAssignments` (packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts:3255)

将中间状态转换为最终的用户-分段归属关系。

#### 2.3.1 Resolved State 解析

函数：`segmentToResolvedState` (packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts:561)

**Performed 节点解析示例**（带时间窗口）：

```sql
-- 处理过期用户（不再满足条件的用户）
INSERT INTO resolved_segment_state
SELECT
  workspace_id, segment_id, state_id, user_id,
  False, max_event_time, now()
FROM resolved_segment_state rss
WHERE rss.segment_state_value = True
  AND (workspace_id, segment_id, state_id, user_id, True) NOT IN (
    -- 当前满足条件的用户
    SELECT
      workspace_id, computed_property_id, state_id, user_id,
      uniqMerge(unique_count) >= 3 as segment_state_value
    FROM computed_property_state_v3 cps_performed
    WHERE cps_performed.state_id = '{state_id}'
    GROUP BY workspace_id, computed_property_id, state_id, user_id
    HAVING segment_state_value = True
  )

-- 处理新进入用户
INSERT INTO resolved_segment_state
SELECT
  workspace_id, segment_id, state_id, user_id,
  True, max_event_time, now()
FROM (
  -- 拥有用户 ID 且满足条件但尚未在段中的用户
  SELECT ...
)
```

#### 2.3.2 And/Or 逻辑组合

函数：`resolvedSegmentToAssignment` (packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts:1584)

将多个子节点的 state 组合为最终 assignment：

```sql
-- AND 逻辑
INSERT INTO computed_property_assignments_v2
SELECT
  workspace_id, 'segment', segment_id, user_id,
  (state_values['{state_id_1}'] AND state_values['{state_id_2}']) as segment_value,
  '', max_state_event_time, now()
FROM (
  SELECT
    workspace_id, segment_id, user_id,
    CAST((groupArray(state_id), groupArray(segment_state_value)), 'Map(String, Boolean)') as state_values,
    max(max_state_event_time) as max_state_event_time
  FROM resolved_segment_state
  GROUP BY workspace_id, segment_id, user_id
)
```

### 2.4 关键表结构

| 表名 | 用途 | 粒度 |
|-----|------|------|
| `computed_property_state_v3` | 中间状态存储 | (workspace, type, computed_property_id, state_id, user_id, event_time) |
| `resolved_segment_state` | 解析后的布尔状态 | (workspace, segment_id, state_id, user_id) |
| `computed_property_assignments_v2` | 最终成员归属 | (workspace, type, computed_property_id, user_id) |
| `processed_computed_properties_v2` | 已处理的变更记录 | 用于去重 |
| `processed_computed_properties` | 周期追踪 | 记录每个 computed property 的处理进度 |

---

## 三、Segment 成员变化驱动 Journey 启动

### 3.1 整体触发流程

```
用户事件流入
    ↓
[定期或增量触发] computePropertiesWorkflow
    ↓
计算 segment 状态变化
    ↓
[processAssignments] 找出新增/变更的成员
    ↓
[triggerSegmentEntryJourney] 启动或发送信号给 Journey
    ↓
[Temporal Workflow] userJourneyWorkflow 执行
```

### 3.2 变更检测机制

核心查询：`buildProcessAssignmentsQuery` (packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts:3786)

该查询找出哪些用户的 segment 归属发生了变化：

```sql
SELECT
  cpa.user_id,
  cpa.latest_segment_value,      -- 当前状态
  cpa.max_assigned_at,
  pcp.user_id as processed_user  -- 是否已处理
FROM (
  -- 当前最新的 segment 赋值
  SELECT
    user_id,
    max(assigned_at) max_assigned_at,
    argMax(segment_value, assigned_at) latest_segment_value
  FROM computed_property_assignments_v2
  WHERE workspace_id = ? AND type = 'segment' AND computed_property_id = ?
  GROUP BY user_id
) cpa
LEFT ANY JOIN (
  -- 已处理过的赋值记录
  SELECT
    user_id,
    argMax(segment_value, processed_at) segment_value
  FROM processed_computed_properties_v2
  WHERE ... AND processed_for = {journey_id}
  GROUP BY user_id
) pcp ON cpa.user_id = pcp.user_id
WHERE
  -- 值发生变化
  (cpa.latest_segment_value != pcp.segment_value OR pcp.user_id = '')
  AND (
    -- 只处理进入分段的用户（latest_segment_value = true）
    -- 或者是之前已在分段中但现在状态变化的用户
    cpa.latest_segment_value = true OR pcp.user_id != ''
  )
```

### 3.3 Journey 触发机制

核心函数：`triggerSegmentEntryJourney` (packages/backend-lib/src/journeys.ts:814)

```typescript
export async function triggerSegmentEntryJourney({
  workspaceId,
  segmentId,
  segmentAssignment,
  journey,
}: {
  ...
}) {
  // 只处理进入分段的用户
  if (!segmentUpdate.currentlyInSegment) {
    return;
  }

  // 使用 Temporal 的 signalWithStart
  await workflowClient.signalWithStart<typeof userJourneyWorkflow, [SegmentUpdate]>(
    userJourneyWorkflow,
    {
      taskQueue: "default",
      workflowId: getUserJourneyWorkflowId({ journeyId, userId }),
      args: [{ journeyId, definition, workspaceId, userId }],
      signal: segmentUpdateSignal,           // 发送 segment 更新信号
      signalArgs: [{
        segmentId,
        currentlyInSegment: true,
        segmentVersion: timestamp,
        type: "segment"
      }],
    }
  );
}
```

### 3.4 Workflow 中的信号处理

在 `userJourneyWorkflow` (packages/backend-lib/src/journeys/userWorkflow.ts:206) 中：

```typescript
// 定义信号
export const segmentUpdateSignal =
  wf.defineSignal<[SegmentUpdate]>("segmentUpdate");

// 注册信号处理器
wf.setHandler(segmentUpdateSignal, (update) => {
  const prev = segmentAssignments.get(update.segmentId);
  // 版本检查，忽略过期更新
  if (prev && prev.segmentVersion >= update.segmentVersion) {
    return;
  }
  segmentAssignments.set(update.segmentId, {
    currentlyInSegment: update.currentlyInSegment,
    segmentVersion: update.segmentVersion,
  });
});

// SegmentEntryNode 等待条件
case JourneyNodeType.SegmentEntryNode: {
  const initialSegmentAssignment =
    (await getSegmentAssignmentHandler({ segmentId: cn.segment, now: Date.now() }))?.inSegment === true;
  if (!initialSegmentAssignment) {
    // 等待 signal 触发
    await wf.condition(() => segmentAssignedTrue(cn.segment));
  }
  // 继续执行后续节点
  nextNode = nodes.get(currentNode.child);
  break;
}
```

### 3.5 "出分段"行为分析

#### 3.5.1 Journey 路径中的过滤位置

在 `processRowsInner` 函数中，`latest_segment_value = false`（出分段）被显式过滤，不会触发 Journey 启动：

```typescript
// packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts:3637-3648
for (const assignment of assignments) {
  let assignmentCategory: ComputedAssignment[];
  if (assignment.processed_for_type === "integration") {
    assignmentCategory = integrationAssignments;
  } else {
    if (!assignment.latest_segment_value) {
      continue;  // ← 出分段用户被直接跳过
    }
    assignmentCategory = journeySegmentAssignments;
  }
  assignmentCategory.push(assignment);
}
```

**过滤位置分析**：
- **第一层过滤（SQL 查询层）**：在 `buildProcessAssignmentsQuery` 中，`typeCondition = "cpa.latest_segment_value = true"`（第 3816 行），但被 `OR (pcp.user_id != '')` 条件部分绕过，允许"之前在段中但现在出分段"的用户进入结果集
- **第二层过滤（代码层）**：在 `processRowsInner` 中，`if (!assignment.latest_segment_value) continue` 无条件跳过所有出分段用户

**为何不会触发启动**：
1. **语义设计**：SegmentEntryNode 仅在用户"进入"分段时触发 Journey，出分段是"退出"事件，不应启动新的 Journey
2. **单向触发模型**：Dittofeed 的 Journey 设计为单向流程，一旦启动后在 Workflow 内部管理生命周期（通过 Signal 通知状态变化），出分段不会终止已启动的 Journey
3. **去重逻辑**：`processed_computed_properties_v2` 表记录了已处理的赋值，结合 `argMax(segment_value, processed_at)` 避免重复处理

#### 3.5.2 Integration 路径的处理差异

Integration 路径**不会过滤** `latest_segment_value = false`，而是将完整的状态变化传递给下游集成：

```typescript
// packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts:3683-3709
...integrationAssignments.flatMap(async (assignment) => {
  switch (assignment.processed_for) {
    case HUBSPOT_INTEGRATION: {
      const update: ComputedPropertyUpdate =
        assignment.type === "segment"
          ? {
              type: "segment",
              segmentId: assignment.computed_property_id,
              segmentVersion: updateVersion,
              currentlyInSegment: assignment.latest_segment_value,  // ← 传递完整状态
            }
          : { ... };

      return startHubspotUserIntegrationWorkflow({
        workspaceId: assignment.workspace_id,
        userId: assignment.user_id,
        workflowClient,
        update,
      });
    }
  }
});
```

**HubSpot 实际处理逻辑**（packages/backend-lib/src/integrations/hubspot/activities.ts:1037-1048）：

```typescript
if (update.currentlyInSegment) {
  return api.addContactToList({
    token: hubspotAccessToken.accessToken,
    listId,
    email,
  });
}
return api.removeContactFromList({  // ← 出分段时从列表移除
  token: hubspotAccessToken.accessToken,
  listId,
  email,
});
```

**两条路径的对比**：

| 维度 | Journey 路径 | Integration 路径 |
|-----|-------------|-----------------|
| `latest_segment_value = false` 处理 | 被过滤（continue） | 完整传递给下游 |
| 触发动作 | 仅启动新 Journey | 双向同步（add/remove） |
| 语义 | 单向"进入"触发 | 双向状态同步 |
| 生命周期管理 | Workflow 内部通过 Signal 管理 | 每次变化都触发独立动作 |

### 3.6 订阅关系建立

在 `processAssignments` 中建立 segment -> journey 的映射：

```typescript
// segment id -> journey ids
const subscribedJourneyMap = journeys.reduce<Map<string, Set<string>>>(
  (memo, j) => {
    const subscribedSegments = getSubscribedSegments(j.definition);
    subscribedSegments.forEach((segmentId) => {
      const processFor = memo.get(segmentId) ?? new Set();
      processFor.add(j.id);
      memo.set(segmentId, processFor);
    });
    return memo;
  },
  new Map()
);
```

然后为每个 (segment, journey) 对创建 `AssignmentProcessor` 进行分页处理。

---

## 四、Period 窗口机制与全量重算条件

### 4.1 增量计算的核心：Period 机制

Period 用于追踪每个 computed property 的计算进度，实现增量计算。

```typescript
// packages/backend-lib/src/computedProperties/periods.ts
interface Period {
  maxFrom: Date;  // 已处理的起始时间
  maxTo: Date;    // 已处理的结束时间（下一次从此开始）
}
```

**获取计算窗口**：
```typescript
const period = periodByComputedPropertyId.get({
  computedPropertyId: segment.id,
  version: segment.definitionUpdatedAt.toString(),  // 版本号作为 part of key
});
const periodBound = period?.maxTo.getTime();  // 增量起始点
```

### 4.2 全量重算的触发条件

函数：`shouldResetComputedProperty` (packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts:129)

```typescript
function shouldResetComputedProperty({
  definitionUpdatedAt,  // segment 定义更新时间
  createdAt,            // segment 创建时间
  now,
  periodBound,          // 当前增量窗口
}): boolean {
  if (!definitionUpdatedAt) {
    return false;
  }
  return (
    definitionUpdatedAt <= now &&
    definitionUpdatedAt >= (periodBound ?? 0) &&
    definitionUpdatedAt > createdAt
  );
}
```

**触发全量重算的场景**：
1. Segment 定义被修改（`definitionUpdatedAt` 更新）
2. 定义更新发生在当前计算周期内

**全量重算时执行**：
```sql
-- 清除旧的 assignment 记录
DELETE FROM computed_property_assignments_v2
WHERE workspace_id = ?
  AND type = 'segment'
  AND computed_property_id = ?
  AND assigned_at < now()
```

### 4.3 两种模式的对比

| 维度 | 增量更新 | 全量重算 |
|-----|---------|---------|
| **触发时机** | 定期调度、事件驱动 | 定义变更、首次运行 |
| **计算范围** | 基于 `periodBound` 的时间窗口 | 所有历史事件 |
| **性能特点** | 计算量小、延迟低 | 计算量大、资源消耗高 |
| **数据一致性** | 依赖 period 追踪 | 保证绝对一致 |
| **适用场景** | 日常运行 | 定义变更、数据修复 |

### 4.4 调度与触发机制

**定期调度**：`computePropertiesSchedulerWorkflow` 按固定周期运行

**手动/事件触发**：
- 队列系统：`WorkspaceQueueItem` 支持 Segment、Journey、Integration、UserProperty、Workspace 级别
- 优先级机制：`QUEUE_ITEM_PRIORITIES` (Explicit > Split > 默认)

```typescript
// packages/backend-lib/src/constants.ts
export const QUEUE_ITEM_PRIORITIES = {
  Explicit: 10,   // 显式触发（最高）
  Split: 50,      // 拆分后的独立任务
  // 默认: 100
};
```

### 4.5 关键配置项

| 配置项 | 作用 | 默认值 |
|-------|------|--------|
| `computePropertiesSplit` | 是否将 workspace 级任务拆分为独立 computed property 任务 | false |
| `clickhouseComputePropertiesRequestTimeout` | ClickHouse 查询超时 | - |
| `clickhouseComputePropertiesMaxExecutionTime` | 查询最大执行时间 | - |
| `readQueryConcurrency` | 读取查询并发限制 | - |
| `readQueryPageSize` | 分页查询页大小 | - |

### 4.6 延迟与权衡

**实时性与资源消耗的平衡**：

1. **计算延迟**：依赖调度周期，非严格实时
2. **批处理优势**：
   - 减少 ClickHouse 查询次数
   - 利用批量处理优化
   - 降低 Temporal workflow 数量

3. **当需要更实时时**：
   - 可以通过 `enqueueRecompute` 显式触发
   - 例如 Manual segment 关联的 Journey 启动时会立即触发计算

---

## 五、代码引用索引

| 功能 | 文件位置 | 关键函数 |
|-----|---------|---------|
| Segment 定义 | packages/isomorphic-lib/src/types.ts | SegmentDefinition, SegmentNode |
| Segment 资源操作 | packages/backend-lib/src/segments.ts | upsertSegment, findEnrichedSegment |
| Segment 编译为查询 | packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts | segmentNodeToStateSubQuery, segmentToResolvedState |
| 计算流程编排 | packages/backend-lib/src/computedProperties/computePropertiesWorkflow/activities/computeProperties.ts | computePropertiesIncremental |
| 赋值变更处理 | packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts | processAssignments, buildProcessAssignmentsQuery |
| Journey 触发 | packages/backend-lib/src/journeys.ts | triggerSegmentEntryJourney, findSubscribedRunningJourneysForSegment |
| User Journey 工作流 | packages/backend-lib/src/journeys/userWorkflow.ts | userJourneyWorkflow, segmentUpdateSignal |
| Period 管理 | packages/backend-lib/src/computedProperties/periods.ts | createPeriods, getPeriodsByComputedPropertyId |
