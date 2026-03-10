# Migration Review: ZFI_ALLOC_PHASE_2 → ZCL_FI_ALLOC_STEP_PHASE2

**Date:** 2026-03-05
**Source program:** `ZFI_ALLOC_PHASE_2`
**Target step class:** `ZCL_FI_ALLOC_STEP_PHASE2`
**Framework:** ZFI_PROCESS (ABAP 7.58 / S/4HANA)
**Reviewer:** OpenCode AI Agent

---

## Overview

This document reviews the migration of the standalone report `ZFI_ALLOC_PHASE_2` into the
ZFI_PROCESS framework step class `ZCL_FI_ALLOC_STEP_PHASE2`.

The original program had two execution modes controlled by a radio button group:

- **`p_run` (Run mode):** Validated the allocation state, deleted existing items
  (`ZFI_ALLOC_BCITM`), loaded base headers from Phase 1, and for each header called
  `apply_rules_pre` / `get_base_by_key_new` to derive and insert allocation items. Updated
  `ZFI_ALLOC_BCHE` totals and the state to "Finished".
- **`p_postp` (Post-processing mode):** Performed a post-processing rules pass via
  `apply_rules_post`. This mode was guarded at the very top by `MESSAGE TYPE 'A'` (abort)
  making it effectively disabled even in the original program.

The migrated step is architecturally the most advanced of the three reviewed so far. It correctly
uses `plan_substeps` to distribute per-header work across parallel substeps and implements
`execute_substep` for the per-header processing. The `p_postp` block has been commented out
entirely, which matches the original program's abort guard on that branch.

However, the step has **three critical defects** and several additional issues. Most critically,
the `execute` method is **structurally incomplete** — the entire rules-application and item
insertion logic that belongs in the main execution path has been moved to `execute_substep` but
the `execute` method body does not reflect this correctly, logging summary counters that are
always zero and missing the DELETE that should happen before substep planning.

---

## Findings

### FINDING-01 — `MESSAGE TYPE 'E'` used for error handling instead of exceptions (Critical)

**Severity:** Critical
**Constitution principle:** V — Error Handling & Observability

**Description:**
The `execute` method uses `MESSAGE e...` statements for the three validation guards, identical
to the issue found in `ZCL_FI_ALLOC_STEP_PHASE1` (FINDING-01):

```abap
IF sy-subrc <> 0.
  MESSAGE e001(zfi_alloc) WITH mv_allocation_id |...|.
ENDIF.

IF ls_state-locked = abap_true.
  MESSAGE e002(zfi_alloc) WITH mv_allocation_id |...|.
ENDIF.

IF ( ls_state-phase_2_status IS NOT INITIAL ... ) AND mv_force_start = abap_false.
  MESSAGE e003(zfi_alloc) WITH mv_allocation_id |...| 'II'.
ENDIF.
```

In a class method `MESSAGE TYPE 'E'` does not cleanly abort execution. It raises
`CX_SY_NO_HANDLER` / `CX_SY_MESSAGE_TYPE_E` which the framework catches only as `cx_root`
fallback, losing the original business message text in the process monitor.

**Mitigation:**
Replace all three `MESSAGE e...` abort patterns with `RAISE EXCEPTION TYPE zcx_fi_process_error`,
exactly as described in `ZCL_FI_ALLOC_STEP_PHASE1` FINDING-01:

```abap
IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid = zcx_fi_process_error=>step_execution_failed
      value  = |Allocation state not found: { mv_allocation_id } | &&
               |{ mv_company_code }/{ mv_fiscal_year }/{ mv_fiscal_period }|.
ENDIF.
```

The same pattern applies to the locked check and the already-processed check.

---

### FINDING-02 — `COMMIT WORK` inside `execute` violates LUW contract (Critical)

**Severity:** Critical
**Constitution principle:** V — Error Handling & Observability (LUW ownership)

**Description:**
The `execute` method issues one `COMMIT WORK` after the state update to 'R':

```abap
UPDATE zfi_alloc_state FROM <fs_state>.
COMMIT WORK.
```

This is the same structural violation as FINDING-02 in both previous reviews. The state is
committed before the DELETE of `ZFI_ALLOC_BCITM` runs, and before any substep is planned.
If the step fails after this commit, the state remains 'R' with no automatic reset and the
framework cannot roll it back.

Additionally, the commented-out `p_postp` block also contained a `COMMIT WORK` that would
have the same problem if that block were ever re-enabled.

