# Question Bank — Linux + Security + Coding

Linux internals & troubleshooting, networking fundamentals, DevSecOps, secrets, light scripting.

Scenario template: see `questions-core-devops.md`.

## Contents
- LSX-1 — "The server is slow" — Linux triage
- LSX-2 — Disk is full but `du` doesn't add up
- LSX-3 — Trace a request that isn't arriving (networking)
- LSX-4 — A container ran as root and now there's a crypto miner
- LSX-5 — Coding: parse a log file (Bash/Python)
- LSX-6 — Coding: a rate limiter
- LSX-7 — Rotate a leaked credential

---

### LSX-1 — "The server is slow" — Linux triage
**Topics:** Linux, performance   **Levels:** junior, mid, senior   **Format:** troubleshooting
**Scenario (say this):** "You get told 'the app server is slow'. You SSH in. Take me through how
you figure out what's wrong, command by command."
**Candidate should clarify:** Slow how — high latency, timeouts, specific endpoint? Since when?
One host or all? Did anything change?
**Strong answer covers:**
- A structured first-60-seconds sweep: `uptime` (load avg trend), `top` / `htop`, `vmstat 1`,
  `free -m`, `df -h`, `iostat -xz 1`, `mpstat -P ALL 1`, `sar`, `dmesg | tail` (OOM killer, disk
  errors), `ss -s`.
- **Interpret, don't just run:** load > cores with high `%us` = CPU-bound; high `%wa` = IO-bound;
  high `%sy` = kernel/syscalls; load high but CPU idle = uninterruptible sleep (D state, usually
  IO or NFS); memory low + swapping = memory pressure; `si/so` in vmstat = swap thrash.
- Narrow to a process: `top` by CPU/MEM, `pidstat`, then `strace -c -p <pid>` / `perf top` for
  CPU, `iotop` for IO.
- Application layer: GC logs, thread dumps, connection pools, slow query log, its own metrics.
- *(senior)* USE method (utilization / saturation / errors for each resource) as the framework.
  Knowing when it's *not* the host — it's a downstream dependency, DNS, a noisy neighbor on shared
  infra, or the client. Checking for the OOM killer and CPU throttling (cgroup limits in
  containers — `nr_throttled`).
**Follow-ups / curveballs:**
- (junior) "Load average is 40 on an 8-core box. What does that tell you, and what's your next
  command?"
- (mid) "`top` shows a process in `D` state using no CPU. What's going on?" (→ uninterruptible
  sleep, blocked on IO — check `iostat`, `iotop`, NFS mounts, `cat /proc/<pid>/stack`)
- (senior) "CPU is 30%, memory fine, disk fine, but the app is still slow. Where now?" (→ network,
  downstream latency, DNS, lock contention inside the app, cgroup CPU throttling despite low
  average, TCP retransmits)
**Red flags:** Reboots first. Runs `top` and stops. Can't interpret load average or `%wa`. Doesn't
know the OOM killer or `dmesg`. No framework, just random commands.
**Tier notes:** service — the command sweep + basic interpretation. product — USE method, D-state,
"it's not the host". MANG — cgroup throttling nuance, `perf`/`strace` fluency, hypothesis-driven
narrowing, ruling out the platform.

---

### LSX-2 — Disk full but `du` doesn't add up
**Topics:** Linux, filesystems   **Levels:** mid, senior   **Format:** troubleshooting
**Scenario (say this):** "`df -h` says `/` is 100% full. You run `du -sh /*` and it adds up to
about 40% of the disk. Where's the rest, and how do you fix it safely?"
**Candidate should clarify:** Is the box still serving? What's on `/` — logs, docker, a database?
**Strong answer covers:**
- **Deleted-but-open files:** a process still holds a file descriptor to a file that's been
  `rm`'d; space isn't freed until the fd closes. `lsof +L1` or `lsof | grep deleted` to find it;
  fix by restarting/HUPing the process or truncating via `/proc/<pid>/fd/<n>`. Classic cause: a
  log file rotated by `rm` instead of `copytruncate`/`logrotate`, with the writer still running.
