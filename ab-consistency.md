# A/B 实验多 Worker 环境一致性保证

## 1. 概述

在 DittoFeed 系统中，A/B 实验通过随机分桶（Random Bucket）机制实现。系统设计确保在多 Worker 环境下，同一用户的实验分配保持一致，并且在实验运行过程中调整流量时能够正确处理历史用户。

## 2. 分桶函数与用户身份绑定

### 2.1 分桶函数实现

DittoFeed 使用 **确定性哈希** 算法来实现用户分桶。关键代码位于 `packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` 第 1479-1539 行：

```sql
reinterpretAsUInt64(reverse(unhex(left(hex(MD5(concat(user_id, ${segmentNameParam}))), 16)))) < (${qb.addQueryValue(node.percent, "Float64")} * pow(2, 64))
```

这个分桶函数的核心逻辑：
1. **输入组合**：将 `user_id` 与 `segment_name` 拼接（使用 segment name 而非 ID 以确保测试可重复性）
2. **哈希计算**：对拼接结果执行 MD5 哈希
3. **数值提取**：取哈希结果的前 16 个字符（64 位），转换为无符号 64 位整数
4. **阈值比较**：将计算出的整数与 `percent * 2^64` 进行比较

### 2.2 确定性保证

分桶函数的确定性体现在：
- **输入不变**：同一用户（`user_id`）和同一实验（`segment_name`）的组合始终产生相同的输入
- **算法确定**：MD5 哈希算法是确定性的，相同输入产生相同输出
- **比较确定**：阈值计算基于 `percent * 2^64`，相同百分比产生相同阈值

这意味着：**对于同一个用户和同一个实验，无论在哪个 Worker 上计算，分桶结果始终一致**。

### 2.3 用户身份绑定

用户身份通过以下机制与实验分配绑定：

1. **用户身份标识**：使用 `user_id` 作为唯一标识，这个 ID 来自用户属性定义中的 ID 类型（`UserPropertyDefinitionType.Id`）
2. **状态 ID 机制**：每个段节点有唯一的 `state_id`，通过 `segmentNodeStateId()` 函数计算（`packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` 第 394-411 行）
   - 使用 UUID v5 算法
   - 输入：`segment.definitionUpdatedAt` + `nodeId` + 版本信息
   - 命名空间：`segment.id`
3. **存储关联**：在 ClickHouse 中，用户分配状态存储在 `resolved_segment_state` 表中，键为 `(workspace_id, segment_id, state_id, user_id)`

## 3. 多 Worker 环境下的一致性保证

### 3.1 数据库层面的一致性

#### 3.1.1 顺序一致性配置

系统提供 `assignmentSequentialConsistency` 配置项（`packages/backend-lib/src/config.ts` 第 831-832 行）：

```typescript
export function assignmentSequentialConsistency(): "1" | "0" {
  return config().assignmentSequentialConsistency ? "1" : "0";
}
```

默认启用（`assignmentSequentialConsistency !== "false"`），在查询 ClickHouse 时设置 `select_sequential_consistency` 参数。

#### 3.1.2 一致性查询机制

在所有查询用户分配的关键函数中，都启用了顺序一致性：

- `findAllSegmentAssignmentsByIds()` - `packages/backend-lib/src/segments.ts` 第 104 行
- `findAllSegmentAssignmentsByIdsForUsers()` - `packages/backend-lib/src/segments.ts` 第 150 行
- `findAllSegmentAssignments()` - `packages/backend-lib/src/segments.ts` 第 221 行
- `findRecentlyUpdatedUsersInSegment()` - `packages/backend-lib/src/segments.ts` 第 1016 行

#### 3.1.3 查询时的一致性策略

读取用户分配时使用 `argMax` 聚合函数获取最新状态：

```sql
argMax(segment_value, assigned_at) as latest_segment_value
```

这确保即使有多个 Worker 写入，读取时也能获取到最新的一致状态。

#### 3.1.4 多 Worker 并发写入的可见性窗口

当多个 Worker 并发处理同一用户时，系统通过以下机制确保数据一致性：

