# AI-skills

Personal collection of [agent skills](https://docs.claude.com/en/docs/claude-code/skills) for Claude Code.

## Skills

| Skill | What it does |
|-------|--------------|
| [`mock-interview`](skills/mock-interview) | Runs a realistic, live DevOps / SRE / Cloud Engineer mock interview — one scenario at a time with follow-ups, calibrated by topic, level (junior→staff), and company tier (service / product / MANG-FAANG). Ends with a scored rubric, model answers, a prioritized study plan, and a saved session log for tracking progress across sessions. |

## Install

With the [`skills` CLI](https://skills.sh):

```bash
npx skills add https://github.com/gauravs30/AI-skills --skill mock-interview
```

Or manually — copy the skill folder into your skills directory:

```bash
git clone https://github.com/gauravs30/AI-skills.git
cp -r AI-skills/skills/mock-interview ~/.claude/skills/mock-interview
```

Then invoke it in Claude Code with `/mock-interview` (e.g. `/mock-interview kubernetes mid Amazon 5`).

## Layout

```
skills/<name>/
  SKILL.md          entry point (frontmatter + workflow)
  references/        loaded as needed
  assets/           templates used in output
  evals/            test cases (evals.json)
```