**Mitigation:**
Remove the `COMMIT WORK`. The state update, DELETE, and subsequent substep planning should all
be part of a single LUW. Implement `rollback` to reset `phase_2_status` to initial if the step
is interrupted:

```abap
METHOD zif_fi_process_step~rollback.
  SELECT SINGLE * FROM zfi_alloc_state
    INTO @DATA(ls_state_rb)
    WHERE company_code  = @mv_company_code
      AND fiscal_year   = @mv_fiscal_year
      AND fiscal_period = @mv_fiscal_period
      AND allocation_id = @mv_allocation_id.
  IF sy-subrc = 0 AND ls_state_rb-phase_2_status = 'R'.
    ls_state_rb-phase_2_status     = space.
    ls_state_rb-phase_2_start_date = '00000000'.
    ls_state_rb-phase_2_start_time = '000000'.
    UPDATE zfi_alloc_state FROM ls_state_rb.
  ENDIF.
ENDMETHOD.
```

---

### FINDING-03 — `execute` method is structurally incomplete — core logic missing (Critical)

**Severity:** Critical
**Constitution principle:** V — Error Handling & Observability / II — SAP Standards Compliance

**Description:**
This is the most significant structural defect in this step. The original program's main
processing loop (rules application, `get_base_by_key_new`, item insertion, header totals update)
has been correctly moved to `execute_substep`. However, the `execute` method was not updated to
reflect this restructuring. Specifically:

**Problem A — Summary counter log messages fire before any substeps run:**
The `execute` method logs the summary counters (`lv_zvd`, `lv_euh`, `lv_dvfct`, `lv_amdp_error`)
immediately after loading header keys — before `plan_substeps` is called and before any substep
executes. All counters are zero at this point:

```abap
" Counters are all 0 here — no substeps have run yet
MESSAGE s000(zfi_alloc) WITH |Počet hlaviček základní tabulky s hodnotou 0: { lv_zvd }| ...
MESSAGE s000(zfi_alloc) WITH |Počet chyb zápisu do tabulky ZFI_ALLOC_BCHE: { lv_euh }| ...
MESSAGE s000(zfi_alloc) WITH |Počet chyb vložení do tabulky ZFI_ALLOC_BCITM: { lv_dvfct }| ...
MESSAGE s000(zfi_alloc) WITH |Počet chyb volání AMDP funkce: { lv_amdp_error }| ...
```

These messages will always log "0" regardless of the actual processing outcome. The
`lv_euh <> 0 OR lv_dvfct <> 0` error check that follows will always take the success branch
(`MESSAGE s009`).

**Problem B — `phase_2_status = 'F'` written before substeps complete:**
The `execute` method writes the "Finished" state to `ZFI_ALLOC_STATE` at its own end:

```abap
<fs_state>-phase_2_status   = 'F'.
<fs_state>-phase_2_end_date = sy-datum.
<fs_state>-phase_2_end_time = sy-uzeit.
UPDATE zfi_alloc_state FROM <fs_state>.
```

But substeps are dispatched and run *after* `execute` returns (the framework calls
`plan_substeps` and then processes substeps after `execute` completes). This means the
allocation state will be marked "Finished" before the actual item-level processing has run.

**Problem C — `MESSAGE e008` in `execute` path:**
The error check `IF lv_euh <> 0 OR lv_dvfct <> 0 ... MESSAGE e008` fires in `execute` before
substeps run. This will always take the `MESSAGE s009` (success) branch, masking any real
substep errors.

**Mitigation:**
Restructure `execute` to be a pure orchestration method: validate, set state to 'R', delete
items, load keys into `mt_base_header`, log the key count, and return `rs_result-success = abap_true`
to signal the framework to call `plan_substeps`. Move the summary log messages, the "Finished"
state update, and the error/success message to the `on_success` / `on_error` hooks:

```abap
METHOD zif_fi_process_step~execute.
  " ... validation (with RAISE EXCEPTION per FINDING-01) ...
  " ... set phase_2_status = 'R', COMMIT removed per FINDING-02 ...
  " ... DELETE FROM zfi_alloc_bcitm ...
  " ... load mt_base_header (SELECT key_id) ...
  " ... log key count ...

  rs_result-success      = abap_true.
  rs_result-message      = |Phase 2 ready: { lines( mt_base_header ) } keys loaded|.
  rs_result-can_continue = abap_true.
ENDMETHOD.

METHOD zif_fi_process_step~on_success.
  " All substeps completed — mark state as Finished
  SELECT SINGLE * FROM zfi_alloc_state INTO @DATA(ls_state)
    WHERE company_code = @mv_company_code AND fiscal_year = @mv_fiscal_year
      AND fiscal_period = @mv_fiscal_period AND allocation_id = @mv_allocation_id.
  IF sy-subrc = 0.
    ls_state-phase_2_status   = 'F'.
    ls_state-phase_2_end_date = sy-datum.
    ls_state-phase_2_end_time = sy-uzeit.
    UPDATE zfi_alloc_state FROM ls_state.
  ENDIF.
ENDMETHOD.

METHOD zif_fi_process_step~on_error.
  " Substep(s) failed — mark state as Error or reset to initial per rollback design
ENDMETHOD.
```

---

### FINDING-04 — `execute_substep` catches exceptions but continues after error (Moderate)

**Severity:** Moderate
**Constitution principle:** V — Error Handling & Observability

**Description:**
In `execute_substep`, when `cx_shdb_exception` or `cx_root` is caught, the method sets
`rs_result-success = abap_false` and `rs_result-can_continue = abap_false`. However, execution
does not `RETURN` after setting these flags — the code falls through to the `LOOP AT lt_base_with_items`
block which will iterate over an empty `lt_base_with_items` (since the exception occurred before
`et_result` was populated). This is not catastrophic in this specific case, but it is fragile.
More importantly, the `rs_result-can_continue = abap_false` is set but then potentially
overwritten at the bottom of the method:

```abap
IF rs_result-success = abap_true AND rs_result-can_continue = abap_true.
  rs_result-message = |Substep executed for ...|.
ENDIF.
```

This guard protects the message but the earlier overwrite risk remains if the loop modifies
`rs_result` fields after the CATCH block.

Additionally, `rs_result-message = lv_dummy` after a MESSAGE catch is unreliable — `lv_dummy`
is `TYPE c` (length 1) and will contain only the first character of the message text
(see also FINDING-10).

**Mitigation:**
Add `RETURN` immediately after setting failure flags in each CATCH block:

```abap
CATCH cx_shdb_exception INTO DATA(lx_shdb).
  rs_result-success      = abap_false.
  rs_result-message      = lx_shdb->get_text( ).
  rs_result-can_continue = abap_false.
  RETURN.
CATCH cx_root INTO DATA(lx_root).
  rs_result-success      = abap_false.
  rs_result-message      = lx_root->get_text( ).
  rs_result-can_continue = abap_false.
  RETURN.
```

Use `lx_shdb->get_text( )` / `lx_root->get_text( )` directly instead of `lv_dummy`.

---

### FINDING-05 — `p_postp` (post-processing) mode entirely commented out without design decision (Moderate)

**Severity:** Moderate
**Constitution principle:** II — SAP Standards Compliance (incomplete migration)

**Description:**
The original program's `p_postp` execution branch — which called `apply_rules_post` — is
entirely commented out in the migrated step with the comment:

```abap
*   Rules post processing - was not used in program zfi_alloc_phase_2
```

The original program did block this path at runtime with `MESSAGE TYPE 'A'` at the top, so it
was effectively never executed. However, the code for the post-processing path was fully
implemented in the program. The migration comment claims it "was not used" without documenting
whether this is a deliberate permanent decision or a deferred item.

If the post-processing logic is permanently retired, the commented-out block should be deleted
entirely — dead commented code violates SAP clean code guidelines and creates confusion about
intent. If it is deferred, it should be tracked as a separate story.

**Mitigation:**
- **If permanently dropped:** Delete the commented-out `p_postp` block and add a class-level
  comment explaining the decision.
- **If deferred:** Remove the commented code, create a separate story for
  `ZCL_FI_ALLOC_STEP_PHASE2_POST` (a distinct step class), and register it in the process
  definition as an optional step.

---

### FINDING-06 — `validate` does not check required parameters before destructive DELETE (Moderate)

**Severity:** Moderate
**Constitution principle:** V — Error Handling & Observability

**Description:**
Identical issue to FINDING-05 in `ZCL_FI_ALLOC_STEP_PHASE1`. The `validate` method returns
success unconditionally. Phase 2 performs a `DELETE FROM zfi_alloc_bcitm` — running with
initial parameters would delete items across all allocations for an empty company code / fiscal
year combination, or match nothing silently depending on the table key structure.

