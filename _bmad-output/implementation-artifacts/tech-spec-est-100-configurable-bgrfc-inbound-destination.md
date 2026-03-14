---
title: 'Configurable bgRFC Inbound Destination per Process Type (EST-100)'
slug: 'est-100-configurable-bgrfc-inbound-destination'
created: '2026-03-13'
status: 'ready-for-dev'
stepsCompleted: [1, 2, 3, 4]
tech_stack: ['ABAP 7.58', 'SAP DDIC', 'ZFI_PROCESS framework', 'bgRFC', 'cl_bgrfc_destination_inbound']
files_to_modify:
  - 'src/zfi_proc_type.tabl.xml'
  - 'src/zcl_fi_process_instance.clas.abap'
  - 'src/zcl_fiproc_health_chk_query.clas.abap'
  - 'src/zcx_fi_process_error.clas.abap'
  - 'src/zfi_process.msag.xml'
code_patterns:
  - 'DDIC field: BGRFC_DEST_NAME_INBOUND NOTNULL, same pattern as BAL_OBJECT in EST-101'
  - 'Private attribute mv_bgrfc_dest_name TYPE bgrfc_dest_name_inbound in ZCL_FI_PROCESS_INSTANCE'
  - 'load_bal_config() extended to also SELECT bgrfc_dest_name_inbound'
  - 'get_bgrfc_destination() simplified to direct read of mv_bgrfc_dest_name — 3-tier heuristic removed'
  - 'Existence check via TRY/CATCH cl_bgrfc_destination_inbound=>create() inside initialize_instance() before save_instance()'
  - 'ensure_test_type_defs() sets BGRFC_DEST_NAME_INBOUND = ZFI_PROCESS on ZFI_PROC_TYPE rows for all queued test types'
  - 'substep_data_structure = ZFI_PROCESS removed from ZFI_PROC_DEF rows for all queued test types'
test_patterns:
  - 'Health check test types recreated with BGRFC_DEST_NAME_INBOUND = ZFI_PROCESS default'
  - 'Manual integration test: create process with invalid destination → ZCX_FI_PROCESS_ERROR raised'
  - 'Manual integration test: ZFI_ALLOC process type with correct destination runs end-to-end'
---

# Tech-Spec: Configurable bgRFC Inbound Destination per Process Type (EST-100)

**Created:** 2026-03-13

## Overview

### Problem Statement

The bgRFC inbound destination used for queued substep dispatch is hardcoded as `'ZFI_PROCESS'` via a 3-tier heuristic in `ZCL_FI_PROCESS_INSTANCE=>get_bgrfc_destination()`:
1. Misuses `ZFI_PROC_DEF-SUBSTEP_DATA_STRUCTURE` field to carry a destination name
2. Parses `DESCRIPTION` text for `[DEST:...]` patterns
3. Falls back to the literal constant `'ZFI_PROCESS'`

This prevents per-process-type bgRFC resource isolation (separate inbound destinations = separate work queues, separate server groups, independent load management). All process types share the same destination, blocking independent scaling.

There is also no validation that the destination exists before a process instance starts — an invalid destination only fails at bgRFC dispatch time, deep inside execution, with a cryptic runtime error.

### Solution

Add a `BGRFC_DEST_NAME_INBOUND` field (NOTNULL, data element `BGRFC_DEST_NAME_INBOUND`) directly to `ZFI_PROC_TYPE`. Load it at instance initialisation time. Replace the 3-tier heuristic with a direct read of this field. Validate the destination exists in `create_process()` before execution begins — same pattern as EST-101's BAL object validation.

### Scope

