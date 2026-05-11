# Dittofeed 身份解析与多来源写入冲突处理分析

## 1. 概述

Dittofeed 作为客户数据平台 (CDP)，支持多客户端 SDK 上报用户行为数据。本报告深入分析以下核心机制：

1. **API Key 多入口鉴权差异**
2. **各入口鉴权与失败语义对比**
3. **匿名身份与已知用户合并逻辑**
4. **多来源写入冲突处理**
5. **端到端时序示例**

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

---

## 3. 各入口鉴权与失败语义对比

### 3.1 入口概览

Dittofeed 提供 7 个公共 API 入口 (`packages/api/src/controllers/publicAppsController.ts`)：

| 入口 | 路径 | 实现状态 | 触发机制 |
|-----|------|---------|---------|
| identify | `POST /api/public/apps/identify` | 完整实现 | 无 |
| track | `POST /api/public/apps/track` | 完整实现 | 触发 Journey |
| page | `POST /api/public/apps/page` | 完整实现 | 无 |
| screen | `POST /api/public/apps/screen` | 完整实现 | 无 |
| group | `POST /api/public/apps/group` | 完整实现 | 无 |
| alias | `POST /api/public/apps/alias` | **未实现** | 返回 400 |
| batch | `POST /api/public/apps/batch` | 完整实现 | 触发 Journey (仅 track) |

### 3.2 各入口鉴权差异分析

#### 3.2.1 Write Key 校验对比

**所有已实现入口（identify/track/page/screen/group/batch）使用相同的 `validateWriteKey` 函数**，鉴权逻辑一致。

```typescript
// 所有入口的鉴权代码模式相同
const workspaceIdFromWriteKey = await validateWriteKey({
  writeKey: request.headers.authorization,
});

if (workspaceIdFromWriteKey.isErr()) {
  return reply.status(401).send({
    message: workspaceIdFromWriteKey.error,  // identify/page/screen/group/batch
    // 或: message: "Invalid write key."    // track 单独硬编码
  });
}
```

| 入口 | validateWriteKey 调用 | 错误消息来源 |
|-----|----------------------|-------------|
| identify | ✅ 相同 | `workspaceIdFromWriteKey.error` |
| track | ✅ 相同 | 硬编码 `"Invalid write key."` |
| page | ✅ 相同 | `workspaceIdFromWriteKey.error` |
| screen | ✅ 相同 | `workspaceIdFromWriteKey.error` |
| group | ✅ 相同 | `workspaceIdFromWriteKey.error` |
| alias | ❌ 未调用 | 无（直接返回 400） |
| batch | ✅ 相同 | `workspaceIdFromWriteKey.error` |

#### 3.2.2 Workspace 可用性判定

**所有入口使用相同的 `canWorkspaceReceiveEvents` 函数**，在 `validateWriteKey` 内部调用：

```typescript
// auth.ts:108-110
if (!canWorkspaceReceiveEvents({ workspace: writeKeySecret.workspace })) {
  return err("WorkspaceIneligible");
}
```

判定规则：
- `workspace.status === WorkspaceStatusDbEnum.Active`
- `workspace.type !== "Parent"`

#### 3.2.3 失败返回语义对比

| 入口 | 401 返回内容 | 400 返回内容 | 成功返回 |
|-----|-------------|-------------|---------|
| identify | `{ message: <ValidateWriteKeyError> }` | 无 | `204 No Content` |
| track | `{ message: "Invalid write key." }` | 无 | `204 No Content` |
| page | `{ message: <ValidateWriteKeyError> }` | 无 | `204 No Content` |
| screen | `{ message: <ValidateWriteKeyError> }` | 无 | `204 No Content` |
| group | `{ message: <ValidateWriteKeyError> }` | 无 | `204 No Content` |
| alias | 无（跳过鉴权） | `{ message: "Not yet implemented." }` | 无 |
| batch | `{ message: <ValidateWriteKeyError> }` | 无 | `204 No Content` |

**关键差异**：

1. **track 入口硬编码错误消息**：
   - 其他入口返回具体错误类型（如 `"InvalidWriteKey"`、`"WorkspaceIneligible"`）
   - track 始终返回 `"Invalid write key."`，隐藏了具体失败原因

2. **alias 入口完全未实现**：
   - 跳过 Write Key 校验
   - 直接返回 `400 Bad Request`
   - 警告日志：`"Client is calling unimplemented endpoint /alias"`

### 3.2.4 ValidateWriteKey 错误类型可达性分析

`ValidateWriteKeyError` 类型在 `auth.ts:64-67` 中声明了三个变体：

```typescript
export type ValidateWriteKeyError =
  | "InvalidWriteKey"
  | "WorkspaceInactive"
  | "WorkspaceIneligible";
```

但在 `validateWriteKey` 函数的实际实现中（`auth.ts:75-116`），只有两个错误类型是可达的：

| 错误类型 | 类型声明 | 运行时可达 | 触发条件 |
|---------|---------|-----------|---------|
| `InvalidWriteKey` | ✅ 声明 | ✅ 可达 | 1. Authorization header 格式错误<br>2. secretKeyId 不是有效 UUID<br>3. secretKeyId 不存在于数据库<br>4. secretKeyValue 不匹配 |
| `WorkspaceInactive` | ✅ 声明 | ❌ **不可达** | 被 `canWorkspaceReceiveEvents` 统一返回 `WorkspaceIneligible` |
| `WorkspaceIneligible` | ✅ 声明 | ✅ 可达 | 1. workspace.status !== Active<br>2. workspace.type === "Parent" |

**WorkspaceInactive 不可达的证据**：

`validateWriteKey` 函数调用 `canWorkspaceReceiveEvents` 进行 Workspace 检查：

```typescript
// auth.ts:108-110
if (!canWorkspaceReceiveEvents({ workspace: writeKeySecret.workspace })) {
  return err("WorkspaceIneligible");  // ⭐ 只返回 WorkspaceIneligible
}
```

