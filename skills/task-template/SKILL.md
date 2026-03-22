---
name: team-task-template
description: Task file schema and template — used by Team Lead when creating task files
---

# Task File Template

Create each new task file using this template. Fill in all frontmatter fields.
Place the file at `tasks/pending/{NNN}-task-{slug}.md`.

---

```markdown
---
id: "{NNN}"
title: "{Human-readable task title}"
owner:
  { db-engineer|backend-engineer|frontend-engineer|ui-ux-designer|qa-engineer }
status: pending
depends-on: []
blocks: []
created: { YYYY-MM-DD }
started: ~
completed: ~
retry: 0
blocked: false
qa-cycles: 0
---

## Project Context

{Package manager, test runner, build system, monorepo layout, off-limits paths — copied from CLAUDE.md or auto-discovered. Specialists must use this and not re-run discovery.}

## Context

Why this task exists (one paragraph drawn directly from the design document).
Do not pad this — only include context the specialist genuinely needs.

## Scope

Exactly what to build. Be specific. Do not add anything not listed here.

## Acceptance Criteria

- [ ] {Specific, measurable criterion — can be verified by QA without ambiguity}
- [ ] {Another criterion}

## Dependencies Available At

{List file paths of approved deliverables this task depends on. Omit section if no deps.}

- {role} deliverable: `deliverables/approved/{NNN}-{slug}/RESULT.md`

## QA Rejection Notes

(Populated by Team Lead on re-open. Leave empty on first dispatch.)
```
