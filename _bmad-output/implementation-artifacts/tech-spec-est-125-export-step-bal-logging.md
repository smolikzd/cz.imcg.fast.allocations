---
title: 'EST-125: Export Step BAL Logging — Phase 2: Logger Injection'
slug: 'tech-spec-est-125-export-step-bal-logging'
created: '2026-03-24'
updated: '2026-03-24'
status: 'ready-for-dev'
stepsCompleted: ['understand', 'investigate', 'generate', 'review']
tech_stack: ['ABAP 7.58', 'ZFI_PROCESS framework', 'SAP BALI API']
files_to_modify:
  - 'src/zfi_ea_ov_ext/zcl_fi_ov_keboola_extractor.clas.abap'
  - 'src/zfi_alloc_process/zcl_fi_alloc_step_extract.clas.abap'
  - 'src/zfi_ea_ov_ext/zfi_ov_keboola_extract.prog.abap'
code_patterns:
  - 'zif_fi_process_logger->message() for all BAL logging'
  - 'IF mo_log IS BOUND guard before every logger call'
  - 'zcl_fi_process_null_logger for standalone callers'
test_patterns:
  - 'Manual: run allocation process with EXPORT step, check SLG1 log'
  - 'Manual: run standalone program zfi_ov_keboola_extract, verify no dumps'
linear_issue: 'EST-125'
target_repository: 'ovysledovka'
constitution_principles:
  - 'Principle II - SAP Standards (ABAP-Doc, naming, line length ≤120)'
  - 'Principle III - Consult SAP Docs (BALI API patterns)'
  - 'Principle V - Error Handling & Observability (exception logging, context in messages)'
---

# Tech-Spec: EST-125 -- Export Step BAL Logging (Phase 2)

