# Process Logger Implementation Plan

**Sprint:** 4  
**Epic:** Exception Handling and Message Persistence  
**Created:** 2026-03-11  
**Owner:** Zdenek Smolik  
**Related:** ADR-008, Implementation Guide, Migration Guide

---

## Overview

**Goal:** Implement process logger architecture to provide multi-language message support, unified audit trail, and clean logging API for ZFI_PROCESS framework.

**Total Effort:** 40-56 hours (~1-1.5 weeks for single developer)

**Repositories:**
- `cz.imcg.fast.planner` (framework infrastructure)
- `cz.imcg.fast.ovysledovka` (step code migration)

---

## Story Breakdown

### Story 4.1: Create Logger Infrastructure (Framework)

**Target Repository:** `cz.imcg.fast.planner`  
**Effort:** 8-12 hours  
**Priority:** High (blocks other stories)  
**Depends On:** None

**Description:**
Create the core logger interface and implementation in the planner framework. This provides the foundation for all logging operations.

**Tasks:**

1. **Create Interface: ZIF_FI_PROCESS_LOGGER** (2 hours)
   - Define `message()` method signature
   - Define `log_exception_from_framework()` method
   - Define `save()` method
   - Add comprehensive interface documentation
   - Create in `src/zif_fi_process_logger.intf.abap`

2. **Create Class: ZCL_FI_PROCESS_LOGGER** (4-6 hours)
   - Implement `ZIF_FI_PROCESS_LOGGER` interface
   - Factory methods: `create_new()`, `attach_existing()`
   - BAL integration: Create log, add messages, save
   - Exception logging with chain support (PREVIOUS)
   - Edge case handling: Log failures, language fallback, null safety
   - Private method: `add_message_to_bal()`
   - Create in `src/zcl_fi_process_logger.clas.abap`

3. **Create Unit Tests** (2-4 hours)
   - Test logger creation (new and attach)
   - Test message logging (all severities)
   - Test exception logging (with chain)
   - Test BAL integration (mock or test system)
   - Test edge cases (logging failures, missing translations)
   - Create in `src/zcl_fi_process_logger.clas.testclasses.abap`

**Acceptance Criteria:**
- ✅ Interface ZIF_FI_PROCESS_LOGGER exists and is documented
- ✅ Class ZCL_FI_PROCESS_LOGGER implements interface
- ✅ Factory methods create BAL log with external number
- ✅ Messages added to BAL with correct severity
- ✅ Exceptions logged with full chain
- ✅ Unit tests pass (>80% code coverage)

**Technical Notes:**
- Use ABAP 7.58 features (inline declarations, VALUE constructor)
- Follow factory pattern (no direct NEW instantiation)
- BAL function modules: BAL_LOG_CREATE, BAL_LOG_MSG_ADD, BAL_DB_SAVE, BAL_DB_LOAD

---

### Story 4.2: Expand Message Classes (Framework & Business)

**Target Repository:** `cz.imcg.fast.planner` (ZFI_PROCESS), `cz.imcg.fast.ovysledovka` (ZFI_ALLOC)  
**Effort:** 4-6 hours  
**Priority:** High (blocks migration)  
**Depends On:** None

**Description:**
Create T100 messages for framework operations and expand allocation message class for business logging.

**Tasks:**

1. **Expand ZFI_PROCESS Message Class** (1-2 hours)
   - Create messages 001-019 (Instance lifecycle)
   - Create messages 020-039 (Step execution)
   - Create messages 040-059 (bgRFC execution)
   - Create messages 060-099 (Framework errors)
   - Create messages 996-999 (Infrastructure errors)
   - Activate message class
   - File: `src/zfi_process.msag.xml` (planner repo)

2. **Plan ZFI_ALLOC Message Numbers** (1 hour)
   - Review existing step code (grep for hard-coded texts)
   - Create inventory spreadsheet (text → message number mapping)
   - Assign numbers: INIT (001-099), PHASE1 (100-199), PHASE2 (200-299), PHASE3 (300-399), CORR (400-499), Errors (500-599)

