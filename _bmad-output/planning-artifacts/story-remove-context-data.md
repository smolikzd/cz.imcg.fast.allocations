# Story: Remove Dead CONTEXT_DATA Column from ZFI_PROC_INST

Status: ready-for-dev

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

- [ ] Task 1: Remove DD03P block from DDIC table XML (AC: #1)
  - [ ] Delete lines 91–95 of `src/zfi_proc_inst.tabl.xml` (the CONTEXT_DATA DD03P block)
- [ ] Task 2: Remove matching field from demo data program (AC: #2)
  - [ ] Delete line 37 of `src/zfi_setup_demo_data.prog.abap` (`context_data TYPE zfi_process_context`)
- [ ] Task 3: Verify no other references to CONTEXT_DATA on ZFI_PROC_INST (AC: #3)
  - [ ] Grep codebase for `context_data` and confirm no reads/writes target ZFI_PROC_INST

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

### File List

- `src/zfi_proc_inst.tabl.xml`
- `src/zfi_setup_demo_data.prog.abap`
