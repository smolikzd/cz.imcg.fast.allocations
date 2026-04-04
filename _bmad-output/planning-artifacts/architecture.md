---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - '_bmad-output/brainstorming/brainstorming-session-2026-04-01-100000.md'
  - '_bmad-output/project-context.md'
workflowType: 'architecture'
project_name: 'cz.imcg.fast.allocations'
user_name: 'Zdenek'
date: '2026-04-03'
lastStep: 8
status: 'complete'
completedAt: '2026-04-03'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements (from brainstorming session 2026-04-01):**

1. Declarative scenario definition (scores) encoding sequential, parallel, gated, and looping execution patterns via flat DDIC tables
2. Framework-agnostic adapter interface (6 methods: start, get_status, cancel, restart, get_result, get_detail_link)
3. Runtime execution instances (performances) with UUID, score snapshot, mutable step statuses
4. Stateless periodic engine (APJ sweep) with run-until-blocked synchronous advancement
5. Business State Registry (BSR) providing strategic overview, prerequisite resolution, and collision locking across performances
6. Three-level observability dashboard (all performances → performance detail → adapter deep link)
7. Periodic scheduling via schedule config table
8. Breakpoint/checkpoint support for manual review gates
9. Hierarchical JSON parameter inheritance (score → loop → step)
10. Adapter type registry with factory instantiation

**Non-Functional Requirements:**

- **Idempotency:** Engine sweeps must be crash-safe — no in-memory state between sweeps
- **Concurrency:** Parallel group dispatch with configurable MAX_PARALLEL limits; BSR collision detection across performances
- **Resilience:** Transient adapter errors (get_status exceptions) must not permanently fail performances
- **Auditability:** Fail-stop error strategy — explicit recovery, no silent skip-and-continue
- **Extensibility:** New frameworks pluggable via adapter registry row — zero orchestrator code changes
- **Isolation:** Score edits must not affect running performances (copy-on-start snapshot)

**Scale & Complexity:**

- Primary domain: ABAP backend / SAP framework orchestration
- Complexity level: High
- Estimated architectural components: ~15 DDIC objects, ~8 classes/interfaces, 2 APJ job registrations
- Two-phase implementation: Foundation (score/performance/engine/dashboard) then Integration (adapters/BSR)

### Technical Constraints & Dependencies

| Constraint | Source | Impact |
|---|---|---|
| ABAP 7.58 on-premise | Technology stack | No ABAP Cloud restrictions; full syntax available |
| DDIC-first types | Constitution Principle I | All persistence and cross-boundary types must be DDIC |
| Factory pattern | Constitution Principle IV | Adapter registry + factory; no direct NEW for framework classes |
| ZCX_FI_PROCESS_ERROR | Constitution Principle V | Error propagation through existing exception hierarchy |
| APJ job framework | Existing pattern | Engine sweep job follows ZCL_FI_PROCESS_JOB_BASE |
| BAL logging | Existing pattern | Engine logging via ZIF_FI_PROCESS_LOGGER |
| No COMMIT WORK in steps | Framework rule | Engine must manage LUW boundaries at orchestrator level |
| SAP HANA DB | Technology stack | Client-dependent tables (MANDT), audit fields required |
| 120-char line limit | abaplint | All ABAP artifacts must comply |
| Existing ZFI_PROCESS framework | Dependency | Orchestrator delegates to, does not replace, planner framework |

### Cross-Cutting Concerns Identified

1. **Concurrency & Locking** — BSR collision detection, parallel group dispatch limits, performance-level mutual exclusion on business keys
2. **Idempotency** — Stateless engine must produce identical results on re-sweep; copy-on-start prevents score mutation during execution
3. **Error Propagation** — Adapter failures must surface through orchestrator as fail-stop; resilient polling must distinguish transient from permanent failures
4. **Parameter Resolution** — Three-level JSON merge (score → loop → step) with placeholder substitution for loop variables
5. **Score Versioning** — Copy-on-start snapshot eliminates versioning complexity but requires careful snapshot integrity
6. **Observability** — Dashboard derived from live performance state + BSR; no separate persistence; must handle high cardinality (60+ steps per year-refresh performance)
7. **Testability** — Phase 1 must be testable with mock adapters; no dependency on real planner framework until Phase 2

## Starter Template Evaluation

### Primary Technology Domain

ABAP 7.58 backend / SAP framework orchestration — no web starter templates apply.

### Package & Naming Convention

**Package:** `ZEN_ORCH`

| Object Type | Pattern | Example |
|---|---|---|
| Package | `ZEN_ORCH` | `ZEN_ORCH` |
| Classes | `ZCL_EN_ORCH_*` | `ZCL_EN_ORCH_ENGINE` |
| Interfaces | `ZIF_EN_ORCH_*` | `ZIF_EN_ORCH_ADAPTER` |
| Exception class | `ZCX_EN_ORCH_ERROR` | Own exception hierarchy, independent of `ZCX_FI_PROCESS_ERROR` |
| Tables | `ZEN_ORCH_*` | `ZEN_ORCH_SCORE` |
| Table types | `ZEN_ORCH_TT_*` | `ZEN_ORCH_TT_SCORE_STEP` |
| Domains | `ZEN_ORCH_*` | `ZEN_ORCH_STATUS` |
| Data elements | `ZEN_ORCH_*` | `ZEN_ORCH_PERF_UUID` |
| Message class | `ZEN_ORCH` | T100 messages for orchestrator |
| BAL object | `ZEN_ORCH` | SLG1 application log object |

### Existing Foundation Analysis

This project builds on the established ZFI_PROCESS framework patterns and project constitution, but lives in its own independent package `ZEN_ORCH` with its own exception class `ZCX_EN_ORCH_ERROR`.

**Architectural Decisions Already Established by Existing Framework:**

