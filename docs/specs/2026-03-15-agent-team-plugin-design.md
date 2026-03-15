# Agent Team Plugin — Design Specification

**Date:** 2026-03-15
**Status:** Approved — awaiting user review before implementation planning
**Feature slug:** agent-team-plugin

---

## 1. Overview

A reusable Claude Code plugin that instantiates a structured AI development team on top of any project. The team runs as a single `/team "feature description"` command, executes a layered pipeline (intent → design → approval → execution → QA → reporting), and coordinates entirely through a file-based blackboard (`.team/` directory).

### Goals
- Catch ambiguity before the team burns tokens (User Proxy layer)
- Produce auditable design artifacts before any code is written (Architect layer)
- Break work into parallel, independently deliverable tasks (Team Lead)
- Integrate existing superpowers skills and global agents — no duplication
- Persist all state in files so runs survive session crashes and resume cleanly

### Constraints
- No experimental Agents Team feature (`TeamCreate`/`SendMessage`)
- Parallelism via `Agent` tool with `run_in_background: true` (within-session)
- Single format throughout: Markdown with YAML frontmatter for metadata
- Reuse all existing global agents (`~/.claude/agents/`) via thin wrappers

---

## 2. Plugin Architecture

```
~/.claude/plugins/.../plugins/agent-team/
├── README.md
├── commands/
│   └── team.md                        ← /team command entry point
├── skills/
│   └── team/
│       ├── SKILL.md                   ← main orchestration logic
│       ├── phases/
│       │   ├── intent.md              ← User Proxy instructions
│       │   ├── design.md              ← Solution Architect instructions
│       │   ├── planning.md            ← Team Lead: read design, create tasks
│       │   ├── execution.md           ← All specialists: team mode protocol
│       │   └── reporting.md           ← Project Manager instructions
│       └── references/
│           ├── init.md                ← .team/ directory initialisation
│           ├── file-conventions.md    ← data model reference
│           ├── task-template.md       ← task file schema
│           └── deliverable-template.md
└── agents/
    ├── user-proxy.md                  ← new (wraps UX Researcher + Feedback Synthesizer)
    ├── team-lead.md                   ← new (wraps Software Architect)
    ├── team-db-engineer.md            ← new (wraps Database Optimizer)
    ├── team-backend-engineer.md       ← wrapper → engineering-backend-architect
    ├── team-frontend-engineer.md      ← wrapper → engineering-frontend-developer
    ├── team-ui-ux-designer.md         ← wrapper → design-ui-designer + design-ux-researcher
    ├── team-qa-engineer.md            ← wrapper → testing-reality-checker
    ├── team-solution-architect.md     ← wrapper → engineering-software-architect
    └── team-project-manager.md       ← wrapper → project-manager-senior + executive-summary-generator
```

### Thin Wrapper Pattern

Every specialist in the plugin is a thin wrapper over an existing global agent in `~/.claude/agents/`. The wrapper adds only three things: when to invoke skills, where to read task context, where to write output. The global agent's full domain expertise is inherited unchanged.

```markdown
---
name: team-{role}
description: {Role} operating in team mode...
model: inherit
---

You are the {GlobalAgent} (all capabilities apply).

## 🤝 Team Mode Protocol

### On Start
1. Read task: `tasks/ongoing/{task-id}.md`
2. Check inbox: `inbox/{role}/unread/` — read all, move each to `read/`
3. Read design context: `plan/active/`
4. If blocked: write to Team Lead inbox, stop

### Implementation
→ Invoke Skill("superpowers:test-driven-development") before writing code
→ Invoke Skill("superpowers:systematic-debugging") on any test failure

### On Completion
→ Invoke Skill("superpowers:verification-before-completion")
1. Write `deliverables/submitted/{task-id}/RESULT.md`
2. Write inbox messages to any peers who need to know
3. Notify Team Lead (they handle LEDGER + file move)
```

---

## 3. Data Model — `.team/` Directory

Created at project root on first `/team` run. Pure Markdown throughout; YAML frontmatter for metadata agents need to parse.

