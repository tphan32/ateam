# Agent Team Plugin

A Claude Code plugin that instantiates a structured AI development team on top of any project.

## What It Does

Run `/team "feature description"` to coordinate a full development pipeline:

1. **Intent** — User Proxy clarifies your request, produces an unambiguous brief
2. **Design** — Solution Architect translates the brief into technical design + ADRs
3. **Approval Gate** — You review and approve (or revise) before any code is written
4. **Execution** — Team Lead coordinates specialists in dependency order (DB → Backend → Frontend, UI/UX in parallel)
5. **QA** — Reality Checker reviews all deliverables against acceptance criteria
6. **Approval Gate** — You review QA findings and approve or request fixes
7. **Report** — Project Manager produces an executive session summary

All state is persisted in `.team/` at your project root — runs survive session crashes and resume cleanly.

## Installation

Install via Claude Code plugin marketplace or copy this directory to
`~/.claude/plugins/cache/<namespace>/agent-team/<version>/`.

## Usage

```
/team "add user authentication with JWT"
/team "build a CSV import feature for the admin dashboard"
```

## Architecture

- **Blackboard pattern** — agents communicate through `.team/` files, not direct calls
- **Team Lead as coordinator** — sole writer of `tasks/LEDGER.md`, sole mover of task files
- **Thin wrappers** — all specialists wrap existing global agents in `~/.claude/agents/`
- **No experimental features** — uses standard `Agent` tool with `run_in_background: true`

## Agents

| Agent | Wraps | Role |
|-------|-------|------|
| user-proxy | UX Researcher + Feedback Synthesizer | Requirement clarification |
| team-solution-architect | Software Architect | Technical design |
| team-lead | Software Architect | Task coordination |
| team-db-engineer | — | Schema + migrations |
| team-backend-engineer | Backend Architect | API implementation |
| team-frontend-engineer | Frontend Developer | UI implementation |
| team-ui-ux-designer | UI Designer + UX Researcher | Design specs |
| team-qa-engineer | Reality Checker | Quality gate |
| team-project-manager | Senior PM + Executive Summary Generator | Reporting |
