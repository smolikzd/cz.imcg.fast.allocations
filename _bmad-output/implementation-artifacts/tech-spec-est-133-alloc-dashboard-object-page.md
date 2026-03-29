---
title: 'Allocation Dashboard Object Page with Step Details'
slug: 'est-133-alloc-dashboard-object-page'
created: '2026-03-29'
status: 'completed'
stepsCompleted: [1, 2, 3, 4, 5]
tech_stack: ['ABAP 7.58', 'RAP Custom Entity', 'IF_RAP_QUERY_PROVIDER', 'CDS Annotations', 'OData V4', 'Fiori Elements List Report + Object Page']
files_to_modify: ['zfi_i_alloc_dashboard_ce.ddls.asddls', 'zcl_fi_alloc_dash_query.clas.abap', 'zfi_i_alloc_dash_step_ce.ddls.asddls (NEW)', 'zfi_i_alloc_dash_step_ce.ddls.xml (NEW)', 'zcl_fi_alloc_dash_step_qry.clas.abap (NEW)', 'zcl_fi_alloc_dash_step_qry.clas.xml (NEW)', 'zfi_ui_alloc_dashboard.srvd.srvdsrv']
code_patterns: ['Custom entity COMPOSITION OF child entity', 'IF_RAP_QUERY_PROVIDER per entity', 'SELECT from ZFI_I_PROCESS_STEP_STATS + JOIN zfi_proc_def for description', '@UI.facet for object page layout', 'format_timestamp pattern from parent query class']
test_patterns: ['Manual Fiori Elements preview testing via service binding']
---

# Tech-Spec: Allocation Dashboard Object Page with Step Details

**Created:** 2026-03-29

## Overview

### Problem Statement

The Allocation Dashboard is a flat list report with no drill-down capability. Users cannot see which process steps ran for a given allocation run in a specific fiscal period, their individual statuses, timestamps, or substep completion counts.

### Solution

Add a Fiori Elements object page to the existing dashboard custom entity. The object page displays all dashboard fields organized in meaningful facets (General, Business Status, Process) plus a table facet showing step-level rows sourced from `ZFI_I_PROCESS_STEP_STATS`. Step rows show aggregated substep counts (total, completed, failed, running, queued) rather than individual substep rows. A new child custom entity and query provider class serve the step data. Empty periods (ProcessInstanceId = 0) are navigable but show empty/default values.

### Scope

**In Scope:**
- Object page annotations on `ZFI_I_ALLOC_DASHBOARD_CE` (header facets, field groups)
- New child custom entity `ZFI_I_ALLOC_DASH_STEP_CE` for step-level data
- New query provider class `ZCL_FI_ALLOC_DASH_STEP_QRY` implementing `IF_RAP_QUERY_PROVIDER`
- Service definition update to expose the step entity
- Composition relationship from dashboard entity to step entity
- Step table showing: StepNumber, Description, Status (with criticality), StartedAt, EndedAt, TotalSubsteps, CompletedSubsteps, FailedSubsteps, RunningSubsteps, QueuedSubsteps

**Out of Scope:**
- Substep-level drill-down (only step-level with aggregated substep counts)
- Actions on the object page (restart, cancel, etc.)
- Editing capabilities
- ABAP Unit tests (manual testing via Fiori Elements preview)

## Context for Development

### Codebase Patterns

- Custom entities use `@ObjectModel.query.implementedBy` with `IF_RAP_QUERY_PROVIDER`
- SAP docs confirm: custom entities CAN have `COMPOSITION OF` child custom entities — each with its own query provider
- Reference: `DEMO_SALES_CUSTOM_COMPOSITION` → `DEMO_SALES_CUSTOM_CHILD` in SAP keyword docs
- Dashboard pattern established by EST-132: `ZFI_I_ALLOC_DASHBOARD_CE` + `ZCL_FI_ALLOC_DASH_QUERY`
- Step stats CDS view `ZFI_I_PROCESS_STEP_STATS` (planner repo) aggregates substep counts per step from `ZFI_I_PROCESS_STEP`
- Step description comes from `zfi_proc_def` table (joined on ProcessType + StepNumber) — same pattern as `ZFIPROC_R_STATS_TP`
- The link from dashboard to steps: `ProcessInstanceId` on dashboard row → `InstanceId` on step records
- Empty periods have `ProcessInstanceId = 0` — step query returns empty result for these
- Timestamps in step data: `StepStartedAt`/`StepEndedAt` are packed UTC (`timestampl`), same as parent entity
- Substep counts are `abap.int8` from CDS SUM aggregation
- Step status is derived in CDS: FAILED > RUNNING > COMPLETED > PENDING
- Constitution: DDIC-First (ty_row exception documented), 120-char line limit, ABAP-Doc on all public methods

