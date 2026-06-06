# 后台任务的重试与幂等保证分析

> 所有代码引用使用仓库相对路径，格式：`路径:行号`。

## 一、整体架构概览

Linkwarden 的后台任务系统由独立的 `apps/worker` 进程承载，通过 `apps/worker/index.ts` 中的 supervisor 模式（进程崩溃后 5 秒自动重启）保障持续运行。核心 worker 入口在 `apps/worker/worker.ts`，初始化后并行启动 6 类任务：

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

核心实现：`apps/worker/lib/getLinkBatchFairly.ts`

**领取流程：**

1. **资格筛选**：先查询有"待处理链接"的用户，同时校验订阅状态（付费 / 试用期内 / 邮箱已验证）
2. **用户轮询**：按 `lastPickedAt` 升序取用户（从未被挑过的用户优先，nulls first），实现用户间的公平调度
3. **链接分配**：每个用户按固定配额 `linksPerUser = floor(maxBatchLinks / users.length)` 轮询取链接，用 `Set<number>` 去重防止同批次重复
4. **更新水位**：用 `updateMany` 批量把涉及用户的 `lastPickedAt` 设为当前时间

**关键查询条件（mode = "links"）**——这也是自动重试的**唯一**判断依据：

```ts
// apps/worker/lib/getLinkBatchFairly.ts:L35-L38
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

`apps/worker/lib/getLinkBatch.ts` 从 ID 两端交替取数据（一半升序、一半降序），用内存 filter 去重，避免"最新/最老"饥饿问题。

### 2.3 领取阶段的并发风险 ⚠️

**当前实现未使用数据库行锁（SELECT ... FOR UPDATE/SKIP LOCKED）。** 在多 worker 实例部署时，多个进程可能读到相同的 `lastPreserved = null` 链接，导致重复归档。单实例部署下无此问题。

---

## 三、请求头探测、页面导航与异常冒泡的真实关系

本章是重新核对后的核心结论，围绕 `apps/worker/lib/archiveHandler.ts` 展开。理解这一章是准确判断"是否会自动重试"的前提。

### 3.1 archiveHandler 的三段式控制流

```
archiveHandler(link, browser)
│
├─ 段 A：try 之前的初始化与短路分支（L29-L108）
│    ├─ SSRF 安全检查 / 非 http(s) URL → 直接写 lastPreserved + 全 unavailable，return
│    ├─ 创建 AbortController + timeoutPromise（5 分钟全局超时）
│    ├─ browser.newContext → protectPageRequests → context.newPage
│    ├─ createFolder × 2（archives/ 与 archives/preview/）
│    └─ 解析 archivalSettings
│
├─ 段 B：try 块（L109-L198）
│    └─ Promise.race([ 主归档 IIFE ,  timeoutPromise  ])
│         ├─ determineLinkType → fetchHeaders (HEAD 请求探测 content-type)
│         ├─ imageHandler / pdfHandler (直链资源走 safeFetch)
│         ├─ page.goto(link.url) 浏览器导航
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

### 3.2 请求头探测 fetchHeaders：异常被完全吞掉，永不冒泡

`apps/worker/lib/fetchHeaders.ts` 的完整结构：

```ts
export default async function fetchHeaders(url: string) {
  try {
    const responsePromise = safeFetch(url, { method: "HEAD" });
    const timeoutPromise = new Promise((_, reject) => {
      setTimeout(() => reject(new Error("Fetch header timeout")), 10 * 1000);
    });
    const response = await Promise.race([responsePromise, timeoutPromise]);
    return (response as Response)?.headers || null;
  } catch (err) {
    console.log(err);
    return null;   // ★ 任何异常都吞掉，返回 null
  }
}
```

**关键点：**
- 整个函数被 try/catch 包裹，**任何异常（网络错误、超时、SSRF 拦截）都被 catch 住并 return null**，绝不会向外冒泡
- 即使 `safeFetch` 返回 4xx/5xx，也正常取 headers 返回——`safeFetch` 对 HTTP 状态码不抛异常（见 3.3）

调用方 `determineLinkType`（`apps/worker/lib/archiveHandler.ts:L234-L263`）拿到 null headers 时，`contentType` 为 undefined，所有 if 判断都不命中，`linkType` 保持默认值 `"url"`，然后写一次 DB，**继续正常走后续的 `page.goto` 流程**。

→ **fetchHeaders 的失败（包括 HEAD 请求的 404/500、10 秒超时、DNS 失败）完全不会导致归档抛异常，也不会阻止后续流程。**

### 3.3 safeFetch：HTTP 状态码 4xx/5xx 不抛异常

`packages/lib/safeFetch.ts` 底层使用 `node-fetch`，采用 `redirect: "manual"` 自行处理重定向。循环内：

```ts
const response = await fetch(validatedUrl.toString(), { ..., redirect: "manual" });
if (!isRedirectStatus(response.status)) {
  return response;   // 4xx/5xx 直接返回，不抛
}
```

**safeFetch 只在以下情况抛异常：**
- 网络层错误：DNS 解析失败、TCP 连接被拒绝、TLS 握手失败
- 重定向次数超过 `maxRedirects=5`
- SSRF 检查被 `assertUrlIsSafeForServerSideFetch` 拦截

**HTTP 404/500 等状态码会正常返回 Response 对象，不会抛。**

### 3.4 Playwright page.goto：HTTP 4xx/5xx 不抛异常

