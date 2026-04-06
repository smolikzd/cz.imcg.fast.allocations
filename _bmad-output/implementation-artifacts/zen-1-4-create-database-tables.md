# Story 1.4: Create database tables

Status: review

## Story

As a developer,
I want all ZEN_ORCH database tables defined, activated, and client-dependent,
So that the engine can persist scores, performances, adapter registrations, and schedule configuration.

## Acceptance Criteria

1. **Given** all data elements from Story 1.3 exist
2. **When** database tables are activated
3. **Then** `ZEN_ORCH_SCORE` exists with primary key MANDT + SCORE_ID, fields DESCRIPTION (CHAR 60), PARAMS_JSON, and audit fields (CREATED_BY, CREATED_AT, CHANGED_BY, CHANGED_AT)
4. **And** `ZEN_ORCH_SCORE_STEP` exists with primary key MANDT + SCORE_ID + SCORE_SEQ, fields ELEM_TYPE, REF_ID, ADAPTER_TYPE, PARAMS_JSON, MAX_PARALLEL
5. **And** `ZEN_ORCH_PERF` exists with primary key MANDT + PERF_UUID, fields SCORE_ID, STATUS, PARAMS_JSON, and audit fields
6. **And** `ZEN_ORCH_PERF_STEP` exists with primary key MANDT + PERF_UUID + SCORE_SEQ + LOOP_ITERATION, fields ELEM_TYPE, REF_ID, ADAPTER_TYPE, PARAMS_JSON, STATUS, WORK_UNIT_HANDLE
7. **And** `ZEN_ORCH_ADAPTER_REG` exists with primary key MANDT + ADAPTER_TYPE, field IMPL_CLASS
8. **And** `ZEN_ORCH_SCHEDULE` exists with primary key MANDT + SCHEDULE_ID, fields SCORE_ID, PARAMS_JSON, CRON_EXPR, ACTIVE flag
9. **And** all tables are client-dependent (MANDT first key field) and include all four audit fields where applicable
10. **And** all tables activate without errors

## Tasks / Subtasks

- [x] Create additional domains and DEs for new fields (DESCRIPTION, SCHEDULE_ID, CRON_EXPR, ACTIVE) — Constitution I compliance
- [x] Create table ZEN_ORCH_SCORE (AC: 3)
- [x] Create table ZEN_ORCH_SCORE_STEP (AC: 4)
- [x] Create table ZEN_ORCH_PERF (AC: 5)
- [x] Create table ZEN_ORCH_PERF_STEP (AC: 6)
- [x] Create table ZEN_ORCH_ADAPTER_REG (AC: 7)
- [x] Create table ZEN_ORCH_SCHEDULE (AC: 8)
- [x] Verify abapGit serialization format (AC: 10)

## Dev Notes

- **Target Repository**: `cz.en.orch` (`/Users/smolik/DEV/cz.en.orch`)
- **Package**: `ZEN_ORCH`
- **Location**: `src/zen_orch/`
- **Serialization**: abapGit XML format — TRANSP tables use `LCL_OBJECT_TABL` serializer. Pattern: `DD02V` (table header), `DD09L` (technical settings), `DD03P_TABLE` (field list).

### TRANSP Table XML Pattern
```xml
<DD02V>
  <TABNAME>...</TABNAME>
  <DDLANGUAGE>E</DDLANGUAGE>
  <TABCLASS>TRANSP</TABCLASS>
  <CLIDEP>X</CLIDEP>   <!-- client-dependent -->
  <DDTEXT>...</DDTEXT>
  <CONTFLAG>A</CONTFLAG>
  <EXCLASS>1</EXCLASS>
</DD02V>
<DD09L>
  <TABNAME>...</TABNAME>
  <AS4LOCAL>A</AS4LOCAL>
  <TABKAT>0</TABKAT>     <!-- 0=customizing, 3=application data -->
  <TABART>APPL0</TABART> <!-- APPL0 for customizing, APPL1 for application -->
  <BUFALLOW>N</BUFALLOW>
</DD09L>
```
- Key fields: `<KEYFLAG>X</KEYFLAG>` + `<NOTNULL>X</NOTNULL>`
- MANDT field always uses standard DE `MANDT`
- Audit fields: standard SAP DEs — CREATED_BY→`ERNAM`, CREATED_AT→`ERDAT`, CHANGED_BY→`AENAM`, CHANGED_AT→`AEDAT`

