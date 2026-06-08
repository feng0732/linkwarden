# Linkwarden Auth/Session 与安全边界代码分析

## 一、整体架构概览

Linkwarden 采用 **Next.js + NextAuth.js (next-auth v4)** 作为认证框架，数据库使用 PostgreSQL + Prisma ORM。整体认证体系分为多个层次，核心文件（仓库相对路径）分布如下：

| 层次 | 关键文件 |
|------|---------|
| NextAuth 主配置 | `apps/web/pages/api/v1/auth/[...nextauth].ts` |
| Token 校验层 | `apps/web/lib/api/verifyToken.ts` |
| 用户校验层 | `apps/web/lib/api/verifyUser.ts` |
| 请求认证层 | `apps/web/lib/api/isAuthenticatedRequest.ts` |
| 凭据校验 | `apps/web/lib/api/verifyByCredentials.ts` |
| 数据模型 | `packages/prisma/schema.prisma` |

---

## 二、登录会话机制 (Session & Auth)

### 2.1 认证策略

NextAuth 配置采用 **JWT 策略**（非数据库 session），有效期 30 天：

```typescript
// apps/web/pages/api/v1/auth/[...nextauth].ts
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

在 `apps/web/pages/api/v1/auth/[...nextauth].ts` 中定义了三个核心回调：

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

**源码证据**（`apps/web/pages/api/v1/auth/[...nextauth].ts#L1489-L1510`）：
```typescript
async session({ session, token }) {
  session.user.id = token.id;

  if (STRIPE_SECRET_KEY) {               // ← 仅 Stripe 模式下才会查库
    const user = await prisma.user.findUnique({
      where: { id: token.id },
      include: { subscriptions: true, parentSubscription: true },
    });
    if (user) {                          // ← 用户不存在时静默跳过，不抛错、不返回 null
      const subscribedUser = await verifySubscription(user);
    }
  }

  return session;                         // ← 无论用户是否存在，都正常返回 session
}
```

- 将 JWT 中的 `id` 注入 `session.user`
- **仅在 Stripe 模式下**查库校验订阅状态；非 Stripe 模式下不做任何 DB 查询
- **关键**：即使用户已删除（findUnique 返回 null），回调也不会拒绝 session、不会清空 session、不会抛错，始终 `return session`

### 2.4 移动端独立会话

移动端不使用 NextAuth cookie 机制，而是通过独立 API 创建长期会话 token：

- 入口 API：`apps/web/pages/api/v1/session/index.ts`
- 创建逻辑：`apps/web/lib/api/controllers/session/createSession.ts`
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

创建入口：`apps/web/pages/api/v1/tokens/index.ts` → `apps/web/lib/api/controllers/tokens/postToken.ts`

创建流程：
1. 校验 token 名称唯一性（同用户下未撤销的 token 不能重名）
2. 根据过期策略计算 `expiryDate` 和 `expiryDateSecond`
3. 使用 `next-auth/jwt` 的 `encode()` 签发 JWT，载荷含 `{ id, iat, exp, jti }`
4. 存储 JTI（而非完整 token）到 `AccessToken` 表，关联 `userId`
5. 仅在创建时返回完整 JWT 给用户（此后不可再获取明文）

撤销入口：`apps/web/pages/api/v1/tokens/[id].ts` → `apps/web/lib/api/controllers/tokens/tokenId/deleteTokenById.ts`

撤销方式：**软删除**，将 `revoked` 字段设为 `true`（而非物理删除），便于审计。

### 3.3 Preserved Format Token（归档文件访问）

用于保护用户归档内容（快照、PDF 等），实现文件在独立域名上的安全短期访问：

- 创建：`apps/web/lib/api/preserved/createPreservedFormatUrl.ts`
- 校验：`apps/web/pages/api/v1/preserved/view.ts`
- 核心安全设计：
  - 必须通过 `NEXT_PUBLIC_USER_CONTENT_DOMAIN` 配置的独立域名访问
  - 宿主头校验：`getRequestHost()` 验证请求 Host 匹配配置域名，防止 Host Header 攻击
  - 短 TTL：300 秒（5 分钟）
  - Scope 校验：`scope === "preserved-format"`
  - 文件路径后缀与 format 匹配校验，防止路径穿越

---

## 四、账号设置与敏感操作校验

### 4.1 敏感操作的密码二次校验

在 `apps/web/lib/api/controllers/users/userId/updateUserById.ts` 和 `apps/web/lib/api/controllers/users/userId/deleteUserById.ts` 中，敏感操作强制要求密码验证：

| 操作 | 密码校验方式 | 代码位置 |
|-----|-------------|---------|
| **修改邮箱** | `bcrypt.compareSync(data.password, user.password)` | `apps/web/lib/api/controllers/users/userId/updateUserById.ts#L104-L134` |
| **修改密码** | 校验旧密码 + 新密码 ≥8 字符 + 新旧不可相同 | `apps/web/lib/api/controllers/users/userId/updateUserById.ts#L139-L166` |
| **删除账号** | 必须验证当前密码（SSO 用户需先设密码） | `apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L39-L60` |
| **管理员删除子用户** | 无需密码，但需校验父子订阅关系 | `apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L61-L104` |

