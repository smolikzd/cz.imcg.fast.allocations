---
story: "5.1"
title: "Replace MESSAGE TYPE 'E' with RAISE EXCEPTION in PHASE1, PHASE2, PHASE3"
epic: "5 — Error Handling and Observability"
sprint: 2
status: done
priority: critical
estimate: 2h
actual: 2h
completed: 2026-03-09
source_findings:
  - CC-03
  - cross-cutting-analysis.md
target_repo: smolikzd/cz.imcg.fast.planner
dependencies:
  - None (independent)
---

# Story 5.1: Replace `MESSAGE TYPE 'E'` with `RAISE EXCEPTION` in PHASE1, PHASE2, PHASE3

## User Story

As a process monitor user,
I want step failures to surface with their full message text,
so that the root cause is visible in the process monitor without needing to dig into
ABAP dumps.

## Background

PHASE1, PHASE2, and PHASE3 use `MESSAGE e00X(zfi_alloc) TYPE 'E'` as a control flow
mechanism to signal failure. In background processing:
- If there is no CATCH handler in the call stack for the implicit `CX_SY_NO_HANDLER`
  exception, the statement causes a runtime dump.
- Even if caught, the message class reference is opaque in the process monitor.

The correct pattern per Constitution Principle V is:
```abap
RAISE EXCEPTION TYPE zcx_fi_process_error
  EXPORTING textid = zcx_fi_process_error=>invalid_operation
            value  = 'Descriptive message text'.
```

**Exception**: item-level log messages inside `LOOP` processing that use
`MESSAGE ... INTO lv_dummy` + `lo_log->log_sy_msg( )` are a different pattern —
they are per-item observability, not control flow, and should be kept.
Only the **control flow** uses of `MESSAGE TYPE 'E'` are replaced.

## Acceptance Criteria

**AC-01 — No MESSAGE TYPE 'E' in control flow:**
Given any validation or guard check fails inside `execute` or `execute_substep`
When the check triggers
Then `ZCX_FI_PROCESS_ERROR` is raised (not `MESSAGE TYPE 'E'`)
And the process monitor shows the full exception text

**AC-02 — No CX_SY_NO_HANDLER in system log:**
Given the replacement is applied
When a step failure occurs
Then no `CX_SY_NO_HANDLER` appears in the system log (SM21)

**AC-03 — Item-level log messages unchanged:**
Given a per-item `MESSAGE ... INTO lv_dummy` + `log_sy_msg( )` pattern
When execution reaches it
Then it is NOT replaced — it is kept as-is (per-item log is correct pattern)

## Tasks

- [x] **Task 5.1.1**: Read each target class (`execute` and `execute_substep` methods)
  and list every `MESSAGE ... TYPE 'E'` occurrence.

- [x] **Task 5.1.2**: For each control-flow `MESSAGE TYPE 'E'` in `ZCL_FI_ALLOC_STEP_PHASE1`:
  - [x] Identify the message class and number (e.g., `e001(zfi_alloc)`)
  - [x] Read the message text from `ZFI_ALLOC` message class (search remote repo for
    `zfi_alloc.msag.xml` or similar)
  - [x] Replace with:
    ```abap
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING textid = zcx_fi_process_error=>invalid_operation
                value  = '<equivalent message text>'.
    ```
  - [x] Verify `zcx_fi_process_error=>invalid_operation` is the most appropriate textid
    (check `zcx_fi_process_error.clas.abap` for available textids 001–015)

- [x] **Task 5.1.3**: Repeat Task 5.1.2 for `ZCL_FI_ALLOC_STEP_PHASE2`

- [x] **Task 5.1.4**: Repeat Task 5.1.2 for `ZCL_FI_ALLOC_STEP_PHASE3`

- [x] **Task 5.1.5**: Verify the local `zcx_fi_process_error.clas.abap` in the framework
  repo has sufficient textids for all replacements. If a new textid is needed, add it.

