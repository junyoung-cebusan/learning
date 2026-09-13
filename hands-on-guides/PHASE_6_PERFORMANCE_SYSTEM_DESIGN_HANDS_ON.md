# Phase 6 — Performance & System Design Hands-on Guide

## Goal

Issue Trackerを題材に、  
**Requirement → Capacity → Architecture → Performance → Failure → Trade-off**まで一貫して設計する。

---

# Step 1 — Requirementsを書き出す

`docs/system-design/requirements.md`

```md
# Functional

- User can create issue
- User can update issue
- User can search issue
- User can receive notification

# Non-functional

- p95 read latency < 300ms
- 99.9% availability
- 1M issues/day
```

---

# Step 2 — Capacity Estimate

仮定:

```text
1M DAU
10 requests/user/day
10M requests/day
```

平均RPS:

```text
10,000,000 / 86,400
≈ 116 RPS
```

Peakを10倍として:

```text
~1,200 RPS
```

Storageも概算する。

---

# Step 3 — High-level Architecture

Diagram:

```text
Browser
  ↓
CDN
  ↓
Frontend
  ↓
Load Balancer
  ↓
API
  ├→ Redis
  ├→ PostgreSQL
  └→ Kafka
       ↓
      Worker
```

---

# Step 4 — Read Pathを分析

```text
Issue List
→ Cache?
→ DB?
→ Replica?
```

各HopのLatencyを計測する。

---

# Step 5 — Write Pathを分析

```text
Create Issue
→ Primary DB
→ Event
→ Async Notification
```

どこまでSyncにするか決める。

---

# Step 6 — Horizontal Scaling Test

LocalでBackend Instanceを複数起動し、Load Balancer経由でRequestを送る。

確認:

```text
SessionをLocal Memoryへ置くと壊れる
StatelessならScaleしやすい
```

---

# Step 7 — Read Replica Design

Concept Practice:

```text
Write → Primary
Read  → Replica
```

Replication Lagがある場合:

```text
Create直後のRead
```

が古い可能性を考える。

---

# Step 8 — Partitioning / Sharding Exercise

候補Key:

```text
organization_id
owner_id
time
```

各候補について表を作る。

```text
Distribution
Hotspot
Cross-shard query
Rebalancing
```

---

# Step 9 — Cache Strategy Document

`docs/system-design/cache.md`

```md
# What to cache
Issue detail

# TTL
60 sec

# Invalidation
On update/delete

# Failure
Fallback to DB
```

---

# Step 10 — Queue Strategy

各処理を分類する。

```text
Create Issue        → Sync
Send Notification   → Async
Generate CSV Export → Async
Audit Log           → Async候補
```

理由を書く。

---

# Step 11 — Frontend Rendering Strategy

各Page:

```text
/login
/issues
/issues/[id]
```

について:

```text
CSR
SSR
RSC
Static
```

どれを選ぶか理由を書く。

---

# Step 12 — Hydration Cost

Chrome Performanceを使う。

Measure:

```text
JS execution
long task
hydration
rerender
```

不必要なClient Componentを減らす。

---

# Step 13 — Bundle Analysis

Next.js Bundle Analyzer等を使ってもよい。

確認:

```text
large dependency
duplicate package
unused client code
```

改善前後を記録する。

---

# Step 14 — Core Web Vitals

確認:

```text
LCP
INP
CLS
```

改善:

```text
large image
blocking JS
layout shift
```

---

# Step 15 — Load Test

Scenario:

```text
80% issue list
15% issue detail
5% create/update
```

Measure:

```text
p50
p95
p99
throughput
error rate
```

---

# Step 16 — Stress Test

徐々に負荷を上げる。

```text
100 RPS
300 RPS
600 RPS
1000 RPS
```

どこでLatency/Errorが急増するか確認する。

---

# Step 17 — Spike Test

短時間だけ急増させる。

