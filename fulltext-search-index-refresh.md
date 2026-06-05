# 全文搜索索引写入、刷新与查询流程

本文档梳理 Linkwarden 项目中全文搜索功能的完整数据流：索引写入时机、刷新机制、查询拼装与结果返回流程。

---

## 一、技术栈概览

项目使用 **Meilisearch** 作为全文搜索引擎，**PostgreSQL** 作为主数据库。

| 组件 | 文件 | 说明 |
|------|------|------|
| Meilisearch 客户端 | [meilisearchClient.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/packages/lib/meilisearchClient.ts) | 根据环境变量 `MEILI_MASTER_KEY` 决定是否启用 |
| 索引版本常量 | [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/packages/lib/constants.ts) | `MEILI_INDEX_VERSION = 1 |
| 索引 Worker | [linkIndexing.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/worker/workers/linkIndexing.ts) | 后台常驻，批量索引处理 |
| 查询构建器 | [searchQueryBuilder.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/searchQueryBuilder.ts) | 解析搜索词、构建过滤器 |
| 搜索控制器 | [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts) | 核心搜索逻辑 |
| 搜索 API | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/search/index.ts) | `/api/v1/search` 端点 |
| 前端 Hook | [links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/packages/router/links.tsx) | `useLinks()` 数据获取 |

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
// 1. 数据转换
const docs = links.map((link) => ({
  ...link,
  collectionOwnerId: link.collection.ownerId,
  collectionMemberIds: link.collection.members.map((m) => m.userId),
  collectionIsPublic: link.collection.isPublic,
  collectionName: link.collection.name,
  tags: link.tags.map((t) => t.name),
  pinnedBy: link.pinnedBy.map((p) => p.id),
  creationTimestamp: Date.parse(link.createdAt.toISOString()) / 1000,
  indexVersion: MEILI_INDEX_VERSION,
}));

// 2. 写入 Meilisearch
const task = await meiliClient.index("links").addDocuments(docs);
await meiliClient.index("links").waitForTask(task.taskUid);

// 3. 更新数据库 indexVersion 标记
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
| 归档处理 | [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/worker/lib/archiveHandler.ts) | [L57](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/worker/lib/archiveHandler.ts#L57-L57)、[L222](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/worker/lib/archiveHandler.ts#L222-L222) |
| 手动归档 | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/links/archive/index.ts) | [L82](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/links/archive/index.ts#L82-L82) |
| 单链接归档 | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/links/[id]/archive/index.ts) | [L62](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/links/[id]/archive/index.ts#L62-L62) |
| 归档上传 | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/archives/index.ts) | [L224](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/archives/index.ts#L224-L224) |
| Worker 归档 | [preservation.tsx](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx) | [L59](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L59-L59)、[L158](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L158-L158) |

### 3.2 标签相关操作

| 操作 | 文件 | 位置 |
|------|------|------|
| 更新标签 | [updateTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/tagId/updateTagById.ts) | [L82](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/tagId/updateTagById.ts#L82-L82) |
| 删除标签 | [deleteTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts) | [L43](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts#L43-L43) |
| 合并标签 | [mergeTags.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/mergeTags.ts) | [L71](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/mergeTags.ts#L71-L71) |
| 批量删标签 | [bulkTagDelete.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/bulkTagDelete.ts) | [L60](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/tags/bulkTagDelete.ts#L60-L60) |

标签更新/删除时会批量更新相关链接的 `indexVersion: null`

```typescript
// 例如 updateTagById.ts L75-L84
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

