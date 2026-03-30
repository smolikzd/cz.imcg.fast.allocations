---
title: 'Add Process Management Actions to Allocation Dashboard'
slug: 'est-134-alloc-dashboard-process-actions'
created: '2026-03-29'
revised: '2026-03-29'
status: 'done'
stepsCompleted: [1, 2, 3, 4]
baseline_commit: '75af5c163f1f144838f8007ff0ee88ac05b9de93'
tech_stack:
  - 'ABAP 7.58 (S/4HANA)'
  - 'RAP unmanaged BDEF on custom entity'
  - 'OData V4 (UI contract C1)'
  - 'Fiori Elements List Report'
  - 'ZCL_FI_PROCESS_MANAGER singleton pattern'
  - 'CL_APJ_RT_API for APJ job scheduling (Execute, Restart)'
files_to_modify:
  - 'zfi_alloc_s_action_buffer.tabl.xml (NEW - DDIC structure for action buffer entry)'
  - 'zfi_alloc_tt_action_buffer.ttyp.xml (NEW - DDIC table type for action buffer)'
  - 'zfi_i_alloc_dashboard_ce.bdef.asbdef (NEW - behavior definition)'
  - 'zbp_fi_alloc_dashboard_ce.clas.abap (NEW - behavior pool with handler + saver)'
code_patterns:
  - 'Unmanaged BDEF on custom entity with actions only (no CRUD)'
  - 'Two-phase pattern: handler validates + buffers, saver executes manager calls'
  - 'Behavior pool: lhc_dashboard handler class + lsc_dashboard saver class'
  - 'Saver inherits from cl_abap_behavior_saver_failed (enables failed/reported in save method for error surfacing)'
  - 'Singleton: zcl_fi_process_manager=>get_instance( )->xxx_process( )'
  - 'Dynamic instance feature control via get_instance_features handler method'
  - 'Error mapping: zcx_fi_process_error -> reported/failed (strict mode 2)'
  - 'Dashboard keys: FiscalYear, AllocationId, CompanyCode, FiscalPeriod (no alias)'
  - 'Child keys: CompanyCode, FiscalYear, FiscalPeriod, AllocationId, StepNumber'
  - 'ProcessInstanceId is a non-key field needed to call manager methods'
  - 'Lock handler: no-op FOR LOCK method required for unmanaged BDEF compilation'
test_patterns:
  - 'Manual testing via Fiori Elements preview (no ABAP Unit in project)'
---

# Tech-Spec: Add Process Management Actions to Allocation Dashboard

**Created:** 2026-03-29
**Revised:** 2026-03-29 (post-adversarial review round 2 -- 12 additional findings addressed)
**Linear Issues:** EST-134 (consolidated from EST-111, EST-112, EST-113, EST-114)
**Target Repository:** `cz.imcg.fast.ovysledovka`

## Overview

### Problem Statement

The Allocation Dashboard (`ZFI_I_ALLOC_DASHBOARD_CE`) is a read-only custom entity that displays allocation process instances. Users currently have no way to control the process lifecycle (execute, cancel, supersede, restart) from the dashboard UI -- they must use other tools or backend transactions. This creates friction and reduces the dashboard's utility as a central operations cockpit.

### Solution

Add four instance-bound RAP actions (ExecuteProcess, CancelProcess, SupersedeProcess, RestartProcess) to the dashboard custom entity by creating an **unmanaged behavior definition** with a behavior pool. Each action uses a **two-phase pattern**: the handler validates the request and buffers it; the saver's `save` method executes the actual `ZCL_FI_PROCESS_MANAGER` call. This architecture is required because the manager methods contain `COMMIT WORK AND WAIT`, which is **forbidden in RAP handler methods** but **permitted in RAP saver methods** (per SAP General RAP BO Implementation Contract). Dynamic instance feature control ensures actions are only available when the process status permits them.

### Key Architecture Decision: Two-Phase Handler/Saver Pattern

The `ZCL_FI_PROCESS_MANAGER` methods (`cancel_process`, `supersede_process`, `request_execute_process`, `request_restart_process`) all call `save_instance()` internally, which executes `COMMIT WORK AND WAIT`. Per the SAP General RAP BO Implementation Contract:

- **Handler methods**: `COMMIT WORK` is **FORBIDDEN** (causes runtime error or undefined behavior)
- **Saver methods**: `COMMIT WORK` is **NOT forbidden** (saver is responsible for persistence in unmanaged BOs)

Therefore, the architecture uses:
1. **Handler** (`lhc_dashboard`): Validates action preconditions (ProcessInstanceId not initial, status valid), populates `failed`/`reported` for validation errors, and buffers valid action requests in a class-level static table.
2. **Saver** (`lsc_dashboard`): Inherits from **`CL_ABAP_BEHAVIOR_SAVER_FAILED`** (not the standard `CL_ABAP_BEHAVIOR_SAVER`). This SAP-provided alternative base class gives the `save` method `failed` and `reported` response parameters, enabling errors to be surfaced to the Fiori UI even from the late save phase. In the `save` method, reads the buffer and calls the appropriate `ZCL_FI_PROCESS_MANAGER` method. Errors from the manager are caught and mapped to `failed`/`reported` using `new_message_from_exception()`.

> **Why `CL_ABAP_BEHAVIOR_SAVER_FAILED`?** SAP docs state: *"The base class CL_ABAP_BEHAVIOR_SAVER_FAILED can be used in exceptional cases where the basic rule [failures must not occur in the late save phase] cannot be met."* Our case qualifies because the external manager methods can fail (invalid status, APJ scheduling failure, optimistic lock conflict) and those errors must reach the user.

#### Double-Commit Interaction Analysis

