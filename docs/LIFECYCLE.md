# Project Lifecycle

> How all parts of the system connect — skills, docs, and artifacts.
> Read this to understand the full picture before starting work.

---

## Entry Points

Every session starts with `using-superpowers` checking which skill applies.

| Situation | Skill |
|---|---|
| Starting after any break or context reset | `/task-resume` |
| Greenfield project — nothing defined yet | `/task-discover` |
| New idea, new feature set | `/task-plan` |
| Approved milestone ready to build | `/task-execute` |
| Bug reported or found | `/task-fix` |
| Code quality sweep needed | `/task-audit` |
| Extraction prompt needs tuning | `/task-prompt` |
| End of session | `/sync` |

---

## Artifact Flow

What each skill reads and writes:

```
task-resume
  reads:  SESSION.md
          CLAUDE.md
          docs/PLAN.md
          docs/specs/[active milestone].md
          docs/BRIEF.md, SCHEMA.md, MODELS.md (if they exist)
          git log + git status
  writes: nothing (read-only recovery)
          → may invoke /sync if unsynced work exists

          ↓ then resumes work from confirmed pickup point

task-discover
  reads:  conversation (user answers)
  writes: docs/BRIEF.md              ← confirmed input for task-plan
          CLAUDE.md                  ← fills Project Overview, Architecture Non-Negotiables, AI Pipeline Rules

          ↓ BRIEF.md is the handoff to task-plan

task-plan
  reads:  docs/BRIEF.md              ← if greenfield, always start here
          codebase (via code-explorer sub-agent)
          docs/PRD.md
  writes: docs/specs/MXX-name.md     ← one per milestone
          docs/PLAN.md               ← new milestones added
          docs/PRD.md                ← updated if scope expands

          ↓ spec files are the handoff to task-execute

task-execute
  reads:  docs/specs/MXX-name.md
          docs/ENGINEERING-PRINCIPLES.md   ← Step 3.5 principles check
          docs/FIXTURES.md                 ← Step 4.6 eval gate
  writes: code + tests
          docs/ISSUES.md             ← P2/P3 violations from Step 3.5
          docs/PLAN.md               ← milestone marked complete
          docs/API.md                ← if endpoints changed
          docs/SCHEMA.md             ← if output schema changed
          docs/MODELS.md             ← if prompts or model config changed

          ↓ when tests fail after 3 iterations → invokes task-fix

task-fix
  reads:  docs/ISSUES.md             ← the logged issue
          codebase                   ← root cause verification
  writes: code fix
          docs/ISSUES.md             ← status updated to fixed

task-audit
  reads:  docs/ENGINEERING-PRINCIPLES.md
          codebase (full or targeted scope)
  writes: docs/ISSUES.md             ← all violations logged with P1/P2/P3

          ↓ P1 violations immediately invoke task-fix

task-prompt
  reads:  docs/MODELS.md             ← current prompt version
          docs/FIXTURES.md           ← eval baseline
  writes: docs/MODELS.md             ← new version entry added
          docs/FIXTURES.md           ← new baseline if prompt improved
          code                       ← prompt version reference updated

sync
  reads:  conversation state
  writes: SESSION.md
          docs/PLAN.md
          docs/DECISIONS.md
          docs/ISSUES.md
          → git commit
```

---

## ENGINEERING-PRINCIPLES.md — The Shared Reference

Two skills read it:

- **`task-audit`** — full or targeted sweep across the codebase
- **`task-execute` Step 3.5** — targeted check on files touched in the current milestone only

Both use the same P1/P2/P3 severity ratings defined in the file, so violations logged by either end up in `docs/ISSUES.md` with consistent priority. P1 always blocks. P2/P3 get logged and continue.

---

## SESSION.md and CLAUDE.md — The Recovery Mechanism

After any context reset or new session, the agent reads:

1. `CLAUDE.md` — rules + active milestone pointer
2. `SESSION.md` — exact pickup point, last completed, immediate next task

These two files together answer: what are the rules, where am I, what's next. Everything else is derivable from the codebase and docs.

---

## Skill Composition

Skills are composable — they invoke each other rather than duplicating logic:

| Caller | Invokes | When |
|---|---|---|
| `task-execute` | `task-fix` | Tests fail after 3 iterations with no progress |
| `task-execute` | `task-fix` | Eval regression after implementation |
| `task-audit` | `task-fix` | P1 violation found in sweep |
| Every skill | `sync` | As the final step after all work is complete |

---

## Full Lifecycle — One Piece of Work End to End

