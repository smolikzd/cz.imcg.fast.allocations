# AGENTS.md — AI Agent Instructions for Planning Repository

## Purpose
This is the **unified planning repository** for the IMCG Fast Allocations project. This repository coordinates development across two code repositories: `cz.imcg.fast.planner` (framework) and `cz.imcg.fast.ovysledovka` (implementation).

---

## 1. Repository Role

**This Repository**: `cz.imcg.fast.allocations`
- **Type**: Planning & Coordination Hub
- **Contains**: BMAD artifacts, constitution, sprint tracking, documentation
- **Does NOT Contain**: ABAP code, DDIC objects, or implementation files

**Code Repositories** (separate):
- `cz.imcg.fast.planner` - Framework code (ZFI_PROCESS orchestration)
- `cz.imcg.fast.ovysledovka` - Implementation code (allocation steps)

---

## 2. General Principles

1. **Follow BMAD Methodology**
   - BMAD workflows in `_bmad/bmm/workflows/` define structured processes
   - If requirements are ambiguous, request clarification *instead of guessing*

2. **This Repository is for Planning Only**
   - Generate planning artifacts (epics, sprints, stories)
   - Track implementation status across code repositories
   - Maintain constitution as source of truth
   - Do NOT generate ABAP code here

3. **Constitution is Sacred**
   - Constitution at `_bmad/_memory/constitution.md` is the source of truth
   - Changes to constitution require governance procedure (see constitution)
   - After constitution changes, run `repos/sync-constitution.sh` to sync to code repos

4. **Be Idempotent**
   - Re-running an instruction must not break existing plans or produce duplicates

---

## 3. Constitution Compliance (MANDATORY)

Before planning any changes, consult `_bmad/_memory/constitution.md`:

- **Principle I - DDIC-First**: No local TYPE definitions (use DDIC structures/table types)
- **Principle II - SAP Standards**: SAP naming conventions, line length ≤120 chars (≤255 absolute)
- **Principle III - Consult SAP Docs**: mcp-sap-docs consulted for new patterns
- **Principle IV - Factory Pattern**: Factory methods used, no direct NEW() instantiation
- **Principle V - Error Handling**: ZCX_FI_PROCESS_ERROR raised with proper context

When creating story artifacts, always include a "Constitution Compliance" section listing which principles apply.

---

## 4. Multi-Repository Coordination

### When Planning Stories

Each story artifact MUST specify:
- **Target Repository**: `planner` (framework) or `ovysledovka` (implementation)
- **Dependencies**: Other stories this depends on (may be in different repo)
- **Constitution Principles**: Which principles apply to the implementation

Example story header:
```yaml
story_id: "7.4"
epic_id: "7"
title: "Document cleanup_old_instances behavior"
target_repository: "planner"  # <-- REQUIRED
depends_on: []
constitution_principles:
  - "Principle V - Error Handling"
  - "Code Documentation Standards"
status: "ready-for-dev"
```

### Cross-Repository Dependencies

If a story spans both repositories:
1. Create a single story artifact
2. Set `target_repository: "both"`
3. In the story description, clearly separate:
   - **Framework changes** (what goes in `planner`)
   - **Implementation changes** (what goes in `ovysledovka`)
4. Story is only "done" when BOTH parts are complete

---

## 5. Sprint Status Tracking

**File**: `_bmad-output/implementation-artifacts/sprint-status.yaml`

This is the **single source of truth** for project status across all repositories.

### Updating Status

When a story is completed in a code repository:

1. **Update status**:
   ```yaml
   - story_id: "7.4"
     status: "done"  # was: "in-progress"
     completion_date: "2026-03-10"
     commit_hash: "29436ab"  # <-- git commit from code repo
     commit_repository: "planner"  # <-- which repo
     notes: "Documented cleanup_old_instances auto-commit behavior"
   ```

2. **Commit to planning repo**:
   ```bash
   git add _bmad-output/implementation-artifacts/sprint-status.yaml
   git commit -m "Story 7.4: Mark as done (planner commit 29436ab)"
   git push origin master
   ```

