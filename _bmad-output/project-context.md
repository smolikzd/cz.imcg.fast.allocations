---
project_name: 'cz.imcg.fast.allocations'
user_name: 'Zdenek'
date: '2026-04-01'
sections_completed: ['technology_stack', 'language_rules', 'framework_rules', 'testing_rules', 'quality_rules', 'workflow_rules', 'anti_patterns']
status: 'complete'
rule_count: 58
optimized_for_llm: true
---

# Project Context for AI Agents

_This file contains critical rules and patterns that AI agents must follow when implementing code in this project. Focus on unobvious details that agents might otherwise miss._

---

## Technology Stack & Versions

| Technology | Version / Details |
|---|---|
| ABAP | 7.58 (SAP S/4HANA compatible, on-premise) |
| SAP DDIC | DDIC-first default — prefer DDIC types, local types acceptable with justification |
| ZFI_PROCESS framework | Custom orchestration framework (cz.imcg.fast.planner) |
| Background processing | bgRFC (queued mode for PHASE2 substeps) |
| APJ (Application Jobs) | SAP Application Jobs — Mode 2 generic + Mode 3 process-type-specific |
| abapGit | XML format for source control |
| ABAP linting | abaplint.json enforces 120-char line length |
| Logging | SAP BAL via ZIF_FI_PROCESS_LOGGER / ZCL_FI_PROCESS_LOGGER (fail-hard, flush interval configurable) |
| Message system | T100 message classes: ZFI_PROCESS (framework), ZFI_ALLOC (allocation domain) |
| Database | SAP HANA (client-dependent tables, MANDT key field required) |
| Fiori / OData | Fiori Elements — RAP unmanaged strict(2), OData V4, custom entities |
| RAP pattern | Unmanaged BDEF with behavior pool, feature control, side effects |

**SAP Tools / Transactions used in dev workflow:**
- SE91 — message class maintenance
- SE63 — message translation
- SLG1 — application log viewer
- abapGit XML — source control

---

## Repository Structure

This is a **planning-only** repository. ABAP code lives in two separate repos:

| Repo | Purpose | Key Package |
|---|---|---|
| `cz.imcg.fast.planner` | ZFI_PROCESS orchestration framework | `ZFI_PROCESS` |
| `cz.imcg.fast.ovysledovka` | Allocation step implementations + Fiori dashboard | `ZFI_ALLOC_PROCESS` |

Do **not** generate ABAP code in this repository. See `AGENTS.md §7`.

---

## Critical Implementation Rules

### Rule 1 — DDIC-First (Constitution Principle I) — Default, Not Absolute

- **Prefer** DDIC table types for method signatures and shared data structures
- **Required** for: types used across class boundaries, in method signatures, or stored in DB tables
- **Acceptable** local `TYPE` for: purely internal helpers (never leave the method/class), loop variables, intermediate parsing structures with no reuse value

```abap
" ✅ CORRECT — DDIC type for method signature (cross-boundary)
DATA lt_steps TYPE zfi_process_tt_proc_step.

" ✅ ALSO ACCEPTABLE — local helper for internal-only parsing use
TYPES: BEGIN OF ty_kv_pair,
         name  TYPE string,
         value TYPE string,
       END OF ty_kv_pair.
TYPES tt_kv_pairs TYPE STANDARD TABLE OF ty_kv_pair WITH EMPTY KEY.

" ❌ WRONG — local type where a DDIC type exists or should exist
TYPES: BEGIN OF ty_step,
         step_number TYPE i,
       END OF ty_step.
DATA lt_steps TYPE STANDARD TABLE OF ty_step.  " use ZFI_PROCESS_TT_PROC_STEP
```

---

### Rule 2 — Line Length ≤ 120 Characters (abaplint enforced) ⚠️ CRITICAL

- Hard limit: **120 chars** for `.abap` files (enforced by abaplint)
- Absolute ABAP parser limit: **255 chars** (causes cryptic "Field X is unknown" parse errors)
- **Wrap** VALUE constructors, method calls, RAISE EXCEPTION at logical boundaries