The manager methods internally call `COMMIT WORK AND WAIT`. The RAP framework also issues a commit after the saver's `save` method returns successfully. This creates a double-commit scenario:

- **If the manager call succeeds**: `COMMIT WORK AND WAIT` persists the status change. The RAP framework's subsequent commit is effectively a no-op (nothing left to commit).
- **If the manager call fails** (raises `zcx_fi_process_error`): The saver populates `failed`/`reported`. When the framework detects errors in `failed`, it performs a **ROLLBACK** instead of a commit. However, if the manager already did a partial `COMMIT WORK AND WAIT` before failing, that commit is already persisted and cannot be rolled back by the framework. In practice, the manager methods are atomic: they either succeed fully (commit) or fail before committing (no partial state).
- **APJ scheduling edge case**: `request_execute_process` and `request_restart_process` call `cl_apj_rt_api=>schedule_job()` which may trigger an implicit commit. If the subsequent `save_instance()` fails, the job may be scheduled but the status not updated. The backend's optimistic lock check mitigates this. Test this scenario early.

> **Note on `request_execute_process` and `request_restart_process`:** These methods schedule APJ jobs via `cl_apj_rt_api=>schedule_job()` with a +2 second delay (to avoid implicit commit from `start_immediately`). The `schedule_job()` call itself may trigger an implicit commit. This is acceptable in the saver phase but should be tested early.

### Scope

**In Scope:**
- Create unmanaged BDEF for `ZFI_I_ALLOC_DASHBOARD_CE` with 4 actions
- Create behavior pool class (`ZBP_FI_ALLOC_DASHBOARD_CE`) with handler + saver
- Implement dynamic feature control (actions enabled/disabled per ProcessStatus)
- Two-phase pattern: handler validates + buffers, saver executes manager methods
- Error handling: translate `ZCX_FI_PROCESS_ERROR` to RAP `reported`/`failed`
- Feature control rules per backend status guards:
  - Execute = `NEW`
  - Cancel = `RUNNING`, `FAILED`, `EXECREQ`, `RESTREQ`
  - Supersede = `COMPLETED`
  - Restart = `FAILED`, `CANCELLED`

**Out of Scope:**
- Modifying the CDS custom entity definition itself
- Adding new CDS fields or annotations
- CRUD operations (dashboard remains read-only for data)
- Authorization checks (deferred -- `authorization master (none)` initially)
- Child entity `ZFI_I_ALLOC_DASH_STEP_CE` actions
- Confirmation dialogs in the UI (can be added later via annotations)

## Context for Development

### Codebase Patterns

- Dashboard is a **custom entity** with `@ObjectModel.query.implementedBy: 'ABAP:ZCL_FI_ALLOC_DASH_QUERY'`
- Custom entities are inherently read-only; an **unmanaged BDEF** is needed to add actions
- **No existing BDEFs** in the ovysledovka repo -- this will be the first
- Planner repo has one **managed** BDEF (`ZFIPROC_R_INSTANCE_TP`) -- different paradigm, not a template
- `ZCL_FI_PROCESS_MANAGER` uses **singleton pattern**: `get_instance()` (not factory method)
- Manager method signatures:
  - `request_execute_process( iv_instance_id )` -- schedules APJ job, sets status `EXECREQ`; guard: `NEW` only
  - `request_restart_process( iv_instance_id )` -- schedules APJ job, sets status `RESTREQ`; guard: `FAILED` or `CANCELLED`
  - `cancel_process( iv_instance_id )` -- best-effort APJ cancellation, sets status `CANCELLED`; guard: `RUNNING`, `FAILED`, `EXECREQ`, `RESTREQ`
  - `supersede_process( iv_instance_id )` -- sets status `SUPERSEDED`; guard: `COMPLETED` only
- All methods share: `IMPORTING iv_instance_id TYPE zfi_process_instance_id RAISING zcx_fi_process_error`
- All methods call `save_instance()` internally which does `COMMIT WORK AND WAIT`
- `request_execute_process` and `request_restart_process` also call `cl_apj_rt_api=>schedule_job()` with +2 second delay
- Backend `request_execute` and `request_restart` include optimistic lock checks (re-read DB status after scheduling to detect concurrent changes)
- Exception class `ZCX_FI_PROCESS_ERROR` implements `if_t100_message` + `if_t100_dyn_msg`
- Dashboard entity has composition child: `_Steps : composition [0..*] of ZFI_I_ALLOC_DASH_STEP_CE`
- Status domain `ZFI_PROCESS_STATUS` (CHAR 20): NEW, RUNNING, COMPLETED, FAILED, CANCELLED, PENDING, SKIPPED, SUPERSEDED, EXECREQ, RESTREQ
- Service binding: `ZFI_UI_ALLOC_DASHBOARD_O4_UI` (OData V4, UI contract C1, published)
- `ProcessInstanceId` is initial when no process instance has been created yet for a given allocation period/company combination. Feature control disables all actions when initial.

**Dashboard entity keys** (composite, no alias):
| Key | Data Element |
|-----|-------------|
| `FiscalYear` | `gjahr` |
| `AllocationId` | `zfi_alloc_id` |
| `CompanyCode` | `bukrs` |
| `FiscalPeriod` | `fins_fiscalperiod` |

**Child entity keys** (5 keys, parent assoc `_Dashboard`):
| Key | Data Element | Role |
|-----|-------------|------|
| `CompanyCode` | `bukrs` | Parent key |
| `FiscalYear` | `gjahr` | Parent key |
| `FiscalPeriod` | `fins_fiscalperiod` | Parent key |
| `AllocationId` | `zfi_alloc_id` | Parent key |
| `StepNumber` | `zfi_process_step_number` | Step's own key |

### Files to Reference

