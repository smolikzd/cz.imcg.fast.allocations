---
story_id: "zen-5-3"
epic_id: "zen-epic-5"
title: "Implement cancel and restart lifecycle operations"
target_repository: "cz.en.orch"
depends_on:
  - "zen-5-2"
constitution_principles:
  - "Principle I - DDIC-First"
  - "Principle II - SAP Standards"
  - "Principle V - Error Handling"
status: "done"
created: "2026-04-06"
---

# Story zen-5-3: Implement Cancel and Restart Lifecycle Operations

Status: done

## Story

As an operator,
I want to cancel a running performance or restart a failed performance,
so that I can intervene when a performance is stuck or has failed and needs to be retried.

## Acceptance Criteria

1. **Given** a performance in STATUS = 'R' (RUNNING) or 'P' (PENDING)
   **When** `cancel_performance( iv_perf_uuid )` is called
   **Then** for each step in STATUS = 'R' (RUNNING), `adapter->cancel( iv_handle )` is called (best-effort — exceptions are caught and logged but do not abort the cancel)
   **And** all steps whose STATUS is NOT 'C' (COMPLETED) are set to STATUS = 'X' (CANCELLED)
   **And** the performance STATUS is set to 'X' (CANCELLED)
   **And** `COMMIT WORK AND WAIT` is issued

2. **Given** a performance in STATUS = 'F' (FAILED) with at least one step in STATUS = 'F'
   **When** `restart_performance( iv_perf_uuid )` is called
   **Then** all FAILED steps are reset to STATUS = 'P' (PENDING) and their `WORK_UNIT_HANDLE` is cleared
   **And** the performance STATUS is set to 'R' (RUNNING)
   **And** `COMMIT WORK AND WAIT` is issued
   **And** the next sweep picks up the performance and re-dispatches the reset step(s)

3. **Given** a performance in STATUS = 'C' (COMPLETED) or 'X' (CANCELLED)
   **When** `cancel_performance( iv_perf_uuid )` is called
   **Then** `ZCX_EN_ORCH_ERROR` is raised with textid `STEP_RESTART_FAILED` and `MV_PERF_UUID` populated (no commit, no DB changes)

4. **Given** a performance in STATUS = 'C' (COMPLETED) or 'X' (CANCELLED)
   **When** `restart_performance( iv_perf_uuid )` is called
   **Then** `ZCX_EN_ORCH_ERROR` is raised with textid `STEP_RESTART_FAILED` and `MV_PERF_UUID` populated

5. **Given** a performance UUID that does not exist in `ZEN_ORCH_PERF`
   **When** `cancel_performance` or `restart_performance` is called
   **Then** `ZCX_EN_ORCH_ERROR` is raised with textid `STEP_RESTART_FAILED`

6. **Given** `cancel_performance` is called for a RUNNING performance
   **When** `adapter->cancel( )` raises `ZCX_EN_ORCH_ERROR` for one RUNNING step
   **Then** the exception is caught and logged; the cancel proceeds for remaining steps; all non-COMPLETED steps are still cancelled; the performance status is still set to 'X'

7. **Given** `ZCL_EN_ORCH_ENGINE` is activated after this story
   **Then** the class activates without errors and abaplint passes (≤120-char line limit)

8. **Given** `ZCL_EN_ORCH_HEALTH_CHK_QUERY` has health check tests added
   **When** the ABAP Unit tests are executed
   **Then** `CANCEL_PERFORMANCE` and `RESTART_PERFORMANCE` health checks report GREEN

## Tasks / Subtasks

