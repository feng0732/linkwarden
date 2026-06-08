# Subscription/Plan 限制机制梳理

## 1. 概述

Linkwarden 的订阅限制体系基于 Stripe 订阅系统，通过 `Subscription` 数据模型、配额检查函数、前端路由守卫和 Stripe Webhook 共同实现。核心限制维度包括：**链接数量上限、团队成员（Seat）数量、RSS 订阅数量、文件上传大小等。

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

**关键字段说明：

| 字段 | 含义 |
|------|------|
| `active` | 订阅是否激活（由 Stripe Webhook 同步） |
| `quantity` | 购买的席位数量（决定团队成员上限） |
| `currentPeriodEnd` | 当前计费周期结束时间，用于判断订阅是否过期 |
| `childUsers` | 通过 `parentSubscriptionId` 关联的子用户 |

### 2.2 User 模型与订阅关联

定义于 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/prisma/schema.prisma#L28-L75)

```prisma
model User {
  // ...
  parentSubscription      Subscription?         @relation("ChildUsers", fields: [parentSubscriptionId], references: [id])
  parentSubscriptionId    Int?
  subscriptions           Subscription?
  // ...
}
```

**用户与订阅的两种关联方式：**

1. **主订阅用户（Subscription.user** — 用户自己购买的订阅，通过 `User.subscriptions` 关联，1:1
2. **子用户（团队成员）** — 通过 `User.parentSubscriptionId` 关联到主用户的订阅，1:N

---

## 3. 订阅计划读取流程

### 3.1 订阅状态验证

