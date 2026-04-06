---
title: 'EST-116b: Instance-level breakpoint API — set/reset breakpoint on step instance'
type: 'feature'
created: '2026-03-31'
status: 'ready-for-dev'
baseline_commit: '76f68c7589042bf4b3c969191447a2b060010f53'
depends_on: 'EST-116 (tech-spec-est-116-breakpoint-step-execution.md)'
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** After EST-116, breakpoints can only be configured statically at process-definition level (`ZFI_PROC_DEF.BREAKPOINT`). There is no way for a caller to set or clear a breakpoint on a specific step of an already-created process instance at runtime — without modifying the shared process definition.

**Approach:** Add a `BREAKPOINT` column to `ZFI_PROC_STEP` (the runtime step table). Expose `set_step_breakpoint( iv_step_number, iv_breakpoint )` on `ZCL_FI_PROCESS_INSTANCE` and a corresponding manager-level wrapper `set_step_breakpoint_process`. Adjust the breakpoint evaluation in `execute()` to check the instance-level flag first; fall back to the definition-level flag. The `SKIP_BREAKPOINTS` field on `ZFI_PROC_INST` is **removed** as it is superseded by this more precise per-step mechanism.

## Boundaries & Constraints

**Always:**
- `BREAKPOINT` field on `ZFI_PROC_STEP` uses the existing domain `ZFI_PROCESS_BOOLEAN` with existing DTEL `ZFI_PROCESS_BREAKPOINT` — no new DTEL needed
- `set_step_breakpoint` modifies `mt_steps` in memory and calls `save_instance()` — same pattern as `set_substep_status`
- `set_step_breakpoint` accepts only **master steps** (`substep_number = '000000000000'`); raise `zcx_fi_process_error=>step_not_found` if the step does not exist in `mt_steps`
- `iv_breakpoint` is of type `abap_bool`; `abap_true` sets the breakpoint, `abap_false` clears it
- `iv_no_commit TYPE abap_bool DEFAULT abap_false` is included, following all other mutating methods
- Breakpoint evaluation order in `execute()`:
  1. If `ls_step-breakpoint = abap_true` → halt (instance-level takes precedence)
  2. Else if `ls_step_def-breakpoint = abap_true` → halt (definition-level fallback)
  3. Otherwise → proceed to `execute_step()`
- `SKIP_BREAKPOINTS` field is **removed** from `ZFI_PROC_INST`, DTEL `ZFI_PROCESS_SKIP_BP` is deleted, and all references in `execute()` are removed
- `initialize_instance()` copies `ZFI_PROC_DEF.BREAKPOINT` into `ZFI_PROC_STEP.BREAKPOINT` when building `mt_steps` — so definition-level breakpoints are pre-seeded into the instance and can be overridden per-step
- Manager wrapper `set_step_breakpoint_process( iv_instance_id, iv_step_number, iv_breakpoint )` follows the pattern of `request_restart_process` — loads instance, delegates, returns
- BAL log message written on each `set_step_breakpoint` call (severity 'I', message class `ZFI_PROCESS`, new message number; text: "Breakpoint on step &1 of instance &2 set to &3")
- All changes go in `cz.imcg.fast.planner` only

**Ask First:**
- If a new BAL message number must be reserved, confirm the next available number in `ZFI_PROCESS` message class before implementing

