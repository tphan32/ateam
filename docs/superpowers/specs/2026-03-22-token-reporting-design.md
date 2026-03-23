# Token Reporting for Agent Team — Design Specification

**Date:** 2026-03-22
**Status:** Draft
**Feature slug:** token-reporting

---

## 1. Overview

Add token consumption tracking and reporting to the agent team pipeline. Each specialist agent's token usage is captured automatically from task completion notifications, stored in the existing task file structure, and aggregated by the Project Manager into a Token Usage section in the session report.

### Goals

- Cost awareness: know how many tokens each session consumes in total and per agent
- Optimization insight: identify which agents and tasks are disproportionately expensive
- Zero friction: no changes to specialist agents; Team Lead captures everything from existing notification data

### Constraints

- Token data comes from `<usage>` blocks in task completion notifications (confirmed via live test: `total_tokens`, `tool_uses`, `duration_ms`)
- No hooks, no transcript parsing, no external infrastructure
- All storage stays within the existing `.team/` file structure
- Estimates only for QA re-run overhead (derived by cross-referencing `qa-cycles` in LEDGER)
- Team Lead runs as a synchronous `Agent` call (not background), so its own token usage is not available via completion notifications and is not tracked

---

## 2. Data Flow

```
Background specialist agent completes
    → task-notification includes <usage> block
    → Team Lead extracts total_tokens, tool_uses, duration_ms
    → writes to task file frontmatter (at tasks/ongoing/{task-id}.md, before move)
    → updates LEDGER.md tokens and duration-ms columns for that task
    → moves task file to tasks/completed/
    → PM reads LEDGER.md + all task files at report time
    → skips tasks with tokens: ~ (incomplete/not-yet-run)
    → computes derived metrics
    → appends Token Usage section to session report
```

---

## 3. Schema Changes

### 3.1 Task File Frontmatter

Four new fields added to `skills/task-template/SKILL.md`:
- `tokens`, `tool-uses`, `duration-ms` — populated by Team Lead from `<usage>` block on completion
- `wave` — populated by Team Lead at task creation time, based on dispatch-wave assignment

```yaml
tokens: ~          # total_tokens from <usage> block; ~ until task completes
tool-uses: ~       # tool_uses from <usage> block; ~ until task completes
                   # (stored in task file only — not propagated to LEDGER, intentional)
duration-ms: ~     # duration_ms from <usage> block; ~ until task completes
wave: ~            # dispatch wave number (1, 2, 3, 4); set by Team Lead at task creation
```

The `tool-uses` field is intentionally excluded from LEDGER — it is available in the task file for per-task detail but not needed for the LEDGER summary view.

### 3.2 LEDGER.md Row Format

Two new columns appended to the existing row format in `skills/file-conventions/SKILL.md`:

```
| NNN | title | owner | status | depends-on | blocks | tokens | duration-ms |
```

Both new columns are initialised to `~` when Team Lead creates the task row, and populated with the actual values when the task reaches `completed` status. Tasks that did not complete in the session retain `~`.

---

## 4. Team Lead Behaviour Change

### 4.1 At Task Creation

When writing each task file and adding its row to LEDGER.md, Team Lead also:
- Sets `wave: {N}` in the task frontmatter using the dispatch-wave assignment rules (Wave 1: db-engineer + ui-ux-designer; Wave 2: backend-engineer; Wave 3: frontend-engineer; Wave 4: qa-engineer)
- Initialises `tokens: ~`, `tool-uses: ~`, `duration-ms: ~` in the task frontmatter
- Initialises `tokens` and `duration-ms` columns as `~` in the LEDGER row

### 4.2 On Specialist Completion

In the "On Specialist Completion" protocol, after confirming `RESULT.md` exists and **before moving the task file** (while it is still at `tasks/ongoing/{task-id}.md`):

1. Extract `total_tokens`, `tool_uses`, `duration_ms` from the `<usage>` block in the completion notification
2. Write to the task file at `tasks/ongoing/{task-id}.md`:
   - `tokens: {total_tokens}`
   - `tool-uses: {tool_uses}`
   - `duration-ms: {duration_ms}`
