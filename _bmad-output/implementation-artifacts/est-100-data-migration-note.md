# EST-100: Data Migration Note — BGRFC_DEST_NAME_INBOUND in ZFI_PROC_TYPE

**Story**: EST-100 — Configurable bgRFC Inbound Destination per Process Type  
**Date**: 2026-03-14  
**Author**: Zdenek Smolik

---

## Context

The `ZFI_PROC_TYPE` table has been extended with one new `NOTNULL` field:

| Field                    | Data Element              | Description                          |
|--------------------------|---------------------------|--------------------------------------|
| `BGRFC_DEST_NAME_INBOUND` | `BGRFC_DEST_NAME_INBOUND` | bgRFC inbound destination for queued substep dispatch |

Because `ZFI_PROC_TYPE` has delivery class **A** (application data), existing rows are **not** updated automatically on transport import. The new field will be blank (`''`) in all existing rows after the transport is applied.

## Risk

A blank `BGRFC_DEST_NAME_INBOUND` will cause `initialize_instance()` to immediately raise
`ZCX_FI_PROCESS_ERROR=>bgrfc_destination_not_found` (message 084) — **no new process instances
can be created** for the affected process type until the field is populated.

This is intentional: the system fails loudly and clearly rather than silently dispatching substeps
to an undefined destination.

## Required Manual Action After Transport Import

### Recommended sequence (minimize downtime)

1. **Before importing transport**: Set `ACTIVE = ''` on all production `ZFI_PROC_TYPE` rows to
   prevent new process creation during the migration window. Wait for all in-flight queued processes
   to complete (or drain them manually).
2. **Import transport**: `ZFI_PROC_TYPE` is activated with blank `BGRFC_DEST_NAME_INBOUND` on all
   existing rows.
3. **Immediately after import**: Run the SQL helper or SM30 update below to populate
   `BGRFC_DEST_NAME_INBOUND` on all production rows.
4. **Re-activate**: Set `ACTIVE = 'X'` again on production process types.

### Standard production process types

For **every existing non-test row** in `ZFI_PROC_TYPE`, set the new field via SE16 / SM30 or a
one-time migration report:

| PROCESS_TYPE           | BGRFC_DEST_NAME_INBOUND |
|------------------------|-------------------------|
| `ZFI_ALLOC`            | `ZFI_PROCESS`           |
| _(any other production type)_ | `ZFI_PROCESS` _(or a custom inbound destination if separate queuing is desired)_ |

> Adjust `BGRFC_DEST_NAME_INBOUND` per business requirements if a process type should dispatch
> queued substeps to a dedicated bgRFC inbound destination (separate work queue, separate server
> group, independent load management).

### Health check test process types (`TEST_%`)

These are managed by `ZCL_FIPROC_HEALTH_CHK_QUERY=>ensure_test_type_defs()` which now includes
`bgrfc_dest_name_inbound = 'ZFI_PROCESS'` in every `MODIFY zfi_proc_type` statement.  
They will be **updated automatically** the next time the health check runs — no manual action
required.

## Validation

After setting values, verify the destination exists in the SAP system by running transaction
**SMQR** (qRFC monitor / inbound destinations) or by running a test process creation:

```
ZCL_FI_PROCESS_MANAGER=>create_process( ... )
```

If `BGRFC_DEST_NAME_INBOUND` contains a non-existent destination name, the system will raise
`ZCX_FI_PROCESS_ERROR` with message 084:
> `bgRFC inbound destination '&1' not found — configure in ZFI_PROC_TYPE`

This surfaces the misconfiguration immediately at instance creation time — before the instance is
persisted, and long before any bgRFC dispatch attempt.

## SQL Helper (SAP HANA)

```sql
UPDATE ZFI_PROC_TYPE
SET BGRFC_DEST_NAME_INBOUND = 'ZFI_PROCESS'
WHERE BGRFC_DEST_NAME_INBOUND = ''
  AND PROCESS_TYPE NOT LIKE 'TEST_%';
```

> Run in SE14 / HANA Studio on non-productive systems only. For productive systems, use a proper
> ABAP correction program or SM30.

## In-Flight Instances

Existing process instances already in progress at transport time are safe to **load and query**
via `load_instance()`. They will only fail when attempting to dispatch queued substeps if
`mv_bgrfc_dest_name` is blank (secondary guard at `process_substeps_queued()` line ~1484 raises
`step_execution_failed`).

**Recommendation**: Complete all in-flight queued processes before importing the transport.
