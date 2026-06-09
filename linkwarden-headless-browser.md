# Linkwarden Headless Browser 抓取与渲染流程分析

## 一、整体架构概览

Linkwarden 的 Headless Browser 功能位于 `apps/worker/` 目录下，基于 **Playwright** 实现。系统由以下核心模块组成：

| 模块 | 文件 | 职责 |
|------|------|------|
| Worker 入口 | [worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/worker.ts) | 初始化并启动多个后台 Worker |
| 链接处理循环 | [linkProcessing.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/workers/linkProcessing.ts) | 批量获取待处理链接，调度浏览器抓取 |
| 浏览器管理 | [browser.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/browser.ts) | Playwright 浏览器启动与配置 |
| 归档核心处理器 | [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/archiveHandler.ts) | 单个链接的完整归档流程编排 |
| 请求安全防护 | [protectPageRequests.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/protectPageRequests.ts) | SSRF 防护，拦截页面内不安全请求 |
| 内容类型探测 | [fetchHeaders.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/fetchHeaders.ts) | 发送 HEAD 请求判断链接类型 |

---

## 二、Worker 启动与调度流程

### 2.1 进程级守护

[index.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/index.ts) 是最外层的进程守护：

- 使用 `child_process.spawn` 启动 `tsx worker.ts` 子进程
- 监听子进程 `exit` 事件，若异常退出则 **5 秒后自动重启**
- 监听 `SIGINT` 信号实现优雅退出

### 2.2 Worker 内部任务调度

[worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/worker.ts) 初始化时并行启动多个 Worker：

```
migrationWorker() → 数据迁移
    ↓
startRSSPolling()        → RSS 轮询
linkProcessing()         → 链接归档（核心）
autoTagPreservedLinks()  → AI 自动打标签
startIndexing()          → 全文索引
trialEndEmailWorker()    → 试用到期邮件
```

其中 `linkProcessing` 是 Headless Browser 抓取的主入口。

---

## 三、页面抓取流程

### 3.1 浏览器生命周期管理

[linkProcessing.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/workers/linkProcessing.ts) 中的浏览器管理策略：

1. **单例浏览器实例**：整个 Worker 循环共享一个 Browser 实例
2. **30 分钟自动重启**：每 `BROWSER_MAX_AGE_MS = 30 * 60 * 1000` 重启一次浏览器，防止内存泄漏和进程僵死
3. **按需重启**：若检测到 `browser.isConnected()` 返回 false，立即重启浏览器

```typescript
const restartBrowser = async (reason: string) => {
  try {
    if (browser && browser.isConnected()) {
      await browser.close();
    }
  } catch {}
  browser = await launchBrowser();
  browserStartTs = Date.now();
};
```

### 3.2 浏览器启动配置

[browser.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/browser.ts) 提供三级配置：

#### 启动选项 `getBrowserOptions()`
- 支持 `PROXY` 环境变量配置 HTTP/HTTPS 代理（含用户名密码认证）
- 支持 `PLAYWRIGHT_LAUNCH_OPTIONS_EXECUTABLE_PATH` 指定自定义 Chromium 路径
- 支持 `PLAYWRIGHT_WS_URL` 连接远程浏览器（CDP 协议）

#### 上下文选项 `getDefaultContextOptions()`
- 模拟设备：`devices["Desktop Chrome"]`
- HTTPS 忽略：当 `ALLOW_INSECURE_TLS=true` 或 `IGNORE_HTTPS_ERRORS=true` 时忽略证书错误

### 3.3 链接批次获取（公平调度）

[getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/getLinkBatchFairly.ts) 实现了**用户间公平调度**：

**链接筛选条件**（mode = "links"）：
- `url` 不为 null
- `lastPreserved` 为 null（尚未处理过）

**用户筛选条件**：
- 有订阅（或试用期内，若无需信用卡）
- 邮箱已验证（若配置了邮件服务）

**调度算法**：
1. 按 `lastPickedAt` 升序选出用户（优先照顾长时间未被调度的用户）
2. 统计每个用户的待处理链接数
3. 以轮询方式从每个用户处取 `linksPerUser` 条链接，直到填满 `maxBatchLinks`
4. 更新被选中用户的 `lastPickedAt` 时间戳

