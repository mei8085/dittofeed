# Dittofeed 身份解析与多来源写入冲突处理分析

## 1. 概述

Dittofeed 作为客户数据平台 (CDP)，支持多客户端 SDK 上报用户行为数据。本报告深入分析以下核心机制：

1. **API Key 多入口鉴权差异**
2. **匿名身份与已知用户合并逻辑**
3. **多来源写入冲突处理**

---

## 2. API Key 多入口鉴权差异

### 2.1 鉴权架构

Dittofeed 的 API Key 鉴权机制位于 `packages/backend-lib/src/auth.ts:75-116`，核心实现为 `validateWriteKey` 函数。

#### 2.1.1 鉴权流程

```
客户端请求 → Authorization Header (Basic Auth)
    ↓
提取 Base64 编码的 Write Key
    ↓
解码为 secretKeyId:secretKeyValue
    ↓
查询数据库验证 secretKeyId 有效性
    ↓
验证 secretKeyValue 匹配
    ↓
检查 Workspace 状态 (Active 且非 Parent)
    ↓
返回 workspaceId
```

#### 2.1.2 Write Key 格式

```
Authorization: Basic base64(secretKeyId:secretKeyValue)
```

- **secretKeyId**: UUID v4 格式，用于数据库查找
- **secretKeyValue**: 8 字节随机生成的安全密钥

### 2.2 多入口鉴权差异

#### 2.2.1 Write Key 管理

Dittofeed 支持**同一 Workspace 下多个 Write Key**，通过 `getOrCreateWriteKey` 函数 (`auth.ts:118-185`) 管理：

```typescript
export async function getOrCreateWriteKey({
  writeKeyName,    // 名称标识，如 "web", "mobile", "server"
  workspaceId,
}: {
  workspaceId: string;
  writeKeyName: string;
}): Promise<WriteKeyResource>
```

**多入口设计意图**：
- **Web 端**: 独立 Write Key，可单独撤销
- **移动端**: 独立 Write Key，支持多应用隔离
- **服务端**: 独立 Write Key，更高安全级别
- **不同环境**: 开发/测试/生产环境隔离

#### 2.2.2 Workspace 权限控制

鉴权时通过 `canWorkspaceReceiveEvents` 函数 (`auth.ts:39-48`) 检查：

```typescript
export function canWorkspaceReceiveEvents({
  workspace,
}: {
  workspace: Workspace;
}): boolean {
  return (
    workspace.status === WorkspaceStatusDbEnum.Active &&
    workspace.type !== "Parent"
  );
}
```

**鉴权差异点**：

| 入口类型 | 鉴权验证点 | 特殊处理 |
|---------|-----------|---------|
| 活跃 Workspace | status === Active | 允许接收事件 |
| 非活跃 Workspace | status !== Active | 返回 WorkspaceIneligible |
| Parent Workspace | type === "Parent" | 返回 WorkspaceIneligible |
| 子 Workspace | parentWorkspaceId 存在 | 数据归属子 Workspace |

#### 2.2.3 错误类型

```typescript
export type ValidateWriteKeyError =
  | "InvalidWriteKey"      // 格式错误或密钥不匹配
  | "WorkspaceInactive"    // Workspace 非活跃
  | "WorkspaceIneligible"; // Workspace 类型不支持
```

### 2.3 数据写入模式

通过 `insertUserEvents` 函数 (`userEvents.ts:90-145`) 支持三种写入模式：

| 模式 | 实现 | 适用场景 |
|-----|------|---------|
| `kafka` | 写入 Kafka Topic `userEventsTopicName` | 高吞吐量生产环境 |
| `ch-async` | ClickHouse 异步插入 | 中等流量 |
| `ch-sync` | ClickHouse 同步插入 | 测试/低流量 |

---

## 3. 匿名身份与已知用户合并逻辑

### 3.1 事件类型支持

Dittofeed 支持 Segment 协议的标准事件类型 (`isomorphic-lib/src/types.ts:75-82`)：

```typescript
export enum EventType {
  Identify = "identify",   // 用户识别
  Track = "track",         // 行为追踪
  Page = "page",           // 页面浏览
  Screen = "screen",       // 屏幕浏览
  Group = "group",         // 用户分组
  Alias = "alias",         // 身份合并 ⭐
}
```

### 3.2 核心字段设计

在 `user_events_v2` 表中 (`userEvents/clickhouse.ts:299-364`)，关键身份字段定义：