**注意**：OAuth/SSO 登录用户默认无密码，执行上述操作前必须通过"忘记密码"流程设置密码，否则直接拒绝。

### 4.2 邮箱变更流程

修改邮箱不立即生效，而是走验证流程：
1. 验证当前密码
2. 调用 `sendChangeEmailVerificationRequest()` 发送验证邮件到**新邮箱**
3. 用户点击链接后才完成邮箱变更

### 4.3 速率限制

| 操作 | 限制 | 窗口 | 代码位置 |
|-----|------|------|---------|
| 邮箱验证请求 | ≤ 4 次 | 5 分钟 | `apps/web/pages/api/v1/auth/[...nextauth].ts#L143-L154` |
| 邀请邮件请求 | ≤ 4 次 | 5 分钟 | `apps/web/pages/api/v1/auth/[...nextauth].ts#L192-L203` |
| 密码重置请求 | ≤ 3 次 | 5 分钟 | `apps/web/pages/api/v1/auth/forgot-password.ts#L29-L43` |

### 4.4 请求认证分层

系统实现了三层认证校验，各 API 按需选择：

| 层级 | 函数 | 检查内容 | 返回 |
|-----|------|---------|------|
| 1 | `verifyToken()` | JWT 格式、过期时间、是否被撤销（查 `AccessToken.revoked===true`） | JWT 对象或错误字符串 |
| 2 | `verifyUser()` | 基于 verifyToken + 用户存在性 + 用户名 + 邮箱验证 + 订阅状态 | User 对象或 null（同时 res 写入 401/404） |
| 3 | `isAuthenticatedRequest()` | 基于 getToken + token 过期 + 撤销检查 + 订阅状态 + 用户存在性 | User 对象或 null |

**撤销检查机制**：两个校验层都会查询 `AccessToken` 表中 `token === token.jti && revoked === true` 的记录，实现了 JWT 的**服务端可撤销**能力（解决 JWT 天然无法撤销的问题）。

---

## 五、所有 API 入口认证方式全量核对

逐一核对 `apps/web/pages/api/v1/` 下全部路由，按认证方式分类：

### 5.1 使用 verifyUser() 的路由（强校验，用户删除后必拦截）

| 路由 | 方法 | 后续 DB 操作 | 用户删除后表现 |
|------|------|-------------|---------------|
| `/v1/tokens` | GET/POST | 查/写 AccessToken | verifyUser 阶段 404 拦截 |
| `/v1/tokens/[id]` | DELETE | 软删 AccessToken | verifyUser 阶段 404 拦截 |
| `/v1/tags` | GET/POST | 查/写 Tag | verifyUser 阶段 404 拦截 |
| `/v1/tags/[id]` | GET/PUT/DELETE | 查/写 Tag | verifyUser 阶段 404 拦截 |
| `/v1/tags/merge` | POST | 合并 Tag | verifyUser 阶段 404 拦截 |
| `/v1/users` | POST | 创建用户 | verifyUser 阶段 404 拦截 |
| `/v1/users/[id]/preference` | GET/PUT | 写用户偏好 | verifyUser 阶段 404 拦截 |
| `/v1/collections` | GET/POST | 查/写 Collection | verifyUser 阶段 404 拦截 |
| `/v1/collections/[id]` | GET/PUT/DELETE | 查/写 Collection | verifyUser 阶段 404 拦截 |
| `/v1/links` | GET/POST | 查/写 Link | verifyUser 阶段 404 拦截 |
| `/v1/links/[id]` | GET/PUT/DELETE | 查/写 Link | verifyUser 阶段 404 拦截 |
| `/v1/links/[id]/archive` | POST | 写归档文件 | verifyUser 阶段 404 拦截 |
| `/v1/links/[id]/highlights` | GET/POST | 查/写 Highlight | verifyUser 阶段 404 拦截 |
| `/v1/links/archive` | POST | 上传归档 | verifyUser 阶段 404 拦截 |
| `/v1/highlights` | POST | 写 Highlight | verifyUser 阶段 404 拦截 |
| `/v1/highlights/[id]` | PUT/DELETE | 改/删 Highlight | verifyUser 阶段 404 拦截 |
| `/v1/dashboard` | GET | 查 Dashboard | verifyUser 阶段 404 拦截 |
| `/v2/dashboard` | GET | 查 Dashboard V2 | verifyUser 阶段 404 拦截 |
| `/v1/search` | POST | 搜索 Link | verifyUser 阶段 404 拦截 |
| `/v1/rss` | GET/POST | 查/写 RSS | verifyUser 阶段 404 拦截 |
| `/v1/rss/[id]` | GET/DELETE | 查/删 RSS | verifyUser 阶段 404 拦截 |
| `/v1/migration` | POST | 导入导出 | verifyUser 阶段 404 拦截 |
| `/v1/worker` | GET | 查 Worker 状态 | verifyUser 阶段 404 拦截 |
| `/v1/worker/preservation` | POST | 触发归档 | verifyUser 阶段 404 拦截 |
| `/v1/archives` | POST | 上传归档 | verifyUser 阶段 404 拦截 |
| `/v1/archives/[linkId]` | POST | 更新归档文件 | verifyUser 阶段 404 拦截 |

