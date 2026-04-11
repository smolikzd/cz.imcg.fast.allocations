# Project Status Consolidation

**Generated**: 2026-04-11 (updated from 2026-03-31)
**Project**: IMCG Fast Allocations
**Planning Repo**: cz.imcg.fast.allocations

---

## Executive Summary

**Project Status**: All planned work complete across all repositories

- **Sprints 1–3** (Migration Remediation): 18/19 stories complete, 1 cancelled — allocation pipeline running end-to-end with full error handling, BAL logging, PHASE3 active
- **Sprint 4** (Logger Architecture): 8/8 stories complete — multi-language BAL logging, configurable bgRFC/BAL parameters
- **Post-Sprint-4 Continuous Flow** (EST-99 → EST-161): All backlog items resolved — APJ lifecycle, allocation dashboard, breakpoint API, bgRFC utilities, stale-state bug fixes
- **ZEN_ORCH Phase 1** (Epics 1–6): Core orchestration engine complete — score/performance model, adapter framework, engine sweep, APJ scheduling, health checks, unit tests all GREEN
- **ZEN_ORCH Phase 2** (Epic 7 — BSR): Business State Registry complete — collision detection, PREREQ_GATE, full unit test coverage (21/21 GREEN)
- **ZEN_ORCH Phase 3** (Epics 8–9 — Dashboard + Planner Adapter): Fiori Elements dashboard, RAP BDEF with actions, and `ZCL_FI_ALLOC_ORCH_ADAPTER` complete
- **Active Backlog**: 0 items

---

## Repository State

### Planning Repository: cz.imcg.fast.allocations
- **Status**: Active planning hub
- **Branch**: master

**Contains**:
- Constitution v1.0.2 (source of truth — last amended 2026-04-10)
- All sprint planning artifacts (Sprints 1–4 + ZEN_ORCH Phases 1–3)
- Sprint status tracking (canonical: `_bmad-output/implementation-artifacts/sprint-status.yaml`)
- BMAD configuration and workflows
- Cross-repository documentation

---

### Framework Repository: cz.imcg.fast.planner
- **Status**: Active
- **Branch**: main
- **Remote**: https://github.com/smolikzd/cz.imcg.fast.planner.git

**Recent Changes** (last notable commits):
```
02dbd00  EST-161: Fix stale ended_at on process restart
450a137  EST-157: bgRFC destination suspend/resume utility
09580d8  EST-143: promote check_parallel_limit() to public; add APJ limit check
7d29aae  EST-144: configurable flush interval in zcl_fi_process_logger
3590d4b  EST-136: RESTREQ bug fix — cancel/supersede before create
```

**Component Inventory**:
- 94+ DDIC objects (tables, structures, types, domains)
- 25+ framework classes
- bgRFC handlers for background processing
- APJ (Application Jobs) infrastructure: job base class, job catalog
- Process orchestration engine with APJ lifecycle
- BAL logging via second DB connection
- bgRFC destination suspend/resume utility

**Key Framework Classes**:
- `ZCL_FI_PROCESS_MANAGER` — Process orchestration singleton
- `ZCL_FI_PROCESS_INSTANCE` — Instance lifecycle management (incl. APJ statuses + SUPERSEDED)
- `ZCL_FI_PROCESS_DEFINITION` — Process type configuration
- `ZCL_FI_PROCESS_STEP_BASE` — Step base class
- `ZCL_FI_PROCESS_JOB_BASE` — APJ job base class (Mode 3)
- `ZCL_FI_PROCESS_LOGGER` / `ZIF_FI_PROCESS_LOGGER` — BAL logging interface
- `ZCL_FI_BGRFC_*` — Background RFC handlers

---

### Implementation Repository: cz.imcg.fast.ovysledovka
- **Status**: Active
- **Branch**: main
- **Remote**: https://github.com/smolikzd/cz.imcg.fast.ovysledovka.git

**Recent Changes** (last notable commits):
```
594aae2  ZEN-9: ZCL_FI_ALLOC_ORCH_ADAPTER (start/get_status/cancel/restart/get_result/get_detail_link)
0eb39da  EST-129: Allocation APJ job catalogs and templates
e52dbac  Fail-hard logger hardening
125dd91  EST-145/146: server-side status guards + gc_status constants in dashboard
e18dcf3  EST-140: add CATCH cx_root in dashboard saver
```