`apps/worker/lib/archiveHandler.ts:L129`：

```ts
await page.goto(link.url, { waitUntil: "domcontentloaded" });
```

Playwright 的 `page.goto()` 行为：
- ✅ **HTTP 4xx/5xx 正常返回 Response 对象，不抛异常**。404 页面的 HTML 会被正常加载，后续的 `page.content()`、`page.screenshot()`、`page.pdf()` 会对错误页面正常执行
- ❌ 只有以下情况抛异常：导航超时（Playwright 默认 30s）、网络层错误（DNS 失败、连接拒绝、TLS 错误）、页面崩溃

**代码中没有任何地方检查 HTTP 状态码**（对 `response.ok`、`response.status` 的 grep 结果为零）。这意味着目标站返回 404 时，系统会把 404 页面当正常内容截图、生成 PDF、解析 Readability——整个流程被视为"归档成功"。

另一个 `page.goto` 在 `apps/worker/lib/preservationScheme/handleArchivePreview.ts:L40`（用于加载 og:image），同样不检查状态码，只在网络层非 SSRF 错误时才向外冒泡。

### 3.5 各子处理器的异常冒泡全景表

下表说明"每个子步骤失败"是否会让 `archiveHandler` 的外层 try 抛异常——但**无论是否抛，只要进入了 try，finally 都会执行并写入 lastPreserved**：

| 子步骤 | 代码位置 | HTTP 4xx/5xx | 网络层错误 (DNS/连接拒绝/TLS) | 超时 | SSRF/安全检查 | 是否冒泡到外层 try |
|---|---|---|---|---|---|---|
| HEAD 请求探测 | `fetchHeaders` → `safeFetch` | ❌ 不抛，正常取 headers | ❌ 被 `fetchHeaders` 的 catch 吞，return null | ❌ 被 catch 吞，return null | ❌ 被 catch 吞，return null | **❌ 永不冒泡** |
| 链接类型判定 | `determineLinkType` | ❌ 不抛（基于 fetchHeaders 返回值） | ❌ 不抛 | ❌ 不抛 | ❌ 不抛 | **❌ 永不冒泡** |
| 图片直链下载 | `imageHandler` → `safeFetch` | ❌ 不抛，4xx body 当图片写盘 | ✅ 抛 | ✅ TCP/HTTP 超时抛 | ✅ 抛（createAgent 内 SSRF 检查） | **✅ 冒泡** |
| PDF 直链下载 | `pdfHandler` → `safeFetch` | ❌ 不抛，4xx body 当 PDF 写盘 | ✅ 抛 | ✅ 抛 | ✅ 抛 | **✅ 冒泡** |
| 浏览器主导航 | `page.goto(link.url)` | ❌ 不抛，错误页面正常加载 | ✅ 抛 | ✅ Playwright 默认 30s 导航超时抛 | ✅ 可能（protectPageRequests 路由拦截 abort） | **✅ 冒泡** |
| OG 图片导航 | `handleArchivePreview` → `page.goto` | ❌ 不抛 | ✅ 抛（非 SSRF 异常显式向外 throw） | ✅ 抛 | ❌ UnsafeUrlError 被 catch 吞 | **✅ 部分冒泡** |
| 可读性解析 | `handleReadability`（纯内存 JSDOM） | N/A | N/A | N/A | N/A | **⚠️ 解析异常会冒泡，但 Buffer 超限直接 return 不抛** |
| 截图/PDF 生成 | `handleScreenshotAndPdf` | N/A（基于已加载页面） | N/A | N/A | N/A | **❌ 内部 Promise.allSettled + .then()，失败只影响自身字段** |
| Monolith 归档 | `handleMonolith` | N/A | N/A（子进程非 0 退出码 reject） | N/A（AbortController 只杀子进程） | N/A | **❌ 末尾 .catch(err => console.error(err)) 完全吞掉** |
| 浏览器全局超时 | `timeoutPromise`（默认 5 分钟） | N/A | N/A | **✅ 显式 reject** | N/A | **✅ 冒泡** |

### 3.6 浏览器超时的完整触发链

超时由 `apps/worker/lib/archiveHandler.ts:L63-L75` 驱动：

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
4. 异常继续向上冒泡到 `apps/worker/workers/linkProcessing.ts:L58-L68` 的 `archiveLink` catch 块，记录错误并检查浏览器连接，但**不再抛出**
5. 最外层 `Promise.allSettled` 只是保险——异常已经在 `archiveLink` 层被吞掉

注意：`abortController.signal` 只传给了 `handleMonolith`，其他子流程（page.goto、截图、PDF、Readability）对超时不感知，它们可能在后台继续跑一会儿，但 `Promise.race` 已经让主流程结束并进入 finally。

### 3.7 Promise.race 并发收尾窗口：timeout 后 IIFE 仍在后台执行

`Promise.race([主归档 IIFE, timeoutPromise])` 的一个重要特性是：**被"输掉"的 Promise 不会被取消**。当 timeoutPromise 先 reject 时，主归档 IIFE 仍然在后台继续执行，这会产生几类竞态风险：

**风险 1：子处理器独立写 DB 与 finally 写 "unavailable" 的竞态**

多个子处理器在执行过程中各自独立调用 `prisma.link.update` 写自己的字段：

