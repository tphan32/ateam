---
name: team-init
description: .team/ directory initialization — run on first /team invocation in any project
---

# .team/ Directory Initialization

Run this initialization on the first `/team` invocation in any project.

## Check

```bash
ls .team/ 2>/dev/null || echo "NOT_INITIALIZED"
```

If output is `NOT_INITIALIZED`, create the full directory structure below.

## Directory Structure to Create

```
.team/
├── plan/
│   ├── PLAN.md
│   ├── active/
│   └── archived/
├── tasks/
│   ├── LEDGER.md
│   ├── pending/
│   ├── ongoing/
│   └── completed/
├── inbox/
│   ├── backend/
│   │   ├── unread/
│   │   └── read/
│   ├── frontend/
│   │   ├── unread/
│   │   └── read/
│   ├── db/
│   │   ├── unread/
│   │   └── read/
│   ├── ui-ux/
│   │   ├── unread/
│   │   └── read/
│   ├── qa/
│   │   ├── unread/
│   │   └── read/
│   └── team-lead/
│       ├── unread/
│       └── read/
├── deliverables/
│   ├── DELIVERABLES.md
│   ├── submitted/
│   └── approved/
└── reports/
    ├── STATUS.md
    └── sessions/
```

## Index File Initial Content

**`.team/plan/PLAN.md`:**

```markdown
# Plan Index

| ID  | Type   | Feature   | Status   | File |
|-----|--------|-----------|----------|------|
```

**`.team/tasks/LEDGER.md`:**

```markdown
---
feature: ""
created: {YYYY-MM-DD}
last-updated: {YYYY-MM-DD}
---

| ID  | Title | Owner | Status | Depends On | Blocks |
|-----|-------|-------|--------|------------|--------|
```

**`.team/deliverables/DELIVERABLES.md`:**

```markdown
# Deliverables Index

| Task | Owner | Status | Summary |
|------|-------|--------|---------|
```

**`.team/reports/STATUS.md`:**

```markdown
# Project Status

*No sessions yet.*
```

## ID Numbering

- All IDs are zero-padded three-digit integers: `001`, `002`, `003`
- Assigned sequentially by Team Lead at planning time
- Source of truth: highest existing ID in `tasks/LEDGER.md`
- IDs are unique across the lifetime of a `.team/` directory (never reused)
