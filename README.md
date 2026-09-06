# フルスタックWeb開発ロードマップ

AIによってコードを書くスピードは大きく変わりつつあります。

実装そのものは以前より速くなり、知らない技術でもAIを使えば短時間で形にできるようになりました。  
その一方で、生成されたコードが本当に正しいのか、なぜその設計になっているのか、どこに問題が起きる可能性があるのかを判断するためには、これまで以上に基礎的な理解が重要になると考えています。

この学習の目的は、単に新しいFrameworkやLibraryの使い方を増やすことではありません。

```text
AIにコードを書いてもらえる時代だからこそ、
自分は「なぜそう動くのか」を説明できるようになる。
```

Frontendだけでなく、Backend、Database、Infrastructure、Security、Performance、Operationsまで実際に手を動かしながら確認し、Software Engineeringの基本をもう一度体系的に学び直します。

特に以下を重視します。

- Frameworkの使い方だけでなく、その裏側の仕組みを理解する
- AIが生成したコードをそのまま受け入れず、自分で正しさを判断できるようにする
- 「動いた」で終わらず、なぜ動くのかを説明できるようにする
- 技術選定の理由とTrade-offを考える
- 小規模では見えないConcurrency / Transaction / Performance / Failureを理解する
- 本番環境で発生する問題を想定して設計する
- 障害が起きたときに、観測・原因特定・復旧まで考える

学習では複数の小さなTutorialを次々に作るのではなく、1つの **Issue Tracker** を段階的に拡張していきます。

```text
Simple CRUD
    ↓
Authentication
    ↓
Large-scale Data
    ↓
Architecture
    ↓
Infrastructure
    ↓
Security / Reliability
    ↓
Observability
    ↓
System Design
```

同じApplicationを育てていくことで、それぞれの技術が独立した知識ではなく、実際のSystemの中でどのようにつながっているのかを理解することを目指します。

---

## Goal

最終的には、AIを活用しながらもAIに依存せず、以下を自分で設計・実装・検証・説明できる状態を目指します。

- フロントエンドからバックエンドまで一貫して実装できる
- GraphQL APIと認証・認可を設計できる
- PostgreSQLのTransaction / Index / Lockを理解できる
- 大量データを安全かつ効率的に処理できる
- Redis / Queue / Kafkaを用途に応じて選択できる
- Docker / AWS / CI/CDを使って本番環境へデプロイできる
- Logging / Metrics / Tracingを使って障害を調査できる
- Performance / Scalability / Reliabilityを考慮した設計ができる
- 技術選定について「なぜその方法を選んだか」を説明できる

---

# Tech Stack

## Backend

- Python
- FastAPI
- Strawberry GraphQL
- SQLAlchemy Async
- PostgreSQL
- Redis
- Kafka / Queue
- Alembic

## Frontend

- TypeScript
- React
- Next.js App Router
- Tailwind CSS
- GraphQL
- TanStack Query（後半で導入予定）

## Infrastructure

- Docker
- AWS
- ECS
- RDS
- ElastiCache
- Terraform
- GitHub Actions

## Observability

- Structured Logging
- Metrics
- Distributed Tracing
- Error Monitoring
- SLI / SLO
- Alerting

---

# Phase 1 — Full-Stack Foundation

## Goal

まずはアプリケーション全体の流れを一度完成させる。

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

認証も含めて、Frontend → API → Databaseまでの基本的な流れを理解する。

---

## Backend

- [ ] FastAPI基本構成
- [ ] Strawberry GraphQL
- [ ] Query / Mutation
- [ ] Resolver
- [ ] Input / Type
- [ ] Service Layer
- [ ] SQLAlchemy ORM
- [ ] Async SQLAlchemy
- [ ] PostgreSQL
- [ ] CRUD
- [ ] User / Issue Relation
- [ ] DataLoader
- [ ] N+1問題
- [ ] JWT Authentication
- [ ] GraphQL Context
- [ ] current_user
- [ ] Authorization基礎
- [ ] Filter
- [ ] Search
- [ ] Cursor Pagination

---

## Frontend

