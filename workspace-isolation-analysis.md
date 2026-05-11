# DittoFeed 多工作空间 ClickHouse 数据隔离机制分析

## 概述

DittoFeed 通过多层机制实现多工作空间（Workspace）的数据隔离，核心策略是：
- **数据层隔离**：所有数据记录都携带 `workspace_id` 字段
- **认证层隔离**：通过 WriteKey 和 JWT 双重认证确保 workspace_id 的合法性
- **查询层隔离**：所有查询都强制携带 workspace_id 过滤条件
- **关系层隔离**：PostgreSQL 外键约束和 ClickHouse 分区/排序键设计

---

## 一、工作空间模型

### 1.1 工作空间类型

在 `packages/backend-lib/src/db/schema.ts:91-95` 定义了三种工作空间类型：

```typescript
export const workspaceType = pgEnum("WorkspaceType", [
  "Root",     // 根工作空间
  "Child",    // 子工作空间
  "Parent",   // 父工作空间（用于分组）
]);
```

工作空间表结构（schema.ts:97-122）：
- `id`: UUID 主键
- `parentWorkspaceId`: 可选，用于建立父子关系
- `status`: Active / Tombstoned / Paused
- `type`: Root / Child / Parent
- `externalId`: 可选，用于外部系统集成

### 1.2 父子工作空间关系

父工作空间可以包含多个子工作空间。查询时，父工作空间的请求会自动包含所有子工作空间的数据。

关键实现：`packages/backend-lib/src/userEvents.ts:468-482`

```typescript
async function buildWorkspaceIdClause(
  workspaceId: string,
  qb: ClickHouseQueryBuilder,
): Promise<string> {
  const childWorkspaceIds = (
    await db()
      .select({ id: dbWorkspace.id })
      .from(dbWorkspace)
      .where(eq(dbWorkspace.parentWorkspaceId, workspaceId))
  ).map((o) => o.id);

  return childWorkspaceIds.length
    ? `workspace_id IN ${qb.addQueryValue(childWorkspaceIds, "Array(String)")}`
    : `workspace_id = ${qb.addQueryValue(workspaceId, "String")}`;
}
```

---

## 二、数据写入阶段的 workspace_id 传递

### 2.1 外部事件写入流程（公共 API）

#### 路径总览
```
HTTP Request (Authorization: Basic <writeKey>)
    ↓
publicAppsController (validateWriteKey)
    ↓
submitIdentify / submitTrackWithTriggers / submitPage 等
    ↓
insertUserEvents
    ↓
insertUserEventsDirect (ClickHouse 直写) 或 Kafka 异步写入
    ↓
user_events_v2 表
```

#### 详细步骤

**Step 1: WriteKey 认证获取 workspace_id**

位置：`packages/api/src/controllers/publicAppsController.ts:51-64`

```typescript
async (request, reply) => {
  const workspaceIdFromWriteKey = await validateWriteKey({
    writeKey: request.headers.authorization,
  });

  if (workspaceIdFromWriteKey.isErr()) {
    return reply.status(401).send({
      message: workspaceIdFromWriteKey.error,
    });
  }

  await submitIdentify({
    workspaceId: workspaceIdFromWriteKey.value,
    data: request.body,
  });
  return reply.status(204).send();
}
```

**Step 2: validateWriteKey 实现**

位置：`packages/backend-lib/src/auth.ts:75-116`

```typescript
export async function validateWriteKey({
  writeKey,
}: {
  writeKey: string;
}): Promise<Result<string, ValidateWriteKeyError>> {
  // 解析 Basic Auth header: base64(secretKeyId:secretKeyValue)
  const encodedWriteKey = writeKey.split(" ")[1];
  const decodedWriteKey = Buffer.from(encodedWriteKey, "base64").toString("utf-8");
  const [secretKeyId, secretKeyValue] = decodedWriteKey.split(":");

  // 从 PostgreSQL 查询 secret 并关联 workspace
  const writeKeySecret = await db().query.secret.findFirst({
    where: eq(dbSecret.id, secretKeyId),
    with: {
      workspace: true,
    },
  });

  // 验证工作空间状态
  if (!canWorkspaceReceiveEvents({ workspace: writeKeySecret.workspace })) {
    return err("WorkspaceIneligible");
  }

  return writeKeySecret.value === secretKeyValue
    ? ok(writeKeySecret.workspaceId)
    : err("InvalidWriteKey");
}
```

