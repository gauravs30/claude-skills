# Question Bank — Core DevOps

CI/CD, infrastructure as code, containers, Kubernetes, GitOps, release strategies.

## Scenario template

```
### <id> — <title>
**Topics:** ...   **Levels:** junior|mid|senior|staff   **Format:** troubleshooting|design|deep-dive|coding
**Scenario (say this):** "<how an interviewer poses it>"
**Candidate should clarify:** <what a strong candidate asks before answering>
**Strong answer covers:** <bulleted key points; deeper bullets are for higher levels>
**Follow-ups / curveballs:** <tagged by level>
**Red flags:** <answers that should lower the score>
**Tier notes:** service / product / MANG differences
```

## Contents
- CD-1 — Failed deploy, need to get back to green
- CD-2 — Design a CI/CD pipeline for a monorepo
- CD-3 — Kubernetes: pod won't start
- CD-4 — Rolling vs blue-green vs canary
- CD-5 — Terraform state got corrupted
- CD-6 — Container image is 1.8 GB and builds take 20 min
- CD-7 — Secrets in the pipeline
- CD-8 — GitOps: prod drifted from Git
- CD-9 — Coding: a health-gated deploy script

---

### CD-1 — Failed deploy, need to get back to green
**Topics:** CD, incident, releases   **Levels:** junior, mid, senior   **Format:** troubleshooting
**Scenario (say this):** "You push a release to production at 4pm Friday. Within two minutes,
error rate on the API jumps to 20% and stays there. You're the engineer who shipped it. What do
you do?"
**Candidate should clarify:** Is there automated rollback? What deploy strategy is in use (rolling
/ blue-green / canary)? Do we have the previous artifact? Is traffic user-facing / revenue-facing?
**Strong answer covers:**
- Roll back first, diagnose second — restore the last known-good version/artifact, don't debug in
  prod under load.
- Mechanics of the rollback for the stated strategy (shift traffic back, redeploy previous image,
  `kubectl rollout undo`, revert the release pointer in GitOps).
- Confirm recovery with the same signal that detected it (error rate, not just "pods are up").
- Communicate: declare an incident, post status, note the time.
- *(senior)* Check for non-reversible changes shipped in the same release — DB migrations, feature
  flags, schema changes, message formats. Rollback is not safe if the new code wrote data the old
  code can't read.
- *(senior)* After: blameless postmortem, and why did CI/canary not catch this — missing test,
  canary too short, no automated rollback.
**Follow-ups / curveballs:**
- (mid) "The release included a database migration that added a NOT NULL column. Does that change
  your rollback?"
- (senior) "Rollback doesn't help — error rate stays at 20%. Now what?" (→ it's not the deploy;
  correlate with dependencies, infra events, traffic)
- (senior) "How do you make Friday-4pm deploys safe enough that this is a non-event?"
**Red flags:** Starts reading logs and adding print statements before rolling back. Doesn't
mention communication. Assumes rollback is always safe. "We don't deploy on Fridays" as the whole
answer.
**Tier notes:** service — expect a clean runbook-style answer. product — expect the migration
nuance and a real postmortem instinct. MANG — expect automated rollback as table stakes and a
discussion of progressive delivery + why the canary missed it.

---

### CD-2 — Design a CI/CD pipeline for a monorepo
**Topics:** CI/CD, IaC, releases   **Levels:** mid, senior, staff   **Format:** design
**Scenario (say this):** "We have a monorepo: ~15 services, a shared library, and Terraform for
infra, all in one Git repo. Right now every push runs everything and takes 40 minutes. Design the
CI/CD pipeline you'd want."
**Candidate should clarify:** How often do people push? How independent are the services? Deploy
cadence — continuous or batched? Same runtime (all containers on k8s) or mixed? Who owns infra
changes?
**Strong answer covers:**
- **Affected-only builds** — detect changed paths, build/test only affected services + their
  dependents (Bazel/Nx/Turborepo, or path filters as a simpler start).
- Pipeline stages: lint → unit → build image → integration → publish artifact → deploy (per env).
- Caching: dependency cache, layer cache, test result cache.
- Parallelism / sharding of test suites.
- Artifact strategy: immutable image tagged by commit SHA, promoted across envs (not rebuilt).
- Infra changes: `terraform plan` on PR as a required check, `apply` gated (manual approval or
  merge-to-main), state locking.
