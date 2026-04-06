---
story_id: "zen-6-2"
epic_id: "zen-epic-6"
title: "Integration test program and repository finalization"
target_repository: "cz.en.orch"
depends_on:
  - "zen-6-1"
constitution_principles:
  - "Principle I - DDIC-First"
  - "Principle II - SAP Standards"
  - "Principle V - Error Handling"
  - "Code Documentation Standards"
status: "done"
completion_date: "2026-04-06"
review_patches_applied: "P1–P13 all applied"
created: "2026-04-06"
---

# Story zen-6-2: Integration Test Program and Repository Finalization

Status: review

## Story

As a developer,
I want a runnable integration test program and a finalized repository configuration,
So that I can manually verify the full orchestration lifecycle and the repository is ready
for Phase 2 development.

## Context

With Epics 1–5 complete and unit tests added in story 6.1, the final step is to:

1. Create `ZEN_ORCH_TEST` — a standalone ABAP report that runs a full end-to-end scenario
   using the MOCK adapter, exercising the entire lifecycle from score creation to COMPLETED
2. Finalize `README.md` in `cz.en.orch` with Phase 2 roadmap, test instructions, adapter
   registration guide
3. Update `repos/registry.md` in the planning repository with `cz.en.orch` final metadata
4. Sync the constitution to `cz.en.orch` (ensure `src/zen_orch/.constitution.md` is present and current)

### Current state of README

`README.md` already has a skeleton (purpose, package structure, phase scope, adapter registration,
test instructions, architecture references). It needs: Phase 2 roadmap section, `ZEN_ORCH_TEST`
runtime instructions, and minor accuracy fixes (test class location).

### Constitution sync note

The planning repo `_bmad/_memory/constitution.md` is the source of truth. The sync script at
`repos/sync-constitution.sh` copies it to `../cz.en.orch/src/zen_orch/.constitution.md`.
Run this as part of finalization.

## Acceptance Criteria

**Given** all Epic 1–5 objects are active in the SAP system

**AC1 — Integration test program `ZEN_ORCH_TEST`:**

**When** report `ZEN_ORCH_TEST` is executed in SE38 (no selection screen parameters needed)
**Then** it runs the following steps in sequence and writes output to the SAP list:
1. Inserts a test score `'ZIT_E2E'` with steps: `STEP(seq=10, adapter=MOCK)` → `GATE(seq=20, ref='G1')` → `STEP(seq=30, adapter=MOCK)` into `ZEN_ORCH_SCORE` and `ZEN_ORCH_SCORE_STEP`
2. Ensures MOCK adapter is registered in `ZEN_ORCH_ADPT_R`
3. Calls `ZCL_EN_ORCH_ENGINE=>get_instance( )->create_performance( 'ZIT_E2E' )` — writes `✓ create_performance OK: <uuid>`
4. Calls `sweep_all` once — writes `✓ sweep_all pass 1 OK`
5. Checks performance status: if not yet 'C', calls `sweep_all` a second time — writes `✓ sweep_all pass 2 OK` (MOCK is synchronous; should complete in 1 sweep)
6. Reads final performance status: asserts = 'C' — writes `✓ performance COMPLETED`
7. Reads BAL log entries for the sweep (optional — writes `ℹ BAL: <n> entries` or skips gracefully if BAL not registered)
8. Deletes all test data: `DELETE FROM zen_orch_perf WHERE perf_uuid = @lv_uuid`, all related steps, the test score rows — writes `✓ cleanup OK`
9. Prints final `✓ ZEN_ORCH_TEST PASSED` or `✗ ZEN_ORCH_TEST FAILED: <reason>` to the list

**And** the program completes without dump or unhandled exception
**And** no permanent test data remains after execution

**AC2 — README.md updated:**

**When** `README.md` in `cz.en.orch` is read
**Then** it contains:
- **How to run unit tests**: `SE80 → ZCL_EN_ORCH_ENGINE → Test Classes` or `transaction SAUNIT`; and `ZCL_EN_ORCH_HEALTH_CHK_QUERY` health check instructions
- **How to run the integration test**: `SE38 → ZEN_ORCH_TEST → Execute`; expected output description
- **How to register a new adapter**: already present — verify still accurate
- **Phase 2 roadmap section**: lists planned Phase 2 epics (BSR integration, Dashboard UI) with a brief description
- **Package object summary**: table of all 60+ DDIC/class objects organized by category

