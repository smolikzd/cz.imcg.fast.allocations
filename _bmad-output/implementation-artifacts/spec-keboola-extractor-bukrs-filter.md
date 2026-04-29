---
title: 'ZCL_FI_OV_KEBOOLA_EXTRACTOR — optional company code filter (IV_BUKRS)'
type: 'feature'
created: '2026-04-29'
status: 'done'
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** `ZCL_FI_OV_KEBOOLA_EXTRACTOR` currently exports data for all company codes. There is no way to restrict an extraction run to a single company code, making targeted exports (e.g. per-BUKRS Keboola loads) impossible without post-filtering.

**Approach:** Add an optional `IV_BUKRS TYPE BUKRS` parameter to the constructor. When provided: (1) propagate it as an additional `WHERE CompanyCode = @mv_bukrs` condition in both the package-discovery SELECT (`ZFI_I_ALLOC_EXTRACT_P`) and the two data-extraction SELECTs (`ZFI_I_ALLOC_EXTRACT_PD` / `ZFI_I_ALLOC_EXTRACT_PD_NAGG`); (2) inject the BUKRS value into the output filename so files are named `ZFI_OV_{BUKRS}_{PERIOD}_{DATE}_{NNN}_part-{SEGMENT}.csv` instead of the default template. When omitted, behavior and filenames are identical to today.

## Boundaries & Constraints

**Always:**
- `IV_BUKRS` is optional — omitting it must leave existing behavior 100% unchanged.
- Use DDIC type `BUKRS` for the parameter and instance attribute; no local TYPE definitions (Constitution Principle I).
- Line length ≤ 120 characters (Constitution Principle II).
- The WHERE condition must be applied dynamically only when `mv_bukrs IS NOT INITIAL` — never add a redundant blank-field filter.
- Both aggregated (`ZFI_I_ALLOC_EXTRACT_PD`) and non-aggregated (`ZFI_I_ALLOC_EXTRACT_PD_NAGG`) paths must be patched.
- The calling report `ZFI_OV_KEBOOLA_EXTRACT` must expose a new optional selection-screen parameter `P_BUKRS TYPE BUKRS` and pass it to the constructor.

**Ask First:**
- (resolved) Filename: when `IV_BUKRS` is supplied, inject BUKRS after the table prefix — `ZFI_OV_{BUKRS}_{PERIOD}_{DATE}_{NNN}_part-{SEGMENT}.csv`. When omitted, filename is unchanged.

**Never:**
- Do not change the CDS view definitions (`ZFI_I_ALLOC_EXTRACT_P`, `_PD`, `_PD_NAGG`) — the `CompanyCode` field already exists in all three views; only the WHERE clause in ABAP changes.
- Do not make `IV_BUKRS` mandatory — that would be a breaking change.
- Do not filter at the in-memory table level; the restriction must be pushed into the SELECT WHERE clause.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|---|---|---|---|
| No BUKRS supplied | `IV_BUKRS` = initial / not passed | All company codes extracted; filename template unchanged — identical to current behavior | N/A |
| BUKRS supplied, data exists | `IV_BUKRS = '1000'`, period has records for 1000 | Only rows where `CompanyCode = '1000'` extracted; filenames contain `1000` token | N/A |
| BUKRS supplied, no matching data | `IV_BUKRS = '9999'`, no records for 9999 | Zero packages found; trailing file still written with BUKRS in name; `MV_TOTAL_ROWS = 0` | No error raised; normal completion |
| BUKRS supplied, invalid value | Non-existent company code | Same as "no matching data" — SAP SELECT returns 0 rows silently | N/A |

</frozen-after-approval>

## Code Map

- `src/zfi_ea_ov_ext/zcl_fi_ov_keboola_extractor.clas.abap` — main class: constructor, `execute`, `get_data` methods require changes
- `src/zfi_ea_ov_ext/zfi_ov_keboola_extract.prog.abap` — calling report: add `P_BUKRS` selection screen param and pass to constructor
- `src/zfi_ea_ov_ext/zfi_i_alloc_extract_p.ddls.asddls` — package-discovery CDS view; `CompanyCode` already present — read-only reference
- `src/zfi_ea_ov_ext/zfi_i_alloc_extract_pd.ddls.asddls` — data CDS view (aggregated); `CompanyCode` already present — read-only reference
- `src/zfi_ea_ov_ext/zfi_i_alloc_extract_pd_nagg.ddls.asddls` — data CDS view (non-aggregated); read-only reference

## Tasks & Acceptance

