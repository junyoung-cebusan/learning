# Phase 1 — Full-Stack Foundation Hands-on Guide

> 既存の1〜12段階の学習順序は変更しない。  
> 既に進めているIssue Trackerをそのまま使い、必要なTestとFrontend実装を追加する。

---

# 0. このPhaseの完成形

```text
Next.js
  ↓ GraphQL
FastAPI + Strawberry
  ↓
Service
  ↓
SQLAlchemy Async
  ↓
PostgreSQL
```

Authentication:

```text
Login
→ JWT
→ Authorization Header
→ GraphQL Context
→ current_user
```

Testing:

```text
Frontend Unit     : Vitest
Frontend Component: React Testing Library
Frontend E2E      : Playwright

Backend Unit      : pytest
Backend Async     : pytest-asyncio
Backend Integration: httpx
```

---

# Step 1 — FastAPI

## Goal

最小のFastAPI Applicationを起動する。

## Install

```bash
uv add fastapi uvicorn
```

## `src/app/main.py`

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/health")
async def health():
    return {
        "status": "ok",
    }
```

## Run

```bash
uv run uvicorn app.main:app --reload
```

## Check

```bash
curl http://localhost:8000/health
```

Expected:

```json
{
  "status": "ok"
}
```

---

# Step 2 — Strawberry GraphQL

## Install

```bash
uv add "strawberry-graphql[fastapi]"
```

## `src/app/graphql/query.py`

```python
import strawberry


@strawberry.type
class Query:
    @strawberry.field
    def hello(self) -> str:
        return "Hello GraphQL"
```

## `src/app/graphql/schema.py`

```python
import strawberry

from app.graphql.query import Query


schema = strawberry.Schema(
    query=Query,
)
```

## `src/app/main.py`

```python
from fastapi import FastAPI
from strawberry.fastapi import GraphQLRouter

from app.graphql.schema import schema


app = FastAPI()

graphql_app = GraphQLRouter(schema)

app.include_router(
    graphql_app,
    prefix="/graphql",
)
```

## GraphiQL

```graphql
query {
  hello
}
```

Expected:

```json
{
  "data": {
    "hello": "Hello GraphQL"
  }
}
```

---

# Step 3 — GraphQL Type / Input / Mutation

## `src/app/graphql/schemas/issue.py`

```python
from datetime import datetime

import strawberry


@strawberry.type
class Issue:
    id: int
    title: str
    description: str | None
    status: str
    created_at: datetime


@strawberry.input
class CreateIssueInput:
    title: str
    description: str | None = None
```

## Memory Store

`src/app/services/issue_memory.py`

```python
from datetime import datetime, timezone

from app.graphql.schemas.issue import (
    CreateIssueInput,
    Issue,
)


issues: list[Issue] = []


def create_issue(
    input: CreateIssueInput,
) -> Issue:
    issue = Issue(
        id=len(issues) + 1,
        title=input.title,
        description=input.description,
        status="OPEN",
        created_at=datetime.now(
            timezone.utc
        ),
    )

    issues.append(issue)

    return issue


def get_issues() -> list[Issue]:
    return issues
```

---

# Step 4 — Memory CRUD

## Query

```python
import strawberry

from app.graphql.schemas.issue import Issue
from app.services import issue_memory


@strawberry.type
class Query:
    @strawberry.field
    def issues(self) -> list[Issue]:
        return issue_memory.get_issues()
```

## Mutation

```python
import strawberry

from app.graphql.schemas.issue import (
    CreateIssueInput,
    Issue,
)
from app.services import issue_memory


@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_issue(
        self,
        input: CreateIssueInput,
    ) -> Issue:
        return issue_memory.create_issue(
            input
        )
```

## GraphQL

```graphql
mutation {
  createIssue(
    input: {
      title: "First issue"
      description: "Memory CRUD"
    }
  ) {
    id
    title
    status
  }
}
```

---

# Step 4.1 — Backend Unit Test

## Install

```bash
uv add --dev pytest pytest-asyncio httpx
```

## `tests/test_issue_memory.py`

```python
from app.graphql.schemas.issue import (
    CreateIssueInput,
)
from app.services import issue_memory


def setup_function():
    issue_memory.issues.clear()


