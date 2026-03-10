# Migration Review: ZFI_ALLOC_PHASE_1 → ZCL_FI_ALLOC_STEP_PHASE1

**Date:** 2026-03-05
**Source program:** `ZFI_ALLOC_PHASE_1`
**Target step class:** `ZCL_FI_ALLOC_STEP_PHASE1`
**Framework:** ZFI_PROCESS (ABAP 7.58 / S/4HANA)
**Reviewer:** OpenCode AI Agent

---

## Overview

This document reviews the migration of the standalone report `ZFI_ALLOC_PHASE_1` into the
ZFI_PROCESS framework step class `ZCL_FI_ALLOC_STEP_PHASE1`. The original program executed
Phase 1 of the allocation process:

1. Validated the allocation state record (`ZFI_ALLOC_STATE`) — checking existence, lock status,
   and initial status.
2. Reset the state to "Running" and cleared all downstream phase timestamps.
3. Deleted existing header (`ZFI_ALLOC_BCHE`) and item (`ZFI_ALLOC_BCITM`) records for the
   allocation run.
4. Called `ZCL_FI_ALLOCATIONS=>GET_BASE_KEYS` to retrieve unique business keys from accounting
   documents.
5. Inserted one `ZFI_ALLOC_BCHE` header record per unique key.
6. Updated the state to "Finished" with end timestamps.

All validation and control-flow logic used `MESSAGE ... TYPE 'E'` to abort the program on error,
which is a valid pattern for standalone reports but is **not compatible** with the framework's
exception-based error handling.

The migration is **functionally close** but contains **three critical defects** and several
additional issues that must be addressed before transport.

---

## Findings

### FINDING-01 — `MESSAGE TYPE 'E'` used for error handling instead of exceptions (Critical)

**Severity:** Critical
**Constitution principle:** V — Error Handling & Observability

**Description:**
The `execute` method uses `MESSAGE e...` statements for all error conditions:

```abap
IF sy-subrc <> 0.
  MESSAGE e001(zfi_alloc) WITH mv_allocation_id |...|.
ENDIF.

IF ls_state-locked = abap_true.
  MESSAGE e002(zfi_alloc) WITH mv_allocation_id |...|.
ENDIF.
```

In a standalone report `MESSAGE TYPE 'E'` aborts execution and displays the error to the user.
Inside a class method it behaves differently — it raises a `CX_SY_NO_HANDLER` runtime exception
(or a `CX_SY_MESSAGE_TYPE_E` depending on the context) that is **not** `ZCX_FI_PROCESS_ERROR`.
The framework's `execute_step` only catches `cx_root` as the last resort fallback and wraps it in
a generic `step_execution_failed` message, losing the original business message text and making
the error untraceable in the process monitor.

Constitution Principle V explicitly states:
> MUST raise ZCX_FI_PROCESS_ERROR (not CX_ROOT or other generic exceptions)
> Error messages MUST include context (step number, process type, instance ID)

**Mitigation:**
Replace every `MESSAGE e...` abort pattern with `RAISE EXCEPTION TYPE zcx_fi_process_error`:

```abap
" Allocation state not found
IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid = zcx_fi_process_error=>step_execution_failed
      value  = |Allocation state not found: { mv_allocation_id } | &&
               |{ mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period }|.
ENDIF.

" Locked
IF ls_state-locked = abap_true.
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid = zcx_fi_process_error=>step_execution_failed
      value  = |Allocation { mv_allocation_id } is locked and cannot be processed|.
ENDIF.

" Already processed (non-initial status)
IF (    ls_state-phase_1_status IS NOT INITIAL
     OR ls_state-phase_2_status IS NOT INITIAL
     OR ls_state-phase_3_status IS NOT INITIAL )
   AND mv_force_start = abap_false.
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid = zcx_fi_process_error=>step_execution_failed
      value  = |Allocation { mv_allocation_id } already processed. | &&
               |Use FORCE_START = true to override.|.
ENDIF.
```

The `MESSAGE s000` informational logging pattern using `lo_log->log_sy_msg( )` is acceptable and
can remain, as it does not affect control flow.

---

### FINDING-02 — `COMMIT WORK` inside step body (Critical)

**Severity:** Critical
**Constitution principle:** V — Error Handling & Observability (LUW ownership)

**Description:**
The `execute` method issues two separate `COMMIT WORK` statements:

```abap
UPDATE zfi_alloc_state FROM <fs_state>.
COMMIT WORK.           " <-- after state reset

DELETE FROM zfi_alloc_bcitm WHERE ...
DELETE FROM zfi_alloc_bche  WHERE ...
COMMIT WORK.           " <-- after bulk delete
```

