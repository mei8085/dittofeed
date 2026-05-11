# Dittofeed 实时分析面板数据流分析报告

## 一、整体架构概览

Dittofeed 的实时分析面板采用经典的 **前端-后端-数据仓库** 三层架构：

```
前端 Dashboard (React + TanStack Query)
        │
        ▼
API 层 (Fastify + analysisController)
        │
        ▼
业务逻辑层 (backend-lib/analysis.ts)
        │
        ▼
ClickHouse 数据仓库
```

核心数据流向：
1. 前端组件状态变化 → 触发查询参数更新
2. React Query 检测查询键变化 → 发起 API 请求
3. API 控制器接收请求 → 调用后端查询逻辑
4. 后端构建 ClickHouse SQL 查询 → 执行查询
5. 结果返回 → 前端图表渲染

---

## 二、查询参数的组装

### 2.1 参数类型定义

查询参数由 `GetChartDataRequest` 定义，位于 `packages/isomorphic-lib/src/types.ts:6477`：

```typescript
{
  workspaceId: string;           // 必填：工作空间 ID
  startDate: string;             // 必填：开始时间 (ISO 格式)
  endDate: string;               // 必填：结束时间 (ISO 格式)
  granularity?: string;          // 可选：时间粒度 (auto, 1minute, 1hour 等)
  groupBy?: string;              // 可选：分组维度 (journey, broadcast, messageState 等)
  filters?: {                    // 可选：过滤器对象
    journeyIds?: string[];       // 按旅程 ID 过滤
    broadcastIds?: string[];     // 按广播 ID 过滤
    channels?: string[];         // 按渠道过滤
    providers?: string[];        // 按供应商过滤
    templateIds?: string[];      // 按模板 ID 过滤
    userIds?: string[];          // 按用户 ID 过滤
    messageStates?: string[];    // 按消息状态过滤
  };
}
```

### 2.2 参数组装流程

参数组装在 `packages/dashboard/src/components/analysisChart.tsx` 中完成，分为以下步骤：

#### 第一步：时间范围选择

组件维护了一个状态对象，包含时间选项和日期范围：

```typescript
// 第 150-161 行：状态定义
interface State {
  selectedTimeOption: string;     // 选中的时间选项 ID
  referenceDate: Date;            // 参考日期（用于相对时间计算）
  dateRange: {
    startDate: string;
    endDate: string;
  };
  groupBy: GroupByOption;         // 分组维度
  // ... 其他状态
}
```

默认时间范围为「最近 7 天」（第 66-71 行）。用户可通过 `DateRangeSelector` 组件选择预设时间或自定义范围。

#### 第二步：过滤器构建

过滤器来自两个来源的合并：
- **硬编码过滤器**（hardcodedFilters）：来自配置，优先级更高
- **动态过滤器**：用户通过 UI 选择的过滤条件

核心逻辑在第 317-363 行：

```typescript
const filters = useMemo(() => {
  // 从 filterState 提取各类过滤值
  const dynamicJourneyIds = getFilterValues(filtersState, "journeyIds");
  const dynamicBroadcastIds = getFilterValues(filtersState, "broadcastIds");
  const dynamicChannels = getFilterValues(filtersState, "channels");
  // ... 其他过滤器
  
  // 合并：硬编码覆盖动态
  const journeyIds = hardcodedFilters?.journeyIds ?? dynamicJourneyIds;
  const broadcastIds = hardcodedFilters?.broadcastIds ?? dynamicBroadcastIds;
  
  // 消息状态应用级联逻辑（如选择 opened 会包含 delivered）
  const expandedMessageStates = messageStates
    ? expandCascadingMessageFilters(messageStates)
    : undefined;
  
  // 至少有一个过滤器才返回对象
  if (!journeyIds && !broadcastIds && ...) {
    return undefined;
  }
  
  return {
    ...(journeyIds && { journeyIds }),
    ...(broadcastIds && { broadcastIds }),
    // ...
  };
}, [filtersState, hardcodedFilters]);
```

#### 第三步：查询参数最终组装

在第 365-376 行，调用 `useAnalysisChartQuery` 时组装最终参数：