3. **Create ZFI_ALLOC Messages (Initial Set)** (2-3 hours)
   - Create framework integration messages (001-010)
   - Create common error messages (500-510)
   - Create placeholder messages for each step range
   - Note: Full message set created during migration (Story 4.5)
   - File: `src/zfi_alloc.msag.xml` (ovysledovka repo)

**Acceptance Criteria:**
- ✅ ZFI_PROCESS has ~30 messages (001-099, 996-999)
- ✅ ZFI_ALLOC has ~20 initial messages (placeholders + common errors)
- ✅ Message inventory spreadsheet created
- ✅ All messages activated and accessible in SE91

**Technical Notes:**
- Message text max 73 characters
- Use numbered placeholders (&1, &2, &3, &4)
- Czech language initially, English translation in Story 4.6

---

### Story 4.3: Integrate Logger with Process Instance (Framework)

**Target Repository:** `cz.imcg.fast.planner`  
**Effort:** 6-8 hours  
**Priority:** High (blocks step migration)  
**Depends On:** Story 4.1

**Description:**
Integrate logger with process instance lifecycle (create/load) and step execution orchestration.

**Tasks:**

1. **Update ZCL_FI_PROCESS_STEP Parent Class** (1-2 hours)
   - Add `mo_log` protected attribute (TYPE REF TO zif_fi_process_logger)
   - Add `initialize_logger()` method
   - Add `get_logger()` helper method (optional, for null safety)
   - Update class documentation
   - File: `src/zcl_fi_process_step.clas.abap`

2. **Update ZCL_FI_PROCESS_INSTANCE** (3-4 hours)
   - Add `mo_log` private attribute
   - Update `create()` method: Create logger with instance UUID as external number
   - Update `load()` method: Attach to existing BAL log by external number
   - Add `get_logger()` public method
   - Log instance lifecycle events (created, loaded, status changed)
   - File: `src/zcl_fi_process_instance.clas.abap`

3. **Update Step Execution Orchestration** (2-3 hours)
   - Update `execute_step()` method: Initialize step logger
   - Add automatic exception logging (TRY-CATCH wrapper)
   - Add automatic step lifecycle logging (started, completed, failed)
   - Ensure logger saved after step execution
   - File: `src/zcl_fi_process_instance.clas.abap` (or wherever steps are executed)

4. **Update Unit Tests** (1 hour)
   - Test logger created in create() method
   - Test logger attached in load() method
   - Test step receives initialized logger
   - Test exception auto-logging in orchestration

**Acceptance Criteria:**
- ✅ Process instance creates logger on create()
- ✅ Process instance attaches logger on load()
- ✅ BAL external number = instance UUID
- ✅ Steps inherit mo_log from parent class
- ✅ Framework initializes step logger before execute()
- ✅ Framework auto-logs exceptions caught during step execution
- ✅ Framework auto-logs step lifecycle (start, complete, fail)
- ✅ Unit tests pass

**Technical Notes:**
- Logger must be created BEFORE any business logic executes
- Use `attach_existing()` to handle process restart scenarios
- External number must match instance UUID for correlation

---

### Story 4.4: Integrate Logger with bgRFC Substeps (Framework)

**Target Repository:** `cz.imcg.fast.planner`  
**Effort:** 4-6 hours  
**Priority:** Medium (needed for parallel execution)  
**Depends On:** Story 4.3

**Description:**
Ensure bgRFC substeps attach to parent process instance's BAL log for unified audit trail.

**Tasks:**

1. **Update Substep Context Serialization** (1-2 hours)
   - Ensure parent instance UUID included in serialized context
   - Verify UUID passed to bgRFC function module
   - File: `src/zcl_fi_process_substep_context.clas.abap` (or similar)

2. **Update Substep Execution** (2-3 hours)
   - In bgRFC work process: Deserialize context to get parent UUID
   - Attach logger using parent UUID as external number
   - Initialize substep with logger
   - Ensure logger saved after substep execution
   - File: Wherever substeps are executed in bgRFC

3. **Test Concurrent Logging** (1-2 hours)
   - Create integration test with 3+ concurrent substeps
   - Verify all substep messages in parent BAL log
   - Verify chronological ordering
   - Verify no message loss or duplication

