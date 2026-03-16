# Story: Remove Dead CONTEXT_DATA Column from ZFI_PROC_INST

Status: done

## Story

As a framework developer,
I want to remove the unused CONTEXT_DATA column from the ZFI_PROC_INST DDIC table,
so that the schema accurately reflects only the data that is actually written and read by the framework.

## Acceptance Criteria

1. The `CONTEXT_DATA` field DD03P block is removed from `zfi_proc_inst.tabl.xml`.
2. The matching `context_data TYPE zfi_process_context` field is removed from the local `ty_process_inst` struct in `zfi_setup_demo_data.prog.abap`.
3. No other framework code references `CONTEXT_DATA` on `ZFI_PROC_INST` (confirmed by codebase search).
4. The following active uses of `context_data` elsewhere in the codebase are NOT touched:
   - `CONTEXT_DATA` field on `ZFI_PROC_STEP` (per-substep JSON payload — active)
   - `CONTEXT_DATA_STRUCTURE` field on `ZFI_PROC_DEF` (DDIC struct name for deserialization — active)
   - `context_data` field in `zif_fi_process_step=>ty_planned_step` (step sets substep payload — active)

## Tasks / Subtasks

- [x] Task 1: Remove DD03P block from DDIC table XML (AC: #1)
  - [x] Delete lines 91–95 of `src/zfi_proc_inst.tabl.xml` (the CONTEXT_DATA DD03P block)
- [x] Task 2: Remove matching field from demo data program (AC: #2)
  - [x] `zfi_setup_demo_data.prog.abap` was removed entirely (commit `3087aad`) — superseded by `ZFI_SETUP_TEST_DATA`. AC #2 is moot.
- [x] Task 3: Verify no other references to CONTEXT_DATA on ZFI_PROC_INST (AC: #3)
  - [x] Grep confirms: only `ZFI_PROC_STEP` and `ZFI_PROC_DEF` reference `CONTEXT_DATA`/`CONTEXT_DATA_STRUCTURE` — neither is `ZFI_PROC_INST`

## Dev Notes

- `CONTEXT_DATA` was declared in the DDIC table but is never populated by any INSERT/UPDATE in framework
  code and is never read by any SELECT. Confirmed by full codebase search in prior session.
- The data element `ZFI_PROCESS_CONTEXT` itself (the ROLLNAME) is NOT removed — it may still be used
  by `CONTEXT_DATA` on `ZFI_PROC_STEP` or `CONTEXT_DATA_STRUCTURE` on `ZFI_PROC_DEF`.
- Constitution Principle I (DDIC-First): removal of a dead column improves schema accuracy.
- Constitution Principle III: mcp-sap-docs consulted — no SAP-specific concerns for removing an unused
  DDIC column via abapGit XML.
- This change requires a DDIC activation in the target SAP system after transport.

### References

- `src/zfi_proc_inst.tabl.xml` lines 91–95 — dead CONTEXT_DATA DD03P block
- `src/zfi_setup_demo_data.prog.abap` line 37 — matching local struct field
- `src/zfi_proc_step.tabl.xml` — unrelated CONTEXT_DATA (substep payload, keep)
- `src/zfi_proc_def.tabl.xml` — unrelated CONTEXT_DATA_STRUCTURE (keep)
- `src/zif_fi_process_step.intf.abap` — unrelated `context_data` in `ty_planned_step` (keep)

## Dev Agent Record

### Agent Model Used

github-copilot/claude-sonnet-4.6

### Completion Notes List

Implemented in planner repo prior to current session. Committed at:
- `95e5b0f` — removed CONTEXT_DATA DD03P block from `zfi_proc_inst.tabl.xml`
- `3087aad` — `zfi_setup_demo_data.prog.abap` removed entirely (AC#2 moot)
- Verified by grep: `CONTEXT_DATA` appears only in `ZFI_PROC_STEP` and `ZFI_PROC_DEF` — not `ZFI_PROC_INST`

### File List

- `src/zfi_proc_inst.tabl.xml`
- `src/zfi_setup_demo_data.prog.abap`
