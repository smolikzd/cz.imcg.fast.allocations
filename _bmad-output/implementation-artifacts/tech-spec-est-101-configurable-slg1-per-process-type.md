---
title: 'Configurable SLG1 Object/Subobject per Process Type'
slug: 'est-101-configurable-slg1-per-process-type'
created: '2026-03-13'
status: 'ready-for-dev'
stepsCompleted: [1, 2, 3, 4]
tech_stack:
  - 'ABAP 7.58'
  - 'SAP DDIC (transparent table, abapGit XML)'
  - 'ZFI_PROCESS framework (planner repo)'
  - 'BALI Application Log API (CL_BALI_* OO API)'
  - 'Classic BAL FM: BAL_OBJECT_SUBOBJECT (SLG0 existence check)'
files_to_modify:
  - 'planner: src/zfi_proc_type.tabl.xml'
  - 'planner: src/zcl_fi_process_definition.clas.abap'
  - 'planner: src/zcl_fi_process_instance.clas.abap'
  - 'planner: src/zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap'
  - 'planner: src/zcl_fiproc_health_chk_query.clas.abap (3 literal replacements + ensure_test_type_defs)'
  - 'ovysledovka: src/zcl_fi_allocations.clas.abap'
code_patterns:
  - 'SELECT SINGLE <fields> FROM zfi_proc_type INTO @DATA(...) WHERE process_type = @mv_process_type AND active = ''X'''
  - 'CALL FUNCTION BAL_OBJECT_SUBOBJECT EXPORTING i_object = ... i_subobject = ... EXCEPTIONS ...'
  - 'RAISE EXCEPTION TYPE zcx_fi_process_error MESSAGE ID ... NUMBER ... WITH ...'
  - 'DATA mv_bal_object TYPE balobj_d. DATA mv_bal_subobject TYPE balsubobj.'
  - 'METHODS get_bal_object RETURNING VALUE(rv_bal_object) TYPE balobj_d.'
test_patterns:
  - 'Health check auto-creates process types with hardcoded ZFI_PROCESS/TESTS values'
  - 'Manual: run SLG1 after process execution, verify logs appear under configured object/subobject'
linear_issue: 'EST-101'
target_repository: 'both'
---

# Tech-Spec: Configurable SLG1 Object/Subobject per Process Type

**Created:** 2026-03-13
**Linear:** EST-101

## Overview

### Problem Statement

The SLG1 BAL object and subobject are hardcoded in the framework as inline string literals (`'ZFI_PROCESS'`/`'CORE'` in `ZCL_FI_PROCESS_INSTANCE` lines 450–451, 525–526 and in `ZFI_BGRFC_EXEC_SUBSTEP` lines 34–35). This makes it impossible to separate logs by process type in SLG1. The `ZCL_FI_ALLOCATIONS` class additionally declares dead BAL constants (`'ZFI_ALLOC'`/`'PHASE1..CORR_BCHE'`) that are only referenced by out-of-scope legacy programs.

### Solution

Add `SLG_OBJECT` (`TYPE BALOBJ_D`) and `SLG_SUBOBJECT` (`TYPE BALSUBOBJ`) fields to the `ZFI_PROC_TYPE` DDIC table. `ZCL_FI_PROCESS_DEFINITION` reads and exposes these values. `ZCL_FI_PROCESS_INSTANCE` validates them against SLG0 at `execute()` time (raising `ZCX_FI_PROCESS_ERROR` on failure) and passes them to the logger factory, replacing all hardcoded literals. `ZFI_BGRFC_EXEC_SUBSTEP` reads the values from the loaded instance's definition. Health check hardcodes `'ZFI_PROCESS'`/`'TESTS'`. Dead BAL constants are removed from `ZCL_FI_ALLOCATIONS`.

### Scope