```abap
" ✅ CORRECT — wrapped VALUE constructor
DATA(ls_ctx) = VALUE ty_structure(
  field1 = value1
  field2 = value2
).

" ✅ CORRECT — wrapped method call
update_status(
  iv_param1 = value1
  iv_param2 = value2
).

" ✅ CORRECT — wrapped RAISE EXCEPTION
RAISE EXCEPTION TYPE zcx_fi_process_error
  EXPORTING
    textid = zcx_fi_process_error=>validation_failed
    value  = lv_company_code.

" ❌ WRONG — single long line
DATA(ls_ctx) = VALUE ty_structure( field1 = value1 field2 = value2 field3 = value3 field4 = value4 ).
```

---

### Rule 3 — Factory Pattern, No Direct `NEW` for Framework Classes (Constitution Principle IV)

- `ZCL_FI_PROCESS_MANAGER`, `ZCL_FI_PROCESS_INSTANCE`, `ZCL_FI_PROCESS_DEFINITION` have **private constructors**
- Always use factory methods: `GET_INSTANCE`, `CREATE`, `LOAD`
- Step implementation classes (implementing `ZIF_FI_PROCESS_STEP`) MAY use public constructors

```abap
" ❌ WRONG
DATA(lo_instance) = NEW zcl_fi_process_instance( ).

" ✅ CORRECT
DATA(lo_instance) = zcl_fi_process_instance=>create(
  iv_process_type = lv_type
  is_context      = ls_context
).
```

---

### Rule 4 — Error Handling: Always Raise `ZCX_FI_PROCESS_ERROR` (Constitution Principle V)

- **Never** use `MESSAGE TYPE 'E'` for flow control in step logic
- **Always** raise `ZCX_FI_PROCESS_ERROR` with `textid`, context values
- Framework catches exceptions automatically and logs them — no manual exception logging in steps
- Crash loud principle: prefer visible failures over silent swallowing

```abap
" ❌ WRONG
MESSAGE e001(zfi_alloc) TYPE 'E'.

" ✅ CORRECT
RAISE EXCEPTION TYPE zcx_fi_process_error
  EXPORTING
    textid = zcx_fi_process_error=>validation_failed
    value  = lv_company_code.
```

**Exception chaining** — always preserve original exception:

```abap
CATCH zcx_external_error INTO DATA(lx_ext).
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid   = zcx_fi_process_error=>technical_error
      value    = lv_context
      previous = lx_ext.    " ← preserve chain
```

---

### Rule 5 — Consult SAP Docs Before New Patterns (Constitution Principle III)

- **Mandatory** before implementing any SAP standard API (BALI, RAP, CDS, bgRFC, T100, APJ, etc.)
- **Mandatory** before any DDIC change
- Verify ABAP syntax compatibility for **ABAP 7.58**
- Use `SAP_Docs_MCP_search` / `SAP_Docs_MCP_fetch` tools before coding unknown APIs

> **Lesson learned (Story 4.4):** Guessing BALI API method names (`get_message_detail()` instead of documented `GET_ALL_VALUES`) caused multiple failed iterations. Always consult docs first.

---

### Rule 6 — SAP Naming Conventions (Constitution Principle II)

| Object Type | Pattern | Example |
|---|---|---|
| Classes | `ZCL_FI_PROCESS_*` | `ZCL_FI_PROCESS_MANAGER` |
| Interfaces | `ZIF_FI_PROCESS_*` | `ZIF_FI_PROCESS_STEP` |
| Exception classes | `ZCX_FI_PROCESS_*` | `ZCX_FI_PROCESS_ERROR` |
| Programs/tables | `ZFI_PROCESS_*` | `ZFI_PROC_INST` |
| Table types | `ZFI_PROCESS_TT_*` | `ZFI_PROCESS_TT_PROC_STEP` |
| Allocation classes | `ZCL_FI_ALLOC_STEP_*` | `ZCL_FI_ALLOC_STEP_PHASE2` |
| APJ job classes | `ZCL_FI_*_JOB_*` | `ZCL_FI_ALLOC_JOB_ALLOC` |
| Behavior pool | `zbp_<entity_name>` | `zbp_fi_alloc_dashboard_ce` |

