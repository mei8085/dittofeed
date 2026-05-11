# SMS 与 Push 多 Provider 统一抽象分析报告

## 1. 概述

Dittofeed 采用了一套统一的消息发送架构，对 SMS（短信）和 MobilePush（移动推送）两条 channel 实现了多 provider 的抽象适配。本文档详细分析其设计理念、实现方式和核心策略。

---

## 2. Channel 与 Provider 体系架构

### 2.1 Channel 类型定义

系统定义了四种核心 channel 类型（`packages/isomorphic-lib/src/types.ts:130-135`）：

```typescript
export const ChannelType = {
  Email: "Email",
  MobilePush: "MobilePush",
  Sms: "Sms",
  Webhook: "Webhook",
} as const;
```

每个 channel 对应特定的标识符键（`packages/isomorphic-lib/src/channels.ts:3-10`）：
- Email → `email`
- MobilePush → `deviceToken`
- Sms → `phone`

### 2.2 Provider 枚举体系

#### SMS Provider（`packages/isomorphic-lib/src/types.ts:180-184`）：
- **Twilio**：主流商业短信服务
- **SignalWire**：替代短信提供商
- **Test**：测试环境模拟 provider

#### MobilePush Provider（`packages/isomorphic-lib/src/types.ts:173-176`）：
- **Firebase (FCM)**：Firebase Cloud Messaging
- **Test**：测试环境模拟 provider

---

## 3. 统一抽象层设计

### 3.1 统一发送入口

核心发送逻辑集中在 `packages/backend-lib/src/messaging.ts`，通过统一的 `sendMessage` 函数分发到各 channel：

```typescript
// 统一发送参数类型
export type SendMessageParameters =
  | SendMessageParametersEmail
  | SendMessageParametersSms
  | SendMessageParametersWebhook
  | SendMessageParametersMobilePush;
```

`sendMessage` 函数的注释（`packages/backend-lib/src/messaging.ts:2390-2395`）明确了设计意图：

```typescript
/**
 * Send a message to a channel
 * Re-tryable errors will be thrown. Non-retryable errors will be returned as
 * error objects.
 * @param params - The parameters for the message
 * @returns The result of the message send
 */
```

**关键点**：
- **可重试错误**：直接 `throw` 抛出异常，由上层 Temporal 工作流引擎处理自动重试
- **不可重试错误**：以 `Result.err()` 形式返回，调用方自行决定如何处理

### 3.2 发送流程抽象

所有 channel 的发送流程遵循相同的模式：

1. **获取模板与模型**：调用 `getSendMessageModels` 验证模板和订阅状态
2. **获取 Provider 配置**：从数据库或父工作区层级获取 provider 配置
3. **模板渲染**：使用 `renderValues` + Liquid 引擎渲染消息内容
4. **解析接收者标识**：从用户属性中提取手机号/设备 token
5. **Provider 分发**：根据 provider 类型调用具体实现
6. **结果统一封装**：转换为标准的 `BackendMessageSendResult`

### 3.3 Provider 选择策略

SMS Provider 选择逻辑（`packages/backend-lib/src/messaging.ts:598-655`）：

1. **优先使用覆盖值**：若调用方指定 `providerOverride`，使用指定 provider
2. **默认 Provider**：查询 `defaultSmsProvider` 表获取工作区默认 provider
3. **层级继承**：当前工作区未配置时，向父工作区（`parentWorkspaceId`）查找
4. **配置验证**：使用 TypeBox schema 验证 provider 配置完整性

---

## 4. 统一响应结构：BackendMessageSendResult

### 4.1 类型定义

```typescript
// packages/isomorphic-lib/src/types.ts:4558-4561
export type BackendMessageSendResult = Result<
  MessageSuccess,
  MessageSendFailure
>;
```

采用 `neverthrow.Result` 模式，成功时返回 `MessageSuccess`，失败时返回 `MessageSendFailure`。

### 4.2 成功响应：MessageSuccess

成功响应是一个联合类型，包含两种情况：

