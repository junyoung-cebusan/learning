# Phase 2 — Scale & Large-Scale Data Processing

## Goal

Phase 1で作成したIssue Trackerをそのまま拡張し、  
**大量データや同時Requestが発生しても、Dataを壊さず効率的に処理できる仕組み**を学ぶ。

このPhaseでは、以下の問いに答えられるようにする。

```text
同じIssueを2人が同時に更新したらどうなるか？
100,000件を一度に処理すると何が問題になるか？
DBが遅いとき、どこを確認すべきか？
同じRequestが2回送られたらどうするか？
HTTP Request内で処理しきれない仕事はどうするか？
```

---

## Step 1 — Transaction

- ACID
- Transaction Boundary
- Commit / Rollback
- `async with session.begin()`

### Practice

```text
Issue作成
+
Activity Log作成
```

どちらか一方が失敗した場合、両方をRollbackする。

### Testing

- pytest
- Rollback Test
- 2つ目のINSERTを意図的に失敗させてAtomicityを確認

---

## Step 2 — Isolation Level / Race Condition

- Read Committed
- Repeatable Read
- Serializable
- Lost Update
- Race Condition

### Practice

```text
User A: OPEN → DONE
User B: OPEN → CANCELED
```

同時に実行し、最終的にどの値が残るかを確認する。

---

## Step 3 — Optimistic / Pessimistic Lock

- Version Column
- Optimistic Concurrency Control
- `SELECT ... FOR UPDATE`
- Pessimistic LockのCost

Issueに`version`を追加し、競合を検出する。

```text
Optimistic Lock
→ 競合が少ない場合に向いている

Pessimistic Lock
→ 競合可能性が高く、強い制御が必要な場合
```

---

## Step 4 — Deadlock

- Deadlockが発生する理由
- Lock Order
- Timeout
- Retry

2つのTransactionが逆順で2つのIssueをLockするScenarioを作り、  
DeadlockまたはLock Waitを再現する。

---

## Step 5 — Idempotency

同じRequestが複数回送られても、結果が1回実行された場合と同じになる仕組みを学ぶ。

```text
Client
→ Idempotency Key
→ API
→ 処理済みか確認
```

同じKeyで2回Requestし、Issueが1件だけ作成されることをTestする。

---

## Step 6 — Index / Query Plan

- B-tree Index
- Composite Index
- Selectivity
- Sequential Scan
- Index Scan
- `EXPLAIN`
- `EXPLAIN ANALYZE`

以下のData量で比較する。

```text
10,000
100,000
1,000,000 Issues
```

対象Query:

```sql
SELECT *
FROM issues
WHERE owner_id = ?
  AND status = ?
ORDER BY created_at DESC
LIMIT 20;
```

候補Index:

```text
(owner_id, status, created_at DESC)
```

---

## Step 7 — Cursor / Keyset Pagination

Phase 1で実装したCursor Paginationを大量Data前提で見直す。

```text
OFFSET Pagination
vs
Keyset Pagination
```

FrontendではInfinite Scrollと組み合わせる。

### Testing

- Duplicate Rowがない
- Missing Rowがない
- Next Cursorが正しい

---

## Step 8 — TanStack Query

ここでServer State管理を導入する。

- Query Key
- Stale Time
- Cache
- Invalidation
- Mutation
- Optimistic Update

Issue CRUDをTanStack Queryベースへ整理する。

---

## Step 9 — Large List / Virtualization

大量のIssueをDOMへ一度にRenderしない。

- Virtualization
- Render Cost
- Infinite Loading

Cursor Pagination + Virtualized Listを組み合わせる。

---

## Step 10 — Batch / Chunk Processing

```text
100,000 Issues
OPEN → ARCHIVED
```

悪い例:

```text
100,000件をすべてMemoryへLoad
```

改善:

```text
1,000件
→ 処理
→ Commit / Checkpoint
→ 次のChunk
```

- Chunk Size
- Memory Usage
- Transaction Size
- Checkpoint
- Retry

---

## Step 11 — Bulk Insert / Bulk Update

ORMで1件ずつ処理する場合とBulk処理を比較する。

```text
1,000
10,000
100,000 rows
```

Performanceだけでなく、ValidationやEvent処理とのTrade-offも確認する。

---

## Step 12 — Streaming / Backpressure

- 全件をMemoryへ載せない理由
- Streaming Result
- Producer / Consumer速度差
- Backpressure

---

## Step 13 — Redis Cache

- Cache Aside
- TTL
- Cache Hit / Miss
- Invalidation
- Stale Data
- Cache Stampede

```text
Request
→ Redis
→ Miss
→ PostgreSQL
→ Redisへ保存
```

---

## Step 14 — Background Job / Queue

長時間処理をHTTP Requestから分離する。

例:

```text
大量Issue Export
```

```text
GraphQL Mutation
→ Job作成
→ Queue
→ Worker
→ 完了状態保存
```

- Producer / Consumer
- Retry
- DLQ
- Idempotent Worker

---

## Step 15 — Kafka

- Topic
- Partition
- Producer
- Consumer
- Consumer Group
- Offset
- Ordering
- At-most-once
- At-least-once
- Exactly-onceの考え方
- Replay

```text
Issue Created
→ Kafka
→ Activity Consumer
→ Activity Log
```

---

## Step 16 — Load Test

測定:

- Throughput
- p50 / p95 / p99 Latency
- Error Rate
- DB Connections

目的は大きな数字を出すことではなく、  
**どこがBottleneckなのかを推測・検証すること**。

---

## Phase 2 Completion Criteria

以下を説明できること。

- Transaction Boundaryをどこに置くか
- Lost Updateがなぜ発生するか
- Optimistic / Pessimistic Lockをどう選ぶか
- Indexが特定Queryに効く理由
- 100,000件をMemory-safeに処理する方法
- Redis Cache Invalidationの考え方
- Queue Consumerを再実行しても安全にする方法
- Kafka PartitionとOrderingの関係
