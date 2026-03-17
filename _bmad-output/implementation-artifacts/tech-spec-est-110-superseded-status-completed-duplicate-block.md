---
title: 'SUPERSEDED status and COMPLETED duplicate block (EST-110)'
slug: 'est-110-superseded-status-completed-duplicate-block'
created: '2026-03-16'
status: 'completed'
completed_date: '2026-03-17'
stepsCompleted: [1, 2, 3, 4, 5, 6]
review_notes: |
  Adversarial review completed 2026-03-17. 12 findings evaluated:
  5 classified as NOISE (F1, F2, F5, F6, F7), 7 fixed automatically (F3, F4, F8, F9, F10, F11, F12).
  Key fixes: blank line in cancel(), gc_status constant ordering in manager class aligned
  to instance class order, supersede() doc comment clarifies ended_at preservation,
  test_allows_when_superseded variable name normalised, test_double_supersede added as
  negative test for idempotency, lt_cleanup_statuses merged into single DATA block,
  SKIPPED DD07V indentation in domain XML restored to match entries 0001-0006.
tech_stack: ['ABAP 7.58', 'SAP DDIC', 'ZFI_PROCESS framework', 'ABAP Unit']
files_to_modify:
  - 'src/zfi_process_status.doma.xml'
  - 'src/zcx_fi_process_error.clas.abap'
  - 'src/zfi_process.msag.xml'
  - 'src/zcl_fi_process_instance.clas.abap'
  - 'src/zcl_fi_process_manager.clas.abap'
  - 'src/zcl_fi_process_manager.clas.testclasses.abap'
  - 'src/zcl_fiproc_health_chk_query.clas.abap'
files_to_create: []
code_patterns:
  - 'gc_status constants: CONSTANTS BEGIN OF gc_status ... END OF gc_status — append new line before END OF'
  - 'supersede() similar to cancel() but NO GET TIME STAMP: guard on status (must be COMPLETED), set status to SUPERSEDED, save_instance() — ended_at is preserved from original completion'
  - 'cancel() allowlist: IF NOT ( status = running OR status = failed ). RAISE invalid_status.'
  - 'execute() CASE: add WHEN gc_status-superseded block matching WHEN gc_status-cancelled style'
  - 'supersede_process() in manager: load_process( id ) then lo_instance->supersede()'
  - 'Duplicate check extension: add OR line_exists( lt_existing[ status = gc_status-completed ] )'
  - 'cleanup_old_instances: Build lt_cleanup_statuses RANGE table with CANCELLED+SUPERSEDED, then WHERE status IN @lt_cleanup_statuses in both SELECT and DELETE'
  - 'Exception text ID pattern: CONSTANTS BEGIN OF ... msgid ZFI_PROCESS msgno NNN END OF'
  - 'Message XML pattern: <T100><SPRSL>E</SPRSL><ARBGB>ZFI_PROCESS</ARBGB><MSGNR>NNN</MSGNR><TEXT>...</TEXT></T100>'
test_patterns:
  - 'ltcl_duplicate_check_test: FOR TESTING DURATION SHORT RISK LEVEL HARMLESS, Given/When/Then, set_instance_status() helper'
  - 'check_duplicate health check: UPDATE zfi_proc_inst SET status, COMMIT, then call create_process()/supersede_process()'
  - 'New unit tests follow exact pattern of existing 5 methods in ltcl_duplicate_check_test'
---

# Tech-Spec: SUPERSEDED status and COMPLETED duplicate block (EST-110)

**Created:** 2026-03-16
**Linear:** EST-110
**Target Repository:** `cz.imcg.fast.planner`

## Overview

### Problem Statement

EST-102 introduced a duplicate check that blocks `create_process()` when a `RUNNING` or `FAILED` instance with the same parameter hash exists. However, COMPLETED instances are currently allowed through — meaning a process type that successfully ran can immediately be re-submitted with the same parameters, with no lifecycle separation between the old run and the new one.

Additionally, there is no way to mark a COMPLETED instance as "superseded by a newer run", leaving monitoring UIs unable to distinguish active-current runs from historical ones.

### Solution

1. **Block COMPLETED in duplicate check**: Extend `create_process()` to also block when a COMPLETED instance exists with the same hash. This forces the caller to explicitly supersede the old instance before creating a new one — making the lifecycle transition explicit and auditable.

2. **Introduce SUPERSEDED status**: A new terminal status `SUPERSEDED` is added to the `ZFI_PROCESS_STATUS` domain. The `supersede()` method on `ZCL_FI_PROCESS_INSTANCE` transitions a COMPLETED instance to SUPERSEDED (only callable from COMPLETED; terminal — non-executable, non-cancellable). `ZCL_FI_PROCESS_MANAGER` exposes `supersede_process( iv_instance_id )` as the standard entry point.

3. **Adjust `cancel()` guard**: Replace the current single-status guard (`blocks only COMPLETED`) with an allowlist: only `RUNNING` or `FAILED` instances may be cancelled.

4. **Update `cleanup_old_instances()`**: COMPLETED instances are now permanent records (lifecycle: COMPLETED → supersede → SUPERSEDED → auto-cleanup). Change cleanup to target `CANCELLED` and `SUPERSEDED` only; COMPLETED is no longer removed.

### Status lifecycle after EST-110

| Status | Duplicate blocks `create_process()`? | Can cancel? | Can execute? | Can supersede? | Cleaned by `cleanup_old_instances()`? |
|--------|--------------------------------------|-------------|--------------|----------------|---------------------------------------|
| NEW | — | No | Yes | No | No |
| RUNNING | Yes — blocks | Yes | No (already running) | No | No |
| FAILED | Yes — blocks | Yes | No (use restart) | No | No |
| COMPLETED | **Yes — blocks (NEW)** | No | No | **Yes (NEW)** | **No (NEW — permanent)** |
| CANCELLED | No — allows | No | No | No | **Yes** |
| SUPERSEDED | **No — allows (NEW)** | No | **No (NEW)** | No | **Yes (NEW)** |

> **Note — PENDING and QUEUED:** These statuses are not listed in the table because EST-110 makes no changes to their behaviour. PENDING and QUEUED are pre-execution statuses that do not appear in the duplicate check's status filter (neither blocks `create_process()`), cannot be cancelled (they fall through to `invalid_status` in the new allowlist guard), and are not targeted by `cleanup_old_instances()`. They continue to behave exactly as before this change.

### Scope

