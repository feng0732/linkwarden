# Linkwarden 错误处理与前端反馈映射关系分析

## 一、整体架构概览

错误处理和前端反馈在代码中通过以下五层串联：

```
后端 Controller → API Route → @linkwarden/router (React Query) → 组件 (toast/状态) → 页面渲染
```

核心依赖：
- **react-hot-toast**：全局 toast 提示
- **@tanstack/react-query**：数据获取、缓存、乐观更新、错误回滚
- **next-i18next**：多语言文案
- **zod**：Schema 校验 (schemaValidation.ts)

---

## 二、同步 API 错误处理流程

### 2.1 错误格式来源（后端）

#### 第一层：Controller 返回值
所有 Controller 统一返回 `{ response, status }` 结构：

[postLink.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/lib/api/controllers/links/postLink.ts#L12-L167)

```typescript
// 返回格式
{
  response: string | object,   // 错误消息字符串 或 成功数据
  status: number               // HTTP 状态码
}
```

典型错误来源：

| 触发场景 | 返回 response | status |
|---|---|---|
| Zod Schema 校验失败 | `Error: {zodMessage} [{path}]` | 400 |
| Collection 不可访问 | `"Collection is not accessible."` | 400 |
| 链接重复 | `"Link already exists"` | 409 |
| 超出订阅配额 | `"Your subscription has reached the maximum number of links allowed."` | 400 |
| 权限不足（updateLinkById） | `"Collection is not accessible."` | 401 |

[updateLinkById.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/lib/api/controllers/links/linkId/updateLinkById.ts#L11-L201)

#### 第二层：用户认证中间件
[verifyUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/lib/api/verifyUser.ts#L14-L73)

认证失败直接 `res.status(401/404).json({ response: "..." })` 并返回 null，不会继续到 Controller。

#### 第三层：API Route 层统一包装
[links/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/pages/api/v1/links/index.ts#L9-L72)

```typescript
const newlink = await postLink(req.body, user.id);
return res.status(newlink.status).json({ response: newlink.response });
```

**最终 HTTP 响应格式固定为：**
```json
{ "response": "错误消息或数据对象" }
```

### 2.2 前端 @linkwarden/router 层（React Query Mutation）

[links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/packages/router/links.tsx#L383-L549)

以 `useAddLink` 为例，三段式错误处理：

**(1) mutationFn - 同步校验 + 请求**

```typescript
mutationFn: async (link) => {
  // 前端预校验：URL 格式
  if (link.url || link.type === "url") {
    try {
      new URL(link.url || "");
    } catch (error) {
      throw new Error("invalid_url_guide");  // 抛出 i18n key
    }
  }

  const response = await fetch("/api/v1/links", { ... });
  const data = await response.json();

  if (!response.ok) throw new Error(data.response);  // 后端错误消息抛出

  return data.response;
}
```

关键：这里 `throw new Error(message)` 的 message 有两种来源：
- **前端预校验**：`"invalid_url_guide"` —— 这是一个 **i18n key**
- **后端返回**：`data.response` —— 这是后端返回的 **英文硬编码字符串**

**(2) onMutate - 乐观更新 + 快照保存**
- 取消相关 queries
- 保存 previousLinks / previousDashboard（用于错误回滚）
- 构造 optimisticLink（临时 ID 为负时间戳）插入缓存

**(3) onError - 错误回滚 + toast**
```typescript
onError: (error, _variables, context) => {
  if (toast && t) toast.error(t(error.message));  // 用 t() 翻译后显示
  else if (Alert) Alert.alert("Error", "...");

  // 回滚缓存到之前的快照
  context.previousLinks?.forEach(([queryKey, data]) => {
    queryClient.setQueryData(queryKey, data);
  });
  queryClient.setQueryData(["dashboardData"], context.previousDashboard);
}
```

### 2.3 前端 Zod 预校验（Modal 层）

[NewLinkModal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/ModalContent/NewLinkModal.tsx#L88-L100)

```typescript
const submit = async () => {
  const dataValidation = PostLinkSchema.safeParse(link);

  if (!dataValidation.success)
    return toast.error(
      `Error: ${dataValidation.error.issues[0].message} [${dataValidation.error.issues[0].path.join(", ")}]`
    );
  // ...
};
```

这层校验**不使用 i18n**，直接显示 Zod 默认英文消息。

---

## 三、归档字段三态定义（null / unavailable / 真实路径）

这是整个串联系统最关键的约定，三种字段值分别代表完全不同的生命周期阶段。

### 3.1 三态含义一览

| 字段值 | 代表阶段 | 产生位置 | 是否被 Worker 取到 | 前端展示 |
|---|---|---|---|---|
| `null` | **队列中待处理** | 创建链接时的初始值；或刷新归档 API 主动清空 | ✅ **是**（Worker 只取 `lastPreserved: null` 的链接） | 加载动画 / 轮询触发 |
| `"unavailable"` | **已处理但失败** | Worker archiveHandler 的 finally 块，当某格式未成功生成时写入 | ❌ **否**（Worker 不再碰它） | 该项被隐藏（或显示灰色占位） |
| 真实路径字符串（如 `"archives/123/456.png"`） | **已处理且成功** | Worker archiveHandler 正常生成文件后写入 | ❌ **否** | 正常展示图片/链接 |

> ⚠️ 核心差异：**`null` 会被 Worker 重新消费，`"unavailable"` 不会。**
> 因此将某链接从失败状态重新纳入处理队列的唯一方法，就是把相关字段**从 `"unavailable"` 改回 `null`**——这是"刷新/重试"API 做的事。
>
> ⚠️ **preview 是特殊字段**：
> 1. 它不参与 `isReady()` 和 `atLeastOneFormatAvailable()` 的判定（这两个函数只看 image/pdf/readable/monolith）
> 2. 但它是前端**轮询启动的唯一判断依据**（Links.tsx 和 DashboardLinks.tsx 都只看 preview）
> 3. 管理员 `allBroken`（重新生成损坏链接）**不会处理 preview=unavailable 的情况**（详见第十一章）

### 3.2 三态在代码中的判断位置

**Worker 取任务的 where 条件（只认 null）**：
[getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/worker/lib/getLinkBatchFairly.ts#L35-L38)
```typescript
const baseLinkWhere: Prisma.LinkWhereInput = {
  url: { not: null },
  lastPreserved: null,   // ← 只有 lastPreserved 为 null 才会被取出处理
};
```

**前端判断某格式是否可用**：
[formatStats.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/packages/lib/formatStats.ts#L1-L20)
```typescript
export function formatAvailable(link, format) {
  return Boolean(link && link[format] && link[format] !== "unavailable");
}
// 返回 true 的情况：格式存在且不是 "unavailable" → 即只有真实路径才通过
```

**前端列表页轮询触发条件（认为"在队列中"）**：
[Links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkViews/Links.tsx#L405-L425)
```typescript
// preview 既不是 archives/ 开头（真实路径），也不是 "unavailable" → 视为 null/处理中
if (links?.some(e => !e.preview?.startsWith("archives") && e.preview !== "unavailable")) {
  interval = setInterval(..., 5000);
}
```

**详情页 isReady（只判断 image/pdf/readable/monolith，完全不看 preview）**：
[LinkDetails.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkDetails.tsx#L106-L114)
```typescript
const isReady = () => {
  return link &&
    (collectionOwner.archiveAsScreenshot ? link.pdf : true) &&
    (collectionOwner.archiveAsMonolith ? link.monolith : true) &&
    (collectionOwner.archiveAsPDF ? link.pdf : true) &&
    link.readable;
  // ↑ 四个判断：pdf（对应截图和PDF两种格式）、monolith、readable
  // ↑ 注意：**完全不包含 preview**
  // ↑ 只要非空就算 ready，"unavailable" 也算 ready
};
```

**atLeastOneFormatAvailable（同样不包含 preview）**：
[formatStats.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/packages/lib/formatStats.ts#L11-L20)
```typescript
export const atLeastOneFormatAvailable = (link) => {
  return (
    formatAvailable(link, "image") ||
    formatAvailable(link, "pdf") ||
    formatAvailable(link, "readable") ||
    formatAvailable(link, "monolith")
    // ↑ 四个格式，**完全不包含 preview**
  );
};
```

因此 preview 无论是 null 还是 "unavailable"，都不影响 LinkDetails 的三态分支判断。

**Worker 统计 failed（全部四个格式均为 unavailable）**：
[getWorkerStats.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/lib/api/controllers/worker/getWorkerStats.ts#L25-L34)
```typescript
const linkFailed = await prisma.link.count({
  where: {
    url: { not: null },
    lastPreserved: { not: null },
    image: "unavailable",
    pdf: "unavailable",
    readable: "unavailable",
    monolith: "unavailable",
  },
});
```

**Worker 统计 pending（lastPreserved 为 null）**：
[getWorkerStats.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/lib/api/controllers/worker/getWorkerStats.ts#L6-L11)
```typescript
const linkPending = await prisma.link.count({
  where: { url: { not: null }, lastPreserved: null },
});
```

### 3.3 失败写入 unavailable 的代码位置

[archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/worker/lib/archiveHandler.ts#L205-L225)

无论 try 成功还是 catch 抛错，finally 都会执行：
```typescript
finally {
  await prisma.link.update({
    where: { id: link.id },
    data: {
      lastPreserved: new Date().toISOString(),   // ← 同时把 lastPreserved 设为非 null
      readable: !finalLink.readable ? "unavailable" : undefined,
      image:    !finalLink.image    ? "unavailable" : undefined,
      monolith: !finalLink.monolith ? "unavailable" : undefined,
      pdf:      !finalLink.pdf      ? "unavailable" : undefined,
      preview:  !finalLink.preview  ? "unavailable" : undefined,
      indexVersion: null,
    },
  });
}
```

`lastPreserved` 被设置为当前时间 → 此链接从 Worker 队列中永久移出（除非人工刷新把它改回 null）。

---

## 四、Worker 失败后无自动重试机制

### 4.1 linkProcessing 循环的 catch 只打日志

[linkProcessing.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/worker/workers/linkProcessing.ts#L45-L73)

```typescript
const archiveLink = async (link) => {
  try {
    await archiveHandler(link, browser);
    console.log(`Succeeded processing link ${link.url}...`);
  } catch (error: any) {
    console.error(`Error processing link ${link.url}...:`, error);  // ← 仅打日志
    if (!browser.isConnected?.()) {
      await restartBrowser("browser disconnected");  // ← 唯一的"重试"：浏览器断了就重启浏览器
    }
  }
};

const processingPromises = links.map((e) => archiveLink(e));
await Promise.allSettled(processingPromises);  // ← 所有链接处理完就进入下一轮
```

关键点：
1. **没有重试队列**：单个链接失败后 catch 只 `console.error`，不会把它重新塞回队列、不会记录重试次数、不会有指数退避。
2. **没有重试计数**：数据库里没有 `retryCount`、`nextRetryAt` 之类的字段。
3. **finally 已经写了 lastPreserved 和 unavailable**：archiveHandler 内部的 finally 在 catch 向外抛出之前就已经把字段标记为失败状态，所以即使外层想重试也找不到这个链接了。
4. **浏览器重启 ≠ 链接重试**：只有当检测到浏览器 websocket 断开时才重启浏览器，但这不会重新处理刚才失败的链接——下一轮循环取的是新的 `lastPreserved: null` 链接。

### 4.2 Worker "重新取任务"的唯一标准：lastPreserved = null

[getLinkBatchFairly.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/worker/lib/getLinkBatchFairly.ts#L35-L38) 已经写死：
```typescript
lastPreserved: null
```

Worker 不会扫描 `image = "unavailable"` 的链接做自动重试，因为它们的 `lastPreserved` 已经被 finally 设为非 null。

**结论：Worker 端的失败是"终态"，必须通过前端手动触发刷新 API，把 lastPreserved 和对应格式字段改回 null，才能让 Worker 重新处理。**

---

## 五、三条刷新/重试入口的真实串联

一共有 **4 个**入口（之前写的 3 个加上"已选链接批量刷新"），对应 **3 条后端 API**：

| 入口 | 前端组件 | 调用 API | 清空字段策略 |
|---|---|---|---|
| 单条刷新（卡片菜单） | [LinkActions.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkActions.tsx#L59-L76) | `PUT /api/v1/links/{id}/archive` | 全部 6 个字段 → null |
| 单条刷新（详情页按钮） | [LinkDetails.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkDetails.tsx#L480-L501) / [LinkModal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/ModalContent/LinkModal.tsx#L123-L139) | `PUT /api/v1/links/{id}/archive` | 全部 6 个字段 → null |
| **已选链接批量刷新** | [LinkListOptions.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkListOptions.tsx#L93-L115) | `DELETE /api/v1/links/archive` | 全部 6 个字段 → null |
| 管理员"全部损坏 / 全部" | [background-jobs.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/pages/admin/background-jobs.tsx#L16-L204) | `DELETE /api/v1/worker/preservation` | 只把 unavailable 的改为 null（有选择性） |

### 5.1 入口 A：单条刷新 — PUT /api/v1/links/{id}/archive

[archive/[id]/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/pages/api/v1/links/%5Bid%5D/archive/index.ts#L39-L72)

```typescript
if (req.method === "PUT") {
  if (!link.url || !isValidUrl(link.url))
    return res.status(200).json({ response: "Invalid URL." });

  await prisma.link.update({
    where: { id: link.id },
    data: {
      image: null,
      pdf: null,
      readable: null,
      monolith: null,
      preview: null,
      lastPreserved: null,   // ← 关键：置 null 后 Worker 下一轮就会取到
      indexVersion: null,
      clientSide: false,
    },
  });

  await removeFiles(link.id, link.collection.id);  // 删除磁盘上的旧文件

  return res.status(200).json({ response: "Link is being archived." });
}
```

前端调用（[LinkActions.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkActions.tsx#L59-L76)）：
```typescript
const updateArchive = async () => {
  const load = toast.loading(t("sending_request"));
  const response = await fetch(`/api/v1/links/${link?.id}/archive`, { method: "PUT" });
  const data = await response.json();
  toast.dismiss(load);

  if (response.ok) {
    refetch();
    toast.success(t("link_being_archived"));
  } else {
    toast.error(data.response);  // 后端英文直显，不走 t()
  }
};
```

### 5.2 入口 B：已选链接批量刷新 — DELETE /api/v1/links/archive

**这是用户最常用的批量路径**，之前的分析遗漏了。

前端触发点（[LinkListOptions.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkListOptions.tsx#L93-L115)）：

```typescript
const bulkRefreshPreservations = async () => {
  const load = toast.loading(t("sending_request"));
  const ids = Object.keys(selectedIds).map(Number);   // 从 useLinkStore 取勾选的链接 ID

  await refreshPreservations.mutateAsync(
    { linkIds: ids },
    {
      onSettled: (data, error) => {
        toast.dismiss(load);
        if (error) {
          toast.error(error.message);
        } else {
          clearSelected();
          setEditMode?.(false);
          toast.success(t("links_being_archived"));   // 注意这里是复数 links_being_archived
        }
      },
    }
  );
};
```

UI 触发位置（[LinkListOptions.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkListOptions.tsx#L176-L273)）：

```
编辑模式（✎ 铅笔图标开启）
  → 出现复选框，勾选多条
  → 工具栏出现 ↻ 图标 (bi-arrow-clockwise)，tooltip=refresh_preserved_formats
  → 点击 → ConfirmationModal
  → 确认后调用 bulkRefreshPreservations()
```

Router hook 实现（[links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/packages/router/links.tsx#L1097-L1122)）：

```typescript
const useArchiveAction = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (payload) => {
      const response = await fetch("/api/v1/links/archive", {
        body: JSON.stringify({ linkIds: payload.linkIds }),
        method: "DELETE",                       // ← 注意是 DELETE，不是 PUT
        headers: { "Content-Type": "application/json" },
      });
      const data = await response.json();
      if (!response.ok) throw new Error(data.response);
      return data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["links"] });
      queryClient.invalidateQueries({ queryKey: ["dashboardData"] });
    },
  });
};
```

后端处理（[links/archive/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/pages/api/v1/links/archive/index.ts#L11-L90)）：

```typescript
if (req.method === "DELETE") {
  const dataValidation = LinkArchiveActionSchema.safeParse(req.body);
  // ...

  const { linkIds } = dataValidation.data;

  if (linkIds) {
    // 权限检查：只能操作自己拥有或有 canDelete 的 collection 内的链接
    const authorizedLinks = await prisma.link.findMany({
      where: {
        id: { in: linkIds },
        url: { not: null },
        OR: [
          { collection: { ownerId: user.id } },
          { collection: { members: { some: { userId: user.id, canDelete: true } } } },
        ],
      },
      select: { id: true, collectionId: true },
    });

    if (authorizedLinks.length === 0) {
      return res.status(401).json({ response: "Permission denied." });
    }

    res.status(200).json({ response: "Success." });  // ← 先返回 HTTP 200

    // 再异步处理（用户不等待）
    for (const link of authorizedLinks) {
      await removeFiles(link.id, link.collectionId);  // 删磁盘文件
      await prisma.link.update({
        where: { id: link.id },
        data: {
          image: null,
          pdf: null,
          readable: null,
          monolith: null,
          preview: null,
          lastPreserved: null,   // ← 置 null，Worker 下一轮取到
          indexVersion: null,
        },
      });
      console.log("Deleted preservation link:", link.id);
    }
    return;
  }
}
```

与单条 PUT 的对比：
| 维度 | 单条 PUT /{id}/archive | 批量 DELETE /links/archive |
|---|---|---|
| HTTP Method | PUT | DELETE |
| 返回时机 | 处理完后返回 | 先 200，再异步处理每条 |
| 权限粒度 | 单链接 getPermission（canUpdate） | 批量 where + OR（owner 或 canDelete） |
| 清空字段 | 相同（6 字段 + clientSide → null） | 相同（6 字段 → null，无 clientSide） |
| removeFiles | 是 | 是 |
| 无效 URL 提前检查 | 有（返回 "Invalid URL."） | 无（url 已在 authorizedLinks where 中过滤 not null） |

### 5.3 入口 C：管理员级刷新 — DELETE /api/v1/worker/preservation

[preservation.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L18-L166)

需要 server admin 权限（`user.id === NEXT_PUBLIC_ADMIN`）。有两个 action：

**action = "allAndRePreserve"（重新生成全部链接）**：
```typescript
// 取自己所有 url 类型链接，全部字段设为 null
const allLinks = await prisma.link.findMany({
  where: { collection: { ownerId: user.id }, type: "url", url: { not: null } },
});
for (const link of allLinks) {
  await removeFiles(link.id, link.collectionId);
  await prisma.link.update({
    where: { id: link.id },
    data: {
      image: null, pdf: null, readable: null, monolith: null, preview: null,
      lastPreserved: null, indexVersion: null,
    },
  });
}
```
等价于对自己的所有链接执行一次"批量刷新"。

**action = "allBroken"（只重新生成损坏的链接）**：
```typescript
// 先找任一格式为 "unavailable" 的链接
const brokenArchives = await prisma.link.findMany({
  where: {
    type: "url", url: { not: null },
    collection: { ownerId: user.id },
    OR: [
      { image: "unavailable" }, { pdf: "unavailable" },
      { readable: "unavailable" }, { monolith: "unavailable" },
      { preview: "unavailable" },
    ],
  },
  include: { createdBy: { select: { archiveAsScreenshot, ... } }, tags: true },
});

for (const link of brokenArchives) {
  // 根据用户设置或标签，判断这个"unavailable"的格式是否是用户真想要的
  const needsReprocessing =
    (link.image === "unavailable" && shouldArchive.archiveAsScreenshot) ||
    (link.monolith === "unavailable" && shouldArchive.archiveAsMonolith) ||
    (link.pdf === "unavailable" && shouldArchive.archiveAsPDF) ||
    (link.readable === "unavailable" && shouldArchive.archiveAsReadable);

  if (needsReprocessing) {
    await prisma.link.update({
      where: { id: link.id },
      data: {
        // 只把用户开启且当前为 unavailable 的那些字段改为 null
        image:    shouldArchive.archiveAsScreenshot && link.image === "unavailable"    ? null : link.image,
        pdf:      shouldArchive.archiveAsPDF && link.pdf === "unavailable"              ? null : link.pdf,
        readable: shouldArchive.archiveAsReadable && link.readable === "unavailable"    ? null : link.readable,
        monolith: shouldArchive.archiveAsMonolith && link.monolith === "unavailable"    ? null : link.monolith,
        lastPreserved: null,  // ← 仍然要置 null，否则 Worker 取不到
        indexVersion: null,
      },
    });
  }
}
```

与"批量 DELETE /api/v1/links/archive"的关键差异：
- **allBroken 不删文件**（没有调用 `removeFiles`），只把 DB 字段改回 null。
- **allBroken 有选择性**：只把用户需要（根据用户设置和归档标签判断）且已失败的那些格式改为 null，不影响已成功的格式。
- **⚠️ allBroken 不会处理 preview**：brokenArchives 查询的 OR 条件包含 `{ preview: "unavailable" }`（能把只 preview 失败的链接捞出来），但 `needsReprocessing` 判断和 `data` 更新对象里完全没有 preview 字段。因此如果一个链接只有 preview = unavailable、其余格式都成功，它会被查出来但 needsReprocessing = false，什么也不做（详见第十一章）。
- **allAndRePreserve 与批量 DELETE 效果相同**：全部字段（包括 preview）→ null + 删文件。

### 5.4 三条 API 的"重新排队"流程总结

所有刷新 API 最终都做同一件事——**把 `lastPreserved` 改回 `null`**，从而让 Worker 的 `getLinkBatchFairly` 在下一轮循环里把这条链接取出来重新处理。

流程链路：
```
用户点击刷新
  ↓
API 把 lastPreserved + 各格式字段 → null，同时删磁盘文件（某些场景）
  ↓
HTTP 200 返回前端
  ↓
前端 toast.success("links_being_archived") + invalidateQueries(["links"])
  ↓
Links.tsx useEffect 检测到 preview=null/undefined，启动 5s 轮询
  ↓
Worker 下一轮 linkProcessing → getLinkBatchFairly(where lastPreserved = null) 取到该链接
  ↓
archiveHandler 执行 → finally 写 "unavailable" 或 真实路径
  ↓
前端轮询拉取到最新状态 → 轮询停止
```

---

## 六、异步归档失败处理流程

### 6.1 Worker 端归档处理

[archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/worker/lib/archiveHandler.ts#L25-L231)

```typescript
try {
  await Promise.race([
    (async () => { /* screenshot/pdf/readable/monolith 等归档操作 */ })(),
    timeoutPromise,  // BROWSER_TIMEOUT 分钟超时
  ]);
} catch (err) {
  console.log("Failed Link:", link.url);
  console.log("Reason:", err);
  throw err;  // 向外抛出，由 linkProcessing catch 打日志
} finally {
  // 无论成功失败，最后都更新数据库
  await prisma.link.update({
    where: { id: link.id },
    data: {
      lastPreserved: new Date().toISOString(),
      readable: !finalLink.readable ? "unavailable" : undefined,
      image:    !finalLink.image    ? "unavailable" : undefined,
      monolith: !finalLink.monolith ? "unavailable" : undefined,
      pdf:      !finalLink.pdf      ? "unavailable" : undefined,
      preview:  !finalLink.preview  ? "unavailable" : undefined,
      indexVersion: null,
    },
  });
}
```

**失败标记机制**：失败的格式字段被设置为字符串 `"unavailable"`，而不是 null。`lastPreserved` 被设置为当前时间，表示"此链接已从队列中取出并处理过"。

### 6.2 页面三种状态渲染

[LinkDetails.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkDetails.tsx#L560-L593)

| 状态 | 条件 | 渲染内容 |
|---|---|---|
| **全部处理中** | `!isReady() && !atLeastOneFormatAvailable(link)` | 大加载动画 + `preservation_in_queue` + `check_back_later` |
| **部分就绪** | `!isReady() && atLeastOneFormatAvailable(link)` | 小加载动画 + `there_are_more_formats` + `check_back_later` |
| **全部完成** | `isReady()` | 展示可用格式（不可用的被 `formatAvailable` 隐藏） |

注意：
1. `isReady()` 和 `atLeastOneFormatAvailable()` **完全不包含 preview**——只看 image/pdf/readable/monolith。preview 为 null 或 unavailable 都不影响详情页三态分支的选择。
2. 当所有格式都是 `"unavailable"` 时，`isReady()` 返回 true（非空即可），`atLeastOneFormatAvailable()` 返回 false（`"unavailable"` 被判定为不可用），但因为 `isReady()` 已为 true，页面直接展示"全部完成"态——这意味着**全失败的链接看起来和全部成功的一样，只是所有格式行都被隐藏了**，用户只能通过缺少内容来推测失败。
3. 前端列表页轮询**只看 preview**，与详情页三态的判断来源完全不同。

### 6.3 列表页的自动轮询

[Links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkViews/Links.tsx#L405-L425) 和 [DashboardLinks.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/DashboardLinks.tsx#L110-L130) 轮询逻辑完全一致。

```typescript
useEffect(() => {
  let interval = null;
  // ⚠️ 只看 preview！image/pdf/readable/monolith 一概不看
  if (links?.some(e => !e.preview?.startsWith("archives") && e.preview !== "unavailable")) {
    interval = setInterval(async () => {
      useData.refetch();
    }, 5000);
  }
  return () => { if (interval) clearInterval(interval); };
}, [links]);
```

轮询条件解读：
- `preview` 既不是 `archives/` 开头（真实路径），也不是 `"unavailable"` → 视为 `null`/处理中 → 启动轮询
- `preview = "unavailable"` → 视为终态（和真实路径一样）→ 不轮询
- ⚠️ **即使 image/pdf/readable/monolith 全部仍为 null（在队列中），只要 preview 已完成或已 unavailable，轮询就不会启动**。这意味着 preview 是前端轮询的唯一哨兵。

这是"队列中"状态自动消失的机制——持续轮询直到 preview 被标记为已完成路径或 `"unavailable"`。

---

## 七、Toast 提示系统

### 7.1 全局配置

[_app.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/pages/_app.tsx#L80-L108)

```typescript
import toast from "react-hot-toast";
import { Toaster, ToastBar } from "react-hot-toast";

<Toaster position="top-center" reverseOrder={false} toastOptions={{ className: "..." }}>
  {(t) => (
    <ToastBar toast={t}>
      {({ icon, message }) => (
        <div data-testid="toast-message-container" data-type={t.type}>
          {icon}
          <span data-testid="toast-message">{message}</span>
          {t.type !== "loading" && <div onClick={() => toast.dismiss(t.id)} />}
        </div>
      )}
    </ToastBar>
  )}
</Toaster>
```

### 7.2 Toast 使用模式

| 模式 | 示例 | 位置 |
|---|---|---|
| **`toast.error(t(key))`** - 翻译后的错误 | `toast.error(t(error.message))` | Mutation onError（useAddLink 等） |
| **`toast.error(string)`** - 后端英文错误直显 | `toast.error(data.response)` | LinkActions.updateArchive、LinkListOptions.bulkRefreshPreservations |
| **`toast.success(t(key))`** - 翻译后的成功 | `toast.success(t("link_created"))` | 成功回调 |
| **`toast.loading(t(key))`** + `toast.dismiss(id)` | 见下 | 异步操作包裹 |

Loading + Dismiss 完整示例（[LinkActions.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkActions.tsx#L59-L76)）：

```typescript
const updateArchive = async () => {
  const load = toast.loading(t("sending_request"));
  const response = await fetch(`/api/v1/links/${link?.id}/archive`, { method: "PUT" });
  const data = await response.json();
  toast.dismiss(load);

  if (response.ok) {
    refetch();
    toast.success(t("link_being_archived"));
  } else {
    toast.error(data.response);  // 后端原始英文
  }
};
```

---

## 八、本地化文案（i18n）

### 8.1 文案文件位置

- 英文：[en/common.json](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/public/locales/en/common.json)
- 中文：[zh/common.json](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/public/locales/zh/common.json)

### 8.2 文案使用与错误消息的矛盾映射

⚠️ **这是最容易混淆的地方**：错误消息 message 字段有三种类型，处理方式不同：

| 类型 | 来源 | message 示例 | 是否传 `t()` | 结果 |
|---|---|---|---|---|
| **i18n key** | 前端 mutationFn 预校验 | `"invalid_url_guide"` | ✅ 是 | `t("invalid_url_guide")` → 正确翻译 |
| **英文硬编码** | 后端 Controller response | `"Link already exists"` | ❌ 仍然传了 `t()` | `t("Link already exists")` → 原样输出英文（key 不存在则 fallback 为 key 本身） |
| **英文硬编码** | Zod 校验前端 Modal | `"Error: Required [url]"` | ❌ 未传 `t()` | 原样显示英文 |

[useAddLink onError](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/packages/router/links.tsx#L521-L523)

```typescript
onError: (error, _variables, context) => {
  if (toast && t) toast.error(t(error.message));  // 两种类型都走 t()
  // ...
};
```

只有前端主动抛出的 `"invalid_url_guide"` 是真正的 i18n key，后端返回的英文错误通过 `t()` 后因找不到翻译而透传。

### 8.3 归档/异步相关 i18n key

| Key | 中文 | 英文 |
|---|---|---|
| `preservation_in_queue` | 链接保存正在处理中... | Link preservation is in the queue |
| `check_back_later` | 请稍后再查看结果 | Please check back later to see the result |
| `there_are_more_formats` | 队列中有更多已保存的格式。 | There are more preserved formats in the queue |
| `link_being_archived` | 链接正在归档... | Link is being archived... |
| `links_being_archived` | 链接正在归档…… | Links are being archived... |
| `refresh_preserved_formats` | 刷新保留格式 | Refresh Preserved Formats |
| `refresh_preserved_formats_confirmation_desc` | 您确定要刷新此链接保留的格式吗？ | Are you sure you want to refresh the preserved formats for this link? |
| `refresh_multiple_preserved_formats_confirmation_desc` | 您确定要刷新 {{count}} 个链接的已保存格式吗？ | Are you sure you want to refresh the preserved formats for {{count}} links? |
| `no_broken_preservations` | 未发现损坏的存档。 | No broken preservations. |
| `links_are_being_represerved` | 链接正在被重新保存…… | Links are being re-preserved... |
| `preview_unavailable` | 预览不可用 | Preview Unavailable |
| `sending_request` | 发送请求... | Sending Request... |
| `regenerate_broken_links` | 重新生成损坏的链接 | Regenerate Broken Links |
| `regenerate_all_links` | 重新生成全部链接 | Regenerate All Links |
| `link_selected` / `links_selected` | 已选择 1 条链接 / 已选择 N 条链接 | 1 link selected / N links selected |

---

## 九、完整调用链示例

### 示例 1：创建链接时 URL 格式错误（前端预校验 → toast i18n）

```
NewLinkModal.submit()
  └─ useAddLink.mutateAsync(link)
       └─ mutationFn
            └─ new URL("bad-url") 抛错
                 └─ throw new Error("invalid_url_guide")
       └─ onError(error, _, context)
            ├─ toast.error(t("invalid_url_guide"))  → 显示"请输入有效的链接地址..."
            └─ 回滚乐观更新
```

### 示例 2：创建重复链接（后端 409 → toast 英文直显）

```
NewLinkModal.submit()
  └─ useAddLink.mutateAsync(link)
       └─ mutationFn → POST /api/v1/links
            └─ Controller postLink → { response: "Link already exists", status: 409 }
            └─ response.ok = false → throw new Error("Link already exists")
       └─ onError
            └─ toast.error(t("Link already exists"))
                 → t() 找不到 key，fallback 显示英文 "Link already exists"
```

### 示例 3：已选链接批量刷新（DELETE /api/v1/links/archive → 重新排队）

```
LinkListOptions 编辑模式
  ├─ 用户勾选 N 条链接 → selectedIds store 更新
  ├─ 点击 ↻ 图标 → ConfirmationModal（refresh_multiple_preserved_formats_confirmation_desc）
  ├─ 确认 → bulkRefreshPreservations()
  │    ├─ toast.loading(t("sending_request"))
  │    ├─ useArchiveAction.mutateAsync({ linkIds })
  │    │    └─ DELETE /api/v1/links/archive  body: { linkIds: [...] }
  │    │         └─ 后端：校验权限 → 先返回 200 → 异步 for 循环每条链接：
  │    │              ├─ removeFiles(id, collectionId)
  │    │              └─ prisma.link.update({ lastPreserved: null, image/pdf/...: null })
  │    ├─ onSettled → toast.dismiss(load)
  │    ├─ toast.success(t("links_being_archived"))
  │    ├─ clearSelected() + setEditMode(false)
  │    └─ onSuccess → invalidateQueries(["links"], ["dashboardData"])
  └─ Links.tsx 重渲染
       └─ useEffect 检测到有 preview 为 null → 启动 5 秒轮询
            └─ Worker 下一轮 getLinkBatchFairly(lastPreserved = null) 取出这些链接
                 └─ archiveHandler 处理 → 写 "unavailable" 或路径
                      └─ 轮询拉取到最终状态 → 轮询停止
```

### 示例 4：单条链接刷新归档（PUT /api/v1/links/{id}/archive → toast + 轮询）

```
LinkActions → updateArchive()
  ├─ toast.loading(t("sending_request"))
  ├─ PUT /api/v1/links/{id}/archive
  │    └─ 后端：校验权限 + 有效 URL → 清空 6 字段为 null → removeFiles → 返回 200
  ├─ toast.dismiss(load)
  ├─ toast.success(t("link_being_archived"))
  └─ refetch() 立即刷新
       └─ Links.tsx useEffect 检测到 preview=null，启动 5s 轮询
            └─ Worker 处理中：预览 loading
            └─ Worker 成功/失败：字段变为路径 或 "unavailable"，轮询停止
```

### 示例 5：归档失败页面状态（"unavailable" → 隐藏项，无自动重试）

```
Worker archiveHandler 失败
  ├─ catch 中仅 console.error，不重试
  └─ finally 块 image/pdf/readable 等被写为 "unavailable"，lastPreserved=now

Worker 下一轮循环：getLinkBatchFairly 只取 lastPreserved=null → 不再取这条链接（永久失败，除非人工刷新）

前端 Links.tsx 轮询拉取
  └─ LinkDetails 渲染
       ├─ formatAvailable(link, "image") → false  → 隐藏截图行
       ├─ formatAvailable(link, "pdf")   → false  → 隐藏 PDF 行
       ├─ isReady() → true（字段非空即可）
       └─ atLeastOneFormatAvailable() → 取决于是否有任一格式是真实路径
```

### 示例 6：管理员只重新生成损坏的（allBroken 选择性置 null）

```
admin/background-jobs → 点击 regenerate_broken_links
  └─ DELETE /api/v1/worker/preservation  body: { action: "allBroken" }
       └─ 后端：找出任意格式为 "unavailable" 的链接
            └─ 对每条链接：根据用户的 archiveAsScreenshot/PDF/... 设置
                 └─ 如果 "用户开启了该格式" 且 "该格式当前为 unavailable"
                      └─ 把该字段改为 null，同时 lastPreserved 改为 null
       └─ （注意：不删除磁盘文件，不影响已成功的格式）
```

---

## 十、容易混淆的要点总结

| 混淆点 | 真相 |
|---|---|
| `toast.error(t(error.message))` 的 message 都是 i18n key？ | ❌ 只有前端主动抛出的少量是 key（如 `invalid_url_guide`），后端返回的全是英文硬编码，`t()` 找不到就原样输出 |
| `"unavailable"` 和 `null` 含义相同？ | ❌ `null` = 在队列中未处理，**Worker 会取**；`"unavailable"` = 已处理但失败，**Worker 不会再取**。前者触发轮询，后者不 |
| Worker 失败后会自动重试吗？ | ❌ 不会。linkProcessing 的 catch 只打 console.error，finally 已经把 lastPreserved 写为非 null，Worker 下一轮取不到这条链接。必须人工通过刷新 API 把 lastPreserved 改回 null |
| `isReady()` 判断归档完成，包含 preview 吗？ | ❌ **完全不包含 preview**。isReady() 只看 pdf（对应截图和PDF）、monolith、readable 四个判断；atLeastOneFormatAvailable() 同样不含 preview。preview 为 null 或 unavailable 都不影响详情页三态 |
| `isReady()` 如何判断"完成"？ | ❌ 只判断字段非空。全部都是 `"unavailable"` 也会被判定为 ready，此时页面看起来"完成了"但所有格式行都被隐藏 |
| `formatAvailable()` 参与页面三态判断？ | ✅ 三态用 `isReady()` + `atLeastOneFormatAvailable()`，后者又依赖 `formatAvailable()` |
| 前端轮询启动的判断依据是什么？ | **只看 preview**（Links.tsx 和 DashboardLinks.tsx 都只检查 preview）——`!preview.startsWith("archives") && preview !== "unavailable"` 才启动轮询。和 isReady/atLeastOneFormatAvailable 的判断源完全不同 |
| 乐观更新失败时 toast 和回滚谁先执行？ | 先 toast，后回滚缓存。在 `onError` 中顺序执行 |
| 刷新重试的入口有几个？ | **4 个**：列表卡片下拉、详情页按钮（这两个走单条 PUT）；已选链接批量（走 DELETE /api/v1/links/archive）；后台 Background Jobs 的全部损坏/全部（走 DELETE /api/v1/worker/preservation） |
| 批量刷新和单条刷新除了条数还有什么区别？ | 批量 DELETE 先返回 200 再异步处理，单条 PUT 处理完后才返回；批量权限校验用 canDelete，单条用 canUpdate；批量不检查 URL 有效性（where 已过滤） |
| allBroken 和 allAndRePreserve 的区别？ | allAndRePreserve 把所有链接的所有格式（含 preview）都置 null（且删文件），等价于全量刷新；allBroken 只把用户开启且当前为 unavailable 的 image/pdf/readable/monolith 置 null（不删文件），**不处理 preview** |
| allBroken 能修复 preview=unavailable 的链接吗？ | ❌ **不能**。brokenArchives 查询的 OR 条件包含 preview=unavailable（能查出来），但 needsReprocessing 和 update data 中完全没有 preview 字段。若链接只有 preview 失败、其余四个都成功 → needsReprocessing = false → 什么都不做。只能走单条刷新、批量已选刷新、或 allAndRePreserve |
| handleArchivePreview 的 buffer 超限会抛异常吗？ | ❌ 不会。screenshot.then() 里 buffer 超限只是 `return console.log(...)`，Promise 正常 resolve，handleArchivePreview 正常返回（静默失败），最后 finally 兜底把 preview 写为 unavailable。只有 page.screenshot() 本身 reject 才会向外抛异常 |
| 让 Worker 重新处理一条链接的唯一手段是什么？ | **把 lastPreserved 改回 null**。Worker 的取任务条件 hardcode 为 `lastPreserved: null`，改其他字段都没用 |

---

## 十一、allBroken 重生成损坏归档的 preview 处理细节

### 11.1 brokenArchives 为何会查到 preview = unavailable

[preservation.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L67-L84)

```typescript
const brokenArchives = await prisma.link.findMany({
  where: {
    type: "url",
    url: { not: null },
    collection: { ownerId: user.id },
    OR: [
      { image: "unavailable" },
      { pdf: "unavailable" },
      { readable: "unavailable" },
      { monolith: "unavailable" },
      { preview: "unavailable" },   // ← preview 被纳入损坏判断
    ],
  },
  // ...
});
```

查询条件的 `OR` 数组包含了五个字段，其中就有 `{ preview: "unavailable" }`。这意味着：

- 只要任一格式（image/pdf/readable/monolith/**preview**）是 `"unavailable"`，该链接就会被视为"损坏归档"而查出来。
- 即使 image/pdf/readable/monolith 四个全部成功，**只有 preview 为 unavailable**，也会被这条查询捞出来。

preview 为什么会变成 unavailable？有两条路径：

**路径 1：skipPreservation 早期退出**（[archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/worker/lib/archiveHandler.ts#L44-L61)）
```typescript
if (skipPreservation || (!link.url?.startsWith("http://") && !link.url?.startsWith("https://"))) {
  await prisma.link.update({
    where: { id: link.id },
    data: {
      lastPreserved: new Date().toISOString(),
      readable: "unavailable",
      image: "unavailable",
      monolith: "unavailable",
      pdf: "unavailable",
      preview: "unavailable",   // ← 五个字段一起写 unavailable
      indexVersion: null,
    },
  });
  return;
}
```
此时五个字段（包括 preview）会一起被写 unavailable。

**路径 2：finally 兜底**（[archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/worker/lib/archiveHandler.ts#L212-L224)）
```typescript
const finalLink = await prisma.link.findUnique({ where: { id: link.id } });
if (finalLink) {
  await prisma.link.update({
    where: { id: link.id },
    data: {
      lastPreserved: new Date().toISOString(),
      readable: !finalLink.readable ? "unavailable" : undefined,
      image:    !finalLink.image    ? "unavailable" : undefined,
      monolith: !finalLink.monolith ? "unavailable" : undefined,
      pdf:      !finalLink.pdf      ? "unavailable" : undefined,
      preview:  !finalLink.preview  ? "unavailable" : undefined,  // ← preview 单独判空
      indexVersion: null,
    },
  });
}
```
这里 preview 是**单独判断**的——`!finalLink.preview` 为 true 就写 unavailable。而 preview 的值是否被写入，取决于 `handleArchivePreview` 是否成功执行。

### 11.2 handleArchivePreview 何时会失败导致 preview = unavailable

[handleArchivePreview.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts#L17-L82)

preview 生成有两条分支，都可能失败：

**分支 1：尝试用 og:image 作为预览图**
```typescript
let ogImageUrl = await page.evaluate(() => {
  const metaTag = document.querySelector('meta[property="og:image"]');
  return metaTag ? (metaTag as any).content : null;
});

if (ogImageUrl) {
  try {
    await assertUrlIsSafeForServerSideFetch(ogImageUrl);  // SSRF 检查可能抛错
    const imageResponse = await page.goto(ogImageUrl);      // 跳转可能失败
    if (imageResponse && !link.preview?.startsWith("archive")) {
      const buffer = await imageResponse.body();
      previewGenerated = await generatePreview(buffer, link.collectionId, link.id);
    }
    await page.goBack();
  } catch (error) {
    if (!(error instanceof UnsafeUrlError)) {
      throw error;  // 非 SSRF 错误向外抛，可能导致整个 archiveHandler 进入 catch
    }
    // 如果是 UnsafeUrlError，则静默继续，走分支 2
  }
}
```

**分支 2：og:image 不存在或失败，直接对页面截图**
```typescript
if (!previewGenerated && !link.preview?.startsWith("archive")) {
  await page
    .screenshot({ type: "jpeg", quality: 20 })
    .then(async (screenshot) => {
      if (
        Buffer.byteLength(screenshot) >
        1024 * 1024 * Number(process.env.PREVIEW_MAX_BUFFER || 10)
      )
        return console.log("Error generating preview: Buffer size exceeded");

      await createFile({
        data: screenshot,
        filePath: `archives/preview/${link.collectionId}/${link.id}.jpeg`,
      });

      await prisma.link.update({
        where: { id: link.id },
        data: {
          preview: `archives/preview/${link.collectionId}/${link.id}.jpeg`,
        },
      });
    });
  // 注意：外层有 await！await 包裹整个 page.screenshot().then(...) Promise 链
  // 路径 A：page.screenshot() 本身 reject → 整个 Promise 链 reject → await 抛出 →
  //    handleArchivePreview 抛错 → archiveHandler catch 捕获 → finally 执行
  // 路径 B：screenshot 成功但 buffer 超限 → .then() 回调里 return console.log(...) →
  //    Promise 正常 resolve（值为 undefined）→ handleArchivePreview 正常返回、不抛错 →
  //    但 preview 字段仍为 null → finally 把它写为 "unavailable"（静默失败，只有 console.log）
}
```

关键点：**`await` 包裹了整个 `page.screenshot().then(...)` 链**，所以 screenshot reject 会向外层传播。但 buffer 超限是 `.then()` 回调内部的 `return`，不抛异常，属于**静默失败**——函数正常返回，preview 没写入 DB，最后 finally 兜底写 unavailable。

**而 preview 与其他四个格式不同之处**：它不受用户 `archivalSettings` 控制——不管用户有没有开启 archiveAsScreenshot/PDF，preview 都会无条件尝试生成（[archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/worker/lib/archiveHandler.ts#L168-L169)）：
```typescript
// Preview —— 无条件执行！
if (!link.preview) await handleArchivePreview(link, page);
```

其他四个格式都有条件判断，例如：
```typescript
// Readability —— 受 archiveAsReadable 控制
if (archivalSettings.archiveAsReadable && !link.readable)
  await handleReadability(content, link);

// Screenshot/PDF —— 受 archiveAsScreenshot / archiveAsPDF 控制
if ((archivalSettings.archiveAsScreenshot && !link.image) || ...)
  await handleScreenshotAndPdf(link, page, archivalSettings);
```

这意味着 **preview 永远会尝试生成**，而且永远可能单独失败。

### 11.3 needsReprocessing 为何不把 preview 纳入重排

[preservation.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/pages/api/v1/worker/preservation.tsx#L127-L160)

```typescript
const needsReprocessing =
  (link.image === "unavailable" && shouldArchive.archiveAsScreenshot) ||
  (link.monolith === "unavailable" && shouldArchive.archiveAsMonolith) ||
  (link.pdf === "unavailable" && shouldArchive.archiveAsPDF) ||
  (link.readable === "unavailable" && shouldArchive.archiveAsReadable);
// ↑ 只有 4 个字段，**没有 preview**
```

```typescript
if (needsReprocessing) {
  await prisma.link.update({
    where: { id: link.id },
    data: {
      image:    shouldArchive.archiveAsScreenshot && link.image === "unavailable"    ? null : link.image,
      pdf:      shouldArchive.archiveAsPDF && link.pdf === "unavailable"              ? null : link.pdf,
      readable: shouldArchive.archiveAsReadable && link.readable === "unavailable"    ? null : link.readable,
      monolith: shouldArchive.archiveAsMonolith && link.monolith === "unavailable"    ? null : link.monolith,
      lastPreserved: null,
      indexVersion: null,
      // ↑ 依然没有 preview 字段
    },
  });
}
```

`shouldArchive` 的类型是 `Omit<ArchivalSettings, "aiTag" | "archiveAsWaybackMachine">`，里面只有：
- `archiveAsScreenshot`
- `archiveAsMonolith`
- `archiveAsPDF`
- `archiveAsReadable`

**完全没有 `archiveAsPreview` 这个概念。** 原因在 11.2 已经说明：preview 是无条件生成的，不受用户开关控制，所以 `ArchivalSettings` 里本来就没有它。

这导致了如下不对称：

| 字段 | brokenArchives OR 条件能捞出来？ | needsReprocessing 判断？ | update 里能置 null？ |
|---|---|---|---|
| image | ✅ | ✅ | ✅ |
| pdf | ✅ | ✅ | ✅ |
| readable | ✅ | ✅ | ✅ |
| monolith | ✅ | ✅ | ✅ |
| **preview** | ✅ | ❌ **没有** | ❌ **没有** |

### 11.4 这个不对称对三态和前端轮询的影响

#### 场景：只有 preview 失败，其余四个都成功

假设一个链接经过 Worker 处理后状态为：
```
image: "archives/1/123.png"      (成功)
pdf: "archives/1/123.pdf"         (成功)
readable: "archives/1/123.html"   (成功)
monolith: "archives/1/123.mhtml"  (成功)
preview: "unavailable"            (失败)
lastPreserved: "2026-06-08T10:00:00.000Z"
```

**1. brokenArchives 查询会把它捞出来**（OR 条件匹配 `preview: "unavailable"`）

**2. needsReprocessing 计算为 false**
- `link.image === "unavailable"` → false
- `link.pdf === "unavailable"` → false
- `link.readable === "unavailable"` → false
- `link.monolith === "unavailable"` → false
- 结果：false → **整个 if 分支跳过，不做任何事**

**3. preview 永远停留在 "unavailable"**
- 没有任何代码把它改回 `null`
- `lastPreserved` 也保持非 null → Worker 不会重新取这条链接
- 结果：**preview 永久为 unavailable，除非用户手动走其他刷新路径**

#### 对三态状态的影响

| 维度 | 值 | 含义 |
|---|---|---|
| Worker 取任务？ | ❌ lastPreserved ≠ null | 不会被重新处理 |
| `formatAvailable(link, "preview")` | ❌ preview = "unavailable" | 认为该格式不可用 |
| `isReady()` 判断 | ✅ true（preview 不参与判断） | isReady() 只看 pdf/monolith/readable，不看 preview → 认为"准备就绪" |
| `atLeastOneFormatAvailable()` | ✅ true（image/pdf 等有真实路径） | 同样不包含 preview → 认为至少有一个可用 |
| 详情页状态 | 走"全部完成"分支 | 因为 isReady() = true，直接渲染完成态 |
| 前端列表轮询 | ❌ preview = "unavailable" | 轮询不启动 |

**4. 前端轮询为什么不启动？**

[Links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkViews/Links.tsx#L405-L425) 和 [DashboardLinks.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/DashboardLinks.tsx#L110-L130) 轮询条件完全一致：

```typescript
if (
  links?.some((e) =>
    !e.preview?.startsWith("archives") && e.preview !== "unavailable"
  )
) {
  interval = setInterval(..., 5000);
}
```

轮询启动的充要条件：`preview` **既不是** `archives/` 开头的真实路径，**也不是** `"unavailable"`。

当 preview = `"unavailable"` 时：
- `!e.preview?.startsWith("archives")` → true
- `e.preview !== "unavailable"` → **false**
- AND → false → 轮询**不启动**

这就是设计意图：`"unavailable"` 被视为"终态"，和真实路径一样不需要再轮询。但问题在于 allBroken 无法把它从终态拉回来。

#### 前端 UI 上的表现（三态渲染）

以 [LinkCard.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkCard.tsx#L98-L123) 为例，preview 三态分支：

```tsx
{formatAvailable(link, "preview") ? (
  // 态 A：真实路径 → 显示预览图
  <Image src={`/api/v1/archives/${link.id}?format=jpeg&preview=true...`} />
) : link.preview === "unavailable" ? (
  // 态 B：unavailable → 显示灰色空块（没有任何文字提示）
  <div className={`bg-gray-50 ${imageHeightClass} bg-opacity-80`}></div>
) : (
  // 态 C：null → 显示骨架屏（loading）
  <div className={`${imageHeightClass} bg-opacity-80 skeleton rounded-none`}></div>
)}
```

用户看到的是：卡片顶部突然变成了一块安静的灰色区域，没有任何错误提示，也没有加载动画——和"用户本来就没开预览图"视觉上几乎无法区分。

### 11.5 对比：其他刷新路径对 preview 的处理

| 刷新 API | 是否把 preview 置 null？ | 是否把 lastPreserved 置 null？ |
|---|---|---|
| `PUT /api/v1/links/{id}/archive`（单条刷新） | ✅ 是 | ✅ 是 |
| `DELETE /api/v1/links/archive`（已选链接批量） | ✅ 是 | ✅ 是 |
| `DELETE /api/v1/worker/preservation` + action=allAndRePreserve | ✅ 是 | ✅ 是 |
| `DELETE /api/v1/worker/preservation` + action=allBroken | ❌ **否** | ❌ 仅当 needsReprocessing=true 时才置 null |

结论：
- 单条刷新、批量已选刷新、全量刷新都会把 preview 改回 null → Worker 会重新生成 preview
- **只有 allBroken 不管 preview**，导致"仅 preview 损坏"的链接无法被 allBroken 修复
- 用户如果发现某条链接 preview 是灰色块，只能通过单条刷新、勾选后批量刷新、或 allAndRePreserve 全量重跑三条路径来修复