**In Scope:**
- Add `SLG_OBJECT` (`BALOBJ_D`) + `SLG_SUBOBJECT` (`BALSUBOBJ`) fields to `ZFI_PROC_TYPE` DDIC table (`zfi_proc_type.tabl.xml`)
- Extend `ZCL_FI_PROCESS_DEFINITION` constructor SELECT + add two private attributes + two getter methods
- Add SLG0 existence check in `ZCL_FI_PROCESS_INSTANCE->execute()` before execution begins — raises `ZCX_FI_PROCESS_ERROR`
- Replace inline literals in `ZCL_FI_PROCESS_INSTANCE` logger create/attach calls (lines 450–451, 525–526)
- Replace inline literals in `ZFI_BGRFC_EXEC_SUBSTEP` logger create call (lines 34–35) — reads from instance definition
- Update `ZCL_FIPROC_HEALTH_CHK_QUERY` hardcoded literals (3 locations) to `'ZFI_PROCESS'`/`'TESTS'`
- Remove dead BAL constants from `ZCL_FI_ALLOCATIONS` (lines 7–11)

**Out of Scope:**
- Legacy programs `ZFI_ALLOC_PHASE_1/2/3.prog`, `ZCL_FI_ALLOC_STEP_PH1`, and `ZCL_SP_BALLOG` — untouched
- Per-step or per-phase subobject granularity (single object+subobject per process type)
- SM30 maintenance view for `ZFI_PROC_TYPE`
- `ZCL_FI_ALLOCATIONS_RULES` — no BAL constants, no changes needed

## Context for Development

### Codebase Patterns

**DDIC table field pattern** (from `zfi_proc_type.tabl.xml` existing fields):
```xml
<DD03P>
  <FIELDNAME>SLG_OBJECT</FIELDNAME>
  <ROLLNAME>BALOBJ_D</ROLLNAME>
  <ADMINFIELD>0</ADMINFIELD>
  <COMPTYPE>E</COMPTYPE>
</DD03P>
```
Non-key, non-mandatory fields use only `FIELDNAME`, `ROLLNAME`, `ADMINFIELD>0`, `COMPTYPE>E`. No `KEYFLAG`, no `NOTNULL`, no `RESERVEDOM`.

**ZCL_FI_PROCESS_DEFINITION** has no existing getter attributes — only getter methods. Add two private data attributes + two methods following the `get_process_type()` pattern (line 66–68):
```abap
"! Get SLG1 BAL object for this process type
"! @parameter rv_bal_object | BAL object name (BALOBJ_D)
METHODS get_bal_object
  RETURNING VALUE(rv_bal_object) TYPE balobj_d.
"! Get SLG1 BAL subobject for this process type
"! @parameter rv_bal_subobject | BAL subobject name (BALSUBOBJ)
METHODS get_bal_subobject
  RETURNING VALUE(rv_bal_subobject) TYPE balsubobj.
```
The constructor SELECT (line 92–96) currently fetches only `process_type`. Must be extended to also fetch `slg_object` and `slg_subobject` in the same SELECT SINGLE.

**SLG0 existence check** — use classic BAL FM (the only available API for on-premise 7.58):
```abap
CALL FUNCTION 'BAL_OBJECT_SUBOBJECT'
  EXPORTING
    i_object    = mv_bal_object
    i_subobject = mv_bal_subobject
  EXCEPTIONS
    object_not_found    = 1
    subobject_not_found = 2
    OTHERS              = 3.
IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE zcx_fi_process_error
    MESSAGE ID 'ZFI_PROCESS' NUMBER '084'
    WITH mv_bal_object mv_bal_subobject.
ENDIF.
```
This FM checks both that the object exists AND that the object+subobject combination is registered (SLG0 metadata tables `BALOBJT`/`BALSUBT`).

**ZCL_FI_PROCESS_INSTANCE logger calls** — replace literals with definition values:
```abap
" BEFORE:
iv_object    = 'ZFI_PROCESS'
iv_subobject = CONV balsubobj( 'CORE' )

" AFTER:
iv_object    = mo_definition->get_bal_object( )
iv_subobject = mo_definition->get_bal_subobject( )
```

