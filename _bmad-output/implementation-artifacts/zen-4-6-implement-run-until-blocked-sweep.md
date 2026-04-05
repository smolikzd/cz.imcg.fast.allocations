---
story_id: "zen-4-6"
epic_id: "zen-epic-4"
title: "Implement run-until-blocked sweep"
target_repository: "cz.en.orch"
depends_on:
  - "zen-4-5"
constitution_principles:
  - "Principle I - DDIC-First"
  - "Principle II - SAP Standards"
  - "Principle III - Consult SAP Docs"
  - "Principle IV - Factory Pattern"
  - "Principle V - Error Handling"
status: "done"
completion_date: "2026-04-05"
commit_hash: "264bbdc"
commit_repository: "cz.en.orch"
created: "2026-04-05"
---

# Story zen-4-6: Implement Run-Until-Blocked Sweep

## User Story

As an engine,
I want the sweep algorithm to advance all active performances as far as possible in a single synchronous pass,
So that each APJ sweep call makes maximum progress without waiting for external async work to complete.

## Background

Stories zen-4-1 through zen-4-5 have delivered the full helper set for the engine:

| Story | What was delivered |
|-------|--------------------|
| zen-4-1 | `ZCL_EN_ORCH_ENGINE` singleton, `gc_status` constants, all method stubs |
| zen-4-2 | `create_performance` + `snapshot_score` |
| zen-4-3 | `resolve_params` (three-level JSON merge, `{{loop_iteration}}` substitution) |
| zen-4-4 | `dispatch_step` + `poll_step_status` |
| zen-4-5 | `evaluate_gate` + `advance_loop` |

Both `sweep_all` and `advance_performance` are currently stubs in
`zcl_en_orch_engine.clas.abap`:

```abap
METHOD sweep_all.
  " Stub — implemented in Story zen-4-6
ENDMETHOD.

METHOD advance_performance.
  " Stub — implemented in Story zen-4-6
ENDMETHOD.
```

This story implements both methods and replaces the GREY `SWEEP_ALL` health
check stub with a real GREEN check.

## Scope

### In scope

- `advance_performance` — loop through `ZEN_ORCH_P_STEP` in `SCORE_SEQ` order,
  call the correct helper per `ELEM_TYPE`, stop when blocked
- `sweep_all` — load all PENDING/RUNNING performances, call `advance_performance`
  for each, `COMMIT WORK` per performance, catch errors and mark FAILED
- `SWEEP_ALL` health check: replace the GREY stub with a real GREEN
  `check_sweep_all` method in `ZCL_EN_ORCH_HEALTH_CHK_QUERY`
- Unit test `test_sweep_all` in `ltcl_health_chk_query`

### Out of scope

- `resume_performance` — implemented in zen-4-7
- `cancel_performance` / `restart_performance` — implemented in zen-5-3
- APJ job class `ZCL_EN_ORCH_JOB_SWEEP` — implemented in zen-5-1
- BSR prerequisite gates — Phase 2 / `evaluate_prereq_gate` remains a stub

## Acceptance Criteria

### AC1 — sweep_all processes all PENDING/RUNNING performances

**Given** performances in STATUS = `'P'` (PENDING) or `'R'` (RUNNING) exist in `ZEN_ORCH_PERF`

**When** `sweep_all` is called

**Then** each performance with STATUS `'P'` or `'R'` is processed by `advance_performance`

**And** performances in terminal status (`'C'`, `'F'`, `'X'`, `'B'`) are skipped entirely

### AC2 — advance_performance marks performance COMPLETED when all steps done

**Given** a performance has a single STEP with STATUS `'P'` and MOCK adapter registered

**When** `sweep_all` is called

**Then** `dispatch_step` is called for the STEP → MOCK returns `'C'` immediately
**And** `advance_performance` detects all steps are `'C'` and sets `ZEN_ORCH_PERF.STATUS = 'C'`
**And** `COMMIT WORK` is issued exactly once for that performance (in `sweep_all`, not inside `advance_performance`)

### AC3 — advance_performance blocks on RUNNING async step

**Given** a step is in STATUS = `'R'` (RUNNING — awaiting external result)