**Mitigation:**
Same pattern as the Phase 1 review:

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
  rs_result-success = abap_true.  rs_result-message = 'Phase 2 validated'.
  rs_result-can_continue = abap_true.
ENDMETHOD.
```

---

### FINDING-07 — `SELECT *` in `execute` loads full header rows; only `key_id` needed (Moderate)

**Severity:** Moderate
**Constitution principle:** III — Consult SAP Documentation (performance)

**Description:**
The `execute` method loads base headers into `mt_base_header` for use in `plan_substeps`:

```abap
SELECT key_id FROM zfi_alloc_bche WHERE ...
INTO CORRESPONDING FIELDS OF TABLE @mt_base_header.
```

The `SELECT key_id` projection is correct — only `key_id` is needed to plan substeps. However,
`mt_base_header` is declared as `TYPE TABLE OF zfi_alloc_bche` (full wide row) and the SELECT
uses `CORRESPONDING FIELDS OF`. This means a full-width internal table is allocated in memory
for all headers even though only `key_id` is used. For large allocations with many thousands of
headers, this wastes application server memory.

Compare this with the original program which did `SELECT *` for full headers — the original
was correct because it used all fields in the processing loop. The step's `execute_substep`
re-reads the full header from the database per substep (`SELECT SINGLE * FROM zfi_alloc_bche`)
so storing the full row in `mt_base_header` is redundant.

**Mitigation:**
Change `mt_base_header` to a lean key-only internal table type, or use an inline DATA declaration:

```abap
DATA lt_keys TYPE STANDARD TABLE OF zfi_alloc_s_bche_key WITH DEFAULT KEY.
SELECT key_id FROM zfi_alloc_bche
  WHERE company_code  = @mv_company_code
    AND fiscal_year   = @mv_fiscal_year
    AND fiscal_period = @mv_fiscal_period
    AND allocation_id = @mv_allocation_id
  INTO CORRESPONDING FIELDS OF TABLE @lt_keys.
```

Where `ZFI_ALLOC_S_BCHE_KEY` is a DDIC structure containing only `key_id`. Then update
`plan_substeps` to loop over `lt_keys` (as an instance attribute) instead of `mt_base_header`.
Alternatively keep `mt_base_header` but add a narrow DDIC table type for it.

---

### FINDING-08 — `mo_log` is `NULL` in `execute_substep` when called via bgRFC (Moderate)

**Severity:** Moderate
**Constitution principle:** V — Error Handling & Observability

**Description:**
`mo_log` is a private instance attribute initialized in `execute`:

```abap
CREATE OBJECT mo_log.
...
mo_log->log_create( is_log = ls_log ).
```

When substeps are dispatched via bgRFC (queued mode), each substep runs in a **separate LUW in
a separate work process** — a completely new object instance is created by `instantiate_step`.
The `init` method is called on this fresh instance, but `init` does not initialize `mo_log`.
Therefore when `execute_substep` calls `mo_log->log_sy_msg( )`, `mo_log` is `NULL` and a
`CX_SY_REF_IS_INITIAL` dump will occur.

Even in serial substep mode, `execute_substep` is called on the **same instance** that ran
`execute`, so `mo_log` would be populated — but this only works by coincidence of execution
order and breaks the moment queued mode is used.

**Mitigation:**
Move `mo_log` initialization into `init` so it is always available regardless of which method
is called first and regardless of execution mode:

```abap
METHOD zif_fi_process_step~init.
  ms_context       = is_context.
  mv_allocation_id = is_context-io_process_instance->get_init_param_value( 'ALLOCATION_ID' ).
  mv_company_code  = is_context-io_process_instance->get_init_param_value( 'COMPANY_CODE' ).
  mv_fiscal_year   = is_context-io_process_instance->get_init_param_value( 'FISCAL_YEAR' ).
  mv_fiscal_period = is_context-io_process_instance->get_init_param_value( 'FISCAL_PERIOD' ).
  mv_force_start   = is_context-io_process_instance->get_init_param_value( 'FORCE_START' ).

  " Initialize log — must happen in init so execute_substep can use it in bgRFC context
  mo_log = NEW zcl_sp_ballog( ).
  DATA ls_log TYPE bal_s_log.
  ls_log-extnumber = |{ mv_company_code }/{ mv_fiscal_year }| &&
                     |/{ mv_fiscal_period }/{ mv_allocation_id }|.
  ls_log-object    = zcl_fi_allocations=>c_bal_object.
  ls_log-subobject = zcl_fi_allocations=>c_bal_subobj_phase2.
  mo_log->log_create( is_log = ls_log ).