- Deploy: per-service, GitOps or pipeline-driven, with the release strategy from CD-4.
- *(senior)* Branch protection, required checks, merge queue to avoid "green on branch, broken on
  main". Handling the shared library — version it or rebuild dependents.
- *(staff)* Rollout across the org: migration path from the current setup, how to keep the 40-min
  pipeline working while the new one is built, ownership model (platform team owns the template,
  service teams own their stage config), guardrails vs. flexibility.
**Follow-ups / curveballs:**
- (mid) "A change to the shared library — what runs?"
- (senior) "How do you stop a broken merge to main from blocking all 15 teams?"
- (staff) "The security team wants an SBOM and image signing on every artifact. Where does that go
  and who owns it?"
**Red flags:** Rebuilds everything on every push and doesn't see the problem. Rebuilds images per
environment. No `terraform plan` gate. Treats infra and app code identically with no approval
step. No caching.
**Tier notes:** service — a clean linear pipeline with path filters is acceptable. product —
expect affected-only, immutable artifacts, plan-gates. MANG — expect the org rollout, ownership
model, and merge queue discussion; push on scale (150 services, not 15).

---

### CD-3 — Kubernetes: pod won't start
**Topics:** Kubernetes, troubleshooting   **Levels:** junior, mid, senior   **Format:** troubleshooting
**Scenario (say this):** "You deploy a new service to Kubernetes. The Deployment is created but no
pods are Ready. `kubectl get pods` shows 0/1 Running for every replica. Walk me through debugging
this."
**Candidate should clarify:** Is it `Running` but not `Ready`, or stuck `Pending` /
`CrashLoopBackOff` / `ImagePullBackOff`? New cluster or existing? Did it ever work?
**Strong answer covers:**
- `kubectl describe pod` — read Events. `kubectl logs` (and `--previous` for crash loops).
- Branch on the state:
  - `Pending` → scheduling: insufficient resources, node selector/affinity, taints, PVC unbound.
  - `ImagePullBackOff` → registry auth (imagePullSecret), wrong tag, network to registry.
  - `CrashLoopBackOff` → app fails on start: bad config/env, missing secret, can't reach a
    dependency, migration on boot failing.
  - `Running` but not `Ready` → readiness probe failing: wrong path/port, probe too aggressive,
    app genuinely not healthy.
- Check the readiness probe definition against what the app actually serves.
- *(mid)* `kubectl get events --sort-by=.lastTimestamp`, check resource requests vs node capacity,
  check the ServiceAccount and RBAC if the app talks to the API.
- *(senior)* Distinguish "my manifest is wrong" from "the platform is broken" (admission webhook
  rejecting, PSP/PSA, quota, CNI issue, node pressure) — and how you'd tell.
**Follow-ups / curveballs:**
- (junior) "`describe` says `0/3 nodes available: 3 Insufficient memory`. What now?"
- (mid) "Logs show the app starting fine, but readiness stays false. Probe hits `/healthz` on
  8080. What do you check?"
- (senior) "It works in staging, fails in prod, same manifest. Where do you look?" (→ secrets,
  network policy, image pull from prod registry, resource quotas, PSA level)
**Red flags:** Jumps straight to `kubectl delete pod` and hopes. Doesn't know `describe` shows
events. Confuses readiness and liveness. Can't name the common pod phases.
**Tier notes:** service/junior — the state-branching checklist is the win. product — expect the
staging-vs-prod diff reasoning. MANG — expect platform-vs-user fault isolation and probe-tuning
nuance.

---

### CD-4 — Rolling vs blue-green vs canary
**Topics:** releases   **Levels:** mid, senior   **Format:** deep-dive
**Scenario (say this):** "Explain rolling, blue-green, and canary deployments. When would you pick
each, and what does each cost you?"
**Candidate should clarify:** For what kind of service — stateless API, stateful, batch? What's
the current infra?
**Strong answer covers:**
- **Rolling:** replace instances incrementally. Cheap (no extra capacity beyond surge), built into
  k8s. Downsides: both versions serve simultaneously (API/DB compat matters), slow to fully roll
  back, limited blast-radius control.
