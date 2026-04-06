---
story_id: "zen-5-1"
epic_id: "zen-epic-5"
title: "Create APJ sweep job class"
target_repository: "cz.en.orch"
depends_on:
  - "zen-4-7"
constitution_principles:
  - "Principle II - SAP Standards"
  - "Principle III - Consult SAP Docs"
  - "Principle V - Error Handling"
status: "done"
created: "2026-04-05"
---

# Story zen-5-1: Create APJ Sweep Job Class

Status: done

## Story

As a system administrator,
I want an APJ job class that invokes the engine sweep,
so that the orchestrator engine runs periodically as a background job without manual intervention.

## Acceptance Criteria

1. **Given** `ZCL_EN_ORCH_ENGINE` exists  
   **When** `ZCL_EN_ORCH_JOB_SWEEP` is activated and registered in the APJ framework (SAJC/SAJT)  
   **Then** the class implements both APJ interfaces: `IF_APJ_DT_EXEC_OBJECT` (design-time) and `IF_APJ_RT_EXEC_OBJECT` (runtime)

2. **Given** the APJ runtime calls `execute`  
   **When** the `execute` method runs  
   **Then** it calls `ZCL_EN_ORCH_ENGINE=>get_instance( )->sweep_all( )` exactly once

3. **Given** `sweep_all` raises `ZCX_EN_ORCH_ERROR`  
   **When** `execute` catches the exception  
   **Then** the exception is logged to a safety-net BAL log (object `ZEN_ORCH`, subobject `SWEEP`) and the job completes without dump (no unhandled exception escapes `execute`)

4. **Given** `sweep_all` raises any unexpected exception (`CX_ROOT`)  
   **When** `execute` catches it  
   **Then** the error text is logged to the safety-net BAL log and the job completes cleanly

5. **Given** no active performances exist  
   **When** the APJ job executes  
   **Then** `sweep_all` is called once and the job completes without error or dump

6. **Given** a job catalog entry and template are needed  
   **When** `ZEN_ORCH_JOB_CAT.sajc.json` and `ZEN_ORCH_JOB_TMPL.sajt.json` are activated  
   **Then** the catalog references `ZCL_EN_ORCH_JOB_SWEEP` and the template references the catalog with no mandatory parameters

7. **Given** `ZCL_EN_ORCH_JOB_SWEEP` is activated  
   **Then** the class has ABAP-Doc on its `execute` method describing its purpose  
   **And** the class activates without errors  
   **And** abaplint passes (≤120-char line limit)

## Tasks / Subtasks

