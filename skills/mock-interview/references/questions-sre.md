# Question Bank — SRE / Reliability

SLIs / SLOs / error budgets, incident response, on-call, postmortems, observability, capacity.

Scenario template: see `questions-core-devops.md`.

## Contents
- SRE-1 — Define an SLO for a checkout service
- SRE-2 — 2am latency spike, error rate flat
- SRE-3 — Postmortem for a repeat incident
- SRE-4 — On-call is burning people out
- SRE-5 — Observability for a new service
- SRE-6 — Error budget is exhausted, product wants to ship
- SRE-7 — Cascading failure / retry storm
- SRE-8 — Capacity planning for a launch spike
- SRE-9 — Slow rollout, partial errors, find it with tracing

---

### SRE-1 — Define an SLO for a checkout service
**Topics:** SLI/SLO, error budgets   **Levels:** mid, senior, staff   **Format:** design
**Scenario (say this):** "You own the checkout service for an e-commerce site. There's no SLO
today. Define one. Walk me through how you'd choose the SLIs, the targets, and what you'd do with
the error budget."
**Candidate should clarify:** What does checkout depend on (payment provider, inventory, cart)?
What's the current performance? Who's the customer of this SLO — the business, other teams? Traffic
pattern (spiky at sales)?
**Strong answer covers:**
- **SLI selection** — user-centric: request success rate (non-5xx, non-timeout) and latency (p99
  or a threshold like "% of requests < 500ms") measured **at the load balancer / from the user's
  side**, not from inside the service. Maybe a "checkout completed" business SLI.
- **Target** — grounded in current performance and user tolerance, not aspiration. E.g. 99.9%
  success over 28 days. Not 100% (impossible, and the budget is the point).
- **Error budget** — 0.1% of 28 days ≈ 40 min. Budget policy: burn it → freeze feature deploys,
  redirect to reliability work; healthy → ship faster, take more risk.
- **Measurement window** — rolling 28d; burn-rate alerts (fast burn = page, slow burn = ticket).
- *(senior)* Excluding dependency failures you can't control vs. owning the user experience anyway
  (you can add fallbacks, queues, retries). Multiple SLOs for different journeys.
- *(staff)* Rolling this out org-wide: how SLOs ladder up to a product-level reliability target,
  who arbitrates the error-budget policy, avoiding SLO theatre (targets nobody enforces).
**Follow-ups / curveballs:**
- (mid) "Why not just alert on CPU and memory?"
- (senior) "Your payment provider has a 99.5% SLA. Can checkout be 99.9%? Show me." (→ math:
  serial dependency caps you; need fallback / async / retry to beat it)
- (staff) "Product says the SLO is blocking a launch. How do you handle that conversation?"
**Red flags:** Picks infra metrics (CPU) as SLIs. Targets 100%. Measures from inside the service.
No error-budget policy — just a number. Doesn't know how serial dependencies compose.
**Tier notes:** service — knows the definitions and can pick a sane SLI/target. product — the
error-budget policy and burn-rate alerting. MANG — dependency math, multi-journey SLOs, org
rollout and the product-conflict conversation.

---

### SRE-2 — 2am latency spike, error rate flat
**Topics:** incident response, observability   **Levels:** junior, mid, senior   **Format:** troubleshooting
**Scenario (say this):** "You're on call for a payments API. 2am, pager fires: p99 latency on the
main endpoint went from 120ms to 3s over ten minutes. Error rate is flat. Walk me through what you
do."
**Candidate should clarify:** Is it all endpoints or one? All regions? Correlated with a deploy?
Traffic volume normal? Is this customer-impacting enough to declare an incident?
**Strong answer covers:**
- **Stabilize before diagnose.** Assess impact, declare an incident if warranted, start a timeline
  and comms.
- Check the obvious correlations first: recent deploy (roll back if suspicious), config change,
  feature flag, traffic spike, dependency latency.
- Use the layers: LB/edge metrics → app metrics (which handler, GC pauses, thread pool / connection
  pool saturation) → downstream (DB query time, cache hit rate, external API latency) → infra (node
  CPU steal, disk, network).
- "Latency up, errors flat" pattern → often resource saturation or a slow dependency, not a crash:
  connection pool exhaustion, DB lock contention, a missing index after data growth, cache
  eviction, noisy neighbor, GC.
- Mitigations available now: scale out, shed load / rate-limit, disable an expensive feature,
  fail over a region, increase a pool size.