`canWorkspaceReceiveEvents` 的实现（`auth.ts:39-48`）：

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

**结论**：
- `workspace.status !== Active`（非活跃） → 返回 `err("WorkspaceIneligible")`
- `workspace.type === "Parent"` → 返回 `err("WorkspaceIneligible")`
- **`"WorkspaceInactive"` 错误类型从未被返回**，它是类型声明中的历史遗留

> 注意：`WorkspaceInactive` 在 `requestContext.ts` 中的用户认证场景是可达的，但在 Write Key 鉴权场景中不可达。

### 3.2.5 各入口 401/400 返回体差异详解

#### 401 鉴权失败返回体

当 `validateWriteKey` 返回错误时，各入口的 HTTP 401 返回体：

| 入口 | HTTP 状态码 | 返回体 | 原因 |
|-----|------------|-------|------|
| identify | 401 | `{ "message": "InvalidWriteKey" }` 或 `{ "message": "WorkspaceIneligible" }` | 直接返回 `workspaceIdFromWriteKey.error` |
| track | 401 | `{ "message": "Invalid write key." }` | **硬编码**，隐藏具体错误类型 |
| page | 401 | `{ "message": "InvalidWriteKey" }` 或 `{ "message": "WorkspaceIneligible" }` | 直接返回 `workspaceIdFromWriteKey.error` |
| screen | 401 | `{ "message": "InvalidWriteKey" }` 或 `{ "message": "WorkspaceIneligible" }` | 直接返回 `workspaceIdFromWriteKey.error` |
| group | 401 | `{ "message": "InvalidWriteKey" }` 或 `{ "message": "WorkspaceIneligible" }` | 直接返回 `workspaceIdFromWriteKey.error` |
| alias | 无 401 | - | **完全跳过鉴权**，直接返回 400 |
| batch | 401 | `{ "message": "InvalidWriteKey" }` 或 `{ "message": "WorkspaceIneligible" }` | 直接返回 `workspaceIdFromWriteKey.error` |

**track 入口硬编码证据**（`publicAppsController.ts:89-105`）：

```typescript
const workspaceIdFromWriteKey = await validateWriteKey({
  writeKey: request.headers.authorization,
});

if (workspaceIdFromWriteKey.isErr()) {
  return reply.status(401).send({
    message: "Invalid write key.",  // ⭐ 硬编码，不使用 error
  });
}
```

**其他入口（如 identify）**（`publicAppsController.ts:140-153`）：

```typescript
const workspaceIdFromWriteKey = await validateWriteKey({
  writeKey: request.headers.authorization,
});

if (workspaceIdFromWriteKey.isErr()) {
  return reply.status(401).send({
    message: workspaceIdFromWriteKey.error,  // ⭐ 使用具体错误类型
  });
}
```

#### 400 返回体

| 入口 | HTTP 状态码 | 返回体 | 触发原因 |
|-----|------------|-------|---------|
| identify | 无 400 | - | 只处理鉴权失败（401）和成功（204） |
| track | 无 400 | - | 只处理鉴权失败（401）和成功（204） |
| page | 无 400 | - | 只处理鉴权失败（401）和成功（204） |
| screen | 无 400 | - | 只处理鉴权失败（401）和成功（204） |
| group | 无 400 | - | 只处理鉴权失败（401）和成功（204） |
| alias | 400 | `{ "message": "Not yet implemented." }` | **端点未实现**，无前置鉴权 |
| batch | 无 400 | - | 只处理鉴权失败（401）和成功（204） |

**alias 入口未实现证据**（`publicAppsController.ts:155-167`）：

```typescript
fastify.withTypeProvider<TypeBoxTypeProvider>().post(
  "/alias",
  async (request, reply) => {
    logger().warn("Client is calling unimplemented endpoint /alias");
    return reply.status(400).send({
      message: "Not yet implemented.",  // ⭐ 直接返回，无 validateWriteKey 调用
    });
  },
);
```

**关键观察**：
- `alias` 是唯一返回 400 的公共入口
- `alias` **不调用 `validateWriteKey`**，因此即使 Write Key 无效也不会返回 401
- `track` 是唯一硬编码错误消息的入口，隐藏了 `InvalidWriteKey` vs `WorkspaceIneligible` 的差异

### 3.3 入口处理逻辑差异

#### 3.3.1 track 与 batch 的 Journey 触发

**track 入口** (`publicAppsController.ts:102-105`)：

```typescript
await submitTrackWithTriggers({
  workspaceId: workspaceIdFromWriteKey.value,
  data: request.body,
});
```

`submitTrackWithTriggers` 会：
1. 持久化文件附件（如果有）
2. 调用 `submitTrack` 写入事件
3. 提取 `userOrAnonymousId`（优先 userId，其次 anonymousId）
4. 调用 `triggerEventEntryJourneys` 触发相关 Journey

**batch 入口** (`publicAppsController.ts:282-285`)：

```typescript
await submitBatchWithTriggers({
  workspaceId: workspaceIdFromWriteKey.value,
  data: request.body,
});
```

`submitBatchWithTriggers` 会：
1. 为 batch 中所有 track 事件持久化文件附件
2. 调用 `submitBatch` 批量写入所有事件
3. 提取 batch 中所有 track 事件的 `userOrAnonymousId`
4. **仅为 track 事件**触发 `triggerEventEntryJourneys`

#### 3.3.2 group 入口的特殊处理

`submitGroup` 函数 (`apps.ts:199-208`) 不直接写入事件，而是转换为 batch：

```typescript
export async function submitGroup({
  workspaceId,
  data,
}: {
  workspaceId: string;
  data: GroupData;
}) {
  const batch = splitGroupEvents(data) satisfies BatchItem[];
  await submitBatch({ workspaceId, data: { batch, context: data.context } });
}
```

#### 3.3.3 各入口调用链

