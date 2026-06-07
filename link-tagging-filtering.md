# Linkwarden Tagging 与列表筛选组合逻辑深度分析（复核校正版）

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

## 三、pinnedCondition 三路径差异（关键校正点）

这是理解整体逻辑的基石。三个接口对 `pinnedOnly=false`（或未传）的处理完全不同。

### 3.1 pinnedCondition 的定义对比

| 接口 | 代码位置 | pinnedOnly=true | pinnedOnly=false / 未传 |
|------|---------|----------------|------------------------|
| **旧列表接口 getLinks** | `apps/web/lib/api/controllers/links/getLinks.ts#L117-L120` | `{ pinnedBy: { some: { id: userId } } }` | `undefined` |
| **搜索接口 searchLinks**（两路径共用） | `apps/web/lib/api/controllers/search/searchLinks.ts#L51-L52` | `{ pinnedBy: { some: { id: userId } } }` | `{}`（空对象） |

**核心差异：**
- `undefined` 在 Prisma 数组中会被**完全忽略**
- `{}`（空对象）在 Prisma WHERE 子句中表示 **TRUE（匹配所有记录）**

---

## 四、多条件筛选的 AND/OR 嵌套逻辑（复核校正）

### 4.1 查询参数类型

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

### 4.2 旧列表接口（getLinks）的完整查询树

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
        // 🔑 额外的标签可见性校验（getLinks 独有，search Fallback 无此校验）
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

当 `tagId=T`、`searchQueryString=S`、`collectionId=C`、`pinnedOnly=true` **全部传入**时：

```
权限过滤
  AND collectionId = C
  AND (
    tagId = T                          ← 分支 A：只要匹配标签
    OR (
      pinnedBy 包含 userId             ← 分支 B-1：已置顶
      OR name 包含 S                    ← 分支 B-2：
      OR url   包含 S                    ← 分支 B-3：
      OR description 包含 S              ← 分支 B-4：
      OR (tags.name 包含 S AND 标签可见)  ← 分支 B-5
    )
  )
```

**语义：** 链接必须属于指定集合，且满足以下条件之一：
1. 被打了指定标签（tagId=T），**不管是否置顶、不管名称是否匹配**
2. **或者**（已置顶 OR 名称匹配 OR URL 匹配 OR 描述匹配 OR 标签名匹配）

---

#### pinnedOnly=false 时的真实行为（getLinks）

当 `pinnedOnly=false`（或未传），但 `tagId=T`、`searchQueryString=S`、`collectionId=C` 同时存在时：

```typescript
// 分支 B 变成：
OR: [
  { pinnedBy: undefined },  // Prisma 忽略 undefined，数组元素消失
  ...searchConditions,
]
// 实际等价于：
OR: [...searchConditions]
```

完整逻辑变为：

```
权限过滤
  AND collectionId = C
  AND (
    tagId = T
    OR (name 包含 S OR url 包含 S OR description 包含 S OR tags.name 包含 S)
  )
```

**语义：** 属于集合 C，且（打了标签 T **或者** 关键词匹配任一字段）

---

### 4.3 搜索接口（searchLinks）的查询树

搜索接口有两条路径，取决于 MeiliSearch 是否可用。

---

#### 路径 A：MeiliSearch 可用 + 有 searchQueryString

**实现位置** `apps/web/lib/api/controllers/search/searchLinks.ts#L54-L152`

