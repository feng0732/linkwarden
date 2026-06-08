# PDF / Image Archive 归档链路分析

## 一、整体架构概览

Linkwarden 的归档链路由 **Worker 进程** 异步驱动，核心流程如下：

```
用户创建 Link / 手动触发重跑
              ↓
linkProcessing Worker 轮询（每 10s）
              ↓
getLinkBatchFairly 公平调度获取待处理批次
              ↓
archiveHandler 归档主处理（Playwright 浏览器）
              ↓
determineLinkType → 分流到 imageHandler / pdfHandler / handleScreenshotAndPdf
              ↓
文件落盘（本地 FS 或 S3）+ DB 更新
              ↓
finally 块统一处理状态收尾（null → "unavailable" 收敛）
```

核心文件（仓库相对路径）：
- Worker 入口：`apps/worker/worker.ts`
- 链接处理循环：`apps/worker/workers/linkProcessing.ts`
- 归档主处理器：`apps/worker/lib/archiveHandler.ts`

---

## 二、格式选择逻辑

### 2.1 两级配置源

归档格式由 **Tag 级配置** 和 **User 级默认配置** 共同决定，Tag 优先：

```typescript
// apps/worker/lib/archiveHandler.ts L85-L107
const archivalTags = link.tags.filter(isArchivalTag);
const archivalSettings: ArchivalSettings =
  archivalTags.length > 0
    ? { /* 从 Tag 的 archiveAs* 字段聚合 */ }
    : { /* 从 User 的 archiveAs* 字段取默认值 */ };
```

判定 Tag 是否为归档 Tag 的依据：只要 Tag 上任意一个 `archiveAs*` 或 `aiTag` 字段被显式设置为 boolean 类型（非 null）。

相关文件：
- Tag 判定：`packages/lib/isArchivalTag.ts`
- User 默认字段（`packages/prisma/schema.prisma` L62-L66）：`archiveAsScreenshot/Monolith/PDF/Readable/WaybackMachine`
- Tag 可选字段（`packages/prisma/schema.prisma` L206-L211）：同名但允许 null

### 2.2 链接类型分流

在实际生成之前，先通过 HTTP HEAD 请求获取 `Content-Type` 来判定链接类型并做**智能分流**：

| Content-Type 前缀 | linkType | 处理器 | 是否启动浏览器 |
|---|---|---|---|
| `image/*` | `image` | `imageHandler` | ❌ 直接下载二进制 |
| `application/pdf` | `pdf` | `pdfHandler` | ❌ 直接下载二进制 |
| 其他 | `url` | Playwright 页面导航 | ✅ 启动浏览器 |

相关代码：`apps/worker/lib/archiveHandler.ts` L233-L263，`apps/worker/lib/fetchHeaders.ts`

HEAD 请求有 **10 秒超时**保护。

---

## 三、截图与 PDF 生成逻辑

### 3.1 原生 PDF/图片链接（非 HTML 页面）

#### imageHandler

- 路径：`apps/worker/lib/preservationScheme/imageHandler.ts`
- 使用 `safeFetch` 直接下载 URL 到 Buffer
- **先调用 `generatePreview` 生成缩略图**（1000px 宽 + quality=20）
- 再把原图写入 `archives/{collectionId}/{id}.{png|jpeg}`
- 大小限制：`SCREENSHOT_MAX_BUFFER`（默认 100MB）

#### pdfHandler

- 路径：`apps/worker/lib/preservationScheme/pdfHandler.ts`
- 直接 `safeFetch` 下载 Buffer
- 大小限制：`PDF_MAX_BUFFER`（默认 100MB）
- **不生成缩略图**（上传 PDF 时 DB 中 `preview` 会被标记为 "unavailable"）

### 3.2 HTML 页面 → 截图/PDF

由 `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts` 统一处理：

**前置步骤：自动滚动加载**

```typescript
await page.evaluate(autoScroll, AUTOSCROLL_TIMEOUT || 30);
```
- 每 100ms 滚动 100px，直到触底或超时（默认 30s）
- 确保懒加载图片/内容被加载后再截图

**截图（Screenshot）**：
- Playwright `page.screenshot({ fullPage: true, type: "jpeg" })`
- 格式固定为 **JPEG**，文件扩展名为 `.jpeg`
- 大小限制：`SCREENSHOT_MAX_BUFFER`（默认 100MB）

**PDF 生成**：
- Playwright `page.pdf({ width: "1366px", height: "1931px", printBackground: true, margin })`
- 固定纸张尺寸（非自适应页面长度）
- 上下边距通过 `PDF_MARGIN_TOP/BOTTOM` 配置（默认 15px）
- 大小限制：`PDF_MAX_BUFFER`（默认 100MB）

两个操作通过 `Promise.allSettled` **并行执行**，互不阻塞，单个失败不影响另一个。

### 3.3 safeFetch 与 SSRF 防护

- 路径：`packages/lib/safeFetch.ts`
- 每一次请求都会通过 `assertUrlIsSafeForServerSideFetch` 做 SSRF 校验
- 自定义 DNS lookup，防止访问内网 IP
- 支持 HTTP/HTTPS/SOCKS 代理
- 最多跟随 5 次重定向
- 浏览器端的页面资源请求也通过 `protectPageRequests` 做了同样的 SSRF 拦截（`apps/worker/lib/protectPageRequests.ts`）

---

## 四、文件元数据与存储

