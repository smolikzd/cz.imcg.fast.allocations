---
title: 'Configurable Max Parallel Instances per Process Type'
slug: 'est-106-max-parallel-instances'
created: '2026-03-16'
status: 'done'
stepsCompleted: [1, 2, 3, 4, 5, 6, 7]
tech_stack: ['ABAP 7.58', 'SAP DDIC', 'ZFI_PROCESS framework', 'SAP HANA']
files_to_modify:
  - 'src/zfi_proc_max_parallel_insts.doma.xml'
  - 'src/zfi_proc_max_parallel_insts.dtel.xml'
  - 'src/zfi_proc_type.tabl.xml'
  - 'src/zcx_fi_process_error.clas.abap'
  - 'src/zcl_fi_process_manager.clas.abap'
  - 'src/zcl_fiproc_health_chk_query.clas.abap'
  - 'src/zfi_process.msag.xml'
code_patterns:
  - 'DDIC-First: domain + dtel before table field'
  - 'Exception constants follow 001-016 sequential numbering; 017 is next'
  - 'Status constant gc_status-running = RUNNING exists in ZCL_FI_PROCESS_MANAGER'
  - 'Parallel check mirrors duplicate check pattern in execute_process()'
  - 'SELECT COUNT(*) WHERE process_type = ... AND status = RUNNING'
test_patterns:
  - 'ZCL_FIPROC_HEALTH_CHK_QUERY: capability row + test logic pattern'
  - 'ensure_test_type_defs() adds TEST_PARLIMIT type with max_parallel_insts = 2'
  - 'build_rows() dispatches to check_parallel_limit() via filter block'
  - 'Test: 2 RUNNING instances -> 3rd execute -> assert parallel_limit_exceeded'
  - 'Test: max_parallel_insts = 0 -> execute -> NOT blocked (unlimited)'
---

# Tech-Spec: Configurable Max Parallel Instances per Process Type

**Created:** 2026-03-16
**Linear:** EST-106
**Target Repository:** `planner` (`cz.imcg.fast.planner`)

## Overview

### Problem Statement

There is no mechanism to limit how many instances of a given process type may run concurrently. The only existing guard (`ALLOW_DUPLICATE`) is a binary, per-parameter-hash duplicate check — it blocks a second instance with the same parameters from being RUNNING or FAILED, but it does not constrain the total count of concurrently executing instances of a type. In operational scenarios where a process is resource-intensive (e.g. PHASE1/PHASE2 allocation runs), running too many in parallel could overload the system.

### Solution

Add a numeric `MAX_PARALLEL_INSTS` field to the `ZFI_PROC_TYPE` customizing table. When an instance is about to be executed (`execute_process()` in the manager), the framework counts currently RUNNING instances of the same process type. If that count is ≥ `MAX_PARALLEL_INSTS` (and `MAX_PARALLEL_INSTS > 0`), a new `parallel_limit_exceeded` exception is raised and execution is blocked. `MAX_PARALLEL_INSTS = 0` means unlimited (check disabled), preserving full backward compatibility.

### Scope

**In Scope:**
- New DDIC domain + data element + field `MAX_PARALLEL_INSTS` on `ZFI_PROC_TYPE`
- Parallel limit check in `ZCL_FI_PROCESS_MANAGER=>execute_process()`
- New exception constant `parallel_limit_exceeded` in `ZCX_FI_PROCESS_ERROR`
- New message 017 in `ZFI_PROCESS` message class
- Self-contained health check test scenario in `ZCL_FIPROC_HEALTH_CHK_QUERY`

**Out of Scope:**
- Replacing or modifying the existing `ALLOW_DUPLICATE` mechanism
- Counting NEW or QUEUED instances toward the limit (only RUNNING counts)
- Any UI/Fiori changes to the process monitor
- Queueing / retry when limit is hit (just raise and stop)

## Context for Development

### Codebase Patterns

