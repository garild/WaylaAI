# Roadmap

Full episode plan for **Building Wayla — An AI Travel Platform From Scratch (Public Build)**.

Status key: `planned` | `ready to film` | `in progress` | `published` | `done`

---

## Content pack (every episode)

Each episode ships:

| # | Deliverable | Platforms |
|---|-------------|-----------|
| 1 | Long-form video | YouTube |
| 2 | 3–5 short clips | TikTok · Reels · Shorts |
| 3 | Carousel or feed post | Instagram |
| 4 | Production brief | [episodes/](./episodes/) |
| 5 | Dialog + titles (copy on set) | [content/](./content/) |

Templates: [episodes/_template.md](./episodes/_template.md) · Playbook: [social/distribution.md](./social/distribution.md) · **Scripts: [content/dialogs-all-episodes.md](./content/dialogs-all-episodes.md)**

---

## Episode 1 — Why I'm Building Wayla

**Status:** in progress (filming)  
**Brief:** [episodes/ep01-why-im-building-wayla.md](./episodes/ep01-why-im-building-wayla.md)

**Build topics:**
- The problem with travel planning today
- Vision for an AI travel companion
- Introduce the architecture

**YouTube title:** Why I'm Building an AI Travel App in Public (Ep 1/22)

**Short clip angles:**
1. "Travel planning shouldn't take 47 browser tabs"
2. "Stop building in secret — $0 marketing, 22 episodes"
3. Architecture diagram tease (React · .NET · AWS · Terraform)
4. "22 episodes. One AI travel platform."
5. "ChatGPT isn't a travel app"

---

## Episode 2 — Designing the System Architecture

**Status:** in progress (HLD drafted)

- React frontend
- .NET backend
- API Gateway
- JWT Authentication
- PostgreSQL
- Redis
- Amazon ECS
- Amazon SQS
- CDN & Load Balancer
- Terraform (IaC) + CI/CD preview — see [infrastructure/](../infrastructure/)

*HLD drafted — see [README.md](../Wayla/README.md)*

**YouTube title (draft):** I Designed the Full AWS Architecture for My AI Travel App

**Short clip angles (draft):**
1. "Every box in this diagram — explained in 60 seconds"
2. "Why JWT at the API Gateway, not in my app"
3. "PostgreSQL + Redis + SQS — why not one database?"
4. Hot take: "ECS over Lambda for this project"

---

## Episode 3 — Planning the Domain

**Status:** planned

**Entities:**

- User
- Country
- City
- Location
- Category
- Trip
- Itinerary
- Saved Place
- Review
- AI Recommendation
- Conversation
- Message

---

## Episode 4 — Building the Backend

**Status:** planned

- Create the .NET solution
- Clean Architecture
- Domain / Application / Infrastructure / API

---

## Episode 5 — The First API

**Status:** planned

- `GET /cities`
- `GET /locations`
- `GET /categories`

---

## Episode 6 — PostgreSQL + Entity Framework

**Status:** planned

- Initial migration
- Cities
- Locations
- Categories

---

## Episode 7 — Authentication

**Status:** planned

- JWT
- Registration
- Login
- User profiles
- Saved places

---

## Episode 8 — The First React Screen

**Status:** planned

- Home page
- Search
- Popular cities
- Trending places

---

## Episode 9 — Integrating AI

**Status:** planned

- Generate personalized itineraries
- AI recommendations
- Hidden gems

---

## Episode 10 — AI Conversations

**Status:** planned

- Conversational travel assistant
- Multi-turn chat
- Save recommendations

---

## Episode 11 — Redis

**Status:** planned

- Cache popular destinations
- Cache search results
- Improve response times

---

## Episode 12 — Amazon SQS

**Status:** planned

- Queue AI requests
- Background workers
- Asynchronous processing

---

## Episode 13 — Search

**Status:** planned

- Search cities
- Attractions
- Restaurants
- Categories

---

## Episode 14 — Maps

**Status:** planned

- Display locations
- Nearby attractions
- Walking routes

---

## Episode 15 — Terraform: AWS Infrastructure as Code

**Status:** planned  
**Docs:** [infrastructure/terraform.md](../infrastructure/terraform.md)

- Bootstrap remote state (S3 + DynamoDB lock)
- Terraform module layout (`network`, `ecr`, `rds`, `elasticache`, `sqs`, `ecs`, `alb`, `api-gateway`)
- `terraform plan` / `apply` for staging
- Map every HLD box to a module

**YouTube title (draft):** I Terraform'd My Entire AWS Stack for an AI Travel App

**Short clip angles (draft):**
1. "Stop clicking in the AWS console — use Terraform"
2. "VPC → ECS → RDS in one apply"
3. "This is how infra PRs should look"

---

## Episode 16 — CI/CD: From Git Push to ECS

**Status:** planned  
**Docs:** [infrastructure/cicd.md](../infrastructure/cicd.md)

- GitHub Actions — API CI (test + docker build)
- GitHub OIDC → AWS (no static keys)
- Push image to ECR, deploy to ECS staging
- Terraform plan on PR, apply on merge
- Frontend CI/CD pipeline

**YouTube title (draft):** CI/CD for My AI Travel App — Git Push to AWS ECS

**Short clip angles (draft):**
1. "Every merge to main deploys to staging"
2. "Terraform plan posted on every infra PR"
3. "OIDC beats AWS access keys"

---

## Episode 17 — Production Deploy & HTTPS

**Status:** planned

- CloudFront CDN → ALB
- ACM certificate + custom domain
- Production ECS deploy (tag-triggered CD)
- Smoke tests + health checks
- First public URL live

**YouTube title (draft):** Deploying Wayla to Production on AWS

---

## Episode 18 — First Real User

**Status:** planned

- Collect feedback
- Discover bugs
- Improve UX

---

## Episode 19 — Fixing User Feedback

**Status:** planned

- Iterate based on real users
- UI improvements
- Performance improvements

---

## Episode 20 — AI Trip Planner

**Status:** planned

- Complete itinerary generation
- Budget estimation
- Transportation
- Daily schedule

---

## Episode 21 — Scaling Infrastructure

**Status:** planned

- Auto Scaling
- Load Balancer
- Database indexing
- Redis optimization

---

## Episode 22 — Launching Wayla

**Status:** planned

- Public MVP
- Lessons learned
- Roadmap
- First users and future plans

---

## Milestone summary

| Milestone | Episodes | Target outcome |
|-----------|----------|----------------|
| Foundation | 1–3 | Vision, architecture, domain model |
| Backend core | 4–7 | .NET API, DB, auth |
| Frontend + AI | 8–12 | React UI, AI features, cache, queues |
| Discovery | 13–14 | Search and maps |
| Infra & deploy | 15–17 | Terraform, CI/CD, HTTPS production |
| Ship & iterate | 18–19 | First users, feedback fixes |
| Scale & launch | 20–22 | Trip planner, scale, public MVP |
