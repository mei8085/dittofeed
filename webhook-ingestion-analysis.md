# Dittofeed Webhook 事件摄入链路分析报告

## 目录

1. [概述](#概述)
2. [Webhook 接收与验证链路](#webhook-接收与验证链路)
3. [事件入队机制](#事件入队机制)
4. [事件去重与幂等处理](#事件去重与幂等处理)
5. [事件消费与 Journey 触发](#事件消费与-journey-触发)
6. [完整链路图](#完整链路图)
7. [关键文件索引](#关键文件索引)

---

## 概述

Dittofeed 支持多种外部系统通过 Webhook 推送事件，包括 Sendgrid、Amazon SES、Resend、Postmark、Mailchimp、Twilio 和 Segment 等。事件从接收、验证、入队到最终触发 Journey 的完整链路涉及多个组件和层次的处理。

---

## Webhook 接收与验证链路

### 1.1 路由与控制器

Webhook 端点在 `router.ts` 中注册，暴露在两个路径下：
- `/api/webhooks/` - 标准授权路径
- `/api/public/webhooks/` - 无需标准授权的公共路径

**代码位置**: `packages/api/src/buildApp/router.ts:88,104`

### 1.2 支持的 Webhook 类型

控制器 `webhooksController.ts` 定义了以下 Webhook 端点：

| Webhook 类型 | 路径 | 验证方式 |
|-------------|------|---------|
| Sendgrid | `/sendgrid` | HMAC 签名验证 |
| Amazon SES | `/amazon-ses` | SNS 签名验证 |
| Resend | `/resend` | Svix 签名验证 |
| Postmark | `/postmark` | 密钥头匹配 |
| Mailchimp | `/mailchimp` | HMAC-SHA1 签名验证 |
| Twilio | `/twilio` | Twilio 官方验证库 |
| Segment | `/segment` | 自定义 HMAC 摘要验证 |

**代码位置**: `packages/api/src/controllers/webhooksController.ts:76-694`

### 1.3 验证流程详解

#### 1.3.1 通用验证步骤

所有 Webhook 都遵循类似的验证流程：

1. **提取 Workspace ID**: 从请求体、查询参数或元数据中提取
2. **获取 Secret**: 从数据库查询对应 Webhook 的配置密钥
3. **验证签名/密钥**: 根据不同提供商的机制进行验证
4. **Workspace 资格检查**: 确保 Workspace 处于激活状态 (`canWorkspaceReceiveEvents`)

#### 1.3.2 Resend Webhook 验证示例

```typescript
// 1. 从 body 提取 workspaceId
const { workspaceId } = request.body.data.tags;

// 2. 查询数据库获取 Resend 配置密钥
const secret = await db().query.secret.findFirst({
  where: and(
    eq(schema.secret.workspaceId, workspaceId),
    eq(schema.secret.name, SecretNames.Resend),
  ),
});

// 3. 使用 Svix 库验证签名
const wh = new Webhook(webhookKey);
const verified = wh.verify(request.rawBody, request.headers);

// 4. 检查 Workspace 资格
if (!secret?.workspace || 
    !canWorkspaceReceiveEvents({ workspace: secret.workspace })) {
  return reply.status(401).send({ message: "Workspace not eligible." });
}

// 5. 提交事件处理
await submitResendEvents({ workspaceId, events: [request.body] });
```

**代码位置**: `packages/api/src/controllers/webhooksController.ts:183-282`

---

## 事件入队机制

### 2.1 事件转换与标准化

不同 Webhook 的事件格式需要转换为 Dittofeed 内部标准格式。以 Resend 为例：

**代码位置**: `packages/backend-lib/src/destinations/resend.ts:66-161`

```typescript
// Resend 事件转换为内部 Batch 格式
export function resendEventToDF({
  workspaceId,
  resendEvent,
}: {
  workspaceId: string;
  resendEvent: ResendEvent;
}): Result<BatchItem, Error> {
  // 1. 提取事件类型和相关数据
  const { type: event } = resendEvent;
  const { created_at, email_id, to } = resendEvent.data;
  
  // 2. 生成唯一的 messageId
  const messageId = uuidv5(`${event}:${email_id}`, workspaceId);
  
  // 3. 映射到内部事件类型
  switch (event) {
    case ResendEventType.Opened:
      eventName = InternalEventType.EmailOpened;
      break;
    case ResendEventType.Clicked:
      eventName = InternalEventType.EmailClicked;
      break;
    // ... 其他事件类型
  }
  
  // 4. 构造标准 Track 事件
  return ok({
    type: EventType.Track,
    event: eventName,
    userId,
    messageId,
    timestamp,
    properties,
  });
}
```

### 2.2 批量提交与入队

事件转换后通过 `submitBatch` 函数进行批量处理：

**代码位置**: `packages/backend-lib/src/apps/batch.ts:89-111`

```typescript
export async function submitBatch(
  { workspaceId, data }: SubmitBatchOptions,
  { processingTime, writeModeOverride } = {},
) {
  // 1. 分批处理（默认由 config().batchChunkSize 控制）
  const chunks = R.chunk(data.batch, batchChunkSize);
  
  // 2. 并行处理每个批次
  await Promise.all(
    chunks.map(async (chunk) => {
      const chunkData = { ...data, batch: chunk };
      return submitBatchChunk(
        { workspaceId, data: chunkData },
        { processingTime, writeModeOverride },
      );
    }),
  );
}
```

### 2.3 写入模式：Kafka vs ClickHouse 直写

`insertUserEvents` 函数支持三种写入模式：

**代码位置**: `packages/backend-lib/src/userEvents.ts:90-145`

```typescript
export async function insertUserEvents(
  { workspaceId, userEvents, events }: InsertUserEventsParams,
  options?: { writeModeOverride?: WriteMode },
): Promise<void> {
  const effectiveWriteMode = options?.writeModeOverride ?? config().writeMode;
  
  switch (effectiveWriteMode) {
    case "kafka":
      // 写入 Kafka 主题
      await (await kafkaProducer()).send({
        topic: userEventsTopicName,
        messages: userEventsWithDefault.map(({ messageRaw, messageId, ... }) => ({
          key: messageId,  // 使用 messageId 作为 Kafka key
          value: JSON.stringify({
            processing_time: processingTime,
            workspace_id: workspaceId,
            message_id: messageId,
            message_raw: messageRaw,
          }),
        })),
      });
      break;
      
    case "ch-async":
      // 异步写入 ClickHouse
      await insertUserEventsDirect({ 
        workspaceId, 
        userEvents: userEventsWithDefault,
        asyncInsert: true,
      });
      break;
      
    case "ch-sync":
      // 同步写入 ClickHouse
      await insertUserEventsDirect({
        workspaceId,
        userEvents: userEventsWithDefault,
      });
      break;
  }
}
```

### 2.4 Kafka 表与物化视图

当使用 Kafka 模式时，ClickHouse 通过 Kafka 引擎表消费数据：

**代码位置**: `packages/backend-lib/src/userEvents/clickhouse.ts:602-657`

```sql
-- Kafka 队列表
CREATE TABLE IF NOT EXISTS user_events_queue_v2 (
  message_raw String, 
  workspace_id String, 
  message_id String
) ENGINE = Kafka(...) 
SETTINGS kafka_thread_per_consumer = 0, kafka_num_consumers = 1;

-- 物化视图将数据从 Kafka 队列转到主表
CREATE MATERIALIZED VIEW IF NOT EXISTS user_events_mv_v2
TO user_events_v2 AS
SELECT * FROM user_events_queue_v2;
```

---

## 事件去重与幂等处理

### 3.1 多层次去重策略

Dittofeed 实现了多层次的去重机制：

| 层级 | 位置 | 机制 |
|------|------|------|
| L1 | Kafka Producer | messageId 作为消息 key |
| L2 | ClickHouse 表 | MergeTree 排序键 + 查询时去重 |
| L3 | Journey 工作流 | `keyedEventIds` Set 跟踪已处理事件 |
| L4 | Postgres 记录 | `userJourneyEvent` 唯一约束 |
| L5 | Temporal 工作流 | 工作流 ID 唯一性 + `WorkflowExecutionAlreadyStartedError` |

### 3.2 L1: Kafka 消息层面

**代码位置**: `packages/backend-lib/src/userEvents.ts:110-127`

使用 `messageId` 作为 Kafka 消息的 key：
```typescript
messages: userEventsWithDefault.map(({ messageRaw, messageId, ... }) => ({
  key: messageId,  // 相同 messageId 的消息会被路由到同一分区
  value: JSON.stringify({ ..., message_id: messageId, ... }),
})),
```

### 3.3 L2: ClickHouse 查询去重

在查询事件时，使用子查询和 `argMax` 进行去重：

**代码位置**: `packages/backend-lib/src/userEvents.ts:343-371`

```typescript
messageIdClause = `
  AND (workspace_id, processing_time, user_or_anonymous_id, event_time, message_id) IN (
    SELECT
      workspace_id,
      max(processing_time),      -- 取最新的处理时间
      user_or_anonymous_id,
      argMax(event_time, processing_time),  -- 取对应最新的事件时间
      message_id
    FROM user_events_v2
    WHERE
      ${workspaceIdClause}
      ${messageIdWhereClause}
    GROUP BY
      workspace_id,
      user_or_anonymous_id,
      message_id  -- 按 messageId 分组去重
  )
`;
```

### 3.4 L3: Journey 工作流内去重

**代码位置**: `packages/backend-lib/src/journeys/userWorkflow.ts:444-527`

```typescript
const keyedEventIds = new Set<string>();

wf.setHandler(trackSignal, async (event) => {
  // 检查是否已处理过该事件
  if (keyedEventIds.has(event.messageId)) {
    logger.info("ignoring duplicate keyed event", {
      journeyId, userId, workspaceId, messageId: event.messageId,
    });
    return;  // 直接忽略重复事件
  }
  
  // 处理事件...
  
  // 记录已处理
  keyedEventIds.add(event.messageId);
});
```

### 3.5 L4: Postgres 唯一约束

`UserJourneyEvent` 表定义了组合唯一索引：

**代码位置**: `packages/backend-lib/src/db/schema.ts:261-288`

```typescript
export const userJourneyEvent = pgTable(
  "UserJourneyEvent",
  {
    id: uuid().primaryKey(),
    userId: text().notNull(),
    journeyId: uuid(),
    type: text().notNull(),
    journeyStartedAt: timestamp().notNull(),
    nodeId: text(),
    eventKey: text(),
    eventKeyName: text(),
  },
  (table) => [
    // 组合唯一约束：journeyId + userId + eventKey + eventKeyName + type + journeyStartedAt + nodeId
    uniqueIndex(
      "UserJourneyEvent_journeyId_userId_eventKey_eventKeyName_typ_key"
    ).using(
      "btree",
      table.journeyId,
      table.userId,
      table.eventKey,
      table.eventKeyName,
      table.type,
      table.journeyStartedAt,
      table.nodeId,
    ),
  ],
);
```

### 3.6 L5: Temporal 工作流幂等

**代码位置**: `packages/backend-lib/src/journeys/userWorkflow/lifecycle.ts:54-109`

```typescript
export async function startKeyedUserJourney({ ... }) {
  // 生成唯一的工作流 ID
  const workflowId = getKeyedUserJourneyWorkflowId({
    workspaceId,
    userId,
    journeyId,
    event,
    entryNode: definition.entryNode,
  });
  
  try {
    // 使用 signalWithStart 原子性操作
    await workflowClient.signalWithStart(userJourneyWorkflow, {
      taskQueue: "default",
      workflowId,  // 工作流 ID 唯一性保证
      signal: trackSignal,
      signalArgs: [{ version: TrackSignalParamsVersion.V2, messageId: event.messageId }],
      args: [{ ..., version: UserJourneyWorkflowVersion.V3, ... }],
    });
  } catch (e) {
    // 如果工作流已存在，忽略重复启动
    if (e instanceof WorkflowExecutionAlreadyStartedError) {
      logger().info("User journey already started.", { workflowId, ... });
      return;
    }
    throw e;
  }
}
```

### 3.7 工作流 ID 生成策略

**代码位置**: `packages/backend-lib/src/journeys/userWorkflow.ts:86-144`

```typescript
// 对于 Keyed Journey，使用自定义 key 生成工作流 ID
export function getKeyedUserJourneyWorkflowIdInner({
  workspaceId,
  userId,
  journeyId,
  eventKey,
  eventKeyValue,
}: { ... }): string | null {
  const combined = uuidV5(
    [userId, eventKey, eventKeyValue].join("-"),
    workspaceId,  // 使用 workspaceId 作为 UUID v5 的命名空间
  );
  return `user-journey-keyed-${workspaceId}-${journeyId}-${combined}`;
}

// 对于非 Keyed Journey
export function getUserJourneyWorkflowId({
  userId,
  journeyId,
}: { ... }): string {
  return `user-journey-${userId}-${journeyId}`;
}
```

### 3.8 可运行性检查 (`isRunnable`)

**代码位置**: `packages/backend-lib/src/journeys/userWorkflow/activities.ts:396-470`

```typescript
export async function isRunnable({
  workspaceId,
  journeyId,
  userId,
  eventKey,
  eventKeyName,
}: { ... }): Promise<boolean> {
  // 1. 查找之前的退出事件
  const [previousExitEvent, journey, workspace] = await Promise.all([
    db().query.userJourneyEvent.findFirst({
      where: and(
        eq(dbUserJourneyEvent.journeyId, journeyId),
        eq(dbUserJourneyEvent.userId, userId),
        eventKey ? eq(dbUserJourneyEvent.eventKey, eventKey) : undefined,
        eventKeyName ? eq(dbUserJourneyEvent.eventKeyName, eventKeyName) : undefined,
        inArray(dbUserJourneyEvent.type, Array.from(ENTRY_TYPES)),
      ),
    }),
    // ...
  ]);
  
  // 2. 如果没有之前的事件，可以运行
  if (!previousExitEvent) {
    return true;
  }
  
  // 3. 检查 Journey 是否允许多次运行
  const canRunMultiple = !!journey?.canRunMultiple;
  
  // 4. 检查 Workspace 是否激活
  if (workspace?.status !== "Active") {
    return false;
  }
  
  return canRunMultiple;
}
```

---

## 事件消费与 Journey 触发

### 4.1 事件触发入口

**代码位置**: `packages/backend-lib/src/apps.ts:48-90,92-145`

```typescript
// 单个 Track 事件触发
export async function submitTrackWithTriggers({
  workspaceId,
  data,
}: { ... }) {
  // 1. 先存储事件
  await submitTrack({ workspaceId, data: { ...data, properties } });
  
  // 2. 获取用户 ID
  let userOrAnonymousId: string | null = null;
  if ("userId" in data) {
    userOrAnonymousId = data.userId;
  } else if ("anonymousId" in data) {
    userOrAnonymousId = data.anonymousId;
  }
  
  // 3. 触发 Journey
  if (userOrAnonymousId) {
    await triggerEventEntryJourneys({
      workspaceId,
      event: { ...data, properties },
      userId: userOrAnonymousId,
    });
  }
}

// 批量事件触发
export async function submitBatchWithTriggers({ ... }) {
  // 1. 存储所有事件
  await submitBatch({ workspaceId, data });
  
  // 2. 对每个 Track 事件触发 Journey
  const triggers: TriggerEventEntryJourneysOptions[] = data.batch.flatMap(
    (message) => {
      if (message.type !== EventType.Track) return [];
      // ... 提取 userId
      return { workspaceId, event: message, userId: userOrAnonymousId };
    },
  );
  
  // 3. 并行触发所有匹配的 Journey
  await Promise.all(
    triggers.map((trigger) => triggerEventEntryJourneys(trigger)),
  );
}
```

### 4.2 Journey 匹配逻辑

**代码位置**: `packages/backend-lib/src/journeys.ts:688-812`

```typescript
const EVENT_TRIGGER_JOURNEY_CACHE = new NodeCache({
  stdTTL: 30,    // 30 秒缓存
  checkperiod: 120,
});

export async function triggerEventEntryJourneys({
  workspaceId,
  event: triggerEvent,
  userId,
}: TriggerEventEntryJourneysOptions): Promise<void> {
  // 1. 从缓存获取或查询该 Workspace 的所有 Running 状态的 Journey
  let journeyDetails: EventTriggerJourneyDetails[] | undefined =
    journeyCache.get(workspaceId);
  
  if (!journeyDetails) {
    const allJourneys = await db().query.journey.findMany({
      where: eq(dbJourney.workspaceId, workspaceId),
    });
    
    // 过滤出 Running 状态且入口是 EventEntryNode 的 Journey
    journeyDetails = allJourneys.flatMap((j) => {
      // ...
      if (
        journey.status !== JourneyResourceStatusEnum.Running ||
        journey.definition.entryNode.type !== JourneyNodeType.EventEntryNode
      ) {
        return [];
      }
      return [{
        event: journey.definition.entryNode.event,  // Journey 监听的事件名
        journeyId: journey.id,
        definition: journey.definition,
        journeyName: journey.name,
      }];
    });
    journeyCache.set(workspaceId, journeyDetails);
  }
  
  // 2. 匹配事件并触发 Journey
  const starts: Promise<unknown>[] = journeyDetails.flatMap(
    ({ journeyId, journeyName, event: journeyEvent, definition }) => {
      // 使用模式匹配检查事件名
      const isMatch = doesEventNameMatch({
        pattern: journeyEvent,  // Journey 配置的事件模式（支持通配符 *）
        event: triggerEvent.event,  // 实际收到的事件名
      });
      
      if (!isMatch) return [];
      
      // 触发 Journey
      return startKeyedJourneyImpl({
        workspaceId,
        userId,
        journeyId,
        event: triggerEvent,
        definition,
      });
    },
  );
  
  await Promise.all(starts);
}
```

### 4.3 事件名匹配算法

**代码位置**: `packages/isomorphic-lib/src/events.ts:1-12`

```typescript
export function doesEventNameMatch({
  pattern,
  event,
}: {
  pattern: string;  // Journey 配置的事件模式
  event: string;    // 实际事件名
}): boolean {
  // 支持通配符后缀匹配
  if (pattern.endsWith("*")) {
    return event.startsWith(pattern.slice(0, -1));
  }
  // 精确匹配
  return pattern === event;
}
```

### 4.4 节点处理记录

**代码位置**: `packages/backend-lib/src/journeys/recordNodeProcessed.ts:11-70`

```typescript
export async function recordNodeProcessed({
  journeyStartedAt,
  userId,
  node,
  journeyId,
  workspaceId,
  eventKey,
  eventKeyName,
}: RecordNodeProcessedParams) {
  const nodeId = getNodeId(node);
  
  const trackedFields = {
    journeyStartedAt: new Date(journeyStartedAt),
    journeyId,
    type: node.type,
    nodeId,
    eventKey,
    eventKeyName,
  };
  
  // 1. 插入 Postgres（利用唯一约束实现幂等）
  await db()
    .insert(dbUserJourneyEvent)
    .values({
      ...trackedFields,
      userId,
      id: randomUUID(),
    })
    .onConflictDoNothing();  // 冲突时忽略
  
  // 2. 同时写入 ClickHouse 用于分析
  await submitTrack({
    workspaceId,
    data: {
      userId,
      event: InternalEventType.JourneyNodeProcessed,
      messageId: uuidv5(messageIdName, workspaceId),
      properties: trackedFields,
    },
  });
}
```

---

## 完整链路图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         外部系统 (Sendgrid, Resend, 等)                       │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 │ POST /api/public/webhooks/{provider}
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    webhooksController.ts (API Layer)                         │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ 1. 解析请求体                                                        │     │
│  │ 2. 提取 workspaceId (从 metadata/tags/query)                        │     │
│  │ 3. 查询数据库获取 Webhook Secret                                     │     │
│  │ 4. 验证签名/密钥 (各提供商不同机制)                                    │     │
│  │ 5. 检查 Workspace 激活状态 (canWorkspaceReceiveEvents)                │     │
│  │ 6. 调用 submit{Provider}Events                                       │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 │ submit{Provider}Events
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                  destinations/{provider}.ts (转换层)                          │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ 1. 提取事件元数据 (email_id, userId 等)                               │     │
│  │ 2. 生成唯一 messageId (uuidv5: event:email_id + workspaceId)         │     │
│  │ 3. 映射到内部事件类型 (InternalEventType.EmailOpened 等)               │     │
│  │ 4. 构造 BatchItem (EventType.Track)                                  │     │
│  │ 5. 调用 submitBatch                                                  │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 │ submitBatch / submitBatchWithTriggers
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     apps/batch.ts + apps.ts (入队层)                          │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ 1. 分批次处理 (R.chunk)                                              │     │
│  │ 2. 构造 InsertUserEvent (添加 context, processingTime, serverTime)   │     │
│  │ 3. 调用 insertUserEvents (3 种模式: kafka / ch-async / ch-sync)      │     │
│  │                                                                    │     │
│  │ submitBatchWithTriggers 额外步骤:                                    │     │
│  │ 4. 过滤出 EventType.Track 事件                                       │     │
│  │ 5. 提取 userId / anonymousId                                        │     │
│  │ 6. 调用 triggerEventEntryJourneys (触发 Journey)                     │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
                    ▼            ▼            ▼
         ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
         │   Kafka 模式  │ │ ch-async 模式 │ │ ch-sync 模式  │
         └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
                │               │               │
                ▼               ▼               │
         ┌──────────────┐        │               │
         │ Kafka Topic  │        │               │
         │ user_events  │        │               │
         └──────┬───────┘        │               │
                │                │               │
                ▼                │               │
         ┌──────────────┐        │               │
         │ ClickHouse   │        │               │
         │ Kafka Engine │        │               │
         │ 表 + 物化视图  │◄───────┘               │
         └──────┬───────┘                        │
                │                                │
                ▼                                │
         ┌────────────────────────────────────┐  │
         │ ClickHouse user_events_v2 表        │◄─┘
         │ (MergeTree, 排序键含 message_id)   │
         │ INDEX message_id_idx TYPE bloom    │
         └──────────────┬─────────────────────┘
                        │
                        │ triggerEventEntryJourneys
                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     journeys.ts (Journey 匹配层)                              │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ 1. 从缓存/数据库获取该 Workspace 所有 Running 状态的 Journey          │     │
│  │ 2. 过滤入口类型为 EventEntryNode 的 Journey                          │     │
│  │ 3. 事件名匹配 (doesEventNameMatch: 精确匹配或通配符*)                 │     │
│  │ 4. 对每个匹配的 Journey 调用 startKeyedUserJourney                   │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 │ startKeyedUserJourney
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│            journeys/userWorkflow/lifecycle.ts (工作流启动层)                   │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ 1. 生成唯一 Workflow ID                                             │     │
│  │    - Keyed: uuidv5(userId+eventKey+keyValue, workspaceId)           │     │
│  │    - Non-Keyed: user-journey-{userId}-{journeyId}                   │     │
│  │ 2. 调用 Temporal signalWithStart (原子操作)                          │     │
│  │ 3. 捕获 WorkflowExecutionAlreadyStartedError 实现幂等                │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 │ Temporal Workflow Execution
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│             journeys/userWorkflow.ts (工作流执行层)                           │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ 1. isRunnable 检查 (是否已运行过 + canRunMultiple 配置)              │     │
│  │ 2. keyedEventIds Set 跟踪已处理事件 (工作流内去重)                   │     │
│  │ 3. 遍历节点执行 (EntryNode → DelayNode → MessageNode → ...)         │     │
│  │ 4. recordNodeProcessed (Postgres 唯一约束 + ClickHouse 记录)        │     │
│  │ 5. shouldReEnter 检查是否需要重新进入                               │     │
│  │ 6. trackSignal 处理器处理后续事件                                   │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 关键文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `packages/api/src/controllers/webhooksController.ts` | Webhook 接收与验证控制器 |
| `packages/api/src/buildApp/router.ts` | API 路由注册 |
| `packages/backend-lib/src/destinations/resend.ts` | Resend 事件转换示例 |
| `packages/backend-lib/src/apps/batch.ts` | 批量事件处理 |
| `packages/backend-lib/src/apps.ts` | 事件提交与 Journey 触发入口 |
| `packages/backend-lib/src/userEvents.ts` | 事件写入 (Kafka/ClickHouse) |
| `packages/backend-lib/src/kafka.ts` | Kafka 生产者/消费者配置 |
| `packages/backend-lib/src/userEvents/clickhouse.ts` | ClickHouse 表结构与物化视图 |
| `packages/backend-lib/src/journeys.ts` | Journey 定义、统计、事件触发匹配 |
| `packages/backend-lib/src/journeys/userWorkflow.ts` | 用户 Journey 工作流实现 |
| `packages/backend-lib/src/journeys/userWorkflow/lifecycle.ts` | 工作流启动与生命周期管理 |
| `packages/backend-lib/src/journeys/userWorkflow/activities.ts` | 工作流活动 (isRunnable, sendMessage 等) |
| `packages/backend-lib/src/journeys/recordNodeProcessed.ts` | 节点处理记录 (幂等关键) |
| `packages/backend-lib/src/db/schema.ts` | Postgres 表结构 (userJourneyEvent 唯一约束) |
| `packages/isomorphic-lib/src/events.ts` | 事件名匹配算法 |

---

## 总结

### 关键设计亮点

1. **多层去重机制**: 从 Kafka 消息层到 Temporal 工作流层，共 5 层去重保证
2. **UUID v5 确定性 ID**: 使用 workspaceId 作为命名空间，确保相同业务键生成相同 ID
3. **Temporal signalWithStart**: 原子性的"启动+发送信号"操作，避免竞态条件
4. **Postgres 唯一约束 + onConflictDoNothing**: 数据库层面的幂等保障
5. **事件名模式匹配**: 支持通配符后缀，增强 Journey 配置灵活性
6. **可配置写入模式**: 支持 Kafka 缓冲、ClickHouse 异步/同步三种模式
7. **Journey 缓存**: 30 秒 TTL 缓存减少数据库查询压力

### 潜在改进点

1. **Webhook 去重**: 当前 Webhook 接收层未做消息 ID 去重，依赖外部系统保证至少一次送达
2. **事件消费延迟**: Kafka → ClickHouse → Journey 触发存在一定延迟
3. **缓存一致性**: Journey 配置变更后，30 秒缓存窗口内可能使用旧配置
