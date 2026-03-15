# Cross-Repository Development Workflow

This document describes how to work across the three repositories in the IMCG Fast Allocations project.

---

## Repository Structure Overview

```
~/DEV/
├── cz.imcg.fast.allocations/     # PLANNING REPOSITORY (you start here)
│   ├── _bmad/_memory/             # Constitution, config (SOURCE OF TRUTH)
│   ├── _bmad-output/              # Epics, sprints, status
│   ├── docs/                      # Architecture, status
│   └── repos/                     # Sync scripts, registry
│
├── cz.imcg.fast.planner/          # FRAMEWORK CODE REPOSITORY
│   ├── src/                       # Framework classes, DDIC
│   │   └── .constitution.md       # SYNCED (read-only)
│   └── AGENTS.md                  # Agent instructions
│
└── cz.imcg.fast.ovysledovka/      # IMPLEMENTATION CODE REPOSITORY
    ├── src/                       # Business logic, allocation steps
    │   └── .constitution.md       # SYNCED (read-only)
    └── AGENTS.md                  # Agent instructions
```

---

## Development Workflows

### 1. Planning a New Feature

**Location**: `cz.imcg.fast.allocations` (planning repo)

**Steps**:

1. **Create Epic** (if needed):
   ```bash
   cd ~/DEV/cz.imcg.fast.allocations
   # Create epic in _bmad-output/planning-artifacts/epics-<name>.md
   ```

2. **Plan Sprint**:
   ```bash
   # Create sprint plan in _bmad-output/planning-artifacts/sprint-N-<name>.md
   # Include:
   # - Story breakdown
   # - Which repository each story targets (planner vs ovysledovka)
   # - Dependencies between stories
   # - Constitution compliance requirements
   ```

3. **Create Story Artifacts**:
   ```bash
   # For each story, create:
   # _bmad-output/planning-artifacts/story-<epic>.<story>-<name>.md
   # Include:
   # - Target repository (planner or ovysledovka)
   # - Acceptance criteria
   # - Constitution principles to follow
   ```

4. **Update Sprint Status**:
   ```bash
   # Edit _bmad-output/implementation-artifacts/sprint-status.yaml
   # Add new stories with status: "ready-for-dev"
   ```

5. **Commit Planning**:
   ```bash
   git add _bmad-output/
   git commit -m "Add Sprint N planning artifacts"
   git push origin master
   ```

---

### 2. Implementing Framework Changes

**Location**: `cz.imcg.fast.planner` (framework repo)

**When**: Story targets framework layer (ZCL_FI_PROCESS_*, DDIC objects, bgRFC)

**Steps**:

1. **Review Constitution**:
   ```bash
   cd ~/DEV/cz.imcg.fast.planner
   cat src/.constitution.md   # Read-only synced copy
   ```

2. **Consult SAP Documentation** (Constitution Principle III):
   ```bash
   # REQUIRED: Before making changes, consult mcp-sap-docs
   # - Search for ABAP patterns you plan to use
   # - Verify DDIC naming conventions
   # - Check compatibility with SAP version
   ```

3. **Implement Changes**:
   ```bash
   # Follow constitution principles:
   # - DDIC-First: Define types in DDIC (no local types)
   # - Factory Pattern: Use factories for complex objects
   # - Error Handling: Raise ZCX_FI_PROCESS_ERROR
   # - Line Length: Max 120 characters
   # - ABAP-Doc: All public methods documented
   ```

4. **Test Changes**:
   ```bash
   # Update test programs (e.g., ZFI_PROCESS_FRAMEWORK)
   # Run demo programs to verify functionality
   ```

5. **Commit with Story Reference**:
   ```bash
   git add .
   git commit -m "Story X.Y: <description>"
   git push origin main
   ```

6. **Update Planning Repo**:
   ```bash
   cd ~/DEV/cz.imcg.fast.allocations
   # Update sprint-status.yaml with commit hash and status
   # Update story artifact with completion notes
   git add _bmad-output/
   git commit -m "Story X.Y: Update status to done (commit abc1234)"
   git push origin master
   ```

---

### 3. Implementing Business Logic Changes

**Location**: `cz.imcg.fast.ovysledovka` (implementation repo)

**When**: Story targets allocation steps or business logic

**Steps**:

1. **Review Constitution**:
   ```bash
   cd ~/DEV/cz.imcg.fast.ovysledovka
   cat src/.constitution.md   # Read-only synced copy
   ```

2. **Consult SAP Documentation** (Constitution Principle III):
   ```bash
   # REQUIRED: Before making changes, consult mcp-sap-docs
   # - Verify ABAP syntax for your SAP version
   # - Check patterns for step implementations
   ```

