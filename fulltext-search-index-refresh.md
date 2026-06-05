# 全文搜索索引写入、刷新与查询流程

本文档梳理 Linkwarden 项目中全文搜索功能的完整数据流：索引写入时机、刷新机制、查询拼装与结果返回流程。

---

## 一、技术栈概览

项目使用 **Meilisearch** 作为全文搜索引擎，**PostgreSQL** 作为主数据库。

| 组件 | 文件 | 说明 |
|------|------|------|
| Meilisearch 客户端 | [meilisearchClient.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/packages/lib/meilisearchClient.ts) | 根据环境变量 `MEILI_MASTER_KEY` 决定是否启用 |
| 索引版本常量 | [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/packages/lib/constants.ts) | `MEILI_INDEX_VERSION = 1` |
| 索引 Worker | [linkIndexing.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/worker/workers/linkIndexing.ts) | 后台常驻，批量索引处理 |
| 查询构建器 | [searchQueryBuilder.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/searchQueryBuilder.ts) | 解析搜索词、构建过滤器 |
| 搜索控制器 | [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts) | 核心搜索逻辑（私有+公开共用） |
| 私有搜索 API | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/search/index.ts) | `/api/v1/search` 端点 |
| 公开搜索 API | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/public/collections/links/index.ts) | `/api/v1/public/collections/links` 端点 |
| 私有前端 Hook | [links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/packages/router/links.tsx) | `useLinks()` 数据获取 |
| 公开前端 Hook | [publicLinks.tsx](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/packages/router/publicLinks.tsx) | `usePublicLinks()` 数据获取 |

---

## 二、索引写入流程

### 2.1 Meilisearch 客户端初始化

