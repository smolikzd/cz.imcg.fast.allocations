---
title: 'EST-206: ZEN_ORCH_ALLOC_LOOP_SETUP — sequential allocation run periods 1–6'
type: 'feature'
created: '2026-04-18'
status: 'done'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** There is no automated way to run the ALLOCATIONS process for periods 1–6 of FY2024 via ZEN_ORCH. Running them manually one at a time is error-prone and gives no orchestrated sequencing guarantee.

**Approach:** Create an idempotent ABAP program `ZEN_ORCH_ALLOC_LOOP_SETUP` in `cz.en.orch` that registers the adapter (direct MODIFY on `ZEN_ORCH_ADPT_R`), defines a score using the **fluent builder API** (`ZCL_EN_ORCH_SCORE_BUILDER`) with 6 sequential STEP elements (one per fiscal period, each with distinct `params_json`), cancels any active prior performance, creates a new performance, and triggers `sweep_all()`.

## Boundaries & Constraints

**Always:**
- Use `ZCL_EN_ORCH_SCORE_BUILDER` fluent API for score + step definition (`for_score()→add_step()×6→build()`).
- Adapter registration remains a direct MODIFY on `ZEN_ORCH_ADPT_R` (no fluent API exists for adapter registry).
- Each of the 6 score steps must have a distinct `params_json` with `fiscal_period` baked in (`"001"` through `"006"`).
- Score-level `params_json` left empty (step-level params take priority).
- Program must be idempotent (safe to re-run without duplicates or errors). `build()` uses DELETE+INSERT internally — this is correct and intentional for the score builder.
- Score ID: `ALLOC_CZ01_2024`. Adapter ID/type: `ZFI_PROCESS`. Steps SEQ 10, 20, 30, 40, 50, 60 (auto-assigned by builder).
- Constitution Principle I: no local TYPE definitions — use DDIC types only.
- Constitution Principle II: SAP naming, line length ≤120 chars.
- Follow `ZEN_ORCH_SETUP` style for structure: top-level DATA declarations, cancel-before-create, post-sweep status read-back.

**Ask First:**
- If `company_code`, `fiscal_year`, or `allocation_id` defaults need to change from `1000`, `2024`, `1`.
- If periods 1–6 are not the correct range.

**Never:**
- Do not use ZEN_ORCH LOOP element (confirmed not suitable for distinct-param iteration).
- Do not generate ABAP in the planning repo — file goes in `cz.en.orch`.
- Do not use raw MODIFY/INSERT for score steps — use `ZCL_EN_ORCH_SCORE_BUILDER` instead.

</frozen-after-approval>

## Code Map

- `/Users/smolik/DEV/cz.en.orch/src/zen_orch_setup.prog.abap` — style reference (MODIFY idiom, cancel-before-create, sweep pattern)
- `/Users/smolik/DEV/cz.en.orch/src/zen_orch_adpt_r.tabl.xml` — adapter registry table (MANDT, ADAPTER_TYPE, IMPL_CLASS, audit fields)
- `/Users/smolik/DEV/cz.en.orch/src/zen_orch_score.tabl.xml` — score definition table (MANDT, SCORE_ID, DESCRIPTION, PARAMS_JSON, audit)
- `/Users/smolik/DEV/cz.en.orch/src/zen_orch_s_step.tabl.xml` — score step table (MANDT, SCORE_ID, SCORE_SEQ, ELEM_TYPE, ADAPTER_TYPE, PARAMS_JSON, MAX_PARALLEL, IS_BREAKPOINT)
- `/Users/smolik/DEV/cz.imcg.fast.ovysledovka/src/zcl_fi_alloc_orch_adapter.clas.abap` — adapter impl; confirms JSON contract and handle format
- **New file**: `/Users/smolik/DEV/cz.en.orch/src/zen_orch_alloc_loop_setup.prog.abap`

## Tasks & Acceptance

**Execution:**
- [x] `/Users/smolik/DEV/cz.en.orch/src/zen_orch_alloc_loop_setup.prog.abap` -- CREATE new ABAP program -- implements the 5-section setup: (1) register adapter, (2) define score `ALLOC_CZ01_2024`, (3) define 6 sequential steps SEQ 10–60 with period-specific params_json, (4) cancel active performances, (5) create performance + sweep_all + status read-back

**Acceptance Criteria:**
- Given the program is run for the first time, when START-OF-SELECTION executes, then `ZEN_ORCH_ADPT_R`, `ZEN_ORCH_SCORE`, and `ZEN_ORCH_S_STEP` contain the expected rows and a new performance UUID exists in `ZEN_ORCH_PERF` with status `R` or `P`.
- Given the program is run a second time with no active performance, when START-OF-SELECTION executes, then no duplicate rows exist and a fresh performance is created.
- Given an active performance exists before the run, when START-OF-SELECTION executes, then the active performance is cancelled prior to the new one being created.
- Given sweep_all fails, when status is read back from `ZEN_ORCH_PERF`, then status `F` is detected and the failed step handle plus SLG1 hint are written to the list.
- Given the program runs, when completed, then each of the 6 steps' `params_json` contains the correct `fiscal_period` (`001`–`006`) with `company_code` `1000`, `fiscal_year` `2024`, `allocation_id` `1`.

## Design Notes

### Why 6 STEP elements instead of LOOP

`advance_loop()` in `zcl_en_orch_engine` clones inner step rows verbatim from iteration-0 — `params_json` is copied unchanged to every iteration. Using LOOP would produce 6 identical `fiscal_period` values, causing ZFI_PROCESS to raise `duplicate_instance` from iteration 2 onward (same `PARAMETER_HASH`). The correct pattern is 6 explicit STEP rows.

### Fluent builder usage (golden example)

```abap
TRY.
    zcl_en_orch_score_builder=>for_score(
        iv_score_id    = 'ALLOC_CZ01_2024'
        iv_description = 'ALLOCATIONS sequential run periods 001-006 FY2024'
      )->add_step(
          iv_adapter_type = 'ZFI_PROCESS'
          iv_params_json  = lv_json_001
      )->add_step(
          iv_adapter_type = 'ZFI_PROCESS'
          iv_params_json  = lv_json_002
      )" ... repeat for 003–006 ...
      ->build( ).
  CATCH zcx_en_orch_error INTO DATA(lx_build).
    WRITE: / 'Score build failed:', lx_build->get_text( ).
    RETURN.
ENDTRY.
```

`build()` uses DELETE+INSERT internally (full replace) — idempotent, no duplicates.

### params_json construction (golden example)

```abap
DATA(lv_json_001) = '{' &&
  '"process_type":"ALLOCATIONS",'        &&
  '"company_code":"1000",'               &&
  '"fiscal_year":"2024",'                &&
  '"fiscal_period":"001",'               &&
  '"allocation_id":"1",'                 &&
  '"export":"' && p_export && '"'        &&
  '}'.
```

`p_export` is TYPE `abap_boolean` — space (false) serialises correctly via `/ui2/cl_json`.

### SCORE_SEQ field name

`SCORE_SEQ` is auto-assigned by the builder (10, 20, 30 … per call order) — no manual seq management needed.

## Verification

**Manual checks (no CLI for ABAP):**
- Run program in SAP system; expect WRITE output confirming MODIFY OK for adapter, score, and all 6 steps.
- Check `ZEN_ORCH_PERF` (SE16 or Fiori dashboard) — performance UUID exists with status `R` or `P`.
- Check `ZEN_ORCH_S_STEP` — 6 rows for `ALLOC_CZ01_2024` with `fiscal_period` `001`–`006`.
- After ZFI_PROCESS completes period 1, engine advances to period 2 automatically.
