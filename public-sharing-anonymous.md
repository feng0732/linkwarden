# Linkwarden Public Sharing 与匿名访问安全边界分析

## 1. 概述

Linkwarden 的公开分享（Public Sharing）机制基于 **Collection 的 `isPublic` 布尔字段实现。当一个 Collection 被标记为公开后，其下的 Links、Tags 以及关联的归档文件（截图/PDF/Monolith/Readable）均可以被**未登录的匿名用户
访问。系统不存在传统的"share token"概念，公开访问完全依赖 Collection 的 `isPublic=true` 状态。

---

## 2. Share Token / Access Token 机制

### 2.1 系统中存在两种 Token：**NextAuth Session Token** 和 **Personal Access Token**，但它们均用于**已认证用户的 API 访问，与公开分享无直接关联。

### 2.2 Token 生成

位于 [postToken.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/controllers/tokens/postToken.ts)

```
编码后 Token 使用 `next-auth/jwt` 的 `encode()` 生成，基于 `NEXTAUTH_SECRET` 签名，Payload 结构：
{
  id: userId,
  iat: issuedAt,
  exp: expiryDate,
  jti: crypto.randomUUID()  // 存储在 AccessToken.token 字段
}
```

- 过期时间支持：7天（默认）、1个月、2个月、3个月、"永不过期"（200年

数据库存储字段：
- `AccessToken.token` 存储的是 JWT 的 `jti`（UUID），而非完整 JWT
- `AccessToken.revoked` 布尔字段标记是否被撤销
- `AccessToken.expires` 记录过期时间

### 2.3 Token 验证

位于 [verifyToken.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/verifyToken.ts)

验证流程：
1. 从请求 Cookie / `Authorization: Bearer <token>` 读取 JWT
2. 使用 `NEXTAUTH_SECRET` 解码
3. 检查 `exp` 是否过期
4. **查询数据库 `accessToken` 表，使用 `jti` 查询 `revoked: true`，若存在则拒绝
5. 对于公开 API（`/api/v1/public/*`）**不经过此验证，完全跳过 Token 检查

---

## 3. 公开页面与匿名访问路由

### 3.1 后端公开 API 路由

| 路由 | 用途 | 鉴权 |
|------|------|------|
| `/api/v1/public/collections/[id]` | 获取单个公开 Collection 信息 | 无，仅检查 `collection.isPublic === true` |
| `/api/v1/public/collections/links` | 获取公开 Collection 下的 Links 列表（分页+搜索） | 无，`publicOnly: true` 强制过滤 |
| `/api/v1/public/collections/tags` | 获取公开 Collection 下的 Tags | 无，先检查 `collection.isPublic === true` |
| `/api/v1/public/links/[id]` | 获取单个公开 Link 详情 | 无，检查 `link.collection.isPublic === true` |
| `/api/v1/public/users/[id]` | 获取公开用户信息 | 无，手动字段白名单裁剪 |
| `/api/v1/archives/[linkId]` | 获取归档文件（截图/PDF/Monolith/预览图） | **可选 Token**，`userId 有则用，无则匿名，检查 `collection.isPublic === true` 即可 |
| `/api/v1/preserved/token` | 生成带时效的归档访问 URL（仅配置 USER_CONTENT_DOMAIN 时） | 可选 Token |
| `/api/v1/preserved/view` | 通过短期 Token 访问归档 | 短期 JWT 验证 |

### 3.2 前端公开页面路由

| 路由 | 用途 |
|------|------|
| `/public/collections/[id]` | 公开 Collection 列表页 |
| `/public/links/[id]` | 公开 Link 详情页 |
| `/public/preserved/[id]` | 公开归档预览页（截图/PDF/Monolith/Readable） |

前端均通过 `router.pathname.startsWith("/public") 识别并走公开 API。

---

## 4. Collection 权限模型

### 4.1 数据模型

位于 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/packages/prisma/schema.prisma#L126-L149)

```prisma
model Collection {
  id         Int                   @id @default(autoincrement())
  isPublic   Boolean               @default(false)
  ownerId    Int
  owner      User                  @relation(...)
  members    UsersAndCollections[]
  links      Link[]
  // ...
}

model UsersAndCollections {
  userId       User       @relation(...)
  collectionId Int
  collection Collection @relation(...)
  canCreate    Boolean
  canUpdate    Boolean
  canDelete    Boolean
  @@id([userId, collectionId])
}
```

