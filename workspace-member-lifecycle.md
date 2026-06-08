# Workspace / Member 生命周期状态分析

## 1. 核心数据模型

### 1.1 用户与订阅的层级关系

在 Linkwarden 中，"工作区"的概念通过 **Subscription（订阅）** 模型实现，而不是独立的 Workspace 表。

- [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/prisma/schema.prisma) 中的关键关系：

| 模型 | 字段 | 关系 | 说明 |
|------|------|------|------|
| `User` | `parentSubscriptionId` → `Subscription.id` | 多对一 | 被邀请的成员（child user） |
| `User` | `subscriptions` → `Subscription` | 一对一 | 订阅所有者（owner） |
| `Subscription` | `userId` → `User.id` | 一对一（unique） | 订阅所属的用户 |
| `Subscription` | `childUsers` → `User[]` | 一对多 | 订阅下的成员列表 |
| `Subscription` | `quantity` | Int | 座位数（seats），即允许的最大成员数（含 owner） |

```
Subscription (owner = User A, quantity = 5)
├── User A (owner)  ← subscriptions.id = X
├── User B          ← parentSubscriptionId = X
├── User C          ← parentSubscriptionId = X
└── ... 最多 (quantity - 1) 个 child users
```

### 1.2 集合（Collection）层面的成员关系

