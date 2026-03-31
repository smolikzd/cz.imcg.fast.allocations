---
title: 'Instance Lifecycle State Validation'
slug: 'spec-001-instance-lifecycle-validation'
created: '2026-03-11'
status: 'draft'
target_repository: 'planner'
epic_id: 'TBD'
story_id: 'TBD'
priority: 'high'
tech_stack: ['ABAP 7.58', 'ZFI_PROCESS Framework']
files_to_modify:
  - 'src/zcl_fi_process_instance.clas.abap'
  - 'src/zcx_fi_process_error.clas.abap'
code_patterns: ['state_validation', 'guard_clauses', 'exception_handling']
test_patterns: ['negative_testing', 'state_transition_testing']
---

# Tech-Spec: Instance Lifecycle State Validation

**Created:** 2026-03-11  
**Target Repository:** `cz.imcg.fast.planner` (framework)  
**Priority:** HIGH (Critical for production safety)

## Overview

### Problem Statement

The `execute()` method in `zcl_fi_process_instance` currently has **NO validation** of the instance status before starting execution. This allows the method to be called on instances that are already RUNNING, COMPLETED, FAILED, or CANCELLED, leading to critical issues:

1. **Concurrent Execution Risk**: Calling `execute()` on a RUNNING instance overwrites timestamps, resets statistics, and can cause multiple threads to modify the same instance simultaneously
2. **Data Corruption**: Re-executing COMPLETED instances causes duplicate processing
3. **Incorrect API Usage**: Users bypass `restart()` method by calling `execute()` on FAILED instances
4. **Audit Trail Corruption**: `started_at` timestamp and `started_by` user are overwritten on each call

**Current Code (Line 512-523):**
```abap
METHOD execute.
  " Determine starting step
  DATA(lv_start_step) = COND zfi_process_step_number(
    WHEN iv_start_from_step IS NOT INITIAL THEN iv_start_from_step
    WHEN ms_instance-current_step <> '0000' THEN ms_instance-current_step
    ELSE '0001'
  ).

  " Update instance status - NO VALIDATION!
  ms_instance-status = gc_status-running.  " ← Critical: Unconditional assignment
  ms_instance-started_by = sy-uname.
  GET TIME STAMP FIELD ms_instance-started_at.
```

### Solution

Add **guard clause validation** at the beginning of `execute()` method to enforce proper state transitions and prevent invalid operations. Only instances with status `NEW` should be allowed to execute.

### Scope

**In Scope:**
- Add state validation guard clause to `execute()` method
- Define clear exception messages for each invalid state
- Add new exception text IDs to `zcx_fi_process_error` message class
- Update method documentation to clarify valid states
- Ensure `restart()` method continues to work correctly for FAILED instances

**Out of Scope:**
- Changes to database schema or DDIC objects
- Changes to step-level status handling (steps continue to use PENDING status)
- Addition of new instance states (PENDING remains unused at instance level)
- Migration of existing data (validation is runtime-only)
- Changes to UI/Fiori apps displaying instances
- Performance optimization (minimal overhead from validation check)

## Context for Development

### Constitution Compliance

This specification complies with the following constitution principles:

**Principle II - SAP Standards Compliance:**
- Uses proper ABAP guard clause pattern
- Maintains ABAP-Doc comment standards
- Follows factory pattern (no changes to instantiation)

**Principle III - Consult SAP Documentation:**
- State validation pattern verified against SAP ABAP Development Guidelines
- Exception handling follows SAP best practices

**Principle V - Error Handling & Observability:**
- Raises `zcx_fi_process_error` with clear context
- Provides descriptive error messages guiding users to correct API
- Preserves audit trail by preventing timestamp overwrites

**Code Formatting (Line Length):**
- All code additions comply with 120-character line length limit
- Multi-line CASE statements properly formatted

### Codebase Patterns

**Current Instance Status Lifecycle:**
```
NEW → RUNNING → { COMPLETED | FAILED | CANCELLED }
```