### 4.2 权限判定矩阵

| 场景 | 判定条件 | 代码位置 |
|------|----------|----------|
| 已登录用户访问私有 Collection | `ownerId === userId` **或** `members.some(m => m.userId === userId)` | [getPermission.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/getPermission.ts) |
| 匿名用户访问 | `collection.isPublic === true` | [getPublicCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/controllers/public/collections/getPublicCollection.ts) |
| 归档文件下载（含匿名
用户） | `ownerId === userId` **或** `members.some` **或** `isPublic === true`（三者任一即可 | [resolveAccessibleArchive.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/archives/resolveAccessibleArchive.ts#L30-L45) |

### 4.3 关键实现细节

`resolveAccessibleArchive` 的 OR 条件：
```typescript
OR: [
  { ownerId: userId || -1 },        // 所有者
  { members: { some: { userId: userId || -1 } },  // 成员
  { isPublic: true }  // 公开
]
```

**注意**：当 `userId` 为 `undefined`（匿名用户）时，前两个条件因 `userId || -1` 变为 `-1`，不可能匹配，仅 `isPublic: true` 生效。这是正确的匿名访问控制路径。

---

## 5. 字段裁剪（序列化过滤）

### 5.1 公开 Link 字段暴露情况

**公开 Links 列表 (`/api/v1/public/collections/links`)：
位于 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L126-L128) 和 [searchLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/controllers/search/searchLinks.ts#L228-L230)

```typescript
omit: {
  textContent: true,  // ✅ 已裁剪（网页提取的正文内容被排除
}
include: {
  tags: true,
  collection: true,
  pinnedBy: undefined,  // 匿名时为 undefined，不包含
}
```

**公开单个 Link** (`/api/v1/public/links/[id]`)：
位于 [getLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/controllers/public/links/linkId/getLinkById.ts)

```typescript
// ❌ 没有 omit textContent！
include: {
  tags: true,
  collection: true,
}
```

**对比已登录用户的 Link：
位于 [getLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/controllers/links/linkId/getLinkById.ts)

```typescript
// ❌ 同样没有 omit textContent
include: { tags: true, collection: true, pinnedBy: { select: { id: true } }
```

### 5.1.1 公开 Link 返回的 collection 对象元数据详解

当使用 Prisma `include: { collection: true }` 时，**仅返回 Collection 模型的标量字段（scalar fields），关联字段（relations）不会自动加载**。根据 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/packages/prisma/schema.prisma#L126-L149)，完整暴露字段如下：

| Collection 字段 | 是否暴露 | 类型 | 安全影响 |
|-----------------|---------|------|---------|
| `id` | ✅ | Int | 可用于遍历其他 API |
| `name` | ✅ | String | 正常元数据 |
| `description` | ✅ | String | 正常元数据 |
| `icon` | ✅ | String? | 正常元数据 |
| `iconWeight` | ✅ | String? | 正常元数据 |
| `color` | ✅ | String | 正常元数据（默认 #0ea5e9） |
| `parentId` | ✅ | Int? | **可用于发现父 Collection，借此探测 Collection 层级结构和未公开的父级 ID** |
| `isPublic` | ✅ | Boolean | **直接暴露该 Collection 的公开状态，可被用于探测私有 Collection 是否被标记为公开** |
| `ownerId` | ✅ | Int | **严重：暴露 Collection 所有者的用户 ID，可进一步调用 `/api/v1/public/users/{ownerId}` 获取用户信息，形成用户信息关联泄露链** |
| `createdById` | ✅ | Int? | 暴露创建者用户 ID（可能与 ownerId 不同） |
| `createdAt` | ✅ | DateTime | 时间元数据 |
| `updatedAt` | ✅ | DateTime | 时间元数据 |

**不返回的关联字段**（Prisma `include: true` 不自动加载 relations）：
- `owner`（User 对象）—— 但 `ownerId` 已暴露，可通过公开用户 API 间接获取
- `members`（UsersAndCollections[]）—— 但单独的 getPublicCollection API 会返回成员信息
- `links`、`subCollections`、`rssSubscriptions`、`DashboardSection`

**信息关联攻击链**：
```
匿名请求 /api/v1/public/links/{linkId}
    → 获取 collection.ownerId = 42
    → 请求 /api/v1/public/users/42
        → 获取该用户的 username、name、头像、归档偏好设置
    → 请求 /api/v1/public/collections/links?collectionId=42（可能存在其他公开 Collection）
        → 枚举该用户的所有公开内容
```

### 5.2 公开 User 字段裁剪与按 email 查找的安全影响

#### 5.2.1 字段白名单裁剪

位于 [getPublicUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/controllers/public/users/getPublicUser.ts#L26-L36)

```typescript
// 显式白名单，仅暴露字段：
{
  id, name, username, image,
  archiveAsScreenshot, archiveAsMonolith,
  archiveAsPDF
}

// 排除的字段（安全字段：password、
// password（解构赋值删除
// 未列出的：
// email, emailVerified, password
```

显式白名单裁剪了以下 User 模型敏感字段（完整模型见 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/packages/prisma/schema.prisma#L28-L75)）：

| 被裁剪的敏感字段 | 用途 |
|-----------------|------|
| `email` | 用户邮箱（✅ 未在响应中返回） |
| `emailVerified` | 邮箱验证时间 |
| `unverifiedNewEmail` | 待验证的新邮箱 |
| `password` | 密码哈希（✅ 解构删除） |
| `locale` | 地区设置 |
| `parentSubscriptionId` | 订阅关联 |
| `collectionOrder` | 用户 Collection 排序偏好 |
| `linksRouteTo` | 链接跳转偏好设置 |
| `aiTaggingMethod` / `aiPredefinedTags` / `aiTagExistingLinks` | AI 标签设置 |
| `theme` / `readableFontFamily` 等 | 界面与可读性设置 |
| `preventDuplicateLinks` | 防重复链接设置 |
| `archiveAsReadable` / `archiveAsWaybackMachine` | 仅返回了 3 个 archiveAs* 字段，其余被裁剪 |
| `isPrivate` | 用户是否私有（**此状态未在 API 中体现，私有用户的公开 Collection 仍可被访问**） |
| `referredBy` / `acceptPromotionalEmails` 等 | 营销相关 |

#### 5.2.2 按 email 查找的用户枚举漏洞

位于 [/api/v1/public/users/[id].ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/pages/api/v1/public/users/[id].ts#L5-L13) 和 [getPublicUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/controllers/public/users/getPublicUser.ts#L3-L22)

路由层的 ID 判定逻辑：
```typescript
// 只要 lookupId 不全是数字，就认为是 username/email 而非数字 ID
const isId = lookupId.split("").every((e) => Number.isInteger(parseInt(e)));
```

数据库查询条件（当 `isId === false` 时）：
```typescript
OR: [
  { username: targetId as string },
  { email: targetId as string },   // ⚠️ 允许按 email 精确匹配查找
]
```

**攻击方式：用户枚举（User Enumeration）**

攻击者可以通过 HTTP 响应状态码差异判断某个 email 是否在系统中注册：

| 请求 | 场景 | 响应 |
|------|------|------|
| `GET /api/v1/public/users/user@example.com` | 该 email 已注册 | `200 OK` + 返回用户白名单信息 |
| `GET /api/v1/public/users/notexist@example.com` | 该 email 未注册 | `404 Not Found` + `{ response: "User not found." }` |

**风险分析**：

1. **钓鱼攻击辅助**：攻击者可先批量验证某组织的员工邮箱是否注册了 Linkwarden，然后对确认存在的账户定向发送钓鱼邮件
2. **密码爆破前置侦察**：确认账户存在后，可针对该 username/email 进行后续的撞库或暴力破解
3. **信息收集**：即使响应不包含 email，确认某个 email 对应账户的存在本身即是信息泄露
4. **无速率限制**：该接口未实现任何速率限制或验证码，攻击者可使用字典大规模自动化枚举
5. **响应时间差异也可能成为旁路信号**：数据库命中与未命中的响应时间差异可用于绕过状态码统一化措施（当前代码未统一状态码）

**注意**：虽然查询条件是 `OR: [{ username }, { email }]`，响应中不会直接返回 email 字段，但攻击者可通过"我用 email A 请求，返回了用户 B 的 username 和 name"这一事实反推 email A ↔ 用户 B 的对应关系，构成间接邮箱泄露。

### 5.3 公开 Collection 字段裁剪

位于 [getPublicCollection.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/controllers/public/collections/getPublicCollection.ts#L10-L23)

```typescript
members: {
  include: {
    user: {
      select: {
        username: true,
        name: true,
        image: true,
        // ✅ 不包含 email、password 等敏感字段
      },
    },
  },
  _count: { select: { links: true } },
```

### 5.4 字段暴露汇总

| 字段 | 公开列表 | 公开单个 Link | 已登录 Link | 备注 |
|------|----------|-----------|-----------|------|
| `id` | ✅ | ✅ | ✅ | |
| `name` | ✅ | ✅ | ✅ | |
| `url` | ✅ | ✅ | ✅ | |
| `description` | ✅ | ✅ | ✅ | |
| `type` | ✅ | ✅ | ✅ | |
| `collectionId` | ✅ | ✅ | ✅ | |
| `tags` | ✅ | ✅ | ✅ | |
| `collection` (完整对象 | ✅ | ✅ | ✅ | |
| `preview` | ✅ | ✅ | ✅ | 预览状态字符串 |
| `image` | ✅ | ✅ | ✅ | 截图文件路径 |
| `pdf` | ✅ | ✅ | ✅ | PDF 文件路径 |
| `readable` | ✅ | ✅ | ✅ | Readable 内容路径 |
| `monolith` | ✅ | ✅ | ✅ | Monolith 文件路径 |
| `textContent` | ❌ 裁剪 | ⚠️ **未裁剪** | ⚠️ **未裁剪** | **潜在敏感：网页提取的网页正文
| `createdAt` | ✅ | ✅ | ✅ | |
| `updatedAt` | ✅ | ✅ | ✅ | |
| `pinnedBy` | ❌ | ❌ | ✅（仅当前用户） | |
| `icon`/`color` 等 | ✅ | ✅ | ✅ | |

### 5.5 textContent 裁剪差异的安全边界

#### 5.5.1 textContent 的来源与内容

textContent 字段由 Worker 在归档 Readability 流程中填充，见 [handleReadability.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/worker/lib/preservationScheme/handleReadability.ts#L20-L58)

```typescript
const article = new Readability(dom.window.document).parse();
const articleText = article?.textContent
  ?.replace(/ +(?= )/g, "")         // 去除连续多空格
  .replace(/(\r\n|\n|\r)/gm, " ") // 去除换行符，全部转为空格
  .slice(0, TEXT_CONTENT_LIMIT ? TEXT_CONTENT_LIMIT : undefined);
// 存储到数据库 Link.textContent
// 同时 JSON.stringify(article) 写入文件系统 archives/{collectionId}/{linkId}_readability.json
```

**textContent 存储内容**：
- 使用 `@mozilla/readability` 提取的网页**纯文本正文**
- 经过清洗（去多余空格、换行符、可选长度限制 `TEXT_CONTENT_LIMIT
- 可能包含：新闻全文、博客文章正文、产品描述、用户评论等网页可见文本

**textContent 与 readable 字段的区别**：

| 维度 | `textContent` | `readable` |
|------|--------------|-----------|
| 存储位置 | Postgres Link 表字段 | 文件系统 JSON 文件 |
| 存储路径 | N/A（直接读库 | `archives/{collectionId}/{linkId}_readability.json |
| 内容格式 | 纯文本字符串 | JSON（含 HTML 内容、标题、作者、元数据） |
| 获取方式 | `/api/v1/public/links/{id}` API 直接返回 | `/api/v1/archives/{linkId}?format=readability` 接口单独获取 |
| HTTP 缓存 | API 默认 API 响应（依赖 API 端缓存 | `max-age=31536000, immutable |
| 列表 API | ❌ 已裁剪 | ✅ 返回文件路径（可进一步获取 |
| 单 Link API | ⚠️ 未裁剪 | ✅ 返回文件路径（可进一步获取 |

#### 5.5.2 裁剪不一致造成的安全边界差异

两个端点的权限相同，但返回粒度差异造成了**安全边界不一致：

```
攻击者视角：
  ┌───────────────────────────────────────────────────────┐
  │  GET /api/v1/public/collections/links           │
  │   (分页列表  ← textContent: true（已裁剪
  │  │
  │  │   看不到网页正文，但能看到 readable 字段：
  │  │   readable: "archives/42/123_readability.json"
  │  │
  │  │   + 换一个端点
  │  ▼
  │  GET /api/v1/public/links/123
  │   (单 Link ← textContent: "完整网页正文内容..."（未裁剪
  │  └───────────────────────────────────────────────────────┘
```

**安全影响分析**：

1. **防御深度不同**：
   - 列表 API（searchLinks）出于性能考虑裁剪 textContent（大量 textContent 体积可能较大
   - 单 Link API 未做同样处理，两个端点安全策略不统

2. **绕过方式：**
   - 攻击者只需遍历所有 linkId（例如从 1 开始递增），逐个请求单 Link API，即可获取所有公开 Link 的 textContent
   - 列表裁剪形同虚设，列表看不到但详情页可以看到

3. **textContent 可能包含的敏感内容：
   - 付费墙后的文章正文（用户原本是通过 Readability 提取时可能提取到完整正文
   - 内部知识库页面内容
   - 含个人身份信息（PII）：姓名、邮箱、电话等
   - 版权受保护的文本内容

4. **与 readable 文件的双重获取**：
   - 即使 textContent 被裁剪，攻击者仍可通过 readable 字段的文件路径通过 `/api/v1/archives/{linkId}?format=readability` 获取**富文本（含 HTML 的完整 Readability JSON 输出，包含比 textContent 更完整的内容（含 HTML 标记、标题、作者、站点名、excerpt、dir、lang 等元数据

#### 5.5.3 textContent 的用途

textContent 在系统中的用途（非展示用途，仅作为 Worker AI 打标签（autoTagLink），在 [autoTagLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/worker/lib/autoTagLink.ts#L76-L78)：

```typescript
const description =
  (link.metaDescription ? link.metaDescription + "..." : undefined) ||
  (link.textContent ? link.textContent?.slice(0, 500) + "..." : undefined;
```

textContent 并未在任何前端界面渲染中展示使用，因此将其从**完全裁掉，没有任何功能损失**。

**结论**：单 Link API 中暴露 textContent 属于**过度数据暴露**，无业务必要且与列表 API 安全策略不一致。

---

## 6. 撤销（Revoke）后的缓存行为

### 6.1 Access Token 撤销机制

位于 [deleteTokenById.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/controllers/tokens/tokenId/deleteTokenById.ts)

撤销操作仅将 `AccessToken.revoked = true`，**不删除记录。

### 6.2 各层缓存分析

#### (1) **服务端 Token 验证缓存**

**无缓存**。每次 `verifyToken` / `isAuthenticatedRequest` 都直接查询数据库检查 `revoked` 状态：

```typescript
// verifyToken.ts:
const revoked = await prisma.accessToken.findFirst({
  where: { token: token.jti, revoked: true },
});
```

**撤销后**下一次请求**立即生效。

#### (2) **归档文件 HTTP 缓存

位于 [/api/v1/archives/[linkId].ts:

```typescript
.setHeader("Cache-Control", "private, max-age=31536000, immutable")
// max-age=31536000 = 1 年！
```

**重大风险**：
- `immutable` 表示浏览器在 max-age 内不会重新验证
- Collection 从公开改为私有后，已缓存的归档文件（截图/PDF/Monolith）在浏览器缓存中仍可被已访问长达 1 年
- `private` 仅表示不被共享缓存（CDN）缓存，不阻止浏览器本地缓存

#### (3) **短期 Preserved Token 缓存

位于 `/api/v1/preserved/token.ts:
```
Cache-Control: no-store  // ✅ 不缓存
```

位于 `/api/v1/preserved/view.ts:
```
Cache-Control: private, no-store  // ✅ 不缓存
```

#### (4) **前端 React Query 缓存

位于 `packages/router` 的 `publicLinks.tsx`、`links.tsx`、`links.tsx` 等：

```typescript
// useInfiniteQuery({
  queryKey: ["publicLinks", { params }],
  refetchOnWindowFocus: false,  // 窗口聚焦不重新获取
})
```

但在 `useUpdateLink`/`useDeleteLink` 等 mutation 成功后会调用 `queryClient.invalidateQueries(["publicLinks"])，主动失效。

**问题**：如果 Collection 的 `isPublic` 从 `true` 改为 `false` 时，前端缓存的数据依然在内存中保留，**直到页面刷新或**。

#### (5) **Meilisearch 索引

位于 `linkIndexing.ts 会更新 `indexVersion: null` 触发重新索引。但如果 Meilisearch 中 `collectionIsPublic` 字段的更新需要等 worker 重新索引完成才能生效，存在时间窗口。

### 6.3 撤销/关闭公开后的缓存失效时间汇总

| 层级 | 缓存策略 | 撤销后失效时间 | 风险等级 |
|------|---------|---------------|---------|
| Access Token DB 检查 | 无缓存，每次查 DB | 立即 | 低 |
| 归档文件浏览器缓存 | `max-age=31536000, immutable` | **最长 1 年** | **高** |
| Preserved 短期 Token | `no-store` | 立即 | 低 |
| Preserved View | `no-store` | 立即 | 低 |
| 前端 React Query 内存缓存 | `refetchOnWindowFocus: false` | 直到页面刷新 / mutation invalidate | 中 |
| Meilisearch 搜索索引 | 异步重新索引 | 分钟级延迟 | 中 |

---

## 7. 下载限制

### 7.1 无速率限制 / 配额限制

**系统中未实现下载速率限制、下载次数限制、IP 限制等。匿名用户可以无限制下载公开 Collection 的所有归档文件。

### 7.2 Monolith 下载的特殊限制

位于 [/api/v1/archives/[linkId].ts#L93-L102]

当配置了 `NEXT_PUBLIC_USER_CONTENT_DOMAIN` 时：

```typescript
if (
  format === ArchivedFormat.monolith &&
  process.env.NEXT_PUBLIC_USER_CONTENT_DOMAIN &&
  !hasBearerAuthorization
) {
  return 403 "Monolith archive access must use the user content domain...
}
```

即浏览器直接访问 Monolith 必须走 `/api/v1/preserved/token` → `/api/v1/preserved/view` 流程，使用 5 分钟 TTL 的短期 JWT。

短期 Token 位于 [createPreservedFormatUrl.ts](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/lib/api/preserved/createPreservedFormatUrl.ts#L7)

```typescript
export const PRESERVED_FORMAT_TOKEN_TTL_SECONDS = 300; // 5 分钟
```

但 **Bearer Authorization 的 Monolith 请求（如 API 调用）仍可直接下载，不受此限制。

其他格式（JPEG/PNG/PDF/Readable）**无任何下载限制。

### 7.3 文件大小限制

仅存在于上传时，下载时无限制。

---

## 8. 预览信息暴露

### 8.1 预览图（Banner 缩略图）

位于 [LinkDetails.tsx](file:///d:/fz/0601/solo-dogfeeding/code/90-linkwarden/apps/web/components/LinkDetails.tsx#L164-L179)

公开页面的 Link 详情页直接通过以下 URL 加载预览图：

```
/api/v1/archives/${link.id}?format=${ArchivedFormat.jpeg}&preview=true
```

该端点走 `resolveAccessibleArchive`，只要 Collection 公开即可匿名访问。预览图文件路径：
```
archives/preview/${collectionId}/${linkId}.jpeg
```

### 8.2 完整归档预览页

`/public/preserved/[id]` 页面支持 5 种格式：
- Readability（可读视图
- Monolith（完整网页
- PDF
- PNG/JPEG 截图

均通过 `resolveAccessibleArchive` 检查 Collection 是否公开，无额外鉴权。

### 8.3 暴露的文件路径模式

| 格式 | 路径模式 |
|------|---------|
| 预览 JPEG | `archives/preview/{collectionId}/{linkId}.jpeg` |
| 截图 PNG/JPEG | `archives/{collectionId}/{linkId}.png / .jpeg` |
| PDF | `archives/{collectionId}/{linkId}.pdf` |
| Monolith | `archives/{collectionId}/{linkId}.html` |
| Readable | `archives/{collectionId}/{linkId}.json` |

文件路径可通过 Link 对象的 `image`、`pdf`、`monolith`、`readable` 字段直接获得，这些字段在公开 Link API 中完整暴露。

### 8.4 textContent 暴露风险

**`textContent` 是 Link 模型存储的网页正文提取文本。在 `/api/v1/public/links/[id]`（单 Link API **未做 omit**，匿名用户可直接获取。这可能包含：
- 网页完整正文
- 敏感文本内容
- 作者信息

而 Link 列表 API（searchLinks）中已通过 `omit: { textContent: true }` 做了裁剪。

**不一致风险**：同一条 Link，列表看不到 textContent，详情可以。

---

## 9. 安全边界总结

### 9.1 已正确实现的安全措施

1. ✅ 公开 Collection 的判定统一基于 `collection.isPublic` 数据库层面过滤
2. ✅ User 信息采用白名单裁剪，不直接返回 email/password/emailVerified
3. ✅ Collection members 仅暴露 username/name/image
4. ✅ Access Token 撤销每次查 DB，立即生效
5. ✅ Preserved 短期 Token 5 分钟 TTL，no-store 缓存
6. ✅ Monolith 在启用 USER_CONTENT_DOMAIN 时强制走短期 Token
7. ✅ searchLinks 列表 API 裁剪 textContent
8. ✅ Highlights（高亮）接口需要 verifyUser 认证，公开路由无法访问
9. ✅ `private` Cache-Control 防止 CDN/代理缓存归档文件

### 9.2 潜在风险点

| 风险 | 位置 | 影响 | 严重等级 |
|------|------|------|---------|
| 归档文件浏览器缓存 1 年 immutable | `/api/v1/archives/[linkId].ts` | 撤销公开后 1 年内浏览器仍可访问已缓存文件 | **高** |
| 公开用户 API 支持按 email 查找导致用户枚举 | `/api/v1/public/users/[id].ts`、`getPublicUser.ts` | 可无速率限制地枚举系统用户邮箱，用于钓鱼/撞库前置侦察 | **高** |
| 公开 Link 返回 collection.ownerId 造成信息关联泄露链 | `public/links/linkId/getLinkById.ts`、`searchLinks.ts` | 暴露 Collection 所有者用户 ID，可进一步查询公开用户信息并枚举该用户所有公开内容 | **中** |
| 单 Link API 未裁剪 textContent | `public/links/linkId/getLinkById.ts` | 匿名用户可获取网页正文提取文本（含潜在 PII/版权内容），且与列表 API 策略不一致 | **中** |
| 公开 Link 返回 collection.parentId 可探测层级结构 | 同上 | 可用于发现父 Collection ID 及未公开的层级结构 | **低** |
| 公开 Link 返回 collection.isPublic 可探测公开状态 | 同上 | 可直接确认 Collection 是否为公开状态 | **低** |
| 无下载速率/次数限制 | 所有 `/api/v1/archives/*` | 匿名用户可无限制下载，潜在带宽滥用 | **中** |
| 前端 React Query 缓存不随 isPublic 变更自动失效 | `packages/router/*` | 关闭公开后前端仍显示旧数据直到刷新 | **低** |
| Meilisearch 索引异步更新 | worker 索引延迟 | 搜索结果中仍可搜到刚关闭公开的 Links（分钟级窗口） | **低** |
| User.isPrivate 字段未被公开用户 API 校验 | `getPublicUser.ts` | 标记为私有的用户其公开 Collection 仍可被访问，isPrivate 形同虚设 | **低** |
| textContent 裁剪与 readable 文件路径暴露形成双重获取 | `searchLinks.ts` vs `getLinkById.ts` vs `/api/v1/archives` | 即使 textContent 被裁剪，仍可通过 readable 字段获取更完整的富文本内容 | **中** |

### 9.3 架构图示

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        匿名用户请求                          │
└──────────────┬───────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│              /api/v1/public/*   无 Token 校验                 │
│  ┌──────────────────────────────────────────────────┐           │
│  │ collection.isPublic === true 数据库层过滤           │           │
│  │ 字段白名单裁剪 (User/Collection 成员)│           │
│  └──────────────────────────────────────────────────┘           │
└──────────────┬───────────────────────────────────────────────────────┘
               │
    ┌──────────┼──────────────────────────────┐
    ▼          ▼                      ▼
┌─────────┐ ┌──────────────┐ ┌──────────────────────┐
│ Links  │ │ Collection │ │ Tags          │
│ 列表/详情│ │   详情      │ │                     │
└────┬────┘ └──────┬─────┘ └──────────┬────────────────┘
     │             │              │
     ▼             ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│            /api/v1/archives/[linkId]                        │
│   resolveAccessibleArchive:                            │
│   owner OR member OR isPublic=true               │
│   Cache-Control: private, max-age=31536000, immutable │
└─────────────────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│  USER_CONTENT_DOMAIN 配置时 Monolith ?                        │
│  ├── 是 → /api/v1/preserved/token (5min TTL JWT, no-store) │
│  │         ↓                                            │
│  │    /api/v1/preserved/view (校验短期 Token, no-store)      │
│  └── 否 → 直接返回文件内容                              │
└─────────────────────────────────────────────────────────────────┘
```
