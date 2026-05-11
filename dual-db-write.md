# 双数据库协同写入机制分析

## 一、架构概述

DittoFeed 采用 ClickHouse + PostgreSQL 双数据库架构，实现不同数据类型的优化存储：

- **ClickHouse**：事件流存储（user_events_v2 等）
- **PostgreSQL**：元数据管理（UserProperty、Segment、Workspace 等）

## 二、完整协同写入链路

### 2.1 数据流总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        事件入口层（API / SDK）                           │
│                                                                         │
│  messageId = UUID（幂等键）                                              │
│  messageRaw = 原始事件 JSON（备份+回溯）                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     阶段一：事件写入 ClickHouse                          │
│                                                                         │
│  目标表：user_events_v2（MergeTree）                                     │
│  写入模式：ch-sync / ch-async / kafka                                   │
│  写入字段：processing_time = now（服务端处理时间戳）                      │
│                                                                         │
│  幂等层：                                                                │
│    ┌─ 应用层：messageId 唯一标识                                         │
│    ├─ 查询层：argMax + GROUP BY 去重                                    │
│    └─ 引擎层：MergeTree 按 ORDER BY 有序存储                            │
│                                                                         │
│  恢复点：无 Period 表，依赖调用方重试 + messageId 保证幂等               │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ 定时调度（每 2 分钟）
┌─────────────────────────────────────────────────────────────────────────┐
│                      阶段二：计算中间状态（computeState）                 │
│                                                                         │
│  目标表：computed_property_state_v3（AggregatingMergeTree）             │
│  时间边界：processing_time >= periodBound                               │
│  写入字段：computed_at = now                                            │
│                                                                         │
│  幂等层：                                                                │
│    ┌─ 引擎层：AggregatingMergeTree + argMaxState/uniqState              │
│    ├─ 进度层：Period 表记录 maxTo（now）                                │
│    └─ 查询层：lowerBoundClause 只处理 periodBound 之后的事件             │
│                                                                         │
│  恢复点：periodBound = 上一轮 Period.maxTo                              │
│          只处理 processing_time >= periodBound 的事件                    │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   阶段三：计算属性赋值（computeAssignments）              │
│                                                                         │
│  目标表：computed_property_assignments_v2（ReplacingMergeTree）         │
│  时间边界：computed_at >= periodBound（针对 state_v3）                  │
│  写入字段：assigned_at = now                                            │
│                                                                         │
│  幂等层：                                                                │
│    ┌─ 引擎层：ReplacingMergeTree 自动去重同 ORDER BY 键记录              │
│    ├─ 进度层：Period 表记录 maxTo（now）                                │
│    └─ 版本层：shouldReset 判断定义是否更新                               │
│                                                                         │
│  恢复点：periodBound = 上一轮 Period.maxTo                              │
│          只处理 computed_at >= periodBound 的状态记录                    │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   阶段四：处理赋值（processAssignments）                  │
│                                                                         │
│  目标表：processed_computed_properties_v2（ReplacingMergeTree）         │
│  时间边界：assigned_at >= periodBound                                   │
│             └─ 但仅当 periodBound > processedForUpdatedAt 时启用        │
│  写入字段：processed_at = now                                           │
│                                                                         │
│  执行顺序：                                                              │
│    1. Promise.all([                                                      │
│         triggerSegmentEntryJourney(),    // 触发 Journey 下游           │
│         startHubspotUserIntegrationWorkflow()    // 触发 Integration 下游        │
│       ])                                                                 │
│    2. await insertProcessedComputedProperties()  // 写入处理记录         │
│                                                                         │
│  幂等层：                                                                │
│    ┌─ 去重表：pcp 表记录已处理的 user_id + 值                            │
│    ├─ JOIN 过滤：LEFT ANY JOIN pcp 过滤已处理                            │
│    ├─ 值变化检测：cpa.value != pcp.value 只处理变化                      │
│    └─ 进度层：Period 表记录 maxTo（now）                                │
│                                                                         │
│  恢复点：periodBound = 上一轮 Period.maxTo                              │
│          + pcp 表记录已处理值（双重保障）                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

## 三、事件流为什么落到 ClickHouse

### 3.1 数据特征分析

事件流具有以下特征：
- **高吞吐量**：用户行为事件（track/identify/page/screen/group/alias）每秒可能产生大量数据
- **时间序列**：事件带有时间戳，查询多按时间范围过滤
- **写多读少**：写入频率远高于读取频率
- **聚合分析**：主要用于 OLAP 场景（用户分群、漏斗分析、趋势统计）

### 3.2 ClickHouse 的技术优势

**1. MergeTree 引擎家族**

```typescript
// userEvents/clickhouse.ts:299-364
CREATE TABLE IF NOT EXISTS user_events_v2 (
  event_type Enum(...),
  event String,
  event_time DateTime64,
  message_id String,
  user_id String,
  anonymous_id String,
  user_or_anonymous_id String,
  properties String,
  hidden Boolean,
  processing_time DateTime64(3) DEFAULT now64(3),
  server_time DateTime64(3),
  message_raw String,
  workspace_id String,
  INDEX message_id_idx message_id TYPE bloom_filter(0.01) GRANULARITY 4
)
ENGINE = MergeTree()
ORDER BY (workspace_id, processing_time, user_or_anonymous_id, event_time, message_id);
```

- **按主键有序存储**：`ORDER BY` 子句决定物理存储顺序，极大优化范围查询
- **列存储**：每列独立存储，聚合查询只需扫描相关列
- **稀疏索引**：每个 primary key 粒度（granularity）只存一个索引项，索引体积小
- **数据合并**：后台异步合并小文件，优化长期存储

**2. 异步写入支持**

```typescript
// userEvents.ts:76-87
const settings: ClickHouseSettings = {
  async_insert: asyncInsert ? 1 : undefined,
  wait_for_async_insert: asyncInsert ? 1 : undefined,
  wait_end_of_query: asyncInsert ? undefined : 1,
};

await clickhouseClient().insert({
  table: `user_events_v2 (message_raw, processing_time, workspace_id, message_id, server_time)`,
  values,
  clickhouse_settings: settings,
  format: "JSONEachRow",
});
```

