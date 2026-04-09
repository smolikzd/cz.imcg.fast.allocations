# Story zen-7-2: ZIF_EN_ORCH_BSR Interface

## Story

**Story ID:** zen-7-2
**Epic ID:** zen-7
**Title:** ZIF_EN_ORCH_BSR Interface
**Target Repository:** cz.en.orch
**Depends On:** ["zen-7-1"]
**Status:** review

**Constitution Principles:**
- Principle II — SAP Standards
- Principle IV — Factory Pattern (interface contract enables mock injection)

As a developer,
I want the BSR interface defined with its full contract,
So that the engine depends on an abstraction and test code can inject a mock BSR.

---

## Acceptance Criteria

**Given** the BSR DDIC types from Story 7.1 exist
**When** `ZIF_EN_ORCH_BSR` is activated
**Then** the interface declares exactly 5 methods:

- `register( IMPORTING iv_bsr_key iv_perf_uuid iv_score_id iv_params_hash RAISING zcx_en_orch_error )`
  — Registers a business key as owned by the given performance. Raises `BSR_KEY_COLLISION` if the key is already held by an active (non-terminal) performance.

- `deregister( IMPORTING iv_bsr_key iv_perf_uuid RAISING zcx_en_orch_error )`
  — Releases a BSR entry when a performance reaches a terminal state. No-op if entry not found (idempotent).

- `check_collision( IMPORTING iv_score_id iv_params_hash RETURNING VALUE(rv_colliding_uuid) TYPE zen_orch_perf_uuid )`
  — Returns the UUID of an active performance with the same score+params hash, or initial value if none exists. Does NOT raise.

- `check_prerequisite( IMPORTING iv_bsr_key RETURNING VALUE(rv_status) TYPE zen_orch_de_status )`
  — Returns the current status of the performance registered under the given BSR key. Returns `gc_status-pending` if key not found.

- `get_overview( RETURNING VALUE(rt_overview) TYPE zen_orch_tt_bsr_overview )`
  — Returns all current BSR entries with resolved live statuses.

**And** a new structure `ZEN_ORCH_S_BSR_OVERVIEW` exists with fields: `BSR_KEY`, `PERF_UUID`, `SCORE_ID`, `STATUS`, `REGISTERED_AT`
**And** a new table type `ZEN_ORCH_TT_BSR_OVERVIEW` exists as TABLE OF `ZEN_ORCH_S_BSR_OVERVIEW`
**And** all methods carry ABAP-Doc comments with `@parameter` and `@raising` annotations
**And** the interface has no reference to ZFI_PROCESS objects (zero dependency)
**And** the interface activates without errors
**And** `BSR_KEY_COLLISION` textid is defined in `ZCX_EN_ORCH_ERROR` with msgno 071

---

## Tasks/Subtasks

- [x] Task 1: Create structure `ZEN_ORCH_S_BSR_OVERVIEW`
  - [x] Create `zen_orch_s_bsr_overview.tabl.xml` in `cz.en.orch/src/` (INTTAB, fields: BSR_KEY, PERF_UUID, SCORE_ID, STATUS, REGISTERED_AT)
- [x] Task 2: Create table type `ZEN_ORCH_TT_BSR_OVERVIEW`
  - [x] Create `zen_orch_tt_bsr_overview.ttyp.xml` in `cz.en.orch/src/` (TABLE OF ZEN_ORCH_S_BSR_OVERVIEW)
- [x] Task 3: Create interface `ZIF_EN_ORCH_BSR`
  - [x] Create `zif_en_orch_bsr.intf.xml` (metadata descriptor)
  - [x] Create `zif_en_orch_bsr.intf.abap` (5 methods with full ABAP-Doc)
- [x] Task 4: Add `BSR_KEY_COLLISION` textid to `ZCX_EN_ORCH_ERROR`
  - [x] Add constant `bsr_key_collision` (msgno 071, attr1=MV_PERF_UUID, attr2=MV_SCORE_ID)
  - [x] Add message 071 "BSR key already held by active performance &1 (score: &2)" to `zen_orch.msag.xml`

---

## Dev Notes

### Architecture Context

This story is part of ZEN_ORCH Phase 2 — Business State Registry (BSR). Phase 1 (Epics 1–6) delivered the engine core.