核心函数：[verifySubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/verifySubscription.ts#L13-L86)

```
verifySubscription(user) 流程：

1. 检查用户是否存在
2. 计算试用期剩余天数（基于 createdAt + TRIAL_PERIOD_DAYS
3. 判定逻辑：
   - 无 subscriptions && 无 parentSubscription && (REQUIRE_CC 或 daysLeft <= 0) → return null（订阅失效
   - parentSubscription.active === true → 有效（子用户
   - subscriptions.active === true && 当前时间 < currentPeriodEnd → 有效
   - 否则，尝试调用 checkSubscriptionByEmail() 从 Stripe 拉取最新状态并同步到本地 DB
```

### 3.2 从 Stripe 拉取订阅信息

函数：[checkSubscriptionByEmail.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/checkSubscriptionByEmail.ts#L5-L29)

通过邮箱从 Stripe API 查询客户订阅信息，返回：

```typescript
{
  active: sub.status === "active" || sub.status === "trialing",
  stripeSubscriptionId,
  currentPeriodStart,
  currentPeriodEnd,
  quantity
}
```

### 3.3 Stripe Webhook 同步

Webhook 端点：[webhook/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/webhook/index.ts#L27-L114)

监听三种事件：

| 事件 | 处理逻辑：

| Stripe 事件 | 处理函数 |
|-------------|----------|
| `customer.subscription.created` | 创建/更新本地 Subscription 记录
| `customer.subscription.updated` | 更新 active、quantity、periodStart、periodEnd
| `customer.subscription.deleted` | 将 active 设为 false

处理函数：[handleSubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/handleSubscription.ts#L17-L98)

---

## 4. 配额检查逻辑

### 4.1 核心检查函数 hasPassedLimit

位置：[verifyCapacity.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/lib/verifyCapacity.ts#L8-L109)

```typescript
hasPassedLimit(userId: number, numberOfImports: number)
```

**判断流程图：**

```
hasPassedLimit(userId, numberOfImports)
│
├─ Stripe 未启用（!stripeEnabled）
│   └─ 统计该用户创建的 links → 与 MAX_LINKS_PER_USER 比较
│
├─ Stripe 启用
│   ├─ 读取用户：parentSubscriptionId、subscriptions、createdAt
│   │
│   ├─ 试用期检查（!REQUIRE_CC && daysLeft > 0）
│   │   └─ 试用期内：统计用户 links → 与 MAX_LINKS_PER_USER 比较
│   │
│   └─ 正式订阅检查
│       ├─ 确定 subscriptionId 和 quantity
│       │   ├─ 子用户：parentSubscriptionId
│       │   └─ 主用户：subscriptions.id
│       │
│       ├─ 子用户成员数量检查（user.parentSubscriptionId
│       │   └─ 统计 childCount = parentSubscriptionId 的子用户数
│       │   └─ childCount + 1 > quantity → return true（超限）
│       │
│       └─ 链接数检查
│           ├─ 组织模式（子用户 或 quantity > 1）
│           │   └─ totalCapacity = quantity × MAX_LINKS_PER_USER
│           │   └─ 统计组织内所有用户的 links
│           │   └─ totalCapacity - (numberOfImports + totalLinks) < 0 → return true
│           │
│           └─ 个人模式
│               └─ 统计用户个人 links → 与 MAX_LINKS_PER_USER 比较
```

### 4.2 环境变量配置

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `MAX_LINKS_PER_USER | 30000 | 每个席位最大链接数 |
| `NEXT_PUBLIC_TRIAL_PERIOD_DAYS | 14 | 试用天数 |
| `NEXT_PUBLIC_REQUIRE_CC | false | 是否需要信用卡（true=需要绑卡才能试用 |
| `RSS_SUBSCRIPTION_LIMIT_PER_USER | 20 | 每用户 RSS 订阅上限 |
| `NEXT_PUBLIC_MAX_FILE_BUFFER | 10 | 上传文件大小限制（MB） |

---

## 5. 各功能开关

### 5.1 链接（Link）数量限制检查调用点

| 场景 | 文件 | 位置 |
|------|------|------|
| 创建新链接 | [postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/links/postLink.ts#L69-L76) |
| 上传归档文件 | [archives/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/archives/index.ts#L27-L41) |
| 导入 HTML 书签 | [importFromHTMLFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L25-L32) |
| 从 Pocket 导入 | [importFromPocket.ts |
| 从 Wallabag 导入 | [importFromWallabag.ts |
| 从 Omnivore 导入 | [importFromOmnivore.ts |
| 从 Linkwarden 导入 | [importFromLinkwarden.ts |
| RSS 自动采集 | [rssHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/lib/rssHandler.ts#L43-L54) |

### 5.2 RSS 订阅数量限制

位置：[rss/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/rss/index.ts#L53-L66)

```typescript
if (rssSubscriptionCount >= RSS_SUBSCRIPTION_LIMIT_PER_USER) {
  return res.status(403).json({
    response: `You have reached the limit of ${RSS_SUBSCRIPTION_LIMIT_PER_USER} RSS subscriptions.`
  });
}
```

### 5.3 上传文件大小限制

位置：[archives/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/archives/index.ts#L44-L65)

- PDF/图片/HTML 文件大小限制：`NEXT_PUBLIC_MAX_FILE_BUFFER` MB

---

## 6. 团队成员（Seat）上限判断

### 6.1 成员邀请创建

位置：[postUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/users/postUser.ts#L84-L132)

邀请用户时，将新用户的 `parentSubscriptionId` 设置为邀请者的订阅 ID。

### 6.2 自动增删成员时的席位检查与 Stripe 同步

#### 成员加入时自动增座

位置：[auth/[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/auth/[...nextauth].ts#L1340-L1363)

```typescript
// 新验证成员数 + 当前用户 + admin > quantity 时
if (verifiedChildUsersCount + 2 > parentSubscription.quantity) {
  await updateSeats(stripeSubscriptionId, verifiedChildUsersCount + 2);
}
```

#### 成员移除时减座

位置：[deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L84-L103)

```typescript
if (removeUser.emailVerified)
  await updateSeats(user.subscriptions.stripeSubscriptionId, user.subscriptions.quantity - 1);
```

### 6.3 updateSeats 函数

位置：[updateSeats.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/updateSeats.ts#L6-L26)

调用 Stripe API 更新订阅数量（试用中：

```typescript
await stripe.subscriptions.update(subscriptionId, {
  billing_cycle_anchor: trialing ? undefined : "now",
  proration_behavior: trialing ? undefined : "create_prorations",
  quantity: seats,
});
```

---

## 7. 前端路由守卫与订阅检查

### 7.1 全局认证重定向

位置：[AuthRedirect.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/layouts/AuthRedirect.tsx#L15-L92)

```typescript
const hasInactiveSubscription =
  user?.id &&
  !user?.subscription?.active &&
  !user.parentSubscription?.active &&
  STRIPE_ENABLED &&
  (REQUIRE_CC || daysLeft <= 0);

if (isLoggedIn && hasInactiveSubscription) {
  redirectTo("/subscribe");
}
```

**重定向逻辑：**

| 条件 | 行为 |
|------|------|
| 已登录 + 订阅失效 + Stripe 启用 + (需绑卡或试用期过 | 跳转 `/subscribe` |
| 已登录 + 未设置用户名 + 是子用户 | 跳转 `/member-onboarding` |
| 未登录 + 访问受保护路由 | 跳转 `/login` |

### 7.2 订阅页面

位置：[subscribe.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/subscribe.tsx#L18-L199)

- 已激活订阅用户访问时自动跳转 dashboard
- 显示试用期剩余天数
- REQUIRE_CC=true 时显示"Start with a X-day free trial
- REQUIRE_CC=false 且试用期已过时显示"Your free trial has ended

### 7.3 功能可见性控制

位置：[SettingsSidebar.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/components/SettingsSidebar.tsx#L126-L141)

```tsx
{process.env.NEXT_PUBLIC_STRIPE && !user?.parentSubscriptionId && (
  // 仅主用户且启用 Stripe 时显示 Billing 菜单
)}
```

子用户（有 parentSubscriptionId 看不到账单设置。

---

## 8. 降级后的已有资源与错误提示

### 8.1 降级判定

| 降级场景 | 触发条件 |
|----------|----------|
| 订阅过期 | `new Date() > subscription.currentPeriodEnd` |
| 订阅被取消 | `customer.subscription.deleted Webhook 事件 → active=false |
| 试用过期 | `daysLeft <= 0 && REQUIRE_CC=false 且无有效订阅 |
| 成员超额 | `childCount + 1 > quantity |

### 8.2 降级后行为

**数据保留**：
- **已有链接（Links）**：数据保留，不会删除，用户仍可查看已有数据
- **已有归档文件**：保留在文件系统中
- **集合、标签、高亮**：全部保留
- **成员数据**：全部保留

**功能限制**：
- **创建新链接**：禁止，返回错误提示
- **上传归档**：禁止
- **数据导入**：禁止
- **RSS 自动采集**：静默跳过（仅日志记录
- **访问 Dashboard**：重定向至订阅页面

### 8.3 错误提示汇总

| 场景 | 错误信息 | HTTP 状态码 |
|------|----------|------------|
| 链接数超限 | `Your subscription has reached the maximum number of links allowed.` | 400 |
| 归档链接数超限（上传前 | `Each collection owner can only have a maximum of ${MAX_LINKS_PER_USER} Links.` | 400 |
| RSS 订阅超限 | `You have reached the limit of ${RSS_SUBSCRIPTION_LIMIT_PER_USER} RSS subscriptions.` | 403 |
| 文件大小超限 | `Sorry, we couldn't process your file. Please ensure it doesn't exceed ${maxMB}MB.` | 400 |
| 成员移除 | `Account removed from subscription.` | 200 |
| 无权限邀请 | `You are not authorized to invite users.` | 401 |
| 演示模式 | `This action is disabled because this is a read-only demo of Linkwarden.` | 400 |
| 注册禁用 | `Registration is disabled.` | 400 |

---

## 9. 支付与计费相关代码索引

| 模块 | 文件路径 |
|------|----------|
| 数据模型 | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/prisma/schema.prisma) |
| 配额检查 | [verifyCapacity.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/lib/verifyCapacity.ts) |
| 订阅验证 | [verifySubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/verifySubscription.ts) |
| Stripe 查询 | [checkSubscriptionByEmail.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/checkSubscriptionByEmail.ts) |
| 订阅处理 | [handleSubscription.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/handleSubscription.ts) |
| Webhook 端点 | [webhook/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/webhook/index.ts) |
| 席位更新 | [updateSeats.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/stripe/updateSeats.ts) |
| 支付结账 | [paymentCheckout.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/paymentCheckout.ts) |
| 创建链接 | [postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/links/postLink.ts) |
| 上传归档 | [archives/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/api/v1/archives/index.ts) |
| RSS 处理 | [rssHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/packages/lib/rssHandler.ts) |
| 前端路由守卫 | [AuthRedirect.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/layouts/AuthRedirect.tsx) |
| 订阅页面 | [subscribe.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/subscribe.tsx) |
| 账单设置 | [billing.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/pages/settings/billing.tsx) |
| 邀请成员 | [InviteModal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/components/ModalContent/InviteModal.tsx) |
| 创建用户 | [postUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/users/postUser.ts) |
| 删除用户 | [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/92-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts) |
