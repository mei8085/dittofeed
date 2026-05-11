# Dittofeed Webhook 事件摄入链路分析报告 V2

## 目录

1. [核心问题澄清](#核心问题澄清)
2. [所有 Webhook 提供商的处理路径汇总](#所有-webhook-提供商的处理路径汇总)
3. [submitBatch vs submitBatchWithTriggers 调用链对比](#submitbatch-vs-submitbatchwithtriggers-调用链对比)
4. [三类 Webhook 的匹配条件详细对比](#三类-webhook-的匹配条件详细对比)
5. [Journey 触发匹配条件详解](#journey-触发匹配条件详解)
6. [关键代码索引](#关键代码索引)

---

## 核心问题澄清

### 1.1 哪些 Webhook 事件会触发 Journey？

**答案：NONE** — 所有 Webhook 事件（Sendgrid、Resend、Twilio、Postmark、Mailchimp、Amazon SES）都**只入库不触发 Journey**。

| 提供商 | 调用函数 | 触发 Journey？ | 说明 |
|-------|---------|---------------|------|
| Sendgrid | `submitSendgridEvents` → `submitBatch` | ❌ 否 | 邮件发送状态回调，仅用于数据统计 |
| Resend | `submitResendEvents` → `submitBatch` | ❌ 否 | 邮件发送状态回调，仅用于数据统计 |
| Twilio | `submitTwilioEvents` → `submitBatch` | ❌ 否 | 短信发送状态回调，仅用于数据统计 |
| Postmark | `submitPostmarkEvents` → `submitBatch` | ❌ 否 | 邮件发送状态回调，仅用于数据统计 |
| Mailchimp | `submitMailChimpEvents` → `submitBatch` | ❌ 否 | 邮件发送状态回调，仅用于数据统计 |
| Amazon SES | `submitAmazonSesEvents` → `submitBatch` | ❌ 否 | 邮件发送状态回调，仅用于数据统计 |
| Segment | `insertUserEvents` (直接) | ❌ 否 | 原始事件透传，依赖外部消费 |
| **客户端 Track API** | `submitTrackWithTriggers` | ✅ 是 | `/api/public/track` 端点 |
| **客户端 Batch API** | `submitBatchWithTriggers` | ✅ 是 | `/api/public/batch` 端点 (其中 Track 事件) |
| **手动分段上传** | `submitBatchWithTriggers` | ✅ 是 | CSV 上传用户更新时 |

### 1.2 为什么邮件/短信提供商的 Webhook 不触发 Journey？

**架构设计意图**：

1. **事件性质不同**：邮件/SMS 提供商的 Webhook 是**状态回调**（已送达、已打开、已点击、已退回等），用于**反馈统计**，而非**业务事件触发**

2. **单向反馈回路**：
   - Journey → 发送邮件/SMS → 邮件/SMS 提供商
   - ← 状态回调（Webhook）←
   - 这些回调是**系统内部状态追踪**，不应该再次触发新的 Journey（避免无限循环）

3. **触发入口在客户端**：
   - 业务事件（如"用户注册"、"订单完成"）通过 **客户端 SDK 调用 `/api/public/track`** 触发
   - 使用的是 `submitTrackWithTriggers` 函数，它会额外调用 `triggerEventEntryJourneys`

---

## 所有 Webhook 提供商的处理路径汇总

### 2.1 完整调用链对比

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Webhook 事件处理路径 (只入库，不触发)                     │
└─────────────────────────────────────────────────────────────────────────────────┘

Sendgrid Webhook
    ↓
webhooksController.ts:91-98
    ↓
handleSendgridEvents (destinations/sendgrid.ts:217-370)
    ↓
submitSendgridEvents (destinations/sendgrid.ts:191-215)
    ↓
submitBatch (apps/batch.ts:89-111)
    ↓
insertUserEvents (userEvents.ts:90-145)
    ↓
Kafka / ClickHouse 直写 (取决于配置)
    ↓
【STOP】只入库，不调用 triggerEventEntryJourneys


┌─────────────────────────────────────────────────────────────────────────────────┐
│                         客户端 API 事件处理路径 (入库 + 触发)                       │
└─────────────────────────────────────────────────────────────────────────────────┘

客户端调用 /api/public/track
    ↓
publicAppsController.ts:91-107
    ↓
submitTrackWithTriggers (apps.ts:48-90)
    ├─→ submitTrack (apps/track.ts) → insertUserEvents ← 入库
    │
    └─→ triggerEventEntryJourneys (journeys.ts:688-812) ← 触发 Journey
            ↓
        遍历该 Workspace 所有 Running 状态的 Journey
            ↓
        doesEventNameMatch 匹配事件名
            ↓
        startKeyedUserJourney 启动工作流


┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Segment Webhook 特殊路径 (原始透传)                        │
└─────────────────────────────────────────────────────────────────────────────────┘

Segment Webhook
    ↓
webhooksController.ts:637-693
    ↓
insertUserEvents 直接调用 (不经过 submitBatch)
    ↓
仅写入 ClickHouse，不做任何解析
    ↓
【STOP】依赖外部系统消费解析
```

### 2.2 关键代码证据

**Sendgrid 只调用 submitBatch：**
```typescript
// packages/backend-lib/src/destinations/sendgrid.ts:191-215
export async function submitSendgridEvents({
  workspaceId,
  events,
}: { ... }) {
  const data: BatchAppData = {
    batch: events.flatMap((e) =>
      sendgridEventToDF({ sendgridEvent: e })
        .unwrapOr([]),
    ),
  };
  await submitBatch({    // ← 注意：这里是 submitBatch，不是 submitBatchWithTriggers
    workspaceId,
    data,
  });
}
```

**Resend 只调用 submitBatch：**
```typescript
// packages/backend-lib/src/destinations/resend.ts:133-161
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
  await submitBatch({    // ← 同样是 submitBatch
    workspaceId,
    data,
  });
}
```

**Twilio 只调用 submitBatch：**
```typescript
// packages/backend-lib/src/destinations/twilio.ts:168-180
return ResultAsync.fromPromise(
  submitBatch({        // ← 也是 submitBatch
    workspaceId,
    data: {
      context: { source: SourceType.Webhook, provider: SmsProviderType.Twilio },
      batch: [item],
    },
  }),
  (e) => (e instanceof Error ? e : Error(e as string)),
);
```

**客户端 Track 调用 submitTrackWithTriggers：**
```typescript
// packages/api/src/controllers/publicAppsController.ts:91-107
async (request, reply) => {
  const workspaceIdFromWriteKey = await validateWriteKey({ ... });
  await submitTrackWithTriggers({    // ← 这里是 WithTriggers 版本
    workspaceId: workspaceIdFromWriteKey.value,
    data: request.body,
  });
  return reply.status(204).send();
}
```

---

## submitBatch vs submitBatchWithTriggers 调用链对比

### 3.1 submitBatch（只入库）

**文件**：`packages/backend-lib/src/apps/batch.ts:89-111`

```
submitBatch({ workspaceId, data })
    ↓
buildBatchUserEvents(data) → 转换为 InsertUserEvent[]
    ↓
分批次 (chunk) 处理
    ↓
insertUserEvents({ workspaceId, userEvents }, { writeModeOverride })
    ├─→ "kafka" 模式：写入 Kafka 主题 user_events_v2
    ├─→ "ch-async" 模式：异步写入 ClickHouse
    └─→ "ch-sync" 模式：同步写入 ClickHouse
    ↓
【END】无后续触发逻辑
```

### 3.2 submitTrackWithTriggers（入库 + 触发）

**文件**：`packages/backend-lib/src/apps.ts:48-90`

```
submitTrackWithTriggers({ workspaceId, data })
    │
    ├─→ 第一步：持久化文件（如果有附件）
    │       ↓
    │       persistFiles({ files, messageId, properties, workspaceId })
    │
    ├─→ 第二步：入库（与 submitTrack 相同）
    │       ↓
    │       submitTrack({ workspaceId, data: { ...data, properties } })
    │           ↓
    │       insertUserEvents(...) → Kafka / ClickHouse
    │
    └─→ 第三步：提取用户 ID 并触发 Journey
            ↓
            userId = data.userId ?? data.anonymousId
            ↓
            triggerEventEntryJourneys({
              workspaceId,
              event: data,
              userId,
            })
            ↓
            遍历匹配的 Journey → 启动工作流
```

### 3.3 submitBatchWithTriggers（批量入库 + 批量触发）

**文件**：`packages/backend-lib/src/apps.ts:92-145`

```
submitBatchWithTriggers({ workspaceId, data })
    │
    ├─→ 第一步：持久化文件（Track 事件的附件）
    │       ↓
    │       遍历 batch，处理 EventType.Track 且有 files 的事件
    │       → persistFiles 持久化，更新 properties
    │
    ├─→ 第二步：全部入库
    │       ↓
    │       submitBatch({ workspaceId, data }) → 全部写入 Kafka/ClickHouse
    │
    └─→ 第三步：筛选 Track 事件触发 Journey
            ↓
            triggers = data.batch.flatMap((message) => {
              if (message.type !== EventType.Track) return [];  // 只处理 Track
              userId = message.userId ?? message.anonymousId
              if (!userId) return [];
              return { workspaceId, event: message, userId };
            })
            ↓
            Promise.all(
              triggers.map((trigger) =>
                triggerEventEntryJourneys(trigger)  // 逐个触发
              )
            )
```

### 3.4 核心差异总结表

| 维度 | submitBatch | submitTrackWithTriggers | submitBatchWithTriggers |
|------|-------------|------------------------|------------------------|
| 入库 | ✅ 是 | ✅ 是 (通过 submitTrack) | ✅ 是 (通过 submitBatch) |
| 触发 Journey | ❌ 否 | ✅ 是 (单个) | ✅ 是 (批量) |
| 适用场景 | 邮件/SMS 状态回调、系统事件 | 客户端单事件 Track | 客户端批量事件 |
| 调用方 | Webhook 控制器 | `/api/public/track` | `/api/public/batch` |
| 事件类型过滤 | 无 | 无 (只处理一个) | 只处理 `EventType.Track` |

---

## 三类 Webhook 的匹配条件详细对比

### 4.1 Sendgrid vs Resend vs Twilio 总体对比

| 维度 | Sendgrid | Resend | Twilio |
|------|----------|--------|--------|
| **Webhook 端点** | `/api/public/webhooks/sendgrid` | `/api/public/webhooks/resend` | `/api/public/webhooks/twilio` |
| **验证方式** | HMAC 签名 (x-twilio-email-*) | Svix 签名 (svix-id/timestamp/signature) | Twilio 官方签名验证 |
| **Workspace ID 来源** | 事件 metadata / smtp-id 回溯 | `request.body.data.tags.workspaceId` | `request.query.workspaceId` (URL 参数) |
| **messageId 生成** | 三类：<br>processed: `processed:{smtp-id}`<br>bounce/spam: `bounce:{smtp-id}`<br>其他: `uuidv5(event:sg_message_id, workspaceId)` | `uuidv5(event:email_id, workspaceId)` | `uuidv5(SmsStatus:MessageSid, workspaceId)` |
| **User ID 来源** | 事件 metadata (优先)<br>bounce/spam 事件需回溯 processed 事件 | `request.body.data.tags.userId` | `request.query.userId` (URL 参数) |
| **事件类型数量** | 7 种 | 6 种 | 2 种 |
| **是否触发 Journey** | ❌ 否 | ❌ 否 | ❌ 否 |

---

### 4.2 Sendgrid 详细匹配条件

#### 4.2.1 数据来源

| 数据项 | 来源 | 说明 |
|--------|------|------|
| workspaceId | 优先：事件 `workspaceId` 字段<br>其次：通过 smtp-id 回溯 processed 事件的 metadata | bounce/spam 事件可能不直接携带 metadata |
| userId | 优先：事件 `userId` 字段<br>其次：回溯 processed 事件的 metadata | 同上 |
| 原始事件时间 | 事件 `timestamp` 字段 (Unix 秒) | 需乘以 1000 转毫秒 |
| messageId | 分三类：<br>processed → `processed:{smtp-id}`<br>bounce/spam → `bounce:{smtp-id}` / `spamreport:{smtp-id}`<br>其他 → `uuidv5(event:sg_message_id, workspaceId)` | smtp-id 用于异步事件关联 |

#### 4.2.2 消息 ID 生成策略（Sendgrid 独有）

**代码位置**：`packages/backend-lib/src/destinations/sendgrid.ts:102-136`

```typescript
switch (event) {
  case "processed":
    // 使用 smtp-id 作为前缀，便于后续 bounce/spam 回溯
    messageId = `processed:${smtpId}`;
    break;
  case "bounce":
    messageId = `bounce:${smtpId}`;
    break;
  case "spamreport":
    messageId = `spamreport:${smtpId}`;
    break;
  default:
    // 其他事件使用 uuidv5
    messageId = uuidv5(`${event}:${sg_message_id}`, sendgridEvent.workspaceId);
    break;
}
```

**设计原因**：
- Sendgrid 的 bounce/spam 事件可能**不携带 metadata**（因为异步发送时可能丢失）
- 需要通过 `smtp-id` 回溯之前的 `processed` 事件，从中获取 workspaceId 和 userId

**回溯逻辑**：`packages/backend-lib/src/destinations/sendgrid.ts:241-306`

```typescript
const delayedEvents = new Map<string, SendgridEvent>();
for (const event of sendgridEvents) {
  switch (event.event) {
    case "spamreport":
    case "bounce":
      // 暂存延迟事件，等待回溯
      delayedEvents.set(event["smtp-id"], event);
      break;
    default:
      immediateEvents.push(event);
      break;
  }
}

// 查找对应的 processed 事件
const processedForDelayedEvents = await findUserEventsById({
  messageIds: Array.from(delayedEvents.keys()).map(
    (id) => `processed:${id}`  // 用 processed: 前缀查找
  ),
});

// 回溯获取 workspaceId 和 userId
for (const event of processedForDelayedEvents) {
  const smtpId = event.message_id.split(":")[1];
  const delayedEvent = delayedEvents.get(smtpId);
  // 从 processed 事件的 properties 中提取 metadata
  const parsedProperties = jsonParseSafeWithSchema(
    event.properties,
    MessageMetadataFields,
  );
  // backfill 到延迟事件
  backfilledDelayedEvents.push({
    ...delayedEvent,
    ...parsedProperties.value,  // 注入 workspaceId, userId 等
  });
}
```

#### 4.2.3 事件类型映射

| Sendgrid 原始事件 | 内部事件类型 (InternalEventType) | 触发动作 |
|------------------|--------------------------------|---------|
| `open` | `DFEmailOpened` | 仅入库，用于分析 |
| `click` | `DFEmailClicked` | 仅入库，用于分析 |
| `bounce` | `DFEmailBounced` | 仅入库，用于分析 |
| `dropped` | `DFEmailDropped` | 仅入库，用于分析 |
| `spamreport` | `DFEmailMarkedSpam` | 仅入库，用于分析 |
| `delivered` | `DFEmailDelivered` | 仅入库，用于分析 |
| `processed` | `DFEmailProcessed` | 仅入库，用于后续回溯 |

**代码位置**：`packages/backend-lib/src/destinations/sendgrid.ts:138-171`

```typescript
switch (event) {
  case "open":
    eventName = InternalEventType.EmailOpened;
    break;
  case "click":
    eventName = InternalEventType.EmailClicked;
    break;
  case "bounce":
    eventName = InternalEventType.EmailBounced;
    break;
  case "dropped":
    eventName = InternalEventType.EmailDropped;
    break;
  case "spamreport":
    eventName = InternalEventType.EmailMarkedSpam;
    break;
  case "delivered":
    eventName = InternalEventType.EmailDelivered;
    break;
  case "processed":
    eventName = InternalEventType.EmailProcessed;
    break;
}
```

#### 4.2.4 验证方式

**代码位置**：`packages/backend-lib/src/destinations/sendgrid.ts:338-354`

```typescript
const publicKey = `-----BEGIN PUBLIC KEY-----\n${webhookKey}\n-----END PUBLIC KEY-----`;

const verified = verifyTimestampedSignature({
  signature: webhookSignature,
  timestamp: webhookTimestamp,
  payload: rawBody,
  publicKey,
});
```

**请求头**：
- `x-twilio-email-event-webhook-signature`
- `x-twilio-email-event-webhook-timestamp`

---

### 4.3 Resend 详细匹配条件

#### 4.3.1 数据来源

| 数据项 | 来源 | 说明 |
|--------|------|------|
| workspaceId | `request.body.data.tags.workspaceId` | 必须存在，否则 200 静默忽略 |
| userId | `request.body.data.tags.userId` | 必须存在，否则转换失败 |
| 原始事件时间 | `request.body.data.created_at` (ISO 字符串) | 直接使用 |
| messageId | `uuidv5(event:email_id, workspaceId)` | email_id 是 Resend 分配的邮件 ID |

**代码位置**：`packages/backend-lib/src/destinations/resend.ts:80-84`

```typescript
const { userId } = resendEvent.data.tags;
if (!userId) {
  return err(new Error("Missing userId or anonymousId."));
}
const messageId = uuidv5(`${event}:${email_id}`, workspaceId);
```

#### 4.3.2 事件类型映射

| Resend 原始事件 | 内部事件类型 | 说明 |
|-----------------|-------------|------|
| `email.opened` | `DFEmailOpened` | 邮件打开 |
| `email.clicked` | `DFEmailClicked` | 链接点击 |
| `email.bounced` | `DFEmailBounced` | 邮件退回 |
| `email.delivery_delayed` | `DFEmailDropped` | 投递延迟 |
| `email.complained` | `DFEmailMarkedSpam` | 垃圾邮件投诉 |
| `email.delivered` | `DFEmailDelivered` | 成功投递 |
| `email.sent` | ❌ 未映射 | 注意：sent 事件被忽略 |

**代码位置**：`packages/backend-lib/src/destinations/resend.ts:88-109`

```typescript
switch (event) {
  case ResendEventType.Opened:
    eventName = InternalEventType.EmailOpened;
    break;
  case ResendEventType.Clicked:
    eventName = InternalEventType.EmailClicked;
    break;
  case ResendEventType.Bounced:
    eventName = InternalEventType.EmailBounced;
    break;
  case ResendEventType.DeliveryDelayed:
    eventName = InternalEventType.EmailDropped;
    break;
  case ResendEventType.Complained:
    eventName = InternalEventType.EmailMarkedSpam;
    break;
  case ResendEventType.Delivered:
    eventName = InternalEventType.EmailDelivered;
    break;
  default:
    return err(new Error(`Unhandled event type: ${event}`));
}
```

**注意**：`email.sent` 事件没有映射到任何内部事件类型，会被忽略。

#### 4.3.3 验证方式

**代码位置**：`packages/api/src/controllers/webhooksController.ts:252-265`

```typescript
const wh = new Webhook(webhookKey);  // Svix Webhook 客户端
const verified = wh.verify(request.rawBody, request.headers);
```

**请求头**：
- `svix-id`
- `svix-timestamp`
- `svix-signature`

---

### 4.4 Twilio 详细匹配条件

#### 4.4.1 数据来源

| 数据项 | 来源 | 说明 |
|--------|------|------|
| workspaceId | `request.query.workspaceId` | **URL 查询参数** |
| userId | `request.query.userId` | **URL 查询参数** |
| 订阅组 | `request.query.subscriptionGroupId` | URL 查询参数 |
| 其他 tags | `request.query` 中其他字段 | 全部作为 properties |
| 原始事件时间 | **未直接使用** | 使用服务器当前时间 |
| messageId | `uuidv5(SmsStatus:MessageSid, workspaceId)` | 使用 Twilio 的 MessageSid |

**代码位置**：`packages/backend-lib/src/destinations/twilio.ts:46-85`

```typescript
// 发送 SMS 时构造的回调 URL
const baseCallbackUrl = `${config().dashboardUrl}/api/public/webhooks/twilio`;
const queryParams = qs.stringify({
  ...omitBy(tags, (_v, key) => key === "channel"),
  subscriptionGroupId,
  userId,
  workspaceId,  // ← 放在 URL 参数中
});
statusCallback = `${baseCallbackUrl}?${queryParams}`;
```

**控制器中提取**：`packages/api/src/controllers/webhooksController.ts:545-546`

```typescript
const { workspaceId, userId, subscriptionGroupId, ...tags } = request.query;
```

#### 4.4.2 事件类型映射

| Twilio 状态 | 内部事件类型 | 说明 |
|-------------|-------------|------|
| `failed` | `DFSmsFailed` | 发送失败 |
| `delivered` | `DFSmsDelivered` | 成功送达 |
| `queued` | ❌ 未映射 | 被忽略 |
| `sending` | ❌ 未映射 | 被忽略 |
| `sent` | ❌ 未映射 | 被忽略 |
| `undelivered` | ❌ 未映射 | 被忽略 |
| `receiving` | ❌ 未映射 | 被忽略 |
| `received` | ❌ 未映射 | 被忽略 |
| `accepted` | ❌ 未映射 | 被忽略 |
| `scheduled` | ❌ 未映射 | 被忽略 |
| `read` | ❌ 未映射 | 被忽略 |
| `partially_delivered` | ❌ 未映射 | 被忽略 |
| `canceled` | ❌ 未映射 | 被忽略 |

**代码位置**：`packages/backend-lib/src/destinations/twilio.ts:124-144`

```typescript
switch (TwilioEvent.SmsStatus) {
  case TwilioMessageStatus.Failed:
    eventName = InternalEventType.SmsFailed;
    break;
  case TwilioMessageStatus.Delivered:
    // TODO wrong  ← 代码注释：这里可能有问题
    eventName = InternalEventType.SmsDelivered;
    break;
  default:
    logger().error({ ... }, "Unhandled Twilio event type");
    return err(new Error(`Unhandled Twilio event type: ${TwilioEvent.SmsStatus}`));
}
```

**注意**：代码中有 `// TODO wrong` 注释，可能存在事件类型映射错误。

#### 4.4.3 验证方式

**代码位置**：`packages/api/src/controllers/webhooksController.ts:578-583`

```typescript
const verified = validateRequest(
  twilioSecret.authToken,
  request.headers["x-twilio-signature"],
  `${backendConfig().dashboardUrl}${request.url}`,
  request.body,
);
```

使用 Twilio 官方 SDK 的 `validateRequest` 函数验证签名。

---

### 4.5 三类 Webhook 事件类型对比表

| 事件描述 | Sendgrid | Resend | Twilio |
|---------|----------|--------|--------|
| 邮件送达 | `delivered` → `DFEmailDelivered` | `email.delivered` → `DFEmailDelivered` | N/A (短信) |
| 邮件打开 | `open` → `DFEmailOpened` | `email.opened` → `DFEmailOpened` | N/A |
| 链接点击 | `click` → `DFEmailClicked` | `email.clicked` → `DFEmailClicked` | N/A |
| 邮件退回 | `bounce` → `DFEmailBounced` | `email.bounced` → `DFEmailBounced` | N/A |
| 垃圾邮件 | `spamreport` → `DFEmailMarkedSpam` | `email.complained` → `DFEmailMarkedSpam` | N/A |
| 投递延迟/失败 | `dropped` → `DFEmailDropped` | `email.delivery_delayed` → `DFEmailDropped` | N/A |
| 已处理 | `processed` → `DFEmailProcessed` | ❌ 无 | N/A |
| 短信送达 | N/A | N/A | `delivered` → `DFSmsDelivered` |
| 短信失败 | N/A | N/A | `failed` → `DFSmsFailed` |

---

## Journey 触发匹配条件详解

### 5.1 触发入口

只有以下入口会调用 `triggerEventEntryJourneys`：

1. **`submitTrackWithTriggers`** → `/api/public/track` 端点
2. **`submitBatchWithTriggers`** → `/api/public/batch` 端点（仅 Track 事件）

### 5.2 匹配流程

**代码位置**：`packages/backend-lib/src/journeys.ts:688-812`

```
triggerEventEntryJourneys({ workspaceId, event, userId })
    ↓
1. 从缓存获取或查询该 Workspace 所有 Journey
    ↓
   EVENT_TRIGGER_JOURNEY_CACHE (TTL 30秒)
    ↓
   缓存 miss 时查询数据库：
   SELECT * FROM journey WHERE workspaceId = ?
    ↓
2. 过滤条件
    ├─ Journey.status === "Running"
    └─ Journey.definition.entryNode.type === "EventEntryNode"
    ↓
3. 事件名匹配
    └─ doesEventNameMatch({
         pattern: journey.definition.entryNode.event,  // Journey 配置的模式
         event: triggerEvent.event,                    // 实际事件名
       })
    ↓
4. 对每个匹配的 Journey
    └─ startKeyedJourneyImpl({
         workspaceId,
         userId,
         journeyId,
         event,
         definition,
       })
```

### 5.3 事件名匹配规则

**代码位置**：`packages/isomorphic-lib/src/events.ts:1-12`

```typescript
export function doesEventNameMatch({
  pattern,  // Journey 配置的事件模式
  event,    // 实际收到的事件名
}: {
  pattern: string;
  event: string;
}): boolean {
  if (pattern.endsWith("*")) {
    // 通配符后缀匹配
    return event.startsWith(pattern.slice(0, -1));
  }
  // 精确匹配
  return pattern === event;
}
```

**示例**：
- 模式 `"user_*"` 匹配 `"user_registered"`、`"user_login"`
- 模式 `"signup"` 只匹配 `"signup"`

### 5.4 Journey 触发后检查

在启动工作流前和启动后还有多层检查：

**第一层：isRunnable 检查**
```typescript
// 检查该用户是否已运行过该 Journey
// 如果 Journey.canRunMultiple === false，且之前已运行过，则不触发
```

**第二层：工作流 ID 唯一性**
```typescript
// Keyed Journey: uuidv5(userId + eventKey + keyValue, workspaceId)
// Non-Keyed: user-journey-{userId}-{journeyId}
// 如果工作流已存在，捕获 WorkflowExecutionAlreadyStartedError 忽略
```

**第三层：工作流内去重**
```typescript
// keyedEventIds Set 跟踪已处理过的事件 messageId
// 重复事件直接忽略
```

---

## 关键代码索引

| 文件 | 功能 | 关键行号 |
|------|------|---------|
| `packages/api/src/controllers/webhooksController.ts` | Webhook 接收与验证 | 76-694 |
| `packages/backend-lib/src/destinations/sendgrid.ts` | Sendgrid 事件转换 (含回溯逻辑) | 77-215, 217-370 |
| `packages/backend-lib/src/destinations/resend.ts` | Resend 事件转换 | 66-161 |
| `packages/backend-lib/src/destinations/twilio.ts` | Twilio 事件转换 | 46-85, 113-181 |
| `packages/backend-lib/src/apps/batch.ts` | submitBatch (只入库) | 89-111 |
| `packages/backend-lib/src/apps.ts` | submitTrackWithTriggers / submitBatchWithTriggers (入库+触发) | 48-145 |
| `packages/backend-lib/src/journeys.ts` | triggerEventEntryJourneys + 事件匹配 | 688-812 |
| `packages/isomorphic-lib/src/events.ts` | doesEventNameMatch 匹配规则 | 1-12 |
| `packages/api/src/controllers/publicAppsController.ts` | 客户端 API 端点 (track/batch 会触发) | 28-288 |
| `packages/isomorphic-lib/src/types.ts` | InternalEventType 定义 | 84-107 |

---

## 总结与架构洞察

### 6.1 核心设计原则

1. **状态回调 ≠ 业务事件触发**
   - 邮件/SMS 提供商的 Webhook 是**状态反馈**，用于分析和追踪
   - 业务事件（如"用户注册"）通过**客户端 SDK Track API** 触发

2. **两条平行链路**
   ```
   ┌─────────────────────────────────────────────────────────────┐
   │  反馈链路 (不触发 Journey)                                    │
   │  Journey → 发送邮件 → 邮件提供商 → Webhook → submitBatch → 入库 │
   └─────────────────────────────────────────────────────────────┘
   
   ┌─────────────────────────────────────────────────────────────┐
   │  触发链路 (触发 Journey)                                      │
   │  客户端 → /api/public/track → submitTrackWithTriggers →       │
   │  insertUserEvents + triggerEventEntryJourneys → 新 Journey    │
   └─────────────────────────────────────────────────────────────┘
   ```

3. **三类 Webhook 的差异化设计**
   - **Sendgrid**：最复杂，需处理异步事件回溯（bounce/spam 可能丢失 metadata）
   - **Resend**：较简洁，metadata 在 `data.tags` 中可靠传递
   - **Twilio**：最特殊，所有关键信息在 **URL 参数**中（而非请求体）

4. **"WithTriggers" 版本的作用**
   - `submitBatch` = 只做持久化
   - `submit*WithTriggers` = 持久化 + 触发 Journey
   - 显式区分避免了邮件/SMS 状态回调意外触发 Journey
