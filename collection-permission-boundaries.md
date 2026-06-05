# 集合权限边界在多人协作下的生效机制

## 一、成员关系数据结构

### 1.1 核心模型定义

**Collection 模型** ([schema.prisma](./packages/prisma/schema.prisma#L126-L149)):

```prisma
model Collection {
  id               Int                   @id @default(autoincrement())
  name             String
  owner            User                  @relation(fields: [ownerId], references: [id], onDelete: Cascade)
  ownerId          Int
  members          UsersAndCollections[]
  parentId         Int?
  parent           Collection?           @relation("SubCollections", fields: [parentId], references: [id], onDelete: Cascade)
  subCollections   Collection[]          @relation("SubCollections")
  isPublic         Boolean               @default(false)
  // ...
}
```

**UsersAndCollections 联结表** ([schema.prisma](./packages/prisma/schema.prisma#L151-L164)):

```prisma
model UsersAndCollections {
  user         User       @relation(fields: [userId], references: [id], onDelete: Cascade)
  userId       Int
  collection   Collection @relation(fields: [collectionId], references: [id], onDelete: Cascade)
  collectionId Int
  canCreate    Boolean
  canUpdate    Boolean
  canDelete    Boolean
  createdAt    DateTime   @default(now())
  updatedAt    DateTime   @default(now()) @updatedAt

  @@id([userId, collectionId])  // 复合主键：一个用户在一个集合中只有一条记录
}
```

### 1.2 Member 类型定义

**Member 接口** ([global.ts](./packages/types/global.ts#L36-L43)):

```typescript
export interface Member {
  collectionId?: number;
  userId: number;
  canCreate: boolean;
  canUpdate: boolean;
  canDelete: boolean;
  user: OptionalExcluding<User, "username" | "name" | "id">;
}
```

### 1.3 角色与权限映射

在前端 [EditCollectionSharingModal.tsx](./apps/web/components/ModalContent/EditCollectionSharingModal.tsx#L248-L253) 中，三个布尔字段组合映射为三种角色：

| 角色 (Role) | canCreate | canUpdate | canDelete | 说明 |
|------------|-----------|-----------|-----------|------|
| **Viewer** (查看者) | `false` | `false` | `false` | 只能查看链接，不能修改 |
| **Contributor** (贡献者) | `true` | `false` | `false` | 可以创建链接，不能修改/删除 |
| **Admin** (管理员) | `true` | `true` | `true` | 可以创建、修改、删除链接 |
| **Owner** (所有者) | - | - | - | 通过 `ownerId` 判断，拥有所有权限 |

> **关键注意**: Owner 不存储在 `UsersAndCollections` 表中，而是通过 `Collection.ownerId` 字段直接判定。Owner 天然拥有所有权限，无需在 members 表中添加记录。

---

## 二、权限判定核心流程

### 2.1 服务端权限查询核心函数

**getPermission** ([getPermission.ts](./apps/web/lib/api/getPermission.ts#L1-L38)):

```typescript
export default async function getPermission({ userId, collectionId, linkId }: Props) {
  if (linkId) {
    // 通过 linkId 反向查询所属集合
    return await prisma.collection.findFirst({
      where: { links: { some: { id: linkId } } },
      include: { members: true },
    });
  } else if (collectionId) {
    // 直接查询集合，同时验证用户是否有权访问
    return await prisma.collection.findFirst({
      where: {
        id: collectionId,
        // 关键：用户必须是所有者 OR 成员之一
        OR: [{ ownerId: userId }, { members: { some: { userId } } }],
      },
      include: { members: true },
    });
  }
}
```

**权限判定的两个关键点**:
1. **查询即过滤（仅 collectionId 分支）**: 当传入 `collectionId` 时，`WHERE` 条件本身就做了权限过滤，如果用户没有权限，返回 `null`
2. **查询不过滤（linkId 分支）**: 当传入 `linkId` 时，仅通过 linkId 反向查找所属集合，**不做用户权限过滤**，权限检查在调用方后续逻辑中进行
3. **返回值包含 members**: 后续逻辑可以从 `members` 数组中精确匹配用户的具体权限

> **重要区别**: `getPermission` 函数的两个分支行为不一致：
> - `collectionId` 分支: 查询时就过滤，无权限返回 `null`
> - `linkId` 分支: 查询时不过滤，总能返回集合（只要 linkId 存在），需要调用方额外检查权限

### 2.2 权限判定通用模式

所有 API 控制器都遵循以下模式，但根据入参类型有所不同：

**当入参为 collectionId 时**:
```
1. 调用 getPermission(userId, collectionId)
2. 检查返回值是否为 null → 无权限，返回 401（getPermission 已做过滤）
3. 检查是否为 Owner: collectionIsAccessible?.ownerId === userId
4. 检查成员具体权限: collectionIsAccessible?.members.some(e => e.userId === userId && e.canXXX)
5. 两者满足其一即可执行操作
```

**当入参为 linkId 时（关键差异）**:
```
1. 调用 getPermission(userId, linkId) → 注意：此步不做权限过滤，只返回所属集合
2. 检查集合是否存在（返回值是否为 null）→ linkId 不存在，返回 401
3. 检查是否为 Owner: collectionIsAccessible?.ownerId === userId
4. 检查成员权限: collectionIsAccessible?.members.some(e => e.userId === userId)
5. 非 Owner 且非成员 → 返回 401 "Collection is not accessible."
6. 若是 Owner 或成员，继续检查具体操作权限（canCreate/canUpdate/canDelete）
```

> **注意**: 通过 linkId 访问时，`getPermission` 不做权限过滤，权限检查完全由调用方负责。这意味着即使是无权限用户，也能通过 linkId 获取到集合的基本信息（ownerId、members 等），但后续操作会被拦截。

---

## 三、各操作的权限要求

### 3.1 集合 (Collection) 操作权限矩阵

| 操作 | Owner | Admin | Contributor | Viewer | 代码位置 |
|------|-------|-------|-------------|--------|----------|
| **查看集合** | ✅ | ✅ | ✅ | ✅ | [getCollectionById.ts](./apps/web/lib/api/controllers/collections/collectionId/getCollectionById.ts#L7-L13) |
| **创建集合(根级)** | ✅ | ❌ | ❌ | ❌ | [postCollection.ts](./apps/web/lib/api/controllers/collections/postCollection.ts#L84-L120) |
| **创建子集合** | ✅ | ✅ (需三个权限全有) | ❌ | ❌ | [postCollection.ts](./apps/web/lib/api/controllers/collections/postCollection.ts#L49-L52) |
| **更新集合(含成员管理)** | ✅ | ❌ | ❌ | ❌ | [updateCollectionById.ts](./apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts#L34) |
| **删除集合(彻底删除)** | ✅ | ❌ | ❌ | ❌ | [deleteCollectionById.ts](./apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L50-L52) |
| **退出集合(仅移除自身成员关系)** | ❌ | ✅ | ✅ | ✅ | [deleteCollectionById.ts](./apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L23-L49) |

### 3.2 链接 (Link) 操作权限矩阵

| 操作 | Owner | Admin | Contributor | Viewer | 代码位置 |
|------|-------|-------|-------------|--------|----------|
| **查看链接** | ✅ | ✅ | ✅ | ✅ | [getLinkById.ts](./apps/web/lib/api/controllers/links/linkId/getLinkById.ts#L20-L24) |
| **创建链接** | ✅ | ✅ | ✅ | ❌ | [setCollection.ts](./apps/web/lib/api/setCollection.ts#L30-L32) |
| **更新链接** | ✅ | ✅ | ❌ | ❌ | [updateLinkById.ts](./apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L72-L74) |
| **删除链接** | ✅ | ✅ | ❌ | ❌ | [deleteLinkById.ts](./apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts#L12-L14) |
| **移动链接到其他集合** | ✅ | ❌ | ❌ | ❌ | [updateLinkById.ts](./apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L88-L96) |
| **Pin 链接到个人仪表板** | ✅ | ✅ | ✅ | ✅ | [updateLinkById.ts](./apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L36-L65) |
| **批量删除链接** | ✅ | ✅ | ❌ | ❌ | [deleteLinksById.ts](./apps/web/lib/api/controllers/links/bulk/deleteLinksById.ts#L24-L26) |

### 3.3 标签 (Tag) 操作权限矩阵

| 操作 | Owner | Admin | Contributor | Viewer | 代码位置 |
|------|-------|-------|-------------|--------|----------|
| **删除标签** | ✅ | ❌ | ❌ | ❌ | [deleteTagById.ts](./apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts#L13-L17) |

> **重要**: 标签的所有权独立于集合，标签有自己的 `ownerId`，只有标签的所有者才能删除标签。这是一个独立的权限边界。

---

## 四、资源隔离机制

### 4.1 集合列表的资源隔离

**getCollections** ([getCollections.ts](./apps/web/lib/api/controllers/collections/getCollections.ts#L4-L10)):

```typescript
await prisma.collection.findMany({
  where: {
    OR: [
      { ownerId: userId },        // 自己是所有者
      { members: { some: { user: { id: userId } } } },  // 自己是成员
    ],
  },
  // ...
});
```

**隔离效果**: 用户只能看到：
1. 自己创建的所有集合（作为 Owner）
2. 被其他用户添加为成员的集合（作为 Viewer/Contributor/Admin）

### 4.2 单个集合的资源隔离

**getCollectionById** ([getCollectionById.ts](./apps/web/lib/api/controllers/collections/collectionId/getCollectionById.ts#L7-L14)):

```typescript
await prisma.collection.findFirst({
  where: {
    id: collectionId,
    OR: [
      { ownerId: userId },
      { members: { some: { user: { id: userId } } } },
    ],
  },
  // ...
});
```

**隔离效果**: 即使知道集合 ID，如果不是 Owner 也不是成员，查询返回 `null`。

### 4.3 链接级别的资源隔离

#### 4.3.1 单个链接查询 (getLinkById)

**getLinkById** ([getLinkById.ts](./apps/web/lib/api/controllers/links/linkId/getLinkById.ts#L12-L24)):

1. 通过 `linkId` 调用 `getPermission` 获取所属集合（此步不做权限过滤）
2. 验证用户对该集合的访问权限（Owner 或 Member）
3. 权限验证通过后才返回链接数据

**pinnedBy 字段隔离**: 查询时就通过 `where: { id: userId }` 过滤，只返回当前用户的 pin 记录，查询后再次做防御性过滤 ([getLinkById.ts](./apps/web/lib/api/controllers/links/linkId/getLinkById.ts#L33-L43)):

```typescript
// 查询时就过滤
pinnedBy: {
  where: { id: userId },
  select: { id: true },
},

// 查询后再次防御性过滤
if (link?.pinnedBy && link.pinnedBy.length > 0) {
  link.pinnedBy = link.pinnedBy.filter((p) => p.id === userId);
}
```

#### 4.3.2 链接列表查询 (getLinks)

**getLinks** ([getLinks.ts](./apps/web/lib/api/controllers/links/getLinks.ts#L93-L110)):

在查询链接列表时，通过嵌套查询条件做权限过滤：

```typescript
where: {
  AND: [
    {
      collection: {
        OR: [
          { ownerId: userId },                    // 是集合所有者
          { members: { some: { userId } } },      // 是集合成员
        ],
      },
    },
    // ... 其他查询条件
  ],
}
```

**隔离效果**: 用户只能看到自己有权访问的集合中的链接。

#### 4.3.3 搜索查询 (searchLinks)

**searchLinks** ([searchLinks.ts](./apps/web/lib/api/controllers/search/searchLinks.ts#L99-L113)):

无论是 Meilisearch 搜索结果回填，还是直接数据库查询，都会应用相同的权限过滤条件：

```typescript
AND: [
  ...(userId
    ? [
        {
          collection: {
            OR: [
              { ownerId: userId },
              { members: { some: { userId } } },
            ],
          },
        },
      ]
    : []),
  // ...
]
```

#### 4.3.4 公开集合 (isPublic = true)

公开集合有独立的 API 路由，无需登录即可访问：

**getPublicCollection** ([getPublicCollection.ts](./apps/web/lib/api/controllers/public/collections/getPublicCollection.ts#L4-L8)):
```typescript
where: {
  id,
  isPublic: true,  // 只查询公开集合
}
```

**getLinkById (公开)** ([getLinkById.ts](./apps/web/lib/api/controllers/public/links/linkId/getLinkById.ts#L10-L16)):
```typescript
where: {
  id: linkId,
  collection: {
    isPublic: true,  // 所属集合必须是公开的
  },
}
```

> **注意**: 公开集合不验证用户身份，任何人都可以查看。但修改操作（创建/更新/删除）仍需登录并验证权限。

---

### 4.4 不同角色的访问结果对比

所有角色（Owner/Admin/Contributor/Viewer）在**查看**资源时，能看到的内容是完全相同的，没有字段级别的访问控制：

| 资源 | Owner | Admin | Contributor | Viewer | 说明 |
|------|-------|-------|-------------|--------|------|
| 集合基本信息 (name, description, color, icon) | ✅ 完整 | ✅ 完整 | ✅ 完整 | ✅ 完整 | 所有成员看到相同的集合信息 |
| 成员列表 (members) | ✅ 完整 | ✅ 完整 | ✅ 完整 | ✅ 完整 | 所有成员都能看到其他成员及其角色 |
| 链接列表 (links) | ✅ 完整 | ✅ 完整 | ✅ 完整 | ✅ 完整 | 所有成员看到相同的链接列表 |
| 链接详情 (name, url, description, tags) | ✅ 完整 | ✅ 完整 | ✅ 完整 | ✅ 完整 | 所有成员看到相同的链接内容 |
| pinnedBy 字段 | ✅ 仅自己 | ✅ 仅自己 | ✅ 仅自己 | ✅ 仅自己 | 只能看到自己的 pin 状态，看不到其他人的 |
| Pin/Unpin 链接 | ✅ 允许 | ✅ 允许 | ✅ 允许 | ✅ 允许 | 所有成员都可以 Pin 链接到个人仪表板 |
| 退出集合 | ❌ 不能 | ✅ 允许 | ✅ 允许 | ✅ 允许 | 成员可以主动退出集合，Owner 不能退出 |
| 创建链接按钮 | ✅ 显示 | ✅ 显示 | ✅ 显示 | ❌ 隐藏 | 前端根据 canCreate 权限控制显示 |
| 编辑/删除链接按钮 | ✅ 显示 | ✅ 显示 | ❌ 隐藏 | ❌ 隐藏 | 前端根据 canUpdate/canDelete 权限控制显示 |
| 成员管理功能 | ✅ 显示 | ❌ 隐藏 | ❌ 隐藏 | ❌ 隐藏 | 仅 Owner 可见 |
| 公开集合设置 | ✅ 显示 | ❌ 隐藏 | ❌ 隐藏 | ❌ 隐藏 | 仅 Owner 可见 |

> **核心设计**: 权限隔离主要体现在**操作能力**上，而非**数据可见性**上。一旦成为集合成员（任何角色），就能看到该集合的所有内容，区别仅在于能执行什么操作。
>
> **Viewer 特别说明**: Viewer 在前端 `usePermissions` Hook 中被视为 `undefined`（无操作权限），因此所有修改按钮都会隐藏。但在服务端，Viewer 作为合法成员，仍然可以正常读取所有数据、Pin 链接、退出集合。详见 [6.1.1 Viewer 角色的特殊处理](#611-viewer-角色的特殊处理前端-vs-服务端差异)。

---

## 五、子集合的权限继承与传播

### 5.1 权限向上追溯机制

**getCollectionRootOwnerAndMembers** ([getCollectionRootOwnerAndMembers.ts](./apps/web/lib/api/getCollectionRootOwnerAndMembers.ts#L18-L73)):

```typescript
export default async function getCollectionRootOwnerAndMembers(parentId: number) {
  let currentId: number | null = parentId;
  let rootOwnerId: number | null = null;
  const userMap = new Map<number, MemberPerms>();

  while (currentId) {
    const col = await prisma.collection.findUnique({
      where: { id: currentId },
      select: { parentId: true, ownerId: true, members: { ... } },
    });

    rootOwnerId = col.ownerId;
    
    // 合并所有者权限
    addUser({ userId: col.ownerId, canCreate: true, canUpdate: true, canDelete: true });
    
    // 合并成员权限（取并集）
    for (const m of col.members) {
      addUser(m);
    }

    currentId = col.parentId ?? null;  // 继续向上追溯
  }

  return { rootOwnerId, members: Array.from(userMap.values()) };
}
```

**权限合并规则** ([getCollectionRootOwnerAndMembers.ts](./apps/web/lib/api/getCollectionRootOwnerAndMembers.ts#L9-L16)):

```typescript
function mergePerms(a: MemberPerms, b: MemberPerms): MemberPerms {
  return {
    userId: a.userId,
    canCreate: a.canCreate || b.canCreate,  // 逻辑 OR，取最宽松
    canUpdate: a.canUpdate || b.canUpdate,
    canDelete: a.canDelete || b.canDelete,
  };
}
```

### 5.2 创建子集合时的权限继承

在 [postCollection.ts](./apps/web/lib/api/controllers/collections/postCollection.ts#L61-L120) 中：

1. 调用 `getCollectionRootOwnerAndMembers(parentId)` 获取根所有者和所有合并后的成员权限（结果存入 `dedupedUsers`）
2. 新子集合的 `ownerId` 设置为根所有者的 ID（不是创建者的 ID）
3. **重要：`dedupedUsers` 仅用于权限检查，不会自动写入子集合的 members 表！**
4. 只有创建者自己会被加入成员表（当创建者不是根所有者时），身份为 Admin

```typescript
const result = await getCollectionRootOwnerAndMembers(collection.parentId);
rootOwnerId = result.rootOwnerId;  // 子集合的 owner 是根集合的 owner
dedupedUsers = result.members;     // 仅用于权限检查，不写入数据库

// 检查创建者是否已在 dedupedUsers 中（仅用于判断，不影响写入）
const exists = dedupedUsers.some((u) => u.userId === userId);
if (!exists) {
  dedupedUsers.push({
    userId,
    canCreate: true,
    canUpdate: true,
    canDelete: true,
  });
}

// 新集合的 owner 是 rootOwnerId，不是创建者 userId
const newCollection = await prisma.collection.create({
  data: {
    owner: { connect: { id: rootOwnerId } },  // 关键点：继承根所有者
    createdBy: { connect: { id: userId } },   // 记录创建者
    // 注意：这里只写入了创建者自己，没有写入 dedupedUsers 中的其他成员！
    members:
      userId !== rootOwnerId
        ? {
            create: [
              {
                userId,
                canCreate: true,
                canUpdate: true,
                canDelete: true,
              },
            ],
          }
        : undefined,
    parent: collection.parentId
      ? { connect: { id: collection.parentId } }
      : undefined,
  },
  // ...
});
```

> **关键修正**: 创建子集合时，**不会自动继承父集合的成员列表**！只有根 Owner 和创建者（如果不是 Owner）对子集合有权限。其他父集合成员需要 Owner 通过更新集合并开启 `propagateToSubcollections` 才能获得子集合权限。

#### 创建子集合后的成员权限示例

假设场景：
- 根集合 Collection 1: Owner = A, Members = [B(Admin), C(Viewer)]
- 用户 B 在 Collection 1 下创建子集合 Collection 2

创建后 Collection 2 的实际权限：
- Owner = A（继承自根集合）
- Members = [B(Admin)]（只有创建者 B 被自动加入）
- C 对 Collection 2 **没有权限**，需要通过 propagateToSubcollections 同步权限

### 5.3 更新集合时的权限向下传播

在 [updateCollectionById.ts](./apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts#L68-L121) 中：

如果 `propagateToSubcollections` 为 `true`:

1. BFS 遍历所有子集合
2. 删除子集合现有的所有成员关系
3. 过滤掉子集合的 Owner（不能将 Owner 自己加入 members 表）
4. 将新的成员权限批量写入所有子集合

```typescript
if (data.propagateToSubcollections) {
  const subCollections = await getAllSubCollections(collectionId);
  
  for (const sub of subCollections) {
    await prisma.usersAndCollections.deleteMany({ where: { collectionId: sub.id } });
    
    const subMembers = uniqueMembers.filter(m => m.userId !== sub.ownerId);
    
    if (subMembers.length > 0) {
      await prisma.usersAndCollections.createMany({
        data: subMembers.map(e => ({
          userId: e.userId,
          collectionId: sub.id,
          canCreate: e.canCreate,
          canUpdate: e.canUpdate,
          canDelete: e.canDelete,
        })),
      });
    }
  }
}
```

---

## 六、前端权限检查

### 6.1 usePermissions Hook

**usePermissions** ([usePermissions.tsx](./apps/web/hooks/usePermissions.tsx#L1-L32)):

```typescript
export default function usePermissions(collectionId: number) {
  const { data: collections = [] } = useCollections();
  const { data: user } = useUser();
  const [permissions, setPermissions] = useState<Member | true>();

  useEffect(() => {
    const collection = collections.find(e => e.id === collectionId);
    if (collection) {
      let getPermission = collection.members.find(e => e.userId === user?.id);
      
      // 三个权限全为 false 等同于没有权限
      if (getPermission?.canCreate === false && 
          getPermission?.canUpdate === false && 
          getPermission?.canDelete === false)
        getPermission = undefined;

      // Owner 返回 true，Member 返回 Member 对象，无权限返回 undefined
      setPermissions(user?.id === collection.ownerId || getPermission);
    }
  }, [user, collections, collectionId]);

  return permissions;
}
```

**返回值语义**:
- `true`: 用户是 Owner，拥有所有权限
- `Member` 对象: 用户是成员（Admin 或 Contributor），可从中读取具体的 `canCreate/canUpdate/canDelete`
- `undefined`: 用户无权限 **或为 Viewer 角色**

---

### 6.1.1 Viewer 角色的特殊处理（前端 vs 服务端差异）

**前端 Hook 中的 Viewer 处理** ([usePermissions.tsx](./apps/web/hooks/usePermissions.tsx#L20-L25)):

```typescript
// 三个权限全为 false 等同于没有权限
if (getPermission?.canCreate === false && 
    getPermission?.canUpdate === false && 
    getPermission?.canDelete === false)
  getPermission = undefined;

// Owner 返回 true，Member 返回 Member 对象，无权限/Viewer 返回 undefined
setPermissions(user?.id === collection.ownerId || getPermission);
```

**关键差异说明**:

| 层面 | Viewer 角色的权限判定 | 行为 |
|------|----------------------|------|
| **前端 Hook** | 三个权限全为 `false` → 视为 `undefined`（无操作权限） | 所有操作按钮隐藏，无法发起修改请求 |
| **服务端 API** | 只要是成员（哪怕三个权限全为 `false`）→ 允许读取 | 可以正常获取集合列表、集合详情、链接列表、链接详情 |

**具体差异表现**:

1. **集合列表查询** ([getCollections.ts](./apps/web/lib/api/controllers/collections/getCollections.ts#L4-L10)):
   ```typescript
   WHERE: {
     OR: [
       { ownerId: userId },
       { members: { some: { user: { id: userId } } } },  // Viewer 满足此条件
     ],
   }
   ```
   ✅ Viewer 可以看到集合出现在列表中

2. **集合详情查询** ([getCollectionById.ts](./apps/web/lib/api/controllers/collections/collectionId/getCollectionById.ts#L7-L14)):
   ```typescript
   WHERE: {
     id: collectionId,
     OR: [
       { ownerId: userId },
       { members: { some: { user: { id: userId } } } },  // Viewer 满足此条件
     ],
   }
   ```
   ✅ Viewer 可以看到集合详情（包括成员列表）

3. **链接列表查询** ([getLinks.ts](./apps/web/lib/api/controllers/links/getLinks.ts#L93-L110)):
   ```typescript
   WHERE: {
     AND: [
       {
         collection: {
           OR: [
             { ownerId: userId },
             { members: { some: { userId } } },  // Viewer 满足此条件
           ],
         },
       },
       // ...
     ],
   }
   ```
   ✅ Viewer 可以看到该集合下的所有链接

4. **链接详情查询** ([getLinkById.ts](./apps/web/lib/api/controllers/links/linkId/getLinkById.ts#L14-L24)):
   ```typescript
   const memberHasAccess = collectionIsAccessible?.members.some(
     (e: UsersAndCollections) => e.userId === userId  // 只检查是否为成员，不检查具体权限
   );
   
   if (collectionIsAccessible?.ownerId !== userId && !memberHasAccess)
     return { response: "Collection is not accessible.", status: 401 };
   ```
   ✅ Viewer 可以看到链接详情（包括名称、URL、描述、标签等）

5. **Pin 链接操作** ([updateLinkById.ts](./apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L36-L65)):
   ```typescript
   const canPinPermission = collectionIsAccessible?.members.some(
     (e: UsersAndCollections) => e.userId === userId  // 只检查是否为成员
   );
   ```
   ✅ Viewer 可以 Pin/Unpin 链接到个人仪表板

6. **退出集合操作** ([deleteCollectionById.ts](./apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L19-L49)):
   ```typescript
   const memberHasAccess = collectionIsAccessible?.members.some(
     (e: UsersAndCollections) => e.userId === userId  // 只检查是否为成员
   );
   
   if (collectionIsAccessible?.ownerId !== userId && memberHasAccess) {
     // 移除成员关系（退出集合）
   }
   ```
   ✅ Viewer 可以退出集合

> **核心设计意图**:
> - 前端 `usePermissions` Hook 主要用于**控制 UI 元素的显示/隐藏**，Viewer 没有任何修改权限，所以被视为 `undefined`（无操作权限）
> - 服务端 API 区分**读取操作**和**修改操作**：
>   - 读取操作：只要是成员即可（Owner 或 Member，不区分角色）
>   - 修改操作：需要检查具体的 `canCreate/canUpdate/canDelete` 权限
> - 这种设计避免了在前端显示 Viewer 无法使用的按钮，同时保证了 Viewer 的正常浏览体验

---

### 6.2 useCollectivePermissions Hook

**useCollectivePermissions** ([useCollectivePermissions.ts](./apps/web/hooks/useCollectivePermissions.ts#L1-L34)):

与 `usePermissions` 逻辑相同，但支持传入多个 `collectionId`，遍历检查用户对所有集合的权限。

### 6.3 前端权限使用示例

在 [EditCollectionSharingModal.tsx](./apps/web/components/ModalContent/EditCollectionSharingModal.tsx#L79) 中:

```typescript
const permissions = usePermissions(collection.id as number);

// 只有 Owner 才能看到/操作成员管理和公开设置
{permissions === true && !isPublicRoute && (
  <>
    <p>{t("members")}</p>
    {/* 添加成员 UI */}
    {/* 成员角色编辑 UI */}
    {/* 移除成员按钮 */}
  </>
)}

// 非 Owner 只能查看角色，不能修改
{permissions === true && !isPublicRoute ? (
  <DropdownMenu> {/* 可编辑的角色下拉 */} </DropdownMenu>
) : (
  <p className="text-sm text-neutral">{t(roleKey)}</p>  // 只读显示
)}
```

---

## 七、访问结果与错误处理

### 7.1 统一的访问拒绝响应

所有权限检查失败的 API 都返回统一格式：

| 场景 | HTTP 状态码 | 响应消息 |
|------|------------|----------|
| 无权限访问集合/链接 | 401 | `"Collection is not accessible."` |
| 无权限创建子集合 | 403 | `"You are not authorized to create a sub-collection here."` |
| 无权限移动链接 | 401 | `"You can't move a link to/from a collection you don't own."` |
| 目标集合不匹配 | 401 | `"Target collection does not match the data."` |
| 无权限删除标签 | 401 | `"Permission denied."` |

### 7.2 成员退出集合的特殊处理

当非 Owner 用户调用删除集合 API 时 ([deleteCollectionById.ts](./apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L23-L49)):

```typescript
if (collectionIsAccessible?.ownerId !== userId && memberHasAccess) {
  // 不是删除集合，而是删除用户与集合的关联关系
  const deletedUsersAndCollectionsRelation = 
    await prisma.usersAndCollections.delete({
      where: {
        userId_collectionId: { userId, collectionId },
      },
    });
  
  // 同时从用户的集合排序和仪表板中移除
  await Promise.all([
    removeFromOrders(userId, collectionId),
    updateDashboardSectionLayout(userId, collectionId),
  ]);
  
  return { response: deletedUsersAndCollectionsRelation, status: 200 };
}
```

---

## 八、权限边界总结图

### 8.1 权限判定总流程

```
┌──────────────────────────────────────────────────────────────────────┐
│                      权限判定总流程                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  入参类型分支:                                                       │
│  ├─ collectionId 分支:                                               │
│  │  1. getPermission(userId, collectionId)                           │
│  │     └─ WHERE: ownerId = userId OR members.some(userId)            │
│  │     └─ 无权限 → 返回 null                                          │
│  │                                                                   │
│  └─ linkId 分支 (关键差异):                                          │
│     1. getPermission(userId, linkId)                                 │
│        └─ WHERE: links.some.id = linkId  (无用户权限过滤!)           │
│        └─ 只要 linkId 存在就返回集合                                  │
│     2. 调用方额外检查: collection.ownerId === userId OR              │
│                        collection.members.some(userId)                │
│        └─ 无权限 → 返回 401 "Collection is not accessible."          │
│                                                                      │
│  后续统一流程:                                                       │
│  3. 检查是否为 Owner: collection.ownerId === userId                  │
│     └─ 是 → 拥有所有权限，跳过成员检查                                │
│     └─ 否 → 进入成员权限检查                                          │
│                                                                      │
│  4. 成员权限检查: collection.members.some(...)                       │
│     ├─ 查看操作/Pin: 只要是成员即可 (任意角色)                        │
│     ├─ 创建操作: 需要 canCreate = true                               │
│     ├─ 更新操作: 需要 canUpdate = true                               │
│     ├─ 删除操作: 需要 canDelete = true                               │
│     └─ 创建子集合: 需要三个权限全有 (canCreate && canUpdate && canDelete) │
│                                                                      │
│  5. 权限通过 → 执行业务逻辑                                           │
│  6. 权限拒绝 → 返回 401/403                                          │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 8.2 子集合权限流向（已修正）

```
┌──────────────────────────────────────────────────────────────────────┐
│                      子集合权限流向                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  创建时 (权限不自动继承!):                                            │
│    向上追溯 → 仅用于确定 rootOwnerId 和权限检查                        │
│    新子集合成员 = [创建者(Admin)] (如果创建者不是 rootOwner)          │
│    其他父集合成员 → 无权限，需手动 propagate                           │
│                                                                      │
│  更新时 (propagateToSubcollections):                                 │
│    向下传播 → 删除所有子集合现有成员 → 批量写入新成员                  │
│                                                                      │
│  示例场景:                                                            │
│    根集合 (Owner: A, Members: [B(Admin), C(Viewer)])                 │
│         │                                                             │
│         ▼ B 创建子集合                                               │
│    子集合 (Owner: A, Members: [B(Admin)])  ← C 没有权限!             │
│         │                                                             │
│         ▼ A 更新集合并开启 propagateToSubcollections                 │
│    子集合 (Owner: A, Members: [B(Admin), C(Viewer)]) ← 同步权限       │
│         │                                                             │
│         ▼ 继续 propagate                                             │
│    孙集合 (Owner: A, Members: [B(Admin), C(Viewer)])                 │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 8.3 不同角色访问结果对比

```
┌──────────────────────────────────────────────────────────────────────┐
│                  不同角色访问结果对比                                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  数据可见性 (所有成员相同):                                           │
│  ✅ 集合基本信息  ✅ 成员列表  ✅ 链接列表  ✅ 链接详情                │
│  ⚠️ pinnedBy 字段: 仅能看到自己的，看不到其他人的                      │
│                                                                      │
│  操作权限 (角色差异):                                                 │
│  ┌──────────┬──────┬───────┬────────────┬────────┐                  │
│  │ 操作     │Owner│ Admin │Contributor │ Viewer │                  │
│  ├──────────┼──────┼───────┼────────────┼────────┤                  │
│  │ 查看     │  ✅  │  ✅   │    ✅      │   ✅   │                  │
│  │ 创建链接 │  ✅  │  ✅   │    ✅      │   ❌   │                  │
│  │ 更新链接 │  ✅  │  ✅   │    ❌      │   ❌   │                  │
│  │ 删除链接 │  ✅  │  ✅   │    ❌      │   ❌   │                  │
│  │ 移动链接 │  ✅  │  ❌   │    ❌      │   ❌   │                  │
│  │ 管理成员 │  ✅  │  ❌   │    ❌      │   ❌   │                  │
│  │ 退出集合 │  ❌  │  ✅   │    ✅      │   ✅   │                  │
│  └──────────┴──────┴───────┴────────────┴────────┘                  │
│                                                                      │
│  核心设计: 权限隔离在"操作能力"，不在"数据可见性"                      │
│  一旦成为成员，就能看到所有数据，区别仅在于能做什么                    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 九、关键代码索引

| 模块 | 文件路径 | 核心功能 |
|------|----------|----------|
| 数据模型 | [schema.prisma](./packages/prisma/schema.prisma#L126-L164) | Collection 和 UsersAndCollections 表定义 |
| 类型定义 | [global.ts](./packages/types/global.ts#L36-L53) | Member 和 CollectionIncludingMembersAndLinkCount 接口 |
| 权限核心 | [getPermission.ts](./apps/web/lib/api/getPermission.ts) | 服务端权限查询核心函数（注意 collectionId 和 linkId 分支行为不同） |
| 权限追溯 | [getCollectionRootOwnerAndMembers.ts](./apps/web/lib/api/getCollectionRootOwnerAndMembers.ts) | 子集合权限向上追溯与合并 |
| 前端 Hook | [usePermissions.tsx](./apps/web/hooks/usePermissions.tsx) | 前端权限检查 Hook |
| 前端 UI | [EditCollectionSharingModal.tsx](./apps/web/components/ModalContent/EditCollectionSharingModal.tsx) | 成员管理与角色分配界面 |
| 集合操作 | [getCollections.ts](./apps/web/lib/api/controllers/collections/getCollections.ts) | 集合列表资源隔离 |
| 集合操作 | [postCollection.ts](./apps/web/lib/api/controllers/collections/postCollection.ts) | 创建/子集合权限继承（仅继承 rootOwner，不自动继承成员） |
| 集合操作 | [updateCollectionById.ts](./apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts) | 更新集合与权限向下传播（propagateToSubcollections） |
| 集合操作 | [deleteCollectionById.ts](./apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts) | 删除集合/退出集合 |
| 链接操作 | [getLinkById.ts](./apps/web/lib/api/controllers/links/linkId/getLinkById.ts) | 获取链接详情（权限检查在调用方） |
| 链接操作 | [getLinks.ts](./apps/web/lib/api/controllers/links/getLinks.ts) | 链接列表查询与权限过滤 |
| 链接操作 | [searchLinks.ts](./apps/web/lib/api/controllers/search/searchLinks.ts) | 搜索查询与权限过滤 |
| 链接操作 | [updateLinkById.ts](./apps/web/lib/api/controllers/links/linkId/updateLinkById.ts) | 更新链接 |
| 链接操作 | [deleteLinkById.ts](./apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts) | 删除链接 |
| 公开集合 | [getPublicCollection.ts](./apps/web/lib/api/controllers/public/collections/getPublicCollection.ts) | 公开集合查询 |
| 公开链接 | [getLinkById.ts](./apps/web/lib/api/controllers/public/links/linkId/getLinkById.ts) | 公开链接查询 |

---

## 十、修正总结

本次核对修正了以下关键错误：

### 1. 创建子集合时的成员写入操作
- **错误描述**: 认为 `dedupedUsers` 中的成员会被自动写入子集合
- **实际情况**: 创建子集合时，**只有根 Owner 和创建者（如果不是 Owner）** 有权限。父集合的其他成员不会被自动加入，需要通过 `propagateToSubcollections` 手动同步。

### 2. 通过 linkId 访问时的权限过滤位置
- **错误描述**: 认为 `getPermission` 函数的 WHERE 条件统一做了权限过滤
- **实际情况**: `getPermission` 有两个分支行为不一致：
  - `collectionId` 分支: 查询时就做权限过滤，无权限返回 `null`
  - `linkId` 分支: 查询时不做权限过滤，只要 linkId 存在就返回集合，权限检查完全由调用方负责

### 3. 不同角色所看到的访问结果
- **补充内容**: 增加了链接列表查询、搜索查询、公开集合的权限隔离机制
- **补充内容**: 增加了不同角色的访问结果对比表，明确"数据可见性相同，操作能力不同"的核心设计
- **修正内容**: `pinnedBy` 字段在查询时就通过 `where: { id: userId }` 过滤，不是查询后才过滤，查询后的过滤是防御性的二次检查

### 4. 文档链接格式
- **修改内容**: 将所有本地绝对路径 `file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/` 改为仓库相对路径 `./`

### 5. Viewer 角色前后端权限差异
- **补充内容**: 详细说明了 Viewer 角色在前端 Hook 和服务端 API 的权限判定差异
  - 前端 `usePermissions` Hook: 三个权限全为 `false` 时视为 `undefined`（无操作权限），用于隐藏所有修改按钮
  - 服务端 API: 读取操作只要是成员即可，不区分角色；修改操作才检查具体权限
- **补充内容**: 列出了 Viewer 在服务端仍然可以执行的操作：查看集合/链接、Pin 链接、退出集合
- **补充内容**: 说明了这种设计的意图：前端控制 UI 显示，服务端区分读取/修改操作
