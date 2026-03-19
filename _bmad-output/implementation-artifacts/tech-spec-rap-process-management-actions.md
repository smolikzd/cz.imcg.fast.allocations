---
title: 'RAP Process Management Actions (EST-111–114)'
slug: 'rap-process-management-actions'
created: '2026-03-17'
status: 'abandoned-needs-redesign'
stepsCompleted: [1, 2, 3, 4]
tech_stack:
  - 'ABAP 7.58 / SAP S/4HANA'
  - 'SAP RAP (managed BO, strict mode 2)'
  - 'OData V4 (SRVB)'
  - 'CDS R_/C_ layering'
files_to_modify:
  - 'src/zfiproc_r_instance_tp.bdef.asbdef'
  - 'src/zbp_fiproc_r_instance_tp.clas.locals_imp.abap'
  - 'src/zfiproc_c_instance_tp.ddls.asddls'
files_to_create:
  - 'src/zfiproc_c_instance_tp.bdef.asbdef'
  - 'src/zfiproc_c_instance_tp.bdef.xml'
code_patterns:
  - 'RAP LHC: lhc_ZFIPROC_R_INSTANCE_TP INHERITING FROM cl_abap_behavior_handler'
  - 'Singleton: ZCL_FI_PROCESS_MANAGER=>get_instance()->'
  - 'Error handling: CATCH zcx_fi_process_error, use if_t100_dyn_msg for message construction'
  - 'RAP action result: RESULT [1] $self'
  - 'RAP feature control: get_instance_features FOR INSTANCE FEATURES'
  - 'Instance-level error: APPEND to reported-zfiproc_r_instance_tp'
  - 'Instance-level failure: APPEND to failed-zfiproc_r_instance_tp'
test_patterns:
  - 'Manual testing via OData V4 service / Fiori app'
  - 'No ABAP Unit tests for RAP layer in this project'
---

# Tech-Spec: RAP Process Management Actions (EST-111–114)

**Created:** 2026-03-17
**Linear:** EST-111 (CancelProcess), EST-112 (ExecuteProcess), EST-113 (SupersedeProcess), EST-114 (RestartProcess)
**Target repository:** `cz.imcg.fast.planner`

---

## Overview

### Problem Statement

The `ZCL_FI_PROCESS_MANAGER` business logic for process lifecycle transitions (execute, cancel,
supersede, restart) is fully implemented as ABAP API but is not exposed through the RAP framework.
Users cannot trigger these transitions from the Fiori UI (`ZFIPROC_UI`) via OData V4 — the process
monitoring app is currently read-only with no actionable controls.

### Solution

Add 4 RAP actions to the existing `ZFIPROC_R_INSTANCE_TP` behavior definition (BDEF), implement
their handlers in the behavior implementation class (`ZBP_FIPROC_R_INSTANCE_TP`), create the
required projection BDEF for `ZFIPROC_C_INSTANCE_TP`, and add `use action` redirections in the
consumption projection CDS view. Each action delegates to the existing `ZCL_FI_PROCESS_MANAGER`
singleton. Instance-based dynamic feature controls guard each action based on the current instance
status so invalid actions are disabled in the UI before they can be triggered.

### Scope

**In Scope:**

- `src/zfiproc_r_instance_tp.bdef.asbdef` — add `Status` to `field (readonly)`, `with features
  instance`, 4 action declarations
- `src/zbp_fiproc_r_instance_tp.clas.locals_imp.abap` — add `get_instance_features` handler + 4
  `FOR ACTION` method implementations
- `src/zfiproc_c_instance_tp.ddls.asddls` — add 4 `use action` redirections
- `src/zfiproc_c_instance_tp.bdef.asbdef` — **NEW FILE**: projection BDEF required for `use action`
  to work in the OData service
- `src/zfiproc_c_instance_tp.bdef.xml` — **NEW FILE**: abapGit metadata for the projection BDEF

**Out of Scope:**

- No changes to `ZFIPROC_R_STEP_TP` behavior
- No changes to `ZCL_FI_PROCESS_MANAGER`, `ZCL_FI_PROCESS_INSTANCE`, or any other business logic
- No UI annotations or button labels (`.ddlx` metadata extensions — future ticket)
- No changes to service definition `ZFIPROC_UI_INSTANCE` (`.srvd`)
- No changes to `cz.imcg.fast.ovysledovka`

---

## Context for Development

### Codebase Patterns

**RAP Architecture — mandatory conventions:**

- Single R-layer BDEF: `zfiproc_r_instance_tp.bdef.asbdef` — `managed`, `strict ( 2 )`, one
  implementation class `zbp_fiproc_r_instance_tp`
