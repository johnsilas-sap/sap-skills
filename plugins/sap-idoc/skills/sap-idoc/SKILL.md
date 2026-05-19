---
name: sap-idoc
description: |
  SAP IDoc (Intermediate Document) development skill. Use when creating custom IDoc
  types, building inbound/outbound processing function modules, configuring partner
  profiles (WE20), working with ALE/EDI distribution models (BD64), handling IDoc
  errors and reprocessing (BD87), extending standard IDocs with Z-segments, or
  monitoring via WE02/WE05. Covers ORDERS, INVOIC, DESADV, SHPMNT, WMMBXY, DELVRY
  and custom message types across EWM, TM, MM, SD, and FI.
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-19"
  sources:
    - "https://help.sap.com/docs/ABAP_PLATFORM_NEW/0b9668e854374d8fa3fc8ec327ff3693"
    - "https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE"
---

# SAP IDoc Development Skill

## Related Skills

- **sap-abap**: General ABAP syntax for FM-based IDoc processors
- **sap-ewm**: EWM-specific IDoc types (WMMBXY, DESADV, WMCATO)
- **sap-tm**: TM-specific IDoc types (SHPMNT, TPSSHT, IFTSTA, IFCSUM)
- **sap-btp-connectivity**: Routing IDocs through BTP Integration Suite / middleware

## IDoc Structure

Every IDoc consists of three record types:

```
Control Record (EDIDC)   — 1 per IDoc: sender, receiver, message type, status
  Data Records (EDIDD)   — 1..n: business data in named segments
    Status Records (EDIDS) — 0..n: processing history and error messages
```

### Control Record Key Fields (EDIDC)

| Field | Description |
|---|---|
| `DOCNUM` | IDoc number (unique) |
| `MESTYP` | Message type (e.g. ORDERS, INVOIC, DESADV) |
| `BASIC_TYPE` | Basic IDoc type (e.g. ORDERS05, INVOIC02) |
| `IDOCTP` | IDoc type (= BASIC_TYPE unless extended) |
| `DIRECT` | Direction: 1 = Outbound, 2 = Inbound |
| `STATUS` | Processing status (01–75) |
| `SNDPRT` | Sender partner type (LS/LI/KU/LF) |
| `SNDPRN` | Sender partner number |
| `RCVPRT` | Receiver partner type |
| `RCVPRN` | Receiver partner number |
| `CREDAT` | Creation date |

### Data Record Key Fields (EDIDD)

| Field | Description |
|---|---|
| `DOCNUM` | IDoc number (links to control record) |
| `SEGNAM` | Segment name (e.g. E1EDKA1, E1EDP01) |
| `SDATA` | Segment data — 1000 bytes, mapped to segment structure |

## Common IDoc Status Codes

| Status | Direction | Meaning |
|---|---|---|
| 01 | Out | IDoc created |
| 02 | Out | Error passing data to port |
| 03 | Out | Data passed to port OK |
| 12 | Out | Dispatch OK |
| 30 | Out | IDoc ready for dispatch |
| 51 | In | Error in application |
| 52 | In | Application not yet executed |
| 53 | In | Application document posted |
| 56 | In | IDoc with errors added |
| 64 | In | IDoc ready to transfer to app |
| 68 | In | Error — no further processing |

## Inbound IDoc Processing

### Standard Function Module Signature

