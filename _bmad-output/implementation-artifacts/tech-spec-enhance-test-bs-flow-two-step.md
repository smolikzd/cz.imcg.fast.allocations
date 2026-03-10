---
title: 'Enhance TEST_BS_FLOW with Two-Step Business Status Transition Testing'
slug: 'enhance-test-bs-flow-two-step'
created: '2026-03-10'
status: 'completed'
stepsCompleted: [1, 2, 3, 4, 5, 6]
completed: '2026-03-10'
tech_stack: ['ABAP 7.58', 'ZFI_PROCESS Framework', 'Test Data Setup', 'RAP Query Provider']
files_to_modify: ['src/zfi_setup_test_data.prog.abap', 'src/zcl_fiproc_health_chk_query.clas.abap']
code_patterns: ['Process Definition Configuration', 'Business Status Transitions', 'Health Check Assertions', 'Test Automation']
test_patterns: ['Integration Testing', 'Business Status Validation', 'Success Path Testing', 'Failure Path Testing']
---

# Tech-Spec: Enhance TEST_BS_FLOW with Two-Step Business Status Transition Testing

**Created:** 2026-03-10

## Overview

### Problem Statement

The framework's business status feature (`business_status_1` and `business_status_2`) lacks comprehensive test coverage:

**Current Gaps:**
1. `TEST_BS_FLOW` only tests BS1 with a single step (no BS2, no multi-step transitions)
2. `TEST_SERIAL_BASIC` and `TEST_SERIAL_FAIL_CTRL` don't verify business statuses at all
3. No validation that BS1 and BS2 can be set simultaneously by steps
4. No validation that business statuses transition correctly through multi-step processes
5. No validation that final business statuses reflect the last step's completion state
6. No validation of failure path business statuses (fail_bs1, fail_bs2)

**Impact:**
We cannot confidently verify that business status transitions work correctly in real-world scenarios where processes have multiple steps that each update business statuses, or where processes fail and need to show failure-specific business statuses.

### Solution

Enhance business status test coverage across three test processes:

**1. TEST_BS_FLOW - Multi-Step Transition Testing:**
- Modify step 0010 to set both BS1 and BS2 with "STEP1" values
- Add step 0020 to set both BS1 and BS2 with "STEP2" values
- Update health check to verify final BS1 and BS2 reflect step 0020 (last step wins)

**2. TEST_SERIAL_BASIC - Success Path Verification:**
- Add BS1 fields (init_bs1, final_bs1, fail_bs1) to step 0010
- Update health check to verify final_bs1 after successful completion

**3. TEST_SERIAL_FAIL_CTRL - Failure Path Verification:**
- Add BS1 fields (init_bs1, final_bs1, fail_bs1) to step 0010
- Update health check to verify fail_bs1 after controlled failure

This provides comprehensive coverage: success path, failure path, single-step, multi-step, BS1-only, and BS1+BS2.

### Scope

**In Scope:**

**Test Data Setup (`zfi_setup_test_data.prog.abap`):**
- Add BS fields to `TEST_SERIAL_BASIC` step 0010 (init_bs1, final_bs1, fail_bs1)
- Add BS fields to `TEST_SERIAL_FAIL_CTRL` step 0010 (init_bs1, final_bs1, fail_bs1)
- Modify `TEST_BS_FLOW` step 0010 to add BS2 fields (init_bs2, final_bs2, fail_bs2)
- Add new `TEST_BS_FLOW` step 0020 with both BS1 and BS2 configured

**Health Check Tests (`zcl_fiproc_health_chk_query.clas.abap`):**
- Update `check_capability` for `TEST_SERIAL_BASIC` to assert final_bs1='BASIC DONE'
- Update `check_capability` for `TEST_SERIAL_FAIL_CTRL` to assert fail_bs1='FAIL CTRL'
- Update `check_business_status` to assert both business_status_1='STEP2 DONE' and business_status_2='STEP2 COMPLETE' for `TEST_BS_FLOW`