**ZFI_BGRFC_EXEC_SUBSTEP** — `lo_instance` is already loaded (line 22). Access definition via `lo_instance->get_definition( )` (public method on `ZCL_FI_PROCESS_INSTANCE`, line 170 of instance class):
```abap
DATA(lo_def) = lo_instance->get_definition( ).
lo_logger = zcl_fi_process_logger=>create_new(
  iv_external_number = CONV balnrext( lv_substep_ext_id )
  iv_object          = lo_def->get_bal_object( )
  iv_subobject       = lo_def->get_bal_subobject( )
).
```

**Health check replacement** — 3 occurrences of `object = 'ZFI_PROCESS'` / `subobject = CONV balsubobj( 'CORE' )`:
- `check_logger_lifecycle` lines 1862–1863 → change subobject to `'TESTS'`
- `check_bgrfc_logger` lines 2021–2023 → change subobject to `'TESTS'`
- `check_bgrfc_error_propagation` lines 2257–2259 → change subobject to `'TESTS'`
Object stays `'ZFI_PROCESS'` in all three; only subobject changes from `'CORE'` to `'TESTS'`.

**ZCL_FI_ALLOCATIONS dead constants removal** — delete lines 7–11:
```abap
" DELETE these 5 lines:
CONSTANTS c_bal_object         TYPE balobj_d  VALUE 'ZFI_ALLOC'.
CONSTANTS c_bal_subobj_phase1  TYPE balsubobj VALUE 'PHASE1'.
CONSTANTS c_bal_subobj_phase2  TYPE balsubobj VALUE 'PHASE2'.
CONSTANTS c_bal_subobj_phase3  TYPE balsubobj VALUE 'PHASE3'.
CONSTANTS c_bal_subobj_corr_bche TYPE balsubobj VALUE 'CORR_BCHE'.
```
Confirmed: these are referenced only in legacy out-of-scope files (`zfi_alloc_phase_1/2/3.prog`, `zcl_fi_alloc_step_ph1`). The new step classes in `zfi_alloc_process/` do not reference them. `c_bal_subobj_corr_bche` is referenced nowhere at all.

### Files to Reference

| File | Repo | Purpose |
| ---- | ---- | ------- |
| `src/zfi_proc_type.tabl.xml` | planner | Add `SLG_OBJECT` + `SLG_SUBOBJECT` fields |
| `src/zcl_fi_process_definition.clas.abap` | planner | Extend SELECT + add 2 private attributes + 2 getter methods |
| `src/zcl_fi_process_instance.clas.abap` | planner | SLG0 check in `execute()` line ~629; replace literals lines 450–451, 525–526 |
| `src/zcl_fi_process_logger.clas.abap` | planner | Reference only — no changes needed (params already exist) |
| `src/zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap` | planner | Replace literals lines 34–35 via `lo_instance->get_definition()` |
| `src/zcl_fiproc_health_chk_query.clas.abap` | planner | Change `'CORE'` → `'TESTS'` at lines 1862–1863, 2022–2023, 2258–2259 |
| `src/zcl_fi_allocations.clas.abap` | ovysledovka | Delete dead BAL constants lines 7–11 |

### Technical Decisions

1. **Field names in ZFI_PROC_TYPE**: Use `SLG_OBJECT` and `SLG_SUBOBJECT` (not `BAL_OBJECT`/`BAL_SUBOBJECT`) to align with transaction naming (`SLG0`, `SLG1`) and avoid confusion with generic "BAL" prefix used in SAP standard type names.

2. **Reuse SAP standard rollnames**: `BALOBJ_D` for object, `BALSUBOBJ` for subobject — no new DDIC data elements or domains required (Principle I compliant, these are SAP-delivered).

3. **Existence check FM**: `BAL_OBJECT_SUBOBJECT` (classic BAL API) — checks object existence AND object+subobject combination validity in a single call. This is on-premise 7.58 compatible and the documented SAP approach.

4. **Check placement**: In `ZCL_FI_PROCESS_INSTANCE->execute()`, after the status guard (line ~628) and before step execution begins. This satisfies the "fail immediately" requirement without touching the constructor or the `load_instance` path.