**When** `advance_performance` processes it

**Then** `poll_step_status` is called → if step remains `'R'`, the loop stops for this performance (run-until-blocked)
**And** the performance STATUS remains `'R'` (not forced to `'C'`)

### AC4 — advance_performance blocks on incomplete GATE

**Given** a GATE step has STATUS `'P'` and not all group STEP members are `'C'`

**When** `advance_performance` reaches the GATE

**Then** `evaluate_gate` is called → gate stays `'P'` → performance loop stops (run-until-blocked)

### AC5 — Failed dispatch fails the performance; sweep continues

**Given** performance A has a STEP where `adapter->start` raises `ZCX_EN_ORCH_ERROR`
**And** performance B is also active

**When** `sweep_all` is called

**Then** `advance_performance` for A propagates `ZCX_EN_ORCH_ERROR`
**And** `sweep_all` catches the exception, sets A's `ZEN_ORCH_PERF.STATUS = 'F'` (FAILED)
**And** `COMMIT WORK` is issued for performance A
**And** `sweep_all` continues processing performance B (fail-stop per performance, not global abort)

### AC6 — COMMIT WORK location: sweep_all only

**Then** `advance_performance`, `dispatch_step`, `poll_step_status`, `evaluate_gate`, and `advance_loop`
must NOT call `COMMIT WORK`
**And** `COMMIT WORK` is called exactly once per performance in `sweep_all` (NFR1: idempotency)

### AC7 — Crash recovery / stateless re-entry

**Given** a sweep ran and committed performance A (steps partially done), then the system crashed

**When** the next `sweep_all` runs

**Then** it re-reads all performance states from `ZEN_ORCH_PERF` and `ZEN_ORCH_P_STEP` from the DB
**And** continues each performance from its last committed state (no in-memory residue from prior sweep)

### AC8 — SWEEP_ALL health check goes GREEN

The GREY `SWEEP_ALL` stub row in `build_rows` is replaced with a call to `check_sweep_all`.

`check_sweep_all` when run:

1. Inserts test score `ZHEALTH_CHK_SCORE` with one STEP (MOCK adapter)
2. Registers MOCK adapter in `ZEN_ORCH_ADPT_R`
3. Calls `create_performance` → gets UUID
4. Calls `sweep_all`
5. Reads `ZEN_ORCH_PERF.STATUS` for the UUID → asserts `'C'` (COMPLETED)
6. Reports GREEN; calls `cleanup_check_data( )`

### AC9 — Unit test passes

`test_sweep_all` in `ltcl_health_chk_query` calls `run_test( 'SWEEP_ALL', … )` and asserts GREEN.

### AC10 — abaplint clean (120-char line limit)

All modified files pass abaplint with no line-length violations.

## Technical Notes

### advance_performance design

