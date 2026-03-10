---
story: "5.4"
title: "Add BAL log initialisation and logging to CORR_BCHE"
epic: "5 — Error Handling and Observability"
sprint: 2
status: done
priority: moderate
estimate: 1h
actual: 1h
completed: 2026-03-09
source_findings:
  - CC-13
  - cross-cutting-analysis.md
target_repo: smolikzd/cz.imcg.fast.planner
dependencies:
  - Story 1.2 (WRITE: removal) — done in Sprint 1; must be completed first
---

# Story 5.4: Add BAL log initialisation and logging to CORR_BCHE

## User Story

As an operations analyst,
I want CORR_BCHE to write a BAL log entry on completion,
so that the total number of headers processed and any errors are visible in the
application log (SLG1).

## Background

CORR_BCHE is the only step class that has no BAL (`ZCL_SP_BALLOG`) log object.
All other step classes (PHASE1, PHASE2, PHASE3) initialise `mo_log` and write
processing details. Without a log, operations have no visibility into how many
`ZFI_ALLOC_BCHE` headers were processed or whether any MODIFY errors occurred.

The fix:
1. Add `mo_log TYPE REF TO zcl_sp_ballog` to PRIVATE SECTION
2. Initialise `mo_log` in `init` (DDIC-First, consistent with other steps)
3. Write a summary log entry in `execute` on completion
4. Write error log entries for any MODIFY errors during header processing

**Pre-condition**: Story 1.2 (WRITE: removal) must be done first — it is done (Sprint 1).

**Note on `zcl_fi_allocations=>c_bal_subobj_corr_bche`**: this constant may not exist.
Before using it, verify whether it exists. If not, either use a literal string for the
sub-object or add the constant to `ZCL_FI_ALLOCATIONS` — check the class source first.

## Acceptance Criteria

**AC-01 — mo_log initialised in init:**
Given `ZCL_FI_ALLOC_STEP_CORR_BCHE` is created and `init` is called
When `init` completes
Then `mo_log` is a valid, non-null `ZCL_SP_BALLOG` instance

**AC-02 — Summary log entry on completion:**
Given CORR_BCHE executes successfully
When `execute` completes
Then a BAL log entry (severity 'S') exists with the total number of
`ZFI_ALLOC_BCHE` headers processed and the count of `no_items = 'Y'` updates

**AC-03 — Error log entries for MODIFY failures:**
Given a `MODIFY zfi_alloc_bche` fails for an individual header
When the error is detected (sy-subrc <> 0)
Then the error is written to `mo_log` (severity 'E') — not silently ignored

**AC-04 — Log saved:**
Given `execute` completes (success or partial failure)
When the method returns
Then `mo_log->save( )` has been called so the entries are persisted

## Tasks

- [x] **Task 5.4.1**: Read `ZCL_FI_ALLOC_STEP_CORR_BCHE` PRIVATE SECTION and confirm
  `mo_log` is not already declared. Read the `init` method to understand where to add
  the `mo_log` initialisation.

- [x] **Task 5.4.2**: Check whether `zcl_fi_allocations=>c_bal_subobj_corr_bche`
  constant exists (search remote repo source for `c_bal_subobj_corr_bche`).
  - If it exists: use it directly
  - If it does not exist: use a string literal `'CORR_BCHE'` for the sub-object for now,
    and create the constant in `ZCL_FI_ALLOCATIONS` as a follow-up

- [x] **Task 5.4.3**: Consult mcp-sap-docs for `ZCL_SP_BALLOG` constructor signature
  (Constitution Principle III). Confirm the required importing parameters
  (object, sub-object, etc.) by comparing with how `mo_log` is initialised in PHASE1
  or PHASE2.

- [x] **Task 5.4.4**: Add to PRIVATE SECTION of `ZCL_FI_ALLOC_STEP_CORR_BCHE`:
  ```abap
  mo_log TYPE REF TO zcl_sp_ballog,
  ```

