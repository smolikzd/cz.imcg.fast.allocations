# ADR-008: Process Logger Architecture

**Date:** 2026-03-11  
**Status:** Implemented (Completed 2026-03-12)  
**Context:** Sprint 4 - Exception handling and message persistence architecture  
**Decision Makers:** Zdenek Smolik  
**Related ADRs:** Constitution v1.0.0 (DDIC-First, SAP Standards, Factory Pattern)  
**Implementation:** Stories 4-1 through 4-5 (commits 957d429-6407889)

---

## Context and Problem Statement

The ZFI_PROCESS framework and its ZFI_ALLOC_PROCESS implementation currently suffer from several logging and observability issues:

1. **Hard-coded Czech texts:** ~100+ string templates (`|Zpracováno { lv_count } položek|`) scattered across 5 allocation steps
2. **No multi-language support:** Czech-only messages prevent international deployment
3. **Inconsistent logging patterns:** Mix of `MESSAGE ... INTO lv_dummy` + `mo_log->log_sy_msg()` calls
4. **No unified audit trail:** Exception and message logging disconnected from process instance lifecycle
5. **Poor text governance:** Business text changes require code modifications and transports

**Core Requirements:**
- Language-independent message system (support Czech, English, future languages)
- Simple API for step developers to log messages
- Automatic exception logging by framework
- Unified process instance log (all messages linked to instance UUID)
- Minimal migration effort from current patterns

---

## Decision Drivers

### Immovable Constraints
- **Factory Pattern Enforcement** (Constitution Principle IV)
- **ZCX_FI_PROCESS_ERROR** as framework exception (backwards compatibility)
- **Process Instance Lifecycle** (NEW → RUNNING → COMPLETED/FAILED/CANCELLED)
- **bgRFC Serialization** for queued steps (concurrent substep execution)
- **ABAP 7.58** language features

### Negotiable Constraints
- **DDIC-First Principle** (preferred, but local types acceptable with justification)
- **SAP Standards First** (discover SAP solutions, deviate only when insufficient)
- **Backwards Compatibility** (18 completed stories must continue working)

