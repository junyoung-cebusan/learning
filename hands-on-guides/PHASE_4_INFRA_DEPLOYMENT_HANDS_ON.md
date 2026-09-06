# Phase 4 — Infrastructure & Deployment Hands-on Guide

## Goal

Issue TrackerをLocalからStaging/Production-like環境へDeployし、  
CI/CDとRollbackまで実際に体験する。

---

# Step 1 — Backend Production Dockerfile

```dockerfile
FROM python:3.12-slim AS runtime

WORKDIR /app

COPY --from=ghcr.io/astral-sh/uv:latest \
  /uv /usr/local/bin/uv

COPY pyproject.toml uv.lock ./

RUN uv sync \
  --frozen \
  --no-dev

COPY src ./src

ENV PYTHONPATH=/app/src

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

Build:

```bash
docker build \
  -t issue-backend \
  ./backend
```

---

# Step 2 — Frontend Production Dockerfile

```dockerfile
FROM node:22-alpine AS build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

RUN npm run build


FROM node:22-alpine AS runtime

WORKDIR /app

COPY --from=build \
  /app ./

CMD [
  "npm",
  "start"
]
```

---

# Step 3 — Docker Compose

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"

  backend:
    build: ./backend
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: password

  redis:
    image: redis:7
```

Run:

```bash
docker compose up --build
```

---

# Step 4 — Alembic

Install:

```bash
uv add alembic
```

Init:

```bash
uv run alembic init alembic
```

Migration:

```bash
uv run alembic revision \
  --autogenerate \
  -m "add issue version"
```

Apply:

```bash
uv run alembic upgrade head
```

Test:

```text
empty DB
→ upgrade head

existing DB
→ upgrade head

rollback
→ downgrade -1
```

---

# Step 5 — Environment Separation

Backend:

```text
DATABASE_URL
REDIS_URL
JWT_SECRET
```

Frontend:

```text
NEXT_PUBLIC_GRAPHQL_URL
```

`.env`をGitへCommitしない。

---

# Step 6 — AWS Architecture

Target:

```text
Internet
   ↓
ALB
   ↓
ECS Backend
   ↓
RDS PostgreSQL
   ↓
ElastiCache Redis
```

FrontendはNext.jsのHosting方式を決める。

---

# Step 7 — ECS

Container ImageをRegistryへPush。

```text
docker build
→ push
→ ECS Task Definition
→ ECS Service
```

Health Endpoint:

```python
@app.get("/health")
async def health():
    return {
        "status": "ok"
    }
```

---

# Step 8 — RDS

Local PostgreSQLからRDSへ接続先を切り替える。

確認:

```text
Public access off
Security Group
Connection count
Backup
```

---

# Step 9 — ElastiCache

`REDIS_URL`をAWS Redisへ変更する。

Cacheが落ちたときApplicationが完全停止しない設計も考える。

---

# Step 10 — Terraform

Example:

```hcl
resource "aws_ecs_cluster" "app" {
  name = "issue-tracker"
}
```

StateをGitへ置かない。

最終的に:

```text
terraform plan
terraform apply
```

で環境を再現する。

---

# Step 11 — GitHub Actions CI

`.github/workflows/ci.yml`

```yaml
name: CI

on:
  pull_request:

jobs:
  backend-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Test
        run: |
          cd backend
          uv sync
          uv run pytest -q
```

Frontend:

```yaml
  frontend-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22

      - run: |
          cd frontend
          npm ci
          npm run test
          npm run build
```

---

# Step 12 — CD

Flow:

```text
main merge
→ Test
→ Build Image
→ Push Registry
→ Migration
→ ECS Deploy
→ Smoke Test
```

---

# Step 13 — Staging Playwright

CI에서Staging URLへ接続する。

```typescript
export default defineConfig({
  use: {
    baseURL:
      process.env.E2E_BASE_URL,
  },
});
```

```bash
E2E_BASE_URL=https://staging.example.com \
npx playwright test
```

---

# Step 14 — Rollback Drill

わざとError VersionをDeployする。

確認:

```text
Health Check失敗
→ Deploy失敗を検知
→ Previous Task DefinitionへRollback
```

DB MigrationとのCompatibilityも確認する。

---

# Step 15 — Smoke Test

Deploy後:

```bash
curl https://api.example.com/health
```

GraphQL:

```graphql
query {
  __typename
}
```

最低限Serviceが生きていることを確認する。

---

# Phase 4 Completion

```text
[ ] Docker build
[ ] Compose
[ ] Alembic
[ ] AWS ECS
[ ] RDS
[ ] Redis
[ ] Terraform
[ ] CI
[ ] CD
[ ] Staging E2E
[ ] Rollback
```
