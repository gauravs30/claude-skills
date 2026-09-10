# Rubric

Use this for the post-interview analysis. Score each question on all five dimensions, then give an
overall score per dimension and a single verdict.

## The five dimensions

### 1. Technical depth & correctness
Is what they said *accurate*, *current*, and *deep enough for the level*? Do they understand
mechanisms, not just names? Penalize confident wrong statements harder than "I'm not sure".

### 2. Structured problem-solving
Before diving in, do they clarify scope and constraints, state assumptions, and decompose the
problem? Do they drive methodically rather than free-associating? Do they manage scope out loud
(especially important at MANG)?

### 3. Tradeoff & judgment
Do they weigh 2+ options with real pros/cons? Do they know when to use what? Do they factor cost,
operational burden, team skill, time-to-value — not just technical elegance? Do they make a
recommendation and own it?

### 4. Production & reliability mindset
Unprompted, do they reason about failure modes, blast radius, rollback/recovery, observability
(how would you know it's working / broken), security and secrets, data safety, and the on-call
human at 3am? This is the dimension that most separates real operators from the theoretically
knowledgeable.

### 5. Communication
Concise and signposted. Adjusts depth to the audience. Answers the question actually asked.
Collaborates with the interviewer (checks in, incorporates the curveball) rather than lecturing.
Handles being interrupted or redirected gracefully.

## The 1–5 scale (level-relative)

The anchors are **relative to the configured level**. A "3" for a junior and a "3" for a staff are
very different absolute answers.

| Score | Meaning |
|-------|---------|
| **5 — Exceptional** | Well above the bar for this level. The kind of answer you'd quote to the panel. |
| **4 — Above bar** | Clearly exceeds what's expected at this level on this dimension. |
| **3 — Meets bar** | Solid, expected answer for this level. No serious gaps. |
| **2 — Below bar** | Noticeable gap. Right direction but shallow, or missing something important, or one clear error. |
| **1 — No signal / wrong** | Couldn't engage, or fundamentally incorrect, or hand-wavy with no substance. |

Use `pass` = 1 for that question/dimension. A question where a `hint` was needed caps that
question's *structure* and *depth* at 3 unless they recovered impressively.

## Verdict mapping

Average the dimension scores, but **judgment overrides arithmetic** — a disqualifying error (e.g.
a security answer that would leak credentials, an incident approach that would extend an outage)
caps the verdict at Lean Hire or below regardless of average.

| Mean dimension score | Baseline verdict | Notes |
|----------------------|------------------|-------|
| ≥ 4.3 | **Strong Hire** | Consistently above bar; would advocate strongly. |
| 3.6 – 4.2 | **Hire** | Meets the bar comfortably, some standout areas. |
| 3.0 – 3.5 | **Lean Hire** | Meets the bar but thin; a weak area to shore up. |
| < 3.0 | **No Hire** | One or more dimensions clearly below bar for the tier/level. |

State the verdict against the **specific tier and level configured** — "No Hire for a MANG staff
loop" can coexist with "would be a Hire for a product senior loop", and it's useful to say so when
true.

## Writing the debrief

- **Cite specifics.** "On Q3 you said etcd stores pod logs — it stores cluster state; logs are on
  the node" beats "brush up on Kubernetes internals".
- **Separate knowledge gaps from skill gaps.** Not knowing a fact is fixable with reading. Not
  clarifying scope before designing is a habit that needs deliberate practice.
- **Make the study plan finite and ordered.** 3–6 items, highest-impact first, each with a
  concrete drill and a self-check.
- **Be honest about the verdict.** The candidate is using this to decide where to apply and how
  much more to prep. A kind lie costs them weeks.
