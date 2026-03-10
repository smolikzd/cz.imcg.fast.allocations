---
title: 'ZFI_ALLOC_PROCESS — Migration Remediation'
slug: 'epics-alloc-remediation'
created: '2026-03-09'
status: 'ready-for-dev'
stepsCompleted: []
inputDocuments:
  - 'docs/migration-reviews/cross-cutting-analysis.md'
  - 'docs/migration-reviews/luw-schema-analysis.md'
  - 'docs/migration-reviews/zfi_alloc_init-migration-review.md'
  - 'docs/migration-reviews/zfi_alloc_phase_1-migration-review.md'
  - 'docs/migration-reviews/zfi_alloc_phase_2-migration-review.md'
  - 'docs/migration-reviews/zfi_alloc_phase_3-migration-review.md'
  - 'docs/migration-reviews/zfi_alloc_corr_bche-migration-review.md'
tech_stack:
  - 'ABAP 7.58'
  - 'SAP DDIC'
  - 'ZFI_PROCESS framework'
  - 'bgRFC / qRFC'
  - 'abapGit XML'
---

# ZFI_ALLOC_PROCESS — Migration Remediation Epic Breakdown

## Overview

This document organises all findings from the ZFI_ALLOC_* migration reviews into
implementable stories. The source findings are the Cross-Cutting Analysis (CC-01 to CC-16)
and the LUW Schema Analysis (LUW-01 to LUW-06).

**Important context carried into this plan:**

- The developer has already removed all `COMMIT WORK` statements from step bodies.
  Stories that address CC-02 therefore focus only on the remaining work: implementing
  `rollback` methods (CC-05). The `COMMIT WORK` removal itself is considered done.
- Substep execution strategy per step (confirmed with project lead):
  - **INIT, PHASE1, CORR_BCHE** — serial by design; `execute_substep`, `on_success`,
    `on_error` are empty stubs only (raise non-support exception or do nothing).
  - **PHASE2** — keep QUEUE mode; real `execute_substep` / `on_success` / `on_error`
    implementation; fix the bgRFC `init` omission bug.
  - **PHASE3** — serial for now; stubs only; no substep architecture in this sprint.

---

## Sprint Progress

### Sprint 1 — Unblock the pipeline (COMPLETE)
**Status**: ✅ Complete | **Total effort**: 3.75h / 3.75h

All 6 stories completed:
- ✅ Story 1.1 (45min) — Serial stubs | commit: 4a8c53f
- ✅ Story 1.2 (5min) — Remove WRITE: | commit: 81e3d67
- ✅ Story 2.2 (1.5h) — bgRFC init fix | commit: 5a9b7c4
- ✅ Story 4.2 (30min) — INIT state INSERT | commit: c3f9e8a
- ✅ Story 4.3 (15min) — lt_items clear | commit: 9d2e6f1
- ✅ Story 5.3 (30min) — lv_dummy TYPE string | commit: b7a4d5e

### Sprint 2 — Structural correctness (COMPLETE)
**Status**: ✅ Complete | **Total effort**: 11h / 11h

All 7 stories completed (Story 4.1 cancelled):
- ✅ Story 2.1 (3h) — Rollback methods | commit: e651daa
- ✅ Story 3.1 (30min) — mo_log to init | commit: 16f0540
- ✅ Story 3.2 (2h) — on_success/on_error | commit: 8bf9cc1
- ✅ Story 3.3 (1h) — CATCH fix | commit: 4d103ce
- ❌ Story 4.1 — BRAND/HIER1 filter | CANCELLED (removed from sprint)
- ✅ Story 5.1 (2h) — MESSAGE→RAISE | commit: e6f7c14
- ✅ Story 5.2 (1.5h) — validate guards | commit: 3f8d430
- ✅ Story 5.4 (1h) — CORR_BCHE BAL log | commit: a430daf

### Sprint 3 — Wire Phase 3 + housekeeping (PENDING)
**Status**: 🔜 Pending | **Total effort**: 0h / 1.75h

4 stories remaining:
- ⏳ Story 6.1 (30min) — Wire PHASE3
- ⏳ Story 7.1 (30min) — Remove ms_context
- ⏳ Story 7.2 (15min) — Header comments
- ⏳ Story 7.3 (30min) — Delete *_ORIG classes

---

## Requirements Inventory

### Functional Requirements

| ID | Finding | Source | Severity |
|----|---------|--------|---------|
| FR-01 | All step classes must be activatable (interface complete) | CC-01 | Critical |
| FR-02 | Step `rollback` methods must undo partial data | CC-02 / CC-05 | Critical |
| FR-03 | Error flow must use `ZCX_FI_PROCESS_ERROR`, not `MESSAGE TYPE 'E'` | CC-03 | Critical |
| FR-04 | `validate` must guard all mandatory parameters before any DML | CC-04 | Moderate |
| FR-05 | BRAND / HIER1 filter parameters must not be silently dropped | CC-09 | Critical |
| FR-06 | `WRITE:` statement in CORR_BCHE must be removed | CC-12 | Critical |
| FR-07 | `init` must be called in `ZFI_BGRFC_EXEC_SUBSTEP` before `execute_substep` | CC-16 / LUW-02 | Critical |
| FR-08 | `mo_log` must be initialised in `init`, not in `execute`, in PHASE2 | Phase 2 FINDING-08 | Critical |
| FR-09 | PHASE2 final state write and summary logging must move to `on_success`/`on_error` | Phase 2 FINDING-03 | High |
| FR-10 | INIT `execute` must always INSERT state row, regardless of `FORCE_START` | CC-15 | Moderate |
| FR-11 | `lv_dummy TYPE c` must be replaced with `TYPE string` for message capture | CC-06 | Moderate |
| FR-12 | `ms_context` dead field must be removed from all step classes | CC-07 | Minor |
| FR-13 | `*_ORIG` dead classes must be deleted from system and repository | CC-10 | Moderate |
| FR-14 | Phase 3 must be wired into the process definition (after other fixes) | CC-11 | Moderate |
| FR-15 | CORR_BCHE must initialise `ZCL_SP_BALLOG` and produce a log | CC-13 | Moderate |
| FR-16 | `lt_items` must be cleared between loop iterations in CORR_BCHE | CC-14 | Critical |
| FR-17 | All class header comments must reflect actual class purpose | CC-08 | Minor |
| FR-18 | PHASE2 CATCH block fall-through must be fixed | Phase 2 FINDING-04 | Moderate |
| FR-19 | `cleanup_old_instances` uncommitted DELETEs must be documented or fixed | LUW-03 | Low |

### Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-01 | All changes must comply with constitution (DDIC-First, SAP Standards, Error Handling) |
| NFR-02 | All new/changed methods must have ABAP-Doc comments |
| NFR-03 | Line length ≤ 120 characters (abaplint enforced) |
| NFR-04 | No local TYPE definitions introduced |
| NFR-05 | mcp-sap-docs consulted before implementing any new pattern |

---

## Epic List

| # | Epic | Findings addressed | Priority |
|---|------|--------------------|---------|
| E1 | Unblock pipeline activation | CC-01, CC-12 | 1 — Must first |
| E2 | Fix LUW and rollback contract | CC-02 (done), CC-05, CC-16, LUW-02 | 1 — Must first |
| E3 | Functional correctness — PHASE2 substep path | Phase 2 FINDING-03, 04, 08 | 2 — High |
| E4 | Functional correctness — data integrity | CC-09, CC-14, CC-15 | 2 — High |
| E5 | Error handling and observability | CC-03, CC-04, CC-13 | 2 — High |
| E6 | Wire Phase 3 into pipeline | CC-11, FR-14 | 3 — After E1–E5 |
| E7 | Code quality and housekeeping | CC-06, CC-07, CC-08, CC-10, LUW-03 | 3 — Low risk |

---

## Epic 1: Unblock Pipeline Activation

**Goal:** Make every step class activatable in ABAP so the pipeline can be transported and
run for the first time. Two independent blockers exist: missing interface method implementations
and a hard runtime crash from a `WRITE:` statement.

---

### Story 1.1: Add serial-mode interface stubs to INIT, PHASE1, CORR_BCHE, PHASE3

As a pipeline operator,
I want all step classes to be activatable in ABAP,
so that the process can be transported and executed without activation errors.

**Background:** `ZIF_FI_PROCESS_STEP` requires nine methods. INIT, PHASE1, CORR_BCHE, and
PHASE3 are all serial-mode steps — they have no parallel execution logic. These three missing
methods must be implemented as explicit serial-safe stubs that document the intent.

**Acceptance Criteria:**

**Given** each of INIT, PHASE1, CORR_BCHE, PHASE3 declares `INTERFACES zif_fi_process_step`
**When** the class is activated in ABAP
**Then** activation succeeds without errors

**Given** `execute_substep` is called on any of these classes
**When** the call is made (e.g., misconfiguration in future)
**Then** it raises `ZCX_FI_PROCESS_ERROR` with a message indicating substeps are not
supported for this step, and `can_continue = abap_false`

**Given** `on_success` or `on_error` is called on any of these classes
**When** the call is made
**Then** the method returns silently (empty implementation — no error, no side effects)

**Tasks / Subtasks:**

- [ ] Task 1.1.1: Add stubs to `ZCL_FI_ALLOC_STEP_INIT`
  - [ ] Add `METHOD zif_fi_process_step~execute_substep` — raise `ZCX_FI_PROCESS_ERROR` with
    message `'Serial step: execute_substep not supported'`
  - [ ] Add `METHOD zif_fi_process_step~on_success` — empty
  - [ ] Add `METHOD zif_fi_process_step~on_error` — empty
- [ ] Task 1.1.2: Add stubs to `ZCL_FI_ALLOC_STEP_PHASE1` — same as 1.1.1
- [ ] Task 1.1.3: Add stubs to `ZCL_FI_ALLOC_STEP_CORR_BCHE` — same as 1.1.1
- [ ] Task 1.1.4: Add stubs to `ZCL_FI_ALLOC_STEP_PHASE3` — same as 1.1.1

**Dev Notes:**
- Constitution Principle V: raise `ZCX_FI_PROCESS_ERROR`, not a generic exception.
- Do not raise in `on_success`/`on_error` — the framework may call them unconditionally.
- PHASE2 already has real `execute_substep`, `on_success`, `on_error` — do not touch.
- Reference: CC-01 in `cross-cutting-analysis.md`.

---

### Story 1.2: Remove `WRITE:` statement from CORR_BCHE

As a pipeline operator,
I want CORR_BCHE to execute cleanly in background mode,
so that step 0003 does not crash the pipeline with `WRITE_ON_NONEXISTENT_DYNPRO`.

**Acceptance Criteria:**

**Given** `ZCL_FI_ALLOC_STEP_CORR_BCHE` is executing in a background context
**When** `execute` completes normally
**Then** no `WRITE_ON_NONEXISTENT_DYNPRO` runtime error is raised

**Given** the `WRITE:` statement is replaced
**When** execution succeeds
**Then** `rs_result-message` contains a meaningful completion message (e.g., total headers
processed)

**Tasks / Subtasks:**

- [ ] Task 1.2.1: Replace `WRITE: / 'Update process completed.'` with
  `rs_result-message = |CORR_BCHE completed. { <lv_count> } headers processed.|`
  (use the header count already available from the processing loop)

**Dev Notes:**
- If a log object is not yet initialised at this stage (CC-13 is addressed in E5), populate
  only `rs_result-message` for now.
- Reference: CC-12 in `cross-cutting-analysis.md`.

---

## Epic 2: Fix LUW and Rollback Contract

