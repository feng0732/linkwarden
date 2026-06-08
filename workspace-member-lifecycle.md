# Workspace / Member 生命周期状态分析

## 0. API 参数定义速查

### DELETE /api/v1/users/:id 调用链

- **路由入口**：[users/[id]/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/pages/api/v1/users/%5Bid%5D/index.ts#L79-L93)
- **控制器**：[deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L10-L15)

| 参数名 | 来源 | 含义 |
|--------|------|------|
| `userId` | `token.id`（当前登录用户 JWT） | **操作者**（谁发起的删除请求） |
| `queryId` | `Number(req.query.id)`（URL `:id`） | **目标用户**（谁的账号被操作） |
| `isServerAdmin` | `user.id === NEXT_PUBLIC_ADMIN` | 操作者是否为服务器管理员（ID=1） |
| `body` | 请求体 JSON | 含 `password`（自删验证）、`cancellation_details` |

---

## 1. 核心数据模型

### 1.1 用户与订阅的层级关系

Linkwarden 的"工作区"通过 **Subscription（订阅）模型隐式实现，没有独立的 Workspace 表。

[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/prisma/schema.prisma#L28-L75)、[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/prisma/schema.prisma#L220-L232)：

```
Subscription (owner = User A, quantity = 5)
├── User A (owner)      subscriptions.id = X     ← 一对一，userId unique
├── User B (member)     parentSubscriptionId = X ← 多对一，可空
├── User C (member)     parentSubscriptionId = X
└── ... 最多 (quantity - 1) 个 child users
```

| 字段 | onDelete 行为 | 来源 migration |
|------|--------------|----------------|
| `Subscription.userId` → `User.id` | **CASCADE**（删 User → 删其 Subscription） | [20250318123928](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/prisma/migrations/20250318123928_add_referential_actions_to_certain_fields/migration.sql#L52-L53) |
| `User.parentSubscriptionId` → `Subscription.id` | **SET NULL**（删 Subscription → child users 字段置空，账号保留） | [20241021175802](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/prisma/migrations/20241021175802_add_child_subscription_support/migration.sql#L44) |

### 1.2 集合与成员关系

[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/prisma/schema.prisma#L126-L164)：

| 模型 | 字段 | onDelete | 来源 |
|------|------|----------|------|
| `Collection.ownerId` → `User.id` | 集合所有者 | **CASCADE**（删 User → 删其所有 Collection） | [20250318131012](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/prisma/migrations/20250318131012_add_referential_action_to_field/migration.sql#L2-L4) |
| `UsersAndCollections.userId` → `User.id` | 集合成员 | **CASCADE**（删 User → 清除其所有成员关系） | [20250318123928](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/prisma/migrations/20250318123928_add_referential_actions_to_certain_fields/migration.sql#L38-L39) |
| `UsersAndCollections.collectionId` → `Collection.id` | 集合侧 | **CASCADE**（删 Collection → 清除其所有成员） | [20250318123928](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/prisma/migrations/20250318123928_add_referential_actions_to_certain_fields/migration.sql#L40-L41) |

**注意**："订阅成员身份"（同一 Subscription）与"集合成员权限"（UsersAndCollections）是**两个独立维度**：
- 订阅成员只共享链接额度和座位池
- 集合成员需单独授权，不会因为同属一个订阅就自动可见

### 1.3 Link 资源的双重归属

[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/prisma/schema.prisma#L166-L198)：

| 字段 | 含义 | onDelete | 说明 |
|------|------|----------|------|
| `Link.collectionId` → `Collection.id` | 链接所在集合 | **CASCADE** | 删 Collection → 其下所有 Link 被删 |
| `Link.createdById` → `User.id` | 谁创建了这条链接（可空） | **CASCADE** | 删 User → **该用户创建的所有 Link 被删**，无论这些 Link 在谁的 Collection 中 |

**重要**：Link 有两条独立的删除路径。如果 User A 在 User B 的 Collection 中创建了 Link：
- User B 删号 → Collection 删除 → Link 通过 `collectionId` Cascade 被删
- User A 删号 → Link 通过 `createdById` Cascade 也被删（即使 Collection 还在） |

---

## 2. deleteUserById 三大分支深度解析

[deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L39-L205)

控制流：

```
deleteUserById(userId, body, isServerAdmin, queryId)
│
├── isServerAdmin === true ?
│     └── YES → 跳过所有检查，直接执行 Transaction 硬删除 queryId
│
└── NO（普通用户）
      │
      ├── queryId === userId ?
      │     ├── YES → 【分支一：用户自删】
      │     │         ├── 校验 body.password === 目标用户密码
      │     │         └── 进入 Transaction 硬删除 queryId
      │     │
      │     └── NO → 操作者 ≠ 目标用户 → 尝试 Owner 移除成员
      │               │
      │               ├── 操作者自己有 parentSubscriptionId（是个 child user）?
      │               │     └── YES → 401 "Permission denied."（成员无权操作）
      │               │
      │               ├── 操作者无 subscriptions?
      │               │     └── YES → 401 "User has no subscription."（非 Owner）
      │               │
      │               └── 检查 queryId 是否是操作者订阅下的 child user?
      │                     ├── NO → 401 "Permission denied."
      │                     └── YES → 【分支二：Owner 软移除成员】
      │                               ├── prisma.user.update({ disconnect: parentSubscription })
      │                               ├── 若成员 emailVerified → updateSeats(quantity - 1)
      │                               └── return 200（不进入 Transaction，不删账号）
      │
      └── （isServerAdmin 进入 Transaction 时同样走下方路径）
             │
             └── 【分支三：硬删除 queryId】Transaction 内执行
                   ├── 删除 MeiliSearch 索引（ownerId = queryId 的 Collection 中的 Link）
                   ├── 删除 archives/{collectionId} 文件夹
                   ├── 删除头像
                   ├── Stripe 处理
                   └── prisma.user.delete({ where: { id: queryId } })
                         └── 触发所有 Prisma onDelete CASCADE / SET NULL
```

### 2.1 分支一：用户自行删号（queryId === userId）

[deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L40-L59)

**触发条件**：`!isServerAdmin && queryId === userId`

**前置校验**：
- 目标用户必须设置了 `password`（否则要求先找回密码）
- `body.password` 必须通过 bcrypt 校验

**进入 Transaction 硬删除**（见 2.3）

### 2.2 分支二：订阅 Owner 移除成员（软移除）

[deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L60-L104)

**触发条件**：`!isServerAdmin && queryId !== userId && 操作者是订阅 Owner && queryId 是其 child user`

**操作**：
1. `prisma.user.update({ where: { id: findChild.id }, data: { parentSubscription: { disconnect: true } }`)
   - 仅清空 child user 的 `parentSubscriptionId` 字段
   - **不删除用户账号**，也不删除其任何资源
2. 若被移除成员 `emailVerified != null`（已验证邮箱）：
   - 调用 `updateSeats(stripeSubscriptionId, quantity - 1)` 同步减少 Stripe 座位数

**移除后状态**：
- 成员账号完整保留，可自行注册独立订阅
- 其 `UsersAndCollections` 集合成员权限**不会自动清理**
- 其创建的 Collection / Link / Tag 全部归自己所有，不受影响

### 2.3 分支三：硬删除（Server Admin 删除 或 自删 通过 Transaction）

[deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L107-L205)

**触发条件**：`isServerAdmin === true` 或 分支一的自删

**执行步骤**：

```
Transaction {
  1. MeiliSearch 清理：
     where: { collection: { ownerId: queryId } }
     → 只删"queryId 拥有的 Collection"里的 Link 索引
     → 注意：queryId 在他人 Collection 中创建的 Link，由 Prisma Cascade 负责

  2. 文件系统清理：
     - 遍历所有 Collection (ownerId = queryId)
       └── 删除 archives/{id} 和 archives/preview/{id}
     - 删除 uploads/avatar/{queryId}.jpg

  3. Stripe 处理（STRIPE_SECRET_KEY 启用时）：
     a) user.subscriptions 存在 且 queryId !== userId
        → 这是 Admin 在删一个订阅 Owner 的号 → 取消 queryId 的 Stripe 订阅
     b) user.subscriptions 存在 且 queryId === userId
        → 用户自删且自己是订阅 Owner → 取消自己的 Stripe 订阅
     c) else if user.parentSubscription 存在 且 user.emailVerified
        → 用户自删且自己是 child user → updateSeats(parent, quantity - 1)

  4. prisma.user.delete({ id: queryId }) → 触发数据库级 Cascade：
     ┌─────────────────────────────────────────────────────────────────┐
     │ 直接被删（onDelete: CASCADE，通过 User 外键）：                   │
     │  ├── Account[]            (Account.userId)                      │
     │  ├── Collection[]         (Collection.ownerId)                  │
     │  ├── Tag[]                (Tag.ownerId)                         │
     │  ├── UsersAndCollections[] (UsersAndCollections.userId)         │
     │  ├── WhitelistedUser[]    (WhitelistedUser.userId)              │
     │  ├── AccessToken[]        (AccessToken.userId)                  │
     │  ├── Highlight[]          (Highlight.userId)                    │
     │  ├── DashboardSection[]   (DashboardSection.userId)             │
     │  └── Subscription         (Subscription.userId)                 │
     │                                                                 │
     │ 间接被删（二级 Cascade）：                                        │
     │  ├── Collection[] 被删 → 其下 Link[] (Link.collectionId) 被删   │
     │  ├── Collection[] 被删 → UsersAndCollections[] 被删             │
     │  ├── Collection[] 被删 → RssSubscription[] 被删                 │
     │  ├── Collection[] 被删 → DashboardSection[collectionId] 被删   │
     │  ├── Link[] 被删 → Highlight[] (Highlight.linkId) 被删         │
     │  └── Subscription 被删 → child Users.parentSubscriptionId = NULL│
     │                                                                 │
     │ 注意：Link.createdById = queryId 的 Link                        │
     │       → 即使在他人的 Collection 中，也会直接 CASCADE 被删        │
     └─────────────────────────────────────────────────────────────────┘
}
```

---

## 3. 邀请流程与激活链路（深度核准）

### 3.1 完整激活链路（含 emailVerified 写入时机）

**入口**：[InviteModal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/components/ModalContent/InviteModal.tsx)

```
步骤 1: Owner 填写邮箱，点击"发送邀请"
    │  submit()
    ▼
步骤 2: useAddUser() → POST /api/v1/users { email, invite: true }
    │  [postUser.ts]
    ├── 前置校验：STRIPE_SECRET_KEY && EMAIL_FROM/EMAIL_SERVER 必须配置
    ├── 前置校验：parentUser 已登录
    ├── 创建新 User:
    │   ├── email = 传入邮箱
    │   ├── emailVerified = null
    │   ├── name = null
    │   ├── password = null
    │   └── parentSubscription = connect(parentUser.subscriptions.id)
    └── 返回新用户
    ▼
步骤 3: signIn("invite", { email, callbackUrl: "/member-onboarding" })
    │  [[...nextauth].ts EmailProvider id="invite" maxAge=1200]
    ├── 查库：该 email 是否存在 emailVerified=null 且 parentSubscription 的用户
    │   └── 不存在 → throw "Invalid email."
    ├── 获取 parentSubscriptionEmail（用于邮件中显示邀请人）
    ├── 限流：5 分钟内同一 identifier 最多 4 个 token
    └── sendInvitationRequest() → 发邮件，链接含 token
    ▼
步骤 4: 被邀请人点击邮件链接 /callback/email?token=...&email=...
    │
    │  ╔══════════════════════════════════════════════════════════╗
    │  ║  PrismaAdapter (@auth/prisma-adapter) 标准 Email Provider  ║
    │  ║ 内部执行顺序（核心）：                                ║
    │  ║                                                    ║
    │  ║  ① adapter.useVerificationToken()                       ║
    │  ║     → 删除并返回匹配的 token（expires > now 才成功）   ║
    │  ║  ② adapter.getUserByEmail(identifier)               ║
    │  ║     → 返回 user 对象（emailVerified 仍为 null）          ║
    │  ║  ③ 调用 callbacks.signIn({ user, email })               ║
    │  ║  ④ signIn callback 返回 true 后                          ║
    │  ║     → adapter.updateUser({ emailVerified: new Date() })║
    │  ║        ↑ 此处写入 emailVerified                    ║
    │  ║  ⑤ 调用 callbacks.jwt(trigger="signIn")                   ║
    │  ╚══════════════════════════════════════════════════════╝
    │
    │
    ├── callbacks.signIn 执行时的状态（此时③）：
    │   ├── user.emailVerified === null（步骤②查询结果）
    │   ├── email.verificationRequest === undefined（点击链接不是发邮件）
    │   └── 条件满足：检查 parentSubscriptionId → 扩容 seat（见 3.3）
    │
    ├── PrismaAdapter 在 signIn callback 返回 true 后：
    │   └── 写入 emailVerified = new Date() ← ✅ 此处首次写入
    │
    ├── callbacks.jwt(trigger="signIn"）：
    │   ├── token.id = user.id
    │   └── 若 user.username 为空 → 自动生成 username
    │   └── 不处理 emailVerified（仅 SSO signUp 时处理 emailVerified）
    │
    └── 签发 JWT session，302 到 /member-onboarding
    ▼
步骤 5: [member-onboarding.tsx] 用户填写 name + password
    │  PUT /api/v1/users/:id
    │  [updateUserById.ts]
    ├── 此时数据库中 emailVerified 已经有值了（步骤④已写入）
    ├── isInvited = (name===null && parentSubscriptionId && !password)
    ├── 强制校验 password 必填
    ├── 更新 name、password（bcrypt hash）
    └── 不更新 emailVerified（已经有值了，无需再写）
    ▼
步骤 6: 跳转 /dashboard，用户已完全激活
```

### 3.2 emailVerified 写入时机（核准总结）

| 场景 | 写入位置 | 触发时机 |
|------|----------|----------|
| SSO（Google/GitHub 等） | [[...nextauth].ts jwt callback](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1427-L1453) | trigger="signUp" 且 accounts.length > 0） |
| 自注册 + Email Provider（id="email"） | **PrismaAdapter 内部**（@auth/prisma-adapter） | 点击邮件链接，signIn callback 返回 true 后 |
| **被邀请 + 邀请邮件（id="invite"）** | **PrismaAdapter 内部**（同上） | 点击邀请邮件链接，signIn callback 返回 true 后 |
| Admin 手动创建用户 | [postUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/postUser.ts#L89) | isAdmin 时直接写入 |

**关键核准结论**：
- **PrismaAdapter 的 Email Provider（无论是 id="email" 或 id="invite"）**在**signIn callback 返回 true 后，在 adapter.updateUser({ emailVerified: new Date() })写入。
- signIn callback 执行时读取到的 user.emailVerified 仍为 null——这是正确的，因为扩容判断正是在更新前的用户对象。
- member-onboarding 页面提交时，数据库中 emailVerified **已经有值**，只需补写。

### 3.3 Seat 自动扩容与激活对齐（核准）

signIn callback 中的扩容逻辑 [[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1326-L1364)：

```typescript
if (!(user as User).emailVerified && !email?.verificationRequest) {
  // 有 parentSubscriptionId → 扩容逻辑
}
```

**触发条件拆解**：

| 条件 | 含义 | 对齐结论 |
|------|------|----------|
| `!user.emailVerified | 用户 emailVerified 为 null | **首次**点击邀请链接才满足 |
| `!email?.verificationRequest | 点击链接（不是"发送邮件"请求 | 排除 signIn("invite") 时 |

**扩容触发时机**：**首次**点击邀请邮件链接时（此时 emailVerified 为 null，PrismaAdapter 写入前）。

扩容计算公式：
```
已验证 child users 数 + 2（当前用户 + 订阅 Owner） > subscription.quantity
→ 若超过 → updateSeats(stripeId, 已验证数 + 2)
```

**与激活与 seat 的对应关系**：

| 用户状态 | emailVerified | 占 seat? | 说明 |
|----------|--------------|----------|------|
| 已创建，未点击邀请链接 | null | 否 | verifiedChildUsersCount 不统计 |
| 已点击邀请链接（完成 onboarding） | 已写入 | **是** | PrismaAdapter 已写入 emailVerified |
| 已点击但未完成 onboarding | 已写入 | **是** | emailVerified 已写入 |
| 已完成 onboarding（name+password） | 已写入 | **是** | 正常激活状态 |

**结论对齐**：Seat 计数完全以 `emailVerified != 为准，与 name/password 无关。用户一一点击邮件链接（emailVerified 写入成功 → 立即占一个 seat。

### 3.4 邀请过期机制

- **令牌有效期**：[[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L141) `maxAge: 1200` → **20 分钟 |
- **过期校验**：`VerificationToken.expires > new Date()`，不满足返回 `"Invalid token."`
- **限流**：5 分钟内同一邮箱最多 4 次发送请求 |

---

## 4. 加入 / 激活状态判定（深度核准）

### 4.1 "待激活的被邀请用户"判定

[updateUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserById.ts#L170-L177)：

```typescript
const isInvited =
  user?.name === null        // 未设置显示名
  && user.parentSubscriptionId   // 被邀请加入
  && !user.password;             // 未设置密码
```

注意：isInvited 与 emailVerified 无关——即使用户点击了链接（emailVerified 已写入），只要 name==null && password==null && parentSubscriptionId 存在 → isInvited 仍为 true。

### 4.2 四种激活状态变化详解

#### 状态 A：已创建用户（未点击邀请链接）

| 字段 | 值 |
|------|----|
| emailVerified | null |
| name | null |
| password | null |
| parentSubscriptionId | 订阅 ID |
| isInvited | true |
| 占 seat? | 否 |

用户存在于 DB 中，但不能登录（无可用登录路径： |
- Credentials Provider：password=null → 抛错 |
- Email Provider（id=invite）→ 需要重新发送邮件 → token 20 分钟过期 |
- 过期 token → "Invalid token." |

#### 状态 B：已点击邀请链接（未完成 onboarding）

| 字段 | 值 |
|------|----|
| emailVerified | new Date()（PrismaAdapter 写入） | 已写入 |
| name | null |
| password | null |
| parentSubscriptionId | 订阅 ID |
| isInvited | true |
| 占 seat? | **是** |

可用登录路径：
- Email Provider（再次点击链接）→ signIn callback 中 emailVerified 已不为 null → **不再扩容 |
- Credentials Provider：password=null → "Invalid credentials." → |

#### 状态 C：完成 onboarding（name + password 已设置）

| 字段 | 值 |
|------|----|
| emailVerified | 已写入 |
| name | 已设置 |
| password | bcrypt hash |
| parentSubscriptionId | 订阅 ID |
| isInvited | false |
| 占 seat? | **是** |

可用登录路径：
- Credentials Provider：emailVerified != null → 通过；password bcrypt 校验 → 通过 |
- Email Provider：正常登录 |

### 4.3 各 Provider 对 emailVerified 的要求

| Provider | emailVerified 要求 | 代码位置 |
|----------|----------------|----------|
| Credentials（密码） | `emailEnabled && !user.emailVerified` → 抛 "Email not verified." | [[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L117-L119) |
| Email Provider（id=email / id=invite） | PrismaAdapter 验证 token 后自动写入 emailVerified | @auth/prisma-adapter） |
| SSO Provider | jwt callback trigger="signUp" 时显式写入 | [[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1427-L1453) |
| API verifyUser 中间件 | `NEXT_PUBLIC_EMAIL_PROVIDER && !user.emailVerified` → 401 | [verifyUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/verifyUser.ts#L49-L58) |

---

## 5. Owner 离开工作区的完整影响

### 5.1 订阅 Owner（User with subscriptions）硬删除时

```
User (Owner) 执行 prisma.user.delete
 │
 ├── Subscription.userId (onDelete: CASCADE)
 │    └── Subscription 记录被删除
 │         └── User.parentSubscriptionId (ON DELETE SET NULL)
 │              └── 所有 child users 的 parentSubscriptionId = NULL
 │                   └── 这些成员账号保留，变为独立用户
 │
 ├── Collection.ownerId (onDelete: CASCADE)
 │    └── Owner 拥有的所有 Collection 被删除
 │         ├── Link.collectionId (CASCADE) → Collection 内所有 Link 被删
 │         ├── UsersAndCollections.collectionId (CASCADE) → 成员关系清除
 │         ├── RssSubscription.collectionId (CASCADE)
 │         └── DashboardSection.collectionId (CASCADE)
 │
 ├── Tag.ownerId (CASCADE) → Owner 所有 Tag 被删
 │
 ├── Link.createdById (CASCADE)
 │    └── Owner 在任何地方（包括他人 Collection）创建的 Link 被删
 │
 ├── UsersAndCollections.userId (CASCADE)
 │    └── Owner 作为成员加入他人 Collection 的关系清除
 │
 ├── Highlight.userId (CASCADE) → Owner 所有高亮被删
 ├── DashboardSection.userId (CASCADE)
 ├── AccessToken (CASCADE) → Owner 所有 API Token 失效
 └── Account[] (CASCADE) → Owner 的 SSO 登录凭证清除
```

**关键结论**：
- 没有"所有权转移"机制——Owner 删号时其所有 Collection/Link/Tag 全部销毁
- Child users 账号保留，但订阅断开，需各自重新订阅
- Child users 自己创建的 Collection 和资源归各自所有，不受 Owner 删除影响

### 5.2 订阅 Owner 软移除 child user 时

- 仅 `parentSubscriptionId = NULL`，**不触动任何业务数据**
- child user 的 UsersAndCollections 集合成员权限仍然有效
- child user 自有的 Collection、Link、Tag 不受任何影响

---

## 6. 权限体系

### 6.1 订阅层级（Workspace 级）

无显式角色字段，按关系推断：

| 身份 | 判断条件 | 能力 |
|------|----------|------|
| **订阅 Owner** | `User.subscriptions != null` | 邀请成员、移除成员、查看订阅成员列表、管理 billing |
| **订阅成员** | `User.parentSubscriptionId != null` | 共享 seat 和链接额度 |
| **独立用户** | 两者皆无 | 仅用自己的资源 |

相关代码：
- 成员列表查询：[getUsers.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/getUsers.ts#L26-L70) `OR: [{parentSubscriptionId}, {subscriptions.id}]`
- 订阅有效性：[verifySubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/stripe/verifySubscription.ts#L32-L34) `parentSubscription.active` 或 `subscriptions.active`

### 6.2 集合层级（Collection 级）

[UsersAndCollections](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/prisma/schema.prisma#L151-L164) 三个布尔字段：

| 字段 | 含义 |
|------|------|
| `canCreate` | 在集合中创建子集合和链接 |
| `canUpdate` | 修改集合信息和链接 |
| `canDelete` | 删除集合和链接 |

`Collection.ownerId` 对应用户**隐式拥有全部权限**，无需存在 UsersAndCollections 记录。

权限判定：
- [getPermission.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/getPermission.ts#L28-L37)：`OR: [{ownerId}, {members.some}]`
- [updateCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts#L34-L35)：仅 owner 可修改
- [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L19-L52)：
  - 成员调 DELETE → 只删自己的 UsersAndCollections 记录（离开集合）
  - Owner 调 DELETE → 真删集合及所有内容

### 6.3 子集合权限继承

创建子集合时：[postCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/collections/postCollection.ts#L36-L82)
- 根集合的 `ownerId` 自动成为所有后代集合的 `ownerId`
- 路径上所有集合的 members（权限 OR 合并）被复制到新子集合

更新集合时 `propagateToSubcollections=true`：[updateCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/collections/collectionId/updateCollectionById.ts#L68-L121)
- 递归所有后代集合
- 清掉后代现有 members，重新写入当前集合的 members（排除后代自己的 owner）

---

## 7. 订阅限制（Seats & Capacity）

### 7.1 座位数（Seats）

`Subscription.quantity` = 座位总数（含 Owner）

| 时机 | 行为 | 代码 |
|------|------|------|
| 邀请成员**首次**接受邀请（点击链接 signIn 时 emailVerified=null） | 已验证 child users + 2 > quantity → **自动扩容** updateSeats(stripeId, 新数量） | [[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1326-L1364) |
| Owner 软移除成员 | 若成员 emailVerified → updateSeats(-1) | [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L93-L97) |
| Child user 自删 | 若成员 emailVerified → updateSeats(-1) | [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L183-L192) |

**自动扩容**：座位不足时系统直接调 Stripe API 加座，不拒绝用户加入。

### 7.2 链接数量（Capacity）

[verifyCapacity.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/packages/lib/verifyCapacity.ts)：

| 用户场景 | 限额 | 统计方式 |
|----------|------|----------|
| 无 Stripe（自托管） | 30,000 / 用户 | `Link.createdById = userId` |
| 试用期内（默认 14 天，REQUIRE_CC=false） | 30,000 / 用户 | 同上 |
| 单用户订阅（quantity=1 且无 parentSubscriptionId） | 30,000 / 用户 | 同上 |
| 团队订阅（quantity>1 或有 parentSubscriptionId） | 30,000 × quantity（共享池） | 同一 subscription 下所有用户的 createdById Links 汇总 |

团队池统计范围：`OR: [{ parentSubscriptionId: subId }, { subscriptions: { id: subId } }]`

### 7.3 订阅有效性

[verifySubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/91-linkwarden/apps/web/lib/api/stripe/verifySubscription.ts)，满足任一即可：
1. `parentSubscription.active === true`
2. `subscriptions.active === true && now <= currentPeriodEnd`
3. 试用期内且 `REQUIRE_CC !== true`

本地数据失效时会调 Stripe API 同步并 upsert。

---

## 8. 资源归属全景

| 资源 | 所有者字段 | onDelete 行为 | 归属说明 |
|------|-----------|--------------|----------|
| Collection | `ownerId` | User 删除 → CASCADE 全删 | 归具体 User，不归 Subscription |
| Link | `collectionId` + `createdById` | 任一侧删除均触发 CASCADE | 归属跟随 Collection；但创建者删号也会销毁 |
| Tag | `ownerId` | CASCADE | 归创建用户 |
| Highlight | `userId` | CASCADE | 归创建用户 |
| DashboardSection | `userId` | CASCADE | 归用户个人 |
| AccessToken | `userId` | CASCADE | 归用户个人 |
| Subscription | `userId` | CASCADE | 归订阅 Owner 用户 |
| Subscription.childUsers | 反向关联 | Subscription 删 → SET NULL | child users 账号保留 |

**核心设计原则**：Subscription 仅用于 billing 授权、成员池和容量限制，**不直接拥有任何业务资源**。所有 Collection/Link/Tag 的 ownerId 都指向具体 User。

---

## 9. 统一状态迁移图

### 9.1 用户（被邀请成员）生命周期（核准版）

```
                          ┌───────────────────────────┐
                          │        [未创建]           │
                          └─────────────┬─────────────┘
                                        │ Owner POST /api/v1/users
                                        │   invite=true
                                        ▼
                          ┌───────────────────────────┐
                          │  [已创建/未点击链接]      │
                          │  emailVerified = null   │  状态 A
                          │  name = null          │  不占 seat
                          │  password = null      │
                          │  parentSubscriptionId=X│
                          └─────────────┬─────────────┘
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              │                         │                         │
              │ 20min 内点击邮件         │ 令牌过期/未点击         │ Owner 提前软移除
              │  PrismaAdapter 写入       │                         │ disconnect
              │  emailVerified=new Date() │                         │
              │  自动扩容 seats(如需）│                         │
              ▼                         ▼                         ▼
    ┌────────────────────┐   ┌────────────────────┐    ┌────────────────────┐
    │ [已点击/未完成    │   │ [僵尸账号]          │    │ [独立用户]          │
    │  onboarding]     │   │ 仍在 DB 中         │    │ parentSubscriptionId│
    │ emailVerified 已 │   │ emailVerified=null │    │   = NULL            │
    │ 写入             │   │ name=null         │    │ 不占 seat           │
    │ name=null        │   │ password=null     │    │                     │
    │ password=null     │   │ 无法通过 Credentials  │    │ 可自行注册订阅       │
    │ isInvited=true   │   │ 登录              │    └────────────────────┘
    │ 占 1 seat        │   │                     │
    └─────────┬──────────┘   └────────────────────┘
              │
              │ 填写 name + password
              │ PUT /api/v1/users/:id
              │  (updateUserById
              ▼
    ┌────────────────────┐
    │ [完全激活]         │ 状态 C
    │ emailVerified 已写入 │ 占 1 seat
    │ name != null       │ 正常使用所有 Provider
    │ password != null   │ 所有 API 调用通过
    │ parentSubscription│
    │   .id = X        │
    └─────────┬──────────┘
              │
     ┌────────┴────────┐
     │                 │
     │ 自删             │ Owner 软移除
     │ prisma.user.delete │ disconnect（disconnect: parentSubscription
     ▼                 │
┌────────────┐          │
│  [已删除]   │          ▼
│ CASCADE 删除│  ┌────────────────────┐
│ 所有关联资源 │  │ [独立用户]          │
└────────────┘  │ 账号保留，资源保留 │
              │ 从订阅断开关联      │
              │ 从订阅断开      │
              └────────────────────┘

    ┌──────────────────────────────────────────────────────────┐
    │  边界状态补充：                                        │
    │  ● 状态 A → 过期 token → 仍停留状态 A（不占 seat）  │
    │  ● 状态 B → 再次点链接 → 停留状态 B（emailVerified   │
    │    已不为 null → 不再触发扩容                    │
    │  ● Owner 硬删除自己 → 触发 SET NULL 所有 child users│
    └──────────────────────────────────────────────────────────┘
```

### 9.2 订阅 Owner 生命周期

```
                 ┌───────────────────────────┐
                 │ [订阅 Owner]              │
                 │ subscriptions.active=true │
                 └─────────────┬─────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
            │ Stripe 取消       │ 自删 DELETE       │ Admin 删除
            │                  │                  │
            ▼                  ▼                  ▼
   ┌─────────────────┐  ┌────────────────┐  ┌────────────────┐
   │ [订阅过期/inactive]│  │   [已删除]      │  │   [已删除]      │
   │ active=false     │  │ CASCADE 删除：  │  │ CASCADE 删除：  │
   │ 或 now>periodEnd │  │  Subscription   │  │  Subscription   │
   │                  │  │  自己所有资源   │  │  自己所有资源   │
   └────────┬─────────┘  │  child users    │  │  child users    │
            │            │  → SET NULL     │  │  → SET NULL     │
            │            └────────────────┘  └────────────────┘
            │ 试用期后所有
            │ 写操作受限
            ▼
   ┌─────────────────┐
   │   [功能受限]     │
   └─────────────────┘
```

### 9.3 邀请令牌生命周期

```
┌───────────────┐
│  [令牌生成]    │  signIn("invite") 写 VerificationToken
│  expires=+20m │
└───────┬───────┘
        │
 ┌──────┴──────┐
 │             │
 │ 20min 内使用 │  超过 20min
 │             │
 ▼             ▼
┌─────────┐  ┌──────────┐
│  [有效]  │  │  [过期]   │  expires <= now → "Invalid token"
│ 建立会话 │  │          │
└─────────┘  └──────────┘

额外限制：同一邮箱 5 分钟内最多 4 个令牌（防滥用）
```

### 9.4 Seat 计数与激活状态对齐总结

```
emailVerified 写入（null）
        │
        │ PrismaAdapter 在用户点击邀请邮件链接、signIn callback 返回 true 后写入
        ▼
┌─────────────────────────────────────────────────────────────┐
│  emailVerified != null  ⇔ 占 1 个 seat
└─────────────────────────────────────────────────────────────┘
        │
        ├── 首次点击 → verifiedChildUsersCount + 2 > quantity → 自动扩容
        │
        ├── name / password 是否设置不影响 seat 计数
        │
        └── 移除/自删时 emailVerified != null → updateSeats(-1)
```
