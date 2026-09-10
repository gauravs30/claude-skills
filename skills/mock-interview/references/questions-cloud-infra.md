# Question Bank — Cloud + Infra Design

AWS / GCP / Azure services, networking, HA / multi-region, cost, infrastructure system design.

Scenario template: see `questions-core-devops.md`. Cloud examples lean AWS but a strong candidate
can answer in GCP/Azure terms — accept equivalents.

## Contents
- CLD-1 — Design a deployment pipeline + hosting for a web app
- CLD-2 — Make a single-region service multi-region
- CLD-3 — The AWS bill doubled this month
- CLD-4 — VPC / networking design
- CLD-5 — Design a centralized logging platform
- CLD-6 — Choose compute: VMs vs containers vs serverless
- CLD-7 — Blast-radius and account/project structure

---

### CLD-1 — Design deployment + hosting for a web app
**Topics:** cloud architecture, CI/CD   **Levels:** mid, senior   **Format:** design
**Scenario (say this):** "A startup has a containerized web app — a React frontend and a Python
API talking to Postgres. They're on one EC2 box that someone SSHes into to deploy. Design what
their infrastructure and deployment should look like for the next two years."
**Candidate should clarify:** Traffic / growth expectations? Team size and cloud skill? Budget
sensitivity? Compliance needs? Existing cloud account?
**Strong answer covers:**
- Frontend: build in CI, serve static assets from object storage + CDN (S3 + CloudFront).
- API: containers on a managed platform — ECS Fargate / EKS / Cloud Run / App Runner — behind a
  load balancer, autoscaled, min 2 instances across AZs.
- Database: **managed** Postgres (RDS / Cloud SQL), Multi-AZ, automated backups + PITR, not
  self-run.
- Networking: VPC, public subnets for LB, private for app + DB, security groups least-privilege.
- IaC for all of it (Terraform / CDK). No SSH-to-deploy — CI builds an immutable image, deploys
  via the platform's API, rolling or blue-green.
- Secrets in a manager; app gets them via role, not env files.
- Observability from day one (metrics, logs, an uptime check, a couple of alerts).
- *(senior)* Right-sizing for a startup — don't hand them EKS + service mesh + multi-region on day
  one. Pick the lowest-ops option that fits (Fargate/Cloud Run). Migration path: strangle the
  EC2 box, cut over with DNS, keep it as rollback for a week. Cost estimate. Environments
  (prod/staging) and how they differ.
**Follow-ups / curveballs:**
- (mid) "Why managed Postgres instead of running it on a cheap instance?"
- (senior) "They get a traffic spike from a product launch — 20x for a day. Does your design hold?
  What's the weak point?" (→ DB connections, single-AZ anything, CDN cache hit rate, autoscaling
  warm-up time)
- (senior) "They're pre-revenue and the CFO wants the monthly cost under $500. What do you cut?"
**Red flags:** Self-managed database. Keeps SSH deploys. Single AZ. No IaC. Over-engineers
(service mesh, multi-region, Kubernetes) for a 5-person startup. No migration plan from the
current box.
**Tier notes:** service — a clean managed-services architecture with IaC. product — right-sizing,
migration strategy, the spike analysis. MANG — push scale ("now it's 5M users"), quantify DB
connection limits, discuss the strangler migration in detail.

---

### CLD-2 — Make a single-region service multi-region
**Topics:** HA, multi-region, data   **Levels:** senior, staff   **Format:** design
**Scenario (say this):** "You run a service in us-east-1. Leadership wants multi-region for
resilience and for lower latency to European users. Walk me through how you'd approach it and what
makes it hard."
**Candidate should clarify:** Is the goal DR (survive a region loss) or active-active (serve from
both)? RTO/RPO? What's the datastore? How much data? Consistency requirements? Compliance (data
residency)?
**Strong answer covers:**
- **The hard part is state, not compute.** Stateless services replicate trivially; the database is
  the problem.
- Options along a spectrum:
  - **Active-passive / warm standby** — async replica in region 2, promote on failover. Simple;
    RPO > 0 (lose in-flight writes), RTO = promotion + DNS/traffic shift time.
  - **Active-active with a single write region** — reads local everywhere, writes routed to the
    primary region. Good latency for reads, write latency for the far region.
  - **Active-active multi-master** — conflict resolution, or a globally-distributed DB (Spanner,
    DynamoDB global tables, CockroachDB). Complexity and cost jump.