3. **Implement Changes**:
   ```bash
   # Modify allocation step classes:
   # - src/zfi_alloc_process/zcl_fi_alloc_step_*.clas.abap
   # 
   # Follow constitution principles:
   # - Implement ZIF_FI_PROCESS_STEP interface
   # - Use DDIC types from framework
   # - Raise ZCX_FI_PROCESS_ERROR on errors
   # - Public constructors OK for step classes
   ```

4. **Test Changes**:
   ```bash
   # Update test program: src/zfi_alloc_process/zfi_alloc_proc_test.prog.abap
   # Run end-to-end allocation pipeline
   ```

5. **Commit with Story Reference**:
   ```bash
   git add .
   git commit -m "Story X.Y: <description>"
   git push origin main
   ```

6. **Update Planning Repo**:
   ```bash
   cd ~/DEV/cz.imcg.fast.allocations
   # Update sprint-status.yaml with commit hash and status
   # Update story artifact with completion notes
   git add _bmad-output/
   git commit -m "Story X.Y: Update status to done (commit xyz5678)"
   git push origin master
   ```

---

### 4. Updating the Constitution

**Location**: `cz.imcg.fast.allocations` (planning repo)

**When**: Constitution principles need changes (rare!)

**Steps**:

1. **Edit Constitution** (planning repo):
   ```bash
   cd ~/DEV/cz.imcg.fast.allocations
   # Edit _bmad/_memory/constitution.md
   # Follow amendment procedure from constitution governance section
   # Update version number (major/minor/patch)
   ```

2. **Sync to Code Repos**:
   ```bash
   ./repos/sync-constitution.sh
   # This copies constitution to:
   # - ../cz.imcg.fast.planner/src/.constitution.md
   # - ../cz.imcg.fast.ovysledovka/src/.constitution.md
   ```

3. **Commit in Planning Repo**:
   ```bash
   git add _bmad/_memory/constitution.md
   git commit -m "Constitution v<version>: <change description>"
   git push origin master
   ```

4. **Commit in Code Repos**:
   ```bash
   cd ~/DEV/cz.imcg.fast.planner
   git add src/.constitution.md
   git commit -m "Sync constitution v<version> from planning repo"
   git push origin main

   cd ~/DEV/cz.imcg.fast.ovysledovka
   git add src/.constitution.md
   git commit -m "Sync constitution v<version> from planning repo"
   git push origin main
   ```

---

### 5. Sprint Review & Retrospective

**Location**: `cz.imcg.fast.allocations` (planning repo)

**When**: End of sprint

**Steps**:

1. **Verify All Stories Complete**:
   ```bash
   cd ~/DEV/cz.imcg.fast.allocations
   # Check _bmad-output/implementation-artifacts/sprint-status.yaml
   # Ensure all stories have status: "done" or "cancelled"
   ```

2. **Review Code Repos**:
   ```bash
   cd ~/DEV/cz.imcg.fast.planner && git log --since="<sprint-start-date>"
   cd ~/DEV/cz.imcg.fast.ovysledovka && git log --since="<sprint-start-date>"
   # Verify all story commits are present
   ```

3. **Update Sprint Status**:
   ```bash
   # Edit sprint-status.yaml
   # - Mark sprint as "done"
   # - Add completion_date
   # - Update epic statuses
   ```

4. **Document Learnings**:
   ```bash
   # Create retrospective document in docs/
   # Document:
   # - What went well
   # - What could improve
   # - Constitution compliance issues
   # - Technical debt identified
   ```

5. **Commit Sprint Completion**:
   ```bash
   git add _bmad-output/ docs/
   git commit -m "Sprint N: Mark as complete"
   git push origin master
   ```

---

## Common Scenarios

### Scenario A: Story Spans Both Repositories

**Example**: Add new step type to framework and implement allocation step

**Workflow**:

1. **Plan**: Create single story that targets both repos
2. **Implement Part 1** (framework changes in `planner`):
   ```bash
   cd ~/DEV/cz.imcg.fast.planner
   # Add new interface method to ZIF_FI_PROCESS_STEP
   # Update ZCL_FI_PROCESS_INSTANCE to call new method
   git commit -m "Story X.Y (part 1/2): Add new step type to framework"
   git push origin main
   ```

3. **Implement Part 2** (business logic in `ovysledovka`):
   ```bash
   cd ~/DEV/cz.imcg.fast.ovysledovka
   # Implement new method in all allocation step classes
   git commit -m "Story X.Y (part 2/2): Implement new step type in allocation steps"
   git push origin main
   ```

4. **Update Planning Repo** (mark story done when BOTH parts complete):
   ```bash
   cd ~/DEV/cz.imcg.fast.allocations
   # Update sprint-status.yaml with BOTH commit hashes
   # Update story artifact with notes about cross-repo implementation
   ```