| File | Purpose |
| ---- | ------- |
| `zfi_i_alloc_dashboard_ce.ddls.asddls` | Parent custom entity CDS (203 lines, 4 key fields) |
| `zfi_i_alloc_dash_step_ce.ddls.asddls` | Child step entity CDS (108 lines, 5 key fields, parent `_Dashboard`) |
| `zcl_fi_alloc_dash_query.clas.abap` | Parent query provider (762 lines) |
| `zcl_fi_alloc_dash_step_qry.clas.abap` | Child step query provider (366 lines) |
| `zfi_ui_alloc_dashboard.srvd.srvdsrv` | Service definition (exposes both entities) |
| `zfi_ui_alloc_dashboard_o4_ui.srvb.xml` | Service binding (OData V4, UI C1, published) |
| `zcl_fi_process_manager.clas.abap` | Singleton with 4 target methods (planner repo) |
| `zcl_fi_process_instance.clas.abap` | Instance class -- `save_instance()` at line 617, `COMMIT WORK AND WAIT` at line 626 (planner repo) |
| `zcx_fi_process_error.clas.abap` | Exception class (263 lines, planner repo) |

### Technical Decisions

- **Unmanaged BDEF** (not managed): Custom entities cannot use managed implementation
- **No CRUD operations**: Only actions defined; dashboard data comes from query provider
- **`strict(2)`**: Follow project convention for strict mode
- **`authorization master (none)`**: No auth checks initially; can be added in future sprint
- **`lock master`**: Required by BDEF syntax; lock handler `FOR LOCK` will be a no-op (read-only entity, no concurrent data modification)
- **Two-phase handler/saver pattern**: Handler validates + buffers; saver executes manager calls. Required because `COMMIT WORK` is forbidden in handler methods but allowed in saver methods.
- **Singleton pattern**: Use `zcl_fi_process_manager=>get_instance( )` (Constitution Principle IV -- singleton is the existing pattern, not `NEW()`)
- **Error surfacing in handler**: Validation errors (ProcessInstanceId initial, status mismatch caught by feature control) use `new_message_from_exception()` to populate `reported`/`failed` tables (strict mode 2 requirement)
- **Error surfacing in saver**: The saver inherits from `CL_ABAP_BEHAVIOR_SAVER_FAILED`, which provides `failed` and `reported` response parameters in the `save` method. When a manager method raises `zcx_fi_process_error`, the saver catches it and maps it to `failed`/`reported` using `new_message_from_exception()`. The RAP framework then rolls back instead of committing, and the error message appears in the Fiori message popover. This is the SAP-recommended approach for savers that call external APIs which can fail.
- **Service definition**: No changes needed -- actions are auto-exposed once BDEF is created for an already-exposed entity
- **`ProcessInstanceId`**: Non-key field on dashboard entity; action handlers must `READ ENTITIES` to retrieve it. If `READ ENTITIES` does not work for custom entities within the own handler, fall back to direct `SELECT` from the underlying DB table or call the query provider directly.
- **No ETag**: Not used initially (actions-only, no CRUD/update operations). If `strict(2)` compilation requires an ETag even for actions-only BDEFs, add a dummy ETag field as a workaround. Document this as a potential compilation issue.
- **Child entity**: No separate implementation class needed. The child entity's `lock dependent by _Dashboard` + `authorization dependent by _Dashboard` delegates all behavior to the parent. The parent's behavior pool handles everything.
- **Race conditions**: Backend `request_execute` and `request_restart` already include optimistic lock checks (re-read DB status after APJ scheduling). For `cancel` and `supersede`, the status guard in the backend method rejects concurrent conflicting changes. No additional RAP-level locking needed beyond the no-op lock handler.

## Implementation Plan

### Tasks

- [ ] **Task 0: Create DDIC Objects for Action Buffer (Constitution Principle I)**
  - Files:
    - `src/zfi_alloc_process/zfi_alloc_s_action_buffer.tabl.xml` (NEW)
    - `src/zfi_alloc_process/zfi_alloc_tt_action_buffer.ttyp.xml` (NEW)
  - Action: Create DDIC structure and table type for the handler-to-saver action buffer. This satisfies Constitution Principle I (DDIC-First: no local TYPE definitions).

  **Structure `ZFI_ALLOC_S_ACTION_BUFFER`:**
  | Field | Data Element | Description |
  |-------|-------------|-------------|
  | `FISCAL_YEAR` | `GJAHR` | Fiscal year (dashboard key) |
  | `ALLOCATION_ID` | `ZFI_ALLOC_ID` | Allocation ID (dashboard key) |
  | `COMPANY_CODE` | `BUKRS` | Company code (dashboard key) |
  | `FISCAL_PERIOD` | `FINS_FISCALPERIOD` | Fiscal period (dashboard key) |
  | `INSTANCE_ID` | `ZFI_PROCESS_INSTANCE_ID` | Process instance ID (from non-key field) |
  | `ACTION` | `CHAR10` | Action identifier: EXECUTE, CANCEL, SUPERSEDE, RESTART |

  **Table Type `ZFI_ALLOC_TT_ACTION_BUFFER`:**
  - Line type: `ZFI_ALLOC_S_ACTION_BUFFER`
  - Table kind: `STANDARD TABLE`
  - Key: empty (no primary key)

  - Notes:
    - The `ACTION` field uses `CHAR10` (not STRING) for DDIC compatibility. Values: `'EXECUTE'`, `'CANCEL'`, `'SUPERSEDE'`, `'RESTART'` -- all fit within 10 characters.
    - These objects are transient (used only as an in-memory buffer between handler and saver), but Constitution Principle I requires DDIC types regardless.
    - Package: `ZFI_ALLOC_PROCESS` (same as all dashboard objects)