### 3.4 单链接处理流程

[archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/archiveHandler.ts) 是单个链接的完整处理流程：

#### 前置检查
1. **SSRF 安全检查**：调用 `assertUrlIsSafeForServerSideFetch(link.url)` 验证 URL 安全性，若为内网/危险 URL 则跳过整个归档流程
2. **协议检查**：仅处理 `http://` 和 `https://` 开头的 URL
3. **全局超时**：设置 `BROWSER_TIMEOUT`（默认 5 分钟）的 AbortController，超时后终止整个流程

#### BrowserContext 与安全防护
```typescript
const context = await browser.newContext(contextOptions);
await protectPageRequests(context);  // 注册路由拦截器
const page = await context.newPage();
```

[protectPageRequests.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/protectPageRequests.ts) 拦截页面内所有请求：
- 放行 `about:`、`blob:`、`data:` 等非网络 URL
- 对其他 URL 执行 SSRF 安全检查，不安全则以 `blockedbyclient` 原因中止请求

#### 链接类型判定

[fetchHeaders.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/fetchHeaders.ts) + `determineLinkType()`：
1. 发送 `HEAD` 请求（10 秒超时）
2. 根据 `content-type` 响应头判定类型：
   - `application/pdf` → 类型 `"pdf"`
   - `image/*` → 类型 `"image"`（并区分 jpeg/png）
   - 其他 → 类型 `"url"`

#### 分支处理

**类型 = image**：走 [imageHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/preservationScheme/imageHandler.ts)
- 直接 `safeFetch` 下载原图
- 生成预览缩略图
- 保存原始文件并写入数据库

**类型 = pdf**：走 [pdfHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/preservationScheme/pdfHandler.ts)
- 直接 `safeFetch` 下载 PDF
- 保存文件并写入数据库

**类型 = url**：走完整的 Headless Browser 流程（见下节）

---

## 四、渲染等待与内容提取

### 4.1 页面导航等待

```typescript
await page.goto(link.url, { waitUntil: "domcontentloaded" });
```

等待策略为 `domcontentloaded`——DOM 树构建完成即继续，不必等所有资源加载完毕。

若链接已有预先生成的 Monolith HTML 文件，则直接用 `page.setContent()` 加载本地 HTML 内容，无需再次请求网络。

### 4.2 Meta Description 提取

```typescript
const metaDescription = await page.evaluate(() => {
  const description = document.querySelector('meta[name="description"]');
  return description?.getAttribute("content") ?? undefined;
});
```

在浏览器上下文中执行 JS，截取前 500 字符存入 `link.metaDescription`。

### 4.3 预览图生成（Preview）

[handleArchivePreview.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts)：

**策略一：优先使用 OG Image**
1. 读取页面 `<meta property="og:image">` 内容
2. 若为相对路径则拼接 `document.location.origin`
3. 对 OG Image URL 做 SSRF 检查后跳转过去抓取图片
4. 调用 `generatePreview()` 生成缩略图，然后 `page.goBack()` 返回原页面

**策略二：回退到页面截图**
- 使用 `page.screenshot({ type: "jpeg", quality: 20 })` 低质量截图
- 限制 Buffer 不超过 `PREVIEW_MAX_BUFFER`（默认 10MB）
- 保存到 `archives/preview/{collectionId}/{linkId}.jpeg`

### 4.4 Readability 正文提取

[handleReadability.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/preservationScheme/handleReadability.ts)：

1. **XSS 净化**：使用 `DOMPurify` 对页面 HTML 进行净化
2. **正文提取**：使用 Mozilla 的 `@mozilla/readability` + `jsdom` 解析出文章正文
3. **文本清洗**：
   - 去重连续空格
   - 去除换行符
   - 按 `TEXT_CONTENT_LIMIT` 截断字符数
4. **持久化**：
   - JSON 序列化 Readability 结果保存到 `archives/{collectionId}/{linkId}_readability.json`
   - 纯文本内容存入 `link.textContent`（用于搜索索引）
   - 文件大小限制 `READABILITY_MAX_BUFFER`（默认 100MB）

