# RSS 订阅阅读器

基于 Go + React + MySQL 的 RSS 阅读器，支持添加订阅源、阅读文章、已读/收藏标记、全文抓取。

## 项目结构

```
rss-in-development/
├── backend/          # Go + Gin + GORM 后端（submodule）
├── frontend/         # React + TypeScript + Vite 前端（submodule）
└── docker-compose.e2e.yml  # MySQL 数据库
```

---

## 快速启动

### 前置依赖

- Go 1.22+
- Node.js 18+
- Docker（用于 MySQL）

### 1. 拉取代码

```bash
git clone --recurse-submodules https://github.com/langgexyz/rss-in-development.git
cd rss-in-development
```

### 2. 启动 MySQL

```bash
docker compose -f docker-compose.e2e.yml up -d --wait
```

数据库连接信息（默认）：
- Host: `127.0.0.1:3306`
- Database: `rss_reader`
- User: `root` / Password: `password`

### 3. 启动后端

```bash
cd backend
go run .
# 服务运行在 http://localhost:8080
```

后端配置文件为 `backend/config.yaml`，可修改端口、数据库 DSN、抓取并发数等。

### 4. 启动前端

```bash
cd frontend
npm install
npm run dev
# 服务运行在 http://localhost:5173
```

打开浏览器访问 **http://localhost:5173**。

---

## 功能说明

| 功能 | 说明 |
|------|------|
| 添加订阅源 | 在侧边栏输入框填入 RSS 地址，点击 `+` |
| 订阅源列表 | 侧边栏展示名称、状态点（黄=pending / 蓝=抓取中 / 绿=成功 / 红=失败）、更新时间 |
| 刷新订阅源 | 悬停订阅源条目，点击 `↺` 手动刷新 |
| 删除订阅源 | 悬停订阅源条目，点击 `✕` 删除 |
| 文章列表 | 主区域按发布时间倒序展示所有文章，未读显示蓝色左边框 + 粗体 |
| 按订阅源筛选 | 点击侧边栏任意订阅源，仅显示该源的文章 |
| 收藏列表 | 点击侧边栏「★ 收藏」，显示所有已收藏文章 |
| 文章详情 | 点击文章进入详情页，自动标记已读 |
| 收藏/取消收藏 | 详情页右上角 `☆ / ★` 按钮 |
| 全文抓取 | 详情页「抓取全文」按钮，对摘要型 RSS 抓取原文 |

---

## 测试

### 后端集成测试

使用 testcontainers 自动启动 Docker MySQL，无需手动准备数据库：

```bash
cd backend
go test ./internal/... -timeout 180s
```

包含：Repository、Service、Handler 三层集成测试，共 25 个用例，无 mock。

### 前端单元测试

```bash
cd frontend
npm test
```

### E2E 测试（全栈）

Playwright 驱动真实浏览器，自动启动 Go 后端 + Vite + MySQL + 本地 RSS fixture 服务器：

```bash
cd frontend
npm run test:e2e
```

---

## API 接口

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/api/feeds` | 添加订阅源 |
| `GET` | `/api/feeds` | 订阅源列表 |
| `GET` | `/api/feeds/:id` | 订阅源详情 |
| `POST` | `/api/feeds/:id/refresh` | 手动刷新 |
| `DELETE` | `/api/feeds/:id` | 删除订阅源 |
| `GET` | `/api/articles` | 文章列表（支持 `feed_id`、`starred`、`unread`、`page`、`page_size` 参数） |
| `GET` | `/api/articles/:id` | 文章详情（自动标记已读） |
| `PATCH` | `/api/articles/:id` | 更新文章（仅允许 `is_starred`、`is_read` 字段） |
| `POST` | `/api/articles/:id/fulltext` | 抓取全文 |
