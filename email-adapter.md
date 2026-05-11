# 邮件 Provider 协作方式分析报告（修订版 v2）

## 1. 概述

Dittofeed 实现了一套完整的邮件发送与事件追踪系统，支持多种邮件服务提供商（Email Provider）在统一接口下协作。

**支持的 Provider 列表：**
- SendGrid
- Amazon SES
- Resend
- PostMark
- MailChimp (Mandrill)
- SMTP
- Gmail
- Test (测试用)

核心设计理念：通过分层抽象屏蔽 Provider 差异，实现统一的发送接口和事件归一化。

---

## 2. 发送适配器：SendGrid 与 SES 的抽象差异

### 2.1 统一调度入口

邮件发送的统一入口位于 `packages/backend-lib/src/messaging.ts` 中的 `sendEmail` 函数。该函数通过 `switch-case` 模式根据 Provider 类型分发给具体适配器。

**核心机制：**
```typescript
switch (emailProvider.type) {
  case EmailProviderType.SendGrid: { ... }
  case EmailProviderType.AmazonSes: { ... }
  case EmailProviderType.Resend: { ... }
  case EmailProviderType.PostMark: { ... }
  case EmailProviderType.MailChimp: { ... }
  // ... 其他 provider
}
```

每个分支负责：
1. 构建 Provider 特有的邮件数据结构
2. 注入元数据标签（metadata/tags/customArgs）
3. 调用对应 Provider 的 `sendMail` 函数
4. 统一处理返回结果，归一化为 `BackendMessageSendResult`

### 2.2 SendGrid 适配器

**文件：** `packages/backend-lib/src/destinations/sendgrid.ts`

**发送特点：**
- 使用 `@sendgrid/mail` SDK
- 通过 `customArgs` 字段注入元数据
- SDK 直接处理邮件构建

```typescript
const mailData: MailDataRequired = {
  to, from, subject, html, ...
  customArgs: {
    workspaceId,
    templateId,
    ...messageTags,  // 包含 messageId, userId, journeyId 等
  },
};
```

**元数据传递机制：**
- SendGrid 的 `customArgs` 会随邮件发送并在 webhook 事件中回传
- 支持批量发送时每个收件人独立的 customArgs

### 2.3 Amazon SES 适配器

**文件：** `packages/backend-lib/src/destinations/amazonses.ts`

**发送特点：**
- 使用 AWS SDK v3 (`@aws-sdk/client-sesv2`)
- 使用 Raw 邮件模式（通过 `nodemailer/lib/mail-composer` 构建 MIME）
- 通过 `EmailTags` 字段注入元数据

```typescript
const input: SendEmailRequest = {
  FromEmailAddress: mailData.from,
  Destination: { ToAddresses, CcAddresses, BccAddresses },
  Content: { Raw: { Data: rawEmailContent } },
  EmailTags: tags,  // 从 mailData.tags 转换而来
};
```

**元数据传递机制：**
- SES 的 `EmailTags` 会在 SNS 通知的 `mail.tags` 字段中返回
- 标签格式为 `{ Name: string, Value: string }[]`

### 2.4 其他 Provider 对比

| Provider | 元数据字段 | 附件处理 | 特殊要求 |
|---------|-----------|---------|---------|
| **SendGrid** | `customArgs` | SDK 内置 | 需要 API Key |
| **Amazon SES** | `EmailTags` | nodemailer 构建 | 需要 AWS 凭证 |
| **Resend** | `tags` | SDK 内置 | 需要 API Key |
| **PostMark** | `Metadata` | SDK 内置 | 需要 API Key + MessageStream |
| **MailChimp** | `metadata` | SDK 内置 | 需要 API Key + `website` 字段 |
| **SMTP** | 无 | nodemailer | 需要 host/port |
| **Gmail** | 无 | Gmail API | OAuth2 access token |

### 2.5 统一返回格式

所有适配器最终返回统一的 `BackendMessageSendResult` 格式：

```typescript
// 成功时
{
  type: InternalEventType.MessageSent,
  variant: {
    type: ChannelType.Email,
    from, to, subject, body, ...
    provider: {
      type: EmailProviderType.SendGrid,  // 或其他类型
      // provider 特有字段（如 messageId）
    },
  },
}

// 失败时
{
  type: InternalEventType.MessageFailure,
  variant: {
    type: ChannelType.Email,
    provider: {
      type: EmailProviderType.SendGrid,
      name: error.name,
      message: error.message,
    },
  },
}
```

---

## 3. 投递反馈 Webhook：事件归一化

### 3.1 Webhook 入口

**文件：** `packages/api/src/controllers/webhooksController.ts`

每个 Provider 有独立的 webhook 端点：
- `/api/webhooks/sendgrid`
- `/api/webhooks/amazon-ses`
- `/api/webhooks/resend`
- `/api/webhooks/postmark`
- `/api/webhooks/mailchimp`
- `/api/webhooks/twilio` (短信)

### 3.2 统一处理流程

每个 webhook 控制器遵循相同的处理流程：

```
1. 接收请求 → 2. 签名验证 → 3. 事件提取 → 4. 格式转换 → 5. 提交批处理
```

### 3.3 SendGrid 事件归一化

**文件：** `packages/backend-lib/src/destinations/sendgrid.ts`

**核心函数：** `sendgridEventToDF()`

**事件映射表：**

| SendGrid 事件 | 内部事件 |
|--------------|---------|
| `open` | `EmailOpened` |
| `click` | `EmailClicked` |
| `bounce` | `EmailBounced` |
| `dropped` | `EmailDropped` |
| `spamreport` | `EmailMarkedSpam` |
| `delivered` | `EmailDelivered` |
| `processed` | `EmailProcessed` |

#### 3.3.1 消息 ID 生成策略（关键修正）

SendGrid 的消息 ID 生成采用**双轨策略**，不同事件类型使用不同的生成方式：

```typescript
// sendgrid.ts:102-136
let messageId: string;
switch (event) {
  case "processed":
    messageId = `processed:${smtpId}`;
    break;
  case "bounce":
    messageId = `bounce:${smtpId}`;
    break;
  case "spamreport":
    messageId = `spamreport:${smtpId}`;
    break;
  default: {
    // open, click, delivered, dropped 等事件
    messageId = uuidv5(
      `${event}:${sg_message_id}`,
      sendgridEvent.workspaceId,
    );
    break;
  }
}
```

**设计原因（代码注释说明）：**
1. `sg_message_id` 并非所有事件都存在，特别是异步事件如 bounce 和 spamreport
2. 需要能够通过 `smtp-id` 查找先前处理过的事件，当接收到异步事件时
3. `smtp-id` 也不是所有事件都存在，特别是即时事件