### Files to Reference

| File | Purpose |
| ---- | ------- |
| `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` | Parent custom entity — ADD object page annotations + `COMPOSITION OF` child |
| `ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_query.clas.abap` | Parent query class — reference for pattern (format_timestamp, get_status_criticality) |
| `ovysledovka/src/zfi_alloc_process/zfi_ui_alloc_dashboard.srvd.srvdsrv` | Service definition — ADD exposure of step entity |
| `planner/src/zfi_i_process_step_stats.ddls.asddls` | Step stats CDS view — data source for step query (SELECT source) |
| `planner/src/zfiproc_r_stats_tp.ddls.asddls` | RAP restricted view — reference for field names, joins to `zfi_proc_def` |
| `planner/src/zfi_proc_step.tabl.xml` | Step table structure (15 fields, step+substep in same table) |
| `planner/src/zfi_proc_def.tabl.xml` | Step definition table (description, class_name, step_type per process_type+step_number) |

### Technical Decisions

- **Composition syntax**: Parent declares `_Steps : composition [0..*] of ZFI_I_ALLOC_DASH_STEP_CE`; child declares `_Dashboard : association to parent ZFI_I_ALLOC_DASHBOARD_CE` with ON condition matching all 4 parent key fields. SAP docs (`ABENCDS_F1_CUSTOM_TP_ASSOCIATION`) confirm custom entities fully support `association to parent` syntax — the to-parent association is required for a valid CDS composition tree.
- **Step-level granularity only** — ~5 rows per instance, substep counts aggregated
- **Data source**: `ZFI_I_PROCESS_STEP_STATS` for counts + timestamps, JOIN `zfi_proc_def` for description/class_name in ABAP (not CDS, since custom entity has no SQL view)
- **Step query receives parent key via filter** — Fiori Elements automatically passes parent key fields as filters to child entity query
- **Object page navigable for empty periods** (ProcessInstanceId = 0) — step table simply empty
- **Timestamps formatted as `char19`** — same pattern as parent, reuse `format_timestamp` logic
- **Step status criticality** — reuse same `get_status_criticality` logic (COMPLETED=green, FAILED=red, RUNNING=yellow)
- **StepStatus semantic note**: The CDS view `ZFI_I_PROCESS_STEP_STATS` derives `StepStatus` from ALL rows in the GROUP BY (including the step header row where `SubstepNumber = '000000000000'`), while substep count fields (`TotalSubsteps`, etc.) explicitly exclude the header row. This means `StepStatus` may show `PENDING` even when all substep counts show completed, if the step header row itself is not yet `COMPLETED`. This is accepted as-is for MVP — the CDS view is owned by the planner repo and its semantics are correct (step status reflects the overall step lifecycle, not just substep progress).
- **StepStatus CDS type**: The CDS CASE expression returns an untyped string literal (inferred `SSTRING`). In the ABAP query class, cast the value to `zfi_process_status` when populating the result row to ensure type consistency.
- **No `@AccessControl.authorizationCheck`** on custom entities (learned in EST-132)
- **No DDLX** — all annotations inline in custom entity (learned in EST-132)

## Implementation Plan

### Tasks

