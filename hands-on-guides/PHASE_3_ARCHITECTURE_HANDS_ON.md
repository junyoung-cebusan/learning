# Phase 3 — Architecture Hands-on Guide

## Goal

Phase 2までのIssue Trackerを、  
**変更しやすくTestしやすい構造**へ段階的にRefactoringする。

---

# Step 1 — 現在のDependencyを描く

現在:

```text
Resolver
→ Service
→ SQLAlchemy
→ PostgreSQL
```

まずCodeを変更せず、各Layerが何を知っているか書き出す。

---

# Step 2 — Repository Interface

`src/app/repositories/issue.py`

```python
from typing import Protocol

from app.domain.issue import Issue


class IssueRepository(Protocol):
    async def get(
        self,
        issue_id: int,
    ) -> Issue | None:
        ...

    async def save(
        self,
        issue: Issue,
    ) -> None:
        ...
```

---

# Step 3 — Domain Model

`src/app/domain/issue.py`

```python
from dataclasses import dataclass


@dataclass
class Issue:
    id: int | None
    title: str
    status: str

    def close(self):
        if self.status == "CLOSED":
            return

        self.status = "CLOSED"
```

ORM Modelと分離する。

---

# Step 4 — SQLAlchemy Repository

```python
class SQLAlchemyIssueRepository:
    def __init__(
        self,
        session: AsyncSession,
    ):
        self.session = session

    async def get(
        self,
        issue_id: int,
    ) -> Issue | None:
        model = await self.session.get(
            IssueModel,
            issue_id,
        )

        if model is None:
            return None

        return Issue(
            id=model.id,
            title=model.title,
            status=model.status,
        )
```

---

# Step 5 — Application Service

```python
class CloseIssue:
    def __init__(
        self,
        repository: IssueRepository,
    ):
        self.repository = repository

    async def execute(
        self,
        issue_id: int,
    ):
        issue = await self.repository.get(
            issue_id
        )

        if issue is None:
            raise ValueError(
                "Issue not found"
            )

        issue.close()

        await self.repository.save(
            issue
        )

        return issue
```

---

# Step 6 — Fake Repository Unit Test

```python
class FakeIssueRepository:
    def __init__(self):
        self.items = {
            1: Issue(
                id=1,
                title="Test",
                status="OPEN",
            )
        }

    async def get(
        self,
        issue_id: int,
    ):
        return self.items.get(
            issue_id
        )

    async def save(
        self,
        issue: Issue,
    ):
        self.items[issue.id] = issue
```

Test:

```python
@pytest.mark.asyncio
async def test_close_issue():
    repository = (
        FakeIssueRepository()
    )

    use_case = CloseIssue(
        repository
    )

    issue = await use_case.execute(
        1
    )

    assert issue.status == "CLOSED"
```

DBなしでBusiness LogicをTestできることを確認する。

---

# Step 7 — ResolverをThinにする

```python
@strawberry.mutation
async def close_issue(
    self,
    id: int,
    info: strawberry.Info,
):
    use_case = (
        info.context.close_issue
    )

    return await use_case.execute(
        id
    )
```

ResolverでSQLAlchemyを直接扱わない。

---

# Step 8 — Dependency Injection

Context作成時にDependencyを組み立てる。

```text
Session
→ Repository
→ Use Case
→ Context
→ Resolver
```

Framework依存を外側へ追い出す。

---

# Step 9 — Modular Monolith

Structure:

```text
app/
  modules/
    issue/
      domain/
      application/
      infrastructure/
      graphql/
    auth/
    activity/
```

一気に移動せず、Issue Moduleから段階的に移す。

---

# Step 10 — Domain Event

```python
@dataclass
class IssueCreated:
    issue_id: int
```

Application Service:

```python
events.append(
    IssueCreated(
        issue_id=issue.id
    )
)
```

Event Handler:

```python
async def handle_issue_created(
    event: IssueCreated,
):
    ...
```

直接Notificationを呼ぶ方式と比較する。

---

# Step 11 — CQRS Mini Practice

Write Model:

```text
Issue
```

Read Model:

```text
IssueSummary
```

Query用Projectionを別構造にする簡単な実験を行う。

「便利だから」ではなく、Read Patternが異なる場合だけ価値があることを確認する。

---

# Step 12 — Frontend Architecture Audit

Phase 1 / Phase 2では、まず機能を完成させることを優先した。

Phase 3では、現在のFrontendを見ながら
**本当に分離が必要な責務だけを分離する**。

確認すること:

```text
[ ] Page Componentが大きすぎないか
[ ] "use client" が必要以上に上位へ広がっていないか
[ ] GraphQL RequestがUIの中に直接混在していないか
[ ] Apollo Cache LogicがComponentへ漏れていないか
[ ] Server State / Client State / Form Stateが混ざっていないか
[ ] 同じ変換処理が複数Componentに重複していないか
[ ] localStorage / window / routerへの依存が広がっていないか
```

このStepではまだFolderを増やさない。

---

# Step 13 — Practical Target Structure

最初から、

```text
api/
hooks/
services/
repositories/
adapters/
usecases/
```

のように細かく分けない。

Issue Featureの規模では、まず以下から始める。

```text
src/
├── app/
│   └── issues/
│       ├── page.tsx
│       ├── loading.tsx
│       └── error.tsx
│
├── features/
│   └── issue/
│       ├── IssueList.tsx
│       ├── IssueForm.tsx
│       ├── issue.graphql.ts
│       └── issue-utils.ts
│
├── graphql/
│   └── *.graphql
│
├── generated/
│   └── graphql.ts
│
└── lib/
    └── apollo/
        └── client.ts
```

責務:

```text
app/
→ Routing / Layout / Server-Client Boundary

features/issue/
→ Issue Feature固有のUIとLogic

graphql/
→ Operation Source

generated/
→ Codegen Output

lib/
→ App全体で共有するInfrastructure
```

Featureが成長してから必要に応じて:

```text
features/issue/
├── api/
├── hooks/
├── components/
└── utils/
```

へ分割する。

**Folder Structureは先に決めすぎない。**

---

# Step 14 — Server ComponentをDefaultにする

Next.js App RouterではServer ComponentがDefault。

Page全体を最初から:

```tsx
"use client";
```

にしない。

Server Componentでよいもの:

```text
Layout
Static Content
Initial Data Fetch
Metadata
非InteractiveなList
```

Client Componentが必要なもの:

```text
Form
Button
useState
useEffect
Apollo Hook
localStorage
window / document
```

目標:

```text
Server Page
├── Server-rendered Content
└── Small Client Island
```

Client Boundaryをできるだけ小さくする。

---

# Step 15 — Data Ownershipを決める

同じEntityを、

```text
RSC側
Apollo Cache側
```

の両方で無秩序に管理しない。

Issueのように:

```text
頻繁にMutationされる
Optimistic Updateする
Normalized Cacheを使う
```

DataはClient Apollo側で所有する方が自然。

例:

```text
Issue List
Issue Detail
Create / Update / Delete
```

はApollo Client側。

一方:

```text
公開ページ
SEO重視
ほぼRead-only
初期表示が重要
```

ならServer Component側でFetchする方が自然。

Phase 3では各Dataについて:

```text
Server-owned
or
Client-owned
```

を明示的に決める。

---

# Step 16 — Apollo ClientはClient-side Interactive Dataへ集中

Issue FeatureはPhase 2でApolloを学んだため、
InteractiveなIssue DataはApolloをMainとする。

Client Component:

```tsx
"use client";

import {
  useQuery,
} from "@apollo/client/react";

import {
  GetIssuesDocument,
} from "@/generated/graphql";


export function IssueList() {
  const {
    data,
    loading,
    error,
  } = useQuery(
    GetIssuesDocument,
  );

  // ...
}
```

この段階で、単に:

```typescript
useIssues()
```

へ1行Wrapするだけなら、
Custom Hookを作らなくてもよい。

---

# Step 17 — Custom Hookは価値がある時だけ作る

Custom Hookを作る条件:

```text
複数Queryを組み合わせる
Pagination Stateを管理する
Filter / Searchと連携する
Mutation + Cache Updateをまとめる
Optimistic Updateをまとめる
Feature固有のError Mappingがある
複数Componentで再利用する
```

例:

```typescript
export function useIssueList(
  status?: string,
) {
  const query =
    useQuery(
      GetIssuesDocument,
      {
        variables: {
          status,
        },
      },
    );

  return {
    issues:
      query.data?.issues ?? [],
    loading:
      query.loading,
    error:
      query.error,
  };
}
```

避ける:

```text
useQuery()
↓
ただReturnするだけのHook
```

を全Queryで機械的に作ること。

---

# Step 18 — API Layerも必要な時だけ分離する

Apollo Clientを使っている場合、
単純なQuery / Mutationまで別Service FunctionでWrapする必要はない。

以下の場合にAPI Layerを作る。