**DDIC Conventions:**
- All DDIC objects use SAP naming convention: domain and data element share the same name (`ZFI_PROC_MAX_PARALLEL_INSTS`).
- Closest INT4 precedent: `ZFI_PROC_RUNT_LIMIT` domain (type `INT4`, length 10, no fixed values) and matching dtel. Copy this pattern exactly.
- New field on `ZFI_PROC_TYPE`: name `MAX_PARALLEL_INSTS`, rollname `ZFI_PROC_MAX_PARALLEL_INSTS`, NOT NULL, initial value `0`.

**Exception Class (`ZCX_FI_PROCESS_ERROR`):**
- Constants `001`–`016` are defined sequentially; a gap then exists before `084`.
- Last used sequential slot: `016` = `duplicate_instance`.
- New constant: `parallel_limit_exceeded` → `msgid = 'ZFI_PROCESS'`, `msgno = '017'`, `attr1 = 'VALUE'` (process type name), others empty.
- Raise pattern (copy from `duplicate_instance` raise in `create_process()`):
  ```abap
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid = zcx_fi_process_error=>parallel_limit_exceeded
      value  = ls_type-process_type.
  ```

**Parallel Check Location — `ZCL_FI_PROCESS_MANAGER=>execute_process()`:**
- Current flow (lines 340–343): `DATA(lo_instance) = load_process( iv_instance_id ).` then `lo_instance->execute( ).`
- The local variable holding the instance is named `lo_instance` (not `instance`). All code snippets in Task 6 must use `lo_instance->`.
- New check is inserted **after** `load_process()`, **before** `lo_instance->execute()`. This is a *new* `SELECT SINGLE` from `ZFI_PROC_TYPE` specific to `execute_process()` — there is no existing type-table read in this method to reuse or extend.
- The structural pattern (SELECT SINGLE type config → conditional count check → raise) mirrors the `ALLOW_DUPLICATE` block in `create_process()` (lines 262–283), but it lives in a different method.
- `gc_status-running` constant (`VALUE 'RUNNING'`) already exists in `ZCL_FI_PROCESS_MANAGER` private section (line ~172).

**Health Check Test (`ZCL_FIPROC_HEALTH_CHK_QUERY`):**
- Capability row structure: `ls_row-capabilityid`, `ls_row-description`, `ls_row-testlogic`.
- New test ID: `'PARALLEL_LIMIT'`.
- `ensure_test_type_defs()` (around line 545): add `TEST_PARLIMIT` to `lt_types` VALUE table with `max_parallel_insts = 2`, `allow_duplicate = 'X'`; add step definition referencing `ZCL_FI_STEP_TEST_SUCCESS` (the standard test step class used by all other test types).
- `build_rows()` filter dispatch (around line 250): add block analogous to the `DUPLICATE_CHECK` block.
- New private method `check_parallel_limit()` mirrors `check_duplicate()` (line 2388).

### Files to Reference

| File | Purpose |
| ---- | ------- |
| `src/zfi_proc_runt_limit.doma.xml` | INT4 domain pattern — copy for new `ZFI_PROC_MAX_PARALLEL_INSTS` domain |
| `src/zfi_proc_runt_limit.dtel.xml` | INT4 dtel pattern — copy for new `ZFI_PROC_MAX_PARALLEL_INSTS` dtel |
| `src/zfi_proc_type.tabl.xml` | Table definition — add `MAX_PARALLEL_INSTS` field |
| `src/zcx_fi_process_error.clas.abap` | Exception class — add `parallel_limit_exceeded` constant (msgno 017) |
| `src/zcl_fi_process_manager.clas.abap` | Manager — insert parallel check in `execute_process()`; `gc_status-running` already present |
| `src/zcl_fi_process_instance.clas.abap` | Instance — `execute()` entry reference only; no changes needed here |
| `src/zcl_fiproc_health_chk_query.clas.abap` | Health check class — `check_duplicate()` at line 2388 is the pattern; `ensure_test_type_defs()` at line ~545; `build_rows()` dispatch at line ~250 |
| `src/zfi_process.msag.xml` | Message class — add message 017 text |

### Technical Decisions

