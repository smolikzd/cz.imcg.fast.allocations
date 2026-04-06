# Story 1.3: Create data elements, structures, and table types

Status: review

## Story

As a developer,
I want all data elements, the start result structure, and table types defined in DDIC,
So that database tables and method signatures use fully typed, self-documenting fields.

## Acceptance Criteria

1. **Given** all domains from Story 1.2 exist and are active
2. **When** data elements, structures, and table types are activated
3. **Then** the following data elements exist and reference the correct domains:
   - `ZEN_ORCH_DE_STATUS` → domain `ZEN_ORCH_STATUS` (CHAR 1)
   - `ZEN_ORCH_DE_ELEM_TYPE` → domain `ZEN_ORCH_ELEM_TYPE` (CHAR 12)
   - `ZEN_ORCH_DE_SCORE_ID` → domain `ZEN_ORCH_SCORE_ID` (CHAR 30)
   - `ZEN_ORCH_DE_ADAPTER_TYPE` → domain `ZEN_ORCH_ADAPTER_TYPE` (CHAR 30)
   - `ZEN_ORCH_DE_SCORE_SEQ` → domain `ZEN_ORCH_SCORE_SEQ` (NUMC 4)
   - `ZEN_ORCH_DE_MAX_PAR` → domain `ZEN_ORCH_MAX_PARALLEL` (INT4)
   - `ZEN_ORCH_PERF_UUID` (RAW 16 — custom domain `ZEN_ORCH_PERF_UUID`)
   - `ZEN_ORCH_WU_HANDLE` (CHAR 255 — custom domain `ZEN_ORCH_WU_HANDLE`)
   - `ZEN_ORCH_DE_REF_ID` (CHAR 20 — custom domain `ZEN_ORCH_REF_ID`)
   - `ZEN_ORCH_DE_LOOP_ITER` (INT4 — custom domain `ZEN_ORCH_LOOP_ITER`)
   - `ZEN_ORCH_DE_PARAMS_JSON` (STRING — custom domain `ZEN_ORCH_PARAMS_JSON`)
   - `ZEN_ORCH_DE_IMPL_CLASS` (CHAR 30 — custom domain `ZEN_ORCH_IMPL_CLASS`)
4. **And** structure `ZEN_ORCH_S_START_RESULT` exists with fields:
   - `WORK_UNIT_HANDLE TYPE zen_orch_wu_handle`
   - `STATUS TYPE zen_orch_de_status`
5. **And** table type `ZEN_ORCH_TT_SCORE_STEP` exists as STANDARD TABLE OF `ZEN_ORCH_SCORE_STEP` with default key
6. **And** table type `ZEN_ORCH_TT_PERF_STEP` exists as STANDARD TABLE OF `ZEN_ORCH_PERF_STEP` with default key
7. **And** all objects activate without errors

## Tasks / Subtasks

- [x] Create data elements referencing ZEN_ORCH domains (AC: 3 — domain-backed DEs)
  - [x] ZEN_ORCH_DE_STATUS → ZEN_ORCH_STATUS
  - [x] ZEN_ORCH_DE_ELEM_TYPE → ZEN_ORCH_ELEM_TYPE
  - [x] ZEN_ORCH_DE_SCORE_ID → ZEN_ORCH_SCORE_ID
  - [x] ZEN_ORCH_DE_ADAPTER_TYPE → ZEN_ORCH_ADAPTER_TYPE
  - [x] ZEN_ORCH_DE_SCORE_SEQ → ZEN_ORCH_SCORE_SEQ
  - [x] ZEN_ORCH_DE_MAX_PAR → ZEN_ORCH_MAX_PARALLEL
- [x] Create data elements with built-in/no-domain types (AC: 3 — standalone DEs)
  - [x] ZEN_ORCH_PERF_UUID (RAW 16 — custom domain ZEN_ORCH_PERF_UUID)
  - [x] ZEN_ORCH_WU_HANDLE (CHAR 255 — custom domain ZEN_ORCH_WU_HANDLE)
  - [x] ZEN_ORCH_DE_REF_ID (CHAR 20 — custom domain ZEN_ORCH_REF_ID)
  - [x] ZEN_ORCH_DE_LOOP_ITER (INT4 — custom domain ZEN_ORCH_LOOP_ITER)
  - [x] ZEN_ORCH_DE_PARAMS_JSON (STRING — custom domain ZEN_ORCH_PARAMS_JSON)
  - [x] ZEN_ORCH_DE_IMPL_CLASS (CHAR 30 — custom domain ZEN_ORCH_IMPL_CLASS)
- [x] Create structure ZEN_ORCH_S_START_RESULT (AC: 4)
- [x] Create table type ZEN_ORCH_TT_SCORE_STEP (AC: 5)
- [x] Create table type ZEN_ORCH_TT_PERF_STEP (AC: 6)
- [x] Verify abapGit serialization format (AC: 7)

## Dev Notes

- **Target Repository**: `cz.en.orch` (`/Users/smolik/DEV/cz.en.orch`)
- **Package**: `ZEN_ORCH`
- **Location**: `src/zen_orch/`
- **Serialization**: abapGit XML format. Data elements → `.dtel.xml`, structures → `.doma.xml` is NOT used; structures use `.tabl.xml` (DOMA serializer is for domains). Structures in abapGit are serialized as `LCL_OBJECT_DTEL` for data elements and `LCL_OBJECT_TABL` for structures/table types. **Table types** use `LCL_OBJECT_TABL` serializer as well.