```
{project-root}/.team/
│
├── plan/
│   ├── PLAN.md                        ← index: all briefs, designs, ADRs
│   ├── active/
│   │   ├── 001-brief-{slug}.md
│   │   ├── 001-design-{slug}.md
│   │   ├── 001-adr-{decision}.md
│   │   └── 001-approval-{slug}.md
│   └── archived/
│       └── (superseded designs moved here)
│
├── tasks/
│   ├── LEDGER.md                      ← Team Lead owned: all tasks, deps, owners, status
│   ├── pending/
│   │   └── 003-task-login-ui.md
│   ├── ongoing/
│   │   └── 002-task-auth-api.md
│   └── completed/
│       └── 001-task-db-schema.md
│
├── inbox/
│   ├── backend/
│   │   ├── unread/
│   │   │   └── 001-from-frontend-api-contract.md
│   │   └── read/
│   ├── frontend/
│   │   ├── unread/
│   │   └── read/
│   ├── db/
│   ├── ui-ux/
│   ├── qa/
│   └── team-lead/
│
├── deliverables/
│   ├── DELIVERABLES.md                ← index: task-id, owner, status, summary
│   ├── submitted/
│   │   └── 002-auth-api/
│   │       └── RESULT.md
│   └── approved/
│       └── 001-db-schema/
│           └── RESULT.md
│
└── reports/
    ├── STATUS.md                      ← rolling project status (PM owned)
    └── sessions/
        └── 2026-03-15-session.md
```

### Index Files

**`plan/PLAN.md`** — design artifact registry:
```markdown
| ID  | Type   | Feature   | Status   | File                            |
|-----|--------|-----------|----------|---------------------------------|
| 001 | brief  | user-auth | approved | active/001-brief-user-auth.md   |
| 001 | design | user-auth | approved | active/001-design-user-auth.md  |
| 001 | adr    | jwt       | accepted | active/001-adr-jwt-vs-session.md|
```

**`tasks/LEDGER.md`** — task registry (Team Lead owned exclusively):
```markdown
---
feature: "{slug}"
created: {date}
last-updated: {date}
---

| ID  | Title      | Owner            | Status    | Depends On | Blocks   |
|-----|------------|------------------|-----------|-----------|----------|
| 001 | DB schema  | db-engineer      | completed | —         | 002, 003 |
| 002 | Auth API   | backend-engineer | ongoing   | 001       | 004      |
| 003 | Login UI   | frontend-engineer| pending   | 001, 002  | 004      |
| 004 | QA flow    | qa-engineer      | pending   | 002, 003  | —        |

Valid status values: `pending` | `ongoing` | `completed` | `paused` | `deadlocked`
```

**`deliverables/DELIVERABLES.md`** — deliverable registry:
```markdown
| Task | Owner            | Status    | Summary                          |
|------|------------------|-----------|----------------------------------|
| 001  | db-engineer      | approved  | Schema + migrations for auth     |
| 002  | backend-engineer | submitted | JWT endpoints, pending QA        |
```

### Task File Schema

```markdown
---
id: "002"
title: "Auth API endpoints"
owner: backend-engineer
status: ongoing
depends-on: ["001"]
blocks: ["004"]
created: 2026-03-15
started: 2026-03-15
completed: ~
retry: 0
blocked: false
qa-cycles: 0
---

## Context
Why this task exists (one paragraph from design).

## Scope
Exactly what to build. Nothing more.

## Acceptance Criteria
- [ ] POST /auth/login returns { user: { id, role, name }, token }
- [ ] Token is httpOnly cookie
- [ ] 401 on invalid credentials

## Dependencies Available At
- DB schema: deliverables/approved/001-db-schema/RESULT.md

## QA Rejection Notes
(populated by Team Lead on re-open)
```

### Inbox Message Schema (one file per message)

```markdown
---
id: "001"
from: frontend-engineer
to: backend-engineer
task-ref: "002"
type: info           ← info | design-issue | blocker
sent: 2026-03-15T14:10
read: false
---

POST /auth/login needs to return `{ user: { id, role, name } }`.
Current design only shows `{ user: { id } }`. Please confirm.
```