- Traffic routing: latency/geo DNS or anycast; health-check-based failover.
- Everything else that's regional: object storage replication, caches (cold in region 2),
  secrets, message queues, cron/singleton jobs (must not run twice).
- Failover **testing** — an untested failover doesn't work. Game days.
- *(staff)* Cost roughly doubles; is the resilience worth it vs. multi-AZ + good backups? Data
  residency / GDPR forcing EU data to stay in EU. Incremental rollout: start with reads, then
  DR-only, then active-active. What "region down" actually looks like (usually partial /
  brownout, not clean).
**Follow-ups / curveballs:**
- (senior) "Your DB is a single RDS Postgres with 4 TB. What are your realistic options?"
- (staff) "us-east-1 has a partial outage — some AZs, some services. Your automated failover fires
  and now you have split-brain. How do you design against that?"
**Red flags:** Treats it as "just deploy to two regions". Ignores the database entirely.
Active-active multi-master with no mention of conflicts. No failover testing. No cost discussion.
Doesn't ask DR-vs-active-active.
**Tier notes:** Senior+ question. product — active-passive with clear RTO/RPO, state challenges,
testing. MANG — the full spectrum, split-brain, global DB tradeoffs, quantified RTO/RPO,
incremental rollout.

---

### CLD-3 — The AWS bill doubled this month
**Topics:** cost, troubleshooting   **Levels:** mid, senior   **Format:** troubleshooting
**Scenario (say this):** "Finance flags that the AWS bill went from $30k to $62k month-over-month
with no announced launch. You're asked to find out why and bring it back down. How do you
approach it?"
**Candidate should clarify:** Any infra changes, new teams, new accounts? Is it one account or an
org? Do we have Cost Explorer / tags / CUR set up?
**Strong answer covers:**
- **Find it:** Cost Explorer grouped by service, then by usage type, then by account/tag, filtered
  to the month. Look for the delta, not the total. Common culprits: NAT gateway data processing,
  cross-AZ / cross-region traffic, S3 request or storage-class costs, EBS/snapshot sprawl,
  forgotten large instances or a runaway autoscaling group, data egress, a logging/observability
  pipeline, orphaned load balancers, dev environments left running.
- **Fix the spike:** stop the specific thing. Then structural: rightsizing, Savings Plans /
  Reserved capacity for steady-state, S3 lifecycle policies, gp3 over gp2, VPC endpoints to avoid
  NAT charges, spot for fault-tolerant work, autoscaling floors, killing idle resources.
- **Prevent recurrence:** mandatory cost-allocation tags, per-team budgets + alerts, anomaly
  detection, a cost dashboard teams actually see, cost in the architecture review.
- *(senior)* Unit economics — cost per request / per customer / per team, so "the bill went up"
  becomes "cost per order went up 12%, here's why". Showback/chargeback to give teams skin in the
  game.
**Follow-ups / curveballs:**
- (mid) "Cost Explorer says the increase is almost all 'EC2-Other'. What is that and how do you
  dig in?" (→ NAT, EBS, data transfer, ELB — need usage-type breakdown)
- (senior) "Every team says their spend is essential. How do you actually get the bill down?" (→
  unit cost + showback changes the conversation; find the 20% that's 80% of waste; RIs/SPs for
  the committed base)
**Red flags:** Starts turning things off before finding the cause. Doesn't know NAT / cross-AZ /
egress are common surprises. No tagging or prevention story. "Just buy Reserved Instances" as the
whole answer.
**Tier notes:** service — Cost Explorer drill-down + the usual suspects + lifecycle/rightsizing.
product — prevention via tags/budgets, Savings Plans strategy. MANG — unit economics,
showback/chargeback, cost as an architectural constraint.

---

### CLD-4 — VPC / networking design
**Topics:** networking   **Levels:** mid, senior   **Format:** design
**Scenario (say this):** "Design the network for a new production environment in AWS: a public web
tier, an internal app tier, a database tier, and it needs to reach the internet for package
updates and to call a partner API over the internet. Talk me through the VPC."
**Candidate should clarify:** Single or multi-account? Need to connect to on-prem or other VPCs?
Expected scale / IP space? Compliance?
**Strong answer covers:**
- One VPC, CIDR sized with room to grow (e.g. /16), spanning ≥2 (ideally 3) AZs.
- **Subnets per tier per AZ:** public (web/LB), private (app), private isolated (DB). Route tables
  per tier.
- Public subnets → Internet Gateway. Private subnets → **NAT gateway** (one per AZ for HA) for
  outbound only. DB subnets → no internet route at all.
