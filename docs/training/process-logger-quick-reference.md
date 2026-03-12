# Process Logger Quick Reference

**Version:** 1.0  
**Date:** March 2026  
**For:** Step Developers

---

## 🚀 Quick Start (3 Steps)

### Step 1: Your step inherits logger automatically

```abap
CLASS zcl_fi_alloc_step_phase2 DEFINITION
  INHERITING FROM zcl_fi_process_step.  " ← mo_log available!
```

### Step 2: Log messages

```abap
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '202'
  iv_message_v1     = lv_count
  iv_severity       = 'S'  " I/S/W/E/A
).
```

### Step 3: Raise exceptions (framework logs automatically)

```abap
RAISE EXCEPTION TYPE zcx_fi_process_error
  EXPORTING textid = zcx_fi_process_error=>validation_failed
            value  = lv_error_context.
```

---

## 📊 Message Severity

| Code | Name | Usage | Color |
|------|------|-------|-------|
| `I` | Information | Progress updates | Blue |
| `S` | Success | Successful operations | Green |
| `W` | Warning | Non-critical issues | Yellow |
| `E` | Error | Business errors | Red |
| `A` | Abort | Critical errors | Red |

**Default:** Use `I` if unsure

---

## 📝 Message Class Ranges

### ZFI_PROCESS (Framework)

- **001-019:** Instance lifecycle
- **020-039:** Step execution
- **040-059:** bgRFC execution
- **996-999:** Infrastructure errors

### ZFI_ALLOC (Business)

- **001-099:** INIT step (20 messages)
- **100-199:** PHASE1 step (25 messages)
- **200-299:** PHASE2 step (27 messages)
- **300-399:** PHASE3 step (25 messages)
- **400-499:** CORR_BCHE step (12 messages)
- **500-599:** General errors (9 messages)

---

## 💻 Code Patterns

### Pattern A: Simple Message

```abap
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '016'
  iv_severity       = 'S'
).
```

### Pattern B: Message with Variables (1-4 placeholders)

```abap
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '202'
  iv_message_v1     = lv_company_code
  iv_message_v2     = lv_item_count
  iv_message_v3     = lv_fiscal_period
  iv_message_v4     = lv_status
  iv_severity       = 'I'
).
```

### Pattern C: Validation Error with Result Structure

```abap
IF mv_allocation_id IS INITIAL.
  MESSAGE e315(zfi_alloc) INTO rs_result-message.
  IF mo_log IS BOUND.
    TRY.
        mo_log->message(
          iv_message_class  = 'ZFI_ALLOC'
          iv_message_number = '315'
          iv_severity       = 'E'
        ).
      CATCH zcx_fi_process_error.
    ENDTRY.
  ENDIF.
  rs_result-success = abap_false.
  RETURN.
ENDIF.
```

### Pattern D: Exception Handling (Framework Auto-Logs)

```abap
" Just raise exception - NO explicit logging needed!
IF lv_validation_failed = abap_true.
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid = zcx_fi_process_error=>validation_failed
      value  = lv_company_code.
ENDIF.
```

---

## 🔍 Viewing Logs (SLG1)

### Search by Instance UUID

1. **Transaction:** `SLG1`
2. **Object:** `ZFI_PROCESS`
3. **Subobject:** `ALLOCATION`
4. **External number:** `<instance_uuid>`
5. **Execute:** F8
6. **Double-click log** to view messages

### Search by Date

1. **Transaction:** `SLG1`
2. **Object:** `ZFI_PROCESS`
3. **Date:** Today's date
4. **Execute:** F8

---

## 📖 T100 Message Guidelines

### DO ✅

- **Keep messages ≤73 characters**
- **Use placeholders:** `&1`, `&2`, `&3`, `&4`
- **Start with action verb:** "Processing...", "Loaded...", "Failed..."
- **Be specific:** Include step name, entity type
- **Translate in SE63:** Czech + English minimum

### DON'T ❌

- **Exceed 73 characters** (split into multiple messages)
- **Hard-code dynamic values** (use placeholders)
- **Use technical jargon** (messages visible to business users)
- **Include sensitive data** (PII, credentials)