**Component Inventory**:
- 3 packages: `zfi_alloc_process`, `zfi_ui`, `zfi_common`
- 5 allocation step classes (implementing `ZIF_FI_PROCESS_STEP`)
- 2 APJ job classes: `ZCL_FI_ALLOC_JOB_ALLOC`, `ZCL_FI_ALLOC_JOB_EXPORT`
- 2 APJ job catalogs + templates (SAJC/SAJT)
- Fiori Elements allocation dashboard (list report + object page)
- RAP unmanaged BDEF with 5 actions (Execute, Cancel, Supersede, Restart, CreateAndExecute)
- Dashboard query providers: `ZCL_FI_ALLOC_DASH_QUERY`, `ZCL_FI_ALLOC_DASH_STEP_QRY`
- ZEN_ORCH planner adapter: `ZCL_FI_ALLOC_ORCH_ADAPTER`

**Allocation Step Classes**:
- `ZCL_FI_ALLOC_STEP_INIT` — Process initialization
- `ZCL_FI_ALLOC_STEP_PHASE1` — Phase 1 allocation logic
- `ZCL_FI_ALLOC_STEP_PHASE2` — Phase 2 allocation logic (bgRFC queue mode)
- `ZCL_FI_ALLOC_STEP_PHASE3` — Phase 3 allocation logic
- `ZCL_FI_ALLOC_STEP_CORR_BCHE` — Correction batch processing (serial)

---

### Orchestration Engine Repository: cz.en.orch
- **Status**: Active
- **Branch**: main (local: `/Users/smolik/DEV/cz.en.orch`)

**Recent Changes** (last notable commits):
```
a73be2d  ZEN-9.4: health check ALLOC_ADAPTER phase 14 — final fix
d3a206a  ZEN-9.4: check_alloc_adapter (Phase 14) added to ZCL_EN_ORCH_HEALTH_CHK_QUERY
af506cf  ZEN-9.3: ZEN_ORCH_SETUP.prog.abap — idempotent adapter + score registration
5ae15be  ZEN-8.5: ZEN_ORCH_UI_PERF + ZEN_ORCH_UI_PERF_O4 service binding
16fd742  ZEN-8.4: BDEF ZEN_ORCH_C_PERF + ZCL_EN_ORCH_BP_PERF behavior pool
```

**Component Inventory** (package `ZEN_ORCH`):
- DDIC: 2 domains (`ZEN_ORCH_STATUS`, `ZEN_ORCH_ELEM_TYPE`), 10+ data elements, 4 database tables, multiple structures and table types
- Database tables: `ZEN_ORCH_SCORE`, `ZEN_ORCH_SCORE_STEP`, `ZEN_ORCH_PERF`, `ZEN_ORCH_P_STEP`, `ZEN_ORCH_BSR`, `ZEN_ORCH_ADAPTER_REG`
- Engine singleton: `ZCL_EN_ORCH_ENGINE`
- Adapter framework: `ZIF_EN_ORCH_ADAPTER`, `ZCL_EN_ORCH_ADAPTER_FACTORY`, `ZCL_EN_ORCH_MOCK_ADAPTER`
- Error handling: `ZCX_EN_ORCH_ERROR`, message class `ZEN_ORCH`
- Logging: `ZIF_EN_ORCH_LOGGER`, `ZCL_EN_ORCH_BAL_LOGGER`
- BSR: `ZIF_EN_ORCH_BSR`, `ZCL_EN_ORCH_BSR`
- APJ job: `ZCL_EN_ORCH_SWEEP_JOB`
- Dashboard: CDS views `ZEN_ORCH_I_PERF`, `ZEN_ORCH_I_P_STEP`, `ZEN_ORCH_C_PERF`, `ZEN_ORCH_C_P_STEP`; BDEF + behavior pool `ZCL_EN_ORCH_BP_PERF`; OData V4 service binding `ZEN_ORCH_UI_PERF_O4`
- Health check: `ZCL_EN_ORCH_HEALTH_CHK_QUERY` (14 capability phases incl. ALLOC_ADAPTER)
- Setup program: `ZEN_ORCH_SETUP.prog.abap` (idempotent adapter + score registration)

