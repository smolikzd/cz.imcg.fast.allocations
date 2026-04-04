# Story zen-4-4: Implement Sequential Step Dispatch and Polling

Status: review

## Story

**Story ID:** zen-4-4
**Epic ID:** zen-epic-4
**Title:** Implement sequential step dispatch and polling
**Target Repository:** cz.en.orch (`/Users/smolik/DEV/cz.en.orch`)
**Depends On:** zen-4-3 (resolve_params complete)
**Constitution Principles:** I (DDIC-First), II (SAP Standards), III (Consult SAP Docs), IV (Factory Pattern), V (Error Handling)

As an engine,
I want to dispatch a STEP element to its adapter and poll its status on subsequent sweeps,
so that the engine advances sequential work units through their lifecycle (PENDING → RUNNING → COMPLETED/FAILED).

## Acceptance Criteria

- [x] AC1: Given a performance step with ELEM_TYPE = 'STEP' and STATUS = 'P' (PENDING), when `dispatch_step` is called for that step, the adapter factory creates the correct adapter for the step's ADAPTER_TYPE, `adapter->start( iv_params_json = <resolved params> )` is called, the returned handle is stored in `ZEN_ORCH_P_STEP.WORK_UNIT_HANDLE`, and the step STATUS is updated to the status returned by `start` (COMPLETED if synchronous, RUNNING if async).
- [x] AC2: Given a step is in STATUS = 'R' (RUNNING) with a non-empty WORK_UNIT_HANDLE, when `poll_step_status` is called for that step, `adapter->get_status( iv_handle )` is called and the step STATUS is updated to the returned status.
- [x] AC3: Given `get_status` raises `ZCX_EN_ORCH_ERROR`, when `poll_step_status` catches it, the step remains RUNNING and a warning is logged (NFR3: resilient polling — transient errors do not fail the performance).
- [x] AC4: Given `adapter->start` raises `ZCX_EN_ORCH_ERROR`, when `dispatch_step` propagates the exception, the caller (engine) marks the performance as FAILED (fail-stop, NFR4).
- [x] AC5: ABAP Unit tests cover AC1 (dispatch with MOCK adapter → step COMPLETED), AC2 (poll_step_status for RUNNING step → COMPLETED), and AC3 (transient get_status error → step stays RUNNING, no exception propagated); all tests pass.

## Tasks / Subtasks

- [x] T1: Implement `dispatch_step` private method in `zcl_en_orch_engine.clas.abap` (AC: 1, 4)
  - [x] T1.1: Use `zcl_en_orch_adapter_factory=>create( is_perf_step-adapter_type )` to get the adapter instance (AC: 1)
  - [x] T1.2: Call `resolve_params( iv_perf_uuid = is_perf_step-perf_uuid is_perf_step = is_perf_step )` to get the resolved JSON for the step (AC: 1)
  - [x] T1.3: Call `lo_adapter->start( iv_params_json = lv_params_json )` to dispatch the work unit (AC: 1)
  - [x] T1.4: Update `ZEN_ORCH_P_STEP`: set `WORK_UNIT_HANDLE = rs_result-work_unit_handle` and `STATUS = rs_result-status` (AC: 1)
  - [x] T1.5: Let `ZCX_EN_ORCH_ERROR` from `start` propagate unhandled (caller — advance_performance — will mark performance FAILED) (AC: 4)
- [x] T2: Implement `poll_step_status` private method in `zcl_en_orch_engine.clas.abap` (AC: 2, 3)
  - [x] T2.1: Use `zcl_en_orch_adapter_factory=>create( is_perf_step-adapter_type )` to get the adapter instance (AC: 2)
  - [x] T2.2: Call `lo_adapter->get_status( iv_handle = is_perf_step-work_unit_handle )` in a TRY/CATCH block (AC: 2, 3)
  - [x] T2.3: On success: update `ZEN_ORCH_P_STEP` STATUS to the returned status (AC: 2)
  - [x] T2.4: On `ZCX_EN_ORCH_ERROR` catch: log a warning (using `ZCL_EN_ORCH_LOGGER`) and do NOT re-raise — step remains RUNNING (AC: 3)
