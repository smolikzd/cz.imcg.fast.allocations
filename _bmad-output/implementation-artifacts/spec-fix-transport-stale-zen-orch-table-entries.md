---
title: 'Fix transport release failure: remove stale ZEN_ORCH_* table entries'
type: 'bugfix'
created: '2026-04-08'
status: 'ready-for-dev'
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Releasing the `cz.en.orch` transport request fails with four "nonexistent table without delete flag" errors because the TR still contains the old table names (`ZEN_ORCH_ADAPTER_REG`, `ZEN_ORCH_PERF_STEP`, `ZEN_ORCH_SCHEDULE`, `ZEN_ORCH_SCORE_STEP`) that were renamed before the TR was released. The target system has never seen these names, so SAP's transport tool refuses to release.

**Approach:** Remove the four stale TABL entries from the transport request object list in SE09/SE10. No code changes are needed — the current table names (`ZEN_ORCH_ADPT_R`, `ZEN_ORCH_P_STEP`, `ZEN_ORCH_SCHED`, `ZEN_ORCH_S_STEP`) are already in the TR under their correct names.

## Boundaries & Constraints

**Always:**
- Work only in SE09/SE10 transport organizer — do not touch any ABAP source or DDIC objects.
- Verify the four correct (new-name) table entries are present in the TR before removing the stale ones.
- Keep `ZEN_ORCH_ADPT_R`, `ZEN_ORCH_P_STEP`, `ZEN_ORCH_SCHED`, `ZEN_ORCH_S_STEP` in the TR untouched.

**Ask First:**
- If the four new-name table entries are NOT present in the TR, halt — do not release until they are added.

**Never:**
- Do not rename or delete any DDIC table objects themselves.
- Do not add a delete flag to the stale entries — that would attempt to delete a non-existent object and may cause a different error.
- Do not create a new transport request for this fix — use the existing TR.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Stale entries removed, new names present | TR with 4 stale + 4 correct TABL entries | TR releases successfully, all 4 tables transported | N/A |
| New-name entries missing from TR | TR has stale entries only | HALT — add correct entries before removing stale ones | Do not release |
| Partial rename (only some stale entries exist) | TR has mix | Remove only the stale entries that are present; verify rest | N/A |

</frozen-after-approval>

## Code Map

- `cz.en.orch` transport request (SE09/SE10) — object list containing stale TABL entries
- `/Users/smolik/DEV/cz.en.orch/src/zen_orch_adpt_r.tabl.xml` — correct name for `ZEN_ORCH_ADAPTER_REG`
- `/Users/smolik/DEV/cz.en.orch/src/zen_orch_p_step.tabl.xml` — correct name for `ZEN_ORCH_PERF_STEP`
- `/Users/smolik/DEV/cz.en.orch/src/zen_orch_sched.tabl.xml` — correct name for `ZEN_ORCH_SCHEDULE`
- `/Users/smolik/DEV/cz.en.orch/src/zen_orch_s_step.tabl.xml` — correct name for `ZEN_ORCH_SCORE_STEP`

## Tasks & Acceptance

**Execution:**
- [ ] `SE09/SE10 → TR object list` -- Open the transport request, navigate to the object list (R3TR TABL entries). Verify the four correct table names are present: `ZEN_ORCH_ADPT_R`, `ZEN_ORCH_P_STEP`, `ZEN_ORCH_SCHED`, `ZEN_ORCH_S_STEP`.
- [ ] `SE09/SE10 → TR object list` -- Select and delete the four stale entries: `ZEN_ORCH_ADAPTER_REG`, `ZEN_ORCH_PERF_STEP`, `ZEN_ORCH_SCHEDULE`, `ZEN_ORCH_SCORE_STEP`. Use the standard "delete object from TR" action (not delete flag).
- [ ] `SE09/SE10` -- Save the TR and attempt release. Confirm the four transport errors no longer appear.

**Acceptance Criteria:**
- Given the TR contains stale entries `ZEN_ORCH_ADAPTER_REG`, `ZEN_ORCH_PERF_STEP`, `ZEN_ORCH_SCHEDULE`, `ZEN_ORCH_SCORE_STEP`, when those entries are removed from the object list and the TR is released, then the release completes without "nonexistent table" errors.
- Given the TR still contains `ZEN_ORCH_ADPT_R`, `ZEN_ORCH_P_STEP`, `ZEN_ORCH_SCHED`, `ZEN_ORCH_S_STEP`, when the TR is released, then all four tables are transported to the target system.

## Design Notes

**Old name → new name mapping:**

| Stale name in TR (remove) | Current name (keep) |
|---|---|
| `ZEN_ORCH_ADAPTER_REG` | `ZEN_ORCH_ADPT_R` |
| `ZEN_ORCH_PERF_STEP` | `ZEN_ORCH_P_STEP` |
| `ZEN_ORCH_SCHEDULE` | `ZEN_ORCH_SCHED` |
| `ZEN_ORCH_SCORE_STEP` | `ZEN_ORCH_S_STEP` |

The tables were renamed during development (shorter names to stay within SAP 18-char table name limits and naming conventions). The TR object list was not cleaned up after the rename — the old entries remained alongside the new ones. SAP's transport release check rejects any TABL entry whose object does not exist in the source system without a delete flag.

## Verification

**Manual checks (if no CLI):**
- In SE09/SE10 after release: confirm the TR status is "Released" (green checkmark).
- In target system SE11: confirm tables `ZEN_ORCH_ADPT_R`, `ZEN_ORCH_P_STEP`, `ZEN_ORCH_SCHED`, `ZEN_ORCH_S_STEP` exist and are active.
- In SE11 on target: confirm `ZEN_ORCH_ADAPTER_REG`, `ZEN_ORCH_PERF_STEP`, `ZEN_ORCH_SCHEDULE`, `ZEN_ORCH_SCORE_STEP` do NOT exist (they were never transported, correctly).