def test_create_issue():
    issue = issue_memory.create_issue(
        CreateIssueInput(
            title="Test",
            description="Unit Test",
        )
    )

    assert issue.id == 1
    assert issue.title == "Test"
    assert issue.status == "OPEN"


def test_get_issues():
    issue_memory.create_issue(
        CreateIssueInput(
            title="A",
        )
    )

    result = issue_memory.get_issues()

    assert len(result) == 1
    assert result[0].title == "A"
```

## Run

```bash
uv run pytest -q
```

---

# Step 5 — PostgreSQL + SQLAlchemy Async

## Docker PostgreSQL

`compose.yaml`

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Run:

```bash
docker compose up -d
```

## Install

```bash
uv add "sqlalchemy[asyncio]" asyncpg
```

## `src/app/database.py`

```python
from sqlalchemy.ext.asyncio import (
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase


DATABASE_URL = (
    "postgresql+asyncpg://"
    "app:password@localhost:5432/app"
)


class Base(DeclarativeBase):
    pass


engine = create_async_engine(
    DATABASE_URL,
    echo=True,
)


SessionLocal = async_sessionmaker(
    bind=engine,
    expire_on_commit=False,
)
```

## Important

```python
session.add(model)            # awaitしない
await session.get(...)
await session.execute(...)
await session.commit()
await session.refresh(model)
await session.delete(model)
await session.rollback()
```

---

# Step 5.1 — Issue Model

`src/app/models/issue.py`

```python
from datetime import datetime

from sqlalchemy import (
    DateTime,
    String,
    Text,
    func,
)
from sqlalchemy.orm import (
    Mapped,
    mapped_column,
)

from app.database import Base


class IssueModel(Base):
    __tablename__ = "issues"

    id: Mapped[int] = mapped_column(
        primary_key=True,
    )

    title: Mapped[str] = mapped_column(
        String(200),
        nullable=False,
    )

    description: Mapped[str | None] = (
        mapped_column(
            Text,
            nullable=True,
        )
    )

    status: Mapped[str] = mapped_column(
        String(20),
        nullable=False,
        default="OPEN",
    )

    created_at: Mapped[datetime] = (
        mapped_column(
            DateTime(timezone=True),
            server_default=func.now(),
            nullable=False,
        )
    )
```

---

# Step 5.2 — Create Tables

`src/app/create_tables.py`

```python
import asyncio

import app.models  # metadata登録用

from app.database import (
    Base,
    engine,
)


async def main():
    async with engine.begin() as conn:
        await conn.run_sync(
            Base.metadata.create_all
        )


asyncio.run(main())
```

Run:

```bash
uv run python -m app.create_tables
```

---

# Step 5.3 — Async CRUD Service

`src/app/services/issue_service.py`

```python
from sqlalchemy import select

from app.database import SessionLocal
from app.graphql.schemas.issue import (
    CreateIssueInput,
    Issue,
)
from app.models.issue import IssueModel


def to_issue(
    model: IssueModel,
) -> Issue:
    return Issue(
        id=model.id,
        title=model.title,
        description=model.description,
        status=model.status,
        created_at=model.created_at,
    )


async def create_issue(
    input: CreateIssueInput,
) -> Issue:
    async with SessionLocal() as session:
        model = IssueModel(
            title=input.title,
            description=input.description,
            status="OPEN",
        )

        session.add(model)

        await session.commit()
        await session.refresh(model)

        return to_issue(model)


async def get_issues() -> list[Issue]:
    async with SessionLocal() as session:
        result = await session.execute(
            select(IssueModel)
            .order_by(
                IssueModel.id.desc()
            )
        )

        models = result.scalars().all()

        return [
            to_issue(model)
            for model in models
        ]
```

---

# Step 5.4 — Async Resolver

```python
@strawberry.field
async def issues(
    self,
) -> list[Issue]:
    return await issue_service.get_issues()
```

```python
@strawberry.mutation
async def create_issue(
    self,
    input: CreateIssueInput,
) -> Issue:
    return await issue_service.create_issue(
        input
    )
```

---

# Step 6 — SQL / Index / Transaction Basics

## SQL確認

```sql
SELECT
    id,
    title,
    status,
    created_at
FROM issues
ORDER BY id DESC;
```

## Index

```sql
CREATE INDEX idx_issues_status
ON issues(status);
```

確認:

```sql
EXPLAIN ANALYZE
SELECT *
FROM issues
WHERE status = 'OPEN';
```

## Transaction

```python
async with SessionLocal() as session:
    async with session.begin():
        ...
```

この段階では「複数のDB更新を1つのUnitとして扱う」意味だけ理解する。  
Isolation / Lock / DeadlockはPhase 2で深掘りする。

---

# Step 7 — User Relation + DataLoader

## User Model

`src/app/models/user.py`

```python
from sqlalchemy import String
from sqlalchemy.orm import (
    Mapped,
    mapped_column,
    relationship,
)

from app.database import Base


class UserModel(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(
        primary_key=True,
    )

    name: Mapped[str] = mapped_column(
        String(100),
        nullable=False,
    )

    email: Mapped[str] = mapped_column(
        String(255),
        nullable=False,
        unique=True,
    )

    password_hash: Mapped[str] = (
        mapped_column(
            String(255),
            nullable=False,
        )
    )

    issues = relationship(
        "IssueModel",
        back_populates="owner",
    )
```

## Issue Relation

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship


owner_id: Mapped[int] = mapped_column(
    ForeignKey("users.id"),
    nullable=False,
)

owner: Mapped["UserModel"] = relationship(
    back_populates="issues",
)
```

---

# Step 7.1 — Strawberry User Type

```python
import strawberry


@strawberry.type
class User:
    id: int
    name: str
    email: str
```

Issue:

```python
@strawberry.type
class Issue:
    id: int
    owner_id: strawberry.Private[int]
    title: str
    description: str | None
    status: str
    created_at: datetime

    @strawberry.field
    async def owner(
        self,
        info: strawberry.Info,
    ) -> User:
        model = await (
            info.context
            .user_loader
            .load(self.owner_id)
        )

        if model is None:
            raise ValueError(
                "Owner not found"
            )

        return User(
            id=model.id,
            name=model.name,
            email=model.email,
        )
```

---

# Step 7.2 — GraphQL Context + DataLoader

```python
from fastapi import Request
from sqlalchemy import select
from strawberry.dataloader import DataLoader
from strawberry.fastapi import BaseContext

from app.database import SessionLocal
from app.models.user import UserModel


class GraphQLContext(BaseContext):
    def __init__(
        self,
        request: Request,
    ):
        super().__init__()

        self.request = request

        self.current_user: (
            UserModel | None
        ) = None

        self.user_loader = DataLoader(
            load_fn=self.load_users,
        )

    async def load_users(
        self,
        keys: list[int],
    ):
        async with SessionLocal() as session:
            result = await session.execute(
                select(UserModel)
                .where(
                    UserModel.id.in_(keys)
                )
            )

            users = (
                result.scalars().all()
            )

        user_map = {
            user.id: user
            for user in users
        }

        return [
            user_map.get(key)
            for key in keys
        ]


async def get_context(
    request: Request,
) -> GraphQLContext:
    return GraphQLContext(
        request=request
    )
```

---

# Step 7.3 — DataLoader確認

GraphQL:

```graphql
query {
  issues {
    id
    title
    owner {
      id
      name
    }
  }
}
```

SQL LogでOwner取得QueryがIssue件数分発行されず、Batchされることを確認する。

---

# Step 8 — JWT Authentication

## Install

```bash
uv add pyjwt passlib bcrypt
```

## JWT

`src/app/auth/jwt.py`

```python
from datetime import (
    datetime,
    timedelta,
    timezone,
)

import jwt


JWT_SECRET = "dev-secret"
JWT_ALGORITHM = "HS256"


def create_access_token(
    user_id: int,
) -> str:
    now = datetime.now(
        timezone.utc
    )

    payload = {
        "sub": str(user_id),
        "iat": now,
        "exp": now + timedelta(
            hours=1
        ),
    }

    return jwt.encode(
        payload,
        JWT_SECRET,
        algorithm=JWT_ALGORITHM,
    )


def decode_access_token(
    token: str,
) -> int | None:
    try:
        payload = jwt.decode(
            token,
            JWT_SECRET,
            algorithms=[
                JWT_ALGORITHM
            ],
        )

        return int(
            payload["sub"]
        )

    except Exception:
        return None
```

---

# Step 8.1 — Contextへcurrent_userを設定

```python
async def get_context(
    request: Request,
):
    context = GraphQLContext(
        request=request,
    )

    authorization = request.headers.get(
        "Authorization",
        "",
    )

    if authorization.startswith(
        "Bearer "
    ):
        token = authorization.removeprefix(
            "Bearer "
        ).strip()

        user_id = decode_access_token(
            token,
        )

        if user_id is not None:
            async with SessionLocal() as session:
                context.current_user = (
                    await session.get(
                        UserModel,
                        user_id,
                    )
                )

    return context
```

---

# Step 8.2 — Authenticated createIssue

```python
@strawberry.mutation
async def create_issue(
    self,
    info: strawberry.Info,
    input: CreateIssueInput,
) -> Issue:
    current_user = (
        info.context.current_user
    )

    if current_user is None:
        raise ValueError(
            "Authentication required"
        )

    return await issue_service.create_issue(
        input=input,
        owner_id=current_user.id,
    )
```

重要:

```python
(value)   # value
(value,)  # tuple
```

---

# Step 8.3 — GraphQL Integration Test

## Test App

`tests/conftest.py`

```python
import pytest
from httpx import (
    ASGITransport,
    AsyncClient,
)

from app.main import app


@pytest.fixture
async def client():
    transport = ASGITransport(
        app=app
    )

    async with AsyncClient(
        transport=transport,
        base_url="http://test",
    ) as client:
        yield client
```

## Authentication required

`tests/test_graphql_auth.py`

```python
import pytest


@pytest.mark.asyncio
async def test_create_issue_requires_auth(
    client,
):
    response = await client.post(
        "/graphql",
        json={
            "query": """
                mutation {
                  createIssue(
                    input: {
                      title: "Test"
                    }
                  ) {
                    id
                  }
                }
            """
        },
    )

    body = response.json()

    assert "errors" in body
    assert (
        body["errors"][0]["message"]
        == "Authentication required"
    )
```

> 実際のDBを使うIntegration Testでは、Test用DatabaseとFixtureを追加する。  
> Phase 3でRepositoryを分離した後はUnit TestとIntegration Testの境界がより明確になる。

---

# Step 9 — Next.js + Tailwind

## Create

```bash
npx create-next-app@latest frontend \
  --typescript \
  --eslint \
  --tailwind \
  --app \
  --src-dir
```

---

# Step 9.1 — GraphQL Client Helper

`frontend/src/lib/graphql.ts`

```typescript
const GRAPHQL_URL =
  process.env.NEXT_PUBLIC_GRAPHQL_URL
  ?? "http://localhost:8000/graphql";


type GraphQLResponse<T> = {
  data?: T;
  errors?: {
    message: string;
  }[];
};


export async function graphqlRequest<T>(
  query: string,
  variables?: Record<string, unknown>,
): Promise<T> {
  const token =
    typeof window !== "undefined"
      ? localStorage.getItem(
          "accessToken",
        )
      : null;

  const response = await fetch(
    GRAPHQL_URL,
    {
      method: "POST",
      headers: {
        "Content-Type":
          "application/json",

        ...(token
          ? {
              Authorization:
                `Bearer ${token}`,
            }
          : {}),
      },
      body: JSON.stringify({
        query,
        variables,
      }),
      cache: "no-store",
    },
  );

  const result:
    GraphQLResponse<T> =
      await response.json();

  if (result.errors?.length) {
    throw new Error(
      result.errors
        .map(
          (error) =>
            error.message
        )
        .join("\n"),
    );
  }

  if (!result.data) {
    throw new Error(
      "No GraphQL data"
    );
  }

  return result.data;
}
```

---

# Step 9.2 — Login UI

Flow:

```text
email / password
→ login Mutation
→ accessToken
→ localStorage
→ /issues
```

このPhaseではJWTの仕組みを理解しやすくするため`localStorage`を使用する。  
HttpOnly Cookie / Refresh Token / CSRFはPhase 5で実装する。

---

# Step 9.3 — Issue CRUD UI

最低限以下を画面で確認する。

```text
Read
Create
Update
Delete
Logout
```

Tailwindのみ使用し、UI Libraryは追加しない。

---

# Step 9.4 — Vitest

## Install

```bash
npm install -D \
  vitest \
  jsdom \
  @testing-library/react \
  @testing-library/jest-dom \
  @testing-library/user-event
```

## Pure Function例

`src/lib/issue.ts`

```typescript
export function normalizeTitle(
  title: string,
) {
  return title.trim();
}
```

`src/lib/issue.test.ts`

```typescript
import {
  describe,
  expect,
  it,
} from "vitest";

import {
  normalizeTitle,
} from "./issue";


describe(
  "normalizeTitle",
  () => {
    it(
      "trims spaces",
      () => {
        expect(
          normalizeTitle(
            "  Issue  ",
          ),
        ).toBe("Issue");
      },
    );
  },
);
```

Run:

```bash
npx vitest
```

---

# Step 9.5 — React Testing Library

Componentの内部stateではなく、Userが見る振る舞いをTestする。

例:

```tsx
import {
  render,
  screen,
} from "@testing-library/react";

import userEvent from (
  "@testing-library/user-event"
);


test(
  "title can be entered",
  async () => {
    const user = (
      userEvent.setup()
    );

    render(
      <CreateIssueForm />
    );

    const input =
      screen.getByPlaceholderText(
        "Title",
      );

    await user.type(
      input,
      "Test Issue",
    );

    expect(input).toHaveValue(
      "Test Issue"
    );
  },
);
```

---

# Step 9.6 — Playwright E2E

## Install

```bash
npm install -D @playwright/test
npx playwright install
```

## `playwright.config.ts`

```typescript
import {
  defineConfig,
} from "@playwright/test";


export default defineConfig({
  use: {
    baseURL:
      "http://localhost:3000",
  },
});
```

## `e2e/issues.spec.ts`

```typescript
import {
  expect,
  test,
} from "@playwright/test";


test(
  "login and create issue",
  async ({ page }) => {
    await page.goto(
      "/login"
    );

    await page
      .getByLabel("Email")
      .fill(
        "hwang@example.com"
      );

    await page
      .getByLabel("Password")
      .fill(
        "password123"
      );

    await page
      .getByRole(
        "button",
        {
          name: "Login",
        },
      )
      .click();

    await expect(
      page
        .getByRole(
          "heading",
          {
            name: "Issues",
          },
        )
    ).toBeVisible();

    await page
      .getByPlaceholder(
        "Title"
      )
      .fill(
        "E2E Issue"
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
        "E2E Issue"
      )
    ).toBeVisible();
  },
);
```

Run:

```bash
npx playwright test
```

---

# Step 10 — Filter / Search / Cursor Pagination

Backend Queryへ以下を追加する。

```text
status
search
cursor
limit
```

FrontendからFilterとSearchを操作し、Network TabでVariablesを確認する。

Testing:

```text
status filter
search keyword
next cursor
duplicate rowなし
```

---

# Step 11 — Docker

Phase 1では全体接続を確認するための入門。

```text
frontend
backend
postgres
```

をDockerで起動する。

Production用Image最適化・Secret・Deployment StrategyはPhase 4で扱う。

---

# Step 12 — AWS Entry

Phase 1ではServiceの役割だけ確認する。

```text
ECS
→ Application Container

RDS
→ PostgreSQL

S3 / CloudFront
→ Frontend Static Asset候補
```

実際のVPC / Terraform / CI/CD / RollbackはPhase 4へ進める。

---

# Phase 1 Completion Checklist

```text
[ ] FastAPI
[ ] Strawberry GraphQL
[ ] Memory CRUD
[ ] PostgreSQL
[ ] Async SQLAlchemy
[ ] SQL / Index基礎
[ ] User Relation
[ ] DataLoader
[ ] JWT
[ ] GraphQL Context
[ ] Next.js CRUD UI
[ ] Vitest
[ ] React Testing Library
[ ] Playwright
[ ] Filter / Search / Cursor
[ ] Docker入門
[ ] AWS入門
```