### 4.1 目录结构与命名规范

所有归档文件统一按 `collectionId` 分目录存储：

| 内容类型 | 存储路径 |
|---|---|
| 图片截图 | `archives/{collectionId}/{linkId}.jpeg` 或 `.png` |
| PDF | `archives/{collectionId}/{linkId}.pdf` |
| Monolith HTML | `archives/{collectionId}/{linkId}.html` |
| Readability JSON | `archives/{collectionId}/{linkId}_readability.json` |
| 预览缩略图 | `archives/preview/{collectionId}/{linkId}.jpeg` |

format → suffix 的映射在 `apps/web/lib/shared/getSuffixFromFormat.ts`。

### 4.2 存储后端抽象

- 文件系统包：`packages/filesystem/`
- 双后端支持：本地磁盘（默认 `data/` 目录，通过 `STORAGE_FOLDER` 配置）或 S3 兼容存储（配置 `SPACES_BUCKET_NAME` 等）
- 核心函数：
  - `packages/filesystem/createFile.ts`：写文件，自动 mkdir
  - `packages/filesystem/readFile.ts`：读文件 + 根据扩展名推断 Content-Type
  - `packages/filesystem/removeFile.ts`：删文件（S3 DeleteObject 或 fs.unlink）

### 4.3 DB 字段（Link 模型）

在 `packages/prisma/schema.prisma` L166-L198 中：

| 字段 | 类型 | 含义 | 状态值（三态） |
|---|---|---|---|
| `type` | String | 链接类型 | `url` / `pdf` / `image` |
| `preview` | String? | 缩略图路径（仅用于前端判定是否展示） | `null`（待生成/队列中）/ 路径字符串（成功）/ `"unavailable"`（失败或不支持） |
| `image` | String? | 截图路径 | 同上 |
| `pdf` | String? | PDF 路径 | 同上 |
| `monolith` | String? | HTML 归档路径 | 同上 |
| `readable` | String? | Readability JSON 路径 | 同上 |
| `metaDescription` | String? | 页面 meta description（截取前 500 字符） | — |
| `lastPreserved` | DateTime? | 最近一次归档时间戳 | `null`（待处理，会被 worker 拾取的必要条件之一）/ 时间戳（已处理完成，无论成败） |
| `indexVersion` | Int? | 搜索索引版本 | 归档完成后置 null 触发重新索引 |
| `clientSide` | Boolean | 是否为用户手动上传（纯标记，Worker 不读取此字段） | 创建 Link 时默认 false；手动上传文件后置 true；手动重跑（PUT /archive）重置为 false |
| `url` | String? | 原始 URL | 非 null → Worker 可参与调度；null → 用户纯上传文件，无原始链接，Worker 永远不处理 |

### 4.4 两条手动上传路径的全链路对比

系统存在两条手动上传路径，它们对 `url`、`lastPreserved`、`clientSide`、`preview` 四个关键字段的处理存在本质差异，也决定了 Worker 是否会后续接管。

#### 路径一：新建上传（UploadFileModal → useUploadFile）

用户从顶部导航 "Upload File" 入口（`apps/web/components/ModalContent/UploadFileModal.tsx`）上传一个 PDF/PNG/JPG 来创建 Link。前端 Hook `packages/router/links.tsx` L940-L977 实现为**两步串行调用**：

```
Step 1: POST /api/v1/links（创建 Link 记录，不包含文件）
  └─ 后端: apps/web/lib/api/controllers/links/postLink.ts
Step 2: POST /api/v1/archives/{newLinkId}?format={format}（上传真实文件）
  └─ 后端: apps/web/pages/api/v1/archives/[linkId].ts
```

**Step 1 字段状态（postLink.ts）：**
- `url` → **null**（上传文件场景不传 url 参数，L106 `url: link.url?.trim() || null`）
- `type` → 由前端根据文件 MIME 推断（`getLinkTypeFromFormat`）
- `lastPreserved` → **保持 null**（L138 `!shouldPreserveUrl && link.url` 条件：url 为 falsy，条件整体为 false，不会提前写入 unavailable）
- `image/pdf/monolith/readable/preview` → **全部 null**
- `clientSide` → 默认 false

**Step 2 字段状态（archives/[linkId].ts POST）：**
- 真正写文件到 `archives/{collectionId}/{linkId}{suffix}`
- 若上传图片（isImage=true）：L247-L251 调用 `generatePreview`，由其内部决定写 preview 路径/"unavailable"/留 null（异常 catch 分支不兜底，永久 null）
- 若上传 PDF（isPDF=true）：L265 `preview: "unavailable"` 强制写
- L277 `clientSide: true` 标记为手动上传
- `lastPreserved` → **不碰，保持 null**（但 Worker 也不会处理，因为 url=null）

**Worker 是否接管？ ❌ 不会。** Worker 待处理条件（`apps/worker/lib/getLinkBatchFairly.ts` L36-L37）：
```typescript
{ url: { not: null }, lastPreserved: null }
```
新建上传的 Link `url=null`，被永久排除在 Worker 调度范围之外。

#### 路径二：已有链接上传（PreservedFormatRow → POST archives/[linkId]）

用户在 Link 详情页手动上传归档文件来替换 Worker 自动生成的内容（或填补 Worker 未生成的格式）。只有一个 API 调用：`POST /api/v1/archives/{linkId}?format={format}&preview={true|false}`（同一路由 `apps/web/pages/api/v1/archives/[linkId].ts` POST 分支）。