| Decision Area | Established Standard | Source |
|---|---|---|
| Type system | DDIC-first (structures, table types, domains) | Constitution Principle I |
| Instantiation | Factory methods, no direct `NEW` for framework classes | Constitution Principle IV |
| Logging | BAL (SLG1) with fail-hard semantics | ADR-008 pattern |
| Background jobs | APJ job pattern | project-context.md |
| DB conventions | MANDT key, audit fields, domains with fixed values | Constitution + project-context |
| Line length | ≤120 chars (abaplint enforced) | project-context.md Rule 2 |
| Documentation | ABAP-Doc on all public methods | Constitution Principle II |

**Key Independence Decisions:**

- **Own exception class** `ZCX_EN_ORCH_ERROR` — orchestrator errors are independent of planner framework errors; adapters translate their exceptions at the boundary
- **Own package** `ZEN_ORCH` — no compile-time dependency on `ZFI_PROCESS` package; adapters bridge the gap at runtime
- **Own naming prefix** `ZEN_ORCH_*` / `ZCL_EN_ORCH_*` — clearly distinguishes orchestrator objects from planner objects

**Decisions Remaining for This Architecture:**

1. Repository assignment (new repo vs. existing planner/ovysledovka)
2. DDIC object catalog (score, performance, BSR, adapter registry tables)
3. Interface definitions (adapter contract, state service contract)
4. Engine class design and instantiation pattern
5. JSON handling approach for parameter resolution
6. LUW/COMMIT strategy for engine sweep boundaries
7. BAL object/subobject allocation for orchestrator logging
8. Orchestrator status model (reuse six universal statuses from brainstorming)
9. Exception class design (textids, context fields)
10. Relationship between orchestrator adapter and planner framework (bridge pattern)

**Note:** First implementation story should create the DDIC foundation (domains, data elements, table types) before any class development — following the activation order documented in project-context.md.

## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**

| # | Decision | Choice | Rationale |
|---|---|---|---|
| D1 | Repository | New repo `cz.en.orch` | Clean separation; independent release cycle; enforces zero compile-time dependency on planner |
| D2 | Compile-time dependency on ZFI_PROCESS | Zero dependency | Orchestrator knows nothing about planner at compile time; adapter bridge class lives in `ZFI_ALLOC_PROCESS` package, not in `ZEN_ORCH` |
| D3 | Engine instantiation | Singleton via `ZCL_EN_ORCH_ENGINE=>get_instance()` | Matches established `ZCL_FI_PROCESS_MANAGER` pattern; consistent developer experience |
| D4 | JSON handling | `XCO_CP_JSON` | Modern released API (Clean Core A, SAP_BASIS); fluent builder/reader pattern; available on 7.58 on-premise. Note: planner currently uses `/UI2/CL_JSON` — orchestrator intentionally uses the newer API |
| D5 | LUW/COMMIT strategy | COMMIT per performance | Each performance advancement committed independently; one failure doesn't roll back other performances in the same sweep |
| D6 | Logging | Own interface `ZIF_EN_ORCH_LOGGER` with own BAL object `ZEN_ORCH` | Complete independence from planner logging; own logger contract |

**Important Decisions (Shape Architecture — from brainstorming, confirmed):**

| Decision | Choice | Source |
|---|---|---|
| Status model | Six universal statuses: PENDING, RUNNING, COMPLETED, FAILED, CANCELLED, PAUSED | Brainstorming #15 |
| Error strategy | Fail-stop — any step failure stops entire performance | Brainstorming #16 |
| Score model | Flat table with TYPE + RefID encoding all execution patterns | Brainstorming #9 |
| Parameter model | Opaque JSON with three-level merge (score → loop → step) | Brainstorming #10, #11 |
| Engine algorithm | Run-until-blocked + periodic APJ sweep | Brainstorming #12, #13 |
| Score versioning | Copy-on-start snapshot into performance steps | Brainstorming #19 |
| Adapter pattern | 6-method interface + registry table + factory | Brainstorming #2, #4 |
| Resilient polling | `get_status()` exception → skip, retry next sweep (not fail) | Brainstorming #20 |

**Deferred Decisions (Phase 2):**

| Decision | Rationale for Deferral |
|---|---|
| BSR persistence structure | Depends on real adapter integration experience |
| State registration timing (before/after `start()`) | Depends on BSR design |
| Performance UUID passing to adapter | Clean solution identified but not finalized |
| Schedule table CRON format | Dashboard/scheduling is Phase 1 priority 4 |

### Data Architecture

**Repository:** `cz.en.orch` (new, independent git repository)
**Package:** `ZEN_ORCH`
**Database:** SAP HANA (client-dependent, MANDT key field, audit fields on all tables)
**JSON library:** `XCO_CP_JSON` — fluent API for parameter serialization, deserialization, and merge

**XCO JSON usage patterns for the orchestrator:**

```abap
" Reading JSON parameters
xco_cp_json=>data->from_string( lv_params_json )->apply( VALUE #(
  ( xco_cp_json=>transformation->pascal_case_to_underscore )
) )->write_to( REF #( ls_params ) ).

" Building JSON parameters
DATA(lo_builder) = xco_cp_json=>data->builder( ).
lo_builder->begin_object(
  )->add_member( 'fiscal_year' )->add_string( lv_fy
  )->add_member( 'period' )->add_string( lv_period
  )->end_object( ).
DATA(lv_json) = lo_builder->get_data( )->to_string( ).

" Serializing from ABAP structure
DATA(lv_json) = xco_cp_json=>data->from_abap( ls_params )->apply( VALUE #(
  ( xco_cp_json=>transformation->underscore_to_camel_case )
) )->to_string( ).
```

### API & Communication Patterns

**Adapter interface contract** (`ZIF_EN_ORCH_ADAPTER`):

