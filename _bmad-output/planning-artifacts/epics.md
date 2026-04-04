---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - '_bmad-output/planning-artifacts/architecture.md'
  - '_bmad-output/brainstorming/brainstorming-session-2026-04-01-100000.md'
  - '_bmad-output/project-context.md'
workflowType: 'epics-and-stories'
project_name: 'ZEN_ORCH — Process Orchestration Engine'
user_name: 'Zdenek'
date: '2026-04-04'
lastStep: 4
status: 'complete'
target_repository: 'cz.en.orch'
---

# ZEN_ORCH — Process Orchestration Engine — Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for the ZEN_ORCH Process Orchestration Engine, decomposing the requirements from the Architecture Decision Document and brainstorming session into implementable stories.

**Target Repository:** `cz.en.orch` (`/Users/smolik/DEV/cz.en.orch`)
**Package:** `ZEN_ORCH`
**Phase Scope:** Phase 1 (Foundation + Engine + Adapters). Phase 2 (BSR, Dashboard) deferred.

## Requirements Inventory

### Functional Requirements

FR1: Declarative scenario definition (scores) encoding sequential, parallel, gated, and looping execution patterns via flat DDIC tables
FR2: Framework-agnostic adapter interface (6 methods: start, get_status, cancel, restart, get_result, get_detail_link)
FR3: Runtime execution instances (performances) with UUID, score snapshot, mutable step statuses
FR4: Stateless periodic engine (APJ sweep) with run-until-blocked synchronous advancement
FR5: Business State Registry (BSR) providing strategic overview, prerequisite resolution, and collision locking across performances — **DEFERRED TO PHASE 2**
FR6: Three-level observability dashboard (all performances → performance detail → adapter deep link) — **DEFERRED TO PHASE 2**
FR7: Periodic scheduling via schedule config table
FR8: Breakpoint/checkpoint support for manual review gates
FR9: Hierarchical JSON parameter inheritance (score → loop → step)
FR10: Adapter type registry with factory instantiation

### Non-Functional Requirements

NFR1: Idempotency — Engine sweeps must be crash-safe; no in-memory state between sweeps
NFR2: Concurrency — Parallel group dispatch with configurable MAX_PARALLEL limits; BSR collision detection (Phase 2)
NFR3: Resilience — Transient adapter errors (get_status exceptions) must not permanently fail performances
NFR4: Auditability — Fail-stop error strategy; explicit recovery, no silent skip-and-continue
NFR5: Extensibility — New frameworks pluggable via adapter registry row; zero orchestrator code changes
NFR6: Isolation — Score edits must not affect running performances (copy-on-start snapshot)

### Additional Requirements

- DDIC-First: All persistence and cross-boundary types must be DDIC objects (Constitution Principle I)
- Factory Pattern: Adapter registry + factory; no direct NEW for framework classes (Constitution Principle IV)
- Own Exception Class: ZCX_EN_ORCH_ERROR independent of ZCX_FI_PROCESS_ERROR (Constitution Principle V)
- Zero compile-time dependency on ZFI_PROCESS package — adapter bridge lives in ZFI_ALLOC_PROCESS
- XCO_CP_JSON for JSON handling (modern released API, Clean Core A)
- COMMIT per performance — each performance advancement committed independently
- Own BAL logging via ZIF_EN_ORCH_LOGGER + ZEN_ORCH BAL object
- abaplint 120-char line limit enforced
- ABAP-Doc on all public methods
- Client-dependent tables (MANDT key, audit fields: CREATED_BY, CREATED_AT, CHANGED_BY, CHANGED_AT)
- 12-layer activation order must be followed in implementation sequence
- Target repository: cz.en.orch (new independent git repo)

### UX Design Requirements

N/A — Phase 1 is backend only. Dashboard UI is deferred to Phase 2.

### FR Coverage Map

| FR | Epic(s) | Coverage |
|----|---------|----------|
| FR1: Declarative score definition | E1 (tables), E4 (execution) | COVERED |
| FR2: 6-method adapter interface | E3 | COVERED |
| FR3: Performance instances + snapshot | E1 (tables), E4 (creation + execution) | COVERED |
| FR4: Stateless engine + sweep | E4 | COVERED |
| FR5: BSR | — | DEFERRED (Phase 2) |
| FR6: Dashboard | — | DEFERRED (Phase 2) |
| FR7: Periodic scheduling | E5 | COVERED |
| FR8: Breakpoint/PAUSED support | E4 (status model), E5 (resume) | COVERED |
| FR9: JSON parameter inheritance | E4 (resolve_params) | COVERED |
| FR10: Adapter registry + factory | E1 (registry table), E3 (interface + factory) | COVERED |

