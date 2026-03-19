---
project_name: 'cz.imcg.fast.allocations'
user_name: 'Zdenek'
date: '2026-03-13'
sections_completed: ['technology_stack', 'language_rules', 'framework_rules', 'testing_rules', 'quality_rules', 'workflow_rules', 'anti_patterns']
status: 'complete'
rule_count: 42
optimized_for_llm: true
---

# Project Context for AI Agents

_This file contains critical rules and patterns that AI agents must follow when implementing code in this project. Focus on unobvious details that agents might otherwise miss._

---

## Technology Stack & Versions

| Technology | Version / Details |
|---|---|
| ABAP | 7.58 (SAP S/4HANA compatible, on-premise) |
| SAP DDIC | DDIC-first — all types defined as DDIC objects |
| ZFI_PROCESS framework | Custom orchestration framework (cz.imcg.fast.planner) |
| Background processing | bgRFC (background RFC, queued mode for PHASE2 substeps) |
| abapGit | XML format for source control |
| ABAP linting | abaplint.json enforces 120-char line length |
| Logging | SAP Application Log (BAL) via ZIF_FI_PROCESS_LOGGER / ZCL_FI_PROCESS_LOGGER |
| Message system | T100 message classes: ZFI_PROCESS (framework), ZFI_ALLOC (allocation domain) |
| Database | SAP HANA (client-dependent tables, MANDT key field required) |

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
| `cz.imcg.fast.ovysledovka` | Allocation step implementations | `ZFI_ALLOC_PROCESS` |

Do **not** generate ABAP code in this repository. See `AGENTS.md §7`.

---

## Critical Implementation Rules

### Rule 1 — DDIC-First (Constitution Principle I) ⚠️ NON-NEGOTIABLE

- **NO** local `TYPE` definitions for structures or table types in class definitions or programs
- **ALL** method signatures use DDIC table types (e.g., `ZFI_PROCESS_TT_PROC_STEP`)
- **ALL** fields based on data elements → domains
- Naming: `ZFI_PROCESS` prefix for domains/data elements/tables, `ZFI_PROCESS_TT_` for table types

```abap
" ❌ WRONG
TYPES: BEGIN OF ty_step,
         step_number TYPE i,
       END OF ty_step.
TYPES tt_steps TYPE STANDARD TABLE OF ty_step.

" ✅ CORRECT — use DDIC table type
DATA lt_steps TYPE zfi_process_tt_proc_step.
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

- **Mandatory** before implementing any SAP standard API (BALI, RAP, CDS, bgRFC, T100, etc.)
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
- Status fields use domains with fixed values (not free text)

---

### Rule 9 — Constants Not Magic Values

- Status values defined as constants in relevant classes (e.g., `gc_status` structure)
- No magic strings/numbers inline
- Process configuration in customizing tables (`ZFI_PROC_TYPE`, `ZFI_PROC_DEF`)

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

## Logging Patterns (BAL via ZIF_FI_PROCESS_LOGGER)

### Standard logging block (used 97 times in codebase)

```abap
MESSAGE <type><number>(<class>) [WITH <vars>] INTO DATA(lv_dummy).
IF mo_log IS BOUND.
  TRY.
      mo_log->message(
        iv_message_class  = '<class>'   " 'ZFI_PROCESS' or 'ZFI_ALLOC'
        iv_message_number = '<number>'  " 3-digit, e.g. '212'
        iv_message_v1     = <var1>      " optional placeholders
        iv_severity       = '<type>'    " I/S/W/E
      ).
    CATCH zcx_fi_process_error.
      " Logger failed - business logic continues unaffected
  ENDTRY.