- [ ] Task 1: Create child custom entity `ZFI_I_ALLOC_DASH_STEP_CE`
  - File: `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dash_step_ce.ddls.asddls` (NEW)
  - File: `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dash_step_ce.ddls.xml` (NEW)
  - Action: Create a custom entity with `@ObjectModel.query.implementedBy: 'ABAP:ZCL_FI_ALLOC_DASH_STEP_QRY'`
  - Key fields (must include parent key + own key):
    - `key CompanyCode : bukrs` — parent key
    - `key FiscalYear : gjahr` — parent key
    - `key FiscalPeriod : fins_fiscalperiod` — parent key
    - `key AllocationId : zfi_alloc_id` — parent key
    - `key StepNumber : zfi_process_step_number` — step's own key
  - Non-key fields:
    - `ProcessInstanceId : zfi_process_instance_id` — resolved instance ID (`@UI.hidden: true`, for debugging)
    - `Description : zfi_process_description` — step description from `zfi_proc_def`
    - `StepStatus : zfi_process_status` — derived status (FAILED/RUNNING/COMPLETED/PENDING)
    - `StepStatusText : zfi_process_status_desc` — human-readable text (`@UI.hidden: true`)
    - `StatusCriticality : abap.int1` — criticality integer (`@UI.hidden: true`)
    - `StepStartedAt : abap.char(19)` — formatted timestamp
    - `StepEndedAt : abap.char(19)` — formatted timestamp
    - `TotalSubsteps : abap.int8` — total substep count
    - `CompletedSubsteps : abap.int8` — completed substep count
    - `FailedSubsteps : abap.int8` — failed substep count
    - `RunningSubsteps : abap.int8` — running substep count
    - `QueuedSubsteps : abap.int8` — queued substep count
  - Annotations:
    - `@UI.headerInfo` with `typeName: 'Process Step'`, `typeNamePlural: 'Process Steps'`
    - `@UI.lineItem` on all visible fields with positions 10-110 and appropriate `importance`
    - `@EndUserText.label` on ALL fields that use built-in types without inherited labels: `StepStartedAt` ('Started At'), `StepEndedAt` ('Ended At'), `TotalSubsteps` ('Total Substeps'), `CompletedSubsteps` ('Completed'), `FailedSubsteps` ('Failed'), `RunningSubsteps` ('Running'), `QueuedSubsteps` ('Queued'), `StatusCriticality` ('Status Criticality'), `StepStatusText` ('Status Text'), `ProcessInstanceId` ('Process Instance')
    - `StepStatus` with `criticality: 'StatusCriticality'` and `criticalityRepresentation: #WITH_ICON`
    - `@ObjectModel.text.element: ['StepStatusText']` on `StepStatus` for text arrangement
    - No `@AccessControl.authorizationCheck` (invalid on custom entities)
    - No `@Consumption.filter` / `@selectionField` (child entity has no filter bar)
  - Notes: Create matching `.ddls.xml` metadata file with `<DDLS>` structure, same pattern as parent entity's XML file

- [ ] Task 2: Add composition and object page annotations to parent entity `ZFI_I_ALLOC_DASHBOARD_CE`
  - File: `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` (MODIFY)
  - Action 2a — Add composition association at the end of the entity body:
    ```
    _Steps : composition [0..*] of ZFI_I_ALLOC_DASH_STEP_CE;
    ```
  - Action 2b — Add `@UI.facet` annotation at entity level (inside the existing `@UI` block before `define custom entity`):
    - `COLLECTION` facet `'General'` (label: 'General Information') containing `FIELDGROUP_REFERENCE` `'General'`
    - `COLLECTION` facet `'BusinessStatus'` (label: 'Business Status') containing `FIELDGROUP_REFERENCE` for business status fields — add `fieldGroup: [{ qualifier: 'BusinessStatus', position: 10 }]` to `BusinessStatus1` and `fieldGroup: [{ qualifier: 'BusinessStatus', position: 20 }]` to `BusinessStatus2`
    - `COLLECTION` facet `'Process'` (label: 'Process Details') containing `FIELDGROUP_REFERENCE` `'Process'`
    - `LINEITEM_REFERENCE` facet `'Steps'` (label: 'Process Steps') targeting `_Steps`
  - Action 2c — **ADD** a second `fieldGroup` entry to `BusinessStatus1` and `BusinessStatus2`:
    - `BusinessStatus1` gets: `fieldGroup: [{ qualifier: 'Process', position: 10 }, { qualifier: 'BusinessStatus', position: 10 }]` (KEEP existing `'Process'` entry, ADD `'BusinessStatus'`)
    - `BusinessStatus2` gets: `fieldGroup: [{ qualifier: 'Process', position: 20 }, { qualifier: 'BusinessStatus', position: 20 }]` (KEEP existing `'Process'` entry, ADD `'BusinessStatus'`)
    - This means both fields appear in BOTH the 'Process Details' and 'Business Status' facets on the object page. This is intentional — business statuses are logically part of the process but deserve their own dedicated facet for visibility.
  - Notes: The parent already has `fieldGroup` annotations with qualifier `'General'` on key fields and `'Process'` on process-related fields. These are reused by the `FIELDGROUP_REFERENCE` facets.