#### 4.2.1 消息发送成功（MessageSendSuccess）

```typescript
{
  type: InternalEventType.MessageSent,  // "DFInternalMessageSent"
  variant: {
    type: ChannelType.Sms | ChannelType.MobilePush,  // 具体 channel 类型
    // SMS 特有字段
    to: string,           // 接收者号码
    body: string,         // 消息内容
    provider: {
      type: SmsProviderType.Twilio | SmsProviderType.SignalWire,
      sid?: string,       // Provider 返回的消息 ID
      status?: string,    // 发送状态
    },
    // Push 特有字段（待完善）
  }
}
```

#### 4.2.2 消息跳过（MessageSkipped）

某些场景下消息被合法跳过，也算作"成功"流程：

```typescript
{
  type: InternalEventType.MessageSkipped,  // "DFMessageSkipped"
  variant: {
    type: MessageSkippedType.SubscriptionState | MessageSkippedType.MissingIdentifier,
    // 订阅状态跳过
    action?: "Subscribe" | "Unsubscribe",
    subscriptionGroupType?: "OptIn" | "OptOut",
    // 缺少标识跳过
    identifierKey?: string,
  }
}
```

### 4.3 失败响应：MessageSendFailure

失败响应分为三大类（`packages/isomorphic-lib/src/types.ts:4546-4550`）：

```typescript
export const MessageSendFailure = Type.Union([
  MessageSendBadConfiguration,   // 配置错误
  MessageServiceFailure,         // 服务调用失败
  MessageSkippedFailure,         // 消息跳过（也归为失败？需注意）
]);
```

---

## 5. 不同 Provider 的结果映射

### 5.1 SMS Provider 结果映射

#### 5.1.1 Twilio 映射（`packages/backend-lib/src/destinations/twilio.ts`）

**成功映射**：
- Twilio SDK 返回 `response.sid`
- 映射为：`{ type: SmsProviderType.Twilio, sid: result.value.sid }`

**失败映射**：
- 捕获 `RestException` 或通用 `Error`
- 映射为：`{ type: SmsProviderType.Twilio, message: error.message }`

**注意**：Twilio 目前没有细粒度的可重试错误码分类，所有错误都以 `Result.err()` 返回，被标记为不可重试。

#### 5.1.2 SignalWire 映射（`packages/backend-lib/src/destinations/signalwire.ts`）

**成功映射**：
- SignalWire SDK 返回 `{ sid, status, errorCode, errorMessage }`
- 当 `!errorCode` 时判定为成功
- 映射为：`{ type: SmsProviderType.SignalWire, sid, status }`

**失败映射**：
- 当存在 `errorCode` 或捕获到异常时
- **可重试错误**：抛出异常（throw）而非返回 Result，由上层工作流引擎自动重试
- **不可重试错误**：返回 `Result.err()`，包含详细错误码

```typescript
{
  type: SmsProviderType.SignalWire,
  errorCode: string,      // Provider 原始错误码
  errorMessage?: string,  // 错误描述
  status: string,         // HTTP 状态
}
```

### 5.2 Push Provider 结果映射与"未实现"状态分析

#### 5.2.1 FCM Provider 基础实现

FCM provider 的基础实现已存在于 `packages/backend-lib/src/destinations/fcm.ts`：

```typescript
export async function sendNotification({
  key,
  ...message
}: Message & { key: string }): Promise<Result<string, Error>> {
  // 成功：Result.ok(fcmMessageId)
  // 失败：Result.err(Error)
}
```

- **成功**：`Result<string, Error>`（string 为 FCM message ID）
- **失败**：包含配置解析错误或 FCM 服务错误

#### 5.2.2 MobilePush "未实现"的实际落点

**核心代码位置**：`packages/backend-lib/src/messaging.ts:2414-2415`

```typescript
case ChannelType.MobilePush:
  throw new Error("not implemented");
```

这是一个直接的 `throw`，不返回 `Result` 类型。

#### 5.2.3 批量发送中的错误归类

