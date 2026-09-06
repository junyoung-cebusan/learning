# GraphQL 풀스택 1~12단계 실습 가이드 — Async 검증판

> 목표: `src/app/graphql`, `models`, `schemas`, `services` 구조를 유지하면서 직접 타이핑해보며  
> FastAPI → Strawberry GraphQL → SQLAlchemy Async → PostgreSQL → DataLoader → JWT → Next.js → Docker → AWS 흐름을 익힌다.
>
> **중요:** 5단계에서 PostgreSQL을 붙이는 순간부터 12단계까지 SQLAlchemy는 **Async 전용**으로 사용한다.
> Sync `Session` 코드와 `AsyncSession` 코드를 섞지 않는다.

---

# 최종 구조

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
│           ├── graphql/
│           │   ├── __init__.py
│           │   ├── query.py
│           │   ├── mutation.py
│           │   └── context.py
│           ├── models/
│           │   ├── __init__.py
│           │   ├── issue.py
│           │   └── user.py
│           ├── schemas/
│           │   ├── __init__.py
│           │   ├── issue.py
│           │   ├── user.py
│           │   └── auth.py
│           └── services/
│               ├── __init__.py
│               ├── issue.py
│               └── auth.py
├── frontend/
├── docker-compose.yml
└── .github/
    └── workflows/
        └── ci.yml
```

레이어:

```text
graphql/  → Query / Mutation Resolver
schemas/  → GraphQL Type / Input
services/ → Business Logic / DB Access
models/   → SQLAlchemy ORM Model
```

최종 요청 흐름:

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

# 0. 프로젝트 설정

## `pyproject.toml`

프로젝트 이름은 `backend`, 실제 import package는 `app`으로 사용한다.

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

설치:

```bash
uv add fastapi uvicorn
uv add "strawberry-graphql[fastapi]"
uv add "sqlalchemy[asyncio]"
uv add asyncpg
```

실행:

```bash
uv run uvicorn app.main:app --reload
```

---

# 1단계 — FastAPI

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

실행:

```bash
uv run uvicorn app.main:app --reload
```

확인:

```text
http://localhost:8000/health
```

---

# 2단계 — Strawberry GraphQL

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

아직 실제 Mutation이 없으므로 확인용 `ping`을 둔다.

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

app.include_router(
    graphql_app,
    prefix="/graphql",
)
```

## GraphiQL 확인

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

# 3단계 — Issue Query

DB는 아직 붙이지 않는다.

## `schemas/issue.py`

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

> `status` 오타에 주의한다. `statue`가 아니다.

## `services/issue.py`