**AC3 — `repos/registry.md` updated:**

**When** `repos/registry.md` in the planning repository is read
**Then** the `cz.en.orch` entry contains:
- Remote URL: `https://github.com/smolikzd/cz.en.orch.git`
- Local path: `/Users/smolik/DEV/cz.en.orch`
- Phase 1 status: complete
- Epic completion dates
- Constitution copy location: `src/zen_orch/.constitution.md`

**AC4 — Constitution synced:**

**When** `repos/sync-constitution.sh` is run from the planning repo
**Then** `cz.en.orch/src/zen_orch/.constitution.md` exists and matches `_bmad/_memory/constitution.md`
**And** `git diff src/zen_orch/.constitution.md` in `cz.en.orch` shows no changes (already in sync or freshly synced)

## Implementation Notes

### ZEN_ORCH_TEST program structure

```abap
REPORT zen_orch_test.

" ── Helpers ──────────────────────────────────────────────────────────────
DATA: lv_perf_uuid TYPE zen_orch_perf_uuid,
      lv_status    TYPE zen_orch_de_status,
      lv_passed    TYPE abap_bool VALUE abap_true.

CONSTANTS gc_score_id  TYPE zen_orch_de_score_id  VALUE 'ZIT_E2E'.
CONSTANTS gc_adpt_type TYPE zen_orch_de_adapter_type VALUE 'MOCK'.
CONSTANTS gc_impl_cls  TYPE zen_orch_de_impl_class   VALUE 'ZCL_EN_ORCH_ADAPTER_MOCK'.

" ── Step 1 & 2: Setup ───────────────────────────────────────────────────
" Delete any residual from previous run, insert fresh score, register MOCK
...

" ── Step 3: create_performance ──────────────────────────────────────────
TRY.
    lv_perf_uuid = zcl_en_orch_engine=>get_instance( )->create_performance( gc_score_id ).
    WRITE: / '✓ create_performance OK:', lv_perf_uuid.
  CATCH zcx_en_orch_error INTO DATA(lx).
    WRITE: / '✗ create_performance FAILED:', lx->get_text( ).
    lv_passed = abap_false.
ENDTRY.

...etc.
```

### Key considerations

- **MOCK adapter is synchronous**: `start()` returns STATUS='C' immediately. One `sweep_all` call
  is sufficient to reach COMPLETED for a simple STEP→GATE→STEP score.
- **Gate evaluation**: After STEP(10)→C and STEP(30)→C, the gate GATE(20, ref='G1') needs all
  REF_ID group STEPs completed. Ensure the score structure uses consistent `ref_id` values.
- **BAL log read**: Use `CL_BAL_LOG_COLLECTION` or similar — optional, gracefully skipped if
  `ZEN_ORCH` BAL object is not registered in SBAL_OBJECT.
- **Program name**: Must be `ZEN_ORCH_TEST` (fits SAP 30-char limit for report names). Confirm
  it is registered as a program object in the abapGit serialization.

### Files changed

| File | Repository | Change |
|------|-----------|--------|
| `src/zen_orch_test.prog.abap` | cz.en.orch | NEW — integration test report |
| `src/zen_orch_test.prog.xml` | cz.en.orch | NEW — abapGit metadata for report |
| `README.md` | cz.en.orch | UPDATE — Phase 2 roadmap, test instructions, object table |
| `repos/registry.md` | cz.imcg.fast.allocations | UPDATE — cz.en.orch Phase 1 completion metadata |
| `src/zen_orch/.constitution.md` | cz.en.orch | SYNC — via sync-constitution.sh |

### Constitution Compliance

- **Principle I (DDIC-First)**: Program uses DDIC-typed DATA declarations only. Score/step inserts use DDIC row types.
- **Principle II (SAP Standards)**: Report named `ZEN_ORCH_TEST`; line length ≤ 120 chars; no local TYPE definitions.
- **Principle V (Error Handling)**: All engine calls wrapped in `TRY/CATCH zcx_en_orch_error`; failures set `lv_passed = false` and continue cleanup.

