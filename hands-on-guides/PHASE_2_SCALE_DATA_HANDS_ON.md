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

# Step 10 — Apollo Client

Phase 2ではApollo ClientをMainのGraphQL Clientとして導入する。

```text
Phase 1
.graphql + Codegen + graphql-request

Phase 2
.graphql + Codegen + Apollo Client
```

`.graphql` FileとCodegenで生成したDocumentはそのまま再利用する。

## Install

```bash
yarn add \
  @apollo/client \
  graphql
```

## Apollo Client

```tsx
"use client";

import {
  ApolloClient,
  ApolloProvider,
  InMemoryCache,
} from "@apollo/client";


const client =
  new ApolloClient({
    uri:
      process.env
        .NEXT_PUBLIC_GRAPHQL_URL,

    cache:
      new InMemoryCache(),
  });


export function Providers({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <ApolloProvider
      client={client}
    >
      {children}
    </ApolloProvider>
  );
}
```

## Query

```tsx
const {
  data,
  loading,
  error,
} = useQuery(
  GetIssuesDocument,
);
```

## Mutation

```tsx
const [
  createIssue,
] = useMutation(
  CreateIssueDocument,
);
```

---

# Step 11 — Apollo Normalized Cache

Apolloの重要な学習ポイントはNormalized Cache。

```text
Issue:1
Issue:2
User:1
```

同じEntityをQuery単位ではなくEntity単位で扱う。

確認する。

```text
Issue Listを取得
→ Issue Detailを取得
→ updateIssueを実行
→ 同じIssue Entityがどう更新されるか
```

Apollo DevToolsでCacheを確認する。

---

# Step 12 — Apollo Cursor Pagination

Phase 2で作ったKeyset PaginationをApolloへ接続する。

学習:

```text
typePolicies
keyArgs
merge
fetchMore
```

例:

```typescript
new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        issues: {
          keyArgs: [
            "status",
            "search",
          ],

          merge(
            existing = {
              items: [],
            },
            incoming,
          ) {
            return {
              ...incoming,
              items: [
                ...existing.items,
                ...incoming.items,
              ],
            };
          },
        },
      },
    },
  },
});
```

---

# Step 13 — Apollo Optimistic Update / Cache Policy

Optimistic Updateを実装する。

```tsx
createIssue({
  variables: {
    input,
  },
  optimisticResponse: {
    createIssue: {
      __typename: "Issue",
      id: -1,
      title: input.title,
      status: "OPEN",
    },
  },
});
```

確認:

```text
Server Response前にUI更新
→ Request失敗
→ Rollback
```

さらに以下を比較する。

```text
cache-first
cache-and-network
network-only
no-cache
```

---

# Step 14 — graphql-request + TanStack Query 比較実習

ApolloをMain実装として残し、同じIssue Listの一部だけを別BranchまたはComparison Componentで実装する。

## Install

```bash
yarn add @tanstack/react-query
```

Phase 1の`graphql-request` Clientを再利用する。

```tsx
const query =
  useQuery({
    queryKey: [
      "issues",
      status,
      search,
    ],

    queryFn: () =>
      getGraphQLClient()
        .request(
          GetIssuesDocument,
          {
            status,
            search,
          },
        ),
  });
```

Mutation後:

```tsx
queryClient.invalidateQueries({
  queryKey: ["issues"],
});
```

比較観点:

| Apollo Client | TanStack Query |
|---|---|
| GraphQL専用 | Protocol非依存 |
| Normalized Cache | Query Key Cache |
| Entity中心 | Query Result中心 |
| GraphQL Cache機能が豊富 | REST / GraphQL両方で使える |

同じ機能について以下を比較する。

```text
Pagination
Optimistic Update
Cache Update
Invalidation
Type Safety
Boilerplate
Debugしやすさ
```

Main ApplicationではApolloを使い続ける。
TanStack Query版は比較学習用とする。

---


# Step 15 — Apollo Client Testing