**Goal:** Ensure the rollback window is intact for all steps, fix the framework bug that
prevents `init` from being called in the bgRFC substep execution path, and implement
meaningful `rollback` methods in all steps that perform destructive DML.

**Note:** `COMMIT WORK` has already been removed from all step bodies by the developer.
This epic focuses on the remaining work: rollback implementation and the framework-level
`init` call bug.

---

### Story 2.1: Implement `rollback` methods in PHASE1, PHASE2, CORR_BCHE, PHASE3

As a process framework,
I want each step to clean up its partial work on rollback,
so that a failed or cancelled process does not leave the system in an unrecoverable partial
state.

**Acceptance Criteria:**

**Given** a step fails or is cancelled after writing `phase_X_status = 'R'` to `ZFI_ALLOC_STATE`
**When** the framework calls `rollback`
**Then** `phase_X_status` is reset to its initial value (`' '`) in `ZFI_ALLOC_STATE`

**Given** PHASE1 has partially deleted and re-inserted rows in `ZFI_ALLOC_BCHE` / `ZFI_ALLOC_BCITM`
**When** `rollback` is called
**Then** `phase_1_status` is reset; any partial INSERT rows written before the failure are
deleted (clean slate for retry)

**Given** PHASE3 has partially inserted rows into `ZFI_ALLOC_ITEMS`
**When** `rollback` is called
**Then** `phase_3_status` is reset; all `ZFI_ALLOC_ITEMS` rows for the current
`mv_allocation_id` / `mv_company_code` / `mv_fiscal_year` / `mv_fiscal_period` are deleted

**Given** any step's `rollback` method completes
**When** the operator retries the process (with `FORCE_START = false` where applicable)
**Then** the step re-executes from a clean initial state

**Tasks / Subtasks:**

- [ ] Task 2.1.1: Implement `rollback` in `ZCL_FI_ALLOC_STEP_PHASE1`
  - [ ] `UPDATE zfi_alloc_state SET phase_1_status = ' '` WHERE allocation key matches
  - [ ] `DELETE FROM zfi_alloc_bche` WHERE allocation key matches (undo partial headers)
  - [ ] `DELETE FROM zfi_alloc_bcitm` WHERE allocation key matches (undo partial items)
- [ ] Task 2.1.2: Implement `rollback` in `ZCL_FI_ALLOC_STEP_PHASE2`
  - [ ] `UPDATE zfi_alloc_state SET phase_2_status = ' '` WHERE allocation key matches
  - [ ] `DELETE FROM zfi_alloc_bcitm` WHERE allocation key matches (undo any substep inserts)
- [ ] Task 2.1.3: Implement `rollback` in `ZCL_FI_ALLOC_STEP_CORR_BCHE`
  - [ ] No data ownership beyond `ZFI_ALLOC_BCHE.no_items` updates — reset is not reliably
    possible without a snapshot. Set `rollback` to log a warning that manual re-run is
    needed and return cleanly. Document this limitation.
- [ ] Task 2.1.4: Implement `rollback` in `ZCL_FI_ALLOC_STEP_PHASE3`
  - [ ] `UPDATE zfi_alloc_state SET phase_3_status = ' '` WHERE allocation key matches
  - [ ] `DELETE FROM zfi_alloc_items` WHERE allocation key matches (undo partial inserts)
- [ ] Task 2.1.5: Verify `ZCL_FI_ALLOC_STEP_INIT` — `rollback` body: `DELETE FROM zfi_alloc_state`
  WHERE allocation key matches (full state row removal on init rollback)

**Dev Notes:**
- All `rollback` DML must use the instance variables populated in `init` (`mv_allocation_id`,
  `mv_company_code`, etc.) — these are available because `rollback` is called in the normal
  execution path where `init` has already run.
- `rollback` must NOT call `COMMIT WORK` — the framework manages the LUW.
- Constitution Principle V: if the key variables are initial (unexpected state), raise
  `ZCX_FI_PROCESS_ERROR` with context rather than silently doing nothing.
- Reference: CC-05, CC-02, LUW schema analysis §7.3.

---

### Story 2.2: Fix framework bug — call `init` before `execute_substep` in bgRFC FM

As a PHASE2 substep executor,
I want `init` to be called before `execute_substep` in the bgRFC function module,
so that all instance variables and the log object are populated when a substep runs
in queued mode.

**Acceptance Criteria:**

**Given** a PHASE2 substep is queued and executed by the bgRFC infrastructure
**When** `ZFI_BGRFC_EXEC_SUBSTEP` runs
**Then** `lo_step->init( ls_context )` is called before `lo_step->execute_substep( ... )`

**Given** `init` is called with a properly constructed context
**When** `execute_substep` runs
**Then** `mv_allocation_id`, `mv_company_code`, `mv_fiscal_year`, `mv_fiscal_period`
are populated with the values from the process instance parameters

**Given** a substep executes successfully
**When** it completes
**Then** no `CX_SY_REF_IS_INITIAL` dump occurs from `mo_log` access (after Story 2.3 is also applied)

**Tasks / Subtasks:**

- [ ] Task 2.2.1: In `src/zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap`:
  - [ ] Retrieve or construct the `ZCL_FI_PROCESS_INSTANCE` object for `lv_instance_id`
    (use factory method consistent with how the framework elsewhere retrieves instances)
  - [ ] Build `ls_context TYPE zif_fi_process_step=>ts_context` with `io_process_instance`
    set to the retrieved instance object
  - [ ] Add `lo_step->init( ls_context )` call after `CREATE OBJECT lo_step TYPE (...)` and
    before `lo_step->execute_substep( lv_substep_number )`
  - [ ] Wrap the `init` call in `TRY-CATCH zcx_fi_process_error` — on failure, set substep
    status to FAILED and return