**Created:** 2026-03-24
**Linear Issue:** [EST-125](https://linear.app/smolikzd/issue/EST-125/)
**Target Repository:** `cz.imcg.fast.ovysledovka`

## Phase History

**Phase 1 (completed, commit `d522096`):** Fixed 5 logging issues in the step class only:
class header, execution context, CATCH block, orphan MESSAGE, extraction summary row count.
Messages 600/601 updated with `&1`/`&2` placeholders.

**Phase 2 (this spec):** Inject the framework logger into the extractor class, replace
the legacy `ZCL_SP_BALLOG` logging, and add per-file logging for every stored file.

## Overview

### Problem Statement

The Keboola extractor (`ZCL_FI_OV_KEBOOLA_EXTRACTOR`) creates its own BAL log
(`ZCL_SP_BALLOG`, object `ZFI`/subobject `ZFI_KBC`) that is completely separate from
the process framework log. This means:

1. **Per-file storage events are invisible** in the process BAL log (SLG1) -- operators
   cannot see which files were written or how many.
2. **Two BAL logs** are created for a single export step execution -- confusing for support.
3. The extractor's internal messages (package counts, iteration progress, file names)
   go to a separate log that nobody looks at.
4. The existing `MESSAGE s001(00) WITH ...` progress lines in `execute()` are status-bar
   only messages -- they never reach any BAL log at all.

### Solution

1. **Replace `mo_log` (ZCL_SP_BALLOG)** in the extractor with `mo_log` typed as
   `zif_fi_process_logger` (the framework logger interface).
2. **Add optional constructor parameter** `io_log TYPE REF TO zif_fi_process_logger`.
3. **Convert all logging** from `MESSAGE...INTO lv_dummy` + `mo_log->log_sy_msg()` to
   `mo_log->message()` with explicit T100 parameters. Keep message class `ZFI_OV_KBC`.
4. **Add per-file logging**: after each FTP upload or app server write, log the filename
   via `mo_log->message()`.
5. **Convert status-bar MESSAGE s001(00)** progress lines to `mo_log->message()` calls
   using `ZFI_OV_KBC` message 000 (generic `&1 &2 &3 &4`).
6. **Remove** the `ZCL_SP_BALLOG` instantiation from the constructor (no more separate log).
7. **Step class** passes its `mo_log` to the extractor constructor.
8. **Standalone program** passes `zcl_fi_process_null_logger` (no-op, silent).

### Scope

**In Scope:**
- Replace `ZCL_SP_BALLOG` with `zif_fi_process_logger` in extractor
- Convert all ~12 active `mo_log->log_sy_msg()` calls to `mo_log->message()`
- Convert ~13 `MESSAGE s001(00)` progress lines to `mo_log->message()`
- Add per-file filename logging after `send_to_ftp()` and `send_to_appl_server()`
- Step class: pass `mo_log` to extractor constructor
- Standalone program: instantiate and pass `zcl_fi_process_null_logger`

**Out of Scope:**
- Changes to the ZFI_PROCESS framework (planner repo)
- New messages in `ZFI_OV_KBC` (existing messages 000-008 are sufficient)
- File size logging (filename only per decision)
- Changing the extractor's business logic, file format, or storage methods
- Fixing duplicate `mt_semtag_filter` entries (separate tech debt)
- Adding unit tests

## Context for Development

### Current Logging Inventory in Extractor

#### `mo_log->log_sy_msg()` calls (routed to ZCL_SP_BALLOG today):

| Location | Message | Purpose |
|----------|---------|---------|
| `execute()` L160-161 | `s001(zfi_ov_kbc)` | "Get packages relevant for extraction" |
| `execute()` L167-168 | `s002(zfi_ov_kbc) WITH sy-dbcnt` | "Number of selected packages: &1" |
| `execute()` L171-172 | `s003(zfi_ov_kbc)` | "Start extraction process" |
| `execute()` L179-180 | `e008(zfi_ov_kbc) WITH recordcount` | "Largest package exceeds max rows: &1" |
| `get_data()` L314-315 | `s004(zfi_ov_kbc) WITH mv_part iv_count` | "Iteration: &1, reading: &2" |
| `get_data()` L338-339 | `s005(zfi_ov_kbc) WITH mv_part sy-dbcnt` | "Iteration: &1, extracted: &2" |
| `get_data()` L395-396 | `s007(zfi_ov_kbc) WITH lv_file` | "Saving data into file &1" |
| `trailing_file()` L551-552 | `s007(zfi_ov_kbc) WITH lv_file` | "Saving data into file &1" (trailing) |
| `send_to_appl_server()` L581-582 | `e006(zfi_ov_kbc) WITH filename` | "File &1 could not be opened" |
| `send_finished_to_appl_server()` L609-610 | `e006(zfi_ov_kbc) WITH filename` | "File &1 could not be opened" |

#### `MESSAGE s001(00)` calls (status bar only, no BAL):

| Location | Content |
|----------|---------|
| `execute()` L197 | `processing semantic tag: {lv_staggroup}` |
| `execute()` L200 | `group packet no: {lv_packet_no}` |
| `execute()` L209 | `group packet {n} rows: {count}` |
| `execute()` L229 | `group packet no: {lv_packet_no}` |
| `execute()` L238 | `group packet {n} rows: {count}` |
| `execute()` L247-252 | stats: groups total/left, group rows, total rows, percent, separator |

### Message Class ZFI_OV_KBC (8 messages)

| # | Text | Placeholders |
|---|------|-------------|
| 000 | `&1 &2 &3 &4` | Generic 4-slot |
| 001 | `Get packages relevant for extraction` | None |
| 002 | `Number of selected packages: &1` | &1=count |
| 003 | `Start extraction process` | None |
| 004 | `Iteration: &1, reading packes with record count &2.` | &1=part, &2=count |
| 005 | `Iteration: &1, extracted records: &2.` | &1=part, &2=count |
| 006 | `File &1 could not be opened.` | &1=filename |
| 007 | `Saving data into file &1.` | &1=filename |
| 008 | `Largest packages is bigger than max number of rows. Records &1.` | &1=count |

### Files to Modify

| File | Changes |
|------|---------|
| `src/zfi_ea_ov_ext/zcl_fi_ov_keboola_extractor.clas.abap` | Replace logger type, convert all logging, add per-file log |
| `src/zfi_alloc_process/zcl_fi_alloc_step_extract.clas.abap` | Pass `mo_log` to extractor constructor |
| `src/zfi_ea_ov_ext/zfi_ov_keboola_extract.prog.abap` | Pass `zcl_fi_process_null_logger` to constructor |

### Files to Reference (read-only)

| File | Repo | Purpose |
|------|------|---------|
| `src/zcl_fi_process_step.clas.abap` | planner | Base class with `mo_log` typed `zif_fi_process_logger` |
| `src/zif_fi_process_logger.intf.abap` | planner | Logger interface: `message()`, `log_exception_from_framework()` |
| `src/zcl_fi_process_null_logger.clas.abap` | planner | No-op logger for standalone callers |

## Implementation Plan

### Task 1: Replace logger type in extractor class definition

**File:** `zcl_fi_ov_keboola_extractor.clas.abap`, definition section

Change `mo_log` type from `ZCL_SP_BALLOG` to `zif_fi_process_logger`:

```abap
" BEFORE:
data MO_LOG type ref to ZCL_SP_BALLOG read-only .

" AFTER:
data mo_log type ref to zif_fi_process_logger read-only .
```

Add optional `io_log` parameter to the constructor:

```abap
" BEFORE:
methods CONSTRUCTOR
  importing
    !IV_GJAHR type GJAHR
    ...
    !IT_SEMTAG_FILTER type TY_R_FINS_SEM_TAG optional
  raising
    ZCX_FI_OV_KEBOOLA_EXTRACTOR .

" AFTER (add io_log as first IMPORTING):
methods CONSTRUCTOR
  importing
    !io_log type ref to zif_fi_process_logger optional
    !IV_GJAHR type GJAHR
    ...
    !IT_SEMTAG_FILTER type TY_R_FINS_SEM_TAG optional
  raising
    ZCX_FI_OV_KEBOOLA_EXTRACTOR .
```

### Task 2: Rewrite constructor -- remove ZCL_SP_BALLOG creation

**File:** `zcl_fi_ov_keboola_extractor.clas.abap`, method `constructor`

```abap
" BEFORE (lines 113-131):
DATA ls_log TYPE bal_s_log.
...
"Log initialization
CREATE OBJECT mo_log.
ls_log-object    = 'ZFI'.
ls_log-subobject = 'ZFI_KBC'.
mo_log->log_create( is_log = ls_log ).

" AFTER:
mo_log = io_log.
```

Remove `DATA ls_log TYPE bal_s_log.` and the entire BAL log creation block.
Keep all other constructor logic (parameter assignments, FTP config, etc.) unchanged.

### Task 3: Convert `mo_log->log_sy_msg()` calls to `mo_log->message()`

Every occurrence of the pattern:
```abap
MESSAGE sNNN(zfi_ov_kbc) WITH <args> INTO lv_dummy.
mo_log->log_sy_msg( ).
```

Must be replaced with:
```abap
IF mo_log IS BOUND.
  mo_log->message(
    iv_message_class  = 'ZFI_OV_KBC'
    iv_message_number = 'NNN'
    iv_message_v1     = CONV symsgv( <arg1> )
    ...
    iv_severity       = 'S'  " or 'E' for error messages
  ).
ENDIF.
```

**Conversion table (10 active call sites):**

| Line | Old message | Severity | v1 | v2 | Notes |
|------|-------------|----------|----|----|-------|
| 160-161 | `s001` | S | -- | -- | No placeholders |
| 167-168 | `s002 WITH sy-dbcnt` | S | `sy-dbcnt` | -- | Log after SELECT |
| 171-172 | `s003` | S | -- | -- | No placeholders |
| 179-180 | `e008 WITH recordcount` | E | `recordcount` | -- | Error: max rows exceeded |
| 314-315 | `s004 WITH mv_part iv_count` | S | `mv_part` | `iv_count` | |
| 338-339 | `s005 WITH mv_part sy-dbcnt` | S | `mv_part` | `sy-dbcnt` | Log after SELECT |
| 395-396 | `s007 WITH lv_file` | S | `lv_file` | -- | File name |
| 551-552 | `s007 WITH lv_file` | S | `lv_file` | -- | Trailing file name |
| 581-582 | `e006 WITH filename` | E | `filename` | -- | App server open error |
| 609-610 | `e006 WITH filename` | E | `filename` | -- | Trailing file open error |

**Important:** Remove all `lv_dummy` variables that were used only for `MESSAGE...INTO`.
If `lv_dummy` is declared with `DATA(lv_dummy)` inline, remove that too.

### Task 4: Convert `MESSAGE s001(00)` progress lines

The ~13 `MESSAGE s001(00) WITH |...|` lines in `execute()` are status-bar-only messages.
Convert them to BAL logging using the generic message `ZFI_OV_KBC/000` (`&1 &2 &3 &4`):

```abap
" BEFORE:
MESSAGE s001(00) WITH |processing semantic tag: { lv_staggroup }|.

" AFTER:
IF mo_log IS BOUND.
  mo_log->message(
    iv_message_class  = 'ZFI_OV_KBC'
    iv_message_number = '000'
    iv_message_v1     = CONV symsgv( |tag: { lv_staggroup }| )
  ).
ENDIF.
```

Apply this pattern to all MESSAGE s001(00) lines. Consolidate where possible --
e.g., the stats block (lines 247-252) can be merged into fewer log entries:

```abap
" BEFORE (6 separate MESSAGE calls):
MESSAGE s001(00) WITH |groups total: { lv_total_groups }|.
MESSAGE s001(00) WITH |groups left: { groups_left }|.
MESSAGE s001(00) WITH |group rows: { lv_group_row_count }|.
MESSAGE s001(00) WITH |total rows: { mv_total_rows }|.
MESSAGE s001(00) WITH |completed: { percent_done }%|.
MESSAGE s001(00) WITH |---|.

" AFTER (2 log entries):
IF mo_log IS BOUND.
  mo_log->message(
    iv_message_class  = 'ZFI_OV_KBC'
    iv_message_number = '000'
    iv_message_v1     = CONV symsgv( |{ lv_processed_groups }/{ lv_total_groups } groups| )
    iv_message_v2     = CONV symsgv( |{ lv_group_row_count } rows| )
    iv_message_v3     = CONV symsgv( |total: { mv_total_rows }| )
    iv_message_v4     = CONV symsgv( |{ percent_done }%| )
  ).
ENDIF.
```

### Task 5: Add per-file logging after storage

After each file is stored (FTP or app server), log the filename. Add logging
at the end of each storage method:

**`send_to_ftp()` -- after `lo_ftp->disconnect()` (line 491):**
```abap
IF mo_log IS BOUND.
  mo_log->message(
    iv_message_class  = 'ZFI_OV_KBC'
    iv_message_number = '007'
    iv_message_v1     = CONV symsgv( |{ mv_csv_filename }.gz| )
    iv_severity       = 'S'
  ).
ENDIF.
```

**`send_to_appl_server()` -- after `CLOSE DATASET` (both compressed and uncompressed paths):**
```abap
" Uncompressed path (after line 588):
IF mo_log IS BOUND.
  mo_log->message(
    iv_message_class  = 'ZFI_OV_KBC'
    iv_message_number = '007'
    iv_message_v1     = CONV symsgv( mv_csv_filename_fp )
    iv_severity       = 'S'
  ).
ENDIF.

" Compressed path (after line 596):
IF mo_log IS BOUND.
  mo_log->message(
    iv_message_class  = 'ZFI_OV_KBC'
    iv_message_number = '007'
    iv_message_v1     = CONV symsgv( lv_filename )
    iv_severity       = 'S'
  ).
ENDIF.
```

**`send_finished_to_ftp()` -- after `lo_ftp->disconnect()` (line 643):**
```abap
IF mo_log IS BOUND.
  mo_log->message(
    iv_message_class  = 'ZFI_OV_KBC'
    iv_message_number = '007'
    iv_message_v1     = CONV symsgv( mv_csv_finished_filename )
    iv_severity       = 'S'
  ).
ENDIF.
```

**`send_finished_to_appl_server()` -- after `CLOSE DATASET` (line 614):**
```abap
IF mo_log IS BOUND.
  mo_log->message(
    iv_message_class  = 'ZFI_OV_KBC'
    iv_message_number = '007'
    iv_message_v1     = CONV symsgv( mv_csv_finished_filename_fp )
    iv_severity       = 'S'
  ).
ENDIF.
```

### Task 6: Step class -- pass logger to extractor

**File:** `zcl_fi_alloc_step_extract.clas.abap`, method `execute`

Add `io_log = mo_log` to the extractor constructor call:

```abap
" BEFORE:
mo_kbc = NEW zcl_fi_ov_keboola_extractor(
  iv_gjahr         = mv_fiscal_year
  iv_poper         = mv_fiscal_period
  ...
).

" AFTER:
mo_kbc = NEW zcl_fi_ov_keboola_extractor(
  io_log           = mo_log
  iv_gjahr         = mv_fiscal_year
  iv_poper         = mv_fiscal_period
  ...
).
```

### Task 7: Standalone program -- pass null logger

**File:** `zfi_ov_keboola_extract.prog.abap`

Add `zcl_fi_process_null_logger` instantiation before the extractor constructor:

```abap
" BEFORE:
try.
    lo_kbc = new zcl_fi_ov_keboola_extractor(
      iv_gjahr         = p_gjahr
      ...
    ).

" AFTER:
try.
    lo_kbc = new zcl_fi_ov_keboola_extractor(
      io_log           = new zcl_fi_process_null_logger( )
      iv_gjahr         = p_gjahr
      ...
    ).
```

## Acceptance Criteria

**AC1: All extractor logging goes through framework logger**
- Given: The export step executes via the process framework
- When: The extractor logs messages during execution
- Then: All messages appear in the process BAL log (single SLG1 entry), NOT in a
  separate ZFI/ZFI_KBC log

**AC2: Per-file storage is logged**
- Given: The extractor creates N data part files + 1 trailing file
- When: Each file is stored (FTP or app server)
- Then: The BAL log contains message 007 ("Saving data into file &1") with the
  actual filename for each stored file

**AC3: Progress messages reach BAL**
- Given: The extractor iterates over semantic tag groups
- When: Each group is processed
- Then: The BAL log contains progress entries (package counts, row counts, group
  progress) that were previously status-bar-only

**AC4: Standalone program works with null logger**
- Given: `zfi_ov_keboola_extract` is executed directly (SE38/SA38)
- When: It passes `zcl_fi_process_null_logger` to the extractor
- Then: No runtime errors occur. No BAL log is created. Extraction runs normally.

**AC5: No ZCL_SP_BALLOG references remain**
- Given: All modified files
- Then: No references to `ZCL_SP_BALLOG`, `bal_s_log`, `log_create`, or `log_sy_msg`
  exist in the extractor class

**AC6: Line length compliance**
- Given: All modified/added lines
- Then: No line exceeds 120 characters (per constitution)

## Technical Decisions

1. **`io_log` is optional** in the constructor signature to preserve compilation
   without a caller change. However, both known callers (step class, standalone program)
   will always pass a value. When `io_log` is not supplied, `mo_log` stays unbound
   and all `IF mo_log IS BOUND` guards skip logging silently.

2. **Keep message class ZFI_OV_KBC.** The extractor is a standalone business class
   in package `ZFI_EA_OV_EXT`, not part of the allocation step package. Its messages
   belong to its own domain. The framework logger accepts any T100 message class.

3. **Consolidate verbose stats.** The 6-line stats block per group iteration
   (lines 247-252) should be merged into 1-2 log entries using message 000's four
   placeholder slots.

4. **Reuse message 007** ("Saving data into file &1") for per-file logging. This
   message already exists and has the right semantics. It is called in `get_data()`
   before storage and should also be called after storage in each `send_*` method
   to confirm the file was actually written.

## Dependencies

- `zcl_fi_process_null_logger` must exist in the planner repo (already present,
  commit `871472c`).
- `zif_fi_process_logger` interface must be available (already present in planner repo).
- No new DDIC objects or message definitions are required.

## Testing Strategy

1. **Manual test -- happy path via framework**: Run full allocation process through
   EXPORT step. Check SLG1 -- verify all extractor messages appear in the same BAL
   log as other step messages. Verify per-file entries with filenames.
2. **Manual test -- standalone program**: Run `zfi_ov_keboola_extract` via SE38.
   Verify no runtime errors and no BAL log created.
3. **Negative test -- extractor failure**: Simulate FTP connection error. Verify
   exception details appear in the process BAL log.

## Notes

- The `mt_semtag_filter` in step `init()` contains duplicate entries. Separate issue.
- Local TYPE definitions in both files violate Constitution Principle I (DDIC-First).
  Pre-existing tech debt, out of scope.
- The `1 = 1` guard in `zip_data()` (line 511) is pre-existing dead code. Out of scope.
- Commented-out code blocks throughout the extractor are pre-existing. Out of scope.