- [ ] Next.js App Router
- [ ] TypeScript
- [ ] Tailwind CSS
- [ ] GraphQL fetch helper
- [ ] Login画面
- [ ] JWT保存
- [ ] Issue一覧
- [ ] Issue作成
- [ ] Issue更新
- [ ] Issue削除
- [ ] Logout
- [ ] Loading UI
- [ ] Error UI
- [ ] Browser NetworkでGraphQL Request確認

この段階では仕組みを理解しやすくするため、GraphQL Client Libraryや大規模なState Managementは使用しない。

```text
fetch
useState
useEffect
localStorage
```

を中心に実装する。

---

## Phase 1 Completion Criteria

以下の流れを説明できること。

```text
Login
  ↓
JWT
  ↓
Authorization Header
  ↓
GraphQL Context
  ↓
current_user
  ↓
Service
  ↓
Database
```

また、Frontendから以下のGraphQL操作を実行できること。

```text
Query
Mutation
Create
Read
Update
Delete
```

---

# Phase 2 — Backend Senior & Large-Scale Data Processing

## Goal

「動くアプリケーション」から、

**大量のリクエストや大量データでも壊れにくいアプリケーション**

へ進化させる。

---

## Transaction

- [ ] ACID
- [ ] Transaction Boundary
- [ ] Commit / Rollback
- [ ] Isolation Level
- [ ] Dirty Read
- [ ] Non-repeatable Read
- [ ] Phantom Read
- [ ] Optimistic Lock
- [ ] Pessimistic Lock
- [ ] Race Condition
- [ ] Deadlock
- [ ] Deadlock Retry
- [ ] Idempotency

### Practice

同じIssueを複数ユーザーが同時に更新するケースを実装する。

```text
User A ─┐
        ├─ Update Issue
User B ─┘
```

どのようなRace Conditionが発生するか確認する。

---

## Database Performance

- [ ] Index
- [ ] Composite Index
- [ ] EXPLAIN
- [ ] EXPLAIN ANALYZE
- [ ] Query Plan
- [ ] Sequential Scan
- [ ] Index Scan
- [ ] Slow Query
- [ ] Connection Pool
- [ ] Pagination Performance

### Practice

Issueを大量生成して以下を比較する。

```text
10,000 rows
100,000 rows
1,000,000 rows
```

Indexの有無によるQuery Performanceを比較する。

---

## Large-Scale Data Processing

- [ ] Batch Processing
- [ ] Chunk Processing
- [ ] Cursor Processing
- [ ] Keyset Pagination
- [ ] Bulk Insert
- [ ] Bulk Update
- [ ] Streaming
- [ ] Memory-safe Processing
- [ ] Checkpoint
- [ ] Retry
- [ ] Backpressure
- [ ] Parallel Processing

### Practice

大量のIssueを一括更新する。

```text
100,000 Issues

OPEN
  ↓
PROCESSING
  ↓
DONE
```

すべてを一度にMemoryへ読み込むのではなく、Chunk単位で処理する。

---

## Cache

- [ ] Redis
- [ ] Cache Aside
- [ ] TTL
- [ ] Cache Invalidation
- [ ] Hot Key
- [ ] Cache Stampede
- [ ] Distributed Lock

---

## Queue / Background Job

- [ ] Background Processing
- [ ] Producer
- [ ] Consumer
- [ ] Retry
- [ ] Dead Letter Queue
- [ ] Idempotent Consumer

---

## Kafka

- [ ] Topic
- [ ] Partition
- [ ] Producer
- [ ] Consumer
- [ ] Consumer Group
- [ ] Offset
- [ ] Ordering
- [ ] At-most-once
- [ ] At-least-once
- [ ] Exactly-once concept
- [ ] Retry
- [ ] DLQ
- [ ] Replay

### Practice

Issue作成時にEventを発行する。

```text
Create Issue
    ↓
Kafka Event
    ↓
Worker
    ↓
Activity Log
```

---

## Frontend — Large Data & Server State

Backendの大量データ処理と合わせて、Frontend側でも大量データを扱う。

- [ ] Cursor Pagination
- [ ] Infinite Scroll
- [ ] Virtualized List
- [ ] Debounce Search
- [ ] Request Cancellation
- [ ] Duplicate Submit Prevention
- [ ] Optimistic Update
- [ ] Server State Cache
- [ ] TanStack Query導入
- [ ] Cache Invalidation
- [ ] Retry Strategy

### Practice

