# Token Reporting Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-agent and per-task token consumption tracking to the agent team pipeline, with the orchestrator as the single owner of all aggregation and reporting.

**Architecture:** Team Lead captures specialist token metrics from background completion notifications and writes them to task frontmatter + LEDGER. The orchestrator captures phase-level metrics from synchronous agent tool responses. After Phase 4, the orchestrator reads LEDGER + task files, computes the full picture, appends a Token Usage section to the session report, and displays it to the user.

**Tech Stack:** Markdown files with YAML frontmatter — no code, no build tooling. All changes are prompt/instruction edits to four files.

**Spec:** `docs/superpowers/specs/2026-03-22-token-reporting-design.md`

---

## File Map

| File | Change Type | What changes |
|------|-------------|--------------|
| `skills/task-template/SKILL.md` | Modify | Add `tokens`, `tool-uses`, `duration-ms`, `wave` to frontmatter schema |
| `skills/file-conventions/SKILL.md` | Modify | Add `tokens` and `duration-ms` columns to LEDGER row format |
| `agents/team-lead.md` | Modify | Task creation: add `wave` + token field init; On completion: insert `<usage>` capture before file move |
| `skills/orchestrate/SKILL.md` | Modify | Add `<usage>` capture after each phase dispatch; add post-Phase 4 token computation + report append + user display |

---

## Task 1: Add token fields to task-template schema

**Files:**
- Modify: `skills/task-template/SKILL.md`

- [ ] **Step 1: Read the file**

  Read `skills/task-template/SKILL.md` and confirm the current frontmatter ends with:
  ```yaml
  retry: 0
  blocked: false
  qa-cycles: 0
  ```

- [ ] **Step 2: Add four new fields after `qa-cycles: 0`**

  In the frontmatter template block, replace:
  ```yaml
  retry: 0
  blocked: false
  qa-cycles: 0
  ```
  With:
  ```yaml
  retry: 0
  blocked: false
  qa-cycles: 0
  tokens: ~
  tool-uses: ~
  duration-ms: ~
  wave: ~
  ```

- [ ] **Step 3: Verify**

  Re-read `skills/task-template/SKILL.md`. Confirm:
  - The four new fields appear after `qa-cycles: 0` inside the fenced code block
  - No other content was changed
  - The indentation and `~` sentinel values match exactly

- [ ] **Step 4: Commit**

  ```bash
  git add skills/task-template/SKILL.md
  git commit -m "feat: add token tracking fields to task-template schema"
  ```

---

## Task 2: Update LEDGER row format in file-conventions

**Files:**
- Modify: `skills/file-conventions/SKILL.md`

- [ ] **Step 1: Read the file**

  Read `skills/file-conventions/SKILL.md` and locate the `## LEDGER.md Row Format` section. Confirm it currently reads:
  ```
  | {NNN} | {title} | {owner} | {status} | {depends-on or —} | {blocks or —} |
  ```

- [ ] **Step 2: Update the LEDGER row format and add sentinel note**

  Replace the `## LEDGER.md Row Format` section:

  **Before:**
  ```markdown
  ## LEDGER.md Row Format

  ```
  | {NNN} | {title} | {owner} | {status} | {depends-on or —} | {blocks or —} |
  ```
  ```

  **After:**
  ```markdown
  ## LEDGER.md Row Format

  ```
  | {NNN} | {title} | {owner} | {status} | {depends-on or —} | {blocks or —} | {tokens or ~} | {duration-ms or ~} |
  ```

  `tokens` and `duration-ms` are initialised to `~` at task creation and populated by Team Lead when a task reaches `completed` status. Tasks that do not complete retain `~`. `tool-uses` is intentionally excluded from LEDGER — it is stored in the task file only.
  ```

- [ ] **Step 3: Verify**

  Re-read `skills/file-conventions/SKILL.md`. Confirm:
  - The LEDGER row format now has 8 columns (was 6)
  - The sentinel note explains `~`, population timing, and `tool-uses` exclusion
  - No other sections were modified

- [ ] **Step 4: Commit**

  ```bash
  git add skills/file-conventions/SKILL.md
  git commit -m "feat: add tokens and duration-ms columns to LEDGER row format"
  ```

---

## Task 3: Update Team Lead — task creation protocol

**Files:**
- Modify: `agents/team-lead.md`

