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

#### 自动刷新（React Query 默认触发机制）

虽然分析面板**没有显式设置 `refetchInterval`**（即无固定时间轮询），但 React Query 提供了多种隐式自动刷新机制。由于项目使用默认配置的 `QueryClient`（`packages/dashboard/src/components/app.tsx:28`），以下默认行为全部生效：

##### 1. 窗口重新获得焦点（refetchOnWindowFocus: true，默认启用）

**触发条件**：
- 用户从其他浏览器标签页切回 Dittofeed 标签
- 用户从其他应用切回浏览器

**行为**：
- React Query 自动将 staleTime 已过期的查询标记为需要刷新
- 分析面板的图表/摘要查询由于 `staleTime = 0`（默认值），每次切回标签都会触发重新查询
- 资源名称查询由于 `staleTime = 5 * 60 * 1000`，切回时仅在超过 5 分钟后才刷新

**对分析面板的实际影响**：
- 用户打开其他页面一段时间后回到分析面板，数据会自动更新
- 这是「被动实时」的主要来源

##### 2. 网络重连（refetchOnReconnect: true，默认启用）

**触发条件**：
- 设备从离线状态恢复在线
- 网络连接发生切换（Wi-Fi → 移动数据等）

**行为**：
- 所有 stale 查询自动重新获取
- 分析面板的图表/摘要查询每次网络恢复都会刷新

**对分析面板的实际影响**：
- 解决了离线期间数据可能过期的问题
- 移动端用户从地铁/电梯出来后自动同步最新数据

##### 3. Stale 查询重新挂载（refetchOnMount: true，默认启用）

**触发条件**：
- 组件从 DOM 中卸载后重新挂载
- 典型场景：用户离开分析页面导航到其他页面，再返回
- 或者：父组件状态变化导致子组件条件渲染

**行为**：
- 如果查询数据被标记为 stale，重新挂载时自动刷新
- 分析面板查询始终为 stale（无 staleTime），因此每次返回页面都会重新获取

**对分析面板的实际影响**：
- 导航到「用户」「设置」等页面再返回分析页面，自动刷新
- 这是「按需」刷新的补充

##### 4. 查询键变化触发刷新（显式逻辑）

**触发条件**：
- 任何包含在查询键中的参数发生变化
- 分析面板的查询键：`["analysisChart", { startDate, endDate, granularity, groupBy, filters, workspaceId }]`

**行为**：
- React Query 检测查询键引用变化，发起新请求
- 配合 `keepPreviousData`，旧数据保留显示直到新数据返回

**对分析面板的实际影响**：
- 切换时间范围、过滤器、分组维度时自动刷新
- 这是最频繁的刷新触发点

#### 各刷新机制的关系与优先级