```
identify → submitIdentify → insertUserEvents
track    → submitTrackWithTriggers → submitTrack → insertUserEvents
         → triggerEventEntryJourneys
page     → submitPage → insertUserEvents
screen   → submitScreen → insertUserEvents
group    → submitGroup → splitGroupEvents → submitBatch → insertUserEvents
alias    → 未实现，返回 400
batch    → submitBatchWithTriggers → submitBatch → insertUserEvents
         → triggerEventEntryJourneys (仅 track)
```

### 3.4 数据写入模式

通过 `insertUserEvents` 函数 (`userEvents.ts:90-145`) 支持三种写入模式：

| 模式 | 实现 | 适用场景 |
|-----|------|---------|
| `kafka` | 写入 Kafka Topic `userEventsTopicName` | 高吞吐量生产环境 |
| `ch-async` | ClickHouse 异步插入 | 中等流量 |
| `ch-sync` | ClickHouse 同步插入 | 测试/低流量 |

---

## 4. 匿名身份与已知用户合并逻辑

### 4.1 事件类型支持

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

### 4.2 核心字段设计

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

#### 4.2.1 字段优先级

`user_or_anonymous_id` 采用 `coalesce` 策略：

1. **优先使用 userId**（已知用户）
2. **降级使用 anonymousId**（匿名用户）
3. **保证非空**：`assumeNotNull` 确保至少有一个 ID

#### 4.2.2 排序键设计

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

### 4.3 身份合并机制分析

#### 4.3.1 Alias 事件支持（但未实现处理）

系统在表结构层面支持 `alias` 事件类型：

```sql
event_type Enum(
  'identify' = 1,
  'track' = 2,
  'page' = 3,
  'screen' = 4,
  'group' = 5,
  'alias' = 6    ⭐ 支持身份合并事件（但 API 入口未实现）
)
```

**重要发现**：
- 表结构支持 `alias` 事件类型
- 但 **API 入口 `/alias` 返回 400 "Not yet implemented"**
- 即使写入 alias 事件，也**无 Materialized View 处理逻辑**
- 身份合并需要应用层实现

#### 4.3.2 事件存储模式

Dittofeed 采用**事件溯源 (Event Sourcing)** 模式：

1. **原始事件持久化**：所有事件存储在 `user_events_v2`
2. **材料化视图处理**：通过 Materialized View 派生计算结果
3. **查询时聚合**：使用 `argMax` 等函数在查询时计算最终状态

### 4.4 受保护的用户属性

系统定义了两个受保护属性 (`isomorphic-lib/src/protectedUserProperties.ts`)：

```typescript
const protectedUserProperties = new Set<string>(["id", "anonymousId"]);
```

这两个属性的计算逻辑如下：

#### 4.4.1 "id" 属性计算 (`computePropertiesIncremental.ts:2475-2484`)

```typescript
case UserPropertyDefinitionType.Id: {
  return [
    {
      condition: "True",
      type: "user_property",
      computedPropertyId: userProperty.id,
      argMaxValue: "user_or_anonymous_id",  // ⭐ 关键：使用 coalesce 结果
      stateId,
    },
  ];
}
```

**值来源**：`user_or_anonymous_id` = `coalesce(userId, anonymousId)`

#### 4.4.2 "anonymousId" 属性计算 (`computePropertiesIncremental.ts:2464-2474`)

```typescript
case UserPropertyDefinitionType.AnonymousId: {
  return [
    {
      condition: "True",
      type: "user_property",
      computedPropertyId: userProperty.id,
      argMaxValue: "anonymous_id",  // ⭐ 关键：使用原始 anonymousId
      stateId,
    },
  ];
}
```

**值来源**：直接使用 `anonymous_id` 字段

### 4.5 匿名与已知用户的"合并"时机与查询链路行为

#### 4.5.1 当前实现：**仅并排存储，无自动归并**

**核心结论**：

> **匿名 ID 与已知 userId 不会在服务端自动合并为同一档案。两者以独立的 `user_id` 维度存储，查询链路不会自动归并。**

#### 4.5.2 身份字段存储行为

**场景：匿名浏览 → 登录识别**

```
时间线:
T1: 匿名用户浏览页面
    { type: "page", anonymousId: "anon-123", event: "Page Viewed" }
    
    存储结果:
    user_id = "" (空字符串)
    anonymous_id = "anon-123"
    user_or_anonymous_id = "anon-123" (coalesce("", "anon-123"))

T2: 用户登录，触发 identify
    { type: "identify", userId: "user-456", anonymousId: "anon-123", traits: { email: "user@example.com" } }
    
    存储结果:
    user_id = "user-456"
    anonymous_id = "anon-123"
    user_or_anonymous_id = "user-456" (coalesce("user-456", "anon-123"))

T3: 后续行为使用已知用户 ID
    { type: "track", userId: "user-456", event: "Purchase" }
    
    存储结果:
    user_id = "user-456"
    anonymous_id = "" (空字符串)
    user_or_anonymous_id = "user-456"
```

#### 4.5.3 查询链路的分组维度

**用户属性查询按 `user_id` 分组** (`users.ts:348-349`)：

```sql
SELECT
    cp.user_id,
    cp.computed_property_id,
    cp.type,
    argMax(user_property_value, assigned_at) AS last_user_property_value,
    argMax(segment_value, assigned_at) AS last_segment_value
FROM computed_property_assignments_v2 cp
WHERE
  ${workspaceIdClause}
  AND cp.user_id IN (...)
GROUP BY cp.user_id, cp.computed_property_id, cp.type
```

**关键分析**：

| 时间点 | user_id | anonymous_id | user_or_anonymous_id | 属性归属分组 |
|-------|---------|-------------|---------------------|-------------|
| T1 | "" | "anon-123" | "anon-123" | `user_id = ""` ⚠️ |
| T2 | "user-456" | "anon-123" | "user-456" | `user_id = "user-456"` |
| T3 | "user-456" | "" | "user-456" | `user_id = "user-456"` |

**问题**：
- T1 的事件 `user_id` 为空字符串
- 属性计算按 `user_id` 分组时，T1 的属性会被分到 `user_id = ""` 这个"幽灵用户"
- T2 的 `user_id = "user-456"` 不会自动关联 T1 的属性

