---
title: 'APJ Background Execution & Lifecycle Enhancement'
slug: 'apj-background-execution-lifecycle'
created: '2026-03-18'
status: 'reviewed'
stepsCompleted: []
tech_stack: ['ABAP 7.58', 'SAP DDIC', 'ZFI_PROCESS framework', 'SAP Application Jobs (APJ)', 'CL_APJ_RT_API']
files_to_modify:
  - 'src/zfi_process_status.doma.xml'
  - 'src/zfi_proc_inst.tabl.xml'
  - 'src/zcx_fi_process_error.clas.abap'
  - 'src/zfi_process.msag.xml'
  - 'src/zcl_fi_process_instance.clas.abap'
  - 'src/zcl_fi_process_manager.clas.abap'
  - 'src/zcl_fi_process_manager.clas.testclasses.abap'
files_to_create:
  - 'src/zcl_fi_process_job.clas.abap'
  - 'src/zcl_fi_process_job.clas.xml'
  - 'src/zfi_process_job_cat.sajc.xml'
  - 'src/zfi_process_job_tmpl.sajt.xml'
  - 'src/zfi_process_jobname.dtel.xml'
  - 'src/zfi_process_jobcount.dtel.xml'
code_patterns:
  - 'gc_status constants: CONSTANTS BEGIN OF gc_status ... END OF gc_status — append new statuses before END OF'
  - 'IF_APJ_DT_EXEC_OBJECT~GET_PARAMETERS: define INST_GUID parameter (CHAR 32) for instance routing'
  - 'IF_APJ_RT_EXEC_OBJECT~EXECUTE: read INST_GUID from IT_PARAMETERS, load instance, status guard, delegate to instance.execute() or instance.restart()'
  - 'schedule-first-status-second: CL_APJ_RT_API=>SCHEDULE_JOB() before any status UPDATE; if scheduling fails, status unchanged'
  - 'cancel() fire-and-forget: set CANCELLED + call CL_APJ_RT_API=>CANCEL_JOB() best-effort in TRY/CATCH'
  - 'Job-side status guard: if status <> expected, RETURN silently (do not raise exception)'
test_patterns:
  - 'ABAP Unit tests: FOR TESTING DURATION SHORT RISK LEVEL HARMLESS'
  - 'Given/When/Then pattern with cl_abap_unit_assert'
---

# Tech-Spec: APJ Background Execution & Lifecycle Enhancement

**Created:** 2026-03-18
**Target Repository:** `cz.imcg.fast.planner`
**Brainstorming Reference:** `_bmad-output/brainstorming/brainstorming-session-2026-03-17-000000.md`

## Overview

### Problem Statement

The ZFI_PROCESS framework currently supports only foreground (synchronous) execution of process instances. Long-running allocations block the caller's session and cannot be offloaded to SAP background infrastructure. Additionally, there is no clean mechanism to:

1. **Schedule background execution** — run an entire process instance as an SAP Application Job (APJ)
2. **Schedule background restart** — restart a FAILED or CANCELLED instance in the background
3. **Track job references** — know which APJ job is running a given instance
4. **Reconcile status** — detect if the APJ job completed/failed but the instance status wasn't updated (e.g. system crash)
5. **Cancel background instances** — cancel both the instance status and the underlying APJ job

The existing `cancel()` method also needs extension to support cancelling instances that are scheduled-but-not-yet-started (new *_REQUESTED statuses).

### Solution

Introduce APJ background execution via a 3-tier approach:

**Tier 1 — Foundation:**
- Add two new DDIC domain values: `EXEC_REQUESTED` and `RESTART_REQUESTED`
- Add two new fields to `ZFI_PROC_INST`: `JOB_NAME` and `JOB_COUNT` (APJ job reference)
- Create a generic job class `ZCL_FI_PROCESS_JOB` implementing `IF_APJ_DT_EXEC_OBJECT` + `IF_APJ_RT_EXEC_OBJECT`
- Create corresponding APJ job catalog entry and job template

**Tier 2 — Core Actions:**
- Add `request_execute()` and `request_restart()` methods to `ZCL_FI_PROCESS_INSTANCE`
- Expose `request_execute_process()` and `request_restart_process()` on `ZCL_FI_PROCESS_MANAGER`
- Extend `cancel()` to handle *_REQUESTED statuses and call `CL_APJ_RT_API=>CANCEL_JOB()` best-effort
- Extend `restart()` to allow restart from CANCELLED (not just FAILED)

**Tier 3 — Safety Layer:**
- Add `check_job_status()` method for on-demand reconciliation
- Extend EST-102 duplicate check and EST-106 parallel limit to count *_REQUESTED as active

### Revised State Machine

```
NEW ──execute()──────────────> RUNNING
NEW ──request_execute()──────> EXEC_REQUESTED ──(job runs)──> RUNNING
RUNNING ──(success)──────────> COMPLETED
RUNNING ──(failure)──────────> FAILED
RUNNING ──cancel()───────────> CANCELLED
EXEC_REQUESTED ──cancel()────> CANCELLED
RESTART_REQUESTED ──cancel()─> CANCELLED
FAILED ──restart()───────────> RUNNING
FAILED ──request_restart()───> RESTART_REQUESTED ──(job runs)──> RUNNING
FAILED ──cancel()────────────> CANCELLED
CANCELLED ──restart()────────> RUNNING
CANCELLED ──request_restart()> RESTART_REQUESTED ──(job runs)──> RUNNING
COMPLETED ──supersede()──────> SUPERSEDED
```

### State Transition Matrix

| Current Status | execute | request_execute | restart | request_restart | cancel | supersede | check_job_status |
|---|---|---|---|---|---|---|---|
| **NEW** | → RUNNING | → EXEC_REQ | - | - | - | - | - |
| **EXEC_REQUESTED** | - | - | - | - | → CANCELLED | - | reconcile |
| **RUNNING** | - | - | - | - | → CANCELLED | - | reconcile |
| **COMPLETED** | - | - | - | - | - | → SUPERSEDED | - |
| **FAILED** | - | - | → RUNNING | → RESTART_REQ | → CANCELLED | - | - |
| **CANCELLED** | - | - | → RUNNING | → RESTART_REQ | - | - | - |
| **RESTART_REQUESTED** | - | - | - | - | → CANCELLED | - | reconcile |
| **SUPERSEDED** | - | - | - | - | - | - | - |

### Scope

**In Scope:**
- Two new DDIC domain fixed values: `EXEC_REQUESTED` (VALPOS 0009), `RESTART_REQUESTED` (VALPOS 0010)
- Two new DDIC data elements: `ZFI_PROCESS_JOBNAME`, `ZFI_PROCESS_JOBCOUNT`
- Two new fields on `ZFI_PROC_INST`: `JOB_NAME`, `JOB_COUNT`
- New class `ZCL_FI_PROCESS_JOB` implementing `IF_APJ_DT_EXEC_OBJECT` + `IF_APJ_RT_EXEC_OBJECT`
- APJ job catalog entry `ZFI_PROCESS_JOB_CAT` and template `ZFI_PROCESS_JOB_TMPL`
- New methods on `ZCL_FI_PROCESS_INSTANCE`: `request_execute()`, `request_restart()`, `check_job_status()`
- New methods on `ZCL_FI_PROCESS_MANAGER`: `request_execute_process()`, `request_restart_process()`, `check_job_status()`
- `gc_status` constants extended: `exec_requested`, `restart_requested` (both classes)
- `cancel()` extended: allowlist includes *_REQUESTED statuses; calls `CL_APJ_RT_API=>CANCEL_JOB()` best-effort; clears job reference fields
- `restart()` extended: also allows restart from CANCELLED status
- EST-102 duplicate check: `EXEC_REQUESTED` and `RESTART_REQUESTED` count as blocking
- EST-106 parallel limit: `EXEC_REQUESTED` and `RESTART_REQUESTED` count as active
- New exception text IDs and T100 messages for scheduling errors
- Unit tests for new methods

