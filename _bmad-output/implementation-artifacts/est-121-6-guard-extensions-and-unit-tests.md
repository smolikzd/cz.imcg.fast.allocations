# Story EST-121.6: Guard Extensions and Unit Tests

Status: done

## Story

As a framework developer,
I want EST-102 duplicate check and EST-106 parallel limit to count *_REQUESTED statuses as active, and unit tests validating all new status guards,
so that background-scheduled instances are properly protected against over-scheduling and all status transitions are verified.

## Acceptance Criteria

1. Given an EXEC_REQUESTED instance with the same hash, when `create_process()` is called, then `zcx_fi_process_error=>duplicate_instance` is raised. (Tech spec AC13)
2. Given a RESTART_REQUESTED instance with the same hash, when `create_process()` is called, then `zcx_fi_process_error=>duplicate_instance` is raised.
3. Given EXEC_REQUESTED instances exist for the same process type, when the parallel limit is reached (counting RUNNING + EXEC_REQUESTED + RESTART_REQUESTED), then `create_process()` raises `parallel_limit_exceeded`. (Tech spec AC14)
4. Unit test: `request_execute()` rejects non-NEW instances (guard-only test).
5. Unit test: `request_restart()` rejects non-FAILED/CANCELLED instances (guard-only test).
6. Unit test: `cancel()` accepts EXEC_REQUESTED status.
7. Unit test: `cancel()` accepts RESTART_REQUESTED status.
8. Unit test: `restart()` accepts CANCELLED status.
9. Unit test: duplicate check blocks EXEC_REQUESTED.
10. Unit test: duplicate check blocks RESTART_REQUESTED.
11. All existing unit tests still pass (no regressions).

## Tasks / Subtasks