**Never:**
- Do not expose `set_step_breakpoint` via RAP action (out of scope)
- Do not add `BREAKPOINT` to substep rows — master step only
- Do not change the `BREAKPOINT` field on `ZFI_PROC_DEF` — it remains as a default/template value

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Set breakpoint on existing step | Instance loaded, step 2 exists (master), `iv_breakpoint = true` | `mt_steps[ step 2 ]-breakpoint = true`, persisted, BAL log written | — |
| Clear breakpoint on existing step | Step 2 has `breakpoint = true`, `iv_breakpoint = false` | `mt_steps[ step 2 ]-breakpoint = false`, persisted, BAL log written | — |
| Step does not exist | `iv_step_number` not in `mt_steps` | `zcx_fi_process_error=>step_not_found` raised | Caller must handle |
| Step is a substep | `substep_number <> '000000000000'` for that step_number | `zcx_fi_process_error=>step_not_found` raised (only master steps accepted) | Caller must handle |
| Instance-level breakpoint = true, def = false | Step has `ZFI_PROC_STEP.BREAKPOINT = true`, `ZFI_PROC_DEF.BREAKPOINT = false` | Engine halts — instance flag wins | — |
| Instance-level breakpoint = false, def = true | Step has `ZFI_PROC_STEP.BREAKPOINT = false`, `ZFI_PROC_DEF.BREAKPOINT = true` | Engine halts — definition flag fires as fallback | — |
| Both = false | Both flags false | Engine does not halt — continues to `execute_step()` | — |
| Definition breakpoint seeded at init | `ZFI_PROC_DEF.BREAKPOINT = true` on step 3 | `initialize_instance()` copies `true` into `ZFI_PROC_STEP.BREAKPOINT` for step 3 | — |
| `no_commit = true` passed | Any call with `iv_no_commit = true` | `save_instance( iv_no_commit = true )` called — no COMMIT issued | — |

</frozen-after-approval>

## Code Map

- `src/zfi_proc_step.tabl.xml` — `ZFI_PROC_STEP` runtime step table; add `BREAKPOINT` field (ROLLNAME `ZFI_PROCESS_BREAKPOINT`) after `QUEUE_ID`
- `src/zfi_proc_inst.tabl.xml` — `ZFI_PROC_INST` instance header; **remove** `SKIP_BREAKPOINTS` field
- `src/zfi_process_skip_bp.dtel.xml` — **delete** DTEL `ZFI_PROCESS_SKIP_BP` (no longer referenced)
- `src/zcl_fi_process_instance.clas.abap` — New public method `set_step_breakpoint`; updated `execute()` breakpoint check; `initialize_instance()` copies DEF breakpoint into step; remove all `skip_breakpoints` references
- `src/zcl_fi_process_manager.clas.abap` — New public method `set_step_breakpoint_process`

## Tasks & Acceptance

**Execution:**

- [ ] `src/zfi_proc_step.tabl.xml` — ADD field `BREAKPOINT ROLLNAME ZFI_PROCESS_BREAKPOINT` after `QUEUE_ID` — per-step instance-level breakpoint flag; reuses existing DTEL
- [ ] `src/zfi_proc_inst.tabl.xml` — REMOVE field `SKIP_BREAKPOINTS` and its `DD03P` block — superseded by per-step instance flag
- [ ] `src/zfi_process_skip_bp.dtel.xml` — DELETE file from repository — DTEL is no longer referenced by any table
- [ ] `src/zcl_fi_process_instance.clas.abap` (`initialize_instance()` ~line 537) — ADD copy of `ls_def_step-breakpoint` to `APPEND VALUE zfi_proc_step(...)` block: `breakpoint = ls_def_step-breakpoint` — seeds definition-level flag into instance steps
- [ ] `src/zcl_fi_process_instance.clas.abap` (PUBLIC SECTION, after `remove_step`) — ADD public method declaration:
  ```abap
  "! Set or clear breakpoint on a master step of this instance
  "! @parameter iv_step_number | Master step number (substep 000000000000)
  "! @parameter iv_breakpoint  | abap_true = set, abap_false = clear
  "! @parameter iv_no_commit   | If true, skip COMMIT WORK (RAP saver mode)
  "! @raising zcx_fi_process_error | step_not_found if step does not exist
  METHODS set_step_breakpoint
    IMPORTING
      iv_step_number TYPE zfi_process_step_number
      iv_breakpoint  TYPE abap_bool
      iv_no_commit   TYPE abap_bool DEFAULT abap_false
    RAISING
      zcx_fi_process_error.
  ```