| 子处理器 | 独立写 DB 的位置 | 写入字段 |
|---|---|---|
| `determineLinkType` | `apps/worker/lib/archiveHandler.ts:L257-L262` | `type` |
| 主流程 | `apps/worker/lib/archiveHandler.ts:L158-L164` | `metaDescription` |
| `imageHandler` | `apps/worker/lib/preservationScheme/imageHandler.ts:L28-L33` | `image` |
| `pdfHandler` | `apps/worker/lib/preservationScheme/pdfHandler.ts:L25-L30` | `pdf` |
| `handleArchivePreview` | `apps/worker/lib/preservationScheme/handleArchivePreview.ts:L73-L78` | `preview` |
| `handleReadability` | `apps/worker/lib/preservationScheme/handleReadability.ts:L98-L103` | `readable` |
| `handleScreenshotAndPdf` | 内部 `page.screenshot().then(...)` / `page.pdf().then(...)` | `image` / `pdf` |
| `handleMonolith` | `apps/worker/lib/preservationScheme/handleMonolith.ts:L57-L66` | `monolith` |

假设以下时序：
1. timeoutPromise reject → Promise.race 结束 → 进入 finally
2. finally 在 L208 执行 `prisma.link.findUnique` 拿到 finalLink（此时 `image` 仍为 null）
3. 后台 IIFE 中 `imageHandler` 执行完毕，独立写 DB：`image = "archives/1/2.jpeg"`
4. finally 在 L213 执行 `prisma.link.update`，按条件 `!finalLink.image` → 把 `image` 覆盖写成 `"unavailable"`

结果：文件实际已落盘（`archives/1/2.jpeg`），但 DB 里 `image` 是 `"unavailable"`，产生**孤儿文件 + 状态不一致**。

反向时序（子处理器后写）的话，DB 里保留的是正确路径，反而没问题。

**风险 2：后台 Playwright 操作被 context.close() 暴力中断**

finally 中的 `context.close()`（`apps/worker/lib/archiveHandler.ts:L229`）会关闭整个浏览器上下文。如果此时后台 IIFE 正在执行 `page.screenshot()`、`page.pdf()`、`page.content()`、`page.evaluate()` 等操作，Playwright 会抛出类似"Target page, context or browser has been closed"的异常：
- 子处理器自带 `.catch`（如 handleMonolith）→ 异常被吞
- 子处理器没有 `.catch`（如 handleReadability、handleArchivePreview）→ 异常在已 resolve 的 Promise 链中变成 **unhandledRejection**，可能导致 Node.js 进程告警（取决于 `process.on("unhandledRejection")` 的处理）
- `imageHandler` 和 `pdfHandler` 是直链下载，不依赖 Playwright context，不受 `context.close()` 影响

**风险 3：sendToWayback 完全不受控**

`apps/worker/lib/archiveHandler.ts:L119` 的 `sendToWayback(link.url)` 没有 await，是 fire-and-forget。timeout 抢占后它仍在后台发起对外 HTTP 请求，但不写 DB 也不写文件，影响有限。

### 3.8 protectPageRequests 请求拦截对归档内容完整性的影响

`apps/worker/lib/protectPageRequests.ts` 通过 Playwright 的 `context.route("**/*", ...)` 拦截**该上下文发起的所有网络请求**（包括主文档、子资源 CSS/JS/图片/font、XHR/Fetch、WebSocket 等）：

```ts
await context.route("**/*", async (route: Route) => {
  const requestUrl = route.request().url();
  if (isNonNetworkUrl(requestUrl)) {   // about:  blob:  data:
    await route.continue();
    return;
  }
  try {
    await assertUrlIsSafeForServerSideFetch(requestUrl);  // SSRF 检查
    await route.continue();
  } catch (error) {
    if (error instanceof UnsafeUrlError) {
      await route.abort("blockedbyclient");   // ★ 被拦截的请求直接 abort
      return;
    }
    throw error;   // 非 SSRF 异常在 route handler 中 throw
  }
});
```

对归档完整性的影响：

| 场景 | 表现 | 后果 |
|---|---|---|
| 页面引用内网 IP 的 CSS/JS（如 `http://127.0.0.1/style.css`） | 请求被 `route.abort("blockedbyclient")` | 截图/PDF 样式错乱、交互脚本不执行 |
| 页面引用内网域名的图片/font | 请求被 abort | 图片位置留白、字体回退到系统默认 |
| SPA 通过 Fetch/XHR 请求内网 API 渲染动态内容 | 请求被 abort | 动态区域为空或显示加载错误 |
| og:image 指向内网 | 主流程 SSRF 已先挡；即使过了，route 也会 abort | preview 退化到 fallback 用 `page.screenshot()` 截整页缩略图，内容可能不完整 |
| `assertUrlIsSafeForServerSideFetch` 内部抛出**非** `UnsafeUrlError`（如 DNS 解析临时故障、依赖服务不可用） | 异常在 route handler 中 throw，Playwright 内部处理 | 该请求失败，页面渲染缺资源；**但不会冒泡到 archiveHandler 的外层 try**（因为 route callback 是 Playwright 内部调度的异步回调） |

**关键点**：protectPageRequests 拦截的是浏览器上下文的**子资源请求**，主文档 `link.url` 的 SSRF 检查在段 A（`apps/worker/lib/archiveHandler.ts:L32-L42`）已经提前完成——两者是双层防护。子资源被 abort 不会让 `page.goto()` 抛异常（Playwright 不会因为子资源加载失败拒绝主文档导航），所以归档流程会继续执行，最终产出**视觉/内容不完整但状态标记为"成功"**的归档。

