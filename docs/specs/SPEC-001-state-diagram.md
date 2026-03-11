# Instance Lifecycle State Diagram

## Current Implementation (Before Fix)

```
┌─────────────────────────────────────────────────────────────────┐
│                    CURRENT BEHAVIOR (UNSAFE)                    │
└─────────────────────────────────────────────────────────────────┘

    create()
      │
      ▼
   ┌─────┐
   │ NEW │◄──────────────────────────────────────┐
   └──┬──┘                                        │
      │                                           │
      │ execute()                                 │
      │ ✓ No validation!                          │
      ▼                                           │
  ┌─────────┐                                     │
  │ RUNNING │                                     │
  └────┬────┘                                     │
       │                                          │
       ├──────► execute() ──► UNSAFE!             │
       │        Overwrites                        │
       │        timestamps                        │
       │                                          │
   Steps complete                                │
   successfully                                  │
       │                                          │
       ▼                                          │
  ┌───────────┐                                  │
  │ COMPLETED │◄─────────────────────────────────┤
  └─────┬─────┘                                  │
        │                                         │
        └──────► execute() ──► UNSAFE!            │
                 Re-executes                     │
                 all steps                        │
                                                  │
   Step fails                                    │
      │                                           │
      ▼                                           │
  ┌────────┐                                      │
  │ FAILED │                                      │
  └────┬───┘                                      │
       │                                          │
       ├──────► execute() ──► UNSAFE!             │
       │        Should use restart()              │
       │                                          │
       └──────► restart() ──► Correct method      │
                │                                 │
                └─────────────────────────────────┘
                Calls execute(start_from_step)


   cancel()
      │
      ▼
  ┌───────────┐
  │ CANCELLED │
  └─────┬─────┘
        │
        └──────► execute() ──► UNSAFE!
                 No validation


Legend:
  ✓ Allowed operation
  ✗ Should be blocked but isn't
  ──► State transition
```

## Proposed Implementation (After Fix)

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROPOSED BEHAVIOR (SAFE)                     │
└─────────────────────────────────────────────────────────────────┘

    create()
      │
      ▼
   ┌─────┐
   │ NEW │
   └──┬──┘
      │
      │ execute() ✓
      │ Only valid entry point
      ▼
  ┌─────────┐
  │ RUNNING │
  └────┬────┘
       │
       ├──────► execute() ✗ BLOCKED
       │        Exception: "already RUNNING"
       │
   Steps complete
   successfully
       │
       ▼
  ┌───────────┐
  │ COMPLETED │
  └─────┬─────┘
        │
        ├──────► execute() ✗ BLOCKED
        │        Exception: "already COMPLETED"
        │
        └──────► cancel() ✗ BLOCKED (existing)
                 Exception: "cannot cancel completed"


   Step fails
      │
      ▼
  ┌────────┐
  │ FAILED │
  └────┬───┘
       │
       ├──────► execute() ✗ BLOCKED
       │        Exception: "use restart() method"
       │
       └──────► restart() ✓
                │
                └──── Validates status = FAILED
                      Calls execute(start_from_step)
                      Special bypass for restart


   cancel()
      │
      ▼
  ┌───────────┐
  │ CANCELLED │
  └─────┬─────┘
        │
        ├──────► execute() ✗ BLOCKED
        │        Exception: "was CANCELLED"
        │
        └──────► cancel() ✓ (idempotent)


Valid State Transitions:
  NEW       → RUNNING    (execute)
  RUNNING   → COMPLETED  (all steps succeed)
  RUNNING   → FAILED     (step fails)
  RUNNING   → CANCELLED  (cancel)
  FAILED    → RUNNING    (restart → execute with start_from_step)
  FAILED    → CANCELLED  (cancel)
  NEW       → CANCELLED  (cancel)

Blocked Operations (NEW):
  RUNNING   → execute()  ✗ "already RUNNING"
  COMPLETED → execute()  ✗ "already COMPLETED"
  FAILED    → execute()  ✗ "use restart()"
  CANCELLED → execute()  ✗ "was CANCELLED"

