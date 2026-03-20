# Story EST-121.1: DDIC Foundation for APJ Background Execution

Status: review

## Story

As a framework developer,
I want the DDIC foundation (domain values, data elements, table fields, exception messages) for APJ background execution in place,
so that all subsequent EST-121 stories can compile and activate against properly defined DDIC objects.

## Acceptance Criteria

1. Given the `ZFI_PROCESS_STATUS` domain, when inspected in SE11, then EXEC_REQUESTED (VALPOS 0009) and RESTART_REQUESTED (VALPOS 0010) exist as fixed values.
2. Given the data elements `ZFI_PROCESS_JOBNAME` (CHAR 32) and `ZFI_PROCESS_JOBCOUNT`, when inspected in SE11, then they exist with correct types, descriptions, and label texts.
3. Given the `ZFI_PROC_INST` table, when inspected, then `JOB_NAME` (rollname ZFI_PROCESS_JOBNAME) and `JOB_COUNT` (rollname ZFI_PROCESS_JOBCOUNT) are appended after `BUSINESS_STATUS_2`.
4. Given the exception class `ZCX_FI_PROCESS_ERROR`, when inspected, then constant `scheduling_failed` exists with msgid `ZFI_PROCESS` / msgno `019`.
5. Given the message class `ZFI_PROCESS`, when inspected, then message 019 exists with text "Background job scheduling failed: &1".
6. All DDIC objects activate cleanly. Existing code (all classes referencing ZFI_PROC_INST) still compiles.

## Tasks / Subtasks

