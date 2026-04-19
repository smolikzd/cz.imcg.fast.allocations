---
title: 'EST-205: Fluent Score Builder for ZEN_ORCH orchestration definition'
type: 'feature'
created: '2026-04-18'
status: 'done'
baseline_commit: '9fb78a4ddedb20dd63d74d4684e3b35985e7fd30'
context:
  - '_bmad/_memory/constitution.md'
  - 'docs/developer-guides/zen-orch-guide.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Defining a ZEN_ORCH score (process template) requires 30–50 lines of repetitive struct-fill + `MODIFY` statements (see `zen_orch_setup.prog.abap`). Every new setup program duplicates this boilerplate, with no validation, no auto-sequencing, and no readable intent.

**Approach:** Introduce a `ZCL_EN_ORCH_SCORE_BUILDER` class in `cz.en.orch` that wraps score and step table writes behind a fluent method-chaining API. Each builder method returns `REF TO` itself so calls chain. `build()` is the single DB-write entry point (idempotent `MODIFY`).

## Boundaries & Constraints

**Always:**
- Constitution Principle I: no local TYPE definitions — all internal tables typed to DDIC line types (`ZEN_ORCH_SCORE`, `ZEN_ORCH_S_STEP`).
- Constitution Principle II: SAP naming (`ZCL_EN_ORCH_SCORE_BUILDER`), line length ≤120 chars.
- Constitution Principle III: SAP docs consulted for any new patterns (already done in spec research).
- Constitution Principle IV: factory entry point via `CLASS-METHODS for_score` — no direct `NEW` instantiation.
- Constitution Principle V: `ZCX_EN_ORCH_ERROR` raised with proper context on all validation failures.
- `SCORE_SEQ` is auto-incremented by the builder (10, 20, 30 …) — callers never set it.
- `build()` uses `MODIFY` semantics (full replace: `DELETE` then `INSERT FROM TABLE`) so re-running a setup program is idempotent.
- `build()` calls `COMMIT WORK AND WAIT` — matching existing engine pattern.
- Chaining methods return `VALUE(ro_builder) TYPE REF TO zcl_en_orch_score_builder`.

**Ask First:**
- If the seq increment (10) or starting seq (10) should differ from the established `zen_orch_setup` convention.
- If `LOOP` element type support is needed in this story (currently out of scope).
- If any additional `ZEN_ORCH_S_STEP` fields beyond those listed in the I/O matrix should be exposed.

**Never:**
- Do not add business logic (no process orchestration) inside the builder.
- Do not modify `ZCL_EN_ORCH_ENGINE` — builder is a separate concern.
- Do not create a local structure type for the internal step buffer — use `zen_orch_s_step` directly.
- Do not split score header write from step write across separate public methods (atomicity).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Happy path — 1 step | `for_score( 'X' )->add_step( 'ZFI_PROCESS' )->build()` | 1 row in `zen_orch_score`, 1 row in `zen_orch_s_step` with `score_seq = 10` | — |
| Happy path — 3 steps | `->add_step()->add_step()->add_step()->build()` | Steps get seq 10, 20, 30 automatically | — |
| Idempotent re-run | `build()` called twice for same `score_id` | Old rows deleted, new rows inserted; net result identical | — |
| Empty score | `for_score( 'X' )->build()` (no steps added) | `ZCX_EN_ORCH_ERROR` raised: "Score must have at least one element" | Raise before any DB write |
| Blank score_id | `for_score( '' )->...->build()` | `ZCX_EN_ORCH_ERROR`: "score_id is required" | Raised in `for_score` constructor |
| GATE after GATE | Two consecutive `add_gate()` calls | Allowed — seq auto-increments normally | No validation needed |
| Blank adapter_type on STEP | `add_step( iv_adapter_type = '' )` | `ZCX_EN_ORCH_ERROR`: "adapter_type is required for STEP elements" | Raised in `add_step` |

</frozen-after-approval>

## Code Map

- `src/zcl_en_orch_score_builder.clas.abap` -- **NEW** — fluent builder class (target of this story)
- `src/zcl_en_orch_score_builder.clas.xml` -- **NEW** — class metadata / activation XML
- `src/zen_orch_setup.prog.abap` -- reference pattern; NOT modified, used as design baseline
- `src/zcl_en_orch_engine.clas.abap` -- engine singleton; NOT modified; builder complements it
- `src/zen_orch_score.tabl.xml` -- DDIC table `zen_orch_score` — score header
- `src/zen_orch_s_step.tabl.xml` -- DDIC table `zen_orch_s_step` — step rows
- `src/zcx_en_orch_error.clas.abap` -- exception class — raised on builder validation failures

## Tasks & Acceptance

