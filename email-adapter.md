# 邮件 Provider 协作方式分析报告

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

**延迟事件处理（关键机制）：**

SendGrid 的 `bounce` 和 `spamreport` 事件可能延迟到达，且不包含完整的 `customArgs`（如 `workspaceId`、`userId`）。

处理策略：
1. 分离即时事件（immediateEvents）和延迟事件（delayedEvents）
2. 延迟事件通过 `smtp-id` 查找已存储的 `processed` 事件
3. 从已存储事件中回填缺失的元数据

```typescript
// sendgrid.ts:241-306
const immediateEvents: SendgridEvent[] = [];
const delayedEvents = new Map<string, SendgridEvent>();

for (const event of sendgridEvents) {
  switch (event.event) {
    case "spamreport":
    case "bounce":
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

**特殊处理：**

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

| Provider | 事件类型 | 内部事件 |
|---------|---------|---------|
| **Resend** | `opened` | `EmailOpened` |
| | `clicked` | `EmailClicked` |
| | `bounced` | `EmailBounced` |
| | `delivery_delayed` | `EmailDropped` |
| | `complained` | `EmailMarkedSpam` |
| | `delivered` | `EmailDelivered` |
| **PostMark** | `Open` | `EmailOpened` |
| | `Click` | `EmailClicked` |
| | `Bounce` | `EmailBounced` |
| | `SpamComplaint` | `EmailMarkedSpam` |
| | `Delivery` | `EmailDelivered` |
| **MailChimp** | `open` | `EmailOpened` |
| | `click` | `EmailClicked` |
| | `hard_bounce` | `EmailBounced` |
| | `spam` | `EmailMarkedSpam` |
| | `delivered` | `EmailDelivered` |

### 3.6 统一输出格式

所有事件最终归一化为 `BatchTrackData` 格式：

```typescript
{
  type: EventType.Track,
  event: InternalEventType.EmailOpened,  // 或其他内部事件
  userId: "user-123",
  messageId: "uuid-v5-generated-id",
  timestamp: "2024-01-01T00:00:00.000Z",
  properties: {
    email: "user@example.com",
    workspaceId: "ws-123",
    journeyId: "journey-456",
    templateId: "template-789",
    // ... 其他元数据
  },
}
```

然后通过 `submitBatch()` 统一提交到事件处理管道。

---

## 4. 消息标识连续性：Provider 切换时的 ID 管理

### 4.1 三层 ID 体系

系统采用三层 ID 体系确保消息标识的连续性：

```
┌─────────────────────────────────────────────────────────┐
│                    第一层：系统消息 ID                    │
│              randomUUID() 生成，发送前确定                │
│              messageId（在 messageTags 中传递）           │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                    第二层：Provider 消息 ID               │
│        Provider 分配的 ID（如 sg_message_id、SES messageId）│
│         用于 webhook 事件与发送邮件的关联                  │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                    第三层：内部事件 ID                    │
│   uuidv5(`${eventType}:${providerMessageId}`, workspaceId)│
│   确定性生成，确保同一消息+同一事件+同一工作区产生相同 ID    │
└─────────────────────────────────────────────────────────┘
```

### 4.2 系统消息 ID 生成

**文件：** `packages/backend-lib/src/messaging.ts`

系统消息 ID 在邮件发送前由系统生成，与 Provider 无关：

```typescript
// messaging.ts:2443-2446（预览发送）
const messageTags: MessageTags = {
  ...(request.tags ?? {}),
  messageId: request.tags?.messageId ?? randomUUID(),
};

// messaging.ts:2589-2629（批量发送）
const messagePromises = users.map(async (user) => {
  const messageId = user.messageId ?? randomUUID();
  // ...
  const messageTags: MessageTags = {
    messageId,
    workspaceId,
    templateId,
    userId: user.id,
  };
});
```

**关键点：**
- 使用 Node.js 内置的 `crypto.randomUUID()` 生成
- 如果调用方已提供 messageId，则使用提供的 ID
- messageId 作为元数据的一部分注入到邮件中

### 4.3 元数据注入机制

系统消息 ID 通过各 Provider 特有的元数据字段注入：

| Provider | 注入字段 | 代码位置 |
|---------|---------|---------|
| SendGrid | `customArgs` | messaging.ts:1262-1266 |
| Amazon SES | `EmailTags` | messaging.ts:1336-1340 |
| Resend | `tags` | messaging.ts:1499-1501 |
| PostMark | `Metadata` | messaging.ts:1572-1593 |
| MailChimp | `metadata` | messaging.ts:1663-1674 |

**统一元数据字段（MESSAGE_METADATA_FIELDS）：**

**文件：** `packages/backend-lib/src/constants.ts`

```typescript
export const MESSAGE_METADATA_FIELDS = [
  "workspaceId",
  "broadcastId",
  "journeyId",
  "runId",
  "messageId",
  "userId",
  "templateId",
  "nodeId",
] as const;
```

### 4.4 内部事件 ID 的确定性生成

所有 Provider 的事件处理都使用 `uuid v5` 算法生成确定性的内部事件 ID：

```typescript
// SendGrid: sendgrid.ts:130-133
messageId = uuidv5(
  `${event}:${sg_message_id}`,
  sendgridEvent.workspaceId,
);