```
┌─────────────────────────────────────────────────────────────────┐
│                    Dittofeed 分析面板刷新机制                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  显式触发（用户主动）                                             │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  手动刷新按钮                                            │    │
│  │  └── onClick → 更新 dateRange → 查询键变化 → 发起请求     │    │
│  └────────────────────────────────────────────────────────┘    │
│                         ↓                                       │
│  显式触发（用户操作界面）                                         │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  切换时间范围/过滤器/分组维度                              │    │
│  │  └── setState → 查询键变化 → 发起请求                     │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │
│                                                                 │
│  隐式触发（React Query 默认行为，无代码配置）                      │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  1. 窗口焦点恢复 (refetchOnWindowFocus: true)           │    │
│  │     └── 切回标签 → 检测 stale → 发起请求                  │    │
│  │                                                        │    │
│  │  2. 网络重连 (refetchOnReconnect: true)                │    │
│  │     └── 网络恢复 → 检测 stale → 发起请求                  │    │
│  │                                                        │    │
│  │  3. 组件重新挂载 (refetchOnMount: true)                │    │
│  │     └── 页面切换回分析 → 检测 stale → 发起请求            │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
│  无固定轮询（无 refetchInterval）                                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  ❌ 不会每隔固定时间自动刷新                              │    │
│  │  ❌ 区别于其他模块（如 usersTableV2 的 autoReload 开关） │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 关键对比：无固定轮询 vs 事件驱动刷新

| 维度 | 固定轮询 (`refetchInterval`) | Dittofeed 分析面板（事件驱动） |
|-----|-----------------------------|------------------------------|
| 触发时机 | 每隔 X 毫秒 | 用户操作 + React Query 默认事件 |
| 网络开销 | 持续消耗 | 按需消耗 |
| 实时性 | 可预测但可能延迟 | 事件触发时立即更新 |
| 数据新鲜度 | 轮询间隔内可能陈旧 | 焦点恢复/网络恢复时确保新鲜 |
| 实现复杂度 | 简单 | 依赖框架默认行为 |

**Dittofeed 的设计哲学**：
1. 不使用固定轮询 → 避免不必要的 ClickHouse 查询压力
2. 依赖 React Query 的「智能刷新」→ 在用户可能需要数据时恰好更新
3. 手动刷新按钮作为兜底 → 用户可以强制获取最新数据

#### 与其他模块的对比

搜索代码库发现两种截然不同的刷新策略：

**策略 A：固定轮询（用于高实时性需求）**
- `packages/dashboard/src/components/usersTableV2.tsx:895,902`
  ```typescript
  refetchInterval: autoReload ? reloadPeriodMs : false  // 用户可开关
  ```
- `packages/dashboard/src/components/recomputedRecently.tsx:67`
  ```typescript
  refetchInterval: 5 * 1000  // 硬编码 5 秒
  ```
- **适用场景**：用户列表（实时监控用户变化）、重新计算状态（需要快速反馈）

**策略 B：事件驱动（用于分析面板）**
- 分析面板的图表/摘要查询：无 `refetchInterval`
- 依赖 `refetchOnWindowFocus`、`refetchOnReconnect`、`refetchOnMount`
- **适用场景**：分析数据（用户查看时才需要更新，避免后台频繁查询）

#### 实际用户体验流程示例

```
场景 1：用户在分析面板停留查看
────────────────────────────────
[10:00] 进入分析页面 → 首次查询，显示数据
[10:01] 切换到「最近 24 小时」→ 查询键变化 → 刷新
[10:05] 切到其他标签页
[10:10] 切回分析标签 → refetchOnWindowFocus 触发 → 自动刷新 ← 隐式刷新

场景 2：网络中断恢复
──────────────────────────
[14:00] 正在查看分析数据
[14:01] 网络断开（Wi-Fi 掉线）
[14:03] 网络恢复 → refetchOnReconnect 触发 → 自动刷新 ← 隐式刷新

场景 3：页面导航
─────────────────
[16:00] 在分析页面
[16:02] 点击侧边栏「用户」→ AnalysisChart 组件卸载
[16:05] 点击「分析」→ 组件重新挂载 → refetchOnMount 触发 → 自动刷新 ← 隐式刷新

场景 4：手动刷新兜底
────────────────────
[09:00] 数据看起来可能过时
[09:00] 点击刷新按钮 → 更新 dateRange → 查询键变化 → 刷新 ← 显式刷新
```

### 3.4 数据新鲜度边界量化分析

在无固定轮询的设计下，数据新鲜度取决于用户行为和 React Query 的缓存生命周期。以下是三种典型场景的量化分析：

---

#### 场景一：页面持续驻留（组件未卸载）

**用户行为**：打开分析面板后，长时间停留在该页面，不切换标签、不导航到其他页面。

**缓存状态**：
- 查询持续处于「活跃」状态（observerCount > 0）
- 不会触发缓存回收（gcTime 不计时）

**刷新时机**：
| 触发条件 | 时间点 | 延迟说明 |
|---------|-------|---------|
| 首次进入页面 | T0 | 立即查询 |
| 切换时间/过滤/分组 | 用户操作时刻 | 立即查询 |
| 窗口焦点恢复 | 用户切回标签时 | 立即刷新 |
| 网络重连 | 网络恢复时 | 立即刷新 |

**数据新鲜度边界**：
- **最坏延迟**：无上限（如果用户持续停留且不触发任何刷新事件）
- **被动刷新点**：仅依赖用户切出再切回标签（refetchOnWindowFocus）
- **典型延迟**：取决于用户切换标签的频率

**示例时序**：
```
T00:00: 用户进入分析页面 → 查询 Q1，显示数据 D0
T00:05: 用户切换到其他标签页
T00:30: 用户切回分析标签 → refetchOnWindowFocus 触发 → 查询 Q2，显示数据 D1
          数据延迟：30 分钟

