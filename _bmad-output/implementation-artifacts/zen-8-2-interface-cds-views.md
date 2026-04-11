# Story zen-8-2: Interface CDS Views for Performance and Step

```yaml
story_id: "zen-8-2"
epic_id: "zen-8"
title: "Interface CDS Views for Performance and Step"
target_repository: "cz.en.orch"
depends_on: ["zen-8-1"]
constitution_principles:
  - "Principle I — DDIC-First"
  - "Principle II — SAP Standards"
  - "Principle III — Consult SAP Docs"
status: "review"
```

---

## User Story

As a developer,
I want interface CDS views over `ZEN_ORCH_PERF` and `ZEN_ORCH_P_STEP`,
So that the consumption view and RAP BDEF have a clean, well-documented data layer.

---

## Pre-Analysis

### Data Sources (from zen-8-1, committed 415bbcd)

**`ZEN_ORCH_PERF` fields:**
- `MANDT` (client key)
- `PERF_UUID` key — `ZEN_ORCH_PERF_UUID`
- `SCORE_ID` — `ZEN_ORCH_DE_SCORE_ID`
- `STATUS` — `ZEN_ORCH_DE_STATUS` (P/R/B/C/F/X)
- `PARAMS_JSON` — `ZEN_ORCH_DE_PARAMS_JSON`
- `PARAMS_HASH` — `ZEN_ORCH_DE_PARAMS_HASH`
- `CREATED_BY` — `ERNAM`
- `CREATED_AT` — `ERDAT` (DATS)
- `CHANGED_BY` — `AENAM`
- `CHANGED_AT` — `AEDAT` (DATS)
- `STARTED_AT` — `ZEN_ORCH_DE_TIMESTAMP` (TIMESTAMPL = DEC15)
- `ENDED_AT` — `ZEN_ORCH_DE_TIMESTAMP` (TIMESTAMPL = DEC15)

**`ZEN_ORCH_P_STEP` fields:**
- `MANDT` (client key)
- `PERF_UUID` key — `ZEN_ORCH_PERF_UUID`
- `SCORE_SEQ` key — `ZEN_ORCH_DE_SCORE_SEQ`
- `LOOP_ITERATION` key — `ZEN_ORCH_DE_LOOP_ITER`
- `ELEM_TYPE` — `ZEN_ORCH_DE_ELEM_TYPE`
- `REF_ID` — `ZEN_ORCH_DE_REF_ID`
- `ADAPTER_TYPE` — `ZEN_ORCH_DE_ADAPTER_TYPE`
- `PARAMS_JSON` — `ZEN_ORCH_DE_PARAMS_JSON`
- `STATUS` — `ZEN_ORCH_DE_STATUS`
- `WORK_UNIT_HANDLE` — `ZEN_ORCH_WU_HANDLE`
- `IS_BREAKPOINT` — `ABAP_BOOLEAN`
- `STARTED_AT` — `ZEN_ORCH_DE_TIMESTAMP`
- `ENDED_AT` — `ZEN_ORCH_DE_TIMESTAMP`

### Duration Calculation

`ZEN_ORCH_DE_TIMESTAMP` is based on domain `TIMESTAMPL` = `DEC(15,0)` (packed ABAP timestamp format `YYYYMMDDHHMMSS`).

**Correct CDS function:** `TSTMP_SECONDS_BETWEEN(started_at, ended_at, 'NULL')` — returns null when either argument is null (i.e., not yet ended). Returns `DEC(21,7)` which we cast to `abap.int4` for display.

**Pattern:**
```cds
cast( case when ended_at is not null and started_at is not null
           then tstmp_seconds_between(started_at, ended_at, 'NULL')
           else null
      end as abap.int4 ) as DurationSec
```

### StatusCriticality Mapping

| Status | Meaning | Criticality |
|--------|---------|-------------|
| `C` | COMPLETED | 3 (green) |
| `F` | FAILED | 1 (red) |
| `X` | CANCELLED | 1 (red) |
| `R` | RUNNING | 2 (orange) |
| `B` | BREAKPOINT | 2 (orange) |
| `P` | PENDING | 0 (grey) |

```cds
cast( case status
        when 'C' then 3
        when 'F' then 1
        when 'X' then 1
        when 'R' then 2
        when 'B' then 2
        else 0
      end as abap.int1 ) as StatusCriticality
```

### abapGit File Structure

Each CDS view = 3 files:
- `<name>.ddls.asddls` — CDS DDL source
- `<name>.ddls.xml` — abapGit metadata (DDLNAME, DDTEXT, SOURCE_TYPE)
- `<name>.ddls.baseinfo` — dependency info (JSON)

