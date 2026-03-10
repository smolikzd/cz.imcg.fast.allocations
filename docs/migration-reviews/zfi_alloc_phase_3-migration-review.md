# Migration Review: ZFI_ALLOC_PHASE_3 → ZCL_FI_ALLOC_STEP_PHASE3

**Date:** 2026-03-05
**Source program:** `ZFI_ALLOC_PHASE_3`
**Target step class:** `ZCL_FI_ALLOC_STEP_PHASE3`
**Framework:** ZFI_PROCESS (ABAP 7.58 / S/4HANA)
**Reviewer:** OpenCode AI Agent

---

## Overview

This document reviews the migration of the standalone report `ZFI_ALLOC_PHASE_3` into the
ZFI_PROCESS framework step class `ZCL_FI_ALLOC_STEP_PHASE3`.

The original program performs allocation item generation for "unassigned" FI line items —
items that could not be matched in Phase 1/2. For each unassigned item it:

1. Looks up the GL account from `I_GLAccountLineItem`.
2. Finds the matching allocation base header (`ZFI_ALLOC_BCHE`) by a full combination of
   business key fields.
3. For each item on that header (`ZFI_ALLOC_BCITM`), calculates a proportional allocated
   amount and inserts it into `ZFI_ALLOC_ITEMS` (line type 'A').
4. Inserts an offset item (line type 'O') for the full original amount.
5. Updates `ZFI_ALLOC_BCHE.allocated` flag.

The migration is a **near-verbatim copy** of the original program's processing logic pasted
into the `execute` method. There is no substep architecture, no `execute_substep`, no
`on_success`, and no `on_error`. The step **cannot be activated** in its current state (see
FINDING-01). It also has a **functional regression** relative to the original program (see
FINDING-05), and carries all of the structural LUW violations seen in previous phases.

---

## Findings

### FINDING-01 — `execute_substep`, `on_success`, `on_error` missing — class cannot be activated (Critical)

**Severity:** Critical
**Constitution principle:** II — SAP Standards Compliance

**Description:**
The class declares `INTERFACES zif_fi_process_step` in its PUBLIC SECTION. The interface
`ZIF_FI_PROCESS_STEP` defines nine methods that must all be implemented. The implementation
section contains only: `init`, `execute`, `validate`, `rollback`, `get_description`, and
`plan_substeps`. Three required interface methods are absent:

- `zif_fi_process_step~execute_substep`
- `zif_fi_process_step~on_success`
- `zif_fi_process_step~on_error`

The ABAP compiler enforces complete interface implementation. This class will fail activation
with a syntax error listing each unimplemented method. It cannot be transported or used
in the framework in its current state.

**Mitigation:**
Add stub implementations for all three missing methods. At minimum:

```abap
METHOD zif_fi_process_step~execute_substep.
  " Phase 3 runs serially in execute — no substeps defined
  rs_result-success      = abap_false.
  rs_result-message      = 'execute_substep not supported for Phase 3'.
  rs_result-can_continue = abap_false.
ENDMETHOD.

METHOD zif_fi_process_step~on_success.
  " Called by framework after all substeps complete (none planned here)
ENDMETHOD.

METHOD zif_fi_process_step~on_error.
  " Called by framework after substep failure
ENDMETHOD.
```

---

### FINDING-02 — `MESSAGE TYPE 'E'` used for validation error control flow (Critical)

**Severity:** Critical
**Constitution principle:** V — Error Handling & Observability

**Description:**
The `execute` method uses `MESSAGE e...` for three validation guards — identical to the
same defect found in all three previously reviewed steps:

```abap
" Guard 1 — allocation state not found
IF sy-subrc <> 0.
  MESSAGE e001(zfi_alloc) WITH mv_allocation_id |{ mv_company_code }/...|.
ENDIF.

" Guard 2 — allocation locked
IF ls_state-locked = abap_true.
  MESSAGE e002(zfi_alloc) WITH mv_allocation_id |{ mv_company_code }/...|.
ENDIF.

" Guard 3 — already processed (force not set)
IF ls_state-phase_3_status IS NOT INITIAL AND mv_force_start = abap_false.
  MESSAGE e003(zfi_alloc) WITH mv_allocation_id |...| 'III'.
ENDIF.
```

In a class method, `MESSAGE TYPE 'E'` raises `CX_SY_NO_HANDLER` / `CX_SY_MESSAGE_TYPE_E`,
not a clean business exception. The framework catches this only as `cx_root` fallback,
losing the structured message text in the process monitor.

**Mitigation:**
Replace with `RAISE EXCEPTION TYPE zcx_fi_process_error`:

```abap
IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid = zcx_fi_process_error=>step_execution_failed
      value  = |Allocation state not found: { mv_allocation_id } | &&
               |{ mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period }|.
ENDIF.

IF ls_state-locked = abap_true.
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid = zcx_fi_process_error=>step_execution_failed
      value  = |Allocation { mv_allocation_id } is locked|.
ENDIF.

IF ls_state-phase_3_status IS NOT INITIAL AND mv_force_start = abap_false.
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid = zcx_fi_process_error=>step_execution_failed
      value  = |Phase III already processed for { mv_allocation_id }. Use FORCE_START to override.|.
ENDIF.
```

The same pattern applies to the `MESSAGE e010`, `e011`, `e006`, `e008` calls inside the
processing loop — though those control individual item skipping, not method abort, and
require a different treatment (log + CONTINUE rather than RAISE).

---

### FINDING-03 — `COMMIT WORK` inside `execute` violates LUW contract (Critical)

**Severity:** Critical
**Constitution principle:** V — Error Handling & Observability (LUW ownership)

**Description:**
The `execute` method contains an intermediate `COMMIT WORK` inside the inner
`LOOP AT lt_base_items`:

```abap
lv_commit_counter += 1.
IF lv_commit_counter >= lc_commit.
  COMMIT WORK.
  CLEAR lv_commit_counter.
ENDIF.
```

This is the same LUW contract violation found in all three previously reviewed steps.
The framework owns `COMMIT WORK` — it commits after status updates in `update_step_status`,
`save_instance`, and `save_failure_state`. Steps issuing their own commits:

1. Can commit partial data before the framework has recorded the step's status transition,
   leaving the framework's own state management inconsistent.
2. Prevent the framework from rolling back a partially-executed step on failure.
3. Create orphaned committed rows in `ZFI_ALLOC_ITEMS` if the step fails after a
   mid-loop commit.

Note: the original program's `p_commit` parameter (configurable commit frequency, default
500) has been replaced with the hardcoded constant `lc_commit = 5000`. This is also a
behavioral change (10x increase in batch size) documented under FINDING-12.

**Mitigation:**
Remove the `COMMIT WORK` and the `lv_commit_counter` entirely. For large datasets the
correct solution is to distribute the unassigned items as substeps (one substep per
unassigned item or per batch). See FINDING-04 for the substep architecture recommendation.
The framework will commit after each substep naturally.

If the processing must remain serial (in `execute`), remove the commit and ensure that
`execute` returns cleanly — the framework will commit once after `execute` completes.
Add `rollback` to clean up partial `ZFI_ALLOC_ITEMS` writes on failure:

```abap
METHOD zif_fi_process_step~rollback.
  DELETE FROM zfi_alloc_items
    WHERE company_code  = @mv_company_code
      AND fiscal_year   = @mv_fiscal_year
      AND fiscal_period = @mv_fiscal_period
      AND allocation_id = @mv_allocation_id.
  " Reset phase status
  SELECT SINGLE * FROM zfi_alloc_state INTO @DATA(ls_rb)
    WHERE company_code = @mv_company_code AND fiscal_year = @mv_fiscal_year
      AND fiscal_period = @mv_fiscal_period AND allocation_id = @mv_allocation_id.
  IF sy-subrc = 0 AND ls_rb-phase_3_status = 'R'.
    CLEAR: ls_rb-phase_3_status, ls_rb-phase_3_start_date, ls_rb-phase_3_start_time.
    UPDATE zfi_alloc_state FROM ls_rb.
  ENDIF.
ENDMETHOD.
```