三种写入模式（config.ts:544-545）：
- `ch-sync`：同步写入，等待查询完成
- `ch-async`：异步插入，客户端无需等待
- `kafka`：通过 Kafka 消息队列解耦

**3. 物化视图自动派生**

```typescript
// userEvents/clickhouse.ts:45-69
CREATE MATERIALIZED VIEW IF NOT EXISTS internal_events_mv
TO internal_events
AS SELECT
  workspace_id,
  user_or_anonymous_id,
  user_id,
  anonymous_id,
  message_id,
  event,
  event_time,
  processing_time,
  properties,
  JSONExtractString(properties, 'templateId') as template_id,
  ...
FROM user_events_v2
WHERE event_type = 'track' AND startsWith(event, 'DF');
```

- 数据写入主表后自动派生到目标表
- 支持复杂的 JSON 字段提取和类型转换
- 减轻应用层处理负担

**4. ReplacingMergeTree 去重能力**

```typescript
// userEvents/clickhouse.ts:380-397
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
ORDER BY (workspace_id, type, computed_property_id, user_id);
```

- 相同 `ORDER BY` 键的重复记录会在后台合并时自动去重
- 保留最新版本（默认按插入顺序）
- 天然支持幂等写入场景

### 3.3 为什么不是 Postgres？

| 对比维度 | PostgreSQL | ClickHouse |
|---------|-----------|------------|
| 单条插入性能 | 中等（需 WAL、索引维护） | 优秀（批量写入、后台合并） |
| 高并发写入 | 有限（锁竞争、连接池） | 优秀（面向批量的设计） |
| 聚合查询性能 | 中等（行存储） | 优秀（列存储 + 向量化执行） |
| 时间序列优化 | 需手动分区索引 | 原生支持 |
| 存储成本 | 较高（行存储 + 多索引） | 较低（列存储 + 高效压缩） |

## 四、元数据为什么放在 PostgreSQL

### 4.1 元数据特征分析

元数据包括：
- 用户属性定义（UserProperty）：名称、定义、状态
- 人群定义（Segment）：规则、状态
- 工作空间（Workspace）：基本信息
- 消息模板（MessageTemplate）：内容定义
- 集成配置（Integration）：第三方连接配置

这些数据具有以下特征：
- **强一致性要求**：需要 ACID 事务保证
- **关系复杂**：多表关联查询频繁
- **读写相对均衡**：CRUD 操作都有
- **业务规则约束**：唯一性、外键等

### 4.2 PostgreSQL 的技术优势

**1. 关系型数据建模**

```typescript
// db/schema.ts:151-184
export const userProperty = pgTable(
  "UserProperty",
  {
    id: uuid().primaryKey().defaultRandom().notNull(),
    workspaceId: uuid().notNull(),
    name: text().notNull(),
    definition: jsonb().notNull(),
    createdAt: timestamp(...).defaultNow().notNull(),
    updatedAt: timestamp(...).defaultNow().$onUpdate(() => new Date()).notNull(),
    resourceType: dbResourceType().default("Declarative").notNull(),
    definitionUpdatedAt: timestamp(...).defaultNow().notNull(),
    status: userPropertyStatus().default("Running").notNull(),
    exampleValue: text(),
  },
  (table) => [
    uniqueIndex("UserProperty_workspaceId_name_key").using(...),
    foreignKey(...).onUpdate("cascade").onDelete("cascade"),
  ],
);
```

- 清晰的外键关系保证数据完整性
- 唯一索引（uniqueIndex）防止重复定义
- 级联删除（onDelete("cascade")）自动清理关联数据

**2. 事务与并发控制**

```typescript
// db.ts:129-156
export async function upsert<TTable extends Table, ...>({
  table,
  values,
  tx: txArg,
  ...onConflict
}: ...): Promise<Result<TTable["$inferSelect"], QueryError>> {
  const tx = txArg ?? db();
  const results = await queryResult(
    tx.insert(table).values(values).onConflictDoUpdate(onConflict).returning(),
  );
  // ...
}
```

- `onConflictDoUpdate` 实现原子性 upsert
- 支持显式事务（`tx` 参数）
- MVCC 提供读已提交隔离级别

**3. 丰富的数据类型**

```typescript
// db/schema.ts:16-20
export const computedPropertyType = pgEnum("ComputedPropertyType", [
  "Segment",
  "UserProperty",
]);
```

- 枚举类型（pgEnum）保证字段值合法性
- JSONB 类型存储灵活的定义数据
- UUID 主键保证分布式唯一性

### 4.3 为什么不是 ClickHouse？

| 对比维度 | ClickHouse | PostgreSQL |
|---------|-----------|------------|
| 事务支持 | 有限（无完整 ACID） | 完整支持 |
| 外键约束 | 不支持 | 原生支持 |
| 行级更新 | 弱（Mutation 代价高） | 优秀 |
| 小表随机读 | 弱（面向批量设计） | 优秀 |
| 复杂查询优化 | OLAP 优化器 | OLTP 优化器 |

## 五、协同写入链路的幂等与恢复详解

### 5.0 关键概念：Period 表

在深入各阶段之前，先理解 Postgres 中 Period 表的作用：

**Postgres Period 表结构**：

```typescript
// db/schema.ts:200-220
export const computedPropertyPeriod = pgTable(
  "ComputedPropertyPeriod",
  {
    id: uuid().primaryKey().defaultRandom().notNull(),
    workspaceId: uuid().notNull(),
    type: computedPropertyType().notNull(),
    computedPropertyId: uuid().notNull(),
    from: timestamp({ withTimezone: true, mode: "string" }),  // 上一轮的 to
    to: timestamp({ withTimezone: true, mode: "string" }).notNull(),  // 本轮的 now
    step: computedPropertyStep().notNull(),  // "ComputeState" | "ComputeAssignments" | "ProcessAssignments"
    version: text().notNull(),  // definitionUpdatedAt.toString()
    createdAt: timestamp(...).defaultNow().notNull(),
  },
  ...
);
```

**Period 查询逻辑**（getPeriodsByComputedPropertyId）：

```typescript
// periods.ts:166-182
SELECT DISTINCT ON (workspaceId, type, computedPropertyId)
  type,
  computedPropertyId,
  version,
  MAX(to) OVER (
    PARTITION BY workspaceId, type, computedPropertyId
  ) as maxTo
FROM ComputedPropertyPeriod
WHERE step = ${step}
ORDER BY workspaceId, type, computedPropertyId, to DESC
```

