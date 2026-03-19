# Story EST-121.4: Lifecycle Extensions (restart/cancel/execute)

Status: ready-for-dev

## Story

As a framework developer,
I want `restart()` to accept CANCELLED and RESTART_REQUESTED statuses, `cancel()` to handle *_REQUESTED statuses with APJ job cancellation, and `execute()` to allow EXEC_REQUESTED as an entry point,
so that the APJ job class can properly delegate to existing methods and users can cancel/restart background-scheduled instances.

## Acceptance Criteria

1. Given a CANCELLED instance, when `restart()` is called (foreground), then the instance transitions to RUNNING and execution resumes. (Tech spec AC8)
2. Given a RESTART_REQUESTED instance, when `restart()` is called (by job class), then it is allowed (not rejected by guard).
3. Given an EXEC_REQUESTED instance, when `cancel()` is called, then status transitions to CANCELLED, `CL_APJ_RT_API=>CANCEL_JOB()` is called best-effort, and job reference fields are cleared. (Tech spec AC6)
4. Given a RESTART_REQUESTED instance, when `cancel()` is called, then same behavior as AC #3.
5. Given a RUNNING instance with a job reference, when `cancel()` is called, then `CL_APJ_RT_API=>CANCEL_JOB()` is called best-effort. (Tech spec AC7)
6. Given `CL_APJ_RT_API=>CANCEL_JOB()` fails (throws CX_APJ_RT), when `cancel()` runs, then the cancel still succeeds — fire-and-forget semantics.
7. Given an EXEC_REQUESTED instance, when the APJ job calls `execute()`, then the outer IF guard allows it through (not rejected). (Tech spec AC4)
8. Given a RESTART_REQUESTED instance, when `execute()` is called directly (not `restart()`), then it is rejected with `invalid_status`.

## Tasks / Subtasks

- [ ] Task 10: Extend `restart()` to allow CANCELLED and RESTART_REQUESTED statuses (AC: #1, #2)
  - [ ] Open `src/zcl_fi_process_instance.clas.abap`
  - [ ] Find the `restart()` status guard (line ~1816): `IF ms_instance-status <> gc_status-failed.`
  - [ ] Replace with: `IF NOT ( ms_instance-status = gc_status-failed OR ms_instance-status = gc_status-cancelled OR ms_instance-status = gc_status-restart_requested ).`
  - [ ] Update the error message value string to list all three allowed statuses
  - [ ] **Known limitation:** A CANCELLED instance with no FAILED step (cancelled mid-execution) will raise `no_failed_step` on restart — this is acceptable and documented as a future enhancement

- [ ] Task 11: Extend `cancel()` to handle *_REQUESTED statuses, call cancel_job, clear job fields (AC: #3, #4, #5, #6)
  - [ ] Replace the `cancel()` method body
  - [ ] Extend allowlist: RUNNING, FAILED, EXEC_REQUESTED, RESTART_REQUESTED
  - [ ] Add APJ job cancellation block:
    - If `ms_instance-job_name IS NOT INITIAL` → TRY `cl_apj_rt_api=>cancel_job()` CATCH `cx_apj_rt` (fire-and-forget)
  - [ ] Set status = cancelled
  - [ ] CLEAR: ms_instance-job_name, ms_instance-job_count
  - [ ] GET TIME STAMP + save_instance() — unchanged from existing

- [ ] Task 12: Extend `execute()` outer IF guard to allow EXEC_REQUESTED (AC: #7, #8)
  - [ ] Find the outer IF guard (lines 630-635): the compound condition rejecting non-NEW/non-FAILED-with-step
  - [ ] Add `AND ms_instance-status <> gc_status-exec_requested` to the rejection condition
  - [ ] Add a WHEN branch in the CASE block for RESTART_REQUESTED → reject with `invalid_status` (cannot use execute() for restart path)

## Dev Notes

### Target Repository
`cz.imcg.fast.planner`

### Constitution Compliance
- **Principle II (SAP Standards):** Line length ≤120 chars. Existing ABAP-Doc preserved.
- **Principle III (Consult SAP Docs):** `CL_APJ_RT_API=>CANCEL_JOB()` parameters: `iv_jobname`, `iv_jobcount`. Raises `CX_APJ_RT` on failure.
- **Principle V (Error Handling):** Cancel fire-and-forget — never propagates APJ infrastructure failures. Status guards raise `invalid_status` with descriptive context.

### Dependencies
- **Depends on:** Story EST-121.3 (gc_status constants must exist for the new WHEN branches)
- **Blocks:** Story EST-121.5 (manager delegation needs cancel/restart to work correctly)

### Technical Decisions
- **Cancel fire-and-forget rationale:** Caller must never be blocked by APJ infrastructure failure. The job-side status guard in ZCL_FI_PROCESS_JOB handles orphaned jobs.
- **CANCELLED is recoverable:** restart() now allows CANCELLED (not just FAILED). This supports user-cancelled instances being restarted.
- **execute() outer IF is the critical fix:** Without allowing `exec_requested` through the outer IF, the entire APJ background execution path is broken.
- **RESTART_REQUESTED explicitly rejected in execute():** You must use restart(), not execute(), for restart path.

### Files to Modify
| File | Action |
|------|--------|
| `src/zcl_fi_process_instance.clas.abap` | Modify — extend restart(), cancel(), execute() |

### Codebase Patterns
- restart() guard (line ~1816): `IF ms_instance-status <> gc_status-failed.` → compound IF
- cancel() current implementation: allowlist guard (RUNNING or FAILED), then status=cancelled, GET TIME STAMP, save_instance()
- execute() outer IF (lines 630-635): `IF ms_instance-status <> gc_status-new AND NOT ( ... ).` → add exec_requested exception
- execute() CASE block: explicit WHEN branches for each non-NEW status — add WHEN for restart_requested

### Critical Warnings
1. **cancel() must preserve existing behavior:** All current cancel use cases (RUNNING, FAILED) must still work identically. Only add new statuses to allowlist and add job cancel logic.
2. **cancel() clears job_name/job_count:** This is the ONLY action that clears these fields. Normal completion/failure preserves them for post-mortem.
3. **restart() known limitation:** CANCELLED instance with no FAILED step will fail with `no_failed_step`. Document as known limitation — future enhancement for `re_execute()`.
4. **execute() outer IF logic:** The condition is a REJECTION guard (returns early if true). Adding `AND status <> exec_requested` means exec_requested will NOT be rejected, allowing it to pass through to the execution logic.

### References
- [Source: tech-spec-apj-background-execution-lifecycle.md#Tasks 10-12]
- [Source: tech-spec-apj-background-execution-lifecycle.md#Technical Decisions 7, cancel fire-and-forget]
- [Source: tech-spec-apj-background-execution-lifecycle.md#Revised State Machine]

## Dev Agent Record

### Agent Model Used
### Debug Log References
### Completion Notes List
### File List
