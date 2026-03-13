---
title: 'Configurable SLG1 Object and Subobject per Process Type (EST-101)'
slug: 'est-101-configurable-bal-object-subobject'
created: '2026-03-13'
status: 'ready-for-dev'
stepsCompleted: [1, 2, 3, 4]
tech_stack:
  - 'ABAP 7.58 (SAP S/4HANA on-premise)'
  - 'SAP DDIC (DDIC-first, abapGit XML)'
  - 'SAP Application Log (BAL) — BALI API (CL_BALI_LOG_FILTER, CL_BALI_LOG_DB)'
  - 'ZFI_PROCESS framework (cz.imcg.fast.planner)'
  - 'bgRFC (ZFI_BGRFC_EXEC_SUBSTEP function module)'
files_to_modify:
  - 'planner: zfi_proc_type.tabl.xml (add BAL_OBJECT, BAL_SUBOBJECT fields)'
  - 'planner: zcl_fi_process_instance.clas.abap (add get_bal_object/get_bal_subobject, store private fields, use in initialize_instance + load_instance)'
  - 'planner: zcl_fi_process_manager.clas.abap (add BAL existence validation in create_process)'
  - 'planner: zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap (use configured BAL values instead of hardcoded CORE)'
  - 'planner: zcl_fiproc_health_chk_query.clas.abap (ensure_test_type_defs + 3 logger health check methods)'
code_patterns:
  - 'DDIC-first: new data elements ZFI_PROCESS_BAL_OBJECT (TYPE BALOBJ_D) and ZFI_PROCESS_BAL_SUBOBJ (TYPE BALSUBOBJ)'
  - 'Factory pattern: no direct NEW for framework classes'
  - 'Error handling: ZCX_FI_PROCESS_ERROR with textid and value'
  - 'ABAP-Doc on all public methods'
  - 'Line length <= 120 chars (abaplint enforced)'
  - 'Constants structure pattern (gc_*)'
test_patterns:
  - 'Manual testing via test programs (ZFI_ALLOC_TEST_PHASE1, etc.)'
  - 'Health check self-test via ZCL_FIPROC_HEALTH_CHK_QUERY (RAP query)'
  - 'SLG1 verification after execution'
---

# Tech-Spec: Configurable SLG1 Object and Subobject per Process Type (EST-101)

**Created:** 2026-03-13

## Overview

### Problem Statement

The BAL (Application Log) object and subobject values are hardcoded as string literals in
multiple places across both code repositories:

**Planner repo hardcoded `'ZFI_PROCESS'`/`'CORE'`:**
- `zcl_fi_process_instance.clas.abap` lines 450–451 (`initialize_instance`)
- `zcl_fi_process_instance.clas.abap` lines 525–526 (`load_instance`)
- `zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap` line 35
- `zcl_fi_process_logger.clas.abap` default parameters in `create_new` and `attach_existing`
  _(Note: the logger defaults are intentionally left unchanged by this spec — see Out of Scope.
  They serve as a last-resort fallback. After this change, all callers pass explicit values
  so the defaults are never reached in practice.)_

**Ovysledovka repo — `ZCL_FI_ALLOCATIONS` constants (read-only, cannot be modified):**
- `c_bal_object = 'ZFI_ALLOC'`
- `c_bal_subobj_phase1 = 'PHASE1'`, `c_bal_subobj_phase2 = 'PHASE2'`, etc.

**Ovysledovka step classes** currently reference `ZCL_FI_ALLOCATIONS` constants when
initializing BAL loggers.

This makes it impossible to configure different BAL object/subobject combinations per
process type at the customizing level (SLG0/SLG1 administration).

### Solution

1. Add `BAL_OBJECT` and `BAL_SUBOBJECT` fields to `ZFI_PROC_TYPE` customizing table.
2. Validate that the configured object/subobject exists in SLG0 before process instance creation.
3. Add `get_bal_object()` and `get_bal_subobject()` getter methods to `ZCL_FI_PROCESS_INSTANCE`
   so all step classes can retrieve the configured values via `is_context-io_process_instance`.
4. Update `ZCL_FI_PROCESS_INSTANCE` to store BAL values at create/load time and use them
   when creating the framework logger.