在 `batchMessageUsers` 函数中（`packages/backend-lib/src/messaging.ts:2593-2770`），每个用户的发送逻辑被包裹在 `try-catch` 块中：

```typescript
try {
  const messageResult = await sendMessage(sendMessageParams);
  // ... 处理 Result 结果
} catch (userError) {
  logger().info(
    { err: userError, userId, workspaceId, templateId },
    "Failed to send message to user",
  );

  return [
    {
      userId: user.id,
      type: BatchMessageUsersResultTypeEnum.RetryableError,  // ← 关键：归类为可重试
      messageId,
      error: {
        message:
          userError instanceof Error
            ? userError.message
            : "Unknown error",
      },
    },
    null,
  ] satisfies MessageSendResultWithResponseItem;
}
```

**归类逻辑**：

| 错误来源 | 捕获方式 | 归类结果 |
|---------|---------|---------|
| `Result.err()` 且 `isNonRetryableError(error)` | 不进入 catch | `NonRetryableError` |
| `Result.err()` 且 `!isNonRetryableError(error)` | 不进入 catch | `RetryableError`（仅 MessageSkipped） |
| `throw new Error()`（如 MobilePush "not implemented"） | catch 块捕获 | `RetryableError` |
| SignalWire 可重试错误（throw） | catch 块捕获 | `RetryableError` |

**MobilePush "未实现" 的实际归类**：
- 由于 `sendMessage` 中直接 `throw new Error("not implemented")`
- 该异常被 `batchMessageUsers` 的 catch 块捕获
- **被归类为 `RetryableError`** 而非 `NonRetryableError`
- 这意味着批量发送后，调用方需要自行决定是否重试这些"未实现"的消息

**设计问题**：
- MobilePush "未实现" 实际上是一个配置/功能缺失问题，理论上应归类为 `NonRetryableError`
- 当前实现通过 `throw` 触发 catch 块，导致被误归类为 `RetryableError`
- 正确的做法应该是返回 `Result.err()` 类型的错误

#### 5.2.4 Journey 工作流中的处理

在 Journey 活动中（`packages/backend-lib/src/journeys/userWorkflow/activities.ts:258-300`），同样有 try-catch 处理：

```typescript
try {
  const result = await sender({ ... });
  return result;
} catch (senderError) {
  const activityInfo = Context.current().info;
  const isLastAttempt = activityInfo.attempt >= (retryCount ?? 3);

  if (isLastAttempt) {
    // 最后一次重试失败，返回 JourneyEarlyExit
    return err({
      type: InternalEventType.JourneyEarlyExit,
      message: `Message failed after maximum retry attempts: ${senderErrorString}`,
    });
  }

  // 不是最后一次尝试，重新抛出让 Temporal 自动重试
  throw senderError;
}
```

**MobilePush 在 Journey 中的行为**：
1. 第一次调用 `sendMessage` → `throw new Error("not implemented")`
2. 被 catch 捕获 → `isLastAttempt = false` → 重新 throw
3. Temporal 根据 `retryPolicy` 自动重试（默认 3 次）
4. 第 3 次重试后 → `isLastAttempt = true` → 返回 `JourneyEarlyExit`
5. Journey 节点根据 `skipOnFailure` 决定是跳过还是终止

**净效果**：MobilePush "未实现" 会触发最多 3 次无意义的重试，然后以 Journey 退出告终。

---

## 6. 错误码归类策略

### 6.1 错误分类层级

系统采用三层错误分类体系：

| 层级 | 类型 | 说明 |
|------|------|------|
| L1 | MessageSendFailure | 顶层失败类型，包含三大子类 |
| L2 | MessageSendBadConfiguration / MessageServiceFailure / MessageSkippedFailure | 错误大类 |
| L3 | 具体 variant | 详细错误信息和 provider 特定字段 |

### 6.2 配置错误（MessageSendBadConfiguration）