- [x] **Task 5.1.6**: Verify that no `MESSAGE TYPE 'E'` remains in the three target methods
  after the replacements (grep the modified files).

## Dev Notes

- **Available textids** (from local `src/zcx_fi_process_error.clas.abap`, msgno 001–015):
  Verify which textid numbers map to which messages. Common useful ones:
  - `invalid_operation` (msgno=013, attr1=VALUE) — general invalid operation with message
  - Check others for more specific context (validation failure, missing parameter, etc.)
- **`value` attribute**: `ZCX_FI_PROCESS_ERROR` constructor accepts `value TYPE string`.
  Use a descriptive string that matches the original message intent.
- **Item-level log messages**: patterns like `MESSAGE e... INTO lv_dummy.` followed by
  `mo_log->log_sy_msg( )` are NOT control flow — do not replace these.
- **`lv_dummy TYPE string`**: Story 5.3 already changed `lv_dummy` to `TYPE string` in
  these files. Confirm this before touching the files.
- **Constitution Principle III**: check `ZCX_FI_PROCESS_ERROR` textids in the local
  framework source before implementing. Do not use non-existent textids.
- Reference: CC-03, `cross-cutting-analysis.md` §1.

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap` | Replace control-flow `MESSAGE TYPE 'E'` with `RAISE EXCEPTION TYPE zcx_fi_process_error` |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` | Same |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap` | Same |

## Constitution Compliance Checklist

- [x] Principle I — DDIC-First: N/A
- [x] Principle II — SAP Standards: line ≤ 120 chars (all lines comply)
- [x] Principle III — Consult SAP Docs: `ZCX_FI_PROCESS_ERROR` textids verified in local source
- [x] Principle IV — Factory Pattern: N/A
- [x] Principle V — Error Handling: `ZCX_FI_PROCESS_ERROR` used with correct textid and value

## Dev Agent Record

### Agent Model Used
claude-sonnet-4.5

### Completion Notes
Successfully replaced 9 control-flow MESSAGE statements with RAISE EXCEPTION TYPE zcx_fi_process_error:

**PHASE1 (3 replacements)**:
- Line 70: e001 → invalid_parameters (allocation not found)
- Line 76: e002 → invalid_operation (allocation locked)
- Line 85: e003 → invalid_status (non-initial state for Phase I)

**PHASE2 (3 replacements)**:
- Line 78: e001 → invalid_parameters (allocation not found)
- Line 83: e002 → invalid_operation (allocation locked)
- Line 91: e003 → invalid_status (non-initial state for Phase II)

**PHASE3 (3 replacements)**:
- Line 74: e001 → invalid_parameters (allocation not found)
- Line 79: e002 → invalid_operation (allocation locked)
- Line 84: e003 → invalid_status (non-initial state for Phase III)

**Textid Mapping**:
- e001 (allocation not found) → `invalid_parameters` (msgno 015)
- e002 (allocation locked) → `invalid_operation` (msgno 013)
- e003 (non-initial state) → `invalid_status` (msgno 011)

**Item-level logging preserved**: 18 occurrences of `MESSAGE e00X INTO lv_dummy` + `log_sy_msg()` kept unchanged per AC-03 (e005, e006, e008, e010, e011 in various CATCH blocks and validation loops).

**Verification**: grep confirmed no control-flow MESSAGE e00X remains; all remaining occurrences are item-level logging (INTO lv_dummy pattern) or commented-out code.

All acceptance criteria met:
- AC-01: All control-flow MESSAGE TYPE 'E' replaced with ZCX_FI_PROCESS_ERROR
- AC-02: No CX_SY_NO_HANDLER risk (proper exception handling with full message text)
- AC-03: Item-level log messages unchanged

Constitution compliance verified: all lines ≤ 120 chars, proper exception handling with descriptive messages.

### Changed Files
| File (remote repo) | Commit |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap` | e6f7c14 |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` | e6f7c14 |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap` | e6f7c14 |