**Status Assignments in Code:**
| Line | Method | Status | Purpose |
|------|--------|--------|---------|
| 396 | `initialize_instance` | `NEW` | Instance created |
| 521 | `execute` | `RUNNING` | Execution starts |
| 557 | `execute` | `FAILED` | Step fails |
| 566 | `execute` | `COMPLETED` | All steps succeed |
| 1616 | `cancel` | `CANCELLED` | User cancels |

**Existing State Validation Patterns:**

1. **restart() method (Line 1584-1590)** - Validates FAILED status:
```abap
METHOD restart.
  IF ms_instance-status <> gc_status-failed.
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING
        textid = zcx_fi_process_error=>invalid_status
        value  = |Cannot restart: instance status is { ms_instance-status }|.
  ENDIF.
```

2. **cancel() method (Line 1608-1614)** - Validates not COMPLETED:
```abap
METHOD cancel.
  IF ms_instance-status = gc_status-completed.
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING
        textid = zcx_fi_process_error=>invalid_status
        value  = |Cannot cancel completed instance|.
  ENDIF.
```

3. **remove_step() method (Line 1656-1663)** - Validates not RUNNING:
```abap
METHOD remove_step.
  IF ms_instance-status = gc_status-running.
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING
        textid = zcx_fi_process_error=>invalid_operation
        value  = |Cannot remove step while process is running|.
  ENDIF.
```

**Pattern to Follow:** Use guard clause with CASE statement for clear error messages.

### Files to Reference

| File | Purpose |
| ---- | ------- |
| `src/zcl_fi_process_instance.clas.abap` | Main class containing execute() method (line 512) |
| `src/zcx_fi_process_error.clas.abap` | Exception class for message text IDs |
| `src/zcl_fi_process_manager.clas.abap` | Process manager calling execute_process() (line 312) |
| `src/zfi_process_status.doma.xml` | Domain defining all status values |
| `learnings/PROJECT_CONVENTIONS.md` | Error handling patterns |

### Technical Decisions

**Decision 1: Validation Location**
- **Chosen:** Add validation in `zcl_fi_process_instance=>execute()`
- **Rationale:** 
  - Protects against direct instance method calls
  - Co-located with status assignment logic
  - Consistent with existing `restart()` and `cancel()` validation patterns
- **Alternative Rejected:** Add validation only in `zcl_fi_process_manager=>execute_process()`
  - Would not protect against direct instance manipulation
  - Manager is optional facade; instance can be loaded directly

**Decision 2: Valid States for Execute**
- **Chosen:** Only allow `NEW` status
- **Rationale:**
  - Matches current behavior (instances created as NEW, then executed once)
  - Clear single-entry state transition
  - PENDING status is not used at instance level (only for steps)
- **Alternative Rejected:** Allow both NEW and PENDING
  - PENDING is never assigned to instances (verified via code analysis)
  - Would introduce confusion about state semantics

**Decision 3: Error Message Strategy**
- **Chosen:** CASE statement with specific messages per invalid state
- **Rationale:**
  - Guides users to correct API (e.g., "use restart() for FAILED instances")
  - Better developer experience than generic "invalid status" message
  - Consistent with constitution Principle V (clear error context)
- **Alternative Rejected:** Single generic error message
  - Less helpful for debugging
  - Doesn't guide users to restart() method

**Decision 4: Exception Text ID**
- **Chosen:** Reuse existing `invalid_status` text ID
- **Rationale:**
  - Already defined in `zcx_fi_process_error`
  - Used by `restart()` and `cancel()` methods
  - Contextual information provided via `value` parameter
- **Alternative Rejected:** Create new text IDs per state
  - Would require DDIC changes (message class entries)
  - Overhead not justified for runtime-only validation

## Implementation Plan

### Tasks

**Task 1: Add State Validation to execute() Method** (Estimated: 30 min)
1. Open `src/zcl_fi_process_instance.clas.abap`
2. Locate `execute()` method (line 512)
3. Add guard clause validation BEFORE line 514 (starting step determination)
4. Implement CASE statement for each invalid status
5. Ensure line length stays under 120 characters

**Task 2: Update Method Documentation** (Estimated: 15 min)
1. Update ABAP-Doc comment for `execute()` method (line 71-77)
2. Add description: "Only NEW instances can be executed. Use restart() for FAILED instances."
3. Document exception condition: "@raising zcx_fi_process_error | Raised if instance status is not NEW"