**触发场景**：
- 模板不存在或类型不匹配
- 模板渲染失败（Liquid 语法错误）
- Provider 未配置
- Provider 配置字段缺失（如 Twilio 的 `accountSid`）
- 接收者标识不存在

**错误码枚举**（`packages/isomorphic-lib/src/types.ts:4209-4219`）：
- `MessageTemplateNotFound`
- `MessageTemplateMisconfigured`
- `MessageTemplateRenderError`（含 `field` 和 `error` 详情）
- `MessageServiceProviderNotFound`
- `MessageServiceProviderMisconfigured`
- `MissingIdentifier`

**重试策略**：此类错误**不可重试**（见 `isNonRetryableError` 函数）

### 6.3 服务调用失败（MessageServiceFailure）

#### 6.3.1 通用结构

```typescript
{
  type: InternalEventType.MessageFailure,  // "DFMessageFailure"
  variant: {
    type: ChannelType.Sms | ChannelType.Email,
    provider: ProviderSpecificFailure,  // Provider 特定错误结构
  }
}
```

#### 6.3.2 SMS Provider 特定错误

**Twilio 错误**（`packages/isomorphic-lib/src/types.ts:4431-4434`）：
```typescript
{
  type: SmsProviderType.Twilio,
  message?: string,  // 原始错误消息
}
```

**SignalWire 错误**（`packages/isomorphic-lib/src/types.ts:4440-4445`）：
```typescript
{
  type: SmsProviderType.SignalWire,
  errorCode: string,        // SignalWire 错误码（如 "21212"）
  errorMessage?: string,    // 错误描述
  status: string,           // HTTP 状态码
}
```

#### 6.3.3 可重试错误识别

SignalWire 实现了明确的可重试错误判断（`packages/backend-lib/src/destinations/signalwire.ts:18-25, 52-60`）：

```typescript
// 可重试错误码集合
const SIGNAL_WIRE_RETRYABLE_ERROR_CODES = new Set([
  "30008",  // Application error
  "30022",  // Throughput limit exceeded（限频）
  "30027",  // T-Mobile limit exceeded（运营商限频）
]);

// 判断逻辑
function isRetryableSignalWireError(error: SignalWireApiError): boolean {
  if (error.status >= 500) return true;  // 服务端 5xx 错误
  if (SIGNAL_WIRE_RETRYABLE_ERROR_CODES.has(error.code)) return true;
  return false;
}
```

**可重试错误处理**：抛出异常而非返回 Result，由上层工作流引擎处理重试

**不可重试错误**：直接返回 `Result.err()`，标记为 `NonRetryableMessageSendFailure`

### 6.4 消息跳过（MessageSkippedFailure）

**触发场景**：
- 用户未订阅该 channel（`SubscriptionState`）
- 用户缺少必要的接收标识（如手机号）

**重试策略**：此类错误**可重试**（见 `isNonRetryableError` 函数逻辑）

---

## 7. 三层失败回退机制详解

系统的失败回退策略分为三个层级：

| 层级 | 机制 | 触发条件 | 实现位置 |
|------|------|---------|---------|
| L1 | Provider 内重试 | SignalWire 可重试错误码 | `packages/backend-lib/src/destinations/signalwire.ts` |
| L2 | 工作流层重试 | 任何 throw 的异常 + retryPolicy | `packages/backend-lib/src/journeys/userWorkflow.ts:955-960` |
| L3 | Provider 间 Failover | 当前 provider 不可重试失败 | **当前不存在** |

### 7.1 第一层：Provider 内重试（Intra-Provider Retry）

#### 7.1.1 设计意图

在单个 provider 内部识别可重试的瞬时错误（如限频、服务端 5xx），通过抛出异常让上层调度器处理重试。

#### 7.1.2 SignalWire 实现

**代码位置**：`packages/backend-lib/src/destinations/signalwire.ts:62-140`

