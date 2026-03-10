---
story: "7.3"
title: "Delete *_ORIG dead classes from system and repository"
epic: "7 — Code Quality and Housekeeping"
sprint: 3
status: ready-for-dev
priority: low
estimate: 30min
source_findings:
  - CC-10
  - cross-cutting-analysis.md
target_repo: smolikzd/cz.imcg.fast.planner
dependencies: []
---

# Story 7.3: Delete *_ORIG Dead Classes from System and Repository

## User Story

As a developer working in the `ZFI_ALLOC_PROCESS` package,
I want obsolete snapshot classes removed,
so that there is no namespace confusion and the package contains only live objects.

## Background

The repository contains dead snapshot classes created as pre-migration backups:
- `ZCL_FI_ALLOC_STEP_PHASE2_ORIG`
- `ZCL_FI_ALLOC_STEP_PHASE3_ORIG`

These classes were created to preserve the original PHASE2 and PHASE3 implementations
before migration. Now that migration is complete and tested, they serve no purpose:
- They are not referenced anywhere in the codebase
- They are not used by any process or report
- They create namespace clutter
- They may confuse developers ("Which one should I modify?")

Keeping dead code in the system is bad practice:
- Wastes repository space
- Makes package navigation harder
- Creates maintenance confusion
- Violates clean code principles

## Acceptance Criteria

**AC-01 — ORIG classes removed from SAP system:**
Given the SAP system
When cleanup is complete
Then `ZCL_FI_ALLOC_STEP_PHASE2_ORIG` does not exist as an active SAP object
And `ZCL_FI_ALLOC_STEP_PHASE3_ORIG` does not exist as an active SAP object

**AC-02 — ORIG files removed from repository:**
Given the git repository
When cleanup is complete
Then `zcl_fi_alloc_step_phase2_orig.clas.*` files are deleted
And `zcl_fi_alloc_step_phase3_orig.clas.*` files are deleted

**AC-03 — No dangling references:**
Given a grep search of the entire repository
When searching for `PHASE2_ORIG` or `PHASE3_ORIG`
Then no matches are found in any source file

## Tasks

- [ ] **Task 7.3.1**: Grep repository for any references to `PHASE2_ORIG`
  (verify zero results before deletion)

- [ ] **Task 7.3.2**: Grep repository for any references to `PHASE3_ORIG`
  (verify zero results before deletion)

- [ ] **Task 7.3.3**: Identify all files for `PHASE2_ORIG` in repository:
  - `zcl_fi_alloc_step_phase2_orig.clas.abap`
  - `zcl_fi_alloc_step_phase2_orig.clas.xml`
  - `zcl_fi_alloc_step_phase2_orig.clas.locals_imp.abap` (if exists)
  - `zcl_fi_alloc_step_phase2_orig.clas.testclasses.abap` (if exists)

- [ ] **Task 7.3.4**: Identify all files for `PHASE3_ORIG` in repository:
  - `zcl_fi_alloc_step_phase3_orig.clas.abap`
  - `zcl_fi_alloc_step_phase3_orig.clas.xml`
  - `zcl_fi_alloc_step_phase3_orig.clas.locals_imp.abap` (if exists)
  - `zcl_fi_alloc_step_phase3_orig.clas.testclasses.abap` (if exists)

- [ ] **Task 7.3.5**: Delete `ZCL_FI_ALLOC_STEP_PHASE2_ORIG` from SAP system
  (Use SE24 or SE80)

- [ ] **Task 7.3.6**: Delete `ZCL_FI_ALLOC_STEP_PHASE3_ORIG` from SAP system
  (Use SE24 or SE80)

- [ ] **Task 7.3.7**: Remove all `PHASE2_ORIG` files from `src/zfi_alloc_process/`
  using `git rm`

- [ ] **Task 7.3.8**: Remove all `PHASE3_ORIG` files from `src/zfi_alloc_process/`
  using `git rm`

- [ ] **Task 7.3.9**: Commit deletion with descriptive message explaining cleanup

## Dev Notes

- **Safety check**: ALWAYS grep for references before deleting
  ```bash
  grep -rn "PHASE2_ORIG" /path/to/repo/src/
  grep -rn "PHASE3_ORIG" /path/to/repo/src/
  ```
  Expected result: 0 matches (files may match themselves, but no code references)

- **SAP system deletion**:
  - Use SE24 (Class Builder) or SE80 (Object Navigator)
  - Select class → Delete
  - Confirm deletion warning
  - Transport may be required depending on system configuration

- **Git cleanup**:
  - Use `git rm <file>` to properly stage deletions
  - Do NOT use shell `rm` command (git won't track deletion)
  - Verify with `git status` before committing

- **Risk**: Very low — dead code removal, no functional impact

- **Benefit**: Cleaner namespace, reduced confusion, follows clean code principles

- Reference: CC-10, `cross-cutting-analysis.md` §10

## Files to Delete

| File (remote repo) | Action |
|--------------------|--------|
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2_orig.clas.abap` | Delete |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2_orig.clas.xml` | Delete |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2_orig.clas.locals_imp.abap` | Delete (if exists) |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase2_orig.clas.testclasses.abap` | Delete (if exists) |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3_orig.clas.abap` | Delete |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3_orig.clas.xml` | Delete |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3_orig.clas.locals_imp.abap` | Delete (if exists) |
| `src/zfi_alloc_process/zcl_fi_alloc_step_phase3_orig.clas.testclasses.abap` | Delete (if exists) |

## Constitution Compliance Checklist

- [x] Not applicable (deletion only, no code changes)

## Command Reference

### Grep for references
```bash
# Check for code references (should be 0)
grep -rn "PHASE2_ORIG" /Users/smolik/DEV/cz.imcg.fast.ovysledovka/src/ --include="*.abap"
grep -rn "PHASE3_ORIG" /Users/smolik/DEV/cz.imcg.fast.ovysledovka/src/ --include="*.abap"
```

### List files to delete
```bash
# Find all ORIG files
ls -la /Users/smolik/DEV/cz.imcg.fast.ovysledovka/src/zfi_alloc_process/*_orig.*
```

### Delete files with git
```bash
cd /Users/smolik/DEV/cz.imcg.fast.ovysledovka

# Delete PHASE2_ORIG files
git rm src/zfi_alloc_process/zcl_fi_alloc_step_phase2_orig.clas.abap
git rm src/zfi_alloc_process/zcl_fi_alloc_step_phase2_orig.clas.xml
git rm src/zfi_alloc_process/zcl_fi_alloc_step_phase2_orig.clas.locals_imp.abap

# Delete PHASE3_ORIG files
git rm src/zfi_alloc_process/zcl_fi_alloc_step_phase3_orig.clas.abap
git rm src/zfi_alloc_process/zcl_fi_alloc_step_phase3_orig.clas.xml
git rm src/zfi_alloc_process/zcl_fi_alloc_step_phase3_orig.clas.locals_imp.abap

# Verify deletions staged
git status

# Commit
git commit -m "Story 7.3: Delete obsolete *_ORIG snapshot classes"
```

## Dev Agent Record

### Agent Model Used

### Completion Notes

### Deleted Files
| File (remote repo) | Commit |
|--------------------|--------|
