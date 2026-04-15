# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个 **RSS 订阅阅读器**笔试项目，技术栈为 React + TypeScript，后端和本地数据库自选。

## 功能优先级

### P0（核心，必须完成）
- 添加 RSS 订阅源（输入 URL → 保存到列表）
- 订阅源列表（名称、更新时间）
- 文章列表（标题、来源、发布时间）
- 文章详情（展示全文或跳转原文）
- 已读/未读标记（点击后自动标记，视觉区分）

### P1
- 手动刷新某个订阅源
- 按订阅源筛选文章
- 收藏文章 + 收藏列表

### P2
- 删除订阅源
- 全文抓取（摘要型 RSS 抓取原文）

## 关键约束

- 状态变更必须持久化，**刷新页面不丢失数据**
- 需处理空状态和边界情况（如无订阅源、RSS 解析失败）
- RSS 源通常有跨域问题，后端需代理 RSS 请求

## 推荐技术选型

| 层级 | 推荐 |
|------|------|
| 前端 | React + TypeScript + Vite |
| 样式 | Tailwind CSS 或 shadcn/ui |
| 后端 | Express / Fastify（Node.js） |
| RSS 解析 | `rss-parser`（npm） |
| 数据库 | SQLite（`better-sqlite3`）或 lowdb |

## 数据结构参考

```typescript
interface Feed {
  id: string;
  url: string;
  title: string;
  lastFetched: string; // ISO 日期
}

interface Article {
  id: string;
  feedId: string;
  title: string;
  link: string;
  content: string;
  publishedAt: string;
  isRead: boolean;
  isStarred: boolean;
}
```

## 开发命令（待项目初始化后填写）

项目尚未初始化，初始化后按实际情况补充以下命令：

```bash
# 安装依赖
npm install

# 启动前端开发服务器
npm run dev

# 启动后端
npm run server

# 构建
npm run build
```
