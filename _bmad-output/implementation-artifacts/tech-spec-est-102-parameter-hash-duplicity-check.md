---
title: 'Introduce duplicity check based on parameter hash for process type'
slug: 'est-102-parameter-hash-duplicity-check'
created: '2026-03-13'
status: 'done'
completion_date: '2026-03-13'
stepsCompleted: [1, 2, 3, 4, 5]
tech_stack: ['ABAP 7.58', 'SAP DDIC', 'ZFI_PROCESS framework', 'ABAP Unit']
files_to_modify:
  - 'src/zfi_proc_type.tabl.xml'
  - 'src/zcx_fi_process_error.clas.abap'
  - 'src/zcl_fi_process_manager.clas.abap'
  - 'src/zcl_fiproc_health_chk_query.clas.abap'
  - 'src/zcl_fiproc_health_chk_query.clas.testclasses.abap'
  - 'src/zfi_process.msag.xml'
files_to_create:
  - 'src/zfi_proc_duplic_check.doma.xml'
  - 'src/zcl_fi_process_manager.clas.testclasses.abap'
code_patterns:
  - 'DDIC domain: char(1), fixed values, consistent with ZFI_PROCESS_ACTIVE domain pattern'
  - 'Exception text ID: CONSTANTS BEGIN OF ... msgid ZFI_PROCESS msgno NNN END OF pattern'
  - 'Duplicate check: SELECT from ZFI_PROC_INST filtered by PARAMETER_HASH + PROCESS_TYPE + STATUS IN (RUNNING, FAILED)'
  - 'Read ALLOW_DUPLICATE flag from ZFI_PROC_TYPE using SELECT SINGLE in create_process()'
  - 'Catch zcx_fi_process_error from get_instances_by_hash (no-rows = no duplicate = proceed)'
test_patterns:
  - 'ABAP Unit in .testclasses.abap local class ltcl_* FOR TESTING DURATION SHORT RISK LEVEL HARMLESS'
  - 'Given/When/Then comments in test methods'
  - 'setup/teardown with DELETE FROM zfi_proc_inst WHERE process_type LIKE TEST_%'
  - 'New test added to zcl_fiproc_health_chk_query health check suite'
---

# Tech-Spec: Introduce duplicity check based on parameter hash for process type

**Created:** 2026-03-13
**Linear:** EST-102
**Target Repository:** `cz.imcg.fast.planner`

## Overview

### Problem Statement

The framework allows multiple concurrent process instances of the same type with identical parameters (same hash) to be created simultaneously. This risks data integrity issues when parallel instances share the same business data — e.g., two January 2026 allocation runs both in `RUNNING` or `FAILED` state competing over the same records.

### Solution

Add a flag `ALLOW_DUPLICATE` (new DDIC domain `ZFI_PROC_DUPLIC_CHECK`, char 1) to `ZFI_PROC_TYPE`. The check is **active by default** — suppressed only when `ALLOW_DUPLICATE = 'X'`. In `create_process()` of `ZCL_FI_PROCESS_MANAGER`, before creating a new instance, check for any existing instance with the same `PARAMETER_HASH` + `PROCESS_TYPE` in status `RUNNING` or `FAILED`. If found, raise `zcx_fi_process_error` with new text ID `duplicate_instance` (message 016).

### Scope

**In Scope:**
- New DDIC domain `ZFI_PROC_DUPLIC_CHECK` (char 1; `' '` = check active / `'X'` = allow duplicate)
- New field `ALLOW_DUPLICATE` on transparent table `ZFI_PROC_TYPE`
- Duplicate check logic inline in `ZCL_FI_PROCESS_MANAGER=>create_process()`
- New exception text ID `duplicate_instance` in `ZCX_FI_PROCESS_ERROR` (msgno 016)
- New T100 message 016 in message class `ZFI_PROCESS`
- Unit tests: check fires on `RUNNING`, check fires on `FAILED`, bypassed when `ALLOW_DUPLICATE = 'X'`, passes when only `COMPLETED`/`CANCELLED` exist
- Integration health check scenario `DUPLICATE_CHECK` in `ZCL_FIPROC_HEALTH_CHK_QUERY`

