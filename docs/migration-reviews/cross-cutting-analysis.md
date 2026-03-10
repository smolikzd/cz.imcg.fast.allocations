# Cross-Cutting Analysis: Migration of ZFI_ALLOC_* to ZFI_PROCESS Framework

**Date:** 2026-03-05
**Repository:** `smolikzd/cz.imcg.fast.planner`
**Scope:** All four migrated step classes + process wiring in `zfi_alloc_proc_test`
**Reviewer:** OpenCode AI Agent

---

## 1. Repository Context

The source repository `cz.imcg.fast.ovysledovka` contains the full allocation process
package `ZFI_ALLOC_PROCESS` under `src/zfi_alloc_process/`. The migrated step classes are:

| Class | Step # | Substep mode | Status in process definition | Original program |
|-------|--------|-------------|------------------------------|-----------------|
| `ZCL_FI_ALLOC_STEP_INIT` | 0001 | SERIAL | Active | **None — net-new step** |
| `ZCL_FI_ALLOC_STEP_PHASE1` | 0002 | SERIAL | Active | `ZFI_ALLOC_PHASE_1` |
| `ZCL_FI_ALLOC_STEP_CORR_BCHE` | 0003 | SERIAL | Active | `ZFI_ALLOC_CORR_BCHE` |
| `ZCL_FI_ALLOC_STEP_PHASE2` | 0004 | **QUEUE** (`queue_count=5, queue_depth=3`) | Active | `ZFI_ALLOC_PHASE_2` |
| `ZCL_FI_ALLOC_STEP_PHASE3` | 0005 | SERIAL | **Commented out — not wired** | `ZFI_ALLOC_PHASE_3` |

The process is defined and tested via `ZFI_ALLOC_PROC_TEST`. There are also `_orig` class
variants (`ZCL_FI_ALLOC_STEP_PHASE2_ORIG`, `ZCL_FI_ALLOC_STEP_PHASE3_ORIG`) — these appear
to be snapshots of earlier iterations retained for reference.

---

## 2. Step Quality Gradient

Reading all five step classes from the repository reveals a clear quality gradient:

```
INIT (best)  >  PHASE2  >  PHASE1 ≈ CORR_BCHE  >  PHASE3 (worst)
```

### ZCL_FI_ALLOC_STEP_INIT — Reference quality, but also has an activation blocker
- `validate` actually checks state and returns proper error messages with `can_continue = abap_false`.
- `execute` is clean, concise, uses `RETURN` for early exit.
- No `COMMIT WORK`, no `MESSAGE TYPE 'E'`, no dead variables.
- **Missing `execute_substep`/`on_success`/`on_error` stubs (activation blocker — see §3.1, CC-01).**
- `execute` does not INSERT a state row on the normal (`force_start = false`) path — FINDING-02
  in the INIT review; see CC-15 below.
- **This is still the model the other steps should follow for `validate` and `execute` structure.**

### ZCL_FI_ALLOC_STEP_PHASE2 — Architecturally advanced, structurally broken
- Only step with a real `plan_substeps` + `execute_substep` implementation.
- Step 0004 is configured as `substep_mode = 'QUEUE'` with `queue_count=5, queue_depth=3`
  — this means bgRFC queued execution is **active in the current test setup**.
- `mo_log` is initialized in `execute`, not `init` — this is a live NULL-dereference dump
  waiting to happen in the queued substep path (FINDING-08 of the Phase 2 review).
- `on_success`/`on_error` are present but **empty stubs** — the 'F' state write and
  summary logging remain in `execute` (before substeps run), making all summary counters
  always zero and `phase_2_status` prematurely set to 'F'.
- `COMMIT WORK` after the 'R' state update.

### ZCL_FI_ALLOC_STEP_PHASE1 — Verbatim report conversion, same defects as CORR_BCHE
- The original program's `p_brand` / `p_hier1` filter parameters are hardcoded to `''`
  in the call to `get_base_keys`, identical to the Phase 3 regression (Phase 3 FINDING-05).
  This was not visible in the Phase 1 review because the original program source was
  reviewed without the repository.
- Two `COMMIT WORK` calls inside `execute` (one after state update, one after DELETEs).
- Missing `execute_substep`/`on_success`/`on_error` stubs (activation blocker).
- `validate` returns unconditional success.

