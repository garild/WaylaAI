# CI/CD

GitHub Actions pipelines for Wayla — build, test, plan, deploy. Episode 16 in the public build series.

---

## Goals

- **Every merge is deployable** — tests pass before merge to `main`
- **Infra changes are reviewed** — `terraform plan` posted on PRs
- **App deploys are boring** — push image to ECR, roll ECS service
- **Public build transparency** — pipeline status badges in README, failed deploys become content

---

## Pipeline overview

```
                    ┌─────────────────────────────────────────┐
                    │              Pull Request               │
                    └─────────────────────────────────────────┘
                      │              │              │
          ┌───────────┘              │              └───────────┐
          ▼                          ▼                          ▼
   api-ci.yml                 frontend-ci.yml          terraform-plan.yml
   · restore/build/test        · lint/test/build          · fmt/validate/plan
   · docker build (no push)    · (no deploy)              · comment plan on PR
          │                          │                          │
          └──────────────────────────┼──────────────────────────┘
                                     ▼
                              merge to main
                                     │
          ┌──────────────────────────┼──────────────────────────┐
          ▼                          ▼                          ▼
   api-cd-staging.yml        frontend-cd-staging.yml    terraform-apply-staging.yml
   · test                     · build + S3/CloudFront      · apply (auto or approval)
   · docker push → ECR        · invalidate cache           │
   · ecs update-service       │                            │
   · smoke test               │                            │
          └──────────────────────────┴──────────────────────────┘
                                     │
                              tag v* (prod)
                                     ▼
                    api-cd-prod.yml + terraform apply to ECS
                    terraform-apply-prod.yml (manual approval)
```

---

## Workflow files (target)

```
Wayla/
└── .github/
    └── workflows/
        ├── api-ci.yml              # PR: .NET test + docker build
        ├── api-cd-staging.yml      # main: deploy API to staging ECS
        ├── api-cd-prod.yml         # tag v*: deploy API to prod (approval gate)
        ├── frontend-ci.yml         # PR: React lint + test + build
        ├── frontend-cd-staging.yml # main: deploy static site / SSR
        ├── terraform-plan.yml      # PR touching infra/: plan + PR comment
        ├── terraform-apply-staging.yml
        └── terraform-apply-prod.yml
```

---

## AWS auth from GitHub (OIDC — no long-lived keys)

```yaml
# Pattern used in all deploy workflows
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::ACCOUNT:role/github-wayla-deploy
      aws-region: eu-west-1
```

Terraform module `iam` creates `github-wayla-deploy` with trust policy for `token.actions.githubusercontent.com` scoped to repo `your-org/Wayla`.

---

## API pipeline — `api-ci.yml` (PR)

| Step | Action |
|------|--------|
| Trigger | PR to `main`, paths `src/**`, `Dockerfile`, `*.csproj` |
| Checkout | `actions/checkout@v4` |
| Setup | .NET 8 SDK |
| Test | `dotnet test` — unit + integration (Testcontainers optional later) |
| Build | `docker build -t wayla-api:${{ github.sha }} .` |
| Push | **No** — validate image builds only |

---

## API pipeline — `api-cd-staging.yml` (main)

| Step | Action |
|------|--------|
| Trigger | Push to `main` (API paths) |
| Test | Same as CI |
| ECR login | `aws ecr get-login-password` |
| Push | `wayla-api:${{ github.sha }}` → ECR |
| Deploy | `aws ecs update-service --cluster wayla-staging --service wayla-api --force-new-deployment` |
| Migrate | Optional job: run EF migrations via one-off ECS task |
| Smoke | `curl -f https://api-staging.wayla.../health` |
| Notify | Fail loudly — good Ep 16 B-roll when it breaks |

Image tag strategy: **`github.sha`** for traceability; alias `staging-latest` optional.

---

## Frontend pipeline — `frontend-ci.yml` / `frontend-cd-staging.yml`

| CI (PR) | CD (main) |
|---------|-----------|
| `npm ci` | Build production bundle |
| `npm run lint` | Sync to S3 bucket OR deploy to hosting |
| `npm test` | CloudFront invalidation `/*` |
| `npm run build` | Env: `VITE_API_URL` → staging API Gateway URL |

If using SSR later (Next.js), swap S3 deploy for container deploy — same pattern as API.

---

## Terraform pipeline — `terraform-plan.yml` (PR)

| Step | Action |
|------|--------|
| Trigger | PR, paths `infra/terraform/**` |
| Setup | `hashicorp/setup-terraform@v3` |
| Init | `terraform init` in `environments/staging` |
| Validate | `terraform fmt -check`, `terraform validate` |
| Plan | `terraform plan -no-color -out=plan.tfplan` |
| Comment | Post plan output to PR via `github-script` or `dflook/terraform-plan` |

