# Linkwarden 链接元数据富集（Metadata Enrichment）规则详解

本文档基于代码阅读，系统梳理 Linkwarden 在链接创建、后台处理、缓存和多语言场景下的元数据抓取与更新规则。

---

## 一、元数据字段总览

每个 Link 记录在数据库中包含以下与元数据相关的字段（定义见 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/packages/prisma/schema.prisma#L166-L198)）：

| 字段 | 类型 | 来源 | 说明 |
|---|---|---|---|
| `name` | String | 用户输入 / 自动抓取 | 链接显示标题 |
| `description` | String | 用户输入 | 用户手工填写的描述，UI 直接展示 |
| `metaDescription` | String? | 自动抓取（Worker） | 页面 `<meta name="description">` 内容，用于 AI 打标签，**不直接在 UI 显示** |
| `preview` | String? | 自动生成 | 预览图（JPEG）文件路径，或 `"unavailable"` |
| `image` | String? | 用户上传 / 自动截图 | 完整截图文件路径，或 `"unavailable"` |
| `icon/iconWeight/color` | String? | 用户输入 | 用户自定义图标及样式 |
| `type` | String | 自动检测 | `url` / `pdf` / `image` |
| `lastPreserved` | DateTime? | 系统维护 | 上次归档处理时间戳，`null` 表示待处理 |
| `indexVersion` | Int? | 系统维护 | 搜索索引版本号，`null` 表示待重建索引 |

---

## 二、标题（name）抓取与优先级

### 2.1 创建时的标题决策链

