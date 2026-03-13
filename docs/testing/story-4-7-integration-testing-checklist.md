# Story 4-7: Integration Testing & Regression Checklist

**Sprint**: 4 (Process Logger Architecture)  
**Story**: 4-7 (Integration Testing & Regression)  
**Target Repositories**: `cz.imcg.fast.planner`, `cz.imcg.fast.ovysledovka`  
**Created**: 2026-03-12  
**Updated**: 2026-03-13  
**Status**: Phase 1 Complete - Phase 2-6 In Progress

---

## Purpose

Verify that the T100 message migration in Story 4-5 produces:
1. **Syntactically valid ABAP code** (compilation succeeds)
2. **Functionally equivalent behavior** (business results unchanged)
3. **Proper logging** (messages appear in SLG1 with correct texts)
4. **Acceptable performance** (<5% overhead for BAL writes)

---

## Prerequisites

- ✅ Story 4-5 complete (all 5 step classes migrated)
- ✅ All commits pushed to both repositories
- ✅ SAP system available (DEV or QAS environment)
- ✅ Test data available (baseline allocation run from pre-migration)

---

## Phase 1: Syntax Verification ✅ COMPLETE (2026-03-13)

**Result**: All syntax errors fixed, code deployed to SAP successfully.

**Issues Found & Resolved**:
1. ❌ **73 syntax errors** - Parameter name mismatch (iv_param vs iv_message_v1-v4) → ✅ Fixed (commits 98cee55, 6b60259, 350f646)
2. ❌ **Interface missing RAISING clause** → ✅ Fixed (commit 0addac0)
3. ❌ **bgRFC deadlock** (MESSAGE_TYPE_X dump) → ✅ Fixed (commit 2d87681 - removed mo_log->save() calls)
4. ❌ **Error message bug** (empty KeyID) → ✅ Fixed (commit c818f92)

**Runtime Issue Discovered** (Not Sprint 4 Related):
- ⚠️ CDS view `ZFI_I_ALLOC_BASE2` missing `ZZSalesGroup`/`ZZSalesOffice` fields
- Causes AMDP failure in Phase 2 substep execution
- Requires CDS view update (deferred - outside Sprint 4 scope)

---

## Phase 1 (Original): Syntax Verification (30 minutes)

### 1.1 Framework Repository Syntax Check

**Repository**: `cz.imcg.fast.planner`

```abap
" Execute in SE38 or SE80
" Program: Check syntax for all modified classes

REPORT z_check_syntax_planner.

DATA: lt_messages TYPE TABLE OF string,
      lv_errors   TYPE i,
      lv_warnings TYPE i.

" Classes to check (from Stories 4-1 to 4-4)
DATA(lt_classes) = VALUE string_table(
  ( 'ZIF_FI_PROCESS_LOGGER' )
  ( 'ZCL_FI_PROCESS_LOGGER' )
  ( 'ZCL_FI_PROCESS_STEP' )
).

LOOP AT lt_classes INTO DATA(lv_class).
  WRITE: / '═══════════════════════════════════════════'.
  WRITE: / 'Checking:', lv_class.
  
  " Use syntax check API
  SYNTAX-CHECK FOR PROGRAM lv_class MESSAGE lv_msg LINE lv_line.
  IF sy-subrc <> 0.
    WRITE: / '❌ ERROR:', lv_msg, 'at line', lv_line.
    ADD 1 TO lv_errors.
  ELSE.
    WRITE: / '✅ OK - No syntax errors'.
  ENDIF.
ENDLOOP.

WRITE: / '═══════════════════════════════════════════'.
WRITE: / 'Summary:', lv_errors, 'errors,', lv_warnings, 'warnings'.
IF lv_errors = 0.
  WRITE: / '✅ All framework classes compile successfully'.
ELSE.
  WRITE: / '❌ FAILED - Syntax errors found'.
ENDIF.
```

**Expected Result**: All classes compile with 0 errors

**Action if Failed**: Review commit diffs, check for missing imports or typos

---

### 1.2 Business Logic Repository Syntax Check

**Repository**: `cz.imcg.fast.ovysledovka`

