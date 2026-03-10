---
story: "7.4"
title: "Document cleanup_old_instances commit intent"
epic: "7 — Code Quality and Housekeeping"
sprint: 3
status: ready-for-dev
priority: low
estimate: 30min
source_findings:
  - LUW-03
  - luw-schema-analysis.md
target_repo: smolikzd/cz.imcg.fast.planner
dependencies: []
---

# Story 7.4: Document cleanup_old_instances Commit Intent

## User Story

As a framework developer,
I want the commit behaviour of `cleanup_old_instances` to be explicit and documented,
so that callers know whether they must issue a `COMMIT WORK` after calling it.

## Background

`ZCL_FI_PROCESS_MANAGER->cleanup_old_instances` performs DELETE operations on
old process instances but does not:
1. Explicitly commit the changes with `COMMIT WORK`, OR
2. Document that the caller must commit

This ambiguity creates risks:
- **If callers assume auto-commit**: Deletes may remain uncommitted, leading to
  memory leaks (process instances not cleaned up)
- **If callers commit unnecessarily**: Double commits may occur if the method
  later adds `COMMIT WORK`

Best practice for framework methods that modify data:
- **Option A**: Auto-commit — method issues `COMMIT WORK AND WAIT` at the end
- **Option B**: Caller responsibility — method ABAP-Doc clearly states "Caller must commit"

Either option is acceptable, but the current ambiguity is not.

## Acceptance Criteria

**AC-01 — Commit behavior explicit:**
Given `ZCL_FI_PROCESS_MANAGER->cleanup_old_instances` is called
When the method completes
Then either:
- A `COMMIT WORK AND WAIT` is added at the end (if auto-commit is intended), OR
- An ABAP-Doc comment clearly states "Caller must commit" (if deferred commit is intended)

**AC-02 — Callers updated if needed:**
Given the commit behavior is changed to auto-commit (Option A)
When existing callers are reviewed
Then any caller that issues `COMMIT WORK` after calling `cleanup_old_instances`
is updated to avoid double commit

## Tasks

- [ ] **Task 7.4.1**: Read `ZCL_FI_PROCESS_MANAGER->cleanup_old_instances` method
  to understand current implementation

- [ ] **Task 7.4.2**: Grep repository for all callers of `cleanup_old_instances`:
  ```bash
  grep -rn "cleanup_old_instances" /path/to/repo/src/
  ```

- [ ] **Task 7.4.3**: Analyze each caller to determine if they issue `COMMIT WORK`
  after calling the method

- [ ] **Task 7.4.4**: Determine design intent:
  - **If 0 callers commit**: Add `COMMIT WORK AND WAIT` to method (Option A)
  - **If all callers commit**: Add ABAP-Doc stating "Caller must commit" (Option B)
  - **If mixed**: Consult with team or choose Option A for consistency

- [ ] **Task 7.4.5**: Implement chosen approach:
  - **Option A**: Add `COMMIT WORK AND WAIT` at end of method
  - **Option B**: Add ABAP-Doc comment at method signature

- [ ] **Task 7.4.6**: If Option A chosen, update callers to remove redundant `COMMIT WORK`

- [ ] **Task 7.4.7**: Verify constitution compliance (Principle III: document LUW behavior)

## Dev Notes

### Option A: Auto-Commit (Recommended if few/no callers)
```abap
METHOD cleanup_old_instances.
  DATA lt_old_instances TYPE TABLE OF zfi_proc_inst.
  
  " ... existing DELETE logic ...
  
  " Commit the cleanup changes
  COMMIT WORK AND WAIT.
ENDMETHOD.
```

**When to choose**:
- Method is rarely called (low coordination cost)
- Callers don't need to batch multiple operations in one LUW
- Immediate persistence is desired

### Option B: Document Caller Responsibility (Recommended if many callers)
```abap
"! Clean up old process instances older than specified days
"! @parameter iv_days_old | Number of days threshold for cleanup
"! @parameter rv_count | Number of instances deleted
"! @raising zcx_fi_process_error | Database error during deletion
"! <p>
"! <strong>LUW Management:</strong> This method performs DELETE operations but does NOT commit.
"! The caller MUST issue COMMIT WORK after calling this method to persist the deletions.
"! </p>
METHOD cleanup_old_instances.
  " ... existing logic ...
ENDMETHOD.
```

**When to choose**:
- Multiple callers exist
- Callers need to batch cleanup with other operations in one LUW
- Flexibility is more important than convenience

### Constitution Principle III Reminder
Always consult SAP documentation for LUW and commit best practices:
- Use `mcp-sap-docs` to search for "COMMIT WORK" patterns
- Verify ABAP Cloud compatibility if applicable
- Follow SAP standard: methods that modify data should document commit behavior

### Reference Patterns

**Good: Explicit auto-commit**
```abap
METHOD delete_old_logs.
  DELETE FROM zlog_table WHERE ...
  COMMIT WORK AND WAIT.
ENDMETHOD.
```

**Good: Documented caller responsibility**
```abap
"! @parameter ... | ...
"! <strong>Note:</strong> Caller must commit after this method.
METHOD delete_old_logs.
  DELETE FROM zlog_table WHERE ...
  " No commit - caller's responsibility
ENDMETHOD.
```

**Bad: Ambiguous (current state)**
```abap
METHOD cleanup_old_instances.
  DELETE FROM zfi_proc_inst WHERE ...
  " No commit, no documentation - AMBIGUOUS
ENDMETHOD.
```

## Files to Change

| File (local repo) | Change |
|-------------------|--------|
| `src/zcl_fi_process_manager.clas.abap` | Add `COMMIT WORK AND WAIT` OR ABAP-Doc comment to `cleanup_old_instances` |
| (Potentially caller files) | Remove redundant `COMMIT WORK` if Option A chosen |

## Constitution Compliance Checklist

- [ ] Principle I — DDIC-First: Not applicable (no new types)
- [ ] Principle II — SAP Standards: ABAP-Doc format if Option B; COMMIT WORK pattern if Option A
- [ ] Principle III — Consult SAP Docs: **REQUIRED** — search for "COMMIT WORK" best practices
- [ ] Principle IV — Factory Pattern: Not applicable
- [ ] Principle V — Error Handling: Document error handling around commit if Option A chosen

## Decision Matrix

| Scenario | Current Callers | Recommendation | Reason |
|----------|----------------|----------------|---------|
| 0-1 callers | Don't commit | **Option A** (auto-commit) | Simple, no coordination needed |
| 0-1 callers | Already commit | **Option B** (document) | Callers managing LUW intentionally |
| 2+ callers | None commit | **Option A** (auto-commit) | Enforce consistency |
| 2+ callers | All commit | **Option B** (document) | Respect existing pattern |
| 2+ callers | Mixed | **Option A** (auto-commit) | Standardize behavior |

## Testing

- [ ] **Unit test**: Call `cleanup_old_instances` and verify old instances are deleted
- [ ] **LUW test**: If Option A chosen, verify commit occurs (check DB immediately after call)
- [ ] **Rollback test**: If Option B chosen, verify no commit occurs (caller can roll back)

## Dev Agent Record

### Agent Model Used

### Completion Notes

### Decision Made
- [ ] Option A: Auto-commit with `COMMIT WORK AND WAIT`
- [ ] Option B: Document caller responsibility with ABAP-Doc

### Changed Files
| File (local repo) | Commit |
|-------------------|--------|
