---
title: 'EST-197: Add delete() action to dashboard'
type: 'feature'
created: '2026-04-17'
status: 'done'
baseline_commit: 'dbad9e1'
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The orchestrator dashboard has no way to permanently remove a finished, failed, or cancelled performance. Terminal records accumulate indefinitely with no clean-up option from the UI.

**Approach:** Add an instance action `DeletePerformance` to `ZEN_ORCH_C_PERF` gated by feature control (enabled only for terminal statuses: `C` Completed, `F` Failed, `X` Cancelled). A new engine method `delete_performance` handles BSR deregistration (best-effort), deletes child step rows, then deletes the performance header. The RAP handler follows the identical call-and-report pattern used by `CancelPerformance`.

## Boundaries & Constraints

**Always:**
- Line length ≤ 120 chars in all ABAP/CDS files
- Delete order: BSR deregister (best-effort) → `ZEN_ORCH_P_STEP` (child) → `ZEN_ORCH_PERF` (header)
- Feature control gate: enabled only when `status IN ('C', 'F', 'X')`
- `iv_skip_commit = abap_true` when called from RAP handler (identical to all other action methods)
- Engine method follows same structure as `cancel_performance` (T-step comments, guard, BSR deregister)

**Ask First:**
- If activation of the BDEF with the new action causes strict(2) errors related to the missing result parameter pattern, HALT and ask Zdenek

**Never:**
- Do not use standard RAP `delete;` operation (this is unmanaged)
- Do not enable delete for `P` (Pending), `R` (Running), or `B` (Paused/Breakpoint)
- Do not `COMMIT WORK` inside the engine method when called with `iv_skip_commit = abap_true`

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Delete COMPLETED | Performance status `C` | Delete button enabled → rows removed from `ZEN_ORCH_P_STEP` + `ZEN_ORCH_PERF`; row disappears from list | `zcx_en_orch_error` → surfaced to Fiori message strip |
| Delete FAILED | Performance status `F` | Same as above | Same |
| Delete CANCELLED | Performance status `X` | Same as above | Same |
| Delete not allowed | Performance status `P`, `R`, or `B` | Delete button greyed out (feature control) | N/A — button not reachable |
| BSR entry already gone | Terminal perf, BSR key not found | `deregister()` fails silently (best-effort, same as cancel_performance T1.6) | Log exception, continue delete |

</frozen-after-approval>

## Code Map

- `src/zen_orch_c_perf.bdef.asbdef` — BDEF; add `DeletePerformance` instance action + side effects
- `src/zcl_en_orch_bp_perf.clas.locals_imp.abap` — RAP handler; add `deleteperformance` method, extend `get_instance_features` + `get_global_authorizations`
- `src/zcl_en_orch_engine.clas.abap` — Engine; add `delete_performance` method declaration + implementation
- `src/zen_orch_c_perf.ddls.asddls` — CDS; add `@UI.lineItem` + `@UI.fieldGroup` entries for `deletePerformance`

## Tasks & Acceptance

**Execution:**

- [x] `src/zen_orch_c_perf.bdef.asbdef` — Add `action ( features: instance ) DeletePerformance external 'deletePerformance';` + side effects entry with `permissions( action DeletePerformance )` and `field *`
- [x] `src/zcl_en_orch_engine.clas.abap` — Add `delete_performance( iv_perf_uuid TYPE zen_orch_perf_uuid, iv_skip_commit TYPE abap_bool DEFAULT abap_false )` in PUBLIC SECTION; implement: guard (only C/F/X), best-effort BSR deregister, DELETE child steps, DELETE header
- [x] `src/zcl_en_orch_bp_perf.clas.locals_imp.abap` — Add `deleteperformance` method declaration + implementation (call engine, report success/error); extend `get_instance_features` (all 3 branches + main COND block); extend `get_global_authorizations`
- [x] `src/zen_orch_c_perf.ddls.asddls` — Add `{ type: #FOR_ACTION, dataAction: 'deletePerformance', label: 'Delete' }` to `@UI.lineItem` and `@UI.fieldGroup` arrays

**Acceptance Criteria:**

- Given a performance with status `C`, `F`, or `X`, when the user opens the dashboard and selects that row, then a **Delete** button is visible and enabled
- Given the user clicks **Delete** and confirms, then the performance row disappears from the list and `ZEN_ORCH_P_STEP` + `ZEN_ORCH_PERF` records are removed
- Given a performance with status `P`, `R`, or `B`, when the user selects that row, then the Delete button is absent or greyed out
- Given `delete_performance` is called and the BSR entry is already gone, then the delete still completes successfully

## Verification

**Manual checks:**
- Activate BDEF in ADT → no activation errors
- Open dashboard, select COMPLETED row → Delete button visible and enabled
- Select RUNNING row → Delete button absent/greyed
- Confirm deletion → row disappears; verify via SE16 `ZEN_ORCH_PERF` + `ZEN_ORCH_P_STEP` rows gone