**两种 ID 生成方式对比：**

| 事件类型 | ID 格式 | 依赖字段 | 用途 |
|---------|---------|---------|------|
| `processed` | `processed:${smtpId}` | `smtp-id` | 作为延迟事件的锚点，存储完整元数据 |
| `bounce` | `bounce:${smtpId}` | `smtp-id` | 与 processed 事件关联 |
| `spamreport` | `spamreport:${smtpId}` | `smtp-id` | 与 processed 事件关联 |
| `open` | uuidv5(`${event}:${sg_message_id}`, workspaceId) | `sg_message_id`, `workspaceId` | 独立事件追踪 |
| `click` | uuidv5(`${event}:${sg_message_id}`, workspaceId) | `sg_message_id`, `workspaceId` | 独立事件追踪 |
| `delivered` | uuidv5(`${event}:${sg_message_id}`, workspaceId) | `sg_message_id`, `workspaceId` | 独立事件追踪 |
| `dropped` | uuidv5(`${event}:${sg_message_id}`, workspaceId) | `sg_message_id`, `workspaceId` | 独立事件追踪 |

#### 3.3.2 延迟事件回填机制

SendGrid 的 `bounce` 和 `spamreport` 事件可能延迟到达，且不包含完整的 `customArgs`（如 `workspaceId`、`userId`）。

**处理流程：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    延迟事件处理流程                              │
│                                                                 │
│  T1: 发送邮件 → SendGrid processed webhook                       │
│      └─ 生成 EmailProcessed 事件                                │
│         └─ messageId = "processed:smtp-xyz"                     │
│         └─ 包含完整元数据：workspaceId, userId, templateId 等   │
│         └─ 存储到 user_events_v2 表                             │
│                                                                 │
│  T2 (延迟): SendGrid bounce webhook                             │
│      └─ 只有 smtp-id = "smtp-xyz"                               │
│      └─ 没有 workspaceId、userId                                │
│                                                                 │
│  T3: 系统处理延迟事件                                            │
│      └─ 分离 immediateEvents 和 delayedEvents                   │
│      └─ 用 messageIds = ["processed:smtp-xyz"] 查表            │
│      └─ 找到已存储的 EmailProcessed 事件                        │
│      └─ 回填元数据到 bounce 事件                                │
│      └─ 生成 EmailBounced 事件                                  │
└─────────────────────────────────────────────────────────────────┘
```

**代码实现：**

```typescript
// sendgrid.ts:241-306
const immediateEvents: SendgridEvent[] = [];
const delayedEvents = new Map<string, SendgridEvent>();

for (const event of sendgridEvents) {
  switch (event.event) {
    case "spamreport":
    case "bounce":
      if (!event["smtp-id"]) continue;
      delayedEvents.set(event["smtp-id"], event);
      break;
    default:
      immediateEvents.push(event);
      break;
  }
}

// 查找已处理的事件来回填延迟事件的元数据
const processedForDelayedEvents = await findUserEventsById({
  messageIds: Array.from(delayedEvents.keys()).map((id) => `processed:${id}`),
});

