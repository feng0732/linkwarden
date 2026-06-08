# Linkwarden API Token 与外部调用权限模型分析

## 1. Token 体系概览

Linkwarden 采用 **JWT + 数据库持久化** 的双轨 token 模型。Access Token（API 令牌、移动端永久会话）统一存储在 `AccessToken` 表中，使用 `jti`（JWT ID）作为关联键；而浏览器 Web 登录的 JWT Cookie **不写入** AccessToken 表，仅存在于客户端 Cookie 中。

### 1.1 AccessToken 数据模型

定义于 [schema.prisma](./packages/prisma/schema.prisma#L234-L246)：

```prisma
model AccessToken {
  id         Int       @id @default(autoincrement())
  name       String
  user       User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  userId     Int
  token      String    @unique           // 存储 JWT 的 jti (UUID)
  revoked    Boolean   @default(false)    // 软撤销标记
  isSession  Boolean   @default(false)    // true=永久会话(移动端), false=API token
  expires    DateTime                      // 过期时间
  lastUsedAt DateTime?                     // 最后使用时间（当前未写入）
  createdAt  DateTime  @default(now())
  updatedAt  DateTime  @default(now()) @updatedAt
}
```

### 1.2 AccessToken 写入入口

整个代码库中**仅有两处**会向 `AccessToken` 表写入记录：

| 入口 | 文件 | isSession | 场景 |
|------|------|-----------|------|
| `POST /api/v1/tokens` | [postToken.ts](./apps/web/lib/api/controllers/tokens/postToken.ts#L80) | `false` | 用户在设置页主动创建 API 访问令牌 |
| `POST /api/v1/session` | [createSession.ts](./apps/web/lib/api/controllers/session/createSession.ts#L32) | `true` | 移动端/外部客户端通过用户名密码创建永久会话 |

浏览器 Web 登录（NextAuth Credentials / OAuth / Email Magic Link）的 JWT **不会**在 AccessToken 表中创建任何记录。JWT 签发仅在 NextAuth 的 `jwt` callback 中执行（[[...nextauth].ts](./apps/web/pages/api/v1/auth/[...nextauth].ts#L1411-L1488)），将 `user.id` 写入 token payload，但不触发任何 AccessToken 数据库写入。

### 1.3 Token Scope 机制

Linkwarden **没有细粒度的 OAuth scope**，所有 AccessToken 等效于该用户的完整身份。但存在一个特殊的独立 scope：

| Scope | 用途 | 定义位置 | TTL |
|-------|------|----------|-----|
| `preserved-format` | 归档内容（PDF/截图/Monolith/可读格式）的单次访问授权 | [createPreservedFormatUrl.ts](./apps/web/lib/api/preserved/createPreservedFormatUrl.ts#L6-L8) | 300 秒 (5 分钟) |

该 scope 的 token 独立于用户会话，使用 `NEXTAUTH_SECRET` 签名，包含 `linkId`、`filePath`、`format`、`scope`、`iat`、`exp` 字段。

### 1.4 Token 过期选项

由 [TokenExpiry 枚举](./packages/types/global.ts#L175-L181) 定义：

```typescript
enum TokenExpiry {
  sevenDays   = 0,  // 7 天
  oneMonth    = 1,  // 30 天
  twoMonths   = 2,  // 60 天
  threeMonths = 3,  // 90 天
  never       = 4,  // 约 200 年 (73000 天)
}
```

创建时由 [postToken.ts](./apps/web/lib/api/controllers/tokens/postToken.ts#L42-L62) 计算实际过期时间，"永不" 实际上是 200 年。

## 2. 认证入口与用户权限体系

### 2.1 认证入口分类

| 入口 | 路径 | 认证方式 | AccessToken 记录 | 说明 |
|------|------|----------|-----------------|------|
| NextAuth Web 登录 | `/api/v1/auth/[...nextauth]` | JWT Cookie (`next-auth.session-token`) | ❌ 不写入 | 浏览器常规登录，JWT 策略，30 天有效期 |
| 凭据会话创建 | `POST /api/v1/session` | 用户名/密码 → 永久 JWT | ✅ `isSession=true` | 移动端/外部客户端使用，约 200 年过期 |
| API Token 创建 | `POST /api/v1/tokens` | 已登录用户 → JWT | ✅ `isSession=false` | 用户主动创建，命名 token |
| 公共数据访问 | `/api/v1/public/*` | 无认证 | N/A | 仅访问 `isPublic=true` 的 collection/link |
| Stripe Webhook | `POST /api/v1/webhook` | Stripe 签名验证 | N/A | 支付事件回调，`STRIPE_WEBHOOK_SECRET` 验签 |
| 归档内容访问 | `/api/v1/preserved/view` | `preserved-format` JWT | N/A | 5 分钟短期 token，必须从独立用户内容域名访问 |

### 2.2 NextAuth 认证配置

[[...nextauth].ts](./apps/web/pages/api/v1/auth/[...nextauth].ts#L1313-L1513)：

- **Session 策略**: JWT（非数据库会话）
- **Max Age**: 30 天 (`30 * 24 * 60 * 60`)
- **支持的 Provider**: Credentials（用户名密码）、Email Magic Link、以及 60+ 种 OAuth/OpenID Connect 提供商（GitHub、Google、Keycloak、Authelia 等）
- **JWT 回调**: `signIn/signUp` 时将 `user.id` 写入 token；`signUp` 时自动补全 username、emailVerified、dashboardSections
- **signIn 回调**: SSO 用户自动关联已有账号、seat 分配、邮箱状态检查；**不写入 AccessToken 表**

### 2.3 认证校验层级

代码中存在两个层次的认证函数：

**Level 1 - `verifyToken`** ([verifyToken.ts](./apps/web/lib/api/verifyToken.ts#L9-L35))：
1. 通过 `getToken({ req })` 从 Cookie/Authorization 解析 JWT
2. 检查 `userId` (token.id) 是否存在
3. 检查 JWT 是否已过期（`token.exp < Date.now()/1000`）
4. **数据库撤销检查**：以 `token.jti` 查询 `AccessToken` 表，若存在匹配记录且 `revoked=true` 则拒绝
5. 返回 JWT 对象或错误字符串

> **关键点**: 所有 JWT（含浏览器 Web 登录 Cookie）均包含 `jti` 字段（NextAuth JWT 默认生成 jti，代码库在 [next-auth.d.ts](./apps/web/types/next-auth.d.ts#L16-L24) 中将 `jti: string` 声明为 JWT 的必填字段）。浏览器 JWT 无法被撤销的根本原因是：**AccessToken 表中没有对应记录**，而非 JWT 缺少 jti。

**Level 2 - `verifyUser`** ([verifyUser.ts](./apps/web/lib/api/verifyUser.ts#L14-L73))：
在 verifyToken 基础上追加：
1. 数据库查找用户（含 subscriptions、parentSubscription）
2. 检查 username 是否存在
3. 若启用了 `NEXT_PUBLIC_EMAIL_PROVIDER`，检查 email 是否已验证
4. 若启用了 Stripe (`STRIPE_SECRET_KEY`)，调用 `verifySubscription` 检查订阅状态（含试用过期）
5. 返回完整 User 对象或 null

**Level 3 - `isAuthenticatedRequest`** ([isAuthenticatedRequest.ts](./apps/web/lib/api/isAuthenticatedRequest.ts#L11-L49))：
类似 verifyUser，但返回 null 而非写 JSON 响应，用于不需要直接写响应的上下文。

### 2.4 用户权限层级

Linkwarden 没有 RBAC 角色系统，权限基于以下层级：

```
Server Admin (NEXT_PUBLIC_ADMIN=1)
  └── Collection Owner (ownerId === userId)
        └── Collection Member (UsersAndCollections 关联)
              ├── canCreate:  boolean  // 可在该 collection 创建 link
              ├── canUpdate:  boolean  // 可修改 link 内容
              └── canDelete:  boolean  // 可删除 link
```

**Server Admin**: 用户 ID 匹配 `NEXT_PUBLIC_ADMIN` 环境变量（默认 1），可查看/修改/删除任意用户。参考 [users/[id]/index.ts](./apps/web/pages/api/v1/users/%5Bid%5D/index.ts#L31-L69)。

**Sub-collection 权限继承**: 通过 [getCollectionRootOwnerAndMembers](./apps/web/lib/api/getCollectionRootOwnerAndMembers.ts#L18-L74) 沿 `parentId` 向上遍历，将所有祖先 collection 的 owner 和成员权限合并（OR 逻辑：任一祖先有权限即有效）。权限传播由 `propagateToSubcollections` 标志在 collection 更新时触发。

**订阅权限**: 若启用 Stripe，非订阅用户（含试用过期）无法访问任何需要认证的 API。参见 [verifySubscription.ts](./apps/web/lib/api/stripe/verifySubscription.ts#L13-L86)。

## 3. Link 操作范围与权限控制

### 3.1 数据可见性基础

所有 Link 必须归属于一个 Collection，Link 的可见性完全由其所属 Collection 的权限决定。

权限查询统一使用 [getPermission](./apps/web/lib/api/getPermission.ts#L9-L38)：
- 通过 `linkId` 反查所属 Collection，或直接通过 `collectionId` 查询
- 匹配条件：`ownerId === userId` **或** `members` 中存在 `userId`

### 3.2 Link CRUD 权限矩阵

| 操作 | API 端点 | Owner 权限 | Member (canCreate) | Member (canUpdate) | Member (canDelete) | 公开访问 |
|------|----------|-----------|-------------------|-------------------|-------------------|---------|
| 列表/搜索 | `GET /api/v1/links`, `GET /api/v1/search` | ✅ 所有可见 collection 的 link | ✅ 同左 | ✅ 同左 | ✅ 同左 | 仅 `isPublic=true` 的 collection |
| 读取单个 | `GET /api/v1/links/[id]` | ✅ | ✅ | ✅ | ✅ | 仅公开 collection |
| 创建 | `POST /api/v1/links` | ✅ | ✅ | ❌ | ❌ | ❌ |
| 更新 | `PUT /api/v1/links/[id]` | ✅ | ❌ (仅可 pin) | ✅ | ❌ | ❌ |
| 删除 | `DELETE /api/v1/links/[id]` | ✅ | ❌ | ❌ | ✅ | ❌ |
| 批量更新 | `PUT /api/v1/links` (bulk) | ✅ | 逐 link 校验 | 逐 link 校验 | 逐 link 校验 | ❌ |
| 批量删除 | `DELETE /api/v1/links` (bulk) | ✅ | ❌ | ❌ | ✅ 逐 link | ❌ |
| 归档内容访问 | `GET /api/v1/archives/[linkId]`, `/api/v1/preserved/*` | ✅ | ✅ | ✅ | ✅ | 仅公开 collection |

### 3.3 创建 Link 的权限流程

[postLink.ts](./apps/web/lib/api/controllers/links/postLink.ts#L12-L167) → [setCollection.ts](./apps/web/lib/api/setCollection.ts#L11-L100)：

1. **collectionId 模式**: 验证用户为 owner **或** 成员具有 `canCreate=true`
2. **collectionName 模式**: 若为 "Unorganized" 查找/创建默认集合；否则以当前用户为 owner 创建新 collection
3. **默认模式**: 自动使用/创建 "Unorganized" 顶层 collection
4. **容量检查**: `hasPassedLimit` 根据订阅/试用状态校验 link 数量上限（默认 30000/用户或 seat）

### 3.4 更新 Link 的特殊规则

[updateLinkById.ts](./apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L11-L201)：

- **仅 Pin 操作**: 非 owner 但为成员（任意权限）可执行 pin/unpin 到自己的 dashboard
- **跨集合移动**: 非 owner **禁止** 将 link 在不同 collection 间移动
- **目标集合校验**: 新 collection 必须与原 collection 属于同一权限上下文
- **URL 变更**: 若 URL 变化，自动清除旧归档文件并重置归档字段

### 3.5 公开 (Public) 访问路径

无需认证即可访问：

| 端点 | 条件 | 返回内容 |
|------|------|---------|
| `GET /api/v1/public/collections/[id]` | `collection.isPublic === true` | collection 信息 + 成员（脱敏）+ link 计数 |
| `GET /api/v1/public/collections/[id]/links` | 同上 | 该 collection 的 links（含 tags） |
| `GET /api/v1/public/collections/[id]/tags` | 同上 | 该 collection 的 tags |
| `GET /api/v1/public/links/[id]` | 所属 collection 为公开 | link + tags + collection |
| `GET /api/v1/public/users/[id]` | `user.isPrivate === false` | 用户公开资料 |
| `GET /api/v1/public/preserved/[id]` | 所属 collection 为公开 | 归档内容（通过 preserved-format token） |

实现参考 [public/links/linkId/getLinkById.ts](./apps/web/lib/api/controllers/public/links/linkId/getLinkById.ts#L3-L24)。

### 3.6 归档内容访问权限

[resolveAccessibleArchive.ts](./apps/web/lib/api/archives/resolveAccessibleArchive.ts#L30-L45)：

访问条件为三选一（OR）：
1. `ownerId === userId`（所有者）
2. `members.some(userId)`（集合成员）
3. `isPublic === true`（公开集合）

## 4. Token 撤销后的会话影响

### 4.1 撤销机制

撤销操作是**软删除**，仅将 `AccessToken.revoked` 置为 `true`，记录仍保留在数据库中。

**Token 撤销 API**: `DELETE /api/v1/tokens/[id]` → [deleteTokenById.ts](./apps/web/lib/api/controllers/tokens/tokenId/deleteTokenById.ts#L3-L24)

```typescript
await prisma.accessToken.update({
  where: { id: tokenExists?.id },
  data: { revoked: true },
});
```

### 4.2 撤销检查点

每次 API 调用都会**实时查询数据库**检查撤销状态：

- [verifyToken.ts#L24-L33](./apps/web/lib/api/verifyToken.ts#L24-L33): 以 `token.jti` 查询 `AccessToken` 表，仅当存在匹配记录且 `revoked=true` 时返回 "Your session has expired"；若 AccessToken 表中无对应 jti 记录，则查询返回 null，撤销检查通过
- [isAuthenticatedRequest.ts#L24-L33](./apps/web/lib/api/isAuthenticatedRequest.ts#L24-L33): 相同逻辑

### 4.3 影响范围（修正）

| Token 类型 | isSession | JWT 含 jti | 写入 AccessToken | 撤销影响 |
|-----------|-----------|------------|------------------|---------|
| 浏览器 JWT Cookie | N/A | ✅ 包含 jti | ❌ 不写入 | ❌ 完全不受撤销列表影响（设计缺口） |
| API Access Token | `false` | ✅ 包含 jti | ✅ 写入 | ✅ 下次调用立即失效（401） |
| 移动端永久会话 (`POST /api/v1/session`) | `true` | ✅ 包含 jti | ✅ 写入 | ✅ 下次调用立即失效（401） |
| Preserved-Format Token | N/A | ❌ 独立 JWT（无 jti） | ❌ 不写入 | ❌ 独立 JWT，5 分钟自然过期，无撤销机制 |

> **关键修正**: 浏览器常规登录产生的 NextAuth JWT Cookie **不会**在 `AccessToken` 表中创建记录，这是其无法被撤销的根本原因。因此：
> 1. `DELETE /api/v1/tokens/[id]` 无法撤销任何浏览器会话，因为该接口操作的是 AccessToken 表，而浏览器 JWT 在该表中没有对应记录
> 2. `verifyToken` 的撤销查询是 `where: { token: token.jti, revoked: true }`（见 [verifyToken.ts#L24-L29](./apps/web/lib/api/verifyToken.ts#L24-L29)），对浏览器 JWT 而言该查询返回 `null`（无匹配记录），故撤销检查永远通过
> 3. 浏览器会话只能通过 JWT 自身过期（30 天）或客户端 `signOut()` 清除 Cookie 来终止
> 4. 前端登出使用 NextAuth 的 `signOut()` 函数（如 [ProfileDropdown.tsx](./apps/web/components/ProfileDropdown.tsx#L77)），仅清除客户端 Cookie，**不通知后端**删除任何会话记录

### 4.4 Token 列表可见性

`GET /api/v1/tokens` ([getTokens.ts](./apps/web/lib/api/controllers/tokens/getTokens.ts#L3-L22)) 仅返回 `revoked=false` 且属于当前用户的 token，返回字段为 `id, name, isSession, expires, createdAt`，不含实际 token 值（token 值仅在创建时返回一次）。

## 5. Rate Limit 实现代码事实

### 5.1 已实现的限流

整个代码库中存在**三处**显式限流逻辑，均基于数据库计数 + 时间窗口：

#### (1) 邮箱验证邮件限流

位置：[[...nextauth].ts](./apps/web/pages/api/v1/auth/[...nextauth].ts#L142-L154) 的 Email Provider `sendVerificationRequest` 中

```typescript
const recentVerificationRequestsCount = await prisma.verificationToken.count({
  where: {
    identifier,
    createdAt: {
      gt: new Date(new Date().getTime() - 1000 * 60 * 5), // 5 分钟窗口
    },
  },
});

if (recentVerificationRequestsCount >= 4)
  throw Error("Too many requests. Please try again later.");
```

**规则**: 同一邮箱 5 分钟内最多发送 4 封验证邮件，超出即拒绝。对 "email"（登录 magic link）和 "invite"（邀请）两个 provider 独立生效。

#### (2) 忘记密码邮件限流

位置：[forgot-password.ts](./apps/web/pages/api/v1/auth/forgot-password.ts#L29-L43)

```typescript
const recentPasswordRequestsCount = await prisma.passwordResetToken.count({
  where: {
    identifier: email,
    createdAt: {
      gt: new Date(new Date().getTime() - 1000 * 60 * 5), // 5 minutes
    },
  },
});

if (recentPasswordRequestsCount >= 3) {
  return res.status(400).json({
    response: "Too many requests. Please try again later.",
  });
}
```

**规则**: 同一邮箱 5 分钟内最多请求 3 次密码重置邮件。

#### (3) 重置密码 Token 消费限流（隐式）

位置：[reset-password.ts](./apps/web/pages/api/v1/auth/reset-password.ts#L71-L78)

```typescript
// Delete tokens older than 5 minutes
await prisma.passwordResetToken.deleteMany({
  where: {
    identifier: email,
    createdAt: {
      lt: new Date(new Date().getTime() - 1000 * 60 * 5), // 5 minutes
    },
  },
});
```

**规则**: 重置密码成功后，清理该邮箱 5 分钟以前的所有 passwordResetToken。这不是限流，但客观上限制了同一时间窗口内有效 token 的数量上限。重置密码 token 自身有效期为 24 小时（见 [sendPasswordResetRequest.ts](./apps/web/lib/api/sendPasswordResetRequest.ts#L14-L20)）。

### 5.2 Rate Limit 汇总表

| 维度 | 是否有 Rate Limit | 限流规则 | 实现位置 |
|------|------------------|---------|---------|
| 邮箱验证/邀请邮件发送 | ✅ 有 | 5 分钟 ≤ 4 封/邮箱 | `[...nextauth].ts` L142-154 |
| 忘记密码邮件发送 | ✅ 有 | 5 分钟 ≤ 3 次/邮箱 | `forgot-password.ts` L29-43 |
| 密码重置 token 消费 | ⚠️ 隐式 | 成功后清理 5 分钟前的旧 token | `reset-password.ts` L71-78 |
| 登录失败 (brute force) | ❌ 无 | — | — |
| API 调用频率 (per user) | ❌ 无 | — | — |
| 邮箱变更请求 | ❌ 无 | — | — |
| Link 创建频率 | ❌ 无 | 仅总量限制 | — |
| 注册请求 (`POST /api/v1/users`) | ❌ 无 | — | — |
| 公共 API 匿名访问 | ❌ 无 | — | — |
| 搜索 API | ❌ 无 | — | — |

### 5.3 邮箱变更（无限流）

[updateUserById.ts](./apps/web/lib/api/controllers/users/userId/updateUserById.ts#L104-L135) 中调用 `sendChangeEmailVerificationRequest` 发送邮箱变更验证邮件，但**无任何频率限制**，可被无限调用。

### 5.4 相关容量限制（非 Rate Limit）

虽无时间窗口限流，但存在以下硬限制：

- **单用户 link 上限**: `MAX_LINKS_PER_USER` 环境变量，默认 30000。见 [verifyCapacity.ts](./packages/lib/verifyCapacity.ts#L3-L109)
- **分页大小**: `PAGINATION_TAKE_COUNT` 环境变量，默认 50 条/页
- **文件上传**: `NEXT_PUBLIC_MAX_FILE_BUFFER` 环境变量，默认 10MB
- **Token 名称长度**: 最多 50 字符（schema 验证）
- **头像大小**: 1.5MB（硬编码于 [updateUserById.ts](./apps/web/lib/api/controllers/users/userId/updateUserById.ts#L88-L93)）

## 6. 审计记录代码事实

### 6.1 审计记录现状

Linkwarden **不存在**专门的审计日志 / 操作日志表。数据库 schema 中无 AuditLog、ActivityLog 等表。

### 6.2 隐式审计字段

各核心模型仅包含基础的时间戳字段（非审计）：

| 模型 | 时间字段 | 操作人字段 |
|------|---------|-----------|
| User | `createdAt`, `updatedAt` | 无 |
| Collection | `createdAt`, `updatedAt` | `createdById` (创建人) |
| Link | `createdAt`, `updatedAt`, `lastPreserved`, `importDate` | `createdById` (创建人) |
| AccessToken | `createdAt`, `updatedAt`, `lastUsedAt?` | 无 (通过 userId 关联) |
| Highlight | `createdAt`, `updatedAt` | `userId` |
| Tag | `createdAt`, `updatedAt` | 无 (通过 ownerId 关联) |

### 6.3 AccessToken.lastUsedAt 状态

`AccessToken` 模型定义了 `lastUsedAt` 字段（`DateTime?`），但**在整个代码库中没有任何写入操作**。全局搜索 `accessToken.*lastUsedAt` 和 `lastUsedAt.*accessToken` 均无结果。该字段仅用于预留，当前始终为 NULL，无法用于审计 token 使用情况。

### 6.4 日志输出

代码中仅存在少量 `console.log` / `console.error` 输出：

| 位置 | 输出内容 |
|------|---------|
| [[...nextauth].ts](./apps/web/pages/api/v1/auth/[...nextauth].ts#L91) | `"User log in attempt..."` (凭据登录尝试) |
| [verify-email.ts](./apps/web/pages/api/v1/auth/verify-email.ts#L74) | `console.log(emailInUse)` (邮箱变更时的调试日志，泄露邮箱信息) |
| [verifySubscription.ts](./apps/web/lib/api/stripe/verifySubscription.ts#L82) | Stripe subscription upsert 错误 |
| [webhook/index.ts](./apps/web/pages/api/v1/webhook/index.ts#L57) | Stripe webhook 签名错误 |
| [webhook/index.ts](./apps/web/pages/api/v1/webhook/index.ts#L107) | Webhook 事件处理错误 |
| [preserved/view.ts](./apps/web/pages/api/v1/preserved/view.ts#L158) | 归档内容加载失败 |
| [preserved/token.ts](./apps/web/pages/api/v1/preserved/token.ts#L60) | 归档 URL 创建失败 |
| [updateUserById.ts](./apps/web/lib/api/controllers/users/userId/updateUserById.ts#L86) | 头像保存错误 |
| [updateUserById.ts](./apps/web/lib/api/controllers/users/userId/updateUserById.ts#L89) | 头像文件过大提示 |

这些日志均输出到 stdout/stderr，无结构化日志、无持久化存储、无日志轮转。

### 6.5 登录事件

NextAuth 提供 `signIn` callback（[[...nextauth].ts](./apps/web/pages/api/v1/auth/[...nextauth].ts#L1326-L1410)），但仅用于 SSO 用户关联、seat 分配和邮箱验证检查，未记录任何登录日志到数据库。

## 7. 安全缺口总结（基于代码事实）

1. **JWT Cookie 无撤销能力**: 浏览器登录态 JWT 不持久化到 AccessToken 表（AccessToken 表中没有对应记录），无法通过撤销接口强制登出。后端无任何使浏览器会话失效的机制。
2. **登录无失败防护**: Credentials 认证无暴力破解防护（无失败计数、无 IP 锁定、无延时）
3. **邮箱变更无限流**: `sendChangeEmailVerificationRequest` 可被无限调用
4. **无 API Rate Limiting**: 除邮件发送外，所有 API 端点无用户级或 IP 级频率限制
5. **无操作审计日志**: 所有数据变更（link/collection/token/user）无持久化操作记录
6. **Token 使用不追踪**: `lastUsedAt` 字段存在但从未写入，无法审计 token 使用
7. **Preserved-Format Token 无撤销**: 5 分钟窗口内一旦签发无法撤回
8. **调试日志泄露**: [verify-email.ts](./apps/web/pages/api/v1/auth/verify-email.ts#L74) 的 `console.log(emailInUse)` 会在生产日志中输出用户邮箱信息