```sql
user_id String DEFAULT JSONExtract(message_raw, 'userId', 'String'),
anonymous_id String DEFAULT JSONExtract(message_raw, 'anonymousId', 'String'),
user_or_anonymous_id String DEFAULT assumeNotNull(
  coalesce(
    JSONExtract(message_raw, 'userId', 'Nullable(String)'),
    JSONExtract(message_raw, 'anonymousId', 'Nullable(String)')
  )
),
```

#### 3.2.1 字段优先级

`user_or_anonymous_id` 采用 `coalesce` 策略：

1. **优先使用 userId**（已知用户）
2. **降级使用 anonymousId**（匿名用户）
3. **保证非空**：`assumeNotNull` 确保至少有一个 ID

#### 3.2.2 排序键设计

```sql
ENGINE = MergeTree()
ORDER BY (
  workspace_id,
  processing_time,
  user_or_anonymous_id,
  event_time,
  message_id
)
```

**设计意图**：
- 按 `user_or_anonymous_id` 排序，便于按用户维度查询
- 按 `processing_time` 和 `event_time` 排序，保证时序处理

### 3.3 身份合并机制分析

#### 3.3.1 Alias 事件支持

系统在表结构层面支持 `alias` 事件类型：

```sql
event_type Enum(
  'identify' = 1,
  'track' = 2,
  'page' = 3,
  'screen' = 4,
  'group' = 5,
  'alias' = 6    ⭐ 支持身份合并事件
)
```

#### 3.3.2 事件存储模式

Dittofeed 采用**事件溯源 (Event Sourcing)** 模式：

1. **原始事件持久化**：所有事件（包括 alias）存储在 `user_events_v2`
2. **材料化视图处理**：通过 Materialized View 派生计算结果
3. **查询时聚合**：使用 `argMax` 等函数在查询时计算最终状态

#### 3.3.3 身份关联实践

**场景 1: 匿名浏览 → 登录识别**

```
时间线:
T1: 匿名用户浏览页面
    { type: "page", anonymousId: "anon-123", event: "Page Viewed" }
    → user_or_anonymous_id = "anon-123"

T2: 用户登录，触发 identify
    { type: "identify", userId: "user-456", anonymousId: "anon-123", traits: { email: "user@example.com" } }
    → user_or_anonymous_id = "user-456" (优先使用 userId)

T3: 后续行为使用已知用户 ID
    { type: "track", userId: "user-456", event: "Purchase" }
    → user_or_anonymous_id = "user-456"
```

**关键观察**：
- T1 和 T2 的 `user_or_anonymous_id` 不同，但可通过 `anonymousId` 字段关联
- 查询时需要同时考虑 `user_id` 和 `anonymous_id` 字段

#### 3.3.4 内部事件中的身份字段

`internal_events` 表 (`clickhouse.ts:17-43`) 同时存储：

```sql
user_or_anonymous_id String,  -- 合并后的 ID
user_id String,               -- 原始 userId
anonymous_id String,          -- 原始 anonymousId
```

**用途**：
- `user_or_anonymous_id` 用于排序和分区
- `user_id` 和 `anonymous_id` 保留原始信息，支持身份追溯

### 3.4 群组关联 (Group 事件)

Dittofeed 支持用户与群组的关联，通过 Materialized View 实现：

```sql
-- userEvents/clickhouse.ts:104-129
create materialized view if not exists group_user_assignments_mv to group_user_assignments
as select
  workspace_id,
  uev.user_id as group_id,
  JSONExtractString(uev.properties, 'userId') as user_id,
  JSONExtractBool(uev.properties, 'assigned') as assigned
from user_events_v2 as uev
where
  uev.event_type = 'track'
  and uev.event = '${InternalEventType.GroupUserAssignment}'
```

**关联表**：
- `group_user_assignments`: group_id → user_ids
- `user_group_assignments`: user_id → group_ids

---

## 4. 多来源写入冲突处理

### 4.1 事件去重机制

#### 4.1.1 Message ID 去重

`user_events_v2` 表定义了 Bloom Filter 索引：

```sql
INDEX message_id_idx message_id TYPE bloom_filter(0.01) GRANULARITY 4
```

#### 4.1.2 查询时去重

在 `findUserEvents` 函数 (`userEvents.ts:342-371`) 中，当使用 `messageId` 过滤时：

