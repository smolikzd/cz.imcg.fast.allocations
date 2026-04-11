# Story zen-9-3: Adapter registration, parse_handle, map_status + smoke score

```yaml
story_id: "zen-9-3"
epic_id: "zen-9"
title: "Adapter registration, parse_handle fix, map_status fix + ZEN_ORCH_SETUP smoke score"
target_repository: "both"
depends_on:
  - "zen-9-1"
  - "zen-9-2"
constitution_principles:
  - "Principle I - DDIC-First"
  - "Principle II - SAP Standards"
  - "Principle V - Error Handling"
status: "done"
completion_date: "2026-04-11"
commit_hash: "af506cf"
commit_repository: "cz.en.orch"
also_touches:
  - repository: "cz.imcg.fast.ovysledovka"
    commit_hash: "code-review-patch-2026-04-11"
    reason: "parse_handle and map_status patches applied during code review"
```

---

## Summary

### Framework changes (`cz.en.orch`)

- **ZEN_ORCH_SETUP** (`zen_orch_setup.prog.abap`, commit `af506cf`): Idempotent setup program
  using `MODIFY`-based upserts to register the ZFI_PROCESS adapter and define the ALLOC_STEP score.
  - Inserts/updates `ZEN_ORCH_ADPT_R`: key `'ZFI_PROCESS'`, class `'ZCL_FI_ALLOC_ORCH_ADAPTER'`
  - Inserts/updates `ZEN_ORCH_SCORE`: key `'ALLOC_STEP'`, single-step sequential score
  - Inserts/updates `ZEN_ORCH_S_STEP`: step row with `score_id='ALLOC_STEP'`, `adapter_type='ZFI_PROCESS'`
  - Params JSON placeholder: `COMPANY_CODE=1000` — must be adjusted before running in target system

### Implementation changes (`cz.imcg.fast.ovysledovka`)

Two correctness patches applied during code review to `zcl_fi_alloc_orch_adapter.clas.abap`:

**Patch A — `parse_handle` (lines 169-183, post-patch):**
- Replaced `NS` operator with an explicit prefix check at position 0:
  `IF strlen( iv_handle ) <= lv_prefix_len OR iv_handle(lv_prefix_len) <> gc_handle_prefix`
- Replaced hardcoded `+12` offset with dynamic `+lv_prefix_len`
- Added empty instance_id guard (length check prevents empty extraction)

**Patch B — `map_status` (lines 182-190, post-patch):**
- Added `WHEN 'SKIPPED' THEN gc_status-completed` — terminal success state
- Added `WHEN 'BREAKPOINT' THEN gc_status-failed` — surfaced as failure so engine can act

---

## Acceptance Criteria

- AC1: `ZEN_ORCH_SETUP` runs without error in a fresh system (MODIFY is idempotent)
- AC2: After setup, `ZEN_ORCH_ADPT_R` contains a row for `ZFI_PROCESS` → `ZCL_FI_ALLOC_ORCH_ADAPTER`
- AC3: After setup, `ZEN_ORCH_SCORE` contains `ALLOC_STEP` score with one step
- AC4: `parse_handle` raises for: empty handle, handle shorter than prefix, handle with wrong prefix
- AC5: `parse_handle` correctly extracts instance_id for a valid handle of any length ≥ prefix+1
- AC6: `map_status` maps COMPLETED, SUPERSEDED, SKIPPED → `gc_status-completed`
- AC7: `map_status` maps FAILED, BREAKPOINT → `gc_status-failed`
- AC8: `map_status` maps CANCELLED → `gc_status-cancelled`
- AC9: `map_status` maps NEW, PENDING, QUEUED, RUNNING, EXECREQ, RESTREQ → `gc_status-running`

---

## Code Review Results (2026-04-11)

### Patches Applied

| # | Location | Change |
|---|----------|--------|
| A | `parse_handle` (lines 169-183) | NS→prefix-at-pos-0 check; +12→+lv_prefix_len; empty guard via length check |
| B | `map_status` (lines 182-190) | Added SKIPPED→completed and BREAKPOINT→failed branches |

### Decisions

| ID | Finding | Decision |
|----|---------|----------|
| D1 | `parse_handle` — `NS` checks substring anywhere, not prefix at position 0 | **PATCH A** |
| D2 | `parse_handle` — hardcoded `+12` magic number | **PATCH A** (combined) |
| D3 | `parse_handle` — no guard for empty instance_id | **PATCH A** (combined) |
| D4 | `map_status` — `SKIPPED`/`BREAKPOINT` unmapped (fell to ELSE→running) | **PATCH B** |
| D11 | Setup program no error handling | DISMISS — admin-only tool, low risk |
| D12 | JSON deserialize silent failure | DISMISS — `process_type IS INITIAL` guard catches common case |

---

## Notes

- `parse_handle` now uses `DATA(lv_prefix_len) = strlen( gc_handle_prefix )` so the offset
  is always correct even if `gc_handle_prefix` changes.
- `BREAKPOINT` mapped to `failed` (not `cancelled`) because the paused state is unexpected
  from the engine's perspective and should surface for operator attention.
- `SKIPPED` mapped to `completed` because ZFI_PROCESS skips are equivalent to a successful
  no-op pass — the engine should advance to the next step.
- ZEN_ORCH_SETUP params use `COMPANY_CODE=1000` as a placeholder; must be adjusted before
  production use (system-specific).
