# Interactive Forms Reference

## Interactive vs. Static Forms

| Feature | Static (Print) | Interactive (Fillable) |
|---|---|---|
| User fills fields | No | Yes |
| Digital signatures | No | Yes |
| Submit button | No | Yes |
| Data extraction | No | Yes (POST to SAP) |
| Use case | Print/archive | Data collection, approval |

## Enabling Interactive Mode (SFPDOCPARAMS)

```abap
ls_docparams = VALUE sfpdocparams(
  langu    = sy-langu
  country  = 'US'
  fillable = abap_true    " Enable form field editing
  dynamic  = abap_false   " Static layout (set true for expanding forms)
).

CALL FUNCTION 'FP_JOB_OPEN'
  EXPORTING ie_docparams    = ls_docparams
  CHANGING  ie_outputparams = ls_outputparams.
```

## Layout: Fillable Fields in Adobe LiveCycle

```
Insert → Text Field
  Object → Field tab:
    Type: Text
    Read-Only: unchecked   " Allow user input

Insert → Check Box
  Object → Field tab:
    Export Value: "X"      " Value sent when checked

Insert → Dropdown List
  Object → Field tab:
    Items: Add each option with export value

Insert → Signature Field
  Object → Field tab:
    Type: Signature
```

## Submit Button (HTTP POST to SAP)

```
Insert → Button
  Object → Field tab:
    Control type: Submit
    Submit URL:   http://<sap_host>:<port>/sap/bc/fp_submit?sap-client=100
    Submit format: PDF (send whole PDF back to SAP)
                or XML (send data only as XML)
```

## Receiving Submitted PDF in SAP (ICF Handler)

```abap
" Activate ICF service: /sap/bc/fp_submit or custom endpoint in SICF

CLASS zcl_interactive_form_handler DEFINITION PUBLIC
  INHERITING FROM cl_http_handler.

  PUBLIC SECTION.
    METHODS handle_request REDEFINITION.
ENDCLASS.

CLASS zcl_interactive_form_handler IMPLEMENTATION.
  METHOD handle_request.
    " Get submitted PDF binary
    DATA(lv_pdf_xstring) = server->request->get_data( ).

    " Extract form data from PDF
    DATA lt_fields TYPE STANDARD TABLE OF fpformfield.

    CALL FUNCTION 'FP_GET_PDF_FORM_DATA'
      EXPORTING iv_pdf = lv_pdf_xstring
      TABLES    et_formfields = lt_fields.

    " Read specific fields by name
    DATA(lv_delivery_no) = VALUE #(
      lt_fields[ fieldname = 'DELIVERY_NO' ]-fieldvalue
      OPTIONAL ).

    DATA(lv_confirmed) = VALUE #(
      lt_fields[ fieldname = 'CONF_CHECKBOX' ]-fieldvalue
      OPTIONAL ).

    " Process confirmation
    IF lv_confirmed = 'X'.
      " ... update EWM/TM/SD ...
    ENDIF.

    server->response->set_status( code = 200 reason = 'OK' ).
  ENDMETHOD.
ENDCLASS.
```

## Extracting Form Data from Submitted PDF

```abap
" FP_GET_PDF_FORM_DATA returns field name/value pairs
DATA lt_fields TYPE STANDARD TABLE OF fpformfield.

CALL FUNCTION 'FP_GET_PDF_FORM_DATA'
  EXPORTING iv_pdf      = lv_pdf_xstring
  TABLES    et_formfields = lt_fields.

" Iterate all fields
LOOP AT lt_fields ASSIGNING FIELD-SYMBOL(<f>).
  " <f>-fieldname  = XFA field name (from layout binding path)
  " <f>-fieldvalue = Text value of the field
  " <f>-fieldtype  = CHAR / DATE / NUMERIC / SIGNATURE / etc.
ENDLOOP.
```

## Signature Field

### Layout Setup

```
Insert → Signature Field
  Name:      SIG_APPROVER
  Object → Field tab:
    Required: ✓
    Signed:   Mark as read-only when signed
```

### Checking Signature in ABAP

```abap
" After receiving submitted PDF:
DATA lt_fields TYPE TABLE OF fpformfield.
CALL FUNCTION 'FP_GET_PDF_FORM_DATA'
  EXPORTING iv_pdf     = lv_pdf_xstring
  TABLES    et_formfields = lt_fields.

LOOP AT lt_fields ASSIGNING FIELD-SYMBOL(<f>)
  WHERE fieldname = 'SIG_APPROVER'.
  IF <f>-fieldtype = 'SIGNATURE' AND <f>-fieldvalue IS NOT INITIAL.
    " Signature present — proceed with approval
  ENDIF.
ENDLOOP.
```

## Web Dynpro Integration

### Displaying Interactive Form in Web Dynpro

```abap
" Web Dynpro ABAP — display interactive Adobe Form in UI
DATA lo_form_data TYPE REF TO if_wd_context_element.

" Get InteractiveForm UI element from layout
DATA(lo_int_form) = wd_this->wd_get_interface_controller( )->get_form_node( ).

" Set form name
lo_int_form->set_pdf_source(
  VALUE wdy_ui_property( name = 'PDFFORMNAME' value = 'ZEWM_CONFIRM_FORM' ) ).

" Pass data context binding
lo_int_form->set_dataSource( '//ContextNode.FormData' ).
```

## Sending Interactive PDF by Email (for Offline Completion)

```abap
" Render interactive PDF
ls_docparams-fillable = abap_true.
ls_outputparams-getpdf = abap_true.

CALL FUNCTION 'FP_JOB_OPEN'
  EXPORTING ie_docparams    = ls_docparams
  CHANGING  ie_outputparams = ls_outputparams.

CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
  EXPORTING i_name = 'ZEWM_CONFIRM_FORM' IMPORTING e_funcname = lv_fm.

CALL FUNCTION lv_fm EXPORTING is_header = ls_hdr.

CALL FUNCTION 'FP_JOB_CLOSE'
  IMPORTING e_result = ls_result.

" Convert to email attachment
DATA lt_pdf_tab TYPE TABLE OF solisti1.
CALL FUNCTION 'SCMS_XSTRING_TO_SOLIX'
  EXPORTING buffer = ls_result-pdf
  TABLES    solix  = lt_pdf_tab.

" Send email — recipient fills and submits back to SAP endpoint
CALL FUNCTION 'SO_NEW_DOCUMENT_ATT_SEND_API1'
  EXPORTING
    document_data = VALUE sofdocchgi1(
      obj_name  = 'CONFIRM'
      obj_descr = |Please confirm delivery { lv_delivery_no }| )
  TABLES
    object_content            = VALUE #( ( line = space ) )
    receivers                 = VALUE #(
      ( receiver = lv_carrier_email rec_type = 'U' ) )
    attachment_header         = VALUE #(
      ( line = |Confirm_Delivery_{ lv_delivery_no }.pdf| ) )
    attachment_object_content = lt_pdf_tab.
```

## Dynamic Forms (Expanding Layout)

```abap
" Enable dynamic layout — form expands as user adds rows
ls_docparams = VALUE sfpdocparams(
  fillable = abap_true
  dynamic  = abap_true    " Layout expands/contracts at runtime
).
```

```
Layout → Subform:
  Type: Flowed
  Dynamic: ✓ (checked in subform properties → Binding)
  Min count: 1
  Max count: unbounded

  Insert → Button (inside subform):
    Control: Regular button
    Click event JavaScript:
      var sf = this.parent.parent;
      sf.addInstance(1);    // Add new row
```