---

### Rule 7 — ABAP-Doc on ALL Public Methods (Constitution Principle II)

Every public method must have ABAP-Doc with:
- `"!` one-line description
- `@parameter` for each parameter
- `@raising` for each exception

```abap
"! Execute the allocation step
"! @parameter is_context | Process execution context
"! @parameter rs_result   | Step execution result
"! @raising   zcx_fi_process_error | Step execution failed
METHODS execute
  IMPORTING is_context TYPE zif_fi_process_step=>ts_context
  RETURNING VALUE(rs_result) TYPE zif_fi_process_step=>ts_result
  RAISING   zcx_fi_process_error.
```

---

### Rule 8 — Database Tables Must Include Audit Fields

All database tables must include:
- `MANDT` (client key, first field)
- `CREATED_BY`, `CREATED_AT`
- `CHANGED_BY`, `CHANGED_AT`
- Status fields use domains with fixed values (not free text) — **exception**: `BUSINESS_STATUS_1/2` are intentionally free-text CHAR(60) business vocabulary fields

---

### Rule 9 — Constants Not Magic Values

- Status values defined as constants in relevant classes (`gc_status` structure pattern)
- No magic strings/numbers inline
- Process configuration in customizing tables (`ZFI_PROC_TYPE`, `ZFI_PROC_DEF`)

```abap
" ✅ CORRECT
CONSTANTS: BEGIN OF gc_status,
             new         TYPE zfi_process_status VALUE 'N',
             running     TYPE zfi_process_status VALUE 'R',
             superseded  TYPE zfi_process_status VALUE 'S',
             execreq     TYPE zfi_process_status VALUE 'E',
             restreq     TYPE zfi_process_status VALUE 'T',
             breakpoint  TYPE zfi_process_status VALUE 'B',
           END OF gc_status.

" ❌ WRONG
IF ms_instance-status = 'S'. " magic string
```

---

## Process Step Interface Contract

All allocation step classes implement `ZIF_FI_PROCESS_STEP` with these required methods:

| Method | Serial Steps (INIT, PHASE1, PHASE3, CORR_BCHE) | PHASE2 (queue mode) |
|---|---|---|
| `init` | Populate instance vars from context | Same + initialize `mo_log` HERE |
| `validate` | Check all mandatory params are not initial | Same |
| `execute` | Main business logic | Queue substeps for bgRFC |
| `execute_substep` | Raise `ZCX_FI_PROCESS_ERROR` ("not supported") | Real substep logic |
| `on_success` | Empty (return silently) | Write final state 'F' to `ZFI_ALLOC_STATE` |
| `on_error` | Empty (return silently) | Write error state 'E' to `ZFI_ALLOC_STATE` |
| `rollback` | Undo DML: DELETE/UPDATE in state tables | Same |

**Critical rule for `init`:** `mo_log` MUST be initialized in `init` (not `execute`) so it is available in bgRFC substep execution paths where `init` is called but `execute` is not.

---

## Process Instance Status Values — Full Reference

| Status | Value | Terminal? | Notes |
|---|---|---|---|
| NEW | 'N' | No | |
| RUNNING | 'R' | No | |
| COMPLETED | 'C' | Yes | Duplicate check blocks new instances when DUPLIC_CHECK is on |
| FAILED | 'F' | No | Can restart |
| CANCELLED | 'X' | Yes | |
| SUPERSEDED | 'S' | Soft-terminal | Replaced by newer instance; `supersede_process()` on manager; `cancel()` allowlist includes it |
| EXECREQ | 'E' | No | APJ execute requested; APJ job calls `execute_process()` |
| RESTREQ | 'T' | No | APJ restart requested; APJ job calls `restart_process()` |
| BREAKPOINT | 'B' | No | Halted at step breakpoint; use `restart_process()` to continue |

