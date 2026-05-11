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

#### 5.1.2 SignalWire 映射（`packages/backend-lib/src/destinations/signalwire.ts`）

**成功映射**：
- SignalWire SDK 返回 `{ sid, status, errorCode, errorMessage }`
- 当 `!errorCode` 时判定为成功
- 映射为：`{ type: SmsProviderType.SignalWire, sid, status }`

**失败映射**：
- 当存在 `errorCode` 或捕获到异常时
- 映射为：
  ```typescript
  {
    type: SmsProviderType.SignalWire,
    errorCode: string,      // Provider 原始错误码
    errorMessage?: string,  // 错误描述
    status: string,         // HTTP 状态
  }
  ```

### 5.2 Push Provider 结果映射

**注意**：MobilePush 的完整发送实现尚未完成（`packages/backend-lib/src/messaging.ts:2414-2415`）：

```typescript
case ChannelType.MobilePush:
  throw new Error("not implemented");
```

但 FCM provider 的基础实现已存在于 `packages/backend-lib/src/destinations/fcm.ts`，其返回：
- **成功**：`Result<string, Error>`（string 为 FCM message ID）
- **失败**：包含配置解析错误或 FCM 服务错误

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
  "30022",  // Throughput limit exceeded
  "30027",  // T-Mobile limit exceeded
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

## 7. 失败回退（Fallback）与重试策略

### 7.1 重试机制

#### 7.1.1 指数退避重试

系统提供通用的指数退避重试工具（`packages/backend-lib/src/retry.ts`）：

```typescript
export async function retryExponential({
  sleep,
  check,
  baseDelay = 1000,   // 初始延迟 1s
  maxAttempts = 10,   // 最多 10 次
  logger,
}): Promise<boolean>;
```

**退避公式**：`delay = baseDelay * 2^attempt`

#### 7.1.2 可重试性判断

核心判断函数（`packages/backend-lib/src/messaging.ts:2516-2529`）：

```typescript
export function isNonRetryableError(
  error: MessageSendFailure,
): error is NonRetryableMessageSendFailure {
  switch (error.type) {
    case InternalEventType.MessageFailure:
      return true;      // 服务调用失败：不可重试
    case InternalEventType.BadWorkspaceConfiguration:
      return true;      // 配置错误：不可重试
    case InternalEventType.MessageSkipped:
      return false;     // 消息跳过：可重试（后续用户可能补充信息）
    default:
      assertUnreachable(error);
  }
}
```

### 7.2 Provider 级回退（待实现）

**当前状态**：系统尚未实现多 provider 之间的自动故障转移（failover）。

**现有能力**：
- 支持通过 `providerOverride` 参数手动指定备用 provider
- 支持工作区级默认 provider 配置

**建议的回退策略**：
1. 配置优先级列表（如 Twilio → SignalWire）
2. 当前 provider 返回不可重试错误时，自动切换到下一个
3. 可重试错误（如限流）则在当前 provider 重试

---

## 8. 跨 Channel 速率与限频协调

### 8.1 限频现状

**SMS Provider 层面**：
- SignalWire 明确识别限频错误码 `"30022"`（throughput limit）和 `"30027"`（T-Mobile limit）
- 这些错误被标记为**可重试**，由上层工作流处理延迟重试

**系统层面**：
- 未发现专门的跨 channel 限频协调器
- 未发现全局的消息发送速率限制器
- 主要依赖 Temporal 工作流引擎的活动调度能力

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

---

## 9. 关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| Channel 类型定义 | `packages/isomorphic-lib/src/types.ts` | 130-135 |
| Provider 枚举 | `packages/isomorphic-lib/src/types.ts` | 173-184 |
| 统一发送入口 | `packages/backend-lib/src/messaging.ts` | 2400+ |
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
| 批量发送 | `packages/backend-lib/src/messaging.ts` | 2536+ |

---

## 10. 总结与展望

### 10.1 当前优势

1. **清晰的类型抽象**：使用 TypeBox 和 neverthrow 构建了强类型的响应结构
2. **统一的发送流程**：所有 channel 遵循相同的获取模板→渲染→发送→封装模式
3. **明确的错误分类**：可重试/不可重试界限清晰，便于上层调度
4. **Provider 配置灵活**：支持工作区默认、调用覆盖、父级继承三级配置

### 10.2 待改进点

1. **MobilePush 实现不完整**：FCM provider 存在但发送逻辑未接入主流程
2. **缺少自动故障转移**：provider 失败时无法自动切换到备用 provider
3. **缺少跨 channel 限频**：无全局速率控制，完全依赖 provider 自身限制
4. **Twilio 可重试判断缺失**：仅 SignalWire 实现了细粒度的可重试错误码识别

### 10.3 建议优先级

| 优先级 | 改进项 | 说明 |
|--------|--------|------|
| P0 | 完成 MobilePush 发送流程 | 打通 FCM 到统一发送入口 |
| P1 | 实现 Provider 故障转移 | 配置 provider 优先级列表，失败自动切换 |
| P1 | 完善 Twilio 可重试判断 | 参照 SignalWire 实现细粒度错误码分类 |
| P2 | 实现跨 channel 限频协调器 | 引入 Redis 限频，支持优先级队列 |
