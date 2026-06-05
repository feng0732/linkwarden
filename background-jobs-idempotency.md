# 后台任务的重试与幂等保证分析

## 一、整体架构概览

Linkwarden 的后台任务系统由独立的 `apps/worker` 进程承载，通过 [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/index.ts#L1-L16) 中的 supervisor 模式（进程崩溃后 5 秒自动重启）保障持续运行。核心 worker 入口在 [worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/worker.ts#L1-L22)，初始化后并行启动 6 类任务：

| Worker | 职责 | 状态/幂等字段 |
|---|---|---|
| `linkProcessing` | 网页归档（截图/PDF/Monolith/Readability/预览） | `lastPreserved`, `image/pdf/readable/monolith/preview` |
| `autoTagPreservedLinks` | AI 自动打标签 | `aiTagged`, `lastPreserved` |
| `startIndexing` | Meilisearch 全文索引同步 | `indexVersion` |
| `startRSSPolling` | RSS 源拉取新文章 | `lastBuildDate` (RssSubscription) |
| `trialEndEmailWorker` | 试用期结束邮件通知 | `trialEndEmailSent` |
| `migrationWorker` | 应用级数据迁移 | `AppMigration.status` (PENDING/APPLIED/FAILED) |

---

## 二、任务领取机制

### 2.1 链接归档 / AI 标签：公平批次调度 `getLinkBatchFairly`

核心实现在 [getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/getLinkBatchFairly.ts#L12-L179)。

**领取流程：**

1. **资格筛选**：先查询有"待处理链接"的用户，同时校验订阅状态（付费 / 试用期内 / 邮箱已验证）
2. **用户轮询**：按 `lastPickedAt` 升序取用户（从未被挑过的用户优先，nulls first），实现用户间的公平调度
3. **链接分配**：每个用户按固定配额 `linksPerUser = floor(maxBatchLinks / users.length)` 轮询取链接，用 `Set<number>` 去重防止同批次重复
4. **更新水位**：用 `updateMany` 批量把涉及用户的 `lastPickedAt` 设为当前时间

**关键查询条件（mode = "links"）** ——这也是自动重试的唯一判断依据：

```ts
// getLinkBatchFairly.ts L35-L38
url: { not: null },
lastPreserved: null,
```

**关键查询条件（mode = "tags"）：**

```ts
url: { not: null },
type: "url",
lastPreserved: { not: null },   // 必须先归档完成才打标
aiTagged: false,
collection: { owner: { aiTaggingMethod: { not: "DISABLED" } } },
```

### 2.2 搜索索引：简单批次 `getLinkBatch`

[getLinkBatch.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/getLinkBatch.ts#L11-L51) 从 ID 两端交替取数据（一半升序、一半降序），用内存 filter 去重，避免"最新/最老"饥饿问题。

### 2.3 领取阶段的并发风险 ⚠️

**当前实现未使用数据库行锁（SELECT ... FOR UPDATE/SKIP LOCKED）。** 在多 worker 实例部署时，多个进程可能读到相同的 `lastPreserved = null` 链接，导致重复归档。单实例部署下无此问题。

---

## 三、归档失败、浏览器超时与 finally 的精确关系

本章是重新核对后的核心结论，围绕 [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/archiveHandler.ts) 展开。

### 3.1 archiveHandler 的三段式控制流

```
archiveHandler(link, browser)
│
├─ 段 A：try 之前的初始化与短路分支（L29-L108）
│    ├─ SSRF 安全检查 / 非 http(s) URL → 直接写 lastPreserved + 全 unavailable，return
│    ├─ 创建 AbortController + timeoutPromise（5 分钟超时）
│    ├─ browser.newContext / newPage
│    └─ 解析 archivalSettings
│
├─ 段 B：try 块（L109-L198）
│    └─ Promise.race([ 主归档 IIFE ,  timeoutPromise  ])
│         ├─ determineLinkType (fetchHeaders)
│         ├─ imageHandler / pdfHandler / page.goto
│         ├─ handleArchivePreview
│         ├─ handleReadability
│         ├─ handleScreenshotAndPdf
│         └─ handleMonolith  ← 自带 .catch 吞错
│
├─ 段 C：catch 块（L199-L202）—— 仅打日志后 throw err 重新抛出
│
└─ 段 D：finally 块（L203-L230）——★ 只要进入了 try 就一定执行
     ├─ clearTimeout
     ├─ prisma.link.findUnique → finalLink
     ├─ finalLink 存在：
     │    ├─ lastPreserved = now
     │    ├─ 对 null 的格式字段写 "unavailable"
     │    └─ indexVersion = null（触发重新索引）
     ├─ finalLink 不存在：removeFiles 清理孤儿文件
     └─ context.close()
```

**JS 语义决定**：一旦控制流进入 `try { ... }`，无论是正常 `return`、被 `throw`、还是 `Promise.race` 被 timeout 抢占，`finally` 都会执行。只有两种例外：
1. 在进入 `try` 之前（段 A）就已经抛异常或 return
2. 进程级崩溃（OOM kill、SIGKILL、宿主机断电）

### 3.2 浏览器超时的完整触发链

超时由 [archiveHandler.ts#L63-L75](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/archiveHandler.ts#L63-L75) 驱动：

```ts
const timeoutPromise = new Promise((_, reject) => {
  timeoutId = setTimeout(() => {
    abortController.abort();                       // ① 通知 Monolith 子进程
    reject(new Error("Browser has been open for more than N minutes."));  // ② 让 Promise.race 结束
  }, BROWSER_TIMEOUT * 60000);
});
```

当 `BROWSER_TIMEOUT`（默认 5 分钟）到达时：

1. `timeoutPromise` 被 reject，`Promise.race` 立即以此 reject 结束
2. 异常进入外层 `catch`，打印日志后被 `throw err` 重新抛出
3. **finally 照常执行**：写 `lastPreserved`、未完成的格式写 `"unavailable"`、关 context
4. 异常继续向上冒泡到 [linkProcessing.ts#L58-L68](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/workers/linkProcessing.ts#L58-L68) 的 `archiveLink` catch 块，记录错误并检查浏览器连接，但**不再抛出**
5. 最外层 `Promise.allSettled` 只是保险——异常已经在 `archiveLink` 层被吞掉

注意：`abortController.signal` 只传给了 `handleMonolith`，其他子流程（page.goto、截图、PDF、Readability）对超时不感知，它们可能在后台继续跑一会儿，但 `Promise.race` 已经让主流程结束并进入 finally。

### 3.3 各子处理器的异常冒泡与 finally 执行关系

下表说明"子模块失败"是否会导致外层 try 抛异常——但无论是否抛，**只要进入了 try，finally 都会执行**：

| 子模块 | 是否 await | 是否内部吞错 | 异常是否冒泡到外层 try |
|---|---|---|---|
| `determineLinkType` → `fetchHeaders` → `safeFetch` | ✅ | ❌ | ✅ 冒泡（DNS/断网/重定向过多等） |
| `imageHandler` / `pdfHandler` → `safeFetch` | ✅ | ❌ | ✅ 冒泡 |
| `page.goto(url)` | ✅ | ❌ | ✅ 冒泡（4xx/5xx、导航超时等） |
| `handleArchivePreview` | ✅ | ⚠️ SSRF 的 UnsafeUrlError 吞掉；其他异常抛出 | ⚠️ 部分冒泡 |
| `handleReadability`（Readability + JSDOM） | ✅ | ❌ | ✅ 冒泡（解析失败、Buffer 超限直接 return 不抛） |
| `handleScreenshotAndPdf` | ✅ | ✅ 内部 `Promise.allSettled` + `.then()` 链无 catch | ❌ 不冒泡；截图/pdf 失败只影响自身字段 |
| `handleMonolith` | ✅ | ✅ 末尾 `.catch(err => console.error(err))` | ❌ 不冒泡；失败只影响 monolith 字段 |
| `timeoutPromise`（浏览器超时） | ✅（由 Promise.race 触发） | ❌ | ✅ 冒泡 → 进入 catch → 进入 finally |

### 3.4 finally 内部如何写入状态

[archiveHandler.ts#L208-L224](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/archiveHandler.ts#L208-L224)：

```ts
const finalLink = await prisma.link.findUnique({ where: { id: link.id } });

if (finalLink) {
  await prisma.link.update({
    where: { id: link.id },
    data: {
      lastPreserved: new Date().toISOString(),   // ★ 链接级"已处理"标记
      readable: !finalLink.readable ? "unavailable" : undefined,
      image:    !finalLink.image    ? "unavailable" : undefined,
      monolith: !finalLink.monolith ? "unavailable" : undefined,
      pdf:      !finalLink.pdf      ? "unavailable" : undefined,
      preview:  !finalLink.preview  ? "unavailable" : undefined,
      indexVersion: null,                          // 触发 Meilisearch 重索引
    },
  });
}
```

关键点：
- **先重新查一次 DB**，不用内存里的旧 `link` 对象——因为子处理器（`imageHandler`、`handleReadability` 等）在执行过程中已经把自己的路径写进 DB 了，直接用内存值会覆盖掉已成功的字段
- **只对 falsy 字段写 `"unavailable"`**，已是 archive 路径的字段传 `undefined`（Prisma 语义=不更新）
- `indexVersion: null` 是有意为之：归档内容更新后需要重新全文索引

---

## 四、lastPreserved 与 unavailable 如何影响自动重试

### 4.1 两层幂等屏障

归档系统实际存在**两层**不同粒度的保护：

| 层级 | 字段 | 作用域 | 影响 |
|---|---|---|---|
| 链接级 | `lastPreserved` | 整条 link | 决定下一轮 `getLinkBatchFairly` 是否会捞起这条链接（= 是否允许整体重试） |
| 格式级 | `image` / `pdf` / `readable` / `monolith` / `preview` | 单个归档产物 | 决定进入归档流程后是否跳过该格式的生成 |

**格式级字段的取值语义：**

| 值 | 含义 | 下次是否再尝试生成该格式 |
|---|---|---|
| `null` | 从未写过 | ✅ 尝试（条件 `!link.image` 为 true） |
| `"archives/.../xxx.jpeg"` | 成功落盘的路径 | ❌ 跳过（已存在且以 archive 开头） |
| `"unavailable"` | 曾经尝试过但失败/放弃 | ❌ 跳过（`"unavailable"` 是 truthy 且不以 "archive" 开头） |

### 4.2 什么情况下会自动重试（lastPreserved 保持 null）

`getLinkBatchFairly` 只看 `lastPreserved == null`。所以只有当 `lastPreserved` 没被写成功时，下一轮轮询才会把这条链接再次捞起。根据 3.1 的控制流分析，仅有三种可能：

| # | 场景 | 为什么 lastPreserved 没写 |
|---|---|---|
| 1 | **段 A 初始化抛异常**：如 `browser.newContext()`、`browser.newPage()` 失败（在进入 try 之前抛出，未触发 finally） | 没走到 finally；也没走到 SSRF 早期分支的 update |
| 2 | **进程级崩溃**：OOM kill、SIGKILL、宿主机断电，导致 finally 虽在 JS 语义上应执行，但运行时已终止 | DB 写入未发生 |
| 3 | **finally 内部的 prisma.update 自身失败**：DB 连接断开、事务冲突等，使 `lastPreserved` 字段没写成功 | finally 尝试执行但 DB 操作报错 |

除此之外的**所有业务失败**（page.goto 404、网络超时、Readability 解析失败、截图 Buffer 超限、浏览器 5 分钟超时、Monolith 子进程异常……）都会进入 finally 并写入 `lastPreserved = now`，因此**不会自动重试**。

### 4.3 精确场景决策表

| 场景 | 是否进入 try | finally 是否执行 | lastPreserved 结果 | 格式字段状态 | 下一轮是否自动重试 |
|---|---|---|---|---|---|
| 所有格式完全成功 | ✅ | ✅ | ✅ 写入时间戳 | 都是 `archives/...` 路径 | ❌ 不重试 |
| page.goto 返回 404 抛异常 | ✅ | ✅ | ✅ 写入时间戳 | 之前完成的保留路径，其余写 `"unavailable"` | ❌ 不重试 |
| fetchHeaders DNS 解析失败 | ✅ | ✅ | ✅ 写入时间戳 | 全部写 `"unavailable"` | ❌ 不重试 |
| handleReadability 抛解析异常 | ✅ | ✅ | ✅ 写入时间戳 | 之前完成的保留路径，`readable=null→"unavailable"` | ❌ 不重试 |
| handleScreenshotAndPdf 内部截图失败 | ✅ | ✅ | ✅ 写入时间戳 | `image=null→"unavailable"`，其他按实际 | ❌ 不重试 |
| handleMonolith 失败（被 .catch 吞） | ✅ | ✅ | ✅ 写入时间戳 | `monolith=null→"unavailable"`，其他按实际 | ❌ 不重试 |
| 浏览器 5 分钟超时（timeoutPromise reject） | ✅ | ✅ | ✅ 写入时间戳 | 已完成的保留路径，未完成的写 `"unavailable"` | ❌ 不重试 |
| SSRF / 非 http(s)（早期 return 分支） | ❌（在 try 之前 return） | ❌ | ✅ 在段 A 直接写入 | 全部写 `"unavailable"` | ❌ 不重试 |
| browser.newContext() 失败（段 A 抛错） | ❌（还没进入 try） | ❌ | ❌ 未写入 | 保持原样（全 null） | ✅ **会重试** |
| 进程 OOM / SIGKILL，死在 try 执行过程中 | ✅（已进入 try） | ❌（进程终止） | ❌ 未写入 | 部分格式可能已写路径，其余 null | ✅ **会重试**（但已写路径的格式下次会被跳过） |
| finally 的 prisma.update 自身报错 | ✅ | ✅（尝试执行） | ❌ 写入失败 | 保持 try 期间各子处理器已写入的状态 | ✅ **会重试** |

### 4.4 重试过程中的格式级"断点续传"

场景"进程 OOM 死在 try 中间"值得单独说明：假设 imageHandler 已成功把截图写入 DB（`image = archives/.../1.jpeg`），但 handleMonolith 还没跑进程就死了。下一轮重试时：

1. `lastPreserved` 仍为 null → 链接会被再次捞起
2. 进入 archiveHandler，try 前的 `!link.image?.startsWith("archive")` 为 false → **跳过截图**
3. `!link.monolith` 为 true → **继续尝试 Monolith**
4. finally 写入 lastPreserved 和剩余的 `"unavailable"`

这是一种天然的**格式级断点续传**——已成功的格式不会重复生成。

### 4.5 为什么设计成"绝大多数失败都不自动重试"

从代码看是有意选择：
1. **Playwright 浏览器资源昂贵**：反复重试失败链接会长时间占用 browser context
2. **失败原因多为不可自动恢复**：目标站点 404、被墙、内容类型异常、SSRf 拦截——这些靠重试几乎不会成功
3. **提供了人工重试入口**：`/api/v1/worker/preservation` DELETE 接口可以把已归档链接的格式字段清空，让它们重新进入待处理队列（见 [packages/router/worker.tsx](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/packages/router/worker.tsx#L23-L49) 的 `useDeletePreservations`）

代价是：暂时性网络抖动导致的失败也被永久标记为 unavailable，需要人工触发重跑。

---

## 五、重复保护（幂等键设计）——非归档类 worker

### 5.1 搜索索引的版本号幂等

用常量 `MEILI_INDEX_VERSION`（见 [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/packages/lib/constants.ts)）作为 schema 版本戳：

- 查询条件：`indexVersion != MEILI_INDEX_VERSION OR indexVersion IS NULL`
- 写入 Meilisearch 成功后，批量把 DB 中这些链接的 `indexVersion` 更新为当前版本号
- 版本号变更即可触发全量重建，无需额外清理逻辑
- Meilisearch `addDocuments` 以 `id` 为主键执行 upsert 语义，重复投递同一文档不会产生副本

### 5.2 AI 自动标签的 aiTagged 标记

[autoTagPreservedLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/workers/autoTagPreservedLinks.ts#L35-L73) 中每个链接的处理有一个 **finally 块无条件写 `aiTagged = true`**：

```ts
} finally {
  await prisma.link
    .update({ where: { id: link.id }, data: { aiTagged: true } })
    .catch(...);
}
```

这意味着**无论 AI API 调用成功还是失败，下一轮都不会再处理这条链接**——失败的链接同样无法自动重试（属于可改进点）。

标签写入自身用 Prisma `connectOrCreate` + DB 唯一约束 `@@unique([name, ownerId])`（见 [autoTagLink.ts#L154-L178](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/autoTagLink.ts#L154-L178)），保障标签不会重复创建。

### 5.3 试用邮件的单次标记

[trialEndEmailWorker.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/workers/trialEndEmailWorker.ts#L40-L124) 用 `trialEndEmailSent = false` 过滤。关键设计：
- 只有 `transporter.sendMail` 成功才把 userId 加入 `processedIds`
- 发送失败时 `continue`，跳过当前用户，**不加入 processedIds**
- 循环末尾才统一 `updateMany` 把 `processedIds` 中的用户标为 `trialEndEmailSent = true`

因此邮件发送失败会在下一轮自动重试，这与归档 worker 的"尽量不重试"策略正好相反。

### 5.4 RSS 的时间戳去重

[rssHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/packages/lib/rssHandler.ts#L28-L91) 比较 feed 的 `lastBuildDate` 与本地存储值，只处理 `pubDate > lastBuildDate` 的新 item。**注意**：创建 link 和更新 `lastBuildDate` 不在同一事务中，崩溃后可能产生重复 link（用户侧 `preventDuplicateLinks` 选项可一定程度缓解）。

### 5.5 迁移任务的状态机

[AppMigration](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/packages/prisma/schema.prisma#L296-L308) 模型用 `PENDING → APPLIED / FAILED` 三态 + `upsert` 初始化 + 按 id 顺序执行，保证迁移最多执行一次。

---

## 六、副作用控制

### 6.1 文件写入的幂等

所有归档产物路径都由 `link.id + collectionId` 唯一确定，例如：

```
archives/{collectionId}/{linkId}.jpeg
archives/{collectionId}/{linkId}.pdf
archives/{collectionId}/{linkId}.html
archives/{collectionId}/{linkId}_readability.json
archives/preview/{collectionId}/{linkId}.jpeg
```

重复执行时，`createFile` 直接**覆盖**同路径文件，不会产生垃圾文件。删除时 [manageFiles.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/packages/filesystem/manageFiles.ts#L4-L68) 也按同样规则定位。

### 6.2 写操作顺序：先文件后 DB

以 [handleMonolith.ts#L57-L66](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/preservationScheme/handleMonolith.ts#L57-L66) 为例：

```ts
await createFile({ data: html, filePath: `archives/${link.collectionId}/${link.id}.html` });
await prisma.link.update({ where: { id: link.id }, data: { monolith: `archives/...` } });
```

**先落盘，再写 DB 路径引用**。若 DB 写入失败，文件可能成为孤儿，但 DB 绝不会指向不存在的文件（避免 UI 层 404）。

### 6.3 部分成功不回滚

同一次归档中多个格式是独立的：handleScreenshotAndPdf 内部用 `Promise.allSettled`、handleMonolith 单独 `.catch`。截图失败不影响 PDF，PDF 失败不影响 Readability。已完成的格式不会因为其他格式失败而被清理。

这是**有意的设计选择**：最大化"至少保留一种归档格式"的概率。代价是需要用 `"unavailable"` 标记未产出的格式，并配合 finally 中的状态二次确认避免误判。

### 6.4 SSRF 防护提前终止

[archiveHandler.ts#L32-L61](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/archiveHandler.ts#L32-L61) 中对非 http(s) URL 或 SSRF 风险 URL，直接批量写 `"unavailable"` 并 return，**不走浏览器流程**，避免副作用扩散。

---

## 七、风险与潜在改进点

| 风险点 | 影响 | 建议 |
|---|---|---|
| 任务领取无行级锁 | 多 worker 实例下重复归档 | 改用 `SELECT ... FOR UPDATE SKIP LOCKED` 或引入 Redis 分布式锁 |
| 绝大多数业务失败不自动重试 | 暂时性网络抖动也被永久标 unavailable | 增加 `retryCount` / `nextRetryAt`，区分"永久失败"与"临时失败"，仅对 5xx/网络异常重试 |
| AI 标签失败也标记 `aiTagged=true` | AI API 抖动导致的失败无法自动重试 | 区分成功/失败标记，或增加重试次数字段 |
| RSS 新建 link 与更新 `lastBuildDate` 非事务 | 崩溃可能漏更时间戳，下次重复创建 link | 包事务；或对 link 加 (url, collectionId) 唯一约束 |
| 无指数退避 + 最大重试次数 | 若未来启用重试，持续失败的链接每轮都占浏览器资源 | 重试间隔指数增长，超过阈值写 `"unavailable"` |
| 孤儿文件（DB 回滚但文件已写入） | 占用磁盘 | 定期扫描 `archives/` 目录与 DB 交叉校验 |
| 同批次内重复 ID 靠内存 Set 去重 | 多实例间不生效 | 数据库层加唯一约束或利用事务 |