ENDIF.
```

**Why the dual pattern?** `MESSAGE...INTO` populates `sy-msg*` fields (for result structures, legacy compatibility), then `mo_log->message()` persists to BAL for audit trail.

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
- `996-999` Infrastructure errors (logger failures, missing translations)

**ZFI_ALLOC:**
- `001-099` INIT step
- `100-199` PHASE1
- `200-299` PHASE2 (most verbose)
- `300-399` PHASE3
- `400-499` CORR_BCHE
- `500-599` General allocation errors

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

---

## Key Framework Classes

| Class | Purpose | Pattern |
|---|---|---|
| `ZCL_FI_PROCESS_MANAGER` | Singleton process orchestrator | Factory: `GET_INSTANCE()` |
| `ZCL_FI_PROCESS_INSTANCE` | Process instance lifecycle (NEW→RUNNING→COMPLETED/FAILED) | Factory: `create()`, `load()` |
| `ZCL_FI_PROCESS_DEFINITION` | Process type config (step sequence) | Factory |
| `ZCL_FI_PROCESS_STEP` (base) | Base class with inherited `mo_log` | Abstract |
| `ZCL_FI_PROCESS_LOGGER` | BAL wrapper implementing `ZIF_FI_PROCESS_LOGGER` | Factory: `create_new()`, `attach_existing()` |

**Process Instance Status Values (for `ZFI_PROCESS_STATUS` domain):**

| Status | Value | Terminal? |
|---|---|---|
| NEW | 'N' | No |
| RUNNING | 'R' | No |
| COMPLETED | 'C' | Yes |
| FAILED | 'F' | No (can restart) |
| CANCELLED | 'X' | Yes |

**PENDING ('P'), QUEUED ('Q'), SKIPPED ('S') are for STEPS only — never for process instances.**

---

## Allocation Step Classes (cz.imcg.fast.ovysledovka)

| Class | Step # | Mode | Purpose |
|---|---|---|---|
| `ZCL_FI_ALLOC_STEP_INIT` | 0001 | SERIAL | Initialize state row in `ZFI_ALLOC_STATE` |
| `ZCL_FI_ALLOC_STEP_PHASE1` | 0002 | SERIAL | Phase 1 allocation logic |
| `ZCL_FI_ALLOC_STEP_PHASE2` | 0003 | QUEUE | Phase 2 (parallel substeps via bgRFC) |
| `ZCL_FI_ALLOC_STEP_PHASE3` | 0004 | SERIAL | Phase 3 allocation item generation |
| `ZCL_FI_ALLOC_STEP_CORR_BCHE` | 0005 | SERIAL | Correction batch processing |

---

## Code Review Checklist (Before Marking Any Story Done)

- [ ] DDIC objects defined — no local TYPE definitions
- [ ] ABAP-Doc on all public methods (@parameter, @raising)
- [ ] Factory pattern used (no `NEW zcl_fi_process_*` directly)
- [ ] `ZCX_FI_PROCESS_ERROR` raised with proper message IDs and context
- [ ] Audit fields populated (created_by, timestamps) for DB records
- [ ] Naming follows `ZCL_FI_*`, `ZIF_FI_*`, `ZCX_FI_*` convention
- [ ] **Line length ≤ 120 chars** (absolute limit 255 — causes parser errors)
- [ ] Long statements wrapped at logical boundaries
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
| Inline `TYPE` definitions for table types | Use DDIC `ZFI_PROCESS_TT_*` types |
| Guessing SAP API method names | Consult `SAP_Docs_MCP_search` before implementing |
| Logging every item in a loop | Log summaries; loop logs flood BAL and slow SLG1 |
| Missing `ASSERT mo_log IS BOUND` guard | Add assert at top of `execute`/`execute_substep` |

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
| I — DDIC-First | No local types; all types in DDIC |
| II — SAP Standards | Naming conventions, ABAP-Doc, transportability |
| III — Consult SAP Docs | mcp-sap-docs BEFORE any new SAP API |
| IV — Factory Pattern | Private constructors, factory methods for framework classes |
| V — Error Handling | `ZCX_FI_PROCESS_ERROR` with context; audit trail |

Amendment procedure: document justification → bump version → sync to code repos via `./repos/sync-constitution.sh`.

---

## Testing & Verification Rules

- No ABAP Unit test classes exist — testing is manual via test programs
- Each step class has a corresponding test program (e.g., `ZFI_ALLOC_TEST_PHASE2`)
- Before marking any story done: run the relevant test program end-to-end
- **Regression scope**: changes to framework classes (`ZCL_FI_PROCESS_*`) require testing ALL 5 allocation steps
- bgRFC substep testing requires SM58 / SLG1 inspection after queue execution
- Test with real company code data — never hardcode test company codes in production code
- `ASSERT` statements acceptable in development; remove before transport to production

---

## Code Quality & Style Rules

- **abaplint** enforces 120-char line limit — CI will reject longer lines
- Absolute ABAP parser limit is 255 chars — exceeding it causes cryptic "Field X is unknown" parse errors
- No commented-out code in committed files — use git history instead
- No TODO/FIXME comments left in transported code
- Method body max ~50 lines — extract helper methods if longer
- Use inline `DATA(...)` only for simple loop variables and result captures; use DDIC types for complex structures
- **Constants structure pattern** — use nested `gc_status` structure, not individual constants:

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
- All objects must be in correct package (`ZFI_PROCESS` or `ZFI_ALLOC_PROCESS`) and assigned to a transport

---

## Development Workflow Rules

### Multi-Repository Coordination

- Code changes go to ONE of two repos — never this planning repo:
  - `cz.imcg.fast.planner` — framework changes (`ZFI_PROCESS` package)
  - `cz.imcg.fast.ovysledovka` — allocation step changes (`ZFI_ALLOC_PROCESS` package)
- After completing a story, update `sprint-status.yaml` with commit hash and target repository

### abapGit Workflow

- Pull from abapGit BEFORE making changes — avoid merge conflicts in XML
- Commit message format: `Story <X.Y>: <short description>`
- DDIC objects must be activated before dependent ABAP classes — activate in dependency order

### Transport & Activation Order

1. DDIC objects (domains → data elements → structures → table types → tables)
2. Exception classes (`ZCX_*`)
3. Interfaces (`ZIF_*`)
4. Base/abstract classes
5. Concrete implementation classes
6. Programs / function modules last

### Constitution Sync

- After any constitution amendment: run `./repos/sync-constitution.sh`
- Verify both code repos receive the updated `.constitution.md`
- Commit sync result in both code repos before marking amendment complete

---

## Critical Don't-Miss Rules

### Proven Bug Sources (from project history)

| Anti-Pattern | Why It Fails | Correct Approach |
|---|---|---|
| `lv_dummy TYPE c` in `MESSAGE...INTO` | `TYPE c` = 1 char, silently truncates message text | Always `lv_dummy TYPE string` |
| `mo_log = NEW ...` in `execute` | bgRFC calls `init→execute_substep`, skips `execute` entirely | Initialize `mo_log` in `init` |
| `COMMIT WORK` in step body | Breaks bgRFC LUW — qRFC manages commit boundary | Never commit in steps |
| `MESSAGE e001(class) TYPE 'E'` | Not caught by framework exception handler | `RAISE EXCEPTION TYPE zcx_fi_process_error` |
| Guessing SAP API method names | Wrong names compile but fail at runtime (Story 4.4: `get_message_detail` vs `GET_ALL_VALUES`) | Always consult `SAP_Docs_MCP_search` first |
| Logging inside `LOOP AT` | Floods BAL with 10,000+ messages; SLG1 becomes unusable | Log summary counts only |
| `NEW zcl_fi_process_instance()` | Private constructor — runtime exception | Use `zcl_fi_process_instance=>create()` |
| Local `TYPES` for table types | Constitution violation; breaks DDIC consistency | Use `ZFI_PROCESS_TT_*` DDIC types |

### Security / Data Rules

- Never include company codes, user IDs, or financial figures in T100 message text
- `MANDT` must be the first key field on all custom DB tables — never omit
- No hardcoded client numbers or system-specific values in code

### Edge Cases Agents Must Handle

- `mo_log IS BOUND` check before every log call — logger may be uninitialized in unit test or direct-call scenarios
- `on_success` / `on_error` must be **idempotent** — framework may call them more than once on retry
- Process instance status `FAILED` is NOT terminal — it can be restarted; only `COMPLETED` and `CANCELLED` are terminal
- `PENDING/QUEUED/SKIPPED` statuses are for **steps only** — never assign to process instances

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

_Last Updated: 2026-03-13_
