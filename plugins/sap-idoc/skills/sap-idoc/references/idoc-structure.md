# IDoc Structure Reference

## Three Record Types

### 1. Control Record (EDIDC / EDIDC40)

One per IDoc — envelope with routing and status.

```abap
DATA ls_ctrl TYPE edidc.

ls_ctrl-mestyp    = 'ORDERS'.          " Message type
ls_ctrl-idoctp    = 'ORDERS05'.        " Basic IDoc type (= IDOCTP if no extension)
ls_ctrl-direct    = '1'.               " 1=Outbound, 2=Inbound
ls_ctrl-rcvprt    = 'LS'.              " Receiver partner type
ls_ctrl-rcvprn    = 'EWM_100'.         " Receiver partner number
ls_ctrl-sndprt    = 'LS'.             " Sender partner type
ls_ctrl-sndprn    = 'PRD_800'.         " Sender partner number
ls_ctrl-rcvpor    = lv_port.           " Receiver port (for outbound)
```

### 2. Data Records (EDIDD / EDIDD40)

One per segment — flat 1000-byte payload mapped to typed segment structures.

```abap
DATA ls_seg TYPE edidd.

ls_seg-segnam = 'E1EDKA1'.            " Segment name
ls_seg-sdata  = CORRESPONDING #( ls_e1edka1 ).  " Cast struct → 1000-byte SDATA

" Reverse: cast SDATA → typed struct
DATA ls_e1edka1 TYPE e1edka1.
ls_e1edka1 = CORRESPONDING #( ls_seg-sdata ).
" OR:
MOVE ls_seg-sdata TO ls_e1edka1.
```

### 3. Status Records (EDIDS / BDIDOCSTAT)

Processing history — added by each processing step.

```abap
DATA ls_status TYPE bdidocstat.

ls_status-docnum = lv_docnum.
ls_status-status = '53'.              " Status code
ls_status-msgty  = 'S'.              " Message type S/E/W/I
ls_status-msgid  = 'MY_MSG_CLASS'.
ls_status-msgno  = '001'.
ls_status-msgv1  = 'Processing complete'.
```

## Segment Anatomy

Each segment type has:
- A name (e.g. `E1EDKA1`)
- An ABAP structure (e.g. `E1EDKA1`)
- A parent segment in the IDoc tree
- Minimum/maximum occurrence rules

### Reading Segments from EDIDD

```abap
" Read first occurrence of a segment
READ TABLE idoc_data ASSIGNING FIELD-SYMBOL(<seg>)
  WITH KEY segnam = 'E1EDK01'
           docnum = <ctrl>-docnum.
IF sy-subrc = 0.
  DATA ls_e1edk01 TYPE e1edk01.
  MOVE <seg>-sdata TO ls_e1edk01.
ENDIF.

" Read all occurrences of a repeating segment
LOOP AT idoc_data ASSIGNING FIELD-SYMBOL(<item_seg>)
  WHERE segnam = 'E1EDP01'
    AND docnum = <ctrl>-docnum.
  DATA ls_item TYPE e1edp01.
  MOVE <item_seg>-sdata TO ls_item.
  " Process item...
ENDLOOP.
```

### Building Segments for Outbound

```abap
DATA: lt_data TYPE TABLE OF edidd,
      ls_seg  TYPE edidd.

" Header segment
DATA ls_e1edk01 TYPE e1edk01.
ls_e1edk01-action = '000'.            " 000=Original, 003=Cancel
ls_e1edk01-curcy  = 'USD'.
ls_e1edk01-hwaer  = 'USD'.

CLEAR ls_seg.
ls_seg-segnam = 'E1EDK01'.
MOVE ls_e1edk01 TO ls_seg-sdata.
APPEND ls_seg TO lt_data.

" Item segment
DATA ls_e1edp01 TYPE e1edp01.
ls_e1edp01-posex = '000010'.          " Item number
ls_e1edp01-action = '000'.
ls_e1edp01-matnr  = lv_material.
ls_e1edp01-menge  = lv_quantity.
ls_e1edp01-menee  = lv_uom.

CLEAR ls_seg.
ls_seg-segnam = 'E1EDP01'.
MOVE ls_e1edp01 TO ls_seg-sdata.
APPEND ls_seg TO lt_data.
```

## IDoc Number (DOCNUM)

IDoc numbers are 16-digit internal keys assigned by SAP. Never hard-code; always use the system-assigned number.

```abap
" After MASTER_IDOC_DISTRIBUTE:
LOOP AT lt_comm_ctrl ASSIGNING FIELD-SYMBOL(<sent>).
  DATA(lv_docnum) = <sent>-docnum.    " Assigned IDoc number
ENDLOOP.

" Lookup IDoc by application document
SELECT docnum FROM edidc
  WHERE mestyp = 'ORDERS'
    AND direct = '1'
    AND credat = @sy-datum
  INTO TABLE @DATA(lt_docnums).
```

## IDoc vs. ALE vs. EDI

| Term | Meaning |
|---|---|
| **IDoc** | The data container format — used internally and externally |
| **ALE** | Application Link Enabling — SAP-to-SAP IDoc exchange |
| **EDI** | Electronic Data Interchange — SAP-to-external partner exchange |
| **Port** | Outbound channel: tRFC port (SAP↔SAP), file port (EDI middleware) |
| **Subsystem** | External EDI translator (e.g. Seeburger, MuleSoft, Dell Boomi) |

## Hierarchy: Message Type → Basic Type → Extension

```
Message Type: ORDERS          (business meaning — "Purchase Order")
  Basic Type: ORDERS05        (structure version)
    Extension: ZORDERS05      (Z-segments added on top)
      → Used in WE20 partner profile as "IDoc Type"
```
