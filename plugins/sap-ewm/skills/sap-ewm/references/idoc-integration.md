# EWM IDoc Integration Reference

## Key EWM IDoc Message Types

| Message Type | Basic Type | Direction | Purpose |
|---|---|---|---|
| `WMMBXY` | `WMMBXY01` | Inbound | Goods movement (GR, GI, transfer) |
| `SHPMNT` | `SHPMNT05` | Inbound/Outbound | Shipment notification (ASN) |
| `DESADV` | `DELVRY07` | Inbound | Delivery/ASN from supplier |
| `DLVRY` | `DELVRY07` | Outbound | Outbound delivery notification |
| `WMCATO` | `WMCATO01` | Outbound | Transfer order confirmation |
| `WMSORD` | `WMSORD01` | Inbound | Warehouse order |

## Partner Profile Setup (WE20)

```
WE20 → Partner type LS (Logical System) or LI (Vendor)
  → Inbound parameters:
      Message type: DESADV
      Process code: DESADV_IN   (or custom Z function module)
      Trigger immediately or via background job
  → Outbound parameters:
      Message type: WMCATO
      Output mode: Transfer IDoc immediately
      Receiver port: your RFC destination
```

## Processing Inbound IDocs

### Standard Process Codes

| Process Code | Message Type | Handler FM |
|---|---|---|
| `DESADV_IN` | DESADV | `/SCWM/IDOC_INPUT_DESADV` |
| `WMMBXY_IN` | WMMBXY | `/SCWM/IDOC_INPUT_WMMBXY` |
| `SHPMNT_IN` | SHPMNT | `/SCWM/IDOC_INPUT_SHPMNT` |

### Custom Inbound IDoc Function Module

```abap
FUNCTION z_idoc_input_custom.
  " Standard IDoc inbound FM signature
  IMPORTING
    VALUE(input_method)  TYPE bdwfap_par-inputmethd
    VALUE(mass_processing) TYPE bdwfap_par-mass_proc
  TABLES
    idoc_contrl  STRUCTURE edidc
    idoc_data    STRUCTURE edidd
    idoc_status  STRUCTURE bdidocstat
    return_variables STRUCTURE bdwfretvar
    serialization_info STRUCTURE bdi_ser.

  LOOP AT idoc_data ASSIGNING FIELD-SYMBOL(<seg>)
    WHERE segnam = 'E1EDL20'.  " Delivery header segment

    " Map IDoc segment to EWM delivery structure
    DATA ls_header TYPE /scdl/ds_dochead_rfc.
    MOVE-CORRESPONDING <seg>-sdata TO ls_header.

    " Create EWM delivery
    CALL FUNCTION '/SCWM/DLV_CREATE_FROM_RFC'
      EXPORTING
        is_header  = ls_header
      IMPORTING
        et_bapiret = DATA(lt_msg).

    " Set IDoc status
    IF lt_msg IS INITIAL.
      idoc_status-status = '53'.  " Posted
    ELSE.
      idoc_status-status = '51'.  " Error
      idoc_status-msgid  = lt_msg[ 1 ]-id.
      idoc_status-msgno  = lt_msg[ 1 ]-number.
    ENDIF.
  ENDLOOP.
ENDFUNCTION.
```

## Sending Outbound IDocs

### WMCATO — Transfer Order Confirmation

EWM can automatically send WMCATO IDocs when a transfer order is confirmed (relevant for
decentralized EWM sending confirmations back to ERP).

```abap
" Trigger WMCATO outbound after TO confirmation
CALL FUNCTION '/SCWM/IDOC_WMCATO_CREATE'
  EXPORTING
    iv_lgnum    = lv_lgnum
    iv_tanum    = lv_task_number  " Confirmed WT number
  IMPORTING
    et_bapiret  = DATA(lt_messages).
```

### Send Generic Outbound IDoc