- [x] Create `ZCL_EN_ORCH_JOB_SWEEP` class (AC: #1, #2, #3, #4, #5, #7)
  - [x] CLASS DEFINITION: `PUBLIC FINAL CREATE PUBLIC`, implementing `IF_APJ_DT_EXEC_OBJECT` and `IF_APJ_RT_EXEC_OBJECT`
  - [x] Implement `IF_APJ_DT_EXEC_OBJECT~GET_PARAMETERS`: return empty `ET_PARAMETER_DEF` and `ET_PARAMETER_VAL` (sweep job has no parameters)
  - [x] Implement `IF_APJ_RT_EXEC_OBJECT~EXECUTE`: create safety-net BAL log, call `sweep_all`, catch `ZCX_EN_ORCH_ERROR` and `CX_ROOT`, log and return cleanly
  - [x] Add ABAP-Doc comment on class and on `execute` method
  - [x] Create `.clas.xml` metadata file
- [x] Create APJ catalog `ZEN_ORCH_JOB_CAT.sajc.json` (AC: #6)
  - [x] Description: "ZEN_ORCH Engine Sweep", `className: "ZCL_EN_ORCH_JOB_SWEEP"`
- [x] Create APJ template `ZEN_ORCH_JOB_TMPL.sajt.json` (AC: #6)
  - [x] Reference `catalogName: "ZEN_ORCH_JOB_CAT"`, empty `parameters`

### Review Findings

- [x] [Review][Patch] Execute ABAP-Doc is implementation comment, not declaration-bound ABAP-Doc [src/zcl_en_orch_job_sweep.clas.abap:28]
- [x] [Review][Patch] Exception text may be truncated in BAL message variable [src/zcl_en_orch_job_sweep.clas.abap:60]

## Dev Notes

### APJ Interface Pattern (from planner repo)

The planner APJ job (`ZCL_FI_PROCESS_JOB`) implements two interfaces:
- `IF_APJ_DT_EXEC_OBJECT` — design-time: declares parameter definitions (none needed for sweep)
- `IF_APJ_RT_EXEC_OBJECT` — runtime: `execute( IMPORTING it_parameters TYPE tt_templ_val )`

The sweep job is simpler than the planner's generic job — it has **no parameters** and a fixed operation (always call `sweep_all`). No parameter mapping or action dispatching is required.

**Class structure skeleton:**

```abap
"! APJ job class that triggers a single ZEN_ORCH engine sweep.
"! <p>Implements the periodic background execution of the
"! orchestration engine. On each APJ invocation, sweep_all is
"! called to advance all active performances as far as
"! possible. Any exception is caught and logged; the job
"! never terminates with an unhandled dump.</p>
CLASS zcl_en_orch_job_sweep DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES if_apj_dt_exec_object.
    INTERFACES if_apj_rt_exec_object.

  PRIVATE SECTION.
ENDCLASS.

CLASS zcl_en_orch_job_sweep IMPLEMENTATION.

  METHOD if_apj_dt_exec_object~get_parameters.
    " No parameters — sweep job requires no input
    CLEAR et_parameter_def.
    CLEAR et_parameter_val.
  ENDMETHOD.

  METHOD if_apj_rt_exec_object~execute.
    "! Execute one engine sweep cycle.
    "! @parameter it_parameters | APJ runtime parameters (unused)
    DATA lo_job_log TYPE REF TO if_bali_log.

    " 1. Create safety-net BAL log for the APJ job itself
    TRY.
        lo_job_log = cl_bali_log=>create_with_header(
          header = cl_bali_header_setter=>create(
            object      = 'ZEN_ORCH'
            subobject   = 'SWEEP'
            external_id = CONV balnrext( sy-datum && sy-uzeit )
          )
        ).
      CATCH cx_bali_runtime.
        " Safety log creation failed — continue without it
    ENDTRY.

    " 2. Invoke sweep; catch any orchestrator exception
    TRY.
        zcl_en_orch_engine=>get_instance( )->sweep_all( ).

      CATCH zcx_en_orch_error INTO DATA(lx_orch).
        IF lo_job_log IS BOUND.
          TRY.
              lo_job_log->add_item(
                item = cl_bali_message_setter=>create(
                  severity   = if_bali_constants=>c_severity_error
                  id         = 'ZEN_ORCH'
                  number     = '001'
                  variable_1 = CONV symsgv( lx_orch->get_text( ) )
                )
              ).
            CATCH cx_bali_runtime ##NO_HANDLER.
          ENDTRY.
        ENDIF.

      CATCH cx_root INTO DATA(lx_root).
        IF lo_job_log IS BOUND.
          TRY.
              lo_job_log->add_item(
                item = cl_bali_free_text_setter=>create(
                  severity = if_bali_constants=>c_severity_error
                  text     = |Unexpected exception: { lx_root->get_text( ) }|
                )
              ).
            CATCH cx_bali_runtime ##NO_HANDLER.
          ENDTRY.
        ENDIF.
    ENDTRY.

    " 3. Save safety-net log and assign to current APJ job
    IF lo_job_log IS BOUND.
      TRY.
          cl_bali_log_db=>get_instance( )->save_log(
            log                        = lo_job_log
            assign_to_current_appl_job = abap_true ).
        CATCH cx_bali_runtime ##NO_HANDLER.
      ENDTRY.
    ENDIF.
  ENDMETHOD.

ENDCLASS.
```

### Safety-net BAL Log Pattern

The safety-net log follows the exact pattern from `ZCL_FI_PROCESS_JOB` in the planner:

| Pattern | Value |
|---------|-------|
| BAL object | `ZEN_ORCH` |
| BAL subobject | `SWEEP` |
| External ID | `sy-datum && sy-uzeit` (YYYYMMDDHHMMSS) |
| Error message | ZEN_ORCH message 001 (ENGINE_SWEEP_FAILED textid) |
| Assign to APJ | `assign_to_current_appl_job = abap_true` |

The safety-net log is always created before calling `sweep_all`. If `sweep_all` succeeds without exception, the empty log is still saved and assigned (consistent with planner pattern — the log is the APJ execution record).

### APJ Catalog and Template JSON Files

**`zen_orch_job_cat.sajc.json`** — job catalog:
```json
{
  "formatVersion": "1",
  "header": {
    "description": "ZEN_ORCH Engine Sweep",
    "originalLanguage": "en"
  },
  "generalInformation": {
    "className": "ZCL_EN_ORCH_JOB_SWEEP"
  }
}
```

**`zen_orch_job_tmpl.sajt.json`** — job template:
```json
{
  "formatVersion": "1",
  "header": {
    "description": "ZEN_ORCH Engine Sweep Job",
    "originalLanguage": "en"
  },
  "generalInformation": {
    "catalogName": "ZEN_ORCH_JOB_CAT"
  },
  "parameters": {
    "singleValueParameters": []
  }
}
```

### File Locations

All files go under `src/zen_orch/` in the `cz.en.orch` repository:

| File | Description |
|------|-------------|
| `zcl_en_orch_job_sweep.clas.abap` | Class definition + implementation |
| `zcl_en_orch_job_sweep.clas.xml` | abapGit metadata |
| `zen_orch_job_cat.sajc.json` | APJ job catalog |
| `zen_orch_job_tmpl.sajt.json` | APJ job template |

### Project Structure Notes

- Architecture layer 11 (see `architecture.md` activation order): `ZCL_EN_ORCH_JOB_SWEEP` depends on `ZCL_EN_ORCH_ENGINE` (layer 10)
- The class is `FINAL` — no inheritance hierarchy needed (contrast with planner's abstract `ZCL_FI_PROCESS_JOB_BASE`)
- The sweep job has **no APJ parameters** — `GET_PARAMETERS` returns empty tables
- Use `CONV balnrext( sy-datum && sy-uzeit )` for external_id (max 40 chars; DATUM=8 + UZEIT=6 = 14 chars — safe)
- `CL_BALI_FREE_TEXT_SETTER=>CREATE` is used for `CX_ROOT` catch (no T100 message; free text only)
- `CL_BALI_MESSAGE_SETTER=>CREATE` requires a T100 message — use ZEN_ORCH message `001` (ENGINE_SWEEP_FAILED)

### abaplint Rules (from prior zen stories)

| Rule | Detail |
|------|--------|
| `definitions_top` | All `DATA` declarations at the top of each method — before any logic |
| `no_inline_in_optional_branches` | No inline declarations inside TRY/CATCH, IF, LOOP branches |
| `strict_sql` | N/A — no SQL in this class |
| Line length | ≤120 chars strictly; use `&&` for long string literals |
| `##NO_HANDLER` | Required on empty `CATCH` blocks |
| `CLEAR et_*` | Use `CLEAR` not `et_parameter_def = VALUE #( )` — safer and idiomatic for exporting params |

### References

- APJ interfaces pattern: [planner repo: `zcl_fi_process_job.clas.abap`]
- APJ catalog format: [planner repo: `zfi_process_job_cat.sajc.json`]
- APJ template format: [planner repo: `zfi_process_job_tmpl.sajt.json`]
- Safety-net BAL log pattern: [planner repo: `zcl_fi_process_job.clas.abap` lines 40–60]
- Architecture: [Source: `_bmad-output/planning-artifacts/architecture.md#Infrastructure & Logging`]
- Activation order layer 11: [Source: `_bmad-output/planning-artifacts/architecture.md#Activation Order`]
- Epic 5 story 5.1 requirements: [Source: `_bmad-output/planning-artifacts/epics.md#Story 5.1`]

## Constitution Compliance

| Principle | Application |
|-----------|-------------|
| **II — SAP Standards** | SAP naming (`ZCL_EN_ORCH_JOB_SWEEP`), ≤120 chars, ABAP-Doc on class and `execute` |
| **III — Consult SAP Docs** | APJ interfaces (`IF_APJ_DT_EXEC_OBJECT`, `IF_APJ_RT_EXEC_OBJECT`) verified against SAP docs; BAL API (`CL_BALI_LOG`, `CL_BALI_LOG_DB`) is released Clean Core A |
| **V — Error Handling** | `ZCX_EN_ORCH_ERROR` caught and logged; `CX_ROOT` caught as safety net; no unhandled exceptions escape the APJ `execute` method |

## Dev Agent Record

### Agent Model Used

claude-sonnet-4.6 (github-copilot/claude-sonnet-4.6)

### Debug Log References

No issues encountered. Skeleton from Dev Notes applied with one correction: the story Dev Notes
referenced message ZEN_ORCH 001 for the ZCX_EN_ORCH_ERROR catch, but message 001 is
"Engine sweep started" (informational). Message 003 "Engine sweep failed: &1" is the correct
error message and matches the `engine_sweep_failed` textid constant in ZCX_EN_ORCH_ERROR.
abaplint `no_inline_in_optional_branches` rule enforced by hoisting all DATA declarations
(lx_orch, lx_root) to the top of the execute method, outside TRY/CATCH blocks.

### Completion Notes List

- Implemented `ZCL_EN_ORCH_JOB_SWEEP`: PUBLIC FINAL CREATE PUBLIC, implements both APJ interfaces
- `GET_PARAMETERS`: returns empty ET_PARAMETER_DEF and ET_PARAMETER_VAL (no parameters)
- `EXECUTE`: creates safety-net BAL log (ZEN_ORCH/SWEEP), calls `sweep_all`, catches
  ZCX_EN_ORCH_ERROR (logged via message 003) and CX_ROOT (logged via free text), saves log
  with assign_to_current_appl_job = abap_true; no unhandled exception can escape
- ABAP-Doc on class definition and on execute method
- All DATA declarations at method top (abaplint definitions_top / no_inline_in_optional_branches)
- All lines ≤ 120 chars; ##NO_HANDLER on all empty CATCH blocks
- Created `zcl_en_orch_job_sweep.clas.xml` with standard abapGit VSEOCLASS metadata
- Created `zen_orch_job_cat.sajc.json`: catalog name ZEN_ORCH_JOB_CAT, className ZCL_EN_ORCH_JOB_SWEEP
- Created `zen_orch_job_tmpl.sajt.json`: catalog ZEN_ORCH_JOB_CAT, empty singleValueParameters

### File List

- `src/zcl_en_orch_job_sweep.clas.abap` (new)
- `src/zcl_en_orch_job_sweep.clas.xml` (new)
- `src/zen_orch_job_cat.sajc.json` (new)
- `src/zen_orch_job_tmpl.sajt.json` (new)