**Out of Scope:**
- Cooperative cancellation (between-step status check during long-running execution) — future enhancement
- RAP BO action wrappers (EST-111–114 abandoned, redesign separate)
- UI/Fiori changes
- Periodic/recurring job scheduling (only immediate single-run jobs)
- `check_job_status()` auto-transition (reports only, does not change status)
- Translation of new messages to Czech (deferred per Sprint policy)

---

## Context for Development

### Technical Preferences & Constraints

- **Constitution Principle I (DDIC-First):** New status values, data elements, and fields must be defined in DDIC — no local TYPE definitions
- **Constitution Principle II (SAP Standards):** Line length ≤ 120 chars; SAP naming conventions; ABAP-Doc on all public methods
- **Constitution Principle III (Consult SAP Docs):** CL_APJ_RT_API, IF_APJ_DT_EXEC_OBJECT, IF_APJ_RT_EXEC_OBJECT are verified as released APIs on S/4HANA on-premise (SAP_BASIS component BC-SRV-APS-APJ, Clean Core Level A)
- **Constitution Principle IV (Factory Pattern):** `ZCL_FI_PROCESS_JOB` has a public constructor (required by APJ framework to instantiate the class). This is an acceptable exception — same pattern as step implementation classes which also use public constructors.
- **Constitution Principle V (Error Handling):** All scheduling errors raise `zcx_fi_process_error` with proper context. APJ exceptions (`CX_APJ_RT`) are caught and wrapped.
- **Schedule-first, status-second:** Always call `CL_APJ_RT_API=>SCHEDULE_JOB()` BEFORE updating instance status. If scheduling fails, status remains unchanged — no broken intermediate states.
  > **LUW detail:** `save_instance()` does `MODIFY` + `COMMIT WORK AND WAIT`. The `schedule_job()` call registers the job in the APJ queue within the same LUW, and both the job registration and the status update are committed together by `save_instance()`. If the COMMIT fails (e.g. DB lock), the APJ job may already be registered but the instance status remains unchanged. The job-side status guard handles this — the job would see the old status (NEW, not EXEC_REQUESTED) and return silently. This is an acceptable edge case.
- **Cancel is fire-and-forget:** `cancel()` always succeeds from the caller's perspective. `CL_APJ_RT_API=>CANCEL_JOB()` is called best-effort in a TRY/CATCH — if it fails, the status is still set to CANCELLED.
- **Job-side status guard:** The APJ job class checks instance status on entry. If status doesn't match the expected value (EXEC_REQUESTED or RESTART_REQUESTED), the job returns silently without executing.
- **`start_immediately` is prohibited:** `CL_APJ_RT_API=>SCHEDULE_JOB()` with `start_immediately = 'X'` performs an implicit COMMIT WORK and cannot be called from within a RAP BO. Always use timestamp-based start (NOW + a small offset).
- **IF_APJ_RT_RUN is not available:** On-premise S/4HANA 7.58 does not have `IF_APJ_RT_RUN`. Use the legacy `IF_APJ_DT_EXEC_OBJECT` + `IF_APJ_RT_EXEC_OBJECT` interfaces.

### Codebase Patterns

- **`gc_status` constants (instance):** `src/zcl_fi_process_instance.clas.abap` lines 26–36 — `CONSTANTS: BEGIN OF gc_status, ... END OF gc_status.` in PUBLIC SECTION. Currently 9 values: new, pending, queued, running, completed, failed, cancelled, superseded, skipped. Add `exec_requested` and `restart_requested` before `skipped`.
- **`gc_status` constants (manager):** `src/zcl_fi_process_manager.clas.abap` lines 178–187 — PRIVATE SECTION. Currently 8 values (no `queued`). Add `exec_requested` and `restart_requested` before `END OF gc_status`.
- **`cancel()` current implementation (instance):** Allowlist guard (`RUNNING` or `FAILED`), then `status = cancelled`, `GET TIME STAMP`, `save_instance()`. Extend allowlist to include *_REQUESTED statuses; add `CL_APJ_RT_API=>CANCEL_JOB()` call and clear job fields.
- **`restart()` current guard (instance, line 1816):** `IF ms_instance-status <> gc_status-failed. RAISE EXCEPTION...`. Change to allow both FAILED and CANCELLED.
- **`execute()` status guard (instance, lines 632–635):** CASE block with explicit WHEN branches for each non-NEW status. Add WHEN branches for `exec_requested` and `restart_requested`.
- **`create_process()` duplicate check (manager, lines 286–293):** Checks `line_exists()` for RUNNING, FAILED, COMPLETED. Add `exec_requested` and `restart_requested`.
- **Parallel instance limit (EST-106):** `get_active_instance_count()` or similar — counts RUNNING instances. Must also count `exec_requested` and `restart_requested`.
- **ZFI_PROC_INST table:** 30 fields. `JOB_NAME` and `JOB_COUNT` will be appended after `BUSINESS_STATUS_2`.
- **Exception text ID pattern:** `CONSTANTS: BEGIN OF xxx, msgid TYPE symsgid VALUE 'ZFI_PROCESS', msgno TYPE symsgno VALUE 'NNN', attr1..attr4, END OF xxx.`
- **CL_APJ_RT_API key types:**
  ```
  cl_apj_rt_api=>ty_template_name  — job template name
  cl_apj_rt_api=>ty_job_text       — job description text
  cl_apj_rt_api=>ty_start_info     — start configuration (timestamp or immediate)
  cl_apj_rt_api=>ty_jobname        — returned job name
  cl_apj_rt_api=>ty_jobcount       — returned job count
  cl_apj_rt_api=>ty_job_status     — job status for get_job_status()
  cl_apj_rt_api=>tt_job_parameter_value — parameter table
  cl_apj_rt_api=>ty_job_parameter_value — single parameter entry
  cl_apj_rt_api=>ty_value_range    — parameter value range entry
  ```
- **IF_APJ_DT_EXEC_OBJECT~GET_PARAMETERS pattern:**
  ```abap
  et_parameter_def = VALUE #(
    ( selname   = 'INSTGUID'
      kind      = if_apj_dt_exec_object=>parameter
      datatype  = 'C'
      length    = 32
      param_text = 'Process Instance ID'
      changeable_ind = abap_true
      mandatory_ind  = abap_true ) ).
  ```
- **IF_APJ_RT_EXEC_OBJECT~EXECUTE pattern:**
  ```abap
  METHOD if_apj_rt_exec_object~execute.
    " Read instance GUID from job parameters
    " Load instance, check status guard, delegate to execute/restart
  ENDMETHOD.
  ```

### Files to Reference

| File | Purpose |
|------|---------|
| `src/zfi_process_status.doma.xml` | Add EXEC_REQUESTED (0009) and RESTART_REQUESTED (0010) |
| `src/zfi_proc_inst.tabl.xml` | Add JOB_NAME and JOB_COUNT fields |
| `src/zfi_process_jobname.dtel.xml` | New data element for JOB_NAME |
| `src/zfi_process_jobcount.dtel.xml` | New data element for JOB_COUNT |
| `src/zcl_fi_process_job.clas.abap` | New APJ job class |
| `src/zfi_process_job_cat.sajc.xml` | APJ job catalog entry |
| `src/zfi_process_job_tmpl.sajt.xml` | APJ job template |
| `src/zcx_fi_process_error.clas.abap` | New exception text IDs |
| `src/zfi_process.msag.xml` | New T100 messages |
| `src/zcl_fi_process_instance.clas.abap` | New methods + cancel/restart extensions |
| `src/zcl_fi_process_manager.clas.abap` | New manager methods + guard extensions |
| `src/zcl_fi_process_manager.clas.testclasses.abap` | Unit tests |

### Technical Decisions

1. **One generic job class for all process types:** `ZCL_FI_PROCESS_JOB` receives the instance GUID as a parameter and delegates to the existing `execute()` / `restart()` methods. No per-process-type job class proliferation. The process type is resolved from the instance record.

2. **IF_APJ_DT_EXEC_OBJECT + IF_APJ_RT_EXEC_OBJECT (legacy interfaces):** `IF_APJ_RT_RUN` is not available on the target S/4HANA 7.58 system. The legacy interfaces provide equivalent functionality — `GET_PARAMETERS` defines a single `INSTGUID` parameter (CHAR 32), and `EXECUTE` receives the parameter table, extracts the GUID, and delegates.