- [ ] **Task 1: Create Behavior Definition for Dashboard Entity**
  - File: `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.bdef.asbdef` (NEW)
  - Action: Create unmanaged BDEF with the following structure:
    ```
    unmanaged implementation in class zbp_fi_alloc_dashboard_ce unique;
    strict(2);

    define behavior for ZFI_I_ALLOC_DASHBOARD_CE
    lock master
    authorization master ( none )
    {
      internal read;

      action ( features: instance ) ExecuteProcess external 'executeProcess';
      action ( features: instance ) CancelProcess external 'cancelProcess';
      action ( features: instance ) SupersedeProcess external 'supersedeProcess';
      action ( features: instance ) RestartProcess external 'restartProcess';

      association _Steps;
    }

    define behavior for ZFI_I_ALLOC_DASH_STEP_CE
    lock dependent by _Dashboard
    authorization dependent by _Dashboard
    {
      association _Dashboard;
    }
    ```
  - Notes:
    - No CRUD operations -- only actions and the composition association
    - All 4 actions are instance-bound (not static) with `features:instance` for dynamic feature control
    - `external` names use camelCase for OData/Fiori convention
    - No `result` or `parameter` -- these are fire-and-forget state-changing actions
    - **Both parent and child behaviors are defined in this single BDEF file.** The child entity needs a minimal behavior stub because composition requires it; no separate implementation class needed for the child.
    - No `field` or `mapping` clauses needed -- no CRUD means no field control
    - `internal read;` may be needed under `strict(2)` for `READ ENTITIES` to work within the handler. If compilation fails without it, add it. If compilation fails with it (because custom entities already have a query provider), remove it. Test both.
    - **No ETag clause**: Not needed for actions-only BDEF. If `strict(2)` requires it, add a dummy ETag as workaround.

- [ ] **Task 2: Create Behavior Pool Class (Global Class Shell)**
  - File: `src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.abap` (NEW)
  - Action: Create the global class definition. The class itself is empty -- all logic lives in local classes (handler + saver):
    ```abap
    "! <p class="shorttext synchronized">Behavior pool for ZFI_I_ALLOC_DASHBOARD_CE</p>
    "! Handles process management actions (Execute, Cancel, Supersede, Restart)
    "! on the Allocation Dashboard custom entity.
    CLASS zbp_fi_alloc_dashboard_ce DEFINITION
      PUBLIC
      ABSTRACT
      FINAL
      FOR BEHAVIOR OF zfi_i_alloc_dashboard_ce.
    ENDCLASS.

    CLASS zbp_fi_alloc_dashboard_ce IMPLEMENTATION.
    ENDCLASS.
    ```
  - Notes: Standard RAP behavior pool pattern. `ABSTRACT FINAL` prevents instantiation.

- [ ] **Task 3: Implement Local Handler Class -- Actions + Lock + Feature Control**
  - File: `src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` (NEW)
  - Action: Create local handler class `lhc_dashboard` with 4 action methods + 1 feature control method + 1 lock method:

  **Class definition:**
    ```abap
    "! <p class="shorttext synchronized">Handler for ZFI_I_ALLOC_DASHBOARD_CE</p>
    "! Validates action preconditions and buffers valid requests for the saver.
    CLASS lhc_dashboard DEFINITION
      INHERITING FROM cl_abap_behavior_handler.

      PRIVATE SECTION.
        "! Static action buffer -- written by handler, read by saver
        CLASS-DATA gt_action_buffer TYPE zfi_alloc_tt_action_buffer.

        METHODS get_instance_features FOR INSTANCE FEATURES
          IMPORTING keys REQUEST requested_features
          FOR ZFI_I_ALLOC_DASHBOARD_CE
          RESULT result.

        METHODS lock FOR LOCK
          IMPORTING keys FOR LOCK ZFI_I_ALLOC_DASHBOARD_CE.

        METHODS executeprocess FOR MODIFY
          IMPORTING keys FOR ACTION ZFI_I_ALLOC_DASHBOARD_CE~ExecuteProcess.

        METHODS cancelprocess FOR MODIFY
          IMPORTING keys FOR ACTION ZFI_I_ALLOC_DASHBOARD_CE~CancelProcess.

        METHODS supersedeprocess FOR MODIFY
          IMPORTING keys FOR ACTION ZFI_I_ALLOC_DASHBOARD_CE~SupersedeProcess.

        METHODS restartprocess FOR MODIFY
          IMPORTING keys FOR ACTION ZFI_I_ALLOC_DASHBOARD_CE~RestartProcess.
    ENDCLASS.
    ```

  **Handler implementation pattern for each action method:**
    ```abap
    METHOD executeprocess.
      " 1. Read entity to get ProcessInstanceId
      READ ENTITIES OF ZFI_I_ALLOC_DASHBOARD_CE IN LOCAL MODE
        ENTITY ZFI_I_ALLOC_DASHBOARD_CE
        FIELDS ( ProcessInstanceId ProcessStatus )
        WITH CORRESPONDING #( keys )
        RESULT DATA(lt_dashboard)
        FAILED DATA(lt_read_failed).

      LOOP AT lt_dashboard INTO DATA(ls_dashboard).
        " 2. Validate ProcessInstanceId is not initial
        IF ls_dashboard-ProcessInstanceId IS INITIAL.
          APPEND VALUE #(
            %tky = ls_dashboard-%tky
            %fail-cause = if_abap_behv=>cause-unspecific
          ) TO failed-zfi_i_alloc_dashboard_ce.

          APPEND VALUE #(
            %tky = ls_dashboard-%tky
            %msg = new_message(
              id       = 'ZFI_PROCESS'
              number   = '030'
              severity = if_abap_behv_message=>severity-error
            )
          ) TO reported-zfi_i_alloc_dashboard_ce.
          CONTINUE.
        ENDIF.

        " 3. Buffer the action request for the saver phase
        APPEND VALUE #(
          fiscal_year   = ls_dashboard-FiscalYear
          allocation_id = ls_dashboard-AllocationId
          company_code  = ls_dashboard-CompanyCode
          fiscal_period = ls_dashboard-FiscalPeriod
          instance_id   = ls_dashboard-ProcessInstanceId
          action        = 'EXECUTE'
        ) TO gt_action_buffer.
      ENDLOOP.
    ENDMETHOD.
    ```
  - The other 3 action methods follow the same pattern, changing only the `action` value:
    - `cancelprocess` -> `action = 'CANCEL'`
    - `supersedeprocess` -> `action = 'SUPERSEDE'`
    - `restartprocess` -> `action = 'RESTART'`
  - Consider extracting a private helper method `buffer_action` to reduce duplication across the 4 methods.
  - **Message number note**: Message `ZFI_PROCESS/030` must exist or be created. If the message class `ZFI_PROCESS` does not have a suitable message for "no process instance", create a new message (e.g., number `030`) with text: `No process instance exists for this allocation`. Check existing messages in `ZFI_PROCESS` first -- if a suitable one already exists, use its number instead.

  **Lock handler (no-op):**
    ```abap
    "! <p class="shorttext synchronized">Lock handler (no-op)</p>
    "! Required for unmanaged BDEF compilation. No actual locking needed
    "! because the dashboard is a read-only custom entity -- concurrent
    "! data modification is not possible through this BO.
    METHOD lock.
      " No-op: Custom entity is read-only; actions delegate to
      " ZCL_FI_PROCESS_MANAGER which handles its own concurrency.
    ENDMETHOD.
    ```

  - Notes:
    - `READ ENTITIES ... IN LOCAL MODE` is used to read the entity's own fields within the handler
    - If `READ ENTITIES` does not work on custom entities, replace with direct `SELECT` from the underlying DB views/tables that the query provider uses. The query provider `ZCL_FI_ALLOC_DASH_QUERY` ultimately reads from `zfi_proc_inst` and joins with allocation config.
    - The static buffer `gt_action_buffer` is shared between handler and saver via `CLASS-DATA`. The saver reads and clears it in `save`.
    - The lock method `FOR LOCK` is required by the unmanaged BDEF -- without it, the BDEF will not compile. It is a no-op because the custom entity is read-only.

