---
stepsCompleted: [1, 2, 3]
inputDocuments: []
session_topic: 'Exception handling and message persistence architecture for ZFI_PROCESS framework'
session_goals: 'Design language-dependent exception texts, simple message API with persistence, exception persistence, unified process instance log, configurable persistence switches'
selected_approach: 'ai-recommended'
techniques_used: ['Constraint Mapping', 'Morphological Analysis', 'First Principles Thinking']
ideas_generated: 9
context_file: 'Both cz.imcg.fast.planner and cz.imcg.fast.ovysledovka repositories analyzed'
brainstorming_complete: true
implementation_started: true
---

# Brainstorming Session Results

**Facilitator:** Zdenek
**Date:** 2026-03-11

## Session Overview

**Topic:** Exception handling and message persistence architecture for the planner framework, specifically examining the ovysledovka step implementation patterns

**Goals:** 
1. Design language-dependent exception text system
2. Create simple API for message issuance with database persistence
3. Implement exception persistence mechanism
4. Build process instance log containing all exceptions and messages
5. Add configurable on/off switches for persistence features

### Context Guidance

**Technical Context Loaded from Both Repositories:**

#### Current State Analysis (from cz.imcg.fast.ovysledovka):

**Exception Handling:**
- Framework exception `ZCX_FI_PROCESS_ERROR` used consistently across all 5 allocation steps
- 23 RAISE EXCEPTION calls across steps (INIT:2, PHASE1:5, PHASE2:9, PHASE3:5, CORR_BCHE:2)
- Text IDs: `general_error`, `invalid_parameters`, `invalid_operation`, `invalid_status`
- Local exception classes: `ZCX_FI_ALLOC_PLAN`, `ZCX_FI_OV_KEBOOLA_EXTRACTOR` (T100-based)
- Exception chaining supported via `previous` parameter

**Message Handling:**
- Message class `ZFI_ALLOC` (12 messages, primarily Czech with partial English)
- Pattern: `MESSAGE s000(zfi_alloc) WITH |text| INTO lv_dummy` + `mo_log->log_sy_msg()`
- Estimated 20-30 MESSAGE calls per step execution
- Heavy use of hard-coded Czech text in string templates (NOT I18N-ready)

**Logging Infrastructure:**
- SAP Application Log (BAL) framework via `ZCL_SP_BALLOG` wrapper
- Logs stored in standard SAP tables: `BALDAT`, `BALHDR` (accessible via SLG1)
- External number pattern: `{company_code}/{fiscal_year}/{fiscal_period}/{allocation_id}`
- Subobjects differentiate phases: PHASE1, PHASE2, PHASE3, CORR_BCHE
- Log object: `ZFI_ALLOC`

**Current Limitations:**
1. **No custom persistence**: Exceptions/messages only in BAL (requires SLG1 transaction)
2. **Hard-coded Czech texts**: ~100+ string template literals in code
3. **No language switching**: Czech-only for dynamic content
4. **No persistence toggle**: All logging always active
5. **No structured error tables**: Error details not queryable outside BAL

#### Framework Architecture (from cz.imcg.fast.planner):

**Exception Framework:**
- `ZCX_FI_PROCESS_ERROR` with 15 error constants (general_error, invalid_parameters, etc.)
- T100 message integration with 200-char truncation for persistence
- Support for `msgv1-4` and `value` parameters

**Process Instance:**
- Status states: NEW, RUNNING, COMPLETED, FAILED, CANCELLED
- Business status: INIT (empty), FINAL (success), FAIL (error)
- Context structure: `ty_context` for step communication
- Lifecycle hooks: `on_success()`, `on_error()` for state finalization

**Orchestration:**
- Serial steps: Execute sequentially in single LUW
- Queued steps: Execute substeps via bgRFC (parallel processing)
- Substep context: Serialization/deserialization for bgRFC execution
- Rollback support: Framework calls `rollback()` on step errors

**Key Patterns:**
- DDIC-first architecture (94 DDIC objects, zero local types)
- Factory pattern for instance creation
- Template method pattern for step execution
- Interface segregation: `zif_fi_process_step` (8 methods)