3. **Single INSTGUID parameter plus ACTION parameter:** The job class needs to know whether to call `execute()` or `restart()`. A second parameter `ACTION` (CHAR 1, values 'E' for execute, 'R' for restart) distinguishes the two paths. Both parameters are mandatory.

4. **Timestamp-based immediate start:** `schedule_job()` uses `is_start_info-timestamp = NOW + 2 seconds` instead of `start_immediately = 'X'`. This avoids the implicit COMMIT WORK issue and is safe to call from RAP context. The 2-second offset provides margin for the LUW to commit before the job starts.

5. **Job reference fields persist after completion:** `JOB_NAME` and `JOB_COUNT` are NOT cleared when a job completes (success or failure). This allows post-mortem investigation via `check_job_status()` or APJ job logs. Fields are only cleared explicitly by `cancel()`.

6. **Job references overwritten on new request:** When a new background request is made (e.g. `request_restart()` after cancel), `JOB_NAME`/`JOB_COUNT` are overwritten with the new job reference. The old job (if still running) hits the status guard and aborts silently.

7. **CANCELLED is recoverable:** `restart()` is extended to allow restart from CANCELLED (not just FAILED). A user-cancelled instance can be restarted. `request_restart()` follows the same allowlist.

8. **check_job_status() is report-only:** It reads APJ job status via `CL_APJ_RT_API=>GET_JOB_STATUS()` and returns the result to the caller. It does NOT auto-transition the instance status. The caller (or user) decides what to do.

9. **Guard extensions are status-based:** For EST-102 duplicate check and EST-106 parallel limit, `EXEC_REQUESTED` and `RESTART_REQUESTED` count as "active" — they block duplicate creation and count toward the parallel limit. This prevents over-scheduling.

10. **Job catalog entry and template via abapGit XML:** The `.sajc.xml` and `.sajt.xml` files are generated by abapGit from the APJ catalog entry and template objects created in ADT. The spec provides the parameters; actual XML is generated by the system.

11. **Optimistic locking on status transitions:** The `request_execute()` and `request_restart()` methods must verify the expected status is still current after `schedule_job()` succeeds, immediately before saving. Use `SELECT SINGLE status FROM zfi_proc_inst WHERE instance_id = @ms_instance-instance_id INTO @DATA(lv_current_status)` and verify it matches the expected pre-condition. If it has changed (another caller modified it concurrently), cancel the scheduled job best-effort and raise an exception. This prevents the race condition where foreground `execute()` and background `request_execute()` run concurrently on the same NEW instance.

12. **APJ parameter name mapping (two different structures):** The APJ framework uses two different parameter structures with different field names. When **scheduling** via `CL_APJ_RT_API=>SCHEDULE_JOB()`, parameters use `ty_job_parameter_value` with field `name` and a nested `t_value` sub-table (each entry: sign/option/low/high). When **receiving** in `IF_APJ_RT_EXEC_OBJECT~EXECUTE`, the `it_parameters` table is flat — each row has `selname` (not `name`), plus `sign`, `option`, `low`, `high` directly on the row. The parameter names must match between scheduling (`name = 'INSTGUID'`) and reading (`selname = 'INSTGUID'`), but the field structure is completely different.

---

## Implementation Plan

### Tasks

Tasks are ordered by dependency (DDIC first, then exception infrastructure, then job class, then instance methods, then manager methods, then guard extensions, then tests).

#### Tier 1 — Foundation

- [ ] **Task 1: Add EXEC_REQUESTED and RESTART_REQUESTED fixed values to `ZFI_PROCESS_STATUS` domain**
  - File: `src/zfi_process_status.doma.xml` *(modify)*
  - Action: Append two new `<DD07V>` entries inside `<DD07V_TAB>` after the SUPERSEDED entry (VALPOS 0008):
    ```xml
    <DD07V>
     <VALPOS>0009</VALPOS>
     <DDLANGUAGE>E</DDLANGUAGE>
     <DOMVALUE_L>EXEC_REQUESTED</DOMVALUE_L>
     <DDTEXT>Execution Requested</DDTEXT>
    </DD07V>
    <DD07V>
     <VALPOS>0010</VALPOS>
     <DDLANGUAGE>E</DDLANGUAGE>
     <DOMVALUE_L>RESTART_REQUESTED</DOMVALUE_L>
     <DDTEXT>Restart Requested</DDTEXT>
    </DD07V>
    ```
  - Notes: VALPOS 0009 and 0010 are the next available slots after 0008 (SUPERSEDED, added by EST-110). Domain is CHAR(20) — both values fit within the 20-character limit (EXEC_REQUESTED = 14 chars, RESTART_REQUESTED = 17 chars).
    > **Pre-condition check:** Before inserting, verify the highest VALPOS in the file is 0008. If it differs, adjust accordingly.

- [ ] **Task 2: Create DDIC data elements for JOB_NAME and JOB_COUNT**
  - File A: `src/zfi_process_jobname.dtel.xml` *(create)*
  - Action: Create data element `ZFI_PROCESS_JOBNAME` based on built-in type CHAR(32). Description: "APJ Job Name".
    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <abapGit version="v1.0.0" serializer="LCL_OBJECT_DTEL" serializer_version="v1.0.0">
     <asx:abap xmlns:asx="http://www.sap.com/abapxml" version="1.0">
      <asx:values>
       <DD04V>
        <ROLLNAME>ZFI_PROCESS_JOBNAME</ROLLNAME>
        <DDLANGUAGE>E</DDLANGUAGE>
        <HEADLEN>20</HEADLEN>
        <SCRLEN1>10</SCRLEN1>
        <SCRLEN2>15</SCRLEN2>
        <SCRLEN3>20</SCRLEN3>
        <DDTEXT>APJ Job Name</DDTEXT>
        <REPTEXT>APJ Job Name</REPTEXT>
        <SCRTEXT_S>Job Name</SCRTEXT_S>
        <SCRTEXT_M>APJ Job Name</SCRTEXT_M>
        <SCRTEXT_L>APJ Job Name</SCRTEXT_L>
        <DTELMASTER>E</DTELMASTER>
        <DATATYPE>CHAR</DATATYPE>
        <LENG>000032</LENG>
       </DD04V>
      </asx:values>
     </asx:abap>
    </abapGit>
    ```
  - File B: `src/zfi_process_jobcount.dtel.xml` *(create)*
  - Action: Create data element `ZFI_PROCESS_JOBCOUNT` based on built-in type CHAR(8). Description: "APJ Job Count".
    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <abapGit version="v1.0.0" serializer="LCL_OBJECT_DTEL" serializer_version="v1.0.0">
     <asx:abap xmlns:asx="http://www.sap.com/abapxml" version="1.0">
      <asx:values>
       <DD04V>
        <ROLLNAME>ZFI_PROCESS_JOBCOUNT</ROLLNAME>
        <DDLANGUAGE>E</DDLANGUAGE>
        <HEADLEN>20</HEADLEN>
        <SCRLEN1>10</SCRLEN1>
        <SCRLEN2>15</SCRLEN2>
        <SCRLEN3>20</SCRLEN3>
        <DDTEXT>APJ Job Count</DDTEXT>
        <REPTEXT>APJ Job Count</REPTEXT>
        <SCRTEXT_S>Job Count</SCRTEXT_S>
        <SCRTEXT_M>APJ Job Count</SCRTEXT_M>
        <SCRTEXT_L>APJ Job Count</SCRTEXT_L>
        <DTELMASTER>E</DTELMASTER>
        <DATATYPE>CHAR</DATATYPE>
        <LENG>000008</LENG>
       </DD04V>
      </asx:values>
     </asx:abap>
    </abapGit>
    ```
  - Notes: Data types and lengths match `CL_APJ_RT_API=>TY_JOBNAME` (CHAR 32) and `CL_APJ_RT_API=>TY_JOBCOUNT` (CHAR 8). **IMPORTANT:** In some SAP systems, job count is `NUMC(8)` (e.g. `TBTCJOB-JOBCOUNT`). Verify the actual type of `CL_APJ_RT_API=>TY_JOBCOUNT` in the target system before creating the data element — if it's NUMC, use `DATATYPE=NUMC` instead of `DATATYPE=CHAR` to preserve leading-zero semantics.
    > **Constitution Principle I compliance:** Both fields use dedicated DDIC data elements, not raw built-in types.