**字段状态：**
- `url` → 非 null（已有 Link，Worker 之前处理过或正待处理）
- 上传图片 → `generatePreview` 同上
- 上传 PDF → `preview: "unavailable"` 同上
- `clientSide: true`（会覆盖之前的 false）
- `lastPreserved` → 不碰，保持原值
- `image/pdf/monolith` → 对应格式写入路径

**Worker 是否接管？ ⚠️ 视 `lastPreserved` 和 `url` 而定。**
- 如果上传前该 Link `lastPreserved` 为时间戳（已处理过）→ Worker 不会再拾取
- 如果上传前该 Link `lastPreserved` 为 null（仍在队列中）→ Worker 仍会调度并**覆盖**用户手动上传的内容（因为 archiveHandler 中 L122 `if (linkType === "image" && !link.image)` 或 L125 `if (linkType === "pdf" && !link.pdf)` 条件可能已被用户上传满足为 false，但 HTML 类型仍会继续处理）。这是一个潜在的冲突：用户上传后如果 Worker 尚未完成，则自动归档会再次写入同名文件覆盖用户上传。

#### 两条路径字段对比表

| 字段 | 新建上传（Step1+Step2） | 已有链接上传 |
|---|---|---|
| `url` | null | 非 null（保留原值） |
| `lastPreserved` | Step1: null → Step2: 不变（仍 null） | 不碰，保持原值 |
| `clientSide` | Step1: false → Step2: true | 直接写 true |
| `preview`（图片） | generatePreview 内部决定（路径/"unavailable"/永久 null） | 同上 |
| `preview`（PDF） | 强制 "unavailable" | 强制 "unavailable" |
| Worker 会否接管 | ❌ url=null 永远不调度 | ⚠️ 取决于 lastPreserved：null=可能接管覆盖；时间戳=不再接管 |

---

## 五、preview 状态机深度分析

preview 字段存在 **三态**：`null`（队列中/待生成）、路径字符串（成功）、`"unavailable"`（失败或不支持）。不同入链路径的回写规则存在精细差异，尤其是 `generatePreview` 的异常分支与超限分支、`allBroken` 对 preview 的特殊处理。

### 5.1 generatePreview：异常与超限的不对称回写

`packages/lib/generatePreview.ts` 内部分三条退出路径，只有其中一条会主动写 DB：

| 退出路径 | 触发条件 | DB 行为 | 返回值 | preview 最终状态依赖 |
|---|---|---|---|---|
| **入参不合法** | `buffer`/`collectionId`/`linkId` 任一为 falsy（L10、L54） | 不写 DB | `false` | 若在 worker 流程 → finally 收敛为 `"unavailable"`；若在 API 上传流程 → **永久 `null`** |
| **超限（Buffer size exceeded）** | 压缩后 Jimp Buffer > `PREVIEW_MAX_BUFFER`（默认 10MB，L22-L34） | **显式写 `preview: "unavailable"`** | `false` | 已落定为 `"unavailable"` |
| **成功** | 写入文件并写路径（L36-L48） | 写 `preview: 路径字符串` | `true` | 已落定为路径 |
| **异常（catch 块）** | `Jimp.read` 解码失败、`getBufferAsync` 抛错等（L49-L51） | **不写 DB**，只打 `console.error` | `false` | 若在 worker 流程 → finally 收敛为 `"unavailable"`；若在 API 上传流程 → **永久 `null`** |

关键差异：**超限是唯一主动回写 "unavailable" 的失败路径**；解码异常等只吞掉错误、留 null 给上层收敛。这意味着如果手动上传图片时 Jimp 解码失败，preview 会永久停留在 null，前端一直显示 loading spinner（因为 API 上传无 finally 兜底）。

### 5.2 handleArchivePreview（HTML 页面归档）：og:image 主路径与 Playwright fallback 的级联

`apps/worker/lib/preservationScheme/handleArchivePreview.ts` 的执行是一个两级瀑布流：

```
尝试 og:image 路径
     │
     ├─ og:image 不存在 → 直接进入 fallback
     │
     ├─ og:image URL 不安全（SSRF 拦截 UnsafeUrlError）→ 错误被吞，previewGenerated=false → 进入 fallback
     │
     ├─ og:image 其他异常（网络错误等）→ 向上抛出，走 archiveHandler catch → finally 收敛
     │
     └─ og:image 成功加载 → 调用 generatePreview(buffer, ...)
             │
             ├─ generatePreview 成功 → previewGenerated=true → ✅ 跳过 fallback
             │
             ├─ generatePreview 超限 → DB 已写 "unavailable"，previewGenerated=false
             │     └─ 进入 fallback（条件：!previewGenerated && !link.preview?.startsWith("archive")
             │           "unavailable" 确实不以 "archive" 开头 ⇒ 会覆盖写新的截图
             │
             └─ generatePreview 异常（catch）→ DB 仍为 null，previewGenerated=false
                   └─ 进入 fallback ⇒ 会覆盖写新的截图

fallback 路径（Playwright 直接截图）
     │
     ├─ screenshot Buffer > PREVIEW_MAX_BUFFER（L63-L67）→ 只打 console.log，不写文件不写 DB
     │     └─ preview 留 null → finally 收敛为 "unavailable"
     │
     └─ screenshot 成功 → createFile + 写 preview 路径 ⇒ 最终成功
```