- Security groups as the primary control, least-privilege, referencing each other (web SG → app
  SG → db SG), not CIDR ranges. NACLs as a coarse backstop.
- **VPC endpoints** for AWS services (S3, ECR, etc.) to avoid NAT cost and keep traffic private.
- Partner API over the internet: egress via NAT, ideally through a known set of NAT EIPs the
  partner can allowlist; consider a proxy for egress control/logging.
- *(senior)* Multi-account: this VPC in a workload account, centralized egress via a shared
  networking account + Transit Gateway, DNS with Route 53 Resolver, no overlapping CIDRs across
  the org. Flow logs for audit/debugging. IPv6 considerations.
**Follow-ups / curveballs:**
- (mid) "Why not just put the app tier in public subnets with a restrictive security group?"
- (senior) "NAT gateway data processing charges are $8k/month. How do you cut that without losing
  outbound access?" (→ VPC endpoints for AWS traffic, review what's actually egressing, maybe a
  fleet of NAT instances for very high volume, cache package mirrors)
- (senior) "You need to peer with 12 other VPCs. Peering or Transit Gateway?" (→ TGW; peering is
  non-transitive and O(n²))
**Red flags:** Everything in one subnet. DB with a public route. Security groups by IP instead of
by reference. Never heard of VPC endpoints or Transit Gateway. Single NAT gateway called "HA".
**Tier notes:** service — the three-tier subnet layout, SGs, NAT, IGW correct. product — VPC
endpoints, NAT cost awareness, flow logs. MANG — multi-account centralized egress, TGW vs peering
at scale, DNS architecture, CIDR management across the org.

---

### CLD-5 — Design a centralized logging platform
**Topics:** observability infra, system design   **Levels:** senior, staff   **Format:** design
**Scenario (say this):** "Design a centralized logging platform for a company with ~200 services
across 3 environments producing about 5 TB of logs per day. Engineers need to search recent logs
fast; compliance needs 1 year of retention."
**Candidate should clarify:** Structured logs already? Search latency expectations? Query patterns
(debugging vs analytics vs audit)? Budget? Self-host vs managed appetite? Multi-region?
**Strong answer covers:**
- **Pipeline:** agent on each host/pod (Fluent Bit / Vector / OTel Collector) → buffer/transport
  (Kafka / Kinesis) → processing (parse, enrich with metadata, redact PII, route) → sinks.
- **Tiered storage by access pattern:**
  - Hot (last 7–14 days): a search index (OpenSearch / Loki / a vendor) — fast queries, expensive.
  - Warm/cold (up to 1 year): object storage (S3) in a columnar format (Parquet), queried on
    demand (Athena / S3 Select) — cheap, slower. Lifecycle to Glacier for the tail if allowed.
- **Cost control:** sampling of high-volume low-value logs, drop debug in prod, per-team quotas,
  cardinality/label discipline, chargeback. 5 TB/day in a hot index is the budget-killer — keep
  the hot window small.
- Multi-tenancy: per-team access control, per-team dashboards, isolation so one team's volume
  spike doesn't degrade everyone.
- Reliability of the pipeline itself: buffering/backpressure so a sink outage doesn't drop logs or
  block apps; the logging system must not take down the services it observes.
- Compliance: retention lock / WORM on the audit tier, encryption, access audit, PII redaction in
  the pipeline (not after).
- *(staff)* Build vs buy TCO (Splunk/Datadog bill at this volume vs. an ops team running
  OpenSearch). Migration from whatever exists. Standardizing log format across 200 services (the
  real hard part — needs a library + linting + carrots).
**Follow-ups / curveballs:**
- (senior) "The hot index is $60k/month. Halve it without engineers losing the ability to debug."
- (staff) "One service starts logging 10x normal volume and floods the pipeline. What happens, and
  what should happen?"
**Red flags:** Everything into one giant Elasticsearch cluster with 1-year retention. No tiering.
No sampling or quotas. Pipeline that drops logs or backs up into the apps under load. PII redaction
"later". No build-vs-buy awareness.
**Tier notes:** Senior+ question. product — the pipeline + hot/cold tiering + cost control. MANG —
backpressure design, multi-tenancy isolation, build-vs-buy TCO, standardizing format across 200
services, the volume-spike failure mode.

---

