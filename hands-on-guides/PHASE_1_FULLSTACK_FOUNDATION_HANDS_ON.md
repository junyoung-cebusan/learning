# Phase 1 — Full-Stack Foundation Hands-on Guide

> 実務に近い流れを優先し、最初からPostgreSQL + SQLAlchemy Asyncを使用する。  
> Memory CRUDは使用しない。  
> TestはBackend / Frontendの機能実装が一通り完成した後に追加する。

---

# 0. 完成イメージ

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
Backend
→ pytest / pytest-asyncio / httpx

Frontend
→ Vitest / React Testing Library

E2E
→ Playwright
```

---

# Step 1 — FastAPI

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
    return {"status": "ok"}
```

Run:

```bash
uv run uvicorn app.main:app --reload
```

---

# Step 2 — Strawberry GraphQL

## Install

```bash
uv add "strawberry-graphql[fastapi]"
```

## Query

```python
import strawberry


@strawberry.type
class Query:
    @strawberry.field
    def hello(self) -> str:
        return "Hello GraphQL"
```

## Schema

```python
import strawberry

from app.graphql.query import Query


schema = strawberry.Schema(
    query=Query,
)
```

## Router

```python
from strawberry.fastapi import GraphQLRouter

graphql_app = GraphQLRouter(schema)

app.include_router(
    graphql_app,
    prefix="/graphql",
)
```

GraphiQL:

```graphql
query {
  hello
}
```

---

# Step 3 — PostgreSQL + SQLAlchemy Async

## PostgreSQL

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

AsyncSession:

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

# Step 4 — Issue Model

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

    description: Mapped[str | None] = mapped_column(
        Text,
        nullable=True,
    )

    status: Mapped[str] = mapped_column(
        String(20),
        nullable=False,
        default="OPEN",
    )

    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )
```

---

# Step 5 — Create Tables

`src/app/create_tables.py`

```python
import asyncio

import app.models

from app.database import Base, engine


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

# Step 6 — GraphQL Issue Type / Input

`src/app/graphql/schemas/issue.py`

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


@strawberry.input
class UpdateIssueInput:
    title: str | None = None
    description: str | None = None
    status: str | None = None
```

---

# Step 7 — Issue CRUD Service

`src/app/services/issue_service.py`

```python
from sqlalchemy import select

from app.database import SessionLocal
from app.graphql.schemas.issue import (
    CreateIssueInput,
    Issue,
    UpdateIssueInput,
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

        return [
            to_issue(model)
            for model
            in result.scalars().all()
        ]


async def get_issue(
    issue_id: int,
) -> Issue | None:
    async with SessionLocal() as session:
        model = await session.get(
            IssueModel,
            issue_id,
        )

        if model is None:
            return None

        return to_issue(model)


async def update_issue(
    issue_id: int,
    input: UpdateIssueInput,
) -> Issue | None:
    async with SessionLocal() as session:
        model = await session.get(
            IssueModel,
            issue_id,
        )

        if model is None:
            return None

        if input.title is not None:
            model.title = input.title

        if input.description is not None:
            model.description = input.description

        if input.status is not None:
            model.status = input.status

        await session.commit()
        await session.refresh(model)

        return to_issue(model)


async def delete_issue(
    issue_id: int,
) -> bool:
    async with SessionLocal() as session:
        model = await session.get(
            IssueModel,
            issue_id,
        )

        if model is None:
            return True

        await session.delete(model)
        await session.commit()

        return True
```

---

# Step 8 — Query / Mutation

```python
@strawberry.type
class Query:
    @strawberry.field
    async def issues(
        self,
    ) -> list[Issue]:
        return await issue_service.get_issues()

    @strawberry.field
    async def issue(
        self,
        id: int,
    ) -> Issue | None:
        return await issue_service.get_issue(id)
```

```python
@strawberry.type
class Mutation:
    @strawberry.mutation
    async def create_issue(
        self,
        input: CreateIssueInput,
    ) -> Issue:
        return await issue_service.create_issue(
            input
        )

    @strawberry.mutation
    async def update_issue(
        self,
        id: int,
        input: UpdateIssueInput,
    ) -> Issue | None:
        return await issue_service.update_issue(
            id,
            input,
        )

    @strawberry.mutation
    async def delete_issue(
        self,
        id: int,
    ) -> bool:
        return await issue_service.delete_issue(
            id
        )
```

---

# Step 9 — GraphiQL CRUD

Create:

```graphql
mutation {
  createIssue(
    input: {
      title: "First issue"
      description: "PostgreSQL CRUD"
    }
  ) {
    id
    title
    status
  }
}
```

Read:

```graphql
query {
  issues {
    id
    title
    status
  }
}
```

Update:

```graphql
mutation {
  updateIssue(
    id: 1
    input: {
      status: "DONE"
    }
  ) {
    id
    status
  }
}
```

Delete:

```graphql
mutation {
  deleteIssue(id: 1)
}
```

---

# Step 10 — SQL / Index / Transaction Basics

```sql
SELECT
    id,
    title,
    status,
    created_at
FROM issues
ORDER BY id DESC;
```

Index:

```sql
CREATE INDEX idx_issues_status
ON issues(status);
```

```sql
EXPLAIN ANALYZE
SELECT *
FROM issues
WHERE status = 'OPEN';
```

Transaction:

```python
async with SessionLocal() as session:
    async with session.begin():
        ...
```

Isolation / Lock / DeadlockはPhase 2で扱う。

---

# Step 11 — User Relation + DataLoader

User Model:

```python
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

    password_hash: Mapped[str] = mapped_column(
        String(255),
        nullable=False,
    )

    issues = relationship(
        "IssueModel",
        back_populates="owner",
    )
```

Issue Relation:

```python
owner_id: Mapped[int] = mapped_column(
    ForeignKey("users.id"),
    nullable=False,
)

owner: Mapped["UserModel"] = relationship(
    back_populates="issues",
)
```

GraphQL Context:

```python
class GraphQLContext(BaseContext):
    def __init__(
        self,
        request: Request,
    ):
        super().__init__()

        self.request = request
        self.current_user: UserModel | None = None

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

            users = result.scalars().all()

        user_map = {
            user.id: user
            for user in users
        }

        return [
            user_map.get(key)
            for key in keys
        ]
```

---

# Step 12 — JWT Authentication

Install:

```bash
uv add pyjwt passlib bcrypt
```

Create:

```python
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
```

Decode:

```python
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

        return int(payload["sub"])

    except Exception:
        return None
```

---

# Step 13 — current_user

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
            token
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

Authenticated Mutation:

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

---

# Step 14 — Next.js + Tailwind

```bash
npx create-next-app@latest frontend \
  --typescript \
  --eslint \
  --tailwind \
  --app \
  --src-dir
```

Pages:

```text
/login
/issues
```

Flow:

```text
Login
→ JWT保存
→ Issue List
→ Create
→ Update
→ Delete
→ Logout
```

Phase 1では`localStorage`を使用する。  
HttpOnly Cookie / Refresh Token / CSRFはPhase 5で扱う。

---

# Step 15 — GraphQL Request Helper

```typescript
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
    process.env
      .NEXT_PUBLIC_GRAPHQL_URL!,
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
    },
  );

  const result = await response.json();

  if (result.errors?.length) {
    throw new Error(
      result.errors[0].message
    );
  }

  return result.data;
}
```

---

# Step 16 — Filter / Search / Cursor Pagination

Backend:

```text
status
search
cursor
limit
```

Frontend:

```text
Filter
Search
Next Cursor
```

大量Data最適化はPhase 2で深掘りする。

---

# Step 17 — Backend Testing

完成したBackend実装を対象にTestを追加する。

Install:

```bash
uv add --dev \
  pytest \
  pytest-asyncio \
  httpx
