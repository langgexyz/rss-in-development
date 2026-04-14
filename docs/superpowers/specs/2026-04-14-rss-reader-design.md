# RSS 订阅阅读器 — 技术设计文档

**日期**：2026-04-14  
**技术栈**：Go + Gin + MySQL（后端）/ React + TypeScript + Vite（前端）  
**开发模式**：工程化，前后端分离，各自独立 git 仓库，通过 submodule 挂载到主仓库

---

## 一、仓库结构

```
04141823/                          # 主仓库
├── CLAUDE.md
├── 笔试题.md
├── docs/superpowers/specs/        # 设计文档
├── frontend/                      # git submodule → 独立仓库（React + TypeScript）
└── backend/                       # git submodule → 独立仓库（Go + Gin）
```

初始化命令：

```bash
# 先在外部创建有初始提交的普通仓库（submodule add 要求目标有至少一个 commit）
git init /tmp/rss-frontend && git -C /tmp/rss-frontend commit --allow-empty -m "init"
git init /tmp/rss-backend  && git -C /tmp/rss-backend  commit --allow-empty -m "init"

# 在主仓库根目录挂载为 submodule
git submodule add /tmp/rss-frontend frontend
git submodule add /tmp/rss-backend  backend
```

---

## 二、数据库设计（MySQL）

### feeds 表

```sql
CREATE TABLE feeds (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    url             VARCHAR(500)    NOT NULL UNIQUE,  -- 500×4=2000 字节，低于 InnoDB 3072 字节索引上限；覆盖绝大多数 RSS 源地址
    title           VARCHAR(512)    NOT NULL DEFAULT '',
    description     TEXT,
    site_url        VARCHAR(2048),
    fetch_status      ENUM('pending', 'fetching', 'success', 'failed') NOT NULL DEFAULT 'pending',
    fetch_error       VARCHAR(1024),
    last_fetched_at   DATETIME,                -- 我们最近一次抓取的时刻
    source_updated_at DATETIME,                -- RSS channel 自身的更新时间（lastBuildDate/updated）
    created_at        DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### articles 表

```sql
CREATE TABLE articles (
    id               BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    feed_id          BIGINT UNSIGNED NOT NULL,
    guid_hash        CHAR(64) NOT NULL,              -- SHA256(normalized_guid)，定长，用于唯一约束
    title            VARCHAR(1024) NOT NULL DEFAULT '',
    link             VARCHAR(2048),
    content          MEDIUMTEXT,
    author           VARCHAR(512),
    published_at     DATETIME,
    is_read          TINYINT(1) NOT NULL DEFAULT 0,
    is_starred       TINYINT(1) NOT NULL DEFAULT 0,
    is_full_content  TINYINT(1) NOT NULL DEFAULT 0,  -- 全文已抓取，刷新时保护 content 不被摘要覆盖
    created_at       DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    UNIQUE KEY uq_feed_guid_hash (feed_id, guid_hash),  -- CHAR(64) 定长，索引无溢出风险
    INDEX idx_feed_id (feed_id),
    INDEX idx_published_at (published_at),
    FOREIGN KEY (feed_id) REFERENCES feeds(id) ON DELETE CASCADE
);
```

**关键决策：**

- `guid_hash CHAR(64)`：存储 `SHA256(normalized_guid)`，定长避免 MySQL 索引溢出（VARCHAR(2048) 的 utf8mb4 索引会超出 InnoDB 3072 字节上限）；不存原始 guid，需要时重抓 feed 即可取到
- `guid_hash` 生成优先级：`SHA256(原始guid/id)` → `SHA256(link)` → `SHA256(title+pubDate_UTC)`
- 入库使用 `ON DUPLICATE KEY UPDATE`（非 `INSERT IGNORE`）：title/link/author 随源更新；`content` 仅在 `is_full_content=0` 时更新；`is_read`/`is_starred` 永不覆盖（用户状态）
- `fetch_status` 存在 feeds 表：展示抓取进度，与文章读取完全解耦；未来可扩展为 SSE 推送通知
- `is_full_content=1` 时刷新订阅源不覆盖 content
- `content` 使用 `MEDIUMTEXT`：支持 RSS 全文内容

---

## 三、API 设计

### 订阅源

| Method   | Path                      | 说明                                              |
|----------|---------------------------|---------------------------------------------------|
| `POST`   | `/api/feeds`              | 添加订阅源，异步触发抓取，返回 202；URL 重复返回 409 |
| `GET`    | `/api/feeds`              | 获取所有订阅源列表                                |
| `GET`    | `/api/feeds/:id`          | 获取单个订阅源（含 fetch_status）                 |
| `POST`   | `/api/feeds/:id/refresh`  | 手动刷新，异步触发重新抓取，返回 202；未到最小间隔返回 429 |
| `DELETE` | `/api/feeds/:id`          | 删除订阅源（级联删除文章）                        |

### 文章

| Method  | Path                | 说明                                                                      |
|---------|---------------------|---------------------------------------------------------------------------|
| `GET`   | `/api/articles`     | 文章列表，默认按 `published_at DESC`，支持 `?feed_id=&starred=&unread=&page=&page_size=`；不含 content 字段 |
| `GET`   | `/api/articles/:id` | 文章详情，含完整 content 字段，同时自动将 is_read 置为 true               |
| `PATCH` | `/api/articles/:id` | 更新 is_read / is_starred，返回更新后的完整文章对象                       |

### Request / Response 示例

**POST /api/feeds**
```json
// Request
{ "url": "https://example.com/rss.xml" }

