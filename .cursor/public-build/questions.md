# Unresolved Questions

Open items that need answers before or during implementation. Remove or resolve entries by moving answers to [decisions.md](./decisions.md).

---

## Identity & auth

- [ ] **Keycloak vs Supabase AUTH** — HLD lists both as external IdPs. Which is primary for MVP? Use one, support both, or start with Supabase for speed?
- [ ] **Internal Auth service** — When is the in-VPC Auth service needed vs gateway-only JWT validation?
- [ ] **Social login** — Google/Apple sign-in in MVP or post-launch?

---

## Product scope

- [ ] **iOS timeline** — Web-first is implied; when does native iOS development start relative to Episode 8?
- [ ] **Maps provider** — Google Maps, Mapbox, or Apple Maps for Episode 14?
- [ ] **AI provider** — OpenAI, Anthropic, Azure OpenAI, or self-hosted? Cost and latency implications for SQS-backed workers.
- [ ] **Review moderation** — How are user reviews validated before public display?

---

## Domain & data

- [ ] **Seed data strategy** — Where does initial city/location/category data come from (manual, OSM, third-party API)?
- [ ] **Geospatial queries** — PostGIS extension or application-level distance calculation for "nearby attractions"?
- [ ] **Itinerary versioning** — Can users fork or version an AI-generated itinerary?

---

## Infrastructure

- [ ] **ECS: Fargate vs EC2** — Default for Wayla Api tasks?
- [ ] **Single AZ vs Multi-AZ** — HLD shows AZ A / Subnet 1 only; when to expand for production?
- [ ] **Rate limit value** — HLD mentions 1% rate limit; is that literal or placeholder?
- [ ] **CDN choice** — AWS CloudFront vs "Front Door" naming in diagram — confirm actual service.

---

## Infrastructure & IaC

- [ ] **AWS region** — Primary region for Terraform state and all modules (e.g. `eu-west-1`)?
- [ ] **Terragrunt** — Plain Terraform per env vs Terragrunt for DRY — defer until post-MVP?
- [ ] **Migration runner** — One-off ECS task vs dedicated migration job in CD pipeline?
- [ ] **Frontend hosting** — S3 + CloudFront vs ECS for React static build?
- [ ] **Prod approval** — Single reviewer or solo `workflow_dispatch` for MVP?

---

## Public build & launch

- [ ] **Publishing cadence** — One episode per week? Tie to git tags or branches?
- [ ] **Repo visibility** — Single monorepo (frontend + backend + infra) or split repos?
- [ ] **MVP definition** — Shippable MVP at Episode 17 (HTTPS prod)?

---