```
METHOD advance_performance.
  " Load all performance steps in SCORE_SEQ order (lowest first)
  SELECT *
    FROM zen_orch_p_step
    WHERE perf_uuid = @iv_perf_uuid
    ORDER BY score_seq ASCENDING, loop_iteration ASCENDING
    INTO TABLE @DATA(lt_steps).

  DATA lv_blocked TYPE abap_bool.

  LOOP AT lt_steps INTO DATA(ls_step).
    CASE ls_step-status.
      WHEN gc_status-completed OR gc_status-failed
           OR gc_status-cancelled OR gc_status-paused.
        CONTINUE.                          " Skip terminal/control steps

      WHEN gc_status-pending.
        CASE ls_step-elem_type.
          WHEN 'STEP'.
            dispatch_step( ls_step ).
            " Re-read updated status from DB after dispatch
            SELECT SINGLE status FROM zen_orch_p_step ... INTO @DATA(lv_new).
            IF lv_new = gc_status-running.
              lv_blocked = abap_true.  " Async — stop scanning
              EXIT.
            ENDIF.

          WHEN 'GATE'.
            evaluate_gate( ls_step ).
            " Re-read: if gate still PENDING → blocked
            SELECT SINGLE status FROM zen_orch_p_step ... INTO @DATA(lv_gate).
            IF lv_gate <> gc_status-completed.
              lv_blocked = abap_true.
              EXIT.
            ENDIF.

          WHEN 'PREREQ_GATE'.
            evaluate_prereq_gate( ls_step ).  " Stub — always stays PENDING
            lv_blocked = abap_true.
            EXIT.

          WHEN 'LOOP'.
            " advance_loop checks if inner steps done; if not → blocked
            evaluate_gate_or_wait_for_loop_inner( ls_step ).  " see below

          WHEN 'END_LOOP'.
            CONTINUE.  " Controlled by advance_loop, not directly

        ENDCASE.

      WHEN gc_status-running.
        poll_step_status( ls_step ).
        " Re-read: if still RUNNING → blocked
        SELECT SINGLE status FROM zen_orch_p_step ... INTO @DATA(lv_polled).
        IF lv_polled = gc_status-running.
          lv_blocked = abap_true.
          EXIT.
        ENDIF.
    ENDCASE.
  ENDLOOP.

  " Mark performance COMPLETED only if all steps are in a terminal status
  IF lv_blocked = abap_false.
    DATA lv_all_done TYPE abap_bool VALUE abap_true.
    LOOP AT lt_steps INTO DATA(ls_check).
      SELECT SINGLE status FROM zen_orch_p_step
        WHERE perf_uuid = @iv_perf_uuid AND score_seq = @ls_check-score_seq
          AND loop_iteration = @ls_check-loop_iteration
        INTO @DATA(lv_final_status).
      IF lv_final_status <> gc_status-completed
        AND lv_final_status <> gc_status-cancelled.
        lv_all_done = abap_false.
        EXIT.
      ENDIF.
    ENDLOOP.
    IF lv_all_done = abap_true.
      UPDATE zen_orch_perf SET status = @gc_status-completed
        WHERE perf_uuid = @iv_perf_uuid.
    ENDIF.
  ENDIF.
ENDMETHOD.
```

**Simplification note:** The pseudocode above is conceptual. The actual
implementation should be pragmatic:

- After each helper call, re-read the updated status from DB (single SELECT
  SINGLE) to decide whether blocked.
- The LOOP elem_type handling: a LOOP step itself is only a control marker
  (status = 'P' while iterating). When the engine encounters a LOOP element:
  - Check if all inner STEPs for the current LOOP_ITERATION (same REF_ID) are
    COMPLETED. If yes → call `advance_loop` to either clone next iteration or
    mark LOOP/END_LOOP done. If no → the inner steps will be encountered
    later in SCORE_SEQ order and dispatched/polled directly. The LOOP step
    itself is not blocking — the inner steps block naturally.
  - Simpler: treat LOOP and END_LOOP as no-ops in the dispatch loop; inner
    STEPs are dispatched normally. `advance_loop` is called from the outer
    sweep logic after all inner steps in a LOOP_ITERATION are COMPLETED.
  - **Recommended approach:** At the end of processing all steps, check if
    any LOOP step's inner steps for the current iteration are all COMPLETED.
    If so, call `advance_loop` for that LOOP step. Repeat the outer loop until
    no more progress. This avoids having to track LOOP state inside the single-
    pass scan.

**Alternative simpler approach (recommended for this story):**

```
advance_performance outer loop:
  FOR EACH step in SCORE_SEQ/LOOP_ITERATION order:
    IF PENDING STEP    → dispatch_step; IF still RUNNING → mark blocked, EXIT
    IF RUNNING STEP    → poll_step_status; IF still RUNNING → mark blocked, EXIT
    IF PENDING GATE    → evaluate_gate; IF still PENDING → mark blocked, EXIT
    IF PENDING LOOP    → check inner steps; if all done call advance_loop; else skip
    IF END_LOOP        → skip (mirrored by advance_loop)
    IF terminal status → skip

  After full pass with no block:
    Re-check all steps terminal → if yes: UPDATE PERF STATUS = 'C'
```

### sweep_all design