```abap
" Execute in SE38 or SE80
" Program: Check syntax for all migrated step classes

REPORT z_check_syntax_ovysledovka.

DATA: lv_errors   TYPE i,
      lv_warnings TYPE i.

" Step classes to check (from Story 4-5)
DATA(lt_classes) = VALUE string_table(
  ( 'ZCL_FI_ALLOC_STEP_INIT' )
  ( 'ZCL_FI_ALLOC_STEP_PHASE1' )
  ( 'ZCL_FI_ALLOC_STEP_PHASE2' )
  ( 'ZCL_FI_ALLOC_STEP_PHASE3' )
  ( 'ZCL_FI_ALLOC_STEP_CORR_BCHE' )
).

LOOP AT lt_classes INTO DATA(lv_class).
  WRITE: / '═══════════════════════════════════════════'.
  WRITE: / 'Checking:', lv_class.
  
  SYNTAX-CHECK FOR PROGRAM lv_class MESSAGE DATA(lv_msg) LINE DATA(lv_line).
  IF sy-subrc <> 0.
    WRITE: / '❌ ERROR:', lv_msg, 'at line', lv_line.
    ADD 1 TO lv_errors.
  ELSE.
    WRITE: / '✅ OK - No syntax errors'.
  ENDIF.
ENDLOOP.

WRITE: / '═══════════════════════════════════════════'.
WRITE: / 'Summary:', lv_errors, 'errors'.
IF lv_errors = 0.
  WRITE: / '✅ All step classes compile successfully'.
ELSE.
  WRITE: / '❌ FAILED - Syntax errors found'.
ENDIF.
```

**Expected Result**: All 5 step classes compile with 0 errors

**Action if Failed**: Review migration commits, check message class activation

---

### 1.3 Message Class Activation

**Transaction**: SE91

1. Open message class `ZFI_ALLOC`
2. Verify message count: **118 messages** (001-012, 013-020, 100-124, 200-226, 300-324, 400-411, 500-507)
3. Check random samples:
   - Message 200: "Allocation ID &1 for period &2/&3/&4 does not exist"
   - Message 310: "Deleting allocation items"
   - Message 403: "CORR_BCHE completed. &1 headers processed"
4. Verify both languages exist (EN + CS)
5. Click **Activate** (if not already active)

**Expected Result**: All 118 messages active, bilingual

---

## Phase 2: Unit Testing (2-3 hours)

### 2.1 Logger Unit Tests (from Story 4-1)

**Transaction**: SE80 → Test → Unit Tests

**Class**: `ZCL_FI_PROCESS_LOGGER`

```bash
# Run all unit tests
# Expected: All tests pass (green)
```

**Tests to verify**:
- ✅ `create_with_process_instance` - Logger creation succeeds
- ✅ `message_with_valid_params` - T100 message logged correctly
- ✅ `message_with_invalid_class` - Invalid message class raises exception
- ✅ `save_creates_bal_entries` - BALHDR/BALDAT tables updated

**Expected Result**: 100% pass rate

---

### 2.2 Process Instance Unit Tests (from Story 4-3)

**Class**: `ZCL_FI_PROCESS_INSTANCE`

**Tests to verify**:
- ✅ `init_creates_logger` - Logger bound after initialization
- ✅ `logger_available_in_step` - Step classes can access mo_log
- ✅ `logger_lifecycle_managed` - Logger saved on completion

**Expected Result**: 100% pass rate

---

### 2.3 Step Class Message Coverage (manual verification)

For each step class, verify message integration:

**Test Script Template** (adapt for each step):

```abap
" Example: Test INIT step message 013
REPORT z_test_init_step_msg_013.

DATA: lo_step     TYPE REF TO zcl_fi_alloc_step_init,
      lo_instance TYPE REF TO zcl_fi_process_instance,
      ls_context  TYPE zif_fi_process_step=>ty_step_context,
      ls_result   TYPE zif_fi_process_step=>ty_step_result.

" Setup
CREATE OBJECT lo_instance.
" ... (populate instance with test data)

ls_context-io_process_instance = lo_instance.

" Execute
CREATE OBJECT lo_step.
lo_step->zif_fi_process_step~init( ls_context ).
ls_result = lo_step->zif_fi_process_step~validate( ls_context ).

" Verify
ASSERT ls_result-success = abap_true.
ASSERT ls_result-message CS 'validated successfully'.  " Message 020

" Check log
DATA(lt_log_messages) = lo_instance->get_logger( )->get_messages( ).
ASSERT lines( lt_log_messages ) > 0.

WRITE: / '✅ Message 020 logged correctly'.
```

