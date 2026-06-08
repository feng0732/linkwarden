# Linkwarden 链接元数据富集（Metadata Enrichment）规则详解

> 仓库：`feng0732/linkwarden` · 分支：`task-96`
> 本文档基于逐行代码阅读，每条结论均附带 **仓库相对路径** + **GitHub 可打开链接**，可直接复核。

---

## 一、元数据字段总览

Link 模型在 `packages/prisma/schema.prisma` 中定义，与元数据相关的字段如下：
`packages/prisma/schema.prisma#L166-L198` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/packages/prisma/schema.prisma#L166-L198))

| 字段 | 类型 | 来源 | 说明 |
|---|---|---|---|
| `name` | String | 用户输入 / 自动抓取 | 链接显示标题 |
| `description` | String | 用户输入 | 用户手工填写的描述，UI 直接展示 |
| `metaDescription` | String? | 自动抓取（Worker） | 页面 `<meta name="description">` 内容，用于 AI 打标签，**不直接在 UI 显示** |
| `preview` | String? | 自动生成 | 预览图文件路径，或 `"unavailable"` |
| `image` | String? | 用户上传 / 自动截图 | 完整截图文件路径，或 `"unavailable"` |
| `icon` | String? | 用户输入 | 自定义 Phosphor 图标名；**仅此字段决定是否显示自定义图标（与 favicon 二选一）** |
| `iconWeight` | String? | 用户输入 | 自定义图标粗细（`thin`/`light`/`regular`/`bold`/`fill`/`duotone`），默认 `"regular"` |
| `color` | String? | 用户输入 | 自定义图标颜色（十六进制色值），默认主题主色 `--p` |
| `type` | String | 自动检测 | `url` / `pdf` / `image` |
| `lastPreserved` | DateTime? | 系统维护 | 上次归档处理时间戳，`null` 表示待处理 |
| `indexVersion` | Int? | 系统维护 | 搜索索引版本号，`null` 表示待重建索引 |

**"unavailable" 语义判定**：字段为 truthy 且不等于 `"unavailable"` 时才算可用。
`packages/lib/formatStats.ts#L4-L9` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/packages/lib/formatStats.ts#L4-L9))
```js
return Boolean(link && link[format] && link[format] !== "unavailable");
```

---

## 二、标题（name）抓取与优先级

### 2.1 创建时的标题决策链

在创建链接时，标题按以下优先级确定：
`apps/web/lib/api/controllers/links/postLink.ts#L78-L88` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/lib/api/controllers/links/postLink.ts#L78-L88))

```
1. 用户提供的 link.name（非空字符串时）
   ↓
2. 若 URL 安全可抓取，则从页面 <title> 标签中提取
   （正则 /<title.*>([^<]*)<\/title>/）
   ↓
3. 回退为 URL 本身
   ↓
4. 完全无 URL 时为空字符串
```

### 2.2 fetchTitleAndHeaders 实现细节

`apps/web/lib/shared/fetchTitleAndHeaders.ts` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/lib/shared/fetchTitleAndHeaders.ts))

| 特性 | 代码位置 | 说明 |
|---|---|---|
| 协议限制 | `#L7-L8` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/lib/shared/fetchTitleAndHeaders.ts#L7-L8)) | 仅处理 `http://` 或 `https://` |
| 超时保护 | `#L12-L16` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/lib/shared/fetchTitleAndHeaders.ts#L12-L16)) | 10 秒超时（Promise.race + setTimeout） |
| 内容注入 | `#L23-L27` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/lib/shared/fetchTitleAndHeaders.ts#L23-L27)) | 允许直接传入 HTML content（客户端上传场景） |
| 容错兜底 | `#L40-L43` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/lib/shared/fetchTitleAndHeaders.ts#L40-L43)) | 任何异常均返回 `{ title: "", headers: null }` |

### 2.3 Worker 是否覆盖标题？

**不会。** Worker `archiveHandler` 只更新：`metaDescription`、`preview`、`image`、`pdf`、`readable`、`monolith`、`lastPreserved`、`indexVersion`。`name` 字段在创建后只由用户编辑（`apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L151`）。

---

## 三、描述（description / metaDescription）双字段机制

### 3.1 `description`（用户描述）

- **写入来源**：仅用户手工输入
  - 创建时：`apps/web/lib/api/controllers/links/postLink.ts#L108` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/lib/api/controllers/links/postLink.ts#L108))
  - 编辑时：`apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L153` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L153))