Reviewers see infra diff before merge — core public-build moment.

---

## Terraform pipeline — `terraform-apply-staging.yml` (main)

| Step | Action |
|------|--------|
| Trigger | Merge to `main`, infra paths |
| Plan + Apply | `terraform apply -auto-approve` (staging only) |
| Prod | **Manual** `workflow_dispatch` + environment protection rule |

Use GitHub **Environments** `staging` and `production` with required reviewers on prod.

---

## Database migrations in CD

Controlled step — avoid racing app deploy:

1. Build + push image
2. Run **migration task** (ECS run-task, one-off Fargate) with new image
3. Wait for migration exit code 0
4. Roll ECS service to new image

Document rollback: keep previous task definition revision; migrations need backward-compatible strategy.

---

## Release strategy — DEV → Staging → Production

Three environments, one pipeline philosophy: **fast in DEV, automatic in Staging, deliberate in Production.**

```
  Feature branch                    main                         tag v*
       │                              │                             │
       ▼                              ▼                             ▼
  ┌─────────┐                  ┌─────────────┐              ┌─────────────┐
  │   DEV   │  ── merge ──▶   │   STAGING   │  ── promote ─▶│    PROD     │
  │  (lab)  │                  │  (pre-prod) │              │  (live)     │
  └─────────┘                  └─────────────┘              └─────────────┘
  CI + optional deploy         auto CD on merge             gated CD on tag
  cheap / single-AZ            smoke tests                  blue-green roll
```

### Environment matrix

| | **DEV** | **Staging** | **Production** |
|---|---------|-------------|----------------|
| **Purpose** | Experimentation, infra spikes, broken builds OK | Pre-prod validation, demo URLs, CI/CD proof | Public users, real data |
| **Trigger** | Push to `develop` or `workflow_dispatch` | Merge to `main` | Git tag `v*.*.*` (semver) |
| **Approval** | None | None (auto) | GitHub Environment `production` — required reviewer(s) |
| **AWS sizing** | Minimal Fargate, single-AZ RDS `db.t4g.micro` | Staging-sized, mirrors prod topology | Prod sizing, Multi-AZ when ready |
| **Image tag** | `dev-${{ github.sha }}` | `${{ github.sha }}` + `staging-latest` | `v1.2.3` (tag name) + `prod-latest` |
| **Terraform** | Local `apply` or optional `terraform-apply-dev.yml` | Auto `apply` on merge | Manual `workflow_dispatch` + approval after tag |
| **Frontend URL** | `dev.wayla.app` (or ephemeral) | `staging.wayla.app` | `wayla.app` |
| **API URL** | `api-dev.wayla.app` | `api-staging.wayla.app` | `api.wayla.app` |
| **DB migrations** | Dev can reset; destructive OK | Forward-only, backward-compatible | Forward-only + rollback plan documented |
| **Release cadence** | Continuous — every push | Continuous — every merge to `main` | On demand — tag when staging is green |

### Branch & trigger map

| Branch / event | CI | CD target | Terraform |
|----------------|-----|-----------|-----------|
| Feature PR → `develop` or `main` | `api-ci`, `frontend-ci` | — | `plan` on PR if `infra/**` touched |
| Push to `develop` | Full test suite | **DEV** (optional auto) | `apply` dev (optional workflow) |
| Merge to `main` | Full test suite | **Staging** | `apply` staging |
| Tag `v1.2.3` on `main` | Full test suite | **Production** | `apply` prod (approval gate) |

### DEV environment

DEV is the sandbox — cheap, fast, and allowed to break.

| Step | Action |
|------|--------|
| Trigger | Push to `develop`, or manual `workflow_dispatch` from any branch |
| CI | Same as PR — `dotnet test`, `npm test`, docker build (no push on PR) |
| Deploy | `api-cd-dev.yml` — push to ECR, roll ECS `wayla-dev` cluster |
| Infra | Prefer local `terraform apply` in `environments/dev/` during Ep 15; optional GitHub workflow later |
| Secrets | Separate SSM prefix `/wayla/dev/*` — never share prod credentials |
| Teardown | `terraform destroy` acceptable; RDS can be recreated |

**When to use DEV vs local:** Local Docker Compose for app iteration; DEV AWS when testing ECS task defs, IAM, API Gateway, or migration scripts against real AWS services.

### Staging environment

Staging proves the full loop before anything touches prod.

