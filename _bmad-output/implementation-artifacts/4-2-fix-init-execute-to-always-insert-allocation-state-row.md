---
story: "4.2"
title: "Fix INIT execute to always INSERT allocation state row"
epic: "4 — Functional Correctness — Data Integrity"
sprint: 1
status: done
priority: moderate
estimate: 30min
source_findings:
  - CC-15
  - zfi_alloc_init-migration-review.md
target_repo: smolikzd/cz.imcg.fast.planner
dependencies:
  - None (independent)
---

# Story 4.2: Fix INIT `execute` to always INSERT allocation state row

## User Story

As a downstream step (PHASE1, PHASE2, PHASE3),
I want the INIT step to always create the `ZFI_ALLOC_STATE` row,
so that subsequent steps can UPDATE the row without finding zero matches.

## Background

`ZCL_FI_ALLOC_STEP_INIT->execute` currently handles two branches:
- `FORCE_START = true`: DELETE old row + INSERT new row ✅
- `FORCE_START = false`: does nothing ❌ **bug**

When `FORCE_START = false` and no state row exists yet (first run), no INSERT
is performed. PHASE1, PHASE2, and PHASE3 then attempt `UPDATE zfi_alloc_state`
and affect zero rows — their status changes are silently lost.

The corrected logic must also handle the idempotent case:
`FORCE_START = false` + row already exists → skip insert, return success.

## Acceptance Criteria

**AC-01 — First run, FORCE_START false:**
Given `FORCE_START = abap_false` and no existing `ZFI_ALLOC_STATE` row
When INIT's `execute` completes
Then a `ZFI_ALLOC_STATE` row exists with all phase statuses set to initial values

**AC-02 — Force restart:**
Given `FORCE_START = abap_true` and an existing `ZFI_ALLOC_STATE` row
When INIT's `execute` completes
Then the old row is deleted and a new row is inserted

**AC-03 — Idempotent run:**
Given `FORCE_START = abap_false` and an existing `ZFI_ALLOC_STATE` row
When INIT's `execute` runs
Then the existing row is NOT deleted and NOT re-inserted; `execute` returns success

**AC-04 — No premature commit:**
Given any of the above paths
When `execute` completes
Then no `COMMIT WORK` is issued inside `execute` — the framework manages the LUW

## Tasks

- [ ] **Task 4.2.1**: Read `ZCL_FI_ALLOC_STEP_INIT->execute` in full to confirm current logic.

- [ ] **Task 4.2.2**: Refactor the `execute` method:

  ```abap
  " Check for existing state row
  SELECT SINGLE allocation_id FROM zfi_alloc_state
    INTO @DATA(lv_exists)
    WHERE allocation_id = @mv_allocation_id
      AND company_code  = @mv_company_code
      AND fiscal_year   = @mv_fiscal_year
      AND fiscal_period = @mv_fiscal_period.

  IF mv_force_start = abap_true.
    " Force restart: wipe and re-create
    DELETE FROM zfi_alloc_state
      WHERE allocation_id = mv_allocation_id
        AND company_code  = mv_company_code
        AND fiscal_year   = mv_fiscal_year
        AND fiscal_period = mv_fiscal_period.
    INSERT zfi_alloc_state FROM @ls_state.
  ELSEIF sy-subrc <> 0.
    " First run (no row yet): create it
    INSERT zfi_alloc_state FROM @ls_state.
  ELSE.
    " Row already exists and FORCE_START = false: idempotent, skip
    rs_result-message = 'INIT: state row already exists, skipping.'.
  ENDIF.
  ```
  Adjust to exact field names and structure in the actual class.

- [ ] **Task 4.2.3**: Confirm `ls_state` is built with all required fields before the
  INSERT (allocation_id, company_code, fiscal_year, fiscal_period, all phase status
  fields set to initial value, mandt = sy-mandt, created_by = sy-uname,
  created_at = sy-datum).

- [ ] **Task 4.2.4**: Verify `FORCE_START` is read from process parameters in `init`
  (should already be `mv_force_start`; confirm attribute name).

## Dev Notes

- The corrected implementation template is in
  `docs/migration-reviews/zfi_alloc_init-migration-review.md` FINDING-02.
- `INSERT` must not be followed by `COMMIT WORK` — the framework commits
  after `execute` returns.
- Constitution Principle I: `ls_state` must be typed against the DDIC structure
  of `ZFI_ALLOC_STATE`, not a local type.
- Line length ≤ 120 chars.

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_init.clas.abap` | Refactor `execute` INSERT/skip logic |

## Constitution Compliance Checklist

- [ ] Principle I — DDIC-First: `ls_state` typed against DDIC structure
- [ ] Principle II — SAP Standards: line ≤ 120 chars
- [ ] Principle III — Consult SAP Docs: N/A (standard ABAP SQL)
- [ ] Principle IV — Factory Pattern: N/A
- [ ] Principle V — Error Handling: `INSERT` failure handled (check `sy-subrc`)

## Dev Agent Record

### Agent Model Used
github-copilot/claude-sonnet-4.6

### Completion Notes
Refactored execute method with three-branch logic:
1. FORCE_START=true → DELETE + INSERT (force restart)
2. FORCE_START=false + no existing row → INSERT (first run)
3. FORCE_START=false + row exists → skip, return success (idempotent)
`ls_state` typed as `TYPE zfi_alloc_state` (DDIC-First). No COMMIT WORK added.

### Changed Files
| File (remote repo) | Commit |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_init.clas.abap` | `068fcf0f` |