关键细节（L42、L59）：两个分支都有前置守卫 `!link.preview?.startsWith("archive")`——如果 preview 已经是有效路径（如其他 handler 已写入），则跳过。但 `"unavailable"` **不**以 "archive" 开头，所以 og:image 超限写了 "unavailable" 后，fallback 仍会执行并可能**覆盖**为成功路径。

### 5.3 手动上传图片 vs PDF：显式不对称

`apps/web/pages/api/v1/archives/[linkId].ts` POST 分支 L241-L279：

| 上传类型 | preview 处理 | 潜在风险 |
|---|---|---|
| **图片（isImage=true）** | L247-L251 调用 `generatePreview(fileBuffer, ...)`，由其内部决定写路径/"unavailable"/留 null；L265 `preview: undefined`（不做二次写） | generatePreview 异常 catch 分支不写 DB，且上传 API **无 finally 兜底**，preview 永久为 null |
| **PDF（isPDF=true）** | L265 `preview: "unavailable"`，无论 PDF 是否带封面图一律强制 unavailable | 不支持 PDF 封面提取 |
| **HTML（isHTML=true）** | preview 无处理（undefined），保持原值 | — |

### 5.4 worker finally：通用收敛兜底

```typescript
// apps/worker/lib/archiveHandler.ts L213-L224
await prisma.link.update({
  where: { id: link.id },
  data: {
    lastPreserved: new Date().toISOString(),
    preview:    !finalLink.preview    ? "unavailable" : undefined,
    readable:   !finalLink.readable   ? "unavailable" : undefined,
    image:      !finalLink.image      ? "unavailable" : undefined,
    monolith:   !finalLink.monolith   ? "unavailable" : undefined,
    pdf:        !finalLink.pdf        ? "unavailable" : undefined,
    indexVersion: null,
  },
});
```

- `undefined` 表示不修改（保留已写入的路径或 "unavailable"）
- `!finalLink.preview` 为 true（仍为 null 或空字符串）时写入 `"unavailable"`
- **覆盖所有 worker 归档路径**：原生 image/pdf、HTML 页面、Monolith、Readability 统一在此收口

此外还有两处**提前写 unavailable**（不进入 try 块就直接返回）：
- 创建 Link 时 URL 不安全（SSRF）：`apps/web/lib/api/controllers/links/postLink.ts` L138-L148 一次性写全字段 unavailable
- archiveHandler 入口 skipPreservation（全局禁用 / 非 http(s) / SSRF）：`apps/worker/lib/archiveHandler.ts` L44-L61

### 5.5 allBroken 重跑：preview 是"二等公民"

`apps/web/pages/api/v1/worker/preservation.tsx` 中 allBroken 对 preview 的处理存在三层不对称：

**1. 查询阶段（L77-L83）**：preview = "unavailable" **会**让链接被纳入 `brokenArchives` 结果集
```
OR: [
  { image: "unavailable" }, { pdf: "unavailable" },
  { readable: "unavailable" }, { monolith: "unavailable" },
  { preview: "unavailable" }   ← preview 参与筛选
]
```

**2. needsReprocessing 判定（L127-L132）**：preview **完全不参与**
```typescript
const needsReprocessing =
  (link.image === "unavailable" && shouldArchive.archiveAsScreenshot) ||
  (link.monolith === "unavailable" && shouldArchive.archiveAsMonolith) ||
  (link.pdf === "unavailable" && shouldArchive.archiveAsPDF) ||
  (link.readable === "unavailable" && shouldArchive.archiveAsReadable);
// preview 没有对应 shouldArchive 开关，不参与判定
```

**3. update 数据（L135-L159）**：preview 字段**完全缺席**，不会被重置为 null

综合效果：
- **只有 preview 坏了、其他格式都好**：链接被查询出来，但 `needsReprocessing=false` → 不重置 lastPreserved → worker **不会**重跑，preview 永远 unavailable
- **至少有一个主格式（image/pdf/readable/monolith）也坏了**：needsReprocessing=true → lastPreserved=null → worker 重跑 → `handleArchivePreview` 会再次尝试生成 preview（但 preview 本身不会被重置为 null，前置守卫 `!link.preview?.startsWith("archive")` 会判断 "unavailable" 确实不以 "archive" 开头 → 允许重新生成并覆盖）

### 5.6 preview 完整状态转移图

```
          创建 Link（URL 安全）
                │
                ▼
           [ null ] ◄────────────── PUT /archive（重跑）/ allAndRePreserve / 修改 URL
                │                        /
                │                       /
     ┌──────────┼──────────────────────/───────────────┐
     │          │                                     │
     ▼          ▼                                     ▼
generatePreview  handleArchivePreview            手动上传图片
  超限分支     og:image 超限/成功/fallback           generatePreview
  写"unavailable"  各自写路径或"unavailable"          超限→"unavailable"
                │                                     异常→永久 null
                ▼                                     │
         [ 路径字符串 ] ◄─────────────────────────────┘
                │
                │ skipPreservation / 创建URL不安全 / finally 收敛
                │ / 手动上传 PDF / allBroken 不重置 preview
                ▼
         [ "unavailable" ]
```

### 5.7 前端对三态的渲染

判定函数 `formatAvailable` 位于 `packages/lib/formatStats.ts`：

```typescript
export function formatAvailable(link, format) {
  return Boolean(link && link[format] && link[format] !== "unavailable");
}
```

