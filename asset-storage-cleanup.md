# Linkwarden 资源文件存储与清理策略协作说明

## 一、资源文件存放位置

### 1.1 存储后端抽象

所有文件操作通过 `@linkwarden/filesystem` 包统一封装，支持两种存储后端，**运行时通过 `s3Client` 是否存在自动切换**：

| 存储方式 | 触发条件 | 实际落点 |
|---|---|---|
| 本地文件系统 | `SPACES_ENDPOINT` / `SPACES_REGION` / `SPACES_KEY` / `SPACES_SECRET` 任意一个未配置 | 根目录由 `STORAGE_FOLDER`（默认 `data`）决定，最终绝对路径 = `path.join(process.cwd(), "../..", STORAGE_FOLDER, filePath)` |
| S3 兼容对象存储 | 上述四个变量全部配置 | Bucket = `SPACES_BUCKET_NAME`，对象 Key = 传入的 `filePath` |

> S3 客户端初始化见：[s3Client.ts](packages/filesystem/s3Client.ts)
> 本地模式路径拼接在每个文件系统函数内部都重复了一遍 `path.join(process.cwd(), "../..", storagePath, filePath)`，未抽取成常量。

### 1.2 目录结构与文件命名（全部为无前置斜杠的相对路径 Key）

```
archives/<collectionId>/<linkId>.pdf                  # PDF 归档
archives/<collectionId>/<linkId>.png                  # 截图 PNG
archives/<collectionId>/<linkId>.jpeg                 # 截图 JPEG
archives/<collectionId>/<linkId>.jpg                  # 截图 JPG（仅 removeFiles 兼容删除）
archives/<collectionId>/<linkId>.html                 # Monolith 单文件网页
archives/<collectionId>/<linkId>_readability.json     # Readability 正文 JSON
archives/preview/<collectionId>/<linkId>.jpeg         # 预览缩略图（固定 JPEG）
uploads/avatar/<userId>.jpg                            # 用户头像（固定 JPG）
```