- [x] **Task 5.4.5**: In `init`, add `mo_log` initialisation after existing setup:
  ```abap
  mo_log = NEW zcl_sp_ballog(
    iv_object    = zcl_fi_allocations=>c_bal_object      " or appropriate constant
    iv_subobject = 'CORR_BCHE'                           " or constant if it exists
    iv_desc      = |Allocation { mv_allocation_id } CORR_BCHE| ).
  ```
  Adjust constructor parameters to match the actual `ZCL_SP_BALLOG` signature.

- [x] **Task 5.4.6**: In `execute`, after the header processing loop, add:
  ```abap
  mo_log->log_free_text(
    iv_severity = 'S'
    iv_text     = |CORR_BCHE: { <total_headers_processed> } headers processed, | &&
                  |{ <no_items_updated_count> } no_items updated.| ).
  mo_log->save( ).
  ```
  Use the actual counter variable names from the `execute` method.

- [x] **Task 5.4.7**: Within the header processing loop, after any MODIFY failure
  (where sy-subrc is checked), add:
  ```abap
  mo_log->log_free_text(
    iv_severity = 'E'
    iv_text     = |CORR_BCHE error on header { <key_value> }: MODIFY failed.| ).
  ```

- [x] **Task 5.4.8**: Verify `mo_log` is NOT re-initialised inside `execute` (it is
  initialised in `init` only).

## Dev Notes

- **Constitution Principle I (DDIC-First)**: `mo_log TYPE REF TO zcl_sp_ballog` —
  `zcl_sp_ballog` is a class, not a local type. This is compliant.
- **Constitution Principle III**: consult mcp-sap-docs for `ZCL_SP_BALLOG`
  constructor and `log_free_text`/`log_sy_msg`/`save` method signatures before
  implementing. Compare with PHASE1/PHASE2 usage as a practical reference.
- **BAL sub-object constant**: do not create a new constant in `ZCL_FI_ALLOCATIONS`
  without first verifying it does not exist. If the constant must be created, do it
  in the same commit for traceability.
- **`c_bal_object`**: verify the correct BAL object constant for this application area
  in `ZCL_FI_ALLOCATIONS`.
- Reference: CC-13, `cross-cutting-analysis.md` §4.

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap` | Add `mo_log` to PRIVATE SECTION; initialise in `init`; write summary and error entries in `execute` |
| `src/zfi_alloc_process/zcl_fi_allocations.clas.abap` | Add `c_bal_subobj_corr_bche` constant if it does not exist |

## Constitution Compliance Checklist

- [x] Principle I — DDIC-First: `mo_log TYPE REF TO zcl_sp_ballog` (class reference, not local type)
- [x] Principle II — SAP Standards: line ≤ 120 chars
- [x] Principle III — Consult SAP Docs: `ZCL_SP_BALLOG` API verified via PHASE2 reference implementation
- [x] Principle IV — Factory Pattern: `CREATE OBJECT mo_log` — consistent with existing step pattern
- [x] Principle V — Error Handling: UPDATE errors written to log, not silently ignored

## Dev Agent Record

### Agent Model Used
claude-sonnet-4.5

### Completion Notes
- **Constant creation**: Added `c_bal_subobj_corr_bche = 'CORR_BCHE'` to `ZCL_FI_ALLOCATIONS` (line 11) as the constant did not exist
- **Pattern consistency**: Used PHASE2 as reference for `mo_log` initialization pattern
- **Constructor signature**: Used `CREATE OBJECT` + `log_create()` pattern (not constructor with parameters) consistent with existing steps
- **Counters added**: 
  - `lv_headers_count` - tracks total headers processed
  - `lv_no_items_count` - tracks headers with no items (no_items='Y')
- **Error logging**: Added error log entries after both UPDATE statements with sy-subrc check
- **Summary logging**: Added success log entry with counts before mo_log->save()
- **Verification**: Confirmed mo_log only initialized once in init method, not re-initialized in execute

### Changed Files
| File (remote repo) | Commit |
|--------------------|--------|
| `src/zcl_fi_allocations.clas.abap` | a430daf09f4b9ba2539b20c58ef49609bee44d9b |
| `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap` | a430daf09f4b9ba2539b20c58ef49609bee44d9b |