- *(senior)* Reasoning about *why errors are flat* — retries/timeouts masking it, or the slowness
  is upstream of the error accounting. USE / RED method to structure the hunt.
**Follow-ups / curveballs:**
- (junior) "Where do you look first and why?"
- (mid) "DB CPU is at 95%. You didn't deploy. What could cause that with no code change?" (→ data
  growth crossing a plan tipping point, a batch job, a missing index, lock contention, a new
  query from another service, stats out of date)
- (senior) "Rolling back the last deploy doesn't help. Latency's still 3s. What's your next move
  and how do you avoid thrashing?"
**Red flags:** Starts SSHing into boxes before assessing impact or declaring an incident. No
mental model of the request path. Assumes it must be the code. Doesn't consider mitigations,
only root cause. No comms.
**Tier notes:** service — a structured checklist through the layers. product — the
saturation-vs-crash reasoning and real mitigations. MANG — RED/USE structure, hypothesis-driven
debugging, "errors flat" analysis, and managing the incident (roles, comms cadence).

---

### SRE-3 — Postmortem for a repeat incident
**Topics:** postmortems, culture   **Levels:** mid, senior, staff   **Format:** deep-dive
**Scenario (say this):** "The same class of incident — a config change taking down a service — has
happened three times in six months. Each had a postmortem with action items. It keeps happening.
What's going wrong and how do you fix it?"
**Candidate should clarify:** Are the action items being completed? Same team each time? Same root
cause or same symptom?
**Strong answer covers:**
- Postmortems that don't change anything are theatre. Likely failures: action items not
  prioritized against feature work, items that treat symptoms not systems ("be more careful",
  "add a checklist"), no owner, no due date, no follow-up.
- **Fix the system, not the human.** Config changes taking down prod = make bad config
  un-shippable: schema validation, staged config rollout with canary, automated rollback on
  health regression, config as code with the same review/CI as app code, typed config.
- Track action items like incidents-prevented: dashboard, SLA on completion, review in a regular
  reliability meeting, escalate stale ones.
- *(senior)* Blameless culture check — if people are hiding near-misses, you only see the ones that
  blew up. Encourage reporting.
- *(staff)* Is this a single team's problem or a platform gap? If three teams hit it, the platform
  should make it impossible. Org-level tracking of incident themes; an "error budget" for the
  class of failure.
**Follow-ups / curveballs:**
- (mid) "Give me two concrete action items from a config-change outage that actually reduce
  recurrence."
- (staff) "Leadership wants to add a mandatory change-approval board. Good idea?" (→ mostly no;
  slows everything, doesn't catch the failure, pushes people to batch risky changes; invest in
  progressive delivery instead)
**Red flags:** Blames the engineers. Action items are all "add a checklist / be careful". Doesn't
distinguish symptom from system. Adds process instead of removing the failure mode.
**Tier notes:** service — process discipline and tracking. product — system fixes over process,
blameless culture. MANG — platform-level prevention, org incident-theme tracking, skepticism of
heavyweight change control.

---

### SRE-4 — On-call is burning people out
**Topics:** on-call, operations   **Levels:** mid, senior, staff   **Format:** deep-dive
**Scenario (say this):** "Your team's on-call engineers are getting paged 15–20 times per week,
several overnight. People are quitting. You're asked to fix on-call. Where do you start?"
**Candidate should clarify:** How many are actionable vs noise? Same alerts repeating? Rotation
size? Is there a follow-the-sun option?
**Strong answer covers:**
- **Measure first** — page volume, % actionable, % overnight, top offenders, MTTA/MTTR, which
  alerts and which services.
- Kill noise: alerts that don't need human action *now* → tickets or delete. Every alert should be
  urgent, actionable, and documented (linked runbook).
- Fix the top 3 pagers with engineering — auto-remediation, capacity, a real bug fix, better
  defaults. Treat the noisiest alert as a project.
