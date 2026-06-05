# 链接抓取与归档状态流转机制

## 一、抓取入口

### 1.1 创建链接自动触发

**入口模块**：创建链接控制器（`apps/web/lib/api/controllers/links/postLink.ts`）

当用户通过 API 创建新链接时，系统会自动初始化归档状态：

```typescript
// 创建链接时，如果URL不安全，直接标记为不可用
const newLink = await prisma.link.create({
  data: {
    url: link.url?.trim() || null,
    // ...
    ...(!shouldPreserveUrl && link.url
      ? {
          lastPreserved: new Date().toISOString(),
          readable: "unavailable",
          image: "unavailable",
          monolith: "unavailable",
          pdf: "unavailable",
          preview: "unavailable",
          indexVersion: null,
        }
      : {}),
  },
});
```

**关键点**：
- 如果 URL 不安全（SSRFI 检测不通过），立即标记所有归档格式为 `"unavailable"`，并设置 `lastPreserved` 完成时间
- 否则，`lastPreserved` 保持 `null`，等待 worker 处理

### 1.2 手动重新归档

**入口模块**：单链接归档 API（`apps/web/pages/api/v1/links/[id]/archive/index.ts`）

用户可以通过 PUT 请求触发单个链接的重新归档：

```typescript
// 重置所有状态字段为 null
await prisma.link.update({
  where: { id: link.id },
  data: {
    image: null,
    pdf: null,
    readable: null,
    monolith: null,
    preview: null,
    lastPreserved: null,
    indexVersion: null,
    clientSide: false,
  },
});
```

### 1.3 文件上传归档

**入口模块**：归档上传 API（`apps/web/pages/api/v1/archives/index.ts`）

用户上传文件时，系统使用了一个"临时锁定"机制防止与 worker 竞争：

```typescript
// 创建时临时锁定，设置 lastPreserved = 1970-01-01，防止归档处理器选中
const link = await prisma.link.create({
  data: {
    // ...
    lastPreserved: new Date(0).toISOString(), // 临时标记为"已处理"
    aiTagged: true,
    indexVersion: 1,
  },
});

// 上传完成后解锁，设置 lastPreserved = null，允许 worker 处理
await prisma.link.update({
  where: { id: link.id },
  data: {
    // ...
    lastPreserved: null, // 重新变为待处理
    aiTagged: false,
    indexVersion: null,
  },
});
```

---

## 二、归档推进

### 2.1 Worker 主循环

**入口模块**：链接处理 Worker（`apps/worker/workers/linkProcessing.ts`）

Worker 采用无限循环 + 公平调度的方式处理链接：

```typescript
export async function linkProcessing(interval = 10) {
  let browser = await launchBrowser();

  while (true) {
    // 每30分钟重启浏览器防止内存泄漏
    if (Date.now() - browserStartTs >= BROWSER_MAX_AGE_MS) {
      await restartBrowser("30-minute rotation");
    }

    // 公平获取一批待处理链接
    const links = await getLinkBatchFairly({
      maxBatchLinks: ARCHIVE_TAKE_COUNT,
      mode: "links",
    });

    if (links.length === 0) {
      await delay(interval);
      continue;
    }

    // 并行处理
    const processingPromises = links.map((e) => archiveLink(e));
    await Promise.allSettled(processingPromises);

    await delay(interval);
  }
}
```

### 2.2 公平调度算法

**入口模块**：公平批次获取器（`apps/worker/lib/getLinkBatchFairly.ts`）

为了防止单个用户占用全部处理资源，系统采用多用户轮询调度：

