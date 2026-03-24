---
title: 'EST-125: Fix Export Step BAL Logging'
slug: 'tech-spec-est-125-export-step-bal-logging'
created: '2026-03-24'
status: 'ready-for-dev'
stepsCompleted: ['understand', 'investigate', 'generate', 'review']
tech_stack: ['ABAP 7.58', 'ZFI_PROCESS framework', 'SAP BALI API']
files_to_modify:
  - 'src/zfi_alloc_process/zcl_fi_alloc_step_extract.clas.abap'
code_patterns:
  - 'log_message() with iv_message_v1..v4 for context'
  - 'log_and_set_result() for combined result+log'
  - 'mo_log->log_exception_from_framework() in CATCH blocks'
test_patterns:
  - 'Manual: run allocation process with EXPORT step, check SLG1 log'
linear_issue: 'EST-125'
target_repository: 'ovysledovka'
constitution_principles:
  - 'Principle II - SAP Standards (ABAP-Doc, naming)'
  - 'Principle III - Consult SAP Docs (BALI API patterns)'
  - 'Principle V - Error Handling & Observability (exception logging, context in messages)'
---

# Tech-Spec: EST-125 -- Fix Export Step BAL Logging

**Created:** 2026-03-24
**Linear Issue:** [EST-125](https://linear.app/smolikzd/issue/EST-125/)
**Target Repository:** `cz.imcg.fast.ovysledovka`

## Overview

### Problem Statement

The export step (`ZCL_FI_ALLOC_STEP_EXTRACT`) does not write meaningful data to the SAP Application Log (BAL). When operators investigate failed or successful export runs in SLG1, they see only two generic messages (600=error, 601=success) with zero context -- no fiscal year/period, no exception details, no extraction summary. This is inconsistent with all other steps in the pipeline (INIT, PHASE1, PHASE2, CORR_BCHE, PHASE3) which provide rich, actionable log entries.

Six specific issues were identified:

| # | Severity | Issue |
|---|----------|-------|
| 1 | HIGH | Missing exception detail logging -- `lo_ex` captured but never passed to `mo_log->log_exception_from_framework()` |
| 2 | HIGH | No execution context in log -- messages 600/601 have zero placeholder values (no year/period) |
| 3 | ~~MEDIUM~~ | ~~Empty `validate()` method~~ -- **Skipped per project lead decision** |
| 4 | MEDIUM | Orphan MESSAGE statement -- `MESSAGE s117(zfi_alloc) INTO rs_result-message` fills `sy-msg*` but `log_sy_msg()` is never called |
| 5 | LOW | No extraction summary -- no log of record count or files written after `mo_kbc->execute()` |
| 6 | LOW | Stale class header comment -- says "Phase 1" instead of "Export/Extract" |

### Solution

Align `ZCL_FI_ALLOC_STEP_EXTRACT` with the logging patterns established by INIT and PHASE1 steps:

1. Add `mo_log->log_exception_from_framework()` in the CATCH block
2. Add fiscal year/period context to all log messages via `iv_message_v1`/`iv_message_v2`
3. Replace orphan MESSAGE with `log_and_set_result()` pattern
4. Add post-execution summary logging (record count if available from extractor)
5. Fix class header comment

### Scope

**In Scope:**
- Fix 5 logging issues in `zcl_fi_alloc_step_extract.clas.abap` (validate guards intentionally skipped)
- May need new T100 messages in message class `ZFI_ALLOC` for context-rich logging (or reuse existing 600/601 with placeholders if message definition allows)

**Out of Scope:**
- Changes to `zcl_fi_ov_keboola_extractor` (the extractor class itself)
- Changes to the ZFI_PROCESS framework base class or logger
- Adding unit tests (no test infrastructure exists for step classes yet)
- Changing the extractor's business logic or file output format

## Context for Development

### Codebase Patterns

**Pattern 1: Exception logging in CATCH blocks** (from PHASE1, lines 66-68):
```abap
CATCH cx_amdp_execution_failed INTO DATA(lx_amdp_p1).
  IF mo_log IS BOUND.
    mo_log->log_exception_from_framework( lx_amdp_p1 ).
  ENDIF.
  log_message( iv_message_class = 'ZFI_ALLOC' iv_message_number = '011'
               iv_message_v1 = CONV symsgv( mv_company_code )
               iv_message_v2 = CONV symsgv( mv_fiscal_year )
               iv_severity = 'E' ).
```

**Pattern 2: Context logging with placeholders** (from INIT, line 39):
```abap
log_message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '000'
  iv_message_v1     = |{ mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period }/{ mv_allocation_id }|
).
```

**Pattern 3: log_and_set_result() for combined result+log** (from INIT, lines 56-60):
```abap
rs_result-message = log_and_set_result(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '016'
  iv_severity       = 'S'
).
```

### Files to Reference

| File | Purpose |
| ---- | ------- |
| `ovysledovka: src/zfi_alloc_process/zcl_fi_alloc_step_extract.clas.abap` | **Target file** -- the export step to fix |
| `ovysledovka: src/zfi_alloc_process/zcl_fi_alloc_step_init.clas.abap` | Reference: validate pattern, log_and_set_result pattern |
| `ovysledovka: src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap` | Reference: exception logging pattern, context logging |
| `planner: src/zcl_fi_process_step.clas.abap` | Base class: `log_message()`, `log_sy_msg()`, `log_and_set_result()` |
| `planner: src/zif_fi_process_logger.intf.abap` | Logger interface: `log_exception_from_framework()` |

### Technical Decisions

1. **Reuse existing message numbers where possible.** Messages 600 (error) and 601 (success) already exist. If they accept placeholder variables (`&1`, `&2`), add fiscal year/period as `iv_message_v1`/`iv_message_v2`. If they are defined without placeholders, new message numbers will be needed.

2. **validate() is intentionally left as-is.** The EXPORT step parameters (fiscal year/period) are already validated by the INIT step earlier in the pipeline. Adding redundant guards here is unnecessary.

3. **Extraction summary depends on extractor API.** If `zcl_fi_ov_keboola_extractor` exposes a method to get record count after `execute()` (e.g., `get_record_count()` or a public attribute), log it. If not, log a simple "extraction completed successfully" with context. Do NOT modify the extractor class.

4. **Keep the local TYPE definitions.** Issues #3 (local `ty_sel`, `ty_r_fins_sem_tag`) are pre-existing and out of scope for this logging fix. A separate story should address DDIC-first compliance.

## Implementation Plan

### Tasks

**Task 1: Fix class header comment** (Issue #6)
- File: `zcl_fi_alloc_step_extract.clas.abap`, lines 1-2
- Change:
  ```abap
  " BEFORE:
  *& Class ZCL_FI_ALLOC_STEP_PHASE1
  *& Phase 1: Fetch source documents and prepare allocation base

  " AFTER:
  *& Class ZCL_FI_ALLOC_STEP_EXTRACT
  *& Export: Extract period data to Keboola via FTP
  ```

**Task 2: Add execution context logging at start of execute()** (Issue #2)
- File: `zcl_fi_alloc_step_extract.clas.abap`, method `execute`, insert before TRY
- Log fiscal year/period context using an appropriate message number:
  ```abap
  log_message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '000'
    iv_message_v1     = |{ mv_fiscal_year }/{ mv_fiscal_period }|
  ).
  ```
  Note: Message 000 is used by INIT for context. If it accepts `&1`, reuse it. Otherwise define a new message like "Export started for period &1/&2".

**Task 3: Fix CATCH block -- add exception detail logging** (Issue #1)
- File: `zcl_fi_alloc_step_extract.clas.abap`, lines 80-84
- Add `mo_log->log_exception_from_framework()` before `log_message()`:
  ```abap
  CATCH zcx_fi_ov_keboola_extractor INTO DATA(lo_ex).
    IF mo_log IS BOUND.
      mo_log->log_exception_from_framework( lo_ex ).
    ENDIF.
    log_message(
      iv_message_class  = 'ZFI_ALLOC'
      iv_message_number = '600'
      iv_message_v1     = CONV symsgv( mv_fiscal_year )
      iv_message_v2     = CONV symsgv( mv_fiscal_period )
      iv_severity       = 'E'
    ).
    rs_result-success      = abap_false.
    rs_result-can_continue = abap_false.
    RETURN.
  ```

**Task 4: Fix success path -- replace orphan MESSAGE with log_and_set_result()** (Issues #4 + #2)
- File: `zcl_fi_alloc_step_extract.clas.abap`, lines 87-91
- Replace the current pattern:
  ```abap
  " BEFORE:
  log_message( iv_message_class = 'ZFI_ALLOC'
               iv_message_number = '601' iv_severity = 'S' ).
  MESSAGE s117(zfi_alloc) INTO rs_result-message.

  " AFTER:
  rs_result-message = log_and_set_result(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '601'
    iv_message_v1     = CONV symsgv( mv_fiscal_year )
    iv_message_v2     = CONV symsgv( mv_fiscal_period )
    iv_severity       = 'S'
  ).
  ```
  This eliminates the orphan MESSAGE and logs the success with context in a single call.

**Task 5 (Optional): Add extraction summary** (Issue #5)
- After `mo_kbc->execute()`, check if the extractor object exposes record count
- If available: `log_message( ... iv_message_v1 = CONV #( lv_record_count ) ... )`
- If not available: Skip this task (do not modify the extractor class)
- Candidate approach: check for public attributes or methods on `zcl_fi_ov_keboola_extractor` that return stats

### Acceptance Criteria

**AC1: Exception details appear in SLG1**
- Given: The Keboola extractor raises `zcx_fi_ov_keboola_extractor` during export
- When: The step fails and the operator opens SLG1
- Then: The log contains the full exception text from `log_exception_from_framework()` AND the error message 600 with fiscal year/period in the placeholders

**AC2: Success log contains execution context**
- Given: The export step completes successfully
- When: The operator opens SLG1
- Then: The log contains at minimum: (a) context entry with fiscal year/period, (b) success message 601 with fiscal year/period

**AC3: No orphan MESSAGE statements**
- Given: Any execution path through the export step
- When: A `MESSAGE ... INTO` statement is executed
- Then: Either `log_sy_msg()` or `log_and_set_result()` is called to persist it to BAL

**AC4: Class header is accurate**
- Given: The class file header comment
- Then: It references "ZCL_FI_ALLOC_STEP_EXTRACT" and "Export" (not "PHASE1")

**AC5: Line length compliance**
- Given: All modified/added lines
- Then: No line exceeds 120 characters (per constitution)

## Additional Context

### Dependencies

- No blocking dependencies. This is a self-contained fix within a single class file.
- The export step's existing behavior (calling `zcl_fi_ov_keboola_extractor`) is unchanged.
- Message numbers 600, 601 must already exist in message class `ZFI_ALLOC`. Verify before implementation; create new messages only if existing ones don't accept placeholders.

### Testing Strategy

1. **Manual test -- happy path**: Run full allocation process through EXPORT step for a valid fiscal year/period. Check SLG1 -- verify context message and success message with year/period visible.
2. **Manual test -- extractor failure**: Simulate extractor error (e.g., invalid FTP path). Verify SLG1 shows full exception text AND error message 600 with year/period.

### Notes

- The `mt_semtag_filter` in `init()` contains duplicate entries (e.g., NTINC_ALAC, PL_RESULT, Z3_EBITDA appear twice). This is a separate issue and out of scope for this spec.
- The local TYPE definitions (`ty_sel`, `ty_r_fins_sem_tag`) violate Constitution Principle I (DDIC-First). This is pre-existing technical debt and out of scope.
- This spec does NOT require changes to the `cz.imcg.fast.planner` framework repository.