- Alert on symptoms (SLO burn) not causes (one node's CPU); dedupe/group; sensible thresholds and
  for-durations.
- Operational load as a first-class backlog item with capacity reserved; on-call not also expected
  to deliver a full sprint.
- Rotation health: enough people (6–8), comp/time-off-in-lieu for nights, handoff notes.
- *(staff)* If a service can't be made quiet, that's a signal about its architecture or ownership.
  Follow-the-sun to eliminate overnight pages. Set a page-budget SLO and hold the line.
**Follow-ups / curveballs:**
- (mid) "An alert fires every night at 3am, auto-resolves in 5 min, never customer-impacting. What
  do you do with it?"
- (staff) "You've cut pages 70% but one legacy service is still 8/week and the team that owns it
  won't invest. What now?"
**Red flags:** "Just add more people to the rotation." Keeps non-actionable alerts "just in case".
No measurement. Treats on-call pain as inevitable.
**Tier notes:** service — noise reduction + runbooks + rotation size. product — symptom-based
alerting, operational load as backlog. MANG — page-budget SLO, follow-the-sun, architectural
signal from un-quietable services.

---

### SRE-5 — Observability for a new service
**Topics:** observability   **Levels:** junior, mid, senior   **Format:** design
**Scenario (say this):** "A team is about to launch a new service. What observability do you want
in place before it takes production traffic?"
**Candidate should clarify:** What kind of service — request/response, async worker, data
pipeline? What are its dependencies? Existing platform (Prometheus, OTel, a vendor)?
**Strong answer covers:**
- **The three pillars, with purpose:**
  - Metrics — RED (rate, errors, duration) for request services or USE (utilization, saturation,
    errors) for resources; SLI metrics specifically.
  - Logs — structured (JSON), with a request/trace ID, levels, no secrets/PII, sampled if
    high-volume.
  - Traces — distributed tracing across the dependency graph; at least the critical path.
- Dashboards: one "is it healthy" overview (SLIs), one per-dependency, one for the on-call.
- Alerts: SLO burn-rate alerts wired before launch; nothing that pages that isn't actionable.
- Health/readiness endpoints that reflect real dependency health.
- *(mid)* Cardinality discipline (labels), cost of retention, exemplars linking metrics→traces.
- *(senior)* "How would you know it's broken *before* a customer tells you?" — synthetic probes,
  golden-signal alerting, watching the SLI not the host. What's the debugging story at 3am — can
  you go from alert → dashboard → trace → log line in a couple of clicks?
**Follow-ups / curveballs:**
- (junior) "What's the difference between a metric and a log, and when do you reach for each?"
- (mid) "Your logging bill is now $40k/month. How do you cut it without going blind?"
- (senior) "The service is async — jobs off a queue. RED doesn't quite fit. What do you measure?"
  (→ queue depth, age of oldest message, processing rate, failure/DLQ rate, end-to-end latency)
**Red flags:** "Add logging" with no structure or strategy. Alerts on host metrics. No traces for
a microservice. Can't explain what each pillar is *for*. Ignores cost/cardinality.
**Tier notes:** service — the three pillars + basic dashboards + health checks. product —
SLI-driven alerting, cost control, the 3am debugging path. MANG — cardinality/cost architecture,
async-service instrumentation, "know before the customer" via synthetics + burn-rate.

---

### SRE-6 — Error budget exhausted, product wants to ship
**Topics:** error budgets, culture   **Levels:** senior, staff   **Format:** deep-dive
**Scenario (say this):** "Your service blew its error budget two weeks into the quarter — a bad
month. The error-budget policy says feature deploys freeze until it recovers. Product has a
committed launch next week. The VP asks you to make an exception. What do you do?"
**Candidate should clarify:** What caused the burn — one incident or chronic? Is the launch
reliability-risky itself? What does "exception" mean — skip the freeze entirely, or ship with
extra guardrails?
**Strong answer covers:**
- The policy exists so this conversation is about data, not vibes. Start there.
- Separate a **one-off incident** (already fixed, low recurrence risk) from **chronic burn** (the
  service is genuinely unreliable). A one-off with a clear fix is a reasonable case for a scoped
  exception; chronic burn is not.
- If you ship: reduce the risk — extra canary time, feature flag with instant kill, no other
  changes in flight, heightened monitoring, a named rollback owner.
- Make the tradeoff explicit and shared: "we can ship, and here's the reliability risk we're
  accepting and who signed off." Put it in writing.
- *(staff)* The meta-issue: if exceptions are routine, the policy is dead and you should redesign
  it (or the org doesn't actually value reliability and that's a different problem). Escalation
  path, who owns the budget, how to make the freeze rare by shipping safer continuously.
**Follow-ups / curveballs:**
- (senior) "The VP says 'reliability is not the priority this quarter, ship it.' Your move." (→
  disagree-and-commit if it's their call to make and it's documented; make sure the risk is
  visible; don't martyr yourself, don't silently comply either)
- (staff) "How do you design an error-budget policy that survives contact with a launch-driven
  org?"
**Red flags:** Rigidly refuses with no risk analysis. Caves instantly with no guardrails. Makes it
a personal standoff. Doesn't get the decision in writing.
**Tier notes:** Mostly a product/MANG question. product — scoped exception + guardrails + written
tradeoff. MANG — one-off vs chronic distinction, disagree-and-commit, policy redesign.

---

### SRE-7 — Cascading failure / retry storm
**Topics:** reliability patterns, incident   **Levels:** senior, staff   **Format:** deep-dive
**Scenario (say this):** "Service A calls service B. B has a brief 30-second blip. When B recovers,
it immediately falls over again and now A is down too. This keeps oscillating. What's happening and
how do you stop it?"
**Candidate should clarify:** Do A's calls to B retry? Timeouts configured? Is there a queue
between them? How is B scaled?
**Strong answer covers:**
- **Retry storm / thundering herd:** during B's blip, A's requests time out and retry; failed
  work queues up; when B recovers it's hit with the backlog + normal load at once → falls over →
  repeat. Made worse by synchronized retries and unbounded queues.
- Immediate mitigation: shed load at B (rate-limit / reject fast), stop A's retries temporarily,
  scale B, drain the backlog gradually.
- Fixes:
  - Retries with **exponential backoff + jitter**, capped attempts, and a **retry budget** (don't
    let retries exceed X% of traffic).
  - **Circuit breaker** in A — trip open when B is failing, fail fast, half-open to probe.
  - **Timeouts** everywhere, shorter than the caller's patience; deadline propagation.
  - **Load shedding** and admission control at B; prioritize by request importance.
  - Bounded queues; drop or reject when full rather than growing unboundedly.
- *(staff)* Backpressure end-to-end, graceful degradation (serve stale/cached, reduced
  functionality), capacity headroom for recovery surges, and testing this with fault injection so
  it's not discovered in prod.
**Follow-ups / curveballs:**
- (senior) "A already has retries with backoff and it still happened. Why?" (→ no jitter, no retry
  budget, no circuit breaker, queue unbounded, every client retries at the same cap)
- (staff) "How do you validate the fix before trusting it in prod?" (→ load test with induced
  dependency failure; game day; chaos experiment)
**Red flags:** "Just add retries." Doesn't know circuit breakers or jitter. No load-shedding
concept. Thinks scaling B infinitely is the answer. Can't explain why recovery makes it worse.
**Tier notes:** Senior+ question. product — backoff+jitter, circuit breaker, timeouts, bounded
queues. MANG — retry budgets, deadline propagation, end-to-end backpressure, fault-injection
validation.

---

### SRE-8 — Capacity planning for a launch spike
**Topics:** capacity, reliability   **Levels:** mid, senior, staff   **Format:** design
**Scenario (say this):** "Marketing is running a campaign in three weeks. They expect it to drive
roughly 10x normal traffic to the signup and checkout flows for about a 6-hour window. Today the
system runs comfortably at 2,000 requests/second. How do you make sure launch day doesn't fall
over?"
**Candidate should clarify:** Is 10x a peak or an average over the window? Which endpoints
specifically? What's the current headroom and autoscaling setup? Any hard dependencies (payment
provider, a shared DB, third-party APIs) with their own limits? What's the acceptable degradation?
**Strong answer covers:**
- **Quantify:** 10x of 2,000 rps ≈ 20,000 rps peak. Work out what that means per tier — app
  instances (rps per instance × N), DB (connections, IOPS, read/write split), caches, queues,
  and every downstream. Find the **bottleneck resource** first; adding web capacity is pointless
  if the DB or the payment provider caps you.
