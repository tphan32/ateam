---
name: reporting
description: Project Manager phase — produce session report and update rolling status
---

# Phase 4: Reporting

Dispatch the `team-project-manager` agent to document the session outcome.

## Agent

`team-project-manager`

## Context to Pass

- `tasks/LEDGER.md` — full task picture
- `deliverables/DELIVERABLES.md` — what was built
- `plan/PLAN.md` — what was designed
- `reports/STATUS.md` — rolling project history

## Outputs

- `reports/sessions/{YYYY-MM-DD}-session.md` — session report (325–475 words, SCQA format)
- Updated `reports/STATUS.md` — one paragraph appended

## Completion Signal

Project Manager writes both output files and stops.
SKILL.md proceeds to completion after PM agent returns.
