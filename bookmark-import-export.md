# Linkwarden 书签导入/导出迁移代码梳理

---

## 1. 总体架构

导入/导出功能围绕 `/api/v1/migration` 这个 API 路由展开，支持 5 种导入格式和一种 JSON 导出格式。

**关键文件：**

- API 路由入口：[migration/index.ts](apps/web/pages/api/v1/migration/index.ts)
- 前端 UI 入口：[ImportDropdown.tsx](apps/web/components/ImportDropdown.tsx)
- 前端上传处理：[importBookmarks.ts](apps/web/lib/client/importBookmarks.ts)
- 类型定义：[global.ts](packages/types/global.ts#L139-L145)

**支持的导入格式（MigrationFormat 枚举）：**

```
linkwarden = 0  → Linkwarden 自有 JSON 备份
htmlFile   = 1  → Netscape 书签 HTML（Chrome/Firefox 等通用格式）
wallabag   = 2  → Wallabag JSON 导出
omnivore   = 3  → Omnivore metadata_*.json（打包为 zip）
pocket     = 4  → Pocket CSV 导出
```

**调用流程（代码顺序）：**

1. 用户点击 [ImportDropdown.tsx](apps/web/components/ImportDropdown.tsx#L54-L77) 下拉菜单选择格式，触发隐藏 `<input type="file">`
2. 文件选择后调用 [importBookmarks.ts](apps/web/lib/client/importBookmarks.ts#L23-L97) 的 `importBookmarks(e, format)`
3. 前端 `FileReader` 读取文件内容（Omnivore 走 `readAsArrayBuffer` + JSZip 解压，其余走 `readAsText`）
4. 以 `POST /api/v1/migration` 发送 `{ format, data }` 给后端
5. 路由 [migration/index.ts](apps/web/pages/api/v1/migration/index.ts#L61-L101) 根据 `format` 分发到具体导入控制器
6. GET `/api/v1/migration` 走导出分支，返回 backup.json 下载

---

## 2. 外部书签导入详细流程

### 2.1 HTML 书签导入（最复杂，含文件夹嵌套）

**控制器：** [importFromHTMLFile.ts](apps/web/lib/api/controllers/migration/importFromHTMLFile.ts)

**执行顺序：**

1. **DOM 清洗** ([L14-L21](apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L14-L21))：用 JSDOM 解析后，将 `<meta>`、`<META>`、`<P>` 标签的 outerHTML 替换为 innerHTML（消除干扰标签）。

2. **容量预检** ([L22-L32](apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L22-L32))：统计所有 `<A>` 标签数量，调用 `hasPassedLimit(userId, totalImports)` 校验是否超出订阅配额，超出直接返回 400。

3. **Himalaya 解析为 AST** ([L34](apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L34))：`parse(document.documentElement.outerHTML)` 把 HTML 转成 Node 树。

4. **DD 节点重构** — `processNodes()` ([L262-L300](apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L262-L300))：
   - 遍历找到所有 `<DL>`
   - 若某个 `<DT>` 的下一个兄弟是 `<DD>`（描述），且 `<DT>` 内含 `<A>`，则把 `<DD>` 移入 `<A>` 的 children，然后从 `<DL>` 中删除原 `<DD>`
   - 这样后续递归时描述文本可以和链接一起被捕获

5. **递归处理** — `processBookmarks(userId, data, parentCollectionId?)` ([L47-L151](apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L47-L151))：
   - 遇到 `<DT><H3>名称</H3>` → 创建/查找 collection，递归传入新的 `parentCollectionId`
   - 遇到 `<DT><A HREF="...">` → 解析 URL、名称、tags（属性 `tags` 按逗号分割）、`ADD_DATE`（Unix 秒转 Date）、`<DD>` 描述 → 调用 `createLink`
   - 若无 `parentCollectionId`（根级链接），先创建/复用名为 "Imports" 的 collection

### 2.2 Linkwarden 自有格式导入

**控制器：** [importFromLinkwarden.ts](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts)

- `JSON.parse(rawData)` 为 `Backup` 类型
- 统计 `data.collections[*].links.length` 做容量预检
- 形式上用 `prisma.$transaction(async () => {...}, { timeout: 30000 })` 包裹，但回调内所有操作均使用全局 `prisma` 客户端而非事务参数 `tx`，**事务实际上不生效**（详见 7.4.1）
- 遍历 collections：创建新 collection（仅恢复 name/description/color 三个字段），再逐个创建 link + tags（connectOrCreate），最后尝试匹配 `pinnedLinks` 中的 URL 标记为 pinned
- **不会恢复**：collection 父子层级（parentId）、RSS 订阅、icon/iconWeight/isPublic 等 collection 字段、旧 id 到新 id 的映射（详见 6.3）

### 2.3 Pocket 导入（CSV）

**控制器：** [importFromPocket.ts](apps/web/lib/api/controllers/migration/importFromPocket.ts)

- PapaParse 解析 CSV（`header: true`），过滤掉无 `url` 的行
- 所有链接放入新建的 "Imports" collection
- 字段映射：
  - `title` → `name`
  - `url` → `url`
  - `time_added`（Unix 秒）→ `importDate`
  - `tags`（按 `|` 分割）→ `tags.connectOrCreate`

### 2.4 Wallabag 导入（JSON）

**控制器：** [importFromWallabag.ts](apps/web/lib/api/controllers/migration/importFromWallabag.ts)

- 过滤无 `url` 的条目
- 所有链接放入新建的 "Imports" collection
- 字段映射：
  - `is_starred` → `pinnedBy.connect`
  - `title` → `name`
  - `content` → `textContent`（截断到 2047 字符）
  - `created_at` → `importDate`
  - `tags` 数组 → `tags.connectOrCreate`

### 2.5 Omnivore 导入（ZIP 内 JSON）

**控制器：** [importFromOmnivore.ts](apps/web/lib/api/controllers/migration/importFromOmnivore.ts)
**前端 ZIP 预处理：** [importBookmarks.ts#processOmnivoreZipFile](apps/web/lib/client/importBookmarks.ts#L5-L21)

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

**函数：** `createCollection(userId, collectionName, parentId?)` ([importFromHTMLFile.ts#L153-L198](apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L153-L198))

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

所有导入控制器（以及常规 [postLink.ts](apps/web/lib/api/controllers/links/postLink.ts#L120-L137)）都采用相同模式：

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

### 4.2 Tag 名称的 trim/slice 操作顺序（深度分析）

**各导入器 where 与 create 的操作顺序对比：**

| 导入器 | where 条件（查找已有 tag） | create 时（新建 tag） | 顺序一致？ |
|--------|--------------------------|---------------------|-----------|
| [HTML](apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L216) | `tag.trim()` — L241 | `tag.trim()` — L246 | ✅ 一致 |
| [Linkwarden](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L82-L97) | `tag.name?.slice(0, 49)` — L85 **（只 slice，不 trim）** | `tag.name?.trim().slice(0, 49)` — L90 | ❌ **不一致** |
| [Pocket](apps/web/lib/api/controllers/migration/importFromPocket.ts#L79-L91) | `tag?.slice(0, 50).trim()` — L83 | `tag?.slice(0, 50).trim()` — L88 | ✅ 一致 |
| [Wallabag](apps/web/lib/api/controllers/migration/importFromWallabag.ts#L99-L114) | `tag?.trim().slice(0, 49)` — L102 | `tag?.trim().slice(0, 49)` — L107 | ✅ 一致 |
| [Omnivore](apps/web/lib/api/controllers/migration/importFromOmnivore.ts#L89-L104) | `label?.trim().slice(0, 49)` — L92 | `label?.trim().slice(0, 49)` — L97 | ✅ 一致 |
| [postLink](apps/web/lib/api/controllers/links/postLink.ts#L120-L137) | `tag.name.trim()` — L124 | `tag.name.trim()` — L129 | ✅ 一致（无 slice） |

**补充：HTML 导入有预处理步骤** [importFromHTMLFile.ts#L216](apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L216)：

```javascript
tags = tags?.map((tag) => tag.trim().slice(0, 49));  // 预处理：先 trim 再 slice
```

然后 where/create 中再执行 `tag.trim()`（不再 slice），实际效果等价于 `trim().slice(0,49).trim()`。

**不一致的风险分析 — Linkwarden 导入器：**

在 [importFromLinkwarden.ts#L82-L97](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L82-L97)：

```javascript
tags: {
  connectOrCreate: link.tags.map((tag) => ({
    where: {
      name_ownerId: {
        name: tag.name?.slice(0, 49),           // ❌ 只 slice，不 trim
        ownerId: userId,
      },
    },
    create: {
      name: tag.name?.trim().slice(0, 49),      // ✅ 先 trim 再 slice
      owner: { connect: { id: userId } },
    },
  })),
}
```

问题路径：
1. 备份文件中 tag 名为 `" tech "`（前后有空格，长度 6）
2. `where` 查找：`slice(0,49)` 后仍为 `" tech "`（因为 6 < 49），**查找的是带空格的 " tech "**
3. 若数据库中已有标准化 tag `"tech"`（trim 后），where 条件匹配失败
4. 触发 create 分支：`trim().slice(0,49)` → `"tech"`
5. 由于 `name_ownerId` 唯一约束，尝试创建已存在的 `"tech"`，**抛出唯一约束冲突错误**
6. 整个事务回滚 → 所有 collection/link/tag 都不入库，但由于 catch 吞异常，前端仍收到 200 成功

**Pocket 导入器的顺序注意：**

在 [importFromPocket.ts#L79-L91](apps/web/lib/api/controllers/migration/importFromPocket.ts#L79-L91)：

```javascript
name: tag?.slice(0, 50).trim()  // 先 slice 再 trim
```

先 slice 再 trim 的潜在问题：如果原 tag 是 `"  verylongname...(超过50字符)...  "`，先 slice 到 50 可能把末尾空格切掉，然后 trim 后长度可能更短，且如果第 50 个字符刚好是空格的一部分，结果与先 trim 再 slice 可能不同。

**各格式 tag 来源差异：**

| 格式 | Tag 原始字段 | 分割方式 | 长度限制 |
|------|-------------|---------|---------|
| HTML | `<A>` 标签属性 `tags` | 逗号 `,` | 49 字符 |
| Linkwarden | `link.tags[*].name` | 已为数组 | 49 字符 |
| Pocket | CSV `tags` 列 | 竖线 `\|` | 50 字符（slice 0,50） |
| Wallabag | `tags` 数组 | 已为数组 | 49 字符 |
| Omnivore | `labels` 数组 | 已为数组 | 49 字符 |

---

## 5. 重复链接处理

### 5.1 手动新建链接的去重（`preventDuplicateLinks`）

**位置：** [postLink.ts#L47-L67](apps/web/lib/api/controllers/links/postLink.ts#L47-L67)

仅当用户设置 `preventDuplicateLinks = true` 时生效：
1. URL `trim` 并去掉末尾 `/`
2. 同时查询带 `www.` 和不带 `www.` 两种形式
3. 查询范围限定在该用户拥有的 collection 内
4. 命中则返回 `409 "Link already exists"`

### 5.2 导入期间的重复链接处理

**所有导入控制器均未启用 URL 去重检查。** 具体行为：

- HTML 导入的 `createLink()` ([importFromHTMLFile.ts#L200-L260](apps/web/lib/api/controllers/migration/importFromHTMLFile.ts#L200-L260)) 直接 `prisma.link.create`，没有 `findFirst` 查重
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
| link.description | 254（Wallabag 的 textContent 为 2047） |
| tag.name | 49~50 |

超长字段统一 `.slice()` 静默截断，不抛错。

---

## 6. 导出数据结构

### 6.1 导出控制器

**位置：** [exportData.ts](apps/web/lib/api/controllers/migration/exportData.ts)

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

**位置：** [global.ts#Backup](packages/types/global.ts#L125-L132)

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
- `collections.links` 显式 omit 了大体积字段：`textContent`、`preview`、`image`、`readable`、`monolith`、`pdf`，并 include 了 `tags`
- `pinnedLinks: true` 是 Prisma 简写，默认返回所有 Link 标量字段（**包含**上述 6 个大字段），但**不 include 任何关联**（包括 tags）
- `pinnedLinks` 为独立扁平数组，Linkwarden 导入时尝试通过匹配 `url` 与新建链接关联并设置 pinned

### 6.3 导出/导入不对称与字段丢失分析（自有格式）

导出时通过深度 include 取出了完整的用户数据结构，但导入时仅恢复了其中很小一部分。以下是逐项对比：

#### 6.3.1 Collection 字段丢失

Collection 模型在 Prisma schema 中定义了以下字段 [schema.prisma#L126-L149](packages/prisma/schema.prisma#L126-L149)：

| 字段 | 导出时包含？ | 导入时恢复？ | 备注 |
|------|------------|------------|------|
| `name` | ✅ 是 | ✅ 是 | `e.name?.trim().slice(0, 254)` |
| `description` | ✅ 是 | ✅ 是 | `e.description?.trim().slice(0, 254)` |
| `color` | ✅ 是 | ✅ 是 | `e.color?.trim().slice(0, 50)` |
| `parentId` | ✅ 是（Backup 继承 Collection） | ❌ **否** | 父子层级完全丢失，所有 collection 均为顶级 |
| `icon` | ✅ 是 | ❌ **否** | 自定义图标丢失 |
| `iconWeight` | ✅ 是 | ❌ **否** | 图标粗细丢失 |
| `isPublic` | ✅ 是 | ❌ **否** | 公开/私有状态丢失，默认 private |
| `ownerId` | ✅ 是 | ✅ 是（重新关联当前 userId） | 但值是新的，非备份中旧值 |
| `createdById` | ✅ 是 | ✅ 是（重新关联当前 userId） | 同上 |
| `createdAt` | ✅ 是 | ❌ **否** | 使用数据库 `now()` |
| `updatedAt` | ✅ 是 | ❌ **否** | 使用数据库 `now()` |
| `rssSubscriptions` | ✅ 是（显式 include） | ❌ **否** | RSS 订阅完全丢失（见 6.3.2） |
| `members` | ❌ 否（未 include） | ❌ 否 | 导出时就不包含 |
| `links` | ✅ 是（显式 include） | ✅ 是 | 逐条重建 |

**关于 `parentId`（父子层级丢失）的具体代码**：
- 导入 collection 的 create 语句仅设置了 5 个字段 [importFromLinkwarden.ts#L34-L50](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L34-L50)：
  ```javascript
  data: {
    owner: { connect: { id: userId } },
    name: e.name?.trim().slice(0, 254),
    description: e.description?.trim().slice(0, 254),
    color: e.color?.trim().slice(0, 50),
    createdBy: { connect: { id: userId } },
  }
  ```
- **没有** `parent: { connect: ... }` 或 `parentId: ...`
- 即使想设置也有两个障碍：
  1. 没有建立 **旧 collection.id → 新 collection.id** 的映射表，无法知道 parentId 对应的新 id
  2. 导入顺序是扁平遍历 `data.collections`，父 collection 可能在子 collection 之后创建（取决于 JSON 中数组顺序）

#### 6.3.2 RSS 订阅完全丢失

- 导出时显式 `include: { rssSubscriptions: true }` [exportData.ts#L8-L9](apps/web/lib/api/controllers/migration/exportData.ts#L8-L9)，backup.json 中每个 collection 都带有 `rssSubscriptions` 数组
- RSS 模型字段：`id, url, name, lastBuildDate, collectionId, ownerId, createdAt, updatedAt` [schema.prisma#L248-L258](packages/prisma/schema.prisma#L248-L258)
- 导入控制器 `importFromLinkwarden.ts` **完全没有处理 `rssSubscriptions`** 的代码——零行
- 结果：备份中的 RSS 订阅静默丢失，用户恢复数据后需要重新手动添加 RSS 源

#### 6.3.3 pinnedLinks 导出结构与导入匹配问题（深度对比）

**两个查询的配置差异** [exportData.ts#L10-L25](apps/web/lib/api/controllers/migration/exportData.ts#L10-L25)：

```javascript
collections: {
  include: {
    rssSubscriptions: true,
    links: {
      omit: {                      // ↓ 显式排除 6 个大字段
        textContent: true,
        preview: true,
        image: true,
        readable: true,
        monolith: true,
        pdf: true,
      },
      include: { tags: true },     // ← 显式 include tags 关联
    },
  },
},
pinnedLinks: true,                 // ← Prisma 简写，无 omit，无 include
```

**字段差异对照表（Link 模型共 29 个字段/关联）**：

| 类别 | 字段/关联 | `collections.links` | `pinnedLinks: true` |
|------|----------|-------------------|-------------------|
| **标量（ID）** | `id` | ✅ | ✅ |
| **标量（基础）** | `name`, `type`, `description`, `url`, `color`, `icon`, `iconWeight`, `clientSide`, `aiTagged`, `metaDescription`, `indexVersion`, `importDate`, `createdAt`, `updatedAt`, `lastPreserved`, `createdById`, `collectionId` | ✅ 全部 | ✅ 全部 |
| **大体积标量** | `textContent`, `preview`, `image`, `readable`, `monolith`, `pdf` | ❌ **omit 排除**（减小备份） | ✅ **完整包含**（可能数 MB/条） |
| **关联** | `tags`（Tag[]） | ✅ **显式 include** | ❌ **不返回**（默认不加载关联） |
| **关联** | `highlight`, `pinnedBy`, `collection`, `createdBy` | ❌ 不 include | ❌ 不 include |

**关键差异 1：pinnedLinks 包含完整大字段但不含 tags**

这是 `pinnedLinks: true`（Prisma 默认行为）与 `collections.links`（手动配置）最核心的不同：

- `collections.links` 经过精心优化：omit 6 个可能达数 MB 的归档字段，include 了 tags（用于恢复标签）
- `pinnedLinks: true` 是"原样全拿"——所有标量字段原样返回（包括 textContent/preview 等大 BLOB），但 tags 等关联完全不加载

**对备份体积的影响**：假设用户置顶了 100 条链接，每条 textContent 平均 100KB，则 pinnedLinks 数组就额外膨胀约 **10MB**，而同样这些链接在 collections.links 中仅约数十 KB。

**关键差异 2：TypeScript 类型定义与实际运行时不一致**

类型声明 [global.ts#L129-L132](packages/types/global.ts#L129-L132)：
```typescript
export interface Backup extends Omit<User, "password" | "id"> {
  collections: CollectionIncludingLinks[];
  pinnedLinks: LinksIncludingTags[];   // ← 声称 tags: Tag[]
}

export interface LinksIncludingTags extends Link {
  tags: Tag[];   // ← 声明 tags 必存在
}
```

但实际运行时，由于 Prisma 查询 `pinnedLinks: true` 没有 `include: { tags: true }`，返回的 pinnedLinks 数组中**每个对象都没有 `tags` 字段**。TypeScript 类型声称有 `tags: Tag[]`，运行时值为 `undefined`——这是一个编译期类型谎言。

**对 URL 匹配的影响**：

URL 匹配不依赖 tags 或大字段，所以 pinnedLinks 导出结构对 `pinnedLink.url === newLink.url` 比较**没有直接负面影响**——`url` 是 Link 的标量字段，两个查询都会返回。

但存在一个间接风险：如果未来某条 pinned 链接的 URL 恰好为 `null`（Link 模型中 `url` 是 `String?` 可选），而对应 link 在 collections.links 中创建时 URL 被其他逻辑填充（或反之），也会导致匹配不上。

**对旧 ID → 新 ID 映射的影响**：

pinnedLinks 导出时**完整包含旧 `id`**（Link.id 是标量字段，`pinnedLinks: true` 必然返回）。这意味着：

```jsonc
// pinnedLinks[i] 实际导出的结构（注意：没有 tags，但有完整 id/url/collectionId）
{
  "id": 42,                 // ✅ 旧 link.id —— 精确映射的关键！
  "url": "https://example.com/article",
  "collectionId": 5,        // ✅ 旧 collection.id
  "name": "Article Title",
  "type": "url",
  "description": "...",
  "textContent": "<html>完整页面内容...</html>",  // ✅（意外包含，本应 omit）
  "preview": "base64...",   // ✅（意外包含）
  "image": "base64...",     // ✅（意外包含）
  // "tags": [...]          // ❌ 不存在（类型定义说谎）
  // highlight, pinnedBy    // ❌ 不存在
}
```

导出时**拥有完美的精确关联信息**（`pinnedLinks[i].id` 与 `collections[j].links[k].id` 是同一旧数据库主键），但导入时完全未利用：
- 导入代码不建立任何 旧 id → 新 id 的映射表
- 退而求其次使用 URL 模糊匹配

这不是"信息不足"，而是"信息浪费"——精确匹配所需的所有数据都在 backup.json 里，只是代码没读。

**对置顶恢复判断的影响**（导出结构 × 导入逻辑）：

| 置顶恢复要素 | 导出是否提供 | 导入是否利用 | 结果 |
|------------|------------|------------|------|
| 旧 link.id（精确唯一键） | ✅ pinnedLinks[i].id | ❌ 完全忽略 | 浪费精确关联机会 |
| 旧 collection.id（辅助去重） | ✅ pinnedLinks[i].collectionId | ❌ 完全忽略 | 同 URL 跨集合时无法区分，pinned 状态误扩散 |
| URL（模糊匹配键） | ✅ pinnedLinks[i].url（原值） | ⚠️ 使用但与 newLink.url（trim+slice）做 === | 空格/超长导致不匹配 |
| tags（辅助匹配） | ❌ 运行时不存在（类型声称有） | ❌ 未利用 | — |
| 大字段（textContent 等） | ✅ 完整包含（本应 omit） | ❌ 未利用 | 徒增备份体积 |

**导入匹配方式** [importFromLinkwarden.ts#L101-L113](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L101-L113)：

```javascript
// 对每个新创建的 link，遍历整个 pinnedLinks 数组做 URL 字符串精确匹配
data?.pinnedLinks.forEach(async (pinnedLink) => {
  if (pinnedLink.url === newLink.url) {  // ❌ 只比较 URL，不利用 id/collectionId
    await prisma.link.update({ ... });    // 设置 pinnedBy
  }
});
```

**不匹配风险场景**：
1. 同一用户在两个不同 collection 中收藏了相同 URL，其中只有一个被 pinned → 导入时**两个新链接都会被标记为 pinned**（pinnedLinks 有 collectionId 但未用）
2. URL 在导出后导入前被规范化（如 `link.url.trim().slice(0,2047)` 与备份中的 `pinnedLink.url` 原值有空格/长度差异）→ **完全匹配不上**，pinned 状态丢失
3. pinnedLinks 数组中的 link.id 可以与 `collections[*].links[*].id` 精确配对 → 但代码没有建立 旧 id → 新 id 的映射表，浪费了精确关联的机会

---

## 7. 部分失败与进度反馈

### 7.1 前端进度反馈

**文件：** [importBookmarks.ts](apps/web/lib/client/importBookmarks.ts)

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

在 [migration/index.ts](apps/web/pages/api/v1/migration/index.ts) 中：

| 场景 | HTTP 状态 | 响应消息 |
|------|----------|---------|
| body 超过 `IMPORT_LIMIT`（默认 10MB） | 413 | `Import file exceeds the XMB size limit.` |
| body JSON 解析失败 | 400 | `Invalid request body provided.` |
| 演示模式（`NEXT_PUBLIC_DEMO=true`） | 400 | `This action is disabled...` |
| 用户未通过 verifyUser | 401 | - |
| 超出链接配额 | 400 | `Your subscription has reached the maximum number of links allowed.` |

### 7.3 后端部分失败（单条记录级）

**单条静默跳过的场景：**
1. URL 解析失败（`new URL()` throw）→ `continue` / `return`，不记录任何日志
2. 字段超长 → 静默 `.slice()` 截断
3. 空 tag 名称 → 仍会被创建（空字符串）

### 7.4 事务失败后成功返回与置顶关系入库不一致（深度分析）

#### 7.4.1 事务策略与 `tx` 客户端缺失问题（核心修正）

**Prisma 交互式事务的正确用法**：`prisma.$transaction(async (tx) => { tx.model.create(...) })`，必须使用回调参数 `tx` 执行操作，才能纳入同一数据库事务。

**所有 4 个使用了 `$transaction` 的导入器（Linkwarden/Pocket/Wallabag/Omnivore）都犯了同一个错误**：回调签名为 `async () => {...}`（**无 `tx` 参数**），内部全部使用全局 `prisma` 客户端执行语句。

后果：
- 每一条 `prisma.collection.create` / `prisma.link.create` / `prisma.link.update` 都在**各自独立的自动提交事务**中执行
- 回调抛异常时，**此前已成功执行的语句不会回滚**（因为已经各自提交）
- `{ timeout: 30000 }` 只限制回调函数本身的执行时长，不提供任何原子性保证
- 因此"全部成功或全部回滚"是**假象**——实际行为是"逐条写入、中途失败留下半成品"

| 格式 | 是否调用 $transaction | 内部使用 tx？ | 实际原子性 | catch 行为 | 最终返回 |
|------|---------------------|-------------|-----------|-----------|---------|
| HTML | **否** | - | 无 | 无 | 逐条创建，中途异常抛到顶层 |
| [Linkwarden](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L27-L29) | 是 | **否，用全局 prisma** | **无（伪事务）** | `console.log(err)` 吞掉 | **始终 200** |
| [Pocket](apps/web/lib/api/controllers/migration/importFromPocket.ts) | 是 | **否，用全局 prisma** | **无（伪事务）** | `console.log(err)` 吞掉 | **始终 200** |
| [Wallabag](apps/web/lib/api/controllers/migration/importFromWallabag.ts) | 是 | **否，用全局 prisma** | **无（伪事务）** | `console.log(err)` 吞掉 | **始终 200** |
| [Omnivore](apps/web/lib/api/controllers/migration/importFromOmnivore.ts) | 是 | **否，用全局 prisma** | **无（伪事务）** | `console.error + throw err` 重抛 | 失败时 500 |

> **修正认知**：HTML 导入与 Linkwarden/Pocket/Wallabag/Omnivore 导入的实际数据一致性行为**没有区别**——都是非事务性的逐条写入。唯一区别是后 4 者有一个无效的 `$transaction` 外壳和吞异常的 `.catch()`。

#### 7.4.2 路径一：异常被 catch 吞掉仍返回 200（假阳性）

以 [importFromLinkwarden.ts#L119-L121](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L119-L121) 为例：

```javascript
await prisma
  .$transaction(async () => {
    // ... 使用全局 prisma 逐条创建 collections、links、tags ...
    // 注意：即使某条抛异常，之前已执行的语句也不会回滚
  }, { timeout: 30000 })
  .catch((err) => console.log(err));   // ⚠️ catch 仅打印，不向上传递

return { response: "Success.", status: 200 };  // ⚠️ 无论是否异常都执行
```

**可能触发异常的原因：**
- 30 秒超时（`timeout: 30000`，仅限制回调时长）
- 数据库唯一约束冲突（如前述 Linkwarden tag where/create 不一致导致）
- 数据库连接中断
- Prisma 客户端错误
- `createFolder` 文件系统操作失败

**后果：**
- 已成功执行的 create/update **不会回滚**（伪事务），数据库中留下部分已导入的数据
- 但控制器返回 `status: 200`，前端 `toast.success("Imported the Bookmarks!")`
- 2 秒后页面刷新，用户看到部分数据但可能以为是全部导入成功

Pocket 和 Wallabag 导入器存在完全相同的问题。

#### 7.4.3 路径二：Linkwarden pinnedLinks 异步更新未等待 + 脱离伪事务

**代码位置：** [importFromLinkwarden.ts#L101-L113](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L101-L113)

```javascript
// Import pinnedLinks
data?.pinnedLinks.forEach(async (pinnedLink) => {    // ⚠️ forEach + async 回调
  if (pinnedLink.url === newLink.url) {
    await prisma.link.update({                        // ⚠️ 全局 prisma 客户端
      where: { id: newLink.id },
      data: { pinnedBy: { connect: { id: userId } } },
    });
  }
});
// ⚠️ 没有 await Promise.all(...)，forEach 立即返回
```

**问题详解：**

1. **forEach + async = 悬空 Promise**：
   - `Array.prototype.forEach` 是同步函数，传入 `async` 回调时不收集 Promise 也不 `await`
   - 每个 `prisma.link.update` 在后台独立执行，事务回调在它们完成前就已返回

2. **与伪事务的交互**：
   - 由于事务回调内本身使用的就是全局 `prisma`（伪事务，每条语句自动提交），link.create 和 link.update 都是独立提交
   - 但因为 forEach 没有 await，**update 的执行顺序完全不确定**：可能在下一个 link.create 之后、可能在整个事务回调返回之后
   - 即使回调整体抛异常，已提交的 create 和 update 都不会回滚

3. **导致的不一致路径**：

| 场景 | 结果 |
|------|------|
| 回调返回时 pinnedLinks update 还没执行完 | link 已入库，pinnedBy 关系稍后写入或丢失 |
| 部分 update 成功、部分失败 | pinnedLinks 状态与备份列表不一致 |
| update 失败但无 try/catch 包裹 | 产生未处理 Promise rejection（UnhandledPromiseRejection） |
| URL 匹配失败（见路径三） | pinned 状态完全丢失 |

#### 7.4.4 路径三：pinnedLinks URL 匹配不一致

pinnedLinks 通过 URL 精确字符串匹配（`pinnedLink.url === newLink.url`），但 link.url 在创建时经过了 `trim().slice(0, 2047)` 处理 [importFromLinkwarden.ts#L66](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L66)，而匹配时的 `pinnedLink.url` 是备份文件中的原始值。

如果备份中 `pinnedLinks[i].url` 含有前后空格或长度超过 2047 被截断，而对应 link 创建时被规范化了，则 `===` 比较失败，pinned 状态不会被设置——即使 URL 本质相同。

#### 7.4.5 路径四：旧 id → 新 id 映射缺失导致 pinnedLinks 无法精确关联

导出 backup.json 时，`pinnedLinks` 数组中每个元素包含完整的 Link 字段（包括**旧数据库中的 id**）：
```json
"pinnedLinks": [
  { "id": 42, "url": "https://example.com", "name": "...", "collectionId": 5, ... }
]
```

理论上可以用 `pinnedLinks[*].id`（旧 link id）与 `collections[*].links[*].id`（旧 link id）做精确匹配，建立 **旧 id → 新 id** 映射表，再用新 id 设置 pinnedBy。

但当前实现：
- 导入时完全忽略备份中的所有旧 id（collection.id、link.id、tag.id 全部丢弃，由数据库 autoincrement 重新生成）
- 没有建立任何 旧 id → 新 id 的映射
- 只能退而求其次用 URL 做模糊匹配（可能重复、可能被规范化而不匹配）

这是导出结构与导入逻辑的根本性不对称。

### 7.5 容量校验细节

**函数：** [hasPassedLimit(userId, numberOfImports)](packages/lib/verifyCapacity.ts#L8-L109)

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

**文件：** [importFromHTMLFile.test.ts](apps/web/lib/api/controllers/migration/importFromHTMLFile.test.ts)

已覆盖的测试场景（vitest + 真实 Prisma）：

1. 超出链接限额时返回 400 且不写入数据
2. 根级链接自动归入 "Imports" collection，正确导入 tags/ADD_DATE/描述（含 HTML 实体解码 `&amp;` → `&`）
3. 嵌套 collection（Recipes → Desserts）正确创建父子层级并分配链接
4. 已存在 "Imports" collection 时复用，不创建重复
5. 空文件夹名回退为 "Untitled Collection"
6. 无效 URL 静默跳过，仅有效 URL 入库

另有一段按 importDate 排序 ID 的测试被注释掉（`sortBookmarksTreeByEffectiveDate` 整个函数也被注释），当前未启用。

---

## 9. 关键不一致点汇总

| # | 问题 | 位置 | 影响 |
|---|------|------|------|
| 1 | `$transaction` 回调未使用 `tx` 参数，所有语句用全局 `prisma` 执行，**伪事务** | Linkwarden/Pocket/Wallabag/Omnivore 4 个控制器 | 每条语句独立自动提交，中途异常不回滚，留下半成品数据 |
| 2 | Linkwarden 导入 where 中 tag 只 `slice` 不 `trim`，create 中 `trim().slice()` | [importFromLinkwarden.ts#L85 vs L90](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L85-L90) | 带空格 tag 名触发唯一约束冲突 → 异常被吞 + 假 200 + 部分数据已入库 |
| 3 | Linkwarden/Pocket/Wallabag 事务 catch 吞异常，始终返回 200 | [importFromLinkwarden.ts#L119](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L119) 等 | 发生异常但前端显示成功（假阳性），用户误判导入完成 |
| 4 | Linkwarden pinnedLinks 使用 `forEach(async)`，无 await | [importFromLinkwarden.ts#L102-L113](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L102-L113) | pinned 状态可能部分/全部丢失，或产生未处理 Promise 拒绝 |
| 5 | pinnedLinks URL 精确匹配，未与 link 创建时的 `trim().slice()` 对齐 | [importFromLinkwarden.ts#L66 vs L103](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L66-L103) | URL 含空格/超长时 pinned 状态无法匹配 |
| 6 | **pinnedLinks 导出包含 textContent/preview/image 等 6 个大体积字段**（collections.links 已 omit） | [exportData.ts#L10-L25](apps/web/lib/api/controllers/migration/exportData.ts#L10-L25) | backup.json 体积严重膨胀，置顶 100 条可能额外增加 10MB+ |
| 7 | **pinnedLinks 导出不含 tags，但 TS 类型 LinksIncludingTags 声明有 tags** | [exportData.ts#L25](apps/web/lib/api/controllers/migration/exportData.ts#L25) vs [global.ts#L129-L132](packages/types/global.ts#L129-L132) | 编译期类型谎言，运行时 `pinnedLinks[i].tags` 为 `undefined` |
| 8 | 缺少旧 id → 新 id 映射表，pinnedLinks 可通过 id 精确关联却退化为 URL 模糊匹配 | [importFromLinkwarden.ts#L101-L113](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L101-L113) | 同 URL 跨 collection 时 pinned 状态错误扩散；导出的 id/collectionId 完全浪费 |
| 9 | Linkwarden 自有格式导入不恢复 collection `parentId`，父子层级丢失 | [importFromLinkwarden.ts#L34-L50](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L34-L50) | 所有 collection 平铺为顶级，嵌套结构不可逆丢失 |
| 10 | 导出 include 了 `rssSubscriptions`，但导入完全不处理 | [exportData.ts#L8-L9](apps/web/lib/api/controllers/migration/exportData.ts#L8-L9) vs 整个 [importFromLinkwarden.ts](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts) | 所有 RSS 订阅静默丢失 |
| 11 | Collection 字段部分丢失（icon/iconWeight/isPublic/createdAt/updatedAt） | [importFromLinkwarden.ts#L34-L50](apps/web/lib/api/controllers/migration/importFromLinkwarden.ts#L34-L50) | 自定义图标、公开状态、原始创建时间等不可逆丢失 |
| 12 | Pocket 导入 tag 先 `slice(0,50)` 再 `trim()`，其他多数先 `trim()` 再 `slice()` | [importFromPocket.ts#L83](apps/web/lib/api/controllers/migration/importFromPocket.ts#L83) | 边界情况 tag 名称截断结果不一致 |
| 13 | HTML 导入不使用事务，其他格式使用伪事务 | 各控制器 | HTML 导入中途异常抛到顶层（但也不回滚），其他 4 种异常被吞返回 200 |
| 14 | 所有导入器均未做 URL 去重（与手动 postLink 的 `preventDuplicateLinks` 不一致） | 各控制器 | 重复导入或 URL 已存在时产生重复链接 |