- **Blue-green:** stand up full new environment, cut over atomically, keep old for fast rollback.
  Instant rollback, clean testing of green before cutover. Costs: 2x capacity during the window,
  cutover is all-or-nothing, stateful/session handling is harder, DB is shared and still needs
  compat.
- **Canary:** route a small % to the new version, watch metrics, ramp or abort. Best blast-radius
  control, real production signal. Costs: needs good metrics + automation to be safe, slower,
  requires traffic-splitting infra, both versions run (compat again).
- The cross-cutting constraint: **backward/forward compatibility** of API contracts, DB schema,
  message formats, and cache entries — every strategy runs two versions at once.
- *(senior)* Automated canary analysis (Argo Rollouts / Flagger), SLO-based abort, and pairing the
  strategy with feature flags to decouple deploy from release.
**Follow-ups / curveballs:**
- (mid) "Your service holds websocket connections. Which strategy, and what breaks?"
- (senior) "Canary looks healthy at 5% for 10 minutes, you go to 100%, error rate spikes. How is
  that possible?" (→ load-dependent bugs, resource saturation, downstream rate limits, cache
  stampede, cohort bias in the 5%)
**Red flags:** Thinks blue-green removes the need for schema compatibility. Doesn't mention cost /
capacity. Can't articulate why canary needs metrics automation. Conflates deploy and release.
**Tier notes:** service — clear definitions + one good "when" each. product — the compatibility
constraint and feature-flag decoupling. MANG — automated canary analysis, statistical validity of
the canary sample, SLO-based gating.

---

### CD-5 — Terraform state got corrupted
**Topics:** IaC, Terraform, incident   **Levels:** mid, senior   **Format:** troubleshooting
**Scenario (say this):** "A colleague ran `terraform apply` from their laptop while CI was also
applying. Now `terraform plan` wants to destroy and recreate half your production infrastructure.
What happened and what do you do?"
**Candidate should clarify:** Where's the state — S3 + DynamoDB lock, Terraform Cloud, local?
Is there state file versioning? What exactly does the plan want to change?
**Strong answer covers:**
- Root cause: concurrent applies without locking → state now disagrees with reality, or the two
  applies interleaved and one overwrote the other's state.
- **Do not apply.** A plan that wants to destroy prod is a stop sign.
- Recovery: pull current state, compare to reality. Restore a known-good state version (S3 object
  versioning / Terraform Cloud state history). Use `terraform state` surgery — `state rm`,
  `import`, `state mv` — to reconcile individual resources rather than a blanket apply.
- `terraform plan` iteratively until it shows no destructive changes, then apply.
- *(senior)* Prevention: remote state with locking (DynamoDB table / native lock), no local applies
  — CI is the only actor, enforced by permissions. `prevent_destroy` lifecycle on critical
  resources. Plan/apply separation with saved plan files. Drift detection on a schedule.
