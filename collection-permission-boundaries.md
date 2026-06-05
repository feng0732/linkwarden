# 集合权限边界在多人协作下的生效机制

## 一、成员关系数据结构

### 1.1 核心模型定义

**Collection 模型** ([schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/packages/prisma/schema.prisma#L126-L149)):

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

**UsersAndCollections 联结表** ([schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/packages/prisma/schema.prisma#L151-L164)):

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

**Member 接口** ([global.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/packages/types/global.ts#L36-L43)):

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

在前端 [EditCollectionSharingModal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/components/ModalContent/EditCollectionSharingModal.tsx#L248-L253) 中，三个布尔字段组合映射为三种角色：

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

**getPermission** ([getPermission.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/getPermission.ts#L1-L38)):

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
1. **查询即过滤**: `WHERE` 条件本身就做了权限过滤，如果用户没有权限，返回 `null`
2. **返回值包含 members**: 后续逻辑可以从 `members` 数组中精确匹配用户的具体权限

### 2.2 权限判定通用模式

所有 API 控制器都遵循以下模式：

```
1. 调用 getPermission(userId, collectionId | linkId)
2. 检查返回值是否为 null → 无权限，返回 401
3. 检查是否为 Owner: collectionIsAccessible?.ownerId === userId
4. 检查成员具体权限: collectionIsAccessible?.members.some(e => e.userId === userId && e.canXXX)
5. 两者满足其一即可执行操作
```

---

## 三、各操作的权限要求

### 3.1 集合 (Collection) 操作权限矩阵

| 操作 | Owner | Admin | Contributor | Viewer | 代码位置 |
|------|-------|-------|-------------|--------|----------|
| **查看集合** | ✅ | ✅ | ✅ | ✅ | [getCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/collectionId/getCollectionById.ts#L7-L13) |
| **创建集合(根级)** | ✅ | ❌ | ❌ | ❌ | [postCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/postCollection.ts#L84-L120) |
| **创建子集合** | ✅ | ✅ (需三个权限全有) | ❌ | ❌ | [postCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/postCollection.ts#L49-L52) |
| **更新集合(含成员管理)** | ✅ | ❌ | ❌ | ❌ | [updateCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts#L34) |
| **删除集合(彻底删除)** | ✅ | ❌ | ❌ | ❌ | [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L50-L52) |
| **退出集合(仅移除自身成员关系)** | ❌ | ✅ | ✅ | ✅ | [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L23-L49) |

### 3.2 链接 (Link) 操作权限矩阵

| 操作 | Owner | Admin | Contributor | Viewer | 代码位置 |
|------|-------|-------|-------------|--------|----------|
| **查看链接** | ✅ | ✅ | ✅ | ✅ | [getLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/links/linkId/getLinkById.ts#L20-L24) |
| **创建链接** | ✅ | ✅ | ✅ | ❌ | [setCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/setCollection.ts#L30-L32) |
| **更新链接** | ✅ | ✅ | ❌ | ❌ | [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L72-L74) |
| **删除链接** | ✅ | ✅ | ❌ | ❌ | [deleteLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts#L12-L14) |
| **移动链接到其他集合** | ✅ | ❌ | ❌ | ❌ | [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L88-L96) |
| **Pin 链接到个人仪表板** | ✅ | ✅ | ✅ | ✅ | [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L36-L65) |
| **批量删除链接** | ✅ | ✅ | ❌ | ❌ | [deleteLinksById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/links/bulk/deleteLinksById.ts#L24-L26) |

### 3.3 标签 (Tag) 操作权限矩阵

| 操作 | Owner | Admin | Contributor | Viewer | 代码位置 |
|------|-------|-------|-------------|--------|----------|
| **删除标签** | ✅ | ❌ | ❌ | ❌ | [deleteTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts#L13-L17) |

> **重要**: 标签的所有权独立于集合，标签有自己的 `ownerId`，只有标签的所有者才能删除标签。这是一个独立的权限边界。

---

## 四、资源隔离机制

### 4.1 集合列表的资源隔离

**getCollections** ([getCollections.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/getCollections.ts#L4-L10)):

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

**getCollectionById** ([getCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/collectionId/getCollectionById.ts#L7-L14)):

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

**getLinkById** ([getLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/links/linkId/getLinkById.ts#L12-L24)):

1. 通过 `linkId` 调用 `getPermission` 获取所属集合
2. 验证用户对该集合的访问权限（Owner 或 Member）
3. 权限验证通过后才返回链接数据

**特殊隔离**: 即使是成员，返回的 `pinnedBy` 字段也会被过滤，只保留当前用户的 pin 记录 ([getLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/links/linkId/getLinkById.ts#L41-L43)):

```typescript
// strip out the other users from pinnedBy
if (link?.pinnedBy && link.pinnedBy.length > 0) {
  link.pinnedBy = link.pinnedBy.filter((p) => p.id === userId);
}
```

---

## 五、子集合的权限继承与传播

### 5.1 权限向上追溯机制

**getCollectionRootOwnerAndMembers** ([getCollectionRootOwnerAndMembers.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/getCollectionRootOwnerAndMembers.ts#L18-L73)):

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

**权限合并规则** ([getCollectionRootOwnerAndMembers.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/getCollectionRootOwnerAndMembers.ts#L9-L16)):

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

在 [postCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/postCollection.ts#L61-L81) 中：

1. 调用 `getCollectionRootOwnerAndMembers(parentId)` 获取根所有者和所有合并后的成员
2. 新子集合的 `ownerId` 设置为根所有者的 ID（不是创建者的 ID）
3. 如果创建者不是根所有者，自动将创建者以 Admin 身份加入成员表

```typescript
const result = await getCollectionRootOwnerAndMembers(collection.parentId);
rootOwnerId = result.rootOwnerId;  // 子集合的 owner 是根集合的 owner

// 新集合的 owner 是 rootOwnerId，不是创建者 userId
const newCollection = await prisma.collection.create({
  data: {
    owner: { connect: { id: rootOwnerId } },  // 关键点
    createdBy: { connect: { id: userId } },   // 记录创建者
    members: userId !== rootOwnerId ? {
      create: [{ userId, canCreate: true, canUpdate: true, canDelete: true }]
    } : undefined,
    // ...
  },
});
```

### 5.3 更新集合时的权限向下传播

在 [updateCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts#L68-L121) 中：

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

**usePermissions** ([usePermissions.tsx](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/hooks/usePermissions.tsx#L1-L32)):

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
- `Member` 对象: 用户是成员，可从中读取具体的 `canCreate/canUpdate/canDelete`
- `undefined`: 用户无权限

### 6.2 useCollectivePermissions Hook

**useCollectivePermissions** ([useCollectivePermissions.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/hooks/useCollectivePermissions.ts#L1-L34)):

与 `usePermissions` 逻辑相同，但支持传入多个 `collectionId`，遍历检查用户对所有集合的权限。

### 6.3 前端权限使用示例

在 [EditCollectionSharingModal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/components/ModalContent/EditCollectionSharingModal.tsx#L79) 中:

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

当非 Owner 用户调用删除集合 API 时 ([deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L23-L49)):

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

```
┌─────────────────────────────────────────────────────────────────┐
│                         权限判定总流程                            │
├─────────────────────────────────────────────────────────────────┤
│  1. getPermission(userId, collectionId | linkId)                │
│     └─ WHERE: ownerId = userId OR members.some(userId)          │
│     └─ 返回 Collection + members[] 或 null                       │
│                                                                  │
│  2. 检查是否为 Owner: collection.ownerId === userId             │
│     └─ 是 → 拥有所有权限，跳过成员检查                           │
│     └─ 否 → 进入成员权限检查                                     │
│                                                                  │
│  3. 成员权限检查: collection.members.some(...)                  │
│     ├─ 查看操作: 只要是成员即可 (任意角色)                       │
│     ├─ 创建操作: 需要 canCreate = true                          │
│     ├─ 更新操作: 需要 canUpdate = true                          │
│     ├─ 删除操作: 需要 canDelete = true                          │
│     └─ 特殊操作: 需要三个权限全有 (如创建子集合)                 │
│                                                                  │
│  4. 权限通过 → 执行业务逻辑                                      │
│  5. 权限拒绝 → 返回 401/403                                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                       子集合权限流向                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  创建时: 向上追溯 → 合并权限 → 继承根 Owner                      │
│  更新时: 向下传播 → 覆盖子集合 → 批量同步成员                     │
│                                                                  │
│    根集合 (Owner: A, Members: [B(Admin), C(Viewer)])            │
│         │                                                        │
│         ▼ 向上追溯取 rootOwnerId = A                             │
│    子集合 (Owner: A, Members: [创建者(Admin) + 继承的成员])      │
│         │                                                        │
│         ▼ 向下传播 (propagateToSubcollections)                  │
│    孙集合 (Owner: A, Members: [与父集合相同])                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 九、关键代码索引

| 模块 | 文件路径 | 核心功能 |
|------|----------|----------|
| 数据模型 | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/packages/prisma/schema.prisma#L126-L164) | Collection 和 UsersAndCollections 表定义 |
| 类型定义 | [global.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/packages/types/global.ts#L36-L53) | Member 和 CollectionIncludingMembersAndLinkCount 接口 |
| 权限核心 | [getPermission.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/getPermission.ts) | 服务端权限查询核心函数 |
| 权限追溯 | [getCollectionRootOwnerAndMembers.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/getCollectionRootOwnerAndMembers.ts) | 子集合权限向上追溯与合并 |
| 前端 Hook | [usePermissions.tsx](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/hooks/usePermissions.tsx) | 前端权限检查 Hook |
| 前端 UI | [EditCollectionSharingModal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/components/ModalContent/EditCollectionSharingModal.tsx) | 成员管理与角色分配界面 |
| 集合操作 | [getCollections.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/getCollections.ts) | 集合列表资源隔离 |
| 集合操作 | [postCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/postCollection.ts) | 创建/子集合权限继承 |
| 集合操作 | [updateCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts) | 更新集合与权限向下传播 |
| 集合操作 | [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts) | 删除集合/退出集合 |
| 链接操作 | [getLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/links/linkId/getLinkById.ts) | 获取链接详情 |
| 链接操作 | [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts) | 更新链接 |
| 链接操作 | [deleteLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/37-linkwarden/apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts) | 删除链接 |