// 回填逻辑
const backfilledDelayedEvents: SendgridEvent[] = [];
for (const event of processedForDelayedEvents) {
  const smtpId = event.message_id.split(":")[1];
  const delayedEvent = delayedEvents.get(smtpId);
  if (!delayedEvent) continue;
  
  // 从已存储事件的 properties 中解析元数据
  const parsedProperties = jsonParseSafeWithSchema(
    event.properties,
    MessageMetadataFields,
  );
  
  // 回填
  backfilledDelayedEvents.push({
    ...delayedEvent,
    ...parsedProperties.value,
  });
}
```

**关键函数：** `findUserEventsById`

```typescript
// userEvents.ts:922-976
export async function findUserEventsById({
  messageIds,
  workspaceId,
}: {
  messageIds: string[];
  workspaceId?: string;
}): Promise<UserEventsWithTraits[]> {
  // SELECT ... FROM user_events_v2 WHERE message_id IN (...)
  // 用 message_id 精确匹配查找
}
```

### 3.4 Amazon SES 事件归一化

**文件：** `packages/backend-lib/src/destinations/amazonses.ts`

**核心函数：** `submitAmazonSesEvents()`

**事件映射表：**

| SES 通知类型 | 内部事件 |
|-------------|---------|
| `Bounce` | `EmailBounced` |
| `Complaint` | `EmailMarkedSpam` |
| `Delivery` | `EmailDelivered` |
| `Open` | `EmailOpened` |
| `Click` | `EmailClicked` |

#### 3.4.1 SES 消息 ID 生成策略

与 SendGrid 不同，SES 的所有事件都使用 **统一的 uuidv5 策略**：

```typescript
// amazonses.ts:243-246
const messageId = uuidv5(
  `${event.eventType}:${event.mail.messageId}`,
  workspaceId,
);
```

**SES 消息 ID 来源：**
- `event.mail.messageId`：SES 分配的消息 ID，在 SNS 通知中返回
- 所有事件类型（Bounce, Complaint, Delivery, Open, Click）都包含此字段

#### 3.4.2 SES 元数据注入与回填链路

SES 不采用 SendGrid 那样的延迟事件回填机制，而是通过 **EmailTags 完整传递** 元数据。

**完整链路：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    SES 元数据传递链路                            │
│                                                                 │
│  发送阶段 (messaging.ts:1336-1340):                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  const mailData: SesMailData = {                          │  │
│  │    ...                                                    │  │
│  │    tags: {                                                │  │
│  │      workspaceId,                                         │  │
│  │      templateId,                                          │  │
│  │      ...messageTags,  // messageId, userId, journeyId 等 │  │
│  │    },                                                     │  │
│  │  };                                                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                          ↓                                      │
│                     SES API                                     │
│                          ↓                                      │
│  SNS 通知阶段:                                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  event.mail.tags = {                                      │  │
│  │    workspaceId: ["ws-123"],     // 注意：SES 用数组封装    │  │
│  │    userId: ["user-456"],                                  │  │
│  │    messageId: ["msg-789"],                                │  │
│  │    templateId: ["tpl-abc"],                               │  │
│  │    ...                                                    │  │
│  │  }                                                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                          ↓                                      │
│  事件处理阶段 (amazonses.ts:186-197):                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  // SES 标签转换：数组 → 标量                              │  │
│  │  let tags: Record<string, string>;                        │  │
│  │  if (event.mail.tags) {                                   │  │
│  │    const mappedTags: Record<string, string> = {};         │  │
│  │    for (const [key, values] of Object.entries(...)) {    │  │
│  │      const [value] = values;  // 取第一个值              │  │
│  │      if (value) mappedTags[key] = value;                 │  │
│  │    }                                                      │  │
│  │    tags = mappedTags;                                     │  │
│  │  }                                                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                          ↓                                      │
│  事件提交阶段 (amazonses.ts:247-272):                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  const metadataTags = R.pick(tags, MESSAGE_METADATA_FIELDS);│  │
│  │  const item: BatchTrackData = {                            │  │
│  │    type: EventType.Track,                                  │  │
│  │    event: InternalEventType.EmailOpened,                   │  │
│  │    userId: tags.userId,                                    │  │
│  │    messageId: uuidv5(..., workspaceId),                   │  │
│  │    timestamp,                                              │  │
│  │    properties: {                                           │  │
│  │      email: event.mail.destination?.[0],                   │  │
│  │      ...metadataTags,  // workspaceId, messageId 等        │  │
│  │    },                                                      │  │
│  │  };                                                        │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.4.3 SES 特殊处理：SNS 订阅确认

SES 通过 SNS（Simple Notification Service）发送通知，需要额外处理：

1. **订阅确认**：SNS 会发送 `SubscriptionConfirmation` 类型的消息，需要调用确认 URL
2. **签名验证**：使用 `sns-payload-validator` 库验证 SNS 消息签名

```typescript
// amazonses.ts:146-175
switch (body.Type) {
  case AmazonSNSEventTypes.SubscriptionConfirmation:
  case AmazonSNSEventTypes.UnsubscribeConfirmation:
    // 调用确认 URL
    const confirmed = await confirmSubscription(body);
    break;
  case AmazonSNSEventTypes.Notification:
    // 处理实际的邮件事件
    const result = await handleSesNotification(body);
    break;
}
```

### 3.5 其他 Provider 事件映射

| Provider | 事件类型 | 内部事件 | ID 生成 |
|---------|---------|---------|---------|
| **Resend** | `opened` | `EmailOpened` | uuidv5(`${event}:${email_id}`, workspaceId) |
| | `clicked` | `EmailClicked` | 同上 |
| | `bounced` | `EmailBounced` | 同上 |
| | `delivery_delayed` | `EmailDropped` | 同上 |
| | `complained` | `EmailMarkedSpam` | 同上 |
| | `delivered` | `EmailDelivered` | 同上 |
| **PostMark** | `Open` | `EmailOpened` | uuidv5(`${event}:${MessageID}`, workspaceId) |
| | `Click` | `EmailClicked` | 同上 |
| | `Bounce` | `EmailBounced` | 同上 |
| | `SpamComplaint` | `EmailMarkedSpam` | 同上 |
| | `Delivery` | `EmailDelivered` | 同上 |
| **MailChimp** | `open` | `EmailOpened` | uuidv5(`${e.event}:${e.msg._id}`, workspaceId) |
| | `click` | `EmailClicked` | 同上 |
| | `hard_bounce` | `EmailBounced` | 同上 |
| | `spam` | `EmailMarkedSpam` | 同上 |
| | `delivered` | `EmailDelivered` | 同上 |

### 3.6 统一输出格式

所有事件最终归一化为 `BatchTrackData` 格式：

```typescript
{
  type: EventType.Track,
  event: InternalEventType.EmailOpened,  // 或其他内部事件
  userId: "user-123",
  messageId: "uuid-v5-generated-id",      // 事件的唯一 ID
  timestamp: "2024-01-01T00:00:00.000Z",
  properties: {
    email: "user@example.com",
    workspaceId: "ws-123",
    journeyId: "journey-456",
    templateId: "template-789",
    messageId: "system-message-id",       // 系统消息 ID，用于关联
    // ... 其他元数据
  },
}
```

然后通过 `submitBatch()` 统一提交到事件处理管道。

---

## 4. 消息追踪链路详解

### 4.1 核心概念澄清

在深入分析之前，先澄清几个容易混淆的概念：

#### 4.1.1 什么是 `message_id`？

每个事件（无论是 MessageSent 还是 EmailOpened）都有一个 `message_id` 字段，这是**事件自身的唯一标识**。

- 存储位置：`user_events_v2.message_id`
- 生成方式：取决于事件类型
  - MessageSent：外部传入或系统生成
  - SendGrid processed/bounce/spamreport：`${event}:${smtpId}`
  - SendGrid 其他事件 / SES 等：uuidv5(`${event}:${providerMsgId}`, workspaceId)

#### 4.1.2 什么是 `origin_message_id`？

`origin_message_id` 不是发送事件天然就有的，而是**通过物化视图从 properties 中提取的系统消息 ID**。

```sql
-- userEvents/clickhouse.ts:65
JSONExtractString(properties, 'messageId') as origin_message_id
```

- 来源：`properties.messageId`（系统消息 ID）
- 用途：关联发送事件和状态事件
- 特点：所有事件（MessageSent 和状态事件）都可以有 origin_message_id

#### 4.1.3 什么是系统消息 ID？

系统消息 ID 是在**发送前生成的、与 Provider 无关的**唯一标识。

- 生成方式：`randomUUID()`
- 注入位置：messageTags.messageId
- 传递路径：
  - 发送时 → 注入到 Provider 元数据（customArgs/EmailTags/tags 等）
  - webhook 时 → 从元数据提取 → 放入 `properties.messageId`
  - 物化视图 → 提取为 `origin_message_id`

### 4.2 字段流向全景图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           字段流向全景图                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  阶段 1: 发送邮件                                                            │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  1. 生成系统消息 ID: sys-msg-001 = randomUUID()                       │  │
│  │  2. 放入 messageTags: { messageId: "sys-msg-001", ... }               │  │
│  │  3. 注入 Provider 元数据:                                             │  │
│  │     - SendGrid: customArgs.messageId = "sys-msg-001"                  │  │
│  │     - SES: EmailTags.messageId = "sys-msg-001"                        │  │
│  │     - Resend: tags.messageId = "sys-msg-001"                          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                      ↓                                       │
│  阶段 2: Provider 发送邮件                                                   │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  SendGrid 分配: sg_message_id, smtp-id                               │  │
│  │  SES 分配: event.mail.messageId                                      │  │
│  │  Resend 分配: email_id                                                │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                      ↓                                       │
│  阶段 3: Webhook 接收                                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  SendGrid webhook:                                                    │  │
│  │  ├─ customArgs.messageId = "sys-msg-001"  (回传的系统消息 ID)         │  │
│  │  └─ sg_message_id / smtp-id           (Provider 消息 ID)             │  │
│  │                                                                       │  │
│  │  SES SNS 通知:                                                        │  │
│  │  ├─ mail.tags.messageId = ["sys-msg-001"]  (回传的系统消息 ID)       │  │
│  │  └─ mail.messageId                    (Provider 消息 ID)              │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                      ↓                                       │
│  阶段 4: 事件归一化                                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  生成 BatchTrackData:                                                 │  │
│  │  {                                                                    │  │
│  │    messageId: "事件自身 ID",      ← 事件的唯一标识                    │  │
│  │    event: "DFEmailOpened",                                            │  │
│  │    properties: {                                                      │  │
│  │      messageId: "sys-msg-001",   ← 系统消息 ID，用于关联              │  │
│  │      ...                                                              │  │
│  │    }                                                                  │  │
│  │  }                                                                    │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                      ↓                                       │
│  阶段 5: 存储到 ClickHouse                                                   │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  user_events_v2 表:                                                   │  │
│  │  ├─ message_id: "事件自身 ID"                                        │  │
│  │  ├─ properties: "{ messageId: 'sys-msg-001', ... }"                  │  │
│  │  └─ event: "DFEmailOpened"                                           │  │
│  │                                                                       │  │
│  │  ↓ 物化视图 internal_events_mv                                       │  │
│  │                                                                       │  │
│  │  internal_events 表:                                                  │  │
│  │  ├─ message_id: "事件自身 ID"                                        │  │
│  │  ├─ origin_message_id: "sys-msg-001"  ← 从 properties.messageId 提取│  │
│  │  └─ event: "DFEmailOpened"                                           │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                      ↓                                       │
│  阶段 6: 投递查询                                                            │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  message_sends CTE:                                                    │  │
│  │  ├─ 筛选: event = 'DFInternalMessageSent'                            │  │
│  │  └─ 选取: message_id (发送事件的事件自身 ID)                           │  │
│  │                                                                       │  │
│  │  status_events CTE:                                                   │  │
│  │  ├─ 筛选: 状态事件 (EmailOpened, EmailClicked 等)                     │  │
│  │  └─ 选取: origin_message_id (系统消息 ID)                            │  │
│  │                                                                       │  │
│  │  关联条件:                                                            │  │
│  │  └─ ms.message_id = se.origin_message_id  ← 关键关联！               │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 关键关联机制详解

#### 4.3.1 物化视图：origin_message_id 的来源

**文件：** `packages/backend-lib/src/userEvents/clickhouse.ts`

`origin_message_id` 不是事件表原生就有的字段，而是通过物化视图**动态提取**的：

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS internal_events_mv
TO internal_events
AS SELECT
  message_id,
  event,
  -- ... 其他字段
  JSONExtractString(properties, 'messageId') as origin_message_id,  -- 关键！
  hidden
FROM user_events_v2
WHERE event_type = 'track' AND startsWith(event, 'DF');
```