- Behavior implementation class: global class `zbp_fiproc_r_instance_tp.clas.abap` is
  `ABSTRACT FINAL FOR BEHAVIOR OF zfiproc_r_instance_tp` — it remains an empty shell; all RAP
  handler methods live exclusively in the Local Handler Class (LHC) in `locals_imp.abap`
- LHC class: `lhc_ZFIPROC_R_INSTANCE_TP INHERITING FROM cl_abap_behavior_handler` — all `FOR
  ACTION`, `FOR INSTANCE FEATURES`, `FOR INSTANCE AUTHORIZATION` methods are declared and
  implemented here
- No `locals_def.abap` exists and none needs to be created — all types are DDIC types
- **Projection BDEF is required**: `use action` redirections in a C-layer CDS view only work when
  a projection BDEF (`ZFIPROC_C_INSTANCE_TP`) exists that projects from the R-layer BDEF

**Business logic access pattern — Constitution Principle IV (Factory):**
```abap
zcl_fi_process_manager=>get_instance( )->execute_process( iv_instance_id = <key>-InstanceId ).
```
Never use `NEW zcl_fi_process_manager( )`.

**Status string literals — why constants cannot be used:**
`gc_status` is declared `PRIVATE` in `zcl_fi_process_manager` and is inaccessible from the LHC.
No shared constant class exists. Use domain value string literals directly; they are stable and
defined in the `ZFI_PROCESS_STATUS` DDIC domain (CHAR 20, 8 fixed values):

| Literal | Meaning |
|---------|---------|
| `'NEW'` | Not yet executed |
| `'RUNNING'` | Currently executing |
| `'COMPLETED'` | Finished successfully |
| `'FAILED'` | Execution failed |
| `'CANCELLED'` | Manually cancelled |
| `'PENDING'` | Queued for bgRFC |
| `'SUPERSEDED'` | Replaced by newer instance |
| `'SKIPPED'` | Step skipped |

**Action guard rules — derived from actual `zcl_fi_process_instance` business logic:**

| RAP Action | Enable button when Status = | Underlying method | Error textid on violation |
|------------|----------------------------|-------------------|--------------------------|
| `ExecuteProcess` | `'NEW'` | `execute_process()` | `invalid_status` (msg 011) |
| `CancelProcess` | `'RUNNING'` or `'FAILED'` | `cancel_process()` | `invalid_status` (msg 011) |
| `SupersedeProcess` | `'COMPLETED'` | `supersede_process()` | `instance_superseded` (msg 018) |
| `RestartProcess` | `'FAILED'` | `restart_process()` | `invalid_status` (msg 011) |

**Error surfacing — Constitution Principle V:**

Use `if_t100_dyn_msg` to construct messages from the exception's own T100 key. This correctly
handles all message variables (`value`, `limit`, `running`) for all error types including
`parallel_limit_exceeded` (msg 017) which uses all three attributes:

```abap
CATCH zcx_fi_process_error INTO DATA(lx_error).
  APPEND VALUE #(
    %tky = <key>-%tky
    %msg = new_message_from_bapi(
             i_msgid   = lx_error->if_t100_message~t100key-msgid
             i_msgno   = lx_error->if_t100_message~t100key-msgno
             i_msgty   = 'E'
             i_msgv1   = lx_error->value
             i_msgv2   = lx_error->limit
             i_msgv3   = lx_error->running )
  ) TO reported-zfiproc_r_instance_tp.
  APPEND VALUE #( %tky = <key>-%tky )
    TO failed-zfiproc_r_instance_tp.
```

Note: both `reported` and `failed` tables must be appended. The `reported` entry provides the
inline message; the `failed` entry signals to RAP that this key did not complete successfully.

**Important: `execute_process()` is asynchronous (bgRFC):**
`zcl_fi_process_manager=>execute_process()` dispatches execution via bgRFC. The status transition
from `NEW` to `RUNNING` happens asynchronously — it is NOT visible in the RAP buffer immediately
after the call. Therefore the `execute_process` action handler must NOT attempt to re-read the
entity for its result parameter. Instead, return the entity as-is from the buffer (still `NEW`).
The Fiori app will refresh the list after the action completes, picking up the updated status.
This is the correct pattern for async RAP actions. The other three actions (`cancel`, `supersede`,
`restart`) call `COMMIT WORK AND WAIT` internally and their status changes ARE visible in the
buffer after the call.