- **Inodes exhausted:** `df -i` — millions of tiny files (mail spool, session files, cache) fill
  the inode table while bytes are fine.
- **Mount hiding data:** files written to a directory *before* a filesystem was mounted over it —
  they're consuming space on the parent but hidden. Check with a bind mount or `mount --bind / /mnt`.
- **`du` vs `df` legit differences:** `du` needs permissions to traverse everything (run as root),
  sparse files, filesystem reserved blocks (~5% for root), block size rounding.
- Safe fix: identify first, never blind-delete on a prod box; truncate logs rather than delete;
  clear package caches / old kernels / docker (`docker system prune`) as safe wins; add monitoring
  + logrotate so it doesn't recur.
**Follow-ups / curveballs:**
- (mid) "`lsof | grep deleted` shows a 30 GB file held by your app. You can't restart the app
  right now. Options?" (→ `: > /proc/<pid>/fd/<n>` to truncate in place, or `truncate`; understand
  the risk)
- (senior) "`df -i` shows inodes at 100%. `find` to locate the offender is itself taking forever.
  Why, and what do you do?" (→ traversing millions of files; target likely dirs; `find <dir>
  -xdev -type f | wc -l` per subtree)
**Red flags:** `rm -rf` something to "make space" without diagnosing. Never heard of deleted-open
files or inode exhaustion. Doesn't know `df -i`. Restarts the box hoping it helps (it might, which
hides the lesson).
**Tier notes:** service — knows deleted-open-files and `df -i`. product — the mount-hiding case,
safe in-place truncation, logrotate prevention. MANG — all of it fluently, plus the risk analysis
of each remediation on a live box.

---

### LSX-3 — Trace a request that isn't arriving
**Topics:** networking   **Levels:** mid, senior   **Format:** troubleshooting
**Scenario (say this):** "Service A can't reach Service B on port 8443. From A, `curl https://B:8443`
hangs and times out. Both are Linux hosts in the same cloud VPC. Debug it with me, layer by
layer."
**Candidate should clarify:** Same subnet / AZ? Did it ever work? Is B actually listening? DNS or
IP?
**Strong answer covers:**
- **Bottom-up or top-down, but systematic:**
  - DNS: `dig B` / `getent hosts B` — resolving to the right address?
  - L3 reachability: `ping B` (may be blocked, not definitive), `traceroute`/`mtr`.
  - L4 to the port: `nc -vz B 8443` / `curl -v` — connection refused (nothing listening / firewall
    reject) vs. timeout (packet dropped silently — security group, NACL, routing).
  - On B: `ss -tlnp | grep 8443` — is the process listening, and on `0.0.0.0` or just `127.0.0.1`?
  - Firewall both ends: cloud security groups / NACLs, host `iptables`/`nftables`/`ufw`.
  - `tcpdump -ni any port 8443` on both ends — do SYNs leave A? arrive at B? does B SYN-ACK? is
    the ACK lost? This pinpoints the drop.
- Interpretation: SYN sent, nothing back = dropped inbound to B (SG/NACL/route). SYN-ACK sent by B
  but A never sees it = dropped on the return path. RST = something actively refusing.
- *(senior)* MTU / path MTU black holes (connection opens, hangs on larger payloads — TLS
  handshake). Asymmetric routing. Conntrack table full. TLS-layer issues once TCP is fine
  (`openssl s_client`).
**Follow-ups / curveballs:**
- (mid) "`nc -vz` says 'connection refused' immediately. What does that rule out, and what's
  likely?" (→ packet is reaching a host that's sending RST; not a silent drop — so routing/SG
  inbound is fine; B isn't listening on that interface, or a host firewall is REJECTing)
