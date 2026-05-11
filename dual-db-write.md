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
│                        阶段一：事件写入 ClickHouse                        │
│                                                                         │
│  目标表：user_events_v2（MergeTree）                                     │
│  写入模式：ch-sync / ch-async / kafka                                   │
│  物化视图：自动派生到 internal_events、group_user_assignments 等表       │
│                                                                         │
│  幂等保证：                                                              │
│    ┌─ 应用层：messageId 唯一标识                                         │
│    ├─ 查询层：argMax + GROUP BY 去重                                    │
│    └─ 引擎层：MergeTree 稀疏索引，同 ORDER BY 键数据有序                  │
│                                                                         │
│  失败恢复：                                                              │
│    ┌─ Kafka 模式：消息队列回溯                                           │
│    ├─ 同步/异步模式：依赖调用方重试                                      │
│    └─ messageId 保证重复提交无害                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ 定时调度（每 2 分钟）
┌─────────────────────────────────────────────────────────────────────────┐
│                        阶段二：计算中间状态                              │
│                                                                         │
│  目标表：computed_property_state_v3（AggregatingMergeTree）             │
│  处理：computeState()                                                   │
│                                                                         │
│  幂等保证：                                                              │
│    ┌─ State 引擎：argMaxState、uniqState 幂等聚合                       │
│    ├─ Period 表：记录已处理的时间范围                                   │
│    └─ lowerBoundClause：只处理 period.maxTo 之后的事件                  │
│                                                                         │
│  失败恢复：                                                              │
│    ┌─ Postgres 的 ComputedPropertyPeriod 表持久化进度                   │
│    ├─ 下次调度从 periodBound（maxTo）继续                                │
│    └─ 只增量处理新事件，不重复计算                                       │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        阶段三：计算属性赋值                              │
│                                                                         │
│  目标表：computed_property_assignments_v2（ReplacingMergeTree）         │
│  处理：computeAssignments()                                             │
│                                                                         │
│  幂等保证：                                                              │
│    ┌─ 引擎层：ReplacingMergeTree 自动去重同 ORDER BY 键记录              │
│    ├─ Period 表：按时间范围增量                                          │
│    └─ shouldReset：定义更新时重置重新计算                                │
│                                                                         │
│  失败恢复：                                                              │
│    ┌─ Period 表的 maxTo 记录已处理截止时间                              │
│    ├─ lowerBoundClause 避免重复处理                                     │
│    └─ ReplacingMergeTree 重复写入取最新版本                             │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        阶段四：处理赋值（触发下游）                       │
│                                                                         │
│  目标表：processed_computed_properties_v2（ReplacingMergeTree）         │
│  处理：processAssignments()                                             │
│                                                                         │
│  下游触发：                                                              │
│    ┌─ Journey（旅程）：进入/退出节点触发                                 │
│    └─ Integration（集成）：HubSpot 等第三方同步                          │
│                                                                         │
│  幂等保证：                                                              │
│    ┌─ LEFT ANY JOIN pcp 表过滤已处理的赋值                              │
│    ├─ processed_for + processed_for_type 区分不同处理目标               │
│    └─ 按 user_id 游标分页，cursor = lastUserId                          │
│                                                                         │
│  失败恢复：                                                              │
│    ┌─ pcp 表记录已处理的 user_id + 值                                   │
│    ├─ 下次处理时 JOIN 过滤，只处理值变化的 user                         │
│    └─ Period 表记录已处理的 assigned_at 范围                             │
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

### 5.1 阶段一：事件写入 ClickHouse

#### 5.1.1 写入流程

```
调用方（API / SDK）
    │
    ▼
生成 messageId（UUID）
    │
    ▼
writeMode 判断
    │
    ├─ kafka → 写入 Kafka 主题（由消费者异步消费到 CH）
    ├─ ch-async → ClickHouse async_insert（服务端异步缓冲）
    └─ ch-sync → ClickHouse 同步写入（wait_end_of_query=1）
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

> **关键点**：阶段一没有 Period 表，依赖 messageId 保证重复提交无害。事件一旦写入成功，后续阶段从 ClickHouse 读取。

---

### 5.2 阶段二：计算中间状态（computeState）

#### 5.2.1 处理流程

```
定时调度器触发（每 2 分钟）
    │
    ▼
从 Postgres 读取 Segments/UserProperties 定义
    │
    ▼
从 Period 表读取上一轮进度
    period = periodByComputedPropertyId.get()
    periodBound = period?.maxTo.getTime()
    │
    ▼