```
┌──────────────────────────────────────────────────────────────┐
│  阶段 1：MeiliSearch 检索（粗筛）                               │
│  输入：                                                       │
│    meiliQuery   = tokens 中 field==="general" 的词拼接         │
│    meiliFilters = [                                           │
│      ① 基础权限：                                             │
│        publicOnly → ["collectionIsPublic = true"]             │
│        否则 → ["(collectionOwnerId = U) OR (collectionMemberIds = U)"] │
│      ② 搜索语法解析出的过滤器（从 searchQueryString 解析）：     │
│         tag:xxx / !tag:xxx                                    │
│         collection:xxx / !collection:xxx  ← 注意：这是按集合名，不是 query.collectionId │
│         pinned:true/false                                     │
│         url:xxx / name:xxx / description:xxx / type:xxx       │
│         before:xxx / after:xxx / public:true                  │
│    ]    ⚠️ collectionId（URL 参数） 不在此阶段过滤！            │
│  输出：匹配的 linkId 数组（meiliIds）                          │
└──────────────────────────┬───────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  阶段 2：PostgreSQL 二次过滤（精筛 + 补全数据）                  │
│  where: {                                                     │
│    id: { in: meiliIds },   ← 限定在 MeiliSearch 结果内         │
│    AND: [                                                      │
│      ① 权限过滤（同前）                                         │
│      ② collectionCondition（query.collectionId 在此生效！）     │
│      {                                                         │
│        OR: [                                                   │
│          ...tagCondition,       ← tagId 匹配                   │
│          {                                                      │
│            ...pinnedCondition,  ← pinnedOnly（展开对象）        │
│          },                                                     │
│        ],                                                       │
│      },                                                         │
│    ],                                                           │
│  }                                                              │
│  ⚠️ searchConditions 在此阶段不重复执行（已由 Meili 处理）       │
└──────────────────────────────────────────────────────────────┘
```

**🔑 关键校正：collectionId 在 MeiliSearch 过滤器中不存在！**

`buildMeiliFilters` 的函数签名 `apps/web/lib/api/searchQueryBuilder.ts#L93-L101`：

```typescript
export function buildMeiliFilters({
  tokens,
  userId,
  publicOnly,
}: {
  tokens: Token[];
  userId?: number;
  publicOnly?: boolean;
}): string[]
```

没有 `collectionId` 参数。`query.collectionId` 仅在阶段 2 的 PostgreSQL 二次过滤中通过 `...collectionCondition` 生效。

---

#### 阶段 2 中 pinnedCondition 的特殊行为（关键校正）

MeiliSearch 路径阶段 2 的代码 `apps/web/lib/api/controllers/search/searchLinks.ts#L116-L123`：

```typescript
{
  OR: [
    ...tagCondition,
    {
      ...pinnedCondition,  // 🔑 直接展开对象，不是数组元素
    },
  ],
}
```

**情况 1：pinnedOnly=true**

```typescript
pinnedCondition = { pinnedBy: { some: { id: userId } } }
// 展开后：
OR: [
  ...tagCondition,                    // 可能有 { tags: { some: { id: T } } }
  { pinnedBy: { some: { id: userId } } },  // 置顶条件
]
```

语义：`tagId 匹配 OR 已置顶` ✓

**情况 2：pinnedOnly=false（或未传）**

```typescript
pinnedCondition = {}  // 空对象
// 展开后：
OR: [
  ...tagCondition,  // 可能有 { tags: { some: { id: T } } }
  {},               // 🔴 空对象 = TRUE（匹配所有记录）
]
// 整个 OR 表达式恒为 TRUE！
```

**🔴 严重后果：当 pinnedOnly=false 且 tagId 存在时，tagId 过滤也失效了。**

因为 `OR: [条件A, TRUE]` 结果永远是 TRUE，无论条件A是否满足。

---

#### 四条件同时存在且 pinnedOnly=true 时（MeiliSearch 路径）

`tagId=T`、`searchQueryString=S`、`collectionId=C`、`pinnedOnly=true`：

```
【阶段 1：MeiliSearch】
全文检索 query = "S"（general tokens）
AND filter:
  ├── (collectionOwnerId=U OR collectionMemberIds=U)
  └── （如有搜索语法 tag:/collection:/pinned: 也在此过滤）
        ↓ 输出 meiliIds

【阶段 2：PostgreSQL】
AND
├── id IN (meiliIds)
├── 权限过滤
├── collection.id = C           ← collectionId 仅在此生效
└── OR
    ├── tags.some: id = T       ← 分支 A：tagId 匹配
    └── pinnedBy.some: id = U   ← 分支 B：已置顶
```

**结果语义 =**
（MeiliSearch 搜索 "S" 的结果） AND （属于集合 C） AND （被打了 tag#T **OR** 已置顶）

**⚠️ 关键差异：** searchQueryString 只在阶段 1 的 Meili 全文检索生效，阶段 2 的 PG 二次过滤不再检查名称/URL/描述/标签名匹配。

