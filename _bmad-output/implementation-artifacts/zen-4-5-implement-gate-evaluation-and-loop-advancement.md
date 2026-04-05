---
story_id: "zen-4-5"
epic_id: "zen-epic-4"
title: "Implement gate evaluation and loop advancement"
target_repository: "cz.en.orch"
depends_on:
  - "zen-4-4"
constitution_principles:
  - "Principle I - DDIC-First"
  - "Principle II - SAP Standards"
  - "Principle III - Consult SAP Docs"
  - "Principle IV - Factory Pattern"
  - "Principle V - Error Handling"
status: "ready-for-dev"
created: "2026-04-05"
---

# Story zen-4-5: Implement Gate Evaluation and Loop Advancement

## User Story

As an engine,
I want to evaluate GATE elements and advance LOOP elements,
So that parallel groups wait for all members before proceeding, and loop iterations execute the correct number of times.

## Background

Stories zen-4-1 through zen-4-4 have delivered:
- Engine singleton with status constants (`gc_status`)
- `create_performance` + `snapshot_score`
- `resolve_params` (three-level JSON merge)
- `dispatch_step` + `poll_step_status`

The private methods `evaluate_gate`, `evaluate_prereq_gate`, and `advance_loop` are currently stubs in `zcl_en_orch_engine.clas.abap`. This story implements them.

The health check query (`zcl_en_orch_health_chk_query`) currently has `GATE_EVALUATION` as a GREY stub. This story replaces it with a real GREEN check. A new `LOOP_ADVANCEMENT` GREEN check is also added.

Unit tests are added in `zcl_en_orch_health_chk_query.clas.testclasses.abap` (the consolidated test location after the engine test class was deleted in zen-4-4 code review).

## Scope

### In scope
- `evaluate_gate` implementation (GATE elem_type: mark COMPLETED when all group STEP members done)
- `advance_loop` implementation (LOOP elem_type: increment iteration or complete; mirror to END_LOOP)
- `evaluate_prereq_gate` — remains a stub (Phase 2 / BSR cross-performance prerequisite resolution)
- `GATE_EVALUATION` health check: replace GREY stub with real active check
- `LOOP_ADVANCEMENT` health check: new active GREEN check
- Unit tests for both new checks

### Out of scope
- `advance_performance` — implemented in zen-4-6
- `sweep_all` — implemented in zen-4-6
- `evaluate_prereq_gate` full implementation — Phase 2 (BSR deferred)

## Acceptance Criteria

### AC1 — evaluate_gate: all group steps COMPLETED → gate becomes COMPLETED

**Given** a performance has:
- A GATE step with REF_ID = `'GRP_A'` and STATUS = `'P'` (PENDING)
- Two STEP elements with REF_ID = `'GRP_A'` and STATUS = `'C'` (COMPLETED)

**When** `evaluate_gate` is called with the GATE step

**Then** the GATE step STATUS in `ZEN_ORCH_P_STEP` is updated to `'C'` (COMPLETED)

### AC2 — evaluate_gate: any group step still running → gate remains PENDING

**Given** a performance has:
- A GATE step with REF_ID = `'GRP_A'` and STATUS = `'P'`
- One STEP with REF_ID = `'GRP_A'` STATUS = `'C'`
- One STEP with REF_ID = `'GRP_A'` STATUS = `'R'` (RUNNING)

**When** `evaluate_gate` is called

**Then** the GATE step STATUS remains `'P'` (blocking run-until-blocked)

### AC3 — advance_loop: iteration < max → reset inner steps and increment

**Given** a LOOP step with MAX_PARALLEL = 3 (iteration count), current LOOP_ITERATION = 1, STATUS = `'P'`
**And** all inner STEP elements for that LOOP (same REF_ID, iteration 1) are STATUS = `'C'`

**When** `advance_loop` is called

**Then** LOOP_ITERATION is incremented to 2
**And** new `ZEN_ORCH_P_STEP` rows are inserted for the inner steps at LOOP_ITERATION = 2 with STATUS = `'P'`
**And** the LOOP step row at iteration 0 remains STATUS = `'P'` (still in progress — not done yet)
**And** the END_LOOP step at iteration 0 remains STATUS = `'P'`

### AC4 — advance_loop: iteration exhausted → LOOP and END_LOOP become COMPLETED

**Given** a LOOP step with MAX_PARALLEL = 2 (iteration count), LOOP_ITERATION = 2, STATUS = `'P'`
**And** all inner STEP elements for iteration 2 are STATUS = `'C'`

**When** `advance_loop` is called

**Then** the LOOP step STATUS is updated to `'C'`
**And** the END_LOOP step STATUS is updated to `'C'`

### AC5 — evaluate_prereq_gate remains a stub

`evaluate_prereq_gate` body contains a comment indicating it is deferred to Phase 2 and does nothing.

### AC6 — Health check: GATE_EVALUATION goes GREEN

The `GATE_EVALUATION` row in `build_rows` is no longer a GREY stub but an active check method `check_gate_evaluation`. When run against the engine with MOCK adapter:
- Creates a score with two STEP elements and one GATE (all REF_ID = `'GRP_A'`)
- Creates a performance
- Manually sets the two STEP rows to STATUS = `'C'`
- Calls `evaluate_gate` with the GATE step
- Asserts GATE STATUS = `'C'` in DB

### AC7 — Health check: LOOP_ADVANCEMENT GREEN check added