5. **`ZCL_FI_PROCESS_DEFINITION` getter methods**: Read-only, no side effects. Consistent with `get_process_type()` pattern. The constructor SELECT is extended to `SELECT SINGLE process_type, slg_object, slg_subobject INTO @DATA(ls_type) FROM zfi_proc_type ...`.

6. **`ZFI_BGRFC_EXEC_SUBSTEP` access path**: Uses `lo_instance->get_definition()` (already public, line 170 of instance class). No new methods needed on the instance class.

7. **Health check**: Object stays `'ZFI_PROCESS'`; subobject changes from `'CORE'` to `'TESTS'`. The SLG0 registration for `ZFI_PROCESS/TESTS` must exist in the system — developer is responsible for registering it via SLG0 transaction before running health checks.

8. **`ZCL_FI_ALLOCATIONS` dead constants**: Deleted. Confirmed unused in the new codebase. The legacy programs that reference them are out of scope and will continue to produce a syntax warning after deletion — but since they are not compiled in the new package path, this is acceptable. (Confirm with Zdenek if a stub comment should replace the constants instead of hard deletion.)

## Implementation Plan

### Tasks

**Task 1 — DDIC: Add SLG_OBJECT + SLG_SUBOBJECT to ZFI_PROC_TYPE**
- File: `planner: src/zfi_proc_type.tabl.xml`
- Action: Insert two new `<DD03P>` entries after `ALLOW_DUPLICATE` field and before `CREATED_BY`:
  ```xml
  <DD03P>
    <FIELDNAME>SLG_OBJECT</FIELDNAME>
    <ROLLNAME>BALOBJ_D</ROLLNAME>
    <ADMINFIELD>0</ADMINFIELD>
    <COMPTYPE>E</COMPTYPE>
  </DD03P>
  <DD03P>
    <FIELDNAME>SLG_SUBOBJECT</FIELDNAME>
    <ROLLNAME>BALSUBOBJ</ROLLNAME>
    <ADMINFIELD>0</ADMINFIELD>
    <COMPTYPE>E</COMPTYPE>
  </DD03P>
  ```

**Task 2 — ZCL_FI_PROCESS_DEFINITION: Extend SELECT + add getters**
- File: `planner: src/zcl_fi_process_definition.clas.abap`
- Action A — PUBLIC SECTION: Add two method declarations after `get_process_type` (line 66):
  ```abap
  "! Get SLG1 BAL object for this process type
  "! @parameter rv_bal_object | BAL object name (BALOBJ_D)
  METHODS get_bal_object
    RETURNING VALUE(rv_bal_object) TYPE balobj_d.
  "! Get SLG1 BAL subobject for this process type
  "! @parameter rv_bal_subobject | BAL subobject name (BALSUBOBJ)
  METHODS get_bal_subobject
    RETURNING VALUE(rv_bal_subobject) TYPE balsubobj.
  ```
- Action B — PRIVATE SECTION: Add two data attributes:
  ```abap
  DATA mv_bal_object    TYPE balobj_d.
  DATA mv_bal_subobject TYPE balsubobj.
  ```
- Action C — constructor (line 92–96): Replace single-field SELECT with multi-field fetch:
  ```abap
  " BEFORE:
  SELECT SINGLE process_type
    FROM zfi_proc_type
    INTO @DATA(lv_type)
    WHERE process_type = @mv_process_type
      AND active = 'X'.
  IF sy-subrc <> 0.
    RAISE EXCEPTION TYPE zcx_fi_process_error ...
  ENDIF.

  " AFTER:
  SELECT SINGLE process_type, slg_object, slg_subobject
    FROM zfi_proc_type
    INTO @DATA(ls_type)
    WHERE process_type = @mv_process_type
      AND active = 'X'.
  IF sy-subrc <> 0.
    RAISE EXCEPTION TYPE zcx_fi_process_error ...  " existing raise — unchanged
  ENDIF.
  mv_bal_object    = ls_type-slg_object.
  mv_bal_subobject = ls_type-slg_subobject.
  ```
