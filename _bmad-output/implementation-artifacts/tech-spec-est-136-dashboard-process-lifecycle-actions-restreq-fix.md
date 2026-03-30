---
title: 'Dashboard Process Lifecycle Actions & RESTREQ Bug Fix'
slug: 'dashboard-process-lifecycle-actions-restreq-fix'
linear_issue: 'EST-136'
linear_url: 'https://linear.app/smolikzd/issue/EST-136/dashboard-process-lifecycle-actions-and-restreq-bug-fix'
created: '2026-03-30'
status: 'ready-for-dev'
stepsCompleted: [1, 2, 3, 4]
tech_stack: ['ABAP 7.58', 'RAP', 'CDS', 'Fiori Elements', 'APJ']
files_to_modify:
  - 'zcl_fi_process_instance.clas.abap (planner)'
  - 'zcl_fi_process_job.clas.abap (planner)'
  - 'zcl_fi_process_manager.clas.abap (planner)'
  - 'zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap (ovysledovka)'
  - 'zfi_i_alloc_dashboard_ce.bdef.asbdef (ovysledovka)'
code_patterns:
  - 'Two-phase RAP handler/saver pattern (buffer in handler, execute in saver)'
  - 'Factory pattern for process creation (ZCL_FI_PROCESS_MANAGER)'
  - 'DDIC-first (no local TYPE definitions)'
  - 'ZCX_FI_PROCESS_ERROR for all error handling'
  - 'Handler SELECT SINGLE from zfi_i_alloc_dashboard for validation'
  - 'Action buffer with CHAR10 action field: EXECUTE, CANCEL, SUPERSEDE, RESTART'
  - 'Saver CASE lv_action WHEN pattern with iv_no_commit = abap_true'
  - 'Feature control via get_instance_features with status-based enablement'
test_patterns: []
---

# Tech-Spec: Dashboard Process Lifecycle Actions & RESTREQ Bug Fix

**Created:** 2026-03-30

## Overview

### Problem Statement

The allocation dashboard lacks full process lifecycle management, causing three issues:

1. **No Create + Execute for empty rows:** Dashboard rows representing months with no allocation process instance have no way to create and execute a new process — users must use a separate tool or report (`zfi_alloc_proc_exec`). The dashboard's feature control disables all actions when `ProcessInstanceId IS INITIAL`.

2. **No Execute after Supersede:** After a user supersedes a COMPLETED instance (existing Supersede action), the row shows status SUPERSEDED but there is no action available to create a new instance and execute it. The user workflow should be: (Step 1) Supersede the COMPLETED instance, (Step 2) Execute on the now-SUPERSEDED row to create a new instance and run it.

3. **RESTREQ restart bug:** When a user clicks Restart on a FAILED instance, status changes to RESTREQ and an APJ job is scheduled. The job fires, loads the instance, and calls `restart()`, which internally calls `execute(iv_start_from_step = <failed_step>)`. However, `execute()`'s status guard (lines 673-676 of `zcl_fi_process_instance`) only accepts NEW, EXECREQ, or FAILED+start_from_step — it rejects RESTREQ and raises `zcx_fi_process_error`. The exception is silently caught in `zcl_fi_process_job` (line 172) with the incorrect assumption that "the instance already set itself to FAILED internally." In reality, the exception fires before any step runs, so the instance stays in RESTREQ permanently. The application log shows only "Process instance loaded" messages with no error — the failure is invisible to the user.

### Solution

**For issues #1 and #2:** Add a new `CreateAndExecute` action to the dashboard that creates a new process instance and requests execution. This action is enabled in two scenarios:
- **Empty rows** (`ProcessInstanceId IS INITIAL`): Create instance with `process_type = 'ALLOCATIONS'` and init params from row keys, then request execution via APJ.
- **SUPERSEDED rows** (`ProcessStatus = 'SUPERSEDED'`): Same flow — the duplicate check in `create_process()` already excludes SUPERSEDED instances, so creation succeeds. The user must first use the existing Supersede action on COMPLETED rows before this action becomes available.

**For issue #3:** Fix the `execute()` status guard in `zcl_fi_process_instance` to accept RESTREQ when `iv_start_from_step` is provided (matching the `restart()` flow). Additionally, fix the silent exception swallowing in `zcl_fi_process_job` to log the error and set the instance to FAILED when the exception fires before step execution.