**Execution:**
- [x] `src/zcl_en_orch_score_builder.clas.abap` -- CREATE class with `CREATE PRIVATE`, factory method `for_score`, chaining methods `add_step` / `add_gate` / `add_prereq_gate`, and terminal method `build` -- implements the fluent API described in Design Notes; all validation errors raise `ZCX_EN_ORCH_ERROR`
- [x] `src/zcl_en_orch_score_builder.clas.xml` -- CREATE activation XML for the new class -- required for abapgit transport
- [x] `src/zcl_en_orch_score_builder.clas.abap` (test classes section) -- ADD ABAP Unit tests covering all I/O matrix scenarios: happy path 1-step, happy path 3-steps, idempotent re-run, empty score guard, blank score_id guard, blank adapter_type guard -- validates builder correctness without manual testing

**Acceptance Criteria:**
- Given a score builder instance, when `add_step` is called three times and `build()` is called, then `zen_orch_s_step` contains exactly 3 rows with `score_seq` values 10, 20, 30.
- Given `build()` is called twice for the same `score_id`, when the second call completes, then the row count in `zen_orch_s_step` for that score equals the number of steps added (no duplicates).
- Given `for_score( '' )` is called, when the factory method executes, then `ZCX_EN_ORCH_ERROR` is raised before any method chain is returned.
- Given `add_step( iv_adapter_type = '' )`, when `build()` is called, then `ZCX_EN_ORCH_ERROR` is raised and no rows are written to DB.
- Given `for_score( 'X' )->build()` with no elements added, when `build()` executes, then `ZCX_EN_ORCH_ERROR` is raised.
- Given a valid chain with `add_gate` and `add_prereq_gate`, when `build()` completes, then `zen_orch_s_step` rows have `elem_type` values `GATE` and `PREREQ_GATE` respectively with correct auto-assigned `score_seq`.

## Spec Change Log

## Design Notes

**Method signatures (key API surface):**

```abap
CLASS-METHODS for_score                        " Factory — CREATE PRIVATE enforced here
  IMPORTING iv_score_id    TYPE zen_orch_de_score_id
            iv_description TYPE zen_orch_de_description OPTIONAL
  RETURNING VALUE(ro_builder) TYPE REF TO zcl_en_orch_score_builder
  RAISING   zcx_en_orch_error.                " blank score_id guard

METHODS add_step
  IMPORTING iv_adapter_type  TYPE zen_orch_de_adapter_type
            iv_params_json   TYPE string              DEFAULT ''
            iv_max_parallel  TYPE zen_orch_de_max_par DEFAULT 1
            iv_is_breakpoint TYPE abap_bool           DEFAULT abap_false
  RETURNING VALUE(ro_builder) TYPE REF TO zcl_en_orch_score_builder
  RAISING   zcx_en_orch_error.                " blank adapter_type guard

METHODS add_gate
  IMPORTING iv_ref_id        TYPE zen_orch_de_ref_id  DEFAULT ''
            iv_is_breakpoint TYPE abap_bool           DEFAULT abap_false
  RETURNING VALUE(ro_builder) TYPE REF TO zcl_en_orch_score_builder.

METHODS add_prereq_gate
  IMPORTING iv_ref_score_id TYPE zen_orch_de_score_id
  RETURNING VALUE(ro_builder) TYPE REF TO zcl_en_orch_score_builder.

METHODS build
  RAISING zcx_en_orch_error.                  " empty-score guard + DB writes
```

**Internal state (PRIVATE section) — all DDIC-typed per Principle I:**
```abap
mv_score_id    TYPE zen_orch_de_score_id.
ms_score_hdr   TYPE zen_orch_score.            " score header struct
mt_steps       TYPE STANDARD TABLE OF zen_orch_s_step WITH DEFAULT KEY.
mv_next_seq    TYPE zen_orch_de_score_seq VALUE 10.  " auto-increments by 10
```

**`build()` sequence:**
1. Guard: `mt_steps` is empty → raise error.
2. `DELETE FROM zen_orch_score WHERE score_id = mv_score_id.`
3. `DELETE FROM zen_orch_s_step WHERE score_id = mv_score_id.`
4. `MODIFY zen_orch_score FROM ms_score_hdr.`
5. `INSERT zen_orch_s_step FROM TABLE mt_steps.`
6. `COMMIT WORK AND WAIT.`

## Verification

**Manual checks (no CLI build available for ABAP):**
- Activate `ZCL_EN_ORCH_SCORE_BUILDER` in SE24 — no syntax errors, activation succeeds.
- Run ABAP Unit tests via SE80 / ATC — all test methods in the test class pass (green).
- Execute a small test program calling the builder for score `TEST_FLUENT` with 2 steps; verify rows in `ZEN_ORCH_SCORE` and `ZEN_ORCH_S_STEP` via SE16N.
- Re-run the test program; verify row counts unchanged (idempotency confirmed).