**Dev Notes:**
- Constitution Principle III: consult mcp-sap-docs for `ZCL_FI_PROCESS_INSTANCE` factory
  pattern before adding instance retrieval.
- Constitution Principle V: `init` failure must not silently swallow the error; log and
  set substep FAILED.
- The bgRFC FM must NOT issue `COMMIT WORK` — the qRFC infrastructure commits the LUW.
- This fix is required for Story 3.1 (PHASE2 `mo_log` move) to have any effect in the
  bgRFC path.
- Reference: CC-16, LUW-02, `luw-schema-analysis.md` §6.

---

## Epic 3: Functional Correctness — PHASE2 Substep Path

**Goal:** Fix the three structural defects in PHASE2's substep lifecycle so that queued
parallel execution produces correct results and correct final state.

---

### Story 3.1: Move `mo_log` initialisation from `execute` to `init` in PHASE2

As a PHASE2 substep,
I want the application log to be initialised in `init`,
so that it is available in all execution paths including bgRFC queued substep execution.

**Acceptance Criteria:**

**Given** a PHASE2 step object is created and `init` is called
**When** `init` completes
**Then** `mo_log` is a valid, non-null `ZCL_SP_BALLOG` instance

**Given** `execute_substep` is called (in bgRFC context, after `init` is called per Story 2.2)
**When** any log write operation is attempted
**Then** no `CX_SY_REF_IS_INITIAL` exception is raised

**Given** `execute` is called (non-substep serial fallback path)
**When** log write operations occur in `execute`
**Then** the same already-initialised `mo_log` is used without re-initialisation

**Tasks / Subtasks:**

- [ ] Task 3.1.1: Move `mo_log = NEW zcl_sp_ballog( ... )` from `execute` to `init` in
  `ZCL_FI_ALLOC_STEP_PHASE2`
- [ ] Task 3.1.2: Remove the `mo_log` initialisation line from `execute` (guard: check it is
  not initialised again there)
- [ ] Task 3.1.3: Verify all `mo_log->` calls in `execute` and `execute_substep` now reference
  the instance initialised in `init`

**Dev Notes:**
- This story and Story 2.2 are independent but must both be done for the bgRFC path to work
  correctly. Recommend delivering Story 2.2 first, then this story.
- Reference: Phase 2 FINDING-08, CC-16, `luw-schema-analysis.md` §6.

---

### Story 3.2: Move final state write and summary logging to `on_success` / `on_error` in PHASE2

As a PHASE2 process monitor user,
I want the final `phase_2_status` and summary counters to reflect the actual substep results,
so that the process monitor shows accurate completion data rather than always showing zero
counters and premature 'F' status.

**Acceptance Criteria:**

**Given** all PHASE2 substeps complete successfully
**When** the framework calls `on_success`
**Then** `ZFI_ALLOC_STATE.phase_2_status` is set to `'F'` (finished/complete)
**And** the BAL log contains the correct total of items processed across all substeps

**Given** one or more PHASE2 substeps fail
**When** the framework calls `on_error`
**Then** `ZFI_ALLOC_STATE.phase_2_status` reflects a failed/partial state
**And** the BAL log records the failure context

**Given** the `execute` method runs (before substeps are dispatched)
**When** `execute` completes
**Then** `ZFI_ALLOC_STATE.phase_2_status` is set to `'R'` (running) only
**And** no final state ('F') is written prematurely

**Tasks / Subtasks:**

- [ ] Task 3.2.1: Remove the `phase_2_status = 'F'` UPDATE and summary log write from `execute`
- [ ] Task 3.2.2: Implement `on_success` in `ZCL_FI_ALLOC_STEP_PHASE2`:
  - [ ] Aggregate results across substep records (query `ZFI_PROC_STEP` substep rows to sum
    counters, or use a result accumulation pattern consistent with the framework)
  - [ ] `UPDATE zfi_alloc_state SET phase_2_status = 'F'` WHERE allocation key matches
  - [ ] Write summary to `mo_log`
- [ ] Task 3.2.3: Implement `on_error` in `ZCL_FI_ALLOC_STEP_PHASE2`:
  - [ ] `UPDATE zfi_alloc_state SET phase_2_status = 'E'` (or appropriate error code)
  - [ ] Write failure summary to `mo_log`
- [ ] Task 3.2.4: Verify `execute` now only writes 'R' and dispatches substeps; no final state
  writes remain in `execute`

**Dev Notes:**
- `on_success` and `on_error` must NOT issue `COMMIT WORK` — the framework manages the LUW.
- If `ZFI_ALLOC_STATE` uses a fixed-value domain for `phase_2_status`, verify the domain
  values include 'E' for error or choose an appropriate existing value.
- Constitution Principle III: consult mcp-sap-docs on substep result aggregation patterns
  if none exists in the current framework.
- Reference: Phase 2 FINDING-03, `cross-cutting-analysis.md` §2.

---

### Story 3.3: Fix PHASE2 CATCH block fall-through

As a PHASE2 process monitor user,
I want exceptions during substep execution to be correctly logged and surfaced,
so that failures are visible in the process monitor rather than silently swallowed.

**Acceptance Criteria:**

**Given** an exception is caught in `execute_substep`'s CATCH block
**When** the CATCH block runs
**Then** the exception detail is written to `mo_log`
**And** `rs_result-success = abap_false` is set
**And** `rs_result-message` contains the exception text

**Given** a `ZCX_FI_PROCESS_ERROR` is raised inside `execute_substep`
**When** it is caught
**Then** it is re-raised or its message is propagated — not silently discarded

**Tasks / Subtasks:**

- [ ] Task 3.3.1: Review all CATCH blocks in `execute` and `execute_substep` of
  `ZCL_FI_ALLOC_STEP_PHASE2`
- [ ] Task 3.3.2: For each CATCH block that does not set `rs_result-success = abap_false` or
  write to `mo_log`, add the missing lines