### 3.9 finally 内部如何写入状态

`apps/worker/lib/archiveHandler.ts:L208-L224`：

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
- 但如 3.7 所述，`findUnique` 和后续 `update` 之间存在**读-改-写竞态窗口**，后台子处理器的并发写入可能被 finally 覆盖成 "unavailable"

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

### 4.2 自动重试的真正边界：段 A 逐行拆解

`getLinkBatchFairly` 只看 `lastPreserved == null`。结合 3.1 的控制流，段 A（`apps/worker/lib/archiveHandler.ts:L29-L108`）的每一步都可能在进入 try 之前抛异常，从而让 `lastPreserved` 保持 null 并触发下一轮自动重试。逐行分析：

| 段 A 代码行 | 操作 | 是否可能抛异常 | 抛异常时的后果 |
|---|---|---|---|
| L32-L42 | `assertUrlIsSafeForServerSideFetch(link.url)` | ✅ **可能**：如果抛出 `UnsafeUrlError` 之外的异常（例如 SSRF 检查依赖的 DNS 解析失败、底层网络错误），`else { throw error }` 分支会向外抛出 | 异常冒泡到 `archiveLink` catch；`lastPreserved` 未写 → **自动重试** |
| L44-L61 | SSRF 命中 / 非 http(s) 直接写 DB + return | ❌ 不抛（写 DB 本身可能抛，但无 catch）；如果写 DB 抛异常则冒泡 | 如果 `prisma.link.update` 抛异常 → `lastPreserved` 未写 → **自动重试** |
| L63-L75 | 创建 AbortController 与 timeoutPromise | ❌ 纯同步操作，不抛 | — |
| L77-L78 | `browser.newContext(contextOptions)` | ✅ **可能**：Playwright 浏览器进程崩溃、资源不足、WS 连接断开 | 异常冒泡；`lastPreserved` 未写 → **自动重试**；⚠️ **context 可能已部分创建但未被关闭（无 finally），造成浏览器上下文泄漏** |
| L79 | `protectPageRequests(context)` 注册路由 | ⚠️ 极低概率：如果 context 已关闭，`context.route()` 可能抛 | 异常冒泡；同上有 context 泄漏风险 |
| L80 | `context.newPage()` | ✅ **可能**：Playwright 内部错误、上下文已失效 | 异常冒泡；**context 已创建但不会被关闭，造成泄漏**；`lastPreserved` 未写 → **自动重试** |
| L82-L83 | `createFolder(...)` × 2（`packages/filesystem/createFolder.ts:L5-L18`） | ✅ **可能**：`fs.mkdirSync` 在磁盘权限不足、磁盘满、I/O 错误时同步抛出 | 异常冒泡；context 已创建但不关闭 → **自动重试 + context 泄漏** |
| L85-L107 | 计算 archivalSettings（读 link.tags / 用户设置） | ❌ 纯内存操作，不抛 | — |

**段 A 失败的共同副作用**：`browser.newContext()` 之后任何一步抛异常，因为还没进入 try，`finally` 里的 `context.close()` 不会执行，browser context 残留在 Chromium 进程中占用资源。worker 的浏览器 30 分钟整体重启（`apps/worker/workers/linkProcessing.ts:L9,L18-L33` 的 `BROWSER_MAX_AGE_MS` 常量 + `restartBrowser` 函数）是兜底清理手段。

综上，**真正会触发自动重试的只有 3 类场景**：

| # | 场景 | 为什么 lastPreserved 没写 |
|---|---|---|
| 1 | **段 A 初始化抛异常**（上表列举的 assertUrl 非 SSRF 异常、newContext/newPage 失败、createFolder 失败、段 A 内 prisma.update 失败） | 没走到 finally；也没走到 SSRF 早期分支的 update |
| 2 | **进程级崩溃**：OOM kill、SIGKILL、宿主机断电 | DB 写入未发生 |
| 3 | **finally 内部的 prisma.update 自身失败**：DB 连接断开、事务冲突等 | finally 尝试执行但 DB 操作报错 |

**所有其他情况——包括 HTTP 404/500、页面导航超时、5 分钟浏览器全局超时、DNS 失败、Readability 解析异常、imageHandler/pdfHandler 网络错误——都会进入 finally 并写入 `lastPreserved = now`，因此不会自动重试。**

### 4.3 精确场景决策表