As documented in the review for `ZCL_FI_ALLOC_STEP_CORR_BCHE` (FINDING-01), `COMMIT WORK` inside
a step violates the framework LUW contract. Here the problem is compounded: there are two commits
at different points in the step, meaning a failure after the second commit (e.g. during the INSERT
loop) leaves the database in a partially written state with no framework-level ability to roll
back, because the state update and the deletes have already been committed independently.

Additionally, the `COMMIT WORK` after the state update is immediately followed by the DELETE
statements — if the DELETEs fail, the state has already been persisted as "Running" with no
corresponding data.

**Mitigation:**
Remove both `COMMIT WORK` statements. All database changes in this step (state update, deletes,
inserts) should form a single LUW that the framework commits after `execute` returns
`rs_result-success = abap_true`. The `rollback` method should then handle the compensating
state reset if needed:

```abap
METHOD zif_fi_process_step~rollback.
  " Reset phase 1 status if execute was interrupted mid-flight
  SELECT SINGLE * FROM zfi_alloc_state
    INTO @DATA(ls_state_rb)
    WHERE company_code  = @mv_company_code
      AND fiscal_year   = @mv_fiscal_year
      AND fiscal_period = @mv_fiscal_period
      AND allocation_id = @mv_allocation_id.
  IF sy-subrc = 0 AND ls_state_rb-phase_1_status = 'R'.
    ls_state_rb-phase_1_status     = space.
    ls_state_rb-phase_1_start_date = '00000000'.
    ls_state_rb-phase_1_start_time = '000000'.
    UPDATE zfi_alloc_state FROM ls_state_rb.
  ENDIF.
ENDMETHOD.
```

---

### FINDING-03 — Missing interface method implementations (Critical)

**Severity:** Critical
**Constitution principle:** II — SAP Standards Compliance (compile error)

**Description:**
`ZIF_FI_PROCESS_STEP` requires implementations for `execute_substep`, `on_success`, and
`on_error`. The class provides none. The class will fail ABAP activation with a syntax error.
This is the same defect as FINDING-02 in the `ZCL_FI_ALLOC_STEP_CORR_BCHE` review.

**Mitigation:**
Add three empty stubs:

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

### FINDING-04 — `p_brand` and `p_hier1` parameters dropped without documentation (Moderate)

**Severity:** Moderate
**Constitution principle:** II — SAP Standards Compliance (behavioral change)

**Description:**
The original program accepted `p_brand` and `p_hier1` as selection screen parameters and passed
them to `ZCL_FI_ALLOCATIONS=>GET_BASE_KEYS`. The migrated step passes hardcoded empty strings:

```abap
" Original program:
zcl_fi_allocations=>get_base_keys( EXPORTING iv_brand = p_brand
                                             iv_hier1  = p_hier1 ... )

" Migrated step:
zcl_fi_allocations=>get_base_keys( EXPORTING iv_brand = ''
                                             iv_hier1  = '' ... )
```

The log messages also reflect this — the original logged the actual brand and hier1 values;
the migrated step logs empty strings:

```abap
MESSAGE s000(zfi_alloc) WITH |Značka: { '' }| INTO lv_dummy.
MESSAGE s000(zfi_alloc) WITH |Hier1: { '' }| INTO lv_dummy.
```

This is a **silent behavioral change**. If the original process relied on brand or hierarchy
filtering to limit the scope of Phase 1 output, the migrated step will produce a superset of
records — potentially causing performance issues or incorrect downstream results.

**Mitigation:**
Either:

- **Option A (full parity):** Add `BRAND` and `HIER1` as process parameters, retrieve them in
  `init`, and pass them through:

```abap
" In private section:
DATA mv_brand TYPE wrf_brand_id.
DATA mv_hier1 TYPE zmatklh1.

" In init:
mv_brand = is_context-io_process_instance->get_init_param_value('BRAND').
mv_hier1 = is_context-io_process_instance->get_init_param_value('HIER1').

" In execute:
zcl_fi_allocations=>get_base_keys(
  EXPORTING iv_brand = mv_brand
            iv_hier1 = mv_hier1 ... ).
```

- **Option B (intentional drop):** If brand/hier1 filtering is deliberately removed for the
  framework execution (e.g. framework always processes the full allocation), document this
  explicitly in the class header comment and confirm with the functional owner.

---

### FINDING-05 — `validate` does not check required parameters (Moderate)

**Severity:** Moderate
**Constitution principle:** V — Error Handling & Observability

