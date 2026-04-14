# RSS 订阅阅读器实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 按 P0 → P1 → P2 优先级构建一个完整的 RSS 订阅阅读器，前后端分别在独立 git 仓库中开发，通过 submodule 挂载到主仓库。

**Architecture:** 后端 Go + Gin + MySQL，提供 RESTful API；fetcher 层使用 `gofeed` 统一解析 RSS 2.0 / Atom / JSON Feed，规范化为内部 `NormalizedFeed` 后写入 MySQL，抓取与客户端读取完全解耦；前端 React + TypeScript + TanStack Query，路由驱动视图切换。

**Tech Stack:** Go 1.22+, Gin, GORM, gofeed, MySQL 8.0+, React 18, TypeScript, Vite, TanStack Query, React Router, DOMPurify, Tailwind CSS

**设计文档：** `docs/superpowers/specs/2026-04-14-rss-reader-design.md`

---

## 阶段 0：项目初始化

### Task 1：初始化后端 Go 项目

**Files:**
- Create: `backend/go.mod`
- Create: `backend/config.yaml`
- Create: `backend/config/config.go`
- Create: `backend/migrations/001_init.sql`

- [ ] **Step 1：初始化 Go 模块**

```bash
cd backend
go mod init rss-backend
go get github.com/gin-gonic/gin@latest
go get gorm.io/gorm@latest
go get gorm.io/driver/mysql@latest
go get github.com/mmcdole/gofeed@latest
go get github.com/spf13/viper@latest
go get github.com/stretchr/testify@latest
```

- [ ] **Step 2：创建 `config.yaml`**

```yaml
# backend/config.yaml
server:
  port: 8080
  cors_origins:
    - "http://localhost:5173"

database:
  dsn: "root:password@tcp(127.0.0.1:3306)/rss_reader?charset=utf8mb4&parseTime=True&loc=UTC"

fetcher:
  timeout_seconds: 15
  max_articles_per_feed: 100
  max_concurrent: 5
  min_refresh_interval: 300
```

- [ ] **Step 3：创建 `config/config.go`**

```go
// backend/config/config.go
package config

import "github.com/spf13/viper"

type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    Fetcher  FetcherConfig
}

type ServerConfig struct {
    Port        int
    CORSOrigins []string `mapstructure:"cors_origins"`
}

type DatabaseConfig struct {
    DSN string
}

type FetcherConfig struct {
    TimeoutSeconds     int `mapstructure:"timeout_seconds"`
    MaxArticlesPerFeed int `mapstructure:"max_articles_per_feed"`
    MaxConcurrent      int `mapstructure:"max_concurrent"`
    MinRefreshInterval int `mapstructure:"min_refresh_interval"`
}

func Load() (*Config, error) {
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")
    if err := viper.ReadInConfig(); err != nil {
        return nil, err
    }
    var cfg Config
    return &cfg, viper.Unmarshal(&cfg)
}
```

- [ ] **Step 4：创建 `migrations/001_init.sql`**

```sql
-- backend/migrations/001_init.sql
CREATE TABLE IF NOT EXISTS feeds (
    id               BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    url              VARCHAR(500)  NOT NULL UNIQUE,
    title            VARCHAR(512)  NOT NULL DEFAULT '',
    description      TEXT,
    site_url         VARCHAR(2048),
    fetch_status     ENUM('pending','fetching','success','failed') NOT NULL DEFAULT 'pending',
    fetch_error      VARCHAR(1024),
    last_fetched_at  DATETIME,
    source_updated_at DATETIME,
    created_at       DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at       DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS articles (
    id               BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    feed_id          BIGINT UNSIGNED NOT NULL,
    guid_hash        CHAR(64)      NOT NULL,
    title            VARCHAR(1024) NOT NULL DEFAULT '',
    link             VARCHAR(2048),
    content          MEDIUMTEXT,
    author           VARCHAR(512),
    published_at     DATETIME,
    is_read          TINYINT(1)    NOT NULL DEFAULT 0,
    is_starred       TINYINT(1)    NOT NULL DEFAULT 0,
    is_full_content  TINYINT(1)    NOT NULL DEFAULT 0,
    created_at       DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uq_feed_guid_hash (feed_id, guid_hash),
    INDEX idx_feed_id (feed_id),
    INDEX idx_published_at (published_at),
    FOREIGN KEY (feed_id) REFERENCES feeds(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

- [ ] **Step 5：在 MySQL 中创建数据库并执行迁移**

```bash
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS rss_reader CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root -p rss_reader < migrations/001_init.sql
```

- [ ] **Step 6：创建目录骨架**

```bash
mkdir -p internal/{handler,service,fetcher,model,repository}
```

- [ ] **Step 7：提交**

```bash
git add .
git commit -m "feat: init backend project structure and migrations"
```

---

### Task 2：初始化前端项目

**Files:**
- Create: `frontend/` (Vite 项目)

- [ ] **Step 1：创建 Vite + React + TypeScript 项目**

```bash
cd frontend
npm create vite@latest . -- --template react-ts
npm install
npm install @tanstack/react-query react-router-dom axios dompurify
npm install -D @types/dompurify tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

- [ ] **Step 2：配置 Tailwind**

```js
// frontend/tailwind.config.js
export default {
  content: ["./index.html", "./src/**/*.{ts,tsx}"],
  theme: { extend: {} },
  plugins: [],
}
```

```css
/* frontend/src/index.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

- [ ] **Step 3：配置 Vite 代理（开发时转发 API 请求到后端）**

```ts
// frontend/vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': 'http://localhost:8080',
    },
  },
})
```

- [ ] **Step 4：提交**

```bash
git add .
git commit -m "feat: init frontend project with Vite + React + TS"
```

---

## 阶段 1：P0 核心功能

### Task 3：后端 GORM 模型

**Files:**
- Create: `backend/internal/model/feed.go`
- Create: `backend/internal/model/article.go`

- [ ] **Step 1：创建 Feed 模型**

```go
// backend/internal/model/feed.go
package model

import "time"

type Feed struct {
    ID               uint       `gorm:"primaryKey;autoIncrement"`
    URL              string     `gorm:"type:varchar(500);not null;uniqueIndex"`
    Title            string     `gorm:"type:varchar(512);not null;default:''"`
    Description      string     `gorm:"type:text"`
    SiteURL          string     `gorm:"type:varchar(2048)"`
    FetchStatus      string     `gorm:"type:enum('pending','fetching','success','failed');not null;default:'pending'"`
    FetchError       string     `gorm:"type:varchar(1024)"`
    LastFetchedAt    *time.Time
    SourceUpdatedAt  *time.Time
    CreatedAt        time.Time
    UpdatedAt        time.Time
}
```

- [ ] **Step 2：创建 Article 模型**

```go
// backend/internal/model/article.go
package model

import "time"

type Article struct {
    ID             uint       `gorm:"primaryKey;autoIncrement"`
    FeedID         uint       `gorm:"not null;index"`
    GUIDHash       string     `gorm:"type:char(64);not null;uniqueIndex:uq_feed_guid_hash"`
    Title          string     `gorm:"type:varchar(1024);not null;default:''"`
    Link           string     `gorm:"type:varchar(2048)"`
    Content        string     `gorm:"type:mediumtext"`
    Author         string     `gorm:"type:varchar(512)"`
    PublishedAt    *time.Time
    IsRead         bool       `gorm:"not null;default:false"`
    IsStarred      bool       `gorm:"not null;default:false"`
    IsFullContent  bool       `gorm:"not null;default:false"`
    CreatedAt      time.Time
}
```

- [ ] **Step 3：提交**

```bash
git add internal/model/
git commit -m "feat: add GORM models for Feed and Article"
```

---

### Task 4：Fetcher 层

**Files:**
- Create: `backend/internal/fetcher/normalized.go`
- Create: `backend/internal/fetcher/fetcher.go`
- Create: `backend/internal/fetcher/fetcher_test.go`

- [ ] **Step 1：创建 NormalizedFeed 内部模型**

```go
// backend/internal/fetcher/normalized.go
package fetcher

import "time"

type NormalizedFeed struct {
    Title       string
    Description string
    SiteURL     string
    UpdatedAt   time.Time
    Articles    []NormalizedArticle
}

// GUIDHash 降级链：SHA256(原始guid) → SHA256(link) → SHA256(title+pubDate_UTC)
type NormalizedArticle struct {
    GUIDHash    string
    Title       string
    Link        string
    Content     string
    Author      string
    PublishedAt time.Time
}
```

- [ ] **Step 2：写 fetcher 测试（先写，后实现）**

```go
// backend/internal/fetcher/fetcher_test.go
package fetcher_test

import (
    "context"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "rss-backend/internal/fetcher"
)

