---
name: planning
description: Team Lead phase — decompose approved design into task graph and dispatch all specialists
---

# Phase 3: Planning + Execution

Dispatch the `team-lead` agent as a single long-running call.
Team Lead internally manages all specialist dispatches and wave coordination.

## Agent

`team-lead`

## Context to Pass

- Paths to all `plan/active/` artifacts (brief, design, all ADRs)
- Instruction to invoke `Skill("team:task-template")` for task file schema

## Execution Waves (Team Lead manages these internally)

```
Wave 1 (parallel): team-db-engineer + team-ui-ux-designer   (no dependencies)
Wave 2:            team-backend-engineer                     (depends: DB done)
Wave 3:            team-frontend-engineer                    (depends: DB + Backend done)
Wave 4:            team-qa-engineer                          (depends: all implementation done)
```

## Completion Signals

Team Lead reports to SKILL.md with one of:
- `COMPLETE` — all tasks completed and QA approved all deliverables
- `DEADLOCK: {task-id} — {reason}` — task permanently blocked, downstream blocked
- `DESIGN_ISSUE: {issue summary}` — specialist found design problem, team paused

## On DEADLOCK

SKILL.md presents to user: "Task {task-id} is permanently blocked because {reason}. Options: fix it manually, skip this task, or stop."
Act on user's choice.

## On DESIGN_ISSUE

SKILL.md presents issue to user.
If user approves fix: re-dispatch `team-solution-architect` with issue context, then re-dispatch `team-lead` to resume.
If user rejects: stop or continue without affected tasks (user's decision).
