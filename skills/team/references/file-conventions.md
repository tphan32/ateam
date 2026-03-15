# File Conventions Reference

Complete data model for `.team/` directory. Authoritative reference for file schemas,
naming conventions, and movement rules. All agents must follow this exactly.

---

## Naming Conventions

### Plan Files

```
plan/active/{NNN}-{type}-{feature-slug}.md
```
Types: `brief`, `design`, `adr`, `approval`
Example: `plan/active/001-design-user-auth.md`

### Task Files

```
tasks/{state}/{NNN}-task-{feature-slug}.md
```
States: `pending`, `ongoing`, `completed`
Example: `tasks/ongoing/002-task-auth-api.md`

### Inbox Messages

```
inbox/{role}/unread/{msg-id}-from-{sender}-{topic}.md
```
Roles: `backend`, `frontend`, `db`, `ui-ux`, `qa`, `team-lead`
Example: `inbox/backend/unread/001-from-frontend-api-contract.md`

### Deliverables

```
deliverables/{state}/{task-id}/RESULT.md
```
States: `submitted`, `approved`
Example: `deliverables/submitted/002-auth-api/RESULT.md`

### Session Reports

```
reports/sessions/{YYYY-MM-DD}-session.md
```

---

## File Schemas

### Plan Brief

```markdown
---
id: "{NNN}"
type: brief
feature: "{slug}"
status: approved
created: {YYYY-MM-DD}
---

## Feature Request
## Scope
## Users
## Constraints
## Success Criteria
## Exclusions
## Open Questions
```

### Plan Design

```markdown
---
id: "{NNN}"
type: design
feature: "{slug}"
status: approved
created: {YYYY-MM-DD}
brief: "{NNN}-brief-{slug}.md"
---

## Architecture Overview
## System Boundaries
## Technology Choices
## Component Map
## Data Model
## API Surface
## Integration Points
## Security Considerations
## Implementation Notes
```

### ADR

```markdown
---
id: "{NNN}"
type: adr
decision: "{decision-slug}"
status: accepted
created: {YYYY-MM-DD}
---

## Context
## Decision
## Consequences
## Alternatives Considered
```

### Inbox Message

```markdown
---
id: "{NNN}"
from: {role}
to: {role}
task-ref: "{task-id}"
type: info | design-issue | blocker
sent: {YYYY-MM-DDThh:mm}
read: false
---

{message body}
```

---

## PLAN.md Row Format

```
| {NNN} | {type} | {feature} | {status} | {relative-path} |
```

## LEDGER.md Row Format

```
| {NNN} | {title} | {owner} | {status} | {depends-on or —} | {blocks or —} |
```

Valid status values: `pending` | `ongoing` | `completed` | `paused` | `deadlocked`

---

## Movement Rules

| Event | Who moves | From → To |
|---|---|---|
| Task created | Team Lead | — → `tasks/pending/` |
| Specialist dispatched | Team Lead | `pending/` → `ongoing/` |
| Specialist completes | Team Lead | `ongoing/` → `completed/` |
| QA rejects | Team Lead | `completed/` → `ongoing/` |
| Task re-opened by user | Team Lead | `completed/` → `ongoing/` |
| Design superseded | Solution Architect | `plan/active/` → `plan/archived/` |
| Deliverable approved | QA Engineer | `deliverables/submitted/` → `deliverables/approved/` |
| Inbox message read | Specialist | `inbox/{role}/unread/` → `inbox/{role}/read/` |
| Design issue raised | Team Lead | set status `paused` in LEDGER |
| Design fix applied | Team Lead | `paused` → `ongoing/` |

**Team Lead is the only agent that writes `tasks/LEDGER.md` and moves task files.**