- [ ] Task 3: Create step query provider class `ZCL_FI_ALLOC_DASH_STEP_QRY`
  - File: `ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_step_qry.clas.abap` (NEW)
  - File: `ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_step_qry.clas.xml` (NEW)
  - Action: Create class implementing `IF_RAP_QUERY_PROVIDER`
  - Class structure:
    - `PUBLIC SECTION`: `INTERFACES if_rap_query_provider.`
    - `PRIVATE SECTION`:
      - `ty_row` local type matching child entity fields including `ProcessInstanceId` (pragmatic DDIC-First exception)
      - `tt_rows` table type of `ty_row`
      - `ty_raw_row` for CDS select result: `InstanceId` (key), `StepNumber` (key), `ProcessType`, timestamps as `timestampl` (`StepStartedAt`, `StepEndedAt`), counts as `int8` (`TotalSubsteps`, `RunningSubsteps`, `CompletedSubsteps`, `QueuedSubsteps`, `FailedSubsteps`), `StepStatus` as `zfi_process_status`
      - `tt_raw_rows` table type of `ty_raw_row`
      - Criticality constants (same values as parent: 3=positive, 1=negative, 2=critical, 0=neutral)
      - `get_status_criticality` method (same logic as parent class)
      - `get_status_text` method (same logic as parent class)
      - `format_timestamp` method (same logic as parent class)
  - `if_rap_query_provider~select` implementation:
    1. Wrap entire body in `TRY.` / `CATCH cx_rap_query_filter_no_range.` — on exception, return empty result (same pattern as parent class). Also handle any unexpected exceptions gracefully with early RETURN + empty result.
    2. Extract filters from `io_request->get_filter()->get_as_ranges()`
    3. Extract parent key values from filter ranges: `CompanyCode`, `FiscalYear`, `FiscalPeriod`, `AllocationId` (read the LOW value from each EQ range entry)
    4. Look up `ProcessInstanceId`: use **`SELECT SINGLE ProcessInstanceId`** from `ZFI_I_ALLOC_DASHBOARD` WHERE `CompanyCode = lv_cc AND FiscalYear = lv_fy AND FiscalPeriod = lv_fp AND AllocationId = lv_aid`. SELECT SINGLE ensures exactly one row; the 4 parent key fields form the unique key of the enrichment view.
    5. If `ProcessInstanceId` is initial or 0, return empty result (no steps for empty periods)
    6. SELECT from `ZFI_I_PROCESS_STEP_STATS` WHERE `InstanceId = lv_instance_id` into `tt_raw_rows`
    7. For each raw row, resolve description from `zfi_proc_def` — bulk SELECT: collect distinct `ProcessType` + `StepNumber` pairs from raw rows, then `SELECT process_type, step_number, description FROM zfi_proc_def FOR ALL ENTRIES IN @lt_keys WHERE process_type = ... AND step_number = ...`
    8. Convert timestamps via `format_timestamp`, cast `StepStatus` to `zfi_process_status`, compute criticality/text
    9. Fill parent key fields (CompanyCode, FiscalYear, FiscalPeriod, AllocationId) and `ProcessInstanceId` from lookup into each result row
    10. Handle paging (`get_offset`, `get_page_size`) and total record count
    11. `io_response->set_data( lt_result )`
  - Notes:
    - ABAP-Doc on all public methods (constitution requirement)
    - 120-char line limit enforced
    - No dynamic WHERE needed — step query is simple (single InstanceId)
    - No sorting needed — default by StepNumber ascending is fine (~5 rows)
    - Create matching `.clas.xml` metadata file

- [ ] Task 4: Update service definition to expose child entity
  - File: `ovysledovka/src/zfi_alloc_process/zfi_ui_alloc_dashboard.srvd.srvdsrv` (MODIFY)
  - Action: Add `expose ZFI_I_ALLOC_DASH_STEP_CE;` after the existing expose line
  - Notes: Service binding `ZFI_UI_ALLOC_DASHBOARD_O4` must be re-published in SAP after this change