- [ ] `src/zcl_fi_process_instance.clas.abap` (IMPLEMENTATION, after `METHOD set_substep_status`) — ADD `METHOD set_step_breakpoint` implementation:
  ```abap
  METHOD set_step_breakpoint.
    READ TABLE mt_steps ASSIGNING FIELD-SYMBOL(<ls_step>)
      WITH KEY step_number    = iv_step_number
               substep_number = '000000000000'.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE zcx_fi_process_error
        EXPORTING
          textid = zcx_fi_process_error=>step_not_found
          value  = |Step { iv_step_number } not found in instance { ms_instance-instance_id }|.
    ENDIF.

    <ls_step>-breakpoint = iv_breakpoint.

    mo_log->message(
      iv_message_class  = 'ZFI_PROCESS'
      iv_message_number = '0XX'           " reserve next available number
      iv_message_v1     = CONV #( iv_step_number )
      iv_message_v2     = CONV #( ms_instance-instance_id )
      iv_message_v3     = CONV #( iv_breakpoint )
      iv_severity       = 'I'
    ).
    mo_log->save( ).

    save_instance( iv_no_commit = iv_no_commit ).
  ENDMETHOD.
  ```
- [ ] `src/zcl_fi_process_instance.clas.abap` (`execute()` breakpoint check ~line 789) — REPLACE current check with two-level check:
  ```abap
  " Breakpoint check: instance-level takes precedence over definition-level
  DATA(ls_step_def) = mo_definition->get_step( ls_step-step_number ).
  IF ls_step-breakpoint = abap_true
  OR ( ls_step_def-breakpoint = abap_true AND ls_step-breakpoint <> abap_true ).
  ```
  Simplify to the clearest readable form:
  ```abap
  IF ls_step-breakpoint = abap_true OR ls_step_def-breakpoint = abap_true.
  ```
  _(instance-level `false` explicitly overrides def-level `true` — requires instance flag to be copied at init and settable to false; because `false` is the initial value, this only works correctly when `initialize_instance` seeds the def breakpoint. If `ls_step-breakpoint` is initial/false and `ls_step_def-breakpoint` is true, the OR fires. If caller explicitly sets `ls_step-breakpoint = false` to suppress a def breakpoint — that suppression is NOT achieved by simple OR. See Design Notes.)_
- [ ] `src/zcl_fi_process_instance.clas.abap` — REMOVE all references to `ms_instance-skip_breakpoints` and `gc_skip_breakpoints` (if any constant was added)
- [ ] `src/zcl_fi_process_manager.clas.abap` (PUBLIC SECTION, after `request_restart_process`) — ADD:
  ```abap
  "! Set or clear breakpoint on a step instance
  "! @parameter iv_instance_id | Process instance ID
  "! @parameter iv_step_number  | Master step number
  "! @parameter iv_breakpoint   | abap_true = set, abap_false = clear
  METHODS set_step_breakpoint_process
    IMPORTING
      iv_instance_id TYPE zfi_process_instance_id
      iv_step_number TYPE zfi_process_step_number
      iv_breakpoint  TYPE abap_bool
    RAISING
      zcx_fi_process_error.
  ```
- [ ] `src/zcl_fi_process_manager.clas.abap` (IMPLEMENTATION) — ADD:
  ```abap
  METHOD set_step_breakpoint_process.
    DATA(lo_instance) = load_process( iv_instance_id ).
    lo_instance->set_step_breakpoint(
      iv_step_number = iv_step_number
      iv_breakpoint  = iv_breakpoint
    ).
  ENDMETHOD.
  ```

**Acceptance Criteria:**
- Given a loaded instance with step 2 in `mt_steps`, when `set_step_breakpoint( '0002', abap_true )` is called, then `ZFI_PROC_STEP.BREAKPOINT = 'X'` for that row and a BAL log entry is written
- Given step 2 has `BREAKPOINT = true` and step definition has `BREAKPOINT = false`, when `execute()` reaches step 2, then the instance halts with status `BREAKPOINT`
- Given step 2 has `BREAKPOINT = false` and step definition has `BREAKPOINT = true`, when `execute()` reaches step 2, then the instance halts with status `BREAKPOINT` (definition fallback fires)
- Given both flags are false on step 2, when `execute()` reaches step 2, then `execute_step()` is called normally (no halt)
- Given `initialize_instance()` is called for a process type where step 3 has `ZFI_PROC_DEF.BREAKPOINT = true`, then `ZFI_PROC_STEP.BREAKPOINT = true` for step 3 in the new instance
- Given a step number that does not exist in `mt_steps`, when `set_step_breakpoint` is called, then `zcx_fi_process_error=>step_not_found` is raised
- Given `SKIP_BREAKPOINTS` field no longer exists on `ZFI_PROC_INST`, then all existing activation units compile without reference to that field

