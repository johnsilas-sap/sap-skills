# SmartForms Style Sheets Reference

## SMARTSTYLES Overview

SMARTSTYLES defines the visual style for SmartForms — fonts, paragraph formats, character formats, tab stops, and page format. A style is referenced in the form's Global Settings.

```
SMARTSTYLES → Create Style
  Style name: ZDELIVERY_STYLE
  Description: EWM Delivery Note Style

  Components:
    ├── Header Data          (description, defaults)
    ├── Paragraph Formats    (block-level styles)
    ├── Character Formats    (inline styles, barcodes)
    ├── Default Values       (form-level font/language)
    └── Page Formats         (paper size, margins)
```

## Paragraph Formats

Each paragraph format defines layout for a block of text.

### Common Paragraph Formats

```
Paragraph: HEADING
  Font:          Arial Bold 14pt
  Alignment:     Centered
  Space above:   3mm
  Space below:   2mm
  Line spacing:  Auto

Paragraph: BODY
  Font:          Arial 10pt
  Alignment:     Left
  Space above:   0mm
  Space below:   1mm
  Line spacing:  Single

Paragraph: LABEL
  Font:          Arial Bold 9pt
  Alignment:     Left
  Space above:   0mm
  Space below:   0mm

Paragraph: TABLE_HDR
  Font:          Arial Bold 9pt  (inverted: white text / dark background via shading)
  Alignment:     Left

Paragraph: TABLE_DATA
  Font:          Arial 9pt
  Alignment:     Left
  Line spacing:  Single

Paragraph: FOOTER
  Font:          Arial 8pt
  Alignment:     Centered

Paragraph: WATERMARK
  Font:          Arial 60pt
  Color:         Light gray (RGB 200/200/200)
  Rotation:      45 degrees
  Alignment:     Centered
```

### Tab Stops in Paragraph Format

```
Paragraph: TABLE_ROW
  Tab stops:
    15mm   Left      " Item No column
    75mm   Left      " Material description column
    120mm  Right     " Quantity (right-align numbers)
    135mm  Left      " UOM
    165mm  Left      " Batch
```

## Character Formats

Character formats apply inline — within a paragraph.

### Common Character Formats

```
Character: BOLD
  Font attributes: Bold ✓

Character: HIGHLIGHT
  Font color: Red

Character: CODE128
  Output type: Barcode
  Barcode type: CODE128C
  Module width: 0.21mm
  Bar height:   15mm
  Quiet zone:   10 modules
  Interpretation line: ✓ (print human-readable text below barcode)

Character: PDF417_BC
  Output type: Barcode
  Barcode type: PDF417
  Columns: 4
  Security level: 2

Character: QR_BC
  Output type: Barcode
  Barcode type: QR Code
  Module size: 3pt
```

### Using Character Formats in Text

```
" In text element editor:
<BOLD>Important:&</>  Ship immediately.

" Barcode using character format:
<CODE128>&<FS_ITEM>-HU_ID&</>

" Highlight field in red if quantity exceeds threshold:
" (apply character format inline)
<HIGHLIGHT>&GV_EXCESS_QTY&</>
```

## Default Values

```
SMARTSTYLES → Default Values:
  Font family:   Arial
  Font size:     10pt
  Font style:    Regular
  Script:        Latin (change for CJK or Arabic)
  Language:      E (English — affects hyphenation)
```

## Page Formats

Page formats define paper size and margins. Referenced from form Global Settings.

### Standard Page Formats

```
SMARTSTYLES → Page Format: A4
  Paper size:  A4 (210mm × 297mm)
  Orientation: Portrait

SMARTSTYLES → Page Format: A4L
  Paper size:  A4 (297mm × 210mm)
  Orientation: Landscape

SMARTSTYLES → Page Format: LABEL_100X150
  Paper size:  100mm × 150mm    " Zebra/Intermec label
  Orientation: Portrait
  Margins:
    Top:    3mm
    Bottom: 3mm
    Left:   3mm
    Right:  3mm
```

### Custom Label Page Format

```
" For warehouse labels (100mm × 150mm thermal):
SMARTSTYLES → Page Format → Create
  Name:        ZLABEL_100X150
  Paper size:
    Width:     100.00mm
    Height:    150.00mm
  Margins:
    Top:        3.00mm
    Bottom:     3.00mm
    Left:       3.00mm
    Right:      3.00mm
```

## Assigning Style to Form

```
SMARTFORMS → Form → Global Settings → Form Attributes:
  Style: ZDELIVERY_STYLE    " Enter style name here
```

## Style Transport

SMARTSTYLES are separate Workbench objects — transport them independently from forms.

```
SE09 → Create transport request
  Object type: SF   (SAPscript/SmartForms)
  Object name: ZDELIVERY_STYLE

" Or: SMARTSTYLES → Utilities → Transport
```

## Common Issues

### Barcode not rendering

```
Cause:  Character format Output Type not set to "Barcode"
Fix:    SMARTSTYLES → Character Format → Output type: Barcode
        Verify barcode type matches data encoding (CODE128C = numeric pairs)

Cause:  Value contains non-encodable characters
Fix:    Strip/replace special chars in ABAP before passing to form
```

### Font not available on spool server

```
Cause:  Font defined in style not installed on print server
Fix:    Use SAP built-in fonts (Courier, Helvetica, Times)
        Or install custom font via SPAD → Font Families
```

### Style changes not reflected in active form

```
Fix:    After editing SMARTSTYLES, re-activate the form in SMARTFORMS (Ctrl+F3)
        The form embeds style definitions at activation time
```