```text
100 RPS
→ 1000 RPS
→ 100 RPS
```

Recovery時間を見る。

---

# Step 18 — Soak Test

数十分〜数時間実行する。

確認:

```text
memory leak
connection leak
queue lag accumulation
cache growth
```

---

# Step 19 — Bottleneck Analysis

Evidence:

```text
CPU
Memory
DB Query
Lock Wait
Connection Pool
Redis
Kafka Lag
Network
Frontend Waterfall
```

「たぶんDB」ではなくMetric / Trace / Query Planで証明する。

---

# Step 20 — Failure Design Matrix

`docs/system-design/failure.md`

```md
| Failure | Detection | Impact | Fallback | Recovery |
|---|---|---|---|---|
| DB Down | health/metric | write fail | none/read cache | restore |
| Redis Down | metric | slower read | DB | reconnect |
| Kafka Down | lag/error | delayed event | retry | replay |
```

---

# Step 21 — Final Design Document

`docs/system-design/issue-tracker.md`

必須:

```text
Requirements
Capacity
Architecture
API
Data Model
Indexes
Cache
Queue
Security
Observability
Failure
Scaling
Frontend Strategy
Testing Strategy
Trade-offs
```

---

# Step 22 — Design Review Questions

自分で回答する。

```text
なぜGraphQL?
なぜPostgreSQL?
なぜRedis?
なぜKafka?
なぜこのIndex?
なぜこのPartition Key?
何がSingle Point of Failure?
どこが最初にBottleneckになる?
Trafficが100倍なら?
Dataが100倍なら?
```

---

# Step 23 — Additional System Design Practice

同じTemplateを使って:

```text
Notification System
File Upload Service
Chat
News Feed
Autocomplete
Rate Limiter
Analytics Pipeline
```

を設計する。

---

# Final Completion Criteria

```text
1. Requirementを整理できる
2. Scaleを概算できる
3. Architectureを描ける
4. API/Data Modelを説明できる
5. Bottleneckを測定できる
6. Failureを設計できる
7. Securityを説明できる
8. Observabilityを設計できる
9. Frontend Performanceも説明できる
10. Trade-offを説明できる
```

最終目標は、  
**AIを使って実装を高速化しながらも、AIが生成した設計やCodeを自分で検証し、Web System全体を判断できるFull-Stack Engineerになること。**

---

# Step 24 — gRPCをSystem Designへ組み込む

Phase 3では:

```text
Issue API
→ gRPC
→ User Profile Service
```

という1対1のService Communicationを実装した。

Phase 6ではこれを:

```text
複数Instance
Failure
Load Balancing
Retry
Observability
Streaming
Capacity
```

まで含む分散Systemとして考える。

---

# Step 25 — Deadline Budgetを設計する

Request全体のTimeoutが3秒の場合:

```text
Browser
→ GraphQL API
→ User Service
→ DB
```

各Layerが3秒ずつ待つのは間違い。

例:

```text
Total Budget        3000ms
GraphQL処理          500ms
gRPC User Service   1000ms
DB                   500ms
Buffer              1000ms
```

Upstream Deadlineより
Downstream Deadlineを短く設定する。

確認:

```text
Deadline Propagation
Timeout Budget
Cancellation
```

---

# Step 26 — Retry / Backoff / Idempotency

Retryしてよい処理と、
危険な処理を分ける。

比較:

```text
GetUser
→ Read
→ Retryしやすい

CreatePayment
→ Side Effect
→ 無条件Retryは危険
```

gRPCでは:

```text
UNAVAILABLE
DEADLINE_EXCEEDED
```

等を見ながらRetry Strategyを考える。

学習:

```text
Max Retry Count
Exponential Backoff
Jitter
Retry Storm
Idempotency
```

Retryは「失敗したら全部再実行」ではない。

---

# Step 27 — gRPC Load Balancing / Service Discovery

User Serviceを複数Instanceにする。