### Scope

**In Scope:**
- New `CreateAndExecute` action on dashboard for empty rows and SUPERSEDED rows
- Feature control: enable new action when `ProcessInstanceId IS INITIAL` or `ProcessStatus = 'SUPERSEDED'`
- Behavior definition update with new action declaration and side effects
- Handler method: buffer action with row keys (no instance_id needed)
- Saver branch: call `create_process()` then `request_execute_process()` with `iv_no_commit = abap_true`
- Add `iv_no_commit` parameter to `create_process()` and thread through instance creation chain (framework change in planner repo)
- Fix `execute()` status guard in `zcl_fi_process_instance` to accept RESTREQ when `iv_start_from_step` is provided
- Fix silent exception swallowing in `zcl_fi_process_job` — log error, set instance to FAILED

**Out of Scope:**
- Changes to the dashboard CDS views or query provider (row structure stays the same)
- Changes to process creation/execution core logic beyond the status guard fix
- New DDIC objects (action buffer CHAR10 field fits new action value without changes)
- UI/Fiori Elements annotation changes (action visibility driven by feature control)
- Batch/mass execution of multiple rows at once
- Combo "supersede + create + execute" in one action (user does supersede as a separate step)

## Context for Development

### Codebase Patterns

- **Two-phase RAP pattern**: Handler validates and buffers actions into a static table (`gt_action_buffer` of type `zfi_alloc_tt_action_buffer`), saver executes manager calls with `iv_no_commit = abap_true`. Saver inherits from `cl_abap_behavior_saver_failed` to surface errors in save phase.
- **Handler validation pattern**: Each handler method does `SELECT SINGLE ProcessInstanceId FROM zfi_i_alloc_dashboard WHERE <keys>` to resolve the instance ID, then appends to buffer. The new action must skip this for empty rows (no instance to resolve).
- **Action buffer**: Structure `zfi_alloc_s_action_buffer` with fields: `FISCAL_YEAR`, `ALLOCATION_ID`, `COMPANY_CODE`, `FISCAL_PERIOD`, `INSTANCE_ID` (ZFI_PROCESS_INSTANCE_ID), `ACTION` (CHAR10). Existing action values: `'EXECUTE'`, `'CANCEL'`, `'SUPERSEDE'`, `'RESTART'`.
- **Saver pattern**: `LOOP AT lhc_dashboard=>gt_action_buffer`, `CASE ls_action-action`, each branch calls the appropriate manager method in TRY/CATCH. Buffer cleared in `cleanup`/`cleanup_finalize`.
- **Feature control**: `get_instance_features` reads status per key, returns `%features` with `if_abap_behv=>fc-o-<enabled/disabled>` per action. Currently returns all-disabled when `ProcessInstanceId IS INITIAL`.
- **Factory pattern**: All process instances created via `zcl_fi_process_manager=>create_process()`. No direct `NEW` instantiation.
- **DDIC-first**: All structures and table types are DDIC objects. No local TYPE definitions.
- **Error handling**: All errors raised as `zcx_fi_process_error` with semantic `textid` constants.
- **Process type**: Always `'ALLOCATIONS'` (hardcoded for this dashboard).
- **Init params**: `ZFI_ALLOC_PROCESS_PARAMS` with fields COMPANY_CODE, FISCAL_YEAR, FISCAL_PERIOD, ALLOCATION_ID, EXPORT.
- **Duplicate check**: `create_process()` blocks creation when RUNNING/FAILED/COMPLETED/EXECREQ/RESTREQ instance exists with same param hash. SUPERSEDED/CANCELLED allow new creation.
- **Supersede flow**: `supersede()` on instance only accepts COMPLETED status, sets to SUPERSEDED, persists. Manager's `supersede_process()` loads instance then delegates to `instance->supersede()`.
- **COMMIT constraint**: `create_process()` currently calls `initialize_instance()` -> `save_instance()` without `iv_no_commit`, causing `COMMIT WORK AND WAIT`. This is forbidden in RAP savers. Needs `iv_no_commit` parameter threaded through the creation chain.

### Files to Reference