---

#### 四条件同时存在且 pinnedOnly=false 时（MeiliSearch 路径）

`tagId=T`、`searchQueryString=S`、`collectionId=C`、`pinnedOnly=false`：

```
【阶段 2：PostgreSQL】
AND
├── id IN (meiliIds)
├── 权限过滤
├── collection.id = C
└── OR
    ├── tags.some: id = T       ← 分支 A
    └── {}                      ← 分支 B：空对象 = TRUE
//                                  → 整个 OR = TRUE，完全不过滤！
```

**结果语义 =**
（MeiliSearch 搜索 "S" 的结果） AND （属于集合 C）
**tagId 过滤完全失效！** 🔴

---

#### 路径 B：MeiliSearch 不可用 或 无 searchQueryString（DB Fallback）

**实现位置** `apps/web/lib/api/controllers/search/searchLinks.ts#L155-L255`

```typescript
where: {
  AND: [
    // ① 权限过滤
    ...(userId ? [权限条件] : []),
    // ② collectionId 过滤
    ...collectionCondition,
    // ③ tagId 与 (pinned + search) 的 OR
    {
      OR: [
        ...tagCondition,
        {
          // 有 search 用 OR，无 search 用 AND
          [query.searchQueryString ? "OR" : "AND"]: [
            pinnedCondition,  // 🔑 pinnedOnly=false 时是 {}（空对象）
            ...searchConditions,
          ],
        },
      ],
    },
  ],
}
```

与旧列表接口 getLinks 的差异：

| 差异点 | getLinks | searchLinks Fallback |
|--------|----------|---------------------|
| pinnedOnly=false 时 | `undefined`（Prisma 忽略） | `{}`（空对象 = TRUE） |
| tags.name 搜索可见性校验 | ✅ 有 `OR: [ownerId, members]` | ❌ 无 |

---

#### Fallback 路径中 pinnedCondition 的特殊行为（关键校正）

**情况 1：pinnedOnly=true + 有 searchQueryString**

```typescript
pinnedCondition = { pinnedBy: { some: { id: userId } } }
// 分支 B：
OR: [
  { pinnedBy: { some: { id: userId } } },
  ...searchConditions,
]
```

语义：`置顶 OR 搜索匹配` ✓

**情况 2：pinnedOnly=false + 有 searchQueryString**

```typescript
pinnedCondition = {}  // 空对象
// 分支 B：
OR: [
  {},               // 🔴 TRUE
  ...searchConditions,
]
// 整个 OR = TRUE（第一个元素就为真）
```

**🔴 后果：** 分支 B 恒为 TRUE → 外层 OR（tagId OR 分支B）也恒为 TRUE → **tagId 过滤完全失效！**

**情况 3：pinnedOnly=false + 无 searchQueryString**

```typescript
pinnedCondition = {}  // 空对象
// 分支 B：
AND: [
  {},               // TRUE
  ...[],            // searchConditions 为空
]
// AND [TRUE] = TRUE
```

同样：分支 B 恒为 TRUE → tagId 过滤完全失效！🔴

---

### 4.4 三接口四条件叠加对比表（校正版）

当 **tagId=T、searchQueryString=S、collectionId=C、pinnedOnly=PO** 同时传入时：