### ZCL_FI_ALLOC_STEP_CORR_BCHE — Has additional unique defects not in other steps
- `COMMIT WORK` inside a `DO` loop (chunk processing pattern) — same LUW violation.
- `WRITE: / 'Update process completed.'` — a classical ABAP UI statement. This will
  cause a runtime error (`WRITE_ON_NONEXISTENT_DYNPRO`) when the step runs as a
  background process or in bgRFC context. This is a **hard runtime failure** not present
  in any other step.
- `SELECT * FROM zfi_alloc_bcitm` inside a loop over headers (`LOOP AT lt_headers`) —
  loads all items for every header into `lt_items`, but only checks `sy-subrc`. The
  `lt_items` table is reused across iterations without a `CLEAR` — items from a previous
  header iteration will bleed into the next if `sy-subrc <> 0`.

### ZCL_FI_ALLOC_STEP_PHASE3 — Cannot activate, functional regression, not wired
- Phase 3 is commented out of the process definition in `ZFI_ALLOC_PROC_TEST`. It is
  not currently part of the active pipeline.
- The class is identical to `ZCL_FI_ALLOC_STEP_PHASE3_ORIG` — no iteration has occurred.
- All defects from the Phase 3 migration review apply without modification.

---

## 3. Cross-Cutting Findings

These findings apply across multiple or all step classes and represent **systemic patterns**
rather than isolated bugs.

---

### CC-01 — Missing interface method stubs block activation for Init, Phase1, Phase3, CORR_BCHE (Critical)

**Affects:** `ZCL_FI_ALLOC_STEP_INIT`, `ZCL_FI_ALLOC_STEP_PHASE1`,
             `ZCL_FI_ALLOC_STEP_CORR_BCHE`, `ZCL_FI_ALLOC_STEP_PHASE3`
**Constitution principle:** II — SAP Standards Compliance

All four classes declare `INTERFACES zif_fi_process_step` but implement only a subset of
the nine required methods. The ABAP compiler will refuse to activate any of them:

| Class | Missing methods |
|-------|----------------|
| `ZCL_FI_ALLOC_STEP_INIT` | `execute_substep`, `on_success`, `on_error` |
| `ZCL_FI_ALLOC_STEP_PHASE1` | `execute_substep`, `on_success`, `on_error` |
| `ZCL_FI_ALLOC_STEP_CORR_BCHE` | `execute_substep`, `on_success`, `on_error` |
| `ZCL_FI_ALLOC_STEP_PHASE3` | `execute_substep`, `on_success`, `on_error` |

Phase 2 is the only correctly complete step.

**Note:** The earlier version of this document listed INIT as "Can activate: Yes". The
INIT review (FINDING-01) corrects this — INIT also lacks the three methods and cannot
be activated.

**Mitigation:** Add stub implementations for the three missing methods in each class.
The stubs for `execute_substep` should explicitly signal that substeps are not supported
(return `success = abap_false`, `can_continue = abap_false`) to prevent silent no-ops
if the framework ever calls them. `on_success` and `on_error` stubs can be empty.

---

### CC-02 — `COMMIT WORK` in every step body violates framework LUW contract (Critical)

**Affects:** `ZCL_FI_ALLOC_STEP_PHASE1` (×2), `ZCL_FI_ALLOC_STEP_PHASE2` (×1),
             `ZCL_FI_ALLOC_STEP_CORR_BCHE` (×1 per loop iteration), `ZCL_FI_ALLOC_STEP_PHASE3` (×1 per N items)

The framework owns the LUW. Every step except INIT contains at least one `COMMIT WORK`.
This is the single most widespread structural defect across the migration.

**Consequence map:**

| Step | Where | Consequence |
|------|-------|-------------|
| PHASE1 | After 'R' state write | State committed before DELETEs; partial state if DELETE fails |
| PHASE1 | After DELETEs | Unnecessary mid-execute commit; prevents atomic rollback |
| PHASE2 | After 'R' state write | State committed before substep planning; prevents rollback |
| CORR_BCHE | After each chunk | Partial header updates committed before full scan completes |
| PHASE3 | Every 5000 inner items | Partial `ZFI_ALLOC_ITEMS` rows committed; cannot be rolled back |

The primary consequence is not loss of atomicity with the framework status tables (the
framework does not provide a single bundled business-data + status commit) but destruction
of the **rollback window**: once a step commits mid-execute, partial data cannot be undone
by the framework's `rollback` call or by an ABAP runtime rollback on work process timeout.