### Example T100 Messages

```
✅ GOOD:
  "Processing company &1, period &2"
  "Loaded &1 cost centers"
  "Validation failed: &1"

❌ BAD:
  "Now we are processing the company code 1000..."  ← Too long
  "Processing company 1000"  ← Hard-coded value
  "CX_SY_OPEN_SQL_ERROR in line 42"  ← Too technical
```

---

## 🛠️ Creating T100 Messages

### Using SE91

1. **Execute:** `SE91`
2. **Message class:** `ZFI_ALLOC` (or `ZFI_PROCESS`)
3. **Change mode**
4. **Add message:** Enter number and text
5. **Use placeholders:** `&1`, `&2`, `&3`, `&4`
6. **Save and activate**

### Using ADT

1. **Open:** `src/zfi_alloc.msag.xml`
2. **Right-click:** "Open With" → "ABAP Message Class Editor"
3. **New Message:** Enter number and text
4. **Save:** Ctrl+S
5. **Activate:** Ctrl+F3

---

## ⚠️ Common Pitfalls

### Pitfall 1: Logger Not Initialized

**Symptom:** CX_SY_REF_IS_INITIAL dump  
**Solution:** Add defensive check

```abap
METHOD execute.
  ASSERT mo_log IS BOUND.  " Fails fast if not initialized
  " ... business logic ...
ENDMETHOD.
```

### Pitfall 2: Message Not Found in SLG1

**Symptom:** "Message ZFI_ALLOC/042 not found"  
**Cause:** Message not created or not translated  
**Solution:**
1. Check SE91 → ZFI_ALLOC → verify message exists
2. Check SE63 → verify translation

### Pitfall 3: Too Many Messages in Loop

**Problem:**
```abap
❌ BAD:
LOOP AT lt_items INTO DATA(ls_item).
  mo_log->message( ... ).  " 10,000 messages!
ENDLOOP.
```

**Solution:**
```abap
✅ GOOD:
LOOP AT lt_items INTO DATA(ls_item).
  " ... process item ...
ENDLOOP.
mo_log->message(
  iv_message_v1 = lines( lt_items )  " "Processed 10,000 items"
  ...
).
```

---

## 📚 Documentation Links

### Core Documents

- **Architecture:** `docs/architecture/ADR-008-process-logger-architecture.md`
- **Implementation Guide:** `docs/developer-guides/process-logger-implementation-guide.md`
- **Migration Guide:** `docs/developer-guides/process-logger-migration-guide.md`

### Key Transactions

- **SE91:** Message class maintenance
- **SE63:** Translation (messages)
- **SLG1:** Display application logs
- **BALHDR:** BAL log headers (database table)
- **BALDAT:** BAL log messages (database table)

---

## 🎯 Best Practices Summary

### DO ✅

- Log business milestones (phase started, company processed)
- Use appropriate severity (I/S/W/E)
- Provide context in variables (company, period, count)
- Raise exceptions for errors (framework logs automatically)
- Keep messages concise (≤73 chars)

### DON'T ❌

- Log every loop iteration (log summaries)
- Hard-code values in messages (use &1, &2)
- Include sensitive data (PII, passwords)
- Call save() or log_exception_from_framework() (framework-only)
- Forget defensive check (ASSERT mo_log IS BOUND)

---

## 🚨 Emergency Contacts

**Technical Support:** Zdenek Smolik (smolik@imcg.cz)  
**Documentation:** See `docs/` directory in Git repository  
**Training:** Process Logger Training Presentation (45 minutes)

---

## 📊 Migration Statistics (Story 4-5)

- **Messages migrated:** 97 hard-coded strings → T100
- **Step classes updated:** 5 (INIT, PHASE1, PHASE2, PHASE3, CORR_BCHE)
- **Code expansion:** +1,027 lines (+58%)
- **Languages supported:** Czech + English (bilingual)
- **Total effort:** ~24 hours

---

**Version:** 1.0 (March 2026)  
**Print this page for your desk!**