```python
from datetime import (
    datetime,
    timezone,
)

from app.schemas.issue import Issue


issues: list[Issue] = [
    Issue(
        id=1,
        title="GraphQL 공부",
        description="Resolver 복습",
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

from app.schemas.issue import Issue
from app.services import issue as issue_service


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

## GraphiQL 확인 — Read

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

# 4단계 — Mutation / Memory CRUD

이 단계까지는 DB가 없으므로 sync 함수로 충분하다.

## `services/issue.py`

```python
from app.schemas.issue import (
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

from app.schemas.issue import (
    CreateIssueInput,
    Issue,
    UpdateIssueInput,
)
from app.services import issue as issue_service


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

## GraphiQL 확인 — Memory CRUD

### Create

```graphql
mutation {
  createIssue(
    input: {
      title: "Memory CRUD"
      description: "Create 확인"
    }
  ) {
    id
    title
    description
    status
  }
}
```

### Read List

```graphql
query {
  issues {
    id
    title
    status
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
  updateIssue(
    id: 2
    input: {
      title: "Memory CRUD 수정"
      status: "DONE"
    }
  ) {
    id
    title
    status
  }
}
```

### Delete

```graphql
mutation {
  deleteIssue(id: 2)
}
```

### Delete 확인

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
> Memory CRUDが完成した時点で、最も基本的なTestだけを追加する。

## Test Tool

```bash
uv add --dev pytest pytest-asyncio httpx
```

Phase 1ではBackend Testを以下の役割に分ける。

```text
pytest
→ Unit / Integration Test

pytest-asyncio
→ async function 테스트

httpx
→ FastAPI / GraphQL HTTP Integration Test
```

## 最初のTest対象

最初はDBより単純なMemory ServiceからTestする。

```text
create_issue()
get_issues()
get_issue()
update_issue()
delete_issue()
```

重要なのはFrameworkそのものをTestすることではなく、**自分たちのコードの振る舞いを検証すること**である。

예:

```python
def test_create_issue():
    # arrange
    # act
    # assert
    ...
```

この段階ではTestの基本構造とArrange / Act / Assertの流れを理解する。

---

# 5단계 — PostgreSQL + SQLAlchemy Async

> **이 단계부터 끝까지 Async SQLAlchemy만 사용한다.**

## PostgreSQL

root의 `docker-compose.yml`:

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

실행:

```bash
docker compose up -d
```

## `database.py`

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

## AsyncSession 규칙

```python
session.add(model)            # await X

await session.get(...)        # await O
await session.execute(...)    # await O
await session.commit()        # await O
await session.refresh(model)  # await O
await session.delete(model)   # await O
await session.rollback()      # await O
```

잘못된 코드:

```python
await session.add(model)
```

잘못된 코드:

```python
await await session.commit()
```

잘못된 코드:

```python
with SessionLocal() as session:
```

정상:

```python
async with SessionLocal() as session:
```

---

## `models/issue.py`

7단계 전까지는 `owner_id`를 만들지 않는다.

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

app.include_router(
    graphql_app,
    prefix="/graphql",
)
```

## `services/issue.py` — Async CRUD로 교체

```python
from sqlalchemy import select

from app.database import SessionLocal
from app.models.issue import IssueModel
from app.schemas.issue import (
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

## `graphql/query.py` — 전부 async

```python
import strawberry

from app.schemas.issue import Issue
from app.services import issue as issue_service


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

## `graphql/mutation.py` — 전부 async

```python
import strawberry

from app.schemas.issue import (
    CreateIssueInput,
    Issue,
    UpdateIssueInput,
)
from app.services import issue as issue_service


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

> DB 단계 이후 `create_issue`, `update_issue`, `delete_issue` Resolver는 모두 `async def`다.

## GraphiQL 확인 — PostgreSQL CRUD

Create:

```graphql
mutation {
  createIssue(
    input: {
      title: "PostgreSQL CRUD"
      description: "INSERT 확인"
    }
  ) {
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
  updateIssue(
    id: 1
    input: {
      title: "PostgreSQL CRUD 수정"
      status: "DONE"
    }
  ) {
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

# 6단계 — SQL / Index / Transaction

DB 접속:

```bash
docker compose exec postgres \
  psql -U app -d app
```

조회:

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

실행 계획:

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

다시:

```sql
BEGIN;

UPDATE issues
SET status = 'DONE'
WHERE id = 1;

COMMIT;
```

## SQLAlchemy transaction 예

```python
async with SessionLocal() as session:
    try:
        # 여러 DB 작업
        await session.commit()

    except Exception:
        await session.rollback()
        raise
```

## GraphiQL 확인 — CRUD + SQL Log

Create:

```graphql
mutation {
  createIssue(
    input: {
      title: "SQL 관찰용 Issue"
      description: "INSERT 로그 확인"
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

로그:

```text
Create → INSERT
Read   → SELECT
Update → UPDATE
Delete → DELETE
```

---

# 7단계 — User 관계 / N+1 / DataLoader

관계:

```text
User 1 ─── N Issue
```

이 단계에서 처음 `owner_id`를 추가한다.

학습 중이라 migration 대신 DB 초기화가 가능하다.

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

추가:

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship
```

클래스 내부:

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

## `schemas/user.py`

```python
import strawberry


@strawberry.type
class User:
    id: int
    name: str
    email: str
```

## `schemas/issue.py`

> `owner()`는 **반드시 `Issue` 클래스 내부에 들여쓰기해서 정의한다.**

```python
from datetime import datetime

import strawberry

from app.schemas.user import User


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

## `to_issue()` 수정

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

## JWT 전 임시 User seed

`owner_id`가 `NOT NULL`이므로 먼저 User가 필요하다.

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

확인:

```sql
SELECT *
FROM users;
```

user id가 `1`이라고 가정한다.

JWT 전까지만 `create_issue()`에서 임시 owner id를 넣는다.

```python
model = IssueModel(
    title=input.title,
    description=input.description,
    status="OPEN",
    owner_id=1,
)
```

> 8단계에서는 이 하드코딩을 제거하고 `current_user.id`를 사용한다.

## GraphiQL 확인 — Relation / DataLoader

Create:

```graphql
mutation {
  createIssue(
    input: {
      title: "DataLoader Issue"
      description: "Owner relation"
    }
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

목표 SQL:

```text
issues SELECT 1번
users SELECT ... WHERE id IN (...) 1번
```

---

# 8단계 — JWT Authentication / Authorization

설치:

```bash
uv add pyjwt "pwdlib[argon2]"
```

## `models/user.py`

추가:

```python
password_hash: Mapped[str] = mapped_column(
    String(512),
    nullable=False,
)
```

학습 중이면 DB를 다시 초기화한다.

## `security.py`

```python
from datetime import (
    datetime,
    timedelta,
    timezone,
)

import jwt

from pwdlib import PasswordHash


JWT_SECRET = "dev-secret-change-me"
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

## `schemas/auth.py`

```python
import strawberry

from app.schemas.user import User


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
from app.schemas.auth import (
    AuthPayload,
    LoginInput,
    RegisterInput,
)
from app.schemas.user import User
from app.security import (
    create_access_token,
    hash_password,
    verify_password,
)


async def register(
    input: RegisterInput,
) -> AuthPayload:
    async with SessionLocal() as session:
        model = UserModel(
            name=input.name,
            email=input.email.lower(),
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

## `graphql/context.py` — JWT 적용

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

## `services/issue.py` — owner 기반 CRUD

Create:

```python
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
```

Update:

```python
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
```

Delete:

```python
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

## `graphql/mutation.py` — 인증 CRUD

```python
import strawberry

from app.schemas.auth import (
    AuthPayload,
    LoginInput,
    RegisterInput,
)
from app.schemas.issue import (
    CreateIssueInput,
    Issue,
    UpdateIssueInput,
)
from app.services import auth as auth_service
from app.services import issue as issue_service


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

현재는 access-token-only stateless JWT이므로 서버 Mutation이 필수는 아니다.

```text
Logout
→ Client에서 access token 삭제
```

Next.js:

```typescript
localStorage.removeItem(
  "accessToken",
);
```

Refresh Token / Redis session을 추가하는 Senior 단계에서는 서버-side revoke/logout을 다룬다.

## GraphiQL 확인 — JWT CRUD

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
  login(
    input: {
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
    input: {
      title: "JWT CRUD"
      description: "Authenticated Create"
    }
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

실패 케이스:

```text
Header 없음        → Authentication 실패
잘못된 JWT         → Authentication 실패
다른 User의 Issue  → Authorization 실패
```

---


# 8段階拡張 — Authentication / GraphQL Integration Test

JWTまで実装した後は、Service単位のTestから一段階進み、実際のGraphQL Requestを検証する。

## 確認するScenario

```text
Register
→ Login
→ JWT 발급
→ Authorization Header
→ createIssue
→ current_user
→ owner_id 저장
```

最低限、以下のCaseをTestする。

```text
1. 정상 로그인
2. 잘못된 비밀번호
3. Token 없이 인증 Mutation 호출
4. 정상 Token으로 Issue 생성
5. 생성된 Issue의 owner 확인
6. 존재하지 않는 User의 Token 처리
```

## Test Level

```text
Unit Test
Service 하나의 로직

Integration Test
GraphQL + Context + Service + Database 연결

E2E
Frontend까지 포함한 실제 사용자 흐름
```

この段階では特に、**Unit TestとIntegration Testの違い**を区別して理解する。

---

# 9단계 — Next.js + Tailwind로 실제 CRUD 화면 만들기

> 목표: GraphiQL에서 확인하던 JWT 로그인과 Issue CRUD를 이번에는 **브라우저 화면에서 직접 확인**한다.
>
> 이 단계에서는 학습 범위를 불필요하게 넓히지 않기 위해 Apollo Client, urql, React Query, Zustand,
> React Hook Form 같은 라이브러리를 추가하지 않는다.
>
> 사용하는 것은:
>
> ```text
> Next.js App Router
> TypeScript
> Tailwind CSS
> Browser fetch
> localStorage
> ```
>
> 뿐이다.

---

## 9-1. Next.js 프로젝트 생성

프로젝트 root에서:

```bash
npx create-next-app@latest frontend \
  --typescript \
  --eslint \
  --tailwind \
  --app \
  --src-dir \
  --use-npm
```

현재 `create-next-app`은 TypeScript, Tailwind CSS, App Router를 공식적으로 지원한다.

생성 후:

```bash
cd frontend
npm run dev
```

확인:

```text
http://localhost:3000
```

이번 단계의 frontend 구조:

```text
frontend/
├── .env.local
├── src/
│   ├── app/
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── issues/
│   │       └── page.tsx
│   └── lib/
│       └── graphql.ts
└── package.json
```

---

## 9-2. Backend CORS

브라우저의 Next.js application은:

```text
http://localhost:3000
```

Backend는:

```text
http://localhost:8000
```

이므로 개발 환경에서는 CORS 허용이 필요하다.

Backend `src/app/main.py`:

```python
from fastapi.middleware.cors import (
    CORSMiddleware,
)
```

`app = FastAPI(...)` 생성 후:

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

개발 단계에서는 이렇게 사용하지만 production에서는 허용 origin을 실제 frontend domain으로 제한한다.

---

## 9-3. GraphQL URL 환경변수

`frontend/.env.local`:

```env
NEXT_PUBLIC_GRAPHQL_URL=http://localhost:8000/graphql
```

`NEXT_PUBLIC_` prefix가 붙은 환경변수는 browser/client 코드에서도 사용할 수 있다.

환경변수를 추가한 뒤에는 dev server를 재시작한다.

```bash
npm run dev
```

---

## 9-4. 최소 GraphQL Request Helper

`src/lib/graphql.ts`:

```typescript
const GRAPHQL_URL =
  process.env.NEXT_PUBLIC_GRAPHQL_URL
  ?? "http://localhost:8000/graphql";


type GraphQLError = {
  message: string;
};


type GraphQLResponse<T> = {
  data?: T;
  errors?: GraphQLError[];
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
        "Content-Type": "application/json",

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
            error.message,
        )
        .join("
"),
    );
  }

  if (!result.data) {
    throw new Error(
      "GraphQL response has no data",
    );
  }

  return result.data;
}
```

흐름:

```text
React Component
  ↓
graphqlRequest()
  ↓
POST /graphql
  ↓
Authorization: Bearer JWT
  ↓
FastAPI / Strawberry
```

별도 GraphQL client library 없이도 GraphQL은 결국 HTTP POST 요청이므로 충분히 실습할 수 있다.

---

## 9-5. Root 페이지

`src/app/page.tsx`:

```tsx
import Link from "next/link";


export default function Home() {
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
}
```

여기서는 routing만 확인한다.

---

# 9-6. Login 화면

이 화면에서는:

```text
email/password 입력
  ↓
login Mutation
  ↓
accessToken 응답
  ↓
localStorage 저장
  ↓
/issues 이동
```

을 확인한다.

`src/app/login/page.tsx`:

```tsx
"use client";

import {
  FormEvent,
  useState,
} from "react";

import {
  useRouter,
} from "next/navigation";

import {
  graphqlRequest,
} from "@/lib/graphql";


const LOGIN = `
  mutation Login(
    $input: LoginInput!
  ) {
    login(
      input: $input
    ) {
      accessToken

      user {
        id
        name
        email
      }
    }
  }
`;


type LoginResponse = {
  login: {
    accessToken: string;

    user: {
      id: number;
      name: string;
      email: string;
    };
  } | null;
};


export default function LoginPage() {
  const router = useRouter();

  const [
    email,
    setEmail,
  ] = useState("");

  const [
    password,
    setPassword,
  ] = useState("");

  const [
    error,
    setError,
  ] = useState("");

  const [
    loading,
    setLoading,
  ] = useState(false);


  async function handleSubmit(
    event: FormEvent<HTMLFormElement>,
  ) {
    event.preventDefault();

    setError("");
    setLoading(true);

    try {
      const data =
        await graphqlRequest<LoginResponse>(
          LOGIN,
          {
            input: {
              email,
              password,
            },
          },
        );

      if (!data.login) {
        throw new Error(
          "Login failed",
        );
      }

      localStorage.setItem(
        "accessToken",
        data.login.accessToken,
      );

      router.push(
        "/issues",
      );
    } catch (error) {
      setError(
        error instanceof Error
          ? error.message
          : "Unknown error",
      );
    } finally {
      setLoading(false);
    }
  }


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
        onSubmit={
          handleSubmit
        }
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
            onChange={
              (event) =>
                setEmail(
                  event.target.value,
                )
            }
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
            onChange={
              (event) =>
                setPassword(
                  event.target.value,
                )
            }
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
          {
            loading
              ? "Logging in..."
              : "Login"
          }
        </button>
      </form>
    </main>
  );
}
```

---

# 9-7. Issue CRUD 화면

하나의 화면에서 CRUD를 모두 확인한다.

```text
READ
Issue 목록

CREATE
새 Issue form

UPDATE
DONE 버튼

DELETE
Delete 버튼
```

복잡한 state library는 쓰지 않고 React `useState`만 사용한다.

`src/app/issues/page.tsx`:

```tsx
"use client";

import {
  FormEvent,
  useCallback,
  useEffect,
  useState,
} from "react";

import {
  useRouter,
} from "next/navigation";

import {
  graphqlRequest,
} from "@/lib/graphql";


type Issue = {
  id: number;
  title: string;
  description: string | null;
  status: string;

  owner: {
    id: number;
    name: string;
    email: string;
  };
};


const GET_ISSUES = `
  query GetIssues {
    issues {
      id
      title
      description
      status

      owner {
        id
        name
        email
      }
    }
  }
`;


const CREATE_ISSUE = `
  mutation CreateIssue(
    $input: CreateIssueInput!
  ) {
    createIssue(
      input: $input
    ) {
      id
      title
      description
      status

      owner {
        id
        name
        email
      }
    }
  }
`;


const UPDATE_ISSUE = `
  mutation UpdateIssue(
    $id: Int!
    $input: UpdateIssueInput!
  ) {
    updateIssue(
      id: $id
      input: $input
    ) {
      id
      title
      description
      status
    }
  }
`;


const DELETE_ISSUE = `
  mutation DeleteIssue(
    $id: Int!
  ) {
    deleteIssue(
      id: $id
    )
  }
`;


type IssuesResponse = {
  issues: Issue[];
};


type CreateIssueResponse = {
  createIssue: Issue;
};


type UpdateIssueResponse = {
  updateIssue: Issue | null;
};


type DeleteIssueResponse = {
  deleteIssue: boolean;
};


export default function IssuesPage() {
  const router = useRouter();

  const [
    issues,
    setIssues,
  ] = useState<Issue[]>([]);

  const [
    title,
    setTitle,
  ] = useState("");

  const [
    description,
    setDescription,
  ] = useState("");

  const [
    loading,
    setLoading,
  ] = useState(true);

  const [
    saving,
    setSaving,
  ] = useState(false);

  const [
    error,
    setError,
  ] = useState("");


  const loadIssues =
    useCallback(
      async () => {
        setError("");

        try {
          const data =
            await graphqlRequest<
              IssuesResponse
            >(
              GET_ISSUES,
            );

          setIssues(
            data.issues,
          );
        } catch (error) {
          setError(
            error instanceof Error
              ? error.message
              : "Unknown error",
          );
        } finally {
          setLoading(false);
        }
      },
      [],
    );


  useEffect(
    () => {
      const token =
        localStorage.getItem(
          "accessToken",
        );

      if (!token) {
        router.replace(
          "/login",
        );

        return;
      }

      void loadIssues();
    },
    [
      loadIssues,
      router,
    ],
  );


  async function handleCreate(
    event: FormEvent<HTMLFormElement>,
  ) {
    event.preventDefault();

    if (!title.trim()) {
      return;
    }

    setSaving(true);
    setError("");

    try {
      const data =
        await graphqlRequest<
          CreateIssueResponse
        >(
          CREATE_ISSUE,
          {
            input: {
              title,
              description:
                description || null,
            },
          },
        );

      setIssues(
        (current) => [
          ...current,
          data.createIssue,
        ],
      );

      setTitle("");
      setDescription("");
    } catch (error) {
      setError(
        error instanceof Error
          ? error.message
          : "Unknown error",
      );
    } finally {
      setSaving(false);
    }
  }


  async function markDone(
    issueId: number,
  ) {
    setError("");

    try {
      const data =
        await graphqlRequest<
          UpdateIssueResponse
        >(
          UPDATE_ISSUE,
          {
            id: issueId,

            input: {
              status: "DONE",
            },
          },
        );

      if (!data.updateIssue) {
        return;
      }

      setIssues(
        (current) =>
          current.map(
            (issue) =>
              issue.id === issueId
                ? {
                    ...issue,
                    status:
                      data.updateIssue!
                        .status,
                  }
                : issue,
          ),
      );
    } catch (error) {
      setError(
        error instanceof Error
          ? error.message
          : "Unknown error",
      );
    }
  }


  async function removeIssue(
    issueId: number,
  ) {
    setError("");

    try {
      const data =
        await graphqlRequest<
          DeleteIssueResponse
        >(
          DELETE_ISSUE,
          {
            id: issueId,
          },
        );

      if (!data.deleteIssue) {
        return;
      }

      setIssues(
        (current) =>
          current.filter(
            (issue) =>
              issue.id !== issueId,
          ),
      );
    } catch (error) {
      setError(
        error instanceof Error
          ? error.message
          : "Unknown error",
      );
    }
  }


  function logout() {
    localStorage.removeItem(
      "accessToken",
    );

    router.replace(
      "/login",
    );
  }


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
        onSubmit={
          handleCreate
        }
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
          onChange={
            (event) =>
              setTitle(
                event.target.value,
              )
          }
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
          onChange={
            (event) =>
              setDescription(
                event.target.value,
              )
          }
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
          {
            saving
              ? "Creating..."
              : "Create"
          }
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
        <p>
          Loading...
        </p>
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

          {issues.map(
            (issue) => (
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
                        {
                          issue.description
                        }
                      </p>
                    )}

                    <p
                      className="
                        mt-2
                        text-xs
                        text-gray-400
                      "
                    >
                      Owner:
                      {" "}
                      {issue.owner.name}
                    </p>
                  </div>

                  <div
                    className="
                      flex
                      shrink-0
                      gap-2
                    "
                  >
                    {issue.status !==
                      "DONE" && (
                      <button
                        type="button"
                        onClick={
                          () =>
                            void markDone(
                              issue.id,
                            )
                        }
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
                      onClick={
                        () =>
                          void removeIssue(
                            issue.id,
                          )
                      }
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
            ),
          )}
        </section>
      )}
    </main>
  );
}
```

---

# 9-8. 실제로 확인할 CRUD 흐름

Backend와 frontend를 둘 다 실행한다.

Backend:

```bash
cd backend