```typescript
const baseLinkWhere: Prisma.LinkWhereInput = {
  url: { not: null },
  lastPreserved: null, // 核心条件：只取未处理的链接
};

// 步骤1：找出所有有待处理链接的付费/试用用户
const users = await prisma.user.findMany({
  where: {
    createdLinks: { some: { ...baseLinkWhere } },
    // 订阅/试用期检查...
  },
  orderBy: [{ lastPickedAt: "asc" }], // 优先处理很久没被调度的用户
  take: maxBatchLinks,
});

// 步骤2：从每个用户轮流取链接，直到凑够批次大小
while (picked.size < maxBatchLinks) {
  for (const { id: userId } of users) {
    const userLinks = await prisma.link.findMany({
      where: { ...baseLinkWhere, createdBy: { id: userId } },
      orderBy: [{ createdAt: "desc" }], // 用户内按创建时间倒序
      skip: nextOffset.get(userId) ?? 0,
      take: toTake,
    });
    // ...
  }
}

// 步骤3：标记用户已被调度
await prisma.user.updateMany({
  where: { id: { in: uniqueUsersWithLinks } },
  data: { lastPickedAt: now },
});
```

### 2.3 归档处理核心

**入口模块**：归档处理器（`apps/worker/lib/archiveHandler.ts`）

核心处理流程：

```typescript
export default async function archiveHandler(
  link: LinkWithCollectionOwnerAndTags,
  browser: Browser
) {
  // 1. 安全检查
  await assertUrlIsSafeForServerSideFetch(link.url);

  // 2. 浏览器超时控制（默认5分钟）
  const abortController = new AbortController();
  const timeoutPromise = new Promise((_, reject) => {
    timeoutId = setTimeout(() => {
      abortController.abort();
      reject(new Error(`Browser timeout after ${BROWSER_TIMEOUT} minutes`));
    }, BROWSER_TIMEOUT * 60000);
  });

  // 3. 确定归档设置（标签级 > 用户级）
  const archivalSettings = archivalTags.length > 0
    ? { /* 基于标签的设置 */ }
    : { /* 基于用户默认设置 */ };

  // 4. 浏览器访问页面
  await Promise.race([
    (async () => {
      const { linkType } = await determineLinkType(link.id, link.url);

      if (linkType === "image") {
        await imageHandler(link, imageExtension);
      } else if (linkType === "pdf") {
        await pdfHandler(link);
      } else {
        await page.goto(link.url, { waitUntil: "domcontentloaded" });

        // 并行执行各种归档格式
        if (!link.preview) await handleArchivePreview(link, page);
        if (archivalSettings.archiveAsReadable && !link.readable)
          await handleReadability(content, link);
        if ((archivalSettings.archiveAsScreenshot && !link.image) ||
            (archivalSettings.archiveAsPDF && !link.pdf))
          await handleScreenshotAndPdf(link, page, archivalSettings);
        if (archivalSettings.archiveAsMonolith && !link.monolith)
          await handleMonolith(link, content, abortController.signal);
      }
    })(),
    timeoutPromise,
  ]);
}
```

---

## 三、状态记录

### 3.1 数据模型

**定义位置**：Link 数据模型（`packages/prisma/schema.prisma#L166-L198`）

Link 模型中与归档相关的字段：

```prisma
model Link {
  // 归档内容路径字段
  preview         String?   // 缩略图路径
  image           String?   // 截图路径
  pdf             String?   // PDF路径
  readable        String?   // 可读性JSON路径
  monolith        String?   // 单文件HTML路径

  // 状态控制字段
  type            String    // 链接类型：url/pdf/image
  indexVersion    Int?      // 索引版本（用于搜索）
  clientSide      Boolean   // 是否为客户端上传
  lastPreserved   DateTime? // 核心状态标记：null=待处理，有值=已处理
}
```

### 3.2 状态字段约定

系统**没有统一的 `archiveStatus` 枚举字段**，而是通过字段值的约定来表示状态：

| 字段值 | 含义 |
|--------|------|
| `null` | 待处理 / 需要重新处理 |
| `"unavailable"` | 已尝试处理但失败 / 无法归档 |
| 字符串路径（如 `"archives/123/456.pdf"`） | 处理成功，值为文件存储路径 |

### 3.3 状态流转核心逻辑

状态流转的关键在于归档处理器中的 `try-catch-finally` 结构：