### 4.5 截图与 PDF 生成

[handleScreenshotAndPdf.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts)：

#### 自动滚动（Auto-Scroll）
为了触发懒加载图片/内容，截图前先执行自动滚动：

```typescript
const autoScroll = async (AUTOSCROLL_TIMEOUT: number) => {
  // 每 100ms 向下滚动 100px
  // 直到滚动距离 >= document.body.scrollHeight
  // 或达到 AUTOSCROLL_TIMEOUT（默认 30 秒）
};
```

滚动完成后滚回顶部 `window.scroll(0, 0)`。

#### 全页截图
- `page.screenshot({ fullPage: true, type: "jpeg" })`
- 大小限制 `SCREENSHOT_MAX_BUFFER`（默认 100MB）
- 保存到 `archives/{collectionId}/{linkId}.jpeg`

#### PDF 导出
- `page.pdf({ width: "1366px", height: "1931px", printBackground: true })`
- 上下边距默认 `15px`，可通过 `PDF_MARGIN_TOP` / `PDF_MARGIN_BOTTOM` 配置
- 大小限制 `PDF_MAX_BUFFER`（默认 100MB）
- 保存到 `archives/{collectionId}/{linkId}.pdf`

截图和 PDF 使用 `Promise.allSettled` 并行执行，互不影响。

### 4.6 Monolith 单文件归档

[handleMonolith.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/preservationScheme/handleMonolith.ts)：

使用外部命令行工具 `monolith` 将页面保存为自包含的单个 HTML 文件：

1. 通过 `child_process.spawn` 调用 `monolith`
2. 从 stdin 注入当前页面的 HTML 内容
3. 参数：`-I`（移除图片）、`-b`（指定 base URL）、`-j`（移除 JS）、`-F`（去除框架）、`-q`（静默模式）
4. 支持 `MONOLITH_CUSTOM_OPTIONS` 环境变量自定义参数
5. 输出限制 `MONOLITH_MAX_BUFFER`（默认 100MB）
6. 保存到 `archives/{collectionId}/{linkId}.html`
7. 支持通过 `AbortSignal` 超时中断

### 4.7 Wayback Machine 提交

[sendToWayback.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/preservationScheme/sendToWayback.ts)：
- 异步发送 GET 请求到 `https://web.archive.org/save/{url}`
- **fire-and-forget**：不等待结果，失败静默忽略

---

## 五、失败处理与重试机制

### 5.1 单链接级别的失败处理

在 [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/lib/archiveHandler.ts) 的 `finally` 块中：

```typescript
await prisma.link.update({
  where: { id: link.id },
  data: {
    lastPreserved: new Date().toISOString(),
    readable: !finalLink.readable ? "unavailable" : undefined,
    image: !finalLink.image ? "unavailable" : undefined,
    monolith: !finalLink.monolith ? "unavailable" : undefined,
    pdf: !finalLink.pdf ? "unavailable" : undefined,
    preview: !finalLink.preview ? "unavailable" : undefined,
    indexVersion: null,
  },
});
```

**关键逻辑**：
- 无论成功或失败，都会设置 `lastPreserved` 为当前时间
- 对于未能成功生成的格式，字段值被标记为 `"unavailable"` 而不是 `null`
- 标记为 `"unavailable"` 的链接**不会在后续批次中被重试**（因为查询条件是 `lastPreserved: null`）

这意味着：**单个链接只有一次处理机会，失败即标记为不可用，不会自动重试**。

### 5.2 批次级别的容错

在 [linkProcessing.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/workers/linkProcessing.ts) 中：

```typescript
const processingPromises = links.map((e) => archiveLink(e));
await Promise.allSettled(processingPromises);
```

- 使用 `Promise.allSettled`：单个链接失败不影响同批次其他链接
- 每个 `archiveLink` 内部的 try-catch 捕获异常并打印日志
- 若检测到浏览器断开连接，触发 `restartBrowser("browser disconnected")`

### 5.3 部分步骤的局部容错

