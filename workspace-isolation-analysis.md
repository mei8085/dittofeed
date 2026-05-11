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

**行为语义**：
- 无子工作空间：返回 `workspace_id = 'xxx'`
- 有子工作空间：返回 `workspace_id IN ('child1', 'child2', ...)`
- **注意**：Parent 类型工作空间本身不接收事件（`canWorkspaceReceiveEvents` 排除 Parent），数据实际存储在 Child 工作空间中

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

### 3.4 查询条件构建（普通查询）

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
    { ...queryClauses, workspaceIdClause },  // 覆盖 queryClauses 中的 workspaceIdClause
    includeContext,
  );

  return {
    query: `${innerQuery} LIMIT ${offset},${limit}`,
    queryParams: qb.getQueries(),
  };
}
```

**关键细节**：
- `buildUserEventsQuery` 先调用 `buildWorkspaceIdClause` 处理父子关系
- 然后用结果**覆盖** `queryClauses` 中的 `workspaceIdClause`
- 这种设计意味着外层 WHERE 条件是正确的，但嵌入在子查询字符串中的条件可能不一致

### 3.5 按 messageId 查询的语义

位置：`packages/backend-lib/src/userEvents.ts:342-374`

```typescript
function buildUserEventQueryClauses(...) {
  // 注意：这里使用的是固定值，不包含子工作空间
  const workspaceIdClause = `workspace_id = ${qb.addQueryValue(workspaceId, "String")}`;

  let messageIdClause = "";
  if (messageId) {
    let messageIdWhereClause: string;
    if (typeof messageId === "string") {
      messageIdWhereClause = `AND message_id = ${qb.addQueryValue(messageId, "String")}`;
    } else {
      messageIdWhereClause = `AND message_id IN ${qb.addQueryValue(messageId, "Array(String)")}`;
    }

    // 关键：messageIdClause 内部嵌入了固定的 workspaceIdClause
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
          ${workspaceIdClause}  -- 这里使用的是固定值！
          ${messageIdWhereClause}
        GROUP BY
          workspace_id,
          user_or_anonymous_id,
          message_id
      )
    `;
  }
  // ...
}
```

**语义差异**：

| 场景 | 子查询条件 | 父子支持 |
|------|-----------|----------|
| 普通查询（无 messageId） | 外层使用 buildWorkspaceIdClause | ✓ 支持 |
| messageId 查询 | 子查询内部使用固定 `workspace_id = ?` | ✗ 不支持 |

**影响**：当用父工作空间的 ID 查询指定 messageId 的事件时：
- 外层 WHERE：`workspace_id IN ('child1', 'child2', ...)`
- messageId 子查询：`workspace_id = 'parent-id'`
- 由于 Parent 工作空间不接收事件，子查询可能返回空结果

### 3.6 Internal Events 两阶段查询的语义

位置：`packages/backend-lib/src/userEvents.ts:386-465`

```typescript
function buildUserEventQueryClauses(...) {
  // 内部事件过滤条件（用于两阶段查询）
  const internalEventsConditions: string[] = [];
  if (hasInternalEventFilters) {
    internalEventsConditions.push(
      `workspace_id = ${qb.addQueryValue(workspaceId, "String")}`,  // 固定值！
    );

    if (startDate) {
      internalEventsConditions.push(
        `processing_time >= ${qb.addQueryValue(startDate, "DateTime64(3)")}`,
      );
    }
    if (endDate) {
      internalEventsConditions.push(
        `processing_time <= ${qb.addQueryValue(endDate, "DateTime64(3)")}`,
      );
    }
    if (broadcastId) {
      internalEventsConditions.push(
        `broadcast_id = ${qb.addQueryValue(broadcastId, "String")}`,
      );
    }
    if (journeyId) {
      internalEventsConditions.push(
        `journey_id = ${qb.addQueryValue(journeyId, "String")}`,
      );
    }
    if (hasAllInternalEvents) {
      internalEventsConditions.push(
        `event IN ${qb.addQueryValue(dfEvents, "Array(String)")}`,
      );
    }
  }
  // ...
}
```

位置：`packages/backend-lib/src/userEvents.ts:510-545`

```typescript
function buildUserEventInnerQuery(...) {
  // 两阶段查询模式
  if (hasInternalEventFilters && internalEventsConditions.length > 0) {
    return `
      SELECT ...
      FROM user_events_v2
      WHERE
        ${workspaceIdClause}  -- 外层：可能是 IN 子句
        AND (workspace_id, processing_time, user_or_anonymous_id, event_time, message_id) IN (
          SELECT
            workspace_id, processing_time, user_or_anonymous_id, event_time, message_id
          FROM internal_events
          WHERE ${internalEventsConditions.join(" AND ")}  -- 内层：固定 workspace_id = ?
        )
        ...
    `;
  }
  // ...
}
```

**语义差异**：

| 层级 | 查询条件 | 父子支持 |
|------|----------|----------|
| 外层 user_events_v2 | `buildWorkspaceIdClause` 结果（可能是 IN） | ✓ |
| 内层 internal_events 子查询 | 固定 `workspace_id = ?` | ✗ |

**影响**：当用父工作空间的 ID 按 broadcastId 或 journeyId 查询时：
- 外层：查子工作空间的数据
- 内层 internal_events 子查询：查父工作空间本身的数据
- 由于 Parent 不接收事件，内层子查询可能返回空
- **结果**：整个查询可能返回空，而实际上子工作空间中有数据

### 3.7 对比 users.ts 中的正确模式

位置：`packages/backend-lib/src/users.ts:166-261`

```typescript
// users.ts 中的模式：先定义可复用的函数
const buildWorkspaceIdClause = (qb: ClickHouseQueryBuilder) =>
  childWorkspaceIds.length > 0
    ? `workspace_id IN (${qb.addQueryValue(childWorkspaceIds, "Array(String)")})`
    : `workspace_id = ${qb.addQueryValue(workspaceId, "String")}`;

// 在需要的地方调用此函数
const workspaceIdClause = buildWorkspaceIdClause(qb);
```

**差异**：
- `users.ts`：在构建所有条件**之前**获取 childWorkspaceIds，然后通过函数调用注入
- `userEvents.ts`：`buildUserEventQueryClauses` 先构建条件（使用固定值），外层再尝试覆盖

### 3.8 其他查询函数的父子支持状态

| 函数 | 位置 | workspace_id 条件 | 父子支持 |
|------|------|------------------|----------|
| `findManyInternalEvents` | userEvents.ts:153-173 | `workspace_id = {workspaceId}` | ✗ |
| `findUserIdByMessageId` | userEvents.ts:175-195 | `workspace_id = {workspaceId}` | ✗ |
| `findIdentifyTraits` | userEvents.ts:232-260 | `workspace_id = {workspaceId}` | ✗ |
| `findTrackProperties` | userEvents.ts:262-307 | `workspace_id = ${workspaceIdParam}` | ✗ |
| `findUserEventsById` | userEvents.ts:922-976 | `workspace_id = ?`（可选！） | ✗ |
| `buildUserEventsQuery` | userEvents.ts:586-623 | 调用 `buildWorkspaceIdClause` | ✓ |
| `findUserEventCount` | userEvents.ts:689-754 | 调用 `buildWorkspaceIdClause` | ✓ |

**注意**：`findUserEventsById` 的 `workspaceId` 参数是可选的！如果不传，查询将没有 workspace_id 过滤条件。

---

## 四、四类接口的完整传递链

### 4.1 接口参数定义

位置：`packages/isomorphic-lib/src/types.ts`

```typescript
// events 接口
export const GetEventsRequest = Type.Object({
  workspaceId: Type.String(),
  searchTerm: Type.Optional(Type.String()),
  userId: Type.Optional(UserId),
  offset: Type.Optional(Type.Number()),
  limit: Type.Optional(Type.Number()),
  messageId: Type.Optional(Type.Union([Type.String(), Type.Array(Type.String())])),
  startDate: Type.Optional(Type.Number()),
  endDate: Type.Optional(Type.Number()),
  event: Type.Optional(Type.Array(Type.String())),
  broadcastId: Type.Optional(Type.String()),
  journeyId: Type.Optional(Type.String()),
  eventType: Type.Optional(Type.String()),
  includeContext: Type.Optional(Type.Boolean()),
});

// download 接口（与 events 基本一致，去掉 offset/limit）
export const DownloadEventsRequest = Type.Omit(GetEventsRequest, ["offset", "limit"]);

// traits 接口（仅 workspaceId）
export const GetTraitsRequest = Type.Object({
  workspaceId: Type.String(),
});

// properties 接口（仅 workspaceId）
export const GetPropertiesRequest = Type.Object({
  workspaceId: Type.String(),
});
```

### 4.2 Events 接口传递链

**接口**：`GET /api/events`

**完整路径**：
```
HTTP Request (query: workspaceId, searchTerm, startDate, endDate, etc.)
    ↓
requestContext 中间件（验证 JWT，提取 workspaceId）
    ↓
eventsController.GET / (eventsController.ts:22-77)
    ↓
findManyEventsWithCount(request.query)
    ├── findUserEvents(params)
    │   └── buildUserEventsQuery(params)
    │       ├── buildWorkspaceIdClause(workspaceId, qb)  ✓ 父子扩展
    │       └── buildUserEventQueryClauses(params, qb)
    └── findUserEventCount(params)
        └── buildWorkspaceIdClause(workspaceId, qb)  ✓ 父子扩展
    ↓
chQuery() → ClickHouse
    ↓
返回转换（剔除 workspace_id 字段）
    ↓
HTTP Response { events: [...], count: N }
```

**父子支持**：✓（但 messageId 和 internal events 过滤存在语义差异）

### 4.3 Traits 接口传递链

**接口**：`GET /api/events/traits`

**完整路径**：
```
HTTP Request (query: workspaceId)
    ↓
requestContext 中间件
    ↓
eventsController.GET /traits (eventsController.ts:79-97)
    ↓
findIdentifyTraits({ workspaceId: request.query.workspaceId })
    ↓
ClickHouse 查询（userEvents.ts:232-260）
    ├── 直接使用 `workspace_id = {workspaceId}`
    ├── 无父子扩展
    └── 查询 identify 事件的 properties 键
    ↓
返回 { traits: ["email", "name", ...] }
```

**父子支持**：✗

位置：`packages/backend-lib/src/userEvents.ts:232-260`

```typescript
export async function findIdentifyTraits({
  workspaceId,
  limit = 500,
}: {
  workspaceId: string;
  limit?: number;
}): Promise<string[]> {
  const query = `
    SELECT DISTINCT
      arrayJoin(JSONExtractKeys(properties)) AS trait
    FROM user_events_v2
    WHERE
      workspace_id = {workspaceId:String}  -- 固定值，无子扩展
      and event_type = 'identify'
    limit {limit:Int32}
  `;
  // ...
}
```

### 4.4 Properties 接口传递链

**接口**：`GET /api/events/properties`

**完整路径**：
```
HTTP Request (query: workspaceId)
    ↓
requestContext 中间件
    ↓
eventsController.GET /properties (eventsController.ts:99-117)
    ↓
findTrackProperties({ workspaceId: request.query.workspaceId })
    ↓
ClickHouse 查询（userEvents.ts:262-307）
    ├── 直接使用 `workspace_id = ${workspaceIdParam}`
    ├── 无父子扩展
    └── 查询 track 事件的 properties 键
    ↓
返回 { properties: { "event1": ["prop1", "prop2"], ... } }
```

**父子支持**：✗

位置：`packages/backend-lib/src/userEvents.ts:262-307`

```typescript
export async function findTrackProperties({
  workspaceId,
  limit = 500,
}: {
  workspaceId: string;
  limit?: number;
}): Promise<GetPropertiesResponse["properties"]> {
  const qb = new ClickHouseQueryBuilder();
  const workspaceIdParam = qb.addQueryValue(workspaceId, "String");  // 固定值
  const limitParam = qb.addQueryValue(limit, "Int32");
  const query = `
    SELECT
      arrayJoin(JSONExtractKeys(properties)) AS property,
      max(processing_time) AS max_processing_time,
      event
    FROM user_events_v2
    WHERE
      workspace_id = ${workspaceIdParam}  -- 固定值，无子扩展
      and event_type = 'track'
    GROUP BY property, event
    ORDER BY max_processing_time DESC
    LIMIT ${limitParam}
  `;
  // ...
}
```

### 4.5 Download 接口传递链

**接口**：`GET /api/events/download`

**完整路径**：
```
HTTP Request (query: workspaceId, startDate, endDate, etc.)
    ↓
requestContext 中间件
    ↓
eventsController.GET /download (eventsController.ts:119-140)
    ↓
buildEventsFile(request.query)
    ├── findManyEventsWithCount(params)
    │   ├── findUserEvents(params)
    │   │   └── buildUserEventsQuery(params)
    │   │       └── buildWorkspaceIdClause  ✓ 父子扩展
    │   └── findUserEventCount(params)
    │       └── buildWorkspaceIdClause  ✓ 父子扩展
    └── 构建 CSV 内容
    ↓
返回 CSV 文件（Content-Type: text/csv）
```

**父子支持**：✓（与 events 接口存在相同的语义差异）

### 4.6 四类接口对比汇总

| 接口 | 路径 | 核心函数 | 父子支持 | 说明 |
|------|------|----------|----------|------|
| events | `/api/events` | `buildUserEventsQuery` | ✓ | 支持，但 messageId / internal events 过滤存在差异 |
| traits | `/api/events/traits` | `findIdentifyTraits` | ✗ | 直接 `workspace_id = ?` |
| properties | `/api/events/properties` | `findTrackProperties` | ✗ | 直接 `workspace_id = ?` |
| download | `/api/events/download` | `buildEventsFile` | ✓ | 复用 events 查询逻辑 |

---

## 五、ClickHouse 表结构与隔离设计

### 5.1 核心表结构

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

### 5.2 物化视图的数据传播

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

### 5.3 冷存储表
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

## 六、PostgreSQL 数据库隔离

### 6.1 表设计模式

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

### 6.2 外键约束示例

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

### 6.3 查询时的 workspace 过滤

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

## 七、跨数据库数据同步与隔离

### 7.1 计算属性同步流程

ClickHouse 的 `computed_property_assignments_v2` 数据会同步到 PostgreSQL 的 `SegmentAssignment` 和 `UserPropertyAssignment` 表。

**关键**：同步过程中始终携带 workspace_id。

### 7.2 ClickHouse 到 PostgreSQL 的数据流动

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

## 八、工作空间生命周期管理

### 8.1 暂停/恢复工作空间

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

### 8.2 冷存储数据移动

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

### 8.3 逻辑删除（Tombstone）

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

## 九、接口返回阶段

### 9.1 事件查询接口

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

### 9.2 认证后的访问控制

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

## 十、隔离边界总结

### 10.1 ClickHouse 隔离边界

| 层级 | 实现方式 | 关键位置 |
|------|----------|----------|
| 表设计 | 所有表包含 `workspace_id` 字段 | userEvents/clickhouse.ts |
| 排序键 | `workspace_id` 作为排序键首位 | ORDER BY (workspace_id, ...) |
| 分区 | 部分表按 `workspace_id` 分区 | PARTITION BY (workspace_id, ...) |
| 写入 | 所有 INSERT 携带 `workspace_id` | insertUserEventsDirect |
| 查询 | 所有 SELECT 强制 `workspace_id` 过滤 | buildUserEventQueryClauses |
| MV传播 | 物化视图继承源表 `workspace_id` | internal_events_mv 等 |

### 10.2 PostgreSQL 隔离边界

| 层级 | 实现方式 | 关键位置 |
|------|----------|----------|
| 表设计 | 所有业务表含 `workspaceId` 外键 | db/schema.ts |
| 外键约束 | `ON DELETE CASCADE` | 各表 foreignKey 定义 |
| 唯一约束 | 联合唯一索引包含 `workspaceId` | uniqueIndex 定义 |
| 查询 | 所有查询显式 `eq(workspaceId)` | 各业务模块 |

### 10.3 认证层隔离边界

| 场景 | 认证方式 | workspace_id 来源 |
|------|----------|-------------------|
| 公共事件 API | WriteKey (Basic Auth) | Secret → workspaceId |
| 管理 API | JWT + Session | requestContext + header/query |
| 内部服务 | 直接传递 | 函数参数 |

### 10.4 潜在风险点

1. **原生 SQL 注入风险**：使用 `ClickHouseQueryBuilder` 参数化查询避免
2. **跨 workspace 数据泄露**：通过 `requestContext` 中间件验证防止
3. **父子工作空间数据混淆**：`buildWorkspaceIdClause` 显式处理
4. **工作空间删除不完全**：级联删除 + 逻辑删除 + 冷存储三重保障

### 10.5 复核发现的语义差异问题

#### 问题 1：messageId 查询不支持父子工作空间

**影响函数**：`buildUserEventQueryClauses` 中 `messageIdClause` 构建

**问题描述**：
- 子查询内部嵌入的是固定 `workspace_id = ?`
- 即使外层被 `buildWorkspaceIdClause` 覆盖，子查询内部已固化

**修复建议**：参考 `users.ts` 模式，在构建条件前先获取 childWorkspaceIds

#### 问题 2：internal events 两阶段查询语义不一致

**影响函数**：`internalEventsConditions` 构建

**问题描述**：
- 外层 user_events_v2：`buildWorkspaceIdClause` 结果（可能 IN）
- 内层 internal_events：固定 `workspace_id = ?`
- 查询 Parent 工作空间时，内层查不到数据

**修复建议**：将 childWorkspaceIds 传入 `buildUserEventQueryClauses`

#### 问题 3：traits / properties 接口不支持父子

**影响函数**：`findIdentifyTraits`, `findTrackProperties`

**问题描述**：
- 直接使用固定 `workspace_id = ?`
- 与 events / download 接口行为不一致

**修复建议**：在函数内部调用 `buildWorkspaceIdClause` 或复用通用逻辑

#### 问题 4：findUserEventsById 中 workspaceId 可选

**影响函数**：`findUserEventsById`

**问题描述**：
- 参数 `workspaceId?: string`
- 如果不传，查询没有 workspace_id 过滤

**修复建议**：将 `workspaceId` 改为必需参数

---

## 十一、SendGrid 延迟事件回填链路深度分析

### 11.1 延迟事件回填的业务背景

SendGrid Webhook 事件分为两类：

| 事件类型 | 示例 | 是否包含 workspaceId | 处理方式 |
|---------|------|---------------------|---------|
| 即时事件 | open, click, delivered, dropped | 是（通过 sg_message_id + workspaceId 推断） | 直接处理 |
| 延迟事件 | bounce, spamreport | 否 | 需通过 smtp-id 回填 |

**延迟事件的问题**：
- bounce/spamreport 事件只有 `smtp-id`，没有 `sg_message_id`
- 无法直接从事件中提取 `workspaceId`
- 需要先通过 `smtp-id` 查找之前的 `processed` 事件

### 11.2 完整回填链路

**位置**：`packages/backend-lib/src/destinations/sendgrid.ts:217-370`

```
SendGrid Webhook Request (bounce/spamreport events)
    ↓
handleSendgridEvents()
    ↓
Step 1: 分类事件
    ├── immediateEvents: open, click, delivered, dropped
    └── delayedEvents: bounce, spamreport (by smtp-id)
    ↓
Step 2: 跨 workspace 查询 processed 事件  ← 风险点
    ├── findUserEventsById({ messageIds: ['processed:smtp1', ...] })
    │   └── 没有传递 workspaceId！
    └── ClickHouse 查询: WHERE message_id IN (...)
    ↓
Step 3: 后置校验（逐个事件）
    ├── 解析 properties 中的 workspaceId
    ├── 校验是否一致
    └── 不一致则跳过
    ↓
Step 4: 签名验证（按 workspace）
    ├── 查询 workspace 的 SendGrid secret
    └── verifyTimestampedSignature()
    ↓
Step 5: 写入事件
    └── submitSendgridEvents({ workspaceId, events })
```

### 11.3 findUserEventsById 的调用分析

**调用位置**：`packages/backend-lib/src/destinations/sendgrid.ts:267-269`

```typescript
const processedForDelayedEvents = await findUserEventsById({
  messageIds: Array.from(delayedEvents.keys()).map((id) => `processed:${id}`),
  // 没有传递 workspaceId！
});
```

**函数定义**：`packages/backend-lib/src/userEvents.ts:922-976`

```typescript
export async function findUserEventsById({
  messageIds,
  workspaceId,  // 可选参数
}: {
  messageIds: string[];
  workspaceId?: string;
}): Promise<UserEventsWithTraits[]> {
  // ...
  if (workspaceId) {
    clauses.push(`workspace_id = ${qb.addQueryValue(workspaceId, "String")}`);
  }
  // 不传 workspaceId 时，WHERE 只有 message_id IN (...)
}
```

**生成的 SQL（不传 workspaceId）**：
```sql
SELECT workspace_id, user_id, ...
FROM user_events_v2
WHERE message_id IN ('processed:smtp-id-1', 'processed:smtp-id-2', ...)
-- 没有 workspace_id 过滤！
```

**生成的 SQL（传 workspaceId）**：
```sql
SELECT workspace_id, user_id, ...
FROM user_events_v2
WHERE message_id IN ('processed:smtp-id-1', ...)
  AND workspace_id = 'ws-xxx'
```

### 11.4 后置校验逻辑分析

**位置**：`packages/backend-lib/src/destinations/sendgrid.ts:273-306`

```typescript
for (const event of processedForDelayedEvents) {
  const smtpId = event.message_id.split(":")[1];
  if (!smtpId) continue;

  const delayedEvent = delayedEvents.get(smtpId);
  if (!delayedEvent) continue;

  // 解析 processed 事件的 properties（包含 MESSAGE_METADATA_FIELDS）
  const parsedProperties = jsonParseSafeWithSchema(
    event.properties,
    MessageMetadataFields,
  );
  if (parsedProperties.isErr()) continue;

  const { workspaceId: processedWorkspaceId, userId } = parsedProperties.value;
  if (!processedWorkspaceId || !userId) continue;

  // 关键点：第一次匹配到的 workspaceId 成为期望值
  if (!workspaceId) {
    workspaceId = processedWorkspaceId;
  }

  // 不匹配则跳过（软校验）
  if (workspaceId !== processedWorkspaceId) {
    continue;
  }

  backfilledDelayedEvents.push({
    ...delayedEvent,
    ...parsedProperties.value,
  });
}
```

**校验逻辑拆解**：

| 步骤 | 逻辑 | 风险点 |
|------|------|--------|
| 1 | 从 `processed` 事件 properties 提取 `workspaceId` | 依赖数据完整性 |
| 2 | 第一个匹配到的值设为 `workspaceId` | 批量事件时顺序敏感 |
| 3 | 后续事件必须与此值一致 | 跨 workspace 批量事件会丢失 |
| 4 | 不一致则 `continue`（跳过）而非报错 | 静默失败，难以排查 |

### 11.5 可复现的误配场景

#### 场景 A：批量延迟事件跨 workspace 冲突

**前提条件**：
- Workspace A 和 Workspace B 共享同一个 SendGrid Webhook 端点
- 两个 workspace 的延迟事件在同一批次到达

**触发流程**：
1. Workspace A 发送邮件 → `smtp-id = "batch-001"`
2. Workspace B 发送邮件 → `smtp-id = "batch-002"`
3. SendGrid 同时推送两个 bounce 事件
4. 批次中没有 immediate events（只有 bounces）

**执行结果**：
```
workspaceId = undefined（初始值）

findUserEventsById 查询：
  message_id IN ('processed:batch-001', 'processed:batch-002')
  ↓
  返回两条记录：
    1. workspace_id = ws-B, message_id = processed:batch-002
    2. workspace_id = ws-A, message_id = processed:batch-001

后置校验：
  第 1 条：workspaceId = undefined → 设为 ws-B ✓
  第 2 条：workspaceId = ws-B, processedWorkspaceId = ws-A → 不匹配，跳过 ✗

最终：只有 Workspace B 的事件被处理，A 的 bounce 事件丢失
```

**为什么可能发生**：
- ClickHouse 查询结果顺序不确定（除非显式 ORDER BY）
- 先返回哪条记录取决于存储顺序

#### 场景 B：无效签名请求的性能攻击

**前提条件**：
- 攻击者知道 Webhook 端点
- 攻击者构造大量 bounce 事件

**触发流程**：
1. 攻击者发送 1000 个 bounce 事件，每个都有不同的 smtp-id
2. 即使签名无效，代码仍会执行到 Step 2
3. `findUserEventsById` 跨 workspace 查询 1000 个 messageId

**影响**：
- 每次无效请求都会触发 ClickHouse 查询
- 大量无效请求可能造成性能压力
- 签名验证在查询**之后**执行

**代码时序**：
```typescript
// Step 2: 查询在签名验证之前！
const processedForDelayedEvents = await findUserEventsById({...});

// ... 中间处理 ...

// Step 4: 签名验证在最后
const verified = verifyTimestampedSignature({...});
if (!verified) {
  return err({ message: "Invalid signature." });
}
```

#### 场景 C：父子工作空间的延迟事件处理

**前提条件**：
- Parent 工作空间 P 有子工作空间 C1、C2
- C1 发送邮件，workspace_id = C1
- SendGrid Webhook 配置在 P 的层面

**分析**：
- 延迟事件没有 workspaceId，只能通过 processed 事件推断
- processed 事件的 workspace_id 是 C1
- 推断出的 workspaceId = C1
- 签名验证使用 C1 的 secret（不是 P 的）

**这是否是问题**？
- 如果 Webhook 端点是按 workspace 隔离的（每个 workspace 有不同的 URL），则没问题
- 如果共享端点，则需要确保推断出正确的 workspace

### 11.6 隔离边界分析

#### 数据泄露风险评估

| 风险点 | 评估 | 说明 |
|--------|------|------|
| 跨 workspace 数据查询 | ⚠️ 中等 | 确实跨 workspace 查询了，但结果不会被误用 |
| 错误 workspace 写入 | ✅ 低 | 后置校验和签名验证双重保障 |
| 数据泄露给调用方 | ✅ 低 | 查询结果在服务内部使用，不对外暴露 |

**结论**：
- 不存在跨 workspace 数据泄露风险
- 后置校验确保只有匹配的事件被处理
- 签名验证确保只有合法请求能写入数据

#### 性能影响评估

| 场景 | 有 workspaceId | 无 workspaceId |
|------|---------------|---------------|
| 查询范围 | 单 workspace | 全表 |
| 索引使用 | 排序键前缀 workspace_id + message_id | 仅 message_id bloom filter |
| 性能 | 最优 | 次之 |

**ClickHouse 表结构**：
```sql
ORDER BY (workspace_id, processing_time, user_or_anonymous_id, event_time, message_id)
INDEX message_id_idx message_id TYPE bloom_filter(0.01) GRANULARITY 4
```

**分析**：
- 传 `workspaceId`：可以利用排序键前缀，精确定位数据范围
- 不传 `workspaceId`：只能用 bloom filter 过滤 message_id，可能扫描更多 granule

### 11.7 改进建议

#### 建议 1：重构流程，先推断 workspaceId 再查询

**当前问题**：先查询，再从结果推断 workspaceId

**改进方案**：
```typescript
// 方案 A：如果有 immediate events，先从其中提取 workspaceId
let workspaceId: string | undefined;
for (const event of sendgridEvents) {
  if (event.workspaceId) {
    workspaceId = event.workspaceId;
    break;
  }
}

// 如果有 workspaceId，查询时带上
const processedForDelayedEvents = await findUserEventsById({
  messageIds: [...],
  workspaceId,  // 传 workspaceId（如果有）
});
```

#### 建议 2：将 workspaceId 设为必需参数

**修改**：`packages/backend-lib/src/userEvents.ts:922-928`

```typescript
export async function findUserEventsById({
  messageIds,
  workspaceId,
}: {
  messageIds: string[];
  workspaceId: string;  // 改为必需
}): Promise<UserEventsWithTraits[]>
```

**影响**：
- 编译时检查所有调用点
- SendGrid 场景需要特殊处理（先推断再查询）

#### 建议 3：签名验证前置

**当前时序**：查询 → 校验 → 签名验证

**建议时序**：
1. 先尝试从 immediate events 提取 workspaceId
2. 如果有 workspaceId，先验证签名
3. 再执行查询

```typescript
// 改进后
let workspaceId: string | undefined;
for (const event of sendgridEvents) {
  if (event.workspaceId) {
    workspaceId = event.workspaceId;
    break;
  }
}

// 如果有 workspaceId，先验证签名
if (workspaceId) {
  const verified = verifyTimestampedSignature({...});
  if (!verified) {
    return err({ message: "Invalid signature." });
  }
}

// 再执行查询
const processedForDelayedEvents = await findUserEventsById({
  messageIds: [...],
  workspaceId,
});
```

#### 建议 4：增强错误日志和监控

**问题**：当前不一致的事件被静默跳过

**改进**：
```typescript
if (workspaceId !== processedWorkspaceId) {
  logger().warn(
    {
      expectedWorkspaceId: workspaceId,
      actualWorkspaceId: processedWorkspaceId,
      messageId: event.message_id,
      smtpId,
    },
    "Workspace mismatch in delayed event backfill, skipping",
  );
  continue;
}
```

### 11.8 调用点唯一性确认

**全局搜索结果**：`findUserEventsById` 只有一个调用点

| 调用位置 | 传递 workspaceId | 说明 |
|---------|-----------------|------|
| `sendgrid.ts:267-269` | 否 | SendGrid 延迟事件回填 |

**结论**：
- 这是唯一的调用点
- 修改函数签名影响范围可控

### 11.9 同批次多 workspace 事件的隔离边界深度分析

#### 11.9.1 问题背景

SendGrid Webhook 可能在同一批次中推送多个邮件的事件。如果这些邮件来自不同的 workspace，就会触发本章节分析的隔离边界问题。

**事件类型与 workspaceId 来源**：

| 事件类型 | 示例 | workspaceId 来源 | 是否需要回填 |
|---------|------|-----------------|-------------|
| Immediate | open, click, delivered, dropped, processed | 事件本身的 `sg_message_id` + `workspaceId` 自定义参数 | 否 |
| Delayed | bounce, spamreport | 无，需通过 `smtp-id` 回填 | 是 |

#### 11.9.2 当前流程的完整时序分析

**位置**：`packages/backend-lib/src/destinations/sendgrid.ts:217-370`

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: 推断全局 workspaceId（line 234-240）                    │
├─────────────────────────────────────────────────────────────────┤
│  for (const event of sendgridEvents) {                          │
│    if (event.workspaceId) {                                     │
│      workspaceId = event.workspaceId;  // 取第一个！            │
│      break;                                                     │
│    }                                                           │
│  }                                                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: 分类事件（line 241-265）                                 │
├─────────────────────────────────────────────────────────────────┤
│  for (const event of sendgridEvents) {                          │
│    switch (event.event) {                                       │
│      case "spamreport":                                         │
│      case "bounce":                                             │
│        delayedEvents.set(smtp-id, event);  // 延迟事件           │
│        break;                                                   │
│      default:                                                   │
│        immediateEvents.push(event);  // ⚠️ 没有校验 workspaceId！│
│        break;                                                   │
│    }                                                           │
│  }                                                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: 延迟事件回填（line 267-306）                              │
├─────────────────────────────────────────────────────────────────┤
│  3a. 跨 workspace 查询 processed 事件                            │
│  3b. 解析每个事件的 properties，提取 processedWorkspaceId        │
│  3c. 校验：workspaceId === processedWorkspaceId                  │
│  3d. 不匹配则跳过（continue）                                     │
│                                                                │
│  ✅ delayed events 有校验！                                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: 合流写入（line 364-367）                                 │
├─────────────────────────────────────────────────────────────────┤
│  await submitSendgridEvents({                                   │
│    workspaceId,  // 单一值！从 Step 1 推断                       │
│    events: [...immediateEvents, ...backfilledDelayedEvents],    │
│  });                                                            │
│                                                                │
│  ⚠️ 所有事件用同一个 workspaceId 写入！                           │
└─────────────────────────────────────────────────────────────────┘
```

#### 11.9.3 写入链路确认

**submitSendgridEvents**（`packages/backend-lib/src/destinations/sendgrid.ts:191-215`）：
```typescript
export async function submitSendgridEvents({
  workspaceId,  // 参数传入
  events,
}: {
  workspaceId: string;
  events: SendgridEvent[];
}) {
  const data: BatchAppData = {
    batch: events.flatMap((e) =>
      sendgridEventToDF({ sendgridEvent: e })
        .mapErr((error) => { ... })
        .unwrapOr([]),
    ),
  };
  await submitBatch({
    workspaceId,  // 使用传入的，不使用事件内部的
    data,
  });
}
```

**submitBatch**（`packages/backend-lib/src/apps/batch.ts:89-110`）：
```typescript
export async function submitBatch({ workspaceId, data }, ...) {
  await Promise.all(
    chunks.map(async (chunk) => {
      return submitBatchChunk(
        { workspaceId, data: chunkData },  // 继续传递
        ...
      );
    }),
  );
}
```

**submitBatchChunk**（`packages/backend-lib/src/apps/batch.ts:68-87`）：
```typescript
await insertUserEvents({
  workspaceId,  // 最终写入使用这个值
  userEvents,
}, ...);
```

**结论**：写入 ClickHouse 时，`workspace_id` 列使用的是 `submitBatch` 参数传入的值，而不是事件本身的 `workspaceId`。

#### 11.9.4 校验缺口对比

| 事件类型 | Step 1 来源 | Step 2 分类校验 | Step 3 回填校验 | Step 4 写入校验 | 风险 |
|---------|------------|----------------|----------------|----------------|------|
| **Immediate** | 事件本身 `workspaceId` | ❌ 无 | N/A | ❌ 无 | **误写** |
| **Delayed** | 从 processed 推断 | N/A | ✅ 有 | ❌ 无 | 数据丢失 |

**关键发现**：
- Immediate events 在 Step 2 被加入 `immediateEvents` 数组时，**没有任何校验**
- Step 4 写入时，所有事件使用同一个 `workspaceId`
- 如果批次中有来自多个 workspace 的 immediate events，除了第一个之外的都会被误写

### 11.10 可复现的误写场景

#### 场景 D：多 workspace Immediate Events 混合同一批次

**严重级别**：🔴 高（跨 workspace 误写）

**前提条件**：
1. Workspace A（`ws-a`）和 Workspace B（`ws-b`）共享同一个 SendGrid 账号配置
2. 两个 workspace 使用同一个 Webhook 端点（URL 相同）
3. 两个 workspace 都配置了正确的 SendGrid Webhook 签名密钥

**触发条件**：
1. Workspace A 发送邮件 → 邮件被打开 → SendGrid 记录 `open` 事件
2. Workspace B 发送邮件 → 邮件被点击 → SendGrid 记录 `click` 事件
3. 由于网络延迟或批处理，两个事件在**同一批次**推送到 Webhook

**批次内容示例**：
```json
[
  {
    "email": "user-a@example.com",
    "event": "open",
    "timestamp": 1715000001,
    "sg_event_id": "sg-event-001",
    "sg_message_id": "sg-msg-a-001.filterdrecv-8-7",
    "smtp-id": "<smtp-a-001@dittofeed>",
    "workspaceId": "ws-a",
    "userId": "user-a-uuid",
    "broadcastId": "broadcast-a",
    "journeyId": "journey-a",
    "runId": "run-a"
  },
  {
    "email": "user-b@example.com",
    "event": "click",
    "timestamp": 1715000002,
    "sg_event_id": "sg-event-002",
    "sg_message_id": "sg-msg-b-001.filterdrecv-8-7",
    "smtp-id": "<smtp-b-001@dittofeed>",
    "workspaceId": "ws-b",  // ⚠️ 不同的 workspaceId
    "userId": "user-b-uuid",
    "broadcastId": "broadcast-b",
    "journeyId": "journey-b",
    "runId": "run-b"
  }
]
```

**执行流程**：

```
Step 1: 推断 workspaceId
  遍历事件：
    第 1 个事件：event.workspaceId = "ws-a"
    workspaceId = "ws-a"
    break;  // 只取第一个！
  结果：workspaceId = "ws-a"

Step 2: 分类事件
  第 1 个事件 (open):
    switch default → immediateEvents.push(event)  // ✅ 无校验
  第 2 个事件 (click):
    switch default → immediateEvents.push(event)  // ⚠️ 无校验！
  结果：immediateEvents = [event1, event2]

Step 3: 延迟事件回填
  无延迟事件 → 跳过
  结果：backfilledDelayedEvents = []

Step 4: 签名验证
  查询 ws-a 的 webhookKey
  使用 ws-a 的密钥验证签名
  假设签名有效（请求确实来自 SendGrid）
  结果：verified = true

Step 5: 合流写入
  submitSendgridEvents({
    workspaceId: "ws-a",  // ⚠️ 单一值
    events: [event1, event2],  // 两个事件
  })

  → insertUserEvents(workspaceId = "ws-a", ...)
```

**最终结果**：

| 事件 | 原始 workspaceId | 实际写入 workspace_id | 结果 |
|------|-----------------|----------------------|------|
| User A 的 open | ws-a | ws-a | ✅ 正确 |
| User B 的 click | ws-b | ws-a | ❌ **误写！** |

**后果**：
- Workspace B 的事件被错误地写入到 Workspace A
- Workspace A 的分析中会出现不属于它的用户行为数据
- Workspace B 缺失了自己的 `click` 事件
- 计费统计、用户画像、旅程分析都会受到影响

#### 场景 E：父子工作空间 Immediate Events 混合

**严重级别**：🟡 中（数据归属错误）

**前提条件**：
- Parent 工作空间 P 有子工作空间 C1、C2
- C1 和 C2 使用同一个 Webhook 端点配置

**触发条件**：
- C1 的 `delivered` 事件和 C2 的 `open` 事件同批次到达

**结果**：
- 如果 C1 的事件先被遍历，`workspaceId = ws-c1`
- C2 的事件会被写入到 ws-c1
- 父工作空间查询时，`buildWorkspaceIdClause` 会返回 `IN (ws-c1, ws-c2)`
- 所以在 Parent 层面可能看不到问题
- 但在子工作空间层面，C2 会缺失自己的事件

#### 场景 F：Immediate + Delayed 混合事件（对照）

**说明**：此场景验证 delayed events 确实有校验

**前提条件**：
- Workspace A 的 `open` 事件（immediate）
- Workspace B 的 `bounce` 事件（delayed）

**执行流程**：
```
Step 1: workspaceId = "ws-a"（从 A 的 open 事件）

Step 2: 分类
  A 的 open → immediateEvents
  B 的 bounce → delayedEvents (smtp-id = bounce-b)

Step 3: 延迟事件回填
  查询 processed:bounce-b
  假设是 B 的事件 → processedWorkspaceId = "ws-b"
  校验：ws-a !== ws-b → continue（跳过）

Step 4: 写入
  只有 A 的 open 事件被写入（workspaceId = ws-a）
```

**结果**：
- ✅ B 的 bounce 事件被正确过滤（不会被写入 A）
- ⚠️ 但 B 的 bounce 事件也没有被写入 B（丢失）
- 延迟事件有校验，不会误写，但可能丢失

### 11.11 现有校验缺口分析

#### 缺口 1：Immediate Events 无 workspaceId 校验

**位置**：`packages/backend-lib/src/destinations/sendgrid.ts:244-265`

```typescript
for (const event of sendgridEvents) {
  switch (event.event) {
    case "spamreport":
    case "bounce":
      // ... 延迟事件处理
      break;
    default:
      // ⚠️ 直接加入数组，不校验 event.workspaceId
      immediateEvents.push(event);
      break;
  }
}
```

**问题**：
- `delayedEvents` 的后续处理有 `processedWorkspaceId` 校验
- `immediateEvents` 没有任何校验
- 不一致的处理逻辑

#### 缺口 2：写入时使用推断值而非事件内部值

**位置**：`packages/backend-lib/src/destinations/sendgrid.ts:364-367`

```typescript
await submitSendgridEvents({
  workspaceId,  // 推断的单一值
  events: [...immediateEvents, ...backfilledDelayedEvents],
});
```

**问题**：
- `sendgridEventToDF` 会从事件中提取 `workspaceId` 放入 `properties`
- 但 `submitBatch` 使用的是单独传入的 `workspaceId`
- 两者可能不一致

#### 缺口 3：签名验证不按事件分组

**位置**：`packages/backend-lib/src/destinations/sendgrid.ts:319-354`

**问题**：
- 签名验证使用推断出的 `workspaceId` 对应的密钥
- 如果批次中有多个 workspace 的事件，只验证一个密钥
- 理论上，另一个 workspace 的事件应该用自己的密钥验证

**但实际上**：
- SendGrid Webhook 是按整个 HTTP 请求签名的
- 不是按事件签名
- 所以这个设计是合理的
- 但前提是：同一批次的事件应该来自同一个 workspace

### 11.12 可落地的修复方案

#### 方案 B（推荐）：校验 Immediate Events 的 workspaceId 一致性

**复杂度**：低
**安全性**：✅ 高
**数据完整性**：⚠️ 中（不一致事件会被丢弃）

**设计思路**：
- 与 `delayedEvents` 的处理方式一致
- 在分类 `immediateEvents` 时校验 `event.workspaceId`
- 不一致则跳过并记录 warning

**代码改动**（`packages/backend-lib/src/destinations/sendgrid.ts`）：

```typescript
// Step 1: 先收集所有有 workspaceId 的事件
const eventWorkspaceIds = sendgridEvents
  .map(e => e.workspaceId)
  .filter(Boolean) as string[];
const uniqueWorkspaceIds = [...new Set(eventWorkspaceIds)];

if (uniqueWorkspaceIds.length > 1) {
  logger().warn(
    {
      workspaceIds: uniqueWorkspaceIds,
      eventCount: sendgridEvents.length,
    },
    "Multiple workspaceIds found in same SendGrid webhook batch. Events with mismatched workspaceId will be skipped.",
  );
}

// Step 2: 分类时校验
for (const event of sendgridEvents) {
  switch (event.event) {
    case "spamreport":
    case "bounce":
      // ... 延迟事件处理（保持不变）
      break;
    default:
      // 新增：校验 immediate event 的 workspaceId
      if (workspaceId && event.workspaceId && event.workspaceId !== workspaceId) {
        logger().warn(
          {
            expectedWorkspaceId: workspaceId,
            actualWorkspaceId: event.workspaceId,
            eventType: event.event,
            sgMessageId: event.sg_message_id,
            email: event.email,
          },
          "Immediate event workspaceId mismatch, skipping",
        );
        continue;  // 跳过不一致的事件
      }
      immediateEvents.push(event);
      break;
  }
}
```

**优点**：
1. 与现有 `delayedEvents` 处理逻辑一致
2. 最小化代码改动（约 20 行）
3. 安全性最高——不一致的事件直接跳过
4. 有日志，便于排查问题

**缺点**：
- 如果确实有跨 workspace 批量事件，部分事件会丢失
- 但这是正确的行为——应该是配置问题（共享 Webhook），应该告警

#### 方案 A（备选）：按 workspaceId 分组独立处理

**复杂度**：高
**安全性**：✅ 高
**数据完整性**：✅ 高

**设计思路**：
1. 将事件按 `workspaceId` 分组
2. 对每组：
   - 独立查找对应的 `webhookKey`
   - 独立签名验证
   - 独立写入

**优点**：
- 即使配置了共享 Webhook，也能正确处理
- 数据不会丢失

**缺点**：
- 需要大幅重构现有逻辑
- 签名验证逻辑复杂（需要为每个组查找密钥）
- 如果有延迟事件，需要为每个组独立查询 `processed` 事件
- 改动风险较高

#### 方案 C（不推荐）：使用事件内部的 workspaceId 写入

**复杂度**：中
**安全性**：⚠️ 中
**数据完整性**：✅ 高

**设计思路**：
- 在 `submitSendgridEvents` 中，不使用统一的 `workspaceId`
- 而是逐个事件使用 `event.workspaceId`

**问题**：
- 签名验证仍然只验证一个 `workspaceId`
- 如果事件来自其他 workspace，其事件的签名合法性无法验证
- 可能被利用注入伪造事件

### 11.13 修复建议优先级

| 优先级 | 修复项 | 影响范围 | 实施难度 |
|--------|--------|---------|---------|
| P0 | 方案 B：校验 immediate events 的 workspaceId | 安全性 | 低 |
| P1 | 增强错误日志和监控 | 可观测性 | 低 |
| P2 | 签名验证前置（如果有 immediate events） | 性能/安全 | 中 |
| P3 | 方案 A：按 workspaceId 分组处理 | 完整性 | 高 |

### 11.14 延迟回填的键设计边界分析

#### 11.14.1 问题背景

SendGrid 延迟事件（bounce/spamreport）的处理依赖两个关键设计：
1. `delayedEvents` Map：以 `smtp-id` 为键存储延迟事件
2. messageId 命名约定：`processed:${smtp-id}`、`bounce:${smtp-id}`

这两个设计都缺少 workspace 维度，导致潜在的跨空间错配和事件丢失。

#### 11.14.2 messageId 命名设计对比

**位置**：`packages/backend-lib/src/destinations/sendgrid.ts:102-136`

```typescript
switch (event) {
  case "processed":
    if (!smtpId) {
      return err(new Error("Missing smtp-id for processed event."));
    }
    messageId = `processed:${smtpId}`;  // ⚠️ 没有 workspace 维度！
    break;
  case "bounce":
    if (!smtpId) {
      return err(new Error("Missing smtp-id for bounce event."));
    }
    messageId = `bounce:${smtpId}`;  // ⚠️ 没有 workspace 维度！
    break;
  case "spamreport":
    if (!smtpId) {
      return err(new Error("Missing smtp-id for spamreport event."));
    }
    messageId = `spamreport:${smtpId}`;  // ⚠️ 没有 workspace 维度！
    break;
  default: {
    if (!sendgridEvent.workspaceId || !sg_message_id) {
      return err(
        new Error(
          `Missing workspaceId or sg_message_id for event: ${event}.`,
        ),
      );
    }
    // ✅ immediate events 使用 workspaceId 作为 namespace
    messageId = uuidv5(
      `${event}:${sg_message_id}`,
      sendgridEvent.workspaceId,  // ⬅️ 关键差异！
    );
    break;
  }
}
```

**设计对比表**：

| 事件类型 | messageId 生成公式 | workspace 维度 | 唯一性保证 |
|---------|-------------------|---------------|-----------|
| open/click/delivered/dropped | `uuidv5(event:sg_message_id, workspaceId)` | ✅ namespace | 强（UUID v5） |
| processed | `processed:smtp-id` | ❌ 无 | 弱（仅 smtp-id） |
| bounce | `bounce:smtp-id` | ❌ 无 | 弱（仅 smtp-id） |
| spamreport | `spamreport:smtp-id` | ❌ 无 | 弱（仅 smtp-id） |

**关键差异**：
- **uuidv5**：两个不同 workspace 即使有相同的 `sg_message_id`，生成的 messageId 也不同（因为 namespace 不同）
- **字符串拼接**：两个不同 workspace 有相同的 `smtp-id`，就会有相同的 messageId

#### 11.14.3 delayedEvents Map 设计缺陷

**位置**：`packages/backend-lib/src/destinations/sendgrid.ts:241-259`

```typescript
const delayedEvents = new Map<string, SendgridEvent>();  // ⚠️ 单一值！

for (const event of sendgridEvents) {
  switch (event.event) {
    case "spamreport":
    case "bounce":
      if (!event["smtp-id"]) {
        // ... 日志
        continue;
      }
      delayedEvents.set(event["smtp-id"], event);  // ⚠️ 后设置的覆盖先设置的！
      break;
    // ...
  }
}
```

**问题 1：同批次同 smtp-id 多事件覆盖**

```
同批次事件：
  Event 1: bounce, smtp-id = "smtp-abc-123", bounce_type = "hard"
  Event 2: spamreport, smtp-id = "smtp-abc-123"

执行：
  delayedEvents.set("smtp-abc-123", bounceEvent);     // Map: {"smtp-abc-123" → bounce}
  delayedEvents.set("smtp-abc-123", spamreportEvent);  // Map: {"smtp-abc-123" → spamreport} 覆盖！

结果：bounceEvent 丢失 ❌
```

**问题 2：后续查询只查一次**

```typescript
// 只去重后的键
const processedForDelayedEvents = await findUserEventsById({
  messageIds: Array.from(delayedEvents.keys()).map((id) => `processed:${id}`),
});

// 遍历结果匹配
for (const event of processedForDelayedEvents) {
  const smtpId = event.message_id.split(":")[1];
  const delayedEvent = delayedEvents.get(smtpId);  // 只返回一个事件！
  // ...
}
```

即使 `processedForDelayedEvents` 返回了多个 workspace 的 `processed:smtp-id` 事件，也只能匹配到一个 delayedEvent。

### 11.15 可复现的键设计边界问题

#### 场景 G：同批次同 smtp-id 多事件被覆盖

**严重级别**：🟡 中（事件丢失）

**前提条件**：
- 同一邮件发送后，SendGrid 推送多个延迟事件
- 常见组合：`bounce + spamreport`、多次 `bounce` 通知

**触发条件**：
1. 邮件发送 → SendGrid 记录 `processed` 事件
2. 邮件被退回 → SendGrid 推送 `bounce` 事件
3. 用户举报垃圾邮件 → SendGrid 推送 `spamreport` 事件
4. 由于网络延迟，两个事件在**同一批次**到达

**批次内容示例**：
```json
[
  {
    "event": "bounce",
    "email": "hard@bounce.com",
    "timestamp": 1715000001,
    "smtp-id": "<smtp-multi-001@dittofeed>",
    "bounce_type": "hard",
    "status": "5.1.1"
  },
  {
    "event": "spamreport",
    "email": "user@spam.com",
    "timestamp": 1715000002,
    "smtp-id": "<smtp-multi-001@dittofeed>"  // ⚠️ 相同的 smtp-id！
  }
]
```

**执行流程**：

```
Step 1: 推断 workspaceId
  两个事件都没有 workspaceId（延迟事件）
  workspaceId = undefined

Step 2: 分类事件
  Event 1 (bounce, smtp-id = "<smtp-multi-001@dittofeed>"):
    delayedEvents.set("<smtp-multi-001@dittofeed>", bounceEvent)
    // Map: { "<smtp-multi-001@dittofeed>" → bounceEvent }

  Event 2 (spamreport, smtp-id = "<smtp-multi-001@dittofeed>"):
    delayedEvents.set("<smtp-multi-001@dittofeed>", spamreportEvent)
    // Map: { "<smtp-multi-001@dittofeed>" → spamreportEvent }  ⚠️ 覆盖！

  结果：delayedEvents = { "<smtp-multi-001@dittofeed>" → spamreportEvent }
        // bounceEvent 丢失！

Step 3: 延迟事件回填
  messageIds = ["processed:<smtp-multi-001@dittofeed>"]

  假设查询到 processed 事件，workspaceId = ws-a
  backfilledDelayedEvents = [spamreportEvent 回填后的数据]

Step 4: 写入
  只有 spamreport 事件被写入
  // bounce 事件永久丢失 ❌
```

**后果**：
- bounce 事件丢失，无法记录邮件失败原因
- 发送者声誉和退信率统计不准确
- 影响后续邮件发送策略（如自动停止发送到硬退信地址）

#### 场景 H：不同 workspace 相同 smtp-id 导致匹配歧义

**严重级别**：🟡 中（跨空间错配）

**前提条件**：
1. Workspace A 和 Workspace B 的邮件恰好有相同的 `smtp-id`（理论可能）
2. 两个 workspace 的 `processed` 事件 messageId 都是 `processed:smtp-123`
3. 没有 immediate events（只有延迟事件）

**触发流程**：
1. Workspace A 的 bounce 事件到达，`smtp-id = "smtp-123"`
2. `findUserEventsById({ messageIds: ["processed:smtp-123"] })`

**ClickHouse 查询**：
```sql
SELECT * FROM user_events_v2
WHERE message_id IN ('processed:smtp-123')
-- ⚠️ 没有 workspace_id 过滤！
```

**可能的查询结果**（取决于 ClickHouse 存储顺序）：

```
返回记录 1:
  workspace_id = ws-b
  message_id = processed:smtp-123
  properties = { workspaceId: "ws-b", userId: "user-b", ... }

返回记录 2:
  workspace_id = ws-a
  message_id = processed:smtp-123
  properties = { workspaceId: "ws-a", userId: "user-a", ... }
```

**后续处理**：

```
遍历 processedForDelayedEvents:

  第 1 条 (ws-b):
    processedWorkspaceId = "ws-b"
    workspaceId = undefined → 设置为 "ws-b" ✓
    backfilledDelayedEvents.push(bounce 事件 + ws-b metadata)

  第 2 条 (ws-a):
    processedWorkspaceId = "ws-a"
    workspaceId = "ws-b" → 不匹配 ❌
    continue（跳过）
```

**最终结果**：

| 实际归属 | 推断归属 | 结果 |
|---------|---------|------|
| Workspace A 的 bounce | Workspace B 处理 | ❌ 错配！ |

**后果**：
- Workspace A 的 bounce 事件被标记为 Workspace B 的
- 或者如果 B 的密钥签名验证失败，事件完全丢失
- 取决于签名验证的 workspaceId

#### 场景 I：父子工作空间 smtp-id 冲突

**严重级别**：🟡 中（数据归属问题）

**前提条件**：
- Parent P 有子工作空间 C1、C2
- C1 和 C2 的邮件恰好有相同的 smtp-id
- C1 的 processed 事件先被查询到

**结果**：
- C2 的 bounce 事件会被错误地与 C1 的 processed 事件匹配
- 或者被跳过（因为 workspaceId 校验）
- 在 Parent 层面可能看不到问题（`buildWorkspaceIdClause` 包含所有子空间）
- 但在子工作空间层面，事件归属错误

### 11.16 键设计修复方案

#### 方案 1（P0）：修复 delayedEvents Map 覆盖问题

**复杂度**：低
**影响**：事件完整性

**当前设计**：
```typescript
const delayedEvents = new Map<string, SendgridEvent>();
delayedEvents.set(event["smtp-id"], event);  // 后设置的覆盖先设置的
```

**修复方案**：
```typescript
const delayedEvents = new Map<string, SendgridEvent[]>();  // 改为数组

for (const event of sendgridEvents) {
  switch (event.event) {
    case "spamreport":
    case "bounce":
      if (!event["smtp-id"]) {
        // ...
        continue;
      }
      const smtpId = event["smtp-id"];
      const existing = delayedEvents.get(smtpId) ?? [];
      delayedEvents.set(smtpId, [...existing, event]);  // 追加而非覆盖
      break;
    // ...
  }
}

// 后续处理也需要修改
const backfilledDelayedEvents: SendgridEvent[] = [];

for (const event of processedForDelayedEvents) {
  const smtpId = event.message_id.split(":")[1];
  if (!smtpId) continue;

  const delayedEventList = delayedEvents.get(smtpId);
  if (!delayedEventList || delayedEventList.length === 0) continue;

  const parsedProperties = jsonParseSafeWithSchema(
    event.properties,
    MessageMetadataFields,
  );
  if (parsedProperties.isErr()) continue;

  const { workspaceId: processedWorkspaceId, userId } = parsedProperties.value;
  if (!processedWorkspaceId || !userId) continue;

  if (!workspaceId) {
    workspaceId = processedWorkspaceId;
  }
  if (workspaceId !== processedWorkspaceId) {
    continue;
  }

  // 为每个匹配的延迟事件回填
  for (const delayedEvent of delayedEventList) {
    backfilledDelayedEvents.push({
      ...delayedEvent,
      ...parsedProperties.value,
    });
  }

  // 处理完后可以移除，避免重复处理
  delayedEvents.delete(smtpId);
}
```

**优点**：
1. 最小化改动（约 10-15 行）
2. 解决同 smtp-id 多事件覆盖问题
3. 不影响现有逻辑的其他部分

**缺点**：
- 如果多个 workspace 有相同 smtp-id 的 processed 事件，可能重复匹配
- 需要配合方案 2 或 3 一起使用

#### 方案 2（P1）：为 processed 事件的 messageId 添加 workspace 维度

**复杂度**：中
**影响**：跨空间隔离

**当前设计**：
```typescript
case "processed":
  messageId = `processed:${smtpId}`;  // 无 workspace 维度
```

**问题**：
- `processed` 事件本身是否有 `workspaceId`？
- 从 `sendgridEventToDF` 看，`processed` 事件不要求 `workspaceId`（走 case 而非 default）
- 但在 `handleSendgridEvents` 中，`processed` 被归类为 immediate event

**修复方案**：
```typescript
case "processed":
  if (!smtpId) {
    return err(new Error("Missing smtp-id for processed event."));
  }
  // 如果有 workspaceId，使用 uuidv5 生成唯一 messageId
  if (sendgridEvent.workspaceId) {
    messageId = uuidv5(`processed:${smtpId}`, sendgridEvent.workspaceId);
  } else {
    // 保持向后兼容
    messageId = `processed:${smtpId}`;
  }
  break;
```

**问题**：
- 查询时也需要用相同的方式生成 messageId
- 需要知道 workspaceId 才能生成正确的 messageId
- 但延迟事件场景中，我们还不知道 workspaceId（这是我们要查找的）

**这是一个鸡生蛋问题**：
- 延迟事件没有 workspaceId
- 需要通过 smtp-id 查找 processed 事件来获取 workspaceId
- 如果 processed 事件的 messageId 包含 workspaceId，我们怎么生成它？

**结论**：方案 2 不适用于延迟回填场景，因为查询时还不知道 workspaceId。

#### 方案 3（P1）：在 findUserEventsById 中传递 workspaceId

**复杂度**：中
**影响**：查询隔离

**问题**：
- 当前 `findUserEventsById` 不传递 workspaceId
- 导致跨 workspace 查询

**修复方案**：

首先，尝试从 immediate events 获取 workspaceId：
```typescript
let workspaceId: string | undefined;
for (const event of sendgridEvents) {
  if (event.workspaceId) {
    workspaceId = event.workspaceId;
    break;
  }
}

// 传递 workspaceId（如果有）
const processedForDelayedEvents = await findUserEventsById({
  messageIds: Array.from(delayedEvents.keys()).map((id) => `processed:${id}`),
  workspaceId,  // 新增：传递 workspaceId
});
```

**如果没有 immediate events**（只有延迟事件）：
- 仍然无法传递 workspaceId
- 只能依赖后置校验

**优点**：
1. 有 immediate events 时，可以精准查询
2. 利用 ClickHouse 排序键（workspace_id 是首位），查询性能更好

**缺点**：
1. 只有延迟事件时，仍然无法避免跨空间查询
2. 需要配合其他方案

#### 方案 4（P2）：查询时按 workspace_id 分组处理

**复杂度**：高
**影响**：完整解决方案

**设计思路**：
1. 查询时不传递 workspaceId，允许跨空间返回
2. 遍历结果时，按 `processedWorkspaceId` 分组
3. 每组独立签名验证和写入

```typescript
// 按 workspaceId 分组
const byWorkspace = new Map<string, { processed: any[], delayed: SendgridEvent[] }>();

for (const event of processedForDelayedEvents) {
  const parsedProperties = jsonParseSafeWithSchema(...);
  if (parsedProperties.isErr()) continue;

  const { workspaceId: processedWorkspaceId, userId } = parsedProperties.value;
  if (!processedWorkspaceId || !userId) continue;

  const smtpId = event.message_id.split(":")[1];
  if (!smtpId) continue;
  const delayedEvent = delayedEvents.get(smtpId);
  if (!delayedEvent) continue;

  const group = byWorkspace.get(processedWorkspaceId) ?? { processed: [], delayed: [] };
  group.processed.push(event);
  group.delayed.push({ ...delayedEvent, ...parsedProperties.value });
  byWorkspace.set(processedWorkspaceId, group);
}

// 每组独立签名验证和写入
for (const [wsId, { delayed }] of byWorkspace) {
  // 1. 查询 wsId 的 secret
  // 2. 验证签名（如果签名是按请求签名，这里可能有问题）
  // 3. 写入事件
  await submitSendgridEvents({ workspaceId: wsId, events: delayed });
}
```

**问题**：
- SendGrid 签名是按整个 HTTP 请求签名的
- 不是按事件或按 workspace 签名的
- 如果请求中包含多个 workspace 的事件，签名验证需要特殊处理

#### 方案 5（P1）：增强 smtp-id 与 workspace 关联查询

**复杂度**：中
**影响**：查询准确性

**设计思路**：
1. 查询时获取所有匹配的 processed 事件
2. 但在后置校验时，同时校验 `smtp-id` 和 `workspaceId`
3. 如果有多个匹配，选择正确的一个

**当前问题**：
- ClickHouse 返回顺序不确定
- 第一个匹配的会被设为 `workspaceId`
- 后续匹配的会被跳过

**改进方案**：
```typescript
// 先收集所有可能的 workspaceId
const candidateWorkspaceIds = new Set<string>();
const processedBySmtpId = new Map<string, any[]>();

for (const event of processedForDelayedEvents) {
  const parsedProperties = jsonParseSafeWithSchema(...);
  if (parsedProperties.isErr()) continue;

  const { workspaceId: processedWorkspaceId } = parsedProperties.value;
  if (!processedWorkspaceId) continue;

  const smtpId = event.message_id.split(":")[1];
  if (!smtpId) continue;

  const list = processedBySmtpId.get(smtpId) ?? [];
  list.push({ event, parsedProperties });
  processedBySmtpId.set(smtpId, list);
  candidateWorkspaceIds.add(processedWorkspaceId);
}

// 如果只有一个候选 workspaceId，直接使用
if (candidateWorkspaceIds.size === 1) {
  workspaceId = [...candidateWorkspaceIds][0];
}
// 如果有多个，需要额外判断
else if (candidateWorkspaceIds.size > 1) {
  logger().warn(
    { candidateWorkspaceIds: [...candidateWorkspaceIds] },
    "Multiple workspaceIds found for delayed events",
  );
  // 可以：
  // 1. 只处理 workspaceId 明确的事件
  // 2. 或者记录告警后全部跳过
}
```

### 11.17 修复方案推荐优先级

| 优先级 | 方案 | 解决问题 | 改动量 | 风险 |
|--------|------|---------|--------|------|
| P0 | 方案 1：delayedEvents Map 支持数组 | 同批次同 smtp-id 多事件覆盖 | 低 | 低 |
| P1 | 方案 3：传递 workspaceId 到 findUserEventsById | 有 immediate events 时的精准查询 | 低 | 低 |
| P1 | 方案 5：增强候选 workspaceId 处理 | 多 workspace 匹配歧义 | 中 | 中 |
| P2 | 方案 4：按 workspace 分组独立处理 | 完整解决跨空间问题 | 高 | 中 |

**推荐实施顺序**：
1. **立即**：实施方案 1（修复事件丢失）
2. **短期**：实施方案 3（提升查询精准度和性能）
3. **中期**：实施方案 5（增强多 workspace 匹配处理）
4. **长期**：评估方案 4（完整重构）

---

## 十二、关键代码文件索引

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
| 用户查询（正确父子模式参考） | packages/backend-lib/src/users.ts |
| SendGrid 邮件发送与 Webhook 处理 | packages/backend-lib/src/destinations/sendgrid.ts |
| 常量定义（MESSAGE_METADATA_FIELDS） | packages/backend-lib/src/constants.ts |
| 类型定义 | packages/isomorphic-lib/src/types.ts |