5. Update `ZFI_BGRFC_EXEC_SUBSTEP` to use configured BAL values instead of hardcoded `'CORE'`.
6. Update health check to populate `bal_object`/`bal_subobject` for test types, and fix
   BALI query filters that must use `'TESTS'` subobject.

> **Note:** The ovysledovka step classes (`ZCL_FI_ALLOC_STEP_*`) require **no changes**.
> They all use the inherited `mo_log` (protected attribute on `ZCL_FI_PROCESS_STEP`), which
> is injected by the framework via `initialize_logger()` before every `execute()` call
> (see `zcl_fi_process_instance.clas.abap` line 738). Once the framework's `ZCL_FI_PROCESS_INSTANCE`
> uses the configured BAL values (Tasks 3e/3f), all step classes automatically log to the
> correct object/subobject — zero ovysledovka changes needed.

### Scope

**In Scope:**

- DDIC: Two new data elements (`ZFI_PROCESS_BAL_OBJECT`, `ZFI_PROCESS_BAL_SUBOBJ`) and
  two new fields on `ZFI_PROC_TYPE` table
- Framework (`cz.imcg.fast.planner`):
  - `ZFI_PROC_TYPE` table extension
  - `ZCL_FI_PROCESS_INSTANCE` — private storage + public getters + use in initialize/load
  - `ZCL_FI_PROCESS_MANAGER` — BAL existence validation in `create_process`
  - `ZFI_BGRFC_EXEC_SUBSTEP` — use configured values
  - `ZCL_FIPROC_HEALTH_CHK_QUERY` — test type setup + health check BALI filter fixes

**Out of Scope:**

- Modification of `ZCL_FI_ALLOCATIONS` (read-only per project constraint)
- Any changes to ovysledovka step classes (`ZCL_FI_ALLOC_STEP_*`) — they inherit `mo_log`
  from `ZCL_FI_PROCESS_STEP` which is injected by the framework before each step executes
- Changes to `ZCL_FI_PROCESS_LOGGER` default parameter values (planner logger defaults
  remain as-is; callers pass explicit values)
- UI for maintaining `ZFI_PROC_TYPE` records (SE16 / SPRO)
- Migration of existing `ZFI_PROC_INST` log records to new object/subobject values
- Per-substep BAL subobject differentiation (all steps in a process type share one subobject)

---

## Context for Development

### Codebase Patterns

- **DDIC-First (Rule 1):** All new fields must use DDIC data elements. Two new data elements
  needed: `ZFI_PROCESS_BAL_OBJECT` (referencing domain `BALOBJ_D`) and `ZFI_PROCESS_BAL_SUBOBJ`
  (referencing domain `BALSUBOBJ`). No local TYPE definitions allowed.

- **Factory pattern (Rule 3):** `ZCL_FI_PROCESS_INSTANCE` has `CREATE PRIVATE` — only touched
  through `create()` and `load()` factory methods. New getter methods are public instance methods.

- **Error handling (Rule 4):** Validation error in `create_process` raises `ZCX_FI_PROCESS_ERROR`
  with `textid = zcx_fi_process_error=>validation_failed` and descriptive `value`.

- **ABAP-Doc (Rule 7):** All new public methods need `"!` header + `@parameter` + `@raising`.

- **Line length (Rule 2):** abaplint enforces ≤ 120 chars. Multi-line wrapping required for
  RAISE EXCEPTION and method calls.

- **Logging pattern:** Standard dual pattern — `MESSAGE...INTO DATA(lv_dummy)` then
  `mo_log->message(...)` with `IS BOUND` guard and `CATCH zcx_fi_process_error`.

- **Constants structure pattern (Rule 9):** New BAL values stored as private data members
  (`mv_bal_object TYPE balobj_d`, `mv_bal_subobject TYPE balsubobj`) on `ZCL_FI_PROCESS_INSTANCE`.

- **`initialize_instance` flow:** Reads `ZFI_PROC_TYPE` (via `mo_definition`) — BAL values
  must be loaded here and stored in private members before logger creation.

- **`load_instance` flow:** Reads instance from `ZFI_PROC_INST`, then `ZFI_PROC_TYPE` is
  accessed via `populate_param_columns`. BAL values loaded separately via SELECT from
  `ZFI_PROC_TYPE` by `ms_instance-process_type`.

