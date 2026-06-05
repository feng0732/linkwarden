# Linkwarden 资源文件存储与清理策略协作说明

## 一、资源文件存放位置

### 1.1 存储后端抽象

所有文件操作通过 `@linkwarden/filesystem` 包统一封装，支持两种存储后端：

| 存储方式 | 触发条件 | 根路径配置 |
|---|---|---|
| 本地文件系统 | 未配置完整 S3 环境变量 | `STORAGE_FOLDER`（默认 `data`），实际路径 = `<project_root>/../../<STORAGE_FOLDER>/` |
| S3 兼容对象存储 | 同时配置 `SPACES_ENDPOINT` / `SPACES_REGION` / `SPACES_KEY` / `SPACES_SECRET` | `SPACES_BUCKET_NAME` 指定的 bucket |

> 实现见：[s3Client.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/filesystem/s3Client.ts)、[createFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/filesystem/createFile.ts)、[readFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/filesystem/readFile.ts)

### 1.2 目录结构与文件类型

所有归档文件按 `collectionId` 分目录存储，按 `linkId + 后缀` 命名：

```
archives/
├── <collectionId>/
│   ├── <linkId>.pdf                # PDF 归档
│   ├── <linkId>.png                # 截图 (PNG)
│   ├── <linkId>.jpeg / .jpg        # 截图 (JPEG)
│   ├── <linkId>.html               # Monolith 单文件网页归档
│   └── <linkId>_readability.json   # Readability 正文提取结果
└── preview/
    └── <collectionId>/
        └── <linkId>.jpeg           # 预览缩略图

uploads/
└── avatar/
    └── <userId>.jpg                # 用户头像
```