- (senior) "TCP connects fine, TLS handshake starts then hangs. Small requests work, large ones
  don't. What is this?" (→ MTU / PMTUD black hole; ICMP 'frag needed' being dropped)
**Red flags:** Only tries `ping` and gives up. Doesn't know `ss`/`netstat` to check listeners.
Can't distinguish "refused" from "timed out". Never reaches for `tcpdump`. Doesn't check whether
B binds to localhost only.
**Tier notes:** service — DNS→ping→port→listener→firewall checklist. product — tcpdump on both
ends, refused-vs-timeout interpretation. MANG — MTU black holes, conntrack, asymmetric routing,
reads a packet capture confidently.

---

### LSX-4 — Container ran as root, now there's a miner
**Topics:** container security, incident   **Levels:** mid, senior, staff   **Format:** deep-dive
**Scenario (say this):** "Your monitoring flags one Kubernetes node pegged at 100% CPU. You find a
process that's a crypto miner, launched from inside a pod that runs as root. Walk me through your
response and then how you prevent it."
**Candidate should clarify:** Is this prod? What's the pod — a known service or something
unexpected? Any data-access concern or just compute theft?
**Strong answer covers:**
- **Respond (contain → eradicate → recover):**
  - Isolate: cordon the node, `NetworkPolicy` / security-group the pod, don't just `kill` — you
    lose forensics. Snapshot the node / capture the pod filesystem and process list first if
    feasible.
  - Scope: how did they get in? Exposed dashboard, SSRF, a vulnerable dependency (RCE), a leaked
    kubeconfig, a supply-chain'd image. Check other nodes/pods for the same.
  - Eradicate: kill the pod, roll the node, rotate any credentials the pod could reach (its
    ServiceAccount token, node IAM role, secrets it mounted, anything in its namespace).
  - Assess data exposure — the SA token + node role define the blast radius; what could that
    identity read?
  - Postmortem, disclosure if customer data was in reach.
- **Prevent:**
  - Pod Security: `runAsNonRoot`, drop all capabilities, read-only root FS, no privilege
    escalation, seccomp `RuntimeDefault`. Enforce with Pod Security Admission / a policy engine
    (Kyverno / Gatekeeper).
  - Least-privilege ServiceAccounts; disable auto-mount of the SA token where not needed; scope
    node IAM tightly (IRSA / Workload Identity so pods don't inherit the node role).
  - Image provenance: scan images, pin digests, admission control to only run signed images from
    your registry, minimal base images.
  - Network egress policy (miners need to phone home to a pool) — default-deny egress, allowlist.
  - Runtime detection (Falco / cloud equivalent) for exec-into-pod, crypto-mining signatures,
    unexpected outbound.
- *(staff)* Defense in depth argument — no single control; the root cause (the RCE / exposure) vs
  the amplifiers (root, token, egress). Org rollout of PSA baseline→restricted, the migration
  pain, exceptions process.
**Follow-ups / curveballs:**
- (mid) "The pod's ServiceAccount had `cluster-admin`. What's your blast radius now?" (→ whole
  cluster: read every secret, create workloads, likely pivot to cloud via node role — treat as
  full cluster compromise)
- (staff) "You want to enforce 'no root containers' cluster-wide but 30 legacy services need it.
  How do you roll it out?" (→ audit mode first, per-namespace enforcement, fix the offenders,
  time-boxed exceptions with owners, then flip to enforce)
**Red flags:** `kill -9` the miner and calls it done. Doesn't rotate credentials. Doesn't ask how
they got in. Never heard of Pod Security Admission or seccomp. No egress control concept. Doesn't
grasp that the SA token is the real risk.
**Tier notes:** mid — contain-without-destroying-forensics + PSA/non-root + scanning. product —
credential rotation, egress policy, runtime detection, root-cause vs amplifier. MANG — full IR
structure, IRSA/Workload Identity, org-wide policy rollout, blast-radius reasoning from the
identity.

---