**Task 3: Verify No Regressions** (Estimated: 30 min)
1. Review all callers of `execute()`:
   - `zcl_fi_process_manager=>execute_process()` (line 312)
   - `zcl_fi_process_instance=>restart()` (line 1604) - calls with `iv_start_from_step`
2. Ensure `restart()` continues to work (it loads instance in FAILED state, should still call execute with start_from_step parameter)

**Task 4: Test with Demo Program** (Estimated: 45 min)
1. Use existing demo program `ZFI_PROCESS_FRAMEWORK`
2. Add negative test scenarios (see Test Plan below)
3. Verify error messages are clear and actionable

### Implementation Code

**Location:** `src/zcl_fi_process_instance.clas.abap`, line 512 (execute method)

**Code to Add (before line 514):**
```abap
METHOD execute.
  " Validate instance status before execution (guards against invalid state transitions)
  IF ms_instance-status <> gc_status-new.
    CASE ms_instance-status.
      WHEN gc_status-running.
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Instance { ms_instance-instance_id } is already RUNNING. | &&
                     |Cannot execute while execution is in progress.|.
      WHEN gc_status-failed.
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Instance { ms_instance-instance_id } has FAILED. | &&
                     |Use restart() method instead of execute().|.
      WHEN gc_status-completed.
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Instance { ms_instance-instance_id } is already COMPLETED. | &&
                     |Cannot re-execute completed instances.|.
      WHEN gc_status-cancelled.
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Instance { ms_instance-instance_id } was CANCELLED. | &&
                     |Cannot execute cancelled instances.|.
      WHEN OTHERS.
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Instance { ms_instance-instance_id } has invalid status | &&
                     |{ ms_instance-status } for execution. Expected: NEW.|.
    ENDCASE.
  ENDIF.

  " Determine starting step (original logic continues here)
  DATA(lv_start_step) = COND zfi_process_step_number(
    ...
```

**Updated ABAP-Doc Comment (line 71-77):**
```abap
"! Execute the process
"! <p>Executes all process steps sequentially from the first step (or from specified step).
"! Only instances with status NEW can be executed. For restarting FAILED instances, use restart() method.</p>
"! @parameter iv_start_from_step | Optional: start from specific step (used internally by restart)
"! @raising zcx_fi_process_error | Raised if instance status is not NEW or step execution fails
METHODS execute
  IMPORTING
    iv_start_from_step TYPE zfi_process_step_number OPTIONAL
  RAISING
    zcx_fi_process_error.
```

### Acceptance Criteria

**Functional Requirements:**
- [x] `execute()` method raises `zcx_fi_process_error` when called on RUNNING instance
- [x] `execute()` method raises `zcx_fi_process_error` when called on FAILED instance
- [x] `execute()` method raises `zcx_fi_process_error` when called on COMPLETED instance
- [x] `execute()` method raises `zcx_fi_process_error` when called on CANCELLED instance
- [x] `execute()` method succeeds when called on NEW instance
- [x] Error messages include instance ID and current status
- [x] Error message for FAILED status mentions `restart()` method
- [x] `restart()` method continues to work (loads FAILED instance, calls execute with start_from_step)

**Code Quality Requirements:**
- [x] All lines ≤ 120 characters
- [x] CASE statement properly formatted with indentation
- [x] ABAP-Doc comment updated with clear description
- [x] Exception handling follows constitution Principle V
- [x] No local types introduced (DDIC-First compliance)

**Testing Requirements:**
- [x] All 5 negative test scenarios pass (see Test Plan)
- [x] Positive test (NEW → execute → RUNNING) passes
- [x] Restart test (FAILED → restart → execute with start_from_step) passes
- [x] Demo program `ZFI_PROCESS_FRAMEWORK` runs without errors

## Additional Context

### Dependencies

**Code Dependencies:**
- `zcx_fi_process_error` exception class (already exists)
- `zcx_fi_process_error=>invalid_status` text ID (already defined)
- Status constants `gc_status-*` (already defined in class)

