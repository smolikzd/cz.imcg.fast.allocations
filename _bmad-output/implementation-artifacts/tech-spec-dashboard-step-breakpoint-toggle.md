---
title: 'Dashboard step-level breakpoint toggle action'
type: 'feature'
created: '2026-03-31'
status: 'done'
baseline_commit:
  planner: '2ec42c66ab68c5c088e451affff8a109ff5269cd'
  ovysledovka: '062d527f23de22418193061c77302705464dceb7'
context:
  - '_bmad/_memory/constitution.md'
  - '_bmad-output/implementation-artifacts/tech-spec-est-116-breakpoint-step-execution.md'
---

<frozen-after-approval reason="human-owned intent -- do not modify unless human renegotiates">

## Intent

**Problem:** The breakpoint flag on `ZFI_PROC_STEP` can only be set programmatically via `ZCL_FI_PROCESS_MANAGER=>set_step_breakpoint_process()`. Dashboard operators have no UI way to set or clear a breakpoint on an individual step of a running instance, and the step table does not display whether a breakpoint is active.

**Approach:** Add a `Breakpoint` boolean field to the step custom entity, display it as an icon column (first position) in the step line-item table using `sap-icon://breakpoint`, and expose a `ToggleBreakpoint` RAP action on the step child entity that calls the existing manager API. The action is only enabled when a process instance exists (ProcessInstanceId is not initial).

## Boundaries & Constraints

**Always:**
- Planner repo (`cz.imcg.fast.planner`): Expose `Breakpoint` field in `ZFI_I_PROCESS_STEP` CDS view so `ZFI_I_PROCESS_STEP_STATS` can propagate it
- Ovysledovka repo (`cz.imcg.fast.ovysledovka`): All dashboard changes (CDS custom entity, BDEF, handler, query provider)
- Constitution Principle I (DDIC-First): No new local types -- reuse existing `ZFI_PROCESS_BREAKPOINT` data element
- Constitution Principle III: RAP action pattern follows existing dashboard actions (buffer + saver)
- The breakpoint icon column must be the first column (position 5, before StepNumber at 10)
- Action delegates to `ZCL_FI_PROCESS_MANAGER=>set_step_breakpoint_process()` with `iv_no_commit = abap_true` in the saver (same pattern as other dashboard actions)
- The action buffer structure `ZFI_ALLOC_S_ACTION_BUFFER` gains a `STEP_NUMBER` field to carry the step key for the breakpoint toggle

**Ask First:**
- If the action needs to be restricted to certain process statuses beyond "instance exists" (e.g. only when RUNNING or BREAKPOINT)

**Never:**
- Do not create a separate BDEF entity for steps -- use the existing parent BDEF with an action scoped to step context
- Do not change any planner framework logic beyond CDS field exposure

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Set breakpoint | Step has Breakpoint = false, user clicks Toggle | Breakpoint set to true, icon appears, success msg | N/A |
| Clear breakpoint | Step has Breakpoint = true, user clicks Toggle | Breakpoint set to false, icon disappears, success msg | N/A |
| No process instance | Empty period row (ProcessInstanceId initial) | ToggleBreakpoint action disabled via instance features | N/A |
| Step not found | Race condition: step deleted between UI read and action | zcx_fi_process_error caught, mapped to RAP error msg | Manager raises step_not_found |
| Completed step | Step already COMPLETED, user toggles breakpoint | Breakpoint toggled (flag set/cleared on DB) -- harmless, execution already past | N/A |

</frozen-after-approval>

## Code Map

**Planner repo** (`cz.imcg.fast.planner`):
- `src/zfi_i_process_step.ddls.asddls` -- Base CDS view on `zfi_proc_step`; currently missing `breakpoint` field
- `src/zfi_i_process_step_stats.ddls.asddls` -- Aggregated stats view; needs to propagate breakpoint from header row

