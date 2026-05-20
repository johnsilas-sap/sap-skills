---
name: sap-adobe-forms
description: |
  SAP Adobe Forms (ADS) development skill. Use when building or modifying Adobe forms
  in SFP (Form Builder), writing ABAP print programs using FP_JOB_OPEN/FP_JOB_CLOSE,
  designing form interfaces and context mappings, scripting forms with JavaScript (XFA),
  configuring ADS connections, printing via PPF, generating PDF output for delivery
  notes, labels, invoices, GR slips, warehouse documents, or handling interactive PDF
  forms with user input. Covers both print and interactive form scenarios.
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-19"
  sources:
    - "https://help.sap.com/docs/ABAP_PLATFORM_NEW/a996ed769f474ef78e6b35e03f4bffe0"
    - "https://help.sap.com/docs/SAP_NETWEAVER"
---

# SAP Adobe Forms Development Skill

## Related Skills

- **sap-abap**: ABAP for print programs — internal tables, structures, string processing
- **sap-ewm**: EWM-triggered forms — delivery notes, warehouse labels, TO confirmation slips
- **sap-tm**: TM-triggered forms — freight documents, dangerous goods declarations, CMR
- **sap-smartforms**: Alternative form technology — migration patterns from SmartForms to Adobe

## Architecture Overview

```
ABAP Print Program
  → FP_JOB_OPEN (open ADS job)
  → FP_FUNCTION_MODULE_NAME (get generated FM name)
  → Generated FM call (pass interface data)
      → ADS (Adobe Document Services) renders PDF
  → FP_JOB_CLOSE (finalize, output to printer/spool/email)
```

ADS runs as a Java application — on SAP NetWeaver AS Java or as a separate ADS server.
The connection is configured in SICF and tested via transaction `SFPADM`.

## Form Structure (SFP)

Each Adobe Form has two parts, edited separately in SFP:

```
Form (SFP)
  ├── Interface        — ABAP-side: imports, globals, tables, field symbols
  │     ├── Import parameters  (data passed into the form)
  │     ├── Global definitions (types and data visible in layout)
  │     └── Initialization code (ABAP executed before layout)
  └── Layout           — Adobe LiveCycle Designer: visual design
        ├── Master pages      (headers, footers, page numbering)
        ├── Subforms          (repeating rows, conditional sections)
        ├── Fields            (text, barcode, image, signature)
        └── Scripts           (JavaScript for field behavior)
```

## Creating a Form (SFP)

```
SFP → Create
  Form name:    ZEWM_DELIVERY_NOTE
  Description:  EWM Delivery Note

  Interface tab:
    Import → Add parameter:
      Name:     IS_HEADER      Type: ZEWM_DELIV_HDR_S   Pass by value
      Name:     IT_ITEMS       Type: ZEWM_DELIV_ITEM_TAB Pass by value
      Name:     IS_ADDRESS     Type: ADRS               Pass by value

  Layout tab → opens Adobe LiveCycle Designer
```

## ABAP Print Program Pattern

### Complete Print Program

```abap
REPORT z_print_ewm_delivery_note.

DATA: lv_fm_name   TYPE rs38l_fnam,
      ls_docparams TYPE sfpdocparams,
      ls_outputparams TYPE sfpoutputparams,
      ls_header    TYPE zewm_deliv_hdr_s,
      lt_items     TYPE zewm_deliv_item_tab.

" 1. Populate form data
PERFORM fill_delivery_data
  CHANGING ls_header lt_items.

" 2. Set output parameters
ls_outputparams = VALUE sfpoutputparams(
  nodialog  = abap_true           " No print dialog popup
  getpdf    = abap_true           " Return PDF in memory
  dest      = lv_printer_name ).  " Spool destination

" 3. Open form processing job
CALL FUNCTION 'FP_JOB_OPEN'
  CHANGING
    ie_outputparams = ls_outputparams
  EXCEPTIONS
    cancel          = 1
    usage_error     = 2
    system_error    = 3
    internal_error  = 4.
IF sy-subrc <> 0.
  MESSAGE 'Error opening form job' TYPE 'E'.
ENDIF.

" 4. Get generated function module name
CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
  EXPORTING
    i_name     = 'ZEWM_DELIVERY_NOTE'
  IMPORTING
    e_funcname = lv_fm_name.

" 5. Call generated FM (interface matches form parameters exactly)
CALL FUNCTION lv_fm_name
  EXPORTING
    is_header    = ls_header
    it_items     = lt_items
  EXCEPTIONS
    usage_error  = 1
    system_error = 2
    internal_error = 3.
IF sy-subrc <> 0.
  MESSAGE 'Error calling form FM' TYPE 'E'.
ENDIF.

" 6. Close job and retrieve PDF
DATA: ls_result  TYPE sfpjoboutput,
      lv_pdf     TYPE xstring.

CALL FUNCTION 'FP_JOB_CLOSE'
  IMPORTING
    e_result    = ls_result
  EXCEPTIONS
    usage_error = 1
    system_error = 2
    internal_error = 3.

lv_pdf = ls_result-pdf.            " Raw PDF bytes — use for email/storage
```

