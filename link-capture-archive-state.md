# 链接抓取与归档状态流转机制

## 一、抓取入口

### 1.1 创建链接自动触发

**入口文件**：[postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/web/lib/api/controllers/links/postLink.ts)

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

**入口文件**：[links/[id]/archive/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/web/pages/api/v1/links/%5Bid%5D/archive/index.ts)

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

**入口文件**：[archives/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/web/pages/api/v1/archives/index.ts)

用户上传文件时，系统使用了一个巧妙的"临时锁定"机制：

```typescript
// 创建时临时锁定，防止 archiveHandler 竞争
const link = await prisma.link.create({
  data: {
    // ...
    lastPreserved: new Date(0).toISOString(), // 临时标记为已处理（1970年）
    aiTagged: true,
    indexVersion: 1,
  },
});

// 上传完成后解锁，允许后续补充处理
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

**入口文件**：[linkProcessing.ts](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/worker/workers/linkProcessing.ts)

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

**入口文件**：[getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/worker/lib/getLinkBatchFairly.ts)

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

**入口文件**：[archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/worker/lib/archiveHandler.ts)

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

**入口文件**：[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/packages/prisma/schema.prisma#L166-L198)

Link 模型中与归档相关的字段：

```prisma
model Link {
  // 归档内容路径
  preview         String?   // 缩略图路径
  image           String?   // 截图路径
  pdf             String?   // PDF路径
  readable        String?   // 可读性JSON路径
  monolith        String?   // 单文件HTML路径

  // 状态控制字段
  lastPreserved   DateTime? // 最后处理时间，null=待处理
  type            String    // 链接类型：url/pdf/image
  indexVersion    Int?      // 索引版本（用于搜索）
  clientSide      Boolean   // 是否为客户端上传
  lastPreserved   DateTime? // 核心状态标记
}
```

### 3.2 状态字段约定

系统**没有统一的 `archiveStatus` 枚举字段**，而是通过字段值的约定来表示状态：

| 字段值 | 含义 |
|--------|------|
| `null` | 待处理 / 需要重新处理 |
| `"unavailable"` | 处理失败 / 无法归档 |
| 字符串路径（如 `"archives/123/456.pdf"`） | 处理成功，值为文件存储路径 |

### 3.3 状态流转过程

```
创建链接
    ↓
lastPreserved = null
所有归档字段 = null
    ↓
Worker 选中处理
    ↓
逐个处理各归档格式
  ├─ 成功 → 字段 = 文件路径
  └─ 失败 → 字段保持 null 或后续标记为 "unavailable"
    ↓
finally 块统一收尾
    ↓
lastPreserved = 当前时间
对于仍为 null 的字段 → 标记为 "unavailable"
```

**finally 块的统一处理逻辑**（[archiveHandler.ts#L208-L224](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/worker/lib/archiveHandler.ts#L208-L224)）：

```typescript
const finalLink = await prisma.link.findUnique({
  where: { id: link.id },
});

