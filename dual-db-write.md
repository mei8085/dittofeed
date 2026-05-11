# 双数据库协同写入机制分析

## 一、架构概述

DittoFeed 采用 ClickHouse + PostgreSQL 双数据库架构，实现不同数据类型的优化存储：

- **ClickHouse**：事件流存储（user_events_v2 等）
- **PostgreSQL**：元数据管理（UserProperty、Segment、Workspace 等）

## 二、事件流为什么落到 ClickHouse

### 2.1 数据特征分析

事件流具有以下特征：
- **高吞吐量**：用户行为事件（track/identify/page/screen/group/alias）每秒可能产生大量数据
- **时间序列**：事件带有时间戳，查询多按时间范围过滤
- **写多读少**：写入频率远高于读取频率
- **聚合分析**：主要用于 OLAP 场景（用户分群、漏斗分析、趋势统计）

### 2.2 ClickHouse 的技术优势

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
  JSONExtractString(properties, 'broadcastId') as broadcast_id,
  JSONExtractString(properties, 'journeyId') as journey_id,
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

### 2.3 为什么不是 Postgres？

| 对比维度 | PostgreSQL | ClickHouse |
|---------|-----------|------------|
| 单条插入性能 | 中等（需 WAL、索引维护） | 优秀（批量写入、后台合并） |
| 高并发写入 | 有限（锁竞争、连接池） | 优秀（面向批量的设计） |
| 聚合查询性能 | 中等（行存储） | 优秀（列存储 + 向量化执行） |
| 时间序列优化 | 需手动分区索引 | 原生支持 |
| 存储成本 | 较高（行存储 + 多索引） | 较低（列存储 + 高效压缩） |

## 三、元数据为什么放在 PostgreSQL

### 3.1 元数据特征分析

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