3. Update the LEDGER.md row for this task: set `tokens` and `duration-ms` columns
4. Proceed with existing move (`tasks/ongoing/` → `tasks/completed/`) and downstream dispatch logic

Team Lead's own token usage is not tracked. Team Lead runs as a synchronous Agent call, not a background task, so no `<usage>` notification is emitted for it.

---

## 5. Project Manager Behaviour Change

### 5.1 On Start (additions)

In addition to the existing reads, PM also reads all task files in `tasks/completed/` to retrieve `tool-uses`, `wave`, and `qa-cycles` values (not stored in LEDGER).

### 5.2 Compute

Tasks with `tokens: ~` are skipped in all aggregations (they did not complete this session). All computed totals apply only to tasks with a numeric `tokens` value.

**Wave identification:** PM infers wave groupings from the `wave` field in each task file (set by Team Lead at creation). No hard-coded role-to-wave mapping is required.

Metrics to compute:

- **Per-task row:** agent role, task id/title, tokens, tool-uses, duration (formatted as `Xm Ys`), `tokens_per_tool_use = tokens / tool-uses` (omit column if tool-uses is 0)
- **Session total:** sum of tokens, tool-uses, and duration-ms across all completed tasks
- **Most expensive task:** task with highest token count, shown with percentage of session total
- **QA re-run overhead:** for each task where `qa-cycles > 0`, estimate re-run cost as `tokens × (qa-cycles / (qa-cycles + 1))`. Sum these per-task estimates into a single aggregate figure. Label the result as estimated. Omit this line if no tasks have `qa-cycles > 0`.
- **Wave bottleneck:** group tasks by `wave` value; the bottleneck is the wave whose total `duration-ms` (sum of all tasks in that wave) is highest. Report the wave number, agent role(s), and total duration.

### 5.3 Report Section

Append after the existing SCQA + Next Steps sections. This section is **excluded from the 325–475 word count** that applies to the narrative sections.

```markdown
## Token Usage

| Agent | Task | Tokens | Tool Uses | Duration | Tokens/Tool |
|-------|------|--------|-----------|----------|-------------|
| {role} | {NNN}-{slug} | {N,NNN} | {N} | {Xm Ys} | {N,NNN} |
...

**Session total:** {N,NNN} tokens · {N} tool uses · {Xm Ys}

**Most expensive task:** {role}/{task-id} ({N,NNN} tokens — {N}% of session)

**QA re-run overhead:** {N} tasks had QA cycles — est. {N,NNN} tokens in rejected work ({N}% of session)
*(omit line if no QA re-runs occurred)*

**Wave bottleneck:** Wave {N} ({role(s)}) — {Xm Ys} total
```

---

## 6. Files Changed

| File | Change |
|------|--------|
| `skills/task-template/SKILL.md` | Add `tokens`, `tool-uses`, `duration-ms`, `wave` fields to frontmatter schema |
| `skills/file-conventions/SKILL.md` | Add `tokens` and `duration-ms` columns to LEDGER.md row format; note `~` sentinel for both |
| `agents/team-lead.md` | At task creation: set `wave` and initialise token fields. On completion: extract `<usage>`, write metrics before moving task file, update LEDGER |
| `agents/team-project-manager.md` | Add task file reads to On Start; add token aggregation logic; add Token Usage section (word-count exempt) to session report |

No changes to: specialist agent files, `skills/deliverable-template/SKILL.md`, `skills/orchestrate/SKILL.md`, `skills/execution/SKILL.md`.

`skills/reporting/SKILL.md` requires no changes — it governs which agent is dispatched and which index files are passed, not the internal format of the session report. Report format is governed by `agents/team-project-manager.md`.

---

## 7. Out of Scope

- Input vs output token split (notification only provides `total_tokens`)
- Real-time token display during a run
- Token budget enforcement or hard limits
- Historical trend charts across sessions
- Team Lead token tracking (runs synchronously, no `<usage>` notification emitted)
