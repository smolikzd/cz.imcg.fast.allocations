---
title: 'Allocation Process Dashboard (Fiori Elements)'
slug: 'est-132-alloc-process-dashboard'
created: '2026-03-27'
updated: '2026-03-28'
status: 'completed'
stepsCompleted: [1, 2, 3, 4]
adversarial_review: 'completed (Round 1: 21 findings resolved; Round 2: 21 findings resolved; Round 3: 17 findings resolved; Feynman Technique: 5 fixes applied)'
tech_stack:
  - 'SAP RAP (Custom Entity with IF_RAP_QUERY_PROVIDER)'
  - 'CDS View Entities (I_ layer for calendar grid)'
  - 'OData V4'
  - 'Fiori Elements List Report'
  - 'ABAP 7.58'
files_to_modify:
  - 'ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dash_1.ddls.asddls (modify - restore AllocationId with COALESCE, remove hardcoded WHERE)'
  - 'ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard.ddls.asddls (modify - add AllocationId to key)'
  - 'ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls (create - custom entity definition)'
  - 'ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_query.clas.abap (create - query provider class)'
  - 'ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_query.clas.xml (create - class XML metadata for abapGit)'
  - 'ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddlx.asddlxs (create - DDLX metadata extension)'
  - 'ovysledovka/src/zfi_alloc_process/zfi_ui_alloc_dashboard.srvd.srvdsrv (create - service definition)'
   - 'ovysledovka/src/zfi_alloc_process/zfi_ui_alloc_dashboard_o4.srvb.xml (create in SAP - service binding)'
   - 'ovysledovka/src/zfi_process_status_desc.dtel.xml (create - DDIC data element for status description text, domain DESCR = CHAR80; at src/ root, not in zfi_alloc_process/ subpackage — consistent with all other DTEL files in the repo)'
code_patterns:
  - 'Custom entity with @ObjectModel.query.implementedBy (same pattern as health check in planner repo)'
  - 'I_ views for calendar-grid LEFT OUTER JOIN (reusable data layer)'
  - 'Query class implements IF_RAP_QUERY_PROVIDER, handles filtering/sorting/paging'
  - 'COALESCE(allocation_id, 0) for NULL-safe keys from LEFT OUTER JOIN'
  - 'StatusCriticality computed field for RAP criticality coloring'
  - 'All @UI annotations in DDLX, annotating the custom entity'
  - 'Importance tiers: #HIGH (always visible), #MEDIUM (default visible), #LOW (on demand)'
  - 'Service definition exposes custom entity, service binding uses _O4 suffix'
test_patterns:
  - 'No ABAP Unit tests for MVP — manual testing via Fiori preview in /sap/bc/adt/businessservices/odatav4/'
  - 'Verify OData V4 metadata document loads correctly'
  - 'Verify list report renders with all columns in Fiori Elements preview'
  - 'Verify rows with ProcessInstanceId show process status, business statuses, timestamps'
  - 'Verify empty periods appear with AllocationId=0 and empty process fields'
  - 'Verify criticality coloring: COMPLETED=green, FAILED/CANCELLED=red, RUNNING=yellow'
---

# Tech-Spec: Allocation Process Dashboard (Fiori Elements)

**Created:** 2026-03-27
**Updated:** 2026-03-28 (post-adversarial review Round 1: switched from managed BO to custom entity; Round 2: DDIC types, mandatory filters, sort, row behavior, DDLX fixes; Round 3: DDIC timestamp types, status description type, @ObjectModel.text.element placement, get_as_sql_string() filter pattern, value help definitions, JOIN alias qualification; Feynman Technique: DTEL activation order, alias naming for get_as_sql_string(), is_data_requested() guard, ORDER BY implementation hint)
**Linear Issue:** EST-132
**Target Repository:** cz.imcg.fast.ovysledovka (ZFI_ALLOC_PROCESS package)

## Overview

### Problem Statement

There is no allocation-specific monitoring screen. Users must check `ZFI_ALLOC_STATE` via SE16 or use the generic process instance Fiori app which lacks allocation context (company code, fiscal period, allocation ID). There's no way to see all allocation runs for a year at a glance.

### Solution

