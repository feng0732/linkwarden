# Linkwarden Tagging 与列表筛选逻辑深度分析

## 一、核心数据模型与关联机制

### 1.1 Tag 模型：标签所有权与唯一性

Tag 通过 `@@unique([name, ownerId])` 约束实现**同用户下标签名唯一**：

```prisma
model Tag {
  id          Int       @id @default(autoincrement())
  name        String
  owner       User      @relation(fields: [ownerId], references: [id], onDelete: Cascade)
  ownerId     Int
  links       Link[]    @relation("LinkTags")

  @@unique([name, ownerId])
}
```

> **代码位置**：`packages/prisma/schema.prisma`

### 1.2 Link ↔ Tag 多对多关联

Link 与 Tag 通过 `LinkTags` 中间表实现多对多：

```prisma
model Link {
  ...
  tags              Tag[]             @relation("LinkTags")
  collection        Collection?       @relation(fields: [collectionId], references: [id])
  collectionId      Int?
  pinnedBy          User[]            @relation("PinnedLinks")
  ...
}
```

### 1.3 Collection 模型：嵌套与可见性

```prisma
model Collection {
  ...
  ownerId     Int
  isPublic    Boolean         @default(false)
  parent      Collection?     @relation("CollectionTree", fields: [parentId], references: [id])
  parentId    Int?
  members     Membership[]
  links       Link[]
  ...
}
```

---

## 二、打标与去重机制

### 2.1 单链接打标：connectOrCreate + owner 校验

打标逻辑在 `apps/web/lib/api/controllers/links/linkId/updateLinkById.ts` 中，核心是：

```typescript
tags: {
  connectOrCreate: data.tags.map((e) => ({
    where: {
      name_ownerId: {
        name: e.name,
        ownerId: collection?.ownerId as number,
      },
    },
    create: {
      name: e.name,
      owner: { connect: { id: collection?.ownerId as number } },
    },
  })),
},
```

关键点：
- 标签 `ownerId` 始终是 `collection.ownerId`，而非当前操作人
- 使用 `name_ownerId` 唯一索引保证同用户下标签不重复创建

### 2.2 批量打标：串行循环，非原子事务

`apps/web/lib/api/controllers/links/bulk/updateLinks.ts`：

```typescript
for (const link of data.linkIds) {
  const response = await updateLinkById({
    params: { linkId: link },
    body: cleanData,
    req,
  });
  if (response.statusCode > 299) returnNext(response);
}
```

批量打标是**串行调用** `updateLinkById`，非原子事务，中途失败已处理的链接不会回滚。

---

## 三、pinnedOnly 参数完整传递链路

### 3.1 前端默认值：几乎永远是 undefined

`packages/router/links.tsx` 中 `useLinks` hook：

```typescript
const queryParamsObject = {
  ...
  pinnedOnly:
    params.pinnedOnly ?? router.pathname === "/links/pinned"
      ? true
      : undefined,
  ...
};
```

`buildQueryString` 会过滤 undefined 值：

```typescript
const buildQueryString = (params: LinkRequestQuery) => {
  return Object.keys(params)
    .filter((key) => params[key as keyof LinkRequestQuery] !== undefined)
    ...
};
```

**结论**：除 `/links/pinned` 页面显式传 `pinnedOnly=true` 外，其余所有场景下 pinnedOnly 参数均不会出现在 URL 中。

公开页面路径 `packages/router/publicLinks.tsx` 逻辑完全一致。

### 3.2 API 路由层：undefined 保持为 undefined

**旧列表接口** `apps/web/pages/api/v1/links/index.ts`：

```typescript
pinnedOnly: req.query.pinnedOnly
  ? req.query.pinnedOnly === "true"
  : undefined,
```

**搜索接口** `apps/web/pages/api/v1/search/index.ts`：

```typescript
pinnedOnly: req.query.pinnedOnly
  ? req.query.pinnedOnly === "true"
  : undefined,
```

**公开集合接口** `apps/web/pages/api/v1/public/collections/links/index.ts`：

```typescript
pinnedOnly: req.query.pinnedOnly
  ? req.query.pinnedOnly === "true"
  : undefined,
```

三处路由完全一致：URL 无参数 → `undefined`，URL 有 `true` → `true`，其余（如 `false`）→ `undefined`。

### 3.3 前端默认请求接口

`packages/router/links.tsx` 中 `useFetchLinks` 默认请求的是**搜索接口**：

