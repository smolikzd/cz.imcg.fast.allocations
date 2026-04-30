---
title: 'EST-212: Enable CreateAndExecute for cancelled process instances (reverse EST-208)'
type: 'feature'
created: '2026-04-30'
status: 'ready-for-dev'
target_repository: 'ovysledovka'
depends_on: ['EST-192', 'EST-208']
constitution_principles:
  - 'Principle II - SAP Standards'
  - 'Principle V - Error Handling'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** After EST-192 enabled Restart for CANCELLED instances, EST-208 disabled `CreateAndExecute` for CANCELLED to avoid ambiguity. However, users legitimately need **both** options: Restart to resume from where the process stopped, and Create & Execute to start entirely from scratch.

**Approach:** Reverse the EST-208 changes for `CreateAndExecute` on CANCELLED:
1. Re-enable `CreateAndExecute` button in feature control for CANCELLED status.
2. Remove the server-side OData guard added in EST-208 that rejects CANCELLED instances.

No framework changes needed — `create_process()` already supports creating a new instance regardless of existing ones.

## Boundaries & Constraints

**Always:**
- `CreateAndExecute` remains enabled for `SUPERSEDED` — no change to that path.
- `CreateAndExecute` remains enabled for rows with no instance (`ProcessInstanceId IS INITIAL`).
- Use `zcl_fi_process_instance=>gc_status-cancelled` constant — no magic strings.
- `RestartProcess` must still work for CANCELLED (EST-192 regression must pass).
- Line length ≤ 120 chars.

**Ask First:** nothing — scope is fully defined by Linear EST-212.

**Never:**
- Modify framework classes (`ZCL_FI_PROCESS_INSTANCE`, `ZCL_FI_PROCESS_MANAGER`).
- Change any other action's feature control.
- Disable `RestartProcess` for CANCELLED — both actions must coexist.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| CreateAndExecute on CANCELLED (UI) | Instance in CANCELLED status | Button **enabled** — user can click | N/A |
| Restart on CANCELLED (UI) | Instance in CANCELLED status | Restart button also enabled (EST-192 unchanged) | N/A |
| CreateAndExecute on SUPERSEDED (UI) | Instance in SUPERSEDED status | Button remains enabled | N/A |
| CreateAndExecute — no instance (UI) | Row with no ProcessInstanceId | Button remains enabled | N/A |
| Direct OData call on CANCELLED | POST CreateAndExecute on CANCELLED row | Server-side guard allows it; new instance created | Framework raises error if params invalid |

</frozen-after-approval>

## Code Map

- `cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap`
  - `get_instance_features` — feature control COND for `%action-CreateAndExecute` (around line 122)
  - `createandexecute` action handler — server-side status guard added by EST-208 (around line 344)

## Tasks & Acceptance

**Execution:**
- [ ] `zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` — In `get_instance_features`, add `OR lv_status = zcl_fi_process_instance=>gc_status-cancelled` back to the `%features-%action-CreateAndExecute` COND expression — re-enables button for CANCELLED in UI
- [ ] `zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` — In `METHOD createandexecute`, remove the server-side status guard that rejects CANCELLED (the guard added in EST-208 around line 344) — allows direct OData call for CANCELLED too

**Acceptance Criteria:**
- Given a CANCELLED instance, when the dashboard renders, the Create & Execute button is **enabled**.
- Given a CANCELLED instance, when the user clicks Create & Execute, a new process instance is created and started.
- Given a CANCELLED instance, when the dashboard renders, the Restart button is **also enabled** (EST-192 regression).
- Given a SUPERSEDED instance, when the dashboard renders, the Create & Execute button remains enabled.
- Given a row with no process instance, when the dashboard renders, the Create & Execute button remains enabled.
- Run health checks via `ZCL_FIPROC_HEALTH_CHK_QUERY` — all checks must pass.

## Suggested Review Order

**Feature control gate**
- `gc_status-cancelled` added back to WHEN clause for `CreateAndExecute`.
  `locals_imp.abap` → `get_instance_features` → `%features-%action-CreateAndExecute` COND

**Server-side action guard removal**
- EST-208 guard (status = CANCELLED → reject) removed from `createandexecute` handler.
  `locals_imp.abap` → `METHOD createandexecute` → status guard block

## Spec Change Log

- **2026-04-30 (initial):** Story created. Reverses EST-208 CANCELLED restriction. Both Restart and Create & Execute to coexist for CANCELLED status.

## Design Notes

EST-208 rationale was: "users have a cleaner path (Restart)". EST-212 overrules this — users need explicit choice. The technical path is simply undoing the two EST-208 changes:

1. Feature control COND for `CreateAndExecute`: restore `OR lv_status = zcl_fi_process_instance=>gc_status-cancelled`.
2. `createandexecute` server-side guard: remove the CANCELLED rejection IF block.

The `create_process()` framework call itself is unchanged and works fine for CANCELLED rows (creates a new independent instance).

## Verification

**Manual checks:**
- Open dashboard in Fiori, find a CANCELLED instance.
  - Create & Execute button must be **enabled**.
  - Restart button must also be **enabled** (EST-192 regression).
- Click Create & Execute on a CANCELLED instance → new instance must be created and transition to RUNNING/QUEUED.
- Click Restart on a CANCELLED instance → instance must transition to RESTREQ then RUNNING.
- Find a SUPERSEDED instance → Create & Execute must still be enabled.
- Find a row with no instance → Create & Execute must still be enabled.
- Run health checks via `ZCL_FIPROC_HEALTH_CHK_QUERY` — all checks must pass.
