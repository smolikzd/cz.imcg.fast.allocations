# Story zen-7-1: BSR DDIC Foundation

## Story

**Story ID:** zen-7-1
**Epic ID:** zen-7
**Title:** BSR DDIC Foundation
**Target Repository:** cz.en.orch
**Depends On:** []
**Status:** in-progress

**Constitution Principles:**
- Principle I — DDIC-First
- Principle II — SAP Standards

As a developer,
I want the BSR persistence table and supporting DDIC types defined,
So that the engine can store and query cross-performance business key registrations.

---

## Acceptance Criteria

**Given** the ZEN_ORCH package and existing DDIC types from Phase 1 exist
**When** the BSR DDIC objects are activated
**Then** a new domain `ZEN_ORCH_BSR_KEY` (CHAR 255, no fixed values — opaque business key) exists
**And** a new data element `ZEN_ORCH_DE_BSR_KEY` referencing `ZEN_ORCH_BSR_KEY` exists with field label "BSR Business Key"
**And** table `ZEN_ORCH_BSR` exists with:
  - Primary key: `MANDT`, `BSR_KEY`
  - Fields: `PERF_UUID TYPE zen_orch_perf_uuid`, `SCORE_ID TYPE zen_orch_de_score_id`, `PARAMS_HASH TYPE zen_orch_de_params_hash` (SHA-256 hex of params JSON), `REGISTERED_AT TYPE timestampl`, `REGISTERED_BY TYPE syuname`
  - Delivery class: A (application data)
  - Client-dependent (MANDT first key field)
**And** a new domain `ZEN_ORCH_PARAMS_HASH` (CHAR 64) exists
**And** a new data element `ZEN_ORCH_DE_PARAMS_HASH` (CHAR 64) exists for the SHA-256 hash field
**And** all objects activate without errors

**Notes:**
- `BSR_KEY` is an opaque string — the engine never interprets it; callers compose it (e.g., `"ALLOC:1000:2026:001"`)
- `PARAMS_HASH` enables fast duplicate-run detection without full JSON comparison
- No table type needed for BSR table — engine queries it directly by key

---

## Tasks/Subtasks

- [x] Task 1: Create domain `ZEN_ORCH_BSR_KEY` (CHAR 255)
  - [x] Create `zen_orch_bsr_key.doma.xml` in `cz.en.orch/src/`
- [x] Task 2: Create data element `ZEN_ORCH_DE_BSR_KEY`
  - [x] Create `zen_orch_de_bsr_key.dtel.xml` in `cz.en.orch/src/`
- [x] Task 3: Create domain `ZEN_ORCH_PARAMS_HASH` (CHAR 64)
  - [x] Create `zen_orch_params_hash.doma.xml` in `cz.en.orch/src/`
- [x] Task 4: Create data element `ZEN_ORCH_DE_PARAMS_HASH`
  - [x] Create `zen_orch_de_params_hash.dtel.xml` in `cz.en.orch/src/`
- [x] Task 5: Create table `ZEN_ORCH_BSR`
  - [x] Create `zen_orch_bsr.tabl.xml` in `cz.en.orch/src/`

---

## Dev Notes

### Architecture Context

This story is part of ZEN_ORCH Phase 2 — Business State Registry (BSR). Phase 1 (Epics 1–6) delivered the engine core.

**Repository:** `cz.en.orch` at `/Users/smolik/DEV/cz.en.orch`
**Package:** `ZEN_ORCH`
**abapGit format:** XML serialization (all DDIC objects as `.xml` files in `src/`)

### DDIC Object Naming Conventions (from Phase 1)

- Domains: `ZEN_ORCH_<NAME>` (no `DE_` prefix)
- Data elements: `ZEN_ORCH_DE_<NAME>`
- Tables: `ZEN_ORCH_<NAME>`

### Activation Order

1. Domain `ZEN_ORCH_BSR_KEY`
2. Domain `ZEN_ORCH_PARAMS_HASH`
3. Data element `ZEN_ORCH_DE_BSR_KEY` (references domain ZEN_ORCH_BSR_KEY)
4. Data element `ZEN_ORCH_DE_PARAMS_HASH` (references domain ZEN_ORCH_PARAMS_HASH)
5. Table `ZEN_ORCH_BSR` (references ZEN_ORCH_DE_BSR_KEY, ZEN_ORCH_PERF_UUID, ZEN_ORCH_DE_SCORE_ID, ZEN_ORCH_DE_PARAMS_HASH)

### Table Fields Reference

| Field | Key | Data Element / Type | Description |
|-------|-----|---------------------|-------------|
| MANDT | X | MANDT | Client |
| BSR_KEY | X | ZEN_ORCH_DE_BSR_KEY | Opaque business key (CHAR 255) |
| PERF_UUID | | ZEN_ORCH_PERF_UUID | Owning performance UUID |
| SCORE_ID | | ZEN_ORCH_DE_SCORE_ID | Score identifier |
| PARAMS_HASH | | ZEN_ORCH_DE_PARAMS_HASH | SHA-256 hex of params JSON (CHAR 64) |
| REGISTERED_AT | | TIMESTAMPL | Registration timestamp |
| REGISTERED_BY | | SYUNAME | Registering user |

---

## Dev Agent Record

### Implementation Plan

Story is DDIC-only — no ABAP class code. All deliverables are abapGit XML files placed in `cz.en.orch/src/`.

Files created:
- `zen_orch_bsr_key.doma.xml` — Domain CHAR 255 for opaque BSR business key
- `zen_orch_de_bsr_key.dtel.xml` — Data element for BSR business key field
- `zen_orch_params_hash.doma.xml` — Domain CHAR 64 for SHA-256 hex hash
- `zen_orch_de_params_hash.dtel.xml` — Data element for params hash field
- `zen_orch_bsr.tabl.xml` — Database table ZEN_ORCH_BSR (client-dep, delivery class A)

### Completion Notes

All 4 DDIC XML files created following Phase 1 conventions (BOM prefix, LCL_OBJECT_DOMA/DTEL/TABL serializers, DDLANGUAGE E, CONTFLAG A for table).

Table `ZEN_ORCH_BSR` mirrors the structure of `ZEN_ORCH_PERF`: transparent table, client-dependent (CLIDEP X), delivery class A, TABKAT 3/TABART APPL1, BUFALLOW N.

REGISTERED_AT uses SAP standard type `TIMESTAMPL` (long timestamp); REGISTERED_BY uses `SYUNAME` — both are well-known SAP standard data elements, no domain needed.

---

## File List

**Repository: cz.en.orch**

- `src/zen_orch_bsr_key.doma.xml` (new)
- `src/zen_orch_de_bsr_key.dtel.xml` (new)
- `src/zen_orch_params_hash.doma.xml` (new)
- `src/zen_orch_de_params_hash.dtel.xml` (new)
- `src/zen_orch_bsr.tabl.xml` (new)

---

## Change Log

- 2026-04-09: Story created and implemented — 5 DDIC XML files added to cz.en.orch/src/ (domains, data elements, BSR table)

---

## Status

review
