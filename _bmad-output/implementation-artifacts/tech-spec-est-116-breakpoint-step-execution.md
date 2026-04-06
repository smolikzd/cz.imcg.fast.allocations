---
title: 'EST-116: Breakpoint feature on step level — intentional execution pause and restart'
type: 'feature'
created: '2026-03-31'
status: 'done'
baseline_commit: '7d29aae67ade1fce49b0430935e1d3b6a7621599'
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** There is no way to deliberately pause a process instance before a specific step executes — operators must either let the full run complete or let it fail. This makes it impossible to stage execution or inspect state mid-run without introducing artificial errors.

**Approach:** Add a `BREAKPOINT` boolean flag to `ZFI_PROC_DEF` (step definition) and a `SKIP_BREAKPOINTS` boolean flag to `ZFI_PROC_INST` (instance runtime override). At execution time, when the next step to execute has `BREAKPOINT = true` and the instance does not have `SKIP_BREAKPOINTS = true`, the engine halts the instance with a new `BREAKPOINT` status and commits before executing that step. Each subsequent restart halts at the next breakpoint in sequence. The existing `request_restart` / `restart` / APJ flow is reused to resume — no new Fiori action is introduced.

## Boundaries & Constraints

**Always:**
- `BREAKPOINT` field on `ZFI_PROC_DEF` uses domain `ZFI_PROCESS_BOOLEAN` with new DTEL `ZFI_PROCESS_BREAKPOINT`
- `SKIP_BREAKPOINTS` field on `ZFI_PROC_INST` uses domain `ZFI_PROCESS_BOOLEAN` with new DTEL `ZFI_PROCESS_SKIP_BP`
- Both DTELs follow the exact pattern of `ZFI_PROCESS_ACTIVE` (same domain, same field lengths)
- New domain fixed value `'BREAKPOINT'` added to `ZFI_PROCESS_STATUS` (CHAR 20 — fits)
- New constant `gc_status-breakpoint TYPE zfi_process_status VALUE 'BREAKPOINT'` added to `gc_status`
- Breakpoint check fires in `execute()` loop **after** the already-completed skip (line 782–784), **before** `execute_step()` — the breakpointed step must NOT have started executing
- When halting: set `ms_instance-status = gc_status-breakpoint`, call `save_instance()`, then `RETURN` (no RAISE — this is an intentional, non-error stop)
- `restart()` and `request_restart()` accept `gc_status-breakpoint` exactly like `FAILED` / `RESTREQ`
- `restart()` finds the resume step by searching for the first non-COMPLETED master step (not just FAILED) — this correctly picks up the paused step
- Status value help (`ZFIPROC_I_InstanceStatus_VH`) reads domain fixed values — no CDS change needed

**Ask First:**
- If `SKIP_BREAKPOINTS` needs to be settable via Fiori UI (not just programmatically), a RAP action will be required — confirm before adding it.

**Never:**
- Do not introduce a new Fiori action in this spec — `request_restart` / restart path is sufficient
- Do not add `BREAKPOINT` to the runtime step table `ZFI_PROC_STEP` — definition-time concept only
- Do not change semantics of existing statuses
- All changes go in `cz.imcg.fast.planner` only

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Breakpoint on step 2, no override | Instance RUNNING, step 1 done, step 2 `BREAKPOINT = true`, `SKIP_BREAKPOINTS = false` | Halts before step 2; instance status = `BREAKPOINT`; step 2 not started | N/A |
| Breakpoint, override active | Same as above but `SKIP_BREAKPOINTS = true` | Breakpoint ignored; step 2 executes normally; runs to completion | N/A |
| Consecutive breakpoints (steps 2 + 3) | Both have `BREAKPOINT = true`, no override | First run: halts at step 2. After restart: halts at step 3. After second restart: completes | N/A |
| Breakpoint on step 1 (first step) | Instance NEW, step 1 `BREAKPOINT = true` | Halts immediately; no step executed; status = `BREAKPOINT` | N/A |
| Breakpoint on already-completed step | Step 2 `BREAKPOINT = true` but already `COMPLETED` (e.g. after partial restart) | Breakpoint flag ignored; execution continues to step 3 | N/A |
| Restart from BREAKPOINT status | `request_restart()` called on `BREAKPOINT` instance | APJ job scheduled; status → `RESTREQ`; `restart()` resumes from the paused step | Concurrent status-change guard unchanged |
| No steps have BREAKPOINT | Normal process definition | No change in behavior; full completion as before | N/A |

</frozen-after-approval>

## Code Map