---

### FINDING-04 — No substep architecture — scalability and timeout risk (Critical)

**Severity:** Critical
**Constitution principle:** V — Error Handling & Observability / II — SAP Standards Compliance

**Description:**
Phase 3 iterates over `lt_unassigned` — potentially a very large set of FI line items.
For each unassigned item it performs: one `SELECT SINGLE` on `I_GLAccountLineItem`, one
`SELECT` on `ZFI_ALLOC_BCHE` (multi-condition), and one `SELECT` on `ZFI_ALLOC_BCITM`,
plus multiple DML statements. All of this runs as a single synchronous block inside
`execute`.

For large company codes or fiscal periods, this loop can:
- Exceed the ABAP dialog work process timeout (600 seconds).
- Accumulate a massive single LUW if `COMMIT WORK` is removed (FINDING-03).
- Provide no progress visibility to the framework during execution.

Phase 2 correctly addressed this by planning one substep per base header. Phase 3
has a natural substep boundary at the `LOOP AT lt_unassigned` level — each unassigned
item (or a batch of them) should be a separate substep.

**Mitigation:**
Adopt the same substep architecture as Phase 2:

1. `execute`: validate, set state to 'R', delete existing items, call
   `get_unassigned(...)` to get the unassigned keys, store them in an instance attribute
   (e.g. `mt_unassigned`), return success.
2. `plan_substeps`: loop over `mt_unassigned`, register one substep per item (or per
   batch of N items). Return the substep list.
3. `execute_substep`: receive the unassigned item key via `is_substep_data`, perform the
   GL account lookup, base header match, `ZFI_ALLOC_BCITM` loop, and item insertions.
4. `on_success`: write `phase_3_status = 'F'`, log summary counters.
5. `on_error`: reset `phase_3_status` or mark as errored.

This also removes the need for `COMMIT WORK` inside the loop — the framework commits
after each substep.

---

### FINDING-05 — `BRAND` and `HIER1` filter parameters silently dropped — functional regression (Critical)

**Severity:** Critical
**Constitution principle:** II — SAP Standards Compliance (functional completeness)

**Description:**
The original program accepted two optional filter parameters:

```abap
PARAMETERS: p_brand  TYPE wrf_brand_id,
            p_hier1  TYPE zmatklh1.
```

These were passed to `get_unassigned`:

```abap
zcl_fi_allocations=>get_unassigned(
  EXPORTING iv_cc       = p_cc
            iv_fis_year = p_fis_yr
            iv_fis_per  = p_fis_pr
            iv_brand    = p_brand    " <-- optional filter
            iv_hier1    = p_hier1    " <-- optional filter
  IMPORTING et_result   = DATA(lt_unassigned) ).
```

The migrated step does not read `BRAND` or `HIER1` parameters from `init`, and passes
empty literals instead:

```abap
zcl_fi_allocations=>get_unassigned(
  EXPORTING iv_cc       = mv_company_code
            iv_fis_year = mv_fiscal_year
            iv_fis_per  = mv_fiscal_period
            iv_brand    = ''          " <-- hardcoded empty
            iv_hier1    = ''          " <-- hardcoded empty
  IMPORTING et_result   = DATA(lt_unassigned) ).
```

This is a **silent functional regression**. Any allocation run that previously used the
brand or hierarchy1 filter to process a subset of unassigned items will now process the
full unassigned set. This can produce duplicate allocation items or incorrect results
when Phase 3 is run in partial/filtered mode.

