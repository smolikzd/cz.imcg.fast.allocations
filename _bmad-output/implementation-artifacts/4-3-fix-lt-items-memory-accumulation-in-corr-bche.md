---
story: "4.3"
title: "Fix lt_items memory accumulation in CORR_BCHE"
epic: "4 — Functional Correctness — Data Integrity"
sprint: 1
status: done
priority: critical
estimate: 15min
source_findings:
  - CC-14
  - cross-cutting-analysis.md
  - zfi_alloc_corr_bche-migration-review.md
target_repo: smolikzd/cz.imcg.fast.planner
dependencies:
  - None (independent)
---

# Story 4.3: Fix `lt_items` memory accumulation in CORR_BCHE

## User Story

As a production system running CORR_BCHE over a large header set,
I want the item table to be cleared between loop iterations,
so that application server memory is not exhausted by accumulating all items
across all headers simultaneously.

## Background

`ZCL_FI_ALLOC_STEP_CORR_BCHE->execute` processes allocation headers in a
`LOOP AT lt_headers`. Inside the loop, items are selected into `lt_items`:

```abap
LOOP AT lt_headers INTO ls_header.
  SELECT * FROM zfi_alloc_bcitm
    INTO TABLE lt_items           " ← never cleared!
    WHERE ...
  " process lt_items
ENDLOOP.
```

Because `lt_items` is never cleared before each iteration, rows from every
previous header accumulate in the table. On header N, `lt_items` contains
all rows from headers 1 through N. This grows quadratically and will exhaust
work process memory on any reasonably-sized dataset.

## Acceptance Criteria

**AC-01 — Items table scoped to current header:**
Given the inner loop processes N headers sequentially
When each header's item SELECT runs
Then `lt_items` contains only the items for the current header (not items from
all previous headers combined)

**AC-02 — Correct item count used in UPDATE:**
Given `lt_items` is correctly scoped
When `no_items` is calculated from `lt_items`
Then it reflects only the item count for the current header

## Tasks

- [ ] **Task 4.3.1**: In the `LOOP AT lt_headers` block in
  `ZCL_FI_ALLOC_STEP_CORR_BCHE->execute`:

  **Option A** (preferred — inline declaration, cleanest):
  ```abap
  LOOP AT lt_headers INTO ls_header.
    SELECT * FROM zfi_alloc_bcitm
      INTO TABLE @DATA(lt_items)   " fresh table each iteration
      WHERE ...
  ```

  **Option B** (if inline declaration cannot be used in this context):
  ```abap
  LOOP AT lt_headers INTO ls_header.
    CLEAR lt_items.
    SELECT * FROM zfi_alloc_bcitm
      INTO TABLE lt_items
      WHERE ...
  ```

  Prefer Option A — verify abaplint configuration accepts inline `@DATA` in
  this SELECT context. If accepted, remove the outer `DATA lt_items` declaration.

## Dev Notes

- Using inline `@DATA(lt_items)` (Option A) is the cleaner ABAP 7.4+ approach
  and prevents the bug from recurring — the table cannot accidentally accumulate
  from a previous iteration.
- If choosing Option A: delete the `DATA lt_items TYPE ...` declaration at the
  top of the method (or the class's local section) to avoid "duplicate symbol" error.
- If choosing Option B: add `CLEAR lt_items.` as the **first** statement inside
  the loop, before the SELECT.
- Line length ≤ 120 chars.

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap` | Clear `lt_items` before each SELECT in loop |

## Constitution Compliance Checklist

- [ ] Principle I — DDIC-First: no new type definitions introduced
- [ ] Principle II — SAP Standards: line ≤ 120 chars
- [ ] Principle III — Consult SAP Docs: N/A (standard ABAP table handling)
- [ ] Principle IV — Factory Pattern: N/A
- [ ] Principle V — Error Handling: N/A (data scoping fix)

## Dev Agent Record

### Agent Model Used
github-copilot/claude-sonnet-4.6

### Completion Notes
Bundled into Story 1.1 CORR_BCHE commit. Used Option A (inline @DATA(lt_items)) —
removed outer DATA declaration and replaced with inline `INTO TABLE @DATA(lt_items)`.

### Changed Files
| File | Commit |
|------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap` | `3c75fc3e` |