```typescript
const response = await fetch(
  "/api/v1/search?cursor=" + ...
);
```

---

## 四、Prisma 对 `{ pinnedBy: undefined }` 的处理行为（严谨验证）

### 4.1 三接口的 pinnedCondition 构造对比

| 接口 | 代码写法 | pinnedOnly=undefined 时的结果 |
|------|---------|-----------------------------|
| getLinks | `{ pinnedBy: query.pinnedOnly ? { some: { id: userId } } : undefined }` | `{ pinnedBy: undefined }`（对象，含 undefined 属性值） |
| searchLinks | `const pinnedCondition = query.pinnedOnly && userId ? { pinnedBy: { some: { id: userId } } } : {}` | `{}`（空对象） |

### 4.2 Prisma 对 undefined 属性值的处理规则

Prisma 官方文档和实际行为明确：**WHERE 条件对象中值为 `undefined` 的属性会被 Prisma 查询引擎忽略并剔除**。

证据来自 Linkwarden 项目自身的代码模式（全文共 49 处类似用法）：

```typescript
// apps/web/lib/api/controllers/tags/getTags.ts
mode: POSTGRES_IS_ENABLED ? ("insensitive" as const) : undefined,

// apps/web/lib/api/controllers/links/getLinks.ts
mode: POSTGRES_IS_ENABLED ? "insensitive" : undefined,

// apps/web/lib/api/controllers/search/searchLinks.ts
mode: POSTGRES_IS_ENABLED ? "insensitive" : undefined,
```

上述所有模式中，`field: condition ? value : undefined` 被项目开发者用作**"条件满足则加此字段过滤，条件不满足则忽略此字段"**的标准写法。

### 4.3 等价推导

基于上述 Prisma 行为：

```
getLinks 传入 Prisma:  { pinnedBy: undefined }
          ↓ Prisma 剔除 undefined 属性
          等价于:      {}（空对象）

searchLinks 传入 Prisma:  {}
          ↓ 无需处理
          结果:        {}（空对象）
```

### 4.4 空对象 `{}` 在 AND/OR 数组中的语义

在 Prisma WHERE 条件中：
- **AND 数组中**：`{}` 表示"无条件"，等价于逻辑 **TRUE**，不影响其他条件
- **OR 数组中**：`{}` 表示"无条件通过"，等价于逻辑 **TRUE**，导致整个 OR 分支恒真

---

## 五、三接口 AND/OR 嵌套结构对比

### 5.1 getLinks（旧列表接口）

**代码位置**：`apps/web/lib/api/controllers/links/getLinks.ts`

```typescript
AND: [
  // 层1：权限校验
  { collection: { isPublic: true } },
  { OR: [...] }, // ownerId / collection.members

  // 层2：collectionId 过滤（可选）
  query.collectionId ? { collectionId: query.collectionId } : {},

  // 层3：tagId OR (pinnedOnly OR/AND searchConditions)
  {
    OR: [
      ...tagCondition,
      {
        [query.searchQueryString ? "OR" : "AND"]: [
          {
            pinnedBy: query.pinnedOnly
              ? { some: { id: userId } }
              : undefined,
          },
          ...searchConditions,
        ],
      },
    ],
  },
]
```

### 5.2 searchLinks（搜索接口）

**代码位置**：`apps/web/lib/api/controllers/search/searchLinks.ts`

**路径 A（MeiliSearch + PostgreSQL 二次过滤）**：
- 阶段 1（MeiliSearch）：`buildMeiliFilters(tokens, userId, publicOnly)` + 全文检索
- 阶段 2（PostgreSQL 二次过滤）：结构与 getLinks 类似

**路径 B（DB Fallback）**：

```typescript
AND: [
  // 层1：权限校验
  publicOnly ? { collection: { isPublic: true } } : {},
  publicOnly ? {} : { OR: [...] }, // ownerId / collection.members

  // 层2：collectionId 过滤（可选）
  query.collectionId ? { collectionId: query.collectionId } : {},

  // 层3：tagId OR (pinnedCondition OR/AND searchConditions)
  {
    OR: [
      ...tagCondition,
      {
        [query.searchQueryString ? "OR" : "AND"]: [
          pinnedCondition, // = query.pinnedOnly && userId ? { ... } : {}
          ...searchConditions,
        ],
      },
    ],
  },
]
```

---

## 六、两种典型场景的查询树可视化

### 6.1 场景一：pinnedOnly = true（仅 /links/pinned 页面）

