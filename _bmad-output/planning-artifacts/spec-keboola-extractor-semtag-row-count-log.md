# Tech Spec: Log Per-Semantic-Tag Export Row/Document Counts in ZCL_FI_OV_KEBOOLA_EXTRACTOR

**Target Repository:** `cz.imcg.fast.ovysledovka`
**Author:** BMad create-story workflow (planning repo, ad-hoc ops request — no Linear ticket yet)

## 1. Problem

One of the gz CSV files produced by `ZCL_FI_OV_KEBOOLA_EXTRACTOR` and shipped to the FTP server was found empty, and it's unclear why. Extraction is split per **semantic tag** (`fins_sem_tag`), each producing its own set of CSV part files. The working theory is that the given semantic tag simply had **no matching data** for the extracted fiscal year/period(/company code), so its resulting file(s) end up with zero data rows.

There is currently no clear, aggregated application-log entry stating: "Semantic tag X → N rows/documents exported" for the whole run. Per-packet row counts ARE logged (`ZFI_OV_KBC 000`, `iv_message_v2 = 'rows: <n>'`) inside the packet loop in `execute()`, but:
- These are logged **per packet chunk**, not summed per semantic tag — a developer/support person has to manually add up multiple messages to know the total for one tag.
- There is **no explicit warning/notice when a semantic tag's total is zero**, which is exactly the diagnostic signal needed here.
- `SET_COUNT_ONLY` mode (`get_data` with `mv_count_only = abap_true`) already computes `ev_count` via `SELECT COUNT(*)` without loading data — this pattern could be reused/referenced for lightweight diagnostics, but it's not wired into the normal `execute()` flow for a final per-tag summary.

## 2. Goal

For every fiscal-year/period export run, the application log (via `mo_log` / `ZIF_FI_PROCESS_LOGGER`, message class `ZFI_OV_KBC`) must clearly show, **per semantic tag**:
- The total number of rows/documents exported (sum across all packets/parts for that tag).
- A distinct, easy-to-spot log entry (severity `W` or higher) when a semantic tag's total is **zero**, so this scenario is no longer silent and can be correlated with the "empty gz file" symptom.

This must **not** change the extraction logic, file naming, packaging, or FTP transfer behavior — logging only.

## 3. Current Code Reference

File: `src/zfi_ea_ov_ext/zcl_fi_ov_keboola_extractor.clas.abap`

Relevant loop structure in `EXECUTE` (approx. lines 217–324):

```abap
LOOP AT lt_packages INTO wa_package
  GROUP BY wa_package-SemanticTag INTO DATA(lv_staggroup).
  lv_processed_groups += 1.
  lv_packet_no = 1.
  " ... per-tag start message (msg 000, v1 = "tag: <lv_staggroup>") ...
  LOOP AT GROUP lv_staggroup INTO DATA(wa_grouppackage).
    IF ( lv_count + wa_grouppackage-recordcount ) > mv_packet_size.
      " flush current packet: get_data(...) -> ev_count = lv_group_row_count
      " msg 000 "packet <n> rows: <lv_group_row_count>"
      lv_packet_no += 1.
      mv_total_rows += lv_group_row_count.        " <-- running GRAND total across ALL tags
      CLEAR: lv_count, lr_pid, lr_stag.
      " start next packet accumulation ...
    ELSE.
      lv_count = lv_count + wa_grouppackage-recordcount.
      " accumulate lr_pid ...
    ENDIF.
  ENDLOOP.
  " final flush for this tag's last packet: get_data(...) -> ev_count = lv_group_row_count
  " msg 000 "packet <n> rows: <lv_group_row_count>"
  CLEAR: lv_count, lr_pid, lr_stag.
  " stats block: msg 000 "x/y groups", "n rows" (LAST packet only), "total: mv_total_rows", "pct%"
  mv_total_rows += lv_group_row_count.
ENDLOOP.
```

