---
title: 'Process Instance Insight — Parameter Projection & Business Status'
slug: 'story-process-insight-params-bs'
created: '2026-03-04'
status: 'ready-for-dev'
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
tech_stack:
  - 'ABAP 7.58'
  - 'SAP DDIC'
  - 'RAP / CDS OData V4'
  - 'Fiori Elements'
  - 'abapGit XML'
files_to_modify:
  - 'src/zfi_proc_inst.tabl.xml'
  - 'src/zfi_proc_def.tabl.xml'
  - 'src/zcl_fi_process_instance.clas.abap'
  - 'src/zcl_fi_process_definition.clas.abap'
  - 'src/zfiproc_r_instance_tp.ddls.asddls'
  - 'src/zfiproc_c_instance_tp.ddls.asddls'
  - 'src/zfiproc_c_instance_tp.ddlx.asddlxs'
  - 'src/zfiproc_r_instance_tp.bdef.asbdef'
  - 'src/zfi_setup_demo_data.prog.abap'
---

# Story: Process Instance Insight — Parameter Projection & Business Status

**Status:** ready-for-dev
**Source:** brainstorming-session-2026-03-04.md
**Implements:** Two features shipped as one story (shared DDIC activation cycle)

---

## Story

As a process monitor user,
I want to see the four financial input parameters (allocation ID, company code, fiscal period,
fiscal year) as filterable columns in the process instance list report,
and I want to see live business status indicators (e.g., "NOT AVAILABLE / IN PREPARATION")
that describe where the financial data currently is — independently of the technical process status —
so that I can find the right process instance quickly and understand data availability at a glance.

---

## Acceptance Criteria

**Parameter Projection (AC-P)**

- AC-P1: When a new process instance is created for a process type that has a
  `PARAMETER_STRUCTURE` defined in `ZFI_PROC_TYPE`, the fields `PARAM_VAL_1..5` on
  `ZFI_PROC_INST` are populated with the serialized string values of the first five DDIC
  structure fields (in DDIC field-definition order, not alphabetical), and `PARAM_LABEL_1..5`
  are populated with the `SCRTEXT_L` (40-char) label of each field from the DDIC data element,
  resolved in the logon language (`SY-LANGU`) of the creating user.
- AC-P2: If the process type has no `PARAMETER_STRUCTURE`, or if `PARAMETER_DATA` is empty,
  `PARAM_VAL_1..5` and `PARAM_LABEL_1..5` remain empty — no error is raised.
- AC-P3: If the structure has fewer than five fields, only the populated slots carry values;
  the remaining slots are empty. The list report always shows all five columns.
- AC-P4: `PARAM_VAL_n` values are truncated to 20 characters if the serialized value exceeds
  this length. `PARAM_LABEL_n` values are truncated to 40 characters.
- AC-P5: The existing `PARAMETER_DATA` blob is unchanged; `PARAM_VAL/LABEL` columns are
  additive projections populated at creation time only.
- AC-P6: All five `ParamVal1..5` columns appear in the Fiori list report and as selection
  fields (smart filter bar). The process-type filter is mandatory (enforced via annotation).
- AC-P7: `ParamLabel_n` is wired as the text element for `ParamVal_n` in the consumption view
  (`@ObjectModel.text.element`), so the column header adapts to the process type's parameter
  names.

**Business Status (AC-B)**