在 [postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/api/controllers/links/postLink.ts#L78-L88) 中，标题按以下优先级确定：

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

- **超时保护**：10 秒超时（`Promise.race` + `setTimeout`）
- **协议限制**：仅处理 `http://` 或 `https://` 开头的 URL
- **容错**：任何异常（网络错误、解析失败等）均返回 `{ title: "", headers: null }`，不中断创建流程
- **内容注入**：允许直接传入 `content` 参数从 HTML 字符串中提取（用于客户端上传 HTML 文件的场景）

### 2.3 Worker 是否覆盖标题？

**不会。** Worker 的 [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts) 只更新：`metaDescription`、`preview`、`image`、`pdf`、`readable`、`monolith`、`lastPreserved`、`indexVersion`。`name` 字段在创建后只由用户编辑（见 [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L151)）。

---

## 三、描述（description / metaDescription）双字段机制

Linkwarden 采用**两个独立描述字段**的设计，用途截然不同：

### 3.1 `description`（用户描述）

- **写入来源**：仅用户手工输入（创建时 `link.description`、编辑时 `data.description`）
- **显示位置**：
  - Masonry 卡片视图：[LinkMasonry.tsx](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkMasonry.tsx#L147-L151)
  - RSS 导出：`link.description`
  - 阅读器视图标题 fallback：`link.name || link.description || link.url`
- **Worker 永不覆盖**

### 3.2 `metaDescription`（页面元描述）

- **写入来源**：仅 Worker 后台抓取，在 [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts#L151-L164) 中：
  ```js
  // Playwright 在浏览器上下文中执行
  const description = document.querySelector('meta[name="description"]');
  return description?.getAttribute("content") ?? undefined;
  ```
- **处理规则**：
  - `trim()` 后截取前 **500 字符**
  - 若未找到则不更新（传 `undefined` 给 Prisma）
- **用途**：
  - AI 自动打标签时的上下文输入（优先级高于 `textContent`），见 [autoTagLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/autoTagLink.ts#L76-L78)：
    ```
    description = metaDescription || textContent?.slice(0,500)
    ```
- **UI 中不显示**：在前端代码中未发现任何直接读取 `link.metaDescription` 用于展示的逻辑。

---

## 四、Favicon 抓取

### 4.1 架构特点

Favicon **不存入数据库**，而是通过 API 按需动态获取。

### 4.2 获取流程

前端 [LinkIcon.tsx](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L49-L69)：

```
用户设置了自定义 icon/iconWeight/color ?
   ├─ 是 → 显示 Phosphor 自定义图标
   └─ 否 → 链接类型为 "url" ?
              ├─ 是 → 请求 /api/v1/getFavicon?url={origin}
              ├─ pdf → 显示 bi-file-earmark-pdf
              └─ image → 显示 bi-file-earmark-image
```

### 4.3 Favicon API 规则

实现在 [getFavicon/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/getFavicon/index.ts)：

**来源优先级（依次尝试，第一个成功即返回）：**

1. **Google Favicon 服务**：
   ```
   https://t2.gstatic.com/faviconV2?client=SOCIAL&type=FAVICON
     &fallback_opts=TYPE,SIZE,URL&url={origin}&size=64
   ```

2. **DuckDuckGo 图标服务**：
   ```
   https://icons.duckduckgo.com/ip3/{hostname}.ico
   ```

**请求约束：**
- 单次请求超时 **1.5 秒**（`AbortController`）
- 仅接受 `image/*` Content-Type
- 自动跟随重定向

**URL 规范化与缓存：**
- 将任意 URL 重定向到其 `origin` 的规范形式（308 永久重定向）
- 成功响应：`Cache-Control: public, max-age=86400, s-maxage=2592000, stale-while-revalidate=604800, immutable`
  - 浏览器缓存 **1 天**
  - CDN 缓存 **30 天**
  - 后台可重新验证 **7 天**
- 失败（204 No Content）：`Cache-Control: public, max-age=3600, s-maxage=86400, stale-while-revalidate=604800`
  - 浏览器缓存 **1 小时**，避免频繁重试

**UI Fallback：** Favicon 加载完成前显示占位图标（`bi-link-45deg`），通过 `onLoad` + opacity 切换。

---

## 五、预览图（preview）生成规则

### 5.1 生成入口

Worker 的 [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts#L169)：
```js
if (!link.preview) await handleArchivePreview(link, page);
```
**关键判断**：仅当 `link.preview` 为空（非 `"unavailable"` 也非已有路径）时才生成。已有的预览图不会被覆盖。

### 5.2 两阶段 Fallback

实现在 [handleArchivePreview.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts)：

**阶段 1：OG Image 提取**
1. 在页面中查找 `<meta property="og:image">` 的 `content`
2. 相对路径自动补全为绝对 URL（使用 `document.location.origin`）
3. SSRF 安全检查（`assertUrlIsSafeForServerSideFetch`）
4. Playwright `page.goto(ogImageUrl)` 获取图像 buffer
5. 调用 `generatePreview()` 处理：
   - Jimp 读取 → resize 宽 **1000px**（高度自适应）→ JPEG **quality 20**
   - 大小检查：`PREVIEW_MAX_BUFFER`（默认 10MB），超限则标记为 `"unavailable"`
   - 存储路径：`archives/preview/{collectionId}/{linkId}.jpeg`
6. 成功后 `page.goBack()` 返回原页面

**阶段 2：页面截图（阶段 1 失败时的 Fallback）**
- Playwright 整页截图：`page.screenshot({ type: "jpeg", quality: 20 })`
- 同样执行 10MB 大小检查
- 存储路径与 OG Image 相同

### 5.3 预览图服务端缓存

归档文件通过 [[linkId].ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/archives/%5BlinkId%5D.ts#L122-L127) 读取时设置：
```
Cache-Control: private, max-age=31536000, immutable
```
即 **1 年私有缓存，标记 immutable**（URL 中附带 `&updatedAt={link.updatedAt}` 作为版本戳来触发浏览器刷新）。

---

## 六、用户手工字段优先级总结

以下字段**永不被 Worker 覆盖**，完全由用户控制：

| 字段 | 优先级 |
|---|---|
| `name` | 用户输入 > 自动抓取标题 > URL |
| `description` | 仅用户输入（Worker 不触及） |
| `icon` / `iconWeight` / `color` | 仅用户输入 |
| `tags` | 用户输入 + AI 自动追加（`aiTag` 时） |

**当用户修改 URL 时**（见 [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L150-L163)）：

- ✅ **保留**：`name`、`description`、`icon`、`iconWeight`、`color`、`tags`
- ❌ **清空**（触发重新处理）：`image`、`pdf`、`readable`、`monolith`、`preview`、`lastPreserved`、`indexVersion`

---

## 七、失败 Fallback 机制总表

| 操作 | 失败场景 | Fallback 行为 |
|---|---|---|
| 标题抓取 | 网络错误 / 超时 / 无 title 标签 | 返回空字符串 → 使用 URL 作为 name |
| metaDescription 抓取 | 无 `<meta name="description">` | 不更新数据库（字段保持 null/原值） |
| Favicon 第 1 源（Google） | 超时 / 非图片 / HTTP 错误 | 尝试 DuckDuckGo |
| Favicon 第 2 源（DuckDuckGo） | 全部失败 | 返回 204 No Content → UI 显示占位图标 |
| Preview OG Image 抓取 | 无 og:image / 网络错误 / 不安全 URL | 回退到 Playwright 页面截图 |
| Preview 处理 | buffer > 10MB / Jimp 处理异常 | `preview = "unavailable"`，不再重试 |
| 浏览器页面加载 | 超时（默认 5 分钟 `BROWSER_TIMEOUT`） | 抛出异常 → finally 块标记所有未生成字段为 `"unavailable"` |
| 任意归档字段 | 处理失败未生成值 | Worker finally 中设为 `"unavailable"`（见 [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts#L208-L224)） |
| URL 安全检查（SSRF） | 内网 IP / 非 http(s) 协议 | 跳过所有归档处理，字段批量设为 `"unavailable"` |

---

## 八、缓存失效与重新处理

### 8.1 核心触发条件：`lastPreserved = null`

Worker 在 [getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/getLinkBatchFairly.ts#L35-L38) 中选择待处理链接：
```js
baseLinkWhere = {
  url: { not: null },
  lastPreserved: null   // ← 核心条件
}
```
按 `createdAt` 倒序（新链接优先），在多用户之间公平轮询分配处理配额。

### 8.2 触发 `lastPreserved = null` 的场景

| 场景 | 代码位置 |
|---|---|
| 新建链接（默认值） | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/packages/prisma/schema.prisma#L192) |
| 用户修改链接 URL | [updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L162) |
| 用户手动「删除归档」（DELETE /api/v1/links/archive） | [archive/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/links/archive/index.ts#L73-L83) |
| 管理员「重新归档全部」（DELETE /api/v1/worker/preservation?action=allAndRePreserve） | [preservation.tsx](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L50-L60) |
| 管理员「重试失败归档」（action=allBroken），仅对标记为 `"unavailable"` 且启用了对应归档选项的字段 | [preservation.tsx](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L127-L159) |
| 客户端上传文件后（临时占位后立即置空触发后续 AI 标签） | [archives/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/web/pages/api/v1/archives/index.ts#L222-L224) |

### 8.3 处理完成后的状态

Worker 在 [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/archiveHandler.ts#L212-L224) 的 finally 块中：
```js
data: {
  lastPreserved: new Date().toISOString(),   // ← 标记为已处理
  readable: !finalLink.readable ? "unavailable" : undefined,
  image:    !finalLink.image    ? "unavailable" : undefined,
  monolith: !finalLink.monolith ? "unavailable" : undefined,
  pdf:      !finalLink.pdf      ? "unavailable" : undefined,
  preview:  !finalLink.preview  ? "unavailable" : undefined,
  indexVersion: null,                           // ← 触发搜索索引重建
}
```

### 8.4 搜索索引缓存

- `indexVersion = null` 触发 [linkIndexing.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/workers/linkIndexing.ts) 将该 link 重新同步到 Meilisearch
- 索引版本号由常量 `MEILI_INDEX_VERSION` 控制，版本升级可触发全站全量重索引

---

## 九、多语言页面（i18n）对元数据抓取的影响

### 9.1 User Locale 字段

User 模型中有 [locale](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/packages/prisma/schema.prisma#L37) 字段（默认 `"en"`），该字段**仅控制 Linkwarden 自身 UI 的显示语言**（通过 next-i18next）。

### 9.2 Worker 抓取时的 Locale

Worker 浏览器上下文在 [browser.ts](file:///d:/fz/0601/solo-dogfeeding/code/96-linkwarden/apps/worker/lib/browser.ts#L34-L51) 中：
```js
export function getDefaultContextOptions(): BrowserContextOptions {
  const base: BrowserContextOptions = {
    ...devices["Desktop Chrome"],   // Playwright 内置的 Desktop Chrome 设备配置
    ignoreHTTPSErrors: ...,
  };
  // 未设置 locale、extraHTTPHeaders(Accept-Language) 等
  return base;
}
```

**关键结论：Worker 不传递任何用户级 locale 信息。** Playwright 的 `devices["Desktop Chrome"]` 默认行为：
- `locale`：未显式指定时使用系统/浏览器默认（通常为 `en-US`）
- `Accept-Language` 请求头：随浏览器默认 locale

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
       │
       ▼
Worker 轮询 (getLinkBatchFairly: lastPreserved=null 的链接)
       │
       ├─► Playwright newContext()  ← 使用默认 locale，不传递用户语言
       │
       ├─► determineLinkType()  HEAD 请求 → content-type → url/pdf/image
       │
       ├─► page.goto(url, waitUntil: "domcontentloaded")
       │       │
       │       ├─► meta[name="description"] → metaDescription (500 字符)
       │       │
       │       ├─► handleArchivePreview:
       │       │     ├─► og:image → Jimp 处理 (1000px, q=20)
       │       │     └─► (失败) → page.screenshot (jpeg, q=20)
       │       │
       │       ├─► (可选) Readability 正文提取
       │       ├─► (可选) Screenshot / PDF
       │       ├─► (可选) Monolith 单文件存档
       │       └─► (可选) Wayback Machine
       │
       ├─ finally:
       │    ├─ lastPreserved = now()
       │    ├─ 未生成的字段 → "unavailable"
       │    └─ indexVersion = null (触发搜索索引)
       ▼
Favicon (仅 UI 渲染时按需请求)
       │
       ├─► GET /api/v1/getFavicon?url={origin}
       │     ├─► Google faviconV2 (1.5s timeout)
       │     └─► (失败) DuckDuckGo ip3
       │     └─► (全部失败) 204 No Content → UI 显示占位图标
```
