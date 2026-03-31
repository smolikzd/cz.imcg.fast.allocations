# Sprint 3 Quick Reference Checklist

**Total Effort**: 2.25 hours | **Stories**: 5

---

## Story 7.1: Remove ms_context (30min) ⏳

**Files to modify**: 5 step classes  
**Pattern**: Remove `DATA ms_context TYPE ...` and `ms_context = is_context` lines

- [x] INIT: Remove ms_context field and assignment
- [x] PHASE1: Remove ms_context field and assignment
- [x] PHASE2: Remove ms_context field and assignment
- [x] PHASE3: Remove ms_context field and assignment
- [x] CORR_BCHE: Remove ms_context field and assignment
- [x] Grep check: Verify no reads of ms_context remain
- [x] Commit with message: "Story 7.1: Remove dead ms_context field from all step classes"

---

## Story 7.2: Update Header Comments (15min) ⏳

**Files to modify**: 5 step classes  
**Pattern**: Replace template comments with actual descriptions (lines 1-2)

### Target Comments
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

- [x] Update INIT header
- [x] Update PHASE1 header
- [x] Update PHASE2 header
- [x] Update PHASE3 header
- [x] Update CORR_BCHE header
- [x] Commit with message: "Story 7.2: Update header comments in all step classes"

---

## Story 7.3: Delete *_ORIG Classes (30min) ⏳

**Files to delete**: 6 files (3 per class: .abap, .xml, .locals_imp.abap if exists)

### Pre-deletion Checks
- [x] Grep for `PHASE2_ORIG` references (should be 0)
- [x] Grep for `PHASE3_ORIG` references (should be 0)

### SAP System Deletion
- [x] Delete ZCL_FI_ALLOC_STEP_PHASE2_ORIG from SE24/SE80
- [x] Delete ZCL_FI_ALLOC_STEP_PHASE3_ORIG from SE24/SE80

### Repository Cleanup
- [x] Delete `zcl_fi_alloc_step_phase2_orig.clas.abap`
- [x] Delete `zcl_fi_alloc_step_phase2_orig.clas.xml`
- [x] Delete `zcl_fi_alloc_step_phase2_orig.clas.locals_imp.abap` (if exists)
- [x] Delete `zcl_fi_alloc_step_phase3_orig.clas.abap`
- [x] Delete `zcl_fi_alloc_step_phase3_orig.clas.xml`
- [x] Delete `zcl_fi_alloc_step_phase3_orig.clas.locals_imp.abap` (if exists)
- [x] Commit with message: "Story 7.3: Delete obsolete *_ORIG snapshot classes"

---

## Story 6.1: Wire PHASE3 (30min) ⏳

**File to modify**: `zfi_alloc_proc_test.clas.abap`  
**Method**: `setup_process_data`

### Tasks
- [x] Locate commented-out step 0005 block in setup_process_data
- [x] Uncomment the PHASE3 step block
- [x] Verify `substep_mode = 'SERIAL'` (not 'QUEUE')
- [x] Check parameter passing to PHASE3
- [x] **RUN END-TO-END TEST** - verify PHASE3 executes
- [x] Commit with message: "Story 6.1: Wire PHASE3 step into process definition"

### Expected Pipeline Order
```
0001 → INIT
0002 → PHASE1
0003 → PHASE2
0004 → CORR_BCHE
0005 → PHASE3 ← NOW ACTIVE
```

---

## Story 7.4: Document cleanup_old_instances (30min) ⏳

**File to modify**: `zcl_fi_process_manager.clas.abap`  
**Method**: `cleanup_old_instances`

### Decision Tree
1. **Read method**: Understand current behavior
2. **Find callers**: Grep for `cleanup_old_instances`
3. **Determine intent**:
   - Callers rely on auto-commit? → Add `COMMIT WORK AND WAIT`
   - Callers manage commits? → Add ABAP-Doc comment

### Option A: Add Commit
```abap
METHOD cleanup_old_instances.
  " ... existing DELETE logic ...
  
  COMMIT WORK AND WAIT.
ENDMETHOD.
```

### Option B: Document Caller Responsibility
```abap
"! Clean up old process instances
"! @parameter iv_days_old | Number of days threshold
"! @raising zcx_fi_process_error | Database error
"! <strong>Note:</strong> This method performs DELETE operations but does not commit.
"! Caller must issue COMMIT WORK after calling this method.
METHOD cleanup_old_instances.
  " ... existing logic ...
ENDMETHOD.
```

- [x] Read cleanup_old_instances method
- [x] Grep for all callers
- [x] Analyze caller commit patterns
- [x] Choose Option A or B
- [x] Implement chosen approach
- [x] Update callers if behavior changes
- [x] Commit with message: "Story 7.4: Document cleanup_old_instances commit behavior"

---

## Sprint Completion Checklist

### Code Quality
- [x] All 5 stories committed to remote repo
- [x] All story artifacts updated with completion notes in local repo
- [x] Constitution compliance checks pass
- [x] No new warnings introduced

### Testing
- [x] Full five-step pipeline executes end-to-end
- [x] PHASE3 processes allocation items correctly
- [x] BAL logs show entries for all five steps
- [x] Rollback scenarios tested (PHASE3 failure triggers rollback)

### Documentation
- [x] Sprint 3 marked complete in epic plan
- [x] All story artifacts have commit hashes
- [x] Sprint summary added to epic document
- [x] Migration epic closed

### Final Push
- [x] Push remote repo (cz.imcg.fast.ovysledovka)
- [x] Push local repo (cz.imcg.fast.planner)
- [x] Tag release (optional): `v1.0.0-migration-complete`

---

## Quick Commands Reference

### Grep for Dead Code
```bash
# Check ms_context is not read
grep -n "ms_context" <file> | grep -v "ms_context ="

# Check *_ORIG references
grep -rn "PHASE2_ORIG" /Users/smolik/DEV/cz.imcg.fast.ovysledovka/src/
grep -rn "PHASE3_ORIG" /Users/smolik/DEV/cz.imcg.fast.ovysledovka/src/
```

### File Paths (Remote Repo)
```
src/zfi_alloc_process/zcl_fi_alloc_step_init.clas.abap
src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap
src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap
src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap
src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap
src/zfi_alloc_process/zfi_alloc_proc_test.clas.abap
```

### File Paths (Local Repo)
```
src/zcl_fi_process_manager.clas.abap
```

---

**Ready to start?** Begin with Story 7.1 (ms_context removal) or execute stories in parallel.
