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

## 3. deleteUserById 代码级分支逻辑深度分析

代码位置: [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts)

调用入口: [pages/api/v1/users/[id]/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/pages/api/v1/users/[id]/index.ts#L86-L91)

### 3.1 函数签名与参数含义

```typescript
export default async function deleteUserById(
  userId: number,        // 当前已认证登录用户的 ID
  body: DeleteUserBody,   // 请求体，含 password 和可选的 cancellation_details
  isServerAdmin: boolean, // 当前用户是否为系统管理员
  queryId: number         // URL 路径中的目标用户 ID（被操作对象）
)
```

**参数来源（路由层）** [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/pages/api/v1/users/[id]/index.ts#L12-L34):

| 参数 | 赋值方式 | 含义 |
|------|---------|------|
| `userId` | `token.id` | 从 JWT 解析的**当前会话登录用户**ID |
| `queryId` | `Number(req.query.id)` | URL `/api/v1/users/{id}` 中的**目标操作对象**ID |
| `isServerAdmin` | `user?.id === Number(process.env.NEXT_PUBLIC_ADMIN \|\| 1)` | 当前登录用户 ID 是否等于配置的管理员 ID（默认 1） |
| `body` | `req.body` | 请求 JSON，类型见下方 |

**DeleteUserBody 类型定义** [global.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/types/global.ts#L152-L158):
```typescript
export type DeleteUserBody = {
  password: string; // 必填，用于验证用户身份（管理员删除时可能绕过）
  cancellation_details?: {
    comment?: string;
    feedback?: Stripe.SubscriptionCancelParams.CancellationDetails.Feedback;
  };
};
```

### 3.2 queryId 与 userId 的使用差异对照

| 代码位置 | 使用变量 | 用途 |
|---------|---------|------|
| L16-L17 | `userId` | `prisma.user.findUnique({ where: { id: userId } })` — 查询**当前登录用户**信息，用于后续密码校验和订阅判断 |
| L40 | `queryId === userId` | 判断操作类型：删除自己 / 删除别人 |
| L74-L76 | `queryId` + `userId` 对应的 subscription | 查询目标用户 `queryId` 是否属于当前用户 `userId` 的订阅子账号 |
| L84-L91 | `queryId` (findChild.id) | 子账号移除场景：**仅 disconnect，不删除账号** |
| L112 | `queryId` | `where: { collection: { ownerId: queryId } }` — 查询目标用户拥有的收藏夹下的所有 Link |
| L121 | `queryId` | `where: { ownerId: queryId }` — 查询目标用户拥有的所有 Collection |
| L133 | `queryId` | `uploads/avatar/${queryId}.jpg` — 删除目标用户头像 |
| L156 | `queryId !== userId` | Stripe 分支：管理员删除他人时，查目标用户的订阅并取消 |
| L157-L158 | `queryId` | `where: { userId: queryId }` — 查询目标用户的 stripeSubscriptionId |
| L173 | `queryId === userId` | Stripe 分支：用户删除自己时，取消自己的订阅 |
| L199-L200 | `queryId` | `prisma.user.delete({ where: { id: queryId } })` — **最终删除目标用户** |

**核心区别**:
- `userId` = 「我是谁」（操作者身份）—— 用于权限校验、密码比对、订阅归属判断
- `queryId` = 「操作谁」（目标对象）—— 用于数据查询、文件删除、Stripe 操作、最终删除数据库记录

### 3.3 完整决策树（管理员 vs 非管理员路径）

```
deleteUserById(userId, body, isServerAdmin, queryId)
│
├─ Step 1: 查询当前用户（L16-L30）
│   prisma.user.findUnique({
│     where: { id: userId },        // 注意：查的是 userId（当前登录用户），不是 queryId
│     include: { subscriptions, parentSubscription }
│   })
│   └─ user 不存在 → return 404 "Invalid credentials."
│
├─ Step 2: 权限与身份校验（L39-L105）
│   │
│   ├─ ═══════ 管理员路径 (isServerAdmin === true) ═══════
│   │   跳过 Step 2 的所有校验，直接进入 Step 3 删除事务
│   │
│   └─ ═══════ 普通用户路径 (isServerAdmin === false) ═══════
│       │
│       ├─ Branch 2A: queryId === userId —— 用户删除自己
│       │   │
│       │   ├─ user.password 存在（非 OAuth 用户）
│       │   │   └─ bcrypt.compareSync(body.password, user.password)
│       │   │       ├─ 密码错误 → return 401 "Invalid credentials."
│       │   │       └─ 密码正确 → 通过，进入 Step 3
│       │   │
│       │   └─ user.password 不存在（OAuth 用户如 Google/GitHub 登录）
│       │       └─ return 401 "User has no password. Please reset..."
│       │
│       └─ Branch 2B: queryId !== userId —— 用户尝试删除别人
│           │
│           ├─ user.parentSubscriptionId 存在（当前登录用户自己是子账号）
│           │   └─ return 401 "Permission denied."（子账号无权删除任何人）
│           │
│           └─ user.parentSubscriptionId 不存在（当前用户是订阅主账号）
│               │
│               ├─ !user.subscriptions → return 401 "User has no subscription."
│               │
│               ├─ 查找 findChild: { id: queryId, parentSubscriptionId: user.subscriptions.id }
│               │   └─ !findChild → return 401 "Permission denied."（目标不是自己的子账号）
│               │
│               └─ findChild 存在（确认是自己的子账号）
│                   ├─ prisma.user.update({ where: { id: findChild.id },
│                   │      data: { parentSubscription: { disconnect: true } } })
│                   │      └─ ⚠️ 仅解除订阅关联，不删除子账号本身
│                   ├─ findChild.emailVerified
│                   │   └─ updateSeats(stripeSubscriptionId, quantity - 1) 减少 Stripe 席位
│                   └─ return 200 "Account removed from subscription."
│                          └─ ⚠️⚠️⚠️ 提前返回！不执行 Step 3 的删除事务
│
└─ Step 3: 删除事务（L107-L205）
    仅 isServerAdmin=true 或 Branch 2A（用户删自己且密码正确）能到达此处
```

### 3.4 子账号移除时的提前返回（Branch 2B）

代码位置: [deleteUserById.ts#L60-L103](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L60-L103)

**触发条件**:
1. 当前用户不是管理员
2. `queryId !== userId`（删的不是自己）
3. 当前用户自己不是子账号（`!user.parentSubscriptionId`）
4. 当前用户有订阅（`user.subscriptions` 存在）
5. 目标用户 `queryId` 是当前用户的子账号（通过 `parentSubscriptionId` 关联）

**执行动作**:
```typescript
// 仅解除关联，不删除用户
const removeUser = await prisma.user.update({
  where: { id: findChild.id },
  data: { parentSubscription: { disconnect: true } },
});

// 邮箱已验证的用户才占用席位，释放一个席位
if (removeUser.emailVerified)
  await updateSeats(user.subscriptions.stripeSubscriptionId, quantity - 1);

return { response: "Account removed from subscription.", status: 200 };
```

**结果**:
- 子账号**仍然存在**于数据库中，只是 `parentSubscriptionId` 被置为 NULL
- 父账号的 Stripe 席位减少 1（若子账号已验证邮箱）
- 子账号的所有 Collection、Link、Tag 数据**完全保留**
- 子账号后续登录时会因为没有有效订阅而被拦截（参考路由层 [index.ts#L43-L65](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/pages/api/v1/users/[id]/index.ts#L43-L65) 的 `verifySubscription` 检查）
- **不进入 Step 3 的删除事务**，因此不会触发任何 createdById 级联

### 3.5 删除事务内部逻辑（Step 3）

代码位置: [deleteUserById.ts#L107-L205](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L107-L205)

事务配置: `{ timeout: 20000 }`（20 秒超时）

```
$transaction(async (prisma) => {
  │
  ├─ 3.1 清理 Meilisearch 索引（L111-L118）
  │   where: { collection: { ownerId: queryId } }
  │   ⚠️ 只查目标用户「拥有的收藏夹」下的 Link，遗漏了 createdById=queryId 但属于别人收藏夹的 Link
  │   await meiliClient.index("links").deleteDocuments(linkIds)
  │
  ├─ 3.2 清理归档文件目录（L120-L131）
  │   where: { ownerId: queryId } → 查目标用户拥有的所有 Collection
  │   for each collection:
  │     removeFolder("archives/{collectionId}")
  │     removeFolder("archives/preview/{collectionId}")
  │   ⚠️ 同样遗漏：别人收藏夹中被 createdById CASCADE 删除的 Link 的归档文件
  │
  ├─ 3.3 删除头像文件（L133）
  │   removeFile("uploads/avatar/{queryId}.jpg")
  │
  ├─ 3.4 Stripe 相关处理（L135-L196）—— 独立 try-catch 包裹
  │   ├─ 可选：发送取消原因邮件到 hello@linkwarden.app
  │   │
  │   └─ try {
  │      │
  │      ├─ Branch A: user.subscriptions?.id && queryId !== userId
  │      │   管理员删除某订阅用户 → 查目标用户的 stripeSubscriptionId 并 cancel
  │      │
  │      ├─ Branch B: user.subscriptions?.id && queryId === userId
  │      │   用户自己删自己（且自己是主订阅者）→ cancel 自己的 stripe 订阅
  │      │
  │      └─ Branch C: user.parentSubscription?.id && user.emailVerified
  │          用户是子账号自删 → updateSeats(父订阅, quantity - 1) 释放席位
  │      │
  │      } catch (err) { console.log(err) }
  │      ⚠️ Stripe 错误被单独吞掉，不影响后续删除
  │
  └─ 3.5 最终删除用户（L198-L201）
      prisma.user.delete({ where: { id: queryId } })
      └─ 触发数据库级联删除，详见第 4 节分析

}).catch((err) => console.log(err))
   ⚠️ 整个事务的任何错误都被吞掉（只 log，不抛出）

return 200 "User account and all related data deleted successfully."
```

---

## 4. createdById 在删除时的实际影响与事务报错分析

### 4.1 删除 User 的完整级联路径

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

### 4.2 事务报错对 createdById 级联判断的影响

事务中有**两层独立的错误捕获**，影响各不相同：

#### 第一层：Stripe 专用 try-catch（L155-L195）

```typescript
try {
  // Stripe 订阅取消 / updateSeats / 发邮件
} catch (err) {
  console.log(err);  // 仅打印，不抛出
}
```

**对 createdById 级联的影响**: **无影响**。Stripe 错误被单独捕获并吞掉，代码继续向下执行 `prisma.user.delete()`，createdById 级联正常触发。

#### 第二层：整个事务的 .catch（L205）

```typescript
await prisma.$transaction(
  async (prisma) => { /* ... 所有逻辑 ... */ },
  { timeout: 20000 }
).catch((err) => console.log(err));
```

**事务 ACID 特性**：如果事务内部任意位置抛出未捕获的异常，整个事务**回滚**，所有数据库操作撤销（包括 `prisma.user.delete()` 和由其触发的 createdById 级联）。

#### 各阶段报错的具体影响矩阵

| 报错阶段 | 代码位置 | 报错场景举例 | 数据库是否回滚 | createdById 级联是否执行 | 已产生的副作用（非数据库） |
|---------|---------|------------|--------------|------------------------|----------------------|
| Meilisearch 删除 | L118 | Meilisearch 服务不可用、网络超时 | ✅ 回滚 | ❌ **不执行** | ⚠️ Meilisearch 索引已删（无法恢复） |
| 删除归档目录 | L124-L131 | 文件系统权限不足、目录被占用 | ✅ 回滚 | ❌ **不执行** | ⚠️ 已删除的归档目录无法恢复 |
| 删除头像文件 | L133 | 文件不存在、权限不足 | ✅ 回滚 | ❌ **不执行** | ⚠️ 头像文件已删（如果执行到了） |
| Stripe 操作 | L155-L195 | Stripe API 超时、卡片异常 | ❌ 被内层 try-catch 吞掉，不回滚 | ✅ **正常执行** | Stripe 订阅可能已取消 / 席位已调整 |
| prisma.user.delete | L199-L201 | DB 连接中断、死锁、剩余 RESTRICT 外键冲突（理论上不存在） | ✅ 回滚 | ❌ **不执行** | 前述 Meilisearch/文件副作用已发生 |
| 事务超时 | L203 | 数据量过大、删除耗时超 20s | ✅ 回滚 | ❌ **不执行** | 前述 Meilisearch/文件副作用已发生 |

**严重的不一致风险**：由于事务外副作用（Meilisearch 删除、文件系统删除）先于数据库删除执行，且没有补偿机制，一旦事务在中间报错回滚，会出现：

| 资源 | 状态 | 结果 |
|------|------|------|
| Meilisearch 索引 | 已被删除 | 搜索时找不到这些 Link，但数据库中实际还存在 |
| 归档文件目录 | 已被删除 | 用户点击查看归档时 404，但数据库中 Link/preset 值还指向这些路径 |
| 头像文件 | 已被删除 | 用户头像显示破裂，但数据库中用户记录仍存在 |
| 数据库 | 事务回滚，User/Collection/Link/Tag **全部保留** | 数据完整性未受损，但与外部资源脱节 |

### 4.3 应用层清理 vs 数据库级联的覆盖缺口

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
- 该用户作为**成员**在别人的共享收藏夹中创建的所有 Link（collection.ownerId ≠ queryId，但 Link.createdById = queryId）

**遗漏后果**:
1. **Meilisearch 索引残留**：这些 Link 的文档不会被删除，搜索时出现孤儿记录
2. **归档文件残留**：这些 Link 的归档文件（archives/{collectionId}/{linkId}.*）不会被删除，磁盘残留
3. **与子账号移除路径的叠加**：子账号提前返回路径完全不触发数据库删除，因此也不会有 createdById CASCADE 问题，但该路径本身只做 `parentSubscription.disconnect`

### 4.4 删除 Collection / Link / Tag 时 createdById 的影响

#### 删除 Collection
代码位置: [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts)

- **Collection.createdById**：不反向影响。删除 Collection 时，Collection.createdById 是 Collection → User 的引用，删除 Collection 本身不影响其创建者 User。
- **Link.collectionId CASCADE**：删除收藏夹 → 删除其下所有 Link → Link 被删除时，Link.createdById 也不反向影响 User。

#### 删除 Link
代码位置: [deleteLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts)

- **Link.createdById**：不反向影响。Link.createdById 是 Link → User 的引用，删除 Link 不会导致 User 被删除。

#### 删除 Tag
代码位置: [deleteTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts)

- Tag 没有 createdById 字段，不涉及此问题。

---

## 5. 迁移演进过程

迁移目录: [packages/prisma/migrations/](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/)

### 5.1 初始版本（20230719_init）
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

### 5.2 唯一约束的移除

#### 收藏夹名唯一约束移除（20240218）
迁移文件: [20240218080348_allow_duplicate_collection_names/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20240218080348_allow_duplicate_collection_names/migration.sql)

```sql
DROP INDEX "Collection_name_ownerId_key";
```
同一用户可以创建多个同名收藏夹，仅靠 `id` 区分。

#### AccessToken 名称唯一约束移除（20240124）
迁移文件: [20240124201018_removed_name_unique_constraint/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20240124201018_removed_name_unique_constraint/migration.sql)

```sql
DROP INDEX "AccessToken_name_userId_key";
```

---

### 5.3 子收藏夹功能引入（20240125）
迁移文件: [20240125124457_added_subcollection_relations/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20240125124457_added_subcollection_relations/migration.sql)

```sql
ALTER TABLE "Collection" ADD COLUMN "parentId" INTEGER;
ALTER TABLE "Collection" ADD CONSTRAINT ... FOREIGN KEY ("parentId") 
    REFERENCES "Collection"("id") ON DELETE SET NULL ON UPDATE CASCADE;
```
初始策略: 删除父收藏夹时，子收藏夹的 `parentId` 置为 NULL（提升为根级）。20250318 改为 CASCADE。

---

### 5.4 createdById 字段完整迁移时间线

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

此时两者行为再次一致。

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

### 5.5 全局级联策略变更（20250318）

这是最关键的迁移系列，将大部分 `RESTRICT` / `SET NULL` 改为 `CASCADE`，但 **Collection.createdById 被遗漏**。

#### 第一批（20250318123928）
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

### 5.6 多对多连接表主键化（20250627 upgrade_to_v6）
迁移文件: [20250627132552_upgrade_to_v6/migration.sql](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/20250627132552_upgrade_to_v6/migration.sql)

```sql
ALTER TABLE "_LinkToTag" ADD CONSTRAINT "_LinkToTag_AB_pkey" PRIMARY KEY ("A", "B");
DROP INDEX "_LinkToTag_AB_unique";
ALTER TABLE "_PinnedLinks" ADD CONSTRAINT "_PinnedLinks_AB_pkey" PRIMARY KEY ("A", "B");
DROP INDEX "_PinnedLinks_AB_unique";
```
隐式多对多表从"唯一索引 + 无主键"升级为"复合主键"。

---

## 6. 业务逻辑中的补充清理（非数据库级联）

### 6.1 删除收藏夹 [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts)

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

### 6.2 删除标签 [deleteTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts)

```
├─ 权限校验（ownerId === userId）
├─ 删除 Tag（触发级联：删除所有 _LinkToTag 关联）
└─ 将受影响 Link 的 indexVersion 置 null（触发 Meilisearch 重新索引）
```

### 6.3 删除链接 [deleteLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts)

```
├─ 权限校验（所有者或有 canDelete 权限的成员）
├─ 删除 Link（触发级联：删除 Highlight、解除 Tag 关联、解除置顶）
├─ 删除归档文件
└─ 从 Meilisearch 索引删除
```

---

## 7. 唯一约束分析

### 7.1 当前存在的唯一约束

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

### 7.2 关键业务依赖的唯一约束

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

## 8. 回滚风险分析（校正版）

### 8.1 高风险迁移（不可逆或可能丢失数据）

| 迁移 | 风险等级 | 原因 |
|------|---------|------|
| **20250318 Link.createdById 改为 CASCADE** | 🔴 **极高** | 回退到 SET NULL 时，数据库中可能已经因为 CASCADE 删除了大量 Link，这些数据永久丢失。更严重的是，共享收藏夹场景中**他人的数据被意外删除**，无法恢复。此外，deleteUserById 中事务报错导致的不一致性会因级联范围扩大而加剧。 |
| **20250318 全局级联变更（其余关系）** | ⚠️ 高 | 将 RESTRICT 改为 CASCADE 是单向的。回退到 RESTRICT 会导致应用层所有删除操作因外键约束失败（代码逻辑已依赖 CASCADE 清理子表，不做手动预删除）。 |
| **20231027 删除 Account / Session 表** | 🔴 极高 | 直接 DROP TABLE，所有数据永久丢失。无回退脚本，只能依赖备份恢复。 |
| **20241026 createdById 强制 NOT NULL** | ⚠️ 高（但已被后续迁移回退） | 若历史数据存在脏数据（Link.collectionId 悬空），数据回填 SQL 返回 NULL，迁移直接失败。 |
| **20240218 移除 Collection 唯一约束** | ⚠️ 中 | 一旦用户创建了同名收藏夹，再恢复唯一约束会因数据冲突失败，需先手动清理重名数据。 |
| **20250627 多对多表主键化** | ⚠️ 中 | 如果连接表中存在重复行（尽管有唯一索引不太可能），加主键会失败。回退需删主键再建唯一索引。 |

### 8.2 Link.createdById CASCADE 的回滚专项分析

**当前状态**: Link.createdById 是 ON DELETE CASCADE

**回滚方案（假设要改回 SET NULL）**:
1. 数据库层面：`ALTER TABLE "Link" DROP CONSTRAINT ...; ALTER TABLE "Link" ADD CONSTRAINT ... ON DELETE SET NULL;`
2. 但此时已经因 CASCADE 被删除的 Link **无法恢复**
3. 而且，应用层 deleteUserById.ts 的清理逻辑也不完整（见 4.3 节），即使回滚级联策略，也需要同时修复清理代码

**更安全的方向**: 保持 SET NULL，删除用户时只清空 Link.createdById，保留链接数据（因为链接的真正归属是通过 Collection.ownerId 决定的，而非 createdById）。

### 8.3 事务报错相关风险

由于 deleteUserById 中事务外副作用（Meilisearch、文件系统）先于数据库删除执行，且整个事务的 `.catch` 只 log 不抛出，即使事务完全回滚，函数仍然返回 200 "deleted successfully"，这意味着：

1. **前端被误导**：用户以为账号已删，但实际数据库中数据还在
2. **静默不一致**：没有告警、没有重试、没有补偿逻辑，只能靠用户手动发现
3. **createdById 级联判断失真**：调用方以为 createdById 已清理，实际数据库未动

### 8.4 唯一约束回滚冲突模式

典型场景（Tag 为例，目前仍保留唯一约束）:
1. 用户创建标签 "work"
2. 某迁移临时移除 `Tag(name, ownerId)` 唯一约束
3. 用户在此期间又创建了一个名为 "work" 的标签
4. 回滚迁移恢复唯一约束 → 因已有重名数据，迁移失败

**Collection 已实际发生此模式**: 20240218 已永久移除唯一约束，且代码逻辑不再依赖。

### 8.5 Prisma 迁移本身的回滚限制

Prisma Migrate **不提供自动回滚机制**。每个 `migration.sql` 仅包含正向变更。回滚方案:
- 依赖数据库备份（PITR，时间点恢复）
- 手动编写反向迁移 SQL
- 使用 `prisma migrate resolve` 标记失败迁移为已应用/已回滚

---

## 9. 历史数据兼容性总结

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

## 10. 关键发现与建议总结

### 10.1 已确认的代码行为

1. **Collection.createdById 与 Link.createdById 行为不一致**
   - Collection.createdById → SET NULL（schema.prisma 未显式指定，Prisma 对 nullable 外键默认为 SetNull，与最后一次迁移 20241030 一致）
   - Link.createdById → CASCADE（schema.prisma 显式指定 `onDelete: Cascade`，迁移 20250318 修改）
   - 这种不一致可能是 20250318 全局级联变更时**遗漏了 Collection.createdById**

2. **共享收藏夹场景下 owner ≠ createdBy**
   - [postCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/collections/postCollection.ts#L28-L92) 中，rootOwnerId（收藏夹所有者）与 userId（创建者）可以不同
   - Link 同理：成员在共享收藏夹中创建的链接，collection.ownerId ≠ link.createdById

3. **deleteUserById 的三条主要路径**
   - **管理员路径**：跳过所有校验，直接执行删除事务
   - **普通用户删自己**：需密码验证（OAuth 用户被拒），通过后执行删除事务
   - **普通用户删子账号**：仅 `parentSubscription.disconnect` + 减席位，**提前 return，不执行删除事务，不触发 createdById 级联**

4. **删除用户存在外部资源清理遗漏**
   - [deleteUserById.ts#L111-L118](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L111-L118) 只按 collection.ownerId 查找 Link
   - 遗漏了 createdById = queryId 但属于他人收藏夹的 Link → Meilisearch 孤儿索引 + 归档文件残留

5. **事务报错导致严重不一致**
   - 事务外副作用（Meilisearch、文件系统）先于数据库删除执行
   - 整个事务的 `.catch` 只 log 不抛出，函数始终返回 200
   - 一旦事务中间报错回滚，Meilisearch 和文件已删，但数据库保留，前端被误导

### 10.2 建议

1. **统一 createdById 的级联策略**：考虑将 Link.createdById 也改为 SET NULL，与 Collection.createdById 保持一致。链接的归属应由 Collection.ownerId 决定，而不是创建者。删除用户不应导致他人收藏夹中的数据丢失。

2. **修复 deleteUserById 的清理范围**：
   ```typescript
   // 补充查询 createdById = queryId 的 Link
   const createdLinks = await prisma.link.findMany({
     where: { createdById: queryId },
     select: { id: true, collectionId: true }
   });
   // 一并清理 Meilisearch 索引和归档文件
   ```

3. **在 schema.prisma 中为 Collection.createdBy 显式添加 `onDelete: SetNull`**：消除歧义，避免未来 Prisma 默认行为变更导致不一致。

4. **事务报错处理优化**：
   - 事务外副作用（Meilisearch、文件系统）应放到事务成功提交之后执行，或实现 Saga 补偿模式
   - `.catch` 不应静默吞掉错误，应至少抛出 500 状态码让前端感知失败
   - 考虑将 20 秒超时延长，或分批处理大账户删除

---

## 11. 关键代码文件索引

| 文件 | 说明 |
|------|------|
| [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/schema.prisma) | 完整数据模型定义 |
| [users/[id]/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/pages/api/v1/users/[id]/index.ts) | 用户 API 路由层（参数来源） |
| [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts) | 删除用户核心逻辑（分支、事务、级联触发点） |
| [postCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/collections/postCollection.ts) | 创建收藏夹（owner 与 createdBy 可分离） |
| [postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/links/postLink.ts) | 创建链接 |
| [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts) | 删除收藏夹（含递归子收藏夹） |
| [deleteLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts) | 删除链接 |
| [deleteTagById.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/tagId/deleteTagById.ts) | 删除标签 |
| [createOrUpdateTags.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/createOrUpdateTags.ts) | 标签 upsert（依赖唯一约束） |
| [mergeTags.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/controllers/tags/mergeTags.ts) | 标签合并（事务内删旧创新） |
| [updateSeats.ts](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/apps/web/lib/api/stripe/updateSeats.ts) | Stripe 席位增减 |
| [global.ts (DeleteUserBody)](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/types/global.ts#L152-L158) | DeleteUserBody 类型定义 |
| [migrations/](file:///d:/fz/0601/solo-dogfeeding/code/94-linkwarden/packages/prisma/migrations/) | 全部迁移历史目录 |
