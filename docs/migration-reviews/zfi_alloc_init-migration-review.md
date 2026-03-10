# Migration Review: ZCL_FI_ALLOC_STEP_INIT

**Date:** 2026-03-05
**Step class:** `ZCL_FI_ALLOC_STEP_INIT`
**Process step:** 0001 — INIT (SERIAL substep mode)
**Original program:** None — this is a net-new step with no standalone report counterpart
**Reviewer:** OpenCode AI Agent

---

## Overview

`ZCL_FI_ALLOC_STEP_INIT` initializes the `ZFI_ALLOC_STATE` table row for an allocation run.
When `FORCE_START = abap_true` it deletes an existing state row and re-inserts a fresh one,
allowing a process re-run from scratch. Without `FORCE_START`, it validates that no state row
exists and exits successfully (creating nothing). It is step 0001 in the `ALLOCATIONS` process
definition and is configured as `substep_mode = 'SERIAL'`, `mandatory = 'X'`, `retry_allowed = 'X'`.

This step has **no original standalone program** to compare against. The review is a
framework-quality assessment only.

INIT is the best-quality step in the package and serves as a reference model for the other
steps. It avoids every cross-cutting defect that afflicts PHASE1–PHASE3 and CORR_BCHE:
no `COMMIT WORK`, no `MESSAGE TYPE 'E'`, a real `validate` implementation, and concise
`execute` logic using early RETURN. Despite this, three findings require attention.

---

## Findings

---

### FINDING-01 — Missing `execute_substep`, `on_success`, `on_error` — activation blocker

**Severity:** Critical
**Constitution principle:** II — SAP Standards Compliance

The class declares `INTERFACES zif_fi_process_step` but the implementation block does not
contain methods for `execute_substep`, `on_success`, or `on_error`. These are mandatory
abstract methods of `ZIF_FI_PROCESS_STEP`. The ABAP compiler will refuse to activate the
class until all nine interface methods are implemented.

The cross-cutting analysis (§5) incorrectly listed INIT as "Can activate: Yes". This
review corrects that assessment: **INIT cannot be activated in its current state**.

Note: The process definition marks step 0001 with `substep_mode = 'SERIAL'` — the framework
will never call `execute_substep` for this step. However, the interface still requires the
method to be present.

**Mitigation:** Add the three stub implementations. For a SERIAL step that does not use
substeps, the correct stubs are:

```abap
METHOD zif_fi_process_step~execute_substep.
  " Not used — this step does not support substep execution
  rs_result-success     = abap_false.
  rs_result-message     = 'execute_substep is not supported for this step'.
  rs_result-can_continue = abap_false.
ENDMETHOD.

METHOD zif_fi_process_step~on_success.
  " No post-substep success hook needed
ENDMETHOD.

METHOD zif_fi_process_step~on_error.
  " No post-substep error hook needed
ENDMETHOD.
```

---

### FINDING-02 — `execute` does not insert state row when `FORCE_START = abap_false` and no row exists

**Severity:** Moderate
**Constitution principle:** (behavioral correctness)

When `mv_force_start = abap_false`, the `execute` method's `IF mv_force_start = abap_true`
block is skipped entirely and the method returns `success = abap_true, message = 'Process initialized'`
without inserting any row into `ZFI_ALLOC_STATE`.

```abap
METHOD zif_fi_process_step~execute.
  IF mv_force_start = abap_true.
    DELETE FROM zfi_alloc_state WHERE ...   " DELETE + INSERT path
    INSERT INTO zfi_alloc_state VALUES ...
    IF sy-subrc <> 0.
      rs_result-success = abap_false.
      RETURN.
    ENDIF.
  ENDIF.                                    " <-- normal path falls through here

  rs_result-success     = abap_true.        " <-- success, but no row was written
  rs_result-message     = |Process initialized|.
  rs_result-can_continue = abap_true.
ENDMETHOD.
```

The `validate` method ensures no state row exists before `execute` runs (for the
`force_start = abap_false` path). So `validate` passes → `execute` runs → no INSERT
occurs → step succeeds with `message = 'Process initialized'` but `ZFI_ALLOC_STATE`
is still empty.

Subsequent steps (PHASE1, PHASE2, PHASE3) all read and update `phase_X_status` in
`ZFI_ALLOC_STATE`. If no row exists, their SELECT/UPDATE logic will fail silently or
set `phase_X_status = 'R'` on a non-existent row (UPDATE with WHERE match failing,
`sy-subrc <> 0`, causing each step to report an error and halt).

The correct behavior is: INIT should **always** ensure a `ZFI_ALLOC_STATE` row exists
after successful execution, regardless of `FORCE_START`. The `INSERT` should be
unconditional (preceded by a DELETE only when `FORCE_START` is set):