- [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/prisma/schema.prisma#L126-L164)：

| 模型 | 字段 | 说明 |
|------|------|------|
| `Collection` | `ownerId` → `User.id` | 集合所有者，拥有全部权限 |
| `UsersAndCollections` | `userId + collectionId`（复合主键） | 集合成员关系 |
| `UsersAndCollections` | `canCreate / canUpdate / canDelete` | 细粒度权限（布尔值） |

**注意**：订阅层的"成员身份"与集合层的"成员权限"是**两个独立维度**：
- 订阅成员（同一 Subscription 下的用户）共享资源池和链接额度
- 集合成员需要在每个 Collection 上单独赋予，不自动继承订阅身份

---

## 2. 邀请流程

### 2.1 完整邀请链路

**入口**：[InviteModal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/components/ModalContent/InviteModal.tsx)

```
步骤1: 订阅 Owner 填写被邀请人邮箱
    ↓
步骤2: 调用 POST /api/v1/users (invite=true)
    ↓  [postUser.ts]
    ├── 权限校验：需 stripe + email 均启用
    ├── 权限校验：当前用户必须已登录
    ├── 创建新 User，emailVerified = null（未验证）
    ├── 关联 parentSubscriptionId = 当前用户的订阅
    └── 返回新用户信息
    ↓
步骤3: signIn("invite", { email, callbackUrl: "/member-onboarding" })
    ↓  [[...nextauth].ts 的 invite EmailProvider]
    ├── 查找：email 匹配、且 emailVerified = null、且有 parentSubscription 的用户
    ├── 获取 parentSubscriptionEmail（订阅所有者邮箱，用于邮件展示）
    ├── 限流：5 分钟内最多 4 次
    └── 调用 sendInvitationRequest 发送邀请邮件
    ↓
步骤4: 被邀请人点击邮件链接 /callback/email?token=...&email=...
    ↓  [NextAuth signIn callback]
    ├── 检查 VerificationToken 是否有效且未过期
    ├── 若用户 emailVerified = null（首次接受邀请）：
    │   ├── 统计已验证 child users 数量
    │   ├── 若 (已验证数 + 2) > subscription.quantity
    │   │   └── 调用 updateSeats() 自动扩容 Stripe 订阅座位数
    │   └── 此时 emailVerified 仍为 null（稍后更新）
    └── 签发会话，跳转 /member-onboarding
    ↓
步骤5: [member-onboarding.tsx] 用户填写 name + password
    ↓  [updateUserById.ts]
    ├── 检测 isInvited = (name===null && parentSubscriptionId && !password)
    ├── 校验 password 必填
    ├── 更新 name、password（加密）
    └── **注意：此处并未自动设置 emailVerified = new Date()**
```

### 2.2 邀请过期机制

邀请令牌的有效期定义在 [[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L138-L142)：

- `maxAge: 1200` → 令牌有效期 **20 分钟**
- `VerificationToken.expires` 记录具体过期时间
- [verify-email.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/pages/api/v1/auth/verify-email.ts#L30-L37)：验证时检查 `expires > new Date()`，过期返回 "Invalid token"
- 5 分钟内同一邮箱最多发送 4 次邀请（防止滥用）

---

## 3. 加入与激活

### 3.1 新用户（被邀请）的激活状态判断

[updateUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserById.ts#L170-L177) 中定义的 `isInvited` 条件：

```typescript
const isInvited =
  user?.name === null && user.parentSubscriptionId && !user.password;
```

满足以下全部条件即视为"待激活的被邀请用户"：
1. `name` 为空（系统初始值 `null`）
2. 有 `parentSubscriptionId`（说明是被邀请加入的）
3. 没有设置 `password`

此时用户必须设置密码才能继续使用系统。

### 3.2 邮箱验证

- 自注册用户：通过 EmailProvider 发送验证邮件，验证后 `emailVerified` 设置
- SSO 用户：[[...nextauth].ts jwt callback](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1427-L1453) 自动设置 `emailVerified = new Date()`
- 凭证登录：必须 `emailVerified` 不为 `null` 才能登录（如 email provider 启用）
- **被邀请用户的 emailVerified**：当前代码路径中，在 member-onboarding 设置密码时**并未自动置为已验证**，这是一个潜在的不一致点

---

## 4. 退出 / 被移除

### 4.1 三种退出场景

[deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts) 中处理了三种情况：

| 场景 | queryId 与 userId 的关系 | 行为 |
|------|--------------------------|------|
| **用户自删** | `queryId === userId` | 删除整个账号及所有关联数据 |
| **订阅 Owner 移除成员** | `queryId !== userId` 且 `user.parentSubscriptionId != null` | 仅断开订阅关联，保留用户账号 |
| **Server Admin 操作** | `isServerAdmin === true` | 直接删除目标用户 |

### 4.2 场景一：订阅 Owner 移除成员（软移除）

[deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L60-L104)：

```
条件：
  - 操作者是 Subscription 的 owner（通过 parentSubscriptionId 反向查找确认）
  - 被操作人是该订阅的 child user

操作：
  1. prisma.user.update({ disconnect: parentSubscription })
     └── 仅清空 parentSubscriptionId，用户账号保留
  2. 若被移除成员 emailVerified 已验证
     └── updateSeats(stripeSubscriptionId, quantity - 1)
         └── 同步减少 Stripe 订阅的座位数
```

被移除后用户的状态：
- 账号仍然存在，可以独立注册自己的订阅
- 不再能访问原 Owner 订阅下的资源额度
- 之前通过 `UsersAndCollections` 赋予的集合成员权限**不会自动清理**

### 4.3 场景二：用户自主删号（硬删除）

[deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L107-L205)：

在 Prisma Transaction 内执行：

```
1. 删除 MeiliSearch 中所有本人拥有的链接索引
2. 遍历所有 owned Collection：
   └── 删除 archives/{collectionId} 和 archives/preview/{collectionId} 文件夹
3. 删除用户头像文件 uploads/avatar/{userId}.jpg
4. Stripe 处理（若启用）：
   ├── 若是删除他人且对方是订阅 owner → 取消其 Stripe 订阅
   ├── 若是自删且自己是订阅 owner → 取消自己的 Stripe 订阅
   └── 若是自删且自己是 child user 且已验证 → 减少父订阅的 seats
5. prisma.user.delete({ id }) → 级联删除：
   ├── Account[] （onDelete: Cascade）
   ├── Collection[] (ownerId → onDelete: Cascade)
   ├── Tag[] (ownerId → onDelete: Cascade)
   ├── UsersAndCollections[] (userId → onDelete: Cascade)
   ├── WhitelistedUser[] (onDelete: Cascade)
   ├── AccessToken[] (onDelete: Cascade)
   ├── Highlight[] (userId → onDelete: Cascade)
   ├── DashboardSection[] (userId → onDelete: Cascade)
   └── 以及间接通过 Collection 删除的 Link[] 等
```

---

## 5. Owner 离开工作区

### 5.1 订阅 Owner 删号的影响

当 Subscription 的 owner（即 `Subscription.userId` 对应的 User）被删除时：

**Cascade 链**（由 Prisma schema 定义）：

```
User (owner) 被删除
├── Subscription.userId → onDelete: Cascade
│   └── Subscription 被删除
│       └── User.parentSubscriptionId → onDelete: SetNull（需验证）
│           └── 所有 child users 的 parentSubscriptionId 被置为 null
│               └── 这些成员变为独立用户，但各自的账号保留
├── Collection.ownerId → onDelete: Cascade
│   └── 所有 owned Collection 被删除
│       ├── Link.collectionId → onDelete: Cascade → 链接全部删除
│       ├── UsersAndCollections.collectionId → onDelete: Cascade → 成员关系清除
│       ├── RssSubscription.collectionId → onDelete: Cascade
│       └── DashboardSection.collectionId → onDelete: Cascade
├── Tag.ownerId → onDelete: Cascade
├── Highlight.userId → onDelete: Cascade
└── ...其他关联数据
```

**关键问题**：Owner 离开时**没有任何所有权转移机制**。
- 所有该 Owner 创建的 Collection、Link、Tag 全部被删除
- Child users 的账号虽然保留，但失去订阅关联，需要各自重新订阅

---

## 6. 角色与权限体系

### 6.1 两个层级的权限

Linkwarden 有**两套独立的权限体系**：

#### 层级 A：订阅层级（Workspace 级）

无显式角色字段，通过关系推断：

| 身份 | 判断条件 | 能力 |
|------|----------|------|
| **订阅 Owner** | `User.subscriptions != null` | 邀请/移除成员、管理订阅 billing、查看全部成员 |
| **订阅成员** | `User.parentSubscriptionId != null` | 共享链接额度、可被加入集合 |
| **独立用户** | 两者皆无 | 仅使用自己的资源 |

相关代码：
- [getUsers.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/getUsers.ts)：Owner 查询同一订阅下的所有成员（`OR: [{parentSubscriptionId}, {subscriptions.id}]`）
- [verifySubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/stripe/verifySubscription.ts)：`parentSubscription.active` 或 `subscriptions.active` 视为有效订阅

#### 层级 B：集合层级（Collection 级）

通过 `UsersAndCollections` 表的三个布尔字段控制：

| 权限字段 | 含义 |
|----------|------|
| `canCreate` | 可在集合中创建子集合和链接 |
| `canUpdate` | 可修改集合信息和链接 |
| `canDelete` | 可删除集合和链接 |

另外，`Collection.ownerId` 对应的用户**隐式拥有全部权限**，无需在 `UsersAndCollections` 中存在记录。

权限检查代码：
- [getPermission.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/getPermission.ts#L28-L37)：`OR: [{ownerId: userId}, {members: {some: {userId}}}]`
- [updateCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts#L34-L35)：仅 owner 可修改集合
- [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L19-L49)：
  - 成员调用删除 API → 实际是"离开集合"（删除 UsersAndCollections 记录）
  - Owner 调用删除 API → 真正删除集合

### 6.2 子集合的权限继承

创建子集合时：[postCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/collections/postCollection.ts#L61-L82)

```
1. 调用 getCollectionRootOwnerAndMembers(parentId)
2. 沿 parent 链向上遍历到根集合
3. 根集合的 ownerId 成为新子集合的 ownerId
4. 收集路径上所有集合的 members，合并权限（OR 逻辑）
5. 将所有这些成员和权限赋予新子集合
```

更新集合成员时的传播选项：[updateCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts#L68-L121)

- 若 `propagateToSubcollections = true`，递归遍历所有后代集合
- 删除后代集合的全部现有成员
- 将当前集合的成员（排除后代集合自己的 owner）复制到后代集合

---

## 7. 订阅限制（Seats & Capacity）

### 7.1 座位数（Seats）限制

**座位数 = Subscription.quantity**，包含订阅 owner 本人。

座位检查发生在两处：

#### A. 邀请成员接受时

[[...nextauth].ts signIn callback](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1326-L1364)：

```typescript
const verifiedChildUsersCount = await prisma.user.count({
  where: {
    parentSubscriptionId,
    id: { not: user.id },
    emailVerified: { not: null },  // 只计已验证的
  },
});

if (verifiedChildUsersCount + 2 > parentSubscription.quantity) {
  // +2 = 当前用户 + 订阅 owner
  await updateSeats(stripeSubscriptionId, verifiedChildUsersCount + 2);
}
```

**自动扩容逻辑**：当被邀请的成员首次登录且座位不足时，**自动调用 Stripe API 增加座位数**，而不是拒绝加入。

#### B. 成员被移除时

[deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L93-L97)：成员被移除且已验证邮箱时，座位数 -1。

### 7.2 链接数量限制

[verifyCapacity.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/lib/verifyCapacity.ts)：

| 用户类型 | 链接限额 | 计算方式 |
|----------|----------|----------|
| 无 Stripe（自托管） | 30,000 / 用户 | 按创建者统计 createdById = userId |
| 试用期内（默认 14 天） | 30,000 / 用户 | 同上 |
| 单用户订阅（quantity = 1） | 30,000 / 用户 | 同上 |
| 团队订阅（quantity > 1 或有 parentSubscriptionId） | 30,000 × quantity（总量） | 统计同一订阅下所有用户创建的链接总数 |

团队订阅的统计范围：
```typescript
OR: [
  { parentSubscriptionId: subscriptionId },
  { subscriptions: { id: subscriptionId } },
]
```

### 7.3 订阅有效性检查

[verifySubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/stripe/verifySubscription.ts)：

用户有效订阅的条件（满足其一即可）：
1. `parentSubscription.active === true`
2. `subscriptions.active === true` 且 `now <= currentPeriodEnd`
3. 试用期未过（若 `REQUIRE_CC !== true`）

若本地数据失效，会调用 Stripe API 远程查询并 upsert 回本地数据库。

---

## 8. 资源归属总结

### 8.1 各资源的所有权字段

| 资源 | 所有者字段 | 级联删除行为 | 说明 |
|------|-----------|-------------|------|
| `Collection` | `ownerId` | User 删除 → 全部删除 | 集合归用户个人所有，不归订阅 |
| `Link` | `collectionId` → `Collection.ownerId` | Collection 删除 → 链接删除 | 链接的归属跟随集合 |
| `Link` | `createdById`（可选） | User 删除 → 创建者置空或删除？ | 仅记录创建者信息，不决定归属 |
| `Tag` | `ownerId` | User 删除 → 全部删除 | 标签归用户个人所有 |
| `Highlight` | `userId` | User 删除 → 全部删除 | 高亮归用户个人所有 |
| `DashboardSection` | `userId` | User 删除 → 全部删除 | 仪表板配置归用户个人 |
| `AccessToken` | `userId` | User 删除 → 全部删除 | |

### 8.2 订阅与资源的关系

**重要结论**：Subscription 不直接拥有任何业务资源（Collection、Link、Tag 等）。

- 订阅的作用仅为：
  1. 授权访问（billing 检查）
  2. 定义成员池（通过 parentSubscriptionId）
  3. 限制链接总量（capacity 检查）
  4. 限制座位数（seats 检查）

- 资源的实际所有者是**具体的 User**（通过 `ownerId` 字段），而非 Subscription

这意味着：
- 被邀请的成员创建的 Collection 归该成员自己所有，不是订阅 owner 所有
- 订阅 owner 被删除时，只会删除自己的 Collection，不会删除成员的 Collection
- 成员之间看不到对方的私有 Collection，除非显式通过 `UsersAndCollections` 授权

---

## 9. 状态迁移图

### 9.1 用户邀请生命周期

```
[未创建]
   │
   │  Owner 点击邀请 (POST /api/v1/users?invite=true)
   ▼
[已创建 / 未验证]
  - emailVerified = null
  - name = null
  - password = null
  - parentSubscriptionId = 订阅 ID
   │
   │  接收邀请邮件 (20 分钟内有效)
   │  点击链接登录
   ▼
[首次登录 / 待 Onboarding]
  - 自动扩容 seats（如需要）
  - 跳转 /member-onboarding
   │
   │  设置 name + password (PUT /api/v1/users/:id)
   ▼
[已激活成员]
  - name != null
  - password != null
  - emailVerified 状态依赖路径 (SSO/自注册会置为已验证)
   │
   ├──► Owner 移除 ──► [独立用户] (parentSubscriptionId = null, 账号保留)
   │
   └──► 自主删号 ────► [已删除] (所有个人资源级联删除)
```

### 9.2 订阅所有者生命周期

```
[订阅 Owner]
  - subscriptions.active = true
   │
   ├──► 取消订阅 ──► [订阅过期 / inactive]
   │                    │
   │                    └──► 试用期后所有操作受限
   │
   └──► 删除账号 ──► [已删除]
                         │
                         ├── Subscription 级联删除
                         ├── 所有 child users 变为独立用户
                         └── Owner 的所有 Collection/Link/Tag 级联删除
```