- [ ] Task 3.3.3: Ensure no CATCH block has fall-through logic that sets success to true after
  catching an exception

**Dev Notes:**
- Use `lv_dummy TYPE string` (not `TYPE c`) for `MESSAGE ... INTO lv_dummy` — this story
  should be done in concert with Story 5.3 (CC-06) if touching the same file.
- Reference: Phase 2 FINDING-04.

---

## Epic 4: Functional Correctness — Data Integrity

**Goal:** Fix regressions and bugs that cause the process to operate on the wrong scope of
data or leave the system in an inconsistent state.

---

### Story 4.1: Restore BRAND / HIER1 filter parameters in PHASE1 and PHASE3

As a business user running the allocation process,
I want PHASE1 and PHASE3 to apply BRAND and HIER1 filters correctly,
so that the allocation is scoped to the intended subset of data rather than processing all
line items regardless of brand or hierarchy.

**Acceptance Criteria:**

**Given** the process is started with `BRAND` and `HIER1` parameters set
**When** PHASE1 calls `ZCL_FI_ALLOCATIONS=>GET_BASE_KEYS`
**Then** `iv_brand` and `iv_hier1` are passed the values from the process parameters
**And** the resulting `lt_base_keys` contains only items matching those filters

**Given** the process is started with `BRAND` and `HIER1` parameters set
**When** PHASE3 calls `ZCL_FI_ALLOCATIONS=>GET_UNASSIGNED`
**Then** `iv_brand` and `iv_hier1` are passed the values from the process parameters

**Given** BRAND and HIER1 are not present in `ZFI_ALLOC_PROCESS_PARAMS` or
`ZFI_ALLOC_ADD_PROCESS_PARAMS`
**When** this story is implemented
**Then** the DDIC tables are extended with `BRAND` and `HIER1` fields before step code
reads them

**Tasks / Subtasks:**

- [ ] Task 4.1.1: Verify current DDIC structure of `ZFI_ALLOC_PROCESS_PARAMS` and
  `ZFI_ALLOC_ADD_PROCESS_PARAMS` (confirm BRAND and HIER1 are absent)
- [ ] Task 4.1.2: If absent — add `BRAND TYPE wrf_brand_id` and `HIER1 TYPE zmatklh1`
  (or appropriate data elements) to `ZFI_ALLOC_PROCESS_PARAMS` (or `ZFI_ALLOC_ADD_PROCESS_PARAMS`)
  as optional fields
- [ ] Task 4.1.3: In `ZCL_FI_ALLOC_STEP_PHASE1->init`: read BRAND and HIER1 into
  `mv_brand` and `mv_hier1` private attributes using `get_init_param_value`
- [ ] Task 4.1.4: In `ZCL_FI_ALLOC_STEP_PHASE1->execute`: replace `iv_brand = ''` and
  `iv_hier1 = ''` with `iv_brand = mv_brand` and `iv_hier1 = mv_hier1`
- [ ] Task 4.1.5: In `ZCL_FI_ALLOC_STEP_PHASE3->init`: same as 4.1.3
- [ ] Task 4.1.6: In `ZCL_FI_ALLOC_STEP_PHASE3->execute`: same as 4.1.4
- [ ] Task 4.1.7: Add `mv_brand` and `mv_hier1` to the PRIVATE SECTION of both
  `ZCL_FI_ALLOC_STEP_PHASE1` and `ZCL_FI_ALLOC_STEP_PHASE3`
- [ ] Task 4.1.8: Update log messages in PHASE1 that read `Značka: { '' }` and `Hier1: { '' }`
  to use the actual variable values

**Dev Notes:**
- Constitution Principle I (DDIC-First): use proper DDIC data elements for the new fields.
  Do not use local TYPE definitions.
- Constitution Principle III: consult mcp-sap-docs for the correct data elements for
  `wrf_brand_id` and `zmatklh1` in ABAP 7.58 on-premise.
- If BRAND/HIER1 are optional (process may run without them), treat initial value as
  "no filter" — pass `' '` or `''` to the method, which is the original default.
- Reference: CC-09, `cross-cutting-analysis.md` §3.

---

### Story 4.2: Fix INIT `execute` to always INSERT allocation state row

As a downstream step (PHASE1, PHASE2, PHASE3),
I want the INIT step to always create the `ZFI_ALLOC_STATE` row,
so that subsequent steps can UPDATE the row without finding zero matches.

**Acceptance Criteria:**

**Given** `FORCE_START = abap_false` and no existing state row
**When** INIT's `execute` completes
**Then** a `ZFI_ALLOC_STATE` row exists with all phase statuses set to initial values

**Given** `FORCE_START = abap_true` and an existing state row
**When** INIT's `execute` completes
**Then** the old row is deleted and a new row is inserted

**Given** `FORCE_START = abap_false` and an existing state row (already initialised)
**When** INIT's `execute` runs
**Then** the existing row is NOT deleted and NOT re-inserted; `execute` returns success

**Tasks / Subtasks:**

- [ ] Task 4.2.1: Refactor the `execute` method of `ZCL_FI_ALLOC_STEP_INIT`:
  - [ ] Check for existing `ZFI_ALLOC_STATE` row
  - [ ] If `FORCE_START = true`: DELETE existing row (if any); INSERT new row
  - [ ] If `FORCE_START = false` and row exists: skip insert, return success
  - [ ] If `FORCE_START = false` and row does NOT exist: INSERT new row (this is the
    missing branch — the current bug)

**Dev Notes:**
- The corrected implementation template is in `zfi_alloc_init-migration-review.md` FINDING-02.
- `INSERT` must not issue `COMMIT WORK` — the framework commits after `execute` returns.
- Reference: CC-15, `zfi_alloc_init-migration-review.md`.

---

### Story 4.3: Fix `lt_items` memory accumulation in CORR_BCHE

