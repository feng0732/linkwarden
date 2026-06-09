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

**模板加载方式示例**（来自 [trialEndEmailWorker.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/trialEndEmailWorker.ts#L77-L94)）：

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

## 三、周期性 Worker 调度框架（可作为 Digest 发送节奏参考）

### 3.1 Worker 总入口

**文件：** [worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/worker.ts)

Worker 进程通过 [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/index.ts) 以子进程方式启动，崩溃后 5 秒自动重启。主入口 `init()` 顺序启动以下任务：

```typescript
async function init() {
  await migrationWorker();
  startRSSPolling();              // RSS 源轮询
  linkProcessing(workerIntervalInSeconds);   // 链接归档处理
  autoTagPreservedLinks(workerIntervalInSeconds); // AI 自动打标签
  startIndexing(workerIntervalInSeconds);    // 全文索引
  trialEndEmailWorker();          // 试用期结束邮件（现有周期性邮件）
}
```

### 3.2 现有调度模式对比

| Worker | 调度方式 | 间隔/节奏 | 文件 |
|--------|----------|-----------|------|
| `linkProcessing` | `while(true)` + `delay(interval)` | 默认 10 秒，由 `ARCHIVE_SCRIPT_INTERVAL` 控制 | [linkProcessing.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/linkProcessing.ts#L11) |
| `startRSSPolling` | `while(true)` + `delay(pollingIntervalInSeconds)` | 默认 **60 分钟**，由 `NEXT_PUBLIC_RSS_POLLING_INTERVAL_MINUTES` 控制 | [rssPolling.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/rssPolling.ts#L8-L37) |
| `trialEndEmailWorker` | `while(true)` + 批处理 + `sleep(pauseMs)` | 每批 10 用户，批间 **30 秒** 休眠 | [trialEndEmailWorker.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/trialEndEmailWorker.ts#L9-L124) |

### 3.3 `trialEndEmailWorker` —— 唯一现存周期性邮件的完整流程

这是理解未来 Digest Worker 最直接的参考。其处理路径如下：

1. **前置条件检查**（第 22-30 行）：
   - `NEXT_PUBLIC_TRIAL_PERIOD_DAYS > 1`
   - 设置了 `STRIPE_SECRET_KEY`
   - `NEXT_PUBLIC_REQUIRE_CC !== "true"`

2. **候选用户筛选查询**（第 42-56 行）：
   ```
   trialEndEmailSent = false
   AND emailVerified IS NOT NULL
   AND createdAt <= cutoff (当前时间 - trialDays)
   AND createdAt >= '2025-09-25' (安全上界)
   ORDER BY createdAt ASC, take 10
   ```
   同时 include 自身订阅与父订阅（家庭计划）状态。

3. **用户资格二次过滤**（第 68-70 行）：
   - 已拥有 `subscriptions.active` 或 `parentSubscription.active` → 跳过发送，但仍标记已处理
   - `user.email` 为空 → 跳过

4. **邮件发送**（第 76-94 行）：
   - 加载 `trialEnded.html` Handlebars 模板
   - 通过 Nodemailer transporter 发送
   - **失败不标记**，留待下一批重试

5. **状态持久化**（第 111-115 行）：
   - 使用 `updateMany` 将 `processedIds` 批量标记 `trialEndEmailSent = true`

---

## 四、用户偏好与过滤字段

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
| `acceptPromotionalEmails` | Boolean | `false` | 注册时勾选是否接收推广/功能更新邮件。对应翻译键 `accept_promotional_emails` |
| `emailVerified` | DateTime? | `null` | 邮箱是否已验证。**所有周期性邮件均应过滤此项** |
| `lastPickedAt` | DateTime? | 迁移后无默认 | 用于 Worker 公平调度（见 4.3），非邮件专用 |
| `trialEndEmailSent` | Boolean | `false` | 试用期结束邮件是否已发送（幂等标记） |

### 4.2 `acceptPromotionalEmails` 的完整流转

1. **注册时采集**：
   - UI：[register.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/register.tsx#L256-L268) —— 仅当 `NEXT_PUBLIC_STRIPE` 启用时展示 Checkbox
   - Schema 校验：[schemaValidation.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/lib/schemaValidation.ts#L56) —— `z.boolean().default(false)`
   - 持久化：[postUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/users/postUser.ts#L113)

2. **账户设置页缺失**：
   - [account.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/settings/account.tsx) 和 [preference.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/settings/preference.tsx) 中**均未提供修改该偏好的入口**
   - `UpdateUserSchema`（[schemaValidation.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/lib/schemaValidation.ts#L60-L95)）中**也未包含该字段的更新定义**

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

## 五、摘要数据收集的潜在来源（Dashboard 数据聚合逻辑）

虽然没有 Digest 专用的数据聚合器，但 Dashboard 控制器已实现了"最近收藏、固定链接、统计概览"等摘要所需的核心查询，可直接复用。

### 5.1 Dashboard V2 数据聚合

**文件：** [getDashboardDataV2.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts)

该查询一次返回以下摘要数据：

| 指标 | 查询逻辑 | 数量限制 |
|------|----------|----------|
| `numberOfPinnedLinks` | 用户可访问的 Collection 中被该用户 pinned 的 Link 总数 | - |
| `numberOfTags` | 作为 owner 或通过成员身份可触及的 Tag 总数 | - |
| `pinnedLinks` | 最近 16 条 pinned 链接（含 tags、collection、pinnedBy 关系） | 16 条 |
| `recentlyAddedLinks` | 最近 16 条新增链接（含 tags、collection 关系） | 16 条 |
| `collectionLinks` | 按 DashboardSection 中 COLLECTION 类型指定的各收藏夹最近 16 条链接 | 每收藏夹 16 条 |

**权限过滤条件**（所有 Link 查询共享）：
```
collection.ownerId = userId
OR collection.members.some(userId)
```

### 5.1 Dashboard V1（旧版）

**文件：** [getDashboardData.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/dashboard/getDashboardData.ts)

仅返回 pinned + recent 链接的合并去重列表（各取 10 条）。

---

## 六、实现 Email Digest 的建议路径

基于上述分析，若要新增定期摘要邮件功能，建议按以下路径实现：

### 6.1 数据模型扩展

在 `User` 模型中新增字段（schema.prisma）：
- `digestFrequency` 枚举（`DISABLED` / `DAILY` / `WEEKLY` / `MONTHLY`）
- `lastDigestSentAt` DateTime（幂等标记，替代轮询时的时间窗口计算）

### 6.2 用户偏好设置入口

在以下文件中补充 Digest 频率设置 UI 与 Schema：
- `preference.tsx` 或 `account.tsx` 增加频率选择控件
- `schemaValidation.ts` 的 `UpdateUserSchema` 增加字段定义
- `updateUserById.ts` 对应字段持久化

### 6.3 Digest Worker 新建

参考 `trialEndEmailWorker.ts` 新建 `apps/worker/workers/emailDigestWorker.ts`：
1. 按频率筛选候选用户（`digestFrequency != DISABLED`、`emailVerified != null`、基于 `lastDigestSentAt` 判断是否到达发送窗口）
2. 过滤 `acceptPromotionalEmails = true`（若 Digest 定位为营销类）
3. 调用 `getDashboardDataV2` 的查询逻辑收集该用户的摘要数据
4. 新建 `digestDaily.html` / `digestWeekly.html` Handlebars 模板
5. 使用 `transporter.sendMail` 发送，成功后更新 `lastDigestSentAt`

### 6.4 注册到 Worker 入口

在 [worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/worker.ts) 的 `init()` 中追加 `startEmailDigest()`。

---

## 七、相关文件索引

| 类别 | 文件路径 |
|------|----------|
| **邮件传输** | [packages/lib/transporter.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/lib/transporter.ts) |
| **Worker 主入口** | [apps/worker/worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/worker.ts) |
| **Worker 进程守护** | [apps/worker/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/index.ts) |
| **试用期邮件 Worker** | [apps/worker/workers/trialEndEmailWorker.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/trialEndEmailWorker.ts) |
| **试用期邮件模板** | [apps/worker/templates/trialEnded.html](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/templates/trialEnded.html) |
| **邮箱验证发送** | [apps/web/lib/api/sendVerificationRequest.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/sendVerificationRequest.ts) |
| **Prisma 数据模型** | [packages/prisma/schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/prisma/schema.prisma) |
| **用户注册（偏好采集）** | [apps/web/pages/register.tsx](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/register.tsx) |
| **用户创建 API** | [apps/web/lib/api/controllers/users/postUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/users/postUser.ts) |
| **用户更新 API** | [apps/web/lib/api/controllers/users/userId/updateUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserById.ts) |
| **Schema 校验** | [packages/lib/schemaValidation.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/packages/lib/schemaValidation.ts) |
| **公平调度（lastPickedAt）** | [apps/worker/lib/getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/lib/getLinkBatchFairly.ts) |
| **Dashboard V2 聚合** | [apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/dashboard/getDashboardDataV2.ts) |
| **RSS 轮询 Worker** | [apps/worker/workers/rssPolling.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/worker/workers/rssPolling.ts) |
| **配置 API** | [apps/web/pages/api/v1/config/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/pages/api/v1/config/index.ts) |
| **用户注销（读取偏好）** | [apps/web/lib/api/controllers/users/userId/deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/127-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts) |