**Service binding re-activation after changes:**
After activating the new/changed objects, re-activate (not re-publish) the service binding
`ZFIPROC_UI_INSTANCE_O4` in ADT to refresh the OData `$metadata` document. The binding was
already published; a re-publish is not required, but a re-activation is needed for the new
actions to appear in `$metadata`.

### Files to Reference

| File | Purpose |
|------|---------|
| `src/zfiproc_r_instance_tp.bdef.asbdef` | R-layer BDEF — modify to add actions |
| `src/zbp_fiproc_r_instance_tp.clas.locals_imp.abap` | LHC — extend with new methods |
| `src/zbp_fiproc_r_instance_tp.clas.abap` | Global BP class — READ ONLY, no changes |
| `src/zfiproc_c_instance_tp.ddls.asddls` | C-layer projection CDS — add use action lines |
| `src/zfiproc_r_instance_tp.bdef.xml` | Reference for abapGit XML format of existing BDEF |
| `src/zcl_fi_process_manager.clas.abap` | READ ONLY — method signatures for delegation |
| `src/zcx_fi_process_error.clas.abap` | READ ONLY — exception attributes and T100 key |

### Technical Decisions

1. **Action result type `RESULT [1] $self`**: Returns the refreshed instance so Fiori updates the
   row. Exception: `execute_process` returns the entity as-is (still `NEW`) because bgRFC is
   async — the row will update on the next Fiori refresh cycle.

2. **`with features instance` on entity level**: Enables per-instance dynamic feature control.
   Each action declares `( features : instance )` to link to the `get_instance_features` handler.

3. **Projection BDEF is a new file**: `ZFIPROC_C_INSTANCE_TP` currently has no BDEF. It must be
   created as a `projection` type BDEF that uses `transactional_query` provider contract,
   projecting from `ZFIPROC_R_INSTANCE_TP`, with `use action` for all 4 actions.

4. **`if_t100_dyn_msg` for error messages**: The exception's own T100 key is used for message
   construction to avoid hardcoding message numbers and to support all message variable slots.

5. **`Status` added to `field (readonly)` in BDEF**: Required so RAP recognises it as a managed
   field accessible via `READ ENTITIES ... FIELDS ( Status )` in `get_instance_features`.

---

## Implementation Plan

### Tasks

Tasks are ordered by dependency: R-layer BDEF first, then implementation, then C-layer.

---

- [x] **Task 1: Update R-layer BDEF — add Status to readonly, feature control, and 4 actions**
  - File: `src/zfiproc_r_instance_tp.bdef.asbdef`
  - Action: Replace the entire file with the following exact content:

    ```
    managed implementation in class zbp_fiproc_r_instance_tp unique;
    strict ( 2 );

    define behavior for ZFIPROC_R_INSTANCE_TP
    persistent table zfi_proc_inst
    lock master
    authorization master ( instance )
    with features instance
    {
      field ( readonly )
      InstanceId,
      ProcessType,
      Status,
      CreatedBy,
      CurrentStep,
      EndedAt,
      ErrorMessage,
      ParameterData,
      StartedAt,
      StartedBy,
      ParamVal1,
      ParamVal2,
      ParamVal3,
      ParamVal4,
      ParamVal5,
      ParamLabel1,
      ParamLabel2,
      ParamLabel3,
      ParamLabel4,
      ParamLabel5,
      BusinessStatus1,
      BusinessStatus2;
      //create;
      //update;
      //delete;
      association _Step { create; }

      action ( features : instance ) ExecuteProcess   result [1] $self;
      action ( features : instance ) CancelProcess    result [1] $self;
      action ( features : instance ) SupersedeProcess result [1] $self;
      action ( features : instance ) RestartProcess   result [1] $self;
    }

    define behavior for ZFIPROC_R_STEP_TP //alias <alias_name>
    persistent table zfi_proc_step
    lock dependent by _Instance
    authorization dependent by _Instance
    //etag master <field_name>
    {
      //update;
      //delete;
      field ( readonly )
      InstanceId,
      StepNumber,
      SubstepNumber,
      StepType,
      Description,
      StartedAt,
      ContextData,
      EndedAt,
      ErrorMessage,
      QueueId,
      ResultData,
      RetryCount,
      Status;
      association _Instance;
    }
    ```

  - Notes:
    - `Status` is added to the `field ( readonly )` block (was missing — required for
      `READ ENTITIES ... FIELDS ( Status )` in `get_instance_features`)
    - `with features instance` on the entity header enables the `get_instance_features` handler
    - `( features : instance )` on each action declaration links it to that handler
    - `result [1] $self` returns the updated entity after the action
    - `ZFIPROC_R_STEP_TP` behavior is unchanged — reproduced verbatim to show full file context

