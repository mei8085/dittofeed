# Dittofeed 消息模板渲染机制分析报告

## 目录

1. [概述](#1-概述)
2. [模板引擎与变量绑定机制](#2-模板引擎与变量绑定机制)
3. [多语言 Fallback 链路](#3-多语言-fallback-链路)
4. [跨渠道字符和尺寸限制校验逻辑](#4-跨渠道字符和尺寸限制校验逻辑)
5. [核心代码文件索引](#5-核心代码文件索引)

---

## 1. 概述

### 1.1 技术栈说明

**重要澄清**：Dittofeed 使用的是 **Liquid** 模板引擎（通过 `liquidjs` 库），而非 Handlebars。Liquid 语法与 Handlebars 相似（都使用 `{{ variable }}` 语法），但属于不同的模板引擎实现。

- **模板引擎**：Liquid (liquidjs)
- **邮件渲染**：MJML → HTML（通过 `mjml2html`）
- **支持渠道**：Email, SMS, Webhook, MobilePush

### 1.2 核心渲染流程

```
模板字符串
    ↓
Liquid 渲染 (变量绑定)
    ↓
MJML 转换 (如果启用)
    ↓
最终输出 (HTML/纯文本/JSON)
```

---

## 2. 模板引擎与变量绑定机制

### 2.1 Liquid 引擎配置

**文件位置**：`packages/backend-lib/src/liquid.ts:48-74`

```typescript
export const liquidEngine = new Liquid({
  strictVariables: true,      // 严格变量检查 - 未定义变量抛出错误
  lenientIf: true,            // 宽松条件判断
  relativeReference: false,   // 禁用相对引用
  fs: { ... }                 // 自定义文件系统（用于 layout）
});
```

**关键配置说明**：
- `strictVariables: true`：引用未定义变量时会抛出错误，而不是静默返回空值
- `lenientIf: true`：条件判断中对 undefined/null 更宽容

### 2.2 核心渲染函数

**文件位置**：`packages/backend-lib/src/liquid.ts:258-297`

```typescript
export function renderLiquid({
  template,
  userProperties,
  workspaceId,
  subscriptionGroupId,
  identifierKey,
  secrets = {},
  mjml = false,
  tags,
  isPreview = false,
  messageId,
}: RenderLiquidOptions): string
```

### 2.3 变量注入上下文

渲染时注入的完整上下文对象：

```typescript
{
  user: UserPropertyAssignments,      // 用户属性对象
  workspace_id: string,               // 工作区 ID
  subscription_group_id: string,      // 订阅组 ID
  secrets: Secrets,                   // 敏感密钥
  identifier_key: string,             // 标识符键名
  tags: MessageTags,                  // 消息标签
  is_preview: boolean,                // 是否预览模式
  message_id: string                  // 消息 ID
}
```

### 2.4 用户属性访问方式

在模板中通过 `{{ user.属性名 }}` 访问用户属性：

**示例**：
```liquid
Hello {{ user.firstName }},

Your email is {{ user.email }}.

{% if user.language == "zh-CN" %}
  你好！
{% else %}
  Hello!
{% endif %}
```

### 2.5 自定义 Liquid 标签

系统注册了以下自定义标签：

| 标签 | 用途 | 文件位置 |
|------|------|----------|
| `{% unsubscribe_link %}` | 生成带链接文本的退订链接 | `liquid.ts:160-172` |
| `{% unsubscribe_url %}` | 生成纯退订 URL | `liquid.ts:175-179` |
| `{% subscription_management_link %}` | 生成订阅管理链接 | `liquid.ts:181-194` |
| `{% subscription_management_url %}` | 生成纯订阅管理 URL | `liquid.ts:196-200` |
| `{% view_in_browser_url %}` | 生成网页版查看 URL | `liquid.ts:234-238` |

### 2.5.1 各渠道标识符键（Identifier Key）

**文件位置**：`packages/isomorphic-lib/src/channels.ts:3-10`

```typescript
export const CHANNEL_IDENTIFIERS: Record<
  Exclude<ChannelType, "Webhook">,
  string
> = {
  [ChannelType.Email]: "email",
  [ChannelType.MobilePush]: "deviceToken",
  [ChannelType.Sms]: "phone",
};
```

**说明**：
- Email → `email` 用户属性
- SMS → `phone` 用户属性
- MobilePush → `deviceToken` 用户属性
- Webhook → 无固定标识符，支持自定义属性访问

**使用示例**：
```liquid
{% unsubscribe_link 点击这里退订 %}

或者只获取 URL:
{% unsubscribe_url %}
```

### 2.6 MJML 渲染流程

**文件位置**：`packages/backend-lib/src/liquid.ts:288-296`

```typescript
if (!mjml) {
  return liquidRendered;
}
try {
  return mjml2html(liquidRendered).html;
} catch (e) {
  const error = e as Error;
  if (error.message.includes(MJML_NOT_PRESENT_ERROR)) {
    return liquidRendered;  // MJML 解析失败时回退到纯 HTML
  }
  throw e;
}
```

**MJML 处理规则**：
1. 首先执行 Liquid 变量绑定
2. 如果 `mjml: true`，调用 `mjml2html()` 转换
3. 如果 MJML 结构错误（如缺少 `<mjml>` 标签），回退返回 Liquid 渲染结果
4. 其他 MJML 错误会向上抛出

### 2.7 多字段批量渲染

**文件位置**：`packages/backend-lib/src/messaging.ts:561-596`

对于邮件模板，系统使用 `renderValues` 函数批量渲染多个字段：

```typescript
function renderValues<T extends TemplateDictionary<T>>({
  templates,
  ...rest
}: Omit<Parameters<typeof renderLiquid>[0], "template"> & {
  templates: T;
}): Result<...>
```

**邮件模板渲染的字段**（`messaging.ts:922-945`）：

```typescript
templates: {
  from: { contents: messageTemplateDefinition.from },
  subject: { contents: messageTemplateDefinition.subject },
  body: { contents: emailBody, mjml: true },
  replyTo: { contents: messageTemplateDefinition.replyTo },
  name: { contents: messageTemplateDefinition.name },
  cc: { contents: messageTemplateDefinition.cc },
  bcc: { contents: messageTemplateDefinition.bcc },
}
```

**自定义 Headers 也支持模板**（`messaging.ts:1011-1031`）：

```typescript
for (const header of messageTemplateDefinition.headers) {
  headersToRender[header.name] = {
    contents: header.value,
  };
}
```

### 2.8 两种邮件模板类型

**文件位置**：`packages/isomorphic-lib/src/types.ts:1697-1724`

1. **Code 模式**（直接编写 MJML/HTML）
2. **Low Code 模式**（可视化编辑器生成的 JSON 结构）

**Low Code 到 MJML 的转换**：`packages/backend-lib/src/messaging.ts:896-905`

```typescript
if (messageTemplateDefinition.emailContentsType === EmailContentsType.LowCode) {
  const mjml = toMjml({
    content: messageTemplateDefinition.body,
    mode: "render",
  });
  emailBody = mjml;
} else {
  emailBody = messageTemplateDefinition.body;
}
```

---

## 3. 多语言 Fallback 链路

### 3.1 当前状态：功能缺失

**重要发现**：Dittofeed 代码库中**不存在完整的多语言模板 fallback 机制**。

虽然系统预留了多语言相关的基础设施：

| 组件 | 状态 | 文件位置 |
|------|------|----------|
| `language` 用户属性 | ✅ 已定义 | `bootstrap.ts:245-251` |
| `defaultLanguageUserPropertyId` 配置 | ✅ 已定义 | `config.ts:62` |
| 基于语言的模板选择 | ❌ 未实现 | - |
| 多语言版本 fallback | ❌ 未实现 | - |

### 3.2 完整调用链分析：为何多语言无法工作

以下是从 Journey 消息节点到最终渲染的 **完整 12 层调用链**，**每层都用代码位置证明 `language` 完全未参与模板选择**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              完整调用链总览                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. Journey MessageNode 定义 (types.ts:1167-1185)                           │
│       │                                                                     │
│       ├── variant.templateId: 固定的模板 ID 字符串                           │
│       └── ⚠️ 无 language 字段                                               │
│                                                                             │
│  2. User Workflow 消息节点处理 (userWorkflow.ts:852-961)                     │
│       │                                                                     │
│       ├── 从 currentNode.variant.templateId 提取固定模板 ID                  │
│       ├── 构造 SendParamsV2 (无 language 参数)                              │
│       └── 调用 sendMessageV2()                                              │
│                                                                             │
│  3. sendMessageV2 入口 (userWorkflow/activities.ts:303-394)                 │
│       │                                                                     │
│       ├── sendMessageFactory(sendMessage) 闭包                             │
│       └── 委托给 sendMessageInner()                                         │
│                                                                             │
│  4. sendMessageInner (userWorkflow/activities.ts:174-301)                   │
│       │                                                                     │
│       ├── findAllUserPropertyAssignments() ← 这里获取到 user.language       │
│       ├── userPropertyAssignments 包含 language（如果用户有这个属性）        │
│       ├── 但**不读取** language 用于任何模板选择决策                         │
│       └── 调用 sender() → sendMessage()                                     │
│                                                                             │
│  5. sendMessage() 分发 (messaging.ts:2399-2420)                             │
│       │                                                                     │
│       ├── SendMessageParameters 类型无 language 字段                        │
│       ├── switch (params.channel) 分发到各渠道函数                           │
│       └── MobilePush: throw new Error("not implemented")                    │
│                                                                             │
│  6. getSendMessageModels() 模板获取 (messaging.ts:401-503)                   │
│       │                                                                     │
│       ├── 参数: { workspaceId, templateId, channel, useDraft, ... }         │
│       ├── 调用 findMessageTemplate({ id: templateId, channel })             │
│       └── ⚠️ 未传递 userPropertyAssignments 或 language                    │
│                                                                             │
│  7. findMessageTemplate() 数据库查询 (messaging.ts:183-205)                  │
│       │                                                                     │
│       ├── 签名: function findMessageTemplate({ id, channel })               │
│       ├── 查询: where: eq(dbMessageTemplate.id, id)                         │
│       ├── ⚠️ 只有 id 一个查询条件，无 language/locale 过滤                   │
│       └── 版本验证: definition.type === channel                             │
│                                                                             │
│  8. 模板版本选择 (messaging.ts:484-489)                                      │
│       │                                                                     │
│       ├── definitionFromDraft = useDraft ? draft : null                     │
│       ├── finalDef = definitionFromDraft ?? definition ?? null              │
│       └── ⚠️ 只有 useDraft 标志，无 language 版本选择                        │
│                                                                             │
│  9. 渠道特定发送函数 (sendEmail/sendSms/sendWebhook)                         │
│       │                                                                     │
│       ├── 接收 userPropertyAssignments 参数                                 │
│       ├── 调用 renderValues() 渲染各字段                                    │
│       └── ⚠️ 模板 ID 已固定，language 只用于变量绑定                        │
│                                                                             │
│ 10. renderValues() 批量渲染 (messaging.ts:561-596)                          │
│       │                                                                     │
│       ├── 遍历 templates 对象（from, subject, body 等）                     │
│       ├── 对每个字段调用 renderLiquid()                                     │
│       └── ⚠️ 只是变量绑定，不选择模板版本                                   │
│                                                                             │
│ 11. renderLiquid() 核心渲染 (liquid.ts:258-297)                             │
│       │                                                                     │
│       ├── 构造 context: { user: userProperties, ... }                      │
│       ├── user.language 作为普通变量注入 Liquid 上下文                      │
│       ├── 可用于 {% if user.language == "zh-CN" %} 条件判断                  │
│       └── ⚠️ 但这是模板内逻辑，不是系统级 fallback                           │
│                                                                             │
│ 12. 最终输出                                                                 │
│       │                                                                     │
│       └── 返回渲染后的字符串，无语言相关后处理                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2.1 每层调用链的代码位置证明

#### 第 1 层：Journey MessageNode 定义

**文件位置**：`packages/isomorphic-lib/src/types.ts:1167-1185`

```typescript
export const MessageNode = Type.Object({
  ...BaseNode,
  type: Type.Literal(JourneyNodeType.MessageNode),
  name: Type.Optional(Type.String()),
  subscriptionGroupId: Type.Optional(Type.String()),
  variant: MessageVariant,      // ← 包含固定 templateId
  child: Type.String(),
  syncProperties: Type.Optional(Type.Boolean()),
  skipOnFailure: Type.Optional(Type.Boolean()),
  retryCount: Type.Optional(Type.Number({ description: "Number of retry attempts (default: 3)" })),
});
```

**MessageVariant**（`types.ts:1054-1163`）：
```typescript
export const MobilePushMessageVariant = Type.Object({
  type: Type.Literal(ChannelType.MobilePush),
  templateId: Type.String(),           // ← 固定字符串 ID
  providerOverride: Type.Optional(Type.Enum(MobilePushProviderType)),
});
// ⚠️ 无 language 或 locale 字段
```

**结论**：MessageNode 绑定固定的 `templateId`，没有语言选择的空间。

#### 第 2 层：User Workflow 消息节点处理

**文件位置**：`packages/backend-lib/src/journeys/userWorkflow.ts:852-961`

```typescript
case JourneyNodeType.MessageNode: {
  const messageId = uuid4();
  // ... 触发消息 ID 处理 ...

  const messagePayload: Omit<activities.SendParams, "templateId"> = {
    userId,
    workspaceId,
    journeyId,
    subscriptionGroupId: currentNode.subscriptionGroupId,
    runId,
    nodeId: currentNode.id,
    messageId,
    triggeringMessageId,
    // ⚠️ 无 language 字段
  };

  let variant: RenameKey<MessageVariant, "type", "channel">;
  switch (currentNode.variant.type) {
    case ChannelType.Email:
    case ChannelType.Sms:
    case ChannelType.Webhook:
    case ChannelType.MobilePush:
      variant = {
        ...omit(currentNode.variant, ["type"]),
        channel: currentNode.variant.type,
        // ⚠️ 直接使用 currentNode.variant.templateId
        // ⚠️ 没有任何基于用户属性的动态模板选择
      };
      break;
  }

  const sendMesssageParams: SendParamsV2 = {
    ...messagePayload,
    ...variant,
    events: keyedEvents,
    context: entryEventProperties,
    isHidden,
    // ⚠️ 仍无 language 字段
  };

  const { sendMessageV2 } = proxyActivities<typeof activities>({
    startToCloseTimeout: "2 minutes",
    retry: { maximumAttempts: currentNode.retryCount ?? 3 },
  });
  const messageSucceeded = await sendMessageV2(sendMesssageParams);
```

**结论**：直接从 Journey 节点提取 `templateId`，无任何动态选择逻辑。

#### 第 3-4 层：sendMessageV2 → sendMessageInner

**文件位置**：`packages/backend-lib/src/journeys/userWorkflow/activities.ts:303-394`

```typescript
export function sendMessageFactory(sender: Sender) {
  return async function sendMessageWithSender(
    params: SendParamsV2,
  ): Promise<boolean> {
    // ... Journey 状态检查 ...
    const sendResult = await sendMessageInner({
      ...params,
      sender,
    });
    // ... 事件跟踪 ...
  };
}

export const sendMessageV2 = sendMessageFactory(sendMessage);
```

**sendMessageInner**（`activities.ts:174-301`）：
```typescript
async function sendMessageInner({
  userId,
  workspaceId,
  runId,
  nodeId,
  templateId,     // ← 来自 params，是固定值
  journeyId,
  messageId,
  subscriptionGroupId,
  context: deprecatedContext,
  events,
  sender,
  retryCount,
  ...rest
}: SendParamsInner): Promise<SendMessageInnerResult> {
  // ... context 处理 ...

  const [userPropertyAssignments, journey, subscriptionGroup] =
    await Promise.all([
      findAllUserPropertyAssignments({ userId, workspaceId, context }),
      // ↑ 这里获取到了所有用户属性，可能包含 language
      // ...
    ]);

  // ... subscriptionGroupDetails 处理 ...

  try {
    const result = await sender({
      workspaceId,
      useDraft: false,
      templateId,       // ← 仍然使用原始固定 ID
      userId,
      userPropertyAssignments,  // ← 传递给 sendMessage()
      subscriptionGroupDetails,
      messageTags,
      ...rest,
    });
    return result;
  } catch (senderError) {
    // ... 重试逻辑 ...
  }
}
```

**关键发现**：
- `findAllUserPropertyAssignments()` 获取到了用户属性（包括 `language`）
- 但 `templateId` 从未被重新赋值，仍然是 Journey 节点中的固定值
- 没有任何 `if (userPropertyAssignments.language === "zh-CN") { templateId = "..." }` 这样的逻辑

#### 第 5 层：sendMessage() 分发

**文件位置**：`packages/backend-lib/src/messaging.ts:2399-2420`

```typescript
export async function sendMessage(
  params: SendMessageParameters,
): Promise<BackendMessageSendResult> {
  return withSpan({ name: "sendMessage" }, async (span) => {
    span.setAttributes({
      channel: params.channel,
      workspaceId: params.workspaceId,
      templateId: params.templateId,
      // ⚠️ 没有 language 相关属性
    });
    switch (params.channel) {
      case ChannelType.Email:
        return sendEmail(params);
      case ChannelType.Sms:
        return sendSms(params);
      case ChannelType.MobilePush:
        throw new Error("not implemented");  // ← Mobile Push 发送未实现
      case ChannelType.Webhook:
        return sendWebhook(params);
    }
  });
}
```

**SendMessageParameters**（`messaging.ts:510-543`）：
```typescript
interface SendMessageParametersBase {
  workspaceId: string;
  userId: string;
  templateId: string;
  userPropertyAssignments: UserPropertyAssignments;
  subscriptionGroupDetails?: Omit<SubscriptionGroupDetails, "name">;
  messageTags?: MessageTags;
  useDraft?: boolean;
  isPreview?: boolean;
  // ⚠️ 无 language 参数
}
```

#### 第 6-7 层：模板查询

**文件位置**：`packages/backend-lib/src/messaging.ts:183-205`

```typescript
export async function findMessageTemplate({
  id,         // ← 只有 ID 参数
  channel,
}: {
  id: string;
  channel: ChannelType;
}): Promise<Result<MessageTemplateResource | null, Error>> {
  if (!validateUuid(id)) {
    logger().info({ id, channel }, "Invalid message template id");
    return ok(null);
  }
  const template = await db().query.messageTemplate.findFirst({
    where: eq(dbMessageTemplate.id, id),
    // ⚠️ 只有 id 一个条件！
    // ⚠️ 没有: eq(dbMessageTemplate.language, userLanguage)
  });
  if (!template) {
    return ok(null);
  }

  return enrichMessageTemplate(template).map((t) => {
    const definition = t.draft ?? t.definition ?? null;
    return definition && definition.type === channel ? t : null;
    // ⚠️ 只验证 channel 类型，不验证 language
  });
}
```

**调用者**（`messaging.ts:440-450`）：
```typescript
const [messageTemplateResult, subscriptionGroupSecret] = await Promise.all([
  findMessageTemplate({
    id: templateId,    // ← 固定 ID
    channel,
    // ⚠️ 没有传递 userPropertyAssignments
    // ⚠️ 没有 language 参数
  }),
  // ...
]);
```

#### 第 8 层：模板版本选择

**文件位置**：`packages/backend-lib/src/messaging.ts:484-489`

```typescript
const definitionFromDraft =
  useDraft && messageTemplate.draft
    ? messageTemplateDraftToDefinition(messageTemplate.draft).unwrapOr(null)
    : null;
const messageTemplateDefinition: MessageTemplateResourceDefinition | null =
  definitionFromDraft ?? messageTemplate.definition ?? null;

if (!messageTemplateDefinition) {
  return err({
    type: InternalEventType.BadWorkspaceConfiguration,
    variant: {
      type: BadWorkspaceConfigurationType.MessageTemplateNotFound,
      templateId,
    },
  });
}
```

**优先级说明**：
1. `useDraft=true` 且存在 `draft` → 使用 draft
2. 否则使用 `definition`
3. 都不存在 → 错误
4. ⚠️ **完全没有** `zh-CN` → `en-US` → `default` 的语言版本 fallback 链

#### 第 9-11 层：渠道发送与渲染

以 **Email** 为例（`messaging.ts:889-969`）：
```typescript
// 渲染前：模板 ID 已固定
const identifierKey =
  messageTemplateDefinition.identifierKey ??
  CHANNEL_IDENTIFIERS[ChannelType.Email];

// 标识符值检查（无语言参与）
const identifier = userPropertyAssignments[identifierKey];
if (!identifier || typeof identifier !== "string") {
  return err({ type: MessageSkippedType.MissingIdentifier, identifierKey });
}

// 批量渲染（只做变量绑定，不选模板）
const renderedValues = renderValues({
  templates: {
    from: { contents: messageTemplateDefinition.from },
    subject: { contents: messageTemplateDefinition.subject },
    body: { contents: emailBody, mjml: true },
    // ...
  },
  userProperties: userPropertyAssignments,  // ← language 作为普通变量
  workspaceId,
  subscriptionGroupId: subscriptionGroupDetails?.id,
  identifierKey,
  secrets: subscriptionGroupSecret,
  tags: messageTags,
  isPreview,
  messageId,
});
```

**renderLiquid**（`liquid.ts:269-285`）：
```typescript
const context = {
  user: userProperties,       // ← language 在 user 对象中
  workspace_id: workspaceId,
  subscription_group_id: subscriptionGroupId,
  identifier_key: identifierKey,
  secrets,
  tags,
  is_preview: isPreview,
  message_id: messageId,
};
const liquidRendered = await liquidEngine.parseAndRender(template, context);
```

### 3.2.2 数据模型缺失证明

**MessageTemplate 表结构**（无 language 列）：

**文件位置**：`packages/isomorphic-lib/src/types.ts:1856-1864`

```typescript
const MessageTemplateResourceProperties = {
  workspaceId: Type.String(),
  id: Type.String(),
  name: Type.String(),
  type: Type.Enum(ChannelType),
  definition: Type.Optional(MessageTemplateResourceDefinition),
  draft: Type.Optional(MessageTemplateResourceDraft),
  updatedAt: Type.Number(),
  // ⚠️ 无 language 列！
  // ⚠️ 无 templateGroupId 列！
} as const;
```

**结论**：无法存储 `(name, language)` 复合唯一的多语言版本。

### 3.2.3 多语言调用链缺失总结

| 检查点 | 代码位置 | 是否有 language 参与 |
|--------|----------|---------------------|
| Journey 节点定义 | `types.ts:1167` | ❌ 固定 templateId |
| 参数类型定义 | `messaging.ts:510-543` | ❌ 无 language 字段 |
| 模板查询 | `messaging.ts:183-205` | ❌ 仅按 id 查询 |
| 模板版本选择 | `messaging.ts:484-489` | ❌ 仅 useDraft 控制 |
| 数据模型 | `types.ts:1856-1864` | ❌ 无 language 列 |
| 变量绑定 | `liquid.ts:269-285` | ✅ 作为普通 user 属性注入 |

### 3.3 现有的 Fallback 机制

系统中存在以下几种 **非语言相关** 的 fallback 机制：

#### 3.3.1 模板版本 Fallback（Draft → Definition）

**文件位置**：`packages/backend-lib/src/messaging.ts:484-489`

```typescript
const definitionFromDraft =
  useDraft && messageTemplate.draft
    ? messageTemplateDraftToDefinition(messageTemplate.draft).unwrapOr(null)
    : null;
const messageTemplateDefinition: MessageTemplateResourceDefinition | null =
  definitionFromDraft ?? messageTemplate.definition ?? null;
```

**优先级**：
1. `useDraft=true` 且存在 draft → 使用 draft
2. 否则使用 definition
3. 都不存在 → 抛出错误

#### 3.3.2 MJML 解析 Fallback

**文件位置**：`packages/backend-lib/src/liquid.ts:292-293`

```typescript
if (error.message.includes(MJML_NOT_PRESENT_ERROR)) {
  return liquidRendered;  // 回退到纯 HTML
}
```

#### 3.3.3 标识符键 Fallback

**文件位置**：`packages/backend-lib/src/messaging.ts:891-893`

```typescript
const identifierKey =
  messageTemplateDefinition.identifierKey ??
  CHANNEL_IDENTIFIERS[ChannelType.Email];
```

**默认标识符**：
- Email → `email`
- SMS → `phone`
- MobilePush → `deviceToken`

### 3.4 多语言支持的基础设施

虽然没有完整的多语言机制，但以下组件可用于未来扩展：

**默认用户属性**（`bootstrap.ts:245-251`）：

```typescript
{
  name: "language",
  workspaceId,
  definition: {
    type: UserPropertyDefinitionType.Trait,
    path: "language",
  },
  exampleValue: '"en-US"',
}
```

**配置项**（`config.ts:62`）：
```typescript
defaultLanguageUserPropertyId: Type.Optional(Type.String()),
```

### 3.5 模板中手动实现多语言

用户可以通过 Liquid 条件判断手动实现简单的多语言：

```liquid
{% if user.language == "zh-CN" %}
  你好，{{ user.name }}！
{% elsif user.language == "ja-JP" %}
  こんにちは、{{ user.name }}！
{% else %}
  Hello, {{ user.name }}!
{% endif %}
```

---

## 4. 跨渠道字符和尺寸限制校验逻辑

### 4.1 现有校验机制总览

| 校验类型 | 实现状态 | 校验时机 |
|----------|----------|----------|
| 严格变量检查 | ✅ 实现 | 渲染时 |
| UUID 格式校验 | ✅ 实现 | 模板保存时 |
| 名称唯一性 | ✅ 实现 | 模板保存时 |
| 标识符键存在性 | ✅ 实现 | 模板保存时 |
| Schema 结构校验 | ✅ 实现 | 保存和渲染时 |
| SMS 字符数限制 | ❌ 未实现 | - |
| 邮件主题长度 | ❌ 未实现 | - |
| 附件大小限制 | ❌ 未实现 | - |
| Webhook 负载大小 | ❌ 未实现 | - |

### 4.2 模板保存时的校验

**文件位置**：`packages/backend-lib/src/messaging.ts:227-382`

#### 4.2.1 UUID 格式校验

```typescript
if (data.id && !validateUuid(data.id)) {
  return err({
    type: UpsertMessageTemplateValidationErrorType.IdError,
    message: "Invalid message template id, must be a valid v4 UUID",
  });
}
```

#### 4.2.2 标识符键校验（Email/SMS）

**文件位置**：`packages/backend-lib/src/messaging.ts:239-263`

```typescript
const definitionToValidate = data.definition ?? data.draft;
if (
  definitionToValidate &&
  (definitionToValidate.type === ChannelType.Email ||
    definitionToValidate.type === ChannelType.Sms) &&
  "identifierKey" in definitionToValidate &&
  definitionToValidate.identifierKey
) {
  const { identifierKey } = definitionToValidate;
  const userPropertyExists = await db().query.userProperty.findFirst({
    where: and(
      eq(dbUserProperty.workspaceId, data.workspaceId),
      eq(dbUserProperty.name, identifierKey),
    ),
  });
  if (!userPropertyExists) {
    return err({
      type: UpsertMessageTemplateValidationErrorType.InvalidIdentifierKey,
      message: `User property "${identifierKey}" does not exist in the workspace`,
      identifierKey,
    });
  }
}
```

**校验逻辑**：
1. 只对 Email 和 SMS 模板生效
2. 检查指定的 `identifierKey` 是否存在于工作区的用户属性定义中
3. 不存在则返回 `InvalidIdentifierKey` 错误

#### 4.2.3 数据库约束校验

通过 PostgreSQL 约束实现：
- 名称唯一性（`UniqueConstraintViolation`）
- 外键约束（`ForeignKeyViolation`）

### 4.3 渲染时的校验

#### 4.3.1 严格变量模式

**文件位置**：`packages/backend-lib/src/liquid.ts:49`

```typescript
strictVariables: true
```

**行为**：
- 引用 `{{ user.nonExistentProperty }}` 会抛出错误
- 错误会被捕获并包装为 `MessageTemplateRenderError`

#### 4.3.2 标识符值校验

**文件位置**：`packages/backend-lib/src/messaging.ts:960-969`（Email）

```typescript
const identifier = userPropertyAssignments[identifierKey];
if (!identifier || typeof identifier !== "string") {
  return err({
    type: InternalEventType.MessageSkipped,
    variant: {
      type: MessageSkippedType.MissingIdentifier,
      identifierKey,
    },
  });
}
```

**SMS 版本**（`messaging.ts:1879-1902`）：支持 string 和 number 类型

#### 4.3.3 Webhook 特殊校验

**文件位置**：`packages/backend-lib/src/messaging.ts:2257-2282`

```typescript
const parsedBody = jsonParseSafe(renderedBody.value.body);
if (parsedBody.isErr()) {
  return err({
    type: InternalEventType.BadWorkspaceConfiguration,
    variant: {
      type: BadWorkspaceConfigurationType.MessageTemplateRenderError,
      field: "body",
      error: "Failed to parse webhook json payload",
    },
  });
}

const validatedBody = schemaValidateWithErr(parsedBody.value, ParsedWebhookBody);
if (validatedBody.isErr()) {
  return err({
    type: InternalEventType.BadWorkspaceConfiguration,
    variant: {
      type: BadWorkspaceConfigurationType.MessageTemplateRenderError,
      field: "body",
      error: `Failed to validate webhook json payload`,
    },
  });
}
```

**Webhook 校验流程**：
1. JSON 语法解析
2. Schema 结构验证（必须包含 `config`，可选 `secret`）

### 4.4 错误类型定义

**文件位置**：`packages/isomorphic-lib/src/types.ts:1894-1936`

```typescript
export enum UpsertMessageTemplateValidationErrorType {
  IdError = "IdError",
  UniqueConstraintViolation = "UniqueConstraintViolation",
  InvalidIdentifierKey = "InvalidIdentifierKey",
}
```

**运行时渲染错误**（`types.ts:4221-4229`）：

```typescript
export const MessageTemplateRenderError = Type.Object({
  type: Type.Literal(BadWorkspaceConfigurationType.MessageTemplateRenderError),
  field: Type.String(),    // 出错的字段名
  error: Type.String(),    // 错误信息
});
```

### 4.5 缺失的渠道特定校验

#### 4.5.1 SMS 字符限制

**当前状态**：未实现

**行业标准**：
- GSM-7 编码：单条 160 字符，多条每条 153 字符
- UCS-2 编码：单条 70 字符，多条每条 67 字符

**代码位置**：无相关校验，直接透传给 SMS 提供商

#### 4.5.2 邮件长度限制

**当前状态**：未实现

**常见限制**：
- 主题行：建议 50-78 字符
- 发件人名称：建议 < 32 字符

#### 4.5.3 附件限制

**当前状态**：仅验证文件类型，无大小限制

**文件位置**：`packages/backend-lib/src/messaging.ts:1080-1150`

```typescript
const file = schemaValidateWithErr(assignment, AppDataFileInternal);
// 仅验证结构，不验证大小
```

### 4.6 错误处理与用户反馈

**文件位置**：`packages/api/src/controllers/contentController.ts:320-563`

测试消息发送时，系统会将错误转换为用户友好的建议：

```typescript
case ChannelType.Sms: {
  switch (provider.type) {
    case SmsProviderType.Twilio: {
      const suggestions: string[] = [];
      if (provider.message) {
        suggestions.push(provider.message);
      } else {
        suggestions.push(
          "Failed to send SMS via Twilio. Check your Twilio configuration and the phone number format.",
        );
      }
      // ...
    }
  }
}
```

---

## 5. 核心代码文件索引

| 文件路径 | 主要职责 |
|----------|----------|
| `packages/backend-lib/src/liquid.ts` | Liquid 引擎配置、自定义标签、核心渲染函数 |
| `packages/backend-lib/src/messaging.ts` | 消息发送流程、模板选择、多字段渲染、校验逻辑 |
| `packages/backend-lib/src/messaging/sms.ts` | SMS 提供商配置管理 |
| `packages/backend-lib/src/messaging/email.ts` | 邮件相关工具（退订头等） |
| `packages/api/src/controllers/contentController.ts` | 模板 API 端点（渲染、保存、测试发送） |
| `packages/isomorphic-lib/src/types.ts` | 模板类型定义、错误类型、Schema 定义 |
| `packages/isomorphic-lib/src/messageTemplates.ts` | Draft/Definition 转换工具 |
| `packages/emailo/src/toMjml.ts` | Low Code 编辑器 JSON 到 MJML 的转换 |
| `packages/backend-lib/src/bootstrap.ts` | 默认用户属性定义（包括 language） |
| `packages/backend-lib/src/config.ts` | 配置项定义（包括 defaultLanguageUserPropertyId） |

---

## 总结

### 关键发现

1. **模板引擎**：使用 Liquid 而非 Handlebars，但语法兼容
2. **变量绑定**：通过 `renderLiquid` 注入，支持 `{{ user.xxx }}` 访问
3. **MJML 支持**：Liquid 渲染后转换，失败时有优雅降级
4. **多语言**：基础设施存在（language 用户属性），但完整调用链中无语言参与的模板选择或 fallback
5. **校验机制**：基础校验完善（变量、Schema、标识符），但缺少渠道特定限制

### 多语言缺失详解

经过完整调用链分析，多语言机制在三个层面缺失：

| 层面 | 现状 | 后果 |
|------|------|------|
| **数据模型** | `MessageTemplate` 表无 `language` 列 | 无法存储同一模板的多种语言版本 |
| **模板选择** | `findMessageTemplate({ id, channel })` 仅按 ID 查找 | 发送时无法根据 `user.language` 动态选择对应语言模板 |
| **版本路由** | Journey/Broadcast 节点绑定固定 `templateId` | 即使有多语言模板，消息节点也无法动态路由 |

### 各渠道限制校验现状

| 渠道 | 变量校验 | Schema 校验 | 标识符校验 | 内容长度/大小校验 | 发送功能状态 |
|------|---------|------------|-----------|------------------|-------------|
| **Email** | ✅ | ✅ | ✅ | ❌ | ✅ |
| **SMS** | ✅ | ✅ | ✅ | ❌（GSM-7/UCS-2 字符数） | ✅ |
| **Mobile Push** | ✅（模板级） | ✅（模板级） | ✅ | ❌（4KB FCM/APNS 限制） | ❌（未实现） |
| **Webhook** | ✅ | ✅ | - | ❌（JSON payload 大小） | ✅ |

### 建议改进方向

1. **多语言支持**：
   - 数据层：为 MessageTemplate 添加 `language` 列，引入 `templateGroupId`
   - 查询层：实现 `findMessageTemplate({ templateGroupId, language })` 并 fallback 到默认语言
   - 配置层：使用 `defaultLanguageUserPropertyId` 确定默认语言用户属性
   - 发送层：在 `getSendMessageModels()` 中读取 `userPropertyAssignments[languageProp]` 进行模板选择

2. **渠道限制校验**：
   - **SMS**：添加 GSM-7/UCS-2 字符计数和分段警告
   - **Mobile Push**：实现发送功能，并添加 FCM/APNS 4KB payload 检查
   - **Email**：添加主题行、发件人名称长度建议
   - **Webhook**：添加 payload 大小检查
   - **附件**：添加文件大小检查
