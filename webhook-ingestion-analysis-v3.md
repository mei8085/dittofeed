# Dittofeed Webhook 事件摄入链路分析报告 V3

## 目录

1. [核心修正：Segment 写入模式](#核心修正segment-写入模式)
2. [所有入口到落库的完整证据链](#所有入口到落库的完整证据链)
3. [写入层：共同点 vs 差异](#写入层共同点-vs-差异)
4. ["只入库不触发 Journey"的边界条件核查清单](#只入库不触发-journey的边界条件核查清单)
5. [三类 Webhook 匹配条件对比（修正版）](#三类-webhook-匹配条件对比修正版)
6. [关键代码索引](#关键代码索引)

---

## 核心修正：Segment 写入模式

### 1.1 之前的错误描述

**v2 报告错误描述**：
> "Segment Webhook → insertUserEvents 直接调用 (不经过 submitBatch) → 仅写入 ClickHouse"

**实际证据修正**：

Segment Webhook 虽然直接调用 `insertUserEvents`，但 `insertUserEvents` 函数内部**仍然支持三种写入模式**，由 `config().writeMode` 控制：

```typescript
// packages/backend-lib/src/userEvents.ts:90-145
export async function insertUserEvents(
  { workspaceId, userEvents, events }: InsertUserEventsParams,
  options?: { writeModeOverride?: WriteMode },
): Promise<void> {
  // 关键：写入模式由配置决定，不是硬编码
  const effectiveWriteMode = options?.writeModeOverride ?? config().writeMode;

  switch (effectiveWriteMode) {
    case "kafka":
      // 写入 Kafka 主题
      await (await kafkaProducer()).send({
        topic: userEventsTopicName,
        messages: userEventsWithDefault.map(({ messageRaw, messageId, ... }) => ({
          key: messageId,
          value: JSON.stringify({
            processing_time: processingTime,
            workspace_id: workspaceId,
            message_id: messageId,
            server_time: serverTime,
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

**配置来源**：

```typescript
// packages/backend-lib/src/config.ts:30
const BaseRawConfigProps = {
  writeMode: Type.Optional(WriteMode),  // 可配置：kafka | ch-async | ch-sync
  // ...
};
```

### 1.2 修正结论

| 入口 | 调用路径 | 写入模式配置 |
|------|---------|-------------|
| Segment Webhook | 直接调用 `insertUserEvents` | ✅ 支持 `config().writeMode` 三种模式 |
| Sendgrid Webhook | `submitBatch` → `insertUserEvents` | ✅ 支持 `config().writeMode` 三种模式 |
| Resend Webhook | `submitBatch` → `insertUserEvents` | ✅ 支持 `config().writeMode` 三种模式 |
| 客户端 Track API | `submitTrack` → `insertUserEvents` | ✅ 支持 `config().writeMode` 三种模式 |

**所有入口最终都通过 `insertUserEvents` 落库，都遵循同一套 `writeMode` 配置。**

---

## 所有入口到落库的完整证据链

### 2.1 完整调用树

```
                        ┌─────────────────────────────┐
                        │     insertUserEvents()      │
                        │   (唯一落库入口函数)        │
                        │  支持三种 writeMode 模式    │
                        └──────────────┬──────────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
   直接调用                      间接调用 (通过 submitBatch)   测试代码
        │                              │
        │              ┌───────────────┼───────────────┐
        │              │               │               │
        ▼              ▼               ▼               ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
   │Segment   │  │Sendgrid  │  │Resend    │  │Twilio    │
   │Webhook   │  │Webhook   │  │Webhook   │  │Webhook   │
   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
        │              │              │              │
        │              └──────────────┼──────────────┘
        │                             │
        │                        submitBatch()
        │                             │
        │                    ┌────────┼────────┐
        │                    │        │        │
        ▼                    ▼        ▼        ▼
   ┌──────────┐         ┌──────────┐ ┌──────────┐ ┌──────────┐
   │submit    │         │Postmark  │ │Mailchimp │ │Amazon SES│
   │Identify  │         │Webhook   │ │Webhook   │ │Webhook   │
   │submit    │         └──────────┘ └──────────┘ └──────────┘
   │Track     │
   │submit    │
   │Page      │
   │submit    │
   │Screen    │
   │track     │
   │Internal  │
   │Events    │
   └──────────┘

   (注：submitTrackWithTriggers = submitTrack + triggerEventEntryJourneys)
   (注：submitBatchWithTriggers = submitBatch + triggerEventEntryJourneys)
```

### 2.2 详细调用链证据

#### 证据 1：Segment Webhook 直接调用 insertUserEvents

**代码位置**：`packages/api/src/controllers/webhooksController.ts:682-690`

```typescript
await insertUserEvents({
  workspaceId,
  userEvents: [
    {
      messageId: request.body.messageId,
      messageRaw: request.rawBody,  // 注意：Segment 是原始透传，不做解析
    },
  ],
});
```

**关键特征**：`messageRaw` 直接是 `request.rawBody`，**不做任何解析转换**，依赖外部消费系统解析。

---

#### 证据 2：其他 Webhook 通过 submitBatch 间接调用

**Sendgrid**：`packages/backend-lib/src/destinations/sendgrid.ts:191-215`

```typescript
export async function submitSendgridEvents({
  workspaceId,
  events,
}: { ... }) {
  const data: BatchAppData = {
    batch: events.flatMap((e) =>
      sendgridEventToDF({ sendgridEvent: e }).unwrapOr([]),
    ),
  };
  await submitBatch({ workspaceId, data });  // ← 先走 submitBatch
}
```

**Resend**：`packages/backend-lib/src/destinations/resend.ts:133-161`

```typescript
export async function submitResendEvents({
  workspaceId,
  events,
}: { ... }) {
  const data: BatchAppData = {
    context: { source: SourceType.Webhook, provider: EmailProviderType.Resend },
    batch: events.flatMap((e) =>
      resendEventToDF({ workspaceId, resendEvent: e }).unwrapOr([]),
    ),
  };
  await submitBatch({ workspaceId, data });  // ← 先走 submitBatch
}
```

**Twilio**：`packages/backend-lib/src/destinations/twilio.ts:168-180`

```typescript
return ResultAsync.fromPromise(
  submitBatch({        // ← 先走 submitBatch
    workspaceId,
    data: {
      context: { source: SourceType.Webhook, provider: SmsProviderType.Twilio },
      batch: [item],
    },
  }),
  (e) => (e instanceof Error ? e : Error(e as string)),
);
```

---

#### 证据 3：submitBatch 最终调用 insertUserEvents

**代码位置**：`packages/backend-lib/src/apps/batch.ts:68-111`

```typescript
export async function submitBatchChunk(
  { workspaceId, data }: SubmitBatchOptions,
  { processingTime, writeModeOverride } = {},
) {
  const userEvents = buildBatchUserEvents(data, { processingTime });

  await insertUserEvents(       // ← 最终调用 insertUserEvents
    { workspaceId, userEvents },
    { writeModeOverride },     // ← 支持写入模式覆盖
  );
}

export async function submitBatch(
  { workspaceId, data }: SubmitBatchOptions,
  { processingTime, writeModeOverride } = {},
) {
  const chunks = R.chunk(data.batch, batchChunkSize);
  await Promise.all(
    chunks.map(async (chunk) => {
      const chunkData = { ...data, batch: chunk };
      return submitBatchChunk(
        { workspaceId, data: chunkData },
        { processingTime, writeModeOverride },  // ← 支持传入 writeModeOverride
      );
    }),
  );
}
```

**关键发现**：`submitBatch` 支持 `writeModeOverride` 参数，可以覆盖全局 `config().writeMode`。

---

#### 证据 4：客户端 API 的两种调用方式

**`submitTrack`（只入库）**：`packages/backend-lib/src/apps/track.ts:7-30`

```typescript
export async function submitTrack({
  workspaceId,
  data,
}: { ... }) {
  const userEvent: InsertUserEvent = {
    messageRaw: JSON.stringify({
      type: "track",
      properties,
      timestamp,
      ...rest,
    }),
    messageId: data.messageId,
  };
  await insertUserEvents({ workspaceId, userEvents: [userEvent] });  // ← 只入库
}
```

**`submitTrackWithTriggers`（入库 + 触发）**：`packages/backend-lib/src/apps.ts:48-90`

```typescript
export async function submitTrackWithTriggers({
  workspaceId,
  data,
}: { ... }) {
  // 第一步：入库（调用 submitTrack，最终是 insertUserEvents）
  await submitTrack({
    workspaceId,
    data: { ...data, properties },
  });

  // 第二步：额外触发 Journey
  let userOrAnonymousId: string | null = null;
  if ("userId" in data) {
    userOrAnonymousId = data.userId;
  } else if ("anonymousId" in data) {
    userOrAnonymousId = data.anonymousId;
  }

  if (userOrAnonymousId) {
    await triggerEventEntryJourneys({    // ← 关键区别：额外调用这个
      workspaceId,
      event: { ...data, properties },
      userId: userOrAnonymousId,
    });
  }
}
```

---

#### 证据 5：Batch API 的两种调用方式

**`submitBatch`（只入库）**：见证据 3

**`submitBatchWithTriggers`（入库 + 触发）**：`packages/backend-lib/src/apps.ts:92-145`

```typescript
export async function submitBatchWithTriggers({
  workspaceId,
  data: unprocessedData,
}: SubmitBatchOptions) {
  // 第一步：全部入库（调用 submitBatch）
  await submitBatch({ workspaceId, data });

  // 第二步：筛选 Track 事件，逐个触发 Journey
  const triggers: TriggerEventEntryJourneysOptions[] = data.batch.flatMap(
    (message) => {
      if (message.type !== EventType.Track) {  // ← 只处理 Track 事件
        return [];
      }
      let userOrAnonymousId: string | null = null;
      if ("userId" in message) {
        userOrAnonymousId = message.userId;
      } else if ("anonymousId" in message) {
        userOrAnonymousId = message.anonymousId;
      }
      if (!userOrAnonymousId) return [];
      return { workspaceId, event: message, userId: userOrAnonymousId };
    },
  );

  await Promise.all(
    triggers.map((trigger) => triggerEventEntryJourneys(trigger)),  // ← 触发
  );
}
```

---

#### 证据 6：其他直接调用 insertUserEvents 的入口

**submitIdentify**：`packages/backend-lib/src/apps.ts:22-46`

```typescript
export async function submitIdentify({
  workspaceId,
  data,
}: { ... }) {
  const userEvent: InsertUserEvent = {
    messageRaw: JSON.stringify({
      type: "identify",
      traits,
      timestamp,
      ...rest,
    }),
    messageId: data.messageId,
  };
  await insertUserEvents({ workspaceId, userEvents: [userEvent] });
}
```

**submitPage**：`packages/backend-lib/src/apps.ts:147-171`

**submitScreen**：`packages/backend-lib/src/apps.ts:173-197`

**trackInternalEvents**：`packages/backend-lib/src/userEvents.ts:204-230`

```typescript
export async function trackInternalEvents(props: {
  workspaceId: string;
  events: InternalEvent[];
}): Promise<Result<void, Error>> {
  const userEvents = props.events.map((mr) => ({
    userId: mr.userId,
    messageId: mr.messageId,
    messageRaw: JSON.stringify(mr),
  }));
  await insertUserEvents({ workspaceId, userEvents });
  return ok(undefined);
}
```

---

## 写入层：共同点 vs 差异

### 3.1 所有入口的共同点

| 共同点 | 证据 |
|--------|------|
| **最终都调用 `insertUserEvents`** | 见证据 1-6 |
| **都支持三种 writeMode** | `config().writeMode`: `kafka`, `ch-async`, `ch-sync` |
| **messageId 作为唯一标识** | `insertUserEvents` 中 `messageId` 用于 Kafka key 和 ClickHouse 去重 |
| **messageRaw 是 JSON 字符串** | 最终都被序列化为 `JSON.stringify()` 存入 |

### 3.2 各入口的差异

| 维度 | Segment Webhook | 其他 Webhook<br>(Sendgrid/Resend/Twilio 等) | 客户端 API<br>(Track/Identify 等) |
|------|-----------------|-------------------------------------------|---------------------------------|
| **调用路径** | 直接 `insertUserEvents` | `submitBatch` → `insertUserEvents` | 直接 `insertUserEvents`<br>或 `submitBatch` → `insertUserEvents` |
| **事件解析** | ❌ **不解析**，原始透传<br>`messageRaw = request.rawBody` | ✅ 解析转换<br>`resendEventToDF()` 等 | ✅ 解析转换<br>标准 Segment 协议 |
| **context 字段** | ❌ 无 context | ✅ 有 context<br>`{source: "webhook", provider: "resend"}` | ✅ 有 context（用户可传） |
| **serverTime 字段** | ❌ 无 | ✅ `buildBatchUserEvents` 中添加<br>`serverTime: new Date().toISOString()` | ✅ 只有 submitBatch 路径有 |
| **writeModeOverride 支持** | ❌ 无 | ✅ `submitBatch` 支持 | ✅ `submitBatch` 支持 |
| **是否触发 Journey** | ❌ 否 | ❌ 否 | ✅ 只有 `*WithTriggers` 版本触发 |

### 3.3 Segment 特殊设计解读

Segment Webhook 的"原始透传"设计意图：

```
┌─────────────────────────────────────────────────────────────────────┐
│  Segment Webhook 设计：原始透传，依赖外部消费系统解析                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Segment.com 推送 → Dittofeed /api/public/webhooks/segment          │
│                           ↓                                         │
│                   直接写入 ClickHouse/Kafka                          │
│                   (不做任何 JSON 解析)                               │
│                           ↓                                         │
│              外部 ETL 系统 / ClickHouse Materialized View            │
│              负责解析 message_raw 中的 JSON                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**代码证据**：`packages/api/src/controllers/webhooksController.ts:682-690`

```typescript
await insertUserEvents({
  workspaceId,
  userEvents: [
    {
      messageId: request.body.messageId,
      messageRaw: request.rawBody,  // ← 直接是原始字符串，不解析
    },
  ],
});
```

对比 **Resend 的解析转换**：`packages/backend-lib/src/destinations/resend.ts:66-131`

```typescript
export function resendEventToDF({
  workspaceId,
  resendEvent,
}: { ... }): Result<BatchItem, Error> {
  // 1. 提取字段
  const { type: event } = resendEvent;
  const { created_at, email_id, to } = resendEvent.data;
  const email = to[0]!;
  const { userId } = resendEvent.data.tags;
  
  // 2. 映射到内部事件类型
  switch (event) {
    case ResendEventType.Opened:
      eventName = InternalEventType.EmailOpened;
      break;
    // ...
  }
  
  // 3. 构造标准格式
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

---

## "只入库不触发 Journey"的边界条件核查清单

### 4.1 核心判断公式

```
是否触发 Journey = 是否调用了 triggerEventEntryJourneys()
```

### 4.2 核查清单（可逐行验证）

#### ✅ 会触发 Journey 的入口

| 入口 | 调用函数 | 核查代码 |
|------|---------|---------|
| `/api/public/track` | `submitTrackWithTriggers` | `apps.ts:81` 调用 `triggerEventEntryJourneys` |
| `/api/public/batch` | `submitBatchWithTriggers` | `apps.ts:143` 调用 `triggerEventEntryJourneys`（仅 Track 事件） |

#### ❌ 只入库不触发 Journey 的入口

| 入口 | 调用函数 | 核查代码 | 为什么不触发 |
|------|---------|---------|-------------|
| `/api/public/webhooks/sendgrid` | `submitSendgridEvents` → `submitBatch` | `sendgrid.ts:211` 只调用 `submitBatch`，无 `triggerEventEntryJourneys` | 邮件状态回调，用于分析 |
| `/api/public/webhooks/resend` | `submitResendEvents` → `submitBatch` | `resend.ts:157` 只调用 `submitBatch` | 同上 |
| `/api/public/webhooks/twilio` | `submitTwilioEvents` → `submitBatch` | `twilio.ts:169` 只调用 `submitBatch` | 同上（短信） |
| `/api/public/webhooks/postmark` | `submitPostmarkEvents` → `submitBatch` | `postmark.ts:159` 只调用 `submitBatch` | 同上 |
| `/api/public/webhooks/mailchimp` | `submitMailChimpEvents` → `submitBatch` | `mailchimp.ts:209` 只调用 `submitBatch` | 同上 |
| `/api/public/webhooks/amazon-ses` | `submitAmazonSesEvents` → `submitBatch` | `amazonses.ts:264` 只调用 `submitBatch` | 同上 |
| `/api/public/webhooks/segment` | `insertUserEvents`（直接） | `webhooksController.ts:682` 直接调用 `insertUserEvents` | 原始透传，依赖外部消费 |
| `/api/public/identify` | `submitIdentify` | `apps.ts:42` 只调用 `insertUserEvents` | Identify 只更新用户属性 |
| `/api/public/page` | `submitPage` | `apps.ts:167` 只调用 `insertUserEvents` | Page 事件不触发 Journey |
| `/api/public/screen` | `submitScreen` | `apps.ts:193` 只调用 `insertUserEvents` | Screen 事件不触发 Journey |
| `/api/public/group` | `submitGroup` → `submitBatch` | `apps.ts:207` 只调用 `submitBatch` | Group 事件会拆分为 Track+Identify，但不触发 Journey |

### 4.3 关键代码核查点

#### 核查点 1：submitBatch 是否调用 triggerEventEntryJourneys？

**答案：否**

```typescript
// packages/backend-lib/src/apps/batch.ts:89-111
export async function submitBatch(...) {
  // 只做分块和入库，没有任何 trigger 相关逻辑
  await Promise.all(
    chunks.map(async (chunk) => {
      return submitBatchChunk(...);
    }),
  );
}

// packages/backend-lib/src/apps/batch.ts:68-87
export async function submitBatchChunk(...) {
  const userEvents = buildBatchUserEvents(data, { processingTime });
  await insertUserEvents(   // ← 直接到 insertUserEvents，没有 trigger
    { workspaceId, userEvents },
    { writeModeOverride },
  );
}
```

#### 核查点 2：submitTrackWithTriggers 的差异在哪？

**答案：多了一步 triggerEventEntryJourneys**

```typescript
// packages/backend-lib/src/apps.ts:48-90
export async function submitTrackWithTriggers(...) {
  // 第一步：入库（与 submitTrack 相同）
  await submitTrack({ workspaceId, data: { ...data, properties } });

  // 第二步：额外触发（这是关键差异）
  if (userOrAnonymousId) {
    await triggerEventEntryJourneys({   // ← 只有 WithTriggers 版本有这行
      workspaceId,
      event: { ...data, properties },
      userId: userOrAnonymousId,
    });
  }
}
```

#### 核查点 3：只有 Track 事件能触发 Journey？

**答案：是**

```typescript
// packages/backend-lib/src/apps.ts:120-140
const triggers: TriggerEventEntryJourneysOptions[] = data.batch.flatMap(
  (message) => {
    // 关键过滤：只处理 EventType.Track
    if (message.type !== EventType.Track) {
      return [];  // Identify/Page/Screen/Group 都被过滤掉
    }
    // ...
  },
);
```

### 4.4 边界条件总结

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  触发 Journey 的必要条件（必须同时满足）：                                │
│                                                                         │
│  1. 事件类型 = EventType.Track                                          │
│     (Identify/Page/Screen/Group 都不会触发)                             │
│                                                                         │
│  2. 有 userId 或 anonymousId                                            │
│                                                                         │
│  3. 调用函数是 *WithTriggers 版本                                        │
│     - submitTrackWithTriggers ✅                                        │
│     - submitBatchWithTriggers ✅ (仅 Track 事件)                         │
│                                                                         │
│  4. Journey 状态 = Running                                              │
│                                                                         │
│  5. Journey 入口类型 = EventEntryNode                                   │
│                                                                         │
│  6. 事件名匹配 (精确匹配或通配符*)                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 三类 Webhook 匹配条件对比（修正版）

### 5.1 修正后的总体对比

| 维度 | Sendgrid | Resend | Twilio |
|------|----------|--------|--------|
| **Webhook 端点** | `/api/public/webhooks/sendgrid` | `/api/public/webhooks/resend` | `/api/public/webhooks/twilio` |
| **验证方式** | HMAC 签名 (x-twilio-email-*) | Svix 签名 (svix-id/timestamp/signature) | Twilio 官方签名验证 |
| **Workspace ID 来源** | 事件 metadata / smtp-id 回溯 | `request.body.data.tags.workspaceId` | `request.query.workspaceId` (URL 参数) |
| **messageId 生成** | 三类：<br>processed: `processed:{smtp-id}`<br>bounce/spam: `bounce:{smtp-id}`<br>其他: `uuidv5(event:sg_message_id, workspaceId)` | `uuidv5(event:email_id, workspaceId)` | `uuidv5(SmsStatus:MessageSid, workspaceId)` |
| **User ID 来源** | 事件 metadata (优先)<br>bounce/spam 事件需回溯 processed 事件 | `request.body.data.tags.userId` | `request.query.userId` (URL 参数) |
| **事件类型数量** | 7 种 | 6 种 | 2 种 |
| **写入模式** | ✅ 遵循 `config().writeMode` (kafka/ch-async/ch-sync) | ✅ 遵循 `config().writeMode` | ✅ 遵循 `config().writeMode` |
| **是否触发 Journey** | ❌ 否 | ❌ 否 | ❌ 否 |
| **是否解析事件** | ✅ 解析为标准 Track 事件 | ✅ 解析为标准 Track 事件 | ✅ 解析为标准 Track 事件 |

### 5.2 Sendgrid 详细匹配条件

#### 数据来源

| 数据项 | 来源 | 说明 |
|--------|------|------|
| workspaceId | 优先：事件 `workspaceId` 字段<br>其次：通过 smtp-id 回溯 processed 事件的 metadata | bounce/spam 事件可能不直接携带 metadata |
| userId | 优先：事件 `userId` 字段<br>其次：回溯 processed 事件的 metadata | 同上 |
| 原始事件时间 | 事件 `timestamp` 字段 (Unix 秒) | 需乘以 1000 转毫秒 |
| messageId | 分三类：<br>processed → `processed:{smtp-id}`<br>bounce/spam → `bounce:{smtp-id}` / `spamreport:{smtp-id}`<br>其他 → `uuidv5(event:sg_message_id, workspaceId)` | smtp-id 用于异步事件关联 |

#### 匹配规则

| 规则 | 说明 |
|------|------|
| **事件类型映射** | `open`→`DFEmailOpened`, `click`→`DFEmailClicked`, `bounce`→`DFEmailBounced`, `dropped`→`DFEmailDropped`, `spamreport`→`DFEmailMarkedSpam`, `delivered`→`DFEmailDelivered`, `processed`→`DFEmailProcessed` |
| **异步事件回溯** | bounce/spam 事件可能不携带 metadata，需通过 smtp-id 查找之前的 processed 事件，从 properties 中提取 workspaceId 和 userId |
| **写入** | `submitBatch` → `insertUserEvents`，遵循 `config().writeMode` |

#### 触发动作

| 动作 | 说明 |
|------|------|
| **入库** | 写入 Kafka 或 ClickHouse（取决于配置） |
| **触发 Journey** | ❌ 不触发（只调用 `submitBatch`，无 `triggerEventEntryJourneys`） |
| **用途** | 邮件发送状态统计、打开率分析、点击追踪 |

### 5.3 Resend 详细匹配条件

#### 数据来源

| 数据项 | 来源 | 说明 |
|--------|------|------|
| workspaceId | `request.body.data.tags.workspaceId` | 必须存在，否则 200 静默忽略 |
| userId | `request.body.data.tags.userId` | 必须存在，否则转换失败 |
| 原始事件时间 | `request.body.data.created_at` (ISO 字符串) | 直接使用 |
| messageId | `uuidv5(event:email_id, workspaceId)` | email_id 是 Resend 分配的邮件 ID |

#### 匹配规则

| 规则 | 说明 |
|------|------|
| **事件类型映射** | `email.opened`→`DFEmailOpened`, `email.clicked`→`DFEmailClicked`, `email.bounced`→`DFEmailBounced`, `email.delivery_delayed`→`DFEmailDropped`, `email.complained`→`DFEmailMarkedSpam`, `email.delivered`→`DFEmailDelivered` |
| **注意** | `email.sent` 事件没有映射，会被忽略 |
| **写入** | `submitBatch` → `insertUserEvents`，遵循 `config().writeMode` |

#### 触发动作

| 动作 | 说明 |
|------|------|
| **入库** | 写入 Kafka 或 ClickHouse（取决于配置） |
| **触发 Journey** | ❌ 不触发 |
| **用途** | 邮件发送状态统计 |

### 5.4 Twilio 详细匹配条件

#### 数据来源

| 数据项 | 来源 | 说明 |
|--------|------|------|
| workspaceId | `request.query.workspaceId` | **URL 查询参数**，发送 SMS 时构造到回调 URL |
| userId | `request.query.userId` | **URL 查询参数** |
| 订阅组 | `request.query.subscriptionGroupId` | URL 查询参数 |
| 其他 tags | `request.query` 中其他字段 | 全部作为 properties |
| 原始事件时间 | **未直接使用** | 使用服务器当前时间 |
| messageId | `uuidv5(SmsStatus:MessageSid, workspaceId)` | 使用 Twilio 的 MessageSid |

#### 匹配规则

| 规则 | 说明 |
|------|------|
| **事件类型映射** | `failed`→`DFSmsFailed`, `delivered`→`DFSmsDelivered` |
| **注意 1** | 只有 2 种状态被处理，其他状态（queued、sending、sent、undelivered、received 等）会被忽略 |
| **注意 2** | 代码中有 `// TODO wrong` 注释，`DFSmsDelivered` 映射可能存在问题 |
| **写入** | `submitBatch` → `insertUserEvents`，遵循 `config().writeMode` |

#### 触发动作

| 动作 | 说明 |
|------|------|
| **入库** | 写入 Kafka 或 ClickHouse（取决于配置） |
| **触发 Journey** | ❌ 不触发 |
| **用途** | 短信发送状态统计 |

---

## 关键代码索引

| 文件 | 功能 | 关键行号 |
|------|------|---------|
| `packages/backend-lib/src/userEvents.ts` | `insertUserEvents`（唯一落库入口，支持 3 种写入模式） | 90-145 |
| `packages/backend-lib/src/apps/batch.ts` | `submitBatch`（分块、构建 InsertUserEvent） | 13-111 |
| `packages/backend-lib/src/apps.ts` | `submitTrackWithTriggers` / `submitBatchWithTriggers`（唯一会触发 Journey 的入口） | 48-145 |
| `packages/backend-lib/src/apps/track.ts` | `submitTrack`（只入库，不触发） | 7-30 |
| `packages/api/src/controllers/webhooksController.ts` | 所有 Webhook 端点 | 76-694 |
| `packages/backend-lib/src/destinations/sendgrid.ts` | Sendgrid 事件转换（含异步回溯逻辑） | 77-215, 217-370 |
| `packages/backend-lib/src/destinations/resend.ts` | Resend 事件转换 | 66-161 |
| `packages/backend-lib/src/destinations/twilio.ts` | Twilio 事件转换 | 46-85, 113-181 |
| `packages/backend-lib/src/journeys.ts` | `triggerEventEntryJourneys`（Journey 触发入口） | 688-812 |
| `packages/backend-lib/src/config.ts` | `writeMode` 配置定义 | 30 |

---

## 总结

### 6.1 核心修正点

1. **Segment Webhook 不是"仅写入 ClickHouse"**，它像所有其他入口一样，遵循 `config().writeMode` 配置，支持 Kafka、ch-async、ch-sync 三种模式

2. **所有入口最终都通过 `insertUserEvents` 落库**，这是唯一的落库入口函数

3. **"只入库不触发 Journey"的判断标准**：是否调用了 `triggerEventEntryJourneys`

### 6.2 架构设计原则

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  两条平行链路，职责分离：                                            │
│                                                                     │
│  链路 A：状态反馈（邮件/SMS Webhook）                                 │
│  ─────────────────────────────                                     │
│  Journey → 发送邮件 → 邮件提供商 → Webhook → submitBatch → 入库     │
│  用途：状态统计、打开率分析                                          │
│  特征：只入库，不触发新 Journey                                      │
│                                                                     │
│  链路 B：业务触发（客户端 Track API）                                │
│  ─────────────────────────────                                    │
│  客户端 → /api/public/track → submitTrackWithTriggers →            │
│  insertUserEvents + triggerEventEntryJourneys → 新 Journey          │
│  用途：触发业务自动化流程                                            │
│  特征：入库 + 触发 Journey                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.3 设计意图

1. **避免循环触发**：邮件/SMS Webhook 是 Journey 发送后的状态反馈，如果这些事件也能触发 Journey，可能造成无限循环

2. **显式 vs 隐式**：
   - 业务触发必须显式使用 `*WithTriggers` 版本
   - 系统内部事件（邮件状态、Segment 透传）使用非 `*WithTriggers` 版本，只做持久化

3. **Track 事件是唯一触发点**：
   - Identify：更新用户属性，不触发
   - Page/Screen：页面浏览，不触发
   - Group：分组管理，不触发
   - Track：用户行为，可触发
