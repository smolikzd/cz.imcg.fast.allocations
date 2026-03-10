# Sprint 3 Planning Complete! 

## Overview

**Sprint 3: Wire Phase 3 + Housekeeping**  
**Status**: ✅ Ready for Development  
**Total Stories**: 5  
**Estimated Effort**: 2.25 hours  

---

## Planning Documents Created

### 📋 Main Planning Document
**Location**: `_bmad-output/planning-artifacts/sprint-3-plan.md`

**Contents**:
- Detailed story descriptions with background and rationale
- Full acceptance criteria for each story
- Task breakdowns with step-by-step instructions
- Files to change/delete for each story
- Constitution compliance checklists
- Risk assessment and mitigation strategies
- Post-sprint activities and success criteria

### ✅ Quick Reference Checklist
**Location**: `_bmad-output/planning-artifacts/sprint-3-checklist.md`

**Contents**:
- One-page checklist for each story
- Copy-paste code patterns
- Command references for common operations
- File paths for all modified files
- Quick decision tree for Story 7.4

---

## Sprint 3 Stories

### Story Execution Order (Recommended)

```
1️⃣  Story 7.1 (30min) — Remove ms_context dead field
    ├─ 5 files to modify (all step classes)
    └─ Simple: Remove field declaration and assignment

2️⃣  Story 7.2 (15min) — Update header comments
    ├─ 5 files to modify (all step classes)
    └─ Simple: Replace lines 1-2 with actual descriptions

3️⃣  Story 7.3 (30min) — Delete *_ORIG classes
    ├─ 6 files to delete
    ├─ 2 classes to delete from SAP system
    └─ Medium: Requires SAP system access + git cleanup

4️⃣  Story 6.1 (30min) — Wire PHASE3 into pipeline ⚠️ KEY STORY
    ├─ 1 file to modify (zfi_alloc_proc_test)
    ├─ Uncomment step 0005 block
    └─ REQUIRES: End-to-end test before completion

5️⃣  Story 7.4 (30min) — Document cleanup_old_instances
    ├─ 1 file to modify (zcl_fi_process_manager)
    └─ Decision: Add commit or document caller responsibility
```

### Parallelization Option

Stories 7.1, 7.2, 7.3, and 7.4 have **no dependencies** and can be executed in **any order** or **in parallel**.

Only Story 6.1 should be done **after housekeeping** (Stories 7.x) to ensure clean codebase before wiring Phase 3.

---

## Story Details Summary

### 6.1 — Wire PHASE3 (30min) 🎯 CRITICAL
**Goal**: Enable full five-step pipeline  
**Change**: Uncomment step 0005 in `setup_process_data`  
**Risk**: ⚠️ Moderate — requires end-to-end test  
**Impact**: HIGH — completes pipeline activation  

**Before**:
```
INIT → PHASE1 → PHASE2 → CORR_BCHE
```

**After**:
```
INIT → PHASE1 → PHASE2 → CORR_BCHE → PHASE3 ✨
```

---

### 7.1 — Remove ms_context (30min) 🧹
**Goal**: Remove dead field from all step classes  
**Change**: Delete `DATA ms_context TYPE ...` and assignment  
**Risk**: ✅ Very Low — unused field  
**Impact**: Code cleanliness  

**Files**: All 5 step classes (INIT, PHASE1, PHASE2, PHASE3, CORR_BCHE)

---

### 7.2 — Update Header Comments (15min) 📝
**Goal**: Replace template comments with actual descriptions  
**Change**: Update lines 1-2 in each step class header  
**Risk**: ✅ Zero — documentation only  
**Impact**: Developer experience  

**Example**:
```abap
" BEFORE:
*& Class ZCL_STEP_FETCH_DATA
*& Example step: Fetch data from database

" AFTER:
*& Class ZCL_FI_ALLOC_STEP_PHASE1
*& Phase 1: Fetch source documents and prepare allocation base
```

---

### 7.3 — Delete *_ORIG Classes (30min) 🗑️
**Goal**: Remove obsolete snapshot classes  
**Delete**:
- `ZCL_FI_ALLOC_STEP_PHASE2_ORIG` (SAP system + 3 files)
- `ZCL_FI_ALLOC_STEP_PHASE3_ORIG` (SAP system + 3 files)

**Risk**: ✅ Very Low — dead code  
**Impact**: Namespace cleanliness  

---

### 7.4 — Document cleanup_old_instances (30min) 📚
**Goal**: Clarify commit behavior of framework method  
**Change**: Add `COMMIT WORK` or ABAP-Doc comment  
**Risk**: ⚠️ Moderate — may change LUW behavior  
**Impact**: Framework clarity  

**Decision Tree**:
1. Read method implementation
2. Find all callers (grep)
3. Analyze commit patterns
4. Choose: Auto-commit OR Document caller responsibility

---

## Prerequisites Check ✅

All Sprint 3 prerequisites are **COMPLETE**:

- ✅ **Sprint 1** — Pipeline activation blockers resolved (6 stories)
- ✅ **Sprint 2** — Structural correctness fixed (7 stories)
- ✅ **Story 1.1** — Serial stubs added to PHASE3
- ✅ **Story 2.1** — Rollback methods implemented
- ❌ **Story 4.1** — CANCELLED (BRAND/HIER1 filter not required)

**Ready to proceed!**

---

## Success Criteria

### Code Quality
- [ ] All 5 stories completed and committed
- [ ] Constitution compliance checks pass
- [ ] No dead code remains (ms_context, *_ORIG)
- [ ] All header comments accurate