- **bgRFC substep logger:** Currently hardcoded `'ZFI_PROCESS'`/`'CORE'`. After this change,
  the FM will call `lo_instance->get_bal_object()`/`get_bal_subobject()` after loading the instance.

- **Health check test types:** All `TEST_%` types in `ensure_test_type_defs` must receive
  `bal_object = 'ZFI_PROCESS'` / `bal_subobject = 'TESTS'`. The 3 BALI query methods
  (`check_logger_integration`, `check_bgrfc_logger`, `check_bgrfc_error_propagation`) use
  `gc_bal_subobject` (= `'CORE'`) but must switch to `gc_bal_subobj_tests` (= `'TESTS'`)
  for test process logs.

- **Ovysledovka step classes:** No changes required. The 5 step classes all use the inherited
  `mo_log` (protected attribute on `ZCL_FI_PROCESS_STEP`) which is set by the framework via
  `lo_step_base->initialize_logger( mo_log )` before each step executes. The getters
  `get_bal_object()` / `get_bal_subobject()` are added for completeness and future use,
  but the ovysledovka step classes do not need to call them.

### Files to Reference

| File | Purpose |
| ---- | ------- |
| `planner/src/zfi_proc_type.tabl.xml` | Customizing table — add `BAL_OBJECT`, `BAL_SUBOBJECT` fields |
| `planner/src/zcl_fi_process_instance.clas.abap` | Add private members + getters + use in initialize/load |
| `planner/src/zcl_fi_process_manager.clas.abap` | Add BAL existence validation in `create_process` |
| `planner/src/zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap` | Use `get_bal_object()`/`get_bal_subobject()` |
| `planner/src/zcl_fiproc_health_chk_query.clas.abap` | Fix test type setup + BALI query subobjects |
| `planner/src/zcl_fi_process_step.clas.abap` | Reference — confirms `mo_log` injection via `initialize_logger()` (no changes) |
| `planner/src/zif_fi_process_step.intf.abap` | Reference — `ty_context` with `io_process_instance` |
| `ovysledovka/src/zcl_fi_allocations.clas.abap` | READ ONLY — constants reference only |
| `allocations/_bmad-output/project-context.md` | Rules, patterns, pitfalls |

### Technical Decisions

1. **Mandatory fields**: `BAL_OBJECT` and `BAL_SUBOBJECT` in `ZFI_PROC_TYPE` are mandatory
   (non-nullable). Every process type registration must include them.

2. **New DDIC data elements** (not reusing raw `BALOBJ_D`/`BALSUBOBJ` directly as rollnames):
   - `ZFI_PROCESS_BAL_OBJECT TYPE BALOBJ_D` — BAL object for process type logging
   - `ZFI_PROCESS_BAL_SUBOBJ TYPE BALSUBOBJ` — BAL subobject for process type logging
   These follow the project naming convention `ZFI_PROCESS_*` for all data elements.

3. **Storage strategy**: BAL values loaded into private members `mv_bal_object` and
   `mv_bal_subobject` on `ZCL_FI_PROCESS_INSTANCE` during `initialize_instance`/`load_instance`.
   A simple SELECT from `ZFI_PROC_TYPE` on `ms_instance-process_type` in both methods.
   Getter methods simply return the private members.

4. **Validation location**: BAL object/subobject existence checked in
   `ZCL_FI_PROCESS_MANAGER=>create_process` before calling `zcl_fi_process_instance=>create()`.
   Uses `SELECT COUNT( * )` from `BALOBJCT` (SLG0 BAL object catalog table).
   The validation runs unconditionally — no `IS NOT INITIAL` guard. Empty values are
   prevented at the DDIC level (`NOTNULL` on `ZFI_PROC_TYPE` fields). Any zero-count
   result raises `ZCX_FI_PROCESS_ERROR` with `textid = validation_failed` and
   `value = |BAL object/subobject '{ obj }/{ subobj }' not found in SLG0|`.

5. **One subobject per process type**: All steps in a process instance (including bgRFC
   substeps) log to the same configured subobject. Steps are distinguished by external
   log ID, not by subobject.

