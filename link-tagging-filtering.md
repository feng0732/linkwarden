# Linkwarden Tagging 与列表筛选关联逻辑深度分析

## 一、数据模型与核心关联

### 1.1 数据库模型

**Tag 模型** `packages/prisma/schema.prisma#L200-L218`

```prisma
model Tag {
  id        Int      @id @default(autoincrement())
  name      String
  links     Link[]
  owner     User     @relation(fields: [ownerId], references: [id], onDelete: Cascade)
  ownerId   Int
  @@unique([name, ownerId])   // 同一用户下标签名唯一
  @@index([ownerId])
}
```

**Link 模型** `packages/prisma/schema.prisma#L166-L198`

```prisma
model Link {
  id           Int         @id @default(autoincrement())
  collection   Collection  @relation(fields: [collectionId], references: [id], onDelete: Cascade)
  collectionId Int
  tags         Tag[]        // 多对多：Link ↔ Tag
  pinnedBy     User[]       @relation("PinnedLinks")
  indexVersion Int?         // MeiliSearch 索引版本标记
}
```

**Collection 模型** `packages/prisma/schema.prisma#L126-L149`

```prisma
model Collection {
  id        Int                   @id @default(autoincrement())
  owner     User                  @relation(fields: [ownerId], references: [id], onDelete: Cascade)
  ownerId   Int
  members   UsersAndCollections[]  // 协作成员
  links     Link[]
  isPublic  Boolean               @default(false)
}
```

### 1.2 Link-Tag 关联机制

**创建链接时** `apps/web/lib/api/controllers/links/postLink.ts#L120-L137`

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

**关键点：** 标签的 ownerId 始终是集合的所有者，而非当前操作用户。通过 `connectOrCreate` + `name_ownerId` 唯一约束实现自动去重。

**更新链接时** `apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L169-L176`

```typescript
tags: removePreviousTags
  ? { set: [], connectOrCreate: tagConnectOrCreate }  // 清空后重设
  : { connectOrCreate: tagConnectOrCreate },            // 追加
```

---

## 二、Tag Scope 与 Collection Scope

### 2.1 Tag Scope（标签可见性范围）

**实现位置** `apps/web/lib/api/controllers/tags/getTags.ts`

两种查询模式：

#### 模式 A：按用户维度查询（userId）

```typescript
where: {
  AND: [
    ...(searchCondition ? [searchCondition] : []),
    {
      OR: [
        { ownerId: userId },  // ① 用户自己拥有的标签
        ...(memberCollectionIds.length > 0
          ? [{
              links: {
                some: { collectionId: { in: memberCollectionIds } },  // ② 协作集合中已使用的标签
              },
            }]
          : []),
      ],
    },
  ],
}
```

用户可见标签 = 用户拥有的标签 ∪ 协作集合中已被使用的标签

#### 模式 B：按集合维度查询（collectionId）

```typescript
where: {
  AND: [
    ...(searchCondition ? [searchCondition] : []),
    { links: { some: { collectionId } } },  // 该集合中任一链接使用过的标签
  ],
}
```

### 2.2 Collection Scope（链接可见性范围）

链接可见性由其所属 Collection 的权限决定，所有查询均以此为第一层过滤：

```typescript
// 权限 AND 条件
{
  collection: {
    OR: [
      { ownerId: userId },           // ① 集合所有者
      { members: { some: { userId } } },  // ② 协作成员
    ],
  },
}
```

---

## 三、多条件筛选的 AND/OR 嵌套逻辑（核心）

本节重点分析当 `tagId`、`searchQueryString`、`collectionId`、`pinnedOnly` **同时存在**时，两个接口的查询树差异。

### 3.1 查询参数类型

`packages/types/global.ts#L105-L112`

```typescript
export type LinkRequestQuery = {
  sort?: Sort;
  cursor?: number;
  collectionId?: number;      // 按集合筛选
  tagId?: number;             // 按标签筛选
  pinnedOnly?: boolean;       // 仅显示置顶
  searchQueryString?: string; // 搜索关键词/高级语法
};
```

---

### 3.2 旧列表接口（getLinks）的完整查询树

**实现位置** `apps/web/lib/api/controllers/links/getLinks.ts#L97-L127`

