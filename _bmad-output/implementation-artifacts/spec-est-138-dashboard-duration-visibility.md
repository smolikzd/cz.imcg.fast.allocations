---
title: 'EST-138: Dashboard Process and Step Duration Visibility'
type: 'feature'
created: '2026-04-01'
status: 'done'
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The Allocation Dashboard shows `StartedAt` and `EndedAt` timestamps for both the process run and individual steps, but provides no computed duration column. Users must mentally subtract times to assess how long a run took or how long each step took, which is friction-heavy especially for multi-step processes and in-progress runs.

**Approach:** Add a formatted duration string (e.g. `1h 23m 45s`) to the process-level list report row and to each step row on the object page. For the process, use the stored `TOTAL_DURATION` (`INT4`, seconds) field from `zfi_proc_inst` when available; if 0/initial (process started but framework has not yet written it), fall back to computing elapsed seconds from `StartedAt` to now (`SY-DATUM` + `SY-UZEIT`). For steps, compute duration from `StepStartedAt` / `StepEndedAt` raw timestamps (no stored field exists at step level).

## Boundaries & Constraints

**Always:**
- Target repository: `cz.imcg.fast.ovysledovka`
- Duration displayed as formatted string `Xh Ym Zs` (e.g. `1h 23m 45s`, `45s`, `2h 5s`) — omit zero components except when all are zero (show blank); no numeric type needed for display
- Process duration: prefer `zfi_proc_inst.TOTAL_DURATION` (INT4 seconds) when > 0; if 0/initial and `StartedAt` is populated, compute elapsed from `StartedAt` to current ABAP system time (`SY-DATUM` / `SY-UZEIT`) as live fallback for running processes
- Step duration computed in `ZCL_FI_ALLOC_DASH_STEP_QRY` using `StepStartedAt` / `StepEndedAt` raw `TIMESTAMPL` fields from `ZFI_I_PROCESS_STEP_STATS`; if `StepEndedAt` is initial (running step), fall back to elapsed from `StepStartedAt` to now — same live-diff rule as process-level; computation happens in ABAP code (not CDS)
- Line length ≤ 120 chars (abaplint enforced)
- ABAP-Doc on any new public methods
- No `COMMIT WORK` in RAP handlers/savers

**Ask First:**
- (resolved — no open questions)

**Never:**
- Do not add a numeric/INT duration field to the custom entities — formatted string is sufficient for display
- Do not change the CDS view `ZFI_I_PROCESS_STEP_STATS` (it is in the planner repo, out of scope)
- Do not surface `AvgSubstepDuration` — that was deliberately excluded per EST-133 (substep average, not step total)
- Do not add duration to the list report filter bar

</frozen-after-approval>

## Code Map

- `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` -- parent custom entity; add `Duration : abap.char(20)` field (enough for `99d 23h 59m 59s` = 17 chars plus margin)
- `ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_query.clas.abap` -- parent query class; read `TOTAL_DURATION` from `zfi_proc_inst`, add live-fallback logic from `StartedAt`, format to `Xd Xh Xm Xs`, populate `Duration` field
- `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddlx.asddlxs` -- DDLX; add `@UI.lineItem` for `Duration` and `@EndUserText.label`
- `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dash_step_ce.ddls.asddls` -- step child custom entity; add `StepDuration : abap.char(20)` field
- `ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_step_qry.clas.abap` -- step query class; compute duration from raw `StepStartedAt`/`StepEndedAt` `TIMESTAMPL` fields, format to `Xd Xh Xm Xs`, populate `StepDuration`

## Tasks & Acceptance

