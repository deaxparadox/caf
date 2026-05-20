---
name: task-resume
description: Session recovery after any break or context reset. Reconstructs exact project state from docs and git, reconciles synced vs unsynced work, and confirms pickup point before resuming. Run at the start of any session that follows a break.
---

# Task Resume — Session Recovery

## Overview

Run this at the start of any session that follows a break — whether the last session ended with `/sync` or was cut short. Its job is to reconstruct exact project state, surface any discrepancy between what was planned and what was actually done, and confirm the pickup point before any work resumes.

Do not assume anything. Derive state from files and git — not from memory.

---

## Step 1 — Read the Core State Files

Read all of these. Do not skip any because the project "looks simple":

1. `SESSION.md` — last known pickup point (may be stale if last session was unsynced)
2. `CLAUDE.md` — active milestone, core rules
3. `docs/PLAN.md` — full milestone status
4. Active spec file: `docs/specs/[active milestone].md` — task checklist, acceptance criteria
5. `docs/BRIEF.md` — if exists, product intent and constraints
6. `docs/SCHEMA.md` — if exists, current extraction output format
7. `docs/MODELS.md` — if exists, active prompt versions
8. `docs/ISSUES.md` — any open issues that affect current milestone

---

## Step 2 — Check Actual Git State

Run these and read the output:

```bash
git log --oneline -10       # what was actually committed
git status                  # any uncommitted changes
git diff --stat HEAD        # what changed but wasn't committed
```

This is ground truth. `SESSION.md` describes intent — git describes reality.

---

## Step 3 — Reconcile

Compare `SESSION.md` + spec task checklist against git state. Look for:

- **Clean sync:** `SESSION.md` matches last commit, no uncommitted changes → state is reliable, proceed to Step 4
- **Unsynced commits:** commits exist that `SESSION.md` doesn't mention → update understanding of what was completed
- **Uncommitted work:** files changed but not committed → work was in progress when session closed
- **Spec vs reality gap:** spec tasks marked complete but no corresponding code exists, or code exists for tasks not in the spec

For any discrepancy, derive what actually happened from the git diff and file state — do not guess.

---

## Step 4 — Present Reconstructed State

Show the user a clear picture of where things stand:

```
## Session Recovery

**Last synced:** [date/commit from git log]
**Active milestone:** [from CLAUDE.md]
**Milestone status:** [X of N tasks complete, based on git + spec]

**Uncommitted work:** [files changed but not committed, or "none"]
**Open issues affecting this milestone:** [from ISSUES.md, or "none"]

**Recommended pickup point:** [specific next task from spec]
```

**CONFIRMATION GATE — PAUSE HERE.**

Ask: "Does this match your understanding? Any corrections before we continue?"

Do not proceed until confirmed. If the user corrects anything — update your understanding before resuming.

---

## Step 5 — Handle Unsynced Work

If uncommitted changes exist from the previous session:

**Option A — Changes are good, just not committed:**
Run `/sync` now to commit them cleanly before resuming work.

**Option B — Changes are incomplete or broken:**
Surface what's incomplete, discuss with the user whether to continue from that state or reset to last commit. Do not make this decision unilaterally.

**Option C — No uncommitted changes, but SESSION.md is stale:**
`SESSION.md` wasn't updated — run `/sync` after confirming state to bring it current, then resume.

---

## Step 6 — Resume

Once state is confirmed and any unsynced work is handled:

- Resume from the confirmed pickup point in the active spec
- If resuming mid-milestone: re-read the spec task breakdown, pick up at the first uncompleted task
- If the previous milestone just completed: confirm `PLAN.md` reflects it, then proceed to the next milestone or run `/task-plan` if no next milestone exists

---

## What Good Looks Like

A complete `/task-resume` run produces:
- Confirmed understanding of current project state
- No ambiguity about what was done vs what still needs doing
- Clean git state (no orphaned uncommitted work)
- A clear, confirmed pickup point

Work resumes from a known-good state — not from assumptions.