- [ ] **Task 3: Add JOB_NAME and JOB_COUNT fields to ZFI_PROC_INST table**
  - File: `src/zfi_proc_inst.tabl.xml` *(modify)*
  - Action: Append two new `<DD03P>` field entries after `BUSINESS_STATUS_2`:
    ```xml
    <DD03P>
     <FIELDNAME>JOB_NAME</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_JOBNAME</ROLLNAME>
    </DD03P>
    <DD03P>
     <FIELDNAME>JOB_COUNT</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_JOBCOUNT</ROLLNAME>
    </DD03P>
    ```
  - Notes: Fields are NOT key fields, NOT required (initial value is blank). A blank JOB_NAME/JOB_COUNT means the instance was executed in foreground (no APJ job).
    > **Table activation:** After adding fields to the XML, activate the table in SE11 or via ADT. Existing records get initial (blank) values for the new fields — no data migration needed.
    > **Pre-condition check:** Verify the existing field order in `zfi_proc_inst.tabl.xml` ends with BUSINESS_STATUS_2. If additional fields have been added since the last review, insert after the actual last field.

- [ ] **Task 4: Add exception text IDs and T100 messages for scheduling errors**
  - File A: `src/zcx_fi_process_error.clas.abap` *(modify)*
  - Action: Insert new CONSTANTS block after `instance_superseded`:
    ```abap
    CONSTANTS:
      BEGIN OF scheduling_failed,
        msgid TYPE symsgid VALUE 'ZFI_PROCESS',
        msgno TYPE symsgno VALUE '019',
        attr1 TYPE scx_attrname VALUE 'VALUE',
        attr2 TYPE scx_attrname VALUE '',
        attr3 TYPE scx_attrname VALUE '',
        attr4 TYPE scx_attrname VALUE '',
      END OF scheduling_failed.
    ```
  - File B: `src/zfi_process.msag.xml` *(modify)*
  - Action: Insert new `<T100>` block after message 018:
    ```xml
    <T100>
     <SPRSL>E</SPRSL>
     <ARBGB>ZFI_PROCESS</ARBGB>
     <MSGNR>019</MSGNR>
     <TEXT>Background job scheduling failed: &amp;1</TEXT>
    </T100>
    ```
  - Notes: Message 019 is for `request_execute()`/`request_restart()` when `CL_APJ_RT_API=>SCHEDULE_JOB()` raises `CX_APJ_RT`. A `job_cancel_failed` exception is NOT defined — the `cancel()` method uses fire-and-forget semantics and never propagates job cancel failures.
    > **Pre-condition check:** Verify message number 019 is free in `zfi_process.msag.xml`. Also verify 020 is free — if so, use 019 for scheduling_failed and reserve 020 for future APJ-related messages.

#### Tier 2 — Core Actions

- [ ] **Task 5: Create `ZCL_FI_PROCESS_JOB` — the generic APJ job class**
  - File: `src/zcl_fi_process_job.clas.abap` *(create)*
  - Action: Create new class implementing `IF_APJ_DT_EXEC_OBJECT` and `IF_APJ_RT_EXEC_OBJECT`:

    ```abap
    "! Generic APJ background job class for ZFI_PROCESS framework
    "! <p>Implements IF_APJ_DT_EXEC_OBJECT and IF_APJ_RT_EXEC_OBJECT
    "! to execute or restart process instances as SAP Application Jobs.
    "! Receives instance GUID and action as job parameters.</p>
    CLASS zcl_fi_process_job DEFINITION
      PUBLIC
      FINAL
      CREATE PUBLIC.

      PUBLIC SECTION.
        INTERFACES if_apj_dt_exec_object.
        INTERFACES if_apj_rt_exec_object.

        "! Action constant: Execute
        CONSTANTS gc_action_execute TYPE c LENGTH 1 VALUE 'E'.
        "! Action constant: Restart
        CONSTANTS gc_action_restart TYPE c LENGTH 1 VALUE 'R'.

      PROTECTED SECTION.
      PRIVATE SECTION.
    ENDCLASS.

    CLASS zcl_fi_process_job IMPLEMENTATION.

      METHOD if_apj_dt_exec_object~get_parameters.
        et_parameter_def = VALUE #(
          ( selname        = 'INSTGUID'
            kind           = if_apj_dt_exec_object=>parameter
            datatype       = 'C'
            length         = 32
            param_text     = 'Process Instance ID'
            changeable_ind = abap_true
            mandatory_ind  = abap_true )
          ( selname        = 'ACTION'
            kind           = if_apj_dt_exec_object=>parameter
            datatype       = 'C'
            length         = 1
            param_text     = 'Action (E=Execute, R=Restart)'
            changeable_ind = abap_true
            mandatory_ind  = abap_true ) ).
      ENDMETHOD.

      METHOD if_apj_rt_exec_object~execute.
        " Extract parameters from job framework
        DATA lv_instance_id TYPE zfi_process_instance_id.
        DATA lv_action      TYPE c LENGTH 1.

        LOOP AT it_parameters INTO DATA(ls_param).
          CASE ls_param-selname.
            WHEN 'INSTGUID'. lv_instance_id = ls_param-low.
            WHEN 'ACTION'.   lv_action = ls_param-low.
          ENDCASE.
        ENDLOOP.

        IF lv_instance_id IS INITIAL.
          RETURN. "silent — missing parameter, cannot proceed
        ENDIF.

        TRY.
            DATA(lo_manager) =
              zcl_fi_process_manager=>get_instance( ).
            DATA(lo_instance) =
              lo_manager->load_process( lv_instance_id ).

            " Job-side status guard:
            " Only proceed if instance is in the expected status.
            " If status has changed (e.g. cancelled), abort silently.
            DATA(lv_status) = lo_instance->get_status( ).

            CASE lv_action.
              WHEN gc_action_execute.
                IF lv_status <>
                   zcl_fi_process_instance=>gc_status-exec_requested.
                  RETURN. "status changed — abort silently
                ENDIF.
                lo_instance->execute( ).

              WHEN gc_action_restart.
                IF lv_status <>
                   zcl_fi_process_instance=>gc_status-restart_requested.
                  RETURN. "status changed — abort silently
                ENDIF.
                lo_instance->restart( ).

              WHEN OTHERS.
                RETURN. "unknown action — abort silently
            ENDCASE.

          CATCH zcx_fi_process_error INTO DATA(lx_err).
            " Log the error to application log if possible.
            " The instance status will be FAILED (set by execute/restart
            " internally on step failure). No further action needed.
            " Job completes with error visible in APJ job log.
            RETURN.
        ENDTRY.
      ENDMETHOD.

    ENDCLASS.
    ```

  - Notes:
    - The class has a public constructor — this is required by the APJ framework which instantiates the class via `CREATE OBJECT`. Constitution Principle IV exception is documented.
    - The job-side status guard is the critical safety mechanism: if an instance was cancelled between scheduling and job start, the job detects this and returns silently.
    - The `CATCH zcx_fi_process_error` at the end handles any process execution failure. The instance status is already set to FAILED by the framework internally — the job class just ensures clean exit.
    - `GET_PARAMETERS` defines `INSTGUID` (SELNAME max 8 chars — fits) and `ACTION` (1 char).
    > **Silent return vs exception in job class:** The job class returns silently on all guard failures instead of raising exceptions. This is intentional — the APJ framework treats unhandled exceptions as job failures with cryptic error messages. A silent return produces a clean "completed" job that simply did nothing, which is the correct behavior when the instance was cancelled or already processed.

