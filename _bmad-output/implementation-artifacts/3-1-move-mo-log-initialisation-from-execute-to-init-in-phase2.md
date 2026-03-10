---
story: "3.1"
title: "Move mo_log initialisation from execute to init in PHASE2"
epic: "3 — Functional Correctness — PHASE2 Substep Path"
sprint: 2
status: done
completed: 2026-03-09
priority: critical
estimate: 30min
source_findings:
  - zfi_alloc_phase_2-migration-review.md
  - FINDING-08
  - CC-16
target_repo: smolikzd/cz.imcg.fast.planner
dependencies:
  - Story 2.2 (bgRFC init fix) — must be done first for the bgRFC substep path to benefit
---

# Story 3.1: Move `mo_log` initialisation from `execute` to `init` in PHASE2

## User Story

As a PHASE2 substep,
I want the application log to be initialised in `init`,
so that it is available in all execution paths including bgRFC queued substep execution.

## Background

`ZCL_FI_ALLOC_STEP_PHASE2->execute` currently initialises `mo_log`:
```abap
mo_log = NEW zcl_sp_ballog( ... ).
```

When a PHASE2 substep executes via bgRFC (`ZFI_BGRFC_EXEC_SUBSTEP`), the call sequence is:
1. `CREATE OBJECT lo_step TYPE (lv_class_name)`
2. `lo_step->init( ls_ctx )` ← added by Story 2.2
3. `lo_step->execute_substep( lv_substep_number )`

`execute` is never called in the bgRFC path. Therefore `mo_log` is `NULL` when
`execute_substep` runs, and any `mo_log->...` call raises `CX_SY_REF_IS_INITIAL`.

The fix: move `mo_log = NEW zcl_sp_ballog( ... )` from `execute` to `init`.
`execute` must then use the already-initialised `mo_log` without re-creating it.

## Acceptance Criteria

**AC-01 — `mo_log` initialised in `init`:**
Given a PHASE2 step object is created and `init` is called
When `init` completes
Then `mo_log` is a valid, non-null `ZCL_SP_BALLOG` instance

**AC-02 — No CX_SY_REF_IS_INITIAL in bgRFC path:**
Given `execute_substep` is called in the bgRFC context (after `init` per Story 2.2)
When any `mo_log->` call is made inside `execute_substep`
Then no `CX_SY_REF_IS_INITIAL` exception is raised

**AC-03 — `execute` uses the same instance:**
Given `execute` is called (non-substep serial fallback path)
When log write operations occur inside `execute`
Then the same `mo_log` instance initialised in `init` is used
And `mo_log` is NOT re-initialised inside `execute`

## Tasks

- [ ] **Task 3.1.1**: Read `ZCL_FI_ALLOC_STEP_PHASE2->init` in full to understand the
  current content and where `mo_log` initialisation should be inserted.

- [ ] **Task 3.1.2**: Read `ZCL_FI_ALLOC_STEP_PHASE2->execute` in full to locate the
  exact `mo_log = NEW zcl_sp_ballog( ... )` line (with its parameters).

- [ ] **Task 3.1.3**: Move the `mo_log = NEW zcl_sp_ballog( ... )` line from `execute`
  to `init` (append after existing `init` logic, before `ENDMETHOD`).

- [ ] **Task 3.1.4**: Remove the `mo_log = NEW zcl_sp_ballog( ... )` line from `execute`.
  Do NOT replace it with any other initialisation.

- [ ] **Task 3.1.5**: Verify all `mo_log->...` calls in both `execute` and `execute_substep`
  still compile (no reference before assignment).

- [ ] **Task 3.1.6**: Confirm `mo_log` is declared as `TYPE REF TO zcl_sp_ballog` in
  PRIVATE SECTION (not as an inline variable inside `execute`).

## Dev Notes

- **Dependency on Story 2.2**: this change only fixes the bgRFC path if Story 2.2 has
  already been applied (i.e., `init` is actually called before `execute_substep` in
  `ZFI_BGRFC_EXEC_SUBSTEP`). Both stories are needed for full correctness.
- **`mo_log` constructor parameters**: copy the exact constructor call from `execute`
  to `init` — do not change the parameters.
- **Constitution Principle III**: if the `ZCL_SP_BALLOG` constructor signature is
  unclear, consult mcp-sap-docs before proceeding.
- Reference: Phase 2 FINDING-08, CC-16, `luw-schema-analysis.md` §6.

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` | Move `mo_log = NEW zcl_sp_ballog(...)` from `execute` to `init` |

## Constitution Compliance Checklist

- [ ] Principle I — DDIC-First: N/A (no new types)
- [ ] Principle II — SAP Standards: line ≤ 120 chars
- [ ] Principle III — Consult SAP Docs: `ZCL_SP_BALLOG` constructor verified
- [ ] Principle IV — Factory Pattern: `NEW` is acceptable for log instantiation (consistent with existing pattern)
- [ ] Principle V — Error Handling: `init` already wraps setup in TRY-CATCH per Story 2.2; cover `mo_log` init failure there

## Dev Agent Record

### Agent Model Used
claude-sonnet-4.5 (github-copilot/claude-sonnet-4.5)

### Completion Notes
- Moved `CREATE OBJECT mo_log TYPE zcl_sp_ballog` and `mo_log->log_create()` from execute method to init method
- Removed `ls_log` declaration from execute (now declared in init where it's needed)
- Ensures `mo_log` is available in both serial (execute) and bgRFC (execute_substep) paths
- Fixes CX_SY_REF_IS_INITIAL exception when execute_substep runs without execute being called
- All AC met: AC-01 (mo_log in init), AC-02 (no exception in bgRFC path), AC-03 (execute reuses same instance)

### Changed Files
| File (remote repo) | Commit |
|--------------------|--------|
| src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap | 16f0540 |