```typescript
where: {
  AND: [
    // ───────── 第一层：权限过滤（必须全部满足） ─────────
    {
      collection: {
        OR: [
          { ownerId: userId },
          { members: { some: { userId } } },
        ],
      },
    },

    // ───────── 第二层：collectionId 过滤（必须满足） ─────────
    ...collectionCondition,  // [{ collection: { id: query.collectionId } }]

    // ───────── 第三层：tagId 与 (pinnedOnly + search) 的 OR 关系 ─────────
    {
      OR: [
        // 分支 A：tagId 匹配
        ...tagCondition,  // [{ tags: { some: { id: query.tagId } } }]

        // 分支 B：pinnedOnly 与 searchConditions 的组合
        {
          // 🔑 关键点：有 searchQueryString 时用 OR，无时用 AND
          [query.searchQueryString ? "OR" : "AND"]: [
            {
              pinnedBy: query.pinnedOnly
                ? { some: { id: userId } }
                : undefined,  // undefined 时 Prisma 忽略该条件
            },
            ...searchConditions,  // name/url/description/tags.name contains
          ],
        },
      ],
    },
  ],
}
```

**searchConditions 的构成**（当有 searchQueryString 时）：

```typescript
searchConditions = [
  { name: { contains: query.searchQueryString, mode: "insensitive" } },
  { url: { contains: query.searchQueryString, mode: "insensitive" } },
  { description: { contains: query.searchQueryString, mode: "insensitive" } },
  {
    tags: {
      some: {
        name: { contains: query.searchQueryString, mode: "insensitive" },
        // 🔑 额外的标签可见性校验
        OR: [
          { ownerId: userId },
          { links: { some: { collection: { members: { some: { userId } } } } } },
        ],
      },
    },
  },
];
```

---

#### 用布尔逻辑表达式表示（四条件同时存在时）

当 `tagId`、`searchQueryString`、`collectionId`、`pinnedOnly=true` **全部传入**时：

```
权限过滤
  AND collectionId = X
  AND (
    tagId = Y                          ← 分支 A：只要匹配标签
    OR (
      pinnedBy 包含 userId             ← 分支 B-1：已置顶
      OR name 包含关键词                ← 分支 B-2：
      OR url   包含关键词                ← 分支 B-3：
      OR description 包含关键词          ← 分支 B-4：
      OR (tags.name 包含关键词 AND 标签可见)  ← 分支 B-5
    )
  )
```

**语义：** 链接必须属于指定集合，且满足以下条件之一：
- 被打了指定标签（tagId），**不管是否置顶、不管名称是否匹配**
- **或者**（已置顶 OR 名称匹配 OR URL 匹配 OR 描述匹配 OR 标签名匹配）

**注意：tagId 与 pinnedOnly/search 是 OR 关系，不是 AND 关系！**

这意味着当用户同时指定 tagId 和 pinnedOnly=true 时，结果包含：
1. 所有打了该标签的链接（不管是否置顶）
2. 加上所有置顶且匹配搜索的链接（不管是否有该标签）

---

### 3.3 搜索接口（searchLinks）的查询树

搜索接口有两条路径，取决于 MeiliSearch 是否可用。

---

#### 路径 A：MeiliSearch 可用 + 有 searchQueryString

**实现位置** `apps/web/lib/api/controllers/search/searchLinks.ts#L54-L152`

```
┌─────────────────────────────────────────────────────┐
│  阶段 1：MeiliSearch 检索（先粗筛）                    │
│  输入：                                              │
│    meiliQuery   = searchQueryString 中的 general 词  │
│    meiliFilters = [                                 │
│      ① (collectionOwnerId=userId OR collectionMemberIds=userId) │
│      ② 搜索语法解析出的 tag:/collection:/pinned: 等过滤器        │
│    ]                                                │
│  输出：匹配的 linkId 数组（meiliIds）                 │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│  阶段 2：PostgreSQL 二次过滤（精筛 + 补全数据）        │
│  where: {                                           │
│    id: { in: meiliIds },   ← 限定在 MeiliSearch 结果内│
│    AND: [                                            │
│      ① 权限过滤（同前）                               │
│      ② collectionId 条件                             │
│      {                                               │
│        OR: [                                         │
│          ...tagCondition,       ← tagId 匹配         │
│          { ...pinnedCondition }, ← pinnedOnly 匹配   │
│        ],                                            │
│      },                                              │
│    ],                                                │
│  }                                                   │
└─────────────────────────────────────────────────────┘
```