```
METHOD sweep_all.
  " Load all active performance UUIDs
  SELECT perf_uuid
    FROM zen_orch_perf
    WHERE status = @gc_status-pending
       OR status = @gc_status-running
    INTO TABLE @DATA(lt_active_perfs).

  LOOP AT lt_active_perfs INTO DATA(ls_perf).
    TRY.
        advance_performance( ls_perf-perf_uuid ).
      CATCH zcx_en_orch_error INTO DATA(lx_err).
        " Fail-stop: one performance failure must not abort the sweep
        UPDATE zen_orch_perf
          SET status = @gc_status-failed
          WHERE perf_uuid = @ls_perf-perf_uuid.
        TRY.
            DATA(lo_logger) = zcl_en_orch_logger=>create( ).
            lo_logger->log_exception( lx_err ).
          CATCH zcx_en_orch_error.                        "#EC NEEDED
        ENDTRY.
    ENDTRY.
    COMMIT WORK AND WAIT.  " Independent commit per performance (NFR1)
  ENDLOOP.
ENDMETHOD.
```

### Key design rules (enforce in implementation)

| Rule | Why |
|------|-----|
| `COMMIT WORK` only in `sweep_all`, once per performance iteration | NFR1 idempotency — crash in mid-sweep → re-sweep resumes from last commit |
| `advance_performance` raises `ZCX_EN_ORCH_ERROR` on unrecoverable errors | Fail-stop (NFR4) — `sweep_all` catches and marks FAILED |
| `advance_performance` does NOT call `COMMIT WORK` | Same as above |
| Re-read step status from DB after each helper call | Steps may change status in DB; in-memory lt_steps snapshot is stale |
| SELECT active performances at start of `sweep_all` | Stateless — no in-memory state from prior sweep (NFR1, AC7) |
| PAUSED performances (STATUS = `'B'`) are skipped by `sweep_all` | Only `resume_performance` (zen-4-7) can unblock them |

### LOOP element handling in advance_performance

The LOOP/END_LOOP dance is tricky. The recommended implementation:

1. On encountering a LOOP step (STATUS `'P'`):
   - Check if inner steps for the current `is_loop_step-loop_iteration`
     (WHERE ref_id = loop_ref_id AND elem_type = 'STEP') are all `'C'`.
   - If yes → call `advance_loop( ls_loop_step )`.
     - If `advance_loop` incremented the iteration (more to go): the new
       inner steps are now PENDING and will be dispatched on this or the
       next pass. LOOP step stays `'P'`.
     - If `advance_loop` exhausted iterations: LOOP and END_LOOP are now `'C'`.
   - If no (inner steps still running/pending) → CONTINUE (the inner steps
     themselves will be encountered in SCORE_SEQ order and will block/dispatch).
2. On encountering an END_LOOP step: always CONTINUE (its status is managed
   by `advance_loop`).

### Health check: SWEEP_ALL

Replace in `build_rows`:

```abap
" Before (GREY stub):
lv_logic = 'sweep_all iterates all PENDING/RUNNING performances and advances each.'
         && ' Implemented in Story zen-4-6. Excluded from health check until'
         && ' sweep_all is fully implemented and safe to call in dialog context.'.
add_grey_row(
  iv_cap_id = 'SWEEP_ALL'
  iv_desc   = 'sweep_all — advances all active performances (zen-4-6)'
  iv_logic  = lv_logic
).
```

With:

```abap
" After (real check):
IF mv_filter_test_id IS INITIAL OR mv_filter_test_id = 'SWEEP_ALL'.
  check_sweep_all( ).
ENDIF.
```

`check_sweep_all` method template:

