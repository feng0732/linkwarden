# Prisma 数据模型关系与迁移分析

## 1. 核心数据模型关系

### 1.1 User（用户）模型
定义位置: [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/schema.prisma#L28-L75)

**核心字段**:
- `id` (Int, PK, autoincrement)
- `username` (String?, unique)
- `email` (String?, unique)
- `password` (String?)
- `isPrivate` (Boolean)
- `parentSubscriptionId` (Int?) — 支持子账号订阅关系

**关系**:
- 1:N → `collections` (拥有的收藏夹, `owner`)
- 1:N → `tags` (拥有的标签)
- N:M → `pinnedLinks` (置顶的链接, 通过 `_PinnedLinks`)
- 1:N → `createdLinks` (创建的链接, **Link.createdById CASCADE**)
- 1:N → `createdCollections` (创建的收藏夹, **Collection.createdById SET NULL**)
- 1:N → `highlights` (高亮)
- N:M → `collectionsJoined` (加入的收藏夹, 通过 `UsersAndCollections`)
- 1:1 → `subscriptions` (订阅)
- 1:N → `whitelistedUsers`、`accessTokens`、`dashboardSections`、`accounts`

---

### 1.2 Collection（收藏夹）模型
定义位置: [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/schema.prisma#L126-L149)

**核心字段**:
- `id` (Int, PK)
- `name` (String) — **无唯一约束（曾有后移除）**
- `description` (String)
- `color` (String, default: "#0ea5e9")
- `icon`, `iconWeight` (String?)
- `parentId` (Int?) — 支持自引用层级结构
- `ownerId` (Int, NOT NULL) — 所属用户
- `createdById` (Int?) — 创建者用户，**可与 ownerId 不同**
- `isPublic` (Boolean)

**关系**:
| 关系 | onDelete 行为 | 说明 |
|------|--------------|------|
| `owner` → User | `Cascade` | 删除用户 → 删除其拥有的所有收藏夹 |
| `parent` → Collection | `Cascade` | 删除父收藏夹 → 级联删除所有子收藏夹 |
| `createdBy` → User | **SET NULL**（默认行为，schema 未显式指定，nullable 外键 Prisma 默认为 SetNull） | 删除创建者 → 仅清空 createdById 字段，收藏夹本身保留 |
| `members` → User | 中间表 Cascade | 通过 UsersAndCollections |
| `links` → Link | N/A（反向） | 删除收藏夹 → 通过 Link.collectionId 级联删除链接 |

**索引**:
- `@@index([ownerId])`

> ⚠️ **重要**: `owner`（所有者）与 `createdBy`（创建者）可以是**不同用户**。在共享收藏夹场景下，成员用户在别人的收藏夹下创建子收藏夹时，子收藏夹的 ownerId 是根收藏夹所有者，而 createdById 是实际创建者。参考 [postCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/collections/postCollection.ts#L28-L92)。

---

### 1.3 Link（链接）模型
定义位置: [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/schema.prisma#L166-L198)

**核心字段**:
- `id` (Int, PK)
- `name` (String, default: "")
- `type` (String, default: "url")
- `url` (String?)
- `description` (String)
- `collectionId` (Int, NOT NULL)
- `createdById` (Int?) — 创建者用户
- 归档相关: `image`, `pdf`, `readable`, `monolith`, `preview`, `textContent`
- `icon`, `iconWeight`, `color`
- `aiTagged` (Boolean)
- `indexVersion` (Int?) — Meilisearch 索引版本
- `lastPreserved`, `importDate` (DateTime?)
- `clientSide` (Boolean)

**关系**:
| 关系 | onDelete 行为 | 说明 |
|------|--------------|------|
| `collection` → Collection | `Cascade` | 删除收藏夹 → 删除其下所有链接 |
| **`createdBy` → User** | **`Cascade`** | **删除创建者用户 → 删除该用户创建的所有链接（即使链接在别人收藏夹中！）** |
| `pinnedBy` → User | Cascade（中间表） | 删除用户或链接 → 解除置顶关系 |
| `tags` → Tag | Cascade（中间表） | 删除链接或标签 → 解除关联 |
| `highlight` → Highlight | N/A（反向） | 通过 Highlight.linkId Cascade 删除 |

**索引**:
- `@@index([collectionId])`

> ⚠️ **关键风险**: Link.createdById 是 CASCADE。在共享收藏夹场景中，成员创建的链接归属于收藏夹所有者（通过 collection），但 createdById 是成员自己。删除该成员用户时，**收藏夹所有者的数据（链接）会被意外级联删除**。

---

### 1.4 Tag（标签）模型
定义位置: [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/schema.prisma#L200-L218)

**核心字段**:
- `id` (Int, PK)
- `name` (String)
- `ownerId` (Int, NOT NULL)
- 归档选项覆盖: `archiveAsScreenshot`, `archiveAsMonolith`, `archiveAsPDF`, `archiveAsReadable`, `archiveAsWaybackMachine` (Boolean?)
- `aiTag` (Boolean?)
- `aiGenerated` (Boolean, default: false)

**关系**:
- N:1 → `owner` (所属用户, `onDelete: Cascade`)
- N:M → `links` (关联的链接, 通过 `_LinkToTag`)

**唯一约束与索引**:
- `@@unique([name, ownerId])` — **同一用户下标签名必须唯一**，被 `createOrUpdateTags` 的 upsert 逻辑所依赖
- `@@index([ownerId])`

---

### 1.5 关系连接表

#### UsersAndCollections（用户-收藏夹多对多）
定义位置: [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/schema.prisma#L151-L164)

- 复合主键: `@@id([userId, collectionId])`
- 权限字段: `canCreate`, `canUpdate`, `canDelete` (Boolean)
- 两边均为 `onDelete: Cascade`

#### _LinkToTag（链接-标签多对多）
- 复合主键 `(A, B)`，对应 Link.id ↔ Tag.id
- 两边均为 `ON DELETE CASCADE`

#### _PinnedLinks（用户-链接置顶多对多）
- 复合主键 `(A, B)`，对应 Link.id ↔ User.id
- 两边均为 `ON DELETE CASCADE`

---

## 2. ER 关系概览

```
User (1) ─── owner (Cascade) ──< Collection (N)
  │                                  │  ^
  │                                  │  │ parent/subCollections (Cascade 自引用)
  │                                  v  │
  │ createdBy (SET NULL) ──< Collection
  │                                  │
  │                                  │ links (通过 collectionId Cascade)
  │                                  v
  │ createdBy (CASCADE!) ────< Link (N) >──< Tag (N) >── owner (Cascade) ──> User (1)
  │                                  │
  │                                  │ pinnedBy (中间表 Cascade)
  │                                  v
  └────────────< UsersAndCollections >─────────────┘
                   (成员权限表，两边 Cascade)
```

**createdById 行为差异**:
- Collection.createdById → 删除创建者：**SET NULL**（保留收藏夹）
- Link.createdById → 删除创建者：**CASCADE**（删除链接！）

---

## 3. 迁移演进过程

迁移目录: [packages/prisma/migrations/](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/)

### 3.1 初始版本（20230719_init）
迁移文件: [20230719181459_init/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20230719181459_init/migration.sql)

**初始外键级联策略**:
| 关系 | ON DELETE |
|------|-----------|
| Account.userId | CASCADE |
| Session.userId | CASCADE |
| Collection.ownerId | **RESTRICT** |
| UsersAndCollections.userId | **RESTRICT** |
| UsersAndCollections.collectionId | **RESTRICT** |
| Link.collectionId | **RESTRICT** |
| Tag.ownerId | **RESTRICT** |
| _LinkToUser / _LinkToTag | CASCADE |

**初始唯一约束**:
- `Collection(name, ownerId)` — 同一用户下收藏夹名唯一（后被移除）
- `Tag(name, ownerId)` — 同一用户下标签名唯一（至今保留）
- `User.username`, `User.email`

---

### 3.2 唯一约束的移除

#### 收藏夹名唯一约束移除（20240218）
迁移文件: [20240218080348_allow_duplicate_collection_names/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20240218080348_allow_duplicate_collection_names/migration.sql)

```sql
DROP INDEX "Collection_name_ownerId_key";
```
**影响**: 同一用户可以创建多个同名收藏夹，仅靠 `id` 区分。

#### AccessToken 名称唯一约束移除（20240124）
迁移文件: [20240124201018_removed_name_unique_constraint/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20240124201018_removed_name_unique_constraint/migration.sql)

```sql
DROP INDEX "AccessToken_name_userId_key";
```

---

### 3.3 子收藏夹功能引入（20240125）
迁移文件: [20240125124457_added_subcollection_relations/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20240125124457_added_subcollection_relations/migration.sql)

```sql
ALTER TABLE "Collection" ADD COLUMN "parentId" INTEGER;
ALTER TABLE "Collection" ADD CONSTRAINT ... FOREIGN KEY ("parentId") 
    REFERENCES "Collection"("id") ON DELETE SET NULL ON UPDATE CASCADE;
```
初始策略: 删除父收藏夹时，子收藏夹的 `parentId` 置为 NULL（提升为根级）。20250318 改为 CASCADE。

---

### 3.4 createdById 字段完整迁移时间线

这是 Collection.createdById 与 Link.createdById 行为产生**分歧**的关键演化过程：

#### 阶段一：字段首次添加（20241021）
迁移文件: [20241021175802_add_child_subscription_support/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20241021175802_add_child_subscription_support/migration.sql)

```sql
ALTER TABLE "Collection" ADD COLUMN "createdById" INTEGER;
ALTER TABLE "Link" ADD COLUMN "createdById" INTEGER;

-- Collection.createdById: ON DELETE SET NULL
ALTER TABLE "Collection" ADD CONSTRAINT "Collection_createdById_fkey" 
    FOREIGN KEY ("createdById") REFERENCES "User"("id") ON DELETE SET NULL ON UPDATE CASCADE;

-- Link.createdById: ON DELETE SET NULL
ALTER TABLE "Link" ADD CONSTRAINT "Link_createdById_fkey" 
    FOREIGN KEY ("createdById") REFERENCES "User"("id") ON DELETE SET NULL ON UPDATE CASCADE;
```

此时两者行为一致：删除用户 → 置空 createdById。

#### 阶段二：强制 NOT NULL + RESTRICT（20241026）
迁移文件: [20241026161909_assign_createdby_to_collection_owners_and_make_field_required/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20241026161909_assign_createdby_to_collection_owners_and_make_field_required/migration.sql)

**数据填充逻辑**:
```sql
UPDATE "Link" SET "createdById" = (
  SELECT "ownerId" FROM "Collection" WHERE "Collection"."id" = "Link"."collectionId"
);
UPDATE "Collection" SET "createdById" = "ownerId";
```
然后将字段设为 NOT NULL，外键均改为 `ON DELETE RESTRICT`。

**风险**：如果 Link.collectionId 指向不存在的 Collection（脏数据），子查询返回 NULL，导致后续 SET NOT NULL 失败，迁移中断。

#### 阶段三：回退为可空 + SET NULL（20241030）
迁移文件: [20241030200844_createdby_fields_can_be_null/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20241030200844_createdby_fields_can_be_null/migration.sql)

```sql
ALTER TABLE "Collection" ALTER COLUMN "createdById" DROP NOT NULL;
ALTER TABLE "Link" ALTER COLUMN "createdById" DROP NOT NULL;

-- 两者均恢复为 ON DELETE SET NULL
ALTER TABLE "Collection" ADD CONSTRAINT "Collection_createdById_fkey" 
    FOREIGN KEY ("createdById") REFERENCES "User"("id") ON DELETE SET NULL ON UPDATE CASCADE;
ALTER TABLE "Link" ADD CONSTRAINT "Link_createdById_fkey" 
    FOREIGN KEY ("createdById") REFERENCES "User"("id") ON DELETE SET NULL ON UPDATE CASCADE;
```

**原因推测**: 导入外部数据时可能不存在创建者，或创建者被删除后需要保留数据。此时两者行为再次一致。

#### 阶段四：两者产生分歧 —— Link.createdById 改为 CASCADE（20250318）
迁移文件: [20250318123928_add_referential_actions_to_certain_fields/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20250318123928_add_referential_actions_to_certain_fields/migration.sql)

```sql
-- ⚠️ 只修改了 Link.createdById！
ALTER TABLE "Link" DROP CONSTRAINT "Link_createdById_fkey";
ALTER TABLE "Link" ADD CONSTRAINT "Link_createdById_fkey" 
    FOREIGN KEY ("createdById") REFERENCES "User"("id") ON DELETE CASCADE ON UPDATE CASCADE;

-- ❌ Collection.createdById_fkey 在此次迁移中完全没有出现，未被修改！
```

**最终分歧**:
| 外键 | 当前 ON DELETE | 最后修改迁移 |
|------|--------------|-------------|
| Collection.createdById | **SET NULL** | 20241030（未被 20250318 修改） |
| Link.createdById | **CASCADE** | 20250318 |

---

### 3.5 全局级联策略变更（20250318）

这是最关键的迁移系列，将大部分 `RESTRICT` / `SET NULL` 改为 `CASCADE`，但 **Collection.createdById 被遗漏**。

#### 第一批（20250318123928）
迁移文件: [20250318123928_add_referential_actions_to_certain_fields/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20250318123928_add_referential_actions_to_certain_fields/migration.sql)

改为 CASCADE 的关系:
- `Collection.parentId` (SET NULL → CASCADE)
- `UsersAndCollections.userId` (RESTRICT → CASCADE)
- `UsersAndCollections.collectionId` (RESTRICT → CASCADE)
- **`Link.createdById`** (SET NULL → CASCADE) ⚠️
- `Link.collectionId` (RESTRICT → CASCADE)
- `Tag.ownerId` (RESTRICT → CASCADE)
- `Subscription.userId` (RESTRICT → CASCADE)
- `AccessToken.userId` (RESTRICT → CASCADE)
- `RssSubscription.collectionId` (RESTRICT → CASCADE)
- `Highlight.linkId`, `Highlight.userId` (RESTRICT → CASCADE)

> ⚠️ **Collection.createdById 未被修改**，仍停留在 20241030 的 SET NULL 状态。

#### 第二批（20250318130241）
- `WhitelistedUser.userId` → CASCADE

#### 第三批（20250318131012）
- `Collection.ownerId` (RESTRICT → CASCADE) — 最后一个核心关系切换

---

### 3.6 多对多连接表主键化（20250627 upgrade_to_v6）
迁移文件: [20250627132552_upgrade_to_v6/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20250627132552_upgrade_to_v6/migration.sql)

```sql
ALTER TABLE "_LinkToTag" ADD CONSTRAINT "_LinkToTag_AB_pkey" PRIMARY KEY ("A", "B");
DROP INDEX "_LinkToTag_AB_unique";
ALTER TABLE "_PinnedLinks" ADD CONSTRAINT "_PinnedLinks_AB_pkey" PRIMARY KEY ("A", "B");
DROP INDEX "_PinnedLinks_AB_unique";
```
隐式多对多表从"唯一索引 + 无主键"升级为"复合主键"。

---

## 4. createdById 在删除时的实际影响（对照代码）

### 4.1 删除 User 的完整级联路径

代码位置: [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts)

#### 数据库级联效果

| 外键关系 | ON DELETE | 实际效果 |
|---------|-----------|---------|
| Collection.ownerId | Cascade | 删除该用户拥有的所有收藏夹 |
| Tag.ownerId | Cascade | 删除该用户拥有的所有标签 |
| **Collection.createdById** | **SET NULL** | 该用户创建的所有收藏夹的 createdById → NULL，**收藏夹本身保留**（只要 owner 不是该用户） |
| **Link.createdById** | **CASCADE** | **该用户创建的所有链接被直接删除，即使这些链接当前属于其他用户的收藏夹！** |
| Link.collectionId → (通过 Collection.ownerId 级联) | Cascade | 该用户拥有的收藏夹下的链接被删除 |
| Highlight.linkId / Highlight.userId | Cascade | 上述被删链接/用户相关的高亮被删除 |
| UsersAndCollections.userId | Cascade | 该用户的成员资格被解除 |
| UsersAndCollections.collectionId → (通过 Collection.ownerId 级联) | Cascade | 该用户收藏夹的所有成员资格被解除 |
| DashboardSection.userId / DashboardSection.collectionId | Cascade | 相关仪表盘分区被删除 |
| Subscription.userId | Cascade | 订阅被删除 |
| AccessToken.userId | Cascade | 访问令牌被删除 |
| WhitelistedUser.userId | Cascade | 白名单记录被删除 |
| Account.userId | Cascade | OAuth 账户被删除 |
| _LinkToTag / _PinnedLinks | Cascade | 中间表关联自动清理 |

#### 应用层清理 vs 数据库级联的**覆盖缺口**

在 [deleteUserById.ts#L111-L118](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L111-L118)：

```typescript
// 应用层只查找了「该用户拥有的 Collection 下的 Link」
const links = await prisma.link.findMany({
  where: { collection: { ownerId: queryId } },
  select: { id: true },
});
await meiliClient?.index("links").deleteDocuments(linkIds);
```

但由于 **Link.createdById CASCADE**，数据库还会删除以下链接（应用层**未清理**）：
- 该用户作为**成员**在别人的共享收藏夹中创建的所有 Link（collection.ownerId ≠ userId，但 Link.createdById = userId）

**遗漏后果**:
1. **Meilisearch 索引残留**：这些 Link 的文档不会被删除，搜索时出现孤儿记录
2. **归档文件残留**：这些 Link 的归档文件（archives/{collectionId}/{linkId}.*）不会被删除，磁盘残留

---

### 4.2 删除 Collection 的影响

代码位置: [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts)

#### 数据库级联效果
| 外键关系 | ON DELETE | 实际效果 |
|---------|-----------|---------|
| Link.collectionId | Cascade | 删除收藏夹下所有链接 |
| Collection.parentId (对子收藏夹) | Cascade | 递归删除所有层级的子收藏夹 |
| Highlight.linkId → (通过 Link 级联) | Cascade | 链接的高亮被删除 |
| UsersAndCollections.collectionId | Cascade | 解除所有成员关系 |
| RssSubscription.collectionId | Cascade | 删除 RSS 订阅 |
| DashboardSection.collectionId | Cascade | 删除仪表盘分区 |
| _LinkToTag / _PinnedLinks | Cascade | 中间表自动清理 |
| **Collection.createdById** | — | 删除收藏夹本身，不影响其创建者 User |

**createdById 不反向影响**：Collection.createdById 只是 Collection 引用 User，删除 Collection 不会反作用于 User。

---

### 4.3 删除 Link 的影响

代码位置: [deleteLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts)

#### 数据库级联效果
| 外键关系 | ON DELETE | 实际效果 |
|---------|-----------|---------|
| Highlight.linkId | Cascade | 删除链接的所有高亮 |
| _LinkToTag (Link 侧) | Cascade | 解除与所有 Tag 的关联 |
| _PinnedLinks (Link 侧) | Cascade | 解除所有用户的置顶 |
| **Link.createdById** | — | 删除链接本身，不影响其创建者 User |

**Link.createdById 不反向影响 User**：Link.createdById 是 Link 引用 User，删除 Link 不会导致 User 被删除。

---

### 4.4 删除 Tag 的影响

代码位置: [deleteTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts)

| 外键关系 | ON DELETE | 实际效果 |
|---------|-----------|---------|
| _LinkToTag (Tag 侧) | Cascade | 解除该 Tag 与所有 Link 的关联，Tag 本身被删除，Link 保留 |

---

## 5. 业务逻辑中的补充清理（非数据库级联）

数据库级联无法覆盖文件系统、搜索引擎索引等外部资源。

### 5.1 删除收藏夹 [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts)

```
事务内执行:
├─ 递归删除所有子 Collection（重复下述流程）
├─ 删除 UsersAndCollections 关系
├─ 删除归档文件目录 archives/{collectionId} 和 archives/preview/{collectionId}
├─ 从 User.collectionOrder 数组中移除该收藏夹 ID
├─ 清理 DashboardSection 中引用该收藏夹的记录，并调整顺序
├─ 从 Meilisearch 索引中批量删除该收藏夹下所有 Link
└─ 删除该 Collection（触发数据库级联：删除 Link → 删除 Highlight、解除 Tag 关联等）
```

### 5.2 删除用户 [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts)

```
事务内执行:
├─ ❌ 查询「该用户拥有的 Collection 下的 Link」，从 Meilisearch 删除
│     ⚠️ 遗漏：该用户创建但属于其他用户收藏夹的 Link（被 Link.createdById CASCADE 删除）
├─ 删除该用户所有 Collection 的归档文件目录
│     ⚠️ 遗漏：该用户创建但属于其他用户收藏夹的 Link 的归档文件
├─ 删除用户头像文件
├─ 处理 Stripe 订阅取消/席位变更
└─ 删除 User（触发数据库级联：Collection → Link → ... 整条链 + createdById CASCADE）
```

### 5.3 删除标签 [deleteTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts)

```
├─ 权限校验（ownerId === userId）
├─ 删除 Tag（触发级联：删除所有 _LinkToTag 关联）
└─ 将受影响 Link 的 indexVersion 置 null（触发 Meilisearch 重新索引）
```

### 5.4 删除链接 [deleteLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts)

```
├─ 权限校验（所有者或有 canDelete 权限的成员）
├─ 删除 Link（触发级联：删除 Highlight、解除 Tag 关联、解除置顶）
├─ 删除归档文件
└─ 从 Meilisearch 索引删除
```

---

## 6. 唯一约束分析

### 6.1 当前存在的唯一约束

| 模型 | 约束字段 | 说明 |
|------|---------|------|
| User | `username` | 全局唯一用户名 |
| User | `email` | 全局唯一邮箱 |
| Tag | `(name, ownerId)` | 同一用户下标签名唯一，被 upsert 逻辑依赖 |
| Account | `(provider, providerAccountId)` | OAuth 账户唯一 |
| VerificationToken | `(identifier, token)` | 验证令牌唯一 |
| VerificationToken | `token` | 单列唯一 |
| PasswordResetToken | `token` | 单列唯一 |
| Subscription | `stripeSubscriptionId` | Stripe 订阅 ID 全局唯一 |
| Subscription | `userId` | 每个用户最多一个订阅 |
| AccessToken | `token` | 访问令牌全局唯一 |
| DashboardSection | `(userId, collectionId)` | 同一用户对同一收藏夹只能有一个仪表盘分区 |
| AppMigration | `name` | 应用迁移名称唯一 |
| UsersAndCollections | `(userId, collectionId)` | 复合主键 |
| _LinkToTag | `(A, B)` | 复合主键 |
| _PinnedLinks | `(A, B)` | 复合主键 |

### 6.2 关键业务依赖的唯一约束

**Tag(name, ownerId)** — 在 [createOrUpdateTags.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/createOrUpdateTags.ts#L12-L18) 中使用 `upsert`:

```typescript
prisma.tag.upsert({
  where: { name_ownerId: { name: tag.label, ownerId: userId } },
  update: { ... },
  create: { ... }
})
```
**强依赖**: `name_ownerId` 必须是唯一索引，否则 upsert 无法工作。

---

## 7. 回滚风险分析（校正版）

### 7.1 高风险迁移（不可逆或可能丢失数据）

| 迁移 | 风险等级 | 原因 |
|------|---------|------|
| **20250318 Link.createdById 改为 CASCADE** | 🔴 **极高** | 回退到 SET NULL 时，数据库中可能已经因为 CASCADE 删除了大量 Link，这些数据永久丢失。更严重的是，共享收藏夹场景中**他人的数据被意外删除**，无法恢复。 |
| **20250318 全局级联变更（其余关系）** | ⚠️ 高 | 将 RESTRICT 改为 CASCADE 是单向的。回退到 RESTRICT 会导致应用层所有删除操作因外键约束失败（代码逻辑已依赖 CASCADE 清理子表，不做手动预删除）。 |
| **20231027 删除 Account / Session 表** | 🔴 极高 | 直接 DROP TABLE，所有数据永久丢失。无回退脚本，只能依赖备份恢复。 |
| **20241026 createdById 强制 NOT NULL** | ⚠️ 高（但已被后续迁移回退） | 若历史数据存在脏数据（Link.collectionId 悬空），数据回填 SQL 返回 NULL，迁移直接失败。 |
| **20240218 移除 Collection 唯一约束** | ⚠️ 中 | 一旦用户创建了同名收藏夹，再恢复唯一约束会因数据冲突失败，需先手动清理重名数据。 |
| **20250627 多对多表主键化** | ⚠️ 中 | 如果连接表中存在重复行（尽管有唯一索引不太可能），加主键会失败。回退需删主键再建唯一索引。 |

### 7.2 Link.createdById CASCADE 的回滚专项分析

**当前状态**: Link.createdById 是 ON DELETE CASCADE

**回滚方案（假设要改回 SET NULL）**:
1. 数据库层面：`ALTER TABLE "Link" DROP CONSTRAINT ...; ALTER TABLE "Link" ADD CONSTRAINT ... ON DELETE SET NULL;`
2. 但此时已经因 CASCADE 被删除的 Link **无法恢复**
3. 而且，应用层 deleteUserById.ts 的清理逻辑也不完整（见 4.1 节），即使回滚级联策略，也需要同时修复清理代码

**更安全的方向**: 保持 SET NULL，删除用户时只清空 Link.createdById，保留链接数据（因为链接的真正归属是通过 Collection.ownerId 决定的，而非 createdById）。

### 7.3 数据迁移回填风险

**20241026 迁移中的数据回填**:
```sql
UPDATE "Link" SET "createdById" = (
  SELECT "ownerId" FROM "Collection" WHERE "Collection"."id" = "Link"."collectionId"
);
```
**风险点**: 如果存在脏数据（Link.collectionId 指向不存在的 Collection），子查询返回 NULL，而后续 `SET NOT NULL` 会导致迁移失败。

### 7.4 唯一约束回滚冲突模式

典型场景（Tag 为例，目前仍保留唯一约束）:
1. 用户创建标签 "work"
2. 某迁移临时移除 `Tag(name, ownerId)` 唯一约束
3. 用户在此期间又创建了一个名为 "work" 的标签
4. 回滚迁移恢复唯一约束 → 因已有重名数据，迁移失败

**Collection 已实际发生此模式**: 20240218 已永久移除唯一约束，且代码逻辑不再依赖。

### 7.5 Prisma 迁移本身的回滚限制

Prisma Migrate **不提供自动回滚机制**。每个 `migration.sql` 仅包含正向变更。回滚方案:
- 依赖数据库备份（PITR，时间点恢复）
- 手动编写反向迁移 SQL
- 使用 `prisma migrate resolve` 标记失败迁移为已应用/已回滚

---

## 8. 历史数据兼容性总结

| 变更 | 对历史数据的处理 | 兼容性 |
|------|----------------|--------|
| Collection 新增字段（icon, color 等） | 均有 DEFAULT 值 | ✅ 自动兼容 |
| Link 新增归档字段 | 均有 DEFAULT 或为 nullable | ✅ 自动兼容 |
| Link.url 变为 nullable | 无默认值但允许 NULL | ⚠️ 旧数据必有值，兼容 |
| Tag 新增归档选项覆盖字段 | 均为 nullable | ✅ 自动兼容 |
| User 新增大量偏好字段 | 均有 DEFAULT | ✅ 自动兼容 |
| createdById 字段添加 | 迁移中用 SQL 回填，后改为可空 | ✅ 已兼容 |
| 多对多连接表加主键 | 历史数据已保证唯一 | ✅ 兼容 |
| 级联策略变更 | 不影响现有数据，只影响后续删除行为 | ✅ 兼容，但语义变化需上层感知 |
| 移除唯一约束 | 不影响数据 | ✅ 兼容 |

---

## 9. 关键发现与建议总结

### 9.1 已确认的代码行为

1. **Collection.createdById 与 Link.createdById 行为不一致**
   - Collection.createdById → SET NULL（schema.prisma 未显式指定，Prisma 对 nullable 外键默认为 SetNull，与最后一次迁移 20241030 一致）
   - Link.createdById → CASCADE（schema.prisma 显式指定 `onDelete: Cascade`，迁移 20250318 修改）
   - 这种不一致可能是 20250318 全局级联变更时**遗漏了 Collection.createdById**

2. **共享收藏夹场景下 owner ≠ createdBy**
   - [postCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/collections/postCollection.ts#L28-L92) 中，rootOwnerId（收藏夹所有者）与 userId（创建者）可以不同
   - Link 同理：成员在共享收藏夹中创建的链接，collection.ownerId ≠ link.createdById

3. **删除用户存在外部资源清理遗漏**
   - [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L111-L118) 只按 collection.ownerId 查找 Link
   - 遗漏了 createdById = userId 但属于他人收藏夹的 Link → Meilisearch 孤儿索引 + 归档文件残留

### 9.2 建议

1. **统一 createdById 的级联策略**：考虑将 Link.createdById 也改为 SET NULL，与 Collection.createdById 保持一致。链接的归属应由 Collection.ownerId 决定，而不是创建者。删除用户不应导致他人收藏夹中的数据丢失。

2. **修复 deleteUserById 的清理范围**：
   ```typescript
   // 补充查询 createdById = userId 的 Link
   const createdLinks = await prisma.link.findMany({
     where: { createdById: queryId },
     select: { id: true, collectionId: true }
   });
   // 一并清理 Meilisearch 索引和归档文件
   ```

3. **在 schema.prisma 中为 Collection.createdBy 显式添加 `onDelete: SetNull`**：消除歧义，避免未来 Prisma 默认行为变更导致不一致。

---

## 10. 关键代码文件索引

| 文件 | 说明 |
|------|------|
| [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/schema.prisma) | 完整数据模型定义 |
| [postCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/collections/postCollection.ts) | 创建收藏夹（owner 与 createdBy 可分离） |
| [postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/links/postLink.ts) | 创建链接 |
| [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts) | 删除收藏夹（含递归子收藏夹） |
| [deleteLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts) | 删除链接 |
| [deleteTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts) | 删除标签 |
| [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts) | 删除用户 |
| [createOrUpdateTags.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/createOrUpdateTags.ts) | 标签 upsert（依赖唯一约束） |
| [mergeTags.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/mergeTags.ts) | 标签合并（事务内删旧创新） |
| [migrations/](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/) | 全部迁移历史目录 |