```abap
FUNCTION z_idoc_input_zmytype.
*"----------------------------------------------------------------------
*"  Tables
*"    IDOC_CONTRL  TYPE  EDIDC    " Control records
*"    IDOC_DATA    TYPE  EDIDD    " Data records
*"    IDOC_STATUS  TYPE  BDIDOCSTAT  " Status records (fill for result)
*"    RETURN_VARIABLES TYPE BDWFRETVAR
*"    SERIALIZATION_INFO TYPE BDI_SER
*"  Importing
*"    VALUE(INPUT_METHOD)    TYPE BDWFAP_PAR-INPUTMETHD
*"    VALUE(MASS_PROCESSING) TYPE BDWFAP_PAR-MASS_PROC
*"----------------------------------------------------------------------

  LOOP AT idoc_contrl ASSIGNING FIELD-SYMBOL(<ctrl>).

    TRY.
        " Extract header segment
        DATA(ls_hdr_seg) = VALUE e1zmyhdr(
          CORRESPONDING #( idoc_data[
            segnam = 'E1ZMYHDR'
            docnum = <ctrl>-docnum ] ) ).

        " Extract item segments
        DATA lt_items TYPE TABLE OF e1zmyitm.
        LOOP AT idoc_data ASSIGNING FIELD-SYMBOL(<seg>)
          WHERE segnam = 'E1ZMYITM'
            AND docnum = <ctrl>-docnum.
          APPEND CORRESPONDING e1zmyitm( <seg>-sdata ) TO lt_items.
        ENDLOOP.

        " Business logic
        CALL FUNCTION 'Z_PROCESS_MY_IDOC'
          EXPORTING
            is_header = ls_hdr_seg
            it_items  = lt_items.

        COMMIT WORK.

        " Success status
        APPEND VALUE bdidocstat(
          docnum = <ctrl>-docnum
          status = '53'           " Application document posted
          msgty  = 'S'
          msgid  = 'ZMY_MSGS'
          msgno  = '001' ) TO idoc_status.

      CATCH cx_root INTO DATA(lo_ex).
        ROLLBACK WORK.
        APPEND VALUE bdidocstat(
          docnum = <ctrl>-docnum
          status = '51'           " Error in application
          msgty  = 'E'
          msgid  = 'ZMY_MSGS'
          msgno  = '002'
          msgv1  = lo_ex->get_text( ) ) TO idoc_status.
    ENDTRY.

  ENDLOOP.

ENDFUNCTION.
```

### Register Inbound FM as Process Code (WE42)

```
WE42 → Create process code
  Process code:   ZMYTYPE_IN
  Function module: Z_IDOC_INPUT_ZMYTYPE
  Processing:      Trigger by background program
```

## Outbound IDoc Processing

### Build and Send Outbound IDoc

```abap
DATA: ls_ctrl  TYPE edidc,
      lt_data  TYPE TABLE OF edidd,
      lt_ctrl  TYPE TABLE OF edidc.

" 1. Control record
ls_ctrl = VALUE edidc(
  mestyp  = 'ZMYTYPE'
  idoctp  = 'ZMYBASIC01'
  rcvprt  = 'LS'
  rcvprn  = lv_receiver_logical_system
  sndprt  = 'LS'
  sndprn  = sy-sysid ).

" 2. Header segment
APPEND VALUE edidd(
  segnam = 'E1ZMYHDR'
  sdata  = CORRESPONDING #( ls_my_header ) ) TO lt_data.

" 3. Item segments
LOOP AT lt_items ASSIGNING FIELD-SYMBOL(<item>).
  APPEND VALUE edidd(
    segnam = 'E1ZMYITM'
    sdata  = CORRESPONDING #( <item> ) ) TO lt_data.
ENDLOOP.

" 4. Distribute
CALL FUNCTION 'MASTER_IDOC_DISTRIBUTE'
  EXPORTING
    master_idoc_control        = ls_ctrl
  TABLES
    communication_idoc_control = lt_ctrl
    master_idoc_data           = lt_data
  EXCEPTIONS
    error_in_idoc_control      = 1
    error_sending_to_partner   = 2.

COMMIT WORK.
```

### Trigger Outbound from Application Event (Message Control)

```abap
" Used in output condition records (NAST table)
" Called by output type processing via V1 condition

CALL FUNCTION 'IDOC_OUTPUT_ZMYTYPE'
  EXPORTING
    object      = ls_nast
  TABLES
    idoc_data   = lt_data
  EXCEPTIONS
    error_idoc  = 1.
```

## Creating Custom IDoc Types

### Step-by-Step (WE31 → WE30 → WE82 → WE57)

```
1. WE31 — Create segment type
   Segment: E1ZMYHDR
   Fields:  ZFIELD1 (CHAR 10), ZFIELD2 (NUMC 8), ...

2. WE30 — Create basic IDoc type
   Basic type: ZMYBASIC01
   Add segment E1ZMYHDR (mandatory, 1 occurrence)
   Add segment E1ZMYITM (optional, 1–999 occurrences)

3. WE82 — Assign basic type to message type
   Message type: ZMYTYPE
   Basic type:   ZMYBASIC01
   Release:      current release

4. WE57 — Link message type to inbound FM
   Message type:    ZMYTYPE
   Function module: Z_IDOC_INPUT_ZMYTYPE
   Direction:       2 (inbound)
```

