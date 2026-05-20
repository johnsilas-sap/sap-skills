# PPF Integration Reference

## PPF Architecture

Post Processing Framework (PPF) triggers form output automatically when business events occur (delivery save, TO confirmation, GR posting).

```
Business Event (EWM/TM/SD)
  → PPF Condition check (NAST / application-specific)
  → PPF Action Profile selected
  → IF_FPF_ACTION~EXECUTE called on action class
  → Print program logic inside Execute
  → FP_JOB_OPEN → form FM → FP_JOB_CLOSE
```

## Creating the PPF Action Class

### Class Definition

```abap
CLASS zcl_ppf_ewm_delivery_note DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES if_fpf_action.

  PRIVATE SECTION.
    METHODS:
      get_delivery_data
        IMPORTING iv_docno    TYPE /scdl/db_proci_o-docno
        EXPORTING es_header   TYPE zewm_deliv_hdr_s
                  et_items    TYPE zewm_deliv_item_tab,
      get_printer
        IMPORTING iv_whno     TYPE /scwm/lgnum
        RETURNING VALUE(rv_dest) TYPE rspopname.
ENDCLASS.
```

### Execute Method

```abap
CLASS zcl_ppf_ewm_delivery_note IMPLEMENTATION.

  METHOD if_fpf_action~execute.
    DATA: lv_fm_name      TYPE rs38l_fnam,
          ls_outputparams TYPE sfpoutputparams,
          ls_header       TYPE zewm_deliv_hdr_s,
          lt_items        TYPE zewm_deliv_item_tab.

    " Get document key from PPF application object
    DATA(lv_docno) = io_appl_object->get_key( )-docno.

    " Populate form data
    get_delivery_data(
      EXPORTING iv_docno  = lv_docno
      IMPORTING es_header = ls_header
                et_items  = lt_items ).

    " Determine printer by warehouse
    DATA(lv_dest) = get_printer( ls_header-lgnum ).

    " Set output parameters
    ls_outputparams = VALUE sfpoutputparams(
      nodialog  = abap_true
      dest      = lv_dest
      copies    = 1
      reqimmed  = abap_true ).

    " Print
    CALL FUNCTION 'FP_JOB_OPEN'
      CHANGING   ie_outputparams = ls_outputparams
      EXCEPTIONS cancel = 1 usage_error = 2 system_error = 3.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE /bobf/cx_frw_businesserror
        MESSAGE e001(zforms) WITH 'FP_JOB_OPEN failed'.
    ENDIF.

    CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
      EXPORTING  i_name     = 'ZEWM_DELIVERY_NOTE'
      IMPORTING  e_funcname = lv_fm_name.

    CALL FUNCTION lv_fm_name
      EXPORTING
        is_header = ls_header
        it_items  = lt_items
      EXCEPTIONS
        usage_error    = 1
        system_error   = 2
        internal_error = 3.

    CALL FUNCTION 'FP_JOB_CLOSE'
      EXCEPTIONS usage_error = 1 system_error = 2 internal_error = 3.

    " Return number of copies printed
    ev_result = 1.
  ENDMETHOD.

ENDCLASS.
```

## Configuring PPF Actions (Transaction SPPFCADM / SPPFC)

### Step 1 — Define Action Profile

```
SPPFCADM → PPF: Action Profiles and Actions
  Application:  /SCWM/DLV         (EWM Delivery)
  Create Action Profile:
    Name:        ZEWM_DELIV_PRINT
    Description: EWM Delivery Note Print
```

### Step 2 — Create Action Definition

```
Action Profile → Actions → Create:
  Action Name:   ZEWM_PRINT_NOTE
  Description:   Print Delivery Note
  Processing:    Using Method
  Method:        ZCL_PPF_EWM_DELIVERY_NOTE (class implementing IF_FPF_ACTION)
  Processing Time: Immediate / Planned / Background

  Schedule Condition:
    (Define when this action is scheduled — e.g., on delivery creation)
  Start Condition:
    (Define when execution fires — e.g., on Good Issue posting)
```

### Step 3 — Assign Action Profile to Document Type

