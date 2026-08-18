# Design a Distributed Cache

# 设计分布式缓存

Design a distributed caching system like Redis Cluster or Memcached.

设计一个类似 Redis 集群或 Memcached 的分布式缓存系统。

---

## Table of Contents / 目录

- [Requirements / 需求分析](#requirements)
- [Capacity Estimation / 容量估算](#capacity-estimation)
- [High-Level Design / 高层设计](#high-level-design)
- [API Design / API 设计](#api-design)
- [Data Partitioning / 数据分区](#data-partitioning)
- [Replication / 复制](#replication)
- [Consistency / 一致性](#consistency)
- [Eviction Policy / 淘汰策略](#eviction-policy)
- [Cache Invalidation / 缓存失效](#cache-invalidation)
- [Failure Handling / 故障处理](#failure-handling)
- [Scaling Strategy / 扩展策略](#scaling-strategy)
- [Trade-offs / 权衡分析](#trade-offs)
- [中文版本](#中文版本)

---

## Requirements

### Functional Requirements / 功能需求

1. `put(key, value, ttl)` — Store key-value pair with optional TTL
2. `get(key)` — Retrieve value by key
3. `delete(key)` — Remove key
4. `batch_get(keys)` — Retrieve multiple keys
5. `batch_put(kvpairs)` — Store multiple key-value pairs
6. Support for various data types (string, hash, list, set, sorted set)
7. Atomic operations (INCR, DECR, APPEND)

### Non-Functional Requirements / 非功能需求

1. Ultra-low latency: < 1ms for reads, < 2ms for writes
2. High throughput: 1M+ QPS per cluster
3. High availability: 99.999% (five nines)
4. Horizontal scalability
5. Data durability (configurable)
6. Linearizable reads (optional, for strong consistency mode)

---

## Capacity Estimation

### Traffic Estimates / 流量估算

```
Assumptions:
  - 1B total cache entries
  - 1M QPS (peak 3M)
  - 70% reads, 30% writes
  - Average value size: 1KB
  - Hot data: 20% of entries serve 80% of traffic

Read QPS:
  1M × 0.7 = 700K reads/sec

Write QPS:
  1M × 0.3 = 300K writes/sec
```

### Storage Estimates / 存储估算

```
Cache entries:
  1B entries × 1KB = 1 TB

With overhead (metadata, pointers, fragmentation):
  ~1.5 TB usable memory

With replicas (2x for durability):
  ~3 TB total memory

Across 64 GB servers:
  3 TB / 64 GB = 47 servers minimum
  With headroom (70% utilization): ~67 servers
```

### Bandwidth Estimates / 带宽估算

```
Inbound (writes):
  300K writes/sec × 1KB = 300 MB/sec

Outbound (reads):
  700K reads/sec × 1KB = 700 MB/sec

Internal (replication):
  300K writes/sec × 1KB × 2 replicas = 600 MB/sec
```

---

## High-Level Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Distributed Cache Architecture                     │
│                                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                     │
│  │ Client 1 │    │ Client 2 │    │ Client 3 │                     │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘                     │
│       │               │               │                            │
│       └───────────────┼───────────────┘                            │
│                       │                                             │
│                       ▼                                             │
│              ┌────────────────┐                                    │
│              │   Client SDK   │                                    │
│              │  (Consistent   │                                    │
│              │   Hash Ring)   │                                    │
│              └───────┬────────┘                                    │
│                      │                                              │
│       ┌──────────────┼──────────────┐                             │
│       │              │              │                              │
│       ▼              ▼              ▼                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                        │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │                        │
│  │          │  │          │  │          │                        │
│  │ Slot 0-  │  │ Slot 5461│  │ Slot 10923│                        │
│  │ 5460     │  │ -10922   │  │ -16383   │                        │
│  │          │  │          │  │          │                        │
│  │ Primary  │  │ Primary  │  │ Primary  │                        │
│  └──┬───┬───┘  └──┬───┬───┘  └──┬───┬───┘                        │
│     │   │         │   │         │   │                             │
│     ▼   ▼         ▼   ▼         ▼   ▼                             │
│  ┌────┐┌────┐ ┌────┐┌────┐ ┌────┐┌────┐                          │
│  │Rep1││Rep2│ │Rep3││Rep4│ │Rep5││Rep6│  (Replicas)             │
│  └────┘└────┘ └────┘└────┘ └────┘└────┘                          │
│                                                                     │
│  ┌────────────────────────────────────────────────┐               │
│  │           Cluster Manager / Coordinator          │               │
│  │                                                  │               │
│  │  - Node discovery and health monitoring          │               │
│  │  - Slot assignment and rebalancing               │               │
│  │  - Failure detection and failover                │               │
│  │  - Configuration management                      │               │
│  └────────────────────────────────────────────────┘               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## API Design

### Single Key Operations / 单键操作

```bash
# Store with TTL (seconds)
PUT /cache/{key}
Content-Type: application/json
{
  "value": {"user_id": 123, "name": "John"},
  "ttl": 3600
}
Response: 201 Created

# Retrieve
GET /cache/{key}
Response: 200 OK
{
  "value": {"user_id": 123, "name": "John"},
  "ttl_remaining": 3542
}

# Delete
DELETE /cache/{key}
Response: 204 No Content
```

### Batch Operations / 批量操作

```bash
# Batch get
POST /cache/mget
{
  "keys": ["user:123", "user:456", "user:789"]
}
Response:
{
  "results": {
    "user:123": {"name": "John"},
    "user:456": {"name": "Jane"},
    "user:789": null
  }
}

# Batch put
POST /cache/mset
{
  "entries": [
    {"key": "user:123", "value": {"name": "John"}, "ttl": 3600},
    {"key": "user:456", "value": {"name": "Jane"}, "ttl": 3600}
  ]
}
```

### Atomic Operations / 原子操作

```bash
# Increment
POST /cache/{key}/incr
{
  "amount": 1
}
Response: {"value": 43}

# Compare and swap (CAS)
PUT /cache/{key}/cas
{
  "value": "new_value",
  "cas_token": "abc123"
}
Response: 200 OK (or 409 Conflict if token mismatch)

# Set if not exists (SETNX)
POST /cache/{key}/setnx
{
  "value": "only_if_not_exists",
  "ttl": 300
}
Response: 201 Created (or 409 Already Exists)
```

---

## Data Partitioning

### Consistent Hashing with Virtual Nodes / 带虚拟节点的一致性哈希

```
Hash Ring with 16384 slots:

              slot 0
               │
      Node A ●─────────────● Node B
     (0-5460)  │             │ (5461-10922)
               │             │
               │   ╭────╮    │
               │   │Ring│    │
               │   ╰────╯    │
               │             │
      Node C ●─────────────● slot 16383
     (10923-16383)

Key mapping:
  hash(key) → slot_number → responsible_node

Example:
  key = "user:123"
  hash("user:123") = 7456
  slot 7456 → Node B (5461-10922)
```

### Slot Assignment / 槽位分配

```
┌─────────────────────────────────────────────────────────┐
│                Slot Assignment Table                      │
│                                                         │
│  Slot Range    │ Primary Node │ Replica 1 │ Replica 2  │
│  ──────────────┼──────────────┼───────────┼────────────│
│  0 - 5460     │    Node A    │  Node B   │  Node C    │
│  5461 - 10922 │    Node B    │  Node C   │  Node A    │
│  10923 - 16383│    Node C    │  Node A   │  Node B    │
│                                                         │
│  Gossip protocol propagates this table to all nodes     │
└─────────────────────────────────────────────────────────┘
```

### Resharding / 重新分片

```
Adding Node D to cluster:

Before:
  Node A: slots 0-5460      (5461 slots)
  Node B: slots 5461-10922  (5462 slots)
  Node C: slots 10923-16383 (5461 slots)

After:
  Node A: slots 0-4095      (4096 slots)
  Node B: slots 4096-8191   (4096 slots)
  Node C: slots 8192-12287  (4096 slots)
  Node D: slots 12288-16383 (4096 slots)

  Slots migrated (not data copied):
  1. Node A: slots 4096-5460 → Node D
  2. Node B: slots 8192-10922 → Node D
  3. Node C: slots 12288-16383 already on Node D

  Migration is online, no downtime!
```

---

## Replication

### Asynchronous Replication / 异步复制

```
Write Flow:
  Client ──PUT──▶ Primary Node
                      │
                      ├── 1. Write to memory (ACK to client)
                      │
                      ├── 2. Write to WAL (Write-Ahead Log)
                      │
                      └── 3. Async replicate to replicas
                              │
                         ┌────┼────┐
                         ▼    ▼    ▼
                      Rep1  Rep2  Rep3
                      (async, may lag)

  Latency: < 1ms (no replica wait)
  Risk: Data loss if primary crashes before replication
```

### Semi-Synchronous Replication / 半同步复制

```
Write Flow:
  Client ──PUT──▶ Primary Node
                      │
                      ├── 1. Write to memory
                      │
                      ├── 2. Replicate to at least 1 replica
                      │       │
                      │       └──▶ Replica ACK
                      │
                      └── 3. ACK to client

  Latency: ~2ms (waits for 1 replica)
  Trade-off: Higher latency, lower data loss risk
```

### Replication Log / 复制日志

```
┌─────────────────────────────────────────────────────────┐
│               Replication Strategies                      │
│                                                         │
│  1. Full Sync:                                         │
│     Primary sends entire dataset to replica             │
│     Used: Initial sync, recovery after long disconnect  │
│     Cost: Very high for large datasets                  │
│                                                         │
│  2. Partial Resync (PSYNC):                            │
│     Use replication backlog buffer                      │
│     ┌─────────────────────────────────────────┐        │
│     │ Backlog Buffer (1MB circular buffer)    │        │
│     │ [offset 1000 ... offset 2000]           │        │
│     └─────────────────────────────────────────┘        │
│     If replica offset is within buffer → partial sync  │
│     Otherwise → full sync                              │
│                                                         │
│  3. RDB Snapshots:                                     │
│     Periodic point-in-time snapshots                    │
│     Compact binary format                               │
│     Used for backups and initial replication            │
└─────────────────────────────────────────────────────────┘
```

---

## Consistency

### Consistency Levels / 一致性级别

```
┌─────────────────────────────────────────────────────────┐
│              Consistency Levels                          │
│                                                         │
│  ONE:    Write/read from single node                    │
│          Fastest, weakest consistency                    │
│                                                         │
│  QUORUM: Write/read from majority (N/2 + 1)            │
│          Balance of speed and consistency                │
│                                                         │
│  ALL:    Write/read from all nodes                      │
│          Slowest, strongest consistency                  │
│                                                         │
│  Example with 3 replicas:                              │
│  ┌──────────┬──────────┬──────────┬──────────┐         │
│  │ Level    │ Nodes    │ Latency  │ Consist. │         │
│  ├──────────┼──────────┼──────────┼──────────┤         │
│  │ ONE      │ 1        │ ~1ms     │ Weak     │         │
│  │ QUORUM   │ 2        │ ~2ms     │ Strong   │         │
│  │ ALL      │ 3        │ ~3ms     │ Strongest│         │
│  └──────────┴──────────┴──────────┴──────────┘         │
└─────────────────────────────────────────────────────────┘
```

### Conflict Resolution / 冲突解决

```
Network partition scenario:

  Before partition:
    Node A (Primary) ←── consistent
    Node B (Replica)  ←── consistent

  During partition:
    Node A (Primary) ── writes x=1
    Node B (Promoted) ── writes x=2  (split brain!)

  Resolution strategies:
  1. Last-Write-Wins (LWW): Use timestamps
     → x=2 wins (if Node B wrote later)

  2. Vector Clocks:
     → Track causal relationships
     → Detect conflicts, let application resolve

  3. CRDT (Conflict-free Replicated Data Types):
     → Data structures that automatically merge
     → G-Counter, LWW-Register, OR-Set

  4. Lease-based:
     → Primary holds lease (time-limited authority)
     → Old primary cannot accept writes after lease expires
```

---

## Eviction Policy

### LRU Implementation / LRU 实现

```
Doubly Linked List + Hash Map:

  ┌───────────────────────────────────────────────────┐
  │                  LRU Cache                         │
  │                                                   │
  │  Hash Map:           Doubly Linked List:          │
  │  ┌─────────┐         ┌───┐   ┌───┐   ┌───┐     │
  │  │ key →   │────────▶│ A │◄─▶│ B │◄─▶│ C │     │
  │  │ node    │         │MRU│   │   │   │LRU│     │
  │  └─────────┘         └───┘   └───┘   └───┘     │
  │                                                   │
  │  GET(key):                                        │
  │    1. Hash lookup → node O(1)                     │
  │    2. Move node to head (MRU position)            │
  │    3. Return value                                │
  │                                                   │
  │  PUT(key, value):                                 │
  │    1. If exists: update + move to head            │
  │    2. If new: insert at head                      │
  │    3. If over capacity: evict tail (LRU)          │
  │                                                   │
  │  All operations: O(1)                             │
  └───────────────────────────────────────────────────┘
```

### LRU Variants / LRU 变体

```
1. LRU-K (LRU with access frequency):
   Track last K access times
   Better at distinguishing one-time vs repeated access

2. TinyLFU (Least Frequently Used with aging):
   ┌──────────────────────────────────────────┐
   │  Admission Policy:                       │
   │                                          │
   │  New item ──▶ Count-Min Sketch           │
   │               (estimate frequency)       │
   │                                          │
   │  If freq(new) > freq(victim):            │
   │    Admit new item, evict victim          │
   │  Else:                                   │
    │    Reject new item                      │
   │                                          │
   │  Periodic frequency decay (aging)        │
   └──────────────────────────────────────────┘

3. W-TinyLFU (Window TinyLFU - used in Caffeine):
   ┌──────────────────────────────────────────┐
   │  ┌────────┐  ┌────────────────────────┐ │
   │  │ Window │─▶│  Main Cache (SegLRU)   │ │
   │  │  LRU   │  │                        │ │
   │  │(recent)│  │  (frequent items)      │ │
   │  └────────┘  └────────────────────────┘ │
   │                                          │
   │  New items enter Window LRU              │
   │  Promoted to Main if frequently accessed │
   └──────────────────────────────────────────┘
```

### Eviction Comparison / 淘汰策略对比

| Policy | Hit Rate | Memory | Complexity | Best For |
|--------|----------|--------|------------|----------|
| LRU | Good | O(n) | Simple | General purpose |
| LFU | Better | O(n) | Medium | Skewed access |
| FIFO | Fair | O(n) | Simplest | Time-series |
| Random | Variable | O(1) | Simplest | Uniform access |
| W-TinyLFU | Best | O(n) | Complex | Mixed workloads |

---

## Cache Invalidation

### TTL-Based Invalidation / 基于 TTL 的失效

```
┌─────────────────────────────────────────────────────────┐
│              TTL Invalidation                            │
│                                                         │
│  PUT key, value, TTL=3600                               │
│                                                         │
│  Time ──────────────────────────────────────────▶      │
│  T=0        T=1800       T=3599       T=3600            │
│  ┌───┐      ┌───┐        ┌───┐        ┌───┐           │
│  │key│      │key│        │key│        │   │ ← expired │
│  └───┘      └───┘        └───┘        └───┘           │
│                                      lazy delete       │
│                                      (on next access)  │
│                                                         │
│  Lazy deletion: Check TTL on access, delete if expired │
│  Active deletion: Background thread scans for expired  │
│  Hybrid: Lazy + periodic active scan                   │
└─────────────────────────────────────────────────────────┘
```

### Event-Based Invalidation / 基于事件的失效

```
┌─────────────────────────────────────────────────────────┐
│              Event-Driven Invalidation                    │
│                                                         │
│  ┌──────────┐  write   ┌──────────┐  publish  ┌─────┐│
│  │ Service A│────────▶│ Database │─────────▶│ Pub ││
│  └──────────┘         └──────────┘          │ Sub ││
│                                              └──┬──┘│
│                                                 │    │
│                              ┌───────────────────┤    │
│                              │                   │    │
│                              ▼                   ▼    │
│                        ┌──────────┐       ┌──────────┐│
│                        │ Cache    │       │ Cache    ││
│                        │ Node 1   │       │ Node 2   ││
│                        │ DEL key  │       │ DEL key  ││
│                        └──────────┘       └──────────┘│
│                                                         │
│  Write-through: Update DB + Cache atomically           │
│  Write-invalidate: Update DB, invalidate cache         │
│  Write-behind: Update cache, async update DB           │
└─────────────────────────────────────────────────────────┘
```

---

## Failure Handling

### Node Failure Detection / 节点故障检测

```
┌─────────────────────────────────────────────────────────┐
│              Failure Detection                            │
│                                                         │
│  Gossip Protocol:                                       │
│  ┌────────┐  heartbeat  ┌────────┐                    │
│  │ Node A │────────────▶│ Node B │                    │
│  │        │◀────────────│        │                    │
│  └────────┘  heartbeat  └────────┘                    │
│                                                         │
│  Each node sends heartbeat to random peers every 1s    │
│  If no heartbeat received within 5s → suspect          │
│  If no heartbeat within 15s → mark as failed           │
│                                                         │
│  Failure detection time: 5-15 seconds                  │
│                                                         │
│  ┌─────────────────────────────────────────────┐       │
│  │  Node States:                                │       │
│  │                                             │       │
│  │  PFAIL (Possible Fail):                     │       │
│  │    - Detected by single node                │       │
│  │    - Needs majority confirmation            │       │
│  │                                             │       │
│  │  FAIL (Confirmed Fail):                     │       │
│  │    - Majority of nodes agree                │       │
│  │    - Triggers failover                      │       │
│  └─────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

### Failover Process / 故障转移流程

```
Primary Node A fails:

  Step 1: Detection (5-15 sec)
    Nodes B, C detect Node A heartbeat timeout
    Mark Node A as PFAIL → majority confirms → FAIL

  Step 2: Replica Promotion
    Node A's replica (Node A-replica) receives FAIL notification
    Node A-replica runs election among replicas
    Majority votes → Node A-replica promoted to Primary

  Step 3: Cluster Reconfiguration
    New Primary (A-replica) takes over slots 0-5460
    Cluster config propagated via Gossip
    Clients discover new topology

  Step 4: Old Primary Recovery (when comes back online)
    Node A recovers → becomes replica of A-replica
    Full sync from new primary

  Timeline:
  ┌─────────────────────────────────────────────────────┐
  │ 0s        5s         15s        20s        30s      │
  │ │         │          │          │          │        │
  │ ▼         ▼          ▼          ▼          ▼        │
  │ Failure   Detect     Promote    Config     Ready    │
  │           (PFAIL)    Replica    Propagate           │
  │                                                     │
  │ Total failover time: ~15-30 seconds                │
  └─────────────────────────────────────────────────────┘
```

### Data Durability / 数据持久性

```
┌─────────────────────────────────────────────────────────┐
│              Durability Options                          │
│                                                         │
│  1. Memory Only (default):                             │
│     - Fastest (< 1ms writes)                           │
│     - Data lost on crash                                │
│     - Use: Pure cache, data recoverable from DB        │
│                                                         │
│  2. RDB Snapshots:                                     │
│     - Periodic binary snapshots (every N minutes)      │
│     - Point-in-time recovery                           │
│     - Use: Backups, disaster recovery                  │
│                                                         │
│  3. AOF (Append-Only File):                            │
│     - Log every write operation                        │
│     - fsync policy: always / every_sec / no            │
│     - Use: Durability with reasonable performance      │
│                                                         │
│  4. RDB + AOF (hybrid):                                │
│     - AOF for recent writes, RDB for fast restart      │
│     - Best of both worlds                              │
│     - Use: Production deployments                      │
│                                                         │
│  Durability vs Performance trade-off:                  │
│  Memory only < RDB < AOF(every_sec) < AOF(always)      │
│  Fastest ◄─────────────────────────────────► Durable   │
└─────────────────────────────────────────────────────────┘
```

---

## Scaling Strategy

### Horizontal Scaling / 水平扩展

```
┌─────────────────────────────────────────────────────────┐
│              Scaling Operations                          │
│                                                         │
│  Scale Out (add nodes):                                │
│  1. Add new node to cluster                            │
│  2. Reshard: migrate slots from existing nodes         │
│  3. Online migration (no downtime)                     │
│  4. Client SDK auto-discovers new topology             │
│                                                         │
│  Scale In (remove nodes):                              │
│  1. Migrate slots from node to be removed              │
│  2. Wait for migration to complete                     │
│  3. Remove node from cluster                           │
│                                                         │
│  Auto-scaling:                                         │
│  ┌──────────────────────────────────────────┐          │
│  │ Monitor: memory usage, QPS, latency      │          │
│  │                                          │          │
│  │ IF memory_usage > 75%:                   │          │
│  │   → Add node + reshard                   │          │
│  │                                          │          │
│  │ IF memory_usage < 30% AND nodes > min:   │          │
│  │   → Remove node + reshard                │          │
│  └──────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
```

### Multi-Region / 多区域

```
┌─────────────────────────────────────────────────────────┐
│              Multi-Region Deployment                     │
│                                                         │
│  ┌─────────────────┐      ┌─────────────────┐         │
│  │ Region: US-East │      │ Region: APAC    │         │
│  │                 │      │                 │         │
│  │ ┌───┐ ┌───┐   │      │ ┌───┐ ┌───┐   │         │
│  │ │ A │ │ B │   │      │ │ D │ │ E │   │         │
│  │ └───┘ └───┘   │      │ └───┘ └───┘   │         │
│  │ ┌───┐         │      │ ┌───┐         │         │
│  │ │ C │         │ Async│ │ F │         │         │
│  │ └───┘         │◄────►│ └───┘         │         │
│  └─────────────────┘ Cross-Region        └─────────────────┘
│                     Replication                         │
│                                                         │
│  Strategy:                                              │
│  - Active-Active: Both regions serve traffic           │
│  - Conflict: LWW or region-local writes                │
│  - Latency: Reads local, writes async cross-region    │
└─────────────────────────────────────────────────────────┘
```

---

## Trade-offs

| Decision | Option A | Option B | Chosen |
|----------|----------|----------|--------|
| Consistency | Strong (quorum) | Eventual | Configurable |
| Replication | Async | Semi-sync | Async (performance) |
| Eviction | LRU | W-TinyLFU | LRU (simpler) |
| Durability | Memory only | AOF | Configurable |
| Partitioning | Hash slots | Range | Hash slots (even) |
| Failover | Automatic | Manual | Automatic |
| Conflict | LWW | Vector clocks | LWW (simpler) |
| Serialization | JSON | MessagePack | MessagePack (compact) |

---

## 中文版本

### 需求分析

**功能需求**：put/get/delete 操作、批量操作、支持多种数据类型（String、Hash、List、Set、Sorted Set）、原子操作（INCR、DECR）。

**非功能需求**：超低延迟（读 <1ms，写 <2ms）、高吞吐（100 万+ QPS）、高可用（99.999%）、水平扩展、可配置持久性。

### 容量估算

- 读 QPS：70 万/秒
- 写 QPS：30 万/秒
- 缓存条目：10 亿条 × 1KB = 1 TB
- 含副本：~3 TB
- 64GB 服务器：最少 47 台，考虑余量 ~67 台

### 高层设计

客户端 SDK 内置一致性哈希环，直接路由到目标节点。每个节点管理一部分哈希槽（共 16384 个槽），每个主节点有副本实现高可用。集群管理器负责节点发现、健康监控、故障转移和重新分片。

### 数据分区

一致性哈希将 16384 个槽分配给多个节点。`hash(key) % 16384` 决定 key 所在的槽，槽映射到对应的主节点。添加新节点时，在线迁移部分槽位，无需停机。

### 复制

**异步复制**：主节点写入内存后立即返回客户端，异步复制到副本。延迟最低（<1ms），但主节点崩溃可能丢数据。

**半同步复制**：等待至少一个副本确认后才返回客户端。延迟约 2ms，数据安全性更高。

**复制日志**：支持全量同步（初始同步/恢复）和部分同步（基于复制积压缓冲区的增量同步）。

### 一致性

| 级别 | 节点数 | 延迟 | 一致性 |
|------|--------|------|--------|
| ONE | 1 | ~1ms | 弱 |
| QUORUM | N/2+1 | ~2ms | 强 |
| ALL | N | ~3ms | 最强 |

冲突解决：最后写入胜出（LWW）、向量时钟、CRDT、租约机制。

### 淘汰策略

**LRU**：哈希表 + 双向链表，所有操作 O(1)。最常用。

**W-TinyLFU**：窗口 LRU（最近访问）+ 主缓存（频繁访问），通过 Count-Min Sketch 估计频率。命中率最高，Caffeine 采用。

### 故障处理

**故障检测**：Gossip 协议心跳，5 秒怀疑，15 秒确认。

**故障转移**：15-30 秒完成。副本选举 → 提升为主节点 → 集群配置传播 → 客户端发现新拓扑。

**数据持久性**：纯内存（最快，可能丢数据）→ RDB 快照（定期快照）→ AOF（记录每个写操作）→ RDB + AOF（混合方案）。

### 扩展策略

- **水平扩展**：添加节点，在线迁移槽位，SDK 自动发现
- **自动扩缩容**：监控内存、QPS、延迟，超过阈值自动添加/移除节点
- **多区域部署**：Active-Active，跨区域异步复制，读本地写异步

### 权衡决策

| 决策 | 选择 | 原因 |
|------|------|------|
| 一致性 | 可配置 | 按场景选择 |
| 复制 | 异步 | 性能优先 |
| 淘汰策略 | LRU | 简单通用 |
| 持久性 | 可配置 | 灵活性 |
| 分区 | 哈希槽 | 分布均匀 |
| 故障转移 | 自动 | 高可用 |
| 冲突解决 | LWW | 简单有效 |
| 序列化 | MessagePack | 紧凑高效 |
