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
# 在主仓库根目录执行
git init frontend && git -C frontend commit --allow-empty -m "init"
git submodule add ./frontend frontend

git init backend && git -C backend commit --allow-empty -m "init"
git submodule add ./backend backend
```

---

## 二、数据库设计（MySQL）

### feeds 表

```sql
CREATE TABLE feeds (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    url             VARCHAR(2048)   NOT NULL UNIQUE,
    title           VARCHAR(512)    NOT NULL DEFAULT '',
    description     TEXT,
    site_url        VARCHAR(2048),
    fetch_status    ENUM('pending', 'fetching', 'success', 'failed') NOT NULL DEFAULT 'pending',
    fetch_error     VARCHAR(1024),
    last_fetched_at DATETIME,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### articles 表

```sql
CREATE TABLE articles (
    id           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    feed_id      BIGINT UNSIGNED NOT NULL,
    guid         VARCHAR(2048)   NOT NULL,
    title        VARCHAR(1024)   NOT NULL DEFAULT '',
    link         VARCHAR(2048),
    content      MEDIUMTEXT,
    author       VARCHAR(512),
    published_at DATETIME,
    is_read      TINYINT(1) NOT NULL DEFAULT 0,
    is_starred   TINYINT(1) NOT NULL DEFAULT 0,
    created_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    UNIQUE KEY uq_feed_guid (feed_id, guid),
    INDEX idx_feed_id (feed_id),
    INDEX idx_published_at (published_at),
    FOREIGN KEY (feed_id) REFERENCES feeds(id) ON DELETE CASCADE
);
```

**关键决策：**

- `(feed_id, guid)` 联合唯一索引：重复抓取幂等入库（`INSERT IGNORE`）
- `fetch_status` 存在 feeds 表：客户端轮询抓取进度，与文章读取完全解耦
- `is_read` / `is_starred` 存在 articles 表：单用户场景，无需 user_articles 关联表
- `content` 使用 `MEDIUMTEXT`：支持 RSS 全文内容

---

## 三、API 设计

### 订阅源

| Method   | Path                      | 说明                                 |
|----------|---------------------------|--------------------------------------|
| `POST`   | `/api/feeds`              | 添加订阅源，异步触发抓取，返回 202   |
| `GET`    | `/api/feeds`              | 获取所有订阅源列表                   |
| `GET`    | `/api/feeds/:id`          | 获取单个订阅源（含 fetch_status）    |
| `POST`   | `/api/feeds/:id/refresh`  | 手动刷新，异步触发重新抓取，返回 202 |
| `DELETE` | `/api/feeds/:id`          | 删除订阅源（级联删除文章）           |

### 文章

| Method  | Path                | 说明                                           |
|---------|---------------------|------------------------------------------------|
| `GET`   | `/api/articles`     | 文章列表，支持 `?feed_id=&starred=&unread=&page=&page_size=` |
| `GET`   | `/api/articles/:id` | 文章详情（同时标记已读）                       |
| `PATCH` | `/api/articles/:id` | 更新 is_read / is_starred                      |

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
```

**GET /api/feeds/:id**（前端轮询用）
```json
{
  "id": 1,
  "title": "阮一峰的网络日志",
  "url": "https://feeds.feedburner.com/ruanyifeng",
  "fetch_status": "success",
  "fetch_error": null,
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
│   ├── fetcher/                   # RSS 抓取（HTTP + XML 解析，不接触 DB）
│   │   └── fetcher.go
│   ├── model/                     # GORM 模型
│   │   ├── feed.go
│   │   └── article.go
│   └── repository/                # 数据库操作
│       ├── feed.go
│       └── article.go
└── migrations/
    └── 001_init.sql
```

### 依赖关系

```
Handler → Service → Repository → MySQL
              └──→ Fetcher → 外部 RSS 源
```

Fetcher 只负责网络请求和 XML 解析，不接触数据库，可独立测试。

### 核心数据流：添加订阅源

```
Handler.CreateFeed()
  │ 1. 校验 URL 格式
  │ 2. repo.CreateFeed()                  → 写 feeds 表，status=pending
  │ 3. go service.TriggerFetch(feedID)    ← goroutine，立即返回 202
  │
  └─▶ TriggerFetch()（goroutine）
        │ 1. repo.UpdateFetchStatus(feedID, "fetching")
        │ 2. fetcher.Fetch(url)            ← HTTP 拉取，超时见 config.yaml
        │ 3. 解析 RSS items
        │ 4. repo.UpsertArticles()         ← INSERT IGNORE on (feed_id, guid)
        │ 5. repo.UpdateFetchStatus(feedID, "success") + 更新 title/last_fetched_at
        └─▶ 失败时：repo.UpdateFetchStatus(feedID, "failed", errMsg)
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
│   │   ├── AddFeedForm          # 输入 URL + 提交，提交后轮询 fetch_status
│   │   ├── FeedList
│   │   │   └── FeedItem         # 名称、更新时间、刷新按钮、删除按钮
│   │   └── NavLinks             # 全部文章 / 收藏
│   └── MainContent
│       ├── ArticleList          # 支持分页，未读/已读视觉区分
│       │   └── ArticleItem      # 标题、来源、时间、星标按钮
│       └── ArticleDetail        # 展示全文，进入时自动 PATCH is_read=true
```

### 状态管理（TanStack Query）

| 数据 | Query Key | 策略 |
|------|-----------|------|
| 订阅源列表 | `['feeds']` | 添加/删除后 invalidate |
| 单个 feed 状态 | `['feeds', id]` | 添加/刷新后每 2s 轮询，status 变为 success/failed 后停止 |
| 文章列表 | `['articles', filters]` | feed_id / starred / page 变化时重新 fetch |
| 文章详情 | `['articles', id]` | 进入详情页时 fetch，同时触发 PATCH is_read=true |

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
| P2 | 全文抓取 | 扩展 fetcher，抓取原文 HTML |