**Ovysledovka repo** (`cz.imcg.fast.ovysledovka`):
- `src/zfi_alloc_process/zfi_i_alloc_dash_step_ce.ddls.asddls` -- Step custom entity; add Breakpoint field + icon column
- `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.bdef.asbdef` -- BDEF; add ToggleBreakpoint action on step child entity
- `src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` -- Handler; add togglebreakpoint method + step-aware instance features + saver logic
- `src/zfi_alloc_process/zcl_fi_alloc_dash_step_qry.clas.abap` -- Step query provider; read and map breakpoint field
- `src/zfi_alloc_process/zfi_alloc_s_action_buffer.tabl.xml` -- Add STEP_NUMBER field for breakpoint toggle routing

## Tasks & Acceptance

**Execution:**

_Planner repo:_
- [ ] `src/zfi_i_process_step.ddls.asddls` -- ADD `zfi_proc_step.breakpoint as Breakpoint` field to the select list -- expose existing DB column to CDS consumers
- [ ] `src/zfi_i_process_step_stats.ddls.asddls` -- ADD `max( case when SubstepNumber = '000000000000' then Breakpoint else '' end ) as Breakpoint` to the aggregation -- propagate master-step breakpoint flag through the stats view

_Ovysledovka repo:_
- [ ] `src/zfi_alloc_process/zfi_alloc_s_action_buffer.tabl.xml` -- ADD field `STEP_NUMBER` with rollname `ZFI_PROCESS_STEP_NUMBER` after `INSTANCE_ID` -- carry step key for breakpoint toggle action
- [ ] `src/zfi_alloc_process/zfi_i_alloc_dash_step_ce.ddls.asddls` -- ADD `Breakpoint` field (type `zfi_process_breakpoint`) with `@UI.lineItem` at position 5 with `iconUrl: 'sap-icon://breakpoint'` and `criticality: 'BreakpointCriticality'`; ADD hidden `BreakpointCriticality` (abap.int1) field; ADD ToggleBreakpoint action button annotation on lineItem
- [ ] `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.bdef.asbdef` -- ADD child behavior definition `define behavior for ZFI_I_ALLOC_DASH_STEP_CE` with `action (features: instance) ToggleBreakpoint` and side effects affecting `field *` + `messages`
- [ ] `src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` -- ADD step handler class `lhc_step` with `togglebreakpoint` method (reads current Breakpoint value, buffers 'TOGGLEBP' with inverse value and step_number) and `get_instance_features` (disables ToggleBreakpoint when ProcessInstanceId is initial); EXTEND saver `lsc_dashboard` to handle 'TOGGLEBP' action via `lo_manager->set_step_breakpoint_process()`
- [ ] `src/zfi_alloc_process/zcl_fi_alloc_dash_step_qry.clas.abap` -- ADD `Breakpoint` and `BreakpointCriticality` to `ty_row`; READ `Breakpoint` from `ZFI_I_PROCESS_STEP_STATS`; compute criticality (1=red when true, 0=neutral when false); map to result rows

**Acceptance Criteria:**
- Given a process instance with steps visible on the Object Page, when the step table renders, then the first column shows a breakpoint icon (red) for steps with `Breakpoint = true` and no icon for `Breakpoint = false`
- Given a step with `Breakpoint = false`, when the user clicks ToggleBreakpoint, then the step's breakpoint flag is set to `true` in `ZFI_PROC_STEP` and the icon appears after refresh
- Given a step with `Breakpoint = true`, when the user clicks ToggleBreakpoint, then the breakpoint is cleared and the icon disappears
- Given an empty period row (no process instance), when the Object Page step table is shown, then the ToggleBreakpoint action button is disabled
- Given `set_step_breakpoint_process()` raises `zcx_fi_process_error`, then the error message is displayed in the Fiori message strip

## Design Notes

**Child entity behavior in unmanaged BDEF** -- RAP allows defining behavior for child entities within the same BDEF. The step entity gets its own handler class (`lhc_step`) inheriting from `cl_abap_behavior_handler`. The saver remains shared (`lsc_dashboard`) -- it already loops over the action buffer, and the new 'TOGGLEBP' action is simply another case in the CASE statement.