- [ ] **Step 1: Read the file**

  Read `agents/team-lead.md` and locate the `### Task File Creation` section. Confirm it currently reads:

  ```markdown
  ### Task File Creation

  For each task:
  1. Create `tasks/pending/{NNN}-task-{slug}.md` using `Skill("team/references/task-template")`
  2. Fill all frontmatter: id, title, owner, status, depends-on, blocks
  3. Write Context (from design), Scope (exact deliverable), Acceptance Criteria (measurable)
  4. List dependency file paths under `## Dependencies Available At`

  Then write `tasks/LEDGER.md` with all tasks and their full dependency graph.
  ```

- [ ] **Step 2: Add wave assignment and token field initialisation**

  Replace the `### Task File Creation` section with:

  ```markdown
  ### Task File Creation

  For each task:
  1. Create `tasks/pending/{NNN}-task-{slug}.md` using `Skill("team/references/task-template")`
  2. Fill all frontmatter: id, title, owner, status, depends-on, blocks
  3. Set `wave` based on dispatch-wave assignment:
     - Wave 1: `db-engineer` and `ui-ux-designer`
     - Wave 2: `backend-engineer`
     - Wave 3: `frontend-engineer`
     - Wave 4: `qa-engineer`
  4. Initialise `tokens: ~`, `tool-uses: ~`, `duration-ms: ~` in frontmatter
  5. Write Context (from design), Scope (exact deliverable), Acceptance Criteria (measurable)
  6. List dependency file paths under `## Dependencies Available At`

  Then write `tasks/LEDGER.md` with all tasks and their full dependency graph. Initialise `tokens` and `duration-ms` columns as `~` for every row.
  ```

- [ ] **Step 3: Verify**

  Re-read `agents/team-lead.md`. Confirm:
  - `wave` assignment step lists the correct role-to-wave mapping
  - Token fields are initialised to `~`
  - LEDGER creation note mentions initialising `tokens` and `duration-ms` as `~`
  - Step numbering is sequential (1–6) with no gaps
  - No other sections were modified

- [ ] **Step 4: Commit**

  ```bash
  git add agents/team-lead.md
  git commit -m "feat: team-lead sets wave and initialises token fields at task creation"
  ```

---

## Task 4: Update Team Lead — on specialist completion protocol

**Files:**
- Modify: `agents/team-lead.md`

- [ ] **Step 1: Read the file**

  Read `agents/team-lead.md` and locate the `### On Specialist Completion` section. Confirm step 2 currently reads:
  ```markdown
  2. If RESULT.md exists:
     a. Move task file: `tasks/ongoing/` → `tasks/completed/`
     b. Update `LEDGER.md`: status → `completed`, set `completed` date in task frontmatter
     c. Check LEDGER for tasks whose dependencies are now all completed
     d. Dispatch next unblocked wave
  ```

- [ ] **Step 2: Insert token capture steps between RESULT.md check and file move**

  Replace the entire `### On Specialist Completion` section with:

  ```markdown
  ### On Specialist Completion

  When a specialist notifies completion via `inbox/team-lead/unread/`:

  1. Check whether `deliverables/submitted/{task-id}/RESULT.md` exists
     - If it does NOT exist:
       a. Read `retry` value from task frontmatter
       b. If `retry < 2`: increment `retry` in task frontmatter, re-dispatch specialist with same task file
       c. If `retry >= 2`: escalate to SKILL.md orchestrator: `RETRY_EXCEEDED: {task-id}` — do not re-dispatch
       d. **Stop processing this completion** — do not move task file
  2. Extract `total_tokens`, `tool_uses`, `duration_ms` from the `<usage>` block in the completion notification
  3. Write to the task file at `tasks/ongoing/{task-id}.md` (before moving it):
     - `tokens: {total_tokens}`
     - `tool-uses: {tool_uses}`
     - `duration-ms: {duration_ms}`
  4. Update the LEDGER.md row for this task: set `tokens` and `duration-ms` columns to the extracted values
  5. Move task file: `tasks/ongoing/` → `tasks/completed/`
  6. Update `LEDGER.md`: status → `completed`, set `completed` date in task frontmatter
  7. Check LEDGER for tasks whose dependencies are now all completed
  8. Dispatch next unblocked wave

  **On QA rejection re-dispatch:** When QA rejects a task and the specialist completes again, steps 2–4 overwrite the token fields with the latest run's values (last-write wins). This is correct — the final delivery's cost is what matters.
  ```

- [ ] **Step 3: Verify**

  Re-read `agents/team-lead.md`. Confirm:
  - Steps 2, 3, 4 are new (token capture) and appear before step 5 (move)
  - Step 3 explicitly writes to `tasks/ongoing/{task-id}.md` (pre-move path)
  - The QA re-dispatch note is present and explains last-write-wins
  - Steps 5–8 match the previous steps 2a–2d exactly
  - Step 1 (RESULT.md check) is unchanged

- [ ] **Step 4: Commit**

  ```bash
  git add agents/team-lead.md
  git commit -m "feat: team-lead captures specialist token usage on completion"
  ```

