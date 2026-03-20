---
title: 'Real-time BAL Log Visibility During Step Execution (EST-120)'
type: 'bugfix'
created: '2026-03-20'
status: 'done'
baseline_commit: '773f8b897dea2fbce828d42e15f64f08285f60ca'
target_repository: 'planner'
context:
  - '_bmad-output/project-context.md'
  - '_bmad/_memory/constitution.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

# Real-time BAL Log Visibility During Step Execution (EST-120)

## Intent

**Problem:** BAL log messages logged during step execution are only visible in SLG1 after the step finishes (or the process completes). During long-running steps, operators have no real-time visibility into progress. This is because `message()` only adds items to the in-memory `mo_log` handle, and `save()` is called once after step completion. The actual DB persist requires a subsequent `COMMIT WORK`.

**Approach:** Switch the framework from `save()` (default connection, needs COMMIT) to `save_with_2nd_connection()` (service connection, auto-commits immediately) for all BAL log persistence. This method is already proven in the bgRFC function module. The BALI API constraint that prevents mixing both save methods on the same handle means we must use one method consistently — `save_with_2nd_connection()` is the correct choice since it provides immediate persistence without requiring COMMIT WORK.

## Boundaries & Constraints

**Always:**
- Use `save_with_2nd_connection()` consistently — never mix with `save()` on the same log handle
- Keep the `save()` method on the interface (backward compatibility) but it becomes a simple delegate to `save_with_2nd_connection()`
- `message()` must remain defensive (never raise exceptions, never break business logic)
- Constitution Principles I (DDIC-First), II (SAP Standards), V (Error Handling) apply

**Ask First:**
- If auto-flush frequency needs throttling (e.g., only flush every N messages or every T seconds) to avoid DB overhead — start with every-message flush and optimize only if performance is observed to be a problem
- If `save()` callers outside the framework rely on the default-connection semantics

**Never:**
- Do not add `COMMIT WORK` to step code
- Do not change the bgRFC FM — it already works correctly with `save_with_2nd_connection()`
- Do not change the null logger (`ZCL_FI_PROCESS_NULL_LOGGER`)
- Do not break the BALI constraint: never mix `save()` and `save_with_2nd_connection()` on the same handle

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Serial step logs message | `mo_log->message()` called during step execute | Message immediately visible in SLG1 (within seconds) | If `save_with_2nd_connection()` fails, message stays in memory; no exception raised |
| bgRFC substep logs message | Same pattern via bgRFC FM | No change — already uses `save_with_2nd_connection()` | Same as current |
| Step crashes mid-execution | Exception after several messages logged | All messages up to last flush are persisted in SLG1 | Messages between last flush and crash are lost (acceptable) |
| BALI save fails intermittently | `cx_bali_runtime` from `save_log_2nd_db_connection` | Failure silently caught; next `message()` call attempts flush again | No exception propagated |
| Framework calls `save()` after step | `mo_log->save()` called in `execute_step()` | Delegates to `save_with_2nd_connection()` — no mixing violation | Same defensive pattern |

</frozen-after-approval>

## Code Map

- `planner: src/zif_fi_process_logger.intf.abap` — Interface definition; `save()` doc update
- `planner: src/zcl_fi_process_logger.clas.abap` — Logger implementation; `message()` auto-flush + `save()` delegation
- `planner: src/zcl_fi_process_logger.clas.testclasses.abap` — Unit tests for the new behavior
- `planner: src/zcl_fi_process_instance.clas.abap` — Framework orchestrator; no code changes needed (existing `save()` calls will delegate automatically)
- `planner: src/zfi_bgrfc.fugr.zfi_bgrfc_exec_substep.abap` — bgRFC FM; NO changes (already correct)

## Tasks & Acceptance

**Execution:**
- [ ] `planner: src/zcl_fi_process_logger.clas.abap` — In `message()` method: after `add_item_to_log()`, call `save_with_2nd_connection()` to immediately persist. Wrap in TRY/CATCH cx_bali_runtime (defensive). This makes every message immediately visible in SLG1.
- [ ] `planner: src/zcl_fi_process_logger.clas.abap` — In `save()` method: replace `cl_bali_log_db=>get_instance( )->save_log( )` with `save_with_2nd_connection()` delegation. This eliminates the mixed-method risk from framework `save()` calls after step completion.
- [ ] `planner: src/zif_fi_process_logger.intf.abap` — Update ABAP-Doc on `save()` to note it now delegates to 2nd connection (no COMMIT WORK required by caller).
- [ ] `planner: src/zcl_fi_process_logger.clas.testclasses.abap` — Add test: log 3 messages, verify they are persisted in BALHDR/BALDAT without explicit `save()` call (auto-flush verification).

**Acceptance Criteria:**
- Given a running serial step that logs messages via `mo_log->message()`, when a user opens SLG1 while the step is still executing, then the messages are visible immediately (no need to wait for step completion).
- Given `save()` is called by the framework after step completion, when the log handle was already flushed via `save_with_2nd_connection()`, then no BALI MESSAGE_TYPE_X dump occurs.
- Given a BALI runtime error during auto-flush, when `message()` is called, then the error is silently caught and business logic is unaffected.

## Spec Change Log

## Design Notes

**Why auto-flush in `message()` rather than exposing a separate `flush()` method:**
The interface contract says `message()` is fire-and-forget. Adding auto-flush inside `message()` achieves real-time visibility without requiring any changes to the ~97 existing call sites in step classes, the framework orchestrator, or the bgRFC FM. A separate `flush()` method would require callers to know when to flush and would risk being forgotten.

**Why `save()` delegates instead of being removed:**
`save()` is called in 5 places in `execute_step()` and 2 places in the bgRFC FM (via `save_with_2nd_connection()`). Removing `save()` from the interface would be a breaking change. Delegating makes all existing call sites safe without code changes.

**BALI 2nd connection behavior:**
`save_log_2nd_db_connection()` uses a SAP service connection that commits independently. It can be called multiple times on the same handle — each call persists all items currently in the in-memory log (including previously saved items, which is idempotent). This is the same pattern the bgRFC FM already relies on.

## Verification

**Manual checks (if no CLI):**
- Run a long-running serial step (e.g., PHASE1 with large dataset). While running, open SLG1 and refresh — messages should appear incrementally.
- Run the ABAP Unit test class for `ZCL_FI_PROCESS_LOGGER` — all tests must pass including the new auto-flush test.
- Execute a bgRFC substep flow (PHASE2) — verify no MESSAGE_TYPE_X dumps in ST22.