- `src/zfi_process_breakpoint.dtel.xml` -- New DTEL; domain `ZFI_PROCESS_BOOLEAN`; label "Breakpoint"
- `src/zfi_process_skip_bp.dtel.xml` -- New DTEL; domain `ZFI_PROCESS_BOOLEAN`; label "Skip Breakpoints"
- `src/zfi_proc_def.tabl.xml` -- `ZFI_PROC_DEF` step definition table; add `BREAKPOINT` field
- `src/zfi_proc_inst.tabl.xml` -- `ZFI_PROC_INST` instance header table; add `SKIP_BREAKPOINTS` field
- `src/zfi_process_status.doma.xml` -- `ZFI_PROCESS_STATUS` domain; add fixed value `'BREAKPOINT'`
- `src/zcl_fi_process_instance.clas.abap` -- Main engine: new constant, breakpoint check in `execute()`, extended status guards in `restart()` and `request_restart()`
- `src/zcl_fi_process_definition.clas.abap` -- `get_step()` returns full `zfi_proc_def` row incl. new field; no code change needed — reference only

## Tasks & Acceptance

**Execution:**
- [x] `src/zfi_process_breakpoint.dtel.xml` -- CREATE new DTEL `ZFI_PROCESS_BREAKPOINT`, domain `ZFI_PROCESS_BOOLEAN`, label "Breakpoint" -- DDIC-First: step definition flag needs its own DTEL
- [x] `src/zfi_process_skip_bp.dtel.xml` -- CREATE new DTEL `ZFI_PROCESS_SKIP_BP`, domain `ZFI_PROCESS_BOOLEAN`, label "Skip Breakpoints" -- DDIC-First: instance override flag needs its own DTEL
- [x] `src/zfi_proc_def.tabl.xml` -- ADD field `BREAKPOINT ROLLNAME ZFI_PROCESS_BREAKPOINT` after `RETRY_ALLOWED` -- per-step definition flag
- [x] `src/zfi_proc_inst.tabl.xml` -- ADD field `SKIP_BREAKPOINTS ROLLNAME ZFI_PROCESS_SKIP_BP` after `JOB_COUNT` -- per-instance override flag
- [x] `src/zfi_process_status.doma.xml` -- ADD fixed value `'BREAKPOINT'` / "Breakpoint" at position 0011 -- enables domain validation and status VH display
- [x] `src/zcl_fi_process_instance.clas.abap` (~line 26) -- ADD `breakpoint TYPE zfi_process_status VALUE 'BREAKPOINT'` to the `gc_status` structure
- [x] `src/zcl_fi_process_instance.clas.abap` (`execute()` loop ~line 782) -- ADD breakpoint check: after the COMPLETED skip, before `execute_step()`, call `mo_definition->get_step( ls_step-step_number )`; if `ls_def-breakpoint = abap_true AND ms_instance-skip_breakpoints <> abap_true` then set `ms_instance-status = gc_status-breakpoint`, call `save_instance()`, `RETURN`
- [x] `src/zcl_fi_process_instance.clas.abap` (`restart()` ~line 1963) -- EXTEND status guard to allow `gc_status-breakpoint`; CHANGE the resume-step lookup to "find first master step with status NOT IN (COMPLETED, SKIPPED)" (sorted by step_number ascending) — SKIPPED is a terminal non-runnable status and must be excluded alongside COMPLETED
- [x] `src/zcl_fi_process_instance.clas.abap` (`execute()` breakpoint halt) -- ADD BAL log message when halting on breakpoint — all other terminal transitions (FAILED, COMPLETED) log; breakpoint must too
- [x] `src/zcl_fi_process_instance.clas.abap` (`request_restart()` ~line 1993) -- EXTEND status guard to allow `gc_status-breakpoint`

**Acceptance Criteria:**
- Given a step definition with `BREAKPOINT = true` and `SKIP_BREAKPOINTS` not set, when `execute()` reaches that step, then the instance status is persisted as `BREAKPOINT` and that step has not executed
- Given consecutive breakpoints on steps 2 and 3, when execute → restart → restart is performed, then instance halts at step 2 on first run, halts at step 3 after first restart, completes after second restart
- Given `SKIP_BREAKPOINTS = true` on the instance, when `execute()` runs past a breakpointed step, then the breakpoint is ignored and execution continues normally
- Given an instance with status `BREAKPOINT`, when `request_restart()` is called, then it schedules an APJ job and transitions to `RESTREQ` without raising an exception
- Given an instance with status `RESTREQ` (coming from a breakpoint), when `restart()` is called, then execution resumes from the paused step (first non-COMPLETED master step)
- Given all steps have `BREAKPOINT = false` (or initial), when `execute()` runs, then behavior is identical to before (no regression)
- Given a breakpointed step that is already `COMPLETED` (restart scenario), when execution reaches it, then the breakpoint flag is ignored and execution continues

## Spec Change Log

_2026-03-31: Added `SKIP_BREAKPOINTS` instance-level override flag (new DTEL `ZFI_PROCESS_SKIP_BP`, new field on `ZFI_PROC_INST`). Confirmed: consecutive breakpoints halt one at a time per restart cycle. Removed "Ask First" item for override semantics — resolved as disable-all boolean._