T01:00: 用户切出
T02:00: 用户切回 → 查询 Q3，显示数据 D2
          数据延迟：1 小时

如果用户持续驻留不切出：
T00:00 → T03:00 数据仍为 D0
          数据延迟：3 小时（无上限）
```

**补救措施**：
- 手动刷新按钮（仅非 custom 时间范围可用）
- 切换过滤条件/时间范围（立即刷新）

---

#### 场景二：短时离开后返回（缓存命中）

**用户行为**：离开分析页面，但在 `gcTime`（默认 5 分钟）内返回。

**缓存配置**：
- 项目未自定义 `gcTime` → 使用 React Query v5 默认值：**5 分钟 (300,000ms)**
- 查询「失活」后开始计时，超过 gcTime 后缓存被回收

**刷新时机**：
- 组件重新挂载时触发 `refetchOnMount: true`（默认）
- 由于 `staleTime = 0`，挂载时一定发起新查询
- 但旧数据仍在缓存中作为 `placeholderData`（如果设置了）

**数据新鲜度边界**：
| 离开时长 | 缓存状态 | 刷新行为 | 最坏延迟 |
|---------|---------|---------|---------|
| < 5 分钟 | 缓存存活 | 挂载时立即刷新 | 离开时长 + 查询耗时 |
| 5-30 秒 | 缓存存活 | 挂载时立即刷新 | ~30 秒 + 查询耗时 |
| 30 秒-5 分钟 | 缓存存活 | 挂载时立即刷新 | ~5 分钟 + 查询耗时 |

**关键观察**：
- 虽然缓存未被回收，但由于 `staleTime = 0`，**挂载时一定会发起新查询**
- 旧数据仅用于显示占位（`keepPreviousData` 效果）
- 「缓存命中」在这里指「旧数据可用于平滑过渡」，而非「跳过新查询」

**示例时序**：
```
T00:00: 在分析页面 → 查询 Q1，数据 D0
T00:01: 点击「用户」页面 → AnalysisChart 卸载，缓存开始计时
T00:03: 点击「分析」页面 → 离开 2 分钟（< 5 分钟）
        → 缓存存活，refetchOnMount 触发
        → 发起 Q2，期间显示 D0（keepPreviousData）
        → Q2 返回 D1，更新显示
        数据延迟：2 分钟（离开期间的数据未更新） + 查询耗时

T00:05: 再次离开
T00:09: 再次返回 → 离开 4 分钟（< 5 分钟）
        → 同样立即刷新
        数据延迟：4 分钟 + 查询耗时
```

---

#### 场景三：超过 gcTime 后返回（缓存回收）

**用户行为**：离开分析页面超过 5 分钟后返回。

**缓存状态**：
- 查询已被标记为「失活」超过 gcTime
- React Query 的垃圾回收器已清理缓存数据
- 缓存中无该查询的数据

**刷新时机**：
- 组件挂载时发现无缓存数据
- 立即发起新查询（无旧数据可用于占位）
- 用户可能看到 Loading 状态

**数据新鲜度边界**：
| 离开时长 | 缓存状态 | 刷新行为 | 最坏延迟 |
|---------|---------|---------|---------|
| 5-10 分钟 | 已回收 | 挂载时立即查询 | 离开时长 + 查询耗时 |
| 1 小时 | 已回收 | 挂载时立即查询 | 1 小时 + 查询耗时 |
| 1 天 | 已回收 | 挂载时立即查询 | 1 天 + 查询耗时 |

**关键区别**：
- 无 `placeholderData` 可用，用户看到空白/Loading
- 但查询行为与场景二相同（都会刷新）
- 「缓存回收」影响的是用户体验（是否有旧数据过渡），而非是否刷新

**示例时序**：
```
T00:00: 在分析页面 → 查询 Q1，数据 D0
T00:01: 离开页面 → 缓存开始计时
T00:06: gcTime 计时结束（5 分钟）→ 缓存被回收
T00:10: 返回页面 → 离开 9 分钟（> 5 分钟）
        → 无缓存数据，立即发起 Q2
        → 用户看到 Loading 状态
        → Q2 返回 D1，更新显示
        数据延迟：9 分钟 + 查询耗时

