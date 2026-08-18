# Design a Search Engine

# 设计搜索引擎

Design a web search engine like Google or Bing.

设计一个类似 Google 或 Bing 的网络搜索引擎。

---

## Table of Contents / 目录

- [Requirements / 需求分析](#requirements)
- [Capacity Estimation / 容量估算](#capacity-estimation)
- [High-Level Design / 高层设计](#high-level-design)
- [API Design / API 设计](#api-design)
- [Web Crawler / 网络爬虫](#web-crawler)
- [Indexer / 索引器](#indexer)
- [Query Processing / 查询处理](#query-processing)
- [Ranking Algorithm / 排序算法](#ranking-algorithm)
- [Database Schema / 数据库 Schema](#database-schema)
- [Scaling Strategy / 扩展策略](#scaling-strategy)
- [Trade-offs / 权衡分析](#trade-offs)
- [中文版本](#中文版本)

---

## Requirements

### Functional Requirements / 功能需求

1. Crawl and index billions of web pages
2. Full-text search with relevance ranking
3. Support Boolean operators (AND, OR, NOT)
4. Auto-complete / suggestions
5. Spell correction
6. Image and video search
7. Search filters (date, language, region)

### Non-Functional Requirements / 非功能需求

1. Search latency < 500ms (p99)
2. Index freshness: new pages searchable within 24 hours
3. 99.99% availability
4. Handle 100K queries/second
5. Support 100B+ indexed pages

---

## Capacity Estimation

### Traffic Estimates / 流量估算

```
Assumptions:
  - 100B pages indexed
  - 100K QPS (peak 300K)
  - Each page average 10KB
  - Re-index every 30 days

Crawl rate:
  100B pages / 30 days / 86400 sec ≈ 38,500 pages/sec

Query rate:
  100,000 queries/sec (peak 300,000)
```

### Storage Estimates / 存储估算

```
Document Store:
  100B pages × 10KB = 1 PB (raw pages)
  With compression (5x): ~200 TB

Inverted Index:
  ~30% of document store = 60 TB
  With compression: ~20 TB

Forward Index:
  ~20% of document store = 40 TB
  With compression: ~10 TB

Total Storage:
  ~230 TB (compressed)
  With replicas (3x): ~690 TB

Page Metadata:
  100B pages × 500 bytes = 50 TB
```

### Memory Estimates / 内存估算

```
Hot Index (in memory):
  Top 10% most-accessed terms
  ~2 TB across cluster

Query Cache:
  Popular queries cached
  ~100 GB Redis cluster
```

---

## High-Level Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Search Engine Architecture                            │
│                                                                         │
│  ┌──────────┐                                                          │
│  │  User    │                                                          │
│  └────┬─────┘                                                          │
│       │                                                                 │
│       ▼                                                                 │
│  ┌──────────┐     ┌──────────────────────────────────────────────┐    │
│  │   API    │     │              Offline Pipeline                  │    │
│  │ Gateway  │     │                                              │    │
│  └────┬─────┘     │  ┌──────────┐  ┌──────────┐  ┌──────────┐ │    │
│       │           │  │  Web     │  │ Content  │  │ Inverted │ │    │
│       ▼           │  │ Crawler  │─▶│ Parser   │─▶│ Indexer  │ │    │
│  ┌──────────┐     │  └──────────┘  └──────────┘  └────┬─────┘ │    │
│  │  Query   │     │                                    │        │    │
│  │ Processor│     │                               ┌────▼─────┐ │    │
│  │          │     │                               │ Index    │ │    │
│  │- Parse   │     │                               │ Store    │ │    │
│  │- Correct │     │                               │ (Elastic │ │    │
│  │- Expand  │     │                               │  search) │ │    │
│  └────┬─────┘     │                               └──────────┘ │    │
│       │           └──────────────────────────────────────────────┘    │
│       ▼                                                                 │
│  ┌──────────┐                                                          │
│  │  Ranker  │                                                          │
│  │          │                                                          │
│  │- TF-IDF  │                                                          │
│  │- BM25    │                                                          │
│  │- PageRank│                                                          │
│  │- ML Rank │                                                          │
│  └────┬─────┘                                                          │
│       │                                                                 │
│       ▼                                                                 │
│  ┌──────────┐                                                          │
│  │  Result  │                                                          │
│  │ Assembler│                                                          │
│  │          │                                                          │
│  │- Snippets│                                                          │
│  │- Filters │                                                          │
│  │- Ads     │                                                          │
│  └──────────┘                                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## API Design

### Search Query / 搜索查询

```http
GET /api/search?q={query}&page=1&limit=10&lang=en&region=us&date_range=month

Response (200 OK):
{
  "query": "system design interview",
  "total_results": 1250000,
  "page": 1,
  "results": [
    {
      "title": "System Design Interview Guide",
      "url": "https://example.com/system-design",
      "snippet": "Complete guide to acing your <b>system design interview</b>...",
      "display_url": "example.com › system-design",
      "cached_url": "https://cache.example.com/...",
      "last_crawled": "2025-01-14T08:00:00Z",
      "page_rank": 0.95,
      "relevance_score": 0.92
    }
  ],
  "suggestions": [
    "system design interview questions",
    "system design interview preparation"
  ],
  "related_searches": [
    "distributed systems interview",
    "scalability interview questions"
  ],
  "search_time_ms": 45
}
```

### Auto-complete / 自动补全

```http
GET /api/autocomplete?q={prefix}&limit=5

Response (200 OK):
{
  "suggestions": [
    {"text": "system design", "score": 0.95},
    {"text": "system design interview", "score": 0.90},
    {"text": "system design primer", "score": 0.85}
  ]
}
```

### Index Page / 索引页面

```http
POST /api/index
Content-Type: application/json
Authorization: Internal

{
  "url": "https://example.com/page",
  "content": "...",
  "title": "Page Title",
  "links_out": ["https://other.com/page1"],
  "last_modified": "2025-01-15T10:00:00Z"
}
```

---

## Web Crawler

### Crawler Architecture / 爬虫架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Web Crawler Pipeline                           │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐ │
│  │  Seed    │───▶│   URL    │───▶│  Fetcher │───▶│  Parser  │ │
│  │  URLs    │    │ Frontier │    │          │    │          │ │
│  └──────────┘    │(Priority │    │  HTTP    │    │ Extract  │ │
│                  │  Queue)  │    │  Client  │    │  Links   │ │
│                  └────┬─────┘    └──────────┘    │  Content │ │
│                       │                          └────┬─────┘ │
│                       │                               │       │
│                       │    ┌──────────────────────────┘       │
│                       │    │                                   │
│                       ▼    ▼                                   │
│                  ┌──────────────┐                              │
│                  │  URL Filter  │                              │
│                  │              │                              │
│                  │ - Dedup      │                              │
│                  │ - Robots.txt │                              │
│                  │ - Politeness │                              │
│                  │ - Priority   │                              │
│                  └──────┬───────┘                              │
│                         │                                      │
│                         ▼                                      │
│                  ┌──────────────┐                              │
│                  │  URL Store   │                              │
│                  │  (Visited)   │                              │
│                  └──────────────┘                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### URL Frontier / URL 前沿队列

```
Priority Queue Implementation:

  ┌─────────────────────────────────────────┐
  │           URL Frontier                   │
  │                                         │
  │  High Priority Queue:                   │
  │  ┌─────────────────────────────┐       │
  │  │ news sites, frequently       │       │
  │  │ updated pages                │       │
  │  └─────────────────────────────┘       │
  │                                         │
  │  Medium Priority Queue:                 │
  │  ┌─────────────────────────────┐       │
  │  │ Regular pages, blogs         │       │
  │  └─────────────────────────────┘       │
  │                                         │
  │  Low Priority Queue:                    │
  │  ┌─────────────────────────────┐       │
  │  │ Rarely changing pages        │       │
  │  └─────────────────────────────┘       │
  │                                         │
  │  Politeness: Max 1 req/sec per domain  │
  └─────────────────────────────────────────┘

  BFS (Breadth-First): Good for discovery
  DFS (Depth-First): Good for topic crawling
  Priority-based: Best for freshness
```

### Crawler Politeness / 爬虫礼貌性

```
Rules:
  1. Respect robots.txt
  2. Max 1 request per second per domain
  3. Exponential backoff on errors
  4. Identify as bot in User-Agent

  ┌──────────┐  robots.txt  ┌──────────┐
  │ Crawler  │─────────────▶│  Domain  │
  │          │◀─────────────│          │
  │          │  parse rules │          │
  │          │              │          │
  │          │  crawl       │          │
  │          │─────────────▶│          │
  │          │◀─────────────│          │
  │          │  200 OK      │          │
  └──────────┘              └──────────┘

  Rate limiting per domain using token bucket
```

---

## Indexer

### Inverted Index / 倒排索引

```
Document Collection:
  Doc 1: "the cat sat on the mat"
  Doc 2: "the dog sat on the log"
  Doc 3: "cats and dogs are friends"

Inverted Index:
  ┌─────────┬──────────────────┐
  │  Term   │  Posting List    │
  ├─────────┼──────────────────┤
  │  the    │ [1, 2]           │
  │  cat    │ [1]              │
  │  cats   │ [3]              │
  │  sat    │ [1, 2]           │
  │  on     │ [1, 2]           │
  │  mat    │ [1]              │
  │  dog    │ [2]              │
  │  dogs   │ [3]              │
  │  log    │ [2]              │
  │  and    │ [3]              │
  │  are    │ [3]              │
  │  friends│ [3]              │
  └─────────┴──────────────────┘

Query "cat AND sat":
  → intersect([1], [1,2]) = [1]

Query "cat OR dog":
  → union([1], [2]) = [1, 2]
```

### Posting List Structure / 倒排列表结构

```
For term "system":

┌──────────────────────────────────────────────────────────┐
│  Posting List for "system"                                │
│                                                          │
│  ┌─────────┬──────────┬─────────┬──────────┬──────────┐ │
│  │ Doc ID  │ TF       │ Positions│ Metadata │          │ │
│  ├─────────┼──────────┼─────────┼──────────┼──────────┤ │
│  │ doc_123 │ 5        │[0,12,45]│ title,URL│          │ │
│  │ doc_456 │ 3        │[7,23]   │ title,URL│          │ │
│  │ doc_789 │ 8        │[1,5,19] │ title,URL│          │ │
│  └─────────┴──────────┴─────────┴──────────┴──────────┘ │
│                                                          │
│  Compressed using:                                       │
│  - Variable-byte encoding for doc IDs                    │
│  - Delta encoding for sorted doc IDs                     │
│  - Frame-of-reference for positions                      │
└──────────────────────────────────────────────────────────┘
```

### Index Construction / 索引构建

```
MapReduce Pipeline:

  Map Phase:
  ┌──────────┐                    ┌──────────┐
  │ Doc 1    │───▶ Map ──────────▶│ term1:   │
  │          │    (tokenize,      │ (doc1,tf)│
  └──────────┘     stem)          │ term2:   │
                                  │ (doc1,tf)│
  ┌──────────┐                    └──────────┘
  │ Doc 2    │───▶ Map ──────────▶│ term1:   │
  │          │                    │ (doc2,tf)│
  └──────────┘                    │ term3:   │
                                  │ (doc2,tf)│
                                  └──────────┘

  Reduce Phase:
  ┌──────────────────────────────────────────┐
  │ term1: [(doc1,tf), (doc2,tf), ...]      │
  │ term2: [(doc1,tf), ...]                  │
  │ term3: [(doc2,tf), ...]                  │
  └──────────────────────────────────────────┘

  Output: Inverted index shards
```

---

## Query Processing

### Query Pipeline / 查询管道

```
User Query: "best system design interview tips"
         │
         ▼
  ┌──────────────┐
  │ 1. Tokenize  │  → ["best", "system", "design", "interview", "tips"]
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ 2. Normalize │  → lowercase, remove accents
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ 3. Stem/Lem  │  → ["best", "system", "design", "interview", "tip"]
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ 4. Stop Words│  → ["system", "design", "interview", "tip"]
  │    Removal   │    (removed: "best" as stop word)
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ 5. Spell     │  → "interview" (correct)
  │    Check     │
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ 6. Query     │  → expand with synonyms
  │    Expansion │  → "system" OR "systems"
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ 7. Index     │  → retrieve posting lists
  │    Lookup    │  → intersect for AND
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ 8. Rank      │  → BM25 + PageRank + ML
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ 9. Assemble  │  → snippets, thumbnails
  │    Results   │  → pagination
  └──────────────┘
```

---

## Ranking Algorithm

### TF-IDF / 词频-逆文档频率

```
TF(t,d) = (count of term t in doc d) / (total terms in doc d)
IDF(t) = log(N / df(t))  where N = total docs, df = docs containing t
Score = TF × IDF

Example:
  Query: "system design"
  Doc 1: "system design interview" (3 words)
    TF("system") = 1/3
    IDF("system") = log(100B / 10B) = 1
    Score = 1/3 × 1 = 0.33
```

### BM25 / BM25 算法

```
BM25(q,d) = Σ IDF(qi) × [f(qi,d) × (k1+1)] / [f(qi,d) + k1 × (1 - b + b × |d|/avgdl)]

Parameters:
  k1 = 1.2 (term frequency saturation)
  b = 0.75 (length normalization)
  avgdl = average document length

BM25 improves on TF-IDF:
  - Term frequency saturation (diminishing returns)
  - Document length normalization
  - Better handling of long documents
```

### PageRank / PageRank 算法

```
PageRank(A) = (1-d) + d × Σ [PageRank(T) / OutLinks(T)]
  where d = 0.85 (damping factor)
  T = pages that link to A

Iterative computation:

  Iteration 0: All pages = 1/N
  Iteration 1: Recalculate based on links
  ...
  Iteration ~50: Converges

  ┌──────┐  links to   ┌──────┐
  │Page A│─────────────▶│Page B│
  │PR=0.5│              │PR=0.3│
  └──┬───┘              └──┬───┘
     │                     │
     │ links to            │ links to
     ▼                     ▼
  ┌──────┐              ┌──────┐
  │Page C│◀─────────────│Page D│
  │PR=0.7│   links to   │PR=0.4│
  └──────┘              └──────┘
```

### ML-Based Ranking / 基于 ML 的排序

```
┌─────────────────────────────────────────────────────────┐
│              ML Ranking Pipeline                         │
│                                                         │
│  Features:                                              │
│  - BM25 score                                           │
│  - PageRank                                             │
│  - Click-through rate (CTR)                             │
│  - Dwell time                                           │
│  - Freshness (last update time)                         │
│  - Domain authority                                     │
│  - Content quality signals                              │
│  - User location/language                               │
│                                                         │
│  Model: Learning-to-Rank (LambdaMART / BERT-based)     │
│                                                         │
│  Training Data:                                         │
│  - Click logs (implicit feedback)                       │
│  - Human ratings (explicit feedback)                    │
│  - A/B test results                                     │
└─────────────────────────────────────────────────────────┘
```

---

## Database Schema

### Document Store / 文档存储

```sql
CREATE TABLE documents (
    doc_id        BIGINT PRIMARY KEY,
    url           VARCHAR(2048) UNIQUE NOT NULL,
    domain        VARCHAR(255) NOT NULL,
    title         VARCHAR(1000),
    content       TEXT,
    content_hash  VARCHAR(64),           -- for dedup
    language      VARCHAR(5),
    last_crawled  TIMESTAMP,
    last_modified TIMESTAMP,
    page_rank     FLOAT DEFAULT 0.0,
    status        ENUM('active','dead','blocked'),
    content_length INT,

    INDEX idx_domain (domain),
    INDEX idx_status (status),
    INDEX idx_last_crawled (last_crawled)
);
```

### Index Metadata / 索引元数据

```sql
CREATE TABLE index_shards (
    shard_id      INT PRIMARY KEY,
    term_range_start VARCHAR(100),
    term_range_end   VARCHAR(100),
    server_id     INT,
    doc_count     BIGINT,
    size_bytes    BIGINT,
    last_updated  TIMESTAMP
);
```

### Query Log / 查询日志

```sql
CREATE TABLE query_log (
    query_id      BIGINT PRIMARY KEY,
    query_text    VARCHAR(500),
    results_count INT,
    clicked_url   VARCHAR(2048),
    click_position INT,
    user_id       BIGINT,
    timestamp     TIMESTAMP,
    search_time_ms INT,

    INDEX idx_timestamp (timestamp),
    INDEX idx_query_hash (query_text)
) PARTITION BY RANGE (timestamp);
```

---

## Scaling Strategy

### Index Sharding / 索引分片

```
Two sharding strategies:

1. Document-based sharding:
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ Shard 0  │  │ Shard 1  │  │ Shard 2  │
   │ Doc 0-33B│  │Doc 33-66B│  │Doc 66-100B│
   └──────────┘  └──────────┘  └──────────┘

   Query → Scatter to all shards → Gather results → Merge rank

2. Term-based sharding:
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ Shard 0  │  │ Shard 1  │  │ Shard 2  │
   │ Terms A-H│  │ Terms I-P│  │ Terms Q-Z│
   └──────────┘  └──────────┘  └──────────┘

   Query → Route to relevant shard(s) based on terms

   Hybrid: Term-based for hot terms, document-based for cold
```

### Cache Strategy / 缓存策略

```
┌─────────────────────────────────────────────────────┐
│              Multi-layer Cache                       │
│                                                     │
│  L1: Query Result Cache (Redis, 5 min TTL)          │
│      Key: hash(normalized_query)                    │
│      Hit rate: ~30% (popular queries)               │
│                                                     │
│  L2: Posting List Cache (in-memory, 1 hr)           │
│      Hot terms' posting lists                       │
│      Hit rate: ~60%                                 │
│                                                     │
│  L3: Document Cache (CDN, 24 hr)                    │
│      Cached page snapshots for snippets             │
│      Hit rate: ~80%                                 │
│                                                     │
│  Overall: ~95% cache hit rate                       │
└─────────────────────────────────────────────────────┘
```

### Freshness Pipeline / 新鲜度管道

```
┌─────────────────────────────────────────────────────────┐
│              Index Freshness                              │
│                                                         │
│  Tier 1: Real-time (seconds)                            │
│  - Breaking news, social media                          │
│  - Kafka stream → direct index update                   │
│                                                         │
│  Tier 2: Near real-time (minutes)                       │
│  - Frequently updated sites                             │
│  - Priority crawl queue                                 │
│                                                         │
│  Tier 3: Batch (hours/days)                             │
│  - Static content, archives                             │
│  - Scheduled batch crawl                                │
│                                                         │
│  Freshness = f(page_importance, update_frequency)       │
└─────────────────────────────────────────────────────────┘
```

---

## Trade-offs

| Decision | Option A | Option B | Chosen |
|----------|----------|----------|--------|
| Index structure | Inverted index | Forward index | Inverted (fast search) |
| Crawler | BFS | Priority-based | Priority (freshness) |
| Ranking | TF-IDF | BM25 + PageRank | BM25 + ML |
| Sharding | By document | By term | Hybrid |
| Index store | Custom | Elasticsearch | Custom (scale) |
| Freshness | Batch | Real-time | Tiered |
| Query cache | Redis | In-process | Multi-layer |

---

## 中文版本

### 需求分析

**功能需求**：爬取和索引数十亿网页、全文搜索和相关性排序、布尔运算符、自动补全/建议、拼写纠错、搜索过滤。

**非功能需求**：搜索延迟 <500ms（p99）、新页面 24 小时内可搜索、99.99% 可用性、支持 10 万 QPS、1000 亿+ 索引页面。

### 容量估算

- 爬取速率：~38,500 页/秒
- 查询速率：10 万 QPS（峰值 30 万）
- 文档存储：~200 TB（压缩后）
- 倒排索引：~20 TB（压缩后）
- 总存储（含 3 副本）：~690 TB

### 高层设计

分为**在线路径**和**离线路径**：

**离线路径**：Web 爬虫 → 内容解析器 → 倒排索引构建器 → 索引存储（Elasticsearch）。

**在线路径**：用户查询 → 查询处理器（分词、纠错、扩展）→ 索引查找 → 排序器（TF-IDF + BM25 + PageRank + ML）→ 结果组装（摘要、过滤、广告）。

### Web 爬虫

URL 前沿队列按优先级分为高/中/低三级。爬虫遵守 robots.txt，每个域名限制 1 请求/秒。使用 BFS 发现新页面，优先级爬取保证新鲜度。

### 倒排索引

倒排索引将每个词项映射到包含它的文档列表。查询"cat AND sat"通过交集操作快速找到同时包含两个词的文档。使用 MapReduce 构建索引，支持增量更新。

倒排列表使用变长字节编码、差值编码和帧内引用压缩，大幅减少存储空间。

### 查询处理管道

分词 → 规范化 → 词干提取 → 停用词去除 → 拼写检查 → 查询扩展（同义词）→ 索引查找 → 排序 → 结果组装。

### 排序算法

**TF-IDF**：词频 × 逆文档频率。简单但有效。

**BM25**：改进的 TF-IDF，加入词频饱和度和文档长度归一化。

**PageRank**：基于链接图的网页权威度计算。

**ML 排序**：综合 BM25 分数、PageRank、点击率、停留时间、新鲜度、域名权威度等特征，使用 Learning-to-Rank 模型。

### 扩展策略

- **索引分片**：按文档分片（通用）或按词项分片（热门词），混合方案最优
- **缓存**：查询结果缓存（Redis）+ 倒排列表缓存（内存）+ 文档缓存（CDN），总命中率 ~95%
- **新鲜度分层**：实时（秒级，Kafka 流）+ 近实时（分钟级，优先爬取）+ 批量（小时级，定时爬取）

### 权衡决策

| 决策 | 选择 | 原因 |
|------|------|------|
| 索引结构 | 倒排索引 | 搜索速度快 |
| 爬虫策略 | 优先级爬取 | 保证新鲜度 |
| 排序算法 | BM25 + ML | 精度和效果平衡 |
| 分片策略 | 混合方案 | 兼顾热门词和长尾 |
| 索引存储 | 自建 | 规模需求 |
| 新鲜度 | 分层方案 | 平衡实时性和成本 |