**🔑 阶段 2 的关键不同：** 在 MeiliSearch 路径中，PostgreSQL 二次过滤 **不再重复 searchConditions**（因为 MeiliSearch 已经做了全文检索），只剩 tagId 和 pinnedOnly 的 OR：

```typescript
// searchLinks.ts 第 116-123 行
{
  OR: [
    ...tagCondition,        // tagId 匹配
    { ...pinnedCondition }, // pinnedOnly 匹配
    // ⚠️ 注意：此处没有 searchConditions！
  ],
}
```

**MeiliSearch 过滤器构造** `apps/web/lib/api/searchQueryBuilder.ts#L93-L207`

```typescript
// 基础权限过滤器
const filters: string[] = publicOnly
  ? ["collectionIsPublic = true"]
  : [`(collectionOwnerId = ${userId}) OR (collectionMemberIds = ${userId})`];

// 按搜索语法追加过滤器
for (const { field, value, isNegative } of tokens) {
  switch (field) {
    case "tag":
      filters.push(isNegative
        ? `NOT tags = "${value}"`
        : `tags = "${value}"`);
      break;
    case "collection":
      filters.push(isNegative
        ? `NOT collectionName = "${value}"`
        : `collectionName = "${value}"`);
      break;
    case "pinned":
      filters.push(value === "true"
        ? (isNegative ? `NOT pinnedBy = ${userId}` : `pinnedBy = ${userId}`)
        : (isNegative ? `pinnedBy = ${userId}` : `NOT pinnedBy = ${userId}`));
      break;
    // ...
  }
}
```

所有过滤器之间是 **AND 关系**（MeiliSearch 中 filter 数组默认 AND）。

---

#### 路径 B：MeiliSearch 不可用 或 无 searchQueryString（DB Fallback）

**实现位置** `apps/web/lib/api/controllers/search/searchLinks.ts#L155-L255`

此路径的逻辑与旧列表接口 **几乎完全一致**，仅有一处细微差异：

```typescript
// searchLinks Fallback 的 searchConditions（第 180-189 行）
searchConditions.push({
  tags: {
    some: {
      name: { contains: query.searchQueryString, mode: "insensitive" },
      // ⚠️ 注意：此处没有旧接口中的 OR 标签可见性校验！
    },
  },
});
```

除此之外，整体 AND/OR 结构与旧接口相同。

---

### 3.4 三接口四条件叠加对比表

当 **tagId=T、searchQueryString=S、collectionId=C、pinnedOnly=true** 同时传入时：

| 条件 | 旧接口 getLinks | 搜索接口 MeiliSearch 路径 | 搜索接口 Fallback |
|------|----------------|--------------------------|-------------------|
| 权限过滤 | ✅ AND | ✅ Meili阶段 AND + PG阶段 AND | ✅ AND |
| collectionId=C | ✅ AND | ✅ Meili阶段(语法解析) + PG阶段 AND | ✅ AND |
| tagId=T | ✅ OR 分支A | ✅ PG阶段 OR 分支A | ✅ OR 分支A |
| pinnedOnly=true | ✅ OR 分支B-1 | ✅ PG阶段 OR 分支B | ✅ OR 分支B-1 |
| S 匹配 name/url/description | ✅ OR 分支B-2~4 | ✅ Meili 全文检索 | ✅ OR 分支B-2~4 |
| S 匹配 tags.name（含可见性校验） | ✅ OR 分支B-5 | ✅ Meili tags 字段 + 语法 `tag:` | ✅ OR 分支B-5（无可见性校验） |
| 高级语法 `tag:/collection:/pinned:` | ❌ 不支持 | ✅ Meili filter 解析 | ❌ 不支持 |
| 搜索语法中的否定（`!tag:`） | ❌ 不支持 | ✅ Meili NOT 过滤 | ❌ 不支持 |

---

## 四、多条件同时存在时的完整查询树可视化

### 4.1 场景：全部四个条件同时传入

请求参数示例：
```
GET /api/v1/search?collectionId=5&tagId=12&pinnedOnly=true&searchQueryString=react
```

---

#### 旧列表接口 getLinks 的查询树

