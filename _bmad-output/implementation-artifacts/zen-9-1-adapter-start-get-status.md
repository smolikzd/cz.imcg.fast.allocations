# Story zen-9-1: Adapter start + get_status

```yaml
story_id: "zen-9-1"
epic_id: "zen-9"
title: "ZCL_FI_ALLOC_ORCH_ADAPTER — start and get_status methods"
target_repository: "cz.imcg.fast.ovysledovka"
depends_on:
  - "zen-3-1"   # ZIF_EN_ORCH_ADAPTER interface
  - "zen-4-1"   # ZCL_EN_ORCH_ENGINE (gc_status constants)
constitution_principles:
  - "Principle I - DDIC-First"
  - "Principle II - SAP Standards"
  - "Principle IV - Factory Pattern"
  - "Principle V - Error Handling"
status: "done"
completion_date: "2026-04-11"
commit_hash: "594aae2"
commit_repository: "cz.imcg.fast.ovysledovka"
also_touches:
  - repository: "cz.en.orch"
    commit_hash: "e027652"
    reason: "Added adapter_poll_failed textid to ZCX_EN_ORCH_ERROR"
```

---

## Summary

Implements the first two methods of `ZCL_FI_ALLOC_ORCH_ADAPTER` — the bridge between
`ZEN_ORCH` orchestration and the `ZFI_PROCESS` allocation engine.

**Files changed:**
- `cz.imcg.fast.ovysledovka/src/zcl_fi_alloc_orch_adapter.clas.abap` (all methods, 191 lines)
- `cz.en.orch/src/zcx_en_orch_error.clas.abap` (added `adapter_poll_failed` textid, lines 41-48)

---

## Acceptance Criteria

- AC1: `start` deserializes `iv_params_json` into `ty_params` using `/UI2/CL_JSON`
- AC2: `start` raises `zcx_en_orch_error=>adapter_start_failed` if `process_type IS INITIAL`
- AC3: `start` calls `zcl_fi_process_manager->create_process` with `iv_no_commit = abap_true`
- AC4: `start` calls `request_execute_process` to schedule APJ execution (commits internally)
- AC5: `start` returns handle `'ZFI_PROCESS:<instance_id>'` and status `gc_status-running`
- AC6: `get_status` parses handle via `parse_handle`, loads instance, maps status via `map_status`
- AC7: `get_status` wraps `zcx_fi_process_error` in `zcx_en_orch_error=>adapter_poll_failed`

---

## Code Review Results (2026-04-11)

### Patches Applied

None for start/get_status methods specifically — see zen-9-3 (parse_handle) and zen-9-3 (map_status).

### Decisions

| ID | Finding | Decision |
|----|---------|----------|
| D5 | Exception class boundary (`zcx_fi_process_error` uncaught) | DISMISS — already handled in actual code (TRY/CATCH present) |
| D8 | Wrong textid (`adapter_poll_failed`) for start errors | DISMISS — actual code correctly uses `adapter_start_failed` |
| D10 | `get_result`/`get_detail_link` silent empty | ACCEPTED — intentional fire-and-forget / Phase 4 deferred |

---

## Notes

- The `ZFI_ALLOC_PROCESS_PARAMS` init structure and `zfi_alloc_process_params` field names
  (`allocation_id`, `export`) match the framework DDIC exactly.
- Handle format: `'ZFI_PROCESS:<CHAR32 instance_id>'` — prefix is 12 chars (`gc_handle_prefix`).
- `start` returns `gc_status-running` immediately (APJ not yet polled); engine polls via `get_status`.
- `adapter_poll_failed` textid added to `ZCX_EN_ORCH_ERROR` in cz.en.orch commit `e027652`.
