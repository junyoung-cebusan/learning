# Phase 5 — Security, Reliability & Observability

## Goal

正常系だけでなく、  
**攻撃・認証切れ・Network Error・Dependency Failureを前提に運用できるSystem**へ進化させる。

---

## Step 1 — Browser Authentication再設計

Phase 1:

```text
JWT
→ localStorage
→ Authorization Header
```

Phase 5:

```text
Access / Refresh Token
HttpOnly Cookie
Secure
SameSite
```

を比較して実装する。

---

## Step 2 — Refresh Token

- Access Token Lifetime
- Refresh Token Lifetime
- Rotation
- Reuse Detection
- Revocation
- Logout

### Testing

- Expiry
- Refresh Success
- Revoked Token
- Reuse

---

## Step 3 — CSRF / XSS

```text
HttpOnly
→ JavaScriptからTokenを直接読ませない

Cookie自動送信
→ CSRFを考慮
```

SameSite / CSRF Token Strategyを学ぶ。

---

## Step 4 — OAuth 2.0 / OpenID Connect

- Authorization Code
- PKCE
- ID Token
- Access Token
- OIDC

---

## Step 5 — Authorization

- Authentication vs Authorization
- RBAC
- ABAC
- Resource Ownership
- GraphQL Field Authorization

Role例:

```text
Admin
Member
Issue Owner
```

---

## Step 6 — GraphQL Security

- Query Depth
- Query Complexity
- Introspection Policy
- Rate Limit
- Field-level Permission

---

## Step 7 — Timeout / Retry

- Connect Timeout
- Read Timeout
- Retryable Error
- Non-retryable Error
- Exponential Backoff
- Jitter

すべてのErrorをRetryしない。

---

## Step 8 — Circuit Breaker

DependencyがDownしたとき、連続RequestでSystem全体が悪化する状況を理解する。

---

## Step 9 — Rate Limiting

対象例:

- Login
- Expensive GraphQL Query
- Public API

Redisを使ったRate Limitを実装する。

---

## Step 10 — Graceful Shutdown / Health Check

- Liveness
- Readiness
- In-flight Request
- Worker Shutdown
- Consumer Offset

---

## Step 11 — Structured Logging

例:

```json
{
  "level": "ERROR",
  "request_id": "...",
  "user_id": 10,
  "operation": "createIssue"
}
```

Sensitive DataはLogへ残さない。

---

## Step 12 — Request / Correlation ID

```text
Next.js
→ GraphQL
→ FastAPI
→ Worker
```

を1つのIDで追跡する。

---

## Step 13 — Metrics

RED:

```text
Rate
Errors
Duration
```

追加:

- DB Connections
- Redis Hit Rate
- Queue Lag
- Worker Failures

---

## Step 14 — Distributed Tracing

```text
HTTP
→ GraphQL Resolver
→ DB
→ Kafka
→ Worker
```

- Trace
- Span
- Parent / Child
- Context Propagation

---

## Step 15 — Dashboard / Alert

Dashboard:

- Throughput
- p95 / p99 Latency
- Error Rate
- DB Connections
- Queue Lag

AlertはInfrastructure数値だけでなくUser Impactと結びつける。

---

## Step 16 — SLI / SLO

例:

```text
SLI
Issue Query Success Rate

SLO
99.9% / 30 days
```

Error Budgetまで扱う。

---

## Step 17 — Frontend Reliability

- Error Boundary
- Retry UX
- Timeout UX
- Unauthorized UX
- Role-based UI
- Web Vitals
- Error Monitoring

---

## Step 18 — Failure Testing

意図的にFailureを発生させる。

```text
PostgreSQL Down
Redis Down
Kafka Consumer Down
External API Timeout
Expired Token
Unauthorized Request
```

確認:

```text
Userには何が見えるか？
Logには何が残るか？
Metricはどう変化するか？
Traceで原因を追えるか？
Recovery後に正常化するか？
```

---

## Phase 5 Completion Criteria

- HttpOnly Cookieを選ぶ理由
- CSRFとXSSの違い
- Refresh Token Rotationの目的
- RetryしてはいけないRequest
- Logging / Metrics / Tracingの役割の違い
- SLI / SLOの設計方法