**Step 3: 工作空间状态验证**

位置：`packages/backend-lib/src/auth.ts:39-62`

```typescript
export function canWorkspaceReceiveEvents({
  workspace,
}: {
  workspace: Workspace;
}): boolean {
  return (
    workspace.status === WorkspaceStatusDbEnum.Active &&
    workspace.type !== "Parent"  // Parent 类型工作空间不能接收事件
  );
}
```

**Step 4: 事件封装与写入**

位置：`packages/backend-lib/src/apps.ts:22-46`

```typescript
export async function submitIdentify({
  workspaceId,
  data,
}: {
  workspaceId: string;
  data: IdentifyData;
}) {
  const userEvent: InsertUserEvent = {
    messageRaw: JSON.stringify({
      type: "identify",
      traits: data.traits ?? {},
      timestamp: data.timestamp ?? new Date().toISOString(),
      ...R.omit(data, ["timestamp", "traits"]),
    }),
    messageId: data.messageId,
  };
  await insertUserEvents({
    workspaceId,
    userEvents: [userEvent],
  });
}
```

**Step 5: ClickHouse 写入**

位置：`packages/backend-lib/src/userEvents.ts:51-88`

```typescript
async function insertUserEventsDirect({
  workspaceId,
  userEvents,
  asyncInsert,
}: ...) {
  const values = userEvents.map((e) => {
    const value = {
      workspace_id: workspaceId,  // 关键：写入时携带 workspace_id
      message_raw: e.messageRaw,
      processing_time: e.processingTime ?? null,
      message_id: e.messageId,
      server_time: e.serverTime ?? null,
    };
    return value;
  });

  await clickhouseClient().insert({
    table: `user_events_v2 (message_raw, processing_time, workspace_id, message_id, server_time)`,
    values,
    format: "JSONEachRow",
  });
}
```

### 2.2 Kafka 异步写入模式

当配置为 Kafka 写入模式时（`config().writeMode === "kafka"`）：

位置：`packages/backend-lib/src/userEvents.ts:90-145`

```typescript
export async function insertUserEvents(
  { workspaceId, userEvents, events }: InsertUserEventsParams,
  options?: { writeModeOverride?: WriteMode },
): Promise<void> {
  switch (effectiveWriteMode) {
    case "kafka": {
      await (
        await kafkaProducer()
      ).send({
        topic: userEventsTopicName,
        messages: userEventsWithDefault.map(
          ({ messageRaw, messageId, processingTime, serverTime }) => ({
            key: messageId,
            value: JSON.stringify({
              processing_time: processingTime,
              workspace_id: workspaceId,  // Kafka 消息也携带 workspace_id
              message_id: messageId,
              server_time: serverTime,
              message_raw: messageRaw,
            }),
          }),
        ),
      });
      break;
    }
    // ... ch-async 和 ch-sync 模式
  }
}
```

Kafka 到 ClickHouse 的桥接通过 ClickHouse 的 Kafka Engine 表和 Materialized View 实现：

位置：`packages/backend-lib/src/userEvents/clickhouse.ts:602-657`

```sql
CREATE TABLE IF NOT EXISTS user_events_queue_v2
(message_raw String, workspace_id String, message_id String)
ENGINE = Kafka(...)

CREATE MATERIALIZED VIEW IF NOT EXISTS user_events_mv_v2
TO user_events_v2 AS
SELECT *
FROM user_events_queue_v2;
```

### 2.3 内部事件写入

位置：`packages/backend-lib/src/userEvents.ts:204-230`

```typescript
export async function trackInternalEvents(props: {
  workspaceId: string;
  events: InternalEvent[];
}): Promise<Result<void, Error>> {
  const { workspaceId } = props;
  
  const userEvents = props.events.map((p) => ({ ... }))
    .map((mr) => ({
      userId: mr.userId,
      messageId: mr.messageId,
      messageRaw: JSON.stringify(mr),
    }));

  await insertUserEvents({ workspaceId, userEvents });
  return ok(undefined);
}
```

---

## 三、查询构建阶段的 workspace_id 传递

### 3.1 事件查询总览

