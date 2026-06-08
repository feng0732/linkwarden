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

### 3.5 请求参数进入搜索接口前的类型转换链路

参数从前端到后端 `searchLinks` 控制器经历 **4 次形态转换**，每一步都有类型丢失或错配风险：

```
前端 LinkRequestQuery (强类型对象)
    │  buildQueryString + encodeURIComponent
    ▼
URL query string (全是字符串)
    │  Next.js 解析为 req.query (string | string[] | undefined)
    ▼
后端路由层手动类型转换 (强转/条件判断)
    │  [/api/v1/search/index.ts] / [/api/v1/public/collections/links/index.ts]
    ▼
searchLinks({ query: LinkRequestQuery, userId, publicOnly })
```

#### 转换细节对照表（以私有接口 [search/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/pages/api/v1/search/index.ts#L15-L28) 为例）

| 参数 | 前端类型 | URL 中形态 | req.query 类型 | 后端转换代码 | 转换后类型 | 边界漏洞 |
|------|---------|-----------|---------------|-------------|-----------|---------|
| sort | `Sort \| undefined` (number enum) | `"sort=0"` | `string \| undefined` | `Number(req.query.sort as string)` | `number` (可能为 NaN) | **不传时为 NaN**，而 NaN 不等于 Sort 枚举任一值，fallback 到 `{ id: "desc" }` (等同于 DateNewestFirst) |
| cursor | `number \| undefined` | `"cursor=50"` / 无 | `string \| undefined` | `req.query.cursor ? Number(...) : undefined` | `number \| undefined` | cursor=0 被当作"无游标"，在 prisma cursor 模式下不会 skip |
| collectionId | `number \| undefined` | `"collectionId=3"` | `string \| undefined` | `req.query.collectionId ? Number(...) : undefined` | `number \| undefined` | 传空串 `""` 走 undefined 分支；传 `"abc"` 转为 NaN 进入查询会查不到结果 |
| tagId | `number \| undefined` | `"tagId=7"` | `string \| undefined` | `req.query.tagId ? Number(...) : undefined` | `number \| undefined` | 同上 |
| pinnedOnly | `boolean \| undefined` | `"pinnedOnly=true"` / `"pinnedOnly=false"` / 无 | `string \| undefined` | `req.query.pinnedOnly ? req.query.pinnedOnly === "true" : undefined` | `boolean \| undefined` | `"false"` 字符串被视为 truthy，会正确返回 false；**不传和传空串都返回 undefined**，但前端若传 pinnedOnly=false 则 URL 中有值且被转为 false，后端不会走 pinned 过滤 |
| searchQueryString | `string \| undefined` | `"searchQueryString=hello"` | `string \| undefined` | 原样透传 | `string \| undefined` | 前端 SearchBar 用 `router.query.q`，但 useLinks 接收的参数名是 `searchQueryString`，两者通过页面组件中转时需 rename |

#### 公共接口 vs 私有接口的参数差异