**Mitigation:**
Add `mv_brand` and `mv_hier1` to the private section and read them in `init`:

```abap
PRIVATE SECTION.
  DATA mv_brand  TYPE wrf_brand_id.
  DATA mv_hier1  TYPE zmatklh1.
```

```abap
METHOD zif_fi_process_step~init.
  ...
  mv_brand  = is_context-io_process_instance->get_init_param_value( 'BRAND' ).
  mv_hier1  = is_context-io_process_instance->get_init_param_value( 'HIER1' ).
ENDMETHOD.
```

Then pass them to `get_unassigned`:

```abap
zcl_fi_allocations=>get_unassigned(
  EXPORTING iv_cc       = mv_company_code
            iv_fis_year = mv_fiscal_year
            iv_fis_per  = mv_fiscal_period
            iv_brand    = mv_brand
            iv_hier1    = mv_hier1
  IMPORTING et_result   = DATA(lt_unassigned) ).
```

---

### FINDING-06 — `validate` does not guard parameters before destructive DELETE (Moderate)

**Severity:** Moderate
**Constitution principle:** V — Error Handling & Observability

**Description:**
The `validate` method returns success unconditionally:

```abap
METHOD zif_fi_process_step~validate.
  rs_result-success = abap_true.
  rs_result-message = 'Phase 2 validated'.   " copy-paste error — see also FINDING-13
  rs_result-can_continue = abap_true.
ENDMETHOD.
```

The `execute` method begins with a `DELETE FROM zfi_alloc_items` and
`DELETE FROM zfi_alloc_i_aggr`. Running with an initial (empty) `mv_company_code` or
`mv_allocation_id` would produce a `DELETE` without a meaningful WHERE clause match,
or — depending on DDIC key structure — could affect rows across all allocations.

**Mitigation:**
Add parameter guards in `validate`:

```abap
METHOD zif_fi_process_step~validate.
  IF mv_allocation_id IS INITIAL.
    rs_result-success = abap_false.  rs_result-message = 'ALLOCATION_ID required'.
    rs_result-can_continue = abap_false.  RETURN.
  ENDIF.
  IF mv_company_code IS INITIAL.
    rs_result-success = abap_false.  rs_result-message = 'COMPANY_CODE required'.
    rs_result-can_continue = abap_false.  RETURN.
  ENDIF.
  IF mv_fiscal_year IS INITIAL.
    rs_result-success = abap_false.  rs_result-message = 'FISCAL_YEAR required'.
    rs_result-can_continue = abap_false.  RETURN.
  ENDIF.
  IF mv_fiscal_period IS INITIAL.
    rs_result-success = abap_false.  rs_result-message = 'FISCAL_PERIOD required'.
    rs_result-can_continue = abap_false.  RETURN.
  ENDIF.
  rs_result-success = abap_true.  rs_result-message = 'Phase 3 validated'.
  rs_result-can_continue = abap_true.
ENDMETHOD.
```

---

### FINDING-07 — N+1 SELECT pattern inside the processing loop (Moderate)

**Severity:** Moderate
**Constitution principle:** III — Consult SAP Documentation (performance)

**Description:**
The `LOOP AT lt_unassigned` contains three nested SELECT statements per unassigned item:

```abap
LOOP AT lt_unassigned INTO DATA(ls_unassigned).
  " 1 — GL account lookup
  SELECT SINGLE ChartOfAccounts, GLAccount FROM I_GLAccountLineItem INTO ...
    WHERE ... AND AccountingDocument = @ls_unassigned-accountingdocument
              AND LedgerGLLineItem = @ls_unassigned-ledgergllineitem ...

  " 2 — base header lookup
  SELECT * FROM zfi_alloc_bche INTO TABLE @DATA(li_base_header)
    WHERE ... AND customer = @ls_unassigned-customer AND ...

  " 3 — base items for matched header
  SELECT * FROM zfi_alloc_bcitm INTO CORRESPONDING FIELDS OF TABLE @lt_base_items
    WHERE ... AND key_id = @ls_base_header-key_id.
ENDLOOP.
```