**1. 写入等待机制 (`wait_end_of_query`)**

在 `insertSegmentAssignments()` 函数中（`packages/backend-lib/src/segments.ts` 第 1023-1040 行），写入操作使用 `wait_end_of_query: 1` 设置：

```typescript
await client.insert({
  table: "computed_property_assignments_v2",
  values: assignments,
  format: "JSONEachRow",
  clickhouse_settings: { wait_end_of_query: 1 },
});
```

这确保：
- 写入操作同步等待数据完全插入
- 函数返回时，数据已对后续查询可见
- 避免了"写入后立即读取但数据未可见"的竞态条件

**2. 去重前后的可见性窗口**

ClickHouse MergeTree 引擎的去重机制分为两个阶段：

**阶段一：写入后立即可见（查询时去重）**
- 数据写入后立即对查询可见
- 即使有重复行，`argMax` 聚合函数会在查询时自动选择最新版本
- 这是"最终一致性"的关键保障

**阶段二：后台合并去重**
- ClickHouse 后台定期执行合并操作
- 合并时会删除重复行，只保留最新版本
- 这个过程是异步的，不影响查询结果的正确性

**可见性时间线示例：**

```
时间轴 →
T0: Worker A 和 Worker B 同时开始处理用户 X
T1: Worker A 完成计算，写入 (segment_value=true, assigned_at=T1)
    → 数据立即可见，查询返回 true
T2: Worker B 完成计算，写入 (segment_value=true, assigned_at=T2)
    → 数据立即可见，存在两行记录
T3: 查询执行 argMax(segment_value, assigned_at)
    → 返回 T2 对应的 true（最新版本）
T4: ClickHouse 后台合并完成
    → 只保留 T2 的记录，查询仍然返回 true
```

**关键保证：**
- 即使存在重复行，`argMax` 始终返回正确的最新状态
- 去重前后查询结果一致
- 唯一的区别是存储效率，而非结果正确性

#### 3.1.5 去重正确性依据：职责边界分析

**重要澄清：** 查询时按 `assigned_at` 选最新值与后台合并优化的职责边界是完全分离的：

**1. 查询时的正确性保证（必须保证）**

`argMax(segment_value, assigned_at)` 的职责：
- **正确性层面**：从所有可能存在的重复行中选择"逻辑上最新"的版本
- **数据层依赖**：依赖 `assigned_at` 时间戳作为版本指示器
- **查询语义**：返回具有最大 `assigned_at` 的行对应的 `segment_value`

**2. 后台合并优化（性能优化，不影响正确性）**

ClickHouse ReplacingMergeTree 合并的职责：
- **性能层面**：删除物理重复行，减少存储占用和查询时的扫描量
- **不影响正确性**：合并前后查询结果必须一致
- **合并时机**：异步执行，由 ClickHouse 后台调度

**职责边界图示：**

```
┌─────────────────────────────────────────────────────────────┐
│                    正确性保证层（必须正确）                   │
│                                                             │
│   argMax(segment_value, assigned_at)                        │
│   依赖 assigned_at 时间戳选择逻辑上最新的版本                 │
│                                                             │
│   无论是否合并，查询结果必须一致                              │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │ 数据来源
┌─────────────────────────────────────────────────────────────┐
│                    存储优化层（性能优化）                     │
│                                                             │
│   ClickHouse ReplacingMergeTree 后台合并                    │
│   删除物理重复行，减少存储占用                                │
│                                                             │
│   合并时机不确定，但不影响查询语义                            │
└─────────────────────────────────────────────────────────────┘
```

#### 3.1.6 同毫秒写入的风险分析

**场景描述：** 两个 Worker（或同一 Worker 的两次计算）对同一用户在同一毫秒内完成写入。

**风险 1：时间戳精度问题**

```typescript
// computePropertiesQueueWorkflow.ts:275
const now = Date.now();  // 毫秒级精度

// computePropertiesIncremental.ts 中写入时使用
toDateTime64(${nowSeconds}, 3) as assigned_at  // DateTime64(3) = 毫秒精度
```