### Output Parameters Reference (SFPOUTPUTPARAMS)

| Field | Type | Description |
|---|---|---|
| `DEST` | Printer name | Spool destination |
| `NODIALOG` | Flag | Suppress print dialog |
| `GETPDF` | Flag | Return PDF in `FP_JOB_CLOSE` result |
| `PDFARCHIVE` | Flag | Archive PDF to SAP ArchiveLink |
| `PREVIEW` | Flag | Open PDF preview in browser |
| `COPIES` | Integer | Number of copies |
| `REQIMMED` | Flag | Print immediately (skip spool) |
| `LANGU` | Language | Form language (default: logon language) |
| `COUNTRY` | Country | For locale-specific number/date formatting |

## Document Parameters (SFPDOCPARAMS)

```abap
ls_docparams = VALUE sfpdocparams(
  langu   = sy-langu                " Form language
  country = 'US'                    " Country for formatting
  fillable = abap_false             " Static PDF (not interactive)
  dynamic  = abap_false ).          " Static layout (no dynamic expansion)
```

Pass `ls_docparams` to `FP_JOB_OPEN` via `ie_docparams` parameter for language/country control.

## Form Interface Design

### Import Parameter Best Practices

```abap
" Define clean interface structures — avoid passing entire DB tables
TYPES:
  BEGIN OF zewm_deliv_hdr_s,
    delivery_no  TYPE vbeln_vl,
    ship_date    TYPE wadat_ist,
    ship_to_name TYPE name1,
    ship_to_addr TYPE adrs,
    total_weight TYPE ntgew,
    weight_uom   TYPE gewei,
    carrier_name TYPE name1,
    tracking_no  TYPE ztracking,
  END OF zewm_deliv_hdr_s,

  BEGIN OF zewm_deliv_item_s,
    item_no      TYPE posnr,
    matnr        TYPE matnr,
    matnr_desc   TYPE maktx,
    quantity     TYPE menge_d,
    uom          TYPE meins,
    batch        TYPE charg_d,
    hu_id        TYPE venum,
    barcode      TYPE zbarcode,
  END OF zewm_deliv_item_s,

  zewm_deliv_item_tab TYPE TABLE OF zewm_deliv_item_s.
```

### Initialization Code (Interface ABAP Tab)

Runs before layout rendering — use for computed fields, formatting:

```abap
" In SFP Interface → Initialization tab:
" Convert unit of measure for display
CALL FUNCTION 'CONVERSION_EXIT_CUNIT_OUTPUT'
  EXPORTING
    input  = IS_HEADER-WEIGHT_UOM
    language = SY-LANGU
  IMPORTING
    output = GS_DISPLAY-WEIGHT_UOM_TEXT.

" Format date for form locale
WRITE IS_HEADER-SHIP_DATE TO GS_DISPLAY-SHIP_DATE_TEXT DD/MM/YYYY.
```

## Layout Design (Adobe LiveCycle Designer)

### Subform for Repeating Items (Table Rows)

```
Layout:
  Subform: ItemTable (Flowed, Repeat for each data item)
    Bind to: $record.IT_ITEMS[*]
    Row fields:
      ItemNo    → bind: ITEM_NO
      Material  → bind: MATNR_DESC
      Quantity  → bind: QUANTITY
      UOM       → bind: UOM
      Batch     → bind: BATCH
      Barcode   → bind: BARCODE (barcode field type)
```

### Barcode Fields

```
Field type: Barcode
Barcode type: Code128 / QR Code / PDF417 / EAN-13
Bind to data node containing the barcode value string
```

For EWM labels, common barcode types:
- **Code128** — serial numbers, HU IDs
- **QR Code** — delivery/TO references with multiple fields
- **PDF417** — dangerous goods labels (UN number, quantity, class)