6. **bgRFC substep logger**: After this change uses `lo_instance->get_bal_object()` and
   `lo_instance->get_bal_subobject()` instead of hardcoded `'ZFI_PROCESS'`/`'CORE'`.

7. **Health check test types**: All `TEST_%` process types get `bal_object = 'ZFI_PROCESS'`
   / `bal_subobject = 'TESTS'`. The BALI filter in `check_logger_integration`,
   `check_bgrfc_logger`, `check_bgrfc_error_propagation` changes from `gc_bal_subobject`
   (`'CORE'`) to `gc_bal_subobj_tests` (`'TESTS'`).

8. **`ZCL_FI_PROCESS_LOGGER` defaults**: Not changed. The defaults `'ZFI_PROCESS'`/`'CORE'`
   remain, but all actual callers now pass explicit values so defaults are never used.

---

## Implementation Plan

### Tasks

#### DDIC Changes (planner repo — activate first)

**Task 1: Create two new data elements**

- `ZFI_PROCESS_BAL_OBJECT`: data element, domain `BALOBJ_D`, short text "BAL Object"
- `ZFI_PROCESS_BAL_SUBOBJ`: data element, domain `BALSUBOBJ`, short text "BAL Subobject"

abapGit XML files to create:
- `planner/src/zfi_process_bal_object.dtel.xml`
- `planner/src/zfi_process_bal_subobj.dtel.xml`

> **Transport:** Add both data elements to the workbench transport request for this change.
> Activate in SE11 before activating the `ZFI_PROC_TYPE` table (Task 2).

**Task 2: Extend `ZFI_PROC_TYPE` table**

Add two fields after `ALLOW_DUPLICATE`, before `CREATED_BY`:

```xml
<DD03P>
 <FIELDNAME>BAL_OBJECT</FIELDNAME>
 <ROLLNAME>ZFI_PROCESS_BAL_OBJECT</ROLLNAME>
 <ADMINFIELD>0</ADMINFIELD>
 <NOTNULL>X</NOTNULL>
 <COMPTYPE>E</COMPTYPE>
</DD03P>
<DD03P>
 <FIELDNAME>BAL_SUBOBJECT</FIELDNAME>
 <ROLLNAME>ZFI_PROCESS_BAL_SUBOBJ</ROLLNAME>
 <ADMINFIELD>0</ADMINFIELD>
 <NOTNULL>X</NOTNULL>
 <COMPTYPE>E</COMPTYPE>
</DD03P>
```

> **Activation window risk:** Adding `NOTNULL` fields to an existing table means all
> existing rows are temporarily invalid between table activation (this task) and the
> data-fill step (Task 12). During this window, `ZCL_FI_PROCESS_MANAGER=>create_process`
> will reject any process type whose row has empty `BAL_OBJECT`/`BAL_SUBOBJECT`.
> Plan to execute Task 12 in the same transport/session immediately after table activation
> to minimize this window. Do not activate this table in production without a data
> migration plan ready.

> **Transport:** Add `ZFI_PROC_TYPE` to the same workbench transport request as the
> data elements. The table XML change in abapGit must be committed and activated in SE11
> as part of the same transport.

#### Framework Changes (planner repo)

**Task 3: Extend `ZCL_FI_PROCESS_INSTANCE`**

3a. Add private members to `PRIVATE SECTION`:
```abap
DATA mv_bal_object    TYPE balobj_d.
DATA mv_bal_subobject TYPE balsubobj.
```

3b. Add two public getter methods to `PUBLIC SECTION` (with ABAP-Doc):
```abap
"! Get configured BAL object for this process type
"! @parameter rv_object | BAL object (e.g. ZFI_ALLOC)
METHODS get_bal_object
  RETURNING
    VALUE(rv_object) TYPE balobj_d.

"! Get configured BAL subobject for this process type
"! @parameter rv_subobject | BAL subobject (e.g. PHASE1)
METHODS get_bal_subobject
  RETURNING
    VALUE(rv_subobject) TYPE balsubobj.
```

3c. Add private helper method `load_bal_config` to `PRIVATE SECTION`:
```abap
"! Load BAL object/subobject from ZFI_PROC_TYPE into private members
METHODS load_bal_config.
```

