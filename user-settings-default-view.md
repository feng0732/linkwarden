# 用户设置与默认视图关联路径梳理

本文档梳理 Linkwarden 项目中用户偏好、工作区默认值、列表布局、通知设置及多设备同步之间的代码关联路径，特别关注本地化和缓存更新的实现。

---

## 一、整体架构概览

Linkwarden 的用户设置采用 **双层存储架构**：

| 层级 | 存储位置 | 同步范围 | 典型数据 |
|------|----------|----------|----------|
| 服务端持久层 | PostgreSQL (Prisma) | 跨设备同步 | 主题、语言、归档偏好、Dashboard布局、AI设置 |
| 客户端本地层 | Web: localStorage / Mobile: AsyncStorage+SecureStore | 单设备本地 | 视图模式、列数、显示项、颜色主题、公告状态 |

数据流的核心原则：**服务端数据通过 React Query 缓存并提供乐观更新，本地数据通过 Zustand 直接操作存储并同步状态。**

---

## 二、用户偏好设置（服务端同步层）

### 2.1 数据库模型定义

用户偏好字段全部定义在 `User` 模型中：

[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/prisma/schema.prisma#L28-L75)

核心字段及默认值：

```prisma
model User {
  locale                  String                @default("en")          // 本地化语言
  theme                   Theme                 @default(dark)          // 主题 (dark/light/auto)
  linksRouteTo            LinksRouteTo          @default(ORIGINAL)      // 点击链接跳转目标
  preventDuplicateLinks   Boolean               @default(false)         // 防止重复链接
  archiveAsScreenshot     Boolean               @default(true)          // 归档：截图
  archiveAsMonolith       Boolean               @default(true)          // 归档：网页
  archiveAsPDF            Boolean               @default(true)          // 归档：PDF
  archiveAsReadable       Boolean               @default(true)          // 归档：可读性
  archiveAsWaybackMachine Boolean               @default(false)         // 归档：Wayback Machine
  aiTaggingMethod         AiTaggingMethod       @default(DISABLED)      // AI标签方式
  aiPredefinedTags        String[]              @default([])            // AI预定义标签
  aiTagExistingLinks      Boolean               @default(false)         // AI标记已有链接
  readableFontFamily      String?               @default("sans-serif")  // 可读视图字体
  readableFontSize        String?               @default("20px")        // 可读视图字号
  readableLineHeight      String?               @default("1.8")         // 可读视图行高
  readableLineWidth       String?               @default("normal")      // 可读视图行宽
  dashboardSections       DashboardSection[]                             // Dashboard布局
  acceptPromotionalEmails Boolean               @default(false)         // 接收促销邮件
}
```

### 2.2 偏好更新的两条 API 路径

项目将用户偏好更新拆分为两条独立的 API 路径：

#### 路径 A：`/api/v1/users/[id]/preference`（PUT）
专门处理 **主题与可读性设置**。

- API 路由：[preference.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/api/v1/users/[id]/preference.tsx)
- 控制器：[updateUserPreference.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserPreference.ts)
- 可更新字段：`theme`, `readableFontFamily`, `readableFontSize`, `readableLineHeight`, `readableLineWidth`
- 权限校验：`user.id !== queryId` 返回 401

#### 路径 B：`/api/v1/users/[id]`（PUT）
处理 **除主题/可读性外的所有用户设置**。

- 控制器：[updateUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserById.ts)
- 可更新字段：`name`, `username`, `email`, `password`, `image`, `collectionOrder`, `locale`, `archiveAs*`, `linksRouteTo`, `preventDuplicateLinks`, `aiTaggingMethod`, `aiPredefinedTags`, `aiTagExistingLinks`

### 2.3 前端设置页面

偏好设置页面统一在：[preference.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/settings/preference.tsx)

该页面分为四个设置区块，各自独立保存：

| 区块 | 使用的 Mutation | 更新内容 |
|------|----------------|----------|
| 主题/颜色 | `useUpdateUserPreference` + `updateSettings`（本地） | 主题（服务端）+ 主题色（本地） |
| AI设置 | `useUpdateUser` | aiTaggingMethod, aiPredefinedTags, aiTagExistingLinks |
| 归档设置 | `useUpdateUser` + `useUpsertTags` | 全局归档偏好 + 每个Tag的归档规则 |
| 链接设置 | `useUpdateUser` | preventDuplicateLinks, linksRouteTo |

关键交互：主题切换时 **双重写入** —— 同时调用服务端 API 并立即设置 DOM 属性：

```typescript
onClick={() => {
  updateUserPreference.mutate({ theme: theme as any });
  document.documentElement.setAttribute("data-theme", theme);  // 立即生效
}}
```

---

## 三、工作区默认值与列表布局

### 3.1 Dashboard 布局（服务端同步）

Dashboard 布局存储在独立的 `DashboardSection` 表中，支持用户自定义显示区块和排序。

**数据模型**：[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/prisma/schema.prisma#L275-L294)

```prisma
model DashboardSection {
  userId       Int
  collectionId Int?
  type         DashboardSectionType   // STATS | RECENT_LINKS | PINNED_LINKS | COLLECTION
  order        Int                    // 显示顺序
  @@unique([userId, collectionId])
}
```

**更新流程**：

1. UI 组件：[DashboardLayoutDropdown.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/components/DashboardLayoutDropdown.tsx)
   - 支持勾选启用/禁用区块
   - 支持拖拽排序（@dnd-kit）
   - 可搜索过滤

2. 更新 API：`PUT /api/v2/dashboard`
   - 控制器：[updateDashboardLayout.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/api/controllers/dashboard/updateDashboardLayout.ts)
   - 策略：**全量替换** —— 先 `deleteMany` 删除该用户所有记录，再 `createMany` 批量创建启用的区块（事务包裹）

3. 前端 Hook：[useUpdateDashboardLayout](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/dashboardData.tsx#L37-L83)
   - `onMutate`: 乐观更新 React Query 缓存中的 `user.dashboardSections`
   - `onError`: 回滚到之前数据
   - `onSuccess`: 失效 `["user"]` 和 `["dashboardData"]` 缓存

### 3.2 列表视图布局（本地存储）

列表视图相关设置 **不与服务端同步**，仅保存在当前设备的 localStorage。

**状态管理 Store**：[localSettings.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/store/localSettings.ts)

```typescript
type LocalSettings = {
  viewMode: string;      // "card" | "masonry" | "list"，默认 "card"
  color: string;         // 主题色变量，默认 "--default"
  show: {                // 各字段显示开关，全部默认 true
    link: boolean;
    name: boolean;
    description: boolean;
    image: boolean;
    tags: boolean;
    icon: boolean;
    collection: boolean;
    preserved_formats: boolean;
    date: boolean;
  };
  columns: number;       // 列数，0=自适应（默认），1-8=固定
  sortBy?: Sort;         // 排序方式，默认 DateNewestFirst
};
```

**初始化时机**：[useInitialData.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/hooks/useInitialData.tsx)
- 在 Session 状态变化时调用 `setSettings()`
- 从 localStorage 读取各字段，缺失则填默认值

**视图切换组件**：[ViewDropdown.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/components/ViewDropdown.tsx)
- 三种视图模式：Card（卡片）/ Masonry（瀑布流）/ List（列表）
- Dashboard 页面只支持 Card 视图的显示项配置
- List 视图不支持 tags/image/description 显示配置
- 列数滑块仅在非 List 视图下显示

**视图渲染组件**：[Links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/components/LinkViews/Links.tsx)

列数自适应规则（当 `columns === 0` 时）：

| 窗口宽度 | 列数 |
|----------|------|
| ≥ 1501px | 5 |
| ≥ 881px | 4 |
| ≥ 701px | 3 |
| ≥ 501px | 2 |
| < 501px | 1 |

---

## 四、通知设置

### 4.1 版本公告通知

公告系统完全基于 **localStorage**，不涉及服务端用户设置同步，三个 localStorage 键的读写分布在三个文件中。

**localStorage 键与读写责任矩阵**：

| 键名 | 写入位置 | 读取位置 | 说明 |
|------|----------|----------|------|
| `announcementId` | [getLatestVersion.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/client/getLatestVersion.ts#L19-L20) | [getLatestVersion.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/client/getLatestVersion.ts#L2)、[Announcement.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/components/Announcement.tsx#L11) | 最新公告版本号，如 "2.15.0" |
| `announcementMessage` | [getLatestVersion.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/client/getLatestVersion.ts#L21-L22) | [getLatestVersion.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/client/getLatestVersion.ts#L3)、[Announcement.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/components/Announcement.tsx#L12) | 自定义公告消息 i18n key，优先级低于 announcementId |
| `showAnnouncementBar` | [MainLayout.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/layouts/MainLayout.tsx#L28-L33) | [MainLayout.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/layouts/MainLayout.tsx#L14) | "true"/"false"，公告栏显示/隐藏状态 |

**完整读写流程**：

1. **初始化读取（MainLayout）**：[MainLayout.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/layouts/MainLayout.tsx#L14-L22)
   - 组件挂载时读取 `showAnnouncementBar`，用于初始化 `showAnnouncement` state（无值则默认显示）
   - 同时读取 `sidebarIsCollapsed`（非公告相关，但共用同一套写入模式）

2. **版本检查 + 写入公告版本/消息（getLatestVersion）**：[getLatestVersion.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/client/getLatestVersion.ts#L1-L24)
   - 先读取本地 `announcementId` 和 `announcementMessage`
   - 请求远程 `https://linkwarden.app/blog/latest-announcement.json` 获取最新版本
   - 若远程版本或消息与本地不一致：
     - 调用 `setShowAnnouncement(true)` 触发显示公告栏
     - **写入 `announcementId`**：`localStorage.setItem("announcementId", latestAnnouncement)`
     - **写入 `announcementMessage`**：`localStorage.setItem("announcementMessage", latestMessage)`

3. **公告栏显示状态持久化（MainLayout useEffect）**：[MainLayout.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/layouts/MainLayout.tsx#L28-L33)
   - 监听 `showAnnouncement` state 变化
   - 每次变化（包括用户点击关闭按钮触发的 `toggleAnnouncementBar`）都会**写入 `showAnnouncementBar`**

4. **展示组件（Announcement，只读）**：[Announcement.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/components/Announcement.tsx#L11-L44)
   - 读取 `announcementId`（优先，显示为带版本号链接的新版公告）
   - 若无 `announcementId` 则回退读取 `announcementMessage`（显示为自定义 i18n 消息）
   - 关闭按钮通过 props 调用 `toggleAnnouncementBar`，由 MainLayout 的 useEffect 间接写入 localStorage

### 4.2 促销邮件通知偏好

促销邮件偏好字段为 `acceptPromotionalEmails`，默认值 `false`。该字段的生命周期存在设计缺口——**仅在注册时可设置，注册后无法通过任何官方 UI 修改**。

#### 完整链路梳理：

**1. 注册阶段（可设置）**

- 注册表单 UI：[register.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/register.tsx#L256-L292)
  - 显示条件：`process.env.NEXT_PUBLIC_STRIPE` 为 true（即启用 Stripe 付费模式时才显示）
  - 默认状态：`acceptPromotionalEmails: false`
  - 控件类型：Checkbox，用户可主动勾选

- Schema 校验：[schemaValidation.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/lib/schemaValidation.ts#L36-L58)
  - `PostUserSchema` 中明确定义：`acceptPromotionalEmails: z.boolean().default(false)`

- 数据入库：[postUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/api/controllers/users/postUser.ts#L84-L132)
  ```typescript
  acceptPromotionalEmails: acceptPromotionalEmails || false,
  ```
  注册时与 `dashboardSections` 默认布局一同写入数据库。

**2. 账号设置阶段（不可修改）**

- 账号设置页面：[account.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/settings/account.tsx)
  - UI 层面：**完全没有** 促销邮件偏好的 Checkbox 或其他控件
  - 提交 payload：仅包含 `id, name, username, email, locale, image, isPrivate, password`
  - 变更检测 `hasAccountChanges` 也不比较该字段

- Schema 校验：[schemaValidation.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/lib/schemaValidation.ts#L60-L95)
  - `UpdateUserSchema` 中 **未包含** `acceptPromotionalEmails` 字段（与 `PostUserSchema` 不对称）
  - 即使通过 API 直接调用也会被 Zod Schema 过滤掉

**3. 偏好设置页面（不可修改）**

- 偏好设置页面 [preference.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/settings/preference.tsx) 的四个区块（主题/AI/归档/链接）均不涉及该字段。

**结论：** 用户一旦注册完成，除非直接操作数据库，否则没有任何官方途径可以修改促销邮件订阅状态。

---

## 五、多设备同步

### 5.1 访问令牌机制

多设备通过 **AccessToken** 模型进行认证授权：

**数据模型**：[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/prisma/schema.prisma#L234-L246)

```prisma
model AccessToken {
  token      String    @unique
  userId     Int
  name       String
  isSession  Boolean   @default(false)   // true=登录会话, false=手动创建的API令牌
  revoked    Boolean   @default(false)
  expires    DateTime
  lastUsedAt DateTime?
}
```

**Web 端管理页面**：[access-tokens.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/settings/access-tokens.tsx)
- 展示所有令牌，`isSession=true` 的行显示为高亮并带 Tooltip "Permanent Session"
- 支持创建新令牌、撤销令牌

**会话创建 API**：`POST /api/v1/session`
- 控制器：[createSession.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/api/controllers/session/createSession.ts)

### 5.2 移动端认证与数据同步

移动端使用独立的 Zustand Store 管理认证状态：

**认证 Store**：[auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/mobile/store/auth.ts)

存储方式：`expo-secure-store`（加密存储）
- `TOKEN`：访问令牌
- `INSTANCE`：服务器实例地址（支持自托管）

登录流程：
1. `POST {instance}/api/v1/session` 提交用户名密码
2. 返回 token 后存入 SecureStore
3. 后续所有 API 请求携带 `Authorization: Bearer {token}`

登出流程（完整清理）：
```typescript
signOut: async () => {
  await SecureStore.deleteItemAsync("TOKEN");
  await SecureStore.deleteItemAsync("INSTANCE");
  queryClient.cancelQueries();
  queryClient.clear();           // 清空 React Query 缓存
  mmkvPersister.removeClient?.(); // 清空持久化缓存
  await clearCache();             // 删除本地归档文件缓存
}
```

### 5.3 移动端本地数据设置

移动端独立的本地设置 Store：[data.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/mobile/store/data.ts)

```typescript
interface MobileData {
  shareIntent: { hasShareIntent: boolean; url: string };  // 不持久化
  theme: "light" | "dark" | "system";                     // 默认 "system"
  preferredBrowser: "app" | "system";                     // 默认 "app"
  preferredCollection: Collection | null;                 // 默认 null
}
```

存储方式：`@react-native-async-storage/async-storage`（非加密）
- `shareIntent` 字段 **不持久化**（仅运行时使用）
- 其他字段持久化到本地

**移动端设置页面**：[settings/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/mobile/app/(tabs)/settings/index.tsx)

可配置项：
- 主题：System / Light / Dark（使用 nativewind 的 `setColorScheme`）
- 首选浏览器：App 内浏览器 / 系统默认浏览器
- 共享链接保存到的默认收藏夹

首选收藏夹页面：[preferredCollection.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/mobile/app/(tabs)/settings/preferredCollection.tsx)

### 5.4 归档缓存管理（移动端）

移动端缓存工具：[cache.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/mobile/lib/cache.ts)

```typescript
clearCache()        // 删除 archivedData 和 mmkv 目录
deleteLinkCache()   // 删除单个链接的所有归档格式文件
```

---

## 六、本地化（i18n）实现

### 6.1 支持的语言

配置文件：[next-i18next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/next-i18next.config.js)

```javascript
locales: [
  "en", "it", "fr", "zh", "zh-TW", "uk", "pt-BR",
  "ja", "es", "de", "nl", "tr", "pl", "ru"
]
defaultLocale: "en"
```

翻译文件目录：`apps/web/public/locales/{lang}/common.json`

### 6.2 语言偏好存储与应用

**存储位置**：User 模型的 `locale` 字段，默认 `"en"`。

**应用位置**：[updateUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserById.ts#L204)

```typescript
locale: i18n.locales.includes(data.locale || "") ? data.locale : "en",
```
白名单校验：不在支持列表中的语言自动回退到 `"en"`。

**Web 端初始化**：
- `_app.tsx` 通过 `appWithTranslation(App)` 包裹
- [useUser](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/user.tsx#L46) Hook 获取用户数据后设置 `data-theme`，但语言切换由 next-i18next 的浏览器语言检测处理

### 6.3 日期格式化中的本地化应用

示例见 [access-tokens.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/settings/access-tokens.tsx#L85-L98)：

```typescript
new Date(token.createdAt).toLocaleDateString(t("locale"), {
  month: "short", day: "numeric", year: "numeric",
})
```
翻译 key `locale` 会返回对应语言的 BCP 47 标签（如 `"zh-CN"`）。

---

## 七、缓存更新机制

### 7.1 Web 端 React Query 缓存

配置位置：[_app.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/_app.tsx#L18-L24)

```typescript
new QueryClient({
  defaultOptions: {
    queries: { staleTime: 1000 * 30 },  // 30秒新鲜期
  },
});
```

核心缓存键：

| Cache Key | 数据 | 失效/更新时机 |
|-----------|------|--------------|
| `["user"]` | 当前用户完整信息（含偏好、dashboardSections） | `useUpdateUser`, `useUpdateUserPreference`, `useUpdateDashboardLayout` 成功后 |
| `["dashboardData"]` | Dashboard 聚合数据 | `useUpdateDashboardLayout` 成功后 |
| `["tokens"]` | 访问令牌列表 | 创建/撤销令牌后 |

### 7.2 用户偏好写入缓存行为全景

并非所有写操作都完整实现了「乐观覆盖 → 失败回滚 → 失效刷新」三段式。以下是与用户设置直接相关的所有 Mutation 的详细行为对比：

| Mutation | 所在文件 | 乐观覆盖 (onMutate) | 失败回滚 (onError) | 失效刷新 / 权威更新 (onSuccess) | 备注 |
|----------|----------|---------------------|---------------------|--------------------------------|------|
| `useUpdateUser` | [user.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/user.tsx#L57-L84) | ✅ 直接设置 `["user"]` 缓存 | ❌ **未实现** | ✅ `setQueryData(["user"])` 用服务端数据覆盖 | 无失败回滚，出错后 UI 停留在乐观数据 |
| `useUpdateUserPreference` | [user.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/user.tsx#L86-L126) | ✅ 直接设置 `["user"]` 缓存 | ❌ **未实现** | ✅ `setQueryData(["user"])` + 设置 `data-theme` DOM | 同 `useUpdateUser`，无失败回滚 |
| `useUpdateDashboardLayout` | [dashboardData.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/dashboardData.tsx#L37-L83) | ✅ 保存 `previousData` 快照并设置 `["user"]` | ✅ `setQueryData(["user"], context.previousData)` | ✅ `invalidateQueries(["user", "dashboardData"])` | **唯一完整实现三段式** 的用户设置 Mutation |
| `useAddToken` | [tokens.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/tokens.tsx#L22-L47) | ❌ **未实现** | ❌ **未实现** | ✅ `setQueryData(["tokens"])` 追加新令牌 | 纯被动更新，请求成功前列表不变 |
| `useRevokeToken` | [tokens.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/tokens.tsx#L49-L69) | ❌ **未实现** | ❌ **未实现** | ✅ `setQueryData(["tokens"])` 过滤已撤销 | 无乐观删除，请求失败时令牌始终留在列表中，无需"还原" |
| `useUpsertTags` (归档标签，偏好设置关联) | [tags.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/tags.tsx#L322-L345) | ❌ **未实现** | ❌ **未实现** | ✅ `invalidateQueries(["tags", "dashboardData"])` | 等待服务端响应后才刷新 UI |

#### 风险点总结（聚焦用户设置与通知偏好相关缓存）：

1. **`useUpdateUser` / `useUpdateUserPreference` 无失败回滚（影响：本地化语言、归档偏好、AI 设置、链接跳转设置、主题、可读性设置）**：
   - 请求发出前已通过 `onMutate` 将新值乐观写入 `["user"]` 缓存
   - 如果 API 请求失败（网络错误、权限校验失败等），**没有 `onError` 回调将缓存恢复为请求前的数据**
   - 用户看到的 UI 与数据库实际持久化状态不一致，必须手动刷新页面才能恢复正确数据

2. **`useUpsertTags` 无乐观更新（影响：偏好设置中的归档标签规则）**：
   - 用户在偏好设置页面修改各标签的归档规则后，需等待网络请求完成才会反映到 UI
   - 网络响应慢时用户可能误以为点击无效而重复点击保存

3. **通知偏好 `acceptPromotionalEmails` 无修改路径（缓存层不存在更新风险，但存在功能缺口）**：
   - 见 4.2 节，该字段注册后无法通过任何官方途径修改，其缓存值始终等于注册时写入数据库的值
   - 不存在缓存与数据库不一致的风险，但存在用户无法退订促销邮件的合规风险

4. **公告通知（localStorage 层）存在写入行为，但无异步缓存一致性风险**：
   - 公告系统的三处写入均为同步 localStorage 操作，不涉及服务端用户设置同步，不存在网络失败导致的缓存不一致
   - `getLatestVersion` 在检测到新版本时会**写入 `announcementId` 和 `announcementMessage`**（见 4.1 节读写责任矩阵）
   - `MainLayout` 的 useEffect 监听 `showAnnouncement` state，用户每次关闭/触发显示公告栏都会**写入 `showAnnouncementBar`**
   - 由于 localStorage 写入是同步的 DOM API，没有异步回滚需求，不存在 React Query 层那种乐观更新失败的问题

#### 乐观更新标准三段式（以 `useUpdateDashboardLayout` 为范本）：

```typescript
// 第 1 段：onMutate —— 请求发出前立即乐观更新
onMutate: async (newData) => {
  await queryClient.cancelQueries({ queryKey: ["user"] });      // 取消进行中的请求避免覆盖
  const previousData = queryClient.getQueryData(["user"]);      // 保存快照用于回滚
  queryClient.setQueryData(["user"], (oldData: any) => ({
    ...oldData,
    dashboardSections: newData.filter(s => s.enabled).sort(/*...*/),
  }));
  return { previousData };                                      // 传给 onError 使用
},

// 第 2 段：onError —— 请求失败时回滚
onError: (err, newData, context) => {
  queryClient.setQueryData(["user"], context?.previousData);   // 恢复快照
},

// 第 3 段：onSuccess —— 请求成功后用权威数据刷新
onSuccess: async () => {
  await queryClient.invalidateQueries({ queryKey: ["user"] });          // 失效后自动重新拉取
  await queryClient.invalidateQueries({ queryKey: ["dashboardData"] });
},
```

### 7.3 本地 localStorage 即时同步

`localSettings` Store 的 `updateSettings` 方法采用 **双写策略**：
- 立即写入 localStorage
- 同步更新 Zustand state
- 主题色变化时还立即修改 CSS 变量：
  ```typescript
  document.documentElement.style.setProperty("--p", `var(${color})`);
  ```

### 7.4 移动端持久化

移动端使用三层持久化方案：

| 层级 | 技术 | 存储内容 |
|------|------|----------|
| 加密层 | expo-secure-store | TOKEN, INSTANCE |
| KV 层 | @react-native-async-storage/async-storage | 主题、首选浏览器、首选收藏夹 |
| 文件层 | expo-file-system | 网页归档缓存（PDF/HTML/图片等） |
| 查询缓存 | mmkv + react-query persister | API 响应缓存 |

登出时所有层级均被完整清理。

---

## 八、完整数据流转路径图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户设置操作入口                              │
│  [settings/preference.tsx]  [DashboardLayoutDropdown]  [ViewDropdown]│
└──────────────┬──────────────────────────┬────────────────────────────┘
               │                          │
    ┌──────────▼──────────┐    ┌──────────▼───────────┐
    │  Server Mutations   │    │  Local Store Only    │
    │  (React Query)      │    │  (Zustand)           │
    │  useUpdateUser      │    │  useLocalSettingsStore│
    │  useUpdateUserPref  │    │  useDataStore (移动)  │
    │  useUpdateDashLayout│    └──────────┬───────────┘
    └──────────┬──────────┘               │
               │                          │
    ┌──────────▼──────────┐    ┌──────────▼───────────┐
    │  API Controllers    │    │  localStorage        │
    │  updateUserById     │    │  AsyncStorage        │
    │  updateUserPref     │    │  SecureStore         │
    │  updateDashLayout   │    │  (仅当前设备)         │
    └──────────┬──────────┘    └──────────────────────┘
               │
    ┌──────────▼──────────┐
    │  PostgreSQL         │
    │  User Table         │
    │  DashboardSection   │
    │  AccessToken        │
    │  (跨所有设备同步)    │
    └─────────────────────┘
```

---

## 九、关键文件索引

| 文件路径 | 核心职责 |
|----------|----------|
| [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/prisma/schema.prisma) | 数据模型定义（User, DashboardSection, AccessToken, Tag 等） |
| [global.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/types/global.ts) | 全局类型定义（ViewMode, Sort, MobileData, AccountSettings 等） |
| [preference.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/settings/preference.tsx) | Web 端偏好设置页面 |
| [updateUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserById.ts) | 用户信息更新控制器（含本地化、归档、链接设置） |
| [updateUserPreference.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserPreference.ts) | 主题/可读性偏好更新控制器 |
| [localSettings.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/store/localSettings.ts) | Web 端本地设置 Zustand Store |
| [ViewDropdown.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/components/ViewDropdown.tsx) | 视图模式/列数/显示项 下拉组件 |
| [Links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/components/LinkViews/Links.tsx) | 链接列表渲染（Card/Masonry/List 三种视图） |
| [DashboardLayoutDropdown.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/components/DashboardLayoutDropdown.tsx) | Dashboard 布局编辑下拉组件 |
| [updateDashboardLayout.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/api/controllers/dashboard/updateDashboardLayout.ts) | Dashboard 布局更新控制器 |
| [user.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/user.tsx) | useUser/useUpdateUser/useUpdateUserPreference Hooks |
| [dashboardData.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/dashboardData.tsx) | useDashboardData/useUpdateDashboardLayout Hooks |
| [Announcement.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/components/Announcement.tsx) | 公告通知组件 |
| [getLatestVersion.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/client/getLatestVersion.ts) | 最新版本/公告检查 |
| [MainLayout.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/layouts/MainLayout.tsx) | 主布局（公告显示、侧边栏折叠状态） |
| [next-i18next.config.js](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/next-i18next.config.js) | i18n 语言配置 |
| [_app.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/_app.tsx) | App 根组件（QueryClient, i18n, SessionProvider） |
| [auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/mobile/store/auth.ts) | 移动端认证 Store |
| [data.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/mobile/store/data.ts) | 移动端本地设置 Store |
| [cache.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/mobile/lib/cache.ts) | 移动端缓存清理工具 |
| [settings/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/mobile/app/(tabs)/settings/index.tsx) | 移动端设置页面 |
| [access-tokens.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/settings/access-tokens.tsx) | Web 端访问令牌管理页面 |
| [register.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/register.tsx) | 注册页面（促销邮件偏好唯一入口） |
| [account.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/pages/settings/account.tsx) | 账号设置页面（不含促销邮件偏好修改） |
| [postUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/api/controllers/users/postUser.ts) | 用户创建控制器（写入 acceptPromotionalEmails + 默认 DashboardSections） |
| [schemaValidation.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/lib/schemaValidation.ts) | Zod Schema 校验（PostUserSchema 含 acceptPromotionalEmails，UpdateUserSchema 不含） |
| [tokens.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/tokens.tsx) | useTokens/useAddToken/useRevokeToken Hooks |
| [tags.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/tags.tsx) | useUpsertTags 等标签相关 Hooks |
| [collections.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/packages/router/collections.tsx) | 收藏集 CRUD Hooks（含被注释的 onMutate） |
