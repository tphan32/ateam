# Token Reporting for Agent Team — Design Specification

**Date:** 2026-03-23
**Status:** Draft
**Feature slug:** token-reporting

---

## 1. Overview

Add token consumption tracking and reporting to the agent team pipeline. Token usage is captured at every agent boundary and accumulated by the orchestrator — the single owner of all token data. After Phase 4 completes, the orchestrator computes the full session picture, appends a Token Usage section to the session report, and displays a summary to the user.

### Goals

- Cost awareness: know how many tokens each session consumes in total and per agent/phase
- Optimization insight: identify which agents and tasks are disproportionately expensive
- Single responsibility: orchestrator owns all token computation; no agent duplicates this logic

### Design pattern

Matches the established multi-agent framework pattern (OpenAI Agents SDK, LangGraph, CrewAI):
- **Capture at the unit-of-work boundary** — each agent completion emits `<usage>`
- **Accumulate at the orchestration layer** — orchestrator aggregates; agents do not self-compute
- **Two-level model** — per-task detail preserved + session-level aggregate always available

### Constraints

- Token data comes from `<usage>` blocks in agent completion responses (confirmed via live test: `total_tokens`, `tool_uses`, `duration_ms`)
- `<usage>` is present on both synchronous Agent calls (inline in tool response) and background Agent calls (in task notification) — confirmed via live test
- Only `total_tokens` is available — no input/output split (platform constraint)
- All storage stays within the existing `.team/` file structure
- PM agent is unchanged — it writes the SCQA narrative report only

---

## 2. Data Flow

```
Phase 1: Orchestrator dispatches User Proxy (sync)
    → captures <usage> from tool response

Phase 2: Orchestrator dispatches Solution Architect (sync)
    → captures <usage> from tool response

Phase 3: Orchestrator dispatches Team Lead (sync, long-running)
    Team Lead internally dispatches specialists (background)
        → on each specialist completion notification, extracts <usage>
        → writes tokens, tool-uses, duration-ms to task file frontmatter (before move)
        → updates LEDGER.md tokens + duration-ms columns
    → Orchestrator captures Team Lead's own <usage> from tool response

Phase 4: Orchestrator dispatches PM (sync)
    PM writes SCQA narrative report (no token logic)
    → Orchestrator captures PM's own <usage> from tool response

Post-Phase 4: Orchestrator computes session token picture
    → reads LEDGER.md + all task files in tasks/completed/ (for specialist breakdown)
    → combines with phase-level <usage> captured in Phases 1–4
    → computes derived metrics
    → appends Token Usage section to session report
    → displays token summary to user
```

---

## 3. Schema Changes

### 3.1 Task File Frontmatter

Four new fields added to `skills/task-template/SKILL.md`:

```yaml
tokens: ~          # total_tokens from <usage> block; ~ until task completes
tool-uses: ~       # tool_uses from <usage> block; ~ until task completes
                   # (stored in task file only — not propagated to LEDGER, intentional)
duration-ms: ~     # duration_ms from <usage> block; ~ until task completes
wave: ~            # dispatch wave number (1–4); set by Team Lead at task creation
```

`tool-uses` is intentionally excluded from LEDGER — it is available per-task in the task file but is not needed for the LEDGER summary view.

### 3.2 LEDGER.md Row Format

Two new columns appended to the existing row format in `skills/file-conventions/SKILL.md`:

```
| NNN | title | owner | status | depends-on | blocks | tokens | duration-ms |
```

Both columns are initialised to `~` at task creation and populated when the task reaches `completed` status. Tasks that did not complete in the session retain `~`.

---

## 4. Team Lead Behaviour Change

Team Lead's only token responsibility is capturing specialist `<usage>` from background completion notifications.

### 4.1 At Task Creation