### CLD-6 — Choose compute: VMs vs containers vs serverless
**Topics:** cloud architecture   **Levels:** mid, senior   **Format:** deep-dive
**Scenario (say this):** "For a given workload, how do you decide between plain VMs, containers on
a managed orchestrator, and serverless functions? Give me your decision framework."
**Candidate should clarify:** What's the workload — steady web service, spiky API, batch, event
processing, stateful? Team size and skill? Existing platform?
**Strong answer covers:**
- **Serverless (Lambda/Cloud Functions/Cloud Run):** best for spiky/low/unpredictable traffic,
  event-driven, glue, and small teams — no infra to run, scale-to-zero, pay-per-use. Costs:
  cold starts, execution time limits, vendor lock-in, harder local dev, cost can invert at high
  steady volume, statefulness is awkward.
- **Containers on managed orchestrator (ECS/EKS/GKE/Cloud Run):** best for most long-running
  services — portable, good tooling, right level of control, autoscaling. Costs: you own more
  (cluster upgrades, capacity, networking) especially with EKS; complexity budget.
- **VMs:** when you need full control (specific kernel, GPUs, licensing, long-lived stateful
  processes, lift-and-shift), or the simplest possible thing for one box. Costs: you own patching,
  images, scaling, config management.
- The framework: traffic shape (spiky→serverless, steady→containers/VMs), operational appetite,
  statefulness, latency sensitivity (cold starts), cost at expected volume, team skill, portability
  needs, existing investment.
- *(senior)* It's not all-or-nothing — serverless for the edges (auth, webhooks, cron), containers
  for the core, VMs for the one weird thing. Cost modeling at projected scale, not today's.
**Follow-ups / curveballs:**
- (mid) "An API with 10 req/min most of the day and 5000 req/min for one hour at 9am. Which, and
  why?"
- (senior) "You picked Lambda, it's now doing 500M invocations/month and the bill is huge and
  cold starts hurt p99. What do you do?" (→ provisioned concurrency, or move the hot path to
  containers; the workload outgrew the model)
**Red flags:** Dogmatic ("serverless always" / "Kubernetes always"). No cost-at-scale thinking.
Ignores team skill and operational load. Doesn't know serverless limits (timeout, cold start,
state).
**Tier notes:** service — a sensible framework + can place common workloads. product — hybrid
architectures, cost modeling, the "outgrew the model" scenario. MANG — quantified cost crossover
analysis, cold-start math, lock-in and portability strategy.

---

### CLD-7 — Blast radius and account structure
**Topics:** cloud governance, security   **Levels:** senior, staff   **Format:** design
**Scenario (say this):** "A company runs everything in one AWS account. A misconfigured IAM policy
in a dev script recently deleted a production S3 bucket. Design how you'd restructure to prevent
this class of problem."
**Candidate should clarify:** How many teams / environments? Compliance regime? Appetite for
migration pain? Using Organizations already?
**Strong answer covers:**
- **Multi-account via AWS Organizations** — separate accounts per environment (prod/staging/dev)
  and ideally per team/workload. An account is the strongest isolation boundary in AWS (blast
  radius, billing, quotas, security).
- **Service Control Policies (SCPs)** — org-level guardrails: deny deleting prod data, deny
  disabling CloudTrail, restrict regions, require encryption.
- Centralized: logging account (CloudTrail, Config), security account (GuardDuty, audit),
  networking account (shared VPC/TGW), identity via SSO with role assumption — no long-lived
  IAM users.
- Least-privilege roles per environment; dev credentials cannot touch prod because prod is a
  different account they have no role in.
- Landing zone tooling (Control Tower / org-formation / Terraform) so new accounts are
  provisioned consistently.
- Protections on critical resources: bucket policies, `prevent_destroy`, MFA-delete / object lock,
  backups in a separate account.
- *(staff)* Migration path — you can't big-bang this; new workloads land in the new structure,
  existing ones migrate by priority, prod first for isolation. Cost of the migration vs. risk.
  Governance model: who approves new accounts, how guardrails evolve, break-glass access.
**Follow-ups / curveballs:**
- (senior) "SCPs can lock you out too. How do you avoid bricking an account?"
- (staff) "Teams complain the guardrails slow them down and they can't self-serve. How do you
  balance that?" (→ paved road: generous permissions inside a sandbox account, tight on prod;
  guardrails as deny-lists not allow-lists; fast path to request exceptions)
**Red flags:** Solves it with tighter IAM policies in the same account. Doesn't know SCPs or
Organizations. No central logging/security account. No migration plan. Guardrails so tight nobody
can work.
**Tier notes:** Senior+ question. product — multi-account per environment, SCPs, SSO, central
logging. MANG — full landing-zone design, org governance model, paved-road philosophy, migration
strategy, break-glass.
