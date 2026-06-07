# Webpage Preservation Pipeline 代码分析

## 一、整体架构概览

Linkwarden 的网页保存（Preservation）Pipeline 采用 **多 Worker 协同 + 浏览器自动化** 的架构，核心分为以下几个模块：

| 模块 | 位置 | 职责 |
|------|------|------|
| Worker 进程入口 | [worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/index.ts) | 启动并守护所有 Worker，异常退出后 5 秒自动重启 |
| Worker 主循环 | [worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/worker.ts) | 初始化并并行启动 5 类 Worker（linkProcessing、linkIndexing、autoTagPreservedLinks、RSS、trialEndEmail） |
| 链接批处理调度 | [getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/getLinkBatchFairly.ts) | 公平地从多个用户间分配待处理链接配额 |
| 核心保存 Handler | [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/archiveHandler.ts) | 单个链接的完整保存流程编排 |
| 保存方案 (Scheme) | [preservationScheme/](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme) | 7 种具体的保存格式实现 |
| 文件系统适配层 | [packages/filesystem/](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/packages/filesystem) | 支持本地磁盘与 S3 两种存储后端 |
| SSRF 防护层 | [ssrf.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/packages/lib/ssrf.ts) | 防止服务端请求伪造攻击 |
| 安全 HTTP 客户端 | [safeFetch.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/packages/lib/safeFetch.ts) | 封装了 SSRF 检查、代理、重定向限制的 fetch |

整体数据模型（Link 表快照字段）定义在 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/packages/prisma/schema.prisma#L166-L198)。

---

## 二、抓取流程流转分析

### 2.1 Worker 进程守护与启动链

```
[index.ts] spawn tsx worker.ts
    │  (进程退出 → setTimeout 5s 重启)
    ▼
[worker.ts: init()]
    ├─ migrationWorker()           // 先跑数据迁移
    ├─ startRSSPolling()           // RSS 轮询
    ├─ linkProcessing(10s)         // ★ 链接保存 (核心)
    ├─ autoTagPreservedLinks(10s)  // AI 自动打标
    ├─ startIndexing(10s)          // MeiliSearch 全文索引
    └─ trialEndEmailWorker()       // 试用期结束邮件
```

`linkProcessing` 是保存 Pipeline 的主循环入口，位于 [linkProcessing.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/workers/linkProcessing.ts)。

### 2.2 浏览器生命周期管理