**Expected Result**: Each step's validation/execute methods log messages

---

## Phase 3: Integration Testing (2-3 hours)

### 3.1 Happy Path Test

**Scenario**: Full allocation run (3 phases + correction)

**Setup**:
- Company: 1000
- Fiscal Year: 2024
- Fiscal Period: 001
- Allocation ID: TEST001

**Execution**:
```abap
" Transaction: SE38
" Program: ZFI_ALLOC_RUN (or custom test program)

PARAMETERS: p_bukrs TYPE bukrs DEFAULT '1000',
            p_gjahr TYPE gjahr DEFAULT '2024',
            p_poper TYPE poper DEFAULT '001',
            p_allid TYPE zfi_alloc_id DEFAULT 'TEST001'.

START-OF-SELECTION.
  " Execute full allocation
  zcl_fi_allocations=>run_allocation(
    iv_company_code  = p_bukrs
    iv_fiscal_year   = p_gjahr
    iv_fiscal_period = p_poper
    iv_allocation_id = p_allid
  ).
```

**Verification Steps**:

1. **Check Process Status**:
   ```sql
   SELECT * FROM zfi_process_inst 
     WHERE process_id = 'TEST001'
   ```
   - Expected: `status = 'COMPLETED'`

2. **Check SLG1 Logs**:
   - Transaction: SLG1
   - Object: `ZFI_ALLOC`
   - External Number: `1000/2024/001/TEST001`
   - Expected: 4 log handles (INIT, PHASE1, PHASE2, PHASE3, CORR_BCHE)
   - Expected: Messages visible in English or Czech (based on user language)

3. **Check Business Results**:
   ```sql
   -- Verify records created
   SELECT COUNT(*) FROM zfi_alloc_state 
     WHERE allocation_id = 'TEST001'
   
   SELECT COUNT(*) FROM zfi_alloc_bche
     WHERE allocation_id = 'TEST001'
   
   SELECT COUNT(*) FROM zfi_alloc_bcitm
     WHERE allocation_id = 'TEST001'
   
   SELECT COUNT(*) FROM zfi_alloc_items
     WHERE allocation_id = 'TEST001'
   ```
   - Compare counts with baseline (pre-migration run)
   - Expected: **Exact match** (business logic unchanged)

4. **Check Message Content** (SLG1):
   - Open log for PHASE1
   - Expected messages visible:
     - Message 100: "Processing allocations - Phase I"
     - Message 104: "Company code: 1000"
     - Message 107: "Allocation ID: TEST001"
   - Verify parameters populated correctly (&1, &2, etc.)

**Expected Result**: ✅ All checks pass, business results match baseline

---

### 3.2 Error Scenario Test

**Scenario**: Validation failure triggers error messages

**Setup**:
- Company: (empty)
- Fiscal Year: 2024
- Fiscal Period: 001
- Allocation ID: TEST002

**Execution**:
```abap
" Force validation error by omitting company code
TRY.
    zcl_fi_allocations=>run_allocation(
      iv_company_code  = ''  " Invalid
      iv_fiscal_year   = '2024'
      iv_fiscal_period = '001'
      iv_allocation_id = 'TEST002'
    ).
  CATCH zcx_fi_process_error INTO DATA(lx_error).
    WRITE: / '❌ Expected error:', lx_error->get_text( ).
ENDTRY.
```

**Verification**:
1. Exception raised: `zcx_fi_process_error`
2. Message logged in SLG1: "INIT: COMPANY_CODE is missing" (Message 016)
3. Process status: `FAILED` or `NOT_STARTED`

**Expected Result**: ✅ Error handled gracefully, logged correctly

---

### 3.3 bgRFC Parallel Execution Test (PHASE2)