- AC-B1: When a step whose definition has a non-empty `INIT_BS1` (or `INIT_BS2`) begins
  executing (step header status transitions to `RUNNING` for serial steps, or `QUEUED` for
  queued steps), the corresponding `BUSINESS_STATUS_1` (or `BUSINESS_STATUS_2`) on
  `ZFI_PROC_INST` is set to that config value and committed to the database immediately (before
  the step's `execute()` method is called).
- AC-B2: When a step completes successfully (status → COMPLETED), `BUSINESS_STATUS_1/2` are
  updated with `FINAL_BS1/2` from the step config and committed in the same `COMMIT WORK AND
  WAIT` as the COMPLETED status update. If a step's `FINAL_BS1/2` config is empty, the
  corresponding live field is cleared to empty.
- AC-B3: When a step fails (status → FAILED), `BUSINESS_STATUS_1/2` are updated with
  `FAIL_BS1/2` from the step config and committed in the same commit as the FAILED status. If
  `FAIL_BS1/2` is empty, the field is cleared. `FINAL_BS1/2` is NOT applied on failure.
- AC-B4: When a step is restarted, `INIT_BS1/2` of the restarted step is applied cleanly,
  overwriting the previous value with no special logic.
- AC-B5: A step class can call `io_process_instance->set_business_status( iv_bs1, iv_bs2 )`
  during its `execute()` method. The values are buffered. When the step completes, the buffered
  values override `FINAL_BS1/2` from config. On failure, the buffer is ignored and config
  `FAIL_BS1/2` is applied.
- AC-B6: `BUSINESS_STATUS_1/2` appear in the Fiori list report (as columns and filter fields)
  and in the object page header (identification section, above the step facet).

**Demo Data (AC-D)**

- AC-D1: `ZFI_SETUP_DEMO_DATA` creates a new process type `DEMO_FINANCIAL_PROCESS` with
  `PARAMETER_STRUCTURE = 'ZFI_PROCESS_S_FIN_PARAMS'` and four step definitions with full BS
  config (`INIT_BS1`, `FINAL_BS1`, `FAIL_BS1` populated).
- AC-D2: `ZFI_SETUP_DEMO_DATA` adds `INIT_BS1`, `FINAL_BS1`, `FAIL_BS1` config values to all
  five steps of the existing `TEST_PROCESS` process type.
- AC-D3: Running `ZFI_PROCESS_FRAMEWORK` with `DEMO_FINANCIAL_PROCESS` and test init params
  results in `PARAM_VAL_1..4` populated on the created instance row in `ZFI_PROC_INST`.
- AC-D4: After the `DEMO_FINANCIAL_PROCESS` instance runs to completion, `BUSINESS_STATUS_1`
  reflects the final step's `FINAL_BS1` value.

---

## Scope

**In Scope:**
- 12 new columns on `ZFI_PROC_INST` (5 param values + 5 param labels + 2 business statuses)
- 6 new config columns on `ZFI_PROC_DEF` (INIT/FINAL/FAIL × BS1/BS2)
- 3 new DDIC domains + 3 new DDIC data elements
- 1 new DDIC structure `ZFI_PROCESS_S_FIN_PARAMS` (4 fields, for demo/test)
- Framework: populate flat param columns at instance creation
- Framework: write BS at step RUNNING/QUEUED → COMPLETED/FAILED transitions
- Framework: `set_business_status()` public API for programmatic override
- CDS root + consumption view + metadata extension + BDEF updates
- `ZFI_SETUP_DEMO_DATA` extended with BS config and new `DEMO_FINANCIAL_PROCESS` type

**Out of Scope:**
- `ADD_PARAMETER_DATA` is not projected to flat columns (deferred)
- No migration of historical `ZFI_PROC_INST` rows — historical data is deleted; clean cut-over
- No fixed domain values on `BUSINESS_STATUS_1/2` (free-text per design — see Dev Notes)
- No substep-level business status
- No OData write actions for BS (read-only from Fiori)
- No new OData service — existing `ZFIPROC_UI_INSTANCE_O4` is extended

---

## New DDIC Objects (create as new files in src/)

### Domains

| File | Domain Name | Type | Length | Description |
|------|-------------|------|--------|-------------|
| `zfi_process_param_value.doma.xml` | `ZFI_PROCESS_PARAM_VALUE` | CHAR | 20 | Short parameter value |
| `zfi_process_param_label.doma.xml` | `ZFI_PROCESS_PARAM_LABEL` | CHAR | 40 | Parameter field label |
| `zfi_process_bus_status.doma.xml`  | `ZFI_PROCESS_BUS_STATUS`  | CHAR | 60 | Business status descriptor |

All three domains: no fixed values, no conversion routine, case-sensitive = false.

**`zfi_process_param_value.doma.xml`:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<abapGit version="v1.0.0" serializer="LCL_OBJECT_DOMA" serializer_version="v1.0.0">
 <asx:abap xmlns:asx="http://www.sap.com/abapxml" version="1.0">
  <asx:values>
   <DD01V>
    <DOMNAME>ZFI_PROCESS_PARAM_VALUE</DOMNAME>
    <DDLANGUAGE>E</DDLANGUAGE>
    <DATATYPE>CHAR</DATATYPE>
    <LENG>0000000020</LENG>
    <OUTPUTLEN>0000000020</OUTPUTLEN>
    <DDTEXT>Parameter Value (max 20 chars)</DDTEXT>
   </DD01V>
  </asx:values>
 </asx:abap>
</abapGit>
```

**`zfi_process_param_label.doma.xml`:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<abapGit version="v1.0.0" serializer="LCL_OBJECT_DOMA" serializer_version="v1.0.0">
 <asx:abap xmlns:asx="http://www.sap.com/abapxml" version="1.0">
  <asx:values>
   <DD01V>
    <DOMNAME>ZFI_PROCESS_PARAM_LABEL</DOMNAME>
    <DDLANGUAGE>E</DDLANGUAGE>
    <DATATYPE>CHAR</DATATYPE>
    <LENG>0000000040</LENG>
    <OUTPUTLEN>0000000040</OUTPUTLEN>
    <DDTEXT>Parameter Field Label (SCRTEXT_L, 40 chars)</DDTEXT>
   </DD01V>
  </asx:values>
 </asx:abap>
</abapGit>
```

**`zfi_process_bus_status.doma.xml`:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<abapGit version="v1.0.0" serializer="LCL_OBJECT_DOMA" serializer_version="v1.0.0">
 <asx:abap xmlns:asx="http://www.sap.com/abapxml" version="1.0">
  <asx:values>
   <DD01V>
    <DOMNAME>ZFI_PROCESS_BUS_STATUS</DOMNAME>
    <DDLANGUAGE>E</DDLANGUAGE>
    <DATATYPE>CHAR</DATATYPE>
    <LENG>0000000060</LENG>
    <OUTPUTLEN>0000000060</OUTPUTLEN>
    <DDTEXT>Business Status Descriptor (free text)</DDTEXT>
   </DD01V>
  </asx:values>
 </asx:abap>
</abapGit>
```

### Data Elements

| File | Dtel Name | Domain | Short | Medium | Long |
|------|-----------|--------|-------|--------|------|
| `zfi_process_param_value.dtel.xml` | `ZFI_PROCESS_PARAM_VALUE` | `ZFI_PROCESS_PARAM_VALUE` | `Param Val` | `Parameter Value` | `Parameter Value` |
| `zfi_process_param_label.dtel.xml` | `ZFI_PROCESS_PARAM_LABEL` | `ZFI_PROCESS_PARAM_LABEL` | `Param Label` | `Parameter Label` | `Parameter Label` |
| `zfi_process_bus_status.dtel.xml`  | `ZFI_PROCESS_BUS_STATUS`  | `ZFI_PROCESS_BUS_STATUS`  | `Bus. Status` | `Business Status` | `Business Status` |

Use the same abapGit `LCL_OBJECT_DTEL` format as existing dtel.xml files in src/ (DD04V record
with ROLLNAME, DOMNAME, DDTEXT, SCRTEXT_S/M/L, REPTEXT, COMPTYPE=E, DDLANGUAGE=E).

### Structure (new file in src/)

**`zfi_process_s_fin_params.tabl.xml`** — TABCLASS=INTTAB (internal structure, no DB table):

| Position | Field Name | Data Element | Notes |
|----------|------------|--------------|-------|
| 1 | `ALLOCATION_ID` | `ZFI_PROCESS_PARAM_VALUE` | Dataset tenant / allocation identifier |
| 2 | `COMPANY_CODE`  | `BUKRS` (SAP standard) | Company code |
| 3 | `FISCAL_PERIOD` | `MONAT` (SAP standard) | Fiscal period (01–12) |
| 4 | `FISCAL_YEAR`   | `GJAHR` (SAP standard) | Fiscal year (4 digits) |

```xml
<?xml version="1.0" encoding="utf-8"?>
<abapGit version="v1.0.0" serializer="LCL_OBJECT_TABL" serializer_version="v1.0.0">
 <asx:abap xmlns:asx="http://www.sap.com/abapxml" version="1.0">
  <asx:values>
   <DD02V>
    <TABNAME>ZFI_PROCESS_S_FIN_PARAMS</TABNAME>
    <DDLANGUAGE>E</DDLANGUAGE>
    <TABCLASS>INTTAB</TABCLASS>
    <DDTEXT>Financial Process Parameters</DDTEXT>
    <EXCLASS>1</EXCLASS>
   </DD02V>
   <DD03P_TABLE>
    <DD03P>
     <FIELDNAME>ALLOCATION_ID</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_PARAM_VALUE</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>COMPANY_CODE</FIELDNAME>
     <ROLLNAME>BUKRS</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>FISCAL_PERIOD</FIELDNAME>
     <ROLLNAME>MONAT</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>FISCAL_YEAR</FIELDNAME>
     <ROLLNAME>GJAHR</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
   </DD03P_TABLE>
  </asx:values>
 </asx:abap>
</abapGit>
```

---

## Modified DDIC Tables

### `src/zfi_proc_inst.tabl.xml` — Add 12 new DD03P blocks

Append the following 12 `<DD03P>` blocks inside `<DD03P_TABLE>`, after the existing
`ADD_PARAMETER_DATA` block:

```xml
    <DD03P>
     <FIELDNAME>PARAM_VAL_1</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_PARAM_VALUE</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>PARAM_VAL_2</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_PARAM_VALUE</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>PARAM_VAL_3</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_PARAM_VALUE</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>PARAM_VAL_4</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_PARAM_VALUE</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>PARAM_VAL_5</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_PARAM_VALUE</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>PARAM_LABEL_1</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_PARAM_LABEL</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>PARAM_LABEL_2</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_PARAM_LABEL</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>PARAM_LABEL_3</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_PARAM_LABEL</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>PARAM_LABEL_4</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_PARAM_LABEL</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>PARAM_LABEL_5</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_PARAM_LABEL</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>BUSINESS_STATUS_1</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_BUS_STATUS</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>BUSINESS_STATUS_2</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_BUS_STATUS</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
```

### `src/zfi_proc_def.tabl.xml` — Add 6 new DD03P blocks

Append the following 6 `<DD03P>` blocks after the existing `CONTEXT_DATA_STRUCTURE` block:

```xml
    <DD03P>
     <FIELDNAME>INIT_BS1</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_BUS_STATUS</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>INIT_BS2</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_BUS_STATUS</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>FINAL_BS1</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_BUS_STATUS</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>FINAL_BS2</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_BUS_STATUS</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>FAIL_BS1</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_BUS_STATUS</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
    <DD03P>
     <FIELDNAME>FAIL_BS2</FIELDNAME>
     <ROLLNAME>ZFI_PROCESS_BUS_STATUS</ROLLNAME>
     <ADMINFIELD>0</ADMINFIELD>
     <COMPTYPE>E</COMPTYPE>
    </DD03P>
```

---

## Framework Changes

### `src/zcl_fi_process_instance.clas.abap`

#### 1. New private attributes (add to PRIVATE SECTION DATA block)

```abap
DATA mv_pending_bs1  TYPE zfi_process_bus_status.
DATA mv_pending_bs2  TYPE zfi_process_bus_status.
DATA mv_bs_override  TYPE abap_bool VALUE abap_false.
```

#### 2. New public method `SET_BUSINESS_STATUS`

Add to PUBLIC SECTION METHODS:
```abap
"! Buffer business status override values; applied on next FINAL commit
"! @parameter iv_bs1 | Business status 1 override (empty = clear)
"! @parameter iv_bs2 | Business status 2 override (empty = clear)
METHODS set_business_status
  IMPORTING
    iv_bs1 TYPE zfi_process_bus_status OPTIONAL
    iv_bs2 TYPE zfi_process_bus_status OPTIONAL.
```

Implementation:
```abap
METHOD set_business_status.
  mv_pending_bs1 = iv_bs1.
  mv_pending_bs2 = iv_bs2.
  mv_bs_override = abap_true.
ENDMETHOD.
```

#### 3. New private method `POPULATE_PARAM_COLUMNS`

Add to PRIVATE SECTION METHODS:
```abap
"! Populate PARAM_VAL/LABEL flat columns from serialized parameter data
"! @parameter iv_parameter_data | Serialized key=val parameter string
"! @raising zcx_fi_process_error | DDIC structure not found or type error
METHODS populate_param_columns
  IMPORTING iv_parameter_data TYPE zfi_process_context
  RAISING   zcx_fi_process_error.
```

Implementation algorithm — step by step:
```abap
METHOD populate_param_columns.
  " 1. Get parameter structure name from ZFI_PROC_TYPE
  DATA lv_struct_name TYPE zfi_process_param_struct.
  SELECT SINGLE parameter_structure
    FROM zfi_proc_type
    WHERE mandt        = @sy-mandt
      AND process_type = @ms_instance-process_type
    INTO @lv_struct_name.

  " 2. If no structure defined or no param data, nothing to do
  CHECK lv_struct_name IS NOT INITIAL.
  CHECK iv_parameter_data IS NOT INITIAL.

  " 3. Parse iv_parameter_data (KEY=val;KEY=val) into a field-value map
  "    Reuse same escape/parse logic as get_init_param_value():
  "    split by ';' (respecting '\;' escape), then split each token by '='
  DATA lt_pairs TYPE string_table.
  " ... parse logic (align with get_init_param_value implementation) ...

  DATA lt_map TYPE HASHED TABLE OF
    zfi_process_s_kv WITH UNIQUE KEY name.   " or local string pair if no DDIC type
  " Populate lt_map from lt_pairs

  " 4. Reflect structure components via RTTS
  DATA lo_struct TYPE REF TO cl_abap_structdescr.
  TRY.
      lo_struct = CAST cl_abap_structdescr(
        cl_abap_typedescr=>describe_by_name( lv_struct_name ) ).
    CATCH cx_sy_rtti_syntax_error cx_sy_typedescrref_illtype INTO DATA(lx_rtti).
      RAISE EXCEPTION TYPE zcx_fi_process_error
        EXPORTING
          textid       = zcx_fi_process_error=>configuration_error
          process_type = ms_instance-process_type
          step_number  = '0000'.
  ENDTRY.

  " 5. Loop structure components (up to 5, in definition order)
  DATA lv_idx TYPE i VALUE 0.
  LOOP AT lo_struct->components INTO DATA(ls_comp).
    lv_idx += 1.
    IF lv_idx > 5. EXIT. ENDIF.

    " Get label in sy-langu
    DATA lo_elem TYPE REF TO cl_abap_elemdescr.
    TRY.
        lo_elem = CAST cl_abap_elemdescr(
          lo_struct->get_component_type( ls_comp-name ) ).
        DATA(ls_dfies) = lo_elem->get_ddic_field( p_langu = sy-langu ).
      CATCH cx_root.
        CLEAR ls_dfies.
    ENDTRY.

    " Look up value in parsed map
    DATA lv_val TYPE string.
    READ TABLE lt_map INTO DATA(ls_pair)
      WITH TABLE KEY name = ls_comp-name.
    IF sy-subrc = 0.
      lv_val = ls_pair-value.
    ENDIF.

    " Assign to ms_instance slots (truncate to domain length)
    CASE lv_idx.
      WHEN 1.
        ms_instance-param_val_1   =
          CONV zfi_process_param_value( lv_val ).
        ms_instance-param_label_1 =
          CONV zfi_process_param_label( ls_dfies-scrtext_l ).
      WHEN 2.
        ms_instance-param_val_2   =
          CONV zfi_process_param_value( lv_val ).
        ms_instance-param_label_2 =
          CONV zfi_process_param_label( ls_dfies-scrtext_l ).
      WHEN 3.
        ms_instance-param_val_3   =
          CONV zfi_process_param_value( lv_val ).
        ms_instance-param_label_3 =
          CONV zfi_process_param_label( ls_dfies-scrtext_l ).
      WHEN 4.
        ms_instance-param_val_4   =
          CONV zfi_process_param_value( lv_val ).
        ms_instance-param_label_4 =
          CONV zfi_process_param_label( ls_dfies-scrtext_l ).
      WHEN 5.
        ms_instance-param_val_5   =
          CONV zfi_process_param_value( lv_val ).
        ms_instance-param_label_5 =
          CONV zfi_process_param_label( ls_dfies-scrtext_l ).
    ENDCASE.
  ENDLOOP.
ENDMETHOD.
```

> **Note on key-value map type**: If no suitable DDIC structure exists for string pairs, use
> two parallel standard tables or a local-only helper. The parse logic must mirror
> `get_init_param_value()` including `\=` and `\;` escape sequences.

#### 4. Modify `initialize_instance()` — add param column population

Locate the line that assigns `ms_instance-parameter_data = iv_parameter_data` (or where the
PARAMETER_DATA field is set on `ms_instance`) and add immediately after it:

```abap
" Populate flat param columns for display and filtering
populate_param_columns( iv_parameter_data ).
```

This call populates `ms_instance-param_val_1..5` and `ms_instance-param_label_1..5` before
`save_instance()` is called. No separate commit — the values go with the initial INSERT.

#### 5. New private methods `WRITE_INIT_BS`, `WRITE_FINAL_BS`, `WRITE_FAIL_BS`

**Method signatures (add to PRIVATE SECTION):**
```abap
"! Write INIT_BS1/2 to instance header from step config; commits immediately
"! @parameter iv_step_number | Step number of the step that just started
"! @raising zcx_fi_process_error | Step definition not found
METHODS write_init_bs
  IMPORTING iv_step_number TYPE zfi_process_step_number
  RAISING   zcx_fi_process_error.

"! Write FINAL_BS1/2; applies programmatic override if set; resets override flag
"! @parameter iv_step_number | Step number of the step that just completed
"! @raising zcx_fi_process_error | Step definition not found
METHODS write_final_bs
  IMPORTING iv_step_number TYPE zfi_process_step_number
  RAISING   zcx_fi_process_error.

"! Write FAIL_BS1/2 on failure path; ignores programmatic override
"! @parameter iv_step_number | Step number of the step that just failed
"! @raising zcx_fi_process_error | Step definition not found
METHODS write_fail_bs
  IMPORTING iv_step_number TYPE zfi_process_step_number
  RAISING   zcx_fi_process_error.
```

**`write_init_bs` implementation:**
```abap
METHOD write_init_bs.
  " Read step definition for BS config
  DATA ls_def TYPE zfi_proc_def.
  SELECT SINGLE init_bs1, init_bs2
    FROM zfi_proc_def
    WHERE mandt        = @sy-mandt
      AND process_type = @ms_instance-process_type
      AND step_number  = @iv_step_number
    INTO CORRESPONDING FIELDS OF @ls_def.

  " Apply values (empty config = clear the field)
  ms_instance-business_status_1 = ls_def-init_bs1.
  ms_instance-business_status_2 = ls_def-init_bs2.

  " Commit immediately (INIT_BS is written after step goes RUNNING/QUEUED in DB)
  MODIFY zfi_proc_inst FROM ms_instance.
  COMMIT WORK AND WAIT.
ENDMETHOD.
```

**`write_final_bs` implementation:**
```abap
METHOD write_final_bs.
  DATA ls_def TYPE zfi_proc_def.
  SELECT SINGLE final_bs1, final_bs2
    FROM zfi_proc_def
    WHERE mandt        = @sy-mandt
      AND process_type = @ms_instance-process_type
      AND step_number  = @iv_step_number
    INTO CORRESPONDING FIELDS OF @ls_def.

  IF mv_bs_override = abap_true.
    " Programmatic override wins over config
    ms_instance-business_status_1 = mv_pending_bs1.
    ms_instance-business_status_2 = mv_pending_bs2.
    " Reset override for next step
    CLEAR mv_pending_bs1.
    CLEAR mv_pending_bs2.
    mv_bs_override = abap_false.
  ELSE.
    ms_instance-business_status_1 = ls_def-final_bs1.
    ms_instance-business_status_2 = ls_def-final_bs2.
  ENDIF.
  " Caller (execute_step) commits via save_instance() — no separate commit here
ENDMETHOD.
```

**`write_fail_bs` implementation:**
```abap
METHOD write_fail_bs.
  DATA ls_def TYPE zfi_proc_def.
  SELECT SINGLE fail_bs1, fail_bs2
    FROM zfi_proc_def
    WHERE mandt        = @sy-mandt
      AND process_type = @ms_instance-process_type
      AND step_number  = @iv_step_number
    INTO CORRESPONDING FIELDS OF @ls_def.

  " Ignore any programmatic override — config FAIL values take precedence
  ms_instance-business_status_1 = ls_def-fail_bs1.
  ms_instance-business_status_2 = ls_def-fail_bs2.
  " Reset override flag so it does not bleed into the next step after restart
  CLEAR mv_pending_bs1.
  CLEAR mv_pending_bs2.
  mv_bs_override = abap_false.
  " Caller (execute_step failure path) commits via save_failure_state() — no separate commit
ENDMETHOD.
```

#### 6. Modify `execute_step()` — wire BS write calls

In `execute_step()`, locate the call to `update_step_status()` that sets the status to
`RUNNING` (serial) or `QUEUED` (queued mode). Immediately after that call (after the step row
commit), add:

```abap
" Write INIT business status after step begins executing
write_init_bs( iv_step_number ).
```

Locate where the step is marked COMPLETED (just before or instead of `save_instance()` in the
success path). Add before `save_instance()`:

```abap
" Write FINAL business status with same commit as COMPLETED
write_final_bs( iv_step_number ).
```

Locate the failure path where the step goes FAILED (before `save_failure_state()`). Add:

```abap
" Write FAIL business status with same commit as FAILED
write_fail_bs( iv_step_number ).
```

> **Note on commit boundaries**: `write_init_bs()` performs its own `MODIFY + COMMIT` because
> INIT_BS must be persisted before the step's execute() begins. `write_final_bs()` and
> `write_fail_bs()` do NOT commit — they only modify `ms_instance`; the commit is performed by
> the subsequent `save_instance()` or `save_failure_state()` call, ensuring the BS and step
> COMPLETED/FAILED status share one atomic commit.

### `src/zcl_fi_process_definition.clas.abap`

Inspect how this class reads from `ZFI_PROC_DEF`. If the SELECT uses an explicit field list
(e.g., `SELECT class_name, mandatory, ... INTO ...`), add the six new fields:
`init_bs1, init_bs2, final_bs1, final_bs2, fail_bs1, fail_bs2` to that list, and extend the
corresponding internal structure to hold them. If the SELECT uses `*` or
`INTO CORRESPONDING FIELDS OF`, no change is needed on the SELECT itself, but verify the target
structure (`ZFI_PROC_DEF` or a local helper) will include the new columns after DDIC activation.

---

## CDS / RAP Changes

### `src/zfiproc_r_instance_tp.ddls.asddls` — Add 12 fields to SELECT list

After `parameter_data as ParameterData,` add:

```abap
      param_val_1           as ParamVal1,
      param_val_2           as ParamVal2,
      param_val_3           as ParamVal3,
      param_val_4           as ParamVal4,
      param_val_5           as ParamVal5,
      param_label_1         as ParamLabel1,
      param_label_2         as ParamLabel2,
      param_label_3         as ParamLabel3,
      param_label_4         as ParamLabel4,
      param_label_5         as ParamLabel5,
      business_status_1     as BusinessStatus1,
      business_status_2     as BusinessStatus2,
```

### `src/zfiproc_c_instance_tp.ddls.asddls` — Add 12 fields with annotations

After `ParameterData,` add:

```abap
      @EndUserText.label: 'Process Type Filter'
      @Consumption.filter.mandatory: true
      ProcessType,
```

> **Note**: `ProcessType` is already in the view. Move its existing annotation block here (or
> add `@Consumption.filter.mandatory: true` to the existing ProcessType annotation block) to
> enforce mandatory process-type filter at OData level.

Add after `ParameterData`:

```abap
      @ObjectModel.text.element: ['ParamLabel1']
      ParamVal1,
      ParamLabel1,
      @ObjectModel.text.element: ['ParamLabel2']
      ParamVal2,
      ParamLabel2,
      @ObjectModel.text.element: ['ParamLabel3']
      ParamVal3,
      ParamLabel3,
      @ObjectModel.text.element: ['ParamLabel4']
      ParamVal4,
      ParamLabel4,
      @ObjectModel.text.element: ['ParamLabel5']
      ParamVal5,
      ParamLabel5,
      BusinessStatus1,
      BusinessStatus2,
```

### `src/zfiproc_c_instance_tp.ddlx.asddlxs` — Add UI annotations

**Add `@UI.selectionField` to ProcessType annotation block** (it already exists — just add the
mandatory filter reinforcement via annotation if not already present; the `@Consumption`
annotation on the C view handles the mandatory enforcement).

**Add new annotation blocks** for the new fields:

```abap
  @UI: {
        lineItem:       [ { position: 25, importance: #HIGH } ],
        identification: [ { position: 25 } ]
        }
  @UI: { selectionField: [ { position: 30 } ]}
  BusinessStatus1;

  @UI: {
        lineItem:       [ { position: 26, importance: #MEDIUM } ],
        identification: [ { position: 26 } ]
        }
  @UI: { selectionField: [ { position: 31 } ]}
  BusinessStatus2;

  @UI: {
        lineItem:       [ { position: 100, importance: #HIGH } ],
        identification: [ { position: 110 } ]
        }
  @UI: { selectionField: [ { position: 100 } ]}
  ParamVal1;

  @UI: {
        lineItem:       [ { position: 110, importance: #HIGH } ],
        identification: [ { position: 120 } ]
        }
  @UI: { selectionField: [ { position: 110 } ]}
  ParamVal2;

  @UI: {
        lineItem:       [ { position: 120, importance: #HIGH } ],
        identification: [ { position: 130 } ]
        }
  @UI: { selectionField: [ { position: 120 } ]}
  ParamVal3;

  @UI: {
        lineItem:       [ { position: 130, importance: #HIGH } ],
        identification: [ { position: 140 } ]
        }
  @UI: { selectionField: [ { position: 130 } ]}
  ParamVal4;

  @UI: {
        lineItem:       [ { position: 140, importance: #HIGH } ],
        identification: [ { position: 150 } ]
        }
  @UI: { selectionField: [ { position: 140 } ]}
  ParamVal5;
```

> The `ParamLabel1..5` fields are not annotated with `@UI.lineItem` — they serve only as text
> elements for their corresponding `ParamVal_n` fields. They should NOT appear as visible
> columns in the list report.

**Object page header placement**: `BusinessStatus1` and `BusinessStatus2` at
`identification: position: 25/26` place them in the `#IDENTIFICATION_REFERENCE` facet above
the step list. Adjust positions if they conflict with existing fields.

### `src/zfiproc_r_instance_tp.bdef.asbdef` — Add new fields to readonly list

In the `field ( readonly )` block for `ZFIPROC_R_INSTANCE_TP`, add:

```abap
  ParamVal1,
  ParamVal2,
  ParamVal3,
  ParamVal4,
  ParamVal5,
  ParamLabel1,
  ParamLabel2,
  ParamLabel3,
  ParamLabel4,
  ParamLabel5,
  BusinessStatus1,
  BusinessStatus2,
```

---

## Demo Data Changes — `src/zfi_setup_demo_data.prog.abap`

### 1. New process type `DEMO_FINANCIAL_PROCESS`

In `PERFORM create_process_types`, add:

```abap
  ls_type-process_type        = 'DEMO_FINANCIAL'.
  ls_type-description         = 'Demo: Financial Reporting Preparation'.
  ls_type-parameter_structure = 'ZFI_PROCESS_S_FIN_PARAMS'.
  ls_type-active              = abap_true.
  MODIFY zfi_proc_type FROM ls_type.
```

> **Note**: `DEMO_FINANCIAL` is 14 chars — within CHAR 20 domain limit.

### 2. Step definitions for `DEMO_FINANCIAL_PROCESS`

In `PERFORM create_process_definitions`, add four steps. Use the same pattern as existing
steps. Populate BS config fields on each `ls_def` before `MODIFY`:

| Step | Class | INIT_BS1 | FINAL_BS1 | FAIL_BS1 |
|------|-------|----------|-----------|----------|
| 0010 VALIDATE / ZCL_FI_STEP_VALIDATE_INPUT | SERIAL | `NOT AVAILABLE / VALIDATING` | `NOT AVAILABLE / VALIDATED` | `NOT AVAILABLE / VALIDATION FAILED` |
| 0020 FETCH / ZCL_FI_STEP_FETCH_DATA | SERIAL | `NOT AVAILABLE / FETCHING DATA` | `NOT AVAILABLE / DATA FETCHED` | `NOT AVAILABLE / FETCH FAILED` |
| 0030 PROCESS / ZCL_FI_STEP_PROCESS_DATA | SERIAL | `NOT AVAILABLE / IN PREPARATION` | `NOT AVAILABLE / PREPARED` | `NOT AVAILABLE / PREPARATION FAILED` |
| 0040 SAVE / ZCL_FI_STEP_SAVE_RESULTS | SERIAL | `NOT AVAILABLE / SAVING` | `AVAILABLE FOR REPORTING` | `NOT AVAILABLE / SAVE FAILED` |

All steps: `substep_mode = 'SERIAL'`, `mandatory = abap_true`, `active = abap_true`.

### 3. Add BS config to existing `TEST_PROCESS` steps

In the `TEST_PROCESS` step definition block, add BS config columns to all five steps:

| Step | INIT_BS1 | FINAL_BS1 | FAIL_BS1 |
|------|----------|-----------|----------|
| 0001 VALIDATE | `INITIALIZING` | `READY` | `VALIDATION ERROR` |
| 0002 FETCH | `FETCHING` | `DATA LOADED` | `FETCH ERROR` |
| 0003 PROCESS | `PROCESSING` | `PROCESSED` | `PROCESS ERROR` |
| 0004 SAVE | `SAVING` | `SAVED` | `SAVE ERROR` |
| 0005 NOTIFY | `NOTIFYING` | `COMPLETE` | `NOTIFY ERROR` |

### 4. Create a test instance in `ZFI_PROCESS_FRAMEWORK` (optional demo test)

In `ZFI_PROCESS_FRAMEWORK`, add a test path that creates an instance of `DEMO_FINANCIAL` with
init params `ALLOCATION_ID=ALLOC001;COMPANY_CODE=1000;FISCAL_PERIOD=03;FISCAL_YEAR=2026`, runs
it, and prints `PARAM_VAL_1..4` and `BUSINESS_STATUS_1` from the resulting `ZFI_PROC_INST` row
to verify both features work end to end.

---

## Tasks / Subtasks (dependency order — MUST follow this sequence)

- [ ] **Task 1: Create new DDIC domains** (AC-P1, P4, B1–B3)
  - [ ] 1a: Create `src/zfi_process_param_value.doma.xml` (CHAR 20)
  - [ ] 1b: Create `src/zfi_process_param_label.doma.xml` (CHAR 40)
  - [ ] 1c: Create `src/zfi_process_bus_status.doma.xml` (CHAR 60)

- [ ] **Task 2: Create new DDIC data elements** (depends on Task 1)
  - [ ] 2a: Create `src/zfi_process_param_value.dtel.xml`
  - [ ] 2b: Create `src/zfi_process_param_label.dtel.xml`
  - [ ] 2c: Create `src/zfi_process_bus_status.dtel.xml`

- [ ] **Task 3: Create new DDIC structure** (depends on Task 2)
  - [ ] 3a: Create `src/zfi_process_s_fin_params.tabl.xml` (INTTAB, 4 fields)

- [ ] **Task 4: Modify `ZFI_PROC_INST` table** (depends on Task 2)
  - [ ] 4a: Add 12 new DD03P blocks to `src/zfi_proc_inst.tabl.xml` (5 PARAM_VAL, 5 PARAM_LABEL, 2 BUSINESS_STATUS)

- [ ] **Task 5: Modify `ZFI_PROC_DEF` table** (depends on Task 2)
  - [ ] 5a: Add 6 new DD03P blocks to `src/zfi_proc_def.tabl.xml` (INIT/FINAL/FAIL × BS1/BS2)

- [ ] **Task 6: Framework — parameter projection** (depends on Tasks 3, 4)
  - [ ] 6a: Add private attributes `mv_pending_bs1/2`, `mv_bs_override` to class declaration
  - [ ] 6b: Implement `POPULATE_PARAM_COLUMNS` private method
  - [ ] 6c: Call `populate_param_columns()` in `initialize_instance()` after PARAMETER_DATA assignment

- [ ] **Task 7: Framework — business status writes** (depends on Tasks 4, 5)
  - [ ] 7a: Add public method `SET_BUSINESS_STATUS` declaration + implementation
  - [ ] 7b: Implement `WRITE_INIT_BS` private method
  - [ ] 7c: Implement `WRITE_FINAL_BS` private method
  - [ ] 7d: Implement `WRITE_FAIL_BS` private method
  - [ ] 7e: Wire `write_init_bs()` call in `execute_step()` after RUNNING/QUEUED transition
  - [ ] 7f: Wire `write_final_bs()` call in `execute_step()` success path (before `save_instance()`)
  - [ ] 7g: Wire `write_fail_bs()` call in `execute_step()` failure path (before `save_failure_state()`)

- [ ] **Task 8: Check `zcl_fi_process_definition`** (depends on Task 5)
  - [ ] 8a: Verify SELECT from `ZFI_PROC_DEF` includes or will naturally pick up new BS config fields; extend if SELECT uses explicit field list

- [ ] **Task 9: Demo data** (depends on Tasks 3, 4, 5)
  - [ ] 9a: Add `DEMO_FINANCIAL` process type to `create_process_types` form
  - [ ] 9b: Add four step definitions for `DEMO_FINANCIAL` with BS config to `create_process_definitions`
  - [ ] 9c: Add BS config columns to all five `TEST_PROCESS` step definitions

- [ ] **Task 10: CDS / RAP extension** (depends on Tasks 4, 5)
  - [ ] 10a: Add 12 new fields to `src/zfiproc_r_instance_tp.ddls.asddls`
  - [ ] 10b: Add 12 new fields with annotations to `src/zfiproc_c_instance_tp.ddls.asddls`
  - [ ] 10c: Add `@Consumption.filter.mandatory: true` to ProcessType in consumption view
  - [ ] 10d: Add `@UI` annotation blocks to `src/zfiproc_c_instance_tp.ddlx.asddlxs`
  - [ ] 10e: Add new fields to `field(readonly)` list in `src/zfiproc_r_instance_tp.bdef.asbdef`

- [ ] **Task 11: End-to-end verification** (depends on Tasks 6–10)
  - [ ] 11a: Run `ZFI_SETUP_DEMO_DATA` and verify `ZFI_PROC_DEF` rows for `DEMO_FINANCIAL` have BS config populated
  - [ ] 11b: Create a `DEMO_FINANCIAL` instance via `ZFI_PROCESS_FRAMEWORK` with sample financial params; verify `PARAM_VAL_1..4` and `PARAM_LABEL_1..4` on the `ZFI_PROC_INST` row
  - [ ] 11c: Execute the instance; verify `BUSINESS_STATUS_1` transitions through INIT → FINAL values per step config

---

## Dev Notes

### Constitution Compliance

- **Principle I (DDIC-First)**: All new fields use new DDIC domains and data elements.
  No local TYPE definitions introduced. `ZFI_PROCESS_S_FIN_PARAMS` is a proper DDIC INTTAB
  structure. New table types are NOT required here since no method signature passes a collection
  of the new fields.
- **Principle II (SAP Standards)**: All new DDIC objects follow `ZFI_PROCESS_*` naming.
  New column names on tables follow SAP FIELDNAME conventions. No name exceeds 30 chars.
- **Principle III (Consult SAP Docs)**: Before implementing `POPULATE_PARAM_COLUMNS`, consult
  mcp-sap-docs for `CL_ABAP_STRUCTDESCR`, `CL_ABAP_ELEMDESCR->GET_DDIC_FIELD()`, and
  `DFIES` structure field names (specifically `SCRTEXT_L`) in ABAP 7.58.
- **Principle IV (Factory Pattern)**: No constructors changed. `SET_BUSINESS_STATUS` is a
  standard instance method, not a constructor. Factory pattern is unaffected.
- **Principle V (Error Handling)**: `POPULATE_PARAM_COLUMNS` raises `ZCX_FI_PROCESS_ERROR`
  with context if RTTI fails. BS write methods are silent on empty config (by design).

### Free-Text Business Status — Constitution Exception

The constitution states "Status fields MUST use domains with fixed values (not free text)".
This rule applies to **technical process status** (e.g., RUNNING, COMPLETED, FAILED) which
drives framework behavior. `BUSINESS_STATUS_1/2` are **business vocabulary fields** — they are
descriptive strings set by process type configuration, not by the framework engine, and they
drive no conditional logic. Using a CHAR domain without fixed values is intentional per the
brainstorming decision and does not violate the spirit of the constitution.

### Param Label Lookup in RTTS

`CL_ABAP_ELEMDESCR->GET_DDIC_FIELD( p_langu )` returns `DFIES`. Use field `SCRTEXT_L` (40 chars)
for label storage. If the data element has no text in `SY-LANGU`, the label falls back to the
field name. Handle `CX_ROOT` exceptions gracefully — use the field name as a fallback label
rather than failing the instance creation.

### Key-Value Parsing Consistency

`POPULATE_PARAM_COLUMNS` must use the same escape/parse logic as `get_init_param_value()`:
- Separator: `;` (escaped as `\;`)
- Assignment: `=` (escaped as `\=`)
- Backslash: `\\`
Parse order: split by unescaped `;`, then split each token at first unescaped `=`.

### `ms_instance` Type Update

After `ZFI_PROC_INST` gains 12 new columns, `ms_instance TYPE zfi_proc_inst` automatically
includes them (it is typed against the DDIC table). No class declaration change is needed for
`ms_instance`. Ensure DDIC is activated before activating the class.

### Historical Data

All existing `ZFI_PROC_INST` rows will have empty `PARAM_VAL/LABEL` and
`BUSINESS_STATUS_1/2` after the DDIC activation. No migration is required — this is the
agreed clean cut-over.

### `DEMO_FINANCIAL` Process Type Key Length

`DEMO_FINANCIAL` = 14 characters. The domain for `PROCESS_TYPE` is `ZFI_PROCESS_PROC_TYPE`.
Verify its CHAR length supports at least 20 characters (expected from existing process type
names like `DEMO_ORDER_PROCESSING` = 21 chars — confirm actual domain length before activating).

### abapGit Activation Order in Target System

MUST activate in this order to avoid DDIC dependency errors:
1. Domains
2. Data elements
3. Structure `ZFI_PROCESS_S_FIN_PARAMS`
4. Tables `ZFI_PROC_INST`, `ZFI_PROC_DEF`
5. ABAP classes
6. CDS views (R → C → DDLX)
7. BDEF

---

## Files Touched

| File | Change Type | Purpose |
|------|-------------|---------|
| `src/zfi_process_param_value.doma.xml` | NEW | Domain CHAR 20 for param values |
| `src/zfi_process_param_label.doma.xml` | NEW | Domain CHAR 40 for param labels |
| `src/zfi_process_bus_status.doma.xml` | NEW | Domain CHAR 60 for business status |
| `src/zfi_process_param_value.dtel.xml` | NEW | Data element for param value |
| `src/zfi_process_param_label.dtel.xml` | NEW | Data element for param label |
| `src/zfi_process_bus_status.dtel.xml` | NEW | Data element for business status |
| `src/zfi_process_s_fin_params.tabl.xml` | NEW | INTTAB structure for financial params demo |
| `src/zfi_proc_inst.tabl.xml` | MODIFY | +12 columns |
| `src/zfi_proc_def.tabl.xml` | MODIFY | +6 BS config columns |
| `src/zcl_fi_process_instance.clas.abap` | MODIFY | Param projection + BS lifecycle |
| `src/zcl_fi_process_definition.clas.abap` | VERIFY/MODIFY | Ensure new DEF fields are accessible |
| `src/zfiproc_r_instance_tp.ddls.asddls` | MODIFY | +12 new CDS fields |
| `src/zfiproc_c_instance_tp.ddls.asddls` | MODIFY | +12 fields with annotations |
| `src/zfiproc_c_instance_tp.ddlx.asddlxs` | MODIFY | +UI annotations for new fields |
| `src/zfiproc_r_instance_tp.bdef.asbdef` | MODIFY | +12 readonly fields |
| `src/zfi_setup_demo_data.prog.abap` | MODIFY | +DEMO_FINANCIAL type/steps + BS config |

---

## Dev Agent Record

### Agent Model Used

github-copilot/claude-sonnet-4.6

### Completion Notes List

### File List