- **`MAX_PARALLEL_INSTS = 0` = unlimited**: Zero is the default for new field; skips the check entirely. Backward-compatible — existing process types are unaffected.
- **Check at `execute_process()` not `create_process()`**: Since the constraint is about RUNNING count, it must fire at the moment execution is attempted, not at creation time.
- **RUNNING only counts**: Only instances in status `RUNNING` are counted; NEW/QUEUED/FAILED/COMPLETED are excluded.
- **Independent of `ALLOW_DUPLICATE`**: Both checks remain active simultaneously.
- **Soft check (no ENQUEUE)**: Consistent with existing `ALLOW_DUPLICATE` approach — no DB lock is acquired. A race condition is theoretically possible but accepted as per framework conventions.
- **Check in manager, not in instance**: `execute_process()` in `ZCL_FI_PROCESS_MANAGER` already has access to `ZFI_PROC_TYPE` config and `gc_status` constants. Inserting there keeps `ZCL_FI_PROCESS_INSTANCE` free of config-table reads.

## Implementation Plan

### Tasks

- [x] **Task 1: Create DDIC domain `ZFI_PROC_MAX_PARALLEL_INSTS`**
  - File: `src/zfi_proc_max_parallel_insts.doma.xml` *(new)*
  - Action: Create new file by copying `src/zfi_proc_runt_limit.doma.xml`. Change `<DOMNAME>` to `ZFI_PROC_MAX_PARALLEL_INSTS`, set description to `"Max parallel instances"`. Keep type `INT4`, length `10`, no fixed values.
  - Notes: Must be created before dtel (dependency order).

- [x] **Task 2: Create DDIC data element `ZFI_PROC_MAX_PARALLEL_INSTS`**
  - File: `src/zfi_proc_max_parallel_insts.dtel.xml` *(new)*
  - Action: Create new file by copying `src/zfi_proc_runt_limit.dtel.xml`. Change `<ROLLNAME>` to `ZFI_PROC_MAX_PARALLEL_INSTS`, set `<DOMNAME>` to `ZFI_PROC_MAX_PARALLEL_INSTS`. Set short label `"Max Parallel"`, medium label `"Max Parallel Inst."`, long label `"Max Parallel Instances"`.
  - Notes: Must be created before the table field.

- [x] **Task 3: Add `MAX_PARALLEL_INSTS` field to `ZFI_PROC_TYPE`**
  - File: `src/zfi_proc_type.tabl.xml`
  - Action: Add new `<DD03P>` entry at the end of the field list (after `CHANGED_AT`, which is the current last field). Set `<FIELDNAME>` = `MAX_PARALLEL_INSTS`, `<ROLLNAME>` = `ZFI_PROC_MAX_PARALLEL_INSTS`, `<NOTNULL>` = `X`, `<DEFAULT>` = `0`. Add the new `<DD03P>` block as the last element inside `<DD03P_TABLE>`, after the closing `</DD03P>` of the `CHANGED_AT` entry. No `<POSITION>` tag is needed — the abapGit serializer used in this project omits `<POSITION>` from all field entries (confirmed by reading the existing XML); field order is determined by XML element order.
  - Notes: DDIC table change requires transport. After activation in SE11, run a DB conversion (SE14 → "Activate and Adjust Database") to populate the new column with `0` for all existing rows. Without this step, existing rows will have NULL and the NOT NULL constraint will not be satisfied on the DB level.
- [x] **Task 4: Add message 017 to `ZFI_PROCESS` message class**
  - File: `src/zfi_process.msag.xml`
  - Action: Add new `<T100>` entry with `<MSGNR>` = `017`. Message text (EN): `"Process type & has reached the maximum number of parallel instances"`. The `&` placeholder maps to `attr1` (process type name).
  - Notes: Message text must be ≤ 73 characters per SAP T100 restriction.