**PENDING/QUEUED/SKIPPED are for STEPS only — never assign to process instances.**

`cleanup_old_instances()` targets **CANCELLED + SUPERSEDED** — not FAILED.

---

## APJ (Application Jobs) Integration

- **Mode 2** — Generic job class `ZCL_FI_PROCESS_JOB`; triggered via `request_execute_process()` / `request_restart_process()` on manager; sets EXECREQ/RESTREQ status
- **Mode 3** — Process-type-specific job class (e.g., `ZCL_FI_ALLOC_JOB_ALLOC`); extends `ZCL_FI_PROCESS_JOB_BASE`; registered in APJ catalog (SAJC) + template (SAJT)
- APJ job calls `check_parallel_limit()` **before** `execute_process()` — enforced in both sync and async paths
- `assign_to_appl_job()` links the process instance to the APJ job for traceability
- Safety-net BAL log in APJ job base — captures failures even before framework logger is initialized

```abap
" ✅ CORRECT — non-blocking APJ schedule from Fiori action
zcl_fi_process_manager=>get_instance( )->request_execute_process(
  iv_process_type   = 'ALLOCATIONS'
  iv_parameter_data = lv_params
).

" ❌ WRONG — direct execute blocks HTTP thread
zcl_fi_process_manager=>get_instance( )->execute_process(
  iv_process_type   = 'ALLOCATIONS'
  iv_parameter_data = lv_params
).
```

---

## Breakpoint API

- `set_step_breakpoint( iv_step_number )` on instance — sets `BREAKPOINT` field on `ZFI_PROC_DEF` step row
- `set_step_breakpoint_process( iv_instance_id, iv_step_number )` on manager — convenience wrapper
- When execution reaches the breakpoint step: instance status → `BREAKPOINT`, step status → `BREAKPOINT`; process halts cleanly
- To continue: call `restart_process()` — finds first non-completed step automatically
- `SKIP_BREAKPOINTS` field on `ZFI_PROC_INST` — if set, all breakpoints are ignored for that instance run

---

## RAP / Fiori Dashboard Patterns

Lessons learned from EST-132, EST-134, EST-136 (critical — do not repeat these mistakes):

```abap
" ✅ CORRECT — read custom entity in RAP handler
SELECT * FROM zfi_i_alloc_dashboard_ce
  INTO TABLE @DATA(lt_results).

" ❌ WRONG — READ ENTITIES does not work on custom entities
READ ENTITIES OF zfi_i_alloc_dashboard_ce
  ENTITY alloc_dashboard_ce ALL FIELDS WITH ...
" ↑ runtime error

" ✅ CORRECT — COMMIT WORK is prohibited in RAP saver
zcl_fi_process_manager=>get_instance( )->execute_process(
  iv_parameter_data = lv_params
  iv_no_commit      = abap_true   " ← pass this in RAP context
).

" ❌ WRONG — explicit commit inside RAP saver/handler
COMMIT WORK AND WAIT.
```

**BDEF requirements for unmanaged RAP:**
- `authorization master( global )` — required; activation fails without it
- `features: instance` — for per-row enable/disable of actions
- `side effects` block — required to refresh list after action execution
- Feature control: always return explicit `fc-op-enabled` or `fc-op-disabled`; returning initial disables the action silently

**RAP saver `save()` method:**
```abap
" ✅ ALWAYS wrap saver body in CATCH cx_root
METHOD save.
  TRY.
    " ... save logic ...
  CATCH cx_root INTO DATA(lx).
    " log and surface — unhandled exceptions silently abort OData response
  ENDTRY.
ENDMETHOD.
```

---

## Business Status & Parameter Projection