#### 4.5.4 "id" 属性的特殊行为

虽然 "id" 属性使用 `user_or_anonymous_id`，但用户身份的查询维度是 `user_id`，不是 `user_or_anonymous_id`：

```
T1 计算 "id" 属性:
  argMaxValue = "user_or_anonymous_id" = "anon-123"
  但存储在 computed_property_assignments_v2 中的 user_id = ""

T2 计算 "id" 属性:
  argMaxValue = "user_or_anonymous_id" = "user-456"
  存储在 computed_property_assignments_v2 中的 user_id = "user-456"

查询时:
  WHERE user_id = "user-456" → 只能看到 T2 的记录
  看不到 T1 的记录（user_id = ""）
```

#### 4.5.5 需要应用层处理的场景

**当前实现限制**：

1. **匿名阶段的属性不会自动迁移到已知用户**
2. **两个身份的事件保持独立存储**
3. **查询链路无自动归并逻辑**

**建议的应用层策略**：

```typescript
// 策略 1：使用 identify 同时传递 userId 和 anonymousId
// 让 SDK 持续传递 anonymousId，后续事件可通过 anonymousId 关联

// 策略 2：查询时同时考虑两个 ID
const userEvents = await findUserEvents({
  workspaceId,
  // 没有直接的 OR 查询，需要分别查询后合并
  // userId: "user-456",  或
  // 需要通过其他方式关联
});

// 策略 3：使用 batch 批量处理身份关联
// 在应用层收集 anonymousId → userId 映射，批量写入 identify
```

#### 4.5.6 内部事件中的身份字段

`internal_events` 表 (`clickhouse.ts:17-43`) 同时存储：

```sql
user_or_anonymous_id String,  -- 合并后的 ID（用于排序）
user_id String,               -- 原始 userId（查询分组维度）
anonymous_id String,          -- 原始 anonymousId（追溯用）
```

**用途**：
- `user_or_anonymous_id` 用于排序和分区
- `user_id` 用于属性计算的分组维度
- `anonymous_id` 保留原始信息，支持应用层关联追溯

### 4.6 群组关联 (Group 事件)

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

## 5. 多来源写入冲突处理

### 5.1 时间戳字段详解

系统使用两个关键时间戳字段：

| 字段 | 表 | 用途 | 来源 | 默认值 |
|-----|-----|------|------|--------|
| `processing_time` | `user_events_v2` | 事件接收时间 | 服务器接收时间 | `now64(3)` |
| `assigned_at` | `computed_property_assignments_v2` | 属性赋值时间 | 计算属性时的服务器时间 | `now64(3)` |

#### 5.1.1 processing_time 定义

```sql
-- userEvents/clickhouse.ts:351
processing_time DateTime64(3) DEFAULT now64(3),
```

**生成时机**：事件写入 `user_events_v2` 时自动生成

**优先级**：用于事件级去重（messageId 去重）

#### 5.1.2 assigned_at 定义

```sql
-- userEvents/clickhouse.ts:388
assigned_at DateTime64(3) DEFAULT now64(3),
```

**生成时机**：计算属性任务执行时写入 `computed_property_assignments_v2`

**优先级**：用于属性级覆盖（argMax 查询）

### 5.2 事件去重机制

#### 5.2.1 Message ID 去重适用范围

`user_events_v2` 表定义了 Bloom Filter 索引：

```sql
INDEX message_id_idx message_id TYPE bloom_filter(0.01) GRANULARITY 4
```

**去重逻辑** 仅在 `findUserEvents` 使用 `messageId` 参数过滤时生效 (`userEvents.ts:342-371`)：

```typescript
let messageIdClause = "";
let orderByClause = "";
if (messageId) {
  let messageIdWhereClause: string;
  if (typeof messageId === "string") {
    messageIdWhereClause = `AND message_id = ${qb.addQueryValue(messageId, "String")}`;
  } else {
    messageIdWhereClause = `AND message_id IN ${qb.addQueryValue(messageId, "Array(String)")}`;
  }
  // using an inner query allows us to take advantage of the skip index on
  // message_id and dedup events with the same message id
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
if (!messageIdClause.length) {
  orderByClause = "ORDER BY processing_time DESC";
}
```

#### 5.2.2 Message ID 去重的分组维度

**去重分组键**：`(workspace_id, user_or_anonymous_id, message_id)`

```sql
GROUP BY
  workspace_id,
  user_or_anonymous_id,  -- ⭐ 关键：包含用户维度
  message_id
```

**去重策略**：
- 按 `(workspace_id, user_or_anonymous_id, message_id)` 分组
- 取 `max(processing_time)` 的记录
- 使用 `argMax(event_time, processing_time)` 获取对应事件时间

#### 5.2.3 Message ID 去重适用范围总结

| 场景 | 是否去重 | 备注 |
|-----|---------|------|
| 使用 `messageId` 参数查询 | ✅ 去重 | 应用 `max(processing_time)` |
| 不使用 `messageId` 参数查询 | ❌ 不去重 | 直接返回所有记录 |
| 同一 messageId + 同一 user_or_anonymous_id | ✅ 合并为一条 | 取最新 processing_time |
| 同一 messageId + 不同 user_or_anonymous_id | ❌ 视为不同事件 | 分组维度包含 user_or_anonymous_id |
| 写入时 | ❌ 不拦截重复 | 写入无唯一约束，依赖查询时去重 |

### 5.3 用户属性冲突处理

#### 5.3.1 ReplacingMergeTree 引擎

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

#### 5.3.2 查询时 argMax 策略

在 `findAllUserPropertyAssignments` 函数 (`userProperties.ts:418-433`)：

```sql
select
  computed_property_id,
  argMax(user_property_value, assigned_at) as last_value  -- ⭐ 按 assigned_at 取最新
from computed_property_assignments_v2
where
  workspace_id = ${workspaceId}
  and user_id = ${userId}
  and type = 'user_property'
group by computed_property_id
having last_value != ''
```