**Target Repository:**
- `planner` (framework repository: `/Users/smolik/DEV/cz.imcg.fast.planner`)

**Out of Scope:**
- Changes to framework business status logic (already implemented)
- Creation of new step classes (reuse existing test classes)
- Changes to process type definitions in `zfi_proc_type` table
- Manual test scripts (health checks run automatically)

## Context for Development

### Codebase Patterns

**Repository Context:**
- This change targets the **planner** repository (`cz.imcg.fast.planner`)
- Two files require modification: test data setup + health check query

**File 1: Test Data Setup Program**
- Location: `src/zfi_setup_test_data.prog.abap`
- Pattern: FORM `create_test_proc_defs` builds internal table `lt_definitions` using VALUE constructor
- Idempotent: Uses `MODIFY zfi_proc_def` (safe to re-run)

**Existing TEST_BS_FLOW Pattern (lines 534-549):**
Currently has one step with only BS1 fields:
```abap
( mandt         = sy-mandt
  process_type  = 'TEST_BS_FLOW'
  step_number   = '0010'
  step_type     = 'TEST'
  description   = 'Test: BS flow - success step'
  class_name    = 'ZCL_FI_STEP_TEST_SUCCESS'
  mandatory     = 'X'
  active        = 'X'
  retry_allowed = ''
  substep_mode  = 'SERIAL'
  init_bs1      = 'IN PROGRESS'
  final_bs1     = 'DONE'
  fail_bs1      = 'FAILED' )
```

**TEST_SERIAL_BASIC Pattern (lines 244-255):**
Currently has NO business status fields:
```abap
( mandt         = sy-mandt
  process_type  = 'TEST_SERIAL_BASIC'
  step_number   = '0010'
  step_type     = 'TEST'
  description   = 'Test: Always succeeds'
  class_name    = 'ZCL_FI_STEP_TEST_SUCCESS'
  mandatory     = 'X'
  active        = 'X'
  retry_allowed = ''
  substep_mode  = 'SERIAL' )
```

**TEST_SERIAL_FAIL_CTRL Pattern (lines 257-268):**
Currently has NO business status fields:
```abap
( mandt         = sy-mandt
  process_type  = 'TEST_SERIAL_FAIL_CTRL'
  step_number   = '0010'
  step_type     = 'TEST'
  description   = 'Test: Controlled failure (success=false)'
  class_name    = 'ZCL_FI_STEP_FAIL_SERIAL'
  mandatory     = 'X'
  active        = 'X'
  retry_allowed = ''
  substep_mode  = 'SERIAL' )
```

**File 2: Health Check Query Provider**
- Location: `src/zcl_fiproc_health_chk_query.clas.abap`
- Pattern: RAP Query Provider (IF_RAP_QUERY_PROVIDER interface)
- METHOD `build_rows`: Orchestrates all health checks
- METHOD `check_capability`: Generic test runner for simple assertions
- METHOD `check_business_status`: Custom test for TEST_BS_FLOW (lines 875-954)

**Health Check Test Pattern:**
```abap
" 1. Create process instance
lo_instance = lo_manager->create_process( iv_process_type = 'TEST_XXX' ).
lv_id = lo_instance->get_instance_id( ).

" 2. Execute (may throw for failure tests)
TRY.
    lo_manager->execute_process( lv_id ).
  CATCH zcx_fi_process_error.
ENDTRY.

" 3. Poll until terminal status
WHILE lv_status NOT IN ('COMPLETED', 'FAILED', 'CANCELLED').
  WAIT UP TO '0.5' SECONDS.
  lo_instance = lo_manager->load_process( lv_id ).
  lv_status = lo_instance->get_status( ).
ENDWHILE.

" 4. Assert on instance fields
DATA(ls_inst) = lo_instance->get_instance( ).
IF ls_inst-business_status_1 <> 'EXPECTED_VALUE'.
  ls_row-status = gc_red.
  ls_row-message = |Expected ..., got { ls_inst-business_status_1 }|.
ELSE.
  ls_row-status = gc_green.
  ls_row-message = |Assertion passed|.
ENDIF.
```