- `set_business_status( iv_bs1, iv_bs2 )` on instance — buffers override; applied on next FINAL commit
- BS override is **ignored on failure path** — `FAIL_BS1/2` from `ZFI_PROC_DEF` always wins
- `BUSINESS_STATUS_1/2` are free-text CHAR(60) — intentional exception to constitution §DB domain rule
- `PARAM_VAL_1..5` / `PARAM_LABEL_1..5` populated at instance creation only; not updated on restart
- `populate_param_columns()` uses RTTS (`CL_ABAP_STRUCTDESCR`, `CL_ABAP_ELEMDESCR->GET_DDIC_FIELD`) — consult SAP docs before modifying

---

## Configurable Fields on ZFI_PROC_TYPE

| Field | Purpose | Notes |
|---|---|---|
| `BAL_OBJECT` / `BAL_SUBOBJECT` | SLG1 log categorization per process type | EST-101 |
| `BGRFC_DEST_NAME_INBOUND` | bgRFC inbound destination for PHASE2 substeps | Replaces hardcoded heuristic (EST-100) |
| `MAX_PARALLEL_INSTS` | Max concurrent running instances; 0 = unlimited | Checked in both sync execute AND APJ job (EST-106, EST-143) |
| `DUPLIC_CHECK` | Enable parameter hash duplicity check | Blocks RUNNING + COMPLETED duplicates (EST-102, EST-110) |
| `PARAMETER_STRUCTURE` | DDIC INTTAB structure name for param projection | Used by `populate_param_columns()` |

---

## Logging Patterns (BAL via ZIF_FI_PROCESS_LOGGER)

### Standard logging block

```abap
MESSAGE <type><number>(<class>) [WITH <vars>] INTO DATA(lv_dummy).
IF mo_log IS BOUND.
  mo_log->message(
    iv_message_class  = '<class>'
    iv_message_number = '<number>'
    iv_message_v1     = <var1>
    iv_severity       = '<type>'
  ).
ENDIF.
```

**Important:** `lv_dummy` MUST be `TYPE string` — `TYPE c` silently truncates to 1 character.

**Fail-hard logger:** All logging methods now declare `RAISING zcx_fi_process_error` — do NOT add `CATCH` around `mo_log->message()` calls unless you explicitly want to suppress the error. The old defensive `CATCH zcx_fi_process_error` pattern is obsolete.

### Severity guide

| Code | When to use |
|---|---|
| `I` | Progress updates, intermediate results |
| `S` | Successful operations, phase completions |
| `W` | Non-critical issues, zero-result data sets |
| `E` | Business validation errors |

### Message number ranges (T100)

**ZFI_PROCESS:**
- `001-019` Instance lifecycle
- `020-039` Step execution
- `040-059` bgRFC/queued execution
- `060-099` Framework errors
- `996-999` Infrastructure errors

**ZFI_ALLOC:**
- `001-099` INIT step
- `100-199` PHASE1
- `200-299` PHASE2
- `300-399` PHASE3
- `400-499` CORR_BCHE
- `500-599` General allocation errors
- `600-699` Export step

### DO NOT log every loop iteration

```abap
" ❌ BAD — floods log with 10,000+ messages
LOOP AT lt_items INTO DATA(ls_item).
  mo_log->message( ... ).
ENDLOOP.

" ✅ GOOD — log summary only
mo_log->message(
  iv_message_number = '212'
  iv_message_v1     = lines( lt_items )
  iv_severity       = 'I'
).
```

---

## bgRFC / Substep Execution

- **PHASE2** uses `substep_mode = 'QUEUE'` — substeps execute in parallel via bgRFC
- **All other steps** use `substep_mode = 'SERIAL'` — `execute_substep` must raise exception
- Substeps attach to **parent instance's BAL log** automatically (identified by parent UUID)
- **NEVER issue `COMMIT WORK` in step code** — the framework / qRFC infrastructure manages LUW
- bgRFC function module calls `lo_step->init( ls_context )` BEFORE `execute_substep`
- `cx_amdp_execution_failed` is `cx_static_check` — must add `RAISING cx_amdp_execution_failed` to method declarations and wrap calls in `TRY/CATCH`