| 条件 | 旧接口 getLinks | 搜索接口 MeiliSearch 路径 | 搜索接口 Fallback |
|------|----------------|--------------------------|-------------------|
| **权限过滤** | ✅ AND | ✅ Meili阶段(权限filter) + PG阶段 AND | ✅ AND |
| **collectionId=C** | ✅ AND（PG） | ⚠️ 仅 PG 阶段 AND，**Meili 阶段不过滤** | ✅ AND（PG） |
| **搜索语法 collection:Name** | ❌ 不支持 | ✅ Meili filter 阶段按集合名过滤 | ❌ 不支持 |
| **tagId=T** | ✅ OR 分支A | ⚠️ PG 阶段 OR 分支A（但 PO=false 时失效） | ⚠️ OR 分支A（PO=false 时失效） |
| **pinnedOnly=true** | ✅ OR 分支B-1 | ✅ PG 阶段 OR 分支B | ✅ OR 分支B-1 |
| **pinnedOnly=false** | ✅ undefined（被忽略） | 🔴 `{}` → OR恒TRUE → tagId也失效 | 🔴 `{}` → OR恒TRUE → tagId也失效 |
| **S 匹配 name/url/description** | ✅ OR 分支B-2~4 | ✅ Meili 全文检索（PG阶段不重复） | ✅ OR 分支B-2~4 |
| **S 匹配 tags.name（含可见性校验）** | ✅ OR 分支B-5（有校验） | ✅ Meili tags 字段索引 | ✅ OR 分支B-5（**无可见性校验**） |
| **高级语法** `tag:`/`!tag:`/`pinned:` 等 | ❌ 不支持 | ✅ Meili filter 解析 | ❌ 不支持 |
| **搜索结果排序** | PG 按字段排序 | Meili 按相关性 + 指定字段排序 | PG 按字段排序 |
| **分页方式** | cursor-based（ID 游标） | offset-based（数字偏移） | cursor-based（ID 游标） |
| **一致性** | 强一致 | 最终一致（indexVersion 异步） | 强一致 |

---

## 五、多条件同时存在时的完整查询树可视化（校正版）

### 5.1 场景 A：全部四个条件同时传入且 pinnedOnly=true

请求参数示例：
```
GET /api/v1/search?collectionId=5&tagId=12&pinnedOnly=true&searchQueryString=react
```

---

#### 旧列表接口 getLinks 的查询树

```
AND
├── 权限：collection.ownerId=U OR collection.members 包含 U
├── collection.id = 5
└── OR
    ├── 【分支A】tags.some: id = 12
    │       （只要打了 tag#12，其他条件都不看）
    │
    └── 【分支B】OR
        ├── pinnedBy.some: id = U
        ├── name ILIKE "%react%"
        ├── url ILIKE "%react%"
        ├── description ILIKE "%react%"
        └── tags.some: (
              name ILIKE "%react%"
              AND (ownerId=U OR links.collection.members 包含 U)
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
  ├── (collectionOwnerId=U OR collectionMemberIds=U)  ← 仅权限，无 collectionId=5
  └── （如有搜索语法 tag:/collection:/pinned: 也在此过滤）
        ↓ 输出 meiliIds（可能包含非集合5的链接）

【阶段 2：PostgreSQL】
AND
├── id IN (meiliIds)
├── 权限：同上
├── collection.id = 5           ← collectionId=5 仅在此生效
└── OR
    ├── tags.some: id = 12      ← 分支 A：tagId 匹配
    └── pinnedBy.some: id = U   ← 分支 B：已置顶
```

**结果语义 =**
（MeiliSearch 搜索 "react" 的结果） AND （属于集合 5） AND （被打了 tag#12 **OR** 置顶）

**⚠️ 关键差异：**
1. searchQueryString 只在阶段 1 生效，阶段 2 不再检查字段匹配
2. collectionId=5 不在 Meili 阶段过滤，阶段 1 可能返回其他集合的链接，阶段 2 才剔除
3. 阶段 2 中 tagId 和 pinnedOnly 是 OR 关系，与 search 完全解耦

---

### 5.2 场景 B：pinnedOnly=false，其他三条件存在

请求参数示例：
```
GET /api/v1/search?collectionId=5&tagId=12&pinnedOnly=false&searchQueryString=react
```

---

#### 旧列表接口 getLinks

```
AND
├── 权限
├── collection.id = 5
└── OR
    ├── tags.some: id = 12      ← 分支 A：tagId 匹配
    └── OR
        ├── pinnedBy: undefined  ← 被 Prisma 忽略，从数组中移除
        ├── name ILIKE "%react%"
        ├── url ILIKE "%react%"
        ├── description ILIKE "%react%"
        └── tags.name ILIKE "%react%"（含可见性校验）
```

**结果语义 =** （属于集合 5） AND （tag#12 **OR** 关键词匹配）

tagId 过滤正常生效 ✓

---

#### 搜索接口 MeiliSearch 路径