### Session Setup

**Technical Brainstorming Focus:**

We need to design an architecture that addresses these interconnected challenges:

1. **Language Independence**: Move away from hard-coded Czech texts toward a flexible I18N system
2. **Message Persistence**: Beyond SAP BAL - queryable custom tables for business reporting
3. **Exception Persistence**: Store exception details for analysis and debugging
4. **Unified Log**: Single API for both messages and exceptions with consistent persistence
5. **Performance Control**: Toggle persistence on/off for production vs. development scenarios
6. **Process Instance Integration**: Link all logs to process instance lifecycle

**Design Constraints:**
- Must follow DDIC-first principle (no local types)
- Must integrate with existing BAL infrastructure (don't replace, extend)
- Must support existing ZCX_FI_PROCESS_ERROR patterns
- Must not break 18 completed stories across 3 sprints
- Should support both serial and queued step execution modes

## Technique Selection

**Approach:** AI-Recommended Techniques
**Analysis Context:** Exception handling and message persistence architecture for ZFI_PROCESS framework with focus on language-dependent texts, persistence mechanisms, and unified logging

**Recommended Techniques:**

- **Constraint Mapping:** Visualize all architectural constraints (DDIC-first, BAL integration, backwards compatibility) to identify creative freedom zones vs immovable boundaries
- **Morphological Analysis:** Systematically explore parameter combinations (Persistence × Language × API × Integration × Configuration) for comprehensive architecture coverage
- **First Principles Thinking:** Validate design by stripping away SAP conventions and habitual thinking to ensure architecture serves fundamental needs

**AI Rationale:** This sequence was selected because the challenge involves complex systems design with explicit constraints requiring systematic exploration while avoiding over-engineering. The three-phase flow moves from constraint understanding → comprehensive option generation → fundamental validation.

---


## Phase 1: Constraint Mapping - SAP Standards Discovery

**Constraint Refinement:**
- DDIC-First: Preferred (not immovable) - deviations discussable
- SAP Standards: Discover first, don't reinvent - deviate only when insufficient

**Decision:** Conduct SAP Standards Discovery before mapping constraints to understand available building blocks

---

### SAP Standards Discovery Session

**Research Focus Areas:**
1. Multi-language text handling mechanisms
2. Message persistence patterns
3. Exception logging frameworks
4. Configuration/switch mechanisms
5. Process instance logging patterns


### SAP Standards Discovery Results

**1. Multi-Language Text Handling:**
- ✅ T100 Message Framework (SE91) - Currently used, supports multi-language via SE63
- ✅ Text Symbols - Not used, could supplement T100
- ✅ OTR (Online Text Repository) - CL_OTR_TEXT_ACCESS for longer texts
- ⚠️ Limitation: Hard-coded Czech strings bypass T100 system

**2. Message Persistence:**
- ✅ BAL Framework (BALDAT/BALHDR) - Currently used via ZCL_SP_BALLOG
- ✅ Custom Database Tables - Pattern: dual persistence (BAL + custom table)
- ⚠️ BAL Limitation: Requires SLG1 transaction, not easily queryable

**3. Exception Logging:**
- ✅ CX_* Exception Classes - Currently using ZCX_FI_PROCESS_ERROR
- ✅ IF_T100_MESSAGE integration - Exceptions linked to T100 texts
- ❌ No built-in persistence - Must catch and log explicitly

**4. Configuration Switches:**
- ✅ TVARVC (Transaction STVARV) - System-wide parameters
- ✅ Custom Config Table Pattern - More flexible, can cache at init
- ✅ Feature Toggles (SFW5) - More complex, requires registration

**5. Process Instance Logging:**
- ❌ No SAP standard for custom process frameworks
- ✅ Best Practice: Create ZFI_PROCESS_INSTANCE_LOG table
- Pattern: LOG_UUID, INSTANCE_UUID, LOG_TYPE, MESSAGE_TEXT, TIMESTAMP, etc.

**Key Finding:** SAP provides building blocks (T100, BAL, CX_*, config tables) but no integrated solution for custom process framework logging with language support and configurable persistence.

---


### Enhanced SAP Standards Discovery (via SAP Documentation MCP)

**1. Multi-Language Text Handling - Deep Dive:**

**T100 Message Framework (IF_T100_MESSAGE):**
- Interface: IF_T100_MESSAGE contains structured attribute T100KEY (type SCX_T100KEY)
- Structure: { msgid, msgno, attr1-4 } for placeholder mapping
- Extension: IF_T100_DYN_MSG adds MSGTY (message type) and predefined placeholder attributes
- Translation: SE63 transaction manages multi-language texts
- Usage in exceptions: Exception classes implement IF_T100_MESSAGE to link to messages
- **Pattern:**
  ```abap
  CLASS cx_my_error DEFINITION INHERITING FROM cx_static_check.
    INTERFACES if_t100_message.
    CONSTANTS: BEGIN OF my_textid,
                 msgid TYPE symsgid VALUE 'ZMY_MSG_CLASS',
                 msgno TYPE symsgno VALUE '001',
                 attr1 TYPE scx_attrname VALUE 'ATTRIBUTE1',
               END OF my_textid.
  ENDCLASS.
  ```

**OTR (Online Text Repository):**
- Access API: CL_OTR_TEXT_ACCESS
- Transaction: SOTR_EDIT for text maintenance
- Tables: SOTR_HEAD, SOTR_TEXT (for concepts and translations)
- **Use cases:** WebDynpro, BSP, long-form exception texts, HTML texts
- **Advantages over T100:** No 73-char limit, supports rich text, alias-based access
- **Transport:** R3TR SOTR (long texts), R3TR SOTS (short texts)
- **Pattern:**
  ```abap
  DATA(lv_text) = cl_otr_text_access=>get_text(
    concept  = 'OTR_CONCEPT_ALIAS'
    langu    = sy-langu
    fallback = 'EN'
  ).
  ```

**Text Symbols:**
- Defined in program's text elements (GO TO → Text Elements)
- Access: TEXT-001, TEXT-002, etc.
- Translation: SE63 transaction
- **Limitation:** Program-specific, not reusable across programs

---

**2. Application Log (BAL) Framework - Deep Dive:**

**Architecture:**
- Tables: BALHDR (log headers), BALDAT (log data), BALFLD (context fields), BALM (messages)
- Transactions: SLG0 (define objects), SLG1 (analyze logs), SLG2 (delete expired logs)
- Archiving: BC_SBAL archiving object moves logs to archive files
- **Note:** BALM is temporary, data moved to BALHDR/BALDAT on save

**Object/Subobject Model:**
- Object: Logical application area (e.g., 'ZFI_ALLOC')
- Subobject: Refinement within object (e.g., 'PHASE1', 'PHASE2')
- External number: User-defined identifier for filtering (70 char max)

**API Patterns:**
- Classic API: Function modules (BAL_LOG_CREATE, BAL_LOG_MSG_ADD, BAL_DB_SAVE)
- Modern API (ABAP Cloud): CL_BALI_OBJECT_HANDLER, CL_BALI_MESSAGE_HANDLER
- **Pattern:**
  ```abap
  " Create log
  CALL FUNCTION 'BAL_LOG_CREATE'
    EXPORTING
      i_s_log = ls_log_header  " contains object, subobject, extnumber
    IMPORTING
      e_log_handle = lv_log_handle.
  
  " Add message
  CALL FUNCTION 'BAL_LOG_MSG_ADD'
    EXPORTING
      i_log_handle = lv_log_handle
      i_s_msg      = ls_message.  " contains msgty, msgid, msgno, msgv1-4
  
  " Save to database
  CALL FUNCTION 'BAL_DB_SAVE'
    EXPORTING
      i_t_log_handle = lt_log_handles.
  ```

**Limitations:**
- Query complexity: Requires understanding BAL table structure
- Performance: High-volume logging has overhead
- External number: 70-char limit may not accommodate complex keys

---

**3. Exception Classes - Deep Dive:**

**Hierarchy:**
- CX_ROOT (root of all exceptions)
  - CX_STATIC_CHECK (must be declared in method signature)
  - CX_DYNAMIC_CHECK (checked at runtime)
  - CX_NO_CHECK (unrecoverable errors)

**PREVIOUS Attribute:**
- Type: CX_ROOT
- Purpose: Chain exceptions to preserve error context
- **Pattern:**
  ```abap
  TRY.
    " operation
  CATCH cx_some_error INTO DATA(lx_error).
    RAISE EXCEPTION TYPE zcx_my_error
      EXPORTING
        textid   = zcx_my_error=>my_textid
        previous = lx_error.  " Preserve original exception
  ENDTRY.
  ```

**Runtime Error Display:**
- When dump occurs, all exception texts in PREVIOUS chain are shown
- Helps trace root cause through multiple layers

**No Built-in Persistence:**
- Exceptions are transient objects
- Must be explicitly caught and logged to persist
- Common pattern: TRY-CATCH → log to BAL → re-raise or convert

---

**4. Configuration Mechanisms - Deep Dive:**

**TVARVC Table:**
- Purpose: System-wide configuration parameters
- Structure: NAME (variable name, char30), TYPE ('P'=parameter, 'S'=select-option), LOW/HIGH (values)
- Transaction: STVARV (maintain), STVARVC (client-dependent variant)
- **Migration note:** TVARV (client-independent) replaced by TVARVC (client-dependent) in newer releases
- **Pattern:**
  ```abap
  SELECT SINGLE low FROM tvarvc
    INTO @DATA(lv_config_value)
    WHERE name = 'ZFI_LOG_PERSISTENCE_MODE'
      AND type = 'P'.
  IF sy-subrc = 0.
    " Use lv_config_value
  ENDIF.
  ```

**Custom Configuration Table Pattern:**
- **Advantage:** More flexible structure, can add metadata
- **Pattern:**
  ```abap
  " Table: ZFI_PROCESS_CONFIG
  " Fields: MANDT, CONFIG_KEY (key), CONFIG_VALUE, ACTIVE (flag), CHANGED_BY, CHANGED_AT
  
  SELECT SINGLE config_value FROM zfi_process_config
    INTO @DATA(lv_value)
    WHERE config_key = 'LOG_LEVEL'
      AND active = 'X'.
  ```

**Caching Strategy:**
- Read once at instance initialization
- Store in instance/class variable
- Avoid repeated database reads per operation

---

**5. Process Instance Logging - Pattern Synthesis:**

**No SAP Standard for Custom Frameworks:**
- SAP Workflow (SWI5) exists but only for SAP Business Workflow
- Change Documents (CDHDR/CDPOS) for data auditing, not process execution

**Recommended Custom Pattern:**
```
Table: ZFI_PROCESS_INSTANCE_LOG
Structure:
  - LOG_UUID (key, RAW16) - Unique log entry ID
  - INSTANCE_UUID (RAW16) - Foreign key to ZFI_PROCESS_INSTANCE
  - LOG_TIMESTAMP (TIMESTAMPL) - Precision timestamp
  - STEP_NAME (CHAR30) - Which step generated this log
  - LOG_TYPE (CHAR1) - 'E'=Exception, 'M'=Message, 'I'=Info, 'W'=Warning
  - SEVERITY (CHAR1) - 'E'=Error, 'W'=Warning, 'S'=Success, 'I'=Info
  - MESSAGE_CLASS (CHAR20) - T100 message class (if applicable)
  - MESSAGE_NUMBER (NUMC3) - T100 message number
  - MESSAGE_V1-V4 (CHAR50) - Message variables
  - MESSAGE_TEXT (CHAR255) - Rendered message text
  - EXCEPTION_CLASS (CHAR30) - Exception class name (if exception)
  - EXCEPTION_TEXT (STRING) - Full exception text with stack
  - LANGUAGE (LANGU) - Language of rendered text
  - USER_NAME (SYUNAME) - Who triggered the log
  - BAL_LOG_HANDLE (BALLOGHNDL) - Optional link to BAL log entry
  - CONTEXT_DATA (STRING) - JSON/XML with additional context
```

**Access Patterns:**
- By instance: SELECT WHERE instance_uuid = ... ORDER BY log_timestamp
- By severity: SELECT WHERE instance_uuid = ... AND severity IN ('E', 'W')
- By step: SELECT WHERE instance_uuid = ... AND step_name = 'PHASE2'
- Statistics: COUNT(*) GROUP BY severity, step_name

---

## SAP Standards Summary Matrix

| Capability | SAP Standard | API/Transaction | Strengths | Limitations | Recommendation |
|------------|--------------|-----------------|-----------|-------------|----------------|
| **Short text (≤73 chars)** | T100 Messages | SE91, IF_T100_MESSAGE | Multi-language, integrated with exceptions | Character limit, placeholder limit (4) | ✅ Use for exception texts, short messages |
| **Long text (>73 chars)** | OTR | SOTR_EDIT, CL_OTR_TEXT_ACCESS | No length limit, rich text | More complex setup | ✅ Use for descriptions, help texts |
| **Message persistence (audit)** | BAL Framework | SLG1, BAL_* FMs | Standard UI (SLG1), auditable | Query complexity, performance overhead | ✅ Use for audit trail |
| **Message persistence (reporting)** | Custom tables | Direct SQL | Full SQL query power, indexed | Must design & maintain | ✅ Use for business reporting |
| **Exception handling** | CX_STATIC_CHECK | RAISE EXCEPTION, TRY-CATCH | Structured, chainable (PREVIOUS) | No built-in persistence | ✅ Use with explicit logging |
| **System-wide config** | TVARVC | STVARV | Simple, standard | Limited structure | ✅ Use for simple flags/values |
| **Complex config** | Custom table | Direct SQL | Flexible structure, can version | Must design & maintain | ✅ Use for complex settings |
| **Process logging** | (None) | N/A | - | No standard exists | ⚠️ Design custom solution |

---


---

## Phase 2: Morphological Analysis (COMPLETED)

**Technique:** Systematic exploration of architecture parameter combinations

### Architecture Parameters Explored

1. **Persistence Strategy** → BAL-only (extensible via interface)
2. **API Pattern** → Inherited logger attribute (mo_log in parent class)
3. **Exception Handling** → Framework auto-logs (no step API)
4. **Logger Lifecycle** → Created in create()/load() methods
5. **External ID** → Instance UUID (single log per instance)
6. **Concurrent Access** → Unified log (BAL handles bgRFC concurrency)
7. **Language Strategy** → T100 Purist (all messages via SE91/SE63)
8. **Message Classes** → Hierarchical (ZFI_PROCESS + ZFI_ALLOC)
9. **Configuration** → None (YAGNI principle)

---

## Phase 3: First Principles Thinking (COMPLETED)

**Technique:** Validate design by questioning assumptions

### Core Problem Statement
"When a cost allocation process executes (possibly across multiple work processes), business users and developers need to understand what happened, why it happened, and whether it succeeded - in their own language."

### Validations Performed

✅ **Persistence Necessity:** Audit + Debug + Compliance requirements confirmed  
✅ **Multi-language Support:** T100 justified even for Czech-only (text governance)  
✅ **Logger Abstraction:** Serves simplicity, changeability, testability  
✅ **Single Log Per Instance:** Sufficient for current scale, extensible if needed  
✅ **Framework Auto-Logging:** Correct separation (steps=business, framework=observability)  
✅ **Separation from Statistics:** Logger=qualitative, Statistics=quantitative (orthogonal)  
✅ **Scope:** Not over/under-engineered (20-40h migration justified for compliance system)

### Edge Cases Resolved

1. **Statistics Integration:** Out of scope (ZCL_FI_PROCESS_STATISTICS2 handles metrics)
2. **Logger Failure:** Defensive logging (log the failure, don't fail silently)
3. **Language Fallback:** Explicit failure (show warning, no auto-fallback to EN/CS)
4. **Null Logger Safety:** Fail fast (ASSERT, framework bug must be fixed)

---

## Architecture Decisions (Final)

### 9 Core Decisions

1. **Persistence:** BAL-only, interface abstraction allows future table extension
2. **Exception Handling:** Framework auto-logs via TRY-CATCH, no step API needed
3. **API Pattern:** Inherited `mo_log` attribute in `ZCL_FI_PROCESS_STEP`
4. **Logger Lifecycle:** Created in `create()`/`load()`, shared across all steps
5. **External ID:** Instance UUID (single BAL log per process instance lifetime)
6. **Concurrent Access:** Unified log, BAL handles bgRFC substep concurrency
7. **Language Strategy:** T100 Purist (all messages via T100, SE63 translation)
8. **Message Classes:** ZFI_PROCESS (~50 msgs framework) + ZFI_ALLOC (~120 msgs business)
9. **Edge Cases:** Defensive logging, explicit failures, fail fast

---

## Deliverables

### Documentation (COMPLETED)

- **ADR-008:** Architecture Decision Record (docs/architecture/ADR-008-process-logger-architecture.md)
- **Implementation Guide:** Developer guide with API reference, patterns, troubleshooting (docs/developer-guides/process-logger-implementation-guide.md)
- **Migration Guide:** Step-by-step migration from hard-coded texts to T100 (docs/developer-guides/process-logger-migration-guide.md)

### Implementation Plan (COMPLETED)

- **Sprint 4 Plan:** 8 stories, 47-66 hours, dependency graph, risk mitigation (_bmad-output/implementation-artifacts/sprint-4-implementation-plan-process-logger.md)

### Implementation (IN PROGRESS)

**Story 4.1: Logger Infrastructure (STARTED)**

✅ **Interface Created:** `src/zif_fi_process_logger.intf.abap`
- Methods: message(), log_exception_from_framework(), save()
- Comprehensive documentation with usage examples
- Severity guidelines, best practices

✅ **Class Created:** `src/zcl_fi_process_logger.clas.abap`
- Factory methods: create_new(), attach_existing()
- BAL integration: BAL_LOG_CREATE, BAL_LOG_MSG_ADD, BAL_DB_SAVE, BAL_DB_LOAD
- Exception logging with PREVIOUS chain support
- Defensive logging (log failures without breaking business logic)
- Language fallback warning (ZFI_PROCESS/997)
- T100 message extraction from IF_T100_MESSAGE

🔄 **Next Steps:**
- Unit tests for ZCL_FI_PROCESS_LOGGER
- Story 4.2: Expand message classes (ZFI_PROCESS, ZFI_ALLOC)
- Story 4.3: Integrate with ZCL_FI_PROCESS_INSTANCE
- Story 4.4: Integrate with bgRFC substeps
- Story 4.5: Migrate step code (5 allocation steps)
- Story 4.7: Integration testing & regression
- Story 4.8: Documentation & training

---

## Session Summary

**Duration:** 4 hours (2026-03-11)  
**Facilitator:** AI Agent  
**Participant:** Zdenek Smolik  
**Methodology:** BMAD Brainstorming Workflow (Constraint Mapping → Morphological Analysis → First Principles)

**Outcomes:**
- ✅ Complete architecture designed (9 decisions)
- ✅ All edge cases resolved
- ✅ Documentation created (ADR, guides)
- ✅ Implementation plan created (8 stories, 47-66h)
- ✅ Implementation started (logger interface + class)

**Key Insights:**
- YAGNI principle applied successfully (no premature configuration layer)
- T100 Purist strategy justified for text governance (not just translation)
- Separation of concerns validated (logger vs. statistics)
- Framework auto-logging eliminates dual-path complexity
- BAL-first with extensible API protects future flexibility

**Next Actions:**
1. Complete Story 4.1 (unit tests)
2. Create T100 messages (ZFI_PROCESS, ZFI_ALLOC)
3. Integrate logger with process instance lifecycle
4. Migrate 5 allocation steps (~100 hard-coded texts)
5. Regression testing (18 completed stories must pass)
6. SE63 translation to English
7. Team training & deployment

---

**Session Status:** COMPLETED  
**Implementation Status:** IN PROGRESS (Story 4.1)  
**Target Completion:** Sprint 4 End