See `luw-schema-analysis.md` for the full commit inventory, LUW ownership model, and
per-step impact assessment.

**Mitigation:** Remove all `COMMIT WORK` statements from step implementations.
Implement `rollback` methods in each step to compensate for the removal (clean up
partial data on framework-requested rollback). The framework commits naturally after
each step's status update.

---

### CC-03 — `MESSAGE TYPE 'E'` used for error control flow in class methods (Critical)

**Affects:** `ZCL_FI_ALLOC_STEP_PHASE1`, `ZCL_FI_ALLOC_STEP_PHASE2`,
             `ZCL_FI_ALLOC_STEP_PHASE3`

All three active processing steps use `MESSAGE e...` without `INTO` to abort on
validation failures. In class methods this raises a non-catchable system exception
(`CX_SY_NO_HANDLER`) rather than a clean application exit. The framework's `CATCH cx_root`
in the step executor will catch it, but the structured message text from the message class
is lost — only a generic exception text appears in the process monitor.

`ZCL_FI_ALLOC_STEP_INIT` correctly uses `rs_result-success = abap_false` + `RETURN` in
`validate`, which is the proper pattern for pre-execution guard logic.

**Mitigation:** Replace all `MESSAGE e...` abort patterns with
`RAISE EXCEPTION TYPE zcx_fi_process_error EXPORTING textid = ... value = ...`.
For loop-level errors (per-item logging), keep `MESSAGE ... INTO lv_dummy` + `lo_log->log_sy_msg( )`
but use `lv_dummy TYPE string` (see CC-06).

---

### CC-04 — `validate` returns unconditional success in three steps (Moderate)

**Affects:** `ZCL_FI_ALLOC_STEP_PHASE1`, `ZCL_FI_ALLOC_STEP_PHASE2`,
             `ZCL_FI_ALLOC_STEP_CORR_BCHE`, `ZCL_FI_ALLOC_STEP_PHASE3`

The `validate` method in all steps except INIT is a no-op:

```abap
METHOD zif_fi_process_step~validate.
  rs_result-success = abap_true.
  rs_result-message = 'Phase X validated'.
  rs_result-can_continue = abap_true.
ENDMETHOD.
```

The framework calls `validate` before `execute`. Steps that perform destructive DB
operations (`DELETE FROM zfi_alloc_bcitm`, `DELETE FROM zfi_alloc_bche`,
`DELETE FROM zfi_alloc_items`) should guard all key parameters here, before any data
is modified. An empty `validate` means a misconfigured process (missing `ALLOCATION_ID`,
`COMPANY_CODE`, etc.) will proceed directly to destructive DELETEs.

`ZCL_FI_ALLOC_STEP_INIT` demonstrates the correct approach: it reads `ZFI_ALLOC_STATE`
and returns `success = abap_false` with a descriptive message if the precondition fails.

**Mitigation:** Each step's `validate` should check all four mandatory parameters
(`ALLOCATION_ID`, `COMPANY_CODE`, `FISCAL_YEAR`, `FISCAL_PERIOD`) and return
`can_continue = abap_false` for any initial value. Steps that check allocation state
(PHASE1, PHASE2, PHASE3) should move that check from `execute` into `validate`.

---

### CC-05 — `rollback` is a no-op in every step (Moderate)

**Affects:** All four migrated steps (`CORR_BCHE`, `PHASE1`, `PHASE2`, `PHASE3`)

Every step has `" No rollback needed` as its `rollback` body. This is incorrect for
steps that issue destructive DML:

- PHASE1 DELETEs `ZFI_ALLOC_BCITM` and `ZFI_ALLOC_BCHE` and inserts new rows.
- PHASE2 DELETEs `ZFI_ALLOC_BCITM` and inserts new rows via substeps.
- PHASE3 DELETEs `ZFI_ALLOC_ITEMS` and inserts new rows.
- CORR_BCHE updates `ZFI_ALLOC_BCHE.no_items`.

When a step is interrupted (work process timeout, exception, manual cancellation),
the framework calls `rollback`. If `rollback` does nothing, the system is left with
partial data: `phase_X_status = 'R'` in `ZFI_ALLOC_STATE` with no way to re-run the
step without the `FORCE_START` override.

