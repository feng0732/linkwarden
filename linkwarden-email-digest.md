# Linkwarden Email Digest 与定期摘要通知处理路径分析

## 一、核心结论

> **当前状态：Linkwarden v2.x 代码库中未实现完整的 "Email Digest / 定期摘要邮件" 功能。**
>
> 系统已具备完整的**邮件发送基础设施**、**周期性 Worker 调度框架**、以及**用户邮件偏好字段**，但不存在针对用户定期发送 Link 收藏摘要的代码路径。现有的周期性邮件通知仅包含 **试用期结束提醒**（`trialEndEmailWorker`）。

---

## 二、邮件发送基础设施

### 2.1 邮件传输层（Transporter）

**文件：** [transporter.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/lib/transporter.ts)

系统使用 Nodemailer 作为邮件发送引擎，基于环境变量构建传输实例：

```typescript
import { createTransport } from "nodemailer";

export default createTransport({
  url: process.env.EMAIL_SERVER,   // SMTP 服务器连接字符串
  auth: {
    user: process.env.EMAIL_FROM,  // 发件人地址
  },
});
```

**前提条件（emailEnabled）：** 仅当同时设置 `EMAIL_FROM` 和 `EMAIL_SERVER` 两个环境变量时，邮件功能才会启用。该开关定义于多个控制器中，例如 [postUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/users/postUser.ts#L8-L10)。

### 2.2 邮件模板体系

模板位于两处，均使用 **Handlebars** 作为模板引擎：

| 位置 | 模板文件 | 用途 |
|------|----------|------|
| `apps/web/templates/` | [verifyEmail.html](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/templates/verifyEmail.html) | 邮箱验证 |
| `apps/web/templates/` | [passwordReset.html](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/templates/passwordReset.html) | 密码重置 |
| `apps/web/templates/` | [acceptInvitation.html](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/templates/acceptInvitation.html) | 团队邀请 |
| `apps/web/templates/` | [verifyEmailChange.html](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/templates/verifyEmailChange.html) | 邮箱变更验证 |
| `apps/worker/templates/` | [trialEnded.html](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/templates/trialEnded.html) | 试用期结束通知 |

**模板加载方式示例**（来自 [trialEndEmailWorker.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/trialEndEmailWorker.ts#L76-L94)）：

```typescript
const emailsDir = path.resolve(process.cwd(), "templates");
const templateFile = readFileSync(
  path.join(emailsDir, "trialEnded.html"), "utf8"
);
const emailTemplate = Handlebars.compile(templateFile);

await transporter.sendMail({
  from, to: user.email,
  subject: "Your Linkwarden trial has ended",
  html: emailTemplate({ name: user.name, url: process.env.BASE_URL }),
});
```

---

## 三、周期性 Worker 调度框架（Digest 发送节奏参考）

### 3.1 Worker 总入口

**文件：** [worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/worker.ts)

Worker 进程通过 [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/index.ts) 以子进程方式启动，崩溃后 5 秒自动重启。主入口 `init()` 顺序启动以下任务：

```typescript
async function init() {
  await migrationWorker();
  startRSSPolling();                          // RSS 源轮询
  linkProcessing(workerIntervalInSeconds);    // 链接归档处理
  autoTagPreservedLinks(workerIntervalInSeconds); // AI 自动打标签
  startIndexing(workerIntervalInSeconds);     // 全文索引
  trialEndEmailWorker();                      // 试用期结束邮件（唯一现存周期性邮件）
}
```

**关键注意：所有 Worker 均为"立即首次执行"模式**——`while (true)` 循环启动后立即执行第一轮逻辑，随后才进入 delay/sleep，不等待首个间隔到达。

### 3.2 现有调度模式对比（已核对）

| Worker | 调度方式 | 间隔/节奏 | 控制变量（默认值） | 文件 |
|--------|----------|-----------|-------------------|------|
| `linkProcessing` | `while(true)` + `delay(interval)` | 每轮处理后等待 | `ARCHIVE_SCRIPT_INTERVAL`（**10 秒**） | [linkProcessing.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/linkProcessing.ts#L11) |
| `autoTagPreservedLinks` | `while(true)` + `delay(interval)` | 每轮处理后等待 | `ARCHIVE_SCRIPT_INTERVAL`（**10 秒**）；若无 AI Provider 则完全不启动 | [autoTagPreservedLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/autoTagPreservedLinks.ts#L19-L24) |
| `startIndexing` | `while(true)` + `delay(interval)` | 每轮处理后等待 | `ARCHIVE_SCRIPT_INTERVAL`（**10 秒**） | 同上 worker.ts |
| `startRSSPolling` | `while(true)` + `delay(pollingIntervalInSeconds)` | 每轮处理后等待 | `NEXT_PUBLIC_RSS_POLLING_INTERVAL_MINUTES`（**60 分钟**） | [rssPolling.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/rssPolling.ts#L8-L37) |
| `trialEndEmailWorker` | `while(true)` + 批处理 + `sleep(pauseMs)` | 每批处理后固定等待 | `batchSize=10`、`pauseMs=30000`（批间 **30 秒**）。不满足启动条件时直接 `return` | [trialEndEmailWorker.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/trialEndEmailWorker.ts#L9-L124) |

### 3.3 `trialEndEmailWorker` —— 唯一现存周期性邮件的完整流程（已核对）

这是实现 Digest Worker 的最直接参考。其处理路径如下：

1. **启动条件门控**（[trialEndEmailWorker.ts#L22-L30](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/trialEndEmailWorker.ts#L22-L30)）：
   - `NEXT_PUBLIC_TRIAL_PERIOD_DAYS > 1`（必须是有限天数的试用）
   - 设置了 `STRIPE_SECRET_KEY`（商用模式）
   - `NEXT_PUBLIC_REQUIRE_CC !== "true"`（无需绑卡试用）
   - **任一项不满足则 Worker 完全不启动**

2. **候选用户筛选查询**（[trialEndEmailWorker.ts#L42-L56](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/trialEndEmailWorker.ts#L42-L56)）：
   ```
   trialEndEmailSent = false
   AND emailVerified IS NOT NULL
   AND createdAt <= cutoff (当前时间 - trialDays)
   AND createdAt >= '2025-09-25'   // 安全上界，避免追溯老用户
   ORDER BY createdAt ASC, take 10
   ```
   同时 include 自身订阅 `subscriptions` 与父订阅 `parentSubscription`（家庭计划）状态。

3. **空批处理**：若候选为空，执行 `sleep(30s)` 后继续下一轮。

4. **用户资格二次过滤**（[trialEndEmailWorker.ts#L68-L70](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/trialEndEmailWorker.ts#L68-L70)）：
   - 已拥有 `subscriptions.active` 或 `parentSubscription.active` → **跳过发送，但仍标记已处理**
   - `user.email` 为空 → 跳过发送，但仍标记已处理

5. **邮件发送与失败回退**（[trialEndEmailWorker.ts#L74-L104](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/trialEndEmailWorker.ts#L74-L104)）：
   - 加载 `trialEnded.html` Handlebars 模板
   - 通过 Nodemailer transporter 发送
   - **发送失败：不加入 processedIds，执行 sleep(30s) 后 continue** —— 留待下一批重试
   - 发送成功：加入 processedIds

6. **状态持久化**（[trialEndEmailWorker.ts#L111-L119](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/trialEndEmailWorker.ts#L111-L119)）：
   - 使用 `updateMany` 将 `processedIds` 批量标记 `trialEndEmailSent = true`

7. **批间休眠**：`sleep(30s)` 后进入下一轮。

---

## 四、用户偏好与过滤字段（已核对）

### 4.1 User 模型相关字段

**文件：** [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/prisma/schema.prisma#L28-L75)

```prisma
model User {
  // ...
  email                   String?               @unique
  emailVerified           DateTime?
  acceptPromotionalEmails Boolean               @default(false)
  lastPickedAt            DateTime?
  trialEndEmailSent       Boolean               @default(false)
  // ...
}
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `acceptPromotionalEmails` | Boolean | `false` | 注册时勾选是否接收推广/功能更新邮件 |
| `emailVerified` | DateTime? | `null` | 邮箱是否已验证。**所有周期性邮件均应过滤此项** |
| `lastPickedAt` | DateTime? | 迁移后无默认 | 用于 Worker 公平调度（见 4.3），非邮件专用 |
| `trialEndEmailSent` | Boolean | `false` | 试用期结束邮件是否已发送（幂等标记） |

### 4.2 `acceptPromotionalEmails` 的完整流转（已核对）

1. **注册时采集**：
   - **UI 条件门控**：[register.tsx#L256-L292](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/register.tsx#L256-L292) —— 仅当 `process.env.NEXT_PUBLIC_STRIPE` 存在时（商用模式）才展示 Checkbox
   - Schema 校验：[schemaValidation.ts#L56](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/lib/schemaValidation.ts#L56) —— `z.boolean().default(false)`
   - 持久化：[postUser.ts#L113](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/users/postUser.ts#L113) —— `acceptPromotionalEmails || false`（null/undefined 一律视为 false）

2. **账户设置页缺失（关键发现）**：
   - [account.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/settings/account.tsx) 和 [preference.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/settings/preference.tsx) 中**均未提供修改该偏好的入口**
   - `UpdateUserSchema`（[schemaValidation.ts#L60-L95](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/lib/schemaValidation.ts#L60-L95)）中**未包含该字段的更新定义**
   - `UpdateUserPreferenceSchema`（[schemaValidation.ts#L97-L113](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/lib/schemaValidation.ts#L97-L113)）中**也未包含该字段**
   - **结论：用户一旦注册就无法在前端界面更改此偏好**

3. **注销时读取**：
   - [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L139-L153) —— 用户注销时将该偏好作为取消原因邮件的一部分发送给 `hello@linkwarden.app`

### 4.3 `lastPickedAt` —— 公平调度机制（非邮件但可借鉴）

**文件：** [getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/lib/getLinkBatchFairly.ts)

该字段用于 Worker 在多用户之间公平分配链接处理资源：

```typescript
const users = await prisma.user.findMany({
  // ...
  orderBy: [{ lastPickedAt: { sort: "asc", nulls: "first" } }, { id: "asc" }],
  select: { id: true, lastPickedAt: true },
  take: maxBatchLinks,
});
// ... 处理完成后更新：
await prisma.user.updateMany({
  where: { id: { in: uniqueUsersWithLinks } },
  data: { lastPickedAt: now },
});
```

Digest 场景可借鉴此模式实现"最久未收到摘要的用户优先"。

---

## 五、Dashboard 数据聚合（摘要数据收集的潜在来源）—— 深度分析

### 5.1 Dashboard 分区开关（dashboardSections）行为详解

#### 5.1.1 数据模型

`DashboardSection` 表结构（来自 schema.prisma）：
```prisma
model DashboardSection {
  id           Int                   @id @default(autoincrement())
  user         User                  @relation(fields: [userId], references: [id], onDelete: Cascade)
  userId       Int
  type         DashboardSectionType  // STATS | RECENT_LINKS | PINNED_LINKS | COLLECTION
  collectionId Int?                  // 仅 COLLECTION 类型有值
  order        Int
}
```

#### 5.1.2 默认分区创建

新用户注册时（[postUser.ts#L114-L131](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/users/postUser.ts#L114-L131)），系统自动创建 3 个默认分区：
```
order: 0  →  STATS（统计卡片）
order: 1  →  RECENT_LINKS（最近链接）
order: 2  →  PINNED_LINKS（固定链接）
```

Seed 脚本（[seed.js#L35-L52](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/prisma/seed.js#L35-L52)）中也使用完全相同的默认配置。

#### 5.1.3 用户编辑分区流程

用户通过 [DashboardLayoutDropdown.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/components/DashboardLayoutDropdown.tsx) 编辑分区：

1. **可选分区列表**：3 个固定类型（STATS、RECENT_LINKS、PINNED_LINKS）+ 用户所有可访问的 Collection（每个 Collection 对应一个 COLLECTION 类型分区）
2. **分区开关**：Checkbox 切换 `enabled` 状态。启用时自动分配 `order = 当前最大 order + 1`
3. **分区排序**：拖拽（dnd-kit）仅限启用分区之间重排，自动重写 order 值
4. **提交**：调用 `/api/v2/dashboard` PUT，发送**全量分区列表**（含启用+禁用）

#### 5.1.4 服务端更新逻辑

[updateDashboardLayout.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/dashboard/updateDashboardLayout.ts)：

```
步骤 1: Schema 校验（UpdateDashboardLayoutSchema）
步骤 2: 提取所有 collectionId，校验用户是否有权限访问（owner 或 member）
        无权限的 collectionId 对应的分区将被过滤掉
步骤 3: 事务处理：
        3a. deleteMany({ where: { userId } }) —— 删除该用户所有现有分区
        3b. 仅保留 enabled=true 的分区，createMany 写入
步骤 4: 调用 getDashboardDataV2 返回最新 Dashboard 数据
```

**关键行为：禁用的分区在 DB 中完全不存在，而不是有一个 enabled=false 的记录。** 因此 "某分区是否启用" = "该分区记录是否存在"。

### 5.2 Dashboard V2 数据聚合查询详解

**文件：** [getDashboardDataV2.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts)

#### 5.2.1 前置：并行加载基础数据（第 23-41 行）

```typescript
const [dashboardSections, numberOfPinnedLinks, numberOfTags] =
  await Promise.all([
    prisma.dashboardSection.findMany({ where: { userId } }),
    prisma.link.count({ ... }),   // 用户 pinned 的链接总数
    prisma.tag.count({ ... }),    // 用户可触及的标签总数
  ]);
```

#### 5.2.2 分区存在性检查与短路返回（第 43-60 行）

```typescript
const viewPinned = dashboardSections.some(s => s.type === "PINNED_LINKS");
const viewRecent = dashboardSections.some(s => s.type === "RECENT_LINKS");
const collectionSections = dashboardSections.filter(s => s.type === "COLLECTION");

if (!viewRecent && !viewPinned && collectionSections.length === 0) {
  return { data: { links: [], numberOfPinnedLinks, numberOfTags }, ... };
}
```

**若用户关闭了 RECENT_LINKS、PINNED_LINKS，且未添加任何 Collection 分区，则直接返回空链接列表。**

#### 5.2.3 条件化并行查询（第 62-147 行）

根据分区存在性决定是否发起对应查询：

| 分区 | 查询条件 | 返回字段 | 数量限制 |
|------|----------|----------|----------|
| PINNED_LINKS | `viewPinned=true` 时才查询 | `pinnedLinks`（含 tags, collection, pinnedBy） | **take: 16** |
| RECENT_LINKS | `viewRecent=true` 时才查询 | `recentlyAddedLinks`（含 tags, collection, pinnedBy） | **take: 16** |
| COLLECTION × N | 每个 collectionSections 条目一个查询 | `collectionsResult`（每条含 colId + links） | **每个收藏夹 take: 16** |

所有 Link 查询的公共权限过滤条件：
```
collection.ownerId = userId
OR collection.members.some(m => m.userId = userId)
```

所有 Link 查询的 `orderBy`：`{ id: "desc" }`（即按创建时间倒序）。

#### 5.2.4 Link 查询 include/omit 与字段返回分析（已核对）

所有 Link 查询使用统一的 Prisma 查询参数（[getDashboardDataV2.ts#L67-L82](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L67-L82)、[#L97-L112](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L97-L112)、[#L131-L146](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts#L131-L146)）：

```typescript
omit: { textContent: true },
include: {
  tags: true,
  collection: true,
  pinnedBy: {
    where: { id: userId },
    select: { id: true },
  },
},
```

**Prisma 查询行为核心规则：未被 `omit` 的标量字段默认全部返回。** 因此：

| 类别 | 处理方式 |
|------|----------|
| **被 omit 排除** | 仅 `textContent`（全文索引大字段，用于搜索，不展示） |
| **标量字段（默认返回）** | id, name, type, description, icon, iconWeight, color, url, preview, image, pdf, readable, monolith, clientSide, aiTagged, metaDescription, indexVersion, lastPreserved, importDate, createdAt, updatedAt, collectionId, createdById |
| **关系字段（include 返回）** | tags（完整 Tag 对象）、collection（完整 Collection 对象）、pinnedBy（仅当前用户的 { id }） |

**前端 TypeScript 类型约束**（[global.ts#L14-L34](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/types/global.ts#L14-L34)）`LinkIncludingShortenedCollectionAndTags` 将 `id/createdAt/collectionId/updatedAt/lastPreserved/importDate` 设为可选，但 DB 层实际可空性不一致：`id/createdAt/collectionId/updatedAt` 非空，`lastPreserved/importDate` 可空——详见 5.2.5.1。

#### 5.2.5 Link 标量字段完整清单与邮件摘要适用性

基于 Prisma `Link` 模型（[schema.prisma#L166-L198](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/prisma/schema.prisma#L166-L198)）、Dashboard 返回字段、以及前端 Card 组件实际消费字段（[DashboardLinks.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/components/DashboardLinks.tsx)、[LinkCard.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkCard.tsx)），整理如下：

| 字段 | 分类 | Prisma 类型 | DB 可空性 | Dashboard 是否返回 | 邮件摘要用途 | 前端 Card 是否使用 |
|------|------|------------|:---------:|:------------------:|-------------|:------------------:|
| `id` | 主键 | Int @default(autoincrement()) | ❌ 非空 | ✅ | 生成详情链接、去重 | ✅（拖拽 key、路由跳转） |
| `name` | 业务元数据 | String @default("") | ❌ 非空 | ✅ | **链接标题（摘要核心）** | ✅（卡片主标题，line-clamp-2） |
| `description` | 业务元数据 | String @default("") | ❌ 非空 | ✅ | **用户自定义描述** | ❌（Dashboard Card 不显示） |
| `metaDescription` | Worker 抓取元数据 | String? | ✅ 可空 | ✅ | **页面元描述（归档 Worker 抓取所得）** | ❌ |
| `url` | **源链接（非归档）** | String? | ✅ 可空 | ✅ | **跳转原始链接（源链接，非归档产物）** | ✅（LinkTypeBadge 显示域名、点击跳转） |
| `type` | 业务元数据 | String @default("url") | ❌ 非空 | ✅ | 链接类型徽标（url/pdf/image/monolith） | ✅（LinkTypeBadge） |
| `preview` | **归档产物** | String? | ✅ 可空 | ✅ | **缩略图（归档 Worker 生成的页面预览图）** | ✅（Image 组件 + 轮询 refetch 判断） |
| `image` | **归档产物** | String? | ✅ 可空 | ✅ | 整页截图（归档产物） | ✅（formatStats.formatAvailable） |
| `pdf` | **归档产物** | String? | ✅ 可空 | ✅ | PDF 归档（归档产物） | ✅（同上） |
| `readable` | **归档产物** | String? | ✅ 可空 | ✅ | Readability 正文提取（归档产物） | ✅（同上） |
| `monolith` | **归档产物** | String? | ✅ 可空 | ✅ | Monolith 单页 HTML（归档产物） | ✅（同上） |
| `icon` | 业务元数据 | String? | ✅ 可空 | ✅ | Phosphor 图标名称（用户自定义图标） | ✅（LinkIcon） |
| `iconWeight` | 业务元数据 | String? | ✅ 可空 | ✅ | 图标粗细 | ✅（LinkIcon） |
| `color` | 业务元数据 | String? | ✅ 可空 | ✅ | 图标颜色 | ✅（LinkIcon） |
| `createdAt` | 时间戳 | DateTime @default(now()) | ❌ 非空 | ✅ | **创建时间（摘要排序+展示）** | ✅（LinkDate，回退用） |
| `importDate` | 时间戳 | DateTime? | ✅ 可空 | ✅ | 导入时间（优先于 createdAt 展示） | ✅（LinkDate，优先用） |
| `updatedAt` | 时间戳 | DateTime @default(now()) @updatedAt | ❌ 非空 | ✅ | 缩略图 URL 缓存刷新参数 | ✅（Image src query） |
| `lastPreserved` | 时间戳 | DateTime? | ✅ 可空 | ✅ | 上次归档成功时间（可选展示） | ❌ |
| `collectionId` | 外键 | Int | ❌ 非空 | ✅ | 关联收藏夹 | ✅（匹配 Collection 对象） |
| `createdById` | 外键 | Int? | ✅ 可空 | ✅ | 创建者 ID（可选展示） | ❌ |
| `clientSide` | 归档标记 | Boolean @default(false) | ❌ 非空 | ✅ | 是否客户端归档（邮件可忽略） | ❌ |
| `aiTagged` | 归档标记 | Boolean @default(false) | ❌ 非空 | ✅ | 是否已 AI 打标签（可选展示） | ❌ |
| `indexVersion` | 归档标记 | Int? | ✅ 可空 | ✅ | 全文索引版本（邮件可忽略） | ❌ |
| `textContent` | 归档产物 | String? | ✅ 可空 | ❌（被 omit） | 无需（邮件不展示全文） | ❌ |
| — **关系字段** — | — | — | — | — | — | — |
| `tags` | 关系 | Tag[] | — | ✅（include） | **标签展示** | ⚠️（Card 未直接渲染，但数据返回） |
| `collection` | 关系 | Collection | — | ✅（include） | **所属收藏夹名称+颜色** | ✅（LinkCollection 组件） |
| `pinnedBy` | 关系 | { id: number }[] | — | ✅（include，仅当前用户） | **固定状态标识** | ✅（LinkPin 组件 + 前端过滤 pinned） |

**邮件摘要核心可复用字段（15 个，全部 Dashboard 已返回）：**
`id`、`name`、`description`、`metaDescription`、`url`、`preview`、`image`、`type`、`createdAt`、`importDate`、`lastPreserved`、`tags`、`collection.name`、`collection.color`、`pinnedBy`

**特别修正：之前评估认为 description 可能不含——实际 Prisma 未 omit 该字段，description 和 metaDescription 均完整返回。**

##### 5.2.5.1 关键字段可空性与值来源详解

###### `createdAt` / `updatedAt` —— 非空，Prisma 自动维护

| 属性 | 值 |
|------|---|
| DB 可空性 | ❌ **非空**（`@default(now())`） |
| 值来源 | Prisma ORM 自动赋值 |
| 特殊行为 | `updatedAt` 带 `@updatedAt`，每次更新行自动刷新 |
| 空值风险 | **无**——每条 Link 必存在 |

###### `description` —— 非空但可能为空字符串

| 属性 | 值 |
|------|---|
| DB 可空性 | ❌ **非空**（`@default("")`） |
| 值来源 | 1. 用户新建 Link 时显式传入（[postLink.ts#L108](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/links/postLink.ts#L108)）<br>2. 导入时从备份文件读取（[importFromLinkwarden.ts#L68](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L68)） |
| 空值形式 | 空字符串 `""`（用户未填写时） |
| 最大长度 | 导入时限制 254 字符（`.slice(0, 254)`） |
| 注意 | Schema 的 `PostLinkSchema` 中 description 为可选（z.optional），但 DB 层 default "" 保证永不 null |

###### `metaDescription` —— 可空，归档 Worker 异步填充

| 属性 | 值 |
|------|---|
| DB 可空性 | ✅ **可空**（`String?`，无 default） |
| 值来源 | **仅由归档 Worker 抓取**——Playwright 访问页面后读取 `<meta name="description">` 内容（[archiveHandler.ts#L151-L164](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/lib/archiveHandler.ts#L151-L164)） |
| 空值场景 | 1. Link 创建后尚未被 Worker 处理<br>2. 页面无 `<meta name="description">` 标签<br>3. 非 URL 类型 Link（pdf/image 等不执行抓取）<br>4. URL 无法被服务端访问（标记 skipPreservation） |
| 值处理 | 抓取后 `.trim().slice(0, 500)`——最多 500 字符 |
| 典型占比 | 非 URL 类型 + 未处理的新 Link 可能有 **20%~50%** 为 null |

###### `importDate` —— 可空，仅导入数据有值

| 属性 | 值 |
|------|---|
| DB 可空性 | ✅ **可空**（`DateTime?`，无 default） |
| 值来源 | 1. Linkwarden 导入：`new Date(link.importDate \|\| link.createdAt)`（[importFromLinkwarden.ts#L69](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L69)）<br>2. Pocket 导入：`new Date(Number(link.time_added) * 1000)`，若 `time_added` 缺失则为 `null`（[importFromPocket.ts#L70-L72](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/migration/importFromPocket.ts#L70-L72)） |
| 空值场景 | **所有手动创建的 Link 均为 null**（[postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/links/postLink.ts) 创建时未设置该字段） |
| 前端展示逻辑 | [LinkDate.tsx#L6](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkDate.tsx#L6)：`link.importDate \|\| link.createdAt`——**importDate 优先，fallback 到 createdAt** |

###### `lastPreserved` —— 可空，归档完成后填充

| 属性 | 值 |
|------|---|
| DB 可空性 | ✅ **可空**（`DateTime?`，无 default） |
| 值来源 | 1. 新建不可抓取的 URL：`postLink.ts#L140` 直接设为当前时间（`!shouldPreserveUrl && link.url` 时）<br>2. 归档 Worker 完成所有格式处理后：[archiveHandler.ts#L216](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/lib/archiveHandler.ts#L216) 设为当前时间<br>3. Worker 跳过归档时（skipPreservation / 非 http URL）：[archiveHandler.ts#L51](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/lib/archiveHandler.ts#L51) 设为当前时间 |
| 空值场景 | **新建且尚未被 Worker 处理的可抓取 URL 类型 Link**——约为新创建后几秒钟到几分钟内为 null |
| 含义 | 表示"归档流程已执行"——无论成功还是失败，只要 Worker 处理过就会有值 |

###### `url` —— 可空，**源链接字段（非归档产物）**，不遵循三态模式

> **重要纠正：之前文档将 url 与 preview/image 等归为"归档三态字段"是错误的。url 是 Link 的核心源数据，表示用户收藏的原始外部 URL，与 Worker 归档产物完全不同。**

| 属性 | 值 |
|------|---|
| 分类 | **源链接（业务输入）**，不是归档产物 |
| DB 可空性 | ✅ **可空**（`String?`，无 default） |
| 值来源 | 1. 用户新建时输入：`PostLinkSchema.url` 校验 `z.string().trim().max(2048).url().optional()`（[schemaValidation.ts#L127](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/lib/schemaValidation.ts#L127)）<br>2. 导入时从备份文件/Pocket API 读取（[importFromLinkwarden.ts#L78](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L78)、[importFromPocket.ts#L59](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/migration/importFromPocket.ts#L59)） |
| 空值场景 | **用户手动上传本地文件（PDF/PNG/JPG/HTML）作为 Link 时，url 为 null**——此时 Link 的内容完全是归档文件本身，没有外部源链接 |
| 是否会被设为 "unavailable" | ❌ **绝对不会**。url 是用户原始输入，从未在任何 Worker 或 API 中被设置为 "unavailable" 字符串 |
| 值形式 | 标准 URL 字符串（如 `"https://example.com/article"`），或 `null` |
| 典型占比 | 约 **90%+** 的 Link 有 url（纯上传文件的 Link 占少数） |

**字段定位总结：**

| 维度 | `url`（源链接） | `preview` / `image` / `pdf` 等（归档产物） |
|------|----------------|------------------------------------------|
| 数据来源 | 用户输入/导入 | Worker 异步生成 |
| 值含义 | 被收藏的原始网页地址 | 本地存储的归档文件相对路径 |
| 三态模式 | ❌ 不遵循 | ✅ 严格遵循 |
| 值为 "unavailable" | 永远不会 | 会 |
| 与 type 的关系 | type="url" 时必有 url；type="pdf"/"image"/"monolith" 时 url 可为 null | 与 type 交叉：如 type="pdf" 时 pdf 字段有值，type="url" 时可同时有 image/pdf/readable |

---

###### `preview` / `image` / `pdf` / `readable` / `monolith` —— 可空，**归档产物字段**，严格三态模式

这 5 个字段均为 Worker 归档处理的产物，遵循统一的三态值模式：

| 状态 | 值示例 | 含义 |
|------|--------|------|
| **未处理** | `null` | Link 创建后尚未被 Worker 处理，或用户未启用该归档格式 |
| **处理成功** | `"archives/preview/12/456.jpeg"` / `"archives/12/456.pdf"` / `"archives/12/456_readability.json"` 等 | 归档文件的相对路径（相对于存储根目录） |
| **不可用** | `"unavailable"` | Worker 已尝试处理，但确认无法生成该格式（见 [archiveHandler.ts#L197-L212](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/lib/archiveHandler.ts#L197-L212)：Worker 遍历所有归档格式后，将仍为 null 的字段统一标记为 "unavailable"） |

**各字段的生成来源函数：**

| 字段 | 生成函数 | 触发条件 | 说明 |
|------|---------|---------|------|
| `preview` | [handleArchivePreview.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts) | type="url" 且服务端可抓取 | 优先取 og:image，否则 Playwright 低质量截图（quality: 20） |
| `image` | [handleScreenshotAndPdf.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts) | 用户启用 `archiveAsScreenshot` 且 type="url" | 整页高质量截图 |
| `pdf` | 同上 `handleScreenshotAndPdf.ts` | 用户启用 `archiveAsPDF` 且 type="url" | Playwright 生成 PDF |
| `readable` | [handleReadability.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/lib/preservationScheme/handleReadability.ts) | type="url" 且页面有正文内容 | Readability 提取 + DOMPurify 净化，输出 JSON |
| `monolith` | [handleMonolith.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/lib/preservationScheme/handleMonolith.ts) | 用户启用且 type="url" | Monolith 单页 HTML 归档 |

**`preview` 字段额外说明**：通过 `/api/v1/archives/{id}?format=jpeg&preview=true` 接口访问时会附带 `&updatedAt={updatedAt}` 作为缓存刷新参数（[DashboardLinks.tsx#L157](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/components/DashboardLinks.tsx#L157)）。该接口需要用户登录鉴权（[linkId].ts#L104-L119），**邮件中无法直接访问**。

**手动上传文件时的归档字段行为**（[linkId].ts#L262-L280）：用户通过 API POST `/api/v1/archives/{linkId}` 手动上传文件时，直接设置对应字段为归档路径，并将其他不适用的归档格式标记为 "unavailable"（例如上传 PDF 时设置 `preview: "unavailable"`）。

##### 5.2.5.2 邮件摘要展示时的空值处理建议

| 字段 | 空值/默认值 | 邮件中处理建议 |
|------|------------|---------------|
| **`name`** | `""`（空字符串） | 显示 fallback：<br>1. 若有 `url` → 显示域名（`new URL(url).host`）<br>2. 若无 `url` → 显示 `type`（如 `"PDF"`、`"Image"`、`"Monolith"`） |
| **`description`** | `""`（空字符串） | fallback 到 `metaDescription`（若 metaDescription 也为空则不展示描述行） |
| **`metaDescription`** | `null` | fallback 到 `description`（两者都空则省略描述区域） |
| **`url`** | `null` | 说明：表示该 Link 是纯上传文件（无外部源链接）<br>处理：**不渲染外部跳转链接**，但仍渲染 Linkwarden 内部详情页链接 `/dashboard/links/{id}` |
| **`preview`** | `null` 或 `"unavailable"` | 邮件缩略图处理（详见下方跳转链接与缩略图专题建议） |
| **`image` / `pdf` / `readable` / `monolith`** | `null` 或 `"unavailable"` | 邮件摘要中通常无需单独展示这些格式的可用性；可在"可用归档格式"区域汇总展示，或直接忽略 |
| **`importDate`** | `null` | fallback 到 `createdAt`，与前端 LinkDate 组件一致 |
| **`lastPreserved`** | `null` | 不展示该指标；或标注为"待归档" |
| **`tags`** | `[]`（空数组） | 不展示标签行 |
| **`pinnedBy`** | `[]`（空数组） | 不显示"固定"徽标 |

###### 邮件摘要跳转链接与缩略图专题建议

**一、跳转链接策略**

邮件摘要中每条链接应提供**两个独立的跳转入口**：

| 跳转目标 | URL 构造 | 触发条件 | 说明 |
|---------|---------|---------|------|
| **外部原始链接** | `link.url` | 仅当 `link.url != null` | 直接跳到用户收藏的源网页；点击可加 UTM 参数（如 `?utm_source=linkwarden-digest`） |
| **Linkwarden 详情页** | `${BASE_URL}/dashboard/links/${link.id}` | **始终提供** | 进入 Linkwarden 内部的归档详情页，可查看所有归档格式、编辑标签等 |

跳转优先级参考前端实现 [getFormatBasedOnPreference.ts#L15-L48](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/lib/getFormatBasedOnPreference.ts#L15-L48)：
- `type === "url"` 且用户偏好为 ORIGINAL → 跳 `link.url`
- `type === "pdf"` 且有 pdf 归档 → 跳 `/api/v1/archives/{id}?format=pdf`（但邮件中此链接需要鉴权，见下）
- 其他情况 → 跳详情页

**⚠️ 归档文件 API 的鉴权问题：** `/api/v1/archives/{linkId}?format=...` 接口在服务端渲染时通过 Cookie 鉴权（[linkId].ts#L104-L119），但**邮件客户端点击请求不会携带用户登录 Cookie**，因此：
- 邮件中**不应直接链接到 `/api/v1/archives/...`**——用户会看到 401
- 推荐一律跳详情页 `/dashboard/links/{id}`，由详情页再提供各归档格式下载入口

**二、缩略图展示策略**

缩略图有三种来源，按优先级排列：

| 方案 | 实现方式 | 优点 | 缺点 | 适用场景 |
|------|---------|------|------|---------|
| **方案 A：外部 og:image 直链** | 若 `preview` 非 null 且原始 `url` 可推断 og:image（通过 Google S2 Favicon API 或 DuckDuckGo Instant Answer API 获取站点缩略图） | 无需鉴权，邮件直接展示 | og:image 不一定存在；第三方 API 可能有限流 | url 类型 Link 推荐 |
| **方案 B：Linkwarden 用户内容域名** | 若配置了 `NEXT_PUBLIC_USER_CONTENT_DOMAIN`，可通过该域名公开提供归档预览图（需服务端额外配置公开访问路径） | 使用自有归档，数据最准确 | 需额外基础设施配置；未配置则不可用 | 自托管且配置了用户内容域名 |
| **方案 C：站点 Favicon 占位** | `https://www.google.com/s2/favicons?domain={hostname}&sz=128` 或 `https://icons.duckduckgo.com/ip3/{hostname}.ico` | 无需鉴权、几乎 100% 可用、稳定 | 只是小图标，不是页面缩略图 | preview 不可用时的 fallback |
| **方案 D：不展示缩略图** | 用彩色图标 + 标题卡片替代 | 最简洁，无外部依赖 | 视觉效果较差 | 极简摘要模板 |

**推荐缩略图处理逻辑：**

```
1. 若 link.type === "image" 且 link.image 有值
   → 用户上传的图片文件：邮件中无法直链归档，使用方案 C（Favicon）或 D（不展示）
   
2. 若 link.type === "pdf"
   → 使用通用 PDF 图标（inline SVG）或方案 C

3. 若 link.type === "url" 且 link.url != null：
   a. 尝试方案 A（通过第三方 API 获取 og:image）
   b. 若无法获取 → fallback 到方案 C（Favicon 128px）
   c. 若 url 为 null → fallback 到方案 D

4. 所有方案均失败 → 方案 D（纯文字卡片，带 link.icon + link.color 自定义图标或默认图标）
```

**模板渲染优先级（更新版，单条链接完整展示）：**

```
┌─────────────────────────────────────────────────────────┐
│  [缩略图区域]                                            │
│  方案 A/B/C/D 生成的图片，点击跳转到：                    │
│    link.url || `${BASE_URL}/dashboard/links/${link.id}`  │
├─────────────────────────────────────────────────────────┤
│  📌 [pinnedBy 有值时显示固定徽标]  [收藏夹名称·颜色圆点]  │
│                                                         │
│  标题（加粗）：                                          │
│    name != "" → name                                     │
│    url != null  → new URL(url).hostname                  │
│    → type.toUpperCase()                                  │
│                                                         │
│  描述（灰色小字，最多 2 行）：                             │
│    description != "" → description（truncate 200 字符）   │
│    metaDescription != null → metaDescription             │
│    → 不展示                                              │
│                                                         │
│  [标签1] [标签2] ...（tags.length > 0 时展示）            │
│                                                         │
│  📅 日期：(importDate || createdAt).toLocaleDateString() │
│  🔗 外部链接：仅当 url != null 时展示                     │
│  📂 在 Linkwarden 中查看：详情页链接（始终展示）           │
└─────────────────────────────────────────────────────────┘
```

#### 5.2.6 collectionLinks 返回结构（第 149-152 行）

```typescript
const collectionLinks: Record<number, any[]> = {};
collectionsResult.forEach(({ colId, links }) => {
  collectionLinks[colId] = links;
});
```

**实际返回结构：**
```jsonc
{
  "collectionLinks": {
    "5": [ /* id=5 的收藏夹最近 16 条链接 */ ],
    "12": [ /* id=12 的收藏夹最近 16 条链接 */ ]
  }
}
```

- Key 为 `collectionId`（数字），Value 为该收藏夹下最近 16 条 Link 对象数组
- 仅包含用户在 Dashboard 配置中**显式添加**的 COLLECTION 分区
- 未启用或无权限的收藏夹不出现于此对象中

#### 5.2.7 链接合并与去重逻辑（第 154-159 行）

```typescript
const merged = [...recentlyAddedLinks, ...pinnedLinks].sort(
  (a, b) => new Date(b.id).getTime() - new Date(a.id).getTime()
);
const uniqueLinks = merged.filter(
  (link, idx, arr) => idx === arr.findIndex((l) => l.id === link.id)
);
```

**合并流程详解：**

1. **拼接**：`recentlyAddedLinks`（最多 16 条）在前，`pinnedLinks`（最多 16 条）在后，构成最多 32 条的临时数组
2. **排序**：按 `id`（即创建时间）降序排列。注意使用了 `new Date(b.id).getTime()` —— 因为 id 为自增整数，但代码将其当作 Date 字符串解析，**实际等价于数值比较**（自增 id 的大小关系与创建时间先后一致）
3. **去重**：使用 `findIndex` 按 `link.id` 去重。若某条链接同时存在于 `recentlyAddedLinks` 和 `pinnedLinks` 中：
   - 排序后相同 id 的相邻出现
   - `filter` 保留第一次出现（即保留该条），后续重复被丢弃
   - 最终 `uniqueLinks` 的最大长度约为 16~32 条，取决于 pinned 与 recent 的重叠程度

#### 5.2.8 最终返回结构

```typescript
return {
  data: {
    links: uniqueLinks,        // 合并去重后的 pinned+recent 链接
    collectionLinks,           // Record<collectionId, Link[]>
    numberOfPinnedLinks,       // 统计数字
    numberOfTags,              // 统计数字
  },
  statusCode: 200,
  success: true,
  message: "Dashboard data fetched successfully.",
};
```

### 5.3 前端如何消费这些数据

**文件：** [dashboard.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/dashboard.tsx)

前端按 `dashboardSections` 的 `order` 字段排序后逐个渲染 Section：

| Section 类型 | 数据来源 | 渲染方式 |
|--------------|----------|----------|
| `STATS` | `numberOfLinks`（前端 collections 聚合而来）、`collections.length`、`numberOfTags`、`numberOfPinnedLinks` | 4 个 DashboardItem 统计卡片 |
| `RECENT_LINKS` | `links`（即后端合并去重后的 `uniqueLinks`） | 水平滚动卡片列表（[DashboardLinks.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/components/DashboardLinks.tsx)） |
| `PINNED_LINKS` | `links.filter(e => e.pinnedBy && e.pinnedBy[0])` —— **从同一 `links` 数组中前端过滤** | 同上水平滚动卡片列表，支持拖放固定 |
| `COLLECTION` | `collectionLinks[section.collectionId]` | 同上水平滚动卡片列表，支持拖放 |

**重要发现：RECENT_LINKS 和 PINNED_LINKS 共用同一个后端返回的 `links` 数组**，前端仅通过是否存在 `pinnedBy` 关系来区分渲染。

### 5.4 Dashboard 查询对邮件摘要的适用性评估

| 维度 | 现状 | 对 Email Digest 的适配度 | 需要的调整 |
|------|------|-------------------------|-----------|
| **权限过滤** | ✅ 已实现（owner + member 双重校验） | 完全适用 | 无需调整 |
| **时间窗口** | ❌ 无时间过滤，仅取最近 N 条 | **不适用** | 必须添加 `createdAt >= lastDigestSentAt` 过滤条件 |
| **数据量** | ⚠️ 固定 take 16 条/分区 | 邮件摘要可能需要可配置数量（如 weekly=10, daily=5） | 参数化 take 值 |
| **collectionLinks** | ⚠️ 依赖用户 Dashboard 的 COLLECTION 分区配置 | 邮件摘要可能需要"所有收藏夹各取 N 条"而非仅 Dashboard 配置的 | 绕过 dashboardSections，直接查用户可访问的所有 Collection |
| **links 合并去重** | ✅ 已实现按 id 去重 | 完全适用 | 无需调整 |
| **字段内容** | ✅ 除 `textContent` 外所有标量字段均返回（name, description, metaDescription, url, preview, image 等共 23 个），同时 include tags、collection、pinnedBy 关系 | **高度适用**——邮件摘要所需核心字段（标题、URL、描述、预览图、标签、收藏夹、固定状态）全部已返回 | 无需调整，可直接复用 |
| **统计数据** | ✅ numberOfPinnedLinks、numberOfTags | 基本适用 | 邮件摘要可能还需要 period 内新增链接数等增量指标 |
| **分区开关依赖** | ⚠️ 查询依赖 dashboardSections 存在性 | 邮件摘要不应被 Dashboard UI 开关影响 | 移除 section 存在性检查，始终返回完整摘要数据 |

**总体评估：Dashboard V2 查询的数据聚合逻辑**（权限过滤、合并去重、多收藏夹查询、字段完整性）**可以较高程度复用为 Digest 数据层，尤其字段方面无需任何改动**。需要改造的只有三点：剥离对 dashboardSections 的依赖、增加时间窗口过滤、并参数化返回条数。不建议直接调用 `getDashboardDataV2()`，建议新建一个 `getDigestData(userId, since, limit)` 函数在其基础上改造。

---

## 六、实现 Email Digest 的建议路径

基于上述分析，若要新增定期摘要邮件功能，建议按以下路径实现：

### 6.1 数据模型扩展

在 `User` 模型中新增字段（schema.prisma）：
- `digestFrequency` 枚举（`DISABLED` / `DAILY` / `WEEKLY` / `MONTHLY`）
- `lastDigestSentAt` DateTime（幂等标记，替代轮询时的时间窗口计算）

同时需要在 `UpdateUserSchema` 中补充该字段的更新定义，并在账户设置页增加 UI 入口。

### 6.2 用户偏好设置入口

补充 Digest 频率设置 UI 与 Schema：
- `account.tsx` 或 `preference.tsx` 增加频率选择控件
- `schemaValidation.ts` 的 `UpdateUserSchema` 增加字段定义
- `updateUserById.ts` 对应字段持久化
- **建议同时补全 `acceptPromotionalEmails` 的修改入口**，或直接将 Digest 频率作为独立偏好不受其影响

### 6.3 Digest Worker 新建

参考 `trialEndEmailWorker.ts` 新建 `apps/worker/workers/emailDigestWorker.ts`：

1. 候选用户筛选：
   - `digestFrequency != DISABLED`
   - `emailVerified IS NOT NULL`
   - `email IS NOT NULL`
   - `lastDigestSentAt IS NULL OR lastDigestSentAt <= cutoff`（按频率计算 cutoff：DAILY=24h前, WEEKLY=7d前, MONTHLY=30d前）
   - 建议参考 `lastPickedAt` 公平调度模式，`orderBy lastDigestSentAt asc nulls first`

2. 对每个用户调用新的 `getDigestData(userId, since=lastDigestSentAt, limit)` 获取摘要数据

3. 加载对应频率的 Handlebars 模板（`digestDaily.html` / `digestWeekly.html` / `digestMonthly.html`）

4. 使用 `transporter.sendMail` 发送

5. 成功后批量更新 `lastDigestSentAt = now()`

6. 调度节奏：参考 trialEndEmailWorker，使用 `while(true)` + 批处理 + 批间 sleep（如 10 秒），每批处理 20 用户

### 6.4 注册到 Worker 入口

在 [worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/worker.ts) 的 `init()` 中追加 `startEmailDigestWorker()`。

---

## 七、相关文件索引

| 类别 | 文件路径 |
|------|----------|
| **邮件传输** | [packages/lib/transporter.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/lib/transporter.ts) |
| **Worker 主入口** | [apps/worker/worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/worker.ts) |
| **Worker 进程守护** | [apps/worker/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/index.ts) |
| **试用期邮件 Worker**（Digest 参考实现） | [apps/worker/workers/trialEndEmailWorker.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/trialEndEmailWorker.ts) |
| **试用期邮件模板** | [apps/worker/templates/trialEnded.html](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/templates/trialEnded.html) |
| **邮箱验证发送** | [apps/web/lib/api/sendVerificationRequest.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/sendVerificationRequest.ts) |
| **Prisma 数据模型** | [packages/prisma/schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/prisma/schema.prisma) |
| **用户注册（偏好采集+默认分区创建）** | [apps/web/lib/api/controllers/users/postUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/users/postUser.ts) |
| **注册 UI** | [apps/web/pages/register.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/register.tsx) |
| **用户更新 API** | [apps/web/lib/api/controllers/users/userId/updateUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserById.ts) |
| **用户偏好更新 API** | [apps/web/lib/api/controllers/users/userId/updateUserPreference.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserPreference.ts) |
| **Schema 校验** | [packages/lib/schemaValidation.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/lib/schemaValidation.ts) |
| **公平调度（lastPickedAt）** | [apps/worker/lib/getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/lib/getLinkBatchFairly.ts) |
| **Dashboard V2 数据聚合** | [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts) |
| **Dashboard V1（旧版）** | [apps/web/lib/api/controllers/dashboard/getDashboardData.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/dashboard/getDashboardData.ts) |
| **Dashboard 分区更新 API** | [apps/web/lib/api/controllers/dashboard/updateDashboardLayout.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/dashboard/updateDashboardLayout.ts) |
| **Dashboard V2 API 路由** | [apps/web/pages/api/v2/dashboard/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/api/v2/dashboard/index.ts) |
| **Dashboard 分区编辑 UI** | [apps/web/components/DashboardLayoutDropdown.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/components/DashboardLayoutDropdown.tsx) |
| **Dashboard 页面渲染** | [apps/web/pages/dashboard.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/dashboard.tsx) |
| **Dashboard 链接卡片组件** | [apps/web/components/DashboardLinks.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/components/DashboardLinks.tsx) |
| **Dashboard React Query Hook** | [packages/router/dashboardData.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/router/dashboardData.tsx) |
| **RSS 轮询 Worker**（长间隔调度参考） | [apps/worker/workers/rssPolling.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/rssPolling.ts) |
| **AI 自动打标签 Worker** | [apps/worker/workers/autoTagPreservedLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/autoTagPreservedLinks.ts) |
| **用户注销（读取 acceptPromotionalEmails）** | [apps/web/lib/api/controllers/users/userId/deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts) |
| **配置 API** | [apps/web/pages/api/v1/config/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/api/v1/config/index.ts) |
| **数据库 Seed** | [packages/prisma/seed.js](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/prisma/seed.js) |
