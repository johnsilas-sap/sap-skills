# Adobe Forms Print Program Reference

## Minimal Print Program

```abap
REPORT z_print_form.

DATA: lv_fm_name      TYPE rs38l_fnam,
      ls_outputparams TYPE sfpoutputparams,
      ls_data         TYPE zmy_form_data.

" Fill data
ls_data = get_form_data( lv_key ).

" Output: print to default printer
ls_outputparams-nodialog = abap_true.

CALL FUNCTION 'FP_JOB_OPEN'
  CHANGING  ie_outputparams = ls_outputparams
  EXCEPTIONS cancel = 1 usage_error = 2 system_error = 3.
IF sy-subrc <> 0. RETURN. ENDIF.

CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
  EXPORTING i_name     = 'ZMY_FORM'
  IMPORTING e_funcname = lv_fm_name.

CALL FUNCTION lv_fm_name
  EXPORTING is_data = ls_data
  EXCEPTIONS usage_error = 1 system_error = 2 internal_error = 3.

CALL FUNCTION 'FP_JOB_CLOSE'
  EXCEPTIONS usage_error = 1 system_error = 2 internal_error = 3.
```

## Output Scenarios

### Print to Spool (Immediate)

```abap
ls_outputparams = VALUE sfpoutputparams(
  nodialog  = abap_true
  dest      = 'LP01'       " Printer name from SPAD
  reqimmed  = abap_true    " Immediate print, no spool retention
  copies    = 1 ).
```

### Print to Spool (Retained for Manual Release)

```abap
ls_outputparams = VALUE sfpoutputparams(
  nodialog  = abap_true
  dest      = 'LP01'
  reqimmed  = abap_false   " Hold in spool — user releases via SP01
  reqdel    = abap_false   " Keep after printing
  copies    = 2 ).
```

### Retrieve PDF in Memory (No Print)

```abap
ls_outputparams = VALUE sfpoutputparams(
  nodialog  = abap_true
  getpdf    = abap_true ).  " PDF returned by FP_JOB_CLOSE

DATA ls_result TYPE sfpjoboutput.
CALL FUNCTION 'FP_JOB_CLOSE'
  IMPORTING e_result = ls_result.

DATA(lv_pdf_xstring) = ls_result-pdf.  " Raw PDF bytes
```

### Open PDF Preview in Browser

```abap
ls_outputparams = VALUE sfpoutputparams(
  nodialog  = abap_true
  preview   = abap_true ).  " Opens PDF in SAP GUI browser pane
```

### Archive to SAP ArchiveLink

```abap
ls_outputparams = VALUE sfpoutputparams(
  nodialog    = abap_true
  pdfarchive  = abap_true
  archhdr     = VALUE ssfarchhdr(
    object_type  = 'VBDLN'        " Delivery note archive object
    doc_type     = 'PDF'
    mandant      = sy-mandt ) ).
```

## Multi-Form Job (Multiple Forms in One Spool)

```abap
" Open one job → call multiple forms → one spool entry
CALL FUNCTION 'FP_JOB_OPEN'
  CHANGING ie_outputparams = ls_outputparams.

" Form 1: Cover page
CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
  EXPORTING i_name = 'ZEWM_COVER' IMPORTING e_funcname = lv_fm1.
CALL FUNCTION lv_fm1 EXPORTING is_data = ls_cover_data.

" Form 2: Item list
CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
  EXPORTING i_name = 'ZEWM_ITEMS' IMPORTING e_funcname = lv_fm2.
CALL FUNCTION lv_fm2 EXPORTING it_items = lt_items.

" Both forms in same spool job
CALL FUNCTION 'FP_JOB_CLOSE'.
```

## Multiple Documents in a Loop

```abap
" Print one delivery note per delivery — each gets its own spool
LOOP AT lt_deliveries ASSIGNING FIELD-SYMBOL(<dlv>).
  DATA(ls_header) = get_delivery_header( <dlv>-docno ).
  DATA(lt_items)  = get_delivery_items( <dlv>-docno ).

  CALL FUNCTION 'FP_JOB_OPEN'
    CHANGING ie_outputparams = ls_outputparams.

  CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
    EXPORTING i_name = 'ZEWM_DELIVERY_NOTE' IMPORTING e_funcname = lv_fm.

  CALL FUNCTION lv_fm
    EXPORTING is_header = ls_header it_items = lt_items.

  CALL FUNCTION 'FP_JOB_CLOSE'.
ENDLOOP.
```