```
AND
├─ [权限条件] ✅
├─ [collectionId 条件] ✅（可选）
└─ OR
   ├─ tagCondition（可选） ✅ 正常生效
   └─ [OR/AND]
      ├─ pinnedCondition = { pinnedBy: { some: { id: userId } } } ✅
      └─ searchConditions（可选） ✅ 正常生效
```

**此场景下，tagId 和 searchQueryString 均正常工作。**

### 6.2 场景二：pinnedOnly = undefined（绝大多数场景）

```
AND
├─ [权限条件] ✅
├─ [collectionId 条件] ✅（可选）
└─ OR
   ├─ tagCondition（可选） ❌ 因另一分支恒真而失效
   └─ [OR/AND]
      ├─ pinnedCondition = {} = TRUE  ← 恒真！
      └─ searchConditions（可选） ❌ 因 AND 含 TRUE 或 OR 含 TRUE 而失效
```

**此场景下，tagId 和 searchQueryString 在 DB 查询层均失效！**

---

## 七、三接口四条件叠加对比表（最终校正版）

| 条件组合 | getLinks（旧接口） | search MeiliSearch 路径 | search DB Fallback 路径 |
|---------|-------------------|------------------------|------------------------|
| **PO=true, tagId, search, CID** | tag ✅ OR (pinned ✅ OR search ✅)，AND CID ✅ | Meili 阶段：search ✅，pinned ✅，tag ✅（语法），CID ❌<br>PG 阶段：tag ✅ OR (pinned ✅ OR search ✅)，AND CID ✅ | tag ✅ OR (pinned ✅ OR search ✅)，AND CID ✅ |
| **PO=undefined, tagId, search, CID** | tag ❌ OR (TRUE OR search ❌) = **恒真**，AND CID ✅<br>tag 失效，search 失效，CID 有效 | Meili 阶段：search ✅，tag ✅（语法），CID ❌<br>PG 阶段：tag ❌ OR (TRUE OR search ❌) = **恒真**，AND CID ✅<br>最终：CID 有效，Meili search 仍生效，tag/PG search 失效 | tag ❌ OR (TRUE OR search ❌) = **恒真**，AND CID ✅<br>tag 失效，search 失效，CID 有效 |
| **PO=undefined, tagId, no search, CID** | tag ❌ OR (TRUE AND []) = **恒真**，AND CID ✅<br>tag 失效，CID 有效 | Meili 阶段：无 search，tag ✅（语法），CID ❌<br>PG 阶段：tag ❌ OR TRUE = **恒真**，AND CID ✅<br>最终：CID 有效，tag 双阶段均不生效 | tag ❌ OR TRUE = **恒真**，AND CID ✅<br>tag 失效，CID 有效 |
| **PO=undefined, no tagId, search, CID** | [] OR (TRUE OR search ❌) = **恒真**，AND CID ✅<br>search 失效，CID 有效 | Meili 阶段：search ✅，CID ❌<br>PG 阶段：[] OR (TRUE OR search ❌) = **恒真**，AND CID ✅<br>最终：CID 有效，Meili search 仍生效，PG search 失效 | [] OR (TRUE OR search ❌) = **恒真**，AND CID ✅<br>search 失效，CID 有效 |
| **PO=undefined, no tagId, no search, CID** | [] OR (TRUE AND []) = **恒真**，AND CID ✅<br>仅 CID 有效 | Meili 阶段：无 search，CID ❌<br>PG 阶段：[] OR TRUE = **恒真**，AND CID ✅<br>最终：仅 CID 有效 | [] OR TRUE = **恒真**，AND CID ✅<br>仅 CID 有效 |

> **注**：PO = pinnedOnly，CID = collectionId

---

## 八、MeiliSearch 路径 vs DB Fallback 路径结果差异

### 8.1 MeiliSearch 过滤器构造

**代码位置**：`apps/web/lib/api/searchQueryBuilder.ts`

```typescript
export const buildMeiliFilters = (
  tokens: SearchToken[],
  userId: number,
  publicOnly: boolean = false
): string => { ... }
```

**注意**：函数签名**不接收 collectionId 参数**！collectionId 仅在 PostgreSQL 二次过滤阶段生效。

支持的搜索语法：`url:`、`name:`、`description:`、`type:`、`collection:`（按集合名，非 ID）、`pinned:`、`public:`、`before:`、`after:`、`tag:`，否定语法 `!tag:xxx`。

