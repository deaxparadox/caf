---
name: task-audit
description: Sweeps the codebase against engineering principles. Logs every violation as ISS-XXX with severity. Fixes P1 immediately via task-fix, leaves P2/P3 in ISSUES.md. Run when joining an existing codebase, at the end of a milestone set, or when code quality feels like it's drifting.
---

# Task Audit — Engineering Principles Check

## Overview

A full sweep of the codebase (or a targeted area) against the principles in `docs/ENGINEERING-PRINCIPLES.md`. Output is a prioritized list of violations logged in `docs/ISSUES.md`. P1 violations block — they get fixed before anything else proceeds.

Invoke as: `/task-audit` (full sweep) or `/task-audit sessions/` (targeted directory).

---

## Step 0 — Load Principles

Read `docs/ENGINEERING-PRINCIPLES.md` in full before dispatching any sub-agent. The reviewer needs to know exactly what to check — not a vague "follow best practices."

If `docs/ENGINEERING-PRINCIPLES.md` does not exist: stop. Create it first by extracting principles from CLAUDE.md into that file, then proceed.

---

## Step 1 — Sweep

Dispatch a **code-reviewer** sub-agent per principle category (not one massive dispatch — reviewers lose focus with too many concerns at once):

Example dispatch per category:
> "Review all files in [scope] for violations of: [single principle category from ENGINEERING-PRINCIPLES.md]. For each violation found, report: file path, line number, what the violation is, why it violates the principle, severity (P1/P2/P3). Do not suggest fixes — only report violations. P1 = blocks correctness or security. P2 = degrades maintainability. P3 = style/convention."

Categories to sweep (adapt to what's in ENGINEERING-PRINCIPLES.md):
- Import ordering and placement
- Module boundaries (cross-module direct imports)
- N+1 queries (missing select_related / prefetch_related / joins)
- Exception handling (swallowed exceptions, bare except)
- Config and environment (hardcoded values, implicit fallbacks)
- Type hints (missing on public functions and method signatures)
- Layer separation (business logic in wrong layer)
- Logging (missing correlation IDs, missing error context)

Run sweeps in parallel where the categories are independent.

---

## Step 2 — Deduplicate and Prioritize

Consolidate findings from all sub-agents. Remove duplicates (same file + line reported by multiple sweeps).

Assign final severity:
- **P1** — correctness or security risk. Fix before any other work proceeds.
- **P2** — maintainability risk. Fix before the next milestone starts.
- **P3** — convention or style. Fix when touching that file for other reasons.

---

## Step 3 — Log All Violations

Add every violation to `docs/ISSUES.md` under Open Issues:

```
## ISS-XXX — [Principle]: [short description]
**Status:** 🔍 Verified
**Severity:** P1 / P2 / P3
**File:** path/to/file.py:line
**Violation:** [what the code does]
**Principle:** [which principle it violates, from ENGINEERING-PRINCIPLES.md]
```

Group by severity in the log — P1 first.

---

## Step 4 — Fix P1 Violations

For each P1 violation: invoke `task-fix ISS-XXX`.

Do not proceed to Step 5 until all P1 violations are resolved.

---

## Step 5 — Report

Present a summary:

```
Audit complete — [date]
Scope: [full / targeted path]

P1 (fixed): X violations — all resolved
P2 (logged): X violations — fix before next milestone
P3 (logged): X violations — fix on next touch

ISS-XXX through ISS-YYY added to ISSUES.md
```

Then invoke `/sync` to commit the audit findings.

---

## Scope Guidance

| When | Scope |
|---|---|
| Joining existing codebase | Full sweep |
| End of milestone set | Files touched during those milestones |
| Specific area feels wrong | That directory only |
| After a large refactor | Files changed in the refactor |

Do not run a full sweep after every single milestone — it's expensive and produces diminishing returns on recently reviewed code. Targeted sweeps after milestones are handled by `task-execute` Step 3.5.