// Response 202
{
  "id": 1,
  "url": "https://example.com/rss.xml",
  "title": "",
  "fetch_status": "pending",
  "created_at": "2026-04-14T10:00:00Z"
}

// Response 409（URL 已存在）
{ "code": "FEED_ALREADY_EXISTS", "message": "该订阅源已添加" }
```

**GET /api/feeds**
```json
[
  {
    "id": 1,
    "url": "https://feeds.feedburner.com/ruanyifeng",
    "title": "阮一峰的网络日志",
    "fetch_status": "success",
    "source_updated_at": "2026-04-12T09:00:00Z",  // 订阅源自身的更新时间，展示给用户
    "last_fetched_at": "2026-04-14T10:00:30Z"      // 我们的抓取时间，供内部参考
  }
]
```

**GET /api/feeds/:id**（查询单个订阅源状态）
```json
{
  "id": 1,
  "title": "阮一峰的网络日志",
  "url": "https://feeds.feedburner.com/ruanyifeng",
  "fetch_status": "success",
  "fetch_error": null,
  "source_updated_at": "2026-04-12T09:00:00Z",
  "last_fetched_at": "2026-04-14T10:00:30Z"
}
```

**GET /api/articles**
```json
{
  "total": 128,
  "page": 1,
  "page_size": 20,
  "items": [
    {
      "id": 42,
      "feed_id": 1,
      "feed_title": "阮一峰的网络日志",
      "title": "科技爱好者周刊 #293",
      "link": "https://...",
      "author": "阮一峰",
      "published_at": "2026-04-12T09:00:00Z",
      "is_read": false,
      "is_starred": false
    }
  ]
}
```

**PATCH /api/articles/:id**
```json
// Request（只传需要更新的字段）
{ "is_starred": true }

// Response 200
{ "id": 42, "is_read": true, "is_starred": true }
```

**统一错误格式**
```json
{ "code": "FEED_NOT_FOUND", "message": "订阅源不存在" }
```

---

## 四、后端架构（Go + Gin）

### 目录结构

```
backend/
├── main.go
├── config.yaml                    # DB、端口、RSS 超时等配置
├── config/
│   └── config.go
├── internal/
│   ├── handler/                   # Gin handler：参数绑定 + 调用 service
│   │   ├── feed.go
│   │   └── article.go
│   ├── service/                   # 业务逻辑
│   │   ├── feed.go
│   │   └── article.go
│   ├── fetcher/                   # RSS 抓取 + 格式检测 + 规范化映射
│   │   ├── fetcher.go             # 入口：HTTP 拉取，检测格式，调对应 mapper
│   │   ├── normalized.go          # 统一内部模型（NormalizedFeed / NormalizedArticle）
│   │   ├── mapper_rss2.go         # RSS 2.0 → NormalizedFeed
│   │   ├── mapper_atom.go         # Atom 1.0 → NormalizedFeed
│   │   └── mapper_jsonfeed.go     # JSON Feed → NormalizedFeed
│   ├── model/                     # GORM 模型
│   │   ├── feed.go
│   │   └── article.go
│   └── repository/                # 数据库操作
│       ├── feed.go
│       └── article.go
└── migrations/
    └── 001_init.sql
