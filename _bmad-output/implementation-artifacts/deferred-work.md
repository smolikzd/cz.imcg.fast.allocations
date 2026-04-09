# Deferred Work

Items surfaced during review that are not caused by the current story but worth addressing later.

---

## 1. Performance throttling for high-volume BAL logging

- **Linear:** https://linear.app/smolikzd/issue/EST-144/improvement-bal-logger-configurable-flush-interval-for-high-volume
- **Source:** EST-120 review (edge case hunter)
- **Date:** 2026-03-20
- **Description:** With the every-message `flush_to_db()` pattern, each `message()` call triggers a DB round-trip via `save_with_2nd_connection()`. For steps that log hundreds or thousands of messages, this could degrade performance. The current spec explicitly defers this: "start with every-message flush and optimize only if performance is observed to be a problem."
- **Possible approach:** Introduce a configurable flush interval (e.g., flush every N messages or every T seconds) in `ZCL_FI_PROCESS_LOGGER`. Could use a counter field `mv_flush_interval` with a default of 1 (current behavior) that can be raised for high-volume steps.
- **Priority:** Low — only relevant if performance degradation is observed in production.

---

## 2. Server-side status validation in handler action methods

- **Linear:** https://linear.app/smolikzd/issue/EST-145/improvement-server-side-status-validation-in-dashboard-action-handlers
- **Source:** EST-134 review (Blind Hunter finding BH-2)
- **Date:** 2026-03-29
- **Description:** The handler action methods currently only check whether `ProcessInstanceId IS INITIAL` but do not validate that `ProcessStatus` matches the expected status for the action (e.g., Execute should only run when status is `NEW`). Feature control disables buttons in the UI, but a direct OData call could bypass feature control and reach the handler. The backend manager methods have their own status guards and will reject invalid transitions with `zcx_fi_process_error`, so this is defense-in-depth rather than a security gap.
- **Possible approach:** Add status validation in each handler method (e.g., `IF ls_dashboard-ProcessStatus <> 'NEW'` for executeprocess) and populate `failed`/`reported` with a descriptive message before the buffer call.
- **Priority:** Low — backend guards already prevent invalid transitions; this would improve error UX for programmatic OData callers.

---

## 3. Extract status string literals to constants

- **Linear:** https://linear.app/smolikzd/issue/EST-146/improvement-extract-process-status-string-literals-to-constants-in
- **Source:** EST-134 review (Blind Hunter finding BH-5)
- **Date:** 2026-03-29
- **Description:** Status values like `'NEW'`, `'RUNNING'`, `'FAILED'`, `'COMPLETED'`, etc. are hard-coded string literals in `get_instance_features` and potentially in handler validation. If the status domain values change, multiple locations would need updating.
- **Possible approach:** Create constants (either in the handler class or in a shared constants interface) for each status value. Alternatively, read from the ZFI_PROCESS_STATUS domain fixed values.
- **Priority:** Low — domain values are stable and unlikely to change.

---

## 4. ~~Verify message ZFI_PROCESS/030 exists~~ RESOLVED

- **Source:** EST-134 review (Edge Case Hunter finding EC-1)
- **Date:** 2026-03-29
- **Resolved:** 2026-03-31
- **Description:** The handler used `new_message( id = 'ZFI_PROCESS' number = '030' )` for the "no process instance" error. Investigation revealed message 030 exists but has the wrong text (`"Execution requested for instance &1 (job &2/&3)"`) — a logging message, not an error message. Additionally, no message variables were being passed, so placeholders rendered as blanks.
- **Resolution:** Created new message **036** (`"No process instance found for allocation &1/&2/&3/&4"`) in the ZFI_PROCESS message class (planner repo). Updated all 3 handler action methods (CancelProcess, SupersedeProcess, RestartProcess) in `zbp_fi_alloc_dashboard_ce` (ovysledovka repo) to use message 036 with `v1=CompanyCode`, `v2=FiscalYear`, `v3=FiscalPeriod`, `v4=AllocationId`.
- **Priority:** ~~High~~ Resolved.

---

## 5. Consider catching CX_ROOT in saver for unexpected exceptions