Key facts:
- `mv_total_rows` (public attribute, read-only) is the **grand total across the whole run** (all semantic tags combined) — not per tag.
- `lv_group_row_count` at the "stats block" only reflects the **last packet's** row count for that tag, not the tag's total — so today's per-tag stats message is actually misleading if a tag has more than one packet.
- There is no per-tag accumulator today.

## 4. Proposed Changes

### 4.1 Add a per-semantic-tag row accumulator in `EXECUTE`

Introduce a local variable that accumulates rows **within the current tag group** only, reset at the start of each `GROUP BY` iteration, separate from the existing grand-total `mv_total_rows`:

```abap
DATA lv_stag_total_rows TYPE i.
...
LOOP AT lt_packages INTO wa_package
  GROUP BY wa_package-SemanticTag INTO DATA(lv_staggroup).
  lv_processed_groups += 1.
  lv_packet_no = 1.
  CLEAR lv_stag_total_rows.                       " <-- reset per tag
  ...
  LOOP AT GROUP lv_staggroup INTO DATA(wa_grouppackage).
    IF ( lv_count + wa_grouppackage-recordcount ) > mv_packet_size.
      ...
      get_data( ... IMPORTING ev_count = DATA(lv_group_row_count) ).
      ...
      lv_packet_no += 1.
      mv_total_rows += lv_group_row_count.
      lv_stag_total_rows += lv_group_row_count.    " <-- accumulate per tag
      ...
    ELSE.
      ...
    ENDIF.
  ENDLOOP.
  " final flush for this tag
  get_data( ... IMPORTING ev_count = lv_group_row_count ).
  ...
  mv_total_rows += lv_group_row_count.
  lv_stag_total_rows += lv_group_row_count.        " <-- accumulate final packet too
```

### 4.2 Emit an explicit per-tag summary log entry after the tag's packets are fully flushed

Right after the existing "stats block" (currently using `mo_log->message` with `iv_message_v1 = "<processed>/<total> groups"` etc.), add a **new, dedicated** log call summarizing the tag total, and branch severity based on whether it is zero:

```abap
IF mo_log IS BOUND.
  IF lv_stag_total_rows = 0.
    mo_log->message(
      iv_message_class  = 'ZFI_OV_KBC'
      iv_message_number = '000'          " or a new dedicated number, see 4.3
      iv_message_v1     = CONV symsgv( |Semantic tag { lv_staggroup }| )
      iv_message_v2     = CONV symsgv( 'exported 0 rows/documents' )
      iv_severity       = 'W'            " Warning — distinct from the normal 'S' info messages
    ).
  ELSE.
    mo_log->message(
      iv_message_class  = 'ZFI_OV_KBC'
      iv_message_number = '000'
      iv_message_v1     = CONV symsgv( |Semantic tag { lv_staggroup }| )
      iv_message_v2     = CONV symsgv( |exported { lv_stag_total_rows } rows/documents| )
      iv_severity       = 'S'
    ).
  ENDIF.
ENDIF.
```

### 4.3 Preferred: add a dedicated message number (cleaner than reusing free-text `000`)

Message class `ZFI_OV_KBC` already has numbered messages (001–008) with placeholders (`&1`..`&4`). Add a new message, e.g. `009`, with text along the lines of:
`Semantic tag &1: &2 rows/documents exported` — reusable for both the zero and non-zero case (severity is what should differ: `W` when `lv_stag_total_rows = 0`, `S` otherwise). This keeps the message catalog consistent with the rest of the class instead of relying on free-text via `000`.

> Since this repo (`cz.imcg.fast.allocations`) does not contain ABAP/DDIC objects, the actual message number/text must be created in `cz.imcg.fast.ovysledovka` (SE91, message class `ZFI_OV_KBC`) as part of implementation — this spec documents the *intent*, not the final number, in case `009` is already taken.

### 4.4 Handle the "zero packages for a tag at all" edge case

Note that `lt_packages` is built from a `SELECT` against `zfi_i_alloc_extract_p` filtered `WHERE semantictag IN @mt_semtag_filter`. If a filtered semantic tag has **zero packets** (not just zero rows within an existing packet), it will **never appear** in the `GROUP BY` at all — so the per-tag zero-row warning above (4.2) will silently never fire for that tag, since the loop never iterates over it.