**In Scope:**
- New DDIC fixed value `SUPERSEDED` on `ZFI_PROCESS_STATUS` domain (VALPOS 0008)
- New exception text ID `instance_superseded` in `ZCX_FI_PROCESS_ERROR` (msgno 018)
- New T100 message 018 in `ZFI_PROCESS` message class
- `gc_status-superseded` constant added to both `ZCL_FI_PROCESS_INSTANCE` and `ZCL_FI_PROCESS_MANAGER`
- `supersede()` public instance method on `ZCL_FI_PROCESS_INSTANCE`
- `supersede_process()` public method on `ZCL_FI_PROCESS_MANAGER`
- `cancel()` guard replaced with allowlist (only RUNNING or FAILED)
- `execute()` guard extended with SUPERSEDED blocking case
- `create_process()` duplicate check extended: also blocks COMPLETED
- `cleanup_old_instances()` retargeted to CANCELLED + SUPERSEDED (COMPLETED removed)
- Unit tests: 2 new test methods in `ltcl_duplicate_check_test`
- Health check `check_duplicate`: extended with sub-scenarios 3 (COMPLETED blocks) + 4 (SUPERSEDED allows)

**Out of Scope:**
- UI/monitoring changes
- CDS view changes
- `restart_process()` duplicate check
- Translation of new message to Czech (deferred per Sprint policy)
- Translation of SUPERSEDED domain fixed value description to Czech (also deferred — domain text translation follows the same sprint deferral policy as T100 message translations)
- Blocking PENDING/QUEUED statuses from supersede
- Retention period configuration separate from existing `iv_days_to_keep`

---

## Context for Development

### Technical Preferences & Constraints

- **Constitution Principle I (DDIC-First):** New status value must be added to `ZFI_PROCESS_STATUS` DDIC domain — no hardcoded string comparisons
- **Constitution Principle II (SAP Standards):** Line length ≤ 120 chars; SAP naming conventions
- **Constitution Principle V (Error Handling):** Raise `zcx_fi_process_error` with proper `textid` + `value` context string
- `supersede()` is terminal: once SUPERSEDED, the instance cannot be cancelled, re-executed, or superseded again
- `cleanup_old_instances()` performs `COMMIT WORK AND WAIT` internally — callers do not commit
- The health check `check_duplicate` must neutralize the RUNNING seed from sub-scenario 1 (set to CANCELLED) before testing sub-scenario 3 (COMPLETED blocks), to prevent the RUNNING instance masking the COMPLETED block result

### Codebase Patterns

- **`gc_status` constants (instance):** `src/zcl_fi_process_instance.clas.abap` lines 26–35 — `CONSTANTS: BEGIN OF gc_status, ... END OF gc_status.` in PUBLIC SECTION. Add `superseded TYPE zfi_process_status VALUE 'SUPERSEDED',` before `skipped` or after `cancelled`.
- **`gc_status` constants (manager):** `src/zcl_fi_process_manager.clas.abap` lines 170–178 — PRIVATE SECTION, same pattern. Add `superseded TYPE zfi_process_status VALUE 'SUPERSEDED',` before `END OF gc_status`.
  > **Note (pre-existing gap):** The manager `gc_status` block does NOT currently include a `queued` constant (unlike the instance class). `QUEUED` is referenced in the lifecycle table and cancel-guard notes as a non-cancellable status, but no manager-level constant exists for it. Task 6a adds only `superseded`. If future manager logic needs to check `QUEUED` directly, a separate story should add that constant.
- **`cancel()` current guard (instance, line 1821–1827):**
  ```abap
  METHOD cancel.
    IF ms_instance-status = gc_status-completed.
      RAISE EXCEPTION TYPE zcx_fi_process_error
        EXPORTING
          textid = zcx_fi_process_error=>invalid_status
          value  = |Cannot cancel completed instance|.
    ENDIF.
    ms_instance-status = gc_status-cancelled.
    GET TIME STAMP FIELD ms_instance-ended_at.
    save_instance( ).
  ENDMETHOD.
  ```
- **`execute()` CASE guard (instance, lines 625–661):** CASE block with WHEN RUNNING / FAILED / COMPLETED / CANCELLED / OTHERS. Add `WHEN gc_status-superseded.` block before `WHEN OTHERS.`, matching the `WHEN gc_status-cancelled.` pattern exactly.
- **`cleanup_old_instances()` SELECT/DELETE (manager, lines 443–457):** Both statements reference `gc_status-completed` — both must change to use a RANGE table. In ABAP 7.58 Open SQL, `IN ( @var1, @var2 )` with two separate host variables is **not valid syntax**. Use a range table instead:
  ```abap
  DATA lt_cleanup_statuses TYPE RANGE OF zfi_process_status.
  lt_cleanup_statuses = VALUE #(
    ( sign = 'I' option = 'EQ' low = gc_status-cancelled   )
    ( sign = 'I' option = 'EQ' low = gc_status-superseded  ) ).
  ```
  Then both statements use `WHERE status IN @lt_cleanup_statuses AND ended_at < @lv_cutoff_ts`.
- **Duplicate check block (manager, lines 286–293):** Current: `IF line_exists( lt_existing[ status = gc_status-running ] ) OR line_exists( lt_existing[ status = gc_status-failed ] ).` — add a third `OR line_exists( lt_existing[ status = gc_status-completed ] )`.
- **`supersede_process()` manager method:** One-liner pattern identical to `cancel_process()` (lines 401–404):
  ```abap
  METHOD supersede_process.
    DATA(lo_instance) = load_process( iv_instance_id ).
    lo_instance->supersede( ).
  ENDMETHOD.
  ```
- **Exception text ID pattern:** After `parallel_limit_exceeded` block (line 184), before `bgrfc_destination_not_found` (line 186):
  ```abap
  CONSTANTS:
    BEGIN OF instance_superseded,
      msgid TYPE symsgid VALUE 'ZFI_PROCESS',
      msgno TYPE symsgno VALUE '018',
      attr1 TYPE scx_attrname VALUE 'VALUE',
      attr2 TYPE scx_attrname VALUE '',
      attr3 TYPE scx_attrname VALUE '',
      attr4 TYPE scx_attrname VALUE '',
    END OF instance_superseded.
  ```
- **Message XML pattern (msag.xml):** Insert after msg 017 block (line 82), before msg 020 block (line 83):
  ```xml
  <T100>
   <SPRSL>E</SPRSL>
   <ARBGB>ZFI_PROCESS</ARBGB>
   <MSGNR>018</MSGNR>
   <TEXT>Instance &amp;1 cannot be superseded (invalid status)</TEXT>
  </T100>
  ```
