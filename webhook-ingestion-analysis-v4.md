# Dittofeed Webhook 事件摄入链路分析报告 V4

## 目录

1. [失败分支与成功路径总览](#失败分支与成功路径总览)
2. [Webhook 入口失败分支详解](#webhook-入口失败分支详解)
3. [客户端 API 入口失败分支详解](#客户端-api-入口失败分支详解)
4. [事件转换层失败分支详解](#事件转换层失败分支详解)
5. [核查清单总表](#核查清单总表)
6. [关键代码索引](#关键代码索引)

---

## 失败分支与成功路径总览

### 1.1 四层失败边界

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          事件处理四层边界                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  第 0 层：Schema 验证层                                                          │
│  ─────────────────                                                               │
│  - 请求体/请求头类型检查（由 Fastify TypeBox 自动验证）                           │
│  - 失败：HTTP 400，自动记录错误日志                                               │
│                                                                                 │
│  第 1 层：控制器/路由层                                                          │
│  ─────────────────                                                               │
│  - 签名验证失败                                                                 │
│  - Workspace ID 缺失/无效                                                       │
│  - Webhook Secret 未配置                                                        │
│  - Workspace 不活跃/不可用                                                      │
│  - 原始请求体缺失                                                                │
│  - 签名时间戳过期                                                                │
│                                                                                 │
│  第 2 层：事件转换层                                                              │
│  ─────────────────                                                               │
│  - 事件类型未映射（Unhandled event type）                                        │
│  - User ID 缺失                                                                 │
│  - 关键字段缺失（smtp-id、email_id、MessageSid 等）                               │
│  - Metadata 缺失                                                                 │
│                                                                                 │
│  第 3 层：入库/触发层                                                            │
│  ─────────────────                                                               │
│  - 已成功入库，但不触发 Journey                                                  │
│  - 已成功入库，且触发 Journey                                                    │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 全局 onSend Hook

**代码位置**：`packages/api/src/controllers/webhooksController.ts:63-74`

```typescript
// 所有 400 响应都会自动记录错误日志
fastify.addHook("onSend", async (_request, reply, payload) => {
  if (reply.statusCode !== 400) {
    return payload;
  }
  logger().error(
    { payload },
    "Failed to validate webhook payload.",
  );
  return payload;
});
```

---

## Webhook 入口失败分支详解

### 2.1 Sendgrid Webhook 失败分支

**端点**：`/api/public/webhooks/sendgrid`  
**代码位置**：`packages/api/src/controllers/webhooksController.ts:76-112` + `packages/backend-lib/src/destinations/sendgrid.ts:217-370`

| 失败条件 | HTTP 状态码 | 日志级别 | 日志信息 | 响应消息 | 证据代码 |
|---------|------------|---------|---------|---------|---------|
| **Schema 验证失败**（请求头/体不符合 TypeBox 定义） | 400 | error | "Failed to validate webhook payload." + payload | 自动生成 | `webhooksController.ts:63-74` |
| **handleSendgridEvents 内部错误** | 400 | info | "Error handling sendgrid webhook." + err | `result.error.message` | `webhooksController.ts:99-108` |
| **所有事件都无 workspaceId** | 400 | 无（由上层记录） | - | "No workspaceId found in events." | `sendgrid.ts:315-317` |
| **Webhook Secret 未配置** | 400 | 无（由上层记录） | - | "Missing secret." | `sendgrid.ts:333-335` |
| **签名验证失败** | 400 | 无（由上层记录） | - | "Invalid signature." | `sendgrid.ts:352-354` |
| **Workspace 不活跃** | 400 | 无（由上层记录） | - | "WorkspaceIneligible" | `sendgrid.ts:356-361` |
| **原始请求体缺失** | 500 | error | "Missing rawBody on sendgrid webhook." | 抛出 Error（无消息体） | `sendgrid.ts:340-343` |

**成功路径（HTTP 200）**：
- 事件已入库 → 不触发 Journey

---

### 2.2 Resend Webhook 失败分支

**端点**：`/api/public/webhooks/resend`  
**代码位置**：`packages/api/src/controllers/webhooksController.ts:183-282`

| 失败条件 | HTTP 状态码 | 日志级别 | 日志信息 | 响应消息 | 证据代码 |
|---------|------------|---------|---------|---------|---------|
| **Schema 验证失败**（svix-id/timestamp/signature 缺失或请求体格式错误） | 400 | error | "Failed to validate webhook payload." + payload | 自动生成 | `webhooksController.ts:63-74` |
| **workspaceId 缺失（tags 中无）** | 200 | info | "Missing workspaceId on resend tags" + tags | **静默成功**（空响应） | `webhooksController.ts:200-209` |
| **workspaceId 二次检查缺失**（冗余检查） | 400 | error | "Missing workspaceId on resend events." | "Missing workspaceId custom arg." | `webhooksController.ts:211-216` |
| **Webhook Secret 未配置** | 400 | error | "Missing resend webhook secret." + workspaceId | "Missing secret." | `webhooksController.ts:235-245` |
| **原始请求体缺失** | 500 | error | "Missing rawBody on resend webhook." + workspaceId | 空响应 | `webhooksController.ts:247-250` |
| **签名验证失败**（Svix 验证不通过） | 401 | error | "Invalid signature for resend webhook." + workspaceId | "Invalid signature." | `webhooksController.ts:255-265` |
| **Workspace 不活跃** | 401 | 无 | - | "Workspace not eligible." | `webhooksController.ts:267-274` |

**特殊注意**：
- 第 1 个 workspaceId 缺失检查返回 **HTTP 200**（静默忽略）
- 第 2 个 workspaceId 缺失检查返回 **HTTP 400**（似乎是冗余检查，永远不会触发）

**成功路径（HTTP 200）**：
- 事件已入库 → 不触发 Journey

---

### 2.3 Twilio Webhook 失败分支

**端点**：`/api/public/webhooks/twilio`  
**代码位置**：`packages/api/src/controllers/webhooksController.ts:531-615`

| 失败条件 | HTTP 状态码 | 日志级别 | 日志信息 | 响应消息 | 证据代码 |
|---------|------------|---------|---------|---------|---------|
| **Schema 验证失败**（x-twilio-signature 缺失、请求体格式错误、querystring 格式错误） | 400 | error | "Failed to validate webhook payload." + payload | 自动生成 | `webhooksController.ts:63-74` |
| **Webhook Secret 配置解析失败** | 503 | 无 | - | "Twilio configuration not found" | `webhooksController.ts:558-566` |
| **Twilio 配置字段缺失**（authToken/accountSid/messagingServiceSid 任一缺失） | 503 | 无 | - | "Twilio configuration not found" | `webhooksController.ts:568-576` |
| **签名验证失败** | 401 | error | "Invalid signature for twilio webhook." + workspaceId | "Invalid signature." | `webhooksController.ts:585-595` |
| **Workspace 不活跃** | 401 | 无 | - | "Workspace not eligible." | `webhooksController.ts:597-604` |

**成功路径（HTTP 200）**：
- 事件已入库 → 不触发 Journey

---

### 2.4 Postmark Webhook 失败分支

**端点**：`/api/public/webhooks/postmark`  
**代码位置**：`packages/api/src/controllers/webhooksController.ts:284-360`

| 失败条件 | HTTP 状态码 | 日志级别 | 日志信息 | 响应消息 | 证据代码 |
|---------|------------|---------|---------|---------|---------|
| **Schema 验证失败**（x-postmark-secret 缺失或请求体格式错误） | 400 | error | "Failed to validate webhook payload." + payload | 自动生成 | `webhooksController.ts:63-74` |
| **workspaceId 缺失**（Metadata 中无） | 400 | error | "Missing workspaceId in Metadata." | "Missing workspaceId in Metadata." | `webhooksController.ts:300-305` |
| **Webhook Secret 未配置** | 400 | error | "Missing postmark webhook secret." + workspaceId | "Missing secret." | `webhooksController.ts:326-336` |
| **签名验证失败**（x-postmark-secret 与配置不匹配） | 401 | error | "Invalid signature for PostMark webhook." | "Invalid signature." | `webhooksController.ts:338-343` |
| **Workspace 不活跃** | 401 | 无 | - | "Workspace not eligible." | `webhooksController.ts:345-352` |

**成功路径（HTTP 200）**：
- 事件已入库 → 不触发 Journey

---

### 2.5 Mailchimp (Mandrill) Webhook 失败分支

**端点**：`/api/public/webhooks/mailchimp`  
**代码位置**：`packages/api/src/controllers/webhooksController.ts:362-529`

| 失败条件 | HTTP 状态码 | 日志级别 | 日志信息 | 响应消息 | 证据代码 |
|---------|------------|---------|---------|---------|---------|
| **Schema 验证失败**（x-mandrill-signature 缺失、mandrill_events 缺失或格式错误） | 400 | error | "Failed to validate webhook payload." + payload | 自动生成 | `webhooksController.ts:63-74` |
| **mandrill_events JSON 解析失败** | 400 | error | "Failed to parse Mailchimp webhook payload" + err | "Invalid JSON in mandrill_events" | `webhooksController.ts:382-393` |
| **mandrill_events 不是数组** | 400 | error | "Invalid Mailchimp webhook payload" + events | "Invalid Mailchimp webhook payload" | `webhooksController.ts:395-406` |
| **所有事件解析失败**（schemaValidateWithErr 全部失败，最终 parsedEvents.length === 0） | 200 | debug | "No events in Mailchimp webhook" + rawBody | **静默成功**（空响应） | `webhooksController.ts:422-430` |
| **workspaceId 缺失**（所有事件的 metadata 中都无 workspaceId） | 400 | error | "Missing workspaceId in Mailchimp webhook metadata" + workspaceId + parsedEvents | "Missing workspaceId in metadata." | `webhooksController.ts:440-452` |
| **Webhook Secret 未配置** | 400 | error | "Missing mandrill webhook secret." + workspaceId | "Missing secret." | `webhooksController.ts:471-481` |
| **签名验证失败** | 401 | info | "Invalid signature for Mailchimp webhook." + workspaceId + signature + expectedSignature + url + signedData + rawBody | "Invalid signature." | `webhooksController.ts:497-512` |
| **Workspace 不活跃** | 401 | 无 | - | "Workspace not eligible." | `webhooksController.ts:514-521` |

**事件转换层失败（单个事件，不影响其他事件）**：

| 失败条件 | HTTP 状态码 | 日志级别 | 日志信息 | 响应 | 证据代码 |
|---------|------------|---------|---------|------|---------|
| **单个事件 schema 验证失败** | 200（继续处理） | error | "Failed to parse Mailchimp webhook payload" + err | continue（跳过该事件） | `webhooksController.ts:409-418` |

**成功路径（HTTP 200）**：
- 事件已入库 → 不触发 Journey

---

### 2.6 Amazon SES Webhook 失败分支

**端点**：`/api/public/webhooks/amazon-ses`  
**代码位置**：`packages/api/src/controllers/webhooksController.ts:114-181` + `packages/backend-lib/src/destinations/amazonses.ts:297-346`

| 失败条件 | HTTP 状态码 | 日志级别 | 日志信息 | 响应消息 | 证据代码 |
|---------|------------|---------|---------|---------|---------|
| **Schema 验证失败**（请求体格式错误） | 400 | error | "Failed to validate webhook payload." + payload | 自动生成 | `webhooksController.ts:63-74` |
| **SNS 签名验证失败** | 401 | error | "Invalid signature for AmazonSES webhook." + valid.error | "Invalid signature" | `webhooksController.ts:137-143` |
| **订阅确认失败** | 401 | error | "Unable to confirm AmazonSNS subscription." + error | 空响应 | `webhooksController.ts:153-159` |
| **Notification 消息解析失败**（JSON 解析或 schema 验证失败） | 500 | error | "Invalid AmazonSes event payload." + err | 空响应 | `amazonses.ts:304-317` |
| **submitAmazonSesEvents 失败**（workspaceId 缺失、workspace 不可用、事件类型未映射等） | 500 | error | "Error submitting AmazonSes events." + err + notification | 空响应 | `amazonses.ts:328-343` |

**成功路径（HTTP 200）**：
- 事件已入库 → 不触发 Journey

---

### 2.7 Segment Webhook 失败分支

**端点**：`/api/public/webhooks/segment`  
**代码位置**：`packages/api/src/controllers/webhooksController.ts:617-694`

| 失败条件 | HTTP 状态码 | 日志级别 | 日志信息 | 响应消息 | 证据代码 |
|---------|------------|---------|---------|---------|---------|
| **Schema 验证失败**（x-signature 缺失、messageId/timestamp 缺失） | 400 | error | "Failed to validate webhook payload." + payload | 自动生成 | `webhooksController.ts:63-74` |
| **workspaceId 获取失败**（getWorkspaceId 返回 Err） | 400 | 无 | - | 空响应 | `webhooksController.ts:638-641` |
| **workspaceId 缺失** | 400 | 无 | - | "Missing workspaceId. Try setting the df-workspace-id header." | `webhooksController.ts:643-647` |
| **Segment 配置未找到** | 503 | 无 | - | 空响应 | `webhooksController.ts:655-657` |
| **原始请求体缺失** | 500 | 无 | - | 空响应 | `webhooksController.ts:659-662` |
| **签名验证失败** | 401 | 无 | - | 空响应 | `webhooksController.ts:672-674` |
| **Workspace 不活跃** | 401 | 无 | - | "Workspace not eligible." | `webhooksController.ts:676-680` |

**成功路径（HTTP 200）**：
- 事件已入库（原始透传，不解析）→ 不触发 Journey

---

## 客户端 API 入口失败分支详解

### 3.1 公共验证逻辑

所有客户端 API 都使用 `validateWriteKey` 进行验证。

**代码位置**：`packages/backend-lib/src/auth.ts:75-116`

```typescript
export async function validateWriteKey({
  writeKey,
}: {
  writeKey: string;
}): Promise<Result<string, ValidateWriteKeyError>> {
  // 1. 提取 encodedWriteKey
  const encodedWriteKey = writeKey.split(" ")[1];
  if (!encodedWriteKey) {
    return err("InvalidWriteKey");  // ← 失败 1
  }

  // 2. 解码并拆分
  const decodedWriteKey = Buffer.from(encodedWriteKey, "base64").toString("utf-8");
  const [secretKeyId, secretKeyValue] = decodedWriteKey.split(":");
  if (!secretKeyId || !validate(secretKeyId)) {
    return err("InvalidWriteKey");  // ← 失败 2
  }

  // 3. 查询数据库
  const writeKeySecret = await db().query.secret.findFirst({ ... });
  if (!writeKeySecret) {
    return err("InvalidWriteKey");  // ← 失败 3
  }

  // 4. 检查 Workspace 是否可用
  if (!canWorkspaceReceiveEvents({ workspace: writeKeySecret.workspace })) {
    return err("WorkspaceIneligible");  // ← 失败 4
  }

  // 5. 比较密钥
  return writeKeySecret.value === secretKeyValue
    ? ok(writeKeySecret.workspaceId)
    : err("InvalidWriteKey");  // ← 失败 5
}
```

**Workspace 可用条件**：`packages/backend-lib/src/auth.ts:39-48`

```typescript
export function canWorkspaceReceiveEvents({
  workspace,
}: {
  workspace: Workspace;
}): boolean {
  return (
    workspace.status === WorkspaceStatusDbEnum.Active &&  // 必须是 Active 状态
    workspace.type !== "Parent"  // 不能是 Parent 类型
  );
}
```

---

### 3.2 `/api/public/track` 失败分支

**代码位置**：`packages/api/src/controllers/publicAppsController.ts:69-108`

| 失败条件 | HTTP 状态码 | 日志级别 | 响应消息 | 证据代码 |
|---------|------------|---------|---------|---------|
| **Schema 验证失败**（请求体格式错误或 authorization 头格式错误） | 400 | 无（Webhook onSend hook 不拦截这里） | 自动生成 | Fastify 内置 |
| **Write Key 验证失败**（InvalidWriteKey） | 401 | 无 | "Invalid write key." | `publicAppsController.ts:96-100` |
| **Workspace 不可用**（WorkspaceIneligible） | 401 | 无 | WorkspaceIneligible 错误描述 | `auth.ts:108-110` |

**成功路径（HTTP 204）**：
- 事件已入库 → 触发 Journey（调用 `submitTrackWithTriggers`）

---

### 3.3 `/api/public/batch` 失败分支

**代码位置**：`packages/api/src/controllers/publicAppsController.ts:250-288`

| 失败条件 | HTTP 状态码 | 日志级别 | 响应消息 | 证据代码 |
|---------|------------|---------|---------|---------|
| **Schema 验证失败** | 400 | 无 | 自动生成 | Fastify 内置 |
| **Write Key 验证失败** | 401 | 无 | WorkspaceIneligible 错误描述 | `publicAppsController.ts:277-281` |

**成功路径（HTTP 204）**：
- 所有事件已入库
- 仅 `EventType.Track` 事件触发 Journey（调用 `submitBatchWithTriggers`）

---

### 3.4 其他客户端 API 失败分支

| 端点 | 失败条件 | HTTP 状态码 | 成功后的行为 |
|------|---------|------------|-------------|
| `/api/public/identify` | 同 Track | 400/401 | 已入库，**不触发** Journey |
| `/api/public/page` | 同 Track | 400/401 | 已入库，**不触发** Journey |
| `/api/public/screen` | 同 Track | 400/401 | 已入库，**不触发** Journey |
| `/api/public/group` | 同 Track | 400/401 | 已入库，**不触发** Journey |
| `/api/public/alias` | 未实现 | 400 | "Not yet implemented." |

---

## 事件转换层失败分支详解

### 4.1 Sendgrid 事件转换失败

**代码位置**：`packages/backend-lib/src/destinations/sendgrid.ts:77-189` + `packages/backend-lib/src/destinations/sendgrid.ts:217-370`

| 失败条件 | 处理方式 | 日志级别 | 日志信息 | 证据代码 |
|---------|---------|---------|---------|---------|
| **userId 缺失** | 跳过该事件，继续处理其他事件 | error | "Failed to convert sendgrid event to DF." + err | `sendgrid.ts:199-215` |
| **processed 事件缺失 smtp-id** | 跳过该事件 | error | 同上 | `sendgrid.ts:105-108` |
| **bounce 事件缺失 smtp-id** | 跳过该事件 | error | 同上 | `sendgrid.ts:110-114` |
| **spamreport 事件缺失 smtp-id** | 跳过该事件 | error | 同上 | `sendgrid.ts:116-120` |
| **非 processed/bounce/spamreport 事件缺失 workspaceId 或 sg_message_id** | 跳过该事件 | error | 同上 | `sendgrid.ts:123-129` |
| **事件类型未映射** | 跳过该事件 | error | 同上 | `sendgrid.ts:169-170` |
| **bounce/spamreport 事件缺失 smtp-id（handleSendgridEvents 层）** | 跳过该事件，继续处理其他事件 | error | "Missing smtp-id for bounce or spamreport event." + event + workspaceId + sgMessageId | `sendgrid.ts:248-257` |

---

### 4.2 Resend 事件转换失败

**代码位置**：`packages/backend-lib/src/destinations/resend.ts:66-161`

| 失败条件 | 处理方式 | 日志级别 | 日志信息 | 证据代码 |
|---------|---------|---------|---------|---------|
| **userId 缺失** | 跳过该事件，继续处理其他事件 | error | "Failed to convert resend event to DF." + err | `resend.ts:145-155` |
| **事件类型未映射** | 跳过该事件 | error | 同上 | `resend.ts:107-108` |

---

### 4.3 Twilio 事件转换失败

**代码位置**：`packages/backend-lib/src/destinations/twilio.ts:113-181`

| 失败条件 | 处理方式 | 日志级别 | 日志信息 | 证据代码 |
|---------|---------|---------|---------|---------|
| **事件类型未映射**（只有 failed 和 delivered 被处理） | 返回错误 | error | "Unhandled Twilio event type" + workspaceId + userId + TwilioEvent | `twilio.ts:132-144` |
| **userId 缺失** | 返回错误 | 无（直接返回 Error） | - | `twilio.ts:146-148` |

**注意**：Twilio 的失败返回是 `ResultAsync.error`，需要看调用方如何处理。

**调用方处理**：`packages/api/src/controllers/webhooksController.ts:606-613`

```typescript
await submitTwilioEvents({
  ...tags,
  workspaceId,
  userId,
  TwilioEvent: request.body,
  subscriptionGroupId,
});
return reply.status(200).send();  // ← 注意：没有检查返回值！
```

**潜在问题**：`submitTwilioEvents` 返回 `ResultAsync`，但调用方没有检查结果，失败的事件可能被静默忽略。

---

### 4.4 Postmark 事件转换失败

**代码位置**：`packages/backend-lib/src/destinations/postmark.ts:58-163`

| 失败条件 | 处理方式 | 日志级别 | 日志信息 | 证据代码 |
|---------|---------|---------|---------|---------|
| **userId 缺失** | 跳过该事件，继续处理其他事件 | error | "Failed to convert resend event to DF." + err | `postmark.ts:147-156` |
| **事件类型未映射** | 跳过该事件 | error | 同上 | `postmark.ts:104-106` |
| **timestamp 或 userEmail 缺失** | 跳过该事件 | error | 同上 | `postmark.ts:108-110` |

**注意**：日志信息显示 "Failed to convert **resend** event to DF"，这是一个代码复制粘贴的 bug（应该是 postmark）。

---

### 4.5 Mailchimp 事件转换失败

**代码位置**：`packages/backend-lib/src/destinations/mailchimp.ts:129-218`

| 失败条件 | 处理方式 | 日志级别 | 日志信息 | 证据代码 |
|---------|---------|---------|---------|---------|
| **workspaceId 缺失** | 跳过该事件 | info | "Missing workspaceId in mailchimp event" + event | `mailchimp.ts:137-145` |
| **事件类型未映射** | 跳过该事件 | error | "Unhandled mailchimp event" + workspaceId + event | `mailchimp.ts:165-174` |
| **userId 缺失** | 跳过该事件 | info | "Missing userId in mailchimp event" + workspaceId + event | `mailchimp.ts:180-189` |

---

### 4.6 Amazon SES 事件转换失败

**代码位置**：`packages/backend-lib/src/destinations/amazonses.ts:180-278`

| 失败条件 | 处理方式 | 日志级别 | 日志信息 | 证据代码 |
|---------|---------|---------|---------|---------|
| **workspaceId 缺失**（tags 中无） | 返回错误 | 无（返回 Result.error） | - | `amazonses.ts:206-208` |
| **Workspace 不可用** | 返回错误 | 无 | - | `amazonses.ts:210-212` |
| **事件类型未映射** | 返回错误 | 无 | - | `amazonses.ts:237-240` |

---

## 核查清单总表

### 5.1 三类失败与成功路径对照表

| 类别 | 子类别 | 失败条件 | HTTP 状态码 | 是否入库 | 是否触发 Journey | 日志级别 | 典型错误消息 |
|------|--------|---------|------------|---------|-----------------|---------|-------------|
| **❌ 入库前就失败** | Schema 验证 | 请求体/头不符合 TypeBox 定义 | 400 | ❌ 否 | ❌ 否 | error | "Failed to validate webhook payload." |
| **❌ 入库前就失败** | 签名验证 | HMAC/Svix/Twilio 签名不匹配 | 401 | ❌ 否 | ❌ 否 | error | "Invalid signature." |
| **❌ 入库前就失败** | Workspace ID | workspaceId 缺失或无法获取 | 400/200 | ❌ 否 | ❌ 否 | info/error | "Missing workspaceId..." |
| **❌ 入库前就失败** | Secret 配置 | Webhook Secret 未配置或格式错误 | 400/503 | ❌ 否 | ❌ 否 | error | "Missing secret." / "Twilio configuration not found" |
| **❌ 入库前就失败** | Workspace 状态 | Workspace 不是 Active 或类型是 Parent | 401 | ❌ 否 | ❌ 否 | 无 | "Workspace not eligible." / "WorkspaceIneligible" |
| **❌ 入库前就失败** | 原始请求体 | rawBody 缺失 | 500 | ❌ 否 | ❌ 否 | error | "Missing rawBody..." |
| **❌ 部分事件失败** | 事件转换 | userId 缺失 | 200（HTTP） | ⚠️ 部分 | ❌ 否 | error | "Missing userId..." |
| **❌ 部分事件失败** | 事件转换 | 关键字段缺失（smtp-id、email_id、MessageSid 等） | 200（HTTP） | ⚠️ 部分 | ❌ 否 | error | "Missing smtp-id..." |
| **❌ 部分事件失败** | 事件转换 | 事件类型未映射 | 200（HTTP） | ⚠️ 部分 | ❌ 否 | error | "Unhandled event type..." |
| **⚠️ 已入库但不触发** | 设计行为 | 邮件/SMS 状态回调（所有 Webhook） | 200 | ✅ 是 | ❌ 否 | debug | "Received {provider} events." |
| **⚠️ 已入库但不触发** | 设计行为 | Segment 原始透传 | 200 | ✅ 是 | ❌ 否 | debug | 无 |
| **⚠️ 已入库但不触发** | 设计行为 | Identify/Page/Screen/Group 事件 | 204 | ✅ 是 | ❌ 否 | debug | 无 |
| **✅ 已入库且触发** | 设计行为 | `/api/public/track` Track 事件 | 204 | ✅ 是 | ✅ 是 | debug | 无 |
| **✅ 已入库且触发** | 设计行为 | `/api/public/batch` 中的 Track 事件 | 204 | ✅ 是 | ✅ 是 | debug | 无 |

---

### 5.2 Webhook 入口失败分支详细表（按提供商分类）

| 提供商 | 失败条件 | HTTP 状态码 | 日志级别 | 响应消息 | 特殊说明 |
|--------|---------|------------|---------|---------|---------|
| **Sendgrid** | Schema 验证失败 | 400 | error | 自动生成 | 全局 onSend hook |
| **Sendgrid** | 签名验证失败 | 400 | info | "Invalid signature." | handleSendgridEvents 返回 Err |
| **Sendgrid** | workspaceId 缺失 | 400 | 无 | "No workspaceId found in events." | |
| **Sendgrid** | Secret 未配置 | 400 | 无 | "Missing secret." | |
| **Sendgrid** | Workspace 不活跃 | 400 | 无 | "WorkspaceIneligible" | |
| **Sendgrid** | rawBody 缺失 | 500 | error | 空响应 | 抛出 Error |
| **Sendgrid** | 事件转换失败（单个事件） | 200 | error | 无 | 跳过该事件，继续处理 |
| **Resend** | Schema 验证失败 | 400 | error | 自动生成 | |
| **Resend** | workspaceId 缺失（第 1 次检查） | 200 | info | **空响应** | **静默忽略** |
| **Resend** | workspaceId 缺失（第 2 次检查） | 400 | error | "Missing workspaceId custom arg." | 冗余检查，实际不会触发 |
| **Resend** | Secret 未配置 | 400 | error | "Missing secret." | |
| **Resend** | 签名验证失败 | 401 | error | "Invalid signature." | Svix 验证 |
| **Resend** | Workspace 不活跃 | 401 | 无 | "Workspace not eligible." | |
| **Resend** | rawBody 缺失 | 500 | error | 空响应 | |
| **Resend** | 事件转换失败（单个事件） | 200 | error | 无 | 跳过该事件 |
| **Twilio** | Schema 验证失败 | 400 | error | 自动生成 | |
| **Twilio** | Secret 配置解析失败 | 503 | 无 | "Twilio configuration not found" | |
| **Twilio** | Secret 字段缺失 | 503 | 无 | "Twilio configuration not found" | |
| **Twilio** | 签名验证失败 | 401 | error | "Invalid signature." | |
| **Twilio** | Workspace 不活跃 | 401 | 无 | "Workspace not eligible." | |
| **Twilio** | 事件转换失败 | 200 | error | 无 | **返回 Err 但调用方未检查**，可能被静默忽略 |
| **Postmark** | Schema 验证失败 | 400 | error | 自动生成 | |
| **Postmark** | workspaceId 缺失 | 400 | error | "Missing workspaceId in Metadata." | |
| **Postmark** | Secret 未配置 | 400 | error | "Missing secret." | |
| **Postmark** | 签名验证失败 | 401 | error | "Invalid signature." | 密钥头匹配 |
| **Postmark** | Workspace 不活跃 | 401 | 无 | "Workspace not eligible." | |
| **Postmark** | 事件转换失败（单个事件） | 200 | error | 无 | 跳过该事件，日志信息错误显示 "resend" |
| **Mailchimp** | Schema 验证失败 | 400 | error | 自动生成 | |
| **Mailchimp** | mandrill_events JSON 解析失败 | 400 | error | "Invalid JSON in mandrill_events" | |
| **Mailchimp** | mandrill_events 不是数组 | 400 | error | "Invalid Mailchimp webhook payload" | |
| **Mailchimp** | 所有事件解析失败 | 200 | debug | **空响应** | **静默忽略** |
| **Mailchimp** | workspaceId 缺失 | 400 | error | "Missing workspaceId in metadata." | |
| **Mailchimp** | Secret 未配置 | 400 | error | "Missing secret." | |
| **Mailchimp** | 签名验证失败 | 401 | info | "Invalid signature." | **记录详细调试信息**（signature、expectedSignature、url、signedData、rawBody） |
| **Mailchimp** | Workspace 不活跃 | 401 | 无 | "Workspace not eligible." | |
| **Mailchimp** | 单个事件解析失败 | 200 | error | 无 | 跳过该事件 |
| **Mailchimp** | 事件转换失败（单个事件） | 200 | info/error | 无 | 跳过该事件 |
| **Amazon SES** | Schema 验证失败 | 400 | error | 自动生成 | |
| **Amazon SES** | SNS 签名验证失败 | 401 | error | "Invalid signature" | |
| **Amazon SES** | 订阅确认失败 | 401 | error | 空响应 | |
| **Amazon SES** | Notification 解析失败 | 500 | error | 空响应 | |
| **Amazon SES** | submitAmazonSesEvents 失败 | 500 | error | 空响应 | workspaceId 缺失、Workspace 不可用、事件类型未映射 |
| **Segment** | Schema 验证失败 | 400 | error | 自动生成 | |
| **Segment** | workspaceId 获取失败 | 400 | 无 | 空响应 | |
| **Segment** | workspaceId 缺失 | 400 | 无 | "Missing workspaceId. Try setting the df-workspace-id header." | |
| **Segment** | Segment 配置未找到 | 503 | 无 | 空响应 | |
| **Segment** | rawBody 缺失 | 500 | 无 | 空响应 | |
| **Segment** | 签名验证失败 | 401 | 无 | 空响应 | |
| **Segment** | Workspace 不活跃 | 401 | 无 | "Workspace not eligible." | |
| **客户端 Track** | Schema 验证失败 | 400 | 无 | 自动生成 | |
| **客户端 Track** | Write Key 无效 | 401 | 无 | "Invalid write key." | |
| **客户端 Track** | Workspace 不可用 | 401 | 无 | WorkspaceIneligible | |
| **客户端 Track** | 成功 | 204 | debug | 无 | **已入库且触发 Journey** |

---

### 5.3 事件转换层失败模式汇总

| 提供商 | 失败类型 | 失败条件 | 处理方式 | HTTP 状态码 | 日志级别 |
|--------|---------|---------|---------|------------|---------|
| **Sendgrid** | userId 缺失 | 事件 metadata 中无 userId | 跳过该事件 | 200 | error |
| **Sendgrid** | smtp-id 缺失 | processed/bounce/spamreport 事件无 smtp-id | 跳过该事件 | 200 | error |
| **Sendgrid** | workspaceId/sg_message_id 缺失 | 非异步事件无关键字段 | 跳过该事件 | 200 | error |
| **Sendgrid** | 事件类型未映射 | 除 open/click/bounce/dropped/spamreport/delivered/processed 外 | 跳过该事件 | 200 | error |
| **Resend** | userId 缺失 | data.tags.userId 不存在 | 跳过该事件 | 200 | error |
| **Resend** | 事件类型未映射 | 除 opened/clicked/bounced/delivery_delayed/complained/delivered 外（sent 被忽略） | 跳过该事件 | 200 | error |
| **Twilio** | 事件类型未映射 | 除 failed/delivered 外 | 返回 Err | 200** | error |
| **Twilio** | userId 缺失 | query.userId 不存在 | 返回 Err | 200** | 无 |
| **Postmark** | userId 缺失 | Metadata.userId 不存在 | 跳过该事件 | 200 | error |
| **Postmark** | 事件类型未映射 | 除 Open/Click/Bounce/SpamComplaint/Delivery 外 | 跳过该事件 | 200 | error |
| **Postmark** | timestamp/userEmail 缺失 | 关键字段缺失 | 跳过该事件 | 200 | error |
| **Mailchimp** | workspaceId 缺失 | msg.metadata.workspaceId 不存在 | 跳过该事件 | 200 | info |
| **Mailchimp** | 事件类型未映射 | 除 open/click/hard_bounce/spam/delivered 外 | 跳过该事件 | 200 | error |
| **Mailchimp** | userId 缺失 | msg.metadata.userId 不存在 | 跳过该事件 | 200 | info |
| **Amazon SES** | workspaceId 缺失 | tags.workspaceId 不存在 | 返回 Err | 500 | 无 |
| **Amazon SES** | Workspace 不可用 | workspace 状态不对 | 返回 Err | 500 | 无 |
| **Amazon SES** | 事件类型未映射 | 除 Bounce/Complaint/Delivery/Open/Click 外 | 返回 Err | 500 | 无 |

**注意**：Twilio 的失败返回 `ResultAsync.error`，但调用方 `webhooksController.ts` 没有检查返回值，事件可能被**静默忽略**。

---

## 关键代码索引

### 6.1 控制器层

| 文件 | 功能 | 关键行号 |
|------|------|---------|
| `packages/api/src/controllers/webhooksController.ts` | 所有 Webhook 端点 + onSend Hook | 63-74, 76-694 |
| `packages/api/src/controllers/publicAppsController.ts` | 客户端 API 端点 | 28-288 |

### 6.2 事件转换层

| 文件 | 功能 | 关键行号 |
|------|------|---------|
| `packages/backend-lib/src/destinations/sendgrid.ts` | Sendgrid 事件转换 + 签名验证 | 77-370 |
| `packages/backend-lib/src/destinations/resend.ts` | Resend 事件转换 | 66-161 |
| `packages/backend-lib/src/destinations/twilio.ts` | Twilio 事件转换 | 113-181 |
| `packages/backend-lib/src/destinations/postmark.ts` | Postmark 事件转换 | 58-163 |
| `packages/backend-lib/src/destinations/mailchimp.ts` | Mailchimp 事件转换 | 129-218 |
| `packages/backend-lib/src/destinations/amazonses.ts` | Amazon SES 事件转换 + SNS 处理 | 180-346 |

### 6.3 认证层

| 文件 | 功能 | 关键行号 |
|------|------|---------|
| `packages/backend-lib/src/auth.ts` | Write Key 验证 + Workspace 可用性检查 | 39-116 |

---

## 总结

### 7.1 核心发现

1. **四层失败边界**：
   - 第 0 层：Schema 验证（Fastify TypeBox 自动处理）
   - 第 1 层：控制器/路由层（签名、workspace、secret 等）
   - 第 2 层：事件转换层（userId、事件类型映射、关键字段等）
   - 第 3 层：入库/触发层（已入库但不触发 vs 已入库且触发）

2. **静默失败的三个危险点**：
   - **Resend workspaceId 缺失**：返回 HTTP 200，info 级别日志，容易被忽略
   - **Mailchimp 所有事件解析失败**：返回 HTTP 200，debug 级别日志
   - **Twilio 事件转换失败**：返回 `ResultAsync.error`，但调用方未检查

3. **Mailchimp 签名验证日志过于详细**：
   - 记录了 `signature`、`expectedSignature`、`url`、`signedData`、`rawBody`
   - 可能泄露敏感信息

4. **Postmark 日志信息错误**：
   - 日志显示 "Failed to convert **resend** event to DF"，实际是 Postmark

5. **"已入库但不触发"是设计行为**：
   - 所有邮件/SMS Webhook：状态回调，只用于数据分析
   - Segment Webhook：原始透传，依赖外部消费
   - Identify/Page/Screen/Group：非业务行为事件

6. **"已入库且触发"的唯二入口**：
   - `/api/public/track` → `submitTrackWithTriggers`
   - `/api/public/batch` → `submitBatchWithTriggers`（仅 Track 事件）
