# Linkwarden Headless Browser 抓取与渲染流程代码分析

> 所有路径均为仓库根目录的相对路径

---

## 一、核心文件索引

| 模块 | 文件路径 | 关键函数/类 |
|------|---------|------------|
| 进程守护 | `apps/worker/index.ts` | `launch()` |
| Worker 入口 | `apps/worker/worker.ts` | `init()` |
| 链接处理主循环 | `apps/worker/workers/linkProcessing.ts` | `linkProcessing()`, `restartBrowser()`, `archiveLink()` |
| 浏览器启动配置 | `apps/worker/lib/browser.ts` | `launchBrowser()`, `getBrowserOptions()`, `getDefaultContextOptions()` |
| 归档核心调度 | `apps/worker/lib/archiveHandler.ts` | `archiveHandler()`, `determineLinkType()` |
| 批次公平调度 | `apps/worker/lib/getLinkBatchFairly.ts` | `getLinkBatchFairly()` |
| SSRF 路由防护 | `apps/worker/lib/protectPageRequests.ts` | `protectPageRequests()` |
| 响应头探测 | `apps/worker/lib/fetchHeaders.ts` | `fetchHeaders()` |
| 预览图生成 | `apps/worker/lib/preservationScheme/handleArchivePreview.ts` | `handleArchivePreview()` |
| Readability 正文 | `apps/worker/lib/preservationScheme/handleReadability.ts` | `handleReadability()` |
| 截图与 PDF | `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts` | `handleScreenshotAndPdf()`, `autoScroll()` |
| Monolith 归档 | `apps/worker/lib/preservationScheme/handleMonolith.ts` | `handleMonolith()` |
| 图片直链下载 | `apps/worker/lib/preservationScheme/imageHandler.ts` | `imageHandler()` |
| PDF 直链下载 | `apps/worker/lib/preservationScheme/pdfHandler.ts` | `pdfHandler()` |
| Wayback 提交 | `apps/worker/lib/preservationScheme/sendToWayback.ts` | `sendToWayback()` |

---

## 二、页面抓取流程代码引用

### 2.1 Worker 启动与进程守护

**进程级自动重启** — `apps/worker/index.ts#L3-L12`：
```typescript
function launch() {
  const child = spawn("tsx", ["worker.ts"], { stdio: "inherit" });
  child.on("exit", (code, signal) => {
    console.error(`worker exited (code=${code} signal=${signal}) – restarting…`);
    setTimeout(launch, 5000);  // 5秒后重启子进程
  });
}
```

**多 Worker 并行初始化** — `apps/worker/worker.ts#L11-L20`：
```typescript
async function init() {
  await migrationWorker();
  startRSSPolling();
  linkProcessing(workerIntervalInSeconds);       // ← Headless 抓取主循环
  autoTagPreservedLinks(workerIntervalInSeconds);
  startIndexing(workerIntervalInSeconds);
  trialEndEmailWorker();
}
```

### 2.2 浏览器生命周期管理

**浏览器单例 + 30 分钟轮换** — `apps/worker/workers/linkProcessing.ts#L14-L33`：
```typescript
let browser = await launchBrowser();
let browserStartTs = Date.now();
const BROWSER_MAX_AGE_MS = 30 * 60 * 1000;

// 每轮循环检查年龄，超时则重启
if (Date.now() - browserStartTs >= BROWSER_MAX_AGE_MS) {
  await restartBrowser("30-minute rotation");
}
```

**按需重启（连接断开时）** — `apps/worker/workers/linkProcessing.ts#L65-L67`：
```typescript
if (!browser.isConnected?.()) {
  await restartBrowser("browser disconnected");
}
```

### 2.3 浏览器启动配置

