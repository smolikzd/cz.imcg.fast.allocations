---
title: "Sprint 3 — Wire Phase 3 + Housekeeping"
sprint: 3
status: done
total_stories: 5
total_effort: 2.25h
created: 2026-03-09
dependencies:
  - Sprint 1 (complete)
  - Sprint 2 (complete)
---

# Sprint 3 Plan — Wire Phase 3 + Housekeeping

## Sprint Overview

**Goal**: Complete the ZFI_ALLOC_PROCESS migration by wiring Phase 3 into the pipeline and performing code quality housekeeping.

**Status**: Ready for development  
**Total Stories**: 5  
**Estimated Effort**: 2.25 hours  
**Priority**: Low to Moderate (no critical defects)

### Prerequisites (All Complete ✅)
- ✅ Sprint 1: Pipeline activation blockers resolved
- ✅ Sprint 2: Structural correctness issues fixed
- ✅ Story 1.1: Serial stubs added to all steps (including PHASE3)
- ✅ Story 2.1: Rollback methods implemented
- ❌ Story 4.1: BRAND/HIER1 filter (CANCELLED - not required)

**Note**: Story 4.1 was cancelled, but this does not block Story 6.1. The PHASE3 step can be wired without the BRAND/HIER1 filter restoration.

---

## Story Execution Order

### Recommended Sequence

1. **Story 7.1** (30min) — Remove ms_context dead field  
   *Rationale*: Simple cleanup, no dependencies, sets good foundation

2. **Story 7.2** (15min) — Update header comments  
   *Rationale*: Cosmetic, no dependencies, quick win

3. **Story 7.3** (30min) — Delete *_ORIG classes  
   *Rationale*: Cleanup before final wiring, prevents confusion

4. **Story 6.1** (30min) — Wire PHASE3 into pipeline  
   *Rationale*: Core functionality, do after housekeeping complete

5. **Story 7.4** (30min) — Document cleanup_old_instances commit  
   *Rationale*: Framework-level documentation, independent

**Parallel Option**: Stories 7.1, 7.2, 7.3, and 7.4 have no dependencies and can be executed in any order or in parallel if desired.

---

## Story 6.1: Wire Phase 3 into Pipeline

**Epic**: 6 — Wire Phase 3 into Pipeline  
**Priority**: Moderate  
**Effort**: 30 minutes  
**Finding**: CC-11

### User Story
As a pipeline operator,  
I want the full five-step allocation pipeline to execute end-to-end,  
so that Phase 3 (allocation item generation) is part of the process and not silently skipped.

### Background
PHASE3 is currently commented out in `ZFI_ALLOC_PROC_TEST->setup_process_data`.
This story uncomments it, enabling the full five-step pipeline:
- INIT → PHASE1 → PHASE2 → CORR_BCHE → **PHASE3**

### Acceptance Criteria

**AC-01 — Step 0005 registered:**  
Given the process type `ALLOCATIONS` is set up via `ZFI_ALLOC_PROC_TEST`  
When `setup_process_data` runs  
Then step 0005 (PHASE3, `ZCL_FI_ALLOC_STEP_PHASE3`, `substep_mode = 'SERIAL'`) is registered as an active step

**AC-02 — PHASE3 executes:**  
Given the full pipeline runs with valid parameters  
When step 0004 (CORR_BCHE) completes successfully  
Then step 0005 (PHASE3) is scheduled and executed

### Tasks

- [ ] **Task 6.1.1**: Locate the commented-out step 0005 block in `ZFI_ALLOC_PROC_TEST->setup_process_data`
- [ ] **Task 6.1.2**: Uncomment the step 0005 block
- [ ] **Task 6.1.3**: Verify `substep_mode = 'SERIAL'` is set (not 'QUEUE')
- [ ] **Task 6.1.4**: Check if any additional parameters need to be passed to PHASE3
- [ ] **Task 6.1.5**: Run end-to-end test and confirm PHASE3 executes and completes

### Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zfi_alloc_proc_test.clas.abap` | Uncomment step 0005 (PHASE3) block in `setup_process_data` method |

### Constitution Compliance
- Principle I — DDIC-First: No new types created
- Principle II — SAP Standards: Uncommenting code, line length already compliant
- Principle III — Consult SAP Docs: Not applicable (configuration change)
- Principle IV — Factory Pattern: Not applicable
- Principle V — Error Handling: Not applicable

### Dev Notes
- **Prerequisites verified**: Sprint 1 & 2 complete, PHASE3 has serial stubs and rollback
- **Risk**: Low — only uncommenting existing, tested code
- **Testing**: Should run end-to-end test to verify PHASE3 executes successfully