- **Health check sub-scenario isolation (check_duplicate):** After sub-scenario 1 fires successfully, set the seed instance to CANCELLED (to remove it from the RUNNING pool) before seeding a COMPLETED instance for sub-scenario 3.

### Files to Reference

| File | Purpose |
| ---- | ------- |
| `src/zfi_process_status.doma.xml` | Add SUPERSEDED at VALPOS 0008 |
| `src/zcx_fi_process_error.clas.abap` | Add `instance_superseded` CONSTANTS block (msgno 018) |
| `src/zfi_process.msag.xml` | Add message 018 text |
| `src/zcl_fi_process_instance.clas.abap` | Add `gc_status-superseded`; add `supersede()` method; update `cancel()` guard; update `execute()` CASE. Note: `get_instance()` is declared at line 130 and returns `ms_instance` (type `ty_process_inst`) — used in health check seed code. `get_instance_id()` is declared at line 94 and returns the instance ID directly — used in test class `set_instance_status()` calls. |
| `src/zcl_fi_process_manager.clas.abap` | Add `gc_status-superseded`; add `supersede_process()`; extend duplicate check; update cleanup |
| `src/zcl_fi_process_manager.clas.testclasses.abap` | Add `test_blocks_when_completed` and `test_allows_when_superseded` |
| `src/zcl_fiproc_health_chk_query.clas.abap` | Extend `check_duplicate` with sub-scenarios 3+4 |
| `src/zcl_fi_process_instance.clas.abap` (cancel/execute) | Reference — current guard implementations |
| `src/zcl_fi_process_manager.clas.abap` (cleanup/duplicate) | Reference — current implementations |

### Technical Decisions

1. **SUPERSEDED is terminal:** `supersede()` may only be called from COMPLETED status. Once SUPERSEDED, the instance cannot transition to any other status (not executable, not cancellable, not supersedeable again). Raises `instance_superseded` (018) if called from any other status.

2. **`cancel()` allowlist strictly RUNNING + FAILED:** NEW, PENDING, QUEUED, COMPLETED, SUPERSEDED, SKIPPED statuses cannot be cancelled. The guard becomes a positive allowlist: `IF NOT ( status = running OR status = failed ). RAISE invalid_status.`

3. **COMPLETED blocks duplicate check:** An instance in COMPLETED status blocks new creation with the same hash — the caller must call `supersede_process()` first to explicitly acknowledge the old run before starting a new one.

4. **SUPERSEDED allows duplicate check:** A SUPERSEDED instance is effectively "retired" — it no longer blocks new creation. This is the primary use case: supersede → create.

5. **cleanup_old_instances() retargeted:** COMPLETED instances are now permanent audit records. `cleanup_old_instances()` removes CANCELLED and SUPERSEDED only. The `iv_days_to_keep` cutoff applies identically to both statuses.

6. **Message 018 format:** `Instance &1 cannot be superseded (invalid status)` — single-attr pattern: attr1 = VALUE, attr2–attr4 = ''. The `value` string passed at RAISE time contains the full context (instance ID + current status). This is consistent with all other exception messages in `ZCX_FI_PROCESS_ERROR` which use a single `&1` placeholder mapped to the VALUE attribute.

7. **No new `ended_at` stamp for SUPERSEDED:** `supersede()` does NOT set `ended_at` — that timestamp was set when the instance originally completed. Preserving it maintains the original completion time as the authoritative end-of-execution timestamp.

8. **`execute()` CASE: explicit SUPERSEDED block, PENDING/QUEUED via WHEN OTHERS:** The `execute()` CASE block has no explicit `WHEN` for PENDING or QUEUED statuses — they fall through to `WHEN OTHERS`. Adding `WHEN gc_status-superseded` explicitly (Task 5) is intentional: SUPERSEDED is a new status introduced in this ticket and needs a clear, named error message. PENDING and QUEUED already receive a correct rejection response via `WHEN OTHERS`. Adding explicit WHEN branches for them is separate cleanup work, out of scope for EST-110.

---

## Implementation Plan

### Tasks

Tasks are ordered by dependency (DDIC first, then exception, then instance changes, then manager, then tests).

- [x] **Task 1: Add SUPERSEDED fixed value to `ZFI_PROCESS_STATUS` domain**
  - File: `src/zfi_process_status.doma.xml` *(modify)*
  - Action: Append a new `<DD07V>` entry inside `<DD07V_TAB>` after the SKIPPED entry (after line 56):
    ```xml
    <DD07V>
     <VALPOS>0008</VALPOS>
     <DDLANGUAGE>E</DDLANGUAGE>
     <DOMVALUE_L>SUPERSEDED</DOMVALUE_L>
     <DDTEXT>Superseded</DDTEXT>
    </DD07V>
    ```
  - Notes: Follows the exact pattern of all 7 existing entries. VALPOS 0008 is the next available slot.
    > **Pre-condition check:** Before inserting, open `src/zfi_process_status.doma.xml` and inspect the existing `<VALPOS>` values in all `<DD07V>` entries. VALPOS 0008 is assumed to be the next free slot (7 existing entries at VALPOS 0001–0007). If the actual file shows a different highest VALPOS (e.g. a previously added entry at 0008), use the correct next free slot and update the `<VALPOS>` value accordingly.

- [x] **Task 2: Add `instance_superseded` exception text ID and message 018**
  - File A: `src/zcx_fi_process_error.clas.abap` *(modify)*
  - Action: Insert new CONSTANTS block after `parallel_limit_exceeded` (after line 184), before `bgrfc_destination_not_found`:
    ```abap
    CONSTANTS:
      BEGIN OF instance_superseded,
        msgid TYPE symsgid VALUE 'ZFI_PROCESS',
        msgno TYPE symsgno VALUE '018',
        attr1 TYPE scx_attrname VALUE 'VALUE',
        attr2 TYPE scx_attrname VALUE '',
        attr3 TYPE scx_attrname VALUE '',
        attr4 TYPE scx_attrname VALUE '',
      END OF instance_superseded.
    ```
  - File B: `src/zfi_process.msag.xml` *(modify)*
  - Action: Insert new `<T100>` block after msg 017 (after line 82), before msg 020:
    ```xml
    <T100>
     <SPRSL>E</SPRSL>
     <ARBGB>ZFI_PROCESS</ARBGB>
     <MSGNR>018</MSGNR>
     <TEXT>Instance &amp;1 cannot be superseded (invalid status)</TEXT>
    </T100>
    ```
  - Notes: Message 018 sits between exception msgs (016–017) and runtime msgs (020+). No 011–015 gap collision. Single `&1` placeholder — attr1 maps to VALUE which contains the full context string (instance ID + current status). This is consistent with the single-attr pattern used by all other exception messages in this class.
    > **Pre-condition check:** Before inserting msgno 018, open `src/zfi_process.msag.xml` and search for any existing `<MSGNR>018</MSGNR>` entry. If 018 is already occupied by another message, use the next free message number instead and update the `msgno` value in both the CONSTANTS block (File A) and the XML entry (File B) accordingly.

