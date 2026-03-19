# EST-101: Data Migration Note — BAL_OBJECT / BAL_SUBOBJECT in ZFI_PROC_TYPE

**Story**: EST-101 — Configurable SLG1 Object and Subobject per Process Type  
**Date**: 2026-03-13  
**Author**: Zdenek Smolik

---

## Context

The `ZFI_PROC_TYPE` table has been extended with two new `NOTNULL` fields:

| Field         | Data Element             | Domain      | Default |
|---------------|--------------------------|-------------|---------|
| `BAL_OBJECT`  | `ZFI_PROCESS_BAL_OBJECT` | `BALOBJ_D`  | (none)  |
| `BAL_SUBOBJECT` | `ZFI_PROCESS_BAL_SUBOBJ` | `BALSUBOBJ` | (none)  |

Because `ZFI_PROC_TYPE` has delivery class **A** (application data), existing rows are **not** updated automatically on transport import. The new fields will be blank (`''`) in all existing rows after the transport is applied.

## Risk

A blank `BAL_OBJECT` will cause `load_bal_config()` to return empty values, leading to logger creation with an undefined object — which may silently fail or produce misleading SLG1 entries.

## Required Manual Action After Transport Import

For **every existing row** in `ZFI_PROC_TYPE`, set the two new fields via SE16 / SM30 or a one-time migration report.

### Standard production process types

| PROCESS_TYPE | BAL_OBJECT    | BAL_SUBOBJECT |
|--------------|---------------|---------------|
| `ZFI_ALLOC`  | `ZFI_PROCESS` | `CORE`        |
| _(any other production type)_ | `ZFI_PROCESS` | `CORE` |

> Adjust `BAL_OBJECT` / `BAL_SUBOBJECT` per business requirements if a process type should log to a dedicated SLG0 object.

### Health check test process types (`TEST_%`)

These are managed by `ZCL_FIPROC_HEALTH_CHK_QUERY=>ensure_test_type_defs()` which now includes `BAL_OBJECT = 'ZFI_PROCESS'` and `BAL_SUBOBJECT = 'TESTS'` in every MODIFY statement.  
They will be **updated automatically** the next time the health check runs — no manual action required.

## Validation

After setting values, verify in SLG0 (transaction) that the configured `BAL_OBJECT` value exists.  
`ZCL_FI_PROCESS_MANAGER=>create_process` will raise `ZCX_FI_PROCESS_ERROR` with message  
`"BAL object '...' not found in SLG0 (BALOBJCT)"` if the value is invalid, so any misconfiguration will surface immediately on first use.

## SQL Helper (SAP HANA)

```sql
UPDATE ZFI_PROC_TYPE
SET BAL_OBJECT = 'ZFI_PROCESS',
    BAL_SUBOBJECT = 'CORE'
WHERE BAL_OBJECT = ''
  AND PROCESS_TYPE NOT LIKE 'TEST_%';
```

> Run in SE14 / HANA Studio on non-productive systems only. For productive systems, use a proper ABAP correction program or SM30.