在 [linkProcessing.ts:14-27](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/workers/linkProcessing.ts#L14-L27) 中：

- **浏览器启动**：`launchBrowser()` 调用 Playwright 的 `chromium.launch()` 或通过 CDP 连接远程浏览器（`PLAYWRIGHT_WS_URL`）
- **30 分钟自动重启**：`BROWSER_MAX_AGE_MS = 30 * 60 * 1000`，防止浏览器进程内存泄漏
- **断连自动重连**：每个链接处理失败后检测 `browser.isConnected()`，断连则立即重启

浏览器上下文配置在 [browser.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/browser.ts)：
- 模拟 `Desktop Chrome` 设备
- 支持 `PROXY` 环境变量配置 HTTP/SOCKS 代理
- 支持 `ALLOW_INSECURE_TLS` 忽略 HTTPS 错误

### 2.3 公平批处理调度 (Fair Scheduling)

`getLinkBatchFairly` 是关键调度器，防止单个用户"饿死"其他用户：

**挑选逻辑** ([getLinkBatchFairly.ts:12-178](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/getLinkBatchFairly.ts#L12-L178))：

1. 找出所有**有待处理链接**的付费/试用用户，按 `lastPickedAt` (上次被选中时间) 升序 + `id` 升序排列
2. 计算 `linksPerUser = floor(maxBatchLinks / users.length)`，默认每批 5 条
3. 轮询每个用户，依次取出 `linksPerUser` 条链接，直到凑满 `maxBatchLinks`
4. 更新这些用户的 `lastPickedAt = now`

**待处理链接判定**（mode = "links"）：
```sql
WHERE url IS NOT NULL AND lastPreserved IS NULL
ORDER BY createdAt DESC
```

### 2.4 单链接抓取主流程 (archiveHandler)

[archiveHandler.ts:25-231](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/archiveHandler.ts#L25-L231) 是整个 Pipeline 的核心编排函数：

```
archiveHandler(link, browser)
│
├─ 前置检查
│   ├─ DISABLE_PRESERVATION=true → 全部标记 unavailable 并返回
│   ├─ SSRF 安全检查 assertUrlIsSafeForServerSideFetch(url)
│   └─ 非 http(s) URL → 全部标记 unavailable 并返回
│
├─ 超时控制
│   └─ AbortController + BROWSER_TIMEOUT (默认 5 分钟)，超时抛错
│
├─ 浏览器上下文
│   ├─ browser.newContext(contextOptions)
│   ├─ protectPageRequests(context)  // ★ 页面内请求 SSRF 防护
│   └─ context.newPage()
│
├─ 归档设置解析 (ArchivalSettings)
│   ├─ 优先：链接上的 archival tags
│   └─ 回退：用户 User 表的默认设置
│
└─ 抓取执行 (Promise.race 与超时赛跑)
    ├─ ① determineLinkType() → 通过 HEAD 请求 Content-Type 判断 url/pdf/image
    ├─ ② Wayback Machine 异步提交 (fire-and-forget)
    ├─ ③ 按类型分支
    │   ├─ 图片类型 → imageHandler() 直接下载保存
    │   ├─ PDF 类型 → pdfHandler() 直接下载保存
    │   └─ URL 类型 → 浏览器渲染 page.goto()
    │       ├─ metaDescription 提取并存库
    │       ├─ content = page.content() → 获取渲染后 HTML
    │       ├─ handleArchivePreview()  → 生成预览小图
    │       ├─ handleReadability()     → Readability JSON
    │       ├─ handleScreenshotAndPdf()→ 全屏截图 + PDF
    │       └─ handleMonolith()        → 单文件 HTML 归档
    │
    └─ finally
        ├─ 清理 timeout
        ├─ 读回 finalLink: 仍为 null 的字段 → "unavailable" (终止态)
        ├─ lastPreserved = now (链接不再被 Worker 自动拾取)
        ├─ context.close()
        └─ 若 link 已被删除 → removeFiles() 清理已落盘文件
```

### 2.5 页面内请求 SSRF 防护

[protectPageRequests.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/protectPageRequests.ts) 为每个新建的 BrowserContext 注册 `context.route("**/*")` 拦截器：

- `about:` / `blob:` / `data:` 协议 → 直接放行
- 其他 URL → 走 `assertUrlIsSafeForServerSideFetch()` 校验，非法则 `route.abort("blockedbyclient")`

这意味着即使被保存的页面中有恶意资源指向内网 IP，也会被 Playwright 路由层拦截。

> ⚠️ **注意：此防护仅覆盖 Playwright 浏览器进程发起的请求。** Monolith CLI 作为独立子进程下载资源时（见 4.3 节），不走 Playwright 路由，因此其资源请求不在 SSRF 防护范围内。

---

## 三、Readability 内容提取

实现位于 [handleReadability.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/handleReadability.ts)。

### 3.1 处理流程

```
content (Playwright 渲染后的完整 HTML)
    │
    ├─ DOMPurify(window).sanitize(content)   // XSS 净化
    ├─ JSDOM(cleanedUpContent, { url })      // 构造 DOM
    ├─ new Readability(dom.document).parse() // Mozilla Readability 提取
    │
    ├─ article?.textContent 存在且非空?
    │   ├─ 否 → 静默 return，不写 DB 不抛错
    │   │       → 字段保持 null → finally 标记 "unavailable"
    │   └─ 是 → 继续
    │
    ├─ textContent 后处理
    │   ├─ 去重空格: replace(/ +(?= )/g, "")
    │   ├─ 去换行:   replace(/(\r\n|\n|\r)/gm, " ")
    │   └─ 长度截断: slice(0, TEXT_CONTENT_LIMIT)
    │
    ├─ Buffer 大小检查: < READABILITY_MAX_BUFFER (默认 100MB)
    │   └─ 超限 → console.error + return (静默失败)
    │
    ├─ createFile(
    │    data: JSON.stringify(article),
    │    path: archives/{collectionId}/{linkId}_readability.json
    │  )
    │
    └─ DB 更新
         ├─ readable = "archives/.../{id}_readability.json"
         └─ textContent = 处理后的纯文本（用于搜索索引）
```

### 3.2 关键特性

- **输入来源**：使用 Playwright 渲染后的 HTML（而非原始 HTTP 响应），对 SPA/动态渲染页面友好
- **双重净化**：先 DOMPurify 再 Readability，避免提取出的 HTML 中包含恶意脚本
- **双字段存储**：完整文章对象存 JSON 文件，纯文本摘要存 `Link.textContent` 字段供 MeiliSearch 建索引
- **静默失败**：Readability 无法提取有效文章（返回 null 或文本为空）、Buffer 超限均为静默 return，不抛错。字段保持 `null`，由 finally 块兜底标记为 `"unavailable"`（终止态）
- **可选 keepContent**：参数 `keepContent=true` 时把完整净化后 HTML 塞进 `article.content`（未被默认路径调用）

---

## 四、静态资源保存（Monolith 单文件归档）

实现位于 [handleMonolith.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/handleMonolith.ts)。

### 4.1 Monolith 参数与调用方式

使用外部 CLI 工具 `monolith`（Y2Z/monolith，Rust 编写），通过 `child_process.spawn` 调用，从 **stdin** 读入 HTML，向 **stdout** 写出单文件结果：

```bash
monolith \
  - \                        # 位置参数: 从 stdin 读取输入 HTML
  -I \                       # --isolate: 为输出 HTML 添加 CSP 沙箱隔离
  -b {link.url} \            # --base-url: 基础 URL，用于解析 HTML 中的相对路径
  -j -F -q \                 # 默认自定义选项 (可被 MONOLITH_CUSTOM_OPTIONS 完全覆盖)
                             #   -j = --no-js:         排除 JavaScript
                             #   -F = --no-webfonts:  排除 Web Fonts
                             #   -q = --quiet:         静默
  -o -                       # 输出到 stdout
  # stdin: 写入 Playwright 渲染后的 HTML (page.content())
```

参数来源：[handleMonolith.ts:14-24](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/handleMonolith.ts#L14-L24)，参考 Monolith 官方文档。

### 4.2 资源内联方式

Monolith 的默认行为是**把所有可解析到的外部资源转换为 base64 data-URI 内联到 HTML 中**。结合 Linkwarden 的参数：

| 资源类型 | 是否内联 | 控制参数 |
|----------|----------|----------|
| CSS 样式表 | ✅ 默认内联 | `-c` 可排除 |
| **图片** (`<img src>`、CSS `url()`) | ✅ **默认内联** | `-i` 才可排除（**默认不带**，所以图片会被内联） |
| JavaScript | ❌ 排除 | `-j`（默认带） |
| Web Fonts | ❌ 排除 | `-F`（默认带） |
| iframe | ⚠️ 取决于版本 | `-f` 可显式排除（默认不带，但 JS 被排除后 iframe 也通常无法渲染） |
| 音频 | ✅ 默认内联 | `-a` 可排除 |
| 视频 | ✅ 默认内联 | `-v` 可排除 |
| NOSCRIPT 内容 | ❌ 默认不提取 | `-n` 可开启 |

> **关键纠正**：之前误认为 `-I` 是"移除图片"——`-I` 实际是 `--isolate`（隔离/CSP 沙箱），移除图片需要 `-i`。Linkwarden 默认**不带** `-i`，因此**图片会被完整内联为 data-URI**。排除的是 JS（`-j`）和 Web Fonts（`-F`）。

### 4.3 资源请求的独立性与 SSRF 防护缺口

Monolith 是独立的子进程，其资源下载行为完全独立于 Playwright：

1. **输入**：通过 stdin 接收 `page.content()`（Playwright 渲染后 DOM 的序列化 HTML，其中 `<img src>`、`<link href>` 仍是原始 URL）
2. **资源下载**：Monolith 解析该 HTML 后，**自己发起 HTTP/HTTPS 请求**下载 CSS、图片等资源并转 base64
3. **会话隔离**：Monolith 不使用 Playwright 浏览器的 Cookie、localStorage、缓存；官方文档明确说明 "monolith is not aware of your browser's session"
4. **SSRF 防护缺口**：Monolith 发起的请求**不经过** [protectPageRequests.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/protectPageRequests.ts) 中注册的 Playwright 路由拦截器，因此对资源 URL 的 SSRF 检查在此处缺失。Playwright 只拦截了页面内浏览器发起的请求。
5. **代理**：Monolith 会读取系统环境变量 `https_proxy`、`http_proxy`、`no_proxy`（若 Playwright 也通过 `PROXY` 配置了代理则二者共用）

### 4.4 异常与边界

- **超时联动**：通过 `AbortSignal` 与外层 5 分钟浏览器超时联动，超时时 `killSignal: "SIGKILL"` 强杀子进程
- **退出码非 0** → reject
- **输出 Buffer 长度为 0** → reject（Monolith 没产出任何内容）
- **输出大小超限** → `> MONOLITH_MAX_BUFFER`（默认 100MB）时 reject
- **软失败隔离**：在 [archiveHandler.ts:189-193](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/archiveHandler.ts#L189-L193) 中用 `.catch(err => console.error(err))` 吞掉异常，Monolith 失败不会中断其他格式产出。

### 4.5 失败后的状态标记与成功时的 DB 更新时机

**成功路径**：成功时在 handleMonolith 内部立即写库，不等 finally：

[handleMonolith.ts:57-66](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/handleMonolith.ts#L57-L66)：
```typescript
await createFile({ data: html, filePath: `archives/${collectionId}/${id}.html` });
await prisma.link.update({
  where: { id: link.id },
  data: { monolith: `archives/${collectionId}/${id}.html` },
});
resolve();
```

成功写入 DB 后，`monolith` 字段从 `null` 变为具体路径字符串（truthy），finally 块读到该值时 `!truthy = false` → 传 `undefined` → 不动。

**失败路径**：handleMonolith 内部 reject 后，被外层 `.catch(err => console.error(err))` 吞掉：
1. 错误不抛到外层，不影响其他格式执行
2. `monolith` 字段**保持 `null`**
3. finally 块读到 `null` → `!null = true` → 标记为 `"unavailable"`
4. `"unavailable"` 是 truthy 字符串，进入终止态——不会自动重试，只有手动重置为 `null` 才会重新执行

### 4.6 与 Playwright 的"回填"机制

在 [archiveHandler.ts:131-149](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/archiveHandler.ts#L131-L149) 有一段特殊逻辑：

如果进入 archiveHandler 时 `link.monolith` 已经是 `.html` 结尾的路径（说明通过客户端上传了 HTML，见 `clientSide: true`），则：

1. 先执行 `page.goto(link.url)`（仍会发起请求，但结果会被覆盖）
2. 通过 [readFile](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/packages/filesystem/readFile.ts) 读取已有 HTML 文件
3. 用 `page.setContent(fileContent, { waitUntil: "domcontentloaded" })` 把 Playwright 页面替换为该 HTML
4. 后续的 `metaDescription` 提取、`content = page.content()`、`handleArchivePreview`、`handleReadability`、`handleScreenshotAndPdf` 全部基于这份客户端上传的 HTML 生成

这是客户端侧归档（`clientSide: true`）与服务端侧归档的桥梁：用户在自己浏览器登录后抓取页面上传，服务端基于该 HTML 生成其余格式。

---

## 五、其他保存格式实现

### 5.1 截图与 PDF — [handleScreenshotAndPdf.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts)

**前置动作：自动滚动**
- `page.evaluate(autoScroll, AUTOSCROLL_TIMEOUT 默认 30s)`
- 每 100ms 向下滚动 100px，直到触底或 30 秒超时
- 目的：触发懒加载图片/无限滚动内容

**并行产出**（`Promise.allSettled`，互不影响）：
- 截图：`page.screenshot({ fullPage: true, type: "jpeg" })` → `archives/{cid}/{id}.jpeg`，上限 `SCREENSHOT_MAX_BUFFER`（100MB）
- PDF：`page.pdf({ width: "1366px", height: "1931px", printBackground: true, margin: {top/bottom: 15px} })` → `archives/{cid}/{id}.pdf`，上限 `PDF_MAX_BUFFER`（100MB）

**双层存在性检查**：
1. 外层（archiveHandler.ts:176-179）：`archivalSettings.archiveAsScreenshot && !link.image`——字段为 `null` 才会调用此 handler（`"unavailable"` 和 `"archives/..."` 都是 truthy，被跳过）
2. 内层（handleScreenshotAndPdf.ts:22-24,58）：`!link.image?.startsWith("archive")`——进一步确保真实归档路径不重复生成

**静默失败**：Buffer 超限 `console.log` + `return`，截图/PDF Promise reject 被 `Promise.allSettled` 吞掉，字段保持 `null`，finally 标记为 `"unavailable"`。

### 5.2 预览小图 — [handleArchivePreview.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts)

优先级：
1. 读取页面 `<meta property="og:image">`，若存在则 `page.goto()` 该图片 URL，调用 `generatePreview(buffer)`
2. 回退：低质量截图 `page.screenshot({ type: "jpeg", quality: 20 })`

`generatePreview` 位于 [generatePreview.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/packages/lib/generatePreview.ts)，使用 Jimp：
- resize 宽度 1000px，高度自适应
- JPEG quality 20
- 上限 `PREVIEW_MAX_BUFFER`（10MB）
- 输出：`archives/preview/{cid}/{id}.jpeg`

**双层存在性检查**：
1. 外层（archiveHandler.ts:169）：`!link.preview`——字段为 `null` 才会调用此 handler
2. 内层（handleArchivePreview.ts:42,59）：`!link.preview?.startsWith("archive")`——仅真实归档路径会被内层拦截，`"unavailable"` 在此处不拦截（但外层已拦截，实际走不到）

**静默失败**：Buffer 超限 `console.log` + `return`，字段保持 `null`，finally 标记为 `"unavailable"`。

### 5.3 直链图片/PDF — [imageHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/imageHandler.ts) / [pdfHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/pdfHandler.ts)

当 `determineLinkType()` 通过 HEAD 请求的 `Content-Type` 判断出链接本身是图片或 PDF 时，不走浏览器，直接：
- `safeFetch(url).buffer()` 下载
- 生成预览
- 存文件 + 更新 DB
- `return` 提前退出，不执行后面的浏览器渲染流程

**外层跳过条件**（archiveHandler.ts:122,125）：`linkType === "image" && !link.image`（pdf 同理）。字段为 `null` 才会调用，`"unavailable"` 和真实路径都会跳过。

**失败行为**：与其他格式不同，imageHandler/pdfHandler **没有** `.catch()` 包裹。如果下载失败（safeFetch 抛错、Buffer 超限 return 之前的网络错误等），异常会抛到外层，导致整个 archiveHandler 进入 catch 分支并 `throw err`——随后 linkProcessing 会捕获该异常并打印日志。但无论如何，finally 块仍会执行，把已为 `null` 的字段标记为 `"unavailable"`。

Buffer 超限的情况是静默 return，同其他格式。

### 5.4 Wayback Machine — [sendToWayback.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/sendToWayback.ts)

火并 forget 模式（无 await），直接 GET `https://web.archive.org/save/{url}`，成功失败都不影响主流程。

---

## 六、快照状态流转

### 6.1 Link 表的 6 个快照字段

在 [schema.prisma:183-187](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/packages/prisma/schema.prisma#L183-L187)：

| 字段 | 类型 | 含义 | 可能值 |
|------|------|------|--------|
| `preview` | String? | 预览图路径 | `null` / `"archives/preview/..."` / `"unavailable"` |
| `image` | String? | 完整截图路径 | `null` / `"archives/..."` / `"unavailable"` |
| `pdf` | String? | PDF 路径 | `null` / `"archives/..."` / `"unavailable"` |
| `readable` | String? | Readability JSON 路径 | `null` / `"archives/..."` / `"unavailable"` |
| `monolith` | String? | Monolith HTML 路径 | `null` / `"archives/..."` / `"unavailable"` |
| `lastPreserved` | DateTime? | 最近一次处理时间戳 | `null` / 时间 |

### 6.2 状态值语义与 JavaScript 真值

在分析之前，先明确 JavaScript 中的布尔判断（这是理解整个状态机的关键）：

```javascript
!null          // → true   (null 是 falsy)
!undefined     // → true
!"unavailable" // → false  (非空字符串是 truthy，!truthy = false)
!"archives/123.jpeg" // → false (非空字符串是 truthy)
```

| 字段值 | JavaScript 真值 | 含义 | Worker 行为 |
|--------|----------------|------|------------|
| `null` | falsy | **待处理/已重置** | `!null = true` → 会执行该格式 handler |
| `"archives/..."` | truthy | **已成功** | `!"archives/..." = false` → 跳过 |
| `"unavailable"` | truthy | **已终止** | `!"unavailable" = false` → **跳过，不会自动重试** |

**三个状态的精确定义**：

- **`null`（待处理）**：从未尝试过保存，或通过"重新归档"/"批量修复"被手动重置。只有 `null` 才会触发格式 handler 执行。
- **`"archives/..."`（已成功）**：对应格式文件已落盘，值为相对路径。在各 handler 内部成功写盘后**立即**写入 DB，不等 finally。
- **`"unavailable"`（已终止）**：经过一轮完整处理后该格式仍未产出。这是一个**终止态**——一旦被标记，除非被外部 API 重置回 `null`，否则永远不会再执行。可能原因：
  1. URL 本身不支持（非 http(s)、SSRF 不通过、`DISABLE_PRESERVATION` 全局关闭）
  2. 该格式执行失败（被 catch 吞掉或抛错）
  3. **用户未开启该格式**（finally 块不区分"失败"和"未开启"）

### 6.3 archiveHandler 外层的格式跳过条件

在 [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/archiveHandler.ts) 中，每个格式的 handler 被调用前都有外层判断：

| 格式 | 代码行 | 外层判断条件 | 字段为 `"unavailable"` 时结果 | 字段为 `null` 时结果 |
|------|--------|-------------|--------------------------|---------------------|
| preview | L169 | `!link.preview` | `!"unavailable"` = false → **跳过** | `!null` = true → 执行 |
| readable | L172 | `archiveAsReadable && !link.readable` | `!"unavailable"` = false → **跳过** | `!null` = true → 执行 |
| image(截图) | L177 | `archiveAsScreenshot && !link.image` | `!"unavailable"` = false → **跳过** | `!null` = true → 执行 |
| pdf | L178 | `archiveAsPDF && !link.pdf` | `!"unavailable"` = false → **跳过** | `!null` = true → 执行 |
| monolith | L185-186 | `archiveAsMonolith && !link.monolith` | `!"unavailable"` = false → **跳过** | `!null` = true → 执行 |
| imageHandler(直链图) | L122 | `linkType==="image" && !link.image` | `!"unavailable"` = false → **跳过** | `!null` = true → 执行 |
| pdfHandler(直链PDF) | L125 | `linkType==="pdf" && !link.pdf` | `!"unavailable"` = false → **跳过** | `!null` = true → 执行 |

> **核心结论**：`"unavailable"` 与 `"archives/..."` 在跳过判断上等价——二者都是 truthy 字符串，都会导致 `!link.xxx = false`，从而跳过对应 handler。只有 `null` 才会触发执行。

### 6.4 各 handler 内部的第二层跳过条件

部分 handler 在被调用后，内部还有更严格的 `startsWith("archive")` 检查：

**handleArchivePreview** ([handleArchivePreview.ts:42,59](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts#L42-L59))：
```typescript
!link.preview?.startsWith("archive")
```
- `"unavailable".startsWith("archive")` → false → `!false` = true → 内部不拦截（但外层 `!link.preview=false` 已跳过，实际走不到这里）
- `"archives/preview/...".startsWith("archive")` → true → `!true` = false → 内部拦截

**handleScreenshotAndPdf** ([handleScreenshotAndPdf.ts:22-24,58](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L22-L24))：
```typescript
!link.image?.startsWith("archive")
!link.pdf?.startsWith("archive")
```
同上逻辑。

**handleReadability / handleMonolith / imageHandler / pdfHandler**：内部没有二次检查，只要被外层调用就直接执行。

### 6.5 finally 块的精确标记逻辑与幂等性

[archiveHandler.ts:208-224](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/archiveHandler.ts#L208-L224)：

```typescript
const finalLink = await prisma.link.findUnique({ where: { id: link.id } });

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

**关键观察**：

1. **判定条件完全不涉及 `archivalSettings`**。只检查从 DB 重新读取的 `finalLink` 字段是否为 truthy。
2. **无论用户是否开启该格式**，只要最终字段未被成功格式写入具体路径（即仍为 `null`），就会被标记为 `"unavailable"`。
3. **`"unavailable"` 既表示"尝试了但失败"，也表示"用户根本没开启该格式"**——这两种情况在 DB 中无法区分。
4. **幂等性**：如果 `finalLink.readable` 已经是 `"unavailable"`，则 `!"unavailable"` = false，传入 `undefined`，Prisma **不会修改该字段**。因此 finally 块对已标记 unavailable 的字段是安全的重复调用。

### 6.6 以 Monolith 为例的完整状态流转

```
  monolith 字段起始值 = null (新创建链接)
       │
       ├─ Worker 拾取 (lastPreserved IS NULL)
       │
       ├─ 场景A：用户没开 archiveAsMonolith
       │    ├─ archiveHandler L184: !archivalSettings.archiveAsMonolith → false
       │    ├─ handleMonolith 不被调用 → 字段维持 null
       │    ├─ finally: !null = true → 标记 "unavailable"
       │    ├─ lastPreserved = now()
       │    └─ 结果：terminated (终止态，不会再被 Worker 拾取)
       │
       ├─ 场景B：用户开了，但 Monolith 子进程失败
       │    ├─ handleMonolith 被调用，但在内部 reject
       │    ├─ 错误被 .catch(console.error) 吞掉 → 不抛到外层
       │    ├─ 字段维持 null
       │    ├─ finally: !null = true → 标记 "unavailable"
       │    ├─ lastPreserved = now()
       │    └─ 结果：terminated (同上)
       │
       ├─ 场景C：用户开了，且 Monolith 成功
       │    ├─ handleMonolith 成功写盘 → **立即** update DB: monolith = "archives/..."
       │    ├─ finally: !"archives/..." = false → 传 undefined，不动
       │    ├─ lastPreserved = now()
       │    └─ 结果：success
       │
       └─ 场景D：手动点击"重新归档" (PUT /api/v1/links/[id]/archive)
            ├─ DB 更新: monolith = null, lastPreserved = null
            ├─ removeFiles() 删除磁盘文件
            └─ 回到起始状态 null → Worker 再次拾取，从场景 A/B/C 重新开始
```

### 6.7 流转总图

```
 链接创建 (postLink)
      │
      ├─ URL 不安全或 DISABLE_PRESERVATION
      │     └─ 立即 → 全部字段 = "unavailable"，lastPreserved = now
      │        结果：terminated
      │
      └─ 正常 → 全部字段 = null，lastPreserved = null
                 │
                 ▼
           Worker 拾取 (lastPreserved IS NULL)
                 │
                 ▼
           archiveHandler 执行
            ├─ 外层判断：仅 null 字段对应格式会被执行
            ├─ 成功格式 → handler 内部立即写盘 + DB 写入 "archives/..."
            ├─ 失败格式 → 错误被 catch，字段保持 null
            └─ 用户未开格式 → handler 不调用，字段保持 null
                 │
                 ▼
           finally 块
            ├─ 每个仍为 null 的字段 → "unavailable" (terminated)
            ├─ 已是 "unavailable" 或 "archives/..." 的字段 → 不动 (幂等)
            ├─ lastPreserved = now()
            └─ indexVersion = null
                 │
                 ▼
           后续路径
            ├─ 手动重新归档 PUT: 所有字段 → null, lastPreserved → null → 重新拾取
            └─ 批量修复 allBroken DELETE: 仅"应该产出但 unavailable"的字段 → null → 重新拾取
```

### 6.8 批量修复损坏归档 (allBroken)

在 [preservation.tsx:67-164](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L67-L164) 中，管理员可执行 `action === "allBroken"`：

1. **精确查找**：`WHERE image = "unavailable" OR pdf = "unavailable" OR ...`（精确字符串匹配）
2. **重新计算**该链接对应的 `archivalSettings`（Tag 优先，用户默认回退）
3. **条件重置**：仅当 `字段 === "unavailable" && 用户确实开启了该格式` 时，才把该字段改为 `null`
4. `lastPreserved = null` → Worker 重新拾取

这是为了弥补 finally 块"不区分失败和未开启"的设计缺陷——修复逻辑只重置那些真正应该产出但失败了的字段，用户没开的格式保持 `"unavailable"` 不动。

---

## 七、失败恢复机制

### 7.1 进程级崩溃恢复

[index.ts:3-12](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/index.ts#L3-L12)：使用 `child_process.spawn` 包裹 worker，`on("exit")` → `setTimeout(launch, 5000)` 无限重启。SIGINT 除外。

### 7.2 单链接超时

[archiveHandler.ts:63-75](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/archiveHandler.ts#L63-L75)：`BROWSER_TIMEOUT`（默认 5 分钟）后通过 `AbortController.abort()` 中断，外层 `Promise.race` 抛错。

注意：这个超时是**整个单链接处理**（从浏览器上下文创建到所有格式保存完成）的总时间，而非单个网络请求。

### 7.3 浏览器断连恢复

[linkProcessing.ts:65-67](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/workers/linkProcessing.ts#L65-L67)：每个链接处理失败后检测 `browser.isConnected?.()`，为 false 则立即重启浏览器。

### 7.4 30 分钟浏览器轮换

[linkProcessing.ts:31-33](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/workers/linkProcessing.ts#L31-L33)：`Date.now() - browserStartTs >= 30min` 主动重启，避免 Playwright 浏览器进程长期运行导致的内存泄漏。

### 7.5 并行隔离

[linkProcessing.ts:71-72](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/workers/linkProcessing.ts#L71-L72)：一批内多个链接用 `Promise.allSettled` 并行，单个失败不影响同批其他链接。

### 7.6 finally 兜底

无论成功失败，[archiveHandler.ts:203-229](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/archiveHandler.ts#L203-L229) 的 finally 块保证：
1. timeout 被清除
2. `lastPreserved = now`——写入当前时间戳，使 `lastPreserved IS NULL` 的 SQL 查询不再命中，该链接**不会被 Worker 再次自动拾取**
3. 仍为 `null` 的快照字段被标记为 `"unavailable"`——进入终止态，后续即便被重新拾取也会因 `!"unavailable" = false` 被跳过
4. `indexVersion = null`——触发全文索引重建（即使失败也要更新搜索状态）
5. 浏览器 context 被关闭
6. 若链接已被用户删除 → 调用 `removeFiles()` 清理已落盘文件

> 注意：第 3 点中，已为 `"unavailable"` 或 `"archives/..."` 的字段因 truthy 会被传入 `undefined`，Prisma 不做修改——因此 finally 对这些字段是幂等的。

### 7.7 各格式的静默失败模式

各保存格式的失败不会触发整条链接失败，具体模式：

**Monolith** ([archiveHandler.ts:189-193](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/archiveHandler.ts#L189-L193))：
```typescript
await handleMonolith(...).catch(err => console.error(err));
```
显式 `.catch()` 吞掉异常，仅打日志。

**Readability** ([handleReadability.ts:26-61](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/handleReadability.ts#L26-L61))：
当 `articleText` 为空或 `undefined`（Readability 没能提取出有效文章内容）时，函数直接 `return`，不抛错也不写 DB，字段维持 `null`，等待 finally 标记为 `"unavailable"`。Buffer 超限时也是 `console.error` + `return`。

**handleArchivePreview / handleScreenshotAndPdf**：内部用 `.then().catch()` 或 `Promise.allSettled`，截图/PDF 异常被吞掉。Buffer 超限时 `console.log` + `return`。

**imageHandler / pdfHandler**：未被 `.catch()` 包裹，异常会正常抛出。但这两个 handler 仅在 URL 的 `Content-Type` 本身就是 image/pdf 时才会被调用，且外层是 `async () => { ... }`，异常会抛到 `Promise.race`，导致整个 archiveHandler 异常——此时 finally 仍会执行，把尚未写入的字段标记为 `"unavailable"`。

因此，除直链 image/pdf 外的所有格式都具备"静默失败 + finally 兜底 unavailable"的特性。单格式失败不会中断其他格式的执行。

---

## 八、动态网页处理边界

### 8.1 已覆盖的动态性

- **SPA/前端渲染**：`page.goto(url, { waitUntil: "domcontentloaded" })` + Playwright 真实 Chromium，执行页面 JS，拿到的是渲染后 DOM
- **懒加载图片**：`autoScroll` 自动滚动页面（最长 30s），触发 img `loading="lazy"` 和 IntersectionObserver 等
- **Meta/OG 标签**：渲染后 `document.querySelector` 提取，动态注入的也能拿到
- **相对 URL 解析**：Monolith `-b {url}` 传入基础 URL，Readability 用 `JSDOM(..., { url })` 构造 DOM

### 8.2 未覆盖的动态性

- **需要用户交互**：如"点击展开更多""登录后查看"等交互流程完全未处理
- **无限滚动内容**：`autoScroll` 只滚到 `document.body.scrollHeight`，动态加载更多的场景下，30 秒超时可能不够或永远触底不了
- **Canvas/WebGL 渲染内容**：Readability 提取不到画布内容；截图能保存视觉结果，但文本索引丢失
- **WebSocket/Server-Sent Events 实时流**：`waitUntil: "domcontentloaded"` 阶段这些通道刚建立，后续推送内容不会被捕获
- **长任务阻塞**：页面主线程长任务 >50ms 时，Playwright 的 `evaluate` 可能排队，但无显式处理

---

## 九、登录页面处理边界

### 9.1 现状：服务端侧归档完全不处理登录

Pipeline 中**没有任何机制**向 Playwright 浏览器注入登录态 Cookie/Token：

- `getDefaultContextOptions()` 只配置了设备模拟和 TLS 选项，无 `storageState`、无 `extraHTTPHeaders`
- 没有"用户登录凭证管理"相关的数据库表或 API
- Playwright 上下文是每个链接新建一次（`browser.newContext()`），上下文间完全隔离，无法累积会话

### 9.2 对登录墙页面的实际行为

```
页面要求登录 → 302 到登录页 或 返回 200 但内容是登录表单
    ↓
page.goto() 正常完成
    ↓
对登录页 DOM 执行 Readability / Screenshot / Monolith
    ↓
最终产出的是"登录页面"的快照，而非原始 URL 内容
    ↓
finally 块标记该链接所有格式为 available/unavailable
    ↓
lastPreserved 被写入 → Worker 不再重试
```

**没有重试、没有检测、没有告警**，用户只会看到自己保存的链接全是登录页截图。

### 9.3 绕过路径：客户端侧归档 (clientSide)

[archives/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/web/pages/api/v1/archives/index.ts) 提供了 POST `/api/v1/archives?format={0..4}&preview=1` 接口：

1. 用户在自己浏览器里登录目标网站、看到完整内容
2. 前端用浏览器扩展/手动方式抓取当前页面 HTML 或截图
3. 上传文件到该接口，直接写入 `archives/{cid}/{id}.{suffix}`
4. DB 更新：`clientSide = true`，`lastPreserved = null`（重新触发 Worker）

此时 Worker 里的 [archiveHandler.ts:132-149](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/archiveHandler.ts#L132-L149) 会**用上传的 HTML 替换 page 内容**，再基于它生成 Readability、截图、PDF。

这是目前唯一能可靠保存登录后页面的方式。

---

## 十、资源去重边界

### 10.1 链接级去重

在 [postLink.ts:47-67](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/web/lib/api/controllers/links/postLink.ts#L47-L67) 实现，由用户配置 `preventDuplicateLinks` 开关：

```
对输入 URL:
  1. trim() + 去掉末尾斜杠
  2. 生成 { www 版本, 非 www 版本 } 两个候选
  3. WHERE ownerId = 当前用户 AND (url = www版 OR url = 非www版)
  4. 命中 → 返回 409 "Link already exists"
```

**边界特征**：
- ✅ 覆盖 `http://www.example.com` vs `http://example.com`
- ❌ 不覆盖 `http` vs `https` 差异
- ❌ 不覆盖 query string 顺序差异、`utm_*` 跟踪参数差异
- ❌ 不覆盖 URL fragment (`#section`)
- ❌ 不覆盖相同内容不同域名（如镜象站）
- ✅ 仅在同一用户（`ownerId`）范围内去重，不同用户互不影响
- ❌ 仅创建时检查，更新链接 URL 时不复用此逻辑

### 10.2 文件级去重

**完全没有实现**。所有保存格式都使用固定路径 `archives/{collectionId}/{linkId}.{suffix}`：

- 同一链接多次触发"重新归档"会直接**覆盖**旧文件（`createFile` 是写覆盖语义）
- 不同链接即使内容完全相同（例如同一篇文章被保存两次、不同用户保存同一 URL），也会各自独立保存一份
- 没有内容哈希、没有 de-dupe 索引、没有跨链接的字节级比较

### 10.3 Monolith 内部资源去重

Monolith 的输出是单 HTML 文件，所有资源都已被序列化为 base64 data-URI 嵌入其中。关于去重：

1. **单文件内部重复资源**：取决于 Monolith 自身实现。如果同一 HTML 中多个 `<img>` 引用相同 URL，Monolith **可能**只下载一次并复用 base64 字符串（Linkwarden 代码层未干预，取决于上游 Rust 实现）。
2. **跨链接资源去重**：完全不存在。即便链接 A 和链接 B 的页面都引用了同一张 `https://cdn.example.com/logo.png`，Monolith 也会分别下载、分别 base64 编码、分别嵌入到各自的 `.html` 文件中——两份完全相同的 base64 字符串各自占用磁盘空间。
3. **与 Playwright 缓存的关系**：Playwright 在 `page.goto()` 渲染时已经下载过一次 CSS/图片资源，但 Monolith 是独立子进程，完全不知道这些缓存，会**重新发起独立 HTTP 请求**下载所有引用的资源并内联。这意味着同一份资源在服务端至少被下载两次（一次 Playwright 渲染，一次 Monolith 内联），可能更多（截图又触发一次资源加载）。

### 10.4 资源内联后的不可逆性与去重失效

Monolith 输出的 HTML 中，资源已被转换为 base64 data-URI：

```html
<img src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...">
```

这带来两个去重层面的副作用：

1. **无法在文件系统层识别重复资源**：data-URI 直接嵌在 HTML 里，没有独立文件，系统级文件去重（如 ZFS dedup、S3 相同对象合并）无法在资源粒度生效。
2. **无法通过 URL 追踪来源**：输出文件中不再保留原始资源 URL，因此也无法事后通过 URL 做反向去重。

### 10.5 存在性检查（幂等性）与重试触发条件

每种保存格式的执行都需要同时满足"用户开启"和"字段为 null"两个条件：

| 格式 | 外层判断代码 | 判断逻辑拆解 |
|------|-------------|-------------|
| preview | `!link.preview` | `null` → true (执行); `"unavailable"` → false (跳过); `"archives/..."` → false (跳过) |
| readable | `archiveAsReadable && !link.readable` | 需同时满足: 开关=true **且** 字段=null |
| image | `archiveAsScreenshot && !link.image` | 同上 |
| pdf | `archiveAsPDF && !link.pdf` | 同上 |
| monolith | `archiveAsMonolith && !link.monolith` | 同上 |

**关于 `"unavailable"` 的关键事实**：
- 在 JavaScript 中，非空字符串 `"unavailable"` 是 **truthy** 的
- `!"unavailable"` → `false`
- 因此字段一旦被标记为 `"unavailable"`，就会被当作"已处理完成"而**永远跳过**
- `"unavailable"` 是一个**终止态**，不会自动重试

**重试的唯一触发方式：把字段重置为 `null`**

实现这一点的两个入口：

1. **手动"重新归档"按钮** ([links/[id]/archive/index.ts:51-67](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/web/pages/api/v1/links/%5Bid%5D/archive/index.ts#L51-L67))：PUT 请求把所有字段重置为 `null`，同时把 `lastPreserved` 重置为 `null`，并 `removeFiles()` 删除磁盘文件。
2. **管理员批量修复 allBroken** ([preservation.tsx:67-164](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L67-L164))：仅把"用户确实开启了该格式且字段为 `"unavailable"`"的字段重置为 `null`。

只有当字段被重置为 `null`（falsy）时，`!link.xxx` 才会返回 `true`，对应的格式 handler 才会被再次执行。

另外，部分 handler 内部还有第二层 `startsWith("archive")` 检查（见 6.4 节），但由于外层已经用 `!link.xxx` 把 `"unavailable"` 和 `"archives/..."` 都过滤掉了，这个内层检查对 unavailable 实际上没有影响——handleArchivePreview 和 handleScreenshotAndPdf 在字段为 unavailable 时根本不会被调用。

### 10.6 Monolith 与去重的关系总结

| 去重维度 | 现状 |
|----------|------|
| 同一链接多次归档 | ✅ 覆盖旧文件，不产生新副本 |
| 不同链接保存相同 URL | ❌ 独立保存，无去重 |
| 同一 HTML 内重复资源 | ⚠️ 取决于 Monolith 上游实现 |
| 跨链接相同资源（图片/CSS） | ❌ 完全无去重，均 base64 独立内联 |
| Playwright 缓存复用 | ❌ Monolith 独立子进程，重新下载所有资源 |
| 存储层字节级去重 | ❌ 未实现（依赖底层文件系统自行支持） |

---

## 十一、存储层与文件布局

[packages/filesystem/](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/packages/filesystem) 提供统一抽象：

| 函数 | 作用 |
|------|------|
| `createFile({ filePath, data })` | 写入；若配置 S3 则走 S3，否则写本地 `data/` 目录（相对 cwd 上两级） |
| `readFile({ filePath })` | 读回 Buffer + contentType |
| `removeFile({ filePath })` | 删除单文件 |
| `removeFiles(linkId, collectionId)` | 批量删除一个链接的所有格式文件 |
| `moveFiles(linkId, fromCid, toCid)` | 链接迁移集合时整体搬移 |
| `createFolder({ filePath })` | 本地模式下 mkdir -p；S3 模式无操作 |

目录结构（`{STORAGE_FOLDER}` 默认为 `data`）：
```
data/
  archives/
    {collectionId}/
      {linkId}.jpeg              # 截图
      {linkId}.pdf               # PDF
      {linkId}.html              # Monolith
      {linkId}_readability.json  # Readability
      {linkId}.png               # PNG 直链
    preview/
      {collectionId}/
        {linkId}.jpeg            # 预览小图
```

---

## 十二、配置项汇总

| 环境变量 | 默认值 | 作用 |
|----------|--------|------|
| `DISABLE_PRESERVATION` | false | 全局禁用服务端归档，全部立即标 unavailable |
| `ARCHIVE_SCRIPT_INTERVAL` | 10 (秒) | Worker 空转时轮询间隔 |
| `ARCHIVE_TAKE_COUNT` | 5 | 每批处理链接数 |
| `BROWSER_TIMEOUT` | 5 (分钟) | 单链接保存总超时 |
| `AUTOSCROLL_TIMEOUT` | 30 (秒) | 自动滚动加载懒加载内容时长 |
| `TEXT_CONTENT_LIMIT` | (无限) | Readability 文本截断长度 |
| `READABILITY_MAX_BUFFER` | 100 (MB) | Readability JSON 大小上限 |
| `MONOLITH_MAX_BUFFER` | 100 (MB) | Monolith HTML 大小上限 |
| `SCREENSHOT_MAX_BUFFER` | 100 (MB) | 截图 JPEG 大小上限 |
| `PDF_MAX_BUFFER` | 100 (MB) | PDF 大小上限 |
| `PREVIEW_MAX_BUFFER` | 10 (MB) | 预览图大小上限 |
| `IGNORE_URL_SIZE_LIMIT` | false | 跳过 HEAD 请求直接按 URL 类型处理 |
| `ALLOW_PRIVATE_NETWORK_ACCESS` | false | 关闭 SSRF 防护（开发用） |
| `ALLOW_INSECURE_TLS` / `IGNORE_HTTPS_ERRORS` | false | 忽略自签名/过期证书 |
| `PROXY` | - | Playwright + safeFetch 共用代理 URL |
| `PLAYWRIGHT_WS_URL` | - | 连接远程浏览器（CDP） |
| `MONOLITH_CUSTOM_OPTIONS` | `-j -F -q` | 覆盖 monolith CLI 选项（默认: 排除 JS、排除 Web Fonts、静默）。注意 `-I` (隔离/CSP沙箱) 是硬编码的，不受此变量覆盖 |
| `PDF_MARGIN_TOP` / `PDF_MARGIN_BOTTOM` | 15px | PDF 上下边距 |
