# Subscription/Plan 限制机制梳理

## 1. 概述

Linkwarden 的订阅限制体系基于 **Stripe 订阅系统**，通过四层机制协同生效：
1. **数据模型层**：`Subscription` / `User` 表存储订阅状态和席位数量
2. **API 全局拦截层**：`verifyUser()` 对几乎所有 v1 API 请求进行订阅校验
3. **业务配额检查层**：`hasPassedLimit()` / `verifyLinkLimit()` 在具体操作前检查链接数、成员数等
4. **前端路由守卫层**：`AuthRedirect` / `useUser()` 控制页面跳转和功能可见性

核心限制维度包括：**全局订阅有效性**、**链接数量上限**、**团队成员（Seat）数量**、**RSS 订阅数量**、**文件上传大小**。

---

## 2. 数据模型

### 2.1 Subscription 模型

定义于 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/prisma/schema.prisma#L220-L232)

```prisma
model Subscription {
  id                   Int      @id @default(autoincrement())
  active               Boolean
  stripeSubscriptionId String   @unique
  currentPeriodStart   DateTime
  currentPeriodEnd     DateTime
  quantity             Int      @default(1)
  user                 User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  userId               Int      @unique
  childUsers           User[]   @relation("ChildUsers")
  createdAt            DateTime @default(now())
  updatedAt            DateTime @default(now()) @updatedAt
}
```

**关键字段说明：**

| 字段 | 含义 |
|------|------|
| `active` | 订阅是否激活（由 Stripe Webhook 同步，deleted 事件会设为 false） |
| `quantity` | 购买的席位数量，决定团队成员上限（默认 1） |
| `currentPeriodStart` / `currentPeriodEnd` | 当前计费周期起止时间，用于判断订阅是否过期 |
| `childUsers` | 通过 `parentSubscriptionId` 关联的子用户（团队成员） |

### 2.2 User 模型与订阅关联

定义于 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/prisma/schema.prisma#L28-L75)

```prisma
model User {
  parentSubscription      Subscription?         @relation("ChildUsers", fields: [parentSubscriptionId], references: [id])
  parentSubscriptionId    Int?
  subscriptions           Subscription?
  // ...
}
```

**用户与订阅的两种关联方式：**

| 关联方式 | 字段 | 说明 |
|----------|------|------|
| 主订阅用户 | `User.subscriptions` | 用户自己购买的订阅，1:1 关系 |
| 子用户（团队成员） | `User.parentSubscriptionId` | 指向主用户的 Subscription.id，1:N 关系 |

---

## 3. 订阅计划读取流程

### 3.1 前端订阅状态获取链路

```
前端 useUser() hook
    ↓ 调用 GET /api/v1/users/{userId}
    ↓ getUserById() 读取数据库
    ↓ 返回 subscription { active, quantity } + parentSubscription { active, user.email }
    ↓ 各组件根据这些字段控制功能开关和跳转
```

