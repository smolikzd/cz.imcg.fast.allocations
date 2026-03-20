# Story EST-121.3: Status Constants and Background Request Methods

Status: ready-for-dev

## Story

As a framework developer,
I want `gc_status` constants for the new statuses and `request_execute()` / `request_restart()` methods on the process instance,
so that callers can schedule background execution and restart via SAP Application Jobs with proper status tracking and optimistic locking.

## Acceptance Criteria

1. Given `zcl_fi_process_instance`, when `gc_status` is inspected, then `exec_requested` (VALUE 'EXEC_REQUESTED') and `restart_requested` (VALUE 'RESTART_REQUESTED') exist in PUBLIC SECTION before `skipped`.
2. Given `zcl_fi_process_manager`, when `gc_status` is inspected, then same two constants exist in PRIVATE SECTION before `END OF gc_status`.
3. Given a NEW instance, when `request_execute()` is called, then status transitions to EXEC_REQUESTED, `JOB_NAME`/`JOB_COUNT` are populated, and an APJ job is scheduled. (Tech spec AC1)
4. Given a FAILED instance, when `request_restart()` is called, then status transitions to RESTART_REQUESTED and `JOB_NAME`/`JOB_COUNT` are populated. (Tech spec AC2)
5. Given a CANCELLED instance, when `request_restart()` is called, then status transitions to RESTART_REQUESTED. (Tech spec AC3)
6. Given `schedule_job()` raises `CX_APJ_RT`, when `request_execute()` is called, then `zcx_fi_process_error=>scheduling_failed` is raised and instance status remains NEW (unchanged). (Tech spec AC15)
7. Given a non-NEW instance, when `request_execute()` is called, then `zcx_fi_process_error=>invalid_status` is raised. (Tech spec AC11)
8. Given a non-FAILED/CANCELLED instance, when `request_restart()` is called, then `zcx_fi_process_error=>invalid_status` is raised. (Tech spec AC12)
9. Given a concurrent status change between scheduling and saving, when the optimistic lock check detects mismatch, then the orphaned job is cancelled best-effort and an exception is raised.

## Tasks / Subtasks

