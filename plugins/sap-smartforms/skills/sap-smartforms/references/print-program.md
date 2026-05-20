# SmartForms Print Program Reference

## Minimal Print Program

```abap
REPORT z_print_smartform.

DATA: lv_fm_name TYPE rs38l_fnam,
      ls_control TYPE ssfctrlop,
      ls_compop  TYPE ssfcompop,
      ls_data    TYPE zmy_form_hdr.

ls_data = get_form_data( lv_key ).

ls_control-no_dialog = 'X'.

CALL FUNCTION 'SSF_FUNCTION_MODULE_NAME'
  EXPORTING  formname           = 'ZMY_FORM'
  IMPORTING  fm_name            = lv_fm_name
  EXCEPTIONS no_form            = 1
             no_function_module = 2
             OTHERS             = 3.
IF sy-subrc <> 0.  RETURN.  ENDIF.

CALL FUNCTION lv_fm_name
  EXPORTING
    control_parameters = ls_control
    output_options     = ls_compop
    is_data            = ls_data
  EXCEPTIONS OTHERS = 1.
```

## Output Scenarios

### Print to Spool (Immediate)

```abap
ls_control = VALUE ssfctrlop(
  no_dialog = 'X'
  getotf    = space ).

ls_compop = VALUE ssfcompop(
  tdprinter = 'LP01'
  tdcopies  = 1
  tdimmed   = 'X'      " Print immediately — no spool retention
  tddelete  = 'X' ).   " Delete spool after printing
```

### Print to Spool (Hold for Manual Release)

```abap
ls_compop = VALUE ssfcompop(
  tdprinter = 'LP01'
  tdcopies  = 2
  tdimmed   = space    " Hold in spool — release via SP01
  tddelete  = space    " Keep spool entry after printing
  tdlifetime = 8 ).    " Retain 8 days
```

### Retrieve OTF (No Print — for PDF conversion or email)

```abap
ls_control = VALUE ssfctrlop(
  no_dialog = 'X'
  getotf    = 'X' ).   " Return OTF instead of printing

DATA lt_otf TYPE TABLE OF itcoo.

CALL FUNCTION lv_fm_name
  EXPORTING
    control_parameters = ls_control
    output_options     = ls_compop
    is_header          = ls_header
    it_items           = lt_items
  IMPORTING
    otf_data           = lt_otf
  EXCEPTIONS OTHERS = 1.
```

### Convert OTF to PDF (XSTRING)

```abap
DATA: lt_pdf_lines TYPE TABLE OF tline,
      lv_pdf_len   TYPE i,
      lv_pdf_xstr  TYPE xstring.

CALL FUNCTION 'CONVERT_OTF'
  EXPORTING  format                = 'PDF'
  IMPORTING  bin_filesize          = lv_pdf_len
  TABLES     otf                   = lt_otf
             lines                 = lt_pdf_lines
  EXCEPTIONS err_max_linewidth     = 1
             err_format            = 2
             err_conv_not_possible = 3
             OTHERS                = 4.
IF sy-subrc <> 0.  RETURN.  ENDIF.

CALL FUNCTION 'SCMS_TLINE_TO_XSTRING'
  EXPORTING  lineslen = lv_pdf_len
  IMPORTING  buffer   = lv_pdf_xstr
  TABLES     lines    = lt_pdf_lines.
```

### Alternative: SSF_CREATE_PDF_RESULT

```abap
" Simpler: call generated FM with getotf, then use SSF helper
DATA ls_job_info TYPE ssfcrescl.

CALL FUNCTION lv_fm_name
  EXPORTING
    control_parameters = VALUE ssfctrlop( no_dialog = 'X' getotf = 'X' )
    output_options     = ls_compop
    is_header          = ls_header
    it_items           = lt_items
  IMPORTING
    job_output_info    = ls_job_info
  EXCEPTIONS OTHERS = 1.

DATA lv_pdf TYPE xstring.
CALL FUNCTION 'SSF_CREATE_PDF_RESULT'
  EXPORTING  is_result = ls_job_info
  IMPORTING  e_pdf     = lv_pdf.
```

## Multi-Document Batch (One Form per Document)

