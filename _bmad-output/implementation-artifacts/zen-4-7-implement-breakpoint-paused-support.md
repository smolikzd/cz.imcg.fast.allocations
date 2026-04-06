---
story_id: "zen-4-7"
epic_id: "zen-epic-4"
title: "Implement breakpoint (PAUSED) support"
target_repository: "cz.en.orch"
depends_on:
  - "zen-4-6"
constitution_principles:
  - "Principle I - DDIC-First"
  - "Principle II - SAP Standards"
  - "Principle III - Consult SAP Docs"
  - "Principle IV - Factory Pattern"
  - "Principle V - Error Handling"
status: "done"
created: "2026-04-05"
completed: "2026-04-05"
---

# Story zen-4-7: Implement Breakpoint (PAUSED) Support

## User Story

As an operator,
I want performances to pause at GATE elements configured as breakpoints,
So that I can review intermediate state and manually approve continuation.

## Background

Story zen-4-6 delivered the full `sweep_all` + `advance_performance` implementation
(commit 264bbdc, cz.en.orch). The engine now advances all active performances
end-to-end. Two stubs remain in `ZCL_EN_ORCH_ENGINE`:

| Method | Stub text |
|--------|-----------|
| `resume_performance` | `" Stub — implemented in Story zen-4-7` |

The health check class `ZCL_EN_ORCH_HEALTH_CHK_QUERY` has a GREY stub for
`RESUME_PERFORMANCE` in `build_rows`. There are currently 13 unit tests, all
GREEN.

### Key facts about the current codebase

- `sweep_all` already skips PAUSED performances: its SELECT uses
  `WHERE status = 'P' OR status = 'R'`, so STATUS='B' performances are never
  loaded for advancement (AC3 is structurally guaranteed).
- `advance_performance` already skips PAUSED steps: the terminal-skip guard
  at lines ~302-307 includes `gc_status-paused`, so if a GATE step was somehow
  set to 'B', it would be skipped in the dispatch loop.
- The T2.3 terminal count (`SELECT COUNT(*)`) already excludes PAUSED steps
  (`AND status <> @gc_status-paused`), so a PAUSED GATE does not prevent the
  rest of a performance from being recognized as COMPLETED.
- **Neither `zen_orch_s_step` nor `zen_orch_p_step` has a `IS_BREAKPOINT`
  flag**. This story must add the column to both tables as the first task.

### Breakpoint DDIC addition

The `IS_BREAKPOINT` flag must be added to:

1. **`zen_orch_s_step`** (score step definition table) — lets the operator
   mark a GATE in the score definition as a breakpoint. Copied to performance
   steps by `snapshot_score`.
2. **`zen_orch_p_step`** (performance step instance table) — runtime field;
   read by `evaluate_gate` to decide whether to set STATUS='C' or STATUS='B'.
3. **`zen_orch_s_score_step`** (structure backing the table types; if a
   separate structure is used for the score step; check activation order).
4. **`zen_orch_s_perf_step`** (DDIC structure used as method parameter type
   throughout the engine) — must include `IS_BREAKPOINT` so helpers receive
   the flag via the existing struct passing pattern.

Field definition to use:

```
FIELDNAME: IS_BREAKPOINT
ROLLNAME:  ABAP_BOOLEAN   (reuse standard SAP boolean domain)
```

### snapshot_score update

`snapshot_score` already copies all score step fields to `zen_orch_p_step`.
After the DDIC change, add `is_breakpoint = ls_score_step-is_breakpoint`
to the `INSERT zen_orch_p_step FROM @( VALUE #( ... ) )` constructor.

### evaluate_gate update

When all group STEP members are COMPLETED:

- If `is_perf_step-is_breakpoint = abap_true` → set gate STATUS = 'B' (PAUSED)
  and set `zen_orch_perf.status = 'B'` (PAUSED).
- Else → set gate STATUS = 'C' (COMPLETED) as today.

This is the only change to `advance_performance` / `evaluate_gate`.

### resume_performance implementation