### 3.2 PostgreSQL 的技术优势

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
    uniqueIndex("UserProperty_workspaceId_name_key").using(
      "btree",
      table.workspaceId.asc().nullsLast().op("uuid_ops"),
      table.name.asc().nullsLast().op("text_ops"),
    ),
    foreignKey({
      columns: [table.workspaceId],
      foreignColumns: [workspace.id],
      name: "UserProperty_workspaceId_fkey",
    })
      .onUpdate("cascade")
      .onDelete("cascade"),
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
export const segmentStatus = pgEnum("SegmentStatus", [
  "NotStarted",
  "Running",
  "Paused",
]);
export const userPropertyStatus = pgEnum("UserPropertyStatus", [
  "NotStarted",
  "Running",
  "Paused",
]);
```

- 枚举类型（pgEnum）保证字段值合法性
- JSONB 类型存储灵活的定义数据
- UUID 主键保证分布式唯一性

### 3.3 为什么不是 ClickHouse？

| 对比维度 | ClickHouse | PostgreSQL |
|---------|-----------|------------|
| 事务支持 | 有限（无完整 ACID） | 完整支持 |
| 外键约束 | 不支持 | 原生支持 |
| 行级更新 | 弱（Mutation 代价高） | 优秀 |
| 小表随机读 | 弱（面向批量设计） | 优秀 |
| 复杂查询优化 | OLAP 优化器 | OLTP 优化器 |

## 四、写入幂等与失败重试机制

### 4.1 幂等性设计

**1. 事件写入的幂等键**

```typescript
// userEvents.ts:34-39
export interface InsertUserEvent {
  messageRaw: string | Record<string, unknown>;
  processingTime?: string;
  serverTime?: string;
  messageId: string;  // 幂等键
}
```

每个事件都有唯一的 `messageId`，用于：
- 避免重复提交
- 支持查询时去重

**2. 查询时的去重逻辑**

```typescript
// userEvents.ts:352-371
let messageIdClause = "";
let orderByClause = "";
if (messageId) {
  // 使用子查询 + argMax 实现去重
  messageIdClause = `
    AND (workspace_id, processing_time, user_or_anonymous_id, event_time, message_id) IN (
      SELECT
        workspace_id,
        max(processing_time),
        user_or_anonymous_id,
        argMax(event_time, processing_time),
        message_id
      FROM user_events_v2
      WHERE
        ${workspaceIdClause}
        ${messageIdWhereClause}
      GROUP BY
        workspace_id,
        user_or_anonymous_id,
        message_id
    )
  `;
}
```

- `max(processing_time)` 取最新处理时间
- `argMax(event_time, processing_time)` 取对应事件时间
- 确保同一 message_id 只返回一条记录

**3. ReplacingMergeTree 的去重能力**

对于 `computed_property_assignments_v2` 等表：
- `ORDER BY (workspace_id, type, computed_property_id, user_id)` 作为去重键
- 相同键的多次写入会在合并时保留最新版本
- 后台自动合并，无需应用层干预

### 4.2 失败重试机制

**1. 计算属性的处理进度跟踪**

```typescript
// userEvents/clickhouse.ts:408-428
CREATE TABLE IF NOT EXISTS processed_computed_properties_v2 (
  workspace_id LowCardinality(String),
  user_id String,
  type Enum('user_property' = 1, 'segment' = 2),
  computed_property_id LowCardinality(String),
  processed_for LowCardinality(String),
  processed_for_type LowCardinality(String),
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

核心设计思想：
- 记录哪些赋值已经被"处理"过
- `processed_for` 和 `processed_for_type` 区分不同的处理目标
- 失败时可以从 `max_event_time` 断点续传

**2. 配置化的重试参数**

```typescript
// config.ts:781-785
waitForComputePropertiesMaxAttempts: parseMaxAttempts(
  rawConfig.waitForComputePropertiesMaxAttempts,
  nodeEnv === NodeEnvEnum.Test ? 1 : 5,
),
```

```typescript
// config.ts:698-700
computePropertiesAttempts: rawConfig.computePropertiesAttempts
  ? parseInt(rawConfig.computePropertiesAttempts)
  : 150,
```

关键重试参数：
- `waitForComputePropertiesMaxAttempts`：等待计算属性的最大尝试次数（默认 5 次）
- `computePropertiesAttempts`：计算属性流程的最大尝试次数（默认 150 次）
- `computePropertiesJitterMs`：重试抖动，避免惊群效应
- `computePropertiesSchedulerInterval`：调度器轮询间隔

**3. 增量计算的时间窗口**

```typescript
// computedProperties/periods.ts（概念）
type Period = {
  from: Date;  // 起始时间
  to: Date;    // 结束时间
  step: string; // 处理步骤
};
```

计算属性按时间窗口增量处理：
- 每个 Period 记录处理的时间范围
- 失败时只重跑未完成的 Period
- `ComputedPropertyPeriod` 表持久化进度

### 4.3 一致性保证

**1. 顺序一致性配置**

```typescript
// config.ts:831-833
export function assignmentSequentialConsistency(): "1" | "0" {
  return config().assignmentSequentialConsistency ? "1" : "0";
}
```

```typescript
// userProperties.ts:434-440
const result = await chQuery({
  query,
  query_params: qb.getQueries(),
  clickhouse_settings: {
    select_sequential_consistency: assignmentSequentialConsistency(),
  },
});
```

- `select_sequential_consistency=1` 确保读取已提交的数据
- 在需要强一致读取的场景启用

**2. 双库协调的最终一致性**

用户属性定义（Postgres）→ 属性赋值（ClickHouse）的协调：
1. Postgres 存储定义（id、name、definition、status）
2. ClickHouse 存储赋值（computed_property_id 关联 Postgres 的 id）
3. 定期计算流程从 Postgres 读取定义，在 ClickHouse 计算赋值
4. 通过 `definitionUpdatedAt` 判断是否需要重新计算

```typescript
// computedProperties/computePropertiesIncremental.ts:129-148
function shouldResetComputedProperty({
  definitionUpdatedAt,
  createdAt,
  now,
  periodBound,
}: {
  definitionUpdatedAt: number;
  createdAt: number;
  now: number;
  periodBound?: number;
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

定义更新后，相关计算会被重置重新执行。

## 五、Schema 演进时的双库兼容做法

### 5.1 版本化演进

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

### 5.2 物化视图解耦

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

### 5.3 PostgreSQL 的 Schema 迁移

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

### 5.4 数据清理与 TTL

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

### 5.5 灰度发布策略

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
  const qb = new ClickHouseQueryBuilder();
  const workspaceIdParam = qb.addQueryValue(workspaceId, "String");
  const computedPropertyIdParam = qb.addQueryValue(id, "String");

  const queries = [
    `DELETE FROM computed_property_state_v3 WHERE ... settings mutations_sync = 0, lightweight_deletes_sync = 0;`,
    `DELETE FROM computed_property_assignments_v2 WHERE ...`,
    `DELETE FROM processed_computed_properties_v2 WHERE ...`,
    `DELETE FROM computed_property_state_index WHERE ...`,
    `DELETE FROM updated_computed_property_state WHERE ...`,
    `DELETE FROM updated_property_assignments_v2 WHERE ...`,
  ];

  await Promise.all(
    queries.map((query) =>
      chCommand({
        query,
        query_params: qb.getQueries(),
      }),
    ),
  );

  return deleted;
}
```

删除操作的双库协调：
1. 先删除 Postgres 元数据（阻止新的计算）
2. 再清理 ClickHouse 中的派生数据
3. 使用 `mutations_sync = 0` 异步执行，不阻塞

## 六、数据流总览

```
┌─────────────────────────────────────────────────────────────────┐
│                      事件输入层                                   │
├─────────────────────────────────────────────────────────────────┤
│  API / SDK / Segment.io Webhook                                 │
│         │                                                       │
│         ▼                                                       │
│  messageId 生成（UUID）                                          │
│         │                                                       │
│         ▼                                                       │
├─────────────────────────────────────────────────────────────────┤
│                      写入路由层                                   │
├─────────────────────────────────────────────────────────────────┤
│  writeMode 配置：                                                │
│  ├── kafka: 写入 Kafka 主题                                      │
│  ├── ch-async: ClickHouse 异步插入                               │
│  └── ch-sync: ClickHouse 同步插入                                │
│         │                                                       │
│         ▼                                                       │
├─────────────────────────────────────────────────────────────────┤
│                      存储层                                      │
├──────────────────────────────┬──────────────────────────────────┤
│        ClickHouse            │           PostgreSQL             │
├──────────────────────────────┼──────────────────────────────────┤
│  user_events_v2 (主表)        │  Workspace (工作空间)            │
│  internal_events (物化视图)   │  UserProperty (属性定义)         │
│  computed_property_* (属性值) │  Segment (人群定义)              │
│  *_assignments (用户赋值)     │  MessageTemplate (模板)          │
│  *_state (计算状态)           │  Integration (集成)              │
│  processed_* (处理进度)       │  Secret (密钥)                   │
│  updated_* (变更追踪, TTL)    │  WriteKey (写入密钥)             │
└──────────────────────────────┴──────────────────────────────────┘
         │                                   │
         │ 定期计算（每 2 分钟）               │ 定义变更（实时）
         ▼                                   ▼
├─────────────────────────────────────────────────────────────────┤
│                      计算层                                      │
├─────────────────────────────────────────────────────────────────┤
│  1. 从 Postgres 读取属性/人群定义                                 │
│  2. 检查 definitionUpdatedAt 是否更新                            │
│  3. 按时间窗口增量计算赋值                                        │
│  4. 写入 computed_property_assignments_v2                       │
│  5. 记录 Period 进度                                             │
│  6. 更新 processed_computed_properties_v2                       │
│  7. 触发后续处理（旅程、同步到第三方）                              │
└─────────────────────────────────────────────────────────────────┘
```

## 七、关键设计决策总结

| 决策点 | 方案 | 理由 |
|-------|------|------|
| 事件存储 | ClickHouse + MergeTree | 高吞吐、时间序列优化、列存储聚合 |
| 元数据存储 | PostgreSQL | 事务、关系、外键、行级更新 |
| 幂等键 | messageId + ReplacingMergeTree | 业务标识 + 引擎级去重 |
| 失败重试 | Period + processed_* 表 | 断点续传、进度持久化 |
| Schema 演进 | 版本化表名 + IF NOT EXISTS + 物化视图 | 零停机、渐进迁移 |
| 双库协调 | 最终一致性 + definitionUpdatedAt | 定义与赋值解耦 |
| 数据清理 | TTL + 分区 | 自动过期、批量删除高效 |

## 八、代码位置索引

| 功能 | 文件路径 | 关键行 |
|-----|---------|-------|
| ClickHouse 连接 | `packages/backend-lib/src/clickhouse.ts` | 204-218 |
| 事件写入 | `packages/backend-lib/src/userEvents.ts` | 90-145 |
| 表结构定义 | `packages/backend-lib/src/userEvents/clickhouse.ts` | 293-657 |
| 用户属性定义 | `packages/backend-lib/src/db/schema.ts` | 151-222 |
| PostgreSQL 连接 | `packages/backend-lib/src/db.ts` | 106-123 |
| 配置与写入模式 | `packages/backend-lib/src/config.ts` | 543-545 |
| 计算属性增量 | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 1-200+ |
| 属性赋值查询 | `packages/backend-lib/src/userProperties.ts` | 399-450 |