3d. Implement `load_bal_config`:
```abap
METHOD load_bal_config.
  SELECT SINGLE bal_object, bal_subobject
    FROM zfi_proc_type
    INTO ( @mv_bal_object, @mv_bal_subobject )
    WHERE process_type = @ms_instance-process_type.
  IF sy-subrc <> 0.
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING
        textid = zcx_fi_process_error=>validation_failed
        value  = |Process type '{ ms_instance-process_type }' not found in ZFI_PROC_TYPE|.
  ENDIF.
ENDMETHOD.
```

> **Note:** The `IF sy-subrc <> 0` guard is a safety net for `load_instance` (background
> path). In `create_process`, the manager already validates the process type exists before
> this point. The guard ensures `mv_bal_object`/`mv_bal_subobject` are never silently
> left initial if a process type row is missing.

3e. In `initialize_instance`: call `load_bal_config( )` after `ms_instance-process_type` is set
(before logger creation). Replace hardcoded `'ZFI_PROCESS'`/`'CORE'` literals:
```abap
load_bal_config( ).
mo_log = zcl_fi_process_logger=>create_new(
  iv_external_number = CONV balnrext( ms_instance-instance_id )
  iv_object          = mv_bal_object
  iv_subobject       = mv_bal_subobject
).
```

3f. In `load_instance`: call `load_bal_config( )` after SELECT from `ZFI_PROC_INST`.
Replace hardcoded literals similarly:
```abap
load_bal_config( ).
mo_log = zcl_fi_process_logger=>attach_existing(
  iv_external_number = CONV balnrext( ms_instance-instance_id )
  iv_object          = mv_bal_object
  iv_subobject       = mv_bal_subobject
).
```

3g. Implement getter methods (trivial return):
```abap
METHOD get_bal_object.    rv_object    = mv_bal_object.    ENDMETHOD.
METHOD get_bal_subobject. rv_subobject = mv_bal_subobject. ENDMETHOD.
```

**Task 4: Add BAL validation in `ZCL_FI_PROCESS_MANAGER=>create_process`**

The existing `SELECT SINGLE ... FROM zfi_proc_type` block (lines 217–224) already reads
the process type row. Extend that existing SELECT to also fetch `bal_object` and
`bal_subobject` (add them to the `INTO` target) — do **not** add a second SELECT.

Then, immediately after the existing block, add the SLG0 existence check:

```abap
" Validate BAL object/subobject exists in SLG0 (BALOBJCT)
DATA lv_bal_check TYPE i.
SELECT COUNT( * )
  FROM balobjct
  INTO @lv_bal_check
  WHERE balobj    = @lv_proc_type-bal_object
    AND balobjsub = @lv_proc_type-bal_subobject.
IF lv_bal_check = 0.
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid = zcx_fi_process_error=>validation_failed
      value  = |BAL object/subobject '{ lv_proc_type-bal_object }/|
             && |{ lv_proc_type-bal_subobject }' not found in SLG0|.
ENDIF.
```

> **Note:** `lv_proc_type` (or the equivalent local variable / structure used in the
> existing SELECT) must have `bal_object` and `bal_subobject` added to it.
> Since `ZFI_PROC_TYPE` is a DDIC table and `ty_process_type TYPE zfi_proc_type` is
> declared in the manager class, the new fields are automatically available once the
> table is activated — no extra declarations needed beyond adding them to the `INTO` clause.

**Task 5: Update `ZFI_BGRFC_EXEC_SUBSTEP`**

Replace hardcoded literals at lines 34–35:
```abap
" Before (line 34-35):
iv_object          = 'ZFI_PROCESS'
iv_subobject       = CONV balsubobj( 'CORE' )

" After:
iv_object          = lo_instance->get_bal_object( )
iv_subobject       = lo_instance->get_bal_subobject( )
```

Note: `lo_instance` is already loaded at this point (line 22). The getter call is safe.

**Task 6: Update `ZCL_FIPROC_HEALTH_CHK_QUERY`**

6a. In `ensure_test_type_defs`: Add `bal_object` and `bal_subobject` to every entry in
the `lt_types` VALUE constructor. All test process types use
`bal_object = gc_bal_object` (`'ZFI_PROCESS'`) / `bal_subobject = gc_bal_subobj_tests` (`'TESTS'`).