---

## Story 7.1: Remove ms_context Dead Field

**Epic**: 7 — Code Quality and Housekeeping  
**Priority**: Low  
**Effort**: 30 minutes  
**Finding**: CC-07

### User Story
As a developer maintaining step classes,  
I want dead private fields removed,  
so that the class declaration accurately reflects the data actually used.

### Background
All five step classes have a `ms_context` field in PRIVATE SECTION that is written in `init` but never read subsequently. This is dead code from template generation.

### Acceptance Criteria

**AC-01 — ms_context removed:**  
Given any of the five step classes  
When a code review is performed  
Then no `ms_context` field exists in the PRIVATE SECTION

### Tasks

- [ ] **Task 7.1.1**: Grep each class to confirm `ms_context` is not read (only written)
- [ ] **Task 7.1.2**: Remove `ms_context` from PRIVATE SECTION in `ZCL_FI_ALLOC_STEP_INIT`
- [ ] **Task 7.1.3**: Remove `ms_context = is_context` assignment from `init` in INIT
- [ ] **Task 7.1.4**: Repeat for `ZCL_FI_ALLOC_STEP_PHASE1`
- [ ] **Task 7.1.5**: Repeat for `ZCL_FI_ALLOC_STEP_PHASE2`
- [ ] **Task 7.1.6**: Repeat for `ZCL_FI_ALLOC_STEP_PHASE3`
- [ ] **Task 7.1.7**: Repeat for `ZCL_FI_ALLOC_STEP_CORR_BCHE`

### Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_init.clas.abap` | Remove `ms_context` field and assignment |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap` | Remove `ms_context` field and assignment |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` | Remove `ms_context` field and assignment |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap` | Remove `ms_context` field and assignment |
| `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap` | Remove `ms_context` field and assignment |

### Constitution Compliance
- Principle I — DDIC-First: Removing dead code, no type changes
- Principle II — SAP Standards: Code removal only
- Principle III — Consult SAP Docs: Not applicable
- Principle IV — Factory Pattern: Not applicable
- Principle V — Error Handling: Not applicable

### Dev Notes
- **Safety check**: Always grep for reads before removing to ensure it's truly dead
- **Pattern**: Look for `DATA ms_context TYPE zif_fi_process_step=>ts_context.` and `ms_context = is_context.`
- **Risk**: Very low — removing unused field

---

## Story 7.2: Update Header Comments

**Epic**: 7 — Code Quality and Housekeeping  
**Priority**: Low  
**Effort**: 15 minutes  
**Finding**: CC-08

### User Story
As a developer reading step class source,  
I want the file header comment to describe the actual class,  
so that I understand the class purpose from the first line rather than reading a placeholder template comment.

### Background
All step classes have template header comments like:
```abap
*& Class ZCL_STEP_FETCH_DATA
*& Example step: Fetch data from database
```
These should be replaced with actual class names and descriptions.

### Acceptance Criteria

**AC-01 — Header describes actual class:**  
Given any of the five step classes  
When the source is opened  
Then the first two comment lines describe the actual class purpose

### Tasks

- [ ] **Task 7.2.1**: Update header comment in `ZCL_FI_ALLOC_STEP_INIT` to describe INIT step
- [ ] **Task 7.2.2**: Update header comment in `ZCL_FI_ALLOC_STEP_PHASE1` to describe PHASE1
- [ ] **Task 7.2.3**: Update header comment in `ZCL_FI_ALLOC_STEP_PHASE2` to describe PHASE2
- [ ] **Task 7.2.4**: Update header comment in `ZCL_FI_ALLOC_STEP_PHASE3` to describe PHASE3
- [ ] **Task 7.2.5**: Update header comment in `ZCL_FI_ALLOC_STEP_CORR_BCHE` to describe CORR_BCHE

### Suggested Header Comments

```abap
" INIT:
*& Class ZCL_FI_ALLOC_STEP_INIT
*& Allocation process initialization: validate parameters and create state row

" PHASE1:
*& Class ZCL_FI_ALLOC_STEP_PHASE1
*& Phase 1: Fetch source documents and prepare allocation base

" PHASE2:
*& Class ZCL_FI_ALLOC_STEP_PHASE2
*& Phase 2: Calculate allocation amounts and update targets

" PHASE3:
*& Class ZCL_FI_ALLOC_STEP_PHASE3
*& Phase 3: Generate allocation line items

" CORR_BCHE:
*& Class ZCL_FI_ALLOC_STEP_CORR_BCHE
*& Correction step: Update batch header flags based on item existence
```