**启动选项（代理/自定义路径/远程 CDP）** — `apps/worker/lib/browser.ts#L9-L32`：
- `PROXY` / `PROXY_BYPASS` / `PROXY_USERNAME` / `PROXY_PASSWORD` → 代理配置
- `PLAYWRIGHT_LAUNCH_OPTIONS_EXECUTABLE_PATH` → 自定义 Chromium
- `PLAYWRIGHT_WS_URL` → 走 CDP 连接远程浏览器而非本地 `chromium.launch`

**上下文选项（设备模拟/TLS）** — `apps/worker/lib/browser.ts#L34-L51`：
- 模拟 `devices["Desktop Chrome"]`
- `ALLOW_INSECURE_TLS=true` 或 `IGNORE_HTTPS_ERRORS=true` 时忽略证书错误

### 2.4 链接批次公平调度

**待处理链接筛选条件（决定是否会被重试）** — `apps/worker/lib/getLinkBatchFairly.ts#L35-L38`：
```typescript
const baseLinkWhere: Prisma.LinkWhereInput = {
  url: { not: null },
  lastPreserved: null,   // ← 只取 lastPreserved 仍为 null 的链接
};
```
> **关键**：`lastPreserved` 一旦被写入非 null 值（无论成功失败），该链接就**永远不会**再进入下一批处理。

**轮询取链接（用户间公平）** — `apps/worker/lib/getLinkBatchFairly.ts#L110-L145`：
- 每轮从每个用户取 `linksPerUser = maxBatchLinks / users.length` 条
- 直到取满 `maxBatchLinks` 或所有用户无剩余链接
- 最后更新 `users.lastPickedAt = now`，保证下次轮询优先照顾其他用户

### 2.5 单链接抓取前置检查

**SSRF 安全检查** — `apps/worker/lib/archiveHandler.ts#L32-L42`：
```typescript
try {
  await assertUrlIsSafeForServerSideFetch(link.url);
} catch (error) {
  if (error instanceof UnsafeUrlError) {
    skipPreservation = true;   // ← 危险 URL 直接跳过所有归档
  }
}
```

**全局超时（默认 5 分钟）** — `apps/worker/lib/archiveHandler.ts#L63-L75`：
```typescript
const abortController = new AbortController();
const timeoutPromise = new Promise((_, reject) => {
  timeoutId = setTimeout(() => {
    abortController.abort();
    reject(new Error(`Browser has been open for more than ${BROWSER_TIMEOUT} minutes.`));
  }, BROWSER_TIMEOUT * 60000);
});
// 主流程通过 Promise.race([archiveLogic, timeoutPromise]) 竞争
```

**BrowserContext 路由级 SSRF 防护** — `apps/worker/lib/protectPageRequests.ts#L15-L35`：
```typescript
await context.route("**/*", async (route: Route) => {
  const requestUrl = route.request().url();
  if (isNonNetworkUrl(requestUrl)) { await route.continue(); return; }
  try {
    await assertUrlIsSafeForServerSideFetch(requestUrl);
    await route.continue();
  } catch (error) {
    if (error instanceof UnsafeUrlError) {
      await route.abort("blockedbyclient");   // ← 拦截页面内所有对内网的请求
    }
  }
});
```

### 2.6 链接类型判定与分支

**HEAD 请求探测 Content-Type** — `apps/worker/lib/fetchHeaders.ts#L6-L21`（10 秒超时）：
```typescript
const responsePromise = safeFetch(url, { method: "HEAD" });
const timeoutPromise = new Promise((_, reject) => {
  setTimeout(() => reject(new Error("Fetch header timeout")), 10 * 1000);
});
const response = await Promise.race([responsePromise, timeoutPromise]);
```

**类型分流** — `apps/worker/lib/archiveHandler.ts#L112-L195`：
| 判定类型 | 处理函数 | 是否需要 Playwright |
|---------|---------|-------------------|
| `image` | `imageHandler()` — 直接 `safeFetch` 下载 + 生成预览 | ❌ |
| `pdf` | `pdfHandler()` — 直接 `safeFetch` 下载保存 | ❌ |
| `url` | 走完整 Playwright 流程（导航 + 多种内容提取） | ✅ |