- [x] **Task 5: Add `parallel_limit_exceeded` constant to `ZCX_FI_PROCESS_ERROR`**
  - File: `src/zcx_fi_process_error.clas.abap`
  - Action: Add a new constant after the `duplicate_instance` constant block. Use identical structure:
    ```abap
    CONSTANTS:
      parallel_limit_exceeded TYPE sotr_conc VALUE '...' ##NO_TEXT.
    ```
    Add the corresponding `BEGIN OF parallel_limit_exceeded` … `END OF parallel_limit_exceeded` definition with `msgid = 'ZFI_PROCESS'`, `msgno = '017'`, `attr1 = 'VALUE'`, `attr2 = ''`, `attr3 = ''`, `attr4 = ''`.
  - Notes: The `sotr_conc` GUID value must be a valid UUID. In abapGit-based workflows the GUID is stored in the XML and is not auto-replaced on activation — it must be a real UUID that does not collide with existing OTR entries. Generate one using `CL_SYSTEM_UUID=>CREATE_UUID_C32_STATIC( )` in a scratch report, or copy and modify an existing GUID from another constant in this class (increment one hex digit to guarantee uniqueness). Do NOT use a random placeholder string — it will either fail activation or silently create a duplicate OTR entry. Follow the exact formatting of the `duplicate_instance` constant block.

- [x] **Task 6: Insert parallel limit check in `ZCL_FI_PROCESS_MANAGER=>execute_process()`**
  - File: `src/zcl_fi_process_manager.clas.abap`
  - Action: The current method body (lines 340–343) is:
    ```abap
    METHOD execute_process.
      DATA(lo_instance) = load_process( iv_instance_id ).
      lo_instance->execute( ).
    ENDMETHOD.
    ```
    Replace with:
    ```abap
    METHOD execute_process.
      DATA(lo_instance) = load_process( iv_instance_id ).

      " --- Parallel instance limit check (EST-106) ---
      DATA(lv_process_type) = lo_instance->get_process_type( ).

      SELECT SINGLE max_parallel_insts
        FROM zfi_proc_type
        INTO @DATA(lv_max_parallel_insts)
        WHERE process_type = @lv_process_type.

      IF lv_max_parallel_insts > 0.
        SELECT COUNT(*)
          FROM zfi_proc_inst
          INTO @DATA(lv_running_count)
          WHERE process_type = @lv_process_type
            AND status       = @gc_status-running.

        IF lv_running_count >= lv_max_parallel_insts.
          RAISE EXCEPTION TYPE zcx_fi_process_error
            EXPORTING
              textid = zcx_fi_process_error=>parallel_limit_exceeded
              value  = lv_process_type.
        ENDIF.
      ENDIF.
      " --- End parallel instance limit check ---

      lo_instance->execute( ).
    ENDMETHOD.
    ```
  - Notes:
    - Local instance variable is `lo_instance` (confirmed from source). Do not use `instance->`.
    - `get_process_type()` is confirmed as the correct method name on `ZCL_FI_PROCESS_INSTANCE` (line 104 of instance class).
    - `gc_status-running` is confirmed present in the private section (line ~172).
    - `SELECT COUNT(*)` returns type `INT8`; `lv_max_parallel_insts` is `INT4` (from `ZFI_PROC_MAX_PARALLEL_INSTS` domain). Direct comparison (`IF lv_running_count >= lv_max_parallel_insts`) is valid in ABAP 7.58 — no `CONV` needed and consistent with the existing codebase style.
    - This is a new `SELECT SINGLE` from `ZFI_PROC_TYPE` — there is no existing type-table read in `execute_process()` to reuse.
    - Line length must stay ≤ 120 chars (all lines above comply).

