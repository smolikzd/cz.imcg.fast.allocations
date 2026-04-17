---
title: 'EST-197: Add delete() action to dashboard'
type: 'feature'
created: '2026-04-17'
status: 'in-review'
baseline_commit: '092ff4e737c7b6be08109f56031fd709759a80ef'
context:
  - '_bmad-output/implementation-artifacts/spec-est-150-151-152-dashboard-filter-ux-and-action-confirmations.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The allocation dashboard has no way to permanently remove a finished or cancelled process instance. Users must leave dead records cluttering the list forever, with no clean-up option from the UI.

**Approach:** Add an instance action `DeleteRecord` to `ZFI_I_ALLOC_DASHBOARD_CE` gated by feature control (enabled only for terminal statuses: `COMPLETED`, `SUPERSEDED`, `CANCELLED`). The handler buffers a `'DELETE'` token; the saver deletes rows from `ZFI_ALLOC_STATE`, `ZFI_PROC_STEP`, and `ZFI_PROC_INST` in dependency order, without committing. A `Common.IsActionCritical` annotation triggers a Fiori Elements confirmation dialog.

## Boundaries & Constraints

**Always:**
- Line length ≤ 120 chars in all ABAP/CDS files
- SAP naming conventions — no new DDIC objects needed; `'DELETE'` fits the existing `ACTION` field (CHAR 10) in `ZFI_ALLOC_S_ACTION_BUFFER`
- Delete order: `ZFI_ALLOC_STATE` → `ZFI_PROC_STEP` → `ZFI_PROC_INST` (child before parent)
- Feature control must guard the action: enabled only when `ProcessStatus` ∈ { `COMPLETED`, `SUPERSEDED`, `CANCELLED` }
- Both annotation files must be updated: the WAPA-deployed XML and the local dev XML
- `iv_no_commit = abap_true` pattern used in saver (consistent with all other actions)

**Ask First:**
- If the BDEF action name `DeleteRecord` (external `deleteRecord`) conflicts with any standard RAP delete operation or causes activation errors, HALT and ask Zdenek before renaming
- If `cleanup_old_instances` in `zcl_fi_process_manager` accepts a single `instance_id` parameter and can replace the inline DELETE statements, ask Zdenek whether to reuse it or inline the DELETEs

