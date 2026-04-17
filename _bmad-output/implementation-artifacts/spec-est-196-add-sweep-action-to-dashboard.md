---
title: 'EST-196: Add sweep() action to dashboard (instance)'
type: 'feature'
created: '2026-04-17'
status: 'done'
baseline_commit: 'ddefeca'
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The orchestrator dashboard has no way to manually trigger advancement of a single performance without waiting for the next APJ job execution. Operators sometimes need to immediately advance a PENDING or RUNNING performance for debugging or ad-hoc processing.

**Approach:** Add an instance action `SweepPerformance` to `ZEN_ORCH_C_PERF` gated by feature control (enabled only for active statuses: `P` Pending, `R` Running). A new engine method `sweep_performance` calls the existing private `advance_performance( iv_perf_uuid )` for the targeted performance, replicating the fail-stop error handling from `sweep_all`. The RAP handler follows the identical call-and-report pattern used by all existing actions.

## Boundaries & Constraints

**Always:**
- Line length ≤ 120 chars in all ABAP/CDS files
- Feature control gate: enabled only when `status IN ('P', 'R')`
- `iv_skip_commit = abap_true` when called from RAP handler (identical to all other action methods)
- Fail-stop error handling in engine: mark performance FAILED + log on `advance_performance` exception (mirrors `sweep_all`)
- BSR deregistration after advance if performance reached terminal state (mirrors `sweep_all`)

**Never:**
- Do not enable Sweep for terminal statuses C, F, X (no work to advance)
- Do not enable Sweep for PAUSED (B) — use Resume instead
- Do not `COMMIT WORK` inside the engine method when called with `iv_skip_commit = abap_true`

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Sweep PENDING | Performance status `P` | Sweep button enabled → `advance_performance` runs; performance advances | `zcx_en_orch_error` → perf marked FAILED, surfaced to Fiori message strip |
| Sweep RUNNING | Performance status `R` | Same as above | Same |
| Sweep not allowed | Performance status `B`, `C`, `F`, or `X` | Sweep button greyed out (feature control) | N/A — button not reachable |
| advance_performance fails | Any active perf, adapter error | Performance marked FAILED with ended_at timestamps; pending steps also marked FAILED | Logged to BAL, error reported to Fiori |
| BSR deregistration after terminal | Performance completes/fails during sweep | BSR deregistered best-effort | Exception logged, delete still continues |

</frozen-after-approval>

## Code Map

- `src/zcl_en_orch_engine.clas.abap` — Engine; add `sweep_performance` method declaration + implementation
- `src/zen_orch_c_perf.bdef.asbdef` — BDEF; add `SweepPerformance` instance action + extend all 5 side effects blocks
- `src/zcl_en_orch_bp_perf.clas.locals_imp.abap` — RAP handler; add `sweepperformance` method, extend `get_instance_features` + `get_global_authorizations`
- `src/zen_orch_c_perf.ddls.asddls` — CDS; add `@UI.lineItem` + `@UI.fieldGroup` entries for `sweepPerformance`

## Tasks & Acceptance

**Execution:**

- [x] `src/zcl_en_orch_engine.clas.abap` — Add `sweep_performance( iv_perf_uuid TYPE zen_orch_perf_uuid, iv_skip_commit TYPE abap_bool DEFAULT abap_false )` in PUBLIC SECTION; implement: guard (only P/R), call `advance_performance`, fail-stop handling, BSR deregister if terminal, conditional commit
- [x] `src/zen_orch_c_perf.bdef.asbdef` — Add `action ( features: instance ) SweepPerformance external 'sweepPerformance';` + extend all 5 side-effects entries with `action SweepPerformance`
- [x] `src/zcl_en_orch_bp_perf.clas.locals_imp.abap` — Add `sweepperformance` method declaration + implementation; extend `get_instance_features` (both disabled block + COND block); extend `get_global_authorizations`
- [x] `src/zen_orch_c_perf.ddls.asddls` — Add `{ type: #FOR_ACTION, dataAction: 'sweepPerformance', label: 'Sweep' }` to `@UI.lineItem` and `@UI.fieldGroup` (position 50)

**Acceptance Criteria:**

- Given a performance with status `P` or `R`, when the user opens the dashboard and selects that row, then a **Sweep** button is visible and enabled
- Given the user clicks **Sweep**, then `sweep_performance` is called; performance advances; success message 'Performance sweep triggered' appears
- Given `advance_performance` fails during the sweep, then the performance is marked FAILED with timestamps and the error is surfaced in the Fiori message strip
- Given a performance with status `B`, `C`, `F`, or `X`, when the user selects that row, then the Sweep button is absent or greyed out

## Verification

**Manual checks:**
- Activate BDEF in ADT → no activation errors
- Open dashboard, select PENDING or RUNNING row → Sweep button visible and enabled
- Select COMPLETED/FAILED/CANCELLED/PAUSED row → Sweep button absent/greyed
- Click Sweep on PENDING row → success toast; performance advances (re-check Status column)