前端组件 `apps/web/components/Preservation/PreservationContent.tsx` L256-L277 的分支逻辑：

```
formatAvailable(link, "preview") === true   → 渲染真实内容（<img>）
link.preview === "unavailable"               → 渲染 404 "Format not available" 占位
其他（即 null）                               → 渲染 BeatLoader "preservation_in_queue"
```

跳转兜底：`packages/lib/getFormatBasedOnPreference.ts` 中如果目标格式值为 null 或 `"unavailable"`，返回 null，前端回退到原始 URL。

### 5.8 前端缩略图展示路径的关键纠正：DB 字段 ≠ 文件路径

**一个极易混淆的设计：`link.preview` 字段存储的值**只用于**前端三态判定**（是否展示 <Image> 组件），**不用于实际文件读取的路径拼接**。真正读取文件时走的是**约定路径**而非 DB 存储值。

**LinkCard 缩略图展示链路**（`apps/web/components/LinkViews/LinkComponents/LinkCard.tsx` L101-L124）：

```tsx
{formatAvailable(link, "preview") ? (
  <Image
    src={`/api/v1/archives/${link.id}?format=${ArchivedFormat.jpeg}&preview=true&updatedAt=${link.updatedAt}`}
    ...
  />
) : link.preview === "unavailable" ? (
  <div className="bg-gray-50 ..."></div>  // 灰色占位
) : (
  <div className="skeleton ..."></div>      // loading 骨架
)}
```

**请求实际文件的后端路由**（`apps/web/pages/api/v1/archives/[linkId].ts` L85-L128 `handleGet`）：
1. 解析 query：`linkId`、`format`、`preview=true|false`
2. 调 `resolveAccessibleArchive` 做权限校验并拼接物理路径

**路径拼接逻辑**（`apps/web/lib/api/archives/resolveAccessibleArchive.ts` L70-L72）：
```typescript
const filePath = isPreview
  ? `archives/preview/${collection.id}/${linkId}.jpeg`
  : `archives/${collection.id}/${linkId + suffix}`;
```

核心结论：
- **preview 文件路径完全由约定生成**：`archives/preview/{collectionId}/{linkId}.jpeg`，固定为 jpeg 格式，不依赖 DB 中 `link.preview` 存储的字符串
- **非 preview 文件路径同理**：`archives/{collectionId}/{linkId}{suffix}`，suffix 由 format 参数查表得到（`apps/web/lib/shared/getSuffixFromFormat.ts`）
- `link.preview` DB 字段**唯一作用**是供 `formatAvailable(link, "preview")` 判定——值是否存在且 ≠ "unavailable"
- 副作用：如果 someone 手动把 `link.preview` 改成了一个随意字符串但磁盘上文件不存在，`formatAvailable` 会判定为 true，前端会发起请求，后端 `readFile` 会返回 404，此时 `<Image>` 的 `onError` 会隐藏该元素（LinkCard L110-L113）

同理，`image`/`pdf`/`monolith`/`readable` 等其他归档字段也只用于判定存在性，实际访问路径由 `format` 和 `linkId` 约定拼接。

### 5.9 现有缺陷汇总

| 缺陷 | 影响 | 触发场景 |
|---|---|---|
| 手动上传图片 generatePreview 异常不写 DB | preview 永久 null，前端一直 loading | 上传的图片 Jimp 无法解码（损坏/格式不支持） |
| allBroken 不处理 preview 单独失败 | preview 永远 unavailable，除非手动单链接重跑 | 缩略图生成挂了但截图/PDF 都好 |
| og:image 超限后 fallback 可能做无用功 | generatePreview 已写 "unavailable"，fallback 仍再截图一次 | og:image 是超大图且压缩后仍超限 |
| finally 收敛无法区分失败原因 | 用户看不到是"解码失败"还是"文件超限"还是"网络错误" | 所有 worker 失败路径 |
| 已有链接上传与 Worker 并发冲突 | 用户上传的文件可能被尚未完成的 Worker 归档覆盖 | 用户在 Worker 处理中的链接上手动上传归档 |

---

## 六、手动触发重跑归档的 API 路径

共三个层级的手动触发入口，均以**重置字段为 null + 删除旧文件**的方式将链接重新投入队列。

### 6.1 单链接重跑：`PUT /api/v1/links/[id]/archive`

文件：`apps/web/pages/api/v1/links/[id]/archive/index.ts`

权限要求：collection owner 或具有 `canUpdate` 权限的 member。

执行动作：
1. 校验 link.url 存在且为合法 URL
2. DB 重置（L51-L64）：
   ```
   image/pdf/readable/monolith/preview → null
   lastPreserved → null
   indexVersion → null
   clientSide → false
   ```
3. 调用 `removeFiles(link.id, collectionId)` 删除磁盘/S3 上所有旧归档文件
4. 返回 `"Link is being archived."`，worker 下一轮轮询会拾取该链接（`lastPreserved = null`）

### 6.2 批量删除归档（可用于批量重跑）：`DELETE /api/v1/links/archive`

文件：`apps/web/pages/api/v1/links/archive/index.ts`

权限要求：请求体中 `linkIds` 列表里的每个链接，用户必须是 owner 或有 `canDelete` 权限。

请求体 schema：`LinkArchiveActionSchema`，包含 `linkIds: number[]`。

