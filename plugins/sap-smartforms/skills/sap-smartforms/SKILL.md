---
name: sap-smartforms
description: |
  SAP SmartForms development skill. Use when building or modifying SmartForms
  in transaction SMARTFORMS, writing ABAP print programs using SSF_FUNCTION_MODULE_NAME,
  designing form layouts with pages/windows/text/tables/graphics, configuring style sheets
  in SMARTSTYLES, writing conditions and events, generating PDF output, or migrating
  SmartForms to Adobe Forms. Covers delivery notes, invoices, purchase orders, warehouse
  labels, and other business document output for EWM, TM, SD, MM, and FI scenarios.
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-19"
  sources:
    - "https://help.sap.com/docs/ABAP_PLATFORM_NEW/4a368c163b08418890a406d413933ba7"
---

# SAP SmartForms Development Skill

## Related Skills

- **sap-adobe-forms**: Adobe ADS forms — modern successor to SmartForms
- **sap-abap**: ABAP for print programs — internal tables, structures, string processing
- **sap-ewm**: EWM-triggered forms — delivery notes, warehouse labels, TO slips
- **sap-tm**: TM-triggered forms — freight documents, dangerous goods declarations

## Architecture Overview

```
ABAP Print Program
  → SSF_FUNCTION_MODULE_NAME (get generated FM name)
  → Generated FM call (pass interface data)
      → SmartForms engine renders OTF (Output Text Format)
  → OTF → spool (print) or CONVERT_OTFTOPDF (PDF bytes)
```

SmartForms runs entirely in ABAP — no Java server required (unlike Adobe Forms).

## Form Structure (SMARTFORMS)

```
Form (SMARTFORMS)
  ├── Global Settings    — form attributes, style, output type
  ├── Form Interface     — ABAP-side: imports, exports, tables, exceptions, globals
  │     ├── Import parameters
  │     ├── Export parameters
  │     ├── Tables
  │     ├── Exceptions
  │     └── Global Definitions (types, data, subroutines)
  └── Form               — Visual layout
        ├── Pages              (page definitions with windows)
        │     └── Windows      (positioned content areas on each page)
        │           └── Elements (text, table, graphic, address, folder)
        ├── Flow Logic         (table events, page breaks, conditions)
        └── Subroutines        (ABAP code blocks reusable within form)
```

## Creating a Form (SMARTFORMS)

```
SMARTFORMS → Create
  Form name:   ZEWM_DELIVERY_NOTE
  Description: EWM Delivery Note

  Global Settings tab:
    Form Attributes:
      Style:       ZDELIVERY_STYLE     (from SMARTSTYLES)
      Start page:  FIRST               (name of first page)

  Form Interface tab:
    Import → Add:
      Name: IS_HEADER   Type: ZEWM_DELIV_HDR_S   Pass by value ✓
      Name: IT_ITEMS    Type: ZEWM_DELIV_ITEM_TAB Pass by value ✓
```

## ABAP Print Program Pattern

### Complete Print Program

```abap
REPORT z_print_ewm_delivery_note.

DATA: lv_fm_name  TYPE rs38l_fnam,
      ls_header   TYPE zewm_deliv_hdr_s,
      lt_items    TYPE zewm_deliv_item_tab,
      ls_control  TYPE ssfctrlop,
      ls_compop   TYPE ssfcompop.

" 1. Populate form data
PERFORM fill_delivery_data CHANGING ls_header lt_items.

" 2. Set control options (output behavior)
ls_control = VALUE ssfctrlop(
  no_dialog   = 'X'         " No print dialog popup
  preview     = space       " No preview
  getotf      = space       " Output to spool (not OTF retrieval)
).

" 3. Set composer options (printer/copies)
ls_compop = VALUE ssfcompop(
  tdprinter = 'LP01'        " Printer from SPAD
  tdcopies  = 1
  tdimmed   = 'X'           " Immediate print
).

" 4. Get generated function module name
CALL FUNCTION 'SSF_FUNCTION_MODULE_NAME'
  EXPORTING  formname     = 'ZEWM_DELIVERY_NOTE'
  IMPORTING  fm_name      = lv_fm_name
  EXCEPTIONS no_form      = 1
             no_function_module = 2
             OTHERS       = 3.
IF sy-subrc <> 0.
  MESSAGE 'Form not found or not activated' TYPE 'E'.
ENDIF.

" 5. Call generated FM
CALL FUNCTION lv_fm_name
  EXPORTING
    control_parameters = ls_control
    output_options     = ls_compop
    user_settings      = space
    is_header          = ls_header
    it_items           = lt_items
  EXCEPTIONS
    formatting_error   = 1
    internal_error     = 2
    send_error         = 3
    user_canceled      = 4
    OTHERS             = 5.
IF sy-subrc <> 0.
  MESSAGE 'Error calling form' TYPE 'E'.
ENDIF.
```

### Retrieve OTF and Convert to PDF

