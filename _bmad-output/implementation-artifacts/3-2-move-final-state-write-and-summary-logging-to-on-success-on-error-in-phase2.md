---
story: "3.2"
title: "Move final state write and summary logging to on_success/on_error in PHASE2"
epic: "3 — Functional Correctness — PHASE2 Substep Path"
sprint: 2
status: done
completed: 2026-03-09
priority: high
estimate: 2h
source_findings:
  - zfi_alloc_phase_2-migration-review.md
  - FINDING-03
  - cross-cutting-analysis.md
target_repo: smolikzd/cz.imcg.fast.planner
dependencies:
  - None (independent; Story 3.1 recommended first to ensure mo_log is available)
---

# Story 3.2: Move final state write and summary logging to `on_success` / `on_error` in PHASE2

## User Story

As a PHASE2 process monitor user,
I want the final `phase_2_status` and summary counters to reflect the actual substep results,
so that the process monitor shows accurate completion data rather than always showing zero
counters and premature 'F' status.

## Background

`ZCL_FI_ALLOC_STEP_PHASE2->execute` currently:
1. Sets `phase_2_status = 'R'` (running) in `ZFI_ALLOC_STATE`
2. Dispatches all substeps via bgRFC
3. **Immediately** sets `phase_2_status = 'F'` and writes a summary log entry

Step 3 is wrong: it runs before any substep has completed. The summary log always shows
zero items processed, and the status is permanently 'F' even if substeps fail.

The correct lifecycle for a QUEUE-mode step:
- `execute`: set status to 'R', dispatch substeps — **nothing more**
- `on_success` (called by framework when all substeps complete): set status to 'F', write summary
- `on_error` (called by framework when any substep fails): set status to 'E', write failure summary

**Note:** The current `on_success` and `on_error` method bodies in PHASE2 are empty.
This story implements them with real logic.

**Note:** `execute` still has one `COMMIT WORK` statement after `UPDATE zfi_alloc_state
FROM <fs_state>`. Remove it as part of this story.

## Acceptance Criteria

**AC-01 — `execute` sets status to 'R' only:**
Given `execute` is called
When it completes
Then `ZFI_ALLOC_STATE.phase_2_status = 'R'`
And no final status ('F' or 'E') is written in `execute`
And no `COMMIT WORK` is issued in `execute`

**AC-02 — `on_success` sets status 'F' and writes summary:**
Given all PHASE2 substeps complete successfully
When the framework calls `on_success`
Then `ZFI_ALLOC_STATE.phase_2_status = 'F'`
And `mo_log` contains a summary message with the total items processed

**AC-03 — `on_error` sets status 'E' and writes failure log:**
Given one or more PHASE2 substeps fail
When the framework calls `on_error`
Then `ZFI_ALLOC_STATE.phase_2_status = 'E'` (or the appropriate error value from the domain)
And `mo_log` records the failure context

**AC-04 — No COMMIT WORK in on_success/on_error:**
Given either callback completes
When it returns
Then no `COMMIT WORK` has been issued inside the method

## Tasks

- [ ] **Task 3.2.1**: Read `ZCL_FI_ALLOC_STEP_PHASE2->execute` in full. Identify:
  - The exact `COMMIT WORK` line after `UPDATE zfi_alloc_state` — remove it
  - The `phase_2_status = 'F'` UPDATE statement — move it to `on_success`
  - The summary log write — move it to `on_success`

- [ ] **Task 3.2.2**: Verify the valid domain values for `ZFI_ALLOC_STATE.phase_2_status`
  (check DDIC domain definition). Confirm 'F' = finished, 'E' = error (or determine
  the correct error value — may be 'X', 'A', etc.).

- [ ] **Task 3.2.3**: Remove from `execute`:
  - The `phase_2_status = 'F'` UPDATE (and any associated `COMMIT WORK`)
  - The summary log write after substep dispatch

- [ ] **Task 3.2.4**: Implement `on_success` in `ZCL_FI_ALLOC_STEP_PHASE2`:
  ```abap
  METHOD zif_fi_process_step~on_success.
    " Update final status to finished
    UPDATE zfi_alloc_state
      SET phase_2_status = 'F'
      WHERE allocation_id = @mv_allocation_id
        AND company_code  = @mv_company_code
        AND fiscal_year   = @mv_fiscal_year
        AND fiscal_period = @mv_fiscal_period.
    " Write summary to BAL log
    IF mo_log IS BOUND.
      mo_log->log_free_text(
        iv_severity = 'S'
        iv_text     = |PHASE2 completed successfully.| ).
      mo_log->save( ).
    ENDIF.
  ENDMETHOD.
  ```
  Adjust to actual `mo_log` API (consult mcp-sap-docs / `ZCL_SP_BALLOG` interface).

