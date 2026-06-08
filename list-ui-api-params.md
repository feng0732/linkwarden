# List UI 状态与 API 参数映射协作分析

## 一、整体架构概览

Linkwarden 的 List UI 采用 **多层状态管理** 架构：

```
UI 交互层 (SearchBar / SortDropdown / ViewDropdown)
        ↓
页面局部状态 (useState: viewMode / sortBy / editMode)
        ↓
持久化层 (localStorage: viewMode / sortBy / columns)
        ↓
Zustand Store (useLocalSettingsStore / useLinkStore)
        ↓
数据层 (React Query: useLinks / useCollections / useDashboardData)
        ↓
API 层 (/api/v1/search / /api/v1/links / /api/v1/public/...)
```

---

## 二、前端筛选状态管理

### 2.1 筛选参数类型定义

[LinkRequestQuery](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/types/global.ts#L105-L112) 定义了所有可传递的查询参数：

```typescript
export type LinkRequestQuery = {
  sort?: Sort;
  cursor?: number;
  collectionId?: number;
  tagId?: number;
  pinnedOnly?: boolean;
  searchQueryString?: string;
};
```

### 2.2 各页面的筛选参数来源

| 页面 | collectionId | tagId | pinnedOnly | searchQueryString | sort |
|------|-------------|-------|-----------|-------------------|------|
| [links/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/pages/links/index.tsx#L24-L26) | - | - | - | - | localStorage |
| [links/pinned.tsx](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/pages/links/pinned.tsx#L22-L25) | - | - | **true** | - | localStorage |
| [collections/[id].tsx](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/pages/collections/[id].tsx#L51-L54) | **router.query.id** | - | - | - | localStorage |
| [tags/[id].tsx](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/pages/tags/[id].tsx#L54-L57) | - | **router.query.id** | - | - | localStorage |
| [search.tsx](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/pages/search.tsx#L38-L41) | - | - | - | **router.query.q** | localStorage |

### 2.3 易遗漏点分析

1. **参数来源分散**：`collectionId` / `tagId` 来自 `router.query.id`，但不同页面解析逻辑不同，容易在新增路由时忘记对应
2. **隐式依赖 localStorage**：`sort` 默认从 `localStorage.getItem("sortBy")` 读取，而不是通过 props 传递
3. **searchQueryString 需要 decodeURIComponent**：见 [search.tsx](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/pages/search.tsx#L40)，但其他传入位置若忘记解码会导致搜索不到

---

## 三、分页与排序协作机制

### 3.1 游标分页实现

[useLinks](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L26-L60) 使用 React Query 的 `useInfiniteQuery` 实现无限滚动：

```typescript
// 前端：queryKey 包含完整 params，确保参数变化时触发重新查询
queryKey: ["links", { params }],
initialPageParam: 0,
getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
```

**前端分页触发**：在 [Links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/components/LinkViews/Links.tsx#L376-L387) 中通过 `useInView` 监听底部占位元素：

```typescript
useEffect(() => {
  if (!inView) return;
  if (!useData.hasNextPage) return;
  if (useData.isFetchingNextPage) return; // 防止重复请求
  useData.fetchNextPage();
}, [inView, useData.hasNextPage, useData.isFetchingNextPage, useData.fetchNextPage]);
```

### 3.2 服务端分页逻辑

[searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts) 中有两种分页模式：

| 模式 | 游标类型 | 分页方式 | nextCursor 计算 |
|------|---------|---------|----------------|
| Meilisearch | offset (数字偏移) | `offset + limit` | `hits.length === limit ? offset + limit : null` |
| Prisma (无 Meili) | id (主键游标) | `cursor: { id } + skip: 1` | `links.length === take ? lastLink.id : null` |

### 3.3 排序参数映射

前端 [Sort 枚举](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/types/global.ts#L87-L92) 到后端排序字段的映射：

| Sort 值 | 枚举名 | Prisma orderBy | Meilisearch sort |
|--------|--------|---------------|------------------|
| 0 | DateNewestFirst | `{ id: "desc" }` | `["id:desc"]` |
| 1 | DateOldestFirst | `{ id: "asc" }` | `["id:asc"]` |
| 2 | NameAZ | `{ name: "asc" }` | `["name:asc"]` |
| 3 | NameZA | `{ name: "desc" }` | `["name:desc"]` |

### 3.4 查询字符串构建

[buildQueryString](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L106-L116) 的过滤逻辑：

```typescript
Object.keys(params)
  .filter((key) => params[key] !== undefined)  // 仅过滤 undefined，不过滤 falsy 值
  .map((key) => `${encodeURIComponent(key)}=${encodeURIComponent(params[key])}`)
  .join("&");
```

**⚠️ 潜在风险**：`pinnedOnly: false` 会被序列化为 `pinnedOnly=false` 发送，但后端仅检查 `pinnedOnly && userId`，因此值为 `false` 时不会触发 pinned 过滤。然而对于布尔参数的语义可能造成混淆。

---

## 四、筛选变更后的分页复位

### 4.1 复位函数实现

[resetInfiniteQueryPagination](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L1124-L1138) 是核心复位逻辑：

```typescript
const resetInfiniteQueryPagination = async (queryClient, queryKey) => {
  // 1. 手动截断：只保留第一页数据
  queryClient.setQueriesData({ queryKey }, (oldData) => {
    if (!oldData) return undefined;
    return {
      pages: oldData.pages.slice(0, 1),
      pageParams: oldData.pageParams.slice(0, 1),
    };
  });

  // 2. 发起失效查询，触发重新获取第一页
  await queryClient.invalidateQueries(queryKey);
};
```

### 4.2 复位触发场景

目前仅在 [SortDropdown](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/components/SortDropdown.tsx#L30-L33) 中排序变更时调用：

```typescript
const handleValueChange = (value: string) => {
  resetInfiniteQueryPagination(queryClient, ["links"]);
  setSort(value as unknown as Sort);
};
```

### 4.3 易遗漏的复位场景

| 场景 | 是否复位 | 代码位置 | 风险 |
|------|---------|---------|------|
| sortBy 改变 | ✅ 是 | [SortDropdown.tsx](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/components/SortDropdown.tsx#L30-L33) | - |
| collectionId 改变 (路由跳转) | ⚠️ 隐式 | 路由变化触发新 queryKey | 依赖 queryKey 变化自动重新查询，但已有旧数据可能短暂闪烁 |
| tagId 改变 (路由跳转) | ⚠️ 隐式 | 同上 | 同上 |
| searchQueryString 改变 (路由跳转) | ⚠️ 隐式 | 同上 | 同上 |
| pinnedOnly 改变 | ❌ 未显式 | 无 | /links → /links/pinned 切换时依赖路由重载 |
| viewMode 改变 | ❌ 不需要 | 纯 UI 展示 | 不影响数据获取 |

**关键问题**：在同一页面内动态改变筛选参数（非路由跳转场景）时，若未手动调用 `resetInfiniteQueryPagination`，则 React Query 会因为 queryKey 变化而自动创建新的查询，但旧查询的缓存仍然存在。

---

## 五、数据刷新机制

### 5.1 手动轮询刷新

在 [Links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/components/LinkViews/Links.tsx#L405-L425) 中针对未完成存档的链接进行每 5 秒轮询：

```typescript
useEffect(() => {
  let interval = null;
  if (links?.some(e => !e.preview?.startsWith("archives") && e.preview !== "unavailable")) {
    interval = setInterval(async () => {
      useData.refetch().catch(console.error);
    }, 5000);
  }
  return () => interval && clearInterval(interval);
}, [links]);
```

### 5.2 React Query 失效驱动刷新

Mutation 成功后通过 `invalidateQueries` 触发刷新：

| Mutation | 失效的 queryKey | 代码位置 |
|----------|-----------------|---------|
| useAddLink | dashboardData, collections, tags, publicLinks | [links.tsx:543-546](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L543-L546) |
| useUpdateLink | links, link[id], dashboardData, collections, tags, publicLinks | [links.tsx:698-703](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L698-L703) |
| useDeleteLink | dashboardData, collections, tags, publicLinks | [links.tsx:805-808](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L805-L808) |

### 5.3 乐观更新机制

以 [useAddLink](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L426-L520) 为例的完整乐观更新流程：

```
onMutate:
  1. cancelQueries(["links"], ["dashboardData"])    // 阻止并发覆盖
  2. 快照备份 previousLinks / previousDashboard     // 用于回滚
  3. 生成 tempId = -Date.now()                       // 客户端临时 ID
  4. setQueriesData → upsertLinkInInfiniteData      // 插入乐观数据
  5. return context { previousLinks, optimisticId }  // 传递上下文

onError:
  1. 遍历 context.previousLinks 还原数据              // 回滚
  2. toast.error 提示

onSuccess:
  1. setQueriesData → 用服务端返回的真实 ID 替换 tempId
  2. invalidateQueries 刷新相关缓存
```

---

## 六、并发更新影响分析

### 6.1 竞态条件与保护

| 保护机制 | 位置 | 作用 |
|---------|------|------|
| `cancelQueries` onMutate | 所有 mutation | 防止进行中的查询结果覆盖乐观更新 |
| `isFetchingNextPage` 守卫 | [Links.tsx:379](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/components/LinkViews/Links.tsx#L379) | 防止无限滚动重复请求 |
| `setQueriesData` 同步更新 | mutation onMutate/onSuccess | 本地状态立即更新，避免等待网络 |

### 6.2 多 queryKey 数据同步问题

`useLinks` 的 queryKey 为 `["links", { params }]`，不同筛选参数会产生不同缓存条目。Mutation 使用 `setQueriesData({ queryKey: ["links"] })` 匹配**所有**以 `["links"]` 开头的 key。

**存在的同步风险**：
1. 用户在 `/collections/1` 编辑了链接 → `setQueriesData` 更新所有 `["links", ...]` 缓存
2. 用户切换到 `/collections/2` → 若该缓存已存在，数据是正确同步的
3. 但若 `/collections/2` 缓存尚未加载，后续加载时会从服务端获取最新数据 → **最终一致**

### 6.3 queryKey 参数敏感性

[buildQueryString](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L34-L48) 通过 `useMemo` 依赖：

```typescript
const queryString = useMemo(() => buildQueryString({
  sort, collectionId, tagId, pinnedOnly, searchQueryString
}), [sort, params.collectionId, params.tagId, params.pinnedOnly, params.searchQueryString]);
```

**易遗漏**：若新增 `LinkRequestQuery` 字段（如 `type` 过滤），需同时：
1. 添加到 `buildQueryString` 参数
2. 添加到 `useMemo` 依赖数组
3. 后端 `searchLinks.ts` 对应处理
4. 类型 `LinkRequestQuery` 定义

---

## 七、空态处理

### 7.1 空态判断逻辑

各页面统一的判断模式（以 [links/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/pages/links/index.tsx#L57-L59) 为例）：

```typescript
{!data.isLoading && links && !links[0] && <NoLinksFound text={...} />}
```

**判断三元组**：
1. `!data.isLoading` — 必须加载完成（排除骨架屏阶段）
2. `links` — 数据存在（排除 undefined）
3. `!links[0]` — 数组为空（第一个元素不存在）

### 7.2 不同场景的空态组件

| 场景 | 空态组件 | 说明 |
|------|---------|------|
| collection/tag 无结果 | [NoLinksFound.tsx](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/components/NoLinksFound.tsx) | 含"创建新链接"按钮 |
| search 无结果 | 内联 `<p>{t("nothing_found")}</p>` | [search.tsx:58](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/pages/search.tsx#L58)，仅文字提示 |
| pinned 无结果 | 内联 SVG + 文案 | [pinned.tsx:48-L66](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/pages/links/pinned.tsx#L48-L66)，含引导说明 |
| dashboard recent/pinned 无结果 | 内联引导卡片 | [dashboard.tsx:294-L317](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/pages/dashboard.tsx#L294-L317)，含 Add Link / Import 按钮 |

### 7.3 空态风险点

1. **Loading 与 Empty 的切换**：骨架屏（skeleton）在 `(hasNextPage || isLoading)` 时显示 [Links.tsx:141](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/components/LinkViews/Links.tsx#L141)，当第一页就是空数据时，`isLoading` 结束 + `links` 为空 → 空态显示，骨架屏消失 —— 该时序正确
2. **`links` 依赖 `dataUpdatedAt`**：[links.tsx:52-L54](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L52-L54) 中 `links` 的 useMemo 依赖 `query.dataUpdatedAt` 而非 `query.data`，确保 mutation 乐观更新后视图正确刷新

---

## 八、搜索高级查询参数

### 8.1 搜索语法解析

[parseSearchTokens](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/lib/api/searchQueryBuilder.ts#L20-L83) 将查询字符串解析为 Token：

```
"tag:react !pinned:true after:2024-01-01 hello world"
  ↓
[
  { field: "tag", value: "react", isNegative: false },
  { field: "pinned", value: "true", isNegative: true },
  { field: "after", value: "2024-01-01", isNegative: false },
  { field: "general", value: "hello", isNegative: false },
  { field: "general", value: "world", isNegative: false }
]
```

支持的搜索字段：`url`, `name`, `description`, `type`, `collection`, `pinned`, `public`, `before`, `after`, `tag`，其余归类为 `general`（全文搜索）。

### 8.2 前端 SearchBar 参数传递

[SearchBar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/components/SearchBar.tsx#L93-L111) 仅在 Enter 时通过 URL query 跳转：

```typescript
onKeyDown={(e) => {
  if (e.key === "Enter") {
    router.push("/search?q=" + encodeURIComponent(searchQuery));
  }
}}
```

**设计选择**：搜索不使用实时筛选，而是路由跳转 → queryKey 变化 → 全新查询。这避免了高频输入引发的并发请求，但代价是无法在当前列表页即时过滤。

---

## 九、总结：协作流程图

```
用户交互
  │
  ├─ 输入搜索 → Enter → router.push(/search?q=...) → queryKey变化 → 重新useInfiniteQuery
  │
  ├─ 选择排序 → SortDropdown
  │              ├─ resetInfiniteQueryPagination(["links"])  ← 截断缓存
  │              └─ setSort → localStorage + useState → queryString变化 → 重新查询
  │
  ├─ 切换视图 → ViewDropdown → localStorage + useState → 仅UI变化，不触发数据请求
  │
  ├─ 滚动到底部 → useInView → fetchNextPage() → cursor+1 → 追加pages
  │
  └─ CRUD操作 → useAddLink/useUpdateLink/useDeleteLink
                 ├─ onMutate: cancelQueries + 乐观更新 + 快照备份
                 ├─ onError: 回滚快照
                 └─ onSuccess: 替换乐观ID + invalidateQueries刷新
```

## 十、易遗漏 Checklist

1. ☐ 新增 `LinkRequestQuery` 字段时，同步更新：类型定义 → `useLinks` 参数 → `buildQueryString` → `useMemo` 依赖 → 后端 `searchLinks`
2. ☐ 排序/筛选变更时，除了 queryKey 自动变化，考虑是否需要 `resetInfiniteQueryPagination` 截断旧缓存
3. ☐ 新页面使用 `useLinks` 时，检查 `searchQueryString` 是否需要 `decodeURIComponent`
4. ☐ Mutation 的 `setQueriesData` 是否覆盖了所有需要乐观更新的 queryKey 变体
5. ☐ 空态判断必须同时检查 `!isLoading`，避免骨架屏阶段闪烁显示"无数据"
6. ☐ `useMemo`/`useEffect` 的依赖数组是否包含所有参与 queryString 构建的参数