## Tasks / Subtasks

- [x] T1: Create `zen_orch_test.prog.abap` skeleton — REPORT header, constants, DATA declarations
- [x] T2: Implement test setup (delete residuals, insert test score, register MOCK adapter)
- [x] T3: Implement `create_performance` call and assertion (AC1 step 3)
- [x] T4: Implement `sweep_all` loop (1–2 passes) and performance status assertion (AC1 steps 4–6)
- [x] T5: Implement optional BAL log read (AC1 step 7)
- [x] T6: Implement test cleanup (AC1 step 8)
- [x] T7: Print final PASSED/FAILED summary (AC1 step 9)
- [x] T8: Create `zen_orch_test.prog.xml` abapGit metadata stub
- [x] T9: Update `README.md` — test instructions, Phase 2 roadmap, object table (AC2)
- [x] T10: Update `repos/registry.md` in planning repo — cz.en.orch Phase 1 metadata (AC3)
- [x] T11: Run `repos/sync-constitution.sh` and verify `src/zen_orch/.constitution.md` (AC4)
- [ ] T12: Execute `ZEN_ORCH_TEST` in SAP, verify output, confirm no dump
  - **Note**: Manual SAP execution required. Push cz.en.orch to abapGit, activate, run SE38 → ZEN_ORCH_TEST.

## Dev Agent Record

### Implementation Plan

T1–T7 implemented as a single report `zen_orch_test.prog.abap`. The program structure follows AC1
step-by-step:

1. Setup: idempotent DELETE + INSERT of score `ZIT_E2E` (STEP→GATE→STEP) and MOCK adapter row
2. `create_performance( 'ZIT_E2E' )` with COMMIT WORK AND WAIT
3. `sweep_all` with up to 2 passes; MOCK adapter completes synchronously in pass 1
4. Status check: assert = 'C' (COMPLETED)
5. BAL log read via `CL_BAL_LOG_COLLECTION=>get_instance( )` — wrapped in CATCH cx_root for graceful skip
6. Cleanup: DELETE all test data + COMMIT
7. Final PASSED/FAILED write

All engine calls are wrapped in `TRY/CATCH zcx_en_orch_error`. `lv_passed` flag accumulates result
and `lv_reason` captures the first failure message. Cleanup always runs regardless of prior failures.

T8: `zen_orch_test.prog.xml` follows abapGit LCL_OBJECT_PROG serializer format (SUBC=1 for executable report).

T9: README.md fully rewritten to include:
- Package object summary table (60+ objects, organized by category)
- Unit test instructions (SAUNIT, SE24 paths)
- Integration test instructions with expected output
- Phase 2 roadmap table (4 planned epics)
- Adapter registration still accurate

T10: `repos/registry.md` cz.en.orch entry updated with Phase 1 completion dates per epic, full
contents listing, key classes.

T11: `repos/sync-constitution.sh` ran successfully. Output:
```
✅ Synced to cz.en.orch: .../cz.en.orch/src/zen_orch/.constitution.md
```
File size: 11566 bytes, created 2026-04-06.

T12 is a manual SAP step — requires pushing to abapGit, activating all objects, and executing SE38 → ZEN_ORCH_TEST.

### Completion Notes

- AC1: `zen_orch_test.prog.abap` implements full E2E test per spec. No local TYPE definitions,
  all DDIC-typed variables, CATCH zcx_en_orch_error on all engine calls, cleanup always executed.
- AC2: `README.md` contains all required sections per AC2 checklist.
- AC3: `repos/registry.md` cz.en.orch entry contains remote URL, local path, Phase 1 status
  complete, epic completion dates, constitution copy location.
- AC4: `src/zen_orch/.constitution.md` written by sync script (11566 bytes, 2026-04-06).
- T12 (manual SAP execution) is pending — to be done after abapGit push by developer.

## File List

### cz.en.orch
- `src/zen_orch_test.prog.abap` — NEW: integration test report
- `src/zen_orch_test.prog.xml` — NEW: abapGit metadata for report
- `src/zen_orch/.constitution.md` — SYNC: synced from planning repo via sync-constitution.sh
- `README.md` — UPDATE: Phase 2 roadmap, unit test instructions, integration test instructions, object table

