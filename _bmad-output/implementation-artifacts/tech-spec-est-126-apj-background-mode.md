---
title: 'Add Background Processing Mode to Allocation Reports'
slug: 'est-126-apj-background-mode'
created: '2026-03-24'
status: 'implemented'
stepsCompleted: [1, 2, 3, 4, 5]
tech_stack: ['ABAP 7.58', 'ZFI_PROCESS framework', 'APJ (Application Jobs)', 'abapGit']
files_to_modify:
  - 'src/zfi_alloc_process/zfi_alloc_proc_exec.prog.abap'
  - 'src/zfi_alloc_process/zfi_alloc_proc_export.prog.abap'
code_patterns: ['AS CHECKBOX DEFAULT X', 'lo_manager->request_execute_process() for APJ', 'lo_manager->execute_process() for online', 'SWITCH for human-readable status', 'Separate TRY/CATCH for reload']
test_patterns: ['Manual test via SE38 — run with p_bgrnd on/off', 'Verify APJ job in Fiori Application Jobs app', 'Check zfi_proc_inst for JOB_NAME/JOB_COUNT']
linear_issue: 'EST-126'
---

# Tech-Spec: Add Background Processing Mode to Allocation Reports

**Created:** 2026-03-24
**Linear Issue:** [EST-126](https://linear.app/smolikzd/issue/EST-126/add-background-processing-mode-apj-to-allocation-execution-reports)

## Overview

### Problem Statement

Both `ZFI_ALLOC_PROC_EXEC` and `ZFI_ALLOC_PROC_EXPORT` execute process instances synchronously via `lo_manager->execute_process()`. This blocks the user's SAP GUI session for the entire duration of the allocation run (which can take minutes for large datasets with PHASE2 queue processing). The ZFI_PROCESS framework now supports APJ background scheduling (implemented in EST-121), but the allocation reports don't use it yet.

### Solution

Add a selection screen checkbox `p_bgrnd` (`AS CHECKBOX DEFAULT 'X'`) to both reports. After `create_process()`, branch on this parameter:
- **Background** (`p_bgrnd = abap_true`): Call `lo_manager->request_execute_process()` which schedules an APJ job and returns immediately. Display job name/count and instance ID.
- **Online** (`p_bgrnd = abap_false`): Call `lo_manager->execute_process()` as before (existing behavior, unchanged).

### Scope

**In Scope:**
- Add `p_bgrnd` parameter to `ZFI_ALLOC_PROC_EXEC`
- Add `p_bgrnd` parameter to `ZFI_ALLOC_PROC_EXPORT`
- Branch execution logic based on `p_bgrnd` value
- Output appropriate status messages for both modes
- Reload instance after APJ scheduling to display JOB_NAME/JOB_COUNT

**Out of Scope:**
- No DDIC changes (no new structures, domains, data elements)
- No framework changes (APJ API already exists from EST-121)
- No new step classes
- No APJ catalog/template changes (already deployed in EST-121)
- No selection text maintenance (maintained in SAP via SE38/SE63)

## Context for Development

### Codebase Patterns

Both reports follow an identical pattern:
1. `PARAMETERS` block for selection screen
2. Local class `lcl_process_actions` with `setup_process_data` method for process type registration and step definition
3. `START-OF-SELECTION` block that:
   - Builds parameter structures
   - Gets manager singleton via `zcl_fi_process_manager=>get_instance()`
   - Calls `lcl_process_actions=>setup_process_data()`
   - Creates instance via `lo_manager->create_process()`
   - Executes via `lo_manager->execute_process()`

The change only affects the final execution call — replacing it with a conditional branch.

### Framework APJ API (from EST-121)

The manager provides two execution methods:
- `execute_process( iv_instance_id )` — synchronous, blocks caller
- `request_execute_process( iv_instance_id )` — schedules APJ job (2s delay), sets status to `EXECREQ`, writes JOB_NAME/JOB_COUNT to `zfi_proc_inst`, returns immediately

After `request_execute_process()`, the instance can be reloaded to read:
- `ls_instance-status` → `'EXECREQ'`
- `ls_instance-job_name` → APJ job name
- `ls_instance-job_count` → APJ job count

The APJ job class (`zcl_fi_process_job`) picks up the instance, verifies EXECREQ status, and calls `execute()` in the background.

### Files to Reference

| File | Purpose |
| ---- | ------- |
| `src/zfi_alloc_process/zfi_alloc_proc_exec.prog.abap` | Allocation execution report (179 lines) — **MODIFY** |
| `src/zfi_alloc_process/zfi_alloc_proc_export.prog.abap` | Export report (116 lines) — **MODIFY** |
| `zcl_fi_process_manager.clas.abap` (planner repo) | Manager API — `request_execute_process()` reference |
| `zcl_fi_process_instance.clas.abap` (planner repo) | Instance — `get_instance()` returns `zfi_proc_inst` structure |

### Technical Decisions

1. **Parameter naming**: `p_bgrnd` — short, follows SAP convention, fits 8-char ABAP parameter name limit
2. **Default value**: `abap_true` — background mode is the preferred default; online is the fallback
3. **Parameter placement**: After the existing `p_export` parameter (in EXEC) / after `p_del_pc` (in EXPORT), with a comment line separator for clarity
4. **Output after APJ scheduling**: Reload instance via `lo_manager->load_process()` to get JOB_NAME/JOB_COUNT from DB, display them via `WRITE` statements
5. **No process type registration changes needed**: APJ scheduling works with any process type — no `bgrfc_dest_name_inbound` needed for APJ (that's bgRFC only)

## Implementation Plan

### Tasks

- [x] Task 1: Add background processing parameter to `ZFI_ALLOC_PROC_EXEC`
  - File: `src/zfi_alloc_process/zfi_alloc_proc_exec.prog.abap`
  - Action: Add `PARAMETERS: p_bgrnd AS CHECKBOX DEFAULT 'X'.` after the `p_export` parameter line, with a comment separator
  - Notes: Uses `AS CHECKBOX` (not `TYPE abap_bool`) so SAP renders a proper checkbox on the selection screen. Comment: `" Background processing (APJ)`

- [x] Task 2: Replace direct `execute_process` with conditional branch in `ZFI_ALLOC_PROC_EXEC`
  - File: `src/zfi_alloc_process/zfi_alloc_proc_exec.prog.abap`
  - Action: Extract instance ID into `lv_id`, change "will be started" to "created", replace `execute_process()` call with:
    ```abap
    IF p_bgrnd = abap_true.
      " Schedule background execution via APJ
      lo_manager->request_execute_process( iv_instance_id = lv_id ).
      WRITE: / |Background job scheduled.|.

      " Reload instance to get job references
      TRY.
          DATA(lo_reload) = lo_manager->load_process( lv_id ).
          DATA(ls_inst) = lo_reload->get_instance( ).
          DATA(lv_status_text) = SWITCH string( ls_inst-status
            WHEN 'NEW'       THEN |New|
            WHEN 'EXECREQ'   THEN |Execution Requested|
            WHEN 'RESTREQ'   THEN |Restart Requested|
            WHEN 'RUNNING'   THEN |Running|
            WHEN 'COMPLETED' THEN |Completed|
            WHEN 'FAILED'    THEN |Failed|
            WHEN 'CANCELLED' THEN |Cancelled|
            ELSE ls_inst-status ).
          WRITE: / |Instance:  { lv_id }|.
          WRITE: / |Status:    { lv_status_text }|.
          WRITE: / |Job name:  { ls_inst-job_name }|.
          WRITE: / |Job count: { ls_inst-job_count }|.
        CATCH zcx_fi_process_error INTO DATA(lx_reload).
          WRITE: / |Instance:  { lv_id }|.
          WRITE: / |Reload warning: { lx_reload->get_text( ) }|.
      ENDTRY.
    ELSE.
      " Synchronous execution (online)
      lo_manager->execute_process( iv_instance_id = lv_id ).
    ENDIF.
    ```
  - Notes: CATCH block on reload is separate from main TRY — scheduling succeeds even if reload fails. Error output uses string templates consistently. Status displayed as human-readable text.

- [x] Task 3: Add background processing parameter to `ZFI_ALLOC_PROC_EXPORT`
  - File: `src/zfi_alloc_process/zfi_alloc_proc_export.prog.abap`
  - Action: Add `PARAMETERS: p_bgrnd AS CHECKBOX DEFAULT 'X'.` after the `p_del_pc` parameter line, with a comment separator
  - Notes: Same `AS CHECKBOX` syntax as Task 1

- [x] Task 4: Replace direct `execute_process` with conditional branch in `ZFI_ALLOC_PROC_EXPORT`
  - File: `src/zfi_alloc_process/zfi_alloc_proc_export.prog.abap`
  - Action: Identical conditional pattern as Task 2 — extract `lv_id`, replace `execute_process()` call with IF/ELSE branch
  - Code: Same as Task 2 code block above (identical logic, identical variable names)

### Acceptance Criteria

- [ ] AC 1: Given both reports are activated, when the selection screen is displayed, then a `p_bgrnd` checkbox appears (rendered as a proper checkbox via `AS CHECKBOX`) with default checked
- [ ] AC 2: Given `p_bgrnd` checked in `ZFI_ALLOC_PROC_EXEC`, when the report is executed, then an APJ job is scheduled, the instance status is `Execution Requested` (or `Running` if the job started quickly), JOB_NAME and JOB_COUNT are displayed on the report output, and the report returns immediately without blocking
- [ ] AC 3: Given `p_bgrnd` unchecked in `ZFI_ALLOC_PROC_EXEC`, when the report is executed, then the process runs synchronously (existing behavior unchanged)
- [ ] AC 4: Given `p_bgrnd` checked in `ZFI_ALLOC_PROC_EXPORT`, when the report is executed, then an APJ job is scheduled with the same behavior as AC 2
- [ ] AC 5: Given `p_bgrnd` unchecked in `ZFI_ALLOC_PROC_EXPORT`, when the report is executed, then the export process runs synchronously (existing behavior unchanged)
- [ ] AC 6: Given an APJ job is scheduled, when checking `zfi_proc_inst` in SE16, then JOB_NAME and JOB_COUNT columns are populated for the new instance
- [ ] AC 7: Given the APJ catalog/template are NOT active, when `p_bgrnd` is checked, then `zcx_fi_process_error` is raised and displayed in the report's CATCH block

## Additional Context

### Dependencies

- **EST-121 (APJ Background Execution)**: All 6 stories must be deployed and active in SAP. Specifically:
  - `zcl_fi_process_manager` must have `request_execute_process()` method
  - `zcl_fi_process_job` must be deployed
  - APJ catalog `ZFI_PROCESS_JOB_CAT` and template `ZFI_PROCESS_JOB_TMPL` must be activated in ADT
- **abapGit**: Changes are committed to `cz.imcg.fast.ovysledovka` and pulled into SAP

### Testing Strategy

**Manual testing (SE38):**
1. Run `ZFI_ALLOC_PROC_EXEC` with `p_bgrnd` checked → verify APJ job appears in "Application Jobs" Fiori app, verify instance status = EXECREQ in SE16 (`zfi_proc_inst`)
2. Run `ZFI_ALLOC_PROC_EXEC` with `p_bgrnd` unchecked → verify synchronous execution (existing behavior)
3. Run `ZFI_ALLOC_PROC_EXPORT` with `p_bgrnd` checked → same as #1
4. Run `ZFI_ALLOC_PROC_EXPORT` with `p_bgrnd` unchecked → same as #2
5. Wait for APJ job to complete → verify instance reaches COMPLETED in `zfi_proc_inst`

### Notes

- The `WRITE` output for background mode is intentionally simple — just job reference info. For monitoring, users should use the Fiori "Application Jobs" app or the health check dashboard.
- Future enhancement: could add a progress polling loop for background mode, but that defeats the purpose of non-blocking execution. Keep it simple.
- Constitution compliance: No DDIC changes, no new patterns, no factory violations. Just a parameter and an IF/ELSE branch.
- **Pre-existing constitution violation (F3)**: Both reports use `NEW zcl_fi_process_definition(...)` instead of a factory method. This is out of scope for EST-126 — tracked for a future cleanup story.
- **Selection text maintenance (F7)**: Selection texts for `p_bgrnd` (and all other parameters) are maintained in SAP via SE38/SE63, not in source code. After activation, maintain the text as "Background processing" in English.

### Adversarial Review Summary

An adversarial review (14 findings) was conducted. Fixes applied:
- **F1 (Critical)**: Changed `TYPE abap_bool` to `AS CHECKBOX DEFAULT 'X'` for proper SAP GUI rendering
- **F4 (High)**: Added explicit code block for EXPORT program (Task 4)
- **F6 (High)**: Softened AC 2 wording to account for race condition (status may be RUNNING)
- **F9 (Medium)**: Added human-readable status text mapping via SWITCH
- **F10 (Medium)**: Wrapped reload after scheduling in separate TRY/CATCH
- **F13 (Low)**: Used string templates consistently for all WRITE output

Accepted as-is: F2 (generic CATCH is fine), F5 (LUW analysis), F8 (default=true per user), F11 (docs), F12 (framework handles duplicates), F14-16 (editorial).