ENDMETHOD.
```

---

### FINDING-09 — Parallel allocation state machine not integrated with framework lifecycle (Moderate)

**Severity:** Moderate
**Constitution principle:** V — Error Handling & Observability

**Description:**
Identical architectural concern to FINDING-07 in `ZCL_FI_ALLOC_STEP_PHASE1`. The step writes
`phase_2_status = 'R'` in `execute` and `phase_2_status = 'F'` in `execute` (before substeps
run — see FINDING-03). There is no framework-level notification to `ZFI_ALLOC_STATE` if the
step ultimately fails at the substep level, leaving `phase_2_status` stuck at 'F' even if the
allocation items are incomplete.

**Mitigation:**
Same as Phase 1 FINDING-07. Once FINDING-03 is addressed by moving the 'F' write to
`on_success` / `on_error`, this is partially resolved. For a full solution, consider using the
framework's `set_business_status` mechanism to surface allocation phase status.

---

### FINDING-10 — Rule number comparison uses hardcoded string literals (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (magic strings)

**Description:**
The profit center override logic uses a chain of hardcoded string comparisons for rule IDs:

```abap
IF lv_rule = '00001' OR lv_rule = '00002' OR lv_rule = '00003' OR lv_rule = '00004'
                     OR lv_rule = '00007' OR lv_rule = '00008' OR lv_rule = '00009'
                     OR lv_rule = '00010' OR lv_rule = '00011'.
```

This is a direct copy from the original program and represents a list of special-case rule IDs
that require a profit center override. Magic string constants violate constitution Principle II.
If a new rule is added to this special-case group, the developer must know to update this code.

**Mitigation:**
Define a constant range table or a set of named constants in `ZCL_FI_ALLOCATIONS_RULES` (or a
DDIC customizing table) that identifies which rules require profit center override. This
centralizes the business rule configuration and removes the magic strings from the step class.
At minimum, extract to named constants in the class:

```abap
CONSTANTS:
  lc_rule_profit_center_rules TYPE string
    VALUE '00001|00002|00003|00004|00007|00008|00009|00010|00011'.

