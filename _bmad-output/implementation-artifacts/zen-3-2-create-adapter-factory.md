# Story zen-3-2: Create Adapter Factory

## Story

**Story ID:** zen-3-2
**Epic ID:** zen-epic-3
**Title:** Create adapter factory
**Target Repository:** cz.en.orch
**Depends On:** zen-3-1 (ZIF_EN_ORCH_ADAPTER must exist)
**Constitution Principles:** I (DDIC-First), II (SAP Standards), IV (Factory Pattern)

As a developer,
I want the adapter factory to instantiate the correct adapter class from the registry,
So that the engine never uses direct `NEW` for adapter creation and new adapters require zero engine code changes.

## Acceptance Criteria

- [x] AC1: `ZCL_EN_ORCH_ADAPTER_FACTORY=>create( iv_adapter_type )` reads `IMPL_CLASS` from `ZEN_ORCH_ADAPTER_REG` for the given `ADAPTER_TYPE`
- [x] AC2: Factory instantiates the class dynamically via `CREATE OBJECT lo_adapter TYPE (lv_impl_class)` and returns instance cast to `ZIF_EN_ORCH_ADAPTER`
- [x] AC3: If `ADAPTER_TYPE` is not found in the registry, raises `ZCX_EN_ORCH_ERROR` with textid `ADAPTER_START_FAILED` and `MV_ADAPTER_TYPE` populated
- [x] AC4: Factory class has `CREATE PRIVATE` constructor (no direct NEW from callers)
- [x] AC5: Calling `create` twice with the same valid adapter type returns two independent instances
- [x] AC6: Class has ABAP-Doc on the public `create` method with `@parameter` and `@raising` annotations
- [x] AC7: `ZCL_EN_ORCH_ADAPTER_FACTORY` activates without errors (abapGit-compatible XML)

## Tasks / Subtasks

- [x] T1: Create `zcl_en_orch_adapter_factory.clas.xml` — class metadata (CREATE PRIVATE, FINAL) (AC: 4, 7)
- [x] T2: Create `zcl_en_orch_adapter_factory.clas.abap` — class implementation (AC: 1, 2, 3, 4, 6)
  - [x] CLASS-METHOD `create( iv_adapter_type TYPE zen_orch_de_adapter_type ) RETURNING VALUE(ro_adapter) TYPE REF TO zif_en_orch_adapter RAISING zcx_en_orch_error`
  - [x] SELECT SINGLE from `ZEN_ORCH_ADAPTER_REG` for `iv_adapter_type`
  - [x] If not found: RAISE EXCEPTION TYPE zcx_en_orch_error textid = zcx_en_orch_error=>adapter_start_failed mv_adapter_type = iv_adapter_type
  - [x] CREATE OBJECT ro_adapter TYPE (ls_reg-impl_class)
  - [x] Cast to ZIF_EN_ORCH_ADAPTER and assign to ro_adapter
- [x] T3: Verify `CREATE OBJECT ... TYPE (dynamic_name)` works with interface reference (AC: 2, 5)
- [x] T4: Verify no ZFI_PROCESS reference (AC: per AGENTS.md zero-dependency rule)

## Dev Notes

### Class Design

`ZCL_EN_ORCH_ADAPTER_FACTORY` — CREATE PRIVATE, FINAL:

```abap
CLASS zcl_en_orch_adapter_factory DEFINITION
  PUBLIC FINAL CREATE PRIVATE.

  PUBLIC SECTION.
    "! Create an adapter instance for the given adapter type.
    "! Reads IMPL_CLASS from ZEN_ORCH_ADAPTER_REG and instantiates dynamically.
    "! @parameter iv_adapter_type | Adapter type key from ZEN_ORCH_ADAPTER_REG
    "! @parameter ro_adapter | Adapter instance implementing ZIF_EN_ORCH_ADAPTER
    "! @raising zcx_en_orch_error | Raised if adapter type not registered
    CLASS-METHODS create
      IMPORTING iv_adapter_type TYPE zen_orch_de_adapter_type
      RETURNING VALUE(ro_adapter) TYPE REF TO zif_en_orch_adapter
      RAISING zcx_en_orch_error.

ENDCLASS.
```

### Implementation Logic

```abap
METHOD create.
  DATA ls_reg TYPE zen_orch_adapter_reg.

  SELECT SINGLE impl_class
    FROM zen_orch_adapter_reg
    WHERE adapter_type = iv_adapter_type
    INTO ls_reg-impl_class.

  IF sy-subrc <> 0.
    RAISE EXCEPTION TYPE zcx_en_orch_error
      EXPORTING
        textid       = zcx_en_orch_error=>adapter_start_failed
        mv_adapter_type = iv_adapter_type.
  ENDIF.

  CREATE OBJECT ro_adapter TYPE (ls_reg-impl_class).
ENDMETHOD.
```

### Dynamic Instantiation Pattern

- `CREATE OBJECT ro_adapter TYPE (lv_class_name)` — ABAP standard for dynamic class instantiation
- The return type `REF TO zif_en_orch_adapter` automatically ensures the dynamic instance is cast to the interface
- If the dynamic class does not implement `ZIF_EN_ORCH_ADAPTER`, a runtime exception `CX_SY_CREATE_OBJECT_ERROR` will be raised — this does NOT need to be caught silently; it propagates as a crash-loud error (Constitution Principle V)