```abap
METHOD check_sweep_all.
  DATA ls_row TYPE ty_row.
  DATA lv_start_ts  TYPE timestampl.
  DATA lv_end_ts    TYPE timestampl.
  DATA lv_perf_uuid TYPE zen_orch_perf_uuid.

  ls_row-capabilityid = 'SWEEP_ALL'.
  ls_row-description  = 'sweep_all: single STEP performance → STATUS=C after one sweep'.
  ls_row-testlogic    = |Creates score { gc_test_score_id } with 1 STEP (MOCK adapter).|
                     && ' Registers MOCK adapter. Calls create_performance → UUID.'
                     && ' Calls sweep_all. Asserts ZEN_ORCH_PERF.STATUS = C (COMPLETED).'
                     && ' Proves advance_performance dispatches the STEP, detects all'
                     && ' steps COMPLETED, updates PERF status, and sweep_all commits.'.

  GET TIME STAMP FIELD lv_start_ts.
  TRY.
      MODIFY zen_orch_adpt_r FROM @( VALUE #(
        adapter_type = 'MOCK'
        impl_class   = 'ZCL_EN_ORCH_ADAPTER_MOCK'
      ) ).

      INSERT zen_orch_score FROM @( VALUE #(
        score_id    = gc_test_score_id
        description = 'Health check sweep_all test'
        params_json = '{}'
        created_by  = sy-uname
        created_at  = sy-datum
      ) ).
      INSERT zen_orch_s_step FROM @( VALUE #(
        score_id     = gc_test_score_id
        score_seq    = 10
        elem_type    = 'STEP'
        adapter_type = 'MOCK'
        params_json  = '{}'
      ) ).

      DATA(lo_engine) = zcl_en_orch_engine=>get_instance( ).
      lv_perf_uuid = lo_engine->create_performance( iv_score_id = gc_test_score_id ).
      COMMIT WORK AND WAIT.  " Commit performance creation before sweep

      lo_engine->sweep_all( ).

      SELECT SINGLE status FROM zen_orch_perf
        INTO @DATA(lv_perf_status)
        WHERE perf_uuid = @lv_perf_uuid.

      GET TIME STAMP FIELD lv_end_ts.
      ls_row-durationms = CONV i( ( lv_end_ts - lv_start_ts ) * 1000 ).

      IF lv_perf_status = 'C'.
        ls_row-status  = gc_green.
        ls_row-message = |sweep_all completed; PERF STATUS=C|.
      ELSE.
        ls_row-status  = gc_red.
        ls_row-message = |Expected PERF STATUS=C after sweep, got { lv_perf_status }|.
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

**Important:** The health check calls `COMMIT WORK AND WAIT` after
`create_performance` to ensure the performance row is visible to `sweep_all`
when it runs its SELECT. This mirrors real APJ behavior where performance
creation and sweep are separate transactions.

### LOOP step in P_STEP key lookup

`advance_loop` is called with the LOOP step's `zen_orch_s_perf_step`. The LOOP
step row in `ZEN_ORCH_P_STEP` has `loop_iteration = 0` always (it is a control
marker, not a work unit). The **current logical iteration** is tracked by the
`loop_iteration` of the inner STEP rows that `advance_loop` inserts.

When `advance_performance` encounters a LOOP step (elem_type = 'LOOP', always
at loop_iteration = 0), it must:

```abap
" Find current iteration: max loop_iteration among inner steps for this ref_id
DATA lv_cur_iter TYPE zen_orch_de_loop_iter.
SELECT MAX( loop_iteration )
  FROM zen_orch_p_step
  WHERE perf_uuid = @is_loop_step-perf_uuid
    AND ref_id    = @is_loop_step-ref_id
    AND elem_type = 'STEP'
  INTO @lv_cur_iter.

