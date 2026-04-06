# Code Repository Registry

This file tracks the code repositories that belong to this planning project.

## Repositories

### Planning Repository: cz.imcg.fast.allocations
- **Remote**: https://github.com/smolikzd/cz.imcg.fast.allocations.git
- **Local Path**: /Users/smolik/DEV/cz.imcg.fast.allocations
- **Purpose**: Planning and coordination hub (BMAD artifacts, constitution, sprint tracking)
- **Owner**: Zdenek Smolik
- **Constitution**: Source of truth at `_bmad/_memory/constitution.md`

### Framework Repository: cz.imcg.fast.planner
- **Remote**: https://github.com/smolikzd/cz.imcg.fast.planner.git
- **Local Path**: /Users/smolik/DEV/cz.imcg.fast.planner
- **Purpose**: ZFI_PROCESS orchestration framework
- **Owner**: Zdenek Smolik
- **Constitution Copy**: `src/.constitution.md` (synced from planning repo)

**Contents**:
- 94 DDIC objects (tables, structures, types, domains, data elements)
- 23 classes (framework core)
- bgRFC background processing handlers
- Process manager and step orchestration
- UI components and monitoring
- Test programs and demo data setup

**Key Classes**:
- `ZCL_FI_PROCESS_MANAGER` - Singleton process orchestrator
- `ZCL_FI_PROCESS_INSTANCE` - Process instance management
- `ZCL_FI_PROCESS_DEFINITION` - Process type definitions
- `ZCL_FI_PROCESS_STEP_BASE` - Base class for steps
- `ZCL_FI_BGRFC_*` - bgRFC handlers

### Orchestration Engine Repository: cz.en.orch
- **Remote**: https://github.com/smolikzd/cz.en.orch.git
- **Local Path**: /Users/smolik/DEV/cz.en.orch
- **Purpose**: ZEN_ORCH process orchestration engine (framework-agnostic)
- **Owner**: Zdenek Smolik
- **Constitution Copy**: `src/zen_orch/.constitution.md` (synced from planning repo)

**Phase 1 Status**: Complete (2026-04-06)

| Epic | Title | Completion Date |
|------|-------|-----------------|
| Epic 1 | Repository Bootstrap & DDIC Foundation | 2026-04-04 |
| Epic 2 | Error Handling & Logging Infrastructure | 2026-04-04 |
| Epic 3 | Adapter Framework | 2026-04-04 |
| Epic 4 | Engine Core | 2026-04-05 |
| Epic 5 | APJ Scheduling & Performance Lifecycle | 2026-04-06 |
| Epic 6 | Integration Testing & Repository Finalization | 2026-04-06 |

**Contents**:
- 60+ DDIC objects (domains, data elements, tables, structures, table types)
- 6 classes (`ZCL_EN_ORCH_*`) including engine, adapters, logger, health check
- 2 interfaces (`ZIF_EN_ORCH_*`): adapter and logger contracts
- 1 exception class (`ZCX_EN_ORCH_ERROR`)
- Message class `ZEN_ORCH`
- APJ Job Catalog and Template (`ZEN_ORCH_JOB_CAT`, `ZEN_ORCH_JOB_TMPL`)
- CDS view and OData V4 service for health check monitoring
- Integration test report `ZEN_ORCH_TEST`
- Gate-based process orchestration engine
- Adapter framework (no compile-time dependency on ZFI_PROCESS or ZFI_ALLOC_PROCESS)

**Key Classes**:
- `ZCL_EN_ORCH_ENGINE` - Engine singleton (sweep_all, create/cancel/restart/resume performance)
- `ZCL_EN_ORCH_ADAPTER_FACTORY` - Adapter instantiation from registry
- `ZCL_EN_ORCH_ADAPTER_MOCK` - Mock adapter for testing (synchronous COMPLETED)
- `ZCL_EN_ORCH_LOGGER` - BAL logger implementation
- `ZCL_EN_ORCH_HEALTH_CHK_QUERY` - Health check OData query class

