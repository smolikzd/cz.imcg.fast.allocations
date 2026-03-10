---
story: "1.2"
title: "Remove WRITE: statement from CORR_BCHE"
epic: "1 — Unblock Pipeline Activation"
sprint: 1
status: done
priority: critical
estimate: 5min
source_findings:
  - CC-12
  - cross-cutting-analysis.md
  - zfi_alloc_corr_bche-migration-review.md
target_repo: smolikzd/cz.imcg.fast.planner
---

# Story 1.2: Remove `WRITE:` statement from CORR_BCHE

## User Story

As a pipeline operator,
I want CORR_BCHE to execute cleanly in background mode,
so that step 0003 does not crash the pipeline with a `WRITE_ON_NONEXISTENT_DYNPRO`
runtime error.

## Background

`ZCL_FI_ALLOC_STEP_CORR_BCHE->execute` contains a `WRITE:` statement:

```abap
WRITE: / 'Update process completed.'.
```

`WRITE:` requires an active screen (dynpro). Background jobs have no screen.
When this step runs in background (the only production mode), the ABAP runtime
raises a `WRITE_ON_NONEXISTENT_DYNPRO` short dump, crashing the entire pipeline.

The fix is to replace the `WRITE:` with a proper result message in `rs_result-message`.
Story 5.4 (E5) will later add full BAL logging — this story only removes the crash.

## Acceptance Criteria

**AC-01 — No runtime dump in background:**
Given `ZCL_FI_ALLOC_STEP_CORR_BCHE` is executing in a background context
When `execute` completes normally
Then no `WRITE_ON_NONEXISTENT_DYNPRO` runtime error is raised

**AC-02 — Result message populated:**
Given the `WRITE:` statement is replaced
When execution succeeds
Then `rs_result-message` contains a meaningful completion message, e.g.:
`CORR_BCHE completed. 42 headers processed.`
(using the actual header count from the processing loop)

## Tasks

- [ ] **Task 1.2.1**: In `ZCL_FI_ALLOC_STEP_CORR_BCHE->execute`:
  - [ ] Locate `WRITE: / 'Update process completed.'.`
  - [ ] Confirm the header count variable available at that point in the loop
    (e.g., `lines( lt_headers )` or a running counter `lv_count`)
  - [ ] Replace with:
    ```abap
    rs_result-message = |CORR_BCHE completed. { lv_count } headers processed.|.
    ```
    (adjust variable name to match what is actually in scope)

## Dev Notes

- BAL logging (CC-13) is addressed in Story 5.4. This story touches only the
  `WRITE:` line — no other changes needed.
- If a running counter is not yet present, introduce `DATA lv_count TYPE i.` and
  `ADD 1 TO lv_count.` inside `LOOP AT lt_headers`. This is the minimal safe change.
- Do not add `COMMIT WORK` — the framework manages the LUW.
- Line length ≤ 120 chars (constitution Principle II).

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap` | Replace `WRITE:` with `rs_result-message` assignment |

## Constitution Compliance Checklist

- [ ] Principle I — DDIC-First: no new local TYPE definitions
- [ ] Principle II — SAP Standards: line ≤ 120 chars
- [ ] Principle III — Consult SAP Docs: N/A (simple statement replacement)
- [ ] Principle IV — Factory Pattern: N/A
- [ ] Principle V — Error Handling: N/A (success path only)

## Dev Agent Record

### Agent Model Used
github-copilot/claude-sonnet-4.6

### Completion Notes
Bundled into Story 1.1 CORR_BCHE commit. WRITE: statement replaced with
`rs_result-message = |CORR_BCHE completed. { lv_row_count } headers processed.|`.

### Changed Files
| File | Commit |
|------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap` | `3c75fc3e` |