### 8.2 四种差异场景推演

| 场景 | MeiliSearch 路径结果 | DB Fallback 路径结果 | 是否一致 |
|-----|---------------------|---------------------|---------|
| PO=undefined + tagId + search | Meili 阶段按 search 词检索返回相关结果，PG 阶段 OR 恒真不过滤，最终返回** Meili 排序后的搜索结果** | PG 单阶段 OR 恒真，返回** CID 内所有链接**，与 search 和 tagId 无关 | ❌ **不一致** |
| PO=undefined + tagId + no search | Meili 阶段无关键词，仅基础权限过滤，PG 阶段 OR 恒真，最终返回** CID 内所有链接** | PG 单阶段 OR 恒真，返回** CID 内所有链接** | ✅ 一致 |
| PO=undefined + no tagId + search | Meili 阶段按 search 词检索，PG 阶段 OR 恒真，最终返回** Meili 搜索结果** | PG 单阶段 OR 恒真，返回** CID 内所有链接** | ❌ **不一致** |
| PO=true + tagId + search | Meili 阶段 tag+search+pinned 过滤，PG 二次过滤，**结果精准** | PG 单阶段全量过滤，**结果精准** | ✅ 一致（排序可能不同） |

---

## 九、标签删除/合并与搜索联动

### 9.1 数据一致性保障

三处操作均将关联链接的 `indexVersion` 置为 `null`：

- **删除标签**：`apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts`
- **批量删除标签**：`apps/web/lib/api/controllers/tags/bulk/bulkTagDelete.ts`
- **合并标签**：`apps/web/lib/api/controllers/tags/mergeTags.ts`

```typescript
await prisma.link.updateMany({
  where: { tags: { some: { id: tag.id } } },
  data: { indexVersion: null },
});
```

`indexVersion = null` 触发后台异步任务将链接重新提交给 MeiliSearch 索引，实现最终一致性。

### 9.2 tags.name 搜索的独有可见性校验

**仅 getLinks 中存在**，`apps/web/lib/api/controllers/links/getLinks.ts`：

```typescript
tagCondition = [
  {
    tags: {
      some: {
        name: {
          contains: query.searchQueryString,
          mode: POSTGRES_IS_ENABLED ? "insensitive" : undefined,
        },
        OR: [
          { ownerId: userId },
          { links: { some: { collection: { members: { some: { userId } } } } } },
        ],
      },
    },
  },
];
```

searchLinks 的 DB Fallback 路径**缺少**此可见性校验，可能返回用户无权查看的标签匹配结果。

---

## 十、关键设计要点与潜在问题汇总

### ✅ 设计合理之处

1. **标签去重机制**：利用 `@@unique([name, ownerId])` + Prisma `connectOrCreate` 实现原子级去重
2. **双路径搜索架构**：MeiliSearch 提供高性能全文检索，DB Fallback 保障可用性
3. **最终一致性**：标签变更后通过 `indexVersion = null` 触发异步重索引
4. **权限分层**：权限条件放在 AND 最外层，始终优先执行

### ⚠️ 潜在问题

1. **tagId / searchQueryString 在绝大多数场景下 DB 层失效**（高优先级）
   - 前端默认不传 pinnedOnly → pinnedOnly=undefined → OR 分支恒真 → tagId 和 search 在 DB 查询层永远不生效
   - 仅 `/links/pinned` 页面不受影响

2. **三路径行为不一致**（中优先级）
   - MeiliSearch 路径 + PO=undefined + search 关键词：返回 Meili 搜索结果
   - DB Fallback 路径 + PO=undefined + search 关键词：返回集合内所有链接，忽略搜索词
   - 最终用户可能因 MeiliSearch 可用性得到截然不同的结果

3. **searchLinks DB Fallback 缺少标签可见性校验**（低优先级）
   - 当 searchQueryString 匹配到用户无权限查看的标签名称时，Fallback 路径可能返回不应该出现的链接
   - getLinks 路径对此做了校验，但 Fallback 路径未同步

4. **批量打标非原子事务**（低优先级）
   - 串行调用 `updateLinkById`，中途失败已打标的链接不会回滚

5. **buildMeiliFilters 不支持 collectionId**（中优先级）
   - MeiliSearch 阶段无法按 collectionId 过滤，全部依赖 PG 二次过滤
   - 搜索语法中的 `collection:Name`（按集合名）与 URL 参数 `collectionId`（按 ID）是两个独立机制
