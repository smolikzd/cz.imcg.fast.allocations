# Process Instance Lifecycle Documentation

**Document Type**: Architecture Documentation  
**Component**: ZFI_PROCESS Framework - Instance Lifecycle  
**Version**: 1.0.0  
**Date**: 2026-03-11  
**Author**: Technical Documentation  
**Status**: Final

---

## Table of Contents

1. [Overview](#overview)
2. [Instance Status Values](#instance-status-values)
3. [Lifecycle State Machine](#lifecycle-state-machine)
4. [Lifecycle Methods](#lifecycle-methods)
5. [Status Validation Rules](#status-validation-rules)
6. [Code Examples](#code-examples)
7. [Integration with Process Manager](#integration-with-process-manager)
8. [Error Handling](#error-handling)

---

## 1. Overview

### Purpose

This document describes the complete lifecycle of a process instance in the ZFI_PROCESS framework, including all status values, state transitions, and methods that affect the instance lifecycle.

### Scope

- **In Scope**: Process instance lifecycle only (class `zcl_fi_process_instance`)
- **Out of Scope**: Step/substep lifecycle (managed separately within instances)

### Key Principles

1. **Immutable Status Transitions**: Once COMPLETED or CANCELLED, instances cannot transition to other states
2. **Concurrency Protection**: RUNNING instances cannot be re-executed
3. **Restart Safety**: Only FAILED instances can be restarted
4. **Factory Pattern**: All instances created via factory methods (`create()` or `load()`)
5. **Status Validation**: All lifecycle methods validate current status before state transitions

---

## 2. Instance Status Values

Process instances use a **subset** of the `ZFI_PROCESS_STATUS` domain. The domain contains 8 values, but only 5 apply to instances.

### Used by Instances

| Status | Value | Description | Terminal? |
|--------|-------|-------------|-----------|
| `NEW` | `'N'` | Instance initialized, ready for first execution | No |
| `RUNNING` | `'R'` | Instance currently executing | No |
| `COMPLETED` | `'C'` | Instance finished successfully, all steps completed | **Yes** |
| `FAILED` | `'F'` | Instance failed due to step error | No (can restart) |
| `CANCELLED` | `'X'` | Instance cancelled by user/system | **Yes** |

### NOT Used by Instances

| Status | Value | Description | Used By |
|--------|-------|-------------|---------|
| `PENDING` | `'P'` | Awaiting execution | Steps only |
| `QUEUED` | `'Q'` | Queued for background execution | Steps only |
| `SKIPPED` | `'S'` | Skipped due to conditions | Steps only |

**Important**: PENDING status is **never** assigned to process instances. It is used exclusively for step-level lifecycle management.

---

## 3. Lifecycle State Machine

### State Transition Diagram

```
                    ┌─────────────────────────────────────┐
                    │                                     │
                    │  create() or load()                 │
                    │                                     │
                    └──────────────┬──────────────────────┘
                                   │
                                   ▼
                            ┌─────────────┐
                            │     NEW     │
                            └──────┬──────┘
                                   │
                                   │ execute()
                                   │
                                   ▼
                            ┌─────────────┐
                            │   RUNNING   │◄────────┐
                            └──────┬──────┘         │
                                   │                │
                    ┌──────────────┼──────────────┐ │
                    │              │              │ │
         Step fails │   All steps  │   cancel()   │ │
                    │   succeed    │              │ │
                    ▼              ▼              ▼ │
             ┌────────────┐ ┌────────────┐ ┌──────────────┐
             │   FAILED   │ │ COMPLETED  │ │  CANCELLED   │
             └─────┬──────┘ └────────────┘ └──────────────┘
                   │             │                 │
                   │             │                 │
                   │             └─────────────────┴─── TERMINAL STATES
                   │                                    (no further transitions)
                   │
                   │ restart()
                   │
                   └───────────────────────────────────┘
```

### State Transition Table

| From State | Method | To State | Condition |
|-----------|---------|----------|-----------|
| `NEW` | `execute()` | `RUNNING` | Always allowed |
| `RUNNING` | `execute()` | `FAILED` | Step execution fails |
| `RUNNING` | `execute()` | `COMPLETED` | All steps succeed |
| `RUNNING` | `cancel()` | `CANCELLED` | User/system cancellation |
| `FAILED` | `restart()` | `RUNNING` | Restart from failed step |
| `COMPLETED` | - | - | **Terminal state** |
| `CANCELLED` | - | - | **Terminal state** |

### Terminal States

**COMPLETED** and **CANCELLED** are **terminal states**:
- No method can transition from these states
- Attempting to execute/restart/cancel will raise `zcx_fi_process_error` with `invalid_status`
- Instances in terminal states can only be read or deleted

---

## 4. Lifecycle Methods

### 4.1 Factory Methods (Instance Creation)

#### `create()` - Create New Instance

**Signature**:
```abap
CLASS-METHODS create
  IMPORTING
    iv_process_id        TYPE zfi_process_id
    iv_config            TYPE zfi_proc_config OPTIONAL
    iv_description       TYPE zfi_proc_desc OPTIONAL
    iv_background        TYPE abap_bool DEFAULT abap_false
    iv_auto_commit       TYPE abap_bool DEFAULT abap_true
  RETURNING
    VALUE(ro_instance)   TYPE REF TO zcl_fi_process_instance
  RAISING
    zcx_fi_process_error.
```

**Behavior**:
- Creates new instance with status = `NEW`
- Assigns unique instance ID (GUID)
- Initializes timestamps (created_at, updated_at)
- Stores instance in database table `zfi_proc_inst`
- Returns instance reference

**Initial Status**: `NEW`

**Code Location**: `zcl_fi_process_instance.clas.abap:41-50`

---

#### `load()` - Load Existing Instance

**Signature**:
```abap
CLASS-METHODS load
  IMPORTING
    iv_instance_id       TYPE zfi_proc_inst_id
  RETURNING
    VALUE(ro_instance)   TYPE REF TO zcl_fi_process_instance
  RAISING
    zcx_fi_process_error.
```

**Behavior**:
- Loads instance from database by instance ID
- Restores status from persisted data
- Loads all steps and substeps
- Raises exception if instance not found

**Status**: Preserved from database

**Code Location**: `zcl_fi_process_instance.clas.abap:55-61`

---

### 4.2 Execution Methods

#### `execute()` - Execute Process Instance

**Signature**:
```abap
METHODS execute
  IMPORTING
    iv_start_from_step   TYPE i OPTIONAL
  RAISING
    zcx_fi_process_error.
```

**Behavior**:
1. **Validates status** (see Status Validation Rules below)
2. Sets status to `RUNNING`
3. Updates `started_at` timestamp
4. Executes steps sequentially (or from `iv_start_from_step`)
5. On success: Sets status to `COMPLETED`, updates `completed_at`
6. On failure: Sets status to `FAILED`, captures error details

**Status Transitions**:
- `NEW` → `RUNNING` → `{COMPLETED | FAILED}`
- `FAILED` → `RUNNING` → `{COMPLETED | FAILED}` (only when called by `restart()`)

**Validation Rules**:
- ✅ **Allowed**: Status = `NEW`
- ✅ **Allowed**: Status = `FAILED` AND `iv_start_from_step` IS NOT INITIAL (restart bypass)
- ❌ **Blocked**: Status = `RUNNING` (concurrency protection)
- ❌ **Blocked**: Status = `COMPLETED` (immutability)
- ❌ **Blocked**: Status = `CANCELLED` (immutability)
- ❌ **Blocked**: Status = `FAILED` AND `iv_start_from_step` IS INITIAL (use `restart()` instead)

**Exception Raised**:
```abap
RAISE EXCEPTION TYPE zcx_fi_process_error
  EXPORTING
    textid       = zcx_fi_process_error=>invalid_status
    mv_msgv1     = 'execute()'
    mv_msgv2     = <current_status>
    mv_msgv3     = 'NEW'
    mv_object_id = mv_instance_id.
```

**Code Location**: `zcl_fi_process_instance.clas.abap:516-620`

**Usage Example**:
```abap
DATA(lo_instance) = zcl_fi_process_instance=>create(
  iv_process_id = 'ALLOC_PROCESS' ).

lo_instance->add_step( iv_step_class = 'ZCL_FI_ALLOC_STEP_LOAD' ).
lo_instance->add_step( iv_step_class = 'ZCL_FI_ALLOC_STEP_CALC' ).

TRY.
    lo_instance->execute( ).  " NEW → RUNNING → COMPLETED
    MESSAGE 'Process completed successfully' TYPE 'S'.
  CATCH zcx_fi_process_error INTO DATA(lx_error).
    MESSAGE lx_error->get_text( ) TYPE 'E'.
ENDTRY.
```

---

#### `restart()` - Restart Failed Instance

**Signature**:
```abap
METHODS restart
  RAISING
    zcx_fi_process_error.
```

**Behavior**:
1. **Validates status** = `FAILED` (raises exception otherwise)
2. Identifies failed step number
3. Calls `execute( iv_start_from_step = failed_step_number )`
4. Bypasses execute() validation via `iv_start_from_step` parameter

**Status Transitions**:
- `FAILED` → `RUNNING` → `{COMPLETED | FAILED}`

**Validation Rules**:
- ✅ **Allowed**: Status = `FAILED`
- ❌ **Blocked**: Status ≠ `FAILED` (any other status)

**Exception Raised**:
```abap
RAISE EXCEPTION TYPE zcx_fi_process_error
  EXPORTING
    textid       = zcx_fi_process_error=>invalid_status
    mv_msgv1     = 'restart()'
    mv_msgv2     = mv_status
    mv_msgv3     = 'FAILED'
    mv_object_id = mv_instance_id.
```

**Code Location**: `zcl_fi_process_instance.clas.abap:1633-1653`

**Usage Example**:
```abap
" First execution fails
TRY.
    lo_instance->execute( ).
  CATCH zcx_fi_process_error.
    " Handle error, fix data
ENDTRY.

" Restart from failed step
IF lo_instance->get_status( ) = zcl_fi_process_instance=>c_status_failed.
  TRY.
      lo_instance->restart( ).  " FAILED → RUNNING → COMPLETED
      MESSAGE 'Process restarted and completed' TYPE 'S'.
    CATCH zcx_fi_process_error INTO DATA(lx_error).
      MESSAGE lx_error->get_text( ) TYPE 'E'.
  ENDTRY.
ENDIF.
```

---

#### `cancel()` - Cancel Running Instance

**Signature**:
```abap
METHODS cancel
  RAISING
    zcx_fi_process_error.
```

**Behavior**:
1. **Validates status** ≠ `COMPLETED` (cannot cancel completed instances)
2. Sets status to `CANCELLED`
3. Updates `cancelled_at` timestamp
4. Cancels all running/pending steps

**Status Transitions**:
- `NEW` → `CANCELLED`
- `RUNNING` → `CANCELLED`
- `FAILED` → `CANCELLED`

**Validation Rules**:
- ✅ **Allowed**: Status = `NEW`, `RUNNING`, `FAILED`
- ❌ **Blocked**: Status = `COMPLETED` (immutability)
- ❌ **Blocked**: Status = `CANCELLED` (already cancelled)

**Exception Raised**:
```abap
RAISE EXCEPTION TYPE zcx_fi_process_error
  EXPORTING
    textid       = zcx_fi_process_error=>invalid_status
    mv_msgv1     = 'cancel()'
    mv_msgv2     = mv_status
    mv_msgv3     = 'not COMPLETED or CANCELLED'
    mv_object_id = mv_instance_id.
```

**Code Location**: `zcl_fi_process_instance.clas.abap:1658-1668`

**Usage Example**:
```abap
" User requests cancellation
IF lo_instance->get_status( ) = zcl_fi_process_instance=>c_status_running.
  TRY.
      lo_instance->cancel( ).  " RUNNING → CANCELLED
      MESSAGE 'Process cancelled' TYPE 'S'.
    CATCH zcx_fi_process_error INTO DATA(lx_error).
      MESSAGE lx_error->get_text( ) TYPE 'E'.
  ENDTRY.
ENDIF.
```

---

### 4.3 Query Methods

#### `get_status()` - Read Current Status

**Signature**:
```abap
METHODS get_status
  RETURNING
    VALUE(rv_status) TYPE zfi_process_status.
```

**Behavior**:
- Returns current instance status
- Does not modify state
- No validation required

**Code Location**: `zcl_fi_process_instance.clas.abap:98-101`

**Usage Example**:
```abap
DATA(lv_status) = lo_instance->get_status( ).

CASE lv_status.
  WHEN zcl_fi_process_instance=>c_status_new.
    " Handle new instance
  WHEN zcl_fi_process_instance=>c_status_running.
    " Handle running instance
  WHEN zcl_fi_process_instance=>c_status_completed.
    " Handle completed instance
  WHEN zcl_fi_process_instance=>c_status_failed.
    " Handle failed instance
  WHEN zcl_fi_process_instance=>c_status_cancelled.
    " Handle cancelled instance
ENDCASE.
```

---

### 4.4 Configuration Methods (Status-Aware)

#### `remove_step()` - Remove Step from Instance

**Signature**:
```abap
METHODS remove_step
  IMPORTING
    iv_step_number TYPE i
  RAISING
    zcx_fi_process_error.
```

**Behavior**:
1. **Validates status** ≠ `RUNNING` (cannot modify running instances)
2. Removes step from configuration
3. Renumbers subsequent steps

**Validation Rules**:
- ✅ **Allowed**: Status = `NEW`, `FAILED`, `COMPLETED`, `CANCELLED`
- ❌ **Blocked**: Status = `RUNNING` (data consistency protection)

**Code Location**: `zcl_fi_process_instance.clas.abap:1706-1727`

---

## 5. Status Validation Rules

### Summary Table

| Method | NEW | RUNNING | COMPLETED | FAILED | CANCELLED |
|--------|-----|---------|-----------|--------|-----------|
| `execute()` | ✅ | ❌ | ❌ | ❌* | ❌ |
| `restart()` | ❌ | ❌ | ❌ | ✅ | ❌ |
| `cancel()` | ✅ | ✅ | ❌ | ✅ | ❌ |
| `remove_step()` | ✅ | ❌ | ✅ | ✅ | ✅ |
| `get_status()` | ✅ | ✅ | ✅ | ✅ | ✅ |

**Legend**:
- ✅ = Allowed
- ❌ = Blocked (raises `zcx_fi_process_error` with `invalid_status`)
- ❌* = Blocked unless called internally by `restart()` with `iv_start_from_step`

### Validation Implementation Pattern

All lifecycle methods follow this pattern:

```abap
METHOD lifecycle_method.
  " Guard clause: validate status
  IF mv_status <> c_status_expected.
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING
        textid       = zcx_fi_process_error=>invalid_status
        mv_msgv1     = 'method_name()'
        mv_msgv2     = mv_status
        mv_msgv3     = 'expected_status'
        mv_object_id = mv_instance_id.
  ENDIF.

  " Execute method logic
  " ...
ENDMETHOD.
```

---

## 6. Code Examples

### Example 1: Complete Lifecycle (Success Path)

```abap
" Create new instance
DATA(lo_instance) = zcl_fi_process_instance=>create(
  iv_process_id  = 'ALLOC_PROCESS'
  iv_description = 'Cost allocation process' ).

" Status: NEW

" Configure steps
lo_instance->add_step( iv_step_class = 'ZCL_FI_ALLOC_STEP_LOAD' ).
lo_instance->add_step( iv_step_class = 'ZCL_FI_ALLOC_STEP_CALC' ).
lo_instance->add_step( iv_step_class = 'ZCL_FI_ALLOC_STEP_POST' ).

" Execute
TRY.
    lo_instance->execute( ).
    " NEW → RUNNING → COMPLETED
    
    IF lo_instance->get_status( ) = zcl_fi_process_instance=>c_status_completed.
      MESSAGE 'Process completed successfully' TYPE 'S'.
    ENDIF.
    
  CATCH zcx_fi_process_error INTO DATA(lx_error).
    MESSAGE lx_error->get_text( ) TYPE 'E'.
ENDTRY.
```

---

### Example 2: Restart After Failure

```abap
" Execute process
TRY.
    lo_instance->execute( ).
  CATCH zcx_fi_process_error INTO DATA(lx_error).
    " Step 2 failed
    " Status: FAILED
    MESSAGE lx_error->get_text( ) TYPE 'E'.
ENDTRY.

" Fix underlying data issue
" ...

" Restart from failed step
IF lo_instance->get_status( ) = zcl_fi_process_instance=>c_status_failed.
  TRY.
      lo_instance->restart( ).
      " FAILED → RUNNING → COMPLETED
      MESSAGE 'Process restarted successfully' TYPE 'S'.
    CATCH zcx_fi_process_error INTO DATA(lx_restart_error).
      MESSAGE lx_restart_error->get_text( ) TYPE 'E'.
  ENDTRY.
ENDIF.
```

---

### Example 3: User Cancellation

```abap
" Start background process
DATA(lo_instance) = zcl_fi_process_instance=>create(
  iv_process_id = 'LONG_RUNNING_PROCESS'
  iv_background = abap_true ).

lo_instance->execute( ).
" Status: RUNNING (in background)

" User clicks "Cancel" button
IF lo_instance->get_status( ) = zcl_fi_process_instance=>c_status_running.
  TRY.
      lo_instance->cancel( ).
      " RUNNING → CANCELLED
      MESSAGE 'Process cancelled by user' TYPE 'S'.
    CATCH zcx_fi_process_error INTO DATA(lx_error).
      MESSAGE lx_error->get_text( ) TYPE 'E'.
  ENDTRY.
ENDIF.
```

---

### Example 4: Validation Error Handling

```abap
" Attempt to execute COMPLETED instance (invalid)
DATA(lo_instance) = zcl_fi_process_instance=>load(
  iv_instance_id = '550E8400E29B41D4A716446655440000' ).

" Instance is already COMPLETED
IF lo_instance->get_status( ) = zcl_fi_process_instance=>c_status_completed.
  TRY.
      lo_instance->execute( ).  " This will fail
    CATCH zcx_fi_process_error INTO DATA(lx_error).
      " lx_error->get_text( ) = "Cannot execute() on instance in status COMPLETED (expected: NEW)"
      MESSAGE lx_error->get_text( ) TYPE 'E'.
  ENDTRY.
ENDIF.
```

---

### Example 5: Status-Based UI Logic

```abap
METHOD display_process_actions.
  DATA(lv_status) = lo_instance->get_status( ).

  CASE lv_status.
    WHEN zcl_fi_process_instance=>c_status_new.
      " Show: [Execute] [Cancel]
      enable_button( 'EXECUTE' ).
      enable_button( 'CANCEL' ).
      
    WHEN zcl_fi_process_instance=>c_status_running.
      " Show: [Cancel] (disabled: Execute, Restart)
      disable_button( 'EXECUTE' ).
      enable_button( 'CANCEL' ).
      disable_button( 'RESTART' ).
      
    WHEN zcl_fi_process_instance=>c_status_failed.
      " Show: [Restart] [Cancel]
      disable_button( 'EXECUTE' ).
      enable_button( 'RESTART' ).
      enable_button( 'CANCEL' ).
      
    WHEN zcl_fi_process_instance=>c_status_completed.
      " Show: nothing (all disabled)
      disable_button( 'EXECUTE' ).
      disable_button( 'RESTART' ).
      disable_button( 'CANCEL' ).
      
    WHEN zcl_fi_process_instance=>c_status_cancelled.
      " Show: nothing (all disabled)
      disable_button( 'EXECUTE' ).
      disable_button( 'RESTART' ).
      disable_button( 'CANCEL' ).
  ENDCASE.
ENDMETHOD.
```

---

## 7. Integration with Process Manager

The `zcl_fi_process_manager` class provides high-level lifecycle orchestration:

### Manager Methods Wrapping Instance Lifecycle

```abap
" Execute process
zcl_fi_process_manager=>execute_process(
  iv_process_id = 'ALLOC_PROCESS' ).
" Internally: create() → execute()

" Restart failed process
zcl_fi_process_manager=>restart_process(
  iv_instance_id = lv_instance_id ).
" Internally: load() → restart()

" Cancel running process
zcl_fi_process_manager=>cancel_process(
  iv_instance_id = lv_instance_id ).
" Internally: load() → cancel()
```

**Code Location**: `zcl_fi_process_manager.clas.abap:312-325`

---

## 8. Error Handling

### Exception: `zcx_fi_process_error`

All lifecycle methods raise `zcx_fi_process_error` for status validation failures.

**Exception Attributes**:
```abap
textid       = zcx_fi_process_error=>invalid_status
mv_msgv1     = '<method_name>'       " e.g., 'execute()'
mv_msgv2     = '<current_status>'    " e.g., 'COMPLETED'
mv_msgv3     = '<expected_status>'   " e.g., 'NEW'
mv_object_id = '<instance_id>'       " e.g., '550E8400...'
```

**Error Message Format**:
```
Cannot <method_name> on instance in status <current_status> (expected: <expected_status>)
Instance ID: <instance_id>
```

**Example**:
```
Cannot execute() on instance in status COMPLETED (expected: NEW)
Instance ID: 550E8400E29B41D4A716446655440000
```

### Error Handling Best Practices

```abap
TRY.
    lo_instance->execute( ).
  CATCH zcx_fi_process_error INTO DATA(lx_error).
    " Log error details
    zcl_logger=>log_exception(
      ix_exception = lx_error
      iv_severity  = 'ERROR' ).
    
    " Display user-friendly message
    MESSAGE lx_error->get_text( ) TYPE 'E'.
    
    " Take corrective action based on status
    CASE lo_instance->get_status( ).
      WHEN zcl_fi_process_instance=>c_status_failed.
        " Offer restart option
      WHEN zcl_fi_process_instance=>c_status_running.
        " Offer cancel option
    ENDCASE.
ENDTRY.
```

---

## Appendix A: Status Constants

All status constants defined in `zcl_fi_process_instance`:

```abap
CONSTANTS:
  c_status_new       TYPE zfi_process_status VALUE 'N',
  c_status_pending   TYPE zfi_process_status VALUE 'P',  " NOT used by instances
  c_status_running   TYPE zfi_process_status VALUE 'R',
  c_status_completed TYPE zfi_process_status VALUE 'C',
  c_status_failed    TYPE zfi_process_status VALUE 'F',
  c_status_queued    TYPE zfi_process_status VALUE 'Q',  " NOT used by instances
  c_status_cancelled TYPE zfi_process_status VALUE 'X',
  c_status_skipped   TYPE zfi_process_status VALUE 'S'.  " NOT used by instances
```

**Code Location**: `zcl_fi_process_instance.clas.abap:26-35`

---

## Appendix B: Database Schema

Process instances persisted in `zfi_proc_inst` table:

**Key Fields**:
- `instance_id` (GUID) - Primary key
- `process_id` (CHAR 30) - Process definition identifier
- `status` (CHAR 1) - Current lifecycle status
- `started_at` (TIMESTAMP) - Execution start time
- `completed_at` (TIMESTAMP) - Completion time (NULL if not completed)
- `cancelled_at` (TIMESTAMP) - Cancellation time (NULL if not cancelled)
- `error_message` (STRING) - Error details (NULL if no error)

**Status Transitions Tracked by Timestamps**:
- `created_at` → Status = NEW
- `started_at` → Status = RUNNING
- `completed_at` → Status = COMPLETED
- `cancelled_at` → Status = CANCELLED
- Error captured → Status = FAILED

---

## Appendix C: Related Documentation

- **Technical Specification**: `docs/specs/SPEC-001-instance-lifecycle-validation.md`
- **State Diagrams**: `docs/specs/SPEC-001-state-diagram.md`
- **Implementation Summary**: `docs/specs/SPEC-001-implementation-summary.md`
- **Framework Constitution**: `_bmad/_memory/constitution.md`

---

**Document Version**: 1.0.0  
**Last Updated**: 2026-03-11  
**Maintained By**: ZFI_PROCESS Framework Team  
**Review Cycle**: Quarterly