// Amazon SES: amazonses.ts:243-246
const messageId = uuidv5(
  `${event.eventType}:${event.mail.messageId}`,
  workspaceId,
);

// Resend: resend.ts:84
const messageId = uuidv5(`${event}:${email_id}`, workspaceId);

// PostMark: postmark.ts:112
const messageId = uuidv5(`${event}:${postMarkEvent.MessageID}`, workspaceId);

// MailChimp: mailchimp.ts:147
const messageId = uuidv5(`${e.event}:${e.msg._id}`, workspaceId);
```

**UUID v5 的特点：**
- 基于命名空间（workspaceId）和名称（`事件类型:Provider消息ID`）生成
- 相同输入永远产生相同输出
- 无需存储映射关系即可追溯

### 4.5 Provider 切换时的连续性保证

**场景分析：** 假设同一工作区先使用 SendGrid 发送邮件，后切换到 SES。

**连续性保证机制：**

1. **系统消息 ID 独立于 Provider**
   - 系统消息 ID 在发送前生成，不依赖 Provider
   - 即使切换 Provider，系统消息 ID 的生成逻辑不变

2. **内部事件 ID 基于 workspaceId**
   - 内部事件 ID 使用 workspaceId 作为命名空间
   - 只要工作区不变，命名空间不变

3. **延迟事件的回填策略（SendGrid 特有）**
   - 延迟事件通过 `smtp-id` 查找已存储的 `processed` 事件
   - 确保即使事件延迟到达，也能关联到正确的消息元数据

**示例场景：**

```
时间线：
T1: 发送邮件 M1（使用 SendGrid）
    - 系统消息 ID: msg-001
    - sg_message_id: sg-abc
    - smtp-id: smtp-xyz
    - 生成 MessageSent 事件（messageId = msg-001）
    - 生成 EmailProcessed 事件（messageId = processed:smtp-xyz）

T2: 切换 Provider 到 SES
    - 配置更新为 SES

T3: 发送邮件 M2（使用 SES）
    - 系统消息 ID: msg-002
    - SES messageId: ses-def
    - 生成 MessageSent 事件（messageId = msg-002）

T4: SendGrid 延迟发送 bounce 事件
    - 事件中只有 smtp-id: smtp-xyz，没有 workspaceId
    - 系统查找 processed:smtp-xyz 事件
    - 回填元数据：workspaceId, userId, templateId 等
    - 生成 EmailBounced 事件
    - 与 T1 的 EmailProcessed 事件正确关联
```

### 4.6 投递追踪中的消息关联

**文件：** `packages/backend-lib/src/deliveries.ts`

投递查询通过 `origin_message_id` 关联发送事件和状态事件：

```typescript
// deliveries.ts:432-435
LEFT JOIN status_events se ON
  ms.workspace_id = se.workspace_id
  AND ms.message_id = se.origin_message_id
  AND ms.user_or_anonymous_id = se.user_or_anonymous_id
```

**关联字段：**
- `ms.message_id`：MessageSent 事件的消息 ID
- `se.origin_message_id`：状态事件（opened、clicked 等）中引用的原始消息 ID

---

## 5. 架构总结

### 5.1 整体架构图

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
│                     统一发送层 (sendEmail)                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │           switch(emailProvider.type) 分发                      │  │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐    │  │
│  │  │SendGrid│ │Amazon  │ │ Resend │ │PostMark│ │MailChimp│   │  │
│  │  │ Adapter│ │ SES    │ │Adapter │ │Adapter │ │ Adapter │   │  │
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
│  │  sendgridEventToDF  submitAmazonSesEvents  resendEventToDF   │  │
│  │  postMarkEventToDF  submitMailChimpEvents                    │  │
│  └──────────────────────────────┬───────────────────────────────┘  │
│                                 │                                   │
│                                 ▼                                   │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              BatchTrackData (统一格式)                         │  │
│  │  { type, event, userId, messageId, timestamp, properties }   │  │
│  └──────────────────────────────┬───────────────────────────────┘  │
│                                 │                                   │
└─────────────────────────────────┼───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      事件处理管道 (Event Pipeline)                   │
│  submitBatch() → ClickHouse 存储 → 投递查询 / 分析报表               │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 关键设计原则

1. **分层抽象**
   - 发送层：统一的 `sendEmail` 入口，屏蔽 Provider 差异
   - 事件层：统一的 `submitBatch` 出口，归一化事件格式

2. **确定性 ID 生成**
   - 系统消息 ID：`randomUUID()`，发送前确定
   - 内部事件 ID：`uuidv5()`，基于 Provider 消息 ID 和 workspaceId

3. **元数据贯通**
   - 发送时注入：通过 customArgs/tags/metadata 传递
   - 接收时提取：从 webhook 事件中恢复
   - 延迟事件：通过已存储事件回填缺失元数据

4. **签名验证**
   - 每个 webhook 端点都有独立的签名验证机制
   - 防止伪造的事件注入

### 5.3 文件索引

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
| 投递查询 | `packages/backend-lib/src/deliveries.ts` |
| 类型定义 | `packages/isomorphic-lib/src/types.ts` |