| Method | Signature | Notes |
|---|---|---|
| `start` | `IMPORTING iv_params_json TYPE string RETURNING VALUE(rs_result) TYPE zen_orch_start_result RAISING zcx_en_orch_error` | Returns handle + status (COMPLETED/FAILED/RUNNING) |
| `get_status` | `IMPORTING iv_handle TYPE string RETURNING VALUE(rv_status) TYPE zen_orch_status RAISING zcx_en_orch_error` | Polled by engine; exception = skip, retry next sweep |
| `cancel` | `IMPORTING iv_handle TYPE string RAISING zcx_en_orch_error` | Best-effort cancellation |
| `restart` | `IMPORTING iv_handle TYPE string RETURNING VALUE(rs_result) TYPE zen_orch_start_result RAISING zcx_en_orch_error` | Restart from failed state |
| `get_result` | `IMPORTING iv_handle TYPE string RETURNING VALUE(rv_result_json) TYPE string RAISING zcx_en_orch_error` | Optional result data |
| `get_detail_link` | `IMPORTING iv_handle TYPE string RETURNING VALUE(rv_url) TYPE string` | Deep link to framework-native UI |

**Error handling:** Own exception class `ZCX_EN_ORCH_ERROR` with textids for orchestrator-specific errors. Adapters catch their framework-specific exceptions and translate to `ZCX_EN_ORCH_ERROR` at the boundary.

**LUW strategy:** Engine sweep processes multiple active performances. Each performance's advancement is wrapped in its own LUW — COMMIT WORK after each performance completes its run-until-blocked cycle. One performance failure does not affect others.

### Infrastructure & Logging

**Logging interface:** `ZIF_EN_ORCH_LOGGER` — own contract, independent of `ZIF_FI_PROCESS_LOGGER`

**BAL configuration:**
- BAL object: `ZEN_ORCH`
- BAL subobject: Per-score or per-performance (TBD in implementation)
- Viewable via SLG1

**APJ job:** Engine sweep job extends standard APJ pattern (own job class in `ZEN_ORCH` package, registered in SAJC/SAJT)

### Decision Impact Analysis

**Implementation Sequence:**

1. **New repository** `cz.en.orch` — create repo, abapGit setup, abaplint config
2. **DDIC foundation** — domains (status, score types), data elements, structures, table types
3. **Database tables** — score definition, score steps, performance header, performance steps, adapter registry
4. **Exception class** `ZCX_EN_ORCH_ERROR` — textids for all orchestrator error scenarios
5. **Logger interface** `ZIF_EN_ORCH_LOGGER` + implementation `ZCL_EN_ORCH_LOGGER`
6. **Adapter interface** `ZIF_EN_ORCH_ADAPTER` — 6-method contract
7. **Engine class** `ZCL_EN_ORCH_ENGINE` — singleton, run-until-blocked + sweep logic
8. **Mock adapter** `ZCL_EN_ORCH_ADAPTER_MOCK` — for Phase 1 testing without planner dependency
9. **APJ sweep job** — periodic engine invocation
10. **Planner adapter** (in `ZFI_ALLOC_PROCESS` package, NOT in `ZEN_ORCH`) — bridges to `ZCL_FI_PROCESS_MANAGER`

**Cross-Component Dependencies:**

- `ZEN_ORCH` has ZERO compile-time dependency on `ZFI_PROCESS` or `ZFI_ALLOC_PROCESS`
- Planner adapter class lives in `ZFI_ALLOC_PROCESS` and depends on both `ZEN_ORCH` (implements `ZIF_EN_ORCH_ADAPTER`) and `ZFI_PROCESS` (calls `ZCL_FI_PROCESS_MANAGER`)
- This creates a one-way dependency: `ZFI_ALLOC_PROCESS` → `ZEN_ORCH` (not the reverse)
- Phase 1 is fully testable with `ZCL_EN_ORCH_ADAPTER_MOCK` — no planner dependency needed

## Implementation Patterns & Consistency Rules

### Critical Conflict Points Identified

**8 areas** where AI agents implementing `ZEN_ORCH` code could make different choices, leading to incompatible artifacts:

### 1. DDIC Field Naming

**Rule:** Underscore-separated, uppercase, max 30 chars. Field names match the domain concept, not abbreviations.

| Pattern | Example | Anti-Pattern |
|---|---|---|
| Score ID | `SCORE_ID` | `SCRID`, `SC_ID` |
| Performance UUID | `PERF_UUID` | `PERFORMANCE_UUID` (too long in compound keys), `PF_UUID` |
| Sequence number | `SCORE_SEQ` | `SEQ`, `SEQUENCE_NR` |
| Adapter type | `ADAPTER_TYPE` | `ADPT_TYPE`, `TYPE` |
| Loop iteration | `LOOP_ITERATION` | `LOOP_ITER`, `ITERATION` |
| Work unit handle | `WORK_UNIT_HANDLE` | `HANDLE`, `WU_HANDLE` |

**Audit fields (mandatory on all tables):**
`MANDT`, `CREATED_BY`, `CREATED_AT`, `CHANGED_BY`, `CHANGED_AT`

### 2. JSON Key Convention

**Rule:** `snake_case` keys in all JSON parameter strings. This matches ABAP field naming convention and avoids transformation overhead.

```abap
" ✅ CORRECT
" {"fiscal_year": "2026", "company_code": "1000", "period": "001"}

" ❌ WRONG
" {"fiscalYear": "2026", "companyCode": "1000"}
" {"FiscalYear": "2026", "CompanyCode": "1000"}
```

**XCO transformation:** When deserializing JSON to ABAP structures, use `xco_cp_json=>transformation->pascal_case_to_underscore` only if the source JSON uses camelCase (external input). Internal orchestrator JSON always uses snake_case — no transformation needed.

### 3. Status Domain Values

**Domain:** `ZEN_ORCH_STATUS` (CHAR 1)

| Status | Value | Terminal? | Constant Name |
|---|---|---|---|
| PENDING | `P` | No | `gc_status-pending` |
| RUNNING | `R` | No | `gc_status-running` |
| COMPLETED | `C` | Yes | `gc_status-completed` |
| FAILED | `F` | No (restartable) | `gc_status-failed` |
| CANCELLED | `X` | Yes | `gc_status-cancelled` |
| PAUSED | `B` | No (breakpoint) | `gc_status-paused` |

