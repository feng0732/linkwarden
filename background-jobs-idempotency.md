# 后台任务的重试与幂等保证分析

## 一、整体架构概览

Linkwarden 的后台任务系统由独立的 `apps/worker` 进程承载，通过 `index.ts` 中的 supervisor 模式（进程崩溃后 5 秒自动重启）保障持续运行。核心 worker 入口在 [worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/worker.ts#L1-L22)，初始化后并行启动 5 类任务：

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

核心实现在 [getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/getLinkBatchFairly.ts#L1-L179)。

**领取流程：**

1. **资格筛选**：先查询有"待处理链接"的用户，同时校验订阅状态（付费 / 试用期内 / 邮箱已验证）
2. **用户轮询**：按 `lastPickedAt` 升序取用户（从未被挑过的用户优先，nulls first），实现用户间的公平调度
3. **链接分配**：每个用户按固定配额 `linksPerUser = floor(maxBatchLinks / users.length)` 轮询取链接，用 `Set<number>` 去重防止同批次重复
4. **更新水位**：用 `updateMany` 批量把涉及用户的 `lastPickedAt` 设为当前时间

**关键查询条件（mode = "links"）：**

```
url != null AND lastPreserved == null
```

**关键查询条件（mode = "tags"）：**

```
url != null AND type = "url" AND lastPreserved != null AND aiTagged = false
AND collection.owner.aiTaggingMethod != "DISABLED"
```

### 2.2 搜索索引：简单批次 `getLinkBatch`

[getLinkBatch.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/getLinkBatch.ts#L11-L51) 从 ID 两端交替取数据（一半升序、一半降序），用内存 filter 去重，避免"最新/最老"饥饿问题。

### 2.3 领取阶段的并发风险 ⚠️

**当前实现未使用数据库行锁（SELECT ... FOR UPDATE/SKIP LOCKED）。** 在多 worker 实例部署时，多个进程可能读到相同的 `lastPreserved = null` 链接，导致重复归档。单实例部署下无此问题。

---

## 三、重复保护（幂等键设计）

每个 worker 都用"状态字段 + 条件查询"构成天然的幂等屏障。

### 3.1 链接归档的幂等防护

在 [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/archiveHandler.ts#L25-L231) 中，**每个子格式都在执行前做存在性检查**：

```ts
// 截图 / PDF
if (archivalSettings.archiveAsScreenshot && !link.image?.startsWith("archive")) { ... }
if (archivalSettings.archiveAsPDF && !link.pdf?.startsWith("archive")) { ... }

// 预览 / 可读性 / Monolith
if (!link.preview) await handleArchivePreview(link, page);
if (archivalSettings.archiveAsReadable && !link.readable) await handleReadability(content, link);
if (archivalSettings.archiveAsMonolith && !link.monolith && link.url) { ... }
```

配合查询条件 `lastPreserved = null`，链接一旦完成归档（哪怕部分格式失败被标为 "unavailable"），就不会再进入下一轮。

### 3.2 "unavailable" 标记的含义

`finally` 块中，对缺失的格式字段写入字符串 `"unavailable"`：

```ts
await prisma.link.update({
  where: { id: link.id },
  data: {
    lastPreserved: new Date().toISOString(),
    readable: !finalLink.readable ? "unavailable" : undefined,
    image:    !finalLink.image    ? "unavailable" : undefined,
    monolith: !finalLink.monolith ? "unavailable" : undefined,
    pdf:      !finalLink.pdf      ? "unavailable" : undefined,
    preview:  !finalLink.preview  ? "unavailable" : undefined,
    indexVersion: null,
  },
});
```

这是一个**终止标记**——写入后该格式永远不会被重试。统计接口 [getWorkerStats.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/web/lib/api/controllers/worker/getWorkerStats.ts#L5-L69) 也据此区分 `done` 和 `failed`。

### 3.3 搜索索引的版本号幂等

用常量 `MEILI_INDEX_VERSION`（见 [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/packages/lib/constants.ts)）作为 schema 版本戳：

- 查询条件：`indexVersion != MEILI_INDEX_VERSION OR indexVersion IS NULL`
- 写入 Meilisearch 成功后，批量把 DB 中这些链接的 `indexVersion` 更新为当前版本号

版本号变更即可触发全量重建，无需额外清理逻辑。

### 3.4 试用邮件的单次标记

`trialEndEmailWorker` 用 `trialEndEmailSent = false` 过滤，发送成功后才批量置 `true`。**发送失败不入 `processedIds`，下轮自动重试。**

### 3.5 RSS 的时间戳去重

[rssHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/packages/lib/rssHandler.ts#L7-L100) 比较 feed 的 `lastBuildDate` 与本地存储值：

```ts
if (!rssSubscription.lastBuildDate ||
    new Date(rssSubscription.lastBuildDate) < new Date(feedLastPubDate)) {
  // 只筛选 pubDate > lastBuildDate 的 item 创建 link
  ...
  // 所有新 link 创建完才更新时间戳
  await prisma.rssSubscription.update({ data: { lastBuildDate: new Date(feedLastPubDate) } });
}
```

**注意**：这里创建 link 和更新 `lastBuildDate` 不在同一事务中，崩溃后可能产生重复 link（但用户侧有 `preventDuplicateLinks` 选项可一定程度缓解）。

### 3.6 迁移任务的状态机

[AppMigration](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/packages/prisma/schema.prisma#L296-L308) 模型用 `PENDING → APPLIED / FAILED` 三态保证迁移最多执行一次；`upsert` 初始化 + 按 id 顺序执行 + 成功写入 APPLIED 后跳过。

---

## 四、重试条件与策略

### 4.1 隐式重试（无计数器、无退避）

所有 worker 都是 **while(true) + delay(interval)** 的轮询模型，没有显式的 `retry_count`、`next_retry_at` 字段或指数退避。**重试完全依赖"状态字段未被设置"这一事实。**

| 场景 | 是否重试 | 原因 |
|---|---|---|
| 归档在 `lastPreserved` 写入前崩溃 | ✅ 是 | `lastPreserved` 仍为 null，下一轮继续 |
| 归档在某个子格式（如截图）中途失败，异常抛出到 `linkProcessing` 的 catch | ✅ 是 | `finally` 未执行（异常在 promise 中被 reject，但 `Promise.allSettled` 只是吞掉）→ `lastPreserved` 仍为 null |
| 归档所有子格式跑完后进入 `finally`，即使部分失败也写了 `"unavailable"` | ❌ 否 | `lastPreserved` 已设置，格式已被永久标为 unavailable |
| Meilisearch `addDocuments` 成功但 DB `updateMany` 前崩溃 | ✅ 是 | `indexVersion` 未更新，下轮继续 add（Meilisearch 按主键 id upsert，天然幂等） |
| 邮件发送过程中异常 | ✅ 是 | 该用户 id 不加入 `processedIds`，`trialEndEmailSent` 仍为 false |
| AI 打标签 API 调用失败 | ❌ 否 | `finally` 块无论成功失败都把 `aiTagged` 设为 true |

### 4.2 浏览器超时保护

[archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/archiveHandler.ts#L63-L75) 用 `AbortController` + `setTimeout` 做单链接级超时（默认 5 分钟，`BROWSER_TIMEOUT` 环境变量），`Promise.race` 抢占。超时后抛错，由上层 `catch` 记录并触发重试。

浏览器实例自身每 30 分钟重启一次（`BROWSER_MAX_AGE_MS`），防止内存泄漏。

### 4.3 进程级自动重启

[index.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/index.ts#L1-L16) 中 supervisor 监听 worker 子进程 exit，5 秒后重新 spawn，保证 worker 崩溃不会导致整体任务中断。

---

## 五、副作用控制

### 5.1 文件写入的幂等

所有归档产物路径都由 `link.id + collectionId` 唯一确定，例如：

```
archives/{collectionId}/{linkId}.jpeg
archives/{collectionId}/{linkId}.pdf
archives/{collectionId}/{linkId}.html
archives/{collectionId}/{linkId}_readability.json
archives/preview/{collectionId}/{linkId}.jpeg
```

重复执行时，`createFile` 直接**覆盖**同路径文件，不会产生垃圾文件。删除时 [manageFiles.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/packages/filesystem/manageFiles.ts#L4-L68) 也按同样规则定位。

### 5.2 写操作顺序：先文件后 DB

以 Monolith 为例（[handleMonolith.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/preservationScheme/handleMonolith.ts#L57-L66)）：

```ts
await createFile({ data: html, filePath: `archives/${link.collectionId}/${link.id}.html` });
await prisma.link.update({ where: { id: link.id }, data: { monolith: `archives/...` } });
```

**先落盘，再写 DB 路径引用**。若 DB 写入失败，文件可能成为孤儿，但 DB 绝不会指向不存在的文件（避免 404）。

### 5.3 部分成功不回滚

同一次归档中多个格式是独立的 `Promise.allSettled`，截图失败不影响 PDF，PDF 失败不影响 Readability。已完成的格式不会因为其他格式失败而被清理。

这是**有意的设计选择**：最大化"至少保留一种归档格式"的概率。代价是需要用 `"unavailable"` 标记未产出的格式，避免 UI 层误判。

### 5.4 标签写入的 connectOrCreate

[autoTagLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/autoTagLink.ts#L154-L178) 中：

```ts
tags: {
  connectOrCreate: tags.map((tag) => ({
    where: { name_ownerId: { name: tag, ownerId: user.id } },
    create: { name: tag, owner: { connect: { id: user.id } }, aiGenerated: true },
  })),
}
```

利用 Prisma `connectOrCreate` + DB 唯一约束 `@@unique([name, ownerId])`，保障标签不会重复创建，重复执行结果一致。

### 5.5 Meilisearch 的主键幂等

Meilisearch `addDocuments` 以 `id` 为主键执行 upsert 语义，重复投递同一文档不会产生副本，仅覆盖最新内容。

### 5.6 SSRF 防护提前终止

[archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/39-linkwarden/apps/worker/lib/archiveHandler.ts#L32-L61) 中对非 http(s) URL 或 SSRF 风险 URL，直接批量写 `"unavailable"` 并 return，**不走浏览器流程**，避免副作用扩散。

---

## 六、风险与潜在改进点

| 风险点 | 影响 | 建议 |
|---|---|---|
| 任务领取无行级锁 | 多 worker 实例下重复归档 | 改用 `SELECT ... FOR UPDATE SKIP LOCKED` 或引入 Redis 分布式锁 |
| AI 标签失败也标记 `aiTagged=true` | 失败后无法自动重试 | 区分成功/失败标记，或增加重试次数字段 |
| RSS 新建 link 与更新 `lastBuildDate` 非事务 | 崩溃可能漏更时间戳，下次重复创建 link | 包事务；或对 link 加 (url, collectionId) 唯一约束 |
| 无退避、无最大重试次数 | 持续失败的链接每轮都被捞起，浪费浏览器资源 | 增加 `retryCount` / `nextRetryAt`，超过阈值写 `"unavailable"` |
| 孤儿文件（DB 回滚但文件已写入） | 占用磁盘 | 定期扫描 `archives/` 目录与 DB 交叉校验 |
| 同批次内重复 ID 靠内存 Set 去重 | 多实例间不生效 | 数据库层加唯一约束或利用事务 |