### Table Classification
- Customizing (TABKAT=0, APPL0): ZEN_ORCH_SCORE, ZEN_ORCH_SCORE_STEP, ZEN_ORCH_ADAPTER_REG, ZEN_ORCH_SCHEDULE
- Application data (TABKAT=3, APPL1): ZEN_ORCH_PERF, ZEN_ORCH_PERF_STEP

### Constitution Compliance
- **Principle I - DDIC-First**: 4 new custom domains + 4 new custom DEs created (ZEN_ORCH_DESCRIPTION, ZEN_ORCH_SCHEDULE_ID, ZEN_ORCH_CRON_EXPR, ZEN_ORCH_ACTIVE + their DEs). Standard SAP DEs used for MANDT and audit fields.
- **Principle II - SAP Standards**: ZEN_ORCH_ naming prefix, TRANSP with CLIDEP=X, proper technical settings.

### Activation Note
After this story, `ZEN_ORCH_TT_SCORE_STEP` and `ZEN_ORCH_TT_PERF_STEP` from Story 1.3 can be fully activated.

## Dev Agent Record

### Agent Model Used
github-copilot/claude-sonnet-4.6

### Debug Log References
- Created 4 additional domains (ZEN_ORCH_DESCRIPTION, ZEN_ORCH_SCHEDULE_ID, ZEN_ORCH_CRON_EXPR, ZEN_ORCH_ACTIVE) and 4 DEs to satisfy Constitution Principle I for all new field types.
- Audit fields use standard SAP DEs (ERNAM/ERDAT/AENAM/AEDAT) — these are released standard SAP elements.
- ZEN_ORCH_ACTIVE domain has fixed values (X=Active, space=Inactive).
- Table classification: runtime tables (PERF, PERF_STEP) are APPL1 application data; config tables (SCORE, SCORE_STEP, ADAPTER_REG, SCHEDULE) are APPL0 customizing.

### Completion Notes List
- 4 new domains created: zen_orch_description (CHAR 60), zen_orch_schedule_id (CHAR 30), zen_orch_cron_expr (CHAR 100), zen_orch_active (CHAR 1, fixed values)
- 4 new DEs created: zen_orch_de_description, zen_orch_de_schedule_id, zen_orch_de_cron_expr, zen_orch_de_active
- 6 TRANSP tables created: zen_orch_score, zen_orch_score_step, zen_orch_perf, zen_orch_perf_step, zen_orch_adapter_reg, zen_orch_schedule
- All tables are client-dependent (MANDT key field)
- ZEN_ORCH_TT_SCORE_STEP and ZEN_ORCH_TT_PERF_STEP from Story 1.3 are now resolvable

## File List

Additional domains (Constitution I compliance):
- `src/zen_orch/zen_orch_description.doma.xml`
- `src/zen_orch/zen_orch_schedule_id.doma.xml`
- `src/zen_orch/zen_orch_cron_expr.doma.xml`
- `src/zen_orch/zen_orch_active.doma.xml`

Additional data elements (Constitution I compliance):
- `src/zen_orch/zen_orch_de_description.dtel.xml`
- `src/zen_orch/zen_orch_de_schedule_id.dtel.xml`
- `src/zen_orch/zen_orch_de_cron_expr.dtel.xml`
- `src/zen_orch/zen_orch_de_active.dtel.xml`

Database tables:
- `src/zen_orch/zen_orch_score.tabl.xml`
- `src/zen_orch/zen_orch_score_step.tabl.xml`
- `src/zen_orch/zen_orch_perf.tabl.xml`
- `src/zen_orch/zen_orch_perf_step.tabl.xml`
- `src/zen_orch/zen_orch_adapter_reg.tabl.xml`
- `src/zen_orch/zen_orch_schedule.tabl.xml`

## Change Log

- 2026-04-04: Story implemented — 4 extra domains, 4 extra DEs, 6 TRANSP tables created. Status → review.