---

## 6. Common Agent Tasks

### Task A: Create Sprint Plan

**Input**: User provides sprint goal and story list

**Steps**:
1. Review constitution (`_bmad/_memory/constitution.md`)
2. Review existing epics (`_bmad-output/planning-artifacts/epics-*.md`)
3. Create sprint plan:
   - `_bmad-output/planning-artifacts/sprint-N-plan.md` (detailed plan)
   - `_bmad-output/planning-artifacts/sprint-N-checklist.md` (task checklist)
   - `_bmad-output/planning-artifacts/sprint-N-visual-summary.md` (overview)
4. Create story artifacts:
   - `_bmad-output/planning-artifacts/story-<epic>.<story>-<name>.md`
   - **REQUIRED**: Include `target_repository` field
5. Update `sprint-status.yaml` with new stories (status: "ready-for-dev")
6. Commit all artifacts

**Output**: Complete sprint plan ready for development

### Task B: Update Story Status

**Input**: User says "Story X.Y is done, commit abc1234 in planner repo"

**Steps**:
1. Open `_bmad-output/implementation-artifacts/sprint-status.yaml`
2. Find story X.Y
3. Update:
   ```yaml
   status: "done"
   completion_date: "<today>"
   commit_hash: "abc1234"
   commit_repository: "planner"  # or "ovysledovka"
   ```
4. Update story artifact with completion notes
5. Commit changes

**Output**: Status tracking updated, story marked complete

### Task C: Generate Cross-Repository Status Report

**Input**: User asks "What's the current status?"

**Steps**:
1. Read `_bmad-output/implementation-artifacts/sprint-status.yaml`
2. Group stories by:
   - Sprint (1, 2, 3)
   - Status (done, in-progress, ready-for-dev)
   - Repository (planner, ovysledovka, both)
3. Calculate metrics:
   - Stories completed per sprint
   - Stories per repository
   - Overall completion percentage
4. Present summary

**Output**: Status report showing progress across all repos

### Task D: Sync Constitution to Code Repos

**Input**: User modified constitution and asks to sync

**Steps**:
1. Verify constitution version bumped (major.minor.patch)
2. Run sync script:
   ```bash
   cd /Users/smolik/DEV/cz.imcg.fast.allocations
   ./repos/sync-constitution.sh
   ```
3. Verify sync output shows success for both repos
4. Remind user to commit in code repos:
   ```bash
   cd ~/DEV/cz.imcg.fast.planner
   git add src/.constitution.md
   git commit -m "Sync constitution v<version>"
   git push
   
   cd ~/DEV/cz.imcg.fast.ovysledovka
   git add src/.constitution.md
   git commit -m "Sync constitution v<version>"
   git push
   ```

**Output**: Constitution synced to both code repositories

---

## 7. DO NOT Generate Code

**IMPORTANT**: This planning repository does NOT contain ABAP code.

If the user asks you to:
- Generate ABAP classes
- Create DDIC objects
- Write method implementations
- Modify ABAP code

**STOP and clarify**:
```
This is the planning repository. To implement ABAP code, we need to work in:
- cz.imcg.fast.planner (for framework changes)
- cz.imcg.fast.ovysledovka (for business logic)

Would you like me to:
A) Create a story artifact planning this change?
B) Switch to a code repository to implement it?
C) Generate implementation guidance for developers?
```

---

## 8. Directory Structure Reference

