---
story: "5.3"
title: "Fix lv_dummy TYPE c to TYPE string in PHASE1, PHASE2, PHASE3"
epic: "5 — Error Handling and Observability"
sprint: 1
status: done
priority: moderate
estimate: 30min
source_findings:
  - CC-06
  - cross-cutting-analysis.md
target_repo: smolikzd/cz.imcg.fast.planner
dependencies:
  - None (independent; Story 3.3 touches PHASE2 CATCH blocks and should be done together)
---

# Story 5.3: Fix `lv_dummy TYPE c` to `TYPE string` in PHASE1, PHASE2, PHASE3

## User Story

As a process monitor user,
I want error messages to contain their full text,
so that the message is readable rather than truncated to a single character.

## Background

`MESSAGE ... INTO lv_dummy` is used in PHASE1, PHASE2, and PHASE3 to capture
message text after a runtime error or CATCH block. The variable is declared as:

```abap
DATA lv_dummy TYPE c.
```

`TYPE c` without a length specification defaults to length 1. Any message text
longer than one character is silently truncated. `rs_result-message = lv_dummy`
then sets a one-character result message — invisible to operators in the process
monitor.

The fix is a single-line change per file: `TYPE c` → `TYPE string`.

## Acceptance Criteria

**AC-01 — Full message text captured:**
Given a `MESSAGE ... INTO lv_dummy` is executed during step processing
When the message text is longer than 1 character
Then `lv_dummy` holds the full message text (not just the first character)

**AC-02 — Process monitor shows full message:**
Given `rs_result-message = lv_dummy` is set after an error catch
When the process monitor displays the step result message
Then the full message text is visible

**AC-03 — No other TYPE c sinks:**
Given the three files are reviewed
When the review is complete
Then no other variable of `TYPE c` (without explicit length) is used as a
`MESSAGE ... INTO` sink in these three files

## Tasks

- [ ] **Task 5.3.1**: In `ZCL_FI_ALLOC_STEP_PHASE1`:
  - [ ] Find `DATA lv_dummy TYPE c` (or equivalent)
  - [ ] Change to `DATA lv_dummy TYPE string`
  - [ ] Verify all `MESSAGE ... INTO lv_dummy` usages still compile

- [ ] **Task 5.3.2**: In `ZCL_FI_ALLOC_STEP_PHASE2`:
  - [ ] Same as 5.3.1

- [ ] **Task 5.3.3**: In `ZCL_FI_ALLOC_STEP_PHASE3`:
  - [ ] Same as 5.3.1

- [ ] **Task 5.3.4**: In each of the three files, scan for any other
  `DATA ... TYPE c.` (length-1) variables used as `MESSAGE ... INTO` sinks and
  fix those too.

## Dev Notes

- This is a one-line change per file. Low risk, high value.
- `MESSAGE ... INTO lv_dummy` with `TYPE string` is fully supported in ABAP 7.58.
- If doing Story 3.3 (CATCH block fix) in the same session on PHASE2, coordinate
  so both changes are included in the same commit.
- Line length ≤ 120 chars (no impact — single-word change).

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap` | `TYPE c` → `TYPE string` for `lv_dummy` |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` | `TYPE c` → `TYPE string` for `lv_dummy` |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap` | `TYPE c` → `TYPE string` for `lv_dummy` |

## Constitution Compliance Checklist

- [ ] Principle I — DDIC-First: no new type definitions
- [ ] Principle II — SAP Standards: line ≤ 120 chars
- [ ] Principle III — Consult SAP Docs: N/A (standard ABAP type)
- [ ] Principle IV — Factory Pattern: N/A
- [ ] Principle V — Error Handling: improves error message visibility

## Dev Agent Record

### Agent Model Used
github-copilot/claude-sonnet-4.6

### Completion Notes
PHASE3 fix was bundled with Story 1.1 PHASE3 commit. PHASE1 and PHASE2 fixed
separately. PHASE2 has two lv_dummy declarations — one in `execute` method and one
in `execute_substep` method — both changed to TYPE string.

### Changed Files
| File (remote repo) | Commit |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap` | `28786117` (bundled with 1.1) |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap` | `829c5fa6` |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` | `82fc4d28` |
