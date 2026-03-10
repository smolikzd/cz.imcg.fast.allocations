# LUW Schema Analysis: ZFI_PROCESS Framework

**Date:** 2026-03-09  
**Repository:** `smolikzd/cz.imcg.fast.planner` + `/Users/smolik/DEV/cz.imcg.fast.planner`  
**Scope:** `ZCL_FI_PROCESS_INSTANCE`, `ZCL_FI_PROCESS_MANAGER`, `ZFI_BGRFC_EXEC_SUBSTEP`,  
          all five `ZCL_FI_ALLOC_STEP_*` classes  
**Reviewer:** OpenCode AI Agent

---

## 1. Purpose

This document maps every `COMMIT WORK` in the ZFI_PROCESS framework, defines the LUW
ownership contract between the framework and step implementations, and evaluates the
CC-02 finding ("COMMIT WORK in step bodies") in light of the full commit inventory.

It also documents a newly discovered framework bug (`init` not called in the bgRFC
execution function module) and its impact on all substep-enabled steps.

---

## 2. Full Commit Inventory

### 2.1 `ZCL_FI_PROCESS_INSTANCE`

| # | Method | Statement | Condition | What it persists |
|---|--------|-----------|-----------|-----------------|
| F-01 | `save_instance` | `COMMIT WORK AND WAIT` | Always | `MODIFY zfi_proc_inst` (instance header) + `MODIFY zfi_proc_step FROM TABLE` (all step rows) |
| F-02 | `save_failure_state` | `COMMIT WORK AND WAIT` | Always | `MODIFY zfi_proc_inst` + optional single `MODIFY zfi_proc_step` (failed step row) |
| F-03 | `execute_step` | `COMMIT WORK AND WAIT` | After persisting `result_data` to step header (before substep planning) | Step header `result_data` column |
| F-04 | `execute_step` | `COMMIT WORK AND WAIT` | After inserting newly planned substep rows | `ZFI_PROC_STEP` substep rows |
| F-05 | `process_substeps_queued` | `COMMIT WORK AND WAIT` | After round-robin `queue_id` assignment to all substep rows | `ZFI_PROC_STEP` `queue_id` column for all substeps |
| F-06 | `process_substeps_queued` | `COMMIT WORK AND WAIT` | Safety fallback: substep still has no `queue_id` after round-robin | Single `ZFI_PROC_STEP` row `queue_id` |
| **F-07** | `process_substeps_queued` | **`COMMIT WORK` (no `AND WAIT`)** | One per substep, before/after submitting bgRFC unit to queue | **Required to enqueue the bgRFC/qRFC unit — see §4.3** |
| F-08 | `process_substeps_queued` | `COMMIT WORK AND WAIT` | Polling loop, periodic heartbeat | `MODIFY zfi_proc_inst` (progress counter) |
| F-09 | `update_step_status` | `COMMIT WORK AND WAIT` | Only when `iv_do_commit = abap_true` (default) | `MODIFY zfi_proc_step` (step status row) |
| F-10 | `write_init_bs` | `COMMIT WORK AND WAIT` | Always | `MODIFY zfi_proc_inst` INIT business status — deliberate early commit (see §4.4) |
| F-11 | `add_dynamic_step` | `COMMIT WORK AND WAIT` | Always | `INSERT zfi_proc_step` (new dynamic step row) |
| F-12 | `remove_step` | `COMMIT WORK AND WAIT` | Always | `DELETE zfi_proc_step` |

**Methods with NO commit (caller is responsible):**

| Method | Why no commit |
|--------|--------------|
| `write_final_bs` | Comment: "Caller commits via `save_instance()` / `update_step_status()`" |
| `write_fail_bs` | Comment: "Caller commits via `save_failure_state()`" |

### 2.2 `ZCL_FI_PROCESS_MANAGER`

| # | Method | Statement | What it persists |
|---|--------|-----------|-----------------|
| M-01 | `register_process_type` | `COMMIT WORK AND WAIT` | `MODIFY zfi_proc_type` |
| — | `cleanup_old_instances` | **No commit** | `DELETE` statements are left open — caller must commit or rely on process end |

### 2.3 `ZFI_BGRFC_EXEC_SUBSTEP` (function module)

No `COMMIT WORK` anywhere in the function module body.

The bgRFC/qRFC infrastructure commits the function module's LUW automatically when the
FM returns successfully. This is the qRFC contract: the queue manager owns the commit for
the FM execution LUW.

All `set_substep_status` calls inside the FM explicitly pass `iv_do_commit = abap_false`,
relying entirely on the qRFC commit.