**Table Structure:**
The `ZFI_PROC_DEF` table supports 6 business status fields:
- `INIT_BS1`, `INIT_BS2` - Status when step begins execution
- `FINAL_BS1`, `FINAL_BS2` - Status when step completes successfully
- `FAIL_BS1`, `FAIL_BS2` - Status when step fails

**Multi-Step Example:**
`TEST_SERIAL_MULTI` (lines 283-316) shows the pattern for defining multiple steps for the same process type.

### Files to Reference

| File | Purpose |
| ---- | ------- |
| `/Users/smolik/DEV/cz.imcg.fast.planner/src/zfi_setup_test_data.prog.abap` | Test data setup program - modify TEST_SERIAL_BASIC (lines 244-255), TEST_SERIAL_FAIL_CTRL (lines 257-268), TEST_BS_FLOW (lines 534-549), add TEST_BS_FLOW step 0020 |
| `/Users/smolik/DEV/cz.imcg.fast.planner/src/zcl_fiproc_health_chk_query.clas.abap` | Health check query provider - update check_capability calls (lines 98-115) and check_business_status method (lines 875-954) |
| `/Users/smolik/DEV/cz.imcg.fast.planner/src/zfi_proc_def.tabl.xml` | Process definition table structure - reference for BS1/BS2 field definitions |
| `/Users/smolik/DEV/cz.imcg.fast.allocations/_bmad/_memory/constitution.md` | Project constitution - Principle I (DDIC-First), Principle II (SAP Standards) |

### Technical Decisions

**Business Status Values:**

Use descriptive, test-specific values to distinguish different test scenarios:

**TEST_SERIAL_BASIC (success path):**
- `init_bs1='BASIC IN PROGRESS'`
- `final_bs1='BASIC DONE'`
- `fail_bs1='BASIC FAILED'` (not expected to be used, but required field)

**TEST_SERIAL_FAIL_CTRL (failure path):**
- `init_bs1='FAIL IN PROGRESS'`
- `final_bs1='FAIL DONE'` (not expected to be used)
- `fail_bs1='FAIL CTRL'` (this will be verified in test)

**TEST_BS_FLOW (multi-step transition):**
- Step 0010:
  - `init_bs1='STEP1 IN PROGRESS'`, `init_bs2='STEP1 STATUS2'`
  - `final_bs1='STEP1 DONE'`, `final_bs2='STEP1 COMPLETE'`
  - `fail_bs1='STEP1 FAILED'`, `fail_bs2='STEP1 ERROR'`
- Step 0020:
  - `init_bs1='STEP2 IN PROGRESS'`, `init_bs2='STEP2 STATUS2'`
  - `final_bs1='STEP2 DONE'`, `final_bs2='STEP2 COMPLETE'`
  - `fail_bs1='STEP2 FAILED'`, `fail_bs2='STEP2 ERROR'`

**Rationale for Values:**
- Prefix indicates which test scenario ("BASIC", "FAIL", "STEP1", "STEP2")
- Makes test failures immediately obvious in health check reports
- "STEP1" vs "STEP2" proves transitions occur (final should always be STEP2)

**Reuse Existing Step Classes:**
- `TEST_SERIAL_BASIC` and `TEST_BS_FLOW` use `ZCL_FI_STEP_TEST_SUCCESS` (always succeeds)
- `TEST_SERIAL_FAIL_CTRL` uses `ZCL_FI_STEP_FAIL_SERIAL` (controlled failure via success=false)
- No new step classes needed

