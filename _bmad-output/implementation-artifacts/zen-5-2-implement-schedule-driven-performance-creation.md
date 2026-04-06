---
story_id: "zen-5-2"
epic_id: "zen-epic-5"
title: "Implement schedule-driven performance creation"
target_repository: "cz.en.orch"
depends_on:
  - "zen-5-1"
constitution_principles:
  - "Principle I - DDIC-First"
  - "Principle II - SAP Standards"
  - "Principle III - Consult SAP Docs"
  - "Principle V - Error Handling"
status: "done"
created: "2026-04-06"
---

# Story zen-5-2: Implement Schedule-Driven Performance Creation

Status: done

## Story

As an operator,
I want active schedule entries in `ZEN_ORCH_SCHED` to automatically trigger new performances,
so that recurring orchestration scenarios (e.g., monthly allocation runs) start without manual intervention.

## Acceptance Criteria

1. **Given** a row exists in `ZEN_ORCH_SCHED` with `ACTIVE = 'X'`, a valid `SCORE_ID`, and a `CRON_EXPR` indicating the current time is due  
   **When** `sweep_all` is called  
   **Then** `check_schedules` is called at the start of the sweep (before advancing performances)  
   **And** a new performance is created for that schedule entry via `create_performance`

2. **Given** `check_schedules` has fired and created a performance for schedule entry S  
   **When** the same sweep cycle continues  
   **Then** the same schedule does not trigger again within the same sweep execution

3. **Given** a schedule entry has `ACTIVE = space` (inactive)  
   **When** `check_schedules` processes it  
   **Then** no performance is created for that entry

4. **Given** a schedule entry points to a `SCORE_ID` that no longer exists in `ZEN_ORCH_SCORE`  
   **When** `check_schedules` processes it  
   **Then** the `ZCX_EN_ORCH_ERROR` (SCORE_NOT_FOUND) is caught, an error is logged via the engine logger, and the schedule entry is skipped — the rest of the sweep continues unaffected

5. **Given** a schedule entry whose `CRON_EXPR` does NOT indicate the current time as due  
   **When** `check_schedules` processes it  
   **Then** no performance is created for that entry

6. **Given** `check_schedules` is called  
   **When** the CRON evaluation determines a schedule is due  
   **Then** the schedule row's `CHANGED_AT` field is updated to `sy-datum` and `CHANGED_BY` to `sy-uname` to record the last-triggered timestamp  
   **And** this update is committed as part of the normal per-performance `COMMIT WORK AND WAIT` sequence in `sweep_all`

7. **Given** `ZCL_EN_ORCH_ENGINE` is activated after this story  
   **Then** the class activates without errors and abaplint passes (≤120-char line limit)

## Tasks / Subtasks

