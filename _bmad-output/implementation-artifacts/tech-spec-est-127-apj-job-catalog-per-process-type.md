---
title: 'APJ Job Catalog per Process Type'
slug: 'est-127-apj-job-catalog-per-process-type'
created: '2026-03-25'
status: 'ready-for-dev'
stepsCompleted: [1, 2, 3, 4]
tech_stack: ['ABAP 7.58', 'SAP APJ Framework', 'IF_APJ_DT_EXEC_OBJECT', 'IF_APJ_RT_EXEC_OBJECT', 'abapGit', 'DDIC', 'CL_APJ_RT_API']
files_to_modify:
  - 'src/zcl_fi_process_job.clas.abap'
  - 'src/zcl_fi_process_job.clas.xml'
  - 'src/zfi_proc_type.tabl.xml'
  - 'src/zcx_fi_process_error.clas.abap'
  - 'src/zcx_fi_process_error.clas.xml'
  - 'src/zfi_process.msag.xml'
files_to_create:
  - 'src/zcl_fi_process_job_base.clas.abap'
  - 'src/zcl_fi_process_job_base.clas.xml'
  - 'src/zcl_fi_process_job_base.clas.testclasses.abap'
  - 'src/zfi_process_job_cat_name.dtel.xml'
  - 'src/zfi_process_job_cat_name.doma.xml'
  - 'src/zcl_fi_process_job_test.clas.abap'
  - 'src/zcl_fi_process_job_test.clas.xml'
  - 'src/zfi_proc_job_test_cat.sajc.json'
  - 'src/zfi_proc_job_test_tmpl.sajt.json'
code_patterns:
  - 'Singleton manager (CREATE PRIVATE + get_instance)'
  - 'Factory methods (CREATE PRIVATE + static create/load)'
  - 'KEY=val;KEY=val parameter serialization with escape handling'
  - 'Schedule-first/status-second optimistic locking for APJ'
  - 'Silent error handling in APJ runtime (no unhandled exceptions)'
  - 'DDIC-first: all structures and types in DDIC'
  - 'abapGit JSON serialization for SAJC/SAJT objects'
test_patterns:
  - 'ABAP Unit in testclasses includes'
  - 'Self-contained tests: upsert process type records in setup(), cleanup in teardown()'
  - 'Direct DB manipulation for status setup (UPDATE zfi_proc_inst SET status)'
  - 'Exception assertion via TRY/CATCH + cl_abap_unit_assert=>assert_equals on t100key'
  - 'Tests in zcl_fi_process_manager.clas.testclasses.abap (7 existing tests)'
linear_issue: 'EST-127'
---

# Tech-Spec: APJ Job Catalog per Process Type