_2026-03-31 (review loopback 1): bad_spec #6 — `restart()` resume-step lookup excluded only COMPLETED; SKIPPED is also terminal and non-runnable. Fixed: exclude both COMPLETED and SKIPPED. bad_spec #8 — BREAKPOINT halt produced no BAL log; all other terminal transitions log. Added log task. KEEP: execute() guard extension, request_restart() guard, all DDIC artefacts, breakpoint check placement and RETURN pattern are correct and must survive re-derivation._

## Design Notes

**`restart()` resume-step lookup change** — the current implementation searches for `status = FAILED`. A breakpointed instance has no failed step; the paused step is still `PENDING` (or `NEW`). Revised approach: find the first master step (`substep_number = '000000000000'`) whose status is not `COMPLETED` and not `SKIPPED` (both are terminal non-runnable statuses), sorted by `step_number` ascending. Raise `no_failed_step` if no such step exists.

```abap
" Find first resumable master step (excludes COMPLETED and SKIPPED — both non-runnable)
LOOP AT mt_steps INTO DATA(ls_resume_step)
  WHERE substep_number = '000000000000'
    AND status        <> gc_status-completed
    AND status        <> gc_status-skipped
  ORDER BY step_number ASCENDING.
  EXIT.
ENDLOOP.
```

**`ms_instance-skip_breakpoints`** is read directly from the in-memory instance struct (populated by `load()` from `ZFI_PROC_INST`) — no extra SELECT needed inside the execute loop.

## Verification

**Manual checks (if no CLI):**
- Activate DDIC objects in order: DTELs → TABL (`ZFI_PROC_DEF`) → TABL (`ZFI_PROC_INST`) → DOMA (`ZFI_PROCESS_STATUS`)
- Set `BREAKPOINT = true` on step 2 of a test process; run → verify instance is `BREAKPOINT`, step 2 untouched
- Call `request_restart()` → verify `RESTREQ`; let APJ run → verify resumes at step 2 and completes (if no further breakpoints)
- Set `BREAKPOINT = true` on steps 2 and 3; verify two restart cycles needed before completion
- Set `SKIP_BREAKPOINTS = true` on the instance; run → verify both breakpoints ignored, full completion in one run
- Run a process with no breakpoints → verify identical outcome to before

## Suggested Review Order

**Execution engine — core logic**

- Entry point: breakpoint check inserted in the step loop, clean `RETURN` halt
  [`zcl_fi_process_instance.clas.abap:789`](../../../cz.imcg.fast.planner/src/zcl_fi_process_instance.clas.abap#L789)

- `restart()` rewritten: status guard extended + SKIPPED excluded from resume lookup
  [`zcl_fi_process_instance.clas.abap:1982`](../../../cz.imcg.fast.planner/src/zcl_fi_process_instance.clas.abap#L1982)

- `request_restart()` guard extended to allow BREAKPOINT status
  [`zcl_fi_process_instance.clas.abap:2025`](../../../cz.imcg.fast.planner/src/zcl_fi_process_instance.clas.abap#L2025)

- `execute()` status guard extended: BREAKPOINT allowed via restart path
  [`zcl_fi_process_instance.clas.abap:690`](../../../cz.imcg.fast.planner/src/zcl_fi_process_instance.clas.abap#L690)

- New `gc_status-breakpoint` constant alongside all other status constants
  [`zcl_fi_process_instance.clas.abap:26`](../../../cz.imcg.fast.planner/src/zcl_fi_process_instance.clas.abap#L26)

**DDIC — schema changes**

- New `BREAKPOINT` field on step definition table
  [`zfi_proc_def.tabl.xml:77`](../../../cz.imcg.fast.planner/src/zfi_proc_def.tabl.xml#L77)

- New `SKIP_BREAKPOINTS` field on instance header table
  [`zfi_proc_inst.tabl.xml:244`](../../../cz.imcg.fast.planner/src/zfi_proc_inst.tabl.xml#L244)

- New `BREAKPOINT` fixed value in status domain (drives VH automatically)
  [`zfi_process_status.doma.xml:71`](../../../cz.imcg.fast.planner/src/zfi_process_status.doma.xml#L71)

**DDIC — new data elements**

- `ZFI_PROCESS_BREAKPOINT` DTEL: reuses `ZFI_PROCESS_BOOLEAN` domain
  [`zfi_process_breakpoint.dtel.xml:1`](../../../cz.imcg.fast.planner/src/zfi_process_breakpoint.dtel.xml#L1)

- `ZFI_PROCESS_SKIP_BP` DTEL: reuses `ZFI_PROCESS_BOOLEAN` domain
  [`zfi_process_skip_bp.dtel.xml:1`](../../../cz.imcg.fast.planner/src/zfi_process_skip_bp.dtel.xml#L1)