### Files to Change

| File (remote repo) | Change |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_init.clas.abap` | Update header comment lines 1-2 |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap` | Update header comment lines 1-2 |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` | Update header comment lines 1-2 |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap` | Update header comment lines 1-2 |
| `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap` | Update header comment lines 1-2 |

### Constitution Compliance
- Principle I — DDIC-First: Comment changes only
- Principle II — SAP Standards: Following ABAP comment style
- Principle III — Consult SAP Docs: Not applicable
- Principle IV — Factory Pattern: Not applicable
- Principle V — Error Handling: Not applicable

### Dev Notes
- **Quick win**: Two lines per file, pure documentation
- **Risk**: Zero — comment-only change

---

## Story 7.3: Delete *_ORIG Classes

**Epic**: 7 — Code Quality and Housekeeping  
**Priority**: Low  
**Effort**: 30 minutes  
**Finding**: CC-10

### User Story
As a developer working in the `ZFI_ALLOC_PROCESS` package,  
I want obsolete snapshot classes removed,  
so that there is no namespace confusion and the package contains only live objects.

### Background
The repository contains dead snapshot classes:
- `ZCL_FI_ALLOC_STEP_PHASE2_ORIG`
- `ZCL_FI_ALLOC_STEP_PHASE3_ORIG`

These are pre-migration backups that are no longer needed.

### Acceptance Criteria

**AC-01 — ORIG classes removed from SAP system:**  
Given the SAP system  
When cleanup is complete  
Then `ZCL_FI_ALLOC_STEP_PHASE2_ORIG` and `ZCL_FI_ALLOC_STEP_PHASE3_ORIG` do not exist as active objects

**AC-02 — ORIG files removed from repository:**  
Given the git repository  
When cleanup is complete  
Then `zcl_fi_alloc_step_phase2_orig.clas.*` and `zcl_fi_alloc_step_phase3_orig.clas.*` files are deleted

### Tasks

- [ ] **Task 7.3.1**: Grep repository for any references to `PHASE2_ORIG` (should be zero)
- [ ] **Task 7.3.2**: Grep repository for any references to `PHASE3_ORIG` (should be zero)
- [ ] **Task 7.3.3**: Delete `ZCL_FI_ALLOC_STEP_PHASE2_ORIG` from SAP system (SE24 or SE80)
- [ ] **Task 7.3.4**: Delete `ZCL_FI_ALLOC_STEP_PHASE3_ORIG` from SAP system
- [ ] **Task 7.3.5**: Remove corresponding `.clas.abap`, `.clas.xml`, `.clas.locals_imp.abap` files from `src/zfi_alloc_process/`
- [ ] **Task 7.3.6**: Commit deletion to git with message explaining cleanup

### Files to Delete

| File (remote repo) | Action |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2_orig.clas.abap` | Delete |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2_orig.clas.xml` | Delete |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2_orig.clas.locals_imp.abap` | Delete (if exists) |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3_orig.clas.abap` | Delete |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3_orig.clas.xml` | Delete |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3_orig.clas.locals_imp.abap` | Delete (if exists) |

### Constitution Compliance
- Not applicable (deletion only)

### Dev Notes
- **Safety check**: Always grep for references before deleting
- **SAP system cleanup**: Use SE24 or SE80 to delete class objects
- **Git cleanup**: Use `git rm` to stage deletions properly
- **Risk**: Very low — dead code removal

---

## Story 7.4: Document cleanup_old_instances Commit

**Epic**: 7 — Code Quality and Housekeeping  
**Priority**: Low  
**Effort**: 30 minutes  
**Finding**: LUW-03

### User Story
As a framework developer,  
I want the commit behaviour of `cleanup_old_instances` to be explicit and documented,  
so that callers know whether they must issue a `COMMIT WORK` after calling it.

### Background
`ZCL_FI_PROCESS_MANAGER->cleanup_old_instances` performs DELETE operations but does not explicitly commit or document whether the caller must commit.

This ambiguity can lead to:
- Uncommitted deletes (if caller assumes auto-commit)
- Double commits (if caller commits unnecessarily)

### Acceptance Criteria

**AC-01 — Commit behavior explicit:**  
Given `ZCL_FI_PROCESS_MANAGER->cleanup_old_instances` is called  
When the method completes  
Then either:
- A `COMMIT WORK AND WAIT` is added at the end (if auto-commit is intended), OR
- An ABAP-Doc comment clearly states "Caller must commit" (if deferred commit is intended)

