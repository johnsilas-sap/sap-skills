# Adobe Forms Layout Design Reference

## Adobe LiveCycle Designer Basics

Opened from SFP → Layout tab. The layout is an XFA (XML Forms Architecture) document stored in SAP as part of the form object.

### Key Panels

| Panel | Purpose |
|---|---|
| Design View | Visual canvas — drag and drop fields |
| Data View | Shows form data nodes (mapped from interface) |
| Object | Properties for selected field/subform |
| Script Editor | JavaScript/FormCalc for field events |
| Hierarchy | Tree of all subforms and fields |
| Report | Errors and warnings |

## Master Pages

Master pages define repeating page elements: headers, footers, background.

```
Insert → Master Page
  Content areas define where body content flows
  Header/footer areas are fixed per page

Page numbering in footer:
  Text field: "Page " & Page() & " of " & Count(PageAll())
  Script language: FormCalc (easier for page functions)
```

## Body Pages

One or more body pages define the main content area. Content flows from body page to body page when it overflows.

```
Insert → Body Page
  Assign to Master Page (for header/footer)
  Set content area bounds
```

## Subforms

Subforms are the key layout construct — everything is a subform.

### Static Subform (Fixed Content)

```
Insert → Subform
  Type: Positioned (fixed layout)
  Used for: headers, address blocks, single-occurrence sections
```

### Repeating Subform (Table Rows)

```
Insert → Subform
  Type: Flowed
  Repeat: for each data item ✓
  Overflow: create new page ✓

  Bind to: IT_ITEMS[*]   (data node for the items table)

  Add child subform for each row:
    Fields bound to: ITEM_NO, MATNR_DESC, QTY_TEXT, BARCODE
```

### Conditional Subform (Show/Hide)

```javascript
// In subform's Initialize event:
if (IS_HEADER.HAZMAT_FLAG.rawValue == "X") {
    this.presence = "visible";
} else {
    this.presence = "hidden";
}
```

## Fields

### Text Field

```
Insert → Text Field
  Bind to: IS_HEADER.SHIP_TO_NAME

  Object → Field tab:
    Type: Text
    Locale: same as document

  Object → Value tab:
    Type: Text
    Default: (blank)
```

### Numeric Field with Formatting

```
Insert → Numeric Field
  Bind to: IS_HEADER.TOTAL_WEIGHT

  Object → Field tab:
    Type: Numeric
    Locale: de_DE (for European formatting)

  Object → Format tab:
    Type: Numeric
    Pattern: #,###.## (thousands separator, 2 decimals)
```

### Date Field

```
Insert → Date/Time Field
  Bind to: IS_HEADER.SHIP_DATE

  Object → Format tab:
    Type: Date
    Pattern: DD.MM.YYYY  (or MM/DD/YYYY for US)
```

### Image Field

```
Insert → Image Field
  Bind to: IS_HEADER.COMPANY_LOGO   (XSTRING data from ABAP)

  " In ABAP interface — pass logo as base64 or raw binary:
  DATA ls_hdr TYPE zmy_form_hdr.
  CALL FUNCTION 'SSFC_GET_GRAPHIC'
    EXPORTING objid = 'COMPANY_LOGO'
    IMPORTING content = ls_hdr-company_logo_xstr.
```

## Barcodes

### Code 128 (Serial Numbers, HU IDs)

```
Insert → Barcode
  Barcode type: Code 128
  Bind to: IT_ITEMS.HU_ID

  Object → Field tab:
    Presence: Visible
    Module width: 1.5pt (adjust for scanner readability)

" Best practice: strip special characters in ABAP before passing
DATA(lv_barcode_val) = condense( <item>-hu_id ).
```

### QR Code (Multi-Field Reference)

```
Insert → Barcode
  Barcode type: QR Code
  Error correction: M (Medium)
  Bind to: computed field

" Build QR code value in ABAP initialization:
DATA(lv_qr_data) = |{ IS_HEADER-DELIVERY_NO }|
               && |^{ IS_HEADER-SHIP_DATE }|
               && |^{ IS_HEADER-TOTAL_WEIGHT }|.
GS_DISPLAY-QR_CODE_VALUE = lv_qr_data.
```

### PDF417 (Dangerous Goods Labels)

```
Insert → Barcode
  Barcode type: PDF417
  Columns: 4
  Rows: auto

" For DG labels — encode UN number + class + quantity:
GS_DISPLAY-DG_BARCODE = |UN{ IS_DG-UN_NUMBER }|
                      && |;{ IS_DG-HAZ_CLASS }|
                      && |;{ IS_DG-PACK_GROUP }|
                      && |;{ IS_DG-QUANTITY } { IS_DG-UOM }|.
```

## Tables

```
Insert → Table
  Rows: Header row + Detail row
  Columns: Item, Material, Description, Qty, UOM, Batch

  Detail row → subform → Flowed → Repeat for each data item
  Bind detail row to: IT_ITEMS[*]

  Overflow: Table continues on next page automatically
            Add header row repeat on new page:
              Table properties → "Repeat header rows on each new page" ✓
```

## Page Numbering and Counts

```
" FormCalc (recommended for page functions):
" In text field → Script Editor → Calculate:

" Page X of Y:
"Page " & Page() & " of " & Count(PageAll())

" Total items count (in footer):
Count(IT_ITEMS[*].ITEM_NO)

" Running total:
Sum(IT_ITEMS[*].QUANTITY)
```

## Watermarks

```
Insert → Text on master page
  Text: "COPY" or "DRAFT"
  Position: Center of page
  Rotate: 45 degrees
  Transparency: 30%
  Color: Light gray

" Make conditional (show only for copies > 1):
// Initialize event:
if (xfa.form.form1.#subform[0].COPY_NO.rawValue > 1) {
    this.presence = "visible";
} else {
    this.presence = "hidden";
}
```

## Overflow Handling

```
Subform properties → Overflow tab:
  Leader:    (subform to show at start of overflow — e.g. continued header)
  Trailer:   (subform to show at end of page — e.g. "continued on next page")
  Overflow:  Target body page for overflow content
```