```
API Request (workspaceId in query/header/body)
    ↓
requestContext 中间件验证 workspace 访问权限
    ↓
eventsController 接收请求
    ↓
findManyEventsWithCount
    ↓
buildUserEventsQuery
    ├── buildWorkspaceIdClause (处理父子工作空间)
    └── buildUserEventQueryClauses (构建 WHERE 条件)
    ↓
chQuery (ClickHouse 查询)
```

### 3.2 请求上下文验证

位置：`packages/api/src/buildApp/requestContext.ts:39-95`

```typescript
fastify.addHook("preHandler", async (request, reply) => {
  const rc = await getRequestContextFastify(request);
  // 处理认证错误...

  const requestWorkspaceIdResult = await getWorkspaceId(request);
  if (requestWorkspaceIdResult.isErr()) {
    return reply.status(400).send();
  }

  const { workspace, member, memberRoles } = rc.value;
  const workspaceId = requestWorkspaceIdResult.value;
  
  // 关键：验证请求中的 workspaceId 与认证上下文匹配
  if (workspaceId !== workspace.id) {
    logger().error({ workspaceId, workspaceIdFromRequest: workspace.id },
      "workspace id does not match");
    return reply.status(403).send();
  }

  request.requestContext.set("workspace", workspace);
  request.requestContext.set("member", member);
  request.requestContext.set("memberRoles", memberRoles);
});
```

### 3.3 workspaceId 获取逻辑

位置：`packages/api/src/workspace.ts:29-88`

```typescript
export function getWorkspaceIdFromReq(
  req: FastifyRequest,
): Result<string | null, GetWorkspaceIdentifierError> {
  const workspaceIdSources: unknown[] = [req.body, req.query];
  const workspaceIdValues: string[] = [];

  // 1. 从 body 和 query 参数中提取
  for (const source of workspaceIdSources) {
    const result = schemaValidate(source, withWorkspaceId);
    if (result.isOk()) {
      workspaceIdValues.push(result.value.workspaceId);
    }
  }

  // 2. 从 header 中提取 (x-workspace-id)
  const workspaceIdHeader = req.headers[WORKSPACE_ID_HEADER];
  if (typeof workspaceIdHeader === "string") {
    workspaceIdValues.push(workspaceIdHeader);
  }

  // 3. 验证所有来源的 workspaceId 一致
  const distinctValues = new Set(workspaceIdValues);
  if (distinctValues.size > 1) {
    return err({ type: "MismatchedWorkspaceIds", ... });
  }

  // 4. 验证 UUID 格式
  const [workspaceId] = workspaceIdValues;
  if (workspaceId && !validateUuid(workspaceId)) {
    return err({ type: "InvalidWorkspaceId", ... });
  }
  return ok(workspaceId);
}
```

### 3.4 查询条件构建

位置：`packages/backend-lib/src/userEvents.ts:311-466`

```typescript
function buildUserEventQueryClauses(
  params: GetEventsRequest,
  qb: ClickHouseQueryBuilder,
) {
  const { workspaceId, startDate, endDate, userId, ... } = params;

  // 基础 workspace_id 过滤
  const workspaceIdClause = `workspace_id = ${qb.addQueryValue(workspaceId, "String")}`;

  // 其他过滤条件...
  const startDateClause = startDate
    ? `AND processing_time >= ${qb.addQueryValue(startDate, "DateTime64(3)")}`
    : "";

  // 内部事件优化查询（使用 internal_events 表）
  const hasInternalEventFilters = broadcastId || journeyId || hasAllInternalEvents;
  
  const internalEventsConditions: string[] = [];
  if (hasInternalEventFilters) {
    internalEventsConditions.push(
      `workspace_id = ${qb.addQueryValue(workspaceId, "String")}`,  // 再次过滤
    );
    // ... 其他条件
  }

  return { workspaceIdClause, ..., internalEventsConditions };
}
```

### 3.5 父子工作空间查询处理

位置：`packages/backend-lib/src/userEvents.ts:586-623`