构建 lowerBoundClause：
    period > 0 ? `processing_time >= ${periodBound}` : ""
    │
    ▼
INSERT INTO computed_property_state_v3
SELECT ... FROM user_events_v2
WHERE processing_time <= now
  AND processing_time >= periodBound（如果有）
  AND (segment/property 条件)
    │
    ▼
写入 Postgres Period 表
    from = 上一轮的 to
    to = now
    step = "ComputeState"
    onConflictDoNothing()
```

#### 5.2.2 幂等机制

| 层级 | 机制 | 实现位置 |
|-----|------|---------|
| **表引擎** | AggregatingMergeTree | `userEvents/clickhouse.ts:572` |
| **聚合函数** | argMaxState、uniqState | `computePropertiesIncremental.ts:3129-3132` |
| **进度跟踪** | Period 表 + lowerBoundClause | `computePropertiesIncremental.ts:3094-3097` |

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
    from: timestamp({ withTimezone: true, mode: "string" }),
    to: timestamp({ withTimezone: true, mode: "string" }).notNull(),
    step: computedPropertyStep().notNull(),  // "ComputeState" | "ComputeAssignments" | "ProcessAssignments"
    version: text().notNull(),  // definitionUpdatedAt.toString()
    createdAt: timestamp(...).defaultNow().notNull(),
  },
  ...
);
```

**恢复流程**：

```
本次调度失败（computeState 抛出异常）
    │
    │ 此时 Postgres Period 表还未更新
    │ createPeriods 在 computeState 最后执行
    │
    ▼
下次调度
    │
    ├─ getPeriodsByComputedPropertyId() → 返回上一轮成功的 Period
    ├─ periodBound = 上一轮成功的 maxTo
    ├─ lowerBoundClause 过滤已处理的事件
    └─ 从断点继续增量计算
```

> **关键点**：Period 写入在阶段最后。如果中间失败，下次从上次成功的 periodBound 继续。

---

### 5.3 阶段三：计算属性赋值（computeAssignments）

#### 5.3.1 处理流程

```
从 Period 表读取 ComputeAssignments 进度
    periodBound = period?.maxTo.getTime()
    │
    ├─ 是否需要重置？
    │   shouldReset = definitionUpdatedAt > periodBound（定义更新了）
    │   └─ 是 → 全量重新计算
    │
    ▼
构建 lowerBoundClause：
    getLowerBoundClause(periodBound)
    │
    ▼
INSERT INTO computed_property_assignments_v2
SELECT ... FROM resolved_segment_state / resolved_user_property_state
WHERE computed_at <= now
  AND state_id IN ...
  ${lowerBoundClause}
    │
    ▼
写入 Postgres Period 表
    step = "ComputeAssignments"
```

#### 5.3.2 幂等机制

| 层级 | 机制 | 实现位置 |
|-----|------|---------|
| **表引擎** | ReplacingMergeTree | `userEvents/clickhouse.ts:396` |
| **进度跟踪** | Period 表 + lowerBoundClause | `computePropertiesIncremental.ts:3295` |
| **版本控制** | definitionUpdatedAt 触发重置 | `computePropertiesIncremental.ts:3326-3331` |

**ReplacingMergeTree 去重**：

```sql
ENGINE = ReplacingMergeTree()
ORDER BY (workspace_id, type, computed_property_id, user_id)
```

- 相同 `(workspace_id, type, computed_property_id, user_id)` 的记录
- 后台合并时只保留最新版本
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
本次调度失败
    │
    │ computeAssignments 抛出异常
    │ Period 表未更新
    │
    ▼
下次调度
    │
    ├─ periodBound = 上一轮成功的 maxTo
    ├─ lowerBoundClause = `assigned_at >= ${periodBound}`
    ├─ 只增量处理新赋值
    └─ 重复赋值由 ReplacingMergeTree 自动去重
```

---

### 5.4 阶段四：处理赋值（processAssignments）

#### 5.4.1 处理流程

```
从 Period 表读取 ProcessAssignments 进度
    periodBound = period?.maxTo.getTime()
    │
    ▼
构建 AssignmentProcessor 列表（每个 journey/integration 一个）
    processed_for = journeyId / integrationName
    processed_for_type = "journey" / "integration"
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
                cpa.value != pcp.value  -- 值变化才处理
                AND assigned_at >= periodBound
        │
        ▼
        processRows(rows)：
            ├─ Journey：startSegmentEntryWorkflow()
            └─ Integration：startHubspotUserIntegrationWorkflow()
        │
        ▼
        insertProcessedComputedProperties(assignments)
            └─ 写入 processed_computed_properties_v2
        │
        ▼
        cursor = lastUserId
    │
    ▼
