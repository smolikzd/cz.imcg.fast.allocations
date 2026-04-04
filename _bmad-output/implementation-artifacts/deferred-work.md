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