### Functionality
- [ ] Full five-step pipeline executes end-to-end
- [ ] PHASE3 processes allocation items correctly
- [ ] BAL logs show entries for all five steps
- [ ] Rollback scenarios tested

### Documentation
- [ ] All story artifacts updated with completion notes
- [ ] Sprint 3 marked complete in epic plan
- [ ] Migration epic closed

---

## Risk Assessment

### Low Risk Stories (Can execute confidently)
- ✅ Story 7.1 — Dead code removal
- ✅ Story 7.2 — Documentation only
- ✅ Story 7.3 — Dead class deletion

### Moderate Risk Stories (Require testing)
- ⚠️ Story 6.1 — Enables new step, **MUST run end-to-end test**
- ⚠️ Story 7.4 — May change LUW behavior, **review callers carefully**

---

## Testing Strategy

### Per-Story Testing
- **Story 7.1, 7.2, 7.3**: Syntax check only (no functional change)
- **Story 6.1**: **MANDATORY** end-to-end test with all 5 steps
- **Story 7.4**: Unit test cleanup_old_instances if commit added

### Integration Testing (After Story 6.1)
1. **Full pipeline test**: INIT → PHASE1 → PHASE2 → CORR_BCHE → PHASE3
2. **BAL log verification**: Check SLG1 for entries from all steps
3. **Rollback test**: Trigger PHASE3 error, verify rollback cascade
4. **Performance check**: Compare execution time vs. 4-step pipeline

---

## File Change Summary

### Files to Modify (9 files)
| File | Stories | Changes |
|------|---------|---------|
| `zcl_fi_alloc_step_init.clas.abap` | 7.1, 7.2 | Remove ms_context, update header |
| `zcl_fi_alloc_step_phase1.clas.abap` | 7.1, 7.2 | Remove ms_context, update header |
| `zcl_fi_alloc_step_phase2.clas.abap` | 7.1, 7.2 | Remove ms_context, update header |
| `zcl_fi_alloc_step_phase3.clas.abap` | 7.1, 7.2 | Remove ms_context, update header |
| `zcl_fi_alloc_step_corr_bche.clas.abap` | 7.1, 7.2 | Remove ms_context, update header |
| `zfi_alloc_proc_test.clas.abap` | 6.1 | Uncomment PHASE3 block |
| `zcl_fi_process_manager.clas.abap` | 7.4 | Add commit or ABAP-Doc |

### Files to Delete (6 files)
- `zcl_fi_alloc_step_phase2_orig.clas.abap`
- `zcl_fi_alloc_step_phase2_orig.clas.xml`
- `zcl_fi_alloc_step_phase2_orig.clas.locals_imp.abap` (if exists)
- `zcl_fi_alloc_step_phase3_orig.clas.abap`
- `zcl_fi_alloc_step_phase3_orig.clas.xml`
- `zcl_fi_alloc_step_phase3_orig.clas.locals_imp.abap` (if exists)

---

## Next Steps

### Ready to Start Development?

**Option 1: Sequential Execution** (Recommended for single developer)
```
Start with Story 7.1 → 7.2 → 7.3 → 6.1 → 7.4
```

**Option 2: Parallel Execution** (If multiple developers or batch processing)
```
Execute Stories 7.1, 7.2, 7.3, 7.4 in parallel
Then execute Story 6.1 after housekeeping complete
```

**Option 3: Critical Path First** (If PHASE3 activation is urgent)
```
Execute Story 6.1 immediately
Then clean up with Stories 7.1, 7.2, 7.3, 7.4
```

### Commands to Begin

```bash
# View full plan
cat /Users/smolik/DEV/cz.imcg.fast.planner/_bmad-output/planning-artifacts/sprint-3-plan.md

# View checklist
cat /Users/smolik/DEV/cz.imcg.fast.planner/_bmad-output/planning-artifacts/sprint-3-checklist.md

# Start development (example: Story 7.1)
# Say: "Start Story 7.1" or "Continue with next story"
```

---

## Planning Artifacts Location

All planning documents are committed and pushed to:
- **Repository**: `smolikzd/cz.imcg.fast.planner` (local repo)
- **Branch**: `main`
- **Directory**: `_bmad-output/planning-artifacts/`

**Files**:
1. `epics-alloc-remediation.md` — Master epic plan (updated with Sprint 3)
2. `sprint-3-plan.md` — Detailed Sprint 3 plan (NEW)
3. `sprint-3-checklist.md` — Quick reference checklist (NEW)

---

## Project Status After Planning

### Completed
- ✅ Sprint 1: 6 stories, 3.75h
- ✅ Sprint 2: 7 stories, 11h
- ✅ Sprint 3: **PLANNED** (5 stories, 2.25h)

### Total Project
- **Total Stories**: 18 (13 complete, 5 planned)
- **Total Effort**: 17h (14.75h complete, 2.25h remaining)
- **Progress**: 82% complete by effort

### Migration Timeline
```
Sprint 1: ████████████████████████ 100% (6/6 stories)
Sprint 2: ████████████████████████100% (7/7 stories)
Sprint 3: ░░░░░░░░░░░░░░░░░░░░░░░░   0% (0/5 stories)
          └─── READY TO START ───┘
```

---

**Sprint 3 planning is complete!** 🎉

All documents are ready. Would you like to:
1. **Start Story 7.1** (Remove ms_context)?
2. **Start Story 6.1** (Wire PHASE3) immediately?
3. **Review a specific story** in detail before starting?
4. **Take a different approach**?

Just say "continue" to start with Story 7.1, or specify which story you'd like to begin with!