uv run uvicorn \
  app.main:app \
  --reload
```

Frontend:

```bash
cd frontend

npm run dev
```

브라우저:

```text
http://localhost:3000/login
```

## 1. Login

이미 GraphiQL에서 `register`한 사용자로 로그인한다.

예:

```text
hwang@example.com
password123
```

성공하면:

```text
localStorage
└── accessToken
```

이 저장되고 `/issues`로 이동해야 한다.

Browser DevTools에서도 확인한다.

```text
Application
→ Local Storage
→ http://localhost:3000
→ accessToken
```

---

## 2. Read

`/issues` 진입 시:

```graphql
query GetIssues {
  issues {
    id
    title
    description
    status
    owner {
      id
      name
      email
    }
  }
}
```

가 호출된다.

화면에 DB의 Issue 목록이 나타나는지 확인한다.

---

## 3. Create

Create Issue form에:

```text
Title:
Frontend CRUD

Description:
Next.js에서 생성
```

입력하고 Create를 누른다.

화면에 바로 새 Issue가 추가되어야 한다.

PostgreSQL에서도 확인한다.

```sql
SELECT
    id,
    title,
    status,
    owner_id
FROM issues
ORDER BY id;
```

---

## 4. Update

생성한 Issue의:

```text
Done
```

버튼을 누른다.

GraphQL:

```graphql
mutation UpdateIssue(
  $id: Int!
  $input: UpdateIssueInput!
) {
  updateIssue(
    id: $id
    input: $input
  ) {
    id
    status
  }
}
```

variables:

```json
{
  "id": 1,
  "input": {
    "status": "DONE"
  }
}
```

화면의 status가:

```text
OPEN
→ DONE
```

으로 바뀌어야 한다.

---

## 5. Delete

Delete를 누르면:

```graphql
mutation DeleteIssue(
  $id: Int!
) {
  deleteIssue(
    id: $id
  )
}
```

가 실행된다.

성공하면 해당 card가 화면에서 사라져야 한다.

---

## 6. Logout

Logout:

```typescript
localStorage.removeItem(
  "accessToken",
);
```

그리고:

```text
/issues
→ /login
```

으로 이동한다.

현재 단계의 JWT는 access-token-only이므로 이것이 logout이다.

---

# 9-9. Browser Network에서 GraphQL 확인

Chrome DevTools:

```text
Network
→ graphql
```

요청을 확인한다.

Request Headers:

```text
Authorization:
Bearer eyJ...
```

Request Payload:

```json
{
  "query": "...",
  "variables": {
    "input": {
      "title": "Frontend CRUD"
    }
  }
}
```

Response:

```json
{
  "data": {
    "createIssue": {
      "id": 1,
      "title": "Frontend CRUD"
    }
  }
}
```

GraphiQL에서 보던 GraphQL 요청이 실제 browser HTTP request로 어떻게 전송되는지 확인하는 것이 중요하다.

---

# 9-10. 인증 실패도 화면에서 확인

DevTools → Application → Local Storage에서:

```text
accessToken
```

을 삭제하고 `/issues`를 새로고침한다.

현재 frontend 코드에서는 token이 없으므로:

```text
/issues
→ /login
```

으로 이동한다.

그 다음 token을 임의로:

```text
abc
```

처럼 넣은 뒤 API를 호출하면 backend JWT 검증이 실패해야 한다.

이 차이를 이해한다.

```text
Token 없음
→ frontend에서 login 페이지로 이동