**Constants pattern** (on `ZCL_EN_ORCH_ENGINE` or dedicated constants class):

```abap
CONSTANTS: BEGIN OF gc_status,
             pending   TYPE zen_orch_status VALUE 'P',
             running   TYPE zen_orch_status VALUE 'R',
             completed TYPE zen_orch_status VALUE 'C',
             failed    TYPE zen_orch_status VALUE 'F',
             cancelled TYPE zen_orch_status VALUE 'X',
             paused    TYPE zen_orch_status VALUE 'B',
           END OF gc_status.
```

**Note:** `B` for PAUSED matches the existing planner convention (`BREAKPOINT = 'B'`). `P` for PENDING is new (planner uses `N` for NEW).

### 4. Score Element TYPE Domain Values

**Domain:** `ZEN_ORCH_ELEM_TYPE` (CHAR 12)

| Type | Value | Purpose |
|---|---|---|
| Step | `STEP` | Work unit dispatch |
| Loop start | `LOOP` | Iteration boundary open |
| Loop end | `END_LOOP` | Iteration boundary close |
| Gate | `GATE` | Wait-for-all in RefID group |
| Prerequisite gate | `PREREQ_GATE` | BSR query (Phase 2) |

### 5. Exception Textid Naming

**Pattern:** `<AREA>_<CONDITION>` in uppercase. Short, descriptive.

```abap
CLASS zcx_en_orch_error DEFINITION
  INHERITING FROM cx_static_check
  CREATE PUBLIC.

  CONSTANTS:
    BEGIN OF engine_sweep_failed,
      msgid TYPE symsgid VALUE 'ZEN_ORCH',
      msgno TYPE symsgno VALUE '001',
      attr1 TYPE scx_attrname VALUE 'MV_PERF_UUID',
      attr2 TYPE scx_attrname VALUE '',
      attr3 TYPE scx_attrname VALUE '',
      attr4 TYPE scx_attrname VALUE '',
    END OF engine_sweep_failed,
    BEGIN OF adapter_start_failed,
      msgid TYPE symsgid VALUE 'ZEN_ORCH',
      msgno TYPE symsgno VALUE '002',
      attr1 TYPE scx_attrname VALUE 'MV_ADAPTER_TYPE',
      attr2 TYPE scx_attrname VALUE 'MV_PERF_UUID',
      attr3 TYPE scx_attrname VALUE '',
      attr4 TYPE scx_attrname VALUE '',
    END OF adapter_start_failed,
    BEGIN OF score_not_found,
      msgid TYPE symsgid VALUE 'ZEN_ORCH',
      msgno TYPE symsgno VALUE '003',
      attr1 TYPE scx_attrname VALUE 'MV_SCORE_ID',
      attr2 TYPE scx_attrname VALUE '',
      attr3 TYPE scx_attrname VALUE '',
      attr4 TYPE scx_attrname VALUE '',
    END OF score_not_found,
    BEGIN OF performance_collision,
      msgid TYPE symsgid VALUE 'ZEN_ORCH',
      msgno TYPE symsgno VALUE '004',
      attr1 TYPE scx_attrname VALUE 'MV_SCORE_ID',
      attr2 TYPE scx_attrname VALUE 'MV_PERF_UUID',
      attr3 TYPE scx_attrname VALUE '',
      attr4 TYPE scx_attrname VALUE '',
    END OF performance_collision,
    BEGIN OF invalid_score_structure,
      msgid TYPE symsgid VALUE 'ZEN_ORCH',
      msgno TYPE symsgno VALUE '005',
      attr1 TYPE scx_attrname VALUE 'MV_SCORE_ID',
      attr2 TYPE scx_attrname VALUE 'MV_DETAIL',
      attr3 TYPE scx_attrname VALUE '',
      attr4 TYPE scx_attrname VALUE '',
    END OF invalid_score_structure,
    BEGIN OF step_restart_failed,
      msgid TYPE symsgid VALUE 'ZEN_ORCH',
      msgno TYPE symsgno VALUE '006',
      attr1 TYPE scx_attrname VALUE 'MV_PERF_UUID',
      attr2 TYPE scx_attrname VALUE 'MV_SCORE_SEQ',
      attr3 TYPE scx_attrname VALUE '',
      attr4 TYPE scx_attrname VALUE '',
    END OF step_restart_failed.

  DATA mv_perf_uuid    TYPE zen_orch_perf_uuid.
  DATA mv_score_id     TYPE zen_orch_score_id.
  DATA mv_adapter_type TYPE zen_orch_adapter_type.
  DATA mv_score_seq    TYPE zen_orch_score_seq.
  DATA mv_detail       TYPE string.
```

### 6. Engine Method Structure

**Public action methods** (called by APJ job or test programs):

| Method | Purpose |
|---|---|
| `sweep_all` | Process all active performances (APJ entry point) |
| `create_performance` | Create new performance from score |
| `cancel_performance` | Cancel an active performance |
| `restart_performance` | Restart from failed step |
| `resume_performance` | Resume from PAUSED (breakpoint) |

**Private helper methods** (engine internals):

| Method | Purpose |
|---|---|
| `advance_performance` | Run-until-blocked for one performance |
| `dispatch_step` | Start one work unit via adapter |
| `poll_step_status` | Check async step via `get_status()` |
| `evaluate_gate` | Check if gate's RefID group is all COMPLETED |
| `advance_loop` | Increment loop iteration or exit |
| `resolve_params` | Three-level JSON merge + placeholder substitution |
| `snapshot_score` | Copy score rows into performance steps |

**Rule:** Public methods handle LUW (COMMIT). Private methods never commit.

### 7. Adapter Handle Format