## Extending Standard IDocs with Z-Segments

```
1. WE31 — Create Z-segment (e.g. E1ZMYEXT)
   Add your custom fields

2. WE30 — Extend standard basic type
   Open ORDERS05 → Create extension → ZORDERS05
   Add E1ZMYEXT under the relevant parent segment

3. WE20 — Update partner profile to use ZORDERS05
   Basic type: ZORDERS05 (instead of ORDERS05)
```

### Read Custom Extension Segment

```abap
" In inbound FM, read custom extension segment
DATA ls_ext TYPE e1zmyext.
READ TABLE idoc_data ASSIGNING FIELD-SYMBOL(<ext_seg>)
  WITH KEY segnam = 'E1ZMYEXT'
           docnum = <ctrl>-docnum.
IF sy-subrc = 0.
  ls_ext = CORRESPONDING #( <ext_seg>-sdata ).
ENDIF.
```

## Partner Profile Configuration (WE20)

```
WE20 → Partner type options:
  LS = Logical System (system-to-system)
  LI = Vendor
  KU = Customer
  B  = Bank

Inbound Parameters:
  Message type, process code, trigger (immediate/background)

Outbound Parameters:
  Message type, basic type, receiver port, output mode
  (Transfer IDoc immediately / Collect IDocs)
```

### Partner Types by Use Case

| Scenario | Partner Type | Example |
|---|---|---|
| ERP ↔ EWM (same company) | LS | PRD_800, EWM_100 |
| ERP ↔ TM (same company) | LS | PRD_800, TMS_100 |
| Vendor EDI (purchase orders) | LI | Vendor number |
| Customer EDI (sales orders) | KU | Customer number |
| Carrier EDI (shipments) | LS or LI | Carrier system |

## ALE Distribution Model (BD64)

```
BD64 → Maintain Distribution Model
  → Add message type to model:
      Sender:   PRD_800 (ERP)
      Receiver: EWM_100
      Message:  WMMBXY

BD64 → Generate partner profiles (after model change)
  → Automatically creates WE20 entries
```

## Error Handling and Reprocessing

### Query Error IDocs

```abap
SELECT docnum, mestyp, direct, status, credat, cretim
  FROM edidc
  WHERE status IN ('51', '56', '68')      " Error statuses
    AND mestyp  = @lv_message_type
    AND credat >= @lv_from_date
  ORDER BY credat DESCENDING
  INTO TABLE @DATA(lt_errors).
```

### Reprocess Programmatically

```abap
" Collect error IDoc numbers
DATA lt_docnums TYPE TABLE OF edidc-docnum.
SELECT docnum FROM edidc
  WHERE status = '51' AND mestyp = 'DESADV'
  INTO TABLE @lt_docnums.

" Trigger reprocessing
CALL FUNCTION 'IDOC_INBOUND_ASYNCHRONOUS'
  TABLES
    idoc_control_rec_40 = lt_ctrl
    idoc_data_rec_40    = lt_data.
```

### Update IDoc Status Manually

```abap
" Mark IDoc as processed after manual correction
CALL FUNCTION 'IDOC_STATUS_WRITE_TO_DATABASE'
  EXPORTING
    idoc_number = lv_docnum
  TABLES
    idoc_status = VALUE #( (
      docnum = lv_docnum
      status = '53'
      msgty  = 'S'
      msgid  = 'ZMY'
      msgno  = '001'
      msgv1  = 'Manually corrected and reprocessed' ) ).
```

## Key Transactions

| Transaction | Purpose |
|---|---|
| `WE02` | IDoc list — search by type, partner, status, date |
| `WE05` | IDoc list with status filter |
| `WE06` | IDoc active monitoring |
| `WE19` | Test tool — simulate inbound IDoc manually |
| `WE20` | Partner profile maintenance |
| `WE30` | IDoc type maintenance (basic types) |
| `WE31` | Segment type maintenance |
| `WE42` | Process code maintenance (inbound) |
| `WE57` | FM/message type assignment |
| `WE60` | IDoc documentation / segment browser |
| `WE82` | Message type ↔ basic type assignment |
| `BD64` | ALE distribution model |
| `BD87` | Reprocess failed inbound IDocs |
| `SALE` | ALE configuration (logical systems, RFC) |
| `SM58` | tRFC monitor (stuck outbound IDocs) |