如果两个计算在**同一毫秒**内完成：
- Worker A：`now = 1778476948123` → `assigned_at = '2026-05-11 10:02:28.123'`
- Worker B：`now = 1778476948123` → `assigned_at = '2026-05-11 10:02:28.123'`
- 两行数据的 `assigned_at` 完全相同

**风险 2：argMax 在时间戳相同时的行为不确定性**

当 `argMax(value, time)` 遇到多个相同 `time` 值时：
- ClickHouse 文档明确说明：**如果存在多个最大值，返回遇到的第一个值**
- 但"第一个"的定义取决于：
  - 数据存储顺序（写入顺序）
  - 数据分区（不同 part 的读取顺序）
  - 合并状态（合并后可能改变顺序）
- **结果可能非确定性**

**风险 3：ReplacingMergeTree 合并的不确定性**

对于 `resolved_segment_state` 表：
```sql
ENGINE = ReplacingMergeTree()
ORDER BY (workspace_id, segment_id, state_id, user_id)
```

没有指定版本列时：
- ClickHouse 合并时**保留遇到的最后一行**
- "最后一行"的定义同样取决于存储顺序
- **合并结果可能非确定性**

#### 3.1.7 现有实现的风险规避策略

**策略 1：工作流层面的并发控制（主要规避机制）**

`computePropertiesQueueWorkflow.ts` 中实现了工作区级别的并发控制：

```typescript
// 1. membership 集合防止同一工作区同时入队
const membership = new Set<string>();

// 2. 信号量控制并发，但同一工作区不会重复入队
const semaphore = new Semaphore(concurrency);

// 3. 如果工作区已在处理中（inFlight 或 membership），不会重复添加
if (!membership.has(newKey)) {
  priorityQueue.push(newItem);
  membership.add(newKey);
}
```

**效果：**
- 同一工作区**不会被多个 Worker 同时处理**
- 从根本上避免了"两个 Worker 同时处理同一用户"的场景

**策略 2：确定性分桶算法（次要规避机制）**

即使出现极端情况下的重复计算（例如工作流重试）：
- 分桶算法是**确定性**的：`MD5(user_id + segment_name)`
- 两次计算的 `segment_value` 完全相同
- `argMax` 选择哪一行都一样，因为值相同

**策略 3：幂等写入设计（容错机制）**

- 即使写入重复行，由于值相同，查询结果无差异
- ClickHouse 合并时无论保留哪一行，值都一样
- 从业务角度看，结果完全等价

#### 3.1.8 风险承受范围分析

**可能出现非确定性的边界场景：**

| 场景 | 发生概率 | 业务影响 |
|------|---------|---------|
| 同一工作区被两个 Worker 同时处理 | 极低（工作流并发控制） | 由于分桶确定性，无影响 |
| 工作流重试导致同一用户重复计算 | 低（故障场景） | 由于分桶确定性，无影响 |
| 实验定义变更过程中出现重复计算 | 低（变更期间） | 需要仔细处理 definitionUpdatedAt |
| 分布式系统时钟漂移导致时间错乱 | 极低（需要 NTP 配置错误） | 可能导致"旧值覆盖新值" |

**承受能力评估：**

系统可以**安全承受**绝大多数边界场景：
1. **可接受的风险**：由于分桶算法的确定性，即使出现重复写入，值也是相同的
2. **需要注意的场景**：实验定义变更期间，需要确保 `definitionUpdatedAt` 正确更新
3. **系统层面的防护**：工作流的并发控制机制已经避免了主要的并发风险

### 3.2 计算层面的一致性

#### 3.2.1 确定性计算

分桶计算完全是确定性的，不依赖任何 Worker 本地状态：
- 不使用随机数生成器
- 所有输入都来自数据库或用户标识
- 哈希算法是标准的、可重现的

#### 3.2.2 幂等写入

即使多个 Worker 对同一用户进行相同的分桶计算，写入操作是幂等的：
- 计算结果相同（确定性）
- 写入到同一个 `resolved_segment_state` 记录
- ClickHouse 合并树引擎会自动处理重复写入