```abap
" Retrieve OTF instead of printing
ls_control = VALUE ssfctrlop(
  no_dialog = 'X'
  getotf    = 'X'   " Return OTF data instead of spooling
).

DATA lt_otf TYPE TABLE OF itcoo.

CALL FUNCTION lv_fm_name
  EXPORTING
    control_parameters = ls_control
    output_options     = ls_compop
    is_header          = ls_header
    it_items           = lt_items
  IMPORTING
    otf_data           = lt_otf    " OTF output
  EXCEPTIONS OTHERS = 1.

" Convert OTF to PDF
DATA: lt_lines    TYPE TABLE OF tline,
      lv_pdf_len  TYPE i,
      lv_pdf_xstr TYPE xstring.

CALL FUNCTION 'CONVERT_OTF'
  EXPORTING  format                = 'PDF'
  IMPORTING  bin_filesize          = lv_pdf_len
  TABLES     otf                   = lt_otf
             lines                 = lt_lines
  EXCEPTIONS err_max_linewidth     = 1
             err_format            = 2
             err_conv_not_possible = 3
             OTHERS                = 4.

" Convert lines to XSTRING for email/storage
CALL FUNCTION 'SCMS_TLINE_TO_XSTRING'
  EXPORTING  lineslen = lv_pdf_len
  IMPORTING  buffer   = lv_pdf_xstr
  TABLES     lines    = lt_lines.
```

### SSFCTRLOP Field Reference

| Field | Description |
|---|---|
| `NO_DIALOG` | Suppress print dialog (`'X'` = suppress) |
| `GETOTF` | Return OTF data instead of printing |
| `PREVIEW` | Show print preview |
| `LANGU` | Form language (default: logon language) |
| `COUNTRY` | Country for date/number formatting |
| `UPDTASK` | Process in update task |

### SSFCOMPOP Field Reference

| Field | Description |
|---|---|
| `TDPRINTER` | Printer destination (from SPAD) |
| `TDCOPIES` | Number of copies |
| `TDIMMED` | Immediate print (`'X'`) |
| `TDDEST` | Alternative: output device |
| `TDDELETE` | Delete spool after printing |
| `TDLIFETIME` | Spool retention time (days) |

## Form Interface Design

### Import Parameters Best Practices

```abap
" Clean flat structures — avoid raw DB tables
TYPES:
  BEGIN OF zewm_deliv_hdr_s,
    delivery_no  TYPE vbeln_vl,
    ship_date    TYPE wadat_ist,
    ship_to_name TYPE name1,
    city         TYPE ort01,
    country      TYPE land1,
    total_weight TYPE char15,     " Pre-formatted: "1,234.56 KG"
    carrier_name TYPE name1,
  END OF zewm_deliv_hdr_s,

  BEGIN OF zewm_deliv_item_s,
    item_no      TYPE posnr,
    matnr_desc   TYPE maktx,
    quantity     TYPE char15,     " Pre-formatted quantity + UOM
    batch        TYPE charg_d,
    hu_id        TYPE venum,
  END OF zewm_deliv_item_s,

  zewm_deliv_item_tab TYPE TABLE OF zewm_deliv_item_s.
```

### Global Definitions — ABAP Subroutines

```abap
" In Global Definitions → ABAP Subroutines:
FORM format_weight
  USING    iv_weight TYPE ntgew
           iv_uom    TYPE gewei
  CHANGING cv_text   TYPE char15.
  WRITE iv_weight TO cv_text LEFT-JUSTIFIED.
  CONDENSE cv_text.
  cv_text = cv_text && ' ' && iv_uom.
ENDFORM.
```

## Layout Design

### Pages

Each page has a name and links to a "next page" for multi-page output:

```
FIRST  → next page: NEXT
NEXT   → next page: NEXT  (loops for additional pages)
```

### Windows

Windows are positioned areas on a page:

| Window Type | Purpose |
|---|---|
| MAIN | Scrolling content — overflows to next page |
| Secondary | Fixed content — header, footer, address block |

### Tables in Main Window

```
Table node → Loop over: IT_ITEMS
  Table header (prints once or on each page):
    Row: "Item | Material | Qty | UOM"
  Table line (prints for each IT_ITEMS row):
    Row: ITEM_NO | MATNR_DESC | QUANTITY | UOM
  Table footer (prints at end):
    Row: "Total: " & SUM_QTY
```

### Conditions

```abap
" Condition on a node (e.g. show HAZMAT window only if flagged):
IS_HEADER-HAZMAT_FLAG = 'X'

" Condition with ABAP expression:
" In condition tab, select ABAP expression:
IS_HEADER-DOC_TYPE = 'OUTBD' AND IS_HEADER-COPIES > 1
```

## Key Transactions

| Transaction | Purpose |
|---|---|
| `SMARTFORMS` | Form Builder — create/edit forms |
| `SMARTSTYLES` | Style editor — paragraph/character formats |
| `SF62` | SmartForms administration |
| `SP01` | Spool monitor |
| `SPAD` | Printer definitions |
| `SSF_TEST` | Test SmartForms rendering |
