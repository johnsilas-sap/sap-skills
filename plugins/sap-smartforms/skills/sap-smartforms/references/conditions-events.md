# Conditions and Events Reference

## Conditions

Every SmartForms element (window, text, table, folder, etc.) can have a condition that controls whether it renders.

### Condition Types

| Type | Example |
|---|---|
| Field comparison | `IS_HEADER-HAZMAT_FLAG = 'X'` |
| Field IS INITIAL | `IS_HEADER-BATCH IS INITIAL` |
| Field IS NOT INITIAL | `IS_HEADER-TRACKING_NO IS NOT INITIAL` |
| ABAP expression | Complex logic with AND/OR/NOT |

### Setting a Condition

```
Select element → Object Properties → Condition tab:
  Activate condition: ✓
  Condition type:     ABAP Expression

  Expression:
    IS_HEADER-HAZMAT_FLAG = 'X'

  " Multi-condition:
    IS_HEADER-DOC_TYPE = 'OUTBD'
    AND IS_HEADER-COPY_NO > 1

  " Using global variable set in initialization:
    GV_IS_DANGEROUS_GOODS = ABAP_TRUE
```

### Condition on a Window

```
" Show HAZMAT window only on dangerous goods deliveries
Window: HAZMAT_INFO
  Condition: IS_HEADER-HAZMAT_FLAG = 'X'
```

### Condition on a Table Row (Line Type)

```
" Show alternate row background for even rows:
Line Type: ALT_ROW
  Condition: SY-TABIX MOD 2 = 0

" Show special handling line only if batch exists:
Line Type: BATCH_ROW
  Condition: <FS_ITEM>-BATCH IS NOT INITIAL
```

### Condition on a Text Element

```
" Show 'COPY' watermark only on copies
Text: WATERMARK_TEXT
  Condition: IS_HEADER-COPY_NO > 1
```

## Table Events

Events run ABAP code at specific points during table rendering.

### Available Table Events

| Event | Fires |
|---|---|
| Table Header | Before first table row (or each page if header repeats) |
| Table Footer | After last table row |
| New Value | When a sort key field changes value (subtotal break) |
| Last Value | On the last occurrence of a sort key group |
| Before Output | Before each table row is output |
| After Output | After each table row is output |

### Table Footer with Totals

```abap
" Table → Events → Table Footer → ABAP code:
GV_TOTAL_QTY    = 0.
GV_TOTAL_WEIGHT = 0.

LOOP AT IT_ITEMS ASSIGNING FIELD-SYMBOL(<item>).
  GV_TOTAL_QTY    = GV_TOTAL_QTY    + <item>-QUANTITY.
  GV_TOTAL_WEIGHT = GV_TOTAL_WEIGHT + <item>-WEIGHT.
ENDLOOP.
```

### Subtotal Break (New Value Event)

```abap
" Table → Events → New Value → Sort field: MATNR

" Event code runs when MATNR changes:
" Print subtotal row for completed group
GV_GROUP_SUBTOTAL = 0.
LOOP AT IT_ITEMS ASSIGNING FIELD-SYMBOL(<i>) WHERE MATNR = LV_PREV_MATNR.
  GV_GROUP_SUBTOTAL = GV_GROUP_SUBTOTAL + <i>-QUANTITY.
ENDLOOP.
" GV_GROUP_SUBTOTAL now available for subtotal row output
LV_PREV_MATNR = <FS_ITEM>-MATNR.
```

### Before Output Event (Row-Level Logic)

```abap
" Table → Events → Before Output → ABAP code:
" Runs before each row — use to set display flags

IF <FS_ITEM>-QUANTITY > 100.
  GV_ROW_STYLE = 'HIGHLIGHT'.   " Switch to highlighted line type
ELSE.
  GV_ROW_STYLE = 'NORMAL'.
ENDIF.

" Format quantity for display
WRITE <FS_ITEM>-QUANTITY TO GV_QTY_TEXT LEFT-JUSTIFIED.
CONDENSE GV_QTY_TEXT.
GV_QTY_DISPLAY = GV_QTY_TEXT && ' ' && <FS_ITEM>-UOM.
```

## Page Break Events

### Forcing a Page Break

```
" In window/element properties:
Pagination → Page break before: ✓

" Conditional page break using Alternative element:
Alternative
  Condition: <FS_ITEM>-FORCE_BREAK = 'X'
  True:  (empty element with "New page" pagination)
  False: (empty — no break)
```

### Before Page Break Event (Table)

```abap
" Table → Events → Before Output (page-related):
" Use SFSY-NEWPAGE system field to detect page change
IF SFSY-NEWPAGE = ABAP_TRUE.
  " First row on a new page — reset counters
  GV_PAGE_ITEM_COUNT = 0.
ENDIF.
```

## PERFORM Node (Calling Subroutines)

```
Insert → Program Lines → PERFORM
  Subroutine:  format_batch_no   (defined in Global Definitions → Subroutines)
  Parameters:
    USING:    <FS_ITEM>-BATCH    (pass field as input)
    CHANGING: GV_BATCH_DISPLAY   (receive formatted output)
```

```abap
" In Global Definitions → Subroutines:
FORM format_batch_no
  USING    iv_batch TYPE charg_d
  CHANGING cv_out   TYPE char18.

  cv_out = iv_batch.
  CONDENSE cv_out NO-GAPS.
  " Pad to 18 chars for barcode minimum width
  DO 18 - strlen( cv_out ) TIMES.
    cv_out = cv_out && '0'.
  ENDDO.
ENDFORM.
```

## Alternative Element (If/Else Layout Branching)

```
Alternative
  Condition: IS_HEADER-DOC_TYPE = 'INTERNATIONAL'
  True branch:
    Window: INTL_HEADER   (international header with customs info)
  False branch:
    Window: DOM_HEADER    (domestic header — simpler)
```

## Loop Element (Outside Table)

```
" Repeat a block of elements for each row of an internal table
Insert → Loop
  Table: IT_HAZMAT_ITEMS
  Work area: <FS_HM>
  Condition: IS_HEADER-HAZMAT_FLAG = 'X'

  Children:
    Text: UN No: &<FS_HM>-UN_NUMBER& Class: &<FS_HM>-HAZ_CLASS&
```

## SmartForms System Fields (SFSY)

| Field | Value |
|---|---|
| `SFSY-PAGE` | Current page number |
| `SFSY-FORMPAGES` | Total pages in this form call |
| `SFSY-FORMLINES` | Total lines output in MAIN window |
| `SFSY-TABLEROWS` | Rows output by most recent Table |
| `SFSY-NEWPAGE` | True if this is the first row on a new page |
| `SFSY-MAINEND` | True if MAIN window has ended (last page) |
| `SFSY-WINDOWNAME` | Name of current window |
| `SFSY-PAGENAME` | Name of current page |

```
" Using system fields in text elements:
Page &SFSY-PAGE& of &SFSY-FORMPAGES&

" Conditional element on last page only:
Condition: SFSY-MAINEND = ABAP_TRUE
```
