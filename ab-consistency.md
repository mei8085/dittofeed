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

这种设计确保了在分布式、多 Worker 环境下，A/B 实验的分配逻辑正确、一致，为实验结果的可信度提供了坚实的基础。
