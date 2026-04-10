---
title: 'EST-167: Dashboard FiscalYear filter — multiple values and full operators'
type: 'feature'
created: '2026-04-10'
status: 'done'
baseline_commit: '80c9852da693c3d7e185d8fc471a7c86790ac2f3'
baseline_repository: 'ovysledovka'
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The FiscalYear filter on the Allocation Dashboard (`ZFI_I_ALLOC_DASHBOARD_CE`) only accepts a single value with the "equal to" operator. Users cannot select multiple fiscal years (e.g., 2024 and 2025) or use range/comparison operators (e.g., BT, GE), which makes period-spanning analysis impossible.

**Approach:** Add `@Consumption.filter.multipleSelections: true` and `selectionType: #INTERVAL` (or `#RANGE`) to the `FiscalYear` element annotation in the CDS custom entity. The ABAP query provider (`ZCL_FI_ALLOC_DASH_QUERY`) already handles multi-value and range conditions via its `build_range_condition` method — no ABAP changes required.

## Boundaries & Constraints

**Always:**
- Change is confined to `ZFI_I_ALLOC_DASHBOARD_CE` CDS custom entity only.
- `@Consumption.filter.mandatory: true` and `@Consumption.filter.defaultValue: '2026'` are preserved (filter stays required with sensible default).
- Existing key structure, field order, and all other annotations remain unchanged.
- Constitution Principle II: SAP naming conventions and line length ≤120 chars must be respected.

**Ask First:**
- If the SAP system requires activation of related objects (service binding, OData) after CDS change — confirm before activating in production system.

**Never:**
- Do not modify `ZCL_FI_ALLOC_DASH_QUERY` (backend already supports all operators).
- Do not change `ZFI_I_ALLOC_DASH_STEP_CE` or any other entity.
- Do not remove the `mandatory: true` constraint (filter must always have a value).
- Do not split into separate consumption view — the custom entity is intentionally the consumption layer.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Single year (existing behavior) | FiscalYear = '2025' (EQ) | Rows for 2025 returned — unchanged | N/A |
| Multiple discrete years | FiscalYear IN ('2024', '2025') | Rows for both 2024 and 2025 returned | N/A |
| Year range (BETWEEN) | FiscalYear BT '2023' '2025' | Rows for 2023, 2024, 2025 returned | N/A |
| Comparison operators (GE/LE) | FiscalYear GE '2023' | All rows from 2023 onwards | N/A |
| No FiscalYear value | Filter cleared | Filter bar validation prevents query (mandatory: true) | Fiori shows mandatory field validation |

</frozen-after-approval>

## Code Map

- `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` -- CDS custom entity; `FiscalYear` element annotation is the only change target
- `src/zfi_alloc_process/zcl_fi_alloc_dash_query.clas.abap` -- Query provider; `build_range_condition` (line ~707) already handles all operators including multi-value OR and BETWEEN — read-only reference

## Tasks & Acceptance

**Execution:**
- [x] `src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls` -- Replace the `@Consumption.filter.mandatory` block on `FiscalYear` with an expanded annotation that adds `multipleSelections: true` and `selectionType: #INTERVAL` while preserving `mandatory: true` and `defaultValue: '2026'` -- Enables multi-value and range operators on the FiscalYear filter bar input

**Acceptance Criteria:**
- Given the Allocation Dashboard filter bar, when the user opens the FiscalYear input, then the input field allows multiple values (e.g., selecting 2024 and 2025 separately).
- Given multiple FiscalYear values selected (e.g., 2024 and 2025), when the user executes the query, then rows for all selected years are returned in the list.
- Given the FiscalYear filter cleared entirely, when the user attempts to execute, then the filter bar prevents execution with a mandatory field validation message (existing behavior preserved).
- Given FiscalYear filter with a BT (between) range (e.g., 2023–2025), when the user executes, then rows for all years in the range are returned.

## Design Notes

The `@Consumption.filter` annotation change on `FiscalYear`:

```cds
// Before:
@Consumption.filter.mandatory: true
@Consumption.filter.defaultValue: '2026'
key FiscalYear : gjahr;

// After:
@Consumption.filter: {
  mandatory: true,
  defaultValue: '2026',
  multipleSelections: true,
  selectionType: #INTERVAL
}
key FiscalYear : gjahr;
```

`selectionType: #INTERVAL` enables both discrete multi-value selection and range (BT/NB) operators in the Fiori filter bar. The existing `build_range_condition` method in `ZCL_FI_ALLOC_DASH_QUERY` already generates the correct SQL for all `sign`/`option` combinations returned by `get_as_ranges()`.

## Verification

**Manual checks (if no CLI):**
- After CDS activation: Open the Allocation Dashboard Fiori app, verify the FiscalYear filter accepts multiple values and a range input.
- Verify that selecting years 2024+2025 returns data from both years in the list.
- Verify that clearing FiscalYear still shows a mandatory field error.

## Suggested Review Order

- Annotation change: `multipleSelections` + `selectionType: #INTERVAL` added to FiscalYear filter
  [`zfi_i_alloc_dashboard_ce.ddls.asddls:75`](../../../ovysledovka/src/zfi_alloc_process/zfi_i_alloc_dashboard_ce.ddls.asddls#L75)

- Backend range handling reference (no changes needed — all operators already supported)
  [`zcl_fi_alloc_dash_query.clas.abap:707`](../../../ovysledovka/src/zfi_alloc_process/zcl_fi_alloc_dash_query.clas.abap#L707)

## Deferred Items (pre-existing, not caused by this change)

- **DEFER-1**: `apply_alloc_id_postfilter` — AllocationId placeholder stamp guard (`lines( it_alloc_range ) = 1`) silently skips stamping when AllocationId filter has multiple entries. `zcl_fi_alloc_dash_query.clas.abap:802`
- **DEFER-2**: `build_range_condition` — E+NB combination generates double-negation (`NOT ( NOT ... BETWEEN ... )`), logically inverting exclusion intent. `zcl_fi_alloc_dash_query.clas.abap:741`
- **DEFER-3**: `build_where_without_alloc_id` — Unsupported `option` values silently produce empty condition; if FiscalYear was the only filter, unbounded fallback SELECT at line 326 fires. `zcl_fi_alloc_dash_query.clas.abap:760`