**Out of Scope:**
- Checks in `execute_process()` or `restart_process()`
- UI/monitoring changes
- CDS view changes
- Blocking `NEW`/`PENDING`/`QUEUED` statuses

---

## Context for Development

### Technical Preferences & Constraints

- **Constitution Principle I (DDIC-First):** New flag must be a DDIC domain + table field — no local TYPE definitions
- **Constitution Principle IV (Factory Pattern):** `create_process()` is the single factory entry point — check belongs there
- **Constitution Principle V (Error Handling):** Raise `zcx_fi_process_error` with proper `textid` + `value` context string
- `get_instances_by_hash()` raises `zcx_fi_process_error=>instance_not_found` when no rows found — this is the "no duplicate" path; catch it and proceed normally
- Line length ≤ 120 characters
- No new public methods — check logic is 5–8 lines inline in `create_process()`

### Codebase Patterns

- **DDIC domain for flags:** `ZFI_PROCESS_ACTIVE` domain (char 1, fixed values `X`/space) is the reference pattern. New `ZFI_PROC_DUPLIC_CHECK` follows the same XML structure.
- **Table field addition:** `ZFI_PROC_TYPE` fields defined in `<DD03P_TABLE>`. New field appended as last `<DD03P>` entry: no `<KEYFLAG>`, `ROLLNAME` = `ZFI_PROC_DUPLIC_CHECK`, `ADMINFIELD = 0`, `COMPTYPE = E`.
- **Exception text ID pattern:** 15 CONSTANTS blocks (001–015) in `zcx_fi_process_error`, each `msgid = 'ZFI_PROCESS'` + sequential `msgno`. New `duplicate_instance` = msgno **016**, `attr1 = 'VALUE'`, attr2–4 empty.
- **Message class:** `ZFI_PROCESS.msag.xml` — exception msgs occupy 001–015, runtime msgs 020+. New msg **016**: `Duplicate instance found: process type &1, status &2`.
- **create_process() SELECT:** Currently `SELECT SINGLE parameter_structure, add_parameter_structure FROM zfi_proc_type INTO (@lv_param_struct, @lv_add_param_struct) WHERE process_type = @iv_process_type AND active = 'X'`. Extend to also read `allow_duplicate` into a new local variable.
- **Hash query + catch pattern:**
  ```abap
  DATA lt_active TYPE zfi_tt_proc_inst.
  TRY.
    lt_active = get_instances_by_hash(
      iv_parameter_hash = lv_hash
      iv_process_type   = iv_process_type ).
  CATCH zcx_fi_process_error.
    " No instances found — no duplicate, proceed
  ENDTRY.
  IF line_exists( lt_active[ status = gc_status-running ] ) OR
     line_exists( lt_active[ status = gc_status-failed  ] ).
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING
        textid = zcx_fi_process_error=>duplicate_instance
        value  = |Process type { iv_process_type } already has an active instance|
               && | with the same parameters|.
  ENDIF.
  ```
- **Health check test pattern:** `zcl_fiproc_health_chk_query` runs scenarios directly against DB using `TEST_%` process types; cleans up in `cleanup_test_instances`. New `DUPLICATE_CHECK` scenario: create instance via `zcl_fi_process_instance=>create()`, manually UPDATE its status to `RUNNING` in DB, then call `lo_manager->create_process()` for same type/params and expect `zcx_fi_process_error`.
- **Unit test pattern:** Local class `ltcl_*`, `FOR TESTING DURATION SHORT RISK LEVEL HARMLESS`, `setup`/`teardown` with `DELETE FROM zfi_proc_inst WHERE process_type LIKE 'TEST_%'`, Given/When/Then comments.

### Files to Reference

