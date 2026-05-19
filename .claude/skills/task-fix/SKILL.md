---
name: task-fix
description: Orchestrates the full bug-fix pipeline. Use for any reported bug, issue, or engineering violation. Enforces: baseline → analyze → log → spec → approval gate → implement → test → eval (if AI pipeline touched) → sync.
---

# Task Fix — Bug Fix Pipeline

## Overview

This skill is the orchestrator. You dispatch focused sub-agents for each step and validate their output before proceeding. Do not skip steps. Do not combine steps.

---

## Pipeline

### Step 0 — Baseline

Run the full test suite. Capture pass/fail count.

```bash
# run tests — command depends on stack (see docs/SETUP.md)
```

If tests are already failing on unrelated things, note them. They are not this bug's responsibility.

**Gate:** Record baseline. Proceed regardless of result (the bug may already be failing tests).

---

### Step 1 — Analyze & Verify Root Cause

Dispatch a **code-explorer** sub-agent:

> "Read the code relevant to [issue description]. Trace the execution path. Identify the root cause. Do NOT suggest fixes — only report what the code actually does and where it diverges from expected behavior. Report: file paths, line numbers, what the code does, what it should do."

**Gate:** Root cause must be verified from actual code — not assumed. If the sub-agent reports uncertainty, dispatch again with more targeted file paths.

---

### Step 1.5 — Log the Issue

Add entry to `docs/ISSUES.md`:

```
## ISS-XXX — [Short description]
**Status:** 🔍 Verified — awaiting spec
**Root cause:** [one sentence from Step 1]
**Files affected:** [list from Step 1]
**Reproduction:** [steps to reproduce]
```

---

### Step 2 — Write Spec

Write fix spec at `docs/phase1/specs/FIX-XXX-[short-name].md`:

```markdown
## ISS-XXX Fix Spec

**Root cause:** (from Step 1)
**Fix approach:** (what will change and why)
**Files to modify:** (list)
**Acceptance criteria:**
- [ ] ...
- [ ] Tests pass
- [ ] Eval passes (if AI pipeline touched)
```

**APPROVAL GATE — PAUSE HERE.**

Show the spec to the user. Do not proceed until explicit approval.

---

### Step 3 — Implement

Dispatch a **general-purpose** sub-agent with:

> "Implement the fix described in [spec path]. Modify only the files listed. Do not refactor surrounding code. Do not add features. Imports at top of file. Return: list of files changed and what changed in each."

**Gate:** Review the changes. If the sub-agent modified files outside the spec scope, reject and re-dispatch with tighter constraints.

---

### Step 4 — Test

Run the full test suite. Compare to Step 0 baseline.

Expected: previously failing tests now pass. No new failures.

If new failures appeared: analyze before proceeding. Do not mark done with regressions.

---

### Step 4.5 — Eval (if AI pipeline touched)

If the fix touched extraction prompts, model calls, schema validation, or output processing:

Dispatch **task-prompt** eval step only (not the full prompt iteration loop):

> "Run eval against all fixtures. Compare to last recorded baseline in docs/FIXTURES.md. Report: per-fixture pass/fail, overall score, any regressions."

**Gate:** No regressions allowed. If eval score dropped, the fix introduced a new problem — do not ship.

---

### Step 5 — Update Docs & Sync

Update `docs/ISSUES.md`: change status to ✅ Fixed, add fix summary.

Then invoke `/sync` to commit everything.

---

## What Sub-Agents You Dispatch

| Step | Agent type | Purpose |
|---|---|---|
| 1 | `code-explorer` | Root cause analysis — read-only |
| 3 | `general-purpose` | Implementation |
| 4.5 | eval script | Pipeline regression check |