```
EWM Customizing → Warehouse Management → Shipping
  → Delivery Types → Assign PPF Action Profile
    Delivery Type:    OUTBD
    Action Profile:   ZEWM_DELIV_PRINT
```

## Partner-Dependent Printing (NAST-Style)

```abap
" In action class — check partner output condition
METHOD if_fpf_action~execute.
  " Get output medium from PPF context
  DATA(lv_medium) = io_appl_object->get_partner_fct( )-medium.

  CASE lv_medium.
    WHEN '1'.  " Print
      ls_outputparams-dest = get_printer( ... ).
      " ... print logic ...
    WHEN '5'.  " Email
      ls_outputparams-getpdf = abap_true.
      " ... email logic ...
    WHEN OTHERS.
      RETURN.
  ENDCASE.
ENDMETHOD.
```

## EWM Delivery PPF Application Objects

| Application | Object Type | Key Field |
|---|---|---|
| `/SCWM/DLV` | EWM Delivery | `DOCNO` |
| `/SCWM/TORD` | Transfer Order | `TANUM` |
| `/SCWM/WO` | Warehouse Order | `WHORDNO` |

## TM Freight Order PPF

```abap
" TM PPF action class — implements IF_FPF_ACTION
METHOD if_fpf_action~execute.
  DATA(lv_froid) = io_appl_object->get_key( )-doc_id.

  " Get freight order header
  DATA(lo_fo_service) = /scmtms/cl_fro_bopf_handler=>get_instance( ).
  DATA(ls_fo_hdr) = lo_fo_service->get_header( lv_froid ).

  " Build form data from FO
  DATA ls_hdr TYPE ztm_form_hdr.
  ls_hdr-fro_id     = ls_fo_hdr-doc_id.
  ls_hdr-carrier    = ls_fo_hdr-carrier_id.
  ls_hdr-load_date  = ls_fo_hdr-loading_date.

  " ... print ...
ENDMETHOD.
```

## Multiple Copies for Different Recipients

```abap
" In execute method — print 1 original + 1 copy for driver
DATA: lv_fm TYPE rs38l_fnam,
      ls_out TYPE sfpoutputparams.

" Original: warehouse printer
ls_out = VALUE sfpoutputparams(
  nodialog = abap_true dest = lv_wh_printer copies = 1 ).
CALL FUNCTION 'FP_JOB_OPEN' CHANGING ie_outputparams = ls_out.
CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
  EXPORTING i_name = 'ZEWM_DELIVERY_NOTE' IMPORTING e_funcname = lv_fm.
CALL FUNCTION lv_fm EXPORTING is_header = ls_hdr it_items = lt_items.
CALL FUNCTION 'FP_JOB_CLOSE'.

" Driver copy: dock printer
ls_out-dest  = lv_dock_printer.
ls_out-copies = 2.
CALL FUNCTION 'FP_JOB_OPEN' CHANGING ie_outputparams = ls_out.
CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
  EXPORTING i_name = 'ZEWM_DELIVERY_NOTE' IMPORTING e_funcname = lv_fm.
CALL FUNCTION lv_fm EXPORTING is_header = ls_hdr it_items = lt_items.
CALL FUNCTION 'FP_JOB_CLOSE'.
```

## Testing PPF Actions

```
SPPF → Manual Trigger
  Application:  /SCWM/DLV
  Object Key:   <delivery docno>
  Action Profile: ZEWM_DELIV_PRINT
  → Execute

  Check spool: SP01
  Check PPF log: SPPF → Display Log
```

## Error Handling in PPF Context

```abap
" Raise /bobf/cx_frw_businesserror to log in PPF message log
TRY.
    " ... print logic ...
  CATCH cx_root INTO DATA(lo_ex).
    CALL FUNCTION 'FP_JOB_CLOSE' EXCEPTIONS OTHERS = 0.
    RAISE EXCEPTION TYPE /bobf/cx_frw_businesserror
      EXPORTING textid = /bobf/cx_frw_businesserror=>general_error
                previous = lo_ex.
ENDTRY.
```
