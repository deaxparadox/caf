# CLAUDE.md — Project Constitution

> This file is read before doing anything in this project.
> It is the source of truth for how this project is built and maintained.

---

## How to Work on This Project

**All work goes through a skill workflow — not freeform implementation.**

| Task type | Skill |
|---|---|
| Starting after any break or context reset | `/task-resume` |
| Greenfield project — nothing defined yet | `/task-discover` |
| New idea, feature set, or significant change | `/task-plan` |
| Implement an approved milestone | `/task-execute` |
| Fix a bug or reported issue | `/task-fix` |
| Tune extraction prompts | `/task-prompt` |
| End of session state sync | `/sync` |

The typical flow is: `/task-discover` (greenfield only) → `/task-plan` (produces specs) → `/task-execute` (consumes specs) → `/sync`.

See `docs/LIFECYCLE.md` for how all skills, docs, and artifacts connect.

Do not bypass the skills. The workflows exist because freeform implementation breaks things.

---

## Active Milestone

**M01 — TBD** (not yet planned — run `/task-plan` first)

---

## Core Rules

**1. Ask before assuming**
If a requirement is ambiguous, ask before implementing. If the answer is in a spec or ADR — use it.

**2. Search before implementing**
For anything package-specific or model-specific — verify the current API before writing code. Don't build against assumptions that can be checked.

**3. Discuss before building**
If you see a better approach, say so before writing any code.

**4. Don't hold back on product quality**
If you see a gap, a missing flow, a decision that will hurt later, or a better approach — say so. One observation at a time, most critical first. The goal is a good product. Staying silent to avoid friction is not acceptable.

**5. Spec before implementing — mandatory**
`task-plan` writes specs. `task-execute` consumes them. Never implement without a spec.

**6. Log before fixing — mandatory**
Every bug goes in `docs/ISSUES.md` first. `task-fix` enforces this.

**7. Eval before shipping — mandatory for AI pipeline changes**
Any change touching extraction prompts, model calls, or output schema must be evaluated against fixtures. `task-execute` and `task-fix` both enforce this gate.

**8. Resume before working — mandatory after any break**
If this session follows a break or context reset, run `/task-resume` before doing anything else. Derive state from files and git — never assume.

---

## Project Overview

AI-powered OCR application that scans architectural drawings, blueprints, floor plans, and construction documents to automatically extract structured information and generate detailed outputs.

**Core pipeline:**
```
Document ingestion → preprocessing → OCR / vision model → structured extraction → output generation
```

---

## Architecture Non-Negotiables

> To be defined once stack is confirmed. Add here before first implementation.

---

## AI Pipeline Rules

- Every model call must have a timeout — never block indefinitely
- Model failures must degrade gracefully — return a meaningful error, never crash
- Never expose raw model errors to end users
- Log every model call: parameters, response time, token usage, errors
- Prompts are versioned in `docs/MODELS.md` — never change a prompt without logging the version
- Extraction output must be validated against the schema before saving anything

---

## Docs Structure

```
SESSION.md          ← pickup point + current milestone status
CLAUDE.md           ← this file
docs/
  BRIEF.md          ← confirmed project inputs from /task-discover (product, stack, constraints)
  PRD.md            ← product requirements (evolves with milestones)
  PLAN.md           ← all milestones in one file
  specs/            ← one spec per milestone: M01-name.md, M02-name.md
  adr/              ← architectural decisions: ADR-001.md, ADR-002.md
  API.md            ← endpoint reference
  SCHEMA.md         ← canonical extraction output format
  MODELS.md         ← models, prompts (versioned), token costs
  FIXTURES.md       ← test fixture index and eval methodology
  DECISIONS.md      ← D-XXX design decisions
  ISSUES.md         ← bugs (open/closed) + improvements
  DEVELOPER.md      ← patterns, gotchas, conventions
  SETUP.md          ← how to run the project
```

No phase directories. Milestones are labels in PLAN.md and prefixes on spec files — not folders.

---

## When in Doubt

1. Read the spec for the active milestone
2. Read the relevant ADR — has this decision been made?
3. Read SCHEMA.md — does this change affect the output format?
4. If still unclear — ask before implementing
