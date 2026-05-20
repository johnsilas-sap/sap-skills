# SmartForms Form Structure Reference

## Top-Level Nodes in SMARTFORMS Navigator

```
Form: ZEWM_DELIVERY_NOTE
  ├── Global Settings
  │     ├── Form Attributes   (style, currency/quantity formatting)
  │     └── Form Output       (output type, default printer)
  ├── Form Interface
  │     ├── Import Parameters
  │     ├── Export Parameters
  │     ├── Changing Parameters
  │     ├── Tables
  │     ├── Exceptions
  │     └── Global Definitions
  │           ├── Global Data
  │           ├── Types
  │           ├── Field Symbols
  │           ├── Initialization (ABAP code)
  │           └── Subroutines   (FORM...ENDFORM blocks)
  └── Pages and Windows
        ├── FIRST  (first page)
        │     ├── HEADER  (window — secondary)
        │     ├── MAIN    (window — main)
        │     └── FOOTER  (window — secondary)
        └── NEXT   (continuation page)
              ├── HEADER
              ├── MAIN
              └── FOOTER
```

## Global Settings

### Form Attributes

```
SMARTFORMS → Form → Global Settings → Form Attributes:
  Style:             ZDELIVERY_STYLE   (from SMARTSTYLES)
  Default currency:  EUR
  Default quantity:  ST
  Start page:        FIRST
```

### Form Output (Default Output Type)

```
Global Settings → Form Output:
  Output device (printer): LP01
  Number of copies: 1
  Print immediately: ✓ (can override in print program)
```

## Form Interface

### Import Parameters

```
Form Interface → Import:
  Name        Type                    Pass by value
  IS_HEADER   ZEWM_DELIV_HDR_S       ✓
  IT_ITEMS    ZEWM_DELIV_ITEM_TAB    ✓
  IV_COPIES   I              Default: 1
```

### Export Parameters (rare — for returning data to print program)

```
Form Interface → Export:
  Name           Type
  EV_PAGE_COUNT  I
```

### Tables (legacy — prefer Import for table parameters)

```
Form Interface → Tables:
  IT_ITEMS   ZEWM_DELIV_ITEM_TAB   (alternative to Import)
```

### Exceptions

```
Form Interface → Exceptions:
  FORMATTING_ERROR   (standard)
  INTERNAL_ERROR     (standard)
  SEND_ERROR         (standard)
  USER_CANCELED      (standard)
```

## Global Definitions

### Global Data

```abap
" Visible to all nodes, subroutines, and events in the form
DATA:
  gs_display_hdr    TYPE gty_display_hdr,
  gt_display_items  TYPE gtt_display_items,
  gv_total_qty      TYPE menge_d,
  gv_page_count     TYPE i.
```

### Types

```abap
TYPES:
  BEGIN OF gty_display_hdr,
    delivery_no    TYPE vbeln_vl,
    ship_date_text TYPE char10,    " Formatted date
    total_weight   TYPE char15,    " Pre-formatted with UOM
  END OF gty_display_hdr,

  BEGIN OF gty_display_item,
    item_no        TYPE posnr,
    matnr_desc     TYPE maktx,
    qty_text       TYPE char15,    " Formatted quantity + UOM
    barcode_val    TYPE char40,
  END OF gty_display_item,

  gtt_display_items TYPE TABLE OF gty_display_item.
```

### Initialization Code

Runs once before form rendering — use for formatting and computed fields.

```abap
" Format ship date
WRITE IS_HEADER-SHIP_DATE TO GS_DISPLAY_HDR-SHIP_DATE_TEXT DD/MM/YYYY.

" Pre-format weight
WRITE IS_HEADER-TOTAL_WEIGHT TO GV_WEIGHT_TXT LEFT-JUSTIFIED.
CONDENSE GV_WEIGHT_TXT.
GS_DISPLAY_HDR-TOTAL_WEIGHT = GV_WEIGHT_TXT && ' ' && IS_HEADER-WEIGHT_UOM.

" Build display items
LOOP AT IT_ITEMS ASSIGNING FIELD-SYMBOL(<item>).
  DATA lv_qty TYPE char15.
  WRITE <item>-QUANTITY TO lv_qty LEFT-JUSTIFIED.
  CONDENSE lv_qty.

  APPEND VALUE gty_display_item(
    item_no     = <item>-ITEM_NO
    matnr_desc  = <item>-MATNR_DESC
    qty_text    = lv_qty && ' ' && <item>-UOM
    barcode_val = <item>-HU_ID ) TO GT_DISPLAY_ITEMS.
ENDLOOP.
```

### Subroutines

```abap
" In Global Definitions → Subroutines:
" Called from within form nodes using PERFORM node

FORM get_uom_text
  USING    iv_uom  TYPE meins
           iv_lang TYPE langu
  CHANGING cv_text TYPE maktx.

  CALL FUNCTION 'CONVERSION_EXIT_CUNIT_OUTPUT'
    EXPORTING input    = iv_uom
              language = iv_lang
    IMPORTING output   = cv_text.
ENDFORM.
```

## Pages

### Page Properties

```
Insert → Page
  Name:          FIRST
  Description:   First Page
  Next page:     NEXT             (name of page to use when MAIN overflows)
  Page counter:  ✓                (include this page in count)
```

### Multi-Page Setup

```
FIRST  → Next page: NEXT
NEXT   → Next page: NEXT    (self-reference = loop until MAIN content ends)

" For last page only:
FIRST  → Next page: NEXT
NEXT   → Next page: LAST
LAST   → Next page: (empty — ends form)
```

## Windows

### Window Types

| Type | Behavior |
|---|---|
| MAIN | Scrolling — content flows to next page when full |
| Secondary | Fixed position — does not overflow; independent per page |

### Window Properties

```
Insert → Window
  Name:       HEADER
  Type:       Secondary
  Position:   Left: 0mm  Top: 0mm
  Width:      210mm
  Height:     30mm

Insert → Window
  Name:       MAIN
  Type:       MAIN
  Position:   Left: 10mm  Top: 35mm
  Width:      190mm
  Height:     220mm
```

### Assigning Windows to Pages

Each page has its own set of window assignments — a window defined in FIRST must be re-added to NEXT separately (with the same or different position).

```
FIRST page:
  HEADER  (Left:0 Top:0 W:210 H:30)
  MAIN    (Left:10 Top:35 W:190 H:220)
  FOOTER  (Left:0 Top:267 W:210 H:20)

NEXT page:
  HEADER_CONT (can use different header for continuation pages)
  MAIN        (same dimensions — MAIN content continues from FIRST)
  FOOTER      (same footer)
```
