---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - '_bmad-output/planning-artifacts/architecture.md'
  - '_bmad-output/planning-artifacts/epics.md'
  - '_bmad-output/brainstorming/brainstorming-session-2026-04-01-100000.md'
workflowType: 'epics-and-stories'
project_name: 'ZEN_ORCH — Process Orchestration Engine — Phase 2'
user_name: 'Zdenek'
date: '2026-04-09'
lastStep: 4
status: 'complete'
target_repository: 'cz.en.orch'
---

# ZEN_ORCH Phase 2 — Business State Registry (BSR)

## Overview

Phase 1 delivered the core orchestration engine: score/performance model, adapter framework, APJ sweep, breakpoints, and full unit test coverage. All Phase 1 unit tests are GREEN.

Phase 2 adds the **Business State Registry (BSR)** — a cross-performance coordination layer that was intentionally deferred from Phase 1 until real adapter integration experience was available (architecture.md §Deferred Decisions).

**Target Repository:** `cz.en.orch` (`/Users/smolik/DEV/cz.en.orch`)
**Package:** `ZEN_ORCH`
**Phase Scope:** Epic 7 (BSR). Dashboard (FR6) deferred to Phase 3.

---

## Phase 2 Requirements

### Functional Requirements Addressed

| FR | Description | Status |
|----|-------------|--------|
| FR5 | Business State Registry — strategic overview, prerequisite resolution, collision locking | **THIS PHASE** |
| FR6 | Three-level observability dashboard | Deferred to Phase 3 |

### BSR Design Decisions