```
0a. Session follows a break or context reset
   └─ /task-resume
       ├─ Read all state files + git
       ├─ Reconcile spec vs actual code state
       ├─ Present reconstructed state → CONFIRMATION GATE
       ├─ Handle any unsynced work (/sync if needed)
       └─ Resume from confirmed pickup point

0b. Greenfield project — nothing defined yet
   └─ /task-discover
       ├─ Structured questions (product, flow, stack, AI, constraints)
       ├─ Clarify gaps + research ambiguities
       ├─ Proactive product observations
       ├─ Summary confirmed → CONFIRMATION GATE
       ├─ docs/BRIEF.md written
       └─ CLAUDE.md project sections filled (Overview, Architecture, AI Pipeline)

          ↓ then proceed to /task-plan

1. Idea arrives (with BRIEF.md or existing codebase)
   └─ /task-plan
       ├─ Theory analysis (code-explorer sub-agent)
       ├─ Milestone breakdown → APPROVAL GATE
       ├─ Specs written to docs/specs/
       ├─ docs/PLAN.md updated
       └─ /sync → commit

2. Implement a milestone
   └─ /task-execute MXX
       ├─ Read spec + verify dependencies complete
       ├─ Baseline test run
       ├─ Understand codebase (code-explorer sub-agent)
       ├─ Write failing tests
       ├─ Implement in chunks (general-purpose sub-agent per chunk)
       ├─ Step 3.5: principles check on touched files
       │   ├─ P1 found → /task-fix → resume
       │   └─ P2/P3 → log in ISSUES.md, continue
       ├─ Full test run
       │   ├─ Pass → continue
       │   ├─ Fail < 3 iterations → analyze, fix, re-run
       │   ├─ Fail 3 iterations → /task-fix → resume
       │   └─ Spec wrong → PAUSE, surface gap, wait for direction
       ├─ Eval gate (if AI pipeline touched)
       │   └─ Regression → /task-fix → resume
       ├─ docs/PLAN.md milestone marked complete
       └─ /sync → commit

3. Bug appears
   └─ /task-fix
       ├─ Log in ISSUES.md
       ├─ Verify root cause from actual code
       ├─ Write fix spec → APPROVAL GATE
       ├─ Implement fix
       ├─ Tests pass + eval passes
       ├─ ISSUES.md updated to fixed
       └─ /sync → commit

4. Code quality drift
   └─ /task-audit
       ├─ Sweep against ENGINEERING-PRINCIPLES.md
       ├─ All violations logged in ISSUES.md with P1/P2/P3
       ├─ P1 violations → /task-fix for each
       └─ /sync → commit

5. Extraction prompt degrades
   └─ /task-prompt
       ├─ Baseline eval against fixtures
       ├─ Analyze failure patterns (code-explorer sub-agent)
       ├─ Propose prompt change → APPROVAL GATE
       ├─ Apply change, new version in MODELS.md
       ├─ Eval new version → compare to baseline
       ├─ Improved: mark active, update FIXTURES.md baseline
       ├─ Not improved: mark rejected, revert, iterate
       └─ /sync → commit
```

---

## Approval Gates

Two gates exist across all workflows. Nothing proceeds past them without explicit user confirmation:

| Gate | Skill | What is shown |
|---|---|---|
| Milestone breakdown approved | `task-plan` | List of milestones with dependencies |
| Spec approved | `task-plan`, `task-fix`, `task-prompt` | Full spec or prompt change before any code is written |

---

## Docs at a Glance

| File | Written by | Read by | Purpose |
|---|---|---|---|
| `SESSION.md` | `sync` | Every session start | Exact pickup point |
| `CLAUDE.md` | Human | Every session start | Rules + active milestone |
| `docs/BRIEF.md` | `task-discover` | `task-plan` | Confirmed project inputs — product, stack, constraints |
| `docs/PRD.md` | `task-plan` | `task-plan` | What to build |
| `docs/PLAN.md` | `task-plan`, `task-execute`, `sync` | `task-execute`, `sync` | Milestone status |
| `docs/specs/` | `task-plan` | `task-execute` | Per-milestone contract |
| `docs/ENGINEERING-PRINCIPLES.md` | Human | `task-audit`, `task-execute` | What good code looks like |
| `docs/ISSUES.md` | `task-fix`, `task-audit`, `task-execute` | `task-fix` | Bugs + violations + improvements |
| `docs/MODELS.md` | `task-prompt`, `task-execute` | `task-prompt` | Prompt versions + costs |
| `docs/FIXTURES.md` | `task-prompt`, `task-execute` | `task-prompt`, `task-execute` | Eval baselines |
| `docs/SCHEMA.md` | `task-execute` | `task-execute`, `task-audit` | Extraction output format |
| `docs/API.md` | `task-execute` | Developers | Endpoint reference |
| `docs/DECISIONS.md` | `sync` | Any skill | D-XXX design decisions |
| `docs/adr/` | Human | Any skill | Architectural decisions |