- [x] **Task 3: Add `gc_status-superseded` constant and `supersede()` method to `ZCL_FI_PROCESS_INSTANCE`**
  - File: `src/zcl_fi_process_instance.clas.abap` *(modify)*

  > **Line number note:** Line numbers below reflect the file before any edits in this task set. After Action A adds a line to the PUBLIC SECTION constant block, and Action B adds ~8 lines for the method declaration, all subsequent line references in Tasks 4 and 5 shift accordingly. Use the code patterns to locate insertion points — do not rely solely on line numbers.

  - Action A (PUBLIC SECTION — gc_status constant, line 34): Add `superseded` before `skipped`:
    ```abap
                 superseded TYPE zfi_process_status VALUE 'SUPERSEDED',
    ```
  - Action B (PUBLIC SECTION — method declaration, after `cancel` at line 91): Add:
    ```abap
    "! Supersede a completed process instance
    "! <p>Marks this instance as superseded by a newer run.
    "! Only callable from COMPLETED status. Terminal — the instance cannot be
    "! re-executed, cancelled, or superseded again after this call.</p>
    "! @raising zcx_fi_process_error | Raised if instance is not in COMPLETED status
    METHODS supersede
      RAISING
        zcx_fi_process_error.
    ```
  - Action C (IMPLEMENTATION — new METHOD block, near `cancel` implementation around line 1832): Add after the `cancel` ENDMETHOD:
    ```abap
    METHOD supersede.
      IF ms_instance-status <> gc_status-completed.
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>instance_superseded
            value  = |Instance { ms_instance-instance_id } cannot be superseded: |
                   && |current status is { ms_instance-status }. Expected: COMPLETED.|.
      ENDIF.
      ms_instance-status = gc_status-superseded.
      save_instance( ).
    ENDMETHOD.
    ```
  - Notes: No `ended_at` update — original completion timestamp is preserved. `save_instance()` persists the status change.

- [x] **Task 4: Update `cancel()` guard to allowlist in `ZCL_FI_PROCESS_INSTANCE`**
  - File: `src/zcl_fi_process_instance.clas.abap` *(modify)*

  > **Line number note:** Line numbers below reflect the file before any edits in this task set. After Task 3 adds ~1 line to the constant block and ~8 lines for the method declaration, the `cancel()` implementation will have shifted by ~9 lines. Use the code pattern (`METHOD cancel.` followed by the COMPLETED-only guard) to locate the correct block — do not rely solely on line numbers.

  - Action: Replace the current `cancel()` guard (lines 1821–1831) with the allowlist pattern:
    ```abap
    METHOD cancel.
      IF NOT ( ms_instance-status = gc_status-running OR
               ms_instance-status = gc_status-failed ).
        RAISE EXCEPTION TYPE zcx_fi_process_error
          EXPORTING
            textid = zcx_fi_process_error=>invalid_status
            value  = |Cannot cancel instance { ms_instance-instance_id }: |
                   && |status { ms_instance-status } is not cancellable. |
                   && |Only RUNNING or FAILED instances may be cancelled.|.
      ENDIF.
      ms_instance-status = gc_status-cancelled.
      GET TIME STAMP FIELD ms_instance-ended_at.
      save_instance( ).
    ENDMETHOD.
    ```
  - Notes: Previous guard only blocked COMPLETED. New guard is a positive allowlist: RUNNING and FAILED are the only cancellable statuses. All other statuses (NEW, PENDING, QUEUED, COMPLETED, SUPERSEDED, SKIPPED, CANCELLED) raise `invalid_status`.
    > **Breaking behaviour change:** Previously, calling `cancel()` on a NEW instance did NOT raise an exception (only COMPLETED was blocked). After this change, NEW instances also raise `invalid_status`. This is intentional — see Technical Decision 2 and the Notes section. No dedicated unit test is prescribed for this in the current sprint; developers should manually verify that `cancel()` on a NEW instance raises `invalid_status` post-implementation.

- [x] **Task 5: Add SUPERSEDED blocking case to `execute()` CASE in `ZCL_FI_PROCESS_INSTANCE`**
  - File: `src/zcl_fi_process_instance.clas.abap` *(modify)*

  > **Line number note:** Line numbers below reflect the file before any edits in this task set. After Tasks 3 and 4 insert lines above the `execute()` method body, the CASE block will have shifted. Use the code pattern (`WHEN gc_status-cancelled.` block near the end of the CASE statement in `METHOD execute.`) to locate the insertion point — do not rely solely on line numbers.

  - Action: In the `execute()` CASE block (around line 648), add `WHEN gc_status-superseded.` before `WHEN OTHERS.`:
    ```abap
        WHEN gc_status-superseded.
          RAISE EXCEPTION TYPE zcx_fi_process_error
            EXPORTING
              textid = zcx_fi_process_error=>invalid_status
              value  = |Instance { ms_instance-instance_id } was SUPERSEDED. | &&
                       |Cannot execute superseded instances.|.
    ```
  - Notes: Pattern matches `WHEN gc_status-cancelled.` exactly (same indentation, same structure, same message style).

