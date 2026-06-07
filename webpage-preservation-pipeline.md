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
        ├─ 读回 finalLink，把 null 的字段标记为 "unavailable"
        ├─ context.close()
        └─ 若 link 已被删除 → removeFiles() 清理已落盘文件
```

### 2.5 页面内请求 SSRF 防护

[protectPageRequests.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/protectPageRequests.ts) 为每个新建的 BrowserContext 注册 `context.route("**/*")` 拦截器：

- `about:` / `blob:` / `data:` 协议 → 直接放行
- 其他 URL → 走 `assertUrlIsSafeForServerSideFetch()` 校验，非法则 `route.abort("blockedbyclient")`

这意味着即使被保存的页面中有恶意资源指向内网 IP，也会被 Playwright 路由层拦截。

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
    ├─ article.textContent 后处理
    │   ├─ 去重空格: replace(/ +(?= )/g, "")
    │   ├─ 去换行:   replace(/(\r\n|\n|\r)/gm, " ")
    │   └─ 长度截断: slice(0, TEXT_CONTENT_LIMIT)
    │
    ├─ Buffer 大小检查: < READABILITY_MAX_BUFFER (默认 100MB)
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
- **可选 keepContent**：参数 `keepContent=true` 时把完整净化后 HTML 塞进 `article.content`（未被默认路径调用）

---

## 四、静态资源保存（Monolith 单文件归档）

实现位于 [handleMonolith.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/handleMonolith.ts)。

### 4.1 Monolith 调用方式

使用外部 CLI 工具 `monolith`（需要系统安装），通过 `child_process.spawn` 调用：

```bash
monolith - \
  -I \                       # 移除 images
  -b {baseUrl} \             # 基础 URL 用于解析相对路径
  -j -F -q \                 # 默认选项：去掉 JS、去掉 iframe、静默
  -o -                       # 输出到 stdout
  # 从 stdin 读入 Playwright 渲染后的 HTML
```

> `-j` (no-js) 和 `-F` (no-iframes) 是默认行为，可通过 `MONOLITH_CUSTOM_OPTIONS` 环境变量完全覆盖。

### 4.2 异常与边界

- 通过 `AbortSignal` 与外层 5 分钟超时联动（`killSignal: "SIGKILL"`）
- 退出码非 0 → reject
- 输出 Buffer 为 0 → reject
- 输出 > `MONOLITH_MAX_BUFFER`（默认 100MB）→ reject
- **异常吞掉**：`archiveHandler.ts:189-193` 中 `.catch(err => console.error(err))`，即 Monolith 失败不影响其他格式产出

### 4.3 与 Playwright 的"回填"机制

在 [archiveHandler.ts:132-149](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/archiveHandler.ts#L132-L149) 有一段特殊逻辑：

如果 `link.monolith` 已经是 `.html` 结尾（说明通过客户端上传了 HTML），则：
1. 读取该文件内容
2. 用 `page.setContent(fileContent)` 替换浏览器页面
3. 后续的 Readability、Screenshot、PDF 全都基于这份**客户端上传的 HTML** 生成

这是客户端侧归档（`clientSide: true`）与服务端侧归档的桥梁。

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

**存在性检查**：`!link.image?.startsWith("archive")`，即已有归档路径则跳过。

### 5.2 预览小图 — [handleArchivePreview.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts)

优先级：
1. 读取页面 `<meta property="og:image">`，若存在则 `page.goto()` 该图片 URL，调用 `generatePreview(buffer)`
2. 回退：低质量截图 `page.screenshot({ type: "jpeg", quality: 20 })`

`generatePreview` 位于 [generatePreview.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/packages/lib/generatePreview.ts)，使用 Jimp：
- resize 宽度 1000px，高度自适应
- JPEG quality 20
- 上限 `PREVIEW_MAX_BUFFER`（10MB）
- 输出：`archives/preview/{cid}/{id}.jpeg`

### 5.3 直链图片/PDF — [imageHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/imageHandler.ts) / [pdfHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/preservationScheme/pdfHandler.ts)

当 `determineLinkType()` 通过 HEAD 请求的 `Content-Type` 判断出链接本身是图片或 PDF 时，不走浏览器，直接：
- `safeFetch(url).buffer()` 下载
- 生成预览
- 存文件 + 更新 DB
- `return` 提前退出，不执行后面的浏览器渲染流程

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

### 6.2 状态值语义

- **`null`（待处理）**：从未尝试过保存，或被手动重置（"重新归档"按钮会把所有字段置 `null` + `lastPreserved = null`）
- **`"archives/..."`（已成功）**：对应格式文件已落盘，值为相对路径
- **`"unavailable"`（已失败/不支持）**：已尝试但未产出，或 URL 本身不支持（非 http(s)、SSRF 不通过、`DISABLE_PRESERVATION`）

### 6.3 流转图

```
 链接创建 (postLink)
      │
      ├─ URL 不安全或 DISABLE_PRESERVATION
      │     └─ 立即 → 全部字段 = "unavailable"，lastPreserved = now
      │
      └─ 正常 → 全部字段 = null，lastPreserved = null
                 │
                 ▼
           Worker 拾取 (lastPreserved IS NULL)
                 │
                 ▼
           archiveHandler 执行
            ├─ 成功的格式 → 写入 "archives/..." 路径
            ├─ 失败的格式 → 保持 null
            └─ 格式用户没开 → 保持 null
                 │
                 ▼
           finally 块扫尾
            ├─ 每个仍为 null 的字段 → "unavailable"
            ├─ lastPreserved = now
            └─ indexVersion = null (触发重新索引)
                 │
                 ▼
           手动点击"重新归档" ([id]/archive PUT)
            ├─ image/pdf/readable/monolith/preview = null
            ├─ lastPreserved = null
            ├─ indexVersion = null
            ├─ clientSide = false
            └─ removeFiles() 清盘 → 重新进入 Worker 拾取