- **UI 显示位置**：
  - Masonry 卡片：`apps/web/components/LinkViews/LinkComponents/LinkMasonry.tsx#L147-L151` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkMasonry.tsx#L147-L151)) — 仅当 `show.description && link.description`
  - 阅读器标题 fallback：`apps/web/components/Preservation/ReadableView.tsx#L441` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/Preservation/ReadableView.tsx#L441)) — `link.name \|\| link.description \|\| link.url`
  - RSS 导出：`link.description`
- **Worker 永不覆盖**

### 3.2 `metaDescription`（页面元描述）

- **写入来源**：仅 Worker 后台抓取
  `apps/worker/lib/archiveHandler.ts#L151-L164` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/archiveHandler.ts#L151-L164))
  ```js
  // Playwright 浏览器上下文中执行
  const description = document.querySelector('meta[name="description"]');
  return description?.getAttribute("content") ?? undefined;
  ```
- **处理规则**：
  - `trim()` 后截取前 **500 字符**
  - 未找到则传 `undefined` 给 Prisma，**不覆盖原值**
- **用途**：AI 自动打标签时的上下文输入（优先级高于 `textContent`）
  `apps/worker/lib/autoTagLink.ts#L76-L78` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/autoTagLink.ts#L76-L78))
  ```
  description = (metaDescription ? metaDescription + "..." : undefined)
                || (textContent ? textContent?.slice(0, 500) + "..." : undefined)
  ```
- **UI 中不显示**：前端代码中未发现直接读取 `link.metaDescription` 用于展示的逻辑。

---

## 四、Favicon 抓取

### 4.1 架构特点

Favicon **不存入数据库**，通过 API 按需动态获取。

### 4.2 LinkIcon 完整分支决策链

`apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx))

**⚠️ 核心事实**：只有 `link.icon` 一个字段控制"自定义图标 vs favicon"的分支选择。`iconWeight` 和 `color` 仅在自定义图标分支中作为样式参数使用，不参与分支判断。

完整分支逻辑（代码 `#L39-L88`，[GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L39-L88)）：

```
link.icon 有值（非空字符串）?
   ├─ 是 → 分支 A：显示 Phosphor 自定义图标
   │       ├─ icon 名：link.icon（如 "bookmark", "link" 等）
   │       ├─ weight：link.iconWeight || "regular"   （默认 regular）
   │       │         可选值：thin / light / regular / bold / fill / duotone
   │       └─ color： link.color || oklchVariableToHex("--p")
   │                  （默认主题主色 --p，从 oklch 变量转换为 hex）
   │
   └─ 否 → 进入自动分支
            │
            ├─ link.type === "url" && url 合法可解析为 URL?
            │    └─ 是 → 分支 B：请求 favicon
            │             GET /api/v1/getFavicon?url={origin}
            │             加载完成前显示占位图标 bi-link-45deg
            │
            ├─ link.type === "pdf"?
            │    └─ 是 → 分支 C：显示占位图标 bi-file-earmark-pdf
            │
            ├─ link.type === "image"?
            │    └─ 是 → 分支 D：显示占位图标 bi-file-earmark-image
            │
            └─ 以上均不满足 → 分支 E：不渲染任何内容（undefined）
```

**各分支代码定位**：

| 分支 | 代码位置 | 说明 |
|---|---|---|
| A 自定义图标 | `#L39-L48` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L39-L48)) | 渲染 `<Icon>` 组件（Phosphor） |
| B Favicon | `#L49-L70` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L49-L70)) | Next.js `<Image>` + `onLoad` 切换 opacity |
| C PDF 占位 | `#L71-L75` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L71-L75)) | Bootstrap Icons bi-file-earmark-pdf |
| D Image 占位 | `#L76-L80` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L76-L80)) | Bootstrap Icons bi-file-earmark-image |
| E 不渲染 | `#L81-L88` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L81-L88)) | 注释掉的 Monolith 分支 + undefined |

**使用 LinkIcon 的视图组件**：
- Masonry 瀑布流：`apps/web/components/LinkViews/LinkComponents/LinkMasonry.tsx#L122` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkMasonry.tsx#L122))
- List 列表：`apps/web/components/LinkViews/LinkComponents/LinkList.tsx#L93` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkList.tsx#L93))

