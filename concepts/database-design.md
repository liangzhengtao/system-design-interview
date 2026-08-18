# Database Design

# 数据库设计

Choosing the right database and designing schemas for scale.

选择合适的数据库并设计可扩展的 schema。

---

## Table of Contents / 目录

- [SQL vs NoSQL / SQL 与 NoSQL 对比](#sql-vs-nosql)
- [Database Types / 数据库类型](#database-types)
- [Indexing / 索引](#indexing)
- [Sharding / 分片](#sharding)
- [Replication / 复制](#replication)
- [ACID vs BASE / ACID 与 BASE](#acid-vs-base)
- [Normalization vs Denormalization / 规范化与反规范化](#normalization-vs-denormalization)
- [Database Selection Guide / 数据库选择指南](#database-selection-guide)
- [中文版本](#中文版本)

---

## SQL vs NoSQL

### Side-by-Side Comparison / 并排对比

```
┌─────────────────────────┬─────────────────────────┐
│         SQL             │         NoSQL           │
│   (Relational)          │   (Non-relational)      │
├─────────────────────────┼─────────────────────────┤
│                         │                         │
│  ┌───┬───┬───┐         │  {                      │
│  │ id│name│age│         │    "id": 1,             │
│  ├───┼───┼───┤         │    "name": "John",      │
│  │ 1 │John│ 30│         │    "age": 30,           │
│  │ 2 │Jane│ 25│         │    "address": {         │
│  └───┴───┴───┘         │      "city": "NYC"      │
│                         │    }                    │
│  Fixed schema           │  }                      │
│  Tables with rows       │  Flexible schema        │
│  JOIN queries           │  Documents/KV/Graph     │
│  ACID transactions      │  BASE transactions      │
│  Vertical scaling       │  Horizontal scaling     │
│                         │                         │
└─────────────────────────┴─────────────────────────┘
```

### Detailed Comparison / 详细对比

| Feature / 特性 | SQL | NoSQL |
|---------------|-----|-------|
| Schema / 模式 | Fixed, predefined | Dynamic, flexible |
| Scaling / 扩展 | Vertical (sharding hard) | Horizontal (built-in) |
| Query Language | SQL (standardized) | Varies by database |
| Transactions | Strong ACID | BASE (eventual consistency) |
| Relationships | JOINs, foreign keys | Embedded documents, references |
| Best For | Complex queries, transactions | High throughput, flexible data |
| Examples | PostgreSQL, MySQL, Oracle | MongoDB, Cassandra, Redis |

---

## Database Types

### 1. Relational (RDBMS) / 关系型数据库

```
┌────────────────────────────────────────────┐
│              Relational DB                  │
│                                            │
│  ┌──────────┐    ┌──────────┐             │
│  │  Users   │    │  Orders  │             │
│  ├──────────┤    ├──────────┤             │
│  │ id (PK)  │◄──┐│ id (PK)  │             │
│  │ name     │   ││ user_id  │ (FK)       │
│  │ email    │   ││ amount   │             │
│  └──────────┘   ││ date     │             │
│                  │└──────────┘             │
│                  │                         │
│  JOIN users ON users.id = orders.user_id   │
│                                            │
│  Use: PostgreSQL, MySQL, SQL Server, Oracle│
└────────────────────────────────────────────┘
```

**When to use:** Complex relationships, ACID transactions, structured data, complex queries with JOINs.

### 2. Document Store / 文档数据库

```
┌────────────────────────────────────────────┐
│              Document Store                 │
│                                            │
│  {                                         │
│    "_id": "user123",                       │
│    "name": "John Doe",                     │
│    "email": "john@example.com",            │
│    "orders": [                             │
│      {                                     │
│        "order_id": "ord001",               │
│        "items": ["laptop", "mouse"],       │
│        "total": 1299.99                    │
│      }                                     │
│    ],                                      │
│    "address": {                            │
│      "city": "New York",                   │
│      "zip": "10001"                        │
│    }                                       │
│  }                                         │
│                                            │
│  Use: MongoDB, CouchDB, Firestore          │
└────────────────────────────────────────────┘
```

**When to use:** Semi-structured data, nested documents, rapid development, content management.

### 3. Key-Value Store / 键值存储

```
┌────────────────────────────────────────────┐
│              Key-Value Store                │
│                                            │
│  Key            →    Value                 │
│  ─────────────────────────────             │
│  "user:123"     →    "{name:John,...}"     │
│  "session:abc"  →    "{token:xyz,...}"     │
│  "cache:page1"  →    "<html>...</html>"    │
│                                            │
│  O(1) lookup by key                        │
│  No complex queries                        │
│                                            │
│  Use: Redis, DynamoDB, Memcached, etcd     │
└────────────────────────────────────────────┘
```

**When to use:** Caching, session storage, real-time leaderboards, simple lookups.

### 4. Wide-Column Store / 宽列存储

```
┌────────────────────────────────────────────┐
│            Wide-Column Store                │
│                                            │
│  Row Key    │ Column Family 1  │ CF 2      │
│  ───────────┼──────────────────┼────────── │
│  user:1     │ name: John       │ addr: NYC │
│             │ age: 30          │ zip: 10001│
│             │ email: j@x.com   │           │
│  ───────────┼──────────────────┼────────── │
│  user:2     │ name: Jane       │ addr: LA  │
│             │ age: 25          │           │
│                                            │
│  Use: Cassandra, HBase, ScyllaDB           │
└────────────────────────────────────────────┘
```

**When to use:** Time-series data, IoT, high write throughput, distributed systems.

### 5. Graph Database / 图数据库

```
┌────────────────────────────────────────────┐
│              Graph Database                 │
│                                            │
│    (John)───FRIENDS_WITH───▶(Jane)         │
│       │                        │           │
│       │LIKES                   │WORKS_AT   │
│       ▼                        ▼           │
│    (Pizza)                (Google)         │
│                                            │
│  MATCH (p:Person)-[:FRIENDS_WITH]          │
│        ->(f:Person)                        │
│  WHERE p.name = "John"                     │
│  RETURN f                                  │
│                                            │
│  Use: Neo4j, Amazon Neptune, ArangoDB      │
└────────────────────────────────────────────┘
```

**When to use:** Social networks, recommendation engines, fraud detection, knowledge graphs.

---

## Indexing

### What is an Index / 什么是索引

```
Without Index (Full Table Scan):
  ┌───────────────────────────────┐
  │ Table: users (1M rows)        │
  │                               │
  │ SELECT * FROM users           │
  │ WHERE email = 'john@test.com' │
  │                               │
  │ Scan: 1,000,000 rows → slow! │
  └───────────────────────────────┘

With Index:
  ┌───────────────────────────────┐
  │ B-Tree Index on email         │
  │                               │
  │        [m-p]                  │
  │       /     \                 │
  │    [d-l]    [q-z]            │
  │    / \      / \              │
  │  [d-f][g-l][m-o][p-z]       │
  │                               │
  │ Lookup: O(log n) = 20 steps  │
  │ Scan: ~20 rows → fast!       │
  └───────────────────────────────┘
```

### B-Tree Index Structure / B-树索引结构

```
                    ┌───────────────────┐
                    │    [M]            │  Root
                    │   /    \          │
                    │  /      \         │
           ┌───────▼──┐    ┌──▼───────┐
           │ [D, H]   │    │ [Q, T]   │  Internal
           │ / | \    │    │ / | \    │
           │/  |  \   │    │/  |  \   │
          ┌▼┐┌▼┐┌▼┐  ┌▼┐┌▼┐┌▼┐
          │A││E││I│  │N││R││U│  Leaf (data pointers)
          │B││F││J│  │O││S││V│
          │C││G││K│  │P│  │W│
          └─┘└─┘└─┘  └─┘└─┘└─┘

  Height: O(log n)
  Node size: typically 4KB-16KB
  Fan-out: hundreds of children per node
```

### Index Types / 索引类型

| Type | Structure | Use Case |
|------|-----------|----------|
| B-Tree | Balanced tree | Range queries, sorting |
| Hash | Hash table | Exact match lookups |
| Composite | Multi-column B-Tree | Multi-column queries |
| Full-Text | Inverted index | Text search |
| GiST | Generalized search tree | Geospatial, ranges |
| Partial | B-Tree with WHERE | Subset of rows |

### Index Trade-offs / 索引权衡

```
Write Performance:
  Without index: INSERT O(1)   (append)
  With index:    INSERT O(log n) (update B-Tree)

  More indexes = slower writes

Read Performance:
  Without index: SELECT O(n)   (full scan)
  With index:    SELECT O(log n) (B-Tree lookup)

  More indexes = faster reads

  ┌─────────────────────────────────────┐
  │  Trade-off: Read speed vs Write    │
  │  speed + Storage overhead           │
  └─────────────────────────────────────┘
```

---

## Sharding

### What is Sharding / 什么是分片

```
Single Database (bottleneck at 5K QPS):
  ┌──────────┐
  │ Database │  ← All data, all queries
  │ (1M rows)│
  └──────────┘

Sharded Database (scales to 50K QPS):
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  Shard 0 │  │  Shard 1 │  │  Shard 2 │
  │ (333K)   │  │ (333K)   │  │ (333K)   │
  │ users    │  │ users    │  │ users    │
  │ A-I      │  │ J-R      │  │ S-Z      │
  └──────────┘  └──────────┘  └──────────┘
```

### Sharding Strategies / 分片策略

```
1. Range-Based Sharding (按范围分片):
   ┌─────────────────────────────────────┐
   │ Shard 0: user_id 1      - 333,333  │
   │ Shard 1: user_id 333,334 - 666,666 │
   │ Shard 2: user_id 666,667 - 1,000,000│
   └─────────────────────────────────────┘
   Pros: Range queries efficient
   Cons: Hotspots if data uneven (e.g., recent users)

2. Hash-Based Sharding (按哈希分片):
   shard_id = hash(user_id) % num_shards

   user_id 12345 → hash(12345) % 3 = 0 → Shard 0
   user_id 67890 → hash(67890) % 3 = 2 → Shard 2

   Pros: Even distribution
   Cons: Range queries span all shards

3. Directory-Based Sharding (目录分片):
   ┌─────────────────────┐
   │ Lookup Table        │
   │ A-I  → Shard 0      │
   │ J-R  → Shard 1      │
   │ S-Z  → Shard 2      │
   └─────────────────────┘
   Pros: Flexible, easy resharding
   Cons: Single point of failure (lookup table)
```

### Sharding Challenges / 分片挑战

```
1. Cross-Shard Queries:
   SELECT * FROM orders WHERE user_id = 123
   AND product_id = 456

   If user_id and product_id are on different shards:
   → Must query multiple shards → scatter-gather

2. Resharding:
   Adding Shard 3 requires data redistribution:

   Before: hash(key) % 3
   After:  hash(key) % 4

   ~75% of data needs to move!

   Solution: Consistent hashing (see load-balancing.md)

3. Hotspot (Celebrity Problem):
   Justin Bieber's data → 1000x normal user
   → One shard overloaded

   Solution: Split celebrity data across multiple shards
```

---

## Replication

### Leader-Follower Replication / 主从复制

```
                    Write
                      │
                      ▼
              ┌──────────────┐
              │   Leader     │
              │   (Primary)  │
              └──────┬───────┘
                     │
          ┌──────────┼──────────┐
          │ Replication Log     │
          │ (WAL / binlog)      │
          ▼          ▼          ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ Follower │ │ Follower │ │ Follower │
   │  (Read)  │ │  (Read)  │ │  (Read)  │
   └──────────┘ └──────────┘ └──────────┘

   Writes → Leader only
   Reads  → Any follower (read replicas)
```

### Leader-Leader Replication / 多主复制

```
   ┌──────────┐      ┌──────────┐
   │ Leader A │◄────►│ Leader B │
   │ (DC-East)│      │(DC-West) │
   └────┬─────┘      └─────┬────┘
        │                  │
   ┌────▼─────┐      ┌────▼─────┐
   │Follower A│      │Follower B│
   └──────────┘      └──────────┘

   Writes → Any leader
   Conflict resolution needed!
```

### Replication Lag / 复制延迟

```
Timeline:
  T1: Leader writes x=1
  T2: Follower still has x=0 (replication lag)
  T3: Client reads from follower → gets x=0 (stale!)
  T4: Follower catches up → x=1

  Solutions:
  1. Read-after-write consistency: read from leader after write
  2. Monotonic reads: always read from same follower
  3. Consistent prefix reads: preserve causal order
```

---

## ACID vs BASE

### ACID (SQL Databases)

```
A - Atomicity    All or nothing
    ┌──────────────────────────┐
    │ BEGIN TRANSACTION        │
    │   UPDATE accounts        │
    │   SET balance = balance  │
    │     - 100 WHERE id = 1   │
    │   UPDATE accounts        │
    │   SET balance = balance  │
    │     + 100 WHERE id = 2   │
    │ COMMIT  ← All succeed    │
    │ -- or ROLLBACK ← All fail│
    └──────────────────────────┘

C - Consistency   Data always valid (constraints)
I - Isolation     Concurrent txns don't interfere
D - Durability    Committed data survives crashes
```

### BASE (NoSQL Databases)

```
BA - Basic Availability   System guarantees availability
S  - Soft State          State may change over time
E  - Eventual Consistency All replicas converge eventually

  T=0:  Node A: x=1, Node B: x=0
  T=1:  Node A: x=1, Node B: x=1  (converged)
```

### Comparison / 对比

| Property | ACID | BASE |
|----------|------|------|
| Consistency | Strong | Eventual |
| Availability | Lower | Higher |
| Scaling | Harder | Easier |
| Latency | Higher | Lower |
| Use Case | Banking, inventory | Social media, cache |

---

## Normalization vs Denormalization

### Normalization / 规范化

```
3NF (Third Normal Form):

  Users Table:           Orders Table:
  ┌─────┬──────┐        ┌─────┬─────────┬────────┐
  │ id  │ name │        │ id  │ user_id │ amount │
  ├─────┼──────┤        ├─────┼─────────┼────────┤
  │ 1   │ John │        │ 1   │ 1       │ 100    │
  │ 2   │ Jane │        │ 2   │ 1       │ 200    │
  └─────┴──────┘        │ 3   │ 2       │ 150    │
                         └─────┴─────────┴────────┘

  No data duplication
  JOIN required for queries
```

### Denormalization / 反规范化

```
Denormalized:

  Orders Table:
  ┌─────┬─────────┬────────┬───────────┐
  │ id  │ user_id │ amount │ user_name │
  ├─────┼─────────┼────────┼───────────┤
  │ 1   │ 1       │ 100    │ John      │  ← duplicated
  │ 2   │ 1       │ 200    │ John      │  ← duplicated
  │ 3   │ 2       │ 150    │ Jane      │
  └─────┴─────────┴────────┴───────────┘

  Data duplicated for read speed
  No JOIN needed
  Update anomalies possible
```

---

## Database Selection Guide

```
┌──────────────────────────────────────────────────────────┐
│                  Decision Tree                            │
│                  决策树                                    │
│                                                          │
│  Need ACID transactions?                                 │
│  ├── Yes → PostgreSQL / MySQL                            │
│  └── No                                                  │
│      │                                                   │
│      ├── Flexible schema needed?                         │
│      │   ├── Yes → MongoDB (Document)                    │
│      │   └── No                                          │
│      │       │                                            │
│      │       ├── Key-Value lookups only?                 │
│      │       │   ├── Yes → Redis / DynamoDB              │
│      │       │   └── No                                  │
│      │       │       │                                    │
│      │       │       ├── Time-series data?               │
│      │       │       │   ├── Yes → InfluxDB / TimescaleDB│
│      │       │       │   └── No                          │
│      │       │       │       │                            │
│      │       │       │       ├── Graph relationships?    │
│      │       │       │       │   ├── Yes → Neo4j         │
│      │       │       │       │   └── No                  │
│      │       │       │       │       │                    │
│      │       │       │       │       └── High write      │
│      │       │       │       │           throughput?      │
│      │       │       │       │           ├── Yes →Cassandra│
│      │       │       │       │           └── No →PostgreSQL│
```

---

## 中文版本

### SQL 与 NoSQL 对比

**SQL（关系型数据库）**：固定 schema，支持 ACID 事务和复杂 JOIN 查询，适合结构化数据和强一致性场景。代表：PostgreSQL、MySQL。

**NoSQL（非关系型数据库）**：灵活 schema，水平扩展能力强，适合高吞吐量和半结构化数据。分为文档数据库（MongoDB）、键值存储（Redis）、宽列存储（Cassandra）、图数据库（Neo4j）。

### 索引

索引通过 B-树等数据结构加速查询。无索引时全表扫描 O(n)，有索引时 B-树查找 O(log n)。权衡：更多索引 = 更快读取，但更慢写入和更多存储空间。

索引类型：B-树（范围查询）、哈希（精确匹配）、复合索引（多列查询）、全文索引（文本搜索）、空间索引（地理查询）。

### 分片

分片将数据水平拆分到多个数据库实例。策略：

| 策略 | 描述 | 优缺点 |
|------|------|--------|
| 范围分片 | 按 ID 范围划分 | 范围查询高效，但可能有热点 |
| 哈希分片 | hash(key) % N | 分布均匀，但范围查询需跨片 |
| 目录分片 | 查找表映射 | 灵活，但查找表是单点故障 |

分片挑战：跨片查询、重新分片、热点问题（名人数据）。

### 复制

**主从复制**：一个主节点处理写入，多个从节点处理读取。适合读多写少场景。

**多主复制**：多个主节点都可处理写入。需要冲突解决机制。

**复制延迟**：从节点可能滞后于主节点，导致读到旧数据。解决：读后一致性、单调读、一致前缀读。

### ACID 与 BASE

**ACID**：原子性、一致性、隔离性、持久性。适合银行、库存等强一致性场景。

**BASE**：基本可用、软状态、最终一致性。适合社交媒体、缓存等高可用场景。

### 规范化与反规范化

**规范化**：消除数据冗余，通过 JOIN 查询。写入高效，查询需要 JOIN。

**反规范化**：冗余数据以加速读取。查询高效，但更新异常风险。

### 数据库选择指南

| 场景 | 推荐数据库 |
|------|-----------|
| ACID 事务 | PostgreSQL / MySQL |
| 灵活 schema | MongoDB |
| 键值查找/缓存 | Redis / DynamoDB |
| 时序数据 | InfluxDB / TimescaleDB |
| 图关系 | Neo4j |
| 高写入吞吐 | Cassandra |
| 地理空间 | PostgreSQL + PostGIS |
