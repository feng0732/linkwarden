# Linkwarden 阅读后工作流与状态标记代码分析

## 概述

Linkwarden 是一个开源的链接管理/书签管理应用。经过代码分析，该项目**并没有独立的"阅读后"（Read Later）状态字段或"收藏"（Favorite）功能**。而是通过以下机制组合实现类似功能：

- **置顶（Pin）**：用户级别的个人标记，相当于"收藏/稍后阅读"
- **集合（Collection）**：用于组织链接的文件夹，可作为"阅读列表"
- **标签（Tag）**：辅助分类
- **成员协作（Members）**：多用户共享集合的权限体系

---

## 一、核心数据模型

### 1. Link 模型（链接）

数据模型定义见 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/packages/prisma/schema.prisma#L166-L198)

```prisma
model Link {
  id              Int         @id @default(autoincrement())
  name            String      @default("")
  url             String?
  collection      Collection  @relation(fields: [collectionId], references: [id])
  collectionId    Int
  tags            Tag[]
  pinnedBy        User[]      @relation("PinnedLinks")  // 多对多：谁置顶了这个链接
  createdBy       User?       @relation("CreatedLinks", fields: [createdById], references: [id])
  createdById     Int?
  // ... 其他字段
}
```

**关键发现**：Link 模型中**没有** `status`（已读/未读状态）、`favorite`（收藏）、`readAt`（阅读时间）等字段。

### 2. User 模型与置顶关系

见 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/packages/prisma/schema.prisma#L28-L75)

```prisma
model User {
  pinnedLinks    Link[]                @relation("PinnedLinks")  // 用户置顶的所有链接
  collections    Collection[]          // 用户拥有的集合
  collectionsJoined UsersAndCollections[]  // 用户加入的协作集合
}
```

### 3. 集合成员权限模型

见 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/packages/prisma/schema.prisma#L151-L164)

```prisma
model UsersAndCollections {
  user         User       @relation(fields: [userId], references: [id])
  userId       Int
  collection   Collection @relation(fields: [collectionId], references: [id])
  collectionId Int
  canCreate    Boolean    // 可创建链接
  canUpdate    Boolean    // 可更新链接
  canDelete    Boolean    // 可删除链接
  @@id([userId, collectionId])
}
```

成员角色映射（前端逻辑）见 [EditCollectionSharingModal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/components/ModalContent/EditCollectionSharingModal.tsx#L248-L261)：

| 角色 | canCreate | canUpdate | canDelete | 说明 |
|------|-----------|-----------|-----------|------|
| viewer | false | false | false | 仅查看 |
| contributor | true | false | false | 可创建链接 |
| admin | true | true | true | 完全管理 |

---

## 二、置顶（Pin）机制 —— 相当于"收藏/稍后阅读"

### 1. 置顶的用户隔离特性

置顶是**完全用户隔离**的——每个用户对同一个链接可以独立置顶/取消置顶，互不影响。

#### 数据库查询层面

见 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L132-L137)

```typescript
include: {
  tags: true,
  collection: true,
  pinnedBy: userId
    ? {
        where: { id: userId },  // 只查询当前用户的置顶状态
        select: { id: true },
      }
    : undefined,
},
```

查询返回的 `pinnedBy` 数组**只包含当前用户**（如果该用户置顶了此链接）。前端通过检查 `pinnedBy.length > 0` 判断是否置顶。

#### 置顶/取消置顶操作

见 [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L40-L65)

```typescript
// 协作成员即使没有更新权限也可以置顶
if (canPinPermission && data.pinnedBy && data.pinnedBy[0]) {
  const updatedLink = await prisma.link.update({
    data: {
      pinnedBy: data?.pinnedBy
        ? data.pinnedBy[0]?.id === userId
          ? { connect: { id: userId } }     // 置顶：建立关联
          : { disconnect: { id: userId } }   // 取消：断开关联
        : undefined,
    },
    // ...
  });
}
```

使用 Prisma 的 `connect` / `disconnect` 操作多对多关系，**不会影响其他用户的置顶状态**。

### 2. 置顶操作的权限分支详解

置顶操作在 [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts) 中有**两条独立的执行路径**，针对不同身份的用户：

#### 路径一：纯置顶操作（协作成员专用快速路径）

**触发条件**（第41行）：
```typescript
if (canPinPermission && data.pinnedBy && data.pinnedBy[0])
```

- `canPinPermission`：用户是集合成员（任何角色，包括 viewer）
- 请求体中只修改 `pinnedBy` 字段，且 `pinnedBy[0]` 有值

**权限判断**（第36-38行）：
```typescript
const canPinPermission = collectionIsAccessible?.members.some(
  (e: UsersAndCollections) => e.userId === userId
);
```
即：只要在 `members` 列表中存在即可，**不需要 `canUpdate=true`**。这意味着 **viewer 角色也能置顶**。

**此路径的限制**：
- 只更新 `pinnedBy` 字段，其他字段（name, url, collection, tags 等）一律不更新
- 执行完后直接 return，跳过后续的完整权限校验

#### 路径二：完整更新操作（所有者 + admin 角色）

当用户不仅修改置顶，还修改其他字段（如名称、URL、集合、标签等）时，进入此路径。

**权限判断**（第72-74行 + 第97-101行）：
```typescript
const memberHasAccess = collectionIsAccessible?.members.some(
  (e: UsersAndCollections) => e.userId === userId && e.canUpdate
);

// 非所有者且无更新权限则不能编辑
if (collectionIsAccessible?.ownerId !== userId && !memberHasAccess)
  return { response: "Collection is not accessible.", status: 401 };
```

即：需要是**所有者**或成员中 `canUpdate=true`（admin 角色）。

**额外限制**（第88-96行）：
- 非所有者不能将链接移动到其他集合
- 非所有者不能将链接从当前集合移走