| File | Purpose |
| ---- | ------- |
| `src/zfi_proc_type.tabl.xml` | Add `ALLOW_DUPLICATE` DD03P field entry |
| `src/zfi_proc_duplic_check.doma.xml` | **CREATE NEW** — char(1) domain, fixed values |
| `src/zcx_fi_process_error.clas.abap` | Add `duplicate_instance` CONSTANTS block (msgno 016) |
| `src/zfi_process.msag.xml` | Add message 016 text |
| `src/zcl_fi_process_manager.clas.abap` | Extend SELECT + add inline duplicate check in `create_process()` |
| `src/zcl_fiproc_health_chk_query.clas.abap` | Add `DUPLICATE_CHECK` health check scenario (new private method `check_duplicate`) |
| `src/zcl_fiproc_health_chk_query.clas.testclasses.abap` | Add `test_duplicate_check FOR TESTING` method |
| `src/zcl_fi_process_manager.clas.testclasses.abap` | **CREATE NEW** — unit tests for `create_process()` duplicate logic |
| `src/zcl_fi_process_instance.clas.abap` | Reference only — `gc_status` constants, status values |
| `src/zcl_fi_process_logger.clas.testclasses.abap` | Reference only — unit test structural pattern |
| `src/zfi_process_active.doma.xml` | Reference only — domain XML structure to mirror for new domain |

### Technical Decisions

1. **Flag semantics — inverted:** Field `ALLOW_DUPLICATE` on `ZFI_PROC_TYPE`; blank (default) = check is active; `'X'` = check disabled. Consistent with SAP checkbox convention where `'X'` = true/enabled.
2. **New domain `ZFI_PROC_DUPLIC_CHECK`:** char(1), fixed values `' '` (check active) and `'X'` (allow duplicate). Mirrors `ZFI_PROCESS_ACTIVE` domain XML structure exactly.
3. **Exception message 016:** T100 text `Duplicate instance found: process type &1, status &2`. The `value` attribute carries the human-readable context; &1/&2 placeholders for message display (attr1 = VALUE, attr2 = '' — full context in VALUE string).
4. **No new private method in manager:** Duplicate check is 8–10 lines inline in `create_process()` — self-contained and readable without extraction.
5. **SELECT extension:** The existing `SELECT SINGLE` in `create_process()` is extended from 2 to 3 fields in the `INTO` list — single DB round-trip, no additional read.
6. **Hash source:** `lv_parameter_data` is already computed before the duplicate check (after serialization). Use it as input to `get_instances_by_hash()` — but `get_instances_by_hash()` takes a hash value, not raw data. Must first compute hash via `zcl_fi_process_instance=>create()` path — **problem**: hash is computed inside `initialize_instance()` which is private. **Resolution:** Read the hash from a newly created-but-not-yet-saved instance, OR compute it independently. Better: call `cl_abap_message_digest` directly inline using the same SHA-256 pattern from `generate_parameter_hash()` — but that duplicates logic. **Final decision:** Add a new `CLASS-METHOD get_parameter_hash( iv_parameter_data ) RETURNING rv_hash` as public static method on `ZCL_FI_PROCESS_INSTANCE` so manager can compute the hash without creating an instance.
7. **Checked statuses:** `RUNNING` and `FAILED` only. References `gc_status` constants from `ZCL_FI_PROCESS_MANAGER` (which already defines them identically).
8. **Test split:** Pure unit tests (direct DB manipulation, no full pipeline) → new `zcl_fi_process_manager.clas.testclasses.abap`; integration health check scenario → existing `zcl_fiproc_health_chk_query`.

> **Note on Decision 6:** `generate_parameter_hash()` is currently a private instance method on `ZCL_FI_PROCESS_INSTANCE`. To avoid duplicating SHA-256 logic, it must be promoted to a public static (`CLASS-METHODS`) method. This is a minimal, non-breaking change — existing private call inside the class becomes a self-call to the static method.

---

## Implementation Plan

### Tasks

Tasks are ordered by dependency (DDIC first, then code, then tests).