执行动作（对每个授权链接）：
1. `removeFiles(link.id, collectionId)` 删除所有旧文件
2. DB 重置：`image/pdf/readable/monolith/preview → null`、`lastPreserved → null`、`indexVersion → null`

注意：这个接口先返回 `200 Success` 再执行删除循环（异步），客户端无法直接感知单个链接的失败。

### 6.3 管理员级批量操作：`DELETE /api/v1/worker/preservation`

文件：`apps/web/pages/api/v1/worker/preservation.tsx`

权限要求：`user.id === Number(process.env.NEXT_PUBLIC_ADMIN || 1)`，仅服务端管理员。

支持两种 action：

#### action = `allAndRePreserve`
对当前用户所有 `type = "url"` 且有 URL 的链接：
- `removeFiles` 全部删除
- `image/pdf/readable/monolith/preview → null`、`lastPreserved → null`、`indexVersion → null`

#### action = `allBroken`（精细重跑失败项）
筛选条件：任意归档字段等于 `"unavailable"` 的链接（含 `preview`）。

处理逻辑：
1. 聚合用户级或 Tag 级的 `archivalSettings`（与 archiveHandler 中一致的算法）
2. 判定 `needsReprocessing`：只看 4 个主格式 `shouldArchive.xxx === true` 且 `link.xxx === "unavailable"`，**preview 不参与判定**
3. 若 `needsReprocessing=true`：重置对应主格式为 null、`lastPreserved → null`、`indexVersion → null`；**preview 字段完全不碰**
4. 若 `needsReprocessing=false`（如只有 preview = "unavailable"）：**什么都不做**，链接不会被重跑

preview 的间接重跑路径：只有当某个主格式（image/pdf/readable/monolith）同时失败触发 needsReprocessing=true 时，`lastPreserved` 才会被置 null → worker 重跑 → `handleArchivePreview` 的前置守卫 `!link.preview?.startsWith("archive")` 对 `"unavailable"` 返回 true → 允许重新生成并覆盖 preview。如果只有 preview 坏了，必须走单链接手动重跑（PUT /api/v1/links/[id]/archive）。

示例：用户只开启了 PDF 归档而截图失败（可能只是未开启），那么 `allBroken` 不会误重置截图字段。

---

## 七、字段重置规则汇总

所有会触发归档字段重置的场景及重置范围：

| 触发场景 | image | pdf | readable | monolith | preview | lastPreserved | indexVersion | clientSide | 删除旧文件 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 单链接手动重跑（PUT /archive） | ✅ null | ✅ null | ✅ null | ✅ null | ✅ null | ✅ null | ✅ null | ✅ false | ✅ 全部格式 |
| 批量删除归档（DELETE /links/archive） | ✅ null | ✅ null | ✅ null | ✅ null | ✅ null | ✅ null | ✅ null | — | ✅ 全部格式 |
| 管理员 allAndRePreserve | ✅ null | ✅ null | ✅ null | ✅ null | ✅ null | ✅ null | ✅ null | — | ✅ 全部格式 |
| 管理员 allBroken | ⚠️ 条件A | ⚠️ 条件A | ⚠️ 条件A | ⚠️ 条件A | ⚠️ 条件B | ⚠️ 条件C | ⚠️ 条件C | — | ❌ |
| 修改链接 URL（updateLinkById） | ✅ null | ✅ null | ✅ null | ✅ null | ✅ null | ✅ null | ✅ null | — | ✅ 全部格式 |
| 归档 finally 失败收敛 | ❌ → unavailable | ❌ → unavailable | ❌ → unavailable | ❌ → unavailable | ❌ → unavailable | ✅ 写入时间 | ✅ null | — | ❌ |
| 创建链接时 URL 不安全 | ❌ → unavailable | ❌ → unavailable | ❌ → unavailable | ❌ → unavailable | ❌ → unavailable | ✅ 写入时间 | ✅ null | — | ❌ |
| 移动链接到其他 collection | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | — | ❌（改用 moveFiles）|

**条件说明：**
- **条件A**：只有该格式在用户/Tag 配置中被启用（`shouldArchive.xxx === true`）且当前值为 `"unavailable"` 时才重置为 null。
- **条件B（preview 特有）**：update 数据中 preview 字段**完全缺席**，不会被直接重置。但如果条件C为 true（lastPreserved 被置 null），worker 重跑时 `handleArchivePreview` 的前置守卫 `!link.preview?.startsWith("archive")` 对 `"unavailable"` 返回 true，允许再次生成 preview 并覆盖原值。
- **条件C**：`needsReprocessing` 为 true 时才重置为 null（即至少有一个主格式满足条件A）。若**仅 preview = "unavailable"** 而其他主格式都正常，needsReprocessing=false，lastPreserved 不动，worker 不会重跑，preview 永久停留在 unavailable。

修改 URL 的重置逻辑位于 `apps/web/lib/api/controllers/links/linkId/updateLinkById.ts` L133-L163；移动 collection 的文件搬迁位于同文件 L195-L197（调用 `moveFiles`，`packages/filesystem/manageFiles.ts`）。

---

## 八、下载权限体系

归档文件不直接通过静态文件服务暴露，而是经过两层权限校验：

### 8.1 层级一：直接 API 访问（`GET /api/v1/archives/[linkId]`）

文件：`apps/web/pages/api/v1/archives/[linkId].ts`