写入 Postgres Period 表
    step = "ProcessAssignments"
```

#### 5.4.2 幂等机制

| 层级 | 机制 | 实现位置 |
|-----|------|---------|
| **去重表** | processed_computed_properties_v2 | `computePropertiesIncremental.ts:3879-3901` |
| **JOIN 过滤** | LEFT ANY JOIN pcp 过滤已处理 | `computePropertiesIncremental.ts:3851-3910` |
| **值变化检测** | `cpa.value != pcp.value` | `computePropertiesIncremental.ts:3899-3900` |
| **分页游标** | cursor = lastUserId | `computePropertiesIncremental.ts:3951` |
| **表引擎** | ReplacingMergeTree | `userEvents/clickhouse.ts:407-427` |

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
    WHERE assigned_at >= ${periodBound}  -- 只处理新赋值
    GROUP BY user_id
  ) cpa
  LEFT ANY JOIN (
    SELECT user_id,
           argMax(segment_value, processed_at) segment_value,
           argMax(user_property_value, processed_at) user_property_value
    FROM processed_computed_properties_v2
    WHERE processed_for = ${journeyId}  -- 区分不同处理目标
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

#### 5.4.3 失败恢复

**场景 1：processRows 之前失败**

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

**场景 2：processRows 部分成功后失败**

```
第 1 页成功：
    ├─ 触发 journey 工作流
    └─ 写入 processed_computed_properties_v2（user_id_1 ~ user_id_100）

第 2 页失败
    │
    │ Period 表还没更新
    │
    ▼
下次调度
    │
    ├─ 全量查询（periodBound = 上一轮成功）
    ├─ JOIN pcp 时，user_id_1 ~ 100 已匹配，被过滤
    ├─ 只处理 user_id_101 之后
    └─ cursor 重新从 null 开始，但实际只处理剩余数据
```

**场景 3：下游触发失败但 pcp 已写入**

```typescript
// computePropertiesIncremental.ts:3746-3748
await insertProcessedComputedProperties({
  assignments: processedAssignments,
});
```

如果 `startSegmentEntryWorkflow()` 失败但 `insertProcessedComputedProperties()` 已执行：
- pcp 表记录了"已处理"
- 但实际下游没触发

**补偿机制**：值变化检测确保最终一致性。

如果赋值值没有变化，下次调度不会再处理。但如果值变化了，会重新触发。

> **关键点**：processAssignments 是"至少一次"语义。下游系统需要自己处理幂等（如 journey 的消息去重）。

---

### 5.5 幂等与恢复汇总表

| 阶段 | 步骤 | 幂等层 | 恢复点 | 实现 |
|-----|------|--------|--------|------|
| **一** | 事件写入 | messageId + MergeTree | 无，依赖调用方重试 | `userEvents.ts` |
| **二** | computeState | AggregatingMergeTree + Period | `period.maxTo`（processing_time） | `computePropertiesIncremental.ts:3006` |
| **三** | computeAssignments | ReplacingMergeTree + Period | `period.maxTo`（assigned_at） | `computePropertiesIncremental.ts:3255` |
| **四** | processAssignments | pcp 表 + ReplacingMergeTree + Period | `period.maxTo` + pcp 记录 | `computePropertiesIncremental.ts:4049` |

## 六、最终一致性的收敛机制

### 6.1 收敛条件

双库最终一致性通过以下机制收敛：

**1. Period 表的 monotonic 前进**

```typescript
// periods.ts:219-229
newPeriods.push({
  from: previousPeriod ? previousPeriod.maxTo : null,  // 上一轮的 to
  to: nowD,                                              // 本轮截止
  version: segment.definitionUpdatedAt.toString(),       // 定义版本
  ...
});
```

- `from` 永远是上一轮的 `to`
- `to` 单调递增
- 不会回退处理已覆盖的时间范围

**2. 定义更新的版本控制**

```
Postgres 定义：
    UserProperty {
      id,
      definition,
      definitionUpdatedAt: 1700000000000,  ← 版本号
      status
    }

ClickHouse 赋值：
    computed_property_assignments_v2 {
      computed_property_id = id,
      assigned_at
    }

Period 表：
    ComputedPropertyPeriod {
      computedPropertyId = id,
      version = "1700000000000",  ← 版本号匹配
      maxTo
    }