**Never:**
- Do not add a standard RAP `delete;` operation (that is managed RAP syntax — this is unmanaged)
- Do not delete `ZFI_ALLOC_STATE` rows for other allocation keys sharing the same `PROCESS_INSTANCE_ID` — scope is exactly the key from the action's import parameter
- Do not commit in the handler (`CALL METHOD ... COMMIT WORK` is saver's job)
- Do not enable delete for `FAILED`, `RUNNING`, `QUEUED`, `PENDING`, `NEW`, or any non-terminal status

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Delete COMPLETED | Instance with status `COMPLETED` | Confirmation dialog → on confirm: row removed from list, `ZFI_ALLOC_STATE` + `ZFI_PROC_STEP` + `ZFI_PROC_INST` deleted | If DELETE fails, raise `ZCX_FI_PROCESS_ERROR`; row stays |
| Delete CANCELLED | Instance with status `CANCELLED` | Same as above | Same |
| Delete SUPERSEDED | Instance with status `SUPERSEDED` | Same as above | Same |
| Delete not allowed | Instance with status `RUNNING`, `FAILED`, etc. | Delete button hidden / greyed out (feature control disables it) | N/A — button not reachable |
| Confirmation cancelled | User clicks Delete → then cancels dialog | No action taken, instance remains | N/A |

</frozen-after-approval>

## Code Map

- `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.bdef.asbdef` — BDEF for root entity; add `DeleteRecord` instance action with feature control
- `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` — root custom entity CDS; add `@UI.lineItem` and `@UI.identification` entries for the delete action button
- `src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` — behavior pool local classes; add `deleterecord` method to `lhc_dashboard`, extend `get_instance_features`, extend saver `WHEN 'DELETE'` block
- `src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.abap` — shell class; add method declaration for `deleterecord` in `lhc_dashboard` (if needed by abapgit format)
- `dashboard-ui/webapp/annotations/annotation.xml` — local dev annotation file; add `Common.IsActionCritical` for `deleteRecord`
- `src/zfi_alloc_process/zfi_alloc_db.wapa.annotations_-annotation.xml` — deployed WAPA annotation file; same change

## Tasks & Acceptance

**Execution:**

- [x] `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.bdef.asbdef` — Add `action ( features : instance ) DeleteRecord external 'deleteRecord';` after the existing instance actions; add `side effects { action DeleteRecord affects field *; }` block
- [x] `src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` — In `lhc_dashboard`: (1) add `METHODS deleterecord FOR MODIFY IMPORTING keys FOR ACTION ZFI_I_ALLOC_DASHBOARD_CE~DeleteRecord RESULT et_result.`; (2) in `get_instance_features` set `%action-DeleteRecord = if_abap_behv=>fc-o-disabled` unless status ∈ {COMPLETED, SUPERSEDED, CANCELLED}; (3) implement `deleterecord` to validate status and append `action = 'DELETE'` token to `gt_action_buffer`; (4) in `lsc_dashboard=>save()` add `WHEN 'DELETE':` block that deletes `ZFI_ALLOC_STATE`, then `ZFI_PROC_STEP`, then `ZFI_PROC_INST` using the buffered keys, respecting `iv_no_commit = abap_true`
- [x] `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` — Add `{ type: #FOR_ACTION, dataAction: 'deleteRecord', label: 'Delete' }` to `@UI.lineItem` annotation array and `@UI.identification` annotation array
- [x] `dashboard-ui/webapp/annotations/annotation.xml` — Add `<Annotations Target="SAP__self.deleteRecord(SAP__self.ZFI_I_ALLOC_DASHBOARD_CEType)"><Annotation Term="Common.IsActionCritical" Bool="true" /></Annotations>`
- [x] `src/zfi_alloc_process/zfi_alloc_db.wapa.annotations_-annotation.xml` — Same `Common.IsActionCritical` block as above (keep both annotation files in sync)

**Acceptance Criteria:**

- Given a process instance with status `COMPLETED`, `SUPERSEDED`, or `CANCELLED`, when the user opens the dashboard and selects that row, then a **Delete** button is visible and enabled in the toolbar
- Given the user clicks **Delete**, when the Fiori Elements confirmation dialog appears and the user confirms, then the instance row disappears from the list and the underlying `ZFI_ALLOC_STATE`, `ZFI_PROC_STEP`, and `ZFI_PROC_INST` records are removed from the database
- Given a process instance with any non-terminal status (e.g., `RUNNING`, `FAILED`, `QUEUED`), when the user selects that row, then the Delete button is absent or greyed out (feature control enforces this)
- Given the user clicks Delete but then cancels the confirmation dialog, then no data is deleted and the row remains

## Design Notes

**Action buffer pattern** — all existing actions use `gt_action_buffer` (type `zfi_alloc_tt_action_buffer`) to pass intent from the handler to `lsc_dashboard=>save()`. The buffer row structure (`ZFI_ALLOC_S_ACTION_BUFFER`) contains the allocation composite key plus an `ACTION` CHAR(10) field. `'DELETE'` fits within 10 chars. Follow the exact same pattern — no new structures required.

**Delete order is non-negotiable** — `ZFI_PROC_STEP` has a FK to `ZFI_PROC_INST`. Delete steps first, then instance header. Also delete `ZFI_ALLOC_STATE` first (it only holds the link). Example from `createandexecute` saver shows the pattern for `ZFI_ALLOC_STATE` DELETE by composite key:
```abap
DELETE FROM zfi_alloc_state
  WHERE company_code  = @ls_buf-company_code
    AND fiscal_year   = @ls_buf-fiscal_year
    AND fiscal_period = @ls_buf-fiscal_period
    AND allocation_id = @ls_buf-allocation_id.
```

## Verification

**Manual checks (no CLI for ABAP):**
- Activate BDEF → confirm no activation errors in SE80 / ADT
- Open dashboard in browser, select a COMPLETED row → Delete button visible
- Select a RUNNING row → Delete button absent/greyed
- Click Delete on COMPLETED row → confirmation dialog appears
- Confirm deletion → row disappears, verify DB via SE16 that `ZFI_ALLOC_STATE`, `ZFI_PROC_STEP`, `ZFI_PROC_INST` rows are gone