- [x] Task 1: Add EXEC_REQUESTED and RESTART_REQUESTED domain values (AC: #1)
  - [x] Open `src/zfi_process_status.doma.xml`
  - [x] Verify highest VALPOS is 0008 (SUPERSEDED from EST-110)
  - [x] Append two `<DD07V>` entries: VALPOS 0009 EXEC_REQUESTED, VALPOS 0010 RESTART_REQUESTED
  - [x] Verify both values fit within CHAR(20) domain length (14 and 17 chars respectively)

- [x] Task 2: Create DDIC data elements for JOB_NAME and JOB_COUNT (AC: #2)
  - [x] Create `src/zfi_process_jobname.dtel.xml` — ZFI_PROCESS_JOBNAME, CHAR(32), labels "APJ Job Name"
  - [x] Create `src/zfi_process_jobcount.dtel.xml` — ZFI_PROCESS_JOBCOUNT
  - [x] **IMPORTANT pre-check:** Verify actual type of `CL_APJ_RT_API=>TY_JOBCOUNT` in target system. If NUMC(8), use DATATYPE=NUMC. If CHAR(8), use DATATYPE=CHAR. Default assumption: CHAR(8).
  - [x] Constitution Principle I: Both fields use dedicated DDIC data elements, not raw built-in types

- [x] Task 3: Add JOB_NAME and JOB_COUNT fields to ZFI_PROC_INST (AC: #3)
  - [x] Open `src/zfi_proc_inst.tabl.xml`
  - [x] Verify the existing field order ends with BUSINESS_STATUS_2 (if additional fields added, insert after actual last field)
  - [x] Append `<DD03P>` entries for JOB_NAME (ROLLNAME=ZFI_PROCESS_JOBNAME) and JOB_COUNT (ROLLNAME=ZFI_PROCESS_JOBCOUNT)
  - [x] Fields are NOT key fields, NOT required (initial value = blank)
  - [x] Existing records get blank values — no data migration needed

- [x] Task 4: Add exception text ID and T100 message for scheduling errors (AC: #4, #5)
  - [x] Open `src/zcx_fi_process_error.clas.abap`
  - [x] Insert `scheduling_failed` constant block after `instance_superseded` (msgno 019)
  - [x] Open `src/zfi_process.msag.xml`
  - [x] Verify message 019 is free, then insert `<T100>` entry: "Background job scheduling failed: &1"
  - [x] Reserve message 020 for future APJ-related messages

## Dev Notes

### Target Repository
`cz.imcg.fast.planner`

### Constitution Compliance
- **Principle I (DDIC-First):** All new types are DDIC objects — domain values, data elements, table fields. No local TYPE definitions.
- **Principle II (SAP Standards):** SAP naming conventions followed (ZFI_PROCESS prefix). Line length ≤120 chars.
- **Principle V (Error Handling):** New exception constant follows existing pattern (msgid/msgno/attr1-4).

### Dependencies
- **Depends on:** Nothing — this is the foundation story
- **Blocks:** All subsequent EST-121 stories (2-6)
- **Prerequisite knowledge:** EST-110 already added SUPERSEDED at VALPOS 0008

### Files to Modify
| File | Action |
|------|--------|
| `src/zfi_process_status.doma.xml` | Modify — add 2 domain values |
| `src/zfi_process_jobname.dtel.xml` | Create — new data element |
| `src/zfi_process_jobcount.dtel.xml` | Create — new data element |
| `src/zfi_proc_inst.tabl.xml` | Modify — add 2 fields |
| `src/zcx_fi_process_error.clas.abap` | Modify — add scheduling_failed constant |
| `src/zfi_process.msag.xml` | Modify — add message 019 |

### Codebase Patterns
- Domain XML: `<DD07V>` entries inside `<DD07V_TAB>` with VALPOS, DDLANGUAGE, DOMVALUE_L, DDTEXT
- Data element XML: `<DD04V>` with ROLLNAME, DDTEXT, REPTEXT, SCRTEXT_S/M/L, DATATYPE, LENG
- Table XML: `<DD03P>` entries with FIELDNAME, ROLLNAME
- Exception constant: `CONSTANTS: BEGIN OF xxx, msgid TYPE symsgid VALUE 'ZFI_PROCESS', msgno TYPE symsgno VALUE 'NNN', attr1..attr4, END OF xxx.`
- Message XML: `<T100>` with SPRSL=E, ARBGB=ZFI_PROCESS, MSGNR, TEXT

### Critical Warnings
1. **JOB_COUNT type uncertainty:** The default assumption is CHAR(8), but if the target system's `CL_APJ_RT_API=>TY_JOBCOUNT` is NUMC(8), the data element must use NUMC to preserve leading-zero semantics. Check the target system before activating.
2. **Pre-condition checks:** Verify VALPOS 0008 is the highest in the domain XML. Verify BUSINESS_STATUS_2 is the last field in the table XML. If other changes were made since the spec was written, adjust insertion points.

### References
- [Source: tech-spec-est-121-apj-background-execution-lifecycle.md#Tasks 1-4]
- [Source: constitution.md#Principle I - DDIC-First Architecture]
- [Source: constitution.md#Principle V - Error Handling]

## Dev Agent Record

### Agent Model Used
claude-opus-4.6

### Debug Log References
- User clarified `CL_APJ_RT_API=>TY_JOBCOUNT` maps to SAP domain `BTCJOBCNT` (base type TEXT8 = CHAR 8)
- JOB_COUNT data element references SAP standard domain `BTCJOBCNT` directly (same pattern as `ZFI_PROCESS_BAL_OBJECT` → `BALOBJ_D`)
- JOB_NAME uses custom domain `ZFI_PROCESS_JOBNAME` CHAR(32) since no SAP standard domain fits
- Message 019 was confirmed free; message 020 is already used (Step started) so reservation is moot

### Completion Notes List
- Task 1: Added EXEC_REQUESTED (VALPOS 0009) and RESTART_REQUESTED (VALPOS 0010) to ZFI_PROCESS_STATUS domain. Both fit within CHAR(20) limit.
- Task 2: Created ZFI_PROCESS_JOBNAME domain (CHAR 32) + data element. Created ZFI_PROCESS_JOBCOUNT data element referencing SAP standard domain BTCJOBCNT (CHAR 8).
- Task 3: Appended JOB_NAME and JOB_COUNT fields to ZFI_PROC_INST after BUSINESS_STATUS_2. Not key fields, not required.
- Task 4: Added `scheduling_failed` exception constant (msgno 019) to ZCX_FI_PROCESS_ERROR. Added T100 message 019 "Background job scheduling failed: &1" to ZFI_PROCESS message class.

### File List
| File | Repo | Action |
|------|------|--------|
| `src/zfi_process_status.doma.xml` | planner | Modified — added EXEC_REQUESTED (0009), RESTART_REQUESTED (0010) |
| `src/zfi_process_jobname.doma.xml` | planner | Created — CHAR(32) domain for APJ job name |
| `src/zfi_process_jobname.dtel.xml` | planner | Created — data element referencing ZFI_PROCESS_JOBNAME domain |
| `src/zfi_process_jobcount.dtel.xml` | planner | Created — data element referencing SAP BTCJOBCNT domain |
| `src/zfi_proc_inst.tabl.xml` | planner | Modified — added JOB_NAME, JOB_COUNT fields |
| `src/zcx_fi_process_error.clas.abap` | planner | Modified — added scheduling_failed constant (msgno 019) |
| `src/zfi_process.msag.xml` | planner | Modified — added message 019 |