- [x] **Task 7: Add `check_parallel_limit()` method to `ZCL_FIPROC_HEALTH_CHK_QUERY`**
  - File: `src/zcl_fiproc_health_chk_query.clas.abap`
  - Action (3 sub-changes):

  **7a — `ensure_test_type_defs()`** (around line 545): Add `TEST_PARLIMIT` and `TEST_PARLIMIT_UNLIM` to `lt_types` VALUE table:
  ```abap
  ( process_type = 'TEST_PARLIMIT'       description = 'Parallel limit test type'
    allow_duplicate = 'X'  max_parallel_insts = 2 )
  ( process_type = 'TEST_PARLIMIT_UNLIM' description = 'Parallel limit test type - unlimited'
    allow_duplicate = 'X'  max_parallel_insts = 0 )
  ```
  **Note:** The snippet above shows only the fields relevant to this feature. Fill in all other mandatory fields (`mandt`, `active`, `created_by`, `created_at`, `changed_by`, `changed_at`, `bal_object`, `bal_subobject`) using the `TEST_DUPCHK` entry (line 547 of `zcl_fiproc_health_chk_query.clas.abap`) as the exact template.
  Add a step definition entry for each of `TEST_PARLIMIT` and `TEST_PARLIMIT_UNLIM` pointing to `ZCL_FI_STEP_TEST_SUCCESS` (the correct step class — confirmed from source; `ZCL_FI_STEP_TEST_OK` does not exist).

  **7b — `build_rows()` filter dispatch** (around line 250): Add after the `DUPLICATE_CHECK` filter block:
  ```abap
  IF mv_filter_test_id IS INITIAL OR mv_filter_test_id = 'PARALLEL_LIMIT'.
    check_parallel_limit( ).
  ENDIF.
  ```

  **7c — New private method `check_parallel_limit()`**: Declare in private section and implement as a new method, modelled on `check_duplicate()`. The method body:
  - Adds a capability row: `capabilityid = 'PARALLEL_LIMIT'`, description = `'Max parallel instances enforced per process type'`.
  - **Sub-scenario 1** ("Limit enforced"):
    1. Call `create_process( 'TEST_PARLIMIT' ... )` twice → get `lv_inst_id_1`, `lv_inst_id_2`.
    2. `UPDATE zfi_proc_inst SET status = 'RUNNING' WHERE instance_id = lv_inst_id_1`.
    3. `UPDATE zfi_proc_inst SET status = 'RUNNING' WHERE instance_id = lv_inst_id_2`.
    4. Create third instance `lv_inst_id_3`.
    5. Try `execute_process( lv_inst_id_3 )` inside `TRY … CATCH zcx_fi_process_error INTO lx_error`.
     6. Assert `lx_error->if_t100_message~t100key = zcx_fi_process_error=>parallel_limit_exceeded`. Add result row: PASS if caught, FAIL if not.
  - **Sub-scenario 2** ("Unlimited when 0"):
    1. `TEST_PARLIMIT_UNLIM` type with `max_parallel_insts = 0` is already defined in `ensure_test_type_defs()` (see 7a above).
    2. Create instance → call `execute_process()`.
    3. Assert no `parallel_limit_exceeded` is raised (execution may fail for other reasons — only assert absence of this specific exception). Add result row accordingly.
  - Notes: Mirror the row-appending and cleanup patterns from `check_duplicate()` exactly. The CLEANUP block must call `cleanup_test_instances()` — the shared helper that deletes from `zfi_proc_step` and `zfi_proc_inst` for all `TEST_%` rows. Do NOT delete from `zfi_proc_type`: the type rows for `TEST_PARLIMIT` and `TEST_PARLIMIT_UNLIM` are managed by `ensure_test_type_defs()` via MODIFY (idempotent UPSERT) and are intentionally left in place across runs — this matches the existing pattern in `check_duplicate()` exactly. The cleanup must run even if assertions fail.

### Acceptance Criteria

- [x] **AC 1:** Given a process type with `MAX_PARALLEL_INSTS = 2` and 2 instances currently in status `RUNNING`, when `execute_process()` is called for a third instance of that type, then exception `ZCX_FI_PROCESS_ERROR` with textid `parallel_limit_exceeded` is raised and the third instance is not started.

- [x] **AC 2:** Given a process type with `MAX_PARALLEL_INSTS = 0` (default) and any number of RUNNING instances, when `execute_process()` is called, then the parallel limit check is skipped entirely and execution proceeds normally (no `parallel_limit_exceeded` exception).

- [x] **AC 3:** Given an existing process type with no `MAX_PARALLEL_INSTS` field value set (field is newly added with default 0), when `execute_process()` is called, then behavior is identical to AC 2 — no regression for existing process types.

- [x] **AC 4:** Given a process type with `MAX_PARALLEL_INSTS = 2` and only 1 instance currently `RUNNING`, when `execute_process()` is called for a second instance, then execution is NOT blocked (count < limit).