```typescript
export async function sendSms({ ... }): Promise<Result<SmsSignalWireSuccess, MessageSignalWireServiceFailure>> {
  try {
    const { sid, status, errorCode, errorMessage } = await client.messages.create({ ... });

    if (!errorCode) {
      return ok({ sid, status, type: SmsProviderType.SignalWire });
    }

    // 检查是否为可重试错误码
    if (SIGNAL_WIRE_RETRYABLE_ERROR_CODES.has(errorCode.toString())) {
      // 可重试：抛出异常，由上层重试
      throw new Error(
        `transient signalwire error: code=${errorCode} message=${errorMessage}`,
      );
    }

    // 不可重试：返回 Result.err()
    return err({
      type: SmsProviderType.SignalWire,
      errorCode: errorCode.toString(),
      errorMessage,
      status,
    });
  } catch (error) {
    const signalWireErr = error as SignalWireApiError;
    if (isRetryableSignalWireError(signalWireErr)) {
      // HTTP 5xx 等：抛出异常
      throw new Error(
        `transient signalwire error: code=${signalWireErr.code} message=${signalWireErr.message}`,
      );
    }
    // 其他错误：返回 Result.err()
    return err({
      type: SmsProviderType.SignalWire,
      errorCode: signalWireErr.code,
      errorMessage: signalWireErr.message,
      status: signalWireErr.status.toString(),
    });
  }
}
```

#### 7.1.3 Twilio 现状

**代码位置**：`packages/backend-lib/src/destinations/twilio.ts:46-111`

```typescript
export async function sendSms({ ... }): Promise<Result<{ sid: string }, RestException | Error>> {
  try {
    const response = await client.messages.create(createPayload);
    return ok({ sid: response.sid });
  } catch (e) {
    if (e instanceof RestException) {
      return err(e);  // 所有 RestException 都返回 Result.err()
    }
    const error = e as Error;
    return err(error);  // 所有其他错误也返回 Result.err()
  }
}
```

**问题**：Twilio 没有细粒度的可重试判断，所有错误都以 `Result.err()` 返回，被 `isNonRetryableError` 判定为不可重试。这意味着 Twilio 的限频错误（HTTP 429）也不会触发自动重试。

### 7.2 第二层：工作流层重试（Workflow-Level Retry）

#### 7.2.1 Temporal Activity 重试策略

**代码位置**：`packages/backend-lib/src/journeys/userWorkflow.ts:955-960`

```typescript
const { sendMessageV2 } = proxyActivities<typeof activities>({
  startToCloseTimeout: "2 minutes",
  retry: {
    maximumAttempts: currentNode.retryCount ?? 3,  // 默认 3 次
  },
});
```

**重试配置**：
- `startToCloseTimeout: "2 minutes"`：单次活动执行超时
- `maximumAttempts`：从 Journey 节点配置读取，默认为 3
- Temporal 默认使用指数退避（initialInterval: 1s, backoffCoefficient: 2.0）

#### 7.2.2 Activity 内的重试熔断

**代码位置**：`packages/backend-lib/src/journeys/userWorkflow/activities.ts:258-300`

```typescript
try {
  const result = await sender({ ... });
  return result;
} catch (senderError) {
  const activityInfo = Context.current().info;
  const isLastAttempt = activityInfo.attempt >= (retryCount ?? 3);

  if (isLastAttempt) {
    // 最后一次尝试：返回 JourneyEarlyExit 而非继续抛出
    logger().error("sender failed after maximum retry attempts", { ... });
    return err({
      type: InternalEventType.JourneyEarlyExit,
      message: `Message failed after maximum retry attempts: ${senderErrorString}`,
    });
  }

  // 不是最后一次：重新抛出，让 Temporal 调度重试
  throw senderError;
}
```

**工作流层重试流程**：

```
sendMessage throw 异常
        ↓
Temporal 根据 retryPolicy 调度重试（指数退避）
        ↓
第 N 次重试
        ↓
    ┌─── N < maxAttempts? ───┐
    │                        │
   是                       否
    │                        │
    ↓                        ↓
重新 throw         返回 JourneyEarlyExit
让 Temporal 继续         （节点退出）
```

#### 7.2.3 与 Provider 内重试的关系