" Build loop step struct with current iteration for advance_loop
DATA(ls_loop_for_adv) = VALUE zen_orch_s_perf_step(
  perf_uuid      = is_loop_step-perf_uuid
  score_seq      = is_loop_step-score_seq
  loop_iteration = lv_cur_iter   " <-- current, not 0
  elem_type      = 'LOOP'
  ref_id         = is_loop_step-ref_id
  status         = is_loop_step-status
).
advance_loop( ls_loop_for_adv ).
```

## Constitution Compliance

| Principle | Application |
|-----------|-------------|
| **I — DDIC-First** | All types from DDIC: `zen_orch_s_perf_step`, `zen_orch_perf_uuid`, `zen_orch_de_status` |
| **II — SAP Standards** | SAP naming, ≤120 chars, ABAP-Doc on public methods |
| **III — Consult SAP Docs** | ABAP Open SQL syntax for SELECT with OR condition verified |
| **IV — Factory Pattern** | Engine is singleton via `get_instance`; adapter factory called in `dispatch_step` (already implemented) |
| **V — Error Handling** | `ZCX_EN_ORCH_ERROR` raised from `advance_performance` for unrecoverable errors; `sweep_all` catches per-performance |

**No COMMIT WORK** in `advance_performance` or any helper — only in `sweep_all`.

## Files to Modify

| File | Change |
|------|--------|
| `src/zcl_en_orch_engine.clas.abap` | Implement `sweep_all` and `advance_performance` (replace stubs) |
| `src/zcl_en_orch_health_chk_query.clas.abap` | Replace GREY `SWEEP_ALL` stub with real `check_sweep_all`; add method declaration |
| `src/zcl_en_orch_health_chk_query.clas.testclasses.abap` | Add `test_sweep_all` method (declaration + implementation) |

## Previous Story Learnings (from zen-4-5)

- **Engine is a GLOBAL FRIEND of health check query** — `zcl_en_orch_health_chk_query`
  is listed in `GLOBAL FRIENDS` of `ZCL_EN_ORCH_ENGINE`. This grants the health
  check access to the engine's private methods for unit testing. No change needed.
- **Test class is `ltcl_health_chk_query`** — all unit tests live in
  `zcl_en_orch_health_chk_query.clas.testclasses.abap`. The standalone engine
  test class was deleted in zen-4-4 code review. All new tests go here.
- **`cleanup_check_data( )` must be called at the end of every `check_*` method**
  to prevent score duplicate key errors on subsequent check runs.
- **`COMMIT WORK AND WAIT` in `cleanup_test_data` and `cleanup_check_data`** —
  always present; new health check must follow the same pattern.
- **`VALUE zen_orch_s_perf_step( ... )` constructor** — used in all health
  checks to build the step structure explicitly rather than relying on SELECT
  field order. Follow this pattern.
- **abaplint 120-char line limit** — all string concatenations using `&&`
  must respect this limit. Use multiple `&&` continuation lines if needed.
- **Method name length ≤30 chars** — `check_sweep_all` (15 chars) is fine.
- **`" Stub — …` comment in current `sweep_all` and `advance_performance`** —
  these stubs contain only the comment. Replacing them = full reimplementation.

## Definition of Done

- [x] `advance_performance` implemented; handles PENDING dispatch, RUNNING poll,
      GATE evaluation, LOOP advancement, and blocked detection
- [x] `sweep_all` implemented: selects active performances, calls
      `advance_performance`, commits per performance, catches + fails on error
- [x] No `COMMIT WORK` in `advance_performance` or any helper method
- [x] `SWEEP_ALL` health check is active GREEN (not GREY)
- [x] `test_sweep_all` unit test passes (GREEN)
- [x] All previously passing unit tests (12 total) remain GREEN
- [x] abaplint passes (no 120-char violations, no check_syntax violations)
- [ ] Code pushed to `cz.en.orch` main branch

## Dev Agent Record

### Agent Model Used

github-copilot/claude-sonnet-4.6

### Debug Log References

- abaplint used to validate 120-char line limit (AC10) — zero violations
- abaplint `check_syntax` violations found and fixed: `ls_step` was inlined via `LOOP AT ... INTO DATA(ls_step)` — abaplint couldn't resolve type for method calls; fixed by declaring `DATA ls_step TYPE zen_orch_s_perf_step` at top of method
- `strict_sql` violations fixed: moved `INTO` clause to last position in all SELECT statements in `advance_performance` and `check_sweep_all`
- `definitions_top` + `no_inline_in_optional_branches` fixed: all DATA declarations moved to top of `advance_performance`; `sweep_all` uses explicit `DATA ls_perf_uuid TYPE zen_orch_perf_uuid` and `STANDARD TABLE OF zen_orch_perf_uuid`
- Residual `db_operation_in_loop` (2 new, in `sweep_all` UPDATE and `advance_performance` SELECT SINGLE after dispatch) are architecturally inherent to the run-until-blocked pattern; same pattern pre-exists in `advance_loop` and `evaluate_gate`

### Completion Notes List