| File | Repo | Purpose |
| ---- | ---- | ------- |
| `zcl_fi_process_instance.clas.abap` | planner | `execute()` guard at lines 673-676 (BUG); `restart()` lines 1935-1961; `initialize_instance()` line 562 calls `save_instance()` without no_commit; `create()` line 445 |
| `zcl_fi_process_job.clas.abap` | planner | APJ job, 207 lines; status guard lines 143-160; silent catch lines 172-174 (BUG) |
| `zcl_fi_process_manager.clas.abap` | planner | `create_process()` lines 272-364 (needs iv_no_commit); `request_execute_process()` lines 482-485 |
| `zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` | ovysledovka | Handler (lines 6-275) with 4 action methods + feature control; Saver (lines 278-381) with save/cleanup |
| `zfi_i_alloc_dashboard_ce.bdef.asbdef` | ovysledovka | Behavior definition with 4 actions + feature control mapping |
| `zfi_i_alloc_dashboard_ce.ddls.asddls` | ovysledovka | Custom entity CDS — reference only, no changes |
| `zfi_alloc_s_action_buffer.tabl.xml` | ovysledovka | Action buffer DDIC structure — no changes needed (CHAR10 fits new value) |
| `zfi_alloc_process_params.tabl.xml` | ovysledovka | Init params structure (5 fields) — reference for create_process call |
| `zfi_alloc_proc_exec.prog.abap` | ovysledovka | Test report showing create + execute pattern — reference implementation |

### Technical Decisions

1. **`execute()` guard fix**: Extend the existing guard to accept RESTREQ when `iv_start_from_step` is provided. This correctly models the semantic: RESTREQ + start_from_step means "restart was requested, now execute from the failed step." Less invasive than temporarily reverting to FAILED.

2. **Job class fix**: After catching `zcx_fi_process_error`, check if the instance status is still not FAILED/CANCELLED. If so, log the exception message via the instance logger and attempt to set status to FAILED. This makes failures visible in APJ monitoring (F5264).

3. **Two-step user workflow for re-execution**: The user first supersedes a COMPLETED instance (existing Supersede action), then executes on the SUPERSEDED row (new action). No combo action — keeps each action simple and auditable.

4. **Single new action for both scenarios**: One action handles both empty rows and SUPERSEDED rows. The handler buffers the action; the saver calls `create_process()` + `request_execute_process()`. For SUPERSEDED rows, the duplicate check already passes because SUPERSEDED is excluded.

5. **Feature control branching**: The feature control method must be restructured to handle the `ProcessInstanceId IS INITIAL` case — instead of disabling everything, enable only the new action. For non-initial instances, the new action is enabled when status = SUPERSEDED.

6. **No new DDIC objects**: The existing action buffer structure accommodates the new action value (`'CREATEEXEC'`) in its CHAR10 ACTION field.

7. **`create_process()` needs `iv_no_commit`**: The manager's `create_process()` internally calls `initialize_instance()` -> `save_instance()` without `iv_no_commit`, causing `COMMIT WORK AND WAIT`. This is forbidden in RAP savers. Fix: add `iv_no_commit TYPE abap_bool DEFAULT abap_false` and thread through the creation chain. Default preserves backward compatibility.

## Implementation Plan

### Tasks

#### Part A: Framework Fixes (planner repo)

- [ ] Task 1: Add `iv_no_commit` to `create_process()`
  - File: `zcl_fi_process_manager.clas.abap`
  - Action: Add `iv_no_commit TYPE abap_bool DEFAULT abap_false` parameter to `create_process()` method signature (around line 30). Forward it to the instance creation flow via `zcl_fi_process_instance=>create()`.
  - Notes: Default `abap_false` preserves backward compatibility for all existing callers (test report, etc.). Currently `create_process()` always commits because `initialize_instance()` calls `save_instance()` without `iv_no_commit`.