#### 3.2.3 增量计算机制

系统采用增量计算而非全量重算：
- 使用 `updated_computed_property_state` 表跟踪需要更新的用户
- 只处理有变化的用户
- 通过 `periodBound` 参数限制计算范围

## 4. 实验运行中调整流量

### 4.1 流量调整的实现

调整实验流量通过修改 Random Bucket 节点的 `percent` 参数实现。关键逻辑在分桶函数中：

```sql
... < (${qb.addQueryValue(node.percent, "Float64")} * pow(2, 64))
```

当 `percent` 变化时：
- 阈值 `percent * 2^64` 发生变化
- 部分用户的分桶结果会改变
- 这是设计行为：增加流量会将更多用户纳入实验组，减少流量则相反

### 4.2 流量调整的一致性保证

#### 4.2.1 历史用户的处理

当流量调整时，已经进入实验的用户如何处理？

1. **状态 ID 不变**：`segmentNodeStateId()` 只依赖 `definitionUpdatedAt` 和 `nodeId`
   - 如果只是修改 `percent` 但不更新 `definitionUpdatedAt`，`state_id` 保持不变
   - 这意味着用户的分配状态会被重新计算

2. **定义更新触发**：当段定义更新时（`definitionUpdatedAt` 变化），`state_id` 会变化
   - 这会触发全量重新计算
   - 旧的 `state_id` 对应的历史数据被保留但不再使用

#### 4.2.2 增量更新机制

系统通过 `shouldResetComputedProperty()` 函数判断是否需要重置计算（`packages/backend-lib/src/computedProperties/computePropertiesIncremental.ts` 第 129-148 行）：

```typescript
function shouldResetComputedProperty({
  definitionUpdatedAt,
  createdAt,
  now,
  periodBound,
}: {
  definitionUpdatedAt: number;
  createdAt: number;
  now: number;
  periodBound?: number;
}): boolean {
  if (!definitionUpdatedAt) {
    return false;
  }
  return (
    definitionUpdatedAt <= now &&
    definitionUpdatedAt >= (periodBound ?? 0) &&
    definitionUpdatedAt > createdAt
  );
}
```

当 `definitionUpdatedAt` 更新时，系统会重置计算状态。

### 4.3 流量调整的最佳实践

1. **小幅度调整**：避免流量大幅波动，逐步调整
2. **监控指标**：调整后密切关注实验指标变化
3. **记录变更**：记录每次流量调整的时间和原因
4. **考虑用户体验**：用户从实验组变对照组可能影响体验，建议只增加流量不减少

## 5. 历史用户归档

### 5.1 数据存储结构

系统使用 ClickHouse 存储用户分配历史，关键表：

1. **computed_property_state_v3** - 计算中间状态
   - 存储每个用户在每个状态节点的计算状态
   - 包含 `last_value`、`unique_count` 等聚合字段

2. **resolved_segment_state** - 解析后的段状态
   - 存储用户是否在某个段中
   - 包含 `segment_state_value` 和 `max_event_time`
   - 键：`(workspace_id, segment_id, state_id, user_id)`

3. **computed_property_assignments_v2** - 最终分配结果
   - 存储用户最终的段分配
   - 包含 `segment_value` 和 `assigned_at`
   - 使用 `argMax(segment_value, assigned_at)` 获取最新状态

### 5.2 历史数据的保留

系统默认保留所有历史分配记录：

1. **时间戳记录**：每次分配都记录 `assigned_at` 时间戳
2. **版本追踪**：通过 `state_id` 区分不同版本的计算逻辑
3. **可追溯性**：可以通过历史数据追溯用户在任意时间点的分配状态

### 5.3 删除实验时的数据清理

删除段时，系统会清理相关数据（`packages/backend-lib/src/segments.ts` 第 1154-1174 行）：