**此路径也会顺带处理置顶**（第177-181行）：
```typescript
pinnedBy: data?.pinnedBy
  ? data.pinnedBy[0]?.id === userId
    ? { connect: { id: userId } }
    : { disconnect: { id: userId } }
  : undefined,
```

#### 权限矩阵汇总

| 用户身份 | 纯置顶（仅改pinnedBy） | 完整更新（改其他字段） | 移动链接到其他集合 |
|---------|----------------------|---------------------|-----------------|
| 集合所有者 (owner) | ✅ 通过路径二 | ✅ 通过路径二 | ✅ |
| 协作成员 admin (canUpdate=true) | ✅ 通过路径一或二 | ✅ 通过路径二 | ❌ |
| 协作成员 contributor (canCreate=true) | ✅ 通过路径一 | ❌ | ❌ |
| 协作成员 viewer (全false) | ✅ 通过路径一 | ❌ | ❌ |
| 非成员 | ❌ 401 | ❌ 401 | ❌ |

### 3. 前端置顶交互

见 [pinLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/client/pinLink.ts)

```typescript
const pinLink = async (link: LinkIncludingShortenedCollectionAndTags) => {
  const isAlreadyPinned = link?.pinnedBy && link.pinnedBy[0] ? true : false;
  updateLink.mutateAsync({
    ...link,
    pinnedBy: (isAlreadyPinned
      ? [{ id: undefined }]   // 取消置顶
      : [{ id: user?.id }]) as any,  // 置顶
  });
};
```

前端判断置顶状态的工具函数见 [links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/packages/router/links.tsx#L266-L268)：
```typescript
const isLinkPinned = (link?: LinkIncludingShortenedCollectionAndTags) => {
  return Boolean(link?.pinnedBy && link.pinnedBy.length > 0);
};
```

UI 组件见 [LinkPin.tsx](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkPin.tsx)，在卡片右上角显示图钉图标：
- 未置顶：`bi-pin`（空心图标）
- 已置顶：`bi-pin-fill`（实心图标）

### 4. 仅显示置顶链接

见 [pinned.tsx](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/pages/links/pinned.tsx)

```typescript
const { links, data } = useLinks({
  sort: sortBy,
  pinnedOnly: true,  // 只查询当前用户置顶的链接
});
```

后端查询条件见 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L51-L52)

```typescript
const pinnedCondition =
  query.pinnedOnly && userId ? { pinnedBy: { some: { id: userId } } } : {};
```

