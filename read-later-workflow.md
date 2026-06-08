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

### 2. 前端置顶交互

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

UI 组件见 [LinkPin.tsx](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkPin.tsx)，在卡片右上角显示图钉图标：
- 未置顶：`bi-pin`（空心图标）
- 已置顶：`bi-pin-fill`（实心图标）

### 3. 仅显示置顶链接

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

---

## 三、排序（Sort）机制

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

### 2. 排序持久化

排序设置保存在 `localStorage` 中，见 [SortDropdown.tsx](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/components/SortDropdown.tsx#L26-L33)

```typescript
useEffect(() => {
  updateSettings({ sortBy });  // 保存到 localStorage
}, [sortBy]);
```

### 3. 后端排序实现

#### 数据库直接查询（无 Meilisearch 时）

见 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L26-L30)

```typescript
let order: Order = { id: "desc" };
if (query.sort === Sort.DateNewestFirst) order = { id: "desc" };
else if (query.sort === Sort.DateOldestFirst) order = { id: "asc" };
else if (query.sort === Sort.NameAZ) order = { name: "asc" };
else if (query.sort === Sort.NameZA) order = { name: "desc" };
```

**注意**：日期排序使用 `id` 字段，因为 ID 是自增的，与创建时间正相关。

#### Meilisearch 搜索引擎排序

见 [linkIndexing.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/worker/workers/linkIndexing.ts#L47-L49)

```typescript
await meiliClient
  .index("links")
  .updateSortableAttributes(["id", "name"]);
```

搜索时的排序见 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L72-L81)

### 4. 关于"推荐排序"和"历史记录影响"

**重要结论**：当前代码中**不存在**以下功能：

- ❌ 基于用户阅读历史的智能推荐排序
- ❌ 基于点击次数/访问频率的排序
- ❌ "最近阅读"排序（只有"最近添加"）
- ❌ 记录用户打开/阅读链接的历史的机制

打开链接的逻辑（见 [openLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/100-linkwarden/apps/web/lib/client/openLink.ts)）只是简单跳转，没有记录任何行为数据：

```typescript
const openLink = (link, user, openModal) => {
  if (user.linksRouteTo === LinksRouteTo.DETAILS) {
    openModal();
  } else {
    const format = getFormatBasedOnPreference({ link, preference: user.linksRouteTo });
    window.open(
      format !== null ? `/preserved/${link?.id}?format=${format}` : link.url,
      "_blank"
    );
  }
};
```

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
  // ...
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
    return await prisma.collection.findFirst({
      where: { links: { some: { id: linkId } } },
      include: { members: true },
    });
  } else if (collectionId) {
    return await prisma.collection.findFirst({
      where: {
        id: collectionId,
        OR: [{ ownerId: userId }, { members: { some: { userId } } }],
      },
      include: { members: true },
    });
  }
}
```

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

**特殊情况**：置顶操作不需要 `canUpdate` 权限，只要是集合成员就可以置顶（见上文置顶机制部分）。

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

---

## 六、整体工作流架构图

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
        ├── 可独立置顶链接（不影响他人）
        └── 根据 canCreate/canUpdate/canDelete 执行操作
```

### 数据查询流程

```
前端请求 (sort, pinnedOnly, collectionId, searchQueryString)
        │
        ▼
API 路由验证用户身份 (verifyUser)
        │
        ▼
权限过滤 (强制条件)：
  ├── collection.ownerId = userId
  └── OR collection.members 包含 userId
        │
        ▼
应用查询条件：
  ├── pinnedOnly → pinnedBy.some(id = userId)
  ├── collectionId → collection.id = ?
  ├── tagId → tags.some(id = ?)
  └── searchQueryString → 全文搜索 + 高级语法
        │
        ▼
排序 (orderBy):
  ├── id desc/asc (日期)
  └── name asc/desc (名称)
        │
        ▼
返回结果时过滤 pinnedBy：
  pinnedBy.where = { id: userId } ← 只返回当前用户的置顶状态
```

---

## 七、关键发现与问题总结

### ✅ 用户隔离做得好的部分

1. **置顶完全用户隔离**：通过多对多关系 `pinnedBy` 实现，每个用户独立管理
2. **查询权限强制过滤**：所有链接查询都加入了集合所有权/成员检查
3. **操作权限细分**：canCreate/canUpdate/canDelete 三级权限
4. **Meilisearch 索引预存权限**：搜索时快速过滤，避免关联查询

### ⚠️ 用户隔离不明显/可能混淆的地方

1. **没有"已读/未读"状态**：如果用户期望阅读后工作流（Read Later），只能通过置顶或集合间接实现
2. **没有独立的"收藏"功能**：置顶承担了收藏的角色，但语义上可能不清晰
3. **置顶状态只返回当前用户**：`pinnedBy` 数组中只有当前用户 ID，协作者之间看不到彼此的置顶（这是设计，但用户可能想知道"谁置顶了这个"）
4. **没有阅读历史记录**：无法追踪用户是否打开/阅读过某个链接，也无法基于此排序
5. **排序方式有限**：只有 4 种基础排序，没有"推荐"、"最近阅读"、"最常访问"等

### 🔧 潜在改进方向

1. 为 Link 增加 `status` 字段（如 UNREAD, READ, ARCHIVED）实现真正的阅读后状态
2. 增加 `lastOpenedAt` 字段记录用户最后打开时间，支持"最近阅读"排序
3. 区分"收藏"（Favorite）和"置顶"（Pin）两种语义
4. 增加基于阅读行为的推荐排序算法
5. 让集合所有者可以查看成员的置顶/收藏情况（协作场景）