- [ ] Task 5: Activate and test in SAP
  - Action: Activate all objects in correct dependency order:
    1. `ZCL_FI_ALLOC_DASH_STEP_QRY` (query class — no CDS dependencies, must exist before child entity references it)
    2. `ZFI_I_ALLOC_DASH_STEP_CE` (child entity — references query class via `@ObjectModel.query.implementedBy`)
    3. `ZFI_I_ALLOC_DASHBOARD_CE` (parent entity — references child entity via composition)
    4. `ZFI_UI_ALLOC_DASHBOARD` (service definition — references both entities)
    5. Re-publish service binding `ZFI_UI_ALLOC_DASHBOARD_O4`
  - Test: Open Fiori Elements preview from service binding
  - Notes: If activation errors occur, apply fixes iteratively (same approach as EST-132)

### Acceptance Criteria

- [ ] AC 1: Given the list report is displayed, when the user clicks a row (any fiscal period), then the object page opens showing the row's details organized in header area and body facets.

- [ ] AC 2: Given the object page is open, when the page loads, then the header title shows the computed AllocationTitle (e.g., `1000/2024/001/1`) and description shows ProcessTypeDescription (e.g., "Allocations").

- [ ] AC 3: Given the object page is open, when the page loads, then the header area shows an "Identification" field group with CompanyCode, FiscalYear, FiscalPeriod, and AllocationId, plus data points for "SAP Reporting" (BusinessStatus1) and "External Reporting" (BusinessStatus2).

- [ ] AC 4: Given the object page is open, when the page loads, then the "Process Details" body facet shows ProcessStatus (with criticality icon), ProcessType (with description text), ProcessInstanceId, StartedAt, EndedAt, and CreatedBy.

- [ ] AC 5: Given the object page is open for a row with ProcessInstanceId > 0, when the page loads, then the "Process Steps" table facet displays step rows with columns: StepNumber, Description, StepStatus (with criticality icon), StepStartedAt, StepEndedAt, TotalSubsteps, CompletedSubsteps, FailedSubsteps, RunningSubsteps, QueuedSubsteps.

- [ ] AC 6: Given the object page is open for an empty period (ProcessInstanceId = 0), when the page loads, then the "Process Steps" table is empty (no rows) and no error occurs.

- [ ] AC 7: Given a step has status FAILED, when displayed in the step table, then the StepStatus cell shows a red criticality icon. Given status COMPLETED, then green. Given RUNNING, then yellow/warning.

- [ ] AC 8: Given step timestamps are populated (non-zero), when displayed, then they render as `YYYY-MM-DD HH:MM:SS`. Given timestamps are zero/initial, then the cell is blank.

- [ ] AC 9: Given the service definition exposes both `ZFI_I_ALLOC_DASHBOARD_CE` and `ZFI_I_ALLOC_DASH_STEP_CE`, when the service binding is published, then both entities are accessible via OData V4 metadata.

## Additional Context

### Dependencies

- **EST-132** (Allocation Dashboard MVP) must be active and working in SAP — provides the parent entity, query class, service definition, and service binding
- **`ZFI_I_PROCESS_STEP_STATS`** CDS view must be active in system (planner repo) — provides aggregated step data. **Note**: This view has `@AccessControl.authorizationCheck: #CHECK`. If an access control object exists for it, the step query SELECT will be subject to authorization. Verify in SAP that no restrictive DCL exists, or that the dashboard user has the required authorization. If the step table is unexpectedly empty for a real instance, check access control first.
- **`zfi_proc_def`** table must contain step definitions for the allocation process type — provides step descriptions
- **`ZFI_I_ALLOC_DASHBOARD`** enrichment CDS view (from EST-132) — used to resolve parent key → ProcessInstanceId in the step query
- Service binding `ZFI_UI_ALLOC_DASHBOARD_O4` must be re-published after changes

### Testing Strategy

- **No ABAP Unit tests** — manual testing via Fiori Elements preview (consistent with MVP approach)
- Manual test steps:
  1. Open service binding `ZFI_UI_ALLOC_DASHBOARD_O4` → Preview
  2. Fill mandatory filters (CompanyCode, FiscalYear, AllocationId) and execute
  3. Click a row with a real ProcessInstanceId → verify object page opens with 4 facets
  4. Verify "General Information" facet field values match the list row
  5. Verify "Business Status" facet shows both status fields
  6. Verify "Process Details" facet shows all process fields with correct formatting
  7. Verify "Process Steps" table shows step rows with criticality icons and formatted timestamps
  8. Navigate back, click an empty-period row (ProcessInstanceId = 0) → verify object page opens, step table is empty, no errors
  9. Verify step count columns (Total, Completed, Failed, Running, Queued) show correct integer values