**Acceptance Criteria:**
- ✅ Substeps attach to parent's BAL log (same external number)
- ✅ Substep messages appear in parent log (SLG1)
- ✅ Concurrent substeps don't lose messages (BAL handles locking)
- ✅ Integration test passes with parallel execution

**Technical Notes:**
- BAL handles concurrent writes via internal locking
- Each work process loads log by external number, adds messages, saves independently
- Test with actual bgRFC (not just local parallel processing)

---

### Story 4.5: Migrate Step Code (Business Logic)

**Target Repository:** `cz.imcg.fast.ovysledovka`  
**Effort:** 12-16 hours  
**Priority:** High (core migration work)  
**Depends On:** Stories 4.1, 4.2, 4.3

**Description:**
Migrate all 5 allocation steps from hard-coded string templates to T100 message-based logging.

**Tasks:**

1. **Create Full T100 Message Set** (3-4 hours)
   - Review step code, extract all hard-coded texts (~100 texts)
   - Create T100 messages per inventory spreadsheet
   - Handle long texts (split into multiple messages)
   - Assign appropriate severities
   - Activate ZFI_ALLOC message class
   - File: `src/zfi_alloc.msag.xml`

2. **Migrate zcl_fi_alloc_step_init** (1-2 hours)
   - Replace ~10-15 hard-coded texts with mo_log->message() calls
   - Remove MESSAGE ... INTO lv_dummy + mo_log->log_sy_msg() patterns
   - Test compilation and execution
   - Commit changes
   - File: `src/zfi_alloc_process/zcl_fi_alloc_step_init.clas.abap`

3. **Migrate zcl_fi_alloc_step_phase1** (2-3 hours)
   - Replace ~15-20 hard-coded texts
   - Update exception handling (remove explicit logging)
   - Test with real data
   - Commit changes
   - File: `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap`

4. **Migrate zcl_fi_alloc_step_phase2** (3-4 hours)
   - Replace ~30-40 hard-coded texts (most complex step)
   - Handle loop logging (convert to summaries)
   - Update substep logging
   - Test with parallel execution
   - Commit changes
   - File: `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap`

5. **Migrate zcl_fi_alloc_step_phase3** (2-3 hours)
   - Replace ~15-20 hard-coded texts
   - Update document creation logging
   - Test integration with FI module
   - Commit changes
   - File: `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap`

6. **Migrate zcl_fi_alloc_step_corr_bche** (1-2 hours)
   - Replace ~10-15 hard-coded texts
   - Update correction logging
   - Test batch processing
   - Commit changes
   - File: `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap`

**Acceptance Criteria:**
- ✅ All ~100 hard-coded string templates removed
- ✅ All 5 step files use mo_log->message() exclusively
- ✅ No MESSAGE ... INTO lv_dummy patterns remain
- ✅ All files compile without errors
- ✅ Git commits created per step file
- ✅ No exceptions explicitly logged in step code (framework handles)

**Technical Notes:**
- Work one file at a time, test after each
- Commit frequently (per file) for easy rollback
- Use migration guide patterns for transformation
- Test with representative data (company 1000, period 01)

---

### Story 4.6: Translation to English (Optional)

**Target Repository:** `cz.imcg.fast.planner`, `cz.imcg.fast.ovysledovka`  
**Effort:** 3-4 hours  
**Priority:** Low (can defer to future sprint)  
**Depends On:** Stories 4.2, 4.5

**Description:**
Translate all T100 messages from Czech to English for international deployment.

**Tasks:**

1. **Translate ZFI_PROCESS Messages** (1 hour)
   - Execute SE63
   - Translate ~30 framework messages (CS → EN)
   - Review translations with native speaker (if available)
   - Save translations

2. **Translate ZFI_ALLOC Messages** (2-3 hours)
   - Execute SE63
   - Translate ~100 business messages (CS → EN)
   - Review business terminology consistency
   - Save translations

3. **Test English User Experience** (30 minutes)
   - Log in as user with LANGU = EN
   - Execute allocation
   - Verify SLG1 shows English texts
   - Verify no missing translation warnings (ZFI_PROCESS/997)