**Scenario**: PHASE2 step executes substeps in parallel via bgRFC

**Setup**:
- Company: 1000
- Fiscal Year: 2024
- Fiscal Period: 001
- Allocation ID: TEST003
- PHASE1 must complete first (to populate zfi_alloc_bche with keys)

**Execution**:
```abap
" Run PHASE2 only (after PHASE1 completes)
" Framework will spawn bgRFC units for each key_id
```

**Verification**:
1. **Check Substep Logs** (SLG1):
   - Object: `ZFI_ALLOC`
   - Subobject: `PHASE2_SUB`
   - Expected: Multiple log handles (one per key_id)
   - Each log should contain message 226: "PHASE2 substep completed successfully for key_id &1"

2. **Check bgRFC Scheduler**:
   - Transaction: SBGRFCMON
   - Verify: All PHASE2 substeps completed successfully
   - No "hanging" or failed units

3. **Check Callback Execution**:
   - SLG1 → PHASE2 main log
   - Expected: Message 009 ("Process completed successfully") from `on_success` method
   - Verify timestamp shows callbacks executed after all substeps

**Expected Result**: ✅ Parallel execution succeeds, logs created per substep

---

## Phase 4: Regression Testing (2-3 hours)

### 4.1 Run Existing Allocation Tests

**Prerequisite**: Test suite from Sprints 1-3 exists

**Execution**:
```bash
# If automated test suite exists
# Run all regression tests

# Example: ABAP Unit test classes
# Transaction: SE80 → Mass Test
```

**Tests to run** (from previous sprints):
- Sprint 1: Process framework basic execution
- Sprint 2: State persistence and recovery
- Sprint 3: bgRFC substep orchestration

**Expected Result**: ✅ All regression tests pass (no failures)

---

### 4.2 Compare Business Results

**Scenario**: Run identical allocation twice (before/after migration)

**Setup**:
1. **Baseline Run** (pre-migration):
   - Export data: `zfi_alloc_state`, `zfi_alloc_bche`, `zfi_alloc_bcitm`, `zfi_alloc_items`
   - Save counts and sample records

2. **Post-Migration Run**:
   - Delete data from tables
   - Re-run same allocation with same parameters
   - Export data again

**Comparison Script**:
```sql
-- Compare record counts
SELECT 'zfi_alloc_state' AS table_name,
       COUNT(*) AS post_count,
       5 AS baseline_count  -- Example
  FROM zfi_alloc_state
 WHERE allocation_id = 'TEST_BASELINE'

UNION ALL

SELECT 'zfi_alloc_bche',
       COUNT(*),
       120  -- Example baseline
  FROM zfi_alloc_bche
 WHERE allocation_id = 'TEST_BASELINE'

-- Add other tables...
```

**Expected Result**: ✅ Exact match (counts, amounts, keys)

---

### 4.3 Exception Handling Regression

**Scenario**: Verify exceptions still propagate correctly

**Test Cases**:
1. **Invalid Allocation ID** (Message 200):
   - Expected: Exception raised, logged to SLG1
2. **Locked Allocation** (Message 201):
   - Expected: Exception raised, rollback executed
3. **Database Insert Failure** (Message e006):
   - Expected: Error logged, process marked failed

**Expected Result**: ✅ All error scenarios handled as before

---

## Phase 5: Performance Testing (1 hour)

### 5.1 Execution Time Comparison

**Setup**:
- Run large allocation (1000+ items)
- Measure before/after migration

**Execution**:
```abap
DATA: lv_start TYPE timestampl,
      lv_end   TYPE timestampl,
      lv_diff  TYPE tzntstmpl.

GET TIME STAMP FIELD lv_start.

" Run allocation
zcl_fi_allocations=>run_allocation( ... ).

GET TIME STAMP FIELD lv_end.

lv_diff = lv_end - lv_start.
WRITE: / 'Execution time:', lv_diff, 'microseconds'.
```

**Baseline** (pre-migration): X seconds  
**Post-Migration**: Y seconds  
**Overhead**: (Y-X)/X × 100%

**Expected Result**: ✅ Overhead <5%

---

