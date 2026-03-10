# cz.imcg.fast.allocations - Planning Repository

## Purpose

This repository serves as the **unified planning and coordination hub** for the IMCG Fast Allocations project, which spans multiple code repositories.

## Architecture

The project follows a **Hybrid Planning Repository** structure:

```
cz.imcg.fast.allocations/     ← Planning & Coordination (THIS REPO)
├── _bmad/                     ← BMAD methodology artifacts
│   ├── bmm/workflows/         ← BMAD workflow definitions
│   └── _memory/               ← Constitution, config, project memory
├── _bmad-output/              ← Generated planning artifacts
│   ├── planning-artifacts/    ← Epics, sprints, stories
│   └── implementation-artifacts/ ← Sprint status, implementation plans
├── docs/                      ← Cross-repo documentation
│   ├── architecture/          ← System architecture decisions
│   └── migration-reviews/     ← Migration analysis documents
└── repos/                     ← Repository metadata and sync scripts

cz.imcg.fast.planner/         ← Framework Code Repository
└── src/                       ← ZFI_PROCESS framework (orchestration, bgRFC, DDIC)

cz.imcg.fast.ovysledovka/     ← Implementation Code Repository
└── src/                       ← Business allocation step implementations
```

## Code Repositories

### Framework Repository: `cz.imcg.fast.planner`
- **Remote**: https://github.com/smolikzd/cz.imcg.fast.planner.git
- **Purpose**: Contains the ZFI_PROCESS orchestration framework
- **Contents**: 
  - Core framework classes (94 DDIC objects, 23 classes)
  - Process manager, step base classes, bgRFC handlers
  - UI components and monitoring tools
  - Framework constitution and standards

### Implementation Repository: `cz.imcg.fast.ovysledovka`
- **Remote**: https://github.com/IMCG-SRO/cz.imcg.fast.ovysledovka.git
- **Purpose**: Contains business-specific allocation process implementations
- **Contents**:
  - 5 allocation step classes (INIT, PHASE1, PHASE2, PHASE3, CORR_BCHE)
  - Test programs for allocation pipeline
  - Business logic for cost allocation

## Project Status

**Current Status**: ✅ Sprint 3 Complete (All Development Done)

- **Sprint 1**: 6/6 stories ✅ (Unblock allocation pipeline)
- **Sprint 2**: 7/8 stories ✅, 1 cancelled (Structural correctness)
- **Sprint 3**: 5/5 stories ✅ (Wire Phase 3 + housekeeping)

**Total**: 18/19 stories completed (95% completion rate)

See `_bmad-output/implementation-artifacts/sprint-status.yaml` for detailed status.

## Constitution

The project follows the **ZFI_PROCESS Framework Constitution v1.0.0** with five core principles:

1. **DDIC-First Architecture**: No local TYPE definitions
2. **SAP Standards Compliance**: Naming conventions, line length limits
3. **Consult SAP Documentation**: mcp-sap-docs for new patterns
4. **Factory Pattern Enforcement**: No direct NEW() instantiation
5. **Unified Error Handling**: ZCX_FI_PROCESS_ERROR exceptions

See `_bmad/_memory/constitution.md` for full details.

## Development Workflow

### For New Features or Changes:

1. **Plan in this repo**: Create epics/stories in `_bmad-output/planning-artifacts/`
2. **Update sprint status**: Track progress in `_bmad-output/implementation-artifacts/sprint-status.yaml`
3. **Implement in code repos**: 
   - Framework changes → `cz.imcg.fast.planner`
   - Business logic → `cz.imcg.fast.ovysledovka`
4. **Sync back to planning**: Update status and document learnings

### Constitution Compliance:

All code changes must comply with the constitution. Run checks before committing:
```bash
# In code repositories
./scripts/check-constitution.sh
```

## AI Agent Guidelines

See `AGENTS.md` in this repository for instructions on:
- BMAD methodology compliance
- Constitution enforcement
- Cross-repository coordination
- Code generation standards

## Getting Started

### Clone All Repositories:

```bash
# Planning repository
git clone https://github.com/IMCG-SRO/cz.imcg.fast.allocations.git

# Framework repository
git clone https://github.com/smolikzd/cz.imcg.fast.planner.git

# Implementation repository  
git clone https://github.com/IMCG-SRO/cz.imcg.fast.ovysledovka.git
```

### Directory Layout:

```bash
~/DEV/
├── cz.imcg.fast.allocations/   # Planning (start here)
├── cz.imcg.fast.planner/        # Framework code
└── cz.imcg.fast.ovysledovka/    # Implementation code
```

## Maintenance

- **Constitution updates**: Edit in planning repo, sync to code repos via `repos/sync-constitution.sh`
- **Sprint planning**: All planning happens in `_bmad-output/planning-artifacts/`
- **Status tracking**: Single source of truth in `_bmad-output/implementation-artifacts/sprint-status.yaml`

## Contact

- **Project Lead**: Zdenek Smolik (smolik@imcg.cz)
- **Organization**: IMCG s.r.o.