| 模块 | 容错策略 |
|------|---------|
| `fetchHeaders` | 10 秒超时或请求失败返回 `null`，链接类型默认为 `"url"` |
| `handleArchivePreview` | OG Image 下载失败回退到页面截图 |
| `handleScreenshotAndPdf` | 截图和 PDF 用 `Promise.allSettled` 独立执行 |
| `handleMonolith` | 单独的 `.catch(err => console.error(err))`，失败不影响其他格式 |
| `sendToWayback` | 完全静默失败 |

### 5.4 进程级别的自动恢复

最外层 [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/125-linkwarden/apps/worker/index.ts)：
- Worker 子进程异常退出 → 5 秒后重启
- 浏览器每 30 分钟定期重启，防止资源泄漏

---

## 六、配置项汇总

| 环境变量 | 默认值 | 说明 |
|---------|--------|------|
| `BROWSER_TIMEOUT` | `5` | 单链接浏览器处理超时（分钟） |
| `ARCHIVE_SCRIPT_INTERVAL` | `10` | Worker 轮询间隔（秒） |
| `ARCHIVE_TAKE_COUNT` | `5` | 每批次处理链接数 |
| `AUTOSCROLL_TIMEOUT` | `30` | 自动滚动超时（秒） |
| `PREVIEW_MAX_BUFFER` | `10` | 预览图大小上限（MB） |
| `SCREENSHOT_MAX_BUFFER` | `100` | 截图大小上限（MB） |
| `PDF_MAX_BUFFER` | `100` | PDF 大小上限（MB） |
| `READABILITY_MAX_BUFFER` | `100` | Readability JSON 大小上限（MB） |
| `MONOLITH_MAX_BUFFER` | `100` | Monolith HTML 大小上限（MB） |
| `TEXT_CONTENT_LIMIT` | 无限制 | 正文纯文本字符数上限 |
| `PROXY` | - | HTTP/HTTPS 代理地址 |
| `PROXY_BYPASS` | - | 代理绕过规则 |
| `PROXY_USERNAME` | - | 代理用户名 |
| `PROXY_PASSWORD` | - | 代理密码 |
| `PLAYWRIGHT_WS_URL` | - | 远程浏览器 CDP 地址 |
| `PLAYWRIGHT_LAUNCH_OPTIONS_EXECUTABLE_PATH` | - | 自定义 Chromium 路径 |
| `ALLOW_INSECURE_TLS` / `IGNORE_HTTPS_ERRORS` | `false` | 是否忽略 HTTPS 证书错误 |
| `MONOLITH_CUSTOM_OPTIONS` | `-j -F -q` | Monolith 自定义参数 |
| `PDF_MARGIN_TOP` / `PDF_MARGIN_BOTTOM` | `15px` | PDF 页边距 |

---

## 七、流程时序图（简化）

```
linkProcessing (无限循环)
    │
    ├─→ 检查浏览器年龄，必要时重启
    │
    ├─→ getLinkBatchFairly() ── 公平调度获取 N 条链接
    │
    └─→ 并发处理每条链接 (Promise.allSettled)
            │
            └─→ archiveHandler(link, browser)
                    │
                    ├─→ SSRF 安全检查
                    ├─→ 创建 BrowserContext (带 SSRF 路由拦截)
                    ├─→ 5 分钟全局超时 AbortController
                    │
                    ├─→ fetchHeaders() ── HEAD 请求判断类型
                    │       │
                    │       ├─ image → imageHandler() ── 直接下载
                    │       ├─ pdf   → pdfHandler()   ── 直接下载
                    │       └─ url   → 继续 Playwright 流程
                    │
                    ├─→ page.goto(url, waitUntil: "domcontentloaded")
                    ├─→ 提取 metaDescription
                    │
                    ├─→ handleArchivePreview()  ── OG Image / 截图
                    ├─→ handleReadability()      ── Readability 正文
                    ├─→ handleScreenshotAndPdf() ── autoScroll → 截图/PDF
                    ├─→ handleMonolith()         ── 单文件 HTML (异步)
                    └─→ sendToWayback()          ── 提交 archive.org (异步)
                            │
                            └─→ finally: 标记 lastPreserved + 未生成格式为 unavailable
```