**Description:**
Identical issue to FINDING-04 in the `ZCL_FI_ALLOC_STEP_CORR_BCHE` review. The `validate`
method returns success unconditionally. Phase 1 performs destructive operations (DELETE on
`ZFI_ALLOC_BCITM` and `ZFI_ALLOC_BCHE`) — a step that silently proceeds with initial parameters
could delete records for the wrong allocation.

**Mitigation:**

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
  rs_result-message      = 'Phase 1 validated'.
  rs_result-can_continue = abap_true.
ENDMETHOD.
```

This is especially important for this step given the destructive DELETE operations — a guard in
`validate` prevents accidental data loss on misconfigured process instances.

---

### FINDING-06 — `p_commit` parameter and `lv_commit_counter` present but unused (Moderate)

**Severity:** Moderate
**Constitution principle:** II — SAP Standards Compliance (dead code / incomplete migration)

**Description:**
The original program declared `p_commit TYPE int4 DEFAULT 500` as a selection screen parameter,
presumably to control intermediate commit frequency during the INSERT loop. The migrated step
declares `lv_commit_counter TYPE int4` as a local variable but neither initialises it with a
meaningful value nor ever reads it to trigger a commit. The variable is only cleared inside the
loop (`CLEAR lv_commit_counter`) and then ignored.

This suggests the intermediate commit logic was intentionally removed (consistent with FINDING-02
addressing LUW ownership) but the counter variable was left in as dead code. Additionally, since
`COMMIT WORK` statements are being removed per FINDING-02, the counter serves no purpose.

**Mitigation:**
Remove `lv_commit_counter` from the local variable declarations and remove the `CLEAR lv_commit_counter`
statement inside the loop. If a future decision is made to implement substep-based chunking
(see FINDING-02 mitigation option), this would be re-introduced as part of that design.

---

### FINDING-07 — Allocation state managed inside step instead of via framework status (Moderate)

**Severity:** Moderate
**Constitution principle:** V — Error Handling & Observability

**Description:**
The step directly writes phase status values ('R', 'F') to the `ZFI_ALLOC_STATE` DDIC table,
maintaining its own parallel lifecycle state outside the ZFI_PROCESS framework:

```abap
<fs_state>-phase_1_status     = 'R'.   " Running
...
UPDATE zfi_alloc_state FROM <fs_state>.