- [x] **Task 6: Update `ZCL_FI_PROCESS_MANAGER` — constant, supersede_process(), duplicate check, cleanup**
  - File: `src/zcl_fi_process_manager.clas.abap` *(modify)*

  - **6a — Add `gc_status-superseded` to PRIVATE SECTION constant (line 177):**
    ```abap
                 superseded TYPE zfi_process_status VALUE 'SUPERSEDED',
    ```
    Insert before `END OF gc_status.`

  - **6b — Add `supersede_process()` PUBLIC method declaration (after `cancel_process` at line 117):**
    ```abap
    "! Supersede completed process instance
    "! @parameter iv_instance_id | Instance ID
    METHODS supersede_process
      IMPORTING
        iv_instance_id TYPE zfi_process_instance_id
      RAISING
        zcx_fi_process_error.
    ```

  - **6c — Add `supersede_process()` implementation (after `cancel_process` METHOD block at line 404):**
    ```abap
    METHOD supersede_process.
      DATA(lo_instance) = load_process( iv_instance_id ).
      lo_instance->supersede( ).
    ENDMETHOD.
    ```

  - **6d — Extend duplicate check to block COMPLETED (lines 286–287):**
    Replace:
    ```abap
       IF line_exists( lt_existing[ status = gc_status-running ] ) OR
          line_exists( lt_existing[ status = gc_status-failed  ] ).
    ```
    With:
    ```abap
       IF line_exists( lt_existing[ status = gc_status-running   ] ) OR
          line_exists( lt_existing[ status = gc_status-failed    ] ) OR
          line_exists( lt_existing[ status = gc_status-completed ] ).
    ```
    Also update the `RAISE EXCEPTION` `value =` string immediately below (lines 291–292). Replace the trailing segment:
    ```abap
                   && | with the same parameters (RUNNING or FAILED)|.
    ```
    With:
    ```abap
                   && | with the same parameters (RUNNING, FAILED or COMPLETED)|.
    ```

  - **6e — Retarget `cleanup_old_instances()` SELECT and DELETE (lines 443–457):**
    First, declare the range table at the **top of the method body** alongside any existing `DATA` declarations (not inline immediately before the SELECT — mid-method `DATA` statements are a syntax error in ABAP). Add:
    ```abap
    DATA lt_cleanup_statuses TYPE RANGE OF zfi_process_status.
    lt_cleanup_statuses = VALUE #(
      ( sign = 'I' option = 'EQ' low = gc_status-cancelled  )
      ( sign = 'I' option = 'EQ' low = gc_status-superseded ) ).
    ```
    Then replace the SELECT:
    ```abap
    SELECT *
      FROM zfi_proc_inst
      INTO CORRESPONDING FIELDS OF TABLE @lt_old_instances
      WHERE status = @gc_status-completed
        AND ended_at < @lv_cutoff_ts.
    ```
    With:
    ```abap
    SELECT *
      FROM zfi_proc_inst
      INTO CORRESPONDING FIELDS OF TABLE @lt_old_instances
      WHERE status IN @lt_cleanup_statuses
        AND ended_at < @lv_cutoff_ts.
    ```
    And replace the DELETE:
    ```abap
    DELETE FROM zfi_proc_inst
      WHERE status = @gc_status-completed
        AND ended_at < @lv_cutoff_ts.
    ```
    With:
    ```abap
    DELETE FROM zfi_proc_inst
      WHERE status IN @lt_cleanup_statuses
        AND ended_at < @lv_cutoff_ts.
    ```
  - Notes: Both statements must be changed together. The `LOOP AT lt_old_instances` for step deletion between SELECT and DELETE is unaffected — it already iterates whatever the SELECT returns.
    > **Edge case — null `ended_at`:** The cleanup query filters `ended_at < @lv_cutoff_ts`. SUPERSEDED instances should always have `ended_at` set (they can only be superseded from COMPLETED, which always sets `ended_at` on completion). However, if `ended_at` is NULL due to a data inconsistency, the instance will never match the cutoff condition and will never be cleaned up. No code change is required for this edge case — it is accepted as a defensive-only risk — but developers should be aware when diagnosing unexpected DB growth.
    > **⚠️ DEPLOYMENT WARNING — First-run CANCELLED mass-delete:** Before EST-110, `cleanup_old_instances()` only deleted COMPLETED instances — CANCELLED instances were never cleaned by it. After this deployment, the **first invocation** of `cleanup_old_instances()` will delete ALL CANCELLED instances (and SUPERSEDED instances, though those are new) older than `iv_days_to_keep` days, including any historical CANCELLED records that have accumulated since the system went live. This may constitute an unexpected and irreversible loss of audit records. Recommended pre-deployment actions (choose one): (a) run `SELECT COUNT(*) FROM zfi_proc_inst WHERE status = 'CANCELLED' AND ended_at < <cutoff_ts>` to count affected records and confirm the volume is acceptable; (b) temporarily raise `iv_days_to_keep` for the first scheduled run to reduce the sweep; or (c) accept the sweep if historical CANCELLED records are not required for audit purposes. Coordinate with ops before first deployment.