- 每个 step（ComputeState / ComputeAssignments / ProcessAssignments）独立维护进度
- 返回的是 **当前 version** 的 maxTo（version 来自 definitionUpdatedAt）
- `get({ version, computedPropertyId })` 按 version 匹配

**Period 创建逻辑**（createPeriods）：

```typescript
// periods.ts:213-229
for (const segment of segments) {
  const version = segment.definitionUpdatedAt.toString();
  const previousPeriod = periodByComputedPropertyId?.get({
    version,
    computedPropertyId: segment.id,
  });
  newPeriods.push({
    from: previousPeriod ? previousPeriod.maxTo : null,  // 上一轮的 to
    to: nowD,                                              // 本轮的 now
    version,
    ...
  });
}
```

- `from` = 上一轮同 version 的 `to`
- `to` = 本轮调度开始时的 `now`
- 每轮结束时写入 Postgres

---

### 5.1 阶段一：事件写入 ClickHouse

#### 5.1.1 写入流程

```
调用方（API / SDK）
    │
    ▼
生成 messageId（UUID）
    │
    ▼
写入 user_events_v2：
    processing_time = now64(3)  // 服务端处理时间戳
    message_raw = 原始 JSON
    message_id = UUID
    │
    ▼
物化视图自动派生：
    internal_events_mv → internal_events
    group_user_assignments_mv → group_user_assignments
    ...
```

#### 5.1.2 幂等机制

| 层级 | 机制 | 实现位置 |
|-----|------|---------|
| **应用层** | messageId 唯一标识 | `userEvents.ts:36` |
| **查询层** | argMax + GROUP BY 去重 | `userEvents.ts:352-371` |
| **引擎层** | MergeTree 按 ORDER BY 有序存储 | `userEvents/clickhouse.ts:364` |

**查询去重逻辑**：

```typescript
// userEvents.ts:352-371
messageIdClause = `
  AND (workspace_id, processing_time, user_or_anonymous_id, event_time, message_id) IN (
    SELECT
      workspace_id,
      max(processing_time),
      user_or_anonymous_id,
      argMax(event_time, processing_time),
      message_id
    FROM user_events_v2
    WHERE ${workspaceIdClause}
      AND message_id = ${messageIdParam}
    GROUP BY workspace_id, user_or_anonymous_id, message_id
  )
`;
```

#### 5.1.3 失败恢复

| 写入模式 | 失败场景 | 恢复方式 |
|---------|---------|---------|
| `kafka` | Kafka 消费失败 | Kafka offset 回溯重放 |
| `ch-async` | 客户端成功但服务端失败 | 需调用方重试，messageId 保证幂等 |
| `ch-sync` | 网络超时/服务端错误 | 调用方重试，messageId 保证幂等 |

> **关键点**：阶段一没有 Period 表。事件一旦写入成功，后续阶段从 ClickHouse 读取 `processing_time` 作为时间锚点。

---

### 5.2 阶段二：计算中间状态（computeState）

#### 5.2.1 处理流程

```
定时调度器触发（每 2 分钟）
    │
    ├─ now = 当前时间戳
    │
    ▼
从 Postgres 读取 Segments/UserProperties 定义
    │
    ▼
从 Postgres Period 表读取上一轮进度：
    period = getPeriodsByComputedPropertyId(step="ComputeState")
    periodBound = period?.maxTo.getTime()  // 上一轮的 to
    │
    ▼
构建 lowerBoundClause：
    period > 0
        ? `and processing_time >= toDateTime64(${periodBound / 1000}, 3)`
        : ""  // 冷启动，无边界
    │
    ▼
INSERT INTO computed_property_state_v3
SELECT
  ...
  argMaxState(last_value, ue.event_time),
  uniqState(unique_value),
  ...
  toDateTime64(${now / 1000}, 3) as computed_at  // 本轮 now
FROM user_events_v2 ue
WHERE
  workspace_id = ${workspaceId}
  AND processing_time <= toDateTime64(${now / 1000}, 3)  // 截止到 now
  AND (${lowerBoundClause})  // 从 periodBound 开始
  AND (segment/property 条件)
GROUP BY ...
    │
    ▼
写入 Postgres Period 表：
    step = "ComputeState"
    from = previousPeriod?.maxTo  // 上一轮的 to
    to = now  // 本轮的 now
    version = segment.definitionUpdatedAt.toString()
```

#### 5.2.2 幂等机制

| 层级 | 机制 | 实现位置 |
|-----|------|---------|
| **表引擎** | AggregatingMergeTree | `userEvents/clickhouse.ts:572` |
| **聚合函数** | argMaxState、uniqState 幂等聚合 | `computePropertiesIncremental.ts:3129-3132` |
| **进度跟踪** | Period 表 + lowerBoundClause | `computePropertiesIncremental.ts:3094-3097` |

**时间边界口径**：
- **过滤条件**：`processing_time >= periodBound`（针对 user_events_v2）
- **截止条件**：`processing_time <= now`
- **写入字段**：`computed_at = now`

**AggregatingMergeTree 幂等**：

```typescript
// userEvents/clickhouse.ts:572-595
CREATE TABLE IF NOT EXISTS computed_property_state_v3 (
  workspace_id LowCardinality(String),
  type Enum('user_property' = 1, 'segment' = 2),
  computed_property_id LowCardinality(String),
  state_id LowCardinality(String),
  user_id String,
  last_value SimpleAggregateFunction(argMax, String, DateTime64(3)),
  unique_value SimpleAggregateFunction(uniq, String),
  truncated_event_time DateTime64(3),
  grouped_message_id SimpleAggregateFunction(groupArray, Array(String)),
  computed_at DateTime64(3),
)
ENGINE = AggregatingMergeTree()
ORDER BY (workspace_id, type, computed_property_id, state_id, user_id, truncated_event_time);
```

`argMaxState`、`uniqState` 是幂等聚合函数：
- 多次聚合结果一致
- 重复插入不改变最终结果

#### 5.2.3 失败恢复