const sampleRSS2 = `<?xml version="1.0"?>
<rss version="2.0">
  <channel>
    <title>Test Feed</title>
    <link>https://example.com</link>
    <lastBuildDate>Mon, 14 Apr 2026 10:00:00 +0000</lastBuildDate>
    <item>
      <title>Article One</title>
      <link>https://example.com/1</link>
      <guid>guid-001</guid>
      <description>Summary of article one</description>
      <author>Alice</author>
      <pubDate>Mon, 14 Apr 2026 09:00:00 +0000</pubDate>
    </item>
    <item>
      <title>Article Two</title>
      <link>https://example.com/2</link>
      <description>Summary of article two</description>
    </item>
  </channel>
</rss>`

func TestFetch_RSS2(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/rss+xml")
        w.Write([]byte(sampleRSS2))
    }))
    defer srv.Close()

    f := fetcher.New(10, 100)
    feed, err := f.Fetch(context.Background(), srv.URL)

    require.NoError(t, err)
    assert.Equal(t, "Test Feed", feed.Title)
    assert.Equal(t, "https://example.com", feed.SiteURL)
    assert.False(t, feed.UpdatedAt.IsZero())
    assert.Len(t, feed.Articles, 2)

    a := feed.Articles[0]
    assert.Equal(t, "Article One", a.Title)
    assert.Equal(t, "https://example.com/1", a.Link)
    assert.Equal(t, "Alice", a.Author)
    assert.NotEmpty(t, a.GUIDHash)
    assert.Len(t, a.GUIDHash, 64) // SHA256 hex = 64 chars
}

func TestFetch_GUIDHash_Fallback(t *testing.T) {
    // Article Two 无 guid，应 fallback 到 SHA256(link)
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(sampleRSS2))
    }))
    defer srv.Close()

    f := fetcher.New(10, 100)
    feed, err := f.Fetch(context.Background(), srv.URL)
    require.NoError(t, err)

    a1 := feed.Articles[0]
    a2 := feed.Articles[1]
    assert.NotEqual(t, a1.GUIDHash, a2.GUIDHash, "不同文章的 GUIDHash 不应相同")
}

func TestFetch_MaxArticles(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(sampleRSS2))
    }))
    defer srv.Close()

    f := fetcher.New(10, 1) // maxArticles = 1
    feed, err := f.Fetch(context.Background(), srv.URL)
    require.NoError(t, err)
    assert.Len(t, feed.Articles, 1)
}

func TestFetch_InvalidURL(t *testing.T) {
    f := fetcher.New(1, 100)
    _, err := f.Fetch(context.Background(), "http://127.0.0.1:1")
    assert.Error(t, err)
}
```

- [ ] **Step 3：运行测试，确认失败**

```bash
cd backend
go test ./internal/fetcher/ -v
```

预期：`FAIL — fetcher.New undefined`

- [ ] **Step 4：实现 `fetcher.go`**

```go
// backend/internal/fetcher/fetcher.go
package fetcher

import (
    "context"
    "crypto/sha256"
    "fmt"
    "net/http"
    "time"

    "github.com/mmcdole/gofeed"
)

type Fetcher struct {
    client  *http.Client
    parser  *gofeed.Parser
    maxArts int
}

func New(timeoutSecs, maxArticles int) *Fetcher {
    return &Fetcher{
        client:  &http.Client{Timeout: time.Duration(timeoutSecs) * time.Second},
        parser:  gofeed.NewParser(),
        maxArts: maxArticles,
    }
}

func (f *Fetcher) Fetch(ctx context.Context, url string) (*NormalizedFeed, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return nil, fmt.Errorf("create request: %w", err)
    }
    resp, err := f.client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("http get: %w", err)
    }
    defer resp.Body.Close()

    feed, err := f.parser.Parse(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("parse feed: %w", err)
    }
    return f.normalize(feed), nil
}

func (f *Fetcher) normalize(feed *gofeed.Feed) *NormalizedFeed {
    nf := &NormalizedFeed{
        Title:       feed.Title,
        Description: feed.Description,
        SiteURL:     feed.Link,
    }
    if feed.UpdatedParsed != nil {
        nf.UpdatedAt = feed.UpdatedParsed.UTC()
    } else if feed.PublishedParsed != nil {
        nf.UpdatedAt = feed.PublishedParsed.UTC()
    }

    items := feed.Items
    if len(items) > f.maxArts {
        items = items[:f.maxArts]
    }

    for _, item := range items {
        na := NormalizedArticle{
            GUIDHash: computeGUIDHash(item),
            Title:    item.Title,
            Link:     item.Link,
            Author:   authorName(item),
            Content:  itemContent(item),
        }
        switch {
        case item.PublishedParsed != nil:
            na.PublishedAt = item.PublishedParsed.UTC()
        case item.UpdatedParsed != nil:
            na.PublishedAt = item.UpdatedParsed.UTC()
        default:
            na.PublishedAt = time.Now().UTC()
        }
        nf.Articles = append(nf.Articles, na)
    }
    return nf
}

func computeGUIDHash(item *gofeed.Item) string {
    if item.GUID != "" {
        return sha256Hex(item.GUID)
    }
    if item.Link != "" {
        return sha256Hex(item.Link)
    }
    return sha256Hex(item.Title + item.Published)
}

func sha256Hex(s string) string {
    h := sha256.Sum256([]byte(s))
    return fmt.Sprintf("%x", h)
}

func authorName(item *gofeed.Item) string {
    if item.Author != nil {
        return item.Author.Name
    }
    return ""
}

func itemContent(item *gofeed.Item) string {
    if item.Content != "" {
        return item.Content
    }
    return item.Description
}
```

- [ ] **Step 5：运行测试，确认通过**

```bash
go test ./internal/fetcher/ -v
```

预期：所有测试 PASS

- [ ] **Step 6：提交**

```bash
git add internal/fetcher/
git commit -m "feat: add fetcher with gofeed normalization and GUID hash fallback"
```

---

### Task 5：Feed Repository

**Files:**
- Create: `backend/internal/repository/feed.go`
- Create: `backend/internal/repository/feed_test.go`

- [ ] **Step 1：写 Feed Repository 测试**

```go
// backend/internal/repository/feed_test.go
package repository_test

import (
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    "rss-backend/internal/model"
    "rss-backend/internal/repository"
)

func setupTestDB(t *testing.T) *gorm.DB {
    // 使用测试数据库（需提前创建 rss_reader_test）
    dsn := "root:password@tcp(127.0.0.1:3306)/rss_reader_test?charset=utf8mb4&parseTime=True&loc=UTC"
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    require.NoError(t, err)
    db.AutoMigrate(&model.Feed{}, &model.Article{})
    t.Cleanup(func() {
        db.Exec("DELETE FROM articles")
        db.Exec("DELETE FROM feeds")
    })
    return db
}

func TestFeedRepository_Create(t *testing.T) {
    db := setupTestDB(t)
    repo := repository.NewFeedRepository(db)

    feed, err := repo.Create("https://example.com/rss.xml")
    require.NoError(t, err)
    assert.NotZero(t, feed.ID)
    assert.Equal(t, "pending", feed.FetchStatus)
}

func TestFeedRepository_Create_DuplicateURL(t *testing.T) {
    db := setupTestDB(t)
    repo := repository.NewFeedRepository(db)

    _, err := repo.Create("https://example.com/rss.xml")
    require.NoError(t, err)

    _, err = repo.Create("https://example.com/rss.xml")
    assert.Error(t, err, "重复 URL 应返回错误")
}

func TestFeedRepository_List(t *testing.T) {
    db := setupTestDB(t)
    repo := repository.NewFeedRepository(db)

    repo.Create("https://a.com/rss")
    repo.Create("https://b.com/rss")

    feeds, err := repo.List()
    require.NoError(t, err)
    assert.Len(t, feeds, 2)
}

func TestFeedRepository_UpdateStatus(t *testing.T) {
    db := setupTestDB(t)
    repo := repository.NewFeedRepository(db)

    feed, _ := repo.Create("https://example.com/rss.xml")
    err := repo.UpdateStatus(feed.ID, "success", "", nil, nil, "My Blog")
    require.NoError(t, err)

    updated, _ := repo.GetByID(feed.ID)
    assert.Equal(t, "success", updated.FetchStatus)
    assert.Equal(t, "My Blog", updated.Title)
}
```

- [ ] **Step 2：运行测试，确认失败**

```bash
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS rss_reader_test CHARACTER SET utf8mb4;"
go test ./internal/repository/ -run TestFeedRepository -v
```

预期：`FAIL — repository.NewFeedRepository undefined`

- [ ] **Step 3：实现 `repository/feed.go`**

```go
// backend/internal/repository/feed.go
package repository

import (
    "time"

    "gorm.io/gorm"
    "rss-backend/internal/model"
)

type FeedRepository struct {
    db *gorm.DB
}

func NewFeedRepository(db *gorm.DB) *FeedRepository {
    return &FeedRepository{db: db}
}