## Epic List

### Epic 1: Repository Bootstrap & DDIC Foundation
The ZEN_ORCH package exists, activates cleanly, and provides the complete type system (domains, data elements, structures, table types, database tables) that all subsequent epics build on.
**FRs covered:** FR1, FR3, FR10

### Epic 2: Error Handling & Logging Infrastructure
The orchestrator has its own exception class, message class, and logging system — all operational code can raise errors and write logs from day one.
**FRs covered:** Cross-cutting (enables all FRs)

### Epic 3: Adapter Framework
An operator can register a new framework adapter by adding a row to the registry table, and the orchestrator can instantiate and interact with adapters through a uniform 6-method contract.
**FRs covered:** FR2, FR10

### Epic 4: Engine Core
The system can create a performance from a score, execute it end-to-end via the run-until-blocked algorithm, handle sequential/parallel/gated/loop patterns, and commit each performance independently.
**FRs covered:** FR1, FR3, FR4, FR8, FR9

### Epic 5: APJ Scheduling & Performance Lifecycle
An operator can schedule periodic engine sweeps, and can cancel, restart, or resume individual performances through the engine's public API.
**FRs covered:** FR7, FR8

### Epic 6: Integration Testing & Repository Finalization
The orchestrator is verified end-to-end with the mock adapter, the repository is properly configured (abaplint, constitution, README), and the package is ready for Phase 2 work.
**FRs covered:** Validates all Phase 1 FRs

### Dependency Flow

```
E1 (DDIC Foundation)
 └──▶ E2 (Errors & Logging)
       └──▶ E3 (Adapter Framework)
             └──▶ E4 (Engine Core)
                   └──▶ E5 (Scheduling & Lifecycle)
                         └──▶ E6 (Integration Testing)
```

---

## Epic 1: Repository Bootstrap & DDIC Foundation

The ZEN_ORCH package exists, activates cleanly, and provides the complete type system (domains, data elements, structures, table types, database tables) that all subsequent epics build on. Follows architecture activation order layers 1–5.

**Target repository:** `cz.en.orch`
**Constitution principles:** I (DDIC-First), II (SAP Standards)

---

### Story 1.1: Initialize repository and package scaffold

As a developer,
I want the `cz.en.orch` repository configured with abapGit structure, abaplint config, and the ZEN_ORCH package definition,
So that all subsequent DDIC objects and classes have a proper, transportable home.

**Acceptance Criteria:**

**Given** the `cz.en.orch` git repository exists at `/Users/smolik/DEV/cz.en.orch`
**When** the repository scaffold is created
**Then** `src/zen_orch/package.devc.xml` exists defining package `ZEN_ORCH`
**And** `.abaplint.json` is present and enforces a 120-character line length limit
**And** `.gitignore` excludes standard non-source SAP artifacts
**And** `src/zen_orch/.constitution.md` is present (copied from planning repo via sync-constitution.sh)
**And** `README.md` describes the repository purpose and package structure

---

### Story 1.2: Create ZEN_ORCH domains

As a developer,
I want all ZEN_ORCH domains defined in DDIC,
So that data elements and table fields have proper value ranges, fixed values, and field labels.

**Acceptance Criteria:**

**Given** the ZEN_ORCH package exists
**When** all domains are activated in ABAP
**Then** `ZEN_ORCH_STATUS` (CHAR 1) exists with fixed values: P=Pending, R=Running, C=Completed, F=Failed, X=Cancelled, B=Paused
**And** `ZEN_ORCH_ELEM_TYPE` (CHAR 12) exists with fixed values: STEP, LOOP, END_LOOP, GATE, PREREQ_GATE
**And** `ZEN_ORCH_SCORE_ID` (CHAR 30) exists with appropriate field label
**And** `ZEN_ORCH_ADAPTER_TYPE` (CHAR 30) exists with appropriate field label
**And** `ZEN_ORCH_SCORE_SEQ` (NUMC 4) exists for sequence numbering
**And** `ZEN_ORCH_MAX_PARALLEL` (INT4) exists for parallel dispatch limits
**And** all domains activate without errors

---

### Story 1.3: Create data elements, structures, and table types

As a developer,
I want all data elements, the start result structure, and table types defined in DDIC,
So that database tables and method signatures use fully typed, self-documenting fields.

**Acceptance Criteria:**