```
本次调度失败（computeState 抛出异常）
    │
    │ 此时：
    │   - computed_property_state_v3 可能已部分写入
    │   - Postgres Period 表还未更新
    │   - createPeriods 在 computeState 最后执行
    │
    ▼
下次调度
    │
    ├─ now' = 新的当前时间戳
    ├─ getPeriodsByComputedPropertyId() → 返回上一轮成功的 Period
    ├─ periodBound = 上一轮成功的 maxTo（没变）
    ├─ lowerBoundClause = `processing_time >= ${periodBound}`
    ├─ 截止条件 = `processing_time <= now'`
    │
    ├─ 重复处理范围：[periodBound, now) 区间内的事件
    │   └─ 但 AggregatingMergeTree + 幂等聚合函数保证结果一致
    │
    └─ 新增处理范围：[now, now') 区间内的新事件
```

> **关键点**：
> 1. Period 写入在阶段最后。如果中间失败，下次从断点继续
> 2. 重复处理 [periodBound, now) 区间是安全的：幂等聚合函数 + AggregatingMergeTree 保证最终一致

---

### 5.3 阶段三：计算属性赋值（computeAssignments）

#### 5.3.1 处理流程

```
从 Postgres Period 表读取进度：
    period = getPeriodsByComputedPropertyId(step="ComputeAssignments")
    periodBound = period?.maxTo.getTime()
    │
    ├─ 是否需要重置？
    │   shouldReset = definitionUpdatedAt > periodBound
    │       && definitionUpdatedAt <= now
    │       && definitionUpdatedAt > createdAt
    │   ├─ 是 → lowerBoundClause = ""（全量重算）
    │   └─ 否 → 继续增量计算
    │
    ▼
构建 lowerBoundClause（针对 computed_property_state_v3）：
    function getLowerBoundClause(bound?: number): string {
      return bound && bound > 0
        ? `and computed_at >= toDateTime64(${bound / 1000}, 3)`
        : "";
    }
    │
    ▼
INSERT INTO computed_property_assignments_v2
SELECT
  ...
  toDateTime64(${now / 1000}, 3) as assigned_at  // 本轮 now
FROM resolved_segment_state  // 从 computed_property_state_v3 派生
WHERE
  computed_at <= toDateTime64(${now / 1000}, 3)  // 截止到 now
  ${lowerBoundClause}  // 从 periodBound 开始
    │
    ▼
写入 Postgres Period 表：
    step = "ComputeAssignments"
    from = previousPeriod?.maxTo
    to = now
    version = segment.definitionUpdatedAt.toString()
```

#### 5.3.2 幂等机制

| 层级 | 机制 | 实现位置 |
|-----|------|---------|
| **表引擎** | ReplacingMergeTree | `userEvents/clickhouse.ts:396` |
| **进度跟踪** | Period 表 + lowerBoundClause | `computePropertiesIncremental.ts:3295` |
| **版本控制** | definitionUpdatedAt 触发重置 | `computePropertiesIncremental.ts:3326-3331` |

**时间边界口径**：
- **过滤条件**：`computed_at >= periodBound`（针对 computed_property_state_v3）
- **截止条件**：`computed_at <= now`
- **写入字段**：`assigned_at = now`

> **注意**：这里的 time field 是 `computed_at`（阶段二写入的时间戳），不是 `processing_time`

**ReplacingMergeTree 去重**：

```sql
ENGINE = ReplacingMergeTree()
ORDER BY (workspace_id, type, computed_property_id, user_id)
```

- 相同 `(workspace_id, type, computed_property_id, user_id)` 的记录
- 后台合并时只保留最新版本（`assigned_at` 最大的）
- 重复插入自动去重

**定义更新检测**：

```typescript
// computePropertiesIncremental.ts:129-148
function shouldResetComputedProperty({
  definitionUpdatedAt,
  createdAt,
  now,
  periodBound,
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

- `definitionUpdatedAt > periodBound`：定义更新时间在上次处理时间之后
- 需要全量重新计算（不使用 lowerBoundClause）

#### 5.3.3 失败恢复

```
场景 A：正常失败（无定义更新）

本次调度失败
    │
    │ Period 表未更新
    │
    ▼
下次调度
    │
    ├─ periodBound = 上一轮成功的 maxTo
    ├─ lowerBoundClause = `computed_at >= ${periodBound}`
    ├─ 只增量处理 computed_at >= periodBound 的状态记录
    └─ 重复赋值由 ReplacingMergeTree 自动去重（保留 assigned_at 最大的）
```

```
场景 B：定义更新（shouldReset = true）

定义更新：
    Postgres: definitionUpdatedAt = T_update

本次调度（now = T_now > T_update）
    │
    ├─ periodBound = 旧版本的 maxTo（< T_update）
    ├─ shouldReset = (T_update > periodBound) = true
    ├─ lowerBoundClause = ""（全量重算）
    │
    ├─ INSERT 全量数据到 computed_property_assignments_v2
    │   └─ 新记录的 assigned_at = T_now（更大）
    │
    └─ 写入 Period 表：
        version = T_update.toString()  // 新版本号
        to = T_now
```

```
场景 C：定义更新后再次调度

下次调度
    │
    ├─ currentVersion = T_update.toString()
    ├─ getPeriodsByComputedPropertyId(version=currentVersion)
    │   └─ 返回版本 T_update 的 Period
    ├─ periodBound = T_now（上一轮的 to）
    ├─ shouldReset = (T_update > T_now) = false
    ├─ 恢复增量计算
    └─ lowerBoundClause = `computed_at >= ${T_now}`
```

> **关键点**：
> 1. 定义更新会触发全量重算，但 ReplacingMergeTree 保证最终只保留最新版本
> 2. Period 的 version 与 definitionUpdatedAt 绑定，定义更新后旧 version 的 Period 不再使用

---

### 5.4 阶段四：处理赋值（processAssignments）

#### 5.4.1 处理流程

```
从 Postgres Period 表读取进度：
    period = getPeriodsByComputedPropertyId(step="ProcessAssignments")
    periodBound = period?.maxTo.getTime()
    │
    ▼
构建 AssignmentProcessor 列表（每个 journey/integration 一个）：
    processed_for = journeyId / integrationName
    processed_for_type = "journey" / "integration"
    processedForUpdatedAt = journey.updatedAt / integration.updatedAt
    │
    ▼
AssignmentProcessor.process() 分页处理：
    cursor = null
    while (retrieved >= pageSize)
        │
        ▼
        构建查询：
            FROM computed_property_assignments_v2 cpa
            LEFT ANY JOIN processed_computed_properties_v2 pcp
                ON cpa.user_id = pcp.user_id
            WHERE
                assigned_at >= periodBound  -- 仅当 periodBound > processedForUpdatedAt
                AND cpa.value != pcp.value   -- 值变化才处理
        │
        ▼
        processRows(rows)：
            ┌─ 步骤 1：分类
            │    journeySegmentAssignments / integrationAssignments
            │
            ├─ 步骤 2：并行触发下游
            │    await Promise.all([
            │      triggerSegmentEntryJourney(),      // 旅程工作流
            │      startHubspotUserIntegration()      // HubSpot 集成
            │    ])
            │
            └─ 步骤 3：写入处理记录
                 await insertProcessedComputedProperties({
                   assignments: processedAssignments
                 })
        │
        ▼
        cursor = lastUserId
    │
    ▼
写入 Postgres Period 表：
    step = "ProcessAssignments"
    from = previousPeriod?.maxTo
    to = now
    version = segment.definitionUpdatedAt.toString()
```

#### 5.4.2 执行顺序（关键！）

**代码实现**（processRowsInner）：

```typescript
// computePropertiesIncremental.ts:3650-3750

// 步骤 1：分类
const journeySegmentAssignments: ComputedAssignment[] = [];
const integrationAssignments: ComputedAssignment[] = [];
for (const assignment of assignments) {
  // ...分类到两个数组
}

// 步骤 2：并行触发下游（Promise.all）
await Promise.all([
  ...journeySegmentAssignments.flatMap((assignment) => {
    return triggerSegmentEntryJourney({...});
  }),
  ...integrationAssignments.flatMap(async (assignment) => {
    return startHubspotUserIntegrationWorkflow({...});
  }),
]);

// 步骤 3：写入处理记录（在下游触发之后）
const processedAssignments: ComputedPropertyAssignment[] =
  assignments.flatMap((assignment) => ({
    user_property_value: assignment.latest_user_property_value,
    segment_value: assignment.latest_segment_value,
    ...assignment,
  }));

await insertProcessedComputedProperties({
  assignments: processedAssignments,
});

return cursor;
```

**执行顺序**：
1. **先**：`Promise.all([下游触发...])`
2. **后**：`await insertProcessedComputedProperties()`

**语义**：
- "至少一次"（at-least-once）
- 如果下游触发成功但 pcp 写入失败：下次调度会重新触发下游
- 下游系统需要自己处理幂等（如 journey 的消息去重）

#### 5.4.3 幂等机制

| 层级 | 机制 | 实现位置 |
|-----|------|---------|
| **去重表** | processed_computed_properties_v2 | `computePropertiesIncremental.ts:3879-3901` |
| **JOIN 过滤** | LEFT ANY JOIN pcp 过滤已处理 | `computePropertiesIncremental.ts:3851-3910` |
| **值变化检测** | `cpa.value != pcp.value` | `computePropertiesIncremental.ts:3899-3900` |
| **分页游标** | cursor = lastUserId | `computePropertiesIncremental.ts:3951` |
| **表引擎** | ReplacingMergeTree | `userEvents/clickhouse.ts:407-427` |

**时间边界口径**：

```typescript
// computePropertiesIncremental.ts:3823-3833
const period = periodByComputedPropertyId.get({
  computedPropertyId,
  version: computedPropertyVersion,
});
const periodBound = period?.maxTo.getTime();

// 注意：仅当 periodBound > processedForUpdatedAt 时启用边界
const lowerBoundClause =
  periodBound && periodBound > 0 && periodBound > processedForUpdatedAt
    ? `and assigned_at >= toDateTime64(${periodBound / 1000}, 3)`
    : "";
```

- **过滤条件**：`assigned_at >= periodBound`（针对 computed_property_assignments_v2）
- **限制条件**：仅当 `periodBound > processedForUpdatedAt` 时启用
- **原因**：如果 journey/integration 被更新过（updatedAt > periodBound），可能需要重新处理旧赋值

**核心去重查询**：

```typescript
// computePropertiesIncremental.ts:3851-3910
const query = `
  SELECT cpa.*, pcp.*
  FROM (
    SELECT user_id,
           argMax(segment_value, assigned_at) latest_segment_value,
           argMax(user_property_value, assigned_at) latest_user_property_value
    FROM computed_property_assignments_v2
    WHERE assigned_at >= ${periodBound}  -- 只处理新赋值（有限制条件）
    GROUP BY user_id
  ) cpa
  LEFT ANY JOIN (
    SELECT user_id,
           argMax(segment_value, processed_at) segment_value,
           argMax(user_property_value, processed_at) user_property_value
    FROM processed_computed_properties_v2
    WHERE processed_for = ${journeyId}      -- 区分不同处理目标
      AND processed_for_type = "journey"
    GROUP BY user_id
  ) pcp
  ON cpa.user_id = pcp.user_id
  WHERE cpa.latest_segment_value != pcp.segment_value  -- 值变化才处理
     OR cpa.latest_user_property_value != pcp.user_property_value
`;
```

**processed_computed_properties_v2 表结构**：

```typescript
// userEvents/clickhouse.ts:408-427
CREATE TABLE IF NOT EXISTS processed_computed_properties_v2 (
  workspace_id LowCardinality(String),
  user_id String,
  type Enum('user_property' = 1, 'segment' = 2),
  computed_property_id LowCardinality(String),
  processed_for LowCardinality(String),        -- journeyId / integrationName
  processed_for_type LowCardinality(String),   -- "journey" / "integration"
  segment_value Boolean,
  user_property_value String,
  max_event_time DateTime64(3),
  processed_at DateTime64(3) DEFAULT now64(3),
)
ENGINE = ReplacingMergeTree()
ORDER BY (
  workspace_id,
  computed_property_id,
  processed_for_type,
  processed_for,
  user_id
);
```

- `processed_for + processed_for_type` 区分不同下游（同一赋值可能触发多个下游）
- 每个下游独立维护自己的处理进度

#### 5.4.4 失败恢复

**场景 1：下游触发之前失败**

```
查询构建失败 / ClickHouse 查询失败
    │
    │ 还没执行 processRows
    │ processed_computed_properties_v2 没写入
    │
    ▼
下次调度
    │
    ├─ periodBound = 上一轮成功的 maxTo
    ├─ 全量查询未处理的赋值
    └─ JOIN pcp 过滤已处理（如果部分页面成功）
```

**场景 2：部分页面成功后失败**

```
第 1 页成功：
    ├─ 步骤 1：触发 journey 工作流 ✓
    ├─ 步骤 2：写入 processed_computed_properties_v2 ✓
    └─ pcp 记录：user_id_1 ~ user_id_100

第 2 页失败（下游触发失败 或 pcp 写入失败）
    │
    │ Period 表还没更新
    │
    ▼
下次调度
    │
    ├─ periodBound = 上一轮成功的 maxTo
    ├─ 全量查询（lowerBoundClause 可能启用也可能不启用）
    ├─ JOIN pcp 时，user_id_1 ~ 100 已匹配，被过滤
    ├─ 只处理 user_id_101 之后
    └─ cursor 重新从 null 开始，但实际只处理剩余数据
```

**场景 3：下游触发成功但 pcp 写入失败**

```
第 1 页：
    ├─ 步骤 1：triggerSegmentEntryJourney() ✓ （已触发）
    ├─ 步骤 2：insertProcessedComputedProperties() ✗ （失败）
    │
    └─ 结果：下游已触发，但 pcp 没记录

下次调度：
    │
    ├─ 同样的赋值会被再次查询（pcp 没记录）
    ├─ 同样的赋值会被再次处理
    ├─ 下游会被再次触发
    │
    └─ 依赖：下游系统自己处理幂等
         （如 journey 工作流的消息去重机制）
```

> **关键点**：
> 1. 执行顺序：先触发下游，再写入 pcp
> 2. 语义：至少一次（at-least-once）
> 3. 下游系统需要自己处理幂等
> 4. Period 表 + pcp 表双重保障恢复

---

### 5.5 幂等与恢复汇总表

| 阶段 | 步骤 | 时间边界字段 | lowerBoundClause 条件 | 幂等层 | 恢复点 |
|-----|------|------------|----------------------|--------|--------|
| **一** | 事件写入 | `processing_time` | 无 | messageId + MergeTree | 无，依赖调用方重试 |
| **二** | computeState | `processing_time` | `processing_time >= periodBound` | AggregatingMergeTree + Period | `period.maxTo` |
| **三** | computeAssignments | `computed_at` | `computed_at >= periodBound` | ReplacingMergeTree + Period + shouldReset | `period.maxTo` |
| **四** | processAssignments | `assigned_at` | `assigned_at >= periodBound`<br>（仅当 `periodBound > processedForUpdatedAt`） | pcp 表 + JOIN 过滤 + 值变化 + Period | `period.maxTo` + pcp 记录 |

## 六、最终一致性的收敛机制

### 6.1 各阶段收敛路径串联

#### 阶段一：事件写入 → 阶段二：状态计算

```
阶段一写入：
    user_events_v2:
        (messageId=1, processing_time=T1)
        (messageId=2, processing_time=T2)

阶段二调度（now=T3）：
    ├─ periodBound = 0（冷启动，无 lowerBoundClause）
    ├─ 处理范围：processing_time <= T3（全部）
    ├─ 写入 computed_property_state_v3: computed_at=T3
    └─ 写入 Period 表：to=T3, version=V1

阶段二再次调度（now=T4）：
    ├─ periodBound = T3
    ├─ lowerBoundClause: processing_time >= T3
    ├─ 截止条件：processing_time <= T4
    ├─ 只处理 processing_time >= T3 的事件
    └─ 写入 Period 表：from=T3, to=T4
```

**收敛点**：`period.maxTo` 单调递增，`processing_time >= periodBound` 只增量处理新事件

---

#### 阶段二：状态计算 → 阶段三：属性赋值

```
阶段二写入：
    computed_property_state_v3:
        (user_A, state=X, computed_at=T3)
        (user_B, state=Y, computed_at=T3)

阶段三调度（now=T3）：
    ├─ periodBound = 0（冷启动）
    ├─ 处理范围：computed_at <= T3（全部）
    ├─ 写入 computed_property_assignments_v2: assigned_at=T3
    └─ 写入 Period 表：to=T3, version=V1

阶段二再次调度（now=T4）：
    ├─ 写入新状态：computed_at=T4

阶段三再次调度（now=T4）：
    ├─ periodBound = T3
    ├─ lowerBoundClause: computed_at >= T3
    ├─ 截止条件：computed_at <= T4
    ├─ 只处理 computed_at >= T3 的状态记录
    └─ 写入 Period 表：from=T3, to=T4
```

**收敛点**：`computed_at >= periodBound` 只增量处理新状态

---

#### 阶段三：属性赋值 → 阶段四：赋值处理

```
阶段三写入：
    computed_property_assignments_v2:
        (user_A, segment_value=true, assigned_at=T3)
        (user_B, segment_value=false, assigned_at=T3)

阶段四调度（now=T3）：
    ├─ periodBound = 0（冷启动）
    ├─ lowerBoundClause 可能启用也可能不启用
    │   （取决于 periodBound > processedForUpdatedAt）
    ├─ JOIN pcp（空）→ 全部需要处理
    │
    ├─ 步骤 1：触发下游（journey / integration）
    ├─ 步骤 2：写入 pcp: processed_at=T3
    │       (user_A, segment_value=true, processed_for=J1)
    │       (user_B, segment_value=false, processed_for=J1)
    │
    └─ 写入 Period 表：to=T3

阶段三再次调度（now=T4）：
    ├─ 写入新赋值：
    │   (user_A, segment_value=false, assigned_at=T4)  ← 变化
    │   (user_C, segment_value=true, assigned_at=T4)   ← 新增

阶段四再次调度（now=T4）：
    ├─ periodBound = T3
    ├─ lowerBoundClause 可能启用
    │
    ├─ JOIN pcp 比较：
    │   user_A: false != true → 触发（变化）
    │   user_B: false == false → 跳过
    │   user_C: 无记录 → 触发（新增）
    │
    ├─ 触发下游（user_A 和 user_C）
    ├─ 写入 pcp 新值
    │
    └─ 写入 Period 表：from=T3, to=T4
```

**收敛点**：
1. `cpa.value != pcp.value` 只处理值变化
2. `assigned_at >= periodBound`（有限制条件）只处理新赋值
3. 双重过滤保证不会重复处理

---

### 6.2 定义更新时的收敛路径

```
初始状态：
    Postgres:
        UserProperty: definitionUpdatedAt=V0 (0)
    Period:
        [version=V0, maxTo=T1]
        [version=V0, maxTo=T2]

时刻 T3：定义更新
    Postgres:
        definitionUpdatedAt=V3 (T3)

时刻 T4：调度（now=T4）
    │
    ├─ 阶段二（computeState）：
    │   ├─ currentVersion = V3
    │   ├─ getPeriod(version=V3) → null
    │   ├─ periodBound = 0（冷启动）
    │   ├─ 全量重算：processing_time <= T4
    │   └─ Period: [version=V3, from=null, to=T4]
    │
    ├─ 阶段三（computeAssignments）：
    │   ├─ currentVersion = V3
    │   ├─ shouldReset = (V3 > periodBound=0) = true
    │   ├─ lowerBoundClause = ""（全量重算）
    │   └─ Period: [version=V3, from=null, to=T4]
    │
    └─ 阶段四（processAssignments）：
        ├─ currentVersion = V3
        ├─ getPeriod(version=V3) → null
        ├─ periodBound = 0
        ├─ 全量查询（lowerBoundClause 可能不启用）
        ├─ JOIN pcp（旧值）
        │   ├─ 值相同 → 跳过
        │   └─ 值不同 → 触发
        └─ Period: [version=V3, from=null, to=T4]

时刻 T5：再次调度（now=T5）
    │
    ├─ currentVersion = V3
    ├─ getPeriod(version=V3) → maxTo=T4
    ├─ periodBound = T4
    ├─ 恢复增量计算
    │   ├─ 阶段二：processing_time >= T4
    │   ├─ 阶段三：computed_at >= T4
    │   └─ 阶段四：assigned_at >= T4（有限制条件）
    │
    └─ 收敛完成，恢复正常增量处理
```

---

### 6.3 收敛条件汇总

| 机制 | 作用 | 实现 |
|-----|------|------|
| **Period monotonic** | `from` 永远是上一轮的 `to`，不会回退 | `periods.ts:219-229` |
| **Version 绑定** | Period 的 version 与 definitionUpdatedAt 绑定 | `periods.ts:214, 233` |
| **shouldReset** | 定义更新时触发全量重算 | `computePropertiesIncremental.ts:129-148` |
| **lowerBoundClause** | 只增量处理新数据 | 各阶段的条件拼接 |
| **pcp 值快照** | 记录已处理的值，只触发变化 | `processAssignments` 的 JOIN 逻辑 |
| **ReplacingMergeTree** | 重复写入取最新版本 | ClickHouse 表引擎 |
| **argMaxState/uniqState** | 幂等聚合，重复计算结果一致 | AggregatingMergeTree |

---

### 6.4 收敛时间线示例

```
时刻 T0:
  ├─ Postgres: definitionUpdatedAt=0 (V0)
  └─ Period: 空

时刻 T1: 第 1 轮调度（now=T1）
  ├─ 阶段一：事件 E1(processing_time=T1) 写入
  ├─ 阶段二：全量重算 → Period[V0, to=T1]
  ├─ 阶段三：全量重算 → Period[V0, to=T1]
  └─ 阶段四：全量处理 → Period[V0, to=T1], pcp[user_A, value=V]

时刻 T2: 第 2 轮调度（now=T2）
  ├─ 阶段一：事件 E2(processing_time=T2) 写入
  ├─ 阶段二：periodBound=T1 → 增量 [T1, T2]
  ├─ 阶段三：periodBound=T1 → 增量 [T1, T2]
  └─ 阶段四：periodBound=T1 → 增量 + 值变化检测

时刻 T3: 定义更新
  └─ Postgres: definitionUpdatedAt=T3 (V3)

时刻 T4: 第 3 轮调度（now=T4）
  ├─ currentVersion = V3
  ├─ 阶段二：getPeriod(V3)=null → periodBound=0 → 全量
  ├─ 阶段三：shouldReset=true → 全量
  ├─ 阶段四：getPeriod(V3)=null → periodBound=0 → 全量查询
  │              ├─ 值相同 → 跳过
  │              └─ 值不同 → 触发
  └─ Period[V3, to=T4] 写入

时刻 T5: 第 4 轮调度（now=T5）
  ├─ currentVersion = V3
  ├─ periodBound = T4
  ├─ 恢复增量计算
  │   ├─ 阶段二：processing_time >= T4
  │   ├─ 阶段三：computed_at >= T4
  │   └─ 阶段四：assigned_at >= T4（有限制条件）
  └─ Period[V3, from=T4, to=T5] 写入
```

## 七、Schema 演进时的双库兼容做法

### 7.1 版本化演进

**1. 表命名版本化**

当前使用的表：
- `user_events_v2`（替代 v1）
- `computed_property_assignments_v2`（替代 v1）
- `processed_computed_properties_v2`（替代 v1）
- `computed_property_state_v3`（迭代到 v3）

```typescript
// config.ts:650-651
defaultUserEventsTableVersion:
  rawConfig.defaultUserEventsTableVersion ?? "",
```

**2. 增量创建而非修改**

```typescript
// userEvents/clickhouse.ts:532-539
await Promise.all(
  queries.map((query) =>
    clickhouseClient().exec({
      query,
      clickhouse_settings: { wait_end_of_query: 1 },
    }),
  ),
);
```

所有表创建都使用 `IF NOT EXISTS`：
- 幂等执行，重复创建不会报错
- 新表可以与旧表并存
- 通过代码逻辑控制使用哪张表

### 7.2 物化视图解耦

**1. 主表保持稳定，派生表按需新增**

```
user_events_v2（主表，稳定）
├── internal_events_mv → internal_events（内部事件派生）
├── group_user_assignments_mv → group_user_assignments（分组派生）
├── user_group_assignments_mv → user_group_assignments（用户分组派生）
├── user_property_idx_num_mv → user_property_idx_num（数值索引）
├── user_property_idx_str_mv → user_property_idx_str（字符串索引）
└── user_property_idx_date_mv → user_property_idx_date（日期索引）
```

优势：
- 主表 schema 变更影响范围小
- 新增物化视图不影响现有逻辑
- 可以逐步迁移查询到新表

**2. 新旧字段兼容**

```typescript
// userEvents/clickhouse.ts:306-356
CREATE TABLE IF NOT EXISTS user_events_v2 (
  event_type Enum(...) DEFAULT JSONExtract(...),
  event String DEFAULT JSONExtract(...),
  event_time DateTime64 DEFAULT assumeNotNull(parseDateTime64BestEffortOrNull(...)),
  message_id String,
  user_id String DEFAULT JSONExtract(...),
  anonymous_id String DEFAULT JSONExtract(...),
  user_or_anonymous_id String DEFAULT assumeNotNull(coalesce(...)),
  properties String DEFAULT assumeNotNull(coalesce(...)),
  hidden Boolean DEFAULT JSONExtractBool(...),
  processing_time DateTime64(3) DEFAULT now64(3),
  server_time DateTime64(3),
  message_raw String,
  workspace_id String,
  INDEX message_id_idx message_id TYPE bloom_filter(0.01) GRANULARITY 4
)
```

关键字段使用 `DEFAULT` 子句：
- 从 `message_raw` 动态提取
- 旧数据自动填充默认值
- 新增字段无需数据迁移

### 7.3 PostgreSQL 的 Schema 迁移

**1. 代码优先的迁移**

使用 Drizzle ORM 的迁移机制：
```typescript
// bootstrap.ts:38
import { drizzleMigrate } from "./migrate";
```

**2. 枚举值的扩展**

```typescript
// db/schema.ts:16-20
export const computedPropertyType = pgEnum("ComputedPropertyType", [
  "Segment",
  "UserProperty",
]);
```

PostgreSQL 枚举可以安全扩展：
- 新增值不会影响现有数据
- 只追加不修改

### 7.4 数据清理与 TTL

**1. 临时表的自动清理**

```typescript
// userEvents/clickhouse.ts:433-444
create table if not exists updated_computed_property_state(
  workspace_id LowCardinality(String),
  type Enum('user_property' = 1, 'segment' = 2),
  computed_property_id LowCardinality(String),
  state_id LowCardinality(String),
  user_id String,
  computed_at DateTime64(3)
) Engine=MergeTree
partition by toYYYYMMDD(computed_at)
order by computed_at
TTL toStartOfDay(computed_at) + interval 24 hour;  // 24 小时后自动删除
```

```typescript
// userEvents/clickhouse.ts:449-459
create table if not exists updated_property_assignments_v2(
  workspace_id LowCardinality(String),
  type Enum('user_property' = 1, 'segment' = 2),
  computed_property_id LowCardinality(String),
  user_id String,
  assigned_at DateTime64(3)
) Engine=MergeTree
partition by toYYYYMMDD(assigned_at)
order by assigned_at
TTL toStartOfDay(assigned_at) + interval 24 hour;
```

- TTL（Time To Live）自动清理过期数据
- 按日期分区便于批量删除
- 减少手动数据迁移需求

### 7.5 灰度发布策略

**1. 特性开关**

```typescript
// config.ts:180-181
computePropertiesSplit: Type.Optional(BoolStr),
enableColdStorage: Type.Optional(BoolStr),
```

通过配置控制新功能启用：
- `computePropertiesSplit`：是否启用拆分计算
- `enableColdStorage`：是否启用冷存储

**2. 渐进式迁移**

```typescript
// userProperties.ts:977-1010
export async function deleteUserProperty({
  workspaceId,
  id,
}: DeleteUserPropertyRequest): Promise<UserProperty | null> {
  const [deleted] = await db()
    .delete(dbUserProperty)
    .where(...)
    .returning();

  if (!deleted) {
    return null;
  }

  // 同时清理 ClickHouse 中的相关数据
  const queries = [
    `DELETE FROM computed_property_state_v3 WHERE ... settings mutations_sync = 0, lightweight_deletes_sync = 0;`,
    `DELETE FROM computed_property_assignments_v2 WHERE ...`,
    `DELETE FROM processed_computed_properties_v2 WHERE ...`,
    ...
  ];

  await Promise.all(
    queries.map((query) =>
      chCommand({ query, query_params: qb.getQueries() }),
    ),
  );

  return deleted;
}
```

删除操作的双库协调：
1. 先删除 Postgres 元数据（阻止新的计算）
2. 再清理 ClickHouse 中的派生数据
3. 使用 `mutations_sync = 0` 异步执行，不阻塞

## 八、代码位置索引

| 功能 | 文件路径 | 关键行 |
|-----|---------|-------|
| ClickHouse 连接 | `packages/backend-lib/src/clickhouse.ts` | 204-218 |
| 事件写入 | `packages/backend-lib/src/userEvents.ts` | 90-145 |
| 事件查询去重 | `packages/backend-lib/src/userEvents.ts` | 352-371 |
| 表结构定义 | `packages/backend-lib/src/userEvents/clickhouse.ts` | 293-657 |
| 用户属性定义（Postgres） | `packages/backend-lib/src/db/schema.ts` | 151-222 |
| Period 表结构 | `packages/backend-lib/src/db/schema.ts` | 200-220 |
| PostgreSQL 连接 | `packages/backend-lib/src/db.ts` | 106-123 |
| 配置与写入模式 | `packages/backend-lib/src/config.ts` | 543-545 |
| computeState | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 3006-3180 |
| computeAssignments | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 3255-3878 |
| processAssignments | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 4049-4292 |
| processRowsInner（执行顺序） | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 3602-3750 |
| buildProcessAssignmentsQuery | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 3786-3912 |
| getLowerBoundClause | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 494-498 |
| shouldReset 判断 | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 129-148 |
| Period 查询 | `packages/backend-lib/src/computedProperties/periods.ts` | 138-193 |
| Period 创建 | `packages/backend-lib/src/computedProperties/periods.ts` | 195-284 |
| 属性赋值查询 | `packages/backend-lib/src/userProperties.ts` | 399-450 |
