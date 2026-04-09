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
status: "review"
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
