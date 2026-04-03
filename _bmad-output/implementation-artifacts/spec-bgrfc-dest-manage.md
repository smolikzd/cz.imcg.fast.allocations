---
title: 'bgRFC Destination Suspend/Resume Utility'
type: 'feature'
created: '2026-04-02'
status: 'done'
baseline_commit: 'cceae34'
context:
  - 'learnings/TPOOL_FORMAT.md'
  - 'learnings/PROJECT_CONVENTIONS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** There is no simple way to programmatically suspend and resume bgRFC inbound destination processing. The SBGRFCMON transaction provides this capability interactively but SAP exposes no public API for automation. The ZFI_PROCESS framework dispatches substeps via bgRFC, and operations teams need a scriptable tool to pause/resume processing during maintenance windows or incident handling.

**Approach:** Create a standalone executable program `ZFI_PROCESS_BGRFC_DEST_MANAGE` in `cz.imcg.fast.planner` with a selection screen accepting a bgRFC destination name and a suspend/resume checkbox. The program validates the destination exists via `CL_BGRFC_DESTINATION_INBOUND=>CREATE()`, then calls SAP-internal bgRFC scheduler function modules to toggle processing state. No direct database writes — only FM-based API calls.

## Boundaries & Constraints

**Always:**
- Validate destination exists before attempting state change (catch `CX_BGRFC_INVALID_DESTINATION`)
- Write success/failure messages via `MESSAGE` statement (no WRITE)
- Follow constitution: DDIC types only (use `bgrfc_dest_name_inbound` for dest param), line length ≤120 chars
- Two files: `.prog.abap` + `.prog.xml` (abapGit format, TPOOL per `learnings/TPOOL_FORMAT.md`)
- Target repository: `cz.imcg.fast.planner`, directory `src/`

**Ask First:**
- If no SAP FM for destination suspend/resume can be confirmed at implementation time, should we abort with an informational message or try an alternative approach?

**Never:**
- Do not modify bgRFC destination configuration (SBGRFCCONF settings) — only toggle the scheduler processing state
- Do not delete or modify existing bgRFC units/queues
- Do not use CALL TRANSACTION to invoke SBGRFCMON
- Do not perform direct database writes (no INSERT/UPDATE/DELETE on bgRFC tables)

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Suspend valid dest | p_dest='ZFI_PROCESS', p_susp=X | Processing suspended, success message | N/A |
| Resume valid dest | p_dest='ZFI_PROCESS', p_susp='' | Processing resumed, success message | N/A |
| Invalid destination | p_dest='NONEXIST', any | Error message, program stops | CX_BGRFC_INVALID_DESTINATION caught, MESSAGE TYPE 'E' |
| Already suspended | p_dest (already suspended), p_susp=X | Idempotent — success message (no error) | N/A |
| Already active | p_dest (already active), p_susp='' | Idempotent — success message (no error) | N/A |
| Empty destination | p_dest='', any | Mandatory field error on selection screen | OBLIGATORY attribute on parameter |

</frozen-after-approval>

## Code Map

- `src/zfi_process_bgrfc_dest_manage.prog.abap` -- New program: selection screen + suspend/resume logic
- `src/zfi_process_bgrfc_dest_manage.prog.xml` -- abapGit metadata + TPOOL (report title, selection texts)
- `src/zcl_fi_process_instance.clas.abap:576-586` -- Reference: existing bgRFC destination validation pattern
- `src/zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap` -- Reference: existing bgRFC FM in project
- `learnings/examples/prog_xml_example.xml` -- Reference: XML structure template

## Tasks & Acceptance

**Execution:**
- [x] `src/zfi_process_bgrfc_dest_manage.prog.abap` -- Created: selection screen + suspend/resume via CL_BGRFC_MONITOR_API / IF_BGRFC_MONITOR_INBOUND
- [x] `src/zfi_process_bgrfc_dest_manage.prog.xml` -- Created: abapGit XML with PROGDIR, TPOOL (R: report title, S: P_DEST + P_SUSP selection texts)
- [x] `src/zfi_process.msag.xml` -- Updated: added messages 100-107 for bgRFC destination management