SOURCE_TYPE for standard CDS view entity = `V` (not `Q` which is custom entity).

---

## Acceptance Criteria

**AC1 — `ZEN_ORCH_I_PERF` interface view:**
- `define view entity ZEN_ORCH_I_PERF` on `zen_orch_perf`
- Exposes all fields with CamelCase aliases
- `DurationSec` calculated field (integer seconds, NULL if not ended)
- `StatusCriticality` calculated field (0/1/2/3)
- `@Semantics.systemDateTime.createdAt: true` on `CreatedAt`
- `@EndUserText.label` on each field
- Activates without errors

**AC2 — `ZEN_ORCH_I_P_STEP` interface view:**
- `define view entity ZEN_ORCH_I_P_STEP` on `zen_orch_p_step`
- Exposes all fields with CamelCase aliases
- Same `DurationSec` and `StatusCriticality` pattern
- `@EndUserText.label` on each field
- Activates without errors

---

## Tasks / Subtasks

- [x] Task 1 — Implement `ZEN_ORCH_I_PERF` (3 files)
  - [x] `zen_orch_i_perf.ddls.asddls`
  - [x] `zen_orch_i_perf.ddls.xml`
  - [x] `zen_orch_i_perf.ddls.baseinfo`

- [x] Task 2 — Implement `ZEN_ORCH_I_P_STEP` (3 files)
  - [x] `zen_orch_i_p_step.ddls.asddls`
  - [x] `zen_orch_i_p_step.ddls.xml`
  - [x] `zen_orch_i_p_step.ddls.baseinfo`

- [x] Task 3 — Verify line length ≤ 120 chars for all CDS source lines
- [x] Task 4 — Commit to `cz.en.orch`

---

## Constitution Compliance

| Principle | Compliance |
|-----------|------------|
| I — DDIC-First | All field types from DDIC data elements; no local types |
| II — SAP Standards | `ZEN_ORCH_I_*` naming; `@EndUserText.label` on each field |
| III — Consult SAP Docs | `TSTMP_SECONDS_BETWEEN` verified from ABAP docs (DEC15 TIMESTAMPL format) |
| IV — Factory Pattern | N/A — CDS views only |
| V — Error Handling | N/A — CDS views only |

---

## Dev Notes

### DurationSec: why TSTMP_SECONDS_BETWEEN not UTCL_SECONDS_BETWEEN

`ZEN_ORCH_DE_TIMESTAMP` is `TIMESTAMPL` (domain) = DEC(15,0) — the ABAP packed timestamp format (`YYYYMMDDHHMMSSF7`). This is NOT a `UTCLONG` type. Therefore:
- Use `TSTMP_SECONDS_BETWEEN(tstmp1, tstmp2, 'NULL')` — for DEC15 packed timestamps
- Do NOT use `UTCL_SECONDS_BETWEEN` — that requires UTCLONG type
- `on_error = 'NULL'` returns NULL when either arg is initial/invalid (correct for not-yet-ended)

### abapGit SOURCE_TYPE

- Custom entity (e.g., health check): `SOURCE_TYPE = Q`
- Standard view entity (SELECT FROM table): `SOURCE_TYPE = V`

### References

- epics-zen-phase3.md#Story 8.2
- ABAP docs: ABENCDS_TIMESTAMP_FUNCTIONS_V2
- Pattern: `zen_orch_i_health_chk_ce.ddls.*` (same 3-file pattern, different SOURCE_TYPE)

---

## Dev Agent Record

### Agent Model Used

github-copilot/claude-sonnet-4.6

### Completion Notes List

Implementation date: 2026-04-11

Created 6 files in `cz.en.orch/src/`:
- `zen_orch_i_perf.ddls.asddls` — interface view on ZEN_ORCH_PERF
- `zen_orch_i_perf.ddls.xml`
- `zen_orch_i_perf.ddls.baseinfo`
- `zen_orch_i_p_step.ddls.asddls` — interface view on ZEN_ORCH_P_STEP
- `zen_orch_i_p_step.ddls.xml`
- `zen_orch_i_p_step.ddls.baseinfo`

### File List

- `src/zen_orch_i_perf.ddls.asddls` (new)
- `src/zen_orch_i_perf.ddls.xml` (new)
- `src/zen_orch_i_perf.ddls.baseinfo` (new)
- `src/zen_orch_i_p_step.ddls.asddls` (new)
- `src/zen_orch_i_p_step.ddls.xml` (new)
- `src/zen_orch_i_p_step.ddls.baseinfo` (new)