**时间戳优先级**：
- 使用 `argMax(value, assigned_at)` 确保获取最新值
- `assigned_at` 字段由 `now64(3)` 自动生成，毫秒级精度

#### 5.3.3 processing_time 与 assigned_at 的优先级关系

| 阶段 | 时间戳 | 用途 | 优先级 |
|-----|-------|------|--------|
| 事件写入 | `processing_time` | 标记事件接收时间 | 事件级去重基准 |
| 事件查询去重 | `processing_time` | 同一 messageId 取最新事件 | `max(processing_time)` |
| 属性计算 | `assigned_at` | 标记属性赋值时间 | 属性级覆盖基准 |
| 属性查询 | `assigned_at` | 同一属性取最新赋值 | `argMax(..., assigned_at)` |

**关键差异**：

```
processing_time：事件接收时间
  ↓ 写入 user_events_v2
  ↓ 计算属性任务读取事件（可能延迟）
  ↓ 写入 computed_property_assignments_v2
assigned_at：属性计算时间（通常晚于 processing_time）
```

**示例**：

```
T1 (processing_time): 事件到达并写入 user_events_v2
T2 (assigned_at):    计算属性任务运行，写入属性赋值

T2 >= T1（通常 T2 > T1）
```

### 5.4 最终一致性影响

#### 5.4.1 一致性模型

Dittofeed 采用**最终一致性**模型：

1. **写入与读取分离**：写入 `user_events_v2`，读取可能来自计算后的派生表
2. **异步计算**：属性计算通过定时任务或消息触发
3. **后台合并**：ClickHouse `ReplacingMergeTree` 依赖后台 Merge

#### 5.4.2 一致性延迟点

| 延迟点 | 来源 | 影响 |
|-------|------|------|
| 事件传播 | Kafka 消费延迟 | 事件写入后不能立即可见 |
| 属性计算 | 计算任务调度间隔 | 新事件的属性更新延迟 |
| ClickHouse Merge | 后台 Merge 时机 | `ReplacingMergeTree` 可能返回重复数据 |
| 异步插入 | `ch-async` 模式 | 写入不等待确认 |

#### 5.4.3 顺序一致性配置

支持可选的顺序一致性读取：

```typescript
// userProperties.ts:437-439
clickhouse_settings: {
  select_sequential_consistency: assignmentSequentialConsistency(),
}
```

通过 `config.assignmentSequentialConsistency` 配置控制。

### 5.5 多来源写入场景处理

#### 5.5.1 同一属性多来源更新

**场景**：Web SDK 和 Mobile SDK 同时更新同一用户的 `email` 属性

```
来源 A (Web), T1:
  { userId: "user-1", traits: { email: "web@example.com" }, timestamp: "2024-01-01T10:00:00Z" }
  → user_events_v2.processing_time = PT1
  → 计算任务运行后: computed_property_assignments.assigned_at = AT1

来源 B (Mobile), T2 (T2 > T1):
  { userId: "user-1", traits: { email: "mobile@example.com" }, timestamp: "2024-01-01T10:05:00Z" }
  → user_events_v2.processing_time = PT2 (PT2 > PT1)
  → 计算任务运行后: computed_property_assignments.assigned_at = AT2 (AT2 > AT1)
```

**解决结果**：
- 查询时 `argMax(email, assigned_at)` 返回 `"mobile@example.com"`
- 以**服务器处理时间** `assigned_at` 为准，而非客户端 `timestamp`

#### 5.5.2 不同属性多来源合并

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

#### 5.5.3 匿名与已知用户属性合并

**场景**：匿名阶段收集 `device`，登录后补充 `email`

```
T1 (匿名):
  { anonymousId: "anon-123", traits: { device: "iPhone" } }
  → user_id = ""
  → user_or_anonymous_id = "anon-123"

T2 (登录):
  { userId: "user-456", anonymousId: "anon-123", traits: { email: "user@example.com" } }
  → user_id = "user-456"
  → user_or_anonymous_id = "user-456"
```

**当前实现限制**：
- 两个事件的 `user_id` 不同（"" vs "user-456"）
- 属性查询按 `user_id` 分组，T1 的 `device` 不会自动关联到 `user-456`
- **需要应用层处理身份关联**

### 5.6 批量写入处理

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

### 5.7 计算属性增量更新

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

## 6. 端到端时序示例

### 6.1 场景：匿名浏览 → 登录识别 → 跨端冲突

**参与者**：
- Web SDK：匿名浏览，然后登录
- Mobile SDK：登录后同时上报

**时间线**：

