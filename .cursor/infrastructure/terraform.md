# Terraform

AWS infrastructure as code for WaylaAI. Episode 15 in the public build series.

---

## Why Terraform

- **Reproducible** — dev, staging, and prod from the same modules
- **Public build friendly** — every infra change is a PR with `terraform plan` output
- **HLD-aligned** — one module per diagram box
- **No click-ops** — VPC, ECS, RDS, Redis, SQS, ALB, API Gateway versioned in git

---

## Repository layout (target)

Lives in the [Wayla](../Wayla/) code repo when implemented:

```
Wayla/
└── infra/
    └── terraform/
        ├── README.md
        ├── versions.tf          # Terraform + provider pins
        ├── backend.tf           # S3 remote state (per env)
        ├── modules/
        │   ├── network/         # VPC 10.0.0.0/16, subnets, NAT, SGs
        │   ├── ecr/             # Private container registry
        │   ├── ecs-cluster/     # ECS cluster + capacity (Fargate)
        │   ├── ecs-service/     # Wayla Api task def + service + autoscaling
        │   ├── rds/             # PostgreSQL (Wayla Db)
        │   ├── elasticache/     # Redis
        │   ├── sqs/             # Queues + DLQ
        │   ├── alb/             # Application Load Balancer
        │   ├── api-gateway/     # REST/HTTP API, JWT authorizer, VPC link
        │   ├── cloudfront/      # CDN (Episode 17)
        │   ├── iam/             # ECS task roles, execution roles
        │   └── secrets/         # SSM Parameter Store / Secrets Manager refs
        └── environments/
            ├── dev/
            │   ├── main.tf
            │   ├── variables.tf
            │   ├── terraform.tfvars
            │   └── outputs.tf
            ├── staging/
            └── prod/
```

Each environment composes modules:

```hcl
# environments/staging/main.tf (conceptual)
module "network" {
  source = "../../modules/network"
  vpc_cidr = "10.0.0.0/16"
  azs      = ["eu-west-1a"] # expand Multi-AZ in prod
}

module "rds" {
  source     = "../../modules/rds"
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.private_subnet_ids
  # ...
}

module "ecs_service" {
  source         = "../../modules/ecs-service"
  cluster_id     = module.ecs_cluster.id
  image          = var.api_image_uri
  target_group_arn = module.alb.target_group_arn
  # ...
}
```

---

## Remote state bootstrap (one-time, manual or tiny bootstrap stack)

Create before first `terraform apply`:

| Resource | Purpose |
|----------|---------|
| S3 bucket | `wayla-terraform-state` — versioning ON, encryption ON |
| DynamoDB table | `wayla-terraform-locks` — state locking |
| IAM user/role | CI OIDC role for GitHub Actions (preferred over long-lived keys) |

Backend block per environment:

```hcl
terraform {
  backend "s3" {
    bucket         = "wayla-terraform-state"
    key            = "wayla/staging/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "wayla-terraform-locks"
    encrypt        = true
  }
}
```

---

## Module rollout order

Apply in dependency order

| Step | Module | Delivers |
|------|--------|----------|
| 1 | `network` | VPC `10.0.0.0/16`, public + private subnets, NAT, security groups |
| 2 | `ecr` | Private registry for Wayla Api image |
| 3 | `rds` | PostgreSQL — Wayla Db |
| 4 | `elasticache` | Redis subnet group + cluster |
| 5 | `sqs` | Main queue + DLQ |
| 6 | `ecs-cluster` | Fargate cluster |
| 7 | `alb` | Load balancer + target group + HTTPS listener (cert in Ep 17) |
| 8 | `ecs-service` | Task definition, service, logs to CloudWatch |
| 9 | `api-gateway` | Routes, throttling, JWT authorizer, VPC link to ALB/ECS |
| 10 | `iam` | Least-privilege task roles (SQS, SSM, RDS connect) |

Episode 17 adds `cloudfront` → ALB origin, custom domain, ACM cert.

---

## Variables & secrets

| Type | Storage | Examples |
|------|---------|----------|
| Non-secret config | `terraform.tfvars` per env | `instance_class`, `desired_count` |
| Secrets | AWS Secrets Manager / SSM | DB password, JWT issuer URLs |
| CI-injected | GitHub Actions secrets | `AWS_ROLE_ARN`, no static keys |

Never commit `.tfvars` with secrets. Use `TF_VAR_` env vars in CI for context-specific `terraform.tfvars` in private vault.

---

## HLD alignment checklist

Track against [Wayla README infrastructure checkpoints](../Wayla/README.md):

| Checkpoint | Terraform module |
|------------|------------------|
| Setup VPC with Security Group | `network` |
| Create Private ECR | `ecr` |
| Deploy Wayla Db (RDS PostgreSQL) | `rds` |
| Deploy Redis (ElastiCache) | `elasticache` |
| Configure SQS queues | `sqs` |
| Setup Load Balancer | `alb` |
| AWS Configure API GW | `api-gateway` |
| Deploy WebApi on ECS | `ecs-service` |
| Wire CDN to ALB | `cloudfront` (Ep 17) |

---

## Local dev workflow

```bash
cd infra/terraform/environments/dev
terraform init
terraform plan -out=plan.tfplan
terraform apply plan.tfplan
```

Pre-commit / CI: `terraform fmt -check`, `tflint`, `terraform validate`.

Optional: [Terragrunt](https://terragrunt.gruntwork.io/) later if env duplication grows — not needed for MVP.

---

## Episode 15 — social angles (draft)

**YouTube:** I Terraform'd My Entire AWS Stack for an AI Travel App

**Shorts:**
1. "Stop clicking in the AWS console — use Terraform"
2. "VPC → ECS → RDS in one `terraform apply`"
3. "This is how infra PRs should look" (plan output on screen)
4. "Fargate vs EC2 — why I picked Fargate"

---

## Open questions

See [questions.md](../public-build/questions.md#infrastructure--iac) — resolve into [decisions.md](../public-build/decisions.md) during Episode 15.
