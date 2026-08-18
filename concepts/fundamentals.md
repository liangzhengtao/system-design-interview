# System Design Fundamentals

# 系统设计基础理论

Core principles every system design interview requires.

每个系统设计面试都需要的核心原则。

---

## Table of Contents / 目录

- [Scalability / 可扩展性](#scalability)
- [CAP Theorem / CAP 定理](#cap-theorem)
- [Consistency Models / 一致性模型](#consistency-models)
- [Latency vs Throughput / 延迟与吞吐量](#latency-vs-throughput)
- [Availability / 可用性](#availability)
- [Bottlenecks / 瓶颈分析](#bottlenecks)
- [中文版本](#中文版本)

---

## Scalability

Scalability is the ability of a system to handle increased load by adding resources.

可扩展性是系统通过增加资源来处理更大负载的能力。

### Vertical Scaling (Scale Up) / 垂直扩展

```
┌─────────────────────────────┐
│         Single Server       │
│                             │
│  CPU: 4 → 8 → 16 → 32     │
│  RAM: 16GB → 64GB → 256GB  │
│  SSD: 500GB → 2TB → 8TB    │
│                             │
│  More powerful machine      │
└─────────────────────────────┘
```

**Pros:**
- Simple — no code changes needed
- No distributed system complexity
- Strong consistency guaranteed

**Cons:**
- Hardware limit (single machine ceiling)
- Single point of failure (SPOF)
- Cost grows non-linearly
- Downtime for upgrades

### Horizontal Scaling (Scale Out) / 水平扩展

```
                    ┌──────────┐
                    │   Load   │
                    │ Balancer │
                    └────┬─────┘
               ┌─────────┼─────────┐
               │         │         │
          ┌────▼───┐┌────▼───┐┌────▼───┐
          │Server 1││Server 2││Server 3│
          │        ││        ││        │
          │ CPU: 4 ││ CPU: 4 ││ CPU: 4 │
          │RAM: 16G││RAM: 16G││RAM: 16G│
          └────────┘└────────┘└────────┘

  Add more machines as needed
```

**Pros:**
- Virtually unlimited scaling
- Redundancy built-in
- Cost-effective with commodity hardware
- No downtime for adding nodes

**Cons:**
- Distributed system complexity
- Data consistency challenges
- Network overhead
- Requires load balancing

### Scaling Comparison / 扩展方式对比

| Aspect / 方面 | Vertical / 垂直 | Horizontal / 水平 |
|--------------|----------------|-------------------|
| Complexity / 复杂度 | Low / 低 | High / 高 |
| Cost / 成本 | High / 高 | Moderate / 中 |
| Limit / 上限 | Hardware ceiling | Theoretically unlimited |
| SPOF / 单点故障 | Yes / 是 | No / 否 |
| Downtime / 停机 | Required / 需要 | Not required / 不需要 |
| Data Consistency | Easy / 容易 | Challenging / 有挑战 |

---

## CAP Theorem

The CAP theorem states that a distributed system can guarantee at most **two** of three properties simultaneously.

CAP 定理指出，分布式系统最多只能同时保证三个属性中的**两个**。

```
                    Consistency
                        /\
                       /  \
                      /    \
                     /  CA  \
                    / (RDBMS)\
                   /          \
                  /____________\
           CP    /              \    AP
          /     /    Pick Two    \     \
         /     /                  \     \
        /     /                    \     \
       /_____/______________________\_____\
    Availability                Partition
                                Tolerance
```

### The Three Properties / 三个属性

| Property | Meaning / 含义 | Example / 示例 |
|----------|---------------|---------------|
| **C**onsistency | Every read receives the most recent write | Linearizable reads |
| **A**vailability | Every request receives a (non-error) response | 99.99% uptime |
| **P**artition Tolerance | System works despite network partitions | Cross-datacenter |

### CA Systems (Rare in Practice) / CA 系统

```
┌──────────┐      ┌──────────┐
│  Node A  │◄────►│  Node B  │
└──────────┘      └──────────┘
  Single datacenter, no partitions

Examples: Traditional RDBMS (PostgreSQL, MySQL single-node)
```

### CP Systems / CP 系统

```
┌──────────┐  X  ┌──────────┐
│  Node A  │─────│  Node B  │
└──────────┘     └──────────┘
  Network partition detected
  → Reject writes to maintain consistency

Examples: MongoDB (with majority concern), HBase, Redis Cluster
```

### AP Systems / AP 系统

┌──────────┐  X  ┌──────────┐
│  Node A  │─────│  Node B  │
└──────────┘     └──────────┘
  Network partition detected
  → Accept writes, serve stale data (eventual consistency)

Examples: Cassandra, DynamoDB, CouchDB

### PACELC Theorem (Extended CAP) / PACELC 定理

```
if (Partition) {
    choose between Availability and Consistency
} else {
    choose between Latency and Consistency
}
```

| System | P: A or C | E: L or C |
|--------|-----------|-----------|
| Cassandra | A | L |
| MongoDB | C | C |
| DynamoDB | A | L |
| PostgreSQL | C | C |

---

## Consistency Models

### Strong Consistency / 强一致性

```
Timeline:
  Client A ──write(x=1)──▶ Server
  Client B ──read(x)─────▶ Server ──returns 1──▶ Client B
                            (guaranteed latest value)
```

- Every read returns the most recent write
- Slower — requires coordination between nodes
- Used in: banking, inventory systems, leader-based replication

### Eventual Consistency / 最终一致性

```
Timeline:
  Client A ──write(x=1)──▶ Node 1
                            Node 1 ──propagate──▶ Node 2
                            (takes time T)
  Client B ──read(x)─────▶ Node 2 ──returns old value──▶ Client B
                            (may be stale for time T)

  After time T:
  Client B ──read(x)─────▶ Node 2 ──returns 1──▶ Client B
                            (now consistent)
```

- All replicas converge to the same value eventually
- Faster — no coordination required
- Used in: DNS, social media feeds, CDN

### Causal Consistency / 因果一致性

```
Operations with causal relationship are seen in order:

  Client A: post("Hello")  →  post("World is a reply to Hello")
                │                        │
                ▼                        ▼
  Client B: sees "Hello"   →  sees "World" (always in this order)

  But unrelated operations may be seen in any order:
  Client C: may see "World" before "Hello" if no causal link
```

### Read-Your-Writes Consistency / 读己之写一致性

```
Client A ──write(x=1)──▶ Server
           ──read(x)───▶ Server ──returns 1 (guaranteed)
           ──read(x)───▶ from another replica? → redirect to write node
```

### Consistency Trade-off Summary / 一致性权衡总结

```
Strong ◄────────────────────────────────────────► Eventual
   │                                                   │
   ├─ Higher latency                                 ├─ Lower latency
   ├─ Lower availability                             ├─ Higher availability
   ├─ Easier to reason about                         ├─ Harder to reason about
   └─ Examples: Bank, Inventory                      └─ Examples: DNS, Feed
```

---

## Latency vs Throughput

### Latency / 延迟

Time to complete a single operation.

完成单个操作所需的时间。

```
Operation Timeline:
  |-- Network --|-- Processing --|-- Disk I/O --|
  |   10ms      |     5ms        |    15ms      |
  |<─────────────── 30ms total ────────────────>|
```

**Common Latency Numbers:**
| Operation | Latency / 延迟 |
|-----------|---------------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| RAM reference | 100 ns |
| SSD random read | 150 μs |
| HDD random read | 10 ms |
| Round trip same datacenter | 500 μs |
| Round trip cross-continent | 150 ms |

### Throughput / 吞吐量

Number of operations completed per unit time.

单位时间内完成的操作数量。

```
Throughput = Total Operations / Time Period

  QPS (Queries Per Second)
  ├── Read QPS:  10,000
  ├── Write QPS:  1,000
  └── Total QPS: 11,000

  RPS (Requests Per Second)
  └── HTTP requests handled per second
```

### The Relationship / 关系

```
Throughput
    ▲
    │        ┌─────────────────────
    │       /
    │      /   Saturation Point
    │     /    (bottleneck reached)
    │    /
    │   /
    │  /
    │ /
    │/──────────────────────────────▶ Latency
    Low                              High

Below saturation: latency stays low, throughput grows
At saturation: latency spikes, throughput plateaus
```

---

## Availability

### Availability Percentage / 可用性百分比

| Availability | Downtime / Year | Downtime / Day | Common Name |
|-------------|-----------------|----------------|-------------|
| 99% | 3.65 days | 14.4 min | Two nines |
| 99.9% | 8.77 hours | 1.44 min | Three nines |
| 99.99% | 52.6 min | 8.64 sec | Four nines |
| 99.999% | 5.26 min | 0.864 sec | Five nines |

### High Availability Patterns / 高可用模式

```
Active-Passive (Failover):
┌──────────┐     ┌──────────┐
│  Active  │────►│ Passive  │  (standby, takes over on failure)
│  Primary │     │ Replica  │
└──────────┘     └──────────┘

Active-Active:
┌──────────┐
│ Active 1 │◄──┐
└──────────┘   │
               ├── Load Balancer
┌──────────┐   │
│ Active 2 │◄──┘
└──────────┘
Both serve traffic simultaneously

Redundancy:
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Server 1 │ │ Server 2 │ │ Server 3 │
└──────────┘ └──────────┘ └──────────┘
  N+1 or N+2 redundancy
```

### Availability Formula / 可用性公式

```
For redundant components:

  Serial:    A_total = A1 × A2 × A3
  Parallel:  A_total = 1 - (1-A1) × (1-A2) × (1-A3)

Example - 2 servers in parallel, each 99% available:
  A = 1 - (1-0.99) × (1-0.99) = 1 - 0.0001 = 99.99%
```

---

## Bottlenecks

### Common Bottleneck Patterns / 常见瓶颈模式

```
1. Database Bottleneck (most common)
   ┌────────┐    ┌────────┐    ┌────────────┐
   │ Client │───▶│ Server │───▶│  Database  │ ◄── BOTTLENECK
   └────────┘    └────────┘    │ (slow query│
                               │  + disk)   │
                               └────────────┘

2. Network Bottleneck
   ┌────────┐    ~~~~~~~~~~~~    ┌────────┐
   │ Client │──── bandwidth ────▶│ Server │
   └────────┘    (limited)       └────────┘

3. CPU Bottleneck
   ┌────────┐    ┌────────────┐
   │ Request│───▶│ 100% CPU   │ ◄── Cannot process more
   │ Queue  │    │ Utilization│
   └────────┘    └────────────┘
```

### Back-of-the-Envelope Estimation / 粗略估算

Key numbers to memorize for interviews:

面试需要记住的关键数字：

```
Power of 2:
  2^10 = 1 Thousand    (1K)
  2^20 = 1 Million     (1M)
  2^30 = 1 Billion     (1B)
  2^40 = 1 Trillion    (1T)

Time:
  1 day     = 86,400 seconds ≈ 10^5
  1 month   = 2.6M seconds   ≈ 2.5 × 10^6
  1 year    = 31.5M seconds  ≈ 3 × 10^7

Storage (per day):
  1M requests × 1KB each = 1 GB/day
  1B requests × 1KB each = 1 TB/day

QPS to Daily:
  1 QPS  = ~86,400 requests/day
  1K QPS = ~86.4M requests/day
  1M QPS = ~86.4B requests/day
```

### Capacity Estimation Template / 容量估算模板

```
Step 1: Estimate users
  DAU (Daily Active Users) = ?

Step 2: Estimate operations
  Read QPS = DAU × reads_per_user / 86400
  Write QPS = DAU × writes_per_user / 86400

Step 3: Estimate storage
  Storage = DAU × data_per_user_per_day × 365 × years

Step 4: Estimate bandwidth
  Bandwidth = QPS × avg_response_size

Step 5: Identify bottlenecks
  Compare QPS with typical DB/Cache limits:
  - Single MySQL: ~5K QPS reads
  - Single Redis: ~100K QPS
  - Single Memcached: ~200K QPS
```

---

## Summary / 总结

```
┌─────────────────────────────────────────────────────────┐
│                  Key Takeaways                          │
│                  关键要点                                 │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Start with vertical, scale horizontally when needed │
│     先垂直扩展，需要时再水平扩展                           │
│                                                         │
│  2. CAP: Pick 2, but P is mandatory in distributed      │
│     CAP: 三选二，但分布式系统中 P 是必须的               │
│                                                         │
│  3. Consistency model depends on business requirements  │
│     一致性模型取决于业务需求                               │
│                                                         │
│  4. Optimize for the bottleneck, not everything         │
│     优化瓶颈点，而非所有环节                               │
│                                                         │
│  5. Always estimate capacity before designing           │
│     设计前先估算容量                                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 中文版本

### 可扩展性

可扩展性是系统通过增加资源来处理更大负载的能力。

**垂直扩展（Scale Up）**：升级单台机器的硬件资源——CPU、内存、磁盘。优点是简单，无需代码改动；缺点是存在硬件上限，有单点故障风险，成本增长非线性。

**水平扩展（Scale Out）**：增加更多机器。优点是理论上无上限扩展，自带冗余；缺点是分布式系统复杂度高，数据一致性有挑战，需要负载均衡。

### CAP 定理

分布式系统最多只能同时保证三个属性中的两个：

- **一致性（Consistency）**：每次读取都能获取最新写入
- **可用性（Availability）**：每个请求都能收到非错误响应
- **分区容错（Partition Tolerance）**：网络分区时系统仍能工作

实际系统中，CA 类型很少见（单节点 RDBMS），常见的是 CP（MongoDB、HBase）和 AP（Cassandra、DynamoDB）。

### 一致性模型

| 模型 | 特点 | 适用场景 |
|------|------|----------|
| 强一致性 | 每次读取返回最新值 | 银行、库存系统 |
| 最终一致性 | 所有副本最终收敛 | DNS、社交媒体 |
| 因果一致性 | 有因果关系的操作有序 | 评论回复链 |
| 读己之写 | 能读到自己刚写入的数据 | 用户设置 |

### 延迟与吞吐量

**延迟**：完成单个操作所需时间。关键数字——RAM 访问 ~100ns，SSD 随机读 ~150μs，同数据中心往返 ~500μs，跨洲往返 ~150ms。

**吞吐量**：单位时间内完成的操作数量（QPS/RPS）。低于饱和点时延迟稳定、吞吐增长；达到饱和点时延迟飙升、吞吐停滞。

### 可用性

| 可用性 | 年停机时间 | 等级名称 |
|--------|-----------|---------|
| 99% | 3.65 天 | 两个九 |
| 99.9% | 8.77 小时 | 三个九 |
| 99.99% | 52.6 分钟 | 四个九 |
| 99.999% | 5.26 分钟 | 五个九 |

高可用模式：主备切换（Active-Passive）、双活（Active-Active）、冗余（N+1/N+2）。

### 粗略估算要点

- 1 天 ≈ 86,400 秒 ≈ 10^5
- 1M 请求 × 1KB = 1 GB/天
- 单 MySQL 约 5K 读 QPS，单 Redis 约 100K QPS
- 设计前先估算：DAU → 读写 QPS → 存储量 → 带宽 → 瓶颈分析

### 关键要点

1. 先垂直扩展，需要时再水平扩展
2. CAP 中 P 在分布式系统中必须保证，实际是 CP 或 AP 二选一
3. 一致性模型根据业务需求选择，不盲目追求强一致
4. 找到瓶颈再优化，不要过早优化
5. 设计前必须做容量估算，用数字说话