**Given** all domains from Story 1.2 exist and are active
**When** data elements, structures, and table types are activated
**Then** the following data elements exist and reference the correct domains: `ZEN_ORCH_DE_STATUS`, `ZEN_ORCH_DE_ELEM_TYPE`, `ZEN_ORCH_DE_SCORE_ID`, `ZEN_ORCH_DE_ADAPTER_TYPE`, `ZEN_ORCH_DE_SCORE_SEQ`, `ZEN_ORCH_PERF_UUID` (RAW 16 / SYSUUID_X16), `ZEN_ORCH_WU_HANDLE` (CHAR 255, no domain — opaque handle), `ZEN_ORCH_DE_REF_ID` (CHAR 20), `ZEN_ORCH_DE_LOOP_ITER` (INT4), `ZEN_ORCH_DE_PARAMS_JSON` (STRING), `ZEN_ORCH_DE_MAX_PAR`, `ZEN_ORCH_DE_IMPL_CLASS` (CHAR 30)
**And** structure `ZEN_ORCH_S_START_RESULT` exists with fields `WORK_UNIT_HANDLE TYPE zen_orch_wu_handle` and `STATUS TYPE zen_orch_de_status`
**And** table type `ZEN_ORCH_TT_SCORE_STEP` exists as TABLE OF `ZEN_ORCH_SCORE_STEP` (forward reference — activated after Story 1.4)
**And** table type `ZEN_ORCH_TT_PERF_STEP` exists as TABLE OF `ZEN_ORCH_PERF_STEP` (forward reference — activated after Story 1.4)
**And** all objects activate without errors

---

### Story 1.4: Create database tables

As a developer,
I want all ZEN_ORCH database tables defined, activated, and client-dependent,
So that the engine can persist scores, performances, adapter registrations, and schedule configuration.

**Acceptance Criteria:**

**Given** all data elements from Story 1.3 exist
**When** database tables are activated
**Then** `ZEN_ORCH_SCORE` exists with primary key MANDT + SCORE_ID, fields DESCRIPTION (CHAR 60), PARAMS_JSON, and audit fields (CREATED_BY, CREATED_AT, CHANGED_BY, CHANGED_AT)
**And** `ZEN_ORCH_SCORE_STEP` exists with primary key MANDT + SCORE_ID + SCORE_SEQ, fields ELEM_TYPE, REF_ID, ADAPTER_TYPE, PARAMS_JSON, MAX_PARALLEL
**And** `ZEN_ORCH_PERF` exists with primary key MANDT + PERF_UUID, fields SCORE_ID, STATUS, PARAMS_JSON, and audit fields
**And** `ZEN_ORCH_PERF_STEP` exists with primary key MANDT + PERF_UUID + SCORE_SEQ + LOOP_ITERATION, fields ELEM_TYPE, REF_ID, ADAPTER_TYPE, PARAMS_JSON, STATUS, WORK_UNIT_HANDLE
**And** `ZEN_ORCH_ADAPTER_REG` exists with primary key MANDT + ADAPTER_TYPE, field IMPL_CLASS
**And** `ZEN_ORCH_SCHEDULE` exists with primary key MANDT + SCHEDULE_ID, fields SCORE_ID, PARAMS_JSON, CRON_EXPR, ACTIVE flag
**And** all tables are client-dependent (MANDT first key field) and include all four audit fields
**And** all tables activate without errors

---

## Epic 2: Error Handling & Logging Infrastructure

The orchestrator has its own exception class, message class, and BAL logging system. All operational code in subsequent epics can raise structured errors and write application log entries from day one, with no compile-time dependency on ZFI_PROCESS.

**Target repository:** `cz.en.orch`
**Constitution principles:** II (SAP Standards), V (Error Handling — own exception class)
**Depends on:** Epic 1

---

### Story 2.1: Create message class and exception class

As a developer,
I want the ZEN_ORCH message class and exception class available,
So that all orchestrator errors carry structured context (performance UUID, score ID, adapter type) and display meaningful messages.

**Acceptance Criteria:**

**Given** the ZEN_ORCH package and DDIC types from Epic 1 exist
**When** the message class and exception class are activated
**Then** message class `ZEN_ORCH` (T100) exists with messages in number ranges: 001–019 engine lifecycle, 020–039 adapter dispatch, 040–059 score/performance creation, 060–079 validation errors
**And** exception class `ZCX_EN_ORCH_ERROR` exists inheriting from `CX_STATIC_CHECK`
**And** `ZCX_EN_ORCH_ERROR` declares textids for: `ENGINE_SWEEP_FAILED`, `ADAPTER_START_FAILED`, `SCORE_NOT_FOUND`, `PERFORMANCE_COLLISION`, `INVALID_SCORE_STRUCTURE`, `STEP_RESTART_FAILED`
**And** each textid references message class `ZEN_ORCH` with the appropriate message number
**And** `ZCX_EN_ORCH_ERROR` has instance attributes: `MV_PERF_UUID TYPE zen_orch_perf_uuid`, `MV_SCORE_ID TYPE zen_orch_de_score_id`, `MV_ADAPTER_TYPE TYPE zen_orch_de_adapter_type`, `MV_SCORE_SEQ TYPE zen_orch_de_score_seq`, `MV_DETAIL TYPE string`
**And** `ZCX_EN_ORCH_ERROR` has no reference to `ZCX_FI_PROCESS_ERROR` (zero dependency)
**And** the class activates without errors