```typescript
export async function buildUserEventsQuery(
  params: GetEventsRequest,
  qb: ClickHouseQueryBuilder,
  includeContext?: boolean,
): Promise<{ query: string; queryParams: Record<string, unknown>; }> {
  const { workspaceId, limit = 100, offset = 0 } = params;

  // 关键：构建包含子工作空间的 workspace_id 条件
  const workspaceIdClause = await buildWorkspaceIdClause(workspaceId, qb);
  const queryClauses = buildUserEventQueryClauses(params, qb);

  const innerQuery = buildUserEventInnerQuery(
    { ...queryClauses, workspaceIdClause },
    includeContext,
  );

  return {
    query: `${innerQuery} LIMIT ${offset},${limit}`,
    queryParams: qb.getQueries(),
  };
}
```

### 3.6 内部查询构建示例

位置：`packages/backend-lib/src/userEvents.ts:484-584`

```typescript
function buildUserEventInnerQuery(...) {
  // 两阶段查询模式：先从 internal_events 过滤，再关联主表
  if (hasInternalEventFilters && internalEventsConditions.length > 0) {
    return `
      SELECT workspace_id, user_id, ...
      FROM user_events_v2
      WHERE
        ${workspaceIdClause}
        AND (workspace_id, processing_time, user_or_anonymous_id, event_time, message_id) IN (
          SELECT
            workspace_id, processing_time, user_or_anonymous_id, event_time, message_id
          FROM internal_events
          WHERE ${internalEventsConditions.join(" AND ")}
        )
        ${userIdClause}
        ...
    `;
  }

  // 普通查询模式
  return `
    SELECT workspace_id, user_id, ...
    FROM user_events_v2
    WHERE
      ${workspaceIdClause}
      ${startDateClause}
      ${endDateClause}
      ...
  `;
}
```

---

## 四、ClickHouse 表结构与隔离设计

### 4.1 核心表结构

#### user_events_v2（主事件表）
位置：`packages/backend-lib/src/userEvents/clickhouse.ts:299-365`

```sql
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
  processing_time DateTime64(3),
  server_time DateTime64(3),
  message_raw String,
  workspace_id String,  -- 隔离关键字段
  INDEX message_id_idx message_id TYPE bloom_filter(0.01) GRANULARITY 4
)
ENGINE = MergeTree()
ORDER BY (
  workspace_id,          -- 排序键首位，确保同 workspace 数据物理相邻
  processing_time,
  user_or_anonymous_id,
  event_time,
  message_id
);
```

**关键设计**：
- `workspace_id` 是排序键（ORDER BY）的第一个字段
- 这确保同一工作空间的数据在物理存储上相邻
- 查询时 ClickHouse 可以快速定位到对应 workspace 的数据范围

#### internal_events（内部事件表）
位置：`packages/backend-lib/src/userEvents/clickhouse.ts:17-43`

```sql
CREATE TABLE IF NOT EXISTS internal_events (
  workspace_id String,
  user_or_anonymous_id String,
  user_id String,
  anonymous_id String,
  message_id String,
  event String,
  event_time DateTime64(3),
  processing_time DateTime64(3),
  properties String,
  template_id String,
  broadcast_id String,
  journey_id String,
  ...
)
ENGINE = MergeTree()
ORDER BY (workspace_id, processing_time, event, user_or_anonymous_id, message_id);
```

#### computed_property_assignments_v2（计算属性分配表）
位置：`packages/backend-lib/src/userEvents/clickhouse.ts:380-397`

```sql
CREATE TABLE IF NOT EXISTS computed_property_assignments_v2 (
  workspace_id LowCardinality(String),  -- 使用 LowCardinality 优化
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

### 4.2 物化视图的数据传播

#### internal_events 的 MV
位置：`packages/backend-lib/src/userEvents/clickhouse.ts:45-69`

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS internal_events_mv
TO internal_events
AS SELECT
  workspace_id,  -- 从源表继承
  user_or_anonymous_id,
  ...
FROM user_events_v2
WHERE event_type = 'track' AND startsWith(event, 'DF');
```

#### computed_property_state_v3 分区设计
位置：`packages/backend-lib/src/userEvents/clickhouse.ts:131-157`

```sql
CREATE TABLE IF NOT EXISTS computed_property_state_v3 (
  workspace_id LowCardinality(String),
  ...
)
ENGINE = AggregatingMergeTree()
PARTITION BY (
  workspace_id,        -- 按 workspace_id 分区！
  toYear(event_time)
)
ORDER BY (workspace_id, type, computed_property_id, state_id, user_id, event_time);
```