### 5.2 只使用 verifyToken() / getToken() 的路由（需进一步核对）

| 路由 | 方法 | 认证函数 | verifyToken 后的 DB 查询 | 用户删除后实际拦截点 |
|------|------|---------|-------------------------|---------------------|
| `/v1/users/me` | GET | `verifyToken()` | `getUserById(userId)` 查 User 表 | getUserById 返回 null → 404 |
| `/v1/users/[id]` | GET/PUT/DELETE | `verifyToken()` | 查 User、查权限做操作 | User 不存在 → 404/401 |
| `/v1/preserved/token` | GET | `verifyToken()` | `resolveAccessibleArchive()` 查 Collection 权限 | Collection 已级联删除 → 401 |
| `/v1/avatar/[id]` | GET | `verifyToken()`（可选） | `prisma.user.findUnique({ id: queryId })` 查**目标用户** | 目标用户不存在 → 400 "File inaccessible." |
| `/v1/archives/[linkId]` | GET | `verifyToken()`（可选） | `resolveAccessibleArchive()` 查 Collection 权限 | Collection 已级联删除 → 401 |
| `/v1/payment` | GET | `getToken()` (next-auth 原生) | `prisma.user.findUnique({ id: token.id })` 查 User 表 + email | User 不存在 → 404 "User not found." |

**结论**：以上 6 条"只做 token 校验"的路由，在 verifyToken 之后**全部都有进一步的数据库查询**，查询的实体在用户删除后均已不存在或不可访问，因此**实际均能被有效拦截**，不存在裸奔端点。

### 5.3 完全公开无认证的路由（任何人可访问）

| 路由 | 方法 | 查询逻辑 | 用户删除后表现 |
|------|------|---------|---------------|
| `/v1/auth/*` | 多方法 | NextAuth 内置流程 | 不受影响 |
| `/v1/logins` | GET | 读取环境变量返回登录方式配置 | 不受影响 |
| `/v1/config` | GET | 读取环境变量返回实例配置 | 不受影响 |
| `/v1/getFavicon` | GET | 代理外部 favicon 服务 | 不受影响 |
| `/v1/webhook` | POST | Stripe 签名校验 + 处理订阅事件 | 不受影响 |
| `/v1/session` | POST | `verifyByCredentials()` 用户名密码登录 | 不受影响（登录入口本身） |
| `/v1/public/collections/[id]` | GET | `prisma.collection.findFirst({ id, isPublic: true })` | Collection 已级联删除 → 400 "Collection not found." |
| `/v1/public/collections/links` | GET | `searchLinks({ publicOnly: true })` | Collection 已级联删除 → 空结果 |
| `/v1/public/collections/tags` | GET | 先查 `collection.isPublic === true`，再查 tags | Collection 已级联删除 → 404 "Collection not found." |
| `/v1/public/links/[id]` | GET | `prisma.link.findFirst({ id, collection: { isPublic: true } })` | Link 已级联删除 → 返回 null（200 但 body 为 null） |
| `/v1/public/users/[id]` | GET | `prisma.user.findFirst({ id/username/email })`，返回脱敏字段 | User 不存在 → 404 "User not found." |
| `/v1/preserved/view` | GET | `decodePreservedFormatToken()` 独立短期 token | Token 5 分钟自失效，与用户存在性无关 |

---

## 六、账号删除后的访问收敛（详细边界分析与代码证据）

### 6.1 数据库级级联删除

Prisma schema 中几乎所有关联模型都定义了 `onDelete: Cascade`，保证用户删除时关联数据被清理：

```prisma
// packages/prisma/schema.prisma
Account          @relation(..., onDelete: Cascade)
Collection       @relation(..., onDelete: Cascade)
Tag              @relation(..., onDelete: Cascade)
AccessToken      @relation(..., onDelete: Cascade)
Subscription     @relation(..., onDelete: Cascade)
...
```

级联删除涉及的全部模型：`Account`、`Collection`、`Tag`、`Link`、`Highlight`、`UsersAndCollections`、`AccessToken`、`Subscription`、`RssSubscription`、`DashboardSection`、`WhitelistedUser` 等。

### 6.2 应用级事务删除

`apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L107-L205` 中使用 Prisma `$transaction`（20秒超时）执行以下清理：