- [ ] Task 2: Thread `iv_no_commit` through instance creation chain
  - File: `zcl_fi_process_instance.clas.abap`
  - Action: Thread `iv_no_commit` through the full chain: `create()` (line 445) -> `constructor()` (line 463) -> `initialize_instance()` (line 482) -> `save_instance()` (line 562). Each method needs the new parameter added and forwarded. The `save_instance()` method already supports `iv_no_commit` (line 640) -- only the callers above it need changes.
  - Notes: The chain is: `create_process(iv_no_commit)` -> `create(iv_no_commit)` -> `NEW zcl_fi_process_instance(iv_no_commit)` -> `initialize_instance(iv_no_commit)` -> `save_instance(iv_no_commit)`.

- [ ] Task 3: Fix `execute()` status guard to accept RESTREQ
  - File: `zcl_fi_process_instance.clas.abap`
  - Action: At lines 673-676, extend the guard condition. Change from:
    ```abap
    IF ms_instance-status <> gc_status-new
       AND ms_instance-status <> gc_status-exec_requested
       AND NOT ( ms_instance-status = gc_status-failed
                 AND iv_start_from_step IS NOT INITIAL ).
    ```
    To:
    ```abap
    IF ms_instance-status <> gc_status-new
       AND ms_instance-status <> gc_status-exec_requested
       AND NOT ( ( ms_instance-status = gc_status-failed
                   OR ms_instance-status = gc_status-restart_requested )
                 AND iv_start_from_step IS NOT INITIAL ).
    ```
  - Notes: This allows `restart()` -> `execute(iv_start_from_step)` to succeed when instance is in RESTREQ status.

- [ ] Task 4: Fix silent exception swallowing in APJ job class
  - File: `zcl_fi_process_job.clas.abap`
  - Action: Replace the empty CATCH block at lines 172-174 with error handling:
    ```abap
    CATCH zcx_fi_process_error INTO DATA(lx_error).
      IF lo_instance IS BOUND.
        DATA(lv_cur_status) = lo_instance->get_status( ).
        IF lv_cur_status <> gc_sts_failed
           AND lv_cur_status <> gc_sts_cancelled.
          DATA(lo_log) = lo_instance->get_logger( ).
          IF lo_log IS BOUND.
            lo_log->message(
              iv_message_class  = lx_error->if_t100_message~t100key-msgid
              iv_message_number = lx_error->if_t100_message~t100key-msgno
              iv_message_v1     = lx_error->if_t100_dyn_msg~msgv1
              iv_message_v2     = lx_error->if_t100_dyn_msg~msgv2
              iv_message_v3     = lx_error->if_t100_dyn_msg~msgv3
              iv_message_v4     = lx_error->if_t100_dyn_msg~msgv4
              iv_severity       = 'E' ).
          ENDIF.
          lo_instance->mark_as_failed( ).
        ENDIF.
      ENDIF.
    ```
  - Notes: `mark_as_failed()` may not exist as a public method. During implementation, check available methods. If none exists, add a minimal public method that sets `ms_instance-status = gc_status-failed` and calls `save_instance()`. The key requirement: (a) log the error, (b) transition out of EXECREQ/RESTREQ to FAILED.

#### Part B: Dashboard Changes (ovysledovka repo)

- [ ] Task 5: Add `CreateAndExecute` action to behavior definition
  - File: `zfi_i_alloc_dashboard_ce.bdef.asbdef`
  - Action: Add new action declaration after existing actions (line 13):
    ```
    action ( features: instance ) CreateAndExecute external 'createAndExecute';
    ```
    Add side effects block (same pattern as existing):
    ```
    action CreateAndExecute affects field *,
      permissions ( action ExecuteProcess, action CancelProcess,
                    action SupersedeProcess, action RestartProcess,
                    action CreateAndExecute ),
      messages;
    ```
    Also add `action CreateAndExecute` to the permissions list of all existing side effects blocks.

- [ ] Task 6: Add handler method for `CreateAndExecute`
  - File: `zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap`
  - Action: (a) Add method declaration in `lhc_dashboard` PRIVATE SECTION:
    ```abap
    METHODS createandexecute FOR MODIFY
      IMPORTING keys FOR ACTION ZFI_I_ALLOC_DASHBOARD_CE~CreateAndExecute.
    ```
    (b) Implement method -- unlike existing handlers, no SELECT SINGLE needed. Buffer directly:
    ```abap
    METHOD createandexecute.
      LOOP AT keys INTO DATA(ls_key).
        APPEND VALUE #(
          fiscal_year   = ls_key-FiscalYear
          allocation_id = ls_key-AllocationId
          company_code  = ls_key-CompanyCode
          fiscal_period = ls_key-FiscalPeriod
          instance_id   = ''
          action        = 'CREATEEXEC'
        ) TO gt_action_buffer.
      ENDLOOP.
    ENDMETHOD.
    ```
  - Notes: No instance_id validation needed. The action is valid for rows without an instance. The saver creates the instance.