T01:00: 再次离开
T02:30: 再次返回 → 离开 1.5 小时
        → 缓存已回收，立即刷新
        数据延迟：1.5 小时 + 查询耗时
```

---

#### 三种场景对比总结

| 维度 | 场景一：持续驻留 | 场景二：短时离开（< 5 分钟） | 场景三：长时离开（> 5 分钟） |
|-----|----------------|---------------------------|---------------------------|
| 组件状态 | 持续挂载 | 卸载后重新挂载 | 卸载后重新挂载 |
| 缓存状态 | 活跃（不计时） | 存活但 stale | 已回收 |
| 刷新触发 | refetchOnWindowFocus | refetchOnMount | refetchOnMount |
| 占位数据 | 有（当前数据） | 有（keepPreviousData） | 无（Loading） |
| 最坏延迟 | 无上限（取决于用户行为） | 离开时长 | 离开时长 |
| 典型延迟 | 取决于切标签频率 | 分钟级 | 小时级 |
| 手动刷新可用 | 是（非 custom） | 是（非 custom） | 是（非 custom） |

---

### 3.5 Custom 时间范围的手动刷新禁用与兜底策略

#### 为什么禁用？

代码位置：`packages/dashboard/src/components/analysisChart.tsx:733`

```typescript
disabled={state.selectedTimeOption === "custom"}
```

**设计原因**（第 390-404 行的 `onRefresh` 逻辑）：

```typescript
const onRefresh = useCallback(() => {
  setState((draft) => {
    const option = timeOptions.find((o) => o.id === draft.selectedTimeOption);
    if (option === undefined || option.type !== "minutes") {
      return;  // custom 类型直接返回，不执行刷新
    }
    const endDate = new Date();
    const startDate = subMinutes(endDate, option.minutes);
    draft.dateRange = {
      startDate: startDate.toISOString(),
      endDate: endDate.toISOString(),
    };
  });
}, [setState]);
```

**关键逻辑**：
- 预设时间范围（如「最近 7 天」）：`option.type === "minutes"`
  - 可以通过 `subMinutes(endDate, option.minutes)` 重新计算
  - 刷新 = 更新 endDate 为当前时间 + 重新计算 startDate
- 自定义时间范围（custom）：`option.type === "custom"`
  - 用户手动选择了绝对时间范围（如「2026-05-01 至 2026-05-10」）
  - 没有「minutes」属性，无法自动推导「刷新到当前」的语义
  - 如果强行刷新，应该更新什么？保持范围不变？还是扩展到当前？语义不明确

**时间选项类型定义**（第 62-109 行）：

```typescript
type TimeOption =
  | { type: "minutes"; id: string; minutes: number; label: string }
  | { type: "custom"; id: "custom"; label: string };

// 预设选项示例
{ type: "minutes", id: "last-7-days", minutes: 7 * 24 * 60, label: "Last 7 days" }
{ type: "custom", id: "custom", label: "Custom Date Range" }
```

---

#### 兜底策略分析

当手动刷新按钮禁用时，用户仍有多种方式获取最新数据：

| 策略 | 适用场景 | 操作方式 | 效果 |
|-----|---------|---------|------|
| **重新选择时间范围** | 所有场景 | 点击 DateRangeSelector，选择预设或重新选择 custom | 查询键变化 → 立即刷新 |
| **窗口焦点恢复** | 切出后切回 | 用户自然行为 | refetchOnWindowFocus → 自动刷新 |
| **页面导航** | 离开后返回 | 点击其他页面再回来 | refetchOnMount → 自动刷新 |
| **网络重连** | 离线恢复 | 系统自动检测 | refetchOnReconnect → 自动刷新 |
| **修改过滤条件** | 有过滤器时 | 添加/移除/修改过滤器 | 查询键变化 → 立即刷新 |
| **修改分组维度** | 有分组时 | 切换 groupBy | 查询键变化 → 立即刷新 |

**兜底策略优先级**：

```
最高优先级（用户主动操作，立即生效）
├── 1. 重新选择时间范围（最直接）
│       选择预设 → 立即刷新
│       重新选择 custom → 立即刷新
│
├── 2. 修改过滤条件
│       添加/移除 journeyId → 立即刷新
│       切换 channel → 立即刷新
│
└── 3. 修改分组维度
        切换 groupBy → 立即刷新