> 文件后缀与格式枚举映射见：[getSuffixFromFormat.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/lib/shared/getSuffixFromFormat.ts)、[global.ts#L160-L166](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/types/global.ts#L160-L166)

### 1.3 文件系统核心 API

| 函数 | 文件 | 作用 |
|---|---|---|
| `createFile` | [createFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/filesystem/createFile.ts) | 写入文件（支持 base64 或 Buffer） |
| `readFile` | [readFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/filesystem/readFile.ts) | 读取文件，返回 `{ file, contentType, status }` |
| `fileExists` | [fileExists.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/filesystem/fileExists.ts) | 判断文件是否存在 |
| `removeFile` | [removeFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/filesystem/removeFile.ts) | 删除单个文件 |
| `removeFolder` | [removeFolder.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/filesystem/removeFolder.ts) | 递归删除目录（S3 下分页列举后批量删除） |
| `moveFile` | [moveFile.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/filesystem/moveFile.ts) | 移动文件（S3 下 = copy + delete） |
| `createFolder` | [createFolder.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/filesystem/createFolder.ts) | 创建目录（S3 下 no-op） |
| `removeFiles` | [manageFiles.ts#L4-L31](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/filesystem/manageFiles.ts#L4-L31) | 删除一个 link 的所有归档变体 |
| `moveFiles` | [manageFiles.ts#L33-L68](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/filesystem/manageFiles.ts#L33-L68) | 将一个 link 的所有归档变体从一个 collection 移到另一个 |

---

## 二、引用关系

### 2.1 数据库存储的文件路径

在 Prisma schema 中，`Link` 模型的以下字段存储文件相对路径（或特殊字符串 `"unavailable"`）：

| 字段 | 对应文件 | 说明 |
|---|---|---|
| `Link.image` | `archives/<collectionId>/<linkId>.(png\|jpeg\|jpg)` | 截图 |
| `Link.pdf` | `archives/<collectionId>/<linkId>.pdf` | PDF |
| `Link.readable` | `archives/<collectionId>/<linkId>_readability.json` | Readability 正文 |
| `Link.monolith` | `archives/<collectionId>/<linkId>.html` | Monolith 网页 |
| `Link.preview` | `archives/preview/<collectionId>/<linkId>.jpeg` | 预览缩略图 |
| `User.image` | `uploads/avatar/<userId>.jpg` | 用户头像 |

> Schema 定义见：[schema.prisma#L166-L198](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/packages/prisma/schema.prisma#L166-L198)

**字段值约定：**
- `null`：尚未生成，worker 将在后续轮询中处理
- `"unavailable"`：已尝试归档但失败（或用户配置禁用），不再重试（除非手动触发重新归档）
- 文件路径字符串：归档成功，可通过 API 读取

### 2.2 读取文件的入口

#### 2.2.1 归档文件读取（archives API）

`GET /api/v1/archives/[linkId]?format=<ArchivedFormat>&preview=<0|1>`

流程：
1. 通过 `resolveAccessibleArchive` 校验用户对 link 所在 collection 的权限（owner / member / 公开集合）
2. 拼装文件路径：
   - `preview=1` → `archives/preview/<collectionId>/<linkId>.jpeg`
   - 其他 → `archives/<collectionId>/<linkId + suffix>`
3. 调用 `readFile` 读文件并返回

> 实现见：[\[linkId\].ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/pages/api/v1/archives/[linkId].ts)、[resolveAccessibleArchive.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/lib/api/archives/resolveAccessibleArchive.ts)

#### 2.2.2 Monolith 安全读取（preserved API，用户内容域名隔离）

当配置 `NEXT_PUBLIC_USER_CONTENT_DOMAIN` 时，Monolith HTML 必须走独立域名以防止 XSS：

1. `GET /api/v1/preserved/token?linkId=&format=`：服务端签发 JWT token（有效期 300s），包含 `filePath`、`linkId`、`format`
2. 前端用 token 访问 `NEXT_PUBLIC_USER_CONTENT_DOMAIN/api/v1/preserved/view?token=`
3. 服务端校验 Host 头、token 有效性，然后读取文件返回

> 实现见：[createPreservedFormatUrl.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/lib/api/preserved/createPreservedFormatUrl.ts)、[view.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/pages/api/v1/preserved/view.ts)

#### 2.2.3 头像读取

`GET /api/v1/avatar/[id]`：直接读取 `uploads/avatar/<id>.jpg`，无需鉴权（头像视为公开）。

> 实现见：[\[id\].ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/pages/api/v1/avatar/[id].ts)

#### 2.2.4 上传归档文件

`POST /api/v1/archives/[linkId]?format=&preview=`：允许用户手动上传 PDF / 图片 / HTML 作为归档，流程：
1. 鉴权 + 检查 collection 权限（`canCreate`）
2. 校验文件大小（默认 10MB）和 MIME 类型
3. 若是图片，额外生成 preview
4. 调用 `createFile` 落盘，并更新 `Link.image / pdf / monolith` 字段

> 实现见：[\[linkId\].ts#L130-L290](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/pages/api/v1/archives/[linkId].ts#L130-L290)

---

## 三、清理触发点

### 3.1 触发场景总览

| 场景 | 触发方式 | 清理内容 | 关键实现 |
|---|---|---|---|
| 删除单个 Link | `DELETE /api/v1/links/[id]` | 该 link 所有归档文件 + MeiliSearch 索引 | [deleteLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts) |
| 批量删除 Links | `DELETE /api/v1/links` body `{ids:[]}` | 循环调用 removeFiles | [deleteLinksById.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/lib/api/controllers/links/bulk/deleteLinksById.ts) |
| 删除 Collection（Owner） | `DELETE /api/v1/collections/[id]` | 递归删子集合、对应 archives 目录 + preview 目录 + MeiliSearch | [deleteCollectionById.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts) |
| 成员离开 Collection | 同上（member 视角） | 仅解除 UsersAndCollections 关系，**不删文件** | 同上 |
| 删除 User | `DELETE /api/v1/users/[id]` | 其所有 collection 的 archives 目录 + 头像 + Stripe 订阅 | [deleteUserById.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts) |
| 修改 Link 的 URL | `PUT /api/v1/links/[id]`（URL 变化） | 删除旧 URL 对应的归档文件，DB 字段置 null | [updateLinkById.ts#L133-L144](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L133-L144) |
| Link 跨 Collection 移动 | `PUT /api/v1/links/[id]`（collectionId 变化） | 调用 `moveFiles` 在目录间移动文件，**不删文件** | [updateLinkById.ts#L195-L197](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L195-L197) |
| 清空单条 Link 归档 | `PUT /api/v1/links/[id]/archive` | 删除归档文件并将 DB 字段置 null，等待 worker 重新生成 | [index.ts (links/\[id\]/archive)](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/pages/api/v1/links/[id]/archive/index.ts) |
| 批量清空 Links 归档 | `DELETE /api/v1/links/archive` | 对指定 linkIds 执行 removeFiles + 字段置 null | [index.ts (links/archive)](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/pages/api/v1/links/archive/index.ts) |
| 管理员批量重置 | `DELETE /api/v1/worker/preservation` action=`allAndRePreserve` / `allBroken` | 删所有归档 或 仅重置标记为 "unavailable" 的条目 | [preservation.tsx](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx) |
| Worker 归档过程中 Link 被删 | archiveHandler finally 块 | 检测 finalLink 为 null 时立即 removeFiles | [archiveHandler.ts#L208-L227](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/worker/lib/archiveHandler.ts#L208-L227) |
| 用户移除头像 | `PUT /api/v1/users/[id]` 传 `image=""` | 删除 `uploads/avatar/<userId>.jpg` | [updateUserById.ts#L94-L96](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/lib/api/controllers/users/userId/updateUserById.ts#L94-L96) |

### 3.2 Worker 归档流程中的清理协作

Worker 入口：[worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/worker/worker.ts) → [linkProcessing.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/worker/workers/linkProcessing.ts) 每 N 秒轮询未归档的 link。

关键协作点在 [archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/worker/lib/archiveHandler.ts)：

1. **归档前创建目录**：`createFolder({ filePath: "archives/preview/{collectionId}" })` 和 `archives/{collectionId}`。
2. **若归档过程中 link 已被删除**：`finally` 块中 `prisma.link.findUnique` 返回 null → 调用 `removeFiles(link.id, link.collectionId)` 清理半成品。
3. **归档失败标记**：对未生成的格式，将对应字段写入 `"unavailable"`（而不是保留 null），避免 worker 无限重试。需要再次归档时，前端应调用 `PUT /links/[id]/archive` 将字段重置为 null。

### 3.3 删除 Collection 的递归逻辑

[deleteCollectionById.ts#L106-L150](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L106-L150) 中 `deleteSubCollections` 递归处理：

```
对每个子集合 subCollection:
  ├── 递归 deleteSubCollections(subCollection.id)
  ├── 删除 UsersAndCollections 关联
  ├── 查询所有 links 的 id → 删除 MeiliSearch 文档
  ├── prisma.link.deleteMany (级联删 Highlight 等)
  ├── prisma.collection.delete
  └── removeFolder("archives/{subCollection.id}")
      removeFolder("archives/preview/{subCollection.id}")
```

注意：文件删除发生在 DB 记录删除之后（先删子集合关联数据，再删目录），且不在 Prisma `$transaction` 内与 DB 操作一起原子化。

### 3.4 删除 User 的清理逻辑

[deleteUserById.ts#L108-L205](file:///d:/fz/0601/solo-dogfeeding/code/40-linkwarden/apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L108-L205) 在一个 prisma `$transaction(timeout=20s)` 内：

1. 查询 ownerId = 该用户的所有 link → 删除 MeiliSearch 文档
2. 查询其所有 collection → `Promise.all` 删除每个 collection 的 `archives/{id}` 和 `archives/preview/{id}` 目录
3. 删除头像 `uploads/avatar/{userId}.jpg`
4. 处理 Stripe 订阅取消 / seat 扣减
5. 最终 `prisma.user.delete`（通过 `onDelete: Cascade` 级联清理 Collection / Link / Tag / Highlight 等表）

---

## 四、残留处理与容错

### 4.1 可能产生残留文件的场景

| 场景 | 原因 | 是否有自动清理 |
|---|---|---|
| 事务 / API 中断 | 删除 collection 时 DB 回滚但 `removeFolder` 已执行；或反之 DB 已删但 API 在 removeFolder 前崩溃 | **无**，需人工干预 |
| S3 删除静默失败 | `removeFile` / `removeFolder` 中 S3 异常仅 `console.log`，未抛出 | **无** |
| 本地 fs.unlink 回调失败 | `removeFile` 的 fs.unlink 为异步，出错仅日志 | **无** |
| link 跨 collection 移动中异常 | `moveFile` 在 S3 上先 copy 再 delete，若 copy 完 delete 前崩溃，源文件残留 | **无** |
| 临时上传文件未清理 | `formidable` 产生的临时文件，仅在上传成功时 `fs.unlinkSync` | 若上传失败回调中抛错，可能残留 formidable 临时目录文件（由操作系统 / tmpwatch 处理） |
| 归档中途 Worker 进程被杀 | archiveHandler 正在写多个格式时崩溃，半成品文件无 DB 引用 | **无**，且 DB 字段保持 null 会让后续 worker 再次覆盖式写入 |
| DB 字段与实际文件不一致 | 文件在存储端被人为删除，但 DB 仍保留路径 | 读取时 `readFile` 返回 404，前端展示 "File not found" 模板 |

### 4.2 现存容错措施

1. **文件操作静默失败**：所有 `removeFile` / `removeFolder` / `moveFile` 在本地和 S3 模式下都用 `try/catch` + `console.log`，保证上层业务（如删除 Link / Collection）即使文件删不掉也能继续完成 DB 层删除，避免"删不掉文件导致记录无法删除"。

2. **读取端优雅降级**：`readFile` 在文件不存在时返回 `{ file: "File not found.", contentType: "text/plain", status: 404 }`，并附带 HTML 模板说明，不会 500。

3. **重复写入幂等**：归档流程对已存在的格式（例如 `link.image` 非空时跳过截图），避免重复写。手动上传时若 PNG/JPEG 互斥也会先检查。

4. **"unavailable" 标记熔断**：归档失败的格式会被标记为 `"unavailable"`，worker 不会重复尝试，防止无限浪费资源。

### 4.3 缺失能力（当前代码未实现）

- **孤儿文件扫描**：没有定时任务扫描 `archives/**` / `uploads/**`，比对 DB 中存在的 `collectionId` / `linkId` / `userId`，删除无主文件。
- **回收站 / Soft Delete**：所有删除均为硬删除，DB 和文件均无回退可能。
- **事务一致性**：文件系统操作未参与 DB 事务，无法回滚。
- **操作审计**：删除文件无日志记录（仅 `console.log`）。

### 4.4 关键环境变量

| 变量 | 默认 | 作用 |
|---|---|---|
| `STORAGE_FOLDER` | `data` | 本地存储根目录名 |
| `SPACES_ENDPOINT` / `SPACES_REGION` / `SPACES_KEY` / `SPACES_SECRET` / `SPACES_BUCKET_NAME` | - | S3/DO Spaces 配置，同时存在时启用 S3 模式 |
| `SPACES_FORCE_PATH_STYLE` | - | S3 path-style 访问开关 |
| `NEXT_PUBLIC_USER_CONTENT_DOMAIN` | - | 配置后强制 Monolith 通过独立域名 + JWT token 访问，防御 XSS |
| `NEXT_PUBLIC_MAX_FILE_BUFFER` | `10` | 手动上传文件大小限制（MB） |
| `DISABLE_PRESERVATION` | - | 设为 `true` 时所有归档全部标记为 unavailable，不生成文件 |

---

## 五、调用链简图

```
用户上传 / Worker 归档
        │
        ▼
  createFile / createFolder  ─────────┐
        │                             │
        ▼                             │
  DB 写入 Link.image/pdf/...          │
        │                             │
        ▼                             │
  前端访问 ──► archives API           │
        │     resolveAccessibleArchive│
        │     readFile                │
        ▼                             │
  文件返回 ◄───────────────────────────┘

用户删除 Link / Collection / User / 重新归档
        │
        ▼
  prisma.link/collection/user.delete(...)
        │
        ▼
  removeFiles(linkId, collectionId)  ◄── 删除所有格式文件
  或 removeFolder("archives/{id}")   ◄── 删整个 collection 目录
  或 removeFolder("archives/preview/{id}")
  或 removeFile("uploads/avatar/{id}.jpg")
        │
        ▼
  meiliClient.deleteDocument(s)      ◄── 同步清理搜索索引
```
