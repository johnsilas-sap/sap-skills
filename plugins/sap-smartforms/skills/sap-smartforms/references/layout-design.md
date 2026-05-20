# SmartForms Layout Design Reference

## Element Types

| Element | Insert Via | Use For |
|---|---|---|
| Text | Insert → Text | Static labels, field output, mixed text/field |
| Table | Insert → Table | Repeating rows from internal table |
| Graphic | Insert → Graphic | Company logo, static images |
| Address | Insert → Address | Formatted postal address (SAPscript address) |
| Folder | Insert → Folder | Group elements with shared condition |
| Template | Insert → Template | Fixed grid layout (non-repeating rows) |
| PERFORM | Insert → Program Lines | Call subroutine from Global Definitions |
| Alternative | Insert → Alternative | If/else branch between two sub-nodes |
| Loop | Insert → Loop | Repeat over internal table (outside Table node) |

## Text Elements

### Static Text

```
Insert → Text
  Text: "Delivery Note"
  Style: ZDELIVERY_STYLE
  Paragraph format: HEADING   (from SMARTSTYLES)
```

### Field Output (Variable)

```
" In text editor, use &...& notation to embed field values:
Delivery No.: &IS_HEADER-DELIVERY_NO&
Ship Date:    &IS_HEADER-SHIP_DATE&(DD/MM/YYYY)
Weight:       &IS_HEADER-TOTAL_WEIGHT&
```

### Mixed Text and Field

```
" Multi-line text element with inline fields:
Shipped to: &IS_HEADER-SHIP_TO_NAME&
            &IS_HEADER-STREET&
            &IS_HEADER-CITY&, &IS_HEADER-POSTAL_CODE&
            &IS_HEADER-COUNTRY&
```

### Field Formatting Patterns

```
&FIELD&              " Default formatting
&FIELD(I)&           " Ignore leading zeros
&FIELD(Z)&           " Suppress trailing zeros
&FIELD(C)&           " Compress (remove spaces)
&FIELD(T)&           " Translate: use text for check/radio
&FIELD(<9)&          " Fixed-width, left-justified 9 chars
&FIELD(>9)&          " Fixed-width, right-justified 9 chars
&SY-PAGNO&           " Current page number
&SFSY-FORMPAGES&     " Total pages in this form call
&SFSY-FORMLINES&     " Total lines output in MAIN window
```

## Tables

### Table Node Structure

```
Table (Loop over: IT_ITEMS, using: <FS_ITEM>)
  ├── Header (prints once at top or on each new page)
  │     └── Row: "Item | Material | Qty | UOM | Batch"
  ├── Main Area (prints for each row of IT_ITEMS)
  │     └── Line Type: DATA_ROW
  │           └── Text: &<FS_ITEM>-ITEM_NO& | &<FS_ITEM>-MATNR_DESC& | ...
  └── Footer (prints once at end of table)
        └── Row: "Total: &GV_TOTAL_QTY& pieces"
```

### Table Properties

```
Table node:
  Data: Loop over IT_ITEMS
        Work area: <FS_ITEM>   (field symbol name — no angle brackets in UI)
  Line types:
    DATA_ROW: Height 5mm, fixed
    ALT_ROW:  Height 5mm, shading (for alternating row colors)

  Header: print on first page only  OR  print on each new page ✓
  Events: Table header, Table footer
```

### Column Definitions (Line Type)

```
Line type: DATA_ROW
  Cells:
    Cell 1: Width 15mm  — &<FS_ITEM>-ITEM_NO&
    Cell 2: Width 60mm  — &<FS_ITEM>-MATNR_DESC&
    Cell 3: Width 25mm  — &<FS_ITEM>-QUANTITY&  (right-aligned)
    Cell 4: Width 15mm  — &<FS_ITEM>-UOM&
    Cell 5: Width 30mm  — &<FS_ITEM>-BATCH&
    Cell 6: Width 45mm  — barcode: &<FS_ITEM>-HU_ID&
```

### Table with Subtotals

```
Table → Events → New Value (subtotal break on MATNR):
  " In event code:
  IF <FS_ITEM>-MATNR <> LV_PREV_MATNR.
    " Print subtotal row for previous material
    LV_SUBTOTAL = 0.
  ENDIF.
  LV_SUBTOTAL = LV_SUBTOTAL + <FS_ITEM>-QUANTITY.
  LV_PREV_MATNR = <FS_ITEM>-MATNR.
```

## Graphics (Company Logo)

```
Insert → Graphic
  Graphic name: COMPANY_LOGO       " Stored in transaction SE78
  Object:       GRAPHICS
  ID:           BMAP
  Type:         BMP / TIFF / GIF

" To upload graphic: SE78 → Standard Graphics → Upload BMP
```

## Address Node

```
Insert → Address
  Address type: Postal address structure
  Address data: IS_HEADER-SHIP_TO_ADDR   (type ADRC / ADRS)

  OR: Manual field binding:
    Name:    IS_HEADER-SHIP_TO_NAME
    Street:  IS_HEADER-STREET
    City:    IS_HEADER-CITY
    ZIP:     IS_HEADER-POSTAL_CODE
    Country: IS_HEADER-COUNTRY
```

## Template Node (Fixed Grid)

Unlike Table (which loops), Template is a fixed structure — no looping. Use for fixed-section layouts like document header blocks.

```
Insert → Template
  Rows:
    Row 1: Height 8mm
      Cell 1: W:40mm — "Delivery No:"
      Cell 2: W:60mm — &IS_HEADER-DELIVERY_NO&
      Cell 3: W:40mm — "Ship Date:"
      Cell 4: W:50mm — &IS_HEADER-SHIP_DATE&(DD/MM/YYYY)
    Row 2: Height 8mm
      Cell 1: W:40mm — "Ship To:"
      Cell 2: W:150mm — &IS_HEADER-SHIP_TO_NAME&
```

## Folder (Grouping with Condition)

```
Insert → Folder
  Name:      HAZMAT_SECTION
  Condition: IS_HEADER-HAZMAT_FLAG = 'X'
  Children:  (all child elements appear/disappear together)
    Text: "*** DANGEROUS GOODS — SEE REVERSE ***"
    Table: IT_DG_ITEMS
```

## Alternative (If/Else)

```
Insert → Alternative
  Condition: IS_HEADER-DOC_TYPE = 'COPY'
  True:   (branch shown when condition is true)
    Text: "COPY — NOT FOR CUSTOMS"
  False:  (branch shown when condition is false)
    Text: "ORIGINAL"
```

## Page Number and Form Pages

```
" In text element (footer):
Page &SY-PAGNO& of &SFSY-FORMPAGES&

" Total items printed:
&SFSY-TABLEROWS&    " Rows output by most recent Table node

" Running sum — use global variable updated in table event:
Total: &GV_TOTAL_QTY& &IS_HEADER-WEIGHT_UOM&
```

## Watermarks

```
" In secondary window on master page — positioned over content:
Insert → Text
  Text: "C O P Y"
  Paragraph format: WATERMARK   (define in SMARTSTYLES — large, gray, rotated)
  Condition: IS_HEADER-COPY_NO > 1
```

## Barcodes

SmartForms supports 1D barcodes natively via bar code objects:

```
" In text element, insert barcode:
Insert → Barcode → Code 128
  Value: &<FS_ITEM>-HU_ID&

" Or use SAPscript barcode character format:
" In SMARTSTYLES → Character Format:
  Output type: Barcode
  Barcode type: CODE128C
  Module width: 0.21mm (narrow bar)

" In text element:
<BC128>&<FS_ITEM>-HU_ID&</>
```
