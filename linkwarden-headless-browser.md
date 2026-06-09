# Linkwarden 无头浏览器（Playwright）初始化失败与资源泄漏代码链路梳理

本文档从代码实现角度梳理 Linkwarden Worker 中 Playwright 无头浏览器的资源生命周期管理，重点聚焦 **初始化阶段（try/finally 之前）的异常** 导致的资源泄漏、Timeout 注册与清除的错位、以及与 Worker 重试机制的叠加影响。

核心分析文件：[archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/archiveHandler.ts)

---

## 目录

1. [整体架构：Browser / Context / Page 三级资源](#1-整体架构browser--context--page-三级资源)
2. [archiveHandler 初始化序列：资源获取的精确顺序](#2-archivehandler-初始化序列资源获取的精确顺序)
3. [timeoutPromise 注册与 clearTimeout 的位置错位](#3-timeoutpromise-注册与-cleartimeout-的位置错位)
4. [context.newPage 失败语义：Page 是否完成创建](#4-contextnewpage-失败语义page-是否完成创建)
5. [Context 关闭路径：仅在 finally 内触发](#5-context-关闭路径仅在-finally-内触发)
6. [try/finally 外异常的资源泄漏全景](#6-tryfinally-外异常的资源泄漏全景)
7. [Worker 重试机制与泄漏的叠加效应](#7-worker-重试机制与泄漏的叠加效应)
8. [次级泄漏：handleScreenshotAndPdf autoScroll 中的定时器](#8-次级泄漏handlescreenshotandpdf-autoscroll-中的定时器)
9. [修复建议](#9-修复建议)
10. [关键文件索引](#10-关键文件索引)

---

## 1. 整体架构：Browser / Context / Page 三级资源

Worker 使用 Playwright 的**三级资源分层模型**，每一级都需要显式关闭：

```
Browser（浏览器进程）
  ├── 每 30 分钟轮换一次（linkProcessing.ts#L31-L33）
  ├── 所有并行归档任务共享同一个 Browser
  │
  ├── BrowserContext（隔离上下文）← 每个 link 一个
  │     ├── Cookie / Storage / Cache 相互隔离
  │     ├── 注册 SSRF 防护路由（protectPageRequests）
  │     │
  │     └── Page（页面）← 每个 Context 一个
  │           ├── 实际加载 URL、截图、PDF
  │           └── 关闭 Context 时自动关闭 Page
  │
  └── ...（更多并行 Context）
```

**三级资源的生命周期**：

| 资源 | 创建位置 | 关闭位置 | 所有者 |
|---|---|---|---|
| Browser | [linkProcessing.ts#L14](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/workers/linkProcessing.ts#L14) `launchBrowser()` | [linkProcessing.ts#L21](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/workers/linkProcessing.ts#L21) `restartBrowser()` | Worker 全局循环 |
| BrowserContext | [archiveHandler.ts#L78](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/archiveHandler.ts#L78) `browser.newContext()` | [archiveHandler.ts#L229](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/archiveHandler.ts#L229) `context.close()` | 单个 archiveHandler 调用 |
| Page | [archiveHandler.ts#L80](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/archiveHandler.ts#L80) `context.newPage()` | Context 关闭时自动关闭 | 单个 archiveHandler 调用 |

---

## 2. archiveHandler 初始化序列：资源获取的精确顺序

[archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/archiveHandler.ts) 中的初始化序列是理解所有泄漏问题的关键。以下是 **try 块开始之前**（第 109 行之前）的完整代码，每一行都可能抛出异常：

```
行号 | 操作                                    | 资源/副作用
-----|----------------------------------------|-------------------
 63  | const abortController = new AbortController();     | ✅ 创建对象
 64  | let timeoutId: NodeJS.Timeout | undefined;        | ✅ 声明变量
 66  | const timeoutPromise = new Promise((_, reject) => {| ✅ 创建 Promise
 67  |   timeoutId = setTimeout(() => {                   | ⚠️ 注册定时器（OS 级资源）
 68  |     abortController.abort();                        | （5 分钟后触发）
 69  |     reject(new Error(...));
 70  |   }, BROWSER_TIMEOUT * 60000);
 71  | });
     |                                        |
 77  | const contextOptions = getDefaultContextOptions();| ✅ 计算选项
 78  | const context = await browser.newContext(...);     | ⚠️ 创建 BrowserContext（浏览器进程级资源）
 79  | await protectPageRequests(context);               | ✅ 注册 SSRF 路由拦截器
 80  | const page = await context.newPage();              | ⚠️ 创建 Page（浏览器进程级资源）
     |                                        |
 82  | createFolder({ filePath: ... });                   | ✅ 创建本地文件夹（可抛出）
 83  | createFolder({ filePath: ... });                   | ✅ 创建本地文件夹（可抛出）
     |                                        |
 85  | const archivalTags = link.tags.filter(...);        | ✅ 计算
 86  | const archivalSettings: ArchivalSettings =         | ✅ 计算
 ... |   ...                                              |
108  | };                                                |
     |                                        |
109  | try {                                             | 🟢 try 块从此开始
     |                                        |
     |                                        |    只有执行到这里，finally 才会在退出时运行
     |                                        |    上面任一行抛出 → finally 永不执行
```

**关键事实**：第 66-67 行注册的 `timeoutId`（定时器）和第 78 行创建的 `context`、第 80 行创建的 `page` 均位于 **try 块之外**。如果第 67 行之后、第 109 行之前的任何操作抛出异常，这些资源将不会被清理。

---

## 3. timeoutPromise 注册与 clearTimeout 的位置错位

### 3.1 注册位置：try 块之前

[archiveHandler.ts#L66-L75](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/archiveHandler.ts#L66-L75)：

```typescript
const timeoutPromise = new Promise((_, reject) => {
  timeoutId = setTimeout(() => {          // ← 在 Promise executor 中同步注册
    abortController.abort();
    reject(
      new Error(
        `Browser has been open for more than ${BROWSER_TIMEOUT} minutes.`
      )
    );
  }, BROWSER_TIMEOUT * 60000);            // 默认 5 分钟
});
```

**注册时机**：`setTimeout` 在 Promise 构造函数的 executor 中**同步执行**，因此 `timeoutId` 在第 75 行 `timeoutPromise` 创建完成时就已持有有效的 OS 定时器句柄。

### 3.2 清除位置：try 块内的 finally

[archiveHandler.ts#L203-L206](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/archiveHandler.ts#L203-L206)：

```typescript
} finally {
  if (timeoutId !== undefined) {
    clearTimeout(timeoutId);              // ← 仅在进入 try 后才会执行
  }
  ...
}
```

### 3.3 错位导致的泄漏

| 执行路径 | 定时器是否清除 |
|---|---|
| 正常执行完成 → 进入 finally | ✅ 清除 |
| try 块内异常 → 进入 finally | ✅ 清除 |
| **第 66-108 行之间抛出异常（未进入 try）** | ❌ **永不清除** |

**泄漏后果**：
- 定时器在 5 分钟后仍然触发：
  - 调用 `abortController.abort()` —— 此时 `archiveHandler` 早已退出，`abortController` 无人引用，`abort()` 调用是无害的 no-op
  - 调用 `reject(new Error(...))` —— 拒绝一个无任何 `.catch()` / `await` 监听的 **孤儿 Promise**，在 Node.js 中触发 `unhandledRejection` 事件（若 Node.js 配置了 `--unhandled-rejections=strict` 会导致进程崩溃）
- 每个泄漏的定时器占用 Node.js 事件循环一个句柄，延迟进程自然退出

---

## 4. context.newPage 失败语义：Page 是否完成创建

### 4.1 Playwright `newPage()` 的原子性

[archiveHandler.ts#L80](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/archiveHandler.ts#L80)：

```typescript
const page = await context.newPage();
```

Playwright 的 `BrowserContext.newPage()` 遵循 **all-or-nothing** 语义：

| 结果 | Page 对象是否存在 | 是否需要额外关闭 |
|---|---|---|
| ✅ 正常返回 | 赋值给 `page` 变量 | 否，关闭 Context 时自动关闭 |
| ❌ 抛出异常 | `page` 变量仍为 `undefined`（无赋值发生） | 否，内部已回滚所有已分配资源 |

**结论**：`context.newPage()` 失败时**不会泄漏 Page**——但会导致**已创建的 Context 泄漏**（见下一节）。

### 4.2 `browser.newContext()` 失败语义

[archiveHandler.ts#L78](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/archiveHandler.ts#L78)：

```typescript
const context = await browser.newContext(contextOptions);
```

同理，`browser.newContext()` 也是 all-or-nothing：
- 失败时 `context` 仍为 `undefined`，无资源泄漏
- 但此时 `timeoutId` 已注册 → **定时器泄漏**

---

## 5. Context 关闭路径：仅在 finally 内触发

### 5.1 唯一的关闭调用

[archiveHandler.ts#L229](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/archiveHandler.ts#L229)：

```typescript
await context?.close().catch(() => {});
```

- 位于 `finally` 块的**最后一行**
- 使用可选链 `context?.` 防止 `context` 为 `undefined` 时抛错
- `.catch(() => {})` 吞掉所有关闭错误（防止关闭失败影响后续处理）

### 5.2 关闭路径的覆盖范围

| 执行路径 | context.close() 是否执行 |
|---|---|
| 正常执行完成 → 进入 finally | ✅ 执行 |
| try 块内异常 → 进入 finally | ✅ 执行 |
| **第 78-108 行之间抛出异常（context 已创建但未进入 try）** | ❌ **永不执行** |
| 第 66-77 行之间抛出异常（context 未创建） | N/A（context 为 undefined，可选链跳过） |

### 5.3 泄漏的 BrowserContext 有多重？

一个 Playwright BrowserContext 泄漏意味着：
- **浏览器进程级内存**：约 50-200 MB（含 V8 隔离域、Cookie Jar、HTTP Cache、Storage）
- **OS 资源**：若干文件描述符（IPC 管道）、共享内存段
- **内部引用**：Playwright 维护的 Context 链表仍持有该对象，GC 无法回收
- **持续的 CPU 开销**：若页面已开始加载但未停止，可能有后台 JS 定时器、网络请求继续运行

---

## 6. try/finally 外异常的资源泄漏全景

以下表格枚举**初始化阶段（第 63-108 行）**每个可能抛出处的泄漏情况：

| 抛出行 | 失败操作 | timeoutId 已注册？ | context 已创建？ | page 已创建？ | 泄漏资源 |
|---|---|---|---|---|---|
| 66-75 | timeoutPromise 创建（极低概率，仅 OOM 时） | ❌ | ❌ | ❌ | 无 |
| 77 | getDefaultContextOptions()（极低概率） | ✅ | ❌ | ❌ | **定时器** |
| 78 | browser.newContext() | ✅ | ❌ | ❌ | **定时器** |
| 79 | protectPageRequests(context) | ✅ | ✅ | ❌ | **定时器 + Context** |
| 80 | context.newPage() | ✅ | ✅ | ❌ | **定时器 + Context** |
| 82-83 | createFolder()（磁盘满、权限等） | ✅ | ✅ | ✅ | **定时器 + Context**（Page 随 Context 一起泄漏） |
| 85-107 | archivalSettings 计算（极低概率） | ✅ | ✅ | ✅ | **定时器 + Context** |

**高频泄漏组合**：
- 磁盘满 → createFolder 抛错 → 定时器 + Context + Page 全部泄漏
- 浏览器进程崩溃 → newContext / newPage 抛错 → 可能泄漏定时器（取决于崩溃发生在哪一步）
- SSRF 防护初始化异常 → protectPageRequests 抛错 → 定时器 + Context 泄漏

---

## 7. Worker 重试机制与泄漏的叠加效应

### 7.1 三层重试/重启机制

项目存在**三级重启/重试**，但只有最底层的进程死亡才能清理泄漏：

```
第 1 层：linkProcessing 内部 try/catch
  └── [linkProcessing.ts#L45-L68](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/workers/linkProcessing.ts#L45-L68)
      archiveHandler 抛错 → catch → console.error → 检查 browser.isConnected()
        ├── 若仍连接 → 继续处理下一批（泄漏的 Context/定时器累积！）
        └── 若断开 → restartBrowser（关闭旧 Browser，启动新 Browser）
                       此时旧 Browser 关闭 → 其下所有 Context 随之关闭（清理了 Context 泄漏）
                       但 ⚠️ 泄漏的定时器仍在 Node.js 事件循环中！

第 2 层：Browser 30 分钟强制轮换
  └── [linkProcessing.ts#L31-L33](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/workers/linkProcessing.ts#L31-L33)
      每 30 分钟 restartBrowser("30-minute rotation")
        → 关闭旧 Browser → 清理所有累积的 Context 泄漏
        → 但 ⚠️ 泄漏的定时器仍在！

第 3 层：worker 子进程崩溃自动重启
  └── [index.ts#L3-L12](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/index.ts#L3-L12)
      child.on("exit") → setTimeout(launch, 5000)
        → 进程死亡 → OS 回收所有资源（定时器、内存、文件描述符全部清理）
        → ✅ 唯一能彻底清理定时器泄漏的机制
```

### 7.2 泄漏累积的最坏情况

假设平均每 10 个 link 有 1 个在 try 块前初始化失败，每批处理 5 个 link（`ARCHIVE_TAKE_COUNT=5`），Worker 每 10 秒处理一批：

| 时间 | 累积泄漏 | 是否被清理 |
|---|---|---|
| 1 分钟后 | 约 3 个定时器 + 可能 3 个 Context | ❌ 未清理 |
| 10 分钟后 | 约 30 个定时器 + 可能 30 个 Context（数 GB 内存） | ❌ 未清理 |
| 30 分钟轮换 | ✅ Context 全部清理，❌ 定时器仍有 90 个 | Context 被轮换清理，定时器未清理 |
| 进程级崩溃 | ✅ 全部清理 | 仅在崩溃时 |

---

## 8. 次级泄漏：handleScreenshotAndPdf autoScroll 中的定时器

[handleScreenshotAndPdf.ts#L96-L119](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts#L96-L119) 中的 `autoScroll` 函数也存在两处资源管理瑕疵：

```typescript
const autoScroll = async (AUTOSCROLL_TIMEOUT: number) => {
  const timeoutPromise = new Promise<void>((resolve) => {
    setTimeout(() => {               // ⚠️ 注册 setTimeout，从未调用 clearTimeout
      resolve();
    }, AUTOSCROLL_TIMEOUT * 1000);
  });

  const scrollingPromise = new Promise<void>((resolve) => {
    let totalHeight = 0;
    let distance = 100;
    let scrollDown = setInterval(() => {  // ⚠️ 注册 setInterval
      // ... 滚动逻辑 ...
      if (totalHeight >= scrollHeight) {
        clearInterval(scrollDown);         // ✅ 仅在滚动完成时清除
        window.scroll(0, 0);
        resolve();
      }
    }, 100);
  });

  await Promise.race([scrollingPromise, timeoutPromise]);
  // ⚠️ 若 timeoutPromise 先 resolve，scrollDown interval 永远不会被清除！
  // ⚠️ timeoutPromise 的 setTimeout 无论谁赢，都永远不会被 clearTimeout
};
```

虽然 `setInterval` 是在浏览器 Page 的 `page.evaluate()` 上下文中执行（页面销毁后自动失效），但 `timeoutPromise` 的 `setTimeout` 是 Node.js 侧的，每次滚动超时都会泄漏一个定时器句柄。这个泄漏频率低于 `archiveHandler` 的主泄漏，但在大量长页面场景下会累积。

---

## 9. 修复建议

### 9.1 关键修复：把所有资源获取移入 try 块

将 timeoutPromise 注册、context 创建、page 创建全部移入 try 块内，或在 try 块之前增加一个独立的 finally 守卫：

```typescript
// 方案 A：最小改动——在 try 之前的资源也用独立 try/finally 包裹
const abortController = new AbortController();
let timeoutId: NodeJS.Timeout | undefined;
let context: BrowserContext | undefined;
let page: Page | undefined;

try {
  timeoutId = setTimeout(() => { ... }, BROWSER_TIMEOUT * 60000);

  const contextOptions = getDefaultContextOptions();
  context = await browser.newContext(contextOptions);
  await protectPageRequests(context);
  page = await context.newPage();

  createFolder(...);
  createFolder(...);

  const archivalSettings = ...;

  // 原有 try 块内容（Promise.race 等）可以嵌套在此
  try {
    await Promise.race([...]);
  } catch (err) {
    console.log("Failed Link:", link.url);
    throw err;
  } finally {
    // 原有 finally 的业务逻辑（DB 更新等）
  }
} finally {
  // ✅ 统一资源清理：无论在哪一步失败都执行
  if (timeoutId !== undefined) clearTimeout(timeoutId);
  await context?.close().catch(() => {});
}
```

### 9.2 修复 autoScroll 定时器

```typescript
const timeoutPromise = new Promise<void>((resolve) => {
  const id = setTimeout(() => resolve(), AUTOSCROLL_TIMEOUT * 1000);
  timeoutRef.current = id;  // 外部持有引用以便清除
});
// ... race 之后
clearTimeout(timeoutRef.current);
```

### 9.3 Worker 侧防御：定期检查 Context 数量

在 `linkProcessing` 的 `restartBrowser` 之前，增加 Browser Context 数量检查，若异常增长则提前重启：

```typescript
const contexts = browser.contexts();
if (contexts.length > ARCHIVE_TAKE_COUNT * 2) {
  await restartBrowser("excessive context count (possible leak)");
}
```

---

## 10. 关键文件索引

| 文件 | 作用 |
|---|---|
| [apps/worker/lib/archiveHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/archiveHandler.ts) | 核心归档处理器，资源泄漏所在地 |
| [apps/worker/lib/browser.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/browser.ts) | Browser 启动选项、Context 默认选项 |
| [apps/worker/lib/protectPageRequests.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/protectPageRequests.ts) | Context SSRF 路由拦截器 |
| [apps/worker/workers/linkProcessing.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/workers/linkProcessing.ts) | Worker 主循环、Browser 轮换、archiveHandler 调用与重试 |
| [apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/preservationScheme/handleScreenshotAndPdf.ts) | 截图/PDF 处理，含 autoScroll 定时器泄漏 |
| [apps/worker/lib/preservationScheme/handleArchivePreview.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/lib/preservationScheme/handleArchivePreview.ts) | 预览图生成 |
| [apps/worker/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/index.ts) | Worker 子进程启动器，崩溃自动重启 |
| [apps/worker/worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/124-linkwarden/apps/worker/worker.ts) | Worker 多任务入口（linkProcessing/RSS/索引等） |