1. **搜索索引清理**：从 Meilisearch 删除所有用户链接文档
2. **文件系统清理**：
   - 删除所有收藏夹的归档目录（`archives/{collectionId}` 和 `archives/preview/{collectionId}`）
   - 删除用户头像文件（`uploads/avatar/{userId}.jpg`）
3. **订阅清理**（Stripe 模式下）：
   - 取消主订阅或减少子用户座位数
   - 可选项发送取消原因邮件
4. **用户记录删除**：最后 `prisma.user.delete()`

---

### 6.3 归档读取（GET /v1/archives/[linkId]）拦截边界

**代码证据**：`apps/web/pages/api/v1/archives/[linkId].ts#L85-L128`

**调用链源码证据**：
```typescript
// apps/web/pages/api/v1/archives/[linkId].ts
async function handleGet(req: NextApiRequest, res: NextApiResponse) {
  const linkId = Number(req.query.linkId);
  const format = Number(req.query.format);
  const isPreview = Boolean(req.query.preview);

  // Verify token => If present, get user ID
  const token = await verifyToken({ req });          // 第一步：只做 token 校验
  const userId = typeof token === "string" ? undefined : token?.id;

  const resolvedArchive = await resolveAccessibleArchive({  // 第二步：查 DB 权限
    linkId, format, isPreview, userId,
  });
  // ...
}
```

`resolveAccessibleArchive` 内部查询（见 `apps/web/lib/api/archives/resolveAccessibleArchive.ts`）：
```typescript
prisma.collection.findFirst({
  where: {
    links: { some: { id: linkId } },
    OR: [
      { ownerId: userId || -1 },        // 私有：所有者匹配
      { members: { some: { userId } } }, // 私有：成员匹配
      { isPublic: true }                 // 公开集合
    ]
  }
});
```

**用户删除后的实际拦截**：
| 场景 | 拦截机制 | HTTP 状态 |
|------|---------|----------|
| 读取自己的私有归档 | Collection 已级联删除 → findFirst 返回 null | 401 "You don't have access to this collection." |
| 读取公开集合归档 | Collection 已级联删除 → findFirst 返回 null | 401 "You don't have access to this collection." |
| 作为成员读取他人归档 | 成员关系 UsersAndCollections 已级联删除，若集合属他人且公开则可访问 | 取决于集合所有者是否删除 |

**文件层兜底**：磁盘上的归档文件在 deleteUserById 事务中被 `removeFolder()` 物理删除，即使数据库绕过，文件本身也已不存在。

---

### 6.4 头像读取（GET /v1/avatar/[id]）拦截边界

**代码证据**：`apps/web/pages/api/v1/avatar/[id].ts#L6-L40`

**调用链源码证据**：
```typescript
// apps/web/pages/api/v1/avatar/[id].ts
export default async function Index(req: NextApiRequest, res: NextApiResponse) {
  const queryId = Number(req.query.id);

  if (!queryId) return res.status(401).send("Invalid parameters.");

  // verifyToken 仅用于"可选地"获取请求者 userId——注意该 userId 实际上未被使用
  const token = await verifyToken({ req });
  const userId = typeof token === "string" ? undefined : token?.id;

  if (req.method === "GET") {
    // 实际只查 URL 路径里的目标用户 queryId
    const targetUser = await prisma.user.findUnique({
      where: { id: queryId },
    });

    if (!targetUser) {                       // ← 关键拦截点
      return res.status(400).send("File inaccessible.");
    }

    const { file, contentType, status } = await readFile(
      `uploads/avatar/${queryId}.jpg`
    );
    return res.setHeader("Content-Type", contentType).status(status as number).send(file);
  }
}
```

**用户删除后的实际拦截**：
| 场景 | 拦截机制 | HTTP 状态 |
|------|---------|----------|
| 读取已删除用户的头像 | targetUser = prisma.user.findUnique 返回 null → 命中 `if (!targetUser)` | 400 "File inaccessible." |
| 未登录用户读取存在用户头像 | targetUser 存在 → 直接返回头像文件（设计行为） | 200 |
| 已登录用户读取他人头像 | targetUser 存在 → 直接返回头像文件（设计行为） | 200 |

**文件层兜底**：磁盘头像文件在 deleteUserById 事务中被 `removeFile()` 物理删除，双重保险。

**安全结论**：头像 API **不需要认证即可读取任意存在用户的头像**（用于公开页面展示头像的设计行为）。用户删除后通过 targetUser 存在性检查 + 文件删除被收敛。

---

### 6.5 公开归档/公开数据访问拦截边界

#### 6.5.1 公开集合元数据（GET /v1/public/collections/[id]）

**代码证据**：`apps/web/lib/api/controllers/public/collections/getPublicCollection.ts#L3-L32`

