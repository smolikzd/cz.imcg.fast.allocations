---
title: 'Allocation APJ Job Catalogs and Templates'
slug: 'alloc-apj-job-catalogs'
created: '2026-03-26'
status: 'implementation-complete'
stepsCompleted: [1, 2, 3, 4]
tech_stack: ['ABAP 7.58', 'SAP APJ Framework', 'IF_APJ_DT_EXEC_OBJECT', 'IF_APJ_RT_EXEC_OBJECT', 'abapGit', 'DDIC', 'ZCL_FI_PROCESS_JOB_BASE']
files_to_modify:
  # CREATE in cz.imcg.fast.ovysledovka
  - 'src/zfi_alloc_process/zcl_fi_alloc_job_alloc.clas.abap'
  - 'src/zfi_alloc_process/zcl_fi_alloc_job_alloc.clas.xml'
  - 'src/zfi_alloc_process/zcl_fi_alloc_job_export.clas.abap'
  - 'src/zfi_alloc_process/zcl_fi_alloc_job_export.clas.xml'
  - 'src/zfi_alloc_process/zfi_alloc_job_cat.sajc.json'
  - 'src/zfi_alloc_process/zfi_alloc_job_tmpl.sajt.json'
  - 'src/zfi_alloc_process/zfi_alloc_exp_job_cat.sajc.json'
  - 'src/zfi_alloc_process/zfi_alloc_exp_job_tmpl.sajt.json'
  # MODIFY in cz.imcg.fast.ovysledovka
  - 'src/zfi_alloc_process/zfi_alloc_process_params.tabl.xml'
  - 'src/zfi_alloc_process/zfi_alloc_proc_exec.prog.abap'
  - 'src/zfi_alloc_process/zfi_alloc_proc_export.prog.abap'
  - 'src/zfi_alloc_process/zcl_fi_alloc_step_extract.clas.abap'
code_patterns:
  - 'Inherit ZCL_FI_PROCESS_JOB_BASE (FINAL, CREATE PUBLIC)'
  - 'Constants for process type + APJ SELNAME parameter names'
  - 'Redefine get_parameters(), get_process_type(), map_parameters()'
  - 'Optionally redefine map_additional_parameters()'
  - 'DDIC-first: all types via DDIC structures/table types (Constitution Principle I)'
  - 'Factory pattern for process creation (Constitution Principle IV)'
  - 'Error handling via ZCX_FI_PROCESS_ERROR (Constitution Principle V)'
  - 'Parameter serialization: KEY=val;KEY=val format, alphabetical, 300 char limit'
  - 'SAJC/SAJT JSON format for abapGit-managed catalogs/templates'
test_patterns:
  - 'No automated unit tests — APJ integration tested manually via Fiori Application Jobs (F5264)'
  - 'Verify catalog visible, template loads defaults, job runs end-to-end'
  - 'Verify EXPORT skip logic in ZCL_FI_ALLOC_STEP_EXTRACT via process run with EXPORT=initial'
linear_issue: 'EST-129'
depends_on: 'EST-127'
target_repository: 'cz.imcg.fast.ovysledovka'
---

# Tech-Spec: Allocation APJ Job Catalogs and Templates