```sql
AND (workspace_id, processing_time, user_or_anonymous_id, event_time, message_id) IN (
  SELECT
    workspace_id,
    max(processing_time),                    -- 取最新处理时间
    user_or_anonymous_id,
    argMax(event_time, processing_time),     -- 取最新事件时间
    message_id
  FROM user_events_v2
  WHERE ${workspaceIdClause} ${messageIdWhereClause}
  GROUP BY
    workspace_id,
    user_or_anonymous_id,
    message_id
)
```

**去重策略**：
- 按 `message_id` 分组
- 取 `max(processing_time)` 的记录
- 使用 `argMax(event_time, processing_time)` 获取对应事件时间

### 4.2 用户属性冲突处理

#### 4.2.1 ReplacingMergeTree 引擎

`computed_property_assignments_v2` 表使用 `ReplacingMergeTree`：

```sql
CREATE TABLE IF NOT EXISTS computed_property_assignments_v2 (
  workspace_id LowCardinality(String),
  type Enum('user_property' = 1, 'segment' = 2),
  computed_property_id LowCardinality(String),
  user_id String,
  segment_value Boolean,
  user_property_value String,
  max_event_time DateTime64(3),
  assigned_at DateTime64(3) DEFAULT now64(3),
)
ENGINE = ReplacingMergeTree()
ORDER BY (
  workspace_id,
  type,
  computed_property_id,
  user_id
);
```

**冲突解决**：
- 主键：`(workspace_id, type, computed_property_id, user_id)`
- 同一主键的多条记录，`ReplacingMergeTree` 在合并时保留最新版本
- 合并触发时机：后台 Merge 操作，非实时

#### 4.2.2 查询时 argMax 策略

在 `findAllUserPropertyAssignments` 函数 (`userProperties.ts:418-433`)：

```sql
select
  computed_property_id,
  argMax(user_property_value, assigned_at) as last_value  -- 取最新赋值
from computed_property_assignments_v2
where
  workspace_id = ${workspaceId}
  and user_id = ${userId}
  and type = 'user_property'
group by computed_property_id
having last_value != ''
```

**时间戳优先级**：
- 使用 `argMax(value, timestamp)` 确保获取最新值
- `assigned_at` 字段由 `now64(3)` 自动生成，毫秒级精度

#### 4.2.3 顺序一致性配置

支持可选的顺序一致性读取：

```typescript
// userProperties.ts:437-439
clickhouse_settings: {
  select_sequential_consistency: assignmentSequentialConsistency(),
}
```

通过 `config.assignmentSequentialConsistency` 配置控制。

### 4.3 多来源写入场景处理

#### 4.3.1 同一属性多来源更新

**场景**：Web SDK 和 Mobile SDK 同时更新同一用户的 `email` 属性

```
来源 A (Web):
  { userId: "user-1", traits: { email: "web@example.com" }, timestamp: "2024-01-01T10:00:00Z" }
  → assigned_at = T1

来源 B (Mobile):
  { userId: "user-1", traits: { email: "mobile@example.com" }, timestamp: "2024-01-01T10:05:00Z" }
  → assigned_at = T2 (T2 > T1)
```

**解决结果**：
- 查询时 `argMax(email, assigned_at)` 返回 `"mobile@example.com"`
- 以**服务器处理时间** `assigned_at` 为准，而非客户端 `timestamp`

#### 4.3.2 不同属性多来源合并

**场景**：Web SDK 更新 `name`，Mobile SDK 更新 `phone`

```
来源 A (Web):
  { userId: "user-1", traits: { name: "Alice" } }

来源 B (Mobile):
  { userId: "user-1", traits: { phone: "+1234567890" } }
```

**解决结果**：
- 两个属性独立存储在 `computed_property_assignments_v2`
- 查询时合并返回：`{ name: "Alice", phone: "+1234567890" }`
- **无冲突**，属性级独立更新

#### 4.3.3 匿名与已知用户属性合并

**场景**：匿名阶段收集 `device`，登录后补充 `email`

```
T1 (匿名):
  { anonymousId: "anon-123", traits: { device: "iPhone" } }
  → user_or_anonymous_id = "anon-123"

T2 (登录):
  { userId: "user-456", anonymousId: "anon-123", traits: { email: "user@example.com" } }
  → user_or_anonymous_id = "user-456"
```

**当前实现限制**：
- 两个事件的 `user_or_anonymous_id` 不同
- 属性查询按 `user_id` 分组，T1 的 `device` 不会自动关联到 `user-456`
- **需要应用层处理身份关联**

