# Adobe Forms Interface Reference

## Interface Overview

The form interface is the ABAP-side contract between the print program and the form layout. It defines what data the layout can see.

```
SFP → Form → Interface tab
  ├── Import          → Parameters passed from print program
  ├── Export          → Parameters returned to print program (rare)
  ├── Changing        → Bidirectional parameters (rare)
  ├── Tables          → Internal table parameters (legacy — prefer Import)
  ├── Exceptions      → Exception definitions
  └── Global Data     → Types and data available across all nodes
        ├── Types     → TYPES statements
        ├── Data      → DATA statements
        └── Field Symbols → FIELD-SYMBOLS
  └── Initialization  → ABAP code run before layout rendering
```

## Import Parameters

### Adding Parameters in SFP

```
Interface → Import → Add parameter:
  Name:          IS_HEADER
  Type:          ZEWM_DELIV_HDR_S    (structure)
  Pass by value: ✓                   (always use for forms)

  Name:          IT_ITEMS
  Type:          ZEWM_DELIV_ITEM_TAB (table type)
  Pass by value: ✓

  Name:          IV_COPIES
  Type:          I
  Default:       1
```

### Parameter Naming Conventions

| Prefix | Meaning |
|---|---|
| `IS_` | Import structure |
| `IT_` | Import table |
| `IV_` | Import scalar value |
| `IC_` | Import constant |

### Avoiding Interface Performance Issues

- Pass flat structures, not large DB result sets
- Pre-compute display values in the print program (don't do SELECT inside init code)
- Keep table parameters narrow — only fields the layout actually uses
- Break very large forms into multiple form calls if data sets exceed ~10,000 rows

## Global Data Definitions

Global data is visible in:
- Initialization code
- All layout script events (via `xfa.sourceSet` or FormCalc/JavaScript)

```abap
" In SFP → Interface → Global Data → Types:
TYPES:
  BEGIN OF gty_display_item,
    item_no    TYPE posnr,
    matnr_desc TYPE maktx,
    qty_text   TYPE char20,    " Pre-formatted quantity + UOM
    barcode    TYPE char40,
  END OF gty_display_item,
  gtt_display_items TYPE TABLE OF gty_display_item.

" In SFP → Interface → Global Data → Data:
DATA:
  gs_display_hdr   TYPE gty_display_hdr,
  gt_display_items TYPE gtt_display_items,
  gv_page_count    TYPE i.
```

## Initialization Code

Runs once before the layout renders. Use for:
- Formatting values for display (dates, UOM texts, currency)
- Building computed/derived fields
- Populating global tables from import parameters
- Language-dependent lookups

```abap
" SFP → Interface → Initialization → Code:

" Format ship date
WRITE IS_HEADER-SHIP_DATE TO GS_DISPLAY_HDR-SHIP_DATE_TEXT
  DD/MM/YYYY.

" Convert weight UOM to display text
CALL FUNCTION 'CONVERSION_EXIT_CUNIT_OUTPUT'
  EXPORTING
    input    = IS_HEADER-WEIGHT_UOM
    language = SY-LANGU
  IMPORTING
    output   = GS_DISPLAY_HDR-WEIGHT_UOM_TEXT.

" Build display items table with pre-formatted values
LOOP AT IT_ITEMS ASSIGNING FIELD-SYMBOL(<item>).
  DATA lv_qty_text TYPE char20.
  WRITE <item>-QUANTITY TO lv_qty_text.
  CONDENSE lv_qty_text.

  APPEND VALUE gty_display_item(
    item_no    = <item>-ITEM_NO
    matnr_desc = <item>-MATNR_DESC
    qty_text   = lv_qty_text && ' ' && <item>-UOM
    barcode    = <item>-HU_ID ) TO GT_DISPLAY_ITEMS.
ENDLOOP.

" Count pages (for "Page X of Y" in footer)
GV_PAGE_COUNT = LINES( IT_ITEMS ) DIV 20 + 1.
```

## Form Variants (SFPVAR)

Variants allow locale-specific layouts for the same interface:

```
SFPVAR → Create variant:
  Form:     ZEWM_DELIVERY_NOTE
  Variant:  DE                   " German locale
  Layout:   ZEWM_DELIVERY_NOTE_DE

" Print program selects variant by language:
CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
  EXPORTING
    i_name     = 'ZEWM_DELIVERY_NOTE'
    i_variant  = CASE sy-langu
                   WHEN 'D' THEN 'DE'
                   WHEN 'F' THEN 'FR'
                   ELSE '' END
  IMPORTING
    e_funcname = lv_fm_name.
```

## Context Mapping in Layout

After defining the interface, map parameters to the layout data binding:

```
SFP → Layout → Data View panel:
  Form1
    └── DataConnection
          └── IS_HEADER        (mapped to import parameter IS_HEADER)
                ├── SHIP_DATE
                ├── SHIP_TO_NAME
                └── ...
          └── IT_ITEMS[*]      (mapped to import table IT_ITEMS)
                ├── ITEM_NO
                ├── MATNR_DESC
                └── ...
```

## Common Interface Structures for EWM Documents

### Delivery Note Header

```abap
TYPES:
  BEGIN OF zewm_deliv_note_hdr,
    delivery_no   TYPE vbeln_vl,
    delivery_date TYPE wadat_ist,
    ship_to_name  TYPE name1,
    street        TYPE stras_gp,
    city          TYPE ort01,
    postal_code   TYPE pstlz,
    country       TYPE land1,
    carrier_name  TYPE name1,
    tracking_no   TYPE char30,
    total_weight  TYPE char15,       " Pre-formatted: "1,234.56 KG"
    total_volume  TYPE char15,       " Pre-formatted: "12.34 M3"
    page_count    TYPE i,
  END OF zewm_deliv_note_hdr.
```

### Warehouse Label

```abap
TYPES:
  BEGIN OF zewm_wh_label,
    hu_id         TYPE venum,        " Handling unit ID (for barcode)
    delivery_no   TYPE vbeln_vl,
    matnr         TYPE matnr,
    matnr_desc    TYPE maktx,
    batch         TYPE charg_d,
    quantity      TYPE char15,
    dest_bin      TYPE lgpla,
    dest_bin_bc   TYPE char20,       " Barcode: LGNUM+LGTYP+LGPLA
    storage_type  TYPE lgtyp,
  END OF zewm_wh_label.
```