**In Scope:**
- New NOTNULL field `BGRFC_DEST_NAME_INBOUND` on `ZFI_PROC_TYPE` (no wrapper data element needed — use SAP's existing data element directly)
- Private attribute `mv_bgrfc_dest_name` on `ZCL_FI_PROCESS_INSTANCE`
- Extend `load_bal_config()` to also read `bgrfc_dest_name_inbound` (or rename method to `load_process_type_config()`)
- Simplify `get_bgrfc_destination()` to direct read of `mv_bgrfc_dest_name` — remove all 3 heuristic tiers entirely
- Destination existence check **inside `initialize_instance()`** in `ZCL_FI_PROCESS_INSTANCE` (after `load_bal_config()`, before `save_instance()`) — via TRY/CATCH on `cl_bgrfc_destination_inbound=>create()`
- Update `ensure_test_type_defs()` in `ZCL_FIPROC_HEALTH_CHK_QUERY` to set `BGRFC_DEST_NAME_INBOUND = 'ZFI_PROCESS'` on all queued test process types
- Remove `substep_data_structure = 'ZFI_PROCESS'` workaround from all test type definitions
- Data migration note for existing production process types (same format as `est-101-data-migration-note.md`)
- New T100 message for invalid destination error

**Out of Scope:**
- Multiple destinations per process type
- Destination creation or management UI
- Non-queued (SERIAL) steps — destination is irrelevant for them
- Additional bgRFC destination types (outbound, transactional)
- Public getter `get_bgrfc_dest_name()` — not needed (check is internal to `ZCL_FI_PROCESS_INSTANCE`)

---

## Context for Development

### Codebase Patterns

**EST-101 is the direct precedent** — follow exactly the same pattern:

| EST-101 (BAL config) | EST-100 (bgRFC dest config) |
|---|---|
| Added `BAL_OBJECT` + `BAL_SUBOBJECT` to `ZFI_PROC_TYPE` | Add `BGRFC_DEST_NAME_INBOUND` to `ZFI_PROC_TYPE` |
| Created wrapper data elements `ZFI_PROCESS_BAL_OBJECT` / `ZFI_PROCESS_BAL_SUBOBJ` | No wrapper needed — use SAP data element `BGRFC_DEST_NAME_INBOUND` directly |
| Private attributes `mv_bal_object`, `mv_bal_subobject` on instance | Private attribute `mv_bgrfc_dest_name TYPE bgrfc_dest_name_inbound` |
| `load_bal_config()` SELECT from `ZFI_PROC_TYPE` | Extend same method to also read `bgrfc_dest_name_inbound` |
| BAL validation in `create_process()`: SELECT from `balobjct` | Destination validation: SELECT from `bgrfc_inbdest` |
| `ensure_test_type_defs()` sets `BAL_OBJECT = 'ZFI_PROCESS'`, `BAL_SUBOBJECT = 'TESTS'` | Also set `BGRFC_DEST_NAME_INBOUND = 'ZFI_PROCESS'` in same MODIFY statements |

**`load_bal_config()` current implementation** (`zcl_fi_process_instance.clas.abap` lines 2326–2337):
```abap
METHOD load_bal_config.
  SELECT SINGLE bal_object, bal_subobject
    FROM zfi_proc_type
    INTO ( @mv_bal_object, @mv_bal_subobject )
    WHERE process_type = @ms_instance-process_type.
  IF sy-subrc <> 0.
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING
        textid = zcx_fi_process_error=>process_type_not_found
        value  = |Process type '{ ms_instance-process_type }' not found in ZFI_PROC_TYPE|.
  ENDIF.
ENDMETHOD.
```
→ Extend the SELECT to also read `bgrfc_dest_name_inbound INTO @mv_bgrfc_dest_name`.

**BAL existence check pattern** in `zcl_fi_process_manager.clas.abap` (EST-101) — **for reference only, not replicated here**:
```abap
SELECT SINGLE objekt FROM balobjct
  INTO @DATA(lv_bal_obj)
  WHERE spras = @sy-langu
    AND balobj = @lv_bal_object.
IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid = zcx_fi_process_error=>...
      value  = lv_bal_object.
ENDIF.
```
EST-100 does **not** replicate this SELECT pattern. Instead, the destination existence check uses `cl_bgrfc_destination_inbound=>create()` inside `initialize_instance()` — see Task 5.

**Current `get_bgrfc_destination()` — lines 1679–1716** (to be replaced entirely):
```abap
" Tier 1 — misuse of substep_data_structure
SELECT SINGLE substep_data_structure FROM zfi_proc_def ...
IF lv_dest IS NOT INITIAL. rv_dest = lv_dest. RETURN. ENDIF.

" Tier 2 — parse DESCRIPTION text
...parse [DEST:xxx] pattern...

" Tier 3 — hardcoded fallback
IF lv_dest IS INITIAL.
  lv_dest = 'ZFI_PROCESS'.
ENDIF.
```
→ Replace entirely with:
```abap
METHOD get_bgrfc_destination.
  rv_dest = mv_bgrfc_dest_name.
ENDMETHOD.
```

**`process_substeps_queued()` — how destination is consumed** (lines 1469–1514):
```abap
DATA(lv_dest) = get_bgrfc_destination( iv_step_number ).
dest_name = CONV bgrfc_dest_name_inbound( lv_dest ).
...
my_destination = cl_bgrfc_destination_inbound=>create( dest_name ).
my_unit = my_destination->create_qrfc_unit( ).
```
→ No changes needed here — `get_bgrfc_destination()` will return the correct value directly.

**Test type `substep_data_structure` workaround** (`zcl_fiproc_health_chk_query.clas.abap`):
- Line 678: `substep_data_structure = 'ZFI_PROCESS'` → remove (or keep if it has a real structural meaning — verify)
- Lines 705, 732, 759, 939: same → remove
- Add `bgrfc_dest_name_inbound = 'ZFI_PROCESS'` to the same MODIFY statements

### Files to Reference

| File | Purpose |
|------|---------|
| `src/zfi_proc_type.tabl.xml` | Add `BGRFC_DEST_NAME_INBOUND` field after `BAL_SUBOBJECT` (line ~82) |
| `src/zcl_fi_process_instance.clas.abap` | Extend `load_bal_config()`, add existence check in `initialize_instance()`, replace `get_bgrfc_destination()`, add private attribute |
| `src/zcl_fiproc_health_chk_query.clas.abap` | Update `ensure_test_type_defs()` for all queued test types |
| `src/zcx_fi_process_error.clas.abap` | Add `bgrfc_destination_not_found` constant (msg 084) |
| `src/zfi_process.msag.xml` | Add T100 message 084 |
| `_bmad-output/implementation-artifacts/est-101-data-migration-note.md` | Reference pattern for data migration note |

### Technical Decisions

1. **No wrapper data element** — use SAP's `BGRFC_DEST_NAME_INBOUND` data element directly as the ROLLNAME in DDIC. EST-101 needed wrappers because SAP's `BALOBJ_D` domain lacked the ZFI_PROCESS namespace labeling; `BGRFC_DEST_NAME_INBOUND` already has correct label and search help.

2. **NOTNULL field** — consistent with BAL_OBJECT/BAL_SUBOBJECT. Requires data migration for existing rows. `ensure_test_type_defs()` handles test types automatically.

3. **Existence check inside `initialize_instance()`** — the check is placed after `load_bal_config()` (line ~465) and before `save_instance()` (line ~491) in `ZCL_FI_PROCESS_INSTANCE`. This means: (a) `mv_bgrfc_dest_name` is already populated when the check runs; (b) if the destination is invalid, the exception is raised before the instance is persisted — no orphaned instances (resolves F17). No public getter and no manager-side check are needed.

4. **Existence check via `cl_bgrfc_destination_inbound=>create()` TRY/CATCH** — avoids querying an internal SAP table (`bgrfc_inbdest`) whose key field name is not documented in abapgit-accessible sources. If `create()` succeeds the destination exists; catch `cx_bgrfc_*` and raise `ZCX_FI_PROCESS_ERROR=>bgrfc_destination_not_found`. Verify exact exception class name in SE24 before coding (`cx_bgrfc_destination_not_found` or similar).

5. **No `IS NOT INITIAL` guard** — the existence check runs unconditionally. A blank destination (un-migrated row after transport) will immediately raise `bgrfc_destination_not_found` during `initialize_instance()`. This is intentional: fail loudly and clearly rather than silently bypass the check. The existing runtime guard at line 1484 (`IF lv_dest IS INITIAL → step_execution_failed`) is a secondary safety net only.

6. **Remove 3-tier heuristic entirely** — the `substep_data_structure` field has a real purpose (DDIC structure name for substep parameters) that was being abused. Cleaning this up removes a confusing side-effect from the codebase.

7. **Extend `load_bal_config()`** rather than creating a new method — avoids a second SELECT on `ZFI_PROC_TYPE` and keeps the config-loading logic in one place. Technical debt note: the method name no longer reflects its full scope. Do NOT rename — two callers exist (`initialize_instance` line ~465, `load_instance` line ~541); renaming provides no functional benefit.

---

## Implementation Plan

### Tasks

#### Task 1 — Extend `ZFI_PROC_TYPE` DDIC table

**File:** `src/zfi_proc_type.tabl.xml`

Add new field entry after `BAL_SUBOBJECT` (after line ~82) and before `CREATED_BY`:

```xml
<DD03P>
  <FIELDNAME>BGRFC_DEST_NAME_INBOUND</FIELDNAME>
  <ROLLNAME>BGRFC_DEST_NAME_INBOUND</ROLLNAME>
  <ADMINFIELD>0</ADMINFIELD>
  <NOTNULL>X</NOTNULL>
  <COMPTYPE>E</COMPTYPE>
</DD03P>
```

**Activation order:** Activate table in SE11 before activating dependent ABAP classes.

---

#### Task 2 — Add private attribute to `ZCL_FI_PROCESS_INSTANCE`

**File:** `src/zcl_fi_process_instance.clas.abap`

Add private attribute after `mv_bal_subobject` (near line ~235):

```abap
DATA mv_bgrfc_dest_name TYPE bgrfc_dest_name_inbound.
```

---

#### Task 3 — Extend `load_bal_config()` to also read `bgrfc_dest_name_inbound`

**File:** `src/zcl_fi_process_instance.clas.abap` — method `load_bal_config` (lines 2326–2337)

Replace the SELECT with:

```abap
METHOD load_bal_config.
  SELECT SINGLE bal_object, bal_subobject, bgrfc_dest_name_inbound
    FROM zfi_proc_type
    INTO ( @mv_bal_object, @mv_bal_subobject, @mv_bgrfc_dest_name )
    WHERE process_type = @ms_instance-process_type.
  IF sy-subrc <> 0.
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING
        textid = zcx_fi_process_error=>process_type_not_found
        value  = ms_instance-process_type.
  ENDIF.
ENDMETHOD.
```

> Optional: rename `load_bal_config` → `load_process_type_config` if it doesn't break any existing callers (check all call sites first).

---

#### Task 4 — Simplify `get_bgrfc_destination()` — remove 3-tier heuristic

**File:** `src/zcl_fi_process_instance.clas.abap` — method `get_bgrfc_destination` (lines 1679–1716)

Replace entire method body with:

```abap
METHOD get_bgrfc_destination.
  "#EC UNUSED_PARAM
  rv_dest = mv_bgrfc_dest_name.
ENDMETHOD.
```

The method signature is unchanged (still takes `iv_step_number`, still returns `rv_dest`) — the sole caller in `process_substeps_queued()` (line ~1476) requires no changes. The parameter is now unused; suppress the warning with `"#EC UNUSED_PARAM`.

> Technical note: removing `iv_step_number` entirely is possible (no other callers found — confirmed via codebase search). However, keeping it avoids touching the caller and eliminates risk. Add a comment if desired: `" iv_step_number retained for signature compatibility — destination is now process-type-level`.

---

#### Task 5 — Add destination existence check in `initialize_instance()`

**File:** `src/zcl_fi_process_instance.clas.abap` — method `initialize_instance` (around line ~465, after `load_bal_config()` call, before `save_instance()` call at line ~491)

```abap
" Validate bgRFC inbound destination exists (fail before persisting instance)
TRY.
    cl_bgrfc_destination_inbound=>create( mv_bgrfc_dest_name ).
  CATCH cx_bgrfc_destination_not_found cx_bgrfc_destination_invalid.  "verify class names in SE24
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING
        textid = zcx_fi_process_error=>bgrfc_destination_not_found
        value  = mv_bgrfc_dest_name.
ENDTRY.
```

**Why here, not in `create_process()`:**
- `initialize_instance()` is called at line ~400 inside `ZCL_FI_PROCESS_INSTANCE=>create()`, which is called from `create_process()` in the manager
- `load_bal_config()` (line ~465) populates `mv_bgrfc_dest_name` before this check runs
- `save_instance()` (line ~491) persists the instance — placing the check between these two calls guarantees no orphaned instances on invalid destination
- The check runs unconditionally — a blank destination raises `bgrfc_destination_not_found` immediately (no `IS NOT INITIAL` bypass)

> **Verify in SE24**: Confirm the exact `cx_bgrfc_*` exception class(es) raised by `cl_bgrfc_destination_inbound=>create()` when the destination does not exist. Likely `cx_bgrfc_destination_not_found` or `cx_bgrfc_*`. Catch the correct class(es).

---

#### Task 6 — Add exception text to `ZCX_FI_PROCESS_ERROR` and T100 message

**File:** `src/zcx_fi_process_error.clas.abap`

Add new constant in the `CONSTANTS BEGIN OF` block using the exact project pattern (confirmed from `zcx_fi_process_error.clas.abap` lines 1–80):

```abap
BEGIN OF bgrfc_destination_not_found,
  msgid TYPE symsgid VALUE 'ZFI_PROCESS',
  msgno TYPE symsgno VALUE '084',
  attr1 TYPE scx_attrname VALUE 'VALUE',
  attr2 TYPE scx_attrname VALUE '',
  attr3 TYPE scx_attrname VALUE '',
  attr4 TYPE scx_attrname VALUE '',
END OF bgrfc_destination_not_found,
```

> Note: Do NOT use `TYPE sotr_cid VALUE '...'` — that is not the pattern used in this project. Every constant is a `BEGIN OF ... END OF` block with `msgid`, `msgno`, and `attr1`–`attr4` fields.

**File:** `src/zfi_process.msag.xml`

Add new message **084** (confirmed next available — messages 060–083 checked, 084 is free). Master language is `E` (English):

| Msg# | Text (EN) |
|------|-----------|
| 084 | bgRFC inbound destination '&1' not found — configure in ZFI_PROC_TYPE |

---

#### Task 8a — Update `ZFI_PROC_TYPE` entries in `ensure_test_type_defs()`

**File:** `src/zcl_fiproc_health_chk_query.clas.abap` — `ZFI_PROC_TYPE` MODIFY block (lines ~390–567)

For each of the 5 queued test process types, add `bgrfc_dest_name_inbound = 'ZFI_PROCESS'` to the VALUE constructor inside the `MODIFY zfi_proc_type FROM TABLE lt_types` block:

- `TEST_QUEUED_BASIC`
- `TEST_QUEUED_FAIL`
- `TEST_QUEUED_EXCEPTION`
- `TEST_PIPELINE`
- `TEST_PARAM_QUEUED`

These entries are in the `ZFI_PROC_TYPE` MODIFY block (ends at line ~567). Do not confuse with the `ZFI_PROC_DEF` block below.

---

#### Task 8b — Remove `substep_data_structure` workaround from `ZFI_PROC_DEF` entries

**File:** `src/zcl_fiproc_health_chk_query.clas.abap` — `ZFI_PROC_DEF` MODIFY block (lines ~576–942)

Remove `substep_data_structure = 'ZFI_PROCESS'` from the step definition rows for the 5 queued test types (approx. lines 678, 705, 732, 759, 939).

**This is safe** — confirmed by code review:
- `ZCL_FI_STEP_PIPELINE_CONSUMER` does NOT use `substep_data_structure` for DDIC deserialization
- `substep_data_structure = 'ZFI_PROCESS'` was set purely as a destination workaround (Tier 1 of the old heuristic in `get_bgrfc_destination()`)
- After Task 4, `get_bgrfc_destination()` no longer reads `substep_data_structure` at all
- Leave `substep_data_structure` blank or set to actual DDIC structure name if these steps have real parameter structures

---

#### Task 9 — Write data migration note

Create file: `_bmad-output/implementation-artifacts/est-100-data-migration-note.md`

Document required SE16/SM30 action for existing production `ZFI_PROC_TYPE` rows (same format as `est-101-data-migration-note.md`):

| PROCESS_TYPE | BGRFC_DEST_NAME_INBOUND |
|---|---|
| `ZFI_ALLOC` | `ZFI_PROCESS` |
| _(any other production type)_ | `ZFI_PROCESS` _(or custom destination)_ |

Include SQL helper and validation steps.

---

### Acceptance Criteria

**AC1 — DDIC field exists:**
Given `ZFI_PROC_TYPE` in SE11,
When inspecting the table structure,
Then field `BGRFC_DEST_NAME_INBOUND` with data element `BGRFC_DEST_NAME_INBOUND` and NOTNULL = X is present after `BAL_SUBOBJECT`.

**AC2 — Process type config loaded at instance init:**
Given `ZFI_ALLOC` has `BGRFC_DEST_NAME_INBOUND = 'ZFI_PROCESS'` in `ZFI_PROC_TYPE`,
When a process instance is created for type `ZFI_ALLOC`,
Then `get_bgrfc_dest_name()` (or internal attribute `mv_bgrfc_dest_name`) equals `'ZFI_PROCESS'`.

**AC3 — `get_bgrfc_destination()` returns configured value:**
Given `ZFI_ALLOC` has `BGRFC_DEST_NAME_INBOUND = 'ZFI_PROCESS'` in `ZFI_PROC_TYPE`,
When `get_bgrfc_destination()` is called,
Then it returns `'ZFI_PROCESS'` — no heuristic parsing, no hardcoded fallback.

**AC4 — Invalid destination rejected before instance is saved:**
Given a process type with `BGRFC_DEST_NAME_INBOUND = 'INVALID_DEST'` in `ZFI_PROC_TYPE`,
When `ZCL_FI_PROCESS_INSTANCE=>create()` is called (via `create_process()`),
Then `ZCX_FI_PROCESS_ERROR=>bgrfc_destination_not_found` is raised with `VALUE = 'INVALID_DEST'` **before** `save_instance()` is called — no orphaned instance row in the database.

**AC5 — Valid destination passes validation:**
Given `BGRFC_DEST_NAME_INBOUND = 'ZFI_PROCESS'` and that destination exists in the SAP system,
When `create_process()` is called,
Then no exception is raised and execution proceeds normally.

**AC6 — Health check test types have correct destination:**
Given `ensure_test_type_defs()` is executed,
When inspecting `ZFI_PROC_TYPE` for `TEST_QUEUED_BASIC`, `TEST_QUEUED_FAIL`, `TEST_QUEUED_EXCEPTION`, `TEST_PIPELINE`, `TEST_PARAM_QUEUED`,
Then each has `BGRFC_DEST_NAME_INBOUND = 'ZFI_PROCESS'`.

**AC7 — `substep_data_structure` workaround removed:**
Given the updated `ensure_test_type_defs()`,
When inspecting `ZFI_PROC_DEF` rows for queued test types,
Then `substep_data_structure` is no longer set to `'ZFI_PROCESS'` (unless it has a legitimate structural purpose).

**AC8 — End-to-end queued execution works:**
Given `ZFI_ALLOC` process type with `BGRFC_DEST_NAME_INBOUND = 'ZFI_PROCESS'`,
When a full allocation run is executed,
Then PHASE2 substeps are dispatched to the `ZFI_PROCESS` bgRFC inbound destination and complete successfully.

---

## Additional Context

### Dependencies

- **EST-101** must be deployed first (it extended `ZFI_PROC_TYPE` with `BAL_OBJECT`/`BAL_SUBOBJECT` and established the config-loading pattern) — EST-101 code is already merged.
- **Transport order**: Activate `ZFI_PROC_TYPE` table in SE11 before activating `ZCL_FI_PROCESS_INSTANCE`.
- **Data migration — critical sequence**:
  1. **Before transport**: Set `ACTIVE = ''` on all production process types in `ZFI_PROC_TYPE` to prevent new process creation during migration window
  2. **Import transport**: DDIC table activated with blank `BGRFC_DEST_NAME_INBOUND` on all existing rows (NOTNULL = blank string, not NULL)
  3. **Migrate immediately**: Run SE16/SM30 to set `BGRFC_DEST_NAME_INBOUND` on all rows — see `est-100-data-migration-note.md`
  4. **Re-activate**: Set `ACTIVE = 'X'` again on production process types
  - Known production process types: `ZFI_ALLOC` (confirmed in `cz.imcg.fast.allocations`). Check `cz.imcg.fast.ovysledovka` for any additional types registered in that package.
  - **In-flight instances**: Existing instances already in progress are safe to load and query. They will only fail when attempting to dispatch queued substeps if `mv_bgrfc_dest_name` is blank (secondary guard at line 1484). Recommendation: **complete all in-flight queued processes before importing the transport**.

### Testing Strategy

- **No ABAP Unit tests** — project uses manual test programs and health check (see project-context.md)
- **Health check** (`ZCL_FIPROC_HEALTH_CHK_QUERY`) acts as the automated regression suite — updating `ensure_test_type_defs()` is the primary test coverage
- **Manual validation path**:
  1. Deploy transport
  2. Run SE16 migration for `ZFI_PROC_TYPE`
  3. Run health check — all `TEST_QUEUED_*` types must pass
  4. Run `ZFI_ALLOC` process end-to-end, verify PHASE2 substeps complete
  5. Test negative case: temporarily set `BGRFC_DEST_NAME_INBOUND = 'BAD_DEST'` on a test type, call `create_process()`, confirm clear error message

### Notes

- **`cl_bgrfc_destination_inbound=>create()` exception class**: Verify exact `cx_bgrfc_*` exception class name in SE24 before coding Task 5. Expected: `cx_bgrfc_destination_not_found` — but confirm. Catch the most specific class available.
- **`load_bal_config()` method name**: Intentionally not renamed to `load_process_type_config()`. Two callers exist (`initialize_instance` ~line 465, `load_instance` ~line 541). Renaming provides no functional benefit. Logged as technical debt.
- **`ZFI_PROC_TYPE` BUFALLOW = N**: No table buffering. Every `initialize_instance()` call hits the database for `load_bal_config()`. This is the existing behavior (same as EST-101) and is acceptable given process creation frequency.
- **DDIC field LENG/OUTPUTLEN**: Inherited from the `BGRFC_DEST_NAME_INBOUND` data element domain. No override needed — intentional.
- **Constitution compliance**:
  - Principle I (DDIC-First): Field uses standard SAP data element directly ✅
  - Principle II (SAP Standards): Field name matches data element name, line lengths ≤120 chars ✅
  - Principle III (Consult SAP Docs): Verify `cx_bgrfc_*` exception class in SE24 before coding ⚠️
  - Principle IV (Factory Pattern): No new instantiation patterns ✅
  - Principle V (Error Handling): `ZCX_FI_PROCESS_ERROR=>bgrfc_destination_not_found` with msg 084, `BEGIN OF/END OF` pattern ✅
- **Target repository**: `cz.imcg.fast.planner` (all changes are in ZFI_PROCESS package)
