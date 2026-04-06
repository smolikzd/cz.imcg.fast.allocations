---
stepsCompleted: [1, 2, 3, 4]
inputDocuments: []
session_topic: 'Enhance lifecycle control of ZFI_PROCESS planner process instances'
session_goals: 'Generate ideas for clean unambiguous lifecycle action semantics; define what cancelling, restarting and superseding mean in practice at the technical/framework level; identify edge cases and implementation options for background execution'
selected_approach: 'ai-recommended'
techniques_used: ['Question Storming', 'Morphological Analysis', 'Chaos Engineering']
ideas_generated: ['Generic APJ Job Template', 'Request/Process Split Pattern', 'Cooperative Cancellation via Status Check', 'Cancel as Cleanup Gateway', 'Timestamp-Based Immediate Start', 'Two New Fields on ZFI_PROC_INST', 'Status Reconciliation (Open)', 'Check Status Action', 'Job-Side Status Guard', 'Cancel is Fire-and-Forget Cleanup', 'Schedule-First Status-Second', 'Background Restart Reuses Existing Mechanics', 'Job References Always Reflect Current Job', 'Job References Persist After Completion', 'Requested Counts as Running for Parallel Limits', 'Requested Statuses Are Active for Duplicate Detection']
session_active: false
workflow_completed: true
session_closed: true
closed_date: '2026-04-01'
context_file: ''
---

# Brainstorming Session Results

**Facilitator:** Zdenek
**Date:** 2026-03-17

## Session Overview

**Topic:** Enhance lifecycle control of ZFI_PROCESS planner process instances
**Goals:** Generate ideas for clean, unambiguous lifecycle action semantics; define what cancelling, restarting and superseding mean in practice at the technical/framework level; identify edge cases and implementation options for background execution

### Session Setup

Technical/architectural brainstorming — UX is out of scope. Focus is on framework-level semantics and behavior of lifecycle actions: background execution, cancellation, restart, and supersede.

## Technique Selection

**Approach:** AI-Recommended Techniques
**Analysis Context:** Process instance lifecycle control with focus on clean action semantics, edge cases, and implementation options

**Recommended Techniques:**

- **Question Storming:** Surface all unknowns and edge cases before defining semantics — map the question landscape first
- **Morphological Analysis:** Decompose lifecycle control into key parameters (state, action, mode, commitment level, concurrency) and systematically explore all valid combinations
- **Chaos Engineering:** Deliberately stress-test the designed model against failure scenarios to harden the design

**AI Rationale:** Complex state-machine design benefits from question-first exploration (prevents premature decisions), followed by systematic parameter decomposition (ensures completeness), concluded by adversarial stress-testing (ensures robustness).

---

## Technique Execution

### Technique 1: Question Storming

**Purpose:** Surface all unknowns and edge cases before defining lifecycle semantics.

#### Codebase Context Gathered

Before generating questions, the actual framework source was analyzed:

- **restart()** at `zcl_fi_process_instance.clas.abap:1816` — a thin 20-line wrapper that finds the first FAILED step via `READ TABLE mt_steps WITH KEY status = gc_status-failed`, delegates to `execute(iv_start_from_step = <failed_step>)`, step status reset happens implicitly in `execute_step()` -> `update_step_status(RUNNING)`, `retry_count` is incremented on each restart.
- **execute()** step loop at line 714 — iterates over steps, calls `execute_step()` for each.
- **Status guard** at lines 632-635 — validates preconditions before execution.
- **CL_APJ_RT_API** — `schedule_job` requires a job template and returns jobname + jobcount; `get_job_status` polls by jobname/jobcount; `cancel_job` cancels running or deletes not-yet-started jobs; `start_immediately` does implicit COMMIT WORK (cannot be called from within RAP BO — must use timestamp-based start).
- **Existing DDIC domain values:** NEW (0001), RUNNING (0002), COMPLETED (0003), FAILED (0004), CANCELLED (0005), PENDING (0006), SKIPPED (0007), SUPERSEDED (0008).

#### Questions Generated

**Background Execution:**
1. How does the APJ job class discover which process instance to execute? (Answer: pass instance GUID as job parameter)
2. What happens if the APJ job starts but the instance status has already changed? (Led to: Job-Side Status Guard idea)
3. Should background execution use the same `execute()` method or a separate path? (Answer: same method, the job class is just a thin wrapper)
4. How do we handle the COMMIT WORK constraint from `start_immediately`? (Answer: use timestamp-based start with near-future timestamp)
5. Where do we store the APJ job reference (jobname/jobcount)? (Led to: Two New Fields on ZFI_PROC_INST idea)