[meilisearchClient.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/packages/lib/meilisearchClient.ts#L1-L10)

```typescript
// 仅当 MEILI_MASTER_KEY 存在时初始化
const apiKey = process.env.MEILI_MASTER_KEY;
export const meiliClient = apiKey
  ? new MeiliSearch({
      host: process.env.MEILI_HOST || "http://meilisearch:7700",
      apiKey,
    })
  : null;
```

### 2.2 索引 Schema 设置

在 Worker 启动时调用 [setupLinksIndexSchema()](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/worker/workers/linkIndexing.ts#L9-L61) 完成初始化：

1. **创建索引**（如果不存在）：主键为 `id`
2. **设置可过滤属性**：`collectionOwnerId`、`collectionMemberIds`、`collectionName`、`tags`、`pinnedBy`、`url`、`type`、`name`、`description`、`collectionIsPublic`、`creationTimestamp`
3. **设置可排序属性**：`id`、`name`

### 2.3 批量获取待索引链接

Worker 通过 [getLinkBatch.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/worker/lib/getLinkBatch.ts) 获取需要索引的链接：

- 查询条件：`indexVersion != MEILI_INDEX_VERSION` 或 `indexVersion IS NULL`
- 额外过滤：订阅状态、试用期等业务条件
- 取数策略：新旧各取一半（`take/2` 旧到新 + `take/2` 新到旧），去重后合并

### 2.4 文档转换与写入

[linkIndexing.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/worker/workers/linkIndexing.ts#L133-L159)

```typescript
// 1. 数据转换（展开关联数据，便于 Meilisearch 过滤）
const docs = links.map((link) => ({
  ...link,
  collectionOwnerId: link.collection.ownerId,      // 用于权限过滤
  collectionMemberIds: link.collection.members.map((m) => m.userId),
  collectionIsPublic: link.collection.isPublic,    // 用于公开搜索过滤
  collectionName: link.collection.name,
  tags: link.tags.map((t) => t.name),
  pinnedBy: link.pinnedBy.map((p) => p.id),
  creationTimestamp: Date.parse(link.createdAt.toISOString()) / 1000,
  indexVersion: MEILI_INDEX_VERSION,
}));

// 2. 写入 Meilisearch
const task = await meiliClient.index("links").addDocuments(docs);
await meiliClient.index("links").waitForTask(task.taskUid);

// 3. 更新数据库 indexVersion 标记为已索引
await prisma.link.updateMany({
  where: { id: { in: ids } },
  data: { indexVersion: MEILI_INDEX_VERSION },
});
```

### 2.5 索引处理循环

[startIndexing()](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/worker/workers/linkIndexing.ts#L63-L179)

- 常驻循环，每 `interval`（默认10秒）执行一次
- 每次处理 `takeCount`（默认50条）链接
- 无待索引链接时 sleep interval 等待

---

## 三、索引刷新时机（触发重新索引）

核心机制：将 `indexVersion` 设置为 `null`，Worker 循环检测到后重新索引。

### 3.1 链接相关操作

| 操作 | 文件 | 位置 |
|------|------|------|
| 创建链接 | [postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/links/postLink.ts) | [L146](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/links/postLink.ts#L146-L146) |
| 更新链接 | [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts) | [L163](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L163-L163) |
| 归档处理 | [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/worker/lib/archiveHandler.ts) | [L57](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/worker/lib/archiveHandler.ts#L57-L57) |
| 手动归档 | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/links/archive/index.ts) | [L82](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/links/archive/index.ts#L82-L82) |
| 单链接归档 | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/links/[id]/archive/index.ts) | [L62](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/links/[id]/archive/index.ts#L62-L62) |
| 归档上传 | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/archives/index.ts) | [L224](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/archives/index.ts#L224-L224) |
| Worker 归档 | [preservation.tsx](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx) | [L59](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L59-L59) |

### 3.2 标签相关操作

| 操作 | 文件 | 位置 |
|------|------|------|
| 更新标签 | [updateTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/tagId/updateTagById.ts) | [L82](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/tagId/updateTagById.ts#L82-L82) |
| 删除标签 | [deleteTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts) | [L43](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts#L43-L43) |
| 合并标签 | [mergeTags.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/mergeTags.ts) | [L71](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/mergeTags.ts#L71-L71) |
| 批量删标签 | [bulkTagDelete.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/bulkTagDelete.ts) | [L60](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/bulkTagDelete.ts#L60-L60) |

标签更新/删除时会批量更新相关链接的 `indexVersion: null`

```typescript
// 例如 updateTagById.ts L73-L84
const linkIds = links.map((link) => link.id);
await prisma.link.updateMany({
  where: { id: { in: linkIds } },
  data: { indexVersion: null },
});
```

### 3.3 集合相关操作

| 操作 | 文件 | 位置 |
|------|------|------|
| 更新集合 | [updateCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts) | [L199](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts#L199-L199) |
| 删除集合 | [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts) | [L45](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L45-L45) |

集合更新/删除时同样批量更新相关链接。

### 3.4 索引刷新时序图

```
用户操作 → 设置 indexVersion = null → Worker 循环检测 → 重新索引 → indexVersion = 1
    ↓
创建/更新链接
更新/删除标签
更新/删除集合
归档处理
```

---

## 四、索引删除

删除操作是**即时**的，不通过 indexVersion 机制。

### 4.1 单条删除

[deleteLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts#L30)

```typescript
await prisma.link.delete({ where: { id: linkId } });
await meiliClient?.index("links").deleteDocument(deleteLink.id);
```

### 4.2 批量删除

[deleteLinksById.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/links/bulk/deleteLinksById.ts#L50)

```typescript
await prisma.link.deleteMany({ where: { id: { in: linkIds } });
await meiliClient?.index("links").deleteDocuments(linkIds);
```

---

## 五、查询拼装流程

### 5.1 支持的搜索条件

[SEARCH_CONDITIONS](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/searchQueryBuilder.ts#L7-L18) 定义了高级搜索操作符：

| 操作符 | 说明 |
|--------|------|
| `url:` | 匹配 URL |
| `name:` | 匹配名称 |
| `description:` | 匹配描述 |
| `type:` | 匹配类型 |
| `collection:` | 匹配集合名 |
| `pinned:` | 是否置顶 |
| `public:` | 是否公开 |
| `before:` | 创建日期之前 |
| `after:` | 创建日期之后 |
| `tag:` | 匹配标签 |
| `!field:` | 否定条件 |

### 5.2 搜索词解析

[parseSearchTokens()](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/searchQueryBuilder.ts#L20-L83)

```typescript
// 输入示例: "react url:github.com !tag:archive after:2024-01-01"
// 输出 Token 数组:
[
  { field: "general", value: "react", isNegative: false },
  { field: "url", value: "github.com", isNegative: false },
  { field: "tag", value: "archive", isNegative: true },
  { field: "after", value: "2024-01-01", isNegative: false },
]
```

解析逻辑：
1. 按空格分割（引号内保留）
2. 识别 `!` 前缀标记否定条件
3. 识别 `field:` 格式标记字段
4. 其余标记为 `general`（全文搜索）

### 5.3 Meilisearch 查询构建

[buildMeiliQuery()](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/searchQueryBuilder.ts#L85-L91)

```typescript
// 仅将 general 字段的 value 拼接为全文搜索字符串
const generalValues = tokens.filter(t => t.field === "general").map(t => t.value);
return generalValues.join(" ");
```

### 5.4 过滤器构建

[buildMeiliFilters()](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/searchQueryBuilder.ts#L93-L207)

```typescript
// 基础权限过滤（二选一）：
- 公开：`collectionIsPublic = true`
- 私有：`(collectionOwnerId = ${userId}) OR (collectionMemberIds = ${userId})`

// 各字段转换：
- url/name/description/type/collection/tag → 等值匹配
- pinned:true → `pinnedBy = ${userId}`
- public:true → `collectionIsPublic = true`
- before/after → `creationTimestamp < / > ${timestamp}`
- 否定条件前缀加 `NOT`
```

---

## 六、公开集合搜索入口与 public 条件

### 6.1 公开搜索 API 入口

公开集合搜索不经过 `/api/v1/search`，而是使用独立的公开路由：

**API 端点**：[index.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/public/collections/links/index.ts#L1-L38)

```typescript
// /api/v1/public/collections/links?collectionId=123&searchQueryString=react
export default async function collections(req, res) {
  const convertedData: LinkRequestQuery = {
    collectionId: Number(req.query.collectionId),  // 必填
    searchQueryString: req.query.searchQueryString,
    sort: Number(req.query.sort),
    cursor: req.query.cursor ? Number(req.query.cursor) : undefined,
    pinnedOnly: req.query.pinnedOnly === "true",
  };

  // 关键：调用 searchLinks 时传入 publicOnly: true
  const { statusCode, ...data } = await searchLinks({
    query: convertedData,
    publicOnly: true,  // ← 标记为公开搜索
  });

  return res.status(statusCode).json(data);
}
```

**前端入口**：[publicLinks.tsx](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/packages/router/publicLinks.tsx#L9-L81)

```typescript
const usePublicLinks = (params) => {
  const router = useRouter();
  const queryParamsObject = {
    collectionId: router.query.id,  // 从 URL 路径获取 collectionId
    searchQueryString: router.query.q ? decodeURIComponent(router.query.q) : undefined,
    sort: params.sort,
  };
  // 请求 /api/v1/public/collections/links
};
```

**公开页面**：[index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/public/collections/[id]/index.tsx#L65-L70)

```typescript
// /public/collections/[id]?q=searchterm
const { links, data } = usePublicLinks({
  sort: sortBy,
  searchQueryString: router.query.q
    ? decodeURIComponent(router.query.q as string)
    : undefined,
});
```

### 6.2 public 条件参与查询的三层过滤

`publicOnly: true` 会在**三层**过滤中生效：

#### 第一层：Meilisearch 过滤器

[buildMeiliFilters()](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/searchQueryBuilder.ts#L102-L104)

```typescript
const filters: string[] = publicOnly
  ? ["collectionIsPublic = true"]  // ← 公开搜索：仅查公开集合
  : [`(collectionOwnerId = ${userId}) OR (collectionMemberIds = ${userId})`];
```

#### 第二层：PostgreSQL collection 条件

[searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L41-L49)

```typescript
const collectionCondition = [];
if (query.collectionId || publicOnly) {
  collectionCondition.push({
    collection: {
      id: query.collectionId,
      ...(publicOnly ? { isPublic: true } : {}),  // ← 二次校验公开状态
    },
  });
}
```

#### 第三层：搜索词中的 public: 操作符

[searchQueryBuilder.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/searchQueryBuilder.ts#L160-L168)

```typescript
case "public":
  if (value === "true") {
    filters.push(
      isNegative
        ? `NOT collectionIsPublic = true`
        : `collectionIsPublic = true`
    );
  }
  break;
```

### 6.3 私有搜索 vs 公开搜索对比

| 维度 | 私有搜索 `/api/v1/search` | 公开搜索 `/api/v1/public/collections/links` |
|------|--------------------------|--------------------------------------------|
| 调用者 | 登录用户 | 任意访客 |
| 鉴权 | `verifyUser()` 校验登录 | 无鉴权 |
| `publicOnly` | `false`（或不传） | `true` |
| Meilisearch 过滤 | `(collectionOwnerId = X) OR (collectionMemberIds = X)` | `collectionIsPublic = true` |
| `collectionId` | 可选 | 必填 |
| PostgreSQL 二次校验 | `ownerId = X OR members.some.userId = X` | `isPublic = true` |
| 搜索语法 | 支持 `public:true` 过滤 | 支持但无意义（已限定公开） |

---

## 七、Meilisearch id 回查后的排序与分页关系

### 7.1 排序关系分析

**关键结论**：Meilisearch 和 PostgreSQL 使用**相同的排序条件**，但 PostgreSQL 会**重新排序**，可能破坏 Meilisearch 的全文相关度顺序。

**代码流程** [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L54-L153)：

```typescript
// ┌─────────────────────────────────────────────────────────────────┐
// │ 第一步：Meilisearch 搜索（负责全文检索 + 分页 + 排序）          │
// └─────────────────────────────────────────────────────────────────┘
const meiliResp = await meiliClient.index("links").search(meiliQuery, {
  filter: meiliFilters,
  attributesToRetrieve: ["id"],    // 只取 id
  limit,                            // 分页大小
  offset,                           // 分页偏移
  sort:                             // 排序条件
    query.sort === Sort.DateNewestFirst ? ["id:desc"] :
    query.sort === Sort.DateOldestFirst ? ["id:asc"] :
    query.sort === Sort.NameAZ ? ["name:asc"] :
    query.sort === Sort.NameZA ? ["name:desc"] : ["id:desc"],
});

// 提取 id 列表（按 Meilisearch 排序顺序）
const meiliIds = meiliResp.hits.map((h: any) => h.id);
// 例如：[100, 95, 88, 72, 61, ...]

// ┌─────────────────────────────────────────────────────────────────┐
// │ 第二步：PostgreSQL 回查（负责权限二次校验 + 数据补充）          │
// └─────────────────────────────────────────────────────────────────┘
const links = await prisma.link.findMany({
  where: {
    id: { in: meiliIds },           // 用 Meilisearch 返回的 id 列表查询
    AND: [/* 额外过滤条件 */],
  },
  include: { tags: true, collection: true, pinnedBy: ... },
  orderBy: order,                   // ← PostgreSQL 再次排序！
});

// order 变量定义（L26-L30）：
let order: Order = { id: "desc" };
if (query.sort === Sort.DateNewestFirst) order = { id: "desc" };
else if (query.sort === Sort.DateOldestFirst) order = { id: "asc" };
else if (query.sort === Sort.NameAZ) order = { name: "asc" };
else if (query.sort === Sort.NameZA) order = { name: "desc" };
```

**排序对应表**：

| Sort 枚举 | Meilisearch sort | PostgreSQL orderBy | 一致性 |
|-----------|------------------|-------------------|--------|
| DateNewestFirst | `["id:desc"]` | `{ id: "desc" }` | 一致 |
| DateOldestFirst | `["id:asc"]` | `{ id: "asc" }` | 一致 |
| NameAZ | `["name:asc"]` | `{ name: "asc" }` | 一致 |
| NameZA | `["name:desc"]` | `{ name: "desc" }` | 一致 |

**潜在问题**：
- Meilisearch 的全文搜索相关度排序被显式 `sort` 参数覆盖
- PostgreSQL 再次 `orderBy` 确保了排序一致性，但也意味着**全文相关度不影响排序**
- 如果 PostgreSQL 二次校验过滤掉了某些 id，结果顺序仍然正确（因为重新排序了）

### 7.2 分页关系分析

**分页计算** [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L64-L65,L142)：

```typescript
const limit = paginationTakeCount;   // 默认 50
const offset = query.cursor || 0;    // 从 cursor 获取，初始为 0

// nextCursor 计算基于 Meilisearch 的结果，而非 PostgreSQL
const nextCursor = meiliResp.hits.length === limit ? offset + limit : null;
```

**前端分页** [links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/packages/router/links.tsx#L72-L103)：

```typescript
return useInfiniteQuery({
  queryKey: ["links", { params }],
  queryFn: async (params) => {
    // URL: /api/v1/search?cursor=0&sort=0&searchQueryString=...
    // 第一页 cursor=0, 第二页 cursor=50, 第三页 cursor=100...
    const url = "/api/v1/search?cursor=" + params.pageParam + "&" + params;
  },
  initialPageParam: 0,
  getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
});
```

**分页时序示例**：
```
第一页：offset=0, limit=50 → Meilisearch 返回 hits[0..49] → nextCursor=50
第二页：offset=50, limit=50 → Meilisearch 返回 hits[50..99] → nextCursor=100
第三页：offset=100, limit=50 → Meilisearch 返回 hits[100..149] → ...
```

**潜在问题**：
- 如果 PostgreSQL 二次校验过滤掉了部分结果（比如 hits 返回 50 个 id，但只有 45 个通过权限校验）
- 那么 `links` 数组只有 45 条，但 `nextCursor` 仍然是 `offset + 50`
- 这意味着**下一页会跳过 5 条本应可见的结果**（如果它们存在于后续偏移中）
- 这是一个设计缺陷，但在实践中影响较小（因为 Meilisearch 过滤器已经做了权限过滤）

### 7.3 排序分页完整性检查

**Meilisearch 层已经过滤了权限**：
- 私有搜索：`(collectionOwnerId = X) OR (collectionMemberIds = X)`
- 公开搜索：`collectionIsPublic = true`

**PostgreSQL 层二次校验是冗余的**：
- 理论上，Meilisearch 返回的 id 都应该通过 PostgreSQL 校验
- 二次校验的存在是为了**防止 Meilisearch 索引滞后**导致的权限泄露
- 例如：某链接刚被从公开改为私有，但 Meilisearch 索引还未更新

**最终结论**：
1. 排序：Meilisearch 和 PostgreSQL 使用相同排序条件，结果顺序一致
2. 分页：cursor 基于 offset 递增，由 Meilisearch 分页保证
3. 二次校验：可能减少返回数量，但不影响排序和分页逻辑

---

## 八、无 Meilisearch 时的降级方案

当 `meiliClient` 为 null 时，使用 PostgreSQL 的 `contains` 进行模糊搜索。

[searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L155-L255)

```typescript
// 搜索条件 OR 连接：
- name LIKE ?
- url LIKE ?
- description LIKE ?
- tags.name LIKE ?

// PostgreSQL 使用 mode: "insensitive"（不区分大小写）
```

注意：降级方案不支持高级搜索操作符（`url:`、`tag:` 等），仅支持简单的全文模糊匹配。

---

## 九、完整时序图

```
┌───────────────────────────────────────────────────────────────────┐
│                        用户操作链路                                  │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  创建/更新链接      ──►  indexVersion = null                      │
│  更新/删除标签      ──►  indexVersion = null                      │
│  更新/删除集合      ──►  indexVersion = null                      │
│  归档处理            ──►  indexVersion = null                      │
│  删除链接            ──►  meili.deleteDocument()                   │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                      Worker 索引循环                           │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  while (true) {                                                    │
│    查询 indexVersion != 1 OR indexVersion IS NULL              │
│    转换文档 → addDocuments() → waitForTask()                   │
│    更新 indexVersion = 1                                           │
│    sleep(10s)                                                     │
│  }                                                                  │
│                                                                     │
└───────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                      私有搜索链路                              │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  搜索框输入 → 回车 → /search?q=encoded                         │
│    │                                                               │
│    ▼                                                               │
│  useLinks() 调用 /api/v1/search                              │
│    │                                                               │
│    ├─ parseSearchTokens() 解析搜索词                            │
│    ├─ buildMeiliQuery() 构建查询字符串                            │
│    ├─ buildMeiliFilters(publicOnly=false)                        │
│    │    → 基础过滤: (collectionOwnerId = X) OR (collectionMemberIds = X) │
│    │                                                               │
│    ▼                                                               │
│  meiliClient.search() → 返回 id 列表[50个]                       │
│    │  (已按权限过滤 + 排序 + 分页)                               │
│    ▼                                                               │
│  prisma.link.findMany({ id: { in: [50个id] } })                   │
│    │  (二次权限校验 + 重新排序)                                   │
│    ▼                                                               │
│  返回 links + nextCursor(=offset+50)                              │
│    │                                                               │
│    ▼                                                               │
│  前端无限滚动渲染                                                 │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                      公开搜索链路                              │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  /public/collections/123?q=react                                │
│    │                                                               │
│    ▼                                                               │
│  usePublicLinks() 调用 /api/v1/public/collections/links          │
│    │    ?collectionId=123&searchQueryString=react                │
│    │                                                               │
│    ├─ parseSearchTokens() 解析搜索词                            │
│    ├─ buildMeiliQuery() 构建查询字符串                            │
│    ├─ buildMeiliFilters(publicOnly=true)                         │
│    │    → 基础过滤: collectionIsPublic = true                   │
│    │                                                               │
│    ▼                                                               │
│  meiliClient.search() → 返回 id 列表[50个]                       │
│    │  (已按公开过滤 + 排序 + 分页)                               │
│    ▼                                                               │
│  prisma.link.findMany({ id: { in: [50个id] } })                   │
│    │  (二次校验 isPublic=true + 重新排序)                         │
│    ▼                                                               │
│  返回 links + nextCursor(=offset+50)                              │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

---

## 十、关键设计要点

1. **索引版本机制**：通过 `indexVersion` 字段实现增量索引，避免重复索引。
2. **异步索引**：索引更新是异步的，用户操作后需要等待 Worker 循环处理（默认10秒间隔）。
3. **读写分离**：Meilisearch 负责全文检索和初筛，PostgreSQL 负责权限和数据完整性。
4. **权限双重过滤**：Meilisearch 索引时固化权限字段，查询时过滤；PostgreSQL 回查时二次校验，防止索引滞后导致泄露。
5. **公开/私有统一逻辑**：`searchLinks()` 函数通过 `publicOnly` 参数复用同一套代码。
6. **排序一致性**：Meilisearch 和 PostgreSQL 使用相同排序条件，确保结果顺序一致。
7. **分页基于偏移**：使用 `offset + limit` 而非 cursor 分页，实现简单但可能跳过被过滤的结果。
8. **优雅降级**：Meilisearch 不可用时自动降级到 PostgreSQL 模糊搜索。
9. **批量关联刷新**：标签/集合变更批量标记相关链接重新索引，保证数据一致性。
