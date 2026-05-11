# Journey 全链路分析报告

## 1. 概述

DittoFeed 的 Journey 系统是一个基于 Temporal 的可视化工作流编排引擎，允许用户通过拖拽式界面设计复杂的客户旅程。系统将可视化的画布编辑转换为可执行的 Temporal 工作流，支持延迟、消息发送、条件分支等多种节点类型。

## 2. 核心架构组件

### 2.1 前端层 (Dashboard)
- **journeysBuilder.tsx**: 使用 ReactFlow 构建可视化画布编辑器
- **store.ts**: 管理画布状态（节点、边）
- **nodeTypes**: 各类节点的 UI 组件
- **useJourneyMutation.ts**: 调用后端 API 保存/更新 Journey

### 2.2 API 层
- **journeysController.ts**: REST API 控制器，处理 CRUD 操作
  - `GET /`: 获取 Journey 列表
  - `PUT /`: 创建或更新 Journey
  - `DELETE /`: 删除 Journey
  - `GET /stats`: 获取 Journey 统计数据

### 2.3 业务逻辑层 (Backend-Lib)
- **journeys.ts**: Journey 核心业务逻辑
  - `upsertJourney()`: 创建/更新 Journey，包含验证和状态转换
  - `triggerEventEntryJourneys()`: 基于事件触发 Journey
  - `triggerSegmentEntryJourney()`: 基于分段触发 Journey
  - `getJourneysStats()`: 计算 Journey 性能统计

- **journeys/userWorkflow.ts**: Temporal 工作流定义
  - `userJourneyWorkflow()`: 核心工作流函数，执行 Journey 节点
  - 信号处理: `segmentUpdateSignal`, `trackSignal`, `reEvaluateSegmentsSignal`

- **journeys/userWorkflow/activities.ts**: Temporal 活动实现
  - `sendMessageV2()`: 发送消息（邮件、短信、Webhook、推送）
  - `getSegmentAssignment()`: 获取用户分段分配
  - `isRunnable()`: 检查 Journey 是否可执行
  - `onNodeProcessedV2()`: 记录节点处理

### 2.4 数据模型

**JourneyDefinition 结构** (packages/isomorphic-lib/src/types.ts:1274-1278):
```typescript
{
  entryNode: EntryNode,      // 入口节点
  exitNode: ExitNode,        // 出口节点
  nodes: JourneyBodyNode[]   // 中间节点数组
}
```

**节点类型**:
- **Entry Nodes**: 
  - `SegmentEntryNode`: 基于用户进入分段触发
  - `EventEntryNode`: 基于特定事件触发
  
- **Body Nodes**:
  - `MessageNode`: 发送消息（邮件、短信、Webhook、移动推送）
  - `DelayNode`: 延迟执行（固定时间、本地时间、用户属性时间）
  - `SegmentSplitNode`: 条件分支（基于分段）
  - `WaitForNode`: 等待事件/分段（带超时）
  - `RandomCohortNode`: 随机分组
  - `RateLimitNode`: 速率限制

- **Exit Node**:
  - `ExitNode`: 旅程结束节点

## 3. 全链路流程

### 3.1 流程概览

```
[前端画布编辑] 
      ↓
[API 接收 + 验证]
      ↓
[数据库存储 (JSON definition)]
      ↓
[触发器检测 (事件/分段)]
      ↓
[Temporal 工作流启动]
      ↓
[节点顺序执行]
      ↓
[消息投递]
      ↓
[状态持久化]
```

### 3.2 详细步骤

#### 步骤 1: 画布编辑 (前端)

1. 用户在 `journeysBuilder.tsx` 中使用 ReactFlow 拖拽节点
2. 节点通过 `createNewConnections()` 函数连接
3. 状态管理通过 Zustand store (`appStore.ts`) 维护
4. 点击保存时调用 `useJourneyMutation`

#### 步骤 2: API 接收与验证

**路径**: `PUT /journeys` → `journeysController.ts:93-113`

1. 接收 `UpsertJourneyResource` 请求体
2. 调用 `upsertJourney()` 进行业务处理

#### 步骤 3: 业务验证与存储

**关键函数**: `upsertJourney()` (packages/backend-lib/src/journeys.ts:946-1248)

1. **验证阶段**:
   - 检查 UUID 格式
   - 获取依赖的分段资源
   - 执行约束检查: `getJourneyConstraintViolations()`
     - 不能同时有 EventEntryNode 和 WaitForNode
     - SegmentEntry 旅程不能使用 KeyedPerformed 分段