**关键点：**
- `origin_message_id` = `properties.messageId`
- `properties.messageId` 是系统消息 ID
- 所有 DF 前缀的 track 事件都可以有 origin_message_id
- **发送事件天然没有 origin_message_id 字段**，是物化视图提取的

#### 4.3.2 投递查询的关联逻辑

**文件：** `packages/backend-lib/src/deliveries.ts`

投递查询使用两个 CTE（Common Table Expression）来关联发送事件和状态事件：

```sql
-- CTE 1: message_sends - 发送事件
WITH message_sends AS (
  SELECT
    workspace_id,
    user_or_anonymous_id,
    processing_time,
    message_id,           -- 发送事件的事件自身 ID
    event_time,
    triggering_message_id
  FROM internal_events
  WHERE
    event = 'DFInternalMessageSent'  -- 筛选发送事件
    AND ... 其他过滤条件
),

-- CTE 2: status_events - 状态事件
status_events AS (
  SELECT
    workspace_id,
    user_or_anonymous_id,
    origin_message_id,    -- 从 properties.messageId 提取的系统消息 ID
    argMax(event, event_time) as last_event,
    max(event_time) as max_event_time
  FROM internal_events
  WHERE
    ... 状态事件过滤条件
    AND origin_message_id IN (
      SELECT message_id FROM message_sends  -- 只关联有发送事件的状态
    )
  GROUP BY workspace_id, user_or_anonymous_id, origin_message_id
)

-- 最终关联
FROM message_sends ms
LEFT JOIN status_events se ON
  ms.workspace_id = se.workspace_id
  AND ms.message_id = se.origin_message_id  -- 关键关联！
  AND ms.user_or_anonymous_id = se.user_or_anonymous_id
```

**关联逻辑解析：**

| 来源 | 字段 | 含义 |
|-----|------|------|
| message_sends (ms) | `message_id` | 发送事件的**事件自身 ID** |
| status_events (se) | `origin_message_id` | 状态事件的**系统消息 ID**（从 properties.messageId 提取） |
| 关联条件 | `ms.message_id = se.origin_message_id` | 发送事件的事件 ID = 状态事件的系统消息 ID |

**这意味着：**
- MessageSent 事件的 `message_id`（事件自身 ID）必须等于系统消息 ID
- 状态事件的 `origin_message_id` 也等于系统消息 ID
- 因此可以通过系统消息 ID 关联

### 4.4 完整示例：从发送到投递查询

让我们通过一个完整的示例来理解整个字段流向。

#### 4.4.1 场景设定

- 工作区 ID：`ws-123`
- 用户 ID：`user-456`
- 模板 ID：`tpl-789`
- Provider：先使用 SendGrid，后切换到 SES

#### 4.4.2 阶段 1：使用 SendGrid 发送邮件 M1

**步骤 1：生成系统消息 ID**

```typescript
// messaging.ts:2443-2446
const messageTags: MessageTags = {
  messageId: randomUUID(),  // 生成：sys-msg-001
  workspaceId: "ws-123",
  templateId: "tpl-789",
  userId: "user-456",
  // ... 其他字段
};
```

**步骤 2：注入到 SendGrid customArgs**

```typescript
// messaging.ts:1262-1266
const mailData: MailDataRequired = {
  to: "user@example.com",
  from: "sender@example.com",
  subject: "Test Email",
  html: "<div>Hello</div>",
  customArgs: {
    workspaceId: "ws-123",
    templateId: "tpl-789",
    ...messageTags,  // messageId: "sys-msg-001"
  },
};
```

**步骤 3：MessageSent 事件存储**

```typescript
// 提交到系统
{
  messageId: "sys-msg-001",           // 事件自身 ID = 系统消息 ID
  type: EventType.Track,
  event: "DFInternalMessageSent",
  userId: "user-456",
  properties: {
    messageId: "sys-msg-001",         // 系统消息 ID（重复一份，用于关联）
    workspaceId: "ws-123",
    templateId: "tpl-789",
    journeyId: "jrn-abc",
    variant: {
      type: "Email",
      from: "sender@example.com",
      to: "user@example.com",
      provider: { type: "SendGrid" }
    }
  },
  timestamp: "2024-01-01T10:00:00.000Z",
}
```

