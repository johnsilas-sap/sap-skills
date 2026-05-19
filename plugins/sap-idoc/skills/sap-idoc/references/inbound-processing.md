# IDoc Inbound Processing Reference

## Processing Flow

```
External System / SAP Partner
  → IDoc arrives (tRFC, file, HTTP)
  → EDIDC / EDIDD records written to database (status 64)
  → Workflow or background job triggers processing
  → Process code lookup (WE20 → WE42)
  → Inbound FM called with IDOC_CONTRL + IDOC_DATA
  → FM sets status 53 (success) or 51 (error) in IDOC_STATUS
  → Status written to EDIDS
```

## Standard Inbound FM Signature

```abap
FUNCTION z_idoc_input_zmytype.
*"----------------------------------------------------------------------
*" Global data declarations:
  DATA: lt_error_status TYPE TABLE OF bdidocstat.

*"  Tables:
*"    IDOC_CONTRL         TYPE  EDIDC
*"    IDOC_DATA           TYPE  EDIDD
*"    IDOC_STATUS         TYPE  BDIDOCSTAT
*"    RETURN_VARIABLES    TYPE  BDWFRETVAR
*"    SERIALIZATION_INFO  TYPE  BDI_SER
*"  Importing:
*"    VALUE(INPUT_METHOD)    TYPE  BDWFAP_PAR-INPUTMETHD
*"    VALUE(MASS_PROCESSING) TYPE  BDWFAP_PAR-MASS_PROC
*"----------------------------------------------------------------------

  LOOP AT idoc_contrl ASSIGNING FIELD-SYMBOL(<ctrl>).

    " Validate IDoc type
    IF <ctrl>-mestyp <> 'ZMYTYPE'.
      APPEND VALUE bdidocstat(
        docnum = <ctrl>-docnum
        status = '51'
        msgty  = 'E'
        msgv1  = |Unexpected message type: { <ctrl>-mestyp }| ) TO idoc_status.
      CONTINUE.
    ENDIF.

    TRY.
        " Parse segments
        DATA ls_header TYPE e1zmyhdr.
        READ TABLE idoc_data ASSIGNING FIELD-SYMBOL(<hdr>)
          WITH KEY segnam = 'E1ZMYHDR' docnum = <ctrl>-docnum.
        IF sy-subrc = 0.
          MOVE <hdr>-sdata TO ls_header.
        ELSE.
          RAISE EXCEPTION TYPE cx_udm_message
            MESSAGE e001(zmymsgs) WITH 'Missing header segment'.
        ENDIF.

        DATA lt_items TYPE TABLE OF e1zmyitm.
        LOOP AT idoc_data ASSIGNING FIELD-SYMBOL(<itm>)
          WHERE segnam = 'E1ZMYITM' AND docnum = <ctrl>-docnum.
          DATA ls_item TYPE e1zmyitm.
          MOVE <itm>-sdata TO ls_item.
          APPEND ls_item TO lt_items.
        ENDLOOP.

        " Business logic
        my_process_idoc_data( is_header = ls_header it_items = lt_items ).
        COMMIT WORK.

        " Success
        APPEND VALUE bdidocstat(
          docnum = <ctrl>-docnum
          status = '53'
          msgty  = 'S'
          msgid  = 'ZMYMSGS'
          msgno  = '001'
          msgv1  = ls_header-ref_docno ) TO idoc_status.

      CATCH cx_root INTO DATA(lo_ex).
        ROLLBACK WORK.
        APPEND VALUE bdidocstat(
          docnum = <ctrl>-docnum
          status = '51'
          msgty  = 'E'
          msgid  = 'ZMYMSGS'
          msgno  = '002'
          msgv1  = lo_ex->get_text( ) ) TO idoc_status.
    ENDTRY.

  ENDLOOP.

ENDFUNCTION.
```

## Segment Parsing Patterns

### Single Occurrence Segment (Mandatory)