### Movement Rules

| Event | Who moves | From → To |
|---|---|---|
| Task created | Team Lead | — → `tasks/pending/` |
| Specialist dispatched | Team Lead | `pending/` → `ongoing/` |
| Specialist completes | Team Lead | `ongoing/` → `completed/` |
| QA rejects | Team Lead | `completed/` → `ongoing/` |
| Task re-opened | Team Lead | `completed/` → `ongoing/` |
| Design superseded | Solution Architect | `plan/active/` → `plan/archived/` |
| Deliverable approved | QA Engineer | `deliverables/submitted/` → `deliverables/approved/` |
| Inbox message read | Specialist | `inbox/{role}/unread/` → `inbox/{role}/read/` |
| Design issue raised | Team Lead | `ongoing/` → `ongoing/` (status: paused in LEDGER) |
| Design fix applied | Team Lead | paused → `ongoing/` (resumes), invalidated completed → `ongoing/` |

**Team Lead is the only agent that writes `tasks/LEDGER.md` and moves task files.**

---

## 4. Agent Role Definitions

### 4.1 User Proxy (new)

Wraps: `design-ux-researcher` (interview methodology) + `product-feedback-synthesizer` (synthesis)

**Purpose:** Catch ambiguity before the team starts. Produces a brief so complete the Architect has zero follow-up questions.

**Skills invoked:** `superpowers:brainstorming`

**Process:**
1. Restate the request in one sentence — confirm or correct
2. Ask clarifying questions one at a time (max 5) across: scope, users, constraints, success criteria, exclusions
3. Synthesize answers into structured brief
4. Flag unresolved items in Open Questions

**Output:** `plan/active/{NNN}-brief-{slug}.md` + PLAN.md row