**Rule:** Adapter handles are **opaque strings** (max 255 chars). The orchestrator stores them in `WORK_UNIT_HANDLE` but never parses, validates, or interprets them. Only the issuing adapter understands the handle format.

```abap
" Planner adapter might return:
"   "ZFI_PROCESS:ALLOCATIONS:550e8400-e29b-41d4-a716"
" Workflow adapter might return:
"   "WF:APPROVAL:00000001234"
" The orchestrator treats both identically — just a string.
```

**Data element:** `ZEN_ORCH_WU_HANDLE` — CHAR(255), no domain (free-form).

### 8. Table Key Design

**Rule:** Each table uses the key design appropriate to its purpose:

| Table | Primary Key | Rationale |
|---|---|---|
| `ZEN_ORCH_SCORE` | `MANDT, SCORE_ID` | Natural business key — scores are referenced by ID |
| `ZEN_ORCH_SCORE_STEP` | `MANDT, SCORE_ID, SCORE_SEQ` | Natural composite — sequence within score |
| `ZEN_ORCH_PERF` | `MANDT, PERF_UUID` | Surrogate UUID — performances are runtime instances |
| `ZEN_ORCH_PERF_STEP` | `MANDT, PERF_UUID, SCORE_SEQ, LOOP_ITERATION` | Surrogate + natural composite — unique step within performance |
| `ZEN_ORCH_ADAPTER_REG` | `MANDT, ADAPTER_TYPE` | Natural key — adapter types are config entries |
| `ZEN_ORCH_SCHEDULE` | `MANDT, SCHEDULE_ID` | Natural business key |

**No surrogate keys for config tables** (score, adapter registry, schedule). **UUID for runtime tables** (performance header).

### 9. COMMIT Placement

**Rule:** `COMMIT WORK` happens in `sweep_all` after each `advance_performance` call returns. Never inside `advance_performance` or its helpers.

```abap
METHOD sweep_all.
  DATA(lt_active) = get_active_performances( ).
  LOOP AT lt_active INTO DATA(ls_perf).
    TRY.
        advance_performance( ls_perf-perf_uuid ).
      CATCH zcx_en_orch_error INTO DATA(lx).
        mark_failed(
          iv_perf_uuid = ls_perf-perf_uuid
          ix_error     = lx
        ).
    ENDTRY.
    COMMIT WORK.  " one LUW per performance
  ENDLOOP.
ENDMETHOD.
```

### Enforcement Guidelines

**All AI agents implementing ZEN_ORCH code MUST:**

1. Use `ZEN_ORCH_*` / `ZCL_EN_ORCH_*` / `ZIF_EN_ORCH_*` naming — never `ZFI_PROCESS_*` or `ZFI_ORCH_*`
2. Raise `ZCX_EN_ORCH_ERROR` — never `ZCX_FI_PROCESS_ERROR` — in orchestrator code
3. Use `snake_case` JSON keys in all parameter strings
4. Use the exact status char values defined in the `ZEN_ORCH_STATUS` domain
5. Follow the constants structure pattern (`gc_status-pending`, not `gc_pending`)
6. Keep `COMMIT WORK` in `sweep_all` only — never in `advance_performance` or deeper
7. Never parse or validate adapter handles — treat as opaque strings
8. Include audit fields on every database table
9. Follow line length ≤120 chars (abaplint enforced)
10. Consult SAP docs (`SAP_Docs_MCP_search`) before implementing any unfamiliar SAP API

## Project Structure & Boundaries

### Requirements to Components Mapping

| Theme / Requirement Area | ABAP Objects |
|---|---|
| **Theme 1: Score Model** | `ZEN_ORCH_SCORE`, `ZEN_ORCH_SCORE_STEP` tables; domains `ZEN_ORCH_ELEM_TYPE`, `ZEN_ORCH_SCORE_ID`; data elements |
| **Theme 2: Adapter Contracts** | `ZIF_EN_ORCH_ADAPTER` interface, `ZEN_ORCH_ADAPTER_REG` table, `ZCL_EN_ORCH_ADAPTER_FACTORY` class |
| **Theme 3: Engine** | `ZCL_EN_ORCH_ENGINE` class (singleton, sweep + run-until-blocked) |
| **Theme 4: Performance State** | `ZEN_ORCH_PERF`, `ZEN_ORCH_PERF_STEP` tables; `ZEN_ORCH_STATUS` domain |
| **Theme 5: BSR (Phase 2)** | `ZIF_EN_ORCH_BSR` interface, BSR table(s) — deferred |
| **Theme 6: Dashboard (Phase 2)** | CDS views, RAP BDEF — deferred |
| **Cross-cutting: Logging** | `ZIF_EN_ORCH_LOGGER`, `ZCL_EN_ORCH_LOGGER` |
| **Cross-cutting: Errors** | `ZCX_EN_ORCH_ERROR`, message class `ZEN_ORCH` |
| **Cross-cutting: Testing** | `ZCL_EN_ORCH_ADAPTER_MOCK`, test program `ZEN_ORCH_TEST` |
| **Cross-cutting: APJ** | `ZCL_EN_ORCH_JOB_SWEEP` (APJ sweep job class) |

### Complete Project Directory Structure