" ... at the end:
<fs_state>-phase_1_status   = 'F'.    " Finished
UPDATE zfi_alloc_state FROM <fs_state>.
```

The framework tracks step status in `ZFI_PROC_STEP` (RUNNING → COMPLETED/FAILED). Having a
second, parallel status table introduces a risk of divergence — if the framework marks the step
FAILED (e.g. on an exception after the first commit), the allocation state will remain 'R'
indefinitely with no automatic reset.

This is a deeper architectural concern rather than a simple code defect. It reflects the original
program's self-contained state machine which has not been fully integrated with the framework
lifecycle.

**Mitigation:**
Two approaches depending on whether `ZFI_ALLOC_STATE` is consumed by other components outside
the framework:

- **Option A (short-term):** Keep the `ZFI_ALLOC_STATE` updates but ensure the `rollback`
  method resets `phase_1_status` to its pre-execution value (see FINDING-02 rollback mitigation).
  Document that `ZFI_ALLOC_STATE` is the business-facing status while `ZFI_PROC_STEP` is the
  framework-facing status.

- **Option B (long-term):** Retire the `phase_*_status` columns on `ZFI_ALLOC_STATE` and derive
  business status from `ZFI_PROC_STEP` via the process instance. Use the framework's business
  status mechanism (`set_business_status`) to surface allocation phase status in the process
  monitor.

---

### FINDING-08 — BAL log initialized with `CREATE OBJECT` instead of factory pattern (Minor)

**Severity:** Minor
**Constitution principle:** IV — Factory Pattern & Encapsulation

**Description:**
The log object is created with a direct `CREATE OBJECT` call:

```abap
CREATE OBJECT lo_log.
```

Constitution Principle IV states direct `NEW()` / `CREATE OBJECT` instantiation is prohibited
for framework classes. While `ZCL_SP_BALLOG` is an external class (not a ZFI_PROCESS framework
class), using `CREATE OBJECT` is the older syntax. Modern ABAP 7.40+ uses `NEW`:

```abap
lo_log = NEW zcl_sp_ballog( ).
```

More importantly, if `ZCL_SP_BALLOG` ever adds mandatory constructor parameters, this call will
fail silently at runtime. Check whether the class provides a factory method.

**Mitigation:**
Replace with `NEW` operator:

```abap
lo_log = NEW zcl_sp_ballog( ).
```

If `ZCL_SP_BALLOG` provides a factory method (e.g. `GET_INSTANCE` or `CREATE`), use that instead.

---

### FINDING-09 — `lv_dummy`, `ls_bcit`, `lt_messages` declared but partially or never used (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (dead code)

**Description:**
Three local variables are dead:

- `ls_bcit TYPE zfi_alloc_bcitm` — declared, never assigned or read.
- `lt_messages TYPE bapirettab` — declared, never used.
- `lv_dummy TYPE c` — used as a `MESSAGE ... INTO` sink to capture the message without
  triggering a dialog. This is a valid technique in class methods to suppress the message
  dialog; however, the type `c` (length 1) will silently truncate any message text longer than
  one character. It should be `TYPE string` or a sufficiently long fixed-length type.

**Mitigation:**
- Remove `ls_bcit` and `lt_messages` declarations entirely.
- Change `lv_dummy TYPE c` to `lv_dummy TYPE string` to avoid silent truncation of log messages.

---

### FINDING-10 — Redundant `ms_context` field (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (clean code)

**Description:**
Identical issue to FINDING-08 in the `ZCL_FI_ALLOC_STEP_CORR_BCHE` review. The full context
is stored in `ms_context` but is never read after `init` populates the individual `mv_*` fields.

**Mitigation:**
Remove `ms_context` from the private section.

---

## Behavioral Changes vs. Original Program

| Aspect | Original program | Migrated step | Risk |
|--------|-----------------|---------------|------|
| Brand filter | `p_brand` passed to `GET_BASE_KEYS` | Hardcoded `''` | High — superset of records produced |
| Hier1 filter | `p_hier1` passed to `GET_BASE_KEYS` | Hardcoded `''` | High — superset of records produced |
| Commit granularity | Intermediate commits during insert loop via `p_commit` counter | Two commits (state + delete), no insert-loop commits | Medium — single LUW is safer but may time out on large datasets |
| Error handling | `MESSAGE TYPE 'E'` aborts report | Same pattern in class method — unreliable behavior | Critical — see FINDING-01 |
| Log messages brand/hier1 | Logs actual parameter values | Logs empty strings | Low — cosmetic only |

---

## Summary

| ID | Description | Severity | Status |
|----|-------------|----------|--------|
| FINDING-01 | `MESSAGE TYPE 'E'` used for error control flow in class method | Critical | Open |
| FINDING-02 | `COMMIT WORK` inside step violates LUW contract | Critical | Open |
| FINDING-03 | Missing `execute_substep`, `on_success`, `on_error` implementations | Critical | Open |
| FINDING-04 | `p_brand` and `p_hier1` silently dropped — behavioral change | Moderate | Open |
| FINDING-05 | `validate` does not guard required parameters before destructive DELETEs | Moderate | Open |
| FINDING-06 | `lv_commit_counter` declared and cleared but never used | Moderate | Open |
| FINDING-07 | Parallel allocation state machine (`ZFI_ALLOC_STATE`) not integrated with framework lifecycle | Moderate | Open |
| FINDING-08 | `CREATE OBJECT` instead of `NEW` for BAL log instantiation | Minor | Open |
| FINDING-09 | `ls_bcit`, `lt_messages` unused; `lv_dummy TYPE c` truncates log messages | Minor | Open |
| FINDING-10 | Redundant `ms_context` field never read after `init` | Minor | Open |

**Transport readiness: BLOCKED** — FINDING-01, FINDING-02, and FINDING-03 must be resolved
before transport. FINDING-04 must be confirmed or explicitly documented as intentional before
transport.

---

## Next Steps

1. **FINDING-03** — Add three empty interface stubs immediately (zero-risk, required for
   activation).
2. **FINDING-01** — Replace all `MESSAGE TYPE 'E'` abort patterns with
   `RAISE EXCEPTION TYPE zcx_fi_process_error`. This is the most impactful change for framework
   reliability.
3. **FINDING-02** — Remove both `COMMIT WORK` statements. Implement `rollback` to reset
   `phase_1_status` to initial if the step is interrupted.
4. **FINDING-04** — Confirm with functional owner whether brand/hier1 filtering is intentionally
   dropped in the framework context. If yes, add a comment to the class header explaining why.
   If no, add `mv_brand` and `mv_hier1` parameters.
5. **FINDING-05** — Add parameter guards to `validate` before destructive operations.
6. **FINDING-06, FINDING-07, FINDING-08, FINDING-09, FINDING-10** — Address as part of the same
   transport (low effort).
7. For FINDING-07, agree with the functional owner on the long-term strategy for
   `ZFI_ALLOC_STATE` vs. framework step status — open as a separate architectural decision
   if not resolvable in this transport.