**Health Check Assertion Strategy:**
- `TEST_SERIAL_BASIC`: Assert status=COMPLETED AND business_status_1='BASIC DONE'
- `TEST_SERIAL_FAIL_CTRL`: Assert status=FAILED AND business_status_1='FAIL CTRL'
- `TEST_BS_FLOW`: Assert status=COMPLETED AND business_status_1='STEP2 DONE' AND business_status_2='STEP2 COMPLETE'

## Implementation Plan

### Tasks

**Ordered by dependency: Test data setup first, then health check assertions.**

#### Part 1: Test Data Setup (zfi_setup_test_data.prog.abap)

**Task 1: Add Business Status Fields to TEST_SERIAL_BASIC**
- **File**: `src/zfi_setup_test_data.prog.abap`
- **Location**: Lines 244-255 (TEST_SERIAL_BASIC step 0010 definition)
- **Action**: Add three business status fields to the existing step definition
- **Fields to Add**:
  ```abap
  init_bs1  = 'BASIC IN PROGRESS'
  final_bs1 = 'BASIC DONE'
  fail_bs1  = 'BASIC FAILED'
  ```
- **Notes**: Add fields after `substep_mode = 'SERIAL'` line, before the closing parenthesis

**Task 2: Add Business Status Fields to TEST_SERIAL_FAIL_CTRL**
- **File**: `src/zfi_setup_test_data.prog.abap`
- **Location**: Lines 257-268 (TEST_SERIAL_FAIL_CTRL step 0010 definition)
- **Action**: Add three business status fields to the existing step definition
- **Fields to Add**:
  ```abap
  init_bs1  = 'FAIL IN PROGRESS'
  final_bs1 = 'FAIL DONE'
  fail_bs1  = 'FAIL CTRL'
  ```
- **Notes**: Add fields after `substep_mode = 'SERIAL'` line, before the closing parenthesis

**Task 3: Modify TEST_BS_FLOW Step 0010 to Include BS2 Fields**
- **File**: `src/zfi_setup_test_data.prog.abap`
- **Location**: Lines 537-549 (TEST_BS_FLOW step 0010 definition)
- **Action**: Add three BS2 fields to the existing step 0010 definition (BS1 fields already exist)
- **Current BS1 Fields** (keep as-is):
  ```abap
  init_bs1  = 'IN PROGRESS'
  final_bs1 = 'DONE'
  fail_bs1  = 'FAILED'
  ```
- **New BS1 Values** (replace existing):
  ```abap
  init_bs1  = 'STEP1 IN PROGRESS'
  final_bs1 = 'STEP1 DONE'
  fail_bs1  = 'STEP1 FAILED'
  ```
- **BS2 Fields to Add**:
  ```abap
  init_bs2  = 'STEP1 STATUS2'
  final_bs2 = 'STEP1 COMPLETE'
  fail_bs2  = 'STEP1 ERROR'
  ```
- **Notes**: Replace BS1 values to match STEP1 pattern, add BS2 fields after BS1 fields

**Task 4: Add TEST_BS_FLOW Step 0020 Definition**
- **File**: `src/zfi_setup_test_data.prog.abap`
- **Location**: Immediately after TEST_BS_FLOW step 0010 definition (after line 549, before TEST_PARAM_QUEUED section)
- **Action**: Add new step definition entry for step 0020
- **Complete Definition**:
  ```abap
  ( mandt         = sy-mandt
    process_type  = 'TEST_BS_FLOW'
    step_number   = '0020'
    step_type     = 'TEST'
    description   = 'Test: BS flow - second step (BS1+BS2 transition)'
    class_name    = 'ZCL_FI_STEP_TEST_SUCCESS'
    mandatory     = 'X'
    active        = 'X'
    retry_allowed = ''
    substep_mode  = 'SERIAL'
    init_bs1      = 'STEP2 IN PROGRESS'
    init_bs2      = 'STEP2 STATUS2'
    final_bs1     = 'STEP2 DONE'
    final_bs2     = 'STEP2 COMPLETE'
    fail_bs1      = 'STEP2 FAILED'
    fail_bs2      = 'STEP2 ERROR' )
  ```