```
cz.en.orch/                              # Git repository root
├── .abaplint.json                       # abaplint config (120-char limit)
├── .gitignore
├── README.md
├── LICENSE
│
└── src/
    └── zen_orch/                         # Package ZEN_ORCH
        │
        ├── .constitution.md              # Synced constitution (project rules)
        │
        │── package.devc.xml              # Package definition
        │
        │── # ─── DOMAINS ──────────────────────────────────
        │── zen_orch_status.doma.xml           # CHAR 1 — P/R/C/F/X/B
        │── zen_orch_elem_type.doma.xml        # CHAR 12 — STEP/LOOP/END_LOOP/GATE/PREREQ_GATE
        │── zen_orch_score_id.doma.xml         # CHAR 30 — score identifier
        │── zen_orch_adapter_type.doma.xml     # CHAR 30 — adapter type key
        │── zen_orch_score_seq.doma.xml        # NUMC 4 — sequence number in score
        │── zen_orch_max_parallel.doma.xml     # INT4 — max parallel dispatch limit
        │
        │── # ─── DATA ELEMENTS ────────────────────────────
        │── zen_orch_de_status.dtel.xml        # Status (domain ZEN_ORCH_STATUS)
        │── zen_orch_de_elem_type.dtel.xml     # Element type (domain ZEN_ORCH_ELEM_TYPE)
        │── zen_orch_de_score_id.dtel.xml      # Score ID (domain ZEN_ORCH_SCORE_ID)
        │── zen_orch_de_adapter_type.dtel.xml  # Adapter type (domain ZEN_ORCH_ADAPTER_TYPE)
        │── zen_orch_de_score_seq.dtel.xml     # Score sequence (domain ZEN_ORCH_SCORE_SEQ)
        │── zen_orch_perf_uuid.dtel.xml        # Performance UUID (RAW 16 / SYSUUID_X16)
        │── zen_orch_wu_handle.dtel.xml        # Work unit handle (CHAR 255, no domain)
        │── zen_orch_de_ref_id.dtel.xml        # RefID grouping key (CHAR 20)
        │── zen_orch_de_loop_iter.dtel.xml     # Loop iteration counter (INT4)
        │── zen_orch_de_params_json.dtel.xml   # JSON parameter string (STRING)
        │── zen_orch_de_max_par.dtel.xml       # Max parallel (domain ZEN_ORCH_MAX_PARALLEL)
        │── zen_orch_de_impl_class.dtel.xml    # Implementing class name (CHAR 30)
        │
        │── # ─── STRUCTURES ───────────────────────────────
        │── zen_orch_s_start_result.tabl.xml   # Adapter start() return: handle + status
        │
        │── # ─── TABLE TYPES ──────────────────────────────
        │── zen_orch_tt_score_step.ttyp.xml    # TABLE OF zen_orch_score_step
        │── zen_orch_tt_perf_step.ttyp.xml     # TABLE OF zen_orch_perf_step
        │
        │── # ─── DATABASE TABLES ──────────────────────────
        │── zen_orch_score.tabl.xml            # Score definition header
        │── zen_orch_score_step.tabl.xml       # Score step rows (flat, TYPE+RefID)
        │── zen_orch_perf.tabl.xml             # Performance header (UUID key)
        │── zen_orch_perf_step.tabl.xml        # Performance step rows (snapshot + runtime state)
        │── zen_orch_adapter_reg.tabl.xml      # Adapter type registry (config)
        │── zen_orch_schedule.tabl.xml         # Periodic schedule config
        │
        │── # ─── MESSAGE CLASS ────────────────────────────
        │── zen_orch.msag.xml                  # T100 messages (001-099)
        │
        │── # ─── EXCEPTION CLASS ──────────────────────────
        │── zcx_en_orch_error.clas.abap        # Exception class definition
        │── zcx_en_orch_error.clas.xml         # Exception class metadata
        │
        │── # ─── INTERFACES ───────────────────────────────
        │── zif_en_orch_adapter.intf.abap      # 6-method adapter contract
        │── zif_en_orch_adapter.intf.xml
        │── zif_en_orch_logger.intf.abap       # Logger contract
        │── zif_en_orch_logger.intf.xml
        │
        │── # ─── CLASSES ──────────────────────────────────
        │── zcl_en_orch_engine.clas.abap       # Singleton engine (sweep + run-until-blocked)
        │── zcl_en_orch_engine.clas.xml
        │── zcl_en_orch_engine.clas.testclasses.abap  # ABAP Unit tests
        │── zcl_en_orch_adapter_factory.clas.abap     # Registry-based adapter factory
        │── zcl_en_orch_adapter_factory.clas.xml
        │── zcl_en_orch_logger.clas.abap       # BAL logger implementation
        │── zcl_en_orch_logger.clas.xml
        │── zcl_en_orch_adapter_mock.clas.abap # Mock adapter for Phase 1 testing
        │── zcl_en_orch_adapter_mock.clas.xml
        │── zcl_en_orch_job_sweep.clas.abap    # APJ sweep job class
        │── zcl_en_orch_job_sweep.clas.xml
        │
        │── # ─── TEST PROGRAM ─────────────────────────────
        │── zen_orch_test.prog.abap            # Integration test program
        │── zen_orch_test.prog.xml
        │
        │── # ─── BAL OBJECT (SLG1) ───────────────────────
        │── zen_orch.sbal.xml                  # BAL application log object
        │
        └── # ─── PHASE 2 (deferred, placeholders) ────────
            # zif_en_orch_bsr.intf.*           # BSR interface
            # zcl_en_orch_bsr.clas.*           # BSR implementation
            # zen_orch_bsr.tabl.*              # BSR persistence table
            # CDS views for dashboard
            # RAP BDEF + behavior pool
```

### Architectural Boundaries

**Package Boundary — `ZEN_ORCH`:**
- Self-contained: zero compile-time dependency on `ZFI_PROCESS` or `ZFI_ALLOC_PROCESS`
- All objects prefixed `ZEN_ORCH_*` / `ZCL_EN_ORCH_*` / `ZIF_EN_ORCH_*`
- Only exports: `ZIF_EN_ORCH_ADAPTER` interface (implemented by adapters in other packages)

**Adapter Boundary:**
- Adapters implement `ZIF_EN_ORCH_ADAPTER` but live OUTSIDE `ZEN_ORCH` package
- Planner adapter → lives in `ZFI_ALLOC_PROCESS` (ovysledovka repo)
- Future workflow adapter → lives in its own package
- Dependency direction: adapter package → `ZEN_ORCH` (never reverse)