---

## Key Framework Classes

| Class | Purpose | Pattern |
|---|---|---|
| `ZCL_FI_PROCESS_MANAGER` | Singleton process orchestrator | Factory: `GET_INSTANCE()` |
| `ZCL_FI_PROCESS_INSTANCE` | Process instance lifecycle | Factory: `create()`, `load()` |
| `ZCL_FI_PROCESS_DEFINITION` | Process type config (step sequence) | Factory |
| `ZCL_FI_PROCESS_STEP_BASE` | Base class with inherited `mo_log` | Abstract |
| `ZCL_FI_PROCESS_LOGGER` | BAL wrapper implementing `ZIF_FI_PROCESS_LOGGER` | Factory: `create_new()`, `attach_existing()` |
| `ZCL_FI_PROCESS_NULL_LOGGER` | No-op logger stub (for legacy programs) | Direct `NEW` acceptable |
| `ZCL_FI_PROCESS_JOB_BASE` | APJ Mode 3 base class | Extend per process type |
| `ZCL_FI_PROCESS_JOB` | APJ Mode 2 generic job | Registered in APJ catalog |

---

## Allocation Step Classes (cz.imcg.fast.ovysledovka)

| Class | Step # | Mode | Purpose |
|---|---|---|---|
| `ZCL_FI_ALLOC_STEP_INIT` | 0001 | SERIAL | Initialize state row in `ZFI_ALLOC_STATE` |
| `ZCL_FI_ALLOC_STEP_PHASE1` | 0002 | SERIAL | Phase 1 allocation logic |
| `ZCL_FI_ALLOC_STEP_PHASE2` | 0003 | QUEUE | Phase 2 (parallel substeps via bgRFC) |
| `ZCL_FI_ALLOC_STEP_CORR_BCHE` | 0004 | SERIAL | Correction batch processing |
| `ZCL_FI_ALLOC_STEP_PHASE3` | 0005 | SERIAL | Phase 3 allocation item generation |

**APJ Job Classes:**
| Class | Process Type |
|---|---|
| `ZCL_FI_ALLOC_JOB_ALLOC` | ALLOCATIONS |
| `ZCL_FI_ALLOC_JOB_EXPORT` | ALLOC_EXPORT |

**Dashboard Components (ovysledovka):**
| Component | Type | Purpose |
|---|---|---|
| `ZFI_I_ALLOC_DASHBOARD_CE` | Custom entity | List report data source |
| `ZFI_I_ALLOC_DASH_STEP_CE` | Custom entity | Object page step details |
| `ZCL_FI_ALLOC_DASH_QUERY` | Query provider | List report data |
| `ZCL_FI_ALLOC_DASH_STEP_QRY` | Query provider | Step detail data |
| `zbp_fi_alloc_dashboard_ce` | Behavior pool | 5 actions + feature control + saver |

---

## Code Review Checklist (Before Marking Any Story Done)

- [ ] DDIC types used for cross-boundary signatures; local types only for internal helpers
- [ ] ABAP-Doc on all public methods (@parameter, @raising)
- [ ] Factory pattern used (no `NEW zcl_fi_process_*` directly)
- [ ] `ZCX_FI_PROCESS_ERROR` raised with proper message IDs and context
- [ ] Audit fields populated (created_by, timestamps) for DB records
- [ ] Naming follows `ZCL_FI_*`, `ZIF_FI_*`, `ZCX_FI_*` convention
- [ ] **Line length ≤ 120 chars** (absolute limit 255 — causes parser errors)
- [ ] Long statements wrapped at logical boundaries
- [ ] `cx_amdp_execution_failed` declared in `RAISING` clause for AMDP-calling methods
- [ ] RAP saver has `CATCH cx_root` guard
- [ ] No `COMMIT WORK` inside RAP handlers/savers (use `iv_no_commit = abap_true`)
- [ ] Test program updated or regression tested
- [ ] `mcp-sap-docs` consulted for any new SAP API patterns