- [ ] **Task 6: Create APJ job catalog entry and job template**
  - Objects: `ZFI_PROCESS_JOB_CAT` (catalog entry), `ZFI_PROCESS_JOB_TMPL` (template)
  - Action: Create in ADT via *Other ABAP Repository Object → Application Jobs → Application Job Catalog Entry* and *Application Job Template*.
    - **Catalog entry:** Name `ZFI_PROCESS_JOB_CAT`, description "ZFI Process Instance Background Execution", class `ZCL_FI_PROCESS_JOB`. Parameters auto-populated from `GET_PARAMETERS`.
    - **Template:** Name `ZFI_PROCESS_JOB_TMPL`, description "ZFI Process Instance Background Job", based on catalog `ZFI_PROCESS_JOB_CAT`. Default values: `INSTGUID` = blank, `ACTION` = 'E'.
  - Notes: These must be created in the SAP system via ADT (not hand-crafted XML). After creation and activation, `abapGit pull` generates the `.sajc.xml` and `.sajt.xml` files automatically. The template name `ZFI_PROCESS_JOB_TMPL` is used by `request_execute()` and `request_restart()` when calling `CL_APJ_RT_API=>SCHEDULE_JOB()`.
    > **Verify template name in code:** The `request_execute()` method hardcodes the template name as a constant. If the actual template name differs, update the constant.

- [ ] **Task 7: Add `gc_status` constants for new statuses to both classes**
  - File A: `src/zcl_fi_process_instance.clas.abap` *(modify)*
  - Action: In PUBLIC SECTION `gc_status` block, add before `skipped`:
    ```abap
               exec_requested TYPE zfi_process_status
                 VALUE 'EXEC_REQUESTED',
               restart_requested TYPE zfi_process_status
                 VALUE 'RESTART_REQUESTED',
    ```
  - File B: `src/zcl_fi_process_manager.clas.abap` *(modify)*
  - Action: In PRIVATE SECTION `gc_status` block, add before `END OF gc_status`:
    ```abap
               exec_requested TYPE zfi_process_status
                 VALUE 'EXEC_REQUESTED',
               restart_requested TYPE zfi_process_status
                 VALUE 'RESTART_REQUESTED',
    ```
  - Notes: Line length constraint — `RESTART_REQUESTED` value string is 17 chars, constant name is 17 chars. The VALUE clause may need wrapping to stay under 120 chars per line.

- [ ] **Task 8: Add `request_execute()` method to `ZCL_FI_PROCESS_INSTANCE`**
  - File: `src/zcl_fi_process_instance.clas.abap` *(modify)*
  - Action A (PUBLIC SECTION — method declaration, after `execute`):
    ```abap
    "! Request background execution of this process instance
    "! <p>Schedules an APJ background job to execute this instance.
    "! Instance must be in NEW status. Status transitions to
    "! EXEC_REQUESTED on success.</p>
    "! @raising zcx_fi_process_error | If status is not NEW or
    "!   scheduling fails
    METHODS request_execute
      RAISING
        zcx_fi_process_error.
    ```
  - Action B (IMPLEMENTATION — new METHOD block):
    ```abap
    METHOD request_execute.
      " Status guard: only NEW instances can be scheduled
      IF ms_instance-status <> gc_status-new.
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Cannot request execute: instance |
                   && |{ ms_instance-instance_id } status is |
                   && |{ ms_instance-status }. Expected: NEW.|.
      ENDIF.

      " Schedule-first: call APJ API before changing status
      DATA lv_jobname  TYPE cl_apj_rt_api=>ty_jobname.
      DATA lv_jobcount TYPE cl_apj_rt_api=>ty_jobcount.

      DATA(ls_start_info) = VALUE cl_apj_rt_api=>ty_start_info( ).
      GET TIME STAMP FIELD DATA(lv_now).
      ls_start_info-timestamp = cl_abap_tstmp=>add(
        tstmp = lv_now
        secs  = 2 ).

      DATA(lt_params) = VALUE cl_apj_rt_api=>tt_job_parameter_value(
        ( name = 'INSTGUID'
          t_value = VALUE #(
            ( sign = 'I' option = 'EQ'
              low = CONV #( ms_instance-instance_id ) ) ) )
        ( name = 'ACTION'
          t_value = VALUE #(
            ( sign = 'I' option = 'EQ'
              low = zcl_fi_process_job=>gc_action_execute ) ) ) ).

      TRY.
          cl_apj_rt_api=>schedule_job(
            EXPORTING
              iv_job_template_name   = gc_job_template
              iv_job_text            =
                |ZFI_PROCESS exec { ms_instance-instance_id }|
              is_start_info          = ls_start_info
              it_job_parameter_value = lt_params
            IMPORTING
              ev_jobname  = lv_jobname
              ev_jobcount = lv_jobcount ).
        CATCH cx_apj_rt INTO DATA(lx_apj).
          RAISE EXCEPTION TYPE zcx_fi_process_error
            EXPORTING
              textid   = zcx_fi_process_error=>scheduling_failed
              value    = |Instance { ms_instance-instance_id }: |
                       && lx_apj->get_text( )
              previous = lx_apj.
      ENDTRY.

      " Status-second: only update after successful scheduling
      " Optimistic lock: verify status hasn't changed concurrently
      SELECT SINGLE status
        FROM zfi_proc_inst
        WHERE instance_id = @ms_instance-instance_id
        INTO @DATA(lv_current_status).
      IF lv_current_status <> gc_status-new.
        " Race condition: another caller changed the status
        TRY.
            cl_apj_rt_api=>cancel_job(
              iv_jobname  = lv_jobname
              iv_jobcount = lv_jobcount ).
          CATCH cx_apj_rt ##NO_HANDLER.
        ENDTRY.
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Instance { ms_instance-instance_id }|
                   && | status changed concurrently to |
                   && |{ lv_current_status }. Job cancelled.|.
      ENDIF.

      ms_instance-status    = gc_status-exec_requested.
      ms_instance-job_name  = lv_jobname.
      ms_instance-job_count = lv_jobcount.
      save_instance( ).
    ENDMETHOD.
    ```
  - Action C (PRIVATE SECTION or PUBLIC SECTION — add template constant):
    ```abap
    "! APJ job template name for background execution
    CONSTANTS gc_job_template TYPE cl_apj_rt_api=>ty_template_name
      VALUE 'ZFI_PROCESS_JOB_TMPL'.
    ```
  - Notes:
    - **Schedule-first, status-second:** If `schedule_job()` raises `CX_APJ_RT`, the instance status remains NEW — no broken intermediate state.
    - **Timestamp offset:** 2 seconds into the future provides margin for the current LUW to commit before the job picks up the instance.
    - The `iv_job_text` provides a human-readable label in the APJ monitor. Verify the max length of `CL_APJ_RT_API=>TY_JOB_TEXT` — the text "ZFI_PROCESS exec <32-char-GUID>" is ~50 chars; if the type is shorter, truncate or shorten the prefix.
    - `ms_instance-job_name` and `ms_instance-job_count` are the new fields from Task 3.
    > **Field access:** `ms_instance` is the instance's internal data structure (type `ty_process_inst`). After Task 3 adds `JOB_NAME` and `JOB_COUNT` to the DB table, the structure type must also include these fields. Verify that `ty_process_inst` is derived from the DB table (likely via `INCLUDE TYPE` or direct table reference). If it's a separate DDIC structure, it needs to be extended too.