IF lv_rule CA lc_rule_profit_center_rules. " or use a RANGE check
```

Or better — add a method `ZCL_FI_ALLOCATIONS_RULES=>IS_PROFIT_CENTER_OVERRIDE_RULE( iv_rule )`
that encapsulates this logic.

---

### FINDING-11 — `lv_dummy TYPE c` truncates log message content (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance

**Description:**
Both `execute` and `execute_substep` use `lv_dummy TYPE c` (length 1) as the `INTO` target for
`MESSAGE ... INTO lv_dummy`. This truncates message text to one character. When
`rs_result-message = lv_dummy` is set after an error catch, the result message is a single
character — useless for diagnosis in the process monitor.

This is the same defect as FINDING-09 in `ZCL_FI_ALLOC_STEP_PHASE1`.

**Mitigation:**
Change to `lv_dummy TYPE string` throughout both methods.

---

### FINDING-12 — `lt_messages`, `ls_alloc_item`, `lv_unassigned_amount`, `lv_headers_deleted` unused (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (dead code)

**Description:**
Several variables are declared but never used in either `execute` or `execute_substep`:

- `lt_messages TYPE bapirettab` — declared in `execute`, never used.
- `ls_alloc_item TYPE zfi_alloc_items` — declared in both `execute` and `execute_substep`,
  never used.
- `lv_unassigned_amount TYPE fis_hsl` — declared in `execute_substep`, never used.
- `lv_headers_deleted TYPE int4` — declared in both, never used.
- `lv_commit_counter TYPE int4` — declared in `execute_substep`, incremented but the commit
  block is commented out. The variable increment (`lv_items_created += 1`) still happens but
  the commit counter (`lv_commit_counter`) itself is never incremented (the increment line is
  commented out too). Dead variable.

**Mitigation:**
Remove all unused variable declarations. The ABAP extended program check will confirm the
complete list.

---

### FINDING-13 — Redundant `ms_context` field (Minor)

**Severity:** Minor
**Constitution principle:** II — SAP Standards Compliance (clean code)

**Description:**
Identical to the same finding in both previous reviews. `ms_context` is stored in `init` but
never read after the individual `mv_*` fields are populated.

**Mitigation:**
Remove `ms_context` from the private section.

---

## Behavioral Changes vs. Original Program

| Aspect | Original program | Migrated step | Risk |
|--------|-----------------|---------------|------|
| Execution mode | `p_run` / `p_postp` radio button | Only run mode; post-processing commented out | Low if post was never used, but undocumented |
| Processing granularity | Single sequential loop over all headers | One substep per header via `plan_substeps` | Positive change — enables parallelism |
| Item DELETE before processing | Done in `p_run` block | Done in `execute` | Same — correct |
| State 'F' timing | Written after loop completes | Written in `execute` before substeps run | Critical — state incorrect until FINDING-03 fixed |
| Summary counters | Logged after loop with actual values | Logged in `execute` with all-zero values | Critical — log messages misleading |
| Commit granularity | Per `p_commit` rows (default 500) | Commented out — per substep (framework commits) | Positive change — consistent with framework LUW |
| Error on write failures | `MESSAGE e008` if `lv_euh` or `lv_dvfct` > 0 | Always takes success branch (`lv_euh`/`lv_dvfct` = 0 at log time) | Critical — errors silently swallowed |
| `apply_rules_post` | Available (but disabled by `MESSAGE TYPE 'A'`) | Commented out | Low risk if confirmed permanently dropped |

---

## Summary

| ID | Description | Severity | Status |
|----|-------------|----------|--------|
| FINDING-01 | `MESSAGE TYPE 'E'` used for validation error control flow | Critical | Open |
| FINDING-02 | `COMMIT WORK` inside `execute` violates LUW contract | Critical | Open |
| FINDING-03 | `execute` logs zero counters and writes 'F' state before substeps run | Critical | Open |
| FINDING-04 | `execute_substep` continues after CATCH; `rs_result-message` uses `lv_dummy` | Moderate | Open |
| FINDING-05 | `p_postp` block commented out without explicit design decision | Moderate | Open |
| FINDING-06 | `validate` does not guard parameters before destructive DELETE | Moderate | Open |
| FINDING-07 | `mt_base_header` is full-width table; only `key_id` needed for substep planning | Moderate | Open |
| FINDING-08 | `mo_log` is NULL in `execute_substep` under bgRFC / queued mode | Moderate | Open |
| FINDING-09 | `ZFI_ALLOC_STATE` phase status not integrated with framework lifecycle | Moderate | Open |
| FINDING-10 | Hardcoded rule ID strings for profit center override logic | Minor | Open |
| FINDING-11 | `lv_dummy TYPE c` truncates log messages; `rs_result-message` receives 1 char | Minor | Open |
| FINDING-12 | Multiple unused variable declarations across `execute` and `execute_substep` | Minor | Open |
| FINDING-13 | Redundant `ms_context` field never read after `init` | Minor | Open |

**Transport readiness: BLOCKED** — FINDING-01, FINDING-02, and FINDING-03 must be resolved
before transport. FINDING-08 (NULL log reference under bgRFC) must also be fixed before
enabling queued substep mode. FINDING-05 must be confirmed or documented.

---

## Next Steps

1. **FINDING-01** — Replace `MESSAGE TYPE 'E'` patterns with `RAISE EXCEPTION TYPE zcx_fi_process_error`.
2. **FINDING-02** — Remove `COMMIT WORK`. Implement `rollback` for state reset.
3. **FINDING-03** — Restructure `execute` to be orchestration-only. Move the 'F' state write
   and summary logging to `on_success`. Move the error detection (non-zero `lv_euh` / `lv_dvfct`)
   to `on_error`. Consider aggregating substep error counts via `result_data` on each substep.
4. **FINDING-08** — Move `mo_log` initialization from `execute` to `init` so it is always
   available regardless of execution path. This is required before enabling queued mode.
5. **FINDING-04** — Add `RETURN` after CATCH blocks in `execute_substep`; use exception
   `get_text( )` instead of `lv_dummy` for result messages.
6. **FINDING-06** — Add parameter guards to `validate`.
7. **FINDING-05** — Decide on `p_postp`: delete or create a separate step story.
8. **FINDING-07** — Narrow `mt_base_header` to a key-only type.
9. **FINDING-09 through FINDING-13** — Address in the same transport (low effort).