- [x] T3: Add test methods to `ZCL_EN_ORCH_ENGINE_TEST` for `dispatch_step` and `poll_step_status` (AC: 5)
  - [x] T3.1: Ensure MOCK adapter is registered in `ZEN_ORCH_ADPT_R` in setup (or ensure test setup inserts row if missing)
  - [x] T3.2: Test `dispatch_step_completes_mock`: Create performance, call `dispatch_step` for a PENDING STEP with MOCK adapter, assert step STATUS = 'C' and WORK_UNIT_HANDLE starts with 'MOCK:' (AC: 1)
  - [x] T3.3: Test `poll_step_status_completed`: Create performance, manually set a step to STATUS='R' with a handle, call `poll_step_status`, assert step STATUS = 'C' (AC: 2)
  - [x] T3.4: Test `poll_step_status_resilient`: Use unregistered adapter type 'MOCK_FAIL' that raises on factory create; call `poll_step_status`; assert no exception propagated and step stays RUNNING (AC: 3)

## Dev Notes

### Key Design Decisions

**dispatch_step:**
- The method receives `is_perf_step TYPE zen_orch_s_perf_step` — contains all needed fields: `PERF_UUID`, `SCORE_SEQ`, `LOOP_ITERATION`, `ADAPTER_TYPE`, `PARAMS_JSON`, `STATUS`, `WORK_UNIT_HANDLE`.
- `resolve_params` is called first to merge score-level + step-level params.
- After `start`, the step row in `ZEN_ORCH_P_STEP` is updated using the primary key: `PERF_UUID + SCORE_SEQ + LOOP_ITERATION`.
- `ZCX_EN_ORCH_ERROR` from `start` propagates unhandled — the caller (`advance_performance`, zen-4-6) catches it and marks the performance FAILED. This is the fail-stop contract (NFR4).
- `ZCX_EN_ORCH_ERROR` from `resolve_params` also propagates — same fail-stop contract.

**poll_step_status:**
- The method is resilient: `get_status` errors are caught and logged as warnings, not re-raised (NFR3).
- A logger instance is needed. For now, create via `ZCL_EN_ORCH_LOGGER=>create( )` inside the method or get it from an engine attribute. **Decision:** Create a logger inline (no engine-level logger attribute in scope for this story). Since `zcl_en_orch_logger` uses `create( )` factory method, call it inside the catch block.
- If the adapter type is not registered (`zcx_en_orch_error` from factory), that is caught by the same TRY/CATCH — this is acceptable for poll (step stays RUNNING; will retry next sweep).

**Logger usage pattern:**

```abap
METHOD poll_step_status.
  DATA lo_adapter TYPE REF TO zif_en_orch_adapter.
  DATA lv_new_status TYPE zen_orch_de_status.

  " T2.1 — Get adapter from factory
  TRY.
      lo_adapter = zcl_en_orch_adapter_factory=>create( is_perf_step-adapter_type ).
      " T2.2 — Poll current status
      lv_new_status = lo_adapter->get_status( is_perf_step-work_unit_handle ).
    CATCH zcx_en_orch_error INTO DATA(lx_poll).
      " T2.4 — NFR3: Transient errors must not fail the performance — log + return
      DATA(lo_logger) = zcl_en_orch_logger=>create( ).
      lo_logger->log_warning(
        iv_text = |poll_step_status: adapter error for step { is_perf_step-score_seq }: | &&
                  lx_poll->get_text( )
      ).
      RETURN.
  ENDTRY.

  " T2.3 — Update step status in DB
  UPDATE zen_orch_p_step
    SET status = @lv_new_status
    WHERE perf_uuid      = @is_perf_step-perf_uuid
      AND score_seq      = @is_perf_step-score_seq
      AND loop_iteration = @is_perf_step-loop_iteration.
ENDMETHOD.
```

**dispatch_step implementation pattern:**

```abap
METHOD dispatch_step.
  " T1.1 — Get adapter from factory (let ZCX_EN_ORCH_ERROR propagate)
  DATA(lo_adapter) = zcl_en_orch_adapter_factory=>create( is_perf_step-adapter_type ).

  " T1.2 — Resolve parameters (three-level merge)
  DATA(lv_params_json) = resolve_params(
    iv_perf_uuid = is_perf_step-perf_uuid
    is_perf_step = is_perf_step
  ).

  " T1.3 — Dispatch to adapter (let ZCX_EN_ORCH_ERROR propagate — fail-stop)
  DATA(ls_result) = lo_adapter->start( lv_params_json ).

  " T1.4 — Persist result: handle + status
  UPDATE zen_orch_p_step
    SET work_unit_handle = @ls_result-work_unit_handle
        status           = @ls_result-status
    WHERE perf_uuid      = @is_perf_step-perf_uuid
      AND score_seq      = @is_perf_step-score_seq
      AND loop_iteration = @is_perf_step-loop_iteration.
ENDMETHOD.
```

### ZCL_EN_ORCH_LOGGER Interface

