# Story zen-7-5: PREREQ_GATE Engine Support

```yaml
story_id: "zen-7-5"
epic_id: "zen-7"
title: "PREREQ_GATE Engine Support"
target_repository: "cz.en.orch"
depends_on: ["zen-7-4"]
constitution_principles:
  - "Principle II — SAP Standards"
  - "Principle V — Error Handling"
status: "done"
```

---

## User Story

As an engine,
I want to evaluate PREREQ_GATE score elements by querying the BSR,
So that a score can declare cross-performance prerequisites and the engine blocks until they are satisfied.

---

## Acceptance Criteria

**AC1 — evaluate_prereq_gate completes when prerequisite is met:**
- Given a performance step with `ELEM_TYPE = 'PREREQ_GATE'` and `REF_ID` containing a BSR key
- When `evaluate_prereq_gate` is called and BSR returns `gc_status-completed`
- Then the PREREQ_GATE step status is set to `gc_status-completed`

**AC2 — evaluate_prereq_gate fails terminally when prerequisite failed/cancelled:**
- Given the BSR returns `gc_status-failed` or `gc_status-cancelled` for the key
- When `evaluate_prereq_gate` is called
- Then a `ZCX_EN_ORCH_ERROR` is raised (engine marks the performance FAILED)

**AC3 — evaluate_prereq_gate blocks (yields) when prerequisite is not yet done:**
- Given the BSR returns PENDING, RUNNING, or PAUSED for the key
- When `evaluate_prereq_gate` is called
- Then the method returns normally without modifying step status (engine yields)

**AC4 — advance_performance routes PREREQ_GATE to evaluate_prereq_gate:**
- Given `advance_performance` encounters a step with `ELEM_TYPE = 'PREREQ_GATE'` in STATUS = 'P'
- When the step is processed
- Then `evaluate_prereq_gate` is called (same routing logic as `evaluate_gate`)
- And the `lv_blocked` / EXIT logic follows the step status after the call:
  - If the step was set to COMPLETED → do NOT block (continue sweep)
  - If evaluate_prereq_gate raised → exception propagates normally
  - If the step is still PENDING (blocking) → `lv_blocked = abap_true` + EXIT

**AC5 — Already-COMPLETED PREREQ_GATE is skipped (standard step flow):**
- Given a PREREQ_GATE that has already been COMPLETED
- When `advance_performance` processes subsequent steps
- Then execution continues past the PREREQ_GATE without re-checking

**AC6 — evaluate_prereq_gate is a private method on ZCL_EN_ORCH_ENGINE:**
- And the `ELEM_TYPE` routing in `advance_performance` is updated to route `PREREQ_GATE` to `evaluate_prereq_gate`
- And the class activates without errors

---

## Implementation Notes

### BSR key convention
The BSR key for a performance is: `<SCORE_ID>:<PERF_UUID>` (established in zen-7-4).

The PREREQ_GATE step's `REF_ID` field carries the BSR key of the prerequisite performance.

### evaluate_prereq_gate logic
```abap
METHOD evaluate_prereq_gate.
  DATA lv_prereq_status TYPE zen_orch_de_status.

  " Query BSR for the status of the prerequisite performance
  lv_prereq_status = mo_bsr->check_prerequisite( iv_bsr_key = is_perf_step-ref_id ).

  CASE lv_prereq_status.
    WHEN gc_status-completed.
      " Prerequisite satisfied — mark this gate step COMPLETED
      UPDATE zen_orch_p_step
        SET status = @gc_status-completed
        WHERE perf_uuid      = @is_perf_step-perf_uuid
          AND score_seq      = @is_perf_step-score_seq
          AND loop_iteration = @is_perf_step-loop_iteration.

    WHEN gc_status-failed.
      RAISE EXCEPTION TYPE zcx_en_orch_error
        EXPORTING
          textid       = zcx_en_orch_error=>engine_sweep_failed
          mv_perf_uuid = is_perf_step-perf_uuid
          mv_detail    = |PREREQ_GATE seq={ is_perf_step-score_seq }|
                      && | BSR key={ is_perf_step-ref_id } — prerequisite FAILED|.

    WHEN gc_status-cancelled.
      RAISE EXCEPTION TYPE zcx_en_orch_error
        EXPORTING
          textid       = zcx_en_orch_error=>engine_sweep_failed
          mv_perf_uuid = is_perf_step-perf_uuid
          mv_detail    = |PREREQ_GATE seq={ is_perf_step-score_seq }|
                      && | BSR key={ is_perf_step-ref_id } — prerequisite CANCELLED|.

    WHEN OTHERS.
      " PENDING / RUNNING / PAUSED — block (yield, engine retries next sweep)
  ENDCASE.
ENDMETHOD.
```

