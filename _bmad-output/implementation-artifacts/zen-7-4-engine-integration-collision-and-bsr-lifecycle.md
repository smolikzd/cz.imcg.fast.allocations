# Story zen-7-4: Engine Integration — Collision Detection and BSR Lifecycle

```yaml
story_id: "zen-7-4"
epic_id: "zen-7"
title: "Engine Integration — Collision Detection and BSR Lifecycle"
target_repository: "cz.en.orch"
depends_on: ["zen-7-3"]
constitution_principles:
  - "Principle I — DDIC-First"
  - "Principle V — Error Handling"
status: "done"
```

---

## User Story

As an engine,
I want `create_performance` to block duplicate runs and `sweep_all` to maintain BSR registrations automatically,
So that no two performances can run the same score+params simultaneously, and BSR entries are kept clean without manual intervention.

---

## Acceptance Criteria

**AC1 — Collision detection in create_performance:**
- Given `create_performance` is called with a `score_id` + `params_json`
- When a non-terminal performance with the same `score_id` + SHA-256 hash of `params_json` already exists
- Then `ZCX_EN_ORCH_ERROR` is raised with textid `PERFORMANCE_COLLISION` and `MV_PERF_UUID` set to the colliding UUID
- And no new performance row is inserted

**AC2 — BSR registration on successful create:**
- Given `create_performance` succeeds and creates a new performance
- When the method returns the new `perf_uuid`
- Then a BSR entry is registered via `ZIF_EN_ORCH_BSR=>register` using `SCORE_ID + ':' + PERF_UUID` as the BSR key and the params hash

**AC3 — BSR deregistration in sweep_all on terminal state:**
- Given `sweep_all` processes a performance that has just reached a terminal state (C/F/X)
- When the performance status is written as terminal
- Then `ZIF_EN_ORCH_BSR=>deregister` is called for that performance's BSR key
- And the deregister call happens before `COMMIT WORK` for that performance

**AC4 — BSR injection for testability:**
- Given the BSR instance is injected into `ZCL_EN_ORCH_ENGINE`
- When the engine is constructed via `get_instance()`
- Then the engine holds a reference to `ZIF_EN_ORCH_BSR` (created via `ZCL_EN_ORCH_BSR=>create()`)
- And tests can inject a mock BSR via a package-private `set_bsr` method

---

## Implementation Notes

### SHA-256 hashing
Use `CL_ABAP_MESSAGE_DIGEST=>calculate_hash_for_char`:
```abap
TRY.
    cl_abap_message_digest=>calculate_hash_for_char(
      EXPORTING if_algorithm  = 'SHA256'
                if_data       = iv_params_json
      IMPORTING ef_hashstring = lv_hash ).
  CATCH cx_abap_message_digest INTO DATA(lx_hash).
    RAISE EXCEPTION TYPE zcx_en_orch_error
      EXPORTING textid    = zcx_en_orch_error=>engine_sweep_failed
                mv_detail = lx_hash->get_text( )
                previous  = lx_hash.
ENDTRY.
```
Result `ef_hashstring` is an uppercase hex string (64 chars). This becomes the `iv_params_hash` for BSR.
Empty `params_json` is a valid input — hash of empty string is well-defined, no special handling needed.

### BSR key convention
`iv_bsr_key = iv_score_id && ':' && rv_perf_uuid`

This is unique per performance, opaque, and never interpreted.

### `PERFORMANCE_COLLISION` textid
Already defined in `ZCX_EN_ORCH_ERROR` as `msgno 042`, with `attr1 = MV_SCORE_ID`.
The colliding `perf_uuid` must be set in `MV_PERF_UUID` — but `attr2` is currently empty for this textid.
The AC requires `MV_PERF_UUID` to be set on the exception so callers can identify which performance is blocking.
**Decision**: Keep `PERFORMANCE_COLLISION` message text as-is (it already references `MV_SCORE_ID`).
The `MV_PERF_UUID` field is set on the exception instance even though it is not in `attr2` — it is available programmatically.

### Changes to `create_performance`
Insert a collision check between T1.1 (score check) and T1.2 (UUID generation):
1. Compute SHA-256 of `iv_params_json` → `lv_hash`
2. `SELECT SINGLE perf_uuid FROM zen_orch_perf WHERE score_id = ... AND params_hash = ... AND status NOT IN ('C','F','X')`
   - If found → raise `PERFORMANCE_COLLISION` with the found `perf_uuid`
3. On successful insert, call `mo_bsr->register( iv_bsr_key = ... iv_params_hash = lv_hash )`

> **NOTE**: `ZEN_ORCH_PERF` does NOT currently have a `params_hash` column.
> The collision check must instead use a computed approach: SELECT all non-terminal performances with `score_id = iv_score_id` and compare `params_json` in ABAP — or add a persisted `params_hash` column.
>
> Decision: **Add `PARAMS_HASH` column to `ZEN_ORCH_PERF`** (Type: `ZEN_ORCH_DE_PARAMS_HASH`, 64 chars) so the DB filter is efficient. This is a DDIC change (Principle I).
> The column is populated on INSERT and used in the collision SELECT.