---

- [x] **Task 2: Extend LHC — replace locals_imp.abap with full implementation**
  - File: `src/zbp_fiproc_r_instance_tp.clas.locals_imp.abap`
  - Action: Replace the entire file content with the following:

    ```abap
    CLASS lhc_ZFIPROC_R_INSTANCE_TP DEFINITION INHERITING FROM cl_abap_behavior_handler.
      PRIVATE SECTION.

        METHODS get_instance_authorizations FOR INSTANCE AUTHORIZATION
          IMPORTING keys REQUEST requested_authorizations
                    FOR zfiproc_r_instance_tp RESULT result.

        METHODS get_instance_features FOR INSTANCE FEATURES
          IMPORTING keys REQUEST requested_features
                    FOR zfiproc_r_instance_tp RESULT result.

        METHODS execute_process FOR MODIFY
          IMPORTING keys FOR ACTION zfiproc_r_instance_tp~ExecuteProcess RESULT result.

        METHODS cancel_process FOR MODIFY
          IMPORTING keys FOR ACTION zfiproc_r_instance_tp~CancelProcess RESULT result.

        METHODS supersede_process FOR MODIFY
          IMPORTING keys FOR ACTION zfiproc_r_instance_tp~SupersedeProcess RESULT result.

        METHODS restart_process FOR MODIFY
          IMPORTING keys FOR ACTION zfiproc_r_instance_tp~RestartProcess RESULT result.

    ENDCLASS.

    CLASS lhc_ZFIPROC_R_INSTANCE_TP IMPLEMENTATION.

      METHOD get_instance_authorizations.
      ENDMETHOD.

      METHOD get_instance_features.
        READ ENTITIES OF zfiproc_r_instance_tp IN LOCAL MODE
          ENTITY zfiproc_r_instance_tp
            FIELDS ( Status ) WITH CORRESPONDING #( keys )
          RESULT DATA(lt_instances)
          FAILED DATA(lt_failed).

        " Keys that could not be read get all actions disabled
        result = VALUE #(
          FOR ls_failed IN lt_failed
            ( %tky                     = ls_failed-%tky
              %action-ExecuteProcess   = if_abap_behv=>fc-op-disabled
              %action-CancelProcess    = if_abap_behv=>fc-op-disabled
              %action-SupersedeProcess = if_abap_behv=>fc-op-disabled
              %action-RestartProcess   = if_abap_behv=>fc-op-disabled ) ).

        " Set feature control based on current Status
        result = VALUE #( BASE result
          FOR ls_instance IN lt_instances
            ( %tky                     = ls_instance-%tky
              %action-ExecuteProcess   = COND #(
                WHEN ls_instance-Status = 'NEW'
                THEN if_abap_behv=>fc-op-enabled
                ELSE if_abap_behv=>fc-op-disabled )
              %action-CancelProcess    = COND #(
                WHEN ls_instance-Status = 'RUNNING'
                  OR ls_instance-Status = 'FAILED'
                THEN if_abap_behv=>fc-op-enabled
                ELSE if_abap_behv=>fc-op-disabled )
              %action-SupersedeProcess = COND #(
                WHEN ls_instance-Status = 'COMPLETED'
                THEN if_abap_behv=>fc-op-enabled
                ELSE if_abap_behv=>fc-op-disabled )
              %action-RestartProcess   = COND #(
                WHEN ls_instance-Status = 'FAILED'
                THEN if_abap_behv=>fc-op-enabled
                ELSE if_abap_behv=>fc-op-disabled ) ) ).
      ENDMETHOD.

      METHOD execute_process.
        " execute_process dispatches via bgRFC — status transition is asynchronous.
        " Do NOT re-read the entity after the call: the buffer will still show 'NEW'.
        " Return the entity as-is; Fiori refreshes the list after action completion.
        LOOP AT keys ASSIGNING FIELD-SYMBOL(<key>).
          TRY.
              zcl_fi_process_manager=>get_instance( )->execute_process(
                iv_instance_id = <key>-InstanceId ).

              READ ENTITIES OF zfiproc_r_instance_tp IN LOCAL MODE
                ENTITY zfiproc_r_instance_tp
                  ALL FIELDS WITH VALUE #( ( %tky = <key>-%tky ) )
                RESULT DATA(lt_result).

              IF lt_result IS NOT INITIAL.
                result = VALUE #( BASE result
                  ( %tky   = <key>-%tky
                    %param = lt_result[ 1 ] ) ).
              ELSE.
                APPEND VALUE #( %tky = <key>-%tky )
                  TO failed-zfiproc_r_instance_tp.
              ENDIF.

            CATCH zcx_fi_process_error INTO DATA(lx_error).
              APPEND VALUE #(
                %tky = <key>-%tky
                %msg = new_message_from_bapi(
                         i_msgid = lx_error->if_t100_message~t100key-msgid
                         i_msgno = lx_error->if_t100_message~t100key-msgno
                         i_msgty = 'E'
                         i_msgv1 = lx_error->value
                         i_msgv2 = lx_error->limit
                         i_msgv3 = lx_error->running )
              ) TO reported-zfiproc_r_instance_tp.
              APPEND VALUE #( %tky = <key>-%tky )
                TO failed-zfiproc_r_instance_tp.
          ENDTRY.
        ENDLOOP.
      ENDMETHOD.

      METHOD cancel_process.
        " cancel_process calls COMMIT WORK AND WAIT internally — status change is
        " synchronous and visible in buffer after the call.
        LOOP AT keys ASSIGNING FIELD-SYMBOL(<key>).
          TRY.
              zcl_fi_process_manager=>get_instance( )->cancel_process(
                iv_instance_id = <key>-InstanceId ).

              READ ENTITIES OF zfiproc_r_instance_tp IN LOCAL MODE
                ENTITY zfiproc_r_instance_tp
                  ALL FIELDS WITH VALUE #( ( %tky = <key>-%tky ) )
                RESULT DATA(lt_result).

              IF lt_result IS NOT INITIAL.
                result = VALUE #( BASE result
                  ( %tky   = <key>-%tky
                    %param = lt_result[ 1 ] ) ).
              ELSE.
                APPEND VALUE #( %tky = <key>-%tky )
                  TO failed-zfiproc_r_instance_tp.
              ENDIF.

            CATCH zcx_fi_process_error INTO DATA(lx_error).
              APPEND VALUE #(
                %tky = <key>-%tky
                %msg = new_message_from_bapi(
                         i_msgid = lx_error->if_t100_message~t100key-msgid
                         i_msgno = lx_error->if_t100_message~t100key-msgno
                         i_msgty = 'E'
                         i_msgv1 = lx_error->value
                         i_msgv2 = lx_error->limit
                         i_msgv3 = lx_error->running )
              ) TO reported-zfiproc_r_instance_tp.
              APPEND VALUE #( %tky = <key>-%tky )
                TO failed-zfiproc_r_instance_tp.
          ENDTRY.
        ENDLOOP.
      ENDMETHOD.

      METHOD supersede_process.
        " supersede_process calls COMMIT WORK AND WAIT internally — status change is
        " synchronous and visible in buffer after the call.
        LOOP AT keys ASSIGNING FIELD-SYMBOL(<key>).
          TRY.
              zcl_fi_process_manager=>get_instance( )->supersede_process(
                iv_instance_id = <key>-InstanceId ).

              READ ENTITIES OF zfiproc_r_instance_tp IN LOCAL MODE
                ENTITY zfiproc_r_instance_tp
                  ALL FIELDS WITH VALUE #( ( %tky = <key>-%tky ) )
                RESULT DATA(lt_result).

              IF lt_result IS NOT INITIAL.
                result = VALUE #( BASE result
                  ( %tky   = <key>-%tky
                    %param = lt_result[ 1 ] ) ).
              ELSE.
                APPEND VALUE #( %tky = <key>-%tky )
                  TO failed-zfiproc_r_instance_tp.
              ENDIF.

            CATCH zcx_fi_process_error INTO DATA(lx_error).
              " Note: supersede() raises instance_superseded (018) when status is wrong,
              " but may also raise invalid_status (011) in edge cases. Use the exception's
              " own T100 key rather than hardcoding a message number.
              APPEND VALUE #(
                %tky = <key>-%tky
                %msg = new_message_from_bapi(
                         i_msgid = lx_error->if_t100_message~t100key-msgid
                         i_msgno = lx_error->if_t100_message~t100key-msgno
                         i_msgty = 'E'
                         i_msgv1 = lx_error->value
                         i_msgv2 = lx_error->limit
                         i_msgv3 = lx_error->running )
              ) TO reported-zfiproc_r_instance_tp.
              APPEND VALUE #( %tky = <key>-%tky )
                TO failed-zfiproc_r_instance_tp.
          ENDTRY.
        ENDLOOP.
      ENDMETHOD.

      METHOD restart_process.
        " restart_process delegates to execute() internally which uses bgRFC for steps.
        " However restart() itself is synchronous up to the point of re-queuing.
        " The status transitions to RUNNING synchronously — re-read is valid.
        LOOP AT keys ASSIGNING FIELD-SYMBOL(<key>).
          TRY.
              zcl_fi_process_manager=>get_instance( )->restart_process(
                iv_instance_id = <key>-InstanceId ).

              READ ENTITIES OF zfiproc_r_instance_tp IN LOCAL MODE
                ENTITY zfiproc_r_instance_tp
                  ALL FIELDS WITH VALUE #( ( %tky = <key>-%tky ) )
                RESULT DATA(lt_result).

              IF lt_result IS NOT INITIAL.
                result = VALUE #( BASE result
                  ( %tky   = <key>-%tky
                    %param = lt_result[ 1 ] ) ).
              ELSE.
                APPEND VALUE #( %tky = <key>-%tky )
                  TO failed-zfiproc_r_instance_tp.
              ENDIF.

            CATCH zcx_fi_process_error INTO DATA(lx_error).
              APPEND VALUE #(
                %tky = <key>-%tky
                %msg = new_message_from_bapi(
                         i_msgid = lx_error->if_t100_message~t100key-msgid
                         i_msgno = lx_error->if_t100_message~t100key-msgno
                         i_msgty = 'E'
                         i_msgv1 = lx_error->value
                         i_msgv2 = lx_error->limit
                         i_msgv3 = lx_error->running )
              ) TO reported-zfiproc_r_instance_tp.
              APPEND VALUE #( %tky = <key>-%tky )
                TO failed-zfiproc_r_instance_tp.
          ENDTRY.
        ENDLOOP.
      ENDMETHOD.

    ENDCLASS.
    ```

  - Notes:
    - `FAILED DATA(lt_failed)` — correct table type; failed keys get all actions disabled
    - `IF lt_result IS NOT INITIAL` guard prevents `CX_SY_ITAB_LINE_NOT_FOUND` dump
    - Both `reported` and `failed` tables are populated on error (strict mode 2 requirement)
    - `new_message_from_bapi` with the exception's own T100 key handles all message variable
      slots — no hardcoded message numbers
    - `execute_process` comment explicitly documents the async/bgRFC behaviour and why
      the re-read still returns `'NEW'`; this is expected and correct