**Execution:**
- [x] `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` -- add `Duration : abap.char(20);` non-key field after `EndedAt` -- duration display field; `abap.char(20)` covers `99d 23h 59m 59s` (17 chars) with margin
- [x] `ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_query.clas.abap` -- (1) add `inst~total_duration AS TotalDuration` to the existing LEFT OUTER JOIN SELECT on `zfi_proc_inst`; (2) add private helper method `format_duration( iv_seconds TYPE i ) RETURNING VALUE(rv_text) TYPE string` that formats seconds as `Xd Xh Xm Xs` (include days component when ≥ 86400s; omit zero-value components; blank if iv_seconds = 0); (3) when building each result row: if `TotalDuration > 0` call `format_duration( TotalDuration )`; else if `StartedAt` is populated compute elapsed seconds from raw `StartedAt` (`TIMESTAMPL`) to current system time using `CL_ABAP_TSTMP=>SUBTRACT` and call `format_duration`; else leave blank -- live fallback covers in-progress processes where framework has not yet written `TOTAL_DURATION`; keep method ≤ 50 lines
- [x] `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddlx.asddlxs` -- add `@UI.lineItem: [{ position: 95, importance: #MEDIUM }]` and `@EndUserText.label: 'Duration'` on `Duration` field (position 95 places it between `EndedAt` at 90 and `ProcessInstanceId` at 100)
- [x] `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dash_step_ce.ddls.asddls` -- add `StepDuration : abap.char(20);` non-key field after `StepEndedAt`
- [x] `ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_step_qry.clas.abap` -- (1) add private `format_duration` method with identical logic to parent class (includes days; duplicate per EST-133 backlog note); (2) after reading raw `StepStartedAt`/`StepEndedAt` `TIMESTAMPL` values: if both populated compute seconds via `CL_ABAP_TSTMP=>SUBTRACT` and call `format_duration`; if only `StepStartedAt` populated (running step) compute elapsed from `StepStartedAt` to now and call `format_duration`; if `StepStartedAt` initial leave blank; populate `ls_row-StepDuration`

**Acceptance Criteria:**
- Given a completed process run with `TOTAL_DURATION = 754` (seconds), when the list report row is displayed, then the `Duration` column shows `12m 34s`
- Given a running process where `TOTAL_DURATION = 0` and `StartedAt` is populated, when the list report row is displayed, then `Duration` shows a non-blank elapsed string computed from `StartedAt` to now (e.g. `3m 12s`)
- Given an empty-period row (no process instance), when displayed, then `Duration` is blank
- Given a process duration of exactly 3661 seconds, when displayed, then `Duration` shows `1h 1m 1s`
- Given a process duration of 45 seconds, when displayed, then `Duration` shows `45s` (no `0h` or `0m` prefix)
- Given the object page step table is open for a completed instance, when step rows are displayed, then each step with both `StepStartedAt` and `StepEndedAt` populated shows a non-blank `StepDuration` in `Xh Ym Zs` format
- Given a step where `StepStartedAt` is initial, when the step row is displayed, then `StepDuration` is blank
- Given a step with `StepStartedAt` populated and `StepEndedAt` initial (running step), when displayed, then `StepDuration` shows live elapsed time from `StepStartedAt` to now (same live fallback as process-level)

## Design Notes

**Duration format:** `Xd Xh Xm Xs` with zero-components omitted — e.g. `45s`, `2m 5s`, `1h 23m 45s`, `2d 3h`, `5d 10h 30m 15s`. Blank when seconds = 0. Days component included when total seconds ≥ 86400. This is more human-readable than `HH:MM:SS` for allocation runs that may span multiple days.

**Process duration source:** `zfi_proc_inst.TOTAL_DURATION` (INT4, seconds) is the authoritative source for completed/failed runs. For running processes where it is 0/initial, fall back to `CL_ABAP_TSTMP=>SUBTRACT( start, now )` where `now` is built from `SY-DATUM` + `SY-UZEIT` via `CL_ABAP_TSTMP=>SYSTEMTSTAMP` or equivalent. The raw `StartedAt` `TIMESTAMPL` is already selected by the query class.

**Step duration computation:** Compute from `StepStartedAt` / `StepEndedAt` raw `TIMESTAMPL` fields. If both populated → subtract for exact duration. If only `StepStartedAt` populated (running step) → live fallback: elapsed from `StepStartedAt` to now, same as process-level. If `StepStartedAt` initial → blank.