- [x] **Task 7: Update manager test class — rename/invert `test_allows_when_completed`, add `test_allows_when_superseded`**
  - File: `src/zcl_fi_process_manager.clas.testclasses.abap` *(modify)*

  > **IMPORTANT:** The existing method `test_allows_when_completed` (line 32 declaration, line 136 implementation) asserts that COMPLETED does NOT block `create_process()`. After EST-110, COMPLETED blocks — so this method will permanently fail. It must be **renamed and its assertion inverted** to become `test_blocks_when_completed`. Do NOT add a second method with that name alongside the old one.

  - **Action A — Rename declaration (line 32): replace**
    ```abap
    "! Duplicate check passes when existing instance is COMPLETED
    METHODS test_allows_when_completed FOR TESTING RAISING cx_static_check.
    ```
    with:
    ```abap
    "! Duplicate check fires when COMPLETED instance with same hash exists
    METHODS test_blocks_when_completed FOR TESTING RAISING cx_static_check.
    ```

  - **Action B — Add new declaration (after the renamed declaration, after line 32): add**
    ```abap
    "! Duplicate check passes when existing instance is SUPERSEDED
    METHODS test_allows_when_superseded FOR TESTING RAISING cx_static_check.
    ```

  - **Action C — Replace implementation (lines 136–153): replace the entire `test_allows_when_completed` METHOD block**
    ```abap
    METHOD test_blocks_when_completed.
      " Given: a TEST_DUPCHK instance exists with status COMPLETED (same hash)
      DATA(lo_seed) = mo_manager->create_process( iv_process_type = lc_type_check ).
      set_instance_status( iv_instance_id = lo_seed->get_instance_id( )
                           iv_status      = 'COMPLETED' ).

      " When: create_process() is called again with same type and no parameters
      " Then: zcx_fi_process_error is raised with textid = duplicate_instance
      DATA lx_err TYPE REF TO zcx_fi_process_error.
      TRY.
          mo_manager->create_process( iv_process_type = lc_type_check ).
          cl_abap_unit_assert=>fail( 'Expected duplicate_instance exception for COMPLETED' ).
        CATCH zcx_fi_process_error INTO lx_err.
          cl_abap_unit_assert=>assert_equals(
            act = lx_err->if_t100_message~t100key
            exp = zcx_fi_process_error=>duplicate_instance
            msg = 'Wrong textid raised; expected duplicate_instance for COMPLETED' ).
      ENDTRY.
    ENDMETHOD.
    ```

  - **Action D — Add new implementation (after `test_blocks_when_completed` ENDMETHOD, before `test_allows_when_flag_set`): add**
    ```abap
    METHOD test_allows_when_superseded.
      " Given: a TEST_DUPCHK instance exists with status SUPERSEDED (same hash)
      DATA(lo_seed) = mo_manager->create_process( iv_process_type = lc_type_check ).
      set_instance_status( iv_instance_id = lo_seed->get_instance_id( )
                           iv_status      = 'SUPERSEDED' ).

      " When: create_process() is called again with same type and no parameters
      " Then: no exception; new instance returned
      DATA lo_new TYPE REF TO zcl_fi_process_instance.
      TRY.
          lo_new = mo_manager->create_process( iv_process_type = lc_type_check ).
        CATCH zcx_fi_process_error INTO DATA(lx_err2).
          cl_abap_unit_assert=>fail(
            |Unexpected exception for SUPERSEDED case: { lx_err2->get_text( ) }| ).
      ENDTRY.
      cl_abap_unit_assert=>assert_bound(
        act = lo_new
        msg = 'create_process() should return bound instance when seed is SUPERSEDED' ).
    ENDMETHOD.
    ```
  - Notes: Net result: 5 existing tests → 5 tests (1 renamed+inverted) + 1 new = 6 total. `set_instance_status()` helper already exists. Teardown already cleans both process types — no changes needed there.
    > **`set_instance_status()` raw string values:** The helper takes a raw `zfi_process_status` DB value string (e.g. `'SUPERSEDED'`, `'COMPLETED'`) directly — this is test infrastructure for DB injection, not production code. Using raw string literals here is intentional and consistent with all existing test methods in this class. It is NOT a Constitution Principle I violation — Principle I applies to production ABAP comparisons, not to test helpers that manipulate DB state for test setup.
    > **Test ordering safety:** ABAP Unit fires the `teardown` method after **each individual test method** (not only at class level). This means the COMPLETED seed created in `test_blocks_when_completed` is deleted by `teardown` before `test_allows_when_superseded` begins — there is no ordering dependency between these two test methods. This is standard ABAP Unit behaviour (per-method setup/teardown lifecycle). No special ordering or inter-test coordination is required.