---

## 三、渲染等待代码引用

### 3.1 页面导航等待

**`domcontentloaded` 策略** — `apps/worker/lib/archiveHandler.ts#L129`：
```typescript
await page.goto(link.url, { waitUntil: "domcontentloaded" });
```
> 只等 DOM 树构建完毕，不等所有图片/字体/样式资源加载完成，兼顾速度与完整性。

**已有 Monolith 文件直接注入** — `apps/worker/lib/archiveHandler.ts#L132-L149`：
```typescript
if (link.monolith?.endsWith(".html")) {
  const file = await readFile(link.monolith);
  await page.setContent(fileContent.toString("utf-8"), { waitUntil: "domcontentloaded" });
}
```

### 3.2 截图前自动滚动（触发懒加载）

**autoScroll 实现** — `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L96-L119`：
```typescript
const autoScroll = async (AUTOSCROLL_TIMEOUT: number) => {
  const timeoutPromise = new Promise<void>((resolve) => setTimeout(resolve, AUTOSCROLL_TIMEOUT * 1000));
  const scrollingPromise = new Promise<void>((resolve) => {
    let totalHeight = 0;
    const distance = 100;
    const scrollDown = setInterval(() => {
      window.scrollBy(0, distance);           // 每 100ms 向下滚 100px
      totalHeight += distance;
      if (totalHeight >= document.body.scrollHeight) {
        clearInterval(scrollDown);
        window.scroll(0, 0);                  // 滚完回到顶部
        resolve();
      }
    }, 100);
  });
  await Promise.race([scrollingPromise, timeoutPromise]);   // 默认 30 秒强制结束
};
```

**调用时机** — `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L12`：
```typescript
await page.evaluate(autoScroll, Number(process.env.AUTOSCROLL_TIMEOUT) || 30);
```

---

## 四、内容提取代码引用

### 4.1 Meta Description 提取

**浏览器内 JS 求值** — `apps/worker/lib/archiveHandler.ts#L151-L164`：
```typescript
const metaDescription = await page.evaluate(() => {
  const description = document.querySelector('meta[name="description"]');
  return description?.getAttribute("content") ?? undefined;
});
// 截取前 500 字符存入 link.metaDescription
```

### 4.2 预览图（Preview）生成 — OG Image 与回退逻辑

**`handleArchivePreview` 完整控制流** — `apps/worker/lib/preservationScheme/handleArchivePreview.ts#L17-L82`：

```
读取 <meta property="og:image"> → ogImageUrl
        │
        ├─ ogImageUrl === null（页面无 OG 标签）
        │     │
        │     └─ previewGenerated = false → 进入回退截图
        │
        └─ ogImageUrl 存在
              │
              ├─ 相对路径 → 拼接 origin
              │
              └─ try {
                    assertUrlIsSafeForServerSideFetch(ogImageUrl)  ← SSRF 检查
                    page.goto(ogImageUrl)                          ← 跳转到 OG 图
                    imageResponse.body()                            ← 取 buffer
                    generatePreview(...)                            ← 生成缩略图
                    page.goBack()                                   ← 返回原页面
                 } catch (error) {
                    if (error instanceof UnsafeUrlError) {
                        // 静默吞掉，什么也不做
                    } else {
                        throw error;   ← 非 SSRF 异常（网络错误等）重新抛出
                    }
                 }
                      │
                      ├─ SSRF 不安全（UnsafeUrlError）：被静默吞掉
                      │     → previewGenerated = false → 进入回退截图
                      │
                      ├─ 其他异常（page.goto 失败、buffer 读取失败等）：
                      │     → throw 向上冒泡 → handleArchivePreview 整体失败
                      │     → ❌ 不会走回退截图
                      │
                      └─ OG Image 成功：
                            ├─ generatePreview() 返回 true  → previewGenerated = true → 不回退
                            └─ generatePreview() 返回 false → previewGenerated = false → 进入回退截图
```

