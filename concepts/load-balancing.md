# Load Balancing

# 负载均衡

Distributing traffic across multiple servers for reliability and performance.

在多台服务器间分配流量以提高可靠性和性能。

---

## Table of Contents / 目录

- [What is Load Balancing / 什么是负载均衡](#what-is-load-balancing)
- [L4 vs L7 Load Balancing / L4 与 L7 负载均衡](#l4-vs-l7-load-balancing)
- [Load Balancing Algorithms / 负载均衡算法](#load-balancing-algorithms)
- [Health Checks / 健康检查](#health-checks)
- [Session Persistence / 会话保持](#session-persistence)
- [Consistent Hashing / 一致性哈希](#consistent-hashing)
- [Load Balancer Types / 负载均衡器类型](#load-balancer-types)
- [Trade-off Analysis / 权衡分析](#trade-off-analysis)
- [中文版本](#中文版本)

---

## What is Load Balancing

```
                        Without Load Balancing:
                        无负载均衡

                    ┌──────────┐
  10,000 req/s ────▶│ Server 1 │  ← Overloaded!
                    └──────────┘

                        With Load Balancing:
                        有负载均衡

                    ┌──────────┐
                    │   Load   │
  10,000 req/s ───▶│ Balancer │
                    └────┬─────┘
              ┌──────────┼──────────┐
              │          │          │
         3,333     3,333      3,333
              │          │          │
         ┌────▼───┐┌────▼───┐┌────▼───┐
         │Server 1││Server 2││Server 3│
         └────────┘└────────┘└────────┘
```

### Benefits / 优势

| Benefit | Description / 描述 |
|---------|-------------------|
| Scalability / 可扩展性 | Add servers to handle more traffic |
| Availability / 可用性 | Traffic routed away from failed servers |
| Performance / 性能 | Distribute load to prevent hotspots |
| Flexibility / 灵活性 | Remove/add servers without downtime |

---

## L4 vs L7 Load Balancing

### L4 (Transport Layer) / L4 传输层

```
┌────────┐                    ┌────────────────┐
│ Client │──── TCP packet ───▶│ L4 Load        │
└────────┘                    │ Balancer       │
                              │                │
                              │ Inspects:      │
                              │  - Source IP    │
                              │  - Dest IP     │
                              │  - Port number │
                              │  - TCP header  │
                              │                │
                              │ Does NOT look  │
                              │ at content     │
                              └───────┬────────┘
                              ┌───────┼───────┐
                              │               │
                         ┌────▼───┐     ┌────▼───┐
                         │Server A│     │Server B│
                         └────────┘     └────────┘
```

**Characteristics:**
- Fast — minimal packet inspection
- Routes based on IP + port
- Cannot route based on content
- Lower latency
- Examples: AWS NLB, HAProxy (TCP mode), Linux IPVS

### L7 (Application Layer) / L7 应用层

```
┌────────┐                    ┌────────────────┐
│ Client │──── HTTP req ─────▶│ L7 Load        │
└────────┘                    │ Balancer       │
                              │                │
                              │ Inspects:      │
                              │  - URL path    │
                              │  - Headers     │
                              │  - Cookies     │
                              │  - HTTP method │
                              │  - Body        │
                              └───────┬────────┘
                        ┌─────────────┼─────────────┐
                        │             │             │
                   /api/users    /api/images   /api/pay
                        │             │             │
                   ┌────▼───┐   ┌────▼───┐   ┌────▼───┐
                   │User Svc│   │Image   │   │Payment │
                   │ Cluster│   │ Cluster│   │ Cluster│
                   └────────┘   └────────┘   └────────┘
```

**Characteristics:**
- Slower — inspects full request
- Content-based routing
- SSL termination
- Can modify requests/responses
- Examples: AWS ALB, Nginx, Envoy, Traefik

### Comparison / 对比

| Feature / 特性 | L4 | L7 |
|---------------|----|----|
| Speed / 速度 | Faster / 快 | Slower / 慢 |
| Routing / 路由 | IP + Port | Content-based |
| SSL Termination | No | Yes |
| Cost / 成本 | Lower / 低 | Higher / 高 |
| Flexibility | Low / 低 | High / 高 |
| Use Case | TCP/UDP, gaming, IoT | HTTP APIs, web apps |

---

## Load Balancing Algorithms

### 1. Round Robin / 轮询

```
Request flow:
  Req 1 ──▶ Server A
  Req 2 ──▶ Server B
  Req 3 ──▶ Server C
  Req 4 ──▶ Server A  (cycle repeats)
  ...

Simple rotation through servers.
```

**Pros:** Simple, no state needed
**Cons:** Ignores server capacity/load

### 2. Weighted Round Robin / 加权轮询

```
Weights: A=5, B=3, C=2  (total=10)

Request flow:
  Req 1-5  ──▶ Server A (50%)
  Req 6-8  ──▶ Server B (30%)
  Req 9-10 ──▶ Server C (20%)
```

**Pros:** Accounts for different server capacities
**Cons:** Weights are static, don't adapt to real-time load

### 3. Least Connections / 最少连接

```
Server A: 10 active connections
Server B: 3  active connections
Server C: 7  active connections

New request ──▶ Server B (fewest connections)
```

**Pros:** Adapts to actual load
**Cons:** Doesn't account for connection duration

### 4. Least Response Time / 最短响应时间

```
Server A: avg response 50ms
Server B: avg response 120ms
Server C: avg response 30ms

New request ──▶ Server C (fastest response)
```

**Pros:** Routes to fastest server
**Cons:** Can create feedback loops

### 5. IP Hash / IP 哈希

```
hash(client_IP) % num_servers = server_index

Client 192.168.1.10 ──hash──▶ index 1 ──▶ Server B
Client 192.168.1.20 ──hash──▶ index 0 ──▶ Server A
Client 192.168.1.10 ──hash──▶ index 1 ──▶ Server B (same client, same server)
```

**Pros:** Session affinity without cookies
**Cons:** Uneven distribution possible, adding/removing servers reshuffles

### 6. Random / 随机

```
Random selection from available servers.

Simple but statistically balanced over time.
```

### Algorithm Comparison / 算法对比

| Algorithm | State | Fairness | Adaptability | Complexity |
|-----------|-------|----------|--------------|------------|
| Round Robin | None | Equal | None | O(1) |
| Weighted RR | Weights | Weighted | None | O(1) |
| Least Conn | Conn count | Good | High | O(n) |
| Least RT | Response time | Good | High | O(n) |
| IP Hash | None | Variable | None | O(1) |
| Random | None | Statistical | None | O(1) |

---

## Health Checks

### Active Health Checks / 主动健康检查

```
┌──────────┐    health check (every 5s)    ┌──────────┐
│   Load   │──────────────────────────────▶│  Server  │
│ Balancer │                               │          │
│          │◀─────── 200 OK ──────────────│ Healthy  │
└──────────┘                               └──────────┘

After 3 failures:
┌──────────┐    health check              ┌──────────┐
│   Load   │──────────────────────────────▶│  Server  │
│ Balancer │                               │          │
│          │◀─────── timeout/error ───────│ Unhealthy│
│          │                               └──────────┘
│          │
│          │    Mark server as DOWN
│          │    Stop routing traffic
└──────────┘
```

### Health Check Configuration / 健康检查配置

```yaml
health_check:
  interval: 5s          # Check every 5 seconds
  timeout: 3s           # Response timeout
  healthy_threshold: 2  # Consecutive successes to mark healthy
  unhealthy_threshold: 3 # Consecutive failures to mark unhealthy
  path: /health         # HTTP endpoint to check
  expected_codes: [200, 204]
```

### Passive Health Checks / 被动健康检查

```
Real traffic monitoring:

  Req 1 ──▶ Server A ──▶ 200 OK     ✓
  Req 2 ──▶ Server B ──▶ 500 Error  ✗ (1 failure)
  Req 3 ──▶ Server B ──▶ 500 Error  ✗ (2 failures)
  Req 4 ──▶ Server B ──▶ Timeout    ✗ (3 failures → mark DOWN)
  Req 5 ──▶ Server A ──▶ 200 OK     ✓
```

---

## Session Persistence

### Why Sessions Matter / 为什么需要会话保持

```
Without session persistence:
  Req 1 (login)    ──▶ Server A ──▶ session created
  Req 2 (checkout) ──▶ Server B ──▶ session NOT found! ✗

With session persistence:
  Req 1 (login)    ──▶ Server A ──▶ session created
  Req 2 (checkout) ──▶ Server A ──▶ session found ✓
```

### Methods / 方法

```
1. Cookie-based (L7):
   Load Balancer inserts cookie: SERVERID=A
   ┌────────┐         ┌──────────┐
   │ Client │◄────────▶│    LB    │
   │ Cookie:│         │ Route by │
   │SERVER=A│         │  cookie  │
   └────────┘         └────┬─────┘
                       Always
                       Server A
                           │
                      ┌────▼───┐
                      │Server A│
                      └────────┘

2. IP Hash:
   hash(client_IP) → always same server

3. Sticky Sessions (L4):
   Connection-level affinity
```

---

## Consistent Hashing

### The Problem / 问题

```
Traditional Hashing: hash(key) % N

  With 3 servers (N=3):
    key1 hash=7 → 7%3=1 → Server 1
    key2 hash=5 → 5%3=2 → Server 2
    key3 hash=8 → 8%3=2 → Server 2

  Problem: Add Server 4 (N=4):
    key1 hash=7 → 7%4=3 → Server 3 (MOVED!)
    key2 hash=5 → 5%4=1 → Server 1 (MOVED!)
    key3 hash=8 → 8%4=0 → Server 4 (MOVED!)

  ~100% of keys need to be remapped!
```

### The Solution: Consistent Hashing / 解决方案：一致性哈希

```
Hash Ring (0 to 2^32 - 1):

              0
              │
     Server A ●───────────● Server B
    (hash=50) │            │ (hash=150)
              │            │
              │   ╭────╮   │
              │   │Ring│   │
              │   ╰────╯   │
              │            │
     Server C ●───────────● 2^32-1
    (hash=250)│            │(hash=340)
              │

Keys mapped to clockwise next server:
  key1 (hash=80)  → Server B (next clockwise)
  key2 (hash=200) → Server C (next clockwise)
  key3 (hash=30)  → Server A (next clockwise)
```

### Virtual Nodes / 虚拟节点

```
Without virtual nodes:
  Small cluster → uneven distribution

With virtual nodes (3 virtual per physical):
  Server A → A1(hash=50), A2(hash=120), A3(hash=310)
  Server B → B1(hash=80), B2(hash=180), B3(hash=350)
  Server C → C1(hash=20), C2(hash=200), C3(hash=290)

  Much more even distribution!
```

### Adding/Removing Servers / 添加/移除服务器

```
Adding Server D (between A and B):

  Before:
    keys at hash 51-150 → Server B

  After:
    keys at hash 51-90  → Server D (NEW)
    keys at hash 91-150 → Server B

  Only ~25% of keys moved (not 100%!)
```

### Consistent Hashing Formula / 一致性哈希公式

```
Key mapping:
  position = hash(key)
  server = first_server_clockwise_from(position)

Virtual nodes:
  for each server S:
    for i in 0..num_virtual_nodes:
      ring.add(hash(S + "-" + i), S)

Lookup:
  pos = hash(key)
  return ring.ceiling(pos)  // first server >= pos on ring
```

---

## Load Balancer Types

### Hardware Load Balancer / 硬件负载均衡器

```
┌─────────────────────────────┐
│   F5 BIG-IP / Citrix ADC    │
│                             │
│   ✓ High performance        │
│   ✓ Dedicated support        │
│   ✗ Expensive ($10K-$100K+) │
│   ✗ Vendor lock-in          │
└─────────────────────────────┘
```

### Software Load Balancer / 软件负载均衡器

```
┌─────────────────────────────┐
│   Nginx / HAProxy / Envoy   │
│                             │
│   ✓ Free / Open source      │
│   ✓ Flexible configuration  │
│   ✓ Runs on commodity HW    │
│   ✗ Requires management     │
└─────────────────────────────┘
```

### Cloud Load Balancer / 云负载均衡器

```
┌─────────────────────────────┐
│   AWS ALB/NLB / GCP LB      │
│   Azure Load Balancer       │
│                             │
│   ✓ Fully managed           │
│   ✓ Auto-scaling            │
│   ✓ Pay per use             │
│   ✗ Vendor lock-in          │
│   ✗ Cost at scale           │
└─────────────────────────────┘
```

---

## Trade-off Analysis

### Choosing a Load Balancer / 选择负载均衡器

```
                    ┌──────────────────────────────┐
                    │     Decision Flowchart        │
                    └──────────────┬───────────────┘
                                   │
                          ┌────────▼────────┐
                          │  Need content-  │
                          │ based routing?  │
                          └────┬───────┬────┘
                           Yes │       │ No
                               │       │
                     ┌─────────▼──┐ ┌──▼──────────┐
                     │    L7      │ │    L4       │
                     │  (Nginx,   │ │ (HAProxy,   │
                     │   Envoy)   │ │  IPVS)      │
                     └────────────┘ └─────────────┘
```

| Scenario | Recommendation |
|----------|---------------|
| Simple web app | Nginx + Round Robin |
| Microservices | Envoy / Istio |
| High throughput, low latency | L4 (HAProxy TCP / NLB) |
| Global traffic | DNS-based + CDN |
| WebSocket / gRPC | L7 with connection draining |
| Cache servers | Consistent hashing |

---

## 中文版本

### 什么是负载均衡

负载均衡是在多台服务器间分配网络流量的技术。没有负载均衡时，单台服务器承受所有请求，容易过载。引入负载均衡后，请求被均匀分配到多台服务器，提高系统的可扩展性、可用性和性能。

### L4 与 L7 负载均衡

**L4 传输层负载均衡**：基于 IP 地址和端口号转发数据包，不检查内容。速度快，延迟低，适用于 TCP/UDP 场景（游戏、IoT）。代表：AWS NLB、HAProxy TCP 模式。

**L7 应用层负载均衡**：基于 HTTP 内容（URL 路径、请求头、Cookie）路由请求。支持 SSL 终止、内容修改。灵活性高但速度较慢。代表：AWS ALB、Nginx、Envoy。

### 负载均衡算法

| 算法 | 描述 | 适用场景 |
|------|------|----------|
| 轮询（Round Robin） | 按顺序轮流分配 | 服务器配置相同 |
| 加权轮询 | 按权重分配 | 服务器配置不同 |
| 最少连接 | 分配给活跃连接最少的服务器 | 长连接场景 |
| 最短响应时间 | 分配给响应最快的服务器 | 性能敏感场景 |
| IP 哈希 | 基于客户端 IP 哈希 | 需要会话保持 |
| 随机 | 随机选择服务器 | 简单场景 |

### 健康检查

**主动健康检查**：负载均衡器定期（如每 5 秒）向服务器发送探测请求。连续 3 次失败后标记为不健康，停止路由流量。

**被动健康检查**：监控真实流量的响应。连续多个请求失败则标记为不健康。

### 一致性哈希

传统哈希 `hash(key) % N` 在服务器数量变化时会导致 ~100% 的键重新映射。一致性哈希通过哈希环解决这个问题：

1. 将服务器和键映射到同一个哈希环上
2. 键分配给顺时针方向最近的服务器
3. 添加/移除服务器时只影响相邻的键（~1/N 移动）
4. 使用虚拟节点确保均匀分布

### 选择建议

| 场景 | 推荐方案 |
|------|----------|
| 简单 Web 应用 | Nginx + 轮询 |
| 微服务架构 | Envoy / Istio |
| 高吞吐低延迟 | L4（HAProxy TCP / NLB） |
| 全球流量分发 | DNS + CDN |
| WebSocket / gRPC | L7 + 连接排空 |
| 缓存服务器集群 | 一致性哈希 |