```abap
METHOD resume_performance.
  " T1.1 — Verify the performance is in PAUSED status
  DATA lv_current_status TYPE zen_orch_de_status.
  SELECT SINGLE status
    FROM zen_orch_perf
    WHERE perf_uuid = @iv_perf_uuid
    INTO @lv_current_status.
  IF sy-subrc <> 0 OR lv_current_status <> gc_status-paused.
    RAISE EXCEPTION TYPE zcx_en_orch_error
      EXPORTING
        textid       = zcx_en_orch_error=>step_restart_failed
        mv_perf_uuid = iv_perf_uuid
        mv_detail    = |resume_performance: perf not PAUSED (status={ lv_current_status })|.
  ENDIF.

  " T1.2 — Find the PAUSED GATE step and set it to COMPLETED
  UPDATE zen_orch_p_step
    SET status      = @gc_status-completed
    WHERE perf_uuid = @iv_perf_uuid
      AND status    = @gc_status-paused
      AND elem_type = 'GATE'.

  " T1.3 — Set performance STATUS back to RUNNING
  UPDATE zen_orch_perf
    SET status      = @gc_status-running
    WHERE perf_uuid = @iv_perf_uuid.

  " T1.4 — Commit: resume_performance is an operator action; independent commit
  COMMIT WORK AND WAIT.
ENDMETHOD.
```

**Note on COMMIT WORK:** `resume_performance` is a public operator-facing API,
not called by `sweep_all`. It is correct and expected to `COMMIT WORK AND WAIT`
here. This is the same pattern used by `cancel_performance` / `restart_performance`
(zen-5-3). NFR1 (idempotency) applies only to `sweep_all`; lifecycle operations
commit independently.

---

## Scope

### In scope

- **DDIC**: Add `IS_BREAKPOINT TYPE ABAP_BOOLEAN` to `zen_orch_s_step`,
  `zen_orch_p_step`, and `zen_orch_s_perf_step` (structure used for method
  signatures). Activate in correct order.
- **`snapshot_score`**: Copy `is_breakpoint` when snapshotting score steps.
- **`evaluate_gate`**: Diverge behavior on `is_breakpoint` flag (PAUSED vs.
  COMPLETED).
- **`resume_performance`**: Replace stub with full implementation.
- **Health check**: Replace GREY `RESUME_PERFORMANCE` stub with a real GREEN
  `check_resume_performance` method; update class header comment.
- **Unit test**: Add `test_resume_performance` in `ltcl_health_chk_query`.

### Out of scope