```
【阶段 2：PostgreSQL】
AND
├── id IN (meiliIds)
├── 权限
├── collection.id = 5
└── OR
    ├── tags.some: id = 12   ← 分支 A
    └── {}                   ← 分支 B：空对象 = TRUE
//                                  → 整个 OR = TRUE
```

**结果语义 =** （Meili 搜索 "react" 结果） AND （属于集合 5）

**tagId=12 过滤完全失效！** 🔴 所有 Meili 返回且属于集合 5 的链接都会被返回，不管是否打了 tag#12。

---

#### 搜索接口 Fallback

```
AND
├── 权限
├── collection.id = 5
└── OR
    ├── tags.some: id = 12      ← 分支 A
    └── OR
        ├── {}                   ← 🔴 空对象 = TRUE
        ├── name ILIKE "%react%"
        ├── ...
//              → 内层 OR = TRUE → 外层 OR = TRUE
```

**结果语义 =** （属于集合 5）

**tagId=12 和 searchQueryString="react" 都失效！** 🔴 所有属于集合 5 的链接都会被返回，tag 和搜索条件完全不起作用。

---

## 六、MeiliSearch 与 DB Fallback 的结果差异（校正版）

### 6.1 差异总结表

| 维度 | MeiliSearch 路径 | DB Fallback（getLinks / search Fallback） |
|------|-----------------|------------------------------------------|
| **触发条件** | meiliClient 存在 **且** searchQueryString 非空 | meiliClient 不存在 **或** searchQueryString 为空 |
| **collectionId 过滤阶段** | 仅 PG 二次过滤阶段（Meili 阶段不过滤） | PG 单阶段（与其他条件同树） |
| **搜索能力** | 全文检索 + 排名打分 + 高级语法 | 简单 `contains` 子串匹配，无排名 |
| **高级语法** | `tag:` `collection:` `pinned:` `before:` `after:` `!否定` | 不支持 |
| **pinnedOnly=false 对 tagId 的影响** | 🔴 tagId 完全失效（OR 恒 TRUE） | getLinks: ✓ 正常；Fallback: 🔴 tagId 失效 |
| **pinnedOnly=false + 有 search 对 search 的影响** | search 在 Meili 阶段正常执行 | Fallback: 🔴 search 也失效（OR 恒 TRUE） |
| **标签名搜索的可见性校验** | Meili 索引时已过滤 | getLinks: ✅ 有；Fallback: ❌ 无 |
| **一致性** | 最终一致（依赖 indexVersion 异步同步） | 强一致 |
| **分页方式** | offset-based（数字） | cursor-based（ID 游标） |

### 6.2 可能导致结果不一致的场景

#### 场景 1：tagId 存在 + searchQueryString 存在 + pinnedOnly=false

| 路径 | 结果 |
|------|------|
| getLinks | tagId OR 搜索匹配 → tagId 正常 ✓ |
| search Meili | （Meili 搜索结果） AND collectionId → **tagId 失效** 🔴 |
| search Fallback | 仅 collectionId → **tagId 和 search 都失效** 🔴 |

#### 场景 2：tagId 存在 + searchQueryString 存在 + pinnedOnly=true

| 路径 | 结果 |
|------|------|
| getLinks | tagId OR (置顶 OR 搜索匹配) |
| search Meili | （Meili 搜索结果） AND (tagId OR 置顶) |
| search Fallback | tagId OR (置顶 OR 搜索匹配) |

**差异：** 如果某链接打了 tag#12 但名称/URL/描述/标签名都不含 "react"：
- getLinks / Fallback 会返回它（走了 tagId 分支 A）
- MeiliSearch 路径不会返回它（阶段 1 Meili 全文检索未匹配）

#### 场景 3：searchQueryString 恰好匹配了某个协作集合中的私有标签

- **getLinks：** tags.name 搜索时有可见性校验（ownerId OR members），不可见的标签不会被匹配
- **search Fallback：** tags.name 搜索时**没有**可见性校验，可能匹配到不应可见的标签名
- **MeiliSearch 路径：** 索引时已按权限过滤，查询时不额外校验

#### 场景 4：数据刚更新，MeiliSearch 尚未同步