if (finalLink) {
  await prisma.link.update({
    where: { id: link.id },
    data: {
      lastPreserved: new Date().toISOString(),
      readable: !finalLink.readable ? "unavailable" : undefined,
      image: !finalLink.image ? "unavailable" : undefined,
      monolith: !finalLink.monolith ? "unavailable" : undefined,
      pdf: !finalLink.pdf ? "unavailable" : undefined,
      preview: !finalLink.preview ? "unavailable" : undefined,
      indexVersion: null,
    },
  });
}
```

### 3.4 各归档格式的独立处理

每个归档格式都有独立的处理文件，成功后立即更新数据库：

- **预览图**：[handleArchivePreview.ts](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts)
- **截图/PDF**：[handleScreenshotAndPdf.ts](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts)
- **可读性**：[handleReadability.ts](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/worker/lib/preservationScheme/handleReadability.ts)
- **单文件HTML**：[handleMonolith.ts](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/worker/lib/preservationScheme/handleMonolith.ts)

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

### 4.1 隐式重试：基于 `lastPreserved = null`

系统**没有显式的重试队列或重试次数字段**，重试机制完全依赖 `lastPreserved` 字段：

- 成功处理 → `lastPreserved` 设置为当前时间 → 不会再被选中
- 处理失败（抛出异常）→ `finally` 块可能没执行到 → `lastPreserved` 仍为 `null` → 下次轮询会再次选中

**Worker 中的异常捕获**（[linkProcessing.ts#L58-L68](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/worker/workers/linkProcessing.ts#L58-L68)）：

```typescript
const archiveLink = async (link) => {
  try {
    await archiveHandler(link, browser);
    // 成功
  } catch (error: any) {
    console.error(`Error processing link ${link.url}:`, error);
    // 注意：这里没有更新数据库！
    // lastPreserved 仍为 null，下次会重试
  }
};
```

### 4.2 部分成功的问题

这个设计存在一个**重要缺陷**：如果处理过程中部分归档格式成功了，但后续步骤抛出异常，会导致：

1. 已成功的字段已写入文件路径（如 `image = "archives/123/456.jpeg"`）
2. `lastPreserved` 仍为 `null`
3. 下次重试时，这些已成功的格式会被跳过（因为有 `!link.image` 检查）
4. 只有未成功的格式会被重试

**跳过已处理字段的逻辑**（[archiveHandler.ts#L169-L194](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/worker/lib/archiveHandler.ts#L169-L194)）：

```typescript
if (!link.preview) await handleArchivePreview(link, page);
if (archivalSettings.archiveAsReadable && !link.readable)
  await handleReadability(content, link);
if ((archivalSettings.archiveAsScreenshot && !link.image) ||
    (archivalSettings.archiveAsPDF && !link.pdf))
  await handleScreenshotAndPdf(link, page, archivalSettings);
```

### 4.3 最终失败标记

如果异常被 `archiveHandler` 内部捕获并走到了 `finally` 块，那么未成功的字段会被标记为 `"unavailable"`，此时：

- `lastPreserved` 被设置为当前时间
- 字段值为 `"unavailable"`
- **不会自动重试**，需要用户手动触发重新归档

### 4.4 待处理数量统计

**入口文件**：[countUnprocessedBillableLinks.ts](file:///d:/fz/0601/solo-dogfeeding/code/36-linkwarden/apps/worker/lib/countUnprocessedBillableLinks.ts)

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
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
      浏览器访问页面         确定链接类型
             │                     │
             ├─────────────────────┼─────────────────────┐
             ▼                     ▼                     ▼
      生成预览图            提取可读文本            截图 + PDF
             │                     │                     │
             ├─────────────────────┴─────────────────────┘
             │
             ▼
      处理 Monolith（单文件HTML）
             │
             ├─ 全部成功 → 所有字段 = 文件路径
             │
             ├─ 部分成功 → 部分字段 = 路径，部分 = null
             │
             └─ 全部失败 → 所有字段 = null
                        │
                        ▼
                finally 块执行
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
      链接还存在吗？         链接已被删除 → 删除文件
             │
             ▼
      lastPreserved = 现在
      仍为 null 的字段 → "unavailable"
                        │
                        ▼
                  归档结束
```

---

## 六、关键设计特点

| 设计决策 | 优点 | 缺点 |
|---------|------|------|
| 无统一状态字段，多字段独立 | 各格式可独立重试，部分成功不影响整体 | 状态理解困难，没有整体进度指示 |
| 基于 `lastPreserved = null` 的隐式重试 | 简单可靠，无需额外重试队列 | 无法控制重试次数，可能无限重试 |
| finally 块统一标记 `"unavailable"` | 确保处理过的链接有明确终态 | 标记为 unavailable 后不会自动重试 |
| 按用户公平调度 | 防止大用户独占资源 | 实现复杂，多轮数据库查询 |
| 浏览器每30分钟重启 | 防止内存泄漏 | 处理中的链接会失败（但会重试） |

## 七、潜在改进点

1. **增加统一状态字段**：添加 `archiveStatus` 枚举（`PENDING`/`PROCESSING`/`COMPLETED`/`FAILED`），便于理解和查询
2. **增加重试次数限制**：添加 `retryCount` 字段，防止无限重试
3. **部分失败回滚**：如果整体处理失败，考虑回滚已写入的文件和字段
4. **状态流转日志**：记录每次处理的开始/结束时间、错误信息，便于排查
