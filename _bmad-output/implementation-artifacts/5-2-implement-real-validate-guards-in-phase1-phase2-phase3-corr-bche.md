---
story: "5.2"
title: "Implement real validate guards in PHASE1, PHASE2, PHASE3, CORR_BCHE"
epic: "5 — Error Handling and Observability"
sprint: 2
status: done
priority: moderate
estimate: 2h
actual: 1.5h
completed: 2026-03-09
source_findings:
  - CC-04
  - cross-cutting-analysis.md
target_repo: smolikzd/cz.imcg.fast.planner
dependencies:
  - None (independent)
---

# Story 5.2: Implement real `validate` guards in PHASE1, PHASE2, PHASE3, CORR_BCHE

## User Story

As a framework safeguard,
I want `validate` to check all mandatory parameters before any DML is attempted,
so that a misconfigured process fails fast with a clear message rather than performing
destructive operations on incorrect data.

## Background

All step classes must implement `ZIF_FI_PROCESS_STEP~VALIDATE`. Currently PHASE1, PHASE2,
PHASE3, and CORR_BCHE have empty or trivially-passing `validate` implementations.

The framework calls `validate` before `execute`. If `validate` returns
`rs_result-success = abap_false`, `execute` is not called. This is the earliest possible
failure point — no DB reads, no DML, no external calls.

The model is `ZCL_FI_ALLOC_STEP_INIT->validate` which already guards mandatory parameters
correctly.

## Acceptance Criteria

**AC-01 — Missing ALLOCATION_ID fails fast:**
Given a process is started with an initial (empty) `ALLOCATION_ID`
When `validate` runs on any of the four target steps
Then `rs_result-success = abap_false`
And `rs_result-can_continue = abap_false`
And `rs_result-message` identifies `ALLOCATION_ID` as missing

**AC-02 — All four mandatory parameters checked:**
Given any of `ALLOCATION_ID`, `COMPANY_CODE`, `FISCAL_YEAR`, or `FISCAL_PERIOD` is initial
When `validate` runs
Then the method returns failure with a specific message identifying the missing parameter

**AC-03 — Valid parameters pass validation:**
Given all four mandatory parameters are populated with non-initial values
When `validate` runs
Then `rs_result-success = abap_true`
And `execute` is subsequently called by the framework

**AC-04 — No DB access in validate:**
Given `validate` runs
When it completes
Then no SELECT, INSERT, UPDATE, or DELETE statements were executed
(Validate must be fast; DB guards belong in `execute`)

## Tasks

- [x] **Task 5.2.1**: Read `ZCL_FI_ALLOC_STEP_INIT->validate` in full to understand the
  reference implementation pattern.

- [x] **Task 5.2.2**: Read `ZCL_FI_ALLOC_STEP_PHASE1->validate` to understand current state
  (likely empty or returns `success = abap_true` unconditionally).

- [x] **Task 5.2.3**: Implement `validate` in `ZCL_FI_ALLOC_STEP_PHASE1`:
  ```abap
  METHOD zif_fi_process_step~validate.
    rs_result-success      = abap_true.
    rs_result-can_continue = abap_true.

    IF mv_allocation_id IS INITIAL.
      rs_result-success      = abap_false.
      rs_result-can_continue = abap_false.
      rs_result-message      = 'PHASE1: ALLOCATION_ID is missing'.
      RETURN.
    ENDIF.
    IF mv_company_code IS INITIAL.
      rs_result-success      = abap_false.
      rs_result-can_continue = abap_false.
      rs_result-message      = 'PHASE1: COMPANY_CODE is missing'.
      RETURN.
    ENDIF.
    IF mv_fiscal_year IS INITIAL.
      rs_result-success      = abap_false.
      rs_result-can_continue = abap_false.
      rs_result-message      = 'PHASE1: FISCAL_YEAR is missing'.
      RETURN.
    ENDIF.
    IF mv_fiscal_period IS INITIAL.
      rs_result-success      = abap_false.
      rs_result-can_continue = abap_false.
      rs_result-message      = 'PHASE1: FISCAL_PERIOD is missing'.
      RETURN.
    ENDIF.
  ENDMETHOD.
  ```
  Adjust variable names to match exact attribute names in PHASE1.