func (r *FeedRepository) Create(url string) (*model.Feed, error) {
    feed := &model.Feed{URL: url}
    return feed, r.db.Create(feed).Error
}

func (r *FeedRepository) List() ([]model.Feed, error) {
    var feeds []model.Feed
    return feeds, r.db.Order("created_at DESC").Find(&feeds).Error
}

func (r *FeedRepository) GetByID(id uint) (*model.Feed, error) {
    var feed model.Feed
    return &feed, r.db.First(&feed, id).Error
}

// UpdateStatus 更新抓取状态及相关字段，零值字段不覆盖
func (r *FeedRepository) UpdateStatus(id uint, status, errMsg string, lastFetched, sourceUpdated *time.Time, title string) error {
    updates := map[string]interface{}{
        "fetch_status": status,
        "fetch_error":  errMsg,
    }
    if lastFetched != nil {
        updates["last_fetched_at"] = lastFetched
    }
    if sourceUpdated != nil && !sourceUpdated.IsZero() {
        updates["source_updated_at"] = sourceUpdated
    }
    if title != "" {
        updates["title"] = title
    }
    return r.db.Model(&model.Feed{}).Where("id = ?", id).Updates(updates).Error
}

func (r *FeedRepository) Delete(id uint) error {
    return r.db.Delete(&model.Feed{}, id).Error
}
```

- [ ] **Step 4：运行测试，确认通过**

```bash
go test ./internal/repository/ -run TestFeedRepository -v
```

预期：所有测试 PASS

- [ ] **Step 5：提交**

```bash
git add internal/repository/feed.go internal/repository/feed_test.go
git commit -m "feat: add FeedRepository with CRUD"
```

---

### Task 6：Article Repository

**Files:**
- Create: `backend/internal/repository/article.go`
- Create: `backend/internal/repository/article_test.go`

- [ ] **Step 1：写 Article Repository 测试**

```go
// backend/internal/repository/article_test.go
package repository_test

import (
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "rss-backend/internal/fetcher"
    "rss-backend/internal/repository"
)

func TestArticleRepository_Upsert_Idempotent(t *testing.T) {
    db := setupTestDB(t)
    feedRepo := repository.NewFeedRepository(db)
    artRepo := repository.NewArticleRepository(db)

    feed, _ := feedRepo.Create("https://example.com/rss")

    arts := []fetcher.NormalizedArticle{
        {GUIDHash: "abc123aabbcc", Title: "Hello", Link: "https://x.com/1", Content: "body", PublishedAt: time.Now()},
    }

    err := artRepo.Upsert(feed.ID, arts)
    require.NoError(t, err)

    // 重复插入应幂等，不报错
    err = artRepo.Upsert(feed.ID, arts)
    require.NoError(t, err)

    result, err := artRepo.List(repository.ArticleFilter{Page: 1, PageSize: 10})
    require.NoError(t, err)
    assert.Equal(t, int64(1), result.Total, "幂等入库后应只有一条记录")
}

func TestArticleRepository_Upsert_DoesNotOverwriteUserState(t *testing.T) {
    db := setupTestDB(t)
    feedRepo := repository.NewFeedRepository(db)
    artRepo := repository.NewArticleRepository(db)

    feed, _ := feedRepo.Create("https://example.com/rss")
    arts := []fetcher.NormalizedArticle{
        {GUIDHash: "hash001", Title: "Title", Link: "https://x.com/1", Content: "body", PublishedAt: time.Now()},
    }
    artRepo.Upsert(feed.ID, arts)

    // 标记已读
    result, _ := artRepo.List(repository.ArticleFilter{Page: 1, PageSize: 10})
    artID := result.Items[0].ID
    artRepo.Update(artID, map[string]interface{}{"is_read": true})

    // 再次 upsert 相同 guid
    artRepo.Upsert(feed.ID, arts)

    updated, _ := artRepo.GetByID(artID)
    assert.True(t, updated.IsRead, "is_read 不应被 upsert 覆盖")
}

func TestArticleRepository_Upsert_ProtectsFullContent(t *testing.T) {
    db := setupTestDB(t)
    feedRepo := repository.NewFeedRepository(db)
    artRepo := repository.NewArticleRepository(db)

    feed, _ := feedRepo.Create("https://example.com/rss")
    arts := []fetcher.NormalizedArticle{
        {GUIDHash: "hash002", Title: "Title", Content: "full content", PublishedAt: time.Now()},
    }
    artRepo.Upsert(feed.ID, arts)

    result, _ := artRepo.List(repository.ArticleFilter{Page: 1, PageSize: 10})
    artID := result.Items[0].ID
    // 标记全文已抓取
    artRepo.Update(artID, map[string]interface{}{"is_full_content": true})

    // 再次 upsert，content 变为摘要
    arts[0].Content = "short summary"
    artRepo.Upsert(feed.ID, arts)

    updated, _ := artRepo.GetByID(artID)
    assert.Equal(t, "full content", updated.Content, "is_full_content=1 时 content 不应被覆盖")
}

func TestArticleRepository_List_FilterByFeed(t *testing.T) {
    db := setupTestDB(t)
    feedRepo := repository.NewFeedRepository(db)
    artRepo := repository.NewArticleRepository(db)

    feed1, _ := feedRepo.Create("https://a.com/rss")
    feed2, _ := feedRepo.Create("https://b.com/rss")

    artRepo.Upsert(feed1.ID, []fetcher.NormalizedArticle{
        {GUIDHash: "h1", Title: "A1", PublishedAt: time.Now()},
    })
    artRepo.Upsert(feed2.ID, []fetcher.NormalizedArticle{
        {GUIDHash: "h2", Title: "B1", PublishedAt: time.Now()},
    })

    result, err := artRepo.List(repository.ArticleFilter{FeedID: &feed1.ID, Page: 1, PageSize: 10})
    require.NoError(t, err)
    assert.Equal(t, int64(1), result.Total)
    assert.Equal(t, "A1", result.Items[0].Title)
}
```

- [ ] **Step 2：运行测试，确认失败**

```bash
go test ./internal/repository/ -run TestArticleRepository -v
```

- [ ] **Step 3：实现 `repository/article.go`**

```go
// backend/internal/repository/article.go
package repository

import (
    "gorm.io/gorm"
    "rss-backend/internal/fetcher"
    "rss-backend/internal/model"
)

type ArticleRepository struct {
    db *gorm.DB
}

func NewArticleRepository(db *gorm.DB) *ArticleRepository {
    return &ArticleRepository{db: db}
}

// Upsert 幂等入库。ON DUPLICATE KEY UPDATE：
//   - title/link/author 随源更新
//   - content 仅在 is_full_content=0 时更新
//   - is_read/is_starred 永不覆盖
func (r *ArticleRepository) Upsert(feedID uint, articles []fetcher.NormalizedArticle) error {
    for _, a := range articles {
        err := r.db.Exec(`
            INSERT INTO articles (feed_id, guid_hash, title, link, content, author, published_at)
            VALUES (?, ?, ?, ?, ?, ?, ?)
            ON DUPLICATE KEY UPDATE
                title   = VALUES(title),
                link    = VALUES(link),
                author  = VALUES(author),
                content = IF(is_full_content = 1, content, VALUES(content))
        `, feedID, a.GUIDHash, a.Title, a.Link, a.Content, a.Author, a.PublishedAt).Error
        if err != nil {
            return err
        }
    }
    return nil
}

type ArticleFilter struct {
    FeedID   *uint
    Starred  *bool
    Unread   *bool
    Page     int
    PageSize int
}

type ArticleWithFeedTitle struct {
    model.Article
    FeedTitle string
}

type ArticleListResult struct {
    Total int64
    Items []ArticleWithFeedTitle
}

func (r *ArticleRepository) List(f ArticleFilter) (*ArticleListResult, error) {
    query := r.db.Model(&model.Article{}).
        Select("articles.*, feeds.title AS feed_title").
        Joins("LEFT JOIN feeds ON feeds.id = articles.feed_id").
        Order("articles.published_at DESC")

    if f.FeedID != nil {
        query = query.Where("articles.feed_id = ?", *f.FeedID)
    }
    if f.Starred != nil {
        query = query.Where("articles.is_starred = ?", *f.Starred)
    }
    if f.Unread != nil && *f.Unread {
        query = query.Where("articles.is_read = 0")
    }

    var total int64
    if err := query.Count(&total).Error; err != nil {
        return nil, err
    }

    var items []ArticleWithFeedTitle
    offset := (f.Page - 1) * f.PageSize
    err := query.Limit(f.PageSize).Offset(offset).Scan(&items).Error
    return &ArticleListResult{Total: total, Items: items}, err
}