访问流程：
1. `verifyToken`：可选 Bearer Token，解析出 userId（未登录则 userId = undefined）
2. `resolveAccessibleArchive`：查询 Collection 权限，条件为
   - 是 collection owner，**或**
   - 是 collection member，**或**
   - collection 是公开的（`isPublic: true`）
3. 权限通过后拼接物理路径 → `readFile` 读取 → 设置 `Cache-Control: private, max-age=31536000, immutable` 返回

Monolith 格式有额外限制：如果配置了 `NEXT_PUBLIC_USER_CONTENT_DOMAIN`，必须通过带 token 的用户内容域名访问（防止同源 XSS）。

权限解析核心：`apps/web/lib/api/archives/resolveAccessibleArchive.ts`，`apps/web/lib/api/getPermission.ts`

### 8.2 层级二：用户内容域名 + JWT Token（`GET /api/v1/preserved/view`）

文件：`apps/web/pages/api/v1/preserved/view.ts`，`apps/web/lib/api/preserved/createPreservedFormatUrl.ts`

适用于 Monolith HTML 在独立域名下展示（避免 XSS 窃取主站 Cookie）：

1. 前端调用 `createPreservedFormatUrl`，服务端用 `NEXTAUTH_SECRET` 签发一个 **JWT Token**（TTL = 300 秒）
2. Token 中编码：`linkId`、`filePath`、`format`、`scope="preserved-format"`
3. 请求通过独立 `NEXT_PUBLIC_USER_CONTENT_DOMAIN` 进入，服务端：
   - 校验 Host 必须匹配用户内容域名
   - 解码并验证 JWT（scope、exp、格式正确性）
   - 读取文件返回

下载模式：URL 参数 `?download=1` 会加上 `Content-Disposition: attachment; filename="..."` 触发浏览器下载。

### 8.3 上传权限（`POST /api/v1/archives/[linkId]`）

用户手动上传归档文件时（clientSide 模式）：
- `verifyUser` 强校验登录态
- `getPermission` + `userHasCreatePermission`：要求是 **owner 或有 canCreate 权限的 member**
- 校验用户链接数量限制（`MAX_LINKS_PER_USER`，默认 30000）
- 上传文件大小限制：`NEXT_PUBLIC_MAX_FILE_BUFFER`（默认 10MB）
- 允许的 MIME：`application/pdf`、`image/png`、`image/jpg`、`image/jpeg`、`text/html`

前端获取下载/查看 URL 的工具函数：`packages/lib/getPreservedFormatUrl.ts`

---

## 九、任务状态协同机制

### 9.1 任务调度：公平轮询（getLinkBatchFairly）

路径：`apps/worker/lib/getLinkBatchFairly.ts`

核心是**用户级公平**，避免单个用户的海量链接阻塞其他用户：

1. 筛选有资格的用户：有未处理链接 + 订阅有效/试用期内 + 邮箱已验证
2. 用户按 `lastPickedAt`（上次被调度时间）升序排列，优先照顾久未处理的用户
3. 每轮从每个用户轮流取 `linksPerUser = floor(maxBatchLinks / users.length)` 条链接
4. 取完后更新这些用户的 `lastPickedAt = now`
5. 批次大小由 `ARCHIVE_TAKE_COUNT` 控制（默认 5 条）

**待处理链接的判定条件（L36-L37 baseLinkWhere）**：
```typescript
{ url: { not: null }, lastPreserved: null }
```

关键含义：
- **`url != null` 是硬门槛**：用户通过 UploadFileModal 新建上传的 Link（url=null）永远不会被 Worker 调度，完全依赖客户端上传完成。这也是新建上传路径与已有链接上传路径的本质分界线。
- **`clientSide` 字段完全不参与 Worker 调度判定**：在 `apps/worker/` 整个目录下搜索不到任何 `clientSide` 引用，该字段只是前端/API 层的一个纯标记，用于区分"自动归档生成"与"用户手动上传"，不影响调度。
- 已有链接上手动上传（clientSide 被置为 true）后，如果 `lastPreserved` 仍为 null 且 `url != null`，Worker **仍会调度**，可能覆盖用户上传的文件。
- `type` 字段也不参与调度筛选，调度后由 `determineLinkType` 和 `linkType` 再做分流。

### 9.2 并发与浏览器生命周期

路径：`apps/worker/workers/linkProcessing.ts`

- 单例 Playwright Browser **每 30 分钟强制重启**（防止内存泄漏/僵尸进程）
- 批次内的多条链接通过 `Promise.allSettled` **并发**调用 `archiveHandler`
- 每个链接内创建独立的 BrowserContext（隔离状态）
- 单链接浏览器超时：`BROWSER_TIMEOUT`（默认 5 分钟），通过 `AbortController` + `Promise.race` 实现
- 失败时检查 `browser.isConnected()`，如已断开则立即重启

### 9.3 状态收尾（finally 块）

无论归档成功或失败，`apps/worker/lib/archiveHandler.ts` L203-L229 的 `finally` 块统一执行：

1. 清理 timeout
2. **重新从 DB 读取 Link**（确认用户没有在归档过程中删除链接）
3. 如果 Link 仍存在：
   - 写入 `lastPreserved = now`（标记"已处理过一次"，脱离待处理队列）
   - 对所有仍为 null 的归档字段标记为 `"unavailable"`
   - `indexVersion = null` 触发重新索引
4. 如果 Link 已被删除：调用 `removeFiles` 清理所有已生成的归档文件
5. 关闭 BrowserContext

