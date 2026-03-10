# Migration Review: ZFI_ALLOC_CORR_BCHE → ZCL_FI_ALLOC_STEP_CORR_BCHE

**Date:** 2026-03-05
**Source program:** `ZFI_ALLOC_CORR_BCHE`
**Target step class:** `ZCL_FI_ALLOC_STEP_CORR_BCHE`
**Framework:** ZFI_PROCESS (ABAP 7.58 / S/4HANA)
**Reviewer:** OpenCode AI Agent

---

## Overview

This document reviews the migration of the standalone report `ZFI_ALLOC_CORR_BCHE` into the
ZFI_PROCESS framework step class `ZCL_FI_ALLOC_STEP_CORR_BCHE`. The original program corrected
the `NO_ITEMS` flag on `ZFI_ALLOC_BCHE` header records by checking for the existence of matching
rows in `ZFI_ALLOC_BCITM`, processing records in chunks of 10,000 and committing after each chunk.

The migration is **functionally correct in intent** but contains **two critical defects** that
prevent transport readiness, plus several moderate and minor issues.

---

## Findings

### FINDING-01 — `COMMIT WORK` inside step body (Critical)

**Severity:** Critical
**Constitution principle:** V — Error Handling & Observability (LUW ownership)

**Description:**
The `execute` method issues `COMMIT WORK` at the end of every processed chunk:

```abap
" Commit work after processing each chunk
COMMIT WORK.
```

In the framework the LUW is owned by `ZCL_FI_PROCESS_INSTANCE`. The framework itself commits
after recording step status (`execute_step` — `COMMIT WORK AND WAIT` at line 679 and in
`save_instance`/`save_failure_state`). A step that issues intermediate commits:

- Prevents atomic rollback — once a chunk is committed the `rollback` method cannot undo it.
- May commit incomplete framework state (e.g. step status not yet written) if timing interleaves
  with framework saves.
- Will conflict with any future parallel execution model.

The original standalone report committed per chunk because it had no outer LUW controller. In the
framework context each step is a single unit of work.

**Mitigation:**
Remove all `COMMIT WORK` statements from the `execute` method. If the volume of records makes a
single-LUW approach impractical (risk of enqueue timeout or memory), the correct solution is to
split the work into **substeps** — one substep per chunk — and let the framework commit between
substeps. Implement `plan_substeps` to produce one `ty_planned_step` per chunk, passing the offset
via serialized `context_data`, and implement `execute_substep` to process a single chunk.

If a single-LUW approach is acceptable for the expected data volumes, simply remove the commits
and let the framework commit once after `execute` returns.

---

### FINDING-02 — Missing interface method implementations (Critical)

**Severity:** Critical
**Constitution principle:** II — SAP Standards Compliance (compile error)

**Description:**
`ZIF_FI_PROCESS_STEP` requires implementations for `execute_substep`, `on_success`, and
`on_error`. The class provides none. ABAP will refuse to activate the class with a syntax error
because all interface methods must be implemented (even if empty).

Compare with `ZCL_FI_STEP_FETCH_DATA` which provides empty stubs for all three.

**Mitigation:**
Add the three missing method implementations. Minimal stubs are sufficient for a step that does
not use substeps:

```abap
METHOD zif_fi_process_step~execute_substep.
  rs_result = zif_fi_process_step~execute( is_context ).
ENDMETHOD.

METHOD zif_fi_process_step~on_success.
  " No post-substep hook required.
ENDMETHOD.

METHOD zif_fi_process_step~on_error.
  " No post-substep error hook required.
ENDMETHOD.
```

---

### FINDING-03 — `WRITE` statement in class method (Moderate)

**Severity:** Moderate
**Constitution principle:** II — SAP Standards Compliance

**Description:**
The `execute` method contains a carry-over from the standalone report:

```abap
WRITE: / 'Update process completed.'.
```

`WRITE` in a class method context outputs to a spool list that is never displayed in a framework
execution context. The framework communicates results via `rs_result-message`, which is already
correctly populated two lines below.

**Mitigation:**
Remove the `WRITE` statement entirely. The result message is sufficient:

```abap
rs_result-message = |BCHE Correction completed|.
```

---

### FINDING-04 — `validate` does not check required parameters (Moderate)

**Severity:** Moderate
**Constitution principle:** V — Error Handling & Observability

**Description:**
The new step adds `company_code`, `fiscal_year`, and `fiscal_period` to the query filter —
narrowing the scope compared to the original program which only filtered on `allocation_id`. This
is a deliberate and likely correct change. However, the `validate` method returns success
unconditionally without verifying that the required parameters are present. If the process
instance is created without these parameters the step will silently process zero records and
report success — a hard-to-detect data quality issue.

**Mitigation:**
Add parameter presence checks to `validate`:

```abap
METHOD zif_fi_process_step~validate.
  IF mv_allocation_id IS INITIAL.
    rs_result-success      = abap_false.
    rs_result-message      = 'Parameter ALLOCATION_ID is required'.
    rs_result-can_continue = abap_false.
    RETURN.
  ENDIF.
  IF mv_company_code IS INITIAL.
    rs_result-success      = abap_false.
    rs_result-message      = 'Parameter COMPANY_CODE is required'.
    rs_result-can_continue = abap_false.
    RETURN.
  ENDIF.
  IF mv_fiscal_year IS INITIAL.
    rs_result-success      = abap_false.
    rs_result-message      = 'Parameter FISCAL_YEAR is required'.
    rs_result-can_continue = abap_false.
    RETURN.
  ENDIF.
  IF mv_fiscal_period IS INITIAL.
    rs_result-success      = abap_false.
    rs_result-message      = 'Parameter FISCAL_PERIOD is required'.
    rs_result-can_continue = abap_false.
    RETURN.
  ENDIF.
  rs_result-success      = abap_true.
  rs_result-message      = 'BCHE Correction validated'.
  rs_result-can_continue = abap_true.
ENDMETHOD.
```