- Action D — add two METHOD implementations:
  ```abap
  METHOD get_bal_object.
    rv_bal_object = mv_bal_object.
  ENDMETHOD.

  METHOD get_bal_subobject.
    rv_bal_subobject = mv_bal_subobject.
  ENDMETHOD.
  ```

**Task 3 — ZCL_FI_PROCESS_INSTANCE: SLG0 existence check in execute()**
- File: `planner: src/zcl_fi_process_instance.clas.abap`
- Insertion point: After the status guard block (line 628 — `ENDIF.` closing `IF ms_instance-status <> gc_status-new`), before the comment `" Determine starting step` at line 630. Insert between lines 628 and 630.
- **Lifecycle invariant**: `mo_definition` is guaranteed to be bound at this point. It is set at line 409 in `initialize_instance()` and at line 501 in `load_instance()`. Both methods must complete successfully before an instance object exists that can call `execute()` — if either method fails, `ZCX_FI_PROCESS_ERROR` is raised and no instance is returned. Therefore no IS BOUND guard is required, but the developer should verify this invariant holds if new construction paths are ever added.
- Action: Insert SLG0 check block:
  ```abap
  " Validate SLG1 object/subobject registration before starting execution
  CALL FUNCTION 'BAL_OBJECT_SUBOBJECT'
    EXPORTING
      i_object    = mo_definition->get_bal_object( )
      i_subobject = mo_definition->get_bal_subobject( )
    EXCEPTIONS
      object_not_found    = 1
      subobject_not_found = 2
      OTHERS              = 3.
  IF sy-subrc <> 0.
    RAISE EXCEPTION TYPE zcx_fi_process_error
      MESSAGE ID 'ZFI_PROCESS' NUMBER '084'
      WITH mo_definition->get_bal_object( )
           mo_definition->get_bal_subobject( ).
  ENDIF.
  ```
- **Message 084**: Add to message class `ZFI_PROCESS` (next available slot after `083`):
  - Number: `084`
  - Type: `E` (Error)
  - Text: `SLG1 object &1 / subobject &2 not registered in SLG0`
  - This must be added to `zfi_process.msag.xml` as a new `<T100>` entry before activating Task 3 code.

**Task 4 — ZCL_FI_PROCESS_INSTANCE: Replace hardcoded literals in logger calls**
- File: `planner: src/zcl_fi_process_instance.clas.abap`
- Action A — `initialize_instance` (lines 450–451):
  ```abap
  " BEFORE:
  iv_object    = 'ZFI_PROCESS'
  iv_subobject = CONV balsubobj( 'CORE' )
  " AFTER:
  iv_object    = mo_definition->get_bal_object( )
  iv_subobject = mo_definition->get_bal_subobject( )
  ```
- Action B — `load_instance` (lines 525–526): same replacement as Action A.
- Note: `mo_definition` is already initialized before both logger calls in both methods.

**Task 5 — ZFI_BGRFC_EXEC_SUBSTEP: Replace hardcoded literals**
- File: `planner: src/zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap`
- Action A: Insert `DATA(lo_def) = lo_instance->get_definition( ).` after line 22 (`lo_instance = lo_manager->load_process( instance_id ).`), before the inner `TRY` at line 26. This keeps it inside the outer `TRY` block (which starts at line 20) where `lo_instance` is guaranteed to be bound.
- Action B: Replace literals at lines 34–35:
  ```abap
  " BEFORE (lines 34–35):
  iv_object    = 'ZFI_PROCESS'
  iv_subobject = CONV balsubobj( 'CORE' )  " Use CORE subobject; external ID distinguishes bgRFC logs
  " AFTER:
  iv_object    = lo_def->get_bal_object( )
  iv_subobject = lo_def->get_bal_subobject( )
  ```
- Action C: Update the stale block comment at lines 234–235. Change:
  ```abap
  " - Subobject: 'BGRFC' (distinct from parent's 'CORE')
  ```
  To:
  ```abap
  " - Subobject: configured per process type via ZFI_PROC_TYPE.SLG_SUBOBJECT (EST-101)
  ```