**Critical rules:**
- One question per message — never bundle
- Never propose solutions (Architect's job)
- Never add requirements the user did not mention
- Quantify vague terms ("admin users" → "2–3 users with write access")

**Success metric:** Architect reads brief with zero follow-up questions.

---

### 4.2 Solution Architect (wrapper)

Wraps: `engineering-software-architect`

**Purpose:** Translate approved brief into technical design. Produce system boundaries, technology choices, integration patterns, and ADRs.

**Skills invoked:** `feature-dev:code-architect`, `feature-dev:code-explorer`

**Process:**
1. Read `plan/active/{NNN}-brief-{slug}.md`
2. Explore codebase for existing patterns (code-explorer)
3. Design system architecture (code-architect)
4. Write one ADR per significant technology decision

**Outputs:**
- `plan/active/{NNN}-design-{slug}.md`
- `plan/active/{NNN}-adr-{decision}.md` (one per decision)
- Update `plan/PLAN.md`

**Note:** Does NOT re-open design decisions after Team Lead approval. Design is final once approved.

---

### 4.3 Team Lead (new)

Wraps: `engineering-software-architect` (decomposition + dependency thinking)

**Purpose:** Turn approved design into a coordinated task graph. Owns LEDGER exclusively. Dispatches specialists, tracks progress, handles blockers.

**Skills invoked:** `superpowers:writing-plans`, `superpowers:dispatching-parallel-agents`

**Process:**
1. Read all `plan/active/` artifacts
2. Apply bounded-context thinking to identify independent work units
3. DB schema task always created first (upstream dependency of everything)
4. Write LEDGER.md + individual task files in `tasks/pending/`
5. Dispatch first batch (all tasks with no dependencies) in parallel
6. On each completion notification: move file, update LEDGER, unlock and dispatch next batch
7. After all implementation tasks: dispatch QA
8. After QA: report back to SKILL.md orchestrator

**Critical rules:**
- Only agent that writes LEDGER.md
- Only agent that moves task files between folders
- Never writes implementation code
- Does NOT re-open approved design decisions
- QA runs only after ALL implementation tasks complete

**Handles design-issue inbox type:** Pauses dependent tasks, escalates to SKILL.md, optionally re-dispatches Architect.

---

### 4.4 Database Engineer (new)

Wraps: Database Optimizer (PostgreSQL, EXPLAIN ANALYZE, indexing, N+1, safe migrations, Supabase/PlanetScale)

**Purpose:** Schema design and migration layer. Runs first — all specialists depend on schema output.

**Skills invoked:** `superpowers:test-driven-development`, `superpowers:verification-before-completion`

**Critical rules (from Database Optimizer):**
- Every migration reversible (up + down)
- Indexes created CONCURRENTLY (no table locks in production)
- Index every foreign key
- EXPLAIN ANALYZE before marking done
- No business logic in DB
- UUID primary keys unless design specifies otherwise

**Deliverable must include:**
- Schema diagram (tables, columns, relationships)
- Migration file paths
- Index rationale (which query each index serves)
- Query patterns for Backend Engineer to follow
- Inbox messages to backend/ and frontend/ summarising schema decisions

---

### 4.5 Backend Engineer (wrapper)

Wraps: `engineering-backend-architect`

Inherits full thin-wrapper protocol (section 2). Team-mode specifics:

**Skills invoked:** `superpowers:test-driven-development`, `superpowers:verification-before-completion`, `superpowers:systematic-debugging` (on error)

**Additional reads on start:**
- `deliverables/approved/{db-task-id}/RESULT.md` — schema and query patterns from DB Engineer

**Critical rules:**
- Never modify schema — raise `design-issue` inbox message to Team Lead if schema is wrong
- Follow query patterns documented by DB Engineer
- API contracts (request/response shape) must be written to inbox/frontend/ before marking done

**Deliverable must include:**
- API endpoint list (method, path, request/response shape)
- Auth/middleware applied per route
- Error codes and their meanings
- Any environment variables required

---

### 4.6 Frontend Engineer (wrapper)

Wraps: `engineering-frontend-developer`

Inherits full thin-wrapper protocol (section 2). Team-mode specifics:

**Skills invoked:** `superpowers:test-driven-development`, `superpowers:verification-before-completion`, `superpowers:systematic-debugging` (on error)

**Additional reads on start:**
- `inbox/frontend/unread/` — check for API contract messages from Backend Engineer
- `deliverables/approved/{db-task-id}/RESULT.md` — schema shape for form validation alignment

**Dispatch timing:** UI/UX Designer task (if present) has no schema dependency — can run in parallel with DB Engineer in the first wave. Backend Engineer task must be complete before Frontend can finalise API integration.

**Critical rules:**
- Do not assume API shape — wait for inbox confirmation from Backend or read their RESULT.md
- If API contract conflicts with design, raise `design-issue` inbox message to Team Lead

**Deliverable must include:**
- Component list with file paths
- State management decisions
- API calls made (endpoint, method, expected response used)
- Any environment variables required

---

### 4.7 UI/UX Designer (wrapper)

Wraps: `design-ui-designer` (visual output) + `design-ux-researcher` (user validation)

Inherits full thin-wrapper protocol (section 2). Team-mode specifics:

**Skills invoked:** `superpowers:test-driven-development` (component specs as tests), `superpowers:verification-before-completion`

**Dispatch timing:** No schema dependency — dispatched in the first parallel wave alongside DB Engineer.

**Critical rules:**
- Produce design tokens and component specs that Frontend Engineer can implement without questions
- No Figma or external tool output — all specs are Markdown + CSS/code

**Deliverable must include:**
- Design token definitions (colors, spacing, typography)
- Component specs (states, variants, accessibility notes)
- Responsive breakpoint decisions
- Handoff notes for Frontend Engineer written to `inbox/frontend/unread/`

---

### 4.8 QA Engineer (wrapper)

Wraps: `testing-reality-checker`

Inherits full thin-wrapper protocol (section 2). Team-mode specifics:

**Skills invoked:** `superpowers:requesting-code-review`

**Reads on start:**
- `deliverables/submitted/` — all submitted deliverables for this run
- `tasks/LEDGER.md` — to understand scope and acceptance criteria
- Each task file for its `## Acceptance Criteria` section

**Critical rules (from testing-reality-checker):**
- Default to "NEEDS WORK" — requires evidence to approve, not absence of evidence to reject
- Every rejection must cite a specific acceptance criterion that was not met
- After 2 QA rejections on a task: flag to Team Lead for user escalation (section 6.3)

**Deliverable must include:**
- Per-task verdict: `approved` or `rejected` with specific criterion failures
- Update `deliverables/DELIVERABLES.md` status column
- Move approved deliverables to `deliverables/approved/`
- Write rejection notes to the task file `## QA Rejection Notes` for Team Lead to use on re-dispatch

---

### 4.9 Project Manager (wrapper)

Wraps: `project-manager-senior` (task tracking) + `support-executive-summary-generator` (session reports)

Inherits full thin-wrapper protocol (section 2). Team-mode specifics:

**Skills invoked:** none (reads and synthesises only)

**Reads on start:**
- `tasks/LEDGER.md` — full task picture
- `deliverables/DELIVERABLES.md` — what was built
- `plan/PLAN.md` — what was designed
- `reports/STATUS.md` — rolling project history

**Critical rules:**
- Session report is factual — what was planned, what was built, what was approved, what is still open
- STATUS.md is the rolling summary — append session outcome, do not rewrite history
- Use executive-summary-generator format for session reports: situation, findings, impact, next steps

**Deliverable must include:**
- `reports/sessions/{date}-session.md` — session summary (325–475 words, exec summary format)
- Updated `reports/STATUS.md` — one-paragraph session outcome appended

---

## 5. Orchestration Flow (SKILL.md)

```
/team "feature description"
│
├── Pre-flight
│   ├── Check .team/ exists → init if absent
│   ├── Check LEDGER for incomplete runs → offer resume
│   └── Confirm feature description is clear
│
├── Phase 1: Intent
│   └── Agent(user-proxy) → plan/active/brief.md
│
├── Phase 2: Design
│   └── Agent(team-solution-architect) → plan/active/design.md + ADRs
│
├── ⏸ APPROVAL GATE 1
│   ├── Present brief + design + ADRs to user
│   ├── "approved" → continue
│   ├── "revise [section]" → re-dispatch Architect with notes (max 3 cycles)
│   │   After 3 revision cycles without approval: surface to user,
│   │   ask them to resolve the conflict directly before continuing
│   └── "stop" → end
│
├── Phase 3: Planning + Execution
│   └── Agent(team-lead) [single long-running Agent call; Team Lead
│       itself calls Agent() for each specialist batch internally]
│       ├── Skill(writing-plans) → LEDGER + task files in tasks/pending/
│       ├── Skill(dispatching-parallel-agents)
│       ├── Dispatch wave 1: DB Engineer + UI/UX Designer (no deps) [parallel]
│       ├── Dispatch wave 2: Backend Engineer (deps: DB) [when wave 1 DB done]
│       ├── Dispatch wave 3: Frontend Engineer (deps: DB + Backend)
│       ├── Dispatch wave 4: QA (deps: all implementation tasks)
│       └── Report back to SKILL.md on full completion
│
├── ⏸ APPROVAL GATE 2
│   ├── Present QA findings to user
│   ├── "approved" → continue
│   ├── "fix [task-id]" → Team Lead re-opens + re-dispatches
│   │   User-initiated re-opens have no automatic cycle limit —
│   │   the user decides when quality is acceptable.
│   │   QA auto-escalation limit (2 cycles) applies to QA-initiated
│   │   rejections only (section 6.3).
│   └── "stop" → end without PR
│
├── Phase 4: Reporting
│   └── Agent(team-project-manager) → reports/sessions/DATE.md + STATUS.md
│
└── Completion
    └── Skill("superpowers:finishing-a-development-branch")
        → PR / merge / discard recommendation
```

### Skills-to-Phase Mapping

| Phase | Agent | Skills Invoked |
|---|---|---|
| Intent | User Proxy | `superpowers:brainstorming` |
| Design | Solution Architect | `feature-dev:code-architect`, `feature-dev:code-explorer` |
| Planning | Team Lead | `superpowers:writing-plans`, `superpowers:dispatching-parallel-agents` |
| Execution | All specialists | `superpowers:test-driven-development`, `superpowers:verification-before-completion` |
| Error | Any specialist | `superpowers:systematic-debugging` |
| QA | QA Engineer | `superpowers:requesting-code-review` |
| Completion | SKILL.md | `superpowers:finishing-a-development-branch` |

---

## 6. Error Handling + Resilience

### 6.1 Specialist produces no RESULT.md

Task stays `ongoing` in LEDGER. Team Lead re-dispatches with retry flag. After 2 retries: escalate to user.

Task frontmatter: `retry: N` (incremented per re-dispatch).

### 6.2 Dependency deadlock

Task B fails → Task A is permanently blocked. Team Lead marks downstream tasks `deadlocked` in LEDGER. SKILL.md surfaces options: fix B manually, skip, or stop.

### 6.3 QA rejects a deliverable

Team Lead re-opens task: `completed/` → `ongoing/`. Inserts QA rejection notes into task file under `## QA Rejection Notes`. Re-dispatches specialist. After 2 QA rejections: escalate to user before third dispatch.

Task frontmatter: `qa-cycles: N`.

### 6.4 Session dies mid-run

Pre-flight on next `/team` invocation detects orphaned `ongoing` tasks in LEDGER. Offers resume. Team Lead moves orphaned tasks back to `pending/` and re-dispatches. Task files are self-contained — cold-start re-dispatch produces correct output.

### 6.5 Design is wrong mid-execution

Specialist writes inbox message with `type: design-issue` to team-lead inbox before stopping. Team Lead pauses all dependent tasks (`paused` status in LEDGER), escalates to SKILL.md, optionally re-dispatches Architect with issue context.

On design fix:
- Architect updates `plan/active/` design file and moves old version to `plan/archived/`
- Team Lead assesses which completed tasks are invalidated by the change
- Tasks whose deliverables are inconsistent with the new design are re-opened: `completed/` → `ongoing/`, status updated in LEDGER, `qa-cycles` reset to 0
- Tasks whose deliverables are unaffected by the change remain completed
- Team Lead resumes paused tasks after re-opened tasks complete

The `paused` status must appear in LEDGER alongside pending/ongoing/completed/deadlocked. Movement rule addition: `paused` → `ongoing` when Team Lead resumes after design fix.

---

## 7. Performance + Token Efficiency Notes

**Per-specialist cold-start cost:** ~3,000–8,000 tokens (task file + design + role instructions). With 5 parallel specialists: ~15,000–40,000 tokens orientation cost before any implementation begins. Mitigations:

- Write lean plan/active/ files (no prose padding in design.md)
- Task files are self-contained but minimal — only context needed for that task
- LEDGER is a one-file index — Team Lead reads one file for full picture
- Completed tasks in `tasks/completed/` are not scanned unless needed (ad hoc)

**Parallelism:** Team Lead dispatches independent tasks simultaneously via `run_in_background: true`. Bottleneck is sequential layers (User Proxy → Architect → Team Lead → first task wave). Within each wave, specialists run concurrently.

---

## 8. Not In Scope (v1)

- Multi-project team (one `.team/` per project root)
- Cross-session agent persistence (agents start fresh each dispatch)
- Real-time progress UI
- Automated LEDGER conflict resolution when two agents write simultaneously
  (Mitigated in v1: Team Lead is the sole LEDGER writer; specialists write
  only to their own namespaced deliverable directories. Task IDs are assigned
  sequentially by Team Lead at planning time — no two tasks share an ID.
  Deliverable paths are `deliverables/{submitted|approved}/{task-id}/` which
  are unique per task-id, so parallel specialists never write to the same path.)
- Multi-tenant / shared team across multiple developers
- Conductor-style separate Claude Code processes per specialist

These are noted extensions for v2.

---

## 9. Open Questions

None — all resolved during design review.

---

*Spec written 2026-03-15. Ready for implementation planning.*
