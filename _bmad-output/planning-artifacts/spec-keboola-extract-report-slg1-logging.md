# Tech Spec: Wire Real SLG1 Logging into Standalone ZFI_OV_KEBOOLA_EXTRACT Report

**Target Repository:** `cz.imcg.fast.ovysledovka`
**Author:** BMad create-story workflow (planning repo, ad-hoc ops request — no Linear ticket yet)

## 1. Problem

User ran the standalone report `ZFI_OV_KEBOOLA_EXTRACT` and found **no SLG1 messages at all**, despite the recent work (`keboola-semtag-row-count-logging`) adding rich per-semantic-tag extraction/row-count messages inside `ZCL_FI_OV_KEBOOLA_EXTRACTOR->EXECUTE`.

Root cause: `zfi_ov_keboola_extract.prog.abap:55` instantiates the extractor with a **null logger**:

```abap
lo_kbc = new zcl_fi_ov_keboola_extractor(
  io_log = new zcl_fi_process_null_logger( )   " <-- discards everything
  ...
```

`ZCL_FI_PROCESS_NULL_LOGGER` is an intentional no-op implementation of `ZIF_FI_PROCESS_LOGGER` (`message`, `save`, `log_exception_from_framework`, `save_with_2nd_connection` all empty). Every `mo_log->message(...)` call inside `ZCL_FI_OV_KEBOOLA_EXTRACTOR` is guarded by `IF mo_log IS BOUND` and executes, but the messages are built and immediately discarded — nothing ever reaches BAL/SLG1.

By contrast, the productive path — `ZCL_FI_ALLOC_STEP_EXTRACT` (step `EXPORT` of the `ALLOC_EXPORT` process, run via `ZCL_FI_ALLOC_JOB_EXPORT` APJ) — passes the **real** framework logger (`mo_log`, a `ZCL_FI_PROCESS_LOGGER` instance created by the framework, BAL object `ZFI_ALLOC` / subobject `PROCESS`) into the very same `ZCL_FI_OV_KEBOOLA_EXTRACTOR`, so that path already logs correctly to SLG1.

## 2. Goal

Make the standalone report `ZFI_OV_KEBOOLA_EXTRACT` produce a real SLG1 log for every run, using a **new, dedicated BAL subobject** so its ad-hoc/manual runs are clearly distinguishable from the productive `ALLOC_EXPORT` process log stream in SLG1, without touching `ZCL_FI_OV_KEBOOLA_EXTRACTOR`'s logging logic (already correct) or the `ALLOC_EXPORT` process path (already correct).

**Agreed naming:** BAL object `ZFI_ALLOC`, subobject **`KEBOOLA_EXTRACT`**.

## 3. Current Code Reference

File: `src/zfi_ea_ov_ext/zfi_ov_keboola_extract.prog.abap`, lines 53-88 (`TRY` block around report execution).

```abap
try.
    lo_kbc = new zcl_fi_ov_keboola_extractor(
      io_log           = new zcl_fi_process_null_logger( )
      iv_gjahr         = p_gjahr
      ...
    ).
    ...
    lo_kbc->execute( ).
    ...
  catch zcx_fi_ov_keboola_extractor into data(lo_ex).
   message 'Technical issue during extraction process.'(E01) type 'E' .
endtry.
```

No `save()` call exists anywhere in the report today (irrelevant while the logger is a no-op, but will matter once a real logger is wired in — `ZIF_FI_PROCESS_LOGGER->save` must be called explicitly to flush to the DB/SLG1; nothing auto-flushes unless `iv_flush_interval` is reached).

## 4. Proposed Changes

### 4.1 Register the new BAL subobject `KEBOOLA_EXTRACT` under object `ZFI_ALLOC`

BAL object/subobject values are maintained via transaction `SLG0`. Add subobject `KEBOOLA_EXTRACT` under existing object `ZFI_ALLOC` (do **not** create a new BAL object — reuse `ZFI_ALLOC` for consistency with the rest of the allocation logging, matching the existing pattern of `PHASE1`/`PHASE2`/`PHASE3`/`CORR_BCHE`/`PROCESS` subobjects already defined in `ZCL_FI_ALLOCATIONS`/`ZFI_PROC_TYPE` config).

Optionally add a constant alongside the existing ones in `ZCL_FI_ALLOCATIONS` (`src/zcl_fi_allocations.clas.abap`) for consistency:

```abap
CONSTANTS c_bal_subobj_keboola_extract TYPE balsubobj VALUE 'KEBOOLA_EXTRACT'. "#EC NOTEXT
```

### 4.2 Replace the null logger with a real `ZCL_FI_PROCESS_LOGGER` instance in the report

