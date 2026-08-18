# Message Queues

# 消息队列

Asynchronous communication between services for decoupling and reliability.

服务间异步通信，实现解耦和可靠性。

---

## Table of Contents / 目录

- [Why Message Queues / 为什么需要消息队列](#why-message-queues)
- [Core Concepts / 核心概念](#core-concepts)
- [Kafka Deep Dive / Kafka 深入解析](#kafka-deep-dive)
- [RabbitMQ Deep Dive / RabbitMQ 深入解析](#rabbitmq-deep-dive)
- [Event-Driven Architecture / 事件驱动架构](#event-driven-architecture)
- [Message Delivery Guarantees / 消息投递保证](#message-delivery-guarantees)
- [Trade-off Analysis / 权衡分析](#trade-off-analysis)
- [中文版本](#中文版本)

---

## Why Message Queues

### Synchronous vs Asynchronous / 同步与异步

```
Synchronous (Without MQ):
  ┌────────┐  request   ┌─────────┐  HTTP   ┌─────────┐
  │ Client │───────────▶│ Service │────────▶│ Service │
  │        │            │    A    │         │    B    │
  │        │◀───────────│         │◀────────│         │
  └────────┘  response  └─────────┘  result └─────────┘

  - Client waits for full processing
  - Services tightly coupled
  - If B is slow, A is slow
  - If B is down, A fails

Asynchronous (With MQ):
  ┌────────┐  request   ┌─────────┐  publish ┌─────┐ consume ┌─────────┐
  │ Client │───────────▶│ Service │─────────▶│ MQ  │────────▶│ Service │
  │        │            │    A    │          │     │         │    B    │
  │        │◀───────────│         │          └─────┘         │         │
  └────────┘  ack       └─────────┘                          └─────────┘

  - Client gets immediate response
  - Services decoupled
  - A works even if B is slow/down
  - Messages persisted in MQ
```

### Benefits / 优势

| Benefit | Description / 描述 |
|---------|-------------------|
| Decoupling / 解耦 | Services don't need to know about each other |
| Resilience / 弹性 | Downstream failures don't cascade |
| Load Leveling / 削峰 | MQ absorbs traffic spikes |
| Async Processing / 异步 | Long tasks processed in background |
| Scalability / 可扩展 | Add consumers independently |

### Load Leveling / 削峰填谷

```
Traffic Spike:
  QPS: 10,000 ──▶ ┌─────────────┐ ──▶ Consumer: 1,000 QPS
                   │ Message Queue│
                   │  (Buffered) │
                   │  10K→1K/sec │
                   └─────────────┘

  Spike absorbed! Consumer processes at steady rate.
  Messages queued during spike, processed gradually.
```

---

## Core Concepts

### Message Structure / 消息结构

```
┌──────────────────────────────────────────┐
│              Message                      │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │ Headers                          │   │
│  │   - message-id: uuid             │   │
│  │   - timestamp: ISO-8601          │   │
│  │   - content-type: application/json│  │
│  │   - correlation-id: uuid         │   │
│  │   - retry-count: 0               │   │
│  └──────────────────────────────────┘   │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │ Body                             │   │
│  │   {                              │   │
│  │     "event": "order.created",    │   │
│  │     "data": {                    │   │
│  │       "order_id": "123",         │   │
│  │       "user_id": "456",          │   │
│  │       "amount": 99.99            │   │
│  │     }                            │   │
│  │   }                              │   │
│  └──────────────────────────────────┘   │
│                                          │
└──────────────────────────────────────────┘
```

### Producer-Consumer Pattern / 生产者-消费者模式

```
  ┌───────────┐         ┌──────────┐         ┌───────────┐
  │ Producer  │ publish │  Queue   │ consume │ Consumer  │
  │ (Service) │────────▶│ (Buffer) │────────▶│ (Worker)  │
  └───────────┘         └──────────┘         └───────────┘

  1:1  - One producer, one consumer
  1:N  - One producer, many consumers (competing consumers)
  N:1  - Many producers, one consumer (aggregation)
  N:N  - Many producers, many consumers
```

### Pub/Sub Pattern / 发布-订阅模式

```
                    ┌─────────────────┐
                    │     Topic       │
                    │  "order.created"│
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Email    │  │ Inventory│  │ Analytics│
        │ Service  │  │ Service  │  │ Service  │
        └──────────┘  └──────────┘  └──────────┘

  One message → Multiple subscribers
  Each subscriber gets a copy
  Subscribers are independent
```

### Queue vs Topic / 队列与主题

| Feature | Queue | Topic |
|---------|-------|-------|
| Consumption | One consumer per message | All subscribers get copy |
| Pattern | Point-to-point | Pub/Sub |
| Use Case | Task distribution | Event notification |

---

## Kafka Deep Dive

### Kafka Architecture / Kafka 架构

```
┌─────────────────────────────────────────────────────────────┐
│                     Kafka Cluster                            │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Broker 1   │  │   Broker 2   │  │   Broker 3   │     │
│  │              │  │              │  │              │     │
│  │ Partition 0  │  │ Partition 1  │  │ Partition 2  │     │
│  │ (Leader)     │  │ (Leader)     │  │ (Leader)     │     │
│  │ Partition 1  │  │ Partition 2  │  │ Partition 0  │     │
│  │ (Replica)    │  │ (Replica)    │  │ (Replica)    │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         │                 │                 │              │
│         └─────────────────┼─────────────────┘              │
│                           │                                │
│                    ZooKeeper / KRaft                        │
│                    (Metadata, Leader Election)              │
└─────────────────────────────────────────────────────────────┘
```

### Kafka Topics and Partitions / Kafka 主题与分区

```
Topic: "orders"
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  Partition 0        Partition 1        Partition 2         │
│  ┌──┬──┬──┬──┐     ┌──┬──┬──┬──┐     ┌──┬──┬──┬──┐      │
│  │0 │1 │2 │3 │     │0 │1 │2 │3 │     │0 │1 │2 │3 │      │
│  └──┴──┴──┴──┘     └──┴──┴──┴──┘     └──┴──┴──┴──┘      │
│  (Broker 1)         (Broker 2)         (Broker 3)         │
│                                                            │
│  Messages ordered within partition                         │
│  Key-based routing: hash(key) % num_partitions            │
│  No ordering guarantee across partitions                   │
└────────────────────────────────────────────────────────────┘
```

### Kafka Consumer Groups / Kafka 消费者组

```
Consumer Group: "order-processing"
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  Partition 0 ──▶ Consumer 1                                │
│  Partition 1 ──▶ Consumer 2                                │
│  Partition 2 ──▶ Consumer 3                                │
│                                                            │
│  Each partition consumed by exactly one consumer           │
│  in a group (at a time)                                    │
│                                                            │
│  Consumer Group B: "analytics"                             │
│  Partition 0 ──▶ Consumer A  (independent)                 │
│  Partition 1 ──▶ Consumer B                                │
│  Partition 2 ──▶ Consumer C                                │
│                                                            │
│  Different groups consume independently                    │
└────────────────────────────────────────────────────────────┘
```

### Kafka Offset Management / Kafka 偏移量管理

```
Partition 0:
  ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
  │0 │1 │2 │3 │4 │5 │6 │7 │8 │9 │
  └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
                    ▲
                    │
              Consumer offset = 5
              (processed 0-4, will read 5 next)

  Auto-commit: Risk losing messages (commit before processing)
  Manual-commit: Safer (commit after processing)
```

### Kafka vs Traditional MQ / Kafka 与传统 MQ

| Feature | Kafka | Traditional MQ |
|---------|-------|---------------|
| Model | Log-based | Queue-based |
| Message Retention | Time/size-based | Until consumed |
| Replay | Yes (by offset) | No (consumed = gone) |
| Ordering | Per-partition | Per-queue |
| Throughput | Very high (millions/sec) | Moderate |
| Consumer Pattern | Pull | Push or Pull |

---

## RabbitMQ Deep Dive

### RabbitMQ Architecture / RabbitMQ 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    RabbitMQ Broker                            │
│                                                             │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐              │
│  │Producer │────▶│Exchange │────▶│ Queue   │────▶Consumer  │
│  │         │     │         │     │         │              │
│  └─────────┘     └────┬────┘     └─────────┘              │
│                       │                                    │
│              Binding Rules (routing keys)                   │
│                       │                                    │
│              ┌────────┼────────┐                           │
│              ▼        ▼        ▼                           │
│           Queue A  Queue B  Queue C                        │
│              │        │        │                           │
│              ▼        ▼        ▼                           │
│          Consumer Consumer Consumer                         │
└─────────────────────────────────────────────────────────────┘
```

### Exchange Types / 交换机类型

```
1. Direct Exchange (直接交换):
   Producer ──routing_key="order"──▶ Exchange
                                      │
                                      ├──"order"──▶ Queue A ✓
                                      ├──"payment"─▶ Queue B ✗
                                      └──"order"──▶ Queue C ✓

2. Fanout Exchange (扇出交换):
   Producer ──▶ Exchange ──▶ Queue A ✓ (all messages)
                    ├──▶ Queue B ✓
                    └──▶ Queue C ✓

3. Topic Exchange (主题交换):
   Producer ──routing_key="order.created"──▶ Exchange
                                              │
                     "order.*" ──▶ Queue A ✓ (matches)
                     "payment.*"──▶ Queue B ✗ (no match)
                     "#.created"──▶ Queue C ✓ (matches)

4. Headers Exchange (头部交换):
   Routes based on message header attributes
```

### RabbitMQ Message Flow / RabbitMQ 消息流程

```
  1. Producer connects to broker
  2. Producer publishes message to exchange
  3. Exchange routes to queue(s) based on bindings
  4. Message stored in queue (persistent if configured)
  5. Consumer subscribes to queue
  6. Broker pushes message to consumer
  7. Consumer processes and sends ACK
  8. Broker removes message from queue

  ┌────────┐  publish  ┌──────────┐  route   ┌───────┐  push   ┌──────────┐
  │Producer│─────────▶│Exchange │────────▶│ Queue │───────▶│ Consumer │
  └────────┘          └──────────┘         └───────┘        └──────────┘
                                                            │  ACK    │
                                                            └─────────┘
```

---

## Event-Driven Architecture

### Event Sourcing / 事件溯源

```
Instead of storing current state, store all events:

  Traditional:
    User { id: 1, balance: 150, status: "active" }

  Event Sourcing:
    Event 1: UserCreated { id: 1, balance: 200 }
    Event 2: PurchaseMade { amount: 50 }
    Event 3: Current state: balance = 200 - 50 = 150

  ┌──────────────────────────────────────────┐
  │          Event Store                      │
  │                                          │
  │  T1: UserCreated  {id:1, balance:200}    │
  │  T2: PurchaseMade {amount:50}            │
  │  T3: DepositMade  {amount:100}           │
  │                                          │
  │  Current State: balance = 200-50+100=250 │
  │                                          │
  │  Benefits: Full audit trail, time travel │
  └──────────────────────────────────────────┘
```

### CQRS (Command Query Responsibility Segregation) / 命令查询职责分离

```
┌─────────────────────────────────────────────────────────┐
│                    CQRS Pattern                         │
│                                                         │
│  Write Side (Commands)         Read Side (Queries)      │
│  ┌─────────────────┐         ┌─────────────────────┐  │
│  │ Command Handler │         │ Query Handler        │  │
│  │                 │         │                      │  │
│  │ CREATE order    │         │ GET /orders/123      │  │
│  │ UPDATE status   │         │ GET /orders?status=  │  │
│  │ DELETE order    │         │ GET /orders/search   │  │
│  └────────┬────────┘         └──────────┬───────────┘  │
│           │                             │              │
│  ┌────────▼────────┐         ┌──────────▼───────────┐  │
│  │ Write Database  │────────▶│ Read Database        │  │
│  │ (Normalized)    │  sync   │ (Denormalized)       │  │
│  │ PostgreSQL      │         │ Elasticsearch        │  │
│  └─────────────────┘         └──────────────────────┘  │
│                                                         │
│  Optimized for writes           Optimized for reads     │
└─────────────────────────────────────────────────────────┘
```

### Saga Pattern / Saga 模式

```
Distributed Transaction using Sagas:

  Order Saga:
  ┌─────────┐    ┌─────────┐    ┌─────────┐
  │ Create  │───▶│ Reserve │───▶│ Process │
  │ Order   │    │Inventory│    │ Payment │
  └────┬────┘    └────┬────┘    └────┬────┘
       │              │              │
       │         ┌────▼────┐   ┌────▼────┐
       │         │Compensate│  │Compensate│
       │         │ Release  │  │ Refund   │
       │         │Inventory │  │ Payment  │
       │         └─────────┘   └─────────┘
       │              ▲              ▲
       │              │              │
       └──────────────┘              │
           If step 2 fails,         │
           compensate step 1        │
                                    │
       If step 3 fails,            │
       compensate steps 1 & 2 ──────┘
```

---

## Message Delivery Guarantees

### Three Levels / 三个级别

```
1. At-Most-Once (最多一次):
   Producer ──publish──▶ Broker ──deliver──▶ Consumer
     (fire and forget)           (may lose message)

   Message may be lost, never duplicated.
   Use: Metrics, logging where loss acceptable.

2. At-Least-Once (至少一次):
   Producer ──publish──▶ Broker ──deliver──▶ Consumer ──ACK──▶ Broker
     (retry if no ack)        (retry if no ack)

   Message never lost, may be duplicated.
   Use: Most common, idempotent consumers.

3. Exactly-Once (精确一次):
   Producer ──publish(idempotent)──▶ Broker ──deliver──▶ Consumer ──ACK──▶ Broker
                                      (deduplication)    (idempotent processing)

   Message delivered exactly once.
   Use: Financial transactions, critical events.
   Implementation: Hard! Usually at-least-once + idempotency.
```

### Idempotency / 幂等性

```
Idempotent operation: Same result regardless of how many times applied.

  NOT idempotent:
    increment_balance(amount)  // Each call adds more!

  Idempotent:
    set_balance(amount, idempotency_key)
    if idempotency_key already processed:
      return cached result
    else:
      process and store idempotency_key
      return result
```

---

## Trade-off Analysis

### Kafka vs RabbitMQ / Kafka 与 RabbitMQ

| Feature | Kafka | RabbitMQ |
|---------|-------|----------|
| Model | Distributed log | Traditional queue |
| Throughput | Millions/sec | Tens of thousands/sec |
| Latency | Low (ms) | Very low (sub-ms) |
| Ordering | Per-partition | Per-queue |
| Replay | Yes | No |
| Routing | Key-based | Flexible (exchange types) |
| Protocol | Custom binary | AMQP |
| Use Case | Event streaming, big data | Task queues, RPC |
| Complexity | Higher | Lower |

### When to Use Which / 何时使用哪个

```
┌─────────────────────────────────────────────────────────┐
│                   Decision Guide                        │
│                                                         │
│  Need high throughput (>100K msg/sec)?                  │
│  ├── Yes → Kafka                                        │
│  └── No                                                 │
│      │                                                  │
│      ├── Need message replay?                           │
│      │   ├── Yes → Kafka                                │
│      │   └── No                                         │
│      │       │                                           │
│      │       ├── Need complex routing?                  │
│      │       │   ├── Yes → RabbitMQ                     │
│      │       │   └── No                                 │
│      │       │       │                                   │
│      │       │       ├── Need task queue with ACK?      │
│      │       │       │   ├── Yes → RabbitMQ / SQS       │
│      │       │       │   └── No                         │
│      │       │       │       │                           │
│      │       │       │       └── Need event streaming?  │
│      │       │       │           ├── Yes → Kafka        │
│      │       │       │           └── No → RabbitMQ      │
```

---

## 中文版本

### 为什么需要消息队列

消息队列实现服务间的异步通信。同步调用时，客户端必须等待完整处理完成；使用消息队列后，客户端立即得到响应，消息在队列中缓冲，消费者按自己的速率处理。

核心优势：**解耦**（服务无需互相了解）、**弹性**（下游故障不会级联）、**削峰填谷**（吸收流量峰值）、**异步处理**（后台处理长任务）、**可扩展性**（独立扩展消费者）。

### 核心概念

**生产者-消费者模式**：生产者发布消息到队列，消费者从队列消费。支持 1:1、1:N、N:1、N:N 模式。

**发布-订阅模式**：一条消息被所有订阅者接收。每个订阅者获得消息的副本，订阅者之间相互独立。

### Kafka 深入解析

Kafka 是分布式日志系统。**主题（Topic）**分为多个**分区（Partition）**，每个分区内消息有序。消息通过 `hash(key) % num_partitions` 路由到特定分区。

**消费者组**：同一组内每个分区只被一个消费者消费。不同组独立消费。

**偏移量管理**：消费者维护消费位置（offset）。自动提交可能丢消息，手动提交更安全。

### RabbitMQ 深入解析

RabbitMQ 是传统消息队列。消息从生产者发送到**交换机（Exchange）**，交换机根据**绑定规则（Binding）**路由到**队列（Queue）**，再推送给消费者。

交换机类型：直接（按 routing key 精确匹配）、扇出（广播到所有队列）、主题（按通配符匹配）、头部（按消息头属性匹配）。

### 事件驱动架构

**事件溯源**：不存储当前状态，而是存储所有事件。优点：完整审计日志、可时间旅行、可重建任意时刻状态。

**CQRS**：命令（写）和查询（读）分离。写端优化写入（规范化数据库），读端优化读取（反规范化数据库），通过同步机制保持一致。

**Saga 模式**：分布式事务解决方案。将长事务拆分为多个本地事务，每个步骤有补偿操作。失败时按逆序执行补偿。

### 消息投递保证

| 级别 | 描述 | 场景 |
|------|------|------|
| 最多一次 | 消息可能丢失，不会重复 | 指标、日志 |
| 至少一次 | 消息不会丢失，可能重复 | 最常用，配合幂等性 |
| 精确一次 | 消息恰好投递一次 | 金融交易、关键事件 |

**幂等性**：同一操作执行多次结果相同。实现精确一次投递通常依赖至少一次投递 + 幂等性。

### Kafka 与 RabbitMQ 对比

| 特性 | Kafka | RabbitMQ |
|------|-------|----------|
| 模型 | 分布式日志 | 传统队列 |
| 吞吐量 | 百万/秒 | 数万/秒 |
| 延迟 | 低（毫秒级） | 极低（亚毫秒级） |
| 消息重放 | 支持 | 不支持 |
| 路由能力 | 基于 Key | 灵活（多种交换机） |
| 适用场景 | 事件流、大数据 | 任务队列、RPC |