**Cancellation:**
6. What does "cancel" mean for a RUNNING instance in background? (Led to: Cancel as Cleanup Gateway idea)
7. Can we guarantee the background job actually stops? (Answer: no — cancel_job is best-effort; led to cooperative cancellation pattern)
8. Should cancel be synchronous or fire-and-forget? (Decision: fire-and-forget — cancel always succeeds from the caller's perspective)
9. What if the job finishes (success or failure) between the cancel request and the actual cancellation? (Led to: Job-Side Status Guard resolves this)
10. Should CANCELLED be a terminal state or allow restart? (Decision: allow restart from CANCELLED)

**Restart:**
11. Does background restart need any new orchestration vs. foreground restart? (Answer: zero — `restart()` -> `execute()` path is identical, just wrapped in APJ job class)
12. What status should a restart-requested-but-not-yet-started instance be in? (Led to: RESTART_REQUESTED status idea)
13. Should we allow restarting a CANCELLED instance? (Decision: yes — CANCELLED means "stopped by user", not "permanently broken")

**Supersede:**
14. Can a RUNNING instance be superseded? (Decision: no — only COMPLETED instances can be superseded; running must be cancelled first)
15. What happens to the superseded instance's data? (Answer: preserved as-is, just status changes)

**Guards & Safety:**
16. Should *_REQUESTED statuses count as "active" for the parallel instance limit? (Decision: yes — prevents over-scheduling)
17. Should *_REQUESTED statuses block duplicate detection? (Decision: yes — prevents duplicate scheduling race conditions)
18. What if the APJ infrastructure is down when we try to schedule? (Answer: schedule_job raises exception, we don't change status — schedule-first-status-second pattern)

### Technique 2: Morphological Analysis

**Purpose:** Decompose lifecycle control into key parameters and systematically explore all valid combinations.

#### Parameters Identified

| Parameter | Values |
|---|---|
| **Action** | Execute, Request Execute, Restart, Request Restart, Cancel, Supersede, Check Status |
| **Current Status** | NEW, EXEC_REQUESTED, RUNNING, COMPLETED, FAILED, CANCELLED, RESTART_REQUESTED, SUPERSEDED |
| **Execution Mode** | Foreground (inline), Background (APJ) |
| **Commitment** | Immediate (execute now), Deferred (schedule for background) |

#### New Statuses Identified

Two new statuses are needed beyond existing DDIC domain values:

- **EXEC_REQUESTED** — instance scheduled for background execution, job not yet started
- **RESTART_REQUESTED** — failed/cancelled instance scheduled for background restart, job not yet started

These map to the existing PENDING (0006) concept but need distinct semantics. Decision: add two new domain values to `ZFI_DOM_PROC_INST_STATUS`.

#### State Transition Matrix (Final)

| Current Status | Execute | Request Exec | Restart | Request Restart | Cancel | Supersede | Check Status |
|---|---|---|---|---|---|---|---|
| **NEW** | -> RUNNING | -> EXEC_REQ | - | - | - | - | - |
| **EXEC_REQUESTED** | - | - | - | - | -> CANCELLED | - | reconcile |
| **RUNNING** | - | - | - | - | -> CANCELLED | - | reconcile |
| **COMPLETED** | - | - | - | - | - | -> SUPERSEDED | - |
| **FAILED** | - | - | -> RUNNING | -> RESTART_REQ | -> CANCELLED | - | - |
| **CANCELLED** | - | - | -> RUNNING | -> RESTART_REQ | - | - | - |
| **RESTART_REQUESTED** | - | - | - | - | -> CANCELLED | - | reconcile |
| **SUPERSEDED** | - | - | - | - | - | - | - |

**Key design decisions from the matrix:**
- Execute and Request Execute are mutually exclusive (same action, different mode)
- Restart and Request Restart are mutually exclusive (same action, different mode)
- Cancel is allowed from any "active" status (EXEC_REQ, RUNNING, RESTART_REQ, FAILED)
- Supersede only from COMPLETED (strict)
- Check Status only meaningful for statuses with active/pending background jobs
- CANCELLED allows restart (user-initiated stop is recoverable)
- Terminal states: SUPERSEDED only (CANCELLED is recoverable)

#### APJ Integration Pattern

```
Request Execute (foreground caller):
  1. schedule_job() → get jobname/jobcount    ← schedule-first
  2. Update instance: status=EXEC_REQUESTED,  ← status-second
     job_name, job_count
  3. COMMIT WORK

APJ Job Class (background):
  1. Read instance by GUID
  2. Status guard: if status ≠ EXEC_REQUESTED → abort silently
  3. Call instance.execute()                   ← reuses existing method
  4. (status transitions to RUNNING → COMPLETED/FAILED happen inside execute)
```

```
Request Restart (foreground caller):
  1. schedule_job() → get jobname/jobcount
  2. Update instance: status=RESTART_REQUESTED,
     job_name, job_count
  3. COMMIT WORK

APJ Job Class (background):
  1. Read instance by GUID
  2. Status guard: if status ≠ RESTART_REQUESTED → abort silently
  3. Call instance.restart()                   ← reuses existing method
  4. (status transitions handled by restart → execute internally)
```

### Technique 3: Chaos Engineering

**Purpose:** Stress-test the designed lifecycle model against failure scenarios.

#### Scenario 1: Job starts after cancel
**Sequence:** User requests execute → job scheduled → user cancels → job starts
**Resolution:** Job-side status guard checks status on entry. Status is CANCELLED (not EXEC_REQUESTED), so job aborts silently. Safe.

#### Scenario 2: Job infrastructure failure during schedule
**Sequence:** User requests execute → `schedule_job()` raises exception
**Resolution:** Schedule-first-status-second pattern. Status never changed from NEW. No cleanup needed. User sees error, can retry. Safe.

#### Scenario 3: Double-schedule race condition
**Sequence:** Two users simultaneously request execute for same instance
**Resolution:** First caller: `schedule_job()` succeeds, updates status to EXEC_REQUESTED. Second caller: status guard rejects (status is no longer NEW). The guard check + update must be atomic (single DB update with WHERE status = 'NEW'). Safe.

#### Scenario 4: Job completes but status update fails
**Sequence:** Job runs execute() successfully → COMMIT WORK for status update to COMPLETED fails
**Resolution:** This is handled by existing framework — execute() and status updates are in the same LUW. If commit fails, both the business logic and status rollback. The job will appear as failed in APJ. `check_status` reconciliation would detect the mismatch. Acceptable — same risk as foreground execution.

#### Scenario 5: Cancel called on already-completed instance
**Sequence:** Instance completes in background → user (stale UI) clicks cancel
**Resolution:** Cancel's status guard only allows cancel from EXEC_REQUESTED, RUNNING, RESTART_REQUESTED, FAILED. COMPLETED is not in that list. Cancel is rejected with clear error message. Safe.

#### Scenario 6: Network partition — job running but unreachable
**Sequence:** Background job is running → network issues → `cancel_job()` called but can't reach job
**Resolution:** Cancel is fire-and-forget. Instance status set to CANCELLED regardless. If the job eventually completes, it will try to update status but the job-side guard (if implemented at commit time) would detect the mismatch. Alternatively, `check_status` can reconcile later. Acceptable risk — same as any distributed system.

#### Scenario 7: Restart of cancelled instance that still has a running job
**Sequence:** Cancel was fire-and-forget → old job still running → user restarts
**Resolution:** Request Restart schedules a NEW job, overwrites job_name/job_count fields. Old job (if it's still running) hits the status guard on its next check — status is RESTART_REQUESTED, not what it expects. Old job aborts. New job starts, sees RESTART_REQUESTED, proceeds. Safe — job reference fields always reflect the "current" job.

#### Scenario 8: APJ job template missing or misconfigured
**Sequence:** User requests execute → `schedule_job()` fails because template doesn't exist
**Resolution:** Same as Scenario 2. Schedule-first means no status change. Error propagated to caller. Configuration issue, not a design flaw. Safe.

#### Scenario 9: check_status called on instance with no job reference
**Sequence:** Instance is FAILED (from foreground execution, no job reference) → user calls check_status
**Resolution:** check_status sees no job_name/job_count → returns current instance status as-is, no reconciliation needed. Could log info message. Safe.

#### Scenario 10: Massive parallel scheduling — 100 instances at once
**Sequence:** Batch operation schedules 100 background executions simultaneously
**Resolution:** Each schedule_job is independent. APJ infrastructure handles queuing. Parallel instance limit guard (EST-106) counts EXEC_REQUESTED as active, so the guard properly limits how many can be scheduled per process type. Safe.

#### Scenario 11: Instance stuck in EXEC_REQUESTED — job never starts
**Sequence:** Job scheduled but APJ infrastructure never picks it up (system issue)
**Resolution:** `check_status` reconciliation detects this — calls `get_job_status()`, sees job is still queued or has been cancelled by system. Can either: (a) leave as-is and report to user, or (b) automatically transition to FAILED if job is confirmed dead. Design decision: check_status reports but does NOT auto-transition — let the user decide whether to cancel or wait. Safe.

---

## Idea Organization

### Theme 1: APJ Integration Pattern

The foundational technical pattern for running process instances as background jobs.

| # | Idea | Description |
|---|---|---|
| 1 | **Generic APJ Job Template** | One job template + one job class (`ZCL_FI_PROCESS_JOB`) for all process types. Job class implements `IF_APJ_DT_EXEC_OBJECT`. Receives instance GUID as parameter. No per-process-type proliferation. |
| 5 | **Timestamp-Based Immediate Start** | Use `schedule_job` with a near-future timestamp (NOW + 1 second) instead of `start_immediately`, because `start_immediately` does implicit COMMIT WORK and cannot be called from within RAP BO context. |
| 6 | **Two New Fields on ZFI_PROC_INST** | Add `JOB_NAME` (CHAR 32) and `JOB_COUNT` (CHAR 8) fields to the process instance table. These store the APJ job reference for background-executed instances. |
| 14 | **Job References Persist After Completion** | Job reference fields (job_name, job_count) are NOT cleared when a job completes. This allows post-mortem investigation via `check_status` or APJ job logs. Only cleared explicitly by `cancel`. |

### Theme 2: Request/Execute Separation

The pattern for cleanly separating the scheduling request from the actual execution.

| # | Idea | Description |
|---|---|---|
| 2 | **Request/Process Split Pattern** | Foreground caller only schedules (request_execute, request_restart). Background job actually runs (execute, restart). Clean separation of concerns. Caller never blocks. |
| 11 | **Schedule-First Status-Second** | Always call `schedule_job()` BEFORE updating instance status. If scheduling fails, status is unchanged — no broken intermediate states. Atomic from the caller's perspective. |
| 12 | **Background Restart Reuses Existing Mechanics** | `restart()` method already works as a thin wrapper over `execute(iv_start_from_step)`. Background restart just wraps this in the APJ job class — zero new orchestration code needed. |
| 13 | **Job References Always Reflect Current Job** | When a new job is scheduled (e.g., restart after cancel), job_name/job_count fields are overwritten. Old job (if still running) hits the status guard and aborts. No stale references. |

### Theme 3: Cancellation Semantics

Defining what "cancel" means at every level.

| # | Idea | Description |
|---|---|---|
| 4 | **Cancel as Cleanup Gateway** | Cancel is the universal "stop and clean up" action. Works on EXEC_REQUESTED, RUNNING, RESTART_REQUESTED, and FAILED. Always transitions to CANCELLED. |
| 10 | **Cancel is Fire-and-Forget Cleanup** | Cancel always succeeds from the caller's perspective. It sets status to CANCELLED and calls `cancel_job()` best-effort. If the job can't be cancelled (already finished, unreachable), that's fine — the status guard handles it. |

### Theme 4: Safety and Reconciliation

Patterns for maintaining consistency in a distributed (foreground + background) system.

| # | Idea | Description |
|---|---|---|
| 3 | **Cooperative Cancellation via Status Check** | Background jobs cannot be forcefully killed mid-ABAP-statement. Instead, the job checks instance status at safe points (between steps). If status is CANCELLED, it stops cooperatively. |
| 8 | **Check Status Action** | On-demand reconciliation action. Reads APJ job status via `get_job_status()` and compares with instance status. Reports discrepancies. Does NOT auto-transition — user decides. |
| 9 | **Job-Side Status Guard** | First thing the APJ job class does: read instance status. If status doesn't match what's expected (EXEC_REQUESTED for execute, RESTART_REQUESTED for restart), abort silently. Prevents stale/cancelled jobs from executing. |

### Theme 5: Guard Extensions

Extending existing safety mechanisms to cover new statuses.

| # | Idea | Description |
|---|---|---|
| 15 | **Requested Counts as Running for Parallel Limits** | EST-106 (MAX_PARALLEL_INSTS) must count EXEC_REQUESTED and RESTART_REQUESTED as "active" instances. Prevents over-scheduling beyond the configured limit. |
| 16 | **Requested Statuses Active for Duplicate Detection** | EST-102 (parameter hash duplicity check) must treat EXEC_REQUESTED and RESTART_REQUESTED as active. Prevents duplicate scheduling race conditions. |

### Prioritization Tiers

**Tier 1 — Foundation (must implement first):**
- #6 Two New Fields on ZFI_PROC_INST (DDIC change)
- #1 Generic APJ Job Template + Job Class
- #5 Timestamp-Based Immediate Start
- #9 Job-Side Status Guard
- #11 Schedule-First Status-Second

**Tier 2 — Core Actions (depends on Tier 1):**
- #2 Request/Process Split (request_execute, request_restart methods)
- #12 Background Restart Reuses Existing Mechanics
- #4 + #10 Cancel semantics (fire-and-forget cleanup)
- #13 Job References Always Reflect Current Job
- #14 Job References Persist After Completion

**Tier 3 — Safety Layer (depends on Tier 2):**
- #3 Cooperative Cancellation via Status Check (between-step guard)
- #8 Check Status Action (on-demand reconciliation)
- #15 + #16 Guard Extensions (EST-102 and EST-106 updates)

### Implementation Dependency Chain

```
DDIC: New domain values (EXEC_REQUESTED, RESTART_REQUESTED) + new fields (JOB_NAME, JOB_COUNT)
  │
  ├─> Job Template + Job Class (ZCL_FI_PROCESS_JOB)
  │     │
  │     ├─> Job-Side Status Guard (inside job class)
  │     │
  │     └─> request_execute() method on zcl_fi_process_instance
  │           │
  │           ├─> request_restart() method (reuses restart → execute path)
  │           │
  │           └─> cancel() method update (add cancel_job call + clear job fields)
  │                 │
  │                 └─> check_status() method (reconciliation)
  │
  └─> Guard extensions: EST-102 + EST-106 (can be done in parallel with above)
```

---

## Session Summary

**Session Duration:** Extended multi-round session
**Techniques Used:** Question Storming, Morphological Analysis, Chaos Engineering
**Ideas Generated:** 16
**Themes Identified:** 5
**Chaos Scenarios Tested:** 11 (all resolved)

### Key Design Decisions

1. **One generic job template + one generic job class** — no per-process-type proliferation
2. **Schedule-first, status-second** — no broken intermediate states if scheduling fails
3. **Cancel is universal fire-and-forget cleanup** — always succeeds, calls cancel_job best-effort, clears job fields
4. **check_status is on-demand reconciliation** — no periodic polling, does not auto-transition
5. **Job references persist on instance** — overwritten on new request, cleared only by cancel
6. **EXEC_REQUESTED and RESTART_REQUESTED count as "active"** for both parallel limit and duplicate detection guards
7. **CANCELLED is recoverable** — allows restart (user-initiated stop is not permanent)
8. **SUPERSEDED is the only true terminal state**
9. **Job-side status guard** prevents stale/cancelled jobs from executing
10. **Background restart requires zero new orchestration** — reuses existing restart() → execute() path

### Revised State Machine

```
NEW ──execute()──────────> RUNNING
NEW ──request_execute()──> EXEC_REQUESTED ──(job starts)──> RUNNING
RUNNING ──(success)──────> COMPLETED
RUNNING ──(failure)──────> FAILED
RUNNING ──cancel()───────> CANCELLED
EXEC_REQUESTED ──cancel()──> CANCELLED
RESTART_REQUESTED ──cancel()──> CANCELLED
FAILED ──restart()───────> RUNNING
FAILED ──request_restart()──> RESTART_REQUESTED ──(job starts)──> RUNNING
FAILED ──cancel()────────> CANCELLED
CANCELLED ──restart()────> RUNNING
CANCELLED ──request_restart()──> RESTART_REQUESTED ──(job starts)──> RUNNING
COMPLETED ──supersede()──> SUPERSEDED
```

### Next Steps

1. **Create a technical specification** from this brainstorming output — formalize the state machine, DDIC changes, method signatures, and APJ integration pattern into an implementable spec
2. **Update the constitution** if new principles emerge from the design (e.g., background job patterns)
3. **Create epic/stories** in the sprint plan for the 3 implementation tiers
4. **Implement Tier 1** (DDIC + job template + job class) in the planner repository first