| 场景 | 是否进入 try | finally 是否执行 | lastPreserved 结果 | 格式字段状态 | 下一轮是否自动重试 |
|---|---|---|---|---|---|
| 所有格式完全成功 | ✅ | ✅ | ✅ 写入时间戳 | 都是 `archives/...` 路径 | ❌ 不重试 |
| 目标站返回 HTTP 404/500（page.goto 正常加载错误页） | ✅ | ✅ | ✅ 写入时间戳 | 对错误页面生成的 archive 路径 | ❌ 不重试 |
| HEAD 请求探测失败（fetchHeaders 超时/DNS 失败） | ✅ | ✅ | ✅ 写入时间戳 | 继续走 page.goto，按实际产出写路径或 unavailable | ❌ 不重试 |
| page.goto 导航超时（Playwright 默认 30s） | ✅ | ✅ | ✅ 写入时间戳 | 已完成的保留路径，其余写 `"unavailable"` | ❌ 不重试 |
| safeFetch 网络层错误（imageHandler/pdfHandler DNS/连接失败） | ✅ | ✅ | ✅ 写入时间戳 | 之前完成的保留路径，该直链格式写 `"unavailable"` | ❌ 不重试 |
| handleReadability 解析抛异常 | ✅ | ✅ | ✅ 写入时间戳 | 之前完成的保留路径，`readable=null→"unavailable"` | ❌ 不重试 |
| handleScreenshotAndPdf 内部截图失败 | ✅ | ✅ | ✅ 写入时间戳 | `image=null→"unavailable"`，其他按实际 | ❌ 不重试 |
| handleMonolith 失败（被 .catch 吞） | ✅ | ✅ | ✅ 写入时间戳 | `monolith=null→"unavailable"`，其他按实际 | ❌ 不重试 |
| 浏览器 5 分钟全局超时（timeoutPromise reject） | ✅ | ✅ | ✅ 写入时间戳 | 已完成的保留路径，未完成的写 `"unavailable"`；可能存在 3.7 的竞态覆盖 | ❌ 不重试 |
| SSRF / 非 http(s)（早期 return 分支） | ❌（在 try 之前 return） | ❌ | ✅ 在段 A 直接写入 | 全部写 `"unavailable"` | ❌ 不重试 |
| browser.newContext() 失败（段 A 抛错） | ❌（还没进入 try） | ❌ | ❌ 未写入 | 保持原样（全 null） | ✅ **会重试** |
| createFolder 磁盘权限不足（段 A 抛错） | ❌（还没进入 try） | ❌ | ❌ 未写入 | 保持原样（全 null） | ✅ **会重试** |
| 进程 OOM / SIGKILL，死在 try 执行过程中 | ✅（已进入 try） | ❌（进程终止） | ❌ 未写入 | 部分格式可能已写路径，其余 null | ✅ **会重试**（但已写路径的格式下次会被跳过） |
| finally 的 prisma.update 自身报错 | ✅ | ✅（尝试执行） | ❌ 写入失败 | 保持 try 期间各子处理器已写入的状态 | ✅ **会重试** |

### 4.4 重试过程中的格式级"断点续传"

场景"进程 OOM 死在 try 中间"值得单独说明：假设 imageHandler 已成功把截图写入 DB（`image = archives/.../1.jpeg`），但 handleMonolith 还没跑进程就死了。下一轮重试时：

1. `lastPreserved` 仍为 null → 链接会被再次捞起
2. 进入 archiveHandler，try 前的 `!link.image?.startsWith("archive")` 为 false → **跳过截图**
3. `!link.monolith` 为 true → **继续尝试 Monolith**
4. finally 写入 lastPreserved 和剩余的 `"unavailable"`

这是一种天然的**格式级断点续传**——已成功的格式不会重复生成。

### 4.5 为什么设计成"几乎不自动重试"

从代码看是有意选择：
1. **Playwright 浏览器资源昂贵**：反复重试失败链接会长时间占用 browser context
2. **失败原因多为不可自动恢复**：目标站点 404、被墙、内容类型异常、SSRf 拦截——这些靠重试几乎不会成功
3. **4xx/5xx 不视为失败**：系统把 404 页面当有效内容归档，不给用户留"空白"
4. **提供了人工重试入口**：`/api/v1/worker/preservation` DELETE 接口可以把已归档链接的格式字段清空，让它们重新进入待处理队列

代价是：暂时性网络抖动导致的真正失败（如 page.goto 的 DNS 解析失败、safeFetch 的连接超时）也被永久标记为 unavailable，需要人工触发重跑。

### 4.6 人工重跑入口详解

人工重跑通过 `DELETE /api/v1/worker/preservation` 接口（`apps/web/pages/api/v1/worker/preservation.tsx`）调用，前端入口在 `apps/web/pages/admin/background-jobs.tsx`，仅限服务器管理员（`user.id === NEXT_PUBLIC_ADMIN`）使用。Schema 定义 `packages/lib/schemaValidation.ts:L304-L306` 支持两种 action：

#### action = "allAndRePreserve"：全量重跑

```ts
// apps/web/pages/api/v1/worker/preservation.tsx:L36-L66
for (const link of allLinks) {
  await removeFiles(link.id, link.collectionId);   // 删除磁盘上所有归档文件
  await prisma.link.update({
    where: { id: link.id },
    data: {
      image: null, pdf: null, readable: null, monolith: null, preview: null,
      lastPreserved: null,    // 重新进入待处理队列
      indexVersion: null,     // 重新索引
    },
  });
}
```

- 作用范围：管理员自己拥有的全部 `type = "url"` 链接
- 行为：**先删文件，再把所有格式字段 + lastPreserved + indexVersion 全部置 null**
- 效果：下一轮 worker 轮询时这些链接就像从未归档过一样，全部格式从头生成

#### action = "allBroken"：仅重跑失败项

```ts
// apps/web/pages/api/v1/worker/preservation.tsx:L67-L164
// 第一步：筛选有任一格式等于 "unavailable" 的链接
OR: [{ image: "unavailable" }, { pdf: "unavailable" }, ...]

// 第二步：结合用户设置判断是否真的需要该格式
needsReprocessing =
  (link.image === "unavailable" && shouldArchive.archiveAsScreenshot) ||
  (link.pdf   === "unavailable" && shouldArchive.archiveAsPDF)        || ...

// 第三步：只把需要且失败的格式字段置 null
if (needsReprocessing) {
  await prisma.link.update({
    data: {
      image:    shouldArchive.archiveAsScreenshot && link.image    === "unavailable" ? null : link.image,
      pdf:      shouldArchive.archiveAsPDF        && link.pdf      === "unavailable" ? null : link.pdf,
      readable: shouldArchive.archiveAsReadable   && link.readable === "unavailable" ? null : link.readable,
      monolith: shouldArchive.archiveAsMonolith   && link.monolith === "unavailable" ? null : link.monolith,
      lastPreserved: null,   // ★ 只要 needsReprocessing，就整体重置
      indexVersion: null,
    },
  });
}
```

