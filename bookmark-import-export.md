# Linkwarden 书签导入/导出迁移代码梳理

---

## 1. 总体架构

导入/导出功能围绕 `/api/v1/migration` 这个 API 路由展开，支持 5 种导入格式和一种 JSON 导出格式。

**关键文件：**

- API 路由入口：[migration/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/pages/api/v1/migration/index.ts)
- 前端 UI 入口：[ImportDropdown.tsx](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/components/ImportDropdown.tsx)
- 前端上传处理：[importBookmarks.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/client/importBookmarks.ts)
- 类型定义：[global.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/packages/types/global.ts#L139-L145)

**支持的导入格式（MigrationFormat 枚举）：**

```
linkwarden = 0  → Linkwarden 自有 JSON 备份
htmlFile   = 1  → Netscape 书签 HTML（Chrome/Firefox 等通用格式）
wallabag   = 2  → Wallabag JSON 导出
omnivore   = 3  → Omnivore metadata_*.json（打包为 zip）
pocket     = 4  → Pocket CSV 导出
```

**调用流程（代码顺序）：**

1. 用户点击 [ImportDropdown.tsx](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/components/ImportDropdown.tsx#L54-L77) 下拉菜单选择格式，触发隐藏 `<input type="file">`
2. 文件选择后调用 [importBookmarks.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/client/importBookmarks.ts#L23-L97) 的 `importBookmarks(e, format)`
3. 前端 `FileReader` 读取文件内容（Omnivore 走 `readAsArrayBuffer` + JSZip 解压，其余走 `readAsText`）
4. 以 `POST /api/v1/migration` 发送 `{ format, data }` 给后端
5. 路由 [migration/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/pages/api/v1/migration/index.ts#L61-L101) 根据 `format` 分发到具体导入控制器
6. GET `/api/v1/migration` 走导出分支，返回 backup.json 下载

---

## 2. 外部书签导入详细流程

### 2.1 HTML 书签导入（最复杂，含文件夹嵌套）

**控制器：** [importFromHTMLFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromHTMLFile.ts)

**执行顺序：**

1. **DOM 清洗** ([L14-L21](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L14-L21))：用 JSDOM 解析后，将 `<meta>`、`<META>`、`<P>` 标签的 outerHTML 替换为 innerHTML（消除干扰标签）。

2. **容量预检** ([L22-L32](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L22-L32))：统计所有 `<A>` 标签数量，调用 `hasPassedLimit(userId, totalImports)` 校验是否超出订阅配额，超出直接返回 400。

3. **Himalaya 解析为 AST** ([L34](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L34))：`parse(document.documentElement.outerHTML)` 把 HTML 转成 Node 树。

4. **DD 节点重构** — `processNodes()` ([L262-L300](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L262-L300))：
   - 遍历找到所有 `<DL>`
   - 若某个 `<DT>` 的下一个兄弟是 `<DD>`（描述），且 `<DT>` 内含 `<A>`，则把 `<DD>` 移入 `<A>` 的 children，然后从 `<DL>` 中删除原 `<DD>`
   - 这样后续递归时描述文本可以和链接一起被捕获

5. **递归处理** — `processBookmarks(userId, data, parentCollectionId?)` ([L47-L151](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L47-L151))：
   - 遇到 `<DT><H3>名称</H3>` → 创建/查找 collection，递归传入新的 `parentCollectionId`
   - 遇到 `<DT><A HREF="...">` → 解析 URL、名称、tags（属性 `tags` 按逗号分割）、`ADD_DATE`（Unix 秒转 Date）、`<DD>` 描述 → 调用 `createLink`
   - 若无 `parentCollectionId`（根级链接），先创建/复用名为 "Imports" 的 collection

### 2.2 Linkwarden 自有格式导入

**控制器：** [importFromLinkwarden.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromLinkwarden.ts)

- `JSON.parse(rawData)` 为 `Backup` 类型
- 统计 `data.collections[*].links.length` 做容量预检
- 用 `prisma.$transaction({ timeout: 30000 })` 包裹整批操作
- 遍历 collections：创建新 collection（名称/描述/颜色保留），再逐个创建 link + tags（connectOrCreate），最后匹配 `pinnedLinks` 中的 URL 标记为 pinned

### 2.3 Pocket 导入（CSV）

**控制器：** [importFromPocket.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromPocket.ts)

- PapaParse 解析 CSV（`header: true`），过滤掉无 `url` 的行
- 所有链接放入新建的 "Imports" collection
- 字段映射：
  - `title` → `name`
  - `url` → `url`
  - `time_added`（Unix 秒）→ `importDate`
  - `tags`（按 `|` 分割）→ `tags.connectOrCreate`

### 2.4 Wallabag 导入（JSON）

**控制器：** [importFromWallabag.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromWallabag.ts)

- 过滤无 `url` 的条目
- 所有链接放入新建的 "Imports" collection
- 字段映射：
  - `is_starred` → `pinnedBy.connect`
  - `title` → `name`
  - `content` → `textContent`（截断到 2047 字符）
  - `created_at` → `importDate`
  - `tags` 数组 → `tags.connectOrCreate`

### 2.5 Omnivore 导入（ZIP 内 JSON）

**控制器：** [importFromOmnivore.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromOmnivore.ts)
**前端 ZIP 预处理：** [importBookmarks.ts#processOmnivoreZipFile](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/client/importBookmarks.ts#L5-L21)

- 前端先解压 zip，读取所有 `metadata_*` 文件，合并 flatten 成单个数组
- 后端过滤无 `url` 的条目
- 所有链接放入新建的 "Omnivore Imports" collection
- 字段映射：
  - `title` → `name`
  - `description` → `description`
  - `thumbnail` → `image`
  - `savedAt` → `importDate`
  - `labels` → `tags.connectOrCreate`

---

## 3. Collection 映射逻辑

### 3.1 HTML 导入：同名 collection 复用（支持父子层级）

**函数：** `createCollection(userId, collectionName, parentId?)` ([importFromHTMLFile.ts#L153-L198](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L153-L198))

```
查找条件: { parentId, name: collectionName(trim+slice 254), ownerId: userId }
  命中 → 复用已有 id
  未命中 → prisma.collection.create，同时 createFolder(`archives/${id}`)
```

关键点：
- 名称做 `trim().slice(0, 254)` 规范化
- 父子层级通过 `parentId` 精确匹配（不同父级下允许同名 collection）
- 空文件夹名回退到 "Untitled Collection"
- 无父级的根级链接统一使用 "Imports" collection（同样走复用逻辑）

### 3.2 其他格式导入：新建单一 collection

| 格式 | Collection 名称 | 同名复用？ |
|------|----------------|-----------|
| Linkwarden | 按备份中每个 collection 名称逐一创建（总是新建，**不做同名复用**） | 否 |
| Pocket | "Imports" | 否（每次新建） |
| Wallabag | "Imports" | 否（每次新建） |
| Omnivore | "Omnivore Imports" | 否（每次新建） |

> 注意：Linkwarden 格式导入不做 collection 去重，每次导入都会新建所有 collection，即使同名已存在。这与 HTML 导入策略不同。

---

## 4. Tag 映射逻辑

### 4.1 统一机制：`connectOrCreate` + `name_ownerId` 复合唯一键

所有导入控制器（以及常规 [postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/links/postLink.ts#L120-L137)）都采用相同模式：

```javascript
tags: {
  connectOrCreate: tags.map((tag) => ({
    where: {
      name_ownerId: {        // Prisma 复合唯一索引
        name: tagName,
        ownerId: userId,
      },
    },
    create: {
      name: tagName,
      owner: { connect: { id: userId } },
    },
  })),
}
```

### 4.2 各格式 tag 来源差异

| 格式 | Tag 原始字段 | 分割方式 | 长度限制 |
|------|-------------|---------|---------|
| HTML | `<A>` 标签属性 `tags` | 逗号 `,` | 49 字符 |
| Linkwarden | `link.tags[*].name` | 已为数组 | 49 字符 |
| Pocket | CSV `tags` 列 | 竖线 `\|` | 50 字符（slice 0,50） |
| Wallabag | `tags` 数组 | 已为数组 | 49 字符 |
| Omnivore | `labels` 数组 | 已为数组 | 49 字符 |

所有 tag 名称统一执行 `trim()` 后再截断。

---

## 5. 重复链接处理

### 5.1 手动新建链接的去重（`preventDuplicateLinks`）

**位置：** [postLink.ts#L47-L67](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/links/postLink.ts#L47-L67)

仅当用户设置 `preventDuplicateLinks = true` 时生效：
1. URL `trim` 并去掉末尾 `/`
2. 同时查询带 `www.` 和不带 `www.` 两种形式
3. 查询范围限定在该用户拥有的 collection 内
4. 命中则返回 `409 "Link already exists"`

### 5.2 导入期间的重复链接处理

**所有导入控制器均未启用 URL 去重检查。** 具体行为：

- HTML 导入的 `createLink()` ([importFromHTMLFile.ts#L200-L260](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L200-L260)) 直接 `prisma.link.create`，没有 `findFirst` 查重
- Linkwarden / Pocket / Wallabag / Omnivore 的导入同样直接 `prisma.link.create`
- 因此：
  - 导入相同文件两次 → 产生重复链接
  - URL 已存在于用户库中 → 仍会新增一条

### 5.3 URL 有效性校验

所有导入器对每条链接都执行：
```javascript
try { new URL(url.trim()); } catch (e) { return/continue; }
```
无效 URL 静默跳过，不报错，不计入失败反馈。

### 5.4 字段长度截断

| 字段 | 最大长度 |
|------|---------|
| collection.name | 254 |
| collection.description | 254 |
| collection.color | 50 |
| link.url | 2047 |
| link.name | 254 |
| link.description | 254（HTML/Wallabag 的 textContent 为 2047） |
| tag.name | 49~50 |

超长字段统一 `.slice()` 静默截断，不抛错。

---

## 6. 导出数据结构

### 6.1 导出控制器

**位置：** [exportData.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/exportData.ts)

查询语句：
```javascript
prisma.user.findUnique({
  where: { id: userId },
  include: {
    collections: {
      include: {
        rssSubscriptions: true,
        links: {
          omit: { textContent, preview, image, readable, monolith, pdf },
          include: { tags: true },
        },
      },
    },
    pinnedLinks: true,
  },
});
```

导出时剔除 `password` 和 `id` 字段：
```javascript
const { password, id, ...userData } = user;
```

响应头设置为下载：
```
Content-Type: application/json
Content-Disposition: attachment; filename=backup.json
```

### 6.2 导出类型定义

**位置：** [global.ts#Backup](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/packages/types/global.ts#L125-L132)

```typescript
interface CollectionIncludingLinks extends Collection {
  links: LinksIncludingTags[];  // links + tags[]
}

interface Backup extends Omit<User, "password" | "id"> {
  collections: CollectionIncludingLinks[];
  pinnedLinks: LinksIncludingTags[];
}
```

**导出 JSON 结构示意：**

```json
{
  "username": "...",
  "email": "...",
  "collections": [
    {
      "id": 1,
      "name": "Articles",
      "description": "...",
      "color": "...",
      "ownerId": 123,
      "parentId": null,
      "createdAt": "...",
      "rssSubscriptions": [...],
      "links": [
        {
          "id": 10,
          "url": "https://example.com",
          "name": "...",
          "description": "...",
          "type": 0,
          "importDate": "...",
          "collectionId": 1,
          "tags": [
            { "id": 5, "name": "tech", "ownerId": 123 }
          ]
        }
      ]
    }
  ],
  "pinnedLinks": [
    { "id": 10, "url": "...", "tags": [...] }
  ]
}
```

**注意事项：**
- `links` 中故意省略了大体积字段：`textContent`、`preview`、`image`、`readable`、`monolith`、`pdf`（减小备份文件体积）
- `pinnedLinks` 为独立扁平数组，Linkwarden 导入时通过匹配 `url` 与新建链接关联并设置 pinned

---

## 7. 部分失败与进度反馈

### 7.1 前端进度反馈

**文件：** [importBookmarks.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/client/importBookmarks.ts)

使用 `react-hot-toast` 的三步反馈：

| 阶段 | 调用 | 效果 |
|------|------|------|
| 开始上传 | `toast.loading("Importing...")` | 显示加载 toast，保存 id `load` |
| 成功 | `toast.dismiss(load)` + `toast.success("Imported the Bookmarks! Reloading the page...")` + `setTimeout(location.reload, 2000)` | 2 秒后刷新页面 |
| 失败 | `toast.dismiss(load)` + `toast.error(...)` | 显示具体错误 |

**前端捕获的错误类型：**
- ZIP 解析失败（Omnivore）
- HTTP 响应非 2xx：读取 `errorData.response` 作为错误消息
- 网络异常 / fetch 抛错
- FileReader 读文件错误

### 7.2 后端整体失败（请求级）

在 [migration/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/pages/api/v1/migration/index.ts) 中：

| 场景 | HTTP 状态 | 响应消息 |
|------|----------|---------|
| body 超过 `IMPORT_LIMIT`（默认 10MB） | 413 | `Import file exceeds the XMB size limit.` |
| body JSON 解析失败 | 400 | `Invalid request body provided.` |
| 演示模式（`NEXT_PUBLIC_DEMO=true`） | 400 | `This action is disabled...` |
| 用户未通过 verifyUser | 401 | - |
| 超出链接配额 | 400 | `Your subscription has reached the maximum number of links allowed.` |

### 7.3 后端部分失败（单条记录级）

**这是当前实现的关键特性：部分失败不影响整体导入，也不向客户端报告。**

**单条静默跳过的场景：**
1. URL 解析失败（`new URL()` throw）→ `continue` / `return`，不记录任何日志
2. 字段超长 → 静默 `.slice()` 截断
3. 空 tag 名称 → 仍会被创建（空字符串）

**事务策略差异：**

| 格式 | 是否包裹事务 | 超时 | 行为 |
|------|------------|------|------|
| HTML | **否** | - | 逐条创建，某条失败不影响已成功的，但后续可能中断（取决于失败位置） |
| Linkwarden | 是 | 30s | 全部成功或全部回滚；`.catch(err => console.log(err))` 吞异常后仍返回 200 |
| Pocket | 是 | 30s | 同上 |
| Wallabag | 是 | 30s | 同上 |
| Omnivore | 是 | 30s | `.catch(err => { console.error; throw err })` 会抛出，导致返回 500 |

> **关键点**：Linkwarden/Pocket/Wallabag 三个控制器在 `$transaction` 的 `.catch` 中只 `console.log(err)`，然后函数继续返回 `{ response: "Success.", status: 200 }`。这意味着即使事务整体回滚、一条都没导入，前端也会收到 "成功" 的假阳性反馈。只有 Omnivore 会 re-throw 让外层返回 500 错误。

### 7.4 容量校验细节

**函数：** [hasPassedLimit(userId, numberOfImports)](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/packages/lib/verifyCapacity.ts#L8-L109)

执行顺序：
1. 未启用 Stripe：`MAX_LINKS_PER_USER`（默认 30000）- (现有链接数 + 待导入数) < 0
2. 启用 Stripe 且在试用期（无需 CC）：同上
3. 付费用户：
   - 无有效 `subscriptionId` 或 `quantity` → 视为超限
   - 家庭/组织订阅：`quantity * MAX_LINKS_PER_USER` 作为总容量，统计组织内所有用户的链接
   - 个人订阅：按单用户 30000 × quantity 计算

容量检查发生在**导入开始前**，但导入过程中可能有并发写入，存在竞态窗口。

---

## 8. 测试覆盖

**文件：** [importFromHTMLFile.test.ts](file:///d:/fz/0601/solo-dogfeeding/code/89-linkwarden/apps/web/lib/api/controllers/migration/importFromHTMLFile.test.ts)

已覆盖的测试场景（vitest + 真实 Prisma）：

1. 超出链接限额时返回 400 且不写入数据
2. 根级链接自动归入 "Imports" collection，正确导入 tags/ADD_DATE/描述（含 HTML 实体解码 `&amp;` → `&`）
3. 嵌套 collection（Recipes → Desserts）正确创建父子层级并分配链接
4. 已存在 "Imports" collection 时复用，不创建重复
5. 空文件夹名回退为 "Untitled Collection"
6. 无效 URL 静默跳过，仅有效 URL 入库

另有一段按 importDate 排序 ID 的测试被注释掉（`sortBookmarksTreeByEffectiveDate` 整个函数也被注释），当前未启用。