- [x] Add `check_schedules` private method to `ZCL_EN_ORCH_ENGINE` (AC: #1, #2, #3, #4, #5, #6)
  - [x] DEFINITION: declare `check_schedules` in PRIVATE SECTION with ABAP-Doc (raising `zcx_en_orch_error` is NOT raised — errors are caught internally)
  - [x] IMPLEMENTATION: `SELECT * FROM zen_orch_sched WHERE active = 'X'` to load all active schedule rows
  - [x] For each schedule row: call `is_schedule_due` helper; skip if not due
  - [x] For due schedules: call `create_performance( iv_score_id = ls_sched-score_id iv_params_json = ls_sched-params_json )`
  - [x] Catch `ZCX_EN_ORCH_ERROR` from `create_performance` per schedule entry; log warning and continue (AC #4)
  - [x] On success: UPDATE `zen_orch_sched SET changed_at = sy-datum changed_by = sy-uname WHERE schedule_id = ...` (AC #6)

- [x] Add `is_schedule_due` private helper method to `ZCL_EN_ORCH_ENGINE` (AC: #5)
  - [x] DEFINITION: `IMPORTING iv_cron_expr TYPE zen_orch_de_cron_expr RETURNING VALUE(rv_due) TYPE abap_bool`
  - [x] IMPLEMENTATION: parse CRON expression (see Dev Notes for supported format and parsing approach)
  - [x] Return `abap_true` if current `sy-datum` / `sy-uzeit` matches the CRON schedule; `abap_false` otherwise

- [x] Modify `sweep_all` in `ZCL_EN_ORCH_ENGINE` to call `check_schedules` at the start (AC: #1)
  - [x] Insert `check_schedules( ).` as the very first statement in `sweep_all`, before the `SELECT` of active performances
  - [x] Wrap in TRY/CATCH `zcx_en_orch_error` — any unexpected error from `check_schedules` is logged but does not abort the sweep
  - [x] Note: the actual commit of schedule updates happens as part of the first subsequent `COMMIT WORK AND WAIT` in the loop, or can be an explicit `COMMIT WORK AND WAIT` after `check_schedules` if no performances are active

- [x] Add health check for `check_schedules` to `ZCL_EN_ORCH_HEALTH_CHK_QUERY` (AC: #1, #2, #3, #4)
  - [x] Add `check_schedule_trigger` private method
  - [x] Insert test row into `ZEN_ORCH_SCHED` with `ACTIVE = 'X'`, `SCORE_ID = gc_test_score_id`, `CRON_EXPR` = a value the helper evaluates as due (e.g., `'* * * * *'` — always due)
  - [x] Call `zcl_en_orch_engine=>get_instance( )->sweep_all( )` (or directly call a testable surface) and verify a performance row exists in `ZEN_ORCH_PERF` for `gc_test_score_id`
  - [x] Verify an inactive schedule does NOT create a performance
  - [x] Clean up test data in `cleanup_test_data`
  - [x] Add `check_schedule_trigger` to the `build_rows` method dispatch

## Dev Notes

### CRON Expression Format and Evaluation

**Decision:** The CRON field (`ZEN_ORCH_CRON_EXPR`, CHAR 100) holds a **simplified 5-field cron** expression: `minute hour day-of-month month day-of-week`. Each field supports:
- `*` — matches any value
- A single integer — matches that exact value
- A `*/N` step — matches when the value modulo N equals 0

**Examples:**
```
* * * * *          → every minute (always due)
0 * * * *          → every hour at minute 0
0 6 * * 1          → every Monday at 06:00
0 0 1 * *          → first of every month at midnight
0 6 1 1 *          → January 1st at 06:00
```

**Evaluation approach** (from `sy-datum` and `sy-uzeit`):

```abap
METHOD is_schedule_due.
  DATA lv_minute    TYPE i.
  DATA lv_hour      TYPE i.
  DATA lv_day       TYPE i.
  DATA lv_month     TYPE i.
  DATA lv_dow       TYPE i.
  DATA lt_fields    TYPE STANDARD TABLE OF string WITH DEFAULT KEY.
  DATA lv_field     TYPE string.

  " Extract date/time parts from sy-datum (YYYYMMDD) and sy-uzeit (HHMMSS)
  lv_minute = CONV i( sy-uzeit+2(2) ).
  lv_hour   = CONV i( sy-uzeit(2) ).
  lv_day    = CONV i( sy-datum+6(2) ).
  lv_month  = CONV i( sy-datum+4(2) ).

  " Day of week: 1=Monday ... 7=Sunday using ABAP function
  DATA lv_weekday TYPE sy-fdayw.
  CALL FUNCTION 'DATE_GET_WEEK'
    EXPORTING
      date              = sy-datum
    IMPORTING
      week              = DATA(lv_week)
    EXCEPTIONS
      OTHERS            = 1.
  " Derive DOW from date calculation — see full helper below.

  " Split cron into 5 tokens
  SPLIT iv_cron_expr AT space INTO TABLE lt_fields.
  IF lines( lt_fields ) <> 5.
    rv_due = abap_false.
    RETURN.
  ENDIF.

  " Check each field against the schedule value
  rv_due = xsdbool(
    matches_cron_field( iv_field = lt_fields[ 1 ] iv_value = lv_minute )  AND
    matches_cron_field( iv_field = lt_fields[ 2 ] iv_value = lv_hour )    AND
    matches_cron_field( iv_field = lt_fields[ 3 ] iv_value = lv_day )     AND
    matches_cron_field( iv_field = lt_fields[ 4 ] iv_value = lv_month )   AND
    matches_cron_field( iv_field = lt_fields[ 5 ] iv_value = lv_dow )
  ).
ENDMETHOD.
```

**Day-of-week calculation** (no standard ABAP function gives DOW directly):

```abap
" Tomohiko Sakamoto algorithm adapted for ABAP
" Returns 1=Monday, 2=Tuesday, ... 7=Sunday
DATA lv_y TYPE i VALUE CONV i( sy-datum(4) ).
DATA lv_m TYPE i VALUE CONV i( sy-datum+4(2) ).
DATA lv_d TYPE i VALUE CONV i( sy-datum+6(2) ).
DATA lt_t TYPE STANDARD TABLE OF i WITH DEFAULT KEY
  INITIAL SIZE 12.
APPEND 0 TO lt_t. APPEND 3 TO lt_t. APPEND 2 TO lt_t.
APPEND 5 TO lt_t. APPEND 0 TO lt_t. APPEND 3 TO lt_t.
APPEND 5 TO lt_t. APPEND 1 TO lt_t. APPEND 4 TO lt_t.
APPEND 6 TO lt_t. APPEND 2 TO lt_t. APPEND 4 TO lt_t.
IF lv_m < 3. lv_y = lv_y - 1. ENDIF.
" DOW 0=Sunday...6=Saturday; convert to 1=Monday...7=Sunday
DATA(lv_dow_raw) = ( lv_y + lv_y / 4 - lv_y / 100 + lv_y / 400
                   + lt_t[ lv_m ] + lv_d ) MOD 7.
DATA(lv_dow)     = COND i(
  WHEN lv_dow_raw = 0 THEN 7   " Sunday → 7
  ELSE lv_dow_raw                " Monday=1 ... Saturday=6
).
```

**`matches_cron_field` helper:**

```abap
METHOD matches_cron_field.
  " iv_field: cron field string (*, integer, */N)
  " iv_value: current time component as integer
  IF iv_field = '*'.
    rv_match = abap_true.
    RETURN.
  ENDIF.
  IF iv_field CS '*/'.
    DATA(lv_step) = CONV i( iv_field+2 ).
    rv_match = xsdbool( lv_step > 0 AND iv_value MOD lv_step = 0 ).
    RETURN.
  ENDIF.
  rv_match = xsdbool( CONV i( iv_field ) = iv_value ).
ENDMETHOD.
```

> **Note for dev agent:** `DATE_GET_WEEK` is acceptable here for week calculation. However, the cleanest DOW approach is the pure arithmetic Sakamoto method above — it has zero FM dependency. Prefer arithmetic over the FM call.

### `check_schedules` Implementation Skeleton

```abap
METHOD check_schedules.
  DATA lo_logger      TYPE REF TO zif_en_orch_logger.
  DATA lt_schedules   TYPE STANDARD TABLE OF zen_orch_sched WITH DEFAULT KEY.
  DATA ls_sched       TYPE zen_orch_sched.

  " Load all active schedule entries
  SELECT *
    FROM zen_orch_sched
    WHERE active = 'X'
    INTO TABLE @lt_schedules.

  LOOP AT lt_schedules INTO ls_sched.
    " Check if this schedule is due based on CRON expression
    CHECK is_schedule_due( ls_sched-cron_expr ) = abap_true.

    " Trigger performance creation; catch and log errors per entry (AC #4)
    TRY.
        create_performance(
          iv_score_id    = ls_sched-score_id
          iv_params_json = ls_sched-params_json
        ).

        " Record trigger timestamp (AC #6)
        UPDATE zen_orch_sched
          SET changed_at = @sy-datum,
              changed_by = @sy-uname
          WHERE schedule_id = @ls_sched-schedule_id.

      CATCH zcx_en_orch_error INTO DATA(lx_sched).
        " Log and skip — one bad schedule must not abort the sweep (AC #4)
        TRY.
            lo_logger = zcl_en_orch_logger=>create( ).
            lo_logger->log_exception( lx_sched ).
          CATCH zcx_en_orch_error.                          "#EC NEEDED
        ENDTRY.
    ENDTRY.
  ENDLOOP.
ENDMETHOD.
```

### Modified `sweep_all` — Where to Insert `check_schedules`

Add `check_schedules( ).` as the **first action** in `sweep_all`, before loading active performances. The schedule-triggered performances become immediately visible to the subsequent SELECT of active performances (they are inserted as PENDING):

```abap
METHOD sweep_all.
  DATA lo_logger       TYPE REF TO zif_en_orch_logger.
  DATA ls_perf_uuid    TYPE zen_orch_perf_uuid.
  DATA lt_active_perfs TYPE STANDARD TABLE OF zen_orch_perf_uuid
                        WITH DEFAULT KEY.

  " T0 — Fire any due schedule entries before advancing existing performances
  TRY.
      check_schedules( ).
    CATCH zcx_en_orch_error INTO DATA(lx_sched_err).
      " Unexpected error from schedule check — log and continue sweep
      TRY.
          lo_logger = zcl_en_orch_logger=>create( ).
          lo_logger->log_exception( lx_sched_err ).
        CATCH zcx_en_orch_error.                            "#EC NEEDED
      ENDTRY.
  ENDTRY.

  " T1.1 — Load all active performance UUIDs (stateless: re-read DB each call)
  SELECT perf_uuid
    FROM zen_orch_perf
    WHERE status = @gc_status-pending
       OR status = @gc_status-running
    INTO TABLE @lt_active_perfs.

  " T1.2 — Advance each performance independently; catch failures per-performance
  LOOP AT lt_active_perfs INTO ls_perf_uuid.
    TRY.
        advance_performance( ls_perf_uuid ).
      CATCH zcx_en_orch_error INTO DATA(lx_err).
        UPDATE zen_orch_perf
          SET status      = @gc_status-failed
          WHERE perf_uuid = @ls_perf_uuid.
        TRY.
            lo_logger = zcl_en_orch_logger=>create( ).
            lo_logger->log_exception( lx_err ).
          CATCH zcx_en_orch_error.                          "#EC NEEDED
        ENDTRY.
    ENDTRY.
    COMMIT WORK AND WAIT.
  ENDLOOP.

  " If no active performances: commit schedule timestamp updates (AC #6)
  IF lt_active_perfs IS INITIAL.
    COMMIT WORK AND WAIT.
  ENDIF.
ENDMETHOD.
```

> **Critical:** The `COMMIT WORK AND WAIT` at the end handles the case where `check_schedules` updated `CHANGED_AT` but no active performances exist yet in this sweep cycle (e.g., schedule fires and creates a new PENDING performance that will be picked up on the **next** sweep). Without this commit, the timestamp update would be lost.

### New Private Method Declarations (PRIVATE SECTION additions)

Add to `ZCL_EN_ORCH_ENGINE` PRIVATE SECTION:

```abap
"! Check all active schedule entries and create due performances.
"! Errors per schedule entry are caught and logged — method never raises.
METHODS check_schedules.

"! Evaluate whether a CRON expression matches the current system time.
"! Supports: * (wildcard), integer (exact), */N (step).
METHODS is_schedule_due
  IMPORTING
    iv_cron_expr     TYPE zen_orch_de_cron_expr
  RETURNING
    VALUE(rv_due)    TYPE abap_bool.

"! Match a single CRON field token against an integer time component.
METHODS matches_cron_field
  IMPORTING
    iv_field         TYPE string
    iv_value         TYPE i
  RETURNING
    VALUE(rv_match)  TYPE abap_bool.
```

### Table Reference: `ZEN_ORCH_SCHED`

| Field | Type | Notes |
|-------|------|-------|
| `MANDT` | MANDT | Client (key) |
| `SCHEDULE_ID` | `ZEN_ORCH_DE_SCHEDULE_ID` | Natural key (CHAR 30) |
| `SCORE_ID` | `ZEN_ORCH_DE_SCORE_ID` | References `ZEN_ORCH_SCORE` |
| `PARAMS_JSON` | `ZEN_ORCH_DE_PARAMS_JSON` | Performance creation params |
| `CRON_EXPR` | `ZEN_ORCH_DE_CRON_EXPR` | CHAR 100 — 5-field cron string |
| `ACTIVE` | `ZEN_ORCH_DE_ACTIVE` | CHAR 1: `'X'` = active, `space` = inactive |
| `CREATED_BY` | ERNAM | Audit |
| `CREATED_AT` | ERDAT | Audit |
| `CHANGED_BY` | AENAM | Audit (updated on trigger) |
| `CHANGED_AT` | AEDAT | Audit (updated on trigger) |

### Health Check Addition: `check_schedule_trigger`

Add to `ZCL_EN_ORCH_HEALTH_CHK_QUERY`:

**DEFINITION** (in PRIVATE SECTION):
```abap
METHODS check_schedule_trigger.
```

**Add to `build_rows` dispatch** (after `check_resume_performance`):
```abap
check_schedule_trigger( ).
```

**IMPLEMENTATION:**
```abap
METHOD check_schedule_trigger.
  DATA lo_engine  TYPE REF TO zcl_en_orch_engine.
  DATA lv_perf_uuid TYPE zen_orch_perf_uuid.
  DATA lv_count     TYPE i.
  DATA lv_t1        TYPE i.
  DATA lv_t2        TYPE i.
  DATA lv_elapsed   TYPE i.

  " Pre-requisite: test score must exist (created by check_create_performance)
  CHECK mv_filter_test_id IS INITIAL
     OR mv_filter_test_id = 'SCHEDULE_TRIGGER'.

  " Insert an active test schedule pointing at the test score
  DELETE FROM zen_orch_sched WHERE schedule_id = 'ZHEALTH_CHK_SCHED'.
  INSERT zen_orch_sched FROM @( VALUE #(
    schedule_id = 'ZHEALTH_CHK_SCHED'
    score_id    = gc_test_score_id
    params_json = '{}'
    cron_expr   = '* * * * *'   " always due
    active      = 'X'
    created_by  = sy-uname
    created_at  = sy-datum
  ) ).

  GET TIME STAMP FIELD DATA(lt1).
  TRY.
      lo_engine = zcl_en_orch_engine=>get_instance( ).

      " Isolated check: call check_schedules via sweep_all
      " (Performances created in prior checks are already COMPLETED — won't interfere)
      lo_engine->sweep_all( ).

      " Verify that a new PENDING/RUNNING/COMPLETED performance was created
      " for gc_test_score_id after the schedule triggered
      SELECT COUNT( * )
        FROM zen_orch_perf
        WHERE score_id = @gc_test_score_id
        INTO @lv_count.

      GET TIME STAMP FIELD DATA(lt2).
      lv_elapsed = CONV i( ( lt2 - lt1 ) * 1000 ).

      IF lv_count > 0.
        " Verify inactive schedule does NOT trigger: insert inactive sched
        DELETE FROM zen_orch_sched WHERE schedule_id = 'ZHEALTH_CHK_SCHED_INACT'.
        INSERT zen_orch_sched FROM @( VALUE #(
          schedule_id = 'ZHEALTH_CHK_SCHED_INACT'
          score_id    = gc_test_score_id
          params_json = '{}'
          cron_expr   = '* * * * *'
          active      = space   " inactive
          created_by  = sy-uname
          created_at  = sy-datum
        ) ).
        DATA(lv_perf_before) = lv_count.
        lo_engine->sweep_all( ).
        SELECT COUNT( * )
          FROM zen_orch_perf
          WHERE score_id = @gc_test_score_id
          INTO @DATA(lv_perf_after).
        " Inactive schedule should NOT have created another performance
        " (active schedule 'always due' will fire again — so count may increase by 1 only)
        APPEND VALUE #(
          capabilityid      = 'SCHEDULE_TRIGGER'
          description       = 'Schedule-driven performance creation'
          status            = COND #(
            WHEN lv_perf_after > lv_perf_before THEN gc_red   " inactive fired (wrong)
            ELSE gc_green )
          statuscriticality = status_to_criticality(
            COND #( WHEN lv_perf_after > lv_perf_before THEN gc_red ELSE gc_green ) )
          message           = |Perfs before={ lv_perf_before } after={ lv_perf_after }|
          durationms        = lv_elapsed
          testlogic         = 'sweep_all fires active sched; inactive sched does not fire'
        ) TO mt_rows.
      ELSE.
        APPEND VALUE #(
          capabilityid      = 'SCHEDULE_TRIGGER'
          description       = 'Schedule-driven performance creation'
          status            = gc_red
          statuscriticality = gc_crit_red
          message           = 'check_schedules did not create any performance'
          durationms        = lv_elapsed
          testlogic         = 'sweep_all fires active sched; no perf created'
        ) TO mt_rows.
      ENDIF.

    CATCH zcx_en_orch_error INTO DATA(lx).
      GET TIME STAMP FIELD DATA(lt3).
      APPEND VALUE #(
        capabilityid      = 'SCHEDULE_TRIGGER'
        description       = 'Schedule-driven performance creation'
        status            = gc_red
        statuscriticality = gc_crit_red
        message           = lx->get_text( )
        durationms        = CONV i( ( lt3 - lt1 ) * 1000 )
        testlogic         = 'sweep_all fires active sched; exception raised'
      ) TO mt_rows.
  ENDTRY.

  " Clean up test schedules
  DELETE FROM zen_orch_sched WHERE schedule_id = 'ZHEALTH_CHK_SCHED'.
  DELETE FROM zen_orch_sched WHERE schedule_id = 'ZHEALTH_CHK_SCHED_INACT'.
ENDMETHOD.
```

**Also add to `cleanup_test_data`:**
```abap
DELETE FROM zen_orch_sched WHERE schedule_id = 'ZHEALTH_CHK_SCHED'.
DELETE FROM zen_orch_sched WHERE schedule_id = 'ZHEALTH_CHK_SCHED_INACT'.
```

### Project Structure Notes

**Files to modify** (all in `src/` of `cz.en.orch` repository):

| File | Change |
|------|--------|
| `zcl_en_orch_engine.clas.abap` | Add `check_schedules`, `is_schedule_due`, `matches_cron_field` methods; modify `sweep_all` |
| `zcl_en_orch_health_chk_query.clas.abap` | Add `check_schedule_trigger` method; update `build_rows` and `cleanup_test_data` |

**No new files required.** No DDIC changes — `ZEN_ORCH_SCHED` table already exists and is active.

**Architecture layer:** Layer 10 (`ZCL_EN_ORCH_ENGINE`) — no layer change. Health check query is layer 10 as well (depends on engine).

### abaplint Rules (from prior zen stories)

| Rule | Detail |
|------|--------|
| `definitions_top` | All `DATA` declarations at the top of each method — before any logic |
| `no_inline_in_optional_branches` | No inline `INTO DATA(...)` inside TRY/CATCH, IF, LOOP, CASE branches |
| Line length | ≤120 chars strictly; wrap long string templates and method chains |
| `##NO_HANDLER` | Required on empty CATCH blocks; use `"#EC NEEDED` when the CATCH is intentionally non-empty but harmless |
| ABAP-Doc placement | `"!` only directly before a `METHODS` or `CLASS-METHODS` declaration; never inside IMPLEMENTATION body |
| `CONSTANTS: BEGIN OF` | Preceding comment must use `"` not `"!` |

**Specific watch-outs from zen-5-1:**

- Hoist ALL `DATA` declarations to method top — including `DATA lo_logger` — before any `TRY` block
- Use `CONV i( ... )` for string-to-integer conversions (e.g., `CONV i( iv_field )`)
- Use `##NO_HANDLER` on `CATCH cx_bali_runtime ##NO_HANDLER` pattern; here use `"#EC NEEDED` for intentional non-propagation of logger errors
- `COMMIT WORK AND WAIT` — keep all commits in `sweep_all`, never inside `check_schedules` itself
- `xsdbool(...)` for boolean expressions returning `abap_bool`

### References

- Story requirements: [Source: `_bmad-output/planning-artifacts/epics.md#Story 5.2`]
- Schedule table schema: [Source: `cz.en.orch/src/zen_orch_sched.tabl.xml`]
- Active domain values: [Source: `cz.en.orch/src/zen_orch_active.doma.xml`] — `'X'` = active, `space` = inactive
- CRON expression domain: `zen_orch_cron_expr.doma.xml` — CHAR 100
- Engine current state: [Source: `cz.en.orch/src/zcl_en_orch_engine.clas.abap`]
- Health check pattern: [Source: `cz.en.orch/src/zcl_en_orch_health_chk_query.clas.abap`]
- Architecture (scheduling): [Source: `_bmad-output/planning-artifacts/architecture.md#Integration Points`] — `check_schedules()` called from `sweep_all` as first step
- Architecture (COMMIT strategy): [Source: `_bmad-output/planning-artifacts/architecture.md#9. COMMIT Placement`] — COMMIT only in `sweep_all`
- LUW rule: [Source: `_bmad-output/planning-artifacts/architecture.md#LUW Boundary`]

## Constitution Compliance

| Principle | Application |
|-----------|-------------|
| **I — DDIC-First** | `ZEN_ORCH_SCHED` table already exists; `zen_orch_sched` used as transparent table type in SELECT; no local type definitions for DB structures |
| **II — SAP Standards** | Method names `check_schedules`, `is_schedule_due`, `matches_cron_field` ≤30 chars; ≤120-char lines; ABAP-Doc on all new method declarations |
| **III — Consult SAP Docs** | CRON arithmetic uses pure ABAP `MOD` operator and `CONV i()` — no external FM dependency required; `SPLIT AT space` is standard; dev agent must verify any FM used (e.g., `DATE_GET_WEEK`) is available on ABAP 7.58 |
| **V — Error Handling** | `ZCX_EN_ORCH_ERROR` caught per schedule entry in `check_schedules`; logged via `ZCL_EN_ORCH_LOGGER`; logger failure also caught silently; `check_schedules` outer call in `sweep_all` also wrapped in TRY/CATCH |

## Dev Agent Record

### Agent Model Used

claude-sonnet-4.6 (github-copilot/claude-sonnet-4.6)

### Debug Log References

- abaplint `no_inline_in_optional_branches`: pre-declared `lx_sched_err` in `sweep_all` CATCH and `lx_sched` in `check_schedules` CATCH to avoid inline `INTO DATA(...)` inside TRY/CATCH branches.
- `check_schedule_trigger` health check nuance: active schedule (`* * * * *`) always fires on second `sweep_all`, so `lv_perf_after > lv_perf_before` means inactive schedule also fired (failure condition).
- Sakamoto DOW algorithm used (pure arithmetic) — no `DATE_GET_WEEK` FM dependency.

### Completion Notes List

- All 4 task groups implemented: `check_schedules`, `is_schedule_due`, `matches_cron_field` added to `ZCL_EN_ORCH_ENGINE`; `sweep_all` modified to call `check_schedules` as T0; `check_schedule_trigger` health check added to `ZCL_EN_ORCH_HEALTH_CHK_QUERY` with dispatch in `build_rows` and cleanup in `cleanup_test_data`.
- All abaplint rules followed: DATA declarations at method top, no inline `INTO DATA(...)` in optional branches, ≤120-char lines, `"#EC NEEDED` on intentional CATCH blocks, ABAP-Doc only before METHODS declarations.
- `COMMIT WORK AND WAIT` remains exclusively in `sweep_all` — an extra commit added for the case where `check_schedules` fires but no active performances exist in the cycle.
- `check_schedules` never raises — all exceptions are caught and logged internally per schedule entry.

### File List

- `cz.en.orch/src/zcl_en_orch_engine.clas.abap` — added `check_schedules`, `is_schedule_due`, `matches_cron_field` declarations and implementations; modified `sweep_all`
- `cz.en.orch/src/zcl_en_orch_health_chk_query.clas.abap` — added `check_schedule_trigger` declaration and implementation; updated `build_rows` (Phase 9 dispatch) and `cleanup_test_data`

### Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-06 | Story implemented — all tasks complete, status → review | claude-sonnet-4.6 |
| 2026-04-06 | Code review complete — 3 decision-needed, 5 patch, 4 defer, 5 dismissed | claude-sonnet-4.6 |
| 2026-04-06 | All patches applied (P1–P4); D1/D2/D3 resolved; get_logger() added; story → done | claude-sonnet-4.6 |

---

## Review Findings

### Decision-Needed

- [x] [Review][Decision] **`*/0` zero-divisor in `matches_cron_field`** — ABAP does NOT short-circuit boolean AND; both operands of `lv_step > 0 AND iv_value MOD lv_step = 0` are evaluated before the AND is applied. A stored cron field `*/0` causes `COMPUTE_INT_ZERODIVIDE` dump that escapes all `zcx_en_orch_error` catchers and aborts the sweep. **Fixed:** Added `IF lv_step <= 0. rv_match = abap_false. RETURN. ENDIF.` guard before the MOD. [zcl_en_orch_engine.clas.abap — `matches_cron_field`]

- [x] [Review][Decision] **`CONV i( iv_field )` runtime dump on non-numeric cron field** — Malformed/unsupported cron tokens (ranges like `1-5`, comma lists `1,3`, typos `"1x"`) reach the exact-match branch and raise `CX_SY_CONVERSION_NO_NUMBER`. **Fixed:** Wrapped `CONV i( iv_field )` in `TRY...CATCH cx_sy_conversion_no_number`, returning `abap_false` on conversion error. [zcl_en_orch_engine.clas.abap — `matches_cron_field`]

- [x] [Review][Decision] **AC#4 uses fresh logger instance, not the engine logger** — **Fixed:** Added `get_logger()` public method to `ZCL_EN_ORCH_ENGINE` that delegates to `zcl_en_orch_logger=>create()`. `check_schedules` and `sweep_all` now call `get_logger()->log_exception(...)` instead of creating loggers inline. [zcl_en_orch_engine.clas.abap — `get_logger`, `check_schedules`, `sweep_all`]

### Patch

- [x] [Review][Patch] **Inline `INTO DATA(lx_orch)` / `INTO DATA(lx_root)` in new CATCH branches violates abaplint `no_inline_in_optional_branches`** — `check_resume_performance` and `check_schedule_trigger` both use `CATCH zcx_en_orch_error INTO DATA(lx_orch).` and `CATCH cx_root INTO DATA(lx_root).` — abaplint will fail. All exception variables must be declared at method top. **Fixed:** `check_resume_performance` hoisted in prior session; `check_schedule_trigger` hoisted `lx TYPE REF TO zcx_en_orch_error` to DATA section and replaced `INTO DATA(lx)` with `INTO lx`. [zcl_en_orch_health_chk_query.clas.abap]

- [x] [Review][Patch] **`check_schedule_trigger` Part-2 fail condition is inverted — false GREEN when active schedule silently does not fire** — **Fixed:** Changed condition from `WHEN lv_perf_after > lv_perf_before + 1` to `WHEN lv_perf_after <> lv_perf_before + 1`, so both "too many" and "too few" performances result in RED. [zcl_en_orch_health_chk_query.clas.abap — `check_schedule_trigger`]

- [x] [Review][Patch] **Missing `test_schedule_trigger` unit test method in `ltcl_health_chk_query`** — **Fixed:** Added `METHODS test_schedule_trigger FOR TESTING` declaration and implementation calling `run_test( 'SCHEDULE_TRIGGER' )`. [zcl_en_orch_health_chk_query.clas.testclasses.abap]

- [x] [Review][Patch] **`check_resume_performance` early-RETURN path produces no output row** — Investigated: early-return block correctly appends before returning. No double-append on error paths. **Non-issue — dismissed.**

- [x] [Review][Patch] **`resume_performance` does not check `sy-subrc` after final `UPDATE zen_orch_perf SET status = running`** — **Fixed:** Added `IF sy-subrc <> 0. RAISE EXCEPTION TYPE zcx_en_orch_error ... ENDIF.` after the UPDATE. [zcl_en_orch_engine.clas.abap — `resume_performance`]

### Defer

- [x] [Review][Defer] **No idempotency guard: same schedule fires on every `sweep_all` within the same minute** — `is_schedule_due` only evaluates the CRON expression against current time; it does not compare against `changed_at` to prevent re-fire within the same time window. Multiple `sweep_all` calls in the same minute create duplicate performances. Pre-existing design limitation per story spec ("no deduplication beyond CRON evaluation") — deferred, design decision not in scope of zen-5-2.

- [x] [Review][Defer] **`changed_at` stores date only (`sy-datum`), losing sub-minute precision** — The `CHANGED_AT` field is `AEDAT` (date type). The spec explicitly says `sy-datum`. No timestamp companion field. Future idempotency logic would require a paired `changed_time` field. Deferred — spec-compliant as written, out of scope.

- [x] [Review][Defer] **Permanently broken schedule re-fires on every sweep with no circuit-breaker** — If `create_performance` fails (e.g., SCORE_NOT_FOUND), the error is caught/logged and the schedule retries on the next sweep indefinitely. No auto-deactivation or retry-count limit. Deferred — out of scope for zen-5-2; suitable for a future zen-5-x story.

- [x] [Review][Defer] **`APPEND`-based lookup table for Sakamoto DOW is fragile to future edits** — The 12 `APPEND` statements building `lt_t` could be silently broken by insertion/reorder. A `VALUE #(...)` constructor would be safer. Pre-existing architectural choice from the spec's dev notes — deferred, low risk for current implementation.