- 作用范围：仅筛选 5 个格式字段中**至少有一个值为 `"unavailable"`** 的链接
- 行为：
  - **不删文件**：已是 archive 路径的字段（部分成功的产物）保留不动，对应磁盘文件也保留
  - **精确重置**：只把「用户设置启用了该格式 **且** 当前值为 `"unavailable"`」的字段置 null
  - **整体重入队列**：`lastPreserved` 一律置 null，让链接重新进入待处理
  - **选择性跳过重跑**：如果 `needsReprocessing` 为 false（例如用户当前已禁用截图，但历史上截图失败被标 unavailable），则完全不动
- 效果：下一轮 worker 轮询时，未失败的格式会因为字段仍是 `archives/...` 路径被跳过（断点续传），失败的格式重新尝试生成

**两种 action 的共同特点**：都只重置 DB 字段，不向 worker 发通知——worker 在下一次轮询（默认 **10 秒**，`apps/worker/worker.ts:L8-L9` 的 `workerIntervalInSeconds = Number(process.env.ARCHIVE_SCRIPT_INTERVAL) || 10`，通过 `apps/worker/workers/linkProcessing.ts:L41,L83` 的 `delay(interval)` 实现）时自然会把 `lastPreserved = null` 的链接重新捞起。

### 4.7 普通用户的归档重置路径

管理员接口（4.6）是全局批量重置，普通用户在日常使用中有三条独立路径可以触发自己链接的归档重置：

#### 路径 1：单链接重新归档（PUT /api/v1/links/[id]/archive）

这是最常见的用户操作，前端有两个入口：
- `apps/web/components/LinkViews/LinkComponents/LinkActions.tsx:L59-L76` 链接卡片三个点菜单 → "Refresh preserved formats"
- `apps/web/components/ModalContent/LinkModal.tsx:L123-L139` 链接详情 Drawer 右上角三个点 → "Refresh preserved formats"（仅在 `link.type === "url"` 时显示）

后端实现：`apps/web/pages/api/v1/links/[id]/archive/index.ts:L39-L72`

```ts
await prisma.link.update({
  where: { id: link.id },
  data: {
    image: null, pdf: null, readable: null, monolith: null, preview: null,
    lastPreserved: null, indexVersion: null,
    clientSide: false,   // 额外重置客户端归档标记
  },
});
await removeFiles(link.id, link.collection.id);
```

| 维度 | 行为 |
|---|---|
| 权限 | collection.ownerId 或 member.canUpdate（`apps/web/pages/api/v1/links/[id]/archive/index.ts:L30-L37`） |
| 前置检查 | `link.url` 存在且 `isValidUrl(link.url)`，否则直接返回成功但不做任何事 |
| 删文件 | ✅ 调 `removeFiles` 全删 |
| 重置范围 | 5 个格式字段 + `lastPreserved` + `indexVersion` + `clientSide` **全部置 null** |
| 行为等价于 | 管理员 `allAndRePreserve` 但只针对单条链接 |

#### 路径 2：批量重新归档（DELETE /api/v1/links/archive）

前端入口：`apps/web/components/LinkListOptions.tsx:L93-L115,L176-L191`，在列表页编辑模式（铅笔图标激活）下的刷新按钮（`bi-arrow-clockwise`），先通过多选框勾选链接再点按钮触发。Schema 定义 `packages/lib/schemaValidation.ts:L296-L302` 的 `LinkArchiveActionSchema` 接受 `linkIds: number[]`。

后端实现：`apps/web/pages/api/v1/links/archive/index.ts:L11-L90`

```ts
// 先做权限过滤
const authorizedLinks = await prisma.link.findMany({
  where: {
    id: { in: linkIds },
    url: { not: null },
    OR: [
      { collection: { ownerId: user.id } },
      { collection: { members: { some: { userId: user.id, canDelete: true } } } },
    ],
  },
  select: { id: true, collectionId: true },
});

// 先返回 HTTP 200，再在后台循环处理（不阻塞响应）
res.status(200).json({ response: "Success." });
for (const link of authorizedLinks) {
  await removeFiles(link.id, collectionId);
  await prisma.link.update({
    where: { id: link.id },
    data: {
      image: null, pdf: null, readable: null, monolith: null, preview: null,
      lastPreserved: null, indexVersion: null,
    },
  });
}
```

| 维度 | 行为 |
|---|---|
| 权限 | 每条链接单独检查 collection.ownerId 或 member.canDelete；无权限的链接静默过滤掉，不返回错误 |
| 前置检查 | `url != null`（无 URL 的链接不会被处理） |
| 响应时序 | **先返回 200 再做 DB 操作**（fire-and-forget 模式）。如果循环中途 DB 抛异常，已处理的链接已重置，未处理的保持原样 |
| 删文件 | ✅ 每条都调 `removeFiles` |
| 重置范围 | 同单链接：5 个格式 + `lastPreserved` + `indexVersion` 全置 null；注意**不重置 `clientSide`**（与单链接 PUT 的细微差异） |
| 行为等价于 | 对一组授权链接逐个执行 `allAndRePreserve` |