- [x] Implement `cancel_performance` in `ZCL_EN_ORCH_ENGINE` (AC: #1, #3, #5, #6)
  - [x] DEFINITION: already declared — implement the existing stub body
  - [x] IMPLEMENTATION: `SELECT SINGLE status FROM zen_orch_perf WHERE perf_uuid = @iv_perf_uuid INTO @DATA(lv_status)` (pre-declared at top)
  - [x] Guard: if `sy-subrc <> 0` OR `lv_status` is not 'R' or 'P' → RAISE EXCEPTION `STEP_RESTART_FAILED` with `mv_perf_uuid`, `mv_detail` (AC #3, #5)
  - [x] Load all RUNNING steps: `SELECT * FROM zen_orch_p_step WHERE perf_uuid = @iv_perf_uuid AND status = @gc_status-running INTO TABLE @lt_running_steps`
  - [x] For each RUNNING step: call `zcl_en_orch_adapter_factory=>create( ls_step-adapter_type )->cancel( ls_step-work_unit_handle )` inside TRY/CATCH; log exception on error, continue (AC #6)
  - [x] `UPDATE zen_orch_p_step SET status = @gc_status-cancelled WHERE perf_uuid = @iv_perf_uuid AND status <> @gc_status-completed` (all non-COMPLETED steps)
  - [x] `UPDATE zen_orch_perf SET status = @gc_status-cancelled WHERE perf_uuid = @iv_perf_uuid`
  - [x] `COMMIT WORK AND WAIT`

- [x] Implement `restart_performance` in `ZCL_EN_ORCH_ENGINE` (AC: #2, #4, #5)
  - [x] DEFINITION: already declared — implement the existing stub body
  - [x] IMPLEMENTATION: `SELECT SINGLE status FROM zen_orch_perf WHERE perf_uuid = @iv_perf_uuid INTO @DATA(lv_status)` (pre-declared at top)
  - [x] Guard: if `sy-subrc <> 0` OR `lv_status <> gc_status-failed` → RAISE EXCEPTION `STEP_RESTART_FAILED` with `mv_perf_uuid`, `mv_detail`
  - [x] Reset all FAILED steps: `UPDATE zen_orch_p_step SET status = @gc_status-pending, work_unit_handle = space WHERE perf_uuid = @iv_perf_uuid AND status = @gc_status-failed`
  - [x] Set performance to RUNNING: `UPDATE zen_orch_perf SET status = @gc_status-running WHERE perf_uuid = @iv_perf_uuid`
  - [x] Verify `sy-subrc = 0` after the perf UPDATE, RAISE on mismatch
  - [x] `COMMIT WORK AND WAIT`

- [x] Add health check `check_cancel_performance` to `ZCL_EN_ORCH_HEALTH_CHK_QUERY` (AC: #8)
  - [x] DEFINITION: `METHODS check_cancel_performance` in PRIVATE SECTION
  - [x] Add `check_cancel_performance( ).` to `build_rows` dispatch
  - [x] IMPLEMENTATION:
    - [x] Pre-requisite: rely on test score `gc_test_score_id` already in DB (created by `check_create_performance`)
    - [x] Call `create_performance( gc_test_score_id )` to get a fresh PENDING perf UUID
    - [x] Call `cancel_performance( lv_perf_uuid )`
    - [x] `SELECT SINGLE status FROM zen_orch_perf WHERE perf_uuid = @lv_perf_uuid INTO @lv_status`
    - [x] Assert `lv_status = gc_status-cancelled`; append GREEN/RED row with `capabilityid = 'CANCEL_PERFORMANCE'`
  - [x] Add `DELETE FROM zen_orch_perf WHERE perf_uuid = @lv_perf_uuid` and matching `DELETE FROM zen_orch_p_step` to `cleanup_test_data`

- [x] Add health check `check_restart_performance` to `ZCL_EN_ORCH_HEALTH_CHK_QUERY` (AC: #8)
  - [x] DEFINITION: `METHODS check_restart_performance` in PRIVATE SECTION
  - [x] Add `check_restart_performance( ).` to `build_rows` dispatch
  - [x] IMPLEMENTATION:
    - [x] Call `create_performance( gc_test_score_id )` to get a PENDING perf UUID
    - [x] Manually `UPDATE zen_orch_perf SET status = @gc_status-failed WHERE perf_uuid = @lv_perf_uuid`
    - [x] Manually `UPDATE zen_orch_p_step SET status = @gc_status-failed WHERE perf_uuid = @lv_perf_uuid` (at least one step)
    - [x] Call `restart_performance( lv_perf_uuid )`
    - [x] `SELECT SINGLE status FROM zen_orch_perf WHERE perf_uuid = @lv_perf_uuid INTO @lv_status`
    - [x] Assert `lv_status = gc_status-running`
    - [x] Also verify no FAILED steps remain: `SELECT COUNT(*) FROM zen_orch_p_step WHERE perf_uuid = @lv_perf_uuid AND status = @gc_status-failed INTO @lv_failed_count`
    - [x] Assert `lv_failed_count = 0`; append GREEN/RED row with `capabilityid = 'RESTART_PERFORMANCE'`
  - [x] Add cleanup to `cleanup_test_data`

- [x] Add unit test methods to `ltcl_health_chk_query` test class (AC: #8)
  - [x] `METHODS test_cancel_performance FOR TESTING` → calls `run_test( 'CANCEL_PERFORMANCE' )`
  - [x] `METHODS test_restart_performance FOR TESTING` → calls `run_test( 'RESTART_PERFORMANCE' )`

## Dev Notes

### Current State of the Stubs

Both `cancel_performance` and `restart_performance` are already declared with correct signatures in `zcl_en_orch_engine.clas.abap` (lines 80–98). They contain only a comment stub:
```abap
METHOD cancel_performance.
  " Stub — implemented in Story zen-5-3
ENDMETHOD.

METHOD restart_performance.
  " Stub — implemented in Story zen-5-3
ENDMETHOD.
```

**Only the method bodies need to be filled — no DEFINITION section changes are needed.**

### `cancel_performance` Implementation Skeleton

```abap
METHOD cancel_performance.
  DATA lv_status       TYPE zen_orch_de_status.
  DATA lt_running_steps TYPE STANDARD TABLE OF zen_orch_s_perf_step WITH DEFAULT KEY.
  DATA ls_step          TYPE zen_orch_s_perf_step.
  DATA lo_adapter       TYPE REF TO zif_en_orch_adapter.
  DATA lx_cancel        TYPE REF TO zcx_en_orch_error.

  " T1.1 — Read current performance status
  SELECT SINGLE status
    FROM zen_orch_perf
    WHERE perf_uuid = @iv_perf_uuid
    INTO @lv_status.

  " T1.2 — Guard: only RUNNING or PENDING performances can be cancelled (AC #3, #5)
  IF sy-subrc <> 0
    OR ( lv_status <> gc_status-running AND lv_status <> gc_status-pending ).
    RAISE EXCEPTION TYPE zcx_en_orch_error
      EXPORTING
        textid       = zcx_en_orch_error=>step_restart_failed
        mv_perf_uuid = iv_perf_uuid
        mv_detail    = |cancel_performance: perf not RUNNING/PENDING (status={ lv_status })|.
  ENDIF.

  " T1.3 — Best-effort cancel of all RUNNING adapter work units (AC #6)
  SELECT *
    FROM zen_orch_p_step
    WHERE perf_uuid = @iv_perf_uuid
      AND status    = @gc_status-running
    INTO TABLE @lt_running_steps.

  LOOP AT lt_running_steps INTO ls_step.
    TRY.
        lo_adapter = zcl_en_orch_adapter_factory=>create( ls_step-adapter_type ).
        lo_adapter->cancel( ls_step-work_unit_handle ).
      CATCH zcx_en_orch_error INTO lx_cancel.
        " Best-effort: log but do not abort (AC #6)
        TRY.
            get_logger( )->log_exception( lx_cancel ).
          CATCH zcx_en_orch_error.                              "#EC NEEDED
        ENDTRY.
    ENDTRY.
  ENDLOOP.

  " T1.4 — Mark all non-COMPLETED steps as CANCELLED (AC #1)
  UPDATE zen_orch_p_step
    SET status      = @gc_status-cancelled
    WHERE perf_uuid = @iv_perf_uuid
      AND status    <> @gc_status-completed.

  " T1.5 — Mark the performance CANCELLED (AC #1)
  UPDATE zen_orch_perf
    SET status      = @gc_status-cancelled
    WHERE perf_uuid = @iv_perf_uuid.

  " T1.6 — Independent commit (AC #1)
  COMMIT WORK AND WAIT.
ENDMETHOD.
```

### `restart_performance` Implementation Skeleton

```abap
METHOD restart_performance.
  DATA lv_status TYPE zen_orch_de_status.

  " T1.1 — Read current performance status
  SELECT SINGLE status
    FROM zen_orch_perf
    WHERE perf_uuid = @iv_perf_uuid
    INTO @lv_status.

  " T1.2 — Guard: only FAILED performances can be restarted (AC #4, #5)
  IF sy-subrc <> 0 OR lv_status <> gc_status-failed.
    RAISE EXCEPTION TYPE zcx_en_orch_error
      EXPORTING
        textid       = zcx_en_orch_error=>step_restart_failed
        mv_perf_uuid = iv_perf_uuid
        mv_detail    = |restart_performance: perf not FAILED (status={ lv_status })|.
  ENDIF.

  " T1.3 — Reset all FAILED steps to PENDING and clear their work unit handle (AC #2)
  UPDATE zen_orch_p_step
    SET status           = @gc_status-pending,
        work_unit_handle = @space
    WHERE perf_uuid      = @iv_perf_uuid
      AND status         = @gc_status-failed.

  " T1.4 — Set performance STATUS to RUNNING (AC #2)
  UPDATE zen_orch_perf
    SET status      = @gc_status-running
    WHERE perf_uuid = @iv_perf_uuid.
  IF sy-subrc <> 0.
    RAISE EXCEPTION TYPE zcx_en_orch_error
      EXPORTING
        textid       = zcx_en_orch_error=>step_restart_failed
        mv_perf_uuid = iv_perf_uuid
        mv_detail    = 'restart_performance: perf row vanished before status reset to RUNNING'.
  ENDIF.

  " T1.5 — Independent commit (AC #2)
  COMMIT WORK AND WAIT.
ENDMETHOD.
```

### Table Reference: Performance Step Table

Steps are stored in `zen_orch_p_step` (short alias — note this vs. older references):

| Field | Type | Notes |
|-------|------|-------|
| `MANDT` | MANDT | Client (key) |
| `PERF_UUID` | `ZEN_ORCH_PERF_UUID` | Performance UUID (key) |
| `SCORE_SEQ` | `ZEN_ORCH_DE_SCORE_SEQ` | Sequence (key) |
| `LOOP_ITERATION` | `ZEN_ORCH_DE_LOOP_ITER` | Iteration (key, default 0) |
| `ELEM_TYPE` | `ZEN_ORCH_DE_ELEM_TYPE` | STEP / GATE / LOOP / END_LOOP |
| `REF_ID` | `ZEN_ORCH_DE_REF_ID` | Group key for gate/loop |
| `ADAPTER_TYPE` | `ZEN_ORCH_DE_ADAPTER_TYPE` | e.g. 'MOCK' |
| `PARAMS_JSON` | `ZEN_ORCH_DE_PARAMS_JSON` | Step-level params |
| `STATUS` | `ZEN_ORCH_DE_STATUS` | P/R/C/F/X/B |
| `WORK_UNIT_HANDLE` | `ZEN_ORCH_WU_HANDLE` | Opaque adapter handle (clear on restart) |
| `IS_BREAKPOINT` | `ABAP_BOOL` | Breakpoint gate flag |

> **DB table name:** `zen_orch_p_step` — this is the short form used in the existing engine code (see `advance_performance`, `dispatch_step`, `poll_step_status`). Do NOT use `zen_orch_perf_step` — it will fail syntax check.

### Health Check Pattern (from zen-5-2)

The health check class `ZCL_EN_ORCH_HEALTH_CHK_QUERY` follows a strict pattern:

1. **All DATA declarations at method top** — no inline `INTO DATA(...)` inside TRY/CATCH, IF, LOOP, or CASE branches (abaplint `no_inline_in_optional_branches`)
2. **Pre-declared exception variables** — `DATA lx TYPE REF TO zcx_en_orch_error.` at method top, then `CATCH zcx_en_orch_error INTO lx`
3. **`build_rows` dispatch** — new `check_*` calls are appended to the existing method dispatch sequence
4. **`cleanup_test_data`** — any test rows inserted must be deleted here (DELETE by known test keys)
5. **Row append pattern:**
```abap
GET TIME STAMP FIELD DATA(lt1).
" ... test logic ...
GET TIME STAMP FIELD DATA(lt2).
DATA(lv_elapsed) = CONV i( ( lt2 - lt1 ) * 1000 ).
APPEND VALUE #(
  capabilityid      = 'CANCEL_PERFORMANCE'
  description       = 'Cancel a running performance'
  status            = COND #( WHEN lv_status = gc_status-cancelled THEN gc_green ELSE gc_red )
  statuscriticality = status_to_criticality( ... )
  message           = |Status after cancel: { lv_status }|
  durationms        = lv_elapsed
  testlogic         = 'create_performance → cancel_performance → verify CANCELLED'
) TO mt_rows.
```
6. **Unit test method:** `METHODS test_cancel_performance FOR TESTING` → calls `run_test( 'CANCEL_PERFORMANCE' )`

### Key abaplint Rules (mandatory — violations fail CI)

| Rule | Detail |
|------|--------|
| `definitions_top` | ALL `DATA` / `FIELD-SYMBOLS` declarations at method top, before any logic |
| `no_inline_in_optional_branches` | No `INTO DATA(...)` inside TRY/CATCH, IF, LOOP, CASE branches |
| Line length | ≤120 chars strictly; wrap long string templates |
| `##NO_HANDLER` / `"#EC NEEDED"` | Empty CATCH: `##NO_HANDLER`; intentional non-propagation: `"#EC NEEDED"` |
| ABAP-Doc | `"!` only directly before a `METHODS` or `CLASS-METHODS` declaration; never inside IMPLEMENTATION |
| `CONSTANTS: BEGIN OF` | Preceding comment must use `"` not `"!` |

**Watch-outs from zen-5-2:**
- Hoist ALL `DATA` declarations to method top — including `DATA lo_adapter`, `DATA lx_cancel` — before any `TRY` block
- `COMMIT WORK AND WAIT` — keep all commits at the end of the public method; **never inside helpers**
- `gc_status-cancelled` = `'X'`, `gc_status-failed` = `'F'`, `gc_status-pending` = `'P'`, `gc_status-running` = `'R'`
- `UPDATE zen_orch_p_step ... SET work_unit_handle = @space` — use `@space` (host variable) not `space` (literal) in OpenSQL
- `get_logger( )->log_exception( lx )` pattern — `get_logger()` is a public method on `ZCL_EN_ORCH_ENGINE` (added in zen-5-2 patch); no need to call `zcl_en_orch_logger=>create()` directly in `cancel_performance` / `restart_performance`

### Adapter Interface Cancel Method

```abap
" ZIF_EN_ORCH_ADAPTER
cancel( IMPORTING iv_handle TYPE zen_orch_wu_handle RAISING zcx_en_orch_error )
```
The adapter's `cancel` is **best-effort** (AC #6). The caller (`cancel_performance`) must catch `ZCX_EN_ORCH_ERROR` per step and continue. The mock adapter's `cancel` is a no-op that does not raise.

### Status Terminal Check Logic

| Status | Can cancel? | Can restart? |
|--------|-------------|--------------|
| P (PENDING) | YES | NO |
| R (RUNNING) | YES | NO |
| F (FAILED) | NO | YES |
| C (COMPLETED) | NO (raise) | NO (raise) |
| X (CANCELLED) | NO (raise) | NO (raise) |
| B (PAUSED) | Implementation note: PAUSED is not explicitly covered in the ACs; treat as non-cancellable (raise) to keep scope clear — confirmed by epic spec which only mentions R and P for cancel. |

### Project Structure Notes

**Files to modify** (all in `src/` of `cz.en.orch` repository):

| File | Change |
|------|--------|
| `zcl_en_orch_engine.clas.abap` | Replace stub bodies of `cancel_performance` and `restart_performance` |
| `zcl_en_orch_health_chk_query.clas.abap` | Add `check_cancel_performance` and `check_restart_performance` methods; update `build_rows` and `cleanup_test_data` |
| `zcl_en_orch_health_chk_query.clas.testclasses.abap` | Add `test_cancel_performance` and `test_restart_performance` unit test methods |

**No new files required.** No DDIC changes.

**Architecture layer:** Layer 10 (`ZCL_EN_ORCH_ENGINE`) — no layer change. Health check query is layer 10.

### References

- Story requirements: [Source: `_bmad-output/planning-artifacts/epics.md#Story 5.3`]
- Engine current state: [Source: `cz.en.orch/src/zcl_en_orch_engine.clas.abap`] — stubs at lines 316–323
- Performance table schema: [Source: `_bmad-output/planning-artifacts/architecture.md#Table Key Design`]
- Performance step table: `zen_orch_p_step` — used throughout existing engine code (advance_performance line 386, dispatch_step line 549)
- Adapter cancel signature: [Source: `_bmad-output/planning-artifacts/epics.md#Story 3.1`]
- COMMIT strategy: [Source: `_bmad-output/planning-artifacts/architecture.md#9. COMMIT Placement`] — COMMIT only in public methods
- Status constants: [Source: `cz.en.orch/src/zcl_en_orch_engine.clas.abap` lines 22–35]
- Health check pattern: [Source: `cz.en.orch/src/zcl_en_orch_health_chk_query.clas.abap`]
- abaplint rules: [Source: `zen-5-2` Dev Notes § abaplint Rules]
- `resume_performance` (same guard/commit pattern): [Source: `cz.en.orch/src/zcl_en_orch_engine.clas.abap` lines 326–373]

## Constitution Compliance

| Principle | Application |
|-----------|-------------|
| **I — DDIC-First** | `zen_orch_perf` and `zen_orch_p_step` used as transparent table types; no local type definitions for DB structures; `zen_orch_s_perf_step` used as parameter type (already in DDIC) |
| **II — SAP Standards** | Method names `cancel_performance`, `restart_performance` ≤30 chars; ≤120-char lines; ABAP-Doc on all method declarations (already present); snake_case throughout |
| **V — Error Handling** | `ZCX_EN_ORCH_ERROR` raised with `STEP_RESTART_FAILED` for invalid state transitions; `ZCX_EN_ORCH_ERROR` from adapter `cancel()` caught per-step and logged — never propagated to caller |

## Dev Agent Record

### Agent Model Used

claude-sonnet-4.6 (github-copilot/claude-sonnet-4.6)

### Debug Log References

- Both `cancel_performance` and `restart_performance` were already declared in the engine DEFINITION — only stub METHOD bodies needed replacing.
- Health check class already had GREY placeholder rows for both capabilities in `build_rows` — replaced with real `check_*` method calls.
- `lv_logic` variable in `build_rows` (previously used only for the GREY stubs) is now unused but still declared at method top — harmless, not removed to avoid churn.
- `cancel_performance` uses `get_logger( )->log_exception( lx_cancel )` (engine's own logger wrapper) — no need to call `zcl_en_orch_logger=>create( )` directly.
- Health check pattern requires `MODIFY` (not `INSERT`) for idempotent score/step setup, `COMMIT WORK AND WAIT` before calling the method under test, and `cleanup_check_data( )` at the end of each check method.

### Completion Notes List

- `cancel_performance` implemented in `zcl_en_orch_engine.clas.abap`: guards against non-RUNNING/PENDING status, best-effort per-step adapter cancel with logging, bulk UPDATE of non-COMPLETED steps to CANCELLED, PERF status UPDATE to CANCELLED, COMMIT WORK AND WAIT.
- `restart_performance` implemented in `zcl_en_orch_engine.clas.abap`: guards against non-FAILED status, resets all FAILED steps to PENDING with cleared work_unit_handle, sets PERF to RUNNING, COMMIT WORK AND WAIT.
- `check_cancel_performance` and `check_restart_performance` added to `zcl_en_orch_health_chk_query.clas.abap` (DEFINITION + IMPLEMENTATION + `build_rows` Phase 10 dispatch + class header comment).
- `test_cancel_performance` and `test_restart_performance` METHODS declarations added to DEFINITION section and IMPLEMENTATION bodies added to `zcl_en_orch_health_chk_query.clas.testclasses.abap`.
- All abaplint rules observed: DATA declarations at method top, no inline INTO DATA() in optional branches, ≤120-char lines, `##NO_HANDLER` and `"#EC NEEDED` pragmas where needed.

### File List

| File | Repository | Change |
|------|-----------|--------|
| `src/zcl_en_orch_engine.clas.abap` | cz.en.orch | Replaced stub bodies of `cancel_performance` and `restart_performance` |
| `src/zcl_en_orch_health_chk_query.clas.abap` | cz.en.orch | Added `check_cancel_performance` and `check_restart_performance` (DEFINITION + IMPLEMENTATION + `build_rows` Phase 10 + class header) |
| `src/zcl_en_orch_health_chk_query.clas.testclasses.abap` | cz.en.orch | Added `test_cancel_performance` and `test_restart_performance` METHODS declarations and IMPLEMENTATION bodies |

### Review Findings

- [x] [Review][Decision] F3: Schedule changed_at is date-only (AEDAT) — **fixed**: added `LAST_FIRED_TS TYPE TIMESTAMPL` to ZEN_ORCH_SCHED; `is_schedule_due` now guards against re-fire within same UTC minute; `check_schedules` records exact timestamp and commits independently
- [x] [Review][Decision] F8: Diff contains out-of-scope implementations from zen-4-7 and zen-5-2 — **dismissed**: accepted as combined commit covering zen-4-7, zen-5-2, zen-5-3
- [x] [Review][Decision] F14: restart_performance only resets FAILED steps — **dismissed**: intentional design; CANCELLED+FAILED mixed state is considered invalid input; documented via F1 guard
- [x] [Review][Patch] F1: restart_performance does not check sy-dbcnt after step UPDATE — **fixed**: added `IF sy-dbcnt = 0` guard raising STEP_RESTART_FAILED with 'inconsistent state' detail [`zcl_en_orch_engine.clas.abap` T1.3]
- [x] [Review][Patch] F2: get_logger() creates a new logger instance on every call — **fixed**: `mo_logger` instance attribute added; lazy-initialised once per engine instance (singleton per engine) [`zcl_en_orch_engine.clas.abap` get_logger method]
- [x] [Review][Patch] F5: schedule timestamp UPDATE not guaranteed committed — **fixed**: added `COMMIT WORK AND WAIT` immediately after the `UPDATE zen_orch_sched` in `check_schedules` [`zcl_en_orch_engine.clas.abap` check_schedules]
- [x] [Review][Patch] F6: dead TRY/CATCH around check_schedules in sweep_all — **fixed**: removed TRY/CATCH block and unused `lx_sched_err` variable [`zcl_en_orch_engine.clas.abap` sweep_all T0]
- [x] [Review][Patch] F7: get_logger() in PUBLIC SECTION — **fixed**: moved declaration from PUBLIC SECTION to PRIVATE SECTION [`zcl_en_orch_engine.clas.abap` line ~118]
- [x] [Review][Patch] F9: lt_t[ lv_m ] uncaught for lv_m outside 1–12 — **fixed**: added `IF lv_m < 1 OR lv_m > 12` guard before table access, returns false on malformed date [`zcl_en_orch_engine.clas.abap` is_schedule_due]
- [x] [Review][Patch] F10: CONV i( iv_field+2 ) uncaught in */N branch — **fixed**: wrapped in TRY/CATCH cx_sy_conversion_no_number, returns false on malformed token [`zcl_en_orch_engine.clas.abap` matches_cron_field]
- [x] [Review][Patch] F11: SPLIT on cron with double spaces silently never fires — **fixed**: added whitespace normalization loop before SPLIT (collapses consecutive spaces) [`zcl_en_orch_engine.clas.abap` is_schedule_due]
- [x] [Review][Patch] F15: PAUSED steps cancelled in DB without adapter->cancel — **fixed**: clarified by design comment: PAUSED steps are GATE elements with no work unit handle; SELECT for adapter->cancel correctly limits to RUNNING steps [`zcl_en_orch_engine.clas.abap` cancel_performance T1.3]
- [x] [Review][Defer] F4: Schedule snapshot staleness (deactivated mid-loop) [check_schedules] — deferred, pre-existing design limitation; requires ENQUEUE lock or row-level re-read
- [x] [Review][Defer] F12: check_cancel_performance only tests PENDING → CANCELLED path [check_cancel_performance] — deferred, partial health check coverage accepted; full scenario test requires a non-trivial mock setup
- [x] [Review][Defer] F13: check_restart_performance does not invoke sweep_all after restart [check_restart_performance] — deferred, this would be an integration test; unit health check scope is status assertions only

### Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-06 | Initial implementation of zen-5-3: cancel_performance, restart_performance, health checks, unit tests | claude-sonnet-4.6 |
| 2026-04-06 | Code review complete: 9 patches applied (F1–F3, F5–F7, F9–F11, F15), 2 decisions dismissed (F8, F14), 3 deferred (F4, F12, F13); DDIC: last_fired_ts added to ZEN_ORCH_SCHED | claude-sonnet-4.6 |