**分区设计优势**：
- 每个工作空间每年的数据在独立的分区中
- 删除工作空间数据时可以直接 DROP PARTITION
- 查询时 ClickHouse 可以跳过不相关的分区

### 4.3 冷存储表
位置：`packages/backend-lib/src/userEvents/clickhouse.ts:516-530`

```sql
CREATE TABLE IF NOT EXISTS user_events_cold_storage (
  message_raw String,
  processing_time DateTime64(3),
  workspace_id String,
  message_id String,
  server_time DateTime64(3)
)
ENGINE = MergeTree()
PARTITION BY (workspace_id, toYYYYMM(processing_time))  -- 按 workspace 分区
ORDER BY (workspace_id, processing_time, message_id)
SETTINGS storage_policy = 'cold_storage';
```

---

## 五、PostgreSQL 数据库隔离

### 5.1 表设计模式

几乎所有 PostgreSQL 业务表都遵循以下模式（`packages/backend-lib/src/db/schema.ts`）：

| 表名 | workspaceId 字段 | 外键约束 | 唯一索引 |
|------|------------------|----------|----------|
| Workspace | -（主键表） | - | - |
| Segment | workspaceId | ON DELETE CASCADE | (workspaceId, name) |
| UserProperty | workspaceId | ON DELETE CASCADE | (workspaceId, name) |
| Journey | workspaceId | ON DELETE CASCADE | (workspaceId, name) |
| Broadcast | workspaceId | ON DELETE CASCADE | (workspaceId, name) |
| EmailTemplate | workspaceId | ON DELETE CASCADE | - |
| Secret | workspaceId | ON DELETE CASCADE | (workspaceId, name) |
| WriteKey | workspaceId | ON DELETE CASCADE | - |
| Integration | workspaceId | ON DELETE CASCADE | (workspaceId, name) |
| SegmentAssignment | workspaceId | ON DELETE CASCADE | (workspaceId, userId, segmentId) |
| UserPropertyAssignment | workspaceId | ON DELETE CASCADE | (workspaceId, userPropertyId, userId) |
| WorkspaceMemberRole | workspaceId | ON DELETE CASCADE | (workspaceId, workspaceMemberId) |

### 5.2 外键约束示例

```typescript
// Segment 表示例 (schema.ts:736-780)
export const segment = pgTable("Segment", {
  id: uuid().primaryKey().defaultRandom().notNull(),
  workspaceId: uuid().notNull(),  // 必需字段
  name: text().notNull(),
  // ...
}, (table) => [
  uniqueIndex("Segment_workspaceId_name_key")
    .using("btree", table.workspaceId.asc(), table.name.asc()),
  foreignKey({
    columns: [table.workspaceId],
    foreignColumns: [workspace.id],
    name: "Segment_workspaceId_fkey",
  })
    .onUpdate("cascade")
    .onDelete("cascade"),  // 删除 workspace 时级联删除
]);
```

### 5.3 查询时的 workspace 过滤

位置：`packages/backend-lib/src/segments.ts` (示例)

```typescript
// 所有查询都显式包含 workspaceId 过滤
export async function findSegments({ workspaceId }: { workspaceId: string }) {
  return db().query.segment.findMany({
    where: eq(dbSegment.workspaceId, workspaceId),
  });
}
```

---

## 六、跨数据库数据同步与隔离

### 6.1 计算属性同步流程

ClickHouse 的 `computed_property_assignments_v2` 数据会同步到 PostgreSQL 的 `SegmentAssignment` 和 `UserPropertyAssignment` 表。

**关键**：同步过程中始终携带 workspace_id。

### 6.2 ClickHouse 到 PostgreSQL 的数据流动

```
ClickHouse (computed_property_assignments_v2)
    ↓
processed_computed_properties_v2 (标记已处理)
    ↓
定时任务/工作流读取
    ↓
PostgreSQL (SegmentAssignment, UserPropertyAssignment)
    ↓ workspaceId 始终伴随
```

位置：`packages/backend-lib/src/userEvents/clickhouse.ts:280-291`