Meilisearch 搜索中对应的置顶过滤见 [searchQueryBuilder.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/searchQueryBuilder.ts#L148-L158)：
```typescript
case "pinned":
  if (value === "true") {
    filters.push(
      isNegative ? `NOT pinnedBy = ${userId}` : `pinnedBy = ${userId}`
    );
  }
```

---

## 三、排序（Sort）机制深度分析

### 1. 支持的排序方式

见 [global.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/packages/types/global.ts#L87-L92)

```typescript
export enum Sort {
  DateNewestFirst = 0,  // 默认：最新优先
  DateOldestFirst = 1,  // 最旧优先
  NameAZ = 2,           // 名称 A-Z
  NameZA = 3,           // 名称 Z-A
}
```

**关键事实**：只有这 4 种确定性排序，**不存在"按相关性排序"选项**。Sort 枚举中没有定义 Relevance，前端下拉框（SortDropdown.tsx）也没有提供该选项。

### 2. 排序持久化与默认值

排序设置保存在 `localStorage` 中，见 [SortDropdown.tsx](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/components/SortDropdown.tsx#L26-L33)

```typescript
useEffect(() => {
  updateSettings({ sortBy });  // 保存到 localStorage
}, [sortBy]);
```

前端默认值见 [links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/packages/router/links.tsx#L27-L32)：
```typescript
const sort =
  params.sort ??
  (typeof window !== "undefined"
    ? Number(window.localStorage.getItem("sortBy"))
    : 0) ??
  0;  // 默认兜底是 0，即 Sort.DateNewestFirst
```

### 3. 后端排序实现：三层排序架构

排序实际上经历了**三层处理**，其中 Meilisearch 路径有两层排序，PostgreSQL 路径有一层，前端还有可选的客户端排序。

#### 第一层：Meilisearch sort 参数（仅 Meilisearch 路径）

见 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L72-L81)

```typescript
const meiliResp = await meiliClient.index("links").search(meiliQuery, {
  filter: meiliFilters,
  attributesToRetrieve: ["id"],
  limit,
  offset,
  sort:                                            // ← 第一层排序
    query.sort === Sort.DateNewestFirst
      ? ["id:desc"]
      : query.sort === Sort.DateOldestFirst
        ? ["id:asc"]
        : query.sort === Sort.NameAZ
          ? ["name:asc"]
          : query.sort === Sort.NameZA
            ? ["name:desc"]
            : ["id:desc"],   // 所有分支都显式传了 sort，没有 undefined
});
```

**重要结论**：
- **所有 4 种排序方式都显式传了 `sort` 参数**，没有任何分支使用 Meilisearch 的默认相关性排序
- Meilisearch 在显式指定 `sort` 时会完全按该规则排序，其内部的相关性评分机制不会生效
- 索引的可排序字段配置见 [linkIndexing.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/worker/workers/linkIndexing.ts#L47-L49)：
  ```typescript
  await meiliClient.index("links").updateSortableAttributes(["id", "name"]);
  ```

#### 第二层：PostgreSQL ORDER BY（两条路径都有）

见 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L26-L30) 和 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L139)

```typescript
let order: Order = { id: "desc" };
if (query.sort === Sort.DateNewestFirst) order = { id: "desc" };
else if (query.sort === Sort.DateOldestFirst) order = { id: "asc" };
else if (query.sort === Sort.NameAZ) order = { name: "asc" };
else if (query.sort === Sort.NameZA) order = { name: "desc" };

// ... 两处 findMany 调用都使用：
orderBy: order,  // ← 第二层排序
```

**为什么 Meilisearch 路径还要在 PostgreSQL 再次排序？**

这不是 bug，而是必要的——Meilisearch 只返回了匹配的 `id` 列表，PostgreSQL 使用 `WHERE id IN (meiliIds)` 查询时，**IN 子句不保证返回顺序与传入 ID 列表一致**。因此即使 Meilisearch 已经按正确顺序返回了 ID，也必须用 `ORDER BY` 再次确保最终结果顺序正确。

**两次排序规则完全一致**，所以最终顺序是确定的，不存在"覆盖"问题。

#### 第三层：前端客户端排序（可选，useSort hook）

见 [useSort.tsx](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/hooks/useSort.tsx)

```typescript
useEffect(() => {
  const dataArray = [...data];
  if (sortBy === Sort.NameAZ)
    setData(dataArray.sort((a, b) => a.name.localeCompare(b.name)));
  else if (sortBy === Sort.NameZA)
    setData(dataArray.sort((a, b) => b.name.localeCompare(a.name)));
  else if (sortBy === Sort.DateNewestFirst)
    setData(dataArray.sort((a, b) =>
      new Date(b.createdAt as string).getTime() - new Date(a.createdAt as string).getTime()
    ));
  else if (sortBy === Sort.DateOldestFirst)
    setData(dataArray.sort((a, b) =>
      new Date(a.createdAt as string).getTime() - new Date(b.createdAt as string).getTime()
    ));
}, [sortBy, data]);
```

这层排序只对已加载到客户端内存的数据生效，主要用于：
- 多页数据合并后的重新排序
- 本地数据变更后的即时排序更新

**注意**：日期排序使用 `createdAt` 字段（JS Date），而后端使用 `id` 自增字段。由于 ID 自增与创建时间正相关，两者结果理论一致。

### 5. Dashboard 中的排序

见 [getDashboardDataV2.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L154-L159)

```typescript
const merged = [...recentlyAddedLinks, ...pinnedLinks].sort(
  (a, b) => new Date(b.id).getTime() - new Date(a.id).getTime()
);
```

Dashboard 的"最近链接"区域将**最近添加的链接**和**置顶链接**合并后按 ID（时间）倒序排列，然后去重。置顶链接在前端会单独过滤到"置顶链接"区域显示。

---

## 四、协作成员与用户隔离机制

### 1. 链接可见性隔离

**所有链接查询都强制添加权限过滤**，确保用户只能看到自己有权限的链接。

#### 数据库查询层面

见 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L99-L113)

```typescript
AND: [
  {
    collection: {
      OR: [
        { ownerId: userId },                      // 自己拥有的集合
        { members: { some: { userId } } },        // 作为成员加入的集合
      ],
    },
  },
  // ... 其他条件
]
```

#### Meilisearch 搜索层面

见 [searchQueryBuilder.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/searchQueryBuilder.ts#L102-L104)

```typescript
const filters: string[] = publicOnly
  ? ["collectionIsPublic = true"]
  : [`(collectionOwnerId = ${userId}) OR (collectionMemberIds = ${userId})`];
```

索引时预先存储了权限字段，见 [linkIndexing.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/worker/workers/linkIndexing.ts#L133-L143)

```typescript
const docs = links.map((link) => ({
  ...link,
  collectionOwnerId: link.collection.ownerId,
  collectionMemberIds: link.collection.members.map((m) => m.userId),
  collectionIsPublic: link.collection.isPublic,
  collectionName: link.collection.name,
  tags: link.tags.map((t) => t.name),
  pinnedBy: link.pinnedBy.map((p) => p.id),  // 存储所有置顶用户的ID数组
  creationTimestamp: Date.parse(link.createdAt.toISOString()) / 1000,
}));
```

### 2. 标签（Tag）的用户隔离

标签属于集合的所有者（ownerId），但协作成员也能看到关联链接的标签。

见 [getLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/links/getLinks.ts#L46-L66)

```typescript
searchConditions.push({
  tags: {
    some: {
      name: { contains: query.searchQueryString, mode: "insensitive" },
      OR: [
        { ownerId: userId },                              // 自己的标签
        {
          links: {
            some: {
              collection: {
                members: { some: { userId } },            // 协作集合中的标签
              },
            },
          },
        },
      ],
    },
  },
});
```

### 3. 权限检查（操作隔离）

见 [getPermission.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/getPermission.ts)

```typescript
export default async function getPermission({ userId, collectionId, linkId }) {
  if (linkId) {
    const check = await prisma.collection.findFirst({
      where: {
        links: {
          some: {
            id: linkId,
          },
        },
      },
      include: { members: true },
    });

    return check;
  } else if (collectionId) {
    const check = await prisma.collection.findFirst({
      where: {
        id: collectionId,
        OR: [{ ownerId: userId }, { members: { some: { userId } } }],
      },
      include: { members: true },
    });

    return check;
  }
}
```

**注意**：当用 `linkId` 查询时，`getPermission` **不做权限过滤**，只要链接存在就返回所属 collection（含 members）。实际的权限拒绝逻辑在各业务 controller 中完成。

### 4. 操作权限细分

见 [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L72-L101)

```typescript
const memberHasAccess = collectionIsAccessible?.members.some(
  (e) => e.userId === userId && e.canUpdate
);

// 非所有者不能移动链接到其他集合
if (unauthorizedSwitchCollection)
  return { response: "You can't move a link to/from a collection you don't own.", status: 401 };

// 非所有者且无更新权限则不能编辑
if (collectionIsAccessible?.ownerId !== userId && !memberHasAccess)
  return { response: "Collection is not accessible.", status: 401 };
```

**特殊情况**：置顶操作不需要 `canUpdate` 权限，只要是集合成员就可以置顶（见上文置顶机制部分的权限矩阵）。

### 5. 子集合权限继承

见 [getCollectionRootOwnerAndMembers.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/getCollectionRootOwnerAndMembers.ts)

该函数向上遍历集合层级树，合并所有祖先集合的成员权限（使用 OR 逻辑），确保子集合的权限检查包括所有父级的成员。

同时，更新集合时可以选择将成员角色"传播到子集合"，见 [updateCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts#L68-L121)。

---

## 五、搜索高级语法

用户可以通过搜索语法进行精细筛选，见 [searchQueryBuilder.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/searchQueryBuilder.ts#L7-L18)

支持的搜索修饰符：

| 语法 | 说明 | 示例 |
|------|------|------|
| `url:` | 按 URL 搜索 | `url:github.com` |
| `name:` | 按名称搜索 | `name:文档` |
| `description:` | 按描述搜索 | `description:技术` |
| `type:` | 按类型搜索 | `type:pdf` |
| `collection:` | 按集合名搜索 | `collection:工作` |
| `pinned:true/false` | 按置顶状态筛选 | `pinned:true` |
| `public:true` | 仅公开集合 | `public:true` |
| `before:` | 指定日期之前 | `before:2024-01-01` |
| `after:` | 指定日期之后 | `after:2024-01-01` |
| `tag:` | 按标签搜索 | `tag:重要` |
| `!`前缀 | 否定条件 | `!tag:已读` |

这些修饰符在 Meilisearch 路径中全部生效。在 PostgreSQL fallback 路径中，**高级语法不会被解析**，只对 name/url/description/tags 做简单的 `contains` 模糊匹配。

---

## 六、搜索引擎路径 vs 普通查询路径详解

Linkwarden 存在**三条独立的链接查询路径**，分别服务于不同场景。

### 1. 三条路径概览

| 路径 | API 端点 | Controller | 身份认证 | 使用场景 |
|------|---------|------------|---------|---------|
| 路径 A：搜索（推荐） | `GET /api/v1/search` | [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts) | ✅ 需要登录 | 前端所有常规列表查询（useLinks hook 调用） |
| 路径 B：旧版链接（废弃） | `GET /api/v1/links` | [getLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/links/getLinks.ts) | ✅ 需要登录 | 保留兼容，代码中已标记 DEPRECATED |
| 路径 C：公开集合 | `GET /api/v1/public/collections/links` | [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts)（带 `publicOnly=true`） | ❌ 无需登录 | 公开分享的集合页面 |

前端 `useLinks` hook 统一调用路径 A，见 [links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/packages/router/links.tsx#L75-L81)：
```typescript
const url =
  (auth?.instance ? auth?.instance : "") +
  "/api/v1/search?cursor=" +
  params.pageParam + ...
```

### 2. 路径 A（/api/v1/search）的双引擎分支

在 `searchLinks` controller 内部，根据是否启用 Meilisearch 和是否有关键词搜索，又分为两个分支：

#### 分支 A-1：Meilisearch 搜索引擎路径

**触发条件**（第54行）：
```typescript
if (meiliClient && query.searchQueryString)
```
即：Meilisearch 客户端可用 **且** 用户输入了搜索关键词。

**执行流程**：
```
1. 解析搜索关键词中的高级语法（parseSearchTokens）
   ↓
2. 构建 Meilisearch 查询串和过滤器（buildMeiliQuery, buildMeiliFilters）
   - 权限过滤器：(collectionOwnerId = userId) OR (collectionMemberIds = userId)
   - 置顶过滤器：pinnedBy = userId（如果 pinned:true）
   - 其他：url/name/collection/tag/before/after 等
   ↓
3. 调用 Meilisearch 搜索，仅取回匹配的 link.id 列表（attributesToRetrieve: ["id"]）
   - 显式指定 sort 参数（4种排序之一），Meilisearch 不使用相关性排序
   ↓
4. 用 id 列表回查 PostgreSQL 获取完整数据：
   prisma.link.findMany({ where: { id: { in: meiliIds } }, ... })
   - 再次做权限校验（双保险）
   - include: tags, collection, pinnedBy（仅当前用户）
   ↓
5. 排序：PostgreSQL ORDER BY（与 Meilisearch 使用完全相同的排序规则，
   目的是修正 WHERE id IN (...) 不保证返回顺序的问题，不是覆盖）
   ↓
6. 返回结果 + nextCursor（offset + limit 模式）
```

**Meilisearch 索引配置**见 [linkIndexing.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/worker/workers/linkIndexing.ts#L23-L37)：
```typescript
updateFilterableAttributes([
  "collectionOwnerId", "collectionMemberIds", "collectionName",
  "tags", "pinnedBy", "url", "type", "name", "description",
  "collectionIsPublic", "creationTimestamp",
])
```

**索引时存储的置顶数据**（第140行）：
```typescript
pinnedBy: link.pinnedBy.map((p) => p.id),  // 存储所有置顶用户的ID数组
```
Meilisearch 中 `pinnedBy` 字段包含**所有**置顶了该链接的用户 ID，搜索时用 `pinnedBy = ${userId}` 过滤当前用户的置顶。

#### 分支 A-2：PostgreSQL 直接查询（Fallback）

**触发条件**：Meilisearch 不可用，或没有搜索关键词。

**执行流程**：
```
1. 构建 PostgreSQL where 条件
   - 权限过滤：collection.ownerId = userId OR collection.members.some(userId)
   - 关键词搜索：对 name/url/description/tags 做 LIKE contains 模糊匹配
     （注意：不解析高级搜索语法，只是简单 contains）
   - pinnedOnly: pinnedBy.some(id = userId)
   - tagId/collectionId 过滤
   ↓
2. Prisma 直接查询：prisma.link.findMany({ where, include, orderBy })
   - include: tags, collection, pinnedBy（仅当前用户）
   - 分页：cursor-based（take + skip + cursor）
   ↓
3. 返回结果 + nextCursor（基于最后一条记录的 id）
```

### 3. 路径 B（/api/v1/links 旧版）的差异

见 [getLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/links/getLinks.ts)

- **只走 PostgreSQL**，无 Meilisearch 分支
- **标签搜索有额外的权限过滤**：对 tag 的可见性单独做了 ownerId/members 判断
- **分页**：cursor-based，与 A-2 相同
- **返回格式**：`{ response: links }`，而路径 A 是 `{ data: { links, nextCursor } }`
- **已标记废弃**：当 `DISABLE_DEPRECATED_ROUTES=true` 时直接返回 400

### 4. 路径 C（公开集合）的差异

见 [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/pages/api/v1/public/collections/links/index.ts)

- **无用户身份**：不调用 `verifyUser`，直接传入 `publicOnly: true`
- **权限过滤器不同**：
  - PostgreSQL：`collection: { id: query.collectionId, isPublic: true }`
  - Meilisearch：`["collectionIsPublic = true"]`
- **不返回 pinnedBy**：因为没有 userId，`include.pinnedBy` 为 `undefined`
- **单条公开链接查询**见 [getLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/public/links/linkId/getLinkById.ts)，完全不包含 `pinnedBy` 字段

### 5. 筛选条件的"两阶段应用"（Meilisearch 路径的关键问题）

Meilisearch 路径最核心的设计特点是：**筛选条件被分裂到两个阶段应用**，这直接影响了分页数量和翻页行为。

#### 第一阶段：Meilisearch 搜索时应用的筛选

在调用 `meiliClient.index("links").search()` 时（见 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L67-L82)），通过 `buildMeiliFilters` 构建的过滤器（见 [searchQueryBuilder.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/searchQueryBuilder.ts#L93-L207)）只包含：

| 筛选类型 | Meilisearch 阶段是否应用 | 来源 |
|---------|----------------------|------|
| **权限过滤** | ✅ 是 | `(collectionOwnerId = X) OR (collectionMemberIds = X)` |
| **搜索关键词修饰符** | ✅ 是 | 用户在搜索框输入的 `url:` `name:` `description:` `type:` `collection:` `pinned:` `public:` `before:` `after:` `tag:` 等高级语法 |
| **普通搜索词** | ✅ 是 | `buildMeiliQuery` 拼接的全文检索字符串 |
| **query.collectionId** | ❌ 否 | — |
| **query.tagId** | ❌ 否 | — |
| **query.pinnedOnly** | ❌ 否 | — |

**注意**：
- 搜索框输入 `pinned:true` 会在 Meilisearch 阶段过滤（高级语法）
- 但通过 API 参数 `pinnedOnly=true`（如置顶页面）**不会**在 Meilisearch 阶段过滤
- `collectionId`（按集合 ID 过滤）和 `tagId`（按标签 ID 过滤）完全不在 Meilisearch 阶段处理，因为 Meilisearch 索引中根本没有存储 `collectionId` 和 `tagId` 字段——只存了 `collectionName`（集合名称字符串）和 `tags`（标签名称数组）

见 [linkIndexing.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/worker/workers/linkIndexing.ts#L133-L143) 索引时存储的字段：
```typescript
const docs = links.map((link) => ({
  ...link,
  collectionOwnerId: link.collection.ownerId,      // ✅ 有 ownerId
  collectionMemberIds: link.collection.members.map((m) => m.userId),
  collectionName: link.collection.name,            // ⚠️ 只有名称，没有 collectionId
  tags: link.tags.map((t) => t.name),               // ⚠️ 只有标签名数组，没有 tagId
  pinnedBy: link.pinnedBy.map((p) => p.id),
  // ... collectionId、tagId 均未单独存储
}));
```

#### 第二阶段：PostgreSQL 回查时才应用的筛选

Meilisearch 返回匹配的 ID 列表后，在 `prisma.link.findMany` 的 `where` 中（见 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L95-L125)）又追加了以下条件：

```typescript
where: {
  id: { in: meiliIds },  // 限定 Meilisearch 返回的 ID
  AND: [
    // ... 权限过滤（双保险）
    ...collectionCondition,   // ← query.collectionId 在这里！
    {
      OR: [
        ...tagCondition,      // ← query.tagId 在这里！
        { ...pinnedCondition }, // ← query.pinnedOnly 在这里！
      ],
    },
  ],
}
```

---

### 6. 两阶段筛选对分页数量和翻页游标的影响

这是整个设计中**最容易产生问题**的部分。

#### 问题场景演示

假设：
- `paginationTakeCount = 50`
- 用户在集合详情页（`collectionId=123`）中搜索关键词 `"react"`
- Meilisearch 匹配到 1000 条含 `"react"` 的链接（分布在多个集合中）

**执行流程**：

```
第 1 页（offset=0）：
  Meilisearch 搜索 → 返回 50 个 ID（offset=0, limit=50）
     ↓ 这 50 条来自所有集合，不全属于 collectionId=123
  PostgreSQL 回查 + 应用 collectionId=123 过滤
     ↓ 假设只有 12 条真正属于该集合
  返回给前端：12 条链接
  nextCursor = (meiliResp.hits.length === 50) ? 0 + 50 : null = 50
     ↑ 问题：虽然只有 12 条，但只要 Meilisearch 返回了 50 条，就认为有下一页

第 2 页（offset=50）：
  Meilisearch 搜索 → 返回 50 个 ID（offset=50, limit=50）
  PostgreSQL 回查 + collectionId=123 过滤
     ↓ 假设只有 8 条真正属于该集合
  返回给前端：8 条链接
  nextCursor = 100

... 后续若干页可能每页只有几条甚至 0 条
```

#### 问题总结

| 问题 | 说明 | 代码位置 |
|------|------|---------|
| **每页数量不足** | Meilisearch 返回 limit（50）个 ID，但 PostgreSQL 二次过滤后可能远少于 50 条 | [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L95-L140) |
| **nextCursor 判断错误** | `nextCursor` 基于 `meiliResp.hits.length === limit` 判断，而非基于最终返回的 `links.length`。即使回查后只剩 0 条，只要 Meilisearch 返回了 50 条，仍会告知前端"有下一页" | [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L142) |
| **可能出现空页** | 极端情况下，某一页的 50 个 Meilisearch ID 全部不满足 collectionId/tagId/pinnedOnly 条件，导致返回 `links: []`，但 nextCursor 仍有值 | 同上 |
| **用户体验** | 用户可能看到：第一页 12 条、第二页 8 条、第三页 0 条、第四页 15 条… 每页数量不稳定 | — |

#### nextCursor 的计算逻辑对比

见 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L142) vs [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L247-L250)：

```typescript
// Meilisearch 路径（A-1）：基于 Meilisearch 的 hits 数量
const nextCursor = meiliResp.hits.length === limit ? offset + limit : null;

// PostgreSQL 路径（A-2）：基于实际返回的 links 数量
nextCursor:
  links.length === paginationTakeCount
    ? links[links.length - 1].id
    : null,
```

---

### 7. 两条路径的详细对比表（含分页与筛选）

| 对比维度 | Meilisearch 路径 (A-1) | PostgreSQL 路径 (A-2, B) |
|---------|----------------------|-------------------------|
| 触发条件 | meiliClient 可用 + 有关键词 | 其他情况 |
| 搜索能力 | 全文搜索（Meilisearch 倒排索引）+ 所有高级语法 | 简单 LIKE contains，**不支持高级语法** |
| 相关性排序 | ❌ 不存在。显式指定 sort 参数，Meilisearch 相关性不生效 | ❌ 不存在 |
| **权限过滤** | Meilisearch 阶段过滤 + PostgreSQL 回查双保险 | PostgreSQL 一次过滤 |
| **collectionId/tagId/pinnedOnly** | ❌ 仅 PostgreSQL 回查阶段过滤（Meilisearch 阶段不处理） | ✅ 查询时一次过滤 |
| **搜索高级语法（url:、tag: 等）** | ✅ Meilisearch 阶段过滤 | ❌ 不支持 |
| 置顶过滤（高级语法 `pinned:`） | ✅ Meilisearch 阶段过滤 | ❌ 不支持 |
| 置顶过滤（API 参数 `pinnedOnly`） | ❌ 仅 PostgreSQL 回查阶段 | ✅ 查询时一次过滤 |
| 排序机制 | **两层排序，规则一致**：<br>1. Meilisearch sort 参数（确定返回哪些 ID）<br>2. PostgreSQL ORDER BY（修正 IN 子句顺序不确定性） | 直接 PostgreSQL `ORDER BY`（一层排序） |
| **分页模式** | offset/limit（基于数字偏移） | cursor-based（基于最后一条 ID） |
| **nextCursor 判断依据** | `meiliResp.hits.length === limit`（Meilisearch 返回数量） | `links.length === takeCount`（最终返回数量） |
| **每页数量稳定性** | ⚠️ 不稳定，二次过滤后可能远少于 limit | ✅ 稳定，最多返回 takeCount 条 |
| **空页风险** | ⚠️ 有（二次过滤可能全部排除） | ❌ 无 |
| 查询次数 | 2 次（Meilisearch + PostgreSQL 回查） | 1 次 |
| 标签搜索权限 | 无额外过滤（Meilisearch 只存 tag 名称） | 对 tag.ownerId 额外做权限检查 |
| 结果一致性 | 可能短暂不一致（索引有延迟） | 实时一致 |

---

## 七、返回数据结构与用户隔离

### 1. pinnedBy 字段的用户级过滤

这是用户隔离的最关键机制。**所有返回链接的接口都会对 `pinnedBy` 做 where 过滤**，确保只返回当前请求用户的置顶状态。

#### 列表查询（searchLinks.ts）

见 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L132-L137) 和 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L234-L239)：

```typescript
include: {
  tags: true,
  collection: true,
  pinnedBy: userId
    ? {
        where: { id: userId },   // ← Prisma 层过滤
        select: { id: true },
      }
    : undefined,
},
```

#### 单条查询（getLinkById.ts）

见 [getLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/links/linkId/getLinkById.ts#L30-L43)：

```typescript
include: {
  tags: true,
  collection: true,
  pinnedBy: {
    where: { id: userId },
    select: { id: true },
  },
},

// 额外的二次保险（冗余过滤）
if (link?.pinnedBy && link.pinnedBy.length > 0) {
  link.pinnedBy = link.pinnedBy.filter((p) => p.id === userId);
}
```

单条查询做了**双重过滤**：Prisma where + JS filter，确保不会泄露其他用户的置顶信息。

#### 更新操作的返回（updateLinkById.ts）

见 [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L53-L61) 和 [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L183-L192)：

```typescript
include: {
  collection: true,
  // 或 tags: true, collection: true
  pinnedBy: isCollectionOwner
    ? {
        where: { id: userId },
        select: { id: true },
      }
    : undefined,   // ← 非所有者不返回 pinnedBy 字段！
},
```

**重要发现**：更新操作返回时，**只有集合所有者才会返回 `pinnedBy` 字段**，协作成员的更新响应中 `pinnedBy` 为 `undefined`。这是一个额外的隔离措施。

### 2. 不同接口返回的 pinnedBy 对比

| 接口 | 身份 | pinnedBy 返回内容 |
|------|------|------------------|
| 搜索列表 `/api/v1/search` | 已登录用户 | `[{ id: userId }]`（已置顶）或 `[]`（未置顶） |
| 单条链接 `/api/v1/links/:id` | 已登录用户 | `[{ id: userId }]` 或 `[]`（双重过滤） |
| 更新链接 `PUT /api/v1/links/:id` | 集合所有者 | `[{ id: userId }]` 或 `[]` |
| 更新链接 `PUT /api/v1/links/:id` | 协作成员（任何角色） | `undefined`（字段不存在） |
| 公开集合列表 `/api/v1/public/collections/links` | 未登录 | `undefined`（字段不存在） |
| 公开单条链接 `/api/v1/public/links/:id` | 未登录 | `undefined`（字段不存在） |

### 3. 对前端用户隔离的影响

前端判断置顶状态见 [links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/packages/router/links.tsx#L266-L268)：
```typescript
const isLinkPinned = (link) => {
  return Boolean(link?.pinnedBy && link.pinnedBy.length > 0);
};
```

这个判断对三种情况都安全：
- 已置顶 → `pinnedBy = [{ id: 123 }]` → `length > 0` → `true` ✅
- 未置顶 → `pinnedBy = []` → `length = 0` → `false` ✅
- 无权限/公开 → `pinnedBy = undefined` → 可选链返回 `false` ✅

前端置顶计数（Dashboard 页面）见 [links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/packages/router/links.tsx#L239-L244)：
```typescript
const numberOfPinnedLinks = removedLink?.pinnedBy?.length
  ? Math.max(0, (oldData.numberOfPinnedLinks ?? 0) - 1)
  : oldData.numberOfPinnedLinks;
```

### 4. 返回数据中 collection.members 的隔离

虽然查询权限时使用 `include: { members: true }`（见 [getPermission.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/getPermission.ts#L23-L24)），但**返回给前端的链接数据不包含 members 列表**。

前端 Link 类型 `LinkIncludingShortenedCollectionAndTags`（见 [global.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/packages/types/global.ts)）中，collection 只包含：
- `id`, `name`, `ownerId`, `parentId`, `isPublic`, `color`, `icon`

而不包含 `members`，所以前端无法直接查看协作者列表。查看成员需要调用专门的 collection 详情接口。

---

## 八、整体工作流架构图

```
用户 (User)
  │
  ├── 拥有的集合 (Collection.ownerId)
  │     ├── 链接 (Link.collectionId)
  │     │     ├── 置顶关系 (pinnedBy: User[]) ← 用户隔离
  │     │     ├── 标签 (Tag)
  │     │     └── 归档格式 (PDF/截图/可读版等)
  │     └── 成员 (UsersAndCollections) ← 协作权限
  │
  └── 加入的协作集合 (UsersAndCollections.userId)
        ├── 可查看所有链接
        ├── 可独立置顶链接（不影响他人，甚至 viewer 也能置顶）
        └── 根据 canCreate/canUpdate/canDelete 执行操作
```

### 数据查询流程

```
前端请求 (sort, pinnedOnly, collectionId, tagId, searchQueryString)
  │  useLinks hook → GET /api/v1/search
  │  sort 默认 = DateNewestFirst (id desc)，共 4 种排序，无相关性选项
  ▼
API 路由验证用户身份 (verifyUser)
  │
  ▼
┌─ 有搜索词 且 Meilisearch 可用? ──────────────────────┐
│        Yes                                               │        No
▼                                                          ▼
Meilisearch 搜索路径 (两阶段筛选)                    PostgreSQL 直接查询
  │                                                          │
  │ 第一阶段（在 Meilisearch 中）                            │  所有筛选一次性应用：
  │  ✅ 权限过滤 (collectionOwnerId/MemberIds)               │  ✅ 权限过滤
  │  ✅ 搜索高级语法 (url: name: pinned: tag: 等)            │  ✅ collectionId/tagId
  │  ✅ 全文搜索关键词                                       │  ✅ pinnedOnly
  │  ❌ collectionId / tagId (索引中无此字段)               │  ✅ 关键词模糊匹配
  │  ❌ pinnedOnly 参数 (仅高级语法 pinned: 处理)            │  LIKE contains
  │  sort: 显式指定 4 种之一                                  │
  │  (无相关性排序)                                           │
  │  offset/limit 分页 → 取 limit 个 ID                      │
  ▼                                                          │
PostgreSQL 回查完整数据 ◄───────────────────────────────────┘
  │                                                          │
  │ 第二阶段（PostgreSQL 回查时追加过滤）                      │  直接查询：
  │  ✅ collectionId / tagId / pinnedOnly                     │  take + cursor 分页
  │  ✅ 权限过滤（双保险）                                     │  orderBy 排序
  │  include.pinnedBy.where = { id: userId }                  │
  │  orderBy: 与 Meilisearch 相同规则（修正 IN 子句顺序）       │
  │                                                          │
  │ ⚠️  分页问题：                                              │
  │   - Meilisearch 返回 50 个 ID                              │
  │   - 二次过滤后可能只剩 5 条甚至 0 条                        │
  │   - nextCursor = offset+50（按 Meilisearch 数量判断）       │
  │   - 可能出现"有下一页但返回空"的情况                         │
  ▼
返回：{ data: { links: [...], nextCursor } }
  │
  ▼
（可选）前端 useSort hook 客户端再排序
  │  日期用 createdAt（后端用 id），理论一致
  ▼
最终列表展示
```

### 置顶操作流程

```
前端点击图钉图标 (LinkPin.tsx)
  │  pinLink() → useUpdateLink() → PUT /api/v1/links/:id
  ▼
后端 updateLinkById
  │
  ├─ 1. 先检查是否为"纯置顶"请求（canPinPermission && data.pinnedBy[0]）
  │     │  只要是集合成员即可（viewer 也可以）
  │     │  只更新 pinnedBy 字段，connect/disconnect 当前用户
  │     └─ 快速返回（不做后续权限校验）
  │
  └─ 2. 否则走完整更新路径
        │  需要 owner 或 canUpdate=true（admin）
        │  可更新 name/url/tags/collection 等
        │  同时也可更新 pinnedBy
        └─ 非所有者不能移动链接到其他集合
```

---

## 九、关键发现与问题总结

### ✅ 用户隔离做得好的部分

1. **置顶完全用户隔离**：通过多对多关系 `pinnedBy` 实现，每个用户独立管理；返回数据时用 `where: { id: userId }` 严格过滤
2. **查询权限强制过滤**：所有链接查询都加入了集合所有权/成员检查，包括 Meilisearch 路径的预索引过滤
3. **操作权限细分**：canCreate/canUpdate/canDelete 三级权限；置顶操作有独立的快速路径
4. **Meilisearch 索引预存权限**：搜索时快速过滤，避免关联查询；同时 PostgreSQL 回查做了双保险
5. **返回数据脱敏**：非所有者的更新响应不返回 pinnedBy；公开接口完全不返回 pinnedBy

### ⚠️ 设计缺陷与可能混淆的地方

1. **没有"已读/未读"状态**：如果用户期望阅读后工作流（Read Later），只能通过置顶或集合间接实现
2. **没有独立的"收藏"功能**：置顶承担了收藏的角色，但语义上可能不清晰
3. **置顶状态只返回当前用户**：`pinnedBy` 数组中只有当前用户 ID，协作者之间看不到彼此的置顶（这是设计，但用户可能想知道"谁置顶了这个"）
4. **没有阅读历史记录**：无法追踪用户是否打开/阅读过某个链接，也无法基于此排序
5. **排序方式有限**：只有 4 种基础排序（日期/名称），**完全没有"按相关性排序"选项**，也没有"推荐"、"最近阅读"、"最常访问"等
6. **Meilisearch 能力未充分利用**：虽然集成了 Meilisearch，但代码显式指定 sort 参数导致其相关性评分机制完全不生效；用户无法享受搜索引擎的相关性排序优势
7. **前后端排序字段不一致**：后端日期排序使用 `id` 自增字段，前端 useSort hook 使用 `createdAt` 字段；虽然理论一致，但极端情况下（如数据迁移、ID 重置）可能出现差异
8. **Meilisearch 索引的置顶延迟**：索引 worker 是异步的，置顶/取消置顶后立即搜索可能得到旧结果
9. **两条路径搜索能力不对等**：PostgreSQL fallback 不支持高级搜索语法，但前端搜索框不提示这一点
10. **⚠️ Meilisearch 路径筛选分裂导致分页异常（严重）**：
    - `collectionId`、`tagId`、`pinnedOnly` 三个 API 参数只在 PostgreSQL 回查阶段过滤，不在 Meilisearch 阶段过滤
    - 导致 Meilisearch 先取 50 个 ID，二次过滤后可能只剩几条甚至 0 条
    - `nextCursor` 基于 Meilisearch 返回数量判断（`meiliResp.hits.length === limit`），而非最终返回数量，可能产生"有下一页但实际空页"的情况
    - 根本原因：Meilisearch 索引未存储 `collectionId` 和 `tagId` 字段（只存了名称），无法按 ID 过滤
11. **⚠️ 同一筛选的两种入口行为不一致**：
    - 搜索框输入 `pinned:true`（高级语法）→ Meilisearch 阶段过滤，分页正常
    - API 参数 `pinnedOnly=true`（如置顶页面）→ 仅 PostgreSQL 回查阶段过滤，分页异常
    - `tag:` 高级语法（按标签名） vs `tagId` 参数（按标签 ID）同理

### 🔧 潜在改进方向

1. 为 Link 增加 `status` 字段（如 UNREAD, READ, ARCHIVED）实现真正的阅读后状态
2. 增加 `lastOpenedAt` 字段记录用户最后打开时间，支持"最近阅读"排序
3. 区分"收藏"（Favorite）和"置顶"（Pin）两种语义
4. **新增"按相关性排序"选项**：在 Sort 枚举中增加 Relevance 值，当选择该值时 Meilisearch 不传 sort 参数，让其按默认相关性排序；PostgreSQL 路径可使用全文搜索排名（如 PostgreSQL `ts_rank`）
5. 让集合所有者可以查看成员的置顶/收藏情况（协作场景）
6. 统一前后端排序字段：后端日期排序也使用 `createdAt` 字段，避免极端情况下的不一致
7. 在置顶/取消置顶后立即触发 Meilisearch 索引更新，或在应用层做补偿
8. **修复 Meilisearch 分页异常（高优先级）**：
   - 方案 A：在 Meilisearch 索引中新增 `collectionId` 和 `tagId` 字段，并在 `buildMeiliFilters` 中处理 `query.collectionId`、`query.tagId`、`query.pinnedOnly` 参数，让所有筛选在 Meilisearch 阶段一次性完成
   - 方案 B：将 `nextCursor` 改为基于最终 `links.length` 判断；或当二次过滤后数量不足时，循环查询 Meilisearch 下一批 ID 直到凑够 limit 条或无更多结果
   - 方案 C：统一 `pinned:` 高级语法和 `pinnedOnly` 参数的处理路径，避免行为不一致
9. 统一标签和集合的两种过滤入口（按 ID vs 按名称），消除用户体验差异