### 4.3 Favicon API：两级缓存 + 双来源 Fallback

`apps/web/pages/api/v1/getFavicon/index.ts` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/getFavicon/index.ts))

#### 第一级：URL 规范化重定向缓存

任意 URL 先规范化为 origin 形式：
`#L57-L61` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/getFavicon/index.ts#L57-L61))
```js
const canonical = `/api/v1/getFavicon?url=${encodeURIComponent(origin)}`;
if (req.url !== canonical) {
  res.setHeader("Cache-Control", "public, max-age=3600");  // 浏览器缓存 1 小时
  return res.redirect(308, canonical);  // 308 = 永久重定向，浏览器会缓存
}
```

#### 第二级：实际 Favicon 响应缓存

规范化后依次尝试（第一个成功即返回）：
`#L63-L83` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/getFavicon/index.ts#L63-L83))

1. **Google Favicon 服务**：
   ```
   https://t2.gstatic.com/faviconV2?client=SOCIAL&type=FAVICON
     &fallback_opts=TYPE,SIZE,URL&url={origin}&size=64
   ```

2. **DuckDuckGo 图标服务**（Google 失败时尝试）：
   ```
   https://icons.duckduckgo.com/ip3/{hostname}.ico
   ```

**请求约束** `#L8-L31` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/getFavicon/index.ts#L8-L31))：
- 单次请求超时 **1.5 秒**（`AbortController`）
- 仅接受 `image/*` Content-Type
- 自动跟随重定向

**成功响应缓存头** `#L76-L79` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/getFavicon/index.ts#L76-L79))：
```
Cache-Control: public, max-age=86400, s-maxage=2592000, stale-while-revalidate=604800, immutable
```
| 指令 | 值 | 含义 |
|---|---|---|
| `max-age` | 86400 | 浏览器缓存 **1 天** |
| `s-maxage` | 2592000 | CDN/共享缓存 **30 天** |
| `stale-while-revalidate` | 604800 | 过期后 **7 天**内可返回旧值同时后台刷新 |
| `immutable` | — | 资源不可变，浏览器刷新也不重新验证 |

**失败响应（204 No Content）缓存头** `#L86-L89` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/getFavicon/index.ts#L86-L89))：
```
Cache-Control: public, max-age=3600, s-maxage=86400, stale-while-revalidate=604800
```
- 浏览器缓存 **1 小时**（比成功短，便于尽快重试成功）
- CDN 缓存 **1 天**

**UI Fallback**：Favicon 加载完成前显示占位图标（`bi-link-45deg`），通过 `onLoad` + state 切换 opacity
`apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L31-L69` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L31-L69))
（`faviconLoaded` state 在 `#L31` 声明，`onLoad` 在 `#L62` 触发，opacity 切换在 `#L57-L60`）

### 4.4 Favicon 缓存完整链路

```
浏览器请求 /api/v1/getFavicon?url={任意URL}
       │
       ├─ URL 规范化检查
       │    ├─ 非规范形式 → 308 重定向 (缓存 1h)
       │    └─ 规范形式 → 继续
       │
       ├─ 依次尝试来源
       │    ├─ Google faviconV2 (1.5s 超时)
       │    │    └─ 成功 → 200 + 缓存 1天 + immutable ✓
       │    ├─ DuckDuckGo ip3 (1.5s 超时)
       │    │    └─ 成功 → 200 + 缓存 1天 + immutable ✓
       │    └─ 全部失败 → 204 No Content + 缓存 1h
       │
       ▼
```

---

## 五、预览图（preview）生成规则与异常 Fallback

### 5.1 生成入口判断

`apps/worker/lib/archiveHandler.ts#L168-L169` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/archiveHandler.ts#L168-L169))
```js
// Preview
if (!link.preview) await handleArchivePreview(link, page);
```

**⚠️ 关键细节**：条件是 `!link.preview`（falsy 判断）。字符串 `"unavailable"` 是非空字符串（truthy），因此：

| `link.preview` 值 | `!link.preview` | 是否执行 handleArchivePreview |
|---|---|---|
| `null` | `true` | ✅ 执行 |
| `"unavailable"` | `false` | ❌ **永不执行** |
| `"archives/preview/..."` | `false` | 不执行（已有预览图） |

**结论**：一旦 preview 被标记为 `"unavailable"`，Worker 永远不会再自动重试生成预览图，除非手动重置为 `null`。