```
函数入口
    │
    ├─ 安全检查失败或URL不安全 → 直接设置 lastPreserved + unavailable → 返回
    │
    ├─ 浏览器上下文/页面创建失败 → 抛出异常 → 函数终止 → lastPreserved 保持 null
    │
    └─ 进入 try 块
         │
         ├─ 成功 → 各归档格式字段被设置为文件路径
         │
         ├─ 失败 → catch 捕获并重新抛出异常
         │
         └─ finally 块（★ 无论成功失败，只要进入 try 就一定会执行 ★）
              │
              ├─ 查询 link 是否还存在
              │
              ├─ 存在 → 设置 lastPreserved = 当前时间
              │         仍为 null 的字段 → 标记为 "unavailable"
              │
              └─ 不存在 → 删除已生成的文件
```

### 3.4 finally 块的关键作用

**代码位置**：归档处理器 finally 块（`apps/worker/lib/archiveHandler.ts#L203-L230`）

```typescript
finally {
  if (timeoutId !== undefined) {
    clearTimeout(timeoutId);
  }

  const finalLink = await prisma.link.findUnique({
    where: { id: link.id },
  });

  if (finalLink) {
    await prisma.link.update({
      where: { id: link.id },
      data: {
        lastPreserved: new Date().toISOString(), // ★ 总是设置
        readable: !finalLink.readable ? "unavailable" : undefined,
        image: !finalLink.image ? "unavailable" : undefined,
        monolith: !finalLink.monolith ? "unavailable" : undefined,
        pdf: !finalLink.pdf ? "unavailable" : undefined,
        preview: !finalLink.preview ? "unavailable" : undefined,
        indexVersion: null,
      },
    });
  } else {
    await removeFiles(link.id, link.collectionId);
  }

  await context?.close().catch(() => {});
}
```

**重要结论**：只要进入了 `try` 块，无论处理成功还是失败，`finally` 块一定会执行，`lastPreserved` 一定会被设置为当前时间。

### 3.5 各归档格式的独立处理

每个归档格式都有独立的处理文件，成功后立即更新数据库：

| 归档格式 | 处理模块 | 错误处理方式 | 是否向外抛出 |
|---------|---------|-------------|------------|
| 预览图 | 预览图处理器（`apps/worker/lib/preservationScheme/handleArchivePreview.ts`） | 内部 catch 但非 UnsafeUrlError 会重抛 | 可能抛出 |
| 截图/PDF | 截图PDF处理器（`apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts`） | `Promise.allSettled` 捕获 | 不抛出 |
| 可读性 | 可读性处理器（`apps/worker/lib/preservationScheme/handleReadability.ts`） | 大小超限 return，无其他 catch | 可能抛出 |
| 单文件HTML | Monolith处理器（`apps/worker/lib/preservationScheme/handleMonolith.ts`） | 调用时 `.catch()` 捕获 | 不抛出 |
| 图片处理 | 图片处理器（`apps/worker/lib/preservationScheme/imageHandler.ts`） | 大小超限 return，无其他 catch | 可能抛出 |
| PDF处理 | PDF处理器（`apps/worker/lib/preservationScheme/pdfHandler.ts`） | 大小超限 return，无其他 catch | 可能抛出 |

以截图处理为例：

```typescript
page.screenshot({ fullPage: true, type: "jpeg" })
  .then(async (screenshot) => {
    await createFile({
      data: screenshot,
      filePath: `archives/${linkExists.collectionId}/${link.id}.jpeg`,
    });
    await prisma.link.update({
      where: { id: link.id },
      data: {
        image: `archives/${linkExists.collectionId}/${link.id}.jpeg`,
      },
    });
  });
```

---

## 四、失败补偿机制

### 4.1 异常抛出路径完整分析

让我们从归档处理器入口开始，逐条分析可能的异常路径：

**路径1：进入 try 块之前（函数开头至 try 之前）**