```abap
DATA: lt_edidc TYPE TABLE OF edidc,
      lt_edidd TYPE TABLE OF edidd.

" Build control record
APPEND VALUE edidc(
  mestyp   = 'WMCATO'
  mescod   = space
  mesfct   = space
  rcvprt   = 'LS'
  rcvprn   = lv_receiver_logical_system
  sndprt   = 'LS'
  sndprn   = sy-sysid ) TO lt_edidc.

" Build data segments (E1WM_...  structures)
APPEND VALUE edidd(
  segnam = 'E1WM_TANUM'
  sdata  = lv_task_number ) TO lt_edidd.

CALL FUNCTION 'MASTER_IDOC_DISTRIBUTE'
  EXPORTING
    master_idoc_control      = lt_edidc[ 1 ]
  TABLES
    communication_idoc_control = lt_edidc
    master_idoc_data           = lt_edidd.

COMMIT WORK.
```

## IDoc Monitoring and Error Handling

### Transactions

| Transaction | Purpose |
|---|---|
| `WE02` | IDoc list — display sent/received IDocs |
| `WE05` | IDoc list with status filter |
| `WE19` | Test IDoc inbound processing |
| `BD87` | Reprocess failed inbound IDocs |
| `WE20` | Partner profiles |
| `WE60` | IDoc documentation / segment definition |

### Reprocess Failed IDocs

```abap
" Programmatic reprocessing of error IDocs (status 51)
CALL FUNCTION 'IDOC_INBOUND_ASYNCHRONOUS'
  TABLES
    idoc_control_rec_40 = lt_edidc
    idoc_data_rec_40    = lt_edidd.
```

### Query IDoc Status

```abap
" Find failed IDocs for a message type
SELECT docnum, status, direct
  FROM edidc
  WHERE mestyp = 'DESADV'
    AND status IN ('51', '56', '63')  " Error statuses
    AND credat >= @lv_from_date
  INTO TABLE @DATA(lt_failed_idocs).
```

### Common IDoc Status Codes

| Status | Meaning |
|---|---|
| 01 | IDoc created |
| 12 | Dispatch OK |
| 51 | Error in application |
| 52 | Not yet processed |
| 53 | Posted / processed |
| 56 | IDoc with errors added |
| 64 | IDoc ready to transfer |
| 68 | Error — no further processing |

## DESADV (ASN) Processing for EWM

```abap
" DESADV inbound creates EWM inbound delivery
" Standard process: DESADV → /SCWM/IDOC_INPUT_DESADV → EWM Inbound Delivery

" Enhance with BADI for custom field mapping:
" BADI: /SCWM/EX_IDOC_DESADV
" Enhancement spot: /SCWM/ES_IDOC_DESADV

METHOD /scwm/if_ex_idoc_desadv~map_delivery_header.
  " Map custom DESADV fields to EWM delivery header
  " is_e1edl20: DESADV header segment
  " cs_header:  EWM delivery header to modify
  cs_header-zz_custom_field = is_e1edl20-zz_idoc_field.
ENDMETHOD.
```

## EWM ↔ ERP Integration (Decentralized EWM)

In decentralized EWM (separate system from ERP), IDocs handle all stock and delivery sync:

```
ERP (SAP ECC/S4)          IDocs              EWM (Decentralized)
─────────────────    ────────────────    ──────────────────────
Inbound Delivery  ──→ DESADV/SHPMNT ──→ EWM Inbound Delivery
Transfer Order    ──→ WMTA           ──→ Warehouse Task
TO Confirmation   ←── WMCATO        ←── WT Confirmation
Goods Movement    ──→ WMMBXY        ──→ Stock Posting
Inventory         ←── WMINVE        ←── PI Count Result
```

### Middleware and ALE Setup

```
SALE → Logical systems → Define and assign
BD64  → Distribution model → Add message types between logical systems
WE20  → Partner profiles → Inbound/outbound for each message type
SM59  → RFC destinations → ERP ↔ EWM connection
```
