# Phase 6 — Performance & System Design

## Goal

個別技術の知識から一段進み、  
**RequirementとScaleを見てWeb System全体を設計し、BottleneckやFailureを予測できる状態**を目指す。

---

## Step 1 — Requirement Clarification

設計前に確認する。

```text
誰が使うか？
Core Functionは何か？
Consistencyはどこまで必要か？
許容Latencyは？
Data Retentionは？
```

---

## Step 2 — Capacity Estimation

- DAU / MAU
- Requests/sec
- Peak Traffic
- Rows/day
- Storage/year
- Bandwidth

正解よりもAssumptionと計算過程を重視する。

---

## Step 3 — API / Data Model Design

Large-scale Issue Trackerとして再設計する。

- API Boundary
- Table
- Index
- Ownership
- Audit
- Event

---

## Step 4 — Read / Write Path

```text
Client
→ CDN
→ API
→ Cache
→ DB
→ Queue
```

Read-heavy / Write-heavyで設計がどう変わるか確認する。

---

## Step 5 — Horizontal Scaling

- Stateless Application
- Load Balancer
- Session Affinity
- Shared State

---

## Step 6 — Database Replication

- Primary
- Replica
- Replication Lag
- Read-after-write Consistency

---

## Step 7 — Partitioning / Sharding

候補:

```text
owner_id
organization_id
time
```

Trade-off:

- Cross-shard Query
- Rebalance
- Hotspot
- Operational Complexity

---

## Step 8 — Cache Strategy

- What to Cache
- Where to Cache
- TTL
- Invalidation
- Stale Tolerance
- Cache Stampede

---

## Step 9 — Queue Strategy

```text
Sync or Async?
Orderingは必要か？
Retry可能か？
Duplicateは許容できるか？
Delayは許容できるか？
```

---

## Step 10 — Failure Design

Dependencyごとに書く。

```text
DB unavailable
Redis unavailable
Kafka unavailable
Worker overload
Network partition
Deployment failure
```

各Failureについて:

```text
Detection
User Impact
Fallback
Recovery
```

---

## Step 11 — Frontend Rendering Strategy

Next.jsで画面ごとに選択する。

- CSR
- SSR
- RSC
- Static Rendering

選択理由を記録する。

---

## Step 12 — Hydration / Rendering Performance

測定:

- JavaScript Bundle
- Hydration Cost
- Re-render
- Long Task

Browser Performance Profilerを使用する。

---

## Step 13 — Core Web Vitals

- LCP
- INP
- CLS

改善前後を測定する。

---

## Step 14 — Bundle / Network

- Code Splitting
- Lazy Loading
- Duplicate Dependency
- Network Waterfall
- Image / Font Cost
- Performance Budget

---

## Step 15 — Load Test

```text
Load
→ 想定負荷

Stress
→ 限界を探す

Spike
→ 急増

Soak
→ 長時間
```

測定:

- p50 / p95 / p99
- Throughput
- Error Rate
- CPU
- Memory
- DB
- Queue Lag

---

## Step 16 — Bottleneck Analysis

Load Test結果をEvidenceとして分析する。

```text
Application CPU?
DB Query?
Connection Pool?
Lock?
Redis?
Network?
Frontend?
```

---

## Step 17 — Large-scale Issue Tracker Design

最終Design Document:

```text
Requirements
Capacity
Architecture
API
Database
Index
Cache
Queue
Security
Observability
Failure
Scaling
Trade-offs
```

---

## Step 18 — System Design Practice

- Notification System
- File Upload Service
- Chat
- News Feed
- Autocomplete
- Rate Limiter
- Analytics Pipeline

---

## Phase 6 Completion Criteria

どのSystem Design問題でも以下の順序で説明できること。

```text
1. Requirements
2. Constraints / Scale
3. High-level Design
4. Data Model / API
5. Bottleneck
6. Scaling
7. Failure
8. Security
9. Observability
10. Trade-offs
```

特定の「正解」を暗記するのではなく、  
**なぜそのDesignを選んだのかを説明できること**を最終基準とする。