```
================================================================================
T0: 系统初始化
================================================================================
状态:
  - workspace "ws-001" 已创建并激活
  - 受保护属性 "id"、"anonymousId" 已配置
  - 自定义属性 "email"、"device"、"last_visit" 已定义

================================================================================
T1: Web SDK - 匿名用户首次访问 (10:00:00.000)
================================================================================
请求:
  POST /api/public/apps/page
  Authorization: Basic <web-write-key>
  Body:
  {
    "anonymousId": "anon-web-abc123",
    "messageId": "msg-web-001",
    "properties": { "url": "/home" },
    "timestamp": "2024-01-01T10:00:00.000Z"
  }

处理流程:
  1. validateWriteKey: 验证通过，返回 workspaceId = "ws-001"
  2. submitPage: 构造 InsertUserEvent
  3. insertUserEvents: 写入 user_events_v2

存储结果 (user_events_v2):
  workspace_id: "ws-001"
  message_id: "msg-web-001"
  event_type: "page"
  user_id: ""
  anonymous_id: "anon-web-abc123"
  user_or_anonymous_id: "anon-web-abc123"
  event_time: 2024-01-01T10:00:00.000Z
  processing_time: 2024-01-01T10:00:00.050Z  ⭐ 服务器接收时间
  message_raw: { "type": "page", "anonymousId": "anon-web-abc123", ... }

响应: 204 No Content

================================================================================
T2: Web SDK - 匿名 identify 设置 device (10:00:05.000)
================================================================================
请求:
  POST /api/public/apps/identify
  Authorization: Basic <web-write-key>
  Body:
  {
    "anonymousId": "anon-web-abc123",
    "messageId": "msg-web-002",
    "traits": { "device": "Chrome on Windows" },
    "timestamp": "2024-01-01T10:00:05.000Z"
  }

处理流程:
  1. validateWriteKey: 通过
  2. submitIdentify: 构造 InsertUserEvent
  3. insertUserEvents: 写入 user_events_v2

存储结果 (user_events_v2):
  workspace_id: "ws-001"
  message_id: "msg-web-002"
  event_type: "identify"
  user_id: ""
  anonymous_id: "anon-web-abc123"
  user_or_anonymous_id: "anon-web-abc123"
  event_time: 2024-01-01T10:00:05.000Z
  processing_time: 2024-01-01T10:00:05.060Z

响应: 204 No Content

================================================================================
T3: 属性计算任务运行 (10:00:10.000)
================================================================================
computePropertiesIncremental 执行:

处理 user_id = "" 的事件 (匿名用户):
  - "id" 属性: argMaxValue = user_or_anonymous_id = "anon-web-abc123"
  - "anonymousId" 属性: argMaxValue = anonymous_id = "anon-web-abc123"
  - "device" 属性 (Trait 类型): 从 traits 提取 = "Chrome on Windows"

存储结果 (computed_property_assignments_v2):
  1. {
       workspace_id: "ws-001",
       type: "user_property",
       computed_property_id: "<id-prop-id>",
       user_id: "",                              ⚠️ 关键：user_id 为空
       user_property_value: "anon-web-abc123",
       assigned_at: 2024-01-01T10:00:10.100Z     ⭐ 计算时间
     }
  2. {
       workspace_id: "ws-001",
       type: "user_property",
       computed_property_id: "<anonymousId-prop-id>",
       user_id: "",
       user_property_value: "anon-web-abc123",
       assigned_at: 2024-01-01T10:00:10.100Z
     }
  3. {
       workspace_id: "ws-001",
       type: "user_property",
       computed_property_id: "<device-prop-id>",
       user_id: "",
       user_property_value: "Chrome on Windows",
       assigned_at: 2024-01-01T10:00:10.100Z
     }

================================================================================
T4: Web SDK - 用户登录，触发 identify (10:00:15.000)
================================================================================
请求:
  POST /api/public/apps/identify
  Authorization: Basic <web-write-key>
  Body:
  {
    "userId": "user-999",
    "anonymousId": "anon-web-abc123",
    "messageId": "msg-web-003",
    "traits": { "email": "user@example.com" },
    "timestamp": "2024-01-01T10:00:15.000Z"
  }

处理流程:
  1. validateWriteKey: 通过
  2. submitIdentify: 构造 InsertUserEvent
  3. insertUserEvents: 写入 user_events_v2

存储结果 (user_events_v2):
  workspace_id: "ws-001"
  message_id: "msg-web-003"
  event_type: "identify"
  user_id: "user-999"                          ⭐ 现在有 userId
  anonymous_id: "anon-web-abc123"
  user_or_anonymous_id: "user-999"             ⭐ coalesce 优先 userId
  event_time: 2024-01-01T10:00:15.000Z
  processing_time: 2024-01-01T10:00:15.070Z

响应: 204 No Content

================================================================================
T5: Mobile SDK - 登录后首次上报 (10:00:20.000)
================================================================================
请求:
  POST /api/public/apps/track
  Authorization: Basic <mobile-write-key>
  Body:
  {
    "userId": "user-999",
    "anonymousId": "anon-mobile-xyz789",       ⚠️ 不同的 anonymousId
    "messageId": "msg-mobile-001",
    "event": "App Installed",
    "properties": { "platform": "iOS" },
    "timestamp": "2024-01-01T10:00:20.000Z"
  }

处理流程:
  1. validateWriteKey: 通过
  2. submitTrackWithTriggers:
     - submitTrack: 写入事件
     - triggerEventEntryJourneys: 触发相关 Journey
  3. insertUserEvents: 写入 user_events_v2

存储结果 (user_events_v2):
  workspace_id: "ws-001"
  message_id: "msg-mobile-001"
  event_type: "track"
  user_id: "user-999"
  anonymous_id: "anon-mobile-xyz789"
  user_or_anonymous_id: "user-999"
  event_time: 2024-01-01T10:00:20.000Z
  processing_time: 2024-01-01T10:00:20.080Z

响应: 204 No Content

================================================================================
T6: Web SDK 重试 T2 (网络抖动导致重复) (10:00:25.000)
================================================================================
请求:
  POST /api/public/apps/identify
  Authorization: Basic <web-write-key>
  Body:
  {
    "anonymousId": "anon-web-abc123",
    "messageId": "msg-web-002",                ⭐ 相同 messageId
    "traits": { "device": "Chrome on Windows" },
    "timestamp": "2024-01-01T10:00:05.000Z"
  }

处理流程:
  1. validateWriteKey: 通过
  2. submitIdentify: 构造 InsertUserEvent
  3. insertUserEvents: 写入 user_events_v2 (无唯一约束，重复写入)

存储结果 (user_events_v2):
  新增记录:
  workspace_id: "ws-001"
  message_id: "msg-web-002"
  event_type: "identify"
  user_id: ""
  anonymous_id: "anon-web-abc123"
  user_or_anonymous_id: "anon-web-abc123"
  event_time: 2024-01-01T10:00:05.000Z
  processing_time: 2024-01-01T10:00:25.090Z  ⭐ 新的 processing_time

响应: 204 No Content

此时 user_events_v2 中有两条 message_id = "msg-web-002" 的记录
但 processing_time 不同

================================================================================
T7: 属性计算任务再次运行 (10:00:30.000)
================================================================================
computePropertiesIncremental 执行:

处理 user_id = "user-999" 的事件:
  - "id" 属性: argMaxValue = user_or_anonymous_id = "user-999"
  - "anonymousId" 属性: argMaxValue = anonymous_id = "anon-mobile-xyz789"
  - "email" 属性: 从 traits 提取 = "user@example.com"

处理 user_id = "" 的重复事件 (messageId = "msg-web-002"):
  - "device" 属性: 已有值 "Chrome on Windows"
  - 新事件 processing_time 更晚，但内容相同

存储结果 (computed_property_assignments_v2):
  为 user_id = "user-999" 新增:
  4. {
       workspace_id: "ws-001",
       type: "user_property",
       computed_property_id: "<id-prop-id>",
       user_id: "user-999",                    ⭐ 独立的 user_id
       user_property_value: "user-999",
       assigned_at: 2024-01-01T10:00:30.200Z
     }
  5. {
       workspace_id: "ws-001",
       type: "user_property",
       computed_property_id: "<anonymousId-prop-id>",
       user_id: "user-999",
       user_property_value: "anon-mobile-xyz789",
       assigned_at: 2024-01-01T10:00:30.200Z
     }
  6. {
       workspace_id: "ws-001",
       type: "user_property",
       computed_property_id: "<email-prop-id>",
       user_id: "user-999",
       user_property_value: "user@example.com",
       assigned_at: 2024-01-01T10:00:30.200Z
     }

  为 user_id = "" 更新 (由于重复事件):
  7. {
       workspace_id: "ws-001",
       type: "user_property",
       computed_property_id: "<device-prop-id>",
       user_id: "",
       user_property_value: "Chrome on Windows",
       assigned_at: 2024-01-01T10:00:30.200Z   ⭐ 新的 assigned_at
     }

================================================================================
T8: 查询 user_id = "user-999" 的属性
================================================================================
查询:
  findAllUserPropertyAssignments({
    workspaceId: "ws-001",
    userId: "user-999"
  })

SQL 逻辑:
  SELECT
    computed_property_id,
    argMax(user_property_value, assigned_at) as last_value
  FROM computed_property_assignments_v2
  WHERE
    workspace_id = "ws-001"
    AND user_id = "user-999"           ⭐ 关键：只查询 user_id = "user-999"
    AND type = "user_property"
  GROUP BY computed_property_id

返回结果:
  {
    "id": "user-999",
    "anonymousId": "anon-mobile-xyz789",
    "email": "user@example.com"
    // ⚠️ 缺少 "device" 属性！
    // device 属性在 user_id = "" 的记录中
  }

================================================================================
T9: 使用 messageId 去重查询事件
================================================================================
查询:
  findUserEvents({
    workspaceId: "ws-001",
    messageId: ["msg-web-002"]          ⭐ 按 messageId 查询
  })

SQL 逻辑 (去重):
  SELECT ...
  FROM user_events_v2
  WHERE
    workspace_id = "ws-001"
    AND (workspace_id, processing_time, user_or_anonymous_id, event_time, message_id) IN (
      SELECT
        workspace_id,
        max(processing_time),          ⭐ 取最新 processing_time
        user_or_anonymous_id,
        argMax(event_time, processing_time),
        message_id
      FROM user_events_v2
      WHERE
        workspace_id = "ws-001"
        AND message_id IN ("msg-web-002")
      GROUP BY
        workspace_id,
        user_or_anonymous_id,
        message_id
    )

返回结果:
  - 只返回一条记录 (processing_time = 2024-01-01T10:00:25.090Z)
  - 重复记录被去重

================================================================================
T10: Mobile SDK 更新 email (与 Web 冲突) (10:00:35.000)
================================================================================
请求:
  POST /api/public/apps/identify
  Authorization: Basic <mobile-write-key>
  Body:
  {
    "userId": "user-999",
    "messageId": "msg-mobile-002",
    "traits": { "email": "updated@mobile.com" },
    "timestamp": "2024-01-01T10:00:35.000Z"
  }

存储结果 (user_events_v2):
  workspace_id: "ws-001"
  message_id: "msg-mobile-002"
  event_type: "identify"
  user_id: "user-999"
  anonymous_id: ""
  user_or_anonymous_id: "user-999"
  event_time: 2024-01-01T10:00:35.000Z
  processing_time: 2024-01-01T10:00:35.100Z

================================================================================
T11: 属性计算任务运行 (10:00:40.000)
================================================================================
computePropertiesIncremental 执行:

处理 T10 的 identify 事件:
  - "email" 属性: 新值 = "updated@mobile.com"

存储结果 (computed_property_assignments_v2):
  8. {
       workspace_id: "ws-001",
       type: "user_property",
       computed_property_id: "<email-prop-id>",
       user_id: "user-999",
       user_property_value: "updated@mobile.com",
       assigned_at: 2024-01-01T10:00:40.300Z  ⭐ 新的 assigned_at
     }

================================================================================
T12: 再次查询 user_id = "user-999" 的属性
================================================================================
返回结果:
  {
    "id": "user-999",
    "anonymousId": "anon-mobile-xyz789",
    "email": "updated@mobile.com"  ⭐ argMax 取最新 assigned_at 的值
  }

================================================================================
最终状态总结
================================================================================

user_events_v2 中的事件:
  - msg-web-001:  page,   user_id="",            anonymous_id="anon-web-abc123"
  - msg-web-002:  identify, user_id="",          anonymous_id="anon-web-abc123" (PT=10:00:05)
  - msg-web-002:  identify, user_id="",          anonymous_id="anon-web-abc123" (PT=10:00:25, 重复)
  - msg-web-003:  identify, user_id="user-999",  anonymous_id="anon-web-abc123"
  - msg-mobile-001: track,   user_id="user-999", anonymous_id="anon-mobile-xyz789"
  - msg-mobile-002: identify, user_id="user-999", anonymous_id=""

computed_property_assignments_v2 中的属性:
  user_id = "":
    - id: "anon-web-abc123"
    - anonymousId: "anon-web-abc123"
    - device: "Chrome on Windows"

  user_id = "user-999":
    - id: "user-999"
    - anonymousId: "anon-mobile-xyz789"
    - email: "updated@mobile.com"

关键发现:
  1. 匿名阶段 (user_id="") 的 device 属性 不会 自动关联到 user-999
  2. 两个身份保持独立存储
  3. email 属性冲突时，以最后一次计算的 assigned_at 为准
  4. messageId 去重仅在查询时生效，写入不阻止重复
```