```
cz.imcg.fast.allocations/              # THIS REPOSITORY
├── _bmad/
│   ├── bmm/workflows/                 # BMAD workflow definitions
│   └── _memory/
│       ├── constitution.md            # SOURCE OF TRUTH (v1.0.0)
│       └── config.yaml                # BMAD configuration
├── _bmad-output/
│   ├── planning-artifacts/            # Epics, sprints, stories
│   │   ├── epics-alloc-remediation.md
│   │   ├── sprint-1-plan.md
│   │   ├── sprint-2-plan.md
│   │   ├── sprint-3-plan.md
│   │   └── story-*.md
│   └── implementation-artifacts/      # Sprint status (single source of truth)
│       └── sprint-status.yaml
├── docs/
│   ├── architecture/                  # Architecture decisions
│   ├── migration-reviews/             # Migration analysis
│   ├── PROJECT-STATUS.md              # Consolidated status from all repos
│   └── CROSS-REPO-WORKFLOW.md         # Developer workflow guide
├── repos/
│   ├── registry.md                    # Code repository metadata
│   └── sync-constitution.sh           # Constitution sync script
├── README.md                          # Repository overview
└── AGENTS.md                          # THIS FILE
```

---

## 9. Repository Metadata

**Code Repositories**:

| Repository | Remote | Purpose | Constitution |
|------------|--------|---------|--------------|
| `cz.imcg.fast.planner` | https://github.com/smolikzd/cz.imcg.fast.planner.git | Framework (ZFI_PROCESS) | Synced to `src/.constitution.md` |
| `cz.imcg.fast.ovysledovka` | https://github.com/IMCG-SRO/cz.imcg.fast.ovysledovka.git | Implementation (allocation steps) | Synced to `src/.constitution.md` |

**Local Paths** (expected):
- Planning: `/Users/smolik/DEV/cz.imcg.fast.allocations`
- Framework: `/Users/smolik/DEV/cz.imcg.fast.planner`
- Implementation: `/Users/smolik/DEV/cz.imcg.fast.ovysledovka`

---

## 10. Constitution Amendment Procedure

If constitution needs changes (see Constitution § Governance):

1. **Document Justification**:
   - Why is the change necessary?
   - What impact on existing code?
   - What templates need updates?

2. **Update Version**:
   - MAJOR (X.0.0): Principle removed/redefined (breaking)
   - MINOR (0.X.0): New principle added
   - PATCH (0.0.X): Clarifications only

3. **Edit Constitution**:
   ```bash
   # Edit _bmad/_memory/constitution.md
   # Update version at bottom of file
   ```

4. **Sync to Code Repos**:
   ```bash
   ./repos/sync-constitution.sh
   ```

5. **Commit Everywhere**:
   - Planning repo: Constitution change
   - Code repos: Synced constitution

6. **Communicate to Team**:
   - Document in `docs/architecture/ADR-constitution-vX.Y.Z.md`

---

## 11. Best Practices for Agents

### ✅ DO

- Always specify `target_repository` in story artifacts
- Update `sprint-status.yaml` immediately after story completion
- Include constitution principles in every story
- Cross-reference commit hashes from code repos
- Verify constitution version matches across all repos

### ❌ DON'T

- Don't generate ABAP code in this repository
- Don't modify constitution without following amendment procedure
- Don't mark stories "done" without commit hashes from code repos
- Don't create stories without specifying target repository
- Don't forget to sync constitution after changes

---

## 12. Active Technologies

This planning repository tracks development for:

**Framework Repository**:
- ABAP 7.58 (SAP S/4HANA compatible)
- ZFI_PROCESS framework (orchestration engine)
- bgRFC for background processing
- SAP DDIC (94 objects)
- SAP HANA database

**Implementation Repository**:
- ZFI_ALLOC_PROCESS package (5 step classes)
- Business logic for cost allocation (3 phases + correction)
- Integration with ZFI_PROCESS framework

---

## 13. Support & Escalation

**Project Lead**: Zdenek Smolik (smolik@imcg.cz)  
**Organization**: IMCG s.r.o.  
**Methodology**: BMAD (Business Methodology for Agile Development)

**Key Documents**:
- Constitution: `_bmad/_memory/constitution.md`
- Project Status: `docs/PROJECT-STATUS.md`
- Workflow Guide: `docs/CROSS-REPO-WORKFLOW.md`
- Repository Registry: `repos/registry.md`

---

**Version**: 1.0.0  
**Last Updated**: 2026-03-10  
**Maintained By**: Zdenek Smolik