**存储到 user_events_v2：**

| message_id | event | properties |
|-----------|-------|-----------|
| sys-msg-001 | DFInternalMessageSent | { messageId: "sys-msg-001", ... } |

**物化到 internal_events：**

| message_id | event | origin_message_id |
|-----------|-------|------------------|
| sys-msg-001 | DFInternalMessageSent | sys-msg-001 |

#### 4.4.3 阶段 2：SendGrid processed webhook

SendGrid 接收邮件后发送 processed 事件：

```typescript
// SendGrid webhook 数据
{
  event: "processed",
  email: "user@example.com",
  "smtp-id": "smtp-xyz-123",
  sg_message_id: "sg-abc-456",
  customArgs: {
    workspaceId: "ws-123",
    messageId: "sys-msg-001",
    userId: "user-456",
    templateId: "tpl-789"
  },
  timestamp: 1704099610  // 10:00:10
}
```

**归一化为 BatchTrackData：**

```typescript
// sendgrid.ts:102-136
// processed 事件使用 smtp-id 前缀
messageId = `processed:${smtpId}` = "processed:smtp-xyz-123"

// 提取 customArgs 到 properties
{
  messageId: "processed:smtp-xyz-123",
  type: EventType.Track,
  event: "DFEmailProcessed",
  userId: "user-456",
  properties: {
    messageId: "sys-msg-001",         // 系统消息 ID
    workspaceId: "ws-123",
    userId: "user-456",
    templateId: "tpl-789",
    smtpId: "smtp-xyz-123"
  },
  timestamp: "2024-01-01T10:00:10.000Z",
}
```

**存储到 user_events_v2：**

| message_id | event | properties |
|-----------|-------|-----------|
| processed:smtp-xyz-123 | DFEmailProcessed | { messageId: "sys-msg-001", ... } |

**物化到 internal_events：**

| message_id | event | origin_message_id |
|-----------|-------|------------------|
| processed:smtp-xyz-123 | DFEmailProcessed | sys-msg-001 |

#### 4.4.4 阶段 3：SendGrid delivered webhook

收件人收到邮件后，SendGrid 发送 delivered 事件：

```typescript
// SendGrid webhook 数据
{
  event: "delivered",
  email: "user@example.com",
  sg_message_id: "sg-abc-456",
  customArgs: {
    workspaceId: "ws-123",
    messageId: "sys-msg-001",
    userId: "user-456"
  },
  timestamp: 1704100210  // 10:10:10
}
```

**归一化为 BatchTrackData：**

```typescript
// sendgrid.ts:130-133
// delivered 事件使用 uuidv5
messageId = uuidv5(
  "delivered:sg-abc-456",
  "ws-123"
) = "uuidv5-result-789"

// 提交
{
  messageId: "uuidv5-result-789",
  type: EventType.Track,
  event: "DFEmailDelivered",
  userId: "user-456",
  properties: {
    messageId: "sys-msg-001",         // 系统消息 ID
    workspaceId: "ws-123",
    ...
  },
  timestamp: "2024-01-01T10:10:10.000Z",
}
```

**存储到 user_events_v2：**

| message_id | event | properties |
|-----------|-------|-----------|
| uuidv5-result-789 | DFEmailDelivered | { messageId: "sys-msg-001", ... } |

**物化到 internal_events：**

| message_id | event | origin_message_id |
|-----------|-------|------------------|
| uuidv5-result-789 | DFEmailDelivered | sys-msg-001 |

#### 4.4.5 阶段 4：切换 Provider 到 SES

```
时间线推进，工作区配置更新：
  Provider: SendGrid → SES
```

#### 4.4.6 阶段 5：使用 SES 发送邮件 M2

**步骤 1：生成新的系统消息 ID**

```typescript
const messageTags: MessageTags = {
  messageId: randomUUID(),  // 生成：sys-msg-002（新的！）
  workspaceId: "ws-123",
  templateId: "tpl-789",
  userId: "user-456",
};
```

**步骤 2：注入到 SES EmailTags**

```typescript
const mailData: SesMailData = {
  to: "user@example.com",
  from: "sender@example.com",
  tags: {
    workspaceId: "ws-123",
    templateId: "tpl-789",
    ...messageTags,  // messageId: "sys-msg-002"
  },
};
```

**步骤 3：MessageSent 事件存储**

```typescript
{
  messageId: "sys-msg-002",           // 事件自身 ID = 新的系统消息 ID
  event: "DFInternalMessageSent",
  properties: {
    messageId: "sys-msg-002",         // 系统消息 ID
    ...
  },
}
```

**存储到 internal_events：**

| message_id | event | origin_message_id |
|-----------|-------|------------------|
| sys-msg-002 | DFInternalMessageSent | sys-msg-002 |

#### 4.4.7 阶段 6：SES delivered webhook

```typescript
// SES SNS 通知
{
  notificationType: "Delivery",
  mail: {
    messageId: "ses-def-789",
    destination: ["user@example.com"],
    tags: {
      workspaceId: ["ws-123"],
      messageId: ["sys-msg-002"],
      userId: ["user-456"]
    }
  }
}
```

**归一化为 BatchTrackData：**

```typescript
// amazonses.ts:243-246
messageId = uuidv5(
  "Delivery:ses-def-789",
  "ws-123"
) = "uuidv5-result-abc"

// 提交
{
  messageId: "uuidv5-result-abc",
  event: "DFEmailDelivered",
  properties: {
    messageId: "sys-msg-002",         // 系统消息 ID（新的！）
    ...
  },
}
```

**存储到 internal_events：**

| message_id | event | origin_message_id |
|-----------|-------|------------------|
| uuidv5-result-abc | DFEmailDelivered | sys-msg-002 |

#### 4.4.8 阶段 7：SendGrid 延迟 bounce webhook

几天后，SendGrid 发送延迟的 bounce 事件：

```typescript
// SendGrid webhook 数据（延迟）
{
  event: "bounce",
  email: "user@example.com",
  "smtp-id": "smtp-xyz-123",
  // 注意：没有 customArgs！没有 workspaceId、userId
  timestamp: 1704358810  // 3 天后
}
```

**延迟事件回填处理：**