#### 路径 3：URL 变更触发重置（updateLinkById）

当用户在编辑链接时修改了 URL，系统会自动触发全量重置。核心逻辑在 `apps/web/lib/api/controllers/links/linkId/updateLinkById.ts:L133-L163`：

```ts
// 条件判断：新 URL 存在、与旧 URL 不同、且合法
if (data.url && oldLink && oldLink?.url !== data.url && isValidUrl(data.url)) {
  await removeFiles(oldLink.id, oldLink.collectionId);   // 删旧文件
} else if (oldLink?.url !== data.url)
  return { response: "Invalid URL.", status: 401 };       // URL 变了但不合法，直接拒绝

// prisma.link.update 中对每个字段做三元判断
data: {
  image:    oldLink?.url !== data.url ? null : undefined,
  pdf:      oldLink?.url !== data.url ? null : undefined,
  readable: oldLink?.url !== data.url ? null : undefined,
  monolith: oldLink?.url !== data.url ? null : undefined,
  preview:  oldLink?.url !== data.url ? null : undefined,
  lastPreserved: oldLink?.url !== data.url ? null : undefined,
  indexVersion: null,   // ★ 无条件置 null，不管 URL 变没变
}
```

| 维度 | 行为 |
|---|---|
| 触发条件 | `data.url` 与 `oldLink.url` **字符串精确不等**且 `isValidUrl(newUrl)` |
| 非法 URL 处理 | URL 变了但 `isValidUrl` 不通过 → 返回 401，**不更新任何字段**（包括 URL 本身也不会写入） |
| 删文件 | ✅ 用旧 `linkId` + 旧 `collectionId` 删旧文件。注意：URL 变更如果伴随 collectionId 变更（移动到另一个集合），文件会先按旧路径删掉，然后 `L195-L197` 再把新集合下的文件移动过去——但因为已经删了，移动其实是 no-op |
| 重置范围 | 除 `indexVersion` 外，全部以 `oldLink?.url !== data.url` 为条件——URL 变了就置 null，没变就传 `undefined`（不更新） |
| `indexVersion` 特殊处理 | **无条件置 null**。意味着即使只改了 name/description/tags（URL 没变），也会触发 Meilisearch 重新索引（符合预期，因为搜索索引需要最新的元数据） |
| collection 移动 | 如果 collectionId 也变了，`L195-L197` 会调 `moveFiles` 尝试跨集合搬运归档文件。但 URL 变更场景下前面已经 `removeFiles` 了，所以实际不会有文件可搬 |

**批量编辑不触发 URL 重置**：`apps/web/lib/api/controllers/links/bulk/updateLinks.ts:L10-L13` 定义批量编辑只接受 `tags` 和 `collectionId` 两个字段，不允许改 URL，因此批量编辑不会触发归档重置。

### 4.8 所有重置路径的对比总表

| 路径 | 触发者 | 范围 | 删文件 | URL 必须变？ | lastPreserved 重置 | indexVersion 重置 | 其他重置 |
|---|---|---|---|---|---|---|---|
| 管理员 `allAndRePreserve` | 服务器管理员（单用户） | 全部 type="url" 链接 | ✅ | ❌ | ✅ 全量 | ✅ | — |
| 管理员 `allBroken` | 服务器管理员（单用户） | 仅含 unavailable 字段的链接 | ❌ | ❌ | ✅ 只重入队列 | ✅ | 只把「需要且 unavailable」的格式置 null |
| 单链接 PUT /api/v1/links/[id]/archive | 链接所有者 / 有 canUpdate 的成员 | 单条 | ✅ | ❌ | ✅ | ✅ | `clientSide: false` |
| 批量 DELETE /api/v1/links/archive | 链接所有者 / 有 canDelete 的成员 | 勾选的一组（权限过滤后） | ✅ | ❌ | ✅ | ✅ | — |
| URL 变更 updateLinkById | 链接所有者 / 有 canUpdate 的成员 | 单条 | ✅ | ✅（URL 字符串精确不等） | ✅（仅 URL 变时） | ✅（无条件） | — |

---

## 五、重复保护（幂等键设计）——非归档类 worker

### 5.1 搜索索引的版本号幂等

用常量 `MEILI_INDEX_VERSION`（见 `packages/lib/constants.ts`）作为 schema 版本戳：

- 查询条件：`indexVersion != MEILI_INDEX_VERSION OR indexVersion IS NULL`
- 写入 Meilisearch 成功后，批量把 DB 中这些链接的 `indexVersion` 更新为当前版本号
- 版本号变更即可触发全量重建，无需额外清理逻辑
- Meilisearch `addDocuments` 以 `id` 为主键执行 upsert 语义，重复投递同一文档不会产生副本

### 5.2 AI 自动标签的 aiTagged 标记

`apps/worker/workers/autoTagPreservedLinks.ts:L35-L73` 中每个链接的处理有一个 **finally 块无条件写 `aiTagged = true`**：

```ts
} finally {
  await prisma.link
    .update({ where: { id: link.id }, data: { aiTagged: true } })
    .catch(...);
}
```

这意味着**无论 AI API 调用成功还是失败，下一轮都不会再处理这条链接**——失败的链接同样无法自动重试（属于可改进点）。

