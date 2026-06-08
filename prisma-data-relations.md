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
- 1:N → `collections` (拥有的收藏夹)
- 1:N → `tags` (拥有的标签)
- N:M → `pinnedLinks` (置顶的链接，通过 `_PinnedLinks` 连接表)
- 1:N → `createdLinks` (创建的链接)
- 1:N → `createdCollections` (创建的收藏夹)
- 1:N → `highlights` (高亮)
- N:M → `collectionsJoined` (加入的收藏夹，通过 `UsersAndCollections`)
- 1:1 → `subscriptions` (订阅)
- 1:N → `whitelistedUsers`、`accessTokens`、`dashboardSections`、`accounts`

---

### 1.2 Collection（收藏夹）模型
定义位置: [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/schema.prisma#L126-L149)

**核心字段**:
- `id` (Int, PK)
- `name` (String) — **注意：无唯一约束（曾有后移除）**
- `description` (String)
- `color` (String, default: "#0ea5e9")
- `icon`, `iconWeight` (String?)
- `parentId` (Int?) — 支持自引用层级结构
- `ownerId` (Int, NOT NULL)
- `createdById` (Int?)
- `isPublic` (Boolean)

**关系**:
- N:1 → `owner` (所属用户，`onDelete: Cascade`)
- N:1 → `parent` (父收藏夹，自引用，`onDelete: Cascade`)
- 1:N → `subCollections` (子收藏夹)
- N:1 → `createdBy` (创建者，`onDelete: Cascade`)
- N:M → `members` (成员用户，通过 `UsersAndCollections`)
- 1:N → `links` (包含的链接)
- 1:N → `rssSubscriptions`、`DashboardSection`

**索引**:
- `@@index([ownerId])`

---

### 1.3 Link（链接）模型
定义位置: [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/schema.prisma#L166-L198)

**核心字段**:
- `id` (Int, PK)
- `name` (String, default: "")
- `type` (String, default: "url") — url / pdf / image 等
- `url` (String?)
- `description` (String)
- `collectionId` (Int, NOT NULL)
- `createdById` (Int?)
- 归档相关字段: `image`, `pdf`, `readable`, `monolith`, `preview`, `textContent`
- `icon`, `iconWeight`, `color`
- `aiTagged` (Boolean, default: false)
- `indexVersion` (Int?) — Meilisearch 索引版本
- `lastPreserved`, `importDate` (DateTime?)
- `clientSide` (Boolean)

**关系**:
- N:1 → `collection` (所属收藏夹，`onDelete: Cascade`)
- N:1 → `createdBy` (创建者用户，`onDelete: Cascade`)
- N:M → `pinnedBy` (置顶它的用户，通过 `_PinnedLinks`)
- N:M → `tags` (关联的标签，通过 `_LinkToTag`)
- 1:N → `highlight` (高亮)

**索引**:
- `@@index([collectionId])`

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
- N:1 → `owner` (所属用户，`onDelete: Cascade`)
- N:M → `links` (关联的链接，通过 `_LinkToTag`)

**唯一约束与索引**:
- `@@unique([name, ownerId])` — **同一用户下标签名必须唯一**
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
User (1) ──────< Collection (N)
  │                │  ^
  │                │  │ parent/subCollections (自引用)
  │                v  │
  │              Link (N) >──────< Tag (N) ──────> User (1)
  │                │                  (owner)
  │                │ pinnedBy
  │                v
  └─────────────< UsersAndCollections >─────────────┘
                   (成员权限表)
```

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
| **Collection.ownerId** | **RESTRICT** |
| **UsersAndCollections.userId** | **RESTRICT** |
| **UsersAndCollections.collectionId** | **RESTRICT** |
| **Link.collectionId** | **RESTRICT** |
| **Tag.ownerId** | **RESTRICT** |
| _LinkToUser / _LinkToTag | CASCADE |

**初始唯一约束**:
- `Collection(name, ownerId)` — 同一用户下收藏夹名唯一
- `Tag(name, ownerId)` — 同一用户下标签名唯一
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
初始策略: 删除父收藏夹时，子收藏夹的 `parentId` 置为 NULL（提升为根级）。

---

### 3.4 createdBy 字段的引入与反复

#### 阶段一: 字段添加（前期迁移）
为 Collection 和 Link 添加 `createdById`，允许 NULL。

#### 阶段二: 强制非空（20241026）
迁移文件: [20241026161909_assign_createdby_to_collection_owners_and_make_field_required/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20241026161909_assign_createdby_to_collection_owners_and_make_field_required/migration.sql)

**数据填充逻辑**:
```sql
UPDATE "Link" SET "createdById" = (
  SELECT "ownerId" FROM "Collection" WHERE "Collection"."id" = "Link"."collectionId"
);
UPDATE "Collection" SET "createdById" = "ownerId";
```
然后将字段设为 NOT NULL，外键为 `ON DELETE RESTRICT`。

#### 阶段三: 回退为可空（20241030）
迁移文件: [20241030200844_createdby_fields_can_be_null/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20241030200844_createdby_fields_can_be_null/migration.sql)

```sql
ALTER TABLE "Collection" ALTER COLUMN "createdById" DROP NOT NULL;
ALTER TABLE "Link" ALTER COLUMN "createdById" DROP NOT NULL;
-- 外键改为 ON DELETE SET NULL
```

**原因推测**: 导入外部数据时可能不存在创建者，或创建者被删除后需要保留数据。

---

### 3.5 全局级联策略变更（20250318）

这是最关键的迁移系列，将大部分 `RESTRICT` / `SET NULL` 改为 `CASCADE`。

#### 第一批（20250318123928）
迁移文件: [20250318123928_add_referential_actions_to_certain_fields/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20250318123928_add_referential_actions_to_certain_fields/migration.sql)

改为 CASCADE 的关系:
- `Collection.parentId` (SET NULL → CASCADE)
- `UsersAndCollections.userId` (RESTRICT → CASCADE)
- `UsersAndCollections.collectionId` (RESTRICT → CASCADE)
- `Link.createdById` (RESTRICT → CASCADE)
- `Link.collectionId` (RESTRICT → CASCADE)
- `Tag.ownerId` (RESTRICT → CASCADE)
- `Subscription.userId` (RESTRICT → CASCADE)
- `AccessToken.userId` (RESTRICT → CASCADE)
- `RssSubscription.collectionId` (RESTRICT → CASCADE)
- `Highlight.linkId`, `Highlight.userId` (RESTRICT → CASCADE)

#### 第二批（20250318130241）
迁移文件: [20250318130241_add_referential_action_to_field/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20250318130241_add_referential_action_to_field/migration.sql)

- `WhitelistedUser.userId` → CASCADE

#### 第三批（20250318131012）
迁移文件: [20250318131012_add_referential_action_to_field/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20250318131012_add_referential_action_to_field/migration.sql)

- `Collection.ownerId` (RESTRICT → CASCADE) — **最后一个核心关系切换**

---

### 3.6 多对多连接表主键化（20250627 upgrade_to_v6）
迁移文件: [20250627132552_upgrade_to_v6/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20250627132552_upgrade_to_v6/migration.sql)

```sql
ALTER TABLE "_LinkToTag" ADD CONSTRAINT "_LinkToTag_AB_pkey" PRIMARY KEY ("A", "B");
DROP INDEX "_LinkToTag_AB_unique";
ALTER TABLE "_PinnedLinks" ADD CONSTRAINT "_PinnedLinks_AB_pkey" PRIMARY KEY ("A", "B");
DROP INDEX "_PinnedLinks_AB_unique";
```
**意义**: 隐式多对多表从"唯一索引 + 无主键"升级为"复合主键"，提高查询性能与数据完整性。

---

## 4. 当前级联行为总览

基于 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/schema.prisma) 的当前状态:

| 操作 | 级联效果 |
|------|---------|
| **删除 User** | 删除其拥有的所有 Collection、Tag、Subscription、AccessToken、WhitelistedUser、Highlight、DashboardSection、Account、UsersAndCollections 关系；其创建的 Link 的 createdById 置空（但 Link 本身因 collection 已删而级联删除） |
| **删除 Collection** | 删除其下所有 Link、RssSubscription、DashboardSection、UsersAndCollections 关系；所有子 Collection 递归级联删除；子 Collection 的 Link 也随之删除 |
| **删除 Link** | 删除其所有 Highlight；解除与所有 Tag 的关联（`_LinkToTag` 级联）；解除所有用户的置顶（`_PinnedLinks` 级联） |
| **删除 Tag** | 解除该 Tag 与所有 Link 的关联（`_LinkToTag` 级联） |
| **删除 UsersAndCollections** | 仅解除成员关系，不影响 User 或 Collection 本身 |

---

## 5. 业务逻辑中的补充清理（非数据库级联）

数据库级联无法覆盖文件系统、搜索引擎索引等外部资源，因此应用层做了额外处理：

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
├─ 查询该用户所有 Collection 下的 Link，从 Meilisearch 索引删除
├─ 删除所有归档文件目录
├─ 删除用户头像文件
├─ 处理 Stripe 订阅取消/席位变更
└─ 删除 User（触发数据库级联：Collection → Link → ... 整条链）
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
| Tag | `(name, ownerId)` | 同一用户下标签名唯一 |
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
**依赖假设**: `name_ownerId` 必须是唯一索引，否则 upsert 无法工作。

---

## 7. 回滚风险分析

### 7.1 高风险迁移（不可逆或可能丢失数据）

| 迁移 | 风险 | 原因 |
|------|------|------|
| **20250318 级联策略变更** | ⚠️ 高 | 将 RESTRICT 改为 CASCADE 是单向的。回退时数据库中可能已有被级联删除后产生的"孤立"引用数据，但实际不存在了。此外，删除用户/收藏夹行为已依赖 CASCADE，回退到 RESTRICT 会导致删除操作全部失败。 |
| **20240218 移除 Collection 唯一约束** | ⚠️ 中 | 一旦用户创建了同名收藏夹，再恢复唯一约束会因数据冲突而失败。需要先手动清理重名数据。 |
| **20241026 createdById 强制 NOT NULL** | ⚠️ 高（但已被后续迁移回退） | 若历史数据中存在 NULL 值，该迁移直接失败。该迁移包含了数据回填 SQL，但如果 Link 引用的 Collection 不存在，会导致 createdById 为 NULL，迁移失败。 |
| **20231027 删除 Account / Session 表** | 🔴 极高 | 直接 DROP TABLE，所有数据永久丢失。无回退脚本，只能依赖备份恢复。 |
| **20250627 多对多表主键化** | ⚠️ 中 | 如果连接表中存在重复行（尽管有唯一索引不太可能），加主键会失败。回退需删主键再建唯一索引。 |

### 7.2 数据迁移回填风险

**20241026 迁移中的数据回填**:
```sql
UPDATE "Link" SET "createdById" = (
  SELECT "ownerId" FROM "Collection" WHERE "Collection"."id" = "Link"."collectionId"
);
```
**风险点**: 如果存在脏数据（Link.collectionId 指向不存在的 Collection），子查询返回 NULL，而后续 `SET NOT NULL` 会导致迁移失败。

### 7.3 唯一约束回滚冲突模式

典型场景（Tag 为例，目前仍保留唯一约束）:
1. 用户创建标签 "work"
2. 某迁移临时移除 `Tag(name, ownerId)` 唯一约束
3. 用户在此期间又创建了一个名为 "work" 的标签
4. 回滚迁移恢复唯一约束 → 因已有重名数据，迁移失败

**Collection 已实际发生此模式**: 20240218 已永久移除唯一约束，且代码逻辑不再依赖。

### 7.4 Prisma 迁移本身的回滚限制

Prisma Migrate **不提供自动回滚机制**。每个 `migration.sql` 仅包含正向变更。回滚方案:
- 依赖数据库备份（PITR）
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

## 9. 关键代码文件索引

| 文件 | 说明 |
|------|------|
| [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/schema.prisma) | 完整数据模型定义 |
| [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts) | 删除收藏夹（含递归子收藏夹） |
| [deleteLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts) | 删除链接 |
| [deleteTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts) | 删除标签 |
| [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts) | 删除用户 |
| [createOrUpdateTags.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/createOrUpdateTags.ts) | 标签 upsert（依赖唯一约束） |
| [mergeTags.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/mergeTags.ts) | 标签合并（事务内删旧创新） |
| [migrations/](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/) | 全部迁移历史目录 |