### abapGit XML Format Reference

**Data element** (`.dtel.xml`): uses serializer `LCL_OBJECT_DTEL`. Key fields: `ROLLNAME` (DE name), `DOMNAME` (referenced domain), `DATATYPE`/`LENG` (if no domain), `DDTEXT`, `REPTEXT`, `SCRTEXT_S/M/L`.

**Structure** (`.tabl.xml`): uses serializer `LCL_OBJECT_TABL`, `TABCLASS = INTTAB`. Contains `DD03P_TAB` with field list. Each field has `FIELDNAME`, `ROLLNAME` (the data element), `POSITION`, `KEYFLAG`.

**Table type** (`.tabl.xml`): uses serializer `LCL_OBJECT_TTYP`. Contains `DD40V` with `TYPENAME`, `ROWTYPE` (referenced table name), `DATATYPE = 'REF'` or `TYPEKIND`, `TTYPKIND = 'S'` (standard table).

### Reference: Existing planner repo patterns
- Data elements: `/Users/smolik/DEV/cz.imcg.fast.planner/src/*.dtel.xml`
- Structures: `/Users/smolik/DEV/cz.imcg.fast.planner/src/*.tabl.xml` (INTTAB)
- Table types: `/Users/smolik/DEV/cz.imcg.fast.planner/src/*.ttyp.xml`

### Constitution Compliance
- **Principle I - DDIC-First**: All data elements are DDIC objects; no local TYPE definitions. Custom domains created for ALL DEs (including standalone ones) to comply with Constitution Principle I.
- **Principle II - SAP Standards**: ZEN_ORCH naming prefix, proper DDIC structure.
- **Activation Order**: Layer 2 (data elements) → Layer 3 (structures) → Layer 4 (table types). Table types reference tables defined in Story 1.4, so they activate fully only after 1.4.

## Dev Agent Record

### Agent Model Used
github-copilot/claude-sonnet-4.6

### Debug Log References
- Constitution Principle I compliance: created 6 additional custom domains for the "standalone" DEs (ZEN_ORCH_PERF_UUID, ZEN_ORCH_WU_HANDLE, ZEN_ORCH_REF_ID, ZEN_ORCH_LOOP_ITER, ZEN_ORCH_PARAMS_JSON, ZEN_ORCH_IMPL_CLASS) bringing total domain count to 12.
- Table types ZEN_ORCH_TT_SCORE_STEP and ZEN_ORCH_TT_PERF_STEP reference ZEN_ORCH_SCORE_STEP and ZEN_ORCH_PERF_STEP tables from Story 1.4; they will fully activate only after Story 1.4 is complete.

### Completion Notes List
- 6 domain-backed data elements created (zen_orch_de_status, zen_orch_de_elem_type, zen_orch_de_score_id, zen_orch_de_adapter_type, zen_orch_de_score_seq, zen_orch_de_max_par)
- 6 standalone data elements created with custom domains (zen_orch_perf_uuid, zen_orch_wu_handle, zen_orch_de_ref_id, zen_orch_de_loop_iter, zen_orch_de_params_json, zen_orch_de_impl_class)
- 6 additional custom domains created for standalone DEs (zen_orch_perf_uuid, zen_orch_wu_handle, zen_orch_ref_id, zen_orch_loop_iter, zen_orch_params_json, zen_orch_impl_class)
- Structure ZEN_ORCH_S_START_RESULT created (INTTAB, fields: WORK_UNIT_HANDLE + STATUS)
- Table types ZEN_ORCH_TT_SCORE_STEP and ZEN_ORCH_TT_PERF_STEP created

## File List

- `src/zen_orch/zen_orch_de_status.dtel.xml`
- `src/zen_orch/zen_orch_de_elem_type.dtel.xml`
- `src/zen_orch/zen_orch_de_score_id.dtel.xml`
- `src/zen_orch/zen_orch_de_adapter_type.dtel.xml`
- `src/zen_orch/zen_orch_de_score_seq.dtel.xml`
- `src/zen_orch/zen_orch_de_max_par.dtel.xml`
- `src/zen_orch/zen_orch_perf_uuid.dtel.xml`
- `src/zen_orch/zen_orch_wu_handle.dtel.xml`
- `src/zen_orch/zen_orch_de_ref_id.dtel.xml`
- `src/zen_orch/zen_orch_de_loop_iter.dtel.xml`
- `src/zen_orch/zen_orch_de_params_json.dtel.xml`
- `src/zen_orch/zen_orch_de_impl_class.dtel.xml`
- `src/zen_orch/zen_orch_perf_uuid.doma.xml` (additional domain)
- `src/zen_orch/zen_orch_wu_handle.doma.xml` (additional domain)
- `src/zen_orch/zen_orch_ref_id.doma.xml` (additional domain)
- `src/zen_orch/zen_orch_loop_iter.doma.xml` (additional domain)
- `src/zen_orch/zen_orch_params_json.doma.xml` (additional domain)
- `src/zen_orch/zen_orch_impl_class.doma.xml` (additional domain)
- `src/zen_orch/zen_orch_s_start_result.tabl.xml`
- `src/zen_orch/zen_orch_tt_score_step.ttyp.xml`
- `src/zen_orch/zen_orch_tt_perf_step.ttyp.xml`

## Change Log

- 2026-04-04: Story implemented — all DEs, extra domains, structure, and table types created. Status → review.