```
SignalWire 可重试错误
        ↓
Provider 层 throw 异常
        ↓
Activity catch 捕获
        ↓
    ┌─── isLastAttempt? ───┐
    │                       │
   否                      是
    │                       │
    ↓                       ↓
重新 throw          返回 JourneyEarlyExit
让 Temporal 重试         （退出节点）
```

**关键点**：Provider 内重试和工作流层重试是**串联关系**：
1. Provider 层决定哪些错误是"可重试"的（通过 throw vs return err）
2. 工作流层决定最多重试多少次（通过 retryPolicy）

### 7.3 第三层：Provider 间 Failover（当前不存在）

#### 7.3.1 现状分析

**代码证据**：

1. `sendSms` 函数（`packages/backend-lib/src/messaging.ts:1782-2165`）中只有 `switch (smsProvider.type)`，一旦选定 provider 就不会切换
2. 没有 `getNextProvider()`、`tryNextProvider()` 等逻辑
3. 没有 provider 优先级列表的概念
4. 错误处理中没有"失败后尝试下一个 provider"的分支

**现有能力**：
- 支持通过 `providerOverride` 参数**手动**指定 provider
- 支持在 Journey 节点配置中指定 `providerOverride`
- 支持工作区默认 provider 配置

**这意味着**：如果 Twilio 失败（不管什么原因），系统**不会**自动尝试 SignalWire。调用方必须手动发起第二次调用并指定 `providerOverride: SmsProviderType.SignalWire`。

#### 7.3.2 Failover 的实现思路（建议）

如果要实现自动 failover，可能的设计方案：

```typescript
// 伪代码：可能的 failover 实现
export async function sendSmsWithFailover(params): Promise<BackendMessageSendResult> {
  const providers = [SmsProviderType.Twilio, SmsProviderType.SignalWire]; // 优先级列表
  
  for (const providerType of providers) {
    try {
      const result = await sendSms({
        ...params,
        providerOverride: providerType,
      });
      
      if (result.isOk()) {
        return result; // 成功，直接返回
      }
      
      // 检查是否为不可重试错误，如果是则尝试下一个 provider
      if (isNonRetryableError(result.error)) {
        logger().info({ providerType }, "Provider failed, trying next...");
        continue;
      }
      
      // 可重试错误，让上层工作流处理重试
      return result;
    } catch (e) {
      // Provider 内可重试错误，让上层工作流处理
      throw e;
    }
  }
  
  // 所有 provider 都失败
  return err({
    type: InternalEventType.MessageFailure,
    variant: {
      type: ChannelType.Sms,
      provider: { type: "AllProvidersFailed" },
    },
  });
}
```

**Failover 决策表**：

| 错误类型 | 当前 provider 行为 | Failover 行为 |
|---------|-------------------|--------------|
| `Result.err()` + NonRetryable | 返回错误 | ✅ 切换到下一个 provider |
| `Result.err()` + Retryable（MessageSkipped） | 返回错误 | ❌ 不切换，这是用户侧问题 |
| `throw`（可重试异常） | 抛出异常 | ❌ 不切换，让工作流重试当前 provider |

---

## 8. 跨 Channel 速率与限频协调

### 8.1 限频现状

#### 8.1.1 Provider 层面的限频识别

**SignalWire**（`packages/backend-lib/src/destinations/signalwire.ts:18-25`）：

```typescript
const SIGNAL_WIRE_RETRYABLE_ERROR_CODES = new Set([
  "30022",  // Throughput limit exceeded
  "30027",  // T-Mobile limit exceeded
]);
```

- 明确识别限频错误码
- 这些错误被标记为**可重试**（throw 异常）
- 由 Temporal 工作流的指数退避机制处理延迟重试

**Twilio**：
- 目前没有细粒度的限频错误识别
- 所有 `RestException` 都被当作不可重试错误

**FCM**：
- 完整实现尚未接入主流程
- FCM 有 HTTP 429（Too Many Requests）响应，但当前代码未处理