Apollo ClientのTestでは、Apollo内部実装をTestするのではなく、
**Query / Mutationの結果によってCacheとUIが期待通り変化するか**を確認する。

## Test Tool

Phase 1で導入したVitest / React Testing Libraryをそのまま使う。

```bash
yarn add -D \
  @testing-library/react \
  @testing-library/jest-dom \
  @testing-library/user-event \
  vitest \
  jsdom
```

---

## Apollo Test Client

`src/test/apollo-test-client.ts`

```typescript
import {
  ApolloClient,
  InMemoryCache,
} from "@apollo/client";


export function createTestApolloClient() {
  return new ApolloClient({
    cache: new InMemoryCache(),
  });
}
```

---

## Normalized Cache Test

同じ`Issue` Entityが複数Queryに存在しても、Mutation Responseで同じIDが返れば
同じEntityとして更新されることを確認する。

```typescript
import {
  describe,
  expect,
  test,
} from "vitest";

import {
  InMemoryCache,
} from "@apollo/client";


describe(
  "Apollo normalized cache",
  () => {
    test(
      "updates the same issue entity",
      () => {
        const cache =
          new InMemoryCache();

        cache.writeFragment({
          id: "Issue:1",
          fragment: gql`
            fragment TestIssue on Issue {
              id
              title
              status
            }
          `,
          data: {
            __typename: "Issue",
            id: 1,
            title: "Before",
            status: "OPEN",
          },
        });

        cache.writeFragment({
          id: "Issue:1",
          fragment: gql`
            fragment TestIssueUpdate on Issue {
              id
              title
              status
            }
          `,
          data: {
            __typename: "Issue",
            id: 1,
            title: "After",
            status: "DONE",
          },
        });

        const result =
          cache.readFragment({
            id: "Issue:1",
            fragment: gql`
              fragment ReadIssue on Issue {
                id
                title
                status
              }
            `,
          });

        expect(result).toMatchObject({
          title: "After",
          status: "DONE",
        });
      },
    );
  },
);
```

> 実際のCodegen構成では、Test用Fragmentも`.graphql`へ分離してCodegen対象にしてよい。

---

## Pagination Merge Test

`typePolicies.merge`をPure Functionとして切り出してTestする。

```typescript
export function mergeIssuePages(
  existing = {
    items: [],
  },
  incoming: {
    items: unknown[];
  },
) {
  return {
    ...incoming,
    items: [
      ...existing.items,
      ...incoming.items,
    ],
  };
}
```

```typescript
test(
  "merges cursor pages",
  () => {
    const result =
      mergeIssuePages(
        {
          items: [
            { id: 1 },
          ],
        },
        {
          items: [
            { id: 2 },
          ],
        },
      );

    expect(result.items).toEqual([
      { id: 1 },
      { id: 2 },
    ]);
  },
);
```

確認Point:

```text
Page 1
→ fetchMore
→ Page 2
→ 既存Itemsが消えない
→ Duplicateが発生しない
```

---

## Optimistic Update Test

Component Testでは、Server Responseを待つ前にUIへ仮のIssueが表示されることを確認する。

```tsx
test(
  "shows optimistic issue before server response",
  async () => {
    render(
      <IssuePage />,
      {
        wrapper: ApolloTestProvider,
      },
    );

    const user =
      userEvent.setup();

    await user.type(
      screen.getByPlaceholderText(
        "Title",
      ),
      "Optimistic Issue",
    );

    await user.click(
      screen.getByRole(
        "button",
        {
          name: "Create",
        },
      ),
    );

    expect(
      screen.getByText(
        "Optimistic Issue",
      ),
    ).toBeInTheDocument();
  },
);
```

追加で、Mutation Error時にOptimistic UIがRollbackされることも確認する。

---

# Step 16 — TanStack Query Comparison Testing

TanStack Query版では、ApolloのNormalized Cacheとは違い、
**Query Key / Invalidation / Query Result更新**をTestする。

## Test QueryClient

```typescript
import {
  QueryClient,
} from "@tanstack/react-query";


export function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
      },
      mutations: {
        retry: false,
      },
    },
  });
}
```

