---
name: task-execute
description: Implements a single milestone using TDD. Reads the spec, writes failing tests, implements in chunks, runs tests, fixes or escalates. Designed to run with minimal interruptions when the spec is clear.
---

# Task Execute — Milestone Implementation

## Overview

This skill consumes a spec produced by `task-plan`. The spec is the contract — implementation follows the spec, not the other way around. If the spec is wrong, the skill pauses and surfaces it rather than adapting around it.

Invoke as: `/task-execute M01` (or whichever milestone is active).

---

## Before Starting

Read:
1. `docs/specs/[MXX]-[name].md` — the spec for this milestone
2. `docs/PLAN.md` — confirm this milestone's dependencies are marked complete
3. `CLAUDE.md` — confirm active milestone matches what you're about to build

If the spec file does not exist: stop. Run `task-plan` first.
If dependencies are not complete: stop. Complete them first.

---

## Step 0 — Baseline

Run the full test suite. Record pass/fail count. This is the before-state — all currently passing tests must still pass after implementation.

```bash
# see docs/SETUP.md for test command
```

---

## Step 1 — Understand the Codebase

Dispatch a **code-explorer** sub-agent:

> "Read the files listed in [spec path] under 'Files to Create/Modify'. Understand the existing patterns in those files and their neighbors. Report: how existing code is structured, what patterns to follow, any constraints not mentioned in the spec. Do NOT suggest implementation."

This prevents the implementation sub-agent from inventing patterns that don't fit the codebase.

---

## Step 2 — Write Failing Tests

Before any implementation code, write tests that will pass once the milestone is complete.

Tests must:
- Cover every acceptance criterion in the spec
- Cover the main error cases
- Fail right now (implementation doesn't exist yet)
- Not break any existing passing tests

```bash
# run tests — new tests must fail, existing must pass
```

**Gate:** If new tests pass before any implementation, either the feature already exists or the tests are testing nothing. Investigate before proceeding.

---

## Step 3 — Implement

Work through the spec's task breakdown. Dispatch a **general-purpose** sub-agent per logical chunk — not one massive dispatch:

> "Implement [task N from spec] as described in [spec path]. Follow the patterns reported in Step 1. Imports at top of file. Do not implement anything outside this task. Return: files changed and what changed."

After each chunk: run affected tests. Catch regressions early.

---

## Step 3.5 — Principles Check (targeted)

Dispatch a **code-reviewer** sub-agent scoped only to the files modified this milestone:

> "Review these files: [list of files changed in Step 3]. Check for violations of the principles in `docs/ENGINEERING-PRINCIPLES.md`. Report each violation with: file path, line number, which principle, severity (P1/P2/P3). Do not suggest fixes — only report."

**P1 violations (correctness/security risk):** fix before running the full test suite. Log in `docs/ISSUES.md` as ISS-XXX and invoke `task-fix`.

**P2/P3 violations:** log in `docs/ISSUES.md` as ISS-XXX and continue. They do not block this milestone.

If `docs/ENGINEERING-PRINCIPLES.md` does not exist: skip this step and note it in the sync commit message.

---

## Step 4 — Run Full Tests

Run the full test suite.

**Three outcomes:**

### A — All tests pass
Proceed to Step 5.

### B — Tests fail, progress is being made
Iteration count < 3: analyze the failure, dispatch a targeted fix, re-run. Go back to Step 3.

Iteration count = 3 with no progress: go to Outcome C.

### C — Tests fail, no progress after 3 iterations OR regression in existing tests
This is a bug, not an implementation gap. Invoke `task-fix` with the failing test output as the issue description. After `task-fix` resolves it, resume here.

---

## Step 4.5 — Spec Violation Check

If during implementation you discover:
- The spec describes something that conflicts with how the codebase actually works
- A required dependency doesn't exist as described
- An acceptance criterion cannot be met as written

**PAUSE. Do not adapt the implementation to work around it.**

Surface the specific gap to the user:
```
Spec gap found in [MXX]:
- Spec says: [what the spec describes]
- Reality: [what the codebase actually has]
- Recommendation: [one sentence on how to resolve]
```

Wait for direction. This may require a spec revision via `task-plan`.

---

## Step 4.6 — Eval (if AI pipeline involved)

If the milestone touches extraction prompts, model calls, document processing, or output schema:

Run eval against all fixtures. Compare to last baseline in `docs/FIXTURES.md`.

No regressions allowed. If eval score dropped: treat as a failing test — invoke `task-fix`.

If new extraction types were added: add fixtures (minimum 3) and record new baseline.

---

## Step 5 — Mark Complete & Sync

Update `docs/PLAN.md`: mark milestone `[x]` complete.

Update `CLAUDE.md`: advance `Active Milestone` to the next milestone.

Update domain docs if changed:
- `docs/API.md` — if new/changed endpoints
- `docs/SCHEMA.md` — if output schema changed
- `docs/MODELS.md` — if new prompts or model config

Then invoke `/sync` to commit everything.

---

## Iteration Budget

| Situation | Action |
|---|---|
| Tests fail, < 3 iterations | Analyze, fix, re-run |
| Tests fail, 3 iterations, no progress | Invoke `task-fix` |
| Spec is wrong | Pause, surface gap, wait for direction |
| Eval regression | Invoke `task-fix` |
| Dependency missing | Stop, complete dependency first |

Do not exceed the iteration budget by rationalizing "one more try." Three iterations without progress means the problem is structural, not incremental.