**回退截图逻辑** — `apps/worker/lib/preservationScheme/handleArchivePreview.ts#L59-L81`：
```typescript
if (!previewGenerated && !link.preview?.startsWith("archive")) {
  await page
    .screenshot({ type: "jpeg", quality: 20 })
    .then(async (screenshot) => {
      // 超过 PREVIEW_MAX_BUFFER（默认10MB）则跳过
      // 保存到 archives/preview/{collectionId}/{linkId}.jpeg
      // 更新 link.preview 字段
    });
}
```
> 注意：回退截图的 `page.screenshot()` 本身挂在 `.then()` 上，没有对应 `.catch()`；如果截图抛异常，`handleArchivePreview` 会整体失败并向上抛出。

**回退触发条件汇总**：

| 场景 | `previewGenerated` | 是否走回退截图 |
|------|-------------------|--------------|
| 页面无 `og:image` 标签 | `false` | ✅ |
| `og:image` URL 是 SSRF 不安全（`UnsafeUrlError`） | `false` | ✅ |
| `generatePreview()` 生成缩略图返回 `false` | `false` | ✅ |
| `page.goto(ogImageUrl)` 抛网络错误 / `body()` 抛异常 | —（函数整体 throw）| ❌ |
| OG Image 完整成功 | `true` | ❌ |
| `link.preview` 已经是 `archive*` 开头（已有归档预览） | 任意 | ❌ |

### 4.3 Readability 正文提取

**完整流程** — `apps/worker/lib/preservationScheme/handleReadability.ts#L8-L61`：
```
原始 HTML
  ↓ DOMPurify.sanitize()  XSS 净化
  ↓ JSDOM 构建 DOM
  ↓ @mozilla/readability 解析文章
  ↓ 去空格 / 去换行 / TEXT_CONTENT_LIMIT 截断
  ↓
  ├─ JSON 存 archives/{collectionId}/{linkId}_readability.json
  └─ 纯文本存 link.textContent（供搜索索引）
```
大小限制 `READABILITY_MAX_BUFFER`（默认 100MB）。

### 4.4 全页截图

**截图调用** — `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L21-L51`：
```typescript
page.screenshot({ fullPage: true, type: "jpeg" })
// 限制 SCREENSHOT_MAX_BUFFER（默认 100MB）
// 保存 archives/{collectionId}/{linkId}.jpeg
```

### 4.5 PDF 导出

**PDF 调用** — `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L58-L91`：
```typescript
page.pdf({
  width: "1366px",
  height: "1931px",
  printBackground: true,
  margin: { top: "15px", bottom: "15px" },  // 可通过 PDF_MARGIN_* 配置
})
// 限制 PDF_MAX_BUFFER（默认 100MB）
// 保存 archives/{collectionId}/{linkId}.pdf
```

> 截图和 PDF 通过 `Promise.allSettled(processingPromises)` 并行互不影响 — `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L92`

### 4.6 Monolith 单文件归档

**外部 monolith CLI 调用** — `apps/worker/lib/preservationScheme/handleMonolith.ts#L13-L73`：
```typescript
const child = spawn("monolith", [
  "-",            // 从 stdin 读 HTML
  "-I",           // 移除图片
  "-b", link.url, // base URL
  "-j", "-F", "-q", ...  // 去 JS / 去框架 / 静默
  "-o", "-"       // 输出到 stdout
], { signal: abortController.signal, killSignal: "SIGKILL" });

child.stdin.write(htmlFromPage);   // 注入 Playwright 获取到的 HTML
child.stdin.end();
```
- 大小限制 `MONOLITH_MAX_BUFFER`（默认 100MB）
- 支持 `AbortSignal` 超时中断
- **独立 `.catch()` 不阻塞其他格式** — `apps/worker/lib/archiveHandler.ts#L189-L193`