6b. In `check_logger_integration` (line 1868): change `subobject = gc_bal_subobject`
to `subobject = gc_bal_subobj_tests`.

6c. In `check_bgrfc_logger` (line 2028): change `subobject = gc_bal_subobject`
to `subobject = gc_bal_subobj_tests`.

6d. In `check_bgrfc_error_propagation` (line 2264): change `subobject = gc_bal_subobject`
to `subobject = gc_bal_subobj_tests`.

#### Data Migration / Setup

**Task 12: Update existing process type registrations**

Existing `ZFI_PROC_TYPE` records (real allocation process types, not test types) must
be updated with correct `BAL_OBJECT`/`BAL_SUBOBJECT` values before go-live. This is a
manual customizing step (SE16 or SPRO). Suggested values:

| PROCESS_TYPE | BAL_OBJECT | BAL_SUBOBJECT |
|---|---|---|
| `ZFI_ALLOC_PHASE1` (or equivalent) | `ZFI_ALLOC` | `PHASE1` |
| `ZFI_ALLOC_PHASE2` | `ZFI_ALLOC` | `PHASE2` |
| `ZFI_ALLOC_PHASE3` | `ZFI_ALLOC` | `PHASE3` |
| `ZFI_ALLOC_CORR_BCHE` | `ZFI_ALLOC` | `CORR_BCHE` |

Note: Exact process type names must be verified against the real system's `ZFI_PROC_TYPE` table.

> **Transport/client strategy:** The delivery class of `ZFI_PROC_TYPE` must be checked
> before deciding how to transport these data changes:
> - If delivery class **C** (customizing) — records are client-dependent, maintained per
>   client via SE16, and transported via a **customizing transport request** (transaction SM30
>   or equivalent). Include this step in the transport order.
> - If delivery class **A** (application) — records are client-independent; changes are
>   made in the development client and transported in a **workbench transport request**.
> - If delivery class **E** (system table) — records must be maintained in each client
>   individually and are **not transported**.
>
> Check the table delivery class in SE11 → `ZFI_PROC_TYPE` → Delivery and Maintenance
> tab before raising the transport request.

### Acceptance Criteria

1. **AC1 - DDIC**: Two new data elements exist and are activated. `ZFI_PROC_TYPE` has
   `BAL_OBJECT` and `BAL_SUBOBJECT` fields (mandatory, non-nullable).

2. **AC2 - Validation**: Attempting to `create_process` for a process type whose
   `BAL_OBJECT`/`BAL_SUBOBJECT` combination does not exist in SLG0 raises
   `ZCX_FI_PROCESS_ERROR` with descriptive message before the instance is created.
   - **Given** a `ZFI_PROC_TYPE` row with `BAL_OBJECT = 'ZFI_FAKE'` / `BAL_SUBOBJECT = 'NONE'`
     (not registered in SLG0)
   - **When** `ZCL_FI_PROCESS_MANAGER=>create_process` is called with that process type
   - **Then** `ZCX_FI_PROCESS_ERROR` is raised, `value` contains `'ZFI_FAKE/NONE'`,
     and no `ZFI_PROC_INST` row is created

3. **AC3 - Framework logger**: After executing a real allocation process, SLG1 shows
   logs under the configured object/subobject (not `ZFI_PROCESS`/`CORE`).
   - **Given** a process type in `ZFI_PROC_TYPE` with `BAL_OBJECT = 'ZFI_ALLOC'` /
     `BAL_SUBOBJECT = 'PHASE1'`
   - **When** the process is executed end-to-end via the test program `ZFI_ALLOC_TEST_PHASE1`
   - **Then** SLG1 (transaction) shows log entries under object `ZFI_ALLOC` / subobject `PHASE1`,
     and no entries under `ZFI_PROCESS`/`CORE`

4. **AC4 - bgRFC substep logger**: bgRFC substep logs appear under the configured
   object/subobject in SLG1.
   - **Given** a PHASE2 process type with configured `BAL_OBJECT`/`BAL_SUBOBJECT`
   - **When** the PHASE2 process is executed (which uses bgRFC substeps)
   - **Then** SLG1 shows substep log entries under the configured object/subobject
     (not `ZFI_PROCESS`/`CORE`)