- [ ] Task 7: Add `gc_status` constants for new statuses to both classes (AC: #1, #2)
  - [ ] Open `src/zcl_fi_process_instance.clas.abap`
  - [ ] In PUBLIC SECTION `gc_status` block (lines 26-36), add `exec_requested` and `restart_requested` before `skipped`
  - [ ] Open `src/zcl_fi_process_manager.clas.abap`
  - [ ] In PRIVATE SECTION `gc_status` block (lines 178-187), add `exec_requested` and `restart_requested` before `END OF gc_status`
  - [ ] Verify line length ≤120 chars (VALUE 'RESTART_REQUESTED' is 17 chars — may need wrapping)

- [ ] Task 8: Add `request_execute()` method to `ZCL_FI_PROCESS_INSTANCE` (AC: #3, #6, #7, #9)
  - [ ] Add PUBLIC SECTION method declaration with ABAP-Doc (after `execute` method)
  - [ ] Add `gc_job_template` constant: `TYPE cl_apj_rt_api=>ty_template_name VALUE 'ZFI_PROCESS_JOB_TMPL'`
  - [ ] Implement `request_execute()`:
    1. Status guard: only NEW → raise `invalid_status` otherwise
    2. Schedule-first: build `ls_start_info` with timestamp = NOW + 2 seconds via `cl_abap_tstmp=>add()`
    3. Build `lt_params` with INSTGUID + ACTION='E' using `cl_apj_rt_api=>tt_job_parameter_value` structure
    4. Call `cl_apj_rt_api=>schedule_job()` — CATCH `cx_apj_rt` → raise `scheduling_failed`
    5. Optimistic lock: SELECT SINGLE status FROM zfi_proc_inst, verify still NEW
    6. If status changed → cancel orphaned job best-effort, raise `invalid_status`
    7. Status-second: set status = exec_requested, job_name, job_count, then `save_instance()`

- [ ] Task 9: Add `request_restart()` method to `ZCL_FI_PROCESS_INSTANCE` (AC: #4, #5, #8, #9)
  - [ ] Add PUBLIC SECTION method declaration with ABAP-Doc (after `restart` method)
  - [ ] Implement `request_restart()`:
    1. Status guard: only FAILED or CANCELLED → raise `invalid_status` otherwise
    2. Schedule-first: same timestamp pattern as request_execute()
    3. Build `lt_params` with INSTGUID + ACTION='R'
    4. Call `cl_apj_rt_api=>schedule_job()` — CATCH `cx_apj_rt` → raise `scheduling_failed`
    5. Optimistic lock: SELECT SINGLE status, verify still FAILED or CANCELLED
    6. If status changed → cancel orphaned job best-effort, raise `invalid_status`
    7. Status-second: set status = restart_requested, job_name, job_count, then `save_instance()`

## Dev Notes

### Target Repository
`cz.imcg.fast.planner`

### Constitution Compliance
- **Principle I (DDIC-First):** New constants use `TYPE zfi_process_status` (DDIC domain). Job reference fields use DDIC data elements.
- **Principle II (SAP Standards):** ABAP-Doc on all new public methods. Line length ≤120 chars.
- **Principle III (Consult SAP Docs):** `CL_APJ_RT_API=>SCHEDULE_JOB()` parameters verified. Use `cl_apj_rt_api=>ty_start_info`, `cl_apj_rt_api=>tt_job_parameter_value`, `cl_apj_rt_api=>ty_job_parameter_value`.
- **Principle V (Error Handling):** `scheduling_failed` and `invalid_status` exceptions raised with context. `cx_apj_rt` caught and wrapped.

### Dependencies
- **Depends on:** Story EST-121.1 (DDIC objects), Story EST-121.2 (job class + template name)
- **Blocks:** Story EST-121.4 (lifecycle extensions), Story EST-121.5 (manager methods)

### Technical Decisions
- **Schedule-first, status-second:** Always call `schedule_job()` BEFORE updating status. If scheduling fails, status remains unchanged.
- **Timestamp-based immediate start:** NOW + 2 seconds. `start_immediately = 'X'` is prohibited (implicit COMMIT WORK, not RAP-safe).
- **Optimistic locking:** After successful scheduling, re-read DB status. If it changed (race condition), cancel the orphaned job and raise exception.
- **LUW detail:** `save_instance()` does MODIFY + COMMIT WORK AND WAIT. The APJ job registration and status update are committed together.
- **Job references overwritten on new request:** Old job hits status guard and aborts silently.

### Files to Modify
| File | Action |
|------|--------|
| `src/zcl_fi_process_instance.clas.abap` | Modify — add gc_status constants, gc_job_template, request_execute(), request_restart() |
| `src/zcl_fi_process_manager.clas.abap` | Modify — add gc_status constants |

### Codebase Patterns
- gc_status block (instance, PUBLIC SECTION, lines 26-36): `CONSTANTS: BEGIN OF gc_status, new TYPE zfi_process_status VALUE 'NEW', ... END OF gc_status.`
- gc_status block (manager, PRIVATE SECTION, lines 178-187): Same pattern, 8 values currently (no `queued`)
- Schedule job parameter structure: `VALUE #( ( name = 'INSTGUID' t_value = VALUE #( ( sign = 'I' option = 'EQ' low = ... ) ) ) )`
- Existing save pattern: `ms_instance-status = ...; save_instance( ).` — save_instance does MODIFY + COMMIT WORK AND WAIT

### Critical Warnings
1. **Field access to ms_instance-job_name/job_count:** Verify that `ms_instance` structure type includes the new JOB_NAME/JOB_COUNT fields from Story EST-121.1. If `ty_process_inst` is derived from the DB table (via INCLUDE TYPE or direct reference), it auto-includes them. If it's a separate DDIC structure, it needs manual extension.
2. **iv_job_text max length:** Verify max length of `CL_APJ_RT_API=>TY_JOB_TEXT`. The text "ZFI_PROCESS exec <32-char-GUID>" is ~50 chars. If the type is shorter, truncate.
3. **cl_abap_tstmp=>add() availability:** Verify this method exists on ABAP 7.58. If not, use manual timestamp arithmetic.

### References
- [Source: tech-spec-est-121-apj-background-execution-lifecycle.md#Tasks 7-9]
- [Source: tech-spec-est-121-apj-background-execution-lifecycle.md#Technical Decisions 4, 5, 6, 11, 12]
- [Source: constitution.md#Principle I, II, III, V]

## Dev Agent Record

### Agent Model Used
### Debug Log References
### Completion Notes List
### File List