**`abap.char(20)` field size:** `99d 23h 59m 59s` = 17 chars; `20` gives comfortable margin without over-allocating.

## Review Findings

- [x] [Review][Decision] **F1 — Live fallback for running processes: is it needed?** — AC-2 says "Given a running process where TOTAL_DURATION = 0 and StartedAt is populated, Duration shows non-blank elapsed string." Current code only populates duration when `totalruntime > 0` (no ABAP fallback). Session notes say `ZFIPROC_R_INSTANCE_TP.TotalRuntime` is CDS-computed `ceil(tstmp_seconds_between(...))` — always > 0 for running processes. If that is true, AC-2 is satisfied by TotalRuntime and no ABAP fallback is needed. If TotalRuntime can be 0 for a running process (race in first second), AC-2 fails. Needs confirmation. [`zcl_fi_alloc_dash_query.clas.abap:377-381`] — **dismissed**: TotalRuntime from CDS already handles live processes; ABAP fallback confirmed not needed.

- [x] [Review][Decision] **F4 — Duration column position 80 vs. spec position 95** — Spec Task 3 specified `@UI.lineItem position: 95` (between EndedAt=90 and ProcessInstanceId=100). Code uses position 80 (immediately after EndedAt=70). The annotation is inline in the CDS CE (no DDLX file exists, which was an agreed deviation). The position value itself differs from the spec. [`zfi_i_alloc_dashboard_ce.ddls.asddls:162`] — **dismissed**: position 80 is correct; spec's position 95 was based on stale column numbers; Duration logically follows EndedAt.

- [x] [Review][Patch] **F3 — `DATA` declared inside LOOP body** — `lv_step_secs` (TYPE decfloat34) and `lv_now` (TYPE timestampl) are declared inside a conditional block within the LOOP. In ABAP, DATA declarations inside loops are not re-initialised per iteration; they live for the method lifetime. While safe in the current code (both are assigned before use within the guarded branch), this is a maintenance hazard and violates Clean ABAP guidelines. Move both declarations to the method's DATA section (before the LOOP). [`zcl_fi_alloc_dash_step_qry.clas.abap:326,332`] — **fixed**: moved to method DATA section, added explicit `CLEAR lv_step_secs` before use.

- [x] [Review][Patch] **F5 — ABAP-Doc `iv_seconds = 0` should say `<= 0`** — The `format_duration` ABAP-Doc comment in both classes says "Returns empty string when iv_seconds = 0" but the implementation guard is `IF iv_seconds <= 0` (also handles negatives). Fix doc in both files. [`zcl_fi_alloc_dash_query.clas.abap:147`, `zcl_fi_alloc_dash_step_qry.clas.abap:113`] — **fixed**.

- [x] [Review][Defer] **F2 — `CONV i()` truncates fractional seconds (floor, not round)** [`zcl_fi_alloc_dash_query.clas.abap:380`, `zcl_fi_alloc_dash_step_qry.clas.abap:340`] — deferred, pre-existing; sub-second precision irrelevant for human-readable duration display

## Verification

**Manual checks (no CLI for SAP ABAP):**
- Open service binding `ZFI_UI_ALLOC_DASHBOARD_O4` → Preview → run with CompanyCode + FiscalYear + AllocationId filters
- Verify `Duration` column appears in list report (position between EndedAt and ProcessInstanceId)
- Verify completed runs show `Xd Xh Xm Xs` formatted duration with zero-components omitted
- Verify empty-period rows (no process instance) show blank Duration
- Verify a running process row shows a non-blank elapsed duration (live fallback active)
- Open object page for a completed run → Process Steps table → verify `StepDuration` column appears with `Xd Xh Xm Xs` values per completed step
- Verify steps not yet started show blank `StepDuration`; verify running steps show live elapsed duration