When writing each task file and adding its LEDGER row, Team Lead also:
- Sets `wave: {N}` in the task frontmatter (Wave 1: db-engineer + ui-ux-designer; Wave 2: backend-engineer; Wave 3: frontend-engineer; Wave 4: qa-engineer)
- Initialises `tokens: ~`, `tool-uses: ~`, `duration-ms: ~` in the task frontmatter
- Initialises `tokens` and `duration-ms` columns as `~` in the LEDGER row

### 4.2 On Specialist Completion

The existing "On Specialist Completion" protocol begins by confirming `RESULT.md` exists, then moves the task file at step 2a, then updates LEDGER at step 2b. The new token capture steps insert **between the RESULT.md check and the file move**. The updated sequence is:

1. *(existing)* Check `RESULT.md` exists — if not, apply retry logic and stop
2. *(new)* Extract `total_tokens`, `tool_uses`, `duration_ms` from the `<usage>` block in the completion notification
3. *(new)* Write to the task file at `tasks/ongoing/{task-id}.md` (still at its pre-move path):
   - `tokens: {total_tokens}`
   - `tool-uses: {tool_uses}`
   - `duration-ms: {duration_ms}`
4. *(new)* Update the LEDGER.md row: set `tokens` and `duration-ms` columns
5. *(existing, formerly 2a)* Move task file: `tasks/ongoing/` → `tasks/completed/`
6. *(existing, formerly 2b)* Update LEDGER status → `completed`, set `completed` date
7. *(existing)* Check LEDGER for newly unblocked tasks and dispatch next wave

**On QA rejection re-dispatch:** When QA rejects a task, Team Lead moves it back to `tasks/ongoing/` and re-dispatches the specialist. When the specialist completes again, steps 2–4 above overwrite the token fields with the new run's values (last-write wins). This is the correct behaviour — the final delivery's cost is what matters for reporting.

Team Lead does not aggregate, compute, or report token totals — that is the orchestrator's responsibility.

---

## 5. Orchestrator Behaviour Change

Changes to `skills/orchestrate/SKILL.md`.

### 5.1 Capture Phase-Level Usage

After each synchronous Agent dispatch, the orchestrator extracts `<usage>` from the tool response and retains it in memory for the post-Phase 4 computation step:

| Phase | Agent | Variable |
|-------|-------|----------|
| Phase 1 | user-proxy | `phase1_usage` |
| Phase 2 | team-solution-architect | `phase2_usage` |
| Phase 3 | team-lead | `phase3_usage` |
| Phase 4 | team-project-manager | `phase4_usage` |

If a `<usage>` block is absent for any phase (e.g. the agent errored or the response was malformed), treat that phase's tokens, tool-uses, and duration-ms as 0 and omit the row from the Token Usage table rather than failing the computation.

### 5.2 Post-Phase 4: Compute Token Picture

After PM completes (and before invoking `finishing-a-development-branch`):

1. Read `tasks/LEDGER.md` — collect all task rows; extract `tokens` and `duration-ms` per task ID
2. Read all task files in `tasks/completed/` — extract `tool-uses`, `wave`, `qa-cycles` per task ID
3. Join on task ID: for each task ID in LEDGER, look up its task file in `tasks/completed/`. If no matching file exists (task is paused, deadlocked, or otherwise not in `completed/`), skip it — the `tokens: ~` sentinel also covers this case
4. Skip any task where `tokens` is `~` in all aggregations
5. Combine specialist task data with phase-level usage captured in §5.1

**Derived metrics:**
- **Per-task rows:** role, task id/title, tokens, tool-uses, duration (formatted `Xm Ys`), `tokens_per_tool_use = tokens / tool-uses` (render as `—` if tool-uses is 0; the column is always present)
- **Per-phase rows:** agent name, phase label, tokens, tool-uses, duration from `phaseN_usage`
- **Session total:** sum of all task tokens + all phase tokens; sum of tool-uses; sum of duration-ms
- **Most expensive item:** highest-token row across both tasks and phases, with percentage of session total
- **QA re-run overhead:** for each task where `qa-cycles > 0`, estimate re-run cost as `tokens × (qa-cycles / (qa-cycles + 1))`; sum all per-task estimates into one aggregate figure; label as estimated; omit if no tasks have `qa-cycles > 0`
- **Wave bottleneck:** group completed tasks by `wave` field; only consider waves with at least one completed task; the bottleneck is the wave with highest summed `duration-ms`; report wave number, role(s), and total duration