## Spec Change Log

_2026-03-31: Initial spec derived from design session. Clarified that instance-level flag overrides definition-level. SKIP_BREAKPOINTS removed as superseded. Seeding of definition breakpoint into ZFI_PROC_STEP at initialize_instance is required for correct fallback behavior._

## Design Notes

**Breakpoint precedence — "instance overrides definition":**

The user requirement is: instance-level flag takes priority. The cleanest model given `ZFI_PROCESS_BOOLEAN` (initial = false, `abap_true` = set) is:

- At `initialize_instance()`, copy `ZFI_PROC_DEF.BREAKPOINT` → `ZFI_PROC_STEP.BREAKPOINT`. This means the instance step starts with the same value as the definition.
- Caller uses `set_step_breakpoint( step, abap_true )` to **add** a breakpoint that is not in the definition.
- Caller uses `set_step_breakpoint( step, abap_false )` to **suppress** a breakpoint that came from the definition.
- `execute()` only needs to check `ls_step-breakpoint` — no fallback to `ls_step_def` is needed at runtime, because the definition value was already seeded.

This is simpler and more correct than a two-level OR check at runtime. The `execute()` condition simplifies to:

```abap
IF ls_step-breakpoint = abap_true.
  " halt
ENDIF.
```

The `get_step()` call on `mo_definition` inside the execute loop can be removed entirely for the breakpoint check (it may still be needed for other fields). If `get_step()` is only used for the breakpoint check, the call can be dropped — reducing one SELECT/cache-read per step per loop iteration.

**SKIP_BREAKPOINTS removal:**

`ZFI_PROC_INST.SKIP_BREAKPOINTS` and `ZFI_PROCESS_SKIP_BP` DTEL must be removed:
- Remove `DD03P` block from `zfi_proc_inst.tabl.xml`
- Delete `zfi_process_skip_bp.dtel.xml`
- Remove `ms_instance-skip_breakpoints` reference from `execute()` condition
- No other class uses this field (confirmed: only one reference in `zcl_fi_process_instance.clas.abap:792`)

## Verification

**Manual checks:**
- Activate DDIC: `ZFI_PROC_STEP` (new `BREAKPOINT` field) → `ZFI_PROC_INST` (remove `SKIP_BREAKPOINTS`)
- Delete `ZFI_PROCESS_SKIP_BP` DTEL via SE11 after table activation
- Create test instance; verify `ZFI_PROC_STEP.BREAKPOINT` seeded from definition
- Call `set_step_breakpoint( step, abap_true )` — verify DB row updated, BAL log entry present
- Run instance — verify halts at breakpointed step, status = `BREAKPOINT`
- Call `set_step_breakpoint( step, abap_false )` to clear — restart — verify runs through without halt
- Call `set_step_breakpoint` with non-existent step number — verify `step_not_found` exception

## Suggested Review Order

**Schema changes:**
- Add `BREAKPOINT` to `ZFI_PROC_STEP`
  [`zfi_proc_step.tabl.xml`](../../../cz.imcg.fast.planner/src/zfi_proc_step.tabl.xml)
- Remove `SKIP_BREAKPOINTS` from `ZFI_PROC_INST`
  [`zfi_proc_inst.tabl.xml`](../../../cz.imcg.fast.planner/src/zfi_proc_inst.tabl.xml)

**Instance class changes:**
- `initialize_instance()` — seeds DEF breakpoint into step row
  [`zcl_fi_process_instance.clas.abap` ~line 537](../../../cz.imcg.fast.planner/src/zcl_fi_process_instance.clas.abap#L537)
- `execute()` — simplified breakpoint check (step-level only after seeding)
  [`zcl_fi_process_instance.clas.abap` ~line 789](../../../cz.imcg.fast.planner/src/zcl_fi_process_instance.clas.abap#L789)
- New `set_step_breakpoint` method
  [`zcl_fi_process_instance.clas.abap`](../../../cz.imcg.fast.planner/src/zcl_fi_process_instance.clas.abap)

**Manager wrapper:**
- `set_step_breakpoint_process`
  [`zcl_fi_process_manager.clas.abap`](../../../cz.imcg.fast.planner/src/zcl_fi_process_manager.clas.abap)
