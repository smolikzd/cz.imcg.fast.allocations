---
stepsCompleted: [1, 2, 3, 4]
inputDocuments: []
session_topic: 'Abstract process orchestration engine for managing scenarios of heterogeneous work units across multiple frameworks'
session_goals: 'Define orchestration abstraction and work unit concept; design integration contract for framework adapters; define orchestration primitives (sequential, wait-for-all, periodic, conditional); design scenario model with parameterization; design unified state/dashboard model; validate against real-world allocation/export/workflow use cases'
selected_approach: 'ai-recommended'
techniques_used: ['Analogical Thinking', 'Morphological Analysis', 'Chaos Engineering']
ideas_generated:
  - 'Orchestration #1: Score-Based Scenario Definition'
  - 'Orchestration #2: Conductor Integration Contract (Adapter Interface)'
  - 'Orchestration #3: Performance = Runtime Execution Instance'
  - 'Orchestration #4: Instrument Sections = Adapter Types'
  - 'Orchestration #5: Rehearsal Marks = Breakpoints/Checkpoints'
  - 'Orchestration #6: Tempo Marks = Execution Constraints'
  - 'Orchestration #7: Coda/Dal Segno = Loop and Restart Primitives'
  - 'Orchestration #8: Concert Program = Unified Dashboard'
  - 'Orchestration #9: Flat Score with RefID Grouping'
  - 'Orchestration #10: JSON Parameter Contract'
  - 'Orchestration #11: Hierarchical Parameter Inheritance'
  - 'Orchestration #12: Stateless Periodic Engine (APJ Sweep)'
  - 'Orchestration #13: Run-Until-Blocked Engine'
  - 'Orchestration #14: Adapter-Decided Execution Mode'
  - 'Orchestration #15: Six Universal Statuses'
  - 'Orchestration #16: Fail-Stop Error Strategy'
  - 'Orchestration #17: Three-Level Dashboard'
  - 'Orchestration #18: Simple Schedule Table'
  - 'Orchestration #19: Copy-on-Start Score Snapshot'
  - 'Orchestration #20: Resilient Status Polling'
  - 'Orchestration #21: Performance Collision Detection'
  - 'Orchestration #22: Prerequisite Gate (PREREQ_GATE)'
  - 'Orchestration #23: Purpose-Built State Services'
  - 'Orchestration #24: Framework-Owned State, Orchestrator-Queried'
  - 'Orchestration #25: Business State Registry (BSR)'
  - 'Orchestration #26: BSR as Lock Master'
session_active: false
workflow_completed: true
context_file: ''
---

# Brainstorming Session Results

**Facilitator:** Zdenek
**Date:** 2026-04-01

## Session Overview

**Topic:** Abstract process orchestration engine for managing scenarios of heterogeneous work units across multiple frameworks

**Goals:**
1. Define the orchestration abstraction — what is a "work unit" at the orchestration level, independent of any specific framework
2. Design the integration contract (API/interface) that any framework must implement to be orchestratable
3. Define orchestration primitives: sequential chaining, wait-for-all, periodic scheduling, conditional triggers
4. Design the scenario model — how scenarios are defined, parameterized, and executed
5. Design the unified state/dashboard model — scenario progress across heterogeneous frameworks
6. Validate against real-world use cases (year refresh, period-end export, weekly export, allocate-approve-export pipeline)

### Key Design Principle

The orchestrator does NOT know what a "planner process instance" is. It knows what a "work unit" is — something that can be started, monitored for completion/failure, and potentially restarted. The planner framework is just one adapter implementing that contract.

### Real-World Use Cases

| Scenario | Pattern | Details |
|----------|---------|---------|
| **Year refresh** | Sequential chain | 48 allocation instances (4 CC x 12 periods), one by one, with breakpoints/restarts |
| **Period-end export** | Wait-for-all + trigger | Wait for 4 CC allocations to complete, then run export for that period |
| **Weekly export** | Periodic/scheduled | Export process runs on a weekly schedule |
| **Full pipeline** | Chain + wait + external | Allocate -> approve (workflow framework) -> export |