**Task 6 — ZCL_FIPROC_HEALTH_CHK_QUERY: Change CORE → TESTS (3 locations)**
- File: `planner: src/zcl_fiproc_health_chk_query.clas.abap`
- Action: In each of the three `set_descriptor(...)` calls, change subobject from `CONV balsubobj( 'CORE' )` to `CONV balsubobj( 'TESTS' )`. Object `'ZFI_PROCESS'` stays unchanged.
  - Location 1: lines 1862–1863 (`check_logger_lifecycle`) — `subobject` parameter is on line 1863
  - Location 2: lines 2021–2023 (`check_bgrfc_logger`) — `set_descriptor` opens on line 2021, `subobject` on line 2023
  - Location 3: lines 2257–2259 (`check_bgrfc_error_propagation`) — `set_descriptor` opens on line 2257, `subobject` on line 2259
- Prerequisite: Register `ZFI_PROCESS/TESTS` in SLG0 transaction in each target system before running health checks.

**Task 6b — ZCL_FIPROC_HEALTH_CHK_QUERY: Add SLG_OBJECT/SLG_SUBOBJECT to ensure_test_type_defs()**
- File: `planner: src/zcl_fiproc_health_chk_query.clas.abap`
- Action: In `ensure_test_type_defs()`, add `slg_object = 'ZFI_PROCESS'` and `slg_subobject = 'TESTS'` to **every** row in the `lt_types = VALUE #( ... )` table constructor (lines 336–524). All 17 rows must include both fields. Example pattern for each row:
  ```abap
  ( mandt           = sy-mandt
    process_type    = 'TEST_SERIAL_BASIC'
    description     = 'Test: Single serial step, always COMPLETED'
    active          = 'X'
    allow_duplicate = 'X'
    slg_object      = 'ZFI_PROCESS'
    slg_subobject   = 'TESTS'
    created_by      = sy-uname
    created_at      = lv_timestamp
    changed_by      = sy-uname
    changed_at      = lv_timestamp )
  ```
- **Why critical**: After Task 3 is implemented, `execute()` validates `slg_object`/`slg_subobject` via `BAL_OBJECT_SUBOBJECT` at startup. The `ensure_test_type_defs()` method upserts all 17 `TEST_*` rows on every health check run — with no `slg_object`, every row gets a blank value, which fails the SLG0 check with `object_not_found = 1`. Every health check test would raise `ZCX_FI_PROCESS_ERROR` before executing a single step.
- **Transport note**: This task must be activated in the same transport as Task 3, or Task 3 must be activated strictly after this task and after all `ZFI_PROC_TYPE` rows are populated.

**Task 7 — ZCL_FI_ALLOCATIONS: Remove dead BAL constants**
- File: `ovysledovka: src/zcl_fi_allocations.clas.abap`
- Action: Delete lines 7–11 (all five `CONSTANTS c_bal_*` declarations):
  ```abap
  CONSTANTS c_bal_object           TYPE balobj_d  VALUE 'ZFI_ALLOC'. "#EC NOTEXT
  CONSTANTS c_bal_subobj_phase1    TYPE balsubobj VALUE 'PHASE1'. "#EC NOTEXT
  CONSTANTS c_bal_subobj_phase2    TYPE balsubobj VALUE 'PHASE2'. "#EC NOTEXT
  CONSTANTS c_bal_subobj_phase3    TYPE balsubobj VALUE 'PHASE3'. "#EC NOTEXT
  CONSTANTS c_bal_subobj_corr_bche TYPE balsubobj VALUE 'CORR_BCHE'. "#EC NOTEXT
  ```