### 5.2 BAL Log Size Analysis

**Query**:
```sql
-- Check log size
SELECT h.lognumber,
       h.extnumber,
       COUNT(*) AS message_count,
       SUM( LENGTH( d.msg_text ) ) AS total_bytes
  FROM balhdr AS h
  JOIN baldat AS d ON d.lognumber = h.lognumber
 WHERE h.object = 'ZFI_ALLOC'
   AND h.extnumber LIKE '1000/2024/%'
 GROUP BY h.lognumber, h.extnumber
 ORDER BY h.lognumber
```

**Expected Result**: 
- INIT: ~10 messages (~500 bytes)
- PHASE1: ~30 messages (~1500 bytes)
- PHASE2: ~20 messages (~1000 bytes)
- PHASE3: ~25 messages (~1200 bytes)
- CORR_BCHE: ~5 messages (~250 bytes)

**Action**: Document for archiving strategy

---

## Phase 6: Manual SLG1 Verification (30 minutes)

### 6.1 Visual Inspection

**Transaction**: SLG1

1. **Select Logs**:
   - Object: `ZFI_ALLOC`
   - Date: Today
   - Execute

2. **Open Each Log**:
   - INIT, PHASE1, PHASE2, PHASE3, CORR_BCHE
   - Verify messages displayed correctly
   - Check message severity (I/S/W/E)
   - Verify parameters substituted (&1, &2, etc.)

3. **Check Language**:
   - Switch user language (SE80 → System → User Profile → Own Data)
   - Re-open SLG1
   - Verify messages appear in new language

**Expected Result**: ✅ Messages readable, bilingual support works

---

### 6.2 Log External Number Verification

**Check**:
- External number format: `{company_code}/{fiscal_year}/{fiscal_period}/{allocation_id}`
- Example: `1000/2024/001/TEST001`

**Verification**:
```sql
SELECT extnumber, 
       object,
       subobject,
       aldate,
       altime
  FROM balhdr
 WHERE object = 'ZFI_ALLOC'
   AND extnumber LIKE '1000/2024/001/%'
 ORDER BY lognumber
```

**Expected Result**: ✅ External numbers correctly formatted, searchable

---

## Summary Checklist

Mark each section complete:

- [x] **Phase 1**: Syntax Verification ✅ COMPLETE (2026-03-13)
  - [x] Framework classes compile (planner repo) - Fixed with commit 0addac0
  - [x] Step classes compile (ovysledovka repo) - Fixed with commits 98cee55, 6b60259, 350f646
  - [x] Message class ZFI_ALLOC activated (97 messages) - Deployed to SAP
  - [x] bgRFC deadlock resolved - Commit 2d87681
  - [x] Error message bug fixed - Commit c818f92

- [ ] **Phase 2**: Unit Testing ⏳ READY TO START
  - [ ] Logger unit tests pass (Story 4-1)
  - [ ] Process instance tests pass (Story 4-3)
  - [ ] Step message coverage verified

- [ ] **Phase 3**: Integration Testing ⚠️ BLOCKED
  - [ ] Happy path test passes - BLOCKED by CDS view issue
  - [ ] Error scenario test passes
  - [ ] bgRFC parallel execution test passes - BLOCKED by CDS view issue

- [ ] **Phase 4**: Regression Testing
  - [ ] Existing test suite passes
  - [ ] Business results match baseline
  - [ ] Exception handling unchanged

- [ ] **Phase 5**: Performance Testing
  - [ ] Execution time within <5% overhead
  - [ ] BAL log size documented

- [ ] **Phase 6**: Manual SLG1 Verification
  - [ ] Messages visible and readable
  - [ ] Bilingual support works
  - [ ] External numbers correctly formatted

---

## Sign-Off

**Tester**: ___________________________  
**Date**: ___________________________  
**Result**: ☐ PASS  ☐ FAIL (see issues below)  

**Issues Found**:
1. _____________________________________________________
2. _____________________________________________________
3. _____________________________________________________

**Resolution**:
- ☐ Issues resolved, re-test scheduled
- ☐ No issues, Story 4-7 complete

---

**Next Step**: Story 4-8 (Documentation & Training)