### Session Setup

Technical/architectural brainstorming focused on abstract orchestration design. The orchestrator must be framework-agnostic — any system that implements the integration contract can participate as a work unit provider. Current known frameworks: ZFI_PROCESS (planner), approval workflow system. Future frameworks should be pluggable without orchestrator changes.

### Musical Metaphor Dictionary

The session adopted a musical score/conductor metaphor throughout. This dictionary maps metaphor terms to orchestration concepts and concrete examples from the year-refresh use case.

| Metaphor | Orchestration Term | Year Refresh Example |
|----------|-------------------|----------------------|
| **Score** | Scenario definition | YEAR_REFRESH_WITH_EXPORT — template defining 48 allocations + 12 exports |
| **Performance** | Runtime execution instance | "Year refresh 2026 for OVERHEAD" — one concrete run with UUID |
| **Conductor** | Orchestration engine | Stateless periodic sweep that advances the performance through the score |
| **Instrument** | Adapter type | ALLOC_COMPUTE, ALLOC_EXPORT — each wrapping a planner process type |
| **Musician** | Work unit instance | One specific allocation run (CC 1000 / period 001 / FY 2026) |
| **Section** | Parallel group (RefID) | P1 — the 4 company codes computing in parallel for one period |
| **Bar line** | Gate | GATE P1 — wait for all 4 CCs to finish before export |
| **Repeat sign** | Loop | L1 — iterate periods 001 through 012 |
| **Tempo mark** | Execution constraint | MAX_PARALLEL on a RefID group |
| **Rehearsal mark** | Breakpoint/checkpoint | PAUSED status between periods for manual review |
| **Concert program** | Dashboard | Three levels: all performances → year refresh progress → allocation detail |
| **Sheet music** | Score snapshot | Copy of score rows frozen into performance at creation |
| **Stage manager** | Business State Registry (BSR) | Provides strategic overview, prerequisite checks, and collision locking |

---

## Technique 1: Analogical Thinking (Musical Score/Conductor)

**Core analogy:** An orchestrator is like a musical conductor. The conductor doesn't play instruments — they read a score and coordinate musicians. Similarly, the orchestration engine doesn't execute work — it reads scenario definitions and coordinates framework adapters.

### Ideas Generated