### 5.2 handleArchivePreview 两阶段 Fallback 详解

`apps/worker/lib/preservationScheme/handleArchivePreview.ts` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/preservationScheme/handleArchivePreview.ts))

#### 阶段 1：OG Image 提取 `#L21-L57` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/preservationScheme/handleArchivePreview.ts#L21-L57))

```
1. page.evaluate() 查找 <meta property="og:image"> content  (#L21-L24)
   └─ 不存在 → ogImageUrl = null → 跳过此阶段，进入阶段 2
2. 相对路径自动补全为绝对 URL（使用 document.location.origin）(#L29-L36)
3. SSRF 安全检查 assertUrlIsSafeForServerSideFetch(ogImageUrl)  (#L38-L39)
   └─ 不安全 → catch 静默跳过（仅非 UnsafeUrlError 才重新抛出）(#L52-L56)
4. Playwright page.goto(ogImageUrl) 获取图像响应               (#L40)
5. 再次判断：imageResponse 有效 && !link.preview?.startsWith("archive")  (#L42)
   注意：此处 link.preview 是函数参数快照，非 DB 实时值
6. 调用 generatePreview(buffer, collectionId, linkId)           (#L44-L48)
```

**generatePreview 内部行为**：
`packages/lib/generatePreview.ts` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/packages/lib/generatePreview.ts))

| 场景 | 行为 | DB preview 结果 | 返回值 | 代码 |
|---|---|---|---|---|
| 参数无效（buffer/collectionId/linkId 任一假值） | 跳过 | 不变 | `false` | `#L10` |
| Jimp.read 失败或无 image | `console.log` 错误 | 不变 | `false` | `#L14-L16` |
| Jimp 处理异常（catch 块） | `console.error` | 不变 | `false` | `#L49-L51` |
| 处理后 buffer > PREVIEW_MAX_BUFFER（默认 10MB） | log + DB update | **`"unavailable"`** | `false` | `#L22-L33` |
| 一切正常 | resize 1000px × AUTO, JPEG q=20, 写文件 | `archives/preview/...jpeg` | `true` | `#L19-L46` |

**⚠️ 重要**：generatePreview 将 preview 写入 DB 后，JS 变量 `link.preview`（函数参数）仍是调用时原值（通常为 `null`），不同步更新。

#### 阶段 2：页面截图 Fallback `#L59-L81` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/preservationScheme/handleArchivePreview.ts#L59-L81))

```js
if (!previewGenerated && !link.preview?.startsWith("archive")) {
  await page
    .screenshot({ type: "jpeg", quality: 20 })
    .then(async (screenshot) => {
      if (Buffer.byteLength(screenshot) > 1024*1024*Number(PREVIEW_MAX_BUFFER || 10))
        return console.log("Buffer size exceeded");  // 仅 log，不写 DB
      await createFile({ data: screenshot, filePath: "archives/preview/..." });
      await prisma.link.update({ data: { preview: "archives/preview/..." } });
    });
}
```

**进入条件**：
- `!previewGenerated`：阶段 1 未成功生成
- `!link.preview?.startsWith("archive")`：函数参数 `link.preview` 不以 "archive" 开头（通常 `null` → `!null?.startsWith(...)` = `true`）

**阶段 2 异常路径分析**：

| 场景 | 行为 | 最终 DB preview 结果 |
|---|---|---|
| `page.screenshot()` 抛出异常 | `.then()` 不执行 → Promise rejected → 异常上抛到 archiveHandler try/catch | 进入 finally，基于 DB 现状：null → `"unavailable"`；已有 `"unavailable"` → 不变 |
| screenshot buffer 超限 | `return console.log(...)`，不写 DB | 阶段 1 若写了 `"unavailable"` 则保留；否则仍为 null → finally 写 `"unavailable"` |
| `createFile` / `prisma.update` 抛出 | `.then()` 内 Promise rejected → 向上抛出 | 同上，进入 finally 兜底 |
| 一切正常 | 写文件 + DB 更新为有效路径 | `archives/preview/...jpeg` ✅（可覆盖阶段 1 的 `"unavailable"`） |

### 5.3 archiveHandler finally 兜底