---

### Story 2.2: Create logger interface and BAL implementation

As a developer,
I want a logger interface and BAL implementation available,
So that all orchestrator classes can write structured log entries to SLG1 via a consistent contract, independent of the planner framework logger.

**Acceptance Criteria:**

**Given** the exception class from Story 2.1 exists
**When** the logger interface and implementation are activated
**Then** interface `ZIF_EN_ORCH_LOGGER` exists with methods: `log_info`, `log_warning`, `log_error`, `log_exception` (accepting `ZCX_EN_ORCH_ERROR`), `save`
**And** all interface methods carry ABAP-Doc comments with `@parameter` and `@raising` annotations
**And** class `ZCL_EN_ORCH_LOGGER` implements `ZIF_EN_ORCH_LOGGER` using BALI/BAL application log
**And** `ZCL_EN_ORCH_LOGGER` registers under BAL object `ZEN_ORCH`
**And** `ZCL_EN_ORCH_LOGGER` uses a factory method `create` (no direct NEW in caller code)
**And** `ZIF_EN_ORCH_LOGGER` has no reference to `ZIF_FI_PROCESS_LOGGER` (zero dependency)
**And** both objects activate without errors

**Given** a caller creates a logger via `ZCL_EN_ORCH_LOGGER=>create( )`
**When** `log_info` is called with a message text
**Then** the entry is written to the BAL log under object `ZEN_ORCH`
**And** calling `save` persists the log so it is visible in SLG1

---

## Epic 3: Adapter Framework

An operator can register a new framework adapter by inserting a row into `ZEN_ORCH_ADAPTER_REG`, and the engine can instantiate and call any registered adapter through the uniform 6-method `ZIF_EN_ORCH_ADAPTER` contract. A mock adapter enables full engine testing without any external dependency.

**Target repository:** `cz.en.orch`
**Constitution principles:** I (DDIC-First), II (SAP Standards), III (Consult SAP Docs), IV (Factory Pattern)
**Depends on:** Epics 1, 2

---

### Story 3.1: Create adapter interface

As a developer,
I want the `ZIF_EN_ORCH_ADAPTER` interface defined with its 6-method contract,
So that any framework adapter (planner, workflow, mock) implements a uniform interaction pattern with the engine.

**Acceptance Criteria:**

**Given** the DDIC types from Epic 1 exist
**When** `ZIF_EN_ORCH_ADAPTER` is activated
**Then** the interface declares exactly 6 methods with the correct signatures:
- `start( IMPORTING iv_params_json TYPE string RETURNING VALUE(rs_result) TYPE zen_orch_s_start_result RAISING zcx_en_orch_error )`
- `get_status( IMPORTING iv_handle TYPE zen_orch_wu_handle RETURNING VALUE(rv_status) TYPE zen_orch_de_status RAISING zcx_en_orch_error )`
- `cancel( IMPORTING iv_handle TYPE zen_orch_wu_handle RAISING zcx_en_orch_error )`
- `restart( IMPORTING iv_handle TYPE zen_orch_wu_handle RETURNING VALUE(rs_result) TYPE zen_orch_s_start_result RAISING zcx_en_orch_error )`
- `get_result( IMPORTING iv_handle TYPE zen_orch_wu_handle RETURNING VALUE(rv_result_json) TYPE string RAISING zcx_en_orch_error )`
- `get_detail_link( IMPORTING iv_handle TYPE zen_orch_wu_handle RETURNING VALUE(rv_url) TYPE string )`
**And** all methods have ABAP-Doc comments
**And** the interface has no reference to ZFI_PROCESS objects
**And** the interface activates without errors

---

### Story 3.2: Create adapter factory

As a developer,
I want the adapter factory to instantiate the correct adapter class from the registry,
So that the engine never uses direct `NEW` for adapter creation and new adapters require zero engine code changes.

**Acceptance Criteria:**

