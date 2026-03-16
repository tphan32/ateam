---
name: design
description: Solution Architect phase — translate approved brief into technical design and ADRs
---

# Phase 2: Design

Dispatch the `team-solution-architect` agent to translate the approved brief into
a technical design.

## Agent

`team-solution-architect`

## Context to Pass

- Path to approved brief: `plan/active/{NNN}-brief-{slug}.md`
- Instruction to read `plan/PLAN.md` for full context

## Outputs

- `plan/active/{NNN}-design-{slug}.md`
- `plan/active/{NNN}-adr-{decision}.md` (one per significant decision)
- Updated `plan/PLAN.md` rows

## Completion Signal

Architect writes all output files and stops.
The orchestrator reads all new entries in `plan/PLAN.md` to present at Approval Gate 1.

## On Revision (after Approval Gate rejection)

Pass revision notes to re-dispatched Architect in context.
Track revision counter in orchestrator session state.
Increment counter on each re-dispatch.
At counter == 3: surface conflict to user, ask them to resolve directly.
After user resolution: re-dispatch once more with final direction.