### 4.7 Wayback Machine 提交

**Fire-and-Forget** — `apps/worker/lib/preservationScheme/sendToWayback.ts#L13-L20`：
```typescript
axios.get(`https://web.archive.org/save/${url}`, { headers })
  .then(() => console.log(`Sent ${url} to Wayback Machine`))
  .catch(() => {});   // 完全静默失败
```
在 archiveHandler 中**不 await**，异步发出即返回 — `apps/worker/lib/archiveHandler.ts#L118-L120`

---

## 五、失败处理与重试机制代码引用

### 5.0 重提前提：待处理链接的筛选条件

来自 `apps/worker/lib/getLinkBatchFairly.ts#L35-L38`：
```typescript
lastPreserved: null   // ← 只有此字段仍为 null 的链接才会被下一批选中
```
因此，**`lastPreserved` 是否被写入是判断链接能否被重试的唯一依据**。下面按 `archiveHandler` 的执行阶段分三类讨论。

---

### 5.1 阶段 A：主流程前失败（try 之前的早退出分支）

**代码位置** — `apps/worker/lib/archiveHandler.ts#L44-L61`：
```typescript
if (
  skipPreservation ||
  (!link.url?.startsWith("http://") && !link.url?.startsWith("https://"))
) {
  await prisma.link.update({
    where: { id: link.id },
    data: {
      lastPreserved: new Date().toISOString(),   // ← 写 lastPreserved
      readable: "unavailable",
      image: "unavailable",
      monolith: "unavailable",
      pdf: "unavailable",
      preview: "unavailable",
      indexVersion: null,
    },
  });
  return;   // ← 直接返回，不进入 try / finally
}
```

**触发条件**：
- `skipPreservation === true`：由 SSRF 检查抛出 `UnsafeUrlError` 或 `DISABLE_PRESERVATION=true` 导致
- URL 不是 `http://` 或 `https://` 开头

**对重试的影响**：
| 项目 | 结果 |
|------|------|
| `lastPreserved` | ✅ **被写入为当前时间** |
| 各格式字段 | ✅ 全部硬编码为 `"unavailable"` |
| 是否进入 `finally` | ❌ 不会，代码在 `return` 处已退出 |
| 是否关闭 BrowserContext | N/A（Context 尚未创建） |
| **下一批是否会被重试** | ❌ **不会**（`lastPreserved` 非 null） |

---

### 5.2 阶段 B：try 之前但 BrowserContext 创建期间失败

`archiveHandler` 中，**`try {` 关键字出现在第 109 行**，但以下代码在 try 之前执行（`apps/worker/lib/archiveHandler.ts#L63-L108`）：

```
L63  const abortController = new AbortController();
L66  const timeoutPromise = new Promise(...);
L77  const contextOptions = getDefaultContextOptions();
L78  const context = await browser.newContext(contextOptions);   ← 可能抛异常
L79  await protectPageRequests(context);                         ← 可能抛异常
L80  const page = await context.newPage();                       ← 可能抛异常
L82  createFolder(...);
L83  createFolder(...);
L85  const archivalTags = link.tags.filter(isArchivalTag);
L86  const archivalSettings = ...    // 读取用户归档偏好
```

**如果 L78-L80（浏览器上下文/页面创建）抛异常**：
- 异常直接向上抛出，**不会进入 `try`，也不会进入 `finally`**
- `lastPreserved` **不会被写入**（仍为 `null`）
- BrowserContext 可能泄漏（没有 `context.close()`）

**对重试的影响**：
| 项目 | 结果 |
|------|------|
| `lastPreserved` | ❌ **未被写入（仍为 null）** |
| 各格式字段 | 保持不变 |
| 是否进入 `finally` | ❌ 不会 |
| 是否关闭 BrowserContext | ❌ 不会（可能泄漏） |
| **下一批是否会被重试** | ✅ **会**（`lastPreserved` 仍为 null） |