```text
Apollo以外のHTTP Requestもある
複数Operationを1つのUse Caseとしてまとめる
GraphQL Responseを大きく変換する
React以外から同じAPI Logicを使う
SSR / Server Actionから再利用する
```

つまり:

```text
Component
→ Apollo Hook
```

で十分なケースもある。

無条件で:

```text
Component
→ Hook
→ Service
→ API
→ Client
```

にしない。

---

# Step 19 — Shared Infrastructureだけ`lib/`へ置く

Apollo Client生成など、
Featureに依存しないものだけをShared Layerへ置く。

例:

```text
src/lib/apollo/client.ts
```

ここでは:

```text
Endpoint
Auth Header
Apollo Link
Cache
Error Link
```

などApp-wideな設定を扱う。

Feature固有のIssue Logicは入れない。

---

# Step 20 — Server State / Client State / Form State

Stateを3種類に分けて考える。

## Server State

```text
Issue List
Issue Detail
Current User
Pagination Result
```

Apollo Client。

## Client State

```text
Modal
Selected Tab
Temporary Filter UI
Sidebar
```

React State。

## Form State

```text
title
description
validation
dirty
```

Form Component内部。

Server Stateを:

```typescript
useState(...)
```

へ複製しすぎない。

Apollo CacheがSource of Truthなら、
同じIssue Listを別`useState`へコピーしない。

---

# Step 21 — Pure FunctionはUIから分離する

以下はPure Function候補:

```text
Status Label変換
Variables生成
Filter変換
Response → View Model
Date Formatting
Validation Rule
```

例:

```typescript
export function formatIssueStatus(
  status: string,
) {
  return status === "DONE"
    ? "Done"
    : "Open";
}
```

Pure Functionは:

```text
Reactなし
Browser APIなし
Networkなし
Global Stateなし
```

Unit Testしやすい。

---

# Step 22 — SSR / Client Fetchを実際に比較する

## Client Apollo

```text
Browser
→ Apollo
→ GraphQL
→ Render
```

向いている:

```text
Interactive
Mutation多い
Optimistic Update
Normalized Cache
```

## Server Fetch

```text
Request
→ Next.js Server
→ GraphQL
→ RSC / HTML
→ Browser
```

向いている:

```text
Read-heavy
Initial Render重要
SEO
SecretをServerに置きたい
```

DevToolsとServer Logで違いを確認する。

---

# Step 23 — localStorage JWTとSSRの制約

Phase 1ではJWTを`localStorage`へ保存した。

Server ComponentはBrowserの`localStorage`を読めない。

そのためAuthenticated Issue Dataは現状:

```text
Client Component
→ localStorage JWT
→ Apollo
→ Backend
```

が自然。

Phase 3では、
無理にSSRへ移行しない。

Phase 5でHttpOnly Cookieを導入した後:

```text
Browser
→ Cookie
→ Next.js Server
→ GraphQL Backend
```

を扱う。

Phase 3のGoal:

```text
なぜSSRできないのか説明できる
```

こと。

---

# Step 24 — Apollo + Next.js Integrationを理解する

ApolloをApp RouterでSSR / RSCと組み合わせる場合、
単純なClient Providerだけでなく
Next.js向けIntegrationが存在することを理解する。

学習対象:

```text
@apollo/client-integration-nextjs
ApolloNextAppProvider
PreloadQuery
SSR / Hydration
Normalized Cache
```

ただしPhase 3では全Issue DataをSSRへ移さない。

まず:

```text
Client-owned Issue Data
```

を維持した上で、
別のRead-only Queryを使って
Server Fetch / Hydrationを小さく試す。

---

# Step 25 — Hydrationを小さく体験する

Read-onlyなQueryを1つ用意して、

```text
Server Fetch
→ Initial Result
→ Client Hydration
```

を確認する。

確認:

```text
[ ] Clientで不要なDuplicate Fetchがないか
[ ] ServerとClientでData Shapeが一致するか
[ ] Hydration Errorがないか
[ ] Client Cacheへ正しく引き継がれるか
```

重要:

```text
SSRを使える
```

ことと、

```text
SSRを使うべき
```

ことは別。

---

# Step 26 — Route / Feature Boundary

`app/`にはRoute責務を残す。

```tsx
// app/issues/page.tsx

import {
  IssueScreen,
} from "@/features/issue/IssueScreen";


export default function Page() {
  return (
    <IssueScreen />
  );
}
```

Page Fileに:

```text
GraphQL Cache Logic
Complex Form Logic
Response Mapping
Business Rule
```

を詰め込まない。

---

# Step 27 — Loading / Error Boundary