**源码证据**：
```typescript
// apps/web/lib/api/controllers/public/collections/getPublicCollection.ts
export default async function getPublicCollection(id: number) {
  const collection = await prisma.collection.findFirst({
    where: {
      id,
      isPublic: true,    // ← 公开过滤 + 记录存在性
    },
    include: { members: { include: { user: { select: { ... } } } }, _count: { select: { links: true } } }
  });
  if (collection) return { response: collection, status: 200 };
  else return { response: "Collection not found.", status: 400 };
}
```

**用户删除后**：Collection 被 `onDelete: Cascade` 删除 → findFirst 返回 null → **400** "Collection not found."

---

#### 6.5.2 公开集合链接列表（GET /v1/public/collections/links）

**代码证据**：`apps/web/pages/api/v1/public/collections/links/index.ts#L5-L37`

调用 `searchLinks({ publicOnly: true })`，内部查询条件含 `collection.isPublic === true`。用户删除后 Collection 和 Link 均被级联删除 → 返回空结果集。

---

#### 6.5.3 公开链接详情（GET /v1/public/links/[id]）

**代码证据**：`apps/web/lib/api/controllers/public/links/linkId/getLinkById.ts#L3-L24`

**源码证据**：
```typescript
// apps/web/lib/api/controllers/public/links/linkId/getLinkById.ts
export default async function getLinkById(linkId: number) {
  const link = await prisma.link.findFirst({
    where: {
      id: linkId,
      collection: { isPublic: true },   // ← 集合公开过滤
    },
    include: { tags: true, collection: true },
  });
  return { response: link, status: 200 };   // ← 注意：link 为 null 时仍返回 200
}
```

**用户删除后**：Link 被级联删除 → 返回 `null`，HTTP **200**（非 404），body 为 `{"response": null}`。

**小缺陷**：返回 200 + null 语义不严谨，但不构成安全问题。

---

#### 6.5.4 公开用户信息（GET /v1/public/users/[id]）

**代码证据**：`apps/web/lib/api/controllers/public/users/getPublicUser.ts#L3-L39`

**源码证据**：
```typescript
// apps/web/lib/api/controllers/public/users/getPublicUser.ts
export default async function getPublicUser(targetId: number | string, isId: boolean) {
  const user = await prisma.user.findFirst({
    where: isId ? { id: Number(targetId) }
                : { OR: [{ username: targetId }, { email: targetId }] },
  });
  if (!user || !user.id) return { response: "User not found.", status: 404 };

  const { password, ...lessSensitiveInfo } = user;   // ← 剥离密码字段
  const data = { id, name, username, image, archiveAsScreenshot, archiveAsMonolith, archiveAsPDF };
  return { response: data, status: 200 };
}
```

**用户删除后**：findFirst 返回 null → **404** "User not found."。

**注意**：此接口**不检查 User.isPrivate 字段**，只要用户存在即返回脱敏信息（与账号删除收敛无关，属隐私设计项）。

---

### 6.6 长期 Token（API Token / Mobile Session）删除后的拦截边界

**代码证据**：
- Token 撤销软删除逻辑：`apps/web/lib/api/controllers/tokens/tokenId/deleteTokenById.ts`
- verifyToken 撤销检查逻辑：`apps/web/lib/api/verifyToken.ts#L23-L33`

**verifyToken 撤销检查源码证据**：
```typescript
// apps/web/lib/api/verifyToken.ts
if (token.jti) {
  const revoked = await prisma.accessToken.findFirst({
    where: {
      token: token.jti,   // ← JTI 对应 AccessToken.token 字段
      revoked: true,      // ← 只匹配 revoked===true 的软删除记录
    },
  });
  if (revoked) return "Your session has expired. Please login again.";
}
```

**拦截效果差异矩阵**：
| 操作 | revoked 字段 | DB 记录状态 | verifyToken 拦截效果 | 兜底拦截 |
|------|-------------|------------|---------------------|---------|
| 用户主动撤销 token（`DELETE /v1/tokens/[id]`） | `true` | 保留（软删除） | ✅ 命中 `revoked === true` → 拦截 | 无需 |
| 用户删除自己账号 | N/A | 级联物理删除 | ❌ 记录不存在 → findFirst 返回 null → 不命中 | ✅ 后续 DB 查询 User/Collection 等均为空 |
| 管理员删除子用户 | N/A | 级联物理删除 | ❌ 记录不存在 → findFirst 返回 null → 不命中 | ✅ 后续 DB 查询 User/Collection 等均为空 |

**根因**：撤销检查只匹配存在且 `revoked===true` 的记录。级联物理删除后记录不存在，findFirst 返回 null 被视为"未撤销"。

**实际风险结论**：当前所有使用 verifyToken 的路由之后**均有进一步 DB 查询**（见 5.2 节），查询实体在用户删除后均已级联删除，请求在后续步骤仍会被拦截。仅存在**理论风险**：若未来新增一个仅调用 verifyToken() 就直接返回纯静态信息的端点，则该端点在用户删除后仍可被持有有效 JWT 的攻击者访问——当前代码库中不存在此类端点。

