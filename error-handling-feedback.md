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

## 三、异步归档失败处理流程

### 3.1 Worker 端归档处理

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
  throw err;  // 向外抛出，由 Worker 上层决定重试
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

**失败标记机制**：失败的格式字段被设置为字符串 `"unavailable"`，而不是 null。

### 3.2 前端状态判断

[formatStats.ts](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/packages/lib/formatStats.ts#L1-L20)

```typescript
export function formatAvailable(link, format) {
  return Boolean(link && link[format] && link[format] !== "unavailable");
}
```

判断逻辑三态：

| 字段值 | 含义 | 显示 |
|---|---|---|
| `null` / `undefined` / `""` | 尚未处理（在队列中） | 加载动画 |
| `"unavailable"` | 处理失败 | 不显示该项 |
| 路径字符串如 `"archives/123/456.png"` | 处理成功 | 显示链接/图片 |

[LinkDetails.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkDetails.tsx#L106-L114)

```typescript
const isReady = () => {
  return (
    link &&
    (collectionOwner.archiveAsScreenshot === true ? link.pdf : true) &&
    (collectionOwner.archiveAsMonolith === true ? link.monolith : true) &&
    (collectionOwner.archiveAsPDF === true ? link.pdf : true) &&
    link.readable
  );
};
```

注意：`isReady` 只判断"非空"，不区分 `"unavailable"` 还是真实路径。真正展示时由 `formatAvailable` 过滤。

### 3.3 页面三种状态渲染

[LinkDetails.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkDetails.tsx#L560-L593)

| 状态 | 条件 | 渲染内容 |
|---|---|---|
| **全部处理中** | `!isReady() && !atLeastOneFormatAvailable(link)` | 大加载动画 + `preservation_in_queue` + `check_back_later` |
| **部分就绪** | `!isReady() && atLeastOneFormatAvailable(link)` | 小加载动画 + `there_are_more_formats` + `check_back_later` |
| **全部完成** | `isReady()` | 展示可用格式（不可用的被 `formatAvailable` 隐藏） |

### 3.4 列表页的自动轮询

[Links.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkViews/Links.tsx#L405-L425)

```typescript
useEffect(() => {
  let interval = null;
  // 只要有任一链接的 preview 既不是已完成路径也不是 unavailable，就每 5s 刷新
  if (links?.some(e => !e.preview?.startsWith("archives") && e.preview !== "unavailable")) {
    interval = setInterval(async () => {
      useData.refetch();
    }, 5000);
  }
  return () => { if (interval) clearInterval(interval); };
}, [links]);
```

这是"队列中"状态自动消失的机制——持续轮询直到所有格式被标记为已完成路径或 `"unavailable"`。

---

## 四、Toast 提示系统

### 4.1 全局配置

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

### 4.2 Toast 使用模式

| 模式 | 示例 | 位置 |
|---|---|---|
| **`toast.error(t(key))`** - 翻译后的错误 | `toast.error(t(error.message))` | Mutation onError |
| **`toast.error(string)`** - 后端英文错误直显 | `toast.error(data.response)` | LinkActions.updateArchive |
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

## 五、本地化文案（i18n）

### 5.1 文案文件位置

- 英文：[en/common.json](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/public/locales/en/common.json)
- 中文：[zh/common.json](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/public/locales/zh/common.json)

### 5.2 文案使用与错误消息的矛盾映射

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

### 5.3 归档/异步相关 i18n key

| Key | 中文 | 英文 |
|---|---|---|
| `preservation_in_queue` | 链接保存正在处理中... | Link preservation is in the queue |
| `check_back_later` | 请稍后再查看结果 | Please check back later to see the result |
| `there_are_more_formats` | 队列中有更多已保存的格式。 | There are more preserved formats in the queue |
| `link_being_archived` | 链接正在归档... | Link is being archived... |
| `links_being_archived` | 链接正在归档…… | Links are being archived... |
| `refresh_preserved_formats` | 刷新保留格式 | Refresh Preserved Formats |
| `no_broken_preservations` | 未发现损坏的存档。 | No broken preservations. |
| `links_are_being_represerved` | 链接正在被重新保存…… | Links are being re-preserved... |
| `preview_unavailable` | 预览不可用 | Preview Unavailable |

---

## 六、重试入口串联

### 6.1 单条链接级重试

两个入口都会弹出 `ConfirmationModal`，确认后走同一条 API：

**入口 1：LinkActions 下拉菜单**（[LinkActions.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkViews/LinkComponents/LinkActions.tsx#L54-L76)）
```
LinkCard 三个点 → Refresh Preserved Formats → ConfirmationModal → updateArchive()
→ PUT /api/v1/links/{id}/archive → toast.success("link_being_archived") → refetch()
```

**入口 2：LinkModal/LinkDetails 详情页刷新按钮**（[LinkModal.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/ModalContent/LinkModal.tsx#L123-L139), [LinkDetails.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/components/LinkDetails.tsx#L480-L501)）
```
详情页 → ⟳ 按钮 tooltip="refresh_preserved_formats" → ConfirmationModal → updateArchive()
```

### 6.2 批量/管理员级重试

[background-jobs.tsx](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/apps/web/pages/admin/background-jobs.tsx#L16-L204)

两个按钮：

| 按钮 | i18n Key | action | 调用 API |
|---|---|---|---|
| 重新生成损坏的链接 | `regenerate_broken_links` | `"allBroken"` | `DELETE /api/v1/worker/preservation` |
| 重新生成全部链接 | `regenerate_all_links` | `"allAndRePreserve"` | `DELETE /api/v1/worker/preservation` |

对应 router hook：[useDeletePreservations](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/packages/router/worker.tsx#L23-L49)

```typescript
const deletePreservations = useDeletePreservations();

await deletePreservations.mutateAsync(
  { action },
  {
    onSettled: (data, error) => {
      toast.dismiss(load);
      if (error) toast.error(error.message);          // 错误：原始消息
      else toast.success(t("links_are_being_represerved")); // 成功：i18n
    },
  }
);
```

成功后自动：
```typescript
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: ["links"] });
  queryClient.invalidateQueries({ queryKey: ["dashboardData"] });
  queryClient.invalidateQueries({ queryKey: ["worker"] });  // 刷新统计数字
}
```

### 6.3 Worker 统计展示

[useWorker](file:///d:/fz/0601/solo-dogfeeding/code/99-linkwarden/packages/router/worker.tsx#L6-L21) 拉取 `/api/v1/worker` 数据，展示：

- `link.pending` - 待处理
- `link.done` - 已完成
- `link.failed` - 失败数（用于 `regenerate_broken_links` 按钮的影响链接数显示）
- `search.pending` / `search.done` - 搜索索引状态

---

## 七、完整调用链示例

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

### 示例 3：单条链接刷新归档（异步重试 → toast + 轮询）

```
LinkActions → updateArchive()
  ├─ toast.loading(t("sending_request"))
  ├─ PUT /api/v1/links/{id}/archive
  │    └─ 后端：清空 image/pdf/readable/monolith/preview/lastPreserved 为 null
  ├─ toast.dismiss(load)
  ├─ toast.success(t("link_being_archived"))
  └─ refetch() 立即刷新
       └─ Links.tsx useEffect 检测到 preview=null，启动 5s 轮询
            └─ Worker 处理中：预览 loading
            └─ Worker 成功/失败：字段变为路径 或 "unavailable"，轮询停止
```

### 示例 4：归档失败页面状态（"unavailable" → 隐藏项）

```
Worker archiveHandler 失败
  └─ finally 块 image/pdf/readable 等被写为 "unavailable"

前端 Links.tsx 轮询拉取
  └─ LinkDetails 渲染
       ├─ formatAvailable(link, "image") → false  → 隐藏截图行
       ├─ formatAvailable(link, "pdf")   → false  → 隐藏 PDF 行
       ├─ isReady() → true（字段非空即可）
       └─ atLeastOneFormatAvailable() → 取决于是否有任一格式是真实路径
```

---

## 八、容易混淆的要点总结

| 混淆点 | 真相 |
|---|---|
| `toast.error(t(error.message))` 的 message 都是 i18n key？ | ❌ 只有前端主动抛出的少量是 key（如 `invalid_url_guide`），后端返回的全是英文硬编码，`t()` 找不到就原样输出 |
| `"unavailable"` 和 `null` 含义相同？ | ❌ `null` = 在队列中未处理；`"unavailable"` = 已处理但失败。前者触发轮询，后者不 |
| `isReady()` 判断归档完成？ | ❌ 只判断字段非空。全部都是 `"unavailable"` 也会被判定为 ready |
| `formatAvailable()` 参与页面三态判断？ | ✅ 三态用 `isReady()` + `atLeastOneFormatAvailable()`，后者又依赖 `formatAvailable()` |
| 乐观更新失败时 toast 和回滚谁先执行？ | 先 toast，后回滚缓存。在 `onError` 中顺序执行 |
| 重试入口有几种？ | 3 种：列表卡片下拉、详情页按钮（单条）；后台 Background Jobs（批量损坏/全部） |