---

## Common Pitfalls to Avoid

| Pitfall | Correct Approach |
|---|---|
| `lv_dummy TYPE c` for `MESSAGE...INTO` | Use `lv_dummy TYPE string` — `TYPE c` truncates to 1 char |
| `MESSAGE e001(class) TYPE 'E'` for error flow | `RAISE EXCEPTION TYPE zcx_fi_process_error` |
| `mo_log = NEW ...` in `execute` | Initialize `mo_log` in `init` for bgRFC path compatibility |
| `COMMIT WORK` in step body | Never commit in steps — framework manages LUW |
| `COMMIT WORK` in RAP saver/handler | Pass `iv_no_commit = abap_true` to manager methods |
| Inline `TYPE` definitions for cross-boundary types | Use DDIC `ZFI_PROCESS_TT_*` types |
| Guessing SAP API method names | Consult `SAP_Docs_MCP_search` before implementing |
| Logging every item in a loop | Log summaries; loop logs flood BAL and slow SLG1 |
| Missing `mo_log IS BOUND` guard (old pattern) | Logger is now fail-hard — check `IS BOUND` but don't swallow exceptions |
| `READ ENTITIES` on custom entity | Use `SELECT FROM <custom_entity>` in RAP handlers |
| Omitting `authorization master( global )` in BDEF | Required — activation fails without it |
| `execute_process()` from Fiori action directly | Use `request_execute_process()` — APJ handles background |
| `check_parallel_limit()` only in sync path | Also call it before APJ `execute_process()` |
| `cx_amdp_execution_failed` not in RAISING clause | It is `cx_static_check` — escapes unhandled from bgRFC substeps |
| `line_exists()` on method return value | Assign return to local var first, then check |
| Missing `CATCH cx_root` in RAP `save()` | Unhandled exceptions silently abort OData response |
| Treating `SUPERSEDED` as fully terminal | Soft-terminal: `cancel()` allowlist includes it; `cleanup_old_instances()` targets it |
| `execute_process()` to resume after BREAKPOINT | Use `restart_process()` — finds first non-completed step |
| `mv_bs_override` not reset on failure | `write_fail_bs()` resets it; do not short-circuit failure path |
| `NEW zcl_fi_process_null_logger( )` | Acceptable — null logger has public constructor by design |

---

## T100 Message Creation Guidelines

- Max 73 characters per message text
- Use `&1`–`&4` placeholders (max 4)
- Create bilingual: Czech (CS) + English (EN) from the start
- Translate via SE63 (not hardcode English; T100 supports multi-language)
- Message text style: start with action verb ("Processing...", "Loaded...", "Failed...")
- Never include sensitive data (PII, credentials) in message text

---

## Constitution Reference

Constitution v1.0.0 at `_bmad/_memory/constitution.md` is the **source of truth**.

| Principle | Summary |
|---|---|
| I — DDIC-First | Prefer DDIC types; local types acceptable for internal helpers |
| II — SAP Standards | Naming conventions, ABAP-Doc, transportability |
| III — Consult SAP Docs | mcp-sap-docs BEFORE any new SAP API |
| IV — Factory Pattern | Private constructors, factory methods for framework classes |
| V — Error Handling | `ZCX_FI_PROCESS_ERROR` with context; audit trail |

Amendment procedure: document justification → bump version → sync to code repos via `./repos/sync-constitution.sh`.

---

## Testing & Verification Rules

- No ABAP Unit test classes for step logic — testing is manual via test programs
- Framework classes (`ZCL_FI_PROCESS_*`) have ABAP Unit tests (`ltcl_health_tests`, `ltcl_apj_e2e_tests`, `ltcl_breakpoint_test`, etc.)
- Health check class: `ZCL_FIPROC_HEALTH_CHK_QUERY` — run all 36+ health checks before marking stories done
- APJ E2E test has `DURATION MEDIUM` and polls up to 90s — do not reduce timeout
- Before marking any story done: run the relevant test program end-to-end
- **Regression scope**: changes to framework classes require testing ALL 5 allocation steps
- bgRFC substep testing requires SM58 / SLG1 inspection after queue execution
- Test with real company code data — never hardcode test company codes in production code