#### 8.1.2 系统层面的限频协调

**现状**：
- 未发现专门的跨 channel 限频协调器
- 未发现全局的消息发送速率限制器
- 未发现按 provider 分配配额的机制
- 主要依赖 Temporal 工作流引擎的活动调度能力

**批量发送中的并发**（`packages/backend-lib/src/messaging.ts:2588-2590`）：

```typescript
const messagePromises: Promise<MessageSendResultWithResponseItem>[] =
  users.map(async (user) => {
    // 每个用户一个独立的 Promise
    // Promise.all 会并行执行所有发送
  });
```

批量发送使用 `Promise.all` 并行执行，没有内置的并发控制或速率限制。

### 8.2 批量发送协调

批量发送通过 `batchMessageUsers` 函数处理（`packages/backend-lib/src/messaging.ts:2536+`）：

**协调策略**：
1. 并行获取所有用户的订阅状态和属性
2. 为每个用户创建独立的发送 Promise
3. 结果分类为：Success / Skipped / RetryableError / NonRetryableError

```typescript
// 批量结果类型
export const BatchMessageUsersResultTypeEnum = {
  Success: "Success",
  Skipped: "Skipped",
  RetryableError: "RetryableError",
  NonRetryableError: "NonRetryableError",
} as const;
```

**批量发送中的错误追踪**（`packages/backend-lib/src/messaging.ts:2787-2834`）：

```typescript
const trackEvents: KnownBatchTrackData[] = messageResults.flatMap(
  ([responseResult, backendMessageSendResult]) => {
    if (!backendMessageSendResult) return [];  // 被 catch 的异常不追踪事件
    if (responseResult.type === BatchMessageUsersResultTypeEnum.RetryableError) return [];  // 可重试错误也不追踪
    
    // 只有成功或不可重试错误才会记录到用户事件
    // ...
  },
);
```

**事件追踪规则**：

| 结果类型 | 是否记录用户事件 | 说明 |
|---------|-----------------|------|
| Success | ✅ 是 | `MessageSent` 或 `MessageSkipped` |
| Skipped | ✅ 是 | `MessageSkipped` |
| NonRetryableError | ✅ 是 | `BadWorkspaceConfiguration` 或 `MessageFailure` |
| RetryableError | ❌ 否 | 等待后续重试成功后再记录 |
| throw 捕获（如 MobilePush） | ❌ 否 | `backendMessageSendResult = null`，不追踪 |

### 8.3 限频改进建议

**建议实现的协调机制**：

1. **Provider 级限频**：
   - 为每个 provider 配置 QPS 限制
   - 使用令牌桶或漏桶算法
   - 跨进程共享限频状态（如 Redis）

2. **Channel 级优先级**：
   - 定义 SMS vs Push 的发送优先级
   - 高优先级消息优先占用配额

3. **动态退避**：
   - 根据 provider 返回的 `Retry-After` 头动态调整
   - 连续失败时增加退避时间

4. **并发控制**：
   - 批量发送中添加 `p-limit` 或类似机制
   - 限制并行发送数量

---

