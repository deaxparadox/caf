---
name: task-plan
description: Converts a raw idea into a structured implementation plan. Produces theory analysis, milestone breakdown, and per-milestone spec files. Output is consumed by task-execute. Use before any new feature set or significant architectural change.
---

# Task Plan — Idea to Specs

## Overview

This skill is pure thinking — no code written. The output is a set of spec files that `task-execute` can consume. A good plan here means execution can run with minimal interruptions.

---

## Step 0 — Read BRIEF.md (if it exists)

Before anything else, check if `docs/BRIEF.md` exists.

- **If it exists:** Read it. This is the confirmed output of `/task-discover`. Use it as the source of truth for product intent, stack, constraints, and scope. Do not ask the user to repeat information already in the brief.
- **If it does not exist:** Proceed, but note that greenfield projects should have run `/task-discover` first. Ask the user for the minimum needed: what to build, the tech stack, and the primary user flow.

---

## Step 1 — Theory Analysis

Before breaking anything into tasks, understand what the idea actually is.

Dispatch a **code-explorer** sub-agent (if codebase exists) or answer these directly (if greenfield):

> "Given this idea: [idea description] — analyze: (1) what problem does this solve, (2) what does it touch in the existing codebase, (3) what are the unknowns or risks, (4) what assumptions are being made that need to be validated. Do NOT suggest implementation. Report only analysis."

Write a short theory summary in the conversation:

```
## Theory
Problem: ...
Touches: ...
Unknowns: ...
Assumptions: ...
Constraints: ...
```

**On research:** if an unknown has a verifiable answer — library docs, model capabilities, API constraints — look it up before surfacing it as an unknown. Come back with a finding, not just a question. Don't make the user answer something you can verify yourself.

**On product observations:** beyond technical unknowns, look at the idea from a product quality angle. If the described approach has a gap that will produce a worse product, a missing flow, a decision that will be expensive to undo, or a simpler architecture that achieves the same goal — say so now. This is the last moment before milestone boundaries are set. Raising it after specs exist is expensive.

**Gate:** Do not proceed to milestone breakdown until the theory is clear and any important product observations have been raised. If there are unresolved unknowns, surface them to the user now — not after specs are written.

---

## Step 2 — Milestone Breakdown

Break the idea into milestones. Rules:

- Each milestone delivers something runnable and testable — no "setup" milestones that produce nothing observable
- Milestones are ordered by dependency — if M02 needs M01's models, M01 comes first
- Each milestone is independently completable — a milestone that requires two other milestones to be half-done is not a milestone, it's a task
- Scope each milestone to ~1–3 days of work. If larger, split it.

Present the milestone list to the user:

```
M01 — [name]: [one sentence: what it builds and what it enables]
M02 — [name]: [one sentence]
M03 — [name]: [one sentence]
...
Dependencies: M02 requires M01. M03 can run parallel with M02.
```

**APPROVAL GATE — PAUSE HERE.**

User must approve the milestone breakdown before specs are written. Changing a milestone boundary after specs exist is expensive.

---

## Step 3 — Specs Per Milestone

For each approved milestone, write a spec file at `docs/specs/[MXX]-[short-name].md`:

```markdown
# [MXX] — [Feature Name]

## What This Builds
[2-3 sentences. What exists after this milestone that didn't before.]

## What This Does NOT Build
[Explicit scope boundary. What is deferred to a later milestone.]

## Design Decisions
[For each non-obvious choice: Option A vs B, chose A because...]

## Dependencies
[Which prior milestones must be complete. Which external services are needed.]

## Files to Create / Modify
- `path/to/file` — what changes and why

## API Changes
[New endpoints, changed response shapes. Also update docs/API.md when implemented.]

## Schema Changes
[If extraction output format changes. Also update docs/SCHEMA.md when implemented.]

## Task Breakdown
- [ ] Task 1
- [ ] Task 2
- [ ] Task 3

## Acceptance Criteria
- [ ] [Specific, testable outcome]
- [ ] All existing tests still pass
- [ ] Eval passes (if AI pipeline involved)
```

Write all specs before showing them to the user. Then present the full set for review.

**APPROVAL GATE — PAUSE HERE.**

User must approve all specs. After approval, update `docs/PLAN.md` with the new milestones and update `docs/PRD.md` if the idea expands the product scope.

---

## Step 4 — Update Tracking Docs

After spec approval:

**docs/PLAN.md** — add milestone rows:
```
| M01 | [name] | Not started |
| M02 | [name] | Not started |
```

**docs/PRD.md** — add or update scope section if idea expands the product.

**CLAUDE.md** — update `Active Milestone` line to M01 (or whichever is first).

Then invoke `/sync` to commit the plan.

---

## What Good Looks Like

A complete `task-plan` run produces:
- Theory summary (conversation only — not a file)
- Milestone list approved by user
- One spec file per milestone in `docs/specs/`
- Updated `docs/PLAN.md`
- A commit

`task-execute` can pick up any milestone from this point without asking clarifying questions.