```
AND
├── 权限：collection.ownerId=userId OR collection.members 包含 userId
├── collection.id = 5
└── OR
    ├── 【分支A】tags.some: id = 12
    │       （只要打了 tag#12，其他条件都不看）
    │
    └── 【分支B】OR
        ├── pinnedBy.some: id = userId
        ├── name ILIKE "%react%"
        ├── url ILIKE "%react%"
        ├── description ILIKE "%react%"
        └── tags.some: (
              name ILIKE "%react%"
              AND (ownerId=userId OR links.collection.members 包含 userId)
            )
```

**结果语义 =**
（属于集合 5） AND （被打了 tag#12 **OR** （置顶 **OR** 名称/URL/描述/标签名 包含 react））

---

#### 搜索接口 MeiliSearch 路径的查询树

```
【阶段 1：MeiliSearch】
全文检索 query = "react"（general tokens）
AND filter:
  ├── (collectionOwnerId=userId OR collectionMemberIds=userId)
  └── （如有搜索语法 tag:/collection:/pinned: 也在此过滤）
        ↓ 输出 meiliIds

【阶段 2：PostgreSQL】
AND
├── id IN (meiliIds)
├── 权限：同上
├── collection.id = 5
└── OR
    ├── tags.some: id = 12       ← 分支A
    └── pinnedBy.some: id = userId ← 分支B
```

**结果语义 =**
（MeiliSearch 搜索 "react" 的结果） AND （属于集合 5） AND （被打了 tag#12 **OR** 置顶）

**⚠️ 关键差异：** MeiliSearch 路径中，searchQueryString 只在阶段 1 的 Meili 全文检索生效，阶段 2 的 PG 二次过滤中 tagId 和 pinnedOnly 是 OR 关系，且不再检查名称/URL/描述匹配。

---

## 五、MeiliSearch 与 DB Fallback 的结果差异分析

### 5.1 差异总结表

| 维度 | MeiliSearch 路径 | DB Fallback（getLinks / searchLinks Fallback） |
|------|-----------------|----------------------------------------------|
| **触发条件** | meiliClient 存在 **且** searchQueryString 非空 | meiliClient 不存在 **或** searchQueryString 为空 |
| **搜索能力** | 全文检索 + 排名打分 + 高级语法 | 简单 `contains` 子串匹配，无排名 |
| **高级语法** | `tag:` `collection:` `pinned:` `before:` `after:` `!否定` | 不支持 |
| **tagId 与 search** | tagId 在 PG 二次过滤中 OR pinnedOnly | tagId OR (pinnedOnly OR search匹配) |
| **pinnedOnly 与 search** | pinnedOnly 在 PG 中 OR tagId | pinnedOnly OR search匹配（有 search 时） |
| **标签名匹配可见性** | Meili 索引时已过滤，查询时不校验 | getLinks 有校验，search Fallback 无校验 |
| **排序** | Meili 按相关性 + 指定字段排序 | PG 按指定字段排序 |
| **分页方式** | offset-based（数字偏移） | cursor-based（ID 游标） |
| **一致性** | 最终一致（依赖 indexVersion 异步同步） | 强一致（直接查 DB） |
| **性能** | 大数据量下更快 | 全表 contains，大数据量下慢 |

### 5.2 可能导致结果不一致的场景

#### 场景 1：tagId 存在 + searchQueryString 存在 + pinnedOnly=true

- **Fallback：** 命中 tagId 的链接（不管置顶与否）都会被返回，加上命中搜索的置顶链接
- **MeiliSearch：** 链接必须先通过 MeiliSearch 全文检索（匹配 search），然后在 PG 二次过滤中满足 tagId OR pinnedOnly

**差异：** 如果某链接打了 tag#12 但名称/URL/描述/标签名都不含 "react"：
- Fallback 会返回它（因为走了 tagId 分支 A）
- MeiliSearch 路径不会返回它（因为阶段 1 MeiliSearch 全文检索就没匹配到）

#### 场景 2：搜索词恰好匹配了某个标签名

- **Fallback：** searchConditions 中 tags.name 的 contains 会匹配，作为 OR 分支的一部分
- **MeiliSearch：** tags 字段已被索引，全文检索会匹配；如果用了 `tag:xxx` 语法则会走精确过滤

#### 场景 3：数据刚更新，MeiliSearch 尚未同步

- **Fallback：** 返回最新数据（DB 实时查询）
- **MeiliSearch：** 可能返回旧数据或漏掉新数据（直到 Worker 处理 indexVersion=null 的记录）

---

## 六、标签删除与搜索联动

### 6.1 单个标签删除

`apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts`

