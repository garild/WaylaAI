# Future Ideas

Ideas and features planned for  MVP. Not yet committed — move to [decisions.md](./decisions.md) when locked in.

---

## From the episode roadmap

### AI & personalization

- Generate personalized itineraries (Episode 9)
- AI recommendations and hidden gems (Episode 9)
- Conversational travel assistant with multi-turn chat (Episode 10)
- Save recommendations from AI conversations (Episode 10)
- Complete AI trip planner: budget estimation, transportation, daily schedule (Episode 18)

### Search & discovery

- Search cities, attractions, restaurants, and categories (Episode 13)
- Trending places and popular cities on home page (Episode 8)

### Maps & location

- Display locations on a map (Episode 14)
- Nearby attractions (Episode 14)
- Walking routes (Episode 14)

### Performance & scale

- Cache popular destinations and search results in Redis (Episode 11)
- Queue AI requests with background workers via SQS (Episode 12)
- Terraform all AWS resources — VPC, ECS, RDS, Redis, SQS, ALB, API GW (Episode 15)
- GitHub Actions CI/CD — test, ECR push, ECS deploy (Episode 16)
- CloudFront + HTTPS production deploy (Episode 17)
- Auto Scaling, load balancer tuning, DB indexing, Redis optimization (Episode 21)

### Product & growth

- First real user feedback loop (Episode 16)
- Iterate on UX and performance from real usage (Episode 17)
- Public MVP launch with lessons learned (Episode 22)

---

## Beyond plan

- **Reviews** — user-generated reviews on locations (entity exists in domain model)
- **Social sharing** — share itineraries or saved places
- **Collaborative trips** — multiple users planning one trip
- **Offline mode (mobile)** — saved trips and maps without connectivity
- **Multi-language support** — i18n for global travel audience
- **Partner integrations** — booking APIs, transit data, weather
- **Observability stack** — CloudWatch, tracing (X-Ray/OpenTelemetry) per HLD suggested checkpoints
- **Multi-AZ RDS** — production hardening when moving beyond single-AZ dev setup

---