100,000件以上のIssueを前提に、全件取得しないUIを実装する。

```text
Frontend
  ↓
Cursor
  ↓
GraphQL
  ↓
LIMIT N
```

---

# Phase 3 — Application Architecture

## Goal

コード量が増えても変更しやすい構造を作る。

---

## Backend Architecture

- [ ] Resolver / Controller
- [ ] Service
- [ ] Repository
- [ ] Domain Model
- [ ] Dependency Direction
- [ ] Modular Monolith
- [ ] Dependency Injection
- [ ] DDD Basics
- [ ] Aggregate
- [ ] Value Object
- [ ] Domain Service
- [ ] CQRS
- [ ] Event-driven Architecture
- [ ] Microservices Trade-offs

---

## Practice

現在の構成:

```text
Resolver
  ↓
Service
  ↓
SQLAlchemy
```

から、

```text
Resolver
  ↓
Application Service
  ↓
Repository
  ↓
Database
```

へ段階的に分離する。

ただし、単純なCRUDまで過剰に抽象化しない。

---

## Frontend Architecture

- [ ] Feature-based Structure
- [ ] Server Component / Client Component Boundary
- [ ] API Layer
- [ ] Domain Type
- [ ] UI Component
- [ ] Custom Hook
- [ ] Server State / Client State分離
- [ ] Shared Component Boundary
- [ ] Error Boundary
- [ ] Form Responsibility

### Example

```text
features/
└── issue/
    ├── api/
    ├── components/
    ├── hooks/
    ├── types/
    └── utils/
```

Frontendでも「どこに何を書くか」ではなく、

**どの責務をどこに置くか**

を考える。

---

# Phase 4 — Infrastructure & Deployment

## Goal

Local環境だけでなく、実際に運用できる環境へデプロイする。

---

## Container

- [ ] Docker
- [ ] Dockerfile
- [ ] Multi-stage Build
- [ ] Docker Compose
- [ ] Container Networking
- [ ] Environment Variables
- [ ] Secret Management

---

## AWS

- [ ] VPC基礎
- [ ] ECS
- [ ] ALB
- [ ] RDS PostgreSQL
- [ ] ElastiCache Redis
- [ ] S3
- [ ] CloudFront
- [ ] IAM
- [ ] Security Group
- [ ] Route53基礎

---

## Infrastructure as Code

- [ ] Terraform
- [ ] Provider
- [ ] Resource
- [ ] Variable
- [ ] Output
- [ ] State
- [ ] Module

---

## CI/CD

- [ ] GitHub Actions
- [ ] Lint
- [ ] Test
- [ ] Build
- [ ] Docker Image
- [ ] Deploy
- [ ] Migration
- [ ] Rollback Strategy

---

## Frontend Production

- [ ] Next.js Production Build
- [ ] Development / Staging / Production環境
- [ ] Environment Variable管理
- [ ] CDN
- [ ] Browser Cache
- [ ] Static Asset Cache
- [ ] Image Optimization
- [ ] Font Optimization
- [ ] Bundle Analysis
- [ ] Code Splitting
- [ ] Lazy Loading
- [ ] Deployment Strategy

---

# Phase 5 — Security, Reliability & Observability

## Goal

「正常に動く」だけではなく、

**攻撃・障害・ネットワーク異常があっても安全に運用できる**

状態を目指す。

---

## Authentication / Security

Phase 1では理解しやすさを優先して、JWTをlocalStorageへ保存する。

このPhaseでは実運用を想定した認証へ変更する。

- [ ] HttpOnly Cookie
- [ ] Secure Cookie
- [ ] SameSite
- [ ] CSRF
- [ ] Access Token
- [ ] Refresh Token
- [ ] Token Rotation
- [ ] Token Revocation
- [ ] Logout
- [ ] Redis / Session Store
- [ ] OAuth 2.0
- [ ] OpenID Connect
- [ ] RBAC
- [ ] ABAC
- [ ] GraphQL Field Authorization
- [ ] Query Depth Limit
- [ ] Query Complexity Limit

---

## Reliability

- [ ] Timeout
- [ ] Retry
- [ ] Exponential Backoff
- [ ] Circuit Breaker
- [ ] Rate Limiting
- [ ] Graceful Shutdown
- [ ] Health Check
- [ ] Readiness Check
- [ ] Idempotency
- [ ] Failure Recovery