---

### 6.7 浏览器会话（NextAuth Cookie JWT）删除后的真实拦截路径

账号删除后，浏览器端用户的 JWT Cookie 仍然有效（30 天过期前天然无法从服务端直接失效）。需要逐层对照代码澄清每一层是否主动拦截：

#### 6.7.1 NextAuth session 回调：不主动拦截

**代码证据**：`apps/web/pages/api/v1/auth/[...nextauth].ts#L1489-L1510`

如 2.3 节源码所示，`session` 回调的行为：
- 非 Stripe 模式：不查数据库，只做 `session.user.id = token.id` 后直接 `return session`
- Stripe 模式：虽然调用了 `prisma.user.findUnique`，但 `if (user)` 条件未命中时**静默跳过**，不抛错、不清空 session、不返回 null
- **结论**：session 回调无论用户是否存在，都会返回一个合法的 session 对象。useSession() 客户端状态始终判定为 `authenticated`

#### 6.7.2 SSR 本地化查询（getServerSideProps）：不主动拦截

**代码证据**：`apps/web/lib/client/getServerSideProps.ts#L7-L55`

```typescript
const getServerSideProps: GetServerSideProps = async (ctx) => {
  const token = await getToken({ req: ctx.req });
  if (token) {
    const user = await prisma.user.findUnique({ where: { id: token.id } });
    if (user) {
      return { props: { ...serverSideTranslations(user.locale ?? "en", ["common"]) } };
    }
    // ← 用户不存在时：没有 return、没有 redirect、没有 destroy cookie，只是静默 fallthrough
  }
  // 回退到 accept-language 计算 locale，正常渲染页面
  return { props: { ...serverSideTranslations(bestMatch ?? "en", ["common"]) } };
};
```

- SSR 层虽然查了用户，但查不到时**既不 302 重定向也不清除 cookie**，只是 fallback 到浏览器 `Accept-Language` 决定页面语言
- **结论**：SSR 层不会主动把已删除用户踢回登录页，页面正常渲染（只是语言不跟随用户偏好）

#### 6.7.3 前端会话状态（useSession + AuthRedirect）：不主动拦截

**代码证据**：`apps/web/layouts/AuthRedirect.tsx#L15-L91` + `apps/web/hooks/useInitialData.tsx`

```typescript
// AuthRedirect.tsx
const { status } = useSession();           // ← status 只反映 JWT 是否有效
const { data: user } = useUser();           // ← 单独 fetch /api/v1/users/{id}

useEffect(() => {
  const isLoggedIn = status === "authenticated";   // ← 只看 JWT，不看 useUser 结果
  const isUnauthenticated = status === "unauthenticated";

  // 订阅失效判断依赖 user，但 user 为 undefined（fetch 失败）时 hasInactiveSubscription 为 false
  const hasInactiveSubscription = user?.id && !user?.subscription?.active && ...;

  if (isLoggedIn && hasInactiveSubscription) redirectTo("/subscribe");
  else if (isLoggedIn && !user?.name && user?.parentSubscriptionId) redirectTo("/member-onboarding");
  else if (isLoggedIn && !isProtected(router.pathname)) redirectTo("/dashboard");
  else if (isUnauthenticated && isProtected(router.pathname)) redirectTo("/login");
  else setShouldRenderChildren(true);   // ← 用户删除时走这条分支，页面继续渲染
}, [status, user, router.pathname]);
```

- `useSession().status === "authenticated"` 仅由 JWT 有效性决定，不感知用户删除
- `useUser()` hook 调用 `/api/v1/users/{id}` 时，虽然接口会返回 404，但 AuthRedirect 的判断逻辑里**没有监听 `useUser().error` 或 `!user?.id` 来触发登出/重定向**
- **结论**：用户删除后前端不会自动 signOut，`isLoggedIn` 仍为 true，路由守卫会放行，受保护路由页面会正常挂载

#### 6.7.4 用户数据接口 / 业务 API：真正的拦截发生在这里

虽然 session 层、SSR 层、前端路由层都不主动拦截，但页面一旦挂载后会立即发起业务数据请求，这些请求会在后端被拦截：

**① 用户数据接口 `/api/v1/users/me` / `/api/v1/users/[id]`**

代码证据：`apps/web/pages/api/v1/users/me.ts#L1-L19` + `apps/web/lib/api/controllers/users/userId/getUserById.ts#L5-L54`

```typescript
// me.ts
const token = await verifyToken({ req });   // JWT 有效 → 通过
const userId = token.id;
const users = await getUserById(userId);    // ← prisma.user.findUnique 返回 null
return res.status(404).json({ response: "User not found." });

// 前端 useUser hook 收到 404：
// packages/router/user.tsx#L41
if (!response.ok) throw new Error("Failed to fetch user data.");
// react-query 进入 error 状态，但 AuthRedirect 未监听此 error
```

