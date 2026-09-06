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

# Step 12 — Frontend Feature Structure

```text
src/
  features/
    issue/
      api/
      components/
      hooks/
      types/
```

Move:

```text
Issue API
Issue Form
Issue List
Issue hooks
```

---

# Step 13 — Server / Client Boundary

Server Componentでできるもの:

```text
初期表示用Data Fetch
Secretを使う処理
```

Client Component:

```text
Form
Click
Browser API
localStorage
```

各Componentに `"use client"` が本当に必要か確認する。

---

# Step 14 — Test Boundary

Unit:

```text
Domain
Use Case
Pure Function
```

Integration:

```text
Repository + PostgreSQL
GraphQL + Context
```

E2E:

```text
Browser → API → DB
```

各Featureに対してどのLevelでTestすべきか表を作る。

---

# Step 15 — Architecture Decision Record

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
[ ] Domain LogicがFrameworkから分離
[ ] Fake RepositoryでUnit Test可能
[ ] ResolverがThin
[ ] Module Boundaryが説明可能
[ ] Frontend Feature Boundaryが説明可能
[ ] ADRを書ける
```