func (r *ArticleRepository) GetByID(id uint) (*ArticleWithFeedTitle, error) {
    var art ArticleWithFeedTitle
    err := r.db.Model(&model.Article{}).
        Select("articles.*, feeds.title AS feed_title").
        Joins("LEFT JOIN feeds ON feeds.id = articles.feed_id").
        Where("articles.id = ?", id).
        Scan(&art).Error
    if art.ID == 0 {
        return nil, gorm.ErrRecordNotFound
    }
    return &art, err
}

func (r *ArticleRepository) Update(id uint, updates map[string]interface{}) (*model.Article, error) {
    if err := r.db.Model(&model.Article{}).Where("id = ?", id).Updates(updates).Error; err != nil {
        return nil, err
    }
    var art model.Article
    return &art, r.db.First(&art, id).Error
}
```

- [ ] **Step 4：运行测试，确认通过**

```bash
go test ./internal/repository/ -v
```

预期：所有测试 PASS

- [ ] **Step 5：提交**

```bash
git add internal/repository/article.go internal/repository/article_test.go
git commit -m "feat: add ArticleRepository with idempotent upsert"
```

---

### Task 7：Feed Service (P0)

**Files:**
- Create: `backend/internal/service/feed.go`
- Create: `backend/internal/service/feed_test.go`
- Create: `backend/internal/service/semaphore.go`

- [ ] **Step 1：创建信号量**

```go
// backend/internal/service/semaphore.go
package service

var fetchSemaphore chan struct{}

func InitSemaphore(maxConcurrent int) {
    fetchSemaphore = make(chan struct{}, maxConcurrent)
}
```

- [ ] **Step 2：实现 Feed Service**

```go
// backend/internal/service/feed.go
package service

import (
    "context"
    "fmt"
    "time"

    "rss-backend/internal/fetcher"
    "rss-backend/internal/model"
    "rss-backend/internal/repository"
)

type FeedService struct {
    feedRepo *repository.FeedRepository
    artRepo  *repository.ArticleRepository
    fetcher  *fetcher.Fetcher
}

func NewFeedService(fr *repository.FeedRepository, ar *repository.ArticleRepository, f *fetcher.Fetcher) *FeedService {
    return &FeedService{feedRepo: fr, artRepo: ar, fetcher: f}
}

type CreateFeedResult struct {
    Feed *model.Feed
}

func (s *FeedService) CreateFeed(url string) (*model.Feed, error) {
    feed, err := s.feedRepo.Create(url)
    if err != nil {
        return nil, err
    }
    go s.triggerFetch(feed.ID, url)
    return feed, nil
}

func (s *FeedService) ListFeeds() ([]model.Feed, error) {
    return s.feedRepo.List()
}

func (s *FeedService) GetFeed(id uint) (*model.Feed, error) {
    return s.feedRepo.GetByID(id)
}

func (s *FeedService) triggerFetch(feedID uint, url string) {
    fetchSemaphore <- struct{}{}
    defer func() { <-fetchSemaphore }()

    s.feedRepo.UpdateStatus(feedID, "fetching", "", nil, nil, "")

    ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
    defer cancel()

    nf, err := s.fetcher.Fetch(ctx, url)
    if err != nil {
        s.feedRepo.UpdateStatus(feedID, "failed", fmt.Sprintf("%.512s", err.Error()), nil, nil, "")
        return
    }

    if err := s.artRepo.Upsert(feedID, nf.Articles); err != nil {
        s.feedRepo.UpdateStatus(feedID, "failed", "upsert articles failed", nil, nil, "")
        return
    }

    now := time.Now().UTC()
    s.feedRepo.UpdateStatus(feedID, "success", "", &now, &nf.UpdatedAt, nf.Title)
}
```

- [ ] **Step 3：提交**

```bash
git add internal/service/
git commit -m "feat: add FeedService with async fetch goroutine"
```

---

### Task 8：Article Service (P0)

**Files:**
- Create: `backend/internal/service/article.go`

- [ ] **Step 1：实现 Article Service**

```go
// backend/internal/service/article.go
package service

import (
    "rss-backend/internal/repository"
)

type ArticleService struct {
    artRepo *repository.ArticleRepository
}

func NewArticleService(ar *repository.ArticleRepository) *ArticleService {
    return &ArticleService{artRepo: ar}
}

func (s *ArticleService) ListArticles(f repository.ArticleFilter) (*repository.ArticleListResult, error) {
    return s.artRepo.List(f)
}

// GetArticle 返回详情并自动标记已读
func (s *ArticleService) GetArticle(id uint) (*repository.ArticleWithFeedTitle, error) {
    art, err := s.artRepo.GetByID(id)
    if err != nil {
        return nil, err
    }
    if !art.IsRead {
        s.artRepo.Update(id, map[string]interface{}{"is_read": true})
        art.IsRead = true
    }
    return art, nil
}

func (s *ArticleService) UpdateArticle(id uint, updates map[string]interface{}) (*repository.ArticleWithFeedTitle, error) {
    // 只允许更新 is_read / is_starred
    allowed := map[string]interface{}{}
    if v, ok := updates["is_read"]; ok {
        allowed["is_read"] = v
    }
    if v, ok := updates["is_starred"]; ok {
        allowed["is_starred"] = v
    }
    if len(allowed) == 0 {
        return s.artRepo.GetByID(id)
    }
    if _, err := s.artRepo.Update(id, allowed); err != nil {
        return nil, err
    }
    return s.artRepo.GetByID(id)
}
```

- [ ] **Step 2：提交**

```bash
git add internal/service/article.go
git commit -m "feat: add ArticleService with auto-read on GET"
```

---

### Task 9：HTTP Handler (P0)

**Files:**
- Create: `backend/internal/handler/feed.go`
- Create: `backend/internal/handler/article.go`
- Create: `backend/internal/handler/response.go`

- [ ] **Step 1：创建统一 Response 工具**

```go
// backend/internal/handler/response.go
package handler

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

type ErrResponse struct {
    Code    string `json:"code"`
    Message string `json:"message"`
}

func respondErr(c *gin.Context, status int, code, msg string) {
    c.JSON(status, ErrResponse{Code: code, Message: msg})
}

func respondOK(c *gin.Context, data any) {
    c.JSON(http.StatusOK, data)
}
```

- [ ] **Step 2：实现 Feed Handler**

```go
// backend/internal/handler/feed.go
package handler

import (
    "errors"
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"
    "gorm.io/gorm"
    "rss-backend/internal/service"
)

type FeedHandler struct {
    svc *service.FeedService
}

func NewFeedHandler(svc *service.FeedService) *FeedHandler {
    return &FeedHandler{svc: svc}
}

