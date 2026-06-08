# Linkwarden 链接元数据富集（Metadata Enrichment）规则详解

本文档基于逐行代码阅读，系统梳理 Linkwarden 在链接创建、后台处理、缓存失效和多语言场景下的元数据抓取与更新规则。每一条结论都附带可复核的代码证据链接。

---

## 一、元数据字段总览

每个 Link 记录在数据库中包含以下与元数据相关的字段（定义见 [schema.prisma#L166-L198](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/packages/prisma/schema.prisma#L166-L198)）：

| 字段 | 类型 | 来源 | 说明 |
|---|---|---|---|
| `name` | String | 用户输入 / 自动抓取 | 链接显示标题 |
| `description` | String | 用户输入 | 用户手工填写的描述，UI 直接展示 |
| `metaDescription` | String? | 自动抓取（Worker） | 页面 `<meta name="description">` 内容，用于 AI 打标签，**不直接在 UI 显示** |
| `preview` | String? | 自动生成 | 预览图文件路径，或 `"unavailable"` |
| `image` | String? | 用户上传 / 自动截图 | 完整截图文件路径，或 `"unavailable"` |
| `icon/iconWeight/color` | String? | 用户输入 | 用户自定义图标及样式 |
| `type` | String | 自动检测 | `url` / `pdf` / `image` |
| `lastPreserved` | DateTime? | 系统维护 | 上次归档处理时间戳，`null` 表示待处理 |
| `indexVersion` | Int? | 系统维护 | 搜索索引版本号，`null` 表示待重建索引 |

**"unavailable" 语义**：字段值为字符串 `"unavailable"` 时表示该归档项已尝试但失败。判定函数见 [formatStats.ts#L4-L9](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/packages/lib/formatStats.ts#L4-L9)：
```js
return Boolean(link && link[format] && link[format] !== "unavailable");
```
即：字段为 truthy 且不等于 `"unavailable"` 时才算可用。

---

## 二、标题（name）抓取与优先级

### 2.1 创建时的标题决策链

在 [postLink.ts#L78-L88](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/api/controllers/links/postLink.ts#L78-L88) 中，标题按以下优先级确定：

```
1. 用户提供的 link.name（非空字符串时）
   ↓
2. 若 URL 安全可抓取，则从页面 <title> 标签中提取
   （通过 fetchTitleAndHeaders.ts，正则 /<title.*>([^<]*)<\/title>/）
   ↓
3. 回退为 URL 本身
   ↓
4. 完全无 URL 时为空字符串
```

### 2.2 标题抓取实现细节

核心逻辑在 [fetchTitleAndHeaders.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/shared/fetchTitleAndHeaders.ts)：

- **超时保护**：10 秒超时（`Promise.race` + `setTimeout`），见 [fetchTitleAndHeaders.ts#L12-L16](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/shared/fetchTitleAndHeaders.ts#L12-L16)
- **协议限制**：仅处理 `http://` 或 `https://` 开头的 URL，见 [fetchTitleAndHeaders.ts#L7-L8](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/shared/fetchTitleAndHeaders.ts#L7-L8)
- **容错**：任何异常（网络错误、解析失败等）均返回 `{ title: "", headers: null }`，不中断创建流程，见 [fetchTitleAndHeaders.ts#L40-L43](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/shared/fetchTitleAndHeaders.ts#L40-L43)
- **内容注入**：允许直接传入 `content` 参数从 HTML 字符串中提取（用于客户端上传 HTML 文件的场景），见 [fetchTitleAndHeaders.ts#L23-L27](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/shared/fetchTitleAndHeaders.ts#L23-L27)

### 2.3 Worker 是否覆盖标题？

**不会。** Worker 的 [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts) 只更新：`metaDescription`、`preview`、`image`、`pdf`、`readable`、`monolith`、`lastPreserved`、`indexVersion`。`name` 字段在创建后只由用户编辑（见 [updateLinkById.ts#L151](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L151)）。

---

## 三、描述（description / metaDescription）双字段机制

Linkwarden 采用**两个独立描述字段**的设计，用途截然不同：

### 3.1 `description`（用户描述）

- **写入来源**：仅用户手工输入（创建时 `link.description`、编辑时 `data.description`），见 [postLink.ts#L108](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/api/controllers/links/postLink.ts#L108) 和 [updateLinkById.ts#L153](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L153)
- **显示位置**：
  - Masonry 卡片视图：[LinkMasonry.tsx#L147-L151](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkMasonry.tsx#L147-L151) — 仅当 `show.description && link.description` 时显示
  - RSS 导出：`link.description`
  - 阅读器视图标题 fallback：`link.name || link.description || link.url`，见 [ReadableView.tsx#L441](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/components/Preservation/ReadableView.tsx#L441)
- **Worker 永不覆盖**

### 3.2 `metaDescription`（页面元描述）

- **写入来源**：仅 Worker 后台抓取，在 [archiveHandler.ts#L151-L164](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts#L151-L164) 中：
  ```js
  // Playwright 在浏览器上下文中执行
  const description = document.querySelector('meta[name="description"]');
  return description?.getAttribute("content") ?? undefined;
  ```
- **处理规则**：
  - `trim()` 后截取前 **500 字符**
  - 若未找到则不更新（传 `undefined` 给 Prisma，不覆盖原值）
- **用途**：
  - AI 自动打标签时的上下文输入（优先级高于 `textContent`），见 [autoTagLink.ts#L76-L78](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/autoTagLink.ts#L76-L78)：
    ```
    description = (metaDescription ? metaDescription + "..." : undefined)
                  || (textContent ? textContent?.slice(0, 500) + "..." : undefined)
    ```
- **UI 中不显示**：在前端代码中未发现任何直接读取 `link.metaDescription` 用于展示的逻辑。

---

## 四、Favicon 抓取

### 4.1 架构特点

Favicon **不存入数据库**，而是通过 API 按需动态获取。

### 4.2 获取流程

前端 [LinkIcon.tsx#L49-L69](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L49-L69)：

```
用户设置了自定义 icon/iconWeight/color ?
   ├─ 是 → 显示 Phosphor 自定义图标
   └─ 否 → 链接类型为 "url" ?
              ├─ 是 → 请求 /api/v1/getFavicon?url={origin}
              ├─ pdf → 显示 bi-file-earmark-pdf
              └─ image → 显示 bi-file-earmark-image
```

### 4.3 Favicon API：两级缓存与来源 Fallback

实现在 [getFavicon/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/getFavicon/index.ts)：

#### 第一级：URL 规范化重定向缓存

任意 URL 先规范化为 origin 形式，见 [getFavicon/index.ts#L57-L61](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/getFavicon/index.ts#L57-L61)：
```js
const canonical = `/api/v1/getFavicon?url=${encodeURIComponent(origin)}`;
if (req.url !== canonical) {
  res.setHeader("Cache-Control", "public, max-age=3600");  // 1 小时
  return res.redirect(308, canonical);  // 308 = 永久重定向，浏览器会缓存
}
```

#### 第二级：实际 Favicon 响应缓存

规范化后依次尝试以下来源（第一个成功即返回），见 [getFavicon/index.ts#L63-L83](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/getFavicon/index.ts#L63-L83)：

1. **Google Favicon 服务**：
   ```
   https://t2.gstatic.com/faviconV2?client=SOCIAL&type=FAVICON
     &fallback_opts=TYPE,SIZE,URL&url={origin}&size=64
   ```

2. **DuckDuckGo 图标服务**（第 1 个失败时尝试）：
   ```
   https://icons.duckduckgo.com/ip3/{hostname}.ico
   ```

**请求约束**（[getFavicon/index.ts#L8-L31](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/getFavicon/index.ts#L8-L31)）：
- 单次请求超时 **1.5 秒**（`AbortController`）
- 仅接受 `image/*` Content-Type
- 自动跟随重定向

**成功响应缓存头**（[getFavicon/index.ts#L76-L79](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/getFavicon/index.ts#L76-L79)）：
```
Cache-Control: public, max-age=86400, s-maxage=2592000, stale-while-revalidate=604800, immutable
```
- `max-age=86400`：浏览器缓存 **1 天**
- `s-maxage=2592000`：CDN/共享缓存 **30 天**
- `stale-while-revalidate=604800`：过期后 **7 天**内可返回旧值同时后台刷新
- `immutable`：资源不可变，浏览器刷新时也不会重新验证

**失败响应（204 No Content）缓存头**（[getFavicon/index.ts#L86-L89](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/getFavicon/index.ts#L86-L89)）：
```
Cache-Control: public, max-age=3600, s-maxage=86400, stale-while-revalidate=604800
```
- 浏览器缓存 **1 小时**（比成功短，便于尽快重试成功）
- CDN 缓存 **1 天**

**UI Fallback**：Favicon 加载完成前显示占位图标（`bi-link-45deg`），通过 `onLoad` + opacity 切换，见 [LinkIcon.tsx#L31-L69](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L31-L69)。

---

## 五、预览图（preview）生成规则与异常 Fallback（重点校正）

### 5.1 生成入口判断

Worker 的 [archiveHandler.ts#L168-L169](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts#L168-L169)：
```js
// Preview
if (!link.preview) await handleArchivePreview(link, page);
```

**⚠️ 关键细节**：条件是 `!link.preview`（falsy 判断）。字符串 `"unavailable"` 是非空字符串，属于 truthy，因此：
- `preview = null` → 执行 handleArchivePreview ✅
- `preview = "unavailable"` → **不执行** ❌
- `preview = "archives/preview/..."` → 不执行（已有预览图）

这意味着：**一旦 preview 被标记为 `"unavailable"`，Worker 永远不会再自动重试生成预览图**，除非用户/管理员手动重置 preview 为 null。

### 5.2 handleArchivePreview 两阶段 Fallback 详解

实现在 [handleArchivePreview.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts)。

#### 阶段 1：OG Image 提取（[handleArchivePreview.ts#L21-L57](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts#L21-L57)）

```
1. page.evaluate() 查找 <meta property="og:image"> content
   └─ 不存在 → ogImageUrl = null → 跳过此阶段，进入阶段 2
2. 相对路径自动补全为绝对 URL（使用 document.location.origin）
3. SSRF 安全检查（assertUrlIsSafeForServerSideFetch）
   └─ 不安全 → catch 静默跳过（仅当非 UnsafeUrlError 时才重新抛出）
4. Playwright page.goto(ogImageUrl) 获取图像响应
5. 再次安全判断：!link.preview?.startsWith("archive")
   （注意：此处 link.preview 是函数参数快照，非 DB 实时值）
6. 调用 generatePreview(buffer, collectionId, linkId)
```

**generatePreview 内部行为**（[generatePreview.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/packages/lib/generatePreview.ts)）：

| 场景 | 行为 | preview 结果 | 返回值 |
|---|---|---|---|
| buffer / collectionId / linkId 任一无效 | 跳过 | 不变 | `false` |
| Jimp.read 失败或无 image | log 错误 | 不变 | `false` |
| Jimp 处理异常（catch 块） | `console.error` | 不变 | `false` |
| 处理后 buffer > PREVIEW_MAX_BUFFER（默认 10MB） | 见 [generatePreview.ts#L22-L33](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/packages/lib/generatePreview.ts#L22-L33) | **DB 写入 `"unavailable"`** | `false` |
| 一切正常 | resize 1000px × AUTO，JPEG q=20，写入文件 | DB 写入文件路径 | `true` |

**⚠️ 重要**：generatePreview 将 preview 写入 DB 后，JS 变量 `link.preview`（函数参数）仍然是调用时的原值（通常为 null），不会同步更新。

#### 阶段 2：页面截图 Fallback（[handleArchivePreview.ts#L59-L81](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts#L59-L81)）

```js
if (!previewGenerated && !link.preview?.startsWith("archive")) {
  await page
    .screenshot({ type: "jpeg", quality: 20 })
    .then(async (screenshot) => {
      // 大小检查
      if (Buffer.byteLength(screenshot) > 10MB)
        return console.log("Buffer size exceeded");  // 仅 log，不写 DB
      // 写文件 + 更新 DB preview 路径
    });
}
```

进入阶段 2 的条件：
- `!previewGenerated`：阶段 1 未成功生成
- `!link.preview?.startsWith("archive")`：函数参数 `link.preview` 不以 "archive" 开头（通常为 null → `!null?.startsWith(...)` = `true`）

**阶段 2 异常路径分析**：

| 场景 | 行为 | 后续 |
|---|---|---|
| `page.screenshot()` 抛出异常 | `.then()` 不执行 → await 的 Promise rejected → 异常向上抛出到 archiveHandler 的 try/catch | 进入 archiveHandler finally，基于 DB 现状判定 |
| screenshot buffer 超限 | `return console.log(...)`，不写 DB | 如果阶段 1 写入了 `"unavailable"`，保留；否则 preview 仍为 null → finally 写入 `"unavailable"` |
| `createFile` 或 `prisma.update` 抛出 | `.then()` 内 Promise rejected → 向上抛出 | 进入 archiveHandler finally |
| 一切正常 | 写入文件，DB 更新 preview 为有效路径 | 覆盖阶段 1 可能写入的 `"unavailable"` |

### 5.3 archiveHandler finally 块的兜底

无论正常或异常，[archiveHandler.ts#L208-L224](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts#L208-L224) 都会执行：
```js
const finalLink = await prisma.link.findUnique({ where: { id: link.id } });
if (finalLink) {
  await prisma.link.update({
    where: { id: link.id },
    data: {
      lastPreserved: new Date().toISOString(),
      preview: !finalLink.preview ? "unavailable" : undefined,
      // ... 其他字段同样逻辑
    },
  });
}
```

关键判断：`!finalLink.preview` — 从数据库**重新读取**后判定：
- `finalLink.preview = null` → 真值 → 写入 `"unavailable"`
- `finalLink.preview = "unavailable"` → 假值 → 不覆盖（`undefined`）
- `finalLink.preview = "archives/preview/..."` → 假值 → 不覆盖

### 5.4 预览图完整 Fallback 路径图

```
archiveHandler 入口
  │
  ├─ !link.preview?
  │    ├─ 否 (preview="unavailable" 或已有路径) → 跳过，永不重试 ✋
  │    └─ 是 (preview=null)
  │         │
  │         ▼
  │   handleArchivePreview
  │         │
  │         ├─ 阶段 1: og:image
  │         │    ├─ 成功路径: 生成 → DB 写入路径 ✓
  │         │    ├─ 超限: DB 写入 "unavailable" + previewGenerated=false
  │         │    ├─ Jimp 异常: 只 log + previewGenerated=false
  │         │    └─ 无 og:image / SSRF 拦截 / 网络错: previewGenerated=false
  │         │
  │         └─ 阶段 2: 截图 (仅当 previewGenerated=false)
  │              ├─ 成功: 写文件 + DB 写入路径 ✓
  │              ├─ 截图 buffer 超限: 仅 log，不写 DB
  │              └─ screenshot 抛错: 异常上抛
  │
  ▼
finally 块 (一定执行)
  重新读 DB → finalLink.preview
  ├─ null → 写入 "unavailable"
  ├─ "unavailable" → 不变
  └─ 有效路径 → 不变
```

### 5.5 预览图服务端文件缓存

归档文件通过 [[linkId].ts#L122-L127](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/archives/%5BlinkId%5D.ts#L122-L127) 读取时设置：
```
Cache-Control: private, max-age=31536000, immutable
```
即 **1 年私有缓存，标记 immutable**。URL 中附带 `&updatedAt={link.updatedAt}` 作为版本戳来触发浏览器刷新。

---

## 六、用户手工字段优先级总结

以下字段**永不被 Worker 覆盖**，完全由用户控制：

| 字段 | 优先级 |
|---|---|
| `name` | 用户输入 > 自动抓取标题 > URL |
| `description` | 仅用户输入（Worker 不触及，只有 metaDescription 被 Worker 写入另一个字段） |
| `icon` / `iconWeight` / `color` | 仅用户输入 |
| `tags` | 用户输入 + AI 自动追加（`aiTag` 时） |

**当用户修改 URL 时**（见 [updateLinkById.ts#L150-L163](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L150-L163)）：

- ✅ **保留**：`name`、`description`、`icon`、`iconWeight`、`color`、`tags`
- ❌ **清空**（触发重新处理）：`image`、`pdf`、`readable`、`monolith`、`preview`、`lastPreserved`、`indexVersion`

---

## 七、失败 Fallback 机制总表

| 操作 | 失败场景 | Fallback 行为 | 代码证据 |
|---|---|---|---|
| 标题抓取 | 网络错误 / 超时 / 无 title 标签 | 返回空字符串 → 使用 URL 作为 name | [fetchTitleAndHeaders.ts#L40-L43](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/shared/fetchTitleAndHeaders.ts#L40-L43) |
| metaDescription 抓取 | 无 `<meta name="description">` | 不更新数据库（传 undefined，字段保持 null/原值） | [archiveHandler.ts#L158-L164](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts#L158-L164) |
| Favicon 第 1 源（Google） | 超时 / 非图片 / HTTP 错误 | 尝试 DuckDuckGo | [getFavicon/index.ts#L70-L72](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/getFavicon/index.ts#L70-L72) |
| Favicon 第 2 源（DuckDuckGo） | 全部失败 | 返回 204 No Content → UI 显示占位图标 | [getFavicon/index.ts#L85-L90](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/getFavicon/index.ts#L85-L90) |
| Preview OG Image 阶段 | 无 og:image / SSRF 拦截 / 网络错误 | 回退到 Playwright 页面截图 | [handleArchivePreview.ts#L59](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts#L59) |
| Preview generatePreview 超限 | buffer > 10MB | `preview = "unavailable"` 写入 DB，截图 Fallback 仍可覆盖 | [generatePreview.ts#L22-L33](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/packages/lib/generatePreview.ts#L22-L33) |
| Preview 截图 buffer 超限 | screenshot > 10MB | 仅 log 不写 DB → 阶段 1 的 "unavailable" 保留或 finally 写入 | [handleArchivePreview.ts#L63-L67](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts#L63-L67) |
| 浏览器页面加载 | 超时（默认 5 分钟 `BROWSER_TIMEOUT`） | 抛出异常 → finally 块标记所有未生成字段为 `"unavailable"` | [archiveHandler.ts#L66-L75](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts#L66-L75), [archiveHandler.ts#L199-L224](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts#L199-L224) |
| URL 安全检查（SSRF） | 内网 IP / 非 http(s) 协议 | 直接跳过所有归档处理，字段批量设为 `"unavailable"`，**不经过 finally** | [archiveHandler.ts#L44-L60](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts#L44-L60) |

---

## 八、缓存失效与重新处理（重点校正）

### 8.1 核心触发条件：`lastPreserved = null`

Worker 在 [getLinkBatchFairly.ts#L35-L38](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/getLinkBatchFairly.ts#L35-L38) 中选择待处理链接：
```js
baseLinkWhere = {
  url: { not: null },
  lastPreserved: null   // ← 核心条件
}
```
按 `createdAt` 倒序（新链接优先），在多用户之间公平轮询分配处理配额。

**⚠️ 配套条件**：即使 `lastPreserved = null`，Worker 进入 archiveHandler 后还会分别判断每个归档字段。例如 preview 的判断是 `if (!link.preview)`（见 5.1 节），所以：
- `lastPreserved = null` + `preview = "unavailable"` → Worker 会选中该链接，但进入后 **跳过 preview 生成**
- `lastPreserved = null` + `preview = null` → 正常生成 preview

### 8.2 触发 `lastPreserved = null` 的场景与字段重置

| 场景 | 重置字段（包括 preview?） | 代码证据 |
|---|---|---|
| 新建链接（DB 默认值） | 所有归档字段默认 null，lastPreserved 默认 null | [schema.prisma#L183-L192](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/packages/prisma/schema.prisma#L183-L192) |
| **用户修改链接 URL** | ✅ preview=null，image/pdf/readable/monolith 全置 null | [updateLinkById.ts#L157-L163](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L157-L163) |
| **用户手动「删除归档」**（DELETE /api/v1/links/archive） | ✅ **preview=null**，全置 null | [archive/index.ts#L73-L83](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/links/archive/index.ts#L73-L83) |
| **管理员「重新归档全部」**（action=allAndRePreserve） | ✅ **preview=null**，全置 null | [preservation.tsx#L50-L60](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L50-L60) |
| **管理员「重试失败归档」**（action=allBroken） | ❌ **preview 不重置！** 仅重置 image/pdf/readable/monolith 中为 "unavailable" 且用户启用了对应归档选项的字段 | [preservation.tsx#L127-L159](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L127-L159) |
| 客户端上传文件后（临时占位后立即置空） | ✅ preview 不涉及（PDF 时置为 unavailable） | [archives/index.ts#L222-L224](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/archives/index.ts#L222-L224) |

### 8.3 ⚠️ action=allBroken 的 preview 盲区

[preservation.tsx#L127-L132](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L127-L132) 的 `needsReprocessing` 判定：
```js
const needsReprocessing =
  (link.image === "unavailable" && shouldArchive.archiveAsScreenshot) ||
  (link.monolith === "unavailable" && shouldArchive.archiveAsMonolith) ||
  (link.pdf === "unavailable" && shouldArchive.archiveAsPDF) ||
  (link.readable === "unavailable" && shouldArchive.archiveAsReadable);
// ⚠️ 完全没有 preview 的检查！
```

对应的字段重置（[preservation.tsx#L137-L156](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L137-L156)）也不包含 preview。

**结论**：`preview = "unavailable"` 的链接无法通过管理员「重试失败归档」功能触发重新生成 —— 它只能靠：
1. 用户手动"删除归档"（DELETE /api/v1/links/archive）
2. 管理员"重新归档全部"（action=allAndRePreserve）
3. 修改该链接的 URL

### 8.4 处理完成后的状态

Worker 在 [archiveHandler.ts#L212-L224](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts#L212-L224) 的 finally 块中：
```js
data: {
  lastPreserved: new Date().toISOString(),   // ← 标记为已处理（停止轮询）
  readable: !finalLink.readable ? "unavailable" : undefined,
  image:    !finalLink.image    ? "unavailable" : undefined,
  monolith: !finalLink.monolith ? "unavailable" : undefined,
  pdf:      !finalLink.pdf      ? "unavailable" : undefined,
  preview:  !finalLink.preview  ? "unavailable" : undefined,
  indexVersion: null,                           // ← 触发搜索索引重建
}
```

### 8.5 搜索索引缓存

- `indexVersion = null` 触发 [linkIndexing.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/workers/linkIndexing.ts) 将该 link 重新同步到 Meilisearch
- 索引版本号由常量 `MEILI_INDEX_VERSION` 控制，版本升级可触发全站全量重索引

---

## 九、多语言页面（i18n）对元数据抓取的影响

### 9.1 User Locale 字段

User 模型中有 [locale](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/packages/prisma/schema.prisma#L37) 字段（默认 `"en"`），该字段**仅控制 Linkwarden 自身 UI 的显示语言**（通过 next-i18next）。

### 9.2 Worker 抓取时的 Locale

Worker 浏览器上下文在 [browser.ts#L34-L51](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/browser.ts#L34-L51) 中：
```js
export function getDefaultContextOptions(): BrowserContextOptions {
  const base: BrowserContextOptions = {
    ...devices["Desktop Chrome"],   // Playwright 内置的 Desktop Chrome 设备配置
    ignoreHTTPSErrors: ...,
  };
  // 未设置 locale、extraHTTPHeaders(Accept-Language)、userAgent 自定义语言等
  return base;
}
```

**关键结论：Worker 不传递任何用户级 locale 信息。** Playwright 的 `devices["Desktop Chrome"]` 默认行为：
- `locale`：未显式指定时使用系统/浏览器默认（通常为 `en-US`）
- `Accept-Language` 请求头：随浏览器默认 locale
- 不会因不同用户的 `user.locale` 而变化

### 9.3 对多语言站点的影响

对于根据 `Accept-Language` 头或 Cookie / IP 地理位置返回不同语言内容的网站：

| 抓取项目 | 受影响程度 | 说明 |
|---|---|---|
| `<title>` 标题 | ⚠️ 高 | 直接抓取到服务器默认语言版本（通常为英文），与用户预期语言可能不一致 |
| `metaDescription` | ⚠️ 高 | 同上，AI 打标签时使用的是默认语言描述 |
| 预览图（截图） | ⚠️ 高 | 页面文字语言为默认语言，非用户 locale |
| 可读存档（Readability） | ⚠️ 高 | 提取的正文文本为默认语言 |
| Monolith 完整存档 | ⚠️ 高 | 捕获的是默认语言渲染结果 |
| Favicon | ✅ 无影响 | 按 origin 抓取，与语言无关 |

### 9.4 已知限制

当前架构中不存在以下机制：
1. 读取用户 `locale` 并在 Playwright `browser.newContext({ locale: user.locale })` 中传递
2. 设置 `Accept-Language: {user-locale}` 请求头
3. 检测页面 `<html lang="...">` 与用户 locale 不匹配时的告警或重试
4. 存储多语言版本的元数据

因此，所有用户的抓取结果**统一使用 Worker 浏览器的默认 locale**（一般为 `en-US`），不会因用户界面语言不同而产生差异。

---

## 十、端到端时序图

```
用户提交新链接 (POST /api/v1/links)
       │
       ├─► 校验 URL 安全性 (SSRF 检查)
       │
       ├─► fetchTitleAndHeaders(url) ─── 10s 超时
       │       ├─ 成功: 提取 <title>
       │       └─ 失败: ""
       │
       ├─► name = link.name || title || url
       │    description = link.description (用户输入)
       │
       ├─► 写入 DB: lastPreserved=null (等待 Worker 处理)
       │    preview=null, image=null, pdf=null, ...
       │
       ▼
Worker 轮询 (getLinkBatchFairly: lastPreserved=null 的链接)
       │
       ├─► Playwright newContext()  ← 使用默认 locale，不传递用户语言
       │
       ├─► SSRF/协议检查失败?
       │    ├─ 是 → 所有字段批量写 "unavailable", lastPreserved=now() ✋
       │    └─ 否 → 继续
       │
       ├─► determineLinkType()  HEAD 请求 → content-type → url/pdf/image
       │
       ├─► page.goto(url, waitUntil: "domcontentloaded")
       │       │
       │       ├─► meta[name="description"] → metaDescription (500 字符)
       │       │
       │       ├─► !link.preview? → handleArchivePreview
       │       │     ├─ og:image → generatePreview
       │       │     │    ├─ 成功 → DB 写路径 ✓
       │       │     │    └─ 超限 → DB 写 "unavailable"
       │       │     └─ (previewGenerated=false) → page.screenshot fallback
       │       │          ├─ 成功 → 写路径 (可能覆盖 "unavailable") ✓
       │       │          └─ 失败/超限 → 由 finally 兜底
       │       │
       │       ├─► (可选) Readability 正文提取
       │       ├─► (可选) Screenshot / PDF
       │       ├─► (可选) Monolith 单文件存档
       │       └─► (可选) Wayback Machine
       │
       ├─ finally:
       │    ├─ lastPreserved = now()  ← 标记为已处理，停止轮询
       │    ├─ 从 DB 重新读取 finalLink
       │    ├─ 各字段: !finalLink.field ? "unavailable" : undefined
       │    └─ indexVersion = null (触发搜索索引)
       ▼
Favicon (仅 UI 渲染时按需请求)
       │
       ├─► 请求 URL 规范化?
       │    └─ 否 → 308 重定向 (缓存 1 小时)
       │
       └─► GET 规范化的 /api/v1/getFavicon?url={origin}
            ├─► Google faviconV2 (1.5s timeout)  ✓ 缓存 1 天 + immutable
            ├─► (失败) DuckDuckGo ip3
            └─► (全部失败) 204 No Content → UI 显示占位图标 (缓存 1 小时)
```
