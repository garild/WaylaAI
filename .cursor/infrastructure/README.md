# Infrastructure

Terraform and CI/CD for Wayla — planned layout, deploy flow, and how it maps to the HLD.

| Doc | Purpose |
|-----|---------|
| [terraform.md](./terraform.md) | AWS IaC — modules, environments, state, HLD mapping |
| [cicd.md](./cicd.md) | GitHub Actions — build, test, plan, deploy; DEV → Staging → Prod release strategy |

**Series episodes:** [Ep 15 — Terraform](../public-build/roadmap.md#episode-15--terraform-aws-infrastructure-as-code) · [Ep 16 — CI/CD](../public-build/roadmap.md#episode-16--cicd-from-git-push-to-ecs) · [Ep 17 — Production deploy](../public-build/roadmap.md#episode-17--production-deploy--https)

**Code repo target:** `Wayla/infra/` (Terraform) + `Wayla/.github/workflows/` (CI/CD) — created during Episodes 15–16.

---

## Deploy flow (target state)

```
Developer push / merge
        ↓
GitHub Actions — CI
  · dotnet test / npm test
  · docker build
  · terraform fmt + validate + plan (on infra changes)
        ↓
GitHub Actions — CD (main → staging, tag → prod)
  · push image → ECR
  · terraform apply (infra) OR ecs update-service (app)
  · run EF migrations (controlled step)
  · smoke test / health check
        ↓
Live: CDN → ALB → API GW → ECS → RDS / Redis / SQS
```

---

## HLD → Terraform module map

| HLD component | Terraform module | Episode |
|---------------|------------------|---------|
| VPC, subnets, security groups | `modules/network` | 15 |
| ECR | `modules/ecr` | 15 |
| ECS (Wayla Api) | `modules/ecs-service` | 15 |
| RDS PostgreSQL (Wayla Db) | `modules/rds` | 15 |
| ElastiCache Redis | `modules/elasticache` | 15 |
| SQS + DLQ | `modules/sqs` | 15 |
| ALB | `modules/alb` | 15 |
| API Gateway + VPC link | `modules/api-gateway` | 15 |
| CloudFront (CDN) | `modules/cloudfront` | 17 |
| IAM roles (ECS task, CI) | `modules/iam` | 15–16 |
| Secrets / SSM params | `modules/secrets` | 15 |

Auth (Keycloak / Supabase) stays external for MVP — JWT validation at API Gateway only.

---

## Environment strategy

| Environment | Branch / trigger | Purpose |
|-------------|------------------|---------|
| **dev** | Push to `develop` or `workflow_dispatch` | Experimentation, cheap sizing, CI/CD sandbox |
| **staging** | Merge to `main` | Pre-prod, automatic CD validation |
| **prod** | Git tag `v*.*.*` + approval gate | Public users, controlled releases |

State: **one S3 bucket + DynamoDB lock table per org**, keys `wayla/<env>/terraform.tfstate`.

---

## Progress

| Item | Status |
|------|--------|
| Terraform layout documented | Done |
| CI/CD pipelines documented | Done |
| `Wayla/infra/` in code repo | Not started |
| GitHub Actions workflows | Not started |
| AWS bootstrap (state bucket) | Not started |

Update this table as Episodes 15–17 ship.
