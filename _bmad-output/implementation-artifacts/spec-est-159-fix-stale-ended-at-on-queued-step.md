---
title: 'EST-159: Fix stale ended_at on queued/running step restart'
type: 'bugfix'
created: '2026-04-10'
status: 'done'
context: ['EST-161']
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** When a process is restarted after a failure, steps transitioning back to QUEUED or RUNNING carry stale `ended_at` and `error_message` values from the prior failed run. These fields were never cleared in `update_step_status()`, so `MODIFY zfi_proc_step` persisted the stale terminal-state fields to `ZFI_PROC_STEP`, causing the Allocation Dashboard to display a non-empty **Ended At** on an active (QUEUED/RUNNING) step.

**Relationship to EST-161:** EST-161 fixed the same oversight at the **process level** (`ms_instance-ended_at` in `execute()`). EST-159 is the step-level counterpart: `<ls_step>-ended_at` in `update_step_status()`.

**Approach:** Clear `<ls_step>-ended_at` and `<ls_step>-error_message` in both the RUNNING and QUEUED branches of `update_step_status()`, immediately after stamping `started_at`.

## Boundaries & Constraints

**Always:**
- Fix applied exclusively in `update_step_status()` RUNNING and QUEUED branches.
- `ended_at` is only ever set in the COMPLETED/FAILED branch (line 2003) — this invariant is preserved.
- Constitution Principle II (SAP Standards): line length ≤120 chars; audit fields untouched.

**Never:**
- Do not clear `ended_at` in `save_instance` or generic persistence helpers.
- Do not add a guard on prior status — the clear must be unconditional in the non-terminal branches.

</frozen-after-approval>

## Code Map

- `src/zcl_fi_process_instance.clas.abap:1997-2004` — `update_step_status()`: RUNNING/QUEUED/COMPLETED/FAILED branches; fix applied here
- `src/zcl_fi_process_instance.clas.abap:2015` — `MODIFY zfi_proc_step FROM <ls_step>` — persists the now-clean state

## Tasks & Acceptance

**Execution:**
- [x] `src/zcl_fi_process_instance.clas.abap` — Add `CLEAR <ls_step>-ended_at.` and `CLEAR <ls_step>-error_message.` in RUNNING branch (after `started_at` stamp, before `retry_count` increment) and QUEUED branch (after `started_at` stamp)

**Acceptance Criteria:**
- Given a step in FAILED status with a non-empty `ended_at`, when the process is restarted and the step transitions to QUEUED or RUNNING, then `ZFI_PROC_STEP.ENDED_AT` is empty/null after `MODIFY`
- Given a running process in the Allocation Dashboard, when a step is QUEUED or RUNNING, then the step row **Ended At** column displays `–` (empty)
- Given a step that completes successfully, then `ZFI_PROC_STEP.ENDED_AT` is populated with the correct completion timestamp

## Verification

**Manual checks (no CLI available for ABAP):**
- After pushing to SAP via abapGit: restart a previously FAILED allocation run; confirm `ZFI_PROC_STEP.ENDED_AT` is initial for all QUEUED/RUNNING steps (check via SE16 or the dashboard)
- Let the restarted instance run to completion; confirm `ENDED_AT` is populated for completed steps

## Completion

- **Commit:** `1338972` in `cz.imcg.fast.planner`
- **Completed:** 2026-04-10