2. **状态转换验证**:
   - 不能暂停未启动的 Journey
   - 已启动的 Journey 不能设置为 NotStarted

3. **数据库存储**:
   - `definition` 字段: 序列化的 `JourneyDefinition` JSON
   - `status`: Journey 状态 (NotStarted/Running/Paused/Broadcast)
   - `draft`: 草稿版本（编辑时使用）

#### 步骤 4: 触发器检测

**两种触发方式**:

1. **事件触发** (`triggerEventEntryJourneys`):
   - 缓存 30 秒的 EventEntryNode 类型 Journey
   - 事件到达时匹配 `event` 名称
   - 使用 `signalWithStart` 启动工作流

2. **分段触发** (`triggerSegmentEntryJourney`):
   - 用户进入特定分段时触发
   - 使用 `signalWithStart` 传递分段更新信号

#### 步骤 5: Temporal 工作流启动

**工作流 ID 生成**:
- 普通 Journey: `user-journey-${userId}-${journeyId}`
- Keyed Journey: `user-journey-keyed-${workspaceId}-${journeyId}-${hash}`

**启动方式**:
```typescript
workflowClient.signalWithStart(userJourneyWorkflow, {
  taskQueue: "default",
  workflowId,
  args: [{ journeyId, definition, workspaceId, userId }],
  signal: segmentUpdateSignal,  // 或 trackSignal
  signalArgs: [segmentUpdate],
});
```

#### 步骤 6: 工作流执行

**核心循环**: `userJourneyWorkflow()` (packages/backend-lib/src/journeys/userWorkflow.ts:610-1113)

工作流按节点类型顺序执行:

1. **SegmentEntryNode**:
   - 检查用户是否已在分段中
   - 若不在，等待 `segmentUpdateSignal` 信号
   - 使用 `wf.condition()` 阻塞直到条件满足

2. **EventEntryNode**:
   - 直接进入下一节点
   - 事件数据通过 `trackSignal` 传递

3. **DelayNode**:
   - **Second**: 固定毫秒延迟 `sleep(delay)`
   - **LocalTime**: 计算下一个本地时间点
   - **UserProperty**: 基于用户属性值计算延迟

4. **SegmentSplitNode**:
   - 调用 `getSegmentAssignment()` 获取分段状态
   - 根据 `inSegment` 值选择 `trueChild` 或 `falseChild`

5. **WaitForNode**:
   - 等待多个分段中的任意一个
   - 带超时机制 `timeoutSeconds`
   - 使用 `wf.condition()` 实现等待

6. **MessageNode**:
   - 生成唯一 `messageId` (uuid4)
   - 调用 `sendMessageV2()` 活动
   - 根据 `skipOnFailure` 决定失败后是否继续
   - 可选 `syncProperties`: 等待属性计算完成

7. **RandomCohortNode**:
   - 确定性随机数 (`getRandomNumber` local activity)
   - 按百分比选择子节点

8. **ExitNode**:
   - 跳出节点循环
   - 记录最终节点处理

#### 步骤 7: 消息投递

**消息发送流程**: `sendMessageV2()` (packages/backend-lib/src/journeys/userWorkflow/activities.ts:303-394)

1. 检查 Journey 状态（必须是 Running）
2. 获取用户属性赋值
3. 检查订阅组（如果指定）
4. 调用对应渠道的发送器:
   - Email: SendGrid/Postmark/Resend/AmazonSES
   - SMS: Twilio/SignalWire
   - Webhook: HTTP 请求
   - MobilePush: FCM

5. 记录追踪事件 (`submitTrack`)
   - `MessageSent` / `MessageSkipped`
   - `JourneyNodeProcessed`

## 4. 配置快照机制

### 4.1 快照时机

Journey 配置在**启动时**快照到工作流参数中:

```typescript
// packages/backend-lib/src/journeys/userWorkflow/lifecycle.ts:70-95
await workflowClient.signalWithStart(userJourneyWorkflow, {
  args: [{
    journeyId,
    definition,  // <-- 快照的配置
    workspaceId,
    userId,
    // ...
  }],
});
```

### 4.2 快照内容

`JourneyDefinition` 包含完整的执行计划:
- 入口节点及其触发条件
- 所有中间节点及其配置
- 出口节点
- 节点间的父子关系（通过 `child`/`children` 字段）

### 4.3 不变性保证

**关键设计**: 工作流启动后，使用的是**快照版本**的配置，而非实时从数据库读取。

**优势**:
1. 运行中的旅程不受后续编辑影响
2. 保证同一用户的旅程执行一致性
3. 便于问题排查（知道用户运行时的配置版本）