### Implementation Repository: cz.imcg.fast.ovysledovka
- **Remote**: https://github.com/smolikzd/cz.imcg.fast.ovysledovka.git
- **Local Path**: /Users/smolik/DEV/cz.imcg.fast.ovysledovka
- **Purpose**: Business-specific allocation process implementations
- **Owner**: IMCG s.r.o.
- **Constitution Copy**: `src/.constitution.md` (synced from planning repo)

**Contents**:
- 3 packages: `zfi_alloc_process`, `zfi_ui`, `zfi_common`
- 5 allocation step classes implementing `ZIF_FI_PROCESS_STEP`
- Test programs for allocation pipeline
- Business logic for cost allocation

**Key Classes**:
- `ZCL_FI_ALLOC_STEP_INIT` - Initialization step
- `ZCL_FI_ALLOC_STEP_PHASE1` - Phase 1 allocation logic
- `ZCL_FI_ALLOC_STEP_PHASE2` - Phase 2 allocation logic
- `ZCL_FI_ALLOC_STEP_PHASE3` - Phase 3 allocation logic
- `ZCL_FI_ALLOC_STEP_CORR_BCHE` - Correction batch processing

**Test Programs**:
- `ZFI_ALLOC_PROC_TEST` - Main allocation pipeline test

## Sync Requirements

### Constitution Sync
The constitution must be synced to both code repositories whenever it changes:
```bash
./repos/sync-constitution.sh
```

This copies `_bmad/_memory/constitution.md` to:
- `../cz.imcg.fast.planner/src/.constitution.md`
- `../cz.imcg.fast.ovysledovka/src/.constitution.md`
- `../cz.en.orch/src/zen_orch/.constitution.md`

### Planning Artifact Sync
When creating new stories/tasks in `_bmad-output/planning-artifacts/`, ensure they reference the correct repository for implementation.

### Status Sync
After completing work in code repositories:
1. Update `_bmad-output/implementation-artifacts/sprint-status.yaml`
2. Commit git hashes of completed work for audit trail
3. Document any learnings in this planning repo

## Development Workflow

1. **Planning Phase** (in THIS repo):
   - Create epics in `_bmad-output/planning-artifacts/epics-*.md`
   - Create sprint plans in `_bmad-output/planning-artifacts/sprint-N-*.md`
   - Update sprint status in `_bmad-output/implementation-artifacts/sprint-status.yaml`

2. **Implementation Phase** (in CODE repos):
   - Framework changes → `cz.imcg.fast.planner`
   - Business logic → `cz.imcg.fast.ovysledovka`
   - Follow constitution principles (consult mcp-sap-docs, DDIC-first, etc.)

3. **Completion Phase** (back to THIS repo):
   - Update sprint status with completion dates
   - Document learnings in `docs/`
   - Archive sprint artifacts

## Directory Structure Mapping

```
cz.imcg.fast.allocations/              # Planning Repository (THIS)
├── _bmad/_memory/constitution.md      # SOURCE OF TRUTH for constitution
├── _bmad-output/
│   ├── planning-artifacts/            # Epics, sprints, stories
│   └── implementation-artifacts/      # Sprint status, plans
└── repos/
    ├── registry.md                    # THIS FILE
    └── sync-constitution.sh           # Constitution sync script

cz.imcg.fast.planner/                  # Framework Code Repository
├── src/
│   ├── .constitution.md               # SYNCED COPY (read-only)
│   ├── zcl_fi_process_*.clas.abap    # Framework classes
│   └── zfi_*.ddic.xml                 # DDIC objects
└── AGENTS.md                          # Agent instructions (references constitution)

cz.imcg.fast.ovysledovka/              # Implementation Code Repository
├── src/
│   ├── .constitution.md               # SYNCED COPY (read-only)
│   ├── zfi_alloc_process/
│   │   ├── zcl_fi_alloc_*.clas.abap  # Step implementations
│   │   └── zfi_alloc_proc_test.prog.abap
│   └── zfi_ui/
└── AGENTS.md                          # Agent instructions (references constitution)
```

## Notes

- Constitution is **owned** by the planning repo, **synced** to code repos
- Sprint status is **owned** by the planning repo
- Code implementations are **owned** by their respective code repos
- All three repos must be cloned to `~/DEV/` for sync scripts to work
