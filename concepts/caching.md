# Caching

# 缓存

Storing frequently accessed data closer to the consumer for faster retrieval.

将频繁访问的数据存储在离消费者更近的位置以加快检索速度。

---

## Table of Contents / 目录

- [Why Caching / 为什么需要缓存](#why-caching)
- [Cache Levels / 缓存层级](#cache-levels)
- [Caching Strategies / 缓存策略](#caching-strategies)
- [Cache Eviction Policies / 缓存淘汰策略](#cache-eviction-policies)
- [Redis Deep Dive / Redis 深入解析](#redis-deep-dive)
- [CDN / 内容分发网络](#cdn)
- [Browser Cache / 浏览器缓存](#browser-cache)
- [Cache Stampede / 缓存击穿](#cache-stampede)
- [Trade-off Analysis / 权衡分析](#trade-off-analysis)
- [中文版本](#中文版本)

---

## Why Caching

### Without Cache / 无缓存

```
┌────────┐     100ms      ┌──────────┐     50ms     ┌──────────┐
│ Client │───────────────▶│  Server  │─────────────▶│ Database │
│        │◀───────────────│          │◀─────────────│          │
└────────┘    response     └──────────┘    query      └──────────┘

Total latency: 150ms per request
Database load: 10,000 QPS (all hitting DB)
```

### With Cache / 有缓存

```
┌────────┐     100ms      ┌──────────┐
│ Client │───────────────▶│  Server  │
│        │◀───────────────│          │─── Cache HIT (5ms) ──▶ Redis
└────────┘    response     └──────────┘                       Cache

Total latency: 105ms per request (cache hit)
Database load: 1,000 QPS (90% cache hit rate)
```

### Cache Hit Rate Impact / 缓存命中率影响

```
Hit Rate    DB Load Reduction    Response Time
──────────────────────────────────────────────
  50%            2x                50% faster
  80%            5x                80% faster
  90%           10x                90% faster
  99%          100x                99% faster

  Cache hit rate is the most important cache metric!
  缓存命中率是最重要的缓存指标！
```

---

## Cache Levels

```
┌─────────────────────────────────────────────────────────┐
│                    Cache Hierarchy                       │
│                    缓存层级                               │
│                                                         │
│  Fastest ◄──────────────────────────────────► Slowest   │
│  Smallest ◄────────────────────────────────► Largest    │
│                                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐ │
│  │  CPU    │  │ Browser │  │   CDN   │  │  Server  │ │
│  │  Cache  │  │  Cache  │  │  Edge   │  │  Cache   │ │
│  │         │  │         │  │  Cache  │  │ (Redis)  │ │
│  │ ~1ns    │  │ ~0ms    │  │ ~10ms   │  │ ~1ms     │ │
│  │ ~KB     │  │ ~MB     │  │ ~GB     │  │ ~GB-TB   │ │
│  └─────────┘  └─────────┘  └─────────┘  └──────────┘ │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Level Descriptions / 层级描述

| Level | Location | Size | Latency | Use Case |
|-------|----------|------|---------|----------|
| L1/L2 CPU | CPU chip | 64KB-1MB | ~1ns | Hardware |
| Browser | Client browser | 50-200MB | ~0ms | Static assets |
| CDN | Edge servers | TB | 10-50ms | Global content |
| Application | In-process | MB-GB | <1ms | Hot data |
| Redis/Memcached | Dedicated servers | GB-TB | 1-5ms | Shared cache |

---

## Caching Strategies

### 1. Cache-Aside (Lazy Loading) / 旁路缓存

```
Read Flow:
  ┌────────┐                    ┌───────┐
  │ Server │──1. Check cache──▶│ Cache │
  │        │   (GET key)       │(Redis)│
  │        │◀──2. MISS─────────│       │
  │        │                    └───────┘
  │        │
  │        │──3. Query DB────▶┌──────────┐
  │        │◀──4. Result──────│ Database │
  │        │                  └──────────┘
  │        │
  │        │──5. Set cache──▶┌───────┐
  │        │   (SET key,val) │ Cache │
  └────────┘                  └───────┘

Next read:
  ┌────────┐                    ┌───────┐
  │ Server │──1. Check cache──▶│ Cache │
  │        │◀──2. HIT, return──│       │  ← Fast!
  └────────┘                  └───────┘
```

**Pros:** Only caches data that's actually requested; cache failure is non-fatal
**Cons:** First request always slow (cache miss); stale data possible

### 2. Write-Through / 写穿透

```
Write Flow:
  ┌────────┐                   ┌───────┐    ┌──────────┐
  │ Server │──1. Write data───▶│ Cache │───▶│ Database │
  │        │                   │(Redis)│    │          │
  └────────┘                   └───────┘    └──────────┘
                                    │            │
                              Write to     Write to
                              both atomically

Read Flow (always from cache):
  ┌────────┐                   ┌───────┐
  │ Server │──1. Read─────────▶│ Cache │  ← Always hit!
  │        │◀──2. Return───────│       │
  └────────┘                   └───────┘
```

**Pros:** Cache always consistent; reads always fast
**Cons:** Write latency (must write to both); most written data may never be read

### 3. Write-Behind (Write-Back) / 写回

```
Write Flow:
  ┌────────┐                   ┌───────┐
  │ Server │──1. Write data───▶│ Cache │  ← Returns immediately!
  │        │   (acknowledged)  │(Redis)│
  └────────┘                   └───┬───┘
                                   │
                          2. Async flush (batch)
                                   │
                              ┌────▼────┐
                              │ Database│
                              └─────────┘

  Batches writes for efficiency.
  数据库写入异步批量执行。
```

**Pros:** Very fast writes; batches reduce DB load
**Cons:** Data loss risk if cache crashes before flush; complex

### 4. Read-Through / 读穿透

```
Read Flow:
  ┌────────┐                   ┌───────┐    ┌──────────┐
  │ Server │──1. Read─────────▶│ Cache │───▶│ Database │
  │        │                   │(cache │    │ (if miss)│
  │        │◀──2. Return───────│ loads)│◀───│          │
  └────────┘                   └───────┘    └──────────┘

  Cache handles DB loading automatically.
  缓存自动从数据库加载数据。
```

### 5. Refresh-Ahead / 预刷新

```
Cache monitors access patterns:

  Key "user:123" accessed frequently
  TTL expires at T+300s

  At T+240s (80% of TTL):
    Cache proactively refreshes from DB
    No cache miss ever occurs for hot keys
```

### Strategy Comparison / 策略对比

| Strategy | Consistency | Latency | Complexity | Use Case |
|----------|-------------|---------|------------|----------|
| Cache-Aside | Eventual | Variable | Low | General purpose |
| Write-Through | Strong | High writes | Medium | Read-heavy, consistent |
| Write-Behind | Eventual | Low writes | High | Write-heavy |
| Read-Through | Eventual | Low reads | Medium | Read-heavy |
| Refresh-Ahead | Strong | Very low | High | Hot keys |

---

## Cache Eviction Policies

### LRU (Least Recently Used) / 最近最少使用

```
Cache capacity: 3
Access sequence: A, B, C, A, D

Step 1: [A]           Insert A
Step 2: [A, B]        Insert B
Step 3: [A, B, C]     Insert C (cache full)
Step 4: [B, C, A]     Access A (move to front)
Step 5: [C, A, D]     Insert D, evict B (least recently used)
```

### LFU (Least Frequently Used) / 最不经常使用

```
Access counts: A=5, B=2, C=1, D=3

When eviction needed: evict C (lowest count = 1)

Problem: New items with count=1 get evicted quickly
Solution: Decay counts over time
```

### Other Policies / 其他策略

| Policy | Description | Pros | Cons |
|--------|-------------|------|------|
| FIFO | First in, first out | Simple | Ignores usage |
| LIFO | Last in, first out | Simple | Poor hit rate |
| Random | Random eviction | Very simple | Unpredictable |
| TTL | Time-based expiry | Predictable | May evict hot data |
| Size-based | Evict largest items | Saves space | May lose important data |

---

## Redis Deep Dive

### Redis Data Structures / Redis 数据结构

```
┌─────────────────────────────────────────────┐
│              Redis Data Types                │
├─────────────────────────────────────────────┤
│                                             │
│  String    "hello"                          │
│            GET/SET/INCR/APPEND              │
│                                             │
│  List      [1, 2, 3, 4]                     │
│            LPUSH/RPUSH/LPOP/LRANGE          │
│                                             │
│  Set       {a, b, c}                        │
│            SADD/SMEMBERS/SINTER             │
│                                             │
│  Hash      {name: "John", age: 30}          │
│            HGET/HSET/HGETALL                │
│                                             │
│  Sorted Set  [(score1, member1), ...]       │
│            ZADD/ZRANGE/ZRANK                │
│                                             │
│  Stream    Message queue with consumer grp  │
│            XADD/XREAD/XREADGROUP            │
│                                             │
└─────────────────────────────────────────────┘
```

### Redis Cluster Architecture / Redis 集群架构

```
┌───────────────────────────────────────────────────┐
│                 Redis Cluster                      │
│                                                   │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐          │
│  │ Master 1│  │ Master 2│  │ Master 3│          │
│  │Slot 0-  │  │Slot 5461│  │Slot 10923│         │
│  │ 5460    │  │-10922   │  │-16383   │          │
│  └────┬────┘  └────┬────┘  └────┬────┘          │
│       │            │            │                 │
│  ┌────▼────┐  ┌────▼────┐  ┌────▼────┐          │
│  │Replica 1│  │Replica 2│  │Replica 3│          │
│  └─────────┘  └─────────┘  └─────────┘          │
│                                                   │
│  16384 hash slots distributed across masters     │
│  Automatic failover via replicas                  │
└───────────────────────────────────────────────────┘
```

### Redis vs Memcached / Redis 与 Memcached 对比

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data Structures | Rich (String, List, Set, Hash, Sorted Set, Stream) | String only |
| Persistence | RDB + AOF | None |
| Replication | Built-in | None |
| Cluster Mode | Built-in | Client-side |
| Max Value Size | 512MB | 1MB |
| Memory Efficiency | Lower | Higher |
| Use Case | Complex data, pub/sub, queues | Simple key-value caching |

---

## CDN

### How CDN Works / CDN 工作原理

```
User in Tokyo requests image:

Without CDN:
  Tokyo ──────── 150ms ──────────▶ US Server
  (slow, crosses ocean)

With CDN:
  Tokyo ── 10ms ──▶ CDN Edge (Tokyo)
                      │
                      ├── Cache HIT → Return immediately
                      │
                      └── Cache MISS → Fetch from US Server
                                       Cache at edge
                                       Return to user

  Next request from Tokyo: Cache HIT (10ms)
```

### CDN Architecture / CDN 架构

```
                    ┌──────────────┐
                    │   Origin     │
                    │   Server     │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼─────┐┌────▼──────┐┌────▼──────┐
        │ Edge NYC  ││ Edge Tokyo││ Edge London│
        │ (US East) ││ (Asia)    ││ (Europe)   │
        └─────┬─────┘└─────┬─────┘└─────┬─────┘
              │            │            │
         ┌────▼───┐   ┌────▼───┐   ┌────▼───┐
         │ Users  │   │ Users  │   │ Users  │
         │ (US)   │   │ (Japan)│   │ (UK)   │
         └────────┘   └────────┘   └────────┘
```

### CDN Cache Headers / CDN 缓存头

```http
Cache-Control: public, max-age=31536000    # 1 year for static assets
Cache-Control: private, no-cache           # Don't cache in CDN
Cache-Control: public, s-maxage=3600       # CDN cache 1 hour
ETag: "abc123"                             # Conditional request
Last-Modified: Wed, 21 Oct 2025 07:28:00  # Conditional request
```

---

## Browser Cache

### Browser Cache Hierarchy / 浏览器缓存层级

```
┌──────────────────────────────────────────────┐
│            Browser Cache Priority             │
│                                              │
│  1. Memory Cache (fastest)                   │
│     └─ Recent resources in RAM               │
│                                              │
│  2. Service Worker Cache                     │
│     └─ Programmable cache                    │
│                                              │
│  3. HTTP Cache (disk)                        │
│     ├─ Strong cache (Cache-Control/Expires)  │
│     └─协商 cache (ETag/Last-Modified)        │
│                                              │
│  4. Push Cache (HTTP/2)                      │
│     └─ Session-level, per connection         │
│                                              │
└──────────────────────────────────────────────┘
```

---

## Cache Stampede

### The Problem / 问题

```
Cache expires → 1000 concurrent requests all miss cache
→ All 1000 hit database simultaneously
→ Database overwhelmed → crash

  Request ──▶ Cache MISS ──▶ DB query ──▶ expensive!
  Request ──▶ Cache MISS ──▶ DB query ──▶ expensive!
  Request ──▶ Cache MISS ──▶ DB query ──▶ expensive!
  ... (1000 times)
```

### Solutions / 解决方案

```
1. Locking / 分布式锁:
   ┌────────┐     ┌─────────┐
   │ Req 1  │────▶│  GET    │──▶ MISS → Acquire lock → Query DB
   │ Req 2  │────▶│  lock   │──▶ MISS → Wait for lock → Read from cache
   │ Req 3  │────▶│         │──▶ MISS → Wait for lock → Read from cache
   └────────┘     └─────────┘

2. Probabilistic Early Expiration:
   if (current_time > ttl - random(delta)):
       refresh_cache_async()

3. Stale-While-Revalidate:
   Return stale data immediately
   Refresh cache in background
```

---

## Trade-off Analysis

### When to Use Cache / 何时使用缓存

| Scenario | Cache? | Reason |
|----------|--------|--------|
| Read-heavy workload | ✓ Yes | High hit rate, reduce DB load |
| Write-heavy workload | ✗ Maybe | Low hit rate, added complexity |
| Real-time data | ✗ No | Stale data unacceptable |
| User session | ✓ Yes | Fast access, short-lived |
| Expensive computations | ✓ Yes | Avoid recomputation |
| Geographically distributed | ✓ Yes (CDN) | Reduce latency |

### Cache Invalidation / 缓存失效

```
"The two hardest things in CS:
 1. Cache invalidation
 2. Naming things
 3. Off-by-one errors"

Invalidation strategies:
  1. TTL-based:  Set expiry time, auto-delete
  2. Event-based: Invalidate on write (pub/sub)
  3. Version-based: Include version in cache key
  4. Manual: Explicitly delete on update
```

---

## 中文版本

### 为什么需要缓存

缓存将频繁访问的数据存储在离消费者更近的位置。无缓存时，每次请求都要查询数据库（~150ms）；有缓存时，90%+ 的请求可从缓存直接返回（~105ms），数据库负载降低 10 倍。

缓存命中率是最重要的指标——90% 命中率意味着数据库负载降低 10 倍。

### 缓存层级

从快到慢、从小到大：CPU 缓存（~1ns）→ 浏览器缓存（~0ms）→ CDN 边缘缓存（~10ms）→ 应用服务器缓存/Redis（~1ms）。

### 缓存策略

**旁路缓存（Cache-Aside）**：最常用。读时先查缓存，未命中则查数据库并写入缓存。写时先更新数据库，再删除缓存。

**写穿透（Write-Through）**：写操作同时写缓存和数据库。缓存始终一致，但写延迟高。

**写回（Write-Behind）**：写操作只写缓存，异步批量写入数据库。写速度快，但有数据丢失风险。

**读穿透（Read-Through）**：缓存自动从数据库加载数据，应用只需与缓存交互。

### 缓存淘汰策略

| 策略 | 描述 | 适用场景 |
|------|------|----------|
| LRU | 淘汰最久未使用的 | 通用场景，最常用 |
| LFU | 淘汰使用频率最低的 | 有明显热点数据 |
| FIFO | 先进先出 | 数据有时效性 |
| TTL | 按过期时间淘汰 | 数据有明确有效期 |

### Redis 深入解析

Redis 支持丰富的数据结构：String、List、Set、Hash、Sorted Set、Stream。Redis 集群将 16384 个哈希槽分布在多个主节点上，每个主节点有副本实现自动故障转移。

Redis 与 Memcached 对比：Redis 支持丰富数据结构和持久化，Memcached 只支持字符串但内存效率更高。

### CDN 内容分发网络

CDN 将内容缓存到全球边缘节点。用户请求时，从最近的边缘节点返回，延迟从 ~150ms 降到 ~10ms。适用于静态资源（图片、CSS、JS）。

### 缓存击穿

缓存击穿是指缓存过期后大量并发请求同时打到数据库。解决方案：
1. **分布式锁**：只有一个请求去查数据库，其他请求等待
2. **概率性提前刷新**：在 TTL 到期前随机时间刷新
3. **返回旧数据**：立即返回旧数据，后台刷新缓存

### 何时使用缓存

| 场景 | 是否缓存 | 原因 |
|------|---------|------|
| 读多写少 | ✓ 是 | 高命中率，降低数据库负载 |
| 写多读少 | ✗ 慎用 | 命中率低，增加复杂度 |
| 实时数据 | ✗ 否 | 旧数据不可接受 |
| 用户会话 | ✓ 是 | 快速访问，短期有效 |
| 复杂计算结果 | ✓ 是 | 避免重复计算 |
| 全球分布用户 | ✓ CDN | 降低延迟 |