```

### 数据规范化层（Normalized Model）

不同 RSS 格式字段差异显著：

| 字段语义 | RSS 2.0 | Atom 1.0 | JSON Feed |
|----------|---------|----------|-----------|
| 唯一ID | `<guid>` | `<id>` | `id` |
| 内容 | `<description>` | `<content>` / `<summary>` | `content_html` / `content_text` |
| 发布时间 | `<pubDate>` | `<published>` / `<updated>` | `date_published` |
| 作者 | `<author>` | `<author><name>` | `author.name` |
| 链接 | `<link>` | `<link href="">` | `url` |

fetcher 层在 HTTP 拉取后自动检测格式，交由对应 mapper 转换为统一内部结构，repository 和 handler 只与规范化模型交互，与具体格式完全解耦：

```go
// fetcher/normalized.go — 统一内部模型，不对外暴露
type NormalizedFeed struct {
    Title     string
    Description string
    SiteURL   string
    UpdatedAt time.Time          // channel 级别更新时间：RSS <lastBuildDate>，Atom <updated>
    Articles  []NormalizedArticle
}

type NormalizedArticle struct {
    GUIDHash    string    // SHA256(normalized_guid)，存入 articles.guid_hash（CHAR(64)，建唯一索引）
                          // 降级链：SHA256(原始guid/id) → SHA256(link) → SHA256(title+pubDate_UTC)
    Title       string
    Link        string
    Content     string    // 优先取全文，降级取摘要
    Author      string
    PublishedAt time.Time // 统一转换为 UTC；源未提供时使用抓取时间
}
```

mapper 各自独立，新增格式只需新增 mapper 文件，其余层零改动：

```go
// fetcher/fetcher.go
func Fetch(url string) (*NormalizedFeed, error) {
    body := httpGet(url)
    switch detectFormat(body) {
    case FormatRSS2:     return mapperRSS2(body)
    case FormatAtom:     return mapperAtom(body)
    case FormatJSONFeed: return mapperJSONFeed(body)
    default:             return nil, fmt.Errorf("unsupported feed format: %s", url)
    }
}
```

### 依赖关系

```
Handler → Service → Repository → MySQL
              └──→ Fetcher ──→ 格式检测
                         ├──→ RSS2 Mapper   ┐
                         ├──→ Atom Mapper   ├──→ NormalizedFeed → Repository
                         └──→ JSON Mapper   ┘
                         └──→ 外部 RSS 源
```

### 核心数据流：添加订阅源

```
Handler.CreateFeed()
  │ 1. 校验 URL 格式
  │ 2. repo.CreateFeed()                    → 写 feeds 表，status=pending
  │ 3. go service.TriggerFetch(feedID)      ← goroutine，立即返回 202
  │
  └─▶ TriggerFetch()（goroutine）
        │ 1. repo.UpdateFetchStatus(feedID, "fetching")
        │ 2. fetcher.Fetch(url)              ← HTTP 拉取 + 格式检测 + mapper 转换
        │ 3. 得到 NormalizedFeed
        │ 4. 按 published_at DESC 取前 max_articles_per_feed 条（config 配置项）
        │    repo.UpsertArticles()           ← ON DUPLICATE KEY UPDATE on (feed_id, guid_hash)
        │                                       title/link/author 随源更新
        │                                       content 仅在 is_full_content=0 时更新
        │                                       is_read/is_starred 永不覆盖
        │ 5. repo.UpdateFetchStatus(feedID, "success") + 更新 title/last_fetched_at/source_updated_at
        └─▶ 失败时：repo.UpdateFetchStatus(feedID, "failed", errMsg)
```

### 抓取频率控制

防止频繁请求打崩外部 RSS 源：

1. **最小刷新间隔**：同一 feed 两次抓取间隔不得低于 `min_refresh_interval`（配置项），`POST /api/feeds/:id/refresh` 检查 `last_fetched_at`，未到间隔返回 429；`last_fetched_at` 为 NULL（首次添加）时不受限制
2. **HTTP 缓存头**：抓取时携带 `If-Modified-Since` / `If-None-Match`，源返回 304 时跳过解析和入库
3. **全局并发上限**：用带缓冲 channel 做信号量，限制同时进行的抓取 goroutine 数（默认 5），防止批量添加订阅源时并发打崩对方

```go
// 信号量在应用启动时根据 config.fetcher.max_concurrent 初始化
var fetchSemaphore chan struct{}