- [ ] **Task 9: Add `request_restart()` method to `ZCL_FI_PROCESS_INSTANCE`**
  - File: `src/zcl_fi_process_instance.clas.abap` *(modify)*
  - Action A (PUBLIC SECTION — method declaration, after `restart`):
    ```abap
    "! Request background restart of this process instance
    "! <p>Schedules an APJ background job to restart this instance.
    "! Instance must be in FAILED or CANCELLED status. Status
    "! transitions to RESTART_REQUESTED on success.</p>
    "! @raising zcx_fi_process_error | If status is not FAILED or
    "!   CANCELLED, or scheduling fails
    METHODS request_restart
      RAISING
        zcx_fi_process_error.
    ```
  - Action B (IMPLEMENTATION — new METHOD block):
    ```abap
    METHOD request_restart.
      " Status guard: only FAILED or CANCELLED instances
      IF NOT ( ms_instance-status = gc_status-failed OR
               ms_instance-status = gc_status-cancelled ).
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Cannot request restart: instance |
                   && |{ ms_instance-instance_id } status is |
                   && |{ ms_instance-status }. |
                   && |Expected: FAILED or CANCELLED.|.
      ENDIF.

      " Schedule-first
      DATA lv_jobname  TYPE cl_apj_rt_api=>ty_jobname.
      DATA lv_jobcount TYPE cl_apj_rt_api=>ty_jobcount.

      DATA(ls_start_info) = VALUE cl_apj_rt_api=>ty_start_info( ).
      GET TIME STAMP FIELD DATA(lv_now).
      ls_start_info-timestamp = cl_abap_tstmp=>add(
        tstmp = lv_now
        secs  = 2 ).

      DATA(lt_params) = VALUE cl_apj_rt_api=>tt_job_parameter_value(
        ( name = 'INSTGUID'
          t_value = VALUE #(
            ( sign = 'I' option = 'EQ'
              low = CONV #( ms_instance-instance_id ) ) ) )
        ( name = 'ACTION'
          t_value = VALUE #(
            ( sign = 'I' option = 'EQ'
              low = zcl_fi_process_job=>gc_action_restart ) ) ) ).

      TRY.
          cl_apj_rt_api=>schedule_job(
            EXPORTING
              iv_job_template_name   = gc_job_template
              iv_job_text            =
                |ZFI_PROCESS restart { ms_instance-instance_id }|
              is_start_info          = ls_start_info
              it_job_parameter_value = lt_params
            IMPORTING
              ev_jobname  = lv_jobname
              ev_jobcount = lv_jobcount ).
        CATCH cx_apj_rt INTO DATA(lx_apj).
          RAISE EXCEPTION TYPE zcx_fi_process_error
            EXPORTING
              textid   = zcx_fi_process_error=>scheduling_failed
              value    = |Instance { ms_instance-instance_id }: |
                       && lx_apj->get_text( )
              previous = lx_apj.
      ENDTRY.

      " Status-second
      " Optimistic lock: verify status hasn't changed concurrently
      SELECT SINGLE status
        FROM zfi_proc_inst
        WHERE instance_id = @ms_instance-instance_id
        INTO @DATA(lv_current_status).
      IF NOT ( lv_current_status = gc_status-failed OR
               lv_current_status = gc_status-cancelled ).
        " Race condition: another caller changed the status
        TRY.
            cl_apj_rt_api=>cancel_job(
              iv_jobname  = lv_jobname
              iv_jobcount = lv_jobcount ).
          CATCH cx_apj_rt ##NO_HANDLER.
        ENDTRY.
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Instance { ms_instance-instance_id }|
                   && | status changed concurrently to |
                   && |{ lv_current_status }. Job cancelled.|.
      ENDIF.

      ms_instance-status    = gc_status-restart_requested.
      ms_instance-job_name  = lv_jobname.
      ms_instance-job_count = lv_jobcount.
      save_instance( ).
    ENDMETHOD.
    ```
  - Notes: Identical pattern to `request_execute()` except: (a) different status guard (FAILED or CANCELLED), (b) ACTION = 'R', (c) target status = `restart_requested`.

- [ ] **Task 10: Extend `restart()` to allow CANCELLED and RESTART_REQUESTED statuses**
  - File: `src/zcl_fi_process_instance.clas.abap` *(modify)*
  - Action: Replace the `restart()` status guard. Current:
    ```abap
    IF ms_instance-status <> gc_status-failed.
    ```
    Replace with:
    ```abap
    IF NOT ( ms_instance-status = gc_status-failed OR
             ms_instance-status = gc_status-cancelled OR
             ms_instance-status = gc_status-restart_requested ).
    ```
    Also update the error message value string:
    ```abap
        value  = |Cannot restart: instance status is |
               && |{ ms_instance-status }. Expected: |
               && |FAILED, CANCELLED or |
               && |RESTART_REQUESTED.|.
    ```
  - Notes: This enables restart from CANCELLED (user-cancelled instance is recoverable) and from RESTART_REQUESTED (the APJ job class calls `restart()` when action='R'). Without allowing RESTART_REQUESTED, the job class's `restart()` call would be rejected by this guard.
    > **Edge case — CANCELLED with no FAILED step:** If the instance was cancelled mid-execution, there may be no step with status FAILED (all steps may be CANCELLED or NEW). The current `restart()` implementation does `READ TABLE mt_steps WITH KEY status = gc_status-failed`. If no FAILED step is found, it raises `no_failed_step`. This behavior is acceptable for now — a cancelled instance where steps are CANCELLED/NEW should be re-executed from the beginning, not restarted. Document this as a known limitation and future enhancement (add a `re_execute()` path for cancelled instances).

- [ ] **Task 11: Extend `cancel()` to handle *_REQUESTED statuses, call cancel_job, clear job fields**
  - File: `src/zcl_fi_process_instance.clas.abap` *(modify)*
  - Action: Replace the `cancel()` method body. Current allowlist is RUNNING or FAILED. New:
    ```abap
    METHOD cancel.
      IF NOT ( ms_instance-status = gc_status-running OR
               ms_instance-status = gc_status-failed OR
               ms_instance-status = gc_status-exec_requested OR
               ms_instance-status = gc_status-restart_requested ).
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Cannot cancel instance |
                   && |{ ms_instance-instance_id }: status |
                   && |{ ms_instance-status } is not |
                   && |cancellable.|.
      ENDIF.

      " Best-effort APJ job cancellation
      IF ms_instance-job_name IS NOT INITIAL.
        TRY.
            cl_apj_rt_api=>cancel_job(
              iv_jobname  = ms_instance-job_name
              iv_jobcount = ms_instance-job_count ).
          CATCH cx_apj_rt.
            " Fire-and-forget: job cancel failure is not fatal.
            " The job-side status guard will prevent execution
            " when it sees CANCELLED status.
        ENDTRY.
      ENDIF.

      ms_instance-status    = gc_status-cancelled.
      CLEAR: ms_instance-job_name,
             ms_instance-job_count.
      GET TIME STAMP FIELD ms_instance-ended_at.
      save_instance( ).
    ENDMETHOD.
    ```
  - Notes:
    - Allowlist extended: RUNNING, FAILED, EXEC_REQUESTED, RESTART_REQUESTED.
    - `CL_APJ_RT_API=>CANCEL_JOB()` called best-effort — if the job can't be cancelled (already finished, infrastructure error), the CATCH swallows the exception silently. The job-side status guard handles this.
    - Job reference fields cleared to blank — cancel is the only action that clears these fields.
    - `GET TIME STAMP` and `save_instance()` unchanged from existing implementation.
    > **Fire-and-forget rationale:** The caller must never be blocked by an APJ infrastructure failure. Setting status to CANCELLED is the primary goal. If the APJ job still runs, the job-side status guard in `ZCL_FI_PROCESS_JOB` detects the CANCELLED status and returns silently.

- [ ] **Task 12: Extend `execute()` status guard to allow EXEC_REQUESTED (APJ entry point)**
  - File: `src/zcl_fi_process_instance.clas.abap` *(modify)*
  - Action: The `execute()` method has a compound outer IF guard followed by a CASE block. The current outer IF (lines 630-635) is:
    ```abap
    IF ms_instance-status <> gc_status-new
       AND NOT ( ms_instance-status = gc_status-failed
                 AND iv_start_from_step IS NOT INITIAL ).
    ```
    This rejects everything except NEW and FAILED+iv_start_from_step. The APJ job class calls `execute()` when the instance is in EXEC_REQUESTED status — this would be rejected by the outer IF before the CASE block is even reached.

    **Replace the outer IF with:**
    ```abap
    IF ms_instance-status <> gc_status-new
       AND NOT ( ms_instance-status = gc_status-failed
                 AND iv_start_from_step IS NOT INITIAL )
       AND ms_instance-status <> gc_status-exec_requested.
    ```

    Then add a WHEN branch in the CASE block for RESTART_REQUESTED (which should NOT be allowed via `execute()`):
    ```abap
        WHEN gc_status-restart_requested.
          RAISE EXCEPTION TYPE zcx_fi_process_error
            EXPORTING
              textid = zcx_fi_process_error=>invalid_status
              value  = |Instance { ms_instance-instance_id }|
                     && | is RESTART_REQUESTED (awaiting |
                     && |background restart). Use restart() |
                     && |or cancel first.|.
    ```

  - Notes:
    - **This is the critical fix:** Without adding `exec_requested` to the outer IF allowlist, the entire APJ background execution path is broken — the job class's `execute()` call would be rejected.
    - EXEC_REQUESTED is allowed to pass through the outer IF and falls through to the execution logic (same as NEW).
    - RESTART_REQUESTED is explicitly rejected in the CASE — you cannot call `execute()` on a RESTART_REQUESTED instance, only `restart()`.
    - The corresponding fix for `restart()` is in Task 10, which adds RESTART_REQUESTED to the restart guard's allowlist.

