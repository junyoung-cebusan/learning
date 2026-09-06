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

# Step 14 — Next.js + Tailwindで実際の画面を作る

## Goal

GraphiQLで確認してきたLogin / JWT / Issue CRUDを、今度はBrowser上の実画面から操作する。

この段階ではFrontend側の仕組みを見えやすくするため、Apollo Client、urql、TanStack Query、Zustandなどはまだ追加しない。

```text
Next.js App Router
TypeScript
Tailwind CSS
fetch
localStorage
```

だけで完成させる。

> Phase 1ではJWTの仕組みを確認しやすくするため`localStorage`を使用する。HttpOnly Cookie / Refresh Token / CSRFはPhase 5で扱う。

---

## Step 14.1 — Next.js Project作成

Project rootで実行する。

```bash
yarn create next-app frontend \
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

## Step 14.2 — Backend CORS

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

## Step 14.3 — GraphQL URL

`frontend/.env.local`:

```env
NEXT_PUBLIC_GRAPHQL_URL=http://localhost:8000/graphql
```

Environment Variable追加後はDev Serverを再起動する。

```bash
npm run dev
```

---

## Step 14.4 — graphql-request + `.graphql` + GraphQL Code Generator

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

`src/graphql/login.graphql`:

```graphql
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
```

`src/graphql/issues.graphql`:

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

`src/graphql/create-issue.graphql`:

```graphql
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
```

`src/graphql/update-issue.graphql`:

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
    title
    description
    status
  }
}
```

`src/graphql/delete-issue.graphql`:

```graphql
mutation DeleteIssue(
  $id: Int!
) {
  deleteIssue(
    id: $id
  )
}
```

### Codegen Config

`frontend/codegen.ts`:

```typescript
import type {
  CodegenConfig,
} from "@graphql-codegen/cli";


const config:
  CodegenConfig = {
    schema:
      "http://localhost:8000/graphql",

    documents:
      "src/graphql/**/*.graphql",

    generates: {
      "src/generated/graphql.ts": {
        plugins: [
          "typescript",
          "typescript-operations",
          "typed-document-node",
        ],
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
import {
  LoginDocument,
} from "@/generated/graphql";

import {
  getGraphQLClient,
} from "@/lib/graphql-client";


const data =
  await getGraphQLClient().request(
    LoginDocument,
    {
      input: {
        email,
        password,
      },
    },
  );


localStorage.setItem(
  "accessToken",
  data.login.accessToken,
);
```

ここでは`data`に手動で型を書く必要がない。

```typescript
data.login.accessToken
data.login.user.id
data.login.user.name
data.login.user.email
```

がCodegenによって型付けされる。

存在しないFieldを書いた場合:

```typescript
data.login.user.username
```

Schema / Operationに`username`が存在しなければTypeScript Errorになる。

---

#### Variablesの型もCodegenから取得できる

必要であれば生成されたVariables Typeを明示的に利用できる。

```typescript
import type {
  LoginMutationVariables,
} from "@/generated/graphql";


const variables:
  LoginMutationVariables = {
    input: {
      email,
      password,
    },
  };


const data =
  await getGraphQLClient().request(
    LoginDocument,
    variables,
  );
```

ただし通常は`LoginDocument`からVariables Typeが推論されるため、毎回明示的に書く必要はない。

---

#### Query Response TypeをUI Stateに利用する

Issue型をFrontend側で手書きしない。

```typescript
import type {
  GetIssuesQuery,
} from "@/generated/graphql";


type Issue =
  GetIssuesQuery["issues"][number];
```

これにより:

```typescript
const [
  issues,
  setIssues,
] = useState<Issue[]>([]);
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
import {
  GraphQLClient,
} from "graphql-request";


const endpoint =
  process.env
    .NEXT_PUBLIC_GRAPHQL_URL!;


export function getGraphQLClient() {
  const token =
    typeof window !== "undefined"
      ? localStorage.getItem(
          "accessToken",
        )
      : null;

  return new GraphQLClient(
    endpoint,
    {
      headers: token
        ? {
            Authorization:
              `Bearer ${token}`,
          }
        : {},
    },
  );
}
```

この段階ではServer State Cacheはまだ追加しない。
Phase 2でApollo Clientを導入し、GraphQL Cacheを本格的に学ぶ。

---
## Step 14.5 — Root Page

`frontend/src/app/page.tsx`:

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

`/login`と`/issues`へ移動できることを確認する。

---

## Step 14.6 — Login Page

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

import {
  FormEvent,
  useState,
} from "react";

import {
  useRouter,
} from "next/navigation";

import {
  LoginDocument,
} from "@/generated/graphql";

import {
  getGraphQLClient,
} from "@/lib/graphql-client";


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
      const client =
        getGraphQLClient();

      const data =
        await client.request(
          LoginDocument,
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

確認ポイント:

```text
1. Email / Passwordを入力できる
2. Login中はButtonがdisabledになる
3. Error時はMessageが表示される
4. Success時はaccessTokenが保存される
5. /issuesへ遷移する
```

---

## Step 14.7 — Issue CRUD Page

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
  CreateIssueDocument,
  DeleteIssueDocument,
  GetIssuesDocument,
  UpdateIssueDocument,
  type GetIssuesQuery,
} from "@/generated/graphql";

import {
  getGraphQLClient,
} from "@/lib/graphql-client";


type Issue =
  GetIssuesQuery["issues"][number];


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
          const client =
            getGraphQLClient();

          const data =
            await client.request(
              GetIssuesDocument,
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
        await getGraphQLClient().request(
          CreateIssueDocument,
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
        await getGraphQLClient().request(
          UpdateIssueDocument,
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
        await getGraphQLClient().request(
          DeleteIssueDocument,
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

このStepではUI Libraryを追加せず、Tailwindだけで最低限の画面を作る。

---

## Step 14.8 — BrowserでCRUD確認

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
    status,
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

## Step 14.9 — Network TabでGraphQL確認

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

## Step 14.10 — Authentication Failure確認

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

# Step 15 — Filter / Search / Cursor Pagination

Backend Queryへ以下を追加する。

```text
status
search
cursor
limit
```

Frontendにも簡単なFilter / Search UIを追加し、Network TabでGraphQL Variablesを確認する。

大量DataでのPagination / Infinite Scroll / VirtualizationはPhase 2で深掘りする。

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