### 5.3 Append to Session Report

The PM writes the session report to `reports/sessions/{YYYY-MM-DD}-session.md` and returns its path (or the orchestrator derives the path as the newest file in `reports/sessions/` after PM completes). The orchestrator reads that file and appends the Token Usage section. The orchestrator does **not** independently compute `{YYYY-MM-DD}` from the current date — it uses the path from PM's output to avoid midnight-boundary mismatches.

The Token Usage section is **excluded from the 325–475 word count** that governs the PM's SCQA narrative. The PM's word-count rule applies only to the Situation/Complication/Question/Answer/Next Steps sections it writes. The Token Usage section is appended by the orchestrator after PM has finished and is not part of PM's output.

```markdown
## Token Usage

### By Phase

| Agent | Phase | Tokens | Tool Uses | Duration |
|-------|-------|--------|-----------|----------|
| user-proxy | Phase 1: Intent | {N,NNN} | {N} | {Xm Ys} |
| solution-architect | Phase 2: Design | {N,NNN} | {N} | {Xm Ys} |
| team-lead | Phase 3: Execution | {N,NNN} | {N} | {Xm Ys} |
| project-manager | Phase 4: Reporting | {N,NNN} | {N} | {Xm Ys} |

### By Task

| Agent | Task | Tokens | Tool Uses | Duration | Tokens/Tool |
|-------|------|--------|-----------|----------|-------------|
| {role} | {NNN}-{slug} | {N,NNN} | {N} | {Xm Ys} | {N,NNN} |
...

**Session total:** {N,NNN} tokens · {N} tool uses · {Xm Ys}

**Most expensive:** {agent}/{task-or-phase} ({N,NNN} tokens — {N}% of session)

**QA re-run overhead:** {N} tasks had QA cycles — est. {N,NNN} tokens in rejected work ({N}% of session)
*(omit if no QA re-runs occurred)*

**Wave bottleneck:** Wave {N} ({role(s)}) — {Xm Ys} total
```

### 5.4 Display Summary to User

After appending to the session report, print the Token Usage section to the user inline. No additional formatting beyond what is in the report.

---

## 6. Files Changed

| File | Change |
|------|--------|
| `skills/task-template/SKILL.md` | Add `tokens`, `tool-uses`, `duration-ms`, `wave` fields to frontmatter schema |
| `skills/file-conventions/SKILL.md` | Add `tokens` and `duration-ms` columns to LEDGER.md row format; note `~` sentinel |
| `agents/team-lead.md` | At task creation: set `wave`, initialise token fields. On completion: extract `<usage>` from notification, write to task file before move, update LEDGER |
| `skills/orchestrate/SKILL.md` | Capture `<usage>` from all four phase agents; post-Phase 4: read LEDGER + task files, compute full token picture, append to session report, display to user |

**No changes to:**
- `agents/team-project-manager.md` — PM writes SCQA narrative only; token computation is not its responsibility. PM's existing 325–475 word limit applies to its narrative output only; the Token Usage section is appended by the orchestrator and is outside PM's scope.
- Specialist agent files (`team-backend-engineer.md`, etc.)
- `skills/deliverable-template/SKILL.md`
- `skills/execution/SKILL.md`
- `skills/reporting/SKILL.md` — governs which agent is dispatched, not report format

---

## 7. Out of Scope

- Input vs output token split (platform only provides `total_tokens`)
- Real-time token display during a run
- Token budget enforcement or hard limits
- Historical trend charts across sessions