**Given** `ZIF_EN_ORCH_ADAPTER` and `ZEN_ORCH_ADAPTER_REG` table exist
**When** `ZCL_EN_ORCH_ADAPTER_FACTORY=>create( iv_adapter_type )` is called
**Then** the factory reads `IMPL_CLASS` from `ZEN_ORCH_ADAPTER_REG` for the given `ADAPTER_TYPE`
**And** instantiates the class dynamically via `CREATE OBJECT lo_adapter TYPE (lv_impl_class)`
**And** returns the instance cast to `ZIF_EN_ORCH_ADAPTER`
**And** if `ADAPTER_TYPE` is not found in the registry, raises `ZCX_EN_ORCH_ERROR` with textid `ADAPTER_START_FAILED`
**And** the factory class has a private constructor (no direct NEW)
**And** `ZCL_EN_ORCH_ADAPTER_FACTORY` activates without errors

**Given** a valid adapter type is registered
**When** `create` is called twice with the same type
**Then** both calls succeed and return independent instances

---

### Story 3.3: Create mock adapter

As a developer,
I want a mock adapter available for testing the engine,
So that Epic 4 engine development and all ABAP unit tests can run without any dependency on the real ZFI_PROCESS planner framework.

**Acceptance Criteria:**

**Given** `ZIF_EN_ORCH_ADAPTER` exists
**When** `ZCL_EN_ORCH_ADAPTER_MOCK` is activated
**Then** it implements all 6 methods of `ZIF_EN_ORCH_ADAPTER`
**And** `start` immediately returns status COMPLETED (`gc_status-completed`) with a deterministic handle like `'MOCK:<uuid>'`
**And** `get_status` returns COMPLETED for any handle
**And** `cancel`, `restart`, `get_result`, `get_detail_link` return sensible no-op responses
**And** the mock can be registered in `ZEN_ORCH_ADAPTER_REG` with `ADAPTER_TYPE = 'MOCK'` and `IMPL_CLASS = 'ZCL_EN_ORCH_ADAPTER_MOCK'`
**And** `ZCL_EN_ORCH_ADAPTER_MOCK` activates without errors

---

## Epic 4: Engine Core

The system can create a performance from a score, snapshot the score steps, and execute the performance end-to-end via the run-until-blocked algorithm. It handles all execution patterns (sequential, parallel groups, gates, loops, breakpoints) and resolves JSON parameters at three levels. Each performance is committed independently.

**Target repository:** `cz.en.orch`
**Constitution principles:** I, II, III, IV (Factory), V (Error Handling)
**Depends on:** Epics 1, 2, 3

---

### Story 4.1: Create engine singleton with status constants

As a developer,
I want the `ZCL_EN_ORCH_ENGINE` singleton class scaffold with status constants and the public method signatures,
So that all subsequent engine stories build on a consistent, correctly structured class.

**Acceptance Criteria:**

**Given** all DDIC types, exception class, logger, and adapter framework exist
**When** `ZCL_EN_ORCH_ENGINE` is activated
**Then** the class uses the singleton pattern with a `get_instance` class method returning `REF TO zcl_en_orch_engine`
**And** the class declares `CONSTANTS: BEGIN OF gc_status, pending TYPE zen_orch_de_status VALUE 'P', running TYPE zen_orch_de_status VALUE 'R', completed TYPE zen_orch_de_status VALUE 'C', failed TYPE zen_orch_de_status VALUE 'F', cancelled TYPE zen_orch_de_status VALUE 'X', paused TYPE zen_orch_de_status VALUE 'B', END OF gc_status`
**And** the class declares the following public instance methods (stubs for now): `sweep_all`, `create_performance`, `cancel_performance`, `restart_performance`, `resume_performance`
**And** the class declares the following private instance methods (stubs for now): `advance_performance`, `dispatch_step`, `poll_step_status`, `evaluate_gate`, `advance_loop`, `resolve_params`, `snapshot_score`
**And** the class has no reference to ZFI_PROCESS objects (zero compile-time dependency)
**And** the class activates without errors

---

### Story 4.2: Implement performance creation and score snapshot

As an operator,
I want to create a new performance from a score with a parameter set,
So that the orchestrator captures the score definition at start time and the performance is isolated from subsequent score changes.

**Acceptance Criteria:**

**Given** a valid SCORE_ID exists in `ZEN_ORCH_SCORE` with at least one step in `ZEN_ORCH_SCORE_STEP`
**When** `ZCL_EN_ORCH_ENGINE=>get_instance( )->create_performance( iv_score_id = '...' iv_params_json = '...' )` is called
**Then** a new UUID is generated for `PERF_UUID`
**And** a row is inserted into `ZEN_ORCH_PERF` with STATUS = 'P' (PENDING) and the provided PARAMS_JSON
**And** all rows from `ZEN_ORCH_SCORE_STEP` for the score are copied into `ZEN_ORCH_PERF_STEP` with STATUS = 'P' and LOOP_ITERATION = 0
**And** the performance UUID is returned to the caller