### Tasks

- [ ] **Task 7.4.1**: Read `ZCL_FI_PROCESS_MANAGER->cleanup_old_instances` method
- [ ] **Task 7.4.2**: Identify all callers of this method (grep for `cleanup_old_instances`)
- [ ] **Task 7.4.3**: Determine design intent:
  - If callers rely on auto-commit → add `COMMIT WORK AND WAIT` at method end
  - If callers intentionally manage commits → add ABAP-Doc comment
- [ ] **Task 7.4.4**: Implement the chosen approach (commit or documentation)
- [ ] **Task 7.4.5**: Update any callers if behavior changes

### Files to Change

| File (local repo) | Change |
|-------------------|--------|
| `src/zcl_fi_process_manager.clas.abap` | Add `COMMIT WORK AND WAIT` or ABAP-Doc comment to `cleanup_old_instances` |

### Constitution Compliance
- Principle I — DDIC-First: Not applicable
- Principle II — SAP Standards: ABAP-Doc format
- Principle III — Consult SAP Docs: Review COMMIT WORK best practices
- Principle IV — Factory Pattern: Not applicable
- Principle V — Error Handling: Document error handling if commit added

### Dev Notes
- **Design decision**: Prefer explicit documentation over implicit behavior
- **LUW best practice**: Framework methods that modify data should either commit or clearly document caller responsibility
- **Reference**: LUW-03, `luw-schema-analysis.md` §4.2
- **Risk**: Low — documentation or explicit commit clarification

---

## Sprint Success Criteria

### All Stories Complete
- [x] Story 6.1 — PHASE3 wired into pipeline
- [x] Story 7.1 — ms_context removed from all step classes
- [x] Story 7.2 — Header comments updated in all step classes
- [x] Story 7.3 — *_ORIG classes deleted from system and repository
- [x] Story 7.4 — cleanup_old_instances commit behavior documented

### Integration Tests Pass
- [ ] Full five-step pipeline (INIT → PHASE1 → PHASE2 → CORR_BCHE → PHASE3) executes successfully
- [ ] All BAL logs show expected entries for all steps
- [ ] Rollback scenarios tested (if Phase 3 fails, earlier steps roll back)

### Code Quality Verified
- [ ] No dead code (ms_context, *_ORIG classes) remains
- [ ] All header comments accurately describe classes
- [ ] Constitution compliance checks pass with no new violations

---

## Risk Assessment

### Low Risk Stories
- ✅ Story 7.1 (ms_context removal) — Dead code, no functional impact
- ✅ Story 7.2 (header comments) — Documentation only, zero risk
- ✅ Story 7.3 (*_ORIG deletion) — Dead classes, no references

### Moderate Risk Stories
- ⚠️ Story 6.1 (wire PHASE3) — Enables new step, requires end-to-end testing
- ⚠️ Story 7.4 (cleanup commit) — May change LUW behavior if commit added

### Mitigation Strategies
- **Story 6.1**: Run full end-to-end test before marking complete
- **Story 7.4**: Review all callers carefully before deciding commit strategy

---

## Post-Sprint Activities

### After Sprint 3 Complete
1. **Full regression test**: Run complete allocation process with production-like data
2. **Performance benchmark**: Compare execution times pre/post migration
3. **Documentation update**: Update process flow diagrams to show all five steps
4. **Stakeholder demo**: Show full pipeline execution with Phase 3 active
5. **Close migration epic**: Mark ZFI_ALLOC_PROCESS migration as complete

### Known Remaining Items (Out of Scope)
- Story 4.1 (BRAND/HIER1 filter) — Cancelled, filter logic not required per product owner
- Performance optimization — To be addressed in separate performance epic if needed
- Additional BAL log enhancements — May be added as separate stories based on operational feedback

---

## Dev Agent Notes

### Execution Strategy
1. Start with housekeeping (Stories 7.1, 7.2, 7.3) to clean up codebase
2. Wire PHASE3 (Story 6.1) after code is clean
3. Document framework behavior (Story 7.4) last

### Testing Checklist
- [ ] Unit test each story individually
- [ ] Integration test after Story 6.1 (full pipeline)
- [ ] Regression test all previously completed stories
- [ ] Constitution compliance check before each commit

### Commit Message Pattern
```
Story X.Y: <Short description>

- Change 1
- Change 2
- ...

Acceptance criteria met:
- AC-01: <description>
- AC-02: <description>
```

---

**Document Status**: Ready for implementation  
**Last Updated**: 2026-03-09  
**Next Review**: After Sprint 3 completion