### 2.4 Step classes — `COMMIT WORK` instances (all violations)

| Step | Location | What it commits |
|------|----------|----------------|
| `ZCL_FI_ALLOC_STEP_PHASE1` | After `UPDATE zfi_alloc_state SET phase_1_status = 'R'` | State row with 'R' flag before DELETEs |
| `ZCL_FI_ALLOC_STEP_PHASE1` | After `DELETE FROM zfi_alloc_bche` / `zfi_alloc_bcitm` | Partial DELETEs before INSERT completes |
| `ZCL_FI_ALLOC_STEP_PHASE2` | After `UPDATE zfi_alloc_state SET phase_2_status = 'R'` | State row with 'R' flag before substep planning |
| `ZCL_FI_ALLOC_STEP_CORR_BCHE` | Inside `DO` loop after each chunk `UPDATE zfi_alloc_bche` | Partial chunk update (headers processed so far) |
| `ZCL_FI_ALLOC_STEP_PHASE3` | Every 5 000 inner items inside `LOOP AT lt_unassigned` | Partial `ZFI_ALLOC_ITEMS` rows |

---

## 3. LUW Ownership Model

### 3.1 The correct contract

The ZFI_PROCESS framework owns **all `COMMIT WORK` statements**. This is an explicit
architectural decision, not a convention: every framework commit is for framework-owned
tables (`ZFI_PROC_INST`, `ZFI_PROC_STEP`). Step business data is left open in the LUW
between the step's DML and the framework's next status commit.

```
┌────────────────────────────────────────────────────────┐
│  execute_step() orchestration loop                     │
│                                                        │
│  1. step->init(is_context)                             │
│  2. step->validate()                                   │
│  3. update_step_status(RUNNING)  ──► COMMIT (F-09)    │
│                                                        │
│  4. step->execute()                                    │
│     └─ performs DML on business tables (open LUW)     │
│        ZFI_ALLOC_STATE / ZFI_ALLOC_BCHE / etc.        │
│        NO COMMIT HERE                                  │
│                                                        │
│  5. update_step_status(SUCCESS/FAILURE)  ──► COMMIT   │
│     └─ THIS commit closes the open LUW from step 4    │
│        Business data + status committed atomically     │
└────────────────────────────────────────────────────────┘
```

**Key point:** The framework does NOT have a single "bundle business data + status" commit.
Instead, `update_step_status` commits after each status change. This means a step's
business data is always in the same LUW as the next `update_step_status` call. The step
must not close that window prematurely.

### 3.2 What data belongs to which LUW

| Data category | Tables | LUW owner | Commit source |
|---------------|--------|-----------|---------------|
| Framework instance header | `ZFI_PROC_INST` | Framework | `save_instance`, `save_failure_state`, `write_init_bs`, polling heartbeat |
| Framework step status rows | `ZFI_PROC_STEP` | Framework | `update_step_status`, `execute_step` (substep rows), `add_dynamic_step`, `remove_step` |
| Process type registry | `ZFI_PROC_TYPE` | Framework (manager) | `register_process_type` |
| Allocation state | `ZFI_ALLOC_STATE` | **Step** (DML) + **Framework** (commit) | Framework commits via `update_step_status` after step returns |
| Allocation headers | `ZFI_ALLOC_BCHE` | **Step** (DML) + **Framework** (commit) | Same |
| Allocation header items | `ZFI_ALLOC_BCITM` | **Step** (DML) + **Framework** (commit) | Same |
| Allocation items | `ZFI_ALLOC_ITEMS` | **Step** (DML) + **Framework** (commit) | Same |

### 3.3 Rollback ownership

The framework calls `step->rollback()` when a step fails. At that point, any open DML
from the step's `execute` is still in the LUW (unless the step prematurely committed it).
A properly implemented `rollback` method can issue `ROLLBACK WORK` or compensating DML.

Steps that issued `COMMIT WORK` mid-execute have already closed the LUW. Their business
data is permanently committed and cannot be rolled back. This is the primary danger of
step-level commits — it is not about atomicity with the framework status tables, but about
the irreversibility of partial data states.

---

## 4. Special Commit Cases

### 4.1 `update_step_status` with `iv_do_commit = abap_false`

`update_step_status` accepts `iv_do_commit TYPE abap_bool DEFAULT abap_true`. The framework
deliberately passes `iv_do_commit = abap_false` in the bgRFC function module (`ZFI_BGRFC_EXEC_SUBSTEP`)
for all status updates — the qRFC infrastructure commits the entire FM LUW on return.