无论正常或异常，finally 一定执行：
`apps/worker/lib/archiveHandler.ts#L208-L224` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/archiveHandler.ts#L208-L224))
```js
const finalLink = await prisma.link.findUnique({ where: { id: link.id } });
if (finalLink) {
  await prisma.link.update({
    where: { id: link.id },
    data: {
      lastPreserved: new Date().toISOString(),
      preview: !finalLink.preview ? "unavailable" : undefined,
      // readable / image / monolith / pdf 同样逻辑
      indexVersion: null,
    },
  });
}
```

**关键判断**：`!finalLink.preview` — 从数据库**重新读取**后判定：
- `finalLink.preview = null` → 真值 → 写入 `"unavailable"`
- `finalLink.preview = "unavailable"` → 假值 → 不覆盖（`undefined`）
- `finalLink.preview = "archives/preview/..."` → 假值 → 不覆盖

### 5.4 SSRF / 协议检查失败时的提前返回

`apps/worker/lib/archiveHandler.ts#L44-L60` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/archiveHandler.ts#L44-L60))

当 URL 未通过 SSRF 安全检查或非 http(s) 协议时，**直接批量写入 `"unavailable"` 并 return，不经过 finally**：
```js
if (skipPreservation || (!link.url?.startsWith("http://") && !link.url?.startsWith("https://"))) {
  await prisma.link.update({
    data: {
      lastPreserved: new Date().toISOString(),
      readable: "unavailable", image: "unavailable", monolith: "unavailable",
      pdf: "unavailable", preview: "unavailable",
      indexVersion: null,
    },
  });
  return;  // 不经过 finally
}
```

### 5.5 预览图完整 Fallback 路径图

```
archiveHandler 入口
  │
  ├─ SSRF/协议检查失败?
  │    └─ 是 → 所有字段批量写 "unavailable", lastPreserved=now(), return ✋
  │
  ├─ !link.preview?
  │    ├─ 否 (preview="unavailable" 或已有路径) → 跳过，永不重试 ✋
  │    └─ 是 (preview=null)
  │         │
  │         ▼
  │   handleArchivePreview
  │         │
  │         ├─ 阶段 1: og:image (#L21-L57)
  │         │    ├─ 成功 → DB 写路径 ✓
  │         │    ├─ 超限 (generatePreview #L22-L33) → DB 写 "unavailable" + previewGenerated=false
  │         │    ├─ Jimp 异常 (#L49-L51) → 只 log + previewGenerated=false
  │         │    └─ 无 og:image / SSRF 拦截 / 网络错 → previewGenerated=false
  │         │
  │         └─ 阶段 2: 截图 (#L59-L81) — 仅当 previewGenerated=false
  │              ├─ 成功 → 写路径 (可覆盖阶段 1 的 "unavailable") ✓
  │              ├─ 截图 buffer 超限 (#L63-L67) → 仅 log，不写 DB
  │              └─ screenshot 抛错 → 异常上抛
  │
  ▼
finally 块 (#L208-L224) — 一定执行
  重新读 DB → finalLink.preview
  ├─ null → 写入 "unavailable"
  ├─ "unavailable" → 不变
  └─ 有效路径 → 不变
```

### 5.6 预览图服务端文件缓存

归档文件通过 API 读取时设置：
`apps/web/pages/api/v1/archives/[linkId].ts#L122-L127` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/archives/%5BlinkId%5D.ts#L122-L127))
```
Cache-Control: private, max-age=31536000, immutable
```
即 **1 年私有缓存 + immutable**。URL 中附带 `&updatedAt={link.updatedAt}` 作为版本戳触发浏览器刷新。

---

## 六、用户手工字段优先级与显示规则总结

以下字段**永不被 Worker 覆盖**，完全由用户控制：