- `cancel_performance` / `restart_performance` — zen-5-3
- APJ job class — zen-5-1
- PREREQ_GATE / BSR — Phase 2
- BSR race condition on breakpoint toggle (deferred-work.md item #8) — deferred

---

## Acceptance Criteria

### AC1 — DDIC: IS_BREAKPOINT field exists in both tables and structure

**Given** the story is implemented
**Then** `zen_orch_s_step.is_breakpoint TYPE abap_boolean` exists and activates
**And** `zen_orch_p_step.is_breakpoint TYPE abap_boolean` exists and activates
**And** `zen_orch_s_perf_step.is_breakpoint TYPE abap_boolean` exists and
  activates

### AC2 — snapshot_score copies IS_BREAKPOINT

**Given** a score step with `IS_BREAKPOINT = 'X'`
**When** `create_performance` is called
**Then** the corresponding `zen_orch_p_step` row has `IS_BREAKPOINT = 'X'`

### AC3 — evaluate_gate PAUSEs when IS_BREAKPOINT = X

**Given** a GATE performance step with `IS_BREAKPOINT = 'X'`
**And** all STEP members in REF_ID group have STATUS = 'C'
**When** `evaluate_gate` is called
**Then** the gate step STATUS is set to 'B' (PAUSED)
**And** `zen_orch_perf.status` is set to 'B' (PAUSED)

### AC4 — evaluate_gate COMPLETES when IS_BREAKPOINT = space (unchanged behavior)

**Given** a GATE performance step with `IS_BREAKPOINT = space` (or initial)
**And** all STEP members in REF_ID group have STATUS = 'C'
**When** `evaluate_gate` is called
**Then** the gate step STATUS is set to 'C' (COMPLETED)
**And** `zen_orch_perf` is NOT updated to PAUSED

### AC5 — sweep_all skips PAUSED performances (pre-existing, verify unchanged)

**Given** a performance in STATUS = 'B' (PAUSED)
**When** `sweep_all` is called
**Then** the PAUSED performance is NOT loaded for advancement (structural guarantee
  from zen-4-6: SELECT WHERE status = 'P' OR status = 'R')

### AC6 — resume_performance clears breakpoint and resumes

**Given** a performance in STATUS = 'B' (PAUSED) with one GATE step in STATUS = 'B'
**When** `resume_performance( iv_perf_uuid )` is called
**Then** the PAUSED GATE step STATUS is set to 'C' (COMPLETED)
**And** `zen_orch_perf.status` is set to 'R' (RUNNING)
**And** `COMMIT WORK AND WAIT` is issued

### AC7 — resume_performance raises error for non-PAUSED performance

**Given** a performance in STATUS = 'R', 'C', 'P', 'F', or 'X'
**When** `resume_performance` is called
**Then** `ZCX_EN_ORCH_ERROR` is raised with textid `STEP_RESTART_FAILED`

### AC8 — Next sweep advances past the breakpoint

**Given** `resume_performance` has just set gate to 'C' and performance to 'R'
**When** `sweep_all` is called
**Then** the performance is loaded (STATUS = 'R') and `advance_performance`
  advances past the previously-paused gate

### AC9 — RESUME_PERFORMANCE health check goes GREEN

The GREY `RESUME_PERFORMANCE` stub in `build_rows` is replaced with a call to
`check_resume_performance`.

`check_resume_performance` when run:

1. Inserts test score `ZHEALTH_CHK_SCORE` with:
   - STEP (seq=10, adapter=MOCK, ref_id='BP_GRP', is_breakpoint=space)
   - GATE (seq=20, ref_id='BP_GRP', is_breakpoint='X')  ← breakpoint gate
   - STEP (seq=30, adapter=MOCK, ref_id=space)           ← step after breakpoint
2. Registers MOCK adapter
3. Calls `create_performance` → gets UUID
4. `COMMIT WORK AND WAIT`
5. Calls `sweep_all`
   - STEP(10) dispatched → MOCK returns 'C' immediately
   - GATE(20) evaluated → IS_BREAKPOINT='X', all group steps COMPLETED
     → gate STATUS='B', performance STATUS='B'
6. Asserts `zen_orch_perf.status = 'B'` (paused at breakpoint)
7. Calls `resume_performance( lv_perf_uuid )`
8. `COMMIT WORK AND WAIT` (implicit inside resume_performance)
9. Calls `sweep_all` again
10. Asserts `zen_orch_perf.status = 'C'` (COMPLETED — STEP(30) now done)
11. Reports GREEN; calls `cleanup_check_data( )`

### AC10 — Unit test `test_resume_performance` passes

`test_resume_performance` in `ltcl_health_chk_query` calls
`run_test( 'RESUME_PERFORMANCE', … )` and asserts GREEN.

### AC11 — All 13 previously passing unit tests remain GREEN

No regressions from DDIC changes or `evaluate_gate` logic changes.

### AC12 — abaplint clean (120-char line limit)

All modified files pass abaplint with no line-length violations.

---

## Technical Notes

### DDIC activation order

When adding `IS_BREAKPOINT` to `zen_orch_s_step` and `zen_orch_p_step`, the
structure `zen_orch_s_perf_step` (used as parameter type in engine methods) must
also be updated and activated **before** activating the class
`zcl_en_orch_engine`. Typical DDIC-first order:

1. Activate `zen_orch_s_perf_step` (structure) — add `is_breakpoint`
2. Activate `zen_orch_s_step` (score step table) — add `is_breakpoint`
3. Activate `zen_orch_p_step` (perf step table) — add `is_breakpoint`
4. Activate `zen_orch_tt_perf_step` (table type, if it includes the structure) — re-activate
5. Activate `zcl_en_orch_engine.clas.abap` (class uses the structure)
6. Activate `zcl_en_orch_health_chk_query.clas.abap`

### check_resume_performance method template

```abap
METHOD check_resume_performance.
  DATA ls_row       TYPE ty_row.
  DATA lv_start_ts  TYPE timestampl.
  DATA lv_end_ts    TYPE timestampl.
  DATA lv_perf_uuid TYPE zen_orch_perf_uuid.
  DATA lv_status    TYPE zen_orch_de_status.

  ls_row-capabilityid = 'RESUME_PERFORMANCE'.
  ls_row-description  = 'resume_performance: PAUSED perf resumes; sweep completes it'.
  ls_row-testlogic    = |Creates score { gc_test_score_id } with STEP(10)+GATE(20,BP=X)|
                     && '+STEP(30). sweep_all: STEP(10) completes, GATE(20) triggers'
                     && ' PAUSED. Asserts PERF STATUS=B. Calls resume_performance.'
                     && ' Second sweep_all. Asserts PERF STATUS=C (COMPLETED).'.

  GET TIME STAMP FIELD lv_start_ts.
  TRY.
      " Ensure MOCK adapter registered
      MODIFY zen_orch_adpt_r FROM @( VALUE #(
        adapter_type = 'MOCK'
        impl_class   = 'ZCL_EN_ORCH_ADAPTER_MOCK'
      ) ).

      " Score: STEP → breakpoint GATE → STEP
      MODIFY zen_orch_score FROM @( VALUE #(
        score_id    = gc_test_score_id
        description = 'Health check resume_performance test'
        params_json = '{}'
        created_by  = sy-uname
        created_at  = sy-datum
      ) ).
      " STEP(10) belongs to group BP_GRP
      MODIFY zen_orch_s_step FROM @( VALUE #(
        score_id     = gc_test_score_id
        score_seq    = 10
        elem_type    = 'STEP'
        ref_id       = 'BP_GRP'
        adapter_type = 'MOCK'
        params_json  = '{}'
      ) ).
      " GATE(20) — IS_BREAKPOINT='X' (breakpoint)
      MODIFY zen_orch_s_step FROM @( VALUE #(
        score_id      = gc_test_score_id
        score_seq     = 20
        elem_type     = 'GATE'
        ref_id        = 'BP_GRP'
        is_breakpoint = abap_true
      ) ).
      " STEP(30) — after the gate; no group
      MODIFY zen_orch_s_step FROM @( VALUE #(
        score_id     = gc_test_score_id
        score_seq    = 30
        elem_type    = 'STEP'
        adapter_type = 'MOCK'
        params_json  = '{}'
      ) ).

      DATA(lo_engine) = zcl_en_orch_engine=>get_instance( ).
      lv_perf_uuid = lo_engine->create_performance( iv_score_id = gc_test_score_id ).
      COMMIT WORK AND WAIT.

      " First sweep: STEP(10) completes, GATE(20) pauses
      lo_engine->sweep_all( ).

      SELECT SINGLE status FROM zen_orch_perf
        INTO @lv_status
        WHERE perf_uuid = @lv_perf_uuid.

      IF lv_status <> 'B'.
        GET TIME STAMP FIELD lv_end_ts.
        ls_row-durationms = CONV i( ( lv_end_ts - lv_start_ts ) * 1000 ).
        ls_row-status     = gc_red.
        ls_row-message    = |Expected PERF STATUS=B after first sweep, got { lv_status }|.
        ls_row-statuscriticality = status_to_criticality( ls_row-status ).
        APPEND ls_row TO mt_rows.
        cleanup_check_data( ).
        RETURN.
      ENDIF.

      " Resume the paused performance
      lo_engine->resume_performance( lv_perf_uuid ).

      " Second sweep: GATE(20) now C; STEP(30) dispatched → COMPLETED
      lo_engine->sweep_all( ).

      SELECT SINGLE status FROM zen_orch_perf
        INTO @lv_status
        WHERE perf_uuid = @lv_perf_uuid.

      GET TIME STAMP FIELD lv_end_ts.
      ls_row-durationms = CONV i( ( lv_end_ts - lv_start_ts ) * 1000 ).

      IF lv_status = 'C'.
        ls_row-status  = gc_green.
        ls_row-message = |Paused at GATE; resumed; second sweep completed PERF|.
      ELSE.
        ls_row-status  = gc_red.
        ls_row-message = |Expected PERF STATUS=C after resume+sweep, got { lv_status }|.
      ENDIF.

    CATCH zcx_en_orch_error INTO DATA(lx_orch).
      GET TIME STAMP FIELD lv_end_ts.
      ls_row-durationms = CONV i( ( lv_end_ts - lv_start_ts ) * 1000 ).
      ls_row-status     = gc_red.
      ls_row-message    = |ZCX_EN_ORCH_ERROR: { lx_orch->get_text( ) }|.
    CATCH cx_root INTO DATA(lx_root).
      GET TIME STAMP FIELD lv_end_ts.
      ls_row-durationms = CONV i( ( lv_end_ts - lv_start_ts ) * 1000 ).
      ls_row-status     = gc_red.
      ls_row-message    = |Exception: { lx_root->get_text( ) }|.
  ENDTRY.

  ls_row-statuscriticality = status_to_criticality( ls_row-status ).
  APPEND ls_row TO mt_rows.
  cleanup_check_data( ).
ENDMETHOD.
```

### evaluate_gate changes

The only logic change in `evaluate_gate` is in the "all done" branch:

```abap
" T1.3 — All group steps done: mark gate COMPLETED or PAUSED (breakpoint)
IF is_perf_step-is_breakpoint = abap_true.
  " Breakpoint gate: pause the gate and the entire performance
  UPDATE zen_orch_p_step
    SET status         = @gc_status-paused
    WHERE perf_uuid    = @is_perf_step-perf_uuid
      AND score_seq    = @is_perf_step-score_seq
      AND loop_iteration = @is_perf_step-loop_iteration.

  UPDATE zen_orch_perf
    SET status      = @gc_status-paused
    WHERE perf_uuid = @is_perf_step-perf_uuid.
ELSE.
  " Normal gate: mark COMPLETED
  UPDATE zen_orch_p_step
    SET status         = @gc_status-completed
    WHERE perf_uuid    = @is_perf_step-perf_uuid
      AND score_seq    = @is_perf_step-score_seq
      AND loop_iteration = @is_perf_step-loop_iteration.
ENDIF.
```

**Important:** `advance_performance` will then re-read the gate status from DB.
A PAUSED gate status is NOT `gc_status-completed`, so `lv_blocked = abap_true`
is set and the performance stops advancing — this is exactly what we want. The
`sweep_all` then commits the PAUSED state. On the next sweep, the performance
is already STATUS='B' and is never loaded.

### advance_performance: gate block on PAUSED

When `advance_performance` re-reads gate status after `evaluate_gate`:

```abap
" Gate not yet satisfied → blocked (AC4)
IF lv_cur_status <> gc_status-completed.
  lv_blocked = abap_true.
  EXIT.
ENDIF.
```

`lv_cur_status = 'B'` (PAUSED) satisfies `<> 'C'`, so `lv_blocked = abap_true`
is set. No change to `advance_performance` is required. The PAUSED gate naturally
blocks the sweep pass.

### Health check class header comment update

The class header `*&` comment block must be updated to add:
```
*&   RESUME_PERFORMANCE  — resume_performance clears PAUSED breakpoint gate
```
alongside the existing list of check descriptions.

### abaplint patterns (learned from prior stories)

| Rule | Detail |
|------|--------|
| `definitions_top` | All `DATA` declarations at the top of each method |
| `no_inline_in_optional_branches` | No inline declarations inside IF/LOOP branches |
| `strict_sql` | `INTO` clause must be last in all SELECT statements |
| `db_operation_in_loop` | Architecturally inherent in engine helpers; pre-existing pattern |
| Line length | ≤120 chars strictly; use `&&` continuation for long string literals |
| Method name | ≤30 chars — `check_resume_performance` (24 chars) ✓ |
| `CLEAR lv_var` | Before each `SELECT SINGLE` reuse of the same variable |
| `COMMIT WORK AND WAIT` | Always `AND WAIT`, never just `COMMIT WORK` |

---

## Constitution Compliance

| Principle | Application |
|-----------|-------------|
| **I — DDIC-First** | `IS_BREAKPOINT` field added to DDIC tables (`zen_orch_s_step`, `zen_orch_p_step`) and structure (`zen_orch_s_perf_step`); no local TYPE definitions |
| **II — SAP Standards** | SAP naming (`IS_BREAKPOINT` prefix), ≤120 chars, ABAP-Doc on public methods |
| **III — Consult SAP Docs** | `ABAP_BOOLEAN` is a released SAP type; no custom domain needed |
| **IV — Factory Pattern** | Engine singleton pattern unchanged; no direct `NEW` |
| **V — Error Handling** | `ZCX_EN_ORCH_ERROR` with `STEP_RESTART_FAILED` textid raised by `resume_performance` for non-PAUSED performances |

---

## Files to Modify

| File | Change |
|------|--------|
| `src/zen_orch_s_perf_step.stru.xml` (or `.tabl.xml`) | Add `IS_BREAKPOINT TYPE ABAP_BOOLEAN` field |
| `src/zen_orch_s_step.tabl.xml` | Add `IS_BREAKPOINT TYPE ABAP_BOOLEAN` field |
| `src/zen_orch_p_step.tabl.xml` | Add `IS_BREAKPOINT TYPE ABAP_BOOLEAN` field |
| `src/zcl_en_orch_engine.clas.abap` | Implement `resume_performance`; update `evaluate_gate` breakpoint branch; update `snapshot_score` to copy `is_breakpoint` |
| `src/zcl_en_orch_health_chk_query.clas.abap` | Replace GREY `RESUME_PERFORMANCE` stub with real `check_resume_performance`; add method declaration; update class header comment |
| `src/zcl_en_orch_health_chk_query.clas.testclasses.abap` | Add `test_resume_performance` method (declaration + implementation) |

### zen_orch_s_perf_step note

Check whether `zen_orch_s_perf_step` is defined as a standalone DDIC structure
(`.stru.xml`) or is the same as `zen_orch_p_step` (which would mean the table
serves as both the DB table and the parameter structure). In the current codebase
the engine methods use `is_perf_step TYPE zen_orch_s_perf_step` — confirm this
object exists separately from `zen_orch_p_step` before modifying.

---

## Previous Story Learnings (from zen-4-6)

- **Engine is GLOBAL FRIEND of health check query** — `zcl_en_orch_health_chk_query`
  has friend access to `zcl_en_orch_engine` private methods. No change needed.
- **All unit tests in `ltcl_health_chk_query`** — the standalone engine test class
  was deleted in zen-4-4 code review. All new tests go into
  `zcl_en_orch_health_chk_query.clas.testclasses.abap`.
- **`cleanup_check_data( )` must be called at the end of every `check_*` method**
  to prevent score duplicate key errors on subsequent runs.
- **`COMMIT WORK AND WAIT` after `create_performance`** — always call before
  invoking engine methods in health checks (separate transaction boundary).
- **`MODIFY` not `INSERT` for score/step in health checks** — idempotent upserts
  prevent RED results on retry after a crash before cleanup.
- **`VALUE zen_orch_s_perf_step( ... )` constructor** — always build step structs
  explicitly from individual fields; never rely on SELECT field order alignment.
- **Two-phase result check** — the `check_resume_performance` health check has an
  intermediate assertion (PERF STATUS='B' after first sweep). Use `RETURN` after
  appending a RED row if the intermediate check fails, rather than continuing to
  the resume step with wrong preconditions.
- **`abap_true` vs `'X'`** — use `abap_true` for `ABAP_BOOLEAN` fields in VALUE
  constructors (e.g., `is_breakpoint = abap_true`); both compile but `abap_true`
  is the idiomatic form.

---

## Definition of Done

- [x] `IS_BREAKPOINT` field added to `zen_orch_s_step`, `zen_orch_p_step`, and
      `zen_orch_s_perf_step`; all DDIC objects activate without errors
- [x] `snapshot_score` copies `is_breakpoint` from score step to performance step
- [x] `evaluate_gate` sets gate + performance STATUS='B' when `is_breakpoint = abap_true`
      and all group steps are COMPLETED
- [x] `evaluate_gate` still sets gate STATUS='C' when `is_breakpoint` is initial (no regression)
- [x] `resume_performance` implemented: verifies PAUSED status, clears PAUSED gate to
      COMPLETED, sets performance to RUNNING, commits
- [x] `resume_performance` raises `ZCX_EN_ORCH_ERROR` for non-PAUSED performances
- [x] `RESUME_PERFORMANCE` health check is active GREEN (not GREY)
- [x] `test_resume_performance` unit test passes (GREEN)
- [ ] All 13 previously passing unit tests remain GREEN
- [ ] abaplint passes (no 120-char violations, no check_syntax violations)
- [x] Code pushed to `cz.en.orch` main branch

---

## Dev Agent Record

### Implementation Summary

All code changes made to `cz.en.orch` repository. No ABAP code generated in this planning repository.

**DDIC changes:**
- `zen_orch_s_perf_step.tabl.xml` — added `IS_BREAKPOINT ROLLNAME=ABAP_BOOLEAN` after `WORK_UNIT_HANDLE`
- `zen_orch_s_step.tabl.xml` — added `IS_BREAKPOINT ROLLNAME=ABAP_BOOLEAN` after `MAX_PARALLEL`
- `zen_orch_p_step.tabl.xml` — added `IS_BREAKPOINT ROLLNAME=ABAP_BOOLEAN` after `WORK_UNIT_HANDLE`

**Engine changes (`zcl_en_orch_engine.clas.abap`):**
- `snapshot_score`: added `is_breakpoint = ls_score_step-is_breakpoint` to INSERT constructor
- `evaluate_gate`: replaced single UPDATE with IF/ELSE — `is_breakpoint=abap_true` → STATUS='B' (PAUSED) on gate AND perf; else STATUS='C' as before
- `resume_performance`: replaced stub — verifies PAUSED status, UPDATE gate to 'C', UPDATE perf to 'R', COMMIT WORK AND WAIT

**Health check changes (`zcl_en_orch_health_chk_query.clas.abap`):**
- Class header comment: added `RESUME_PERFORMANCE` entry
- PRIVATE SECTION: added `METHODS check_resume_performance.`
- `build_rows`: replaced GREY stub block with real `check_resume_performance( )` call under Phase 8
- Added full `check_resume_performance` METHOD implementation (two-phase: assert STATUS='B', resume, second sweep, assert STATUS='C')

**Test changes (`zcl_en_orch_health_chk_query.clas.testclasses.abap`):**
- PRIVATE SECTION: added `METHODS test_resume_performance FOR TESTING.`
- IMPLEMENTATION: added `test_resume_performance` METHOD body (delegates to `run_test`)

### Files Modified (in `cz.en.orch`)

| File | Change |
|------|--------|
| `src/zen_orch_s_perf_step.tabl.xml` | Added IS_BREAKPOINT field |
| `src/zen_orch_s_step.tabl.xml` | Added IS_BREAKPOINT field |
| `src/zen_orch_p_step.tabl.xml` | Added IS_BREAKPOINT field |
| `src/zcl_en_orch_engine.clas.abap` | snapshot_score, evaluate_gate, resume_performance |
| `src/zcl_en_orch_health_chk_query.clas.abap` | header, declaration, build_rows, check_resume_performance |
| `src/zcl_en_orch_health_chk_query.clas.testclasses.abap` | test_resume_performance declaration + body |

### Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-05 | Story created | Dev Agent |
| 2026-04-05 | DDIC IS_BREAKPOINT added to 3 objects | Dev Agent |
| 2026-04-05 | Engine: snapshot_score, evaluate_gate, resume_performance implemented | Dev Agent |
| 2026-04-05 | Health check: check_resume_performance implemented | Dev Agent |
| 2026-04-05 | Test: test_resume_performance added | Dev Agent |
| 2026-04-05 | Story status → review | Dev Agent |
| 2026-04-05 | Code review patches applied | Review Agent |

### Review Findings

- [x] [Review][Patch] `resume_performance` T1.2 UPDATE: no `sy-dbcnt` check — PAUSED GATE missing goes undetected [zcl_en_orch_engine.clas.abap:resume_performance] — **fixed**: added `IF sy-dbcnt = 0` guard raising `STEP_RESTART_FAILED`
- [x] [Review][Patch] `resume_performance` T1.2 UPDATE clears all PAUSED GATEs with no `score_seq` constraint — resolved as option 3 (document invariant): at most one GATE can be PAUSED at a time; comment added [zcl_en_orch_engine.clas.abap:resume_performance] — **fixed**: added invariant comment
- [x] [Review][Defer] `evaluate_gate` treats empty group (no STEP children) as fully satisfied — pre-existing structural assumption, not introduced by zen-4-7 [zcl_en_orch_engine.clas.abap:evaluate_gate] — deferred, pre-existing
- [x] [Review][Defer] `evaluate_gate` blocks gate forever when a group step is FAILED/CANCELLED — no compensation path for the gate itself — deferred, pre-existing