As a production system running CORR_BCHE over a large header set,
I want the item table to be cleared between loop iterations,
so that application server memory is not exhausted by accumulating all items across
all headers simultaneously.

**Acceptance Criteria:**

**Given** the inner loop processes N headers sequentially
**When** each header's item SELECT runs
**Then** `lt_items` contains only the items for the current header (not items from all
previous headers combined)

**Tasks / Subtasks:**

- [ ] Task 4.3.1: In the `LOOP AT lt_headers` block in `ZCL_FI_ALLOC_STEP_CORR_BCHE->execute`:
  - [ ] Add `CLEAR lt_items.` before the `SELECT * FROM zfi_alloc_bcitm INTO TABLE lt_items`
    (or change to inline declaration `INTO TABLE @DATA(lt_items)` for automatic fresh table)

**Dev Notes:**
- Using inline `@DATA(lt_items)` is the cleaner ABAP 7.4+ approach and prevents the bug
  from recurring. Verify abaplint configuration accepts inline declarations in this context.
- Reference: CC-14, `cross-cutting-analysis.md` §3.

---

## Epic 5: Error Handling and Observability

**Goal:** Replace unreliable error control flow patterns with proper exception handling,
add parameter guards to `validate`, and add BAL logging to the one step that has none.

---

### Story 5.1: Replace `MESSAGE TYPE 'E'` with `RAISE EXCEPTION` in PHASE1, PHASE2, PHASE3

As a process monitor user,
I want step failures to surface with their full message text,
so that the root cause is visible in the process monitor without needing to dig into
ABAP dumps.

**Acceptance Criteria:**

**Given** a validation or runtime guard fails inside `execute` or `execute_substep`
**When** the step signals failure
**Then** a `ZCX_FI_PROCESS_ERROR` is raised with a meaningful message text
**And** the process monitor shows the full message text for the failed step
**And** no `CX_SY_NO_HANDLER` exception appears in the system log

**Tasks / Subtasks:**

- [ ] Task 5.1.1: In `ZCL_FI_ALLOC_STEP_PHASE1`: replace all `MESSAGE e... TYPE 'E'` with
  `RAISE EXCEPTION TYPE zcx_fi_process_error EXPORTING textid = ... value = ...`
- [ ] Task 5.1.2: In `ZCL_FI_ALLOC_STEP_PHASE2`: same
- [ ] Task 5.1.3: In `ZCL_FI_ALLOC_STEP_PHASE3`: same
- [ ] Task 5.1.4: For each replacement, verify the equivalent message text is available in
  message class `ZFI_ALLOC` (or create a new message if none matches)

**Dev Notes:**
- Constitution Principle V: use `ZCX_FI_PROCESS_ERROR`; populate `textid`, `step_number`,
  `process_type` parameters.
- Item-level log messages (per-item errors inside `LOOP`) should remain as
  `MESSAGE ... INTO lv_dummy` + `lo_log->log_sy_msg()`, but `lv_dummy` must be `TYPE string`
  (see Story 5.3).
- Reference: CC-03.

---

### Story 5.2: Implement real `validate` guards in PHASE1, PHASE2, PHASE3, CORR_BCHE

As a framework safeguard,
I want `validate` to check all mandatory parameters before any DML is attempted,
so that a misconfigured process fails fast with a clear message rather than performing
destructive operations on incorrect data.

**Acceptance Criteria:**

**Given** a process is started with an initial (empty) `ALLOCATION_ID`
**When** the framework calls `validate` on any step
**Then** `rs_result-success = abap_false`, `rs_result-can_continue = abap_false`
**And** `rs_result-message` identifies which parameter is missing

**Given** all four mandatory parameters are populated (`ALLOCATION_ID`, `COMPANY_CODE`,
`FISCAL_YEAR`, `FISCAL_PERIOD`)
**When** `validate` runs
**Then** `rs_result-success = abap_true` and execution proceeds

**Tasks / Subtasks:**

- [ ] Task 5.2.1: In `ZCL_FI_ALLOC_STEP_PHASE1->validate`:
  - [ ] Check `mv_allocation_id`, `mv_company_code`, `mv_fiscal_year`, `mv_fiscal_period`
    are not initial
  - [ ] Return `success = abap_false`, `can_continue = abap_false`, descriptive message if any
    are initial
  - [ ] Optionally: move allocation state 'R' guard from `execute` to `validate`
- [ ] Task 5.2.2: In `ZCL_FI_ALLOC_STEP_PHASE2->validate`: same
- [ ] Task 5.2.3: In `ZCL_FI_ALLOC_STEP_PHASE3->validate`: same
- [ ] Task 5.2.4: In `ZCL_FI_ALLOC_STEP_CORR_BCHE->validate`: same

**Dev Notes:**
- Model on `ZCL_FI_ALLOC_STEP_INIT->validate` which already demonstrates the correct pattern.
- Do not perform DB reads in `validate` unless they are essential pre-condition checks —
  keep `validate` fast.
- Reference: CC-04.

---

### Story 5.3: Fix `lv_dummy TYPE c` to `TYPE string` in PHASE1, PHASE2, PHASE3

As a process monitor user,
I want error messages to contain their full text,
so that the message is readable rather than truncated to a single character.

**Acceptance Criteria:**

**Given** a `MESSAGE ... INTO lv_dummy` is executed during step processing
**When** the message text is longer than 1 character
**Then** `lv_dummy` holds the full message text (not just the first character)

**Given** `rs_result-message = lv_dummy` is set after an error catch
**When** the process monitor displays the step result message
**Then** the full message text is visible

**Tasks / Subtasks:**

- [ ] Task 5.3.1: In `ZCL_FI_ALLOC_STEP_PHASE1`: change `DATA lv_dummy TYPE c` to
  `DATA lv_dummy TYPE string`