**getUserById 返回数据结构**，见 [getUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/users/userId/getUserById.ts#L39-L54)：

```typescript
{
  // ... 用户基础信息
  subscription: {
    active: subscriptions?.active ?? false,      // 自身订阅是否激活
    quantity: subscriptions?.quantity ?? 0,       // 购买的席位数量
  },
  parentSubscription: {
    active: parentSubscription?.active,           // 所属父订阅是否激活
    user: { email: parentSubscription?.user.email } // 订阅拥有者邮箱
  },
}
```

### 3.2 API 层订阅验证（全局拦截）

**核心函数 `verifyUser()`**，见 [verifyUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/verifyUser.ts#L14-L73)

这是**最关键的全局拦截点**——几乎所有 `/api/v1/*` 端点都先调用 `verifyUser()` 进行鉴权：

```
verifyUser({ req, res }) 流程：
1. 校验 Token → 获取 userId
2. 查询 user（include subscriptions 和 parentSubscription）
3. 检查 username、emailVerified
4. 【启用 Stripe 时】调用 verifySubscription(user)
   ├─ 有效 → 返回 user 对象，API 继续执行
   └─ 无效 → 返回 401：
      "You are not a subscriber, feel free to reach out to us at
       support@linkwarden.app if you think this is an issue."
```

**使用 verifyUser 的 API 端点（30+ 个）包括：**
- `/api/v1/users/*` GET/PUT/DELETE（不含 POST 注册和 GET /me）
- `/api/v1/links/*`（所有 POST/PUT/DELETE/GET）
- `/api/v1/collections/*`（不含 public 路径）
- `/api/v1/tags/*`、`/api/v1/highlights/*`、`/api/v1/rss/*`
- `/api/v1/archives/index.ts`（上传新归档）、`/api/v1/archives/[linkId].ts` POST（更新归档）
- `/api/v1/dashboard/*`、`/api/v1/search/*`、`/api/v1/migration/*`
- `/api/v1/tokens/*`、`/api/v1/worker/*` 等

### 3.3 四种鉴权函数的订阅检查对比

系统中存在四种独立的鉴权函数，订阅检查行为各不相同：

| 鉴权函数 | 调用方 | 是否检查订阅 | 检查方式 | 检查完整性 |
|----------|--------|-------------|----------|-----------|
| **verifyUser** | 大部分 v1 API | ✅ 是 | verifySubscription() 完整调用 | ⭐⭐⭐⭐⭐ 完整（含试用期、父订阅、过期时间、Stripe 同步） |
| **verifyByCredentials** | POST /api/v1/session | ✅ 是 | verifySubscription() 完整调用 | ⭐⭐⭐⭐⭐ 同上 |
| **isAuthenticatedRequest** | POST /api/v1/users（带 invite） | ⚠️ 部分 | 仅判断 `!user.subscriptions` | ⭐ 不完整（不检查 parentSubscription、active、periodEnd、试用期） |
| **verifyToken** | GET archives、GET preserved/token、GET /users/me、GET avatar | ❌ 否 | 无 | 不检查 |
| **getToken（next-auth）** | GET/POST /api/v1/payment | ❌ 否 | 无 | 不检查 |

---

#### verifyByCredentials：凭证校验中的完整订阅检查

位置 [verifyByCredentials.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/verifyByCredentials.ts#L15-L64)

```typescript
// 密码匹配成功后，启用 Stripe 时执行完整订阅检查
if (STRIPE_SECRET_KEY) {
  const subscribedUser = await verifySubscription(user);
  if (!subscribedUser) {
    return null;  // 订阅失效 → 凭证校验失败
  }
}
```

**关键结论：`POST /api/v1/session` （创建 API Token）虽然不调用 `verifyUser`，但通过 `verifyByCredentials` 执行了**完整的** `verifySubscription` 检查。非订阅用户无法通过用户名密码创建 API Session。**

---

#### isAuthenticatedRequest：有缺陷的简化订阅检查

位置 [isAuthenticatedRequest.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/isAuthenticatedRequest.ts#L35-L48)

```typescript
const findUser = await prisma.user.findFirst({
  where: { id: userId },
  include: { subscriptions: true },  // 未 include parentSubscription！
});

if (STRIPE_SECRET_KEY && findUser && !findUser?.subscriptions) {
  return null;  // 仅有 subscriptions 记录的判断
}
```

**检查缺陷：**
1. ❌ 未检查 `parentSubscription`（子用户身份会被错误拦截）
2. ❌ 未检查 `subscription.active` 状态
3. ❌ 未检查 `currentPeriodEnd` 过期时间
4. ❌ 未处理试用期逻辑
5. ❌ 未尝试从 Stripe 同步最新状态

**调用方：** `POST /api/v1/users` 仅当请求包含 `invite` 参数（管理员邀请成员）时使用。

---

#### verifyToken：纯会话校验，完全不检查订阅

**verifyToken** 定义于 [verifyToken.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/verifyToken.ts#L9-L36)，仅做 token 层面的会话校验：
- JWT token 是否存在
- token 是否过期（`token.exp < Date.now() / 1000`）
- token 是否已被撤销（`accessToken.revoked = true`）

**完全不涉及订阅状态**。使用该函数的路径在订阅降级后，只要会话有效即可继续访问。

---

### 3.4 用户接口与支付接口的订阅校验差异（深度校准）

#### `/api/v1/users/[id]`：同一接口的混合校验模式

定义于 [users/[id]/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/users/%5Bid%5D/index.ts#L11-L94)，**该接口按 HTTP 方法差异采用完全不同的校验策略**：

```
verifyToken (JWT 有效性)
    │
    ├── GET 方法 ──→ 仅校验 userId === queryId || isServerAdmin
    │                  完全不调用 verifySubscription ← 降级后可访问
    │
    └── PUT/DELETE ──→ 启用 STRIPE_SECRET_KEY 时 ──→ 完整 verifySubscription()
                           检查：subscriptions + parentSubscription + active + 过期 + 试用期
                           ↓ 降级后返回 401 非订阅者
```

**逐方法解析：**

| 方法 | 代码位置 | verifyToken | verifySubscription | 权限控制 | 降级后结果 |
|------|---------|-------------|-------------------|---------|-----------|
| GET | 第 35-40 行 | ✅ | ❌ 不调用 | userId === queryId \|\| isServerAdmin | ✅ 200（只能读自己） |
| PUT | 第 43-78 行 | ✅ | ✅ 完整调用（第 54-65 行） | userId === queryId \|\| isServerAdmin | ❌ 401 非订阅者 |
| DELETE | 第 43-93 行 | ✅ | ✅ 完整调用（第 54-65 行） | 删自己 \|\| 管理员删任意 | ❌ 401 非订阅者 |

**注意**：该接口没有使用 `verifyUser()`，而是**内联实现**了 `verifyToken` + 条件性 `verifySubscription` 的组合，属于"第一类（完整检查）"和"第三类（完全不检查）"的混合体。

---

#### `/api/v1/users/me`：仅 verifyToken，完全不检查订阅

定义于 [users/me.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/users/me.ts#L5-L18)：

```typescript
// 整个函数仅做两件事：
const token = await verifyToken({ req });  // 1. 检查 JWT 有效性
const users = await getUserById(userId);   // 2. 返回用户数据
// ❌ 无任何 verifySubscription 调用
```

该接口返回的用户数据中包含 `subscription` 字段，供前端判断是否需要跳转 `/subscribe`。但**后端本身不基于该字段做拦截**。

---

#### `/api/v1/payment`：仅 getToken，完全不检查订阅

定义于 [payment/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/payment/index.ts#L7-L37)：

```typescript
// 仅支持 GET 方法（无 POST）
const token = await getToken({ req });  // next-auth/jwt 原生方法，仅验证 JWT 签名
if (!token?.id) return 404;             // 仅检查 token 是否存在且有 id
// ❌ 无任何 verifySubscription 调用
// ❌ 甚至不使用封装过的 verifyToken
```

**设计合理性**：支付接口必须允许**已过期的非订阅用户**访问以完成续费购买。若此处检查订阅，将导致降级用户无法支付，形成死锁。

---

#### 用户接口家族的完整校验矩阵

| 路径 | 方法 | 鉴权函数 | 订阅检查 | 降级后可访问 |
|------|------|---------|---------|-------------|
| `/api/v1/users`（列表） | GET | verifyUser | ✅ 完整 | ❌ 401 |
| `/api/v1/users`（注册，无 invite） | POST | 无 | ❌ | ✅ 200 |
| `/api/v1/users`（带 invite） | POST | isAuthenticatedRequest | ⚠️ 部分（有缺陷） | ⚠️ 子用户可能被误拦 |
| **`/api/v1/users/[id]`** | **GET** | **verifyToken** | **❌ 不检查** | **✅ 200（只能读自己）** |
| **`/api/v1/users/[id]`** | **PUT** | **verifyToken + 内联 verifySubscription** | **✅ 完整** | **❌ 401** |
| **`/api/v1/users/[id]`** | **DELETE** | **verifyToken + 内联 verifySubscription** | **✅ 完整** | **❌ 401** |
| `/api/v1/users/[id]/preference` | PUT | verifyUser | ✅ 完整 | ❌ 401 |
| **`/api/v1/users/me`** | **GET** | **verifyToken** | **❌ 不检查** | **✅ 200** |

---

### 3.5 三类 API 路径的订阅检查分类

根据是否执行订阅检查以及检查的完整性，将所有 API 路径分为三类：

---

#### 第一类：完整订阅检查（经过 verifyUser、verifyByCredentials 或内联 verifySubscription）

| API 路径 | HTTP 方法 | 检查方式 | 降级后能否访问 |
|----------|-----------|----------|----------------|
| **`/api/v1/users/[id]`** | **PUT/DELETE** | **verifyToken + 内联 verifySubscription（完整）** | ❌ 401 非订阅者 |
| `/api/v1/users/[id]/preference` | PUT | verifyUser | ❌ 401 非订阅者 |
| `/api/v1/users`（GET 列表） | GET | verifyUser | ❌ 401 非订阅者 |
| `/api/v1/links/*` | 所有 | verifyUser | ❌ 401 非订阅者 |
| `/api/v1/collections/*`（除 public） | 多种 | verifyUser | ❌ 401 非订阅者 |
| `/api/v1/tags/*`、`/api/v1/highlights/*` | 多种 | verifyUser | ❌ 401 非订阅者 |
| `/api/v1/rss/*` | GET/POST/DELETE | verifyUser | ❌ 401 非订阅者 |
| `/api/v1/archives/*`（POST 上传/更新） | POST | verifyUser | ❌ 401 非订阅者 |
| `/api/v1/dashboard/*`、`/api/v1/search/*` | GET | verifyUser | ❌ 401 非订阅者 |
| `/api/v1/migration/*` | POST | verifyUser | ❌ 401 非订阅者 |
| `/api/v1/tokens/*` | GET/POST/DELETE | verifyUser | ❌ 401 非订阅者 |
| `/api/v1/worker/*` | GET | verifyUser | ❌ 401 非订阅者（且需管理员） |
| **`/api/v1/session`** | **POST** | **verifyByCredentials → verifySubscription（完整）** | ❌ **凭证校验失败（返回 "Invalid credentials"）** |

---

#### 第二类：部分订阅检查（仅 isAuthenticatedRequest）

| API 路径 | HTTP 方法 | 检查方式 | 降级后行为 |
|----------|-----------|----------|------------|
| `/api/v1/users`（带 `invite` 参数） | POST | isAuthenticatedRequest | ⚠️ **有缺陷**：子用户可能被误拦截，仅检查 `!user.subscriptions`，不检查 active、过期、试用期 |

---

#### 第三类：完全不检查订阅

| API 路径 | HTTP 方法 | 鉴权方式 | 降级后能否访问 | 说明 |
|----------|-----------|----------|----------------|------|
| **`GET /api/v1/users/[id]`** | GET | verifyToken（仅 JWT 有效性） | ✅ 可以 | 仅能读取自己的信息或管理员读任意用户，**不检查订阅**。注意：PUT/DELETE 同一接口会检查订阅 |
| **`GET /api/v1/users/me`** | GET | verifyToken（仅 JWT 有效性） | ✅ 可以 | 获取当前用户信息（含 subscription 状态供前端判断），**完全不检查订阅** |
| **`GET /api/v1/archives/[linkId]`** | GET | verifyToken + resolveAccessibleArchive | ✅ 可以 | 归档读取，仅受集合权限限制 |
| **`GET /api/v1/preserved/token`** | GET | verifyToken + resolveAccessibleArchive | ✅ 可以 | 获取归档临时访问 URL |
| **`GET /api/v1/preserved/view`** | GET | 独立 JWT token（短期签名） | ✅ 可以 | 通过签名 token 直接读归档文件 |
| **`GET /api/v1/avatar/[id]`** | GET | verifyToken（可选） | ✅ 可以 | 读取任意用户头像 |
| **`GET /api/v1/payment`** | GET | getToken（仅 JWT 有效性） | ✅ 可以 | **仅支持 GET**，Stripe 支付结账（必须允许非订阅者访问以完成购买），完全不检查订阅 |
| `/api/v1/public/collections/[id]` | GET | 无（仅检查 isPublic） | ✅ 可以 | 读取公开集合元数据 |
| `/api/v1/public/collections/links` | GET | 无（仅检查 isPublic） | ✅ 可以 | 读取公开集合下的链接列表 |
| `/api/v1/public/collections/tags` | GET | 无（仅检查 isPublic） | ✅ 可以 | 读取公开集合下的标签 |
| `/api/v1/public/links/[id]` | GET | 无（仅检查 isPublic） | ✅ 可以 | 读取公开链接详情 |
| `/api/v1/public/users/[id]` | GET | 无（仅读取公开资料） | ✅ 可以 | 读取用户公开资料 |
| `/api/v1/config` | GET | 无 | ✅ 可以 | 获取实例配置（AI、文件大小等） |
| `/api/v1/logins` | GET | 无 | ✅ 可以 | 获取登录方式列表 |
| `/api/v1/getFavicon` | GET | 无 | ✅ 可以 | 代理获取网站 favicon |
| `/api/v1/users`（注册，无 invite） | POST | 无 | ✅ 可以 | 新用户注册 |
| `/api/v1/auth/forgot-password` | POST | 无（邮箱校验） | ✅ 可以 | 发送密码重置邮件 |
| `/api/v1/auth/verify-email` | POST | 无（token 校验） | ✅ 可以 | 邮箱验证 |
| `/api/v1/auth/reset-password` | POST | 无（token 校验） | ✅ 可以 | 密码重置 |
| `/api/v1/auth/[...nextauth]`（Web 登录） | POST | NextAuth Credentials authorize | ✅ 可以登录 | 仅检查密码和邮箱验证，不检查订阅（登录后 API 调用会被 verifyUser 拦截） |
| `/api/v1/webhook` | POST | Stripe 签名校验 | ✅ 可以 | Stripe Webhook（必须始终可用以接收订阅变更事件） |

**⚠️ 关于 NextAuth Web 登录的重要说明：**

NextAuth Credentials Provider 的 `authorize` 函数（[auth/[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L90-L130)）**仅校验密码和邮箱验证状态，不检查订阅**。这意味着订阅已降级的用户仍可成功登录 Web 界面，但登录后：
1. 前端 `AuthRedirect` 会立即将其重定向至 `/subscribe`
2. 所有调用 `verifyUser` 的 API 请求会返回 401
3. 但用户仍可通过第三类 API 访问归档文件等不受订阅保护的资源

### 3.6 归档读取权限判定（resolveAccessibleArchive）

归档读取不经过订阅校验，其权限完全由 **集合访问权限** 决定，见 [resolveAccessibleArchive.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/archives/resolveAccessibleArchive.ts#L30-L45)：

```sql
WHERE links.some(id = linkId)
  AND (ownerId = userId            -- 集合所有者
       OR members.some(userId)     -- 集合成员
       OR isPublic = true)         -- 公开集合
```

**归档访问判断矩阵（订阅降级后）：**

| 场景 | 集合属性 | 登录用户 | 降级后能否读归档 |
|------|----------|----------|------------------|
| 自己的私有集合 | isPublic=false, ownerId=me | ✅ 是我 | ✅ 可以（集合权限通过） |
| 我加入的私有集合 | isPublic=false, members include me | ✅ 是成员 | ✅ 可以（集合权限通过） |
| 他人的私有集合 | isPublic=false, 我非成员 | ✅ 已登录 | ❌ 401 "You don't have access to this collection." |
| 公开集合 | isPublic=true | 任意（含未登录） | ✅ 可以（通过 isPublic） |

### 3.7 订阅有效性判定逻辑

**核心函数 `verifySubscription()`**，见 [verifySubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/verifySubscription.ts#L13-L86)

```
verifySubscription(user) 判定流程：

1. user 为空 → return null
2. 计算试用期：trialEndTime = createdAt + (TRIAL_PERIOD_DAYS + 1) 天
3. 快速失活判定：
   无 subscriptions && 无 parentSubscription && (REQUIRE_CC || daysLeft <= 0)
   → return null
4. 子用户判定：parentSubscription.active === true → return user（有效）
5. 主用户判定：
   subscriptions.active && 当前时间 < currentPeriodEnd → return user（有效）
6. 兜底：从 Stripe 拉取最新状态
   └─ 调用 checkSubscriptionByEmail(user.email)
      ├─ 拉取成功 → upsert 到本地 DB → return user
      └─ 拉取失败 → return null
```

**关于试用期的关键说明：**
- `REQUIRE_CC=true`：必须绑卡，试用期不受 `daysLeft` 影响，未绑卡无订阅即失效
- `REQUIRE_CC=false`：免绑卡试用，`daysLeft > 0` 时即使无订阅也有效
- 试用期计算额外 +1 天：`(1 + TRIAL_PERIOD_DAYS) * 86400000`，用于兼容"当天"

### 3.8 从 Stripe 拉取订阅信息

函数 [checkSubscriptionByEmail.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/checkSubscriptionByEmail.ts#L5-L29)

通过邮箱从 Stripe API 查询客户订阅，返回：

```typescript
{
  active: sub.status === "active" || sub.status === "trialing",
  stripeSubscriptionId: sub.id,
  currentPeriodStart: item.current_period_start * 1000,
  currentPeriodEnd: item.current_period_end * 1000,
  quantity: item.quantity ?? 1,
}
```

### 3.9 Stripe Webhook 同步

Webhook 端点 [webhook/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/webhook/index.ts#L27-L114)

| Stripe 事件 | 处理行为（handleSubscription） |
|-------------|-------------------------------|
| `customer.subscription.created` | upsert 本地 Subscription 记录 |
| `customer.subscription.updated` | 更新 `active`、`quantity`、`periodStart`、`periodEnd` |
| `customer.subscription.deleted` | 将 `active` 设为 false（保留历史数据） |

处理函数 [handleSubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/handleSubscription.ts#L17-L98)

---

## 4. 计划读取与功能开关的关系

前端通过 `useUser()` 获取订阅状态后，各组件基于以下字段做功能开关：

### 4.1 功能可见性控制总览

| 功能 / UI 元素 | 控制条件 | 代码位置 |
|----------------|----------|----------|
| **试用期顶部横幅** | `NEXT_PUBLIC_STRIPE && !user.subscription.active && !user.parentSubscription.active`（即 isTrialing） | [Navbar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/components/Navbar.tsx#L81-L104) |
| **Settings → Billing 菜单** | `NEXT_PUBLIC_STRIPE && !user.parentSubscriptionId`（子用户看不到账单） | [SettingsSidebar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/components/SettingsSidebar.tsx#L126-L141) |
| **删除账户中的反馈表单** | `NEXT_PUBLIC_STRIPE && !user.parentSubscriptionId`（仅主订阅用户显示取消原因） | [delete.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/settings/delete.tsx#L93-L133) |
| **member-onboarding 页面** | 已登录 + `!user.name && user.parentSubscriptionId`（被邀请成员首次登录） | [AuthRedirect.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/layouts/AuthRedirect.tsx#L63-L66) |
| **subscribe 页面自动跳转** | 已激活订阅（自身或父订阅）→ 跳转 dashboard | [subscribe.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/subscribe.tsx#L41-L48) |
| **账单页面自动跳转** | 子用户或试用期未结束 → 跳转其他页面 | [billing.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/settings/billing.tsx#L45-L59) |

### 4.2 试用期横幅显示逻辑

[Navbar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/components/Navbar.tsx#L68-L104)

```typescript
const isTrialing =
  user?.id &&
  !user?.subscription?.active &&
  !user.parentSubscription?.active;

// STRIPE_ENABLED && isTrialing → 显示 "X days left in your free trial"
```

注意：该逻辑不检查 `daysLeft`，只要两个订阅 active 都是 false 就视为"试用中"，即使试用期实际已过。

### 4.3 subscribe 页面文案逻辑

[subscribe.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/subscribe.tsx#L63-L75)

| 条件 | 显示文案 |
|------|----------|
| `REQUIRE_CC=true` | "Start with a 14-day free trial, cancel anytime!" |
| `REQUIRE_CC=false && daysLeft <= 0` | "Your free trial has ended, subscribe to continue." |
| `REQUIRE_CC=false && daysLeft > 0` | "You have X days left in your free trial." |

---

## 5. 配额检查逻辑（三套独立检查）

### 5.1 环境变量配置

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `MAX_LINKS_PER_USER` | 30000 | 每个席位的最大链接数 |
| `NEXT_PUBLIC_TRIAL_PERIOD_DAYS` | 14 | 试用天数 |
| `NEXT_PUBLIC_REQUIRE_CC` | false | 是否需要绑卡才能试用 |
| `RSS_SUBSCRIPTION_LIMIT_PER_USER` | 20 | 每用户 RSS 订阅上限 |
| `NEXT_PUBLIC_MAX_FILE_BUFFER` | 10 | 上传文件大小限制（MB） |
| `STRIPE_SECRET_KEY` / `NEXT_PUBLIC_STRIPE` | - | 是否启用付费订阅模式 |

### 5.2 检查一：hasPassedLimit —— 完整配额检查

位置 [verifyCapacity.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/lib/verifyCapacity.ts#L8-L109)

```typescript
hasPassedLimit(userId: number, numberOfImports: number): Promise<boolean>
// return true = 超限，false = 未超限
```

**完整判断流程图：**

```
hasPassedLimit(userId, numberOfImports)
│
├─ 【Stripe 未启用】!stripeEnabled
│   └─ 统计 createdById=userId 的 links → 与 MAX_LINKS_PER_USER 比较
│
└─ 【Stripe 启用】
    ├─ 读取 user: parentSubscriptionId、subscriptions { id, quantity }、createdAt
    │
    ├─ 【免绑卡且试用期内】!REQUIRE_CC && daysLeft > 0
    │   └─ 统计 createdById=userId 的 links → 与 MAX_LINKS_PER_USER 比较
    │
    └─ 【正式订阅模式】
        ├─ 确定 subscriptionId 和 quantity
        │   ├─ 子用户: parentSubscriptionId, quantity 从 subscription 表查
        │   └─ 主用户: subscriptions.id, subscriptions.quantity
        │   └─ 任一为空 → return true（视为超限）
        │
        ├─ 【子用户席位检查】user.parentSubscriptionId 存在
        │   └─ childCount = 有多少用户的 parentSubscriptionId = 此 subscriptionId
        │   └─ childCount + 1 > quantity → return true（成员超席位）
        │
        └─ 【链接数检查】
            ├─ 【组织模式】parentSubscriptionId 存在 || quantity > 1
            │   └─ totalCapacity = quantity × MAX_LINKS_PER_USER
            │   └─ 统计所有关联用户（子用户 + 主用户）创建的 links
            │   └─ totalCapacity - (numberOfImports + totalLinks) < 0 → return true
            │
            └─ 【个人模式】
                └─ 统计 createdById=userId 的 links → 与 MAX_LINKS_PER_USER 比较
```

**调用点：**

| 场景 | 文件 |
|------|------|
| 创建新链接 | [postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/links/postLink.ts#L69-L76) |
| 导入 HTML 书签 | [importFromHTMLFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L25-L32) |
| 从 Pocket 导入 | importFromPocket.ts |
| 从 Wallabag 导入 | importFromWallabag.ts |
| 从 Omnivore 导入 | importFromOmnivore.ts |
| 从 Linkwarden 导入 | importFromLinkwarden.ts |
| RSS 自动采集 | [rssHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/lib/rssHandler.ts#L43-L54) |

### 5.3 检查二：verifyLinkLimit —— 归档上传的简化检查

位置 [archives/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/archives/index.ts#L27-L41) 和 [archives/[linkId].ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/archives/%5BlinkId%5D.ts#L40-L55)

```typescript
async function verifyLinkLimit(userId: number) {
  const MAX_LINKS_PER_USER = Number(process.env.MAX_LINKS_PER_USER || 30000);
  const userLinkCount = await prisma.link.count({
    where: { collection: { ownerId: userId } },
  });
  if (userLinkCount > MAX_LINKS_PER_USER) {
    throw new Error(`Each collection owner can only have a maximum of ${MAX_LINKS_PER_USER} Links.`);
  }
}
```

**⚠️ 与 hasPassedLimit 的重要区别：**

| 维度 | hasPassedLimit | verifyLinkLimit |
|------|---------------|-----------------|
| 统计范围 | `createdById`（谁创建的链接） | `collection.ownerId`（谁拥有集合） |
| 组织模式支持 | ✅ 支持（子用户/多席位聚合统计） | ❌ 不支持（仅查单个 userId） |
| 试用期处理 | ✅ 完整处理 REQUIRE_CC 和 daysLeft | ❌ 无试用期逻辑 |
| 席位数量检查 | ✅ 检查成员数是否超 quantity | ❌ 无 |
| 调用前是否已拦截 | 调用前已通过 verifyUser 全局订阅检查 | 同样已通过 verifyUser |
| 错误信息 | "Your subscription has reached the maximum number of links allowed." | "Each collection owner can only have a maximum of X Links." |

### 5.4 检查三：validateFile —— 归档上传的文件大小和类型检查

位置 [archives/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/archives/index.ts#L44-L65) 和 [archives/[linkId].ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/archives/%5BlinkId%5D.ts#L57-L79)

```typescript
function validateFile(file, maxMB, allowedTypes) {
  // 1. 检查 MIME 类型是否在白名单内
  // 2. 检查文件 Buffer 大小是否 ≤ maxMB
}
```

- **允许的 MIME 类型**：`application/pdf`、`image/png`、`image/jpg`、`image/jpeg`、`text/html`
- **大小限制**：`NEXT_PUBLIC_MAX_FILE_BUFFER` MB（默认 10MB）

### 5.5 检查四：RSS 订阅数量限制

位置 [rss/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/rss/index.ts#L53-L66)

```typescript
if (rssSubscriptionCount >= RSS_SUBSCRIPTION_LIMIT_PER_USER) {
  return res.status(403).json({
    response: `You have reached the limit of ${RSS_SUBSCRIPTION_LIMIT_PER_USER} RSS subscriptions.`
  });
}
```

独立于链接数配额，默认每个用户最多 20 个 RSS 订阅。

---

## 6. 团队成员（Seat）上限与自动增减

### 6.1 成员邀请创建

位置 [postUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/users/postUser.ts#L84-L132)

邀请用户时，将新用户的 `parentSubscriptionId` 设为邀请者的订阅 ID：

```typescript
parentSubscription: parentUser && invite
  ? { connect: { id: (parentUser.subscriptions as Subscription).id } }
  : undefined,
```

**邀请前提条件：**
- `stripeEnabled`（STRIPE_SECRET_KEY 存在）
- `emailEnabled`（EMAIL_FROM 和 EMAIL_SERVER 存在）
- 邀请者必须已登录

### 6.2 席位自动增加 —— 真实触发条件

**触发位置**：[auth/[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1326-L1363) 中的 `signIn` callback

**真实触发条件（必须同时满足）：**

1. 用户 `!emailVerified`（首次登录验证邮箱）
2. 用户有 `parentSubscriptionId`（是被邀请的子用户）
3. `STRIPE_SECRET_KEY` 存在
4. `verifiedChildUsersCount + 2 > parentSubscription.quantity`

**席位数量计算公式：**
```
verifiedChildUsersCount = 已验证邮箱且同属一个 subscription 的子用户数（排除当前用户）
新增席位 = verifiedChildUsersCount + 2  （当前用户 + 管理员 + 其他已验证成员）
```

也就是说：邀请时**不会**立即增加席位，只有当被邀请用户实际点击邀请链接完成邮箱验证时，才检测是否需要扩容。

### 6.3 席位自动减少 —— 真实触发条件

有两个场景会触发减座：

#### 场景 A：管理员移除团队成员

位置 [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L84-L103)

触发条件：
- 调用者是订阅主用户（通过权限检查）
- 被移除用户是其 `childUsers` 之一
- 被移除用户 `emailVerified` 为 true（已验证过的才占席位）
- 操作：仅断开 `parentSubscription` 关联，**不删除子用户账户本身**

```typescript
if (removeUser.emailVerified)
  await updateSeats(user.subscriptions.stripeSubscriptionId, user.subscriptions.quantity - 1);
```

#### 场景 B：子用户主动注销自己的账户

位置 [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L183-L192)

触发条件：
- 用户自己请求删除
- 用户有 `parentSubscriptionId`（是子用户）
- 用户 `emailVerified` 为 true

### 6.4 updateSeats 函数

位置 [updateSeats.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/updateSeats.ts#L6-L26)

```typescript
await stripe.subscriptions.update(subscriptionId, {
  billing_cycle_anchor: trialing ? undefined : "now",    // 试用期不改计费锚点
  proration_behavior: trialing ? undefined : "create_prorations", // 试用期不按比例计费
  quantity: seats,
});
```

**⚠️ 注意**：席位增减是直接调用 Stripe API 修改订阅 quantity，会立即影响用户账单（非试用期）。

---

## 7. 降级后的真实行为

### 7.1 降级触发条件汇总

| 降级场景 | 触发条件 |
|----------|----------|
| 订阅过期 | `new Date() > subscription.currentPeriodEnd` 且未自动续费 |
| 订阅被取消 | Stripe 推送 `customer.subscription.deleted` → `active = false` |
| 试用过期（免绑卡模式） | `daysLeft <= 0 && !REQUIRE_CC && 无有效 subscription` |
| 强制下线 | Stripe 端将订阅状态改为非 active / trialing |

### 7.2 降级后分层行为

#### 第一层：前端路由跳转（AuthRedirect）

位置 [AuthRedirect.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/layouts/AuthRedirect.tsx#L34-L80)

```typescript
const hasInactiveSubscription =
  user?.id &&
  !user?.subscription?.active &&
  !user.parentSubscription?.active &&
  STRIPE_ENABLED &&
  (REQUIRE_CC || daysLeft <= 0);

// isLoggedIn && hasInactiveSubscription → 重定向到 /subscribe
```

**前端页面访问边界：**

| 页面路径 | 降级后能否访问 | 说明 |
|----------|----------------|------|
| `/login`、`/register`、`/forgot` | ✅ 可以 | 登录注册相关 |
| `/subscribe` | ✅ 可以 | 订阅/续费页面 |
| `/public/collections/[id]` | ✅ 可以 | 公开集合页面 |
| `/public/links/[id]` | ✅ 可以 | 公开链接详情页面 |
| `/dashboard`、`/collections`、`/links` | ❌ 被重定向 | 私有数据页面 |
| `/settings/*` | ❌ 被重定向 | 设置页面 |
| `/tags`、`/preserved`、`/search` | ❌ 被重定向 | 其他受保护页面 |
| `/member-onboarding` | 条件性 | 仅被邀请未设置用户名的成员可访问 |

#### 第二层：API 拦截（区分 verifyUser 和非 verifyUser 路径）

**经过 verifyUser 的路径**（约 30+ 个端点）：启用 Stripe 时订阅校验失败直接返回：

```json
HTTP 401
{ "response": "You are not a subscriber, feel free to reach out to us at support@linkwarden.app if you think this is an issue." }
```

**⚠️ 重要修正：并非所有 `/api/v1/*` 都经过 verifyUser。**

降级后仍可正常调用的 API（不经过订阅校验或校验仅在凭证阶段）：

| API | 行为 |
|-----|------|
| `GET /api/v1/archives/[linkId]` | ✅ 正常返回归档文件（受集合权限控制） |
| `GET /api/v1/preserved/token` | ✅ 正常返回归档临时 URL |
| `GET /api/v1/preserved/view?token=...` | ✅ 正常返回归档文件（独立签名 token） |
| **`GET /api/v1/users/[id]`** | **✅ 正常返回用户信息**（只能读自己或管理员读任意，不检查订阅） |
| **`GET /api/v1/users/me`** | **✅ 正常返回当前用户信息**（含 subscription 状态供前端判断） |
| `GET /api/v1/avatar/[id]` | ✅ 正常返回头像文件 |
| **`GET /api/v1/payment`** | **✅ 正常可用**（仅 GET，Stripe 支付必须允许非订阅者访问） |
| `GET /api/v1/public/*` | ✅ 正常返回公开集合/链接/用户数据 |
| `GET /api/v1/config` | ✅ 正常返回实例配置 |
| `GET /api/v1/logins` | ✅ 正常返回登录方式列表 |
| `GET /api/v1/getFavicon` | ✅ 正常代理 favicon |
| `POST /api/v1/users`（无 invite 注册） | ✅ 新用户注册正常可用 |
| `POST /api/v1/auth/forgot-password` | ✅ 正常发送密码重置邮件 |
| `POST /api/v1/auth/verify-email` | ✅ 邮箱验证正常可用 |
| `POST /api/v1/auth/reset-password` | ✅ 密码重置正常可用 |
| `POST /api/v1/auth/[...nextauth]`（Web 登录） | ✅ 登录正常成功（后续 API 调用被拦截） |
| `POST /api/v1/webhook` | ✅ Stripe Webhook 始终可用 |
| **`POST /api/v1/session`** | **❌ 凭证校验失败**（通过 verifyByCredentials → verifySubscription 完整检查订阅） |

**⚠️ 重要修正：`POST /api/v1/session`（创建 API Token）在降级后无法使用。** 该接口通过 `verifyByCredentials` 调用了完整的 `verifySubscription`，订阅失效会返回 `Invalid credentials` 错误，与密码错误相同。

降级后被拦截的 API（经过 verifyUser 完整检查）：

| API | 行为 |
|-----|------|
| **`PUT/DELETE /api/v1/users/[id]`** | **❌ 401 非订阅者**（同一接口 GET 可访问，PUT/DELETE 检查订阅） |
| `/api/v1/users/[id]/preference`（PUT） | ❌ 401 非订阅者 |
| `/api/v1/users`（GET 列表） | ❌ 401 非订阅者 |
| `/api/v1/links/*`（所有方法） | ❌ 401 非订阅者 |
| `/api/v1/collections/*`（除 public） | ❌ 401 非订阅者 |
| `/api/v1/tags/*`、`/api/v1/highlights/*` | ❌ 401 非订阅者 |
| `/api/v1/rss/*` | ❌ 401 非订阅者 |
| `POST /api/v1/archives/*`（上传/更新归档） | ❌ 401 非订阅者 |
| `/api/v1/dashboard/*`、`/api/v1/search/*` | ❌ 401 非订阅者 |
| `/api/v1/migration/*`（数据导入） | ❌ 401 非订阅者 |
| `/api/v1/tokens/*`（Token 管理） | ❌ 401 非订阅者 |
| `/api/v1/worker/*` | ❌ 401 非订阅者（且需管理员） |
| `/api/v1/users`（带 `invite` 参数邀请成员） | ⚠️ 可能被拦截（isAuthenticatedRequest 简化检查有缺陷） |

#### 第三层：Worker 后台任务停止处理

位置 [getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/worker/lib/getLinkBatchFairly.ts#L45-L77)

Worker 选取待处理链接时，**只挑选有有效订阅的用户**：

```typescript
where: {
  // ...链接过滤条件
  ...(process.env.STRIPE_SECRET_KEY
    ? {
        OR: [
          { subscriptions: { is: { active: true } } },
          { parentSubscription: { is: { active: true } } },
          ...(REQUIRE_CC ? [] : [
            { createdAt: { gte: 试用期内 } }  // 免绑卡模式下试用期用户也算
          ]),
        ],
      }
    : {}),
}
```

同样，[countUnprocessedBillableLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/worker/lib/countUnprocessedBillableLinks.ts#L3-L29) 统计待处理队列时也过滤非订阅用户。

**影响**：
- 链接归档（PDF/Screenshot/Readable/Monolith）停止处理
- AI 自动打标停止处理
- 已存在的归档文件不会被删除，但新增链接不会有归档

#### 第四层：业务操作配额检查

即使前三层都通过（如 Stripe 临时故障），在具体创建链接、导入数据时，`hasPassedLimit()` 和 `verifyLinkLimit()` 仍会检查。

### 7.3 降级后已有归档资源的访问边界与数据保留

降级后**不会删除任何用户数据**，但不同资源的可访问性取决于访问路径是否经过 verifyUser。

#### 7.3.1 数据保留概览

| 资源类型 | 是否保留 | 物理存储位置 |
|----------|----------|--------------|
| 已有链接（Link 记录） | ✅ 保留 | 数据库 links 表 |
| 归档文件（PDF/图片/HTML/Readable） | ✅ 保留 | 文件系统 archives/{collectionId}/ 目录 |
| 归档预览图 | ✅ 保留 | 文件系统 archives/preview/{collectionId}/ 目录 |
| 集合（Collection） | ✅ 保留 | 数据库 collections 表 |
| 标签（Tag） | ✅ 保留 | 数据库 tags 表 |
| 高亮（Highlight） | ✅ 保留 | 数据库 highlights 表 |
| 团队成员关系 | ✅ 保留 | parentSubscriptionId 字段不清空 |
| 用户账户 | ✅ 保留 | users 表记录保留 |

#### 7.3.2 已有归档资源的实际可访问性

归档资源存在多条读取路径，降级后能否访问取决于**走哪条路径**：

| 访问方式 | 降级后能否访问 | 说明 |
|----------|----------------|------|
| **前端私有页面查看**（/links/[id]） | ❌ 不能 | 路由被 AuthRedirect 重定向到 /subscribe |
| **前端公开页面查看**（/public/links/[id]） | ✅ 可以 | /public/* 不受 AuthRedirect 拦截，且归档 GET API 不检查订阅 |
| **直接调用 GET /api/v1/archives/[linkId]** | ✅ 可以 | 此 API 使用 verifyToken + resolveAccessibleArchive，不经过 verifyUser。只要持有有效会话 token 且对集合有访问权限即可读取 |
| **调用 GET /api/v1/preserved/token** | ✅ 可以 | 获取归档临时签名 URL，同样不检查订阅 |
| **调用 GET /api/v1/preserved/view?token=...** | ✅ 可以 | 独立签名 token，不依赖用户会话或订阅 |
| **前端查看链接列表**（/dashboard） | ❌ 不能 | 路由被重定向，且 GET /api/v1/links 经过 verifyUser 会返回 401 |
| **公开集合的归档** | ✅ 任何人 | 公开集合（isPublic=true）的归档无需登录即可读取，完全不受订阅影响 |

**关键结论：降级后，只要用户的 JWT 会话尚未过期，且目标集合允许其访问（owner/member/isPublic），用户可以通过直接请求归档 API 来下载和查看已有归档文件——尽管前端界面会被重定向。公开集合的归档资源降级后对任何人都仍可访问。**

#### 7.3.3 各资源访问路径与拦截情况

| 资源 | 读 API 路径 | 是否经过 verifyUser | 降级后能否读取 |
|------|------------|---------------------|----------------|
| **私有归档文件** | GET `/api/v1/archives/[linkId]` | ❌ 否（verifyToken + 集合权限） | ✅ 会话有效 + 集合权限 |
| **公开归档文件** | GET `/api/v1/archives/[linkId]` | ❌ 否 | ✅ 任何人（通过 isPublic） |
| **归档临时 URL** | GET `/api/v1/preserved/token` | ❌ 否（verifyToken + 集合权限） | ✅ 会话有效 + 集合权限 |
| **归档签名访问** | GET `/api/v1/preserved/view?token=...` | ❌ 否（独立签名 token） | ✅ 签名 token 有效即可 |
| **指定用户信息** | **GET `/api/v1/users/[id]`** | **❌ 否（仅 verifyToken，不检查订阅）** | ✅ 会话有效 + （读自己 \|\| 管理员） |
| **当前用户信息** | GET `/api/v1/users/me` | ❌ 否（仅 verifyToken） | ✅ 会话有效即可 |
| **支付结账** | **GET `/api/v1/payment`** | **❌ 否（仅 getToken，无 POST 方法）** | ✅ 会话有效即可 |
| **私有链接元数据** | GET `/api/v1/links/[id]`、GET `/api/v1/links` | ✅ 是 | ❌ 401 非订阅者 |
| **公开链接元数据** | GET `/api/v1/public/links/[id]` | ❌ 否 | ✅ 任何人 |
| **私有集合数据** | GET `/api/v1/collections/[id]` | ✅ 是 | ❌ 401 非订阅者 |
| **公开集合数据** | GET `/api/v1/public/collections/[id]` | ❌ 否 | ✅ 任何人 |
| **标签/高亮** | GET `/api/v1/tags`、`/api/v1/highlights` | ✅ 是 | ❌ 401 非订阅者 |
| **用户头像** | GET `/api/v1/avatar/[id]` | ❌ 否（仅 verifyToken） | ✅ 会话有效即可 |
| **Dashboard 数据** | GET `/api/v1/dashboard` | ✅ 是 | ❌ 401 非订阅者 |
| **搜索** | GET `/api/v1/search` | ✅ 是 | ❌ 401 非订阅者 |

### 7.4 完整错误提示清单

| 场景 | 错误信息 | HTTP 状态码 |
|------|----------|------------|
| **非订阅者（verifyUser 全局拦截）** | `You are not a subscriber, feel free to reach out to us at support@linkwarden.app if you think this is an issue.` | 401 |
| **非订阅者（verifyByCredentials 创建 Session）** | `Invalid credentials. You might need to reset your password if you're sure you already signed up with the current username/email.` | 400 |
| **链接数超限（hasPassedLimit）** | `Your subscription has reached the maximum number of links allowed.` | 400 |
| **归档链接数超限（verifyLinkLimit）** | `Each collection owner can only have a maximum of ${MAX_LINKS_PER_USER} Links.` | 400 |
| **归档集合无权限** | `You don't have access to this collection.` | 401 |
| **RSS 订阅超限** | `You have reached the limit of ${RSS_SUBSCRIPTION_LIMIT_PER_USER} RSS subscriptions.` | 403 |
| **文件大小超限** | `Sorry, we couldn't process your file. Please ensure it doesn't exceed ${maxMB}MB.` | 400 |
| **文件类型不支持** | `Sorry, we couldn't process your file. Please ensure it's in [${allowedTypes}] format and doesn't exceed ${maxMB}MB.` | 400 |
| **邮箱未验证** | `Email not verified, please verify your email to continue using Linkwarden.` | 401 |
| **成员被成功移除** | `Account removed from subscription.` | 200 |
| **无权限邀请成员** | `You are not authorized to invite users.` | 401 |
| **演示模式** | `This action is disabled because this is a read-only demo of Linkwarden.` | 400 |
| **注册被禁用** | `Registration is disabled.` | 400 |

---

## 8. 支付与计费相关代码索引

| 模块 | 文件路径 |
|------|----------|
| 数据模型 | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/prisma/schema.prisma) |
| 全局 API 订阅拦截（完整） | [verifyUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/verifyUser.ts) |
| 凭证校验中的订阅检查（完整） | [verifyByCredentials.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/verifyByCredentials.ts) |
| 简化订阅检查（有缺陷） | [isAuthenticatedRequest.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/isAuthenticatedRequest.ts) |
| 纯会话校验（无订阅检查） | [verifyToken.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/verifyToken.ts) |
| 订阅有效性判定（核心） | [verifySubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/verifySubscription.ts) |
| Stripe 远程查询订阅 | [checkSubscriptionByEmail.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/checkSubscriptionByEmail.ts) |
| 完整配额检查（链接/席位） | [verifyCapacity.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/lib/verifyCapacity.ts) |
| 归档权限判定（无订阅检查） | [resolveAccessibleArchive.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/archives/resolveAccessibleArchive.ts) |
| 归档读取 API（GET，无订阅检查） | [archives/[linkId].ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/archives/%5BlinkId%5D.ts) |
| **用户 CRUD（混合校验：GET 不检查，PUT/DELETE 检查）** | [users/[id]/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/users/%5Bid%5D/index.ts) |
| **用户偏好设置（verifyUser 完整检查）** | [users/[id]/preference.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/users/%5Bid%5D/preference.tsx) |
| 获取当前用户（无订阅检查） | [users/me.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/users/me.ts) |
| 支付结账（无订阅检查，仅 GET） | [payment/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/payment/index.ts) |
| 订阅变更处理 | [handleSubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/handleSubscription.ts) |
| Stripe Webhook 端点 | [webhook/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/webhook/index.ts) |
| Stripe 席位更新 | [updateSeats.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/updateSeats.ts) |
| 支付结账流程 | [paymentCheckout.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/paymentCheckout.ts) |
| 获取用户（含订阅状态） | [getUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/users/userId/getUserById.ts) |
| 创建新链接 | [postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/links/postLink.ts) |
| 归档上传（含简化配额检查） | [archives/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/archives/index.ts) |
| Worker 过滤非订阅用户 | [getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/worker/lib/getLinkBatchFairly.ts) |
| RSS 自动采集配额检查 | [rssHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/lib/rssHandler.ts) |
| RSS 订阅数量限制 | [rss/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/rss/index.ts) |
| 前端路由守卫 | [AuthRedirect.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/layouts/AuthRedirect.tsx) |
| 前端用户数据 hook | [user.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/router/user.tsx) |
| 订阅页面 | [subscribe.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/subscribe.tsx) |
| 账单设置（成员管理） | [billing.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/settings/billing.tsx) |
| 邀请成员弹窗 | [InviteModal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/components/ModalContent/InviteModal.tsx) |
| 顶部导航（试用横幅） | [Navbar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/components/Navbar.tsx) |
| 设置侧边栏（功能可见性） | [SettingsSidebar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/components/SettingsSidebar.tsx) |
| 成员入职引导 | [member-onboarding.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/member-onboarding.tsx) |
| 创建用户（含邀请） | [postUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/users/postUser.ts) |
| 删除用户（含席位处理） | [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts) |
| 登录回调（席位自动增加） | [auth/[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts) |
| 删除账户页面 | [delete.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/settings/delete.tsx) |