```typescript
const chartQuery = useAnalysisChartQuery(
  {
    startDate: state.dateRange.startDate,
    endDate: state.dateRange.endDate,
    granularity: "auto",           // 自动选择时间粒度
    ...(state.groupBy && { groupBy: state.groupBy }),
    ...(filters && { filters }),
  },
  {
    placeholderData: keepPreviousData,  // 保持旧数据直到新数据返回
  },
);
```

#### 第四步：workspaceId 注入

在 `useAnalysisChartQuery` Hook（`packages/dashboard/src/lib/useAnalysisChartQuery.ts`）中，从全局状态获取 workspaceId：

```typescript
const { workspace } = useAppStorePick(["workspace"]);
const workspaceId = workspace.value.id;

// 构建查询键时包含 workspaceId
const queryKey = [ANALYSIS_CHART_QUERY_KEY, { ...params, workspaceId }];

// 发起请求时添加 workspaceId
const response = await axios.get(`${baseApiUrl}/analysis/chart-data`, {
  params: {
    ...params,
    workspaceId,
  },
  headers: authHeaders,
});
```

---

## 三、缓存策略与刷新节奏

### 3.1 缓存架构

Dittofeed 使用 **TanStack Query (React Query)** 作为前端缓存层。

#### 图表数据查询（Chart Data）

- **查询键**：`["analysisChart", { ...params, workspaceId }]`
- **缓存键**：所有影响结果的参数都包含在查询键中
- **缓存策略**：
  - 不设置 `staleTime`：数据立即被视为陈旧
  - 使用 `placeholderData: keepPreviousData`：切换参数时保持旧数据显示，避免闪烁

#### 摘要数据查询（Summary Data）

- **查询键**：`["analysisSummary", { ...params, workspaceId }]`
- 缓存策略与图表数据相同

#### 资源名称查询（Resources）

用于将 ID 转换为显示名称（第 378-388 行）：

```typescript
const resourcesQuery = useResourcesQuery(
  {
    journeys: true,
    broadcasts: true,
    messageTemplates: true,
  },
  {
    staleTime: 5 * 60 * 1000,  // 5 分钟内视为新鲜
  },
);
```

### 3.2 刷新机制

#### 手动刷新

第 390-404 行定义了 `onRefresh` 回调：

```typescript
const onRefresh = useCallback(() => {
  setState((draft) => {
    const option = timeOptions.find((o) => o.id === draft.selectedTimeOption);
    if (option === undefined || option.type !== "minutes") {
      return;  // 自定义时间范围不支持刷新
    }
    const endDate = new Date();
    const startDate = subMinutes(endDate, option.minutes);
    draft.dateRange = {
      startDate: startDate.toISOString(),
      endDate: endDate.toISOString(),
    };
    draft.referenceDate = endDate;
  });
}, [setState]);
```

刷新按钮的 UI 在第 731-742 行：

```typescript
<Tooltip title="Refresh Results" placement="bottom-start">
  <IconButton
    disabled={state.selectedTimeOption === "custom"}  // 自定义范围禁用
    onClick={onRefresh}
    sx={{ border: "1px solid", borderColor: "grey.400" }}
  >
    <RefreshIcon />
  </IconButton>
</Tooltip>
```

#### 自动刷新

**当前实现不包含自动轮询刷新**。搜索代码库发现：
- `usersTableV2.tsx` 中使用了 `refetchInterval`（根据 `autoReload` 开关）
- `recomputedRecently.tsx` 中使用了 `refetchInterval: 5 * 1000`（5 秒）
- 但分析面板没有使用 `refetchInterval`

这意味着分析面板的数据是「按需刷新」的，依赖于：
1. 用户点击刷新按钮
2. 切换过滤条件或时间范围
3. 组件重新挂载

### 3.3 缓存失效

通过 `useQueryClient.invalidateQueries` 机制。在分析面板场景中：
- 没有专门的缓存失效逻辑
- 参数变化会自动触发新的查询（查询键变化）
- 资源查询的 staleTime 控制何时重新获取名称映射

---

## 四、图表数据的更新同步机制