**Mitigation:** Implement `rollback` to at minimum reset `phase_X_status` to initial
in `ZFI_ALLOC_STATE`. For PHASE3 specifically, also delete any partially inserted rows
from `ZFI_ALLOC_ITEMS`. The Phase 3 review (FINDING-03) contains a concrete `rollback`
implementation template.

---

### CC-06 — `lv_dummy TYPE c` truncates all log message text to one character (Moderate)

**Affects:** `ZCL_FI_ALLOC_STEP_PHASE1`, `ZCL_FI_ALLOC_STEP_PHASE2`,
             `ZCL_FI_ALLOC_STEP_PHASE3`

All three steps declare `lv_dummy TYPE c` (implicit length 1) as the `INTO` target for
`MESSAGE ... INTO lv_dummy`. The ABAP `MESSAGE INTO` statement truncates the message text
to fit the target variable. With `TYPE c` (length 1), every message captured this way
produces a single-character string. In Phase 2, `rs_result-message = lv_dummy` after
an error catch propagates this single character as the step result message in the process
monitor.

**Mitigation:** Change `lv_dummy TYPE c` to `lv_dummy TYPE string` in all three steps.
A global search in the package for `lv_dummy TYPE c` will identify all occurrences.

---

### CC-07 — `ms_context` stored in `init` but never read (Minor)

**Affects:** All five step classes (including INIT)

Every step stores `ms_context = is_context` in `init` but only uses the individual
`mv_*` fields subsequently. `ms_context` is dead storage in all five classes.

**Mitigation:** Remove `ms_context` from the PRIVATE SECTION of all step classes.
This should be a single coordinated change across the package.

---

### CC-08 — Copy-paste header comment "Class ZCL_STEP_FETCH_DATA" in every step (Minor)

**Affects:** All five step classes

Every class file begins with:
```abap
*& Class ZCL_STEP_FETCH_DATA
*& Example step: Fetch data from database
```

This is the comment from the framework's reference example class (`ZCL_FI_STEP_FETCH_DATA`).
It is present in `ZCL_FI_ALLOC_STEP_INIT`, `PHASE1`, `PHASE2`, `PHASE3`, and `CORR_BCHE`
without modification. While cosmetic, it indicates that all five step classes were
created by direct copy from the same template without review.

**Mitigation:** Update each class's header comment to describe its actual purpose.

---

### CC-09 — `BRAND` / `HIER1` filter parameters dropped in Phase 1 and Phase 3 (Critical)

**Affects:** `ZCL_FI_ALLOC_STEP_PHASE1`, `ZCL_FI_ALLOC_STEP_PHASE3`

Both Phase 1 and Phase 3 originals accept optional `p_brand` and `p_hier1` filter
parameters that narrow the scope of processing. Both migrated step classes hardcode
empty literals in the corresponding framework method calls:

**Phase 1** (`get_base_keys`):
```abap
zcl_fi_allocations=>get_base_keys(
  EXPORTING iv_cc       = mv_company_code
            iv_fis_year = mv_fiscal_year
            iv_fis_per  = mv_fiscal_period
            iv_brand    = ''        " <-- should be mv_brand
            iv_hier1    = ''        " <-- should be mv_hier1
  IMPORTING et_result   = DATA(lt_base_keys) ).
```

The log even reflects the regression — the log message reads:
```abap
MESSAGE s000(zfi_alloc) WITH |Značka: { '' }|.    " brand: empty
MESSAGE s000(zfi_alloc) WITH |Hier1: { '' }|.     " hier1: empty
```

**Phase 3** (`get_unassigned`) — same pattern (see Phase 3 FINDING-05).

This was not identified in the Phase 1 review because the review was performed without
access to the repository. With the repository now available, this is confirmed as the
same regression in both steps.

The Phase 1 regression is arguably more severe than Phase 3's: Phase 1 builds
`ZFI_ALLOC_BCHE` headers. Running Phase 1 without brand/hier1 filtering creates headers
for ALL line items, not just the intended subset — subsequent phases then process a
superset of intended data.

**Mitigation:** Add `mv_brand TYPE wrf_brand_id` and `mv_hier1 TYPE zmatklh1` to the
PRIVATE SECTION of both `ZCL_FI_ALLOC_STEP_PHASE1` and `ZCL_FI_ALLOC_STEP_PHASE3`.
Read both from process parameters in `init`. Verify that `ZFI_ALLOC_PROCESS_PARAMS`
and/or `ZFI_ALLOC_ADD_PROCESS_PARAMS` contain the corresponding fields (check DDIC
tables in the repository).