公共接口 [public/collections/links/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/apps/web/pages/api/v1/public/collections/links/index.ts#L11-L23) **缺少 `tagId` 字段处理**：

```typescript
// 私有接口：支持全部 6 个字段
const convertedData: LinkRequestQuery = { sort, cursor, collectionId, tagId, pinnedOnly, searchQueryString };

// 公共接口：只有 5 个字段，无 tagId
const convertedData: LinkRequestQuery = { sort, cursor, collectionId, pinnedOnly, searchQueryString };
```

这意味着**公共集合页面无法按标签筛选**，不是 bug 而是设计，但在新增参数时两处需同步更新。

#### 易漏类型转换 Checklist

1. ☐ 新增 `LinkRequestQuery` 字段时，**两个**路由文件（私有 + 公共）都要加转换逻辑
2. ☐ Number() 转换后若为 NaN，应 fallback 到 undefined 而非传入查询
3. ☐ `searchQueryString` 前后端参数名不对称：前端 SearchBar 用 `q`，页面需手动映射到 `searchQueryString`

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

### 6.2 多 queryKey 数据同步与筛选缓存错配

`useLinks` 的 queryKey 为 `["links", { params }]`，不同筛选参数会产生**独立的缓存条目**。典型的缓存矩阵如下：

| queryKey | 对应场景 |
|----------|---------|
| `["links", { params: "sort=0" }]` | /links 全部链接 |
| `["links", { params: "sort=0&collectionId=1" }]` | /collections/1 |
| `["links", { params: "sort=0&tagId=5" }]` | /tags/5 |
| `["links", { params: "sort=0&pinnedOnly=true" }]` | /links/pinned |
| `["links", { params: "sort=0&searchQueryString=hello" }]` | /search?q=hello |

Mutation 使用 `setQueriesData({ queryKey: ["links"] })` 匹配**所有**以 `["links"]` 开头的 key——这意味着一次更新会同时写入上述全部 5 个缓存。

#### 6.2.1 错配场景详解：useAddLink 的乐观插入

[upsertLinkInInfiniteData](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L162-L191) 的逻辑是：**已存在则替换，不存在则追加到第一页开头**。

```typescript
const upsertLinkInInfiniteData = (oldData, link, optimisticId) => {
  // ...遍历 pages 尝试替换...
  if (!replaced) {
    // 未找到 → 无脑插入到第一页最前面
    pages[0] = { ...firstPage, links: upsertLinkInList(firstPage?.links ?? [], link, optimisticId) };
  }
  return { ...oldData, pages };
};
```

**典型错配场景**：用户在 `/collections/1` 页面下，通过 NewLinkModal 创建了一个归属 `collectionId=2` 的新链接。各缓存的瞬时状态：

| 缓存 (queryKey) | 应不应该包含新链接 | onMutate 后的实际状态 | 后果 |
|-----------------|-------------------|----------------------|------|
| `params="collectionId=1"` | ❌ 不应包含 | ✅ 被乐观插入 | **错配**：当前页短暂出现一条不属于该集合的链接 |
| `params="collectionId=2"` | ✅ 应该包含 | ✅ 被乐观插入 | 正确（若缓存已存在） |
| `params="collectionId=2"` 尚未加载 | ✅ 应该包含 | ❌ 缓存不存在，无写入 | 正确（首次加载时从服务端获取） |
| `params="tagId=5"` | ❌ 若新链接无此 tag 则不应 | ✅ 被乐观插入 | **错配**：tag 列表混入无关链接 |
| `params="pinnedOnly=true"` | ❌ 若未 pin 则不应 | ✅ 被乐观插入 | **错配**：pinned 页出现未 pin 链接 |
| `params="searchQueryString=xxx"` | 取决于搜索词 | ✅ 被乐观插入 | **错配**：搜索结果页混入不匹配的新链接 |

**修正时序**：onSuccess 中 `invalidateQueries({ queryKey: ["dashboardData", "collections", "tags", "publicLinks"] })` 会触发这些缓存重新获取；但 `["links"]` 缓存**没有被 invalidate**，而是通过 `setQueriesData` 将乐观 ID 替换为真实 ID——意味着上述错配数据会**一直保留**，直到用户切换页面触发新 queryKey 或手动刷新。

#### 6.2.2 错配场景详解：useUpdateLink 的替换

[replaceLinkInInfiniteData](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L307-L328) 的逻辑是：**遍历所有 page，找到匹配 id 则替换，找不到则原样返回**。

```typescript
const replaceLinkInInfiniteData = (oldData, link) => {
  let updated = false;
  const pages = oldData.pages.map(page => ({
    ...page,
    links: page.links.map(item => item.id === link.id ? (updated = true, link) : item)
  }));
  return updated ? { ...oldData, pages } : oldData;  // 未找到则不变更
};
```

**典型错配场景**：用户将链接 A 从 `collectionId=1` 移动到 `collectionId=2`。

| 缓存 | 应不应该包含链接 A | onMutate 后状态 | 后果 |
|------|-------------------|----------------|------|
| `params="collectionId=1"` | ❌ 应该被移除 | ✅ 仍存在（原地替换，未删除） | **错配**：col1 页面残留已移走的链接 |
| `params="collectionId=2"` | ✅ 应该出现 | ❌ 若原缓存中无此 id 则不会被插入 | **错配**：col2 页面看不到新移入的链接 |
| `params="tagId=5"` | 取决于 A 是否仍有 tag5 | 原地替换字段 | 基本正确（若 tag 也变了则仍有短暂错配） |
| `params="pinnedOnly=true"` | 取决于 A 是否仍 pinned | 原地替换 pinnedBy 字段 | 基本正确 |

**修正时序**：useUpdateLink 的 onSuccess **有** `invalidateQueries({ queryKey: ["links"] })`，会触发全量重新获取，所以错配是暂时的。

#### 6.2.3 错配场景详解：useDeleteLink 的移除

[removeLinkFromInfiniteData](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L224-L234) 的逻辑是：**从所有 pages 中过滤掉匹配 id 的链接**。

这个操作是安全的——从所有筛选缓存中删除该 id 都不会引入"不该存在的元素"，最多是从本来就不包含它的缓存中做了一次空过滤。但仍有一个细节：

| 缓存 | onMutate 后 | onSuccess 后 |
|------|------------|-------------|
| 包含该 id 的缓存（如 col1） | ✅ 正确删除 | `setQueriesData` 再次 filter 确保删除 |
| 不包含该 id 的缓存（如 col2） | ⚠️ 无变化（正确） | ⚠️ 无变化 |
| 所有缓存 | - | `invalidateQueries(dashboardData, collections, tags, publicLinks)` |

**注意**：useDeleteLink 的 onSuccess **没有** `invalidateQueries(["links"])`，而是用 `setQueriesData` 手动再删一遍。这意味着删除后不会触发 links 缓存的刷新，依赖乐观删除的准确性——如果 onMutate 删除时有遗漏（例如缓存分页不在第一页），onSuccess 的 filter 也会漏掉。

#### 6.2.4 dashboardData 与 links 缓存的差异

[dashboardData](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/dashboardData.tsx#L16-L35) 是独立缓存（queryKey 为 `["dashboardData"]`，非 `["links", ...]`），结构也不同：

```typescript
// dashboardData 结构（非 infinite pages）
{
  links: Link[],                    // 最近 16 条
  collectionLinks: { [colId]: Link[] },  // 各集合最近 16 条
  numberOfPinnedLinks: number,
  numberOfTags: number,
  ...
}
```

乐观更新函数 [upsertLinkInDashboardData](file:///d:/fz/0601/solo-dogfeeding/code/95-linkwarden/packages/router/links.tsx#L193-L222) 有针对性逻辑：**只把链接插入到对应 collectionId 的 collectionLinks 子列表中**，不会污染其他集合的 dashboard 缓存。这与 `upsertLinkInInfiniteData` 的"全量无脑插入"形成对比——dashboardData 的筛选感知更强，但 links 的分页缓存做不到按筛选条件过滤。

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