- [ ] **Task 3.2.5**: Implement `on_error` in `ZCL_FI_ALLOC_STEP_PHASE2`:
  ```abap
  METHOD zif_fi_process_step~on_error.
    " Update final status to error
    UPDATE zfi_alloc_state
      SET phase_2_status = 'E'
      WHERE allocation_id = @mv_allocation_id
        AND company_code  = @mv_company_code
        AND fiscal_year   = @mv_fiscal_year
        AND fiscal_period = @mv_fiscal_period.
    " Write failure summary to BAL log
    IF mo_log IS BOUND.
      mo_log->log_free_text(
        iv_severity = 'E'
        iv_text     = |PHASE2 failed. Check substep logs for details.| ).
      mo_log->save( ).
    ENDIF.
  ENDMETHOD.
  ```

- [ ] **Task 3.2.6**: Check `on_success` / `on_error` method signatures in
  `ZIF_FI_PROCESS_STEP` to confirm the exact importing parameters available
  (e.g., `io_process_instance`, `io_step_instance`). Use them if aggregation
  of substep results is needed.

- [ ] **Task 3.2.7**: Verify no `COMMIT WORK` in `on_success` or `on_error`.

## Dev Notes

- **Domain values**: before writing `'E'` to `phase_2_status`, verify the domain
  allows it. If `ZFI_ALLOC_STATE.phase_2_status` uses a fixed-value domain, check
  whether 'E' is an allowed value. If not, use whatever value represents error/partial.
- **`on_success`/`on_error` signatures**: from `zif_fi_process_step.intf.abap`:
  - `IMPORTING io_process_instance TYPE REF TO zcl_fi_process_instance`
  - `IMPORTING io_step_instance TYPE REF TO zfi_proc_step` (the DDIC step row)
  These can be used to query substep results from `ZFI_PROC_STEP` substep rows.
- **`mo_log` availability**: after Story 3.1 is applied, `mo_log` is initialised in
  `init`. In `on_success`/`on_error`, check `mo_log IS BOUND` before using it as
  a safety guard.
- **No COMMIT WORK**: the framework calls `on_success`/`on_error` within its own
  LUW management. Do not issue `COMMIT WORK`.
- Reference: Phase 2 FINDING-03, `cross-cutting-analysis.md` §2.

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` | Remove premature final state write from `execute`; implement `on_success` and `on_error`; remove `COMMIT WORK` |

## Constitution Compliance Checklist

- [ ] Principle I — DDIC-First: no new local TYPE definitions
- [ ] Principle II — SAP Standards: line ≤ 120 chars
- [ ] Principle III — Consult SAP Docs: `ZCL_SP_BALLOG` log/save API verified; domain values checked
- [ ] Principle IV — Factory Pattern: N/A
- [ ] Principle V — Error Handling: `on_error` called by framework on failure; `mo_log IS BOUND` guard added

## Dev Agent Record

### Agent Model Used
claude-sonnet-4.5 (github-copilot/claude-sonnet-4.5)

### Completion Notes
- Removed premature final state write (phase_2_status='F') and summary logging from execute method (lines 148-174)
- Implemented on_success method: sets phase_2_status='F', updates end timestamp, writes success message (s009) to BAL log
- Implemented on_error method: sets phase_2_status='E', updates end timestamp, writes error message (e008) to BAL log
- Verified domain ZFI_PHASE_STATUS allows values: P, R, F, E, and space
- Added instance variable validation guards in both on_success and on_error callbacks
- Both callbacks raise ZCX_FI_PROCESS_ERROR if state update fails
- No COMMIT WORK in either callback (framework manages LUW)
- All AC met: AC-01 (execute sets R only), AC-02 (on_success sets F and logs), AC-03 (on_error sets E and logs), AC-04 (no COMMIT WORK)

### Changed Files
| File (remote repo) | Commit |
|--------------------|--------|
| src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap | 8bf9cc1 |