- [ ] **Task 4: Implement Local Handler Class -- Feature Control**
  - File: `src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` (same file as Task 3)
  - Action: Implement `get_instance_features` method:
    1. `READ ENTITIES ... IN LOCAL MODE` to get `ProcessStatus` and `ProcessInstanceId` for all requested keys
    2. Loop through results, set each action's feature control based on status:

    | Action | Enabled when ProcessStatus is | Backend guard |
    |--------|------------------------------|---------------|
    | `ExecuteProcess` | `NEW` | `request_execute`: NEW only |
    | `CancelProcess` | `RUNNING`, `FAILED`, `EXECREQ`, `RESTREQ` | `cancel`: RUNNING, FAILED, EXECREQ, RESTREQ |
    | `SupersedeProcess` | `COMPLETED` | `supersede`: COMPLETED only |
    | `RestartProcess` | `FAILED`, `CANCELLED` | `request_restart`: FAILED, CANCELLED |

    3. Set `%features-%action-XxxProcess` to `if_abap_behv=>fc-o-enabled` or `if_abap_behv=>fc-o-disabled`
    4. For rows with `ProcessInstanceId IS INITIAL`, all actions should be disabled

  - Implementation:
    ```abap
    "! <p class="shorttext synchronized">Dynamic instance feature control</p>
    "! Enables/disables action buttons based on ProcessStatus.
    "! Feature control rules match the backend manager method status guards exactly.
    METHOD get_instance_features.
      READ ENTITIES OF ZFI_I_ALLOC_DASHBOARD_CE IN LOCAL MODE
        ENTITY ZFI_I_ALLOC_DASHBOARD_CE
        FIELDS ( ProcessStatus ProcessInstanceId )
        WITH CORRESPONDING #( keys )
        RESULT DATA(lt_dashboard)
        FAILED DATA(lt_read_failed).

      LOOP AT lt_dashboard INTO DATA(ls_dashboard).
        DATA(lv_status) = ls_dashboard-ProcessStatus.

        " All actions disabled if no process instance exists
        IF ls_dashboard-ProcessInstanceId IS INITIAL.
          APPEND VALUE #(
            %tky = ls_dashboard-%tky
            %features-%action-ExecuteProcess   = if_abap_behv=>fc-o-disabled
            %features-%action-CancelProcess    = if_abap_behv=>fc-o-disabled
            %features-%action-SupersedeProcess = if_abap_behv=>fc-o-disabled
            %features-%action-RestartProcess   = if_abap_behv=>fc-o-disabled
          ) TO result.
          CONTINUE.
        ENDIF.

        APPEND VALUE #(
          %tky = ls_dashboard-%tky

          %features-%action-ExecuteProcess = COND #(
            WHEN lv_status = 'NEW'
            THEN if_abap_behv=>fc-o-enabled
            ELSE if_abap_behv=>fc-o-disabled )

          %features-%action-CancelProcess = COND #(
            WHEN lv_status = 'RUNNING'
              OR lv_status = 'FAILED'
              OR lv_status = 'EXECREQ'
              OR lv_status = 'RESTREQ'
            THEN if_abap_behv=>fc-o-enabled
            ELSE if_abap_behv=>fc-o-disabled )

          %features-%action-SupersedeProcess = COND #(
            WHEN lv_status = 'COMPLETED'
            THEN if_abap_behv=>fc-o-enabled
            ELSE if_abap_behv=>fc-o-disabled )

          %features-%action-RestartProcess = COND #(
            WHEN lv_status = 'FAILED'
              OR lv_status = 'CANCELLED'
            THEN if_abap_behv=>fc-o-enabled
            ELSE if_abap_behv=>fc-o-disabled )

        ) TO result.
      ENDLOOP.
    ENDMETHOD.
    ```

  - Notes:
    - Feature control determines button visibility/clickability in Fiori Elements
    - Rules are aligned **exactly** with the backend manager method status guards:
      - `request_execute`: guard is `NEW` only
      - `cancel`: guard is `RUNNING`, `FAILED`, `EXECREQ`, `RESTREQ`
      - `supersede`: guard is `COMPLETED` only
      - `request_restart`: guard is `FAILED`, `CANCELLED`
    - EXECREQ is intentionally NOT in the Execute feature control (already requested, don't allow re-request)
    - RESTREQ is intentionally NOT in the Restart feature control (same reasoning)
    - **Deliberate design decision (not an oversight)**: `EXECREQ` and `RESTREQ` are excluded from Execute and Restart feature control respectively. A process in `EXECREQ` status already has an APJ job scheduled -- allowing Execute again would either fail (backend guard) or cause duplicate job scheduling. The user should Cancel first if they want to re-execute. Similarly for `RESTREQ` and Restart.
    - For rows with `ProcessInstanceId IS INITIAL`, all actions are disabled (no process instance to act on)

- [ ] **Task 5: Implement Local Saver Class**
  - File: `src/zfi_alloc_process/zbp_fi_alloc_dashboard_ce.clas.locals_imp.abap` (same file as Tasks 3-4)
  - Action: Create local saver class `lsc_dashboard`:
    ```abap
    "! <p class="shorttext synchronized">Saver for ZFI_I_ALLOC_DASHBOARD_CE</p>
    "! Executes buffered action requests by calling ZCL_FI_PROCESS_MANAGER methods.
    "! Inherits from CL_ABAP_BEHAVIOR_SAVER_FAILED to enable error surfacing
    "! from the save phase via failed/reported response parameters.
    "! Manager methods contain COMMIT WORK AND WAIT internally, which is permitted
    "! in the saver phase (but forbidden in handler phase).
    CLASS lsc_dashboard DEFINITION
      INHERITING FROM cl_abap_behavior_saver_failed.

      PROTECTED SECTION.
        METHODS finalize          REDEFINITION.
        METHODS check_before_save REDEFINITION.
        METHODS save              REDEFINITION.
        METHODS cleanup           REDEFINITION.
        METHODS cleanup_finalize  REDEFINITION.
    ENDCLASS.

    CLASS lsc_dashboard IMPLEMENTATION.
      METHOD finalize.
        " No-op: All validation done in handler phase
      ENDMETHOD.

      METHOD check_before_save.
        " No-op: No additional checks needed
      ENDMETHOD.

      METHOD save.
        " Execute all buffered action requests.
        " The save method signature (from CL_ABAP_BEHAVIOR_SAVER_FAILED)
        " includes CHANGING failed and reported parameters, enabling
        " error messages to be surfaced to the Fiori UI.
        DATA(lo_manager) = zcl_fi_process_manager=>get_instance( ).

        LOOP AT lhc_dashboard=>gt_action_buffer INTO DATA(ls_action).
          TRY.
              CASE ls_action-action.
                WHEN 'EXECUTE'.
                  lo_manager->request_execute_process(
                    iv_instance_id = ls_action-instance_id ).
                WHEN 'CANCEL'.
                  lo_manager->cancel_process(
                    iv_instance_id = ls_action-instance_id ).
                WHEN 'SUPERSEDE'.
                  lo_manager->supersede_process(
                    iv_instance_id = ls_action-instance_id ).
                WHEN 'RESTART'.
                  lo_manager->request_restart_process(
                    iv_instance_id = ls_action-instance_id ).
              ENDCASE.
            CATCH zcx_fi_process_error INTO DATA(lx_error).
              " Map the exception to RAP failed/reported for Fiori UI display.
              " CL_ABAP_BEHAVIOR_SAVER_FAILED provides these parameters in save.
              APPEND VALUE #(
                %tky-FiscalYear   = ls_action-fiscal_year
                %tky-AllocationId = ls_action-allocation_id
                %tky-CompanyCode  = ls_action-company_code
                %tky-FiscalPeriod = ls_action-fiscal_period
                %fail-cause       = if_abap_behv=>cause-unspecific
              ) TO failed-zfi_i_alloc_dashboard_ce.

              APPEND VALUE #(
                %tky-FiscalYear   = ls_action-fiscal_year
                %tky-AllocationId = ls_action-allocation_id
                %tky-CompanyCode  = ls_action-company_code
                %tky-FiscalPeriod = ls_action-fiscal_period
                %msg              = new_message_from_exception(
                  excp     = lx_error
                  severity = if_abap_behv_message=>severity-error )
              ) TO reported-zfi_i_alloc_dashboard_ce.
          ENDTRY.
        ENDLOOP.
      ENDMETHOD.

      METHOD cleanup.
        " Clear the action buffer
        CLEAR lhc_dashboard=>gt_action_buffer.
      ENDMETHOD.

      METHOD cleanup_finalize.
        " Clear the action buffer (safety net)
        CLEAR lhc_dashboard=>gt_action_buffer.
      ENDMETHOD.
    ENDCLASS.
    ```
  - Notes:
    - The saver inherits from **`CL_ABAP_BEHAVIOR_SAVER_FAILED`** (not `CL_ABAP_BEHAVIOR_SAVER`). This SAP-provided alternative base class provides `failed` and `reported` CHANGING parameters in the `save` method, enabling error messages to reach the Fiori UI even from the late save phase.
    - The `save` method reads the static buffer `lhc_dashboard=>gt_action_buffer` populated by the handler
    - Manager methods do their own `COMMIT WORK AND WAIT` -- this is safe in the saver phase
    - `request_execute_process` and `request_restart_process` additionally call `cl_apj_rt_api=>schedule_job()`, which may trigger implicit commits. This is acceptable in the saver.
    - Error handling: When a manager method raises `zcx_fi_process_error`, the saver catches it and maps it to `failed`/`reported` using `new_message_from_exception()`. The RAP framework then rolls back instead of committing. The T100 message from the exception is displayed in the Fiori message popover.
    - `cleanup` and `cleanup_finalize` both clear the buffer to prevent stale entries from leaking across requests
    - **Partial execution note**: If multiple actions are buffered and one fails, the preceding successful actions (already committed by the manager) cannot be rolled back. This is acceptable because each action operates on an independent process instance. The failed action's error is surfaced to the user.
    - **`supersede_process()` quirk**: This method uses textid `instance_superseded` for its success path, but raises with `invalid_status` (same as other methods) when the status guard fails. However, unlike the other 3 methods, if `supersede` raises an unexpected error, the textid may differ. The `new_message_from_exception()` call handles this transparently since it reads whatever T100 message the exception carries.

- [ ] **Task 6: Re-activate Service Binding**
  - File: `zfi_ui_alloc_dashboard_o4_ui` (service binding in ADT)
  - Action: After creating the BDEF and behavior pool:
    1. Open service binding `ZFI_UI_ALLOC_DASHBOARD_O4_UI` in ADT
    2. Unpublish the service binding
    3. Re-publish the service binding
    4. Verify the 4 actions appear in the service binding's entity set metadata
  - Notes: Service definition does NOT need changes -- actions on already-exposed entities are auto-included. Only the binding needs re-activation to pick up the new BDEF.

- [ ] **Task 7: Verify in Fiori Elements Preview**
  - Action: Open the Fiori Elements preview from the service binding:
    1. Verify action buttons appear in the list report toolbar
    2. Select a row with status `NEW` -- verify only `ExecuteProcess` is enabled
    3. Select a row with status `FAILED` -- verify `CancelProcess` and `RestartProcess` are enabled
    4. Select a row with status `COMPLETED` -- verify only `SupersedeProcess` is enabled
    5. Select a row with status `RUNNING` -- verify only `CancelProcess` is enabled
    6. Select a row with status `CANCELLED` -- verify only `RestartProcess` is enabled
    7. Select a row with status `EXECREQ` -- verify only `CancelProcess` is enabled
    8. Select a row with status `RESTREQ` -- verify only `CancelProcess` is enabled
    9. Select a row where `ProcessInstanceId` is initial -- verify all actions disabled
    10. Execute `ExecuteProcess` on a `NEW` row -- verify no error; status should change to `EXECREQ` after refresh
    11. Execute `CancelProcess` on a `RUNNING` row -- verify no error; status should change to `CANCELLED` after refresh
    12. Execute `SupersedeProcess` on a `COMPLETED` row -- verify no error; status changes to `SUPERSEDED` after refresh
    13. Execute `RestartProcess` on a `FAILED` row -- verify no error; status should change to `RESTREQ` after refresh

### Acceptance Criteria

- [ ] **AC 1:** Given the dashboard is loaded in Fiori Elements, when a user selects a row with `ProcessStatus = 'NEW'`, then only the `ExecuteProcess` action button is enabled (other 3 are disabled/hidden).

- [ ] **AC 2:** Given a row with `ProcessStatus = 'FAILED'` is selected, when the user clicks `CancelProcess`, then `zcl_fi_process_manager=>get_instance( )->cancel_process( )` is called with the row's `ProcessInstanceId` and no error message appears.

- [ ] **AC 3:** Given a row with `ProcessStatus = 'FAILED'` is selected, when the user clicks `RestartProcess`, then `zcl_fi_process_manager=>get_instance( )->request_restart_process( )` is called with the row's `ProcessInstanceId` and no error message appears.

- [ ] **AC 4:** Given a row with `ProcessStatus = 'COMPLETED'` is selected, when the user clicks `SupersedeProcess`, then `zcl_fi_process_manager=>get_instance( )->supersede_process( )` is called with the row's `ProcessInstanceId` and no error message appears.

- [ ] **AC 5:** Given a row with `ProcessStatus = 'NEW'` is selected, when the user clicks `ExecuteProcess`, then `zcl_fi_process_manager=>get_instance( )->request_execute_process( )` is called with the row's `ProcessInstanceId` and no error message appears. Status changes to `EXECREQ`; the actual process execution runs as an APJ job asynchronously.

- [ ] **AC 6:** Given a row with `ProcessStatus = 'RUNNING'`, when the user views the action toolbar, then only `CancelProcess` is enabled; `ExecuteProcess`, `SupersedeProcess`, and `RestartProcess` are disabled.

- [ ] **AC 7:** Given a row where the `ProcessInstanceId` is initial/empty (no process instance created yet for this allocation period/company combination), when the user views the action toolbar, then all 4 action buttons are disabled.

- [ ] **AC 8:** Given the manager method raises `zcx_fi_process_error` during any phase (handler validation or saver execution), the error message appears in the Fiori message popover with the T100 message text, and the `failed` table is populated (strict mode 2 compliance). The saver uses `CL_ABAP_BEHAVIOR_SAVER_FAILED` which enables `failed`/`reported` population even in the late save phase.

- [ ] **AC 9:** Given no rows are selected, when the user views the dashboard, then all 4 action buttons are disabled (instance-bound actions require selection).

- [ ] **AC 10:** Given a row with `ProcessStatus = 'CANCELLED'`, when the user views the action toolbar, then only `RestartProcess` is enabled.

- [ ] **AC 11:** Given a row with `ProcessStatus = 'EXECREQ'` or `'RESTREQ'`, when the user views the action toolbar, then only `CancelProcess` is enabled.

## Additional Context

### Dependencies

- `ZCL_FI_PROCESS_MANAGER` (planner repo) -- singleton with `get_instance()`, all 4 methods must exist
- `ZCX_FI_PROCESS_ERROR` (planner repo) -- exception class must be accessible cross-package
- `CL_APJ_RT_API` (SAP standard) -- used internally by `request_execute_process` and `request_restart_process` for APJ job scheduling
- Service binding `ZFI_UI_ALLOC_DASHBOARD_O4_UI` must be re-activated after BDEF creation
- ADT required for BDEF creation, class creation, and service binding re-activation
- No changes needed in the planner repo

### Testing Strategy

- **No ABAP Unit tests** -- project convention, manual testing only
- **Fiori Elements Preview** from service binding is the primary test vehicle
- **Test matrix by status** (aligned with backend manager method guards):

| ProcessStatus | Execute | Cancel | Supersede | Restart | Notes |
|---------------|---------|--------|-----------|---------|-------|
| NEW           | Enabled | -      | -         | -       | Only action: schedule APJ execute |
| EXECREQ       | -       | Enabled| -         | -       | APJ job scheduled, can cancel |
| RUNNING       | -       | Enabled| -         | -       | Process running, can cancel |
| COMPLETED     | -       | -      | Enabled   | -       | Only action: supersede |
| FAILED        | -       | Enabled| -         | Enabled | Can cancel or restart |
| CANCELLED     | -       | -      | -         | Enabled | Can restart |
| SUPERSEDED    | -       | -      | -         | -       | Terminal state, no actions |
| PENDING       | -       | -      | -         | -       | Step-level status, no actions |
| SKIPPED       | -       | -      | -         | -       | Step-level status, no actions |
| RESTREQ       | -       | Enabled| -         | -       | APJ restart scheduled, can cancel |
| (initial)     | -       | -      | -         | -       | No process instance, all disabled |

### Compilation Troubleshooting

If the BDEF does not compile under `strict(2)`, try the following in order:

1. **`internal read;`**: If `READ ENTITIES ... IN LOCAL MODE` fails with "entity not readable", add `internal read;` to the behavior definition. If this causes a different error (e.g., custom entity already has a query provider and doesn't need `read`), remove it.
2. **ETag**: If `strict(2)` requires an ETag even for actions-only BDEFs, add `etag master LastChangedAt` to the behavior definition and ensure `LastChangedAt` field exists on the entity.
3. **Lock handler**: If the `FOR LOCK` method signature doesn't match expectations, compare with SAP docs example for unmanaged lock handlers.

### Notes

- **Two-phase pattern (handler/saver) is mandatory**: The `COMMIT WORK AND WAIT` inside `ZCL_FI_PROCESS_MANAGER` methods is forbidden in RAP handler methods but allowed in saver methods. This is confirmed by the SAP General RAP BO Implementation Contract. Do NOT attempt to call manager methods directly in the handler.
- **APJ job scheduling (Execute/Restart):** `request_execute_process()` and `request_restart_process()` schedule APJ jobs via `cl_apj_rt_api=>schedule_job()` with a +2 second delay. The status changes to `EXECREQ`/`RESTREQ` immediately; the actual process runs asynchronously. The user must refresh the dashboard to see the final status.
- **Saver error handling**: The saver inherits from `CL_ABAP_BEHAVIOR_SAVER_FAILED`, which provides `failed` and `reported` CHANGING parameters in the `save` method. When a manager method raises `zcx_fi_process_error`, the exception is caught and mapped using `new_message_from_exception()`. The T100 message text appears in the Fiori message popover. The RAP framework performs a rollback instead of a commit when errors are reported. Note: if the manager already committed before failing (unlikely given the atomic design of the manager methods), that commit cannot be rolled back by the framework.
- **`supersede_process()` textid quirk**: Unlike `cancel`, `request_execute`, and `request_restart` (which all use `invalid_status` textid when the status guard fails), `supersede_process()` uses `instance_superseded` textid for its success path. This is not a problem -- the `new_message_from_exception()` in the saver reads whatever T100 message the exception carries, regardless of textid. But be aware of this when debugging unexpected messages.
- **Constitution compliance:**
  - Principle I (DDIC-First): Action buffer uses DDIC structure `ZFI_ALLOC_S_ACTION_BUFFER` and table type `ZFI_ALLOC_TT_ACTION_BUFFER` (see Task 0). No local TYPE definitions.
  - Principle II (SAP Standards): SAP naming, line length ≤120, ABAP-Doc on public methods of global classes (local handler/saver classes use shorttext synchronized comments per RAP convention)
  - Principle III (Consult SAP Docs): Done -- RAP action syntax, feature control, unmanaged BDEF patterns, COMMIT WORK restrictions, lock handler requirements confirmed via SAP Docs MCP
  - Principle IV (Factory/Singleton): Uses `zcl_fi_process_manager=>get_instance()` singleton
  - Principle V (Error Handling): `zcx_fi_process_error` caught in handler (validation) and saver (execution); both phases map errors to RAP `reported`/`failed` using `new_message_from_exception()` for Fiori UI display
- **Future enhancements** (out of scope):
  - Confirmation dialogs before destructive actions (Cancel, Supersede)
  - Authorization checks (`authorization master (instance)`)
  - Result return (`result [1] $self`) to refresh the row inline after action
  - Actions on child step entity
  - Application log (BAL) integration for additional operational logging beyond Fiori message popover