```abap
LOOP AT lt_deliveries ASSIGNING FIELD-SYMBOL(<dlv>).
  DATA(ls_hdr)  = get_delivery_header( <dlv>-docno ).
  DATA(lt_items) = get_delivery_items( <dlv>-docno ).

  CALL FUNCTION 'SSF_FUNCTION_MODULE_NAME'
    EXPORTING  formname = 'ZEWM_DELIVERY_NOTE'
    IMPORTING  fm_name  = lv_fm_name
    EXCEPTIONS OTHERS   = 1.

  CALL FUNCTION lv_fm_name
    EXPORTING
      control_parameters = ls_control
      output_options     = ls_compop
      is_header          = ls_hdr
      it_items           = lt_items
    EXCEPTIONS OTHERS = 1.
ENDLOOP.
```

## Language and Country Control

```abap
ls_control = VALUE ssfctrlop(
  no_dialog = 'X'
  langu     = 'E'    " English
  country   = 'US'   " US date/number formatting
).
```

## Sending SmartForms PDF by Email

```abap
" Get OTF, convert to XSTRING (see above), then:
DATA lt_solix TYPE TABLE OF solisti1.

CALL FUNCTION 'SCMS_XSTRING_TO_SOLIX'
  EXPORTING buffer = lv_pdf_xstr
  TABLES    solix  = lt_solix.

CALL FUNCTION 'SO_NEW_DOCUMENT_ATT_SEND_API1'
  EXPORTING
    document_data = VALUE sofdocchgi1(
      obj_name  = 'DELIV_NOTE'
      obj_descr = |Delivery Note { lv_delivery_no }|
      doc_size  = xstrlen( lv_pdf_xstr ) )
  TABLES
    object_content            = VALUE #( ( line = space ) )
    receivers                 = VALUE #(
      ( receiver = lv_email rec_type = 'U' ) )
    attachment_header         = VALUE #(
      ( line = |Delivery_Note_{ lv_delivery_no }.pdf| ) )
    attachment_object_content = lt_solix.
```

## Error Handling Pattern

```abap
CALL FUNCTION 'SSF_FUNCTION_MODULE_NAME'
  EXPORTING  formname           = lv_form_name
  IMPORTING  fm_name            = lv_fm_name
  EXCEPTIONS no_form            = 1
             no_function_module = 2
             OTHERS             = 3.
IF sy-subrc = 1.
  MESSAGE e001(zforms) WITH 'Form not found:' lv_form_name.
ELSEIF sy-subrc > 1.
  MESSAGE e002(zforms) WITH 'SSF_FUNCTION_MODULE_NAME failed' sy-subrc.
ENDIF.

CALL FUNCTION lv_fm_name
  EXPORTING
    control_parameters = ls_control
    output_options     = ls_compop
    is_header          = ls_header
    it_items           = lt_items
  EXCEPTIONS
    formatting_error   = 1
    internal_error     = 2
    send_error         = 3
    user_canceled      = 4
    OTHERS             = 5.
CASE sy-subrc.
  WHEN 1. MESSAGE e003(zforms) WITH 'Formatting error'.
  WHEN 2. MESSAGE e003(zforms) WITH 'Internal error'.
  WHEN 3. MESSAGE e003(zforms) WITH 'Send error'.
  WHEN 4. RETURN.   " User cancelled — not an error
ENDCASE.
```

## SSFCTRLOP Full Field Reference

| Field | Type | Description |
|---|---|---|
| `NO_DIALOG` | Flag | Suppress print dialog |
| `GETOTF` | Flag | Return OTF instead of printing |
| `PREVIEW` | Flag | Show print preview dialog |
| `LANGU` | LANGU | Output language |
| `COUNTRY` | LAND1 | Country for formatting |
| `UPDTASK` | Flag | Process in SAP update task |
| `STARTPAGE` | TBNAME | Override start page |

## SSFCOMPOP Full Field Reference

| Field | Type | Description |
|---|---|---|
| `TDPRINTER` | RSPOPNAME | Printer destination |
| `TDCOPIES` | RSCOPIES | Number of copies |
| `TDIMMED` | Flag | Print immediately |
| `TDDELETE` | Flag | Delete spool after print |
| `TDLIFETIME` | RSDYNNR | Spool retention days |
| `TDDEST` | RSPOPDEST | Alternative output device |
| `TDNEWID` | Flag | Force new spool entry |
| `TDTITLE` | RSPOTITLE | Spool job title |