- [ ] **Task 1: Create DDIC domain `ZFI_PROC_DUPLIC_CHECK`**
  - File: `src/zfi_proc_duplic_check.doma.xml` *(create new)*
  - Action: Create domain XML with `DATATYPE = CHAR`, `LENG = 1`, two fixed values: `' '` (blank, label "Check active") and `'X'` (label "Allow duplicate"). Mirror the structure of `src/zfi_process_active.doma.xml` exactly.
  - Notes: Domain name follows `ZFI_PROC_*` naming convention for process-type-level config.

- [ ] **Task 2: Add `ALLOW_DUPLICATE` field to `ZFI_PROC_TYPE` table**
  - File: `src/zfi_proc_type.tabl.xml` *(modify)*
  - Action: Append a new `<DD03P>` entry inside `<DD03P_TABLE>`, after the existing `ACTIVE` field entry:
    ```xml
    <DD03P>
     <FIELDNAME>ALLOW_DUPLICATE</FIELDNAME>
     <ROLLNAME>ZFI_PROC_DUPLIC_CHECK</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    ```
  - Notes: No `<KEYFLAG>`, no `<NOTNULL>` — blank is valid (means check active). ROLLNAME references the new domain from Task 1.

- [ ] **Task 3: Add message 016 to `ZFI_PROCESS` message class**
  - File: `src/zfi_process.msag.xml` *(modify)*
  - Action: Insert a new `<T100T>` message entry for number `016` after the existing `015` entry:
    ```xml
    <T100T>
     <SPRSL>E</SPRSL>
     <ARBGB>ZFI_PROCESS</ARBGB>
     <MSGNR>016</MSGNR>
     <TEXT>Duplicate instance found: process type &amp;1, status &amp;2</TEXT>
    </T100T>
    ```
  - Notes: Use `&amp;1` and `&amp;2` for XML-escaped placeholders. Message 016 sits between exception msgs (001–015) and runtime msgs (020+).

- [ ] **Task 4: Add `duplicate_instance` text ID to `ZCX_FI_PROCESS_ERROR`**
  - File: `src/zcx_fi_process_error.clas.abap` *(modify)*
  - Action: Add a new CONSTANTS block after the `invalid_parameters` block (line ~164):
    ```abap
    CONSTANTS:
      BEGIN OF duplicate_instance,
        msgid TYPE symsgid VALUE 'ZFI_PROCESS',
        msgno TYPE symsgno VALUE '016',
        attr1 TYPE scx_attrname VALUE 'VALUE',
        attr2 TYPE scx_attrname VALUE '',
        attr3 TYPE scx_attrname VALUE '',
        attr4 TYPE scx_attrname VALUE '',
      END OF duplicate_instance.
    ```
  - Notes: Follows the exact pattern of all 15 existing CONSTANTS blocks. `attr1 = 'VALUE'` binds the `value` data attribute to `&1` in the message text.

- [ ] **Task 5: Promote `generate_parameter_hash()` to public static on `ZCL_FI_PROCESS_INSTANCE`**
  - File: `src/zcl_fi_process_instance.clas.abap` *(modify)*
  - Action (definition): Move `generate_parameter_hash` from `PRIVATE SECTION` to `PUBLIC SECTION` and change `METHODS` to `CLASS-METHODS`:
    ```abap
    "! Compute SHA-256 hash for parameter data string
    "! @parameter iv_parameter_data | Serialized parameter data
    "! @parameter rv_hash | SHA-256 hash value
    CLASS-METHODS get_parameter_hash
      IMPORTING
        iv_parameter_data TYPE zfi_process_parameter_data
      RETURNING
        VALUE(rv_hash) TYPE zfi_process_par_hash
      RAISING
        zcx_fi_process_error.
    ```
  - Action (implementation): Rename the existing `generate_parameter_hash` METHOD block to `get_parameter_hash`. Update the two private callers inside the class (`initialize_instance` line ~421 and `update_parameter_data` line ~2050) to call `get_parameter_hash( ... )` instead.
  - Notes: Method logic is unchanged — only visibility and invocation style change. The new name `get_parameter_hash` is more consistent with the `get_*` naming convention of other public methods.

