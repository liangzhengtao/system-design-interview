# Changelog

# 更新日志

All notable changes to this project will be documented in this file.

本文件记录项目的所有重要更改。

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.0.0/)。

## [1.0.0] - 2025-01-01

### Added / 新增

#### Concepts / 核心概念

- **fundamentals.md** — Scalability, CAP theorem, consistency models, latency vs throughput
- **load-balancing.md** — Load balancing strategies, algorithms, L4/L7, consistent hashing
- **caching.md** — Redis, CDN, browser cache, cache strategies, invalidation patterns
- **database-design.md** — SQL vs NoSQL, indexing, sharding, replication, ACID properties
- **message-queues.md** — Kafka, RabbitMQ, event-driven architecture, exactly-once delivery

#### System Designs / 系统设计

- **url-shortener.md** — Complete URL shortener design with Base62 encoding and analytics
- **chat-system.md** — WhatsApp-like chat system with WebSocket and message delivery
- **news-feed.md** — Twitter-like news feed with fan-out and ranking algorithms
- **search-engine.md** — Search engine design with inverted index and crawling
- **distributed-cache.md** — Redis-like distributed cache with consistent hashing

#### Community / 社区

- README.md with bilingual content (English + 中文)
- CONTRIBUTING.md with contribution guidelines
- CODE_OF_CONDUCT.md based on Contributor Covenant
- SECURITY.md with security policy
- LICENSE (MIT)
- .gitignore
- GitHub Actions CI workflow
- Issue templates (bug report, feature request)
- Pull request template

### Infrastructure / 基础设施

- GitHub Actions workflow for Markdown linting and link checking
- Automated table of contents generation
- Spell checking configuration

---

## [Unreleased] / 未发布

### Planned / 计划中

- Additional system designs (Rate Limiter, Notification System, Video Streaming)
- Anki flashcard deck for spaced repetition
- Interactive ASCII diagram viewer
- Video walkthroughs for each design

---

## How to Update This File / 如何更新此文件

When making changes, add entries under the `[Unreleased]` section. When releasing a new version:

更改时，请在 `[Unreleased]` 部分下添加条目。发布新版本时：

1. Move `[Unreleased]` items to a new version section
2. Update the version number and date
3. Follow [Keep a Changelog](https://keepachangelog.com/) format