```
安全检查阶段: assertUrlIsSafeForServerFetch
  ├─ 抛出 UnsafeUrlError → skipPreservation = true，不向外抛出
  └─ 抛出其他异常 → 向外抛出 → 函数终止 → finally 不执行 → lastPreserved 保持 null

跳过处理阶段: skipPreservation 或 URL 非 http/https
  └─ 更新 DB: lastPreserved = 现在，所有字段 = "unavailable" → return → 正常结束

浏览器创建阶段: 创建浏览器上下文和页面
  ├─ browser.newContext() 抛出 → 函数终止 → finally 不执行 → lastPreserved 保持 null
  ├─ protectPageRequests() 抛出 → 函数终止 → finally 不执行 → lastPreserved 保持 null
  └─ browser.newPage() 抛出 → 函数终止 → finally 不执行 → lastPreserved 保持 null

初始化阶段: 创建文件夹、获取归档设置
  └─ createFolder() 抛出 → 函数终止 → finally 不执行 → lastPreserved 保持 null
```

**路径2：进入 try 块之后**

```
try {
  // 任何业务逻辑抛出异常
} catch (err) {
  console.log("Failed Link:", link.url);
  console.log("Reason:", err);
  throw err;  // 重新抛出异常
} finally {
  // ★ 无论是否有异常，这里一定会执行！★
  // 设置 lastPreserved = 当前时间
  // 未成功字段标记为 "unavailable"
}
```

### 4.2 Worker 层的异常捕获

**代码位置**：Worker 异常捕获（`apps/worker/workers/linkProcessing.ts#L45-L69`）

```typescript
const archiveLink = async (link: LinkWithCollectionOwnerAndTags) => {
  try {
    console.log(`- Link ${link.url} for user ${link.collection.ownerId}`);
    await archiveHandler(link, browser);
    console.log(`Succeeded processing link ${link.url}...`);
  } catch (error: any) {
    console.error(`Error processing link ${link.url}:`, error);
    // ★ 注意：这里没有更新数据库！
    // 但 archiveHandler 的 finally 已经执行过了
    // lastPreserved 已经被设置为当前时间

    if (!browser.isConnected?.()) {
      await restartBrowser("browser disconnected");
    }
  }
};
```

**关键理解**：Worker 的 catch 块只是记录日志和重启浏览器，**不会修改数据库状态**。但此时归档处理器的 finally 已经执行，`lastPreserved` 已经被设置。

### 4.3 真实自动重试条件

**自动重试只会发生在以下极端情况**：

| 场景 | 是否重试 | 原因 |
|-----|---------|------|
| 浏览器上下文创建失败 | ✅ 是 | 进入 try 块前抛出，finally 不执行 |
| 页面创建失败 | ✅ 是 | 进入 try 块前抛出，finally 不执行 |
| 文件夹创建失败 | ✅ 是 | 进入 try 块前抛出，finally 不执行 |
| 进程被杀死（OOM、重启等） | ✅ 是 | finally 来不及执行 |
| 页面加载失败 | ❌ 否 | 进入 try 块后抛出，finally 执行 |
| 浏览器超时 | ❌ 否 | 进入 try 块后抛出，finally 执行 |
| 某个归档格式处理失败 | ❌ 否 | finally 执行 |

**核心结论**：正常业务异常（页面无法访问、超时等）**不会自动重试**，只有基础设施层面的异常（浏览器/页面创建失败）才会自动重试。

### 4.4 "unavailable" 终态详解

当一个归档格式字段被标记为 `"unavailable"` 时，表示：

1. **系统已经尝试过处理**：`lastPreserved` 已设置为处理时间
2. **该格式本次处理失败**：可能是网络问题、页面结构问题、大小超限等
3. **不会自动重试**：因为 `lastPreserved` 已设置，worker 不会再选中
4. **需要手动触发**：用户必须调用重新归档 API，将 `lastPreserved` 重置为 `null`

### 4.5 部分成功的场景

如果处理过程中部分归档格式成功，但后续步骤抛出异常：

```
1. handleArchivePreview 成功 → preview = "archives/..."
2. handleReadability 成功 → readable = "archives/..."
3. page.goto 失败（网络中断）→ 抛出异常
4. catch 捕获并重新抛出
5. finally 执行：
   - lastPreserved = 当前时间
   - image = "unavailable"（因为还是 null）
   - pdf = "unavailable"
   - monolith = "unavailable"
   - preview 和 readable 保持已成功的路径
```