**Created:** 2026-03-26
**Linear Issue:** [EST-129](https://linear.app/smolikzd/issue/EST-129/allocation-apj-job-catalogs-and-templates)
**Depends On:** [EST-127](https://linear.app/smolikzd/issue/EST-127/apj-job-catalog-per-process-type) (APJ Job Catalog per Process Type -- framework)
**Target Repository:** `cz.imcg.fast.ovysledovka`

## Overview

### Problem Statement

The allocation processes (`ALLOCATIONS` and `ALLOC_EXPORT`) can only be started via two ABAP report programs (`ZFI_ALLOC_PROC_EXEC` and `ZFI_ALLOC_PROC_EXPORT`). These reports create process instances programmatically and schedule them via the generic `ZFI_PROCESS_JOB_CAT` catalog (Mode 2). Users cannot start allocation runs directly from the Fiori **Application Jobs** app with meaningful, typed parameters -- they must run an SE38 report or rely on SM36 to schedule the reports.

### Solution

Create two dedicated APJ job classes -- one per process type -- that inherit from the framework's `ZCL_FI_PROCESS_JOB_BASE` (delivered by EST-127). Each subclass exposes process-type-specific parameters in the Fiori Application Jobs UI, enabling **Mode 3** execution: users schedule allocation jobs directly from Fiori with typed fields.

Each process type gets:
1. **Job class** -- subclass of `ZCL_FI_PROCESS_JOB_BASE`
2. **Job catalog** (`.sajc.json`) -- links class to APJ framework
3. **Job template** (`.sajt.json`) -- defines default parameter values

The ALLOCATIONS process definition is updated to always include the EXPORT step (step 0006); the `ZCL_FI_ALLOC_STEP_EXTRACT` step class checks an EXPORT parameter at runtime and skips gracefully if the flag is not set.

The FORCE_START parameter is not exposed in the APJ templates.

Existing report programs continue to work (Mode 1 online, Mode 2 APJ via report). The new catalogs add Mode 3 as a parallel entry point.

### Scope

**In Scope:**
- Job class `ZCL_FI_ALLOC_JOB_ALLOC` for process type `ALLOCATIONS`
- Job class `ZCL_FI_ALLOC_JOB_EXPORT` for process type `ALLOC_EXPORT`
- Job catalog + template for each (4 artifacts total)
- Register `JOB_CATALOG_NAME` on `ZFI_PROC_TYPE` for both process types (update registration in reports)
- Update `ZFI_ALLOC_PROC_EXEC` to always include EXPORT step in process definition
- Update `ZCL_FI_ALLOC_STEP_EXTRACT` to check export flag parameter and skip if not set
- Map APJ parameters to existing `ZFI_ALLOC_PROCESS_PARAMS` / `ZFI_ALLOC_ADD_PROCESS_PARAMS`

**Out of Scope:**
- Changes to `ZCL_FI_PROCESS_JOB_BASE` (framework, delivered by EST-127)
- Changes to other step classes or process definitions
- Deprecation or removal of existing report programs
- Any Fiori custom UI development

## Context for Development

### Codebase Patterns

**Job Class Pattern** (from `ZCL_FI_PROCESS_JOB_TEST`):
- Inherit `ZCL_FI_PROCESS_JOB_BASE` with `FINAL` + `CREATE PUBLIC`
- Define a constant `gc_process_type` for the process type string
- Define constants `gc_param_*` for each APJ SELNAME (max 8 chars, uppercase)
- Redefine `get_process_type()` → return constant
- Redefine `if_apj_dt_exec_object~get_parameters()` → populate `et_parameter_def` table with one row per APJ field (SELNAME, KIND=`if_apj_dt_exec_object=>parameter`, DATATYPE, LENGTH, PARAM_TEXT, MANDATORY_IND; optionally COMPONENT_TYPE) and `et_parameter_val` with default values
- Redefine `map_parameters()` → read each field from `it_parameters` via SELNAME lookup using `get_apj_param_value()` and populate `es_params` structure (typed to the process's init params DDIC structure)
- Optionally redefine `map_additional_parameters()` → same but for `es_params` typed to the additional params DDIC structure
- Base class handles: instance creation via `ZCL_FI_PROCESS_MANAGER`, serialization, then dispatches on ACTION parameter: `'E'` → `execute()` (synchronous), `'R'` → `restart()` (re-run failed instance synchronously within APJ session)

**Parameter Serialization** (in `ZCL_FI_PROCESS_MANAGER=>serialize_init_params`):
- Uses RTTS (`cl_abap_structdescr`) to enumerate ALL fields of the structure at runtime
- Serializes as `KEY=val;KEY=val` — alphabetically sorted, backslash-escaping for `=`, `;`, `\`
- Both init params and additional params serialized the same way
- Stored in `ZFI_PROC_INST-PARAMETER_DATA` (300 chars) and `ADD_PARAMETER_DATA` (300 chars)
- Adding EXPORT to `ZFI_ALLOC_PROCESS_PARAMS` changes the hash for ALL future instances (accepted trade-off)

**Parameter Deserialization** (in `ZCL_FI_PROCESS_INSTANCE=>get_init_param_value`):
- Char-by-char state-machine parser, case-insensitive key matching
- Searches `parameter_data` first, then `add_parameter_data`
- Returns string value or empty string if key not found

**Process Definition Registration** (report pattern):
- Reports call `create_process()` with a table of `ZFI_PROC_DEF` entries (step number, step class, description)
- ALLOCATIONS currently registers 5 steps conditionally + EXPORT step only when `p_export = abap_true`
- Change: EXPORT step (0006, `ZCL_FI_ALLOC_STEP_EXTRACT`) always registered

**SAJC/SAJT JSON Format** (abapGit-managed, see reference files for exact structure):
- `.sajc.json`: `{ "formatVersion": "1", "header": { "description": "...", "originalLanguage": "en" }, "generalInformation": { "className": "..." } }`
- `.sajt.json`: `{ "formatVersion": "1", "header": { "description": "...", "originalLanguage": "en" }, "generalInformation": { "catalogName": "..." }, "parameters": { "singleValueParameters": [ { "name": "...", "value": "..." } ] } }`
- Template `singleValueParameters` array: each entry has `name` (SELNAME) and `value` (default LOW value)

**Skip Logic Pattern** (for EXPORT step):
- `ZCL_FI_ALLOC_STEP_EXTRACT` reads params in `init()` via `get_init_param_value()`
- In `execute()`, check if EXPORT flag is set; if not, return success immediately (skip gracefully)
- No error raised — the step simply does nothing

### Files to Reference

| File | Repo | Purpose |
| ---- | ---- | ------- |
| `src/zcl_fi_process_job_base.clas.abap` | planner | Abstract superclass — defines `get_parameters()`, `get_process_type()`, `map_parameters()`, `map_additional_parameters()`, `execute()` flow |
| `src/zcl_fi_process_job_test.clas.abap` | planner | **Primary template** — concrete subclass showing exact pattern to follow |
| `src/zcl_fi_process_job_test.clas.xml` | planner | XML metadata template for job class (interface list, descriptions) |
| `src/zfi_proc_job_test_cat.sajc.json` | planner | Reference catalog JSON format |
| `src/zfi_proc_job_test_tmpl.sajt.json` | planner | Reference template JSON format (parameter defaults) |
| `src/zcl_fi_process_manager.clas.abap` | planner | `create_process()` + `serialize_init_params()` — understand serialization |
| `src/zcl_fi_process_instance.clas.abap` | planner | `get_init_param_value()` — char parser for reading params at runtime |
| `src/zcl_fi_process_job.clas.abap` | planner | Generic Mode 2 job class — contrast with Mode 3 pattern |
| `src/zfi_alloc_process/zfi_alloc_proc_exec.prog.abap` | ovysledovka | **MODIFY** — always register EXPORT step, pass JOB_CATALOG_NAME |
| `src/zfi_alloc_process/zfi_alloc_proc_export.prog.abap` | ovysledovka | **MODIFY** — pass JOB_CATALOG_NAME |
| `src/zfi_alloc_process/zcl_fi_alloc_step_extract.clas.abap` | ovysledovka | **MODIFY** — add EXPORT flag check + skip logic |
| `src/zfi_alloc_process/zfi_alloc_process_params.tabl.xml` | ovysledovka | **MODIFY** — add EXPORT field (ABAP_BOOLEAN) |
| `src/zfi_alloc_process/zfi_alloc_add_process_params.tabl.xml` | ovysledovka | Reference — FORCE_START field in additional params |

### Technical Decisions

1. **Two separate job classes** (not one with conditional logic): Each process type has different mandatory parameters and step definitions. Separate classes keep the pattern clean and each catalog shows only relevant parameters in Fiori.

2. **EXPORT step always registered**: The ALLOCATIONS process definition always includes the EXPORT step (step 0006) in `ZFI_PROC_DEF`. The `ZCL_FI_ALLOC_STEP_EXTRACT` step class checks for an EXPORT flag in the init parameters and skips gracefully (returns success without performing work) if the flag is not set.

3. **FORCE_START not in template**: The FORCE_START additional parameter is not exposed in the APJ template. The job class does not override `map_additional_parameters` -- the base class default is called (since both process types register `ZFI_ALLOC_ADD_PROCESS_PARAMS` as their `add_parameter_structure`) and it CLEARs the structure, leaving FORCE_START initial.

4. **ALLOC_EXPORT leaves COMPANY_CODE and ALLOCATION_ID initial**: The export process only populates FISCAL_YEAR and FISCAL_PERIOD in `ZFI_ALLOC_PROCESS_PARAMS`. This matches the current report behavior.

5. **APJ parameter names** (max 8 chars SELNAME limit):
   - `ALLOCATIONS`: `COMPCODE`, `FISCYEAR`, `FISCPER`, `ALLOCID`, `EXPORT`, `ACTION`
   - `ALLOC_EXPORT`: `FISCYEAR`, `FISCPER`, `ACTION`

6. **Naming convention**: Job catalogs follow the established pattern:
   - `ZFI_ALLOC_JOB_CAT` / `ZFI_ALLOC_JOB_TMPL` for ALLOCATIONS
   - `ZFI_ALLOC_EXP_JOB_CAT` / `ZFI_ALLOC_EXP_JOB_TMPL` for ALLOC_EXPORT

## Implementation Plan

### Tasks

Tasks are ordered by dependency — each task builds on the previous. Line number references are approximate and based on the unmodified source; apply tasks sequentially and use context (surrounding code) rather than exact line numbers.

- [x] **Task 1: Add EXPORT field to `ZFI_ALLOC_PROCESS_PARAMS` DDIC structure**
  - File: `src/zfi_alloc_process/zfi_alloc_process_params.tabl.xml`
  - Action: Add a new `<DD03P>` entry for field `EXPORT` after `ALLOCATION_ID`:
    ```xml
    <DD03P>
     <FIELDNAME>EXPORT</FIELDNAME>
     <ROLLNAME>ABAP_BOOLEAN</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <NOTNULL>X</NOTNULL>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    ```
  - Notes: `<NOTNULL>X</NOTNULL>` included to match the pattern used by all 4 existing fields in `ZFI_ALLOC_PROCESS_PARAMS` (COMPANY_CODE, FISCAL_YEAR, FISCAL_PERIOD, ALLOCATION_ID all have NOTNULL). This changes the parameter hash for all future instances (accepted trade-off). Existing instances are unaffected — they were created with the old structure and their serialized strings don't include EXPORT.

- [x] **Task 2: Always register EXPORT step in `ZFI_ALLOC_PROC_EXEC`**
  - File: `src/zfi_alloc_process/zfi_alloc_proc_exec.prog.abap`
  - Action: Replace the conditional EXPORT step registration (lines 118-132) with unconditional inclusion. The COND block:
    ```abap
    "Add export step if selected
    lt_steps = COND #(
      WHEN p_export = abap_true
          THEN VALUE #( ... )
          ELSE lt_steps ).
    ```
    becomes a direct append after step 0005 in the VALUE expression:
    ```abap
    ( process_type  = lc_process_type
      step_number   = '0006'
      step_type     = 'EXPORT'
      description   = 'Export allocation for Keboola'
      class_name    = 'ZCL_FI_ALLOC_STEP_EXTRACT'
      mandatory     = 'X'
      active        = 'X'
      retry_allowed = 'X'
      substep_mode  = 'SERIAL' )
    ```
  - Action: Populate `ls_parameters-export = p_export` alongside the other init params (around line 164-167).
  - Action: Set `ls_process_type-job_catalog_name = 'ZFI_ALLOC_JOB_CAT'` in `setup_process_data` (after line 46, before `io_manager->register_process_type`).
  - Notes: The `p_export` report parameter stays — Mode 1/2 users still use it. The EXPORT field is now always part of init params so the extract step can read it at runtime.

- [x] **Task 3: Update `ZFI_ALLOC_PROC_EXPORT` report**
  - File: `src/zfi_alloc_process/zfi_alloc_proc_export.prog.abap`
  - Action: Set `ls_process_type-job_catalog_name = 'ZFI_ALLOC_EXP_JOB_CAT'` in `setup_process_data` (after line 41, before `io_manager->register_process_type`).
  - Action: Set `ls_parameters-export = abap_true` alongside the other init params (around line 104-105). The ALLOC_EXPORT process always exports, so this flag is always true.
  - Notes: No structural changes — just register the catalog name and populate the new EXPORT field.

- [x] **Task 4: Add EXPORT skip logic to `ZCL_FI_ALLOC_STEP_EXTRACT`**
  - File: `src/zfi_alloc_process/zcl_fi_alloc_step_extract.clas.abap`
  - Action: In `zif_fi_process_step~init` method, add after line 38:
    ```abap
    DATA(lv_export) = is_context-io_process_instance->get_init_param_value( 'EXPORT' ).
    mv_export = boolc( lv_export IS NOT INITIAL ).
    ```
  - Action: Add instance attribute `mv_export TYPE abap_boolean` to the private section (after `mv_force_start` on line 24).
  - Note: `mv_force_start` is a pre-existing dead field — it is declared in all 5 step classes and populated in `init()` in 4 of them, but never referenced in any `execute()` or `validate()` method. In `ZCL_FI_ALLOC_STEP_EXTRACT` specifically, it is declared but never even populated. Out of scope to clean up here.
  - Action: In `zif_fi_process_step~execute` method, add at the very beginning (before the log_message call on line 64):
    ```abap
    " Skip export if EXPORT flag is not set
    IF mv_export = abap_false.
      log_message(
        iv_message_class  = 'ZFI_ALLOC'
        iv_message_number = '000'
        iv_message_v1     = |EXPORT: Skipped (flag not set)|
      ).
      rs_result-success      = abap_true.
      rs_result-can_continue = abap_true.
      RETURN.
    ENDIF.
    ```
  - Notes: When EXPORT is initial (not set), the step logs a skip message and returns success. The process continues to the next step (or completes if this is the last step). The ALLOC_EXPORT process type always sets EXPORT=true, so this skip only affects ALLOCATIONS runs where the user didn't request export.

- [x] **Task 5: Create `ZCL_FI_ALLOC_JOB_ALLOC` job class**
  - File (CREATE): `src/zfi_alloc_process/zcl_fi_alloc_job_alloc.clas.abap`
  - Action: Create ABAP class inheriting `ZCL_FI_PROCESS_JOB_BASE`, `FINAL`, `CREATE PUBLIC`. Follow exact pattern from `ZCL_FI_PROCESS_JOB_TEST`. Add ABAP-Doc class header comment describing purpose.
  - Constants:
    - `gc_process_type = 'ALLOCATIONS'`
    - `gc_param_comp_code = 'COMPCODE'` (LENGTH 8)
    - `gc_param_fisc_year = 'FISCYEAR'` (LENGTH 8)
    - `gc_param_fisc_period = 'FISCPER'` (LENGTH 7)
    - `gc_param_alloc_id = 'ALLOCID'` (LENGTH 7)
    - `gc_param_export = 'EXPORT'` (LENGTH 6)
  - Redefine `if_apj_dt_exec_object~get_parameters`:
    - COMPCODE: datatype='C', length=4, component_type='BUKRS', param_text='Company Code', mandatory=true
    - FISCYEAR: datatype='N', length=4, param_text='Fiscal Year', mandatory=true
    - FISCPER: datatype='N', length=3, param_text='Fiscal Period', mandatory=true
    - ALLOCID: datatype='C', length=10, param_text='Allocation ID', mandatory=true
    - EXPORT: datatype='C', length=1, param_text='Export to Keboola', mandatory=false
    - ACTION: datatype='C', length=1, param_text='Action (E=Execute, R=Restart)', mandatory=false (inherited constant `gc_param_action` from base)
    - Default values in `et_parameter_val`:
      ```abap
      et_parameter_val = VALUE #(
        ( selname = gc_param_action
          kind    = if_apj_dt_exec_object=>parameter
          sign    = 'I'
          option  = 'EQ'
          low     = gc_action_execute )
      ).
      ```
  - Redefine `get_process_type` → return `gc_process_type`
  - Redefine `map_parameters`:
    ```abap
    DATA ls_params TYPE zfi_alloc_process_params.
    ls_params-company_code  = get_apj_param_value( it_parameters = it_parameters iv_param_name = gc_param_comp_code ).
    ls_params-fiscal_year   = get_apj_param_value( it_parameters = it_parameters iv_param_name = gc_param_fisc_year ).
    ls_params-fiscal_period = get_apj_param_value( it_parameters = it_parameters iv_param_name = gc_param_fisc_period ).
    ls_params-allocation_id = get_apj_param_value( it_parameters = it_parameters iv_param_name = gc_param_alloc_id ).
    ls_params-export        = get_apj_param_value( it_parameters = it_parameters iv_param_name = gc_param_export ).
    es_params = ls_params.
    ```
  - Do NOT redefine `map_additional_parameters` — both process types register `ZFI_ALLOC_ADD_PROCESS_PARAMS` as `add_parameter_structure`, so the base class default IS called and CLEARs the structure (FORCE_START stays initial).

- [x] **Task 6: Create `ZCL_FI_ALLOC_JOB_ALLOC` XML metadata**
  - File (CREATE): `src/zfi_alloc_process/zcl_fi_alloc_job_alloc.clas.xml`
  - Action: Create XML metadata following `zcl_fi_process_job_test.clas.xml` pattern:
    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <abapGit version="v1.0.0" serializer="LCL_OBJECT_CLAS" serializer_version="v1.0.0">
     <asx:abap xmlns:asx="http://www.sap.com/abapxml" version="1.0">
      <asx:values>
       <VSEOCLASS>
        <CLSNAME>ZCL_FI_ALLOC_JOB_ALLOC</CLSNAME>
        <LANGU>E</LANGU>
        <DESCRIPT>APJ Job Class - Allocations Process</DESCRIPT>
        <STATE>1</STATE>
        <CLSCCINCL>X</CLSCCINCL>
        <FIXPT>X</FIXPT>
        <UNICODE>X</UNICODE>
        <REFCLSNAME>ZCL_FI_PROCESS_JOB_BASE</REFCLSNAME>
       </VSEOCLASS>
      </asx:values>
     </asx:abap>
    </abapGit>
    ```

- [x] **Task 7: Create `ZCL_FI_ALLOC_JOB_EXPORT` job class**
  - File (CREATE): `src/zfi_alloc_process/zcl_fi_alloc_job_export.clas.abap`
  - Action: Create ABAP class inheriting `ZCL_FI_PROCESS_JOB_BASE`, `FINAL`, `CREATE PUBLIC`. Simpler than ALLOC — fewer parameters. Add ABAP-Doc class header comment describing purpose.
  - Constants:
    - `gc_process_type = 'ALLOC_EXPORT'`
    - `gc_param_fisc_year = 'FISCYEAR'` (LENGTH 8)
    - `gc_param_fisc_period = 'FISCPER'` (LENGTH 7)
  - Redefine `if_apj_dt_exec_object~get_parameters`:
    - FISCYEAR: datatype='N', length=4, param_text='Fiscal Year', mandatory=true
    - FISCPER: datatype='N', length=3, param_text='Fiscal Period', mandatory=true
    - ACTION: datatype='C', length=1, param_text='Action (E=Execute, R=Restart)', mandatory=false
    - Default values in `et_parameter_val`:
      ```abap
      et_parameter_val = VALUE #(
        ( selname = gc_param_action
          kind    = if_apj_dt_exec_object=>parameter
          sign    = 'I'
          option  = 'EQ'
          low     = gc_action_execute )
      ).
      ```
  - Redefine `get_process_type` → return `gc_process_type`
  - Redefine `map_parameters`:
    ```abap
    DATA ls_params TYPE zfi_alloc_process_params.
    ls_params-fiscal_year   = get_apj_param_value( it_parameters = it_parameters iv_param_name = gc_param_fisc_year ).
    ls_params-fiscal_period = get_apj_param_value( it_parameters = it_parameters iv_param_name = gc_param_fisc_period ).
    ls_params-export        = abap_true.  " ALLOC_EXPORT always exports
    es_params = ls_params.
    ```
  - Notes: COMPANY_CODE and ALLOCATION_ID left initial — matches current report behavior. EXPORT is hardcoded to `abap_true` since this process exists solely to export.

- [x] **Task 8: Create `ZCL_FI_ALLOC_JOB_EXPORT` XML metadata**
  - File (CREATE): `src/zfi_alloc_process/zcl_fi_alloc_job_export.clas.xml`
  - Action: Create XML metadata:
    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <abapGit version="v1.0.0" serializer="LCL_OBJECT_CLAS" serializer_version="v1.0.0">
     <asx:abap xmlns:asx="http://www.sap.com/abapxml" version="1.0">
      <asx:values>
       <VSEOCLASS>
        <CLSNAME>ZCL_FI_ALLOC_JOB_EXPORT</CLSNAME>
        <LANGU>E</LANGU>
        <DESCRIPT>APJ Job Class - Export Allocations Process</DESCRIPT>
        <STATE>1</STATE>
        <CLSCCINCL>X</CLSCCINCL>
        <FIXPT>X</FIXPT>
        <UNICODE>X</UNICODE>
        <REFCLSNAME>ZCL_FI_PROCESS_JOB_BASE</REFCLSNAME>
       </VSEOCLASS>
      </asx:values>
     </asx:abap>
    </abapGit>
    ```

- [x] **Task 9: Create job catalog for ALLOCATIONS**
  - File (CREATE): `src/zfi_alloc_process/zfi_alloc_job_cat.sajc.json`
  - Action: Create SAJC JSON:
    ```json
    {
      "formatVersion": "1",
      "header": {
        "description": "Allocations Process Job Catalog",
        "originalLanguage": "en"
      },
      "generalInformation": {
        "className": "ZCL_FI_ALLOC_JOB_ALLOC"
      }
    }
    ```

- [x] **Task 10: Create job template for ALLOCATIONS**
  - File (CREATE): `src/zfi_alloc_process/zfi_alloc_job_tmpl.sajt.json`
  - Action: Create SAJT JSON:
    ```json
    {
      "formatVersion": "1",
      "header": {
        "description": "Allocations Process Job Template",
        "originalLanguage": "en"
      },
      "generalInformation": {
        "catalogName": "ZFI_ALLOC_JOB_CAT"
      },
      "parameters": {
        "singleValueParameters": [
          { "name": "ACTION", "value": "E" },
          { "name": "ALLOCID", "value": "" },
          { "name": "COMPCODE", "value": "" },
          { "name": "EXPORT", "value": "" },
          { "name": "FISCPER", "value": "" },
          { "name": "FISCYEAR", "value": "" }
        ]
      }
    }
    ```
  - Notes: Parameters listed alphabetically (matching the test template convention). EXPORT defaults empty (user can set 'X' to enable). FORCE_START is NOT included — not exposed in APJ.

- [x] **Task 11: Create job catalog for ALLOC_EXPORT**
  - File (CREATE): `src/zfi_alloc_process/zfi_alloc_exp_job_cat.sajc.json`
  - Action: Create SAJC JSON:
    ```json
    {
      "formatVersion": "1",
      "header": {
        "description": "Export Allocations Process Job Catalog",
        "originalLanguage": "en"
      },
      "generalInformation": {
        "className": "ZCL_FI_ALLOC_JOB_EXPORT"
      }
    }
    ```

- [x] **Task 12: Create job template for ALLOC_EXPORT**
  - File (CREATE): `src/zfi_alloc_process/zfi_alloc_exp_job_tmpl.sajt.json`
  - Action: Create SAJT JSON:
    ```json
    {
      "formatVersion": "1",
      "header": {
        "description": "Export Allocations Process Job Template",
        "originalLanguage": "en"
      },
      "generalInformation": {
        "catalogName": "ZFI_ALLOC_EXP_JOB_CAT"
      },
      "parameters": {
        "singleValueParameters": [
          { "name": "ACTION", "value": "E" },
          { "name": "FISCPER", "value": "" },
          { "name": "FISCYEAR", "value": "" }
        ]
      }
    }
    ```
  - Notes: Only 3 parameters — fiscal year, period, and action. No COMPCODE, ALLOCID, or EXPORT (EXPORT is hardcoded in the job class).

### Acceptance Criteria

- [ ] **AC 1:** Given the DDIC structure `ZFI_ALLOC_PROCESS_PARAMS` is activated, when inspecting its fields in SE11, then it contains 5 fields: COMPANY_CODE, FISCAL_YEAR, FISCAL_PERIOD, ALLOCATION_ID, EXPORT.

- [ ] **AC 2:** Given a user runs `ZFI_ALLOC_PROC_EXEC` with `p_export = abap_false`, when the process definition is created, then step 0006 (EXPORT / `ZCL_FI_ALLOC_STEP_EXTRACT`) is always present in `ZFI_PROC_DEF`.

- [ ] **AC 3:** Given a process instance of type ALLOCATIONS with EXPORT=initial, when `ZCL_FI_ALLOC_STEP_EXTRACT` executes, then it logs "EXPORT: Skipped (flag not set)" and returns `success=true, can_continue=true` without performing any FTP/extraction work.

- [ ] **AC 4:** Given a process instance of type ALLOCATIONS with EXPORT='X', when `ZCL_FI_ALLOC_STEP_EXTRACT` executes, then it performs the Keboola extraction as before (no behavioral change from current production).

- [ ] **AC 5:** Given the job catalog `ZFI_ALLOC_JOB_CAT` is imported, when a user opens Fiori Application Jobs (F5264) and selects this catalog, then 6 parameters are visible: COMPCODE, FISCYEAR, FISCPER, ALLOCID, EXPORT, ACTION.

- [ ] **AC 6:** Given the job catalog `ZFI_ALLOC_EXP_JOB_CAT` is imported, when a user opens Fiori Application Jobs (F5264) and selects this catalog, then 3 parameters are visible: FISCYEAR, FISCPER, ACTION.

- [ ] **AC 7:** Given a user schedules a job via `ZFI_ALLOC_JOB_CAT` with COMPCODE=1000, FISCYEAR=2026, FISCPER=003, ALLOCID=9, EXPORT='X', ACTION='E', when the job runs, then a process instance of type ALLOCATIONS is created with all 6 steps (including EXPORT) and the process executes to completion.

- [ ] **AC 8:** Given a user schedules a job via `ZFI_ALLOC_EXP_JOB_CAT` with FISCYEAR=2026, FISCPER=003, ACTION='E', when the job runs, then a process instance of type ALLOC_EXPORT is created with 1 step (EXPORT) and the process executes the Keboola extraction.

- [ ] **AC 9:** Given a user schedules a job via `ZFI_ALLOC_JOB_CAT` with ACTION='R', when the job runs, then the base class calls `restart()` on the process instance (synchronous re-run of a failed instance) instead of `execute()`.

- [ ] **AC 10:** Given existing report `ZFI_ALLOC_PROC_EXEC` is run with `p_export = abap_true` and `p_bgrnd = 'X'`, when the process is scheduled, then Mode 2 (APJ via report) still works correctly — no regression.

- [ ] **AC 11:** Given existing report `ZFI_ALLOC_PROC_EXPORT` is run with `p_bgrnd = 'X'`, when the process is scheduled, then Mode 2 still works correctly — no regression.

## Constitution Compliance

| Principle | Applicability | Notes |
| --------- | ------------- | ----- |
| **I — DDIC-First** | Applies | EXPORT field added to DDIC structure `ZFI_ALLOC_PROCESS_PARAMS`. All types via DDIC — no local TYPE definitions in job classes. |
| **II — SAP Standards** | Applies | SAP naming conventions followed (`ZCL_FI_ALLOC_JOB_*`, `ZFI_ALLOC_*_CAT/TMPL`). Line length ≤120 chars. |
| **III — Consult SAP Docs** | Applies | APJ interfaces `IF_APJ_DT_EXEC_OBJECT` / `IF_APJ_RT_EXEC_OBJECT` used per SAP documentation. SAJC/SAJT format per abapGit serializer standard. |
| **IV — Factory Pattern** | Applies | Process creation via `ZCL_FI_PROCESS_MANAGER=>create_process()`. No direct `NEW` instantiation of framework objects. |
| **V — Error Handling** | Applies | Errors raised as `ZCX_FI_PROCESS_ERROR` with context. Extract step skip returns success (not an error — deliberate no-op). |

## Additional Context

### Dependencies

| Dependency | Type | Status | Notes |
| ---------- | ---- | ------ | ----- |
| **EST-127**: `ZCL_FI_PROCESS_JOB_BASE` | Blocking | Completed | Abstract superclass in `cz.imcg.fast.planner`. Must be deployed to the SAP system before EST-129 can be activated. |
| `ZCL_FI_PROCESS_MANAGER` | Runtime | Existing | `create_process()`, `serialize_init_params()`, `execute_process()`, `request_execute_process()` — no changes needed. |
| `ZCL_FI_PROCESS_INSTANCE` | Runtime | Existing | `get_init_param_value()` — used by extract step to read EXPORT flag. No changes needed. |
| `ZFI_PROC_TYPE` table | DDIC | Existing | `JOB_CATALOG_NAME` field must already exist (delivered by EST-127). |
| `ABAP_BOOLEAN` data element | DDIC | SAP standard | Used for EXPORT field type. |
| Fiori app F5264 (Application Jobs) | UI | SAP standard | Must be available in the target system for Mode 3 testing. |

### Testing Strategy

**Manual Integration Tests** (no automated unit tests — APJ integration requires a live SAP system):

**Prerequisite**: Before Mode 3 tests, run each report at least once with `p_del_pc = true` to register the process type with the new `JOB_CATALOG_NAME` field in `ZFI_PROC_TYPE`.

1. **DDIC Activation Test**
   - Activate `ZFI_ALLOC_PROCESS_PARAMS` in SE11
   - Verify 5 fields present, EXPORT is type ABAP_BOOLEAN
   - Verify no activation errors or where-used conflicts

2. **Report Regression Test (Mode 1 — Online)**
   - Run `ZFI_ALLOC_PROC_EXEC` with `p_bgrnd = ' '`, `p_export = abap_false`
   - Verify all 6 steps registered in `ZFI_PROC_DEF`, process runs, EXPORT step skipped
   - Run `ZFI_ALLOC_PROC_EXEC` with `p_bgrnd = ' '`, `p_export = abap_true`
   - Verify process runs, EXPORT step executes Keboola extraction

3. **Report Regression Test (Mode 2 — APJ via Report)**
   - Run `ZFI_ALLOC_PROC_EXEC` with `p_bgrnd = 'X'`
   - Verify background job scheduled, process completes
   - Run `ZFI_ALLOC_PROC_EXPORT` with `p_bgrnd = 'X'`
   - Verify background job scheduled, export completes

4. **Mode 3 — ALLOCATIONS Catalog Test**
   - Open Fiori Application Jobs (F5264)
   - Select catalog `ZFI_ALLOC_JOB_CAT`
   - Verify template `ZFI_ALLOC_JOB_TMPL` loads with correct defaults (ACTION=E)
   - Fill in COMPCODE, FISCYEAR, FISCPER, ALLOCID
   - Schedule and run — verify process instance created and completes

5. **Mode 3 — ALLOC_EXPORT Catalog Test**
   - Open Fiori Application Jobs (F5264)
   - Select catalog `ZFI_ALLOC_EXP_JOB_CAT`
   - Verify template `ZFI_ALLOC_EXP_JOB_TMPL` loads with correct defaults (ACTION=E)
   - Fill in FISCYEAR, FISCPER
   - Schedule and run — verify export executes

6. **EXPORT Skip Logic Test**
   - Via Mode 3, schedule ALLOCATIONS job WITHOUT setting EXPORT
   - Verify step 0006 shows "Skipped (flag not set)" in the process log
   - Verify process completes successfully (no error)

7. **ACTION=R (Restart) Test**
   - Via Mode 3, schedule ALLOCATIONS job with ACTION='R'
   - Verify `restart()` is called on the process instance (synchronous re-run of failed instance)

8. **Duplicate Detection Test (consistent across all modes)**
    - Create a process instance via Mode 3 (ALLOCATIONS catalog) with specific parameters (e.g., COMPCODE=1000, FISCYEAR=2026, FISCPER=003, ALLOCID=9)
    - While the instance is still running (non-completed), attempt to create another instance with the same parameters via Mode 3
    - Verify that `create_process()` rejects the duplicate (since `ALLOW_DUPLICATE` is not set on `ZFI_PROC_TYPE` for ALLOCATIONS)
    - Repeat the same test via Mode 2 (report with `p_bgrnd = 'X'`) — verify duplicate detection also fires
    - Note: Duplicate detection is controlled by `ALLOW_DUPLICATE` on `ZFI_PROC_TYPE`, not by FORCE_START. Mode 2 and Mode 3 instances with identical init params but different `add_parameter_data` (due to FORCE_START hash difference) will NOT collide — they are treated as distinct instances

9. **Error Path: Invalid Parameters Test**
   - Via Mode 3, schedule ALLOCATIONS job with empty mandatory fields (COMPCODE, FISCYEAR, etc.)
   - Verify APJ framework rejects the job (mandatory_ind enforcement)

10. **Error Path: Process Type Not Registered Test**
    - If JOB_CATALOG_NAME is not yet set in `ZFI_PROC_TYPE` (report not yet run with `p_del_pc=true`), attempt Mode 3 scheduling
    - Verify meaningful error from base class, not a dump

### Notes

- **Parameter hash impact**: Adding EXPORT to `ZFI_ALLOC_PROCESS_PARAMS` means the serialized parameter string changes for all new instances. The `find_existing_instance` logic (which uses parameter hash to detect duplicates) will no longer match old instances against new ones. This is the accepted trade-off — old instances are historical and should not be reused.

- **Serialized string length**: With 5 fields in `ZFI_ALLOC_PROCESS_PARAMS`, the worst-case serialized string is approximately: `ALLOCATION_ID=9999999999;COMPANY_CODE=9999;EXPORT=X;FISCAL_PERIOD=999;FISCAL_YEAR=9999` = ~85 chars. Well within the 300-char limit.

- **No FORCE_START in Mode 3**: FORCE_START is intentionally not exposed in the APJ template. **Clarification on duplicate detection**: Duplicate detection is controlled by the `ALLOW_DUPLICATE` field on the `ZFI_PROC_TYPE` table, read by `zcl_fi_process_manager=>create_process()`. Neither allocation report sets `allow_duplicate` when registering the process type, so it defaults to initial — meaning duplicate detection is **active for ALL modes** (1, 2, and 3). FORCE_START is serialized into `add_parameter_data` but is **dead code** — it is declared in step classes but never read in any `execute()` or `validate()` method. Its only functional effect is changing the serialized `add_parameter_data` hash: since Mode 2 (ALLOCATIONS report) sets `force_start = abap_true` while Mode 3 leaves it initial, instances from different modes will not hash-collide, preventing false duplicate matches between modes. There is **no behavioral asymmetry** in duplicate detection between modes — all modes enforce it equally via `ALLOW_DUPLICATE`.

- **ALLOC_EXPORT hardcodes EXPORT=true**: The `ZCL_FI_ALLOC_JOB_EXPORT` class sets `ls_params-export = abap_true` unconditionally in `map_parameters()`. This process type exists solely to export — there's no scenario where you'd run ALLOC_EXPORT without wanting the export. The report `ZFI_ALLOC_PROC_EXPORT` is also updated to set this flag.

- **Future consideration**: If additional process types need APJ catalogs in the future, follow the same pattern — one job class per process type, inheriting `ZCL_FI_PROCESS_JOB_BASE`.
