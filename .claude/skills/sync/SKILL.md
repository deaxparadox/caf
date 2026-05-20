---
name: sync
description: End-of-session sync. Updates the 4 tracking files that change between sessions, then commits. Replaces the 15-file sync dance — only touch files that actually changed.
---

# Sync — Session State

## Overview

Updates exactly the files that changed this session. Not 15 files. Not a full audit. Just the 4 that actually track state.

---

## Step 1 — Scan the Conversation

Answer these before touching any file:

- What was completed? (features, fixes, prompt changes)
- What is the immediate next task?
- Were any new D-XXX decisions made?
- Did any task checkboxes in PLAN.md get done?
- Were any bugs logged, verified, or fixed?
- Were any prompts changed? (update MODELS.md version status)
- Did any endpoints change shape? (update API.md)
- Did the extraction schema change? (update SCHEMA.md)

---

## Step 2 — Update SESSION.md (always)

Overwrite completely:

```markdown
# Session State — [DATE] | [one-line summary]

**Last session:** [date]
**Completed this session:** [bullet list]
**Immediate next:** [exact next task with file path]

## Branch
[current branch]

## Milestone Status
[table: milestone ID | name | status]

## Running Stack
[ports and commands]

## Key Docs for Next Session
[3-5 most relevant files]
```

---

## Step 3 — Update PLAN.md (if tasks completed)

Mark completed tasks `[x]`. Update module status if fully done.

---

## Step 4 — Update DECISIONS.md (if new decisions)

Add new D-XXX entries with: date, decision, reason.

---

## Step 5 — Update ISSUES.md (if bugs changed)

- New bug reported → add ISS-XXX entry
- Bug fixed → move to Closed, add fix summary
- New improvement noted → add IMP-XXX entry

---

## Step 6 — Update domain docs (only if changed)

| Doc | Update when |
|---|---|
| `docs/API.md` | Endpoint added, removed, or response shape changed |
| `docs/SCHEMA.md` | Extraction output format changed |
| `docs/MODELS.md` | Prompt version status changed (active/rejected/deprecated) |
| `docs/FIXTURES.md` | New fixtures added or eval baseline updated |
| `docs/DEVELOPER.md` | New pattern or gotcha discovered |

**Skip any doc that did not change this session.**

---

## Step 7 — Commit

```bash
git add -A
git commit -m "Session sync: [date] — [one-line summary]"
```

---

## Red Flags

| Missed thing | Check |
|---|---|
| SESSION.md has old date | Always overwrite |
| Tasks done but PLAN.md unchanged | Scan every task mentioned in conversation |
| Prompt changed but MODELS.md version not updated | Check task-prompt runs |
| New fixture added but FIXTURES.md not updated | Check eval steps |
| API shape changed but API.md stale | Check new endpoints or serializer changes |
