# Process Logger Architecture
## Training Presentation

**Sprint 4: Exception Handling and Message Persistence**  
**Date:** March 2026  
**Presenter:** Zdenek Smolik  
**Duration:** 45 minutes

---

## Slide 1: Agenda

1. **Problem Statement** (5 min)
2. **Architecture Overview** (10 min)
3. **Developer API** (10 min)
4. **Migration Results** (5 min)
5. **Live Demo** (10 min)
6. **Q&A** (5 min)

---

## Slide 2: The Problem We Solved

### Before Sprint 4

```abap
" ❌ Hard-coded Czech text
MESSAGE s000(zfi_alloc) WITH |Zpracováno { lv_count } položek| 
  INTO lv_dummy.
mo_log->log_sy_msg( ).
```

### Issues

- 🇨🇿 **Czech-only messages** → No international deployment
- 📝 **~100 hard-coded strings** → Text changes require code modifications
- 🔀 **Inconsistent patterns** → Mix of MESSAGE + mo_log->log_sy_msg()
- 📊 **No unified audit trail** → Difficult to track process execution
- 🔧 **Poor text governance** → Business users can't update messages

---

## Slide 3: Solution Architecture

### Core Decisions

1. **✅ BAL-Only Persistence** → Leverage SAP Application Log
2. **✅ T100 Purist Language Strategy** → All messages via SE91/SE63
3. **✅ Inherited Logger API** → `mo_log` attribute in base class
4. **✅ Hierarchical Message Classes** → ZFI_PROCESS + ZFI_ALLOC
5. **✅ Framework Auto-Logging** → Exceptions logged automatically

### After Sprint 4

```abap
" ✅ T100 message with placeholders
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '202'  " T100: "Zpracováno &1 položek"
  iv_message_v1     = lv_count
  iv_severity       = 'S'
).
```

---

## Slide 4: Architecture Components

### New Components Delivered

```
┌─────────────────────────────────────────────────────┐
│ Step Classes (ZCL_FI_ALLOC_STEP_*)                 │
│   - Inherits mo_log from base class                │
│   - Calls mo_log->message()                         │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────┐
│ ZIF_FI_PROCESS_LOGGER (Interface)                  │
│   - message()                                       │
│   - log_exception_from_framework()                  │
│   - save()                                          │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────┐
│ ZCL_FI_PROCESS_LOGGER (Implementation)             │
│   - Wraps SAP BAL (BALI functional methods)        │
│   - Handles T100 message rendering                  │
│   - External number = instance UUID                 │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────┐
│ SAP Application Log (BAL)                          │
│   - BALHDR (log headers)                            │
│   - BALDAT (log messages)                           │
│   - SLG1 (display transaction)                      │
└─────────────────────────────────────────────────────┘
```

---

## Slide 5: Message Class Structure

### ZFI_PROCESS (Framework Messages)

| Range | Purpose | Example |
|-------|---------|---------|
| 001-019 | Instance lifecycle | "Process instance created (type &1, UUID &2)" |
| 020-039 | Step execution | "Step &1 completed in &2 seconds" |
| 040-059 | bgRFC execution | "Substep &1 started (company &2, period &3)" |
| 996-999 | Infrastructure errors | "Failed to log message &1/&2" |

### ZFI_ALLOC (Business Messages)

| Range | Purpose | Step | Count |
|-------|---------|------|-------|
| 001-099 | Initialization | INIT | 20 |
| 100-199 | Phase 1 processing | PHASE1 | 25 |
| 200-299 | Phase 2 processing | PHASE2 | 27 |
| 300-399 | Phase 3 finalization | PHASE3 | 25 |
| 400-499 | Correction batch | CORR_BCHE | 12 |
| 500-599 | General errors | All | 9 |

**Total:** 21 → 118 messages (5.6x expansion)

---

## Slide 6: Developer API - Basic Usage

### Step 1: Your Step Inherits Logger

```abap
CLASS zcl_fi_alloc_step_phase2 IMPLEMENTATION.
  METHOD execute.
    " mo_log is automatically available (inherited from base class)
    " Framework initializes mo_log before calling execute()
  ENDMETHOD.
ENDCLASS.
```

### Step 2: Log Messages

```abap
" Informational message
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '200'
  iv_message_v1     = lv_company_code
  iv_severity       = 'I'
).

" Success message
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '202'
  iv_message_v1     = lv_item_count
  iv_severity       = 'S'
).
```

---

## Slide 7: Developer API - Exception Handling

### NO Explicit Exception Logging Needed!

```abap
METHOD execute.
  " Validate input
  IF lt_companies IS INITIAL.
    " Just raise exception - framework logs automatically!
    RAISE EXCEPTION TYPE zcx_fi_process_error
      EXPORTING
        textid = zcx_fi_process_error=>validation_failed
        value  = 'No companies to process'.
  ENDIF.
  
  " Business logic...
ENDMETHOD.
```

**Framework catches and logs automatically:**
- Exception class name
- Exception text (from IF_T100_MESSAGE)
- Full exception chain (PREVIOUS attribute)
- Timestamp, user, program context

---

## Slide 8: Developer API - Message Severity

| Code | Name | When to Use | Example |
|------|------|-------------|---------|
| `I` | Information | Progress updates | "Processing company 1000" |
| `S` | Success | Successful operations | "Created 45 allocation items" |
| `W` | Warning | Non-critical issues | "Cost center inactive" |
| `E` | Error | Business errors | "Validation failed" |
| `A` | Abort | Critical errors | "Config missing" |

**Default to `I` if unsure.**

---

## Slide 9: bgRFC Substep Logging