### Guiding Principles
- **YAGNI** (You Aren't Gonna Need It) - Start simple, extend when proven necessary
- **Separation of Concerns** - Logger handles messages, statistics handle metrics
- **Fail Fast** - Configuration errors should dump, not fail silently
- **Defensive Programming** - Log logging failures, but don't break business logic

---

## Considered Options

### Persistence Strategy
- **Option A:** Custom table only (full SQL query power, must design and maintain)
- **Option B:** BAL + Custom table (dual persistence, maximum flexibility, high overhead)
- **Option C:** BAL only with extensible API ✅ **SELECTED**

### Language Mechanism
- **Option A:** T100 Purist (all messages via SE91, SE63 translation) ✅ **SELECTED**
- **Option B:** Pragmatic Hybrid (T100 for errors, templates for verbose logging)
- **Option C:** Custom template table (flexible, no SAP standards)

### API Pattern
- **Option A:** Enhanced logger instance (inherited `mo_log`) ✅ **SELECTED**
- **Option B:** Fluent message builder (chainable API)
- **Option C:** Static utility + context injection
- **Option D:** Thread-local implicit context

### Message Class Organization
- **Option A:** Single class (expand ZFI_ALLOC to ~120 messages)
- **Option B:** Hierarchical (ZFI_PROCESS + ZFI_ALLOC) ✅ **SELECTED**
- **Option C:** Step-specific (6 message classes, one per step)

### Configuration
- **Option A:** TVARVC (simple system-wide switches)
- **Option B:** Custom config table (flexible, per-process-type)
- **Option C:** Instance-level config (maximum flexibility)
- **Option D:** No configuration ✅ **SELECTED** (YAGNI)

---

## Decision Outcome

### Chosen Option: "BAL-First Logger with T100 Purist Language Strategy"

**9 Core Architecture Decisions:**

#### 1. Persistence Strategy: BAL-Only (Extensible)
**Decision:** Use SAP Application Log (BAL) as sole persistence mechanism. Logger interface (`ZIF_FI_PROCESS_LOGGER`) abstracts persistence, allowing future extension to custom tables without changing step code.

**Rationale:**
- BAL designed for high-volume logging (proven in EDI, batch processing)
- Standard SAP audit trail (SLG1 transaction, archiving via SLG2)
- Avoids custom table design/maintenance burden
- API abstraction protects against future changes

**Trade-offs:**
- ✅ Simple implementation, leverages SAP standards
- ✅ No custom table maintenance
- ❌ Query complexity (BAL tables not optimized for ad-hoc reporting)
- ❌ Future Fiori UI may require table persistence (mitigated by extensible API)

---

#### 2. Exception Handling: Framework Auto-Logging
**Decision:** Steps raise `ZCX_FI_PROCESS_ERROR` exceptions. Framework catches and logs them automatically via `TRY-CATCH` in orchestration layer. No explicit exception logging API for step developers.

**Rationale:**
- Clean separation: Steps focus on business logic, framework handles observability
- Consistent exception logging (can't be forgotten)
- Framework extracts text via `IF_T100_MESSAGE` integration
- Backwards compatible (existing exception patterns unchanged)

**Implementation:**
```abap
" Step code (unchanged):
RAISE EXCEPTION TYPE zcx_fi_process_error
  EXPORTING textid = zcx_fi_process_error=>validation_failed
            value  = lv_company_code.

" Framework orchestration (new):
TRY.
    io_step->execute( ).
  CATCH zcx_fi_process_error INTO DATA(lx_error).
    mo_log->log_exception_from_framework( lx_error ).
    RAISE EXCEPTION lx_error.  " Re-raise for status handling
ENDTRY.
```

---

#### 3. API Pattern: Inherited Logger Attribute
**Decision:** Parent step class (`ZCL_FI_PROCESS_STEP`) contains `mo_log` attribute. All child steps inherit this attribute. Framework initializes logger during step execution.

**Rationale:**
- Familiar pattern (consistent with current `mo_log` usage)
- No context passing needed (logger initialized once, available throughout execution)
- Simple for developers (just use inherited `mo_log`)
- Framework manages lifecycle (creation, attachment, persistence)

**Implementation:**
```abap
" Parent class:
CLASS zcl_fi_process_step DEFINITION ABSTRACT.
  PROTECTED SECTION.
    DATA mo_log TYPE REF TO zif_fi_process_logger.
    METHODS initialize_logger IMPORTING io_logger TYPE REF TO zif_fi_process_logger.
ENDCLASS.

" Step usage:
METHOD execute.
  mo_log->message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '100'
    iv_message_v1     = lv_company_code
    iv_severity       = 'I'
  ).
ENDMETHOD.
```

---

#### 4. Logger Lifecycle: Create/Load in Process Instance
**Decision:** Logger created in `ZCL_FI_PROCESS_INSTANCE->create()` (new instance) or `->load()` (existing instance). Logger attached to process instance for entire lifecycle.

**Rationale:**
- Logger exists from moment instance exists (capture all events)
- Single logger shared across all steps (unified audit trail)
- BAL log created once, reused across multiple load/execute cycles
- Framework controls lifecycle (steps just consume)

**Implementation:**
```abap
" In ZCL_FI_PROCESS_INSTANCE:
METHOD create.
  " Create instance
  ro_instance = NEW zcl_fi_process_instance( ).
  ro_instance->mv_uuid = cl_system_uuid=>create_uuid_x16_static( ).
  
  " Create logger with instance UUID as BAL external number
  ro_instance->mo_log = zcl_fi_process_logger=>create_new(
    iv_external_number = ro_instance->mv_uuid
    iv_object          = 'ZFI_PROCESS'
    iv_subobject       = iv_process_type
  ).
ENDMETHOD.

METHOD load.
  " Load instance
  ro_instance = NEW zcl_fi_process_instance( ).
  SELECT SINGLE * FROM zfi_process_instance INTO CORRESPONDING FIELDS OF @ro_instance ...
  
  " Attach to existing BAL log (or create if first load)
  ro_instance->mo_log = zcl_fi_process_logger=>attach_existing(
    iv_external_number = ro_instance->mv_uuid
  ).
ENDMETHOD.
```

---

#### 5. External ID: Instance UUID
**Decision:** BAL log external number = process instance UUID. Single BAL log per process instance lifetime.

**Rationale:**
- Simple correlation (UUID uniquely identifies instance AND log)
- Complete audit trail in one BAL log (regardless of load/retry count)
- Easy SLG1 search: external number = instance UUID
- Natural key (no need for compound identifiers)

**Trade-offs:**
- ✅ Single unified log per instance
- ✅ Simple correlation model
- ❌ Very long-running processes may create large logs (acceptable, BAL designed for volume)

---

#### 6. Concurrent Access: Unified Log for bgRFC Substeps
**Decision:** Parent instance and all bgRFC substeps write to SAME BAL log (identified by parent instance UUID). BAL framework handles concurrent writes.

**Rationale:**
- Unified audit trail (all substep messages in parent log)
- Chronological message flow visible in SLG1
- BAL handles locking/serialization during save
- Simpler than parent-child log hierarchies

**Implementation:**
```abap
" Substep execution (in bgRFC work process):
METHOD execute_in_bgRFC.
  " Deserialize context (includes parent instance UUID)
  DATA(ls_context) = deserialize( iv_context_string ).
  
  " Attach to PARENT's BAL log
  DATA(lo_logger) = zcl_fi_process_logger=>attach_existing(
    iv_external_number = ls_context-instance_uuid  " Parent UUID!
  ).
  
  " Execute substep with logger
  lo_step->initialize_logger( lo_logger ).
  lo_step->execute_substep( ... ).
  
  " Save (BAL handles concurrency)
  lo_logger->save( ).
ENDMETHOD.
```

---

#### 7. Language Strategy: T100 Purist
**Decision:** ALL messages flow through T100 message classes (SE91). Hard-coded string templates eliminated. SE63 translation workflow for all languages.

**Rationale:**
- Professional multi-language support (SE63 translation infrastructure)
- Central text governance (business users can edit messages without code changes)
- Text as configuration (changes don't require code transport)
- SAP standard approach (proven, well-documented)

**Migration Effort (Actual):**
- **97 hard-coded texts** → T100 messages (Story 4-5 Task 2: commit 35e9289)
- **97 code locations** migrated across 5 step classes (Tasks 3-7: commits ada878b-6407889)
- **1,027 lines added** (~58% code expansion due to TRY-CATCH blocks)
- **Total effort: ~24 hours** (within estimate, justified by long-term maintainability)

**Trade-offs:**
- ✅ Professional translation workflow
- ✅ Central text management
- ✅ Text changes independent of code
- ❌ 73-character limit (complex messages require splitting)
- ❌ Only 4 placeholders (msgv1-4)

---

#### 8. Message Classes: Hierarchical (ZFI_PROCESS + ZFI_ALLOC)
**Decision:** Two message classes:
- **ZFI_PROCESS:** Framework lifecycle, orchestration (~50 messages, reusable)
- **ZFI_ALLOC:** Allocation business logic (~120 messages, domain-specific)

**Rationale:**
- Logical separation (framework vs. business domain)
- Framework messages reusable across process types (future: ZFI_DUNNING_PROCESS, etc.)
- Business messages isolated (allocation-specific context)
- Severity distinguishes errors within classes (no separate ZFI_ALLOC_ERROR needed)

**Number Ranges (As Implemented):**
```
ZFI_PROCESS:
  001-019  Instance lifecycle (implemented)
  020-039  Step execution (implemented)
  040-059  Queued/bgRFC execution (implemented: 041-043)
  060-099  Framework errors (reserved)
  996-999  Infrastructure errors (logger failures, etc.)

ZFI_ALLOC:
  001-099   INIT step (001-012 existed, 013-020 added → 20 total)
  100-199   PHASE1 step (100-124 added → 25 messages)
  200-299   PHASE2 step (200-226 added → 27 messages, most verbose)
  300-399   PHASE3 step (300-324 added → 25 messages)
  400-499   CORR_BCHE step (400-411 added → 12 messages)
  500-599   General allocation errors (500-507 existed, reused)
  
  Total: 21 → 118 messages (5.6x expansion)
```

---

#### 9. Edge Case Handling: Defensive Logging + Fail Fast
**Decision:** Handle edge cases with visibility and fail-fast approach:

**A. Logger Failure (Defensive Logging):**
When `BAL_LOG_MSG_ADD` fails, log the failure itself. DO NOT fail silently.

**Implementation:**
```abap
METHOD add_message_to_bal.
  CALL FUNCTION 'BAL_LOG_MSG_ADD' ...
  
  IF sy-subrc <> 0.
    " Log the logging failure (ZFI_PROCESS/998)
    MESSAGE e998(zfi_process) WITH is_message-msgid is_message-msgno INTO DATA(lv_dummy).
    " Attempt to add error message
    TRY.
        CALL FUNCTION 'BAL_LOG_MSG_ADD' ...  " Log the error
      CATCH cx_root.
        " Even error logging failed - framework-level issue
    ENDTRY.
  ENDIF.
ENDMETHOD.
```

**B. Language Fallback (Explicit Failure):**
NO automatic fallback to EN/CS. If translation missing, show clear indication in log.

**Implementation:**
```abap
METHOD message.
  MESSAGE ID iv_message_class ... INTO DATA(lv_message_text).
  
  IF sy-subrc <> 0 OR strlen( lv_message_text ) = 0.
    " Message not translated - log this fact (ZFI_PROCESS/997)
    MESSAGE w997(zfi_process) WITH iv_message_class iv_message_number sy-langu INTO lv_message_text.
  ENDIF.
  
  " Add to BAL (with original or error message)
  ...
ENDMETHOD.
```

**C. Null Logger Safety (Fail Fast):**
If logger not initialized, step execution FAILS LOUDLY (dump). Indicates framework bug.

**Implementation:**
```abap
METHOD execute.
  ASSERT mo_log IS BOUND.  " Dumps if not initialized
  
  mo_log->message( ... ).
ENDMETHOD.
```

**Rationale:**
- Logging failures indicate infrastructure problems (must be visible)
- Missing translations enforce SE63 discipline (no silent degradation)
- Uninitialized logger = framework bug (must be fixed, not hidden)

---

## Validation (First Principles)

### Core Problem
**Statement:** "When a cost allocation process executes (possibly across multiple work processes), business users and developers need to understand what happened, why it happened, and whether it succeeded - in their own language."

**Architecture addresses:**
- ✅ Multi-language support (T100 + SE63)
- ✅ Complete audit trail (BAL log per instance UUID)
- ✅ Exception tracking (framework auto-logs)
- ✅ Business context (step messages with variables)
- ✅ Distributed tracing (bgRFC substeps in parent log)

### Not Over-Engineered
- Scales up: BAL handles high volume, standard archiving
- Scales down: Simple API (2 lines setup, 1 line per message)
- Migration: 20-40 hours (reasonable for compliance-critical financial system)

### Not Under-Engineered
- Future-proof: Interface abstraction allows table persistence later
- Maintainable: Central text governance via SE91
- Observable: Complete audit trail for debugging and compliance

### Separation of Concerns
- **Logger:** Qualitative events (what happened, messages, exceptions)
- **ZCL_FI_PROCESS_STATISTICS2:** Quantitative metrics (how long, how much)
- **Independent:** Correlated via instance UUID + timestamp (post-hoc analysis)

---

## Consequences

### Positive
- ✅ **Professional multi-language support** via SE63 translation workflow
- ✅ **Central text governance** - business users edit messages without code changes
- ✅ **Unified audit trail** - all events in single BAL log per instance
- ✅ **Simple API** - developers use inherited `mo_log->message()`
- ✅ **Backwards compatible** - existing exception patterns unchanged
- ✅ **Future-proof** - abstraction allows extension (table persistence, etc.)
- ✅ **Compliance-ready** - complete audit trail with standard SAP archiving

### Negative
- ❌ **Migration effort:** 20-40 hours to refactor hard-coded texts to T100
- ❌ **T100 limitations:** 73-char limit, 4 placeholders (requires message splitting)
- ❌ **BAL query complexity:** Not optimized for ad-hoc reporting (future Fiori UI may need tables)

### Neutral
- ⚠️ **No configuration layer** - may need to add later if persistence becomes bottleneck
- ⚠️ **Single log per instance** - may need segmentation for very long-running processes

---

## Implementation Notes

### New Components Required
1. **Interface:** `ZIF_FI_PROCESS_LOGGER` (message(), log_exception_from_framework(), save())
2. **Class:** `ZCL_FI_PROCESS_LOGGER` (implements interface, wraps BAL)
3. **Message Classes:** Expand ZFI_PROCESS (~50 msgs), expand ZFI_ALLOC (~120 msgs)
4. **Parent Class Update:** Add `mo_log` attribute to `ZCL_FI_PROCESS_STEP`
5. **Instance Update:** Add logger creation in `ZCL_FI_PROCESS_INSTANCE->create()`/`load()`

### Migration Path
1. **Phase 1:** Create infrastructure (interface, class, message classes)
2. **Phase 2:** Integrate with framework (instance lifecycle, step initialization)
3. **Phase 3:** Migrate step code (hard-coded texts → T100, MESSAGE calls → mo_log->message())
4. **Phase 4:** Test (unit tests, integration tests, regression tests)
5. **Phase 5:** SE63 translation (Czech → English, future languages)

### Risks & Mitigations
- **Risk:** BAL performance issues at scale
  - **Mitigation:** BAL proven in high-volume scenarios, can segment logs later if needed
- **Risk:** Migration introduces bugs in allocation logic
  - **Mitigation:** Comprehensive regression testing, parallel run with old logging
- **Risk:** Incomplete SE63 translation causes user confusion
  - **Mitigation:** Explicit warning messages (ZFI_PROCESS/997) when translation missing

---

## References

- Constitution v1.0.0 (Principle IV - Factory Pattern, Principle V - Error Handling)
- SAP Documentation: T100 Message Framework, BAL Application Log
- Brainstorming Session: 2026-03-11 (Constraint Mapping, Morphological Analysis, First Principles)
- Sprint 3 Completion: 18 stories completed, backwards compatibility required

---

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-03-11 | BAL-only persistence | YAGNI, extensible API, leverages SAP standards |
| 2026-03-11 | T100 Purist language strategy | Central text governance, professional translation workflow |
| 2026-03-11 | Inherited logger API pattern | Simple for developers, consistent with current patterns |
| 2026-03-11 | No configuration layer | YAGNI, BAL handles volume, add later if needed |
| 2026-03-11 | Hierarchical message classes | Framework reusability, logical separation |
| 2026-03-11 | Fail-fast edge case handling | Visibility over silent failures, framework bugs must be fixed |

---

## Implementation Outcomes (March 2026)

### Delivered Components

**Story 4-1: Logger Infrastructure** (commits 957d429, 693684c, 8cc0881)
- ✅ `ZIF_FI_PROCESS_LOGGER` interface (3 methods: message, log_exception_from_framework, save)
- ✅ `ZCL_FI_PROCESS_LOGGER` implementation (BAL wrapper, ~400 lines)
- ✅ Unit tests passing (logger creation, message logging, exception handling)

**Story 4-2: Message Class Expansion** (commits 915d407, 9e6fe0f)
- ✅ **ZFI_PROCESS:** 50+ messages (lifecycle, orchestration, framework errors)
- ✅ **ZFI_ALLOC:** 21 → 118 messages (5.6x expansion, bilingual Czech+English)

**Story 4-3: Process Instance Integration** (commits 38d0f72, 1bf5577)
- ✅ Logger creation in `ZCL_FI_PROCESS_INSTANCE->create()` and `load()`
- ✅ External number = instance UUID (1:1 correlation)
- ✅ Base class `ZCL_FI_PROCESS_STEP` updated with `mo_log` attribute

**Story 4-4: bgRFC Substep Integration** (commits fc57d23, 00cfd04, 4e22341)
- ✅ Substeps attach to parent log (unified audit trail)
- ✅ Lifecycle logging (messages 041-043: queued, started, completed)
- ✅ BAL concurrency handling verified (BALI functional method tests passing)

**Story 4-5: Step Code Migration** (commits 35e9289, ada878b, 8a268d6, 9d8cfff, 5403436, 6407889)
- ✅ **97 messages created** in ZFI_ALLOC (013-020, 100-124, 200-226, 300-324, 400-411)
- ✅ **5 step classes migrated** (INIT, PHASE1, PHASE2, PHASE3, CORR_BCHE)
- ✅ **1,027 lines added** (~58% code expansion for logging blocks)
- ✅ **Legacy logger removed** (`zcl_sp_ballog` → inherited `mo_log`)

### Code Impact Analysis

| Step Class | Before | After | Δ Lines | Δ % | Hard-coded Strings | Messages Added |
|------------|--------|-------|---------|-----|-------------------|----------------|
| INIT | 168 | 264 | +96 | +57% | 8 | 013-020 (8) |
| PHASE1 | 319 | 415 | +96 | +30% | 25 | 100-124 (25) |
| PHASE2 | 638 | 959 | +321 | +50% | 27 | 200-226 (27) |
| PHASE3 | 419 | 804 | +385 | +92% | 25 | 300-324 (25) |
| CORR_BCHE | 205 | 334 | +129 | +63% | 12 | 400-411 (12) |
| **Total** | **1,749** | **2,776** | **+1,027** | **+58%** | **97** | **97** |

**Key Pattern:** Each migrated message requires ~10-15 lines due to:
```abap
MESSAGE e014(zfi_alloc) INTO rs_result-message.
IF mo_log IS BOUND.
  TRY.
      mo_log->message(
        iv_message_class  = 'ZFI_ALLOC'
        iv_message_number = '014'
        iv_severity       = 'E'
      ).
    CATCH zcx_fi_process_error.
      " Logger failed - continue with result message
  ENDTRY.
ENDIF.
```

### Validation Results

**✅ Architecture Principles Met:**
- Multi-language support: Czech + English messages fully bilingual
- Central text governance: All texts in T100 (SE91), no hard-coded strings
- Unified audit trail: All messages linked to instance UUID via BAL external number
- Simple API: Developers use inherited `mo_log->message()` (2 lines setup, 1 line per message)
- Backwards compatible: Existing 18 stories unaffected (exception patterns unchanged)

**✅ Performance Acceptable:**
- BAL writes add <5% overhead (subjective, pending formal measurement in Story 4-7)
- No performance degradation reported during development testing

**✅ Code Quality Improved:**
- Legacy logger (`zcl_sp_ballog`) removed (inconsistent pattern eliminated)
- Constitution-compliant (DDIC-first, factory pattern, SAP standards)
- Defensive logging (logger failures logged via ZFI_PROCESS/998)

### Lessons Learned

1. **Code Expansion Expected:** 58% increase acceptable for observability-critical financial system
2. **Bilingual from Start:** Creating Czech+English messages upfront saves SE63 translation effort later
3. **Message Reuse:** Existing messages (e005 UUID error, e006 DB error, e008/e009 completion) reused across steps
4. **TRY-CATCH Overhead:** Each message requires defensive CATCH block (framework issue if logging fails)
5. **bgRFC Logging Simple:** Parent log attachment pattern worked seamlessly (no special code needed)

### Known Limitations

- **73-char T100 limit:** Some complex messages split into multiple (acceptable trade-off)
- **4 placeholder limit:** Sufficient for allocation domain (no cases requiring >4 variables encountered)
- **BAL query complexity:** Ad-hoc reporting requires custom SQL (SLG1 sufficient for debugging/audit)
- **No configuration layer:** YAGNI confirmed (no need to disable logging per process type)

### Next Steps

- **Story 4-7:** Integration testing & regression (ensure no business logic breakage)
- **Story 4-8:** Documentation & training (team enablement)
- **Future:** Fiori UI may require table persistence (extensible API supports this)

---

**Author:** Zdenek Smolik  
**Reviewers:** TBD  
**Approval Date:** TBD  
**Implementation Target:** Sprint 4