- [ ] **Task 6: Add duplicate check logic to `ZCL_FI_PROCESS_MANAGER=>create_process()`**
  - File: `src/zcl_fi_process_manager.clas.abap` *(modify)*
  - Action (definition — PRIVATE SECTION): No new methods needed.
  - Action (implementation): In `METHOD create_process`, extend the existing `SELECT SINGLE` to also read `allow_duplicate`:
    ```abap
    DATA lv_allow_duplicate TYPE zfi_proc_duplic_check.
    SELECT SINGLE parameter_structure, add_parameter_structure, allow_duplicate
      FROM zfi_proc_type
      INTO ( @lv_param_struct, @lv_add_param_struct, @lv_allow_duplicate )
      WHERE process_type = @iv_process_type
        AND active       = 'X'.
    ```
    Then, after the parameter serialization block (after `lv_parameter_data` is computed) and before the `zcl_fi_process_instance=>create(...)` call, insert the duplicate check:
    ```abap
    " Duplicate check: active by default; skip only if ALLOW_DUPLICATE = 'X'
    IF lv_allow_duplicate <> 'X'.
      DATA lv_hash TYPE zfi_process_par_hash.
      lv_hash = zcl_fi_process_instance=>get_parameter_hash(
                  iv_parameter_data = lv_parameter_data ).
      DATA lt_existing TYPE zfi_tt_proc_inst.
      TRY.
          lt_existing = get_instances_by_hash(
            iv_parameter_hash = lv_hash
            iv_process_type   = iv_process_type ).
        CATCH zcx_fi_process_error.
          " No instances with this hash — no duplicate, continue
      ENDTRY.
      IF line_exists( lt_existing[ status = gc_status-running ] ) OR
         line_exists( lt_existing[ status = gc_status-failed  ] ).
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>duplicate_instance
            value  = |Process type { iv_process_type } already has an active instance|
                   && | with the same parameters (RUNNING or FAILED)|.
      ENDIF.
    ENDIF.
    ```
  - Notes: `lv_parameter_data` is already populated by the serialization block above this point. If no parameters were passed, `lv_parameter_data` is initial — hash of empty string is computed, which is correct (two no-param instances share the same hash). Line lengths must remain ≤ 120 chars — verify the string template lines. `gc_status-running` and `gc_status-failed` are already defined as private constants in the manager class.

- [ ] **Task 7: Create unit tests in `ZCL_FI_PROCESS_MANAGER.clas.testclasses.abap`**
  - File: `src/zcl_fi_process_manager.clas.testclasses.abap` *(create new)*
  - Action: Create local test class `ltcl_duplicate_check_test` with the following test methods:
    - `test_blocks_when_running` — Given a `TEST_DUPCHK` instance exists with status `RUNNING`, when `create_process()` is called with same process type and same parameters, then `zcx_fi_process_error=>duplicate_instance` is raised.
    - `test_blocks_when_failed` — Same as above but existing instance has status `FAILED`.
    - `test_allows_when_completed` — Given a `TEST_DUPCHK` instance exists with status `COMPLETED`, when `create_process()` is called with same type/params, then no exception is raised and a new instance is returned.
    - `test_allows_when_flag_set` — Given `ZFI_PROC_TYPE` has `ALLOW_DUPLICATE = 'X'` for `TEST_DUPCHK_FREE`, when `create_process()` is called and a RUNNING instance exists, then no exception is raised.
    - `test_allows_different_params` — Given a RUNNING instance exists for param set A, when `create_process()` is called with different param set B (different hash), then no exception is raised.
  - Notes: `setup` creates required `ZFI_PROC_TYPE` entries (or verifies they exist); `teardown` does `DELETE FROM zfi_proc_inst WHERE process_type IN ('TEST_DUPCHK', 'TEST_DUPCHK_FREE')` and `DELETE FROM zfi_proc_type WHERE process_type IN ('TEST_DUPCHK', 'TEST_DUPCHK_FREE')`. For `test_allows_when_flag_set`, temporarily INSERT a `ZFI_PROC_TYPE` row with `ALLOW_DUPLICATE = 'X'` and clean up in teardown.

