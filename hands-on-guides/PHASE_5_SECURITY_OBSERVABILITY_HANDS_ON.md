# Phase 5 — Security, Reliability & Observability Hands-on Guide

## Goal

Issue Trackerを「動くSystem」から  
**安全に運用・観測・復旧できるSystem**へ進化させる。

---

# Step 1 — HttpOnly Cookieへ移行

Phase 1:

```text
localStorage
→ Bearer Header
```

Phase 5:

```text
HttpOnly Cookie
```

Login Response:

```python
response.set_cookie(
    key="access_token",
    value=token,
    httponly=True,
    secure=True,
    samesite="lax",
)
```

FrontendはToken値を直接読まない。

---

# Step 2 — Refresh Token

Table:

```text
refresh_tokens
- id
- user_id
- token_hash
- expires_at
- revoked_at
```

Flow:

```text
Access Token expired
→ Refresh Request
→ Refresh Token検証
→ 新Access Token
→ Refresh Token Rotation
```

---

# Step 3 — Refresh Token Test

Cases:

```text
valid
expired
revoked
reused
```

pytestでそれぞれErrorを確認する。

---

# Step 4 — CSRF

Cookie Authentication時にCSRFを確認する。

Flow例:

```text
CSRF token
→ Custom Header
→ Server compare
```

SameSiteだけに依存しない設計も理解する。

---

# Step 5 — RBAC

Role:

```python
class Role(str, Enum):
    ADMIN = "ADMIN"
    MEMBER = "MEMBER"
```

Authorization:

```python
def require_admin(
    user: UserModel,
):
    if user.role != "ADMIN":
        raise PermissionError()
```

GraphQL Mutationへ適用する。

---

# Step 6 — Ownership Authorization

```python
if issue.owner_id != current_user.id:
    raise PermissionError(
        "Forbidden"
    )
```

Test:

```text
Owner → update成功
Other User → update失敗
Admin → policyに応じて成功
```

---

# Step 7 — GraphQL Complexity

深すぎるQueryを制限する。

Testで意図的にNested Queryを送る。

```text
正常Query → success
複雑Query → rejected
```

---

# Step 8 — Timeout

外部HTTP Call:

```python
async with httpx.AsyncClient(
    timeout=2.0,
) as client:
    response = await client.get(
        url
    )
```

意図的にSlow Endpointへ接続しTimeoutを確認する。

---

# Step 9 — Retry

Retry対象:

```text
connection reset
temporary 503
```

Retryしない:

```text
400
401
validation error
```

Exponential Backoff + Jitterを実装する。

---

# Step 10 — Circuit Breaker

State:

```text
CLOSED
OPEN
HALF_OPEN
```

外部Serviceを意図的に停止し、連続Failure後にCallを止める。

---

# Step 11 — Rate Limit

Redis:

```text
rate:{user_id}:{minute}
```

Increment + Expire。

Test:

```text
1〜N request → success
N+1 → rejected
```

---

# Step 12 — Structured Logging

```python
logger.info(
    "issue_created",
    extra={
        "issue_id": issue.id,
        "user_id": user.id,
        "request_id": request_id,
    },
)
```

Password / Tokenは絶対にLogしない。

---

# Step 13 — Correlation ID

Middleware:

```python
@app.middleware("http")
async def request_id_middleware(
    request,
    call_next,
):
    request_id = (
        request.headers.get(
            "X-Request-ID"
        )
        or str(uuid.uuid4())
    )

    response = await call_next(
        request
    )

    response.headers[
        "X-Request-ID"
    ] = request_id

    return response
```

Frontend → Backend → Workerまで同じIDを渡す。

---

# Step 14 — Metrics

最低限:

```text
request_count
request_latency
error_count
db_connections
redis_hit_rate
queue_lag
```

Prometheus/OpenTelemetry系Toolを使ってよい。

---

# Step 15 — Distributed Tracing

Trace:

```text
Browser
→ API
→ Resolver
→ DB
→ Kafka
→ Worker
```

Trace IDで1Requestを追跡する。

---

# Step 16 — Dashboard

Dashboard Panels:

```text
RPS
p95 latency
5xx rate
DB connections
Redis hit rate
Kafka lag
```

---

# Step 17 — Alert

Alert例:

```text
5xx > 5% for 5m
p95 > 1s for 10m
queue lag increasing
```

CPUだけではなくUser Impactを見る。

---

# Step 18 — SLI / SLO

Example:

```text
SLI:
successful issue queries / all issue queries

SLO:
99.9% / 30 days
```

Error Budgetを計算する。

---

# Step 19 — Frontend Error Boundary

```tsx
"use client";

export default function Error({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <main>
      <p>
        Something went wrong.
      </p>

      <button
        onClick={reset}
      >
        Retry
      </button>
    </main>
  );
}
```

---

# Step 20 — Failure Injection

順番に止める。

```text
PostgreSQL
Redis
Kafka
Worker
External API
```

各Failureで記録する。

```text
User Impact
HTTP/GraphQL Error
Log
Metric
Trace
Recovery
```

---

# Step 21 — Playwright Security E2E

Scenario:

```text
Login
→ Cookie発行
→ Issue access
→ Logout
→ protected page access
→ loginへredirect
```

別Userで他人のIssueを編集できないこともE2Eで確認する。

---

# Phase 5 Completion

```text
[ ] HttpOnly Cookie
[ ] Refresh Token
[ ] CSRF
[ ] RBAC / Ownership
[ ] Timeout / Retry
[ ] Circuit Breaker
[ ] Rate Limit
[ ] Structured Logging
[ ] Metrics
[ ] Tracing
[ ] Dashboard / Alert
[ ] SLI / SLO
[ ] Failure Test
```