To cover this case, add one more check **before** the main `LOOP AT lt_packages ... GROUP BY` (right after the packages are selected, ~line 175 area):
- For each value in `mt_semtag_filter` (the requested/filtered semantic tags), check if it exists at least once in `lt_packages` (e.g. via `LINE_EXISTS` / a helper range check).
- If a filtered semantic tag is completely absent from `lt_packages`, log a warning explicitly, e.g.:
  `iv_message_v1 = 'Semantic tag <X>' `, `iv_message_v2 = 'no packages found — nothing to export'`, `iv_severity = 'W'`.

This directly targets the reported symptom: a semantic tag with **no relevant data at all** produces no packages, no rows, and (today) no log signal explaining why its file is empty.

> Implementation note: `mt_semtag_filter` is a `RANGES` table (`ty_r_fins_sem_tag`), so iterating "requested tags" is only straightforward when the filter consists of simple `EQ` entries. If the filter can contain complex ranges (`BT`, `CP`, etc.), this per-requested-tag check may need to be scoped down to just logging the **set of distinct semantic tags actually present** in `lt_packages` vs. documenting that the filter itself may not resolve to a discrete list. Confirm actual filter usage patterns (how `mt_semtag_filter` is populated by callers, e.g. `ZCL_FI_ALLOC_STEP_EXTRACT`) before implementing this part — flag as an open question if filters are non-trivial.

### 4.5 Do not change `SET_COUNT_ONLY` behavior

`get_data` already supports `mv_count_only` for `SELECT COUNT(*)`-only diagnostics without loading data. This existing capability is unrelated to the new logging requirement (which must work in the normal, non-count-only extraction flow) and should not be touched.

## 5. Constitution Compliance

- **Principle II (SAP Standards):** New/changed lines must stay ≤120 chars; keep existing ABAP Doc/comment style in the class.
- **Principle III (Consult SAP Docs):** No new SAP API pattern introduced (reuses existing `ZIF_FI_PROCESS_LOGGER->message` pattern already used throughout the class) — no additional consultation needed.
- **Principle V (Error Handling):** This is logging-only, not error handling — do not raise `ZCX_FI_OV_KEBOOLA_EXTRACTOR` for a zero-row tag; it is a valid (if noteworthy) business scenario, not an error. Use `iv_severity = 'W'` (warning), not `'E'`.
- **Principle I (DDIC-First):** No new DDIC objects required — reuse `fins_sem_tag`, existing message class `ZFI_OV_KBC`. If a new message number is added, that is a message-class text maintenance activity (SE91), not a new DDIC type.

## 6. Verification Plan

1. **Unit/manual test with a semantic tag known to have data:** Confirm a new `S`-severity summary log line appears per tag with the correct total row count (cross-check against manual `SELECT COUNT(*)` on `zfi_i_alloc_extract_pd` for that tag/period).
2. **Unit/manual test with a semantic tag filtered in but with zero matching packets:** Confirm the new `W`-severity "no packages found" message appears (4.4 case).
3. **Unit/manual test with a semantic tag that has packets but all packets happen to resolve to zero detail rows (aggregation edge case):** Confirm the `W`-severity "0 rows/documents" message appears (4.2 case) — this is the scenario most likely matching the reported empty-gz-file bug.
4. **Regression check:** Confirm `mv_total_rows` (grand total, used in `send_finished_to_appl_server`/`send_finished_to_ftp` trailing file text `"Total rows: <n>"`) is unaffected and still correct after adding the per-tag accumulator.
5. **AL11/FTP check:** Correlate the new per-tag log output against actual file sizes on the FTP/AppL server for the next production run to confirm which semantic tag(s) produce the empty file.

## 7. Out of Scope

- Changing extraction/aggregation SQL logic.
- Changing file naming, zipping, or FTP transfer behavior.
- Adding UI/dashboard visibility for these counts (this is an application-log-only change, viewable via SLG1/transaction log tools).