**Data Boundary:**
- Engine reads/writes only `ZEN_ORCH_*` tables
- Engine never reads `ZFI_PROC_INST`, `ZFI_ALLOC_STATE`, or any planner tables
- Adapters bridge the data gap: they read their own framework tables and translate to orchestrator status

**LUW Boundary:**
- `COMMIT WORK` only in `ZCL_EN_ORCH_ENGINE=>sweep_all`
- One commit per performance advancement
- Adapters must not issue COMMIT — engine owns the LUW

### Integration Points

**Internal Communication (within ZEN_ORCH):**

```
ZCL_EN_ORCH_JOB_SWEEP
  └──▶ ZCL_EN_ORCH_ENGINE=>get_instance( )
         ├──▶ sweep_all( )
         │     └──▶ advance_performance( )
         │           ├──▶ ZCL_EN_ORCH_ADAPTER_FACTORY=>create( )
         │           │     └──▶ ZIF_EN_ORCH_ADAPTER (dispatch/poll)
         │           └──▶ ZIF_EN_ORCH_LOGGER (log progress)
         └──▶ check_schedules( )  " create due performances
```

**External Integration (cross-package, Phase 2):**

```
ZFI_ALLOC_PROCESS package (ovysledovka repo):
  ZCL_FI_ALLOC_ORCH_ADAPTER
    IMPLEMENTS ZIF_EN_ORCH_ADAPTER
    USES ZCL_FI_PROCESS_MANAGER  " planner framework
```

**Data Flow:**

```
Score tables (config)
  ──snapshot──▶ Performance step table (runtime)
                  ──adapter start()──▶ External framework
                  ◀──adapter get_status()── External framework
                  ──COMMIT──▶ DB persistence
```

### Activation Order (Transport/Build Sequence)

| Order | Object Type | Objects |
|---|---|---|
| 1 | Domains | `ZEN_ORCH_STATUS`, `ZEN_ORCH_ELEM_TYPE`, `ZEN_ORCH_SCORE_ID`, `ZEN_ORCH_ADAPTER_TYPE`, `ZEN_ORCH_SCORE_SEQ`, `ZEN_ORCH_MAX_PARALLEL` |
| 2 | Data elements | All `ZEN_ORCH_DE_*` + `ZEN_ORCH_PERF_UUID` + `ZEN_ORCH_WU_HANDLE` |
| 3 | Structures | `ZEN_ORCH_S_START_RESULT` |
| 4 | Table types | `ZEN_ORCH_TT_SCORE_STEP`, `ZEN_ORCH_TT_PERF_STEP` |
| 5 | Database tables | `ZEN_ORCH_SCORE`, `ZEN_ORCH_SCORE_STEP`, `ZEN_ORCH_PERF`, `ZEN_ORCH_PERF_STEP`, `ZEN_ORCH_ADAPTER_REG`, `ZEN_ORCH_SCHEDULE` |
| 6 | Message class | `ZEN_ORCH` |
| 7 | Exception class | `ZCX_EN_ORCH_ERROR` |
| 8 | Interfaces | `ZIF_EN_ORCH_ADAPTER`, `ZIF_EN_ORCH_LOGGER` |
| 9 | Classes | `ZCL_EN_ORCH_LOGGER`, `ZCL_EN_ORCH_ADAPTER_FACTORY`, `ZCL_EN_ORCH_ADAPTER_MOCK` |
| 10 | Engine class | `ZCL_EN_ORCH_ENGINE` (depends on all above) |
| 11 | APJ job class | `ZCL_EN_ORCH_JOB_SWEEP` (depends on engine) |
| 12 | Test program | `ZEN_ORCH_TEST` |

### Development Workflow

**Repository setup:**
1. Create git repo `cz.en.orch` on GitHub
2. Create SAP package `ZEN_ORCH` (development class)
3. Link via abapGit (XML serialization format)
4. Copy `.abaplint.json` from planner repo (same 120-char rule)
5. Sync constitution via `sync-constitution.sh` (after adding new repo to registry)

**Build verification:**
- abaplint CI checks on every push
- ABAP Unit tests in `zcl_en_orch_engine.clas.testclasses.abap`
- Integration test via `ZEN_ORCH_TEST` program (manual, uses mock adapter)

## Architecture Validation Results

### Coherence Validation

**Decision Compatibility:**

| Check | Result | Notes |
|---|---|---|
| D1 (new repo) + D2 (zero dependency) | PASS | New repo enforces the isolation; no accidental USE of planner package |
| D3 (singleton) + D5 (COMMIT per perf) | PASS | Singleton engine controls LUW; no competing commit sources |
| D4 (XCO_CP_JSON) + ABAP 7.58 on-prem | PASS | Verified in Step 4 — XCO is in SAP_BASIS, available on 7.58 |
| D6 (own logger) + D2 (zero dependency) | PASS | Own ZIF_EN_ORCH_LOGGER avoids compile-time ref to ZIF_FI_PROCESS_LOGGER |
| Status model (P/R/C/F/X/B) + Fail-stop | PASS | FAILED is non-terminal (restartable); consistent with fail-stop + explicit recovery |
| Score snapshot + Flat score model | PASS | Flat rows copy cleanly into perf_step table; no tree traversal needed |
| Resilient polling + Fail-stop | PASS | Resilient polling applies to get_status() transient errors only; adapter start() failures are still fail-stop |

**Pattern Consistency:**

| Check | Result |
|---|---|
| Naming: all objects follow `ZEN_ORCH_*` / `ZCL_EN_ORCH_*` / `ZIF_EN_ORCH_*` | PASS |
| Constants: `gc_status-pending` pattern used throughout | PASS |
| JSON: snake_case convention documented with examples | PASS |
| COMMIT: explicitly placed in `sweep_all` only, with code example | PASS |
| Factory: adapter factory + registry table pattern matches Constitution Principle IV | PASS |

**Structure Alignment:**

| Check | Result |
|---|---|
| Directory tree matches all objects referenced in decisions | PASS |
| Activation order matches dependency chain | PASS |
| Phase 2 objects clearly marked as deferred | PASS |