### LSX-5 — Coding: parse a log file
**Topics:** scripting, coding   **Levels:** junior, mid, senior   **Format:** coding
**Scenario (say this):** "Here's an nginx-style access log. Each line has an IP, a timestamp in
brackets, an HTTP method and path in quotes, a status code, and a byte count. Write something that
prints the top 5 IPs by number of requests that resulted in a 5xx status. Bash or Python, your
call. Talk me through it."
**Candidate should clarify:** Rough file size (fits in memory / streaming)? One-off or reusable?
Malformed lines — skip or error? Exact log format / a sample line.
**Strong answer covers:**
- A streaming approach (don't load 10 GB into memory): read line by line, filter 5xx, count per IP
  in a dict/`Counter`, then top 5.
- Python: `collections.Counter`, `str.split` or a regex; `Counter.most_common(5)`. Handle the
  bracket/quote fields correctly. Guard against malformed lines (try/except or a length check),
  don't crash the whole run on one bad line.
- Bash: `awk '$9 ~ /^5/ {print $1}' | sort | uniq -c | sort -rn | head -5` — knows which field is
  which, or uses a more robust parse; understands this is fine for moderate sizes.
- Talks about: correctness of the field parsing (quoted path can contain spaces), 5xx as
  `500 <= code <= 599` not "starts with 5" if codes could be weird, ties in the top 5,
  empty/–  byte counts.
- *(senior)* Complexity (O(n) time, O(unique IPs) memory), testability (factor the parse into a
  function), what changes if it needs to run continuously (tail -F, windowing), or across many
  files (glob, gzip).
**Follow-ups / curveballs:**
- (junior) "Walk me through your parsing of one line. What if the path has a space in it?"
- (mid) "Now do it for a 200 GB file. Does your approach still work?" (→ streaming already does;
  discuss parallelism, `LC_ALL=C sort`, external sort, or a chunked map-reduce)
- (senior) "Make it a reusable function and tell me how you'd test it." (→ pure parse function,
  fixture lines including malformed ones, assert on the counter)
**Red flags:** `file.read().splitlines()` on an arbitrarily large file. Regex that backtracks
catastrophically. Crashes on a malformed line. Off-by-one in field indexing and doesn't verify
with the sample. Can't explain their own one-liner.
**Tier notes:** junior — a working script + can explain it. mid — streaming, malformed-line
handling, the 200 GB question. MANG — clean factoring, complexity analysis, testing strategy,
correctness edge cases (quoted spaces, code ranges, ties).

---

### LSX-6 — Coding: a rate limiter
**Topics:** coding, systems   **Levels:** mid, senior, staff   **Format:** coding
**Scenario (say this):** "Implement a rate limiter: a function `allow(client_id) -> bool` that
returns True if the client is allowed to make a request now, given a limit of N requests per
window of W seconds. Start single-process, in-memory. Talk through your design first."
**Candidate should clarify:** Fixed window or sliding? Exact enforcement or approximate OK?
Single process now — will it need to be distributed? Concurrency (threads)? What N and W?
**Strong answer covers:**
- Names the algorithms and tradeoffs:
  - **Fixed window counter** — simplest; boundary burst problem (2N in a short span across the
    boundary).
  - **Sliding window log** — exact; stores a timestamp per request, O(N) memory per client,
    prune old ones.
  - **Sliding window counter** — weighted blend of current+previous window; good approximation,
    cheap.
  - **Token bucket** — smooth rate + allows bursts up to bucket size; `tokens += elapsed * rate`,
    capped; the usual production choice.
  - **Leaky bucket** — smooths output, queues.
- Picks one (token bucket or sliding-window-counter) and implements it cleanly: per-client state
  in a dict, compute on read using elapsed time (don't run a background thread), handle first-seen
  client.
- Concurrency: a lock around the per-client update, or per-client locks; note the race.
- Memory: clients accumulate — need TTL eviction / LRU so it doesn't grow forever.
- *(senior/staff)* Distributed version: move state to Redis (`INCR` + `EXPIRE` for fixed window,
  or a Lua script for atomic token-bucket), the accuracy/latency/availability tradeoffs, what
  happens if Redis is down (fail open or closed?), clock skew across nodes, and hot-key problems
  for a heavy client.
**Follow-ups / curveballs:**
- (mid) "Why is the fixed-window counter not great? Draw me the failure."
- (senior) "Now it needs to work across 20 API servers. What changes?"
- (staff) "Redis round-trip per request adds 1ms to every call and Redis is now a SPOF. How do you
  mitigate?" (→ local token bucket with periodic sync / borrowing, approximate limits, sharding,
  fail-open with a local fallback limit)
**Red flags:** Background thread per client to refill tokens. No locking, doesn't notice the race.
Unbounded client map. Can't name more than one algorithm. Sliding window log for a huge N without
seeing the memory cost. Doesn't consider the Redis-down case.
**Tier notes:** mid — one correct algorithm implemented, knows fixed-window's flaw. senior —
token bucket, eviction, the distributed sketch. MANG — multiple algorithms with crisp tradeoffs,
atomic Redis implementation, fail-open/closed reasoning, hot-key and skew.

---

### LSX-7 — Rotate a leaked credential
**Topics:** security, incident, secrets   **Levels:** junior, mid, senior   **Format:** troubleshooting
**Scenario (say this):** "A developer accidentally committed an AWS access key to a public GitHub
repo and pushed. They notice 20 minutes later. What do you do, in order?"
**Candidate should clarify:** Is it still live? What did the key have access to? Is there
CloudTrail? Just committed, or in an old commit / already in history?
**Strong answer covers:**
- **Assume it's compromised the moment it hit a public repo** — bots scrape within seconds/minutes.
- Order of operations:
  1. **Deactivate/delete the key** immediately (IAM console/CLI) — this is the containment.
  2. Create a replacement key, update the legitimate consumers (pipeline, app, whoever used it).
  3. **Investigate use:** CloudTrail for that access key ID — any calls you don't recognize?
     Resource creation (EC2 for mining, IAM users/keys for persistence), data access (S3 List/Get),
     new roles/policies.
  4. If abused: broader IR — look for persistence (new IAM principals, modified trust policies,
     Lambda backdoors), assess data exposure, possibly quarantine the account, involve security.
  5. Remove the secret from git history (BFG / `filter-repo`) and force-push — but understand this
     is *cleanup, not containment*; the key must already be dead because the history was public.
  6. Notify per policy; postmortem.
- **Prevent:** pre-commit secret scanning (gitleaks / trufflehog), server-side push protection
  (GitHub secret scanning), no static keys at all — use IAM roles / OIDC / SSO short-lived creds,
  least privilege so a leaked key is low-value, alerting on anomalous API use.
**Follow-ups / curveballs:**
- (junior) "The dev's instinct is to force-push to remove the commit first. Why is that the wrong
  first move?" (→ doesn't contain anything; the key was already scraped; wastes the critical
  minutes; deactivate first)
- (mid) "CloudTrail shows the key launched 50 GPU instances in 4 regions 8 minutes after the push.
  Now what?" (→ full IR: kill the instances, check for IAM persistence, rotate everything the key
  touched, budget/abuse ticket to AWS, assess data access, incident + possible disclosure)
- (senior) "How do you make committed secrets structurally impossible here?"
**Red flags:** Force-pushes to remove the commit as step 1. Doesn't check CloudTrail for actual
abuse. "Just rotate it" without checking what happened. No prevention beyond "be careful". Thinks
scrubbing git history undoes the exposure.
**Tier notes:** junior — deactivate-first, then rotate, then check usage. mid — CloudTrail
investigation, persistence hunting, the abuse scenario. senior — structural prevention (no static
keys), least-privilege blast-radius reduction, org-wide push protection.
