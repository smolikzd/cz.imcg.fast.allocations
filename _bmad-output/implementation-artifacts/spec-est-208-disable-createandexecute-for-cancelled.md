---
title: 'EST-208: Disable CreateAndExecute for cancelled process instances'
type: 'feature'
created: '2026-04-19'
status: 'done'
baseline_commit: 'f75b0747782f4507180d5edea76ef6b0c2bca9d5'
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** `CreateAndExecute` is currently enabled for CANCELLED instances in the Fiori dashboard. Now that EST-192 enables Restart for CANCELLED, users have a cleaner path to resume a cancelled instance. Leaving `CreateAndExecute` enabled creates ambiguity and allows accidentally creating a duplicate new instance from a CANCELLED one.

**Approach:** Remove `gc_status-cancelled` from the `CreateAndExecute` feature control COND expression, and add a matching server-side status guard in the `createandexecute` action handler to reject CANCELLED at the OData layer too. No framework changes needed — `create_process()` itself is not restricted.

## Boundaries & Constraints

**Always:**
- `CreateAndExecute` remains enabled for `SUPERSEDED` and for rows with no instance (`ProcessInstanceId IS INITIAL`).
- Use `zcl_fi_process_instance=>gc_status-cancelled` constant — no magic strings.
- The server-side guard in `createandexecute` must follow the same pattern as `restartprocess`: `SELECT SINGLE` → row-not-found guard → status guard.
- Line length ≤ 120 chars.

**Ask First:** nothing — scope is fully defined.

**Never:**
- Restrict `CreateAndExecute` for SUPERSEDED — that must stay enabled.
- Restrict `CreateAndExecute` for rows with no instance — that must stay enabled.
- Modify framework classes (`ZCL_FI_PROCESS_INSTANCE`, `ZCL_FI_PROCESS_MANAGER`).
- Change any other action's feature control.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| CreateAndExecute on CANCELLED (UI) | Instance in CANCELLED status | Button disabled — not clickable | N/A |
| CreateAndExecute on SUPERSEDED (UI) | Instance in SUPERSEDED status | Button remains enabled | N/A |
| CreateAndExecute — no instance (UI) | Row with no ProcessInstanceId | Button remains enabled | N/A |
| Direct OData call on CANCELLED (bypass UI) | POST CreateAndExecute on CANCELLED row | Server-side guard rejects; error returned to caller | OData error response |

</frozen-after-approval>

## Code Map

- `cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` -- Behavior pool locals: `get_instance_features` (feature control, line 123) and `createandexecute` action handler (line 314, currently no status guard)

## Tasks & Acceptance

**Execution:**
- [x] `zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` -- In `get_instance_features` status-based COND (line 123), remove `OR lv_status = zcl_fi_process_instance=>gc_status-cancelled` from `%features-%action-CreateAndExecute` -- disables button for CANCELLED in UI
- [x] `zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` -- In `METHOD createandexecute`, add SELECT SINGLE + row-not-found guard + status guard rejecting CANCELLED (same pattern as `restartprocess` handler) before the LOOP AT keys -- prevents bypass via direct OData call

**Acceptance Criteria:**
- Given a CANCELLED instance, when the dashboard renders, then the CreateAndExecute button is disabled.
- Given a SUPERSEDED instance, when the dashboard renders, then the CreateAndExecute button remains enabled.
- Given a row with no process instance, when the dashboard renders, then the CreateAndExecute button remains enabled.
- Given a direct OData POST of CreateAndExecute on a CANCELLED instance, then the action is rejected with an error and no new instance is created.
- Given a CANCELLED instance, when the user clicks Restart (EST-192), then Restart still works correctly (regression).

## Suggested Review Order

**Feature control gate**

- CANCELLED removed from WHEN clause → falls to ELSE (disabled); SUPERSEDED unchanged.
  [`locals_imp.abap:122`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap#L122)

**Server-side action guard**

- SELECT SINGLE fetches current status; sy-subrc guard rejects unknown keys.
  [`locals_imp.abap:314`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap#L314)

- Status guard rejects CANCELLED regardless of instance ID (status check only).
  [`locals_imp.abap:344`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap#L344)

## Spec Change Log

- **Review loop 1**: Blind hunter flagged `ProcessInstanceId IS NOT INITIAL AND status = CANCELLED` guard as too narrow (corrupted data with IS INITIAL + CANCELLED would bypass). Fixed: removed `IS NOT INITIAL AND` — check status alone.

## Design Notes

The `createandexecute` handler currently has no server-side guard — by design for the "no instance" case (no ProcessInstanceId to look up). Adding a guard for the CANCELLED case requires a `SELECT SINGLE` on the key fields to retrieve the current `ProcessInstanceId` and `ProcessStatus`. If `ProcessInstanceId IS INITIAL`, the guard must pass (allow) — this preserves the "no instance" path. Only when `ProcessInstanceId IS NOT INITIAL AND status = CANCELLED` should the guard reject.

Pattern to follow from `restartprocess` handler (line 256):
```abap
SELECT SINGLE ProcessInstanceId, ProcessStatus
  FROM zfi_i_alloc_dashboard_ce
  WHERE FiscalYear = @ls_key-FiscalYear ...
  INTO @DATA(ls_row).

IF sy-subrc <> 0.
  " row not found — append to failed/reported, CONTINUE
ENDIF.

IF ls_row-ProcessInstanceId IS NOT INITIAL
   AND ls_row-ProcessStatus = zcl_fi_process_instance=>gc_status-cancelled.
  " append to failed/reported, CONTINUE
ENDIF.
```

## Verification

**Manual checks:**
- Open dashboard, find a CANCELLED instance → CreateAndExecute button must be disabled (greyed out).
- Find a SUPERSEDED instance → CreateAndExecute button must still be enabled.
- Find a row with no instance → CreateAndExecute button must still be enabled.
- Verify a CANCELLED instance can still be Restarted (EST-192 regression).
- Run health checks via `ZCL_FIPROC_HEALTH_CHK_QUERY` — all checks must pass.
