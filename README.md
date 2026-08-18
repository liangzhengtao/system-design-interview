[中文版](README.zh.md)
n<div align="center">

<img src=".banner.svg" width="100%" alt="banner">

</div>


# System Design Interview Guide

**Ace your system design interview. 10 concepts + 5 complete system designs with ASCII diagrams.**

---

<p align="center">
  <a href="#concepts">Concepts</a> •
  <a href="#system-designs">System Designs</a> •
  <a href="#how-to-use">How to Use</a> •
  <a href="#contribute">Contribute</a>
</p>

---

## Why This Guide?

System design interviews are the most challenging part of technical interviews at top companies. This guide provides:

- **Structured framework** — Step-by-step approach to any design question
- **ASCII diagrams** — Visual architecture that you can draw on a whiteboard
- **Trade-off analysis** — Not just one answer, but multiple approaches with pros/cons
- **Capacity estimation** — Real numbers to make your design concrete
- **Bilingual content** — Every file has a complete Chinese version

---

## Table of Contents

- [Concepts](#concepts)
- [System Designs](#system-designs)
- [How to Use](#how-to-use)
- [Contributing](#contribute)
- [License](#license)

---

## Concepts

Foundation knowledge you need before tackling system design questions.

| # | Concept | Key Topics |
|---|---------|------------|
| 1 | [Fundamentals](concepts/fundamentals.md) | Scalability, CAP theorem, consistency models, latency vs throughput |
| 2 | [Load Balancing](concepts/load-balancing.md) | L4/L7, round-robin, consistent hashing, health checks |
| 3 | [Caching](concepts/caching.md) | Redis, CDN, browser cache, cache strategies, invalidation |
| 4 | [Database Design](concepts/database-design.md) | SQL vs NoSQL, indexing, sharding, replication, ACID |
| 5 | [Message Queues](concepts/message-queues.md) | Kafka, RabbitMQ, event-driven architecture, exactly-once |

### Concept Map

```
                        ┌─────────────────────────┐
                        │  System Design           │
                        │  Fundamentals            │
                        └────────────┬────────────┘
                                     │
                 ┌───────────────────┼───────────────────┐
                 │                   │                   │
          ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
          │ Load Balancing│    │   Caching   │    │  Database   │
          └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
                 │                   │                   │
                 └───────────────────┼───────────────────┘
                                     │
                             ┌───────▼───────┐
                             │ Message Queues │
                             └───────┬───────┘
                                     │
                        ┌────────────▼────────────┐
                        │    System Designs        │
                        └─────────────────────────┘
```

---

## System Designs

Complete end-to-end solutions for popular system design interview questions.

| # | Design | Difficulty | Key Concepts |
|---|--------|------------|--------------|
| 1 | [URL Shortener](designs/url-shortener.md) | ⭐⭐ | Base62 encoding, hashing, redirect, analytics |
| 2 | [Chat System](designs/chat-system.md) | ⭐⭐⭐ | WebSocket, message queue, presence, delivery |
| 3 | [News Feed](designs/news-feed.md) | ⭐⭐⭐ | Fan-out, ranking, timeline, social graph |
| 4 | [Search Engine](designs/search-engine.md) | ⭐⭐⭐⭐ | Inverted index, crawling, ranking, NLP |
| 5 | [Distributed Cache](designs/distributed-cache.md) | ⭐⭐⭐⭐ | Consistent hashing, replication, eviction |

---

## How to Use

### For Interview Preparation

```
Step 1: Read concepts/fundamentals.md to build your foundation
Step 2: Study each concept file (2-3 hours total)
Step 3: Practice system designs (1-2 hours each)
Step 4: Time yourself — aim for 45 minutes per design
Step 5: Review trade-offs and practice explaining decisions
```

### For Interview Day

```
┌─────────────────────────────────────────────────┐
│           45-Minute Interview Structure          │
├─────────────────────────────────────────────────┤
│                                                 │
│  0-5 min    Requirements clarification           │
│                                                 │
│  5-10 min   High-level design                   │
│                                                 │
│  10-25 min  Deep dive into core components       │
│                                                 │
│  25-35 min  Scaling & trade-offs                │
│                                                 │
│  35-45 min  Monitoring & failure scenarios       │
│                                                 │
└─────────────────────────────────────────────────┘
```

### The RESHADED Framework

Each design in this guide follows the **RESHADED** framework:

```
R - Requirements       Requirements analysis
E - Estimation          Capacity estimation
S - Storage design      Storage design
H - High-level design   High-level design
A - API design          API design
D - Detailed design     Detailed design
E - Evaluation          Trade-off evaluation
D - Distinctive component  Differentiating component
```

---

## Roadmap

- [x] 5 core concept files
- [x] 5 complete system designs
- [x] Bilingual content (English + Chinese)
- [ ] 5 more designs (Rate Limiter, Notification System, Video Streaming, Web Crawler, Key-Value Store)
- [ ] Anki flashcards for spaced repetition
- [ ] Interactive diagrams

---

## Contribute

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

---

## Star History

If you find this guide helpful, please give it a star!

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

<p align="center">
  Made with ❤️ by <a href="https://github.com/liangzhengtao">liangzhengtao</a>
</p>