```abap
try.
    data(lo_log) = zcl_fi_process_logger=>create_new(
      iv_external_number = |KEBOOLA_{ sy-datum }{ sy-uzeit }|
      iv_object          = 'ZFI_ALLOC'
      iv_subobject       = 'KEBOOLA_EXTRACT'
    ).

    lo_kbc = new zcl_fi_ov_keboola_extractor(
      io_log           = lo_log
      iv_gjahr         = p_gjahr
      iv_poper         = p_poper
      iv_date          = sy-datum
      iv_compress      = p_zip
      iv_ftp_storage   = p_ftp
      iv_packet_size   = p_max_r
      iv_path          = p_path
      iv_data_aggr     = p_daggr
      it_semtag_filter = lt_semtag_filter
      iv_bukrs         = p_bukrs
    ).
    ...
    lo_kbc->execute( ).
    ...
    lo_log->save( ).

  catch zcx_fi_ov_keboola_extractor into data(lo_ex).
    lo_log->log_exception_from_framework( lo_ex ).
    lo_log->save( ).
    message 'Technical issue during extraction process.'(E01) type 'E' .

  catch zcx_fi_process_error into data(lo_log_ex).
    " logger creation itself failed — surface to the user, extraction never ran
    message 'Failed to initialize application log.'(E02) type 'E'.
endtry.
```

Key points:
- `zcl_fi_process_logger=>create_new` raises `zcx_fi_process_error` — add a dedicated `CATCH` so a logger-creation failure doesn't get silently swallowed or confused with an extraction failure.
- `iv_external_number` (type `balnrext`) must be populated with something correlating to this run (date+time is sufficient here since this is a manual/ad-hoc report, not a framework-tracked process instance with a UUID).
- Call `lo_log->save( )` **both** on the happy path and in the `CATCH zcx_fi_ov_keboola_extractor` branch, so partial-run logs are still visible in SLG1 even when extraction fails partway through.
- Text elements `E01`/`E02` — confirm `E02` doesn't already exist in the report's text pool before adding.

### 4.3 Do not change `ZCL_FI_OV_KEBOOLA_EXTRACTOR` or `ZCL_FI_ALLOC_STEP_EXTRACT`

Both already handle `mo_log`/`io_log` correctly (guarded `IF mo_log IS BOUND`, framework logger passed through). This spec only touches the standalone report's instantiation + a new BAL subobject registration.

## 5. Constitution Compliance

- **Principle I (DDIC-First):** No new local types; `balnrext`/`balobj_d`/`balsubobj` are existing DDIC types used as-is.
- **Principle II (SAP Standards):** New/changed lines ≤120 chars; follow existing report coding style (lowercase keywords, as used in this program today).
- **Principle IV (Factory Pattern):** Use `zcl_fi_process_logger=>create_new(...)` factory method — do not `NEW zcl_fi_process_logger(...)` directly (class is `CREATE PRIVATE` and enforces this already).
- **Principle V (Error Handling):** Add explicit `CATCH zcx_fi_process_error` around logger creation; ensure `save()` is called on both success and the existing `zcx_fi_ov_keboola_extractor` failure path so partial logs aren't lost.

## 6. Verification Plan

1. Run `ZFI_OV_KEBOOLA_EXTRACT` for a period with data — confirm SLG1 (object `ZFI_ALLOC`, subobject `KEBOOLA_EXTRACT`) shows the full set of per-semantic-tag messages from `keboola-semtag-row-count-logging` (start/packet/summary/warning messages) for this run's external number.
2. Run with a semantic tag filter known to yield zero packages — confirm the `W`-severity "no packages found" message (from the earlier logging story) is now visible in SLG1 for this report too, not just via the `ALLOC_EXPORT` framework path.
3. Force a failure (e.g. invalid FTP config with `p_ftp = 'X'`) — confirm the exception is logged via `log_exception_from_framework` and `save()` still flushes the partial log before the `MESSAGE ... TYPE 'E'` is raised.
4. Confirm the productive `ALLOC_EXPORT` process path (`ZCL_FI_ALLOC_STEP_EXTRACT` → BAL object `ZFI_ALLOC` / subobject `PROCESS`) is completely unaffected — its log stream must remain separate from the new `KEBOOLA_EXTRACT` subobject stream.
5. Confirm `p_cnts` (count-only) mode still works and produces a sensible log entry (or at minimum doesn't error) with the new logger wired in.

## 7. Out of Scope

- Any change to extraction/aggregation logic, file naming, zipping, or FTP transfer.
- Any change to `ZCL_FI_ALLOC_STEP_EXTRACT` / `ALLOC_EXPORT` process framework path (already correct).
- Migrating the standalone report to run through the `ZFI_PROCESS` framework itself (raised as an open question to the user — not pursued here; this spec only fixes SLG1 visibility for the report as-is).