---

### CC-10 — `ZCL_FI_ALLOC_STEP_PHASE3_ORIG` and `PHASE2_ORIG` are dead classes in active package (Moderate)

**Affects:** Package `ZFI_ALLOC_PROCESS`

The repository contains:
- `ZCL_FI_ALLOC_STEP_PHASE2_ORIG` — an earlier iteration of Phase 2 without substeps
- `ZCL_FI_ALLOC_STEP_PHASE3_ORIG` — identical content to `ZCL_FI_ALLOC_STEP_PHASE3`

These classes are registered in the SAP system as active objects (`.clas.xml` files
present, objects in the package). They are not referenced by any process definition,
program, or other class. Retaining "orig" snapshots as live SAP objects creates:
- Extended program check warnings.
- Namespace confusion (two classes with near-identical names doing the same thing).
- Transport overhead (both travel with the package on every transport).

Git provides version history — "orig" snapshots as live classes are unnecessary.

**Mitigation:** Delete both `*_ORIG` classes from the system and remove them from the
repository. If the snapshots are needed for reference, they belong in git history, not
in the active SAP package.

---

### CC-11 — Phase 3 not wired into process definition (Moderate)

**Affects:** `ZFI_ALLOC_PROC_TEST` process setup, overall pipeline

The `ZFI_ALLOC_PROC_TEST` program's `setup_process_data` method defines steps 0001–0004
and has Phase 3 (step 0005) **commented out**:

```abap
*( process_type  = 'ALLOCATIONS'
*  step_number   = '0005'
*  step_type     = 'PHASE3'
*  ...
*  class_name    = 'ZCL_FI_ALLOC_STEP_PHASE3'
*  substep_mode  = 'SERIAL' )
```

This means the currently executable pipeline stops at Phase 2 (item allocation key
preparation). Phase 3 (actual allocation item generation — the output product of
the entire process) is not yet functional.

Given that `ZCL_FI_ALLOC_STEP_PHASE3` cannot even be activated (CC-01), this is
consistent: Phase 3 is blocked at two independent levels.

**Mitigation:** Phase 3 should not be uncommented until CC-01, CC-02, CC-03, and
FINDING-05 (brand/hier1 regression) are resolved. Once fixed, Phase 3 also needs
substep architecture (Phase 3 FINDING-04) before being configured as `QUEUE` mode
like Phase 2.

---

### CC-12 — `ZCL_FI_ALLOC_STEP_CORR_BCHE` uses `WRITE:` — hard runtime failure in background (Critical)

**Affects:** `ZCL_FI_ALLOC_STEP_CORR_BCHE`

The `execute` method ends with:

```abap
WRITE: / 'Update process completed.'.
```

`WRITE` is a classical ABAP dialog statement. When called outside of a screen context
(background job, bgRFC, or framework process execution), it raises a runtime error
`WRITE_ON_NONEXISTENT_DYNPRO`. This step is step 0003 in the active pipeline and will
crash every process run before Phase 2 is even reached.

**Mitigation:** Replace with a log or `rs_result-message`:
```abap
rs_result-message = 'Update process completed.'.
```

Or use `zcl_sp_ballog` if a log object is initialized (note: CORR_BCHE currently has no
log initialization — see also the missing log pattern below).

---

### CC-13 — `ZCL_FI_ALLOC_STEP_CORR_BCHE` does not initialize `ZCL_SP_BALLOG` (Moderate)

**Affects:** `ZCL_FI_ALLOC_STEP_CORR_BCHE`

Unlike all other steps, CORR_BCHE creates no `zcl_sp_ballog` instance and logs nothing.
All operational output from this step is the `WRITE:` statement (which is broken — CC-12).
A step that processes potentially tens of thousands of headers and updates a flag per row
should produce a BAL log entry at minimum for the record count and any errors.

**Mitigation:** Add `lo_log` initialization identical to Phase 1/2/3 pattern, using
`zcl_fi_allocations=>c_bal_subobj_corr_bche` (define if not yet existing) and log the
total headers processed, and any update errors.

---

### CC-14 — `lt_items` not cleared between loop iterations in CORR_BCHE (Critical)

**Affects:** `ZCL_FI_ALLOC_STEP_CORR_BCHE`

