# IDoc Outbound Processing Reference

## Processing Flow

```
Application event (save, output condition, manual trigger)
  → Outbound FM builds EDIDC + EDIDD records
  → MASTER_IDOC_DISTRIBUTE called
  → Partner profile lookup (WE20) → port + receiver
  → Status 01 (created) → 30 (ready) → 03/12 (dispatched)
  → tRFC to receiving system  OR  file written to port
```

## Building and Sending an IDoc

### Complete Pattern

```abap
DATA: ls_ctrl  TYPE edidc,
      lt_data  TYPE TABLE OF edidd,
      lt_comm  TYPE TABLE OF edidc.   " Communication IDocs (output)

"── 1. Control record ──────────────────────────────────────────────
ls_ctrl = VALUE edidc(
  mestyp   = 'ZMYTYPE'               " Message type
  idoctp   = 'ZMYBASIC01'            " Basic IDoc type (or extension)
  sndprt   = 'LS'
  sndprn   = sy-sysid
  rcvprt   = 'LS'
  rcvprn   = lv_receiver ).

"── 2. Header segment ──────────────────────────────────────────────
DATA ls_hdr TYPE e1zmyhdr.
ls_hdr-ref_docno = lv_document_number.
ls_hdr-doc_date  = sy-datum.
ls_hdr-currency  = lv_currency.

APPEND VALUE edidd(
  segnam = 'E1ZMYHDR'
  sdata  = CONV edidd-sdata( ls_hdr ) ) TO lt_data.

"── 3. Item segments ───────────────────────────────────────────────
LOOP AT lt_my_items ASSIGNING FIELD-SYMBOL(<item>).
  DATA ls_itm TYPE e1zmyitm.
  ls_itm-item_no = <item>-posnr.
  ls_itm-matnr   = <item>-matnr.
  ls_itm-menge   = <item>-menge.
  ls_itm-meins   = <item>-meins.

  APPEND VALUE edidd(
    segnam = 'E1ZMYITM'
    sdata  = CONV edidd-sdata( ls_itm ) ) TO lt_data.
ENDLOOP.

"── 4. Distribute ──────────────────────────────────────────────────
CALL FUNCTION 'MASTER_IDOC_DISTRIBUTE'
  EXPORTING
    master_idoc_control        = ls_ctrl
  TABLES
    communication_idoc_control = lt_comm   " Returns created IDocs
    master_idoc_data           = lt_data
  EXCEPTIONS
    error_in_idoc_control      = 1
    error_sending_to_partner   = 2
    OTHERS                     = 3.

IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE cx_udm_message.
ENDIF.

COMMIT WORK.

" Retrieve assigned IDoc numbers
LOOP AT lt_comm ASSIGNING FIELD-SYMBOL(<comm>).
  DATA(lv_docnum) = <comm>-docnum.
ENDLOOP.
```

## Output Modes

Configured in WE20 outbound parameters:

| Mode | Constant | Behavior |
|---|---|---|
| Transfer immediately | `4` | Sent as soon as `COMMIT WORK` executes |
| Collect IDocs | `2` | Batched — sent by `RSEOUT00` job |
| Transfer immediately (background) | `1` | Sent via background tRFC |

### Sending Collected IDocs (RSEOUT00)

```
SM36 → Schedule RSEOUT00
  MESTYP = ZMYTYPE
  STATUS = 30             " Ready for dispatch
  Schedule as needed (e.g. hourly)
```

## IDoc Ports

### tRFC Port (SAP ↔ SAP)

```
WE21 → Transactional RFC port
  Port name:        RFC_EWM100
  RFC destination:  EWM_100       (from SM59)
  IDoc version:     3 (current)
```

### File Port (SAP ↔ EDI Middleware)

```
WE21 → File port
  Port name:   FILE_CARRIER
  Directory:   /usr/sap/interfaces/outbound/
  File name:   %MESTYP%_%DATE%_%TIME%.idoc
  IDoc version: 3
```

## Triggering from Message Control (Output Types)

Used in SD (billing, delivery), MM (purchase orders) via condition records:

```abap
" Output processing class — called by NAST
METHOD process_output.
  " Build IDoc
  CALL FUNCTION 'IDOC_OUTPUT_ZMYTYPE'
    EXPORTING
      object    = is_nast           " NAST output record
    TABLES
      idoc_data = lt_data
    EXCEPTIONS
      error_idoc_output = 1.

  IF sy-subrc = 0.
    CALL FUNCTION 'MASTER_IDOC_DISTRIBUTE'
      " ... (as above)
  ENDIF.
ENDMETHOD.
```

## Triggering from Application Save (User Exit / BADI)

```abap
" Example: send IDoc when a custom business object is saved
METHOD after_save.
  DATA: ls_ctrl TYPE edidc,
        lt_data TYPE TABLE OF edidd,
        lt_comm TYPE TABLE OF edidc.

  build_idoc(
    EXPORTING is_data   = is_saved_object
    IMPORTING ls_ctrl   = ls_ctrl
              lt_data   = lt_data ).

  CALL FUNCTION 'MASTER_IDOC_DISTRIBUTE'
    EXPORTING
      master_idoc_control        = ls_ctrl
    TABLES
      communication_idoc_control = lt_comm
      master_idoc_data           = lt_data.

  COMMIT WORK.
ENDMETHOD.
```

## Retransmitting an Outbound IDoc

```abap
" Resend a specific IDoc number
CALL FUNCTION 'IDOC_RESEND'
  EXPORTING
    pi_docnum   = lv_docnum
  EXCEPTIONS
    idoc_locked = 1
    idoc_status = 2.

COMMIT WORK.
```

## Checking Outbound Status

```abap
SELECT docnum, status, rcvprn, credat
  FROM edidc
  WHERE mestyp  = 'ZMYTYPE'
    AND direct  = '1'                  " Outbound
    AND status NOT IN ('12', '16')     " Exclude successfully sent
    AND credat  >= @lv_from
  INTO TABLE @DATA(lt_pending).
```

## tRFC Stuck IDocs (SM58)

If outbound IDocs are stuck (status 03 but not 12):

```
SM58 → Filter by function module IDOC_INBOUND_ASYNCHRONOUS
     → Check RFC error reason
     → Execute / delete depending on root cause
```

Common causes:
- RFC destination down → fix SM59 connection
- Receiver system locked → wait and re-trigger
- Authorization failure → check S_RFC object on receiver