> 枚举与后缀的映射见：[getSuffixFromFormat.ts](apps/web/lib/shared/getSuffixFromFormat.ts)、[global.ts#L160-L166](packages/types/global.ts#L160-L166)
> 所有 Worker 归档 Handler 在写文件时都用模板字面量拼路径：[handleScreenshotAndPdf.ts](apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts)、[handleReadability.ts](apps/worker/lib/preservationScheme/handleReadability.ts)、[handleMonolith.ts](apps/worker/lib/preservationScheme/handleMonolith.ts)、[handleArchivePreview.ts](apps/worker/lib/preservationScheme/handleArchivePreview.ts)、[imageHandler.ts](apps/worker/lib/preservationScheme/imageHandler.ts)、[pdfHandler.ts](apps/worker/lib/preservationScheme/pdfHandler.ts)、[generatePreview.ts](packages/lib/generatePreview.ts)

### 1.3 文件系统核心 API 行为核准

| 函数 | 文件 | 关键行为 / 注意点 |
|---|---|---|
| `createFile` | [createFile.ts](packages/filesystem/createFile.ts) | 返回 `boolean` 表示成功/失败；本地会 `fs.mkdir(recursive)` 自动建目录；S3 异常仅 console.error |
| `readFile` | [readFile.ts](packages/filesystem/readFile.ts) | 本地/S3 返回结构不一致（见 §二）；文件末尾有一段 `fileNotFoundTemplate` HTML **从未被 return 使用** |
| `fileExists` | [fileExists.ts](packages/filesystem/fileExists.ts) | S3 用 `HeadObject`；本地用 `fs.existsSync`；任何异常都当"不存在"返回 false |
| `removeFile` | [removeFile.ts](packages/filesystem/removeFile.ts) | **本地模式 fs.unlink 是异步回调、不 await**，调用方返回时删除可能还没完成；S3 用 await 但失败只 console.log |
| `removeFolder` | [removeFolder.ts](packages/filesystem/removeFolder.ts) | S3 用 `ListObjects` + `DeleteObjects` 分页循环，`IsTruncated` 时递归；本地用 `fs.rmdirSync(recursive:true)`；目录不存在时仅打日志 |
| `moveFile` | [moveFile.ts](packages/filesystem/moveFile.ts) | **S3 模式使用回调风格 `s3Client.copyObject(...)`，未 await/Promisify**，调用返回时 copy+delete 尚未执行；本地先 `fs.existsSync` 再 rename，不存在直接跳过 |
| `createFolder` | [createFolder.ts](packages/filesystem/createFolder.ts) | S3 no-op；本地 `fs.mkdirSync(recursive:true)` |
| `removeFiles` | [manageFiles.ts#L4-L31](packages/filesystem/manageFiles.ts#L4-L31) | 对一个 link 并发 await 7 次 `removeFile`（pdf/png/jpeg/jpg/html/preview/readability），每个独立 try/catch |
| `moveFiles` | [manageFiles.ts#L33-L68](packages/filesystem/manageFiles.ts#L33-L68) | 对 7 个文件调用 `moveFile`，在 S3 模式下全部"发射后不管" |

---

## 二、引用关系 & 读取降级（本地 vs S3 精确对比）

### 2.1 数据库字段与文件路径的对应

`Link` 模型相关字段类型是 `String?`（可空），实际取值有三种语义：

| 取值 | 含义 | Worker 是否会再处理 |
|---|---|---|
| `null` | 从未归档 / 已被重置等待重新归档 | **是**（`lastPreserved: null` 才会被 `getLinkBatchFairly` 选中） |
| `"unavailable"` | 已尝试归档但失败 / 被用户配置禁用 | **否**（字符串是 truthy，`!link.image === false`） |
| 形如 `archives/<collectionId>/<linkId>.<suffix>` 的字符串 | 归档成功，值即为文件系统/ S3 Key | **否**（见各 handler 的 `startsWith("archive")` 判断） |

> Worker 批量取待处理 Link 的条件：[getLinkBatchFairly.ts#L36-L38](apps/worker/lib/getLinkBatchFairly.ts#L36-L38)（`url != null AND lastPreserved IS NULL`）

### 2.2 读取文件路径的来源：**DB 字段 vs 实时拼接（不统一！）**

这是一个关键细节：读取文件有**两条路径**，使用的 filePath 来源并不相同：

#### 路径 A — archives API（`GET /api/v1/archives/[linkId]`）
在 [resolveAccessibleArchive.ts#L70-L72](apps/web/lib/api/archives/resolveAccessibleArchive.ts#L70-L72) 中：
```ts
filePath = isPreview
  ? `archives/preview/${collection.id}/${linkId}.jpeg`
  : `archives/${collection.id}/${linkId + suffix}`;
```
**它不读 DB 里存的 `Link.image/pdf/...` 字段，而是根据当前 `collection.id` + `linkId` + `format` 实时拼出来。**

#### 路径 B — Worker 内部读已有 Monolith
在 [archiveHandler.ts#L132-L148](apps/worker/lib/archiveHandler.ts#L132-L148) 中：
```ts
if (link.monolith?.endsWith(".html")) {
  const file = await readFile(link.monolith); // 直接用 DB 里存的路径
}
```

#### 路径 C — 前端显示判断（getFormatBasedOnPreference）
在 [getFormatBasedOnPreference.ts](packages/lib/getFormatBasedOnPreference.ts) 中同时检查 falsy **和** `"unavailable"`：
```ts
if (!link.pdf || link.pdf === "unavailable") return null;
```
再用 `link.image?.endsWith("png")` 判断返回 PNG 还是 JPEG 枚举。

**影响：** 如果 Link 从 collection A 移到 collection B，但 `moveFiles` 执行失败/未完成，那么 DB 字段里存的还是 `archives/A/123.jpeg`，archives API 按实时拼接会去读 `archives/B/123.jpeg`（404），而 Worker 内部若复用 monolith 则会读 DB 里存的旧路径（可能还存在也可能已被删）。

### 2.3 readFile 返回值的本地/S3 差异（核准！）

[readFile.ts](packages/filesystem/readFile.ts) 是读取降级的核心，三种情况返回值不同：

| 情景 | 本地文件系统 | S3 对象存储 |
|---|---|---|
| 文件存在 | `{ file: Buffer, contentType: 按后缀推断, status: 200 }` | `{ file: Buffer, contentType: 按后缀推断, status: 200 }` |
| 文件不存在 | `{ file: "File not found.", contentType: "text/plain", status: 404 }` | `headObject` 抛错 → `{ file: "File not found.", contentType: "text/plain", status: 400 }` **（注意是 400 不是 404）** |
| 内部异常（S3 连不上/权限错等） | N/A（本地异常会直接 throw 到最外层？不，本地没 try/catch，fs.existsSync 和 readFileSync 报错会上抛） | 最外层 catch → `{ file: "An internal occurred, please contact the support team.", contentType: "text/plain" }` **（注意没有 status 字段！）** |

> S3 模式下先 `headObject` 判断存在性，不存在直接 return 400，不会再发 `GetObject`。
> 文件末尾定义的 `fileNotFoundTemplate`（一段 HTML 说明）定义了但从未 return，实际上线不会输出这段 HTML。

#### 上层调用方如何用 status

- **archives API**：[linkId].ts#L122-L127
  ```ts
  const { file, contentType, status } = await readFile(filePath);
  res.setHeader("Content-Type", contentType)
     .status(status as number)       // S3 内部异常时 status=undefined → Number(undefined)=NaN → Express 默认 200
     .send(file);
  ```
- **preserved/view API**：[view.ts#L131-L134](apps/web/pages/api/v1/preserved/view.ts#L131-L134)
  ```ts
  if (status !== 200) {
    return res.status(status as number).send(file);  // status 为 undefined/NaN 时同上
  }
  ```
- **avatar API**：[id].ts#L32-L39
  ```ts
  res.setHeader("Content-Type", contentType)
     .status(status as number)
     .send(file);
  ```

**结论：** 当 S3 内部异常时 `status` 字段缺失，Express/Nest 会按 200 返回但 body 是错误文本，前端按成功解析但拿到的是纯文本，可能导致图片/PDF 解析失败。

---

## 三、清理触发点（按调用链核准）

### 3.1 触发场景总览（带精确调用点）

| 场景 | 触发方式 | 清理动作 | 关键代码 |
|---|---|---|---|
| 删除单 Link | `DELETE /api/v1/links/[id]` | `prisma.link.delete` → `removeFiles(linkId, collectionId)` → MeiliSearch 删文档 | [deleteLinkById.ts#L22-L31](apps/web/lib/api/controllers/links/linkId/deleteLinkById.ts#L22-L31) |
| 批量删 Link | `DELETE /api/v1/links` body `{ids:[]}` | `prisma.link.deleteMany` → 循环 `removeFiles` → MeiliSearch | [deleteLinksById.ts#L35-L51](apps/web/lib/api/controllers/links/bulk/deleteLinksById.ts#L35-L51) |
| 删除 Collection（Owner） | `DELETE /api/v1/collections/[id]` | 递归删子集合 DB 数据 → `removeFolder("archives/{id}")` + `"archives/preview/{id}"` → MeiliSearch → 删自身记录 | [deleteCollectionById.ts#L54-L101](apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L54-L101)、[#L106-L150](apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L106-L150) |
| 成员离开 Collection | 同上（member 调） | 仅解除 `UsersAndCollections` 关系，文件不动 | [deleteCollectionById.ts#L23-L49](apps/web/lib/api/controllers/collections/collectionId/deleteCollectionById.ts#L23-L49) |
| 删除 User | `DELETE /api/v1/users/[id]` | 在 `$transaction` 内：MeiliSearch → 所有 collection `removeFolder` → `removeFile("uploads/avatar/{id}.jpg")` → Stripe → `prisma.user.delete`（级联删 DB） | [deleteUserById.ts#L108-L205](apps/web/lib/api/controllers/users/userId/deleteUserById.ts#L108-L205) |
| 修改 Link URL | `PUT /api/v1/links/[id]` URL 变化 | `removeFiles(oldLink.id, oldLink.collectionId)`，DB 中 image/pdf/readable/monolith/preview/lastPreserved 全部置 `null` | [updateLinkById.ts#L133-L163](apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L133-L163) |
| Link 跨 Collection 移动 | `PUT` 时 collectionId 变化 | `moveFiles(linkId, oldCollectionId, newCollectionId)` | [updateLinkById.ts#L195-L197](apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L195-L197) |
| 清空单条归档并重做 | `PUT /api/v1/links/[id]/archive` | 字段置 `null` + `removeFiles` | [links/[id]/archive/index.ts#L51-L68](apps/web/pages/api/v1/links/[id]/archive/index.ts#L51-L68) |
| 批量清空归档 | `DELETE /api/v1/links/archive` body `{linkIds:[]}` | 每条授权 link → `removeFiles` + 字段置 `null` | [links/archive/index.ts#L62-L87](apps/web/pages/api/v1/links/archive/index.ts#L62-L87) |
| 管理员批量重置 | `DELETE /api/v1/worker/preservation` action=`allAndRePreserve` \| `allBroken` | 全部删/仅重置 `"unavailable"` → 字段置 `null` | [preservation.tsx#L36-L165](apps/web/pages/api/v1/worker/preservation.tsx#L36-L165) |
| Worker 归档过程中 Link 被删 | archiveHandler `finally` 块 | `finalLink == null` → `removeFiles` | [archiveHandler.ts#L208-L227](apps/worker/lib/archiveHandler.ts#L208-L227) |
| 用户清除头像 | `PUT /api/v1/users/[id]` body `image=""` | `removeFile("uploads/avatar/{userId}.jpg")` | [updateUserById.ts#L94-L96](apps/web/lib/api/controllers/users/userId/updateUserById.ts#L94-L96) |

### 3.2 Worker 内 "已归档" 判断的两套不统一逻辑

判断"某个格式是否已归档"散落在三处且条件不一致：

| 位置 | 判断条件 | `"unavailable"` 会被视为"已归档"吗？ |
|---|---|---|
| [archiveHandler.ts#L122](apps/worker/lib/archiveHandler.ts#L122) 等外层入口 | `!link.image`（falsy 判断） | **是**（字符串为 truthy，跳过整个分支，不会再尝试） |
| [handleScreenshotAndPdf.ts#L23](apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L23) / [#L58](apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L58) | `!link.image?.startsWith("archive")` | **否**（`"unavailable"` 不以 "archive" 开头，会继续尝试重新生成——但外层已经跳过了这个 handler，实际上到不了这里） |
| [handleArchivePreview.ts#L42](apps/worker/lib/preservationScheme/handleArchivePreview.ts#L42) / [#L59](apps/worker/lib/preservationScheme/handleArchivePreview.ts#L59) | `!link.preview?.startsWith("archive")` | 同上 |

**但因为外层用 falsy 先把门，`"unavailable"` 的 Link 永远不会进入具体 handler，所以不会被重新处理——必须手动走"清空归档并重做" API 把字段重置成 `null`。**

### 3.3 archiveHandler finally 块的残留处理（核准！）

[archiveHandler.ts#L203-L229](apps/worker/lib/archiveHandler.ts#L203-L229) 的逻辑：

```
finally {
  finalLink = prisma.link.findUnique({ id: link.id })
  if (finalLink) {
    // 对每个字段：若 DB 里该字段仍为 falsy（null/空串），写 "unavailable"
    // 但不会去删已经写入的半成品文件！
    prisma.link.update({
      readable: !finalLink.readable ? "unavailable" : undefined,
      image:    !finalLink.image    ? "unavailable" : undefined,
      ...
    })
  } else {
    // Link 已被删 → 清理所有可能已写入的文件
    removeFiles(link.id, link.collectionId)
  }
}
```

**关键核准点：** 如果 createFile 成功但 prisma.update 失败（网络抖动），则出现"文件存在但 DB 字段仍是 null"的孤儿；finally 块会把字段打成 `"unavailable"`，**不会删那个孤儿文件**。下次该 Link 也不会再被 Worker 选中（因为 `lastPreserved` 已设置，见 getLinkBatchFairly 的 `lastPreserved: null` 条件）。

---

## 四、残留处理 & 容错（逐场景核准）

### 4.1 残留产生的精确场景

| # | 场景 | 代码路径 | 残留内容 | 有无自动清理 |
|---|---|---|---|---|
| 1 | `createFile` 成功 + `prisma.link.update` 失败 | 所有归档 handler（handleScreenshotAndPdf 等） | 存储端存在文件，DB 字段为 null / undefined → 随后 finally 打成 `"unavailable"` | 无，需手动触发"重新归档"或手工删 |
| 2 | 删除 Collection 时 `removeFolder` 之前进程崩溃 | deleteCollectionById.ts | DB 记录已删（或未删，视崩溃点），文件完整保留 | 无；若 DB 已删则文件永久无主 |
| 3 | S3 `removeFile` / `removeFolder` 网络异常 | removeFile.ts / removeFolder.ts try/catch console.log | DB 已删，S3 对象仍在 | 无 |
| 4 | 本地 `removeFile` fs.unlink 回调失败（文件被锁/权限） | removeFile.ts#L27-L29 | DB 已删，磁盘文件仍在 | 无 |
| 5 | `moveFile` S3 copyObject 成功但 removeFile 前崩溃（注意：moveFile 根本不 await） | moveFile.ts#L17-L23 | 目标路径和源路径都有同一文件副本（双倍占用） | 无；源路径在删老 collection 时可能被 `removeFolder` 清走 |
| 6 | `moveFile` 本地 fs.rename 中途崩溃（极罕见） | moveFile.ts#L33-L37 | 取决于 OS atomicity，通常要么都成功要么都失败 | 无 |
| 7 | Worker 处理到一半被 SIGKILL（例如部署重启） | archiveHandler 中间任意位置 | 已写入的文件孤儿；DB 字段仍是 null；`lastPreserved` 仍是 null → **下次会被重新选中并覆盖写入** | 部分"自愈"（覆盖写），但旧版本字节不会被释放 |
| 8 | 人工在存储端删除文件但保留 DB 路径 | 运维操作 | readFile 返回 400/404/无 status，前端显示 "File not found" | 无自愈机制，需要 DB 侧手动重置字段或重新归档 |
| 9 | formidable 上传文件失败时未进入 catch 分支 | archives/[linkId].ts#L199-L289 | OS 临时目录残留 form 文件 | 依赖系统 tmpwatch / 重启清理 |
| 10 | `removeFolder` S3 分页列举时新写入了对象 | removeFolder.ts#L22-L36 IsTruncated 递归 | 后写入的对象可能漏删 | 无 |

### 4.2 现存容错措施（核准）

1. **文件操作静默失败**：`removeFile` / `removeFolder` / `moveFile` 的所有存储异常都被 try/catch 吞掉并仅 console.log，保证 DB 层删除不被阻塞。**代价是残留无感知。**
2. **读取端降级**：文件不存在时返回纯文本错误 + 非 2xx status（本地 404、S3 400、S3 内部异常无 status）。`fileNotFoundTemplate` HTML 声明了但未被实际使用。
3. **重复写入幂等**：
   - archiveHandler 外层：`!link.image` 等 falsy 判断跳过已归档格式；
   - handleScreenshotAndPdf 内层：`!link.image?.startsWith("archive")` 进一步保护；
   - 手动上传 API：`link.image.endsWith("jpeg") && format===png` 这种 PNG/JPEG 互斥检查。
4. **`"unavailable"` 熔断**：防止 Worker 对注定失败的 URL 反复尝试。
5. **`lastPreserved` 时间戳**：作为 Worker 取任务的"水位线"，避免重复选中已经处理过的 Link。

### 4.3 核准后补充的缺失能力

除已记录的孤儿扫描/回收站/事务一致性/操作审计外，根据本次代码核查再补充：

- **`readFile` 返回结构不统一**：S3 内部异常无 `status` 字段，调用方 `status as number` 变成 NaN，可能被 Express 当作 200 返回；建议固定返回结构。
- **`moveFile` S3 未 promisify**：回调风格导致 `moveFiles` 实际是"fire-and-forget"，Link 跨集合移动后 DB 字段还是旧路径，archives API 按新 collectionId 拼路径会 404，直到 moveFile 的异步回调真的跑完（甚至 copyObject 回调报错根本不会跑）。
- **`removeFile` 本地异步未 await**：`fs.unlink(callback)` 没包装成 Promise，`removeFiles` 并发 7 次删除返回时，文件可能还没真正释放磁盘空间。
- **`fileNotFoundTemplate` 死代码**：readFile.ts 末尾 HTML 模板从未被 return，可以清理或在 archives API 上使用以优化 UX。
- **归档处理中断后 DB 与存储端不一致无对账**：createFile 成功但 prisma.update 失败时，文件孤儿且 `"unavailable"` 字段阻断后续重试，只能手工干预。

### 4.4 关键环境变量（核准）

| 变量 | 默认 | 作用 |
|---|---|---|
| `STORAGE_FOLDER` | `data` | 本地存储根目录名 |
| `SPACES_ENDPOINT` / `SPACES_REGION` / `SPACES_KEY` / `SPACES_SECRET` / `SPACES_BUCKET_NAME` / `SPACES_FORCE_PATH_STYLE` | - | 全部存在则启用 S3 模式 |
| `NEXT_PUBLIC_USER_CONTENT_DOMAIN` | - | 配置后 Monolith 必须走独立域名 + JWT token |
| `NEXT_PUBLIC_MAX_FILE_BUFFER` | `10` | 手动上传文件大小 MB 限制 |
| `DISABLE_PRESERVATION` | - | `"true"` 时所有归档标记为 unavailable |
| `SCREENSHOT_MAX_BUFFER` / `PDF_MAX_BUFFER` / `READABILITY_MAX_BUFFER` / `PREVIEW_MAX_BUFFER` / `MONOLITH_MAX_BUFFER` | 均 `100`（preview 为 `10`） | Worker 归档时 Buffer 上限（MB），超限直接 console 并跳过不写 DB |

---

## 五、调用链简图

```
用户上传 / Worker 归档
  │
  ├─ 权限校验 ──► prisma 查询 collection/member
  │
  ├─ createFile(filePath=`archives/<collectionId>/<linkId>.<suffix>`)
  │    ├─ 本地: path.join(process.cwd(), "../..", STORAGE_FOLDER, filePath)
  │    └─ S3:   PutObject(Bucket=SPACES_BUCKET_NAME, Key=filePath)
  │
  ├─ prisma.link.update({ image/pdf/monolith/readable/preview: filePath })
  │    └─ 此处失败会产生孤儿文件（finally 仅把 null 置 unavailable，不删文件）
  │
  ▼
前端访问（两条路径不一致！）
  ├─ archives API: resolveAccessibleArchive 实时拼路径
  │     `archives/${collection.id}/${linkId+suffix}`  ──► readFile
  │
  └─ Worker 内部: readFile(link.monolith)  ◄── 直接使用 DB 存储的路径


用户删除 / 重做归档
  │
  ├─ prisma.link/collection/user.delete(...)   (DB 级联 onDelete: Cascade)
  │
  ├─ removeFiles(linkId, collectionId)         ◄── 7 种文件并发删（本地异步不 await）
  │   或 removeFolder("archives/<collectionId>")
  │   或 removeFolder("archives/preview/<collectionId>")
  │   或 removeFile("uploads/avatar/<userId>.jpg")
  │   或 moveFiles(linkId, fromId, toId)        ◄── S3 回调不 await，fire-and-forget
  │
  └─ meiliClient.deleteDocument(s)  ◄── 清理搜索索引
```