```

- `version = definitionUpdatedAt.toString()` 关联定义版本
- 定义更新时，`shouldReset` 触发全量重算
- 旧版本 Period 被忽略，只处理当前版本

**3. processed_computed_properties_v2 的值快照**

```
用户 A 在 Segment S 中的状态变化：

时间点 | assigned_at | segment_value | processed (journey J)
-------|-------------|---------------|---------------------
  t1   |   t1        |    true       | 已触发进入
  t2   |   t2        |    false      | 已触发退出
  t3   |   t3        |    true       | 已触发重新进入
```

pcp 表记录每个 `(user_id, processed_for)` 的最后值：
- `argMax(segment_value, processed_at)` 取最新值
- 下次比较时，如果值没变化则跳过
- 保证每个值变化只触发一次下游（或多次但值一致）

### 6.2 收敛流程示例

**场景：定义更新 → 全量重算 → 下游触发**

```
┌─ Postgres ──────────────────────────────────────────────────┐
│                                                             │
│  时刻 0: 定义创建                                            │
│    definitionUpdatedAt = 0                                  │
│    status = Running                                         │
│                                                             │
│  时刻 3: 定义更新                                           │
│    definitionUpdatedAt = 3 ← 版本更新                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ 读取定义
┌─ Period 表（Postgres） ─────────────────────────────────────┐
│                                                             │
│  ComputeAssignments Periods:                                │
│    [version="0", maxTo=1]   ← 旧版本，处理到 t1             │
│    [version="0", maxTo=2]   ← 旧版本，处理到 t2             │
│                                                             │
│  时刻 3 调度：                                               │
│    currentVersion = "3"                                     │
│    period = getPeriod(version="3") → null                  │
│    periodBound = 0                                          │
│    shouldReset = (3 > 0) = true                             │
│                                                             │
│  全量计算后：                                                │
│    [version="3", maxTo=4] ← 新版本，从 0 全量重算到 t4     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ 全量重算
┌─ computed_property_assignments_v2（ClickHouse） ────────────┐
│                                                             │
│  t1 之前的赋值：                                              │
│    (user_A, segment_value=true, assigned_at=t1)             │
│    (user_B, segment_value=false, assigned_at=t1)            │
│                                                             │
│  t3 全量重算后：                                              │
│    (user_A, segment_value=true, assigned_at=t4)  ← 替换旧值  │
│    (user_B, segment_value=true,  assigned_at=t4)  ← 替换旧值  │
│    (user_C, segment_value=false, assigned_at=t4) ← 新增     │
│                                                             │
│  ReplacingMergeTree 合并后：                                  │
│    每个 user 只保留 assigned_at 最大的记录                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ 检测值变化
┌─ processed_computed_properties_v2（ClickHouse） ────────────┐
│                                                             │
│  processed_for = journey_J                                   │
│                                                             │
│  t3 之前的记录：                                             │
│    (user_A, segment_value=true, processed_at=t1)            │
│    (user_B, segment_value=false, processed_at=t1)           │
│                                                             │
│  t4 时的 JOIN 比较：                                         │
│    user_A: true == true → 跳过                              │
│    user_B: true != false → 触发（状态变化）                  │
│    user_C: 无记录 → 触发（新增）                             │
│                                                             │
│  写入新记录：                                                │
│    (user_B, segment_value=true, processed_at=t4)            │
│    (user_C, segment_value=false, processed_at=t4)           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    最终状态收敛一致
```

### 6.3 收敛时间线

```
时刻 0: 定义创建
          ↓
时刻 1: 第一轮调度（ComputeState → ComputeAssignments → ProcessAssignments）
          ↓ Period(0, 1)
时刻 2: 第二轮调度
          ↓ Period(1, 2)
时刻 3: 定义更新（definitionUpdatedAt = 3）
          ↓
时刻 4: 第三轮调度
          ├─ shouldReset = true
          ├─ 全量重算（不使用 lowerBound）
          └─ Period(0, 4)  ← 覆盖整个时间线
          ↓
时刻 5: 第四轮调度
          ├─ currentVersion = "3"
          ├─ periodBound = 4
          ├─ 增量计算（4 → 5）
          └─ Period(4, 5)  ← 恢复单调递增
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
| AssignmentProcessor | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 3928-4047 |
| buildProcessAssignmentsQuery | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 3786-3912 |
| Period 管理 | `packages/backend-lib/src/computedProperties/periods.ts` | 138-284 |
| shouldReset 判断 | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 129-148 |
| 属性赋值查询 | `packages/backend-lib/src/userProperties.ts` | 399-450 |
