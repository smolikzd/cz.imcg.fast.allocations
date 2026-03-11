# Process Logger Implementation Guide

**Version:** 1.0  
**Date:** 2026-03-11  
**Audience:** Step developers, framework maintainers  
**Related Documents:** ADR-008 (Architecture Decision Record)

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Logger API Reference](#logger-api-reference)
3. [Message Class Structure](#message-class-structure)
4. [Creating T100 Messages](#creating-t100-messages)
5. [Step Implementation Patterns](#step-implementation-patterns)
6. [Exception Handling](#exception-handling)
7. [bgRFC Substep Logging](#bgrfc-substep-logging)
8. [Testing](#testing)
9. [Troubleshooting](#troubleshooting)

---

## Quick Start

### For Step Developers (Most Common Use Case)

**Your step class inherits from `ZCL_FI_PROCESS_STEP`, which provides `mo_log` attribute automatically.**

```abap
CLASS zcl_fi_alloc_step_phase2 IMPLEMENTATION.
  METHOD execute.
    " Log an informational message
    mo_log->message(
      iv_message_class  = 'ZFI_ALLOC'
      iv_message_number = '200'
      iv_message_v1     = lv_company_code
      iv_severity       = 'I'
    ).
    
    " Log a success message
    mo_log->message(
      iv_message_class  = 'ZFI_ALLOC'
      iv_message_number = '202'
      iv_message_v1     = lv_item_count
      iv_severity       = 'S'
    ).
    
    " For errors, raise exception (framework logs automatically)
    IF lv_validation_failed = abap_true.
      RAISE EXCEPTION TYPE zcx_fi_process_error
        EXPORTING
          textid = zcx_fi_process_error=>validation_failed
          value  = lv_company_code.
    ENDIF.
  ENDMETHOD.
ENDCLASS.
```

**That's it! Framework handles:**
- Logger creation/initialization
- Exception logging (automatic)
- Log persistence to BAL
- Multi-language support (via T100)

---

## Logger API Reference

### Interface: `ZIF_FI_PROCESS_LOGGER`

#### Method: `message()`

**Purpose:** Log a business event or informational message

**Signature:**
```abap
METHODS message
  IMPORTING
    iv_message_class  TYPE symsgid       " T100 message class (e.g., 'ZFI_ALLOC')
    iv_message_number TYPE symsgno       " T100 message number (e.g., '042')
    iv_message_v1     TYPE symsgv OPTIONAL  " Variable 1 (replaces &1)
    iv_message_v2     TYPE symsgv OPTIONAL  " Variable 2 (replaces &2)
    iv_message_v3     TYPE symsgv OPTIONAL  " Variable 3 (replaces &3)
    iv_message_v4     TYPE symsgv OPTIONAL  " Variable 4 (replaces &4)
    iv_severity       TYPE symsgty DEFAULT 'I'.  " I/S/W/E/A/X
```

**Parameters:**

| Parameter | Type | Required | Description | Examples |
|-----------|------|----------|-------------|----------|
| `iv_message_class` | `symsgid` | Yes | T100 message class | `'ZFI_PROCESS'`, `'ZFI_ALLOC'` |
| `iv_message_number` | `symsgno` | Yes | T100 message number | `'001'`, `'042'`, `'500'` |
| `iv_message_v1` | `symsgv` | No | First placeholder value | Company code, item count, etc. |
| `iv_message_v2` | `symsgv` | No | Second placeholder value | Period, document number, etc. |
| `iv_message_v3` | `symsgv` | No | Third placeholder value | Cost center, amount, etc. |
| `iv_message_v4` | `symsgv` | No | Fourth placeholder value | Currency, status, etc. |
| `iv_severity` | `symsgty` | No (default 'I') | Message severity | See severity table below |

**Severity Types:**

| Code | Name | Usage | SLG1 Display |
|------|------|-------|--------------|
| `I` | Information | Progress updates, general info | ℹ️ Blue |
| `S` | Success | Successful operations | ✅ Green |
| `W` | Warning | Non-critical issues | ⚠️ Yellow |
| `E` | Error | Business errors | ❌ Red |
| `A` | Abort | Critical errors (termination) | 🛑 Red |
| `X` | Exit | System errors (dump) | 💀 Red |

**Example Usage:**

```abap
" Informational message (progress update)
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '100'
  iv_message_v1     = lv_company_count
  iv_severity       = 'I'
).

" Success message (operation completed)
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '101'
  iv_message_v1     = lv_company_code
  iv_message_v2     = lv_item_count
  iv_severity       = 'S'
).

" Warning message (non-critical issue)
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '502'
  iv_message_v1     = lv_cost_center
  iv_severity       = 'W'
).
```

---

#### Method: `log_exception_from_framework()` (Framework-Only)

**⚠️ DO NOT CALL FROM STEP CODE - Framework uses this internally**

**Purpose:** Automatically logs exceptions caught by framework orchestration layer

**Step developers:** Just raise exceptions - framework logs them automatically!

```abap
" Step code (correct):
RAISE EXCEPTION TYPE zcx_fi_process_error
  EXPORTING
    textid = zcx_fi_process_error=>validation_failed
    value  = lv_company_code.

" Framework catches and logs automatically via:
" mo_log->log_exception_from_framework( lx_error ).
```

---

#### Method: `save()` (Framework-Only)

**⚠️ DO NOT CALL FROM STEP CODE - Framework uses this internally**

**Purpose:** Persists all logged messages to BAL database

Framework calls this automatically after:
- Step execution completes
- Process instance status changes
- bgRFC substep finishes

---

## Message Class Structure

### ZFI_PROCESS (Framework Messages)

**Purpose:** Process instance lifecycle, orchestration, framework operations

**Number Ranges:**

| Range | Purpose | Examples |
|-------|---------|----------|
| 001-019 | Instance lifecycle | Instance created, loaded, status changed |
| 020-039 | Step execution | Step started, completed, failed |
| 040-059 | Queued/bgRFC execution | Substeps queued, substep started/completed |
| 060-099 | Framework errors | Invalid UUID, instance not found, invalid status |
| 996-999 | Infrastructure errors | Logger failures, missing translations |

**Key Messages:**

```
001  Process instance created (type &1, UUID &2)
002  Process instance loaded (UUID &1, status &2)
003  Process status changed: &1 → &2
020  Step &1 execution started
021  Step &1 completed in &2 seconds
022  Step &1 execution failed
060  Invalid process instance UUID: &1
061  Process instance not found: &1
996  Logger not initialized - framework error
997  Message &1/&2 not translated to language &3
998  Failed to log message &1/&2 - check BAL configuration
```

---

### ZFI_ALLOC (Business Messages)

**Purpose:** Allocation-specific business logic, validation, calculation, data processing

**Number Ranges:**

| Range | Purpose | Step |
|-------|---------|------|
| 001-099 | INIT step messages | Initialization, master data loading |
| 100-199 | PHASE1 step messages | Phase 1 processing, rule application |
| 200-299 | PHASE2 step messages | Phase 2 processing (most verbose) |
| 300-399 | PHASE3 step messages | Phase 3 finalization, document creation |
| 400-499 | CORR_BCHE step messages | Correction batch processing |
| 500-599 | General allocation errors | Validation errors, calculation errors |

**Example Messages:**

```
001  Initialization started for company &1
002  Loaded &1 cost centers for company &2
003  Validation failed: No cost centers for company &1

100  Phase 1 started: Processing &1 companies
101  Company &1: Loaded &2 allocation rules
102  Company &1: Applied rule &2 to &3 cost centers
103  Error: Invalid allocation rule &1 for company &2

200  Phase 2 started: Queued &1 substeps
201  Substep &1: Processing company &2, period &3
202  Substep &1: Calculated &2 allocation items
203  Substep &1: Error in calculation for cost center &2

300  Phase 3 started: Finalizing allocations
301  Company &1: Created &2 FI documents
302  Error: Document creation failed for company &1

400  Correction batch processing started
401  Processing batch &1 with &2 items
402  Correction applied to document &1

500  Validation error: &1
501  Calculation error: &1
502  Master data incomplete: &1
```

---

## Creating T100 Messages

### Using ABAP Development Tools (ADT)

**Step 1: Open Message Class**

1. Navigate to `src/zfi_process.msag.xml` or `src/zfi_alloc.msag.xml`
2. Right-click → Open With → ABAP Message Class Editor

**Step 2: Add New Message**

1. Click "New Message" button
2. Enter message number (follow number range conventions)
3. Enter message text (max 73 characters)
4. Use `&1`, `&2`, `&3`, `&4` as placeholders
5. Save and activate

**Step 3: Translate (if needed)**

1. Right-click message class → Translation
2. Select target language (EN, DE, etc.)
3. Enter translated text
4. Save translation

---

### Using SE91 Transaction (Classic GUI)

**Step 1: Open SE91**

1. Execute transaction `SE91`
2. Enter message class: `ZFI_PROCESS` or `ZFI_ALLOC`
3. Click "Display" or "Change"

**Step 2: Add Message**

1. Find next available message number in appropriate range
2. Click "New Entries" or insert row
3. Enter message number and text
4. Use `&` for placeholders (e.g., "Processed & items")
5. Save

**Step 3: Translate**

1. Execute transaction `SE63`
2. Select "Short Texts" → "ABAP Objects" → "Messages"
3. Enter message class and target language
4. Translate all messages
5. Save

---

### Message Text Guidelines

**✅ DO:**
- Keep messages concise (≤73 characters)
- Use placeholders for dynamic values (`&1`, `&2`, etc.)
- Start with action verb ("Processing...", "Loaded...", "Failed...")
- Be specific about context (include step name, entity type)

**❌ DON'T:**
- Exceed 73 characters (split into multiple messages if needed)
- Hard-code dynamic values in text
- Use technical jargon (messages visible to business users)
- Include sensitive data (PII, credentials)

**Examples:**

```
✅ GOOD:
  "Processing company &1, period &2"
  "Loaded &1 cost centers"
  "Validation failed: &1"

❌ BAD:
  "Now we are processing the company code 1000 for fiscal period 202601 which was loaded from the database"  ← Too long
  "Processing company 1000"  ← Hard-coded value
  "CX_SY_OPEN_SQL_ERROR in line 42"  ← Too technical
```

---

## Step Implementation Patterns

### Pattern 1: Progress Logging

Log key milestones in business logic execution:

```abap
METHOD execute.
  " Log start
  mo_log->message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '100'
    iv_message_v1     = lines( lt_companies )
    iv_severity       = 'I'
  ).
  
  " Business logic
  LOOP AT lt_companies INTO DATA(ls_company).
    " Log progress per company
    mo_log->message(
      iv_message_class  = 'ZFI_ALLOC'
      iv_message_number = '101'
      iv_message_v1     = ls_company-company_code
      iv_message_v2     = lines( lt_rules )
      iv_severity       = 'I'
    ).
    
    " ... processing logic ...
  ENDLOOP.
  
  " Log completion
  mo_log->message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '105'
    iv_message_v1     = lv_total_items
    iv_severity       = 'S'
  ).
ENDMETHOD.
```

---

### Pattern 2: Conditional Warning Logging

Log warnings for non-critical issues that don't stop processing:

```abap
METHOD execute.
  LOOP AT lt_cost_centers INTO DATA(ls_cc).
    " Check for potential issue
    IF ls_cc-allocation_percentage = 0.
      " Log warning (but continue processing)
      mo_log->message(
        iv_message_class  = 'ZFI_ALLOC'
        iv_message_number = '502'
        iv_message_v1     = ls_cc-cost_center
        iv_severity       = 'W'
      ).
    ENDIF.
    
    " Continue processing
    " ...
  ENDLOOP.
ENDMETHOD.
```

---

### Pattern 3: Error Logging with Exception

For critical errors, raise exception (framework logs automatically):

```abap
METHOD execute.
  " Validate input
  IF lt_companies IS INITIAL.
    " Raise exception (framework logs this automatically)
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING
        textid = zcx_fi_process_error=>validation_failed
        value  = 'No companies to process'.
  ENDIF.
  
  " Business logic
  TRY.
      perform_calculation( ).
    CATCH zcx_fi_alloc_plan INTO DATA(lx_error).
      " Re-raise as framework exception (framework logs)
      RAISE EXCEPTION TYPE zcx_fi_process_error
        EXPORTING
          textid   = zcx_fi_process_error=>technical_error
          value    = 'Calculation failed'
          previous = lx_error.  " Chain exceptions
  ENDTRY.
ENDMETHOD.
```

---

### Pattern 4: Verbose Logging (Development/Debug)

During development, log detailed information:

```abap
METHOD execute.
  " Log entry with input parameters
  mo_log->message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '200'
    iv_message_v1     = lv_company_code
    iv_message_v2     = lv_fiscal_period
    iv_severity       = 'I'
  ).
  
  " Log intermediate results
  mo_log->message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '201'
    iv_message_v1     = lines( lt_source_data )
    iv_severity       = 'I'
  ).
  
  " Log data transformations
  mo_log->message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '202'
    iv_message_v1     = lv_before_count
    iv_message_v2     = lv_after_count
    iv_severity       = 'I'
  ).
  
  " Production tip: Add configuration to reduce verbosity later
ENDMETHOD.
```

---

## Exception Handling

### Automatic Exception Logging

**Framework automatically logs all exceptions caught during step execution.**

Step developers just raise exceptions - no explicit logging needed:

```abap
" Step code:
METHOD execute.
  IF lv_validation_failed = abap_true.
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING
        textid = zcx_fi_process_error=>validation_failed
        value  = lv_company_code.
  ENDIF.
ENDMETHOD.

" Framework automatically logs this exception with:
" - Exception class name
" - Exception text (from IF_T100_MESSAGE)
" - Full exception chain (PREVIOUS attribute)
" - Timestamp, user, program context
```

---

### Exception Chaining

Preserve original exception context using `previous` parameter:

```abap
METHOD execute.
  TRY.
      " Call external service
      zcl_external_service=>validate( lv_data ).
      
    CATCH zcx_external_service_error INTO DATA(lx_external).
      " Re-raise as framework exception with chain
      RAISE EXCEPTION TYPE zcx_fi_process_error
        EXPORTING
          textid   = zcx_fi_process_error=>technical_error
          value    = |External validation failed: { lv_data }|
          previous = lx_external.  " ← Preserves original error
  ENDTRY.
ENDMETHOD.
```

**Framework logs the full chain:**
```
[E] Step PHASE2 execution failed
  └─> Technical error: External validation failed: DATA123
      └─> External service timeout (original exception)
```

---

## bgRFC Substep Logging

### Automatic Parent Log Attachment

**Substeps automatically attach to parent instance's BAL log.**

No special code needed - framework handles this:

```abap
" Parent step (queues substeps):
METHOD execute.
  " Log queuing
  mo_log->message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '200'
    iv_message_v1     = lines( lt_substeps )
    iv_severity       = 'I'
  ).
  
  " Queue substeps for bgRFC execution
  LOOP AT lt_substeps INTO DATA(ls_substep).
    queue_substep_for_bgrfc( ls_substep ).
  ENDLOOP.
ENDMETHOD.

" Substep execution (in bgRFC work process):
METHOD execute_substep.
  " Log substep start (goes to PARENT log!)
  mo_log->message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '201'
    iv_message_v1     = iv_substep_index
    iv_message_v2     = ls_context-company_code
    iv_severity       = 'I'
  ).
  
  " Business logic
  " ...
  
  " Log substep completion (goes to PARENT log!)
  mo_log->message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '202'
    iv_message_v1     = iv_substep_index
    iv_message_v2     = lv_item_count
    iv_severity       = 'S'
  ).
ENDMETHOD.
```

**Result in SLG1 (single log, chronological order):**
```
10:15:28  [I] Phase 2 started: Queued 10 substeps
10:15:29  [I] Substep 1: Processing company 1000, period 01
10:15:29  [I] Substep 3: Processing company 1000, period 03  ← Concurrent!
10:15:29  [I] Substep 2: Processing company 1000, period 02  ← Concurrent!
10:15:30  [S] Substep 1: Calculated 45 allocation items
10:15:31  [S] Substep 3: Calculated 52 allocation items
...
```

---

## Testing

### Unit Testing with Logger Mock

Create mock logger for unit tests:

```abap
CLASS lcl_logger_mock DEFINITION FOR TESTING.
  PUBLIC SECTION.
    INTERFACES zif_fi_process_logger.
    DATA mt_messages TYPE STANDARD TABLE OF ty_message.
ENDCLASS.

CLASS lcl_logger_mock IMPLEMENTATION.
  METHOD zif_fi_process_logger~message.
    APPEND VALUE #(
      message_class  = iv_message_class
      message_number = iv_message_number
      message_v1     = iv_message_v1
      severity       = iv_severity
    ) TO mt_messages.
  ENDMETHOD.
  
  METHOD zif_fi_process_logger~log_exception_from_framework.
    " Capture exception
  ENDMETHOD.
  
  METHOD zif_fi_process_logger~save.
    " No-op in tests
  ENDMETHOD.
ENDCLASS.

" Test class:
CLASS ltc_step_phase2 DEFINITION FOR TESTING.
  PRIVATE SECTION.
    DATA mo_step TYPE REF TO zcl_fi_alloc_step_phase2.
    DATA mo_logger_mock TYPE REF TO lcl_logger_mock.
    
    METHODS test_execution FOR TESTING.
ENDCLASS.

CLASS ltc_step_phase2 IMPLEMENTATION.
  METHOD test_execution.
    " Setup
    mo_logger_mock = NEW lcl_logger_mock( ).
    mo_step = NEW zcl_fi_alloc_step_phase2( ).
    mo_step->initialize_logger( mo_logger_mock ).
    
    " Execute
    mo_step->execute( ).
    
    " Assert messages logged
    cl_abap_unit_assert=>assert_not_initial(
      act = lines( mo_logger_mock->mt_messages )
      msg = 'No messages logged'
    ).
    
    " Assert specific message
    cl_abap_unit_assert=>assert_equals(
      act = mo_logger_mock->mt_messages[ 1 ]-message_number
      exp = '200'
      msg = 'Expected Phase 2 start message'
    ).
  ENDMETHOD.
ENDCLASS.
```

---

### Integration Testing

Test with real BAL logging:

```abap
CLASS ltc_integration DEFINITION FOR TESTING.
  PRIVATE SECTION.
    DATA mo_instance TYPE REF TO zcl_fi_process_instance.
    
    METHODS test_full_execution FOR TESTING.
    METHODS cleanup FOR TESTING.
ENDCLASS.

CLASS ltc_integration IMPLEMENTATION.
  METHOD test_full_execution.
    " Create real instance (creates real logger)
    mo_instance = zcl_fi_process_instance=>create(
      iv_process_type = 'ALLOCATION'
      is_context      = VALUE #( company_code = '1000' )
    ).
    
    " Execute step (logs to BAL)
    DATA(lo_step) = zcl_fi_alloc_step_phase2=>create( ).
    lo_step->initialize_logger( mo_instance->get_logger( ) ).
    lo_step->execute( ).
    
    " Verify log saved to BAL
    " (query BALHDR/BALDAT by external number = instance UUID)
  ENDMETHOD.
  
  METHOD cleanup.
    " Clean up test logs in BAL
  ENDMETHOD.
ENDCLASS.
```

---

## Troubleshooting

### Problem: "Logger not initialized" dump

**Symptom:** CX_SY_REF_IS_INITIAL when calling `mo_log->message()`

**Cause:** Framework didn't initialize logger before step execution

**Solution:**
1. Check that step inherits from `ZCL_FI_PROCESS_STEP`
2. Verify framework calls `initialize_logger()` before `execute()`
3. Add defensive check:
   ```abap
   METHOD execute.
     ASSERT mo_log IS BOUND.  " Fails fast with clear message
     " ... business logic ...
   ENDMETHOD.
   ```

---

### Problem: Message text not displayed in SLG1

**Symptom:** SLG1 shows "Message ZFI_ALLOC/042 not found"

**Cause:** Message not created in T100 or not translated

**Solution:**
1. Check message exists: SE91 → ZFI_ALLOC → verify message number
2. Check translation: SE63 → verify message translated to current language
3. Logger will log warning: "Message ZFI_ALLOC/042 not translated to language EN"

---

### Problem: BAL log not found in SLG1

**Symptom:** Can't find log by external number (instance UUID)

**Cause:** Log not saved, or using wrong external number

**Solution:**
1. Verify external number = instance UUID (check BALHDR table)
2. Ensure `mo_log->save()` called (framework does this automatically)
3. Check BAL object/subobject match: Object = 'ZFI_PROCESS', Subobject = process type
4. Use SLG1 with wildcards: External number = `*{uuid_suffix}*`

---

### Problem: Too many messages in log

**Symptom:** SLG1 slow, log has 10,000+ messages

**Cause:** Verbose logging in loops

**Solution:**
1. Review logging in loops - log summaries, not every iteration:
   ```abap
   " ❌ BAD: Log every item
   LOOP AT lt_items INTO DATA(ls_item).
     mo_log->message( ... ).  " 10,000 messages!
   ENDLOOP.
   
   " ✅ GOOD: Log summary
   mo_log->message(
     iv_message_number = '202'
     iv_message_v1     = lines( lt_items )  " "Processed 10,000 items"
   ).
   ```

2. Use appropriate severity (don't log every INFO message)
3. Consider log segmentation (future enhancement)

---

### Problem: Messages in wrong language

**Symptom:** Czech messages shown to English user

**Cause:** Missing SE63 translation

**Solution:**
1. Execute SE63
2. Translate all messages in ZFI_PROCESS and ZFI_ALLOC
3. Logger automatically uses `sy-langu` (current user language)
4. If translation missing, logger logs warning (ZFI_PROCESS/997)

---

## Best Practices

### ✅ DO

- **Log business milestones** (phase started, company processed, documents created)
- **Use appropriate severity** (I for progress, S for success, W for warnings, E for errors)
- **Provide context in variables** (company code, item count, period, etc.)
- **Raise exceptions for errors** (framework logs automatically)
- **Keep messages concise** (≤73 chars, split if needed)
- **Translate all messages** (SE63 workflow)

### ❌ DON'T

- **Log every loop iteration** (log summaries instead)
- **Hard-code values in messages** (use placeholders: &1, &2)
- **Include sensitive data** (PII, passwords, tax IDs)
- **Call framework-only methods** (save(), log_exception_from_framework())
- **Forget to check mo_log IS BOUND** (add ASSERT for safety)
- **Mix business logic with logging** (log should not affect business outcome)

---

## Reference

### Related Documents
- [ADR-008: Process Logger Architecture](../architecture/ADR-008-process-logger-architecture.md)
- [Migration Guide: Process Logger](./process-logger-migration-guide.md)
- Constitution v1.0.0

### T100 Message Classes
- **ZFI_PROCESS:** Framework messages (SE91)
- **ZFI_ALLOC:** Allocation business messages (SE91)

### Transactions
- **SLG1:** Display application logs
- **SE91:** Message class maintenance
- **SE63:** Translation (messages)
- **STVARV:** Maintain TVARVC parameters (future: log configuration)

---

**Version History:**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-11 | Zdenek Smolik | Initial version |

---

**Questions? Contact:** Zdenek Smolik (smolik@imcg.cz)
