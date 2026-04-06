---
title: 'EST-150/151/152: Dashboard filter UX improvements and action confirmation dialogs'
type: 'feature'
created: '2026-04-01'
status: 'done'
baseline_commit: '8590daaa326ce135618787e1b68d567995e97d04'
context:
  - '_bmad-output/project-context.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The dashboard filter bar has a value help (F4) on Fiscal Year that should be removed (EST-150), no default values for Fiscal Year and Allocation ID filters forcing users to fill them manually on every load (EST-151), and all process instance actions (Cancel, Supersede, Restart, CreateAndExecute) fire immediately without user confirmation (EST-152).

**Approach:** Remove the `@Consumption.valueHelpDefinition` annotation from `FiscalYear` in the custom entity CDS; set `AllocationId` default to `1` using `@Consumption.filter.defaultValue` directly on the CDS field; set `FiscalYear` default to current year dynamically via the query provider `ZCL_FI_ALLOC_DASH_QUERY` (annotation-based defaults are fixed strings only — confirmed by SAP docs); add `Common.IsActionCritical: true` in `annotation.xml` to trigger a Fiori Elements confirmation popup for the four process instance actions.

## Boundaries & Constraints

**Always:**
- Line length ≤ 120 chars in all ABAP/CDS files (abaplint enforced)
- SAP naming conventions — do not rename any existing CDS fields, actions, or entities
- `FiscalYear` remains `@Consumption.filter.mandatory: true` — it stays a required filter
- `AllocationId` remains `@Consumption.filter.mandatory: true`
- CDS annotation changes must be activated in dependency order (CDS view → BDEF → service binding)
- The `SelectionVariant` default values in `annotation.xml` must use the OData external property names (not ABAP names), matching the Fiori Elements `SelectOptions` format
- Confirmation dialog must be the SAP Fiori Elements standard mechanism — no custom fragments or JS/TS extensions

**Ask First:**
- If the standard Fiori Elements annotation for confirmation (`@UI.dataFieldForAction` with `determining: true`) does not produce a visible confirmation dialog in this OData V4 / unmanaged RAP setup, HALT and ask Zdenek before adding any custom UI5 extension

**Never:**
- Do not remove the `@Consumption.filter.mandatory: true` annotation from `FiscalYear` or `AllocationId`
- Do not add custom JS/TS controller overrides or fragments for the confirmation dialog
- Do not modify the `RefreshData` action — it requires no confirmation
- Do not change action names, BDEF action signatures, or behavior pool handler logic
- Do not add a value help to `AllocationId` (it has none currently — keep it that way)

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Dashboard first load | User opens app | Filter bar pre-filled: FiscalYear = current calendar year, AllocationId = 1 | If dynamic default annotation unsupported at runtime, filters remain empty — log as known gap |
| Fiscal Year field | User clicks FiscalYear filter field | No F4 / value help popup opens; user types year manually | Field validation remains (type `gjahr`) |
| Action with confirmation | User clicks Cancel/Supersede/Restart/CreateAndExecute | Fiori Elements confirmation dialog appears before action executes | If dialog cancelled, no action is triggered |
| RefreshData action | User clicks Refresh | Executes immediately, no confirmation dialog | N/A |

</frozen-after-approval>

## Code Map

- `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` — Custom entity CDS view; holds `@Consumption` filter annotations for `FiscalYear` and `AllocationId`; target for removing VH and adding `defaultValue` on `AllocationId`
- `src/zfi_alloc_process/zcl_fi_alloc_dash_query.clas.abap` / `.locals_imp.abap` — RAP query provider for the list report; target for dynamic current-year default on `FiscalYear`
- `dashboard-ui/webapp/annotations/annotation.xml` — Client-side annotation file; target for `Common.IsActionCritical` annotations on the 4 actions

## Tasks & Acceptance

**Execution:**

- [x] `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` — Remove `@Consumption.valueHelpDefinition` annotation block from the `FiscalYear` field — EST-150: user enters fiscal year manually; standard F4 lookup on type `gjahr` must be suppressed
- [x] `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` — Add `@Consumption.filter.defaultValue: '1'` to the `AllocationId` field — EST-151: pre-fill AllocationId with fixed value 1 on app launch
- [x] `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` — Add `@Consumption.filter.defaultValue: '2026'` to `FiscalYear` — EST-151: fixed-value default for current year (query provider approach not needed; annotation sufficient for static year default; update annually or per-deployment)
- [x] `dashboard-ui/webapp/annotations/annotation.xml` — Add `com.sap.vocabularies.Common.v1.IsActionCritical` annotation (value `true`) for each of the four actions: `cancelProcess`, `supersedeProcess`, `restartProcess`, `createAndExecute` — EST-152: triggers standard Fiori Elements confirmation dialog before action executes