这是整个流程中**唯一会导致链接自动重试**的失败场景。

---

### 5.3 阶段 C：主流程内失败（try 块内部）

**try/catch/finally 结构** — `apps/worker/lib/archiveHandler.ts#L109-L230`：
```typescript
try {
  await Promise.race([
    (async () => {
      // determineLinkType → imageHandler / pdfHandler / page.goto + 各内容提取
    })(),
    timeoutPromise,
  ]);
} catch (err) {
  console.log("Failed Link:", link.url);
  console.log("Reason:", err);
  throw err;   // ← 捕获后重新抛出
} finally {
  if (timeoutId !== undefined) clearTimeout(timeoutId);

  const finalLink = await prisma.link.findUnique({ where: { id: link.id } });
  if (finalLink) {
    await prisma.link.update({
      where: { id: link.id },
      data: {
        lastPreserved: new Date().toISOString(),   // ← 必然写 lastPreserved
        readable: !finalLink.readable  ? "unavailable" : undefined,
        image:    !finalLink.image     ? "unavailable" : undefined,
        monolith: !finalLink.monolith  ? "unavailable" : undefined,
        pdf:      !finalLink.pdf       ? "unavailable" : undefined,
        preview:  !finalLink.preview   ? "unavailable" : undefined,
        indexVersion: null,
      },
    });
  } else {
    await removeFiles(link.id, link.collectionId);   // 链接已被删除则清理文件
  }

  await context?.close().catch(() => {});   // ← 必然关闭 Context
}
```

**触发条件（try 内任意异常）**：
- `determineLinkType` 中 HEAD 请求异常（fetchHeaders 本身已 try/catch，但极端情况仍可能抛）
- `imageHandler` / `pdfHandler` 抛异常（下载失败、Buffer 超限之外的错误）
- `page.goto()` 失败 / 超时
- `metaDescription` 的 `page.evaluate()` 抛异常
- `handleArchivePreview()` 抛异常（如 OG Image 非 SSRF 错误、回退截图失败）
- `handleReadability()` 抛异常（DOMPurify / Readability / 文件写入失败等）
- `handleScreenshotAndPdf()` 抛异常（autoScroll 抛异常、截图/PDF 异常 — 注意其内部 `Promise.allSettled` 只能隔离截图和 PDF 之间，不能隔离 autoScroll 和上层）
- 全局 `BROWSER_TIMEOUT` 超时触发（`timeoutPromise` reject）
- `handleMonolith` 之外的任何未被局部 catch 的错误

**对重试的影响**：
| 项目 | 结果 |
|------|------|
| `lastPreserved` | ✅ **被写入为当前时间（finally 必然执行）** |
| 各格式字段 | ✅ 仍为空的格式被标记为 `"unavailable"`，已成功生成的保留 |
| 是否关闭 BrowserContext | ✅ `context?.close()` 必然调用 |
| **下一批是否会被重试** | ❌ **不会**（`lastPreserved` 非 null） |

---

### 5.4 三阶段失败行为对比

| 失败阶段 | 代码位置 | `lastPreserved` 是否写入 | 是否会被下一批重试 | 是否关闭 BrowserContext |
|---------|---------|------------------------|-------------------|----------------------|
| **A. 主流程前早退出**（SSRF 不安全 / 非 http(s) / DISABLE_PRESERVATION） | `apps/worker/lib/archiveHandler.ts#L44-L61` | ✅ 写入 | ❌ 不会 | N/A（还未创建） |
| **B. BrowserContext 创建失败**（`browser.newContext` / `newPage` 抛异常） | `apps/worker/lib/archiveHandler.ts#L77-L80`（try 之外） | ❌ 不写入 | ✅ **会重试** | ❌ 可能泄漏 |
| **C. 主流程内任意异常**（try 块内所有代码） | `apps/worker/lib/archiveHandler.ts#L109-L198` | ✅ 写入（finally） | ❌ 不会 | ✅ `finally` 中关闭 |