This is intentional design: the framework can batch multiple status updates without
intermediate commits when the surrounding infrastructure controls the commit boundary.

Step implementations must never call `update_step_status` directly. They receive the
instance object and call `step->execute()` / `step->execute_substep()` — the framework
layer above calls `update_step_status`.

### 4.2 `cleanup_old_instances` — no commit (framework manager oversight)

`ZCL_FI_PROCESS_MANAGER->cleanup_old_instances` issues `DELETE` statements against
`ZFI_PROC_INST` and `ZFI_PROC_STEP` but performs no `COMMIT WORK`. These deletes remain
open until the caller commits or the process reaches a natural commit boundary.

This is either intentional (caller is expected to commit) or an oversight. In practice,
if `cleanup_old_instances` is called from a test program that does not explicitly commit,
the deletes are rolled back on program end. **This should be verified and documented as
either a design intent or a bug.**

### 4.3 bgRFC qRFC enqueue commit (F-07) — mandatory exception

`process_substeps_queued` issues one `COMMIT WORK` (without `AND WAIT`) per substep
immediately after calling the qRFC enqueue API. This commit is architecturally **required**:
the bgRFC/qRFC subsystem does not enqueue the work unit until the enqueuing LUW is committed.
Without this commit, the substep unit would never enter the queue and would be silently lost.

This commit does **not** close any business data LUW — at that point in the flow, the step
business tables have not been touched yet (substep execution is asynchronous). This commit
is purely an infrastructure commit for the qRFC queue table.

This commit is correct and should not be confused with the CC-02 step-body commits.

### 4.4 `write_init_bs` early commit (F-10) — deliberate visibility commit

`write_init_bs` commits immediately after writing the INIT business status to the instance
header. The comment explains the intent: the INIT business status must be visible in the
process monitor before the first step's `execute` method runs. Without this early commit,
the monitor would show no status until the first step completes.

This is a deliberate UX-driven early commit and is architecturally correct.

---

## 5. bgRFC Execution LUW

When a substep is executed in `QUEUE` mode:

```
bgRFC queue infrastructure
 └── ZFI_BGRFC_EXEC_SUBSTEP (function module)
      ├── CREATE OBJECT lo_step TYPE (class_name)       ← fresh instance
      │                                                  ← init() NOT CALLED (bug — see §6)
      ├── lo_step->execute_substep(lv_substep_number)   ← step business DML (open LUW)
      ├── set_substep_status(iv_do_commit = abap_false) ← status DML (still open LUW)
      └── returns to qRFC infrastructure                ← qRFC commits entire LUW
```

The qRFC infrastructure commits the FM's entire LUW on successful return. This means the
step's business data and the substep status row are committed **atomically** by the
infrastructure — not by the step itself, and not by `update_step_status`.

This is the correct pattern for bgRFC substep execution. The FM must not issue its own
`COMMIT WORK` and must not pass `iv_do_commit = abap_true` to `set_substep_status`.

If the FM raises an exception or returns with an error, the qRFC infrastructure rolls back
the LUW and retries according to the queue configuration.

---

## 6. Framework Bug: `init` Not Called in `ZFI_BGRFC_EXEC_SUBSTEP`

### Description

`ZFI_BGRFC_EXEC_SUBSTEP` contains (simplified):

```abap
CREATE OBJECT lo_step
  TYPE (lv_class_name)
  EXPORTING
    iv_process_instance_id = lv_instance_id
    iv_step_number         = lv_step_number.

lo_step->execute_substep( lv_substep_number ).   " ← init() never called
```

The `init` method — which reads all process parameters into instance variables
(`mv_allocation_id`, `mv_company_code`, `mv_fiscal_year`, `mv_fiscal_period`,
`mv_force_start`) and initializes `mo_log` — is **never called** before `execute_substep`.

### Impact by step

| Step | Impact in `execute_substep` |
|------|---------------------------|
| `ZCL_FI_ALLOC_STEP_PHASE2` | `mv_allocation_id`, `mv_company_code`, `mv_fiscal_year`, `mv_fiscal_period` are all initial (`''`/`0`). `mo_log` is `NULL`. Any `mo_log->…` call in `execute_substep` → `CX_SY_REF_IS_INITIAL` dump. Any SELECT/DML using these variables operates on initial key values (full table scans or zero-row results). |

### Severity: Critical (framework bug)

