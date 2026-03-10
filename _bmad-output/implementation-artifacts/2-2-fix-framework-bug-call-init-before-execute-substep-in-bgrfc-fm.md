---
story: "2.2"
title: "Fix framework bug — call init before execute_substep in bgRFC FM"
epic: "2 — Fix LUW and Rollback Contract"
sprint: 1
status: done
priority: critical
estimate: 1.5h
source_findings:
  - CC-16
  - LUW-02
  - luw-schema-analysis.md
  - zfi_alloc_phase_2-migration-review.md
target_repo: smolikzd/cz.imcg.fast.planner
dependencies:
  - None (can be done independently; Story 3.1 depends on this being done first)
---

# Story 2.2: Fix framework bug — call `init` before `execute_substep` in bgRFC FM

## User Story

As a PHASE2 substep executor,
I want `init` to be called before `execute_substep` in the bgRFC function module,
so that all instance variables and the log object are populated when a substep runs
in queued mode.

## Background

`ZFI_BGRFC_EXEC_SUBSTEP` is the bgRFC function module that executes a single
PHASE2 substep in queued mode. Current code:

```abap
CREATE OBJECT lo_step TYPE (lv_class_name).
lo_step->execute_substep( lv_substep_number ).
```

`init` is never called. When `execute_substep` runs:
- `mv_allocation_id`, `mv_company_code`, `mv_fiscal_year`, `mv_fiscal_period` are all initial
- `mo_log` is `NULL` → any log write raises `CX_SY_REF_IS_INITIAL` dump

This makes the entire PHASE2 queued (bgRFC) substep path completely non-functional.

The fix adds `lo_step->init( ls_context )` between `CREATE OBJECT` and
`execute_substep`, with proper error handling.

## Acceptance Criteria

**AC-01 — `init` called before `execute_substep`:**
Given a PHASE2 substep is queued and executed by the bgRFC infrastructure
When `ZFI_BGRFC_EXEC_SUBSTEP` runs
Then `lo_step->init( ls_context )` is called before `lo_step->execute_substep( ... )`

**AC-02 — Instance variables populated:**
Given `init` is called with a properly constructed context containing the process instance
When `execute_substep` runs
Then `mv_allocation_id`, `mv_company_code`, `mv_fiscal_year`, `mv_fiscal_period`
are populated with values from the process instance parameters

**AC-03 — No CX_SY_REF_IS_INITIAL dump from mo_log (after Story 3.1):**
Given `init` has been called and Story 3.1 has moved `mo_log` init to `init`
When a log write occurs inside `execute_substep`
Then no `CX_SY_REF_IS_INITIAL` exception is raised

**AC-04 — `init` failure handled:**
Given `init` raises `ZCX_FI_PROCESS_ERROR` (e.g., missing parameters)
When the CATCH block executes
Then the substep status is set to FAILED and the FM returns cleanly
(no unhandled exception propagates out of the bgRFC FM)

## Tasks

- [ ] **Task 2.2.1**: Read `ZFI_BGRFC_EXEC_SUBSTEP` in full to understand current structure
  and identify how `lv_instance_id` is obtained and how `lo_step` is created.

- [ ] **Task 2.2.2**: Determine how to retrieve or construct `ZCL_FI_PROCESS_INSTANCE`
  - [ ] Search framework code for the factory/retrieval method used elsewhere
    (e.g., `ZCL_FI_PROCESS_INSTANCE=>get_instance( iv_instance_id = lv_instance_id )`)
  - [ ] Consult mcp-sap-docs for the correct pattern (constitution Principle III)

- [ ] **Task 2.2.3**: Build the context structure:
  ```abap
  DATA ls_context TYPE zif_fi_process_step=>ts_context.
  ls_context-io_process_instance = lo_process_instance.
  " populate other context fields if required by ts_context definition
  ```

- [ ] **Task 2.2.4**: Insert `init` call with error handling:
  ```abap
  TRY.
    lo_step->init( ls_context ).
  CATCH zcx_fi_process_error INTO DATA(lx_init).
    " set substep status to FAILED
    lo_step->set_substep_status(
      iv_substep_number = lv_substep_number
      iv_status         = zif_fi_process_step=>gc_status-failed
      iv_do_commit      = abap_false
      iv_message        = lx_init->get_text( ) ).
    RETURN.
  ENDTRY.
  ```
  Adjust to actual API — verify `set_substep_status` signature in the framework.

- [ ] **Task 2.2.5**: Verify the FM does NOT issue `COMMIT WORK` anywhere — qRFC
  infrastructure commits the LUW on FM return.

## Dev Notes

- **Constitution Principle III**: consult mcp-sap-docs for `ZCL_FI_PROCESS_INSTANCE`
  factory retrieval pattern before adding instance retrieval. Do not guess.
- **Constitution Principle V**: `init` failure must not be silently swallowed — log and
  set substep FAILED.
- **bgRFC LUW rule**: The bgRFC FM must NOT issue `COMMIT WORK`. The qRFC infrastructure
  commits the LUW when the FM returns. All `set_substep_status` calls must use
  `iv_do_commit = abap_false`.
- **File location**: This is a framework file in the **local** repo at
  `src/zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap` — not in the remote alloc repo.
- Story 3.1 (`mo_log` move) depends on this story being done first for the bgRFC path
  to work end-to-end.
- Reference: CC-16, LUW-02, `luw-schema-analysis.md` §6.

## Files to Change

| File (local repo: /Users/smolik/DEV/cz.imcg.fast.planner) | Change |
|--------------------------------------------------|--------|
| `src/zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap` | Add `init` call between `CREATE OBJECT` and `execute_substep` |

## Constitution Compliance Checklist

- [ ] Principle I — DDIC-First: no new local TYPE definitions (use `zif_fi_process_step=>ts_context`)
- [ ] Principle II — SAP Standards: line ≤ 120 chars
- [ ] Principle III — Consult SAP Docs: instance retrieval pattern verified
- [ ] Principle IV — Factory Pattern: use existing factory method for instance retrieval
- [ ] Principle V — Error Handling: `init` failure caught, substep set to FAILED

## Dev Agent Record

### Agent Model Used
github-copilot/claude-sonnet-4.6

### Completion Notes
Added `lo_step->init( ls_ctx )` call between the context assembly block and the
`execute_substep` call. The context `ls_ctx` was already fully populated at that
point (instance_id, process_type, parameters, io_process_instance, io_step reference,
substep_data), so no additional context construction was needed. No COMMIT WORK
added — all set_substep_status calls retain `iv_do_commit = abap_false`.

### Changed Files
| File (local repo) | Commit |
|-------------------|--------|
| `src/zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap` | `e05bf00` |