```typescript
// sendgrid.ts:241-306
// 1. 分离延迟事件
delayedEvents.set("smtp-xyz-123", bounceEvent);

// 2. 查找已存储的 processed 事件
const processedEvents = await findUserEventsById({
  messageIds: ["processed:smtp-xyz-123"],
});

// 3. 找到 EmailProcessed 事件，回填元数据
// processed 事件的 properties 包含：
// {
//   messageId: "sys-msg-001",
//   workspaceId: "ws-123",
//   userId: "user-456",
//   templateId: "tpl-789"
// }

// 4. 生成 EmailBounced 事件
messageId = `bounce:${smtpId}` = "bounce:smtp-xyz-123"

{
  messageId: "bounce:smtp-xyz-123",
  event: "DFEmailBounced",
  userId: "user-456",                    // 回填的
  properties: {
    messageId: "sys-msg-001",           // 回填的系统消息 ID！
    workspaceId: "ws-123",              // 回填的
    ...
  },
}
```

**存储到 internal_events：**

| message_id | event | origin_message_id |
|-----------|-------|------------------|
| bounce:smtp-xyz-123 | DFEmailBounced | sys-msg-001 |

#### 4.4.9 阶段 8：投递查询

现在查询投递状态，看看 Provider 切换后是否能正确关联：

```sql
-- message_sends CTE: 筛选发送事件
SELECT message_id
FROM internal_events
WHERE event = 'DFInternalMessageSent'
AND workspace_id = 'ws-123'

-- 结果：
-- message_id
-- sys-msg-001
-- sys-msg-002

-- status_events CTE: 筛选状态事件
SELECT origin_message_id, argMax(event, event_time) as last_event
FROM internal_events
WHERE event IN ('DFEmailDelivered', 'DFEmailBounced', ...)
AND origin_message_id IN ('sys-msg-001', 'sys-msg-002')
GROUP BY origin_message_id

-- 结果：
-- origin_message_id | last_event
-- sys-msg-001       | DFEmailBounced  （延迟事件正确关联！）
-- sys-msg-002       | DFEmailDelivered

-- 最终关联
FROM message_sends ms
LEFT JOIN status_events se ON
  ms.workspace_id = se.workspace_id
  AND ms.message_id = se.origin_message_id  -- 关键关联！

-- 关联结果：
-- ms.message_id = sys-msg-001  →  se.origin_message_id = sys-msg-001  ✓ 匹配！
-- ms.message_id = sys-msg-002  →  se.origin_message_id = sys-msg-002  ✓ 匹配！
```

**最终查询结果：**

| message_id (ms) | origin_message_id (se) | last_event |
|----------------|----------------------|------------|
| sys-msg-001 | sys-msg-001 | DFEmailBounced |
| sys-msg-002 | sys-msg-002 | DFEmailDelivered |

**🎉 Provider 切换后，历史数据仍然正确关联！**

### 4.5 为什么 Provider 切换后还能关联？

通过上面的示例，我们可以清晰地看到 **3 个关键设计**确保了连续性：

#### 设计 1：系统消息 ID 独立于 Provider

```
发送 M1 (SendGrid):
  系统消息 ID = randomUUID() → sys-msg-001
  注入到 customArgs → webhook 回传 → properties.messageId
  物化视图 → origin_message_id = sys-msg-001

发送 M2 (SES):
  系统消息 ID = randomUUID() → sys-msg-002
  注入到 EmailTags → SNS 通知 → properties.messageId
  物化视图 → origin_message_id = sys-msg-002

连续性保证：
  ✓ 生成逻辑不变（始终是 randomUUID()）
  ✓ 存储路径不变（始终是 properties.messageId → origin_message_id）
  ✓ 与 Provider 分配的 ID 完全解耦
```

#### 设计 2：延迟事件回填（SendGrid 特有）

```
T1 (SendGrid):
  发送 M1 → 系统消息 ID = sys-msg-001
  processed webhook → 存储 EmailProcessed 事件
    → properties.messageId = sys-msg-001

T2 (切换到 SES):
  配置更新

T3 (延迟 bounce):
  bounce webhook 只有 smtp-id，没有元数据
  系统查找 message_id = "processed:smtp-xyz" 的事件
  找到 EmailProcessed 事件
  从 properties 提取：
    messageId = sys-msg-001
    workspaceId = ws-123
    userId = user-456
  回填到 bounce 事件
  EmailBounced 事件的 properties.messageId = sys-msg-001

结果：
  ✓ 即使 Provider 切换了
  ✓ 延迟事件仍然能正确关联到历史发送
  ✓ origin_message_id = sys-msg-001
```

#### 设计 3：关联键是系统消息 ID，不是 Provider ID

```
关联条件: ms.message_id = se.origin_message_id

左边 ms.message_id:
  是 MessageSent 事件的事件自身 ID
  等于系统消息 ID

右边 se.origin_message_id:
  是状态事件的系统消息 ID
  从 properties.messageId 提取
  properties.messageId 来自 Provider 元数据
  Provider 元数据来自发送时注入的系统消息 ID

结果：
  ✓ 两边都是系统消息 ID
  ✓ 与 Provider 分配的 ID（sg_message_id, SES messageId）无关
  ✓ Provider 切换不影响关联
```

---

## 5. Provider 切换时的连续性保证

### 5.1 场景分析

假设同一工作区先使用 SendGrid 发送邮件，后切换到 SES：

```
时间线：
T0: 配置 Provider 为 SendGrid

T1: 发送邮件 M1（SendGrid）
    ├─ 系统消息 ID: sys-msg-001
    ├─ SendGrid sg_message_id: sg-abc
    ├─ SendGrid smtp-id: smtp-xyz
    │
    ├─ 生成 MessageSent 事件
    │   └─ message_id (事件自身 ID): sys-msg-001
    │   └─ properties.messageId: sys-msg-001
    │   └─ origin_message_id (物化后): sys-msg-001
    │
    ├─ SendGrid processed webhook
    │   └─ 生成 EmailProcessed 事件
    │       └─ message_id (事件自身 ID): "processed:smtp-xyz"
    │       └─ properties.messageId: sys-msg-001
    │       └─ origin_message_id (物化后): sys-msg-001
    │
    └─ SendGrid delivered webhook
        └─ 生成 EmailDelivered 事件
            └─ message_id (事件自身 ID): uuidv5("delivered:sg-abc", ws-id)
            └─ properties.messageId: sys-msg-001
            └─ origin_message_id (物化后): sys-msg-001

T2: 切换 Provider 到 SES
    └─ 配置更新为 SES

T3: 发送邮件 M2（SES）
    ├─ 系统消息 ID: sys-msg-002
    ├─ SES messageId: ses-def
    │
    ├─ 生成 MessageSent 事件
    │   └─ message_id (事件自身 ID): sys-msg-002
    │   └─ properties.messageId: sys-msg-002
    │   └─ origin_message_id (物化后): sys-msg-002
    │
    └─ SES delivered webhook
        └─ 生成 EmailDelivered 事件
            └─ message_id (事件自身 ID): uuidv5("Delivery:ses-def", ws-id)
            └─ properties.messageId: sys-msg-002
            └─ origin_message_id (物化后): sys-msg-002

T4: SendGrid 延迟发送 bounce 事件
    └─ 只有 smtp-id: smtp-xyz
    └─ 系统查找 "processed:smtp-xyz" 事件
    └─ 找到 EmailProcessed 事件，回填元数据
    └─ 生成 EmailBounced 事件
        └─ message_id (事件自身 ID): "bounce:smtp-xyz"
        └─ properties.messageId: sys-msg-001
        └─ origin_message_id (物化后): sys-msg-001
```