The original program had the same pattern. However the framework step context is the
right place to flag this — queries 1 and 2 fire once per unassigned item. For allocations
with thousands of unassigned items, this creates thousands of round trips to the HANA
database. In particular, query 2 (`SELECT * FROM zfi_alloc_bche`) reloads the full base
header for every unassigned item even though multiple unassigned items may share the same
base header.

**Mitigation:**
- **Short term (low effort):** Cache `ZFI_ALLOC_BCHE` headers in a local hash table keyed
  on the multi-field business key before the loop, then look up in memory instead of
  per-iteration SELECT. Similarly cache `ZFI_ALLOC_BCITM` items per `key_id`.
- **Long term (preferred):** Adopt the substep architecture (FINDING-04) where each
  substep fetches its own data. With bgRFC queued substeps, HANA parallelism distributes
  the DB load.

---

### FINDING-08 — `ZFI_ALLOC_STATE` phase status not integrated with framework lifecycle (Moderate)

**Severity:** Moderate
**Constitution principle:** V — Error Handling & Observability

**Description:**
Identical architectural concern to FINDING-09 in Phase 2 and FINDING-07 in Phase 1.
The step manages `phase_3_status` ('R' → 'F') as a parallel state machine alongside
the framework's own step lifecycle. If `execute` raises an exception or the work process
is cancelled after the `phase_3_status = 'R'` UPDATE but before the `phase_3_status = 'F'`
UPDATE, the allocation state is stuck at 'R'. The framework records its own step failure
but has no mechanism to reset `ZFI_ALLOC_STATE`.

**Mitigation:**
The `rollback` implementation in FINDING-03 covers the 'R' → reset path for failure.
For the 'F' write, move it out of `execute` and into `on_success` (after FINDING-04 is
implemented). For the serial (no-substep) case, once `execute` completes successfully,
the framework will commit, so the 'F' write at the end of `execute` is safe — but only
after FINDING-03 (COMMIT WORK removal) is addressed to keep everything in one LUW.

---

### FINDING-09 — `lv_commit_counter` outer increment off from original program (Moderate)

**Severity:** Moderate
**Constitution principle:** II — SAP Standards Compliance

**Description:**
The original program incremented `lv_commit_counter` once per base item insertion:

```abap
LOOP AT lt_base_items INTO ls_base_item.
  ...
  lv_commit_counter += 1.   " incremented per item
  IF lv_commit_counter >= p_commit.
    COMMIT WORK.
    CLEAR lv_commit_counter.
  ENDIF.
ENDLOOP.
```

The migrated step adds an additional outer increment before the inner loop:

```abap
LOOP AT lt_unassigned INTO DATA(ls_unassigned).
  ...
  lv_commit_counter += 1.   " <-- additional increment not in original (counts outer items)
  ...
  LOOP AT lt_base_items INTO ls_base_item.
    ...
    lv_commit_counter += 1.  " item-level increment
    IF lv_commit_counter >= lc_commit.
      COMMIT WORK.
      CLEAR lv_commit_counter.
    ENDIF.
  ENDLOOP.
```

The outer increment counts unassigned items toward the commit threshold. This changes the
effective commit frequency from "every N base items" to "every N total events (unassigned
items + base items combined)". While this difference is minor in practice (and the commit
should be removed entirely per FINDING-03), it represents unintentional divergence from
the original program's behavior.

**Mitigation:**
Once FINDING-03 is resolved (COMMIT WORK removed), remove `lv_commit_counter` entirely
as it serves no purpose without the COMMIT.

---

### FINDING-10 — Commented-out aggregate items block carried forward without decision (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (dead code)