> **核心结论**：只有阶段 B（BrowserContext 创建期间失败）会导致链接在后续批次中被自动重试；其余失败均会写入 `lastPreserved` 从而永久标记为"已处理"，失败格式字段值为 `"unavailable"`。

---

### 5.5 批次级：单条失败不影响同批其他链接

**`Promise.allSettled` 并发处理** — `apps/worker/workers/linkProcessing.ts#L71-L72`：
```typescript
const processingPromises = links.map((e) => archiveLink(e));
await Promise.allSettled(processingPromises);
```

**单条异常捕获** — `apps/worker/workers/linkProcessing.ts#L45-L69`：
```typescript
const archiveLink = async (link) => {
  try {
    await archiveHandler(link, browser);
  } catch (error) {
    console.error(`Error processing link ${link.url}:`, error);
    if (!browser.isConnected?.()) {
      await restartBrowser("browser disconnected");   // 浏览器挂了就重启
    }
  }
};
```

### 5.6 各模块局部容错汇总

| 模块 | 文件与行号 | 容错行为 | 是否会冒泡到 archiveHandler 的 catch |
|------|-----------|---------|-------------------------------------|
| `fetchHeaders` | `apps/worker/lib/fetchHeaders.ts#L18-L21` | 超时/失败 → 返回 `null`，类型默认 `"url"` | ❌ 内部吞掉 |
| `handleArchivePreview` OG Image SSRF 错误 | `apps/worker/lib/preservationScheme/handleArchivePreview.ts#L52-L56` | `UnsafeUrlError` 静默 → 回退截图 | ❌ 吞掉并回退 |
| `handleArchivePreview` OG Image 非 SSRF 错误 | `apps/worker/lib/preservationScheme/handleArchivePreview.ts#L53-L54` | 其他异常（网络错误等）重新抛出 | ✅ 冒泡 |
| `handleArchivePreview` 回退截图失败 | `apps/worker/lib/preservationScheme/handleArchivePreview.ts#L60-L80` | 无 `.catch()`，Promise reject 冒泡 | ✅ 冒泡 |
| `handleScreenshotAndPdf` 截图 vs PDF | `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L92` | `Promise.allSettled` 让截图和 PDF 互不影响 | 截图/PDF 各自失败不影响对方，但 autoScroll 等前置失败会冒泡 |
| `handleMonolith` | `apps/worker/lib/archiveHandler.ts#L189-L193` | 独立 `.catch(err => console.error(err))` | ❌ 吞掉，只打日志 |
| `sendToWayback` | `apps/worker/lib/preservationScheme/sendToWayback.ts#L20` | `.catch(() => {})` 完全静默 | ❌ 吞掉，且不 await |

### 5.7 进程级自动恢复

| 层级 | 机制 | 文件与行号 |
|------|------|-----------|
| 整个 Worker 进程 | `spawn` 子进程退出 → 5 秒后重启 | `apps/worker/index.ts#L6-L10` |
| 浏览器实例 | 30 分钟定期重启 + 断连按需重启 | `apps/worker/workers/linkProcessing.ts#L18-L33` |

---

## 六、配置项汇总（全部为环境变量）