잘못된 Token
→ backend에서 Authentication 실패
→ GraphQL errors
→ frontend error message
```

---

# 9-11. 이 단계에서 일부러 사용하지 않는 것

이번 단계에서는 다음을 추가하지 않는다.

```text
Apollo Client
urql
TanStack Query
Zustand
Redux
React Hook Form
Zod
shadcn/ui
MUI
```

이유:

```text
지금의 목표
= Next.js에서 GraphQL/JWT/CRUD 흐름을 직접 보는 것
```

이기 때문이다.

먼저:

```text
useState
useEffect
fetch
localStorage
```

만으로 전체 흐름을 이해한다.

이후 frontend를 고도화할 때:

```text
GraphQL Client
Server State Cache
Form Validation
UI Component Library
```

를 추가하면 된다.

---

# 9단계 완료 기준

다음을 직접 설명할 수 있어야 한다.

```text
Login Form
  ↓
login Mutation
  ↓
JWT
  ↓
localStorage
  ↓
graphqlRequest()
  ↓
Authorization Header
  ↓
GraphQL Context
  ↓
current_user
```

그리고 CRUD:

```text
Create
→ createIssue Mutation

Read
→ issues Query

Update
→ updateIssue Mutation

Delete
→ deleteIssue Mutation
```

를 브라우저 화면과 Network tab에서 모두 확인해야 한다.

마지막으로:

```text
GraphiQL에서는 성공
Next.js에서는 실패
```

한다면 Backend API보다:

```text
CORS
Authorization header
localStorage
Client Component
fetch
frontend state
```

를 먼저 확인할 수 있어야 한다.

---



# 9段階拡張 — Frontend Testing

Next.jsのCRUD画面まで完成した後、Frontend Testを追加する。

## Test Stack

```text
Vitest
→ Unit Test