```text
Issue API
      ↓
 Service Discovery
      ↓
 ┌────┼────┐
 ↓    ↓    ↓
U1    U2    U3
```

確認:

```text
Client-side Load Balancing
Proxy / Load Balancer
DNS
Health Check
Connection reuse
```

AWSへ展開する場合は
Phase 4で学んだECS / ALB / Service Discoveryとの関係を考える。

---

# Step 28 — Connection / Channel Lifecycle

RequestごとにChannelを作る構造と、

```text
Application Lifecycle
→ Long-lived Channel
```

を比較する。

高Traffic環境では:

```text
Connection reuse
HTTP/2 multiplexing
Channel management
```

が重要。

Phase 3の単純Clientを
Application Lifecycle単位へRefactorするDesignを考える。

---

# Step 29 — gRPC Streaming Design

Streamingが本当に必要なUse Caseを設計する。

例:

```text
Realtime Activity Feed
Progress Stream
Large Result Stream
Agent / Worker Progress
```

比較:

```text
Polling
WebSocket
SSE
GraphQL Subscription
gRPC Streaming
Kafka
```

Browser向けとService-to-Service向けを混同しない。

---

# Step 30 — Backpressure / Slow Consumer

StreamingではProducerがConsumerより速い場合を考える。

```text
Fast Producer
→ Buffer
→ Slow Consumer
```

検討:

```text
Buffer Size
Flow Control
Cancellation
Memory Growth
Drop / Retry Policy
```

「Streamingにしたら速い」で終わらせない。

---

# Step 31 — gRPC Observability

Phase 5のObservabilityをgRPCへ適用する。

Trace:

```text
Browser Request
→ GraphQL Span
→ gRPC Client Span
→ gRPC Server Span
→ PostgreSQL Span
```

記録したいMetric:

```text
RPC Request Count
RPC Error Rate
p50 / p95 / p99 Latency
Deadline Exceeded Count
Retry Count
Active Streams
Message Size
```

Logには:

```text
request_id
trace_id
rpc_method
grpc_status
duration_ms
```

等を含める。

---

# Step 32 — gRPC Load Test

Unary RPCをLoad Testする。

測定:

```text
RPS
p50
p95
p99
Error Rate
CPU
Memory
Connection Count
```

比較:

```text
Single Instance
Multiple Instances

No Retry
Retry Enabled

Small Message
Large Message
```

「Protobufだから速い」と決めつけず、
実測する。

---

# Step 33 — REST / GraphQL / gRPC Benchmarkの読み方

単純なHello World Benchmarkだけで
Protocolを選ばない。

実際には:

```text
Serialization Cost
Network Latency
Payload Size
Business Logic
DB Query
Cache Hit Ratio
Connection Reuse
Concurrency
```

が全体Latencyを決める。

Protocol Benchmarkと
Application Benchmarkを分けて考える。

---

# Step 34 — Failure Scenario

以下を意図的に発生させる。

```text
User Service停止
User Service Slow
Packet / Network Delay
Deadline Exceeded
Partial Instance Failure
Retry増加
Large Payload
Slow Stream Consumer
```

GraphQL APIがどう振る舞うか確認する。

Design Option:

```text
Fail Fast
Fallback
Partial Response
Cached Response
Retry
Circuit Breaker
```

---

# Step 35 — gRPC System Design Review

最終Design Documentへ追加する。

```text
Service Boundary

.proto Contract

Synchronous / Asynchronous Communication

Deadline Budget

Retry Policy

Load Balancing

Service Discovery

Connection Lifecycle

Streaming Use Case

Observability

Failure Strategy

Capacity Estimate
```

最後に説明する:

```text
なぜこのCallはgRPCなのか

なぜこのEventはKafkaなのか

ServiceがDownした時に
User Requestをどう扱うか

RetryがSystemを悪化させる場合

Streamingを選ぶ基準

p99 Latencyをどう改善するか
```