---

## Task 5: Update orchestrate — phase-level usage capture

**Files:**
- Modify: `skills/orchestrate/SKILL.md`

- [ ] **Step 1: Read the file**

  Read `skills/orchestrate/SKILL.md`. Note the current endings of each phase section:
  - Phase 1 ends: `Wait for brief to be confirmed at plan/active/{NNN}-brief-{slug}.md.`
  - Phase 2 ends: `Wait for design outputs at plan/active/{NNN}-design-{slug}.md and any ADR files.`
  - Phase 3 ends at the `### On DESIGN_ISSUE` block
  - Phase 4 ends: `Wait for session report at reports/sessions/{YYYY-MM-DD}-session.md and confirmation that reports/STATUS.md is updated.`

- [ ] **Step 2: Add usage capture to Phase 1**

  Replace:
  ```markdown
  Wait for brief to be confirmed at `plan/active/{NNN}-brief-{slug}.md`.
  ```
  With:
  ```markdown
  Wait for brief to be confirmed at `plan/active/{NNN}-brief-{slug}.md`.

  Capture `<usage>` from this Agent call as `phase1_usage` (tokens, tool-uses, duration-ms). If `<usage>` is absent, set `phase1_usage` to zeros.
  ```

- [ ] **Step 3: Add usage capture to Phase 2**

  Replace:
  ```markdown
  Wait for design outputs at `plan/active/{NNN}-design-{slug}.md` and any ADR files.
  ```
  With:
  ```markdown
  Wait for design outputs at `plan/active/{NNN}-design-{slug}.md` and any ADR files.

  Capture `<usage>` from this Agent call as `phase2_usage` (tokens, tool-uses, duration-ms). If the Architect is re-dispatched for revisions, accumulate each call's usage into `phase2_usage`. If `<usage>` is absent, set to zeros.
  ```

- [ ] **Step 4: Add usage capture to Phase 3**

  In the Phase 3 section, insert the following after the last line of the `### On DESIGN_ISSUE` block (`If `override`: Instruct Team Lead to resume with paused tasks.`) and before the `---` separator. Add a blank line between the existing content and the new text:

  ```markdown
  Capture `<usage>` from the Team Lead Agent call as `phase3_usage` (tokens, tool-uses, duration-ms). If Team Lead is re-dispatched after a design fix, accumulate each call's usage into `phase3_usage`. If `<usage>` is absent, set to zeros.
  ```

- [ ] **Step 5: Add usage capture to Phase 4**

  Replace:
  ```markdown
  Wait for session report at `reports/sessions/{YYYY-MM-DD}-session.md`
  and confirmation that `reports/STATUS.md` is updated.
  ```
  With:
  ```markdown
  Wait for session report at `reports/sessions/{YYYY-MM-DD}-session.md`
  and confirmation that `reports/STATUS.md` is updated.

  Capture `<usage>` from this Agent call as `phase4_usage` (tokens, tool-uses, duration-ms). Record the session report path: use the explicit path returned by PM if available; otherwise derive it as the newest file in `reports/sessions/` (do not re-derive from the current date independently). If `<usage>` is absent, set to zeros.
  ```

- [ ] **Step 6: Verify**

  Re-read `skills/orchestrate/SKILL.md`. Confirm:
  - All four phases have `<usage>` capture instructions
  - Variable names are `phase1_usage` through `phase4_usage`
  - Each has an "if absent, set to zeros" fallback
  - Phase 2 and Phase 3 note accumulation for re-dispatches
  - Phase 4 records the session report path for use in Completion

- [ ] **Step 7: Commit**

  ```bash
  git add skills/orchestrate/SKILL.md
  git commit -m "feat: orchestrate captures phase-level token usage after each agent dispatch"
  ```

---

## Task 6: Update orchestrate — post-Phase 4 token computation and display

**Files:**
- Modify: `skills/orchestrate/SKILL.md`

- [ ] **Step 1: Read the Completion section**

  Read `skills/orchestrate/SKILL.md` and locate the `## Completion` section. Confirm it currently reads:
  ```markdown
  ## Completion

  After reporting completes:

  → Invoke `Skill("superpowers:finishing-a-development-branch")`

  This skill will assess what was built and recommend: create PR, merge directly, or discard.
  Follow its guidance to close out the branch.
  ```

