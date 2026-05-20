---
name: task-discover
description: Discovery phase for new projects. Ask structured questions, confirm understanding, and write docs/BRIEF.md. Mandatory first step before /task-plan on any greenfield project.
---

# Task Discover — Project Discovery

## Overview

This skill runs before `/task-plan`. Its job is to gather everything `/task-plan` needs to produce good specs — product intent, user flows, stack, constraints — through a structured conversation. Output is `docs/BRIEF.md`, a confirmed input document that `/task-plan` reads instead of starting blind.

Do not write any code. Do not suggest implementations yet. But do not stay silent on important observations — if you see a product gap, a missing flow, a decision that will hurt later, or a clearly better approach, say so. The goal is to build a good product, not to just transcribe what the user said.

**On research:** if an answer is ambiguous and has a verifiable answer — look it up before asking the user to decide blind. Don't present options you haven't evaluated. Come back with a recommendation and the reason.

---

## Step 1 — Ask the Core Questions

Ask the user the following questions. Ask all at once — do not drip them one by one.

```
1. What problem does this product solve? Who is it for?
2. What does a successful end-to-end user flow look like? (Walk me through it step by step.)
3. What is the tech stack? (Language, framework, database, hosting.)
4. If AI is involved: which model/provider? What is it doing?
5. What are the hard constraints? (Budget, deadline, integrations, must-haves, must-nots.)
6. What is explicitly out of scope for the first version?
7. Any prior research, references, or decisions already made?
```

Wait for the user's answers before proceeding.

---

## Step 2 — Clarify Gaps and Research Ambiguities

Review the answers. Identify anything that is:
- Ambiguous (two reasonable interpretations exist)
- Missing (an answer was skipped or vague)
- Contradictory (two answers conflict)

**Before asking the user about ambiguous technical decisions** (e.g. which model to use, which library fits best, what the current API supports) — research it first. Look up the latest docs, compare options, form a recommendation. Then present: "I looked into X — I'd recommend Y because Z. Does that work?" Don't make the user decide blind on something you can verify.

Ask only the follow-up questions that are genuinely blocking and cannot be resolved through research.

Wait for answers before proceeding.

---

## Step 2.5 — Proactive Product Observations

Before moving to the summary, look at the full picture from a product quality perspective. Ask yourself:

- Is there a core user flow that was described but has an obvious gap or broken step?
- Is there an edge case that will definitely happen but wasn't mentioned?
- Is there a decision baked into the description that will be painful to undo later?
- Is there a simpler approach that achieves the same outcome?

If you see something important — say it now. One observation at a time, most critical first. Be direct about why it matters. This is the cheapest moment to catch product problems — before specs are written and before code exists.

Do not stay silent to avoid friction. A good product comes from catching issues early.

---

## Step 3 — Summarize Back

Write a summary in the conversation and ask the user to confirm it is correct:

```
## What I Understood

**Product:** [one sentence — what it does and who it's for]
**Core user flow:** [numbered steps of the primary flow]
**Stack:** [language / framework / database / hosting]
**AI role:** [what the model does, which model/provider]
**Constraints:** [hard limits]
**Out of scope (v1):** [what is deferred]
**Prior decisions:** [anything already locked in]
```

**CONFIRMATION GATE — PAUSE HERE.**

Do not write `BRIEF.md` until the user confirms the summary is accurate. If they correct anything, update the summary and confirm again.

---

## Step 4 — Write BRIEF.md

Once confirmed, write `docs/BRIEF.md`:

```markdown
# Project Brief

> Written by /task-discover. Input for /task-plan.
> Update this file if requirements change before planning begins.

---

## Product

[What it does and who it's for.]

## Core User Flow

[Numbered steps of the primary flow.]

## Tech Stack

| Layer | Choice |
|---|---|
| Language | ... |
| Framework | ... |
| Database | ... |
| Hosting | ... |

## AI Pipeline

| Aspect | Detail |
|---|---|
| Model | ... |
| Provider | ... |
| Role | ... |

## Hard Constraints

- ...

## Out of Scope (v1)

- ...

## Prior Decisions

- ...
```

---

## Step 4.5 — Update CLAUDE.md Project-Specific Sections

The brief contains everything needed to fill in the placeholder sections of `CLAUDE.md`. Update these three sections now — do not leave them as "TBD":

**`## Project Overview`** — write a 1–2 sentence product description and the core pipeline diagram:
```markdown
## Project Overview

[What the product does and who it's for — one sentence.]

**Core pipeline:**
```
[Step 1] → [Step 2] → [Step 3] → [Step 4]
```
```

**`## Architecture Non-Negotiables`** — replace the "TBD" placeholder with stack-specific rules derived from the confirmed stack. At minimum:
- Language + runtime constraints (e.g. async-only, typed, etc.)
- Framework conventions that must be followed
- Database rules (migrations, transactions, indexing)
- Hosting/deployment constraints if known

If the stack is not yet confirmed, leave this section as-is — do not invent rules for an unknown stack.

**`## AI Pipeline Rules`** — if the project uses AI:
- Add the specific model and provider to the existing generic rules
- Add any project-specific constraints (e.g. max tokens, response format requirements, cost limits)
- Keep the generic rules (timeout, graceful degradation, no raw errors to users, log all calls, validate before saving)

If no AI is involved, remove this section entirely — don't leave rules that don't apply.

Do not touch: `## Core Rules`, `## Active Milestone`, `## Docs Structure`, `## When in Doubt` — these are universal.

---

## Step 5 — Hand Off

Tell the user:

> "Brief is written to `docs/BRIEF.md` and CLAUDE.md project sections are populated. Run `/task-plan` to convert this into milestones and specs."

Do not invoke `/task-plan` automatically. The user decides when to proceed.

---

## What Good Looks Like

A complete `/task-discover` run produces:
- One confirmed `docs/BRIEF.md`
- `CLAUDE.md` with all project-specific sections filled in (no "TBD" placeholders)
- No ambiguities that would force `/task-plan` to stop and ask questions
- The user knows exactly what comes next