### Requirements Coverage

**Functional Requirements (all 10 from Step 2):**

| FR | Covered By | Status |
|---|---|---|
| FR1: Declarative score | `ZEN_ORCH_SCORE` + `ZEN_ORCH_SCORE_STEP` tables, `ZEN_ORCH_ELEM_TYPE` domain | COVERED |
| FR2: 6-method adapter | `ZIF_EN_ORCH_ADAPTER` interface definition | COVERED |
| FR3: Performance instances | `ZEN_ORCH_PERF` + `ZEN_ORCH_PERF_STEP` tables, UUID key | COVERED |
| FR4: Stateless engine + sweep | `ZCL_EN_ORCH_ENGINE` singleton, `sweep_all` + `advance_performance` | COVERED |
| FR5: BSR | Deferred to Phase 2 — `ZIF_EN_ORCH_BSR` placeholder noted | DEFERRED (by design) |
| FR6: Three-level dashboard | Deferred to Phase 2 — CDS/RAP placeholders noted | DEFERRED (by design) |
| FR7: Periodic scheduling | `ZEN_ORCH_SCHEDULE` table + `check_schedules()` in engine | COVERED |
| FR8: Breakpoints | PAUSED status (`B`) + `resume_performance` method | COVERED |
| FR9: JSON parameter inheritance | `resolve_params` method, XCO_CP_JSON patterns, three-level merge | COVERED |
| FR10: Adapter registry + factory | `ZEN_ORCH_ADAPTER_REG` table + `ZCL_EN_ORCH_ADAPTER_FACTORY` | COVERED |

**Non-Functional Requirements (all 6):**

| NFR | Architectural Support | Status |
|---|---|---|
| Idempotency | Stateless engine reads DB, advances, writes back; no in-memory state | COVERED |
| Concurrency | MAX_PARALLEL on RefID groups; BSR collision (Phase 2) | COVERED |
| Resilience | `get_status()` exception = skip/retry; adapter errors at boundary | COVERED |
| Auditability | Fail-stop + BAL logging + explicit recovery via restart | COVERED |
| Extensibility | Adapter registry row = new framework; zero orchestrator code change | COVERED |
| Isolation | Copy-on-start snapshot into perf_step table | COVERED |

### Implementation Readiness

**Decision Completeness:**

| Aspect | Assessment |
|---|---|
| All 6 critical decisions documented with rationale | PASS |
| 8 important decisions confirmed from brainstorming | PASS |
| 4 deferred decisions clearly marked with deferral rationale | PASS |
| XCO_CP_JSON code examples provided | PASS |
| Exception textid examples with full constant structure | PASS |
| Engine method structure with public/private split | PASS |

**Structure Completeness:**

| Aspect | Assessment |
|---|---|
| Complete file listing (~45 objects) | PASS |
| Activation order (12 layers) | PASS |
| Repository setup steps | PASS |
| Phase 2 placeholders noted | PASS |

**Pattern Completeness:**

| Aspect | Assessment |
|---|---|
| 9 conflict areas identified with rules + examples | PASS |
| 10 enforcement guidelines for agents | PASS |
| Anti-patterns documented for each naming rule | PASS |

### Gap Analysis

**Critical Gaps: NONE**

**Important Gaps (2):**

1. **Score description field** — `ZEN_ORCH_SCORE` header table needs a `DESCRIPTION` (CHAR 60) field for human-readable display in dashboard and score management. Standard pattern — agents can infer this from constitution DB conventions.

2. **Start result structure fields** — `ZEN_ORCH_S_START_RESULT` fields: `WORK_UNIT_HANDLE TYPE zen_orch_wu_handle` + `STATUS TYPE zen_orch_de_status`. Consistent with naming patterns already documented.

**Nice-to-Have Gaps (1):**

1. **T100 message number ranges** — Suggested: 001-019 engine lifecycle, 020-039 adapter dispatch, 040-059 score/performance creation, 060-079 validation errors.

### Architecture Completeness Checklist

**Requirements Analysis**
- [x] Project context thoroughly analyzed (brainstorming + project-context.md)
- [x] Scale and complexity assessed (~15 DDIC, ~8 classes, 2 phases)
- [x] Technical constraints identified (10 constraints)
- [x] Cross-cutting concerns mapped (7 concerns)

**Architectural Decisions**
- [x] 6 critical decisions documented with rationale
- [x] Technology stack fully specified (ABAP 7.58, XCO_CP_JSON, SAP HANA, BAL)
- [x] Integration patterns defined (adapter boundary, one-way dependency)
- [x] LUW/performance considerations addressed (COMMIT per performance)

**Implementation Patterns**
- [x] Naming conventions established (9 pattern areas)
- [x] Structure patterns defined (table keys, method structure)
- [x] Communication patterns specified (adapter interface, JSON contract)
- [x] Process patterns documented (COMMIT placement, error propagation)

**Project Structure**
- [x] Complete directory structure defined (~45 objects)
- [x] Component boundaries established (package, adapter, data, LUW)
- [x] Integration points mapped (internal call graph, external adapter bridge)
- [x] Requirements to structure mapping complete (6 themes + 4 cross-cutting)

### Architecture Readiness Assessment

**Overall Status:** READY FOR IMPLEMENTATION

**Confidence Level:** HIGH

**Key Strengths:**
- Zero compile-time dependency on planner enforced by separate repository
- Complete DDIC-first object catalog with activation order
- Mock adapter enables Phase 1 testing without any external dependencies
- Musical metaphor provides consistent vocabulary across all documentation
- All 10 FRs have concrete architectural support (2 deferred by design to Phase 2)

**Areas for Future Enhancement:**
- BSR persistence design (Phase 2 — intentionally deferred)
- Dashboard CDS/RAP layer (Phase 2)
- T100 message number range allocation (implementation detail)
- Score header description field (standard addition)