App Routerの:

```text
loading.tsx
error.tsx
```

を実際に使う。

さらにClient Apollo側では:

```text
Query Loading
GraphQL Error
Mutation Error
Optimistic Rollback
```

を分ける。

Route ErrorとFeature Errorを同じものとして扱わない。

---

# Step 28 — Frontend Dependency Direction

実務的な依存方向:

```text
app
↓
feature
↓
generated GraphQL / shared lib
```

必要な場合だけ:

```text
feature component
↓
feature hook
↓
feature api
```

を追加する。

禁止したい方向:

```text
shared lib
→ feature

generated/
→ UI Logic

feature A
→ feature Bの内部実装
```

Rule:

```text
低LevelなShared Layerは
Featureを知らない
```

---

# Step 29 — Refactoring前後のTest

Architecture変更で機能を壊さない。

## Unit

Vitest:

```text
Pure Function
Mapping
Validation
Variables Builder
```

## Component

React Testing Library:

```text
Form
Loading
Error
Interactive UI
```

## Apollo Integration

```text
Query
Mutation
Normalized Cache
Pagination
Optimistic Update
```

## E2E

Playwright:

```text
Login
→ Issue List
→ Create
→ Update
→ Delete
→ Logout
```

重要:

```text
Architecture Test
≠ Folder Structure Test

User BehaviorとBoundaryをTestする
```

---

# Step 30 — Frontend ADR

`docs/adr/002-frontend-data-boundary.md`

```md
# Context

IssueはMutationが多く、
Apollo Normalized Cacheを利用している。

# Decision

Issue EntityはClient Apollo側でOwnershipを持つ。

Server ComponentはDefaultとし、
Interactive部分だけClient Componentにする。

単純なQueryはuseQueryを直接使い、
複雑なOrchestrationがある場合だけCustom Hookを作る。

Shared Infrastructureだけlibへ置く。

# Alternatives

すべてSSR / RSCで取得する。

全QueryをCustom Hook / ServiceでWrapする。

# Trade-offs

Layer数を抑えられるが、
Featureが成長した場合は追加分離が必要になる。
```

---

# Step 31 — Test Boundary

Backend:

```text
Unit
→ Domain / Use Case

Integration
→ Repository + PostgreSQL
→ GraphQL + Context
```

Frontend:

```text
Unit
→ Pure Function

Component
→ UI Interaction

Integration
→ Apollo Cache / GraphQL

E2E
→ Browser → API → DB
```

---

# Step 32 — Frontend Architecture Review

最後に以下を説明する。

```text
なぜServer ComponentをDefaultにするか

なぜIssueはClient Apolloで持つか

なぜ同じEntityをRSCとApollo両方で持たないか

Custom Hookを作る基準

API Layerを作る基準

Server StateとClient Stateの違い

localStorage AuthがSSRを制限する理由

いつSSRを使い、いつ使わないか
```

説明できれば、
Folder Structureを暗記するのではなく
Architecture判断ができている。
# Step 28 — Backend Architecture Decision Record

`docs/adr/001-repository.md`

```md
# Context

ServiceがSQLAlchemyへ直接依存していた。

# Decision

Repositoryを導入する。

# Alternatives

Serviceから直接SQLAlchemyを利用。

# Trade-offs

抽象化は増えるがUnit Testしやすくなる。
```

---

# Phase 3 Completion

```text
Backend
[ ] Domain LogicがFrameworkから分離
[ ] Fake RepositoryでUnit Test可能
[ ] ResolverがThin
[ ] Repository / Application Service / Infrastructureの責務を説明できる
[ ] Module Boundaryが説明可能

Frontend
[ ] Server ComponentをDefaultとして設計できる
[ ] Client Boundaryを必要な場所だけに置ける
[ ] Server-owned / Client-owned Dataを判断できる
[ ] Apollo CacheをSource of Truthとして扱える
[ ] Server State / Client State / Form Stateを区別できる
[ ] Custom Hookを作る基準を説明できる
[ ] API Layerを作る基準を説明できる
[ ] Pure Functionを分離できる
[ ] SSR / Client FetchのTrade-offを説明できる
[ ] localStorage JWTがSSRを制限する理由を説明できる
[ ] Apollo + Next.js Hydrationを小さく実装できる
[ ] Loading / Error Boundaryを設計できる

Architecture
[ ] Over-engineeringを避ける判断ができる
[ ] Refactoring後もTestが通る
[ ] Backend / FrontendそれぞれのADRを書ける
[ ] 「なぜこのLayerが必要か」を説明できる
```
