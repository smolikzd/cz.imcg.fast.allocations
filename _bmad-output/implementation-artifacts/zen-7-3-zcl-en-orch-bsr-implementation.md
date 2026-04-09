# Story zen-7-3: ZCL_EN_ORCH_BSR Implementation

## Story

**Story ID:** zen-7-3
**Epic ID:** zen-7
**Title:** ZCL_EN_ORCH_BSR Implementation
**Target Repository:** cz.en.orch
**Depends On:** ["zen-7-1", "zen-7-2"]
**Status:** review

**Constitution Principles:**
- Principle I — DDIC-First
- Principle II — SAP Standards
- Principle IV — Factory Pattern

As an engine,
I want a concrete BSR implementation backed by `ZEN_ORCH_BSR`,
So that performance registrations, collision checks, and prerequisite queries work against the real database.

---

## Acceptance Criteria

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
**And** returns `gc_status-failed` if the BSR entry exists but the referenced performance row is not found (dangling UUID)

**Given** `deregister` is called
**When** the method executes
**Then** the row matching `BSR_KEY` + `PERF_UUID` is deleted from `ZEN_ORCH_BSR` (no-op if not found)
**And** no COMMIT is issued

**And** `ZCL_EN_ORCH_BSR` has a factory method `create( )` returning `REF TO zif_en_orch_bsr`
**And** the class activates without errors

---

## Tasks/Subtasks

- [x] Task 1: Create class metadata `zcl_en_orch_bsr.clas.xml`
  - [x] Create XML descriptor with CLSNAME=ZCL_EN_ORCH_BSR, description, FIXPT=X, UNICODE=X
- [x] Task 2: Implement `ZCL_EN_ORCH_BSR` class body in `zcl_en_orch_bsr.clas.abap`
  - [x] Class definition: CREATE PRIVATE, INTERFACES zif_en_orch_bsr, factory CLASS-METHOD create
  - [x] Implement `register` — INSERT or overwrite-stale, guard collision on active perf
  - [x] Implement `deregister` — DELETE by BSR_KEY + PERF_UUID, idempotent
  - [x] Implement `check_collision` — SELECT by SCORE_ID + PARAMS_HASH, resolve live status
  - [x] Implement `check_prerequisite` — SELECT by BSR_KEY, resolve live status, return PENDING if not found / FAILED if dangling
  - [x] Implement `get_overview` — SELECT all BSR rows, enrich with live status from PERF
  - [x] Implement factory method `create` — returns NEW ZCL_EN_ORCH_BSR

### Review Findings

- [x] [Review][Patch] `register`: initial-value guard raises `bsr_key_collision` — wrong textid misleads callers [`zcl_en_orch_bsr.clas.abap:59-64`] — fixed: added `bsr_invalid_param` (msg 072), raised with `mv_detail`
- [x] [Review][Patch] `register`: collision RAISE omits `mv_score_id` — message 071 &2 placeholder blank [`zcl_en_orch_bsr.clas.abap:81-85`] — fixed: SELECT now fetches `score_id`, passed as `mv_score_id` on raise
- [x] [Review][Patch] `deregister`: DB error silently swallowed despite `RAISING zcx_en_orch_error` clause [`zcl_en_orch_bsr.clas.abap:107-110`] — fixed: DELETE wrapped in TRY/CATCH cx_sy_open_sql_error
- [x] [Review][Patch] `zen_orch.msag.xml`: T100 entries 064+071 indented one level deeper than all others [`zen_orch.msag.xml:155-167`] — fixed: normalized indentation, added msg 072
- [x] [Review][Defer] `check_collision` + `get_overview`: N+1 SELECT pattern [`zcl_en_orch_bsr.clas.abap:123-133, 176-185`] — deferred, pre-existing pattern; optimization only, no correctness issue
- [x] [Review][Defer] Concurrent `register` race: no INSERT duplicate-key guard [`zcl_en_orch_bsr.clas.abap:93-100`] — deferred, engine LUW is single-threaded by design

---

## Dev Notes

### Architecture Context

- **Repository:** `cz.en.orch` at `/Users/smolik/DEV/cz.en.orch`
- **Package:** `ZEN_ORCH`
- **abapGit format:** XML serialization (`.xml` for metadata, `.abap` for code)

