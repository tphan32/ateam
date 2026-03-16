---
name: execution
description: Specialist execution protocol — shared by all specialists operating within the team
---

# Specialist Execution Protocol

All specialist agents (DB Engineer, Backend, Frontend, UI/UX, QA) follow this
shared protocol. Individual agent files add role-specific details on top of this.

---

## On Start (every specialist)

1. Read assigned task file: `tasks/ongoing/{task-id}.md` (path provided by Team Lead)
2. Check inbox: `inbox/{role}/unread/` — read all messages, move each to `inbox/{role}/read/`
3. Read relevant design files: `plan/active/`
4. Read dependency deliverables listed under `## Dependencies Available At` in task file
5. If blocked before starting: write to `inbox/team-lead/unread/{task-id}-blocked.md`, stop

## Before Writing Code

→ Invoke `Skill("superpowers:test-driven-development")`

## On Test Failure

→ Invoke `Skill("superpowers:systematic-debugging")`

## Before Submitting Deliverable

→ Invoke `Skill("superpowers:verification-before-completion")`

## On Completion

1. Write `deliverables/submitted/{task-id}/RESULT.md`
   (use schema from `~/.claude/skills/team/references/deliverable-template.md`)
2. Write inbox messages to any peers who need to know (per individual agent instructions)
3. Notify Team Lead: write to `inbox/team-lead/unread/{task-id}-complete.md`

## Blocker Protocol

If blocked mid-task (after starting):
1. Document the blocker clearly in an inbox message
2. Write to `inbox/team-lead/unread/{task-id}-blocked.md`:
   - What you were trying to do
   - What is blocking you
   - What you need to unblock
3. Stop work — do not guess, do not work around the blocker

## Inbox Message Schema

See `~/.claude/skills/team/references/file-conventions.md` for the full inbox message frontmatter schema.