- Confirmed safe: not referenced in `zfi_alloc_process/` step classes. Only legacy out-of-scope files (`zfi_alloc_phase_1/2/3.prog`, `zcl_fi_alloc_step_ph1`) reference these constants.
- **WARNING — activation failure risk**: Deleting these constants will cause **syntax errors** (not warnings) in `zfi_alloc_phase_1/2/3.prog` and `zcl_fi_alloc_step_ph1`. Those programs will fail activation with `"Field 'C_BAL_OBJECT' is unknown"`. Before deleting, confirm with Zdenek whether those legacy programs are already inactive/deleted in all systems. If they are still active, choose one of these alternatives instead of hard deletion:
  - **Option A (preferred if legacy programs are active)**: Replace constants with deprecation stub comments:
    ```abap
    " @deprecated - BAL constants removed EST-101; use ZFI_PROC_TYPE SLG_OBJECT/SLG_SUBOBJECT fields
    " CONSTANTS c_bal_object ... (deleted)
    ```
  - **Option B**: Deactivate/delete the legacy programs first, then delete the constants.

### Activation Sequence

The order of activation matters because Task 3 introduces a startup check that rejects blank `slg_object`. Activating out of order will break health checks or cause activation errors.

**Required order:**

1. **DDIC (Task 1)** — Activate `zfi_proc_type.tabl.xml` first. The new fields must exist in the database before any SELECT can reference them.
2. **Populate data** — Update all existing `ZFI_PROC_TYPE` rows (including all 17 `TEST_*` health check rows) with valid `SLG_OBJECT`/`SLG_SUBOBJECT` values. Health check rows → `ZFI_PROCESS/TESTS`. Production allocation row → agreed value (e.g. `ZFI_ALLOC/ALLOCATION`). **Do not activate Task 3 before this step.**
3. **ABAP classes** (Tasks 2, 3, 4, 5, 6, 6b, 7) — Activate definition class first, then instance class, then bgRFC function group, then health check class, then allocations class.
4. **SLG0 registration** — Register `ZFI_PROCESS/TESTS` (health check) and any production SLG objects (e.g. `ZFI_ALLOC/ALLOCATION`) in transaction SLG0 in every target system before executing any process or health check.
5. **Verify** — Run health check query; confirm all checks pass. Execute one process instance; confirm SLG1 logs appear under the configured object/subobject.

**Critical constraint**: Task 3 (SLG0 check in `execute()`) and Task 6b (`ensure_test_type_defs` fix) must be transported and activated together. If Task 3 is active without Task 6b, every health check run will upsert blank `slg_object` rows and immediately fail the SLG0 check.

### Acceptance Criteria

**AC1 — DDIC fields exist**
- Given: `ZFI_PROC_TYPE` has been activated in the system
- When: Developer opens the table in SE11
- Then: Fields `SLG_OBJECT` (type `BALOBJ_D`) and `SLG_SUBOBJECT` (type `BALSUBOBJ`) are present and active

**AC2 — Definition exposes values**
- Given: A process type row in `ZFI_PROC_TYPE` with `SLG_OBJECT = 'ZFI_ALLOC'` and `SLG_SUBOBJECT = 'ALLOCATION'`
- When: `NEW zcl_fi_process_definition( 'ZFI_ALLOC_PROCESS' )->get_bal_object( )` is called
- Then: Returns `'ZFI_ALLOC'`; `get_bal_subobject( )` returns `'ALLOCATION'`

**AC3 — Pre-start check blocks unregistered object/subobject**
- Given: A process type with `SLG_OBJECT = 'DOESNOTEXIST'` / `SLG_SUBOBJECT = 'NONE'` (not in SLG0)
- When: `lo_instance->execute( )` is called
- Then: `ZCX_FI_PROCESS_ERROR` is raised immediately; no step executes; message contains object/subobject names

**AC4 — Framework logs to configured destination**
- Given: A process type with valid `SLG_OBJECT = 'ZFI_ALLOC'` / `SLG_SUBOBJECT = 'ALLOCATION'` registered in SLG0
- When: A process instance is created and executed
- Then: SLG1 shows the instance log under object `ZFI_ALLOC` / subobject `ALLOCATION`; NOT under `ZFI_PROCESS/CORE`