### Conditional Section (Show/Hide)

```javascript
// In subform's Initialize event (JavaScript):
if (IS_HEADER.HAZMAT_FLAG.value == "X") {
    this.presence = "visible";
} else {
    this.presence = "hidden";
}
```

### Page Break Control

```javascript
// Force page break before a subform:
// Subform properties → Pagination → Before: Start new page
// OR in script:
xfa.layout.relayout();
```

### Running Total / Aggregation

```javascript
// In field Calculate event:
var total = 0;
var items = xfa.resolveNodes("IT_ITEMS[*].QUANTITY");
for (var i = 0; i < items.length; i++) {
    total += parseFloat(items.item(i).value) || 0;
}
this.rawValue = total;
```

## PPF Integration (Print via Post Processing Framework)

Used to trigger forms automatically on business events (delivery saving, TO completion, etc.).

### PPF Action Class for Adobe Form

```abap
CLASS zcl_ppf_ewm_delivery_note DEFINITION PUBLIC.
  PUBLIC SECTION.
    INTERFACES if_fpf_action.
ENDCLASS.

CLASS zcl_ppf_ewm_delivery_note IMPLEMENTATION.

  METHOD if_fpf_action~execute.
    DATA: lv_fm_name      TYPE rs38l_fnam,
          ls_outputparams TYPE sfpoutputparams,
          ls_header       TYPE zewm_deliv_hdr_s,
          lt_items        TYPE zewm_deliv_item_tab.

    " Determine document key from PPF context
    DATA(lv_docno) = io_appl_object->get_key( ).

    " Fill form data
    SELECT SINGLE * FROM /scdl/db_proci_o
      WHERE docno = @lv_docno
      INTO @DATA(ls_dlv).

    " ... populate ls_header and lt_items ...

    " Print
    ls_outputparams = VALUE sfpoutputparams(
      nodialog = abap_true
      dest     = 'LP01' ).

    CALL FUNCTION 'FP_JOB_OPEN'
      CHANGING ie_outputparams = ls_outputparams.

    CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
      EXPORTING i_name    = 'ZEWM_DELIVERY_NOTE'
      IMPORTING e_funcname = lv_fm_name.

    CALL FUNCTION lv_fm_name
      EXPORTING
        is_header = ls_header
        it_items  = lt_items.

    CALL FUNCTION 'FP_JOB_CLOSE'.
  ENDMETHOD.

ENDMETHOD.
```

## Email PDF Output

```abap
" Get PDF bytes from FP_JOB_CLOSE
DATA ls_result TYPE sfpjoboutput.
CALL FUNCTION 'FP_JOB_CLOSE'
  IMPORTING e_result = ls_result.

" Send as email attachment
CALL FUNCTION 'SO_NEW_DOCUMENT_ATT_SEND_API1'
  EXPORTING
    document_data = VALUE sofdocchgi1(
      obj_descr = 'Delivery Note'
      obj_name  = 'PDF' )
  TABLES
    object_content    = VALUE #( )
    receivers         = VALUE #(
      ( receiver = lv_email_address rec_type = 'U' ) )
    attachment_header = VALUE #(
      ( line = 'Delivery_Note.pdf' ) )
    attachment_object_content = VALUE #(
      " Convert xstring to solix table
      ( line = ls_result-pdf ) ).
```

## ADS Connection and Administration

```
SFPADM → Adobe Document Services Administration
  → Test connection to ADS server
  → Check ADS version
  → Configure PDF rendering settings

SICF → default_host/sap/bc/fp
  → Adobe Forms ICF service (must be active)

SM59 → ADS RFC destination
  Type: HTTP connection to external server
  Target: ADS Java server host/port
```

### Test ADS Connection

```abap
CALL FUNCTION 'FP_CHECK_DESTINATION'
  EXCEPTIONS
    no_destination = 1
    destination_error = 2.
IF sy-subrc <> 0.
  " ADS not reachable — check SM59 and SICF
ENDIF.
```

## Key Transactions

| Transaction | Purpose |
|---|---|
| `SFP` | Form Builder — create/edit forms and interfaces |
| `SFPADM` | ADS administration and connection test |
| `SFPDEV` | Developer desktop for local Adobe LiveCycle |
| `SFPVAR` | Form variants — locale-specific layouts |
| `SP01` | Spool monitor — view generated print jobs |
| `SICF` | ICF service for Adobe Forms (`/sap/bc/fp`) |
| `SM59` | RFC destination to ADS server |