---

## Observability

### Logging

- [ ] Structured Logging
- [ ] Log Level
- [ ] Request ID
- [ ] Correlation ID
- [ ] Sensitive Data Masking

### Metrics

- [ ] Request Count
- [ ] Error Rate
- [ ] Latency
- [ ] Throughput
- [ ] CPU
- [ ] Memory
- [ ] DB Connections
- [ ] Queue Lag

### Tracing

- [ ] Distributed Tracing
- [ ] Trace ID
- [ ] Span
- [ ] API → DB Trace
- [ ] API → Queue → Worker Trace

### Operations

- [ ] Dashboard
- [ ] Alert
- [ ] SLI
- [ ] SLO
- [ ] Error Budget
- [ ] Incident Investigation

---

## Frontend Reliability & Monitoring

- [ ] Global Error Boundary
- [ ] API Error Handling
- [ ] Timeout UX
- [ ] Retry UX
- [ ] Unauthorized UX
- [ ] Role-based UI
- [ ] Refresh Token Flow
- [ ] Error Monitoring
- [ ] Web Vitals
- [ ] Frontend Performance Monitoring
- [ ] User Action Trace

---

# Phase 6 — Performance & System Design

## Goal

単一機能の実装ではなく、システム全体を設計できるようになる。

---

## Backend / System Design

- [ ] Requirement Clarification
- [ ] Functional Requirements
- [ ] Non-functional Requirements
- [ ] Traffic Estimation
- [ ] Storage Estimation
- [ ] API Design
- [ ] Database Design
- [ ] Cache Strategy
- [ ] Queue Strategy
- [ ] Horizontal Scaling
- [ ] Database Bottleneck
- [ ] Partitioning
- [ ] Sharding
- [ ] Replication
- [ ] Failure Scenario
- [ ] Recovery Strategy

---

## Frontend Performance

- [ ] SSR / CSR / RSCの選択
- [ ] Hydration Cost
- [ ] Rendering Optimization
- [ ] Memoization
- [ ] Bundle Splitting
- [ ] Lazy Loading
- [ ] Virtualization
- [ ] Core Web Vitals
- [ ] Performance Budget
- [ ] Network Waterfall Analysis
- [ ] Browser Performance Profile

---

## System Design Practice

以下を設計する。

- [ ] Large-scale Issue Tracker
- [ ] URL Shortener
- [ ] Notification System
- [ ] File Upload Service
- [ ] Chat System
- [ ] News Feed
- [ ] Autocomplete
- [ ] Rate Limiter
- [ ] Job Queue
- [ ] Analytics Pipeline

---

# Issue Tracker Evolution

このロードマップでは同じIssue Trackerを継続して拡張する。

```text
Phase 1
Simple CRUD
  ↓
JWT Authentication
  ↓
Next.js UI

Phase 2
Transaction
  ↓
Index
  ↓
Large Data
  ↓
Redis
  ↓
Queue / Kafka

Phase 3
Architecture Refactoring
  ↓
Repository
  ↓
Domain Boundary
  ↓
Event-driven

Phase 4
Docker
  ↓
AWS
  ↓
CI/CD

Phase 5
Security
  ↓
Reliability
  ↓
Monitoring / Tracing

Phase 6
Performance
  ↓
Scalability
  ↓
System Design
```

---

# Senior Engineer Questions

各テーマを学習するときは、実装方法だけではなく以下を必ず考える。

## Why?

```text
なぜこの技術を使うのか？
```

## Alternatives?

```text
他にどんな方法があるのか？
```

## Trade-offs?

```text
何を得て、何を失うのか？
```

## Scale?

```text
データ量やTrafficが100倍になったらどうなるか？
```

## Failure?

```text
途中で失敗したらどうなるか？
```

## Recovery?

```text
どのように復旧するか？
```

## Observability?

```text
障害が発生したことをどうやって知るのか？
```

---

# Final Goal

最終的に目指すのは、単にFrontendとBackendの両方を書けるEngineerではない。

```text
Frontend
Backend
Database
Infrastructure
Security
Performance
Operations
```

を横断して考え、

```text
「なぜこの設計なのか」
```

をTrade-offまで含めて説明できるFull-Stack Engineerになることを目標とする。