**Acceptance Criteria:**
- ✅ All ZFI_PROCESS messages translated to English
- ✅ All ZFI_ALLOC messages translated to English
- ✅ English user sees English texts in SLG1
- ✅ No translation warnings logged

**Technical Notes:**
- Use SE63: Short Texts → ABAP Objects → Messages
- Translate in batches (10-20 messages) for consistency
- Keep business terminology consistent (e.g., "allocation" not "distribution")
- Can be done by separate translator (provide spreadsheet export)

---

### Story 4.7: Integration Testing & Regression

**Target Repository:** Both  
**Effort:** 6-8 hours  
**Priority:** High (quality gate)  
**Depends On:** Stories 4.1-4.5

**Description:**
Comprehensive testing to ensure migrated code produces same business results and new logging works correctly.

**Tasks:**

1. **Unit Test Coverage** (2-3 hours)
   - Create mock logger for all 5 step classes
   - Verify message numbers logged
   - Verify message variables populated correctly
   - Achieve >80% code coverage
   - Files: `src/*/zcl_fi_alloc_step_*.clas.testclasses.abap`

2. **Integration Test Suite** (2-3 hours)
   - Test: Happy path (company 1000, period 01)
   - Test: Multi-company (3 companies, 12 periods)
   - Test: Error scenarios (validation failures, missing data)
   - Test: bgRFC parallel execution (10 concurrent substeps)
   - Verify SLG1 logs created with correct external number
   - Compare results with pre-migration baseline

3. **Regression Test Matrix** (2-3 hours)
   - Run all 18 completed stories from Sprints 1-3
   - Verify business results unchanged (item counts, document numbers, amounts)
   - Verify exception handling still works
   - Verify process instance status transitions correct
   - Document any deviations (expected: none)

4. **Performance Baseline** (1 hour)
   - Measure execution time: Pre-migration vs. Post-migration
   - Verify no significant performance degradation (accept <5% overhead for BAL writes)
   - Measure BAL log size (estimate archiving needs)

**Acceptance Criteria:**
- ✅ All unit tests pass
- ✅ All integration tests pass
- ✅ All 18 regression stories pass
- ✅ Business results unchanged (item counts, amounts, documents)
- ✅ Performance within acceptable range (<5% overhead)
- ✅ SLG1 logs accessible and readable
- ✅ No showstopper bugs found

**Technical Notes:**
- Use transaction SLG1 to verify logs manually
- Query BALHDR/BALDAT tables for automated verification
- Keep baseline results from pre-migration for comparison
- Document any expected differences (e.g., message format changes)

---

### Story 4.8: Documentation & Training

**Target Repository:** `cz.imcg.fast.allocations` (planning repo)  
**Effort:** 4-6 hours  
**Priority:** Medium (enables team adoption)  
**Depends On:** Stories 4.1-4.7

**Description:**
Finalize documentation and train team on new logging architecture.

**Tasks:**

1. **Review & Update Documentation** (2-3 hours)
   - Review ADR-008 for accuracy
   - Review Implementation Guide for completeness
   - Review Migration Guide for clarity
   - Add code examples from actual implementation
   - Update with any design changes made during implementation

2. **Create Training Materials** (1-2 hours)
   - Presentation: Architecture overview (15 slides)
   - Demo: Live coding session (show step migration)
   - Cheat sheet: Quick reference card (1-page PDF)
   - FAQ: Common questions and answers

3. **Conduct Team Training** (1-2 hours)
   - Present architecture to team
   - Live demo of new logging API
   - Walk through SLG1 log analysis
   - Q&A session
   - Record session for future reference

**Acceptance Criteria:**
- ✅ Documentation reviewed and updated
- ✅ Training materials created (presentation, demo, cheat sheet)
- ✅ Team training conducted
- ✅ Team comfortable with new logging API
- ✅ Training recording available

**Technical Notes:**
- Schedule training after implementation complete (not during)
- Provide hands-on exercises for team members
- Share documentation links in team wiki/SharePoint

---

## Summary

### Effort Breakdown