React Testing Library
→ Component Test

Playwright
→ E2E Test
```

설치:

```bash
npm install -D vitest jsdom \
  @testing-library/react \
  @testing-library/jest-dom \
  @testing-library/user-event

npm install -D @playwright/test
npx playwright install
```

## Vitest

まずは以下のようなUIから独立したLogicをTestする。

```text
GraphQL response 변환
utility
validation
상태 변환 함수
```

Frameworkを大量にMockするより、**Pure Functionを簡単にTestできる構造**を優先する。

## React Testing Library

Component内部の実装詳細より、Userから見える振る舞いをTestする。

예:

```text
Create form을 입력한다
→ Create 버튼을 누른다
→ loading 상태가 보인다
→ 성공 후 입력값이 초기화된다
```

## Playwright E2E

Phase 1の主要E2E Scenario:

```text
Login
  ↓
Issue 목록 확인
  ↓
Create
  ↓
Update
  ↓
Delete
  ↓
Logout
```

Playwrightでは実際のBrowser上で動作を確認する。

## Phase 1 Test Pyramid

```text
        E2E
     Playwright
        ▲
   Integration
 GraphQL / httpx
        ▲
      Unit
pytest / Vitest
```

E2Eだけを大量に作るのではなく、
高速なUnit TestとIntegration Testを中心にし、
主要なUser FlowのみをE2Eで検証する。

---

# 10단계 — Filter / Search / Cursor Pagination

## Service query

```python
stmt = (
    select(IssueModel)
    .where(
        IssueModel.owner_id
        == owner_id
    )
)

