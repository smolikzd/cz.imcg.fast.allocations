---
story: "3.3"
title: "Fix PHASE2 CATCH block fall-through"
epic: "3 — Functional Correctness — PHASE2 Substep Path"
sprint: 2
status: done
completed: 2026-03-09
priority: moderate
estimate: 1h
source_findings:
  - zfi_alloc_phase_2-migration-review.md
  - FINDING-04
target_repo: smolikzd/cz.imcg.fast.planner
dependencies:
  - None (independent)
---

# Story 3.3: Fix PHASE2 CATCH block fall-through

## User Story

As a PHASE2 process monitor user,
I want exceptions during substep execution to be correctly logged and surfaced,
so that failures are visible in the process monitor rather than silently swallowed.

## Background

`ZCL_FI_ALLOC_STEP_PHASE2->execute_substep` contains CATCH blocks where:
- The exception is caught
- But `rs_result-success` is NOT set to `abap_false`
- And/or no error is written to `mo_log`

This causes silent failure: the framework sees a successful result even when an exception
was caught, so the substep is marked as succeeded and processing continues incorrectly.

Additionally, some CATCH blocks may silently discard `ZCX_FI_PROCESS_ERROR` without
re-raising or propagating the message.

## Acceptance Criteria

**AC-01 — Caught exceptions set success = false:**
Given an exception is caught in any CATCH block in `execute_substep`
When the CATCH block runs
Then `rs_result-success = abap_false` is set
And `rs_result-can_continue = abap_false` is set (unless partial continuation is intentional)

**AC-02 — Exception text captured in result message:**
Given an exception is caught
When the CATCH block runs
Then `rs_result-message` contains the exception text (via `lx->get_text( )` or
`MESSAGE ... INTO lv_dummy` with `lv_dummy TYPE string`)

**AC-03 — Exception written to log:**
Given `mo_log IS BOUND` and an exception is caught
When the CATCH block runs
Then the exception detail is written to `mo_log`

**AC-04 — ZCX_FI_PROCESS_ERROR not silently discarded:**
Given a `ZCX_FI_PROCESS_ERROR` is raised inside `execute_substep`
When it is caught
Then its message is propagated via `rs_result-message`
And it is NOT re-raised as an unhandled exception (return via `rs_result` instead)

**AC-05 — No fall-through sets success after exception:**
Given a CATCH block executes
When execution continues after the CATCH
Then no subsequent code sets `rs_result-success = abap_true` unconditionally

## Tasks

- [ ] **Task 3.3.1**: Read `ZCL_FI_ALLOC_STEP_PHASE2->execute` and `execute_substep`
  in full. List every CATCH block found.

- [ ] **Task 3.3.2**: For each CATCH block:
  - [ ] Check whether `rs_result-success = abap_false` is set — add if missing
  - [ ] Check whether `rs_result-message` is populated — add if missing
    (use `rs_result-message = lx->get_text( ).` or
    `MESSAGE ... INTO DATA(lv_msg). rs_result-message = lv_msg.`)
  - [ ] Check whether the exception is written to `mo_log` — add if missing and `mo_log IS BOUND`
  - [ ] Verify no fall-through logic sets `success = abap_true` after the CATCH block

- [ ] **Task 3.3.3**: Verify `lv_dummy` is `TYPE string` in all files touched (per Story 5.3,
  already applied — confirm it is `TYPE string` not `TYPE c` before using it for message capture).

- [ ] **Task 3.3.4**: Verify the overall TRY-CATCH-ENDTRY structure does not have any
  fall-through path that sets `rs_result-success = abap_true` after a CATCH block ran.

## Dev Notes

- **`lv_dummy TYPE string`**: Story 5.3 already changed `lv_dummy` to `TYPE string` in
  PHASE2. Confirm this is the case before this story; if not, change it here.
- **`mo_log` availability**: after Story 3.1, `mo_log` is initialised in `init`.
  Use `IF mo_log IS BOUND.` guard before calling `mo_log->...`.
- **Do NOT re-raise**: `execute_substep` has signature `RAISING cx_static_check`. It is
  better to surface the error via `rs_result` than to re-raise — the framework inspects
  `rs_result-success` to determine substep outcome.
- **Constitution Principle V**: errors must not be swallowed; use `ZCX_FI_PROCESS_ERROR`
  for framework-level errors.
- Reference: Phase 2 FINDING-04.

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` | Fix all CATCH blocks in `execute` and `execute_substep` |

## Constitution Compliance Checklist

- [ ] Principle I — DDIC-First: N/A
- [ ] Principle II — SAP Standards: line ≤ 120 chars
- [ ] Principle III — Consult SAP Docs: N/A
- [ ] Principle IV — Factory Pattern: N/A
- [ ] Principle V — Error Handling: all exceptions surface via `rs_result`; none silently discarded

## Dev Agent Record

### Agent Model Used
claude-sonnet-4.5 (github-copilot/claude-sonnet-4.5)

### Completion Notes
- Analyzed all CATCH blocks in execute_substep: cx_shdb_exception, cx_root, cx_uuid_error - all already properly handled
- Found critical bug: two UPDATE zfi_alloc_bche error handlers (lines 575-580, 586-591) logged errors but did NOT set rs_result-success=false
- Fixed both UPDATE error handlers to set success=false, message, and can_continue=false
- Fixed compilation error: replaced undefined mv_process_instance_id/mv_step_number variables with ls_phase2_context-key_id in success message
- Verified no fall-through: final IF at line 599 checks success=true before setting success message
- Confirmed lv_dummy is TYPE string (Story 5.3 already applied)
- All AC met: AC-01 (errors set success=false), AC-02 (message captured), AC-03 (logged), AC-05 (no fall-through)

### Changed Files
| File (remote repo) | Commit |
|--------------------|--------|
| src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap | 4d103ce |
