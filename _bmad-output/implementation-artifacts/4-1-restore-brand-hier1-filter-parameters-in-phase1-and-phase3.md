---
story: "4.1"
title: "Restore BRAND / HIER1 filter parameters in PHASE1 and PHASE3"
epic: "4 — Functional Correctness — Data Integrity"
sprint: 2
status: cancelled
cancelled: 2026-03-09
reason: "Not required - user request"
priority: critical
estimate: 2h
source_findings:
  - CC-09
  - cross-cutting-analysis.md
target_repo: smolikzd/cz.imcg.fast.planner
dependencies:
  - DDIC check must be done first (Task 4.1.1); if BRAND/HIER1 absent, DDIC extension required
    before code changes
---

# Story 4.1: Restore BRAND / HIER1 filter parameters in PHASE1 and PHASE3

## User Story

As a business user running the allocation process,
I want PHASE1 and PHASE3 to apply BRAND and HIER1 filters correctly,
so that the allocation is scoped to the intended subset of data rather than processing
all line items regardless of brand or hierarchy.

## Background

The original `ZFI_ALLOC_PROCESS` step classes used `BRAND` and `HIER1` parameters to
filter the base keys passed to `ZCL_FI_ALLOCATIONS=>GET_BASE_KEYS` (PHASE1) and
`ZCL_FI_ALLOCATIONS=>GET_UNASSIGNED` (PHASE3).

During migration, the parameter read logic was lost and hardcoded `''` values are now
passed instead. This causes the process to run on all data regardless of brand/hierarchy
— a data integrity defect (CC-09).

**DDIC prerequisite**: If `BRAND` and `HIER1` are not present in `ZFI_ALLOC_PROCESS_PARAMS`
or `ZFI_ALLOC_ADD_PROCESS_PARAMS`, those DDIC structures must be extended with the
appropriate fields before the step code can read them.

## Acceptance Criteria

**AC-01 — PHASE1 passes BRAND/HIER1 to GET_BASE_KEYS:**
Given the process is started with `BRAND = 'ADIDAS'` and `HIER1 = 'H01'`
When PHASE1 calls `ZCL_FI_ALLOCATIONS=>GET_BASE_KEYS`
Then `iv_brand = 'ADIDAS'` and `iv_hier1 = 'H01'` are passed
And `lt_base_keys` contains only items matching those filters

**AC-02 — PHASE3 passes BRAND/HIER1 to GET_UNASSIGNED:**
Given the same parameters
When PHASE3 calls `ZCL_FI_ALLOCATIONS=>GET_UNASSIGNED`
Then `iv_brand = 'ADIDAS'` and `iv_hier1 = 'H01'` are passed

**AC-03 — Empty BRAND/HIER1 treated as "no filter":**
Given the process is started without BRAND or HIER1 set
When PHASE1 and PHASE3 run
Then `iv_brand = ''` and `iv_hier1 = ''` are passed (preserving original default behaviour)

**AC-04 — DDIC structures contain BRAND and HIER1:**
Given this story is implemented
When `GET_PARAM_VALUE( 'BRAND' )` is called
Then it resolves without a field-not-found error (i.e., the DDIC structure has the field)

## Tasks

- [ ] **Task 4.1.1 (DDIC investigation — must be done first)**:
  - [ ] Check the DDIC definition of `ZFI_ALLOC_PROCESS_PARAMS` in the remote repository
    (look for `.dtel.xml` or `.tabl.xml` files, or search the ABAP source for the structure)
  - [ ] Check `ZFI_ALLOC_ADD_PROCESS_PARAMS` similarly
  - [ ] Determine whether `BRAND` and `HIER1` fields exist in either structure
  - [ ] If absent: confirm the correct data elements to use:
    - `BRAND` → likely `WRF_BRAND_ID` or `ZMATBRAND` (verify in remote repo)
    - `HIER1` → likely `ZMATKLH1` or similar (verify in remote repo)
  - [ ] **If DDIC extension is required**: add the fields to the appropriate structure
    in the remote repository DDIC XML file(s) before implementing step code changes

- [ ] **Task 4.1.2**: Add `mv_brand` and `mv_hier1` private attributes to
  `ZCL_FI_ALLOC_STEP_PHASE1` PRIVATE SECTION:
  ```abap
  mv_brand  TYPE <ddic_brand_type>,
  mv_hier1  TYPE <ddic_hier1_type>,
  ```
  Use the exact DDIC data element types determined in Task 4.1.1.