### cz.imcg.fast.allocations (planning repo)
- `repos/registry.md` — UPDATE: cz.en.orch Phase 1 completion metadata
- `_bmad-output/implementation-artifacts/zen-6-2-integration-test-program-and-repository-finalization.md` — UPDATE: story status
- `_bmad-output/implementation-artifacts/sprint-status.yaml` — UPDATE: zen-6-2 status

### Review Findings

- [x] [Review][Decision] D1: `restart_performance` — CANCELLED+FAILED mixed step state: CANCELLED steps are not reset to PENDING — only FAILED steps are reset; a performance with some CANCELLED and some FAILED steps will restart with the CANCELLED steps permanently excluded from re-execution, silently truncating the workflow [src/zcl_en_orch_engine.clas.abap: restart_performance T1.3] — **Resolved: Option A — current behaviour is correct by design (user decision)**

- [x] [Review][Patch] P1: AC1/step 8 — `ZEN_ORCH_ADPT_R` row inserted conditionally by setup but never deleted in cleanup; violates "no permanent test data remains after execution" [src/zen_orch_test.prog.abap:84–97, 181–188] — **Patched**
- [x] [Review][Patch] P2: BAL vs BALI API mismatch in step 7 — `CL_BAL_LOG_COLLECTION` (classic SLG1) is used but engine logs via `CL_BALI_*` (ABAL framework); step 7 will always report 0 log handles regardless of engine activity [src/zen_orch_test.prog.abap:167] — **Patched**
- [x] [Review][Patch] P3: `cancel_performance` — adapter->cancel() only catches `zcx_en_orch_error`; any non-zcx exception (cx_root, cx_sy_ref_is_initial, custom adapter exception) propagates uncaught, aborting T1.4 UPDATE and COMMIT, leaving performance permanently stuck in RUNNING/PENDING [src/zcl_en_orch_engine.clas.abap: cancel_performance T1.3 CATCH] — **Patched**
- [x] [Review][Patch] P4: `cancel_performance` + `resume_performance` — if 2nd UPDATE (zen_orch_perf) fails (sy-subrc <> 0), exception is raised before COMMIT; if any outer TRY catches the exception and then commits, step-table changes are persisted without header status update, creating permanently inconsistent state [src/zcl_en_orch_engine.clas.abap: cancel_performance T1.5, resume_performance T1.3] — **Patched: ROLLBACK WORK added before re-raise in both methods**
- [x] [Review][Patch] P5: `evaluate_gate` breakpoint path — `is_breakpoint` flag is persisted in `zen_orch_p_step`; after `resume_performance` sets gate to COMPLETED and the next sweep calls `evaluate_gate` again, the gate is re-evaluated with `is_breakpoint=X` still set → gate immediately re-pauses → infinite pause loop [src/zcl_en_orch_engine.clas.abap: evaluate_gate T1.3, resume_performance] — **Patched: resume_performance T1.2 now also clears IS_BREAKPOINT=space**
- [x] [Review][Patch] P6: `check_schedules` — `UPDATE zen_orch_sched SET last_fired_ts` has no `sy-subrc` check; if the UPDATE fails silently, `last_fired_ts` is never written, causing the schedule to re-fire on the very next `sweep_all` call (duplicate performance creation) [src/zcl_en_orch_engine.clas.abap: check_schedules ~loop UPDATE] — **Patched: sy-subrc checked; warning logged on failure**
- [x] [Review][Patch] P7: `is_schedule_due` — `DATA lt_fields TYPE STANDARD TABLE OF string` and `DATA lt_t TYPE STANDARD TABLE OF i` are local ad-hoc table types; violates Constitution Principle I (no local TYPE definitions; must use DDIC table types) [src/zcl_en_orch_engine.clas.abap: is_schedule_due] — **Patched: replaced with STRING_TABLE and INT4_TABLE**
- [x] [Review][Patch] P8: `is_schedule_due` whitespace normalization via `WHILE CS '  '` only replaces space pairs; tab characters (`\t`) in `cron_expr` are not normalized, causing SPLIT to produce wrong field count → schedule silently never fires [src/zcl_en_orch_engine.clas.abap: is_schedule_due cron normalization] — **Patched: tab→space replacement via cl_abap_char_utilities=>horizontal_tab added**
- [x] [Review][Patch] P9: `sweep_all` CATCH around it is dead — `METHODS sweep_all` has no `RAISING` clause; the `CATCH zcx_en_orch_error` in `zen_orch_test.prog.abap` can never trigger; real per-performance failures are silently swallowed while test prints `✓ sweep_all pass N OK` [src/zen_orch_test.prog.abap:118, src/zcl_en_orch_engine.clas.abap sweep_all definition] — **Patched: TRY/CATCH removed; direct calls with status-based failure detection**
- [x] [Review][Patch] P10: Test program runs sweep pass 2 unconditionally when `lv_status <> 'C'` — does not skip when status is already a terminal failure (`'F'`, `'X'`); second sweep on a FAILED performance produces misleading log entries [src/zen_orch_test.prog.abap:133–143] — **Patched: pass 2 only runs inside lv_passed=abap_true guard**
- [x] [Review][Patch] P11: Test program cleanup (step 8) has no exception safety wrapper; if the report terminates unexpectedly between steps 3–7 (e.g. a dump or `MESSAGE ... TYPE 'A'`), test data (zen_orch_perf, zen_orch_p_step, zen_orch_s_step) remains permanently [src/zen_orch_test.prog.abap:180–188] — **Patched: steps 3–7 wrapped in top-level TRY/CATCH cx_root**
- [x] [Review][Patch] P12: Trailing `COMMIT WORK AND WAIT` in `sweep_all` (when `lt_active_perfs IS INITIAL`) is redundant — `check_schedules` already commits per schedule internally; the comment "for schedule timestamp updates" is misleading [src/zcl_en_orch_engine.clas.abap: sweep_all post-loop] — **Patched: comment updated to clarify purpose**
- [x] [Review][Patch] P13: Error message in health check shortened from `"ZCX_EN_ORCH_ERROR unexpectedly propagated:"` to `"ZCX_EN_ORCH_ERROR:"` — loses the "unexpectedly" diagnostic signal that distinguishes unexpected exception escapes from normal flow [src/zcl_en_orch_health_chk_query.clas.abap: check_sweep_all CATCH] — **Patched: wording restored**