```abap
READ TABLE idoc_data ASSIGNING FIELD-SYMBOL(<seg>)
  WITH KEY segnam = 'E1EDK01' docnum = <ctrl>-docnum.
IF sy-subrc <> 0.
  " Missing mandatory segment — raise error
  RAISE EXCEPTION TYPE cx_udm_message.
ENDIF.
DATA ls_e1edk01 TYPE e1edk01.
MOVE <seg>-sdata TO ls_e1edk01.
```

### Repeating Segment (Items)

```abap
DATA lt_items TYPE TABLE OF e1edp01.
LOOP AT idoc_data ASSIGNING FIELD-SYMBOL(<seg>)
  WHERE segnam = 'E1EDP01' AND docnum = <ctrl>-docnum.
  DATA ls_item TYPE e1edp01.
  MOVE <seg>-sdata TO ls_item.
  APPEND ls_item TO lt_items.
ENDLOOP.
```

### Conditional/Optional Segment

```abap
DATA ls_partner TYPE e1edka1.
READ TABLE idoc_data ASSIGNING FIELD-SYMBOL(<prt>)
  WITH KEY segnam = 'E1EDKA1' docnum = <ctrl>-docnum.
IF sy-subrc = 0.
  MOVE <prt>-sdata TO ls_partner.
ENDIF.
" Continue even if segment missing
```

### Parent-Child Segment Relationship

```abap
" E1EDP01 (item) → E1EDP19 (item text) — must correlate by position
DATA lv_item_counter TYPE i VALUE 0.
LOOP AT idoc_data ASSIGNING FIELD-SYMBOL(<seg>)
  WHERE docnum = <ctrl>-docnum.
  CASE <seg>-segnam.
    WHEN 'E1EDP01'.
      ADD 1 TO lv_item_counter.
      " Process item header
    WHEN 'E1EDP19'.
      " This text belongs to current item (lv_item_counter)
  ENDCASE.
ENDLOOP.
```

## Status Codes — Inbound

| Code | Meaning | Action |
|---|---|---|
| 50 | IDoc added to database | Automatic |
| 51 | Error in application | Fix and reprocess |
| 52 | Application not yet executed | Trigger background job |
| 53 | Application document posted | Terminal success |
| 56 | IDoc with errors added | Fix and reprocess |
| 64 | IDoc ready to transfer to app | Pending processing |
| 68 | Error — no further processing | Manual intervention |

## Trigger Modes

### Immediate (Synchronous)

Processed inline when IDoc arrives. Set in WE20 inbound parameters:
- **Trigger**: Trigger immediately

### Background Job

Batch program `RBDAPP01` processes pending IDocs (status 64):

```
SM36 → Schedule job
  Program: RBDAPP01
  Variant: MESTYP = ZMYTYPE, STATUS = 64
  Schedule: every 5 minutes or as needed
```

### Workflow

Status 64 IDocs trigger SAP Business Workplace tasks for manual processing.

## Mass Processing Flag

```abap
" When MASS_PROCESSING = 'X', the FM is called once for ALL IDocs
" of the same type in the batch. Commit after each, not at the end.
IF mass_processing = abap_true.
  " Process each IDoc and commit individually
  LOOP AT idoc_contrl ASSIGNING FIELD-SYMBOL(<ctrl>).
    process_single_idoc( <ctrl> ).
    COMMIT WORK.
  ENDLOOP.
ELSE.
  " Single IDoc — standard processing
  process_single_idoc( idoc_contrl[ 1 ] ).
  COMMIT WORK.
ENDIF.
```

## Testing with WE19

```
WE19 → Enter IDoc number of an existing IDoc
      → Choose "Test with new control record"
      → Modify sender/receiver if needed
      → Execute → Calls inbound FM directly
      → Check result in WE02
```

## BADI: Inbound Enhancement (Standard IDocs)

For extending standard inbound processing without replacing the FM:

```abap
" Example: BADI for ORDERS inbound
" Enhancement spot: EDI_DC_40
" BADI: BADI_IDOC_INPUT_ORDERS

METHOD if_ex_badi_idoc_input_orders~change_data_before_app.
  " Modify parsed application data before it's saved
  " cs_ekko: purchase order header — modify in place
  cs_ekko-zz_custom = lv_value_from_idoc.
ENDMETHOD.
```