if status is not None:
    stmt = stmt.where(
        IssueModel.status
        == status
    )

if search:
    stmt = stmt.where(
        IssueModel.title.ilike(
            f"%{search}%"
        )
    )
```

## Cursor

```python
import base64


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

    _, issue_id = raw.split(
        ":"
    )

    return int(
        issue_id
    )
```

Cursor SQL 개념:

```sql
SELECT *
FROM issues
WHERE owner_id = 1
AND id > 100
ORDER BY id
LIMIT 20;
```

## GraphiQL 확인

Filter:

```graphql
query {
  issues(
    status: "OPEN"
  ) {
    id
    title
    status
  }
}
```

Search:

```graphql
query {
  issues(
    search: "GraphQL"
  ) {
    id
    title
  }
}
```

Update 후 filter 재확인:

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

---

# 11단계 — Docker Compose + CI

## Backend Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN pip install uv

COPY pyproject.toml uv.lock ./

RUN uv sync --frozen

COPY src ./src

EXPOSE 8000

CMD [
  "uv",
  "run",
  "uvicorn",
  "app.main:app",
  "--host",
  "0.0.0.0",
  "--port",
  "8000"
]
```

## CI 개념

```text
push
 ↓
uv sync
 ↓
lint
 ↓
test
 ↓
docker build
```