---

## Sprint and Phase Completion Summary

### Sprint 1: Unblock Allocation Pipeline (6/6 stories)
**Completed**: 2026-03-08
Enable allocation process to run end-to-end in serial mode.
Stories: 1.1, 1.2, 2.2, 4.2, 4.3, 5.3 — all done.
**Outcome**: Pipeline successfully runs from INIT through CORR_BCHE without crashes.

### Sprint 2: Structural Correctness (7/8 stories, 1 cancelled)
**Completed**: 2026-03-09
Ensure proper error handling, state management, and constitution compliance.
Stories: 2.1, 3.1, 3.2, 3.3, 5.1, 5.2, 5.4 — done. Story 4.1 cancelled (feature not needed).
**Outcome**: Framework principles properly applied, error handling standardized.

### Sprint 3: Wire Phase 3 + Housekeeping (5/5 stories)
**Completed**: 2026-03-10
Activate PHASE3 step and clean up technical debt.
Stories: 6.1, 7.1, 7.2, 7.3, 7.4 — all done.
**Outcome**: PHASE3 activated, codebase cleaned, documentation improved.

### Sprint 4: Process Logger Architecture (8/8 stories)
**Completed**: 2026-03-13
Implement multi-language logging with BAL + T100 messages.
Stories: 4-1 through 4-8 — all done. Includes logger infrastructure, message class expansion
(50+ ZFI_PROCESS, 21 ZFI_ALLOC messages), BAL integration with bgRFC substeps, migration of all
5 step classes, English translations.
**Outcome**: Full BAL/SLG1 observability for all process steps.

### Post-Sprint-4 Continuous Flow (EST-99 → EST-161)
**Completed**: 2026-03-14 → 2026-04-08
Key completed items:
- **EST-99**: bgRFC error propagation unit test
- **EST-100**: Configurable bgRFC inbound destination per process type
- **EST-101**: Configurable BAL object/subobject per process type
- **EST-102**: Parameter hash duplicity check (prevents duplicate instances)
- **EST-106**: Configurable max parallel instances per process type
- **EST-110**: SUPERSEDED status + COMPLETED duplicate block
- **EST-116**: Step-level breakpoint API (planner + dashboard UI) — 2026-03-31 / 2026-04-01
- **EST-121**: APJ background execution & lifecycle (6 sub-stories)
- **EST-125**: Fix export step BAL logging
- **EST-127**: APJ job base class (Mode 3 framework) + safety-net log + assign-to-appl-job
- **EST-129**: Allocation APJ job catalogs and templates
- **EST-132**: Allocation dashboard (Fiori Elements list report + object page)
- **EST-134**: Process management actions on dashboard (Execute/Cancel/Supersede/Restart)
- **EST-136**: Dashboard lifecycle actions + RESTREQ bug fix (CreateAndExecute)
- **EST-138**: Dashboard duration visibility — 2026-04-01
- **EST-139 → EST-147**: Backlog clearance (2026-03-31)
- **EST-150/151/152**: Dashboard filter UX & action confirmations — 2026-04-01
- **EST-157**: bgRFC destination suspend/resume utility — 2026-04-03
- **EST-161**: Fix stale `ended_at` on process restart — 2026-04-08

### ZEN_ORCH Phase 1: Foundation + Engine + APJ (Epics 1–6)
**Completed**: 2026-04-06 (code review patches 2026-04-04 → 2026-04-08)
Core orchestration engine from repository bootstrap to full integration tests.
- **Epic 1**: Repository bootstrap + DDIC Foundation (domains, data elements, structures, tables)
- **Epic 2**: Error handling (`ZCX_EN_ORCH_ERROR`) + BAL logging (`ZCL_EN_ORCH_BAL_LOGGER`)
- **Epic 3**: Adapter framework (`ZIF_EN_ORCH_ADAPTER`, factory, mock adapter)
- **Epic 4**: Engine core — `create_performance`, `snapshot_score`, JSON parameter resolution, `dispatch_step`, `poll_step_status`, `evaluate_gate`, `advance_loop`, `run_until_blocked` sweep, breakpoint/paused support
- **Epic 5**: APJ sweep job + schedule-driven performance creation + cancel/restart lifecycle
- **Epic 6**: Integration test program + repository finalization; all unit tests GREEN in SAP
**Outcome**: Fully functional ZEN_ORCH engine with mock adapter, health check, unit tests.