---

## Query Key Test

```typescript
test(
  "uses different cache entries for different filters",
  () => {
    const client =
      createTestQueryClient();

    client.setQueryData(
      [
        "issues",
        { status: "OPEN" },
      ],
      ["open-issue"],
    );

    client.setQueryData(
      [
        "issues",
        { status: "DONE" },
      ],
      ["done-issue"],
    );

    expect(
      client.getQueryData([
        "issues",
        { status: "OPEN" },
      ]),
    ).toEqual([
      "open-issue",
    ]);
  },
);
```

---

## Invalidation Test

```typescript
test(
  "invalidates issue queries after mutation",
  async () => {
    const client =
      createTestQueryClient();

    client.setQueryData(
      ["issues"],
      [{ id: 1 }],
    );

    await client.invalidateQueries({
      queryKey: ["issues"],
    });

    const state =
      client.getQueryState([
        "issues",
      ]);

    expect(
      state?.isInvalidated,
    ).toBe(true);
  },
);
```

---

## Optimistic Update Test

TanStack Queryでは`onMutate`でCacheを直接更新する。

```typescript
onMutate: async (
  newIssue,
) => {
  await queryClient.cancelQueries({
    queryKey: ["issues"],
  });

  const previous =
    queryClient.getQueryData(
      ["issues"],
    );

  queryClient.setQueryData(
    ["issues"],
    (old: Issue[] = []) => [
      ...old,
      newIssue,
    ],
  );

  return {
    previous,
  };
}
```

Error時:

```typescript
onError: (
  _error,
  _variables,
  context,
) => {
  queryClient.setQueryData(
    ["issues"],
    context?.previous,
  );
}
```

Testでは:

```text
Mutation開始
→ Cacheへ仮Data追加
→ Error
→ previous CacheへRollback
```

を確認する。

---

## Apollo vs TanStack Query Test観点

| 観点 | Apollo | TanStack Query |
|---|---|---|
| Cache単位 | Entity | Query Key |
| Update確認 | Entity自動反映 | setQueryData / invalidate |
| Pagination | typePolicies / merge | useInfiniteQuery |
| Optimistic | optimisticResponse | onMutate |
| GraphQL理解 | あり | なし |

同じIssue Listを両方でTestし、Cacheの考え方の違いを実際に確認する。

---

# Step 17 — Client Library E2E

E2EではApolloやTanStack QueryそのものをTestしない。

**Userから見た結果**だけを確認する。

Playwright Scenario:

```text
Login
→ Issue List
→ Cursorで次Page取得
→ Create
→ Optimistic UI確認
→ Update
→ Filter変更
→ Delete
→ Logout
```

Example:

```typescript
test(
  "issue list works with client cache",
  async ({ page }) => {
    await page.goto(
      "/login",
    );

    await login(page);

    await page.goto(
      "/issues",
    );

    await page
      .getByPlaceholder(
        "Title",
      )
      .fill(
        "Apollo E2E Issue",
      );

    await page
      .getByRole(
        "button",
        {
          name: "Create",
        },
      )
      .click();

    await expect(
      page.getByText(
        "Apollo E2E Issue",
      ),
    ).toBeVisible();
  },
);
```

Main ApplicationのE2EはApollo版で実行する。
TanStack Query版はComparison Branchで必要な範囲のみ同じScenarioを再実行する。

---

# Step 18 — Large List / Virtualization

# Step 19 — Chunk Processing

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

# Step 20 — Redis Cache

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

# Step 21 — Background Job

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

# Step 22 — Kafka

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

# Step 20 — Load Test

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

# Phase 2 Testing Completion

```text
[ ] Apollo Normalized Cache Test
[ ] Apollo Pagination Merge Test
[ ] Apollo Optimistic Update / Rollback Test
[ ] TanStack Query Query Key Test
[ ] TanStack Query Invalidation Test
[ ] TanStack Query Optimistic Rollback Test
[ ] Playwright Pagination / CRUD E2E
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