### 5.2 连续性保证的三个关键点

**关键点 1：系统消息 ID 独立于 Provider**

```
Provider 切换前 (SendGrid):
  系统消息 ID = randomUUID() → sys-msg-001
  注入到 customArgs.messageId → webhook 回传 → properties.messageId
  物化视图提取 → origin_message_id = sys-msg-001

Provider 切换后 (SES):
  系统消息 ID = randomUUID() → sys-msg-002
  注入到 EmailTags.messageId → SNS 通知回传 → properties.messageId
  物化视图提取 → origin_message_id = sys-msg-002

连续性保证：
  - 生成逻辑不变
  - 存储位置不变（properties.messageId → origin_message_id）
  - 关联查询逻辑不变
```

**关键点 2：延迟事件回填（SendGrid 特有）**

```
T4 时，SendGrid 延迟发送 bounce 事件：
  bounce 事件只有 smtp-id = smtp-xyz
  没有 workspaceId、userId、messageId

系统处理：
  1. 查找 message_id = "processed:smtp-xyz" 的事件
  2. 找到 EmailProcessed 事件
  3. 从 properties 中提取：
     - workspaceId
     - userId
     - messageId (系统消息 ID: sys-msg-001)
     - templateId
     - journeyId
  4. 回填到 bounce 事件
  5. 生成 EmailBounced 事件，properties.messageId = sys-msg-001

结果：
  - 即使延迟到 Provider 切换后
  - 仍然能正确关联到 T1 发送的邮件
  - origin_message_id = sys-msg-001
  - 投递查询能正确关联 MessageSent 和 EmailBounced
```

**关键点 3：origin_message_id 是系统消息 ID，不是 Provider ID**

```
错误假设：
  origin_message_id 是 Provider 分配的 ID（sg_message_id 或 SES messageId）

实际情况：
  origin_message_id 是从 properties.messageId 提取的
  properties.messageId 是系统生成的（randomUUID()）
  与 Provider 完全无关

Provider 切换影响：
  ✅ 不影响系统消息 ID 的生成
  ✅ 不影响 origin_message_id 的提取
  ✅ 不影响投递查询的关联逻辑
  ✅ 不影响历史数据的查询
```

### 5.3 连续性对比：SendGrid vs SES

| 维度 | SendGrid | SES |
|-----|---------|-----|
| **系统消息 ID 注入** | `customArgs.messageId` | `EmailTags.messageId` |
| **系统消息 ID 回传** | webhook 事件 customArgs | SNS 通知 mail.tags |
| **事件 ID 生成** | processed/bounce/spamreport: `${event}:${smtpId}`<br>其他事件: uuidv5 | 所有事件: uuidv5(`${event}:${mail.messageId}`, workspaceId) |
| **延迟事件处理** | 需要回填（通过 `processed:${smtpId}` 查找） | 不需要（tags 完整传递） |
| **origin_message_id** | 从 properties.messageId 提取（系统消息 ID） | 从 properties.messageId 提取（系统消息 ID） |
| **Provider 切换影响** | 无（系统消息 ID 独立） | 无（系统消息 ID 独立） |

### 5.4 UUID v5 详解

所有 Provider（SendGrid 部分事件除外）都使用 `uuid v5` 算法生成确定性的内部事件 ID：

```typescript
// 通用模式
messageId = uuidv5(
  `${eventType}:${providerMessageId}`,  // 名称
  workspaceId,                           // 命名空间
);
```

**UUID v5 的特点：**
- 基于命名空间（namespace）和名称（name）生成
- 相同输入永远产生相同输出
- 无需存储映射关系即可追溯
- 输入变化则输出完全不同

**这意味着：**
- 同一工作区 + 同一事件类型 + 同一 Provider 消息 ID → 同一事件 ID
- 即使处理多次，也不会产生重复事件
- 但切换 Provider 后，Provider 消息 ID 变了，事件 ID 也会变
- **但 origin_message_id（系统消息 ID）保持不变，所以关联关系不变**

---

## 6. 架构总结

### 6.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        应用层 (Application)                         │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐ │
│  │   旅程引擎       │    │  广播发送器      │    │   预览发送器     │ │
│  │  (Journey)      │    │ (Broadcast)     │    │   (Preview)     │ │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘ │
│           │                      │                      │          │
└───────────┼──────────────────────┼──────────────────────┼──────────┘
            │                      │                      │
            ▼                      ▼                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    统一发送层 (sendEmail)                            │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │        messageTags.messageId = randomUUID()                  │  │
│  │        （系统消息 ID，与 Provider 无关）                      │  │
│  │                                                              │  │
│  │           switch(emailProvider.type) 分发                     │  │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐    │  │
│  │  │SendGrid│ │Amazon  │ │ Resend │ │PostMark│ │MailChimp│   │  │
│  │  │customA │ │EmailTags│ │  tags  │ │Metadata│ │metadata │   │  │
│  │  └────┬───┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘    │  │
│  │       │          │           │          │          │         │  │
│  └───────┼──────────┼───────────┼──────────┼──────────┼─────────┘  │
│          │          │           │          │          │            │
└──────────┼──────────┼───────────┼──────────┼──────────┼────────────┘
           │          │           │          │          │
           ▼          ▼           ▼          ▼          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Provider SDK / API                               │
│  SendGrid API  Amazon SES API  Resend API  PostMark API  Mandrill   │
└─────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ 邮件投递
                                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       收件人邮箱                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │   Gmail      │  │   Outlook    │  │   企业邮箱    │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ 用户行为 / 投递状态
                                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Provider Webhook                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  /sendgrid  /amazon-ses  /resend  /postmark  /mailchimp      │  │
│  └───────────────┬─────────────┬─────────┬──────────┬───────────┘  │
│                  │             │         │          │               │
└──────────────────┼─────────────┼─────────┼──────────┼───────────────┘
                   │             │         │          │
                   ▼             ▼         ▼          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 事件归一化层 (Event Normalization)                   │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  SendGrid:                                                   │  │
