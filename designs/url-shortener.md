# Design a URL Shortener

# 设计短链系统

Design a service like TinyURL or bit.ly that creates short aliases for long URLs.

设计一个类似 TinyURL 或 bit.ly 的短链服务。

---

## Table of Contents / 目录

- [Requirements / 需求分析](#requirements)
- [Capacity Estimation / 容量估算](#capacity-estimation)
- [High-Level Design / 高层设计](#high-level-design)
- [API Design / API 设计](#api-design)
- [Database Schema / 数据库 Schema](#database-schema)
- [Short URL Generation / 短链生成](#short-url-generation)
- [Detailed Design / 详细设计](#detailed-design)
- [Scaling Strategy / 扩展策略](#scaling-strategy)
- [Trade-offs / 权衡分析](#trade-offs)
- [中文版本](#中文版本)

---

## Requirements

### Functional Requirements / 功能需求

1. Given a long URL, generate a short unique URL
2. Given a short URL, redirect to the original long URL
3. Custom aliases (optional, premium feature)
4. Link expiration (optional)
5. Analytics: click count, referrer, geography

### Non-Functional Requirements / 非功能需求

1. High availability (redirects must work 99.99%)
2. Low latency (redirect < 50ms)
3. Short URLs should not be predictable
4. Handle high read-to-write ratio (100:1)

---

## Capacity Estimation

### Traffic Estimates / 流量估算

```
Assumptions:
  - 100M new URLs created per month
  - 100:1 read:write ratio
  - 10B reads per month
  - URL lifetime: 5 years

Write QPS:
  100M / 30 / 86400 ≈ 40 URLs/sec

Read QPS:
  10B / 30 / 86400 ≈ 4,000 URLs/sec

Peak QPS (3x average):
  Write: ~120/sec
  Read: ~12,000/sec
```

### Storage Estimates / 存储估算

```
Per URL record:
  - short_url:    7 bytes
  - long_url:    500 bytes (average)
  - created_at:   8 bytes
  - expires_at:   8 bytes
  - user_id:      8 bytes
  - Total:      ~530 bytes

5 years storage:
  100M/month × 12 × 5 = 6B URLs
  6B × 530 bytes = 3.18 TB

With indexes and overhead: ~5 TB
```

### Bandwidth Estimates / 带宽估算

```
Write bandwidth:
  40 URLs/sec × 530 bytes = 21 KB/sec

Read bandwidth:
  4,000 URLs/sec × 530 bytes = 2.1 MB/sec
```

### Summary Table / 汇总表

| Metric | Value |
|--------|-------|
| Write QPS | 40/sec (peak 120) |
| Read QPS | 4,000/sec (peak 12,000) |
| Storage (5yr) | ~5 TB |
| Write Bandwidth | 21 KB/sec |
| Read Bandwidth | 2.1 MB/sec |

---

## High-Level Design

```
┌─────────────────────────────────────────────────────────────────┐
│                    URL Shortener Architecture                    │
│                                                                 │
│  ┌──────────┐                                                  │
│  │  Client  │                                                  │
│  └────┬─────┘                                                  │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────┐                                                  │
│  │   Load   │                                                  │
│  │ Balancer │                                                  │
│  └────┬─────┘                                                  │
│       │                                                         │
│  ┌────▼─────────────────────────────────────────────────┐      │
│  │              API Servers (Stateless)                   │      │
│  │                                                       │      │
│  │  POST /api/urls ─── Create short URL                 │      │
│  │  GET  /{short}   ─── Redirect to long URL            │      │
│  │  GET  /api/urls/{id} ─── Get URL details             │      │
│  │  GET  /api/urls/{id}/stats ─── Get analytics         │      │
│  └──────┬─────────────────────┬──────────────────────────┘      │
│         │                     │                                 │
│         ▼                     ▼                                 │
│  ┌─────────────┐      ┌─────────────┐                         │
│  │  URL Cache  │      │  Analytics  │                         │
│  │  (Redis)    │      │  (Kafka)    │                         │
│  └──────┬──────┘      └──────┬──────┘                         │
│         │                     │                                 │
│         ▼                     ▼                                 │
│  ┌─────────────┐      ┌─────────────┐                         │
│  │  URL DB     │      │ Analytics DB│                         │
│  │ (PostgreSQL)│      │(ClickHouse) │                         │
│  └─────────────┘      └─────────────┘                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## API Design

### Create Short URL / 创建短链

```http
POST /api/urls
Content-Type: application/json

{
  "long_url": "https://www.example.com/very/long/path/to/resource",
  "custom_alias": "my-link",        // optional
  "expires_at": "2025-12-31T23:59:59Z",  // optional
  "user_id": "user123"              // optional
}

Response (201 Created):
{
  "short_url": "https://short.ly/aBcDeF",
  "long_url": "https://www.example.com/very/long/path/to/resource",
  "created_at": "2025-01-15T10:30:00Z",
  "expires_at": "2025-12-31T23:59:59Z"
}
```

### Redirect / 重定向

```http
GET /{short_url_id}

Response (301 Moved Permanently):
Location: https://www.example.com/very/long/path/to/resource

// 301: Browser caches redirect (reduces server load)
// 302: Server sees every request (needed for analytics)
```

### Get URL Details / 获取 URL 详情

```http
GET /api/urls/{short_url_id}

Response (200 OK):
{
  "short_url": "https://short.ly/aBcDeF",
  "long_url": "https://www.example.com/very/long/path/to/resource",
  "created_at": "2025-01-15T10:30:00Z",
  "clicks": 1234,
  "creator_ip": "192.168.1.1"
}
```

### Get Analytics / 获取分析数据

```http
GET /api/urls/{short_url_id}/stats?period=7d

Response (200 OK):
{
  "total_clicks": 1234,
  "unique_clicks": 890,
  "daily_clicks": [
    {"date": "2025-01-15", "clicks": 150},
    {"date": "2025-01-16", "clicks": 200}
  ],
  "top_referrers": [
    {"referrer": "google.com", "clicks": 400},
    {"referrer": "twitter.com", "clicks": 300}
  ],
  "top_countries": [
    {"country": "US", "clicks": 500},
    {"country": "CN", "clicks": 300}
  ]
}
```

---

## Database Schema

### URL Table / URL 表

```sql
CREATE TABLE urls (
    id            BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code    VARCHAR(7) UNIQUE NOT NULL,
    long_url      TEXT NOT NULL,
    custom_alias  VARCHAR(50) UNIQUE,
    user_id       BIGINT,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at    TIMESTAMP,
    click_count   BIGINT DEFAULT 0,

    INDEX idx_short_code (short_code),
    INDEX idx_custom_alias (custom_alias),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);
```

### Click Analytics Table / 点击分析表

```sql
CREATE TABLE click_events (
    id            BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code    VARCHAR(7) NOT NULL,
    clicked_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address    VARCHAR(45),
    user_agent    TEXT,
    referrer      VARCHAR(2048),
    country       VARCHAR(2),
    city          VARCHAR(100),

    INDEX idx_short_code (short_code),
    INDEX idx_clicked_at (clicked_at)
) PARTITION BY RANGE (clicked_at);
```

---

## Short URL Generation

### Approach 1: Base62 Encoding / Base62 编码

```
Characters: 0-9, a-z, A-Z (62 characters)
Length: 7 characters
Combinations: 62^7 = 3.5 trillion possible URLs

Global Counter Approach:
  1. Use a distributed counter (Redis INCR)
  2. Convert counter to Base62

  Counter: 1234567890
  Base62:  "1LY7VK"

  ┌────────┐  INCR    ┌─────────┐  Base62   ┌──────────┐
  │ Server │─────────▶│  Redis  │──────────▶│ Short URL│
  │        │          │ Counter │  Encode   │ "1LY7VK" │
  └────────┘          └─────────┘          └──────────┘

  Pros: Guaranteed unique, simple
  Cons: Sequential (predictable), single counter bottleneck
```

### Approach 2: MD5/SHA256 Hash / 哈希取子串

```
1. Hash the long URL: MD5(long_url) → 128-bit hash
2. Take first 7 characters (Base62 encoded)
3. Check for collision in DB
4. If collision, take next 7 characters

  long_url: "https://example.com/page"
  MD5: "d41d8cd98f00b204e9800998ecf8427e"
  Base62(7 chars): "d41d8cd"

  ┌──────────┐  MD5     ┌───────────┐  Base62   ┌──────────┐
  │ long_url │────────▶│   Hash    │──────────▶│ Substring│
  └──────────┘         │  Engine   │  Encode   │  "d41d8cd"│
                       └───────────┘          └──────────┘

  Pros: Same URL → same short URL (dedup)
  Cons: Collision possible, need to handle
```

### Approach 3: Pre-Generated Key Service / 预生成 Key 服务

```
┌─────────────────────────────────────────────────────────┐
│                Key Generation Service                     │
│                                                         │
│  ┌─────────┐    ┌─────────────┐    ┌───────────────┐  │
│  │ Key     │───▶│ Key Store   │───▶│  API Server   │  │
│  │ Generator│    │ (Available  │    │ (Fills local  │  │
│  │ (Batch)  │    │  keys pool) │    │  key cache)   │  │
│  └─────────┘    └─────────────┘    └───────────────┘  │
│                                                         │
│  1. Generator creates random 7-char Base62 keys        │
│  2. Stores in "available keys" table                   │
│  3. API servers fetch keys in batch (1000 at a time)   │
│  4. On URL creation, pop key from local cache          │
│  5. No collision check needed!                         │
└─────────────────────────────────────────────────────────┘
```

---

## Detailed Design

### Read Path (Redirect) / 读路径

```
Client ──GET /abc123──▶ Load Balancer ──▶ API Server
                                              │
                              ┌───────────────┤
                              ▼               ▼
                        ┌──────────┐    ┌──────────┐
                        │  Redis   │    │  Check   │
                        │  Cache   │    │  Cache   │
                        │          │    │  First   │
                        │ HIT: return URL│         │
                        │ MISS: query DB │         │
                        └──────────┘    └──────────┘
                                              │
                                              ▼
                                        ┌──────────┐
                                        │PostgreSQL│
                                        │          │
                                        │ Query by │
                                        │short_code│
                                        └──────────┘
                                              │
                                              ▼
                                    301 Redirect → long_url
```

### Write Path (Create) / 写路径

```
Client ──POST /api/urls──▶ API Server
                              │
                              ▼
                         ┌─────────┐
                         │ Validate │
                         │ URL      │
                         │ Format   │
                         └────┬────┘
                              │
                              ▼
                         ┌─────────┐
                         │ Generate │
                         │ Short    │
                         │ Code     │
                         └────┬────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ Check for        │
                    │ Custom Alias?    │
                    └────┬────────┬────┘
                    Yes  │        │ No
                         ▼        ▼
                    ┌─────────┐  ┌─────────┐
                    │ Check   │  │ Use     │
                    │ Alias   │  │ Generated│
                    │ Unique? │  │ Code     │
                    └─────────┘  └─────────┘
                         │
                         ▼
                    ┌─────────┐
                    │ Insert  │
                    │ into DB │
                    └─────────┘
                         │
                         ▼
                    Return short URL
```

### Analytics Path / 分析路径

```
Client ──GET /abc123──▶ API Server
                              │
                   ┌──────────┤
                   │          │
                   ▼          ▼
            301 Redirect   Async Analytics
                              │
                              ▼
                    ┌──────────────┐
                    │  Kafka       │
                    │  Topic:      │
                    │  click_events│
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │Click     │ │Real-time │ │Aggregate │
        │Count     │ │Dashboard │ │Stats     │
        │Updater   │ │(WebSocket│ │(Batch)   │
        └──────────┘ └──────────┘ └──────────┘
              │
              ▼
        ┌──────────┐
        │ ClickHouse│
        │ (Analytics│
        │  DB)      │
        └──────────┘
```

---

## Scaling Strategy

### Database Scaling / 数据库扩展

```
Read Replicas:
  ┌──────────┐
  │ Primary  │──── Writes
  │ (Postgres)│
  └────┬─────┘
       │ Replication
  ┌────┼────┬────────┐
  ▼    ▼    ▼        ▼
┌────┐┌────┐┌────┐┌────┐
│ RR1││ RR2││ RR3││ RR4│  ← Read replicas
└────┘└────┘└────┘└────┘

Sharding by short_code hash:
  shard_id = hash(short_code) % N
```

### Cache Strategy / 缓存策略

```
Redis Cache (80/20 rule - 20% of URLs get 80% of traffic):

  Cache-Aside Pattern:
  1. Check Redis for short_code
  2. Cache HIT → return URL (1ms)
  3. Cache MISS → query PostgreSQL (10ms)
  4. Store in Redis with TTL (24 hours)

  Cache Size:
    20% of 6B URLs = 1.2B entries
    1.2B × 530 bytes ≈ 636 GB
    → Redis Cluster with ~10 nodes (64GB each)
```

### Rate Limiting / 限流

```
┌─────────────────────────────────────────┐
│           Rate Limiting                  │
│                                         │
│  Create URL: 10/hour per user           │
│  Read URL: 1000/hour per IP             │
│  Custom alias: 5/day per free user      │
│                                         │
│  Implementation: Redis sliding window   │
│  Key: rate:{user_id}:{window}           │
│  INCR + EXPIRE                          │
└─────────────────────────────────────────┘
```

---

## Trade-offs

| Decision | Option A | Option B | Chosen |
|----------|----------|----------|--------|
| Redirect Code | 301 (cached) | 302 (server sees) | 302 for analytics |
| ID Generation | Hash | Counter + Base62 | Pre-generated keys |
| Analytics | Real-time | Batch | Hybrid (Kafka + ClickHouse) |
| DB Choice | PostgreSQL | DynamoDB | PostgreSQL (strong consistency) |
| Cache | Redis | Memcached | Redis (rich data structures) |
| Short URL Length | 6 chars (56B combos) | 7 chars (3.5T combos) | 7 (future-proof) |

---

## 中文版本

### 需求分析

**功能需求**：给定长 URL 生成短 URL；给定短 URL 重定向到原始 URL；可选自定义别名和链接过期；点击统计分析。

**非功能需求**：高可用（99.99%）、低延迟（<50ms）、短链不可预测、高读写比（100:1）。

### 容量估算

- 写 QPS：~40/秒（峰值 120）
- 读 QPS：~4,000/秒（峰值 12,000）
- 5 年存储：~5 TB（60 亿条 URL）
- 读带宽：~2.1 MB/秒

### 高层设计

客户端 → 负载均衡 → API 服务器（无状态）→ Redis 缓存 + PostgreSQL（URL 存储）+ Kafka（分析事件）→ ClickHouse（分析数据库）。

### 短链生成方案

1. **Base62 编码**：分布式计数器 + Base62 转换。保证唯一但可预测。
2. **哈希取子串**：MD5(long_url) 取前 7 位。相同 URL 产生相同短链，但有碰撞风险。
3. **预生成 Key 服务**（推荐）：批量生成随机 7 位 Base62 Key，API 服务器从本地缓存取用。无需碰撞检测。

7 位 Base62 可生成 3.5 万亿种组合，足够使用。

### 数据库 Schema

URL 表：short_code（唯一索引）、long_url、user_id、created_at、expires_at、click_count。

点击事件表：short_code、clicked_at、ip_address、referrer、country。按时间分区。

### 读路径（重定向）

1. 检查 Redis 缓存（~1ms）
2. 缓存未命中则查 PostgreSQL（~10ms）
3. 异步发送点击事件到 Kafka
4. 返回 302 重定向

### 扩展策略

- **数据库**：读写分离（主库写，从库读）+ 按 short_code 哈希分片
- **缓存**：80/20 法则，20% 的热门 URL 缓存在 Redis 集群（~636GB，10 节点）
- **限流**：创建 URL 10 次/小时/用户，读取 1000 次/小时/IP
- **分析**：Kafka + ClickHouse 混合方案，支持实时和批量分析

### 权衡决策

| 决策 | 选项 A | 选项 B | 选择 |
|------|--------|--------|------|
| 重定向码 | 301（浏览器缓存） | 302（服务器可见） | 302（需要分析） |
| ID 生成 | 哈希 | 计数器 + Base62 | 预生成 Key |
| 分析 | 实时 | 批量 | 混合方案 |
| 数据库 | PostgreSQL | DynamoDB | PostgreSQL |
| 缓存 | Redis | Memcached | Redis |