- **Load test** at target + margin (say 1.5x) against a prod-like environment, ideally prod
  itself in a controlled window. Test the real user journeys, not just a single endpoint. Watch
  for the knee, not just the ceiling.
- **Pre-scale** rather than trusting autoscaling to react in time (warm-up lag, image pulls, node
  provisioning). Raise autoscaling floors and ceilings; pre-warm caches and connection pools;
  check cloud quotas and ask for increases early (they take days).
- **Dependencies:** confirm the payment provider / SMS / email / third-party APIs can take the
  volume — tell them, get written confirmation, know their rate limits and your behavior when
  throttled.
- **Degradation plan:** what you shed first (non-critical features, async-ify what you can),
  load shedding / queueing at the edge, a "we're busy, try again" path that's better than 5xx.
- **Runbook + staffing:** dashboards for launch day, a war room, on-call staffed up, a clear
  rollback / feature-kill plan, and comms with marketing on timing.
- *(staff)* Cost of holding that capacity vs. the revenue at stake; whether to keep some of it
  afterward; post-launch review to feed the next one.
**Follow-ups / curveballs:**
- (mid) "Your load test hits 12,000 rps and p99 latency triples but errors stay near zero. Ship
  it or not?"
- (senior) "The shared Postgres is the bottleneck and you can't shard in three weeks. Options?"
  (→ read replicas + route reads, aggressive caching, a read-through cache for hot data,
  connection pooling / pgbouncer, defer non-critical writes, raise instance size as a stopgap)