中等优先级（系统被动触发）
├── 4. 窗口焦点恢复
│       用户切出再切回 → 自动刷新
│       不需要额外操作，但依赖用户行为
│
├── 5. 页面导航
│       离开分析页再返回 → 自动刷新
│       成本较高（需要导航）
│
└── 6. 网络重连
        离线恢复 → 自动刷新
        不可控（依赖网络状况）

最低优先级（不可用）
└── 7. 手动刷新按钮
        custom 时间范围时禁用
```

---

#### Custom 时间范围的实际用户流程

```
场景：用户选择了 custom 时间范围
─────────────────────────────────

[10:00] 用户打开 DateRangeSelector
       选择 startDate: 2026-05-01 00:00
       选择 endDate:   2026-05-10 23:59
       → 查询 Q1，显示该范围的数据 D1

[10:05] 用户想刷新数据
       发现刷新按钮被禁用（灰色，不可点击）
       → 方式 A：点击 DateRangeSelector → 重新确认选择 → 刷新
       → 方式 B：切到其他标签再切回 → 自动刷新
       → 方式 C：切换到其他页面再回来 → 自动刷新

[10:10] 用户选择了方式 A
       打开 DateRangeSelector
       点击「Apply」（即使不修改时间）
       → dateRange 状态更新 → 查询键变化 → Q2 刷新
       显示最新数据 D2（仍为 5.1-5.10 范围）
```

---

#### 设计评估

**优点**：
1. **语义明确**：避免了「custom 范围刷新到底应该做什么」的歧义
2. **避免误操作**：防止用户以为刷新会扩展时间范围到当前
3. **引导用户**：通过 DateRangeSelector 明确时间范围的控制权

**潜在问题**：
1. **发现性差**：用户可能不知道为什么按钮被禁用
2. **操作成本**：重新选择时间范围需要额外点击
3. **无提示**：没有 Tooltip 说明禁用原因和替代方式

**改进建议**：
- 保持禁用逻辑，但添加禁用原因的 Tooltip
- 考虑为 custom 范围添加「刷新但保持范围」的选项（语义：用相同范围重新查询）
- 或者：custom 范围下刷新按钮改为「重新查询此范围」

---

### 3.6 缓存失效

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

1. **无固定轮询是设计选择而非缺失**：分析面板没有 `refetchInterval` 是有意为之（避免 ClickHouse 压力），而非缺失。但需要意识到：
   - **持续驻留场景**：数据延迟无上限，完全依赖用户切出再切回标签
   - **可考虑的改进**：
     - 添加「自动刷新」开关（类似 `usersTableV2` 的 `autoReload`）
     - 为短时间范围（如「最近 15 分钟」）自动启用轮询
     - 为分析面板添加「数据最后更新时间」显示，让用户感知新鲜度

2. **缓存策略的取舍**：图表数据没有 `staleTime` 是为了配合 React Query 的默认刷新机制（焦点恢复/网络重连时立即刷新）。但需要意识到：
   - **gcTime = 5 分钟**：短时离开（< 5 分钟）返回时有旧数据作为过渡，长时离开（> 5 分钟）返回时看到 Loading
   - **可考虑的改进**：
     - 为长时间范围（如「最近 30 天」）设置适度的 `staleTime`（如 30 秒）
     - 考虑自定义 `gcTime`（如延长到 10 分钟）以提供更好的导航过渡体验

3. **Custom 时间范围的体验问题**：手动刷新按钮禁用是合理的语义设计，但用户体验可以优化：
   - 添加禁用原因的 Tooltip 提示（「Custom 范围请使用 DateRangeSelector 刷新」）
   - 考虑为 custom 范围添加「重新查询此范围」的替代按钮
   - 或者：简化逻辑，custom 范围的「刷新」=「用相同参数重新查询」

4. **查询性能**：对于大数据量，CTE 方式可能需要优化（如使用物化视图、预聚合表）

5. **错误处理**：查询失败时的用户反馈可以更丰富（区分网络错误、查询超时等）

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