- [x] **Task 8: Extend `check_duplicate` health check with sub-scenarios 3 and 4**
  - File: `src/zcl_fiproc_health_chk_query.clas.abap` *(modify)*
  - Action: Update `testlogic` field description and extend the `check_duplicate` METHOD body (currently lines 2445–2520) to add sub-scenarios 3 and 4 after sub-scenario 2.

  The extended `check_duplicate` METHOD body (replace everything between `METHOD check_duplicate.` and `ENDMETHOD.`):
  ```abap
    "! <p>Validates duplicate check behavior of ZCL_FI_PROCESS_MANAGER=>create_process().</p>
    DATA: ls_row      TYPE ty_row,
          lv_start_ts TYPE timestampl,
          lv_end_ts   TYPE timestampl.

    ls_row-capabilityid = 'DUPLICATE_CHECK'.
    ls_row-description  = 'Duplicate check: blocks RUNNING/FAILED/COMPLETED; allows SUPERSEDED'.
    ls_row-testlogic    = 'Sub-1: Creates TEST_DUPCHK seed (NEW), sets RUNNING, calls'
                        && ' create_process() again — expects duplicate_instance raised.'
                        && ' Sub-2: Verifies hash(empty) <> hash(non-empty).'
                        && ' Sub-3: Seed set COMPLETED — create_process() must still block.'
                        && ' Sub-4: Seed superseded via supersede_process(), then create_process()'
                        && ' must succeed (SUPERSEDED does not block).'.

    GET TIME STAMP FIELD lv_start_ts.

    TRY.
        DATA(lo_manager) = zcl_fi_process_manager=>get_instance( ).

        " --- Sub-scenario 1: existing RUNNING instance blocks creation ---
        DATA(lo_seed) = lo_manager->create_process( iv_process_type = 'TEST_DUPCHK' ).
        DATA(ls_seed) = lo_seed->get_instance( ).
        UPDATE zfi_proc_inst SET status = 'RUNNING'
          WHERE instance_id = @ls_seed-instance_id.
        COMMIT WORK AND WAIT.

        DATA lv_check_fired TYPE abap_bool VALUE abap_false.
        TRY.
            lo_manager->create_process( iv_process_type = 'TEST_DUPCHK' ).
          CATCH zcx_fi_process_error INTO DATA(lx_dup1).
            IF lx_dup1->if_t100_message~t100key = zcx_fi_process_error=>duplicate_instance.
              lv_check_fired = abap_true.
            ENDIF.
        ENDTRY.

        IF lv_check_fired = abap_false.
          ls_row-status  = gc_red.
          ls_row-message = 'Sub-1 FAILED: duplicate check did not fire for RUNNING instance'.
          ls_row-statuscriticality = status_to_criticality( ls_row-status ).
          APPEND ls_row TO mt_rows.
          RETURN.
        ENDIF.

        " --- Sub-scenario 2: different params produce a distinct hash ---
        DATA lv_hash_empty TYPE zfi_process_par_hash.
        DATA lv_hash_diff  TYPE zfi_process_par_hash.
        lv_hash_empty = zcl_fi_process_instance=>get_parameter_hash( iv_parameter_data = '' ).
        lv_hash_diff  = zcl_fi_process_instance=>get_parameter_hash(
                          iv_parameter_data = 'DUPCHK_PARAM=VALUE_DIFF' ).

        IF lv_hash_empty = lv_hash_diff.
          ls_row-status  = gc_red.
          ls_row-message = 'Sub-2 FAILED: hash collision — different params produced same hash'.
          ls_row-statuscriticality = status_to_criticality( ls_row-status ).
          APPEND ls_row TO mt_rows.
          RETURN.
        ENDIF.

        " --- Sub-scenario 3: COMPLETED instance must also block ---
        " Neutralize the RUNNING seed from sub-scenario 1 first (set to CANCELLED)
        " so it does not mask the COMPLETED block result.
        UPDATE zfi_proc_inst SET status = 'CANCELLED'
          WHERE instance_id = @ls_seed-instance_id.
        COMMIT WORK AND WAIT.

        DATA(lo_completed) = lo_manager->create_process( iv_process_type = 'TEST_DUPCHK' ).
        DATA(ls_completed) = lo_completed->get_instance( ).
        UPDATE zfi_proc_inst SET status = 'COMPLETED'
          WHERE instance_id = @ls_completed-instance_id.
        COMMIT WORK AND WAIT.

        DATA lv_completed_blocked TYPE abap_bool VALUE abap_false.
        TRY.
            lo_manager->create_process( iv_process_type = 'TEST_DUPCHK' ).
          CATCH zcx_fi_process_error INTO DATA(lx_dup3).
            IF lx_dup3->if_t100_message~t100key = zcx_fi_process_error=>duplicate_instance.
              lv_completed_blocked = abap_true.
            ENDIF.
        ENDTRY.

        IF lv_completed_blocked = abap_false.
          ls_row-status  = gc_red.
          ls_row-message = 'Sub-3 FAILED: duplicate check did not fire for COMPLETED instance'.
          ls_row-statuscriticality = status_to_criticality( ls_row-status ).
          APPEND ls_row TO mt_rows.
          RETURN.
        ENDIF.

        " --- Sub-scenario 4: SUPERSEDED instance allows new creation ---
        " Supersede the COMPLETED instance; then create_process() must succeed.
        lo_manager->supersede_process( ls_completed-instance_id ).

        DATA lo_after_supersede TYPE REF TO zcl_fi_process_instance.
        TRY.
            lo_after_supersede = lo_manager->create_process(
                                   iv_process_type = 'TEST_DUPCHK' ).
          CATCH zcx_fi_process_error INTO DATA(lx_dup4).
            ls_row-status  = gc_red.
            ls_row-message = |Sub-4 FAILED: create_process() blocked after supersede: |
                           && lx_dup4->get_text( ).
            ls_row-statuscriticality = status_to_criticality( ls_row-status ).
            APPEND ls_row TO mt_rows.
            RETURN.
        ENDTRY.

        IF lo_after_supersede IS NOT BOUND.
          ls_row-status  = gc_red.
          ls_row-message = 'Sub-4 FAILED: create_process() returned unbound after supersede'.
        ELSE.
          GET TIME STAMP FIELD lv_end_ts.
          ls_row-durationms = CONV i( ( lv_end_ts - lv_start_ts ) * 1000 ).
          ls_row-status  = gc_green.
          ls_row-message = 'All 4 sub-scenarios passed: RUNNING/COMPLETED block; ' &&
                           'different-hash and SUPERSEDED allow'.
        ENDIF.

      CATCH zcx_fi_process_error INTO DATA(lx_err).
        ls_row-status  = gc_red.
        ls_row-message = |ZCX: { lx_err->get_text( ) }|.
      CATCH cx_root INTO DATA(lx_root).
        ls_row-status  = gc_red.
        ls_row-message = |Exception: { lx_root->get_text( ) }|.
    ENDTRY.

    GET TIME STAMP FIELD lv_end_ts.
    IF ls_row-durationms = 0.
      ls_row-durationms = CONV i( ( lv_end_ts - lv_start_ts ) * 1000 ).
    ENDIF.
    ls_row-statuscriticality = status_to_criticality( ls_row-status ).
    APPEND ls_row TO mt_rows.
  ```
  - Notes: `TEST_DUPCHK` cleanup is handled by `cleanup_test_instances()` in `zcl_fiproc_health_chk_query`, which is called at the **start** of every `build_rows()` invocation (line 124). This means instances created during a `check_duplicate` run (including any seeds created before an early `RETURN`) persist in the DB until the next `build_rows()` call — they are not cleaned up mid-run. This is consistent with all other health check scenarios and is acceptable because: (a) all TEST_DUPCHK instances are cleaned at the top of every full run, (b) the health check is run infrequently in production-adjacent contexts, and (c) TEST_DUPCHK instances carry a recognisable process_type prefix and cannot be confused with real data. No additional cleanup logic is required in the method body. The neutralization of the sub-1 seed (set to CANCELLED) is necessary because the empty-param hash is shared by all no-param instances; without it, `get_instances_by_hash` would return the RUNNING instance alongside the COMPLETED one, and the RUNNING status alone would be sufficient to trigger the block — but the test goal is to confirm COMPLETED specifically blocks.
    > **`cleanup_test_instances()` scope clarification:** `cleanup_test_instances()` executes `DELETE FROM zfi_proc_inst WHERE process_type LIKE 'TEST_%'` with **no status filter** — it deletes all rows matching the process_type prefix regardless of status. This means that any COMPLETED TEST_DUPCHK instances left behind by a previous interrupted health check run (e.g. a run that returned early after sub-scenario 3 before superseding the seed) will be cleaned on the next `build_rows()` call. There is no risk of COMPLETED orphans persisting indefinitely. COMPLETED is described as a "permanent record" only for non-test process types.
    > **Double `GET TIME STAMP` is intentional:** The provided replacement body calls `GET TIME STAMP FIELD lv_end_ts` twice — once inside the success branch (to capture an accurate end time before computing `durationms`), and once after the outer TRY/ENDTRY as a fallback covering all error paths. The `IF ls_row-durationms = 0` guard prevents `durationms` from being overwritten on the success path. This is not a bug.
    > **⚠️ Replacement includes the final APPEND block:** The provided replacement body already contains the final `GET TIME STAMP FIELD lv_end_ts` / `ls_row-durationms` guard / `ls_row-statuscriticality` / `APPEND ls_row TO mt_rows` lines at the end. Do NOT copy these lines from the old method body after inserting the replacement — they are already present. Replacing only the content between `METHOD check_duplicate.` and `ENDMETHOD.` with the provided block is sufficient.

---

### Acceptance Criteria

- [ ] **AC1:** Given a process type with default config, when `create_process()` is called and a RUNNING instance exists with the same hash, then `zcx_fi_process_error=>duplicate_instance` is raised. *(preserved from EST-102)*

- [ ] **AC2:** Given a process type with default config, when `create_process()` is called and a FAILED instance exists with the same hash, then `zcx_fi_process_error=>duplicate_instance` is raised. *(preserved from EST-102)*