结果：
- 已成功的格式保留文件路径
- 未成功的格式标记为 `"unavailable"`
- `lastPreserved` 已设置，**不会自动重试**
- 下次手动重新归档时，已成功的格式会被跳过（因为有 `!link.preview` 检查）

### 4.6 待处理数量统计

**入口模块**：未处理链接计数器（`apps/worker/lib/countUnprocessedBillableLinks.ts`）

```typescript
const count = await prisma.link.count({
  where: {
    lastPreserved: null, // 待处理
    NOT: { url: null },
    // 只统计付费用户...
  },
});
```

---

## 五、状态流转全景图

```
用户创建链接
    │
    ├─ URL 不安全 → 全部标记为 "unavailable"，lastPreserved = 现在 → 归档结束
    │
    └─ URL 安全 → lastPreserved = null，所有归档字段 = null → 进入待处理队列
                        │
                        ▼
                Worker 轮询（getLinkBatchFairly）
                        │
                        ▼
                按用户公平调度选中链接
                        │
                        ▼
                archiveHandler 处理
                        │
         ┌──────────────┴──────────────┐
         │                             │
         ▼                             ▼
  浏览器上下文创建失败         浏览器上下文创建成功
         │                             │
  抛出异常，函数终止              进入 try 块
  lastPreserved 保持 null              │
  下次自动重试                        │
                                       ▼
                              确定链接类型
                                       │
                              浏览器访问页面
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         ▼                             ▼                             ▼
  生成预览图                    提取可读文本                    截图 + PDF
         │                             │                             │
         ├─────────────────────────────┴─────────────────────────────┘
         │
         ▼
  处理 Monolith（单文件HTML）
         │
         ├─ 全部成功 → 所有字段 = 文件路径
         │
         ├─ 部分成功 → 部分字段 = 路径，部分 = null
         │
         └─ 全部失败 → 所有字段 = null（但过程中可能有部分已写入）
                       │
                       ▼
               catch 捕获异常，重新抛出
                       │
                       ▼
               ★ finally 块执行 ★
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
        链接还存在吗？     链接已被删除 → 删除文件
              │
              ▼
        lastPreserved = 现在
        仍为 null 的字段 → "unavailable"
                       │
                       ▼
                 归档结束
   （不会自动重试，需手动重置 lastPreserved）
```

---

## 六、关键设计特点

| 设计决策 | 优点 | 缺点 |
|---------|------|------|
| 无统一状态字段，多字段独立 | 各格式可独立处理，部分成功不影响整体可用性 | 状态理解困难，没有整体进度指示 |
| 基于 `lastPreserved = null` 的隐式标记 | 简单可靠，无需额外状态字段 | 状态语义不明确，需要理解约定 |
| `finally` 块强制收尾 | 确保处理过的链接有明确终态，避免无限等待 | 正常业务失败不会自动重试，用户体验可能不好 |
| 按用户公平调度 | 防止大用户独占资源，保证多租户公平性 | 实现复杂，多轮数据库查询 |
| 浏览器每30分钟重启 | 防止内存泄漏，稳定性好 | 重启时正在处理的链接会失败（但会被标记为 unavailable） |

## 七、潜在改进点

1. **增加统一状态字段**：添加 `archiveStatus` 枚举（`PENDING`/`PROCESSING`/`COMPLETED`/`PARTIAL_FAILED`/`FAILED`），便于理解和查询
2. **增加重试次数字段**：添加 `retryCount` 字段，对可重试的失败进行有限次自动重试
3. **区分失败类型**：将失败区分为"临时性失败"（网络超时）和"永久性失败"（无效URL），分别处理
4. **部分失败回滚选项**：提供配置项，允许在整体失败时回滚已成功的归档文件
5. **状态流转日志**：记录每次处理的开始/结束时间、错误信息、成功的格式，便于排查