func (h *FeedHandler) Create(c *gin.Context) {
    var req struct {
        URL string `json:"url" binding:"required,url"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        respondErr(c, http.StatusBadRequest, "INVALID_REQUEST", err.Error())
        return
    }

    feed, err := h.svc.CreateFeed(req.URL)
    if err != nil {
        // 唯一约束冲突
        respondErr(c, http.StatusConflict, "FEED_ALREADY_EXISTS", "该订阅源已添加")
        return
    }
    c.JSON(http.StatusAccepted, feed)
}

func (h *FeedHandler) List(c *gin.Context) {
    feeds, err := h.svc.ListFeeds()
    if err != nil {
        respondErr(c, http.StatusInternalServerError, "INTERNAL_ERROR", err.Error())
        return
    }
    respondOK(c, feeds)
}

func (h *FeedHandler) GetByID(c *gin.Context) {
    id, err := strconv.ParseUint(c.Param("id"), 10, 64)
    if err != nil {
        respondErr(c, http.StatusBadRequest, "INVALID_ID", "无效的 ID")
        return
    }
    feed, err := h.svc.GetFeed(uint(id))
    if errors.Is(err, gorm.ErrRecordNotFound) {
        respondErr(c, http.StatusNotFound, "FEED_NOT_FOUND", "订阅源不存在")
        return
    }
    if err != nil {
        respondErr(c, http.StatusInternalServerError, "INTERNAL_ERROR", err.Error())
        return
    }
    respondOK(c, feed)
}
```

- [ ] **Step 3：实现 Article Handler**

```go
// backend/internal/handler/article.go
package handler

import (
    "errors"
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"
    "gorm.io/gorm"
    "rss-backend/internal/repository"
    "rss-backend/internal/service"
)

type ArticleHandler struct {
    svc *service.ArticleService
}

func NewArticleHandler(svc *service.ArticleService) *ArticleHandler {
    return &ArticleHandler{svc: svc}
}

func (h *ArticleHandler) List(c *gin.Context) {
    f := repository.ArticleFilter{
        Page:     1,
        PageSize: 20,
    }

    if v := c.Query("feed_id"); v != "" {
        id, err := strconv.ParseUint(v, 10, 64)
        if err == nil {
            uid := uint(id)
            f.FeedID = &uid
        }
    }
    if v := c.Query("starred"); v == "1" || v == "true" {
        b := true
        f.Starred = &b
    }
    if v := c.Query("unread"); v == "1" || v == "true" {
        b := true
        f.Unread = &b
    }
    if p, err := strconv.Atoi(c.DefaultQuery("page", "1")); err == nil && p > 0 {
        f.Page = p
    }
    if ps, err := strconv.Atoi(c.DefaultQuery("page_size", "20")); err == nil && ps > 0 && ps <= 100 {
        f.PageSize = ps
    }

    result, err := h.svc.ListArticles(f)
    if err != nil {
        respondErr(c, http.StatusInternalServerError, "INTERNAL_ERROR", err.Error())
        return
    }
    respondOK(c, gin.H{
        "total":     result.Total,
        "page":      f.Page,
        "page_size": f.PageSize,
        "items":     result.Items,
    })
}

func (h *ArticleHandler) GetByID(c *gin.Context) {
    id, err := strconv.ParseUint(c.Param("id"), 10, 64)
    if err != nil {
        respondErr(c, http.StatusBadRequest, "INVALID_ID", "无效的 ID")
        return
    }
    art, err := h.svc.GetArticle(uint(id))
    if errors.Is(err, gorm.ErrRecordNotFound) {
        respondErr(c, http.StatusNotFound, "ARTICLE_NOT_FOUND", "文章不存在")
        return
    }
    if err != nil {
        respondErr(c, http.StatusInternalServerError, "INTERNAL_ERROR", err.Error())
        return
    }
    respondOK(c, art)
}

func (h *ArticleHandler) Update(c *gin.Context) {
    id, err := strconv.ParseUint(c.Param("id"), 10, 64)
    if err != nil {
        respondErr(c, http.StatusBadRequest, "INVALID_ID", "无效的 ID")
        return
    }

    var req map[string]interface{}
    if err := c.ShouldBindJSON(&req); err != nil {
        respondErr(c, http.StatusBadRequest, "INVALID_REQUEST", err.Error())
        return
    }

    art, err := h.svc.UpdateArticle(uint(id), req)
    if errors.Is(err, gorm.ErrRecordNotFound) {
        respondErr(c, http.StatusNotFound, "ARTICLE_NOT_FOUND", "文章不存在")
        return
    }
    if err != nil {
        respondErr(c, http.StatusInternalServerError, "INTERNAL_ERROR", err.Error())
        return
    }
    respondOK(c, art)
}
```

- [ ] **Step 4：提交**

```bash
git add internal/handler/
git commit -m "feat: add HTTP handlers for feeds and articles"
```

---

### Task 10：main.go + 路由

**Files:**
- Create: `backend/main.go`

- [ ] **Step 1：实现 `main.go`**

```go
// backend/main.go
package main

import (
    "fmt"
    "log"

    "github.com/gin-gonic/gin"
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    "rss-backend/config"
    "rss-backend/internal/fetcher"
    "rss-backend/internal/handler"
    "rss-backend/internal/model"
    "rss-backend/internal/repository"
    "rss-backend/internal/service"
)

func main() {
    cfg, err := config.Load()
    if err != nil {
        log.Fatalf("load config: %v", err)
    }

    db, err := gorm.Open(mysql.Open(cfg.Database.DSN), &gorm.Config{})
    if err != nil {
        log.Fatalf("connect db: %v", err)
    }
    db.AutoMigrate(&model.Feed{}, &model.Article{})

    service.InitSemaphore(cfg.Fetcher.MaxConcurrent)

    f := fetcher.New(cfg.Fetcher.TimeoutSeconds, cfg.Fetcher.MaxArticlesPerFeed)

    feedRepo := repository.NewFeedRepository(db)
    artRepo := repository.NewArticleRepository(db)

    feedSvc := service.NewFeedService(feedRepo, artRepo, f)
    artSvc := service.NewArticleService(artRepo)

    feedH := handler.NewFeedHandler(feedSvc)
    artH := handler.NewArticleHandler(artSvc)

    r := gin.Default()
    r.Use(corsMiddleware(cfg.Server.CORSOrigins))

    api := r.Group("/api")
    {
        api.POST("/feeds", feedH.Create)
        api.GET("/feeds", feedH.List)
        api.GET("/feeds/:id", feedH.GetByID)

        api.GET("/articles", artH.List)
        api.GET("/articles/:id", artH.GetByID)
        api.PATCH("/articles/:id", artH.Update)
    }

    addr := fmt.Sprintf(":%d", cfg.Server.Port)
    log.Printf("Server running on %s", addr)
    r.Run(addr)
}

func corsMiddleware(origins []string) gin.HandlerFunc {
    return func(c *gin.Context) {
        origin := c.Request.Header.Get("Origin")
        for _, o := range origins {
            if o == origin {
                c.Header("Access-Control-Allow-Origin", origin)
                break
            }
        }
        c.Header("Access-Control-Allow-Methods", "GET,POST,PATCH,DELETE,OPTIONS")
        c.Header("Access-Control-Allow-Headers", "Content-Type")
        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(204)
            return
        }
        c.Next()
    }
}
```

- [ ] **Step 2：启动后端，确认无报错**

```bash
cd backend
go run main.go
# 预期：Server running on :8080
```

- [ ] **Step 3：冒烟测试**

```bash
curl -s -X POST http://localhost:8080/api/feeds \
  -H "Content-Type: application/json" \
  -d '{"url":"https://feeds.feedburner.com/ruanyifeng"}' | jq .
# 预期：{ "id": 1, "fetch_status": "pending", ... }

curl -s http://localhost:8080/api/feeds | jq .
# 预期：包含刚添加的 feed，状态最终变为 success
```

- [ ] **Step 4：提交**

```bash
git add main.go
git commit -m "feat: wire up main.go with routes and CORS"
```

---

### Task 11：前端 P0 — API Client + 路由

**Files:**
- Create: `frontend/src/api/client.ts`
- Create: `frontend/src/api/feeds.ts`
- Create: `frontend/src/api/articles.ts`
- Modify: `frontend/src/main.tsx`
- Create: `frontend/src/App.tsx`

- [ ] **Step 1：创建 Axios client**

```ts
// frontend/src/api/client.ts
import axios from 'axios'

export const client = axios.create({
  baseURL: '/api',
  headers: { 'Content-Type': 'application/json' },
})

export class ApiError extends Error {
  constructor(public code: string, message: string) {
    super(message)
  }
}

client.interceptors.response.use(
  (res) => res,
  (err) => {
    const data = err.response?.data
    return Promise.reject(new ApiError(data?.code ?? 'UNKNOWN', data?.message ?? err.message))
  }
)
```

- [ ] **Step 2：Feed API 函数**

```ts
// frontend/src/api/feeds.ts
import { client } from './client'

export interface Feed {
  id: number
  url: string
  title: string
  description: string
  fetch_status: 'pending' | 'fetching' | 'success' | 'failed'
  fetch_error: string | null
  source_updated_at: string | null
  last_fetched_at: string | null
  created_at: string
}

export const feedsApi = {
  list: () => client.get<Feed[]>('/feeds').then((r) => r.data),
  create: (url: string) => client.post<Feed>('/feeds', { url }).then((r) => r.data),
  getById: (id: number) => client.get<Feed>(`/feeds/${id}`).then((r) => r.data),
}
```

- [ ] **Step 3：Article API 函数**

```ts
// frontend/src/api/articles.ts
import { client } from './client'

export interface Article {
  id: number
  feed_id: number
  feed_title: string
  title: string
  link: string
  content: string
  author: string
  published_at: string | null
  is_read: boolean
  is_starred: boolean
  is_full_content: boolean
}

export interface ArticleListResponse {
  total: number
  page: number
  page_size: number
  items: Article[]
}

export interface ArticleFilter {
  feed_id?: number
  starred?: boolean
  unread?: boolean
  page?: number
  page_size?: number
}

export const articlesApi = {
  list: (filter: ArticleFilter = {}) => {
    const params = new URLSearchParams()
    if (filter.feed_id != null) params.set('feed_id', String(filter.feed_id))
    if (filter.starred) params.set('starred', '1')
    if (filter.unread) params.set('unread', '1')
    params.set('page', String(filter.page ?? 1))
    params.set('page_size', String(filter.page_size ?? 20))
    return client.get<ArticleListResponse>(`/articles?${params}`).then((r) => r.data)
  },
  getById: (id: number) => client.get<Article>(`/articles/${id}`).then((r) => r.data),
  update: (id: number, updates: Partial<Pick<Article, 'is_read' | 'is_starred'>>) =>
    client.patch<Article>(`/articles/${id}`, updates).then((r) => r.data),
}
```

- [ ] **Step 4：配置 TanStack Query + Router**

```tsx
// frontend/src/main.tsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { BrowserRouter } from 'react-router-dom'
import App from './App'
import './index.css'

const queryClient = new QueryClient({
  defaultOptions: { queries: { retry: 1, staleTime: 30_000 } },
})

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </QueryClientProvider>
  </React.StrictMode>
)
```

```tsx
// frontend/src/App.tsx
import { Navigate, Route, Routes } from 'react-router-dom'
import Layout from './components/Layout'
import ArticlesPage from './pages/ArticlesPage'
import ArticleDetailPage from './pages/ArticleDetailPage'

export default function App() {
  return (
    <Routes>
      <Route path="/" element={<Navigate to="/articles" replace />} />
      <Route element={<Layout />}>
        <Route path="/articles" element={<ArticlesPage />} />
        <Route path="/articles/:id" element={<ArticleDetailPage />} />
      </Route>
    </Routes>
  )
}
```

- [ ] **Step 5：提交**

```bash
git add src/
git commit -m "feat: add API client, feed/article API functions, routing"
```

---

### Task 12：前端 P0 — 布局 + 侧边栏 + AddFeedForm

**Files:**
- Create: `frontend/src/components/Layout.tsx`
- Create: `frontend/src/components/Sidebar.tsx`
- Create: `frontend/src/components/AddFeedForm.tsx`
- Create: `frontend/src/components/FeedList.tsx`
- Create: `frontend/src/components/FeedItem.tsx`
- Create: `frontend/src/components/NavLinks.tsx`

- [ ] **Step 1：Layout**

```tsx
// frontend/src/components/Layout.tsx
import { Outlet } from 'react-router-dom'
import Sidebar from './Sidebar'

export default function Layout() {
  return (
    <div className="flex h-screen overflow-hidden">
      <aside className="w-64 border-r border-gray-200 flex flex-col overflow-y-auto bg-gray-50">
        <Sidebar />
      </aside>
      <main className="flex-1 overflow-y-auto">
        <Outlet />
      </main>
    </div>
  )
}
```

- [ ] **Step 2：AddFeedForm**

```tsx
// frontend/src/components/AddFeedForm.tsx
import { useState } from 'react'
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { feedsApi } from '../api/feeds'
import { ApiError } from '../api/client'

export default function AddFeedForm() {
  const [url, setUrl] = useState('')
  const [error, setError] = useState('')
  const qc = useQueryClient()

  const mutation = useMutation({
    mutationFn: () => feedsApi.create(url.trim()),
    onSuccess: () => {
      setUrl('')
      setError('')
      qc.invalidateQueries({ queryKey: ['feeds'] })
    },
    onError: (err: ApiError) => {
      setError(err.code === 'FEED_ALREADY_EXISTS' ? '该订阅源已添加' : `添加失败: ${err.message}`)
    },
  })

  return (
    <form
      className="p-3 border-b border-gray-200"
      onSubmit={(e) => { e.preventDefault(); mutation.mutate() }}
    >
      <input
        className="w-full border rounded px-2 py-1 text-sm mb-1 focus:outline-none focus:ring-1 focus:ring-blue-400"
        placeholder="输入 RSS 地址"
        value={url}
        onChange={(e) => setUrl(e.target.value)}
      />
      {error && <p className="text-xs text-red-500 mb-1">{error}</p>}
      <button
        type="submit"
        disabled={!url.trim() || mutation.isPending}
        className="w-full bg-blue-500 text-white text-sm rounded py-1 disabled:opacity-50"
      >
        {mutation.isPending ? '添加中...' : '添加订阅'}
      </button>
    </form>
  )
}
```

- [ ] **Step 3：FeedItem + FeedList**

```tsx
// frontend/src/components/FeedItem.tsx
import { useNavigate, useSearchParams } from 'react-router-dom'
import type { Feed } from '../api/feeds'

interface Props { feed: Feed }

export default function FeedItem({ feed }: Props) {
  const [params, setParams] = useSearchParams()
  const active = params.get('feed_id') === String(feed.id)

  const statusIcon = {
    pending: '⏳', fetching: '🔄', success: '', failed: '❌',
  }[feed.fetch_status] ?? ''

  const updatedLabel = feed.source_updated_at
    ? new Date(feed.source_updated_at).toLocaleDateString()
    : '未更新'

  return (
    <button
      onClick={() => setParams({ feed_id: String(feed.id) })}
      className={`w-full text-left px-3 py-2 text-sm hover:bg-gray-100 ${active ? 'bg-blue-50 font-medium' : ''}`}
    >
      <div className="truncate">{statusIcon} {feed.title || feed.url}</div>
      <div className="text-xs text-gray-400">{updatedLabel}</div>
    </button>
  )
}
```

```tsx
// frontend/src/components/FeedList.tsx
import { useQuery } from '@tanstack/react-query'
import { feedsApi } from '../api/feeds'
import FeedItem from './FeedItem'

export default function FeedList() {
  const { data: feeds, isLoading } = useQuery({
    queryKey: ['feeds'],
    queryFn: feedsApi.list,
  })

  if (isLoading) return <p className="p-3 text-sm text-gray-400">加载中...</p>
  if (!feeds?.length) return <p className="p-3 text-sm text-gray-400">暂无订阅源</p>

  return (
    <div>
      {feeds.map((f) => <FeedItem key={f.id} feed={f} />)}
    </div>
  )
}
```

- [ ] **Step 4：NavLinks + Sidebar**

```tsx
// frontend/src/components/NavLinks.tsx
import { useSearchParams } from 'react-router-dom'

export default function NavLinks() {
  const [, setParams] = useSearchParams()
  return (
    <div className="p-3 border-b border-gray-200 space-y-1">
      <button onClick={() => setParams({})} className="w-full text-left text-sm px-2 py-1 rounded hover:bg-gray-100">
        全部文章
      </button>
      <button onClick={() => setParams({ starred: '1' })} className="w-full text-left text-sm px-2 py-1 rounded hover:bg-gray-100">
        ⭐ 收藏
      </button>
    </div>
  )
}
```

```tsx
// frontend/src/components/Sidebar.tsx
import AddFeedForm from './AddFeedForm'
import NavLinks from './NavLinks'
import FeedList from './FeedList'

export default function Sidebar() {
  return (
    <>
      <div className="p-3 font-bold text-gray-700 border-b border-gray-200">RSS 阅读器</div>
      <AddFeedForm />
      <NavLinks />
      <FeedList />
    </>
  )
}
```

- [ ] **Step 5：提交**

```bash
git add src/components/
git commit -m "feat: add sidebar with feed list and add-feed form"
```

---

### Task 13：前端 P0 — 文章列表 + 文章详情

**Files:**
- Create: `frontend/src/components/ArticleItem.tsx`
- Create: `frontend/src/components/ArticleList.tsx`
- Create: `frontend/src/pages/ArticlesPage.tsx`
- Create: `frontend/src/pages/ArticleDetailPage.tsx`

- [ ] **Step 1：ArticleItem**

```tsx
// frontend/src/components/ArticleItem.tsx
import { useNavigate } from 'react-router-dom'
import type { Article } from '../api/articles'

interface Props { article: Article }

export default function ArticleItem({ article: a }: Props) {
  const navigate = useNavigate()

  return (
    <div
      onClick={() => navigate(`/articles/${a.id}`)}
      className={`flex items-start gap-2 px-4 py-3 border-b border-gray-100 cursor-pointer hover:bg-gray-50 ${
        !a.is_read ? 'border-l-4 border-l-blue-500' : 'border-l-4 border-l-transparent'
      }`}
    >
      <div className="flex-1 min-w-0">
        <p className={`text-sm truncate ${a.is_read ? 'text-gray-500 font-normal' : 'text-gray-900 font-semibold'}`}>
          {a.title}
        </p>
        <p className="text-xs text-gray-400 mt-0.5">
          {a.feed_title} · {a.published_at ? new Date(a.published_at).toLocaleDateString() : ''}
        </p>
      </div>
      {a.is_starred && <span className="text-yellow-400 text-sm">⭐</span>}
    </div>
  )
}
```

- [ ] **Step 2：ArticleList**

```tsx
// frontend/src/components/ArticleList.tsx
import { useQuery } from '@tanstack/react-query'
import { articlesApi, type ArticleFilter } from '../api/articles'
import ArticleItem from './ArticleItem'

interface Props { filter: ArticleFilter }

export default function ArticleList({ filter }: Props) {
  const { data, isLoading } = useQuery({
    queryKey: ['articles', filter],
    queryFn: () => articlesApi.list(filter),
  })

  if (isLoading) return <p className="p-4 text-gray-400 text-sm">加载中...</p>
  if (!data?.items?.length) return <p className="p-4 text-gray-400 text-sm">暂无文章</p>

  return (
    <div>
      {data.items.map((a) => <ArticleItem key={a.id} article={a} />)}
    </div>
  )
}
```

- [ ] **Step 3：ArticlesPage（解析 URL 参数传给 ArticleList）**

```tsx
// frontend/src/pages/ArticlesPage.tsx
import { useSearchParams } from 'react-router-dom'
import ArticleList from '../components/ArticleList'
import type { ArticleFilter } from '../api/articles'

export default function ArticlesPage() {
  const [params] = useSearchParams()
  const filter: ArticleFilter = {}
  const feedId = params.get('feed_id')
  if (feedId) filter.feed_id = Number(feedId)
  if (params.get('starred') === '1') filter.starred = true

  return (
    <div className="max-w-2xl mx-auto">
      <ArticleList filter={filter} />
    </div>
  )
}
```

- [ ] **Step 4：ArticleDetailPage**

```tsx
// frontend/src/pages/ArticleDetailPage.tsx
import { useParams, useNavigate } from 'react-router-dom'
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import DOMPurify from 'dompurify'
import { articlesApi } from '../api/articles'

export default function ArticleDetailPage() {
  const { id } = useParams<{ id: string }>()
  const navigate = useNavigate()
  const qc = useQueryClient()

  const { data: article, isLoading } = useQuery({
    queryKey: ['articles', Number(id)],
    queryFn: () => articlesApi.getById(Number(id)),
    enabled: !!id,
    // 查询成功后 invalidate 列表，更新已读状态显示
    onSuccess: () => qc.invalidateQueries({ queryKey: ['articles'] }),
  })

  const starMutation = useMutation({
    mutationFn: () => articlesApi.update(Number(id), { is_starred: !article?.is_starred }),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['articles', Number(id)] }),
  })

  if (isLoading) return <p className="p-6 text-gray-400">加载中...</p>
  if (!article) return <p className="p-6 text-red-400">文章不存在</p>

  const isSubstantial = article.content && article.content.length > 200
  const cleanHTML = isSubstantial
    ? DOMPurify.sanitize(article.content)
    : DOMPurify.sanitize(article.content ?? '')

  return (
    <div className="max-w-2xl mx-auto p-6">
      <button onClick={() => navigate(-1)} className="text-sm text-blue-500 mb-4">← 返回</button>
      <div className="flex items-center justify-between mb-2">
        <h1 className="text-xl font-bold text-gray-900">{article.title}</h1>
        <button onClick={() => starMutation.mutate()} className="text-2xl">
          {article.is_starred ? '⭐' : '☆'}
        </button>
      </div>
      <p className="text-sm text-gray-400 mb-4">
        {article.feed_title} · {article.author} · {article.published_at ? new Date(article.published_at).toLocaleDateString() : ''}
      </p>

      {isSubstantial ? (
        <div
          className="prose prose-sm max-w-none"
          dangerouslySetInnerHTML={{ __html: cleanHTML }}
        />
      ) : (
        <>
          <p className="text-gray-600 text-sm mb-4">{article.content}</p>
          {article.link && (
            <a
              href={article.link}
              target="_blank"
              rel="noopener noreferrer"
              className="inline-block bg-blue-500 text-white text-sm px-4 py-2 rounded"
            >
              阅读原文 →
            </a>
          )}
        </>
      )}
    </div>
  )
}
```

- [ ] **Step 5：启动前端，验证 P0 功能**

```bash
cd frontend
npm run dev
# 浏览器打开 http://localhost:5173
# 验证：添加订阅源 → 订阅源列表 → 文章列表 → 文章详情 → 已读标记
```

- [ ] **Step 6：提交**

```bash
git add src/
git commit -m "feat: P0 - article list and detail with read marking"
```

---

## 阶段 2：P1 功能

### Task 14：后端 P1 — Feed 刷新 + 删除路由

**Files:**
- Modify: `backend/internal/service/feed.go`
- Modify: `backend/internal/handler/feed.go`
- Modify: `backend/main.go`

- [ ] **Step 1：在 FeedService 添加 Refresh 方法**

在 `service/feed.go` 中添加：

```go
func (s *FeedService) RefreshFeed(id uint, minIntervalSecs int) error {
    feed, err := s.feedRepo.GetByID(id)
    if err != nil {
        return err
    }

    // 检查最小刷新间隔（首次 last_fetched_at 为 nil 时不限制）
    if feed.LastFetchedAt != nil {
        elapsed := time.Since(*feed.LastFetchedAt).Seconds()
        if elapsed < float64(minIntervalSecs) {
            return ErrTooSoon
        }
    }

    go s.triggerFetch(id, feed.URL)
    return nil
}

var ErrTooSoon = fmt.Errorf("too soon to refresh")
```

- [ ] **Step 2：在 FeedHandler 添加 Refresh + Delete**

在 `handler/feed.go` 中添加：

```go
func (h *FeedHandler) Refresh(c *gin.Context) {
    id, err := strconv.ParseUint(c.Param("id"), 10, 64)
    if err != nil {
        respondErr(c, http.StatusBadRequest, "INVALID_ID", "无效的 ID")
        return
    }
    err = h.svc.RefreshFeed(uint(id), h.minRefreshInterval)
    if errors.Is(err, gorm.ErrRecordNotFound) {
        respondErr(c, http.StatusNotFound, "FEED_NOT_FOUND", "订阅源不存在")
        return
    }
    if errors.Is(err, service.ErrTooSoon) {
        respondErr(c, http.StatusTooManyRequests, "TOO_SOON", "刷新过于频繁，请稍后再试")
        return
    }
    c.JSON(http.StatusAccepted, gin.H{"message": "刷新任务已触发"})
}

func (h *FeedHandler) Delete(c *gin.Context) {
    id, err := strconv.ParseUint(c.Param("id"), 10, 64)
    if err != nil {
        respondErr(c, http.StatusBadRequest, "INVALID_ID", "无效的 ID")
        return
    }
    if err := h.svc.feedRepo.Delete(uint(id)); err != nil {
        respondErr(c, http.StatusInternalServerError, "INTERNAL_ERROR", err.Error())
        return
    }
    c.Status(http.StatusNoContent)
}
```

更新 `FeedHandler` 结构体，加入 `minRefreshInterval int` 字段，在 `NewFeedHandler` 中传入。

- [ ] **Step 3：在 main.go 注册新路由**

```go
api.POST("/feeds/:id/refresh", feedH.Refresh)
api.DELETE("/feeds/:id", feedH.Delete)
```

- [ ] **Step 4：冒烟测试**

```bash
# 刷新
curl -s -X POST http://localhost:8080/api/feeds/1/refresh | jq .
# 预期：{ "message": "刷新任务已触发" }

# 立即再刷新，触发 429
curl -s -X POST http://localhost:8080/api/feeds/1/refresh | jq .
# 预期：{ "code": "TOO_SOON", ... }
```

- [ ] **Step 5：提交**

```bash
git add .
git commit -m "feat: P1 - add feed refresh with rate limiting and delete endpoint"
```

---

### Task 15：前端 P1 — 刷新按钮 + 筛选 + 收藏

**Files:**
- Modify: `frontend/src/components/FeedItem.tsx`
- Modify: `frontend/src/components/NavLinks.tsx`

- [ ] **Step 1：在 feeds.ts 添加 refresh + delete API**

```ts
// 追加到 frontend/src/api/feeds.ts
// 在 feedsApi 对象中添加：
refresh: (id: number) => client.post(`/feeds/${id}/refresh`).then((r) => r.data),
delete: (id: number) => client.delete(`/feeds/${id}`).then((r) => r.data),
```

- [ ] **Step 2：更新 FeedItem，加入刷新按钮和删除按钮**

```tsx
// frontend/src/components/FeedItem.tsx
import { useNavigate, useSearchParams } from 'react-router-dom'
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { feedsApi, type Feed } from '../api/feeds'

interface Props { feed: Feed }

export default function FeedItem({ feed }: Props) {
  const [params, setParams] = useSearchParams()
  const qc = useQueryClient()
  const active = params.get('feed_id') === String(feed.id)

  const refreshMutation = useMutation({
    mutationFn: () => feedsApi.refresh(feed.id),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['feeds'] }),
  })

  const deleteMutation = useMutation({
    mutationFn: () => feedsApi.delete(feed.id),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: ['feeds'] })
      qc.invalidateQueries({ queryKey: ['articles'] })
      setParams({})
    },
  })

  const statusIcon = {
    pending: '⏳', fetching: '🔄', success: '', failed: '❌',
  }[feed.fetch_status] ?? ''

  const updatedLabel = feed.source_updated_at
    ? new Date(feed.source_updated_at).toLocaleDateString()
    : '未更新'

  return (
    <div className={`group flex items-center gap-1 px-2 py-1.5 hover:bg-gray-100 ${active ? 'bg-blue-50' : ''}`}>
      <button
        className="flex-1 text-left text-sm truncate"
        onClick={() => setParams({ feed_id: String(feed.id) })}
      >
        <span>{statusIcon} {feed.title || feed.url}</span>
        <span className="block text-xs text-gray-400">{updatedLabel}</span>
      </button>
      <button
        onClick={(e) => { e.stopPropagation(); refreshMutation.mutate() }}
        disabled={refreshMutation.isPending || feed.fetch_status === 'fetching'}
        className="opacity-0 group-hover:opacity-100 text-xs text-blue-400 px-1"
        title="刷新"
      >↺</button>
      <button
        onClick={(e) => { e.stopPropagation(); deleteMutation.mutate() }}
        className="opacity-0 group-hover:opacity-100 text-xs text-red-400 px-1"
        title="删除"
      >✕</button>
    </div>
  )
}
```

- [ ] **Step 3：验证 P1 功能**

```
浏览器验证：
- hover FeedItem 显示刷新/删除按钮
- 点刷新后 fetch_status 变为 fetching → success
- 点删除后该源从列表消失，文章列表清空
- 点"收藏"NavLink 只显示 is_starred=true 的文章
- 点某个订阅源名称只显示该源文章
```

- [ ] **Step 4：提交**

```bash
git add src/
git commit -m "feat: P1 - feed refresh button, delete, filter by feed and starred"
```

---

## 阶段 3：P2 功能

### Task 16：后端 P2 — 全文抓取

**Files:**
- Create: `backend/internal/service/fulltext.go`
- Modify: `backend/internal/handler/article.go`
- Modify: `backend/main.go`

- [ ] **Step 1：安装依赖**

```bash
cd backend
go get golang.org/x/net/html@latest
```

- [ ] **Step 2：实现全文抓取 Service**

```go
// backend/internal/service/fulltext.go
package service

import (
    "context"
    "fmt"
    "net/http"
    "strings"
    "time"

    "golang.org/x/net/html"
    "rss-backend/internal/repository"
)

type FulltextService struct {
    artRepo *repository.ArticleRepository
    client  *http.Client
}

func NewFulltextService(ar *repository.ArticleRepository) *FulltextService {
    return &FulltextService{
        artRepo: ar,
        client:  &http.Client{Timeout: 15 * time.Second},
    }
}

// FetchFulltext 判断是否为摘要型，若是则抓取原文并更新 content
func (s *FulltextService) FetchFulltext(id uint) (*repository.ArticleWithFeedTitle, error) {
    art, err := s.artRepo.GetByID(id)
    if err != nil {
        return nil, err
    }

    // 已有全文，直接返回
    if art.IsFullContent || len([]rune(art.Content)) >= 500 {
        return art, nil
    }

    if art.Link == "" {
        return art, nil
    }

    ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, art.Link, nil)
    req.Header.Set("User-Agent", "Mozilla/5.0 RSS-Reader/1.0")
    resp, err := s.client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("fetch fulltext: %w", err)
    }
    defer resp.Body.Close()

    doc, err := html.Parse(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("parse html: %w", err)
    }

    content := extractMainContent(doc)
    if content == "" {
        return art, nil
    }

    s.artRepo.Update(id, map[string]interface{}{
        "content":          content,
        "is_full_content":  true,
    })
    art.Content = content
    art.IsFullContent = true
    return art, nil
}

// extractMainContent 优先取 <article>，降级取 <main>，再降级取最长 <div>
func extractMainContent(doc *html.Node) string {
    if n := findNode(doc, "article"); n != nil {
        return nodeText(n)
    }
    if n := findNode(doc, "main"); n != nil {
        return nodeText(n)
    }
    return longestDiv(doc)
}

func findNode(n *html.Node, tag string) *html.Node {
    if n.Type == html.ElementNode && n.Data == tag {
        return n
    }
    for c := n.FirstChild; c != nil; c = c.NextSibling {
        if found := findNode(c, tag); found != nil {
            return found
        }
    }
    return nil
}

func nodeText(n *html.Node) string {
    var sb strings.Builder
    var walk func(*html.Node)
    walk = func(node *html.Node) {
        if node.Type == html.TextNode {
            sb.WriteString(node.Data)
        }
        for c := node.FirstChild; c != nil; c = c.NextSibling {
            walk(c)
        }
    }
    walk(n)
    return strings.TrimSpace(sb.String())
}

func longestDiv(doc *html.Node) string {
    var longest string
    var walk func(*html.Node)
    walk = func(n *html.Node) {
        if n.Type == html.ElementNode && n.Data == "div" {
            t := nodeText(n)
            if len(t) > len(longest) {
                longest = t
            }
        }
        for c := n.FirstChild; c != nil; c = c.NextSibling {
            walk(c)
        }
    }
    walk(doc)
    return longest
}
```

- [ ] **Step 3：在 ArticleHandler 添加 Fulltext 路由**

在 `handler/article.go` 中添加：

```go
type ArticleHandler struct {
    svc         *service.ArticleService
    fulltextSvc *service.FulltextService
}

func NewArticleHandler(svc *service.ArticleService, ftSvc *service.FulltextService) *ArticleHandler {
    return &ArticleHandler{svc: svc, fulltextSvc: ftSvc}
}

func (h *ArticleHandler) FetchFulltext(c *gin.Context) {
    id, err := strconv.ParseUint(c.Param("id"), 10, 64)
    if err != nil {
        respondErr(c, http.StatusBadRequest, "INVALID_ID", "无效的 ID")
        return
    }
    art, err := h.fulltextSvc.FetchFulltext(uint(id))
    if errors.Is(err, gorm.ErrRecordNotFound) {
        respondErr(c, http.StatusNotFound, "ARTICLE_NOT_FOUND", "文章不存在")
        return
    }
    if err != nil {
        respondErr(c, http.StatusInternalServerError, "FETCH_FAILED", err.Error())
        return
    }
    respondOK(c, art)
}
```

- [ ] **Step 4：在 main.go 注册路由**

```go
ftSvc := service.NewFulltextService(artRepo)
artH := handler.NewArticleHandler(artSvc, ftSvc)
// ...
api.GET("/articles/:id/fulltext", artH.FetchFulltext)
```

- [ ] **Step 5：提交**

```bash
git add internal/service/fulltext.go internal/handler/article.go main.go
git commit -m "feat: P2 - full-text fetch endpoint"
```

---

### Task 17：前端 P2 — 全文抓取触发

**Files:**
- Modify: `frontend/src/api/articles.ts`
- Modify: `frontend/src/pages/ArticleDetailPage.tsx`

- [ ] **Step 1：在 articles.ts 添加 fulltext API**

```ts
// 追加到 articlesApi
fetchFulltext: (id: number) =>
  client.get<Article>(`/articles/${id}/fulltext`).then((r) => r.data),
```

- [ ] **Step 2：在 ArticleDetailPage 中添加全文抓取按钮**

在摘要展示区域（`!isSubstantial` 分支）中加入：

```tsx
const fulltextMutation = useMutation({
  mutationFn: () => articlesApi.fetchFulltext(Number(id)),
  onSuccess: (data) => {
    qc.setQueryData(['articles', Number(id)], data)
  },
})

// 在"阅读原文"按钮旁边添加：
{!article.is_full_content && (
  <button
    onClick={() => fulltextMutation.mutate()}
    disabled={fulltextMutation.isPending}
    className="ml-2 inline-block border border-blue-500 text-blue-500 text-sm px-4 py-2 rounded"
  >
    {fulltextMutation.isPending ? '抓取中...' : '抓取全文'}
  </button>
)}
```

- [ ] **Step 3：验证 P2 功能**

```
浏览器验证：
- 点开摘要型文章，显示摘要 + "阅读原文" + "抓取全文"按钮
- 点"抓取全文"后按钮变"抓取中..."
- 成功后页面切换为全文渲染，按钮消失
```

- [ ] **Step 4：提交**

```bash
git add src/
git commit -m "feat: P2 - full-text fetch trigger in article detail"
```

---

## 自审核（Spec Coverage）

| 规格需求 | 对应 Task |
|---------|----------|
| P0 添加订阅源 | Task 5, 7, 9, 11, 12 |
| P0 订阅源列表（名称、更新时间） | Task 5, 9, 12（source_updated_at）|
| P0 文章列表（标题、来源、时间） | Task 6, 8, 10, 13 |
| P0 文章详情（展示全文或跳转原文） | Task 8, 10, 13（DOMPurify + link 按钮）|
| P0 已读/未读标记（GET 自动触发） | Task 8, 13 |
| P1 刷新订阅源（+频率限制） | Task 14, 15 |
| P1 按订阅源筛选 | Task 10, 13（feed_id 参数）|
| P1 收藏功能 + 收藏列表 | Task 10, 13, 15 |
| P2 删除订阅源 | Task 14, 15 |
| P2 全文抓取 | Task 16, 17 |
| 幂等入库（guid_hash） | Task 6 |
| 抓取频率控制（信号量 + 间隔） | Task 7, 14 |
| 多格式 RSS 规范化 | Task 4 |
| 刷新页面不丢失数据 | Task 5, 6（MySQL 持久化）|