---

## Code Quality & Style Rules

- **abaplint** enforces 120-char line limit — CI will reject longer lines
- Absolute ABAP parser limit is 255 chars — exceeding causes "Field X is unknown" parse errors
- No commented-out code in committed files — use git history instead
- No TODO/FIXME comments left in transported code
- Method body max ~50 lines — extract helper methods if longer
- Use inline `DATA(...)` for simple loop variables and result captures; use DDIC types for complex structures
- **Constants structure pattern:**

```abap
" ✅ CORRECT
CONSTANTS: BEGIN OF gc_status,
             new       TYPE zfi_process_status VALUE 'N',
             running   TYPE zfi_process_status VALUE 'R',
           END OF gc_status.

" ❌ WRONG
CONSTANTS gc_status_new     TYPE zfi_process_status VALUE 'N'.
CONSTANTS gc_status_running TYPE zfi_process_status VALUE 'R'.
```

- abapGit XML format — never manually edit `.abap` XML wrappers
- All objects must be in correct package and assigned to a transport

---

## Development Workflow Rules

### Multi-Repository Coordination

- Code changes go to ONE of two repos — never this planning repo:
  - `cz.imcg.fast.planner` — framework changes (`ZFI_PROCESS` package)
  - `cz.imcg.fast.ovysledovka` — allocation step + dashboard changes (`ZFI_ALLOC_PROCESS` package)
- After completing a story, update `sprint-status.yaml` with commit hash and target repository

### abapGit Workflow

- Pull from abapGit BEFORE making changes — avoid merge conflicts in XML
- Commit message format: `Story <X.Y>: <short description>` or `EST-NNN: <short description>`
- DDIC objects must be activated before dependent ABAP classes — activate in dependency order

### Transport & Activation Order

1. DDIC objects (domains → data elements → structures → table types → tables)
2. Exception classes (`ZCX_*`)
3. Interfaces (`ZIF_*`)
4. Base/abstract classes
5. Concrete implementation classes
6. CDS views (R view → C view → DDLX metadata extension)
7. BDEF
8. Programs / function modules last

### Constitution Sync

- After any constitution amendment: run `./repos/sync-constitution.sh`
- Verify both code repos receive the updated `.constitution.md`
- Commit sync result in both code repos before marking amendment complete

---

## Edge Cases Agents Must Handle

- `mo_log IS BOUND` check before log calls — logger may be uninitialized in direct-call scenarios
- `on_success` / `on_error` must be **idempotent** — framework may call them more than once on retry
- Process instance status `FAILED` is NOT terminal — it can be restarted; only `COMPLETED` and `CANCELLED` are fully terminal; `SUPERSEDED` is soft-terminal
- `PENDING/QUEUED/SKIPPED` statuses are for **steps only** — never assign to process instances
- APJ guard statuses `EXECREQ`/`RESTREQ` must be accepted as valid starting statuses in `execute()` / `restart()` guards
- Feature control in RAP: returning initial value from `get_instance_features()` silently disables the action — always return explicit `fc-op-enabled` or `fc-op-disabled`
- `mv_bs_override` flag must be reset on failure path — handled by `write_fail_bs()`; do not bypass

---

## Usage Guidelines

**For AI Agents:**

- Read this file before implementing any code in this project
- Follow ALL rules exactly as documented — they come from real bugs
- When in doubt, prefer the more restrictive option
- Consult `SAP_Docs_MCP_search` before implementing any unfamiliar SAP API
- Update this file if new patterns or anti-patterns emerge

**For Humans:**

- Keep this file lean and focused on agent needs
- Update when technology stack or patterns change
- Review after each sprint for new lessons learned
- Remove rules that become obvious over time

_Last Updated: 2026-04-01_