```abap
METHOD zif_fi_process_step~execute.
  IF mv_force_start = abap_true.
    DELETE FROM zfi_alloc_state
      WHERE company_code  = mv_company_code  AND fiscal_year   = mv_fiscal_year
        AND fiscal_period = mv_fiscal_period AND allocation_id = mv_allocation_id.
  ENDIF.

  INSERT INTO zfi_alloc_state VALUES @( VALUE zfi_alloc_state(
    company_code  = mv_company_code
    fiscal_year   = mv_fiscal_year
    fiscal_period = mv_fiscal_period
    allocation_id = mv_allocation_id
  ) ).
  IF sy-subrc <> 0.
    rs_result-success      = abap_false.
    rs_result-message      = |Failed to initialize state for { mv_allocation_id }|.
    rs_result-can_continue = abap_false.
    RETURN.
  ENDIF.

  rs_result-success      = abap_true.
  rs_result-message      = |Process initialized for { mv_allocation_id }|.
  rs_result-can_continue = abap_true.
ENDMETHOD.
```

With this change, `validate` remains correct: it blocks if a row already exists AND
`FORCE_START` is false (duplicate initialization guard). The `execute` simply performs
the INSERT unconditionally, trusting `validate` to have cleared the path.

---

### FINDING-03 — `ms_context` private field stored in `init` but never read

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (dead code)

The class declares `DATA ms_context TYPE zif_fi_process_step=>ty_context` and assigns
`ms_context = is_context` in `init`. The field is never subsequently accessed.
This is the same pattern present in all five step classes (cross-cutting CC-07).

**Mitigation:** Remove `ms_context` from the PRIVATE SECTION. This change should be
applied uniformly across all five step classes in the same transport.

---

### FINDING-04 — `get_description` does not include key context in the description string

**Severity:** Minor
**Constitution principle:** (usability)

```abap
METHOD zif_fi_process_step~get_description.
  rv_description = 'Perform initialization for allocation ID'.
ENDMETHOD.
```

The description is a static string and does not append the actual `mv_allocation_id`
value. In the process monitor, this would display the same description for every
run, making it harder to distinguish instances. Compare with the inline message
in `validate`:
```abap
rs_result-message = |Allocation ID { mv_allocation_id } is ready|.
```
The `validate` message correctly uses the allocated ID. The description should follow
the same pattern.

**Mitigation:**
```abap
rv_description = |Perform initialization for allocation ID { mv_allocation_id }|.
```

---

### FINDING-05 — Copy-paste header comment from framework template

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (cosmetic)

The class file opens with:
```abap
*& Class ZCL_STEP_FETCH_DATA
*& Example step: Fetch data from database
```

This is the comment from `ZCL_FI_STEP_FETCH_DATA`, the framework's reference example class.
All five step classes share this comment verbatim. See cross-cutting CC-08.

**Mitigation:** Replace with:
```abap
*& Class ZCL_FI_ALLOC_STEP_INIT
*& Allocation process — Step 0001: Initialize allocation state
```

---

## Behavioral Assessment

Unlike the migration reviews for the other four steps, there is no "before vs. after" comparison
here. Instead, this section documents the intended behavior and the delta introduced by FINDING-02.

| Scenario | Current behavior | Expected behavior |
|----------|-----------------|-------------------|
| New allocation, `FORCE_START = false` | `validate` passes, `execute` skips INSERT, returns success | Should INSERT state row |
| New allocation, `FORCE_START = true` | DELETE (0 rows) + INSERT → success ✅ | Same |
| Existing allocation, `FORCE_START = false` | `validate` blocks with "already initialized" ✅ | Same |
| Existing allocation, `FORCE_START = true` | `validate` passes, DELETE + INSERT → success ✅ | Same |
| INSERT fails | Returns `success = abap_false, can_continue = abap_false` ✅ | Same |
| Subsequent PHASE1 with empty state table | Phase 1 UPDATE fails → process halts | State row exists → Phase 1 proceeds |

---

## Summary

| Finding | Severity | Status |
|---------|----------|--------|
| FINDING-01: Missing `execute_substep`, `on_success`, `on_error` | **Critical** | Open |
| FINDING-02: No INSERT when `force_start = false` | **Moderate** | Open |
| FINDING-03: `ms_context` dead field | Minor | Open (part of CC-07 package fix) |
| FINDING-04: `get_description` static string | Minor | Open |
| FINDING-05: Copy-paste header comment | Minor | Open (part of CC-08 package fix) |

**Critical: 1 | Moderate: 1 | Minor: 3**

---

## Next Steps

1. **(R-INIT-01 — Critical):** Add `execute_substep`, `on_success`, `on_error` stubs.
   Can be applied in the same transport as the same fix for PHASE1, CORR_BCHE, and PHASE3
   (cross-cutting CC-01 remediation R-02).
2. **(R-INIT-02 — Moderate):** Restructure `execute` to always INSERT (unconditionally)
   after DELETE-if-forced, as shown in FINDING-02.
3. **(R-INIT-03 — Minor):** Remove `ms_context` field (part of CC-07 package-wide cleanup).
4. **(R-INIT-04 — Minor):** Parameterize `get_description` with `mv_allocation_id`.
5. **(R-INIT-05 — Minor):** Update class header comment (part of CC-08 package-wide cleanup).

**Correction required to cross-cutting analysis:** §5 (Activation Readiness) must be
updated — INIT currently **cannot** activate due to FINDING-01. The pipeline risk
assessment in §6 must also be updated: step 0001 is blocked before step 0003 (CORR_BCHE).
