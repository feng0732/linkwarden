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

**待处理链接筛选条件** — `apps/worker/lib/getLinkBatchFairly.ts#L35-L38`：
```typescript
const baseLinkWhere: Prisma.LinkWhereInput = {
  url: { not: null },
  lastPreserved: null,   // ← 只取从未处理过的链接
};
```

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

### 4.2 预览图（Preview）生成

**策略一：优先 OG Image** — `apps/worker/lib/preservationScheme/handleArchivePreview.ts#L21-L57`：
1. `page.evaluate` 读取 `<meta property="og:image">`
2. 相对路径拼 `document.location.origin`
3. SSRF 检查后 `page.goto(ogImageUrl)` 跳转抓取
4. `generatePreview(buffer, collectionId, linkId)` 生成缩略图
5. `page.goBack()` 回到原页面

**策略二：回退低质量截图** — `apps/worker/lib/preservationScheme/handleArchivePreview.ts#L59-L81`：
```typescript
await page.screenshot({ type: "jpeg", quality: 20 })
// 限制 PREVIEW_MAX_BUFFER（默认 10MB）
// 保存到 archives/preview/{collectionId}/{linkId}.jpeg
```

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

### 5.1 单链接：失败标记为不可用（不重试）

**`finally` 块标记逻辑** — `apps/worker/lib/archiveHandler.ts#L203-L230`：
```typescript
const finalLink = await prisma.link.findUnique({ where: { id: link.id } });
if (finalLink) {
  await prisma.link.update({
    where: { id: link.id },
    data: {
      lastPreserved: new Date().toISOString(),           // ← 标记为"已处理过"
      readable: !finalLink.readable  ? "unavailable" : undefined,
      image:    !finalLink.image     ? "unavailable" : undefined,
      monolith: !finalLink.monolith  ? "unavailable" : undefined,
      pdf:      !finalLink.pdf       ? "unavailable" : undefined,
      preview:  !finalLink.preview   ? "unavailable" : undefined,
      indexVersion: null,
    },
  });
}
```

**关键结论**：
- 无论成功/失败/超时，`lastPreserved` 一定会被写入
- 未生成的格式写 `"unavailable"`（而不是 `null`）
- 批次查询条件是 `lastPreserved: null`，因此该链接**永远不会再进入处理队列**
- → **单链接只有一次处理机会，没有自动重试**

### 5.2 批次级：单条失败不影响同批其他链接

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

### 5.3 各模块局部容错汇总

| 模块 | 文件与行号 | 容错行为 |
|------|-----------|---------|
| `fetchHeaders` | `apps/worker/lib/fetchHeaders.ts#L18-L21` | 超时/失败 → 返回 `null`，类型默认 `"url"` |
| `handleArchivePreview` | `apps/worker/lib/preservationScheme/handleArchivePreview.ts#L52-L57` | OG Image 失败 → 回退到 `page.screenshot` |
| `handleScreenshotAndPdf` | `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L92` | `Promise.allSettled` 让截图和 PDF 互不影响 |
| `handleMonolith` | `apps/worker/lib/archiveHandler.ts#L189-L193` | 独立 `.catch()`，失败只打日志不抛出 |
| `sendToWayback` | `apps/worker/lib/preservationScheme/sendToWayback.ts#L20` | `.catch(() => {})` 完全静默 |

### 5.4 进程级自动恢复

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

## 七、完整调用链速查

```
apps/worker/index.ts (进程守护)
 └─ spawn tsx worker.ts
     └─ apps/worker/worker.ts#init()
         └─ apps/worker/workers/linkProcessing.ts#linkProcessing() [无限循环]
             ├─ 每 30 分钟重启浏览器
             ├─ apps/worker/lib/getLinkBatchFairly.ts#getLinkBatchFairly()  ── 取 N 条链接
             └─ Promise.allSettled 并发:
                 └─ apps/worker/lib/archiveHandler.ts#archiveHandler(link, browser)
                     ├─ SSRF 检查
                     ├─ 5 分钟全局超时 AbortController
                     ├─ BrowserContext + protectPageRequests (路由级 SSRF)
                     ├─ apps/worker/lib/fetchHeaders.ts  HEAD 判断类型
                     │   ├─ image ──► imageHandler.ts    (safeFetch 直下)
                     │   ├─ pdf   ──► pdfHandler.ts      (safeFetch 直下)
                     │   └─ url   ──► Playwright 流程:
                     │       ├─ page.goto(url, domcontentloaded)
                     │       ├─ metaDescription 提取
                     │       ├─ handleArchivePreview.ts   (OG Image → 回退截图)
                     │       ├─ handleReadability.ts      (DOMPurify → Readability → JSON + textContent)
                     │       ├─ handleScreenshotAndPdf.ts (autoScroll → 全页截图 + PDF)
                     │       ├─ handleMonolith.ts         (spawn monolith CLI)
                     │       └─ sendToWayback.ts          (fire-and-forget)
                     └─ finally:
                         ├─ lastPreserved = now
                         └─ 未生成格式 → "unavailable" (不再重试)
```
