---
name: intent
description: User Proxy phase — clarify feature request, produce approved brief
---

# Phase 1: Intent

Dispatch the `user-proxy` agent to clarify the feature request and produce
a structured brief.

## Agent

`user-proxy`

## Context to Pass

- Feature description (the argument from `/team`)
- Current `.team/plan/PLAN.md` if it exists (for ID continuity)

## Output

- `plan/active/{NNN}-brief-{slug}.md`
- Updated `plan/PLAN.md` row

## Initialization Requirement

Before dispatch: read `skills/team/references/init.md` and ensure `.team/` is initialized.

## Completion Signal

User Proxy writes the brief, confirms with user, and stops.
The orchestrator reads the brief path from `plan/PLAN.md` to pass to the Design phase.
