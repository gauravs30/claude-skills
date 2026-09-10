# Company Tiers

The tier changes what the interview screens for and how the loop feels. Use it when resolving the
config and when deciding what to probe. Map real company names to a tier as described in SKILL.md.

## `service` — IT services, consultancies, system integrators

Examples of the type: large IT-services firms, staffing/consulting shops, MSPs, agencies. Roles
are often client-billed and stack-varied.

**What they screen for**
- **Breadth** across common tools — can you be dropped onto a client's stack (Jenkins *or* GitLab
  CI, Ansible *or* Chef, one of the big three clouds) and be productive?
- **Process awareness** — change management, ticketing, ITIL-ish workflows, runbooks, handoffs.
- **Toil automation** — you see a manual, repeated task and you script it.
- **Communication** — you can explain a technical situation to a non-expert client stakeholder
  without drama.
- **Reliability of delivery** — you finish things, document them, and don't leave surprises.

**What they probe less**
- Deep system design at scale, capacity math, first-principles internals.

**Interview texture:** steady, checklist-friendly, lots of "have you used…", "what would you do
if…", "how would you explain this to the client". Curveballs are rare.

**"Meets the bar" =** competent operator who automates with common stacks and communicates cleanly.

## `product` — mid-to-large product companies

Examples of the type: established SaaS companies, fintechs, scale-ups past Series C, most
non-FAANG tech companies with their own product and infra.

**What they screen for**
- **Depth in what you've run** — real ownership of a pipeline, a cluster, an observability stack.
  They will go deep on your actual experience.
- **Tradeoff reasoning** — why this datastore, why this deploy strategy, why this abstraction.
- **Incident credibility** — a real postmortem story, told with specifics and honest lessons.
- **Infra design at their scale** — "design our CI/CD", "design our logging pipeline", "how would
  you cut our AWS bill 30%". Bounded, practical, not planet-scale.
- **Ownership & collaboration** — behavioral signal on driving cross-team work.

**Interview texture:** moderate pressure, conversational, persistent "why" and "what did *you* do".
Expects you to have opinions and defend them.

**"Meets the bar" =** owns a domain end-to-end, makes defensible tradeoffs, has operated real
production systems and can prove it.

## `MANG` — FAANG-tier and peers

Examples of the type: Meta, Amazon, Apple, Netflix, Google, Microsoft, and companies that
explicitly benchmark their bar to them (some top-tier infra startups, HFT firms, etc.).

**What they screen for**
- **First-principles depth** — not "I used X" but "here's how X works and why, and here's what I'd
  do without it".
- **Infra system design at scale** — multi-region, millions of RPS, quantified: capacity math,
  error budgets, failure-domain analysis, migration strategy. Expect a dedicated design round.
- **Coding round** — Bash/Python for automation and data-munging, plus algorithmic-lite problems
  (parse logs, rate-limit, build a small scheduler). Clean, correct, tested-in-your-head code.
- **Structured communication under time pressure** — you scope the problem out loud, state
  assumptions, drive to a recommendation, and manage the clock yourself.
- **Behavioral "raise the bar" signal** — Amazon LP-style / Google "Googleyness": ambiguity,
  conflict, scaling yourself through others, high standards.

**Interview texture:** higher pressure, tight time, interviewer interrupts to redirect ("we have
ten minutes, give me the headline and one deep dive"). They will push until they find your limit —
that's the point, not a sign you're failing.

**"Meets the bar" =** reasons from fundamentals, designs and quantifies at scale, codes cleanly,
communicates with structure under pressure, and shows evidence of raising the bar around them.

## Using tier while probing

- `service`: if they give a deep first-principles answer, acknowledge and move on — don't spend
  the session there. Check breadth instead ("and if the client used GitLab CI instead?").
- `product`: anchor on their real experience; "you said you ran the cluster — walk me through an
  upgrade that went wrong".
- `MANG`: always push one level past their comfortable answer; ask for the number; ask "what
  breaks first at 10x?"; if they don't scope the problem, dock structure and note it.