**Follow-ups / curveballs:**
- (mid) "There's no state versioning and no backup. Now what?" (→ `import` everything, or targeted
  applies, accept it's painful; then fix the backend)
- (senior) "How do you stop laptop applies organizationally, not just with a policy doc?" (→ IAM:
  humans get read-only to the state bucket and no prod write creds; only the CI role can apply)
**Red flags:** "Just run apply and let it recreate things." Doesn't know state can be versioned.
Never heard of `terraform state` subcommands or `import`. No prevention story.
**Tier notes:** service — locking + "CI only" is the expected answer. product — expect state
surgery fluency. MANG — expect the IAM-enforced separation and drift detection, plus module/state
splitting to bound blast radius.

---

### CD-6 — Container image is 1.8 GB and builds take 20 min
**Topics:** containers, CI   **Levels:** junior, mid, senior   **Format:** troubleshooting
**Scenario (say this):** "A team's service image is 1.8 GB and the Docker build takes 20 minutes
in CI. They want it faster and smaller. How do you approach it?"
**Candidate should clarify:** Language/runtime? What's in the image? Multi-stage already? What's
slow — dependency install, compilation, the CI runner itself?
**Strong answer covers:**
- **Multi-stage build** — build deps and toolchain in a builder stage, copy only the artifact into
  a slim runtime base (distroless, alpine, slim).
- **Layer ordering for cache** — copy dependency manifests and install deps *before* copying
  source, so code changes don't bust the dependency layer.
- **BuildKit / cache mounts** — `--mount=type=cache` for package caches; remote layer cache in CI.
- `.dockerignore` to keep build context small.
- Pin base images; don't `apt-get upgrade` the world.
- Measure: `docker history`, `dive` to find the fat layers.
- *(senior)* Is 20 min the build or the CI runner cold-starting / pulling? Warm caches, bigger
  runners, or a build farm. Distinguish image size (pull time, attack surface) from build time
  (developer feedback) — different fixes.
**Follow-ups / curveballs:**
- (junior) "What's the difference between `COPY . .` then `RUN npm install` vs the reverse?"
- (mid) "Multi-stage gets it to 400 MB. The base runtime is still 300 MB of that. Options?"
- (senior) "Build cache works locally but never hits in CI. Why?" (→ ephemeral runners, no shared
  cache backend, cache key includes a timestamp, layers not pushed)
**Red flags:** Doesn't know multi-stage builds. Puts `RUN npm install` before `COPY package.json`.
Thinks alpine is always the answer (glibc/musl gotchas). No measurement, just guesses.
**Tier notes:** service — multi-stage + layer caching + .dockerignore. product — the size-vs-time
distinction and CI cache backends. MANG — build-farm / remote-cache architecture, reproducibility,
supply-chain (pinned digests, provenance).

---

### CD-7 — Secrets in the pipeline
**Topics:** CI/CD, security   **Levels:** mid, senior   **Format:** deep-dive
**Scenario (say this):** "Your CI pipeline needs cloud credentials to deploy, a database password
for integration tests, and a registry token to push images. How do you manage these secrets?"
**Candidate should clarify:** Which CI system? Cloud provider? Self-hosted or SaaS runners?
**Strong answer covers:**
- **No long-lived cloud keys** — use OIDC federation: the CI provider issues a short-lived token,
  the cloud trusts it via a role with a scoped trust policy. No secret to leak.
- Secrets from a manager (Vault, cloud secrets manager, or the CI's native encrypted secrets),
  injected as env/files at run time, scoped to the job that needs them.
- Least privilege: the deploy role can deploy, not admin. Separate creds per environment.
- Never in the repo, never in image layers, never echoed to logs (masking is a backstop, not the
  control).
- Short TTLs, rotation, and audit logging on secret access.
- *(senior)* Protecting against a malicious PR: don't expose prod secrets to fork/PR builds;
  require approval for workflows that touch protected environments; pin third-party actions by
  SHA (supply chain). Runner isolation so one job can't read another's secrets.
**Follow-ups / curveballs:**
- (mid) "A contributor opens a PR that adds `env | curl` to the CI config. What stops your secrets
  from being exfiltrated?"
- (senior) "You're on self-hosted runners in your own VPC. What changes?" (→ runner hygiene,
  ephemeral runners, network egress control, they now have a foothold in your network)
**Red flags:** "Store them as encrypted CI variables" with no mention of OIDC or least privilege.
Exposes secrets to PR builds. Trusts log masking as the primary control. Uses one admin key for
everything.
**Tier notes:** service — encrypted CI secrets + least privilege + not in the repo. product —
OIDC federation and PR-safety. MANG — supply-chain (pinned actions, provenance), runner isolation,
blast-radius analysis of a compromised runner.

---

### CD-8 — GitOps: prod drifted from Git
**Topics:** GitOps, Kubernetes, IaC   **Levels:** mid, senior, staff   **Format:** deep-dive
**Scenario (say this):** "You run GitOps with Argo CD (or Flux) — Git is meant to be the source of
truth for the cluster. During an incident last night, someone ran `kubectl edit` and `kubectl
scale` directly against production to mitigate. This morning, what's the state of the world, and
what do you do?"
**Candidate should clarify:** Is auto-sync on, with or without self-heal / prune? Was the manual
change to something Argo manages? Is the incident actually resolved?
**Strong answer covers:**
- What GitOps does with drift: with self-heal enabled, Argo reverts the manual change on the next
  sync — which could *re-break* prod if the manual change was the mitigation. With self-heal off,
  it shows `OutOfSync` and waits.
- Right sequence: confirm the mitigation is still needed; if yes, get it into Git (PR) so the
  desired state matches reality *before* any sync reverts it; if the incident is over, revert
  cleanly and let Git win.
- Why manual changes are dangerous here: they're invisible to the next person, they don't survive
  a pod reschedule / redeploy, and they create a divergence that bites later.
- *(senior)* Break-glass process: a documented, audited way to bypass GitOps under pressure (pause
  auto-sync for that app, make the change, open a follow-up PR within X hours). RBAC so most
  people *can't* `kubectl edit` prod at all.
- *(staff)* Drift detection and alerting as a standing control; periodic reconcile; the cultural
  work to make "change Git, not the cluster" the reflex even at 3am; where HPA-style legitimate
  runtime state (replica counts) should be excluded from drift detection.
**Follow-ups / curveballs:**
- (mid) "Auto-sync with self-heal is on. What literally happens at the next sync interval?"
- (senior) "The manual change was `kubectl scale` to 20 replicas. HPA is also configured. What's
  the interaction, and what should be in Git?"
- (staff) "How do you let on-call mitigate fast without turning GitOps into theatre?"
**Red flags:** Doesn't know self-heal would revert the mitigation. "Just always let Git win" with
no thought for an in-flight incident. No break-glass concept. Thinks `kubectl edit` on prod is
fine.
**Tier notes:** service — understands drift and that Git should win once the incident's over.
product — the self-heal-reverts-the-mitigation trap, break-glass. MANG — drift as a monitored
control, HPA/GitOps interaction, org-level reflexes.

---

### CD-9 — Coding: a health-gated deploy script
**Topics:** scripting, CD, releases   **Levels:** mid, senior   **Format:** coding
**Scenario (say this):** "Write a script — Bash or Python — that deploys a new version of a
service and only considers the deploy successful if the service passes a health check afterward.
If it doesn't go healthy within a timeout, the script should roll back and exit non-zero. Talk me
through your design before you code."
**Candidate should clarify:** What's the deploy mechanism (kubectl, a systemd unit, an API call)?
What's the health check — an HTTP endpoint, a command? What does 'rollback' mean here — previous
image tag, `kubectl rollout undo`, redeploy the last artifact? Single instance or a fleet?
**Strong answer covers:**
- Structure: capture the current version first (for rollback), trigger the deploy, then **poll**
  the health check with a timeout and interval — not a single check, not `sleep 30 && curl`.
- Polling loop: bounded total time, sleep between attempts, treat non-200 / connection refused /
  timeout as "not healthy yet", succeed on N consecutive healthy checks (not just one — avoid a
  transient pass).
- On timeout: roll back to the captured version, re-verify health of the rolled-back version,
  exit non-zero with a clear message.
- Hygiene: `set -euo pipefail` (Bash) or exceptions (Python); quote variables; don't leave the
  service in an unknown state; log what it's doing; make the timeout/interval configurable.
- *(senior)* Idempotency / re-run safety; what if the rollback itself fails (alert, page, don't
  loop forever); a fleet needs rolling health-gating not all-at-once; the check should hit the
  thing users hit (LB), not localhost; distinguish "deploy API failed" from "deployed but
  unhealthy".
**Follow-ups / curveballs:**
- (mid) "Why poll instead of one check after a fixed sleep?"
- (mid) "Your health check passes once then the pod crashes 10 seconds later. How does your script
  behave, and how would you harden it?"
- (senior) "Rollback also fails the health check. What should the script do?"
**Red flags:** `sleep` then a single `curl`. No rollback, or rollback with no verification.
Doesn't capture the current version before deploying. Ignores script exit codes. Infinite loop
with no timeout. Swallows errors.
**Tier notes:** mid — a correct polling loop with timeout + rollback + non-zero exit. senior —
consecutive-healthy checks, rollback-failure handling, check-from-the-user's-side, fleet rollout.
