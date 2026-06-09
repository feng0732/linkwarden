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
   - 2.4 [Provider 注册条件与按钮显示条件差异](#24-provider-注册条件与按钮显示条件差异)
   - 2.5 [PrismaAdapter 与 linkAccount 定制](#25-prismaadapter-与-linkaccount-定制)
3. [回调处理（Callbacks）与路由](#3-回调处理callbacks与路由)
   - 3.1 [NextAuth 回调路由与 basePath 详解](#31-nextauth-回调路由与-basepath-详解)
   - 3.2 [signIn 回调：准入控制 + 自动账号关联](#32-signin-回调准入控制--自动账号关联)
   - 3.3 [jwt 回调：JWT 载荷构建 + 新用户初始化](#33-jwt-回调jwt-载荷构建--新用户初始化)
   - 3.4 [session 回调：会话对象组装 + 订阅校验](#34-session-回调会话对象组装--订阅校验)
4. [用户会话建立](#4-用户会话建立)
   - 4.1 [SessionProvider 注入（前端）](#41-sessionprovider-注入前端)
   - 4.2 [JWT 的签发与 Cookie 存储](#42-jwt-的签发与-cookie-存储)
   - 4.3 [浏览器会话 Cookie vs API Token Authorization Header：边界详解](#43-浏览器会话-cookie-vs-api-token-authorization-header边界详解)
   - 4.4 [请求鉴权：verifyToken / verifyUser / isAuthenticatedRequest](#44-请求鉴权verifytoken--verifyuser--isauthenticatedrequest)
   - 4.5 [AccessToken 体系：可撤销的 API Token](#45-accesstoken-体系可撤销的-api-token)
   - 4.6 [前端路由守卫：AuthRedirect](#46-前端路由守卫authredirect)
5. [数据模型](#5-数据模型)
6. [完整登录时序图](#6-完整登录时序图)
7. [代码缺陷与不一致汇总](#7-代码缺陷与不一致汇总)

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

### 2.4 Provider 注册条件与按钮显示条件差异

项目中三处（NextAuth Provider 注册、登录按钮清单、.env.sample 文档）使用的环境变量存在**多处不一致**。以下是完整对照表：

| Provider | 后端 Provider 注册条件 ([...nextauth].ts) | 前端按钮显示条件 (logins/index.ts) | .env.sample 文档变量 | 差异说明 |
|---|---|---|---|---|
| **Zoom** | `NEXT_PUBLIC_ZOOM_ENABLED_ENABLED` | `NEXT_PUBLIC_ZOOM_ENABLED` | `NEXT_PUBLIC_ZOOM_ENABLED` | **BUG**: 注册条件变量名拼写错误，多了一个 `_ENABLED`。按 .env.sample 设置变量后，登录按钮会显示，但点击后 NextAuth 会报 "unknown provider"，因为 Provider 实际未注册。代码位置：[...nextauth].ts#L1298](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1298) |
| **Azure AD B2C** | `NEXT_PUBLIC_AZURE_AD_ENABLED` | `NEXT_PUBLIC_AZURE_AD_B2C_ENABLED` | `NEXT_PUBLIC_AZURE_AD_B2C_ENABLED` | **不一致**: B2C 按钮由 `AZURE_AD_B2C_ENABLED` 独立控制，但 B2C Provider 的**实际注册**与普通 Azure AD 共享 `AZURE_AD_ENABLED` 开关。若只设 `AZURE_AD_B2C_ENABLED=true` 未设 `AZURE_AD_ENABLED=true`，按钮会显示但点击失败。代码位置：[...nextauth].ts#L385](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L385) |
| **Azure AD (普通)** | `NEXT_PUBLIC_AZURE_AD_ENABLED` | `NEXT_PUBLIC_AZURE_AD_ENABLED` | `NEXT_PUBLIC_AZURE_AD_ENABLED` | ✅ 一致。但与 B2C 共用开关意味着启用 B2C 必须同时启用普通 Azure AD。 |
| **Apple** | `APPLE_CLIENT_ID` / `APPLE_CLIENT_SECRET` | `APPLE_CUSTOM_NAME` | `APPLE_ID` / `APPLE_SECRET` | **不一致**: .env.sample 文档的变量名是 `APPLE_ID` 和 `APPLE_SECRET`，但代码实际读取的是 `APPLE_CLIENT_ID` 和 `APPLE_CLIENT_SECRET`。按文档设置变量会导致 Apple Provider 因缺少 clientId 而启动失败。代码位置：[...nextauth].ts#L248-L249](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L248-L249) |
| **Bungie** | ❌ 完全未实现 | ❌ 完全未实现 | `NEXT_PUBLIC_BUNGIE_ENABLED` / `BUNGIE_CLIENT_ID` / `BUNGIE_CLIENT_SECRET` / `BUNGIE_API_KEY` | **缺失**: .env.sample 中列出了完整的 Bungie 配置变量（[.env.sample#L176-L180](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/.env.sample#L176-L180)），但无论是 [...nextauth].ts 的 Provider 注册还是 logins/index.ts 的按钮清单，均未包含 Bungie。属于"文档已写但代码未实现"。 |
| **Authelia** | `NEXT_PUBLIC_AUTHELIA_ENABLED` | `NEXT_PUBLIC_AUTHELIA_ENABLED` | `NEXT_PUBLIC_AUTHELIA_ENABLED` | ✅ 变量名一致。但 logins/index.ts 中 Authelia 被放在文件**末尾**（[logins/index.ts#L416-L421](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/logins/index.ts#L416-L421)）而非按字母顺序排列，属于代码组织不一致。 |
| **Synology** | `NEXT_PUBLIC_SYNOLOGY_ENABLED` | `NEXT_PUBLIC_SYNOLOGY_ENABLED` | `NEXT_PUBLIC_SYNOLOGY_ENABLED` | ✅ 一致。使用裸 `type: "oauth"` 手写配置，非 next-auth 预设 Provider。 |

**风险总结**：由于三处（Provider 注册 / 按钮清单 / env 文档）彼此独立维护，任何新增 Provider 或修改变量名都需要三处同步，否则会出现"按钮显示但登录失败"或"文档有但实际不可用"的情况。

### 2.5 PrismaAdapter 与 linkAccount 定制

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

## 3. 回调处理（Callbacks）与路由

### 3.1 NextAuth 回调路由与 basePath 详解

项目使用 NextAuth.js v4.22.1（见 [apps/web/package.json#L63](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/package.json#L63)），其路由并非默认的 `/api/auth/*`，而是通过 `basePath` 重定位到 `/api/v1/auth/*`。

**三处配置必须一致**：

| 配置位置 | 配置项 | 值 |
|---|---|---|
| 环境变量 | `NEXTAUTH_URL` | `http://localhost:3000/api/v1/auth` |
| 前端 SessionProvider | `basePath` | `/api/v1/auth`（[代码位置](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/_app.tsx#L348)） |
| Next.js 文件系统路由 | 文件路径 | `/pages/api/v1/auth/[...nextauth].ts`（Catch-all 路由） |

**NextAuth 标准回调路由表**（均挂载于 `/api/v1/auth/` 下）：

| 路由 | HTTP 方法 | 作用 |
|---|---|---|
| `GET /signin` | GET | 展示所有可用 Provider 的登录入口页（若配置了 `pages.signIn` 自定义页则重定向到该页） |
| `GET /signin/:provider` | GET | 发起具体 Provider 的 OAuth 授权流程（302 重定向到第三方） |
| `POST /signin/:provider` | POST | Email Provider 提交邮箱地址 |
| `GET /callback/:provider` | GET | **核心回调路由**：第三方 OAuth/OIDC Provider 携带 `code` 回调到此，NextAuth 完成 code→token→userinfo 交换 |
| `POST /callback/:provider` | POST | Credentials Provider 的登录提交 |
| `GET /session` | GET | 查询当前会话（SessionProvider 自动调用） |
| `POST /session` | POST | 更新会话（不常用） |
| `GET /csrf` | GET | 获取 CSRF Token |
| `POST /signout` | POST | 登出：清除 Cookie + 触发事件 |
| `GET /providers` | GET | 返回已启用 Provider 列表 JSON |

**自定义页面覆盖**（[代码位置](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1321-L1324)）：

```typescript
pages: {
  signIn: "/login",            // 覆盖默认 /api/v1/auth/signin，重定向到 /login
  verifyRequest: "/confirmation"  // 邮件发送确认页，对应 Email Provider
},
```

**Email Magic Link 回调 URL 构造**：

Email Provider 发送的邮件中的登录链接并非由 NextAuth 默认生成，而是手动构造（见 [sendVerificationRequest.ts#L38-L40](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/sendVerificationRequest.ts#L38-L40) 和 [sendInvitationRequest.ts#L43-L44](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/sendInvitationRequest.ts#L43-L44)）：

```typescript
const callbackUrl = `${process.env.NEXTAUTH_URL}/callback/email?token=${token}&email=${encodeURIComponent(identifier)}`;
```

用户点击邮件链接 → `GET /api/v1/auth/callback/email?token=...&email=...` → NextAuth 验证 `VerificationToken` → 通过则签发 JWT Cookie → 重定向到首页。

**OAuth Provider 回调 URL 格式**（以 Google 为例）：
- 第三方授权服务器回调地址：`{NEXTAUTH_URL}/callback/google`（即 `http://localhost:3000/api/v1/auth/callback/google`）
- 该地址需要在 Google Cloud Console 的 OAuth Client 配置的 "Authorized redirect URIs" 中白名单化
- NextAuth 在发起授权请求时会自动将此 URL 作为 `redirect_uri` 参数传递

**注意 `/api/v1/session` 与 `/api/v1/auth/session` 的区别**（两个同名但完全不同的端点）：

| 端点 | 归属 | 作用 |
|---|---|---|
| `GET /api/v1/auth/session` | NextAuth 内置 | 由 SessionProvider 自动调用，返回当前 Cookie 对应的 session 对象 |
| `POST /api/v1/session` | 项目自定义 API | 使用**用户名/密码**换取 API Token（走 `verifyByCredentials` + `createSession`），不涉及 OAuth 流程。代码：[apps/web/pages/api/v1/session/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/session/index.ts) |

### 3.2 signIn 回调：准入控制 + 自动账号关联

NextAuth 的三个核心回调在 `[...nextauth].ts` 的 `callbacks` 字段中定义，执行顺序为：**signIn → jwt → session**。

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

### 3.3 jwt 回调：JWT 载荷构建 + 新用户初始化

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

### 3.4 session 回调：会话对象组装 + 订阅校验

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

### 4.3 浏览器会话 Cookie vs API Token Authorization Header：边界详解

项目使用的 JWT 鉴权对后端完全透明——`next-auth/jwt` 的 `getToken({ req })` 函数会自动从两个来源提取 JWT，**调用方无需关心来源**。

**Token 提取的双重来源**：

```
                    getToken({ req })
                           │
           ┌───────────────┴───────────────┐
           ▼                               ▼
  Cookie: next-auth.session-token     Authorization: Bearer <jwt>
  (浏览器自动携带，HttpOnly)           (移动端 / API 调用手动设置)
```

`getToken` 的提取优先级（NextAuth v4 源码逻辑）：
1. 先尝试从 Cookie 中读取（`next-auth.session-token` 或 `__Secure-next-auth.session-token`）
2. 若无 Cookie，尝试从 `Authorization` 请求头提取 `Bearer <token>`
3. 都没有则返回 `null`

**浏览器会话 vs API Token 的边界对比**：

| 维度 | 浏览器会话 (Browser Session) | API Token / 移动端会话 |
|---|---|---|
| **传递方式** | HttpOnly Cookie (`next-auth.session-token`) | `Authorization: Bearer <jwt>` Header |
| **创建方式** | OAuth/Email/Credentials 登录成功后 NextAuth 自动 Set-Cookie | `POST /api/v1/session`（用户名密码换 Token）或 `POST /api/v1/tokens`（用户手动创建 API Key） |
| **JWT 有效期** | 30 天（由 `session.maxAge` 控制） | 7/30/60/90 天或"永不"（200 年），由创建时参数决定 |
| **AccessToken.isSession** | `true`（OAuth/Credentials/Email 登录的会话会写入 AccessToken 表） | `false`（用户手动创建的 API Key）或 `true`（`POST /api/v1/session` 创建的程序化会话） |
| **客户端存储** | 浏览器 Cookie 存储，JS 不可读（HttpOnly） | 移动端 App / 第三方脚本自行安全存储 |
| **自动携带** | 浏览器对同源请求自动携带 Cookie | 调用方需手动在每个请求中添加 Header |
| **使用方** | Web 前端（[apps/web](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web)） | 移动端 App（[apps/mobile](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/mobile)）、第三方 API 集成、浏览器扩展等 |
| **撤销方式** | `DELETE /api/v1/tokens/:id` 设置 `revoked=true` | `DELETE /api/v1/tokens/:id` 设置 `revoked=true`（同一机制） |

**移动端的 Bearer Token 使用示例**（来自 [packages/router/user.tsx#L24-L38](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/packages/router/user.tsx#L24-L38)）：

```typescript
const url = auth?.instance + "/api/v1/users/me";
const response = await fetch(url, {
  auth?.session
    ? { headers: { Authorization: `Bearer ${auth.session}` } }
    : undefined  // Web 端走 Cookie，不需要 Header
});
```

**鉴权时后端不区分来源**：`verifyToken`、`verifyUser`、`isAuthenticatedRequest` 都只调用 `getToken({ req })`，无论是浏览器 Cookie 还是 Bearer Header 来的 JWT，走的是完全相同的校验逻辑（查 `token.id`、验过期、查 `AccessToken.revoked`）。

**AccessToken 表中两类记录的区分**：
- `isSession=true`：浏览器 OAuth/Email/Credentials 登录产生的会话，或 `POST /api/v1/session` 程序化创建的会话。前端"已登录设备"列表中会显示。
- `isSession=false`：用户在"设置 → 访问令牌"页面手动创建的 API Key。

两类记录在 `GET /api/v1/tokens` 中都会返回（[getTokens.ts#L5-L8](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/controllers/tokens/getTokens.ts#L5-L8)），撤销接口也相同。

### 4.4 请求鉴权：verifyToken / verifyUser / isAuthenticatedRequest

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

### 4.5 AccessToken 体系：可撤销的 API Token

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

### 4.6 前端路由守卫：AuthRedirect

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

## 7. 代码缺陷与不一致汇总

从代码实现角度梳理出以下可维护性问题和潜在 Bug：

### 7.1 Bug：Zoom Provider 启用变量名拼写错误

- **位置**：[...nextauth].ts#L1298](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L1298)
- **代码**：`if (process.env.NEXT_PUBLIC_ZOOM_ENABLED_ENABLED === "true")`
- **问题**：变量名多了一个 `_ENABLED`，应为 `NEXT_PUBLIC_ZOOM_ENABLED`。按 .env.sample 正确设置变量后，Zoom 登录按钮会在前端显示（logins/index.ts 用的是正确变量名），但点击后 NextAuth 会报 "unknown provider: zoom"，因为 Provider 实际上没有被注册。

### 7.2 不一致：Azure AD B2C 的注册条件与按钮条件不同

- **后端 Provider 注册**：使用 `NEXT_PUBLIC_AZURE_AD_ENABLED`（与普通 Azure AD 共用开关）[[...nextauth].ts#L385](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L385)
- **前端按钮显示**：使用 `NEXT_PUBLIC_AZURE_AD_B2C_ENABLED`（独立开关）[logins/index.ts#L59](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/logins/index.ts#L59)
- **问题**：若只设置 `AZURE_AD_B2C_ENABLED=true` 而未设置 `AZURE_AD_ENABLED=true`，用户会看到 Azure AD B2C 登录按钮但点击后登录失败。同时，启用普通 Azure AD 时会强制注册 B2C Provider，即使不需要 B2C。

### 7.3 不一致：Apple Provider 的 Client 变量名与文档不符

- **代码实际读取**：`APPLE_CLIENT_ID`、`APPLE_CLIENT_SECRET` [[...nextauth].ts#L248-L249](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts#L248-L249)
- **.env.sample 文档**：`APPLE_ID`、`APPLE_SECRET` [.env.sample#L119-L120](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/.env.sample#L119-L120)
- **问题**：部署时按 .env.sample 设置变量会导致 Apple Provider 因缺少 `clientId` 而在运行时报错。

### 7.4 缺失：Bungie Provider 文档已写但代码未实现

- **.env.sample** 中有完整的 Bungie 配置变量（`NEXT_PUBLIC_BUNGIE_ENABLED`、`BUNGIE_CLIENT_ID`、`BUNGIE_CLIENT_SECRET`、`BUNGIE_API_KEY`）[.env.sample#L176-L180](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/.env.sample#L176-L180)
- **[...nextauth].ts** 和 **logins/index.ts** 中完全没有 Bungie 相关代码
- **问题**：属于"文档先行但代码未落地"，用户会被误导认为 Bungie 登录已支持。

### 7.5 架构问题：Provider 配置三处独立维护

| 维护位置 | 内容 | 量级 |
|---|---|---|
| `[...nextauth].ts` | Provider 注册 + linkAccount patch + 启用条件判断 | 40+ 段几乎相同的代码块 |
| `logins/index.ts` | 按钮显示条件 + 自定义名称 | 40+ 段几乎相同的 if 判断 |
| `.env.sample` | 环境变量文档 | 40+ 组变量说明 |

**问题**：
- 新增一个 Provider 需要三处同步修改，极易遗漏（如 Zoom/B2C/Apple/Bungie 的问题已证实）
- `linkAccount` 的猴子补丁被重复执行 40+ 次（每个 OAuth Provider 启用时都会对同一个函数再 patch 一次），虽然逻辑上不影响正确性，但存在性能浪费和可维护性问题
- 40+ 段 `if (NEXT_PUBLIC_XXX_ENABLED)` 判断 + `adapter.linkAccount = (account) => {...}` 代码完全重复，建议重构为配置表驱动

### 7.6 命名冲突：`/api/v1/session` 与 `/api/v1/auth/session` 两个同名端点

| 端点 | 作用 | 鉴权方式 |
|---|---|---|
| `GET /api/v1/auth/session` | NextAuth 内置会话查询，返回 `{ user: { id } }` | NextAuth Cookie / Bearer |
| `POST /api/v1/session` | 自定义端点，用用户名密码换 API Token | 无（需要传 username+password） |

**问题**：开发者不仔细看容易混淆，尤其 `POST /api/v1/session` 的语义在 RESTful 中通常是"创建/更新会话"，但实际它是"用凭据换 Token"的认证端点，和 NextAuth 的 session 概念完全不同。

---

## 8. 关键文件索引

| 文件 | 职责 |
|------|------|
| [apps/web/pages/api/v1/auth/[...nextauth].ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/auth/%5B...nextauth%5D.ts) | NextAuth 核心配置：Provider 注册、Adapter、Callbacks |
| [apps/web/pages/api/v1/logins/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/logins/index.ts) | 登录方式查询 API + `getLogins()` 按钮清单 |
| [apps/web/pages/api/v1/session/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/session/index.ts) | 自定义端点：用户名密码换 API Token（非 NextAuth 内置） |
| [apps/web/pages/api/v1/users/me.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/users/me.ts) | /users/me 端点，仅使用 verifyToken 轻量鉴权 |
| [apps/web/pages/api/v1/tokens/[id].ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/api/v1/tokens/%5Bid%5D.ts) | Token 撤销端点（软删除 revoked=true） |
| [apps/web/pages/confirmation.tsx](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/confirmation.tsx) | Email Magic Link 发送确认页（NextAuth pages.verifyRequest） |
| [apps/web/pages/login.tsx](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/login.tsx) | 登录页，渲染账号密码表单 + SSO 按钮 |
| [apps/web/pages/_app.tsx](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/pages/_app.tsx) | SessionProvider 注入（含 basePath="/api/v1/auth"） |
| [apps/web/layouts/AuthRedirect.tsx](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/layouts/AuthRedirect.tsx) | 前端路由守卫 |
| [apps/web/types/next-auth.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/types/next-auth.d.ts) | Session / JWT / User 类型扩展 |
| [apps/web/lib/api/sendVerificationRequest.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/sendVerificationRequest.ts) | Email Magic Link 发送：手动构造 callback URL |
| [apps/web/lib/api/sendInvitationRequest.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/sendInvitationRequest.ts) | 邀请邮件发送：手动构造 callback URL |
| [apps/web/lib/api/verifyToken.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/verifyToken.ts) | JWT 基础校验（id/exp/revoked） |
| [apps/web/lib/api/verifyUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/verifyUser.ts) | 完整用户鉴权（含邮箱/订阅校验，会写 HTTP 响应） |
| [apps/web/lib/api/isAuthenticatedRequest.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/isAuthenticatedRequest.ts) | 轻量鉴权（不写 HTTP 响应） |
| [apps/web/lib/api/controllers/session/createSession.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/controllers/session/createSession.ts) | 程序化创建会话 JWT（isSession=true） |
| [apps/web/lib/api/controllers/tokens/postToken.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/controllers/tokens/postToken.ts) | 创建用户级 API Token（isSession=false） |
| [apps/web/lib/api/controllers/tokens/getTokens.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/controllers/tokens/getTokens.ts) | 查询用户所有未撤销 Token |
| [apps/web/lib/api/controllers/tokens/tokenId/deleteTokenById.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/web/lib/api/controllers/tokens/tokenId/deleteTokenById.ts) | 软删除 Token（revoked=true） |
| [packages/router/user.tsx](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/packages/router/user.tsx) | 移动端 useUser：演示 Authorization: Bearer 使用方式 |
| [packages/router/tokens.tsx](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/packages/router/tokens.tsx) | Token 增删查 React Query Hooks |
| [packages/prisma/schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/packages/prisma/schema.prisma) | User / Account / AccessToken / VerificationToken 数据模型 |
| [.env.sample](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/.env.sample) | 完整环境变量清单（含 40+ Provider 配置） |