### 4.4 批量写入处理

通过 `submitBatch` 函数 (`apps/batch.ts:89-111`) 处理批量事件：

```typescript
export async function submitBatch(
  { workspaceId, data }: SubmitBatchOptions,
  { processingTime, writeModeOverride } = {},
) {
  const { batchChunkSize } = config();
  const chunks = R.chunk(data.batch, batchChunkSize);

  await Promise.all(
    chunks.map(async (chunk) => {
      const chunkData = { ...data, batch: chunk };
      return submitBatchChunk({ workspaceId, data: chunkData }, { processingTime, writeModeOverride });
    }),
  );
}
```

**批量特性**：
- 按 `batchChunkSize` 分块
- 并行处理多个块
- 每个块内事件顺序处理

### 4.5 计算属性增量更新

通过 `computePropertiesIncremental.ts` 实现增量计算：

**状态管理**：
- `computed_property_state_v3`: 存储计算中间状态
- `updated_computed_property_state`: 追踪最近更新的属性
- `resolved_segment_state`: 解析后的 Segment 状态

**增量查询模式**：
```sql
-- 只处理最近更新的用户
where
  (workspace_id, user_id) in (
    select workspace_id, user_id
    from updated_computed_property_state
    where ...
  )
```

---

## 5. 架构总结

### 5.1 数据流

```
多客户端 SDK
    │
    ▼
┌─────────────────┐
│  API 鉴权层      │ ← validateWriteKey
│  (Basic Auth)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  事件接收层      │ ← submitBatch, submitTrack
│  (批量处理)      │
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│  写入层                  │
│  ┌─────────┐ ┌────────┐ │
│  │  Kafka  │ │  CH    │ │ ← insertUserEvents
│  │  Topic  │ │ Direct │ │
│  └────┬────┘ └───┬────┘ │
└───────┼──────────┼──────┘
        │          │
        ▼          ▼
┌─────────────────────────┐
│  user_events_v2         │ ← MergeTree, 原始事件存储
│  (所有事件持久化)         │
└───────────┬─────────────┘
            │
            ▼ (Materialized Views)
┌──────────────────────────────────────┐
│  派生表                               │
│  ├── computed_property_assignments    │ ← ReplacingMergeTree
│  ├── internal_events                 │
│  ├── group_user_assignments          │
│  └── user_property_index             │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  查询层                               │
│  ├── argMax 时间戳优先级              │
│  ├── message_id 去重                 │
│  └── user_or_anonymous_id 关联        │
└──────────────────────────────────────┘
```

### 5.2 关键设计决策

| 决策点 | 选择 | 理由 |
|-------|------|-----|
| 存储引擎 | MergeTree + ReplacingMergeTree | 高吞吐写入 + 最终一致性 |
| 冲突解决 | 时间戳优先级 (argMax) | 简单可靠，适合异步场景 |
| 身份关联 | user_or_anonymous_id + 双 ID 保留 | 灵活支持匿名→已知用户过渡 |
| 事件去重 | message_id + 查询时聚合 | 避免唯一键开销，支持高吞吐 |
| 多入口 | 多 Write Key + Workspace 隔离 | 安全性和可管理性 |

### 5.3 潜在优化点

1. **Alias 事件处理**：当前支持 `alias` 事件类型，但未实现自动身份合并的 Materialized View
2. **跨 ID 属性关联**：匿名用户属性不会自动迁移到已知用户
3. **实时一致性**：依赖后台 Merge 和查询时 argMax，非强一致性

---

## 6. 附录：核心文件索引

| 功能 | 文件路径 | 关键函数/定义 |
|-----|---------|-------------|
| API 鉴权 | `packages/backend-lib/src/auth.ts` | `validateWriteKey`, `getOrCreateWriteKey` |
| 事件写入 | `packages/backend-lib/src/userEvents.ts` | `insertUserEvents`, `findUserEvents` |
| 表结构定义 | `packages/backend-lib/src/userEvents/clickhouse.ts` | `createUserEventsTables` |
| 批量处理 | `packages/backend-lib/src/apps/batch.ts` | `submitBatch`, `buildBatchUserEvents` |
| 用户属性 | `packages/backend-lib/src/userProperties.ts` | `findAllUserPropertyAssignments` |
| 计算属性 | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 增量计算逻辑 |
| 事件类型 | `packages/isomorphic-lib/src/types.ts` | `EventType`, `InternalEventType` |