In the outer `LOOP AT lt_headers` inside the chunk processing `DO` loop:

```abap
LOOP AT lt_headers INTO DATA(ls_header).
  SELECT *
    FROM zfi_alloc_bcitm
    INTO TABLE lt_items          " <-- no CLEAR before SELECT
    WHERE key_id = ls_header-key_id.

  IF sy-subrc <> 0.              " <-- only checks sy-subrc, not lt_items content
    UPDATE zfi_alloc_bche SET no_items = 'Y' WHERE key_id = ls_header-key_id.
  ELSE.
    UPDATE zfi_alloc_bche SET no_items = 'N' WHERE key_id = ls_header-key_id.
  ENDIF.
ENDLOOP.
```

`lt_items` is declared outside the loop as `DATA lt_items TYPE TABLE OF zfi_alloc_bcitm`.
If header A has items (SELECT succeeds, `lt_items` populated), and header B has no items
(`sy-subrc <> 0`), the `IF sy-subrc <> 0` branch correctly sets `no_items = 'Y'` for B.
This part actually works because only `sy-subrc` is checked.

However, the `lt_items` table accumulates rows across iterations. While this does not
affect the `no_items` flag decision (which uses `sy-subrc`), it causes unnecessary memory
growth — for a large header set, `lt_items` will eventually hold all items from all
previously processed headers simultaneously. For a company code with millions of base
items, this can exhaust application server memory.

**Mitigation:** Add `CLEAR lt_items.` before the SELECT, or use `INTO TABLE @DATA(lt_items)`
(inline declaration) which creates a fresh table per iteration.

---

### CC-15 — INIT step does not insert state row on normal path (Moderate)

**Affects:** `ZCL_FI_ALLOC_STEP_INIT`

When `FORCE_START = abap_false`, the `execute` method skips the `INSERT INTO zfi_alloc_state`
block entirely and returns `success = abap_true` without writing any row. This means a
first-time initialization run with `FORCE_START = false` results in no state row existing.
Subsequent steps (PHASE1, PHASE2, PHASE3) attempt to UPDATE `phase_X_status` in that row;
the UPDATE silently finds zero matches (`sy-subrc <> 0`), and each step reports an error.

See FINDING-02 in `zfi_alloc_init-migration-review.md` for a corrected `execute` implementation.

**Mitigation:** Restructure `execute` to always INSERT after conditionally deleting:
```abap
IF mv_force_start = abap_true.
  DELETE FROM zfi_alloc_state WHERE ...
ENDIF.
INSERT INTO zfi_alloc_state VALUES @( VALUE zfi_alloc_state( ... ) ).
```

---

### CC-16 — Framework bug: `init` never called in `ZFI_BGRFC_EXEC_SUBSTEP` (Critical)

**Affects:** `ZCL_FI_ALLOC_STEP_PHASE2` (the only QUEUE-mode step)  
**Locus:** `src/zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap` (framework, not step code)

The bgRFC function module `ZFI_BGRFC_EXEC_SUBSTEP` instantiates the step class and calls
`execute_substep` directly without first calling `lo_step->init( ls_context )`. All instance
variables populated in `init` (`mv_allocation_id`, `mv_company_code`, `mv_fiscal_year`,
`mv_fiscal_period`, `mv_force_start`) are therefore initial (`''`/`0`) when `execute_substep`
runs. `mo_log` is also NULL.

**Consequence for PHASE2:**
- Any `mo_log->…` call in `execute_substep` raises `CX_SY_REF_IS_INITIAL` → bgRFC unit dump.
- Any SELECT/DML using the key variables operates on initial values — either full table
  scans or zero-row results, depending on the WHERE clause.
- The bgRFC infrastructure retries the failed unit according to queue configuration, but the
  retry will fail identically because the root cause is structural, not transient.

**This makes the QUEUE substep path of `ZCL_FI_ALLOC_STEP_PHASE2` completely non-functional.**

Note: This finding compounds the Phase 2 FINDING-08 (`mo_log` initialized in `execute` not
`init`). Both must be fixed together — moving `mo_log` init to `init` in the step, and
calling `init` from the framework FM. Either fix alone is insufficient.

See `luw-schema-analysis.md` §6 for the detailed impact analysis and required fix.

