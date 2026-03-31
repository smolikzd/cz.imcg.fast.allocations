---
title: 'Dashboard Process Lifecycle Actions & RESTREQ Bug Fix'
slug: 'dashboard-process-lifecycle-actions-restreq-fix'
linear_issue: 'EST-136'
linear_url: 'https://linear.app/smolikzd/issue/EST-136/dashboard-process-lifecycle-actions-and-restreq-bug-fix'
created: '2026-03-30'
status: 'completed'
baseline_commit: 'e4f9944'
baseline_planner: '037c8a5'
baseline_ovysledovka: 'fb45e21'
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8, 9]
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

- [x] Task 1: Add `iv_no_commit` to `create_process()`
  - File: `zcl_fi_process_manager.clas.abap`
  - Action: Add `iv_no_commit TYPE abap_bool DEFAULT abap_false` parameter to `create_process()` method signature (around line 30). Forward it to the instance creation flow via `zcl_fi_process_instance=>create()`.
  - Notes: Default `abap_false` preserves backward compatibility for all existing callers (test report, etc.). Currently `create_process()` always commits because `initialize_instance()` calls `save_instance()` without `iv_no_commit`.

- [x] Task 2: Thread `iv_no_commit` through instance creation chain
  - File: `zcl_fi_process_instance.clas.abap`
  - Action: Thread `iv_no_commit` through the full chain: `create()` (line 445) -> `constructor()` (line 463) -> `initialize_instance()` (starts line 482, calls `save_instance()` at line 562) -> `save_instance()` (already supports `iv_no_commit` at line 640). Each method needs the new parameter added in BOTH the class DEFINITION and IMPLEMENTATION sections:
    - (a) `create()` static method: add `iv_no_commit TYPE abap_bool DEFAULT abap_false` to DEFINITION (PUBLIC SECTION, line ~52) and IMPLEMENTATION (line 445). Forward to `NEW` constructor call.
    - (b) `constructor()`: add `iv_no_commit TYPE abap_bool DEFAULT abap_false` to DEFINITION (PRIVATE SECTION, line ~308) and IMPLEMENTATION (line 463). Store in instance attribute or forward directly to `initialize_instance()`. Note: constructor has both a load path (`iv_instance_id` provided) and a create path (`iv_process_type` provided). The `iv_no_commit` parameter only matters for the create path — the load path does not call `save_instance()`. Default `abap_false` means the load path is unaffected.
    - (c) `initialize_instance()`: add `iv_no_commit TYPE abap_bool DEFAULT abap_false` to DEFINITION (PRIVATE SECTION, line ~318) and IMPLEMENTATION (line 482). Forward to `save_instance()` call at line 562.
    - (d) Update the `load()` factory method call (line ~456) to pass the new constructor parameter as default (no change needed if using `DEFAULT abap_false` — ABAP allows omitting optional parameters).
  - Notes: The chain is: `create_process(iv_no_commit)` -> `create(iv_no_commit)` -> `NEW zcl_fi_process_instance(iv_no_commit)` -> `initialize_instance(iv_no_commit)` -> `save_instance(iv_no_commit)`. The `load()` factory (line ~456) also calls the constructor but with `iv_instance_id` — it takes the load path and never calls `initialize_instance()`, so `iv_no_commit` is irrelevant there.

- [x] Task 3: Fix `execute()` status guard to accept RESTREQ
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
  - Notes: This allows `restart()` -> `execute(iv_start_from_step)` to succeed when instance is in RESTREQ status. Add a code comment explaining why RESTREQ is accepted (the restart flow calls `execute()` with `iv_start_from_step` while instance is still in RESTREQ).