- **Fallback / getLinks：** 返回最新数据
- **MeiliSearch：** 可能返回旧数据或漏掉新数据（直到 Worker 处理 indexVersion=null）

---

## 七、标签删除与搜索联动

### 7.1 单个标签删除

`apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts`

```typescript
// 1. 权限校验：只能删除自己拥有的标签
if (targetTag?.ownerId !== userId) return { response: "Permission denied.", status: 401 };

// 2. 删除标签（Prisma 自动解除多对多关联）
const deletedTag = await prisma.tag.delete({
  where: { id: tagId },
  include: { links: { select: { id: true } } },
});

// 3. 将所有关联链接的 indexVersion 置 null，触发 MeiliSearch 重索引
await prisma.link.updateMany({
  where: { id: { in: linkIds } },
  data: { indexVersion: null },
});
```

### 7.2 批量标签删除

`apps/web/lib/api/controllers/tags/bulkTagDelete.ts`

```typescript
// 1. 找出所有受影响的链接（用户所有标签相关的链接）
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

### 7.3 标签合并

`apps/web/lib/api/controllers/tags/mergeTags.ts`

事务中三步操作：
1. 删除所有旧标签
2. 创建新标签并关联所有受影响链接
3. 所有受影响链接 `indexVersion = null` 触发重索引

---

## 八、批量打标与搜索联动

### 8.1 前端批量编辑流程

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

### 8.2 后端批量打标实现

`apps/web/lib/api/controllers/links/bulk/updateLinks.ts`

```typescript
// 串行调用单链接更新，非原子事务
for (const l of links) {
  await updateLinkById(userId, link.id, updatedData, removePreviousTags);
}
```

每次 `updateLinkById` 都会将该链接的 `indexVersion` 置为 null。

### 8.3 触发重索引的操作汇总

| 操作 | 代码位置 | 影响范围 |
|------|---------|---------|
| 更新链接（含打标） | `apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L163` | 单条链接 |
| 删除单个标签 | `apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts#L36-L45` | 所有关联该标签的链接 |
| 批量删除标签 | `apps/web/lib/api/controllers/tags/bulkTagDelete.ts#L53-L62` | 用户所有标签相关的链接 |
| 合并标签 | `apps/web/lib/api/controllers/tags/mergeTags.ts#L64-L73` | 所有受影响的链接 |

### 8.4 最终一致性时序

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

## 九、关键设计要点（校正总结）

### ✅ 已确认正确的设计

1. **tagId 与 (pinned + search) 是 OR 关系，不是 AND**：指定 tagId 后，即使 pinnedOnly=true，未置顶但打了该标签的链接也会被返回（getLinks 中）。

2. **标签所有权归集合所有者**：成员打标签时，标签 owner 是集合创建者，通过 `name_ownerId` 唯一约束实现跨链接去重。

3. **collectionId（URL 参数）不在 MeiliSearch 阶段过滤**：仅在 PostgreSQL 二次过滤阶段生效，与搜索语法中的 `collection:Name`（按集合名在 Meili 阶段过滤）是两个独立机制。

### 🔴 复核发现的潜在问题

4. **pinnedOnly=false 时 searchLinks 的 tagId 过滤失效**：
   - searchLinks 中 `pinnedCondition = {}`（空对象）而非 `undefined`
   - 空对象在 Prisma OR 数组中 = TRUE，导致整个 OR 分支恒为 TRUE
   - **MeiliSearch 路径：** tagId 失效
   - **Fallback 路径：** tagId 和 searchConditions 都失效（分支 B 恒 TRUE）
   - **getLinks 不受影响**（使用 `undefined`，Prisma 忽略）

5. **searchLinks Fallback 的 tags.name 搜索缺少可见性校验**：getLinks 在搜索标签名时有额外的 `OR: [ownerId, members]` 校验，search Fallback 没有，可能搜索到不应可见的标签。

6. **批量操作 = 串行单操作**：批量打标、批量删除均为循环调用单体函数，非原子事务，可能部分失败部分成功。

7. **搜索结果最终一致**：DB 更新后立即失效前端缓存，但 MeiliSearch 索引有延迟窗口，两条路径可能在短时间内返回不同结果。
