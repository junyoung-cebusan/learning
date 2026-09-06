# フルスタックWeb開発ロードマップ

AIによって実装のスピードが大きく変わる今、  
コードを書くこと自体よりも、**Web開発全体の仕組みを理解し、AIを使いながら正しく判断できる力**がより重要になると考えています。

このロードマップでは、1つの **Issue Tracker** を段階的に拡張しながら、

- Frontend
- Backend
- Database
- Infrastructure
- Security
- Performance
- Operations
- Testing

まで、Web開発全体を実践的に学び直します。

---

# Roadmap

| Phase | Theme | Main Topics |
|---|---|---|
| **1** | Full-Stack Foundation | Next.js, GraphQL, FastAPI, PostgreSQL, JWT, Unit / Integration / E2E |
| **2** | Scale & Data Processing | Transaction, Index, Redis, Queue, Kafka, Large-scale Data, Concurrency Test |
| **3** | Architecture | Service / Repository, Modular Monolith, DDD, Frontend Structure, Testability |
| **4** | Infrastructure & Deployment | Docker, AWS, Terraform, CI/CD, Automated Test |
| **5** | Security & Observability | Cookie Auth, OAuth/OIDC, Retry, Logging, Metrics, Tracing, Failure Test |
| **6** | Performance & System Design | Scaling, Partitioning, Failure Design, Web Performance, Load / Stress Test |

```text
Phase 1
動くものを作る
＋基本的にテストする
   ↓
Phase 2
大量データ・同時アクセスでも壊れにくくする
＋Concurrencyを検証する
   ↓
Phase 3
変更しやすく、テストしやすい構造にする
   ↓
Phase 4
実際にデプロイする
＋CI/CDで自動テストする
   ↓
Phase 5
安全に運用し、障害を観測する
＋Security / Failureを検証する
   ↓
Phase 6
システム全体を設計する
＋Load / Stressで限界を確認する
```

---

# Phase 1 — Full-Stack Foundation

## Goal

FrontendからDatabaseまで、一連の流れを自分で実装・テストできるようにする。

### Backend
- FastAPI
- Strawberry GraphQL
- SQLAlchemy Async
- PostgreSQL
- CRUD
- User / Issue Relation
- DataLoader / N+1
- JWT Authentication
- GraphQL Context
- Filter / Search / Cursor Pagination

### Frontend
- Next.js App Router
- TypeScript
- Tailwind CSS
- GraphQL Request
- Login / Logout
- Issue CRUD
- Loading / Error UI

### Frontend Testing
- Vitest
- React Testing Library
- Component Unit Test
- Hook / Utility Test
- Playwright E2E
- Login → Issue CRUD → Logout

### Backend Testing
- pytest
- pytest-asyncio
- httpx
- Service Unit Test
- Async Service Test
- GraphQL API Integration Test
- Authentication Test
- CRUD Integration Test

### Target

```text
Next.js
  ↓
GraphQL
  ↓
FastAPI
  ↓
SQLAlchemy
  ↓
PostgreSQL
```

---

# Phase 2 — Scale & Large-Scale Data Processing

## Goal

大量データや同時アクセスを前提に、安全かつ効率的に処理できるようにする。

### Backend
- Transaction
- Isolation Level
- Lock
- Race Condition
- Deadlock
- Idempotency
- Index / Query Optimization
- EXPLAIN ANALYZE
- Batch / Chunk Processing
- Bulk Insert / Update
- Streaming
- Redis
- Background Job
- Queue
- Kafka

### Frontend
- Cursor Pagination
- Infinite Scroll
- Virtualized List
- Debounce Search
- Request Cancellation
- Optimistic Update
- TanStack Query
- Server State Cache

### Frontend Testing
- Pagination Test
- Infinite Scroll Test
- Optimistic Update Test
- Request Cancellation Test
- Cache Behavior Test

### Backend Testing
- Transaction Test
- Isolation / Lock Behavior Test
- Race Condition Reproduction
- Deadlock Scenario Test
- Idempotency Test
- Large Dataset Test
- Pagination Test
- Redis Cache Test
- Queue / Consumer Retry Test
- Kafka Consumer Test
- Basic Load Test

### Practice

```text
100,000+ Issues
→ 全件取得しない
→ Chunk単位で処理する
→ Cursorで取得する
→ Cache / Queueを使う
```

---

# Phase 3 — Architecture

## Goal

コード量が増えても、責務が明確で変更しやすく、テストしやすい構造にする。