- [ ] **Task 13: Add `check_job_status()` method to `ZCL_FI_PROCESS_INSTANCE`**
  - File: `src/zcl_fi_process_instance.clas.abap` *(modify)*
  - Action A (PUBLIC SECTION — method declaration):
    ```abap
    "! Check APJ job status for this instance
    "! <p>Reads the APJ job status and returns it alongside the
    "! current instance status. Does NOT auto-transition the
    "! instance — the caller decides what to do with the result.
    "! APJ errors are returned in the status string, not raised.</p>
    "! @parameter rv_job_status | APJ job status text (blank if
    "!   no job reference)
    METHODS check_job_status
      RETURNING
        VALUE(rv_job_status) TYPE string.
    ```
  - Action B (IMPLEMENTATION):
    ```abap
    METHOD check_job_status.
      IF ms_instance-job_name IS INITIAL.
        rv_job_status = 'No background job reference'.
        RETURN.
      ENDIF.

      DATA lv_status     TYPE cl_apj_rt_api=>ty_job_status.
      DATA lv_statustext TYPE cl_apj_rt_api=>ty_job_status_text.

      TRY.
          cl_apj_rt_api=>get_job_status(
            EXPORTING
              iv_jobname         = ms_instance-job_name
              iv_jobcount        = ms_instance-job_count
            IMPORTING
              ev_job_status      = lv_status
              ev_job_status_text = lv_statustext ).

          rv_job_status =
            |Instance status: { ms_instance-status }, |
         && |APJ job status: { lv_statustext } |
         && |({ lv_status }), |
         && |Job: { ms_instance-job_name }/|
         && |{ ms_instance-job_count }|.

        CATCH cx_apj_rt INTO DATA(lx_apj).
          rv_job_status =
            |Job status check failed: { lx_apj->get_text( ) }|.
      ENDTRY.
    ENDMETHOD.
    ```
  - Notes: This is a diagnostic/reconciliation method. Returns a human-readable string combining instance status and APJ job status. Does NOT modify instance status — the caller (monitoring UI, admin) decides what action to take based on the result.

#### Tier 2b — Manager Methods

- [ ] **Task 14: Add `request_execute_process()`, `request_restart_process()`, `check_job_status()` to `ZCL_FI_PROCESS_MANAGER`**
  - File: `src/zcl_fi_process_manager.clas.abap` *(modify)*
  - Action A (PUBLIC SECTION — method declarations):
    ```abap
    "! Request background execution of a process instance
    "! @parameter iv_instance_id | Instance ID
    "! @raising zcx_fi_process_error | If scheduling fails
    METHODS request_execute_process
      IMPORTING
        iv_instance_id TYPE zfi_process_instance_id
      RAISING
        zcx_fi_process_error.

    "! Request background restart of a process instance
    "! @parameter iv_instance_id | Instance ID
    "! @raising zcx_fi_process_error | If scheduling fails
    METHODS request_restart_process
      IMPORTING
        iv_instance_id TYPE zfi_process_instance_id
      RAISING
        zcx_fi_process_error.

    "! Check APJ job status of a process instance
    "! @parameter iv_instance_id | Instance ID
    "! @parameter rv_job_status | Status report string
    "! @raising zcx_fi_process_error | If instance not found
    METHODS check_job_status
      IMPORTING
        iv_instance_id       TYPE zfi_process_instance_id
      RETURNING
        VALUE(rv_job_status) TYPE string
      RAISING
        zcx_fi_process_error.
    ```
  - Action B (IMPLEMENTATION):
    ```abap
    METHOD request_execute_process.
      DATA(lo_instance) = load_process( iv_instance_id ).
      lo_instance->request_execute( ).
    ENDMETHOD.

    METHOD request_restart_process.
      DATA(lo_instance) = load_process( iv_instance_id ).
      lo_instance->request_restart( ).
    ENDMETHOD.

    METHOD check_job_status.
      DATA(lo_instance) = load_process( iv_instance_id ).
      rv_job_status = lo_instance->check_job_status( ).
    ENDMETHOD.
    ```
  - Notes: One-liner delegation pattern, identical to `cancel_process()` and `supersede_process()`. The manager's `check_job_status()` still declares RAISING because `load_process()` can raise if the instance is not found. The instance-level method does NOT raise.

#### Tier 3 — Guard Extensions

- [ ] **Task 15: Extend EST-102 duplicate check to block *_REQUESTED statuses**
  - File: `src/zcl_fi_process_manager.clas.abap` *(modify)*
  - Action: In `create_process()` duplicate check, extend the `line_exists()` chain. Current (after EST-110):
    ```abap
    IF line_exists( lt_existing[ status = gc_status-running   ] ) OR
       line_exists( lt_existing[ status = gc_status-failed    ] ) OR
       line_exists( lt_existing[ status = gc_status-completed ] ).
    ```
    Replace with:
    ```abap
    IF line_exists(
         lt_existing[ status = gc_status-running ] ) OR
       line_exists(
         lt_existing[ status = gc_status-failed ] ) OR
       line_exists(
         lt_existing[ status = gc_status-completed ] ) OR
       line_exists(
         lt_existing[ status = gc_status-exec_requested ] ) OR
       line_exists(
         lt_existing[ status = gc_status-restart_requested ] ).
    ```
    Also update the `value =` error string:
    ```abap
      && | with the same parameters |
      && |(RUNNING, FAILED, COMPLETED, |
      && |EXEC_REQUESTED or RESTART_REQUESTED)|.
    ```
  - Notes: Prevents duplicate scheduling race conditions — if an instance is already scheduled for background execution, a new `create_process()` with the same parameter hash is blocked.
    > **Line length:** The `line_exists()` chain with 5 conditions may exceed 120 chars. Break each `line_exists()` onto its own line with the OR at the end.

- [ ] **Task 16: Extend EST-106 parallel limit to count *_REQUESTED statuses**
  - File: `src/zcl_fi_process_manager.clas.abap` *(modify)*
  - Action: Find the parallel instance count query (EST-106 implementation). The current count likely checks `WHERE status = 'RUNNING'`. Extend to:
    ```abap
    DATA lt_active_statuses TYPE RANGE OF zfi_process_status.
    lt_active_statuses = VALUE #(
      ( sign = 'I' option = 'EQ'
        low = gc_status-running )
      ( sign = 'I' option = 'EQ'
        low = gc_status-exec_requested )
      ( sign = 'I' option = 'EQ'
        low = gc_status-restart_requested ) ).

    SELECT COUNT(*)
      FROM zfi_proc_inst
      INTO @DATA(lv_active_count)
      WHERE process_type = @iv_process_type
        AND status IN @lt_active_statuses.
    ```
  - Notes: Prevents over-scheduling beyond the configured `MAX_PARALLEL_INSTS` limit. Without this, scheduling 100 instances in rapid succession could exceed the limit because EXEC_REQUESTED wouldn't be counted.
    > **Pre-condition:** Verify the actual EST-106 implementation pattern. The above is illustrative — the actual code may use a different query structure. Adapt to match.