**Created:** 2026-03-25
**Linear Issue:** [EST-127](https://linear.app/smolikzd/issue/EST-127/apj-job-catalog-per-process-type)
**Target Repository:** `cz.imcg.fast.planner`

## Overview

### Problem Statement

Currently, the only way to run a process instance via APJ is through a report that pre-creates the instance with serialized parameters, then schedules the generic `ZFI_PROCESS_JOB_CAT`. Users cannot start process instances directly from the Fiori Application Jobs app with meaningful, typed parameters -- they only see `INSTGUID` and `ACTION`.

### Solution

Introduce a framework pattern where each process type can have a dedicated APJ Job Catalog backed by a subclass of a new `ZCL_FI_PROCESS_JOB_BASE` superclass. The subclass defines the process-type-specific parameters (via `get_parameters()`), maps APJ parameters to the parameter structure, and delegates instance creation + execution to the superclass. A new `JOB_CATALOG_NAME` field on `ZFI_PROC_TYPE` links process types to their catalog entries. All three execution modes coexist with full backward compatibility.

### Scope

**In Scope:**
- New abstract superclass `ZCL_FI_PROCESS_JOB_BASE` (shared lifecycle: create instance, serialize params, execute/restart)
- Refactor existing `ZCL_FI_PROCESS_JOB` to inherit from `ZCL_FI_PROCESS_JOB_BASE` (backward compatible)
- New `JOB_CATALOG_NAME` field on `ZFI_PROC_TYPE`
- Abstract methods for subclass implementation (get_parameters, parameter mapping, process type ID)
- Test subclass with its own job catalog + template for unit/integration testing
- Unit tests extending existing test infrastructure

**Out of Scope:**
- Allocation-specific job catalog (goes in `cz.imcg.fast.ovysledovka` separately)
- Changes to Mode 1 (online) or Mode 2 (APJ via report) execution paths
- Any UI/Fiori development beyond standard Application Jobs app usage
- CDS View `ZFIPROC_I_ProcessType` update — the new `JOB_CATALOG_NAME` field is not added to the CDS view (consistent with other fields like `max_parallel_insts` that are also omitted). Optional follow-up task.

## Context for Development

### Three Execution Modes (Coexisting)

| Mode | Flow | APJ Catalog | Parameter Source |
|------|------|-------------|-----------------|
| Mode 1 -- Online | Report -> create_process() -> execute() | None | Report selection screen |
| Mode 2 -- APJ via Report | Report -> create_process() -> request_execute() -> generic ZFI_PROCESS_JOB_CAT | Generic (INSTGUID + ACTION) | Report selection screen, serialized |
| Mode 3 -- APJ via Fiori (NEW) | Fiori App Jobs -> dedicated catalog -> job class creates instance -> execute | Dedicated per process type | Fiori job scheduling UI |

### Constitution Compliance

- **Principle I -- DDIC-First**: New field on ZFI_PROC_TYPE via DDIC, parameter structures in DDIC
- **Principle III -- Consult SAP Docs**: IF_APJ_DT_EXEC_OBJECT / IF_APJ_RT_EXEC_OBJECT interfaces verified
- **Principle IV -- Factory Pattern**: Base class is `ABSTRACT CREATE PUBLIC` — prevents direct instantiation. `CREATE PUBLIC` is required because the APJ framework instantiates concrete subclasses via `NEW`. This follows the spirit of Principle IV (no accidental instantiation of incomplete objects).
- **Principle V -- Error Handling**: ZCX_FI_PROCESS_ERROR raised with proper context

### Codebase Patterns

**Architecture:**
- `ZCL_FI_PROCESS_MANAGER` is a singleton (`CREATE PRIVATE` + `get_instance()`) -- the central orchestration point
- `ZCL_FI_PROCESS_INSTANCE` uses factory methods (`CREATE PRIVATE` + static `create()`/`load()`)
- `ZCL_FI_PROCESS_JOB` is currently a standalone `CREATE PUBLIC` `FINAL` class -- no inheritance

**Parameter System:**
- Serialization in `zcl_fi_process_manager=>serialize_init_params()` (lines 556-605): RTTS reflection on structure components, sorted by name, escaped `KEY=val;KEY=val` format, max 300 chars
- Deserialization in `zcl_fi_process_instance=>get_init_param_value()` (lines 2278-2355): char-by-char state machine parser with escape handling
- Parameter hash: SHA-256 via `cl_abap_message_digest` for duplicate detection
- Flat projection: first 5 components mapped to `PARAM_VAL_1..5` / `PARAM_LABEL_1..5` via DDIC field labels

**APJ Integration:**
- Design-time: `IF_APJ_DT_EXEC_OBJECT~get_parameters` returns `et_parameter_def` (parameter definitions) and `et_parameter_val` (default values)
- Runtime: `IF_APJ_RT_EXEC_OBJECT~execute` receives `it_parameters` with `SELNAME`/`LOW` flat structure
- Scheduling: `cl_apj_rt_api=>schedule_job()` uses `tt_job_parameter_value` with `NAME` + nested `T_VALUE` (different format from runtime!)
- Job catalog: SAJC JSON with `className` pointing to ABAP class
- Job template: SAJT JSON with `catalogName` + default `singleValueParameters`

**Error Handling:**
- `ZCX_FI_PROCESS_ERROR` has 20 text IDs: constants with msgno 001-019 + 084
- **Message numbering scheme**: The exception class text IDs use msgno 001-019 (sequential for exception constants). The message class `ZFI_PROCESS` uses the same numbers but for different purposes (001-029 = log messages, 040-049 = bgRFC log messages, 060-084 = error messages, 995-999 = utility). The next available msgno for a new exception text ID is **020**.
- APJ job class catches all exceptions silently -- APJ framework never sees unhandled exceptions

### Files to Reference

| File | Purpose | Lines | Impact |
|------|---------|-------|--------|
| `src/zcl_fi_process_job.clas.abap` | Current generic APJ class | 143 | Refactor: remove `FINAL`, add `INHERITING FROM zcl_fi_process_job_base` |
| `src/zcl_fi_process_job.clas.xml` | Class XML metadata | 16 | Update superclass reference |
| `src/zcl_fi_process_manager.clas.abap` | Central manager | 629 | No changes needed -- `create_process()` already accepts `ANY` params |
| `src/zcl_fi_process_instance.clas.abap` | Instance lifecycle | 2637 | No changes needed -- `request_execute()` stays for Mode 2 only |
| `src/zfi_proc_type.tabl.xml` | Process type customizing table | 127 | Add `JOB_CATALOG_NAME` field |
| `src/zfi_process_job_cat.sajc.json` | Generic APJ catalog | 10 | Unchanged (backward compat) |
| `src/zfi_process_job_tmpl.sajt.json` | Generic APJ template | 22 | Unchanged (backward compat) |
| `src/zcx_fi_process_error.clas.abap` | Exception class | 253 | Add new text ID for process type resolution failure |
| `src/zcl_fi_process_manager.clas.testclasses.abap` | Existing unit tests | 246 | Reference for test patterns |

### Technical Decisions

- **Dedicated class per process type (Option B)** -- superclass + subclasses pattern. Each process type that needs Fiori job support gets its own ABAP class + catalog + template.
- **JOB_CATALOG_NAME field on ZFI_PROC_TYPE** -- links process type to its dedicated APJ catalog. Used for reverse lookup (catalog -> process type).
- **APJ parameters serialized to KEY=val as usual** -- Mode 3 uses same serialization as Mode 2. The base class maps APJ parameters to a DDIC structure, then calls `create_process()` which serializes as usual. Steps work identically regardless of execution mode.
- **Existing generic ZCL_FI_PROCESS_JOB remains backward compatible** -- refactored to inherit from base but behavior unchanged.
- **Mode 3 bypasses request_execute()** -- the dedicated job class handles the full lifecycle (create instance + execute) directly. `request_execute()` remains exclusively for Mode 2.
- **Base class is ABSTRACT** -- `ZCL_FI_PROCESS_JOB_BASE` is declared `ABSTRACT CREATE PUBLIC`. ABAP 7.40+ fully supports abstract classes that implement interfaces with concrete methods. The `ABSTRACT` keyword prevents accidental direct instantiation while `CREATE PUBLIC` is required because the APJ framework instantiates concrete subclasses via `NEW`. Methods `get_process_type` and `map_parameters` are declared as `ABSTRACT METHODS` in the base class — subclasses MUST implement them.
- **CDS View not updated** -- `ZFIPROC_I_ProcessType` already omits several `ZFI_PROC_TYPE` fields (`max_parallel_insts`, `bgrfc_dest_name_inbound`, etc.). The new `JOB_CATALOG_NAME` field is intentionally not added to the CDS view in this scope. The job class reads the DB table directly. CDS update is a separate optional follow-up.

### Key Design: Mode 3 Flow (in the dedicated subclass)

```
IF_APJ_DT_EXEC_OBJECT~get_parameters()
  -> Subclass returns process-type-specific parameter definitions
  -> Fiori renders typed input fields (Company Code, Fiscal Year, etc.)

IF_APJ_RT_EXEC_OBJECT~execute(it_parameters)
  -> Base class extracts ACTION parameter (default 'E')
  -> Base class calls subclass get_process_type() to identify process type
  -> Base class reads ZFI_PROC_TYPE to get PARAMETER_STRUCTURE name
  -> Base class creates dynamic data ref: CREATE DATA lr_params TYPE (lv_param_struct)
  -> Base class calls subclass map_parameters() to convert APJ params to DDIC structure
  -> Base class calls zcl_fi_process_manager=>create_process(
      iv_process_type = <from subclass>
      is_init_params  = <ls_params field-symbol> )
  -> Base class calls lo_instance->execute() or lo_instance->restart()
  -> All exceptions caught silently (same pattern as current generic class)
```

## Implementation Plan

### Tasks

#### Task 1: DDIC -- New domain and data element for JOB_CATALOG_NAME

- File: `src/zfi_process_job_cat_name.doma.xml` (CREATE)
  - Action: Create domain `ZFI_PROCESS_JOB_CAT_NAME`, type CHAR, length 40 (APJ catalog names are max 40 chars)
  - Notes: Follow existing domain pattern (see `zfi_process_jobname.doma.xml` for reference)
- File: `src/zfi_process_job_cat_name.dtel.xml` (CREATE)
  - Action: Create data element `ZFI_PROCESS_JOB_CAT_NAME` referencing the domain
  - Field labels: Short = "Job Catalog", Medium = "Job Catalog Name", Long = "APJ Job Catalog Name"

#### Task 2: DDIC -- Add JOB_CATALOG_NAME field to ZFI_PROC_TYPE

- File: `src/zfi_proc_type.tabl.xml` (MODIFY)
  - Action: Add new field `JOB_CATALOG_NAME` of type `ZFI_PROCESS_JOB_CAT_NAME` after `MAX_PARALLEL_INSTS`
  - Notes: NOT NULL is not required -- empty means "no dedicated catalog" (uses generic Mode 2 path). This is the process type -> catalog linkage.

#### Task 3: Exception class -- Add new text ID for process type resolution

- File: `src/zcx_fi_process_error.clas.abap` (MODIFY)
  - Action: Add constant `process_type_not_resolved` with **msgno = '020'** (next sequential after existing 019)
  - Purpose: Raised when the base job class cannot determine process type from `JOB_CATALOG_NAME` reverse lookup
- File: `src/zcx_fi_process_error.clas.xml` (MODIFY)
  - Action: Add corresponding T100 message text: "Process type could not be resolved for job catalog &1"
- File: `src/zfi_process.msag.xml` (MODIFY)
  - Action: Add message number 020 with text "Process type could not be resolved for job catalog &1"
  - Notes: Exception text IDs use msgno 001-019 (sequential). This continues that sequence. Do NOT use the 060-084 range (those are error messages used directly by the message class, not by exception text ID constants).

#### Task 4: Create ZCL_FI_PROCESS_JOB_BASE superclass

- File: `src/zcl_fi_process_job_base.clas.abap` (CREATE)
- File: `src/zcl_fi_process_job_base.clas.xml` (CREATE)
- Action: Create the superclass with the following structure:

```abap
CLASS zcl_fi_process_job_base DEFINITION
  PUBLIC
  ABSTRACT
  CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES if_apj_dt_exec_object.
    INTERFACES if_apj_rt_exec_object.

    CONSTANTS gc_action_execute TYPE c LENGTH 1 VALUE 'E'.
    CONSTANTS gc_action_restart TYPE c LENGTH 1 VALUE 'R'.
    CONSTANTS gc_param_action TYPE c LENGTH 6 VALUE 'ACTION'.

  PROTECTED SECTION.
    "! Return the process type this job class handles.
    "! @parameter rv_process_type | Process type identifier
    METHODS get_process_type ABSTRACT
      RETURNING VALUE(rv_process_type) TYPE zfi_process_proc_type.

    "! Map APJ runtime parameters to a parameter structure for create_process().
    "! @parameter it_parameters | APJ runtime parameters (SELNAME/LOW format)
    "! @parameter es_params | Mapped parameter structure (ANY -- actual type is subclass-specific)
    "! @raising zcx_fi_process_error | If mapping fails
    METHODS map_parameters ABSTRACT
      IMPORTING it_parameters TYPE if_apj_rt_exec_object=>tt_templ_val
      EXPORTING es_params     TYPE any
      RAISING   zcx_fi_process_error.

    "! Map APJ runtime parameters to additional parameter structure.
    "! Optional -- subclasses redefine only if they use additional params.
    "! Base implementation does nothing (es_params stays initial).
    "! @parameter it_parameters | APJ runtime parameters (SELNAME/LOW format)
    "! @parameter es_params | Mapped additional parameter structure
    "! @raising zcx_fi_process_error | If mapping fails
    METHODS map_additional_parameters
      IMPORTING it_parameters TYPE if_apj_rt_exec_object=>tt_templ_val
      EXPORTING es_params     TYPE any
      RAISING   zcx_fi_process_error.

    "! Extract a single parameter value from APJ runtime parameter table.
    "! Utility method for subclass use.
    "! @parameter it_parameters | APJ runtime parameters
    "! @parameter iv_param_name | Parameter SELNAME to find
    "! @parameter rv_value | The LOW value, or empty string if not found
    METHODS get_apj_param_value
      IMPORTING it_parameters   TYPE if_apj_rt_exec_object=>tt_templ_val
                iv_param_name   TYPE c
      RETURNING VALUE(rv_value) TYPE string.

  PRIVATE SECTION.
    CONSTANTS gc_sts_exec_requested TYPE zfi_process_status VALUE 'EXECREQ'.
    CONSTANTS gc_sts_restart_requested TYPE zfi_process_status VALUE 'RESTREQ'.
ENDCLASS.
```

- **`if_apj_dt_exec_object~get_parameters`**: Empty implementation in the base class. Subclasses MUST redefine this interface method to return their process-type-specific parameter definitions.

- **`if_apj_rt_exec_object~execute`**: The core Mode 3 lifecycle. Uses dynamic data references to handle generic parameter typing:
  1. Extract ACTION parameter from `it_parameters` (default 'E')
  2. Call `get_process_type()` to get process type
  3. Read `ZFI_PROC_TYPE` to get `PARAMETER_STRUCTURE` name for the process type
  4. **Dynamic typing**: `CREATE DATA lr_params TYPE (lv_param_struct)` to create a data reference of the correct DDIC structure type
  5. `ASSIGN lr_params->* TO <ls_params>` to get a field-symbol for the generic structure
  6. Call `map_parameters( EXPORTING it_parameters = it_parameters IMPORTING es_params = <ls_params> )`
  7. Call `map_additional_parameters()` (optional, may do nothing)
  8. Get manager via `zcl_fi_process_manager=>get_instance()`
  9. Call `lo_manager->create_process( iv_process_type = lv_process_type is_init_params = <ls_params> )`
  10. Based on ACTION: call `lo_instance->execute()` or `lo_instance->restart()`
  11. Catch all `zcx_fi_process_error` and `cx_root` silently (APJ pattern)

- **`get_apj_param_value`**: Utility that loops `it_parameters` looking for `SELNAME = iv_param_name`, returns `LOW` value. Simplifies subclass `map_parameters()` implementation.

- **`get_process_type`**: ABSTRACT — subclasses MUST implement.

- **`map_parameters`**: ABSTRACT — subclasses MUST implement.

- **`map_additional_parameters`**: Concrete with empty body. Subclasses optionally redefine.

#### Task 5: Refactor ZCL_FI_PROCESS_JOB to inherit from base

- File: `src/zcl_fi_process_job.clas.abap` (MODIFY)
  - Action: Change class definition:
    - Remove `FINAL` keyword
    - Add `INHERITING FROM zcl_fi_process_job_base`
    - Remove `INTERFACES if_apj_dt_exec_object` and `INTERFACES if_apj_rt_exec_object` (inherited from base)
    - **REMOVE** the following constants (now inherited from base class): `gc_action_execute`, `gc_action_restart`, `gc_param_action`
    - **REMOVE** the following private constants (now in base class): `gc_sts_exec_requested`, `gc_sts_restart_requested`
    - **KEEP** `gc_param_instguid` (unique to this generic class — not in base)
    - Keep existing `if_apj_dt_exec_object~get_parameters` implementation unchanged (overrides base)
    - Rewrite `if_apj_rt_exec_object~execute` to keep exact same logic but now it's a redefinition of the base method
    - Add `METHODS get_process_type REDEFINITION` and `METHODS map_parameters REDEFINITION` in PROTECTED SECTION (required because base declares them ABSTRACT — the generic class returns initial process type and raises exception respectively, which causes the base execute to exit silently, preserving behavior since this class overrides execute entirely)
  - Notes: The existing behavior is 100% preserved. This class handles Mode 2 (pre-created instances with INSTGUID). It does NOT use `get_process_type()` or `map_parameters()` in its own execute flow -- it has its own complete execute implementation that looks up instance by INSTGUID.

- File: `src/zcl_fi_process_job.clas.xml` (MODIFY)
  - Action: Add `<REFCLSNAME>ZCL_FI_PROCESS_JOB_BASE</REFCLSNAME>` to VS_CLASS section

#### Task 6: Create test subclass ZCL_FI_PROCESS_JOB_TEST

- File: `src/zcl_fi_process_job_test.clas.abap` (CREATE)
- File: `src/zcl_fi_process_job_test.clas.xml` (CREATE)
- Action: Create a concrete test subclass demonstrating the pattern:

```abap
CLASS zcl_fi_process_job_test DEFINITION
  PUBLIC
  INHERITING FROM zcl_fi_process_job_base
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.
    "! Test process type constant
    CONSTANTS gc_process_type TYPE zfi_process_proc_type
      VALUE 'TEST_JOB_CAT'.

    "! APJ parameter names matching ZFI_PROCESS_S_FIN_PARAMS components
    CONSTANTS gc_param_alloc_id TYPE c LENGTH 13 VALUE 'ALLOCATION_ID'.
    CONSTANTS gc_param_comp_code TYPE c LENGTH 12 VALUE 'COMPANY_CODE'.
    CONSTANTS gc_param_fisc_year TYPE c LENGTH 11 VALUE 'FISCAL_YEAR'.
    CONSTANTS gc_param_fisc_period TYPE c LENGTH 13 VALUE 'FISCAL_PERIOD'.

  PROTECTED SECTION.
    METHODS get_process_type REDEFINITION.
    METHODS map_parameters REDEFINITION.

  PRIVATE SECTION.
ENDCLASS.

CLASS zcl_fi_process_job_test IMPLEMENTATION.

  METHOD if_apj_dt_exec_object~get_parameters.
    "! Returns parameter definitions for the test job catalog.
    "! Defines ALLOCATION_ID, COMPANY_CODE, FISCAL_YEAR, FISCAL_PERIOD, and ACTION.
    et_parameter_def = VALUE #(
      ( selname    = gc_param_alloc_id
        kind       = if_apj_dt_exec_object=>parameter
        datatype   = 'C'
        length     = 10
        param_text = 'Allocation ID'
        mandatory_ind = abap_true )
      ( selname    = gc_param_comp_code
        kind       = if_apj_dt_exec_object=>parameter
        datatype   = 'C'
        length     = 4
        param_text = 'Company Code'
        mandatory_ind = abap_true )
      ( selname    = gc_param_fisc_year
        kind       = if_apj_dt_exec_object=>parameter
        datatype   = 'N'
        length     = 4
        param_text = 'Fiscal Year'
        mandatory_ind = abap_true )
      ( selname    = gc_param_fisc_period
        kind       = if_apj_dt_exec_object=>parameter
        datatype   = 'N'
        length     = 3
        param_text = 'Fiscal Period'
        mandatory_ind = abap_true )
      ( selname    = gc_param_action
        kind       = if_apj_dt_exec_object=>parameter
        datatype   = 'C'
        length     = 1
        param_text = 'Action (E=Execute, R=Restart)'
        mandatory_ind = abap_false )
    ).
    et_parameter_val = VALUE #(
      ( selname = gc_param_action
        kind    = if_apj_dt_exec_object=>parameter
        sign    = 'I'  option = 'EQ'
        low     = gc_action_execute )
    ).
  ENDMETHOD.

  METHOD get_process_type.
    "! Returns the test process type identifier.
    rv_process_type = gc_process_type.
  ENDMETHOD.

  METHOD map_parameters.
    "! Maps APJ runtime parameters to ZFI_PROCESS_S_FIN_PARAMS.
    "! @parameter it_parameters | APJ runtime parameters
    "! @parameter es_params | Mapped to ZFI_PROCESS_S_FIN_PARAMS
    DATA ls_params TYPE zfi_process_s_fin_params.
    ls_params-allocation_id  = get_apj_param_value(
      it_parameters = it_parameters iv_param_name = gc_param_alloc_id ).
    ls_params-company_code   = get_apj_param_value(
      it_parameters = it_parameters iv_param_name = gc_param_comp_code ).
    ls_params-fiscal_year    = get_apj_param_value(
      it_parameters = it_parameters iv_param_name = gc_param_fisc_year ).
    ls_params-fiscal_period  = get_apj_param_value(
      it_parameters = it_parameters iv_param_name = gc_param_fisc_period ).
    es_params = ls_params.
  ENDMETHOD.

ENDCLASS.
```

- Notes: Uses existing `ZFI_PROCESS_S_FIN_PARAMS` DDIC structure (already in planner repo). Process type `TEST_JOB_CAT` will be upserted in test setup.

#### Task 7: Create test APJ catalog and template

- File: `src/zfi_proc_job_test_cat.sajc.json` (CREATE)
  - Action: Create APJ catalog pointing to `ZCL_FI_PROCESS_JOB_TEST`
  ```json
  {
    "formatVersion": "1",
    "header": {
      "description": "ZFI Process Test Job Catalog",
      "originalLanguage": "en"
    },
    "generalInformation": {
      "className": "ZCL_FI_PROCESS_JOB_TEST"
    }
  }
  ```

- File: `src/zfi_proc_job_test_tmpl.sajt.json` (CREATE)
  - Action: Create APJ template referencing the test catalog
  ```json
  {
    "formatVersion": "1",
    "header": {
      "description": "ZFI Process Test Job Template",
      "originalLanguage": "en"
    },
    "generalInformation": {
      "catalogName": "ZFI_PROC_JOB_TEST_CAT"
    },
    "parameters": {
      "singleValueParameters": [
        { "name": "ACTION", "value": "E" },
        { "name": "ALLOCATION_ID", "value": "" },
        { "name": "COMPANY_CODE", "value": "" },
        { "name": "FISCAL_YEAR", "value": "" },
        { "name": "FISCAL_PERIOD", "value": "" }
      ]
    }
  }
  ```

#### Task 8: Unit tests for the base class and test subclass

- File: `src/zcl_fi_process_job_base.clas.testclasses.abap` (CREATE)
- Action: Create unit test class with the following tests:

| Test Method | Given | When | Then |
|-------------|-------|------|------|
| `test_get_apj_param_value` | APJ param table with known values | `get_apj_param_value()` called | Correct value returned |
| `test_get_apj_param_missing` | APJ param table without target param | `get_apj_param_value()` called | Empty string returned |
| `test_map_parameters` | APJ params for all 4 fields | Test subclass `map_parameters()` called | Structure populated correctly |
| `test_get_process_type` | Test subclass instance | `get_process_type()` called | Returns `TEST_JOB_CAT` |
| `test_execute_creates_instance` | Valid APJ params, `TEST_JOB_CAT` process type registered with `ZFI_PROCESS_S_FIN_PARAMS` | `if_apj_rt_exec_object~execute()` called on test subclass | New process instance created in `zfi_proc_inst` with correct `process_type = 'TEST_JOB_CAT'` and `parameter_data` containing the serialized KEY=val string. **Assert instance existence and serialized params, NOT final execution status** (status depends on step definitions which are outside this test's scope). |
| `test_execute_action_default` | APJ params without explicit ACTION | execute() called | Defaults to 'E' (execute), instance created |

- **CRITICAL: Test setup MUST populate `parameter_structure`** — Without `parameter_structure = 'ZFI_PROCESS_S_FIN_PARAMS'` on the `TEST_JOB_CAT` process type record, `create_process()` will raise `invalid_parameters` and the test will fail. This is because `create_process()` validates that when `is_init_params` is supplied, the process type must have a `parameter_structure` defined (see `zcl_fi_process_manager` lines 281-287).
- Notes: Follow existing test patterns from `zcl_fi_process_manager.clas.testclasses.abap`:
  - `setup()`: Upsert `TEST_JOB_CAT` process type record with `parameter_structure = 'ZFI_PROCESS_S_FIN_PARAMS'`, `job_catalog_name = 'ZFI_PROC_JOB_TEST_CAT'`
  - `teardown()`: Clean up test instances from `zfi_proc_inst` and `zfi_proc_step`

#### Task 9: Verify backward compatibility

- Action: After all changes, verify:
  1. Existing `ZCL_FI_PROCESS_JOB` compiles without errors
  2. The `if_apj_dt_exec_object~get_parameters` still returns INSTGUID + ACTION
  3. The `if_apj_rt_exec_object~execute` still handles INSTGUID-based execution
  4. All 27 existing health checks pass (run RAP health check in SAP)
  5. Generic `ZFI_PROCESS_JOB_CAT` and `ZFI_PROCESS_JOB_TMPL` unchanged
  6. Existing Mode 1 and Mode 2 flows unaffected
  7. All 7 duplicate check unit tests still pass

### Acceptance Criteria

- [ ] AC 1: Given a subclass of `ZCL_FI_PROCESS_JOB_BASE` with `get_process_type()` returning `TEST_JOB_CAT` and `map_parameters()` mapping APJ params to `ZFI_PROCESS_S_FIN_PARAMS`, when `if_apj_dt_exec_object~get_parameters` is called, then parameter definitions for ALLOCATION_ID, COMPANY_CODE, FISCAL_YEAR, FISCAL_PERIOD, and ACTION are returned with correct types and lengths.

- [ ] AC 2: Given a registered process type `TEST_JOB_CAT` with `parameter_structure = 'ZFI_PROCESS_S_FIN_PARAMS'` and `job_catalog_name = 'ZFI_PROC_JOB_TEST_CAT'`, when `if_apj_rt_exec_object~execute` is called on the test subclass with valid APJ parameters (ALLOCATION_ID=TEST, COMPANY_CODE=1000, FISCAL_YEAR=2026, FISCAL_PERIOD=003), then a new process instance is created in `zfi_proc_inst` with `process_type = 'TEST_JOB_CAT'` and `parameter_data` containing the serialized KEY=val string.

- [ ] AC 3: Given the refactored `ZCL_FI_PROCESS_JOB` inheriting from `ZCL_FI_PROCESS_JOB_BASE`, when `if_apj_dt_exec_object~get_parameters` is called, then it returns exactly INSTGUID (CHAR 32, mandatory) and ACTION (CHAR 1, mandatory, default 'E') -- identical to current behavior.

- [ ] AC 4: Given the refactored `ZCL_FI_PROCESS_JOB`, when `if_apj_rt_exec_object~execute` is called with INSTGUID of a valid EXECREQ instance and ACTION='E', then the instance is executed and reaches COMPLETED or FAILED status -- identical to current behavior.

- [ ] AC 5: Given any exception occurs during Mode 3 execute flow (invalid process type, create_process failure, execute failure), when the base class catches the exception, then it returns silently without raising any exception to the APJ framework.

- [ ] AC 6: Given the `ZFI_PROC_TYPE` table, when a record has `JOB_CATALOG_NAME` populated, then the value matches the SAJC catalog object name for the dedicated job class serving that process type.

- [ ] AC 7: Given all existing unit tests (7 duplicate check tests in `zcl_fi_process_manager.clas.testclasses.abap`), when executed after the refactoring, then all tests pass without modification.

- [ ] AC 8: Given the test subclass APJ catalog `ZFI_PROC_JOB_TEST_CAT` and template `ZFI_PROC_JOB_TEST_TMPL` are deployed to SAP, when a user opens the Fiori Application Jobs app and selects the test template, then input fields for Allocation ID, Company Code, Fiscal Year, and Fiscal Period are displayed.

- [ ] AC 9: Given the base class `get_apj_param_value()` utility method, when called with a parameter name that exists in the APJ parameter table, then the corresponding LOW value is returned. When called with a non-existent parameter name, then an empty string is returned.

- [ ] AC 10: Given APJ runtime parameters without an explicit ACTION parameter, when `if_apj_rt_exec_object~execute` is called on a subclass, then the base class defaults ACTION to 'E' (execute).

## Additional Context

### Dependencies

- No external dependencies beyond existing SAP APJ framework classes
- `CL_APJ_RT_API` and `IF_APJ_DT_EXEC_OBJECT`/`IF_APJ_RT_EXEC_OBJECT` already in use
- `ZCL_FI_PROCESS_MANAGER` and `ZCL_FI_PROCESS_INSTANCE` remain unchanged
- Existing DDIC structure `ZFI_PROCESS_S_FIN_PARAMS` used for test subclass (already in repo)
- Task dependency graph (not strictly linear — some tasks can run in parallel):
  - Tasks 1-2 (DDIC) and Task 3 (exception class) are **independent** — can be done in parallel
  - Task 4 (base class) depends on Tasks 1-2 and Task 3
  - Task 5 (refactor generic class) depends on Task 4
  - Tasks 6-7 (test subclass + catalog/template) depend on Task 4 but are **independent of Task 5** — can be done in parallel with Task 5
  - Task 8 (unit tests) depends on Tasks 5, 6, and 7
  - Task 9 (backward compat verification) depends on all previous tasks

### Testing Strategy

**Unit Tests (automated, ABAP Unit):**
- 6 new tests in `zcl_fi_process_job_base.clas.testclasses.abap` covering:
  - Parameter extraction utility (`get_apj_param_value`)
  - Subclass parameter mapping
  - Process type resolution
  - Full execute lifecycle via test subclass (assert instance creation + serialized params)
  - Action defaulting
- All 7 existing duplicate check tests must continue to pass

**Integration Validation (manual, in SAP):**
- Deploy all objects via abapGit
- Activate DDIC changes (domain, data element, table field)
- Verify `ZFI_PROC_JOB_TEST_CAT` and `ZFI_PROC_JOB_TEST_TMPL` appear in Application Jobs app
- Schedule and run a test job with sample parameters
- Verify process instance created with correct serialized parameters
- Run all 27 RAP health checks -- all must pass

**Backward Compatibility Validation:**
- Generic `ZFI_PROCESS_JOB_CAT` still works for Mode 2 (INSTGUID+ACTION)
- `request_execute()` and `request_restart()` still schedule against `ZFI_PROCESS_JOB_TMPL`
- Allocation reports (`ZFI_ALLOC_PROC_EXEC`, `ZFI_ALLOC_PROC_EXPORT`) unaffected

### Notes

- **Serialization reuse**: The `serialize_init_params()` method is private on `zcl_fi_process_manager`. The base job class uses `create_process()` which calls it internally -- no need to expose serialization separately.
- **300-char limit**: Serialized params have a 300-char max. This is an existing limitation not introduced by this feature. Process types with many or long parameters may hit this limit.
- **Mode 2 isolation**: `zcl_fi_process_instance=>gc_job_template` constant ('ZFI_PROCESS_JOB_TMPL') is only used by `request_execute()` and `request_restart()` -- both Mode 2 only. Mode 3 doesn't touch these methods.
- **Future consideration**: When implementing the allocation-specific job catalog in `cz.imcg.fast.ovysledovka`, create a subclass of `ZCL_FI_PROCESS_JOB_BASE` following the same pattern as `ZCL_FI_PROCESS_JOB_TEST` but mapping to `ZFI_ALLOC_PROCESS_PARAMS` and `ZFI_ALLOC_ADD_PROCESS_PARAMS`.
- **PARAMETER_STRUCTURE still required for Mode 3**: The process type for Mode 3 still needs `PARAMETER_STRUCTURE` populated in `ZFI_PROC_TYPE` because `create_process()` uses it for serialization. The subclass maps APJ params to this structure, and `create_process()` serializes it as usual.

### Naming Convention for APJ Catalogs/Templates

APJ catalog and template names are limited to **40 characters**. The established pattern is:

- Catalog: `ZFI_PROC_JOB_<SUFFIX>_CAT` (prefix = 14 chars + suffix = 4 chars = 22 chars max)
- Template: `ZFI_PROC_JOB_<SUFFIX>_TMPL` (prefix = 14 chars + suffix = 5 chars = 23 chars max)

**Suffix must not exceed ~17 characters** to stay within the 40-char limit. Examples:
- `ZFI_PROC_JOB_TEST_CAT` (21 chars) — test catalog
- `ZFI_PROC_JOB_ALLOC_CAT` (23 chars) — allocation catalog (future)

### Known Limitations

- **Orphaned instances on execute failure**: If `create_process()` succeeds but `execute()` fails in Mode 3, an orphaned instance with status NEW persists in `zfi_proc_inst`. This is the same behavior as Mode 1 (if a report crashes after creating but before executing). No automatic cleanup is provided. Orphaned instances can be identified by status = NEW with no corresponding APJ job run and cleaned up manually or via a periodic housekeeping job (out of scope for this spec).
- **300-char serialization limit**: Existing limitation — not introduced by this feature. Process types with many or long parameters may hit this limit regardless of execution mode.