- [ ] **AC3:** Given a process type with `ALLOW_DUPLICATE = 'X'`, when `create_process()` is called and a RUNNING instance exists, then no exception is raised and a new instance is returned. *(preserved from EST-102)*

- [ ] **AC4:** Given a process type with default config, when `create_process()` is called and the only existing instances with the same hash have status CANCELLED, then no exception is raised. *(preserved from EST-102)*

- [ ] **AC5:** Given a process type with default config, when `create_process()` is called with different parameters (different hash) while a RUNNING instance exists, then no exception is raised. *(preserved from EST-102)*

- [ ] **AC6:** Given a process type with default config and no existing instances, when `create_process()` is called, then no exception is raised. *(preserved from EST-102)*

- [ ] **AC7:** **(NEW)** Given a process type with default config, when `create_process()` is called and a COMPLETED instance exists with the same hash, then `zcx_fi_process_error=>duplicate_instance` is raised.

- [ ] **AC8:** **(NEW)** Given a COMPLETED instance, when `supersede_process( instance_id )` is called, then the instance status becomes `SUPERSEDED` and no exception is raised.

- [ ] **AC9:** **(NEW)** Given a SUPERSEDED instance with the same hash exists, when `create_process()` is called, then no exception is raised and a new instance is returned.

- [ ] **AC10:** **(NEW)** Given an instance in any status other than COMPLETED, when `supersede()` is called on it, then `zcx_fi_process_error=>instance_superseded` is raised.

- [ ] **AC11:** **(NEW)** Given an instance in any status other than RUNNING or FAILED, when `cancel()` is called on it, then `zcx_fi_process_error=>invalid_status` is raised (including NEW, COMPLETED, SUPERSEDED, CANCELLED, SKIPPED).

- [ ] **AC12:** **(NEW)** Given a SUPERSEDED instance, when `execute()` is called on it, then `zcx_fi_process_error=>invalid_status` is raised.

- [ ] **AC13:** **(NEW)** Given `cleanup_old_instances( iv_days_to_keep = N )` is called, then CANCELLED and SUPERSEDED instances older than N days are deleted; COMPLETED instances are NOT deleted regardless of age.

- [ ] **AC14:** Given the health check `DUPLICATE_CHECK` scenario runs, then: all 4 sub-scenarios pass, status = GREEN, description and testlogic fields populated, durationms > 0. *(extended from EST-102)*

- [ ] **AC15:** **(NEW — inherited behaviour, no code change)** Given `supersede_process( iv_instance_id )` is called with a non-existent instance ID, then `zcx_fi_process_error=>instance_not_found` is raised. This is propagated automatically from `load_process()` — no additional guard is required in `supersede_process()`. Developers should verify this path manually after Task 6c.

---

## Additional Context

### Dependencies

- EST-102 is fully implemented — all 8 tasks from that spec are done. EST-110 builds directly on it.
- Task 1 (DDIC) must be activated in SAP before ABAP compilation of Tasks 3–8.
- Task 2 (exception constant + message) must be done before Task 3 (`supersede()` RAISE compiles).
- Tasks 3–5 (instance class) must be done before Task 6 (manager `supersede_process()` delegates to `supersede()`).
- Tasks 3–6 must be done before Task 7 (unit tests) and Task 8 (health check extension).

### Testing Strategy

**Unit Tests** (`zcl_fi_process_manager.clas.testclasses.abap` — Task 7):
- `test_allows_when_completed` renamed to `test_blocks_when_completed` with inverted assertion
- 1 new test method added: `test_allows_when_superseded`
- Total test count stays at 5 (rename) + 1 new = 6
- Run via SE80 / ATC: `RUN UNIT TESTS FOR CLASS zcl_fi_process_manager`
- DURATION SHORT, RISK LEVEL HARMLESS

**Integration Health Check** (`zcl_fiproc_health_chk_query` — Task 8):
- `DUPLICATE_CHECK` scenario extended from 2 to 4 sub-scenarios
- Run via existing health check UI or `set_filter('DUPLICATE_CHECK') → build_rows() → get_results()`
- Existing unit test `test_duplicate_check` in `zcl_fiproc_health_chk_query.clas.testclasses.abap` covers the health check end-to-end — no changes needed to that test class

**Manual Verification:**
1. Activate `ZFI_PROCESS_STATUS` domain in SE11 — verify SUPERSEDED fixed value visible
2. Run ABAP Unit for `ZCL_FI_PROCESS_MANAGER` — all 6 tests GREEN
3. Run health check `DUPLICATE_CHECK` — GREEN (all 4 sub-scenarios)
4. Manually call `supersede_process()` on a COMPLETED instance — verify status = SUPERSEDED in DB
5. Verify `cleanup_old_instances()` removes aged CANCELLED/SUPERSEDED but not COMPLETED

### Notes

- **SUPERSEDED vs CANCELLED cleanup rationale:** COMPLETED instances are permanent audit records — they document that a process ran to completion. CANCELLED instances and SUPERSEDED instances are both "abandoned" records that can be cleaned up after the retention period.
- **⚠️ First-run CANCELLED mass-delete (deployment risk):** Before EST-110, `cleanup_old_instances()` never cleaned CANCELLED instances. After this deployment, the first run will sweep all aged CANCELLED records. Coordinate with ops before deploying — see Task 6e deployment warning for recommended pre-checks.
- **`ended_at` preservation for SUPERSEDED:** The `supersede()` method does not overwrite `ended_at`. The timestamp represents when execution completed, not when it was superseded. This preserves the audit trail.
- **`cancel()` NEW/PENDING/QUEUED not cancellable:** If a NEW instance needs to be discarded without executing, the recommended approach is to delete it directly or let `cleanup_old_instances()` handle it after the retention period. Adding cancel support for pre-execution statuses is a separate, future consideration.
- **Risk: `lx_dup3` and `lx_dup4` variable names in health check:** Standard ABAP inline `INTO DATA(...)` declarations — `lx_dup3` and `lx_dup4` are local to their respective CATCH blocks and do not conflict.
- **Future consideration:** If `restart_process()` should also be guarded against SUPERSEDED (a restarted SUPERSEDED instance doesn't make semantic sense), that guard can be added to `restart()` as a follow-on. Currently `restart()` only allows FAILED — SUPERSEDED will naturally fall through to the existing `not FAILED` error path.