5. **AC5 - Health check**: Health check dashboard shows GREEN for `LOGGER_INTEGRATION`,
   `BGRFC_LOGGER`, and `BGRFC_ERROR_PROP` tests after this change.
   - **Given** the health check test types (`TEST_%`) have `BAL_OBJECT = 'ZFI_PROCESS'` /
     `BAL_SUBOBJECT = 'TESTS'` in `ZFI_PROC_TYPE`
   - **When** `ZCL_FIPROC_HEALTH_CHK_QUERY` is executed (or Fiori health check opened)
   - **Then** all three tests show GREEN status; no BALI filter mismatch causes FALSE NEGATIVE

6. **AC6 - No regression**: All existing health check tests remain GREEN.

---

## Additional Context

### Dependencies

- `ZFI_PROCESS_BAL_OBJECT` and `ZFI_PROCESS_BAL_SUBOBJ` data elements must exist and
  be activated **before** `ZFI_PROC_TYPE` table can be activated.
- `ZFI_PROC_TYPE` table must be activated before `ZCL_FI_PROCESS_INSTANCE` compiles
  (as it reads the new fields in `load_bal_config`).
- `ZCL_FI_PROCESS_INSTANCE` getter methods must be available before any step class
  in ovysledovka is activated.
- SLG0 entries for `ZFI_ALLOC`/`PHASE1` etc. must exist in the target system.
  These are assumed to exist (same as `c_bal_*` constants in `ZCL_FI_ALLOCATIONS` imply).

> **Activation window warning:** Between Task 2 (table activation) and Task 12 (data fill),
> existing `ZFI_PROC_TYPE` rows have empty `BAL_OBJECT`/`BAL_SUBOBJECT`. Any attempt to call
> `create_process` during this window will raise an exception. Coordinate Table 2 + Task 12
> to execute in immediate succession — ideally in the same transport release to production.

**Activation order:**
1. `ZFI_PROCESS_BAL_OBJECT` (data element)
2. `ZFI_PROCESS_BAL_SUBOBJ` (data element)
3. `ZFI_PROC_TYPE` (table — adds new fields)
4. `ZCL_FI_PROCESS_INSTANCE` (new private members + getters)
5. `ZCL_FI_PROCESS_MANAGER` (validation logic)
6. `ZFI_BGRFC_EXEC_SUBSTEP` (function module)
7. `ZCL_FIPROC_HEALTH_CHK_QUERY` (health check fixes)

### Testing Strategy

- **Unit (manual)**: Run `ZFI_ALLOC_TEST_PHASE1` after updating `ZFI_PROC_TYPE` with
  `bal_object = 'ZFI_ALLOC'` / `bal_subobject = 'PHASE1'`. Verify SLG1 shows new object.
- **Health check regression**: Open Fiori app / run `ZCL_FIPROC_HEALTH_CHK_QUERY` directly.
  All tests must stay GREEN. Pay special attention to `LOGGER_INTEGRATION`, `BGRFC_LOGGER`,
  `BGRFC_ERROR_PROP` (these use BALI filters that change from `CORE` to `TESTS`).
- **Negative test**: Insert a `ZFI_PROC_TYPE` row with non-existent SLG0 object.
  Call `create_process` — verify exception raised with correct message.
- **bgRFC path**: Run a PHASE2 process end-to-end. Inspect SLG1 for substep logs under
  the configured object/subobject.

### Notes

- **`ZCL_FI_PROCESS_LOGGER` defaults**: The method signature defaults `'ZFI_PROCESS'`/`'CORE'`
  are not changed. They serve as fallback only. After this change they are never reached
  because all callers pass explicit values.

- **`BALOBJCT` table**: This is the SLG0 catalog table for BAL object/subobject combinations.
  The SELECT uses `balobj = @lv_bal_object AND balobjsub = @lv_bal_subobject` to check
  existence. This is a system table — no Z prefix needed.

- **`ty_process_type` in manager**: The type alias `ty_process_type TYPE zfi_proc_type`
  in `ZCL_FI_PROCESS_MANAGER` automatically picks up new fields once `ZFI_PROC_TYPE` is
  extended. The `register_process_type` method also benefits without code changes.