| Story | Repository | Effort (hours) | Priority | Depends On |
|-------|------------|----------------|----------|------------|
| 4.1 - Logger Infrastructure | planner | 8-12 | High | None |
| 4.2 - Message Classes | both | 4-6 | High | None |
| 4.3 - Instance Integration | planner | 6-8 | High | 4.1 |
| 4.4 - bgRFC Integration | planner | 4-6 | Medium | 4.3 |
| 4.5 - Step Migration | ovysledovka | 12-16 | High | 4.1, 4.2, 4.3 |
| 4.6 - Translation | both | 3-4 | Low | 4.2, 4.5 |
| 4.7 - Testing | both | 6-8 | High | 4.1-4.5 |
| 4.8 - Documentation | allocations | 4-6 | Medium | 4.1-4.7 |
| **TOTAL** | | **47-66** | | |

**Estimated Duration:** 6-8.5 working days (1-1.5 weeks for single developer, 3-5 days for 2 developers)

---

### Dependency Graph

```
4.1 (Logger Infrastructure) ─┬─> 4.3 (Instance Integration) ─┬─> 4.4 (bgRFC Integration) ─┐
                              │                                │                            │
4.2 (Message Classes) ────────┼────────────────────────────────┴─> 4.5 (Step Migration) ─┬─┤
                              │                                                            │ │
                              └───────────────────────────────────> 4.6 (Translation) ─────┤ │
                                                                                            │ │
                                                                                            ▼ ▼
                                                                    4.7 (Testing) ──> 4.8 (Documentation)
```

**Critical Path:** 4.1 → 4.3 → 4.5 → 4.7 → 4.8 (33-46 hours)

---

### Parallelization Opportunities

**Developer 1:**
- Sprint Week 1: Stories 4.1, 4.3, 4.4 (framework infrastructure)

**Developer 2:**
- Sprint Week 1: Stories 4.2, 4.5 (message classes + step migration)

**Both:**
- Sprint Week 2: Stories 4.7, 4.8 (testing + documentation)

**Benefits:** Reduces total calendar time from 8.5 days → 5 days

---

### Risk Mitigation

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Migration introduces bugs | High | Medium | Comprehensive regression testing (Story 4.7) |
| Performance degradation | Medium | Low | Performance baseline, <5% overhead acceptable |
| Incomplete SE63 translation | Low | Medium | Defer translation to future sprint (Story 4.6) |
| BAL log volume issues | Medium | Low | Monitor log size, design supports segmentation |
| Team adoption resistance | Low | Low | Training + documentation (Story 4.8) |

---

### Definition of Done (Sprint 4)

**Minimum (MVP):**
- ✅ Stories 4.1, 4.2, 4.3, 4.5, 4.7 complete
- ✅ All 5 step files migrated (no hard-coded texts)
- ✅ All integration tests pass
- ✅ Czech language supported
- ✅ Documentation published

**Full (Recommended):**
- ✅ All stories 4.1-4.8 complete
- ✅ English translation complete
- ✅ Team trained
- ✅ Production-ready

**Future Enhancements (Backlog):**
- ⏰ Custom table persistence (if BAL query performance becomes issue)
- ⏰ Log segmentation (if single log becomes too large)
- ⏰ Additional languages (DE, SK, etc.)
- ⏰ Fiori UI for log analysis (instead of SLG1)

---

## Implementation Sequence

### Week 1 (Days 1-3): Infrastructure

**Day 1:**
- Morning: Story 4.1 (Logger Infrastructure) - Interface + Class
- Afternoon: Story 4.1 (Logger Infrastructure) - Unit Tests

**Day 2:**
- Morning: Story 4.2 (Message Classes) - ZFI_PROCESS + Inventory
- Afternoon: Story 4.3 (Instance Integration) - Process Instance Updates

**Day 3:**
- Morning: Story 4.3 (Instance Integration) - Step Execution Orchestration
- Afternoon: Story 4.4 (bgRFC Integration)

---

### Week 2 (Days 4-6): Migration & Testing

**Day 4:**
- Morning: Story 4.5 (Step Migration) - Create T100 messages
- Afternoon: Story 4.5 (Step Migration) - Migrate INIT + PHASE1

