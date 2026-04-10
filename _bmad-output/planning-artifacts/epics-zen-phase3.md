---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - '_bmad-output/planning-artifacts/architecture.md'
  - '_bmad-output/planning-artifacts/epics-zen-phase2.md'
  - '_bmad-output/planning-artifacts/epics.md'
  - '_bmad-output/brainstorming/brainstorming-session-2026-04-01-100000.md'
workflowType: 'epics-and-stories'
project_name: 'ZEN_ORCH — Process Orchestration Engine — Phase 3'
user_name: 'Zdenek'
date: '2026-04-10'
lastStep: 4
status: 'complete'
target_repository: 'cz.en.orch (Epic 8), cz.imcg.fast.ovysledovka (Epic 9)'
---

# ZEN_ORCH Phase 3 — Dashboard & Planner Adapter

## Overview

Phase 1 delivered the core orchestration engine (score/performance model, adapter framework, APJ sweep, breakpoints, unit tests). Phase 2 added the Business State Registry (BSR) with collision detection, prerequisite gates, and full unit test coverage. All Phase 2 unit tests are GREEN.

Phase 3 wires the orchestrator to the real world:

1. **Epic 8 — Three-Level Observability Dashboard** (FR6): CDS views + RAP BDEF providing a read/action UI for all performances, drill-down to step detail, and adapter deep links. Lives in `cz.en.orch`.
2. **Epic 9 — Planner Adapter** (integration): `ZCL_FI_ALLOC_ORCH_ADAPTER` bridges ZEN_ORCH to the `ZFI_PROCESS` framework. Lives in `cz.imcg.fast.ovysledovka`. This is the first real adapter replacing the mock.

**Target Repositories:**
- `cz.en.orch` (`/Users/smolik/DEV/cz.en.orch`) — Epic 8
- `cz.imcg.fast.ovysledovka` (`/Users/smolik/DEV/cz.imcg.fast.ovysledovka`) — Epic 9