### advance_performance routing fix
Replace the current `WHEN 'PREREQ_GATE'` block:
```abap
" OLD (always blocks):
WHEN 'PREREQ_GATE'.
  evaluate_prereq_gate( ls_step ).
  lv_blocked = abap_true.
  EXIT.
```

New routing (matches `evaluate_gate` pattern — check status after call):
```abap
WHEN 'PREREQ_GATE'.
  evaluate_prereq_gate( ls_step ).
  " Re-read status: if evaluate_prereq_gate completed it, continue sweep;
  " otherwise block (prerequisite not yet satisfied — yield to next sweep).
  CLEAR lv_cur_status.
  SELECT SINGLE status
    FROM zen_orch_p_step
    WHERE perf_uuid      = @ls_step-perf_uuid
      AND score_seq      = @ls_step-score_seq
      AND loop_iteration = @ls_step-loop_iteration
    INTO @lv_cur_status.
  IF lv_cur_status <> gc_status-completed.
    lv_blocked = abap_true.
    EXIT.
  ENDIF.
```

### BSR cleanup in health check
Both `cleanup_test_data` and `cleanup_check_data` must delete BSR entries for the test score:
```abap
DELETE FROM zen_orch_bsr
  WHERE bsr_key LIKE @( gc_test_score_id && '%' ).
```

### Health check test (check_prereq_gate)
Test scenario:
1. Insert PREREQ_GATE score: score with one `PREREQ_GATE` step and one trailing `STEP`
2. Create a "prerequisite" performance (perf_A) of a different score — or directly insert a BSR entry pointing to a completed performance
3. Inject a mock BSR into the engine via `set_bsr`
4. Create perf for the PREREQ_GATE score
5. First sweep: PREREQ_GATE still PENDING (mock BSR returns PENDING) → blocked
6. Update mock BSR to return COMPLETED
7. Second sweep: PREREQ_GATE passes, trailing STEP dispatched and COMPLETED, PERF = COMPLETED

Because we need a controllable BSR mock, the test directly inserts a BSR entry for a "prereq" performance, then calls `sweep_all` twice — once before the prereq completes and once after — asserting the expected PERF statuses.

---

## Tasks

- [x] 1. Create this story file
- [x] 2. Add BSR table cleanup to `cleanup_test_data` and `cleanup_check_data`
- [x] 3. Implement `evaluate_prereq_gate` method body in `zcl_en_orch_engine.clas.abap`
- [x] 4. Fix `advance_performance` PREREQ_GATE routing to use post-call status check
- [x] 5. Add `check_prereq_gate` method to `zcl_en_orch_health_chk_query.clas.abap`
- [x] 6. Add `test_prereq_gate` FOR TESTING method to `zcl_en_orch_health_chk_query.clas.testclasses.abap`
- [x] 7. Update sprint-status.yaml

### Review Findings

- [x] [Review][Patch] P1: `ref_id` CHAR 20 truncates BSR key in health check test — `lv_prereq_bsr_key` is 50 chars (17 + 1 + 32); `zen_orch_s_step.ref_id` is CHAR 20; INSERT silently truncates; `advance_performance` then passes the truncated key to `check_prerequisite` which finds no BSR row → gate never resolves [`zcl_en_orch_health_chk_query.clas.abap:2087`] — Fix: widen `zen_orch_de_ref_id` domain to CHAR 255, OR use a shorter prereq BSR key in the test (construct a short fixed key, store it in BSR directly, pass same short key as `ref_id`)
- [x] [Review][Patch] P2: No `sy-dbcnt`/`sy-subrc` guard after `UPDATE zen_orch_p_step` in `evaluate_prereq_gate` — if UPDATE affects 0 rows (phantom step), gate silently stays PENDING; sweep blocks forever with no error [`zcl_en_orch_engine.clas.abap:1034`] — Fix: `IF sy-dbcnt = 0. RAISE EXCEPTION ... mv_detail = |evaluate_prereq_gate: step row not found|. ENDIF.`
- [x] [Review][Patch] P3: No `sy-subrc` guard after re-read `SELECT SINGLE` in `advance_performance` PREREQ_GATE branch — if row missing, `lv_cur_status` is initial (≠ COMPLETED) → `lv_blocked = abap_true` → silent infinite stall instead of error [`zcl_en_orch_engine.clas.abap:~685`] — Fix: `IF sy-subrc <> 0. RAISE EXCEPTION ... ENDIF.` after the SELECT
- [x] [Review][Patch] P4: `created_at = sy-datum` in `INSERT zen_orch_score` should be `created_at = lv_ts` (all other inserts in the same method use `lv_ts`) [`zcl_en_orch_health_chk_query.clas.abap:2069`]
- [x] [Review][Patch] P5: Test not resilient to prior crashed run — no upfront cleanup of `gc_test_score_id` score/step data before `INSERT zen_orch_score`; if a previous run left stale rows, `create_performance` raises collision. Fix: add `cleanup_check_data( )` / explicit DELETEs at start of method (before the score INSERT), same as other check_* methods implicitly rely on.
- [x] [Review][Patch] P6: Pre-req BSR/perf rows leaked when exception fires after `INSERT zen_orch_bsr` but before end-of-method cleanup — verified: post-ENDTRY DELETEs at lines 2159-2163 are unconditional and cover `lc_prereq_score_id` prefix for all exit paths (CATCH handlers fall through, early RETURN has its own cleanup). No code change required.
- [x] [Review][Defer] D1: AC5 (already-COMPLETED PREREQ_GATE skipped) not verified in diff — relies on pre-existing step-loop logic that skips non-PENDING steps; not new code, not actionable here — deferred, pre-existing
- [x] [Review][Defer] D2: `WHEN OTHERS` silently blocks on unknown/initial BSR status — consistent with `evaluate_gate` pattern; architectural decision, not a defect in this story — deferred, pre-existing
- [x] [Review][Defer] D3: Concurrent sweep race on PREREQ_GATE (double-dispatch) — pre-existing architectural constraint (single APJ job execution model); out of scope here — deferred, pre-existing
- [x] [Review][Defer] D4: Stale BSR row not deregistered on COMPLETED — BSR lifecycle is managed by `sweep_all` deregister path; PREREQ_GATE does not own the BSR entry for the prereq performance — deferred, by design

