# Linkwarden RSS 订阅源集成深度解析

## 一、整体架构概览

Linkwarden 的 RSS 功能采用 **Worker 轮询 + API 管理** 的双模块架构：

| 模块 | 职责 | 核心文件 |
|------|------|----------|
| Worker 进程 | 后台定时拉取所有订阅源，解析并创建链接 | [rssPolling.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/apps/worker/workers/rssPolling.ts) |
| 核心处理逻辑 | Feed 解析、去重判断、链接创建 | [rssHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/rssHandler.ts) |
| Web API | 用户订阅源的 CRUD 管理 | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/apps/web/pages/api/v1/rss/index.ts)、[[id].ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/apps/web/pages/api/v1/rss/[id].ts) |
| 前端 Hooks | React Query 封装，供 UI 调用 | [rss.tsx](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/router/rss.tsx) |
| 安全防护 | SSRF 防护、安全 DNS 解析 | [ssrf.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/ssrf.ts)、[safeFetch.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/safeFetch.ts) |

---

## 二、订阅源解析

### 2.1 解析流程

解析发生在两处：**用户创建订阅时立即解析** 和 **Worker 定时轮询解析**，两者共用同一套逻辑。

核心解析代码位于 [rssPolling.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/apps/worker/workers/rssPolling.ts#L16-L26)：

```typescript
const parser = new Parser();           // rss-parser 库实例
await assertUrlIsSafeForServerSideFetch(rssSubscription.url);  // SSRF 安全检查
const xml = await safeFetch(rssSubscription.url).then((res) => res.text());  // 安全 HTTP 请求
const feed = await parser.parseString(xml);  // XML → JS 对象
await rssHandler(rssSubscription, feed);     // 交给处理器
```

### 2.2 使用的解析库

使用 `rss-parser`（npm 包），支持 RSS 2.0、Atom 1.0、RSS 1.0 等格式。解析后得到标准结构：

```
feed {
  title, description, link,
  lastBuildDate?,     // 频道级最后构建时间
  items: [
    { title, link, pubDate, guid, description, ... },
    ...
  ]
}
```

### 2.3 安全获取机制（safeFetch）

普通 `fetch` 被替换为 [safeFetch.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/safeFetch.ts)，核心防护措施：

1. **SSRF 域名/IP 黑名单**：[ssrf.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/ssrf.ts#L22-L61) 拦截所有内网地址
   - 主机名：`localhost`、`.local`、`.internal` 等
   - IPv4 段：`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`、`127.0.0.0/8` 等
   - IPv6 段：`::1/128`、`fc00::/7`、`fe80::/10` 等

2. **自定义 DNS Lookup**：在 HTTP Agent 层注入安全 DNS 解析，防止 DNS Rebinding 攻击。

3. **重定向限制**：最多跟随 5 次重定向，每次重定向目标也需通过 SSRF 检查。

4. **代理支持**：若配置 `PROXY` 环境变量，流量走 HTTP/SOCKS 代理。

---

## 三、条目去重机制

去重是 RSS 功能的核心逻辑，完全位于 [rssHandler.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/rssHandler.ts)。

### 3.1 基于时间戳的增量去重

Linkwarden **不使用 GUID/URL 比对**，而是采用 **feed 级时间戳 + item 级时间戳** 的双层过滤策略：

#### 第一层：Feed 级快速判断（[L28-L32](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/rssHandler.ts#L28-L32)）

```typescript
if (
  !rssSubscription.lastBuildDate ||
  (rssSubscription.lastBuildDate &&
    new Date(rssSubscription.lastBuildDate) < new Date(feedLastPubDate))
) {
  // 有新内容，进入下一步
}
```

- 数据库中 `RssSubscription.lastBuildDate` 记录上次处理的"最新时间"
- 若 `feed.lastBuildDate`（频道级）比数据库记录新 → 可能有新条目

#### 第二层：Feed 最新时间的计算（[L12-L26](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/rssHandler.ts#L12-L26)）

```typescript
const feedLastBuildDate = (feed as any).lastBuildDate;
const feedLastPubDate = feedLastBuildDate ??
  feed.items.reduce((acc, item) => {
    const itemPubDate = item.pubDate ? new Date(item.pubDate) : null;
    return itemPubDate && itemPubDate > acc ? itemPubDate : acc;
  }, new Date(0));
```

- 优先使用 RSS 频道的 `lastBuildDate`
- 若 Feed 没有此字段，则遍历所有 item 取最大 `pubDate` 作为频道最新时间

#### 第三层：Item 级精确过滤（[L38-L41](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/rssHandler.ts#L38-L41)）

```typescript
const newItems = feed.items.filter((item) => {
  const itemPubDate = item.pubDate ? new Date(item.pubDate) : null;
  return itemPubDate && itemPubDate > rssSubscription.lastBuildDate!;
});
```

- 只有 `pubDate` **严格大于** 数据库记录时间的条目才会被保留
- **没有 pubDate 的条目直接被丢弃**（`itemPubDate` 为 null → filter 返回 false）

#### 处理完成后更新游标（[L88-L91](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/rssHandler.ts#L88-L91)）

```typescript
await prisma.rssSubscription.update({
  where: { id: rssSubscription.id },
  data: { lastBuildDate: new Date(feedLastPubDate) },
});
```

### 3.2 去重策略的优缺点

| 优点 | 缺点 |
|------|------|
| 实现简单，无需存储每条已处理 item 的 GUID/URL | 依赖 `pubDate` 准确性，若 Feed 不含 pubDate 则条目被丢弃 |
| 数据库压力小，仅存一个时间戳 | 同秒发布的多条新文章可能因 `>` 严格比较导致漏处理 |
| 天然支持大部分 RSS Feed | 若 Feed 修改历史条目的 pubDate 为更新时间，可能造成重复导入 |

---

## 四、链接创建流程

### 4.1 创建前检查：容量限制

在批量创建前，先通过 [hasPassedLimit](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/verifyCapacity.ts) 检查用户是否还有剩余配额：

```typescript
const hasTooManyLinks = await hasPassedLimit(
  rssSubscription.ownerId,
  newItems.length  // 预计新增数量
);
if (hasTooManyLinks) {
  // 超配额，直接跳过整个 Feed 的本轮处理
  return;
}
```

容量检查逻辑（[verifyCapacity.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/verifyCapacity.ts)）：
- **未启用 Stripe**：默认每用户最多 `MAX_LINKS_PER_USER`（30000）条
- **启用 Stripe**：根据订阅等级 `quantity × 30000`，组织内成员共享配额
- **试用期**：无需信用卡，试用期内同样受 30000 限制

### 4.2 并发创建链接

[L56-L85](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/rssHandler.ts#L56-L85) 使用 `Promise.all` 并发创建：

```typescript
await Promise.all(
  newItems.map(async (item) => {
    if (!item.link) return null;              // 无链接的条目跳过

    try {
      await assertUrlIsSafeForServerSideFetch(item.link);  // 每条链接再做 SSRF 检查
    } catch {
      return null;
    }

    return prisma.link.create({
      data: {
        name: item.title,                      // RSS item.title → Link.name
        url: item.link,                        // RSS item.link → Link.url
        type: "url",                           // 固定类型
        createdBy: { connect: { id: rssSubscription.ownerId } },
        collection: { connect: { id: rssSubscription.collectionId } },
      },
    });
  })
);
```

### 4.3 创建订阅时的即时拉取

用户通过 API 创建订阅时（[index.ts L103-L120](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/apps/web/pages/api/v1/rss/index.ts#L103-L120)），不会只存数据库就返回，而是：

1. 写入 `RssSubscription` 记录
2. **立即拉取并解析一次 Feed**
3. **调用 rssHandler 导入历史条目**

这样用户创建订阅后能立刻看到已有文章，不用等下一轮 Worker 轮询。

---

## 五、刷新调度机制

### 5.1 Worker 进程启动

Worker 入口在 [worker.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/apps/worker/worker.ts)：

```typescript
async function init() {
  await migrationWorker();
  startRSSPolling();           // ← RSS 轮询在此启动
  linkProcessing(workerIntervalInSeconds);
  autoTagPreservedLinks(workerIntervalInSeconds);
  startIndexing(workerIntervalInSeconds);
  trialEndEmailWorker();
}
init();
```

`startRSSPolling()` 与其他 Worker 任务**并行运行**，互不阻塞。

### 5.2 轮询循环

[rssPolling.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/apps/worker/workers/rssPolling.ts) 中的核心调度：

```typescript
const pollingIntervalInSeconds =
  (Number(process.env.NEXT_PUBLIC_RSS_POLLING_INTERVAL_MINUTES) || 60) * 60;

export async function startRSSPolling() {
  while (true) {
    const rssSubscriptions = await prisma.rssSubscription.findMany({});

    const parser = new Parser();

    await Promise.all(
      rssSubscriptions.map(async (rssSubscription) => {
        // 单个订阅的拉取+解析+处理（每个订阅独立 try/catch，单个失败不影响全局）
      })
    );
    await delay(pollingIntervalInSeconds);  // 睡眠等待下一轮
  }
}
```

### 5.3 调度特性总结

| 特性 | 说明 |
|------|------|
| **调度方式** | `while(true) + delay()` 的无限循环，非 cron 表达式 |
| **默认间隔** | 60 分钟（可通过 `NEXT_PUBLIC_RSS_POLLING_INTERVAL_MINUTES` 环境变量调整） |
| **并发级别** | 同一轮中所有订阅用 `Promise.all` 并发拉取 |
| **容错策略** | 每个订阅独立 try/catch，单个订阅失败不影响其他订阅和下一轮循环 |
| **间隔计算** | 本轮所有订阅**全部处理完成后**才开始 sleep，不是固定整点触发 |
| **无持久化调度器** | Worker 重启后立即执行第一轮，不记忆上次执行时间 |

### 5.4 架构示意图

```
┌─────────────────────────────────────────────────────┐
│                  Worker 进程                         │
│  ┌───────────────────────────────────────────────┐  │
│  │  startRSSPolling()                             │  │
│  │   │                                            │  │
│  │   ▼                                            │  │
│  │  while(true)                                   │  │
│  │   │                                            │  │
│  │   ├─→ DB: findMany(rssSubscription)            │  │
│  │   │                                            │  │
│  │   ├─→ Promise.all([...])                       │  │
│  │   │    ├─ safeFetch(feed1) → parse → handler   │  │
│  │   │    ├─ safeFetch(feed2) → parse → handler   │  │
│  │   │    └─ ...                                  │  │
│  │   │                                            │  │
│  │   └─→ delay(pollingIntervalInSeconds)          │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## 六、数据模型

Prisma Schema 中定义的 [RssSubscription 模型](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/prisma/schema.prisma#L248-L258)：

```prisma
model RssSubscription {
  id            Int        @id @default(autoincrement())
  url           String                          // RSS Feed 地址
  name          String                          // 用户自定义名称
  lastBuildDate DateTime?                       // 去重游标：上次处理的最新条目时间
  collection    Collection @relation(...)       // 所属收藏夹（链接创建到这里）
  collectionId  Int
  ownerId       Int                             // 所属用户
  createdAt     DateTime   @default(now())
  updatedAt     DateTime   @default(now()) @updatedAt
}
```

### 用户订阅数量限制

API 创建时检查（[index.ts L53-L66](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/apps/web/pages/api/v1/rss/index.ts#L53-L66)）：

```typescript
const RSS_SUBSCRIPTION_LIMIT_PER_USER =
  Number(process.env.RSS_SUBSCRIPTION_LIMIT_PER_USER) || 20;
```

默认每用户最多 20 个 RSS 订阅源。

---

## 七、输入验证

使用 Zod Schema 验证用户提交数据，定义在 [schemaValidation.ts](file:///d:/fz/0601/solo-dogfeeding/code/126-linkwarden/packages/lib/schemaValidation.ts#L249-L254)：

```typescript
export const PostRssSubscriptionSchema = z.object({
  name: z.string().max(50),
  url: z.string().url().max(2048),
  collectionId: z.number().optional(),
  collectionName: z.string().max(50).optional(),
});
```

---

## 八、完整调用链路（新建订阅场景）

```
用户点击"创建RSS订阅"
        │
        ▼
[NewRssSubscriptionModal.tsx]  前端表单 → useAddRssSubscription.mutate()
        │
        ▼
[rss.tsx]  POST /api/v1/rss
        │
        ▼
[api/v1/rss/index.ts]  POST Handler
  ├─ verifyUser()                      鉴权
  ├─ PostRssSubscriptionSchema 校验    Zod 验证
  ├─ 检查订阅数量 ≤ 20                 用户配额
  ├─ assertUrlIsSafeForServerSideFetch SSRF 安全检查
  ├─ setCollection()                   获取/创建收藏夹
  ├─ 检查 name 不重复
  ├─ prisma.rssSubscription.create()   写入数据库
  ├─ safeFetch() + parser.parseString() 立即拉取解析
  └─ rssHandler()                      导入历史条目
        │
        ▼
   返回成功给用户
```

## 九、完整调用链路（Worker 定时刷新场景）

```
Worker 进程启动
        │
        ▼
[worker.ts]  startRSSPolling()
        │
        ▼
[rssPolling.ts]  while(true)
  ├─ prisma.rssSubscription.findMany({})   加载全部订阅
  │
  ├─ 并发遍历每个订阅：
  │    ├─ assertUrlIsSafeForServerSideFetch
  │    ├─ safeFetch(url)                   安全 HTTP 请求
  │    ├─ parser.parseString(xml)          RSS/Atom 解析
  │    └─ rssHandler(subscription, feed)
  │         │
  │         ├─ 计算 feedLastPubDate        频道最新时间
  │         ├─ 比较 lastBuildDate          第一层快速过滤
  │         ├─ 过滤 newItems (pubDate > 游标)  第二层精确去重
  │         ├─ hasPassedLimit()            容量检查
  │         ├─ Promise.all 并发创建 Link   批量入库
  │         └─ 更新 lastBuildDate          移动游标
  │
  └─ delay(NEXT_PUBLIC_RSS_POLLING_INTERVAL_MINUTES × 60)  睡眠等待
```
