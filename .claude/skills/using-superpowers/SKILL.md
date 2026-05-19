---
name: using-superpowers
description: Use when starting any conversation - establishes how to find and use skills, requiring Skill tool invocation before ANY response including clarifying questions
---

<SUBAGENT-STOP>
If you were dispatched as a subagent to execute a specific task, skip this skill.
</SUBAGENT-STOP>

<EXTREMELY-IMPORTANT>
If you think there is even a 1% chance a skill might apply to what you are doing, you ABSOLUTELY MUST invoke the skill.

IF A SKILL APPLIES TO YOUR TASK, YOU DO NOT HAVE A CHOICE. YOU MUST USE IT.

This is not negotiable. This is not optional.
</EXTREMELY-IMPORTANT>

## Available Skills

| Skill | When to use |
|---|---|
| `task-plan` | New idea, feature set, or significant change — produces specs |
| `task-execute` | Implement an approved milestone — consumes specs |
| `task-audit` | Sweep codebase against engineering principles |
| `task-fix` | Fix a bug or reported issue |
| `task-prompt` | Tune extraction prompts — eval-driven iteration |
| `sync` | End of session — update tracking docs and commit |

## The Rule

Invoke the relevant skill BEFORE any response or action. Even a 1% chance means invoke it.

```
User message → Does any skill apply? → Yes (even 1%) → Invoke Skill tool FIRST
                                     → Definitely not → Respond directly
```

## Red Flags — You Are Rationalizing

| Thought | Reality |
|---|---|
| "This is just a simple fix" | Fixes are task-fix. Check the skill. |
| "I need more context first" | Skill check comes BEFORE gathering context. |
| "Let me explore the codebase first" | Skills tell you HOW to explore. |
| "This doesn't need a formal skill" | If a skill exists, use it. |
| "I remember this skill" | Skills evolve. Read current version. |