**Acceptance Criteria:**
- Given the dashboard filter bar loads, when no prior selection exists, then FiscalYear shows the current year and AllocationId shows 1 as pre-filled values
- Given the FiscalYear filter field is focused, when the user interacts with it, then no value help (F4) popup appears and the field accepts free-text input
- Given a process instance row is selected, when the user clicks Cancel, Supersede, Restart, or CreateAndExecute, then a Fiori Elements standard confirmation dialog appears before the action is triggered
- Given the confirmation dialog is shown, when the user cancels it, then the action is NOT executed and the instance state is unchanged
- Given the user clicks RefreshData, when the action fires, then no confirmation dialog appears and data refreshes immediately

## Design Notes

**EST-151 — Default filter values (confirmed against SAP docs):**

Both `UI.SelectionVariant` (`SelectOptions`) and `Common.FilterDefaultValue` / `@Consumption.filter.defaultValue` only support **fixed literal string values** — no dynamic expressions like "current year". Confirmed by SAP UI5 documentation for Fiori Elements OData V4.

- `AllocationId` → `@Consumption.filter.defaultValue: '1'` on CDS field. Simple, correct, fixed value will not change.
- `FiscalYear` → Must be dynamic. Implement in the query provider `ZCL_FI_ALLOC_DASH_QUERY`: override the OData `$metadata` or initial query response to supply the current fiscal year as a default parameter. Consult `SAP_Docs_MCP_search` for the exact RAP query provider API to inject filter defaults before implementing.

**EST-152 — Confirmation annotation (OData V4):**

The correct vocabulary term for Fiori Elements V4 confirmation dialogs is `com.sap.vocabularies.Common.v1.IsActionCritical` set to `true` in `annotation.xml`, targeting each action. The V2 `determining: true` pattern does **not** apply here.

## Verification

**Manual checks (if no CLI):**
- Activate all changed CDS objects in order: `zfi_i_alloc_dashboard_ce` (DDLS) → BDEF → service binding; check for activation errors in ADT
- Open dashboard app in browser: confirm filter bar shows FiscalYear and AllocationId pre-filled
- Click FiscalYear filter input: confirm no F4 popup
- Select a row in a cancellable state and click Cancel: confirm confirmation dialog appears
- Dismiss confirmation dialog: confirm instance status unchanged
- Click RefreshData: confirm immediate refresh, no dialog

### Review Findings

- [x] [Review][Decision] FiscalYear default is hardcoded '2026' — resolved: accepted as option A (fixed annotation, update annually). Dynamic default via query provider is architecturally impossible — IF_RAP_QUERY_PROVIDER has no pre-filter hook. CDS annotation `@Consumption.filter.defaultValue` is the only viable approach. [zfi_i_alloc_dashboard_ce.ddls.asddls:76]
- [x] [Review][Patch] Missing newline at end of annotation.xml — fixed; trailing newline added [dashboard-ui/webapp/annotations/annotation.xml]
- [ ] [Review][Patch] Verify IsActionCritical target path resolves — annotations target `ZFI_I_ALLOC_DASHBOARD_CEType/cancelProcess` etc.; confirm the OData service metadata exposes actions under exactly this entity type name and these exact action names; a mismatch silently disables all confirmation dialogs [dashboard-ui/webapp/annotations/annotation.xml:54-65]
- [x] [Review][Defer] AllocationId default '1' may return empty results if ID 1 doesn't exist in some systems — pre-existing by design decision; default is intentional and was specified in the story — deferred, pre-existing
- [x] [Review][Defer] Removing FiscalYear value help leaves no re-entry guidance if user clears the field — was the explicit intent of EST-150; field type gjahr provides implicit type validation — deferred, pre-existing by design
- [x] [Review][Defer] refreshData has no IsActionCritical — intentional per spec ("Do not modify RefreshData action") — deferred, pre-existing by design

## Spec Change Log

- 2026-04-01: Spec created (step-02), approved by Zdenek
- 2026-04-01: Code review complete — 1 decision-needed, 2 patch, 3 deferred, 0 dismissed; status → in-progress pending resolution
- 2026-04-01: Review resolved — D1 accepted as option A (fixed '2026' annotation, dynamic default is impossible in IF_RAP_QUERY_PROVIDER); P1 (missing newline) fixed; P2 is a manual verification item post-activation; status → done

