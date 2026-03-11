# Process Logger Migration Guide

**Version:** 1.0  
**Date:** 2026-03-11  
**Audience:** Developers migrating existing step code  
**Estimated Effort:** 20-40 hours for complete migration

---

## Table of Contents

1. [Migration Overview](#migration-overview)
2. [Pre-Migration Checklist](#pre-migration-checklist)
3. [Migration Phases](#migration-phases)
4. [Code Transformation Patterns](#code-transformation-patterns)
5. [T100 Message Creation](#t100-message-creation)
6. [Testing Strategy](#testing-strategy)
7. [Rollback Plan](#rollback-plan)
8. [FAQ](#faq)

---

## Migration Overview

### What's Changing?

**Before (Current):**
```abap
" Hard-coded Czech text
MESSAGE s000(zfi_alloc) WITH |Zpracováno { lv_count } položek| INTO lv_dummy.
mo_log->log_sy_msg( ).
```

**After (New Architecture):**
```abap
" T100 message with placeholders
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '042'  " T100: "Zpracováno &1 položek"
  iv_message_v1     = lv_count
  iv_severity       = 'S'
).
```

### Why Migrate?

- ✅ **Multi-language support** via SE63 translation
- ✅ **Central text governance** (no code changes for text updates)
- ✅ **Cleaner code** (no string templates)
- ✅ **Better observability** (structured logging with severity)
- ✅ **Unified audit trail** (process instance UUID links all logs)

### Scope

**Files to Migrate:**
- `zcl_fi_alloc_step_init.clas.abap` (~10-15 hard-coded texts)
- `zcl_fi_alloc_step_phase1.clas.abap` (~15-20 hard-coded texts)
- `zcl_fi_alloc_step_phase2.clas.abap` (~30-40 hard-coded texts, most complex)
- `zcl_fi_alloc_step_phase3.clas.abap` (~15-20 hard-coded texts)
- `zcl_fi_alloc_step_corr_bche.clas.abap` (~10-15 hard-coded texts)

**Total:** ~80-110 hard-coded string templates → T100 messages

---

## Pre-Migration Checklist

### ✅ Prerequisites

- [ ] **ADR-008 approved** (Architecture Decision Record)
- [ ] **Implementation Guide read** (understand new API)
- [ ] **Development system access** (ADT or SE80)
- [ ] **Message class authority** (SE91, SE63)
- [ ] **Test environment available** (for regression testing)

### ✅ Infrastructure Ready

- [ ] **ZIF_FI_PROCESS_LOGGER interface created**
- [ ] **ZCL_FI_PROCESS_LOGGER class implemented**
- [ ] **ZFI_PROCESS message class expanded** (framework messages 001-099, 996-999)
- [ ] **ZFI_ALLOC message class expanded** (business messages 001-599)
- [ ] **ZCL_FI_PROCESS_STEP updated** (mo_log attribute added)
- [ ] **ZCL_FI_PROCESS_INSTANCE updated** (logger creation in create()/load())

### ✅ Backup & Safety

- [ ] **Code backed up** (Git commit before migration)
- [ ] **Test data prepared** (representative allocation scenarios)
- [ ] **Rollback plan documented** (see Rollback Plan section)

---

## Migration Phases

### Phase 1: Inventory & Planning (2-4 hours)

**Goal:** Identify all hard-coded texts and plan T100 message structure

**Steps:**

1. **Search for hard-coded texts:**
   ```abap
   " ADT: Search in Project
   " Pattern: |.*{.*}.*|
   " Or: MESSAGE s000(zfi_alloc) WITH
   ```

2. **Create inventory spreadsheet:**
   
   | File | Line | Current Code | Message Number | T100 Text | Variables | Severity |
   |------|------|--------------|----------------|-----------|-----------|----------|
   | zcl_fi_alloc_step_phase2 | 142 | `\|Zpracováno { lv_count } položek\|` | 202 | Zpracováno &1 položek | lv_count | S |
   | zcl_fi_alloc_step_phase2 | 156 | `\|Chyba validace: { lv_error }\|` | 500 | Chyba validace: &1 | lv_error | E |

3. **Assign message numbers:**
   - INIT: 001-099
   - PHASE1: 100-199
   - PHASE2: 200-299
   - PHASE3: 300-399
   - CORR_BCHE: 400-499
   - Errors: 500-599

4. **Review with team** (ensure numbering makes sense)

**Deliverable:** Inventory spreadsheet with all texts mapped to message numbers

---

### Phase 2: Create T100 Messages (4-8 hours)

**Goal:** Create all T100 messages in ZFI_ALLOC

**Steps:**

1. **Open message class:**
   - ADT: Open `src/zfi_alloc.msag.xml`
   - SE91: Transaction, enter ZFI_ALLOC

2. **Create messages from inventory:**
   ```
   For each row in inventory:
   - Add message with assigned number
   - Copy Czech text from "T100 Text" column
   - Replace variables with &1, &2, &3, &4
   - Save
   ```

3. **Handle long texts (>73 chars):**
   ```
   Original: "Chyba při validaci: Společnost 1000 nemá definované středisko pro období 01"
   
   Split into:
   500: "Chyba validace: Společnost &1, období &2"
   501: "Důvod: Chybí definice nákladových středisek"
   ```

4. **Activate message class**

**Deliverable:** ZFI_ALLOC with ~80-110 new messages activated

---

### Phase 3: Migrate Step Code (8-16 hours)

**Goal:** Replace hard-coded texts with `mo_log->message()` calls

**Steps:**

1. **Work step-by-step** (one file at a time)

2. **For each hard-coded text:**
   - Find in inventory spreadsheet
   - Look up message number
   - Transform code (see Code Transformation Patterns)
   - Test transformation (compile, run)

3. **Commit frequently** (per step file)

**Deliverable:** All 5 step files migrated, compilable, committed

---

### Phase 4: Testing (4-8 hours)

**Goal:** Verify migrated code produces same business results

**Steps:**

1. **Unit tests:**
   - Mock logger (capture messages)
   - Verify message numbers logged
   - Assert business logic unchanged

2. **Integration tests:**
   - Run allocation with test data
   - Verify SLG1 logs created
   - Compare results with pre-migration baseline

3. **Regression tests:**
   - Run full allocation suite
   - Verify 18 completed stories still pass
   - Check exception handling works

**Deliverable:** All tests passing, logs visible in SLG1

---

### Phase 5: Translation (2-4 hours, optional)

**Goal:** Translate Czech messages to English (and other languages)

**Steps:**

1. **Execute SE63:**
   - Select "Short Texts" → "ABAP Objects" → "Messages"
   - Enter message class: ZFI_ALLOC
   - Enter target language: EN

2. **Translate all messages:**
   - Original (CS): "Zpracováno &1 položek"
   - English (EN): "Processed &1 items"

3. **Test with English user:**
   - Log in as user with LANGU = EN
   - Run allocation
   - Verify SLG1 shows English texts

**Deliverable:** ZFI_ALLOC translated to English (Czech + English available)

---

## Code Transformation Patterns

### Pattern 1: Simple String Template → T100

**Before:**
```abap
MESSAGE s000(zfi_alloc) WITH |Zpracováno { lv_count } položek| INTO lv_dummy.
mo_log->log_sy_msg( ).
```

**After:**
```abap
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '202'  " T100: "Zpracováno &1 položek"
  iv_message_v1     = lv_count
  iv_severity       = 'S'
).
```

**T100 Message:**
```
202: Zpracováno &1 položek
```

---

### Pattern 2: Multiple Variables → T100 with msgv1-4

**Before:**
```abap
MESSAGE s000(zfi_alloc) WITH |Společnost { lv_company }, období { lv_period }, položek { lv_count }| INTO lv_dummy.
mo_log->log_sy_msg( ).
```

**After:**
```abap
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '201'
  iv_message_v1     = lv_company
  iv_message_v2     = lv_period
  iv_message_v3     = lv_count
  iv_severity       = 'I'
).
```

**T100 Message:**
```
201: Společnost &1, období &2, položek &3
```

---

### Pattern 3: Long Text → Split into Multiple Messages

**Before:**
```abap
MESSAGE s000(zfi_alloc) WITH |Chyba při validaci alokace: Společnost { lv_company } nemá definované středisko pro období { lv_period }| INTO lv_dummy.
mo_log->log_sy_msg( ).
```

**After:**
```abap
" First message: Context
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '500'
  iv_message_v1     = lv_company
  iv_message_v2     = lv_period
  iv_severity       = 'E'
).

" Second message: Details
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '501'
  iv_severity       = 'E'
).
```

**T100 Messages:**
```
500: Chyba validace: Společnost &1, období &2
501: Důvod: Chybí definice nákladových středisek
```

---

### Pattern 4: Error Message → Exception (No Logging Needed)

**Before:**
```abap
IF lv_validation_failed = abap_true.
  MESSAGE e000(zfi_alloc) WITH |Validace selhala pro { lv_company }| INTO lv_dummy.
  mo_log->log_sy_msg( ).
  RAISE EXCEPTION TYPE zcx_fi_process_error ...
ENDIF.
```

**After:**
```abap
IF lv_validation_failed = abap_true.
  " Just raise exception - framework logs automatically!
  RAISE EXCEPTION TYPE zcx_fi_process_error
    EXPORTING
      textid = zcx_fi_process_error=>validation_failed
      value  = lv_company.
ENDIF.
```

**No T100 message needed** - exception text comes from `ZCX_FI_PROCESS_ERROR`

---

### Pattern 5: Conditional Logging → Same Pattern

**Before:**
```abap
IF lv_debug_mode = abap_true.
  MESSAGE s000(zfi_alloc) WITH |Debug: Zpracováno { lv_count }| INTO lv_dummy.
  mo_log->log_sy_msg( ).
ENDIF.
```

**After:**
```abap
IF lv_debug_mode = abap_true.
  mo_log->message(
    iv_message_class  = 'ZFI_ALLOC'
    iv_message_number = '999'
    iv_message_v1     = lv_count
    iv_severity       = 'I'
  ).
ENDIF.
```

**T100 Message:**
```
999: Debug: Zpracováno &1
```

---

### Pattern 6: Loop Logging → Summary Instead

**Before:**
```abap
LOOP AT lt_items INTO DATA(ls_item).
  MESSAGE s000(zfi_alloc) WITH |Položka { sy-tabix } zpracována| INTO lv_dummy.
  mo_log->log_sy_msg( ).  " ← 10,000 messages!
ENDLOOP.
```

**After:**
```abap
LOOP AT lt_items INTO DATA(ls_item).
  " ... business logic (no logging) ...
ENDLOOP.

" Log summary after loop
mo_log->message(
  iv_message_class  = 'ZFI_ALLOC'
  iv_message_number = '202'
  iv_message_v1     = lines( lt_items )
  iv_severity       = 'S'
).
```

**T100 Message:**
```
202: Zpracováno &1 položek
```

---

## T100 Message Creation

### Quick Reference: SE91 Workflow

1. **Execute SE91**
2. Enter message class: `ZFI_ALLOC`
3. Click "Change"
4. Find next available number in range (e.g., 200-299 for PHASE2)
5. Click "New Entries" or insert row
6. Enter:
   - Message number: `202`
   - Message text: `Zpracováno &1 položek`
7. Save and activate

### Quick Reference: ADT Workflow

1. Navigate to `src/zfi_alloc.msag.xml`
2. Right-click → "Open With" → "ABAP Message Class Editor"
3. Click "New Message" button
4. Enter:
   - Number: `202`
   - Text: `Zpracováno &1 položek`
5. Save (Ctrl+S) and activate (Ctrl+F3)

### Placeholder Guidelines

| Placeholder | Usage | Example |
|-------------|-------|---------|
| `&1` | First variable | `Společnost &1` → "Společnost 1000" |
| `&2` | Second variable | `Období &2` → "Období 01" |
| `&3` | Third variable | `Položek &3` → "Položek 45" |
| `&4` | Fourth variable | `Status &4` → "Status COMPLETED" |
| `&` | Generic (classic) | `Zpracováno &` → "Zpracováno 45" |

**Prefer numbered placeholders (`&1`, `&2`) for clarity!**

---

## Testing Strategy

### Unit Testing

**Create mock logger to verify message calls:**

```abap
CLASS ltc_step_migration DEFINITION FOR TESTING.
  PRIVATE SECTION.
    DATA mo_step TYPE REF TO zcl_fi_alloc_step_phase2.
    DATA mo_logger_mock TYPE REF TO lcl_logger_mock.
    
    METHODS test_message_logged FOR TESTING.
ENDCLASS.

CLASS ltc_step_migration IMPLEMENTATION.
  METHOD test_message_logged.
    " Setup
    mo_logger_mock = NEW lcl_logger_mock( ).
    mo_step = NEW zcl_fi_alloc_step_phase2( ).
    mo_step->initialize_logger( mo_logger_mock ).
    
    " Execute
    mo_step->execute( ).
    
    " Assert message 202 was logged
    READ TABLE mo_logger_mock->mt_messages WITH KEY message_number = '202' TRANSPORTING NO FIELDS.
    cl_abap_unit_assert=>assert_subrc(
      act = sy-subrc
      exp = 0
      msg = 'Message 202 not logged'
    ).
  ENDMETHOD.
ENDCLASS.
```

---

### Integration Testing

**Run allocation with BAL logging:**

```abap
CLASS ltc_integration DEFINITION FOR TESTING.
  PRIVATE SECTION.
    DATA mo_instance TYPE REF TO zcl_fi_process_instance.
    DATA mv_instance_uuid TYPE sysuuid_x16.
    
    METHODS test_allocation_with_logging FOR TESTING.
    METHODS verify_log_in_bal FOR TESTING.
ENDCLASS.

CLASS ltc_integration IMPLEMENTATION.
  METHOD test_allocation_with_logging.
    " Create instance
    mo_instance = zcl_fi_process_instance=>create(
      iv_process_type = 'ALLOCATION'
      is_context      = VALUE #( company_code = '1000' fiscal_period = '01' )
    ).
    mv_instance_uuid = mo_instance->get_uuid( ).
    
    " Execute allocation
    mo_instance->execute( ).
    
    " Verify completed successfully
    cl_abap_unit_assert=>assert_equals(
      act = mo_instance->get_status( )
      exp = zcl_fi_process_instance=>gc_status-completed
    ).
  ENDMETHOD.
  
  METHOD verify_log_in_bal.
    " Query BAL by external number = instance UUID
    SELECT SINGLE * FROM balhdr
      INTO @DATA(ls_log)
      WHERE extnumber = @mv_instance_uuid.
    
    cl_abap_unit_assert=>assert_subrc(
      act = sy-subrc
      exp = 0
      msg = 'BAL log not found'
    ).
    
    " Verify messages exist
    SELECT COUNT(*) FROM baldat
      INTO @DATA(lv_msg_count)
      WHERE log_handle = @ls_log-log_handle.
    
    cl_abap_unit_assert=>assert_differs(
      act = lv_msg_count
      exp = 0
      msg = 'No messages in BAL log'
    ).
  ENDMETHOD.
ENDCLASS.
```

---

### Regression Testing

**Verify existing scenarios still work:**

1. **Baseline (before migration):**
   - Run allocation for company 1000, period 01
   - Capture results: item count, document numbers, amounts
   - Capture SLG1 log: message count, severity distribution

2. **After migration:**
   - Run same allocation
   - Compare results: item count, documents, amounts (MUST match)
   - Compare SLG1 log: message count should be similar, severity appropriate

3. **Test matrix:**
   
   | Scenario | Input | Expected Output | Status |
   |----------|-------|-----------------|--------|
   | Happy path | Company 1000, Period 01 | 45 items, 3 documents | ✅ |
   | Validation error | Invalid company | Exception raised | ✅ |
   | Empty data | No cost centers | Warning logged | ✅ |
   | Multi-period | Periods 01-12 | 540 items, 36 docs | ✅ |

---

## Rollback Plan

### If Migration Fails

**Scenario:** Migrated code has bugs, production deadline approaching

**Rollback Steps:**

1. **Git revert:**
   ```bash
   cd ~/DEV/cz.imcg.fast.ovysledovka
   git log --oneline  # Find pre-migration commit
   git revert <commit-hash>
   git push origin master
   ```

2. **Transport rollback:**
   - Create transport with pre-migration code
   - Import to target system
   - Verify old code active

3. **Clean up T100 messages (optional):**
   - SE91 → ZFI_ALLOC
   - Delete unused messages (or mark inactive)
   - Not required (messages without code references are harmless)

**Recovery Time:** ~30 minutes

---

### Partial Migration Strategy

**If full migration too risky:**

1. **Migrate one step at a time:**
   - Week 1: Migrate `zcl_fi_alloc_step_init` only
   - Week 2: Migrate `zcl_fi_alloc_step_phase1` only
   - Week 3: Migrate `zcl_fi_alloc_step_phase2` only
   - Week 4: Migrate remaining steps

2. **Benefits:**
   - Smaller changes per week
   - Easier to test and validate
   - Lower risk (can rollback single step)

3. **Trade-offs:**
   - Longer total timeline (4 weeks vs. 1 week)
   - Mixed logging patterns during transition

---

## FAQ

### Q: Do I need to migrate ALL hard-coded texts at once?

**A:** No. You can migrate step-by-step (one file per sprint). Old logging code continues to work alongside new code.

---

### Q: What if I can't fit a message in 73 characters?

**A:** Split into multiple messages:
```abap
mo_log->message( iv_message_number = '500' ... ).  " Context
mo_log->message( iv_message_number = '501' ... ).  " Details
```

---

### Q: Can I use more than 4 variables?

**A:** No, T100 messages support only 4 placeholders (`&1` to `&4`). If you need more, split the message or combine variables:
```abap
" Option 1: Split message
mo_log->message( iv_message_v1 = lv_var1 iv_message_v2 = lv_var2 ... ).
mo_log->message( iv_message_v1 = lv_var5 iv_message_v2 = lv_var6 ... ).

" Option 2: Combine variables
DATA(lv_combined) = |{ lv_var1 }/{ lv_var2 }|.
mo_log->message( iv_message_v1 = lv_combined ... ).
```

---

### Q: Should I delete old `MESSAGE ... INTO lv_dummy` code?

**A:** Yes, replace with `mo_log->message()`. Old code should be removed to avoid confusion.

---

### Q: How do I test message text rendering?

**A:** Use SE91 "Test" function:
1. SE91 → ZFI_ALLOC → Display
2. Select message → Click "Test"
3. Enter variable values
4. View rendered text

---

### Q: What if framework doesn't initialize logger?

**A:** Add defensive check:
```abap
METHOD execute.
  ASSERT mo_log IS BOUND.  " Fails fast if not initialized
  " ... business logic ...
ENDMETHOD.
```

This ensures you get a clear error message instead of random dumps.

---

### Q: Can I keep some hard-coded texts?

**A:** Technically yes (for internal debugging), but NOT RECOMMENDED. Hard-coded texts defeat the purpose of migration (no translation, no central governance).

**Exception:** Temporary debug messages during development can use hard-coded text, but must be removed before production.

---

### Q: How do I migrate exception messages?

**A:** DON'T log exceptions explicitly - framework logs them automatically:

```abap
" ❌ OLD (unnecessary):
TRY.
  " ...
CATCH zcx_fi_process_error INTO DATA(lx_error).
  mo_log->log_exception( lx_error ).  " ← Don't do this
  RAISE EXCEPTION lx_error.
ENDTRY.

" ✅ NEW (framework handles):
TRY.
  " ...
CATCH zcx_fi_process_error.
  RAISE EXCEPTION.  " ← Framework logs automatically
ENDTRY.
```

---

### Q: What severity should I use?

| Severity | When to Use | Example |
|----------|-------------|---------|
| `I` (Info) | Progress updates, general information | "Processing company 1000" |
| `S` (Success) | Successful operations | "Created 45 allocation items" |
| `W` (Warning) | Non-critical issues that don't stop processing | "Cost center inactive but used" |
| `E` (Error) | Business errors (with exception) | "Validation failed" |
| `A` (Abort) | Critical errors requiring intervention | "System configuration missing" |
| `X` (Exit) | System errors causing dump | Rarely used manually |

**Default to `I` if unsure.**

---

### Q: How long does SE63 translation take?

**A:** ~2-4 hours for 100 messages (Czech → English). More languages add ~1-2 hours each.

**Tip:** Translate in batches (10-20 messages at a time) to maintain consistency.

---

### Q: Can I automate translation?

**A:** Not recommended. Machine translation often misses business context. Manual SE63 translation ensures quality.

**Alternative:** Use translation memory tools if translating >5 languages.

---

## Migration Checklist

Use this checklist to track migration progress:

### Pre-Migration
- [ ] ADR-008 approved
- [ ] Infrastructure implemented (interface, class, message classes)
- [ ] Code backed up (Git commit)
- [ ] Test data prepared

### Phase 1: Planning
- [ ] Inventory spreadsheet created
- [ ] All hard-coded texts identified (~100)
- [ ] Message numbers assigned (001-599)
- [ ] Team review completed

### Phase 2: T100 Messages
- [ ] ZFI_ALLOC messages created (001-099: INIT)
- [ ] ZFI_ALLOC messages created (100-199: PHASE1)
- [ ] ZFI_ALLOC messages created (200-299: PHASE2)
- [ ] ZFI_ALLOC messages created (300-399: PHASE3)
- [ ] ZFI_ALLOC messages created (400-499: CORR_BCHE)
- [ ] ZFI_ALLOC messages created (500-599: Errors)
- [ ] Message class activated

### Phase 3: Code Migration
- [ ] zcl_fi_alloc_step_init migrated
- [ ] zcl_fi_alloc_step_phase1 migrated
- [ ] zcl_fi_alloc_step_phase2 migrated
- [ ] zcl_fi_alloc_step_phase3 migrated
- [ ] zcl_fi_alloc_step_corr_bche migrated
- [ ] All files compile without errors
- [ ] Git commits created (per step)

### Phase 4: Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Regression tests pass (18 stories)
- [ ] SLG1 logs verified
- [ ] No hard-coded texts remaining

### Phase 5: Translation (Optional)
- [ ] SE63 translation to English
- [ ] English user testing
- [ ] Additional languages (if needed)

### Production Readiness
- [ ] Code review completed
- [ ] Transport created
- [ ] Production deployment plan
- [ ] Rollback plan documented
- [ ] Team trained on new logging API

---

## Support

**Questions during migration?**

- **Technical:** Zdenek Smolik (smolik@imcg.cz)
- **Documentation:** [Implementation Guide](./process-logger-implementation-guide.md)
- **Architecture:** [ADR-008](../architecture/ADR-008-process-logger-architecture.md)

---

**Version History:**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-11 | Zdenek Smolik | Initial version |