- [ ] Task 7: Add saver branch for `CREATEEXEC`
  - File: `zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap`
  - Action: In the `save` method, add WHEN branch before WHEN OTHERS (line 336):
    ```abap
    WHEN 'CREATEEXEC'.
      DATA(ls_params) = VALUE zfi_alloc_process_params(
        company_code  = ls_action-company_code
        fiscal_year   = ls_action-fiscal_year
        fiscal_period = ls_action-fiscal_period
        allocation_id = ls_action-allocation_id
        export        = abap_false ).
      DATA(lo_new_instance) = lo_manager->create_process(
        iv_process_type  = 'ALLOCATIONS'
        is_init_params   = ls_params
        iv_no_commit     = abap_true ).
      lo_manager->request_execute_process(
        iv_instance_id = lo_new_instance->get_instance_id( )
        iv_no_commit   = abap_true ).
    ```
  - Notes: Depends on Task 1-2 (`iv_no_commit` on `create_process`). The duplicate check in `create_process()` blocks if RUNNING/FAILED/COMPLETED/EXECREQ/RESTREQ instance exists -- error propagates to Fiori UI via existing CATCH block.

- [ ] Task 8: Update feature control for new action
  - File: `zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap`
  - Action: Modify `get_instance_features`:
    (a) Row-not-found block (lines 58-68): add `%features-%action-CreateAndExecute = if_abap_behv=>fc-o-disabled`.
    (b) `ProcessInstanceId IS INITIAL` block (lines 70-80): enable CreateAndExecute, disable rest:
    ```abap
    IF ls_db-ProcessInstanceId IS INITIAL.
      APPEND VALUE #(
        %tky = ls_key-%tky
        %features-%action-ExecuteProcess      = if_abap_behv=>fc-o-disabled
        %features-%action-CancelProcess       = if_abap_behv=>fc-o-disabled
        %features-%action-SupersedeProcess    = if_abap_behv=>fc-o-disabled
        %features-%action-RestartProcess      = if_abap_behv=>fc-o-disabled
        %features-%action-CreateAndExecute    = if_abap_behv=>fc-o-enabled
      ) TO result.
      CONTINUE.
    ENDIF.
    ```
    (c) Status-based block (lines 84-111): add CreateAndExecute enablement:
    ```abap
    %features-%action-CreateAndExecute = COND #(
      WHEN lv_status = 'SUPERSEDED'
      THEN if_abap_behv=>fc-o-enabled
      ELSE if_abap_behv=>fc-o-disabled )
    ```

- [ ] Task 9: Update `get_global_authorizations` for new action
  - File: `zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap`
  - Action: Add at line 275:
    ```abap
    result-%action-CreateAndExecute = if_abap_behv=>auth-allowed.
    ```

### Acceptance Criteria

- [ ] AC 1: Given an empty dashboard row (no process instance), when the user clicks CreateAndExecute, then a new process instance is created with process_type ALLOCATIONS and init params from the row keys, and execution is requested via APJ (instance status transitions to EXECREQ).

- [ ] AC 2: Given a dashboard row with status SUPERSEDED, when the user clicks CreateAndExecute, then a new process instance is created with the same allocation params, and execution is requested via APJ (new instance with EXECREQ; old SUPERSEDED instance unchanged).

- [ ] AC 3: Given a dashboard row with status COMPLETED, when the user views the row, then CreateAndExecute is disabled. The user must first Supersede, then CreateAndExecute.

- [ ] AC 4: Given a dashboard row with status RUNNING, FAILED, EXECREQ, or RESTREQ, when the user views the row, then CreateAndExecute is disabled.

- [ ] AC 5: Given a FAILED instance where the user clicks Restart, when the APJ job fires and calls `restart()` -> `execute(iv_start_from_step)`, then the execute method accepts RESTREQ status and the instance resumes from the failed step (RESTREQ -> RUNNING).

