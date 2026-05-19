---
name: task-prompt
description: Orchestrates prompt engineering iteration. Eval-driven: baseline → analyze failure cases → propose change → approval gate → apply → eval → compare → decide. Use whenever extraction quality needs to improve.
---

# Task Prompt — Prompt Engineering Pipeline

## Overview

Prompt changes are not free — a change that improves one extraction type can degrade another. This skill enforces an eval-first loop so every prompt change is measured, not guessed.

Never change a prompt without running this workflow.

---

## Pipeline

### Step 0 — Baseline Eval

Run eval against all current fixtures using the active prompt version.

```bash
# see docs/SETUP.md for eval command
```

Record the baseline score in `docs/FIXTURES.md` if not already recorded for this prompt version.

**Gate:** You must have a numeric baseline before proposing any change. If eval has never been run, run it now and record it before proceeding.

---

### Step 1 — Analyze Failure Cases

Dispatch a **code-explorer** sub-agent:

> "Read the eval results from the last run and the fixture expected outputs in tests/fixtures/expected/. Identify patterns in the failures: which field types are extracted incorrectly, which document types perform worst, what the model seems to misunderstand. Read the current prompt in docs/MODELS.md. Report: the top 3 failure patterns and what in the prompt likely causes each."

**Gate:** You need a clear hypothesis before changing anything. Vague analysis produces vague changes. Push back if the sub-agent's report is not specific.

---

### Step 2 — Propose Prompt Change

Based on Step 1, write the proposed new prompt version here in the conversation.

Show:
- What changed (diff-style if possible)
- Which failure pattern it targets
- What regression risk exists (which other extraction types might be affected)

**APPROVAL GATE — PAUSE HERE.**

User must approve the proposed change before it is written to any file.

---

### Step 3 — Apply Change

Write the new prompt to `docs/MODELS.md` as a new version entry (do not overwrite the old version):

```markdown
### extraction-v[N+1]

**Status:** Testing
**Changed from:** v[N]
**Date:** [today]
**Change:** [one sentence]
**Targeting:** [which failure pattern from Step 1]
**Prompt:**
```[prompt text]```
```

Update the code to use the new prompt version.

---

### Step 4 — Eval New Version

Run eval against all fixtures using the new prompt version.

Capture per-fixture scores. Compare to Step 0 baseline.

---

### Step 5 — Decide

| Result | Action |
|---|---|
| Overall score improved, no regressions | Mark v[N+1] as Active in MODELS.md. Update baseline in FIXTURES.md. Proceed to Step 6. |
| Score improved but regressions on specific fixtures | Analyze the regressions. If acceptable trade-off, get approval, then proceed. |
| Score unchanged or degraded | Mark v[N+1] as Rejected in MODELS.md. Revert code to v[N]. Loop back to Step 1 with new analysis. |

Do not mark old version as deprecated until new version has been confirmed stable.

---

### Step 6 — Sync

Invoke `/sync` to commit the prompt change, updated MODELS.md, and updated FIXTURES.md baseline.

---

## Iteration Limit

If 3 consecutive prompt versions fail to improve the target failure pattern, stop and log an IMP-XXX entry in `docs/ISSUES.md`. The problem may require a different model, better fixtures, or a schema change — not more prompt tuning.
