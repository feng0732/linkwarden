# Linkwarden SSO / 社交登录代码链路梳理

本文档从代码实现角度梳理 Linkwarden 项目中 SSO（单点登录）与社交登录（Social Login）的完整实现链路，涵盖 **Provider 配置**、**回调处理**、**用户会话建立** 三大核心阶段。

项目基于 [NextAuth.js (v4)](https://next-auth.js.org/) 实现认证，使用 **JWT Session 策略**（非数据库 Session 策略），并通过 Prisma ORM 持久化 Account / User / AccessToken 等数据。

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [Provider 配置](#2-provider-配置)
   - 2.1 [环境变量驱动](#21-环境变量驱动)
   - 2.2 [后端 Provider 数组构建](#22-后端-provider-数组构建)
   - 2.3 [前端登录页按钮动态渲染](#23-前端登录页按钮动态渲染)
   - 2.4 [PrismaAdapter 与 linkAccount 定制](#24-prismaadapter-与-linkaccount-定制)
3. [回调处理（Callbacks）](#3-回调处理callbacks)
   - 3.1 [signIn 回调：准入控制 + 自动账号关联](#31-signin-回调准入控制--自动账号关联)
   - 3.2 [jwt 回调：JWT 载荷构建 + 新用户初始化](#32-jwt-回调jwt-载荷构建--新用户初始化)
   - 3.3 [session 回调：会话对象组装 + 订阅校验](#33-session-回调会话对象组装--订阅校验)
4. [用户会话建立](#4-用户会话建立)
   - 4.1 [SessionProvider 注入（前端）](#41-sessionprovider-注入前端)
   - 4.2 [JWT 的签发与 Cookie 存储](#42-jwt-的签发与-cookie-存储)
   - 4.3 [请求鉴权：verifyToken / verifyUser / isAuthenticatedRequest](#43-请求鉴权verifytoken--verifyuser--isauthenticatedrequest)
   - 4.4 [AccessToken 体系：可撤销的 API Token](#44-accesstoken-体系可撤销的-api-token)
   - 4.5 [前端路由守卫：AuthRedirect](#45-前端路由守卫authredirect)
5. [数据模型](#5-数据模型)
6. [完整登录时序图](#6-完整登录时序图)

---

## 1. 整体架构概览

```
┌──────────────────────────────────────────────────────────────────────┐
│                           前端 (Next.js Pages)                        │
│                                                                      │
│  login.tsx ──signIn(providerId)──►  NextAuth Client (next-auth/react)│
│        ▲                                              │              │
│        │           SessionProvider                    ▼              │
│  AuthRedirect  ──────────────────────►  /api/v1/auth/[...nextauth]   │
│     (路由守卫)                          (NextAuth Route Handler)     │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         后端 (NextAuth + Prisma)                      │
│                                                                      │
│  providers[]  ──►  OAuth Flow  ──►  callbacks.signIn                 │
│  (40+ Provider)                      │                               │
│                                      ▼                               │
│                              callbacks.jwt  ──►  encode(JWT)  ──► Cookie │
│                                      │                               │
│                                      ▼                               │
│                              callbacks.session                        │
│                                      │                               │
│                                      ▼                               │
│                         Prisma (User / Account / AccessToken)         │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. Provider 配置

### 2.1 环境变量驱动

所有 SSO/社交登录 Provider 均通过环境变量启用与配置。核心开关变量遵循 `NEXT_PUBLIC_<PROVIDER>_ENABLED` 命名约定，值为 `"true"` 时启用。

**核心环境变量（节选）：**

| 变量 | 作用 |
|------|------|
| `NEXTAUTH_URL` | NextAuth 回调基础 URL，如 `http://localhost:3000/api/v1/auth` |
| `NEXTAUTH_SECRET` | JWT 签名与加密密钥 |
| `NEXT_PUBLIC_CREDENTIALS_ENABLED` | 是否启用用户名/密码登录（默认启用） |
| `NEXT_PUBLIC_EMAIL_PROVIDER` | 是否启用 Email Magic Link 登录 |
| `DISABLE_NEW_SSO_USERS` | 禁止 SSO 自动注册新用户（仅允许已存在账号登录） |
| `NEXT_PUBLIC_GOOGLE_ENABLED` / `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Google 登录配置示例 |

完整环境变量列表见 [.env.sample](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/.env.sample#L106-L476)。

### 2.2 后端 Provider 数组构建

核心文件：[apps/web/pages/api/v1/auth/[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts)

该文件导出一个默认异步函数，内部调用 `NextAuth(req, res, options)`。Provider 列表 `providers: Provider[]` 在模块顶层根据环境变量**动态构建**：

```typescript
const providers: Provider[] = [];

// 1) 账号密码 Provider（默认启用）
if (process.env.NEXT_PUBLIC_CREDENTIALS_ENABLED !== "false") {
  providers.push(CredentialsProvider({
    type: "credentials",
    credentials: {},
    async authorize(credentials, req) {
      // 通过 prisma.user.findFirst 查找用户，bcrypt.compareSync 校验密码
      // 返回 { id: user.id } 或抛错
    }
  }));
}

// 2) Email (Magic Link) Provider + Invite 专用 Provider
if (emailEnabled) {
  providers.push(EmailProvider({ id: "email", ... }));
  providers.push(EmailProvider({ id: "invite", ... }));
}

// 3) 40+ OAuth/OIDC Provider，每个都用相同模式：
if (process.env.NEXT_PUBLIC_GOOGLE_ENABLED === "true") {
  providers.push(GoogleProvider({
    clientId: process.env.GOOGLE_CLIENT_ID!,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    httpOptions: { timeout: 10000 },
  }));
  // 同时 hook adapter.linkAccount，过滤 Prisma schema 不含的字段
}
```

**支持的 Provider 清单（共 40+）：**
42 School, Apple, Atlassian, Auth0, Authelia, Authentik, Azure AD, Azure AD B2C, Battle.net, Box, Cognito, Coinbase, Discord, Dropbox, Duende IdentityServer6, EVE Online, Facebook, FACEIT, Foursquare, Freshbooks, FusionAuth, GitHub, GitLab, Google, HubSpot, IdentityServer4, Kakao, Keycloak, LINE, LinkedIn, Mailchimp, Mail.ru, Naver, Netlify, Okta, OneLogin, Osso, osu!, Patreon, Pinterest, Pipedrive, Reddit, Salesforce, Slack, Spotify, Strava, Synology, Todoist, Twitch, United Effects, VK, Wikimedia, WordPress.com, Yandex, Zitadel, Zoho, Zoom。

其中 **Authelia** 和 **Synology** 未使用 `next-auth/providers/*` 预设，而是以裸 `type: "oauth"` 对象手写配置（含 `wellKnown`、`pkce`、`state` 等）。

### 2.3 前端登录页按钮动态渲染

核心文件：
- 后端 API：[apps/web/pages/api/v1/logins/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/logins/index.ts)
- 前端页面：[apps/web/pages/login.tsx](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/login.tsx)

后端 `getLogins()` 函数根据环境变量返回当前启用的登录方式：

```typescript
export function getLogins() {
  const buttonAuths = [];
  if (process.env.NEXT_PUBLIC_GOOGLE_ENABLED === "true") {
    buttonAuths.push({
      method: "google",
      name: process.env.GOOGLE_CUSTOM_NAME ?? "Google",
    });
  }
  // ... 40+ 个同样模式的 if 判断
  return {
    credentialsEnabled: "...",
    emailEnabled: "...",
    registrationDisabled: "...",
    buttonAuths,
  };
}
```

注意：`getLogins()` 的判断逻辑与 `[...nextauth].ts` 中 Provider 的注册逻辑**是重复且分离的**——修改启用变量时需两处同步。

前端 `login.tsx` 在 `getServerSideProps` 中调用 `getLogins()`（直接 import 函数，非 HTTP 请求），将 `availableLogins` 注入页面 props。`displayLoginExternalButton()` 遍历 `buttonAuths` 渲染按钮：

```tsx
<Button onClick={() => loginUserButton(value.method)}>
  {value.name}
</Button>

function loginUserButton(method: string) {
  signIn(method, {}); // 调用 next-auth/react 的 signIn，走重定向
}
```

### 2.4 PrismaAdapter 与 linkAccount 定制

核心位置：[apps/web/pages/api/v1/auth/[...nextauth].ts#L78-L79](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L78-L79)

```typescript
const adapter = PrismaAdapter(prisma);
```

每个 OAuth Provider 启用时都会**猴子补丁（Monkey Patch）** `adapter.linkAccount`：

```typescript
const _linkAccount = adapter.linkAccount;
adapter.linkAccount = (account) => {
  const { "not-before-policy": _, refresh_expires_in, ...data } = account;
  return _linkAccount ? _linkAccount(data) : undefined;
};
```

**作用**：不同 OAuth Provider 返回的 Token 响应包含 Prisma `Account` 模型中未定义的字段（如 `not-before-policy`、`refresh_expires_in`、Azure AD 的 `profile_info` 等）。若不剥离，Prisma `create` 会因未知字段报错。这是大量重复代码的根源——40+ Provider 每启用一个就 patch 一次。

---

## 3. 回调处理（Callbacks）

NextAuth 的三个核心回调在 `[...nextauth].ts` 的 `callbacks` 字段中定义，执行顺序为：**signIn → jwt → session**。

### 3.1 signIn 回调：准入控制 + 自动账号关联

位置：[apps/web/pages/api/v1/auth/[...nextauth].ts#L1326-L1410](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1326-L1410)

```typescript
async signIn({ user, account, profile, email, credentials }) {
  // 子步骤 A：未验证邮箱的子用户，触发 Stripe 席位扩容
  if (!(user as User).emailVerified && !email?.verificationRequest) {
    // 若 parentSubscriptionId 存在，计算席位并调用 updateSeats()
  }

  // 子步骤 B：SSO 新用户禁用策略（仅对非 credentials 生效）
  if (account?.provider !== "credentials") {
    const existingUser = await prisma.account.findFirst({
      where: { providerAccountId: account?.providerAccountId },
    });
    if (!existingUser && newSsoUsersDisabled) {
      return false; // 拒绝登录
    }

    // 子步骤 C：自动关联同邮箱的已存在账号
    if (user.email && account) {
      const findUser = await prisma.user.findFirst({
        where: { email: user.email },
        include: { accounts: true },
      });
      if (findUser && findUser.accounts.length === 0) {
        // 该用户已通过邮箱注册但尚未绑定任何 SSO Provider → 自动绑定
        await prisma.account.create({
          data: {
            userId: findUser.id,
            type: account.type,
            provider: account.provider,
            providerAccountId: account.providerAccountId,
            id_token: account.id_token,
            access_token: account.access_token,
            // ... 其余 token 字段
          },
        });
      }
    }
  }
  return true;
}
```

**关键逻辑**：
- `DISABLE_NEW_SSO_USERS=true` 时，从未通过 SSO 登录过的 `providerAccountId` 会被直接拒绝。
- 若邮箱相同且用户账号下尚无任何 `Account` 记录（典型场景：用户先通过普通注册创建账号，但未设置密码/未验证邮箱，随后尝试 Google SSO 登录），系统自动将该 SSO Provider 绑定到已存在的 User 上。

### 3.2 jwt 回调：JWT 载荷构建 + 新用户初始化

位置：[apps/web/pages/api/v1/auth/[...nextauth].ts#L1411-L1488](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1411-L1488)

```typescript
async jwt({ token, trigger, user }) {
  token.sub = token.sub ? Number(token.sub) : undefined;
  if (trigger === "signIn" || trigger === "signUp")
    token.id = user?.id as number;

  if (trigger === "signUp") {
    // 分支 A：新注册用户
    const userExists = await prisma.user.findUnique({
      where: { id: token.id },
      include: { accounts: true },
    });

    // A1：SSO 注册用户自动标记邮箱已验证 + 创建默认看板区块
    if (userExists && userExists.accounts.length > 0) {
      await prisma.user.update({
        where: { id: userExists.id },
        data: {
          emailVerified: new Date(),
          dashboardSections: {
            createMany: {
              data: [
                { order: 0, type: "STATS" },
                { order: 1, type: "RECENT_LINKS" },
                { order: 2, type: "PINNED_LINKS" },
              ],
            },
          },
        },
      });
    }

    // A2：SSO 用户若无 username，自动生成 "user<随机数>"
    if (userExists && !userExists.username) {
      await prisma.user.update({
        where: { id: token.id },
        data: { username: "user" + Math.round(Math.random() * 1000000000) },
      });
    }
  } else if (trigger === "signIn") {
    // 分支 B：已有用户登录，同样保证 username 存在
    const user = await prisma.user.findUnique({ where: { id: token.id } });
    if (user && !user.username) {
      // 生成 username...
    }
  }
  return token;
}
```

**JWT 载荷最终字段**（由类型定义 [apps/web/types/next-auth.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/types/next-auth.d.ts) 约束）：
```typescript
interface JWT {
  sub?: number;   // 兼容字段，值为 Number(token.sub)
  id: number;     // 用户 ID（实际鉴权使用此字段）
  iat: number;    // 签发时间
  exp: number;    // 过期时间（30 天后）
  jti: string;    // JWT ID，用于 AccessToken 表关联可撤销
}
```

### 3.3 session 回调：会话对象组装 + 订阅校验

位置：[apps/web/pages/api/v1/auth/[...nextauth].ts#L1489-L1511](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1489-L1511)

```typescript
async session({ session, token }) {
  session.user.id = token.id;  // 将 JWT 中的 id 注入 session.user

  if (STRIPE_SECRET_KEY) {
    const user = await prisma.user.findUnique({
      where: { id: token.id },
      include: { subscriptions: true, parentSubscription: true },
    });
    if (user) {
      await verifySubscription(user); // 触发 Stripe 订阅状态同步
    }
  }
  return session;
}
```

类型扩展见 [apps/web/types/next-auth.d.ts#L4-L14](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/types/next-auth.d.ts#L4-L14)：
```typescript
interface Session {
  user: { id: number };  // 仅暴露 user.id，不暴露邮箱等敏感信息
}
```

---

## 4. 用户会话建立

### 4.1 SessionProvider 注入（前端）

位置：[apps/web/pages/_app.tsx#L48-L114](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/_app.tsx#L48-L114)

```tsx
<SessionProvider
  session={pageProps.session}
  refetchOnWindowFocus={false}
  basePath="/api/v1/auth"    // 注意：非默认 "/api/auth"
>
  <AuthRedirect>
    <Component {...pageProps} />
  </AuthRedirect>
</SessionProvider>
```

关键点：
- `basePath="/api/v1/auth"` 对应 NextAuth 路由挂载位置 `/api/v1/auth/[...nextauth]`。
- `refetchOnWindowFocus={false}` 关闭窗口聚焦时自动续期。

### 4.2 JWT 的签发与 Cookie 存储

NextAuth `session.strategy = "jwt"`（[代码位置](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1316-L1319)）：

```typescript
session: {
  strategy: "jwt",
  maxAge: 30 * 24 * 60 * 60, // 30 天
},
```

JWT 通过 `next-auth/jwt` 的 `encode` 函数使用 `NEXTAUTH_SECRET` 签名，并以 **HttpOnly Cookie**（`next-auth.session-token`）形式存储在浏览器。

API Token（非浏览器会话）场景下使用相同 JWT 结构，由手动调用 `encode` 生成：
- 会话级 API Token：[apps/web/lib/api/controllers/session/createSession.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/controllers/session/createSession.ts)
- 用户自定义 Access Token：[apps/web/lib/api/controllers/tokens/postToken.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/controllers/tokens/postToken.ts)

### 4.3 请求鉴权：verifyToken / verifyUser / isAuthenticatedRequest

项目中存在三层鉴权辅助函数，形成递进关系：

```
getToken({ req })    // next-auth/jwt 提供：从 Cookie/Header 解析 JWT
       │
       ▼
verifyToken(req)     // 基础校验：userId 存在 + 未过期 + 未被撤销
       │
       ├────────────────────────────┐
       ▼                            ▼
verifyUser(req, res)      isAuthenticatedRequest(req)
 （完整用户校验 + 写响应）    （仅返回 user / null，不写响应）
```

**verifyToken**：[apps/web/lib/api/verifyToken.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/verifyToken.ts)
- 检查 `token.id` 是否存在
- 检查 `token.exp < Date.now()/1000` 是否已过期
- 检查 `prisma.accessToken` 中 `token.jti` 是否已被标记 `revoked=true`
- 成功返回 `JWT` 对象，失败返回错误字符串

**verifyUser**：[apps/web/lib/api/verifyUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/verifyUser.ts)
- 在 `verifyToken` 基础上：
  - 查询完整 User（含 subscriptions / parentSubscription）
  - 校验 `username` 是否存在
  - 若启用 Email Provider，校验 `emailVerified` 非空
  - 若启用 Stripe，校验订阅状态
- 失败时直接 `res.status(401).json(...)` 并返回 `null`

**isAuthenticatedRequest**：[apps/web/lib/api/isAuthenticatedRequest.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/isAuthenticatedRequest.ts)
- 与 `verifyUser` 逻辑类似但**不写 HTTP 响应**，适合在中间件或需要自定义响应的场景使用。

### 4.4 AccessToken 体系：可撤销的 API Token

数据模型：[packages/prisma/schema.prisma#L234-L246](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/packages/prisma/schema.prisma#L234-L246)

```prisma
model AccessToken {
  id         Int       @id @default(autoincrement())
  name       String                        // Token 名称，如 "iPhone"
  user       User      @relation(...)
  userId     Int
  token      String    @unique             // 存储 JWT.jti（非 JWT 本身）
  revoked    Boolean   @default(false)      // 软删除标记
  isSession  Boolean   @default(false)      // 是否为浏览器会话 Token
  expires    DateTime                      // 过期时间
  lastUsedAt DateTime?
  ...
}
```

**撤销机制**：JWT 本身无状态不可撤销，因此通过 `jti`（JWT ID）关联 `AccessToken.token` 字段实现"软撤销"。每次鉴权时查询 `revoked` 字段（见 `verifyToken` / `isAuthenticatedRequest`）。

**两种 Token 生成入口**：

| 类型 | 生成位置 | isSession | 有效期 |
|------|---------|-----------|--------|
| 浏览器会话登录 | `createSession()` | `true` | 200 年（实际由 JWT Cookie 30 天控制） |
| 用户手动创建 API Key | `postToken()` | `false` | 7/30/60/90 天或永不 |

两者均使用 `next-auth/jwt` 的 `encode` 生成相同格式 JWT，区别仅在于 `AccessToken.isSession` 标记。

### 4.5 前端路由守卫：AuthRedirect

位置：[apps/web/layouts/AuthRedirect.tsx](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/layouts/AuthRedirect.tsx)

`_app.tsx` 中所有页面均被 `AuthRedirect` 包裹，核心逻辑：

```tsx
const { status } = useSession();   // "loading" | "authenticated" | "unauthenticated"
const { data: user } = useUser();  // 后端 /api/v1/users/me 查询

useEffect(() => {
  const isLoggedIn = status === "authenticated";
  const isUnauthenticated = status === "unauthenticated";

  // 路由分级：公开页（/login, /register 等）vs 受保护页（/dashboard, /settings 等）
  if (isLoggedIn && hasInactiveSubscription) redirectTo("/subscribe");
  else if (isLoggedIn && !user?.name && user?.parentSubscriptionId) redirectTo("/member-onboarding");
  else if (isLoggedIn && !isProtectedRoute) redirectTo("/dashboard");
  else if (isUnauthenticated && isProtectedRoute) redirectTo("/login");
  else setShouldRenderChildren(true);
}, [status, user, router.pathname]);
```

同时调用 `useInitialData()` → `useSession()` 触发会话状态同步。

---

## 5. 数据模型

### Account（第三方账号绑定）

位置：[packages/prisma/schema.prisma#L10-L26](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/packages/prisma/schema.prisma#L10-L26)

```prisma
model Account {
  id                String  @id @default(cuid())
  userId            Int
  type              String        // "oauth" | "oidc" | "credentials" | "email"
  provider          String        // "google" | "github" | ...
  providerAccountId String        // 第三方平台用户唯一 ID
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  session_state     String?
  user              User    @relation(...)
  @@unique([provider, providerAccountId])
}
```

### User（用户）

位置：[packages/prisma/schema.prisma#L28-L53](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/packages/prisma/schema.prisma#L28-L53)

关键字段：
- `username`：唯一，可为空（SSO 新注册由 jwt 回调自动生成）
- `email`：唯一，可为空
- `emailVerified`：DateTime?（SSO 注册自动置为当前时间）
- `password`：仅 credentials 方式使用，bcrypt hash
- `accounts: Account[]`：关联的第三方登录账号

### VerificationToken

位置：[packages/prisma/schema.prisma#L108-L116](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/packages/prisma/schema.prisma#L108-L116)

Email Magic Link 验证所用，`@@unique([identifier, token])`。

---

## 6. 完整登录时序图

以 **Google SSO 新用户首次登录** 为例：

```
 浏览器                         NextAuth(/api/v1/auth)               Google                  Prisma
   │                                   │                               │                      │
   │  1. 点击 "Continue with Google"   │                               │                      │
   │──signIn("google")───────────────►│                               │                      │
   │                                   │  2. 302 重定向到 Google 授权   │                      │
   │◄──────────────────────────────────┤                               │                      │
   │                                   │                               │                      │
   │  3. 用户在 Google 完成授权         │                               │                      │
   │────────────────────────────────────────────────────────────────►│                      │
   │                                   │                               │                      │
   │  4. Google 带 code 回调           │                               │                      │
   │──────────────────────────────────►│                               │                      │
   │                                   │  5. code → token              │                      │
   │                                   │──────────────────────────────►│                      │
   │                                   │                               │                      │
   │                                   │  6. userinfo(profile)         │                      │
   │                                   │◄──────────────────────────────│                      │
   │                                   │                               │                      │
   │                                   │  7. callbacks.signIn()        │                      │
   │                                   │  ├─ 查 Account 是否存在        │                      │
   │                                   │  │   (不存在，且 DISABLE_NEW   │                      │
   │                                   │  │    _SSO_USERS=false → 通过)│                      │
   │                                   │  ├─ 查同 email 已有 User       │◄──SELECT user───     │
   │                                   │  │   (首次登录 → 无)           │                      │
   │                                   │  └─ return true                │                      │
   │                                   │                               │                      │
   │                                   │  8. PrismaAdapter.createUser  │                      │
   │                                   │     + linkAccount             │◄──INSERT user+account│
   │                                   │                               │                      │
   │                                   │  9. callbacks.jwt(trigger=    │                      │
   │                                   │     "signUp")                  │                      │
   │                                   │  ├─ token.id = user.id        │                      │
   │                                   │  ├─ emailVerified = now()     │◄──UPDATE user───     │
   │                                   │  ├─ 创建 dashboardSections    │◄──INSERT───          │
   │                                   │  └─ 生成 username             │◄──UPDATE user───     │
   │                                   │                               │                      │
   │                                   │  10. callbacks.session()      │                      │
   │                                   │     session.user.id = token.id│                      │
   │                                   │                               │                      │
   │                                   │  11. encode(JWT) + Set-Cookie │                      │
   │  ◄────────────────────────────────┤                               │                      │
   │      Set-Cookie: next-auth.session-token=...                      │                      │
   │                                   │                               │                      │
   │  12. 302 重定向 /dashboard        │                               │                      │
   │◄──────────────────────────────────┤                               │                      │
   │                                   │                               │                      │
   │  13. GET /dashboard               │                               │                      │
   │  Cookie: session-token ─────────►│                               │                      │
   │                                   │  14. getToken() 解析 JWT       │                      │
   │                                   │  15. AuthRedirect 判定已登录   │                      │
   │  ◄── 渲染 Dashboard ──────────────│                               │                      │
```

---

## 关键文件索引

| 文件 | 职责 |
|------|------|
| [apps/web/pages/api/v1/auth/[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts) | NextAuth 核心配置：Provider 注册、Adapter、Callbacks |
| [apps/web/pages/api/v1/logins/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/logins/index.ts) | 登录方式查询 API + `getLogins()` |
| [apps/web/pages/login.tsx](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/login.tsx) | 登录页，渲染账号密码表单 + SSO 按钮 |
| [apps/web/pages/_app.tsx](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/_app.tsx) | SessionProvider 注入 |
| [apps/web/layouts/AuthRedirect.tsx](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/layouts/AuthRedirect.tsx) | 前端路由守卫 |
| [apps/web/types/next-auth.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/types/next-auth.d.ts) | Session / JWT / User 类型扩展 |
| [apps/web/lib/api/verifyToken.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/verifyToken.ts) | JWT 基础校验 |
| [apps/web/lib/api/verifyUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/verifyUser.ts) | 完整用户鉴权（含邮箱/订阅校验） |
| [apps/web/lib/api/isAuthenticatedRequest.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/isAuthenticatedRequest.ts) | 轻量鉴权（不写 HTTP 响应） |
| [apps/web/lib/api/controllers/session/createSession.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/controllers/session/createSession.ts) | 程序化创建会话 JWT |
| [apps/web/lib/api/controllers/tokens/postToken.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/controllers/tokens/postToken.ts) | 创建用户级 API Token |
| [packages/prisma/schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/packages/prisma/schema.prisma) | User / Account / AccessToken 等数据模型 |
| [.env.sample](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/.env.sample) | 完整环境变量清单 |
