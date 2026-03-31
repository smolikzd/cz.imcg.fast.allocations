---
story: "6.1"
title: "Wire Phase 3 step into process definition"
epic: "6 — Wire Phase 3 into Pipeline"
sprint: 3
status: done
priority: moderate
estimate: 30min
source_findings:
  - CC-11
target_repo: smolikzd/cz.imcg.fast.planner
dependencies:
  - Story 1.1 (Serial stubs for PHASE3) — done in Sprint 1
  - Story 2.1 (Rollback methods) — done in Sprint 2
  - Story 4.1 (BRAND/HIER1 filter) — CANCELLED, not blocking
---

# Story 6.1: Wire Phase 3 Step into Process Definition

## User Story

As a pipeline operator,
I want the full five-step allocation pipeline to execute end-to-end,
so that Phase 3 (allocation item generation) is part of the process and not silently
skipped.

## Background

PHASE3 is currently commented out in `ZFI_ALLOC_PROC_TEST->setup_process_data`.
This was done during migration to prevent pipeline activation until all blocking
defects were resolved.

Now that:
- ✅ Story 1.1 added serial stubs to PHASE3
- ✅ Story 2.1 implemented rollback method in PHASE3
- ❌ Story 4.1 (BRAND/HIER1 filter) was cancelled (not required)

...we can safely uncomment PHASE3 and enable the full five-step pipeline:
**INIT → PHASE1 → PHASE2 → CORR_BCHE → PHASE3**

## Acceptance Criteria

**AC-01 — Step 0005 registered:**
Given the process type `ALLOCATIONS` is set up via `ZFI_ALLOC_PROC_TEST`
When `setup_process_data` runs
Then step 0005 (PHASE3, `ZCL_FI_ALLOC_STEP_PHASE3`, `substep_mode = 'SERIAL'`) is
registered as an active step

**AC-02 — PHASE3 executes:**
Given the full pipeline runs with valid parameters
When step 0004 (CORR_BCHE) completes successfully
Then step 0005 (PHASE3) is scheduled and executed

**AC-03 — End-to-end test passes:**
Given the full five-step pipeline is activated
When an end-to-end test is run with valid allocation parameters
Then all five steps execute successfully and PHASE3 generates allocation items

## Tasks

- [ ] **Task 6.1.1**: Read `ZFI_ALLOC_PROC_TEST->setup_process_data` to locate the
  commented-out step 0005 block
  
- [ ] **Task 6.1.2**: Uncomment the step 0005 (PHASE3) block

- [ ] **Task 6.1.3**: Verify `substep_mode = 'SERIAL'` is set (not 'QUEUE')

- [ ] **Task 6.1.4**: Verify step sequence number is 0005 (after 0004 CORR_BCHE)

- [ ] **Task 6.1.5**: Check if any parameters need to be passed to PHASE3 via
  `add_process_step` (review PHASE1/PHASE2 patterns)

- [ ] **Task 6.1.6**: Run end-to-end allocation test and verify PHASE3 executes

- [ ] **Task 6.1.7**: Check BAL log (SLG1) to confirm PHASE3 log entries exist

- [ ] **Task 6.1.8**: Verify PHASE3 generates expected allocation items in target tables

## Dev Notes

- **Prerequisites verified**: Sprint 1 & 2 complete, PHASE3 has serial stubs and rollback
- **Expected pattern**: Look for commented block similar to:
  ```abap
  " Step 0005: PHASE3
  " mo_manager->add_process_step(
  "   iv_step_number  = '0005'
  "   iv_step_class   = 'ZCL_FI_ALLOC_STEP_PHASE3'
  "   iv_substep_mode = 'SERIAL'
  " ).
  ```
- **Substep mode**: Must be 'SERIAL' (PHASE3 does not support QUEUE mode in this sprint)
- **Testing is MANDATORY**: This story changes pipeline behavior. End-to-end test required
  before marking complete.
- **Risk**: Moderate — enabling previously disabled step requires verification
- Reference: CC-11, Epic 6 in `epics-alloc-remediation.md`

## Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zfi_alloc_proc_test.clas.abap` | Uncomment step 0005 (PHASE3) block in `setup_process_data` method |

## Constitution Compliance Checklist

- [x] Principle I — DDIC-First: No new types created
- [x] Principle II — SAP Standards: Uncommenting existing code, line length already compliant
- [x] Principle III — Consult SAP Docs: Not applicable (configuration change)
- [x] Principle IV — Factory Pattern: Not applicable
- [x] Principle V — Error Handling: Not applicable (step registration only)

## Testing Requirements

### Unit Test
- Verify step 0005 is registered in process definition
- Verify substep_mode = 'SERIAL'

### Integration Test (MANDATORY)
- Run full allocation process with valid parameters
- Verify all 5 steps execute in order: INIT → PHASE1 → PHASE2 → CORR_BCHE → PHASE3
- Check BAL logs for all steps
- Verify PHASE3 generates allocation items
- Test rollback: trigger error in PHASE3, verify rollback cascade

### Expected Pipeline Flow
```
Step 0001: INIT        ✓ Initialize and validate
Step 0002: PHASE1      ✓ Fetch source documents
Step 0003: PHASE2      ✓ Calculate allocations
Step 0004: CORR_BCHE   ✓ Correct batch headers
Step 0005: PHASE3      ✓ Generate allocation items ← NOW ACTIVE
```

## Dev Agent Record

### Agent Model Used

### Completion Notes

### Changed Files
| File (remote repo) | Commit |
|--------------------|--------|
