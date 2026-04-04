# Story zen-3-1: Create Adapter Interface

## Story

**Story ID:** zen-3-1
**Epic ID:** zen-epic-3
**Title:** Create adapter interface
**Target Repository:** cz.en.orch
**Depends On:** zen-2-2 (done)
**Constitution Principles:** I (DDIC-First), II (SAP Standards), III (Consult SAP Docs)

As a developer,
I want the `ZIF_EN_ORCH_ADAPTER` interface defined with its 6-method contract,
So that any framework adapter (planner, workflow, mock) implements a uniform interaction pattern with the engine.

## Acceptance Criteria

- [ ] AC1: Interface `ZIF_EN_ORCH_ADAPTER` declares exactly 6 methods with correct signatures:
  - `start( IMPORTING iv_params_json TYPE string RETURNING VALUE(rs_result) TYPE zen_orch_s_start_result RAISING zcx_en_orch_error )`
  - `get_status( IMPORTING iv_handle TYPE zen_orch_wu_handle RETURNING VALUE(rv_status) TYPE zen_orch_de_status RAISING zcx_en_orch_error )`
  - `cancel( IMPORTING iv_handle TYPE zen_orch_wu_handle RAISING zcx_en_orch_error )`
  - `restart( IMPORTING iv_handle TYPE zen_orch_wu_handle RETURNING VALUE(rs_result) TYPE zen_orch_s_start_result RAISING zcx_en_orch_error )`
  - `get_result( IMPORTING iv_handle TYPE zen_orch_wu_handle RETURNING VALUE(rv_result_json) TYPE string RAISING zcx_en_orch_error )`
  - `get_detail_link( IMPORTING iv_handle TYPE zen_orch_wu_handle RETURNING VALUE(rv_url) TYPE string )`
- [ ] AC2: All methods have ABAP-Doc comments (`"!` with `@parameter` and `@raising` annotations)
- [ ] AC3: Interface has no reference to ZFI_PROCESS objects (zero compile-time dependency)
- [ ] AC4: Interface uses only DDIC types from `zen_orch_*` namespace (Principle I)
- [ ] AC5: Interface activates without errors (abapGit-compatible XML)

## Tasks / Subtasks

- [ ] T1: Create `zif_en_orch_adapter.intf.xml` — interface metadata (AC: 5)
  - [ ] Package: ZEN_ORCH, no superinterface, no class category
- [ ] T2: Create `zif_en_orch_adapter.intf.abap` — interface source with all 6 methods + ABAP-Doc (AC: 1, 2)
  - [ ] Method `start` — launches a work unit; returns handle + initial status
  - [ ] Method `get_status` — polls current status of a running work unit
  - [ ] Method `cancel` — cancels a running or pending work unit
  - [ ] Method `restart` — restarts a failed or cancelled work unit; returns new handle + status
  - [ ] Method `get_result` — retrieves JSON result payload of a completed work unit
  - [ ] Method `get_detail_link` — returns a UI navigation URL for the work unit (no RAISING — best-effort)
- [ ] T3: Verify no ZFI_PROCESS reference (grep check) (AC: 3)
- [ ] T4: Verify all parameter types are DDIC-based `zen_orch_*` types (AC: 4)

## Dev Notes

### Interface Design

`ZIF_EN_ORCH_ADAPTER` — 6-method uniform adapter contract:

```abap
INTERFACE zif_en_orch_adapter
  PUBLIC.

  "! Start a new work unit for this adapter.
  "! @parameter iv_params_json | Resolved parameters JSON for this step
  "! @parameter rs_result | Start result: work unit handle and initial status
  "! @raising zcx_en_orch_error | Adapter start failure (fail-stop)
  METHODS start
    IMPORTING iv_params_json TYPE string
    RETURNING VALUE(rs_result) TYPE zen_orch_s_start_result
    RAISING zcx_en_orch_error.

  "! Get the current execution status of a work unit.
  "! @parameter iv_handle | Opaque work unit handle returned by start
  "! @parameter rv_status | Current status (zen_orch_de_status fixed values)
  "! @raising zcx_en_orch_error | Transient polling error (engine retries)
  METHODS get_status
    IMPORTING iv_handle TYPE zen_orch_wu_handle
    RETURNING VALUE(rv_status) TYPE zen_orch_de_status
    RAISING zcx_en_orch_error.

  "! Cancel a running or pending work unit.
  "! @parameter iv_handle | Opaque work unit handle to cancel
  "! @raising zcx_en_orch_error | Cancel failure
  METHODS cancel
    IMPORTING iv_handle TYPE zen_orch_wu_handle
    RAISING zcx_en_orch_error.

  "! Restart a failed or cancelled work unit.
  "! @parameter iv_handle | Previous work unit handle
  "! @parameter rs_result | New handle and initial status
  "! @raising zcx_en_orch_error | Restart failure
  METHODS restart
    IMPORTING iv_handle TYPE zen_orch_wu_handle
    RETURNING VALUE(rs_result) TYPE zen_orch_s_start_result
    RAISING zcx_en_orch_error.

  "! Retrieve JSON result payload of a completed work unit.
  "! @parameter iv_handle | Opaque work unit handle
  "! @parameter rv_result_json | JSON-encoded result data
  "! @raising zcx_en_orch_error | Result retrieval failure
  METHODS get_result
    IMPORTING iv_handle TYPE zen_orch_wu_handle
    RETURNING VALUE(rv_result_json) TYPE string
    RAISING zcx_en_orch_error.

  "! Return a UI navigation URL for the work unit (best-effort, no RAISING).
  "! @parameter iv_handle | Opaque work unit handle
  "! @parameter rv_url | Navigation URL or empty string if not available
  METHODS get_detail_link
    IMPORTING iv_handle TYPE zen_orch_wu_handle
    RETURNING VALUE(rv_url) TYPE string.

ENDINTERFACE.
```

### Key DDIC Types Used

- `zen_orch_s_start_result` — structure: `WORK_UNIT_HANDLE TYPE zen_orch_wu_handle`, `STATUS TYPE zen_orch_de_status`
- `zen_orch_wu_handle` — data element: CHAR 255 (opaque adapter handle)
- `zen_orch_de_status` — data element based on domain `ZEN_ORCH_STATUS` (CHAR 1)
- `zcx_en_orch_error` — own exception class (Epic 2, Story 2.1)

### abapGit XML Pattern

Interface `.xml` follows same pattern as `zif_en_orch_logger.intf.xml`. Key fields:
- `<CLSNAME>ZIF_EN_ORCH_ADAPTER</CLSNAME>`
- `<CATEGORY>01</CATEGORY>` (interface)
- `<EXPOSURE>2</EXPOSURE>` (public)

### Reference Files

- `/Users/smolik/DEV/cz.en.orch/src/zen_orch/zif_en_orch_logger.intf.xml` — XML metadata pattern
- `/Users/smolik/DEV/cz.en.orch/src/zen_orch/zif_en_orch_logger.intf.abap` — ABAP-Doc pattern
- `_bmad-output/planning-artifacts/epics.md` — Story 3.1 ACs (authoritative spec)

### Target Directory

`/Users/smolik/DEV/cz.en.orch/src/zen_orch/`

### Files to Create

| File | Content |
|------|---------|
| `zif_en_orch_adapter.intf.xml` | abapGit XML metadata |
| `zif_en_orch_adapter.intf.abap` | Interface with 6 methods + ABAP-Doc |

### Constitution Compliance

- **Principle I — DDIC-First**: All parameter types use `zen_orch_*` DDIC types; no local type definitions in the interface.
- **Principle II — SAP Standards**: Name `ZIF_EN_ORCH_ADAPTER` follows ZIF_ prefix convention; all methods have ABAP-Doc.
- **Principle III — Consult SAP Docs**: Verify ABAP interface syntax for the target ABAP version (7.58) before implementation.
- **No ZFI_PROCESS dependency**: Zero compile-time coupling to planner framework (grep-verified before commit).

## Dev Agent Record

### Agent Model Used

_TBD_

### Debug Log References

_Empty_

### Completion Notes List

_Empty_

### File List

- `src/zen_orch/zif_en_orch_adapter.intf.xml` (new)
- `src/zen_orch/zif_en_orch_adapter.intf.abap` (new)

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-04 | Story file created | SM Agent |

## Status
ready-for-dev