**Acceptance Criteria:**
- Given a valid bgRFC destination and p_susp = X, when program runs, then destination processing is suspended and success message displayed
- Given a valid bgRFC destination and p_susp = '', when program runs, then destination processing is resumed and success message displayed
- Given an invalid destination name, when program runs, then an error message is displayed and no state change occurs
- Program compiles without errors and follows abapGit format conventions

## Design Notes

SAP provides no released public API for bgRFC destination suspend/resume. Research findings:
1. `CL_BGRFC_DESTINATION_INBOUND` — exists for creating destination objects, but has no suspend/resume methods
2. `IF_BGRFC_UNIT` — has LOCK/DELAY for unit-level locking only
3. `CL_QRFC_CLIENT_INBOUND` — has `get_unit_state`/`get_queue_state` for monitoring but no destination-level control
4. Community (SAP Q&A #14073594) confirms destination locking from SBGRFCMON is the only known approach

**Final implementation approach (discovered via SE24 inspection):**

SAP-internal OO monitor API — the same API that SBGRFCMON uses:
1. `CL_BGRFC_MONITOR_API=>CREATE_BGRFC_MONITOR_INBOUND( )` — factory, returns `IF_BGRFC_MONITOR_INBOUND`
2. `IF_BGRFC_MONITOR_INBOUND~GET_DESTINATION_STATE( dest_name )` — returns `BGRFC_DESTINATION_STATE`
3. `IF_BGRFC_MONITOR_INBOUND~LOCK_DESTINATION( dest_name, commit_changes )` — suspends processing
4. `IF_BGRFC_MONITOR_INBOUND~UNLOCK_DESTINATION( dest_name, commit_changes )` — resumes processing

State constants from `IF_BGRFC_MONITOR`:
- `DESTINATION_STATE_NORMAL` = 1605195714 (active)
- `DESTINATION_STATE_LOCKED` = 1605195715 (suspended)
- `DESTINATION_STATE_ERRONEOUS` = 1605195716 (error state)

Exception classes: `CX_BGRFC_INVALID_DESTINATION`, `CX_BGRFC_MONITOR_API_AUTHORITY`, `CX_BGRFC_MONITOR_API`

Messages added to ZFI_PROCESS message class: 100-107 (bgRFC destination management range).

Since this program targets on-premise SAP 7.58, using internal classes is acceptable — this is an admin utility, not a business process integration point.

## Verification

**Manual checks:**
- Program source compiles in SE38 / ADT without errors
- `.prog.xml` imports cleanly via abapGit
- Selection screen renders with correct field labels
- Running with valid destination + checkbox toggled produces expected state change visible in SBGRFCMON

## Spec Change Log

## Suggested Review Order

**Core logic — suspend/resume via SAP-internal monitor API**

- Entry point: selection screen and orchestration flow
  [`zfi_process_bgrfc_dest_manage.prog.abap:19`](../../repos/../../../cz.imcg.fast.planner/src/zfi_process_bgrfc_dest_manage.prog.abap#L19)

- Destination validation before any state change
  [`zfi_process_bgrfc_dest_manage.prog.abap:27`](../../repos/../../../cz.imcg.fast.planner/src/zfi_process_bgrfc_dest_manage.prog.abap#L27)

- State read for idempotent behavior
  [`zfi_process_bgrfc_dest_manage.prog.abap:41`](../../repos/../../../cz.imcg.fast.planner/src/zfi_process_bgrfc_dest_manage.prog.abap#L41)

- Suspend: idempotent check + lock_destination call
  [`zfi_process_bgrfc_dest_manage.prog.abap:65`](../../repos/../../../cz.imcg.fast.planner/src/zfi_process_bgrfc_dest_manage.prog.abap#L65)

- Resume: idempotent check + unlock_destination call
  [`zfi_process_bgrfc_dest_manage.prog.abap:98`](../../repos/../../../cz.imcg.fast.planner/src/zfi_process_bgrfc_dest_manage.prog.abap#L98)

**Messages and metadata**

- New messages 100-107 for destination management
  [`zfi_process.msag.xml:437`](../../repos/../../../cz.imcg.fast.planner/src/zfi_process.msag.xml#L437)

- Program XML: PROGDIR + TPOOL entries
  [`zfi_process_bgrfc_dest_manage.prog.xml:1`](../../repos/../../../cz.imcg.fast.planner/src/zfi_process_bgrfc_dest_manage.prog.xml#L1)