**[Orchestration #1] Score-Based Scenario Definition**
Scenarios are declarative data stored in DDIC config tables, not coded as ABAP classes. A score defines the sequence, parallelism, and gating of work units. CDS/RAP layer for management UI can be added later. This separates the "what to do" (score) from "how to do it" (adapter) from "when to do it" (engine).

**[Orchestration #2] Conductor Integration Contract (Adapter Interface)**
Framework adapters implement a 6-method interface: `start()` (returns handle + status indicating sync/async), `get_status()`, `cancel()`, `restart()`, `get_result()`, `get_detail_link()`. Any framework that implements this contract becomes orchestratable. The orchestrator never knows what's inside the adapter.

**[Orchestration #3] Performance = Runtime Execution Instance**
A "performance" is one concrete execution of a score — it has a UUID, references a score, holds runtime state (current position, step statuses, timestamps). Two persistence structures: performance header (PERF_UUID, SCORE_ID, STATUS, CURRENT_SEQ, LOOP_ITERATION, SCORE_PARAMS_JSON) and performance steps (PERF_UUID, SCORE_SEQ, LOOP_ITERATION, ADAPTER_TYPE, WORK_UNIT_HANDLE, PARAMS_JSON, STATUS, timestamps).

**[Orchestration #4] Instrument Sections = Adapter Types**
Each adapter type (ALLOC_COMPUTE, ALLOC_EXPORT, APPROVAL_WF) is registered in a registry table mapping the type key to an implementing ABAP class. The orchestrator instantiates adapters through a factory. New frameworks plug in by adding a row to the registry — no orchestrator code changes.

**[Orchestration #5] Rehearsal Marks = Breakpoints/Checkpoints**
Performances can be paused at designated points (PAUSED status). Useful for the year-refresh scenario where an operator might want to review results after each period before proceeding. Breakpoints are score elements — the engine sees them and sets performance status to PAUSED, waiting for manual resume.

**[Orchestration #6] Tempo Marks = Execution Constraints**
Parallelism limits on score groups. A RefID group with MAX_PARALLEL=2 means the engine dispatches at most 2 work units from that group concurrently, even if 4 are defined. The planner framework already has internal throttling, but this allows the orchestrator to control it at the scenario level independently.

**[Orchestration #7] Coda/Dal Segno = Loop and Restart Primitives**
LOOP/END_LOOP elements with a shared RefID define iteration boundaries. The engine tracks loop iteration count and replays the enclosed steps for each iteration. Supports the year-refresh pattern: loop over 12 periods, executing the same allocation+export pattern for each.

**[Orchestration #8] Concert Program = Unified Dashboard**
A single entry point showing all orchestration activity. Derived entirely from performance state and BSR data — no separate dashboard persistence. Users see the "concert season" (all active/recent performances) and can drill into any one.

---

## Technique 2: Morphological Analysis

Explored 10 design parameters systematically, combining options to find optimal configurations.

### Ideas Generated

**[Orchestration #9] Flat Score with RefID Grouping**
Score is a flat table with fields: SEQ (sequence number), TYPE (STEP/LOOP/END_LOOP/GATE/PREREQ_GATE), RefID (grouping key), adapter type, and parameters. Five execution rules govern behavior:
1. No RefID → sequential by SEQ order
2. Same RefID → parallel group
3. GATE + RefID → wait-for-all in group
4. LOOP/END_LOOP + RefID → iteration boundary
5. PREREQ_GATE → queries BSR for business condition

**[Orchestration #10] JSON Parameter Contract**
Parameters are passed as opaque JSON strings. The orchestrator never inspects, validates, or transforms parameter content. This keeps the orchestrator completely ignorant of business domain semantics. The adapter is responsible for parsing and interpreting its own parameters.

**[Orchestration #11] Hierarchical Parameter Inheritance**
JSON merge at three levels: score-level parameters → loop-level parameters → step-level parameters. Step wins on conflict. This eliminates repetition — common parameters (fiscal_year, allocation_id) are set once at the score level, and only varying parameters (company_code, period) are set per step.

**[Orchestration #12] Stateless Periodic Engine (APJ Sweep)**
An APJ job runs every N minutes and sweeps all active performances. All state lives in the database. The engine reads state, advances what it can, writes updated state, and exits. No in-memory state between sweeps. Fully idempotent — if a sweep crashes, the next one picks up where it left off.

**[Orchestration #13] Run-Until-Blocked Engine**
The engine runs synchronously through score steps until it hits an async dispatch (adapter returned RUNNING) or an unsatisfied gate. Then it persists state and exits. Fully synchronous scores complete in a single engine call — no waiting for the next sweep. This complements the periodic sweep: the sweep triggers run-until-blocked for each active performance.

**[Orchestration #14] Adapter-Decided Execution Mode**
`start()` returns either COMPLETED/FAILED (adapter ran the work synchronously) or RUNNING (adapter dispatched async work, engine must poll later). The adapter decides at runtime based on its own conditions (e.g., data volume, system load). The orchestrator doesn't need to know in advance whether a step is sync or async.

**[Orchestration #15] Six Universal Statuses**
PENDING (not yet started), RUNNING (in progress), COMPLETED (success), FAILED (error), CANCELLED (user/system abort), PAUSED (breakpoint). These apply uniformly to performances and individual steps. No framework-specific statuses leak into the orchestrator.

**[Orchestration #16] Fail-Stop Error Strategy**
Any step failure stops the entire performance. The performance status becomes FAILED. No partial continuation, no skip-and-continue. Recovery is explicit: fix the issue, then restart from the failed step. Simple, predictable, auditable.

**[Orchestration #17] Three-Level Dashboard**
Level 1: Concert hall — all performances with status summary. Level 2: Performance detail — score progress showing each step's status, timing, and current position. Level 3: Deep link — adapter's `get_detail_link()` opens the framework-native UI for a specific work unit.

**[Orchestration #18] Simple Schedule Table**
Periodic performances (like weekly exports) defined in a schedule config table: SCORE_ID, CRON/interval pattern, PARAMS_JSON, ACTIVE flag. The same engine job that sweeps active performances also checks the schedule table and creates new performances when due.

---

## Technique 3: Chaos Engineering

Tested 11 failure/edge-case scenarios against the design. All resolved.

### Ideas Generated

**[Orchestration #19] Copy-on-Start Score Snapshot**
When a performance is created, the score rows are copied into the performance step table. The engine executes from this snapshot, never from the live score definition. This means score edits don't affect running performances. Score versioning is free — each performance carries its own version of the score.

**[Orchestration #20] Resilient Status Polling**
If `get_status()` throws an exception, the engine treats the step as "unknown" (status unchanged), not "failed." The step is skipped in the current sweep and retried next time. This prevents transient adapter errors (network, lock contention) from permanently failing a performance.

**[Orchestration #21] Performance Collision Detection**
The engine rejects creation of a new performance if an active (non-terminal) performance already exists with the same SCORE_ID and equivalent SCORE_PARAMS_JSON. Prevents accidental duplicate runs of the same scenario.

---

## Validation: State Service Exploration

Extended the design through real-world validation of the period-end export and cross-score coordination scenarios.

### Ideas Generated

**[Orchestration #22] Prerequisite Gate (PREREQ_GATE)**
A score element type that queries a state service for business conditions. Unlike GATE (which checks steps within the current performance), PREREQ_GATE checks conditions across all performances — "have all allocations for period 001 completed, regardless of which score started them?"

**[Orchestration #23] Purpose-Built State Services**
Interface `zif_fi_orch_state_service` with `check_prerequisite()` and `get_state_overview()`. Each business domain implements its own state service with typed DDIC persistence and custom logic. Mirrors the adapter pattern — registry + factory + domain-specific implementation.

**[Orchestration #24] Framework-Owned State, Orchestrator-Queried**
Business state persistence (like the existing ZFI_ALLOC_STATE table) is owned and updated by framework implementations. The state service interface wraps this persistence for the orchestrator to query. The orchestrator reads state through the interface but never writes it.

**[Orchestration #25] Business State Registry (BSR)**
Single persistence linking business keys (CC, FY, FP, allocation ID, process type) to performance UUIDs. Serves three roles:
1. **Strategic overview** — "What is the current state of allocations for FY 2026?" Resolves UUIDs to live performance statuses.
2. **Prerequisite resolution** — "Are all CCs for period 001 allocated so export can proceed?" Queries referenced performance statuses.
3. **Collision detection** — "Can I start this work unit or is another score already running for this business key?"

The BSR holds references, not status copies. Status is always resolved from the live performance data, ensuring no stale information.

**[Orchestration #26] BSR as Lock Master**
Before dispatching a work unit, the engine checks the BSR for active entries matching the same business key. If another performance (from any score) already has a running work unit for that business key, the new dispatch is rejected. This prevents two scores from simultaneously modifying the same business data — e.g., a year-refresh and a single-period re-run both trying to allocate CC 1000 / period 005.

---

## Validation: Year Refresh Process Walkthrough

Proved the design against the concrete year-refresh scenario: recalculate allocations for FY 2026, 4 company codes, 12 periods, export after each period.

### Score: YEAR_REFRESH_WITH_EXPORT

Score-level params: `{"fiscal_year": "2026", "allocation_id": "OVERHEAD"}`

| SEQ | TYPE | RefID | ADAPTER_TYPE | PARAMS_JSON |
|-----|------|-------|-------------|-------------|
| 10 | LOOP | L1 | | `{"loop_var": "period", "from": "001", "to": "012"}` |
| 20 | STEP | P1 | ALLOC_COMPUTE | `{"company_code": "1000", "period": "{period}"}` |
| 30 | STEP | P1 | ALLOC_COMPUTE | `{"company_code": "2000", "period": "{period}"}` |
| 40 | STEP | P1 | ALLOC_COMPUTE | `{"company_code": "3000", "period": "{period}"}` |
| 50 | STEP | P1 | ALLOC_COMPUTE | `{"company_code": "4000", "period": "{period}"}` |
| 60 | GATE | P1 | | |
| 70 | STEP | | ALLOC_EXPORT | `{"period": "{period}"}` |
| 80 | END_LOOP | L1 | | |

### Execution for one iteration (period 001):

1. **SEQ 10 — LOOP L1:** Engine enters loop, sets `{period}` = "001"
2. **SEQ 20-50 — STEP P1 (x4):** Same RefID = parallel group. Engine dispatches all 4 allocations. Each adapter receives merged params: `{"fiscal_year": "2026", "allocation_id": "OVERHEAD", "company_code": "1000", "period": "001"}`
3. **SEQ 60 — GATE P1:** Wait-for-all. Engine checks statuses of SEQ 20, 30, 40, 50 in current iteration. All COMPLETED → proceed. Otherwise → blocked, retry next sweep.
4. **SEQ 70 — STEP (no RefID):** Sequential. Export runs after gate passes.
5. **SEQ 80 — END_LOOP L1:** Increment period to "002", jump back to SEQ 10.

Total: 48 allocations + 12 exports = 60 work units across 12 loop iterations.

**Key validation result:** The GATE at SEQ 60 handles within-performance synchronization. The BSR (PREREQ_GATE) would be needed only when separate scores need to coordinate across performances.

---

## Idea Organization and Prioritization

### Thematic Organization

**Theme 1: Scenario Model (the Score)** — How scenarios are defined as data
- #1 Score-Based Scenario Definition
- #6 Tempo Marks (Execution Constraints / MAX_PARALLEL)
- #7 Coda/Dal Segno (Loop and Restart Primitives)
- #9 Flat Score with RefID Grouping
- #10 JSON Parameter Contract
- #11 Hierarchical Parameter Inheritance

*Pattern: The score is a simple, flat, data-driven structure that encodes complex execution patterns (parallelism, gating, looping) through just two mechanisms: TYPE and RefID.*

**Theme 2: Integration Contracts (the Interfaces)** — How external frameworks plug in
- #2 Conductor Integration Contract (6-method adapter interface)
- #4 Instrument Sections (adapter type registry)
- #14 Adapter-Decided Execution Mode
- #22 Prerequisite Gate (PREREQ_GATE)
- #23 Purpose-Built State Services

*Pattern: Two parallel interface contracts — adapters execute work, BSR provides business state. Both use factory + registry pattern. Both are domain-aware while the orchestrator stays domain-ignorant.*

**Theme 3: Execution Engine (the Conductor)** — How the engine advances performances
- #12 Stateless Periodic Engine (APJ sweep)
- #13 Run-Until-Blocked Engine
- #15 Six Universal Statuses
- #16 Fail-Stop Error Strategy

*Pattern: The engine is deliberately simple — a stateless loop that reads state from DB, advances what it can, and exits. All intelligence is in the score structure and the adapters.*

**Theme 4: Runtime State (the Performance)** — How execution state is persisted
- #3 Performance = Runtime Execution Instance
- #5 Rehearsal Marks (Breakpoints)
- #15 Six Universal Statuses (shared with Theme 3)
- #19 Copy-on-Start Score Snapshot
- #20 Resilient Status Polling
- #21 Performance Collision Detection

*Pattern: The performance is the engine's working memory — an immutable snapshot of the score plus mutable step statuses. Defensive by design.*

**Theme 5: Business State Registry (BSR)** — Cross-performance business truth
- #24 Framework-Owned State, Orchestrator-Queried
- #25 Business State Registry (three roles: overview, prerequisite, locking)
- #26 BSR as Lock Master

*Pattern: The BSR bridges the orchestrator's technical world (UUIDs, statuses) and the business world (CC, FY, FP, allocation ID). It holds references and resolves them on demand — never stale copies.*

**Theme 6: Observability (the Concert Program)** — How users see and interact
- #8 Concert Program (Unified Dashboard)
- #17 Three-Level Dashboard
- #18 Simple Schedule Table

*Pattern: Dashboard is a read-only projection of performance state + BSR data. No separate "dashboard state" — everything derived from existing persistence.*

### Three-Layer Architecture

| Layer | Interface | Purpose |
|-------|-----------|---------|
| BSR | `zif_fi_orch_state_service` | Business truth (overview, prerequisites, locking) |
| Engine | Score tables + performance tables | Execution orchestration |
| Adapters | `zif_fi_orch_adapter` | Work execution delegation |

### Prioritization Results

**Phase 1 — Orchestrator Foundation (isolate orchestrator from work unit implementation)**

| Priority | Theme | What to build | Key ideas |
|----------|-------|---------------|-----------|
| 1st | Theme 1 (Score) | Score definition tables, score management | #1, #6, #7, #9, #10, #11 |
| 2nd | Theme 4 (Performance) | Performance tables, copy-on-start, status model | #3, #5, #15, #19, #20, #21 |
| 3rd | Theme 3 (Engine) | Run-until-blocked + periodic sweep, gate/loop/parallelism-limit logic | #12, #13, #16 |
| 4th | Theme 6 (Dashboard) | Three-level read view, schedule table | #8, #17, #18 |

Testable with a mock adapter — simple implementation returning COMPLETED (sync) or RUNNING (async, completes after N sweeps). No dependency on planner framework or BSR.

**Phase 2 — Integration Layer (connect to real frameworks and business state)**

| Priority | Theme | What to build | Key ideas |
|----------|-------|---------------|-----------|
| 5th | Theme 2 (Contracts) | Real adapter interface, adapter registry, factory | #2, #4, #14, #22, #23 |
| 6th | Theme 5 (BSR) | Business State Registry, prerequisite resolution, lock master | #24, #25, #26 |

### Open Questions for Phase 2

1. **State registration timing:** Should the adapter register in BSR before or after start()? (Deferred — depends on BSR persistence design)
2. **Performance UUID passing:** How does the engine pass perf_uuid to the adapter? Candidate: explicit parameter on adapter interface `start(iv_perf_uuid, iv_params_json)`. (Deferred — clean solution identified but not finalized)
3. **BSR persistence design:** What is the BSR's own persistence structure? Must support business key → perf_uuid mapping with active/terminal distinction. (Deferred to Phase 2 architecture)

---

## Session Summary and Insights

**Key Achievements:**
- 26 ideas generated across 3 creative techniques plus real-world validation
- Complete orchestration architecture designed from first principles using musical metaphor
- Framework-agnostic design validated against 4 real-world use cases (year refresh, period-end export, weekly export, full pipeline)
- Concrete score walkthrough for year-refresh (48 allocations + 12 exports)
- Two-phase implementation strategy with clear dependency ordering
- Business State Registry (BSR) concept discovered through validation — bridges technical orchestration and business domain

**Breakthrough Concepts:**
- **Flat score with RefID** — encodes sequential, parallel, gated, and looping execution patterns through just TYPE + RefID on a flat table. No tree structures, no nested definitions.
- **Run-until-blocked engine** — synchronous scores complete in one call; async scores park and resume on next sweep. Single engine algorithm handles both.
- **BSR as three-in-one** — strategic overview, prerequisite checking, and collision locking from one service with one persistence. Discovered late in the session through chaos engineering and real-world validation.
- **Copy-on-start snapshot** — eliminates score versioning complexity entirely. Each performance carries its own score version.

**Session Reflections:**
The musical conductor metaphor proved remarkably productive — it naturally mapped to every orchestration concept and remained coherent throughout all three techniques. The metaphor exposed the BSR concept (stage manager) as a necessary role that wasn't in the original goals. Chaos engineering was particularly effective at hardening the design — all 11 failure scenarios had clean resolutions within the existing architecture.
