# Linkwarden 标签（Tagging）与列表筛选关联机制分析

## 一、数据模型基础

### 1.1 Tag 模型（标签）

定义位置：[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/packages/prisma/schema.prisma#L200-L218)

```prisma
model Tag {
  id                      Int      @id @default(autoincrement())
  name                    String
  links                   Link[]
  owner                   User     @relation(fields: [ownerId], references: [id], onDelete: Cascade)
  ownerId                 Int
  // 归档设置...
  @@unique([name, ownerId])  // 关键约束：同一用户下标签名唯一
  @@index([ownerId])
}
```

**核心特性：**
- 标签属于用户（ownerId），不属于集合
- 通过 `@@unique([name, ownerId])` 保证同一用户下标签名不重复
- 与 Link 为多对多关系（Tag.links ↔ Link.tags）

### 1.2 Link 模型（链接）

定义位置：[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/packages/prisma/schema.prisma#L166-L198)

```prisma
model Link {
  id              Int         @id @default(autoincrement())
  collection      Collection  @relation(fields: [collectionId], references: [id], onDelete: Cascade)
  collectionId    Int
  tags            Tag[]       // 多对多关联
  pinnedBy        User[]      @relation("PinnedLinks")
  createdBy       User?       @relation("CreatedLinks", fields: [createdById], references: [id], onDelete: Cascade)
  // ...
}
```

**核心特性：**
- Link 必须属于一个 Collection（collectionId 必填，级联删除）
- Link 可以关联多个 Tag，Tag 也可以关联多个 Link
- Link 的可见性由其所属 Collection 的权限决定

### 1.3 Collection 模型（集合）

定义位置：[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/packages/prisma/schema.prisma#L126-L149)

```prisma
model Collection {
  id               Int                   @id @default(autoincrement())
  owner            User                  @relation(fields: [ownerId], references: [id], onDelete: Cascade)
  ownerId          Int
  parentId         Int?                   // 支持嵌套子集合
  parent           Collection?           @relation("SubCollections", fields: [parentId], references: [id], onDelete: Cascade)
  members          UsersAndCollections[]  // 协作成员
  links            Link[]
  isPublic         Boolean               @default(false)
}
```

---

## 二、Tag Scope 与 Collection Scope 的作用域机制

### 2.1 Tag Scope（标签作用域）

定义位置：[getTags.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/tags/getTags.ts)

标签查询有两种作用域模式：

#### 模式一：用户维度查询（userId）

```typescript
// 用户可见的标签 = 自己拥有的标签 + 协作集合中使用的标签
where: {
  AND: [
    ...(searchCondition ? [searchCondition] : []),
    {
      OR: [
        { ownerId: userId },  // 1. 标签属于当前用户
        ...(memberCollectionIds.length > 0
          ? [{
              links: {
                some: {
                  collectionId: { in: memberCollectionIds },  // 2. 标签被用于用户是成员的集合
                },
              },
            }]
          : []),
      ],
    },
  ],
}
```

**说明：**
- 用户不仅能看到自己创建的标签，还能看到在协作集合中已被使用的标签
- 标签的真正所有者是集合的 owner，而非使用标签的成员

#### 模式二：集合维度查询（collectionId）

```typescript
// 获取某个集合下所有链接使用过的标签
where: {
  AND: [
    ...(searchCondition ? [searchCondition] : []),
    {
      links: {
        some: { collectionId },  // 只要有链接在该集合中使用了该标签
      },
    },
  ],
}
```

### 2.2 Collection Scope（集合作用域）

集合是权限控制的核心边界，定义位置：[getLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/links/getLinks.ts#L100-L109)

```typescript
// 链接可见性 = 用户是集合所有者 OR 用户是集合成员
where: {
  collection: {
    OR: [
      { ownerId: userId },
      { members: { some: { userId } } },
    ],
  },
}
```

**权限层级：**
1. **所有者（ownerId）**：完全控制，可移动链接到其他集合
2. **成员（members）**：通过 `UsersAndCollections` 关联表，具备 `canCreate`/`canUpdate`/`canDelete` 权限
3. **公开集合（isPublic）**：无需登录即可浏览

---

## 三、Link 与 Tag 的关联机制

### 3.1 创建链接时关联标签

定义位置：[postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/links/postLink.ts#L120-L137)

```typescript
tags: {
  connectOrCreate: link.tags?.map((tag) => ({
    where: {
      name_ownerId: {
        name: tag.name.trim(),
        ownerId: linkCollection.ownerId,  // 标签 owner = 集合 owner
      },
    },
    create: {
      name: tag.name.trim(),
      owner: { connect: { id: linkCollection.ownerId } },
    },
  })),
}
```

**关键机制：**
- 使用 Prisma 的 `connectOrCreate`：如果标签已存在（按 name + ownerId）则关联，不存在则新建
- **标签的 ownerId 始终是集合的 owner**，而不是当前操作用户
- 这意味着即使是集合成员打标签，标签也归集合创建者所有

### 3.2 更新链接时关联标签

定义位置：[updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L109-L176)

```typescript
// 先去重
const uniqueTags = (() => {
  const seen = new Set<string>();
  return (data.tags ?? []).filter((t) => {
    const key = t.name;
    if (!key) return false;
    if (seen.has(key)) return false;
    seen.add(key);
    return true;
  });
})();

const tagConnectOrCreate = uniqueTags.map((tag) => ({
  where: {
    name_ownerId: {
      name: tag.name,
      ownerId: data.collection.ownerId,
    },
  },
  create: {
    name: tag.name,
    owner: { connect: { id: data.collection.ownerId } },
  },
}));

// 根据 removePreviousTags 参数决定策略
tags: removePreviousTags
  ? { set: [], connectOrCreate: tagConnectOrCreate }  // 清空后重新关联
  : { connectOrCreate: tagConnectOrCreate },            // 追加关联
```

**两种模式：**
- `removePreviousTags = false`（默认）：保留原有标签，追加新标签
- `removePreviousTags = true`：先 `set: []` 清空所有关联，再重新关联

### 3.3 前端状态管理（选中链接）

定义位置：[links.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/store/links.ts)

```typescript
type LinkStore = {
  selectedIds: Record<number, true>;  // 用对象存储 { [linkId]: true }
  toggleSelected: (id: number) => void;
  clearSelected: () => void;
  setSelected: (ids: number[]) => void;
  selectionCount: number;
};
```

---

## 四、标签删除的实现逻辑

### 4.1 单个标签删除

定义位置：[deleteTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts)

```typescript
// 1. 权限校验：只能删除自己拥有的标签
if (targetTag?.ownerId !== userId) return { response: "Permission denied.", status: 401 };

// 2. 删除标签（Prisma 自动解除多对多关联）
const deletedTag = await prisma.tag.delete({
  where: { id: tagId },
  include: { links: { select: { id: true } } },
});

// 3. 将受影响链接的 indexVersion 置为 null，触发 MeiliSearch 重新索引
const linkIds = links.map((link) => link.id);
await prisma.link.updateMany({
  where: { id: { in: linkIds } },
  data: { indexVersion: null },
});
```

**级联影响：**
- Prisma 的多对多关系会自动解除关联（无需手动清理连接表）
- 所有关联过该标签的链接会标记为需要重新索引（`indexVersion: null`）
- 前端 React Query 会同时失效 `["tags"]`、`["links"]`、`["dashboardData"]` 缓存

### 4.2 批量标签删除

定义位置：[bulkTagDelete.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/tags/bulkTagDelete.ts)

```typescript
// 1. 找出所有受影响的链接（用户所有标签相关的链接）
affectedLinks = (await prisma.link.findMany({
  where: { tags: { some: { ownerId: userId } } },
  select: { id: true },
})).map((link) => link.id);

// 2. 批量删除标签
deletedTag = (await prisma.tag.deleteMany({
  where: { ownerId: userId, id: { in: tagIds } },
})).count;

// 3. 标记所有相关链接需要重新索引
await prisma.link.updateMany({
  where: { id: { in: affectedLinks } },
  data: { indexVersion: null },
});
```

### 4.3 标签合并

定义位置：[mergeTags.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/tags/mergeTags.ts)

```typescript
// 事务处理：
await prisma.$transaction(async (tx) => {
  // 1. 删除所有旧标签
  await tx.tag.deleteMany({ where: { ownerId: userId, id: { in: tagIds } } });

  // 2. 创建新标签并关联所有受影响的链接
  const newTag = await tx.tag.create({
    data: {
      name: newTagName,
      ownerId: userId,
      links: { connect: affectedLinks.map((id) => ({ id })) },
    },
  });

  // 3. 标记重新索引
  await tx.link.updateMany({
    where: { id: { in: affectedLinks } },
    data: { indexVersion: null },
  });
});
```

---

## 五、列表筛选与过滤查询的实现

### 5.1 查询参数类型

定义位置：[global.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/packages/types/global.ts#L105-L118)

```typescript
export type LinkRequestQuery = {
  sort?: Sort;               // 排序方式
  cursor?: number;           // 分页游标
  collectionId?: number;     // 按集合筛选
  tagId?: number;            // 按标签筛选
  pinnedOnly?: boolean;      // 仅显示已置顶
  searchQueryString?: string; // 搜索关键词
};

export type TagRequestQuery = {
  sort?: TagSort;   // 排序方式
  cursor?: number;  // 分页游标
  search?: string;  // 标签名搜索
};
```

### 5.2 数据库直接查询（Fallback，无 MeiliSearch）

定义位置：[getLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/links/getLinks.ts)

```typescript
where: {
  AND: [
    // 1. 权限过滤：只能看自己的 or 协作的集合
    { collection: { OR: [{ ownerId: userId }, { members: { some: { userId } } }] } },
    // 2. 集合筛选
    ...collectionCondition,
    {
      OR: [
        // 3. 标签筛选（tagId）
        ...tagCondition,
        {
          // 4. 搜索关键词（OR 逻辑：名称/URL/描述/标签名 任一匹配）
          [query.searchQueryString ? "OR" : "AND"]: [
            pinnedCondition,
            ...searchConditions,  // name/url/description/tags.name 的 contains
          ],
        },
      ],
    },
  ],
}
```

**Tag 过滤条件构造：**
```typescript
if (query.tagId) {
  tagCondition.push({
    tags: { some: { id: query.tagId } },  // 标签 some 关联该 tagId
  });
}
```

**关键词搜索包含 Tag 名称：**
```typescript
searchConditions.push({
  tags: {
    some: {
      name: { contains: query.searchQueryString, mode: "insensitive" },
      // 同时校验标签可见性
      OR: [
        { ownerId: userId },
        { links: { some: { collection: { members: { some: { userId } } } } } },
      ],
    },
  },
});
```

### 5.3 MeiliSearch 高级搜索（推荐路径）

定义位置：[searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts)

#### 搜索语法解析器

定义位置：[searchQueryBuilder.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/searchQueryBuilder.ts)

**支持的搜索条件字段：**
```typescript
const SEARCH_CONDITIONS = [
  "url", "name", "description", "type",
  "collection", "pinned", "public",
  "before", "after", "tag",
];
```

**高级语法示例：**
- `tag:javascript` → 筛选标签为 javascript 的链接
- `!tag:archived` → 排除标签为 archived 的链接
- `collection:Work` → 筛选集合名为 Work 的链接
- `pinned:true` → 仅显示已置顶
- `after:2024-01-01 before:2024-12-31` → 日期范围
- 普通文本 → 在 name/url/description 中全文搜索

**Token 解析流程：**
```typescript
// 解析示例："react tag:frontend !tag:deprecated"
// 结果：
[
  { field: "general", value: "react", isNegative: false },
  { field: "tag", value: "frontend", isNegative: false },
  { field: "tag", value: "deprecated", isNegative: true },
]
```

#### MeiliSearch 过滤器构造

```typescript
case "tag":
  filters.push(
    isNegative
      ? `NOT tags = "${escapeForMeilisearch(value)}"`
      : `tags = "${escapeForMeilisearch(value)}"`
  );
  break;

case "collection":
  filters.push(
    isNegative
      ? `NOT collectionName = "${escapeForMeilisearch(value)}"`
      : `collectionName = "${escapeForMeilisearch(value)}"`
  );
  break;
```

**执行路径：**
1. MeiliSearch 按关键词 + 过滤器返回匹配的 linkId 列表
2. 用这些 id 到 PostgreSQL 做二次查询，补充权限校验和完整数据
3. 返回分页后的结果

### 5.4 前端搜索联动

定义位置：[SearchBar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/components/SearchBar.tsx)

用户按 Enter 后跳转到 `/search?q=xxx`，该页面通过 `useLinks` hook 发起请求：

定义位置：[search.tsx](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/pages/search.tsx)

```typescript
const { links, data } = useLinks({
  sort: sortBy,
  searchQueryString: decodeURIComponent(router.query.q as string),
});
```

前端实际请求的是 `/api/v1/search`（而不是 `/api/v1/links`），由 [useFetchLinks](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/packages/router/links.tsx#L62-L104) 内部决定：

```typescript
// 默认走 search 路由（支持 MeiliSearch）
const url = "/api/v1/search?cursor=" + params.pageParam + "&" + params;
```

---

## 六、批量打标与搜索联动

### 6.1 批量打标前端流程

定义位置：[BulkEditLinksModal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/components/ModalContent/BulkEditLinksModal.tsx)

```typescript
// 提交数据结构
{
  links: [{ id: 1 }, { id: 2 }, ...],  // 选中的链接 ID
  newData: {
    tags: [{ name: "tag1" }, { name: "tag2" }],
    collectionId: 5,  // 可选：同时移动集合
  },
  removePreviousTags: boolean,  // 是否清空原有标签
}
```

### 6.2 批量打标后端实现

定义位置：[updateLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/links/bulk/updateLinks.ts)

```typescript
// 循环调用单链接更新函数
for (const l of links) {
  const updatedLink = await updateLinkById(
    userId,
    link.id as number,
    updatedData,
    removePreviousTags  // 透传参数
  );
}
```

**注意事项：**
- 批量打标实际是串行调用单链接更新，非事务批量操作
- 每个链接更新都会设置 `indexVersion: null`，触发重新索引
- 更新成功后前端失效 `["links"]`、`["tags"]`、`["dashboardData"]` 等查询缓存

### 6.3 搜索联动影响

#### 影响索引的操作

以下操作均会将 `indexVersion` 置为 null，触发后台 Worker 重新同步到 MeiliSearch：

| 操作 | 代码位置 | 触发条件 |
|------|---------|---------|
| 更新链接 | [updateLinkById.ts#L163](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L163) | 每次更新 |
| 删除标签 | [deleteTagById.ts#L36-L45](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts#L36-L45) | 所有关联该标签的链接 |
| 批量删除标签 | [bulkTagDelete.ts#L53-L62](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/tags/bulkTagDelete.ts#L53-L62) | 所有相关链接 |
| 合并标签 | [mergeTags.ts#L64-L73](file:///d:/fz/0601/solo-dogfeeding/code/86-linkwarden/apps/web/lib/api/controllers/tags/mergeTags.ts#L64-L73) | 所有受影响链接 |

#### 搜索结果一致性

由于重新索引是异步的（由 Worker 后台处理），存在短暂的不一致窗口：

```
用户批量打标 → DB 立即更新 → indexVersion=null → Worker 后台同步 → MeiliSearch 更新
                  ↓
            前端立即失效缓存 → 重新请求
                                  ↓
                        如果走 MeiliSearch：可能返回旧数据（直到索引完成）
                        如果走 DB Fallback：返回最新数据
```

---

## 七、核心交互流程图

```
前端页面 (Dashboard/Search/Collection/Tag)
        │
        ▼
React Query Hooks (useLinks / useTags)
        │  构建 queryKey: ["links", { params }] / ["tags", ...]
        ▼
API Routes:
  /api/v1/search          → searchLinks.ts (优先走 MeiliSearch)
  /api/v1/links  [GET]    → getLinks.ts (DB Fallback)
  /api/v1/links  [PUT]    → updateLinks.ts (批量) → updateLinkById.ts
  /api/v1/tags   [GET]    → getTags.ts
  /api/v1/tags   [DELETE] → bulkTagDelete.ts / deleteTagById.ts
  /api/v1/tags/merge      → mergeTags.ts
        │
        ▼
Prisma ORM → PostgreSQL
        │
        ├── Tag ←──(多对多)──→ Link ──(多对一)──→ Collection
        │                                                  │
        │                                                  └── Owner (User)
        │                                                  └── Members (UsersAndCollections)
        ▼
  indexVersion=null 标记
        │
        ▼
Worker 后台进程 → MeiliSearch 索引同步
```

---

## 八、关键设计要点总结

1. **标签所有权归集合所有者**：成员打标签时，标签 owner 是集合创建者而非成员本人，通过 `name_ownerId` 唯一约束去重

2. **双路径搜索**：MeiliSearch 提供高级搜索（支持 `tag:`、`collection:` 等语法），PostgreSQL 提供 Fallback

3. **标签删除自动清理关联**：Prisma 多对多级联 + `indexVersion=null` 触发重索引，无需手动维护连接表

4. **批量操作 = 串行单操作**：批量打标、批量删除均为循环调用单体函数，简单但非原子事务

5. **搜索结果最终一致**：DB 更新后立即失效前端缓存，但 MeiliSearch 索引有延迟，短时间内可能返回旧数据
