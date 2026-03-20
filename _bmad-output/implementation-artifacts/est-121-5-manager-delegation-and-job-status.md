# Story EST-121.5: Manager Delegation and Job Status Check

Status: ready-for-dev

## Story

As a framework developer,
I want `check_job_status()` on the instance for APJ reconciliation and three manager-level delegation methods (`request_execute_process()`, `request_restart_process()`, `check_job_status()`),
so that callers can schedule and monitor background jobs through the standard manager facade.

## Acceptance Criteria

1. Given an instance with no job reference (foreground execution), when `check_job_status()` is called, then it returns "No background job reference". (Tech spec AC9)
2. Given an instance with a job reference, when `check_job_status()` is called, then it returns a string containing both the instance status and the APJ job status. (Tech spec AC10)
3. Given `CL_APJ_RT_API=>GET_JOB_STATUS()` fails (throws CX_APJ_RT), when `check_job_status()` is called, then it returns an error message string (not an exception).
4. Given a valid instance ID, when `request_execute_process()` is called on the manager, then it delegates to `lo_instance->request_execute()`.
5. Given a valid instance ID, when `request_restart_process()` is called on the manager, then it delegates to `lo_instance->request_restart()`.
6. Given a valid instance ID, when `check_job_status()` is called on the manager, then it delegates to `lo_instance->check_job_status()`.
7. Given an invalid instance ID, when any manager method is called, then `zcx_fi_process_error` is raised (from `load_process()`).

## Tasks / Subtasks

- [ ] Task 13: Add `check_job_status()` method to `ZCL_FI_PROCESS_INSTANCE` (AC: #1, #2, #3)
  - [ ] Add PUBLIC SECTION method declaration with ABAP-Doc: `METHODS check_job_status RETURNING VALUE(rv_job_status) TYPE string.`
  - [ ] Implement: if job_name is initial → return "No background job reference"
  - [ ] Otherwise: TRY `cl_apj_rt_api=>get_job_status()` with iv_jobname + iv_jobcount
  - [ ] Build return string: "Instance status: {status}, APJ job status: {statustext} ({status}), Job: {name}/{count}"
  - [ ] CATCH `cx_apj_rt` → return "Job status check failed: {error text}"
  - [ ] Method does NOT raise exceptions — all errors returned as strings
  - [ ] Method does NOT modify instance status — report-only

- [ ] Task 14: Add manager delegation methods (AC: #4, #5, #6, #7)
  - [ ] Add `request_execute_process()` to PUBLIC SECTION:
    ```
    METHODS request_execute_process
      IMPORTING iv_instance_id TYPE zfi_process_instance_id
      RAISING zcx_fi_process_error.
    ```
  - [ ] Implement: `load_process( iv_instance_id )->request_execute( ).`
  - [ ] Add `request_restart_process()` — same pattern, delegates to `request_restart()`
  - [ ] Add `check_job_status()` to PUBLIC SECTION:
    ```
    METHODS check_job_status
      IMPORTING iv_instance_id TYPE zfi_process_instance_id
      RETURNING VALUE(rv_job_status) TYPE string
      RAISING zcx_fi_process_error.
    ```
  - [ ] Implement: `rv_job_status = load_process( iv_instance_id )->check_job_status( ).`
  - [ ] ABAP-Doc on all three methods with @parameter and @raising
  - [ ] Manager `check_job_status()` declares RAISING because `load_process()` can raise

## Dev Notes

### Target Repository
`cz.imcg.fast.planner`

### Constitution Compliance
- **Principle II (SAP Standards):** ABAP-Doc on all new public methods. Consistent with existing delegation pattern (cancel_process, supersede_process).
- **Principle III (Consult SAP Docs):** `CL_APJ_RT_API=>GET_JOB_STATUS()` parameters: iv_jobname, iv_jobcount. Exports: ev_job_status (ty_job_status), ev_job_status_text (ty_job_status_text). Raises CX_APJ_RT.
- **Principle V (Error Handling):** Instance check_job_status() swallows all errors (diagnostic method). Manager methods propagate load_process() exceptions.

### Dependencies
- **Depends on:** Story EST-121.3 (request methods on instance), Story EST-121.4 (cancel/restart extensions)
- **Blocks:** Story EST-121.6 (unit tests may call manager methods)

### Technical Decisions
- **check_job_status() is report-only:** Returns human-readable string. Does NOT auto-transition instance status. Caller decides what action to take.
- **Manager methods are one-liner delegations:** Identical pattern to `cancel_process()` and `supersede_process()`.
- **Manager check_job_status() declares RAISING:** Because `load_process()` can raise if instance not found. The instance-level method does NOT raise.

### Files to Modify
| File | Action |
|------|--------|
| `src/zcl_fi_process_instance.clas.abap` | Modify — add check_job_status() |
| `src/zcl_fi_process_manager.clas.abap` | Modify — add 3 delegation methods |

### Codebase Patterns
- Existing delegation: `cancel_process()` → `load_process( iv_instance_id )->cancel( )`
- Existing delegation: `supersede_process()` → `load_process( iv_instance_id )->supersede( )`
- Instance get_status(): `lo_instance->get_status( )` returns TYPE zfi_process_status

### Critical Warnings
1. **Return type for check_job_status():** TYPE string (not a DDIC type). This is acceptable for a diagnostic method returning dynamic content.
2. **CL_APJ_RT_API=>TY_JOB_STATUS_TEXT type:** Verify this type exists and its length. If not available, use `string` for the local variable.

### References
- [Source: tech-spec-est-121-apj-background-execution-lifecycle.md#Tasks 13-14]
- [Source: tech-spec-est-121-apj-background-execution-lifecycle.md#Technical Decision 8]

## Dev Agent Record

### Agent Model Used
### Debug Log References
### Completion Notes List
### File List
