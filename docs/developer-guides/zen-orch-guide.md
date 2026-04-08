# ZEN_ORCH Orchestration Engine — Complete Guide

**Audience**: Newcomer ABAP developer or system administrator  
**Repository**: `cz.en.orch`  
**Last Updated**: 2026-04-06

---

## Table of Contents

1. [What is ZEN_ORCH?](#1-what-is-zen_orch)
2. [Core Concepts](#2-core-concepts)
3. [Database Tables](#3-database-tables)
4. [Status Lifecycle](#4-status-lifecycle)
5. [Configuration — Setting Up a Process](#5-configuration--setting-up-a-process)
6. [Running a Process](#6-running-a-process)
7. [Monitoring and Operations](#7-monitoring-and-operations)
8. [Background Job (Sweep)](#8-background-job-sweep)
9. [Scheduling (Automatic Triggering)](#9-scheduling-automatic-triggering)
10. [Writing a Custom Adapter](#10-writing-a-custom-adapter)
11. [Registering an Adapter](#11-registering-an-adapter)
12. [Error Handling and Logging](#12-error-handling-and-logging)
13. [Health Check OData Service](#13-health-check-odata-service)
14. [Integration Test](#14-integration-test)
15. [Parameters and JSON Merging](#15-parameters-and-json-merging)
16. [Quick Reference](#16-quick-reference)

---

## 1. What is ZEN_ORCH?

ZEN_ORCH is a **lightweight process orchestration engine** built in ABAP. Think of it as a mini-workflow engine that:

- Defines reusable **process templates** (called *Scores*)
- Runs **instances** of those templates (called *Performances*)
- Dispatches each step of a performance to a **backend system** via a pluggable **Adapter**
- Polls adapter status in a periodic background **Sweep**
- Supports parallel steps, synchronisation gates, loops, breakpoints, and CRON-based scheduling

The engine does **not** contain business logic itself — it only orchestrates. All real work is done by adapters.

---

## 2. Core Concepts

### Score (Process Template)

A **Score** is a reusable definition of a process — like a recipe. It is stored in two tables:

- `ZEN_ORCH_SCORE` — the header (ID, description, default parameters)
- `ZEN_ORCH_S_STEP` — the ordered list of steps in the score

A score never changes at runtime. It is the static blueprint.

### Performance (Running Instance)

A **Performance** is one execution of a score — a specific run started at a specific moment with specific parameters. It lives in:

- `ZEN_ORCH_PERF` — the instance header (which score, current status, runtime parameters)
- `ZEN_ORCH_P_STEP` — one row per step, tracking execution state

Each performance is identified by a UUID (`PERF_UUID`).

### Sweep

The **Sweep** is the engine's heartbeat. Every time `sweep_all()` is called (by the APJ background job), the engine:

1. Fires any CRON schedules that are due
2. Finds all PENDING/RUNNING performances
3. Advances each performance as far as possible — dispatching ready steps to adapters and collecting results

Sweep runs without any user interaction. You schedule it once; it does the rest.

### Adapter

An **Adapter** is a plugin class that bridges the engine to a real backend (e.g., a batch job, an RFC call, an external API). Each adapter implements the interface `ZIF_EN_ORCH_ADAPTER`.

The engine only calls adapters — it has no knowledge of what they actually do.

### Gate

A **Gate** is a synchronisation element inside a score. It does not dispatch work to an adapter. Instead, it waits until **all preceding steps sharing the same `REF_ID` group** have status COMPLETED before allowing the performance to proceed.

Use gates to synchronise parallel branches.

### Breakpoint Gate

A **Breakpoint Gate** is a GATE with `IS_BREAKPOINT = X`. When the engine reaches it, the performance enters status `B` (Paused) and waits for an operator to explicitly call `resume_performance()`. Use this for manual approval checkpoints.

### Loop

Steps with `ELEM_TYPE = LOOP` and `ELEM_TYPE = END_LOOP` delimit a repeating section. The engine tracks iterations via `LOOP_ITERATION` on the step instance rows.

---

## 3. Database Tables

### ZEN_ORCH_ADPT_R — Adapter Registry

Maps an adapter type code to its ABAP implementation class.

| Field | Type | Description |
|-------|------|-------------|
| `ADAPTER_TYPE` | Key | Short identifier (e.g., `MOCK`, `BATCH_JOB`) |
| `IMPL_CLASS` | | Fully-qualified ABAP class name (e.g., `ZCL_EN_ORCH_ADAPTER_MOCK`) |

Maintained via SM30 or direct table maintenance. Every adapter you create must have a row here.

---

### ZEN_ORCH_SCORE — Score Headers

| Field | Description |
|-------|-------------|
| `SCORE_ID` | Unique name for the score |
| `DESCRIPTION` | Human-readable label |
| `PARAMS_JSON` | Default JSON parameters passed to all steps (lowest priority in merge) |

---

### ZEN_ORCH_S_STEP — Score Step Definitions

| Field | Description |
|-------|-------------|
| `SCORE_ID` | Parent score |
| `SCORE_SEQ` | Sequence number (execution order) |
| `ELEM_TYPE` | `STEP`, `GATE`, `LOOP`, `END_LOOP`, `PREREQ_GATE` |
| `REF_ID` | Group identifier for gate synchronisation; also loop label |
| `ADAPTER_TYPE` | Which adapter handles this step (only for `STEP` type) |
| `PARAMS_JSON` | Step-level JSON parameters (highest priority in merge) |
| `MAX_PARALLEL` | Maximum parallel instances of this step (used with loops) |
| `IS_BREAKPOINT` | `X` = this gate is a breakpoint (pause for operator approval) |

Sequence numbers must be unique within a score and define execution order.

---

### ZEN_ORCH_SCHED — Schedules

| Field | Description |
|-------|-------------|
| `SCHEDULE_ID` | Unique schedule name |
| `SCORE_ID` | Which score to instantiate |
| `CRON_EXPR` | 5-field CRON expression (see §9) |
| `ACTIVE` | `X` = schedule is active |
| `PARAMS_JSON` | Parameters to pass when creating the performance |
| `LAST_FIRED_TS` | Timestamp of last successful fire (updated by engine) |

---

### ZEN_ORCH_PERF — Performance Instances (Runtime)

| Field | Description |
|-------|-------------|
| `PERF_UUID` | UUID identifying this run |
| `SCORE_ID` | Which score this run is based on |
| `STATUS` | Current status (`P`, `R`, `C`, `F`, `X`, `B`) |
| `PARAMS_JSON` | Runtime parameters (medium priority in merge) |

Do not edit this table manually. Let the engine manage it.

---

### ZEN_ORCH_P_STEP — Performance Step Instances (Runtime)

| Field | Description |
|-------|-------------|
| `PERF_UUID` | Parent performance |
| `SCORE_SEQ` | Sequence from score definition |
| `LOOP_ITERATION` | Current loop iteration (0 for non-loop steps) |
| `ELEM_TYPE` | Copy of element type from score |
| `REF_ID` | Copy of REF_ID from score |
| `ADAPTER_TYPE` | Copy of adapter type from score |
| `PARAMS_JSON` | Merged parameters for this step execution |
| `STATUS` | Step-level status (`P`, `R`, `C`, `F`, `X`) |
| `WORK_UNIT_HANDLE` | Opaque handle returned by the adapter (used for polling) |
| `IS_BREAKPOINT` | Copy of breakpoint flag from score |

---

## 4. Status Lifecycle

### Performance and Step Statuses

| Code | Label | Meaning |
|------|-------|---------|
| `P` | Pending | Created, not yet dispatched |
| `R` | Running | Adapter work unit active |
| `C` | Completed | Successfully finished |
| `F` | Failed | Error occurred |
| `X` | Cancelled | Operator cancelled |
| `B` | Paused | Stopped at a breakpoint gate — waiting for operator |

### Normal Happy Path (single step, no gate)

```
create_performance()
    → Performance: P
    → Steps: P

sweep_all() [first run]
    → Step dispatched to adapter → Step: R
    → Performance: R

sweep_all() [subsequent runs]
    → Adapter reports COMPLETED → Step: C
    → All steps done → Performance: C
```

### With a Parallel Gate

```
Score: STEP-A (seq 10) | STEP-B (seq 20) | GATE (seq 30, REF_ID=G1) | STEP-C (seq 40)

STEP-A and STEP-B both have REF_ID=G1.
→ Engine dispatches both in parallel.
→ GATE (seq 30) waits until both STEP-A and STEP-B are C.
→ Once gate is satisfied, STEP-C is dispatched.
```

### With a Breakpoint Gate

```
Score: STEP-A → GATE (IS_BREAKPOINT=X) → STEP-B

After STEP-A completes:
→ Engine hits breakpoint gate → Performance status: B (Paused)
→ Operator reviews results
→ Operator calls resume_performance(iv_perf_uuid)
→ Gate cleared to C → STEP-B dispatched → Performance: R
```

### Failure and Restart

```
STEP-A fails → Step: F → Performance: F

Operator investigates root cause, fixes external issue.
Operator calls restart_performance(iv_perf_uuid)
→ Failed steps reset to P → Performance: R
→ Next sweep re-dispatches failed steps
```

---

## 5. Configuration — Setting Up a Process

To configure a new process, you need to:

1. **Register your adapter** in `ZEN_ORCH_ADPT_R` (see §11)
2. **Create a score** in `ZEN_ORCH_SCORE`
3. **Define score steps** in `ZEN_ORCH_S_STEP`
4. *(Optional)* **Create a schedule** in `ZEN_ORCH_SCHED`

### Example: Simple 2-Step Linear Process

**ZEN_ORCH_SCORE**:

| SCORE_ID | DESCRIPTION | PARAMS_JSON |
|----------|-------------|-------------|
| `MY_PROCESS` | My first process | `{"env":"PRD"}` |

**ZEN_ORCH_S_STEP**:

| SCORE_ID | SCORE_SEQ | ELEM_TYPE | ADAPTER_TYPE | PARAMS_JSON |
|----------|-----------|-----------|--------------|-------------|
| `MY_PROCESS` | 10 | `STEP` | `MY_ADAPTER` | `{"step":"extract"}` |
| `MY_PROCESS` | 20 | `STEP` | `MY_ADAPTER` | `{"step":"load"}` |

Steps execute in sequence order (10 → 20). Step 20 only starts after step 10 completes.

### Example: Parallel Steps with Gate

| SCORE_SEQ | ELEM_TYPE | REF_ID | ADAPTER_TYPE | Notes |
|-----------|-----------|--------|--------------|-------|
| 10 | `STEP` | `G1` | `MY_ADAPTER` | Parallel branch A |
| 20 | `STEP` | `G1` | `MY_ADAPTER` | Parallel branch B |
| 30 | `GATE` | `G1` | | Wait for both 10 and 20 |
| 40 | `STEP` | | `MY_ADAPTER` | Runs after gate |

Steps 10 and 20 have the same `REF_ID = G1`. The engine dispatches them in parallel. The GATE at sequence 30 (also `REF_ID = G1`) waits for all `G1` steps to complete.

### Example: Breakpoint Gate

| SCORE_SEQ | ELEM_TYPE | IS_BREAKPOINT | Notes |
|-----------|-----------|---------------|-------|
| 10 | `STEP` | | First step |
| 20 | `GATE` | `X` | Pause for approval |
| 30 | `STEP` | | Second step (runs after approval) |

---

## 6. Running a Process

Processes are started programmatically using the engine singleton.

### Starting a Performance

```abap
DATA(lo_engine) = ZCL_EN_ORCH_ENGINE=>get_instance( ).

DATA(lv_perf_uuid) = lo_engine->create_performance(
    iv_score_id    = 'MY_PROCESS'
    iv_params_json = '{"run_date":"20260101"}'   " optional
).
```

`create_performance()` creates the performance and all step instances with status `P`. Nothing executes yet — execution happens in the next sweep.

### Cancelling a Performance

```abap
lo_engine->cancel_performance( iv_perf_uuid = lv_perf_uuid ).
```

Cancels a PENDING or RUNNING performance. The engine makes a best-effort call to each adapter's `cancel()` method for running steps.

### Restarting a Failed Performance

```abap
lo_engine->restart_performance( iv_perf_uuid = lv_perf_uuid ).
```

Resets all FAILED steps back to PENDING. The performance re-enters RUNNING and the next sweep will re-dispatch.

### Resuming a Paused Performance

```abap
lo_engine->resume_performance( iv_perf_uuid = lv_perf_uuid ).
```

Clears the breakpoint gate to COMPLETED. The performance resumes from the next step.

---

## 7. Monitoring and Operations

### Checking Status — Database

Query `ZEN_ORCH_PERF` directly in SE16N or via ABAP:

```abap
SELECT * FROM zen_orch_perf
  WHERE score_id = 'MY_PROCESS'
  ORDER BY perf_uuid.
```

Check individual step status in `ZEN_ORCH_P_STEP`.

### Checking Logs — SLG1

All engine activity is logged to BAL (Business Application Log):

1. Open transaction `SLG1`
2. Object: `ZEN_ORCH`
3. Subobject: `SWEEP`
4. Run date: today (or the relevant date)

Errors, warnings, and informational messages from every sweep run are recorded here.

### Health Check OData Service

A Fiori-compatible OData V4 service provides a health check overview:

- **Service Binding**: `ZEN_ORCH_UI_HEALTH_O4`
- Activate in `/IWFND/V4_ADMIN` if not already active
- The underlying CDS view is `ZEN_ORCH_I_HEALTH_CHK_CE`

The health check verifies: exception class, logger, adapter factory, score configuration, sweep behaviour, cancel, restart, and resume operations.

---

## 8. Background Job (Sweep)

The engine only advances performances when `sweep_all()` is called. This is done by a scheduled APJ (Application Job).

### Job Details

| Item | Value |
|------|-------|
| Job Catalog | `ZEN_ORCH_JOB_CAT` |
| Job Template | `ZEN_ORCH_JOB_TMPL` |
| Job Class | `ZCL_EN_ORCH_JOB_SWEEP` |
| Parameters | None |

### Scheduling the Sweep Job

1. Open Fiori app **Application Jobs** (or transaction `STC01`)
2. Create a new job from template `ZEN_ORCH_JOB_TMPL`
3. Set recurrence: **every 1 minute** (recommended)
4. Activate the job schedule

The job never terminates with a dump — all exceptions are caught internally and written to BAL (`ZEN_ORCH` / `SWEEP`).

### What Happens in Each Sweep

1. Engine reads all active CRON schedules from `ZEN_ORCH_SCHED`
2. For each due schedule, calls `create_performance()` and updates `LAST_FIRED_TS`
3. Engine reads all performances with status `P` or `R`
4. For each performance, evaluates which steps are ready to run:
   - Steps with all predecessors completed → dispatch to adapter
   - Steps already RUNNING → poll adapter for new status
   - Gates → check if all group members are complete
5. Updates all statuses in the database
6. Logs everything to BAL

---

## 9. Scheduling (Automatic Triggering)

Schedules allow scores to be triggered automatically on a CRON-based timetable.

### CRON Expression Format

```
<minute> <hour> <day> <month> <day-of-week>
```

| Field | Values | Wildcards |
|-------|--------|-----------|
| minute | 0–59 | `*`, `*/N` |
| hour | 0–23 | `*`, `*/N` |
| day | 1–31 | `*`, `*/N` |
| month | 1–12 | `*`, `*/N` |
| day-of-week | 1–7 (Mon=1, Sun=7) | `*`, `*/N` |

### Examples

| CRON Expression | Meaning |
|----------------|---------|
| `*/15 * * * *` | Every 15 minutes |
| `0 6 * * 1` | Every Monday at 06:00 |
| `0 0 1 * *` | First day of every month at midnight |
| `30 8 * * 1-5` | Weekdays at 08:30 *(Note: range syntax not supported — use separate schedules)* |
| `* * * * *` | Every minute |

### Creating a Schedule

Insert a row into `ZEN_ORCH_SCHED`:

| Field | Example |
|-------|---------|
| `SCHEDULE_ID` | `MY_DAILY_RUN` |
| `SCORE_ID` | `MY_PROCESS` |
| `CRON_EXPR` | `0 6 * * 1` |
| `ACTIVE` | `X` |
| `PARAMS_JSON` | `{"mode":"AUTO"}` |

Set `ACTIVE = X` to enable. The next sweep that runs at or after 06:00 on a Monday will fire this schedule.

---

## 10. Writing a Custom Adapter

An adapter is an ABAP class that implements `ZIF_EN_ORCH_ADAPTER`. It is the bridge between ZEN_ORCH and your real process backend.

### Interface Methods

```abap
INTERFACE zif_en_orch_adapter.

  " Start a new work unit. Return a handle for polling.
  METHODS start
    IMPORTING iv_params_json        TYPE string
    RETURNING VALUE(rv_handle)      TYPE string
    RAISING   zcx_en_orch_error.

  " Poll the current status of a running work unit.
  METHODS get_status
    IMPORTING iv_handle             TYPE string
              iv_params_json        TYPE string
    RETURNING VALUE(rv_status)      TYPE zen_orch_status  " C / R / F / X
    RAISING   zcx_en_orch_error.

  " Cancel a running work unit (best-effort).
  METHODS cancel
    IMPORTING iv_handle             TYPE string
    RAISING   zcx_en_orch_error.

  " Restart a failed work unit. Return new handle.
  METHODS restart
    IMPORTING iv_handle             TYPE string
              iv_params_json        TYPE string
    RETURNING VALUE(rv_handle)      TYPE string
    RAISING   zcx_en_orch_error.

  " Retrieve structured result data after completion.
  METHODS get_result
    IMPORTING iv_handle             TYPE string
    RETURNING VALUE(rv_result_json) TYPE string
    RAISING   zcx_en_orch_error.

  " Return a URL or transaction link for drill-through monitoring.
  METHODS get_detail_link
    IMPORTING iv_handle             TYPE string
    RETURNING VALUE(rv_link)        TYPE string.

ENDINTERFACE.
```

### Implementation Skeleton

```abap
CLASS zcl_my_adapter DEFINITION PUBLIC FINAL.
  PUBLIC SECTION.
    INTERFACES zif_en_orch_adapter.
ENDCLASS.

CLASS zcl_my_adapter IMPLEMENTATION.

  METHOD zif_en_orch_adapter~start.
    " Parse iv_params_json
    " Trigger your backend process (batch job, RFC, REST call, etc.)
    " Return an opaque handle that identifies the work unit
    rv_handle = 'MY_JOB_ID_12345'.
  ENDMETHOD.

  METHOD zif_en_orch_adapter~get_status.
    " Use iv_handle to query your backend for current status
    " Map your backend status to: C (Completed), R (Running), F (Failed), X (Cancelled)
    rv_status = 'C'.
  ENDMETHOD.

  METHOD zif_en_orch_adapter~cancel.
    " Best-effort cancellation — do not RAISE if already finished
  ENDMETHOD.

  METHOD zif_en_orch_adapter~restart.
    " Re-trigger the work unit; return new handle
    rv_handle = 'MY_JOB_ID_12346'.
  ENDMETHOD.

  METHOD zif_en_orch_adapter~get_result.
    " Return result payload as JSON string (optional)
    rv_result_json = '{"rows_processed":100}'.
  ENDMETHOD.

  METHOD zif_en_orch_adapter~get_detail_link.
    " Return a URL or transaction string for monitoring (optional)
    rv_link = 'SM37'.
  ENDMETHOD.

ENDCLASS.
```

### Key Rules for Adapter Authors

1. **`start()` must be idempotent-safe** — if called twice with the same parameters, it should either return the same handle or start a new job cleanly. The engine guarantees it calls `start()` only once per step instance, but defensive coding is good practice.

2. **`get_status()` should never RAISE** unless the backend is completely unreachable — return `F` instead of raising for transient failures so the performance can be restarted.

3. **Never raise `ZCX_EN_ORCH_ERROR` from `cancel()`** if the work unit is already finished — just return silently.

4. **`get_result()` and `get_detail_link()` are optional** — the engine does not fail if they return empty strings.

5. **Raise `ZCX_EN_ORCH_ERROR`** (not any other exception class) for unrecoverable errors. Populate `mv_detail` for diagnostics.

---

## 11. Registering an Adapter

After creating your adapter class, register it:

1. Open table `ZEN_ORCH_ADPT_R` in SM30 or SE16N
2. Insert a row:

| ADAPTER_TYPE | IMPL_CLASS |
|--------------|------------|
| `MY_ADAPTER` | `ZCL_MY_ADAPTER` |

3. Use `ADAPTER_TYPE` as the value of `ADAPTER_TYPE` in your score steps (`ZEN_ORCH_S_STEP`)

The adapter factory (`ZCL_EN_ORCH_ADAPTER_FACTORY`) reads this table at runtime and instantiates the class dynamically. No code change is needed in the engine.

---

## 12. Error Handling and Logging

### Exception Class

All engine errors use a single exception class: `ZCX_EN_ORCH_ERROR`

Attributes:

| Attribute | Description |
|-----------|-------------|
| `mv_perf_uuid` | UUID of the affected performance (if applicable) |
| `mv_adapter_type` | Adapter type that caused the error (if applicable) |
| `mv_detail` | Human-readable error description |

### Logging Interface

The engine uses a structured logger (`ZIF_EN_ORCH_LOGGER`) backed by BALI (`ZCL_EN_ORCH_LOGGER`).

Methods:

| Method | Use |
|--------|-----|
| `log_info(iv_msg)` | Informational message |
| `log_warning(iv_msg)` | Warning — process continues |
| `log_error(iv_msg)` | Error — something went wrong |
| `log_exception(ix_ex)` | Log a caught exception |
| `save()` | Persist all buffered log entries to BAL |

All sweep logs are saved to BAL object `ZEN_ORCH`, subobject `SWEEP`.

**Important**: BAL object `ZEN_ORCH` must be registered in table `BALOBJCT` (via SM30). If it is missing, logging will fail silently.

### Verifying BAL Object Registration

1. Open SM30 → Maintenance view for `BALOBJCT`
2. Verify an entry exists for object `ZEN_ORCH`
3. If missing, insert it with description "ZEN_ORCH Orchestration Engine"

---

## 13. Health Check OData Service

ZEN_ORCH ships with an OData V4 health check service to verify system readiness.

### Components

| Component | Name |
|-----------|------|
| CDS View | `ZEN_ORCH_I_HEALTH_CHK_CE` |
| Service Definition | `ZEN_ORCH_UI_HEALTH` |
| Service Binding | `ZEN_ORCH_UI_HEALTH_O4` |

### Activation

1. Open transaction `/IWFND/V4_ADMIN`
2. Click **Add Service**
3. Search for `ZEN_ORCH_UI_HEALTH_O4`
4. Activate

### What It Checks

The health check service verifies end-to-end that the following engine components are operational:

- Exception class instantiation
- Logger creation and save
- Adapter factory lookup
- Score configuration existence
- Sweep execution (non-destructive)
- Cancel, restart, and resume operations

Use this endpoint in monitoring dashboards or automated health checks.

---

## 14. Integration Test

ZEN_ORCH includes a self-contained integration test program.

### Running the Test

1. Open transaction `SE38`
2. Enter program: `ZEN_ORCH_TEST`
3. Press `F8` to execute

### What It Does

The test performs a complete end-to-end execution:

1. Creates a test score `ZIT_E2E` with a `STEP → GATE → STEP` pattern
2. Uses the `MOCK` adapter (completes immediately, no real backend)
3. Starts a performance
4. Runs multiple sweep cycles
5. Verifies all steps reach COMPLETED status
6. **Cleans up all test data** — leaves no artefacts in the database

### Expected Result

All test cases should pass with green status. If any test fails, check:

- `ZEN_ORCH_ADPT_R` has an entry for adapter type `MOCK` pointing to `ZCL_EN_ORCH_ADAPTER_MOCK`
- BAL object `ZEN_ORCH` is registered (see §12)
- No locks or transactional issues on the DB tables

---

## 15. Parameters and JSON Merging

ZEN_ORCH uses a three-level parameter merge. Each step receives a single merged JSON string as `iv_params_json`.

### Merge Levels (lowest to highest priority)

| Level | Source | Priority |
|-------|--------|----------|
| Score | `ZEN_ORCH_SCORE.PARAMS_JSON` | Lowest — default for all steps |
| Performance | `ZEN_ORCH_PERF.PARAMS_JSON` | Middle — set at runtime by caller |
| Step | `ZEN_ORCH_S_STEP.PARAMS_JSON` | Highest — step-specific overrides |

### Merge Behaviour

- All three JSON objects are merged by key
- Higher priority values override lower priority values for the same key
- Non-conflicting keys from all levels are preserved

### Example

Score-level: `{"env":"QAS","timeout":30}`  
Performance-level: `{"run_date":"20260101"}`  
Step-level: `{"timeout":60,"step":"extract"}`

Merged result for the step:
```json
{
  "env": "QAS",
  "timeout": 60,
  "run_date": "20260101",
  "step": "extract"
}
```

`timeout` from step-level (60) overrides score-level (30). `env` and `run_date` are carried through unchanged.

---

## 16. Quick Reference

### Key Transactions

| Transaction | Purpose |
|-------------|---------|
| `SE16N` | Browse DB tables directly |
| `SM30` | Maintain configuration tables (adapter registry, schedules) |
| `SLG1` | View BAL logs (Object: `ZEN_ORCH`, Subobject: `SWEEP`) |
| `SE38` | Run integration test `ZEN_ORCH_TEST` |
| `/IWFND/V4_ADMIN` | Activate OData health check service |
| `STC01` / Fiori App Jobs | Schedule the sweep background job |

### Engine API Cheat Sheet

```abap
DATA(lo) = ZCL_EN_ORCH_ENGINE=>get_instance( ).

" Start a process
DATA(lv_uuid) = lo->create_performance(
    iv_score_id    = 'MY_SCORE'
    iv_params_json = '{"key":"value"}' ).

" Cancel it
lo->cancel_performance( iv_perf_uuid = lv_uuid ).

" Restart after failure
lo->restart_performance( iv_perf_uuid = lv_uuid ).

" Resume after breakpoint
lo->resume_performance( iv_perf_uuid = lv_uuid ).

" Manually trigger a sweep (normally done by APJ)
lo->sweep_all( ).
```

### Status Codes

| Code | Meaning | Operator Action |
|------|---------|-----------------|
| `P` | Pending | Wait for next sweep |
| `R` | Running | Wait for completion |
| `C` | Completed | None |
| `F` | Failed | Investigate → `restart_performance()` |
| `X` | Cancelled | None |
| `B` | Paused (Breakpoint) | Review → `resume_performance()` |

### Adapter ELEM_TYPE Reference

| ELEM_TYPE | Dispatches to Adapter? | Description |
|-----------|----------------------|-------------|
| `STEP` | Yes | Executable unit |
| `GATE` | No | Synchronisation wait |
| `LOOP` | No | Loop start marker |
| `END_LOOP` | No | Loop end marker |
| `PREREQ_GATE` | No | Cross-performance dependency (Phase 2) |

### Common Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Performance stuck in `P` | Sweep job not running | Schedule `ZEN_ORCH_JOB_TMPL` in Fiori App Jobs |
| Performance stuck in `R` | Adapter `get_status()` not returning `C` | Check adapter logic; check BAL logs in SLG1 |
| Performance in `F` | Adapter raised `ZCX_EN_ORCH_ERROR` | Check SLG1 for detail; fix root cause; call `restart_performance()` |
| No logs in SLG1 | BAL object `ZEN_ORCH` not registered | Register in SM30 → `BALOBJCT` |
| Adapter not found | Missing row in `ZEN_ORCH_ADPT_R` | Insert adapter type → class mapping |
| Schedule not firing | `ACTIVE` flag not set, or CRON expression wrong | Verify `ACTIVE = X` and CRON syntax |