**② 所有 verifyUser() 的业务 API（28 条，见 5.1 节）**

`verifyUser()` 在 `apps/web/lib/api/verifyUser.ts` 中会调用 `prisma.user.findUnique({ id: userId })`，查不到时返回 404 并写入响应。前端页面渲染后发起的 `/v1/links`、`/v1/collections` 等请求全部 404/401。

**③ 所有 verifyToken() + 后续 DB 查询的 API（6 条，见 5.2 节）**

`verifyToken()` 通过但后续 `getUserById` / `resolveAccessibleArchive` / `prisma.user.findUnique` 等查询因数据级联删除而返回空 → 404/401。

#### 6.7.5 前端可见效果

用户删除后，持有未过期 JWT Cookie 的访问者会看到：
1. 浏览器被放行进入受保护路由（AuthRedirect 不拦截）
2. 页面白屏或报错（因为 useUser / 业务接口全部 404，前端组件依赖 user/collection 等数据渲染）
3. 若用户手动调用 signOut() 清除 JWT，一切恢复正常

---

### 6.8 认证收敛总结矩阵

| 攻击入口 | 用户删除后是否被拦截 | 拦截发生在哪一层 |
|---------|---------------------|-----------------|
| verifyUser() 路由（28 条） | ✅ 是 | verifyUser 查 User → 404 |
| verifyToken() 路由（6 条） | ✅ 是 | 后续查 User/Collection → 404/401 |
| 公开路由（/v1/public/*） | ✅ 是 | 级联删除后查不到实体 → 400/404/空结果 |
| 头像 /v1/avatar/[id] | ✅ 是 | 查 targetUser 不存在 → 400 + 文件已删除 |
| 归档 GET /v1/archives/[linkId] | ✅ 是 | Collection 级联删除 → 401 + 文件已删除 |
| Preserved Format Token | ⚠️ 不主动拦截 | Token 5 分钟自失效，与用户存在性无关 |
| NextAuth session 回调 | ❌ 不拦截 | 始终 return session（非 Stripe 模式不查库；Stripe 模式查不到也静默跳过） |
| SSR getServerSideProps | ❌ 不拦截 | 用户不存在时静默 fallback 语言，不重定向、不清 cookie |
| 前端 useSession + AuthRedirect | ❌ 不拦截 | 仅看 JWT 有效性；未监听 useUser 错误触发登出 |
| NextAuth Cookie JWT（整体） | ✅ 是（被动收敛） | 页面挂载后的**用户数据接口 / 业务 API 查询失败**，前端无可用数据 |
| 移动端 Bearer Token | ✅ 是（被动收敛） | 所有 API 调用后续均查 DB 实体 → 404/401 |

### 6.9 最终结论复查

账号删除后的访问收敛由**四层被动机制**叠加完成，**没有任何一层在认证入口主动拒绝会话**：

1. **数据库级**：Prisma `onDelete: Cascade` 级联删除所有关联实体（User/Collection/Link/AccessToken 等）
2. **文件系统级**：`deleteUserById` 事务中 `removeFolder` / `removeFile` 物理删除归档文件和头像
3. **API 数据层**：所有业务 API 在 verifyToken 通过后，都会进一步查询业务实体 → 因级联删除而返回空/404/401
4. **前端被动失效**：页面渲染后所有业务请求失败，呈现白屏/错误状态（但不会自动登出）

**核心校准点**：之前"session 回调查不到用户"、"useSession 服务端拦截"的描述不准确。真实情况是：session 回调、SSR、前端路由守卫**均不主动拦截已删除用户**，所有拦截都发生在"更下游"的用户数据查询和业务数据查询阶段。

**风险结论不变**：除 Preserved Format Token（5 分钟自失效）外，不存在实际可利用的持久化数据访问缺口。但存在**用户体验缺口**：已删除用户的浏览器会话不会自动失效，需手动 signOut 或等待 JWT 30 天自然过期。

---

## 七、CSRF 防护与 MFA 扩展点

### 7.1 CSRF 防护现状

**NextAuth.js 内置防护**：
- NextAuth 默认使用 **Double Submit Cookie** 模式防护 CSRF
- 对 `POST /api/auth/*` 的所有请求自动校验 `csrfToken`
- 通过 `next-auth/react` 的 `signIn()`、`signOut()` 自动携带 CSRF token

**应用层缺口**：
- `apps/web/next.config.js` 中未配置额外的安全 headers（如 CSP、X-Frame-Options 等）
- 业务 API（非 NextAuth 路由）未显式校验 CSRF token，依赖 SameSite Cookie
- 移动端使用 Bearer Token 认证，天然免疫 CSRF

**当前防护依赖链**：
1. 浏览器端使用 NextAuth Cookie（SameSite 默认 `lax`）
2. 跨域请求受浏览器 SameSite 策略限制
3. 关键写操作需要密码二次校验（见第四节），降低 CSRF 风险

### 7.2 MFA 扩展点

当前代码**尚未实现 MFA（多因素认证）**，但 NextAuth.js 架构提供了清晰的扩展点：

#### 扩展点 1：`signIn` 回调
在 `apps/web/pages/api/v1/auth/[...nextauth].ts` 的 `signIn` 回调中插入 MFA 挑战逻辑：
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
- `apps/web/lib/api/controllers/users/userId/updateUserById.ts` 修改邮箱/密码处
- `apps/web/lib/api/controllers/users/userId/deleteUserById.ts` 删除账号处

### 7.3 其他安全边界

#### SSRF 防护（Worker 进程）
`apps/worker/lib/protectPageRequests.ts` 使用 Playwright route 拦截，配合 `assertUrlIsSafeForServerSideFetch()` 检查，防止归档抓取时的 SSRF 攻击。

#### 集合级权限（行级安全）
- `apps/web/lib/api/getPermission.ts`：后端校验用户对集合/链接的所有权或成员身份
- `apps/web/lib/api/archives/resolveAccessibleArchive.ts`：归档访问额外检查 `isPublic`，支持公开集合匿名访问
- `apps/web/hooks/usePermissions.tsx`：前端 UI 级别权限控制（仅做显示控制，不替代后端校验）

#### 前端路由守卫
`apps/web/layouts/AuthRedirect.tsx` 实现前端路由级保护：
- 未登录访问受保护路由 → 重定向 `/login`
- 已登录访问公开路由 → 重定向 `/dashboard`
- 订阅失效 → 重定向 `/subscribe`
- 被邀请用户未完善信息 → 重定向 `/member-onboarding`

---

## 八、代码责任边界总结

| 模块 | 责任 | 不负责 |
|------|------|--------|
| **NextAuth 配置** | 认证流程、JWT 签发、SSO 集成、回调钩子 | API 级权限、数据行级权限 |
| **verifyToken()** | JWT 格式、过期、撤销状态（AccessToken.revoked） | 用户是否存在、订阅状态、业务权限 |
| **verifyUser()** | 用户存在性、邮箱验证、订阅有效性、返回响应 | 业务操作的密码二次校验 |
| **isAuthenticatedRequest()** | 轻量认证检查（供 SSR/中间件使用） | 发送响应、深度用户检查 |
| **verifyByCredentials()** | 用户名密码比对 | 会话管理、Token 签发 |
| **deleteUserById()** | 用户数据清理、订阅处理、文件清理、级联删除触发 | 会话 JWT 主动撤销（依赖级联删除+上层校验兜底） |
| **updateUserById()** | 账号设置变更、敏感操作密码校验 | Token 撤销、会话管理 |
| **getPermission()** | 集合/链接的归属和成员权限 | 用户认证、订阅校验 |
| **resolveAccessibleArchive()** | 归档访问的集合级权限（所有者/成员/公开） | 用户认证、Token 校验 |
| **public/* 控制器** | 公开数据读取 + isPublic 过滤 | 用户认证、写操作 |

---

## 九、潜在改进点

1. **用户删除时先批量标记 AccessToken.revoked=true**：在 deleteUserById 事务中，级联删除 AccessToken 之前先批量 `updateMany({ revoked: true })`，确保 verifyToken 层也能主动拦截（当前靠后续 DB 查询兜底）。
2. **NextAuth session 回调增加用户存在性校验**：在 session 回调中 `prisma.user.findUnique` 返回 null 时，主动返回一个不含 user.id 的 session 或触发 signOut，让前端 `useSession().status` 能感知用户删除。
3. **SSR getServerSideProps 增加用户不存在时的重定向**：查不到用户时 `return { redirect: { destination: "/login", permanent: false } }`，从服务端主动踢回登录页。
4. **前端 AuthRedirect 监听 useUser 错误**：`useUser()` 返回 error 或 `!user?.id` 时调用 `signOut()` 清除 JWT，实现自动登出。
5. **CSRF 显式校验**：对业务 API 的写操作（POST/PUT/DELETE）显式校验 CSRF token，不依赖 SameSite。
6. **安全 Headers**：在 next.config 中配置 CSP、HSTS、X-Frame-Options 等安全响应头。
7. **MFA 预留数据模型**：在 User 模型或独立表中预留 MFA 密钥、备份码、启用状态字段。
8. **登录失败限流**：当前凭据登录无显式失败计数和锁定机制，存在暴力破解风险。
9. **GET /v1/public/links/[id] 返回语义**：Link 不存在时应返回 404 而非 200 + null。
10. **getPublicUser 增加 isPrivate 过滤**：公开用户信息接口应检查 User.isPrivate，尊重用户隐私设置。
11. **Preserved Format Token 关联用户校验**：对于非公开集合的归档访问，短期 token 解码后应二次校验用户对该 collection 的权限（当前只校验 token 自身有效性和文件后缀）。