```typescript
const queries = [
  `DELETE FROM computed_property_state_v3 WHERE ...`,
  `DELETE FROM computed_property_assignments_v2 WHERE ...`,
  `DELETE FROM processed_computed_properties_v2 WHERE ...`,
  `DELETE FROM computed_property_state_index WHERE ...`,
  `DELETE FROM updated_computed_property_state WHERE ...`,
  `DELETE FROM updated_property_assignments_v2 WHERE ...`,
  `DELETE FROM resolved_segment_state WHERE ...`,
];
```

使用 `mutations_sync = 0` 和 `lightweight_deletes_sync = 0` 进行异步删除，不阻塞操作。

### 5.4 历史归档策略建议

对于需要长期归档的实验数据：

1. **定期导出**：可以使用 `exportSegmentAssignmentsToCsv()` 功能导出 CSV
2. **冷存储**：系统支持冷存储配置（`enableColdStorage`），可以将历史数据移到更便宜的存储
3. **数据保留策略**：根据业务需求设置合理的数据保留期限
4. **审计日志**：保留实验配置变更历史，用于事后分析

## 6. 架构图示

### 6.1 分桶流程

```
用户事件 → 用户身份识别(user_id)
                ↓
    分桶计算：MD5(user_id + segment_name)
                ↓
          64位整数 < percent * 2^64 ?
                ↓
         是 → 实验组 | 否 → 对照组
                ↓
        写入 ClickHouse (resolved_segment_state)
```

### 6.2 多 Worker 一致性

```
Worker A                     Worker B
   ↓                            ↓
读取用户ID               读取用户ID
   ↓                            ↓
分桶计算(确定性)       分桶计算(确定性)
   ↓                            ↓
  结果相同                  结果相同
   ↓                            ↓
写入 ClickHouse  ←→  写入 ClickHouse
   ↓                            ↓
   幂等写入，数据一致
```

### 6.3 状态流转

```
初始状态 → 计算状态(computed_property_state_v3)
                ↓
          解析状态(resolved_segment_state)
                ↓
          最终分配(computed_property_assignments_v2)
                ↓
         查询时使用 argMax 获取最新值
```

## 7. 关键配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `assignmentSequentialConsistency` | `true` | 是否启用 ClickHouse 顺序一致性 |
| `computePropertiesInterval` | - | 计算属性的时间间隔 |
| `computePropertiesQueueConcurrency` | 30 | 计算队列并发数 |
| `computePropertiesSchedulerInterval` | 10s | 计算调度器间隔 |
| `enableColdStorage` | `false` | 是否启用冷存储 |

## 8. 总结

DittoFeed 的 A/B 实验系统通过以下机制保证多 Worker 环境下的一致性：

1. **确定性分桶**：使用 MD5 哈希算法，确保同一用户在同一实验中始终分到同一组
2. **顺序一致性**：通过 ClickHouse 的 `select_sequential_consistency` 保证读取的一致性
3. **幂等写入**：即使多个 Worker 同时计算，写入结果相同且幂等
4. **增量计算**：只处理有变化的用户，提高效率的同时保持一致性
5. **版本追踪**：通过 `state_id` 和 `definitionUpdatedAt` 管理定义变更
6. **历史可追溯**：保留所有历史分配记录，支持事后分析

### 8.1 补充总结：关键机制详解

**多 Worker 并发写入的可见性保证：**
- `wait_end_of_query=1` 确保写入完成后才返回，避免"写后读"不一致
- `argMax` 聚合函数在查询时自动选择最新版本，即使存在重复行
- 去重前后查询结果一致，只是存储效率不同

**流量调整的生效条件：**
- **必须更新 `definitionUpdatedAt`** 才能触发重算
- 只修改 `percent` 不更新 `definitionUpdatedAt` 会导致裁剪，用户沿用旧状态
- `canPrune()` 函数是核心判断逻辑，检查 `definitionUpdatedAt` 是否在 `[periodBound, now]` 区间内

**Period 机制的作用：**
- 跟踪每个版本的计算进度
- 版本号 = `definitionUpdatedAt.toString()`
- 定义变更会创建新的 Period，旧 Period 数据保留但不再使用

这种设计确保了在分布式、多 Worker 环境下，A/B 实验的分配逻辑正确、一致，为实验结果的可信度提供了坚实的基础。
