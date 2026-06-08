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
- `/api/v1/users/*`、`/api/v1/links/*`、`/api/v1/collections/*`
- `/api/v1/tags/*`、`/api/v1/highlights/*`、`/api/v1/rss/*`
- `/api/v1/archives/*`、`/api/v1/dashboard/*`、`/api/v1/search/*`
- `/api/v1/migration/*`、`/api/v1/tokens/*` 等

### 3.3 订阅有效性判定逻辑

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

### 3.4 从 Stripe 拉取订阅信息

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

### 3.5 Stripe Webhook 同步

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

**影响页面**：所有受保护路由（/dashboard、/settings、/collections、/links、/tags、/preserved、/search 等）都会被强制跳转到 `/subscribe` 页面。只有 /login、/register、/forgot、/subscribe、/public/* 等可访问。

#### 第二层：API 全局拦截（verifyUser）

位置 [verifyUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/verifyUser.ts#L60-L70)

所有 `/api/v1/*` 请求在启用 Stripe 时都会经过订阅校验，不通过直接返回：

```json
HTTP 401
{ "response": "You are not a subscriber, feel free to reach out to us at support@linkwarden.app if you think this is an issue." }
```

这意味着：降级后即使绕过前端跳转，手动调用 API 也无法进行任何写操作（创建链接、上传、导入等），甚至大部分读操作也会被拦截。

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

### 7.3 降级后数据保留

降级后**不会删除任何用户数据**，所有已有资源全部保留：

| 资源类型 | 是否保留 | 能否查看 |
|----------|----------|----------|
| 已有链接（Link 记录） | ✅ 保留 | ✅ 数据库中存在，但 API 被拦截可能无法前端查看 |
| 归档文件（PDF/图片等） | ✅ 文件系统中保留 | ✅ 同上 |
| 集合（Collection） | ✅ 保留 | ✅ 同上 |
| 标签（Tag） | ✅ 保留 | ✅ 同上 |
| 高亮（Highlight） | ✅ 保留 | ✅ 同上 |
| 团队成员关系 | ✅ 保留（parentSubscriptionId 不清空） | ✅ 同上 |
| 用户账户 | ✅ 保留 | ⚠️ 需重新订阅后才能正常使用 |

### 7.4 完整错误提示清单

| 场景 | 错误信息 | HTTP 状态码 |
|------|----------|------------|
| **非订阅者（全局 API 拦截）** | `You are not a subscriber, feel free to reach out to us at support@linkwarden.app if you think this is an issue.` | 401 |
| **链接数超限（hasPassedLimit）** | `Your subscription has reached the maximum number of links allowed.` | 400 |
| **归档链接数超限（verifyLinkLimit）** | `Each collection owner can only have a maximum of ${MAX_LINKS_PER_USER} Links.` | 400 |
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
| 全局 API 订阅拦截 | [verifyUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/verifyUser.ts) |
| 订阅有效性判定 | [verifySubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/verifySubscription.ts) |
| Stripe 远程查询订阅 | [checkSubscriptionByEmail.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/checkSubscriptionByEmail.ts) |
| 完整配额检查（链接/席位） | [verifyCapacity.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/lib/verifyCapacity.ts) |
| 订阅变更处理 | [handleSubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/handleSubscription.ts) |
| Stripe Webhook 端点 | [webhook/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/webhook/index.ts) |
| Stripe 席位更新 | [updateSeats.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/updateSeats.ts) |
| 支付结账 | [paymentCheckout.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/paymentCheckout.ts) |
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