**Mitigation (framework):** Add `lo_step->init( ls_context )` in `ZFI_BGRFC_EXEC_SUBSTEP`
after step instantiation, where `ls_context` is built from the retrieved instance object.

**Mitigation (step):** Move `mo_log` initialization from `execute` to `init` in
`ZCL_FI_ALLOC_STEP_PHASE2` so it is available in any execution path.

---

## 4. Repository-Level Findings vs. Individual Review Corrections

Reading the actual repository sources reveals two corrections to previously issued reviews:

### Correction to Phase 1 review
The Phase 1 review (performed without repository access) did not flag the `BRAND`/`HIER1`
filter regression because the original program source was reviewed in isolation. The
repository confirms that `ZCL_FI_ALLOC_STEP_PHASE1` also hardcodes `iv_brand = ''` and
`iv_hier1 = ''` in `get_base_keys`. This is the same severity as Phase 3 FINDING-05
(Critical — functional regression). The Phase 1 review should be updated to add this
as a Critical finding.

### Correction to Phase 2 review
The Phase 2 review noted FINDING-03 as a critical structural defect (execute writes 'F'
before substeps run, logs zero counters). Reviewing the actual repository class confirms
this is present in the committed code and has not been partially addressed. The
`on_success`/`on_error` stubs are present (which was correctly noted) but empty. The
'F' state write and summary logging remain in `execute` as described.

---

## 5. Activation Readiness by Step

| Step | Can activate? | Blocking reason |
|------|--------------|-----------------|
| `ZCL_FI_ALLOC_STEP_INIT` | **No** | Missing `execute_substep`, `on_success`, `on_error` |
| `ZCL_FI_ALLOC_STEP_PHASE1` | **No** | Missing `execute_substep`, `on_success`, `on_error` |
| `ZCL_FI_ALLOC_STEP_CORR_BCHE` | **No** | Missing `execute_substep`, `on_success`, `on_error` |
| `ZCL_FI_ALLOC_STEP_PHASE2` | Yes | — (complete, but has runtime defects) |
| `ZCL_FI_ALLOC_STEP_PHASE3` | **No** | Missing `execute_substep`, `on_success`, `on_error` |

---

## 6. End-to-End Pipeline Risk Assessment

Based on the current state of the committed code, executing the full `ALLOCATIONS` process
(steps 0001–0004) against a production dataset would encounter the following failures in
sequence:

1. **Step 0001 (INIT):** Cannot be activated (CC-01 — missing `execute_substep`, `on_success`,
   `on_error`). If those stubs are added, INIT will run but will not INSERT a state row on
   the normal (`force_start = false`) path (CC-15), causing all subsequent steps to fail
   when they attempt to UPDATE a non-existent state row.

2. **Step 0002 (PHASE1):** Cannot be activated (CC-01).
   - `COMMIT WORK` ×2 violates LUW contract.
   - Brand/hier1 filter silently ignored — processes ALL line items regardless.
   - `validate` does not guard parameters.
   - `phase_1_status` state machine diverges from framework lifecycle.
   - Does not crash, but produces incorrect scope.

3. **Step 0002 (PHASE1):** Cannot be activated (CC-01). If activation blockers are fixed:
   - `COMMIT WORK` ×2 violates LUW contract.
   - Brand/hier1 filter silently ignored — processes ALL line items regardless.
   - `validate` does not guard parameters.
   - `phase_1_status` state machine diverges from framework lifecycle.
   - Does not crash, but produces incorrect scope.

4. **Step 0003 (CORR_BCHE):** Cannot be activated (CC-01). If activation blockers are fixed:
   **Will crash** with `WRITE_ON_NONEXISTENT_DYNPRO` on the `WRITE:` statement.

5. **Step 0004 (PHASE2):** Not reached due to upstream failures. If reached:
   - `mo_log` will be NULL in `execute_substep` under `QUEUE` mode → dump.
   - Summary counters log as zero, 'F' status written prematurely.

6. **Step 0005 (PHASE3):** Commented out of definition. Cannot be activated.

**Net result: No step can currently be activated. The only activatable class is
`ZCL_FI_ALLOC_STEP_PHASE2`. The pipeline cannot run at all in the current state.**

---

## 7. Prioritized Remediation Roadmap

### Sprint 1 — Unblock pipeline execution (must fix before any test run)