Check `zcl_en_orch_logger.clas.abap` for the `log_warning` method signature. If `log_warning` accepts `iv_text TYPE string`, use that. If it has a different signature (e.g., from `zif_en_orch_logger`), adapt accordingly.

### Test Implementation for AC3 (Resilient Polling)

AC3 requires testing that a transient `get_status` error does NOT propagate. Options:
1. Register a "failing adapter" in `ZEN_ORCH_ADPT_R` with `IMPL_CLASS = 'ZCL_EN_ORCH_ADAPTER_MOCK_FAIL'` — but this class does not exist yet.
2. Use the MOCK adapter but test with an invalid adapter type (factory raises `adapter_not_registered`) — this exercises the catch path in `poll_step_status`.

**Recommended: Option 2 — Use an unregistered adapter type for AC3 test.**

Set the step's `ADAPTER_TYPE` to `'MOCK_FAIL'` (not registered) and call `poll_step_status`. The factory will raise `adapter_not_registered`, which should be caught in the method and logged, while the step remains RUNNING.

```abap
METHOD poll_step_status_resilient.
  " Set up a RUNNING step with an unregistered adapter type
  DATA(lv_uuid) = mo_engine->create_performance( ... ).
  mv_created_perf_uuid = lv_uuid.

  " Manually force step 10 to RUNNING with unregistered adapter type
  UPDATE zen_orch_p_step
    SET status       = 'R'
        adapter_type = 'MOCK_FAIL'
        work_unit_handle = 'TEST_HANDLE'
    WHERE perf_uuid      = lv_uuid
      AND score_seq      = 10
      AND loop_iteration = 0.

  " Build step structure with the updated values
  DATA(ls_step) = VALUE zen_orch_s_perf_step(
    perf_uuid        = lv_uuid
    score_seq        = 10
    loop_iteration   = 0
    elem_type        = 'STEP'
    adapter_type     = 'MOCK_FAIL'
    status           = 'R'
    work_unit_handle = 'TEST_HANDLE'
  ).

  " Call poll_step_status — must NOT raise, step must stay RUNNING
  TRY.
      mo_engine->poll_step_status( ls_step ).
    CATCH zcx_en_orch_error INTO DATA(lx).
      cl_abap_unit_assert=>fail(
        msg = |poll_step_status must not propagate exception: { lx->get_text( ) }|
      ).
  ENDTRY.

  " Assert step is still RUNNING
  SELECT SINGLE status FROM zen_orch_p_step
    WHERE perf_uuid = lv_uuid AND score_seq = 10 AND loop_iteration = 0
    INTO @DATA(lv_status).
  cl_abap_unit_assert=>assert_equals(
    exp = 'R'
    act = lv_status
    msg = 'Step must remain RUNNING after transient get_status error'
  ).
ENDMETHOD.
```

### Test Setup: MOCK Adapter Registration

The MOCK adapter must be in `ZEN_ORCH_ADPT_R` for `dispatch_step` tests. The test `setup` method should insert a MOCK row if it does not exist, and `teardown` should delete it.

```abap
" In setup:
INSERT zen_orch_adpt_r FROM @( VALUE #(
  adapter_type = 'MOCK'
  impl_class   = 'ZCL_EN_ORCH_ADAPTER_MOCK'
) ) ACCEPTING DUPLICATE KEYS.

" In teardown:
DELETE FROM zen_orch_adpt_r WHERE adapter_type = 'MOCK'.
```

**Note:** Using `ACCEPTING DUPLICATE KEYS` prevents failure if the row already exists in the system. In teardown, deleting it might remove a system-level registration — this is acceptable for unit tests in a dev/test system.

**Alternative:** Check if row exists before inserting; restore original state in teardown. For simplicity, use INSERT with ACCEPTING DUPLICATE KEYS and skip delete in teardown (rely on test isolation through score/perf cleanup).

### ZCL_EN_ORCH_LOGGER create method

Check the actual signature. If `ZCL_EN_ORCH_LOGGER=>create( )` requires a BAL object/subobject, add them. The BAL object is `ZEN_ORCH` (per Epic 2 story). Typical pattern:

```abap
DATA(lo_logger) = zcl_en_orch_logger=>create(
  iv_object    = 'ZEN_ORCH'
  iv_subobject = 'ENGINE'
).
```

Check the actual class definition to get the exact signature.

### Files to Modify / Create

| File | Action | Location |
|------|--------|----------|
| `zcl_en_orch_engine.clas.abap` | Implement `dispatch_step` and `poll_step_status` methods (replace stubs) | `src/` |
| `zcl_en_orch_engine_test.clas.abap` | Add 3 test method declarations and implementations | `src/` |

### Project Structure Notes