## 9. 关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| Channel 类型定义 | `packages/isomorphic-lib/src/types.ts` | 130-135 |
| Provider 枚举 | `packages/isomorphic-lib/src/types.ts` | 173-184 |
| 统一发送入口 | `packages/backend-lib/src/messaging.ts` | 2397-2420 |
| sendMessage throw 注释 | `packages/backend-lib/src/messaging.ts` | 2390-2395 |
| SMS 发送实现 | `packages/backend-lib/src/messaging.ts` | 1782-2165 |
| Provider 选择逻辑 | `packages/backend-lib/src/messaging.ts` | 598-655 |
| 响应类型定义 | `packages/isomorphic-lib/src/types.ts` | 4166-4561 |
| 错误类型定义 | `packages/isomorphic-lib/src/types.ts` | 4209-4550 |
| 可重试判断 | `packages/backend-lib/src/messaging.ts` | 2516-2529 |
| 指数退避重试 | `packages/backend-lib/src/retry.ts` | 4-39 |
| Twilio Provider | `packages/backend-lib/src/destinations/twilio.ts` | 46-111 |
| SignalWire Provider | `packages/backend-lib/src/destinations/signalwire.ts` | 62-140 |
| SignalWire 可重试码 | `packages/backend-lib/src/destinations/signalwire.ts` | 18-25 |
| FCM Provider | `packages/backend-lib/src/destinations/fcm.ts` | 31-46 |
| MobilePush not implemented | `packages/backend-lib/src/messaging.ts` | 2414-2415 |
| 批量发送 | `packages/backend-lib/src/messaging.ts` | 2536-2837 |
| 批量发送 catch 块 | `packages/backend-lib/src/messaging.ts` | 2745-2770 |
| Journey Activity 重试 | `packages/backend-lib/src/journeys/userWorkflow/activities.ts` | 258-300 |
| Temporal retryPolicy | `packages/backend-lib/src/journeys/userWorkflow.ts` | 955-960 |

---

## 10. 总结与展望

### 10.1 当前优势

1. **清晰的类型抽象**：使用 TypeBox 和 neverthrow 构建了强类型的响应结构
2. **统一的发送流程**：所有 channel 遵循相同的获取模板→渲染→发送→封装模式
3. **明确的错误分类**：可重试/不可重试界限清晰，便于上层调度
4. **Provider 配置灵活**：支持工作区默认、调用覆盖、父级继承三级配置
5. **两层重试机制**：Provider 内可重试错误识别 + Temporal 工作流自动重试

### 10.2 待改进点

#### 10.2.1 MobilePush 相关

1. **MobilePush 实现不完整**：FCM provider 存在但发送逻辑未接入主流程
2. **"未实现"错误归类不当**：当前通过 throw 导致被误归类为 RetryableError，应返回 Result.err()

#### 10.2.2 重试机制

1. **Twilio 可重试判断缺失**：仅 SignalWire 实现了细粒度的可重试错误码识别，Twilio 的 429/5xx 错误不会触发自动重试
2. **缺少 Provider 间 Failover**：provider 失败时无法自动切换到备用 provider

#### 10.2.3 限频协调

1. **缺少跨 channel 限频**：无全局速率控制，完全依赖 provider 自身限制
2. **批量发送无并发控制**：Promise.all 并行执行，可能超过 provider QPS 限制

### 10.3 建议优先级

| 优先级 | 改进项 | 说明 |
|--------|--------|------|
| P0 | 完成 MobilePush 发送流程 | 打通 FCM 到统一发送入口，确保返回 Result 而非 throw |
| P0 | 修正 MobilePush 错误归类 | 改为返回 `BadWorkspaceConfiguration` 类型的错误 |
| P1 | 实现 Provider 故障转移 | 配置 provider 优先级列表，不可重试错误时自动切换 |
| P1 | 完善 Twilio 可重试判断 | 参照 SignalWire 实现细粒度错误码分类（429、5xx 等） |
| P2 | 实现跨 channel 限频协调器 | 引入 Redis 限频，支持优先级队列和并发控制 |

### 10.4 错误归类速查表

| 场景 | throw vs return err | 批量发送归类 | Journey 行为 |
|------|---------------------|-------------|-------------|
| SignalWire 5xx 错误 | `throw` | RetryableError | Temporal 自动重试 |
| SignalWire 限频 (30022) | `throw` | RetryableError | Temporal 自动重试 |
| SignalWire 认证失败 | `return err()` | NonRetryableError | 不重试，节点退出/跳过 |
| Twilio 任何错误 | `return err()` | NonRetryableError | 不重试，节点退出/跳过 |
| 模板不存在 | `return err()` | NonRetryableError | 不重试，节点退出/跳过 |
| 用户缺少手机号 | `return err()` | RetryableError | 不重试（isNonRetryable=false 但不 throw） |
| MobilePush not implemented | `throw` | RetryableError | Temporal 重试 3 次后退出 |
