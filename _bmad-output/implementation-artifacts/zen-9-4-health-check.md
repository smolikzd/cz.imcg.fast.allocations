# Story zen-9-4: End-to-end integration test (check_alloc_adapter health check)

```yaml
story_id: "zen-9-4"
epic_id: "zen-9"
title: "check_alloc_adapter health check in ZCL_EN_ORCH_HEALTH_CHK_QUERY"
target_repository: "cz.en.orch"
depends_on:
  - "zen-9-1"
  - "zen-9-2"
  - "zen-9-3"
constitution_principles:
  - "Principle I - DDIC-First"
  - "Principle II - SAP Standards"
  - "Principle V - Error Handling"
status: "done"
completion_date: "2026-04-11"
commit_hash: "d3a206a"
commit_repository: "cz.en.orch"
notes: "Fix commits applied on top of d3a206a up to a73be2d"
```

---

## Summary

Adds `check_alloc_adapter` (Phase 14) to `ZCL_EN_ORCH_HEALTH_CHK_QUERY`.
Score ID: `'ZALLOC_ADPT_CHK'`.

**Files changed:**
- `cz.en.orch/src/zcl_en_orch_health_chk_query.clas.abap` (lines 3276-3415)

---

## Acceptance Criteria

- AC1: `check_alloc_adapter` appends a row with score `'ZALLOC_ADPT_CHK'` to `mt_rows`
- AC2: Returns GREY if `ALLOC_STEP` score is not configured in `ZEN_ORCH_SCORE` (not yet set up)
- AC3: Uses a MOCK adapter (not the real ZCL_FI_ALLOC_ORCH_ADAPTER) to avoid creating real ZFI_PROCESS instances
- AC4: MOCK adapter: `create_performance` → `sweep_all` → verify p_step STATUS is `R` or `C`
- AC5: Returns GREEN if sweep succeeds and step ends in R or C
- AC6: Returns RED otherwise (engine error, wrong status)
- AC7: Unconditional cleanup after test (delete test perf + steps regardless of outcome)

---

## Code Review Results (2026-04-11)

### Patches Applied

None — health check implementation is correct as committed.

### Decisions

| ID | Finding | Decision |
|----|---------|----------|
| D9 | Health check uses MOCK, not real adapter | ACCEPTED — intentional per spec AC3; creating real ZFI_PROCESS instances in a health check is undesirable |

---

## Notes

- The health check intentionally uses the mock adapter to test the engine–adapter contract
  without triggering real allocation jobs.
- GREY (not configured) is the correct initial state before `ZEN_ORCH_SETUP` is run.
- Cleanup is unconditional (`CATCH` all, delete rows, `ROLLBACK WORK`) to prevent test
  data from polluting `ZEN_ORCH_PERF`/`ZEN_ORCH_P_STEP`.
- Fix commits up to `a73be2d` addressed activation and minor runtime errors in the
  health check class after initial commit.
