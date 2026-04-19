---
title: 'EST-192: Enable restart for cancelled process instances'
type: 'feature'
created: '2026-04-19'
status: 'done'
baseline_commit: '8914f92e88c8edc7b202cc6ebfb4d67aff60bdc4'
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The Restart action on the Fiori dashboard is disabled for process instances in `CANCELLED` status, even though the framework (`restart()` and `request_restart()` on `ZCL_FI_PROCESS_INSTANCE`) already supports restarting cancelled instances. This forces users to create a new instance instead of resuming a cancelled one.

**Approach:** Enable the Restart button for `CANCELLED` status in the Fiori feature control and remove the server-side OData guard that also blocks it. No framework changes are needed.

## Boundaries & Constraints

**Always:**
- Only `CANCELLED` is added to the restart allowlist — no other status changes.
- Use `zcl_fi_process_instance=>gc_status-cancelled` constant — no magic strings.
- Line length ≤ 120 chars (abaplint enforced).

**Ask First:** nothing — scope is fully defined.

**Never:**
- Modify the framework classes (`ZCL_FI_PROCESS_INSTANCE`, `ZCL_FI_PROCESS_MANAGER`) — they already allow CANCELLED restart.
- Disable `CreateAndExecute` for CANCELLED here — that is tracked separately in EST-208.
- Change any other action's feature control.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Restart cancelled instance (UI) | Instance in CANCELLED status | Restart button enabled; clicking triggers `request_restart_process()` → status transitions to RESTREQ → APJ picks up | Framework raises `invalid_status` if status somehow changed between UI render and save (optimistic lock handles this) |
| Restart non-cancelled, non-failed, non-breakpoint instance | Any other status | Restart button remains disabled | No change |
| Direct OData call for CANCELLED restart (bypass UI) | POST RestartProcess action on CANCELLED row | Server-side guard allows it; `request_restart_process()` called; status → RESTREQ | Framework error surfaces via OData response if lock conflict |

</frozen-after-approval>

## Code Map

- `cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` -- Behavior pool locals: contains `get_instance_features` (feature control, line ~116) and `restartprocess` OData action handler (server-side guard, line ~286)

## Tasks & Acceptance

**Execution:**
- [x] `zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` -- In `get_instance_features`, add `OR lv_status = zcl_fi_process_instance=>gc_status-cancelled` to the `%features-%action-RestartProcess` COND expression -- enables Restart button in UI for CANCELLED instances
- [x] `zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` -- In `restartprocess` action handler, add `AND ls_row-ProcessStatus <> zcl_fi_process_instance=>gc_status-cancelled` to the rejection IF condition -- removes server-side OData block for CANCELLED

**Acceptance Criteria:**
- Given a process instance in CANCELLED status, when the dashboard list is rendered, then the Restart action button is enabled (not greyed out).
- Given a process instance in CANCELLED status, when the user clicks Restart and confirms, then `request_restart_process()` is called and the instance transitions to RESTREQ status.
- Given a process instance in any status other than FAILED, BREAKPOINT, or CANCELLED, when the dashboard renders, then the Restart button remains disabled.
- Given a process instance in COMPLETED status, when the dashboard renders, then the Restart button remains disabled (regression check).

## Suggested Review Order

**UI gate → server guard (same concern, two layers)**

- Feature control: `gc_status-cancelled` added as third enabled condition for Restart button.
  [`locals_imp.abap:116`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap#L116)

- Server-side guard: matching AND-NOT condition extended to allow CANCELLED through.
  [`locals_imp.abap:286`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap#L286)

## Spec Change Log

## Design Notes

The framework (`ZCL_FI_PROCESS_INSTANCE.restart()` and `request_restart()`) already includes `CANCELLED` in its allowed-status guards — confirmed at lines 2092–2103 and 2128–2138. The only barriers are the two UI-layer guards in the behavior pool. The saver already calls `request_restart_process()` with `iv_no_commit = abap_true`, which is correct for RAP context.

## Verification

**Manual checks (if no CLI):**
- Open dashboard in Fiori, find a CANCELLED instance → Restart button must be enabled.
- Click Restart on a CANCELLED instance → instance status must change to RESTREQ, then to RUNNING when APJ picks it up.
- Verify COMPLETED instance still shows Restart as disabled.
- Run health checks via `ZCL_FIPROC_HEALTH_CHK_QUERY` — all checks must pass.