| 变量名 | 默认值 | 生效位置 |
|--------|--------|---------|
| `BROWSER_TIMEOUT` | `5` 分钟 | `apps/worker/lib/archiveHandler.ts#L23` |
| `ARCHIVE_SCRIPT_INTERVAL` | `10` 秒 | `apps/worker/worker.ts#L8` |
| `ARCHIVE_TAKE_COUNT` | `5` | `apps/worker/workers/linkProcessing.ts#L8` |
| `AUTOSCROLL_TIMEOUT` | `30` 秒 | `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L12` |
| `PREVIEW_MAX_BUFFER` | `10` MB | `apps/worker/lib/preservationScheme/handleArchivePreview.ts#L65` |
| `SCREENSHOT_MAX_BUFFER` | `100` MB | `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L30` |
| `PDF_MAX_BUFFER` | `100` MB | `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L69` |
| `READABILITY_MAX_BUFFER` | `100` MB | `apps/worker/lib/preservationScheme/handleReadability.ts#L41` |
| `MONOLITH_MAX_BUFFER` | `100` MB | `apps/worker/lib/preservationScheme/handleMonolith.ts#L52` |
| `TEXT_CONTENT_LIMIT` | 无限制 | `apps/worker/lib/preservationScheme/handleReadability.ts#L13` |
| `PROXY` / `PROXY_BYPASS` / `PROXY_USERNAME` / `PROXY_PASSWORD` | - | `apps/worker/lib/browser.ts#L14-L21` |
| `PLAYWRIGHT_WS_URL` | - | `apps/worker/lib/browser.ts#L56-L58` |
| `PLAYWRIGHT_LAUNCH_OPTIONS_EXECUTABLE_PATH` | - | `apps/worker/lib/browser.ts#L23-L29` |
| `ALLOW_INSECURE_TLS` / `IGNORE_HTTPS_ERRORS` | `false` | `apps/worker/lib/browser.ts#L37-L39` |
| `MONOLITH_CUSTOM_OPTIONS` | `-j -F -q` | `apps/worker/lib/preservationScheme/handleMonolith.ts#L19-L21` |
| `PDF_MARGIN_TOP` / `PDF_MARGIN_BOTTOM` | `15px` | `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L53-L56` |

---

## 七、完整调用链速查（含失败分支）

```
apps/worker/index.ts (进程守护: 子进程挂了 5s 后重启)
 └─ spawn tsx worker.ts
     └─ apps/worker/worker.ts#init()
         └─ apps/worker/workers/linkProcessing.ts#linkProcessing() [无限循环]
             ├─ 每 30 分钟重启浏览器
             ├─ apps/worker/lib/getLinkBatchFairly.ts#getLinkBatchFairly()  ── 取 lastPreserved=null 的链接
             └─ Promise.allSettled 并发:
                 └─ apps/worker/lib/archiveHandler.ts#archiveHandler(link, browser)
                     │
                     ├─ [阶段 A] SSRF 不安全 / 非 http(s) / DISABLE_PRESERVATION
                     │     └─ 直接写 lastPreserved + 所有格式 "unavailable" → return (不重试)
                     │
                     ├─ AbortController + 5 分钟 timeoutPromise
                     │
                     ├─ [阶段 B] browser.newContext() / context.newPage()
                     │     └─ 失败: 异常向上抛出，lastPreserved 不写 → 下批会重试 ✅
                     │
                     ├─ protectPageRequests (路由级 SSRF)
                     ├─ createFolder
                     ├─ 读取 archivalSettings
                     │
                     └─ try {
                          Promise.race([
                            determineLinkType → 分流:
                              ├─ image ──► imageHandler.ts
                              ├─ pdf   ──► pdfHandler.ts
                              └─ url   ──► Playwright 流程:
                                    ├─ page.goto(url, domcontentloaded)
                                    ├─ metaDescription 提取
                                    ├─ handleArchivePreview (OG Image → 仅 SSRF 错误会回退截图)
                                    ├─ handleReadability
                                    ├─ handleScreenshotAndPdf (autoScroll → 全页截图 + PDF)
                                    ├─ handleMonolith (独立 .catch，不冒泡)
                                    └─ sendToWayback (fire-and-forget)
                          , timeoutPromise ])
                        } catch (err) {
                          日志 + re-throw
                        } finally {                              ← [阶段 C] 必然执行
                          ├─ 清 timeout
                          ├─ 写 lastPreserved = now
                          ├─ 未生成的格式 → "unavailable"         ← 不再重试 ❌
                          └─ context.close()
                        }
```
