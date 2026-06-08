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

### 5.2 公开 User 字段裁剪

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

1. ✅ 公开 Collection 的判定统一基于 `collection.isPublic` 数据库层面过滤，
2. ✅ User 信息采用白名单裁剪，不暴露 email/password
3. ✅ Collection members 仅暴露 username/name/image
4. ✅ Access Token 撤销每次查 DB，立
5. ✅ Preserved 短期 Token 5 分钟 TTL，no-store 缓存
6. ✅ Monolith 在启用 USER_CONTENT_DOMAIN 时强制走短期 Token
7. ✅ searchLinks 列表 API 裁剪 textContent

### 9.2 潜在风险点

| 风险 | 位置 | 影响 |
|------|------|------|
| 归档文件浏览器缓存 1 年 immutable | `/api/v1/archives/[linkId].ts
| 撤销公开后 1 年内浏览器仍可访问已缓存文件
|
| 单 Link API 未裁剪 textContent | `public/links/linkId/getLinkById.ts` | 匿名用户可获取网页正文提取文本
| 无下载速率/次数限制 | 所有 `/api/v1/archives/*` | 匿名用户可无限制下载
| 前端 React Query 缓存不随 isPublic 变更自动失效 | `packages/router/*` | 关闭公开后前端仍显示旧数据直到刷新
| Meilisearch 索引异步更新 | worker 索引延迟
| 搜索结果中仍可搜到刚关闭公开的 Links（分钟级窗口
|

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
