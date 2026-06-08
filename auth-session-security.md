# Linkwarden Auth/Session 与安全边界代码分析

## 一、整体架构概览

Linkwarden 采用 **Next.js + NextAuth.js (next-auth v4)** 作为认证框架，数据库使用 PostgreSQL + Prisma ORM。整体认证体系分为多个层次，核心文件分布如下：

| 层次 | 关键文件 |
|------|---------|
| NextAuth 主配置 | [[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/pages/api/v1/auth/[...nextauth].ts) |
| Token 校验层 | [verifyToken.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/verifyToken.ts) |
| 用户校验层 | [verifyUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/verifyUser.ts) |
| 请求认证层 | [isAuthenticatedRequest.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/isAuthenticatedRequest.ts) |
| 凭据校验 | [verifyByCredentials.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/verifyByCredentials.ts) |
| 数据模型 | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/packages/prisma/schema.prisma) |

---

## 二、登录会话机制 (Session & Auth)

### 2.1 认证策略

NextAuth 配置采用 **JWT 策略**（非数据库 session），有效期 30 天：

```typescript
// [[...nextauth].ts#L1316-L1319]
session: {
  strategy: "jwt",
  maxAge: 30 * 24 * 60 * 60, // 30 days
},
```

### 2.2 支持的认证 Provider

1. **Credentials Provider**（用户名/密码）：默认启用，可通过 `NEXT_PUBLIC_CREDENTIALS_ENABLED=false` 禁用
2. **Email Provider**（无密码邮件登录）：需配置 `EMAIL_FROM` 和 `EMAIL_SERVER`，包含两个子 Provider：
   - `email`：普通邮箱登录
   - `invite`：邀请链接登录
3. **50+ OAuth Provider**：Google、GitHub、Apple、Keycloak、Auth0、Azure AD 等，通过环境变量开关控制

### 2.3 认证流程与关键回调