**Description:**
The large block for saving aggregated allocation items (`ZFI_ALLOC_I_AGGR`) is commented
out in both the original program and the migrated step, identically. The comment
(`*"Uložení agregovaných alokačních položek`) provides no explanation of why it was
disabled or whether it is permanently dropped or deferred.

This is the same issue as Phase 2 FINDING-05.

**Mitigation:**
- **If permanently dropped:** Delete the commented block and add a class-level `" NOTE:`
  comment explaining the decision (e.g. `" NOTE: Aggregate item storage deferred to
  separate step ZCL_FI_ALLOC_STEP_PHASE3_AGG — story #XXX`).
- **If deferred:** Remove from this class, create a separate story.

---

### FINDING-11 — `lv_dummy TYPE c` truncates message text (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance

**Description:**
Throughout `execute`, `lv_dummy TYPE c` (length 1) is used as the `INTO` sink for
`MESSAGE ... INTO lv_dummy`. This truncates the message text to one character. The same
defect appears in all three previously reviewed steps (Phase 1 FINDING-09, Phase 2
FINDING-11).

**Mitigation:**
Change the declaration to `lv_dummy TYPE string`.

---

### FINDING-12 — `p_commit` behavioral change: 500 → 5000 (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance

**Description:**
The original program used `p_commit TYPE int4 DEFAULT 500` (configurable at runtime,
defaulting to 500 items per commit). The migrated step hardcodes `lc_commit = 5000`.
This is a 10x increase in items held in a single LUW before commit, increasing memory
pressure and rollback segment load on HANA for large allocations. The change is not
documented anywhere.

Since the COMMIT should be removed entirely per FINDING-03, this constant will become
irrelevant. However if the commit is kept temporarily as a workaround, the value should
be returned to 500 or exposed as a process parameter.

---

### FINDING-13 — Stale "Phase 2" copy-paste strings in Phase 3 class (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance

**Description:**
Multiple string literals reference "Phase 2" in a class that implements Phase 3:

```abap
rs_result-message = |Phase 2 completed|.      " in execute (should be Phase 3)
rs_result-message = 'Phase 2 validated'.      " in validate
rv_description    = 'Perform Phase 2 of allocations process'.  " in get_description
```

This is evidence of copy-paste from the Phase 2 step without updating the descriptive text.

**Mitigation:**
Replace all three with Phase 3 equivalents:

```abap
rs_result-message = |Phase 3 completed|.
rs_result-message = 'Phase 3 validated'.
rv_description    = 'Perform Phase 3 of allocations process'.
```

---

### FINDING-14 — Unused variable declarations (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (dead code)

**Description:**
Several variables are declared in `execute` but never used or are used incorrectly:

- `lv_total_aiccc TYPE fis_hsl` — declared, never assigned or read.
- `lt_messages TYPE bapirettab` — declared, never used.
- `lv_items_created TYPE int4` — incremented (`lv_items_created += 1`) but never logged
  or returned; the original program also never logged this counter, making it dead code
  in both.
- `ls_base_header-allocated = 'Y'` — assigned to the local variable `ls_base_header`
  after the SELECT, but this assignment has no effect; the database is updated by the
  subsequent `UPDATE zfi_alloc_bche SET allocated = 'Y'` statement. The local assignment
  is redundant.

**Mitigation:**
Remove all unused declarations. Run the ABAP Extended Program Check (SLIN) to get the
complete list.

---

### FINDING-15 — Redundant `ms_context` field (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (clean code)

**Description:**
`ms_context` is stored in `init` (`ms_context = is_context`) but never read after
`mv_*` fields are populated. Identical to the same finding in all three previous steps.

**Mitigation:**
Remove `ms_context` from the private section.

---

## Behavioral Changes vs. Original Program