### Notes

- **Pattern establishment**: This is the first composition/child entity in the ovysledovka dashboard — establishes the pattern for future drill-downs (e.g., substep detail page)
- **Parent entity key is composite**: CompanyCode + FiscalYear + FiscalPeriod + AllocationId — all 4 fields must be included as key fields in the child entity
- **ProcessInstanceId lookup**: The step query must look up `ProcessInstanceId` from the parent CDS view using **`SELECT SINGLE`** with the 4 parent key fields, because Fiori Elements passes the parent key fields (not ProcessInstanceId) as filters to the child query. The 4 fields form the unique key of the enrichment view, so SELECT SINGLE is safe.
- **Code duplication**: `format_timestamp`, `get_status_criticality`, and `get_status_text` are duplicated from the parent query class. This is acceptable for MVP. **Backlog**: Create a future story to extract these into a shared utility class `ZCL_FI_ALLOC_DASH_UTILS` to avoid divergence when new status values are added (e.g., `TIMEOUT`).
- **AvgSubstepDuration excluded**: Deliberately excluded from MVP scope. The field is available in `ZFI_I_PROCESS_STEP_STATS` but requires unit-of-measure annotation handling (`@Semantics.quantity.unitOfMeasure`) which adds complexity. Can be added in a follow-up enhancement once the basic step table is proven.
- **~5 rows per instance**: Step table is small enough that paging/sorting are not critical concerns, but basic paging support is included for correctness
- **High-risk item**: Composition syntax on custom entities — while confirmed in SAP docs, this is a less common pattern. If activation fails, fallback is to use an `association [0..*]` instead of `composition [0..*]` and reference it via `@UI.facet` with `targetElement`
- **Activation order matters**: Query class must be activated before child entity (child references class via `@ObjectModel.query.implementedBy`); child entity before parent (parent references child via composition)

## Constitution Compliance

| Principle | Applicability | Notes |
|-----------|--------------|-------|
| **I — DDIC-First** | Applies | All custom entity fields use DDIC types (`bukrs`, `gjahr`, `zfi_process_status`, etc.). Local `ty_row`/`ty_raw_row` types in query class are documented pragmatic exception (single-use internal structures, never exposed). |
| **II — SAP Standards** | Applies | SAP naming conventions followed: `ZFI_I_*` for CDS entities, `ZCL_FI_*` for classes. Line length ≤ 120 chars enforced by abaplint. |
| **III — Consult SAP Docs** | Applied | Custom entity composition syntax verified against SAP keyword docs (`ABENCDS_F1_CUSTOM_COMPOSITION`). |
| **IV — Factory Pattern** | N/A | No object instantiation in this feature — query provider is instantiated by the RAP framework. |
| **V — Error Handling** | Applies | Step query `select` method must wrap filter extraction in TRY-CATCH for `cx_rap_query_filter_no_range`. Return empty result on error (no crash — consistent with parent class pattern). |
| **Code Documentation** | Applies | ABAP-Doc required on all public methods of `ZCL_FI_ALLOC_DASH_STEP_QRY`. |

## Header Redesign

After the core object page implementation, the header was redesigned to improve usability:

- **Computed title field**: Added `AllocationTitle : abap.char(50)` to parent entity, populated in query class as `{CompanyCode}/{FiscalYear}/{FiscalPeriod}/{AllocationId}`. NOT marked `@UI.hidden` (hidden blocks headerInfo rendering — see discovery below). Not added to sort/filter whitelist (computed field, not in CDS view). Field won't appear in tables/forms because it has no `@UI.lineItem`/`@UI.fieldGroup`/`@UI.selectionField` annotations.
- **headerInfo changes**: `title.value` changed from `CompanyCode` to `AllocationTitle`, `description.value` changed from `FiscalPeriod` to `ProcessTypeDescription` (changed from `ProcessStatusText` during iteration — status is already visible via criticality icons)
- **Key fields moved to header**: All 4 key fields (CompanyCode, FiscalYear, FiscalPeriod, AllocationId) placed in a `fieldGroup` with qualifier `HeaderIdent`, displayed via `purpose: #HEADER` + `type: #FIELDGROUP_REFERENCE` facet in the Object Page header area
- **Business statuses as header data points**: `BusinessStatus1` and `BusinessStatus2` moved from body facet to header using `@UI.dataPoint` with qualifiers `BusinessStatus1`/`BusinessStatus2` (titles: "SAP Reporting"/"External Reporting"), referenced via `purpose: #HEADER` + `type: #DATAPOINT_REFERENCE` facets
- **Body facets simplified**: Removed "General Information" and "Business Status" body facets. Remaining: "Process Details" (position 10), "Process Steps" (position 20)
- **`@UI.hidden` blocks headerInfo**: Fields referenced by `headerInfo.title.value` or `headerInfo.description.value` must NOT have `@UI.hidden: true` — Fiori Elements will not render them as Object Page title/subtitle. Fields without `@UI.lineItem`/`@UI.fieldGroup`/`@UI.selectionField` annotations won't appear in tables/forms anyway, so removing `@UI.hidden` is safe. Commit: `2e51444`.
- **Positional column mapping bug in `resolve_type_descriptions`**: `SELECT process_type, description FROM zfi_proc_type INTO TABLE @lt_types` mapped columns positionally (process_type→MANDT, description→PROCESS_TYPE) because `lt_types` was typed with the full table structure. Changed to `INTO CORRESPONDING FIELDS OF TABLE` to map by name. This bug was present since EST-132 and caused `ProcessTypeDescription` to always be empty. Commit: `956e4ec`.
- **Subtitle changed**: `description.value` was initially set to `ProcessStatusText` but was changed to `ProcessTypeDescription` (e.g., "Allocations") for better usability — the status is already shown via data points and criticality icons. Commit: `7cd14e1`.
- **Commits**: `ef30330` (header redesign) → `2e51444` (hidden fix) → `7cd14e1` (subtitle change) → `956e4ec` (CORRESPONDING fix), all on `main` in ovysledovka repo

## Review Notes

- Adversarial review completed with 14 findings
- Findings: 14 total, 2 fixed, 12 skipped
- Resolution approach: Walk-through (F1, F7 fixed; F2-F6, F8-F14 skipped as accepted/noise)
- **F1 (CRITICAL) — FIXED**: Added `association to parent ZFI_I_ALLOC_DASHBOARD_CE` to child entity `ZFI_I_ALLOC_DASH_STEP_CE`. SAP docs (`ABENCDS_F1_CUSTOM_TP_ASSOCIATION`, `ABENCDS_COMPOSITION_GLOSRY`) confirm to-parent association is required for a valid CDS composition tree, and custom entities fully support the syntax. The original tech spec claim was corrected.
- **F7 (MEDIUM) — FIXED**: `CONV_NO_NUMBER` runtime dump — comparing CHAR(32) `ProcessInstanceId` to numeric `0` (`lv_instance_id = 0`) caused dump. Changed to `lv_instance_id IS INITIAL` only. Commit `d37222c`.
- **F2 (HIGH) — SKIPPED**: StepStatus CDS CASE inferred type — handled by `CONV` in ABAP query class; CDS view is in planner repo (out of scope)
- **F4 (MEDIUM) — SKIPPED**: No sorting support — ~5 rows with hardcoded `ORDER BY StepNumber`; acceptable for MVP
- **F3, F5, F6, F8-F14 (LOW/NOISE) — SKIPPED**: Code duplication (documented backlog item), minor label gaps, etc.

## Activation Order

1. `ZCL_FI_ALLOC_DASH_STEP_QRY` — query class (no CDS dependencies; must exist before child entity references it)
2. `ZFI_I_ALLOC_DASH_STEP_CE` — child custom entity (references query class via `@ObjectModel.query.implementedBy`)
3. `ZFI_I_ALLOC_DASHBOARD_CE` — parent custom entity (references child via `composition [0..*]`)
4. `ZFI_UI_ALLOC_DASHBOARD` — service definition (exposes both entities)
5. Re-publish service binding `ZFI_UI_ALLOC_DASHBOARD_O4`