- [x] Task 15: Extend EST-102 duplicate check to block *_REQUESTED statuses (AC: #1, #2)
  - [x] Open `src/zcl_fi_process_manager.clas.abap`
  - [x] Find `create_process()` duplicate check — the `line_exists()` chain (after EST-110 changes)
  - [x] Current: checks RUNNING, FAILED, COMPLETED
  - [x] Add: `line_exists( lt_existing[ status = gc_status-exec_requested ] )` and `line_exists( lt_existing[ status = gc_status-restart_requested ] )`
  - [x] Update the error message value string to include new statuses
  - [x] Break each `line_exists()` onto its own line to stay under 120 chars

- [x] Task 16: Extend EST-106 parallel limit to count *_REQUESTED statuses (AC: #3)
  - [x] Find the parallel instance count query (EST-106 implementation: `check_parallel_limit()` or similar)
  - [x] Current: counts WHERE status = 'RUNNING' (or uses a single-value check)
  - [x] Replace with RANGE table approach:
    ```
    DATA lt_active_statuses TYPE RANGE OF zfi_process_status.
    lt_active_statuses = VALUE #(
      ( sign = 'I' option = 'EQ' low = gc_status-running )
      ( sign = 'I' option = 'EQ' low = gc_status-exec_requested )
      ( sign = 'I' option = 'EQ' low = gc_status-restart_requested ) ).
    SELECT COUNT(*) FROM zfi_proc_inst INTO @DATA(lv_active_count)
      WHERE process_type = @iv_process_type AND status IN @lt_active_statuses.
    ```
  - [x] Adapt to actual EST-106 implementation pattern (the above is illustrative)

- [x] Task 17: Unit tests for new methods (AC: #4-#11)
  - [x] Open `src/zcl_fi_process_manager.clas.testclasses.abap`
  - [x] Add test methods (guard-only — no actual APJ scheduling):
    - `test_req_exec_rejects_running` — create instance, set to RUNNING, call request_execute(), assert exception
    - `test_req_restart_rejects_new` — create instance (NEW), call request_restart(), assert exception
    - `test_cancel_exec_requested` — create instance, set to EXEC_REQUESTED, call cancel(), assert success (status = CANCELLED)
    - `test_cancel_restart_requested` — create instance, set to RESTART_REQUESTED, call cancel(), assert success
    - `test_restart_from_cancelled` — create instance, set to CANCELLED + FAILED step, call restart(), assert success
    - `test_blocks_when_exec_requested` — create instance (EXEC_REQUESTED), call create_process() with same hash, assert duplicate exception
    - `test_blocks_when_restart_req` — same for RESTART_REQUESTED
  - [x] Follow existing test patterns: `FOR TESTING DURATION SHORT RISK LEVEL HARMLESS`, Given/When/Then with `cl_abap_unit_assert`
  - [x] Clean up test data in teardown (DELETE zfi_proc_inst + COMMIT WORK AND WAIT)
  - [x] Run all existing tests to verify no regressions

## Dev Notes

### Target Repository
`cz.imcg.fast.planner`

### Constitution Compliance
- **Principle II (SAP Standards):** Test methods follow existing naming/documentation patterns. Line length ≤120 chars.
- **Principle III (Consult SAP Docs):** ABAP Unit test framework (FOR TESTING DURATION SHORT RISK LEVEL HARMLESS).
- **Principle V (Error Handling):** Guard tests verify correct exception type and textid.

### Dependencies
- **Depends on:** All previous EST-121 stories (1-5)
- **Blocks:** Nothing — this is the final story

### Technical Decisions
- **Guard-only tests:** Tests focus on status guard validation, NOT actual APJ scheduling. CL_APJ_RT_API requires APJ infrastructure which is not available in unit test context.
- **Two test options for future:** (1) Guard-only as implemented here, (2) APJ mock via test doubles for integration-level testing (deferred).
- **RANGE table for parallel count:** More flexible than chaining individual WHERE conditions. Aligns with EST-110 pattern (cleanup_old_instances uses RANGE).

### Files to Modify
| File | Action |
|------|--------|
| `src/zcl_fi_process_manager.clas.abap` | Modify — extend duplicate check and parallel limit |
| `src/zcl_fi_process_manager.clas.testclasses.abap` | Modify — add 7 new test methods |

### Codebase Patterns
- Duplicate check (create_process, lines ~286-293): `IF line_exists( lt_existing[ status = ... ] ) OR ...`
- Parallel limit (check_parallel_limit): SELECT COUNT with WHERE status condition
- Test pattern: `FOR TESTING DURATION SHORT RISK LEVEL HARMLESS`, `cl_abap_unit_assert=>assert_equals()`, `cl_abap_unit_assert=>fail()`
- Test cleanup: `DELETE FROM zfi_proc_inst WHERE ...` + `COMMIT WORK AND WAIT`
- Existing test methods reference: test_blocks_when_completed, test_allows_when_superseded, test_double_supersede (from EST-110)

### Previous Story Intelligence
- **EST-110 test patterns:** test_blocks_when_completed creates an instance, sets status, then calls create_process() and asserts duplicate_instance exception. Follow this exact pattern.
- **fix-proc-test-cleanup (commit 341184d):** Test cleanup must use DELETE + COMMIT WORK AND WAIT in teardown, not just local cleanup.
- **CONV string() type-compat fix (commit a137f1b):** When raising exceptions with mixed types, use explicit CONV to avoid type-compat errors.

### Critical Warnings
1. **Actual EST-106 implementation may differ from spec illustration:** The tech spec provides an illustrative RANGE query. Verify the actual EST-106 code pattern and adapt accordingly.
2. **test_restart_from_cancelled requires a FAILED step:** The restart() method looks for a step with status = FAILED. A CANCELLED instance without any FAILED step will raise no_failed_step. The test must set up both the instance status (CANCELLED) and at least one step with FAILED status.
3. **No regression:** Run ALL existing tests after changes — especially EST-102, EST-106, and EST-110 tests.

### References
- [Source: tech-spec-est-121-apj-background-execution-lifecycle.md#Tasks 15-17]
- [Source: tech-spec-est-121-apj-background-execution-lifecycle.md#Technical Decision 9]
- [Source: tech-spec-est-121-apj-background-execution-lifecycle.md#Testing Strategy]

## Dev Agent Record

### Agent Model Used
claude-opus-4.6 via OpenCode

### Debug Log References
- Open SQL host expression error: method calls cannot be used in WHERE clauses (commit 68eaa29)
- ABAP-Doc syntax errors inside method bodies (commit 13d1484)
- RAP query page size overrun dump: IF_RAP_QUERY_PAGING is an interface, not a structure (commit 810f606)
- VALUE FOR iteration with variable bounds not supported on this release (commit d38e577)
- Per-test cleanup essential: all 7 APJ guard tests share TEST_DUPCHK process type with ALLOW_DUPLICATE = ' ' (commit 6e75128)

### Completion Notes List
- Tasks 15-16 implemented in zcl_fi_process_manager (duplicate check + parallel limit extensions)
- Task 17 tests relocated from manager testclasses to zcl_fiproc_health_chk_query (dual-purpose: RAP health check UI + ABAP Unit)
- 7 APJ guard tests: check_apj_req_exec_rejects_run, check_apj_req_restart_rej_new, check_apj_cancel_exec_requested, check_apj_cancel_restart_req, check_apj_restart_from_cancelled, check_apj_dupchk_exec_req, check_apj_dupchk_restart_req
- All 27 health check rows visible in Fiori UI (maxItems bumped to 50)
- All 7 APJ guard unit tests pass in ltcl_apj_status_guards
- All existing tests pass — no regressions
- 9 commits total for this story (iterative fixes for SAP platform constraints)

### File List
| File | Action |
|------|--------|
| `src/zcl_fi_process_manager.clas.abap` | Modified — extended duplicate check + parallel limit for EXECREQ/RESTREQ |
| `src/zcl_fiproc_health_chk_query.clas.abap` | Modified — added 7 check_apj_* methods with per-test cleanup + paging support |
| `src/zcl_fiproc_health_chk_query.clas.testclasses.abap` | Modified — added ltcl_apj_status_guards with 7 unit test methods |
| `src/zfiproc_i_health_chk_ce.ddls.asddls` | Modified — added maxItems: 50 to presentationVariant |
