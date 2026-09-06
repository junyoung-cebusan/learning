# Phase 2 — Scale & Large-Scale Data Processing Hands-on Guide

## Goal

Phase 1のIssue Trackerをそのまま拡張し、  
**Transaction / Concurrency / Large-scale Data / Cache / Queue / Kafka**を実際に壊しながら学ぶ。

---

# Step 1 — Transaction Boundary

## Schema

`activity_logs`を追加する。

```python
class ActivityLogModel(Base):
    __tablename__ = "activity_logs"

    id: Mapped[int] = mapped_column(
        primary_key=True,
    )

    issue_id: Mapped[int] = mapped_column(
        nullable=False,
    )

    action: Mapped[str] = mapped_column(
        String(50),
        nullable=False,
    )
```

## Service

```python
async def create_issue_with_log(
    input: CreateIssueInput,
    owner_id: int,
):
    async with SessionLocal() as session:
        async with session.begin():
            issue = IssueModel(
                title=input.title,
                owner_id=owner_id,
                status="OPEN",
            )

            session.add(issue)

            await session.flush()

            log = ActivityLogModel(
                issue_id=issue.id,
                action="CREATED",
            )

            session.add(log)

        await session.refresh(issue)

        return issue
```

## Failure Reproduction

`action=None`を入れてNotNullViolationを発生させる。

確認:

```sql
SELECT *
FROM issues
ORDER BY id DESC;

SELECT *
FROM activity_logs
ORDER BY id DESC;
```

両方保存されていないことを確認する。

## Test

```python
@pytest.mark.asyncio
async def test_transaction_rolls_back(
    db_session,
):
    with pytest.raises(Exception):
        async with db_session.begin():
            issue = IssueModel(
                title="rollback",
                owner_id=1,
                status="OPEN",
            )
            db_session.add(issue)
            await db_session.flush()

            log = ActivityLogModel(
                issue_id=issue.id,
                action=None,
            )
            db_session.add(log)
```

---

# Step 2 — Race Condition

Issueにcounterを追加する。

```python
view_count: Mapped[int] = mapped_column(
    default=0,
    nullable=False,
)
```

悪い実装:

```python
model = await session.get(
    IssueModel,
    issue_id,
)

model.view_count += 1

await session.commit()
```

同時に100回実行し、100増えない可能性を確認する。

---

# Step 3 — Optimistic Lock

Version追加:

```python
version: Mapped[int] = mapped_column(
    default=1,
    nullable=False,
)
```

Update:

```python
stmt = (
    update(IssueModel)
    .where(
        IssueModel.id == issue_id,
        IssueModel.version
        == current_version,
    )
    .values(
        status=new_status,
        version=current_version + 1,
    )
)

result = await session.execute(stmt)

if result.rowcount == 0:
    raise ValueError(
        "Concurrent update detected"
    )
```

Testで同じversionを使った2回目Updateが失敗することを確認する。

---

# Step 4 — Pessimistic Lock

```python
result = await session.execute(
    select(IssueModel)
    .where(
        IssueModel.id == issue_id
    )
    .with_for_update()
)

issue = result.scalar_one()
```

別Transactionから同じRowをLockし、待機を確認する。

---

# Step 5 — Deadlock

Transaction A:

```text
Issue 1 lock
→ Issue 2 lock
```

Transaction B:

```text
Issue 2 lock
→ Issue 1 lock
```

意図的に同時実行する。

改善:

```text
常にID昇順でLock
```

Retryが必要な理由も確認する。

---

# Step 6 — Idempotency

Table:

```python
class IdempotencyKeyModel(Base):
    __tablename__ = "idempotency_keys"

    key: Mapped[str] = mapped_column(
        String(100),
        primary_key=True,
    )

    issue_id: Mapped[int] = mapped_column(
        nullable=False,
    )
```

Flow:

```text
key検索
→ 既存なら既存Issue返却
→ なければTransaction内でIssue + Key作成
```

同じkeyを2回送ってIssueが1件だけ増えることをTestする。

---

# Step 7 — Seed 100k / 1M Issues

Script:

```python
async def seed(count: int):
    batch_size = 1000

    async with SessionLocal() as session:
        for start in range(
            0,
            count,
            batch_size,
        ):
            batch = [
                IssueModel(
                    title=f"Issue {i}",
                    status="OPEN",
                    owner_id=1,
                )
                for i in range(
                    start,
                    min(
                        start + batch_size,
                        count,
                    ),
                )
            ]

            session.add_all(batch)

            await session.commit()
```

Measure:

```bash
time uv run python -m app.seed 100000
```

---

# Step 8 — EXPLAIN ANALYZE

Query:

```sql
EXPLAIN ANALYZE
SELECT *
FROM issues
WHERE owner_id = 1
  AND status = 'OPEN'
ORDER BY created_at DESC
LIMIT 20;
```

Index:

```sql
CREATE INDEX
idx_issues_owner_status_created
ON issues(
    owner_id,
    status,
    created_at DESC
);
```

前後を比較する。

---

# Step 9 — Keyset Pagination

Query:

```python
stmt = (
    select(IssueModel)
    .where(
        IssueModel.id < cursor
    )
    .order_by(
        IssueModel.id.desc()
    )
    .limit(limit + 1)
)
```

Response:

```text
items
next_cursor
has_next_page
```

OFFSETとの速度差を大量Dataで比較する。

---

# Step 10 — TanStack Query

Install:

```bash
npm install @tanstack/react-query
```

Provider作成。

```tsx
"use client";

import {
  QueryClient,
  QueryClientProvider,
} from "@tanstack/react-query";
import {
  useState,
} from "react";


export function Providers({
  children,
}: {
  children: React.ReactNode;
}) {
  const [client] = useState(
    () => new QueryClient()
  );

  return (
    <QueryClientProvider
      client={client}
    >
      {children}
    </QueryClientProvider>
  );
}
```

Query:

```tsx
const query = useQuery({
  queryKey: [
    "issues",
    filter,
  ],
  queryFn: () =>
    fetchIssues(filter),
});
```

Mutation後:

```tsx
queryClient.invalidateQueries({
  queryKey: ["issues"],
});
```

---

# Step 11 — Infinite Query

```tsx
useInfiniteQuery({
  queryKey: ["issues"],
  initialPageParam: null,
  queryFn: ({ pageParam }) =>
    fetchIssues({
      cursor: pageParam,
    }),
  getNextPageParam: (
    lastPage,
  ) => lastPage.nextCursor,
});
```

---

# Step 12 — Virtualization

Install:

```bash
npm install @tanstack/react-virtual
```

大量RowをDOMに全部描画せず、Viewport周辺だけRenderする。

DevTools PerformanceでDOM数とRender Costを比較する。

---

# Step 13 — Chunk Processing

```python
async def archive_issues():
    cursor = 0
    batch_size = 1000

    while True:
        async with SessionLocal() as session:
            result = await session.execute(
                select(IssueModel)
                .where(
                    IssueModel.id > cursor,
                    IssueModel.status
                    == "OPEN",
                )
                .order_by(
                    IssueModel.id
                )
                .limit(batch_size)
            )

            rows = result.scalars().all()

            if not rows:
                break

            for issue in rows:
                issue.status = "ARCHIVED"

            cursor = rows[-1].id

            await session.commit()
```

Memory使用量を観測する。

---

# Step 14 — Redis Cache

Install:

```bash
uv add redis
```

Client:

```python
from redis.asyncio import Redis


redis = Redis.from_url(
    "redis://localhost:6379",
    decode_responses=True,
)
```

Cache Aside:

```python
async def get_issue_cached(
    issue_id: int,
):
    key = f"issue:{issue_id}"

    cached = await redis.get(key)

    if cached:
        return json.loads(cached)

    issue = await get_issue(
        issue_id
    )

    await redis.set(
        key,
        json.dumps(issue),
        ex=60,
    )

    return issue
```

Update時:

```python
await redis.delete(
    f"issue:{issue_id}"
)
```

Test:

```text
1回目 DB
2回目 Redis
Update
3回目 DB
```

---

# Step 15 — Background Job

まず簡単なQueue Interfaceを作る。

```python
async def enqueue_export(
    issue_ids: list[int],
):
    ...
```

HTTP RequestではJob IDだけ返す。

```text
Mutation
→ job_id
→ Worker
→ status
```

FrontendはPollingでJob Statusを確認する。

---

# Step 16 — Kafka

Local:

```yaml
kafka:
  image: bitnami/kafka:latest
```

Producer:

```python
await producer.send_and_wait(
    "issue-events",
    {
        "type": "ISSUE_CREATED",
        "issue_id": issue.id,
    },
)
```

Consumer:

```python
async for message in consumer:
    event = message.value
    ...
```

必ずDuplicate Eventを想定する。

---

# Step 17 — Load Test

Toolはk6などを利用してよい。

Scenario:

```text
50 users
→ issues query
→ createIssue
```

記録:

```text
p50
p95
p99
throughput
error rate
DB connection count
```

---

# Phase 2 Completion

必ず以下を自分の言葉で説明する。

```text
Transaction Boundary
Lost Update
Lock Trade-off
Index選定理由
Keyset Pagination
Chunk Processing
Cache Invalidation
Idempotent Consumer
Kafka Ordering
```