### 6.2 时序示例关键结论

1. **身份独立存储**：
   - 匿名事件的 `user_id = ""`，已知用户事件的 `user_id = "user-999"`
   - 两者在 `computed_property_assignments_v2` 中完全独立

2. **属性不自动迁移**：
   - 匿名阶段的 `device` 属性存储在 `user_id = ""`
   - 查询 `user_id = "user-999"` 时看不到 `device`

3. **冲突按 assigned_at 解决**：
   - `email` 属性由 Mobile SDK 更新后，`assigned_at` 较晚
   - `argMax` 查询返回最新值

4. **messageId 去重仅限查询**：
   - 重复写入不会被拦截
   - 只有使用 `messageId` 参数查询时才会去重
   - 去重按 `(workspace_id, user_or_anonymous_id, message_id)` 分组

5. **processing_time vs assigned_at**：
   - `processing_time`：事件接收时间，用于事件级去重
   - `assigned_at`：属性计算时间，用于属性级覆盖
   - `assigned_at` 通常晚于 `processing_time`

---

## 7. 架构总结

### 7.1 数据流

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
┌─────────────────────────────────────────┐
│  事件接收层                              │
│  ├── identify → submitIdentify          │
│  ├── track → submitTrackWithTriggers    │ ← 触发 Journey
│  ├── page → submitPage                  │
│  ├── screen → submitScreen              │
│  ├── group → submitGroup → submitBatch  │
│  ├── alias → 400 Not Implemented        │
│  └── batch → submitBatchWithTriggers    │ ← 触发 Journey (仅 track)
└────────────────────┬────────────────────┘
                     │
                     ▼