### Automatic Parent Log Attachment

```abap
" Parent step (queues substeps)
METHOD execute.
  mo_log->message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '200'
    iv_message_v1     = lines( lt_substeps )
    iv_severity       = 'I'
  ).
  
  " Queue substeps...
ENDMETHOD.

" Substep execution (in bgRFC work process)
METHOD execute_substep.
  " Messages go to PARENT log automatically!
  mo_log->message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '201'
    iv_message_v1     = iv_substep_index
    iv_severity       = 'I'
  ).
ENDMETHOD.
```

**Result:** All substep messages appear in parent log (chronological order)

---

## Slide 10: Viewing Logs in SLG1

### Search by Instance UUID

1. **Execute transaction SLG1**
2. **Enter search criteria:**
   - Object: `ZFI_PROCESS`
   - Subobject: `ALLOCATION`
   - External number: `<instance_uuid>`
3. **Execute** (F8)
4. **Double-click log** to view messages

### Log Display Features

- ✅ Chronological message order (parent + substeps)
- ✅ Severity color-coding (I=blue, S=green, W=yellow, E=red)
- ✅ Message text in current user language (CS/EN)
- ✅ Message details (timestamp, user, program)
- ✅ Exception call stack (for errors)

---

## Slide 11: Migration Results

### What We Delivered (Story 4-5)

| Metric | Value |
|--------|-------|
| **Hard-coded strings migrated** | 97 |
| **T100 messages created** | 97 (ZFI_ALLOC expanded 21 → 118) |
| **Step classes migrated** | 5 (INIT, PHASE1, PHASE2, PHASE3, CORR_BCHE) |
| **Lines of code added** | +1,027 (~58% expansion) |
| **Total effort** | ~24 hours (within 20-40h estimate) |
| **Languages supported** | Czech + English (bilingual) |

### Code Impact Example (PHASE2)

- **Before:** 638 lines
- **After:** 959 lines
- **Growth:** +321 lines (+50%)
- **Reason:** Defensive TRY-CATCH pattern (~10-15 lines per message)

---

## Slide 12: Migration Pattern

### Standard Pattern (Used 97 Times)

```abap
" Before (hard-coded Czech):
MESSAGE s000(zfi_alloc) WITH |Zpracováno { lv_count } položek| 
  INTO lv_dummy.
mo_log->log_sy_msg( ).

" After (T100 + logger):
MESSAGE s202(zfi_alloc) WITH lv_count INTO DATA(lv_dummy).
IF mo_log IS BOUND.
  TRY.
      mo_log->message(
        iv_message_class  = 'ZFI_ALLOC'
        iv_message_number = '202'
        iv_message_v1     = lv_count
        iv_severity       = 'S'
      ).
    CATCH zcx_fi_process_error.
      " Logger failed - continue with business logic
  ENDTRY.
ENDIF.
```

**T100 Message (ZFI_ALLOC/202):**
- Czech: `Zpracováno &1 položek`
- English: `Processed &1 items`

---

## Slide 13: Best Practices

### ✅ DO

- **Log business milestones** (phase started, company processed)
- **Use appropriate severity** (I for progress, S for success, E for errors)
- **Provide context in variables** (company code, item count, period)
- **Raise exceptions for errors** (framework logs automatically)
- **Keep messages concise** (≤73 chars, split if needed)

### ❌ DON'T

- **Log every loop iteration** (log summaries instead)
- **Hard-code values in messages** (use placeholders: &1, &2)
- **Include sensitive data** (PII, passwords, tax IDs)
- **Call framework-only methods** (save(), log_exception_from_framework())
- **Forget to check mo_log IS BOUND** (add ASSERT for safety)

---

## Slide 14: Live Demo

### Demo Script

1. **Show T100 messages in SE91**
   - Navigate to ZFI_ALLOC
   - Show message 202 (Czech + English)

2. **Show step code with logger**
   - Open `zcl_fi_alloc_step_phase2`
   - Show `mo_log->message()` call

3. **Execute allocation process**
   - Run allocation for company 1000, period 01
   - Capture instance UUID

4. **View log in SLG1**
   - Search by external number (UUID)
   - Show messages in chronological order
   - Show bilingual text (Czech vs. English)

5. **Show exception logging**
   - Trigger validation error
   - Show exception in SLG1 with call stack

---

## Slide 15: Resources & Next Steps

### Documentation

- **Architecture:** `docs/architecture/ADR-008-process-logger-architecture.md`
- **Implementation Guide:** `docs/developer-guides/process-logger-implementation-guide.md`
- **Migration Guide:** `docs/developer-guides/process-logger-migration-guide.md`
- **Quick Reference:** `docs/training/process-logger-quick-reference.md` (pending)

### Key Transactions

- **SE91:** Message class maintenance
- **SE63:** Translation (messages)
- **SLG1:** Display application logs
- **STVARV:** TVARVC parameters (future: log configuration)

### Next Steps

- **Story 4-7:** Integration testing & regression (in progress)
- **Story 4-8:** Documentation & training (this presentation!)
- **Production:** Transport to QAS/PRD (pending testing results)

### Questions?

**Contact:** Zdenek Smolik (smolik@imcg.cz)

---

## Appendix: Technical Deep Dive

### Logger Lifecycle

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
  SELECT SINGLE * FROM zfi_process_instance 
    INTO CORRESPONDING FIELDS OF @ro_instance ...
  
  " Attach to existing BAL log
  ro_instance->mo_log = zcl_fi_process_logger=>attach_existing(
    iv_external_number = ro_instance->mv_uuid
  ).
ENDMETHOD.
```

---

**End of Presentation**