## Language and Country Control (SFPDOCPARAMS)

```abap
DATA ls_docparams TYPE sfpdocparams.
ls_docparams-langu   = 'E'.      " English
ls_docparams-country = 'US'.     " US formatting (dates, numbers)

CALL FUNCTION 'FP_JOB_OPEN'
  EXPORTING  ie_docparams    = ls_docparams
  CHANGING   ie_outputparams = ls_outputparams.
```

Language codes:
| Code | Language |
|---|---|
| `D` | German |
| `E` | English |
| `F` | French |
| `S` | Spanish |
| `P` | Portuguese |
| `Z` | Chinese |
| `J` | Japanese |

## Sending PDF by Email

```abap
" Retrieve PDF
ls_outputparams-getpdf = abap_true.
CALL FUNCTION 'FP_JOB_OPEN' CHANGING ie_outputparams = ls_outputparams.
CALL FUNCTION lv_fm EXPORTING is_data = ls_data.
CALL FUNCTION 'FP_JOB_CLOSE' IMPORTING e_result = ls_result.

" Convert XSTRING → SOLIX table for email
DATA lt_pdf_tab TYPE STANDARD TABLE OF solisti1.
CALL FUNCTION 'SCMS_XSTRING_TO_SOLIX'
  EXPORTING buffer = ls_result-pdf
  TABLES    solix  = lt_pdf_tab.

" Send email with PDF attachment
CALL FUNCTION 'SO_NEW_DOCUMENT_ATT_SEND_API1'
  EXPORTING
    document_data = VALUE sofdocchgi1(
      obj_name  = 'DELIV_NOTE'
      obj_descr = |Delivery Note { lv_delivery_no }|
      doc_size  = xstrlen( ls_result-pdf ) )
  TABLES
    object_content            = VALUE #( ( line = space ) )
    receivers                 = VALUE #(
      ( receiver = lv_email rec_type = 'U' ) )
    attachment_header         = VALUE #(
      ( line = |Delivery_Note_{ lv_delivery_no }.pdf| ) )
    attachment_object_content = lt_pdf_tab.
```

## Error Handling Pattern

```abap
TRY.
    CALL FUNCTION 'FP_JOB_OPEN'
      CHANGING ie_outputparams = ls_outputparams
      EXCEPTIONS
        cancel         = 1
        usage_error    = 2
        system_error   = 3
        internal_error = 4.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE zcx_form_error
        MESSAGE e001(zforms) WITH 'FP_JOB_OPEN failed' sy-subrc.
    ENDIF.

    CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
      EXPORTING  i_name     = lv_form_name
      IMPORTING  e_funcname = lv_fm_name
      EXCEPTIONS usage_error = 1 system_error = 2.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE zcx_form_error
        MESSAGE e002(zforms) WITH lv_form_name.
    ENDIF.

    CALL FUNCTION lv_fm_name
      EXPORTING  is_data        = ls_data
      EXCEPTIONS usage_error    = 1
                 system_error   = 2
                 internal_error = 3.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE zcx_form_error
        MESSAGE e003(zforms) WITH lv_fm_name sy-subrc.
    ENDIF.

    CALL FUNCTION 'FP_JOB_CLOSE'
      IMPORTING  e_result       = ls_result
      EXCEPTIONS usage_error    = 1
                 system_error   = 2
                 internal_error = 3.

  CATCH zcx_form_error INTO DATA(lo_ex).
    " FP_JOB_CLOSE must still be called even on error to reset state
    CALL FUNCTION 'FP_JOB_CLOSE' EXCEPTIONS OTHERS = 0.
    RAISE EXCEPTION lo_ex.
ENDTRY.
```

## SFPOUTPUTPARAMS Full Field Reference

| Field | Type | Description |
|---|---|---|
| `DEST` | RSPOPNAME | Printer destination |
| `NODIALOG` | Flag | Suppress print dialog |
| `GETPDF` | Flag | Return PDF bytes |
| `PREVIEW` | Flag | Show preview |
| `PDFARCHIVE` | Flag | Archive to ArchiveLink |
| `COPIES` | Integer | Number of copies |
| `REQIMMED` | Flag | Print immediately |
| `REQDEL` | Flag | Delete spool after print |
| `LANGU` | LANGU | Output language |
| `COUNTRY` | LAND1 | Country for formatting |
| `COVERPAGE` | Flag | Print spool cover page |
| `PDLTYPE` | String | Output format (PDF/OTF) |