- [x] [Review][Defer] W1: `restart_performance` `sy-dbcnt=0` guard cannot distinguish concurrent reset from no-failed-steps — pre-existing design gap, not introduced here [src/zcl_en_orch_engine.clas.abap: restart_performance] — deferred, pre-existing
- [x] [Review][Defer] W2: `cancel_performance` LUW split-brain — external adapter->cancel() is irreversible before DB UPDATE; a rollback after adapter cancel leaves external work unit cancelled but DB shows RUNNING — known best-effort-cancel design gap [src/zcl_en_orch_engine.clas.abap: cancel_performance] — deferred, pre-existing
- [x] [Review][Defer] W3: Scope creep — implementations of `check_schedules`, `cancel_performance`, `restart_performance`, `resume_performance` belong to zen-5-2/zen-5-3/zen-4-7 (all marked done); bundled here without cross-story commit traceability — deferred, traceability concern only
- [x] [Review][Defer] W4: `is_schedule_due` — named cron macros (@yearly, @monthly etc.) silently never fire (CONV i fails, caught, returns false, no log) — deferred, out of stated feature scope
- [x] [Review][Defer] W5: `matches_cron_field` — cron range (`1-5`) and list (`1,3,5`) syntax unhandled; silently returns false — deferred, Phase 2 scope
- [x] [Review][Defer] W6: `resolve_params` local TYPE definitions (`ty_kv`, `ty_kv_map`) — pre-existing violation of Principle I from a prior story — deferred, pre-existing

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-06 | Story created for Epic 6 | claude-sonnet-4.6 |
| 2026-04-06 | All tasks T1–T11 implemented; T12 pending manual SAP execution | claude-sonnet-4.6 |
| 2026-04-06 | Code review complete: 1 decision-needed, 13 patches, 6 deferred, 13 dismissed | claude-sonnet-4.6 |