- (staff) "Marketing won't commit to a time window — they 'might' also email 2M users that day.
  How do you plan under that uncertainty?"
**Red flags:** No math — just "add more servers". Trusts autoscaling to absorb a 10x step with no
pre-scaling. Forgets the database or the third-party dependencies. No load test. No degradation
plan. Doesn't check cloud quotas.
**Tier notes:** service — a sensible checklist: estimate, load test, pre-scale, runbook. product —
bottleneck analysis, dependency confirmation, a real degradation plan. MANG — the capacity math
out loud, knee-vs-ceiling, planning under uncertainty, cost/revenue tradeoff.

---

### SRE-9 — Slow rollout, partial errors, find it with tracing
**Topics:** observability, incident, troubleshooting   **Levels:** mid, senior   **Format:** troubleshooting
**Scenario (say this):** "You roll out a new version of a service across 30 pods over about 20
minutes. During the rollout, roughly 3% of requests start returning 500s — but only some
requests, and it doesn't stop when the rollout finishes. You have metrics, logs, and distributed
tracing. Walk me through finding the cause."
**Candidate should clarify:** Do the 500s correlate with the new-version pods specifically? One
endpoint or many? Did anything else ship (config, migration, a flag)? Is 3% steady or growing?
**Strong answer covers:**
- **Slice the error metric:** by pod / version label first — is it only the new version, only old,
  or both? By endpoint, by client, by region. This alone often localizes it.
- **Use traces:** pull traces for the failing requests (filter by status = 500). Find the span
  that errors — is it in this service, or a downstream call? Compare a failing trace to a passing
  one for the same endpoint.
- Common "new version, partial errors, persists after rollout" causes: a code path only some
  inputs hit (a specific tenant, feature flag, payload shape); a new/changed downstream call that
  fails under some conditions; a connection-pool or client mis-config; a schema/enum the new code
  writes that something else can't read; a dependency that rate-limits the new call pattern; two
  versions of a shared cache entry or message format coexisting.
- Logs: filter to the 500s, get the actual error/stack, correlate with the trace ID.
- **Mitigate while investigating:** if it's clearly the new version, roll back or halt the
  rollout; if rollback isn't clean (migration), consider a feature flag or forward-fix.
- *(senior)* Why it persists after the rollout completes: the trigger isn't "being mid-rollout",
  it's the new code itself on a subset of traffic — so the rollout was a red herring for the
  *timing* but the version is still the cause. Or: a cache/queue now holds poisoned entries
  written during the rollout that outlive it.
**Follow-ups / curveballs:**
- (mid) "Traces show the 500 comes from a call to the `pricing` service that returns 429. Only
  from the new version. What happened?" (→ new version changed the call pattern — no caching, a
  retry loop, N+1 — and is now hitting pricing's rate limit)
- (senior) "Error rate is identical on old and new pods. Does that change your hypothesis?" (→
  yes — probably not the deploy at all; something shared changed: a downstream, a config, data
  growth, a flag flipped by another team, a dependency incident)
**Red flags:** Only looks at aggregate error rate, never slices by version/endpoint. Doesn't use
traces despite having them. Assumes "errors during rollout" = "rollout mechanics" and stops.
Rolls back and declares victory without confirming the errors stopped.
**Tier notes:** mid — slice by version, read a trace, roll back if it's the new version. senior —
the "persists after rollout" reasoning, poisoned-cache/queue entries, the "identical on both
versions" pivot.