This bug makes the **only currently functional queued step** (`ZCL_FI_ALLOC_STEP_PHASE2`,
configured as `substep_mode = 'QUEUE'`) completely non-functional in its queued execution
path. Every bgRFC substep invocation will either dump immediately (on `mo_log` access) or
silently process zero data (if `mo_log` access is skipped and the DML uses initial key values).

### Affected framework file

`src/zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap`

### Required fix

```abap
CREATE OBJECT lo_step
  TYPE (lv_class_name)
  EXPORTING
    iv_process_instance_id = lv_instance_id
    iv_step_number         = lv_step_number.

" FIX: build context and call init before execute_substep
DATA(ls_context) = VALUE zif_fi_process_step=>ts_context(
  io_process_instance = lo_instance ).
lo_step->init( ls_context ).                         " ← add this

lo_step->execute_substep( lv_substep_number ).
```

Where `lo_instance` is the `ZCL_FI_PROCESS_INSTANCE` object, which the FM must retrieve
(or construct) using `lv_instance_id` before the step creation. The exact structure of
`ts_context` must be verified against `ZIF_FI_PROCESS_STEP` to ensure all required fields
are populated.

### Impact on Phase 2 `mo_log` finding

The Phase 2 migration review (FINDING-08) noted that `mo_log` is initialized in `execute`
and not in `init`, making it NULL in `execute_substep`. Even if `mo_log` initialization
were moved to `init` (the recommended fix in the Phase 2 review), it would **still be NULL
in the bgRFC execution path** because `init` is never called by the FM. Both fixes are
required independently:
1. Move `mo_log` initialization to `init` (step fix).
2. Call `init` in `ZFI_BGRFC_EXEC_SUBSTEP` (framework fix).

---

## 7. Reassessment of CC-02 (COMMIT WORK in Step Bodies)

### 7.1 Original finding

CC-02 characterised all step-level `COMMIT WORK` statements as violations of the
"framework owns the LUW" contract. The finding is confirmed — all step commits are
violations. The analysis below provides a more precise characterisation of the consequences.

### 7.2 What the framework's own commits mean for atomicity

The framework does **not** provide a single atomic commit that bundles business data with
the step status row. `update_step_status` commits only the status row; business data was
already committed by a previous step call or will be committed by the next. Each
`update_step_status` call creates its own commit boundary.

This means the framework itself does not guarantee atomicity between business data and step
status. The LUW contract is about **commit ownership** (who issues `COMMIT WORK`) rather
than full atomicity.

### 7.3 Consequence of step-level commits: rollback window destruction

| Scenario | Without step `COMMIT WORK` | With step `COMMIT WORK` |
|----------|---------------------------|------------------------|
| Step fails after partial DML | Framework calls `rollback()` — step can issue `ROLLBACK WORK` or compensating DML | Business data already committed. `rollback()` cannot undo it. Permanent partial state. |
| Step is cancelled mid-execute | Same — rollback is possible | Same — rollback is impossible |
| Work process timeout | ABAP runtime issues `ROLLBACK WORK` automatically for open changes | Already-committed data survives. Only uncommitted changes are rolled back. Inconsistent split. |
| Normal success | No difference | No difference |

The consequences are most severe in the chunk-commit patterns (CORR_BCHE, PHASE3):
a step interrupted mid-loop leaves a partial dataset permanently committed with no reliable
way to determine where processing stopped.

### 7.4 Per-step impact assessment

| Step | Commit pattern | Consequence severity | Notes |
|------|---------------|---------------------|-------|
| PHASE1 | 2× mid-execute | **High** | DELETEs committed before INSERTs complete. If INSERT fails, old data is gone and new data is partial. |
| PHASE2 | 1× after state 'R' | **Medium** | Commits state before substep planning. If substep INSERT fails, state shows 'R' with no substeps. Substep re-creation on retry may produce duplicates. |
| CORR_BCHE | 1× per chunk inside `DO` loop | **High** | Chunks committed independently. Retry starts from beginning but some headers already flagged. Idempotent only if flag logic is idempotent (it is for `no_items`, but not for partial chunk boundaries). |
| PHASE3 | 1× per 5 000 items inside `LOOP` | **High** | Partial `ZFI_ALLOC_ITEMS` committed. Re-run without `FORCE_START` will start from 'R' state (step guard blocks re-entry). Requires manual state reset. |

### 7.5 Verdict

CC-02 stands as a **Critical** finding across all affected steps. The severity is confirmed
and not mitigated by the framework's own commit pattern. The recommended fix remains:

1. Remove all `COMMIT WORK` from step bodies.
2. Implement meaningful `rollback` methods (at minimum: reset `phase_X_status` to initial).
3. For PHASE3: design the inner loop as substeps (`plan_substeps` + `execute_substep`)
   so each substep's commit boundary is the bgRFC infrastructure commit, not a manual
   `COMMIT WORK` inside a loop.

---

## 8. Interaction Map: Framework Commits vs. Step Business Data

The following timeline shows one complete step execution (non-substep path) and identifies
every commit boundary:

```
Time →

[FRAMEWORK: save_instance() called at process start]
  MODIFY zfi_proc_inst (status=RUNNING)
  MODIFY zfi_proc_step FROM TABLE (all steps PENDING)
  COMMIT WORK AND WAIT                          ← F-01: Process start commit

[FRAMEWORK: write_init_bs()]
  MODIFY zfi_proc_inst (init business status)
  COMMIT WORK AND WAIT                          ← F-10: Early visibility commit

[FRAMEWORK: execute_step() — begin step N]
  [FRAMEWORK: update_step_status(RUNNING, iv_do_commit=true)]
    MODIFY zfi_proc_step (step N status=RUNNING)
    COMMIT WORK AND WAIT                        ← F-09: Step enters running state

  [STEP: execute()]
    DML on zfi_alloc_state, zfi_alloc_bche, ...   ← open LUW (no commit here — correct)
    return rs_result

  [FRAMEWORK: update_step_status(SUCCESS, iv_do_commit=true)]
    MODIFY zfi_proc_step (step N status=SUCCESS)
    COMMIT WORK AND WAIT                        ← F-09: Step status + all step DML committed

[FRAMEWORK: execute_step() — begin step N+1]
  ... (same pattern)
```

**Observation:** Between F-09 (RUNNING commit) and F-09 (SUCCESS commit), the step's entire
`execute` body runs in a single open LUW. This is the correct window. Any `COMMIT WORK`
inside `execute` splits this window, making partial data permanent before the step completes.

---

## 9. Summary of Findings

| ID | Finding | Severity | Owner | File |
|----|---------|----------|-------|------|
| LUW-01 | All step-level `COMMIT WORK` statements violate the framework LUW contract and destroy rollback capability | Critical | Step classes | See CC-02 |
| LUW-02 | `init` never called in `ZFI_BGRFC_EXEC_SUBSTEP` — all instance variables and `mo_log` are initial in queued substep execution | Critical | Framework | `zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap` |
| LUW-03 | `cleanup_old_instances` leaves DELETEs uncommitted — caller commit dependency undocumented | Low | Framework | `zcl_fi_process_manager.clas.abap` |
| LUW-04 | bgRFC qRFC enqueue commit (F-07, `COMMIT WORK` without `AND WAIT`) is correct and required — not a violation | N/A (informational) | Framework | `zcl_fi_process_instance.clas.abap` |
| LUW-05 | `write_init_bs` early commit (F-10) is a deliberate visibility commit — not a violation | N/A (informational) | Framework | `zcl_fi_process_instance.clas.abap` |
| LUW-06 | Phase 2 `mo_log` NULL in bgRFC path requires two independent fixes: move init to `init()` (step) AND call `init()` from FM (framework) | Critical | Step + Framework | `zcl_fi_alloc_step_phase2.clas.abap` + FM |

---

## 10. Required Actions

### Framework fixes (owner: ZFI_PROCESS framework team)

| # | Action | Finding |
|---|--------|---------|
| F-FIX-01 | Add `lo_step->init( ls_context )` call in `ZFI_BGRFC_EXEC_SUBSTEP` before `execute_substep` | LUW-02 |
| F-FIX-02 | Clarify and document intent of `cleanup_old_instances` missing commit — add `COMMIT WORK AND WAIT` if unintentional | LUW-03 |

### Step fixes (owner: ZFI_ALLOC_PROCESS development team)

| # | Action | Finding | Also see |
|---|--------|---------|---------|
| S-FIX-01 | Remove all `COMMIT WORK` from PHASE1, PHASE2, CORR_BCHE, PHASE3 | LUW-01 | CC-02 |
| S-FIX-02 | Implement real `rollback` methods in all steps: reset `phase_X_status` + delete partial data | LUW-01 | CC-05 |
| S-FIX-03 | Move `mo_log` initialization from `execute` to `init` in PHASE2 | LUW-06 | Phase 2 FINDING-08 |
| S-FIX-04 | Redesign PHASE3 inner loop as `plan_substeps` + `execute_substep` to avoid chunk-commit pattern entirely | LUW-01 | Phase 3 FINDING-04 |
