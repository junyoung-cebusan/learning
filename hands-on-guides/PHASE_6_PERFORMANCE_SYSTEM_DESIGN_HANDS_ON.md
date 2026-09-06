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