- **Notes**: Insert this entry in the `lt_definitions` VALUE constructor, maintaining the existing comma separation pattern

**Task 5: Run Test Data Setup Program**
- **Program**: `ZFI_SETUP_TEST_DATA`
- **Action**: Execute the program to persist changes to `ZFI_PROC_DEF` table
- **Verification**: Check program output for success message indicating definitions created/updated
- **Notes**: Program is idempotent, safe to re-run

#### Part 2: Health Check Assertions (zcl_fiproc_health_chk_query.clas.abap)

**Task 6: Add Business Status Assertion to TEST_SERIAL_BASIC Health Check**
- **File**: `src/zcl_fiproc_health_chk_query.clas.abap`
- **Method**: `build_rows`
- **Location**: Lines 98-104 (TEST_SERIAL_BASIC check_capability call)
- **Action**: Modify the `check_capability` method call site to perform BS1 assertion after execution
- **Implementation Strategy**: Since `check_capability` is generic, add a post-check assertion block
- **Pseudocode**:
  ```abap
  " After existing check_capability call for SERIAL_EXECUTION
  " Add assertion for business_status_1
  IF lv_status = 'COMPLETED' AND ls_inst-business_status_1 <> 'BASIC DONE'.
    ls_row-status = gc_red.
    ls_row-message = |BS1 expected 'BASIC DONE', got '{ ls_inst-business_status_1 }'|.
  ENDIF.
  ```
- **Alternative**: Inline the test logic from `check_capability` into `build_rows` for TEST_SERIAL_BASIC to add BS1 assertion
- **Notes**: Update `ls_row-message` to include BS1 verification in success message

**Task 7: Add Business Status Assertion to TEST_SERIAL_FAIL_CTRL Health Check**
- **File**: `src/zcl_fiproc_health_chk_query.clas.abap`
- **Method**: `build_rows`
- **Location**: Lines 106-115 (TEST_SERIAL_FAIL_CTRL check_capability call)
- **Action**: Add fail_bs1 assertion after execution completes
- **Implementation Strategy**: Add post-check assertion for failure path
- **Pseudocode**:
  ```abap
  " After existing check_capability call for SERIAL_FAILURE
  " Add assertion for business_status_1 (failure path)
  IF lv_status = 'FAILED' AND ls_inst-business_status_1 <> 'FAIL CTRL'.
    ls_row-status = gc_red.
    ls_row-message = |BS1 expected 'FAIL CTRL', got '{ ls_inst-business_status_1 }'|.
  ENDIF.
  ```
- **Notes**: Update `ls_row-message` to include BS1 verification in success message

**Task 8: Update check_business_status Method to Assert Both BS1 and BS2**
- **File**: `src/zcl_fiproc_health_chk_query.clas.abap`
- **Method**: `check_business_status`
- **Location**: Lines 875-954
- **Action**: Enhance existing TEST_BS_FLOW assertion to check both BS1 and BS2
- **Current Assertion** (line 932):
  ```abap
  ELSEIF ls_inst-business_status_1 <> 'DONE'.
  ```
- **New Assertion Logic**:
  ```abap
  ELSEIF ls_inst-business_status_1 IS INITIAL.
    ls_row-status  = gc_red.
    ls_row-message = 'BUSINESS_STATUS_1 is empty after completion'.
  ELSEIF ls_inst-business_status_1 <> 'STEP2 DONE'.
    ls_row-status  = gc_red.
    ls_row-message = |Expected BS1='STEP2 DONE', got '{ ls_inst-business_status_1 }'|.
  ELSEIF ls_inst-business_status_2 IS INITIAL.
    ls_row-status  = gc_red.
    ls_row-message = 'BUSINESS_STATUS_2 is empty after completion'.
  ELSEIF ls_inst-business_status_2 <> 'STEP2 COMPLETE'.
    ls_row-status  = gc_red.
    ls_row-message = |Expected BS2='STEP2 COMPLETE', got '{ ls_inst-business_status_2 }'|.
  ELSE.
    ls_row-status  = gc_green.
    ls_row-message = |BS1='{ ls_inst-business_status_1 }' BS2='{ ls_inst-business_status_2 }' — correct|.
  ENDIF.
  ```