**Given** a SCORE_ID that does not exist in `ZEN_ORCH_SCORE`
**When** `create_performance` is called
**Then** `ZCX_EN_ORCH_ERROR` is raised with textid `SCORE_NOT_FOUND` and `MV_SCORE_ID` populated

**Given** a score snapshot is taken
**When** the score steps in `ZEN_ORCH_SCORE_STEP` are subsequently modified
**Then** the existing performance steps in `ZEN_ORCH_PERF_STEP` are unchanged (NFR6: isolation)

---

### Story 4.3: Implement JSON parameter resolution

As an engine,
I want to resolve the effective parameters for each step by merging three levels of JSON,
So that step-level parameters override loop-level parameters, which override score-level parameters, and loop variables are substituted.

**Acceptance Criteria:**

**Given** a performance has score-level params `{"company_code": "1000", "fiscal_year": "2026"}` and a step has step-level params `{"period": "001"}`
**When** `resolve_params` is called for that step
**Then** the returned JSON contains all three keys: `{"company_code": "1000", "fiscal_year": "2026", "period": "001"}`
**And** step-level values override score-level values for the same key

**Given** a loop step has loop-level params with a `loop_iteration` placeholder
**When** `resolve_params` is called for a step inside the loop at iteration 3
**Then** the placeholder `{{loop_iteration}}` is substituted with `"3"` in the resolved JSON

**Given** any level has an empty or null params JSON
**When** `resolve_params` merges the levels
**Then** the merge skips the empty level without error and the result contains the other levels' values

**And** `resolve_params` uses `XCO_CP_JSON` for all JSON reading and building operations
**And** all JSON keys use `snake_case` convention

---

### Story 4.4: Implement sequential step dispatch and polling

As an engine,
I want to dispatch a STEP element to its adapter and poll its status on subsequent sweeps,
So that the engine advances sequential work units through their lifecycle (PENDING → RUNNING → COMPLETED/FAILED).

**Acceptance Criteria:**

**Given** a performance step with ELEM_TYPE = 'STEP' and STATUS = 'P' (PENDING)
**When** `dispatch_step` is called for that step
**Then** the adapter factory creates the correct adapter for the step's ADAPTER_TYPE
**And** `adapter->start( iv_params_json = <resolved params> )` is called
**And** the returned handle is stored in `ZEN_ORCH_PERF_STEP.WORK_UNIT_HANDLE`
**And** the step STATUS is updated to the status returned by `start` (COMPLETED if synchronous, RUNNING if async)

**Given** a step is in STATUS = 'R' (RUNNING) with a non-empty WORK_UNIT_HANDLE
**When** `poll_step_status` is called for that step
**Then** `adapter->get_status( iv_handle )` is called
**And** the step STATUS is updated to the returned status
**And** if `get_status` raises `ZCX_EN_ORCH_ERROR`, the exception is caught, the step remains RUNNING, and a warning is logged (NFR3: resilient polling — transient errors do not fail the performance)

**Given** `adapter->start` raises `ZCX_EN_ORCH_ERROR`
**When** `dispatch_step` propagates the exception
**Then** the engine marks the performance as FAILED (fail-stop, NFR4)

---

### Story 4.5: Implement gate evaluation and loop advancement

As an engine,
I want to evaluate GATE elements and advance LOOP elements,
So that parallel groups wait for all members before proceeding, and loop iterations execute the correct number of times.

**Acceptance Criteria:**

**Given** a performance step with ELEM_TYPE = 'GATE' and REF_ID = 'GROUP_A'
**When** `evaluate_gate` is called
**Then** all steps in `ZEN_ORCH_PERF_STEP` with the same REF_ID and ELEM_TYPE = 'STEP' are checked
**And** if all those steps have STATUS = 'C' (COMPLETED), the gate step STATUS is set to 'C'
**And** if any step is still RUNNING or PENDING, the gate remains PENDING (blocking the engine — run-until-blocked)

**Given** a performance step with ELEM_TYPE = 'LOOP' with MAX_PARALLEL iterations configured
**When** `advance_loop` is called after all steps within the loop iteration are COMPLETED
**Then** if the current LOOP_ITERATION is less than MAX_PARALLEL (used as iteration count), LOOP_ITERATION is incremented and all steps inside the loop are reset to PENDING for the next iteration
**And** if MAX_PARALLEL iterations are exhausted, the LOOP step STATUS is set to 'C' (completed)
**And** the END_LOOP step mirrors the LOOP step status