- [x] Task 4: Fix silent exception swallowing in APJ job class + add `mark_as_failed()` method
  - Files: `zcl_fi_process_instance.clas.abap` (new method) AND `zcl_fi_process_job.clas.abap` (CATCH fix)
  - Action (part a — new method in `zcl_fi_process_instance.clas.abap`): Add a public `mark_as_failed()` method. This method does NOT exist yet — it must be added to both DEFINITION (PUBLIC SECTION) and IMPLEMENTATION:
    - DEFINITION: `METHODS mark_as_failed.` (no parameters)
    - IMPLEMENTATION:
      ```abap
      METHOD mark_as_failed.
        ms_instance-status   = gc_status-failed.
        ms_instance-ended_at = utclong_current( ).
        save_instance( ).
      ENDMETHOD.
      ```
    - Notes: No `iv_no_commit` — this method is called from the APJ job class (outside RAP saver), so COMMIT WORK is correct. The `save_instance()` default `iv_no_commit = abap_false` commits.
  - Action (part b — CATCH fix in `zcl_fi_process_job.clas.abap`): Replace the empty CATCH block at lines 172-174 with error handling. Wrap `mark_as_failed()` in its own TRY/CATCH to prevent double-fault (if `save_instance()` inside `mark_as_failed()` fails, the original error is still logged):
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
          TRY.
              lo_instance->mark_as_failed( ).
            CATCH zcx_fi_process_error.
              " mark_as_failed itself failed (e.g., DB lock)
              " Instance stays stuck — error already logged above
          ENDTRY.
        ENDIF.
      ENDIF.
    ```
  - Notes: The key requirements: (a) log the original error to application log, (b) transition out of EXECREQ/RESTREQ to FAILED, (c) if mark_as_failed fails, degrade gracefully — the error is already logged so the admin can investigate via SLG1.

#### Part B: Dashboard Changes (ovysledovka repo)

- [x] Task 5: Add `CreateAndExecute` action to behavior definition
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
    Also add `action CreateAndExecute` to the permissions list of all 4 existing side effects blocks (ExecuteProcess at line 16, CancelProcess at line 20, SupersedeProcess at line 24, RestartProcess at line 28).

- [x] Task 6: Add handler method for `CreateAndExecute`
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

- [x] Task 7: Add saver branch for `CREATEEXEC`
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

- [x] Task 8: Update feature control for new action
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
    (c) Status-based block (lines 84-111): add CreateAndExecute enablement for SUPERSEDED and CANCELLED:
    ```abap
    %features-%action-CreateAndExecute = COND #(
      WHEN lv_status = 'SUPERSEDED' OR lv_status = 'CANCELLED'
      THEN if_abap_behv=>fc-o-enabled
      ELSE if_abap_behv=>fc-o-disabled )
    ```

- [x] Task 9: Update `get_global_authorizations` for new action
  - File: `zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap`
  - Action: Add at line 275:
    ```abap
    result-%action-CreateAndExecute = if_abap_behv=>auth-allowed.
    ```

### Acceptance Criteria

> **Verification status (2026-03-31):** ACs 5, 6, 8 verified from code (ADT). ACs 1, 2, 3, 4, 7 require live SAP/Fiori UI testing — pending transport to dev system.

- [ ] AC 1: Given an empty dashboard row (no process instance), when the user clicks CreateAndExecute, then a new process instance is created with process_type ALLOCATIONS and init params from the row keys, and execution is requested via APJ (instance status transitions to EXECREQ).

- [ ] AC 2: Given a dashboard row with status SUPERSEDED or CANCELLED, when the user clicks CreateAndExecute, then a new process instance is created with the same allocation params, and execution is requested via APJ (new instance with EXECREQ; old SUPERSEDED/CANCELLED instance unchanged).

- [ ] AC 3: Given a dashboard row with status COMPLETED, when the user views the row, then CreateAndExecute is disabled. The user must first Supersede, then CreateAndExecute.

- [ ] AC 4: Given a dashboard row with status RUNNING, FAILED, EXECREQ, RESTREQ, or COMPLETED, when the user views the row, then CreateAndExecute is disabled.

- [x] AC 5: Given a FAILED instance where the user clicks Restart, when the APJ job fires and calls `restart()` -> `execute(iv_start_from_step)`, then the execute method accepts RESTREQ status and the instance resumes from the failed step (RESTREQ -> RUNNING). *(Code-verified 2026-03-31: `execute()` guard in `zcl_fi_process_instance` line 692 accepts `gc_status-restart_requested` when `iv_start_from_step` is provided.)*

- [x] AC 6: Given any APJ job where `execute()` or `restart()` raises an exception before step execution begins, when the exception is caught in the job class, then the error is logged to the application log AND the instance status is set to FAILED (not left stuck in EXECREQ/RESTREQ). *(Code-verified 2026-03-31: `zcl_fi_process_job` CATCH block line 172+ logs via logger then calls `mark_as_failed()` in nested TRY/CATCH.)*

- [ ] AC 7: Given an empty row where a duplicate active instance exists (same params, status RUNNING/FAILED/COMPLETED/EXECREQ/RESTREQ), when the user clicks CreateAndExecute, then the duplicate check raises an error and the Fiori UI displays the error message. *(Pending SAP/Fiori UI verification.)*

- [x] AC 8: Given existing callers of `create_process()` that do not pass `iv_no_commit`, when they call `create_process()`, then behavior is unchanged (default `iv_no_commit = abap_false` preserves COMMIT WORK AND WAIT). *(Code-verified 2026-03-31: `zcl_fi_process_manager->create_process()` line 35 has `iv_no_commit TYPE abap_bool DEFAULT abap_false`.)*

## Additional Context

### Dependencies

- EST-110 (SUPERSEDED status) -- already implemented, provides the supersede capability
- EST-134 (dashboard actions) -- already implemented, provides the handler/saver pattern being extended
- Tasks 1-2 (planner: `iv_no_commit` on `create_process`) must be completed before Task 7 (ovysledovka: saver CREATEEXEC branch)
- **Deployment ordering**: planner repo changes (Tasks 1-4) must be transported to the development system BEFORE ovysledovka changes (Tasks 5-9) can be activated. The ovysledovka BDEF and handler reference `iv_no_commit` on `create_process()` which only exists after planner transport.
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
- Task 4 references `lo_instance->mark_as_failed()` — this method does NOT exist. Task 4 (part a) specifies the full method: set status to FAILED, set `ended_at`, call `save_instance()` (with default commit). The call in the job class is wrapped in TRY/CATCH to handle double-fault gracefully.
- Threading `iv_no_commit` through instance creation (Task 2) touches the constructor. The full chain: `create_process(iv_no_commit)` -> `create(iv_no_commit)` -> `NEW zcl_fi_process_instance(iv_no_commit)` -> `initialize_instance(iv_no_commit)` -> `save_instance(iv_no_commit)`.
- The `COMMIT WORK AND WAIT` constraint in RAP savers is absolute -- any COMMIT in the save phase causes a short dump. This is why Task 1-2 is a prerequisite for Task 7.
- CANCELLED status: explicitly enabled for CreateAndExecute (decision made during pre-implementation review). The duplicate check already allows creation when only CANCELLED instances exist, so this is consistent.

## Spec Change Log

### Change 1 — Pre-implementation adversarial + edge case review (2026-03-30)

**Triggering findings:**
- M1 (bad_spec): Task 2 did not mention DEFINITION section changes or constructor load-path
- M2 (bad_spec): Task 4 lacked explicit `mark_as_failed()` method spec and double-fault TRY/CATCH
- M6 (intent_gap → resolved): CANCELLED status not addressed in feature control
- M9 (patch): Missing deployment ordering note
- M14 (patch): Missing code comment note on Task 3
- M15 (patch): Side effects blocks not enumerated in Task 5

**What was amended:**
- Task 2: Expanded to list all 4 DEFINITION+IMPLEMENTATION changes (create, constructor, initialize_instance, load path)
- Task 3: Added note about code comment for RESTREQ acceptance
- Task 4: Split into part a (new `mark_as_failed()` method with full spec) and part b (CATCH fix with TRY/CATCH around mark_as_failed)
- Task 5: Enumerated all 4 existing side effects blocks by line number
- Task 8: Added CANCELLED to CreateAndExecute enablement alongside SUPERSEDED
- AC 2: Updated to include CANCELLED
- AC 4: Added COMPLETED to disabled list (was implicit, now explicit)
- Dependencies: Added deployment ordering note (planner transport before ovysledovka activation)

**Known-bad state avoided:**
- Without M1 fix: developer would modify implementation but forget DEFINITION sections → compilation error
- Without M2 fix: `mark_as_failed()` call in pseudocode won't compile; double-fault could lose error logging
- Without M6 fix: CANCELLED rows locked out of re-execution despite duplicate check allowing it