**Execution:**
- [x] `src/zfi_ea_ov_ext/zcl_fi_ov_keboola_extractor.clas.abap` -- Add `DATA mv_bukrs TYPE bukrs` to private section; add `IV_BUKRS TYPE BUKRS OPTIONAL` to `CONSTRUCTOR` signature; assign `mv_bukrs = iv_bukrs` in constructor body -- stores the filter for later use
- [x] `src/zfi_ea_ov_ext/zcl_fi_ov_keboola_extractor.clas.abap` -- In `execute` method, branch the `SELECT * FROM zfi_i_alloc_extract_p(...)` on `mv_bukrs IS NOT INITIAL`: when set add `AND companycode = @mv_bukrs` to WHERE -- restricts package discovery to the requested company code
- [x] `src/zfi_ea_ov_ext/zcl_fi_ov_keboola_extractor.clas.abap` -- In `get_data` method, apply the same branching pattern to both SELECT branches (aggregated `ZFI_I_ALLOC_EXTRACT_PD` and non-aggregated `ZFI_I_ALLOC_EXTRACT_PD_NAGG`) -- ensures data rows are also restricted
- [x] `src/zfi_ea_ov_ext/zcl_fi_ov_keboola_extractor.clas.abap` -- In `get_data` and `trailing_file` methods: when `mv_bukrs IS NOT INITIAL`, inject BUKRS into the filename by inserting `_{mv_bukrs}_` between the table prefix (`ZFI_OV_`) and the PERIOD token; when initial, leave filename construction unchanged -- produces `ZFI_OV_1000_202502_20260331_001_part-000031.csv` pattern when BUKRS supplied
- [x] `src/zfi_ea_ov_ext/zfi_ov_keboola_extract.prog.abap` -- Add `PARAMETERS: p_bukrs TYPE bukrs` to selection screen (block bl1, optional); pass `iv_bukrs = p_bukrs` in `NEW zcl_fi_ov_keboola_extractor(...)` call -- exposes filter to end-users running the report manually

**Acceptance Criteria:**
- Given `IV_BUKRS` is not supplied, when `EXECUTE` runs, then the extraction output and filenames are identical to the current baseline (all company codes present).
- Given `IV_BUKRS = '1000'` is supplied, when `EXECUTE` runs, then every row in every exported CSV has `CompanyCode = 1000` and filenames contain the `1000` token (e.g. `ZFI_OV_1000_202502_...`).
- Given `IV_BUKRS = '9999'` (no data), when `EXECUTE` runs, then `MV_TOTAL_ROWS = 0`, the trailing `finished` file is still created (with BUKRS in name) without error.
- Given the report `ZFI_OV_KEBOOLA_EXTRACT` is executed with `P_BUKRS` left blank, when the report runs, then behavior and filenames are unchanged from before this change.

## Design Notes

**Conditional WHERE pattern** — since ABAP Open SQL does not support truly optional WHERE conditions in static SQL, use two separate SELECT statements branched on `mv_bukrs IS NOT INITIAL`:

```abap
IF mv_bukrs IS NOT INITIAL.
  SELECT * INTO TABLE @DATA(lt_packages)
    FROM zfi_i_alloc_extract_p( p_fiscal_year   = @mv_gjahr,
                                 p_fiscal_period = @mv_poper )
    WHERE semantictag IN @mt_semtag_filter
      AND companycode  = @mv_bukrs.
ELSE.
  SELECT * INTO TABLE @lt_packages
    FROM zfi_i_alloc_extract_p( p_fiscal_year   = @mv_gjahr,
                                 p_fiscal_period = @mv_poper )
    WHERE semantictag IN @mt_semtag_filter.
ENDIF.
```

Apply the identical branching pattern in `get_data` for both the aggregated and non-aggregated paths.

**Filename injection** — when `mv_bukrs IS NOT INITIAL`, use `REPLACE` or string template to insert BUKRS between the fixed prefix and the PERIOD token. The `mv_fname` template is user-supplied so do not mutate it; work on a local copy `lv_file`:

```abap
IF mv_bukrs IS NOT INITIAL.
  " Insert BUKRS token: ZFI_OV_ -> ZFI_OV_1000_
  REPLACE FIRST OCCURRENCE OF 'ZFI_OV_' IN lv_file WITH |ZFI_OV_{ mv_bukrs }_|.
ENDIF.
```

Apply in both `get_data` (data file) and `trailing_file` (finished file) after the initial `lv_file = mv_fname` assignment.

## Verification

**Manual checks (if no CLI):**
- After transport: run `ZFI_OV_KEBOOLA_EXTRACT` without `P_BUKRS` — output and filenames must match pre-change baseline.
- Run with `P_BUKRS = <known BUKRS e.g. 1000>` — filenames must contain `1000` token; inspect exported CSV — all `CompanyCode` column values must equal the supplied code.
- Run with `P_BUKRS = <non-existent BUKRS>` — process must complete without dump, trailing file must be created (with BUKRS in name), total rows = 0.