---

- [x] **Task 3: Create projection BDEF (NEW FILE)**
  - File: `src/zfiproc_c_instance_tp.bdef.asbdef` *(create new — does not exist)*
  - Action: Create the file with the following content:

    ```
    projection;
    strict ( 2 );

    define behavior for ZFIPROC_C_INSTANCE_TP //alias <alias_name>
    {
      use action ExecuteProcess;
      use action CancelProcess;
      use action SupersedeProcess;
      use action RestartProcess;
    }
    ```

  - Notes: This projection BDEF is mandatory — without it, `use action` in the CDS projection
    view has no BDEF contract to project from, and the OData service will not surface the actions.
    The `projection; strict ( 2 );` header matches the R-layer's `strict ( 2 )` setting.

  - Also create the abapGit metadata file `src/zfiproc_c_instance_tp.bdef.xml`:

    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <abapGit version="v1.0.0" serializer="LCL_OBJECT_BDEF" serializer_version="v1.0.0">
     <asx:abap xmlns:asx="http://www.sap.com/abapxml" version="1.0">
      <asx:values>
       <BDEF>
        <NAME>ZFIPROC_C_INSTANCE_TP</NAME>
        <TYPE>BDEF/BDP</TYPE>
        <DESCRIPTION>Projection BDEF for Instance (process management actions)</DESCRIPTION>
        <DESCRIPTION_TEXT_LIMIT>60</DESCRIPTION_TEXT_LIMIT>
        <LANGUAGE>EN</LANGUAGE>
        <MASTER_LANGUAGE>EN</MASTER_LANGUAGE>
        <ABAP_LANGU_VERSION>X</ABAP_LANGU_VERSION>
        <SOURCE_FIXED_POINT_ARITHMETIC>true</SOURCE_FIXED_POINT_ARITHMETIC>
        <SOURCE_UNICODE_CHECKS_ACTIVE>true</SOURCE_UNICODE_CHECKS_ACTIVE>
       </BDEF>
      </asx:values>
     </asx:abap>
    </abapGit>
    ```

---

- [x] **Task 4: Add `use action` redirections to consumption projection CDS**
  - File: `src/zfiproc_c_instance_tp.ddls.asddls`
  - Action: The `use action` redirections live in the **projection BDEF** (Task 3), NOT in the
    CDS view. The CDS view itself requires no changes for action exposure. However, verify that
    `Status` is already projected (it is — line 20 of the existing file). No changes needed to
    this file.
  - Notes: This supersedes the earlier plan to add `use action` lines directly into the CDS DDL.
    In RAP, `use action` belongs in the projection BDEF (`.bdef.asbdef`), not in the CDS
    projection view (`.ddls.asddls`). The CDS view's `provider contract transactional_query`
    already links it to the BDEF layer.

---

- [x] **Task 5: Re-activate service binding**
  - Object: `ZFIPROC_UI_INSTANCE_O4` (service binding, type SRVB)
  - Action: In ADT, open `ZFIPROC_UI_INSTANCE_O4`, click **Activate** (not Publish — it is
    already published). This refreshes the OData `$metadata` document so the 4 new actions
    appear as OData function imports / bound actions accessible to the Fiori app.
  - Notes: No structural changes to the service definition (`ZFIPROC_UI_INSTANCE`) are required.

---

### Activation Order

Activate objects in this exact sequence to avoid dependency errors:

1. `ZFIPROC_R_INSTANCE_TP` BDEF (`zfiproc_r_instance_tp.bdef.asbdef`)
2. `ZBP_FIPROC_R_INSTANCE_TP` behavior pool (`zbp_fiproc_r_instance_tp.clas.locals_imp.abap`)
3. `ZFIPROC_C_INSTANCE_TP` projection BDEF (`zfiproc_c_instance_tp.bdef.asbdef`) *(new)*
4. `ZFIPROC_UI_INSTANCE_O4` service binding — re-activate

---

### Acceptance Criteria

- [ ] **AC1 — Execute happy path:** Given an instance with `Status = 'NEW'`, when `ExecuteProcess`
  is called via OData POST, then the action returns HTTP 200, no error messages are returned, and
  the response body contains the instance entity (Status may still show `'NEW'` due to async
  bgRFC dispatch — this is expected and correct).

- [ ] **AC2 — Cancel happy path (RUNNING):** Given an instance with `Status = 'RUNNING'`, when
  `CancelProcess` is called, then the action returns HTTP 200, the response body contains the
  instance with `Status = 'CANCELLED'`, and `EndedAt` is populated with the cancellation
  timestamp.

- [ ] **AC3 — Cancel happy path (FAILED):** Given an instance with `Status = 'FAILED'`, when
  `CancelProcess` is called, then the action returns HTTP 200, the response body contains the
  instance with `Status = 'CANCELLED'`, and `EndedAt` is populated.

- [ ] **AC4 — Supersede happy path:** Given an instance with `Status = 'COMPLETED'`, when
  `SupersedeProcess` is called, then the action returns HTTP 200, the response body contains the
  instance with `Status = 'SUPERSEDED'`, and `EndedAt` retains its original completion
  timestamp (is NOT updated by supersede).

- [ ] **AC5 — Restart happy path:** Given an instance with `Status = 'FAILED'`, when
  `RestartProcess` is called, then the action returns HTTP 200, the response body contains the
  instance with `Status = 'RUNNING'` (synchronous transition), and the previously failed step's
  `RetryCount` is incremented by 1 (verifiable via the `_Step` child collection).

- [ ] **AC6 — Feature control: Execute disabled for non-NEW:** Given an instance with Status ≠
  `'NEW'` (e.g. `'RUNNING'`, `'FAILED'`, `'COMPLETED'`, `'CANCELLED'`, `'SUPERSEDED'`), when the
  OData entity is fetched with `$select=*` or via Fiori, then `ExecuteProcess` is returned as
  `Core.OperationAvailable = false` in the OData response annotations.

- [ ] **AC7 — Feature control: Cancel disabled for non-RUNNING/non-FAILED:** Given an instance
  with Status ∉ {`'RUNNING'`, `'FAILED'`}, then `CancelProcess` is returned as disabled.

- [ ] **AC8 — Feature control: Supersede disabled for non-COMPLETED:** Given an instance with
  Status ≠ `'COMPLETED'`, then `SupersedeProcess` is returned as disabled.

- [ ] **AC9 — Feature control: Restart disabled for non-FAILED:** Given an instance with Status
  ≠ `'FAILED'`, then `RestartProcess` is returned as disabled.

- [ ] **AC10 — Error surfacing — invalid status (Execute on COMPLETED):** Given an instance with
  `Status = 'COMPLETED'`, when `ExecuteProcess` is forced via direct OData POST (bypassing feature
  control), then the OData response contains an instance-level error message with the text from
  `ZFI_PROCESS` message 011 (`invalid_status`), referencing the specific instance ID.

- [ ] **AC11 — Error surfacing — double supersede:** Given an instance with `Status =
  'SUPERSEDED'`, when `SupersedeProcess` is forced via direct OData POST, then the OData response
  contains an instance-level error message from `ZFI_PROCESS` message 018 (`instance_superseded`).

- [ ] **AC12 — Multi-key action:** Given 3 instances selected simultaneously (2 with `Status =
  'NEW'`, 1 with `Status = 'RUNNING'`), when `ExecuteProcess` is triggered for all 3, then the
  2 `NEW` instances process successfully, the `RUNNING` instance returns an inline error message
  (msg 011), and all 3 result entries are returned in a single OData batch response.

- [ ] **AC13 — Actions visible in $metadata:** After re-activating the service binding, the OData
  `$metadata` document at `/sap/opu/odata4/.../zfiproc_ui_instance_o4/$metadata` contains
  `ExecuteProcess`, `CancelProcess`, `SupersedeProcess`, and `RestartProcess` as bound actions
  on the `ZFIPROC_C_INSTANCE_TP` entity type.

---

## Additional Context

### Dependencies

- No new class or DDIC dependencies — all required objects already exist
- `ZFI_PROCESS` message class: messages 011 (`invalid_status`) and 018 (`instance_superseded`)
  confirmed present
- `ZCL_FI_PROCESS_MANAGER` singleton accessible from the RAP LHC context (standard ABAP class,
  no restrictions)
- Projection BDEF `ZFIPROC_C_INSTANCE_TP` is a new artifact — must be created in ADT and
  committed via abapGit

### Testing Strategy

**Manual testing (primary — no ABAP Unit tests exist for the RAP layer):**

1. Activate objects in the order specified in the Activation Order section above
2. Re-activate `ZFIPROC_UI_INSTANCE_O4` service binding
3. Verify AC13 first: check `$metadata` for 4 new bound actions
4. Use Postman or the Fiori app `ZFIPROC_UI` to test ACs 1–5 (happy paths)
5. Verify feature control (ACs 6–9): check button states in Fiori list for instances in each
   status, or inspect OData response annotations
6. Test error paths (ACs 10–12): use direct OData POST to call actions on instances in invalid
   statuses

**Verify the async execute behaviour (AC1):**
After calling `ExecuteProcess` on a `NEW` instance, the response entity will show `Status = 'NEW'`.
Refresh the Fiori list after a few seconds — the status should update to `'RUNNING'` or `'PENDING'`
as bgRFC processes the queue. This is expected behaviour, not a bug.

### Notes

**Known pre-existing naming inconsistency:** `lhc_ZFIPROC_R_INSTANCE_TP` (mixed case) is the
existing LHC class name — it violates strict SAP uppercase naming conventions (Principle II) but
is already present in the codebase. Do not rename it in this ticket; a rename requires coordinated
BDEF regeneration. Note it for a future cleanup task.

**`PENDING` status and Cancel:** `zcl_fi_process_instance->cancel()` does NOT allow cancellation
from `PENDING` status (only `RUNNING` or `FAILED`). Feature control correctly disables
`CancelProcess` for `PENDING` instances. If the business requirement changes to allow cancelling
pending instances, `cancel()` in `zcl_fi_process_instance` must be updated separately.

**`QUEUED` status discrepancy:** A `gc_status-queued = 'QUEUED'` constant exists in
`zcl_fi_process_instance` but `'QUEUED'` is absent from the `ZFI_PROCESS_STATUS` DDIC domain
fixed values. This is a pre-existing issue — not introduced or resolved by this ticket.

**Future — UI button labels:** Action button labels in Fiori default to the action names
(`ExecuteProcess`, `CancelProcess`, etc.). User-friendly labels should be added via `@UI.lineItem`
/ `@UI.identification` annotations in the `.ddlx` metadata extension file
`zfiproc_c_instance_tp.ddlx.asddlxs` in a follow-up ticket.

**Constitution compliance summary:**
- Principle I (DDIC-First): DDIC types `zfi_process_instance_id`, `zfi_process_status` used
  throughout; no local TYPE definitions
- Principle II (SAP Standards): Line length ≤ 120 chars; pre-existing LHC naming noted above
- Principle IV (Factory Pattern): `zcl_fi_process_manager=>get_instance()` used exclusively
- Principle V (Error Handling): `zcx_fi_process_error` caught; `reported` and `failed` tables
  populated; `if_t100_dyn_msg` used for message construction