Build a RAP-based Fiori Elements List Report using a **custom entity** with `IF_RAP_QUERY_PROVIDER`. The I_ CDS views provide a calendar-grid approach: `I_FiscalYearPeriodForCmpnyCode` LEFT OUTER JOIN to `ZFI_ALLOC_STATE` ensures all fiscal periods (including special periods 013-016 if defined for the company code's fiscal year variant) appear per company code, even without allocation runs. The query class reads from the I_ views, joins process instance data, and returns flat results with `COALESCE` for NULL-safe keys.

**Why custom entity instead of managed BO:** The dashboard is a read-only projection of data from multiple sources (SAP fiscal calendar, allocation state table, process instances). It has no persistent table of its own — the LEFT OUTER JOIN produces virtual rows for periods without allocations. A managed BO requires a persistent table and non-null keys, which conflicts with this pattern. Custom entity with `IF_RAP_QUERY_PROVIDER` gives full control over the SELECT logic. This is the same pattern used for `ZFIPROC_I_HEALTH_CHK_CE` in the planner repo.

### Scope

**In Scope:**

- CDS view stack in `ovysledovka` repo (`ZFI_ALLOC_PROCESS` package):
  - **I_ base view** (`ZFI_I_ALLOC_DASH_1`): `I_FiscalYearPeriodForCmpnyCode` LEFT OUTER JOIN `ZFI_ALLOC_STATE` — produces complete calendar grid
  - **I_ enrichment view** (`ZFI_I_ALLOC_DASHBOARD`): association to `ZFIPROC_R_INSTANCE_TP` for process details
  - **Custom entity** (`ZFI_I_ALLOC_DASHBOARD_CE`): flat entity definition with `@ObjectModel.query.implementedBy`
  - **Query class** (`ZCL_FI_ALLOC_DASH_QUERY`): implements `IF_RAP_QUERY_PROVIDER`, reads from I_ views + process instance data
  - **DDLX metadata extension** (`ZFI_I_ALLOC_DASHBOARD_CE`): full Fiori Elements list report annotations
  - **SRVD service definition** (`ZFI_UI_ALLOC_DASHBOARD`)
  - **SRVB OData V4 binding** (`ZFI_UI_ALLOC_DASHBOARD_O4`) — created in SAP, pulled via abapGit
- Key columns: company code, fiscal year, fiscal period, allocation ID (0 = no allocation)
- Display columns: process type (with description), process status (with text + criticality), business status 1/2, started at, ended at, created by, active, locked, description
- Filter/selection fields: company code (**mandatory**), fiscal year (**mandatory**), fiscal period, allocation ID (**mandatory**), process status
- `COALESCE(allocation_id, 0)` in I_ base view for NULL-safe keys
- Status criticality coloring: COMPLETED=green(3), FAILED/CANCELLED=red(1), RUNNING/PENDING/EXECREQ/RESTREQ=yellow(2), others=neutral(0)

**Out of Scope:**

- Individual phase columns (PHASE1/2/3 start/end/status) — status conveyed via process status + business statuses
- Drill-down navigation to process instance detail (object page facets pre-defined in DDLX for future use)
- Actions (rerun, cancel, restart) — can be added to custom entity later
- Aggregate statistics views
- Matrix/cross-tab layout
- Authorization checks (conscious MVP decision — future story)

## Context for Development

### Codebase Patterns

**Custom Entity Pattern (from planner repo `ZFIPROC_I_HEALTH_CHK_CE`):**

| Concern | Where |
|---------|-------|
| Entity definition with typed fields | Custom entity CDS (`define custom entity`) |
| `@ObjectModel.query.implementedBy: 'ABAP:ZCL_...'` | Custom entity CDS |
| Data retrieval, filtering, sorting, paging | Query class (`IF_RAP_QUERY_PROVIDER~select`) |
| `@UI.*` annotations | DDLX metadata extension |
| Service exposure | SRVD (exposes custom entity directly) |

**No BDEF, no behavior pool, no C_ projection** — custom entities are self-contained.

**Calendar-Grid Pattern:**
- `I_FiscalYearPeriodForCmpnyCode` LEFT OUTER JOIN `ZFI_ALLOC_STATE`
- All fiscal periods defined for the company code's fiscal year variant appear per company code per year, even without allocation data (includes special periods 013-016 if applicable)
- `COALESCE(allocation_id, 0)` ensures AllocationId is never NULL (0 = no allocation run)
- Rows with AllocationId=0 have NULL ProcessInstanceId and empty process fields
- Periods WITH allocation data produce N rows (one per AllocationId). Periods WITHOUT allocation data produce 1 row with AllocationId=0. There is NO extra AllocationId=0 row for periods that already have allocations. (e.g., period 003 with allocations 1, 2, 3 produces exactly 3 rows; period 007 with no allocations produces 1 row with AllocationId=0)

**Cross-Repo Data Access:**
- Query class reads from `ZFI_I_ALLOC_DASHBOARD` (I_ enrichment view in ovysledovka)
- Joins to `zfi_proc_inst` (process instance table in planner repo) for process details
- Text resolution for ProcessType via `zfi_proc_type` table
- Text resolution for ProcessStatus via `ZFIPROC_I_InstanceStatus_VH` CDS view (domain fixed values)
- All tables in same SAP system — cross-package SELECT works

### Files to Reference

| File | Purpose |
| ---- | ------- |
| `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dash_1.ddls.asddls` | I_ base view (prototype, exists — needs AllocationId COALESCE + WHERE removal) |
| `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard.ddls.asddls` | I_ enrichment view (prototype, exists — needs AllocationId in key) |
| `ovysledovka/src/zfi_alloc_process/zfi_alloc_state.tabl.xml` | Allocation state table (PK: company_code/fiscal_year/fiscal_period/allocation_id) |
| `ovysledovka/src/zfi_phase_status.doma.xml` | Phase status domain (P/R/F/E/'') |
| `ovysledovka/src/zfi_alloc_id.dtel.xml` | Allocation ID data element (INT2) |
| `planner/src/zfiproc_i_health_chk_ce.ddls.asddls` | **Primary reference**: custom entity CDS definition |
| `planner/src/zcl_fiproc_health_chk_query.clas.abap` | **Primary reference**: query provider class (IF_RAP_QUERY_PROVIDER pattern) |
| `planner/src/zcl_fiproc_health_chk_query.clas.xml` | **Reference**: class XML metadata for abapGit |
| `planner/src/zfiproc_ui_health.srvd.srvdsrv` | **Reference**: service definition exposing custom entity |
| `planner/src/zfiproc_r_instance_tp.ddls.asddls` | R_ process instance view (data source for query class joins) |
| `planner/src/zfiproc_i_instancestatus_vh.ddls.asddls` | Instance status value help (StatusText field, domain ZFI_PROCESS_STATUS) |
| `planner/src/zfiproc_i_processtype.ddls.asddls` | Process type view (Description field, from zfi_proc_type table) |
| `planner/src/zfiproc_c_instance_tp.ddlx.asddlxs` | DDLX (reference: facets, importance tiers, selection fields) |

### Technical Decisions

| # | Decision | Choice | Rationale |
|---|----------|--------|-----------|
| AD-1 | BO type | Custom entity with `IF_RAP_QUERY_PROVIDER` | Dashboard is a read-only projection from multiple sources; no persistent table; LEFT OUTER JOIN produces virtual rows with NULL keys that managed BO cannot handle. Same pattern as health check. |
| AD-2 | Cross-repo data access | Query class joins to planner repo tables via SELECT | Custom entity query class does the SELECT; no CDS association needed at the entity level |
| AD-3 | Date/time handling | Timestamps come from process instance table (`started_at`, `ended_at`) | These are already TIMESTAMPL in `zfi_proc_inst`; no DATS_TIMS_TO_TSTMP conversion needed |
| AD-4 | DDLX metadata extension | Yes, full UI annotations on custom entity | Standard pattern, same as planner DDLX; annotates the custom entity for Fiori Elements |
| AD-5 | Phase scope | No individual phase columns | Status conveyed via process status + business statuses; phases visible in detail (future drill-down) |
| AD-6 | Naming prefix | `ZFI_` prefix (not `ZFIALLOC_`) | Consistent with existing ovysledovka CDS views (`ZFI_I_ALLOC_*`) |
| AD-7 | Service binding creation | Created in SAP system, pulled via abapGit | `.srvb` is XML-serialized, not source-editable CDS |
| AD-8 | NULL AllocationId handling | `COALESCE(allocation_id, 0)` in I_ base view | OData keys must be non-null; 0 semantically means "no allocation run for this period" |
| AD-9 | Status criticality | Computed `StatusCriticality` field (hidden) | COMPLETED=3(green), FAILED/CANCELLED=1(red), RUNNING/PENDING/EXECREQ/RESTREQ=2(yellow), others=0(neutral) |
| AD-10 | Authorization | `@AccessControl.authorizationCheck: #NOT_REQUIRED` | Conscious MVP decision — no auth checks. Future story to add authorization. |

### ZFI_ALLOC_STATE Table Structure (Key Fields for Dashboard)

| Field | Type | Key | Purpose |
|-------|------|-----|---------|
| `COMPANY_CODE` | FIS_BUKRS | PK | Company code |
| `FISCAL_YEAR` | FIS_GJAHR | PK | Fiscal year |
| `FISCAL_PERIOD` | FINS_FISCALPERIOD | PK | Fiscal period (001-012, special periods 013-016 if applicable) |
| `ALLOCATION_ID` | ZFI_ALLOC_ID (INT2) | PK | Allocation identifier |
| `PROCESS_INSTANCE_ID` | ZFI_PROCESS_INSTANCE_ID | | FK to `ZFI_PROC_INST` |
| `ACTIVE` | ZFI_ALLOC_ID_ACTIVE | | Active flag |
| `LOCKED` | ZFI_ALLOC_ID_LOCKED | | Lock flag |
| `DESCRIPTION` | ZFI_ALLOC_DESCRIPTION (domain DESCR = CHAR80) | | Allocation description |

### ZFI_PROCESS_STATUS Domain Fixed Values (Process Instance)

| Value | Meaning | Criticality |
|-------|---------|-------------|
| NEW | New | 0 (neutral) |
| RUNNING | Running | 2 (yellow) |
| COMPLETED | Completed | 3 (green) |
| FAILED | Failed | 1 (red) |
| CANCELLED | Cancelled | 1 (red) |
| PENDING | Pending | 2 (yellow) |
| SKIPPED | Skipped | 0 (neutral) |
| SUPERSEDED | Superseded | 0 (neutral) |
| EXECREQ | Execution Requested | 2 (yellow) |
| RESTREQ | Restart Requested | 2 (yellow) |

### Constitution Compliance

| Principle | Applicability | Notes |
|-----------|--------------|-------|
| **I. DDIC-First** | Fully applies | Custom entity fields use DDIC data elements (FIS_BUKRS, FIS_GJAHR, FINS_FISCALPERIOD, ZFI_ALLOC_ID, ZFI_PROCESS_INSTANCE_ID, ZFI_PROCESS_STATUS, ZFI_PROCESS_BUS_STATUS, ZFI_PROCESS_CREATED_BY, ZFI_ALLOC_ID_ACTIVE, ZFI_ALLOC_ID_LOCKED, ZFI_ALLOC_DESCRIPTION, ZFI_PROCESS_STARTED_AT, ZFI_PROCESS_ENDED_AT, ZFI_PROCESS_STATUS_DESC, etc.). Two documented exceptions: (1) `StatusCriticality : abap.int1` — no DDIC data element exists for RAP criticality integers. (2) Query class local `ty_row` structure — pragmatic exception consistent with reference pattern (`ZCL_FIPROC_HEALTH_CHK_QUERY` also uses local `ty_row`); the structure is purely internal to the query class, never exposed externally, and creating a DDIC structure for a single-use internal type would add complexity without benefit. Cross-package dependency on planner repo DDIC elements noted in Task 3. |
| **II. SAP Standards** | Fully applies | Naming: ZCL_FI_ for class, ZFI_I_ for views. Line length <= 120 chars. Class has ABAP-Doc. abapGit XML format. |
| **III. Consult SAP Docs** | Fully applies | `IF_RAP_QUERY_PROVIDER` API methods (`get_paging`, `get_sort_elements`, `get_filter`, `is_data_requested`, `is_total_numb_of_rec_requested`) verified against SAP documentation. Filter handling uses `get_as_sql_string()` pattern (SAP-documented for unmanaged queries). `@ObjectModel.text.element` placement confirmed: must be in CDS entity (MDE=`-`), not in DDLX. `@Consumption.filter.mandatory` confirmed valid in DDLX (MDE=`X`). `@UI.textArrangement` stays in DDLX alongside `@ObjectModel.text.element` in CDS — split placement pattern confirmed by SAP Fiori showcase. |
| **IV. Factory Pattern** | N/A | Query class instantiated by RAP framework, not by application code. No factory needed. |
| **V. Error Handling** | Partially applies | Query class catches RAP query exceptions (`CX_RAP_QUERY_FILTER_NO_RANGE` etc.). No `ZCX_FI_PROCESS_ERROR` needed — dashboard is read-only, no business logic errors. Unexpected errors propagate to RAP framework. |

## Implementation Plan

### Tasks

Tasks are ordered by dependency — lowest-level artifacts first, each building on the previous.

- [x] **Task 1: Fix I_ base view (`ZFI_I_ALLOC_DASH_1`)**
  - File: `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dash_1.ddls.asddls`
  - Action: Restore `AllocationId` as key field with COALESCE
    - Uncomment `state.allocation_id` and wrap: `key coalesce( state.allocation_id, 0 ) as AllocationId`
    - This ensures periods without allocation data get AllocationId=0 instead of NULL
    - The LEFT OUTER JOIN ON condition stays on company_code/fiscal_year/fiscal_period only (allocation_id is not a join field since `I_FiscalYearPeriodForCmpnyCode` has no allocation_id)
  - Action: Project `Active`, `Locked`, `Description` from `state` alias
    - Add `state.active as Active,`
    - Add `state.locked as Locked,`
    - Add `state.description as Description,`
    - These fields are needed by Task 2 (I_ enrichment view) and ultimately by the custom entity
  - Action: Remove hardcoded WHERE clause
    - Delete the entire `where` block (company codes 1000/2000, fiscal year 2024, periods 001-012)
    - Filtering handled by the query class and Fiori Elements filter bar
  - Notes: After this change, the view returns all company codes, all fiscal years, all periods. Periods without allocation data get AllocationId=0 and NULL Active/Locked/Description. Multiple allocations per period produce multiple rows.

- [x] **Task 2: Fix I_ enrichment view (`ZFI_I_ALLOC_DASHBOARD`)**
  - File: `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard.ddls.asddls`
  - Action: Add `AllocationId` to key and projection
    - Add `key AllocationId,` after `key FiscalPeriod,`
    - The field propagates from `ZFI_I_ALLOC_DASH_1` (already COALESCEd)
  - Action: Add `Active`, `Locked`, `Description` fields from DASH_1
    - These are useful dashboard columns from `ZFI_ALLOC_STATE` that are currently not projected
  - Notes: The `_Instance` association ON condition uses `ProcessInstanceId` — this remains unchanged. AllocationId is a key for uniqueness, not for the association.

- [x] **Task 3: Create custom entity (`ZFI_I_ALLOC_DASHBOARD_CE`)**
  - File: `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` (NEW)
  - Action: Create custom entity definition
  - Reference: `planner/src/zfiproc_i_health_chk_ce.ddls.asddls` (health check custom entity)
  - Annotations:
    - `@EndUserText.label: 'Allocation Dashboard'`
    - `@ObjectModel.query.implementedBy: 'ABAP:ZCL_FI_ALLOC_DASH_QUERY'`
    - `@AccessControl.authorizationCheck: #NOT_REQUIRED` (conscious MVP decision — AD-10)
  - Key fields (all non-nullable due to COALESCE in I_ view):
    - `key CompanyCode : bukrs;`
    - `key FiscalYear : gjahr;`
    - `key FiscalPeriod : fins_fiscalperiod;`
    - `key AllocationId : zfi_alloc_id;`
  - Display fields:
    - `ProcessInstanceId : zfi_process_instance_id;`
    - `@ObjectModel.text.element: ['ProcessTypeDescription']` on next field:
    - `ProcessType : zfi_process_proc_type;`
    - `ProcessTypeDescription : zfi_process_description;` (denormalized text for ProcessType)
    - `@ObjectModel.text.element: ['ProcessStatusText']` on next field:
    - `ProcessStatus : zfi_process_status;`
    - `ProcessStatusText : zfi_process_status_desc;` (denormalized text from domain fixed values — new DDIC data element `ZFI_PROCESS_STATUS_DESC`, domain DESCR = CHAR80)
    - `StatusCriticality : abap.int1;` (hidden, drives criticality coloring — no DDIC type exists for RAP criticality integer, built-in type justified)
    - `BusinessStatus1 : zfi_process_bus_status;`
    - `BusinessStatus2 : zfi_process_bus_status;`
    - `StartedAt : zfi_process_started_at;` (DDIC data element, domain ZFI_PROCESS_TIMESTAMP = DEC 21,7)
    - `EndedAt : zfi_process_ended_at;` (DDIC data element, domain ZFI_PROCESS_TIMESTAMP = DEC 21,7)
    - `CreatedBy : zfi_process_created_by;`
    - `Active : zfi_alloc_id_active;`
    - `Locked : zfi_alloc_id_locked;`
    - `Description : zfi_alloc_description;`
  - Value help annotations (on custom entity, since `@Consumption.valueHelpDefinition` has MDE scope):
    - `@Consumption.valueHelpDefinition: [{ entity: { name: 'I_CompanyCode', element: 'CompanyCode' } }]` on `CompanyCode`
    - `@Consumption.valueHelpDefinition: [{ entity: { name: 'I_FiscalYear', element: 'FiscalYear' } }]` on `FiscalYear`
    - `@Consumption.valueHelpDefinition: [{ entity: { name: 'ZFIPROC_I_InstanceStatus_VH', element: 'InstanceStatus' } }]` on `ProcessStatus`
  - Notes: Custom entity fields are flat — no associations. Text fields (ProcessTypeDescription, ProcessStatusText) are denormalized by the query class. `@ObjectModel.text.element` is placed directly in the custom entity CDS (SAP docs confirm: MDE=`-`, meaning this annotation is NOT allowed in DDLX — it must be in the CDS definition itself). The DDLX places `@UI.textArrangement` on the same fields. StatusCriticality is computed by the query class. Field types use DDIC data elements per Constitution Principle I (DDIC-First). Exceptions: `StatusCriticality : abap.int1` — no DDIC data element exists for RAP criticality integers; built-in type justified.
  - **Cross-package DDIC dependency**: Fields `zfi_process_created_by`, `zfi_process_bus_status`, `zfi_process_description`, `zfi_process_started_at`, `zfi_process_ended_at` are defined in the planner repo (`ZFI_PROCESS` package). These exist in the same SAP system and are accessible cross-package. No transport dependency issue — they are already active in the target system.
  - **New DDIC data element**: `ZFI_PROCESS_STATUS_DESC` — create in ovysledovka repo. Purpose: status description text field (domain DESCR = CHAR80). Used for `ProcessStatusText` to avoid semantic mismatch with `zfi_process_description` (which is for process instance descriptions, not status text). Short description: "Process Status Description".

- [x] **Task 4: Create query provider class (`ZCL_FI_ALLOC_DASH_QUERY`)**
  - File: `ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_query.clas.abap` (NEW)
  - File: `ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_query.clas.xml` (NEW)
  - Reference: `planner/src/zcl_fiproc_health_chk_query.clas.abap` (health check query — `if_rap_query_provider~select` method pattern)
  - Action: Create query class implementing `IF_RAP_QUERY_PROVIDER`
  - Class definition:
    ```abap
    CLASS zcl_fi_alloc_dash_query DEFINITION
      PUBLIC FINAL
      CREATE PUBLIC.
      PUBLIC SECTION.
        INTERFACES if_rap_query_provider.
    ENDCLASS.
    ```
  - `if_rap_query_provider~select` method logic:
    1. Call `io_request->get_filter( )->get_as_sql_string( )` to get filter as SQL WHERE string (e.g., `"COMPANYCODE = '1000' AND FISCALYEAR = '2024' AND ALLOCATIONID = 1"`)
    2. Call `io_request->get_paging( )` for page size/offset
    3. Call `io_request->get_sort_elements( )` for sort order
    4. **Guard: `IF io_request->is_data_requested( )`** — only execute the SELECT and set_data when the framework actually requests data rows (the framework may call `select` only for the total count):
       a. **Single ABAP SQL SELECT** from `ZFI_I_ALLOC_DASHBOARD` (I_ enrichment view) with LEFT OUTER JOINs:
          - `ZFI_I_ALLOC_DASHBOARD` aliased as `dash` provides: CompanyCode, FiscalYear, FiscalPeriod, AllocationId (COALESCEd), ProcessInstanceId, Active, Locked, Description
          - LEFT OUTER JOIN to `zfi_proc_inst` aliased as `inst` ON `dash~ProcessInstanceId = inst~instance_id` — provides: process_type, status, started_at, ended_at, created_by, business_status_1, business_status_2
          - LEFT OUTER JOIN to `zfi_proc_type` aliased as `ptype` ON `inst~process_type = ptype~process_type` — provides: description (as ProcessTypeDescription)
       b. Apply filter SQL string from step 1 as `WHERE (lv_sql_filter)` — dynamic WHERE clause injection. The `get_as_sql_string()` method returns a string compatible with `WHERE ( )` syntax. CompanyCode, FiscalYear, AllocationId are mandatory (UI enforces via `@Consumption.filter.mandatory`). FiscalPeriod and ProcessStatus are optional filters.
       c. Apply sort order from step 3 — map `get_sort_elements()` output to ORDER BY clause. Default sort: CompanyCode ASC, FiscalYear ASC, FiscalPeriod ASC, AllocationId ASC
       d. Handle paging — respect `get_page_size()` and `get_offset()`
       e. Compute `StatusCriticality` based on ProcessStatus mapping (see domain fixed values table above)
       f. Compute `ProcessStatusText` via CASE statement mapping domain fixed values. **Must include ELSE '' (empty string)** as fallback for unknown/future status values.
       g. Set data via `io_response->set_data( lt_result )`
    5. **Guard: `IF io_request->is_total_numb_of_rec_requested( )`** — set total count via `io_response->set_total_number_of_records( )`. This may require a separate COUNT(*) query or reuse the result set count from step 4.
  - `.clas.xml` content:
    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <abapGit version="v1.0.0" serializer="LCL_OBJECT_CLAS" serializer_version="v1.0.0">
     <asx:abap xmlns:asx="http://www.sap.com/abapxml" version="1.0">
      <asx:values>
       <VSEOCLASS>
        <CLSNAME>ZCL_FI_ALLOC_DASH_QUERY</CLSNAME>
        <LANGU>E</LANGU>
        <DESCRIPT>Allocation Dashboard Query Provider</DESCRIPT>
        <STATE>1</STATE>
        <CLSCCINCL>X</CLSCCINCL>
        <FIXPT>X</FIXPT>
        <UNICODE>X</UNICODE>
       </VSEOCLASS>
      </asx:values>
     </asx:abap>
    </abapGit>
    ```
  - Notes:
    - The query class does ALL data retrieval — the custom entity CDS is just a type definition
    - **SELECT architecture**: Single ABAP SQL statement with LEFT OUTER JOINs (not sequential SELECTs). The I_ enrichment view is the anchor, joined to process instance and process type tables. All JOINs use explicit alias qualification (`dash~`, `inst~`, `ptype~`) to avoid ambiguous field references.
    - **Filter handling**: Use `get_as_sql_string()` from `IF_RAP_QUERY_FILTER` — returns a SQL string usable directly in `WHERE (lv_sql_filter)`. This is much simpler than manually converting range tables to WHERE conditions. The method handles all filter operators (EQ, BT, CP, etc.) and multi-value filters. SAP documentation confirms this pattern for unmanaged query implementations. `get_as_ranges()` and `get_as_tree()` are also available but `get_as_sql_string()` is preferred for push-down to the CDS view.
    - **CRITICAL — SELECT alias naming**: The `get_as_sql_string()` output references field names as they appear in the custom entity definition (e.g., `COMPANYCODE`, `FISCALYEAR`, `ALLOCATIONID`). Therefore, all SELECT aliases in the query class MUST exactly match the custom entity field names. For example: `dash~CompanyCode AS CompanyCode`, `inst~status AS ProcessStatus`, `coalesce( dash~AllocationId, 0 ) AS AllocationId`. If aliases differ from entity field names, the dynamic `WHERE (lv_sql_filter)` will fail at runtime with a column-not-found error.
    - **Sort implementation**: Map `get_sort_elements()` to dynamic ORDER BY. If no sort requested, use default: CompanyCode ASC, FiscalYear ASC, FiscalPeriod ASC, AllocationId ASC. Implementation approach: `get_sort_elements()` returns a table of `IF_RAP_QUERY_SORT~TY_SORT_ELEMENT` (fields: `element_name`, `descending`). Build an `ORDER BY` string by looping over the table, appending each element name + ASC/DESC. Use the string in the SELECT with `ORDER BY (lv_order_by)` dynamic clause — same dynamic SQL pattern as the filter. Field names in `element_name` match the custom entity field names.
    - StatusCriticality mapping: COMPLETED=3, FAILED=1, CANCELLED=1, RUNNING=2, PENDING=2, EXECREQ=2, RESTREQ=2, NEW=0, SKIPPED=0, SUPERSEDED=0, initial('')=0
    - ProcessStatusText: Use CASE statement in ABAP (simpler than joining to VH view for domain fixed values). **Must include ELSE '' (empty string)** as fallback for unknown/future status values to avoid runtime issues.
    - **Local `ty_row` structure**: The query class uses a local TYPE definition for the result row (same pattern as reference `ZCL_FIPROC_HEALTH_CHK_QUERY`). This is a documented pragmatic exception to Constitution Principle I (DDIC-First) — the structure is purely internal to the query class, never exposed externally, and creating a DDIC structure for a single-use internal type adds no value. See Constitution Compliance section.
    - **Error handling**: Catch `CX_RAP_QUERY_FILTER_NO_RANGE` and other RAP query exceptions. For unexpected errors, let them propagate (RAP framework handles). No `ZCX_FI_PROCESS_ERROR` — this is read-only, no business logic errors.
    - **Consult SAP docs** for `IF_RAP_QUERY_FILTER` (especially `get_as_sql_string()`), `IF_RAP_QUERY_PAGING`, `IF_RAP_QUERY_SORT` API methods before implementing (Constitution Principle III).

- [x] **Task 5: Create DDLX metadata extension (`ZFI_I_ALLOC_DASHBOARD_CE`)**
  - File: `ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddlx.asddlxs` (NEW)
  - Action: Create metadata extension annotating `ZFI_I_ALLOC_DASHBOARD_CE`
  - Required annotation: `@Metadata.layer: #CORE` (mandatory for DDLX activation)
  - Reference: `planner/src/zfiproc_c_instance_tp.ddlx.asddlxs` (DDLX pattern)
  - Header info:
    - `typeName: 'Allocation Run'`, `typeNamePlural: 'Allocation Runs'`
    - Title: `CompanyCode` + `FiscalYear` + `FiscalPeriod`
  - Facets (object page — pre-defined for future drill-down):
    - `#COLLECTION` "Allocation Run" (pos 10)
    - `#FIELDGROUP_REFERENCE` "General" (pos 10, parent: "Allocation Run")
    - `#FIELDGROUP_REFERENCE` "Process" (pos 20, parent: "Allocation Run")
  - List report columns (`@UI.lineItem`):
    - CompanyCode (pos 10, #HIGH)
    - FiscalYear (pos 20, #HIGH)
    - FiscalPeriod (pos 30, #HIGH)
    - AllocationId (pos 40, #HIGH)
    - ProcessStatus (pos 50, #HIGH, `criticality: 'StatusCriticality'`, `criticalityRepresentation: #WITH_ICON`)
    - BusinessStatus1 (pos 60, #HIGH)
    - BusinessStatus2 (pos 70, #HIGH)
    - StartedAt (pos 80, #MEDIUM)
    - EndedAt (pos 90, #MEDIUM)
    - ProcessInstanceId (pos 100, #LOW)
    - ProcessType (pos 110, #LOW)
    - CreatedBy (pos 120, #LOW)
    - Active (pos 130, #LOW)
    - Locked (pos 140, #LOW)
    - Description (pos 150, #LOW)
  - Hidden fields:
    - `@UI.hidden: true` on `StatusCriticality`
    - `@UI.hidden: true` on `ProcessTypeDescription` (displayed via text arrangement on ProcessType)
    - `@UI.hidden: true` on `ProcessStatusText` (displayed via text arrangement on ProcessStatus)
  - Text arrangements:
    - `@UI.textArrangement: #TEXT_LAST` on `ProcessType` (shows "ALLOC (Cost Allocation)")
    - `@UI.textArrangement: #TEXT_LAST` on `ProcessStatus` (shows "COMPLETED (Completed)")
  - Selection fields (`@UI.selectionField`):
    - CompanyCode (pos 10, **mandatory** — `@Consumption.filter.mandatory: true`)
    - FiscalYear (pos 20, **mandatory** — `@Consumption.filter.mandatory: true`)
    - FiscalPeriod (pos 30)
    - AllocationId (pos 40, **mandatory** — `@Consumption.filter.mandatory: true`)
    - ProcessStatus (pos 50)
  - Field groups for object page:
    - "General": CompanyCode, FiscalYear, FiscalPeriod, AllocationId, Active, Locked, Description
    - "Process": ProcessInstanceId, ProcessType, ProcessStatus, BusinessStatus1, BusinessStatus2, StartedAt, EndedAt, CreatedBy
  - Notes: All UI annotations live here, not in CDS custom entity. Follow planner DDLX importance pattern (#HIGH/#MEDIUM/#LOW). DDLX name matches entity name (`ZFI_I_ALLOC_DASHBOARD_CE`) — this is standard SAP practice. `@Consumption.filter.mandatory` is confirmed valid in DDLX (SAP docs: MDE=`X`). `@UI.textArrangement` works in DDLX with `@ObjectModel.text.element` placed in the CDS custom entity definition (Task 3) — SAP docs confirm this split placement pattern.

- [x] **Task 6: Create service definition (`ZFI_UI_ALLOC_DASHBOARD`)**
  - File: `ovysledovka/src/zfi_alloc_process/zfi_ui_alloc_dashboard.srvd.srvdsrv` (NEW)
  - Action: Create service definition exposing the custom entity
  - Content:
    ```
    @EndUserText.label: 'UI service for Allocation Dashboard'
    define service ZFI_UI_ALLOC_DASHBOARD {
      expose ZFI_I_ALLOC_DASHBOARD_CE;
    }
    ```
  - Reference: `planner/src/zfiproc_ui_health.srvd.srvdsrv` (health check service — same pattern)
  - Notes: Only one entity to expose. Simple single-expose service.

- [x] **Task 7: Create service binding in SAP (`ZFI_UI_ALLOC_DASHBOARD_O4`)**
  - Action: Create in SAP system (ADT / Eclipse), NOT in source code
    1. Open ADT, create new Service Binding
    2. Name: `ZFI_UI_ALLOC_DASHBOARD_O4`
    3. Binding type: OData V4 - UI
    4. Service definition: `ZFI_UI_ALLOC_DASHBOARD`
    5. Publish the service binding
    6. Pull from abapGit to get the `.srvb.xml` file
  - Notes: Service bindings are XML-serialized by abapGit — they cannot be created as source files. Must be created in the SAP system and then pulled.

- [x] **Task 8: Activate and test end-to-end**
  - Action: Activate all objects in dependency order:
    0. `ZFI_PROCESS_STATUS_DESC` (new DDIC data element — must exist before custom entity references it)
    1. `ZFI_I_ALLOC_DASH_1` (modified I_ base view)
    2. `ZFI_I_ALLOC_DASHBOARD` (modified I_ enrichment view — depends on DASH_1)
    3. `ZCL_FI_ALLOC_DASH_QUERY` (new query class — must exist before custom entity references it)
    4. `ZFI_I_ALLOC_DASHBOARD_CE` (new custom entity — references query class via annotation and DDIC element from step 0)
    5. `ZFI_I_ALLOC_DASHBOARD_CE` metadata extension (new DDLX — annotates custom entity, must activate after entity)
    6. `ZFI_UI_ALLOC_DASHBOARD` (new service definition — exposes custom entity)
    7. `ZFI_UI_ALLOC_DASHBOARD_O4` (new service binding — publish after service definition)
  - Note: Activation order is strict. If step 4 is activated before step 3, the custom entity will fail with "class ZCL_FI_ALLOC_DASH_QUERY not found" error. **abapGit note**: When pushing via abapGit, dependency resolution handles activation order automatically. The explicit ordering above is only relevant when activating objects manually in ADT one by one.
  - Action: Test via Fiori Elements preview
    - Open service binding `ZFI_UI_ALLOC_DASHBOARD_O4` in ADT
    - Click "Preview" on the entity set
    - Verify list report renders with all columns
    - Test filters: company code, fiscal year, fiscal period, allocation ID, process status
    - Verify rows with allocation data show: process status (with criticality icon), business statuses, timestamps
    - Verify rows without allocation data appear with AllocationId=0 and empty process fields
    - Verify criticality coloring: COMPLETED=green icon, FAILED=red icon, RUNNING=yellow icon
    - Verify text arrangement: ProcessStatus shows "COMPLETED (Completed)", ProcessType shows "ALLOC (Cost Allocation)"
  - Action: Pull to abapGit after successful test
    - Stage all new and modified objects
    - Commit: `EST-132: Allocation process dashboard MVP`

### Acceptance Criteria

- [ ] **AC 1:** Given fiscal periods exist for company code 1000 in year 2024 (all periods defined for the company code's fiscal year variant), when the dashboard is loaded with filters CompanyCode=1000 and FiscalYear=2024, then all fiscal periods appear as rows in the list report. Periods without allocation data show AllocationId=0. Periods with N allocation runs show N rows (one per AllocationId) — there is NO extra AllocationId=0 row for periods that already have allocations.

- [ ] **AC 2:** Given an allocation run exists in `ZFI_ALLOC_STATE` for CompanyCode=1000, FiscalYear=2024, FiscalPeriod=003, AllocationId=1 with a valid `ProcessInstanceId`, when the dashboard is loaded, then the row for period 003 / AllocationId=1 shows the process status (with criticality icon), business status 1/2, started at, and ended at values from the linked process instance.

- [ ] **AC 3:** Given no allocation run exists for CompanyCode=1000, FiscalYear=2024, FiscalPeriod=007, when the dashboard is loaded, then a row for period 007 appears with AllocationId=0 and empty values for ProcessInstanceId, ProcessStatus, BusinessStatus1/2, StartedAt, and EndedAt.

- [ ] **AC 4:** Given the dashboard is loaded, when the user enters CompanyCode=2000 in the filter bar, then only rows for company code 2000 are displayed.

- [ ] **AC 5:** Given the dashboard is loaded, when the user enters AllocationId=1 in the filter bar, then only rows with AllocationId=1 are displayed (rows with AllocationId=0 are excluded).

- [ ] **AC 6:** Given the dashboard is loaded, when the user views the ProcessStatus column, then status values show criticality icons: green checkmark for COMPLETED, red X for FAILED/CANCELLED, yellow warning for RUNNING/PENDING/EXECREQ/RESTREQ.

- [ ] **AC 7:** Given the service binding `ZFI_UI_ALLOC_DASHBOARD_O4` is published, when the OData V4 metadata document is requested, then it returns a valid `$metadata` document with all expected entity properties.

- [ ] **AC 8:** Given the DDLX metadata extension is active, when the Fiori Elements list report renders, then columns appear in the specified order (CompanyCode, FiscalYear, FiscalPeriod, AllocationId, ProcessStatus, BusinessStatus1, BusinessStatus2, StartedAt, EndedAt) with correct importance tiers (#HIGH columns always visible).

- [ ] **AC 9:** Given the dashboard is loaded with filters CompanyCode=1000, FiscalYear=2024, and AllocationId=0, then only rows with AllocationId=0 are displayed (periods with no allocation runs). Rows with AllocationId > 0 are excluded. This confirms the AllocationId=0 filter works correctly as an edge case.

## Additional Context

### Dependencies

- `ZFI_I_ALLOC_DASHBOARD` (I_ enrichment view, ovysledovka repo) must be active — query class reads from it
- `zfi_proc_inst` (planner repo) must be accessible — query class joins for process details
- `zfi_proc_type` (planner repo) must be accessible — query class joins for process type description
- `I_FiscalYearPeriodForCmpnyCode` (SAP standard) must be available in the system
- `ZFI_ALLOC_STATE` table must exist with `PROCESS_INSTANCE_ID` field

### Testing Strategy

- **No ABAP Unit tests for MVP** — manual testing via Fiori Elements preview
- **OData metadata verification**: Request `$metadata` via browser or ADT to confirm entity structure
- **List report rendering**: Use Fiori Elements preview in ADT service binding
- **Process data display**: Verify rows with valid `ProcessInstanceId` display process instance data
- **Empty period display**: Verify LEFT OUTER JOIN produces AllocationId=0 rows for all fiscal periods
- **Filter verification**: Test each selection field (CompanyCode, FiscalYear, FiscalPeriod, AllocationId, ProcessStatus)
- **Criticality verification**: Confirm status icons appear (green/red/yellow) based on process status value
- **Text arrangement verification**: Confirm ProcessType and ProcessStatus show text descriptions alongside values
- **Cross-repo data access**: Confirm query class can SELECT from `zfi_proc_inst` and `zfi_proc_type` (planner repo tables)

### Adversarial Review Resolution Log

#### Round 1 (21 findings — architecture switch to custom entity)

| Finding | Severity | Resolution |
|---------|----------|------------|
| F1: persistent table mismatch | Critical | Resolved: switched to custom entity (no persistent table) |
| F2: NULL key AllocationId | Critical | Resolved: COALESCE(allocation_id, 0) in I_ base view |
| F3: BDEF field(readonly) for path fields | Critical | Resolved: no BDEF with custom entity |
| F4: .clas.xml missing from file list | High | Fixed: added to files_to_modify and Task 4 |
| F5: AC 5 ambiguous NULL filtering | High | Fixed: AllocationId=0 (not NULL), AC 5 rewritten — filter excludes AllocationId=0 rows |
| F6: @Semantics.dateTime wrong annotation | High | Fixed: removed; timestamps are plain TIMESTAMPL fields in custom entity |
| F7: Multi-hop path expressions unverified | High | Resolved: no path expressions in custom entity; query class does joins; associations verified (StatusText, Description exist) |
| F8: Constitution compliance missing | High | Fixed: added Constitution Compliance section |
| F9: _ProcessStatusVH alias non-standard | Medium | Resolved: no R_ layer; no association aliases needed with custom entity |
| F10: No authorization concept | Medium | Documented: conscious MVP decision (AD-10), noted in Out of Scope |
| F11: Activation order wrong | Medium | Fixed: rewritten for custom entity (class before entity) |
| F12: ZFIPROC_I_InstanceStatus_VH not in reference | Medium | Fixed: added to Files to Reference with field details |
| F13: Multiple AllocationIds per period | Medium | Fixed: AC 1 reworded to describe actual row count behavior |
| F14: @Semantics.user.* on CreatedBy | Medium | N/A: custom entity fields don't support @Semantics annotations; CreatedBy is plain display field |
| F15: Criticality 'if feasible' non-decision | Low | Fixed: concrete criticality mapping defined (AD-9), StatusCriticality field added |
| F16: FiscalPeriod missing from selection fields | Low | Fixed: added FiscalPeriod (pos 30) to selection fields |
| F17: WHERE clause wording inconsistency | Low | Fixed: consistently says "remove" |
| F18: Object page facets YAGNI | Low | Kept: low cost, useful for future drill-down |
| F19: Transport request not mentioned | Low | Out of scope for tech spec — standard SAP development process |
| F20: AD-3 raw DATE+TIME not implemented | Low | Fixed: AD-3 updated — timestamps are TIMESTAMPL from process instance, no separate raw fields |
| F21: DATS_TIMS_TO_TSTMP in tech_stack | Low | Fixed: removed from tech_stack |

#### Round 2 (21 findings — DDIC types, mandatory filters, row behavior, DDLX fixes)

| Finding | Severity | Resolution |
|---------|----------|------------|
| R2-F1: Custom entity field types wrong (Active, Locked, Description, CreatedBy, BusinessStatus1/2, ProcessStatusText) | Critical | Fixed: Active→zfi_alloc_id_active, Locked→zfi_alloc_id_locked, Description→zfi_alloc_description (CHAR80), CreatedBy→zfi_process_created_by, BusinessStatus1/2→zfi_process_bus_status, ProcessStatusText→zfi_process_description (CHAR80, DDIC-First compliant) |
| R2-F2: Task 1 missing Active/Locked/Description projection | Critical | Fixed: added state.active, state.locked, state.description to Task 1 (DASH_1) so Task 2 can propagate them |
| R2-F3: @UI.textArrangement may not work on custom entities | Critical | Documented: user chose to try it; added SAP validation note with fallback plan (show text columns visibly if it fails) |
| R2-F4: AC 1 says "12 periods" but special periods may exist | High | Fixed: reworded to "all fiscal periods defined for the company code's fiscal year variant" |
| R2-F5: Query class SELECT architecture unclear | High | Fixed: clarified single ABAP SQL with LEFT OUTER JOINs from I_ view as anchor |
| R2-F6: Custom entity missing @AccessControl annotation | High | Fixed: added @AccessControl.authorizationCheck: #NOT_REQUIRED to Task 3 annotations |
| R2-F7: DDLX missing @Metadata.layer: #CORE | High | Fixed: added to Task 5 as required annotation |
| R2-F8: Row count description wrong at line 109 | High | Fixed: corrected to "N rows for periods with data, 1 row for empty periods, NO extra AllocationId=0 row" |
| R2-F9: Cross-package DDIC dependency unstated | Medium | Fixed: added cross-package dependency note to Task 3 (planner repo data elements) |
| R2-F10: ProcessStatusText : char40 violates DDIC-First | Medium | Fixed: changed to zfi_process_description (CHAR80 DDIC type) with justification |
| R2-F11: Health check reference doesn't handle filtering | Medium | Fixed: added note to consult SAP docs for filter handling pattern |
| R2-F12: Sort only consumed, not implemented | Medium | Fixed: added sort implementation to query class (map get_sort_elements to ORDER BY, default sort specified) |
| R2-F13: No mandatory filters | Medium | Fixed: CompanyCode, FiscalYear, AllocationId marked mandatory via @Consumption.filter.mandatory |
| R2-F14: Object page title missing FiscalYear | Medium | Fixed: title now CompanyCode + FiscalYear + FiscalPeriod |
| R2-F15: DDLX name same as entity name | Low | Clarified: standard SAP practice, noted in Task 5 |
| R2-F16: Table structure shows wrong DDIC types for Active/Locked/Description | Low | Fixed: corrected to ZFI_ALLOC_ID_ACTIVE, ZFI_ALLOC_ID_LOCKED, ZFI_ALLOC_DESCRIPTION |
| R2-F17: Activation order needs explicit dependency notes | Low | Fixed: added dependency explanations to each activation step |
| R2-F18: Error handling underspecified in query class | Low | Fixed: added CX_RAP_QUERY_FILTER_NO_RANGE catch, propagation behavior documented |
| R2-F19: Constitution compliance wording too soft | Low | Fixed: changed "Applies" to "Fully applies" where warranted, added specific DDIC element list |
| R2-F20: IF_RAP_QUERY_SORT not in SAP docs consultation list | Low | Fixed: added to Constitution Principle III notes |
| R2-F21: StatusCriticality abap.int1 exception not justified | Low | Fixed: added explicit justification (no DDIC data element exists for RAP criticality integers) |

#### Round 3 (17 findings — DDIC timestamps, status description type, text element placement, filter pattern, value helps)

| Finding | Severity | Resolution |
|---------|----------|------------|
| R3-F1: `StartedAt`/`EndedAt` typed as `timestampl` instead of DDIC data elements | Critical | Fixed: changed to `zfi_process_started_at` / `zfi_process_ended_at` (domain ZFI_PROCESS_TIMESTAMP = DEC 21,7) |
| R3-F2: `@Consumption.filter.mandatory` placed in DDLX may not be supported | Critical | Noise: SAP docs confirm MDE=`X` — annotation IS allowed in DDLX |
| R3-F3: DDLX on custom entity diverges from reference | Critical | Noise: SAP docs confirm DDLX works on custom entities (any entity except CDS table functions) |
| R3-F4: `ProcessStatusText : zfi_process_description` semantic mismatch | High | Fixed: new DDIC data element `ZFI_PROCESS_STATUS_DESC` (domain DESCR = CHAR80) created for status text fields |
| R3-F5: Query class local `ty_row` violates Constitution DDIC-First | High | Pragmatic exception: documented in Constitution Compliance section. Reference pattern does the same. Internal-only type, no external exposure. |
| R3-F6: `@UI.textArrangement` without `@ObjectModel.text.element` won't work | High | Fixed: added `@ObjectModel.text.element` to custom entity CDS (Task 3) for ProcessStatus→ProcessStatusText and ProcessType→ProcessTypeDescription. SAP docs confirm: MDE=`-` for this annotation (must be in CDS, not DDLX). `@UI.textArrangement` stays in DDLX. Removed "SAP Validation Required" note from Task 5. |
| R3-F7: Performance — unbounded calendar scan | High | Resolved: `get_as_sql_string()` enables WHERE push-down to CDS view; mandatory filters ensure bounded query |
| R3-F8: No `@Consumption.valueHelpDefinition` for filter fields | Medium | Fixed: added VH for CompanyCode→I_CompanyCode, FiscalYear→I_FiscalYear, ProcessStatus→ZFIPROC_I_InstanceStatus_VH |
| R3-F9: Activation order ambiguity — ADT vs abapGit | Medium | Clarified: abapGit handles dependency resolution automatically; explicit order only for manual ADT activation |
| R3-F10: Ambiguous JOIN ON `process_type = process_type` | Medium | Fixed: all JOINs use explicit alias qualification (`dash~`, `inst~`, `ptype~`) |
| R3-F11: No AC for AllocationId=0 filter edge case | Medium | Fixed: added AC 9 for AllocationId=0 filter behavior |
| R3-F12: CASE for ProcessStatusText has no ELSE fallback | Medium | Fixed: added ELSE '' (empty string) to CASE statement |
| R3-F13: Filter implementation guidance missing | Medium | Resolved: documented `get_as_sql_string()` pattern in Task 4 with SAP documentation reference |
| R3-F14: "001-012" in table structure but special periods 013-016 exist | Low | Fixed: updated description to "001-012, special periods 013-016 if applicable" |
| R3-F15: `@AbapCatalog.viewEnhancementCategory` not discussed | Low | N/A: irrelevant for custom entities (only applies to CDS views) |
| R3-F16: Mandatory AllocationId filter conflicts with "see all periods" use case | Low | Kept mandatory: AllocationId=0 rows always visible when filtered; users filter for specific allocation ID |
| R3-F17: `@Metadata.ignorePropagatedAnnotations` not discussed | Low | Noise: irrelevant for custom entities (no annotation propagation chain) |

### Notes

- Prototype CDS views (`ZFI_I_ALLOC_DASH_1`, `ZFI_I_ALLOC_DASHBOARD`) already created in SAP and pulled to repo
- AllocationId currently commented out in DASH_1 — Task 1 restores it with COALESCE
- WHERE clause in DASH_1 is hardcoded for testing — Task 1 removes it
- This is the first custom entity / SRVD / SRVB in the ovysledovka repo — establishes the pattern
- Service binding must be created in SAP (not source-editable), then pulled via abapGit
- Future enhancements (out of scope): drill-down to process instance, actions (rerun/cancel), phase detail columns, authorization checks

## Review Notes

- **Adversarial review completed** (bmad-code-review skill)
- **Findings:** 21 total — 9 fixed, 3 noise, 3 deferred (verify in system), 6 acknowledged/user-decided
- **Resolution approach:** Walk-through (each finding discussed individually)
- **Key architectural change during review:** Query class rewritten to SELECT from CDS view `ZFI_I_ALLOC_DASHBOARD` instead of re-joining raw tables. Eliminated field name mismatch between custom entity elements and SQL columns. Type descriptions resolved via bulk ABAP lookup (`resolve_type_descriptions` method).
- **Deferred items (verify during manual testing in SAP ADT):**
  - F10: `DESCR` domain length — verify `ZFI_PROCESS_STATUS_DESC` DTEL works with CHAR80
  - F11: CDS field name casing — verify `get_as_sql_string()` filter/sort field names resolve correctly
  - F17: `@Consumption.filter.mandatory` on custom entity DDLX — verify annotation is respected
  - F21: `@UI.textArrangement` may not work on custom entities — test at runtime
- **Implementation status:** All 8 tasks complete (code artifacts created). Tasks 7-8 require manual SAP ADT steps (create service binding `ZFI_UI_ALLOC_DASHBOARD_O4`, activate & test via Fiori Elements preview).