- [ ] **Task 17: Unit tests for new methods**
  - File: `src/zcl_fi_process_manager.clas.testclasses.abap` *(modify)*
  - Action: Add test methods. Due to the APJ infrastructure dependency, these tests focus on status guard validation (not actual job scheduling):
    ```abap
    "! request_execute() rejects non-NEW instances
    METHODS test_req_exec_rejects_running
      FOR TESTING RAISING cx_static_check.

    "! request_restart() rejects non-FAILED/CANCELLED instances
    METHODS test_req_restart_rejects_new
      FOR TESTING RAISING cx_static_check.

    "! cancel() accepts EXEC_REQUESTED status
    METHODS test_cancel_exec_requested
      FOR TESTING RAISING cx_static_check.

    "! cancel() accepts RESTART_REQUESTED status
    METHODS test_cancel_restart_requested
      FOR TESTING RAISING cx_static_check.

    "! restart() accepts CANCELLED status
    METHODS test_restart_from_cancelled
      FOR TESTING RAISING cx_static_check.

    "! Duplicate check blocks EXEC_REQUESTED
    METHODS test_blocks_when_exec_requested
      FOR TESTING RAISING cx_static_check.

    "! Duplicate check blocks RESTART_REQUESTED
    METHODS test_blocks_when_restart_req
      FOR TESTING RAISING cx_static_check.
    ```
  - Notes: Tests that call `request_execute()` or `request_restart()` on a valid instance will attempt to call `CL_APJ_RT_API=>SCHEDULE_JOB()` which requires APJ infrastructure. Two options:
    1. **Test the guard only:** Create instance, set to wrong status, call method, assert exception
    2. **Test with APJ mock:** If the APJ API is mockable, create a test double. This is a more complex approach and may be deferred.

    For this spec, focus on guard-only tests (option 1). The integration test of actual background scheduling should be performed manually or via the health check framework.

---

### Acceptance Criteria

- [ ] **AC1:** Given a NEW instance, when `request_execute()` is called, then status transitions to EXEC_REQUESTED and `JOB_NAME`/`JOB_COUNT` are populated.

- [ ] **AC2:** Given a FAILED instance, when `request_restart()` is called, then status transitions to RESTART_REQUESTED and `JOB_NAME`/`JOB_COUNT` are populated.

- [ ] **AC3:** Given a CANCELLED instance, when `request_restart()` is called, then status transitions to RESTART_REQUESTED.

- [ ] **AC4:** Given an EXEC_REQUESTED instance, when the APJ job starts and instance status is still EXEC_REQUESTED, then the job calls `execute()` and the instance transitions to RUNNING → COMPLETED/FAILED.

- [ ] **AC5:** Given an EXEC_REQUESTED instance that was cancelled before the job started, when the APJ job starts and sees status CANCELLED, then the job returns silently without executing.

- [ ] **AC6:** Given an EXEC_REQUESTED instance, when `cancel()` is called, then status transitions to CANCELLED, `CL_APJ_RT_API=>CANCEL_JOB()` is called best-effort, and job reference fields are cleared.

- [ ] **AC7:** Given a RUNNING instance (background), when `cancel()` is called, then status transitions to CANCELLED and `CL_APJ_RT_API=>CANCEL_JOB()` is called best-effort.

- [ ] **AC8:** Given a CANCELLED instance, when `restart()` is called (foreground), then the instance transitions to RUNNING and execution resumes from the failed step.

- [ ] **AC9:** Given an instance with no job reference (foreground execution), when `check_job_status()` is called, then it returns "No background job reference".

- [ ] **AC10:** Given an instance with a job reference, when `check_job_status()` is called, then it returns a string containing both the instance status and the APJ job status.

- [ ] **AC11:** Given a non-NEW instance, when `request_execute()` is called, then `zcx_fi_process_error=>invalid_status` is raised.

- [ ] **AC12:** Given a non-FAILED/CANCELLED instance, when `request_restart()` is called, then `zcx_fi_process_error=>invalid_status` is raised.

- [ ] **AC13:** Given an EXEC_REQUESTED instance with the same hash, when `create_process()` is called, then `zcx_fi_process_error=>duplicate_instance` is raised.

- [ ] **AC14:** Given an EXEC_REQUESTED instance exists for the same process type, when the parallel limit is reached (counting RUNNING + EXEC_REQUESTED + RESTART_REQUESTED), then `create_process()` raises `parallel_limit_exceeded`.

- [ ] **AC15:** Given `schedule_job()` raises `CX_APJ_RT`, when `request_execute()` is called, then `zcx_fi_process_error=>scheduling_failed` is raised and instance status remains NEW (unchanged).

---

## Additional Context

### Dependencies

- **EST-102 (parameter hash duplicity):** Fully implemented. Tasks 15 extends it.
- **EST-106 (max parallel instances):** Fully implemented. Task 16 extends it.
- **EST-110 (SUPERSEDED status):** Fully implemented. SUPERSEDED domain value at VALPOS 0008 confirmed.
- **Task ordering within this spec:**
  - Tasks 1–3 (DDIC) must be activated before ABAP compilation
  - Task 4 (exceptions) before Tasks 5, 8, 9 (methods that raise them)
  - Task 5 (job class) requires Tasks 1, 4, 7
  - Task 6 (catalog/template) requires Task 5
  - Tasks 8, 9 (request methods) require Tasks 5, 6, 7
  - Tasks 10, 11, 12 (extensions) require Task 7
  - Tasks 15, 16 (guard extensions) can be done in parallel with Tasks 8–13
  - Task 17 (tests) requires all other tasks

### Testing Strategy

**Unit Tests** (Task 17):
- Guard-only tests that validate status preconditions
- Do NOT require APJ infrastructure — test the status guards, not the scheduling
- Run via ATC: `RUN UNIT TESTS FOR CLASS zcl_fi_process_manager`

**Manual Integration Testing:**
1. Create a TEST_BGEXEC process type with a simple step that sleeps 10 seconds
2. Call `request_execute_process()` — verify EXEC_REQUESTED status + JOB_NAME/JOB_COUNT populated
3. Check APJ monitor (SM37 or Fiori app "Application Jobs") — verify job appears
4. Wait for job to complete — verify instance transitions to COMPLETED
5. Cancel test: `request_execute_process()` then immediately `cancel_process()` — verify CANCELLED, job cancelled in APJ monitor
6. Background restart: Create instance, execute (foreground, make it fail), then `request_restart_process()` — verify RESTART_REQUESTED, job runs, instance completes
7. `check_job_status()` — verify returns combined status string
8. Restart from CANCELLED: Cancel a running instance, then `restart()` — verify it resumes

### Notes

- **IF_APJ_DT_EXEC_OBJECT is legacy but functional:** SAP marks this as legacy and recommends `IF_APJ_RT_RUN` for new developments, but `IF_APJ_RT_RUN` is not available on S/4HANA 7.58 on-premise. The legacy interface is fully released (Clean Core Level A) and will continue to work.
- **No cooperative cancellation in this spec:** Long-running steps cannot be interrupted mid-execution. `cancel()` sets the status, but the step continues until it finishes. Between-step cooperative cancellation (checking status before each step) is a future enhancement.
- **CANCELLED restart limitation:** `restart()` currently requires a FAILED step to find the restart point. A CANCELLED instance with no FAILED step (cancelled mid-execution, all steps are CANCELLED/NEW) will raise `no_failed_step`. A future enhancement should add a `re_execute_from_step()` or similar path.
- **APJ job template name is hardcoded:** `gc_job_template` is a constant in the instance class. If the template name needs to be configurable per process type, this can be moved to the `ZFI_PROC_TYPE` customizing table in a future enhancement.
- **Race condition safety:** The schedule-first-status-second pattern, combined with the job-side status guard and optimistic locking (Technical Decision 11), ensures no race conditions between scheduling and cancellation or between foreground and background execution. If two users simultaneously try to schedule the same instance, the first one succeeds (changes status from NEW to EXEC_REQUESTED) and the second's optimistic lock check detects the change and cancels the orphaned job.
- **Adversarial review completed (2026-03-18):** 15 findings identified and resolved — 1 critical (execute() outer IF guard), 4 high (COMMIT behavior, contradictory tasks, race condition), 6 medium, 4 low. All findings addressed in this version of the spec.