| Aspect | Original program | Migrated step | Risk |
|--------|-----------------|---------------|------|
| Brand filter | Passed via `p_brand` parameter | Hardcoded to `''` | **Critical** — silent regression |
| Hierarchy1 filter | Passed via `p_hier1` parameter | Hardcoded to `''` | **Critical** — silent regression |
| Commit frequency | Configurable `p_commit`, default 500 | Hardcoded `lc_commit = 5000` | Moderate — memory/LUW impact |
| Commit placement | After every N base items in inner loop | After N total events (outer + inner) | Minor — effective frequency differs |
| Processing architecture | Single sequential loop | Same (no substeps) | Negative — scalability lost, timeout risk |
| Activation | N/A (standalone report) | **Cannot activate** (3 interface methods missing) | Critical |
| Error handling | `MESSAGE TYPE 'E'` aborts report | Same (incorrect in class context) | Critical |

---

## Summary

| ID | Description | Severity | Status |
|----|-------------|----------|--------|
| FINDING-01 | Missing `execute_substep`, `on_success`, `on_error` — class cannot be activated | Critical | Open |
| FINDING-02 | `MESSAGE TYPE 'E'` used for validation error control flow | Critical | Open |
| FINDING-03 | `COMMIT WORK` inside `execute` violates LUW contract | Critical | Open |
| FINDING-04 | No substep architecture — timeout risk for large datasets | Critical | Open |
| FINDING-05 | `BRAND` and `HIER1` filter params dropped — silent functional regression | Critical | Open |
| FINDING-06 | `validate` does not guard parameters before destructive DELETE | Moderate | Open |
| FINDING-07 | N+1 SELECT pattern in processing loop | Moderate | Open |
| FINDING-08 | `ZFI_ALLOC_STATE` phase status not integrated with framework lifecycle | Moderate | Open |
| FINDING-09 | `lv_commit_counter` outer increment not in original — behavioral change | Moderate | Open |
| FINDING-10 | Commented-out aggregate items block without design decision | Minor | Open |
| FINDING-11 | `lv_dummy TYPE c` truncates message text | Minor | Open |
| FINDING-12 | `p_commit` behavioral change: 500 → 5000 undocumented | Minor | Open |
| FINDING-13 | Stale "Phase 2" copy-paste strings throughout Phase 3 class | Minor | Open |
| FINDING-14 | Unused variable declarations (`lv_total_aiccc`, `lt_messages`, `lv_items_created`) | Minor | Open |
| FINDING-15 | Redundant `ms_context` field never read after `init` | Minor | Open |

**Transport readiness: BLOCKED** — FINDING-01 (class cannot activate) alone blocks
transport. FINDING-02, FINDING-03, and FINDING-05 are also blocking: FINDING-05 is a
functional regression that will silently produce incorrect results for any allocation
that previously used brand or hierarchy1 filtering.

---

## Next Steps

1. **FINDING-01** — Add stub implementations for `execute_substep`, `on_success`,
   `on_error`. This unblocks activation and allows the other issues to be tested.
2. **FINDING-05** — Add `mv_brand` / `mv_hier1` to private section, read in `init`,
   pass to `get_unassigned`. This is a functional regression fix and must be done
   before any test run.
3. **FINDING-02** — Replace all `MESSAGE e...` abort patterns with
   `RAISE EXCEPTION TYPE zcx_fi_process_error`.
4. **FINDING-03** — Remove `COMMIT WORK` and `lv_commit_counter`. Implement `rollback`
   to clean up partial inserts and reset phase status.
5. **FINDING-04** — Design substep architecture: `execute` orchestrates, `plan_substeps`
   distributes unassigned items, `execute_substep` processes one item, `on_success`
   writes 'F' state. This is the structural improvement to make Phase 3 scale.
6. **FINDING-06** — Add parameter guards to `validate`.
7. **FINDING-13** — Fix "Phase 2" → "Phase 3" strings (5-minute fix, do immediately).
8. **FINDING-07 through FINDING-15** — Address in the same transport (low effort).