在 [[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/pages/api/v1/auth/[...nextauth].ts#L1325-L1511) 中定义了三个核心回调：

#### `signIn` 回调（登录时触发）
- 校验用户邮箱验证状态
- 处理 SSO 新用户禁用逻辑（`DISABLE_NEW_SSO_USERS`）
- SSO 用户自动链接已有账号（同 email）
- 订阅用户座位数自动同步到 Stripe

#### `jwt` 回调（JWT 生成/刷新时触发）
- 将用户 ID 注入 token（`token.id = user.id`）
- `signUp` 触发时：自动验证 SSO 用户邮箱、生成默认用户名、初始化 dashboard sections
- `signIn` 触发时：补全缺失的用户名

#### `session` 回调（session 读取时触发）
- 将 JWT 中的 `id` 注入 session.user
- 每次读取 session 时校验订阅状态（Stripe 模式下）

### 2.4 移动端独立会话

移动端不使用 NextAuth cookie 机制，而是通过独立 API 创建长期会话 token：

- 入口 API：[session/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/pages/api/v1/session/index.ts)
- 创建逻辑：[createSession.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/controllers/session/createSession.ts)
- 有效期：200 年（模拟永久），标记 `isSession: true`
- 移动端存储：使用 `expo-secure-store` 加密存储 token 和 instance URL
- 移动端认证请求：通过 `Authorization: Bearer <token>` 头发送

---

## 三、API Token 机制与区别

系统中存在 **6 种不同用途的 token**，边界清晰：

### 3.1 Token 类型对比表

| Token 类型 | 存储位置 | 有效期 | 签发方式 | 用途 | isSession |
|-----------|---------|--------|---------|------|-----------|
| **NextAuth JWT (Cookie)** | HttpOnly Cookie | 30 天 | NextAuth 签发 | Web 端浏览器会话 | N/A |
| **Mobile Session Token** | `AccessToken` 表 | 200 年 | `encode()` 手动签发 | 移动端长期会话 | `true` |
| **API Access Token** | `AccessToken` 表 | 7/30/60/90 天或永不过期 | `encode()` 手动签发 | 第三方 API 集成 | `false` |
| **Preserved Format Token** | URL 参数（无状态） | 5 分钟 | `encode()` 手动签发 | 归档文件短期访问授权 | N/A |
| **PasswordResetToken** | `PasswordResetToken` 表 | 24 小时 | `randomBytes(32)` 十六进制 | 密码重置 | N/A |
| **VerificationToken** | `VerificationToken` 表 | 20 分钟 (1200s) | NextAuth 生成 | 邮箱验证 / 邀请确认 | N/A |

### 3.2 API Access Token 生命周期

创建入口：[tokens/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/pages/api/v1/tokens/index.ts) → [postToken.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/controllers/tokens/postToken.ts)

创建流程：
1. 校验 token 名称唯一性（同用户下未撤销的 token 不能重名）
2. 根据过期策略计算 `expiryDate` 和 `expiryDateSecond`
3. 使用 `next-auth/jwt` 的 `encode()` 签发 JWT，载荷含 `{ id, iat, exp, jti }`
4. 存储 JTI（而非完整 token）到 `AccessToken` 表，关联 `userId`
5. 仅在创建时返回完整 JWT 给用户（此后不可再获取明文）

撤销入口：[tokens/[id].ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/pages/api/v1/tokens/[id].ts) → [deleteTokenById.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/controllers/tokens/tokenId/deleteTokenById.ts)

撤销方式：**软删除**，将 `revoked` 字段设为 `true`（而非物理删除），便于审计。

### 3.3 Preserved Format Token（归档文件访问）

用于保护用户归档内容（快照、PDF 等），实现文件在独立域名上的安全短期访问：

- 创建：[createPreservedFormatUrl.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/preserved/createPreservedFormatUrl.ts)
- 校验：[preserved/view.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/pages/api/v1/preserved/view.ts)
- 核心安全设计：
  - 必须通过 `NEXT_PUBLIC_USER_CONTENT_DOMAIN` 配置的独立域名访问
  - 宿主头校验：`getRequestHost()` 验证请求 Host 匹配配置域名，防止 Host Header 攻击
  - 短 TTL：300 秒（5 分钟）
  - Scope 校验：`scope === "preserved-format"`
  - 文件路径后缀与 format 匹配校验，防止路径穿越

---

## 四、账号设置与敏感操作校验

### 4.1 敏感操作的密码二次校验

在 [updateUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserById.ts) 和 [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts) 中，敏感操作强制要求密码验证：

| 操作 | 密码校验方式 | 代码位置 |
|-----|-------------|---------|
| **修改邮箱** | `bcrypt.compareSync(data.password, user.password)` | [updateUserById.ts#L104-L134](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserById.ts#L104-L134) |
| **修改密码** | 校验旧密码 + 新密码 ≥8 字符 + 新旧不可相同 | [updateUserById.ts#L139-L166](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserById.ts#L139-L166) |
| **删除账号** | 必须验证当前密码（SSO 用户需先设密码） | [deleteUserById.ts#L39-L60](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L39-L60) |
| **管理员删除子用户** | 无需密码，但需校验父子订阅关系 | [deleteUserById.ts#L61-L104](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L61-L104) |

**注意**：OAuth/SSO 登录用户默认无密码，执行上述操作前必须通过"忘记密码"流程设置密码，否则直接拒绝。

### 4.2 邮箱变更流程

修改邮箱不立即生效，而是走验证流程：
1. 验证当前密码
2. 调用 `sendChangeEmailVerificationRequest()` 发送验证邮件到**新邮箱**
3. 用户点击链接后才完成邮箱变更

### 4.3 速率限制

| 操作 | 限制 | 窗口 | 代码位置 |
|-----|------|------|---------|
| 邮箱验证请求 | ≤ 4 次 | 5 分钟 | [[...nextauth].ts#L143-L154](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/pages/api/v1/auth/[...nextauth].ts#L143-L154) |
| 邀请邮件请求 | ≤ 4 次 | 5 分钟 | [[...nextauth].ts#L192-L203](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/pages/api/v1/auth/[...nextauth].ts#L192-L203) |
| 密码重置请求 | ≤ 3 次 | 5 分钟 | [forgot-password.ts#L29-L43](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/pages/api/v1/auth/forgot-password.ts#L29-L43) |

### 4.4 请求认证分层

系统实现了三层认证校验，各 API 按需选择：

| 层级 | 函数 | 检查内容 | 返回 |
|-----|------|---------|------|
| 1 | `verifyToken()` | JWT 有效性、过期时间、是否被撤销 | JWT 对象或错误字符串 |
| 2 | `verifyUser()` | 基于 verifyToken + 用户存在性 + 用户名 + 邮箱验证 + 订阅状态 | User 对象或 null（同时 res 写入 401/404） |
| 3 | `isAuthenticatedRequest()` | 基于 getToken + token 过期 + 撤销检查 + 订阅状态 | User 对象或 null |

**撤销检查机制**：两个校验层都会查询 `AccessToken` 表中 `token === token.jti && revoked === true` 的记录，实现了 JWT 的**服务端可撤销**能力（解决 JWT 天然无法撤销的问题）。

---

## 五、账号删除后的访问收敛

### 5.1 数据库级级联删除

Prisma schema 中几乎所有关联模型都定义了 `onDelete: Cascade`，保证用户删除时关联数据被清理：

```prisma
// 示例：User 关联
Account          @relation(..., onDelete: Cascade)
Collection       @relation(..., onDelete: Cascade)
Tag              @relation(..., onDelete: Cascade)
AccessToken      @relation(..., onDelete: Cascade)
...
```

涉及模型：`Account`、`Collection`、`Tag`、`Link`、`Highlight`、`UsersAndCollections`、`AccessToken`、`Subscription`、`RssSubscription`、`DashboardSection`、`WhitelistedUser` 等。

### 5.2 应用级事务删除

在 [deleteUserById.ts#L107-L205](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L107-L205) 中，使用 Prisma `$transaction`（20秒超时）执行以下清理：

1. **搜索索引清理**：从 Meilisearch 删除所有用户链接文档
2. **文件系统清理**：
   - 删除所有收藏夹的归档目录（`archives/{collectionId}` 和 `archives/preview/{collectionId}`）
   - 删除用户头像文件（`uploads/avatar/{userId}.jpg`）
3. **订阅清理**（Stripe 模式下）：
   - 取消主订阅或减少子用户座位数
   - 可选项发送取消原因邮件
4. **用户记录删除**：最后 `prisma.user.delete()`

### 5.3 认证收敛（潜在缺口）

用户被删除后，其已签发的 JWT 在过期前**天然仍有效**，但系统通过以下机制收敛：

1. **verifyUser() 中的用户存在性检查**：`prisma.user.findUnique({ id: userId })` 返回 null 时返回 404
2. **级联删除 AccessToken**：用户删除时其 `AccessToken` 记录被级联删除，但 JWT 的 `jti` 校验逻辑是查 `revoked === true`，此时记录已不存在，**撤销检查不生效**

**潜在安全缺口分析**：
- 用户删除后，若攻击者持有未过期的 JWT，`verifyToken()` 层不会拦截（因为 `revoked` 检查查不到记录即视为未撤销）
- 但 `verifyUser()` 层和 `isAuthenticatedRequest()` 层会因为查不到用户而拦截
- **结论**：使用 `verifyToken()` 且未进一步查用户的端点存在风险。当前 `verifyToken()` 仅用于：`users/me`、`users/[id]`、`preserved/token`，这些端点后续都会查用户，风险被覆盖。但新增 API 时需注意，不能仅依赖 `verifyToken()`。

---

## 六、CSRF 防护与 MFA 扩展点

### 6.1 CSRF 防护现状

**NextAuth.js 内置防护**：
- NextAuth 默认使用 **Double Submit Cookie** 模式防护 CSRF
- 对 `POST /api/auth/*` 的所有请求自动校验 `csrfToken`
- 通过 `next-auth/react` 的 `signIn()`、`signOut()` 自动携带 CSRF token

**应用层缺口**：
- [next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/next.config.js) 中未配置额外的安全 headers（如 CSP、X-Frame-Options 等）
- 业务 API（非 NextAuth 路由）未显式校验 CSRF token，依赖 SameSite Cookie
- 移动端使用 Bearer Token 认证，天然免疫 CSRF

**当前防护依赖链**：
1. 浏览器端使用 NextAuth Cookie（SameSite 默认 `lax`）
2. 跨域请求受浏览器 SameSite 策略限制
3. 关键写操作需要密码二次校验（见第四节），降低 CSRF 风险

### 6.2 MFA 扩展点

当前代码**尚未实现 MFA（多因素认证）**，但 NextAuth.js 架构提供了清晰的扩展点：

#### 扩展点 1：`signIn` 回调
在 [[...nextauth].ts#L1326-L1410](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/pages/api/v1/auth/[...nextauth].ts#L1326-L1410) 中，可插入 MFA 挑战逻辑：

```
signIn({ user, account, credentials })
  → 检查用户是否启用 MFA
  → 若启用，返回 false 并将 MFA 挑战信息存入临时存储
  → 前端跳转到 MFA 验证页
  → 验证通过后完成登录
```

#### 扩展点 2：JWT + Session 回调
在 `jwt` 回调中注入 MFA 认证状态：
```
jwt({ token, user, trigger })
  → 若 MFA 未完成，在 token 中标记 `mfaPending: true`
  → `session` 回调中传递给前端
  → 前端路由守卫（如 AuthRedirect）拦截未完成 MFA 的请求
```

#### 扩展点 3：敏感操作二次校验
现有敏感操作的密码校验可扩展为 MFA 校验：
- [updateUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserById.ts) 修改邮箱/密码处
- [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts) 删除账号处

### 6.3 其他安全边界

#### SSRF 防护（Worker 进程）
[protectPageRequests.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/worker/lib/protectPageRequests.ts) 使用 Playwright route 拦截，配合 `assertUrlIsSafeForServerSideFetch()` 检查，防止归档抓取时的 SSRF 攻击。

#### 集合级权限（行级安全）
- [getPermission.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/getPermission.ts)：后端校验用户对集合/链接的所有权或成员身份
- [resolveAccessibleArchive.ts](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/lib/api/archives/resolveAccessibleArchive.ts)：归档访问额外检查 `isPublic`，支持公开集合匿名访问
- [usePermissions.tsx](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/hooks/usePermissions.tsx)：前端 UI 级别权限控制（仅做显示控制，不替代后端校验）

#### 前端路由守卫
[AuthRedirect.tsx](file:///d:/fz/0601/solo-dogfeeding/code/97-linkwarden/apps/web/layouts/AuthRedirect.tsx) 实现前端路由级保护：
- 未登录访问受保护路由 → 重定向 `/login`
- 已登录访问公开路由 → 重定向 `/dashboard`
- 订阅失效 → 重定向 `/subscribe`
- 被邀请用户未完善信息 → 重定向 `/member-onboarding`

---

## 七、代码责任边界总结

| 模块 | 责任 | 不负责 |
|------|------|--------|
| **NextAuth 配置** | 认证流程、JWT 签发、SSO 集成、回调钩子 | API 级权限、数据行级权限 |
| **verifyToken()** | JWT 格式、过期、撤销状态 | 用户是否存在、订阅状态、业务权限 |
| **verifyUser()** | 用户存在性、邮箱验证、订阅有效性、返回响应 | 业务操作的密码二次校验 |
| **isAuthenticatedRequest()** | 轻量认证检查（供 SSR/中间件使用） | 发送响应、深度用户检查 |
| **verifyByCredentials()** | 用户名密码比对 | 会话管理、Token 签发 |
| **deleteUserById()** | 用户数据清理、订阅处理、文件清理 | 会话 JWT 撤销（依赖级联删除+上层校验） |
| **updateUserById()** | 账号设置变更、敏感操作密码校验 | Token 撤销、会话管理 |
| **getPermission()** | 集合/链接的归属和成员权限 | 用户认证、订阅校验 |

## 八、潜在改进点

1. **JWT 撤销一致性**：用户删除时，建议将该用户所有 AccessToken 先批量标记 `revoked=true` 再级联删除，确保 `verifyToken()` 层也能拦截
2. **CSRF 显式校验**：对业务 API 的写操作（POST/PUT/DELETE）显式校验 CSRF token，不依赖 SameSite
3. **安全 Headers**：在 next.config 中配置 CSP、HSTS、X-Frame-Options 等安全响应头
4. **MFA 预留数据模型**：在 User 模型或独立表中预留 MFA 密钥、备份码、启用状态字段
5. **登录失败限流**：当前凭据登录无显式失败计数和锁定机制，存在暴力破解风险