[SEARCH_CONDITIONS 定义了高级搜索操作符：

| 操作符 | 说明 |
|--------|------|
| `url:` | 匹配 URL |
| `name:` | 匹配名称 |
| `description:` | 匹配描述 |
| `type:` | 匹配类型 |
| `collection:` | 匹配集合名 |
| `pinned:` | 是否置顶 |
| `before:` | 创建日期之前 |
| `after:` | 创建日期之后 |
| `tag:` | 匹配标签 |
| `!field:` | 否定条件 |

### 5.2 搜索词解析

[parseSearchTokens()](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/searchQueryBuilder.ts#L20-L83)

```typescript
// 输入示例: "react url:github.com !tag:archive after:2024-01-01
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
- 公开：`collectionIsPublic = true
- 私有：`(collectionOwnerId = ${userId}) OR (collectionMemberIds = ${userId})`

// 各字段转换：
- url/name/description/type/collection/tag → 等值匹配
- pinned:true → `pinnedBy = ${userId}
- before/after → `creationTimestamp < / > ${timestamp}
- 否定条件前缀加 `NOT`
```

---

## 六、结果返回流程

### 6.1 搜索 API 调用链

```
前端 useLinks()
       ↓
/api/v1/search [index.ts]
       ↓
searchLinks() [searchLinks.ts]
       ↓
parseSearchTokens() → buildMeiliQuery() → buildMeiliFilters()
       ↓
meiliClient.index("links").search()
       ↓
获取 id 列表 → prisma.link.findMany()
       ↓
返回 links + nextCursor
```

### 6.2 前端数据获取

[useFetchLinks()](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/packages/router/links.tsx#L62-L104)

使用 `@tanstack/react-query` 的 `useInfiniteQuery`：

```typescript
// 请求 URL: /api/v1/search?cursor=0&sort=0&searchQueryString=...
// 返回: { links: [...], nextCursor: 50 }
// 分页: initialPageParam = 0, getNextPageParam = lastPage.nextCursor
```

### 6.3 后端搜索执行

[searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L54-L153)

```typescript
// 1. Meilisearch 搜索（仅返回 id）
const meiliResp = await meiliClient.index("links").search(meiliQuery, {
  filter: meiliFilters,
  attributesToRetrieve: ["id"],
  limit,
  offset,
  sort: [...],
});

// 2. 用 id 回查 PostgreSQL 获取完整数据
const links = await prisma.link.findMany({
  where: { id: { in: meiliIds },
  // 额外权限校验 + 过滤条件
  include: { tags: true, collection: true, pinnedBy: ... },
});

// 3. 返回分页结果
return {
  data: { links, nextCursor: offset + limit },
  statusCode: 200,
  success: true,
};
```

**关键设计**：Meilisearch 仅做全文搜索和排序，PostgreSQL 做权限二次校验和数据补充。

### 6.4 搜索排序映射

| Sort 枚举 | Meilisearch sort | PostgreSQL orderBy |
|-------------|------------------|-------------------|
| DateNewestFirst | `["id:desc"]` | `{ id: "desc" }` |
| DateOldestFirst | `["id:asc"]` | `{ id: "asc" }` |
| NameAZ | `["name:asc"]` | `{ name: "asc" }` |
| NameZA | `["name:desc"]` | `{ name: "desc" }` |

---

## 七、无 Meilisearch 时的降级方案

当 `meiliClient` 为 null 时，使用 PostgreSQL 的 `contains` 进行模糊搜索。

[searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/38-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L156-L255)

```typescript
// 搜索条件 OR 连接：
- name LIKE ?
- url LIKE ?
- description LIKE ?
- description LIKE ?
- tags.name LIKE ?

// PostgreSQL 使用 mode: "insensitive"（不区分大小写
```

注意：降级方案不支持高级搜索操作符（`url:`、`tag:` 等），仅支持简单的全文模糊匹配。

---

## 八、完整时序图

```
┌───────────────────────────────────────────────────────────────────┐
│                        用户操作链路                                  │
├───────────────────────────────────────────────────────────────────┤
│                                                               │
│  创建/更新/删除链接  ──►  indexVersion = null           │
│  更新/删除标签      ──►  indexVersion = null           │
│  更新/删除集合      ──►  indexVersion = null           │
│  归档处理            ──►  indexVersion = null           │
│  删除链接            ──►  meili.deleteDocument()         │
│                                                               │
└───────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                      Worker 索引循环                           │
├───────────────────────────────────────────────────────────────────┤
│                                                               │
│  while (true) {                                                │
│    查询 indexVersion != 1 OR indexVersion IS NULL              │
│    转换文档 → addDocuments() → waitForTask()               │
│    更新 indexVersion = 1                                       │
│    sleep(10s)                                                 │
│  }                                                              │
│                                                                 │
└───────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                      用户搜索链路                              │
├───────────────────────────────────────────────────────────────────┤
│                                                                 │
│  搜索框输入 → 回车 → /search?q=encoded                     │
│    │                                                           │
│    ▼                                                           │
│  useLinks() 调用 /api/v1/search                          │
│    │                                                           │
│    ├─ parseSearchTokens() 解析搜索词                        │
│    ├─ buildMeiliQuery() 构建查询字符串                        │
│    ├─ buildMeiliFilters() 构建过滤器                          │
│    │                                                           │
│    ▼                                                           │
│  meiliClient.search() → 返回 id 列表                          │
│    │                                                           │
│    ▼                                                           │
│  prisma.link.findMany({ id: { in: [...] } })                 │
│    │  (二次权限校验 + 数据补充)                               │
│    ▼                                                           │
│  返回 links + nextCursor                                       │
│    │                                                           │
│    ▼                                                           │
│  前端无限滚动渲染                                             │
│                                                                 │
└───────────────────────────────────────────────────────────────────┘
```

---

## 九、关键设计要点

1. **索引版本机制**：通过 `indexVersion` 字段实现增量索引，避免重复索引。
2. **异步索引**：索引更新是异步的，用户操作后需要等待 Worker 循环处理。
3. **读写分离**：Meilisearch 负责全文检索，PostgreSQL 负责权限和数据完整性。
4. **权限二次校验**：Meilisearch 结果需回查 PostgreSQL 确保数据安全。
5. **优雅降级**：Meilisearch 不可用时自动降级到 PostgreSQL 模糊搜索。
6. **批量更新关联刷新**：标签/集合变更批量标记相关链接重新索引。