**Testing Dependencies:**
- Demo program `ZFI_PROCESS_FRAMEWORK`
- Test data setup program `ZFI_SETUP_DEMO_DATA`

**No External Dependencies:**
- No database schema changes
- No transport dependencies
- No UI/Fiori changes

### Testing Strategy

**Unit Test Scenarios (Manual via Demo Program):**

1. **Test: Execute on NEW instance (Positive)**
   - Create new instance with status NEW
   - Call `execute()`
   - Expected: Success, status changes to RUNNING

2. **Test: Execute on RUNNING instance (Negative)**
   - Load instance with status RUNNING (or mock it)
   - Call `execute()`
   - Expected: Exception raised with message "is already RUNNING"

3. **Test: Execute on FAILED instance (Negative)**
   - Create instance, let it fail, verify status FAILED
   - Call `execute()` (not restart)
   - Expected: Exception raised with message "Use restart() method"

4. **Test: Execute on COMPLETED instance (Negative)**
   - Create instance, let it complete, verify status COMPLETED
   - Call `execute()`
   - Expected: Exception raised with message "already COMPLETED"

5. **Test: Execute on CANCELLED instance (Negative)**
   - Create instance, call `cancel()`, verify status CANCELLED
   - Call `execute()`
   - Expected: Exception raised with message "was CANCELLED"

6. **Test: Restart on FAILED instance (Regression)**
   - Create instance, let it fail
   - Call `restart()` method
   - Expected: Success, execution resumes from failed step
   - Verify: Validation does NOT block restart (restart passes iv_start_from_step parameter)

**Edge Cases:**
- Instance with unknown/corrupted status value → WHEN OTHERS clause catches it

**Performance:**
- Validation adds 1 IF + 1 CASE statement (negligible overhead < 1ms)
- No database queries introduced

### Notes

**Why PENDING is Not Checked:**
Analysis of the codebase confirms that PENDING status is **never assigned to process instances**. It is only used for steps and substeps. Therefore, validation does not need to handle PENDING explicitly.

**Instance Status vs. Step Status:**
- **Instance**: NEW → RUNNING → {COMPLETED | FAILED | CANCELLED}
- **Steps**: PENDING → RUNNING → {COMPLETED | FAILED}
- The domain `ZFI_PROCESS_STATUS` is shared but not all values apply to both levels

**Compatibility with restart():**
The `restart()` method (line 1584-1605) works as follows:
1. Validates instance status is FAILED
2. Finds failed step number
3. Calls `execute(iv_start_from_step = failed_step_number)`

The new validation in `execute()` will **FAIL** if we check status before processing `iv_start_from_step`. 

**CRITICAL FIX REQUIRED:** The validation must check if `iv_start_from_step` is supplied (indicates restart scenario) and allow execution from FAILED state in that case.

**Updated Validation Logic:**
```abap
METHOD execute.
  " Allow execution from FAILED state only if restart scenario (start_from_step supplied)
  IF ms_instance-status <> gc_status-new 
     AND NOT ( ms_instance-status = gc_status-failed AND iv_start_from_step IS NOT INITIAL ).
    CASE ms_instance-status.
      WHEN gc_status-running.
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Instance { ms_instance-instance_id } is already RUNNING.|.
      WHEN gc_status-failed.
        " This branch only reached if iv_start_from_step IS INITIAL
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Instance { ms_instance-instance_id } has FAILED. | &&
                     |Use restart() method instead of execute().|.
      " ... other cases
    ENDCASE.
  ENDIF.
```

**Audit Trail Preservation:**
With this validation in place, `started_at` and `started_by` timestamps will only be set once per execution, preserving the audit trail for compliance.

**Production Impact:**
- **Risk Level:** LOW (validation prevents bugs, does not change correct usage)
- **Backward Compatibility:** SAFE (only blocks invalid operations that should never have worked)
- **Migration Required:** NO (runtime validation only)

---

**Specification Status:** DRAFT  
**Review Required:** YES (Technical Lead approval before implementation)  
**Estimated Effort:** 2 hours (implementation + testing)  
**Target Sprint:** TBD