- Target repository: `/Users/smolik/DEV/cz.en.orch/`
- All source files at flat `src/` level
- abaplint enforces 120-char line limit — keep all lines ≤ 120 chars
- Primary key for `ZEN_ORCH_P_STEP`: `MANDT + PERF_UUID + SCORE_SEQ + LOOP_ITERATION`
- `zen_orch_s_perf_step` structure fields: PERF_UUID, SCORE_SEQ, LOOP_ITERATION, ELEM_TYPE, REF_ID, ADAPTER_TYPE, PARAMS_JSON, STATUS, WORK_UNIT_HANDLE

### Constitution Compliance

- **Principle I — DDIC-First**: All types used in method parameters (`zen_orch_s_perf_step`, `zen_orch_de_status`) are DDIC. No local cross-boundary type definitions.
- **Principle II — SAP Standards**: Line length ≤ 120 chars; ABAP-Doc already present on stubs from Story 4.1.
- **Principle III — Consult SAP Docs**: `zcl_en_orch_adapter_factory=>create` pattern confirmed from Story 3.2. `UPDATE` with WHERE clause is standard ABAP SQL.
- **Principle IV — Factory Pattern**: Adapter obtained via `zcl_en_orch_adapter_factory=>create( )` — no direct NEW for adapter.
- **Principle V — Error Handling**: `dispatch_step` propagates `ZCX_EN_ORCH_ERROR` (fail-stop, NFR4). `poll_step_status` catches and logs transient errors (NFR3). Both behaviors are spec-mandated.

### References

- `_bmad-output/planning-artifacts/epics.md` — Story 4.4 ACs (authoritative spec)
- `_bmad-output/planning-artifacts/architecture.md` — §NFR3 (resilient polling), §NFR4 (fail-stop)
- `/Users/smolik/DEV/cz.en.orch/src/zcl_en_orch_engine.clas.abap` — `dispatch_step` and `poll_step_status` stubs (lines 260–267)
- `/Users/smolik/DEV/cz.en.orch/src/zcl_en_orch_adapter_factory.clas.abap` — factory `create` method
- `/Users/smolik/DEV/cz.en.orch/src/zcl_en_orch_adapter_mock.clas.abap` — MOCK adapter (completes synchronously)
- `/Users/smolik/DEV/cz.en.orch/src/zcl_en_orch_engine_test.clas.abap` — existing test class to extend
- `/Users/smolik/DEV/cz.en.orch/src/zen_orch_p_step.tabl.xml` — `ZEN_ORCH_P_STEP` fields
- `/Users/smolik/DEV/cz.en.orch/src/zen_orch_s_perf_step.tabl.xml` — `ZEN_ORCH_S_PERF_STEP` structure fields

## Dev Agent Record

### Agent Model Used

github-copilot/claude-sonnet-4.6

### Debug Log References

- Fixed `lo_adapter->zif_en_orch_adapter~start(...)` → `lo_adapter->start(...)`: abaplint `check_syntax` cannot resolve explicit interface-prefix calls on interface-typed references; direct method call is correct.
- `log_warning` parameter is `iv_detail TYPE string` (not `iv_text`) — confirmed from `zif_en_orch_logger` interface.
- `ACCEPTING DUPLICATE KEYS` used in `setup` for `INSERT zen_orch_adpt_r` — safe if MOCK row already present in system.

### Completion Notes List

- `dispatch_step`: obtains adapter via factory, calls `resolve_params` for three-level JSON merge, calls `adapter->start`, persists handle + status in `ZEN_ORCH_P_STEP`. ZCX_EN_ORCH_ERROR propagates (fail-stop, NFR4).
- `poll_step_status`: wraps factory create + `adapter->get_status` in TRY/CATCH; on error logs warning via `ZCL_EN_ORCH_LOGGER` and returns silently (NFR3 resilience); on success updates step STATUS in DB.
- Three test methods added: `dispatch_step_completes_mock` (AC1), `poll_step_status_completed` (AC2), `poll_step_status_resilient` (AC3). MOCK adapter registered via `INSERT ... ACCEPTING DUPLICATE KEYS` in `setup`.

### File List

| File | Repository | Change |
|------|-----------|--------|
| `src/zcl_en_orch_engine.clas.abap` | cz.en.orch | Implemented `dispatch_step` and `poll_step_status` (lines ~260–313) |
| `src/zcl_en_orch_engine_test.clas.abap` | cz.en.orch | Added 3 test methods + MOCK adapter registration in setup |

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-04 | Story file created | Dev Agent |
| 2026-04-04 | dispatch_step and poll_step_status implemented; 3 tests added; abaplint validated | Dev Agent |