```

### 6.4 批量修复损坏归档

在 [preservation.tsx:67-164](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L67-L164) 中，管理员可执行 `action === "allBroken"`：

1. 找出该用户下任意字段为 `"unavailable"` 的链接
2. 根据用户/tag 设置判断该格式是否本应产出
3. 对"应该有但实际 unavailable"的字段，重置为 `null`
4. `lastPreserved = null` → 重新进入 Worker 拾取

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
2. `lastPreserved = now`（避免被反复拾取）
3. 未产出的快照字段被标记为 `"unavailable"`（避免 UI 永远 loading）
4. `indexVersion = null`（触发全文索引重建，即使失败也要更新搜索状态）
5. 浏览器 context 被关闭
6. 若链接已被用户删除 → 调用 `removeFiles()` 清理落盘文件

### 7.7 Monolith 软失败

[archiveHandler.ts:189-193](file:///d:/fz/0601/solo-dogfeeding/code/87-linkwarden/apps/worker/lib/archiveHandler.ts#L189-L193)：`handleMonolith().catch(err => console.error(err))`，Monolith 失败不会触发整条链接失败，其他格式仍会继续并最终标记完成。

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
- 不同链接即使内容完全相同，也会各自独立保存一份
- 没有内容哈希、没有 de-dupe 索引

### 10.3 资源内部去重（Monolith 层）

Monolith CLI 自身在把 CSS/图片 data-URI 化时是否有重复资源合并，取决于上游 `monolith` 工具实现，Linkwarden 代码层没有相关处理。

### 10.4 存在性检查（幂等性）

每种保存格式在执行前都会检查对应字段是否已有 `"archives/..."` 路径：

- `handleArchivePreview`：`!link.preview?.startsWith("archive")`
- `handleReadability`：`!link.readable`
- `handleScreenshotAndPdf`：`!link.image?.startsWith("archive")` / `!link.pdf?.startsWith("archive")`
- `handleMonolith`：`!link.monolith`

这保证了 Worker 即使重复拾取同一链接，也不会重复执行已成功的格式（但注意 `"unavailable"` 被当作不存在，因此失败的格式在重新归档时会重试）。

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
| `MONOLITH_CUSTOM_OPTIONS` | `-j -F -q` | 覆盖 monolith CLI 参数 |
| `PDF_MARGIN_TOP` / `PDF_MARGIN_BOTTOM` | 15px | PDF 上下边距 |
