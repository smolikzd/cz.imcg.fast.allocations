# Deferred Work

Items surfaced during review that are not caused by the current story but worth addressing later.

---

## 1. Performance throttling for high-volume BAL logging

- **Source:** EST-120 review (edge case hunter)
- **Date:** 2026-03-20
- **Description:** With the every-message `flush_to_db()` pattern, each `message()` call triggers a DB round-trip via `save_with_2nd_connection()`. For steps that log hundreds or thousands of messages, this could degrade performance. The current spec explicitly defers this: "start with every-message flush and optimize only if performance is observed to be a problem."
- **Possible approach:** Introduce a configurable flush interval (e.g., flush every N messages or every T seconds) in `ZCL_FI_PROCESS_LOGGER`. Could use a counter field `mv_flush_interval` with a default of 1 (current behavior) that can be raised for high-volume steps.
- **Priority:** Low — only relevant if performance degradation is observed in production.

---

## 2. Server-side status validation in handler action methods

- **Source:** EST-134 review (Blind Hunter finding BH-2)
- **Date:** 2026-03-29
- **Description:** The handler action methods currently only check whether `ProcessInstanceId IS INITIAL` but do not validate that `ProcessStatus` matches the expected status for the action (e.g., Execute should only run when status is `NEW`). Feature control disables buttons in the UI, but a direct OData call could bypass feature control and reach the handler. The backend manager methods have their own status guards and will reject invalid transitions with `zcx_fi_process_error`, so this is defense-in-depth rather than a security gap.
- **Possible approach:** Add status validation in each handler method (e.g., `IF ls_dashboard-ProcessStatus <> 'NEW'` for executeprocess) and populate `failed`/`reported` with a descriptive message before the buffer call.
- **Priority:** Low — backend guards already prevent invalid transitions; this would improve error UX for programmatic OData callers.

---

## 3. Extract status string literals to constants

- **Source:** EST-134 review (Blind Hunter finding BH-5)
- **Date:** 2026-03-29
- **Description:** Status values like `'NEW'`, `'RUNNING'`, `'FAILED'`, `'COMPLETED'`, etc. are hard-coded string literals in `get_instance_features` and potentially in handler validation. If the status domain values change, multiple locations would need updating.
- **Possible approach:** Create constants (either in the handler class or in a shared constants interface) for each status value. Alternatively, read from the ZFI_PROCESS_STATUS domain fixed values.
- **Priority:** Low — domain values are stable and unlikely to change.

---

## 4. Verify message ZFI_PROCESS/030 exists

- **Source:** EST-134 review (Edge Case Hunter finding EC-1)
- **Date:** 2026-03-29
- **Description:** The handler uses `new_message( id = 'ZFI_PROCESS' number = '030' )` for the "no process instance" error. If message number 030 does not exist in message class `ZFI_PROCESS`, the user will see a blank or generic message. This needs verification during activation in ADT.
- **Possible approach:** Check message class `ZFI_PROCESS` in SE91/ADT. If 030 does not exist, create it with text like "No process instance exists for this allocation". If a suitable existing message exists, use that number instead.
- **Priority:** High — must be verified before go-live to ensure proper error UX.

---

## 5. Consider catching CX_ROOT in saver for unexpected exceptions

- **Source:** EST-134 review (Edge Case Hunter finding EC-2)
- **Date:** 2026-03-29
- **Description:** The saver currently only catches `zcx_fi_process_error`. If a manager method raises an unexpected exception (e.g., `cx_sy_zerodivide`, a database exception, or an APJ scheduling error that doesn't inherit from `zcx_fi_process_error`), the exception would propagate uncaught to the RAP framework, potentially causing a short dump.
- **Possible approach:** Add a second `CATCH cx_root` after the `zcx_fi_process_error` catch to map any unexpected exception to `failed`/`reported` with a generic error message. This is defensive programming -- the manager methods should only raise `zcx_fi_process_error`, but belt-and-suspenders is prudent.
- **Priority:** Medium — unlikely but impactful if it occurs in production.

---

## 6. Dashboard action to create a new process instance after supersede

- **Source:** EST-134 implementation (state machine analysis)
- **Date:** 2026-03-30
- **Description:** After superseding a COMPLETED process instance, the SUPERSEDED status is terminal — no further actions are available on that row. The EST-110 design intent is that supersede unlocks the duplicate check so a new `create_process()` can be called. However, the dashboard currently has no "Create New Instance" action — the user must create the instance through other means. This breaks the self-contained operations cockpit workflow.
- **Possible approach:** Add a 5th action `CreateProcess` to the dashboard BDEF that calls `zcl_fi_process_manager=>create_process()` with parameters derived from the current row's CompanyCode, FiscalYear, FiscalPeriod, AllocationId. Feature control: enabled only when no active instance exists (i.e., current row shows SUPERSEDED or CANCELLED, or no ProcessInstanceId).
- **Priority:** Medium — workflow gap; users can work around by creating instances via other tools.

---

## 7. Parallel instance limit check bypassed in APJ job execution path

- **Source:** EST-136 pre-implementation review (edge case hunter, EC14)
- **Date:** 2026-03-30
- **Description:** The `check_parallel_limit()` method in `zcl_fi_process_manager` only runs inside `execute_process()` (manager method). However, the APJ job class calls `lo_instance->execute()` directly, bypassing the manager's parallel limit check. With the new `CreateAndExecute` dashboard action making it easier to rapidly create instances, users could exceed the configured `max_parallel_insts` limit. The APJ jobs would all fire and execute simultaneously without throttling.
- **Possible approach:** Move the parallel limit check into `zcl_fi_process_instance->execute()` itself, or add it to `request_execute_process()` at scheduling time, or add it to the APJ job's execute method before calling `lo_instance->execute()`.
- **Priority:** Low — only relevant for high-volume concurrent execution scenarios. The parallel limit was designed as a soft guardrail, not a hard constraint.