- [ ] **Task 4.1.3**: In `ZCL_FI_ALLOC_STEP_PHASE1->init`: read BRAND and HIER1 into
  the new attributes. Use the same parameter-reading mechanism as the other `mv_*` variables:
  ```abap
  mv_brand = get_init_param_value( 'BRAND' ).
  mv_hier1 = get_init_param_value( 'HIER1' ).
  ```
  Verify the exact method name — may be `get_param_value`, `get_process_param`, etc.

- [ ] **Task 4.1.4**: In `ZCL_FI_ALLOC_STEP_PHASE1->execute`: replace hardcoded `''`
  with `mv_brand` and `mv_hier1` in the `ZCL_FI_ALLOCATIONS=>GET_BASE_KEYS` call:
  ```abap
  ZCL_FI_ALLOCATIONS=>GET_BASE_KEYS(
    EXPORTING
      iv_allocation_id = mv_allocation_id
      iv_company_code  = mv_company_code
      iv_fiscal_year   = mv_fiscal_year
      iv_fiscal_period = mv_fiscal_period
      iv_brand         = mv_brand    " was: ''
      iv_hier1         = mv_hier1    " was: ''
    IMPORTING
      et_base_keys     = lt_base_keys ).
  ```

- [ ] **Task 4.1.5**: Update log messages in PHASE1 execute that contain
  `Značka: { '' }` and `Hier1: { '' }` to use the actual variables:
  ```abap
  |Značka: { mv_brand }|
  |Hier1: { mv_hier1 }|
  ```

- [ ] **Task 4.1.6**: Repeat Tasks 4.1.2–4.1.5 for `ZCL_FI_ALLOC_STEP_PHASE3`:
  - [ ] Add `mv_brand` / `mv_hier1` to PRIVATE SECTION
  - [ ] Read from process params in `init`
  - [ ] Pass to `GET_UNASSIGNED` in `execute`
  - [ ] Update log messages

## Dev Notes

- **DDIC-First (Constitution Principle I)**: do not use local TYPE definitions for the
  new attributes. Use DDIC data elements. Verify the correct element names for BRAND
  and HIER1 in this project before writing code.
- **Parameter reading method**: search the existing `init` code in PHASE1/PHASE3 for
  the method used to read other `mv_*` parameters (e.g., `mv_allocation_id`). Use the
  same approach for BRAND and HIER1.
- **DDIC extension**: if the DDIC structure needs extending, follow the same XML format
  as other fields in the structure. Do not introduce local type definitions.
- **Optional fields**: if BRAND/HIER1 are not provided by the caller, treat initial
  value as "no filter" — pass `' '` or `''` to the method.
- Reference: CC-09, `cross-cutting-analysis.md` §3.

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| DDIC file for `ZFI_ALLOC_PROCESS_PARAMS` or `ZFI_ALLOC_ADD_PROCESS_PARAMS` | Add BRAND/HIER1 fields (only if absent per Task 4.1.1) |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap` | Add `mv_brand`/`mv_hier1`; read in `init`; pass in `execute`; fix log messages |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap` | Same as PHASE1 |

## Constitution Compliance Checklist

- [ ] Principle I — DDIC-First: `mv_brand`/`mv_hier1` typed with DDIC data elements; no local types
- [ ] Principle II — SAP Standards: line ≤ 120 chars; DDIC field names ≤ 30 chars
- [ ] Principle III — Consult SAP Docs: data element names verified via mcp-sap-docs / remote DDIC
- [ ] Principle IV — Factory Pattern: N/A
- [ ] Principle V — Error Handling: N/A for this story (parameters are optional; empty = no filter)

## Dev Agent Record

### Agent Model Used
claude-sonnet-4.5 (github-copilot/claude-sonnet-4.5)

### Completion Notes
**CANCELLED** - Story removed from sprint plan at user request on 2026-03-09.
Reason: Not required for the project.

Initial implementation (commit e12a20a) was reverted in commit fc2548c.

### Changed Files
| File (remote repo) | Commit |
|--------------------|--------|
| N/A - Story cancelled | fc2548c (revert) |