예:

```yaml
name: CI

on:
  push:
  pull_request:

jobs:
  backend:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v6

      - run: |
          cd backend
          uv sync --frozen

      - run: |
          cd backend
          uv run python -m compileall src
```

---

# 12단계 — AWS 배포 방향

학습용 기본 architecture:

```text
Internet
   ↓
ALB
   ↓
ECS Fargate
   ↓
FastAPI
   ↓
RDS PostgreSQL
```

Container:

```text
Docker Image
   ↓
ECR
   ↓
ECS
```

CI/CD:

```text
GitHub Actions
   ↓
Test
   ↓
Docker Build
   ↓
ECR Push
   ↓
ECS Deploy
```

다음 Senior 단계:

```text
Redis / ElastiCache
Kafka / Queue
Terraform
Observability
Large-scale Data Processing
OAuth/OIDC
Rate Limiting
System Design
```

---

# Async 오류 체크리스트

## 1. `AsyncSession` context manager

잘못:

```python
with SessionLocal() as session:
```

정상:

```python
async with SessionLocal() as session:
```

## 2. commit / refresh await 누락

잘못:

```python
session.commit()
session.refresh(model)
```

정상:

```python
await session.commit()
await session.refresh(model)
```

## 3. 중복 await

잘못:

```python
await await session.commit()
```