**Package:** `ZEN_ORCH` (Epic 8) / `ZFI_ALLOC_PROCESS` (Epic 9)
**Phase Scope:** Epics 8 and 9. Step-level BSR collision (brainstorming #26) deferred to Phase 4.

---

## Phase 3 Requirements

### Functional Requirements Addressed

| FR | Description | Status |
|----|-------------|--------|
| FR6 | Three-level observability dashboard | **EPIC 8 — THIS PHASE** |
| Integration | Real planner adapter (`ZCL_FI_ALLOC_ORCH_ADAPTER`) | **EPIC 9 — THIS PHASE** |
| Deferred | Step-level BSR collision locking (brainstorming #26) | Phase 4 |

### Key Design Decisions (Phase 3)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Dashboard persistence | Derived from live `ZEN_ORCH_PERF` + `ZEN_ORCH_P_STEP` + `ZEN_ORCH_BSR` — no separate state | Single source of truth; no staleness risk |
| Timing fields | Add `STARTED_AT TIMESTAMPL` and `ENDED_AT TIMESTAMPL` to both `ZEN_ORCH_PERF` and `ZEN_ORCH_P_STEP` | Required for duration display; engine sets them at status transitions |
| CDS layer | Interface view `ZEN_ORCH_I_PERF` on `ZEN_ORCH_PERF`; `ZEN_ORCH_I_P_STEP` on `ZEN_ORCH_P_STEP`; consumption view `ZEN_ORCH_C_PERF` for RAP | Standard CDS two-layer pattern |
| RAP flavor | Unmanaged BDEF on `ZEN_ORCH_C_PERF` — engine owns all writes; RAP only provides action routing | Engine write logic already complete; RAP wraps read + actions |
| Dashboard actions | Cancel, Restart, Resume wired as RAP actions delegating to engine | Reuse existing engine public methods |
| Adapter handle format | `"ZFI_PROCESS:<proc_type>:<instance_id>"` | Opaque string; adapter encodes, adapter decodes |
| Adapter start() returns | RUNNING (async — bgRFC) or COMPLETED (sync) | Planner processes are async-by-default via APJ/bgRFC |
| Adapter repo | `ZFI_ALLOC_PROCESS` package in `ovysledovka` | One-way dependency: `ovysledovka` → `cz.en.orch` |
| Adapter params JSON | `{"process_type":"<type>","company_code":"<CC>","fiscal_year":"<FY>","fiscal_period":"<FP>","alloc_id":"<ID>","force_run":"<X>"}` | Matches `zif_fi_alloc_plan_step` parameter set |

---

## Pre-Condition: Timing Fields (DDIC Enhancement)

Before the dashboard CDS can show duration, two tables need timing fields. This is the first task in Epic 8 and is a prerequisite for all CDS stories.

**`ZEN_ORCH_PERF` additions:**
- `STARTED_AT TYPE timestampl` — set by engine when first non-PENDING step starts
- `ENDED_AT TYPE timestampl` — set by engine when performance reaches terminal status

**`ZEN_ORCH_P_STEP` additions:**
- `STARTED_AT TYPE timestampl` — set by engine when step is dispatched (adapter `start()` called)
- `ENDED_AT TYPE timestampl` — set by engine when step reaches terminal status

**Engine changes (minimal):** Set `STARTED_AT`/`ENDED_AT` at the appropriate status-write points inside `advance_performance`. No new public methods.

---

## Epic 8: Three-Level Observability Dashboard

The orchestrator has a Fiori Elements list report showing all performances with live status, drill-down to step detail with per-step timing and adapter deep links, and action buttons for Cancel, Restart, and Resume.

**Target repository:** `cz.en.orch`
**Constitution principles:** I (DDIC-First), II (SAP Standards), IV (Factory Pattern), V (Error Handling)
**Depends on:** Phase 1 (Epics 1–6, done), Phase 2 (Epic 7, done)

### Dependency Flow

```
E8.1 (Timing DDIC + Engine wiring)
 └──▶ E8.2 (Interface CDS Views)
       └──▶ E8.3 (Consumption CDS View)
             └──▶ E8.4 (RAP BDEF + Actions)
                   └──▶ E8.5 (UI Service Binding + Smoke Test)
```

---

### Story 8.1: Timing DDIC Fields and Engine Wiring

As a developer,
I want `STARTED_AT` and `ENDED_AT` timestamp fields on `ZEN_ORCH_PERF` and `ZEN_ORCH_P_STEP`,
So that the dashboard can display performance and step durations.

**target_repository:** `cz.en.orch`
**depends_on:** []
**constitution_principles:**
  - Principle I — DDIC-First
  - Principle II — SAP Standards

**Acceptance Criteria:**

**Given** tables `ZEN_ORCH_PERF` and `ZEN_ORCH_P_STEP` exist
**When** Story 8.1 is implemented and activated
**Then** `ZEN_ORCH_PERF` has two new fields: `STARTED_AT TYPE timestampl` and `ENDED_AT TYPE timestampl`
**And** `ZEN_ORCH_P_STEP` has two new fields: `STARTED_AT TYPE timestampl` and `ENDED_AT TYPE timestampl`
**And** a new data element `ZEN_ORCH_DE_TIMESTAMP` (TYPE `timestampl`, field label "Timestamp") exists and is used by all four fields

**Given** `ZCL_EN_ORCH_ENGINE` processes a performance
**When** the first STEP-type element transitions from PENDING → RUNNING (adapter `start()` succeeds)
**Then** `ZEN_ORCH_PERF-STARTED_AT` is set to `utclong_current( )` if not already set (first dispatch only)

**Given** a performance step is dispatched
**When** `dispatch_step` sets the step STATUS to RUNNING
**Then** `ZEN_ORCH_P_STEP-STARTED_AT` is set to `utclong_current( )`

**Given** a performance step reaches a terminal status (C/F/X)
**When** the step status is written
**Then** `ZEN_ORCH_P_STEP-ENDED_AT` is set to `utclong_current( )`

**Given** a performance reaches a terminal status (C/F/X)
**When** the performance STATUS is written as terminal
**Then** `ZEN_ORCH_PERF-ENDED_AT` is set to `utclong_current( )`

**And** all existing unit tests remain GREEN after the change
**And** all objects activate without errors

**Notes:**
- Use `utclong_current( )` for timestamp capture (ABAP 7.52+, available on 7.58)
- `STARTED_AT` on `ZEN_ORCH_PERF` is set only once — guard with `IF ls_perf-started_at IS INITIAL`
- No new exception textids needed; no public API changes

---

### Story 8.2: Interface CDS Views for Performance and Step

As a developer,
I want interface CDS views over `ZEN_ORCH_PERF` and `ZEN_ORCH_P_STEP`,
So that the consumption view and RAP BDEF have a clean, well-documented data layer.

**target_repository:** `cz.en.orch`
**depends_on:** ["8.1"]
**constitution_principles:**
  - Principle I — DDIC-First
  - Principle II — SAP Standards

**Acceptance Criteria:**

**Given** `ZEN_ORCH_PERF` with timing fields from Story 8.1 exists
**When** `ZEN_ORCH_I_PERF` is activated
**Then** it is a basic interface view (`define view entity`) on `ZEN_ORCH_PERF`
**And** it exposes all fields: `PerfUuid`, `ScoreId`, `Status`, `ParamsJson`, `ParamsHash`, `CreatedBy`, `CreatedAt`, `ChangedBy`, `ChangedAt`, `StartedAt`, `EndedAt`
**And** it adds a calculated field `DurationSec` (integer seconds from `StartedAt` to `EndedAt`, NULL if not ended) using `CASE WHEN EndedAt IS NOT NULL`
**And** it adds a semantic annotation `@Semantics.systemDateTime.createdAt: true` on `CreatedAt`
**And** a status criticality helper `StatusCriticality` (abap.int1): 3=green(C), 1=red(F/X), 2=orange(R), 0=grey(P/B)

**Given** `ZEN_ORCH_P_STEP` with timing fields from Story 8.1 exists
**When** `ZEN_ORCH_I_P_STEP` is activated
**Then** it is a basic interface view on `ZEN_ORCH_P_STEP`
**And** it exposes all fields: `PerfUuid`, `ScoreSeq`, `LoopIteration`, `ElemType`, `RefId`, `AdapterType`, `ParamsJson`, `Status`, `WorkUnitHandle`, `IsBreakpoint`, `StartedAt`, `EndedAt`
**And** it adds `DurationSec` (same pattern as performance view)
**And** it adds `StatusCriticality` (same mapping)

**And** both views activate without errors
**And** both views carry `@EndUserText.label` on each field

---

### Story 8.3: Consumption CDS View and Association

As a developer,
I want a consumption CDS view `ZEN_ORCH_C_PERF` composing performance and its steps,
So that the RAP BDEF has a single root entity with a to-many composition to steps.

**target_repository:** `cz.en.orch`
**depends_on:** ["8.2"]
**constitution_principles:**
  - Principle I — DDIC-First
  - Principle II — SAP Standards

**Acceptance Criteria:**

**Given** `ZEN_ORCH_I_PERF` and `ZEN_ORCH_I_P_STEP` exist
**When** `ZEN_ORCH_C_PERF` is activated
**Then** it is a consumption view (`define root view entity`) on `ZEN_ORCH_I_PERF`
**And** it declares a composition `_Steps` to `ZEN_ORCH_C_P_STEP` (on `PerfUuid = PerfUuid`)
**And** it carries `@UI.headerInfo` with `typeName: 'Performance'` and title field `ScoreId`
**And** it carries `@UI.lineItem` annotations for: `PerfUuid` (hidden), `ScoreId`, `Status` (with criticality), `CreatedAt`, `DurationSec`
**And** it carries `@UI.selectionField` for `Status` and `ScoreId`
**And** it carries `@UI.facets` for a section "Step Details" referencing `_Steps`

**Given** `ZEN_ORCH_I_P_STEP` exists
**When** `ZEN_ORCH_C_P_STEP` is activated
**Then** it is a consumption view (`define view entity`) on `ZEN_ORCH_I_P_STEP`
**And** it declares association `_Perf` back to `ZEN_ORCH_C_PERF`
**And** it carries `@UI.lineItem` for: `ScoreSeq`, `ElemType`, `AdapterType`, `Status` (with criticality), `WorkUnitHandle`, `DurationSec`
**And** `WorkUnitHandle` carries a `@UI.fieldGroup` entry with label "Adapter Handle" and a URL data point annotation `@Semantics.url: true` (for the deep link pattern — see Story 8.4)

**And** both views activate without errors

---

### Story 8.4: RAP BDEF with Cancel, Restart, Resume Actions

As an operator,
I want a RAP behavior definition with action buttons for Cancel, Restart, and Resume on performances,
So that I can control execution directly from the Fiori Elements list report.

**target_repository:** `cz.en.orch`
**depends_on:** ["8.3"]
**constitution_principles:**
  - Principle II — SAP Standards
  - Principle V — Error Handling

**Acceptance Criteria:**

**Given** `ZEN_ORCH_C_PERF` exists
**When** the RAP BDEF `ZEN_ORCH_I_PERF` is activated (unmanaged, read-only for data)
**Then** it declares three instance actions on the root entity:
  - `cancelPerformance` — enabled when `Status IN ('P','R','B')` — delegates to `ZCL_EN_ORCH_ENGINE=>cancel_performance`
  - `restartPerformance` — enabled when `Status = 'F'` — delegates to `ZCL_EN_ORCH_ENGINE=>restart_performance`
  - `resumePerformance` — enabled when `Status = 'B'` — delegates to `ZCL_EN_ORCH_ENGINE=>resume_performance`

**Given** `cancelPerformance` action is triggered
**When** the behavior pool method `cancel_performance_action` executes
**Then** it calls `ZCL_EN_ORCH_ENGINE=>get_instance( )->cancel_performance( iv_perf_uuid = <key> )`
**And** on `ZCX_EN_ORCH_ERROR`, it appends a failed message to `reported` and does not raise further

**Given** `restartPerformance` action is triggered
**When** the behavior pool method executes
**Then** it calls `ZCL_EN_ORCH_ENGINE=>get_instance( )->restart_performance( iv_perf_uuid = <key> )`
**And** error handling follows the same pattern as cancelPerformance

**Given** `resumePerformance` action is triggered
**When** the behavior pool method executes
**Then** it calls `ZCL_EN_ORCH_ENGINE=>get_instance( )->resume_performance( iv_perf_uuid = <key> )`
**And** error handling follows the same pattern

**And** the behavior pool class `ZCL_EN_ORCH_BP_PERF` (local behavior pool) implements all three action methods
**And** the BDEF specifies `implementation unmanaged` with `authorization master ( global )`
**And** all objects activate without errors

**Notes:**
- Use `@UI.identification` annotations in the consumption view to bind action buttons to the object page
- Behavior pool class name follows pattern `ZCL_EN_ORCH_BP_<entity>` — `ZCL_EN_ORCH_BP_PERF`
- No `COMMIT WORK` in action methods — engine methods own the LUW (they commit internally for lifecycle actions)
- Step-level adapter deep link: `get_detail_link()` is displayed via the `WorkUnitHandle` field as a URL on the step object page; no separate action needed

---

### Story 8.5: OData V4 Service Binding and Manual Smoke Test

As a developer,
I want the OData V4 service binding `ZEN_ORCH_UI_PERF_O4` published and manually tested,
So that the dashboard is accessible in the browser via `/sap/opu/odata4/...`.

**target_repository:** `cz.en.orch`
**depends_on:** ["8.4"]
**constitution_principles:**
  - Principle II — SAP Standards

**Acceptance Criteria:**

**Given** the RAP BDEF and consumption views from Stories 8.3/8.4 are active
**When** `ZEN_ORCH_UI_PERF_O4` (UI service binding, OData V4) is activated and published
**Then** the service is accessible in the browser without HTTP errors
**And** an empty list (no performances yet) renders without error
**And** a test performance created via `ZCL_EN_ORCH_ENGINE=>create_performance( )` appears in the list within seconds
**And** the Cancel action can be triggered on a PENDING performance from the UI (or via HTTP PATCH)
**And** navigating to a performance shows its step list with status and duration columns

**And** the service binding file is committed to `cz.en.orch` alongside the BDEF and behavior pool
**And** the manual smoke test result is recorded in a commit message or code comment (no dedicated test class required)

**Notes:**
- Service binding naming: `ZEN_ORCH_UI_PERF_O4` — consistent with existing `zen_orch_ui_health_o4` pattern in the repo
- Test data: use `ZCL_EN_ORCH_HEALTH_CHK_QUERY` to create a performance, or add a `create_test_performance` helper method
- This story has no automated acceptance test — manual browser/HTTP verification is sufficient

---

## New DDIC Objects — Epic 8

| Object | Type | Story | Notes |
|--------|------|-------|-------|
| `ZEN_ORCH_DE_TIMESTAMP` | Data element `TIMESTAMPL` | 8.1 | Shared by all 4 timing fields |
| `STARTED_AT`, `ENDED_AT` on `ZEN_ORCH_PERF` | Table fields | 8.1 | Table enhancement |
| `STARTED_AT`, `ENDED_AT` on `ZEN_ORCH_P_STEP` | Table fields | 8.1 | Table enhancement |
| `ZEN_ORCH_I_PERF` | Interface CDS view | 8.2 | On `ZEN_ORCH_PERF` |
| `ZEN_ORCH_I_P_STEP` | Interface CDS view | 8.2 | On `ZEN_ORCH_P_STEP` |
| `ZEN_ORCH_C_PERF` | Consumption CDS (root) | 8.3 | RAP root entity |
| `ZEN_ORCH_C_P_STEP` | Consumption CDS (child) | 8.3 | RAP child entity |
| `ZEN_ORCH_I_PERF` (BDEF) | Behavior definition | 8.4 | Unmanaged, 3 actions |
| `ZCL_EN_ORCH_BP_PERF` | Behavior pool class | 8.4 | Action delegation to engine |
| `ZEN_ORCH_UI_PERF_O4` | OData V4 service binding | 8.5 | Fiori Elements entry point |

---

## Activation Order — Epic 8

| Order | Object | Story |
|-------|--------|-------|
| 21 | Data element `ZEN_ORCH_DE_TIMESTAMP` | 8.1 |
| 22 | Table enhancements: `STARTED_AT`, `ENDED_AT` on `ZEN_ORCH_PERF` and `ZEN_ORCH_P_STEP` | 8.1 |
| 23 | Engine changes: set timing fields at dispatch/terminal transitions | 8.1 |
| 24 | Interface views `ZEN_ORCH_I_PERF`, `ZEN_ORCH_I_P_STEP` | 8.2 |
| 25 | Consumption views `ZEN_ORCH_C_PERF`, `ZEN_ORCH_C_P_STEP` | 8.3 |
| 26 | BDEF `ZEN_ORCH_I_PERF` (unmanaged) | 8.4 |
| 27 | Behavior pool class `ZCL_EN_ORCH_BP_PERF` | 8.4 |
| 28 | Service binding `ZEN_ORCH_UI_PERF_O4` | 8.5 |

---

## Epic 9: Planner Adapter (ZCL_FI_ALLOC_ORCH_ADAPTER)

The ZEN_ORCH engine can drive real ZFI_PROCESS allocation processes. `ZCL_FI_ALLOC_ORCH_ADAPTER` implements `ZIF_EN_ORCH_ADAPTER` and bridges to `ZCL_FI_PROCESS_MANAGER`. A score `ALLOC_YEAR_REFRESH` can be defined in `ZEN_ORCH_SCORE` and executed end-to-end through the orchestrator.

**Target repository:** `cz.imcg.fast.ovysledovka`
**Package:** `ZFI_ALLOC_PROCESS`
**Constitution principles:** I (DDIC-First), II (SAP Standards), IV (Factory Pattern), V (Error Handling)
**Depends on:** Epic 8 fully done; `ZIF_EN_ORCH_ADAPTER` from cz.en.orch Phase 1 (done)

### Dependency Flow

```
E9.1 (Adapter class — start + get_status)
 └──▶ E9.2 (cancel + restart + get_result + get_detail_link)
       └──▶ E9.3 (Adapter registration + smoke score)
             └──▶ E9.4 (Integration test)
```

---

### Story 9.1: ZCL_FI_ALLOC_ORCH_ADAPTER — start() and get_status()

As an engine,
I want `start()` and `get_status()` implemented in `ZCL_FI_ALLOC_ORCH_ADAPTER`,
So that the engine can create and poll real ZFI_PROCESS instances.

**target_repository:** `cz.imcg.fast.ovysledovka`
**depends_on:** []
**constitution_principles:**
  - Principle II — SAP Standards
  - Principle IV — Factory Pattern
  - Principle V — Error Handling

**Adapter Handle Format:** `"ZFI_PROCESS:<instance_id>"` — e.g. `"ZFI_PROCESS:00000000000000000000000000001234"`
(`instance_id` is `zfi_process_instance_id`, CHAR 32)

**Parameters JSON contract:**
```json
{
  "process_type":   "<zfi_process_proc_type>",
  "company_code":   "<fis_bukrs>",
  "fiscal_year":    "<fis_gjahr>",
  "fiscal_period":  "<fins_fiscalperiod>",
  "alloc_id":       "<zfi_alloc_id>",
  "force_run":      "X"
}
```
Fields `alloc_id` and `force_run` are optional (adapter treats missing as initial).

**Status Mapping (ZFI_PROCESS → ZEN_ORCH):**

| ZFI_PROCESS status | ZEN_ORCH status |
|--------------------|----------------|
| `N` (NEW) | RUNNING |
| `Q` (QUEUED) | RUNNING |
| `R` (RUNNING) | RUNNING |
| `S` (SUPERSEDED) | COMPLETED |
| `C` (COMPLETED) | COMPLETED |
| `E` (ERROR) | FAILED |
| `A` (CANCELLED) | CANCELLED |
| anything else | RUNNING (treat as in-progress) |

**Acceptance Criteria:**

**Given** `iv_params_json` contains a valid process_type and business parameters
**When** `start( iv_params_json )` is called
**Then** it deserializes params using `XCO_CP_JSON`
**And** calls `ZCL_FI_PROCESS_MANAGER=>get_instance( )->create_process( iv_process_type = ... is_init_params = ... )`
**And** calls `ZCL_FI_PROCESS_MANAGER=>get_instance( )->request_execute_process( iv_instance_id = ... )` to dispatch via APJ
**And** returns `rs_result-work_unit_handle = 'ZFI_PROCESS:' && lv_instance_id`
**And** returns `rs_result-status = gc_status-running`

**Given** `ZCX_FI_PROCESS_ERROR` is raised during `create_process` or `request_execute_process`
**When** `start()` catches it
**Then** it raises `ZCX_EN_ORCH_ERROR` with textid `ADAPTER_START_FAILED`, `MV_ADAPTER_TYPE = 'ZFI_PROCESS'`, `MV_PERF_UUID` = calling performance UUID (passed via `iv_params_json` convention — see Notes)

**Given** `iv_handle` = `"ZFI_PROCESS:<instance_id>"`
**When** `get_status( iv_handle )` is called
**Then** it parses the instance_id from the handle (split by `:`, take field 2)
**And** calls `ZCL_FI_PROCESS_MANAGER=>get_instance( )->load_process( iv_instance_id = ... )`
**And** maps the loaded instance status to a ZEN_ORCH status using the mapping table above
**And** returns the mapped status

**Given** `ZCX_FI_PROCESS_ERROR` is raised during `load_process` (transient error)
**When** `get_status()` catches it
**Then** it re-raises as `ZCX_EN_ORCH_ERROR` with textid `ADAPTER_START_FAILED`
(engine treats all `get_status` exceptions as transient — skip and retry next sweep)

**And** `ZCL_FI_ALLOC_ORCH_ADAPTER` has a factory method `create( ) RETURNING VALUE(ro_adapter) TYPE REF TO zif_en_orch_adapter`
**And** the class activates without errors

**Notes:**
- Adapter does NOT call `execute_process` directly — it uses `request_execute_process` (APJ dispatch) to avoid blocking the engine sweep LUW
- `alloc_id` → `is_init_params` mapping: use `ZFI_ALLOC_INIT_PARAMS` DDIC structure (consult SAP docs / existing step classes for exact field names)
- If `process_type` is missing from params JSON, raise `ZCX_EN_ORCH_ERROR` with textid `ADAPTER_START_FAILED` before calling the planner
- Consult `SAP_Docs_MCP_search` for `XCO_CP_JSON` deserialization pattern before implementing

---

### Story 9.2: ZCL_FI_ALLOC_ORCH_ADAPTER — cancel(), restart(), get_result(), get_detail_link()

As an engine,
I want the remaining four adapter methods implemented,
So that the full `ZIF_EN_ORCH_ADAPTER` contract is satisfied.

**target_repository:** `cz.imcg.fast.ovysledovka`
**depends_on:** ["9.1"]
**constitution_principles:**
  - Principle II — SAP Standards
  - Principle V — Error Handling

**Acceptance Criteria:**

**Given** `iv_handle` = `"ZFI_PROCESS:<instance_id>"`
**When** `cancel( iv_handle )` is called
**Then** it calls `ZCL_FI_PROCESS_MANAGER=>get_instance( )->cancel_process( iv_instance_id = ... iv_no_commit = abap_true )`
**And** `ZCX_FI_PROCESS_ERROR` is caught and re-raised as `ZCX_EN_ORCH_ERROR` with textid `ADAPTER_START_FAILED`
**And** no COMMIT is issued (engine owns the LUW — `iv_no_commit = abap_true`)

**Given** `iv_handle` = `"ZFI_PROCESS:<instance_id>"`
**When** `restart( iv_handle )` is called
**Then** it calls `ZCL_FI_PROCESS_MANAGER=>get_instance( )->restart_process( iv_instance_id = ... )`
**And** calls `request_execute_process( iv_instance_id = ... iv_no_commit = abap_true )` to re-dispatch
**And** returns `rs_result-work_unit_handle = iv_handle` (same handle — same process instance)
**And** returns `rs_result-status = gc_status-running`

**Given** `iv_handle` = `"ZFI_PROCESS:<instance_id>"`
**When** `get_result( iv_handle )` is called
**Then** it loads the process instance and serializes `COMPLETED_AT`, `CREATED_BY`, `STATUS` as a JSON string
**And** if the instance is not found, returns an empty string (no exception — result is optional)

**Given** `iv_handle` = `"ZFI_PROCESS:<instance_id>"`
**When** `get_detail_link( iv_handle )` is called
**Then** it returns a URL string of the form `/sap/bc/adt/oo/classes/zcl_fi_process_instance/source/main?instance_id=<id>` OR the allocation dashboard deep link URL if determinable
**And** if no URL can be determined, returns an empty string (no exception — deep link is optional)

**And** the class activates without errors after all 6 methods are implemented

**Notes:**
- `cancel` uses `iv_no_commit = abap_true` — this is the RAP/adapter convention in the planner; engine commits via `sweep_all`
- `restart` handle stays the same — the engine stores the handle, and the restarted process has the same instance_id
- `get_detail_link` format is not critical — a static format string is acceptable; it can be refined in Phase 4
- `get_result` JSON format: `{"status":"<C/F/X>","completed_by":"<user>"}` — minimal; used for logging only

---

### Story 9.3: Adapter Registration and Smoke Score Definition

As an engineer,
I want `ZCL_FI_ALLOC_ORCH_ADAPTER` registered in `ZEN_ORCH_ADAPTER_REG` and a smoke score `ALLOC_STEP` defined in `ZEN_ORCH_SCORE` / `ZEN_ORCH_SCORE_STEP`,
So that the engine can dispatch a real allocation step when given score `ALLOC_STEP`.

**target_repository:** `cz.imcg.fast.ovysledovka` (adapter registration report) + `cz.en.orch` (score data load program)
**depends_on:** ["9.2"]
**constitution_principles:**
  - Principle II — SAP Standards

**Acceptance Criteria:**

**Given** `ZEN_ORCH_ADAPTER_REG` exists
**When** the adapter registration script (a small ABAP report `ZEN_ORCH_SETUP` or manual SE16 insert) is executed
**Then** a row exists in `ZEN_ORCH_ADAPTER_REG` with `ADAPTER_TYPE = 'ZFI_PROCESS'` and `IMPL_CLASS = 'ZCL_FI_ALLOC_ORCH_ADAPTER'`

**Given** `ZEN_ORCH_SCORE` and `ZEN_ORCH_SCORE_STEP` exist
**When** the setup script inserts the smoke score
**Then** a row exists in `ZEN_ORCH_SCORE` with `SCORE_ID = 'ALLOC_STEP'` and `DESCRIPTION = 'Single Allocation Step Smoke Test'`
**And** a single row exists in `ZEN_ORCH_SCORE_STEP` with `SCORE_ID = 'ALLOC_STEP'`, `SCORE_SEQ = 10`, `ELEM_TYPE = 'STEP'`, `ADAPTER_TYPE = 'ZFI_PROCESS'`, and `PARAMS_JSON` = a valid minimal params JSON (process_type + a known company code from the development system)

**And** the setup script is committed as `ZEN_ORCH_SETUP.prog.abap` in `cz.en.orch`
**And** it is idempotent (safe to run multiple times — uses INSERT OR UPDATE / MODIFY)

---

### Story 9.4: End-to-End Integration Test

As a developer,
I want to verify that the full ZEN_ORCH → Planner Adapter → ZFI_PROCESS chain works end-to-end,
So that we have confirmed integration before committing to production use.

**target_repository:** `cz.en.orch`
**depends_on:** ["9.3"]
**constitution_principles:**
  - Principle II — SAP Standards
  - Principle V — Error Handling

**Acceptance Criteria:**

**Given** score `ALLOC_STEP` is registered and adapter `ZFI_PROCESS` is registered
**When** the integration test program (extend `ZCL_EN_ORCH_HEALTH_CHK_QUERY` with a new capability `ALLOC_ADAPTER`) executes
**Then** `ZCL_EN_ORCH_ENGINE=>create_performance( iv_score_id = 'ALLOC_STEP' )` succeeds and returns a non-initial UUID
**And** a row exists in `ZEN_ORCH_PERF` with STATUS = P or R
**And** `sweep_all( )` runs without raising an exception
**And** after the sweep, the performance step STATUS is either R (RUNNING — bgRFC dispatched) or C (COMPLETED — if synchronous)
**And** the health check capability `ALLOC_ADAPTER` appears as COMPLETED (status C) in `ZEN_ORCH_I_HEALTH_CHK_CE`

**Given** the performance was created in the test
**When** the test completes
**Then** `cancel_performance( iv_perf_uuid = lv_uuid )` is called to clean up (best-effort — no assertion on cancel result)

**And** no changes to production data are required for the smoke test — use a test company code / non-production fiscal year from `ZEN_ORCH_SETUP.prog.abap` params
**And** the health check query result is committed with a note comment documenting which dev system was used

---

## New Objects — Epic 9

| Object | Type | Repo | Story |
|--------|------|------|-------|
| `ZCL_FI_ALLOC_ORCH_ADAPTER` | Class | ovysledovka | 9.1 + 9.2 |
| `ZEN_ORCH_SETUP` | Program (ABAP report) | cz.en.orch | 9.3 |
| Health check capability `ALLOC_ADAPTER` | Code extension to `ZCL_EN_ORCH_HEALTH_CHK_QUERY` | cz.en.orch | 9.4 |

---

## Phase 4 Scope (Deferred)

- **Step-level BSR collision locking** (brainstorming #26) — block dispatch when another performance holds the same business key at step granularity
- **`ZIF_FI_ORCH_STATE_SERVICE`** — domain state service interface for external business state queries (brainstorming #23, #24)
- **Year-refresh score** (`ALLOC_YEAR_REFRESH`) — multi-company, multi-period score with LOOP elements; requires state service for PREREQ_GATE cross-period dependency
- **Score management UI** — CRUD for `ZEN_ORCH_SCORE` / `ZEN_ORCH_SCORE_STEP` (currently managed via SE16/transport)
- **Adapter deep link refinement** — replace placeholder URL in `get_detail_link()` with real allocation dashboard deep link

---

## Phase 3 Summary

| Epic | Stories | Target Repo | Key Output |
|------|---------|-------------|------------|
| Epic 8: Dashboard | 8.1–8.5 | cz.en.orch | Fiori Elements list + detail + actions |
| Epic 9: Planner Adapter | 9.1–9.4 | ovysledovka | `ZCL_FI_ALLOC_ORCH_ADAPTER` + E2E test |