│  │    - processed/bounce/spamreport: `${event}:${smtpId}`      │  │
│  │    - 其他事件: uuidv5(`${event}:${sg_message_id}`, wsId)     │  │
│  │    - 延迟事件回填（通过 processed:smtp-id 查找）              │  │
│  │                                                              │  │
│  │  SES / Resend / PostMark / MailChimp:                       │  │
│  │    - 所有事件: uuidv5(`${event}:${providerMsgId}`, wsId)    │  │
│  │    - 元数据完整传递，无需回填                                 │  │
│  │                                                              │  │
│  │  所有事件都包含: properties.messageId = 系统消息 ID          │  │
│  └──────────────────────────────┬───────────────────────────────┘  │
│                                 │                                   │
│                                 ▼                                   │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              BatchTrackData (统一格式)                         │  │
│  │  {                                                           │  │
│  │    type: EventType.Track,                                    │  │
│  │    event: InternalEventType.EmailOpened,                     │  │
│  │    userId: "user-123",                                       │  │
│  │    messageId: "事件唯一 ID",                                  │  │
│  │    timestamp: "...",                                         │  │
│  │    properties: {                                             │  │
│  │      messageId: "系统消息 ID",  ← 关键！用于关联            │  │
│  │      workspaceId: "ws-123",                                  │  │
│  │      ...                                                     │  │
│  │    }                                                         │  │
│  │  }                                                           │  │
│  └──────────────────────────────┬───────────────────────────────┘  │
│                                 │                                   │
└─────────────────────────────────┼───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      事件存储层 (Event Storage)                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  user_events_v2 (主表)                                        │  │
│  │    message_id: 事件 ID                                        │  │
│  │    properties: JSON（包含系统消息 ID）                        │  │
│  │                                                              │  │
│  │  ↓ 物化视图 internal_events_mv                               │  │
│  │                                                              │  │
│  │  internal_events (物化表)                                    │  │
│  │    message_id: 事件 ID                                        │  │
│  │    origin_message_id: 从 properties.messageId 提取            │  │
│  │                   ← 系统消息 ID，用于关联                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    投递查询层 (Delivery Query)                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  message_sends CTE:                                          │  │
│  │    筛选: event = 'DFInternalMessageSent'                     │  │
│  │    选取: message_id (发送事件的事件 ID)                      │  │
│  │                                                              │  │
│  │  status_events CTE:                                          │  │
│  │    筛选: 状态事件                                            │  │
│  │    选取: origin_message_id (系统消息 ID)                     │  │
│  │                                                              │  │
│  │  关联条件:                                                   │  │
│  │    ms.message_id = se.origin_message_id                      │  │
│  │    ↓ 实际上是:                                               │  │
│  │    发送事件的事件 ID = 状态事件的系统消息 ID                  │  │
│  │                                                              │  │
│  │  结果: 发送事件和状态事件正确关联                            │  │
│  │       无论使用哪个 Provider                                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 关键设计原则

1. **分层抽象**
   - 发送层：统一的 `sendEmail` 入口，屏蔽 Provider 差异
   - 事件层：统一的 `submitBatch` 出口，归一化事件格式

2. **双轨 ID 策略**
   - **系统消息 ID**：`randomUUID()`，发送前确定，与 Provider 无关
   - **事件 ID**：
     - SendGrid processed/bounce/spamreport：`${event}:${smtpId}`
     - 其他事件 / 其他 Provider：`uuidv5()`，基于 Provider 消息 ID 和 workspaceId

3. **元数据贯通**
   - 发送时注入：通过 customArgs/tags/metadata 传递
   - 接收时提取：从 webhook 事件中恢复到 properties.messageId
   - 延迟事件（SendGrid）：通过已存储事件回填缺失元数据

4. **关联独立性**
   - `origin_message_id` 从 `properties.messageId` 提取（系统消息 ID）
   - 与 Provider 分配的 ID 完全解耦
   - Provider 切换不影响历史数据的追踪连续性

5. **物化视图提取**
   - `origin_message_id` 不是事件表原生字段
   - 通过物化视图从 `properties.messageId` 动态提取
   - 所有 DF 前缀的 track 事件都可以有 origin_message_id

6. **签名验证**
   - 每个 webhook 端点都有独立的签名验证机制
   - 防止伪造的事件注入

### 6.3 文件索引

| 功能 | 文件路径 |
|-----|---------|
| 统一发送入口 | `packages/backend-lib/src/messaging.ts` |
| SendGrid 适配器 | `packages/backend-lib/src/destinations/sendgrid.ts` |
| Amazon SES 适配器 | `packages/backend-lib/src/destinations/amazonses.ts` |
| Resend 适配器 | `packages/backend-lib/src/destinations/resend.ts` |
| PostMark 适配器 | `packages/backend-lib/src/destinations/postmark.ts` |
| MailChimp 适配器 | `packages/backend-lib/src/destinations/mailchimp.ts` |
| Webhook 控制器 | `packages/api/src/controllers/webhooksController.ts` |
| 常量定义 | `packages/backend-lib/src/constants.ts` |
| 事件查找 | `packages/backend-lib/src/userEvents.ts` (findUserEventsById) |
| ClickHouse 表定义 | `packages/backend-lib/src/userEvents/clickhouse.ts` |
| 投递查询 | `packages/backend-lib/src/deliveries.ts` |
| 类型定义 | `packages/isomorphic-lib/src/types.ts` |
| 批量提交 | `packages/backend-lib/src/apps/batch.ts` |

### 6.4 核心代码位置速查

**SendGrid 事件 ID 双轨策略：**
- 位置：`packages/backend-lib/src/destinations/sendgrid.ts:102-136`
- 关键：processed/bounce/spamreport 使用 `smtp-id` 前缀，其他事件使用 uuidv5

**SendGrid 延迟事件回填：**
- 位置：`packages/backend-lib/src/destinations/sendgrid.ts:241-306`
- 关键：通过 `findUserEventsById` 查找 `processed:${smtpId}` 事件

**SES 元数据完整传递：**
- 发送注入：`packages/backend-lib/src/messaging.ts:1336-1340`
- webhook 处理：`packages/backend-lib/src/destinations/amazonses.ts:186-197`
- 关键：`EmailTags` → `mail.tags` → `properties.messageId`

**origin_message_id 提取：**
- 位置：`packages/backend-lib/src/userEvents/clickhouse.ts:65`
- 关键：`JSONExtractString(properties, 'messageId') as origin_message_id`

**投递查询关联：**
- 位置：