```typescript
export async function insertProcessedComputedProperties({
  assignments,
}: {
  assignments: ComputedPropertyAssignment[];
}) {
  await clickhouseClient().insert({
    table: `processed_computed_properties_v2 (
      workspace_id, user_id, type, computed_property_id, 
      segment_value, user_property_value, processed_for, processed_for_type
    )`,
    values: assignments,  // assignments 包含 workspace_id
    format: "JSONEachRow",
  });
}
```

---

## 七、工作空间生命周期管理

### 7.1 暂停/恢复工作空间

位置：`packages/backend-lib/src/workspaces.ts:257-294`

```typescript
export async function pauseWorkspace(
  { workspaceId }: { workspaceId: string },
  options: { enableColdStorageOverride?: boolean } = {},
) {
  // 1. 更新 PostgreSQL 状态
  await db()
    .update(dbWorkspace)
    .set({ status: WorkspaceStatusDbEnum.Paused })
    .where(eq(dbWorkspace.id, workspaceId));
  
  // 2. 可选：将 ClickHouse 数据移到冷存储
  const enableColdStorage = options.enableColdStorageOverride ?? config().enableColdStorage;
  if (enableColdStorage) {
    await coldStoreWorkspaceEvents({ workspaceId });
  }
}

export async function resumeWorkspace({ workspaceId }, options = {}) {
  // 1. 恢复 PostgreSQL 状态
  await db()
    .update(dbWorkspace)
    .set({ status: WorkspaceStatusDbEnum.Active })
    .where(eq(dbWorkspace.id, workspaceId));
  
  // 2. 从冷存储恢复数据
  if (enableColdStorage) {
    await restoreWorkspaceEvents({ workspaceId });
  }
}
```

### 7.2 冷存储数据移动

位置：`packages/backend-lib/src/workspaces.ts:26-124`

```typescript
export async function coldStoreWorkspaceEvents({ workspaceId }) {
  const qb = new ClickHouseQueryBuilder();
  const ws = qb.addQueryValue(workspaceId, "String");

  // 1. 复制到冷存储表
  await chCommand({
    query: `
      INSERT INTO user_events_cold_storage (...)
      SELECT ... FROM user_events_v2
      WHERE workspace_id = ${ws}  -- 按 workspace_id 过滤
    `,
    query_params: qb.getQueries(),
  });

  // 2. 删除热数据
  await Promise.all([
    chCommand({
      query: `DELETE FROM user_events_v2 WHERE workspace_id = ${ws}`,
      query_params: qb.getQueries(),
    }),
    chCommand({
      query: `DELETE FROM internal_events WHERE workspace_id = ${ws}`,
      query_params: qb.getQueries(),
    }),
  ]);
}
```

### 7.3 逻辑删除（Tombstone）

位置：`packages/backend-lib/src/workspaces.ts:133-177`

```typescript
export async function tombstoneWorkspace(workspaceId, options = {}) {
  const result = await db().transaction(async (tx) => {
    const workspace = await tx.query.workspace.findFirst({
      where: eq(dbWorkspace.id, workspaceId),
    });
    if (!workspace || workspace.status !== WorkspaceStatusDbEnum.Active) {
      return ok(undefined);
    }
    
    // 标记删除：修改名称和状态，不物理删除
    const externalId = workspace.externalId
      ? `${WORKSPACE_TOMBSTONE_PREFIX}-${workspace.externalId}`
      : undefined;

    await tx.update(dbWorkspace).set({
      name: `${WORKSPACE_TOMBSTONE_PREFIX}-${workspace.name}`,
      externalId,
      status: WorkspaceStatusDbEnum.Tombstoned,
    }).where(eq(dbWorkspace.id, workspaceId));
    
    return ok(undefined);
  });

  // 可选：移动数据到冷存储
  if (enableColdStorage) {
    await coldStoreWorkspaceEvents({ workspaceId });
  }
}
```

---

## 八、接口返回阶段

### 8.1 事件查询接口

位置：`packages/api/src/controllers/eventsController.ts:22-77`