关键设计：**先写文件，再写 DB**，如果 DB 字段仍为 null 说明文件未成功生成，统一收敛为 "unavailable"。

---

## 十、大文件失败后的清理机制

### 10.1 提前终止：Buffer 大小检查

在写入文件**之前**检查 Buffer 大小，超过限制直接 `return console.log(...)`，不落盘、不更新 DB：

| 场景 | 限制变量 | 默认值 |
|---|---|---|
| 截图（含原生图片） | `SCREENSHOT_MAX_BUFFER` | 100 MB |
| PDF（含原生 PDF） | `PDF_MAX_BUFFER` | 100 MB |
| 预览缩略图 | `PREVIEW_MAX_BUFFER` | 10 MB |
| 用户手动上传 | `NEXT_PUBLIC_MAX_FILE_BUFFER` | 10 MB |

代码位置：
- `apps/worker/lib/preservationScheme/imageHandler.ts` L10-L14
- `apps/worker/lib/preservationScheme/pdfHandler.ts` L9-L13
- `apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts` L29-L35 / L68-L74
- `packages/lib/generatePreview.ts` L22-L34

### 10.2 部分成功的处理

由于截图和 PDF 使用 `Promise.allSettled` 并行执行，可能出现**截图成功但 PDF 超限**的情况——此时成功的那个已落盘、已写入 DB；失败的那个仅打日志，DB 字段保持 null，随后在 finally 块被统一标记为 `"unavailable"`。

### 10.3 链接中途删除的兜底

archiveHandler finally 块中会重新查询 Link：
```typescript
const finalLink = await prisma.link.findUnique({ where: { id: link.id } });
if (finalLink) {
  // 正常收尾
} else {
  await removeFiles(link.id, link.collectionId); // 清理所有格式
}
```

### 10.4 主动删除的清理

- 删除单个 Link：`apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts` → 调用 `removeFiles(linkId, collectionId)`
- `removeFiles` 覆盖 7 种格式变体（.pdf/.png/.jpeg/.jpg/.html/preview .jpeg/_readability.json）
- 注意：`removeFile` 使用**异步非阻塞**的 `fs.unlink`（回调式），出错只打日志不抛出，避免清理失败影响主流程

### 10.5 现有问题

1. **超限失败时无精细 DB 标记**：超限只打 `console.log`，DB 字段为 null 会在 finally 被标记为 "unavailable"，但用户无法区分是"文件超限"还是"页面无法访问"。
2. **已写入的部分文件无回滚**：`createFile` 成功但后续 `prisma.link.update` 失败时，磁盘上会遗留孤儿文件，没有清理机制。
3. **上传临时文件清理**：API 上传使用 `fs.unlinkSync` 同步清理，但异常路径（如 throw 之后）可能残留 formidable 临时文件。

---

## 十一、缩略图对性能的影响

### 11.1 缩略图生成链路

路径：`packages/lib/generatePreview.ts`

使用 **Jimp**（纯 JS 图像处理库，无原生依赖）：
1. `Jimp.read(buffer)` 解码图片
2. `resize(1000, Jimp.AUTO)` 等比缩放至 1000px 宽
3. `quality(20)` 输出低质量 JPEG
4. Buffer 二次大小检查（PREVIEW_MAX_BUFFER，默认 10MB）

失败时直接将 `preview` 写为 `"unavailable"`，不影响其他归档格式。

### 11.2 缩略图触发时机

| 场景 | 是否生成预览 |
|---|---|
| 原生图片链接（imageHandler） | ✅ 生成 |
| 用户上传图片（POST /api/v1/archives/[linkId]） | ✅ 生成 |
| HTML 页面归档（handleArchivePreview） | ⚠️ 优先用 `og:image`；失败则用 Playwright `screenshot({ quality: 20 })` 拍一张 10MB 内的低质量图 |
| 原生 PDF 链接（pdfHandler） | ❌ 不生成 |
| 用户上传 PDF | ❌ 标记 preview = "unavailable" |

HTML 预览逻辑位于 `apps/worker/lib/preservationScheme/handleArchivePreview.ts`。

### 11.3 性能影响点

1. **CPU 密集**：Jimp 在主线程做图片解码/缩放/编码，会阻塞 Node.js 事件循环。对大批量图片归档，worker 的处理能力会被图片处理而非网络 IO 瓶颈化。
2. **双份 Buffer**：`imageHandler` 中同时持有原图 Buffer + 缩略图处理 Buffer，内存峰值为原图的 ~2 倍。
3. **缩略图独立 IO**：每个图片链接触发 2 次 `createFile`（原图 + 缩略图）+ 2 次 DB update，翻倍存储操作次数。
4. **og:image 额外网络请求**：`apps/worker/lib/preservationScheme/handleArchivePreview.ts` 会对 og:image URL 做 SSRF 校验后导航过去取图，再 `goBack`，增加一次完整的页面往返。
5. **质量参数**：统一 `quality: 20` 是激进的压缩，文件体积小但视觉质量可能不足。

### 11.4 优化方向（现有代码中未实现）

- 将 Jimp 处理移到 Worker Threads，避免阻塞事件循环
- 对超大原图先流式降采样再解码
- 缩略图生成可配置开关或延迟生成（懒加载）
- 考虑使用原生库（sharp）替代 Jimp 以获得 5~10x 性能提升
