# System Design Interview Guide

# 系统设计面试指南

**Ace your system design interview. 10 concepts + 5 complete system designs with ASCII diagrams.**

**助你拿下系统设计面试。10 个核心概念 + 5 个完整系统设计，配有 ASCII 架构图。**

---

<p align="center">
  <a href="#concepts">Concepts</a> •
  <a href="#system-designs">System Designs</a> •
  <a href="#how-to-use">How to Use</a> •
  <a href="#contribute">Contribute</a>
</p>

---

## Why This Guide? / 为什么选择本指南？

System design interviews are the most challenging part of technical interviews at top companies. This guide provides:

系统设计面试是顶级公司技术面试中最具挑战性的部分。本指南提供：

- **Structured framework** — Step-by-step approach to any design question
- **ASCII diagrams** — Visual architecture that you can draw on a whiteboard
- **Trade-off analysis** — Not just one answer, but multiple approaches with pros/cons
- **Capacity estimation** — Real numbers to make your design concrete
- **Bilingual content** — Every file has a complete 中文版本

---

## Table of Contents / 目录

- [Concepts / 核心概念](#concepts)
- [System Designs / 系统设计](#system-designs)
- [How to Use / 使用指南](#how-to-use)
- [Contributing / 贡献](#contribute)
- [License](#license)

---

## Concepts

Foundation knowledge you need before tackling system design questions.

在解决系统设计问题之前，你需要掌握的基础知识。

| # | Concept / 概念 | Key Topics / 核心主题 |
|---|---------------|----------------------|
| 1 | [Fundamentals / 基础理论](concepts/fundamentals.md) | Scalability, CAP theorem, consistency models, latency vs throughput |
| 2 | [Load Balancing / 负载均衡](concepts/load-balancing.md) | L4/L7, round-robin, consistent hashing, health checks |
| 3 | [Caching / 缓存](concepts/caching.md) | Redis, CDN, browser cache, cache strategies, invalidation |
| 4 | [Database Design / 数据库设计](concepts/database-design.md) | SQL vs NoSQL, indexing, sharding, replication, ACID |
| 5 | [Message Queues / 消息队列](concepts/message-queues.md) | Kafka, RabbitMQ, event-driven architecture, exactly-once |

### Concept Map / 概念图

```
                        ┌─────────────────────────┐
                        │    System Design Fundamentals    │
                        │         基础理论                 │
                        └────────────┬────────────┘
                                     │
                 ┌───────────────────┼───────────────────┐
                 │                   │                   │
          ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
          │ Load Balancing│    │   Caching   │    │  Database   │
          │   负载均衡     │    │    缓存     │    │   数据库    │
          └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
                 │                   │                   │
                 └───────────────────┼───────────────────┘
                                     │
                             ┌───────▼───────┐
                             │ Message Queues │
                             │    消息队列     │
                             └───────┬───────┘
                                     │
                        ┌────────────▼────────────┐
                        │    System Designs        │
                        │    完整系统设计           │
                        └─────────────────────────┘
```

---

## System Designs

Complete end-to-end solutions for popular system design interview questions.

热门系统设计面试题的完整端到端解决方案。

| # | Design / 设计 | Difficulty / 难度 | Key Concepts / 关键概念 |
|---|--------------|-------------------|------------------------|
| 1 | [URL Shortener / 短链系统](designs/url-shortener.md) | ⭐⭐ | Base62 encoding, hashing, redirect, analytics |
| 2 | [Chat System / 聊天系统](designs/chat-system.md) | ⭐⭐⭐ | WebSocket, message queue, presence, delivery |
| 3 | [News Feed / 信息流](designs/news-feed.md) | ⭐⭐⭐ | Fan-out, ranking, timeline, social graph |
| 4 | [Search Engine / 搜索引擎](designs/search-engine.md) | ⭐⭐⭐⭐ | Inverted index, crawling, ranking, NLP |
| 5 | [Distributed Cache / 分布式缓存](designs/distributed-cache.md) | ⭐⭐⭐⭐ | Consistent hashing, replication, eviction |

---

## How to Use

### For Interview Preparation / 面试准备

```
Step 1: Read concepts/fundamentals.md to build your foundation
Step 2: Study each concept file (2-3 hours total)
Step 3: Practice system designs (1-2 hours each)
Step 4: Time yourself — aim for 45 minutes per design
Step 5: Review trade-offs and practice explaining decisions
```

### For Interview Day / 面试当天

```
┌─────────────────────────────────────────────────┐
│           45-Minute Interview Structure         │
│              45 分钟面试结构                      │
├─────────────────────────────────────────────────┤
│                                                 │
│  0-5 min    Requirements clarification           │
│             需求澄清                              │
│                                                 │
│  5-10 min   High-level design                   │
│             高层设计                               │
│                                                 │
│  10-25 min  Deep dive into core components       │
│             深入核心组件                            │
│                                                 │
│  25-35 min  Scaling & trade-offs                │
│             扩展性与权衡分析                        │
│                                                 │
│  35-45 min  Monitoring & failure scenarios       │
│             监控与故障场景                          │
│                                                 │
└─────────────────────────────────────────────────┘
```

### The RESHADED Framework / RESHADED 框架

Each design in this guide follows the **RESHADED** framework:

本指南中每个设计都遵循 **RESHADED** 框架：

```
R - Requirements       需求分析
E - Estimation          容量估算
S - Storage design      存储设计
H - High-level design   高层设计
A - API design          API 设计
D - Detailed design     详细设计
E - Evaluation          评估权衡
D - Distinctive component  差异化组件
```

---

## Roadmap / 路线图

- [x] 5 core concept files
- [x] 5 complete system designs
- [x] Bilingual content (English + 中文)
- [ ] 5 more designs (Rate Limiter, Notification System, Video Streaming, Web Crawler, Key-Value Store)
- [ ] Anki flashcards for spaced repetition
- [ ] Interactive diagrams

---

## Contribute

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

欢迎贡献！详情请参阅 [CONTRIBUTING.md](CONTRIBUTING.md)。

---

## Star History / Star 历史

If you find this guide helpful, please give it a star!

如果你觉得这个指南有帮助，请给我们一个 Star！

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

本项目基于 MIT 许可证开源 - 详情请参阅 [LICENSE](LICENSE) 文件。

---

<p align="center">
  Made with ❤️ by <a href="https://github.com/liangzhengtao">liangzhengtao</a>
</p>