### Scenario B: Emergency Constitution Fix

**Example**: Critical issue found in constitution principle

**Workflow**:

1. **Fix in Planning Repo**:
   ```bash
   cd ~/DEV/cz.imcg.fast.allocations
   # Edit _bmad/_memory/constitution.md
   # Bump PATCH version (e.g., 1.0.0 → 1.0.1)
   git commit -m "Constitution v1.0.1: Fix critical issue in Principle X"
   ```

2. **Sync Immediately**:
   ```bash
   ./repos/sync-constitution.sh
   cd ~/DEV/cz.imcg.fast.planner && git add src/.constitution.md && git commit -m "Sync constitution v1.0.1" && git push
   cd ~/DEV/cz.imcg.fast.ovysledovka && git add src/.constitution.md && git commit -m "Sync constitution v1.0.1" && git push
   cd ~/DEV/cz.imcg.fast.allocations && git push origin master
   ```

### Scenario C: Reviewing Status Across All Repos

**Quick Check**:

```bash
#!/bin/bash
# Check status of all three repos

echo "=== PLANNING REPO ==="
cd ~/DEV/cz.imcg.fast.allocations && git status

echo ""
echo "=== FRAMEWORK REPO ==="
cd ~/DEV/cz.imcg.fast.planner && git status

echo ""
echo "=== IMPLEMENTATION REPO ==="
cd ~/DEV/cz.imcg.fast.ovysledovka && git status
```

---

## Best Practices

### ✅ DO

- Always consult constitution before coding (Principle III)
- Update sprint status immediately after completing a story
- Reference story numbers in all commits (e.g., "Story 6.1: ...")
- Sync constitution after any changes
- Document cross-repo dependencies in story artifacts
- Keep planning repo as single source of truth for status

### ❌ DON'T

- Don't edit `.constitution.md` in code repos (read-only, synced from planning)
- Don't skip constitution compliance checks
- Don't commit to code repos without updating planning repo status
- Don't create local TYPE definitions (Constitution Principle I)
- Don't use MESSAGE TYPE 'E' (use RAISE EXCEPTION - Constitution Principle V)
- Don't exceed 120 character line length (Constitution - Code Formatting)

---

## Troubleshooting

### Constitution Sync Failed

**Problem**: `sync-constitution.sh` can't find code repositories

**Solution**:
```bash
# Verify all three repos are in ~/DEV/
ls ~/DEV/ | grep "cz.imcg.fast"
# Should show: allocations, planner, ovysledovka

# If missing, clone them:
cd ~/DEV/
git clone https://github.com/smolikzd/cz.imcg.fast.planner.git
git clone https://github.com/smolikzd/cz.imcg.fast.ovysledovka.git
```

### Sprint Status Out of Sync

**Problem**: Story shows "done" but code not committed

**Solution**:
```bash
# Find the commit mentioned in sprint-status.yaml
cd ~/DEV/cz.imcg.fast.planner  # or ovysledovka
git log --oneline --all | grep "<commit-hash>"

# If missing, status is wrong - revert status in planning repo
cd ~/DEV/cz.imcg.fast.allocations
# Edit sprint-status.yaml to correct status
```

### Constitution Version Mismatch

**Problem**: Code repos have different constitution versions

**Solution**:
```bash
# Re-sync from planning repo (source of truth)
cd ~/DEV/cz.imcg.fast.allocations
./repos/sync-constitution.sh

# Commit synced versions in code repos
cd ~/DEV/cz.imcg.fast.planner && git add src/.constitution.md && git commit -m "Sync constitution" && git push
cd ~/DEV/cz.imcg.fast.ovysledovka && git add src/.constitution.md && git commit -m "Sync constitution" && git push
```

---

## Quick Reference

| Task | Repository | Command |
|------|------------|---------|
| Plan new feature | allocations | Edit `_bmad-output/planning-artifacts/` |
| Check sprint status | allocations | View `_bmad-output/implementation-artifacts/sprint-status.yaml` |
| Implement framework | planner | Edit `src/zcl_fi_process_*.clas.abap` |
| Implement business logic | ovysledovka | Edit `src/zfi_alloc_process/zcl_fi_alloc_*.clas.abap` |
| Update constitution | allocations | Edit `_bmad/_memory/constitution.md` then run `repos/sync-constitution.sh` |
| Test allocation pipeline | ovysledovka | Run `src/zfi_alloc_process/zfi_alloc_proc_test.prog.abap` |
| Review project status | allocations | View `docs/PROJECT-STATUS.md` |

---

**Last Updated**: 2026-03-10  
**Maintained By**: Zdenek Smolik (smolik@imcg.cz)