### Interface Contract (from zen-7-2)

`ZIF_EN_ORCH_BSR` declares 5 methods — see `zif_en_orch_bsr.intf.abap`. Key design decisions:
- `check_collision` has NO RAISING clause — caller decides
- `deregister` is idempotent — no-op if row not found
- `register` raises `BSR_KEY_COLLISION` (msgno 071) if key held by non-terminal perf
- `register` overwrites stale entries from terminal performances (auto-cleanup on next use)
- `check_prerequisite` returns `gc_status-pending` if key unknown; `gc_status-failed` if dangling UUID (perf row deleted)

### Status Constants

Terminal statuses: `C` (COMPLETED), `F` (FAILED), `X` (CANCELLED)
Non-terminal: `P` (PENDING), `R` (RUNNING), `B` (PAUSED/breakpoint)
Status constants live in `zcl_en_orch_engine=>gc_status`.

### DDIC Tables Used

- `ZEN_ORCH_BSR` (key: MANDT + BSR_KEY; fields: PERF_UUID, SCORE_ID, PARAMS_HASH, REGISTERED_AT, REGISTERED_BY)
- `ZEN_ORCH_PERF` (key: MANDT + PERF_UUID; field used: STATUS)

### Factory Pattern (Constitution Principle IV)

- Class is CREATE PRIVATE
- Single `CLASS-METHOD create RETURNING VALUE(ro_bsr) TYPE REF TO zif_en_orch_bsr`
- Callers: `ZCL_EN_ORCH_BSR=>create( )` — returns interface reference

### No COMMIT in register/deregister

Both methods operate within the engine's LUW. COMMIT WORK is issued by the engine (sweep_all) after all operations for a performance are complete.

### Dangling UUID Decision (from zen-7-2 review)

If `check_prerequisite` finds a BSR entry whose PERF_UUID no longer exists in `ZEN_ORCH_PERF`, return `gc_status-failed` — the predecessor crashed without cleanup. This prevents PREREQ_GATE from blocking forever on a ghost key.

### GET TIME STAMP

Use `GET TIME STAMP FIELD lv_ts` to populate REGISTERED_AT in `register`.

### Activation Order (epics-zen-phase2.md §17)

`ZCL_EN_ORCH_BSR` at position 17, after `ZIF_EN_ORCH_BSR` (16) and DDIC (13–15).

---

## Dev Agent Record

### Implementation Plan

Story delivers:
1. `zcl_en_orch_bsr.clas.xml` — class metadata descriptor
2. `zcl_en_orch_bsr.clas.abap` — class implementation with all 5 interface methods + factory

No unit tests in this story — Story 7.6 covers all BSR unit tests once engine integration (7.4) is also done.

### Completion Notes

- **Completed:** 2026-04-09
- **Files delivered:**
  - `src/zcl_en_orch_bsr.clas.xml` — class metadata (CREATE PRIVATE, FIXPT=X, UNICODE=X)
  - `src/zcl_en_orch_bsr.clas.abap` — full implementation (all 5 interface methods + `create` factory)
- **Key implementation decisions:**
  - `register`: guards initial key/hash, then checks for existing entry; raises `bsr_key_collision` (msgno 071) if existing perf is non-terminal; silently deletes stale entries (terminal perf or dangling UUID) before INSERT
  - `deregister`: single `DELETE` on BSR_KEY + PERF_UUID; idempotent (no error if not found)
  - `check_collision`: loops BSR candidates by SCORE_ID+PARAMS_HASH, resolves live status via PERF; returns first non-terminal UUID or initial
  - `check_prerequisite`: returns `gc_status-pending` for unknown key; returns `gc_status-failed` for dangling UUID; otherwise returns live status from PERF
  - `get_overview`: loads all BSR rows, enriches each with live status (dangling → FAILED); returns `zen_orch_tt_bsr_overview` table
  - Private helper `is_terminal` centralises C/F/X check
  - No COMMIT WORK — engine owns the LUW throughout

---

## File List

**Repository: cz.en.orch**

- `src/zcl_en_orch_bsr.clas.xml` (new)
- `src/zcl_en_orch_bsr.clas.abap` (new)

---

## Change Log

- 2026-04-09: Story file created — ZCL_EN_ORCH_BSR concrete BSR implementation

---

## Status

done