**AC5 — bgRFC substep logs to configured destination**
- Given: Same process type as AC4, with a bgRFC (Phase2) substep
- When: The bgRFC substep executes
- Then: SLG1 shows the substep log under `ZFI_ALLOC/ALLOCATION` with external ID `{instance_id}-{step}-{substep}`

**AC6 — Health check uses ZFI_PROCESS/TESTS**
- Given: `ZFI_PROCESS/TESTS` is registered in SLG0; health check process type has `SLG_OBJECT = 'ZFI_PROCESS'` / `SLG_SUBOBJECT = 'TESTS'`
- When: Health check query runs
- Then: BALI filters use `object = 'ZFI_PROCESS'` / `subobject = 'TESTS'`; no `'CORE'` references remain in health check

**AC7 — Dead constants removed**
- Given: `zcl_fi_allocations.clas.abap` is activated
- When: Developer searches for `c_bal_object`, `c_bal_subobj_phase1/2/3`, `c_bal_subobj_corr_bche`
- Then: None of these constants exist in `ZCL_FI_ALLOCATIONS`

## Additional Context

### Dependencies

- `ZCL_FI_PROCESS_LOGGER` factory methods already accept `iv_object` / `iv_subobject` parameters with defaults — no signature change needed (confirmed from class definition)
- `mo_definition` is initialized before both logger creation calls in `ZCL_FI_PROCESS_INSTANCE` — no sequencing issue
- `lo_instance->get_definition()` is already a public method on `ZCL_FI_PROCESS_INSTANCE` (line 170) — no new method needed on instance class
- New message number in `ZFI_PROCESS` message class required for AC3 — developer to allocate next available number
- SLG0 registration of `ZFI_PROCESS/TESTS` required in each system before health checks pass (AC6)
- All existing `ZFI_PROC_TYPE` data rows must be updated with `SLG_OBJECT` + `SLG_SUBOBJECT` values after DDIC activation

### Testing Strategy

- **Unit**: Instantiate `ZCL_FI_PROCESS_DEFINITION` for a test process type that has SLG_OBJECT/SLG_SUBOBJECT populated; assert getter return values match table data
- **Integration — happy path**: Execute a full process run; open SLG1; confirm logs under configured object/subobject
- **Integration — bgRFC**: Execute a Phase2 process; confirm substep logs in SLG1 under configured object/subobject
- **Integration — fail path**: Set a process type to an unregistered SLG0 object; call `execute()`; confirm `ZCX_FI_PROCESS_ERROR` is raised
- **Health check**: Register `ZFI_PROCESS/TESTS` in SLG0; set health check process type accordingly; run health check; confirm all checks pass

### Notes

- Constitution Principles: I (DDIC-First — reuse BALOBJ_D/BALSUBOBJ), II (SAP Standards — naming, ABAP-Doc), III (Consult SAP Docs — BAL_OBJECT_SUBOBJECT FM confirmed), V (Error Handling — ZCX_FI_PROCESS_ERROR on SLG0 check failure)
- **Principle IV (Factory Pattern) — pre-existing violation acknowledged**: `ZCL_FI_PROCESS_DEFINITION` is declared `CREATE PUBLIC` (line 9), which violates Principle IV. This is a pre-existing condition out of scope for this story. Do NOT claim Principle IV compliance here; the new getter methods follow existing class patterns and do not worsen the violation.
- Task 8 (out of spec but noted): All existing `ZFI_PROC_TYPE` data rows (including health check test data) must be populated with `SLG_OBJECT`/`SLG_SUBOBJECT` values post-activation. Health check rows get `ZFI_PROCESS/TESTS`; allocation process type row gets the agreed production values (e.g., `ZFI_ALLOC/ALLOCATION` — to be confirmed with Zdenek).
- Legacy files `zfi_alloc_phase_1/2/3.prog` + `zcl_fi_alloc_step_ph1` will still reference the deleted `ZCL_FI_ALLOCATIONS` constants — those files may produce activation warnings. Since they are legacy/superseded code, this is accepted. If unacceptable, the constants can be kept with a `"#EC` pragma or a deprecation comment instead of deleted.