**Breakpoint icon rendering** -- Fiori Elements V4 supports `@UI.lineItem.iconUrl` for boolean-to-icon mapping. The field is a CHAR(1) boolean; when `'X'`, the icon renders with criticality 1 (red). When `' '`, the column is empty (criticality 0 hides the icon).

**Step stats breakpoint propagation** -- The stats CDS view is a GROUP BY aggregation. The breakpoint flag lives only on the master step row (substep `000000000000`). Using `MAX( CASE WHEN SubstepNumber = '000000000000' THEN Breakpoint ELSE '' END )` cleanly extracts it per step group.

## Verification

**Manual checks (if no CLI):**
- Activate planner CDS views: `ZFI_I_PROCESS_STEP` then `ZFI_I_PROCESS_STEP_STATS`
- Activate ovysledovka: DDIC structure → CDS entity → BDEF → handler class → query class
- Set a breakpoint on step 2 via SM30/SE16 on `ZFI_PROC_STEP`; open dashboard Object Page; verify icon shows in first column
- Click ToggleBreakpoint on a step; verify DB update and icon toggle on refresh
- Open an empty-period row Object Page; verify ToggleBreakpoint button is greyed out

## Suggested Review Order

**Entry point — where the toggle action lands**

- Step child entity handler: toggle direction resolved, action buffered here
  [`locals_imp.abap:593`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap#L593)

**Schema — new fields exposed through the CDS stack**

- Base view: `breakpoint` column exposed from `zfi_proc_step`
  [`zfi_i_process_step.ddls.asddls:19`](../../../cz.imcg.fast.planner/src/zfi_i_process_step.ddls.asddls#L19)

- Stats view: `MAX(CASE…)` aggregation extracts breakpoint from master step row
  [`zfi_i_process_step_stats.ddls.asddls:67`](../../../cz.imcg.fast.planner/src/zfi_i_process_step_stats.ddls.asddls#L67)

**RAP behavior — action definition and side effects**

- BDEF: child entity behavior block, `ToggleBreakpoint` with instance features + side effects
  [`zfi_i_alloc_dashboard_ce.bdef.asbdef:43`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.bdef.asbdef#L43)

- Handler class definition: `lhc_step` with `get_instance_features` + `togglebreakpoint`
  [`locals_imp.abap:544`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap#L544)

- Feature check: disables action when no process instance exists; stale-variable guard
  [`locals_imp.abap:560`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap#L560)

**Saver — execution and message routing**

- Saver: `SETBP`/`CLEARBP` dispatch to manager with `iv_no_commit = abap_true`
  [`locals_imp.abap:434`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap#L434)

- Manager: `set_step_breakpoint_process` — now forwards `iv_no_commit` to instance method
  [`zcl_fi_process_manager.clas.abap:175`](../../../cz.imcg.fast.planner/src/zcl_fi_process_manager.clas.abap#L175)

- Message routing: SETBP/CLEARBP success and errors reported on step entity `%tky`
  [`locals_imp.abap:450`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap#L450)

**UI annotations and query provider**

- CDS custom entity: `Breakpoint` field at position 5 with icon/criticality/action annotations
  [`zfi_i_alloc_dash_step_ce.ddls.asddls:43`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dash_step_ce.ddls.asddls#L43)

- Query provider: `Breakpoint` + `BreakpointCriticality` mapped from CDS SELECT to result rows
  [`zcl_fi_alloc_dash_step_qry.clas.abap:37`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_step_qry.clas.abap#L37)

**Supporting — buffer structure**

- Action buffer DDIC structure: `STEP_NUMBER` field added for breakpoint toggle routing
  [`zfi_alloc_s_action_buffer.tabl.xml:39`](../../../cz.imcg.fast.ovysledovka/src/zfi_alloc_process/zfi_alloc_s_action_buffer.tabl.xml#L39)