- [x] **AC 5:** Given a process type with `ALLOW_DUPLICATE = ' '` and `MAX_PARALLEL_INSTS = 1`, when two instances with identical parameters are created and the first is RUNNING, when `execute_process()` is called on the second instance, then `duplicate_instance` is raised by `create_process()` — not `parallel_limit_exceeded` — because the duplicate check fires first and prevents the second instance from being created. This verifies the two checks are independent and ordered correctly.

- [x] **AC 6:** Given the health check is executed (`ZCL_FIPROC_HEALTH_CHK_QUERY`), when the `PARALLEL_LIMIT` test scenario runs, then both sub-scenarios (limit enforced + unlimited when 0) pass and are reported as PASS in the health check output.

- [x] **AC 7:** Given the DDIC transport is imported, when `ZFI_PROC_TYPE` is viewed in SE16, then the `MAX_PARALLEL_INSTS` column is present and defaults to `0` for all existing rows.

## Additional Context

### Dependencies

- No blocking story dependencies.
- DDIC activation order: domain → dtel → table field (Tasks 1–3 must be activated in sequence).
- Message class (Task 4) and exception class (Task 5) can be done in parallel with DDIC work.
- Task 6 (manager check) depends on Task 3 (field must exist in DB) and Task 5 (exception constant must exist).
- Task 7 (health check) depends on Task 3 (field on type table) and Task 6 (the method under test).

### Testing Strategy

**Automated (via Health Check framework):**
- `ZCL_FIPROC_HEALTH_CHK_QUERY` test `PARALLEL_LIMIT` covers the main enforcement and unlimited-bypass scenarios.
- Run via transaction or calling `ZCL_FIPROC_HEALTH_CHK_QUERY=>get_instance( )->execute( filter_test_id = 'PARALLEL_LIMIT' )`.

**Manual verification steps:**
1. Activate all DDIC objects. Confirm `ZFI_PROC_TYPE` has `MAX_PARALLEL_INSTS` column in SE11.
2. In SM30/SE16, set `MAX_PARALLEL_INSTS = 1` on an existing process type.
3. Start one instance → confirm it runs.
4. While first is RUNNING, attempt to start a second → confirm `parallel_limit_exceeded` message appears.
5. Set `MAX_PARALLEL_INSTS = 0` on same type → confirm second instance starts without blocking.

**Regression:**
- Existing process types with `MAX_PARALLEL_INSTS = 0` (default) must continue to behave exactly as before.
- Run the full health check suite to confirm no regressions: `ZCL_FIPROC_HEALTH_CHK_QUERY=>get_instance( )->execute( )`.

### Notes

**High-risk items:**
- **`sotr_conc` GUID for `parallel_limit_exceeded`**: In abapGit-based workflows, the GUID in the XML is persisted as-is — the system does NOT auto-generate or replace it on activation. You must supply a real UUID that does not collide with any existing OTR entry. Generate one via `CL_SYSTEM_UUID=>CREATE_UUID_C32_STATIC( )` in a scratch report before editing the XML. Using a random non-UUID string will cause activation failure or a silent OTR duplicate.
- **DB conversion after Task 3**: After activating the table in SE11, run SE14 ("Activate and Adjust Database") to populate the new `MAX_PARALLEL_INSTS` column with `0` for all existing rows. Skipping this will leave NULLs in existing rows despite the NOT NULL declaration in DDIC.
- **`SELECT COUNT(*)` comparison**: `SELECT COUNT(*)` returns `INT8`; `lv_max_parallel_insts` is `INT4`. Direct comparison (`IF lv_running_count >= lv_max_parallel_insts`) is valid in ABAP 7.58 and consistent with the codebase style — no `CONV` needed.

**Known limitations:**
- Soft check only (no DB lock). Two processes started within the same millisecond could both pass the count check before either is set to RUNNING. This is consistent with `ALLOW_DUPLICATE` behavior and accepted by the team.

**Future considerations (out of scope):**
- A `QUEUED` status concept could allow instances to wait rather than fail when the limit is hit.
- A UI indicator on the process monitor showing current parallel count vs. limit.
