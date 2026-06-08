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

公告系统完全基于 **localStorage**，不涉及服务端用户设置。

**检查更新流程**：

1. 触发点：[MainLayout.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/layouts/MainLayout.tsx#L24-L26)
   - 页面加载时调用 `getLatestVersion(setShowAnnouncement)`

2. 版本检查：[getLatestVersion.ts](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/lib/client/getLatestVersion.ts)
   - 请求 `https://linkwarden.app/blog/latest-announcement.json`
   - 对比本地 `announcementId` / `announcementMessage` 与远程
   - 不一致时显示公告栏并更新本地存储

3. 展示组件：[Announcement.tsx](file:///d:/fz/0601/solo-dogfeeding/code/98-linkwarden/apps/web/components/Announcement.tsx)
   - 从 localStorage 读取 `announcementId`（显示版本号）或 `announcementMessage`（显示自定义消息）
   - 使用 `next-i18next` 的 `<Trans>` 组件渲染带链接的多语言文本

**localStorage 中的公告相关键**：
| 键名 | 说明 |
|------|------|
| `showAnnouncementBar` | "true"/"false"，用户是否关闭公告栏 |
| `announcementId` | 最新公告版本号 |
| `announcementMessage` | 公告消息 i18n key |

### 4.2 邮件通知偏好

用户模型中的 `acceptPromotionalEmails` 字段控制是否接收促销邮件，默认 `false`。

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

### 7.2 乐观更新模式

所有写操作均采用 **Optimistic Update** 模式：

以 `useUpdateUserPreference` 为例：
1. `onMutate`: 取消进行中的查询，直接用新数据覆盖缓存（UI 立即响应）
2. `onError`: 回滚到 mutation 前的快照
3. `onSuccess`: 用服务端返回的权威数据再次更新缓存，并设置 `data-theme` DOM 属性

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