```typescript
// 1. 权限校验
if (targetTag?.ownerId !== userId) return 401;

// 2. 删除标签（Prisma 自动解除多对多关联）
const deletedTag = await prisma.tag.delete({
  where: { id: tagId },
  include: { links: { select: { id: true } } },
});

// 3. 🔑 将所有关联链接的 indexVersion 置 null，触发重索引
await prisma.link.updateMany({
  where: { id: { in: linkIds } },
  data: { indexVersion: null },
});
```

### 6.2 批量标签删除

`apps/web/lib/api/controllers/tags/bulkTagDelete.ts`

```typescript
// 1. 先找出所有受影响的链接（用户所有标签相关的链接）
affectedLinks = (await prisma.link.findMany({
  where: { tags: { some: { ownerId: userId } } },
  select: { id: true },
})).map(l => l.id);

// 2. 批量删除标签
await prisma.tag.deleteMany({ where: { ownerId: userId, id: { in: tagIds } } });

// 3. 标记所有相关链接重索引
await prisma.link.updateMany({
  where: { id: { in: affectedLinks } },
  data: { indexVersion: null },
});
```

### 6.3 标签合并

`apps/web/lib/api/controllers/tags/mergeTags.ts`

事务中三步操作：
1. 删除所有旧标签
2. 创建新标签并关联所有受影响链接
3. 所有受影响链接 `indexVersion = null` 触发重索引

---

## 七、批量打标与搜索联动

### 7.1 前端批量编辑流程

`apps/web/components/ModalContent/BulkEditLinksModal.tsx`

```typescript
// 提交数据
{
  links: [{ id: 1 }, { id: 2 }, ...],
  newData: {
    tags: [{ name: "tag1" }, { name: "tag2" }],
    collectionId: 5,          // 可选：同时移动集合
  },
  removePreviousTags: boolean, // 是否清空原有标签
}
```

### 7.2 后端批量打标实现

`apps/web/lib/api/controllers/links/bulk/updateLinks.ts`

```typescript
// 串行调用单链接更新
for (const l of links) {
  await updateLinkById(userId, link.id, updatedData, removePreviousTags);
}
```

每次 `updateLinkById` 都会将该链接的 `indexVersion` 置为 null。

### 7.3 触发重索引的操作汇总

| 操作 | 代码位置 | 影响范围 |
|------|---------|---------|
| 更新链接（含打标） | `apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L163` | 单条链接 |
| 删除单个标签 | `apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts#L36-L45` | 所有关联该标签的链接 |
| 批量删除标签 | `apps/web/lib/api/controllers/tags/bulkTagDelete.ts#L53-L62` | 用户所有标签相关的链接 |
| 合并标签 | `apps/web/lib/api/controllers/tags/mergeTags.ts#L64-L73` | 所有受影响的链接 |

### 7.4 最终一致性时序

```
用户批量打标
    │
    ▼
PostgreSQL 更新 + indexVersion=null   ── 立即生效
    │
    ├─► 前端 React Query 失效缓存 → 重新请求
    │       │
    │       ├─► Fallback 路径：查 DB → 返回最新数据 ✅
    │       └─► MeiliSearch 路径：查 Meili → 可能返回旧数据 ⚠️
    │
    ▼
Worker 后台扫描 indexVersion=null 的链接
    │
    ▼
同步到 MeiliSearch 索引
    │
    ▼
后续 MeiliSearch 查询返回最新数据
```

---

## 八、关键设计要点

1. **tagId 与 (pinned + search) 是 OR 关系，不是 AND**：这是最容易误解的设计。指定 tagId 后，即使 pinnedOnly=true，未置顶但打了该标签的链接也会被返回。

2. **标签所有权归集合所有者**：成员打标签时，标签 owner 是集合创建者，通过 `name_ownerId` 唯一约束实现跨链接去重。

3. **双路径搜索的结果语义不同**：MeiliSearch 路径中 searchQueryString 是"前置筛选"，tagId/pinnedOnly 是"后置 OR 过滤"；Fallback 中所有条件在同一棵查询树中组合。

4. **批量操作 = 串行单操作**：批量打标、批量删除均为循环调用单体函数，非原子事务，部分失败部分成功。

5. **搜索结果最终一致**：DB 更新后立即失效前端缓存，但 MeiliSearch 索引有延迟窗口，两条路径可能在短时间内返回不同结果。