### 4.1 响应式更新原理

基于 React Query 的核心机制：**查询键驱动的响应式更新**。

#### 查询键构建

```typescript
// 图表数据查询键
const queryKey = [
  "analysisChart",
  { ...params, workspaceId }  // 包含所有参数
];
```

当任何参数变化时：
1. React Query 检测到查询键变化
2. 发起新的 API 请求
3. 请求完成后更新 `chartQuery.data`
4. 组件使用 `useMemo` 重新计算图表数据

#### 图表数据转换

第 496-592 行的 `chartData` useMemo：

```typescript
const chartData = useMemo(() => {
  if (!chartQuery.data?.data) return [];
  
  // 将扁平数据转换为 Recharts 需要的格式
  const grouped = new Map<string, Record<string, string | number>>();
  const groups = new Set<string>();
  
  chartQuery.data.data.forEach((point) => {
    const timestamp = new Date(point.timestamp).toISOString();
    const rawGroupLabel = point.groupLabel ?? "Total";
    const groupLabel = mapIdToName(rawGroupLabel, state.groupBy);
    
    groups.add(groupLabel);
    
    if (!grouped.has(timestamp)) {
      grouped.set(timestamp, { timestamp });
    }
    const entry = grouped.get(timestamp);
    if (entry) {
      entry[groupLabel] = point.count;
    }
  });
  
  // 可选：百分比模式转换
  if (state.displayMode === "percentage") {
    // 根据不同 groupBy 类型应用不同的百分比计算逻辑
    // ...
  }
  
  return sortedData;
}, [chartQuery.data, state.displayMode, state.groupBy, mapIdToName]);
```

### 4.2 后端 ClickHouse 查询构建

#### 查询构建器

使用 `ClickHouseQueryBuilder` 类（`packages/backend-lib/src/clickhouse.ts:24`）安全构建查询：

```typescript
class ClickHouseQueryBuilder {
  // 使用参数化查询防止 SQL 注入
  addQueryValue(value: unknown, dataType: string): string {
    const id = `v${this.queries.size}`;
    this.queries.set(id, value);
    return `{${id}:${dataType}}`;  // 返回占位符
  }
}
```

#### SQL 查询结构

`getChartData` 函数（`packages/backend-lib/src/analysis.ts:122`）构建复杂的 CTE 查询：

```sql
WITH sent_messages AS (
  -- 第一步：筛选 SENT 事件作为基础队列
  SELECT
    ie.message_id AS origin_message_id,
    toStartOfHour(ie.processing_time) AS timestamp,
    journey_id as groupKey  -- 根据 groupBy 选择分组字段
  FROM internal_events AS ie
  WHERE
    ie.workspace_id = {v0:String}
    AND ie.processing_time >= parseDateTimeBestEffort({v1:String}, 'UTC')
    AND ie.processing_time <= parseDateTimeBestEffort({v2:String}, 'UTC')
    AND ie.event = 'MESSAGE_SENT'
    AND ie.hidden = false
    -- 应用过滤器（仅对 SENT 事件过滤）
    AND journey_id IN {v3:Array(String)}
    -- ...
),
status_events AS (
  -- 第二步：获取相关的状态事件
  SELECT
    ie.origin_message_id,
    ie.event
  FROM internal_events AS ie
  WHERE
    ie.workspace_id = {v0:String}
    AND ie.event IN ('EMAIL_DELIVERED', 'EMAIL_OPENED', ...)
    AND ie.origin_message_id IN (SELECT origin_message_id FROM sent_messages)
),
message_flags AS (
  -- 第三步：聚合每个消息的最终状态
  SELECT
    origin_message_id,
    max(event IN ('EMAIL_DELIVERED', 'SMS_DELIVERED')) AS has_delivered,
    max(event = 'EMAIL_OPENED') AS has_opened,
    max(event = 'EMAIL_CLICKED') AS has_clicked,
    max(event IN ('EMAIL_BOUNCED', 'SMS_FAILED')) AS has_bounced
  FROM status_events
  GROUP BY origin_message_id
),
message_states_per_message AS (
  -- 第四步：合并 SENT 和状态信息
  SELECT
    sm.origin_message_id,
    sm.groupKey,
    sm.timestamp,
    1 AS has_sent,
    coalesce(mf.has_delivered, 0) AS has_delivered,
    -- ...
  FROM sent_messages sm
  LEFT JOIN message_flags mf USING (origin_message_id)
)
-- 第五步：最终聚合
SELECT
  timestamp,
  groupKey,
  count(*) as count
FROM message_states_per_message
GROUP BY timestamp, groupKey
ORDER BY timestamp ASC
```

