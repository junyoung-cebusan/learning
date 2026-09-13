# Phase 4 — Infrastructure & Deployment Hands-on Guide

## Goal

Phase 1で体験した`EC2 + RDS + 手動Docker Deploy`を出発点に、Issue TrackerをStaging/Production-like構成へ発展させる。  
このPhaseではEC2への手動Deployを繰り返さず、ECR / ALB / ECS Fargate / Service Discovery / CI/CD / Terraform / Rollbackを実際に体験する。

```text
Phase 1
EC2 + RDS + 手動Deploy

        ↓

Phase 4
ECR + ALB + ECS Fargate
    + RDS + ElastiCache
    + Service Discovery
    + CI/CD + Terraform
```

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
    image: postgres:17
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

Phase 1のEC2単体構成から、Container Service中心の構成へ変更する。

Target:

```text
Internet
   ↓
ALB
   ↓
ECS Fargate Service
   ├─ Backend Task A
   └─ Backend Task B
        ↓
RDS PostgreSQL
        ↓
ElastiCache Redis
```

Container Image:

```text
Docker Build
   ↓
ECR
   ↓
ECS Task Definition
```

FrontendはNext.jsのHosting方式を決める。
このPhaseのBackend学習では、まずALB → ECS → RDSの経路を完成させる。

---

# Step 7 — ECR / ECS

Phase 1ではEC2上でImageを直接Buildしたが、このPhaseではImageをECRへ保存し、ECS Task Definitionから参照する。

Flow:

```text
docker build
→ ECR login
→ docker push
→ ECS Task Definition
→ ECS Service
→ ALB Target Group
```

最低限確認すること:

```text
1. ECRにImage Tagが存在する
2. ECS TaskがRUNNINGになる
3. ALB Health Checkがhealthyになる
4. ALB経由で/graphqlへ接続できる
5. Taskを入れ替えてもALB Endpointは変わらない
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

# Step 9.5 — Service Discovery

Phase 3で分離したUser Profile Serviceを複数Instanceで動かす場合、Issue APIが固定IPへ依存しないようService Discoveryを使う。

```text
Issue API
→ Service Discovery / DNS
→ User Profile Service instances
```

確認すること:

```text
Service名で接続できる
Instance入れ替え後も接続先を追従できる
Health Checkで異常Instanceを外せる
```

Phase 6ではこの構成をgRPC Load Balancingの前提として利用する。

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

CIからStaging URLへ接続する。

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
[ ] ECR Push
[ ] ALB
[ ] ECS Fargate
[ ] RDS
[ ] Redis
[ ] Service Discovery
[ ] Terraform
[ ] CI
[ ] CD
[ ] Staging E2E
[ ] Rollback
```