---

### Story 4.6: Implement run-until-blocked sweep

As an engine,
I want the sweep algorithm to advance all active performances as far as possible in a single synchronous pass,
So that each APJ sweep call makes maximum progress without waiting for external async work to complete.

**Acceptance Criteria:**

**Given** one or more performances in STATUS = 'P' or 'R' exist in `ZEN_ORCH_PERF`
**When** `sweep_all` is called
**Then** each active performance is processed by `advance_performance`
**And** for each performance, `advance_performance` loops through steps in SCORE_SEQ order and advances each PENDING or RUNNING step until no further progress is possible (blocked on a RUNNING async step or a GATE waiting for parallel steps)
**And** after `advance_performance` completes for a performance, `COMMIT WORK` is issued once for that performance
**And** if `advance_performance` raises `ZCX_EN_ORCH_ERROR`, the performance is marked FAILED, `COMMIT WORK` is issued, and processing continues with the next performance (one failure does not stop the sweep)
**And** a performance where all steps reach STATUS = 'C' has its own STATUS set to 'C' (COMPLETED)
**And** `COMMIT WORK` is never called inside `advance_performance` or its helpers — only in `sweep_all` (NFR1: idempotency)

**Given** a sweep runs and the system crashes mid-way
**When** the next sweep runs
**Then** it re-reads all performance states from the database and continues from where each performance was last committed (no in-memory state dependency)

---

### Story 4.7: Implement breakpoint (PAUSED) support

As an operator,
I want performances to pause at GATE elements configured as breakpoints,
So that I can review intermediate state and manually approve continuation.

**Acceptance Criteria:**

**Given** a performance step with ELEM_TYPE = 'GATE' is configured as a breakpoint (e.g., via a flag in `ZEN_ORCH_PERF_STEP`)
**When** the engine reaches that gate and all prerequisite steps are COMPLETED
**Then** the gate step STATUS is set to 'B' (PAUSED) instead of 'C'
**And** the performance STATUS is set to 'B' (PAUSED)
**And** subsequent sweeps do not advance the PAUSED performance

**Given** a performance is in STATUS = 'B' (PAUSED)
**When** `resume_performance( iv_perf_uuid )` is called
**Then** the PAUSED gate step STATUS is set to 'C' (COMPLETED)
**And** the performance STATUS is set to 'R' (RUNNING)
**And** the next sweep advances the performance past the breakpoint

---

## Epic 5: APJ Scheduling & Performance Lifecycle

An operator can schedule the engine sweep as a periodic APJ job. The engine checks the schedule table and creates due performances automatically. Operators can also cancel or restart individual performances through the engine's public lifecycle API.

**Target repository:** `cz.en.orch`
**Constitution principles:** II (SAP Standards), III (Consult SAP Docs for APJ pattern)
**Depends on:** Epics 1–4

---

### Story 5.1: Create APJ sweep job class

As a system administrator,
I want an APJ job class that invokes the engine sweep,
So that the orchestrator engine runs periodically as a background job without manual intervention.

**Acceptance Criteria:**

**Given** `ZCL_EN_ORCH_ENGINE` exists
**When** `ZCL_EN_ORCH_JOB_SWEEP` is activated and registered in the APJ framework (SAJC/SAJT)
**Then** the class implements the APJ job interface (pattern consistent with `ZCL_FI_PROCESS_JOB_BASE` approach)
**And** its `execute` method calls `ZCL_EN_ORCH_ENGINE=>get_instance( )->sweep_all( )`
**And** any `ZCX_EN_ORCH_ERROR` raised by `sweep_all` is caught, logged via the engine logger, and does not crash the job
**And** the job class has ABAP-Doc on its `execute` method
**And** the class activates without errors

**Given** the APJ job is scheduled to run every N minutes
**When** the job executes
**Then** `sweep_all` is called once per job execution
**And** the job completes without dump even if no active performances exist

---

### Story 5.2: Implement schedule-driven performance creation

As an operator,
I want active schedule entries in `ZEN_ORCH_SCHEDULE` to automatically trigger new performances,
So that recurring orchestration scenarios (e.g., monthly allocation runs) start without manual intervention.

**Acceptance Criteria:**

**Given** a row exists in `ZEN_ORCH_SCHEDULE` with ACTIVE = true, a valid SCORE_ID, and a CRON_EXPR indicating the current time is due
**When** `sweep_all` calls `check_schedules` at the start of each sweep
**Then** a new performance is created for that schedule entry via `create_performance`
**And** the schedule row is updated to record the last triggered timestamp
**And** the same schedule does not trigger again in the same sweep cycle