#### 时间粒度自动选择

`selectAutoGranularity` 函数（第 20-56 行）根据时间范围自动选择合适的粒度：

| 时间范围 | 自动选择的粒度 |
|---------|--------------|
| ≤ 2 小时 | 1minute |
| ≤ 8 小时 | 5minutes |
| ≤ 24 小时 | 30minutes |
| ≤ 7 天 | 1hour |
| ≤ 30 天 | 6hours |
| ≤ 90 天 | 1day |
| ≤ 365 天 | 7days |
| > 365 天 | 30days |

目标是保持 50-200 个数据点，平衡图表可读性和查询性能。

### 4.3 事件级联逻辑

分析查询实现了邮件事件的级联统计（第 116-121 行注释）：

```
Click → Open → Delivery
  └── 点击事件被视为：点击 + 打开 + 送达
Open → Delivery
  └── 打开事件被视为：打开 + 送达
Delivery
  └── 送达事件仅计入：送达
```

每条消息在每个状态类型中只计数一次（去重）。

---

## 五、关键组件和文件清单

| 路径 | 职责 |
|-----|------|
| `packages/dashboard/src/pages/analysis/overview.page.tsx` | 分析面板页面入口 |
| `packages/dashboard/src/components/analysisChart.tsx` | 主图表组件（状态管理、数据转换、UI） |
| `packages/dashboard/src/components/analysisChart/analysisSummaryPanel.tsx` | 摘要指标面板 |
| `packages/dashboard/src/lib/useAnalysisChartQuery.ts` | 图表数据查询 Hook |
| `packages/dashboard/src/lib/useAnalysisSummaryQuery.ts` | 摘要数据查询 Hook |
| `packages/api/src/controllers/analysisController.ts` | API 控制器层 |
| `packages/backend-lib/src/analysis.ts` | 核心查询逻辑（ClickHouse SQL 构建） |
| `packages/backend-lib/src/clickhouse.ts` | ClickHouse 客户端和查询构建器 |
| `packages/isomorphic-lib/src/types.ts` | 共享类型定义 |

---

## 六、总结

### 设计亮点

1. **参数化查询**：使用 ClickHouseQueryBuilder 防止 SQL 注入
2. **级联统计**：邮件事件的级联逻辑符合营销分析的最佳实践
3. **自动粒度**：根据时间范围智能选择数据粒度
4. **用户体验优化**：`keepPreviousData` 避免切换时的空白闪烁
5. **灵活过滤**：支持硬编码和动态过滤器的合并

### 潜在改进点

1. **缺少自动刷新**：分析面板没有 `refetchInterval`，用户必须手动刷新
2. **缓存策略保守**：图表数据没有 `staleTime`，每次参数变化都重新查询
3. **查询性能**：对于大数据量，CTE 方式可能需要优化（如使用物化视图）
4. **错误处理**：查询失败时的用户反馈可以更丰富

### 数据流时序图

```
用户操作（选择时间/过滤器/分组）
        │
        ▼
analysisChart.tsx: setState 更新状态
        │
        ▼
useAnalysisChartQuery: queryKey 变化
        │
        ▼
TanStack Query: 发起新请求
        │
        ▼
GET /api/analysis/chart-data
        │
        ▼
analysisController.ts: getChartData
        │
        ▼
analysis.ts: 构建 ClickHouse SQL
        │
        ▼
ClickHouse: 执行查询
        │
        ▼
返回 ChartDataPoint[]
        │
        ▼
chartData useMemo: 转换为 Recharts 格式
        │
        ▼
LineChart: 重新渲染
```