- [ ] Task 5.3.2: In `ZCL_FI_ALLOC_STEP_PHASE2`: same
- [ ] Task 5.3.3: In `ZCL_FI_ALLOC_STEP_PHASE3`: same
- [ ] Task 5.3.4: Verify no other variables of `TYPE c` (without explicit length) are used as
  message sinks in these three files

**Dev Notes:**
- This is a one-line change per file. Low risk, high value.
- Reference: CC-06.

---

### Story 5.4: Add BAL log initialisation and logging to CORR_BCHE

As an operations analyst,
I want CORR_BCHE to write a BAL log entry on completion,
so that the total number of headers processed and any errors are visible in the
application log (SLG1).

**Acceptance Criteria:**

**Given** CORR_BCHE executes successfully
**When** `execute` completes
**Then** a BAL log entry exists recording the total number of `ZFI_ALLOC_BCHE` headers
processed and the count of `no_items = 'Y'` updates

**Given** a MODIFY operation fails on an individual header
**When** the error is detected
**Then** the error is written to the BAL log (not silently ignored)

**Tasks / Subtasks:**

- [ ] Task 5.4.1: Add `DATA mo_log TYPE REF TO zcl_sp_ballog` to PRIVATE SECTION of
  `ZCL_FI_ALLOC_STEP_CORR_BCHE`
- [ ] Task 5.4.2: In `init`: initialise `mo_log = NEW zcl_sp_ballog( ... )` with appropriate
  sub-object constant (define `zcl_fi_allocations=>c_bal_subobj_corr_bche` if not yet existing)
- [ ] Task 5.4.3: In `execute`: write summary log entry on completion (total processed, counts)
- [ ] Task 5.4.4: Where MODIFY errors can be detected, write error entries to `mo_log`

**Dev Notes:**
- Constitution Principle III: consult mcp-sap-docs for `ZCL_SP_BALLOG` initialisation pattern
  before implementing.
- Reference: CC-13.

---

## Epic 6: Wire Phase 3 into Pipeline

**Goal:** After all blocking defects in Phase 3 are resolved (CC-01 stubs added in E1,
BRAND/HIER1 fixed in E4, rollback implemented in E2), uncomment Phase 3 from the process
definition so the full pipeline executes end-to-end.

**Prerequisite:** Epics 1, 2, 4 (Stories 1.1, 2.1, 4.1) must all be complete.

---

### Story 6.1: Wire Phase 3 step into process definition

As a pipeline operator,
I want the full five-step allocation pipeline to execute end-to-end,
so that Phase 3 (allocation item generation) is part of the process and not silently
skipped.

**Acceptance Criteria:**

**Given** the process type `ALLOCATIONS` is set up via `ZFI_ALLOC_PROC_TEST`
**When** `setup_process_data` runs
**Then** step 0005 (PHASE3, `ZCL_FI_ALLOC_STEP_PHASE3`, `substep_mode = 'SERIAL'`) is
registered as an active step

**Given** the full pipeline runs with valid parameters
**When** step 0004 (PHASE2) completes successfully
**Then** step 0005 (PHASE3) is scheduled and executed

**Tasks / Subtasks:**

- [ ] Task 6.1.1: Uncomment the step 0005 block in `ZFI_ALLOC_PROC_TEST->setup_process_data`
- [ ] Task 6.1.2: Verify `substep_mode = 'SERIAL'` is set (not 'QUEUE')
- [ ] Task 6.1.3: Run end-to-end test and confirm PHASE3 executes and completes

**Dev Notes:**
- Do not uncomment until E1 Story 1.1 (PHASE3 stubs), E2 Story 2.1 (rollback), and
  E4 Story 4.1 (BRAND/HIER1) are all done and verified.
- Reference: CC-11.

---

## Epic 7: Code Quality and Housekeeping

**Goal:** Remove dead code, fix cosmetic issues, and document framework-level ambiguities.
These changes carry no functional risk but improve maintainability.

---

### Story 7.1: Remove `ms_context` dead field from all step classes

As a developer maintaining step classes,
I want dead private fields removed,
so that the class declaration accurately reflects the data actually used.

**Acceptance Criteria:**

**Given** any of the five step classes
**When** a code review is performed
**Then** no `ms_context` field exists in the PRIVATE SECTION that is written in `init`
but never read subsequently

**Tasks / Subtasks:**

- [ ] Task 7.1.1: Remove `ms_context` from PRIVATE SECTION and `init` assignment in
  `ZCL_FI_ALLOC_STEP_INIT`
- [ ] Task 7.1.2: Same for `ZCL_FI_ALLOC_STEP_PHASE1`
- [ ] Task 7.1.3: Same for `ZCL_FI_ALLOC_STEP_PHASE2`
- [ ] Task 7.1.4: Same for `ZCL_FI_ALLOC_STEP_PHASE3`
- [ ] Task 7.1.5: Same for `ZCL_FI_ALLOC_STEP_CORR_BCHE`

**Dev Notes:**
- Confirm with a grep that `ms_context` is not read anywhere in each class before removing.
- Reference: CC-07.

---

### Story 7.2: Update class header comments in all step classes

As a developer reading step class source,
I want the file header comment to describe the actual class,
so that I understand the class purpose from the first line rather than reading a
placeholder template comment.

**Acceptance Criteria:**

**Given** any of the five step classes
**When** the source is opened
**Then** the first two comment lines do NOT read `*& Class ZCL_STEP_FETCH_DATA` /
`*& Example step: Fetch data from database`
**And** the comment describes the actual class purpose

**Tasks / Subtasks:**

- [ ] Task 7.2.1: Update header comment in `ZCL_FI_ALLOC_STEP_INIT`
- [ ] Task 7.2.2: Update header comment in `ZCL_FI_ALLOC_STEP_PHASE1`
- [ ] Task 7.2.3: Update header comment in `ZCL_FI_ALLOC_STEP_PHASE2`
- [ ] Task 7.2.4: Update header comment in `ZCL_FI_ALLOC_STEP_PHASE3`
- [ ] Task 7.2.5: Update header comment in `ZCL_FI_ALLOC_STEP_CORR_BCHE`