| 字段 | 显示条件 / 优先级 | 默认值 | 代码证据 |
|---|---|---|---|
| `name` | 用户输入 > 自动抓取标题 > URL | — | [postLink.ts#L81-L88](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/lib/api/controllers/links/postLink.ts#L81-L88) |
| `description` | 仅用户输入（Worker 写 metaDescription 到另一个字段，不展示） | 空字符串 `""` | [schema.prisma#L168](https://github.com/feng0732/linkwarden/blob/task-96/packages/prisma/schema.prisma#L168) |
| `icon` | **仅此字段控制分支**：有值 → 显示自定义 Phosphor 图标；无值 → 进入自动分支（favicon/PDF/Image 占位） | `null`（DB 无默认值） | [LinkIcon.tsx#L39](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L39) |
| `iconWeight` | 仅在 `link.icon` 有值时生效：Phosphor 图标粗细 | `"regular"` | [LinkIcon.tsx#L44](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L44) |
| `color` | 仅在 `link.icon` 有值时生效：Phosphor 图标颜色 | 主题主色 `--p`（通过 `oklchVariableToHex` 转换） | [LinkIcon.tsx#L45](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/components/LinkViews/LinkComponents/LinkIcon.tsx#L45) |
| `tags` | 用户输入 + AI 自动追加（aiTag 时） | `[]` | — |

**⚠️ 关键区分**：
- `icon`：**分支控制字段**，决定走自定义图标还是自动图标分支
- `iconWeight` / `color`：**样式参数**，仅在 `icon` 有值时才被读取，不参与分支选择

**Prisma Schema 字段定义对比**：
`packages/prisma/schema.prisma#L178-L180` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/packages/prisma/schema.prisma#L178-L180))
```prisma
icon            String?
iconWeight      String?
color           String?
```
三者均为 `String?`（可空），DB 层**均无默认值**（注意：Collection 模型的 `color` 有默认 `"#0ea5e9"`，但 Link 模型没有）。

**当用户修改 URL 时**：
`apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L150-L163` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L150-L163))

- ✅ **保留**：`name`、`description`、`icon`、`iconWeight`、`color`、`tags`
- ❌ **清空**（触发重新处理）：`image`、`pdf`、`readable`、`monolith`、**`preview`**、`lastPreserved`、`indexVersion`

---

## 七、失败 Fallback 机制总表

| 操作 | 失败场景 | Fallback 行为 | 代码证据 |
|---|---|---|---|
| 标题抓取 | 网络错误 / 超时 / 无 title 标签 | 返回空字符串 → 使用 URL 作为 name | `apps/web/lib/shared/fetchTitleAndHeaders.ts#L40-L43` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/lib/shared/fetchTitleAndHeaders.ts#L40-L43)) |
| metaDescription 抓取 | 无 `<meta name="description">` | 不更新 DB（传 undefined，不覆盖原值） | `apps/worker/lib/archiveHandler.ts#L158-L164` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/archiveHandler.ts#L158-L164)) |
| Favicon 第 1 源（Google） | 超时 / 非图片 / HTTP 错误 | 尝试 DuckDuckGo | `apps/web/pages/api/v1/getFavicon/index.ts#L70-L72` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/getFavicon/index.ts#L70-L72)) |
| Favicon 第 2 源（DuckDuckGo） | 全部失败 | 返回 204 No Content → UI 显示占位图标 | `apps/web/pages/api/v1/getFavicon/index.ts#L85-L90` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/getFavicon/index.ts#L85-L90)) |
| Preview OG Image 阶段 | 无 og:image / SSRF 拦截 / 网络错误 | 回退到 Playwright 页面截图 | `apps/worker/lib/preservationScheme/handleArchivePreview.ts#L59` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/preservationScheme/handleArchivePreview.ts#L59)) |
| Preview generatePreview 超限 | buffer > 10MB | `preview = "unavailable"` 写入 DB，截图 Fallback 仍可覆盖 | `packages/lib/generatePreview.ts#L22-L33` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/packages/lib/generatePreview.ts#L22-L33)) |
| Preview 截图 buffer 超限 | screenshot > 10MB | 仅 log 不写 DB → 阶段 1 的 "unavailable" 保留或 finally 写入 | `apps/worker/lib/preservationScheme/handleArchivePreview.ts#L63-L67` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/preservationScheme/handleArchivePreview.ts#L63-L67)) |
| 浏览器页面加载 | 超时（默认 5 分钟 BROWSER_TIMEOUT） | 抛出异常 → finally 标记未生成字段为 `"unavailable"` | `apps/worker/lib/archiveHandler.ts#L66-L75` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/archiveHandler.ts#L66-L75)) + `#L199-L224` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/archiveHandler.ts#L199-L224)) |
| URL SSRF / 协议检查 | 内网 IP / 非 http(s) 协议 | 直接批量写 `"unavailable"`，**不经过 finally** | `apps/worker/lib/archiveHandler.ts#L44-L60` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/archiveHandler.ts#L44-L60)) |

---

## 八、缓存失效与重新处理

### 8.1 核心触发条件：`lastPreserved = null`

Worker 选择待处理链接的核心条件：
`apps/worker/lib/getLinkBatchFairly.ts#L35-L38` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/getLinkBatchFairly.ts#L35-L38))
```js
baseLinkWhere = {
  url: { not: null },
  lastPreserved: null,   // ← 核心条件
}
```
按 `createdAt` 倒序（新链接优先），在多用户之间公平轮询分配。

**⚠️ 配套条件**：即使 `lastPreserved = null`，Worker 进入 archiveHandler 后还会分别判断每个归档字段。例如 preview 的判断是 `if (!link.preview)`（见 5.1 节），所以：
- `lastPreserved = null` + `preview = "unavailable"` → Worker 选中链接，但**跳过 preview 生成**
- `lastPreserved = null` + `preview = null` → 正常生成 preview

### 8.2 触发 `lastPreserved = null` 的场景与字段重置

| 场景 | 是否重置 preview 为 null | 代码证据 |
|---|---|---|
| 新建链接（DB 默认值） | ✅ 所有归档字段默认 null | `packages/prisma/schema.prisma#L183-L192` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/packages/prisma/schema.prisma#L183-L192)) |
| **用户修改链接 URL** | ✅ preview=null 全置 null | `apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L157-L163` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L157-L163)) |
| **用户手动「删除归档」**（DELETE /api/v1/links/archive） | ✅ **preview=null**，全置 null | `apps/web/pages/api/v1/links/archive/index.ts#L73-L83` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/links/archive/index.ts#L73-L83)) |
| **管理员「重新归档全部」**（action=allAndRePreserve） | ✅ **preview=null**，全置 null | `apps/web/pages/api/v1/worker/preservation.tsx#L50-L60` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/worker/preservation.tsx#L50-L60)) |
| **管理员「重试失败归档」**（action=allBroken） | ❌ **preview 不重置！** 仅重置 image/pdf/readable/monolith | `apps/web/pages/api/v1/worker/preservation.tsx#L127-L159` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/worker/preservation.tsx#L127-L159)) |

### 8.3 ⚠️ action=allBroken 的 preview 盲区

`needsReprocessing` 判定完全不包含 preview：
`apps/web/pages/api/v1/worker/preservation.tsx#L127-L132` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/worker/preservation.tsx#L127-L132))
```js
const needsReprocessing =
  (link.image === "unavailable" && shouldArchive.archiveAsScreenshot) ||
  (link.monolith === "unavailable" && shouldArchive.archiveAsMonolith) ||
  (link.pdf === "unavailable" && shouldArchive.archiveAsPDF) ||
  (link.readable === "unavailable" && shouldArchive.archiveAsReadable);
// ⚠️ 完全没有 preview 的检查！
```

对应的字段重置也不包含 preview：
`apps/web/pages/api/v1/worker/preservation.tsx#L137-L156` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/web/pages/api/v1/worker/preservation.tsx#L137-L156))

**结论**：`preview = "unavailable"` 的链接无法通过管理员「重试失败归档」触发重新生成，只能通过以下三种方式重置：
1. 用户手动"删除归档"（DELETE `/api/v1/links/archive`）
2. 管理员"重新归档全部"（action=allAndRePreserve）
3. 修改该链接的 URL

### 8.4 处理完成后的状态

Worker finally 块：
`apps/worker/lib/archiveHandler.ts#L212-L224` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/archiveHandler.ts#L212-L224))
```js
data: {
  lastPreserved: new Date().toISOString(),   // ← 标记为已处理，停止轮询
  readable: !finalLink.readable ? "unavailable" : undefined,
  image:    !finalLink.image    ? "unavailable" : undefined,
  monolith: !finalLink.monolith ? "unavailable" : undefined,
  pdf:      !finalLink.pdf      ? "unavailable" : undefined,
  preview:  !finalLink.preview  ? "unavailable" : undefined,
  indexVersion: null,                           // ← 触发搜索索引重建
}
```

### 8.5 搜索索引缓存

- `indexVersion = null` 触发 `apps/worker/workers/linkIndexing.ts` 将该 link 重新同步到 Meilisearch
- 索引版本号由常量 `MEILI_INDEX_VERSION` 控制，版本升级可触发全站全量重索引

---

## 九、多语言页面（i18n）对元数据抓取的影响

### 9.1 User Locale 字段

User 模型中有 `locale` 字段（默认 `"en"`）：
`packages/prisma/schema.prisma#L37` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/packages/prisma/schema.prisma#L37))

该字段**仅控制 Linkwarden 自身 UI 的显示语言**（通过 next-i18next）。

### 9.2 Worker 抓取时的 Locale

Worker 浏览器上下文：
`apps/worker/lib/browser.ts#L34-L51` ([GitHub](https://github.com/feng0732/linkwarden/blob/task-96/apps/worker/lib/browser.ts#L34-L51))
```js
export function getDefaultContextOptions(): BrowserContextOptions {
  const base: BrowserContextOptions = {
    ...devices["Desktop Chrome"],   // Playwright 内置 Desktop Chrome 设备配置
    ignoreHTTPSErrors: ...,
  };
  // 未设置 locale、extraHTTPHeaders(Accept-Language) 等用户级语言设置
  return base;
}
```

**关键结论**：Worker 不传递任何用户级 locale 信息。Playwright `devices["Desktop Chrome"]` 默认：
- `locale`：未显式指定时使用系统/浏览器默认（通常 `en-US`）
- `Accept-Language` 请求头：随浏览器默认 locale
- 不会因不同用户的 `user.locale` 而变化

### 9.3 对多语言站点的影响

对于根据 `Accept-Language` / Cookie / IP 返回不同语言内容的网站：

| 抓取项目 | 受影响程度 | 说明 |
|---|---|---|
| `<title>` 标题 | ⚠️ 高 | 抓取到服务器默认语言（通常英文） |
| `metaDescription` | ⚠️ 高 | 同上，AI 打标签使用默认语言描述 |
| 预览图（截图） | ⚠️ 高 | 页面文字为默认语言 |
| 可读存档 | ⚠️ 高 | 正文文本为默认语言 |
| Monolith 完整存档 | ⚠️ 高 | 捕获默认语言渲染结果 |
| Favicon | ✅ 无影响 | 按 origin 抓取，与语言无关 |

### 9.4 已知限制

当前架构不存在以下机制：
1. 读取用户 `locale` 并在 Playwright `browser.newContext({ locale: user.locale })` 传递
2. 设置 `Accept-Language: {user-locale}` 请求头
3. 检测 `<html lang="...">` 与用户 locale 不匹配时的告警或重试
4. 存储多语言版本的元数据

因此，所有用户的抓取结果**统一使用 Worker 浏览器的默认 locale**（一般为 `en-US`）。

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
       ├─► 写入 DB: lastPreserved=null
       │    preview=null, image=null, pdf=null, ...
       │
       ▼
Worker 轮询 (lastPreserved=null)
       │
       ├─► Playwright newContext()  ← 默认 locale，不传递用户语言
       │
       ├─► SSRF/协议检查失败?
       │    └─ 是 → 所有字段批量写 "unavailable", lastPreserved=now() ✋
       │
       ├─► determineLinkType() → HEAD content-type → url/pdf/image
       │
       ├─► page.goto(url, domcontentloaded)
       │       │
       │       ├─► meta[name="description"] → metaDescription (500 字符)
       │       │
       │       ├─► !link.preview? → handleArchivePreview
       │       │     ├─ 阶段 1: og:image → generatePreview
       │       │     │    ├─ 成功 → DB 写路径 ✓
       │       │     │    └─ 超限 → DB 写 "unavailable"
       │       │     └─ previewGenerated=false → 阶段 2: page.screenshot
       │       │          ├─ 成功 → 写路径 (可覆盖 "unavailable") ✓
       │       │          └─ 失败/超限 → finally 兜底
       │       │
       │       ├─► (可选) Readability / Screenshot/PDF / Monolith
       │       └─► (可选) Wayback Machine
       │
       ├─ finally:
       │    ├─ lastPreserved = now()  ← 停止轮询
       │    ├─ 重新读 DB → finalLink
       │    ├─ !finalLink.field ? "unavailable" : undefined
       │    └─ indexVersion = null (触发搜索索引)
       ▼
Favicon (UI 渲染时按需请求)
       │
       ├─► URL 规范化?
       │    └─ 否 → 308 重定向 (缓存 1 小时)
       │
       └─► GET 规范化 URL
            ├─► Google faviconV2 (1.5s) → 成功: 缓存 1 天 + immutable
            ├─► (失败) DuckDuckGo ip3 (1.5s) → 成功: 缓存 1 天 + immutable
            └─► (全部失败) 204 No Content → UI 占位图标 (缓存 1 小时)
```
