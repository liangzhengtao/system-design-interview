# Design a News Feed System

# 设计信息流系统

Design a news feed like Twitter or Facebook Timeline.

设计一个类似 Twitter 或 Facebook 的信息流系统。

---

## Table of Contents / 目录

- [Requirements / 需求分析](#requirements)
- [Capacity Estimation / 容量估算](#capacity-estimation)
- [High-Level Design / 高层设计](#high-level-design)
- [API Design / API 设计](#api-design)
- [Database Schema / 数据库 Schema](#database-schema)
- [Fan-out Strategy / 扇出策略](#fan-out-strategy)
- [Feed Ranking / 信息流排序](#feed-ranking)
- [Media Handling / 媒体处理](#media-handling)
- [Scaling Strategy / 扩展策略](#scaling-strategy)
- [Trade-offs / 权衡分析](#trade-offs)
- [中文版本](#中文版本)

---

## Requirements

### Functional Requirements / 功能需求

1. Users can post tweets (280 chars, images, videos)
2. Users can follow other users
3. Home feed shows posts from followed users
4. Users can like, retweet, reply
5. Trending topics and hashtags
6. Search tweets

### Non-Functional Requirements / 非功能需求

1. Feed generation < 300ms
2. New posts appear in feed within 5 seconds
3. 99.99% availability
4. Read-heavy (100:1 read:write ratio)
5. Handle celebrity accounts (millions of followers)

---

## Capacity Estimation

### Traffic Estimates / 流量估算

```
Assumptions:
  - 500M total users
  - 200M DAU
  - Each user creates 2 posts/day on average
  - Each user views feed 10 times/day
  - Average user follows 200 accounts

Write QPS:
  200M × 2 / 86400 ≈ 4,600 posts/sec

Read QPS:
  200M × 10 / 86400 ≈ 23,000 feed loads/sec

Peak (3x):
  Write: ~14,000/sec
  Read: ~70,000/sec
```

### Storage Estimates / 存储估算

```
Per post:
  - post_id:       8 bytes (Snowflake)
  - user_id:       8 bytes
  - content:     280 bytes (max text)
  - media_urls:  200 bytes (average)
  - metadata:     50 bytes (likes, retweets count)
  - created_at:    8 bytes
  - Total:      ~554 bytes

Daily storage:
  200M × 2 × 554 bytes = 222 GB/day

5 years:
  222 GB × 365 × 5 = 405 TB
  With replicas: ~1.2 PB
```

### Fan-out Estimates / 扇出估算

```
Average user has 200 followers:
  4,600 posts/sec × 200 followers = 920,000 feed writes/sec

Celebrity with 50M followers:
  1 post × 50M followers = 50M feed writes!
  → This is the "fan-out on write" problem

  Solution: Hybrid approach (see Fan-out Strategy)
```

---

## High-Level Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                    News Feed Architecture                            │
│                                                                     │
│  ┌──────────┐                                                      │
│  │  Client  │                                                      │
│  └────┬─────┘                                                      │
│       │                                                             │
│       ▼                                                             │
│  ┌──────────┐      ┌──────────────────────────────────┐           │
│  │   API    │      │         Internal Services          │           │
│  │ Gateway  │      │                                  │           │
│  └──┬───┬───┘      │  ┌──────────┐  ┌──────────┐    │           │
│     │   │          │  │  Post    │  │  Feed    │    │           │
│     │   │          │  │ Service  │  │ Service  │    │           │
│     │   │          │  └────┬─────┘  └────┬─────┘    │           │
│     │   │          │       │              │          │           │
│     │   │          │  ┌────▼─────┐  ┌────▼─────┐    │           │
│     │   │          │  │ Fan-out  │  │  Feed    │    │           │
│     │   │          │  │ Service  │  │  Ranker  │    │           │
│     │   │          │  └────┬─────┘  └──────────┘    │           │
│     │   │          │       │                         │           │
│     │   │          └───────┼─────────────────────────┘           │
│     │   │                  │                                      │
│     │   │          ┌───────┼───────┬───────┐                    │
│     │   │          │       │       │       │                     │
│     │   │     ┌────▼──┐┌───▼──┐┌───▼──┐┌───▼──┐               │
│     │   │     │Post DB││Feed  ││User  ││Social│               │
│     │   │     │(Tweet)││Cache ││Graph ││Graph │               │
│     │   │     └───────┘└──────┘└──────┘└──────┘               │
│     │   │                                                       │
│     │   └─────────────────────────────────────────┐            │
│     │                                             │             │
│     ▼                                             ▼             │
│  GET /feed                    POST /posts                       │
│  (Read path)                  (Write path)                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## API Design

### Create Post / 创建帖子

```http
POST /api/posts
Authorization: Bearer {token}

{
  "content": "Excited about system design! #engineering",
  "media": [
    {"type": "image", "url": "https://cdn.example.com/img1.jpg"}
  ],
  "reply_to": null,
  "visibility": "public"
}

Response (201 Created):
{
  "post_id": "1234567890",
  "user_id": "user_123",
  "content": "Excited about system design! #engineering",
  "created_at": "2025-01-15T10:30:00Z",
  "likes": 0,
  "retweets": 0,
  "replies": 0
}
```

### Get Feed / 获取信息流

```http
GET /api/feed?cursor={last_post_id}&limit=20
Authorization: Bearer {token}

Response (200 OK):
{
  "posts": [
    {
      "post_id": "1234567891",
      "user": {
        "user_id": "user_456",
        "username": "john_doe",
        "avatar": "https://cdn.example.com/avatar.jpg",
        "verified": true
      },
      "content": "Great day for coding!",
      "media": [],
      "likes": 150,
      "retweets": 30,
      "replies": 10,
      "liked_by_me": false,
      "created_at": "2025-01-15T10:25:00Z"
    }
  ],
  "next_cursor": "1234567870",
  "has_more": true
}
```

### Like / Retweet / 点赞/转推

```http
POST /api/posts/{post_id}/like
POST /api/posts/{post_id}/retweet
DELETE /api/posts/{post_id}/like
```

### Follow / 关注

```http
POST /api/users/{user_id}/follow
DELETE /api/users/{user_id}/follow
GET /api/users/{user_id}/followers
GET /api/users/{user_id}/following
```

---

## Database Schema

### Posts Table / 帖子表

```sql
CREATE TABLE posts (
    post_id       BIGINT PRIMARY KEY,      -- Snowflake ID
    user_id       BIGINT NOT NULL,
    content       VARCHAR(280),
    media_urls    JSON,                     -- [{type, url, thumbnail}]
    reply_to_id   BIGINT,                   -- NULL for original posts
    retweet_of_id BIGINT,                   -- NULL for original posts
    visibility    ENUM('public','followers','private'),
    like_count    INT DEFAULT 0,
    retweet_count INT DEFAULT 0,
    reply_count   INT DEFAULT 0,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user_created (user_id, created_at DESC),
    INDEX idx_reply_to (reply_to_id),
    FULLTEXT INDEX idx_content (content)
) PARTITION BY RANGE (created_at);
```

### User Follows Table / 用户关注表

```sql
CREATE TABLE user_follows (
    follower_id   BIGINT NOT NULL,
    following_id  BIGINT NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (follower_id, following_id),
    INDEX idx_following (following_id, follower_id)
);
```

### Feed Cache Table / 信息流缓存

```sql
-- Redis Sorted Set
-- Key: feed:{user_id}
-- Score: post_id (Snowflake = timestamp ordered)
-- Value: post_id

-- Example:
-- ZADD feed:user_123 1234567890 "1234567890"
-- ZREVRANGE feed:user_123 0 19  (get latest 20)
```

### Social Graph / 社交图谱

```sql
-- Redis Set for fast follower/following lookups
-- Key: following:{user_id} → Set of user_ids
-- Key: followers:{user_id} → Set of user_ids

-- SMEMBERS following:user_123
-- SISMEMBER following:user_123 user_456
```

---

## Fan-out Strategy

### Fan-out on Write (Push Model) / 写时扇出（推模式）

```
User A (50 followers) posts:

  1. Store post in Posts DB
  2. Get all followers of A: [B, C, D, E, F, ...]
  3. For each follower, add post_id to their feed cache:

     ┌────────┐  post   ┌──────────┐  fan-out  ┌───────────────┐
     │User A  │────────▶│Post Store│──────────▶│ Feed Cache    │
     └────────┘         └──────────┘           │               │
                                                │ feed:B += post│
                                                │ feed:C += post│
                                                │ feed:D += post│
                                                │ feed:E += post│
                                                └───────────────┘

  Pros: Feed reads are fast (pre-computed)
  Cons: Write amplification, celebrity problem
```

### Fan-out on Read (Pull Model) / 读时扇出（拉模式）

```
User B opens feed:

  1. Get list of users B follows: [A, C, D, E, F]
  2. Fetch latest posts from each followed user
  3. Merge and rank posts
  4. Return to client

  ┌────────┐  get feed  ┌──────────┐  fetch    ┌──────────┐
  │User B  │───────────▶│Feed Svc  │─────────▶│Post Store│
  │        │◀───────────│          │◀─────────│          │
  └────────┘  ranked    │ Merge &  │  posts   └──────────┘
                feed    │ Rank     │
                        └──────────┘

  Pros: No write amplification, no celebrity problem
  Cons: Slow reads (compute on demand)
```

### Hybrid Approach (Twitter's Solution) / 混合方案

```
┌─────────────────────────────────────────────────────────┐
│              Hybrid Fan-out Strategy                     │
│                                                         │
│  User posts:                                            │
│    ├── Normal user (< 10K followers):                   │
│    │   → Fan-out on write (push to all followers)       │
│    │                                                     │
│    └── Celebrity (≥ 10K followers):                     │
│        → Do NOT fan-out on write                        │
│        → Followers pull celebrity posts on read         │
│                                                         │
│  Feed generation:                                       │
│    1. Get pre-computed feed from cache (fan-out writes) │
│    2. Get latest posts from followed celebrities        │
│    3. Merge and rank                                    │
│    4. Return top N posts                                │
│                                                         │
│  ┌─────────┐  pre-computed  ┌─────────┐               │
│  │  Feed   │◀──────────────│Fan-out  │               │
│  │  Cache  │                │ Service │               │
│  └────┬────┘                └─────────┘               │
│       │                                                 │
│       │ merge                                           │
│       ▼                                                 │
│  ┌─────────┐  celebrity    ┌───────────┐               │
│  │  Feed   │◀──── posts ──│ Celebrity │               │
│  │ Ranker  │               │ Posts     │               │
│  └─────────┘               └───────────┘               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Feed Ranking

### Ranking Signals / 排序信号

```
┌─────────────────────────────────────────────────────────┐
│                  Feed Ranking Model                      │
│                                                         │
│  Score = w1 × recency                                   │
│        + w2 × engagement (likes, retweets, replies)     │
│        + w3 × relationship (interaction frequency)      │
│        + w4 × content_type (media boost)                │
│        + w5 × creator_authority (verified, quality)     │
│        - w6 × already_seen                              │
│        - w7 × muted_keywords                            │
│                                                         │
│  Weights (w1-w7) learned from ML model                 │
│  A/B tested continuously                                │
└─────────────────────────────────────────────────────────┘
```

### Chronological vs Ranked / 时间序 vs 排序

```
Chronological (Twitter pre-2016):
  ┌──┬──┬──┬──┬──┬──┬──┬──┐
  │T1│T2│T3│T4│T5│T6│T7│T8│  ← Pure time order
  └──┴──┴──┴──┴──┴──┴──┴──┘
  Simple, transparent
  But: may miss important posts

Ranked (Facebook, Twitter algorithmic):
  ┌──┬──┬──┬──┬──┬──┬──┬──┐
  │T3│T1│T7│T2│T5│T4│T8│T6│  ← Engagement + recency
  └──┴──┴──┴──┴──┴──┴──┴──┘
  Better engagement, personalized
  But: filter bubble, less transparent
```

---

## Media Handling

### Image Upload Pipeline / 图片上传管道

```
┌────────┐  upload   ┌──────────┐  process  ┌───────────┐
│ Client │──────────▶│  Upload  │─────────▶│  Image    │
│        │           │  Service │          │ Processor │
└────────┘           └──────────┘          └─────┬─────┘
                                                  │
                              ┌────────────────────┤
                              │                    │
                              ▼                    ▼
                       ┌───────────┐       ┌───────────┐
                       │ Thumbnail │       │ Full Size │
                       │ 150x150   │       │ 1920x1080 │
                       └─────┬─────┘       └─────┬─────┘
                             │                   │
                             ▼                   ▼
                       ┌─────────────────────────────┐
                       │        S3 + CDN              │
                       │                             │
                       │  /img/{id}/thumb.jpg        │
                       │  /img/{id}/full.jpg         │
                       │  /img/{id}/optimized.webp   │
                       └─────────────────────────────┘
```

---

## Scaling Strategy

### Caching Strategy / 缓存策略

```
Multi-layer caching:

  L1: Client-side cache (app memory, 5 min)
      └─ Avoid redundant API calls

  L2: CDN cache (static assets, media, 24 hr)
      └─ Serves images/videos from edge

  L3: API Gateway cache (hot feeds, 30 sec)
      └─ Caches popular user feeds

  L4: Redis cluster (feed cache, user graph)
      └─ Feed:{user_id} sorted set
      └─ following:{user_id} set

  L5: Database query cache
      └─ PostgreSQL buffer pool

Cache hit rates:
  L1: ~30%  (client refetch within window)
  L2: ~60%  (media CDN)
  L3: ~40%  (API level)
  L4: ~95%  (Redis feed cache)
```

### Database Sharding / 数据库分片

```
Posts DB: Shard by user_id
  shard = hash(user_id) % N

  Benefits:
  - All user's posts on same shard
  - User profile queries are single-shard

  Feed queries span multiple shards:
  → Use pre-computed feed cache (Redis)
  → Only hit DB for cold/expired feeds
```

### Real-time Updates / 实时更新

```
┌─────────────────────────────────────────────────┐
│          Real-time Feed Updates                  │
│                                                 │
│  Client ──WebSocket──▶ Update Service           │
│                           │                     │
│                    ┌──────┼──────┐              │
│                    │      │      │              │
│                    ▼      ▼      ▼              │
│               New Post  Like  Retweet           │
│                    │      │      │              │
│                    ▼      ▼      ▼              │
│              Push to online followers            │
│              via WebSocket                       │
│                                                 │
│  Latency: < 5 seconds for online users          │
│  Offline users: fetch on next app open          │
└─────────────────────────────────────────────────┘
```

---

## Trade-offs

| Decision | Option A | Option B | Chosen |
|----------|----------|----------|--------|
| Fan-out | Write (push) | Read (pull) | Hybrid |
| Feed order | Chronological | Ranked | Ranked (engagement) |
| Post DB | PostgreSQL | Cassandra | Cassandra (write-heavy) |
| Feed cache | Redis | Memcached | Redis (sorted sets) |
| Real-time | Polling | WebSocket | WebSocket |
| Media | Sync upload | Async processing | Async (S3 + Lambda) |
| Celebrity threshold | 1K followers | 10K followers | 10K |

---

## 中文版本

### 需求分析

**功能需求**：发布帖子（280 字、图片、视频）、关注/取关、首页信息流、点赞/转推/回复、热门话题、搜索。

**非功能需求**：信息流生成 <300ms、新帖 5 秒内出现在 Feed、99.99% 可用性、读写比 100:1。

### 容量估算

- 写 QPS：~4,600 帖子/秒（峰值 14,000）
- 读 QPS：~23,000 次 Feed 加载/秒（峰值 70,000）
- 每日存储：222 GB
- 5 年存储：~1.2 PB（含副本）
- 普通用户扇出：4,600 × 200 粉丝 = 92 万次 Feed 写入/秒
- 名人扇出问题：1 帖 × 5000 万粉丝 = 5000 万次写入！

### 高层设计

客户端 → API 网关 → 帖子服务（写入）+ 信息流服务（读取）→ 扇出服务 + 排序器 → 帖子数据库（Cassandra）+ Feed 缓存（Redis）+ 社交图谱（Redis）+ 媒体存储（S3 + CDN）。

### 扇出策略

**写时扇出（推模式）**：用户发帖时，立即将帖子 ID 推送到所有粉丝的 Feed 缓存。读取速度快，但名人发帖会产生巨大写入量。

**读时扇出（拉模式）**：用户打开 Feed 时，实时拉取关注用户的最新帖子并合并排序。无写放大，但读取慢。

**混合方案（Twitter 方案）**：
- 普通用户（<1 万粉丝）：写时扇出
- 名人用户（≥1 万粉丝）：不扇出，粉丝读取时实时拉取
- Feed 生成 = 预计算缓存 + 实时名人帖子 → 合并排序

### 信息流排序

排序信号：时效性、互动量（点赞/转推/回复）、关系亲密度、内容类型（媒体加分）、创作者权威度、已看过惩罚。权重通过 ML 模型学习，持续 A/B 测试。

### 媒体处理

图片上传 → 异步处理（缩略图 150x150、全尺寸 1920x1080、WebP 优化）→ S3 存储 + CDN 分发。

### 扩展策略

- **缓存**：5 层缓存（客户端 → CDN → API 网关 → Redis → 数据库缓冲池）
- **数据库分片**：按 user_id 哈希分片，用户帖子在同一分片
- **实时更新**：WebSocket 推送新帖/点赞/转推，延迟 <5 秒

### 权衡决策

| 决策 | 选择 | 原因 |
|------|------|------|
| 扇出策略 | 混合方案 | 平衡普通用户和名人 |
| Feed 排序 | 算法排序 | 提高用户参与度 |
| 帖子数据库 | Cassandra | 写入密集、时序数据 |
| Feed 缓存 | Redis | Sorted Set 天然支持 |
| 实时更新 | WebSocket | 低延迟推送 |
| 名人阈值 | 1 万粉丝 | 平衡写入量和读取速度 |