```typescript
fastify.withTypeProvider<TypeBoxTypeProvider>().get("/", {
  schema: {
    querystring: GetEventsRequest,  // 包含 workspaceId 字段
    response: { 200: GetEventsResponse },
  },
}, async (request, reply) => {
  const { events: eventsRaw, count } = await findManyEventsWithCount({
    ...request.query,  // workspaceId 从 query 传入
    abortSignal: abortController.signal,
  });

  // 结果转换（不包含 workspace_id，因为调用方已知）
  const events: GetEventsResponseItem[] = eventsRaw.flatMap(({
    message_id, processing_time, user_id, event_type, ...
  }) => ({
    messageId: message_id,
    processingTime: processing_time,
    userId: user_id,
    eventType: event_type,
    // ... 其他字段
  }));

  return reply.status(200).send({ events, count });
});
```

### 8.2 认证后的访问控制

位置：`packages/api/src/buildApp/requestContext.ts:72-88`

```typescript
const requestWorkspaceIdResult = await getWorkspaceId(request);
const { workspace, member, memberRoles } = rc.value;
const workspaceId = requestWorkspaceIdResult.value;

// 强制验证：请求的 workspaceId 必须与认证上下文匹配
if (workspaceId !== workspace.id) {
  logger().error({ workspaceId, workspaceIdFromRequest: workspace.id },
    "workspace id does not match");
  return reply.status(403).send();
}
```

---

## 九、隔离边界总结

### 9.1 ClickHouse 隔离边界

| 层级 | 实现方式 | 关键位置 |
|------|----------|----------|
| 表设计 | 所有表包含 `workspace_id` 字段 | userEvents/clickhouse.ts |
| 排序键 | `workspace_id` 作为排序键首位 | ORDER BY (workspace_id, ...) |
| 分区 | 部分表按 `workspace_id` 分区 | PARTITION BY (workspace_id, ...) |
| 写入 | 所有 INSERT 携带 `workspace_id` | insertUserEventsDirect |
| 查询 | 所有 SELECT 强制 `workspace_id` 过滤 | buildUserEventQueryClauses |
| MV传播 | 物化视图继承源表 `workspace_id` | internal_events_mv 等 |

### 9.2 PostgreSQL 隔离边界

| 层级 | 实现方式 | 关键位置 |
|------|----------|----------|
| 表设计 | 所有业务表含 `workspaceId` 外键 | db/schema.ts |
| 外键约束 | `ON DELETE CASCADE` | 各表 foreignKey 定义 |
| 唯一约束 | 联合唯一索引包含 `workspaceId` | uniqueIndex 定义 |
| 查询 | 所有查询显式 `eq(workspaceId)` | 各业务模块 |

### 9.3 认证层隔离边界

| 场景 | 认证方式 | workspace_id 来源 |
|------|----------|-------------------|
| 公共事件 API | WriteKey (Basic Auth) | Secret → workspaceId |
| 管理 API | JWT + Session | requestContext + header/query |
| 内部服务 | 直接传递 | 函数参数 |

### 9.4 潜在风险点

1. **原生 SQL 注入风险**：使用 `ClickHouseQueryBuilder` 参数化查询避免
2. **跨 workspace 数据泄露**：通过 `requestContext` 中间件验证防止
3. **父子工作空间数据混淆**：`buildWorkspaceIdClause` 显式处理
4. **工作空间删除不完全**：级联删除 + 逻辑删除 + 冷存储三重保障

---

## 十、关键代码文件索引

| 功能 | 文件路径 |
|------|----------|
| ClickHouse 连接与查询构建 | packages/backend-lib/src/clickhouse.ts |
| ClickHouse 表结构定义 | packages/backend-lib/src/userEvents/clickhouse.ts |
| 用户事件写入/查询核心 | packages/backend-lib/src/userEvents.ts |
| 工作空间生命周期管理 | packages/backend-lib/src/workspaces.ts |
| WriteKey 认证 | packages/backend-lib/src/auth.ts |
| PostgreSQL 表结构 | packages/backend-lib/src/db/schema.ts |
| 请求上下文与权限验证 | packages/backend-lib/src/requestContext.ts |
| 公共事件 API 控制器 | packages/api/src/controllers/publicAppsController.ts |
| 事件查询 API 控制器 | packages/api/src/controllers/eventsController.ts |
| API 请求上下文中间件 | packages/api/src/buildApp/requestContext.ts |
| workspaceId 提取逻辑 | packages/api/src/workspace.ts |
| 事件提交封装 | packages/backend-lib/src/apps.ts |
