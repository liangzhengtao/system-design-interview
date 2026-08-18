# Design a Chat System

# 设计聊天系统

Design a real-time chat system like WhatsApp or WeChat.

设计一个类似 WhatsApp 或微信的实时聊天系统。

---

## Table of Contents / 目录

- [Requirements / 需求分析](#requirements)
- [Capacity Estimation / 容量估算](#capacity-estimation)
- [High-Level Design / 高层设计](#high-level-design)
- [API Design / API 设计](#api-design)
- [Database Schema / 数据库 Schema](#database-schema)
- [Message Flow / 消息流程](#message-flow)
- [Presence System / 在线状态系统](#presence-system)
- [Group Chat / 群聊](#group-chat)
- [Scaling Strategy / 扩展策略](#scaling-strategy)
- [Trade-offs / 权衡分析](#trade-offs)
- [中文版本](#中文版本)

---

## Requirements

### Functional Requirements / 功能需求

1. One-on-one messaging (1:1 chat)
2. Group messaging (up to 500 members)
3. Online/offline presence indicators
4. Message delivery status (sent, delivered, read)
5. Media sharing (images, videos, files)
6. Push notifications for offline users
7. Message history and search

### Non-Functional Requirements / 非功能需求

1. Real-time delivery (< 200ms for online users)
2. Message ordering guaranteed per conversation
3. High availability (99.99%)
4. Support 50M concurrent connections
5. End-to-end encryption (E2EE)
6. Message persistence (store until explicitly deleted)

---

## Capacity Estimation

### Traffic Estimates / 流量估算

```
Assumptions:
  - 500M total users
  - 50M DAU (Daily Active Users)
  - Each user sends 40 messages/day
  - Each user receives 100 messages/day (including group)

Write QPS:
  50M × 40 / 86400 ≈ 23,000 messages/sec

Peak QPS (2x):
  ~46,000 messages/sec

Read QPS:
  50M × 100 / 86400 ≈ 58,000 messages/sec

Peak Read:
  ~116,000 messages/sec
```

### Storage Estimates / 存储估算

```
Per message:
  - message_id:  16 bytes (UUID)
  - chat_id:      8 bytes
  - sender_id:    8 bytes
  - content:    100 bytes (average text)
  - timestamp:    8 bytes
  - metadata:    20 bytes (type, status)
  - Total:     ~160 bytes

Daily storage:
  50M × 40 × 160 bytes = 320 GB/day

5 years storage:
  320 GB × 365 × 5 = 584 TB
  With replicas (3x): ~1.75 PB
```

### Connection Estimates / 连接估算

```
Concurrent connections:
  50M DAU × 30% online simultaneously = 15M concurrent
  Peak: ~30M concurrent WebSocket connections

Memory per connection:
  ~10KB per connection (connection state + buffers)

Total memory:
  30M × 10KB = 300 GB (across connection servers)
```

---

## High-Level Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Chat System Architecture                         │
│                                                                     │
│  ┌──────────┐          ┌──────────┐          ┌──────────┐         │
│  │  Client  │          │  Client  │          │  Client  │         │
│  │  (User A)│          │  (User B)│          │  (User C)│         │
│  └────┬─────┘          └────┬─────┘          └────┬─────┘         │
│       │                     │                     │                │
│       │ WebSocket           │ WebSocket           │ WebSocket      │
│       ▼                     ▼                     ▼                │
│  ┌─────────────────────────────────────────────────────────┐      │
│  │                    Load Balancer                         │      │
│  │              (Sticky Sessions / IP Hash)                 │      │
│  └────────────────────────┬────────────────────────────────┘      │
│                           │                                        │
│       ┌───────────────────┼───────────────────┐                   │
│       │                   │                   │                    │
│       ▼                   ▼                   ▼                    │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐                │
│  │ Chat     │      │ Chat     │      │ Chat     │                │
│  │ Server 1 │      │ Server 2 │      │ Server 3 │                │
│  │(Users A,D│      │(Users B,E│      │(Users C,F│                │
│  └────┬─────┘      └────┬─────┘      └────┬─────┘                │
│       │                 │                 │                        │
│       └─────────────────┼─────────────────┘                       │
│                         │                                          │
│           ┌─────────────┼─────────────┐                           │
│           │             │             │                            │
│           ▼             ▼             ▼                            │
│     ┌──────────┐  ┌──────────┐  ┌──────────┐                    │
│     │ Message  │  │ Presence │  │  Push    │                    │
│     │ Queue    │  │ Service  │  │ Notif    │                    │
│     │ (Kafka)  │  │ (Redis)  │  │ Service  │                    │
│     └────┬─────┘  └──────────┘  └──────────┘                    │
│          │                                                        │
│          ▼                                                        │
│     ┌──────────┐  ┌──────────┐  ┌──────────┐                    │
│     │ Message  │  │  User    │  │  Media   │                    │
│     │ DB       │  │  DB      │  │  Storage │                    │
│     │(Cassandra│  │(Postgres)│  │  (S3)    │                    │
│     └──────────┘  └──────────┘  └──────────┘                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## API Design

### WebSocket Connection / WebSocket 连接

```
Connection: wss://chat.example.com/ws?token={auth_token}

Events (Client → Server):
  {
    "type": "send_message",
    "payload": {
      "chat_id": "chat_123",
      "content": "Hello!",
      "content_type": "text",
      "client_msg_id": "uuid-123"    // for dedup
    }
  }

Events (Server → Client):
  {
    "type": "new_message",
    "payload": {
      "message_id": "msg_456",
      "chat_id": "chat_123",
      "sender_id": "user_789",
      "content": "Hello!",
      "timestamp": "2025-01-15T10:30:00Z",
      "status": "delivered"
    }
  }
```

### REST APIs / REST API

```http
# Create 1:1 chat
POST /api/chats
{
  "type": "direct",
  "participant_id": "user_456"
}

# Create group chat
POST /api/chats
{
  "type": "group",
  "name": "Team Chat",
  "participant_ids": ["user_456", "user_789"]
}

# Get chat history
GET /api/chats/{chat_id}/messages?before=msg_100&limit=50

# Send message (fallback if WebSocket unavailable)
POST /api/chats/{chat_id}/messages
{
  "content": "Hello!",
  "content_type": "text"
}

# Mark messages as read
POST /api/chats/{chat_id}/read
{
  "last_read_message_id": "msg_100"
}
```

---

## Database Schema

### Users Table / 用户表

```sql
CREATE TABLE users (
    user_id       BIGINT PRIMARY KEY,
    username      VARCHAR(50) UNIQUE NOT NULL,
    phone_number  VARCHAR(20) UNIQUE NOT NULL,
    avatar_url    VARCHAR(500),
    status        VARCHAR(200),
    last_seen     TIMESTAMP,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_phone (phone_number),
    INDEX idx_username (username)
);
```

### Chats Table / 聊天表

```sql
CREATE TABLE chats (
    chat_id       BIGINT PRIMARY KEY,
    chat_type     ENUM('direct', 'group') NOT NULL,
    name          VARCHAR(100),          -- group name
    avatar_url    VARCHAR(500),          -- group avatar
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_message_at TIMESTAMP,
    created_by    BIGINT REFERENCES users(user_id)
);
```

### Chat Participants Table / 聊天参与者表

```sql
CREATE TABLE chat_participants (
    chat_id       BIGINT REFERENCES chats(chat_id),
    user_id       BIGINT REFERENCES users(user_id),
    role          ENUM('admin', 'member') DEFAULT 'member',
    joined_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_read_msg_id BIGINT,
    muted_until   TIMESTAMP,

    PRIMARY KEY (chat_id, user_id),
    INDEX idx_user_id (user_id)
);
```

### Messages Table / 消息表

```sql
-- Partitioned by chat_id for efficient chat history queries
CREATE TABLE messages (
    message_id    BIGINT,               -- Snowflake ID
    chat_id       BIGINT NOT NULL,
    sender_id     BIGINT NOT NULL,
    content       TEXT,
    content_type  ENUM('text','image','video','file','system'),
    media_url     VARCHAR(500),
    reply_to_id   BIGINT,
    status        ENUM('sent','delivered','read') DEFAULT 'sent',
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (chat_id, message_id),
    INDEX idx_created_at (chat_id, created_at)
) PARTITION BY HASH (chat_id) PARTITIONS 256;
```

---

## Message Flow

### 1:1 Chat Message Flow / 单聊消息流程

```
User A sends message to User B:

Step 1: Client A ──WebSocket──▶ Chat Server 1
Step 2: Chat Server 1 stores message in Message DB
Step 3: Chat Server 1 publishes to Kafka topic "messages"
Step 4: Chat Server 1 sends ACK to Client A ("sent")

Step 5: Chat Server 2 (where User B connected) consumes from Kafka
Step 6: Chat Server 2 checks if User B is online

  If Online:
    Step 7a: Push message via WebSocket to Client B
    Step 8a: Client B sends ACK
    Step 9a: Update message status to "delivered"

  If Offline:
    Step 7b: Send push notification via Push Service
    Step 8b: Store in offline message queue
    Step 9b: Deliver when User B comes online

┌────────┐  WS   ┌────────┐  store  ┌────────┐
│Client A│──────▶│Server 1│────────▶│Msg DB  │
│        │◀──────│        │         └────────┘
│  "sent"│  ack  └───┬────┘
└────────┘           │ publish
                     ▼
                 ┌────────┐ consume ┌────────┐  WS   ┌────────┐
                 │ Kafka  │────────▶│Server 2│──────▶│Client B│
                 └────────┘         │        │  ack  │"deliver│
                                    └────────┘◀──────│  ed"   │
                                                     └────────┘
```

### Message Ordering / 消息排序

```
Using Snowflake IDs for global ordering:

  ┌──────────────────────────────────────────────────────┐
  │                   Snowflake ID (64 bits)              │
  │                                                      │
  │  │ 1 bit │  41 bits   │ 10 bits │  12 bits │        │
  │  │ sign  │ timestamp  │  machine│ sequence │        │
  │  │       │  (ms)      │  ID     │          │        │
  │  └───────┴────────────┴─────────┴──────────┘        │
  │                                                      │
  │  - Timestamp ensures chronological ordering          │
  │  - Machine ID ensures uniqueness across servers      │
  │  - Sequence handles multiple msgs per ms             │
  └──────────────────────────────────────────────────────┘

  Message IDs are monotonically increasing → natural ordering
```

---

## Presence System

### Architecture / 架构

```
┌─────────────────────────────────────────────────────────┐
│                  Presence System                         │
│                                                         │
│  User A connects:                                       │
│  1. Chat Server → Presence Service: SET user:A online   │
│  2. Presence Service → Redis: SET user:A TTL=30s       │
│  3. Heartbeat every 10s: EXPIRE user:A 30s             │
│                                                         │
│  User A disconnects:                                    │
│  1. WebSocket close detected                            │
│  2. Presence Service → Redis: DEL user:A               │
│  3. Publish "user:A offline" to subscribers             │
│                                                         │
│  ┌──────────┐    ┌──────────────┐    ┌──────────┐     │
│  │ Chat     │───▶│  Presence    │───▶│  Redis   │     │
│  │ Server   │    │  Service     │    │          │     │
│  └──────────┘    └──────┬───────┘    └──────────┘     │
│                         │                              │
│                    Pub/Sub to                           │
│                    friends                              │
│                         │                              │
│                    ┌────▼─────┐                         │
│                    │ Notify   │                         │
│                    │ Interested│                        │
│                    │ Users    │                         │
│                    └──────────┘                         │
└─────────────────────────────────────────────────────────┘
```

### Presence States / 在线状态

```
States:
  Online    → Active in last 30 seconds
  Away      → No activity for 5 minutes
  Offline   → No activity for 30+ seconds

Redis Implementation:
  Key: presence:{user_id}
  Value: {state: "online", last_active: timestamp}
  TTL: 30 seconds (auto-expires → offline)
```

---

## Group Chat

### Group Message Fan-out / 群消息扇出

```
User A sends message to Group (100 members):

┌────────┐  send   ┌────────┐  store  ┌──────────┐
│Client A│────────▶│Server 1│────────▶│Message DB│
└────────┘         └───┬────┘         └──────────┘
                       │
                       ▼ publish
                   ┌────────┐
                   │ Kafka  │
                   │ topic: │
                   │ group_ │
                   │ 123    │
                   └───┬────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ Server 2 │ │ Server 3 │ │ Server 4 │
    │ (Users   │ │ (Users   │ │ (Users   │
    │  B,C,D)  │ │  E,F,G)  │ │  H,I,J)  │
    └──────────┘ └──────────┘ └──────────┘
          │            │            │
     Push to      Push to      Push to
     online       online       online
     members      members      members

  Large groups (>50 members):
  → Write-ahead to DB first
  → Async fan-out via Kafka
  → Don't wait for all deliveries
```

### Group Size Strategies / 群规模策略

```
Small Group (<50 members):
  - Fan-out on write to all members
  - Each member has copy in their inbox
  - Fast reads

Large Group (50-500 members):
  - Fan-out on read (pull model)
  - Members fetch latest messages on open
  - Slower reads, faster writes

Broadcast Channel (500+ members):
  - Write once, read many
  - Separate read model
  - No real-time push to all
```

---

## Scaling Strategy

### WebSocket Connection Scaling / WebSocket 连接扩展

```
┌─────────────────────────────────────────────────────┐
│           WebSocket Scaling                          │
│                                                     │
│  Each server: 500K-1M concurrent connections        │
│  Target: 30M concurrent connections                  │
│  Required: ~30-60 chat servers                      │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │           Connection Registry (Redis)         │   │
│  │                                             │   │
│  │  user:A → server:1                          │   │
│  │  user:B → server:2                          │   │
│  │  user:C → server:3                          │   │
│  │                                             │   │
│  │  Used for cross-server message routing      │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  Load Balancer: IP Hash or consistent hashing       │
│  to maintain sticky connections                     │
└─────────────────────────────────────────────────────┘
```

### Database Sharding / 数据库分片

```
Shard by chat_id:
  shard = hash(chat_id) % N

  Benefits:
  - All messages for a chat on same shard
  - Chat history query hits single shard
  - Natural data locality

  Challenges:
  - User's chats spread across shards
  - "My inbox" queries need scatter-gather
  - Solution: Inbox cache (Redis)
```

### Message Storage Tiering / 消息存储分层

```
┌──────────────────────────────────────────────────────┐
│            Storage Tiering                             │
│                                                      │
│  Hot (0-30 days):  Cassandra cluster (SSD)           │
│  Warm (30-365 days): Cassandra + compression         │
│  Cold (1-5 years): S3 + Glacier                      │
│                                                      │
│  Query flow:                                         │
│  1. Check hot storage first                          │
│  2. If not found, check warm                         │
│  3. If not found, check cold (async restore)         │
└──────────────────────────────────────────────────────┘
```

---

## Trade-offs

| Decision | Option A | Option B | Chosen |
|----------|----------|----------|--------|
| Protocol | WebSocket | Long Polling | WebSocket (real-time) |
| Message DB | Cassandra | PostgreSQL | Cassandra (write-heavy, time-series) |
| Presence | Redis | Cassandra | Redis (TTL, fast) |
| Group fan-out | Write fan-out | Read fan-out | Hybrid by group size |
| Message ID | UUID | Snowflake | Snowflake (ordered) |
| Encryption | Server E2E | Client E2E | Client E2EE (WhatsApp style) |
| Offline msgs | Push only | Store & forward | Store & forward |

---

## 中文版本

### 需求分析

**功能需求**：一对一聊天、群聊（最多 500 人）、在线/离线状态、消息状态（已发送/已送达/已读）、媒体分享、推送通知、消息历史搜索。

**非功能需求**：实时送达（<200ms）、消息有序、高可用（99.99%）、支持 5000 万并发连接、端到端加密、消息持久化。

### 容量估算

- 写 QPS：~23,000 条消息/秒（峰值 46,000）
- 读 QPS：~58,000 条消息/秒（峰值 116,000）
- 每日消息存储：320 GB
- 5 年存储：~1.75 PB（含 3 副本）
- 并发连接：~3000 万 WebSocket 连接

### 高层设计

客户端通过 WebSocket 连接到聊天服务器。聊天服务器负责消息收发和路由。Kafka 作为消息队列实现异步通信。Redis 存储在线状态和连接映射。Cassandra 存储消息（写入密集型、时序数据）。PostgreSQL 存储用户和聊天元数据。

### 消息流程

1. 用户 A 通过 WebSocket 发送消息到聊天服务器 1
2. 服务器 1 存储消息到数据库，发布到 Kafka
3. 服务器 1 返回"已发送"确认给客户端 A
4. 聊天服务器 2（用户 B 所在）从 Kafka 消费消息
5. 用户 B 在线：通过 WebSocket 推送；离线：发送推送通知

使用 Snowflake ID（64 位：时间戳 + 机器 ID + 序列号）保证消息全局有序。

### 在线状态系统

Redis 存储用户在线状态，TTL 30 秒。心跳每 10 秒续期。连接断开时自动过期为离线。通过 Pub/Sub 通知好友状态变化。

### 群聊

小群（<50 人）：写时扇出，每个成员收件箱都有副本。大群（50-500 人）：读时扇出，成员打开时拉取最新消息。

### 扩展策略

- **WebSocket 扩展**：每服务器 50-100 万连接，30-60 台服务器。Redis 维护用户→服务器映射。
- **数据库分片**：按 chat_id 哈希分片，同一聊天的所有消息在同一分片。
- **存储分层**：热数据（30 天内）SSD Cassandra；温数据（1 年内）压缩存储；冷数据（1-5 年）S3 + Glacier。

### 权衡决策

| 决策 | 选择 | 原因 |
|------|------|------|
| 通信协议 | WebSocket | 实时双向通信 |
| 消息数据库 | Cassandra | 写入密集、时序数据 |
| 在线状态 | Redis | TTL 支持、高速 |
| 群消息扇出 | 混合方案 | 按群大小自适应 |
| 消息 ID | Snowflake | 全局有序 |
| 加密 | 客户端 E2EE | WhatsApp 风格安全 |