**Repository:** `cz.en.orch` at `/Users/smolik/DEV/cz.en.orch`
**Package:** `ZEN_ORCH`
**abapGit format:** XML serialization (all DDIC objects as `.xml` files in `src/`)

### Interface Design Decisions

- `check_collision` has NO RAISING clause — caller decides what to do with the collision result
- `check_prerequisite` returns `gc_status-pending` (not initial) when key is not registered — this maps to the "not yet started" semantic naturally
- `get_overview` has NO RAISING clause — best-effort; empty table on failure is acceptable
- `register` and `deregister` RAISE because DB errors there are fatal to the engine LUW
- Interface has zero dependency on ZFI_PROCESS — ZEN_ORCH is fully standalone

### New Message Number

- Msgno 007 was already taken (`Performance failed: &1 — &2`)
- Used msgno 071 to start a BSR-related block (071–079 reserved for BSR errors)
- The epics file note "msgno 007 is new" was based on an incomplete view of the message class; 071 is correct

### Activation Order

Per epics-zen-phase2.md §Activation Order:
1. Structure `ZEN_ORCH_S_BSR_OVERVIEW` (step 15)
2. Table type `ZEN_ORCH_TT_BSR_OVERVIEW` (step 15)
3. Interface `ZIF_EN_ORCH_BSR` (step 16)

All DDIC prerequisite types (`ZEN_ORCH_DE_BSR_KEY`, `ZEN_ORCH_PERF_UUID`, `ZEN_ORCH_DE_SCORE_ID`,
`ZEN_ORCH_DE_PARAMS_HASH`, `ZEN_ORCH_DE_STATUS`, `TIMESTAMPL`) already exist from Phase 1 + Story 7.1.

---

## Dev Agent Record

### Implementation Plan

Story delivers:
1. `ZEN_ORCH_S_BSR_OVERVIEW` — flat structure (INTTAB) for overview rows
2. `ZEN_ORCH_TT_BSR_OVERVIEW` — standard table type for `get_overview` return value
3. `ZIF_EN_ORCH_BSR` — interface with 5 methods matching the AC contract exactly
4. `BSR_KEY_COLLISION` constant + message 071 in `ZCX_EN_ORCH_ERROR` and `zen_orch.msag.xml`

No ABAP unit tests required for this story (interface-only — nothing to instantiate and test).
Story 7.6 covers all BSR unit tests once the implementation (7.3) and engine integration (7.4) exist.

### Completion Notes

All 6 files created/modified:
- `zen_orch_s_bsr_overview.tabl.xml` — INTTAB structure, 5 fields (BSR_KEY, PERF_UUID, SCORE_ID, STATUS, REGISTERED_AT)
- `zen_orch_tt_bsr_overview.ttyp.xml` — TABLE OF ZEN_ORCH_S_BSR_OVERVIEW, standard key, no key
- `zif_en_orch_bsr.intf.xml` — Interface metadata (EXPOSURE=2, STATE=1, UNICODE=X)
- `zif_en_orch_bsr.intf.abap` — 5 methods with full ABAP-Doc (@parameter/@raising annotations)
- `zcx_en_orch_error.clas.abap` — Added `bsr_key_collision` constant (msgno 071)
- `zen_orch.msag.xml` — Added message 071 "BSR key already held by active performance &1 (score: &2)"

Message 071 chosen over 007 (as noted in epics) because 007 was already taken by "Performance failed: &1 — &2".

---

## File List

**Repository: cz.en.orch**

- `src/zen_orch_s_bsr_overview.tabl.xml` (new)
- `src/zen_orch_tt_bsr_overview.ttyp.xml` (new)
- `src/zif_en_orch_bsr.intf.xml` (new)
- `src/zif_en_orch_bsr.intf.abap` (new)
- `src/zcx_en_orch_error.clas.abap` (modified — added bsr_key_collision constant)
- `src/zen_orch.msag.xml` (modified — added message 071)

---

## Change Log

- 2026-04-09: Story created and implemented — BSR interface (5 methods), structure + table type for get_overview, BSR_KEY_COLLISION textid added to exception class and message class

---

## Status

review