### ZEN_ORCH Phase 2: Business State Registry — BSR (Epic 7)
**Completed**: 2026-04-10 (incl. unit test bug-fix sprint 20a12ba → a85e2be)
- **7.1**: BSR DDIC Foundation — `ZEN_ORCH_BSR` table, domains, data elements
- **7.2**: `ZIF_EN_ORCH_BSR` interface
- **7.3**: `ZCL_EN_ORCH_BSR` implementation — register/deregister/list
- **7.4**: Engine integration — collision detection on `dispatch_step`, BSR lifecycle wired to cancel/restart
- **7.5**: PREREQ_GATE engine support — `evaluate_prereq_gate` blocks on BSR key status
- **7.6**: Unit tests — 21/21 `LTCL_HEALTH_CHK_QUERY` tests GREEN (incl. post-7.6 bug-fix sprint)
**Outcome**: SHA-256 params-hash based collision detection; prerequisite gate blocks cross-performance dependencies.

### ZEN_ORCH Phase 3: Dashboard + Planner Adapter (Epics 8–9)
**Completed**: 2026-04-11 (code review patches applied same day)
- **8.1**: Timing DDIC (`ZEN_ORCH_DE_TIMESTAMP`, `STARTED_AT`/`ENDED_AT` on both tables) + engine wiring
- **8.2**: Interface CDS views `ZEN_ORCH_I_PERF` + `ZEN_ORCH_I_P_STEP` with `DurationSec`, `StatusCriticality`
- **8.3**: Consumption CDS views `ZEN_ORCH_C_PERF` + `ZEN_ORCH_C_P_STEP` with composition + UI annotations
- **8.4**: RAP BDEF (unmanaged) + behavior pool `ZCL_EN_ORCH_BP_PERF` — Cancel/Restart/Resume actions
- **8.5**: OData V4 service binding `ZEN_ORCH_UI_PERF_O4` — dashboard visible in Fiori Elements preview
- **9.1–9.2**: `ZCL_FI_ALLOC_ORCH_ADAPTER` (all 6 `ZIF_EN_ORCH_ADAPTER` methods) — handle format `ZFI_PROCESS:<instance_id>`, APJ dispatch via `request_execute_process`, status mapping ZFI_PROCESS → ZEN_ORCH
- **9.3**: Adapter registration + `ALLOC_STEP` smoke score — idempotent `ZEN_ORCH_SETUP.prog.abap`
- **9.4**: Health check Phase 14 `ALLOC_ADAPTER` — GREY if not configured, GREEN if sweep succeeds
**Outcome**: Full Fiori Elements observability dashboard for ZEN_ORCH performances; real planner adapter bridging ZEN_ORCH → ZFI_PROCESS.

---

## Active Backlog (as of 2026-04-11)

**Backlog is clear.** All items through EST-161 and ZEN_ORCH Phases 1–3 are resolved.

See `_bmad-output/implementation-artifacts/sprint-status.yaml` for full history.

---

## Deferred Work (Phase 4 — Not Yet Planned)

The following items are explicitly deferred from ZEN_ORCH Phase 3 and not yet planned:

| Item | Description |
|------|-------------|
| Step-level BSR collision locking | Block `dispatch_step` when another performance holds the same business key at step granularity (brainstorming #26) |
| `ZIF_FI_ORCH_STATE_SERVICE` | Domain state service interface for external business state queries (#23, #24) |
| `ALLOC_YEAR_REFRESH` score | Multi-company, multi-period score with LOOP + PREREQ_GATE; requires state service |
| Score management UI | CRUD Fiori app for `ZEN_ORCH_SCORE` / `ZEN_ORCH_SCORE_STEP` (currently SE16/transport) |
| `get_detail_link()` real URL | Replace placeholder with real allocation dashboard deep link in `ZCL_FI_ALLOC_ORCH_ADAPTER` |

---

## Constitution Compliance Status

**Constitution Version**: 1.0.2
**Ratified**: 2025-11-10 | **Last Amended**: 2026-04-10

| Principle | Status | Notes |
|-----------|--------|-------|
| I. DDIC-First Architecture | Compliant | All types in DDIC; `ZEN_ORCH_TT_*` prefixes in cz.en.orch |
| II. SAP Standards Compliance | Compliant | Naming conventions enforced; ABAP-Doc on all public methods |
| III. Consult SAP Documentation | Compliant | `mcp-sap-docs` consulted before implementing BALI, RAP, CDS, bgRFC |
| IV. Factory Pattern & Encapsulation | Compliant | Private constructors + `GET_INSTANCE`/`CREATE` factory methods on all framework classes |
| V. Error Handling & Observability | Compliant | `ZCX_FI_PROCESS_ERROR` (planner), `ZCX_EN_ORCH_ERROR` (ZEN_ORCH) — zero cross-dependency |

**Additional enforced standards (v1.0.2 additions)**:
- ABAP-Doc only before individual declarations in DEFINITION section; never inside method bodies; never between methods at CLASS IMPLEMENTATION level (causes "unknown comments" syntax error)
- `COMMIT WORK` only in `sweep_all` (ZEN_ORCH engine) — never inside `advance_performance` or adapter methods
- Line length ≤ 120 chars (abaplint enforced); absolute max 255 chars

---

## Technical Architecture

### ZFI_PROCESS Framework Layer (cz.imcg.fast.planner)

```
ZCL_FI_PROCESS_MANAGER (Singleton)
    ├── manages  → ZCL_FI_PROCESS_DEFINITION
    ├── creates  → ZCL_FI_PROCESS_INSTANCE
    ├── executes → ZIF_FI_PROCESS_STEP implementations
    └── schedules → APJ via request_execute_process / request_restart_process

Background Processing:
    ZCL_FI_BGRFC_HANDLER    — bgRFC queue management
    ZCL_FI_PROCESS_JOB_BASE — APJ Mode 3 base class
    ZCL_FI_PROCESS_LOGGER   — BAL write via second DB connection

Database Tables:
    ZFI_PROC_TYPE  — process type customizing (BAL obj, bgRFC dest, max parallel, duplic check)
    ZFI_PROC_DEF   — process definitions (steps sequence + business status columns)
    ZFI_PROC_INST  — process instances (runtime state + fin params + business status)
    ZFI_PROC_STEP  — step execution records
```

### Allocation Implementation Layer (cz.imcg.fast.ovysledovka)

```
Allocation Pipeline (5 steps):
    1. ZCL_FI_ALLOC_STEP_INIT       — Initializes process, creates state records
    2. ZCL_FI_ALLOC_STEP_PHASE1     — Phase 1 allocation logic
    3. ZCL_FI_ALLOC_STEP_PHASE2     — Phase 2 allocation logic (bgRFC queue mode)
    4. ZCL_FI_ALLOC_STEP_CORR_BCHE  — Correction batch processing (serial)
    5. ZCL_FI_ALLOC_STEP_PHASE3     — Phase 3 allocation logic (serial)

APJ Integration:
    ZCL_FI_ALLOC_JOB_ALLOC   — APJ job for ALLOCATIONS process type
    ZCL_FI_ALLOC_JOB_EXPORT  — APJ job for ALLOC_EXPORT process type

Fiori Allocation Dashboard:
    ZFI_I_ALLOC_DASHBOARD_CE       — Custom entity (list report)
    ZFI_I_ALLOC_DASH_STEP_CE       — Custom entity (object page — step details)
    ZCL_FI_ALLOC_DASH_QUERY        — Query provider (list report)
    ZCL_FI_ALLOC_DASH_STEP_QRY     — Query provider (object page)
    zbp_fi_alloc_dashboard_ce      — Behavior pool: 5 actions + feature control + saver

ZEN_ORCH Planner Adapter:
    ZCL_FI_ALLOC_ORCH_ADAPTER      — Implements ZIF_EN_ORCH_ADAPTER
                                     Handle: "ZFI_PROCESS:<instance_id>"
                                     Dispatch: request_execute_process (APJ, non-blocking)
                                     Status mapping: ZFI_PROCESS (N/Q/R/S/C/E/A) → ZEN_ORCH
```

### ZEN_ORCH Orchestration Engine (cz.en.orch)

```
ZCL_EN_ORCH_ENGINE (Singleton)
    ├── create_performance( iv_score_id, iv_params_json )
    ├── sweep_all( )                    — main loop; COMMIT WORK here only
    │     └── advance_performance( )    — run-until-blocked for one performance
    │           ├── dispatch_step( )    — adapter.start() + BSR register
    │           ├── poll_step_status()  — adapter.get_status()
    │           ├── evaluate_gate( )    — AND/OR logic on sibling steps
    │           ├── advance_loop( )     — LOOP/END_LOOP iteration counter
    │           └── evaluate_prereq_gate( ) — block on cross-performance BSR key
    ├── cancel_performance( iv_perf_uuid )
    ├── restart_performance( iv_perf_uuid )
    └── resume_performance( iv_perf_uuid )   — clears PAUSED breakpoint

Score model:    ZEN_ORCH_SCORE (header) + ZEN_ORCH_SCORE_STEP (steps: STEP/LOOP/END_LOOP/GATE/PREREQ_GATE)
Performance:    ZEN_ORCH_PERF (header) + ZEN_ORCH_P_STEP (runtime snapshot)
BSR:            ZEN_ORCH_BSR — SHA-256 params hash; cross-performance collision detection
Status domain:  P=Pending  R=Running  C=Completed  F=Failed  X=Cancelled  B=Paused

Dashboard:
    ZEN_ORCH_I_PERF / ZEN_ORCH_I_P_STEP     — Interface CDS views
    ZEN_ORCH_C_PERF / ZEN_ORCH_C_P_STEP     — Consumption CDS views (RAP root + child)
    ZCL_EN_ORCH_BP_PERF                      — RAP behavior pool (Cancel/Restart/Resume actions)
    ZEN_ORCH_UI_PERF_O4                      — OData V4 service binding (Fiori Elements)

Adapter registry:   ZEN_ORCH_ADAPTER_REG — ADAPTER_TYPE → IMPL_CLASS mapping
APJ sweep:          ZCL_EN_ORCH_SWEEP_JOB — scheduled via APJ
Health check:       ZCL_EN_ORCH_HEALTH_CHK_QUERY — 14 capability phases incl. ALLOC_ADAPTER (Phase 14)
Setup:              ZEN_ORCH_SETUP.prog.abap — idempotent MODIFY-based adapter + score registration
```

---

## Known Issues & Technical Debt

None outstanding. All tracked backlog items resolved as of 2026-04-11.

Minor outstanding item (low risk): F17 (`@Consumption.filter.mandatory`) and F21 (`@UI.textArrangement`) from EST-132 need a brief visual check when the Fiori service binding `ZFI_UI_ALLOC_DASHBOARD_O4` is activated in SAP ADT. No code changes expected.

---

## Team & Ownership

**Project Lead**: Zdenek Smolik (smolik@imcg.cz)
**Organization**: IMCG s.r.o.
**Development Methodology**: BMAD (Business Methodology for Agile Development)
**Issue Tracking**: Linear (ENESTA team, EST-NNN convention)

**Repository Ownership**:
- Planning: smolikzd/cz.imcg.fast.allocations
- Framework: smolikzd/cz.imcg.fast.planner
- Implementation: smolikzd/cz.imcg.fast.ovysledovka
- Orchestration engine: cz.en.orch (local: `/Users/smolik/DEV/cz.en.orch`)

---

## References

- Constitution: `_bmad/_memory/constitution.md` (v1.0.2)
- Sprint Status: `_bmad-output/implementation-artifacts/sprint-status.yaml`
- Epic Tracking (Alloc Remediation): `_bmad-output/planning-artifacts/epics-alloc-remediation.md`
- Epic Tracking (ZEN_ORCH Phase 1): `_bmad-output/planning-artifacts/epics.md`
- Epic Tracking (ZEN_ORCH Phase 2): `_bmad-output/planning-artifacts/epics-zen-phase2.md`
- Epic Tracking (ZEN_ORCH Phase 3): `_bmad-output/planning-artifacts/epics-zen-phase3.md`
- Architecture Design: `_bmad-output/planning-artifacts/architecture.md`
- Repository Registry: `repos/registry.md`
- Workflow Guide: `docs/CROSS-REPO-WORKFLOW.md`