---

## Dev Agent Record

### Session 1 — 2026-04-09

**Agent**: Claude Sonnet (via OpenCode)

**Discoveries**:
- `evaluate_prereq_gate` is a stub at lines 1011–1013 of engine
- `advance_performance` PREREQ_GATE block (lines 677–681) always blocks after calling stub
- `cleanup_test_data` / `cleanup_check_data` do NOT clean `ZEN_ORCH_BSR` — must add
- BSR key convention: `<SCORE_ID>:<PERF_UUID>` (established in zen-7-4)
- `set_bsr` is PRIVATE but health check class is in `GLOBAL FRIENDS` — can inject mock BSR
- `check_prerequisite` returns `gc_status-pending` when key is unknown (not yet registered)
- No mock BSR injection needed for `check_prereq_gate` test — real BSR + directly inserted `zen_orch_bsr` rows suffice
- `CONSTANTS lc_prereq_score_id` inside `check_prereq_gate` method; `lc_gate_score_id` references `gc_test_score_id`; separate cleanup needed for prereq score rows

**Files changed**:
- `src/zcl_en_orch_engine.clas.abap` — `evaluate_prereq_gate` fully implemented; `advance_performance` PREREQ_GATE routing fixed (post-call status re-read); doc comment updated
- `src/zcl_en_orch_health_chk_query.clas.abap` — BSR cleanup in `cleanup_test_data` + `cleanup_check_data`; `check_prereq_gate` method declared + routed in `build_rows` + fully implemented
- `src/zcl_en_orch_health_chk_query.clas.testclasses.abap` — `test_prereq_gate FOR TESTING` method declared and implemented

---

## File List

- `src/zcl_en_orch_engine.clas.abap` — Engine: `evaluate_prereq_gate` implementation + `advance_performance` routing fix
- `src/zcl_en_orch_health_chk_query.clas.abap` — Health check: BSR cleanup + `check_prereq_gate` method
- `src/zcl_en_orch_health_chk_query.clas.testclasses.abap` — Test class: `test_prereq_gate` method

---

## Change Log

| Date | Author | Change |
|------|--------|--------|
| 2026-04-09 | Claude Sonnet | Implemented `evaluate_prereq_gate` (BSR query, COMPLETED/FAILED/CANCELLED/PENDING handling) |
| 2026-04-09 | Claude Sonnet | Fixed `advance_performance` PREREQ_GATE routing — post-call status re-read (mirrors `evaluate_gate` pattern) |
| 2026-04-09 | Claude Sonnet | Added BSR cleanup (`DELETE FROM zen_orch_bsr WHERE bsr_key LIKE gc_test_score_id && '%'`) to `cleanup_test_data` + `cleanup_check_data` |
| 2026-04-09 | Claude Sonnet | Added `check_prereq_gate` health check method: two-sweep scenario (PENDING→blocked, then COMPLETED→gate passes + trailing STEP dispatched) |
| 2026-04-09 | Claude Sonnet | Added `test_prereq_gate FOR TESTING` unit test method |
