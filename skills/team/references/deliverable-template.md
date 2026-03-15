# Deliverable Template

Specialists write RESULT.md to `deliverables/submitted/{task-id}/RESULT.md` on completion.
QA Engineer moves approved deliverables to `deliverables/approved/{task-id}/RESULT.md`.

---

```markdown
---
task-id: "{NNN}"
task-title: "{title}"
owner: {role}
submitted: {YYYY-MM-DDThh:mm}
qa-status: pending
---

## Summary

One paragraph describing what was built and any key decisions made.

## Acceptance Criteria Status

- [x] {criterion 1} — completed (cite specific file:line or evidence)
- [x] {criterion 2} — completed (cite specific file:line or evidence)
- [ ] {criterion N if incomplete} — explain why and what is needed

## Artifacts

List every file created or modified:

- `{file-path}` — {what it is and what it does}

## Implementation Notes

Key decisions made during implementation, alternatives rejected, constraints encountered.
Be specific — this is read by QA and the next specialist who depends on this deliverable.

## Verification Evidence

- Tests run: `{command}` → `{output showing pass}`
- Any manual checks performed
```