**Given** a schedule entry points to a SCORE_ID that no longer exists
**When** `check_schedules` processes it
**Then** the error is logged and the schedule entry is skipped — it does not abort the entire sweep

---

### Story 5.3: Implement cancel and restart lifecycle operations

As an operator,
I want to cancel a running performance or restart a failed performance,
So that I can intervene when a performance is stuck or has failed and needs to be retried.

**Acceptance Criteria:**

**Given** a performance in STATUS = 'R' (RUNNING) or 'P' (PENDING)
**When** `cancel_performance( iv_perf_uuid )` is called
**Then** for each step in RUNNING state, `adapter->cancel( iv_handle )` is called (best-effort — exceptions are caught and logged but do not abort the cancel)
**And** all non-COMPLETED steps are set to STATUS = 'X' (CANCELLED)
**And** the performance STATUS is set to 'X' (CANCELLED)
**And** `COMMIT WORK` is issued

**Given** a performance in STATUS = 'F' (FAILED) with one step in STATUS = 'F'
**When** `restart_performance( iv_perf_uuid )` is called
**Then** the failed step STATUS is reset to 'P' (PENDING) and its WORK_UNIT_HANDLE is cleared
**And** the performance STATUS is set to 'R' (RUNNING)
**And** `COMMIT WORK` is issued
**And** the next sweep pick up the performance and re-dispatches the failed step

**Given** a performance in STATUS = 'C' (COMPLETED) or 'X' (CANCELLED)
**When** `cancel_performance` or `restart_performance` is called
**Then** `ZCX_EN_ORCH_ERROR` is raised with textid `STEP_RESTART_FAILED` indicating the operation is not valid for terminal states

---

## Epic 6: Integration Testing & Repository Finalization

The orchestrator is verified end-to-end with the mock adapter via an ABAP Unit test suite and an integration test program. The repository is properly configured and the package is declared ready for Phase 2 work.

**Target repository:** `cz.en.orch`
**Constitution principles:** All (validation)
**Depends on:** Epics 1–5

---

### Story 6.1: ABAP Unit test suite for engine core

As a developer,
I want ABAP Unit tests covering the engine's core logic,
So that regressions in the run-until-blocked algorithm, parameter resolution, and error handling are caught automatically on every push.

**Acceptance Criteria:**

**Given** `ZCL_EN_ORCH_ENGINE` test class `ZCL_EN_ORCH_ENGINE_TEST` (or local test class in `zcl_en_orch_engine.clas.testclasses.abap`) exists
**When** ABAP Unit tests are executed
**Then** the following scenarios are covered with passing tests:
- Creating a performance from a valid score → PENDING status, snapshot created
- Creating a performance from a non-existent score → ZCX_EN_ORCH_ERROR raised
- sweep_all with one STEP performance and MOCK adapter → performance reaches COMPLETED in one sweep
- sweep_all with a GATE → performance blocks at gate until all REF_ID steps are COMPLETED
- sweep_all with transient get_status error → performance stays RUNNING, no FAILED (NFR3)
- sweep_all with start() error → performance marked FAILED (NFR4)
- resolve_params three-level merge → step-level overrides score-level
- cancel_performance on a RUNNING performance → all steps CANCELLED
- restart_performance on a FAILED performance → failed step reset to PENDING
**And** all tests pass with the MOCK adapter (no ZFI_PROCESS dependency)

---

### Story 6.2: Integration test program and repository finalization

As a developer,
I want a runnable integration test program and a finalized repository configuration,
So that I can manually verify the full orchestration lifecycle and the repository is ready for Phase 2 development.

**Acceptance Criteria:**

**Given** all Epic 1–5 objects are active
**When** `ZEN_ORCH_TEST` program is executed
**Then** it runs a complete end-to-end scenario:
  1. Inserts a test score with steps (STEP → GATE → STEP) into `ZEN_ORCH_SCORE` / `ZEN_ORCH_SCORE_STEP`
  2. Registers the MOCK adapter in `ZEN_ORCH_ADAPTER_REG`
  3. Calls `create_performance` and verifies PENDING status
  4. Calls `sweep_all` and verifies the performance reaches COMPLETED
  5. Reads the BAL log and verifies entries were written
  6. Cleans up test data
**And** the program completes without dump or unhandled exception
**And** `README.md` documents: repository purpose, package structure, how to run tests, how to register a new adapter, Phase 2 roadmap
**And** `repos/registry.md` in the planning repo is updated with `cz.en.orch` metadata
**And** `sync-constitution.sh` has been run and `src/zen_orch/.constitution.md` matches the planning repo constitution