- [x] **Task 5.2.4**: Repeat Task 5.2.3 for `ZCL_FI_ALLOC_STEP_PHASE2`
  (replace 'PHASE1' with 'PHASE2' in messages; adjust variable names)

- [x] **Task 5.2.5**: Repeat Task 5.2.3 for `ZCL_FI_ALLOC_STEP_PHASE3`

- [x] **Task 5.2.6**: Repeat Task 5.2.3 for `ZCL_FI_ALLOC_STEP_CORR_BCHE`

- [x] **Task 5.2.7**: Verify `validate` is NOT doing any `SELECT` / DB operations in
  any of the four implementations after the change.

## Dev Notes

- **Model on INIT**: `ZCL_FI_ALLOC_STEP_INIT->validate` already demonstrates the correct
  pattern. Use it as the reference — do not invent a new pattern.
- **Attribute names**: verify the exact private attribute names used in each class for
  the four key fields. They may vary slightly (e.g., `mv_alloc_id` vs `mv_allocation_id`).
- **No DB reads in validate**: keep `validate` purely in-memory. Allocation state status
  checks (e.g., is phase_X_status already 'F'?) belong in `execute`, not `validate`.
- **`rs_result` structure**: check the structure definition in `ZIF_FI_PROCESS_STEP`
  to confirm field names (`success`, `can_continue`, `message`).
- Reference: CC-04, `cross-cutting-analysis.md` §2.

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap` | Implement real `validate` with 4-field guard |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` | Same |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap` | Same |
| `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap` | Same |

## Constitution Compliance Checklist

- [x] Principle I — DDIC-First: N/A (no new types; attribute types already DDIC)
- [x] Principle II — SAP Standards: line ≤ 120 chars (all lines comply)
- [x] Principle III — Consult SAP Docs: N/A (standard ABAP IS INITIAL checks)
- [x] Principle IV — Factory Pattern: N/A
- [x] Principle V — Error Handling: validation failure returned via `rs_result`, not exception

## Dev Agent Record

### Agent Model Used
claude-sonnet-4.5

### Completion Notes
Successfully implemented real validate guards in all four target step classes. Each validate method now performs fail-fast parameter validation before execute() is called by the framework.

**Implementation Pattern** (consistent across all 4 classes):
1. Initialize rs_result with success=true and can_continue=true
2. Check each mandatory parameter (ALLOCATION_ID, COMPANY_CODE, FISCAL_YEAR, FISCAL_PERIOD) with IS INITIAL
3. On failure: set success=false, can_continue=false, specific error message, RETURN immediately
4. On success: set success message and return

**Changes per class**:
- **PHASE1**: Replaced trivial validate with 4 parameter guards (error messages prefixed 'PHASE1:')
- **PHASE2**: Replaced trivial validate with 4 parameter guards (error messages prefixed 'PHASE2:')
- **PHASE3**: Replaced trivial validate with 4 parameter guards (error messages prefixed 'PHASE3:')
- **CORR_BCHE**: Replaced trivial validate with 4 parameter guards (error messages prefixed 'CORR_BCHE:')

**Key Design Decisions**:
- Used step-specific prefixes in error messages to aid troubleshooting
- Purely in-memory validation (no DB operations per AC-04)
- Followed fail-fast pattern: first failure returns immediately
- Used exact attribute names confirmed in previous stories: mv_allocation_id, mv_company_code, mv_fiscal_year, mv_fiscal_period

**Verification**:
- Grepped all 4 validate methods for SELECT/INSERT/UPDATE/DELETE/MODIFY - zero matches (AC-04 verified)
- All lines ≤ 120 chars (Constitution Principle II verified)
- Error messages are specific and identify the missing parameter (AC-01, AC-02 verified)

**Acceptance Criteria Compliance**:
- AC-01: Missing ALLOCATION_ID fails fast with message 'PHASE1: ALLOCATION_ID is missing' (and similar for other steps)
- AC-02: All 4 mandatory parameters checked with specific error messages identifying the missing parameter
- AC-03: Valid parameters (all non-initial) pass validation with rs_result-success=true
- AC-04: Zero DB operations in any validate method (verified by grep)

### Changed Files
| File (remote repo) | Commit |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap` | 3f8d430 |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` | 3f8d430 |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap` | 3f8d430 |
| `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap` | 3f8d430 |