### Changes to `sweep_all`
After `advance_performance` returns (performance has been advanced to its new state), check if the new status is terminal. If yes, deregister before `COMMIT WORK`:
```abap
" After advance_performance returns, read the current status
SELECT SINGLE status, score_id
  FROM zen_orch_perf
  WHERE perf_uuid = @ls_perf_uuid
  INTO @DATA(ls_perf).
IF ls_perf-status = gc_status-completed
OR ls_perf-status = gc_status-failed
OR ls_perf-status = gc_status-cancelled.
  DATA(lv_bsr_key) = ls_perf-score_id && ':' && ls_perf_uuid.
  TRY.
      mo_bsr->deregister( lv_bsr_key ).
    CATCH zcx_en_orch_error.   "#EC NEEDED — best-effort, log only
  ENDTRY.
ENDIF.
COMMIT WORK AND WAIT.
```

> The deregister is best-effort: if it fails (e.g. key was never registered because the performance was created before this story), log but do not fail the sweep.

### Injection pattern
Add to `PRIVATE SECTION`:
```abap
DATA mo_bsr TYPE REF TO zif_en_orch_bsr.

METHODS set_bsr
  IMPORTING io_bsr TYPE REF TO zif_en_orch_bsr.
```
`get_instance` populates `mo_bsr` via `ZCL_EN_ORCH_BSR=>create()` after constructing the engine.

---

## DDIC Changes Required

| Object | Type | Change |
|--------|------|--------|
| `ZEN_ORCH_DE_PARAMS_HASH` | Domain | New: CHAR 64, fixed-values none |
| `ZEN_ORCH_E_PARAMS_HASH` | Data element | New: domain `ZEN_ORCH_DE_PARAMS_HASH` |
| `ZEN_ORCH_PERF` | Table | Add field `PARAMS_HASH TYPE ZEN_ORCH_E_PARAMS_HASH` |

---

## Files to Modify

| File | Change |
|------|--------|
| `src/zen_orch_de_params_hash.doma.xml` | NEW — domain CHAR 64 |
| `src/zen_orch_e_params_hash.dtel.xml` | NEW — data element |
| `src/zen_orch_perf.tabl.xml` | ADD field PARAMS_HASH |
| `src/zcl_en_orch_engine.clas.abap` | ADD mo_bsr, set_bsr, collision check in create_performance, deregister in sweep_all |

---

## Constitution Compliance

- **Principle I — DDIC-First**: New `PARAMS_HASH` column uses new domain + data element. No local TYPE definitions.
- **Principle V — Error Handling**: `PERFORMANCE_COLLISION` raised with context. Hash failure re-raised as `engine_sweep_failed`. Deregister failure logged best-effort (no silent swallow — caught and logged via get_logger).

---

## Review Findings

Code review performed after initial commit `9c109af`. Five issues identified and patched.

### BUG-1 (CRITICAL — FIXED): `register` call missing `iv_perf_uuid` and `iv_score_id`

`zif_en_orch_bsr=>register` requires 4 parameters. The original call only passed `iv_bsr_key` and `iv_params_hash`, leaving `iv_perf_uuid` and `iv_score_id` at initial. Every BSR row would have been inserted with `perf_uuid = initial`, causing `check_prerequisite` to always look up the zero-UUID, find nothing, and return `failed` — meaning all PREREQ_GATEs would have immediately blocked forever.

**Fix**: Added `iv_perf_uuid = lv_uuid` and `iv_score_id = iv_score_id` to the `register` call in `create_performance`.

### BUG-2 (CRITICAL — FIXED): `deregister` call in `sweep_all` missing `iv_perf_uuid`

`zif_en_orch_bsr=>deregister` requires both `iv_bsr_key` and `iv_perf_uuid`. The original call used positional syntax passing only `lv_bsr_key`, leaving `iv_perf_uuid` at initial. The resulting `DELETE WHERE perf_uuid = initial` would match nothing — stale BSR entries would accumulate indefinitely.

**Fix**: Changed to named-parameter syntax: `deregister( iv_bsr_key = lv_bsr_key iv_perf_uuid = ls_perf_uuid )`.

### BUG-3 (IMPORTANT — FIXED): `cancel_performance` does not deregister from BSR

`cancel_performance` sets the performance to CANCELLED and commits, but never calls `mo_bsr->deregister`. `sweep_all` only loads PENDING/RUNNING performances, so a CANCELLED performance never passes through the sweep deregistration path. Externally cancelled performances would leave permanent stale BSR entries.

**Fix**: Added deregister call with best-effort error handling in `cancel_performance` before `COMMIT WORK AND WAIT` (new step T1.6, original T1.6 renumbered to T1.7).

### BUG-4 (QUALITY — FIXED): Inconsistent XML indentation in `zen_orch_perf.tabl.xml`

The edit that added `PARAMS_HASH` also re-indented the `PARAMS_JSON`, `PARAMS_HASH`, and `CREATED_BY` `<DD03P>` blocks with one extra space level relative to the surrounding blocks.

**Fix**: Normalized all three blocks to the standard indentation used by the rest of the file.

### NOTE (DEFERRED): `set_bsr` FRIENDS declaration for test classes

`set_bsr` is PRIVATE and `GLOBAL FRIENDS` only lists `zcl_en_orch_health_chk_query`. Test classes for zen-7-6 will need to inject a mock BSR via `set_bsr`. Deferred to zen-7-6 (test infrastructure story) — no change needed now.

---

## Completion

- status: `done`
- completion_date: 2026-04-09
- commit_hash: 7258f69
- commit_repository: `cz.en.orch`