**如何更新运行中的旅程**:
- 需要设计版本迁移机制
- 或让用户选择重新触发

## 5. Temporal 工作流执行机制

### 5.1 工作流状态机

工作流内部维护的状态:

```typescript
// packages/backend-lib/src/journeys/userWorkflow.ts:383-402
const segmentAssignments = new Map<string, ReceivedSegmentUpdate>();
const nodes = new Map<string, JourneyNode>();
const keyedEventIds = new Set<string>();
let currentNode: JourneyNode = definition.entryNode;
let nextNode: JourneyNode | null = null;
```

### 5.2 信号机制

工作流定义了三个信号处理器:

1. **`segmentUpdateSignal`** (line 51-52):
   - 接收分段状态更新
   - 处理版本检查（忽略过期更新）
   - 触发 `condition` 唤醒

2. **`trackSignal`** (line 444-527):
   - 接收事件追踪数据
   - 用于 Keyed Journey 的事件处理
   - 去重检查（`keyedEventIds`）

3. **`reEvaluateSegmentsSignal`** (line 551-599):
   - 重新评估指定分段
   - 用于手动触发重新检查

### 5.3 活动调用

工作流使用 `proxyActivities` 和 `proxyLocalActivities` 调用外部服务:

```typescript
// 远程活动（可重试）
const { sendMessageV2 } = proxyActivities<typeof activities>({
  startToCloseTimeout: "2 minutes",
  retry: { maximumAttempts: 3 },
});

// 本地活动（轻量级）
const { getRandomNumber } = wf.proxyLocalActivities<typeof activities>({
  startToCloseTimeout: "5 seconds",
});
```

## 6. 长时间运行与状态恢复

### 6.1 长时间运行机制

Temporal 原生支持长时间运行的工作流，DittoFeed 利用这一点实现:

1. **条件等待** (`wf.condition`):
   - SegmentEntryNode: 等待用户进入分段
   - WaitForNode: 等待特定事件或分段变化
   - 可以等待数天、数周甚至数月

2. **睡眠机制** (`sleep`):
   - DelayNode 实现延迟
   - Temporal 持久化定时器，不占用线程

3. **Continue-As-New** (line 1129):
   ```typescript
   if (await shouldReEnter({ journeyId, userId, workspaceId })) {
     if (shouldContinueAsNew) {
       await continueAsNew<typeof userJourneyWorkflow>(props);
     }
   }
   ```
   - 用于循环触发的 Journey（SegmentEntry + reEnter=true）
   - 避免工作流历史无限增长

### 6.2 状态恢复一致性

**Temporal 的保证**:
- 工作流代码必须是**确定性的**
- 所有非确定性操作（随机数、时间、外部调用）必须通过 activities 或 Temporal API

**DittoFeed 的实现**:

1. **随机数**: 使用 `getRandomNumber` local activity
2. **时间**: 使用 `Date.now()` 通过 Temporal 注入
3. **外部状态**: 通过 activities 访问，结果被记录在历史中

**恢复流程**:
1. Worker 崩溃或重启后，Temporal 自动重放历史
2. 工作流代码从第一条事件重新执行
3. Activities 结果从历史中读取（不重复调用）
4. 信号也被重放
5. 最终到达与崩溃前相同的状态

### 6.3 幂等性保证

**工作流 ID 唯一性**:
```typescript
function getUserJourneyWorkflowId({ userId, journeyId }) {
  return `user-journey-${userId}-${journeyId}`;
}
```

- 同一用户 + 同一 Journey = 同一工作流 ID
- 防止重复启动（`WorkflowExecutionAlreadyStartedError`）

**消息发送幂等**:
- `messageId` 在工作流内生成 (uuid4)
- 活动重试时使用相同 `messageId`
- 下游系统可基于此去重

### 6.4 版本兼容

**工作流版本管理**:
```typescript
export const UserJourneyWorkflowVersion = {
  V1: 1,
  V2: 2,
  V3: 3,
} as const;
```

- 工作流参数包含版本号
- 代码根据版本号分支处理
- 保证运行中的旧版本工作流不被破坏

**Temporal Patching**:
```typescript
if (wf.patched("workflow-history-metrics")) {
  // 新逻辑
}
```
- 用于安全地修改工作流代码
- 区分"使用旧历史运行"和"新启动"的工作流

## 7. 关键设计模式

### 7.1 Signal + Condition 模式