Based on architecture.md, brainstorming (#21, #25, #26), and Phase 2 scoping:

| Decision | Choice | Rationale |
|----------|--------|-----------|
| BSR persistence | `ZEN_ORCH_BSR` table in `cz.en.orch` | ZEN_ORCH owns its full persistence; no external dependency |
| Business key format | Opaque string (CHAR 255) — domain encodes its own key | Matches adapter handle approach; orchestrator stays domain-ignorant |
| Collision granularity (Phase 2) | Score + params hash at `create_performance` time (brainstorming #21) | Simpler and sufficient for the allocation use case; step-level BSR collision deferred |
| BSR interface | `ZIF_EN_ORCH_BSR` — single interface; no separate domain state service layer | Phase 2 scope; ZIF_FI_ORCH_STATE_SERVICE is Phase 3 |
| Status resolution | BSR holds perf_uuid references; status always resolved from live `ZEN_ORCH_PERF` | No stale copies; single source of truth |
| PREREQ_GATE | New engine method `evaluate_prereq_gate` calling `ZIF_EN_ORCH_BSR=>check_prerequisite` | Completes the score element type model from Phase 1 |

---

## Epic 7: Business State Registry (BSR)

The orchestrator has a persistent cross-performance coordination layer. Duplicate performance creation is blocked. PREREQ_GATE score elements can check business conditions across all performances. A strategic overview of all active performances by business key is available through the BSR interface.

**Target repository:** `cz.en.orch`
**Constitution principles:** I (DDIC-First), II (SAP Standards), IV (Factory Pattern), V (Error Handling)
**Depends on:** Phase 1 (Epics 1–6, all done)

### Dependency Flow

```
E7.1 (BSR DDIC Foundation)
 └──▶ E7.2 (ZIF_EN_ORCH_BSR Interface)
       └──▶ E7.3 (ZCL_EN_ORCH_BSR Implementation)
             └──▶ E7.4 (Engine Integration — collision + registration)
                   └──▶ E7.5 (PREREQ_GATE Engine Support)
                         └──▶ E7.6 (Unit Tests)
```

---

### Story 7.1: BSR DDIC Foundation

As a developer,
I want the BSR persistence table and supporting DDIC types defined,
So that the engine can store and query cross-performance business key registrations.

**target_repository:** `cz.en.orch`
**depends_on:** []
**constitution_principles:**
  - Principle I — DDIC-First
  - Principle II — SAP Standards

**Acceptance Criteria:**

**Given** the ZEN_ORCH package and existing DDIC types from Phase 1 exist
**When** the BSR DDIC objects are activated
**Then** a new domain `ZEN_ORCH_BSR_KEY` (CHAR 255, no fixed values — opaque business key) exists
**And** a new data element `ZEN_ORCH_DE_BSR_KEY` referencing `ZEN_ORCH_BSR_KEY` exists with field label "BSR Business Key"
**And** table `ZEN_ORCH_BSR` exists with:
  - Primary key: `MANDT`, `BSR_KEY`
  - Fields: `PERF_UUID TYPE zen_orch_perf_uuid`, `SCORE_ID TYPE zen_orch_de_score_id`, `PARAMS_HASH TYPE char64` (SHA-256 hex of params JSON), `REGISTERED_AT TYPE timestampl`, `REGISTERED_BY TYPE syuname`
  - Delivery class: A (application data)
  - Client-dependent (MANDT first key field)
**And** a new data element `ZEN_ORCH_DE_PARAMS_HASH` (CHAR 64) exists for the SHA-256 hash field
**And** all objects activate without errors

**Notes:**
- `BSR_KEY` is an opaque string — the engine never interprets it; callers compose it (e.g., `"ALLOC:1000:2026:001"`)
- `PARAMS_HASH` enables fast duplicate-run detection without full JSON comparison
- No table type needed for BSR table — engine queries it directly by key

---

### Story 7.2: ZIF_EN_ORCH_BSR Interface

As a developer,
I want the BSR interface defined with its full contract,
So that the engine depends on an abstraction and test code can inject a mock BSR.

**target_repository:** `cz.en.orch`
**depends_on:** ["7.1"]
**constitution_principles:**
  - Principle II — SAP Standards
  - Principle IV — Factory Pattern (interface contract enables mock injection)

**Acceptance Criteria:**

**Given** the BSR DDIC types from Story 7.1 exist
**When** `ZIF_EN_ORCH_BSR` is activated
**Then** the interface declares exactly 5 methods:

- `register( IMPORTING iv_bsr_key TYPE zen_orch_de_bsr_key iv_perf_uuid TYPE zen_orch_perf_uuid iv_score_id TYPE zen_orch_de_score_id iv_params_hash TYPE zen_orch_de_params_hash RAISING zcx_en_orch_error )`
  — Registers a business key as owned by the given performance. Raises `BSR_KEY_COLLISION` if the key is already held by an active (non-terminal) performance.

- `deregister( IMPORTING iv_bsr_key TYPE zen_orch_de_bsr_key iv_perf_uuid TYPE zen_orch_perf_uuid RAISING zcx_en_orch_error )`
  — Releases a BSR entry when a performance reaches a terminal state. No-op if entry not found (idempotent).

- `check_collision( IMPORTING iv_score_id TYPE zen_orch_de_score_id iv_params_hash TYPE zen_orch_de_params_hash RETURNING VALUE(rv_colliding_uuid) TYPE zen_orch_perf_uuid )`
  — Returns the UUID of an active performance with the same score+params hash, or initial value if none exists. Does NOT raise — caller decides what to do.

- `check_prerequisite( IMPORTING iv_bsr_key TYPE zen_orch_de_bsr_key RETURNING VALUE(rv_status) TYPE zen_orch_de_status )`
  — Returns the current status of the performance registered under the given BSR key, resolved from live `ZEN_ORCH_PERF`. Returns `gc_status-pending` if key not found (not yet registered = not yet started).

- `get_overview( RETURNING VALUE(rt_overview) TYPE zen_orch_tt_bsr_overview )`
  — Returns all current BSR entries with resolved live statuses (for dashboard/reporting use).

**And** a new table type `ZEN_ORCH_TT_BSR_OVERVIEW` exists as TABLE OF a flat structure `ZEN_ORCH_S_BSR_OVERVIEW` with fields: `BSR_KEY`, `PERF_UUID`, `SCORE_ID`, `STATUS`, `REGISTERED_AT`
**And** all methods carry ABAP-Doc comments with `@parameter` and `@raising` annotations
**And** the interface has no reference to ZFI_PROCESS objects (zero dependency)
**And** the interface activates without errors

---

### Story 7.3: ZCL_EN_ORCH_BSR Implementation

As an engine,
I want a concrete BSR implementation backed by `ZEN_ORCH_BSR`,
So that performance registrations, collision checks, and prerequisite queries work against the real database.

**target_repository:** `cz.en.orch`
**depends_on:** ["7.1", "7.2"]
**constitution_principles:**
  - Principle I — DDIC-First
  - Principle II — SAP Standards
  - Principle IV — Factory Pattern

**Acceptance Criteria:**

**Given** `ZIF_EN_ORCH_BSR` and `ZEN_ORCH_BSR` table exist
**When** `ZCL_EN_ORCH_BSR` is activated
**Then** it implements all 5 methods of `ZIF_EN_ORCH_BSR`

**Given** `register` is called with a key not in `ZEN_ORCH_BSR`
**When** the method executes
**Then** a row is inserted into `ZEN_ORCH_BSR` with the given key, uuid, score_id, params_hash, and current timestamp/user
**And** no COMMIT is issued (engine owns the LUW)

**Given** `register` is called and `ZEN_ORCH_BSR` already has an entry for the key where the referenced performance is non-terminal (status P/R/B)
**When** the method executes
**Then** `ZCX_EN_ORCH_ERROR` is raised with textid `BSR_KEY_COLLISION` and `MV_PERF_UUID` set to the colliding UUID

**Given** `register` is called and the existing entry's performance is terminal (C/F/X)
**When** the method executes
**Then** the stale entry is overwritten with the new registration (stale terminal registrations are auto-cleaned on next use)

**Given** `check_collision` is called with a score_id + params_hash
**When** the method executes
**Then** it queries `ZEN_ORCH_BSR` for entries with matching SCORE_ID + PARAMS_HASH
**And** for each match, resolves the live status from `ZEN_ORCH_PERF`
**And** returns the first UUID where status is non-terminal, or initial value if none active

**Given** `check_prerequisite` is called with a BSR key
**When** the method executes
**Then** it reads the entry from `ZEN_ORCH_BSR` and resolves the live status from `ZEN_ORCH_PERF`
**And** returns the live status, or `gc_status-pending` if the key is not registered

**Given** `deregister` is called
**When** the method executes
**Then** the row matching `BSR_KEY` + `PERF_UUID` is deleted from `ZEN_ORCH_BSR` (no-op if not found)
**And** no COMMIT is issued

**And** `ZCL_EN_ORCH_BSR` has a factory method `create( )` returning `REF TO zif_en_orch_bsr`
**And** the class activates without errors

---

### Story 7.4: Engine Integration — Collision Detection and BSR Lifecycle

As an engine,
I want `create_performance` to block duplicate runs and `sweep_all` to maintain BSR registrations automatically,
So that no two performances can run the same score+params simultaneously, and BSR entries are kept clean without manual intervention.

**target_repository:** `cz.en.orch`
**depends_on:** ["7.3"]
**constitution_principles:**
  - Principle I — DDIC-First
  - Principle V — Error Handling

**Acceptance Criteria:**

**Given** `create_performance` is called with a score_id + params_json
**When** a non-terminal performance with the same score_id + SHA-256 hash of params_json already exists
**Then** `ZCX_EN_ORCH_ERROR` is raised with textid `PERFORMANCE_COLLISION` and `MV_PERF_UUID` set to the colliding UUID
**And** no new performance row is inserted

**Given** `create_performance` succeeds and creates a new performance
**When** the method returns the new perf_uuid
**Then** a BSR entry is registered via `ZIF_EN_ORCH_BSR=>register` using `SCORE_ID + ':' + PERF_UUID` as the BSR key and the params hash

**Given** `sweep_all` processes a performance that has just reached a terminal state (C/F/X)
**When** the performance status is written as terminal
**Then** `ZIF_EN_ORCH_BSR=>deregister` is called for that performance's BSR key
**And** the deregister call happens before `COMMIT WORK` for that performance

**Given** the BSR instance is injected into `ZCL_EN_ORCH_ENGINE`
**When** the engine is constructed via `get_instance( )`
**Then** the engine holds a reference to `ZIF_EN_ORCH_BSR` (created via `ZCL_EN_ORCH_BSR=>create( )`)
**And** tests can inject a mock BSR by calling a package-private setter (or via constructor injection pattern consistent with existing engine design)

**Notes:**
- SHA-256 hashing: use `CL_ABAP_HMAC` or `CL_ABAP_MESSAGE_DIGEST` — consult SAP docs before implementing
- BSR key convention for auto-registration: `<SCORE_ID>:<PERF_UUID>` — unique, opaque, never interpreted

---

### Story 7.5: PREREQ_GATE Engine Support

As an engine,
I want to evaluate PREREQ_GATE score elements by querying the BSR,
So that a score can declare cross-performance prerequisites and the engine blocks until they are satisfied.

**target_repository:** `cz.en.orch`
**depends_on:** ["7.4"]
**constitution_principles:**
  - Principle II — SAP Standards
  - Principle V — Error Handling

**Acceptance Criteria:**

**Given** a performance step with `ELEM_TYPE = 'PREREQ_GATE'` and `REF_ID` containing a BSR key
**When** `evaluate_prereq_gate` is called for that step
**Then** `ZIF_EN_ORCH_BSR=>check_prerequisite( iv_bsr_key = ls_step-ref_id )` is called
**And** if the returned status is `gc_status-completed`, the PREREQ_GATE step status is set to `gc_status-completed`
**And** if the returned status is `gc_status-failed` or `gc_status-cancelled`, the PREREQ_GATE step (and its performance) is set to FAILED — prerequisite failed terminally
**And** if the returned status is PENDING, RUNNING, or PAUSED, the PREREQ_GATE step remains PENDING (blocking the run-until-blocked loop — engine yields and retries next sweep)

**Given** `advance_performance` encounters a step with `ELEM_TYPE = 'PREREQ_GATE'` in STATUS = 'P'
**When** the step is processed
**Then** `evaluate_prereq_gate` is called (same routing logic as `evaluate_gate` and `advance_loop`)

**Given** a PREREQ_GATE that has been COMPLETED (prerequisite satisfied)
**When** `advance_performance` processes subsequent steps
**Then** execution continues past the PREREQ_GATE without re-checking (status already C — standard step flow)

**And** `evaluate_prereq_gate` is a new private method on `ZCL_EN_ORCH_ENGINE`
**And** the `ELEM_TYPE` routing in `advance_performance` is updated to route `PREREQ_GATE` to `evaluate_prereq_gate`
**And** the class activates without errors

---

### Story 7.6: Unit Tests for BSR and PREREQ_GATE

As a developer,
I want ABAP Unit tests covering the BSR implementation and PREREQ_GATE engine logic,
So that regressions in collision detection, prerequisite evaluation, and BSR lifecycle are caught automatically.

**target_repository:** `cz.en.orch`
**depends_on:** ["7.3", "7.4", "7.5"]
**constitution_principles:**
  - Principle II — SAP Standards

**Acceptance Criteria:**

**Given** the BSR unit tests exist in the engine test class (`zcl_en_orch_engine.clas.testclasses.abap`) or a new `zcl_en_orch_bsr.clas.testclasses.abap`
**When** ABAP Unit tests are executed
**Then** the following scenarios pass:

**BSR Implementation (ZCL_EN_ORCH_BSR):**
- `register` with new key → row inserted in BSR table
- `register` with key held by active performance → raises `BSR_KEY_COLLISION`
- `register` with key held by terminal performance → overwrites stale entry (no error)
- `deregister` with existing entry → row deleted
- `deregister` with non-existent entry → no error (idempotent)
- `check_collision` with no active match → returns initial UUID
- `check_collision` with active match → returns colliding UUID
- `check_prerequisite` with key not registered → returns PENDING
- `check_prerequisite` with registered completed performance → returns COMPLETED

**Engine + BSR Integration:**
- `create_performance` with duplicate score+params (active performance exists) → raises `PERFORMANCE_COLLISION`
- `create_performance` success → BSR entry registered
- `sweep_all` with performance reaching COMPLETED → BSR entry deregistered before commit

**PREREQ_GATE:**
- PREREQ_GATE step with prerequisite COMPLETED → gate step set to COMPLETED, performance advances
- PREREQ_GATE step with prerequisite RUNNING → gate step stays PENDING (engine yields)
- PREREQ_GATE step with prerequisite FAILED → gate step + performance set to FAILED

**And** all tests use a mock BSR (injected) or test doubles — no real DB dependencies in unit tests where avoidable
**And** all new tests pass alongside existing Phase 1 unit tests

---

## New Exception Textids Required

Story 7.2/7.4 require two new textids in `ZCX_EN_ORCH_ERROR`:

| Textid | Msgno | attr1 | attr2 | Description |
|--------|-------|-------|-------|-------------|
| `BSR_KEY_COLLISION` | 007 | `MV_PERF_UUID` | `MV_SCORE_ID` | BSR key already held by active performance |
| `PERFORMANCE_COLLISION` already exists | 004 | — | — | Already defined in Phase 1 (reuse) |

Only `BSR_KEY_COLLISION` (msgno 007) is new. `PERFORMANCE_COLLISION` (msgno 004) was already defined in Phase 1 but not wired — Story 7.4 wires it.

---

## New DDIC Objects Summary

| Object | Type | Story | Notes |
|--------|------|-------|-------|
| `ZEN_ORCH_BSR_KEY` | Domain CHAR 255 | 7.1 | Opaque business key |
| `ZEN_ORCH_DE_BSR_KEY` | Data element | 7.1 | References `ZEN_ORCH_BSR_KEY` |
| `ZEN_ORCH_DE_PARAMS_HASH` | Data element | 7.1 | CHAR 64 (SHA-256 hex) |
| `ZEN_ORCH_BSR` | Database table | 7.1 | Client-dep, key: MANDT+BSR_KEY |
| `ZEN_ORCH_S_BSR_OVERVIEW` | Structure | 7.2 | Flat overview row |
| `ZEN_ORCH_TT_BSR_OVERVIEW` | Table type | 7.2 | TABLE OF `ZEN_ORCH_S_BSR_OVERVIEW` |

---

## Activation Order (Phase 2 additions)

| Order | Object | Story |
|-------|--------|-------|
| 13 | Domain `ZEN_ORCH_BSR_KEY`, data elements `ZEN_ORCH_DE_BSR_KEY`, `ZEN_ORCH_DE_PARAMS_HASH` | 7.1 |
| 14 | Table `ZEN_ORCH_BSR` | 7.1 |
| 15 | Structure `ZEN_ORCH_S_BSR_OVERVIEW`, table type `ZEN_ORCH_TT_BSR_OVERVIEW` | 7.2 |
| 16 | Interface `ZIF_EN_ORCH_BSR` | 7.2 |
| 17 | Class `ZCL_EN_ORCH_BSR` | 7.3 |
| 18 | `ZCL_EN_ORCH_ENGINE` updated (collision + registration + PREREQ_GATE) | 7.4 + 7.5 |
| 19 | `ZCX_EN_ORCH_ERROR` updated (add msgno 007 BSR_KEY_COLLISION) | 7.4 |
| 20 | Test classes updated | 7.6 |

---

## Phase 3 Scope (Deferred)

- **FR6: Three-level observability dashboard** — CDS views + RAP BDEF
- **ZIF_FI_ORCH_STATE_SERVICE** — domain state service interface for external business state queries
- **Real planner adapter** (`ZCL_FI_ALLOC_ORCH_ADAPTER` in `ovysledovka` repo) — bridges ZEN_ORCH to ZFI_PROCESS
- **Step-level BSR collision** (brainstorming #26) — block dispatch when another performance holds same business key at step granularity