```

Test Client:

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

Authentication Test:

```python
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

Authenticated Integration Flow:

```text
register
→ login
→ token
→ Authorization Header
→ createIssue
→ owner確認
```

Test用Databaseは開発DBと分離する。

---

# Step 18 — Frontend Unit / Component Test

Install:

```bash
npm install -D \
  vitest \
  jsdom \
  @testing-library/react \
  @testing-library/jest-dom \
  @testing-library/user-event
```

Vitest:

```typescript
export function normalizeTitle(
  title: string,
) {
  return title.trim();
}
```

```typescript
import {
  expect,
  test,
} from "vitest";

import {
  normalizeTitle,
} from "./issue";


test(
  "trims title",
  () => {
    expect(
      normalizeTitle(
        "  Issue  ",
      ),
    ).toBe("Issue");
  },
);
```

React Testing Library:

```tsx
test(
  "user can enter title",
  async () => {
    const user =
      userEvent.setup();

    render(
      <CreateIssueForm />
    );

    const input =
      screen.getByPlaceholderText(
        "Title",
      );

    await user.type(
      input,
      "New Issue",
    );

    expect(input).toHaveValue(
      "New Issue"
    );
  },
);
```

---

# Step 19 — Playwright E2E

Install:

```bash
npm install -D @playwright/test
npx playwright install
```

Scenario:

```text
Login
→ Issue List
→ Create
→ Update
→ Delete
→ Logout
```

Example:

```typescript
test(
  "issue CRUD flow",
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

---

# Step 20 — Docker Integration

Phase 1では以下をまとめて起動できればよい。

```text
frontend
backend
postgres
```

Production向け最適化はPhase 4で扱う。

---

# Step 21 — AWS Entry

Phase 1では役割だけ理解する。

```text
ECS
→ Backend Container

RDS
→ PostgreSQL

S3 / CloudFront
→ Static Asset候補
```

VPC / Terraform / CI/CD / RollbackはPhase 4で扱う。

---

# Phase 1 Completion Checklist

```text
[ ] FastAPI
[ ] Strawberry GraphQL
[ ] PostgreSQL
[ ] SQLAlchemy Async
[ ] Issue CRUD
[ ] SQL / Index / Transaction基礎
[ ] User Relation
[ ] DataLoader
[ ] JWT
[ ] GraphQL Context
[ ] Next.js + Tailwind
[ ] Login / CRUD / Logout UI
[ ] Filter / Search / Cursor
[ ] pytest / pytest-asyncio / httpx
[ ] Vitest
[ ] React Testing Library
[ ] Playwright
[ ] Docker Integration
[ ] AWS Entry
```