---

### FINDING-05 — `SELECT *` where only existence check is needed (Moderate)

**Severity:** Moderate
**Constitution principle:** III — Consult SAP Documentation (performance)

**Description:**
The inner loop fetches all columns and all rows from `ZFI_ALLOC_BCITM` into an internal table
solely to check `sy-subrc`:

```abap
SELECT *
  FROM zfi_alloc_bcitm
  INTO TABLE lt_items
  WHERE key_id = ls_header-key_id.
```

The variable `lt_items` is never read after this point. For headers with many items this
unnecessarily transfers large amounts of data from the database to the application server.

**Mitigation:**
Replace with an existence check using `SELECT SINGLE`:

```abap
SELECT SINGLE @abap_true
  FROM zfi_alloc_bcitm
  INTO @DATA(lv_item_exists)
  WHERE key_id = ls_header-key_id.
```

This projects a single constant, requires the database to find at most one row, and avoids
transferring item data to the application server entirely.

---

### FINDING-06 — `lv_item_exists` declared but never used (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (dead code)

**Description:**
The local variable `lv_item_exists TYPE abap_bool` is declared in the `execute` method but is
never assigned or read. It is a copy-paste artefact. ABAP extended program check will flag this
as an unused variable.

**Mitigation:**
Remove the declaration. If FINDING-05 is addressed with `SELECT SINGLE ... INTO @DATA(lv_item_exists)`,
the inline declaration replaces it automatically.

---

### FINDING-07 — `ADD` statement instead of modern `+=` syntax (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (modern ABAP)

**Description:**
The chunk offset is incremented using the legacy `ADD` statement:

```abap
ADD lc_chunk_size TO lv_row_count.
```

ABAP 7.54+ supports the arithmetic assignment operator. The `ADD` statement is retained for
backward compatibility only and should not appear in new or migrated code targeting ABAP 7.58.

**Mitigation:**

```abap
lv_row_count += lc_chunk_size.
```

---

### FINDING-08 — Redundant `ms_context` alongside base class protected fields (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (clean code)

**Description:**
The class stores the full context in a private `ms_context` field via `init`, but then accesses
individual parameters through separate private fields (`mv_allocation_id`, `mv_company_code`,
etc.). The base class `ZCL_FI_PROCESS_STEP` already provides `mv_process_instance_id` and
`mv_step_number` as protected fields. The `ms_context` field is stored but never read after
`init` completes — it is fully superseded by the five individual fields.

**Mitigation:**
Remove `ms_context` from the private section. The individual `mv_*` fields populated in `init`
are sufficient. This reduces memory footprint and eliminates confusion about which field is
authoritative.

---

## Behavioral Change vs. Original Program

The original program filtered only on `allocation_id`. The new step adds `company_code`,
`fiscal_year`, and `fiscal_period` to the WHERE clause. This is a **narrowing of scope** and
is presumed intentional (the framework provides these parameters; the standalone report did not).
However it must be confirmed:

- If a process instance is ever created without all four parameters, zero records will be
  processed and the step will silently succeed (mitigated by FINDING-04).
- If the original report was run with only `allocation_id` and intentionally processed all
  company codes / periods for that allocation, the migrated step changes that behavior.

**Action required:** Confirm with the functional owner that the additional filters are correct
and that no existing use-case relied on the allocation-ID-only filter.

---

## Summary

| ID | Description | Severity | Status |
|----|-------------|----------|--------|
| FINDING-01 | `COMMIT WORK` inside step violates LUW contract | Critical | Open |
| FINDING-02 | Missing `execute_substep`, `on_success`, `on_error` implementations | Critical | Open |
| FINDING-03 | `WRITE` statement in class method | Moderate | Open |
| FINDING-04 | `validate` does not check required parameters | Moderate | Open |
| FINDING-05 | `SELECT *` for existence check — unnecessary data transfer | Moderate | Open |
| FINDING-06 | `lv_item_exists` declared but never used | Minor | Open |
| FINDING-07 | Legacy `ADD` statement instead of `+=` | Minor | Open |
| FINDING-08 | Redundant `ms_context` field never read after `init` | Minor | Open |

**Transport readiness: BLOCKED** — FINDING-01 and FINDING-02 must be resolved before transport.

---

## Next Steps

1. Fix FINDING-02 immediately (add three empty stubs) — zero-risk, required for activation.
2. Decide on LUW strategy for FINDING-01:
   - **Option A (simple):** Remove commits, run as single LUW. Acceptable if data volumes are
     manageable (test with production-size allocation).
   - **Option B (scalable):** Implement chunk-based substeps via `plan_substeps` /
     `execute_substep`. Recommended if a single allocation can have > ~100k headers.
3. Fix FINDING-04 — add parameter validation to `validate`.
4. Address FINDING-03, FINDING-05, FINDING-06, FINDING-07, FINDING-08 as part of the same
   transport (low effort, improves quality).
5. Confirm behavioral change documented in the "Behavioral Change" section with functional owner.