func InitSemaphore(maxConcurrent int) {
    fetchSemaphore = make(chan struct{}, maxConcurrent)
}

func (s *FeedService) TriggerFetch(feedID uint) {
    go func() {
        fetchSemaphore <- struct{}{}
        defer func() { <-fetchSemaphore }()
        // ... 执行抓取
    }()
}
```

### config.yaml 关键配置项

```yaml
server:
  port: 8080
  cors_origins:
    - "http://localhost:5173"

database:
  dsn: "user:password@tcp(127.0.0.1:3306)/rss_reader?charset=utf8mb4&parseTime=True"

fetcher:
  timeout_seconds: 15          # RSS 抓取 HTTP 超时，可按需调整
  max_articles_per_feed: 100   # 单次抓取最多入库文章数
  max_concurrent: 5            # 最大并发抓取数
  min_refresh_interval: 300    # 同一 feed 最小刷新间隔（秒）
```

---

## 五、前端架构（React + TypeScript + Vite）

### 路由结构

```
/                    → 重定向到 /articles
/articles            → 全部文章列表（默认视图）
/articles?feed_id=1  → 按订阅源筛选
/articles?starred=1  → 收藏列表
/articles/:id        → 文章详情
```

### 组件树

```
App
├── Layout
│   ├── Sidebar
│   │   ├── AddFeedForm          # 输入 URL + 提交，提交后 invalidate 订阅源列表
│   │   ├── FeedList
│   │   │   └── FeedItem         # 名称、source_updated_at（订阅源更新时间）、刷新按钮、删除按钮
│   │   └── NavLinks             # 全部文章 / 收藏
│   └── MainContent
│       ├── ArticleList          # 支持分页，未读/已读视觉区分
│       │   └── ArticleItem      # 标题、来源（feed_title）、发布时间、星标按钮
│       └── ArticleDetail        # 进入时自动标记已读（由后端 GET 触发，前端无需额外 PATCH）
│                                # 展示逻辑：
│                                #   content 有实质内容 → 用 DOMPurify sanitize 后渲染 HTML
│                                #   content 仅为摘要  → 展示摘要 + "阅读原文"按钮（跳转 link）
```

### 状态管理（TanStack Query）

| 数据 | Query Key | 策略 |
|------|-----------|------|
| 订阅源列表 | `['feeds']` | 添加/删除/刷新后 invalidate，重新拉取（含 fetch_status 展示）；未来可改为 SSE 推送 |
| 文章列表 | `['articles', filters]` | feed_id / starred / page 变化时重新 fetch |
| 文章详情 | `['articles', id]` | 进入详情页时 fetch；后端 GET /api/articles/:id 已自动标记已读，无需前端额外发 PATCH |

### 未读 / 已读视觉区分

- **未读**：标题加粗 + 左侧蓝色竖条
- **已读**：标题正常字重 + 灰色文字

---

## 六、功能优先级映射

| 优先级 | 功能 | 涉及 API |
|--------|------|----------|
| P0 | 添加订阅源 | POST /api/feeds |
| P0 | 订阅源列表 | GET /api/feeds |
| P0 | 文章列表展示 | GET /api/articles |
| P0 | 文章详情 | GET /api/articles/:id |
| P0 | 已读/未读标记 | PATCH /api/articles/:id |
| P1 | 刷新订阅源 | POST /api/feeds/:id/refresh |
| P1 | 按订阅源筛选 | GET /api/articles?feed_id= |
| P1 | 收藏功能 | PATCH /api/articles/:id + GET /api/articles?starred=1 |
| P2 | 删除订阅源 | DELETE /api/feeds/:id |
| P2 | 全文抓取 | 见下方说明 |

### P2 全文抓取设计

**触发条件**：文章 `content` 字符数低于阈值（如 500 字），判定为摘要型，触发全文抓取。

**流程**：
1. `GET /api/articles/:id/fulltext` → service 检查 content 长度
2. 若判定为摘要型，向 `article.link` 发起 HTTP GET（带 15s 超时）
3. 用 `golang.org/x/net/html` 解析 HTML，提取正文（优先 `<article>`，降级取 `<main>`，再降级取最长 `<div>`）
4. 更新 `articles.content`，返回完整正文
5. 前端收到后替换展示内容，不再显示"阅读原文"按钮
