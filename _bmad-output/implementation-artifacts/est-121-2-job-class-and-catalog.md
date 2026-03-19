# Story EST-121.2: APJ Job Class and Catalog

Status: ready-for-dev

## Story

As a framework developer,
I want a generic APJ job class (`ZCL_FI_PROCESS_JOB`) with corresponding job catalog entry and template,
so that any process instance can be executed or restarted as a background SAP Application Job.

## Acceptance Criteria

1. Given `ZCL_FI_PROCESS_JOB` exists, when `GET_PARAMETERS` is called, then it returns two mandatory parameters: INSTGUID (CHAR 32) and ACTION (CHAR 1).
2. Given a job runs with ACTION='E' and the instance is in EXEC_REQUESTED status, when `EXECUTE` runs, then it delegates to `lo_instance->execute()`.
3. Given a job runs with ACTION='R' and the instance is in RESTART_REQUESTED status, when `EXECUTE` runs, then it delegates to `lo_instance->restart()`.
4. Given a job runs but the instance status has changed (e.g. CANCELLED), when `EXECUTE` runs, then it returns silently without executing — no exception raised.
5. Given `ZCL_FI_PROCESS_JOB` raises `zcx_fi_process_error` during execution, when caught, then the job completes cleanly (RETURN, no unhandled exception).
6. APJ job catalog entry `ZFI_PROCESS_JOB_CAT` and template `ZFI_PROCESS_JOB_TMPL` exist in the system.

## Tasks / Subtasks

- [ ] Task 5: Create `ZCL_FI_PROCESS_JOB` class (AC: #1, #2, #3, #4, #5)
  - [ ] Create `src/zcl_fi_process_job.clas.abap` — PUBLIC FINAL CREATE PUBLIC
  - [ ] Implement `IF_APJ_DT_EXEC_OBJECT` and `IF_APJ_RT_EXEC_OBJECT` interfaces
  - [ ] Define constants: `gc_action_execute TYPE c LENGTH 1 VALUE 'E'`, `gc_action_restart TYPE c LENGTH 1 VALUE 'R'`
  - [ ] Implement `GET_PARAMETERS`: return INSTGUID (parameter, CHAR 32, mandatory) and ACTION (parameter, CHAR 1, mandatory)
  - [ ] Implement `EXECUTE`:
    - Extract INSTGUID and ACTION from `it_parameters` (flat structure: `selname`, `low`)
    - If INSTGUID is initial → RETURN silently
    - Load instance via `zcl_fi_process_manager=>get_instance( )` then `load_process()`
    - Status guard: for ACTION='E', verify status = `exec_requested`; for ACTION='R', verify status = `restart_requested`
    - If status mismatch → RETURN silently (do NOT raise exception)
    - Delegate to `lo_instance->execute()` or `lo_instance->restart()`
    - CATCH `zcx_fi_process_error` → RETURN (instance already set to FAILED internally)
  - [ ] Create `src/zcl_fi_process_job.clas.xml` metadata file
  - [ ] Add ABAP-Doc on class header and all public members

- [ ] Task 6: Create APJ job catalog entry and template (AC: #6)
  - [ ] Create catalog entry `ZFI_PROCESS_JOB_CAT` in ADT: description "ZFI Process Instance Background Execution", class `ZCL_FI_PROCESS_JOB`
  - [ ] Create template `ZFI_PROCESS_JOB_TMPL` in ADT: description "ZFI Process Instance Background Job", based on catalog `ZFI_PROCESS_JOB_CAT`, default ACTION='E'
  - [ ] After activation, `abapGit pull` to generate `src/zfi_process_job_cat.sajc.xml` and `src/zfi_process_job_tmpl.sajt.xml`
  - [ ] Verify template name matches the constant `gc_job_template` used in Story EST-121.3

## Dev Notes

### Target Repository
`cz.imcg.fast.planner`

### Constitution Compliance
- **Principle II (SAP Standards):** ABAP-Doc on all public methods. SAP naming (ZCL_FI_PROCESS prefix). Line length ≤120 chars.
- **Principle III (Consult SAP Docs):** IF_APJ_DT_EXEC_OBJECT and IF_APJ_RT_EXEC_OBJECT verified as released APIs (Clean Core Level A). IF_APJ_RT_RUN is NOT available on S/4HANA 7.58.
- **Principle IV (Factory Pattern) — EXCEPTION:** Public constructor required by APJ framework (instantiates via CREATE OBJECT). Same pattern as step implementation classes.
- **Principle V (Error Handling):** All errors caught internally — job class never propagates exceptions to APJ framework.

### Dependencies
- **Depends on:** Story EST-121.1 (domain values for status constants, exception text IDs)
- **Blocks:** Story EST-121.3 (request methods reference job template name)

### Technical Decisions
- **One generic job class for all process types:** Process type is resolved from the instance record. No per-process-type job class proliferation.
- **IF_APJ_DT_EXEC_OBJECT + IF_APJ_RT_EXEC_OBJECT (legacy interfaces):** IF_APJ_RT_RUN not available on target system. Legacy interfaces provide equivalent functionality.
- **INSTGUID + ACTION two-parameter design:** ACTION distinguishes execute ('E') vs restart ('R') paths.
- **Silent return on all guard failures:** APJ framework treats unhandled exceptions as job failures with cryptic error messages. A silent return produces a clean "completed" job.
- **APJ parameter structure difference:** When scheduling (`CL_APJ_RT_API=>SCHEDULE_JOB()`), parameters use `ty_job_parameter_value` with field `name` + nested `t_value`. When receiving in `EXECUTE`, the `it_parameters` table is flat — `selname` (not `name`) + `sign/option/low/high` directly on the row.

### Files to Create/Modify
| File | Action |
|------|--------|
| `src/zcl_fi_process_job.clas.abap` | Create — new APJ job class |
| `src/zcl_fi_process_job.clas.xml` | Create — class metadata |
| `src/zfi_process_job_cat.sajc.xml` | Create — APJ catalog entry (via ADT + abapGit pull) |
| `src/zfi_process_job_tmpl.sajt.xml` | Create — APJ template (via ADT + abapGit pull) |

### Codebase Patterns
- Manager singleton: `zcl_fi_process_manager=>get_instance( )` returns singleton
- Load instance: `lo_manager->load_process( lv_instance_id )` returns `zcl_fi_process_instance`
- Instance status: `lo_instance->get_status( )` returns `zfi_process_status`
- Status constants: `zcl_fi_process_instance=>gc_status-exec_requested` (will exist after Story EST-121.3 Task 7, but the class references them)

### Critical Warnings
1. **Compile order:** This class references `gc_status-exec_requested` and `gc_status-restart_requested` which are added in Story EST-121.3 (Task 7). The job class and the status constants may need to be activated together, or the job class can use string literals temporarily and be updated in Story EST-121.3.
2. **SELNAME vs NAME:** The parameter names in `GET_PARAMETERS` use `selname`, but in `SCHEDULE_JOB()` they use `name`. The string values must match ('INSTGUID', 'ACTION') — only the field name differs.
3. **Catalog/template are ADT-only:** Cannot be hand-crafted in XML — must be created in ADT, then pulled via abapGit.

### References
- [Source: tech-spec-apj-background-execution-lifecycle.md#Task 5, Task 6]
- [Source: tech-spec-apj-background-execution-lifecycle.md#Technical Decisions 1-3, 12]
- [Source: constitution.md#Principle III, IV]

## Dev Agent Record

### Agent Model Used
### Debug Log References
### Completion Notes List
### File List