Existing Blocks (unchanged):
  COMPLETED → cancel()   ✗ "cannot cancel completed"
  RUNNING   → remove_step() ✗ "cannot remove while running"
```

## State Validation Logic

### execute() Method Guard Clause

```abap
METHOD execute.
  " Guard clause: Validate instance status before execution
  " Special case: Allow FAILED → RUNNING transition only via restart()
  "               (restart passes iv_start_from_step parameter)
  
  IF ms_instance-status <> gc_status-new 
     AND NOT ( ms_instance-status = gc_status-failed 
               AND iv_start_from_step IS NOT INITIAL ).
    
    CASE ms_instance-status.
      WHEN gc_status-running.
        " Prevent concurrent execution
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING textid = zcx_fi_process_error=>invalid_status
                    value  = |Instance already RUNNING|.
      
      WHEN gc_status-failed.
        " FAILED → execute() is blocked (must use restart)
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING textid = zcx_fi_process_error=>invalid_status
                    value  = |Instance FAILED. Use restart() method.|.
      
      WHEN gc_status-completed.
        " Prevent re-execution of completed processes
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING textid = zcx_fi_process_error=>invalid_status
                    value  = |Instance already COMPLETED|.
      
      WHEN gc_status-cancelled.
        " Prevent execution of cancelled processes
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING textid = zcx_fi_process_error=>invalid_status
                    value  = |Instance was CANCELLED|.
      
      WHEN OTHERS.
        " Handle corrupted/unknown status values
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING textid = zcx_fi_process_error=>invalid_status
                    value  = |Invalid status: { ms_instance-status }|.
    ENDCASE.
  ENDIF.
  
  " Original execute logic continues...
ENDMETHOD.
```

### restart() Method (Unchanged)

```abap
METHOD restart.
  " Existing validation: Only allow restart from FAILED state
  IF ms_instance-status <> gc_status-failed.
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING textid = zcx_fi_process_error=>invalid_status
                value  = |Cannot restart: status is { ms_instance-status }|.
  ENDIF.

  " Find failed step
  READ TABLE mt_steps INTO DATA(ls_failed_step)
    WITH KEY status = gc_status-failed.

  " Call execute with start_from_step parameter
  " This bypasses the NEW check via the special condition
  execute( iv_start_from_step = ls_failed_step-step_number ).
ENDMETHOD.
```

## Status Usage by Level

### Instance Level (zfi_proc_inst.status)

| Status | Used? | Set By | Purpose |
|--------|-------|--------|---------|
| NEW | ✓ YES | `create()` | Initial state after creation |
| RUNNING | ✓ YES | `execute()` | Process executing |
| COMPLETED | ✓ YES | `execute()` | All steps successful |
| FAILED | ✓ YES | `execute()` | Step failed |
| CANCELLED | ✓ YES | `cancel()` | User cancelled |
| **PENDING** | ✗ NO | N/A | **Not used at instance level** |
| QUEUED | ✗ NO | N/A | Not used at instance level |
| SKIPPED | ✗ NO | N/A | Not used at instance level |

### Step Level (zfi_proc_step.status)

| Status | Used? | Purpose |
|--------|-------|---------|
| NEW | ✗ NO | Steps created as PENDING |
| RUNNING | ✓ YES | Step executing |
| COMPLETED | ✓ YES | Step successful |
| FAILED | ✓ YES | Step failed |
| CANCELLED | ✓ YES | Process cancelled |
| **PENDING** | ✓ YES | **Step waiting to execute** |
| QUEUED | ✓ YES | Substep queued in bgRFC |
| SKIPPED | ✓ YES | Step skipped |

## Key Insights

1. **PENDING is step-only status** - Never assigned to instances
2. **Single entry point** - Only NEW instances can call execute()
3. **restart() has special bypass** - Uses iv_start_from_step parameter
4. **Audit trail preserved** - started_at/started_by set only once
5. **Backward compatible** - Only blocks invalid operations