A new `check_loop_advancement` method is added. When run:
- Creates a score with a LOOP (MAX_PARALLEL=2) + one inner STEP (REF_ID = `'LP1'`) + END_LOOP
- Creates a performance
- Sets the inner STEP to STATUS = `'C'` at iteration 0
- Calls `advance_loop` for the LOOP step at iteration 0
- Asserts: new inner STEP row at iteration 1 exists with STATUS = `'P'`
- Then sets the iteration-1 inner STEP to STATUS = `'C'`
- Calls `advance_loop` again (iteration 1, max=2 → exhausted)
- Asserts: LOOP step STATUS = `'C'`, END_LOOP step STATUS = `'C'`

### AC8 — Unit tests pass

`test_gate_evaluation` and `test_loop_advancement` methods in `ltcl_health_chk_query` both assert GREEN (no RED).

### AC9 — No COMMIT WORK inside helpers

`evaluate_gate` and `advance_loop` must not call `COMMIT WORK`. All commits are done in `sweep_all` (zen-4-6).

### AC10 — abaplint clean (120-char line limit)

All modified files pass abaplint with no line-length violations.

## Technical Notes

### evaluate_gate

```
SELECT * FROM zen_orch_p_step
  WHERE perf_uuid = is_perf_step-perf_uuid
    AND ref_id    = is_perf_step-ref_id
    AND elem_type = 'STEP'
  INTO TABLE @DATA(lt_group_steps).

" All must be COMPLETED — if any not, return without updating
LOOP AT lt_group_steps INTO DATA(ls).
  IF ls-status <> gc_status-completed.
    RETURN.
  ENDIF.
ENDLOOP.

UPDATE zen_orch_p_step SET status = @gc_status-completed
  WHERE perf_uuid = @is_perf_step-perf_uuid
    AND score_seq = @is_perf_step-score_seq
    AND loop_iteration = @is_perf_step-loop_iteration.
```

### advance_loop

MAX_PARALLEL is stored in `zen_orch_s_step` (score definition), not in `zen_orch_p_step`. The current LOOP_ITERATION is taken from `is_perf_step-loop_iteration`.

Lookup: get `score_id` from `zen_orch_perf` for this `perf_uuid`, then get `max_parallel` from `zen_orch_s_step` for that `score_id + score_seq`.

If `loop_iteration + 1 < max_parallel`:
1. Increment iteration counter (held in next_iter = loop_iteration + 1)
2. Read the template inner steps (loop_iteration = 0, same REF_ID as LOOP step, elem_type = 'STEP')
   from `zen_orch_p_step`
3. INSERT new rows for each at `loop_iteration = next_iter`, STATUS = 'P'

If `loop_iteration + 1 >= max_parallel`:
1. UPDATE LOOP step STATUS = 'C'
2. UPDATE END_LOOP step STATUS = 'C' (WHERE perf_uuid = ... AND ref_id = LOOP's ref_id AND elem_type = 'END_LOOP')

### evaluate_prereq_gate

Remains a stub comment: `" Stub — BSR prerequisite resolution deferred to Phase 2`.

### LOOP step score_seq in P_STEP

The LOOP step at `score_seq = N` has `loop_iteration = 0` always (it's not a work unit — it's a control marker). Same for END_LOOP. The inner STEPs within a loop carry the `loop_iteration` counter.

The LOOP step itself has a REF_ID (e.g., `'LP1'`). The inner STEPs also carry that same REF_ID. The END_LOOP step also carries that REF_ID.

## Constitution Compliance

- **Principle I**: All types via DDIC (`zen_orch_s_perf_step`, `zen_orch_p_step`)
- **Principle II**: SAP naming, ≤120 chars, no inline comments between methods
- **Principle III**: SAP ABAP open SQL syntax verified (UPDATE ... SET ... WHERE)
- **Principle IV**: No direct NEW — engine is singleton via `get_instance`; no adapter factory calls needed in gate/loop
- **Principle V**: `ZCX_EN_ORCH_ERROR` propagated for structural errors (missing score row); gate/loop themselves don't raise — silently return when not ready

## Files to Modify

| File | Change |
|------|--------|
| `src/zcl_en_orch_engine.clas.abap` | Implement `evaluate_gate`, `advance_loop`; update `evaluate_prereq_gate` stub comment |
| `src/zcl_en_orch_health_chk_query.clas.abap` | Replace GATE_EVALUATION grey stub with `check_gate_evaluation`; add `check_loop_advancement` |
| `src/zcl_en_orch_health_chk_query.clas.testclasses.abap` | Add `test_gate_evaluation` and `test_loop_advancement` |

## Definition of Done

- [ ] `evaluate_gate` implemented and handles both AC1 and AC2
- [ ] `advance_loop` implemented and handles both AC3 and AC4
- [ ] `evaluate_prereq_gate` stub updated with Phase 2 comment
- [ ] `GATE_EVALUATION` health check is active GREEN (not GREY)
- [ ] `LOOP_ADVANCEMENT` health check is active GREEN
- [ ] `test_gate_evaluation` unit test passes (GREEN)
- [ ] `test_loop_advancement` unit test passes (GREEN)
- [ ] No `COMMIT WORK` in any helper method
- [ ] abaplint passes (no 120-char violations)
- [ ] Code pushed to `cz.en.orch` main branch
- [ ] All 12 unit tests pass in SAP