| # | Action | Effort | Affects |
|---|--------|--------|---------|
| R-01 | Remove `WRITE:` from CORR_BCHE | 5 min | CC-12 |
| R-02 | Add `execute_substep`/`on_success`/`on_error` stubs to **INIT**, PHASE1, CORR_BCHE, PHASE3 | 45 min | CC-01 |
| R-03 | Fix `BRAND`/`HIER1` regression in PHASE1 and PHASE3 | 2h | CC-09 |
| R-04 | Replace `MESSAGE TYPE 'E'` with `RAISE EXCEPTION TYPE zcx_fi_process_error` in PHASE1, PHASE2, PHASE3 | 2h | CC-03 |
| R-05 | Remove all `COMMIT WORK` from step bodies; implement `rollback` stubs | 2h | CC-02 |
| R-06 | Fix INIT `execute` to always INSERT state row (restructure if/insert block) | 30 min | CC-15 |
| R-07 | Add `lo_step->init( ls_context )` call in `ZFI_BGRFC_EXEC_SUBSTEP` (framework fix) | 1h | CC-16 |

### Sprint 2 — Structural correctness

| # | Action | Effort | Affects |
|---|--------|--------|---------|
| R-08 | Implement real `validate` guards in PHASE1, PHASE2, PHASE3, CORR_BCHE | 2h | CC-04 |
| R-09 | Move `mo_log` init from `execute` to `init` in PHASE2 | 30 min | Phase 2 FINDING-08, CC-16 |
| R-10 | Move 'F' state write and summary logging from `execute` to `on_success`/`on_error` in PHASE2 | 2h | Phase 2 FINDING-03 |
| R-11 | Implement real `rollback` methods (state reset + partial data cleanup) | 3h | CC-05 |
| R-12 | Fix `lt_items` memory accumulation in CORR_BCHE | 15 min | CC-14 |
| R-13 | Add BAL log to CORR_BCHE | 1h | CC-13 |
| R-14 | Wire Phase 3 into process definition (after R-01 through R-11 complete) | 30 min | CC-11 |

### Sprint 3 — Code quality and architecture

| # | Action | Effort | Affects |
|---|--------|--------|---------|
| R-15 | Design Phase 3 substep architecture (unassigned item → substep) | 4h | Phase 3 FINDING-04 |
| R-16 | Change `lv_dummy TYPE c` to `TYPE string` in PHASE1, PHASE2, PHASE3 | 30 min | CC-06 |
| R-17 | Remove `ms_context` from all step classes | 30 min | CC-07 |
| R-18 | Delete `*_ORIG` classes from system and repository | 30 min | CC-10 |
| R-19 | Fix Phase 2 FINDING-04 (CATCH block fall-through) | 1h | Phase 2 |
| R-20 | Update all class header comments | 15 min | CC-08 |

---

## 8. Summary Table: Finding Coverage Across Steps

| Finding | INIT | PHASE1 | CORR_BCHE | PHASE2 | PHASE3 |
|---------|------|--------|-----------|--------|--------|
| Missing interface methods (CC-01) | **Critical** | **Critical** | **Critical** | — | **Critical** |
| COMMIT WORK (CC-02) | — | **Critical** | **Critical** | **Critical** | **Critical** |
| MESSAGE TYPE 'E' (CC-03) | — | **Critical** | — | **Critical** | **Critical** |
| validate no-op (CC-04) | — | Moderate | Moderate | Moderate | Moderate |
| rollback no-op (CC-05) | — | Moderate | Moderate | Moderate | Moderate |
| lv_dummy TYPE c (CC-06) | — | Minor | — | Minor | Minor |
| ms_context dead field (CC-07) | Minor | Minor | Minor | Minor | Minor |
| Copy-paste header comment (CC-08) | Minor | Minor | Minor | Minor | Minor |
| BRAND/HIER1 regression (CC-09) | — | **Critical** | — | — | **Critical** |
| _ORIG dead classes (CC-10) | — | — | — | Moderate | Moderate |
| Phase 3 not wired (CC-11) | — | — | — | — | Moderate |
| WRITE: crash (CC-12) | — | — | **Critical** | — | — |
| No BAL log (CC-13) | — | — | Moderate | — | — |
| lt_items not cleared (CC-14) | — | — | **Critical** | — | — |
| INIT no INSERT on normal path (CC-15) | **Moderate** | — | — | — | — |
| Framework: init not called in bgRFC FM (CC-16) | — | — | — | **Critical** | — |