```typescript
// 设置信号处理器
wf.setHandler(segmentUpdateSignal, (update) => {
  segmentAssignments.set(update.segmentId, {
    currentlyInSegment: update.currentlyInSegment,
    segmentVersion: update.segmentVersion,
  });
});

// 等待条件
await wf.condition(() => segmentAssignedTrue(cn.segment));
```

**优势**:
- 解耦信号发送和等待逻辑
- 信号可能在等待之前到达（先存储，后检查）
- 支持超时等待

### 7.2 节点处理记录

每个节点处理完成后:
```typescript
await onNodeProcessedV2({
  workspaceId,
  userId,
  node: currentNode,
  journeyStartedAt,
  journeyId,
  // ...
});
```

记录到:
1. **PostgreSQL**: `userJourneyEvent` 表（用于恢复检查）
2. **ClickHouse**: `DFJourneyNodeProcessed` 事件（用于统计分析）

### 7.3 可运行性检查

工作流启动前和关键节点后检查:
```typescript
if (!(await isRunnable({ journeyId, userId, workspaceId }))) {
  logger.info("early exit unrunnable user journey");
  return null;
}
```

检查内容:
- 工作空间是否 Active
- 用户是否已完成过此 Journey（且不允许多次运行）

## 8. 错误处理与重试

### 8.1 Activity 重试

不同活动有不同的重试策略:

```typescript
// 消息发送
const { sendMessageV2 } = proxyActivities({
  startToCloseTimeout: "2 minutes",
  retry: { maximumAttempts: currentNode.retryCount ?? 3 },
});

// 分段计算（长时间运行）
const { waitForComputeProperties } = proxyActivities({
  startToCloseTimeout: "20 minutes",
  heartbeatTimeout: "30 seconds",
  retry: { maximumAttempts: waitForComputePropertiesMaxAttempts },
});
```

### 8.2 节点失败处理

MessageNode 的 `skipOnFailure` 选项:
```typescript
if (!messageSucceeded && !currentNode.skipOnFailure) {
  logger.info("message node early exit");
  nextNode = definition.exitNode;  // 跳到出口
  break;
}
```

### 8.3 最终重试后降级

```typescript
// packages/backend-lib/src/journeys/userWorkflow/activities.ts:271-296
const isLastAttempt = activityInfo.attempt >= (retryCount ?? 3);
if (isLastAttempt) {
  // 不抛出异常，而是记录为 JourneyEarlyExit
  return err({
    type: InternalEventType.JourneyEarlyExit,
    message: `Message failed after maximum retry attempts`,
  });
}
```

## 9. 数据流总结

### 9.1 配置数据流

```
[UI 编辑]
  → [ReactFlow 节点状态]
  → [UpsertJourneyResource API]
  → [PostgreSQL journey.definition (JSON)]
  → [工作流启动时快照到 args.definition]
  → [Temporal 工作流历史记录]
```

### 9.2 执行数据流

```
[触发事件/分段变更]
  → [triggerEventEntryJourneys / triggerSegmentEntryJourney]
  → [signalWithStart 启动工作流]
  → [userJourneyWorkflow 节点循环]
    → [Activity 调用: getSegmentAssignment, sendMessageV2, etc.]
    → [PostgreSQL: userJourneyEvent]
    → [ClickHouse: internal_events (DFJourneyNodeProcessed)]
  → [工作流完成或继续等待]
```

## 10. 关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `packages/dashboard/src/components/journeys/journeysBuilder.tsx` | 可视化画布编辑器 |
| `packages/api/src/controllers/journeysController.ts` | REST API 控制器 |
| `packages/backend-lib/src/journeys.ts` | Journey 核心业务逻辑 |
| `packages/backend-lib/src/journeys/userWorkflow.ts` | Temporal 工作流定义 |
| `packages/backend-lib/src/journeys/userWorkflow/activities.ts` | 工作流活动实现 |
| `packages/backend-lib/src/journeys/userWorkflow/lifecycle.ts` | 工作流启动管理 |
| `packages/isomorphic-lib/src/journeys.ts` | 共享工具函数（节点关系、约束检查） |
| `packages/isomorphic-lib/src/types.ts` | JourneyDefinition 等类型定义 |

## 11. 扩展建议

1. **配置版本管理**: 考虑显式版本化 Journey 配置，支持灰度发布和回滚
2. **工作流历史清理**: 对于极长时间运行的工作流，考虑定期 `continueAsNew` 避免历史过大
3. **监控增强**: 已有的 `reportWorkflowInfo` 活动可扩展更多指标
4. **死信队列**: 对多次失败的消息考虑死信队列机制
5. **可视化调试**: 考虑在 UI 中展示单个用户的 Journey 执行路径