정상:

```python
await session.commit()
```

## 4. `add()`에 await 사용

잘못:

```python
await session.add(model)
```

정상:

```python
session.add(model)
```

## 5. Resolver async 누락

DB service가 async이면 resolver도:

```python
@strawberry.mutation
async def delete_issue(...):
    return await issue_service.delete_issue(...)
```

여야 한다.

## 6. `owner`가 GraphQL schema에 없음

`owner()` resolver가 `Issue` 클래스 밖에 있으면 안 된다.

정상:

```python
@strawberry.type
class Issue:
    ...

    @strawberry.field
    async def owner(...):
        ...
```

## 7. `owner_id NOT NULL`

7단계 이전:

```text
owner_id column 자체가 없음
```

7단계:

```text
seed User + 임시 owner_id
```

8단계 이후:

```text
owner_id = current_user.id
```

---

# 최종 학습 체크

아래를 설명할 수 있어야 한다.

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

# 이 가이드의 불변 규칙

```text
1~4단계
Memory 기반 → sync 함수 가능

5~12단계
PostgreSQL 기반 → Async SQLAlchemy only
```

즉 5단계 이후에는:

```text
create_async_engine
async_sessionmaker
async with SessionLocal()
async def Resolver
async def Service
await execute/get/commit/refresh/delete
session.add()만 await 없음
```

이 규칙에서 벗어난 코드가 나오면 먼저 오류를 의심한다.


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

Phase 1のDocker / AWS部分は、全体の接続を確認するための入門レベルとする。
ProductionレベルのContainer、AWS、Terraform、CI/CDは**Phase 4**で改めて深く扱う。