- [ ] **Task 8: Add `DUPLICATE_CHECK` health check scenario to `ZCL_FIPROC_HEALTH_CHK_QUERY`**
  - File: `src/zcl_fiproc_health_chk_query.clas.abap` *(modify)*
  - Action (definition — PRIVATE SECTION): Add `check_duplicate` to the private METHODS list after `check_bgrfc_error_propagation`.
  - Action (dispatch — `build_rows` method): Add the following IF block after the `BGRFC_ERROR_PROP` block (line ~239):
    ```abap
    IF mv_filter_test_id IS INITIAL OR mv_filter_test_id = 'DUPLICATE_CHECK'.
      check_duplicate( ).
    ENDIF.
    ```
  - Action (implementation — new private METHOD `check_duplicate`): Populate all three UI explanation fields at the top, then execute two sub-scenarios:
    ```abap
    METHOD check_duplicate.
      DATA ls_row TYPE ty_row.
      DATA: lv_start_ts TYPE timestampl,
            lv_end_ts   TYPE timestampl.

      ls_row-capabilityid = 'DUPLICATE_CHECK'.
      ls_row-description  = 'Duplicate instance check: blocks RUNNING/FAILED; allows COMPLETED'.
      ls_row-testlogic    = 'Creates TEST_DUPCHECK instance, sets status=RUNNING directly in DB,'
                          && ' then calls create_process() with same params — expects'
                          && ' zcx_fi_process_error=>duplicate_instance to be raised (check'
                          && ' active by default). Also verifies create_process() with different'
                          && ' params (different hash) succeeds without exception.'.

      GET TIME STAMP FIELD lv_start_ts.

      TRY.
          DATA(lo_manager) = zcl_fi_process_manager=>get_instance( ).

          " Sub-scenario 1: existing RUNNING instance blocks creation
          DATA(lo_existing) = zcl_fi_process_instance=>create(
            iv_process_type   = 'TEST_DUPCHECK'
            iv_parameter_data = 'DUPCHECK_PARAM=VALUE_A'
          ).
          DATA(ls_existing) = lo_existing->get_instance( ).
          UPDATE zfi_proc_inst SET status = 'RUNNING'
            WHERE instance_id = @ls_existing-instance_id.

          DATA lv_check_fired TYPE abap_bool VALUE abap_false.
          TRY.
              lo_manager->create_process( iv_process_type = 'TEST_DUPCHECK' ).
            CATCH zcx_fi_process_error INTO DATA(lx_dup).
              IF lx_dup->if_t100_message~t100key = zcx_fi_process_error=>duplicate_instance.
                lv_check_fired = abap_true.
              ENDIF.
          ENDTRY.

          IF lv_check_fired = abap_false.
            ls_row-status  = gc_red.
            ls_row-message = 'Duplicate check did not fire for RUNNING instance'.
            ls_row-statuscriticality = status_to_criticality( ls_row-status ).
            APPEND ls_row TO mt_rows.
            RETURN.
          ENDIF.

          " Sub-scenario 2: different params (different hash) must succeed
          DATA lo_new TYPE REF TO zcl_fi_process_instance.
          lo_new = lo_manager->create_process( iv_process_type = 'TEST_DUPCHECK' ).
          " Note: no params supplied → different hash from VALUE_A instance

          IF lo_new IS NOT BOUND.
            ls_row-status  = gc_red.
            ls_row-message = 'create_process() with different params returned unbound instance'.
          ELSE.
            GET TIME STAMP FIELD lv_end_ts.
            ls_row-durationms = CONV i( ( lv_end_ts - lv_start_ts ) * 1000 ).
            ls_row-status  = gc_green.
            ls_row-message = 'Duplicate check fires for RUNNING; different params allowed'.
          ENDIF.

        CATCH zcx_fi_process_error INTO DATA(lx_err).
          ls_row-status  = gc_red.
          ls_row-message = |ZCX: { lx_err->get_text( ) }|.
        CATCH cx_root INTO DATA(lx_root).
          ls_row-status  = gc_red.
          ls_row-message = |Exception: { lx_root->get_text( ) }|.
      ENDTRY.

      GET TIME STAMP FIELD lv_end_ts.
      IF ls_row-durationms = 0.
        ls_row-durationms = CONV i( ( lv_end_ts - lv_start_ts ) * 1000 ).
      ENDIF.
      ls_row-statuscriticality = status_to_criticality( ls_row-status ).
      APPEND ls_row TO mt_rows.
    ENDMETHOD.
    ```
  - Notes: Process type `TEST_DUPCHECK` is ≤ 12 chars, fits `ZFI_PROCESS_PROC_TYPE` domain. The existing `cleanup_test_instances` method already does `DELETE FROM zfi_proc_inst WHERE process_type LIKE 'TEST_%'` — no additional cleanup needed. The `create_process()` call with no params uses empty `lv_parameter_data`, which produces a different SHA-256 hash than `'DUPCHECK_PARAM=VALUE_A'`, satisfying sub-scenario 2. Note: `TEST_DUPCHECK` must exist in `ZFI_PROC_TYPE` for `create_process()` to succeed — the health check should INSERT a minimal row in setup or the test uses `zcl_fi_process_instance=>create()` directly for the initial instance (which bypasses the manager's type lookup). Adjust: use `zcl_fi_process_instance=>create()` directly for the seed instance (as done in `check_hooks_hash`), then call `lo_manager->create_process()` for the duplicate attempt — but `create_process()` requires a valid `ZFI_PROC_TYPE` entry. **Resolution:** Insert a minimal `ZFI_PROC_TYPE` row for `TEST_DUPCHECK` at start of `check_duplicate` and DELETE it in the cleanup block (add to `cleanup_test_instances` or handle inline with `ROLLBACK WORK` after check completes — but since health check commits are avoided, INSERT without commit and wrap in a local cleanup). **Simpler resolution:** Register `TEST_DUPCHECK` as a permanent test process type in `ZFI_PROC_TYPE` customizing (same as `TEST_HASH`, `TEST_HOOKS_ERR` etc. are expected to exist). Document this as a prerequisite in the method's ABAP-Doc comment.

- [ ] **Task 9: Add `test_duplicate_check` to health check test class**
  - File: `src/zcl_fiproc_health_chk_query.clas.testclasses.abap` *(modify)*
  - Action (definition): In `PRIVATE SECTION` of `ltcl_health_tests`, add after `test_bgrfc_error_propagation`:
    ```abap
    test_duplicate_check FOR TESTING,
    ```
  - Action (implementation): Add after the `test_bgrfc_error_propagation` METHOD block:
    ```abap
    METHOD test_duplicate_check.
      run_test( iv_test_id   = 'DUPLICATE_CHECK'
                iv_test_name = 'Duplicate Instance Check' ).
    ENDMETHOD.
    ```
  - Notes: Follows the exact pattern of all 19 existing test methods. The `run_test` helper asserts: exactly 1 result returned, status = GREEN. The ABAP Unit framework invokes `check_duplicate` via the health check query class, which exercises the full scenario including the `description` + `testlogic` UI fields being populated. Running this unit test in SE80/ATC verifies the health check UI scenario end-to-end.

---

### Acceptance Criteria

- [ ] **AC1:** Given a process type with default configuration (no `ALLOW_DUPLICATE` flag), when `create_process()` is called and an instance with the same parameter hash exists in status `RUNNING`, then `zcx_fi_process_error` is raised with `textid = duplicate_instance`.

- [ ] **AC2:** Given a process type with default configuration, when `create_process()` is called and an instance with the same parameter hash exists in status `FAILED`, then `zcx_fi_process_error` is raised with `textid = duplicate_instance`.

- [ ] **AC3:** Given a process type with `ALLOW_DUPLICATE = 'X'`, when `create_process()` is called and a `RUNNING` instance with the same hash exists, then no exception is raised and a new instance is returned successfully.

- [ ] **AC4:** Given a process type with default configuration, when `create_process()` is called and the only existing instances with the same hash have status `COMPLETED` or `CANCELLED`, then no exception is raised and a new instance is returned successfully.

- [ ] **AC5:** Given a process type with default configuration, when `create_process()` is called with different parameters (different hash) while a `RUNNING` instance exists, then no exception is raised and a new instance is returned successfully.

- [ ] **AC6:** Given a process type with default configuration and no existing instances at all, when `create_process()` is called, then no exception is raised and a new instance is returned successfully (get_instances_by_hash raises → caught → proceed).

- [ ] **AC7:** Given the health check UI is run, when `DUPLICATE_CHECK` scenario executes via `set_filter('DUPLICATE_CHECK') → build_rows()`, then: (a) the result row has `status = GREEN`, (b) `capabilityid = 'DUPLICATE_CHECK'`, (c) `description` and `testlogic` fields are populated with human-readable explanation text visible in the UI, and (d) `durationms > 0`.

- [ ] **AC8:** The new domain `ZFI_PROC_DUPLIC_CHECK` activates without errors in SAP DDIC. The `ALLOW_DUPLICATE` field is visible in `ZFI_PROC_TYPE` in SE11.

---

## Additional Context

### Dependencies

- All prerequisite framework work (Sprints 1–4) is complete — no blocking dependencies
- `ZCL_FI_PROCESS_INSTANCE=>get_parameter_hash()` (Task 5) must be completed before Task 6 (manager check logic) and Task 7 (unit tests)
- DDIC objects (Tasks 1–2) must be activated in SAP before ABAP compilation of Task 6
- Message 016 (Task 3) and exception constant (Task 4) must exist before the RAISE in Task 6 compiles cleanly

### Testing Strategy

**Unit Tests** (`zcl_fi_process_manager.clas.testclasses.abap` — Task 7):
- 5 test methods covering: RUNNING blocks, FAILED blocks, COMPLETED allows, flag-disabled allows, different-params allows
- Run via SE80 / ATC: `RUN UNIT TESTS FOR CLASS zcl_fi_process_manager`
- DURATION SHORT, RISK LEVEL HARMLESS — safe for CI pipeline

**Integration Health Check** (`zcl_fiproc_health_chk_query` — Task 8):
- Scenario `DUPLICATE_CHECK` creates real DB records, sets status directly, calls manager
- Run via existing health check UI or `set_filter('DUPLICATE_CHECK') → build_rows() → get_results()`
- Validates end-to-end behavior including DDIC activation

**Manual Verification:**
1. Activate `ZFI_PROC_DUPLIC_CHECK` domain in SE11
2. Activate `ZFI_PROC_TYPE` table in SE11 — verify `ALLOW_DUPLICATE` field visible
3. Run ABAP Unit for `ZCL_FI_PROCESS_MANAGER` — all 5 tests GREEN
4. Run health check `DUPLICATE_CHECK` — GREEN
5. In ZFI_PROC_TYPE customizing, set `ALLOW_DUPLICATE = 'X'` for a test type — verify check is bypassed

### Notes

- **Risk: hash of empty parameters.** If `is_init_params` is not supplied, `lv_parameter_data` is initial (empty string). Two no-param instances of the same type will share the same hash — the duplicate check will correctly block the second. This is the desired behavior.
- **Risk: process type not found.** If the SELECT SINGLE finds no active process type row, `lv_allow_duplicate` will be initial (blank) — check will be active. This is safe; the subsequent `zcl_fi_process_instance=>create()` call will raise `process_type_not_found` anyway.
- **Future consideration:** If `restart_process()` needs to be guarded in the future (a restarted FAILED instance creates a new RUNNING one while another RUNNING exists), that can be added later as a separate check in `restart_process()`.
- **Future consideration:** Translation of message 016 to Czech (currently all messages are in English; Czech translations are deferred per Sprint 4 decision — story 4-6).