| Step | Action |
|------|--------|
| Trigger | Merge to `main` (API / frontend / infra paths) |
| Gate | All CI jobs green on the merge commit |
| Deploy order | 1) Build + push image → 2) Run migration task → 3) ECS rolling update → 4) Smoke test |
| Rollback | Revert commit on `main` or redeploy previous task definition revision |
| Soak time | Optional: wait 15–30 min green before tagging prod (manual discipline for MVP) |

### Production environment

Production releases are **tag-driven**, not branch-driven — every deploy is named and traceable.

| Step | Action |
|------|--------|
| Trigger | Push annotated tag `v1.2.3` to `main` |
| Pre-check | Staging smoke tests passed; changelog reviewed |
| Approval | GitHub Environment `production` — blocks until reviewer approves |
| Strategy | **Rolling update** for MVP; blue-green or canary when traffic warrants (Ep 21) |
| Deploy order | Same as staging: image → migrate → roll → smoke → health alarm check |
| Rollback | Deploy previous tag (`v1.2.2`) or ECS task def revision N-1; migrations must be backward-compatible |
| Announce | Tag doubles as release note anchor — link in YouTube Ep 17 "first prod deploy" |

### Semver tagging convention

```
v{MAJOR}.{MINOR}.{PATCH}

v1.0.0  — first public MVP (Episode 17)
v1.1.0  — new feature, backward-compatible
v1.1.1  — hotfix on prod
```

Create tags from `main` only:

```bash
git checkout main && git pull
git tag -a v1.0.0 -m "First production deploy — Wayla MVP"
git push origin v1.0.0
```

### GitHub Environments setup

| Environment | Protection rules |
|-------------|------------------|
| `dev` | No reviewers; optional branch rule: `develop` only |
| `staging` | No reviewers; deploys from `main` |
| `production` | Required reviewers (1+); restrict to `main`; optional wait timer |

Workflow snippet — prod approval gate:

```yaml
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    environment: production   # blocks until approved
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT:role/github-wayla-deploy-prod
          aws-region: eu-west-1
      # ... build, push, migrate, deploy ...
```

Use a **separate IAM role** for prod (`github-wayla-deploy-prod`) with least privilege vs staging.

### Additional workflow files (DEV + prod)

```
Wayla/
└── .github/
    └── workflows/
        ├── api-ci.yml
        ├── api-cd-dev.yml              # develop → DEV ECS
        ├── api-cd-staging.yml          # main → staging ECS
        ├── api-cd-prod.yml             # tag v* → prod ECS (approval gate)
        ├── frontend-ci.yml
        ├── frontend-cd-dev.yml
        ├── frontend-cd-staging.yml
        ├── frontend-cd-prod.yml
        ├── terraform-plan.yml
        ├── terraform-apply-dev.yml     # optional
        ├── terraform-apply-staging.yml
        └── terraform-apply-prod.yml
```

### Release checklist (prod)

- [ ] Staging deploy green; smoke tests pass
- [ ] DB migration tested on staging with production-like data volume (when applicable)
- [ ] Breaking API changes versioned or feature-flagged
- [ ] Rollback steps documented in PR / release notes
- [ ] Tag created on `main`; prod workflow approved
- [ ] Post-deploy: health check, CloudWatch 5xx alarm quiet for 10 min

---

## Observability hooks (post-deploy)

- CloudWatch log group per ECS service — linked in workflow summary
- Optional: post deploy URL to PR comment
- Episode 19: alarms on 5xx rate, ECS CPU, RDS connections

---

## README badges (public build)

```markdown
![API CI](https://github.com/org/Wayla/actions/workflows/api-ci.yml/badge.svg)
![Deploy Staging](https://github.com/org/Wayla/actions/workflows/api-cd-staging.yml/badge.svg)
```

---

## Episode 16 — social angles (draft)

**YouTube:** CI/CD for My AI Travel App — Git Push to AWS ECS

**Shorts:**
1. "Every push to main deploys to staging — automatically"
2. "Terraform plan on every infra PR" (show GitHub comment)
3. "This failed deploy taught me more than the success"
4. "OIDC > AWS access keys — here's why"
5. "DEV → Staging → Prod — my release strategy in 60 seconds" (visual: [ci-cd-shorts-9x16.png](../public-build/social/ci-cd-shorts-9x16.png))

---

## Dependencies

| Requires | Episode |
|----------|---------|
| ECR + ECS exist | 15 (Terraform) |
| GitHub repo | 4+ (code exists) |
| OIDC IAM role | 15 `iam` module |

Episode 17 wires HTTPS + custom domain after CD proves the loop works on staging.