1. `sweep_all` implemented in `zcl_en_orch_engine.clas.abap`: loads PENDING/RUNNING performances, calls `advance_performance` for each, commits per performance (AC6), catches `ZCX_EN_ORCH_ERROR` and marks performance FAILED (AC5)
2. `advance_performance` implemented: stateless SELECT of all steps, run-until-blocked walk per ELEM_TYPE, re-reads step status from DB after each helper call, marks performance COMPLETED when all steps terminal (AC2), marks RUNNING when blocked (AC3/AC4)
3. `check_sweep_all` added to `zcl_en_orch_health_chk_query.clas.abap`: replaces GREY stub, creates 1-STEP MOCK performance, calls sweep_all, asserts STATUS=C (AC8)
4. `test_sweep_all` added to test class (13 total unit tests) (AC9)
5. All abaplint structural issues introduced by the story fixed; no `check_syntax` or `line_length` violations remain

### File List

| File | Change |
|------|--------|
| `src/zcl_en_orch_engine.clas.abap` | Implemented `sweep_all` and `advance_performance` (replaced stubs) |
| `src/zcl_en_orch_health_chk_query.clas.abap` | Replaced GREY SWEEP_ALL stub with `check_sweep_all`; added method declaration |
| `src/zcl_en_orch_health_chk_query.clas.testclasses.abap` | Added `test_sweep_all` declaration and implementation |

### Change Log

| Date | Change |
|------|--------|
| 2026-04-05 | Implementation complete; abaplint clean (no line_length, no check_syntax); status → review; commit ef1d600 (cz.en.orch) |
| 2026-04-05 | Code review complete — 3 patches, 4 deferred, 4 dismissed. Patches committed and pushed; commit 264bbdc (cz.en.orch). Status → done. |

## Tasks / Subtasks

### Review Findings

- [x] [Review][Patch] PAUSED step not excluded from terminal count in T2.3 [`src/zcl_en_orch_engine.clas.abap:406`] — T2.3 `SELECT COUNT(*)` excludes completed/cancelled/failed but not paused. A step in STATUS='B' is skipped in the dispatch loop (treated as terminal) but still counted as non-terminal in T2.3, preventing performance from ever reaching COMPLETED if any step is paused. Fix: add `AND status <> @gc_status-paused` to the COUNT query.
- [x] [Review][Patch] `lv_cur_status` not cleared before SELECT SINGLE re-reads [`src/zcl_en_orch_engine.clas.abap:320,334,390`] — `lv_cur_status` is reused across multiple SELECT SINGLE calls without being cleared. If any SELECT SINGLE returns SY-SUBRC <> 0 (row gone), the variable retains its prior value and may trigger a false RUNNING or false not-completed detection. Fix: add `CLEAR lv_cur_status` before each of the three SELECT SINGLE statements inside advance_performance.
- [x] [Review][Patch] `check_sweep_all` uses INSERT for score/step — duplicate key risk on retry [`src/zcl_en_orch_health_chk_query.clas.abap:+50-58`] — If a prior health check run crashed before `cleanup_check_data` was called, the `INSERT zen_orch_score` and `INSERT zen_orch_s_step` will fail with a duplicate-key error on retry, causing a RED result instead of GREEN. All other check methods use MODIFY for idempotent inserts. Fix: replace `INSERT zen_orch_score FROM` and `INSERT zen_orch_s_step FROM` with `MODIFY zen_orch_score FROM` and `MODIFY zen_orch_s_step FROM`.
- [x] [Review][Defer] Empty inner catch swallows logger failure silently [`src/zcl_en_orch_engine.clas.abap:210-214`] — deferred, pre-existing pattern (`#EC NEEDED` suppress marker intentional; same in poll_step_status)
- [x] [Review][Defer] LOOP new inner steps not in snapshot — not dispatched in same pass [`src/zcl_en_orch_engine.clas.abap:371-384`] — deferred, intentional by design (documented in spec and code comments)
- [x] [Review][Defer] UPDATE in CATCH block unguarded — lock failure leaves performance in indeterminate state [`src/zcl_en_orch_engine.clas.abap:207-209`] — deferred, out of scope; resilience hardening targeted at zen-5-x
- [x] [Review][Defer] `check_sweep_all` cleanup_check_data called after row appended — pre-existing cleanup sequencing risk [`src/zcl_en_orch_health_chk_query.clas.abap`] — deferred, pre-existing pattern across all check methods