**Day 5:**
- Morning: Story 4.5 (Step Migration) - Migrate PHASE2 (most complex)
- Afternoon: Story 4.5 (Step Migration) - Migrate PHASE3 + CORR_BCHE

**Day 6:**
- Morning: Story 4.7 (Integration Testing) - Unit + Integration tests
- Afternoon: Story 4.7 (Integration Testing) - Regression tests

---

### Week 3 (Days 7-8): Polish & Delivery

**Day 7:**
- Morning: Story 4.6 (Translation) - SE63 English translation
- Afternoon: Story 4.8 (Documentation) - Review docs, create training

**Day 8:**
- Morning: Story 4.8 (Documentation) - Team training
- Afternoon: Final testing, transport creation, production deployment

---

## Deliverables

### Code Artifacts (Framework Repo: `cz.imcg.fast.planner`)
- `src/zif_fi_process_logger.intf.abap` - Logger interface
- `src/zcl_fi_process_logger.clas.abap` - Logger implementation
- `src/zcl_fi_process_logger.clas.testclasses.abap` - Unit tests
- `src/zfi_process.msag.xml` - Framework message class (expanded)
- `src/zcl_fi_process_step.clas.abap` - Parent class (mo_log attribute)
- `src/zcl_fi_process_instance.clas.abap` - Instance class (logger integration)

### Code Artifacts (Business Repo: `cz.imcg.fast.ovysledovka`)
- `src/zfi_alloc.msag.xml` - Business message class (expanded to ~120 messages)
- `src/zfi_alloc_process/zcl_fi_alloc_step_init.clas.abap` - Migrated
- `src/zfi_alloc_process/zcl_fi_alloc_step_phase1.clas.abap` - Migrated
- `src/zfi_alloc_process/zcl_fi_alloc_step_phase2.clas.abap` - Migrated
- `src/zfi_alloc_process/zcl_fi_alloc_step_phase3.clas.abap` - Migrated
- `src/zfi_alloc_process/zcl_fi_alloc_step_corr_bche.clas.abap` - Migrated

### Documentation (Planning Repo: `cz.imcg.fast.allocations`)
- `docs/architecture/ADR-008-process-logger-architecture.md` - Architecture decision record
- `docs/developer-guides/process-logger-implementation-guide.md` - Developer guide
- `docs/developer-guides/process-logger-migration-guide.md` - Migration guide
- `_bmad-output/brainstorming/brainstorming-session-2026-03-11-*.md` - Design session notes

### Testing Artifacts
- Unit tests for all new classes
- Integration test suite (happy path, error scenarios, bgRFC)
- Regression test results (18 completed stories)
- Performance baseline comparison

### Training Materials
- Architecture presentation (15 slides)
- Live demo script
- Quick reference cheat sheet (1-page PDF)
- Recorded training session

---

## Success Metrics

### Technical Metrics
- **Code Quality:** >80% unit test coverage
- **Performance:** <5% execution time overhead
- **Completeness:** 0 hard-coded string templates remaining
- **Reliability:** 0 regression test failures

### Business Metrics
- **Multi-language Support:** Czech + English (minimum)
- **Observability:** 100% of process executions logged to BAL
- **Auditability:** Process instance UUID links all log entries
- **Maintainability:** Text changes via SE91 (no code transport needed)

### Adoption Metrics
- **Team Training:** 100% of developers trained
- **Documentation:** 100% of documentation published
- **Production Readiness:** Transport created, deployment plan ready

---

## Next Steps After Completion

1. **Deploy to Quality Assurance** (Day 9)
   - Create transport
   - Import to QA system
   - Run full regression suite
   - Get business user sign-off

2. **Deploy to Production** (Day 10)
   - Schedule deployment window
   - Import transport
   - Monitor first production execution
   - Verify SLG1 logs accessible

3. **Monitor & Iterate** (Ongoing)
   - Monitor BAL log volume (archiving needs)
   - Collect feedback from developers
   - Plan future enhancements (custom tables, Fiori UI)

---

**Plan Owner:** Zdenek Smolik  
**Approval Date:** TBD  
**Implementation Start:** TBD  
**Target Completion:** Sprint 4 End