- [ ] **Step 2: Insert token computation block before the finishing skill**

  Replace the `## Completion` section with:

  ```markdown
  ## Token Computation

  After Phase 4 completes and before invoking the finishing skill:

  ### Step 1: Read specialist task data

  1. Read `tasks/LEDGER.md` — for each task row, extract task ID, `tokens`, `duration-ms`
  2. Read all task files in `tasks/completed/` — for each file, extract task ID, `tool-uses`, `wave`, `qa-cycles`
  3. Join on task ID. Skip any task where:
     - No matching file exists in `tasks/completed/` (task did not complete)
     - `tokens` is `~` (task did not receive a `<usage>` block)

  ### Step 2: Compute metrics

  Using specialist task data (step 1) and phase usage variables (`phase1_usage`–`phase4_usage`):

  **Per-task rows** (one per completed specialist task):
  - Agent role, task ID + title, tokens, tool-uses, duration formatted as `Xm Ys`, `tokens_per_tool_use = tokens / tool-uses` (render `—` if tool-uses is 0)

  **Per-phase rows** (one per phase with non-zero usage):
  - Agent name, phase label (e.g. "Phase 1: Intent"), tokens, tool-uses, duration from `phaseN_usage`

  **Session total:** sum of all task tokens + all phase tokens; sum of tool-uses; sum of duration-ms

  **Most expensive item:** highest-token row across tasks and phases, with percentage of session total

  **QA re-run overhead** (omit if no tasks have `qa-cycles > 0`):
  - For each task where `qa-cycles > 0`: estimate re-run cost = `tokens × (qa-cycles / (qa-cycles + 1))`
  - Sum all per-task estimates → single aggregate figure; label as estimated

  **Wave bottleneck** (only consider waves with ≥ 1 completed task):
  - Group completed tasks by `wave` field
  - Bottleneck = wave with highest summed `duration-ms`
  - Report: wave number, role(s), total duration

  ### Step 3: Append Token Usage section to session report

  Use the session report path recorded from Phase 4 (do not re-derive from current date).
  Read the file and append after all existing sections:

  ~~~markdown
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

  **Session total:** {N,NNN} tokens · {N} tool uses · {Xm Ys}

  **Most expensive:** {agent}/{task-or-phase} ({N,NNN} tokens — {N}% of session)

  **QA re-run overhead:** {N} tasks had QA cycles — est. {N,NNN} tokens in rejected work ({N}% of session)
  *(omit if no QA re-runs occurred)*

  **Wave bottleneck:** Wave {N} ({role(s)}) — {Xm Ys} total
  ~~~

  ### Step 4: Display to user

  Print the full Token Usage section (exactly as appended) to the user inline.

  ---

  ## Completion

  After token computation and display:

  → Invoke `Skill("superpowers:finishing-a-development-branch")`

  This skill will assess what was built and recommend: create PR, merge directly, or discard.
  Follow its guidance to close out the branch.
  ```

- [ ] **Step 3: Verify**

  Re-read `skills/orchestrate/SKILL.md`. Confirm:
  - `## Token Computation` section appears between Phase 4 and `## Completion`
  - All six computation metrics are present (per-task, per-phase, session total, most expensive, QA overhead, wave bottleneck)
  - The session report path note is present ("do not re-derive from current date")
  - The report template uses `~~~markdown` fences (to avoid nesting conflict with the outer markdown)
  - `## Completion` still exists and still invokes `finishing-a-development-branch`
  - No phase sections were accidentally modified

- [ ] **Step 4: Commit**

  ```bash
  git add skills/orchestrate/SKILL.md
  git commit -m "feat: orchestrate computes and displays token usage after reporting phase"
  ```

---

## Task 7: Cross-file consistency check

- [ ] **Step 1: Verify field names are consistent across all four files**

  Read all four modified files and confirm:
  - `skills/task-template/SKILL.md` has fields: `tokens`, `tool-uses`, `duration-ms`, `wave`
  - `skills/file-conventions/SKILL.md` LEDGER row has columns: `tokens`, `duration-ms` (not `tool-uses`)
  - `agents/team-lead.md` writes `tokens`, `tool-uses`, `duration-ms` to task file (step 3) and `tokens`, `duration-ms` to LEDGER (step 4)
  - `skills/orchestrate/SKILL.md` reads `tokens`, `duration-ms` from LEDGER and `tool-uses`, `wave`, `qa-cycles` from task files

- [ ] **Step 2: Verify wave mapping is consistent**

  Confirm the wave mapping in the `### Task File Creation` section of `agents/team-lead.md` (written by Task 3) matches the `### Dispatch Waves` table in the same file (lines ~43–48, unchanged by this plan):
  - Wave 1: db-engineer + ui-ux-designer
  - Wave 2: backend-engineer
  - Wave 3: frontend-engineer
  - Wave 4: qa-engineer

- [ ] **Step 3: Commit if any corrections were made**

  ```bash
  git add .
  git commit -m "fix: correct cross-file field name consistency"
  ```
  (Skip this step if no corrections needed.)