**Dev Notes:** Reference: CC-08.

---

### Story 7.3: Delete `*_ORIG` dead classes from system and repository

As a developer working in the `ZFI_ALLOC_PROCESS` package,
I want obsolete snapshot classes removed,
so that there is no namespace confusion and the package contains only live objects.

**Acceptance Criteria:**

**Given** the SAP system and abapGit repository
**When** cleanup is complete
**Then** `ZCL_FI_ALLOC_STEP_PHASE2_ORIG` does not exist as an active SAP object
**And** `ZCL_FI_ALLOC_STEP_PHASE3_ORIG` does not exist as an active SAP object
**And** their `.clas.xml` and `.clas.abap` files are removed from the repository

**Tasks / Subtasks:**

- [ ] Task 7.3.1: Delete `ZCL_FI_ALLOC_STEP_PHASE2_ORIG` from SAP system
- [ ] Task 7.3.2: Delete `ZCL_FI_ALLOC_STEP_PHASE3_ORIG` from SAP system
- [ ] Task 7.3.3: Remove corresponding files from `src/zfi_alloc_process/` in the git repository

**Dev Notes:**
- Verify neither class is referenced anywhere before deletion (grep for class name).
- Reference: CC-10.

---

### Story 7.4: Document `cleanup_old_instances` commit intent

As a framework developer,
I want the commit behaviour of `cleanup_old_instances` to be explicit and documented,
so that callers know whether they must issue a `COMMIT WORK` after calling it.

**Acceptance Criteria:**

**Given** `ZCL_FI_PROCESS_MANAGER->cleanup_old_instances` is called
**When** the method completes
**Then** either: a `COMMIT WORK AND WAIT` is added at the end (if the intent is auto-commit),
OR an ABAP-Doc comment on the method states "Caller must commit" (if intentional)

**Tasks / Subtasks:**

- [ ] Task 7.4.1: Determine intent by reviewing callers of `cleanup_old_instances` and the
  method's design context
- [ ] Task 7.4.2: Add `COMMIT WORK AND WAIT` to the method if callers rely on it being
  auto-committed, OR add ABAP-Doc `@parameter` / method description comment clarifying
  the caller's responsibility

**Dev Notes:**
- Do not add `COMMIT WORK` blindly if callers intentionally defer the commit.
- Reference: LUW-03, `luw-schema-analysis.md` §4.2.

---

## Dependency Graph

```
E1 (activation)
├── Story 1.1 (stubs)  ─────────────────────── enables E6 Story 6.1
└── Story 1.2 (WRITE:) ─────────────────────── unblocks CORR_BCHE runtime

E2 (LUW / rollback)
├── Story 2.1 (rollback methods) ───────────── enables E6 Story 6.1
└── Story 2.2 (bgRFC init fix) ──────────────── required before E3 Story 3.1 effective

E3 (PHASE2 substep path)
├── Story 3.1 (mo_log to init) ──────────────── depends on Story 2.2
├── Story 3.2 (on_success/on_error) ─────────── independent
└── Story 3.3 (CATCH fix) ───────────────────── independent

E4 (data integrity)
├── Story 4.1 (BRAND/HIER1) ─────────────────── enables E6 Story 6.1
├── Story 4.2 (INIT INSERT) ─────────────────── independent, should be in Sprint 1
└── Story 4.3 (lt_items clear) ─────────────── independent

E5 (error handling)
├── Story 5.1 (MESSAGE→RAISE) ───────────────── independent
├── Story 5.2 (validate guards) ────────────── independent
├── Story 5.3 (lv_dummy TYPE string) ────────── independent, low effort
└── Story 5.4 (CORR_BCHE BAL log) ──────────── depends on Story 1.2 done first

E6 (Phase 3 wiring) ─────────────────────────── depends on E1+E2+E4

E7 (housekeeping) ───────────────────────────── independent, can be done anytime
```

---

## Suggested Sprint Assignment

### Sprint 1 — Unblock the pipeline (all Critical)

| Story | Finding(s) | Effort |
|-------|-----------|--------|
| 1.1 — Serial stubs | CC-01 | 45 min |
| 1.2 — Remove WRITE: | CC-12 | 5 min |
| 2.2 — bgRFC init fix | CC-16 | 1.5h |
| 4.2 — INIT state INSERT | CC-15 | 30 min |
| 4.3 — lt_items clear | CC-14 | 15 min |
| 5.3 — lv_dummy TYPE string | CC-06 | 30 min |

### Sprint 2 — Structural correctness (High / Moderate)

| Story | Finding(s) | Effort |
|-------|-----------|--------|
| 2.1 — Rollback methods | CC-05 | 3h |
| 3.1 — mo_log to init | FINDING-08 | 30 min |
| 3.2 — on_success/on_error | FINDING-03 | 2h |
| 3.3 — CATCH fix | FINDING-04 | 1h |
| 5.1 — MESSAGE→RAISE | CC-03 | 2h |
| 5.2 — validate guards | CC-04 | 2h |
| 5.4 — CORR_BCHE BAL log | CC-13 | 1h |

### Sprint 3 — Wire Phase 3 + housekeeping

| Story | Finding(s) | Effort |
|-------|-----------|--------|
| 6.1 — Wire PHASE3 | CC-11 | 30 min |
| 7.1 — Remove ms_context | CC-07 | 30 min |
| 7.2 — Header comments | CC-08 | 15 min |
| 7.3 — Delete *_ORIG classes | CC-10 | 30 min |
| 7.4 — Document cleanup commit | LUW-03 | 30 min |

---

## Dev Agent Record

### Agent Model Used

github-copilot/claude-sonnet-4.6

### Completion Notes List

### File List (to be updated during implementation)
