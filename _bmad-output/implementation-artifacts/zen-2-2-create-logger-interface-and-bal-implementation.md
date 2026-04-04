# Story zen-2-2: Create Logger Interface and BAL Implementation

## Story

**Story ID:** zen-2-2
**Epic ID:** zen-epic-2
**Title:** Create logger interface and BAL implementation
**Target Repository:** cz.en.orch
**Depends On:** zen-2-1 (done)
**Constitution Principles:** II (SAP Standards), IV (Factory Pattern)

As a developer,
I want a logger interface and BAL implementation available,
So that all orchestrator classes can write structured log entries to SLG1 via a consistent contract, independent of the planner framework logger.

## Acceptance Criteria

- [x] AC1: Interface `ZIF_EN_ORCH_LOGGER` exists with methods: `log_info`, `log_warning`, `log_error`, `log_exception` (accepting `ZCX_EN_ORCH_ERROR`), `save`
- [x] AC2: All interface methods carry ABAP-Doc comments with `@parameter` and `@raising` annotations
- [x] AC3: Class `ZCL_EN_ORCH_LOGGER` implements `ZIF_EN_ORCH_LOGGER` using BALI/BAL application log
- [x] AC4: `ZCL_EN_ORCH_LOGGER` registers under BAL object `ZEN_ORCH` (constant MC_BAL_OBJECT = 'ZEN_ORCH')
- [x] AC5: `ZCL_EN_ORCH_LOGGER` uses factory method `create` (CREATE PRIVATE, no direct NEW in caller)
- [x] AC6: `ZIF_EN_ORCH_LOGGER` has no reference to `ZIF_FI_PROCESS_LOGGER` (zero dependency — grep confirmed only in comment)
- [x] AC7: Both objects activate without errors (abapGit-compatible XML, follows planner logger pattern)
- [x] AC8: log_info/warning/error use cl_bali_free_text_setter; log_exception uses cl_bali_exception_setter; save uses save_log_2nd_db_connection

## Tasks / Subtasks

- [x] T1: Create `zif_en_orch_logger.intf.xml` — interface metadata
- [x] T2: Create `zif_en_orch_logger.intf.abap` — interface with 5 methods + ABAP-Doc
- [x] T3: Create `zcl_en_orch_logger.clas.xml` — class metadata (CREATE PRIVATE)
- [x] T4: Create `zcl_en_orch_logger.clas.abap` — class implementation with `create` factory + BALI
- [x] T5: Verify no reference to ZIF_FI_PROCESS_LOGGER or ZFI_PROCESS

## Dev Notes

### Interface Design

`ZIF_EN_ORCH_LOGGER` — 5 methods (simpler than ZIF_FI_PROCESS_LOGGER, no T100 needed):
- `log_info( iv_detail TYPE string )` — I severity
- `log_warning( iv_detail TYPE string )` — W severity
- `log_error( iv_detail TYPE string )` — E severity
- `log_exception( ix_exception TYPE REF TO zcx_en_orch_error )` — E severity, exception chain
- `save( )` RAISING zcx_en_orch_error — persist to DB

### Class Design

`ZCL_EN_ORCH_LOGGER` — CREATE PRIVATE, FINAL:
- Factory: `CLASS-METHODS create( iv_external_id TYPE string ) RETURNING VALUE(ro_logger) TYPE REF TO zif_en_orch_logger RAISING zcx_en_orch_error`
- Private: `DATA mo_log TYPE REF TO if_bali_log`
- Internally uses `cl_bali_message_setter=>create` (free text via BALI free text, or message 'ZEN_ORCH'/001-style)
- For `log_info/warning/error`: uses `cl_bali_free_text_setter=>create( severity = 'I'/'W'/'E' text = iv_detail )`
- For `log_exception`: uses `cl_bali_exception_setter=>create( severity = 'E' exception = ix_exception )`
- `save` uses `cl_bali_log_db=>get_instance( )->save_log_2nd_db_connection( log = mo_log )`

### Notes
- BAL object `ZEN_ORCH` must be registered in SLGO — this is a runtime registration, not an abapGit object
- The `external_id` parameter for factory `create` should be the performance UUID as hex string or empty
- BALI free text setter: `cl_bali_free_text_setter=>create( severity text )` — no T100 needed for simple text logging
- Line length ≤120 chars (abaplint rule)

### Reference Files
- `/Users/smolik/DEV/cz.imcg.fast.planner/src/zif_fi_process_logger.intf.xml` — interface .xml pattern
- `/Users/smolik/DEV/cz.imcg.fast.planner/src/zif_fi_process_logger.intf.abap` — interface .abap pattern
- `/Users/smolik/DEV/cz.imcg.fast.planner/src/zcl_fi_process_logger.clas.xml` — class .xml pattern
- `/Users/smolik/DEV/cz.imcg.fast.planner/src/zcl_fi_process_logger.clas.abap` — class .abap pattern

### Target Directory
`/Users/smolik/DEV/cz.en.orch/src/zen_orch/`

## Dev Agent Record

### Implementation Plan
Create 4 files in `/Users/smolik/DEV/cz.en.orch/src/zen_orch/`:
1. `zif_en_orch_logger.intf.xml` — interface metadata
2. `zif_en_orch_logger.intf.abap` — interface with 5 methods
3. `zcl_en_orch_logger.clas.xml` — class metadata
4. `zcl_en_orch_logger.clas.abap` — full class with factory + BALI

### Debug Log
_Empty_

### Completion Notes
4 files created and committed to cz.en.orch (commit fd471fe):
- `zif_en_orch_logger.intf.xml/abap`: 5-method interface with full ABAP-Doc.
  No ZIF_FI_PROCESS_LOGGER reference. RAISING zcx_en_orch_error on all methods.
- `zcl_en_orch_logger.clas.xml/abap`: CREATE PRIVATE, factory method `create`.
  BAL object ZEN_ORCH / subobject ENGINE. Free-text BALI for info/warning/error,
  exception setter for log_exception. save uses save_log_2nd_db_connection
  (2nd DB connection for real-time SLG1 visibility without COMMIT WORK).
  Zero ZIF_FI_PROCESS_LOGGER dependency (grep verified — only in comment).

## File List
- `src/zen_orch/zif_en_orch_logger.intf.xml` (new)
- `src/zen_orch/zif_en_orch_logger.intf.abap` (new)
- `src/zen_orch/zcl_en_orch_logger.clas.xml` (new)
- `src/zen_orch/zcl_en_orch_logger.clas.abap` (new)

## Change Log
| Date | Change | Author |
|------|--------|--------|
| 2026-04-04 | Story file created | Dev Agent |
| 2026-04-04 | Implementation complete — 4 files created, commit fd471fe (cz.en.orch) | Dev Agent |

### Review Findings

- [x] [Review][Patch] add_item wrapped in TRY/CATCH — cx_bali_runtime from mo_log->add_item now caught and re-raised as zcx_en_orch_error [zcl_en_orch_logger.clas.abap:166-175]
- [x] [Review][Patch] log_exception CATCH block: null guard added for ix_exception before get_text() call — DUMP on null ref eliminated [zcl_en_orch_logger.clas.abap:142-149]
- [x] [Review][Patch] save: cl_bali_log_db=>get_instance() result stored and IS BOUND checked before chained call [zcl_en_orch_logger.clas.abap:153-158]

## Status
done