┌──────────────────────────────────┐
│  写入层                           │
│  ┌─────────┐ ┌────────────────┐ │
│  │  Kafka  │ │  ClickHouse    │ │ ← insertUserEvents
│  │  Topic  │ │ Direct Insert  │ │
│  └────┬────┘ └──────┬─────────┘ │
└───────┼─────────────┼───────────┘
        │             │
        ▼             ▼
┌──────────────────────────────────┐
│  user_events_v2                  │
│  processing_time = now64(3)      │ ← 事件接收时间
│  MergeTree, 所有事件持久化        │
└──────────────┬───────────────────┘
               │
               ▼ (计算属性任务)
┌──────────────────────────────────┐
│  computed_property_assignments   │
│  assigned_at = now64(3)          │ ← 属性计算时间
│  ReplacingMergeTree              │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  查询层                           │
│  ├── argMax(..., assigned_at)    │ ← 属性级覆盖
│  ├── max(processing_time)        │ ← 事件级去重 (messageId)
│  └── GROUP BY user_id            │ ← 身份维度独立
└──────────────────────────────────┘
```

### 7.2 关键设计决策

| 决策点 | 选择 | 理由 |
|-------|------|-----|
| 存储引擎 | MergeTree + ReplacingMergeTree | 高吞吐写入 + 最终一致性 |
| 冲突解决 | 时间戳优先级 (argMax) | 简单可靠，适合异步场景 |
| 身份关联 | 双 ID 独立存储 | 灵活但需要应用层处理合并 |
| 事件去重 | message_id + 查询时聚合 | 避免唯一键开销，支持高吞吐 |
| 多入口 | 多 Write Key + Workspace 隔离 | 安全性和可管理性 |
| 入口差异 | 统一鉴权 + 差异化触发 | 代码复用 + 业务逻辑分离 |

### 7.3 当前实现的限制

1. **Alias 事件未实现**：API 入口返回 400，表结构支持但无处理逻辑
2. **匿名与已知用户不自动合并**：`user_id` 不同，属性独立存储
3. **messageId 去重仅限查询**：写入不阻止重复，依赖查询时去重
4. **最终一致性**：依赖后台 Merge 和定时计算任务
5. **track 入口错误消息不具体**：硬编码 `"Invalid write key."`

### 7.4 建议的改进方向

1. **实现 alias 入口**：提供身份合并的标准 API
2. **身份关联 Materialized View**：自动关联 anonymousId → userId
3. **属性迁移机制**：匿名属性自动迁移到已知用户
4. **写入时去重选项**：可选的唯一约束或去重逻辑
5. **统一错误消息**：所有入口使用一致的错误返回格式

---

## 8. 附录：核心文件索引

| 功能 | 文件路径 | 关键函数/定义 |
|-----|---------|-------------|
| API 鉴权 | `packages/backend-lib/src/auth.ts` | `validateWriteKey`, `getOrCreateWriteKey` |
| 公共入口 | `packages/api/src/controllers/publicAppsController.ts` | 所有 7 个入口实现 |
| 事件写入 | `packages/backend-lib/src/userEvents.ts` | `insertUserEvents`, `findUserEvents` |
| 入口处理 | `packages/backend-lib/src/apps.ts` | `submitIdentify`, `submitTrackWithTriggers` 等 |
| 表结构定义 | `packages/backend-lib/src/userEvents/clickhouse.ts` | `createUserEventsTables` |
| 批量处理 | `packages/backend-lib/src/apps/batch.ts` | `submitBatch`, `buildBatchUserEvents` |
| 用户属性 | `packages/backend-lib/src/userProperties.ts` | `findAllUserPropertyAssignments` |
| 计算属性 | `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` | 增量计算逻辑 |
| 事件类型 | `packages/isomorphic-lib/src/types.ts` | `EventType`, `UserPropertyDefinitionType` |
| 受保护属性 | `packages/isomorphic-lib/src/protectedUserProperties.ts` | `["id", "anonymousId"]` |