### DDIC Types Used

- `zen_orch_de_adapter_type` — data element CHAR 30
- `zen_orch_adapter_reg` — database table (MANDT + ADAPTER_TYPE → IMPL_CLASS)
- `zcx_en_orch_error` — own exception class with `ADAPTER_START_FAILED` textid

### Reference Files

- `/Users/smolik/DEV/cz.en.orch/src/zen_orch/zcl_en_orch_logger.clas.xml` — CREATE PRIVATE class XML pattern
- `/Users/smolik/DEV/cz.en.orch/src/zen_orch/zcl_en_orch_logger.clas.abap` — class implementation pattern
- `_bmad-output/planning-artifacts/epics.md` — Story 3.2 ACs (authoritative spec)

### Target Directory

`/Users/smolik/DEV/cz.en.orch/src/zen_orch/`

### Files to Create

| File | Content |
|------|---------|
| `zcl_en_orch_adapter_factory.clas.xml` | abapGit XML metadata (CREATE PRIVATE, FINAL) |
| `zcl_en_orch_adapter_factory.clas.abap` | Factory class with `create` CLASS-METHOD |

### Constitution Compliance

- **Principle I — DDIC-First**: All types (parameter + table lookup) use `zen_orch_*` DDIC objects.
- **Principle II — SAP Standards**: Name `ZCL_EN_ORCH_ADAPTER_FACTORY` follows ZCL_ convention; ABAP-Doc on public method.
- **Principle IV — Factory Pattern**: CREATE PRIVATE constructor enforces factory-only instantiation. No direct NEW for adapter creation anywhere in the engine.
- **Principle V — Error Handling**: `ZCX_EN_ORCH_ERROR` raised with `ADAPTER_START_FAILED` textid and `MV_ADAPTER_TYPE` context when type not registered.

## Dev Agent Record

### Agent Model Used

github-copilot/claude-sonnet-4.6

### Debug Log References

- T4 grep check: no ZFI_PROCESS reference in ABAP code lines (comment-only mention) — zero-dependency rule satisfied
- T3 dynamic instantiation: `CREATE OBJECT ro_adapter TYPE (lv_impl_class)` returns `REF TO zif_en_orch_adapter` — interface cast is implicit via return type declaration

### Completion Notes List

- Created `zcl_en_orch_adapter_factory.clas.xml` following `zcl_en_orch_logger.clas.xml` XML pattern
- Created `zcl_en_orch_adapter_factory.clas.abap` with CREATE PRIVATE, FINAL definition
- `create` CLASS-METHOD: SELECT SINGLE impl_class from zen_orch_adapter_reg; raises zcx_en_orch_error=>adapter_start_failed with mv_adapter_type if not found
- Dynamic instantiation via `CREATE OBJECT ro_adapter TYPE (lv_impl_class)` — return type REF TO zif_en_orch_adapter ensures implicit cast
- Two independent calls produce two independent instances (no caching/singleton) — AC5 satisfied structurally

### File List

- `src/zen_orch/zcl_en_orch_adapter_factory.clas.xml` (new)
- `src/zen_orch/zcl_en_orch_adapter_factory.clas.abap` (new)

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-04 | Story file created | SM Agent |
| 2026-04-04 | Implementation complete: zcl_en_orch_adapter_factory.clas.xml + .clas.abap created | claude-sonnet-4.6 |

### Review Findings

- [x] [Review][Decision] F7: Wrong textid for "adapter not registered" — Fixed: added `adapter_not_registered` textid (msg 027) to `zcx_en_orch_error`; factory now uses it for missing/blank registry entries. [zcx_en_orch_error.clas.abap, zcl_en_orch_adapter_factory.clas.abap]
- [x] [Review][Patch] F1: `CREATE OBJECT TYPE (lv_impl_class)` has no TRY/CATCH for `CX_SY_CREATE_OBJECT_ERROR` — Fixed: wrapped in TRY/CATCH, reraises as `zcx_en_orch_error=>adapter_start_failed` with detail. [zcl_en_orch_adapter_factory.clas.abap]
- [x] [Review][Patch] F2: Blank `lv_impl_class` passes `sy-subrc` guard — Fixed: added `IS INITIAL` check on `lv_impl_class` before `CREATE OBJECT`, raises `adapter_not_registered`. [zcl_en_orch_adapter_factory.clas.abap]
- [x] [Review][Patch] F3: No guard for blank/initial `iv_adapter_type` input — Fixed: added `IS INITIAL` guard before SELECT, raises `adapter_not_registered`. [zcl_en_orch_adapter_factory.clas.abap]
- [x] [Review][Defer] F5: Status literal `'C'` not a named constant — pre-existing project-wide pattern, not introduced by this story [zcl_en_orch_adapter_mock.clas.abap] — deferred, pre-existing

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-04 | Story file created | SM Agent |
| 2026-04-04 | Implementation complete: zcl_en_orch_adapter_factory.clas.xml + .clas.abap created | claude-sonnet-4.6 |
| 2026-04-04 | Code review patches: F7+F1+F2+F3 fixed in zcx_en_orch_error + zcl_en_orch_adapter_factory | claude-sonnet-4.6 |

## Status
done