### Backend
- Resolver / Service / Repository
- Dependency Direction
- Modular Monolith
- DDD Basics
- CQRS
- Event-driven Architecture
- Microservices Trade-offs

### Frontend
- Feature-based Structure
- Server / Client Component Boundary
- API Layer
- Domain Types
- Custom Hooks
- Server State / Client State Separation

### Testing
- Service / Repository Separation Test
- Mock / Fake Strategy
- Domain Logic Unit Test
- Architecture Boundary Test
- Frontend Feature Test
- Testabilityを意識したDependency Design

### Target

```text
Resolver
  ↓
Application Service
  ↓
Repository
  ↓
Database
```

---

# Phase 4 — Infrastructure & Deployment

## Goal

Local環境だけでなく、本番を意識した環境へデプロイできるようにする。

### Backend / Infra
- Docker
- Docker Compose
- AWS ECS
- RDS
- ElastiCache
- S3 / CloudFront
- IAM
- Terraform
- GitHub Actions
- CI/CD
- Migration / Rollback

### Frontend
- Production Build
- Environment Management
- CDN / Cache
- Image / Font Optimization
- Bundle Analysis
- Code Splitting
- Staging / Production

### Automated Testing
- Frontend Unit Test in CI
- Backend Unit Test in CI
- GraphQL Integration Test in CI
- Playwright E2E on Staging
- Build Validation
- Migration Test
- Smoke Test
- Deployment Verification
- Rollback Verification

---

# Phase 5 — Security, Reliability & Observability

## Goal

攻撃・障害・ネットワーク異常を前提に、安全に運用できるようにする。

### Security
- HttpOnly / Secure / SameSite Cookie
- Access / Refresh Token
- Token Rotation / Revocation
- CSRF
- OAuth 2.0 / OpenID Connect
- RBAC / ABAC
- GraphQL Authorization
- Query Depth / Complexity Limit

### Reliability
- Timeout
- Retry
- Exponential Backoff
- Circuit Breaker
- Rate Limiting
- Graceful Shutdown
- Health Check

### Observability
- Structured Logging
- Request / Correlation ID
- Metrics
- Distributed Tracing
- Dashboard
- Alerting
- SLI / SLO

### Frontend
- Error Boundary
- Retry / Timeout UX
- Role-based UI
- Refresh Token Flow
- Error Monitoring
- Web Vitals

### Testing
- Authentication / Authorization Test
- CSRF / Cookie Behavior Test
- Token Expiry / Refresh Test
- Permission Test
- Retry / Timeout Test
- Failure Injection
- Rate Limit Test
- Playwright Auth / Permission / Failure Scenario E2E
- Logging / Metrics / Trace Validation

---

# Phase 6 — Performance & System Design

## Goal

個別機能ではなく、Webシステム全体を設計し、限界とボトルネックを検証できるようにする。

### Backend / System
- Requirement Clarification
- Capacity Estimation
- Cache Strategy
- Queue Strategy
- Horizontal Scaling
- Replication
- Partitioning
- Sharding
- Failure Scenario
- Recovery Strategy

### Frontend
- SSR / CSR / RSC Selection
- Hydration Cost
- Rendering Optimization
- Bundle Splitting
- Lazy Loading
- Virtualization
- Core Web Vitals
- Performance Budget

### Testing
- Performance Test
- Load Test
- Stress Test
- Spike Test
- Soak Test
- Bottleneck Analysis
- Failure Scenario Test
- Capacity Verification
- Frontend Performance Measurement

### Practice
- Large-scale Issue Tracker
- Notification System
- File Upload Service
- Chat System
- News Feed
- Analytics Pipeline

---

# Test Stack

## Frontend

```text
Vitest
→ Unit Test

React Testing Library
→ Component Test

Playwright
→ E2E Test
```

## Backend

```text
pytest
→ Unit / Integration Test

pytest-asyncio
→ Async Test

httpx
→ FastAPI / GraphQL Integration Test
```

---

# Learning Principles

実装するときは、毎回以下を確認する。

```text
Why?
なぜこの方法を選ぶのか？

Alternatives?
他の方法はあるか？

Trade-offs?
何を得て、何を失うか？

Scale?
Traffic / Dataが100倍になったらどうなるか？

Failure?
途中で失敗したらどうなるか？

Testing?
どうやって正しさを確認するか？

Observability?
その問題をどうやって検知するか？
```

---

# Final Goal

AIによって開発の進め方が大きく変わる中でも、Web開発全体の仕組みを理解し、AIを単なるコード生成ツールとしてではなく、設計・実装・検証・改善を加速させる手段として活用できるFull-Stack Engineerを目指します。