- **Update Test Logic Description** (lines 881-884):
  ```abap
  ls_row-testlogic =
    'Creates TEST_BS_FLOW (2 steps: 0010 sets STEP1 values, 0020 sets STEP2 values).'
    && ' Executes process. Reloads from DB. Asserts status=COMPLETED and'
    && ' business_status_1=STEP2 DONE, business_status_2=STEP2 COMPLETE.'
    && ' Proves multi-step BS1+BS2 transitions and last-step-wins behavior.'.
  ```
- **Notes**: This verifies that the final business statuses reflect step 0020, not step 0010

**Task 9: Run Health Check and Verify All Tests Pass**
- **Program/Service**: Execute health check (e.g., via Fiori app or direct RAP query)
- **Action**: Run all health checks and verify GREEN status for:
  - `SERIAL_EXECUTION` (TEST_SERIAL_BASIC) - with BS1='BASIC DONE'
  - `SERIAL_FAILURE` (TEST_SERIAL_FAIL_CTRL) - with BS1='FAIL CTRL'
  - `BUSINESS_STATUS` (TEST_BS_FLOW) - with BS1='STEP2 DONE', BS2='STEP2 COMPLETE'
- **Expected Output**: All three tests show GREEN status with updated success messages
- **Notes**: Any RED status indicates implementation error requiring debugging

### Acceptance Criteria

**AC-1: TEST_SERIAL_BASIC Has BS1 Fields Configured**
- **Given** the test data setup program is executed
- **When** querying `ZFI_PROC_DEF` for process_type='TEST_SERIAL_BASIC' and step_number='0010'
- **Then** the definition includes `init_bs1='BASIC IN PROGRESS'`, `final_bs1='BASIC DONE'`, `fail_bs1='BASIC FAILED'`

**AC-2: TEST_SERIAL_BASIC Health Check Verifies Success Path BS1**
- **Given** the health check suite is executed
- **When** `TEST_SERIAL_BASIC` process completes successfully
- **Then** health check shows GREEN status
- **And** health check message confirms `business_status_1='BASIC DONE'`

**AC-3: TEST_SERIAL_FAIL_CTRL Has BS1 Fields Configured**
- **Given** the test data setup program is executed
- **When** querying `ZFI_PROC_DEF` for process_type='TEST_SERIAL_FAIL_CTRL' and step_number='0010'
- **Then** the definition includes `init_bs1='FAIL IN PROGRESS'`, `final_bs1='FAIL DONE'`, `fail_bs1='FAIL CTRL'`

**AC-4: TEST_SERIAL_FAIL_CTRL Health Check Verifies Failure Path BS1**
- **Given** the health check suite is executed
- **When** `TEST_SERIAL_FAIL_CTRL` process fails as designed
- **Then** health check shows GREEN status
- **And** health check message confirms `business_status_1='FAIL CTRL'`
- **And** process instance status = 'FAILED'

**AC-5: TEST_BS_FLOW Step 0010 Enhanced with BS2 Fields**
- **Given** the test data setup program is executed
- **When** querying `ZFI_PROC_DEF` for process_type='TEST_BS_FLOW' and step_number='0010'
- **Then** the definition includes `init_bs1='STEP1 IN PROGRESS'`, `init_bs2='STEP1 STATUS2'`, `final_bs1='STEP1 DONE'`, `final_bs2='STEP1 COMPLETE'`, `fail_bs1='STEP1 FAILED'`, `fail_bs2='STEP1 ERROR'`

