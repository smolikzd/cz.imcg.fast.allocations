---
story: "7.1"
title: "Remove ms_context dead field from all step classes"
epic: "7 — Code Quality and Housekeeping"
sprint: 3
status: done
priority: low
estimate: 30min
source_findings:
  - CC-07
  - cross-cutting-analysis.md
target_repo: smolikzd/cz.imcg.fast.planner
dependencies: []
---

# Story 7.1: Remove ms_context Dead Field from All Step Classes

## User Story

As a developer maintaining step classes,
I want dead private fields removed,
so that the class declaration accurately reflects the data actually used.

## Background

All five step classes (INIT, PHASE1, PHASE2, PHASE3, CORR_BCHE) have a `ms_context`
field in their PRIVATE SECTION:
```abap
DATA ms_context TYPE zif_fi_process_step=>ts_context.
```

This field is written in the `init` method:
```abap
ms_context = is_context.
```

However, `ms_context` is **never read** anywhere in any of the step classes. It is
dead code carried over from the template generation.

Removing this field:
- Reduces memory footprint (small but measurable)
- Improves code clarity (what you see is what's used)
- Follows clean code principles (YAGNI - You Aren't Gonna Need It)

## Acceptance Criteria

**AC-01 — ms_context removed from all classes:**
Given any of the five step classes
When a code review is performed
Then no `ms_context` field exists in the PRIVATE SECTION that is written but never read

**AC-02 — No references remain:**
Given all five step classes are updated
When a grep search is performed for `ms_context`
Then no matches are found in any step class file

## Tasks

- [ ] **Task 7.1.1**: Grep `ZCL_FI_ALLOC_STEP_INIT` to verify `ms_context` is only
  written, never read (search for patterns other than `ms_context =`)

- [ ] **Task 7.1.2**: Remove `DATA ms_context TYPE zif_fi_process_step=>ts_context.`
  from PRIVATE SECTION in `ZCL_FI_ALLOC_STEP_INIT`

- [ ] **Task 7.1.3**: Remove `ms_context = is_context.` from `init` method in
  `ZCL_FI_ALLOC_STEP_INIT`

- [ ] **Task 7.1.4**: Repeat Tasks 7.1.1-7.1.3 for `ZCL_FI_ALLOC_STEP_PHASE1`

- [ ] **Task 7.1.5**: Repeat Tasks 7.1.1-7.1.3 for `ZCL_FI_ALLOC_STEP_PHASE2`

- [ ] **Task 7.1.6**: Repeat Tasks 7.1.1-7.1.3 for `ZCL_FI_ALLOC_STEP_PHASE3`

- [ ] **Task 7.1.7**: Repeat Tasks 7.1.1-7.1.3 for `ZCL_FI_ALLOC_STEP_CORR_BCHE`

- [ ] **Task 7.1.8**: Final verification: grep all five files to confirm no `ms_context`
  references remain

## Dev Notes

- **Safety pattern**: Always grep for reads before removing to ensure it's truly dead
- **Search command**: `grep -n "ms_context" <file> | grep -v "ms_context ="`
  - Should return zero results (only the assignment line exists)
- **Typical location**:
  - PRIVATE SECTION: Near other `DATA` declarations (usually after attribute declarations)
  - init method: Usually first or second line after METHOD signature
- **Risk**: Very low — removing unused field has no functional impact
- **Benefit**: Cleaner code, easier to understand class data model
- Reference: CC-07, `cross-cutting-analysis.md` §7

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_init.clas.abap` | Remove `ms_context` declaration and assignment |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap` | Remove `ms_context` declaration and assignment |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` | Remove `ms_context` declaration and assignment |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap` | Remove `ms_context` declaration and assignment |
| `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap` | Remove `ms_context` declaration and assignment |

## Constitution Compliance Checklist

- [x] Principle I — DDIC-First: Removing dead code, no type changes
- [x] Principle II — SAP Standards: Code removal only, no new code added
- [x] Principle III — Consult SAP Docs: Not applicable (removing code)
- [x] Principle IV — Factory Pattern: Not applicable
- [x] Principle V — Error Handling: Not applicable

## Expected Code Changes Per File

### Before (PRIVATE SECTION):
```abap
PRIVATE SECTION.
  DATA ms_context TYPE zif_fi_process_step=>ts_context.
  DATA mv_company_code TYPE bukrs.
  DATA mv_fiscal_year TYPE gjahr.
  " ... other fields ...
```

### After (PRIVATE SECTION):
```abap
PRIVATE SECTION.
  DATA mv_company_code TYPE bukrs.
  DATA mv_fiscal_year TYPE gjahr.
  " ... other fields ...
```

### Before (init method):
```abap
METHOD zif_fi_process_step~init.
  ms_context = is_context.
  
  mv_company_code = is_context-io_process_instance->get_init_param_value( 'COMPANY_CODE' ).
  " ... other initializations ...
ENDMETHOD.
```

### After (init method):
```abap
METHOD zif_fi_process_step~init.
  mv_company_code = is_context-io_process_instance->get_init_param_value( 'COMPANY_CODE' ).
  " ... other initializations ...
ENDMETHOD.
```

## Dev Agent Record

### Agent Model Used

### Completion Notes

### Changed Files
| File (remote repo) | Commit |
|--------------------|--------|
