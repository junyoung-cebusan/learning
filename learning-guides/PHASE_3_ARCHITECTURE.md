# Phase 3 — Architecture

## Goal

Phase 2までに経験した複雑性を踏まえ、Issue Trackerを  
**変更しやすく、責務が明確で、Testしやすい構造**へRefactoringする。

ArchitectureはFolderを細かく分けることではない。

```text
どのModuleが何を知るべきか？
変更がどこまで波及するか？
なぜこのCodeはTestしづらいのか？
```

を考える。

---

## Step 1 — 現在のDependencyを可視化

```text
GraphQL Resolver
→ Service
→ SQLAlchemy
→ PostgreSQL
```

現在の依存関係と問題点を書く。

---

## Step 2 — Repository

```text
Resolver
→ Application Service
→ Repository
→ SQLAlchemy
```

- IssueRepository Interface
- SQLAlchemyIssueRepository
- ServiceからSQLAlchemy依存を外す

### Testing

- Fake Repository
- DBなしのService Unit Test

---

## Step 3 — Dependency Injection

- Dependency Direction
- Constructor Injection
- Framework DependencyとBusiness Logicの分離

```text
Business Logic
≠ FastAPI / Strawberry / SQLAlchemy
```

---

## Step 4 — Domain Model

- Entity
- Value Object
- Invariant

候補:

```text
IssueStatus
IssueTitle
Issue
```

DDDを目的化せず、実際にRuleがある箇所だけ適用する。

---

## Step 5 — Application Service / Use Case

CRUD中心ではなくUser Actionとして考える。

```text
CreateIssue
ChangeIssueStatus
AssignIssue
ArchiveIssues
```

---

## Step 6 — Modular Monolith

すぐにMicroservicesへ分割しない。

```text
issue/
auth/
activity/
notification/
```

- Module Boundary
- Public Interface
- Cross-module Dependency

---

## Step 7 — Event-driven Architecture

比較:

```text
Create Issue
→ Notificationを直接呼ぶ
```

vs

```text
Create Issue
→ IssueCreated Event
→ Notification Handler
```

CouplingとOperational ComplexityのTrade-offを確認する。

---

## Step 8 — CQRS Basics

```text
Command
→ 状態を変更

Query
→ 読み取り
```

いつ分離する価値があるかを考える。

---

## Step 9 — Microservices Trade-offs

実際に分割する前に以下を検討する。

- Network Failure
- Distributed Transaction
- Tracing
- Deployment
- Schema Compatibility
- Operational Cost

---

## Step 10 — Frontend Feature Structure

```text
features/
  issue/
    api/
    components/
    hooks/
    types/
```

- Feature Boundary
- Shared Component
- Domain Type
- API Type

---

## Step 11 — Server / Client Component Boundary

Next.jsで以下を判断する。

- Interactivity
- Browser API
- Data Fetching
- Bundle Size
- Security Boundary

---

## Step 12 — Server State / Client State

```text
Server State
→ APIから取得するData
→ TanStack Query

Client State
→ UI Local State
```

不要なGlobal Stateを減らす。

---

## Step 13 — Testability

Backend:

- Fake Repository
- Unit Test
- Integration Test Boundary

Frontend:

- Pure Function
- Component Contract
- Feature Integration Test

常に以下を考える。

```text
なぜこのCodeはTestしづらいのか？
Dependencyを変えると簡単になるか？
```

---

## Phase 3 Completion Criteria

- Repositoryを導入した理由を説明できる
- Repositoryが不要な場合を説明できる
- Domain ModelとORM Modelを分離する理由を説明できる
- Modular MonolithがMicroservicesより適切なScenarioを説明できる
- Event-driven ArchitectureのCostを説明できる
- Frontend Feature Boundaryの決め方を説明できる