- **Linear:** https://linear.app/smolikzd/issue/EST-140/fix-cx-root-not-caught-in-dashboard-action-saver-short-dump-risk
- **Source:** EST-134 review (Edge Case Hunter finding EC-2)
- **Date:** 2026-03-29
- **Description:** The saver currently only catches `zcx_fi_process_error`. If a manager method raises an unexpected exception (e.g., `cx_sy_zerodivide`, a database exception, or an APJ scheduling error that doesn't inherit from `zcx_fi_process_error`), the exception would propagate uncaught to the RAP framework, potentially causing a short dump.
- **Possible approach:** Add a second `CATCH cx_root` after the `zcx_fi_process_error` catch to map any unexpected exception to `failed`/`reported` with a generic error message. This is defensive programming -- the manager methods should only raise `zcx_fi_process_error`, but belt-and-suspenders is prudent.
- **Priority:** Medium — unlikely but impactful if it occurs in production.

---

## 6. ~~Dashboard action to create a new process instance after supersede~~ RESOLVED

- **Linear:** https://linear.app/smolikzd/issue/EST-141/dashboard-add-createprocess-action-for-rows-after-supersedecancel
- **Source:** EST-134 implementation (state machine analysis)
- **Date:** 2026-03-30
- **Resolved:** 2026-03-31
- **Description:** After superseding a COMPLETED process instance, the SUPERSEDED status is terminal — no further actions are available on that row. The EST-110 design intent is that supersede unlocks the duplicate check so a new `create_process()` can be called. However, the dashboard currently has no "Create New Instance" action — the user must create the instance through other means. This breaks the self-contained operations cockpit workflow.
- **Resolution:** EST-136 added `CreateAndExecute` action covering both empty rows (no instance) AND rows with SUPERSEDED/CANCELLED status. This fully addresses the gap — users can create and immediately execute from the dashboard in all cases where no active instance exists. A standalone `CreateProcess` (create without execute) was considered but deemed unnecessary given the typical workflow. EST-141 closed as resolved by EST-136.
- **Priority:** ~~Medium~~ Resolved.

---

## 7. Parallel instance limit check bypassed in APJ job execution path

- **Linear:** https://linear.app/smolikzd/issue/EST-143/fix-parallel-instance-limit-bypassed-in-apj-job-execution-path
- **Source:** EST-136 pre-implementation review (edge case hunter, EC14)
- **Date:** 2026-03-30
- **Description:** The `check_parallel_limit()` method in `zcl_fi_process_manager` only runs inside `execute_process()` (manager method). However, the APJ job class calls `lo_instance->execute()` directly, bypassing the manager's parallel limit check. With the new `CreateAndExecute` dashboard action making it easier to rapidly create instances, users could exceed the configured `max_parallel_insts` limit. The APJ jobs would all fire and execute simultaneously without throttling.
- **Possible approach:** Move the parallel limit check into `zcl_fi_process_instance->execute()` itself, or add it to `request_execute_process()` at scheduling time, or add it to the APJ job's execute method before calling `lo_instance->execute()`.
- **Priority:** Low — only relevant for high-volume concurrent execution scenarios. The parallel limit was designed as a soft guardrail, not a hard constraint.

---

## 8. TOCTOU race on breakpoint toggle direction

- **Source:** Dashboard step breakpoint toggle review (blind hunter + edge case hunter)
- **Date:** 2026-03-31
- **Description:** The `togglebreakpoint` handler reads the current `Breakpoint` value from `ZFI_I_PROCESS_STEP_STATS` and pre-resolves the toggle direction (SETBP or CLEARBP). If a concurrent session toggles the same step between the handler's SELECT and the saver's `set_step_breakpoint_process()` call, both will read the same initial state, both will buffer the same action code, and the net result is two SETBP (or two CLEARBP) calls instead of toggle→toggle. The breakpoint ends up in the wrong final state with no error.
- **Possible approach:** Move the direction-resolution READ to the saver (read `zfi_proc_step` directly at save time), or add an optimistic lock check (compare expected value before update). This is a structural limitation of the buffer+saver RAP unmanaged pattern.
- **Priority:** Low — requires two users to click Toggle on the exact same step within milliseconds. Acceptable for an operations cockpit use case.

---

## 9. `ACTION` field in `ZFI_ALLOC_S_ACTION_BUFFER` lacks a DDIC data element

- **Source:** Dashboard step breakpoint toggle review (acceptance auditor)
- **Date:** 2026-03-31
- **Description:** The `ACTION` field in `ZFI_ALLOC_S_ACTION_BUFFER` is defined with raw built-in type attributes (`INTTYPE=C`, `INTLEN=20`) and has no `ROLLNAME` — i.e., no DDIC data element. Constitution Principle I requires all fields to be based on data elements. This is a pre-existing violation; the `STEP_NUMBER` field added by the breakpoint story correctly uses `ROLLNAME ZFI_PROCESS_STEP_NUMBER`.
- **Possible approach:** Create a DDIC data element `ZFI_ALLOC_ACTION_CODE` (CHAR 20, no domain needed for a free-text internal code) and apply it to the `ACTION` field.
- **Priority:** Low — internal buffer structure; no external exposure. Constitution compliance only.

---

## 10. `abap.int1` criticality fields in custom entity and query provider

- **Source:** Dashboard step breakpoint toggle review (blind hunter + acceptance auditor)
- **Date:** 2026-03-31
- **Description:** `BreakpointCriticality` and `StatusCriticality` in `ZFI_I_ALLOC_DASH_STEP_CE` use `abap.int1` directly instead of a DDIC data element. The same pattern applies to the `ty_row.statuscriticality` / `ty_row.breakpointcriticality` fields in `ZCL_FI_ALLOC_DASH_STEP_QRY`. Constitution Principle I requires DDIC-based types. The pattern is consistent across the file (pre-existing), and the spec itself prescribed `abap.int1`, but it remains non-compliant.
- **Possible approach:** Create a DDIC data element `ZFI_ALLOC_UI_CRITICALITY` (type INT1) and apply it to all criticality fields. Update the local type in the query class to use it.
- **Priority:** Low — cosmetic/compliance; criticality rendering is not affected at runtime.

---

## Deferred from: code review of spec-est-138-dashboard-duration-visibility (2026-04-01)

### 11. `CONV i()` truncates fractional seconds (floor, not round)

- **Source:** EST-138 review (blind hunter + edge case hunter)
- **Date:** 2026-04-01
- **Description:** Both `zcl_fi_alloc_dash_query.clas.abap:380` and `zcl_fi_alloc_dash_step_qry.clas.abap:340` use `CONV i( ... )` to convert packed/decfloat seconds to integer. This truncates (floor) rather than rounds. For a TotalRuntime of 1.9999999 the result is `1` not `2`. For the step elapsed case, `CL_ABAP_TSTMP=>SUBTRACT` returns `decfloat34` and the same truncation applies.
- **Possible approach:** Use `CONV i( lv_seconds + '0.5' )` or explicit `ROUND` before the conversion to get round-half-up behaviour.
- **Priority:** Low — sub-second precision is irrelevant for human-readable duration display strings.

---

## Deferred from: code review of spec-est-150-151-152-dashboard-filter-ux-and-action-confirmations (2026-04-01)

### 12. AllocationId default '1' may return empty results in some systems

- **Source:** EST-150/151/152 review (edge case hunter)
- **Date:** 2026-04-01
- **Description:** `@Consumption.filter.defaultValue: '1'` on AllocationId is hardcoded. In any system where ID 1 does not exist, the query provider's mandatory-filter guard is satisfied (a value is present) so no exception is raised, but the result set will be silently empty. Users unfamiliar with the system may assume the dashboard has no records.
- **Possible approach:** No fix needed if '1' is always the correct default in this deployment. If multi-tenant / multi-system, consider documenting that this value must be adjusted per system.
- **Priority:** Low — intentional design decision; acceptable for current deployment.

### 13. Removing FiscalYear value help leaves no re-entry guidance

- **Source:** EST-150/151/152 review (edge case hunter)
- **Date:** 2026-04-01
- **Description:** EST-150 intentionally removes `@Consumption.valueHelpDefinition` from FiscalYear. If the user clears the default and enters an invalid year, the field type `gjahr` provides implicit numeric validation but no F4 browsing. Pre-existing by design.
- **Priority:** Low — intentional per EST-150 spec.

### 14. refreshData has no IsActionCritical — intentional

- **Source:** EST-150/151/152 review (edge case hunter)
- **Date:** 2026-04-01
- **Description:** `refreshData` was explicitly excluded from confirmation dialogs per spec constraint. Not a bug.
- **Priority:** None — intentional, no action needed.

---

## Deferred from: code review of zen-2-1 and zen-2-2 (2026-04-04)

### 15. if_t100_dyn_msg implemented but msgv1–v4 never populated

- **Source:** zen-2-1 code review (blind hunter + edge case hunter)
- **Date:** 2026-04-04
- **Description:** `ZCX_EN_ORCH_ERROR` implements `if_t100_dyn_msg` but the constructor never populates `msgv1`–`msgv4`. Code that renders the exception via `MESSAGE lx TYPE 'E'` or reads `if_t100_dyn_msg~msgv1` will see empty values. T100-based display (SLG1, short dump) works correctly. Deferred: design gap inherited from pattern; no immediate runtime impact.
- **Priority:** Low — only relevant if a caller uses the dynamic message interface explicitly.

---

## Deferred from: code review of zen-3-2, zen-3-3 (2026-04-04)

### 16. Status literal 'C' not a named constant (zen-3-2, zen-3-3)

- **Source:** zen-3-2/zen-3-3 code review (Blind Hunter F5)
- **Date:** 2026-04-04
- **Description:** Status values (`'C'` for COMPLETED) are hard-coded string literals in `ZCL_EN_ORCH_ADAPTER_MOCK` and the factory error path. A named constant (e.g., `gc_status-completed`) tied to the `zen_orch_de_status` domain would make intent explicit and prevent silent breakage if the domain fixed values change.
- **Priority:** Low — domain values are stable by design; extract constants in Epic 4 when the engine introduces its own status constants.

### 17. Mock adapter does not validate handles (zen-3-3)

- **Source:** zen-3-3 code review (Edge Case Hunter F6)
- **Date:** 2026-04-04
- **Description:** `get_status`, `cancel`, `restart`, and `get_result` in `ZCL_EN_ORCH_ADAPTER_MOCK` silently succeed for any handle including empty/foreign ones. This reduces engine integration test fidelity (cannot detect missing handle propagation). Consider a configurable fault-injection mode or at minimum an `IS INITIAL` guard for unit tests in Epic 4.
- **Priority:** Low — intentional for a basic mock; revisit when Epic 4 engine unit tests are written.

### 18. `get_detail_link` no RAISING clause forecloses error reporting for real adapters (zen-3-1)

- **Source:** zen-3-3 code review (Edge Case Hunter F8)
- **Date:** 2026-04-04
- **Description:** `ZIF_EN_ORCH_ADAPTER~get_detail_link` deliberately has no RAISING clause (best-effort contract). Real adapters that do I/O to build a URL cannot raise the domain exception; they must swallow errors or return an empty string. If an adapter ever needs error reporting from this method, the interface contract must be changed — a breaking change. Document this constraint when implementing the first real adapter.
- **Priority:** Low — acceptable for current use case; document before first non-mock adapter is implemented.

### 19. `cl_system_uuid=>create_uuid_c26_static` is classic API (not released for ABAP Cloud) (zen-3-3)

- **Source:** zen-3-3 code review (Acceptance Auditor F9)
- **Date:** 2026-04-04
- **Description:** The mock uses `cl_system_uuid=>create_uuid_c26_static` which is a classic on-premise API. Project targets ABAP 7.58 (on-premise), so this is acceptable now. If the project ever migrates to ABAP Cloud / BTP, replace with `xco_cp=>uuid->version_4->as_string( )` or equivalent released API.
- **Priority:** Low — no impact on current on-premise target.

---

## Deferred from: code review of zen-4-1-create-engine-singleton-with-status-constants (2026-04-04)

### 20. Stub methods return no value/raise no exception — silent empty UUID / empty JSON

- **Source:** zen-4-1 code review (Blind Hunter + Edge Case Hunter)
- **Date:** 2026-04-04
- **Description:** `create_performance` returns initial (all-zero) UUID, `resolve_params` returns empty string, other stubs do nothing and raise no exception. Callers in any integrated test environment before 4.2–4.7 are implemented silently receive bad data. Intentional stub design.
- **Priority:** Low — will be resolved as stories 4.2–4.7 are implemented.

### 21. `sweep_all` has no RAISING clause — error propagation strategy not yet defined

- **Source:** zen-4-1 code review (Edge Case Hunter)
- **Date:** 2026-04-04
- **Description:** `sweep_all` is the APJ entry point and has no `RAISING` clause, which means `advance_performance` errors must be caught and logged internally. The catch strategy (catch-per-performance vs. catch-all) is not yet committed. The design is by-spec for the public API.
- **Priority:** Medium — the catch strategy must be defined in story 4.6 before any production use.

### 22. FINAL class with no interface is untestable via test doubles

- **Source:** zen-4-1 code review (Blind Hunter)
- **Date:** 2026-04-04
- **Description:** `ZCL_EN_ORCH_ENGINE` is `FINAL` with no interfaces, making it impossible to mock in unit tests of dependent classes. All private algorithm methods are also private with no seam. Test strategy must be addressed in story 4.6 (engine implementation) or epic 6 (integration testing).
- **Priority:** Medium — blocking for any unit test that depends on the engine singleton.

### 23. Singleton `go_instance` is never cleared — no reset path for tests or poisoned-instance recovery

- **Source:** zen-4-1 code review (Blind Hunter + Edge Case Hunter)
- **Date:** 2026-04-04
- **Description:** `CLASS-DATA go_instance` is assigned once and never reset. In APJ work-process reuse scenarios, a failed/internally-inconsistent engine state persists for the session lifetime. No `reset_instance` method exists. To be revisited in epic 6 integration testing.
- **Priority:** Low — only relevant for APJ work-process reuse or unit test isolation.

---

## Deferred from: code review of zen-4-3-implement-json-parameter-resolution (2026-04-04)

- **Non-string JSON values (numbers, booleans, nulls) silently produce empty string in resolved params.** `cl_sxml_string_reader` parses non-string value nodes differently; the `IF co_nt_value` branch is skipped, leaving the key with an empty value. Pre-existing gap; architecture §2 enforces string-only values but nothing validates at DB boundary.

- **`{{loop_iteration}}` REPLACE operates on full serialized JSON string — can corrupt a key name that matches the placeholder.** Key names are fixed snake_case per architecture §2, so this cannot occur in practice today. Worth guarding if the key contract ever relaxes.

- **Both levels empty → returns `{}` — no error, no test.** AC3 does not require an error for this case. Callers receiving `{}` should handle it; downstream behavior is not specified here.

- **sXML `element_close` node consumed via `CHECK` skip — fragile but idiomatic.** The parser loop silently discards `element_close` nodes. Works correctly for flat JSON; revisit if nested structures are ever introduced.

- **`FRIENDS zcl_en_orch_engine_test` on production class definition.** Standard SAP ABAP Unit pattern acknowledged in story Dev Notes. Creates a compile-time reference from production code to test artifact. Revisit if a cleaner test-include approach is adopted.

- **`resolve_params` SELECT SINGLE has no concurrency lock.** Read-only computation; engine transaction scope handles consistency. No lock needed for this method in isolation.

---

## Deferred from: code review of story-zen-4-6 (2026-04-05)

- **Empty inner catch swallows logger failure silently** (`sweep_all`, line 210-214). The `#EC NEEDED` suppress marker is intentional and follows the same pattern as `poll_step_status`. Not a defect; revisit if structured error observability is added to the engine.
- **LOOP new inner steps not in snapshot — not dispatched in same pass** (`advance_performance`, line 371-384). By design: after `advance_loop` clones the next iteration's steps, they are not in the `lt_steps` snapshot and will be dispatched on the next sweep. Documented in spec and code comments. Acceptable for the run-until-blocked contract.
- **UPDATE in CATCH block unguarded — lock failure leaves performance in indeterminate state** (`sweep_all`, line 207-209). If the `UPDATE zen_orch_perf SET status = FAILED` fails (lock timeout, network), `COMMIT WORK` proceeds with no status change. Resilience hardening (retry, re-check) is targeted at zen-5-x lifecycle work.
- **`check_sweep_all` cleanup_check_data called after row appended — pre-existing cleanup sequencing risk.** If `cleanup_check_data` raises, the test row is in `mt_rows` but DB state is dirty. Pre-existing pattern across all check methods. Address as part of health-check framework hardening.

## Deferred from: code review of story-zen-4-7 (2026-04-05)

- **`evaluate_gate` treats empty group (no STEP children) as fully satisfied.** When no STEP rows exist for the gate's `ref_id`, the LOOP exits without returning early, so the gate is immediately promoted to PAUSED or COMPLETED. Pre-existing structural assumption in evaluate_gate; not introduced by zen-4-7. Revisit when validating score structure on `create_performance`.
- **~~`evaluate_gate` blocks gate forever when a group step is FAILED or CANCELLED.~~ RESOLVED 2026-04-08.** `evaluate_gate` now propagates failure: FAILED group step → gate marked FAILED + error logged; CANCELLED group step → gate marked CANCELLED. `advance_performance` GATE branch now detects a FAILED gate status and raises `zcx_en_orch_error=>engine_sweep_failed` instead of setting `lv_blocked`, so the performance terminates with FAILED rather than looping indefinitely.

## Deferred from: code review of story-zen-5-2 (2026-04-06)

- **No idempotency guard — same schedule fires on every `sweep_all` within the same minute.** `is_schedule_due` evaluates only the CRON expression; it does not compare against `changed_at` to prevent re-fire within the same time window. Multiple `sweep_all` calls in the same minute create duplicate performances. Spec explicitly accepts this limitation; revisit in a future scheduling story if APJ overlap becomes a problem.
- **`changed_at` stores date only (`sy-datum`), losing sub-minute precision.** `AEDAT` field type; no timestamp companion. Future idempotency logic would require a paired `changed_time` field. Spec-compliant as written.
- **Permanently broken schedule re-fires on every sweep with no circuit-breaker.** If `create_performance` fails (SCORE_NOT_FOUND etc.), the error is caught/logged and the schedule retries indefinitely. No auto-deactivation or retry-count limit. Address in a future scheduling-reliability story.
- **`APPEND`-based Sakamoto DOW lookup table is fragile to future edits.** 12 sequential `APPEND` statements — a VALUE #(...) constructor would be safer. Low risk; refactor opportunity only.

## Deferred from: code review of zen-6-1-abap-unit-test-suite-for-engine-core (2026-04-06)

- **`sweep_all` has no score_id filter — operates on ALL system P/R performances.** Any call to `sweep_all` in a unit test advances all globally active performances, including live production data. This is a pre-existing engine design constraint (sweep is deliberately system-wide). Mitigate in integration test environments by running tests in an isolated client or ensuring no active performances exist. A future story could add a test-mode flag or scoped sweep variant.
- **Singleton engine `mo_logger` can become stale across tests.** `ZCL_EN_ORCH_ENGINE` lazy-initialises `mo_logger` once via `get_logger()`. If the BAL log handle becomes invalid after a prior test's `COMMIT WORK AND WAIT`, subsequent `log_exception()` calls in sweep error handlers silently fail. Pre-existing logger singleton design. Revisit if test diagnostics become a problem — consider resetting `mo_logger` to INITIAL in teardown or adding a logger reset method.

## Deferred from: code review of zen-6-2-integration-test-program-and-repository-finalization (2026-04-06)

- **W1: `restart_performance` sy-dbcnt=0 guard ambiguity.** The guard after `UPDATE p_step WHERE status=FAILED` cannot distinguish a concurrent reset (another process already reset the steps) from a genuine data inconsistency (no FAILED steps exist). Pre-existing design gap in concurrent-access handling. Acceptable for Phase 1 single-process context; revisit if concurrent restarts are ever supported.
- **W2: `cancel_performance` LUW split-brain.** `adapter->cancel()` is an irreversible external side effect called before the DB `UPDATE`. If a failure occurs between the adapter call and the `COMMIT WORK`, the external work unit is cancelled but the DB retains the old step status (RUNNING). This is an accepted best-effort-cancel design trade-off for Phase 1. A future story could introduce a two-phase cancel-request protocol.
- **W3: Story traceability — zen-5-2/5-3/zen-4-7 implementations bundled in zen-6-2 commit.** The implementations of `check_schedules`, `cancel_performance`, `restart_performance`, and `resume_performance` (all previously stubs) were committed as part of zen-6-2 work rather than in their respective stories. The stories are marked done, but commit hashes in sprint-status.yaml do not reference these implementations. Future sprint reviews should cross-check commit content against story deliverables.
- **W4: Named cron macros (`@yearly`, `@monthly`, etc.) silently never fire.** `matches_cron_field` passes named macros to `CONV i()`, which raises `cx_sy_conversion_no_number`, causing silent abap_false return. Out of stated Phase 1 feature scope for the cron parser. Add support or explicit validation/logging in a future cron enhancement story.
- **W5: Cron range (`1-5`) and list (`1,3,5`) syntax unhandled.** Same silent-false behaviour as W4. Standard cron syntax extensions not implemented in Phase 1. Deferred to Phase 2 cron enhancement.
- **W6: `resolve_params` local TYPE definitions (`ty_kv`, `ty_kv_map`).** Pre-existing Principle I violation from a prior story (zen-4-3 or earlier). Should be extracted to DDIC types `ZEN_ORCH_S_KV` and `ZEN_ORCH_TT_KV` in a housekeeping story.

---

## Deferred from: code review of zen-4-2-implement-performance-creation-and-score-snapshot (2026-04-08)

- **BH-4-2-3: No rollback on partial snapshot failure.** If `snapshot_score` raises after the `ZEN_ORCH_PERF` header is inserted, an orphaned PERF row with no PERF_STEP children remains in the DB. No `ROLLBACK WORK` in the caller. Priority: Low — deferred to a cleanup/lifecycle story.
- **BH-4-2-4: `CREATED_AT` stores date only (`sy-datum`, `AEDAT`).** Sub-second precision lost; cannot order within a single day. A `CREATED_TIME` companion or `TIMESTAMP` field would be needed. Priority: Low — spec-compliant as written.
- **BH-4-2-10: Unit tests use `RISK LEVEL DANGEROUS` with real DB writes.** Logic branches not covered by in-memory doubles. Acceptable for Phase 1; revisit in Epic 6. Priority: Low.
- **ECH-4-2-3: Malformed `iv_params_json` stored verbatim at creation time.** No schema validation; `resolve_params` will fail later at dispatch time. Spec-compliant; deferred to a future validation story. Priority: Low.
- **ECH-4-2-7: `SELECT * FROM zen_orch_score_step` in `snapshot_score` has no `ORDER BY` clause.** Row order is non-deterministic. PK uniqueness is unaffected; deterministic ordering aids debugging. Priority: Low.

---

## Deferred from: code review of zen-4-4-implement-sequential-step-dispatch-and-polling (2026-04-08)

- **~~BH-4-4-5: No distinction between permanent and transient adapter errors.~~ RESOLVED 2026-04-08.** `poll_step_status` now classifies `ZCX_EN_ORCH_ERROR` by `msgno`: `adapter_not_registered` (027) and `invalid_score_structure` (061) are treated as permanent — the step is immediately marked FAILED and an error is logged. All other `zcx_en_orch_error` instances remain transient (warn + RETURN). `cx_root` remains transient.
- **BH-4-4-7: Dead `RAISING zcx_en_orch_error` on `poll_step_status` signature.** By design the method never raises (NFR3 catch-and-swallow), making the RAISING clause misleading. Signature cleanup only; no behavioral impact. Priority: Low.

---

## Deferred from: code review of zen-4-5-implement-gate-evaluation-and-loop-advancement (2026-04-08)

- **F-01: `evaluate_gate` treats empty group (no STEP children) as fully satisfied.** When no STEP rows exist for the gate's `ref_id`, the gate is immediately promoted to PAUSED or COMPLETED. Pre-existing structural assumption; spec decision needed (should this be an error or a no-op?). Priority: Low — deferred to score validation story.

---

## Deferred from: code review of story-7.2 (2026-04-09)

### 23. Hash Type Fragility

- **Source:** Story 7.2 review (Blind Hunter BH-5)
- **Date:** 2026-04-09
- **Description:** `iv_params_hash` uses `CHAR 64` which is exactly the length of a hex-encoded SHA-256 hash. If the project switches to a different algorithm (e.g., SHA-512) or needs a binary hash, multiple DDIC objects and interface signatures will need breaking changes.
- **Possible approach:** Consider a domain for the hash or a more generic `STRING` / `XSTRING` type to allow algorithm agility in Phase 3.
- **Priority:** Low — SHA-256 is stable and sufficient for current requirements.

### 24. Overview Performance for Large Datasets

- **Source:** Story 7.2 review (Blind Hunter BH-6)
- **Date:** 2026-04-09
- **Description:** `get_overview` returns a full table of all BSR entries. As the system ages, this table will grow to include thousands of stale terminal registrations (terminal runs that were never cleaned up or re-used). Returning this full set to a UI dashboard will eventually cause timeouts or memory issues.
- **Possible approach:** Add filtering (e.g., return only active registrations) or implement a background cleanup job to purge terminal entries from `ZEN_ORCH_BSR` after a retention period.
- **Priority:** Low — only relevant once a significant volume of runs is reached.

---

## Deferred from: code review of zen-7-3-zcl-en-orch-bsr-implementation (2026-04-09)

### 25. N+1 SELECT Pattern in `check_collision` and `get_overview`

- **Source:** Story zen-7-3 review (Blind Hunter + Edge Case Hunter)
- **Date:** 2026-04-09
- **Description:** Both `check_collision` and `get_overview` use a loop-inside-SELECT pattern — one outer SELECT for BSR candidates, then one `SELECT SINGLE` per row to resolve live status from `ZEN_ORCH_PERF`. For small BSR tables this is acceptable, but at scale this will degrade performance.
- **Possible approach:** Replace with a single SELECT with JOIN between `ZEN_ORCH_BSR` and `ZEN_ORCH_PERF`, or use FOR ALL ENTRIES.
- **Priority:** Low — optimization only, no correctness issue; deferred until volume warrants it.

### 26. Concurrent `register` Race: No Guard on INSERT Duplicate-Key DB Error

- **Source:** Story zen-7-3 review (Edge Case Hunter)
- **Date:** 2026-04-09
- **Description:** `register` does a SELECT then INSERT without atomicity. Two concurrent calls with the same `iv_bsr_key` could both pass the SELECT guard and then the second INSERT would produce an unhandled duplicate-key DB exception. The engine's LUW is designed to be single-threaded per performance, so this is unlikely in normal operation, but is theoretically possible.
- **Possible approach:** Wrap the INSERT in `TRY...CATCH cx_sy_open_sql_error` and re-raise as `bsr_key_collision`, or use `MODIFY` with a uniqueness check.
- **Priority:** Low — engine LUW is single-threaded by design; relevant only if that assumption ever changes.

---

## Deferred from: code review of zen-7-4-engine-integration-collision-and-bsr-lifecycle (2026-04-09)

### 27. Missing secondary index on ZEN_ORCH_PERF for collision check

- **Source:** zen-7-4 review (Edge Case Hunter)
- **Date:** 2026-04-09
- **Description:** The collision check `SELECT SINGLE FROM zen_orch_perf WHERE score_id = ... AND params_hash = ... AND status NOT IN (...)` has no supporting secondary index. For systems with many concurrent performances per score, this results in a full table scan on `ZEN_ORCH_PERF` every time `create_performance` is called.
- **Possible approach:** Add a secondary index on `(SCORE_ID, PARAMS_HASH, STATUS)` to `ZEN_ORCH_PERF`.
- **Priority:** Low — only relevant at high concurrency / large volumes; correctness is unaffected.

### 28. `register` failure leaves orphaned PERF row (pre-existing LUW design)

- **Source:** zen-7-4 review (Edge Case Hunter)
- **Date:** 2026-04-09
- **Description:** If `mo_bsr->register` raises after the `INSERT zen_orch_perf` and `snapshot_score` have already written to the DB, there is no `ROLLBACK WORK` in `create_performance`. An orphaned PERF row with STEP children but no BSR entry remains. The PREREQ_GATE consumers will never find this performance in the BSR and will block indefinitely.
- **Possible approach:** Add `ROLLBACK WORK` in the `CATCH zcx_en_orch_error` after the `register` call, or move `register` before `INSERT zen_orch_perf` and use optimistic rollback.
- **Priority:** Low — `register` is unlikely to fail in practice; the underlying `INSERT zen_orch_bsr` only fails on duplicate key (see #26). Deferred to lifecycle hardening story.

### 29. Magic constant `'SHA256'` in `calculate_hash_for_char` call

- **Source:** zen-7-4 review (Blind Hunter)
- **Date:** 2026-04-09
- **Description:** The algorithm name `'SHA256'` is a string literal embedded directly in the `if_algorithm` parameter. If the project needs to change algorithm (e.g., for compliance), it must be found and updated everywhere it appears.
- **Possible approach:** Define a constant `gc_hash_algorithm TYPE string VALUE 'SHA256'` in a shared constants class or in `ZCL_EN_ORCH_ENGINE` and reference it everywhere hashing is needed.
- **Priority:** Low — SHA-256 is stable; cosmetic improvement only.

### 30. `set_bsr` FRIENDS declaration missing for test classes (deferred to zen-7-6)

- **Source:** zen-7-4 review (Blind Hunter)
- **Date:** 2026-04-09
- **Description:** `set_bsr` is PRIVATE. `GLOBAL FRIENDS` currently only lists `zcl_en_orch_health_chk_query`. Test classes for zen-7-6 that need to inject a mock BSR via `set_bsr` will require the test class to be added to `GLOBAL FRIENDS`.
- **Possible approach:** Add `zcl_en_orch_engine_test` (or the zen-7-6 test class name) to `GLOBAL FRIENDS` in zen-7-6 when the test class is created.
- **Priority:** Medium — blocking for zen-7-6 BSR mock injection; address in zen-7-6 story.