**AC-6: TEST_BS_FLOW Step 0020 Created with Both BS1 and BS2**
- **Given** the test data setup program is executed
- **When** querying `ZFI_PROC_DEF` for process_type='TEST_BS_FLOW' and step_number='0020'
- **Then** the definition exists with all 6 business status fields populated
- **And** values are prefixed with "STEP2" to distinguish from step 0010

**AC-7: TEST_BS_FLOW Two-Step Process Executes Successfully**
- **Given** a new `TEST_BS_FLOW` process instance is created
- **When** the process is executed to completion
- **Then** both step 0010 and step 0020 execute successfully
- **And** process instance status = 'COMPLETED'

**AC-8: TEST_BS_FLOW Final Business Statuses Reflect Last Step**
- **Given** a `TEST_BS_FLOW` process instance has completed
- **When** querying the `ZFI_PROC_INST` table for that instance
- **Then** `business_status_1 = 'STEP2 DONE'` (from step 0020 final_bs1)
- **And** `business_status_2 = 'STEP2 COMPLETE'` (from step 0020 final_bs2)
- **And** BS1/BS2 do NOT show "STEP1" values (proving transition occurred)

**AC-9: TEST_BS_FLOW Health Check Verifies Both BS1 and BS2**
- **Given** the health check suite is executed
- **When** `TEST_BS_FLOW` process completes successfully
- **Then** health check shows GREEN status
- **And** health check message confirms `business_status_1='STEP2 DONE'` AND `business_status_2='STEP2 COMPLETE'`

**AC-10: Idempotent Setup**
- **Given** the test data setup program has been run once
- **When** the program is executed again
- **Then** no errors occur
- **And** all step definitions remain correct (TEST_SERIAL_BASIC, TEST_SERIAL_FAIL_CTRL, TEST_BS_FLOW)

## Additional Context

### Dependencies

- **Framework Business Status Feature**: Already implemented per `story-process-insight-params-bs.md`
- **Existing Step Classes**:
  - `ZCL_FI_STEP_TEST_SUCCESS`: Used by TEST_SERIAL_BASIC and TEST_BS_FLOW (always succeeds)
  - `ZCL_FI_STEP_FAIL_SERIAL`: Used by TEST_SERIAL_FAIL_CTRL (controlled failure)
- **Health Check Infrastructure**: RAP Query Provider pattern already in place
- **Constitution Compliance**:
  - **Principle I (DDIC-First)**: Using existing DDIC table `ZFI_PROC_DEF`, no local types
  - **Principle II (SAP Standards)**: Following existing naming conventions, line length ≤120 chars

### Testing Strategy

**Automated Testing via Health Checks:**
All three test processes have corresponding health check tests that execute automatically:

1. **TEST_SERIAL_BASIC** → Health check: `SERIAL_EXECUTION`
   - Verifies success path with single step
   - Asserts `status='COMPLETED'` AND `business_status_1='BASIC DONE'`
   
2. **TEST_SERIAL_FAIL_CTRL** → Health check: `SERIAL_FAILURE`
   - Verifies failure path with single step
   - Asserts `status='FAILED'` AND `business_status_1='FAIL CTRL'`
   
3. **TEST_BS_FLOW** → Health check: `BUSINESS_STATUS`
   - Verifies success path with two steps
   - Asserts `status='COMPLETED'` AND `business_status_1='STEP2 DONE'` AND `business_status_2='STEP2 COMPLETE'`

**Execution:**
Run health check suite via:
- Fiori app: `ZFI_PROCESS_HEALTH` (if exposed)
- Direct RAP Query execution
- Transaction `SE38` → Program that calls health check query

**Manual Verification (Optional):**
1. Run `ZFI_SETUP_TEST_DATA` in planner repository
2. Execute each test process individually
3. Query `ZFI_PROC_INST` to verify business status values:

```sql
-- TEST_SERIAL_BASIC verification
SELECT instance_id, process_status, business_status_1
  FROM zfi_proc_inst
  WHERE process_type = 'TEST_SERIAL_BASIC'
  ORDER BY created_at DESC
  FETCH FIRST 1 ROWS ONLY;
-- Expected: process_status='COMPLETED', business_status_1='BASIC DONE'

-- TEST_SERIAL_FAIL_CTRL verification
SELECT instance_id, process_status, business_status_1
  FROM zfi_proc_inst
  WHERE process_type = 'TEST_SERIAL_FAIL_CTRL'
  ORDER BY created_at DESC
  FETCH FIRST 1 ROWS ONLY;
-- Expected: process_status='FAILED', business_status_1='FAIL CTRL'

-- TEST_BS_FLOW verification
SELECT instance_id, process_status, business_status_1, business_status_2
  FROM zfi_proc_inst
  WHERE process_type = 'TEST_BS_FLOW'
  ORDER BY created_at DESC
  FETCH FIRST 1 ROWS ONLY;
-- Expected: process_status='COMPLETED', business_status_1='STEP2 DONE', business_status_2='STEP2 COMPLETE'
```

**Success Criteria:**
All health checks show GREEN status with updated assertion messages.

### Notes

**Scope Expansion:**
- Initial request was to add step 0020 to TEST_BS_FLOW only
- During investigation, discovered TEST_SERIAL_BASIC and TEST_SERIAL_FAIL_CTRL had NO business status fields
- Expanded scope to provide comprehensive coverage: success path, failure path, single-step, multi-step, BS1-only, BS1+BS2

**Change Classification:**
- **Additive**: Two new test process definitions (TEST_SERIAL_BASIC BS1, TEST_SERIAL_FAIL_CTRL BS1)
- **Modification**: TEST_BS_FLOW step 0010 BS1 values updated, BS2 fields added
- **Addition**: TEST_BS_FLOW step 0020 created with full BS1+BS2 configuration
- **Enhancement**: Health check assertions updated to verify business statuses

**No Framework Changes Required:**
- Business status feature already implemented
- This change validates existing framework behavior through enhanced test coverage
- All changes confined to test data and test assertions

**Business Status Value Design:**
- Values chosen ("BASIC IN PROGRESS", "FAIL CTRL", "STEP1 DONE", "STEP2 COMPLETE", etc.) are intentionally verbose
- Prefixes distinguish test scenarios: "BASIC" vs "FAIL" vs "STEP1" vs "STEP2"
- Makes test failures immediately obvious in health check reports
- "STEP2" final values prove multi-step transitions work correctly (last step wins)

**Idempotent Setup:**
- `zfi_setup_test_data.prog.abap` uses `MODIFY` statement (not INSERT)
- Safe to re-run after code changes
- No manual cleanup required between test runs

---

## Review Notes

**Adversarial Review Completed**: 2026-03-10

**Findings Summary**: 21 total findings identified
- **Critical**: 2 (Constitution compliance section missing, no SAP docs consultation evidence)
- **High**: 10 (hardcoded values, magic strings, code duplication, missing documentation, no testing evidence)
- **Medium**: 6 (inconsistent naming, timeout hardcoded, incomplete coverage)
- **Low**: 3 (backward compatibility, line length risk, code structure)

**Resolution Approach**: Skip (user chose to acknowledge findings without applying fixes)

**Status**: Implementation complete. User will handle findings separately or consider them informational for this iteration.

**Next Steps**: 
1. Transport/activate ABAP changes in planner repository to SAP system
2. Run `ZFI_SETUP_TEST_DATA` program (SE38) to persist test data changes
3. Execute health checks to verify GREEN status for all three enhanced tests
4. (Optional) Address adversarial review findings in follow-up work
