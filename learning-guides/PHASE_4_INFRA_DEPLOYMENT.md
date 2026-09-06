# Phase 4 — Infrastructure & Deployment

## Goal

Issue TrackerをLocal環境から、  
**再現可能で自動化されたProduction-like環境**へDeployする。

Phase 1のDocker / AWSは入門として扱い、ここで本格的に学ぶ。

---

## Step 1 — Production Dockerfile

Backend / Frontendそれぞれ作成する。

- Image Layer
- Build Cache
- Multi-stage Build
- Non-root User
- `.dockerignore`

---

## Step 2 — Docker Compose

```text
Next.js
FastAPI
PostgreSQL
Redis
Worker
Kafka
```

をLocalでまとめて起動する。

---

## Step 3 — Configuration / Secrets

```text
local
test
staging
production
```

- Environment Variable
- Secret
- Public Configuration
- `NEXT_PUBLIC_*`

---

## Step 4 — Alembic Migration

- Revision
- Upgrade
- Downgrade
- Backward Compatibility
- Migration Order

### Testing

- Empty DBからMigration
- Existing DB Upgrade
- Rollback

---

## Step 5 — AWS Network

```text
VPC
├─ Public Subnet
│  └─ ALB
└─ Private Subnet
   ├─ ECS
   ├─ RDS
   └─ ElastiCache
```

- Subnet
- Route
- Security Group
- Public / Private Boundary

---

## Step 6 — ECS

- Task Definition
- Service
- Desired Count
- Health Check
- Rolling Deployment

Backend / WorkerをDeployする。

---

## Step 7 — RDS PostgreSQL

- Managed DB
- Backup
- Connection Limit
- Multi-AZ
- Parameter / Monitoring

---

## Step 8 — ElastiCache Redis

Phase 2で使ったRedisをManaged Serviceへ移行する。

---

## Step 9 — Frontend Production

- Production Build
- Bundle
- Static Asset
- Image / Font Optimization
- CDN
- Cache Header

---

## Step 10 — S3 / CloudFront

- Object Storage
- CDN
- Cache Invalidation
- Signed URL

---

## Step 11 — Terraform

- Provider
- Resource
- Variable
- Output
- State
- Module

目標:

```text
同じInfrastructureをCodeから再現できる
```

---

## Step 12 — GitHub Actions CI

Pull Request:

```text
Frontend Lint / Unit Test
Backend Unit Test
GraphQL Integration Test
Build
```

---

## Step 13 — CD

```text
Build Image
→ Registry
→ Migration
→ Deploy
→ Smoke Test
```

---

## Step 14 — Playwright on Staging

```text
Login
→ Create
→ Update
→ Delete
→ Logout
```

をStagingで実行する。

---

## Step 15 — Deployment / Rollback

意図的に問題のあるVersionをDeployし、Rollbackを実施する。

- Rolling Deployment
- Rollback
- Migration Compatibility
- Smoke Test

---

## Phase 4 Completion Criteria

- Containerを使う理由
- Local / Production Configurationの分離方法
- Migration FailureからのRecovery
- ECS / RDS / RedisのNetwork Boundary
- CIとCDの役割の違い
- Stagingで何を検証するべきか
