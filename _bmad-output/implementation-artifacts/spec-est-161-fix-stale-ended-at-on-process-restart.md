---
title: 'EST-161: Fix stale ended_at on process restart'
type: 'bugfix'
created: '2026-04-08'
status: 'done'
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** When a process instance is restarted after a failure, the framework transitions status back to `RUNNING` but never clears `ms_instance-ended_at`. The stale failure timestamp is persisted to `ZFI_PROC_INST` by `save_instance()`, causing the Allocation Dashboard to display a non-empty **Ended At** on a running process.

**Approach:** Clear `ms_instance-ended_at` (and `ms_instance-error_message`) in the RUNNING-transition block inside `execute()`, immediately before the counters are reset and `save_instance()` is called.

## Boundaries & Constraints

**Always:**
- Fix is applied exclusively in the RUNNING-transition block (lines ~772–793 of `zcl_fi_process_instance.clas.abap`), not in `save_instance` or `restart`.
- `ended_at` is only ever set in the two terminal-state paths (COMPLETED at line 837, FAILED at line 828) — this invariant must be preserved.
- Constitution Principle II (SAP Standards): line length ≤120 chars; audit fields (changed_by, changed_at) are untouched.

**Ask First:**
- If any other field in `ms_instance` is found to carry stale state from a prior run that could cause a similar display issue (e.g., `error_message` showing old failure text while RUNNING), confirm with user whether to clear it in the same patch.

**Never:**
- Do not clear `ended_at` inside `save_instance` or `save_failure_state` — those are generic persistence helpers and must not carry business logic.
- Do not add a guard condition (e.g. `IF status = FAILED`) — the clear must be unconditional in the RUNNING block so it also covers fresh first-run scenarios where `ended_at` could theoretically be non-initial.
- Do not modify the dashboard CDS views or ABAP query classes — the bug is in the framework data layer.

</frozen-after-approval>

## Code Map

- `src/zcl_fi_process_instance.clas.abap:772-793` — RUNNING-transition block in `execute()`: where `status`, `started_by`, `started_at`, and counters are set before `save_instance()`; `ended_at` is NOT cleared here — this is the fix location
- `src/zcl_fi_process_instance.clas.abap:828` — FAILED path: `GET TIME STAMP FIELD ms_instance-ended_at` (must remain)
- `src/zcl_fi_process_instance.clas.abap:837` — COMPLETED path: `GET TIME STAMP FIELD ms_instance-ended_at` (must remain)
- `src/zcl_fi_process_instance.clas.abap:2050-2051` — `restart()`: calls `execute()` with no field clearing (no change needed there)

## Tasks & Acceptance

**Execution:**
- [ ] `src/zcl_fi_process_instance.clas.abap` -- Add `CLEAR ms_instance-ended_at.` and `CLEAR ms_instance-error_message.` immediately after line 775 (`GET TIME STAMP FIELD ms_instance-started_at.`) and before line 778 (counter resets), in the RUNNING-transition block of `execute()` -- Ensures DB record reflects clean RUNNING state with no stale terminal-state fields from any prior run

**Acceptance Criteria:**
- Given a process instance in FAILED status with a non-empty `ended_at`, when `restart()` is called and `execute()` transitions to RUNNING, then `ZFI_PROC_INST.ENDED_AT` is empty/null after `save_instance()` commits
- Given a running process in the Allocation Dashboard, when step 4 is still PENDING/QUEUED, then the process header **Ended At** column displays `–` (empty), not a timestamp
- Given a process that completes successfully, when the terminal COMPLETED state is saved, then `ZFI_PROC_INST.ENDED_AT` is populated with the correct completion timestamp
- Given a process that fails on a step, when the FAILED state is saved, then `ZFI_PROC_INST.ENDED_AT` is populated with the failure timestamp

## Design Notes

The existing reset block (lines 778–790) already clears all statistics counters on each `execute()` call. The `ended_at` and `error_message` fields are absent from this reset — an oversight, not a design decision. Adding them to the reset block is the minimal, targeted fix.

**Exact change (after line 775):**
```abap
    " Clear terminal-state fields so a restarted instance shows clean RUNNING state
    CLEAR ms_instance-ended_at.
    CLEAR ms_instance-error_message.
```

## Verification

**Manual checks (no CLI available for ABAP):**
- After pushing to SAP via abapGit: restart a previously FAILED allocation run instance; confirm `ZFI_PROC_INST.ENDED_AT` is initial while status = RUNNING (check via SE16 or the dashboard)
- Confirm dashboard **Ended At** column shows `–` while process is Running
- Let the restarted instance run to completion; confirm `ENDED_AT` is populated with the completion time