- [ ] AC 6: Given any APJ job where `execute()` or `restart()` raises an exception before step execution begins, when the exception is caught in the job class, then the error is logged to the application log AND the instance status is set to FAILED (not left stuck in EXECREQ/RESTREQ).

- [ ] AC 7: Given an empty row where a duplicate active instance exists (same params, status RUNNING/FAILED/COMPLETED/EXECREQ/RESTREQ), when the user clicks CreateAndExecute, then the duplicate check raises an error and the Fiori UI displays the error message.

- [ ] AC 8: Given existing callers of `create_process()` that do not pass `iv_no_commit`, when they call `create_process()`, then behavior is unchanged (default `iv_no_commit = abap_false` preserves COMMIT WORK AND WAIT).

## Additional Context

### Dependencies

- EST-110 (SUPERSEDED status) -- already implemented, provides the supersede capability
- EST-134 (dashboard actions) -- already implemented, provides the handler/saver pattern being extended
- Tasks 1-2 (planner: `iv_no_commit` on `create_process`) must be completed before Task 7 (ovysledovka: saver CREATEEXEC branch)
- Task 3 (planner: execute guard fix) is independent, can be done in parallel
- Task 4 (planner: job class fix) is independent, can be done in parallel

### Testing Strategy

**Manual testing (no unit test framework in place):**

1. **RESTREQ bug fix verification:**
   - Create a process instance via test report `zfi_alloc_proc_exec`
   - Force a step failure (e.g., invalid data for step 4)
   - Click Restart on the FAILED instance in the dashboard
   - Verify: instance transitions FAILED -> RESTREQ -> RUNNING -> COMPLETED (or FAILED with error logged)
   - Check application log (SLG1) -- should show step execution messages, not just "loaded"

2. **Job class error logging verification:**
   - Temporarily break a step class to force an immediate exception
   - Trigger execute/restart via dashboard
   - Verify: application log shows the error message, instance is FAILED (not stuck in EXECREQ/RESTREQ)

3. **CreateAndExecute on empty row:**
   - Navigate to dashboard, find a month with no process instance
   - Verify: CreateAndExecute button is enabled, other action buttons are disabled
   - Click CreateAndExecute
   - Verify: new instance created, status transitions to EXECREQ, APJ job visible in F5264

4. **CreateAndExecute on SUPERSEDED row:**
   - Find a COMPLETED instance, click Supersede
   - Verify: status changes to SUPERSEDED, CreateAndExecute becomes enabled
   - Click CreateAndExecute
   - Verify: new instance created with same params, old SUPERSEDED instance unchanged

5. **Duplicate guard:**
   - While a RUNNING instance exists for period X, try CreateAndExecute on same period
   - Verify: error message displayed in Fiori UI about duplicate instance

6. **Backward compatibility:**
   - Run `zfi_alloc_proc_exec` test report as before
   - Verify: create + execute flow still works identically (COMMIT happens as before)

### Notes

- The RESTREQ bug was confirmed via application log analysis (2026-03-30): two "Process instance loaded" messages on restart, no error, no step execution. The exception from `execute()` is silently swallowed by the job class.
- Deferred-work.md item #6 ("Create New Instance action gap") is directly addressed by this spec.
- The `EXPORT` field in `ZFI_ALLOC_PROCESS_PARAMS` is set to `abap_false` for normal execution. `FORCE_START` in additional params defaults to `abap_false`.
- Task 4 references `lo_instance->mark_as_failed()` -- this method may not exist. During implementation, check available public methods. If none exists, add a minimal public method that sets status to FAILED and calls `save_instance()`.
- Threading `iv_no_commit` through instance creation (Task 2) touches the constructor. The full chain: `create_process(iv_no_commit)` -> `create(iv_no_commit)` -> `NEW zcl_fi_process_instance(iv_no_commit)` -> `initialize_instance(iv_no_commit)` -> `save_instance(iv_no_commit)`.
- The `COMMIT WORK AND WAIT` constraint in RAP savers is absolute -- any COMMIT in the save phase causes a short dump. This is why Task 1-2 is a prerequisite for Task 7.
