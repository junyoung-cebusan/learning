# GraphQL Full-Stack 1〜12段階ハンズオンガイド — Async検証版

> 目標: `src/app/graphql`, `src/app/graphql/schemas`, `models`, `services` 構成を維持しながら実際に手を動かして入力し  
> FastAPI → Strawberry GraphQL → SQLAlchemy Async → PostgreSQL → DataLoader → JWT → Next.js → Docker → AWS 流れを身につける。
>
> **重要:** 5段階でPostgreSQLを接続した時点から 12段階までSQLAlchemyは**Async専用**で使用する。
> Sync `Session`のコードと`AsyncSession`のコードを混在させない。

---

# 最終構成

```text
fullstack-relearn/
├── backend/
│   ├── pyproject.toml
│   └── src/
│       └── app/
│           ├── __init__.py
│           ├── main.py
│           ├── database.py
│           ├── security.py
│           ├── models/
│           │   ├── __init__.py
│           │   ├── issue.py
│           │   └── user.py
│           ├── graphql/
│           │   ├── __init__.py
│           │   ├── query.py
│           │   ├── mutation.py
│           │   ├── context.py
│           │   └── schemas/
│           │       ├── __init__.py
│           │       ├── issue.py
│           │       ├── user.py
│           │       └── auth.py
│           └── services/
│               ├── __init__.py
│               ├── issues.py
│               └── auth.py
├── frontend/
│   ├── codegen.ts
│   └── src/
│       ├── app/
│       │   ├── page.tsx
│       │   ├── register/
│       │   │   └── page.tsx
│       │   ├── login/
│       │   │   └── page.tsx
│       │   └── issues/
│       │       └── page.tsx
│       ├── graphql/
│       ├── generated/
│       │   └── graphql.ts
│       └── lib/
│           └── graphql-client.ts
├── docker-compose.yml
└── .github/
    └── workflows/
        └── ci.yml
```

レイヤー:

```text
graphql/  → Query / Mutation Resolver
graphql/schemas/ → GraphQL Type / Input
services/ → Business Logic / DB Access
models/   → SQLAlchemy ORM Model
```

最終Request Flow:

```text
Browser
  ↓
Next.js
  ↓
POST /graphql
  ↓
Query / Mutation Resolver
  ↓
Service
  ↓
SQLAlchemy AsyncSession
  ↓
PostgreSQL
```

---

# 0. Project Setup

## `pyproject.toml`

Project名は`backend`、実際のimport packageは`app`として使用する。

```toml
[project]
name = "backend"
version = "0.1.0"
requires-python = ">=3.12"

[build-system]
requires = ["uv_build"]
build-backend = "uv_build"

[tool.uv.build-backend]
module-name = "app"
module-root = "src"
```

インストール:

```bash
uv add fastapi uvicorn
uv add "strawberry-graphql[fastapi]"
uv add "sqlalchemy[asyncio]"
uv add asyncpg
```

実行:

```bash
uv run uvicorn app.main:app --reload
```

---

# 1段階 — FastAPI

## `src/app/main.py`

```python
from fastapi import FastAPI


app = FastAPI()


@app.get("/health")
def health():
    return {
        "status": "ok",
    }
```

実行:

```bash
uv run uvicorn app.main:app --reload
```

確認:

```text
http://localhost:8000/health
```

---

# 2段階 — Strawberry GraphQL

## `graphql/query.py`

```python
import strawberry


@strawberry.type
class Query:
    @strawberry.field
    def hello(self) -> str:
        return "Hello GraphQL"
```

## `graphql/mutation.py`

まだ実際のMutationはないため、確認用の`ping`を用意する。

```python
import strawberry


@strawberry.type
class Mutation:
    @strawberry.mutation
    def ping(self) -> str:
        return "pong"
```

## `main.py`

```python
import strawberry

from fastapi import FastAPI
from strawberry.fastapi import GraphQLRouter

from app.graphql.mutation import Mutation
from app.graphql.query import Query


schema = strawberry.Schema(
    query=Query,
    mutation=Mutation,
)

graphql_app = GraphQLRouter(
    schema,
)

app = FastAPI()


@app.get("/health")
def health():
    return {
        "status": "ok",
    }


app.include_router(
    graphql_app,
    prefix="/graphql",
)
```

## GraphiQL確認

```text
http://localhost:8000/graphql
```

Query:

```graphql
query {
  hello
}
```

Mutation:

```graphql
mutation {
  ping
}
```

---

# 3段階 — Issue Query

まだDBは接続しない。

## `graphql/schemas/issue.py`

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

> `status`のタイプミスに注意する。`statue`ではない。

## `services/issues.py`

```python
from datetime import (
    datetime,
    timezone,
)

from app.graphql.schemas.issue import Issue


issues: list[Issue] = [
    Issue(
        id=1,
        title="GraphQL学習",
        description="Resolver復習",
        status="OPEN",
        created_at=datetime.now(
            timezone.utc,
        ),
    ),
]


def get_issues() -> list[Issue]:
    return issues


def get_issue(
    issue_id: int,
) -> Issue | None:
    for issue in issues:
        if issue.id == issue_id:
            return issue

    return None
```

## `graphql/query.py`

```python
import strawberry

from app.graphql.schemas.issue import Issue
from app.services import issues as issue_service


@strawberry.type
class Query:
    @strawberry.field
    def issues(
        self,
    ) -> list[Issue]:
        return issue_service.get_issues()

    @strawberry.field
    def issue(
        self,
        id: int,
    ) -> Issue | None:
        return issue_service.get_issue(
            id,
        )
```

## GraphiQL確認 — Read

Read List:

```graphql
query {
  issues {
    id
    title
    description
    status
    createdAt
  }
}
```

Read Detail:

```graphql
query {
  issue(id: 1) {
    id
    title
    description
    status
    createdAt
  }
}
```

Not Found:

```graphql
query {
  issue(id: 999) {
    id
    title
  }
}
```

---

# 4段階 — Mutation / Memory CRUD

この段階まではDBがないため、sync関数で十分である。

## `services/issues.py`

```python
from app.graphql.schemas.issue import (
    CreateIssueInput,
    Issue,
    UpdateIssueInput,
)


def create_issue(
    input: CreateIssueInput,
) -> Issue:
    next_id = max(
        (
            issue.id
            for issue in issues
        ),
        default=0,
    ) + 1

    issue = Issue(
        id=next_id,
        title=input.title,
        description=input.description,
        status="OPEN",
        created_at=datetime.now(
            timezone.utc,
        ),
    )

    issues.append(issue)

    return issue


def update_issue(
    issue_id: int,
    input: UpdateIssueInput,
) -> Issue | None:
    issue = get_issue(
        issue_id,
    )

    if issue is None:
        return None

    if input.title is not None:
        issue.title = input.title

    if input.description is not None:
        issue.description = input.description

    if input.status is not None:
        issue.status = input.status

    return issue


def delete_issue(
    issue_id: int,
) -> bool:
    issue = get_issue(
        issue_id,
    )

    if issue is None:
        return True

    issues.remove(issue)

    return True
```

## `graphql/mutation.py`

```python
import strawberry

from app.graphql.schemas.issue import (
    CreateIssueInput,
    Issue,
    UpdateIssueInput,
)
from app.services import issues as issue_service


@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_issue(
        self,
        input: CreateIssueInput,
    ) -> Issue:
        return issue_service.create_issue(
            input,
        )

    @strawberry.mutation
    def update_issue(
        self,
        id: int,
        input: UpdateIssueInput,
    ) -> Issue | None:
        return issue_service.update_issue(
            id,
            input,
        )

    @strawberry.mutation
    def delete_issue(
        self,
        id: int,
    ) -> bool:
        return issue_service.delete_issue(
            id,
        )
```

## GraphiQL確認 — Memory CRUD

### Create

```graphql
mutation {
  createIssue(input: { title: "Memory CRUD", description: "Create確認" }) {
    id
    title
    description
    status
    createdAt
  }
}
```

### Read List

```graphql
query {
  issues {
    id
    title
    description
    status
    createdAt
  }
}
```

### Read Detail

```graphql
query {
  issue(id: 2) {
    id
    title
    description
    status
  }
}
```

### Update

```graphql
mutation {
  updateIssue(id: 2, input: { title: "Memory CRUD更新", status: "DONE" }) {
    id
    title
    description
    status
    createdAt
  }
}
```

### Delete

```graphql
mutation {
  deleteIssue(id: 2)
}
```

### Delete確認

```graphql
query {
  issue(id: 2) {
    id
  }
}
```

---

# 4段階拡張 — Backend Unit Test入門

> 既存の1〜4段階の順序は変更しない。
> Memory CRUDが完成した時点で、最初の実行可能なUnit Testを追加する。

## Test Tool

```bash
uv add --dev pytest pytest-asyncio httpx
```

この段階ではまず`services/issues.py`だけを直接Testする。
GraphQLやDBはまだTest対象に含めない。

作成するFile:

```text
backend/
└── tests/
    └── test_issue_service.py
```

## `tests/test_issue_service.py`

```python
from datetime import (
    datetime,
    timezone,
)

from app.graphql.schemas.issue import (
    CreateIssueInput,
    Issue,
    UpdateIssueInput,
)
from app.services import issues as issue_service


def setup_function() -> None:
    issue_service.issues.clear()
    issue_service.issues.append(
        Issue(
            id=1,
            title="GraphQL学習",
            description="Resolver復習",
            status="OPEN",
            created_at=datetime.now(
                timezone.utc,
            ),
        )
    )


def test_create_issue() -> None:
    created = issue_service.create_issue(
        CreateIssueInput(
            title="pytest学習",
            description="Unit Test",
        )
    )

    assert created.id == 2
    assert created.title == "pytest学習"
    assert created.status == "OPEN"
    assert len(issue_service.get_issues()) == 2


def test_update_issue() -> None:
    updated = issue_service.update_issue(
        1,
        UpdateIssueInput(
            status="DONE",
        ),
    )

    assert updated is not None
    assert updated.status == "DONE"


def test_delete_issue() -> None:
    deleted = issue_service.delete_issue(1)

    assert deleted is True
    assert issue_service.get_issue(1) is None
```

実行:

```bash
cd backend
uv run pytest tests/test_issue_service.py -q
```

期待結果:

```text
3 passed
```

ここではArrange / Act / Assertと、Frameworkを経由せずServiceの振る舞いを直接Testする感覚を確認する。

---

# 5段階 — PostgreSQL + SQLAlchemy Async

> **この段階から最後までAsync SQLAlchemyのみを使用する。**
> 4段階拡張の`tests/test_issue_service.py`はMemory CRUD専用Testであり、この段階でService実装をAsync DB版へ置き換えると対象実装が消える。
> そのため学習Checkpointとして確認後に削除し、8段階拡張のGraphQL Integration Testへ置き換える。

## PostgreSQL

rootの`docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: password
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

実行:

```bash
docker compose up -d
```

## `database.py`

```python
import os

from sqlalchemy.ext.asyncio import (
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase


DATABASE_URL = os.getenv(
    "DATABASE_URL",
    "postgresql+asyncpg://app:password@localhost:5432/app",
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

## AsyncSessionルール

```python
session.add(model)            # await X

await session.get(...)        # await O
await session.execute(...)    # await O
await session.commit()        # await O
await session.refresh(model)  # await O
await session.delete(model)   # await O
await session.rollback()      # await O
```

誤ったコード:

```python
await session.add(model)
```

誤ったコード:

```python
await await session.commit()
```

誤ったコード:

```python
with SessionLocal() as session:
```

正しいコード:

```python
async with SessionLocal() as session:
```

---

## `models/issue.py`

7段階までは`owner_id`を作成しない。

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

    description: Mapped[
        str | None
    ] = mapped_column(
        Text,
        nullable=True,
    )

    status: Mapped[str] = mapped_column(
        String(30),
        nullable=False,
        default="OPEN",
    )

    created_at: Mapped[
        datetime
    ] = mapped_column(
        DateTime(
            timezone=True,
        ),
        nullable=False,
        server_default=func.now(),
    )
```

## `models/__init__.py`

```python
from app.models.issue import IssueModel


__all__ = [
    "IssueModel",
]
```

## `main.py` lifespan

```python
from contextlib import asynccontextmanager

import strawberry

from fastapi import FastAPI
from strawberry.fastapi import GraphQLRouter

import app.models

from app.database import (
    Base,
    engine,
)
from app.graphql.mutation import Mutation
from app.graphql.query import Query


@asynccontextmanager
async def lifespan(
    app: FastAPI,
):
    async with engine.begin() as conn:
        await conn.run_sync(
            Base.metadata.create_all,
        )

    yield

    await engine.dispose()


schema = strawberry.Schema(
    query=Query,
    mutation=Mutation,
)

graphql_app = GraphQLRouter(
    schema,
)

app = FastAPI(
    lifespan=lifespan,
)


@app.get("/health")
def health():
    return {
        "status": "ok",
    }


app.include_router(
    graphql_app,
    prefix="/graphql",
)
```

## `services/issues.py` — Async CRUDへ置き換え

```python
from sqlalchemy import select

from app.database import SessionLocal
from app.models.issue import IssueModel
from app.graphql.schemas.issue import (
    CreateIssueInput,
    Issue,
    UpdateIssueInput,
)


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


async def get_issues() -> list[Issue]:
    async with SessionLocal() as session:
        result = await session.execute(
            select(IssueModel)
            .order_by(
                IssueModel.id,
            )
        )

        models = (
            result
            .scalars()
            .all()
        )

        return [
            to_issue(model)
            for model in models
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

## `graphql/query.py` — すべてasync

```python
import strawberry

from app.graphql.schemas.issue import Issue
from app.services import issues as issue_service


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
        return await issue_service.get_issue(
            id,
        )
```

## `graphql/mutation.py` — すべてasync

```python
import strawberry

from app.graphql.schemas.issue import (
    CreateIssueInput,
    Issue,
    UpdateIssueInput,
)
from app.services import issues as issue_service


@strawberry.type
class Mutation:
    @strawberry.mutation
    async def create_issue(
        self,
        input: CreateIssueInput,
    ) -> Issue:
        return await issue_service.create_issue(
            input,
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
            id,
        )
```

> DB段階以降、`create_issue`、`update_issue`、`delete_issue` Resolverはすべて`async def`とする。

## GraphiQL確認 — PostgreSQL CRUD

Create:

```graphql
mutation {
  createIssue(input: { title: "PostgreSQL CRUD", description: "INSERT確認" }) {
    id
    title
    description
    status
    createdAt
  }
}
```

Read List:

```graphql
query {
  issues {
    id
    title
    status
    createdAt
  }
}
```

Read Detail:

```graphql
query {
  issue(id: 1) {
    id
    title
    description
    status
  }
}
```

Update:

```graphql
mutation {
  updateIssue(id: 1, input: { title: "PostgreSQL CRUD更新", status: "DONE" }) {
    id
    title
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

# 6段階 — SQL / Index / Transaction

DB接続:

```bash
docker compose exec postgres \
  psql -U app -d app
```

Query:

```sql
SELECT *
FROM issues;
```

Index:

```sql
CREATE INDEX ix_issues_status_created
ON issues (
    status,
    created_at DESC
);
```

実行計画:

```sql
EXPLAIN ANALYZE
SELECT *
FROM issues
WHERE status = 'OPEN'
ORDER BY created_at DESC;
```

Transaction:

```sql
BEGIN;

UPDATE issues
SET status = 'DONE'
WHERE id = 1;

ROLLBACK;
```

もう一度:

```sql
BEGIN;

UPDATE issues
SET status = 'DONE'
WHERE id = 1;

COMMIT;
```

## SQLAlchemy transaction例

```python
async with SessionLocal() as session:
    try:
        # 複数のDB処理
        await session.commit()

    except Exception:
        await session.rollback()
        raise
```

## GraphiQL確認 — CRUD + SQL Log

Create:

```graphql
mutation {
  createIssue(
    input: { title: "SQL確認用Issue", description: "INSERT Log確認" }
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
  updateIssue(id: 1, input: { status: "DONE" }) {
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

Log:

```text
Create → INSERT
Read   → SELECT
Update → UPDATE
Delete → DELETE
```

---

# 7段階 — User Relation / N+1 / DataLoader

Relation:

```text
User 1 ─── N Issue
```

この段階で初めて`owner_id`を追加する。

学習中のため、migrationの代わりにDBを初期化してもよい。

```bash
docker compose down -v
docker compose up -d
```

## `models/user.py`

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
        String(320),
        unique=True,
        nullable=False,
    )

    issues: Mapped[
        list["IssueModel"]
    ] = relationship(
        back_populates="owner",
    )
```

## `models/issue.py`

追加:

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship
```

Class内部:

```python
owner_id: Mapped[int] = mapped_column(
    ForeignKey("users.id"),
    nullable=False,
)

owner: Mapped[
    "UserModel"
] = relationship(
    back_populates="issues",
)
```

## `models/__init__.py`

```python
from app.models.issue import IssueModel
from app.models.user import UserModel


__all__ = [
    "IssueModel",
    "UserModel",
]
```

## `graphql/schemas/user.py`

```python
import strawberry


@strawberry.type
class User:
    id: int
    name: str
    email: str
```

## `graphql/schemas/issue.py`

> `owner()`は**必ず`Issue` Class内部にインデントして定義する。**

```python
from datetime import datetime

import strawberry

from app.graphql.schemas.user import User


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
            .load(
                self.owner_id,
            )
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

## `to_issue()`変更

```python
def to_issue(
    model: IssueModel,
) -> Issue:
    return Issue(
        id=model.id,
        owner_id=model.owner_id,
        title=model.title,
        description=model.description,
        status=model.status,
        created_at=model.created_at,
    )
```

## `graphql/context.py`

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
                    UserModel.id.in_(
                        keys,
                    )
                )
            )

            users = (
                result
                .scalars()
                .all()
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
        request=request,
    )
```

## `main.py`

```python
from app.graphql.context import get_context
```

```python
graphql_app = GraphQLRouter(
    schema,
    context_getter=get_context,
)
```

## JWT導入前の一時User seed

`owner_id`が`NOT NULL`のため、先にUserが必要である。

```sql
INSERT INTO users (
    name,
    email
)
VALUES (
    'Hwang',
    'hwang@example.com'
);
```

確認:

```sql
SELECT *
FROM users;
```

user idを`1`と仮定する。

JWT導入前だけ`create_issue()`に一時的なowner idを設定する。

```python
model = IssueModel(
    title=input.title,
    description=input.description,
    status="OPEN",
    owner_id=1,
)
```

> 8段階ではこのハードコードを削除し、`current_user.id`を使用する。

## GraphiQL確認 — Relation / DataLoader

Create:

```graphql
mutation {
  createIssue(
    input: { title: "DataLoader Issue", description: "Owner relation" }
  ) {
    id
    title
    owner {
      id
      name
      email
    }
  }
}
```

Nested Query:

```graphql
query {
  issues {
    id
    title
    owner {
      id
      name
      email
    }
  }
}
```

目標SQL:

```text
issues SELECT 1回
users SELECT ... WHERE id IN (...) 1回
```

---

# 8段階 — JWT Authentication / Authorization

インストール:

```bash
uv add pyjwt "pwdlib[argon2]"
```

## `models/user.py`

追加:

```python
password_hash: Mapped[str] = mapped_column(
    String(512),
    nullable=False,
)
```

学習中であればDBを再初期化する。

## `security.py`

```python
import os

from datetime import (
    datetime,
    timedelta,
    timezone,
)

import jwt

from pwdlib import PasswordHash


JWT_SECRET = os.getenv(
    "JWT_SECRET",
    "dev-secret-change-me",
)
JWT_ALGORITHM = "HS256"

password_hash = (
    PasswordHash.recommended()
)


def hash_password(
    password: str,
) -> str:
    return password_hash.hash(
        password,
    )


def verify_password(
    password: str,
    hashed_password: str,
) -> bool:
    return password_hash.verify(
        password,
        hashed_password,
    )


def create_access_token(
    user_id: int,
) -> str:
    now = datetime.now(
        timezone.utc,
    )

    payload = {
        "sub": str(user_id),
        "iat": now,
        "exp": now + timedelta(
            hours=1,
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
                JWT_ALGORITHM,
            ],
        )

        return int(
            payload["sub"]
        )

    except Exception:
        return None
```

Local学習ではfallback値を使えるが、AWSへDeployするときは`JWT_SECRET`をEnvironment Variableで必ず上書きする。

## `graphql/schemas/auth.py`

```python
import strawberry

from app.graphql.schemas.user import User


@strawberry.input
class RegisterInput:
    name: str
    email: str
    password: str


@strawberry.input
class LoginInput:
    email: str
    password: str


@strawberry.type
class AuthPayload:
    access_token: str
    user: User
```

## `services/auth.py`

```python
from sqlalchemy import select

from app.database import SessionLocal
from app.models.user import UserModel
from app.graphql.schemas.auth import (
    AuthPayload,
    LoginInput,
    RegisterInput,
)
from app.graphql.schemas.user import User
from app.security import (
    create_access_token,
    hash_password,
    verify_password,
)


async def register(
    input: RegisterInput,
) -> AuthPayload:
    async with SessionLocal() as session:
        email = input.email.lower()

        result = await session.execute(
            select(UserModel).where(
                UserModel.email == email
            )
        )

        if result.scalar_one_or_none() is not None:
            raise ValueError(
                "Email already registered"
            )

        model = UserModel(
            name=input.name,
            email=email,
            password_hash=hash_password(
                input.password,
            ),
        )

        session.add(model)

        await session.commit()
        await session.refresh(model)

        return AuthPayload(
            access_token=create_access_token(
                model.id,
            ),
            user=User(
                id=model.id,
                name=model.name,
                email=model.email,
            ),
        )


async def login(
    input: LoginInput,
) -> AuthPayload | None:
    async with SessionLocal() as session:
        result = await session.execute(
            select(UserModel)
            .where(
                UserModel.email
                == input.email.lower()
            )
        )

        model = result.scalar_one_or_none()

        if model is None:
            return None

        if not verify_password(
            input.password,
            model.password_hash,
        ):
            return None

        return AuthPayload(
            access_token=create_access_token(
                model.id,
            ),
            user=User(
                id=model.id,
                name=model.name,
                email=model.email,
            ),
        )
```

## `graphql/context.py` — JWT適用

```python
from app.security import decode_access_token
```

`get_context()`:

```python
async def get_context(
    request: Request,
) -> GraphQLContext:
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

## `services/issues.py` — ownerベースCRUD

8段階からRead / Create / Update / Deleteをすべてlogin user基準にする。
7段階までの`get_issues()` / `get_issue()`もここで置き換える。

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
        owner_id=model.owner_id,
        title=model.title,
        description=model.description,
        status=model.status,
        created_at=model.created_at,
    )


async def get_issues(
    current_user_id: int,
) -> list[Issue]:
    async with SessionLocal() as session:
        result = await session.execute(
            select(IssueModel)
            .where(
                IssueModel.owner_id
                == current_user_id
            )
            .order_by(IssueModel.id)
        )

        return [
            to_issue(model)
            for model in (
                result.scalars().all()
            )
        ]


async def get_issue(
    issue_id: int,
    current_user_id: int,
) -> Issue | None:
    async with SessionLocal() as session:
        model = await session.get(
            IssueModel,
            issue_id,
        )

        if model is None:
            return None

        if model.owner_id != current_user_id:
            raise ValueError(
                "Forbidden"
            )

        return to_issue(model)


async def create_issue(
    input: CreateIssueInput,
    owner_id: int,
) -> Issue:
    async with SessionLocal() as session:
        model = IssueModel(
            title=input.title,
            description=input.description,
            status="OPEN",
            owner_id=owner_id,
        )

        session.add(model)

        await session.commit()
        await session.refresh(model)

        return to_issue(model)


async def update_issue(
    issue_id: int,
    input: UpdateIssueInput,
    current_user_id: int,
) -> Issue | None:
    async with SessionLocal() as session:
        model = await session.get(
            IssueModel,
            issue_id,
        )

        if model is None:
            return None

        if model.owner_id != current_user_id:
            raise ValueError(
                "Forbidden"
            )

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
    current_user_id: int,
) -> bool:
    async with SessionLocal() as session:
        model = await session.get(
            IssueModel,
            issue_id,
        )

        if model is None:
            return True

        if model.owner_id != current_user_id:
            raise ValueError(
                "Forbidden"
            )

        await session.delete(model)
        await session.commit()

        return True
```

## `graphql/query.py` — 認証Read

MutationだけでなくQueryもlogin userに限定する。
これにより`/issues`で他UserのIssueを取得しない。

```python
import strawberry

from app.graphql.schemas.issue import Issue
from app.services import issues as issue_service


@strawberry.type
class Query:
    @strawberry.field
    async def issues(
        self,
        info: strawberry.Info,
    ) -> list[Issue]:
        current_user = (
            info.context.current_user
        )

        if current_user is None:
            raise ValueError(
                "Authentication required"
            )

        return await issue_service.get_issues(
            current_user_id=current_user.id,
        )

    @strawberry.field
    async def issue(
        self,
        info: strawberry.Info,
        id: int,
    ) -> Issue | None:
        current_user = (
            info.context.current_user
        )

        if current_user is None:
            raise ValueError(
                "Authentication required"
            )

        return await issue_service.get_issue(
            issue_id=id,
            current_user_id=current_user.id,
        )
```

## `graphql/mutation.py` — 認証CRUD

```python
import strawberry

from app.graphql.schemas.auth import (
    AuthPayload,
    LoginInput,
    RegisterInput,
)
from app.graphql.schemas.issue import (
    CreateIssueInput,
    Issue,
    UpdateIssueInput,
)
from app.services import auth as auth_service
from app.services import issues as issue_service


@strawberry.type
class Mutation:
    @strawberry.mutation
    async def register(
        self,
        input: RegisterInput,
    ) -> AuthPayload:
        return await auth_service.register(
            input,
        )

    @strawberry.mutation
    async def login(
        self,
        input: LoginInput,
    ) -> AuthPayload | None:
        return await auth_service.login(
            input,
        )

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

    @strawberry.mutation
    async def update_issue(
        self,
        info: strawberry.Info,
        id: int,
        input: UpdateIssueInput,
    ) -> Issue | None:
        current_user = (
            info.context.current_user
        )

        if current_user is None:
            raise ValueError(
                "Authentication required"
            )

        return await issue_service.update_issue(
            issue_id=id,
            input=input,
            current_user_id=current_user.id,
        )

    @strawberry.mutation
    async def delete_issue(
        self,
        info: strawberry.Info,
        id: int,
    ) -> bool:
        current_user = (
            info.context.current_user
        )

        if current_user is None:
            raise ValueError(
                "Authentication required"
            )

        return await issue_service.delete_issue(
            issue_id=id,
            current_user_id=current_user.id,
        )
```

## Logout

現在はaccess-token-onlyのstateless JWTなので、Server Mutationは必須ではない。

```text
Logout
→ Clientでaccess tokenを削除
```

Next.js:

```typescript
localStorage.removeItem("accessToken");
```

Refresh Token / Redis sessionを追加するSenior段階ではserver-side revoke/logoutを扱う。

## GraphiQL確認 — JWT CRUD

Register:

```graphql
mutation {
  register(
    input: {
      name: "Hwang"
      email: "hwang@example.com"
      password: "password123"
    }
  ) {
    accessToken
    user {
      id
      name
      email
    }
  }
}
```

Login:

```graphql
mutation {
  login(input: { email: "hwang@example.com", password: "password123" }) {
    accessToken
    user {
      id
      name
      email
    }
  }
}
```

Headers:

```json
{
  "Authorization": "Bearer YOUR_TOKEN"
}
```

Create:

```graphql
mutation {
  createIssue(
    input: { title: "JWT CRUD", description: "Authenticated Create" }
  ) {
    id
    title
    owner {
      id
      name
    }
  }
}
```

Update:

```graphql
mutation {
  updateIssue(id: 1, input: { status: "DONE" }) {
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

Read:

```graphql
query {
  issues {
    id
    title
    description
    status
    owner {
      id
      name
    }
  }
}
```

`issues` Queryにも同じ`Authorization` Headerを付ける。
別Userでloginした場合、そのUser自身のIssueだけが返ることを確認する。

失敗ケース:

```text
Headerなし        → Authentication失敗
不正なJWT         → Authentication失敗
他UserのIssue  → Authorization失敗
```

---

# 8段階拡張 — Authentication / GraphQL Integration Test

JWTまで実装した後は、実際のGraphQL HTTP Requestを通して認証FlowをTestする。

このTestでは次の層をまとめて通す。

```text
HTTP
→ FastAPI
→ Strawberry GraphQL
→ Context
→ Service
→ PostgreSQL
```

## Test用Database

普段の開発DBとTestデータを混ぜないため、Test用DBを作成する。

```bash
docker compose exec postgres \
  createdb -U app app_test
```

5段階の`database.py`ですでに`DATABASE_URL`を環境変数から読めるため、Test実行時だけ`app_test`へ切り替える。

## `tests/test_graphql_auth.py`

各Testの前にTest DBのTableを作り直すため、前の実行結果に依存しない。

```python
import pytest
import pytest_asyncio

from httpx import (
    ASGITransport,
    AsyncClient,
)

import app.models

from app.database import (
    Base,
    engine,
)
from app.main import app


@pytest_asyncio.fixture(
    autouse=True,
)
async def reset_database():
    async with engine.begin() as conn:
        await conn.run_sync(
            Base.metadata.drop_all,
        )
        await conn.run_sync(
            Base.metadata.create_all,
        )

    yield


@pytest.mark.asyncio
async def test_register_duplicate_email() -> None:
    transport = ASGITransport(
        app=app,
    )

    async with AsyncClient(
        transport=transport,
        base_url="http://test",
    ) as client:
        mutation = """
        mutation Register($input: RegisterInput!) {
          register(input: $input) {
            accessToken
            user {
              id
              email
            }
          }
        }
        """

        variables = {
            "input": {
                "name": "test",
                "email": "phase1@example.com",
                "password": "password123",
            }
        }

        first = await client.post(
            "/graphql",
            json={
                "query": mutation,
                "variables": variables,
            },
        )

        assert first.status_code == 200
        assert first.json()["data"]["register"][
            "accessToken"
        ]

        second = await client.post(
            "/graphql",
            json={
                "query": mutation,
                "variables": variables,
            },
        )

        body = second.json()

        assert second.status_code == 200
        assert body["data"] is None
        assert body["errors"][0]["message"] == (
            "Email already registered"
        )


@pytest.mark.asyncio
async def test_login_and_authenticated_issue_flow() -> None:
    transport = ASGITransport(
        app=app,
    )

    async with AsyncClient(
        transport=transport,
        base_url="http://test",
    ) as client:
        register = await client.post(
            "/graphql",
            json={
                "query": """
                mutation {
                  register(
                    input: {
                      name: "test"
                      email: "flow@example.com"
                      password: "password123"
                    }
                  ) {
                    accessToken
                  }
                }
                """,
            },
        )

        assert register.status_code == 200

        login = await client.post(
            "/graphql",
            json={
                "query": """
                mutation {
                  login(
                    input: {
                      email: "flow@example.com"
                      password: "password123"
                    }
                  ) {
                    accessToken
                  }
                }
                """,
            },
        )

        token = login.json()[
            "data"
        ]["login"]["accessToken"]

        create = await client.post(
            "/graphql",
            headers={
                "Authorization": f"Bearer {token}",
            },
            json={
                "query": """
                mutation {
                  createIssue(
                    input: {
                      title: "Integration Test"
                      description: "GraphQL HTTP"
                    }
                  ) {
                    id
                    title
                    description
                  }
                }
                """,
            },
        )

        assert create.json()[
            "data"
        ]["createIssue"]["description"] == (
            "GraphQL HTTP"
        )

        issues = await client.post(
            "/graphql",
            headers={
                "Authorization": f"Bearer {token}",
            },
            json={
                "query": """
                query {
                  issues {
                    id
                    title
                    description
                  }
                }
                """,
            },
        )

        assert len(
            issues.json()["data"]["issues"]
        ) == 1

        unauthenticated = await client.post(
            "/graphql",
            json={
                "query": """
                query {
                  issues {
                    id
                  }
                }
                """,
            },
        )

        assert unauthenticated.json()[
            "errors"
        ][0]["message"] == (
            "Authentication required"
        )
```

実行:

```bash
cd backend
DATABASE_URL="postgresql+asyncpg://app:password@localhost:5432/app_test" \
  uv run pytest tests/test_graphql_auth.py -q
```

期待結果:

```text
2 passed
```

ここでは最低限、次を実際にTestしている。

```text
1. 正常な会員登録
2. 重複emailを拒否
3. 登録したEmail / Passwordで正常Login
4. JWT付きでIssueを作成
5. JWT付きで自分のIssueを取得
6. Tokenなしのissues Queryを拒否
```

Login失敗、他UserのIssue更新拒否などはPhase 5でSecurity Testを増やす。

---

# 9段階 — Next.js + Tailwindで実際の画面を作る

## Goal

GraphiQLで確認してきたLogin / JWT / Issue CRUDを、今度はBrowser上の実画面から操作する。

この段階ではFrontend側の仕組みを見えやすくするため、Apollo Client、urql、TanStack Query、Zustandなどはまだ追加しない。

```text
Next.js App Router
TypeScript
Tailwind CSS
graphql-request
GraphQL Code Generator
localStorage
```

だけで完成させる。

> Phase 1ではJWTの仕組みを確認しやすくするため`localStorage`を使用する。HttpOnly Cookie / Refresh Token / CSRFはPhase 5で扱う。

---

## 9.1 — Next.js Project作成

Project rootで実行する。

```bash
npx create-next-app@latest frontend \
  --typescript \
  --eslint \
  --tailwind \
  --app \
  --src-dir
```

起動:

```bash
cd frontend
npm run dev
```

Browser:

```text
http://localhost:3000
```

Frontendの構成:

```text
frontend/
├── .env.local
├── src/
│   └── app/
│       ├── globals.css
│       ├── layout.tsx
│       ├── page.tsx
│       ├── register/
│       │   └── page.tsx
│       ├── login/
│       │   └── page.tsx
│       └── issues/
│           └── page.tsx
└── package.json
```

---

## 9.2 — Backend CORS

Frontendは`http://localhost:3000`、Backendは`http://localhost:8000`なので、開発環境ではCORSを許可する。

`src/app/main.py`:

```python
from fastapi.middleware.cors import (
    CORSMiddleware,
)
```

`app = FastAPI(...)`の後に追加する。

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

Productionでは実際のFrontend Originだけを許可する。

---

## 9.3 — GraphQL URL

`frontend/.env.local`:

```env
NEXT_PUBLIC_GRAPHQL_URL=http://localhost:8000/graphql
```

Environment Variable追加後はDev Serverを再起動する。

```bash
npm run dev
```

---

## 9.4 — graphql-request + `.graphql` + GraphQL Code Generator

FrontendではGraphQL OperationをComponent内のStringとして管理しない。

```text
.graphql File
→ GraphQL Code Generator
→ TypedDocumentNode
→ graphql-request
```

という構成にする。

### Install

```bash
npm install \
  graphql \
  graphql-request

npm install -D \
  @graphql-codegen/cli \
  @graphql-codegen/typescript \
  @graphql-codegen/typescript-operations \
  @graphql-codegen/typed-document-node
```

### Structure

```text
frontend/
├── codegen.ts
└── src/
    ├── app/
    ├── graphql/
    │   ├── register.graphql
    │   ├── login.graphql
    │   ├── issues.graphql
    │   ├── create-issue.graphql
    │   ├── update-issue.graphql
    │   └── delete-issue.graphql
    ├── generated/
    │   └── graphql.ts
    └── lib/
        └── graphql-client.ts
```

### GraphQL Documents

`src/graphql/register.graphql`:

```graphql
mutation Register($input: RegisterInput!) {
  register(input: $input) {
    accessToken

    user {
      id
      name
      email
    }
  }
}
```

`src/graphql/login.graphql`:

```graphql
mutation Login($input: LoginInput!) {
  login(input: $input) {
    accessToken

    user {
      id
      name
      email
    }
  }
}
```

`src/graphql/issues.graphql`:

```graphql
query GetIssues {
  issues {
    id
    title
    description
    status
    createdAt

    owner {
      id
      name
      email
    }
  }
}
```

`src/graphql/create-issue.graphql`:

```graphql
mutation CreateIssue($input: CreateIssueInput!) {
  createIssue(input: $input) {
    id
    title
    description
    status
    createdAt

    owner {
      id
      name
      email
    }
  }
}
```

`src/graphql/update-issue.graphql`:

```graphql
mutation UpdateIssue($id: Int!, $input: UpdateIssueInput!) {
  updateIssue(id: $id, input: $input) {
    id
    title
    description
    status
    createdAt

    owner {
      id
      name
      email
    }
  }
}
```

`GetIssues`と`CreateIssue`は、`Issue` stateへ同じObjectを入れるため、
`id / title / description / status / createdAt / owner`を同じField setとして揃える。
`UpdateIssue`も同じField setを返すようにし、Operation間でIssue shapeをずらさない。

`src/graphql/delete-issue.graphql`:

```graphql
mutation DeleteIssue($id: Int!) {
  deleteIssue(id: $id)
}
```

### Codegen Config

`frontend/codegen.ts`:

```typescript
import type { CodegenConfig } from "@graphql-codegen/cli";

const config: CodegenConfig = {
  schema: "http://127.0.0.1:8000/graphql",
  documents: "src/graphql/**/*.graphql",
  generates: {
    "src/generated/graphql.ts": {
      plugins: ["typescript-operations", "typed-document-node"],
      config: {
        scalars: {
          DateTime: {
            input: "string",
            output: "string",
          },
        },
      },
    },
  },
};

export default config;
```

`package.json`:

```json
{
  "scripts": {
    "codegen": "graphql-codegen --config codegen.ts"
  }
}
```

Backendを起動した状態で実行する。

```bash
npm run codegen
```

生成例:

```text
RegisterDocument
RegisterMutation
RegisterMutationVariables
LoginDocument
LoginMutation
LoginMutationVariables
GetIssuesDocument
CreateIssueDocument
UpdateIssueDocument
DeleteIssueDocument
```

### 生成されたType / Documentを実際に使う

CodegenはTypeを生成するだけで終わりではない。

このPhaseでは、生成されたTypeと`TypedDocumentNode`を実際のFrontendコードで使う。

生成された代表的なもの:

```text
LoginDocument
LoginMutation
LoginMutationVariables

GetIssuesDocument
GetIssuesQuery

CreateIssueDocument
CreateIssueMutation
CreateIssueMutationVariables
```

---

#### `LoginDocument`でResponse / Variablesの型を自動推論する

`LoginDocument`は単なる文字列ではなく、Codegenが生成した`TypedDocumentNode`。

そのため`graphql-request`に渡すと、ResponseとVariablesの型を自動的に推論できる。

```typescript
import { LoginDocument } from "@/generated/graphql";

import { getGraphQLClient } from "@/lib/graphql-client";

const data = await getGraphQLClient().request(LoginDocument, {
  input: {
    email,
    password,
  },
});

if (!data.login) {
  throw new Error("Login failed");
}

localStorage.setItem("accessToken", data.login.accessToken);
```

ここでは`data`に手動で型を書く必要がない。
`login`はBackend Schema上nullableなので、先にnull checkを行う。
その後は次のFieldがCodegenによって型付けされる。

```typescript
data.login.accessToken;
data.login.user.id;
data.login.user.name;
data.login.user.email;
```

存在しないFieldを書いた場合:

```typescript
data.login.user.username;
```

Schema / Operationに`username`が存在しなければTypeScript Errorになる。

---

#### Variablesの型もCodegenから取得できる

必要であれば生成されたVariables Typeを明示的に利用できる。

```typescript
import type { LoginMutationVariables } from "@/generated/graphql";

const variables: LoginMutationVariables = {
  input: {
    email,
    password,
  },
};

const data = await getGraphQLClient().request(LoginDocument, variables);
```

ただし通常は`LoginDocument`からVariables Typeが推論されるため、毎回明示的に書く必要はない。

---

#### Query Response TypeをUI Stateに利用する

Issue型をFrontend側で手書きしない。

```typescript
import type { GetIssuesQuery } from "@/generated/graphql";

type Issue = GetIssuesQuery["issues"][number];
```

これにより:

```typescript
const [issues, setIssues] = useState<Issue[]>([]);
```

の`Issue`型がGraphQL Operationと同期する。

Backend Schemaまたは`.graphql` Operationを変更した場合:

```bash
npm run codegen
```

を再実行する。

するとGenerated Typeが更新され、Frontend側で影響箇所をTypeScript Errorとして確認できる。

---

#### このPhaseで確認すること

```text
1. .graphql Fileを変更
2. npm run codegen
3. src/generated/graphql.tsが更新される
4. Generated Documentをgraphql-requestへ渡す
5. Response / Variablesが自動で型付けされる
6. Generated Query TypeをUI Stateにも利用する
```

つまりこのPhaseの目的は:

```text
Backend Schema
↓
.graphql Operation
↓
Codegen
↓
TypedDocumentNode + TypeScript Type
↓
graphql-request
↓
React UI
```

までを一つの流れとして理解すること。

### GraphQL Client

`src/lib/graphql-client.ts`:

```typescript
import { GraphQLClient } from "graphql-request";

const endpoint = process.env.NEXT_PUBLIC_GRAPHQL_URL;

export const getGraphQLClient = () => {
  if (!endpoint) {
    throw new Error("NEXT_PUBLIC_GRAPHQL_URL is not defined");
  }

  const token =
    typeof window !== "undefined" ? localStorage.getItem("accessToken") : null;

  return new GraphQLClient(endpoint, {
    headers: token
      ? {
          Authorization: `Bearer ${token}`,
        }
      : {},
  });
};
```

この段階ではServer State Cacheはまだ追加しない。
Phase 2でApollo Clientを導入し、GraphQL Cacheを本格的に学ぶ。

---

## 9.5 — Root Page

`frontend/src/app/page.tsx`:

```tsx
import Link from "next/link";

const Home = () => {
  return (
    <main
      className="
        mx-auto
        flex
        min-h-screen
        max-w-2xl
        flex-col
        justify-center
        gap-6
        p-6
      "
    >
      <div>
        <h1
          className="
            text-3xl
            font-bold
          "
        >
          Issue Tracker
        </h1>

        <p
          className="
            mt-2
            text-sm
            text-gray-600
          "
        >
          FastAPI + GraphQL + Next.js
        </p>
      </div>

      <div
        className="
          flex
          gap-3
        "
      >
        <Link
          href="/login"
          className="
            rounded
            bg-black
            px-4
            py-2
            text-white
          "
        >
          Login
        </Link>

        <Link
          href="/register"
          className="
            rounded
            border
            border-gray-300
            px-4
            py-2
          "
        >
          Register
        </Link>

        <Link
          href="/issues"
          className="
            rounded
            border
            border-gray-300
            px-4
            py-2
          "
        >
          Issues
        </Link>
      </div>
    </main>
  );
};

export default Home;
```

`/login`、`/register`、`/issues`へ移動できることを確認する。

---

## 9.6 — Registration Page

この画面で次のFlowを確認する。

```text
Name / Email / Password入力
→ register Mutation
→ AuthPayload受信
→ accessToken保存
→ /issuesへ移動
```

`frontend/src/app/register/page.tsx`:

```tsx
"use client";

import type { FormEvent } from "react";
import { useState } from "react";

import { useRouter } from "next/navigation";

import Link from "next/link";

import { RegisterDocument } from "@/generated/graphql";

import { getGraphQLClient } from "@/lib/graphql-client";

const RegisterPage = () => {
  const router = useRouter();

  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (event: FormEvent<HTMLFormElement>) => {
    event.preventDefault();

    setError("");
    setLoading(true);

    try {
      const client = getGraphQLClient();

      const data = await client.request(RegisterDocument, {
        input: {
          name,
          email,
          password,
        },
      });

      localStorage.setItem("accessToken", data.register.accessToken);

      router.push("/issues");
    } catch (error) {
      setError(error instanceof Error ? error.message : "Unknown error");
    } finally {
      setLoading(false);
    }
  };

  return (
    <main className="mx-auto flex min-h-screen max-w-md items-center p-6">
      <form
        onSubmit={handleSubmit}
        className="w-full space-y-4 rounded-lg border border-gray-200 p-6"
      >
        <h1 className="text-2xl font-bold">Register</h1>

        <div>
          <label htmlFor="name" className="mb-1 block text-sm font-medium">
            Name
          </label>
          <input
            id="name"
            value={name}
            onChange={(event) => setName(event.target.value)}
            className="w-full rounded border border-gray-300 px-3 py-2"
            required
          />
        </div>

        <div>
          <label htmlFor="email" className="mb-1 block text-sm font-medium">
            Email
          </label>
          <input
            id="email"
            type="email"
            value={email}
            onChange={(event) => setEmail(event.target.value)}
            className="w-full rounded border border-gray-300 px-3 py-2"
            required
          />
        </div>

        <div>
          <label htmlFor="password" className="mb-1 block text-sm font-medium">
            Password
          </label>
          <input
            id="password"
            type="password"
            value={password}
            onChange={(event) => setPassword(event.target.value)}
            className="w-full rounded border border-gray-300 px-3 py-2"
            required
          />
        </div>

        {error && <p className="text-sm text-red-600">{error}</p>}

        <button
          type="submit"
          disabled={loading}
          className="w-full rounded bg-black px-4 py-2 text-white disabled:opacity-50"
        >
          {loading ? "Registering..." : "Register"}
        </button>

        <p className="text-center text-sm text-gray-600">
          Already have an account?{" "}
          <Link href="/login" className="underline">
            Login
          </Link>
        </p>
      </form>
    </main>
  );
};

export default RegisterPage;
```

確認する。

```text
1. /registerからUserを作成できる
2. register ResponseのaccessTokenがlocalStorageへ保存される
3. 登録後/issuesへ移動する
4. Logout後、登録したEmail / PasswordでLoginできる
5. 同じEmailでは登録できない
6. LoginへのLinkから/loginへ移動できる
```

---

## 9.7 — Login Page

この画面で次のFlowを確認する。

```text
Email / Password入力
→ login Mutation
→ accessToken取得
→ localStorage保存
→ /issuesへ移動
```

`frontend/src/app/login/page.tsx`:

```tsx
"use client";

import type { FormEvent } from "react";
import { useState } from "react";

import { useRouter } from "next/navigation";

import Link from "next/link";

import { LoginDocument } from "@/generated/graphql";

import { getGraphQLClient } from "@/lib/graphql-client";

const LoginPage = () => {
  const router = useRouter();

  const [email, setEmail] = useState("");

  const [password, setPassword] = useState("");

  const [error, setError] = useState("");

  const [loading, setLoading] = useState(false);

  const handleSubmit = async (event: FormEvent<HTMLFormElement>) => {
    event.preventDefault();

    setError("");
    setLoading(true);

    try {
      const client = getGraphQLClient();

      const data = await client.request(LoginDocument, {
        input: {
          email,
          password,
        },
      });

      if (!data.login) {
        throw new Error("Login failed");
      }

      localStorage.setItem("accessToken", data.login.accessToken);

      router.push("/issues");
    } catch (error) {
      setError(error instanceof Error ? error.message : "Unknown error");
    } finally {
      setLoading(false);
    }
  };

  return (
    <main
      className="
        mx-auto
        flex
        min-h-screen
        max-w-md
        items-center
        p-6
      "
    >
      <form
        onSubmit={handleSubmit}
        className="
          w-full
          space-y-4
          rounded-lg
          border
          border-gray-200
          p-6
        "
      >
        <div>
          <h1
            className="
              text-2xl
              font-bold
            "
          >
            Login
          </h1>

          <p
            className="
              mt-1
              text-sm
              text-gray-500
            "
          >
            GraphQL JWT login
          </p>
        </div>

        <div>
          <label
            className="
              mb-1
              block
              text-sm
              font-medium
            "
          >
            Email
          </label>

          <input
            type="email"
            value={email}
            onChange={(event) => setEmail(event.target.value)}
            required
            className="
              w-full
              rounded
              border
              border-gray-300
              px-3
              py-2
            "
          />
        </div>

        <div>
          <label
            className="
              mb-1
              block
              text-sm
              font-medium
            "
          >
            Password
          </label>

          <input
            type="password"
            value={password}
            onChange={(event) => setPassword(event.target.value)}
            required
            className="
              w-full
              rounded
              border
              border-gray-300
              px-3
              py-2
            "
          />
        </div>

        {error && (
          <p
            className="
              rounded
              bg-red-50
              p-3
              text-sm
              text-red-700
            "
          >
            {error}
          </p>
        )}

        <button
          type="submit"
          disabled={loading}
          className="
            w-full
            rounded
            bg-black
            px-4
            py-2
            text-white
            disabled:opacity-50
          "
        >
          {loading ? "Logging in..." : "Login"}
        </button>

        <p className="text-center text-sm text-gray-600">
          Don&apos;t have an account?{" "}
          <Link href="/register" className="underline">
            Create account
          </Link>
        </p>
      </form>
    </main>
  );
};

export default LoginPage;
```

確認ポイント:

```text
1. Email / Passwordを入力できる
2. Login中はButtonがdisabledになる
3. Error時はMessageが表示される
4. Success時はaccessTokenが保存される
5. /issuesへ遷移する
6. Create accountから/registerへ移動できる
```

---

## 9.8 — Issue CRUD Page

1画面で以下を確認する。

```text
READ   → Issue List
CREATE → FormからIssue作成
UPDATE → Done Button
DELETE → Delete Button
LOGOUT → Token削除
```

`frontend/src/app/issues/page.tsx`:

```tsx
"use client";

import type { FormEvent } from "react";
import { useCallback, useEffect, useState } from "react";

import { useRouter } from "next/navigation";

import {
  CreateIssueDocument,
  DeleteIssueDocument,
  GetIssuesDocument,
  UpdateIssueDocument,
  type GetIssuesQuery,
} from "@/generated/graphql";

import { getGraphQLClient } from "@/lib/graphql-client";

type Issue = GetIssuesQuery["issues"][number];

const IssuesPage = () => {
  const router = useRouter();

  const [issues, setIssues] = useState<Issue[]>([]);

  const [title, setTitle] = useState("");

  const [description, setDescription] = useState("");

  const [loading, setLoading] = useState(true);

  const [saving, setSaving] = useState(false);

  const [error, setError] = useState("");

  const loadIssues = useCallback(async () => {
    setError("");

    try {
      const client = getGraphQLClient();

      const data = await client.request(GetIssuesDocument);

      setIssues(data.issues);
    } catch (error) {
      setError(error instanceof Error ? error.message : "Unknown error");
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => {
    const token = localStorage.getItem("accessToken");

    if (!token) {
      router.replace("/login");

      return;
    }

    void loadIssues();
  }, [loadIssues, router]);

  const handleCreate = async (event: FormEvent<HTMLFormElement>) => {
    event.preventDefault();

    if (!title.trim()) {
      return;
    }

    setSaving(true);
    setError("");

    try {
      const data = await getGraphQLClient().request(CreateIssueDocument, {
        input: {
          title,
          description: description || null,
        },
      });

      setIssues((current) => [...current, data.createIssue]);

      setTitle("");
      setDescription("");
    } catch (error) {
      setError(error instanceof Error ? error.message : "Unknown error");
    } finally {
      setSaving(false);
    }
  };

  const markDone = async (issueId: number) => {
    setError("");

    try {
      const data = await getGraphQLClient().request(UpdateIssueDocument, {
        id: issueId,

        input: {
          status: "DONE",
        },
      });

      const updatedIssue = data.updateIssue;

      if (!updatedIssue) {
        return;
      }

      setIssues((current) =>
        current.map((issue) =>
          issue.id === issueId
            ? {
                ...issue,
                status: updatedIssue.status,
              }
            : issue,
        ),
      );
    } catch (error) {
      setError(error instanceof Error ? error.message : "Unknown error");
    }
  };

  const removeIssue = async (issueId: number) => {
    setError("");

    try {
      const data = await getGraphQLClient().request(DeleteIssueDocument, {
        id: issueId,
      });

      if (!data.deleteIssue) {
        return;
      }

      setIssues((current) => current.filter((issue) => issue.id !== issueId));
    } catch (error) {
      setError(error instanceof Error ? error.message : "Unknown error");
    }
  };

  const logout = () => {
    localStorage.removeItem("accessToken");

    router.replace("/login");
  };

  return (
    <main
      className="
        mx-auto
        min-h-screen
        max-w-3xl
        space-y-8
        p-6
      "
    >
      <header
        className="
          flex
          items-center
          justify-between
        "
      >
        <div>
          <h1
            className="
              text-3xl
              font-bold
            "
          >
            Issues
          </h1>

          <p
            className="
              text-sm
              text-gray-500
            "
          >
            GraphQL CRUD
          </p>
        </div>

        <button
          type="button"
          onClick={logout}
          className="
            rounded
            border
            border-gray-300
            px-3
            py-2
            text-sm
          "
        >
          Logout
        </button>
      </header>

      <form
        onSubmit={handleCreate}
        className="
          space-y-3
          rounded-lg
          border
          border-gray-200
          p-4
        "
      >
        <h2
          className="
            text-lg
            font-semibold
          "
        >
          Create Issue
        </h2>

        <input
          value={title}
          onChange={(event) => setTitle(event.target.value)}
          placeholder="Title"
          className="
            w-full
            rounded
            border
            border-gray-300
            px-3
            py-2
          "
        />

        <textarea
          value={description}
          onChange={(event) => setDescription(event.target.value)}
          placeholder="Description"
          className="
            min-h-24
            w-full
            rounded
            border
            border-gray-300
            px-3
            py-2
          "
        />

        <button
          type="submit"
          disabled={saving}
          className="
            rounded
            bg-black
            px-4
            py-2
            text-white
            disabled:opacity-50
          "
        >
          {saving ? "Creating..." : "Create"}
        </button>
      </form>

      {error && (
        <p
          className="
            rounded
            bg-red-50
            p-3
            text-sm
            text-red-700
          "
        >
          {error}
        </p>
      )}

      {loading ? (
        <p>Loading...</p>
      ) : (
        <section
          className="
            space-y-3
          "
        >
          {issues.length === 0 && (
            <p
              className="
                text-gray-500
              "
            >
              No issues.
            </p>
          )}

          {issues.map((issue) => (
            <article
              key={issue.id}
              className="
                  rounded-lg
                  border
                  border-gray-200
                  p-4
                "
            >
              <div
                className="
                    flex
                    items-start
                    justify-between
                    gap-4
                  "
              >
                <div>
                  <div
                    className="
                        flex
                        items-center
                        gap-2
                      "
                  >
                    <h2
                      className="
                          font-semibold
                        "
                    >
                      {issue.title}
                    </h2>

                    <span
                      className="
                          rounded
                          bg-gray-100
                          px-2
                          py-1
                          text-xs
                        "
                    >
                      {issue.status}
                    </span>
                  </div>

                  {issue.description && (
                    <p
                      className="
                          mt-2
                          text-sm
                          text-gray-600
                        "
                    >
                      {issue.description}
                    </p>
                  )}

                  <p
                    className="
                        mt-2
                        text-xs
                        text-gray-400
                      "
                  >
                    Owner: {issue.owner.name}
                  </p>

                  <p
                    className="
                        mt-1
                        text-xs
                        text-gray-400
                      "
                  >
                    Created: {new Date(issue.createdAt).toLocaleString()}
                  </p>
                </div>

                <div
                  className="
                      flex
                      shrink-0
                      gap-2
                    "
                >
                  {issue.status !== "DONE" && (
                    <button
                      type="button"
                      onClick={() => void markDone(issue.id)}
                      className="
                          rounded
                          border
                          border-gray-300
                          px-3
                          py-1
                          text-sm
                        "
                    >
                      Done
                    </button>
                  )}

                  <button
                    type="button"
                    onClick={() => void removeIssue(issue.id)}
                    className="
                        rounded
                        bg-red-600
                        px-3
                        py-1
                        text-sm
                        text-white
                      "
                  >
                    Delete
                  </button>
                </div>
              </div>
            </article>
          ))}
        </section>
      )}
    </main>
  );
};

export default IssuesPage;
```

このStepではUI Libraryを追加せず、Tailwindだけで最低限の画面を作る。

---

## 9.9 — BrowserでCRUD確認

Backend:

```bash
cd backend
uv run uvicorn app.main:app --reload
```

Frontend:

```bash
cd frontend
npm run dev
```

Browser:

```text
http://localhost:3000/login
```

### Login

Login後、Chrome DevToolsで確認する。

```text
Application
→ Local Storage
→ http://localhost:3000
→ accessToken
```

### Read

`/issues`を開き、DBに保存されているIssueが画面に表示されることを確認する。

### Create

```text
Title: Frontend CRUD
Description: Next.jsから作成
```

Create後、画面に追加されることを確認する。

DBでも確認する。

```sql
SELECT
    id,
    title,
    description,
    status,
    created_at,
    owner_id
FROM issues
ORDER BY id;
```

### Update

`Done`を押してStatusが以下のように変わることを確認する。

```text
OPEN
→ DONE
```

### Delete

`Delete`後、Cardが画面から消え、DBからも削除されていることを確認する。

### Logout

```text
localStorageからaccessToken削除
→ /loginへ移動
```

---

## 9.10 — Network TabでGraphQL確認

Chrome DevTools:

```text
Network
→ graphql
```

確認する内容:

```text
Request Headers
Authorization: Bearer <JWT>

Request Payload
query
variables

Response
data / errors
```

GraphiQLで実行していたQuery / MutationがBrowserからどのように送信されるかを確認する。

---

## 9.11 — Authentication Failure確認

DevToolsから`accessToken`を削除して`/issues`へ移動する。

```text
Tokenなし
→ /loginへRedirect
```

次に無効なTokenを保存してRequestする。

```text
accessToken = abc
```

```text
無効Token
→ Backend JWT検証失敗
→ GraphQL Error
→ Frontend Error表示
```

ここでFrontend側のRoute ControlとBackend側のAuthenticationは別の責務であることを確認する。

---

# 9段階拡張 — Frontend Testing

Frontendでも「Tool名だけ知る」で終わらせず、最低1本ずつ実行する。

## Vitest + React Testing Library

Install:

```bash
cd frontend
npm install -D vitest jsdom @testing-library/react @testing-library/jest-dom
```

`package.json`へ追加:

```json
{
  "scripts": {
    "test": "vitest run"
  }
}
```

`frontend/vitest.config.ts`:

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "jsdom",
    setupFiles: ["./vitest.setup.ts"],
  },
});
```

`frontend/vitest.setup.ts`:

```ts
import "@testing-library/jest-dom/vitest";
```

最初のTestは外部APIを必要としないRoot Pageにする。

`frontend/src/app/page.test.tsx`:

```tsx
import { render, screen } from "@testing-library/react";
import { describe, expect, it } from "vitest";

import Home from "./page";

describe("Home", () => {
  it("shows navigation links", () => {
    render(<Home />);

    expect(screen.getByRole("link", { name: "Login" })).toHaveAttribute(
      "href",
      "/login",
    );

    expect(screen.getByRole("link", { name: "Register" })).toHaveAttribute(
      "href",
      "/register",
    );
  });
});
```

実行:

```bash
cd frontend
npm run test
```

## Playwright

Install:

```bash
cd frontend
npm install -D @playwright/test
npx playwright install chromium
```

`frontend/playwright.config.ts`:

```ts
import { defineConfig } from "@playwright/test";

export default defineConfig({
  use: {
    baseURL: "http://localhost:3000",
  },
});
```

`frontend/e2e/navigation.spec.ts`:

```ts
import { expect, test } from "@playwright/test";

test("register pageへ移動できる", async ({ page }) => {
  await page.goto("/");

  await page.getByRole("link", { name: "Register" }).click();

  await expect(page).toHaveURL(/\/register$/);
});
```

Frontendを起動した状態で実行:

```bash
cd frontend
npm run dev
```

別Terminal:

```bash
cd frontend
npx playwright test
```

Phase 1ではE2Eを大量に書かない。Login / Register / Issue CRUDのうち、本当に重要なUser FlowだけをPlaywrightで確認する。

---

# 10段階 — Filter / Search / Cursor Pagination

9段階で作成したIssue一覧を、Server側Filter / Search / Cursor Paginationへ発展させる。
この段階では次のFileを順番に変更する。

```text
1. backend/src/app/graphql/schemas/issue.py
2. backend/src/app/services/issues.py
3. backend/src/app/graphql/query.py
4. frontend/src/graphql/issues.graphql
5. frontend/src/generated/graphql.ts  ← npm run codegenで自動生成
6. frontend/src/app/issues/page.tsx
```

## 10.1 — `graphql/schemas/issue.py`

既存の`Issue` Typeはそのまま残し、一覧Response用Typeを追加する。

```python
@strawberry.type
class IssueConnection:
    items: list[Issue]
    next_cursor: str | None
```

Strawberryでは`next_cursor`がGraphQL上で`nextCursor`になる。

## 10.2 — `services/issues.py`

8段階の`services/issues.py`全体を作り直さない。
既存のCreate / Update / Deleteと`to_issue()`はそのまま残し、importへ`base64`と`IssueConnection`を追加して、`get_issues()`だけを置き換える。

既存importへの追加:

```python
import base64

from app.graphql.schemas.issue import (
    CreateIssueInput,
    Issue,
    IssueConnection,
    UpdateIssueInput,
)
```

`get_issues()`とCursor helper:

```python


def encode_cursor(
    issue_id: int,
) -> str:
    raw = f"issue:{issue_id}"

    return base64.urlsafe_b64encode(
        raw.encode(),
    ).decode()


def decode_cursor(
    cursor: str,
) -> int:
    raw = base64.urlsafe_b64decode(
        cursor.encode(),
    ).decode()

    prefix, issue_id = raw.split(
        ":",
        maxsplit=1,
    )

    if prefix != "issue":
        raise ValueError(
            "Invalid cursor"
        )

    return int(issue_id)


async def get_issues(
    current_user_id: int,
    status: str | None = None,
    search: str | None = None,
    after: str | None = None,
    first: int = 20,
) -> IssueConnection:
    if first < 1 or first > 100:
        raise ValueError(
            "first must be between 1 and 100"
        )

    async with SessionLocal() as session:
        stmt = (
            select(IssueModel)
            .where(
                IssueModel.owner_id
                == current_user_id
            )
            .order_by(IssueModel.id)
        )

        if status is not None:
            stmt = stmt.where(
                IssueModel.status == status
            )

        if search:
            stmt = stmt.where(
                IssueModel.title.ilike(
                    f"%{search}%"
                )
            )

        if after is not None:
            stmt = stmt.where(
                IssueModel.id
                > decode_cursor(after)
            )

        stmt = stmt.limit(first + 1)

        rows = list(
            (await session.scalars(stmt)).all()
        )

        has_next = len(rows) > first
        models = rows[:first]

        items = [
            to_issue(model)
            for model in models
        ]

        next_cursor = (
            encode_cursor(models[-1].id)
            if has_next and models
            else None
        )

        return IssueConnection(
            items=items,
            next_cursor=next_cursor,
        )
```

## 10.3 — `graphql/query.py`

`IssueConnection`をimportし、`issues` Resolverだけを置き換える。
`issue(id)` Resolverは8段階のままでよい。

```python
from app.graphql.schemas.issue import (
    Issue,
    IssueConnection,
)
```

```python
@strawberry.field
async def issues(
    self,
    info: strawberry.Info,
    status: str | None = None,
    search: str | None = None,
    after: str | None = None,
    first: int = 20,
) -> IssueConnection:
    current_user = (
        info.context.current_user
    )

    if current_user is None:
        raise ValueError(
            "Authentication required"
        )

    return await issue_service.get_issues(
        current_user_id=current_user.id,
        status=status,
        search=search,
        after=after,
        first=first,
    )
```

## 10.4 — GraphiQL確認

```graphql
query {
  issues(status: "OPEN", search: "GraphQL", first: 20) {
    items {
      id
      title
      description
      status
      createdAt

      owner {
        id
        name
        email
      }
    }
    nextCursor
  }
}
```

`nextCursor`が返った場合、次のRequestで`after`へ渡す。

```graphql
query {
  issues(after: "取得したcursor", first: 20) {
    items {
      id
      title
      description
      status
      createdAt

      owner {
        id
        name
        email
      }
    }
    nextCursor
  }
}
```

## 10.5 — `frontend/src/graphql/issues.graphql`

9段階の`GetIssues`を以下へ**置き換える**。
9段階で使用している`description` / `createdAt` / `owner`を落とさず、IssueのField setを維持する。

```graphql
query GetIssues($status: String, $search: String, $after: String, $first: Int) {
  issues(status: $status, search: $search, after: $after, first: $first) {
    items {
      id
      title
      description
      status
      createdAt

      owner {
        id
        name
        email
      }
    }

    nextCursor
  }
}
```

Backendを起動した状態でCodegenを再実行する。

```bash
cd frontend
npm run codegen
```

`frontend/src/generated/graphql.ts`は直接編集しない。

## 10.6 — `frontend/src/app/issues/page.tsx`

`GetIssues`のResponse shapeが`Issue[]`から`IssueConnection`へ変わったため、まずIssue型を変更する。

```tsx
type Issue = GetIssuesQuery["issues"]["items"][number];
```

既存stateへFilter / Search / Cursorを追加する。

```tsx
const [status, setStatus] = useState("");

const [search, setSearch] = useState("");

const [debouncedSearch, setDebouncedSearch] = useState("");

const [nextCursor, setNextCursor] = useState<string | null>(null);
```

Searchは入力ごとにRequestせず、300ms待ってからQueryへ反映する。

```tsx
useEffect(() => {
  const timer = window.setTimeout(() => {
    setDebouncedSearch(search);
  }, 300);

  return () => {
    window.clearTimeout(timer);
  };
}, [search]);
```

9段階の`loadIssues`を以下へ置き換える。

```tsx
const loadIssues = useCallback(
  async (after: string | null = null, append = false) => {
    try {
      const data = await getGraphQLClient().request(GetIssuesDocument, {
        status: status || null,
        search: debouncedSearch || null,
        after,
        first: 20,
      });

      setIssues((current) =>
        append ? [...current, ...data.issues.items] : data.issues.items,
      );

      setNextCursor(data.issues.nextCursor ?? null);

      setError("");
    } catch (error) {
      setError(error instanceof Error ? error.message : "Unknown error");
    } finally {
      setLoading(false);
    }
  },
  [status, debouncedSearch],
);
```

初回表示とFilter / Search変更時は先頭Pageから取得する。
既存のtoken checkを残したまま、Effectを次の形にする。

```tsx
useEffect(() => {
  const token = localStorage.getItem("accessToken");

  if (!token) {
    router.replace("/login");

    return;
  }

  void loadIssues(null, false);
}, [loadIssues, router]);
```

Load More handlerを追加する。

```tsx
const handleLoadMore = async () => {
  if (!nextCursor) {
    return;
  }

  await loadIssues(nextCursor, true);
};
```

Issue Listの前にFilter / Search UIを追加する。

```tsx
<div className="flex gap-3">
  <select
    value={status}
    onChange={(event) => setStatus(event.target.value)}
    className="rounded border border-gray-300 px-3 py-2"
  >
    <option value="">All</option>
    <option value="OPEN">OPEN</option>
    <option value="DONE">DONE</option>
  </select>

  <input
    value={search}
    onChange={(event) => setSearch(event.target.value)}
    placeholder="Search title"
    className="flex-1 rounded border border-gray-300 px-3 py-2"
  />
</div>
```

Issue Listの下にLoad Moreを追加する。

```tsx
{
  nextCursor && (
    <button
      type="button"
      onClick={() => void handleLoadMore()}
      className="rounded border border-gray-300 px-4 py-2"
    >
      Load More
    </button>
  );
}
```

この段階ではCreate / Update後に現在のFilter条件と表示内容がずれないよう、成功後に先頭Pageを再取得してもよい。
Phase 2ではApollo Clientのcache / pagination policyへ移行する。

確認項目:

```text
1. GetIssuesにid / title / description / status / createdAt / ownerが揃っている
2. npm run codegenが成功する
3. statusがGraphQL variablesとして渡される
4. searchが300ms debounce後にDB queryへ反映される
5. nextCursorがnullになるまで次Pageを取得できる
6. Load Moreで既存Issueへ次Pageが追加される
7. Filter / Search変更時は先頭Pageから再取得される
8. 他UserのIssueは一覧へ出ない
```

---

## 10.7 — Pagination変更後のIntegration Test更新

10段階では`issues`の戻り値が`list[Issue]`から`IssueConnection`へ変わる。
8段階拡張で作成した`tests/test_graphql_auth.py`をそのまま残すと、Step 11のCIで古いQuery shapeを実行して失敗する。

`test_login_and_authenticated_issue_flow()`内の`issues` Queryを次へ置き換える。

```python
issues = await client.post(
    "/graphql",
    headers={
        "Authorization": f"Bearer {token}",
    },
    json={
        "query": """
        query {
          issues(first: 20) {
            items {
              id
              title
              description
              status
              createdAt
              owner {
                id
                email
              }
            }
            nextCursor
          }
        }
        """,
    },
)

body = issues.json()["data"]["issues"]

assert len(body["items"]) == 1
assert body["items"][0]["title"] == (
    "Integration Test"
)
assert body["nextCursor"] is None
```

Unauthenticated QueryもConnection shapeへ合わせる。

```python
unauthenticated = await client.post(
    "/graphql",
    json={
        "query": """
        query {
          issues(first: 20) {
            items {
              id
            }
          }
        }
        """,
    },
)

assert unauthenticated.json()[
    "errors"
][0]["message"] == (
    "Authentication required"
)
```

再実行する。

```bash
cd backend
DATABASE_URL="postgresql+asyncpg://app:password@localhost:5432/app_test" \
  uv run pytest tests/test_graphql_auth.py -q
```

これでStep 11のCIも10段階時点のSchemaと同じContractをTestする。

---

# 11段階 — Docker Compose + CI

この段階では「Dockerfileを書いた」で終わらせず、PostgreSQL + BackendをComposeで起動し、CIでもBackend / Frontend Testを実行する。

## 11.1 — `backend/Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN pip install uv

COPY pyproject.toml uv.lock ./
COPY src ./src

RUN uv sync --frozen

EXPOSE 8000

CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## 11.2 — root `docker-compose.yml`

5段階でPostgreSQLだけだったComposeをBackend込みに更新する。
Container間ではDB hostに`localhost`ではなくService名`postgres`を使う。

```yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: password
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 5s
      retries: 10

  backend:
    build:
      context: ./backend
    environment:
      DATABASE_URL: >-
        postgresql+asyncpg://app:password@postgres:5432/app
      JWT_SECRET: local-docker-secret
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres_data:
```

起動:

```bash
docker compose up --build
```

確認:

```bash
curl http://localhost:8000/health
```

GraphQL:

```text
http://localhost:8000/graphql
```

停止:

```bash
docker compose down
```

DBも初期化する場合だけ:

```bash
docker compose down -v
```

## 11.3 — `.github/workflows/ci.yml`

Backend Integration TestはPostgreSQLが必要なので、GitHub ActionsのService Containerを使う。

```yaml
name: CI

on:
  push:
  pull_request:

jobs:
  backend:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: password
          POSTGRES_DB: app_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U app -d app_test"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10

    env:
      DATABASE_URL: >-
        postgresql+asyncpg://app:password@localhost:5432/app_test

    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v6

      - run: |
          cd backend
          uv sync --frozen

      - run: |
          cd backend
          uv run python -m compileall src

      - run: |
          cd backend
          uv run pytest tests/test_graphql_auth.py -q

  frontend:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - run: |
          cd frontend
          npm ci

      - run: |
          cd frontend
          npm run test

      - run: |
          cd frontend
          npm run build
        env:
          NEXT_PUBLIC_GRAPHQL_URL: http://localhost:8000/graphql
```

この段階の確認結果は次になる。

```text
Local:
Docker Compose → PostgreSQL + FastAPI起動

CI:
Backend compile + GraphQL Integration Test
Frontend Vitest + Next.js build
```

Playwright E2EをCIへ追加するのはPhase 4でDeployment flowと一緒に扱う。

---

# 12段階 — AWS Deployment入門

Phase 1ではAWS全体を構築しない。
Localで動いているBackendを一度AWSへ移し、Network / Compute / Databaseの関係を体験することを目的にする。

Phase 4でECS / ALB / ECR / Terraform / Rollbackへ進むため、ここでは次だけを行う。

```text
Local Browser
  ↓
EC2
  └─ FastAPI Docker
       ↓
      RDS PostgreSQL
```

## Free Tier / Costの前提

2026年のAWS新規Accountでは、Free Planを最大6か月利用でき、Signup時の$100 Creditと、Activityによる追加最大$100 Creditが用意されている。
ただし、Account条件・Region・Resourceによって対象や消費量は変わるため、作成前後にBilling / Free Tier画面を必ず確認する。

「Free TierだからResourceを残したままでよい」とは考えない。
Hands-on終了後は不要Resourceを削除する。
また、Public IPv4 Addressは課金対象になり得るため、使用中のCredit / Billingを確認しながら進める。

## 12.1 — EC2作成

AWS Consoleで小さいLinux Instanceを1台作成する。

確認する項目:

```text
Region
Instance Type
Key Pair
Security Group
Public IPv4
```

Security Groupでは学習用に必要なPortだけを開け、Sourceは可能なら自分のPublic IPに限定する。

```text
22   SSH      Source: My IP
8000 FastAPI  Source: My IP
```

SSH接続:

```bash
ssh -i <key-file> \
  <user>@<ec2-public-ip>
```

## 12.2 — EC2へDockerをInstall

Amazon Linux系の場合の例:

```bash
sudo dnf update -y
sudo dnf install -y docker
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker "$USER"
```

一度SSHを切断して再接続した後に確認する。

```bash
docker --version
docker ps
```

## 12.3 — Backend ImageをEC2で起動

Phase 11で作成したBackend Dockerfileを使う。

ProjectをEC2へ配置した後:

```bash
docker build \
  -t issue-backend \
  ./backend
```

RDS接続前はContainer buildが成功することまで確認する。

## 12.4 — RDS PostgreSQL作成

RDSでPostgreSQLを作成する。

確認する項目:

```text
PostgreSQL
小さいInstance Class
Private accessを基本とする
DB名: app
User: app
Backup / Storage設定
```

RDSはEC2と同じVPC内に作成する。
RDS側Security Groupは、Internet全体ではなくEC2のSecurity GroupからPostgreSQL `5432`へ接続できるようにする。

```text
EC2 Security Group
      ↓ 5432
RDS Security Group
```

## 12.5 — EC2からRDS疎通確認

RDS Endpointを使ってBackendの`DATABASE_URL`を設定する。

```bash
export DATABASE_URL="postgresql+asyncpg://app:<password>@<rds-endpoint>:5432/app"
```

Container起動:

```bash
export JWT_SECRET="<long-random-secret>"

docker run \
  --rm \
  -p 8000:8000 \
  -e DATABASE_URL="$DATABASE_URL" \
  -e JWT_SECRET="$JWT_SECRET" \
  issue-backend
```

Local PCからまずHealth Checkを確認する。

```bash
curl http://<ec2-public-ip>:8000/health
```

GraphQLもHTTP POSTで確認する。

```bash
curl \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"query":"query { __typename }"}' \
  http://<ec2-public-ip>:8000/graphql
```

Frontendから確認する場合は`frontend/.env.local`を一時的にEC2へ向け、Dev Serverを再起動する。

```env
NEXT_PUBLIC_GRAPHQL_URL=http://<ec2-public-ip>:8000/graphql
```

Register / Login / Issue CRUDを実行し、DataがRDSへ保存されることを確認する。

## 12.6 — Hands-on終了後にResource削除

学習が終わったら必ず確認する。

```text
[ ] EC2 Instanceを終了
[ ] RDS Instanceを削除
[ ] 不要Snapshotを削除
[ ] 不要Security Group / Volumeを確認
[ ] Billing / Cost Managementを確認
```

Phase 1ではここまででよい。

Phase 4ではこの構成を次へ発展させる。

```text
EC2手動Deploy
  ↓
ECR
  ↓
ALB
  ↓
ECS / Fargate
  ↓
RDS / ElastiCache
  ↓
CI/CD
  ↓
Terraform
```

---

# Async Error Checklist

## 1. `AsyncSession` context manager

誤り:

```python
with SessionLocal() as session:
```

正しいコード:

```python
async with SessionLocal() as session:
```

## 2. commit / refreshのawait漏れ

誤り:

```python
session.commit()
session.refresh(model)
```

正しいコード:

```python
await session.commit()
await session.refresh(model)
```

## 3. 重複 await

誤り:

```python
await await session.commit()
```

正しいコード:

```python
await session.commit()
```

## 4. `add()`にawaitを使用

誤り:

```python
await session.add(model)
```

正しいコード:

```python
session.add(model)
```

## 5. Resolverのasync漏れ

DB serviceがasyncならresolverも:

```python
@strawberry.mutation
async def delete_issue(...):
    return await issue_service.delete_issue(...)
```

である必要がある。

## 6. `owner`がGraphQL schemaにない

`owner()` resolverは`Issue` Classの外に置いてはいけない。

正しいコード:

```python
@strawberry.type
class Issue:
    ...

    @strawberry.field
    async def owner(...):
        ...
```

## 7. `owner_id NOT NULL`

7段階以前:

```text
owner_id column自体がない
```

7段階:

```text
seed User + 一時 owner_id
```

8段階 以降:

```text
owner_id = current_user.id
```

---

# 最終学習チェック

以下を説明できる必要がある。

```text
Query
→ Resolver
→ Async Service
→ AsyncSession
→ SELECT

Mutation
→ Resolver
→ Async Service
→ AsyncSession
→ INSERT / UPDATE / DELETE
```

Create:

```text
session.add()
await commit()
await refresh()
```

Delete:

```text
await session.delete()
await session.commit()
```

DataLoader:

```text
owner resolver
→ user_loader.load(owner_id)
→ batch
→ WHERE id IN (...)
```

JWT:

```text
email/password
→ login
→ JWT sub=user.id
→ Authorization header
→ context.current_user
→ owner-based authorization
```

---

# このGuideの不変ルール

```text
1~4段階
Memoryベース → sync関数でよい

5~12段階
PostgreSQL ベース → Async SQLAlchemy only
```

つまり5段階以降は:

```text
create_async_engine
async_sessionmaker
async with SessionLocal()
async def Resolver
async def Service
await execute/get/commit/refresh/delete
session.add()のみawaitなし
```

このルールから外れたcodeが出たら、まずエラーを疑う。

# Phase 1 完了基準

既存の1〜12段階の実装順序を維持しながら、以下を自分で確認する。

```text
Frontend
Next.js + Tailwind

API
GraphQL + FastAPI

Database
PostgreSQL + SQLAlchemy Async

Authentication
JWT + Context + Authorization

Testing
pytest + pytest-asyncio + httpx
Vitest + React Testing Library
Playwright
```

最終的に、Frontendで使用するIssue Fieldが次の全経路で一致していることも確認する。

```text
GraphQL Issue Schema
→ services.to_issue()
→ GetIssues / CreateIssue / UpdateIssue
→ npm run codegen
→ GetIssuesQuery由来のIssue型
→ IssuesPage

id / title / description / status / createdAt / owner
```

また、10段階で`issues`がConnectionへ変わった後は、Integration TestもConnection shapeへ更新されていることを確認する。

Phase 1のDocker / AWS部分は、全体の接続を確認するための入門レベルとする。
ProductionレベルのContainer、AWS、Terraform、CI/CDは**Phase 4**で改めて深く扱う。