标签写入自身用 Prisma `connectOrCreate` + DB 唯一约束 `@@unique([name, ownerId])`（见 `apps/worker/lib/autoTagLink.ts:L154-L178`），保障标签不会重复创建。

### 5.3 试用邮件的单次标记

`apps/worker/workers/trialEndEmailWorker.ts:L40-L124` 用 `trialEndEmailSent = false` 过滤。关键设计：
- 只有 `transporter.sendMail` 成功才把 userId 加入 `processedIds`
- 发送失败时 `continue`，跳过当前用户，**不加入 processedIds**
- 循环末尾才统一 `updateMany` 把 `processedIds` 中的用户标为 `trialEndEmailSent = true`

因此邮件发送失败会在下一轮自动重试，这与归档 worker 的"尽量不重试"策略正好相反。

### 5.4 RSS 的时间戳去重

`packages/lib/rssHandler.ts:L28-L91` 比较 feed 的 `lastBuildDate` 与本地存储值，只处理 `pubDate > lastBuildDate` 的新 item。**注意**：创建 link 和更新 `lastBuildDate` 不在同一事务中，崩溃后可能产生重复 link（用户侧 `preventDuplicateLinks` 选项可一定程度缓解）。

### 5.5 迁移任务的状态机

`packages/prisma/schema.prisma` 中 `AppMigration` 模型用 `PENDING → APPLIED / FAILED` 三态 + `upsert` 初始化 + 按 id 顺序执行，保证迁移最多执行一次。

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

重复执行时，`createFile` 直接**覆盖**同路径文件，不会产生垃圾文件。删除时 `packages/filesystem/manageFiles.ts` 也按同样规则定位。

### 6.2 写操作顺序：先文件后 DB

以 `apps/worker/lib/preservationScheme/handleMonolith.ts:L57-L66` 为例：

```ts
await createFile({ data: html, filePath: `archives/${link.collectionId}/${link.id}.html` });
await prisma.link.update({ where: { id: link.id }, data: { monolith: `archives/...` } });
```

**先落盘，再写 DB 路径引用**。若 DB 写入失败，文件可能成为孤儿，但 DB 绝不会指向不存在的文件（避免 UI 层 404）。

### 6.3 部分成功不回滚

同一次归档中多个格式是独立的：handleScreenshotAndPdf 内部用 `Promise.allSettled`、handleMonolith 单独 `.catch`。截图失败不影响 PDF，PDF 失败不影响 Readability。已完成的格式不会因为其他格式失败而被清理。

这是**有意的设计选择**：最大化"至少保留一种归档格式"的概率。代价是需要用 `"unavailable"` 标记未产出的格式，并配合 finally 中的状态二次确认避免误判。

### 6.4 SSRF 防护提前终止

`apps/worker/lib/archiveHandler.ts:L32-L61` 中对非 http(s) URL 或 SSRF 风险 URL，直接批量写 `"unavailable"` 并 return，**不走浏览器流程**，避免副作用扩散。

---

## 七、风险与潜在改进点

| 风险点 | 影响 | 建议 |
|---|---|---|
| 任务领取无行级锁 | 多 worker 实例下重复归档 | 改用 `SELECT ... FOR UPDATE SKIP LOCKED` 或引入 Redis 分布式锁 |
| HTTP 4xx/5xx 被当作正常内容归档 | 用户保存的是 404 页面而非目标内容 | 在 `page.goto` 后检查 `response.status()`，对 4xx/5xx 单独标记并考虑允许重试 |
| 绝大多数业务失败不自动重试 | 暂时性网络抖动也被永久标 unavailable | 增加 `retryCount` / `nextRetryAt`，仅对网络层错误和 5xx 重试，对 4xx 直接标 unavailable |
| Promise.race 后子处理器并发写 DB | finally 的 "unavailable" 可能覆盖子处理器刚写入的真实路径，产生孤儿文件 | 用事务 + `SELECT ... FOR UPDATE` 锁定行；或把各格式写入全部推迟到 finally 统一执行 |
| 段 A 失败无 context.close() | browser.newContext/newPage/createFolder 抛异常时 context 泄漏 | 把 context 创建也包进 try，或在段 A 内单独加 try/finally 关 context |
| protectPageRequests 导致子资源被 abort | 截图/PDF 样式缺失、图片空白、动态内容不完整，UI 显示"成功"但内容不可用 | 对主资源（首屏图片/字体）考虑放宽 SSRF 或使用代理；在 finally 中统计被 abort 的子资源数量，超过阈值标警告 |
| AI 标签失败也标记 `aiTagged=true` | AI API 抖动导致的失败无法自动重试 | 区分成功/失败标记，或增加重试次数字段 |
| RSS 新建 link 与更新 `lastBuildDate` 非事务 | 崩溃可能漏更时间戳，下次重复创建 link | 包事务；或对 link 加 (url, collectionId) 唯一约束 |
| 无指数退避 + 最大重试次数 | 若未来启用重试，持续失败的链接每轮都占浏览器资源 | 重试间隔指数增长，超过阈值写 `"unavailable"` |
| 孤儿文件（DB 回滚但文件已写入；或 Promise.race 竞态 finally 覆盖） | 占用磁盘 | 定期扫描 `archives/` 目录与 DB 交叉校验 |
| 同批次内重复 ID 靠内存 Set 去重 | 多实例间不生效 | 数据库层加唯一约束或利用事务 |
