# Story zen-9-2: Adapter cancel + restart + get_result + get_detail_link

```yaml
story_id: "zen-9-2"
epic_id: "zen-9"
title: "ZCL_FI_ALLOC_ORCH_ADAPTER — cancel, restart, get_result, get_detail_link"
target_repository: "cz.imcg.fast.ovysledovka"
depends_on:
  - "zen-9-1"   # start + get_status already implemented in same class
constitution_principles:
  - "Principle I - DDIC-First"
  - "Principle II - SAP Standards"
  - "Principle V - Error Handling"
status: "done"
completion_date: "2026-04-11"
commit_hash: "594aae2"
commit_repository: "cz.imcg.fast.ovysledovka"
```

---

## Summary

Completes `ZCL_FI_ALLOC_ORCH_ADAPTER` with the remaining four methods of `ZIF_EN_ORCH_ADAPTER`.
All six methods were committed together in commit `594aae2`.

**File changed:**
- `cz.imcg.fast.ovysledovka/src/zcl_fi_alloc_orch_adapter.clas.abap` (lines 123-167)

---

## Acceptance Criteria

- AC1: `cancel` calls `cancel_process` on manager; wraps `zcx_fi_process_error` in `adapter_start_failed`
- AC2: `cancel` has no return value (void) — aligns with `ZIF_EN_ORCH_ADAPTER~cancel` signature
- AC3: `restart` calls `request_restart_process`; returns same handle (instance ID unchanged)
- AC4: `restart` returns `gc_status-running` immediately
- AC5: `restart` wraps `zcx_fi_process_error` in `adapter_start_failed`
- AC6: `get_result` returns empty string `''` (ZFI_PROCESS has no structured result payload)
- AC7: `get_detail_link` returns empty string `''` (Phase 4 deep-link deferred)

---

## Code Review Results (2026-04-11)

### Patches Applied

None — methods are correct as implemented.

### Decisions

| ID | Finding | Decision |
|----|---------|----------|
| D6 | Missing commit after `cancel_process` | DISMISS — `cancel_process` commits by default (`iv_no_commit` defaults to `abap_false`) |
| D7 | `request_restart_process` returns new instance (stale handle) | DISMISS — same instance ID reused; handle unchanged |
| D11 | Setup program no error handling | DISMISS — admin-only tool, low risk, deferred |

---

## Notes

- `cancel` uses `adapter_start_failed` textid (not a new textid) — symmetry with `restart`.
- `restart` intentionally returns the same `iv_handle` in `rs_result-work_unit_handle` because
  `request_restart_process` does not create a new instance; it re-queues the existing one.
- `get_detail_link` is a Phase 4 placeholder; the architecture note in the code points to a
  future allocation dashboard deep link URL.
