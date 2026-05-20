# Common SmartForms Patterns Reference

## EWM Delivery Note

### Interface

```abap
TYPES:
  BEGIN OF zewm_dn_hdr,
    delivery_no   TYPE vbeln_vl,
    delivery_date TYPE wadat_ist,
    ship_to_name  TYPE name1,
    street        TYPE stras_gp,
    city          TYPE ort01,
    postal_code   TYPE pstlz,
    country       TYPE land1,
    carrier_name  TYPE name1,
    tracking_no   TYPE char30,
    total_weight  TYPE char15,    " "1,234.56 KG" — pre-formatted
    total_vol     TYPE char15,    " "12.34 M3"
  END OF zewm_dn_hdr,

  BEGIN OF zewm_dn_item,
    item_no       TYPE posnr,
    matnr         TYPE matnr,
    matnr_desc    TYPE maktx,
    qty_text      TYPE char15,   " "100 PC" — pre-formatted
    batch         TYPE charg_d,
    serial_no     TYPE char30,
    hu_id         TYPE venum,    " For barcode
  END OF zewm_dn_item,
  zewm_dn_items TYPE TABLE OF zewm_dn_item.
```

### Form Structure

```
Pages: FIRST → NEXT (loop)
  HEADER window (secondary, 30mm):
    Template: "DELIVERY NOTE" | Logo | Delivery No: &IS_HDR-DELIVERY_NO&
  ADDR window (secondary, 35mm):
    Address node → IS_HDR ship-to fields
  MAIN window (main, 200mm):
    Table: IT_ITEMS
      Header row: Item | Material | Description | Qty | Batch | HU ID
      Data row:   &ITEM_NO& | &MATNR& | &MATNR_DESC& | &QTY_TEXT& | &BATCH& | barcode &HU_ID&
      Footer row: "Total items: &SFSY-TABLEROWS&  Weight: &GV_TOTAL_WEIGHT&"
  FOOTER window (secondary, 15mm):
    Text: "Page &SFSY-PAGE& of &SFSY-FORMPAGES& — &IS_HDR-CARRIER_NAME& Tracking: &IS_HDR-TRACKING_NO&"
```

## EWM Warehouse Label

### Interface

```abap
TYPES:
  BEGIN OF zewm_wh_label,
    hu_id         TYPE venum,       " Handling unit (barcode)
    delivery_no   TYPE vbeln_vl,
    matnr         TYPE matnr,
    matnr_desc    TYPE maktx,
    batch         TYPE charg_d,
    quantity      TYPE char15,
    dest_bin      TYPE lgpla,
    dest_bin_bc   TYPE char20,      " LGNUM+LGTYP+LGPLA for barcode
    storage_type  TYPE lgtyp,
    weight        TYPE char12,
  END OF zewm_wh_label.
```

### Form Structure (100mm × 150mm label)

```
Page Format:  ZLABEL_100X150 (100×150mm)
Pages: LABEL (no next page — one label per form call)
  MAIN window:
    Template:
      Row 1: "SHIP TO:"  | &IS_LBL-DEST_BIN&  (large font)
      Row 2: Barcode (Code128): &IS_LBL-HU_ID&
      Row 3: &IS_LBL-MATNR_DESC&
      Row 4: Qty: &IS_LBL-QUANTITY& | Batch: &IS_LBL-BATCH&
      Row 5: Delivery: &IS_LBL-DELIVERY_NO&
      Row 6: Bin barcode (Code128): &IS_LBL-DEST_BIN_BC&
```

### Print Loop (one label per HU)

```abap
LOOP AT lt_labels ASSIGNING FIELD-SYMBOL(<lbl>).
  CALL FUNCTION 'SSF_FUNCTION_MODULE_NAME'
    EXPORTING  formname = 'ZEWM_WH_LABEL'
    IMPORTING  fm_name  = lv_fm.

  CALL FUNCTION lv_fm
    EXPORTING
      control_parameters = ls_ctrl
      output_options     = ls_comp
      is_label           = <lbl>.
ENDLOOP.
```

## EWM Transfer Order Slip (TO Confirmation)

### Interface

```abap
TYPES:
  BEGIN OF zewm_to_hdr,
    tanum         TYPE /scwm/tanum,
    lgnum         TYPE /scwm/lgnum,
    who           TYPE /scwm/who,
    rsrc          TYPE /scwm/rsrc,    " Resource (forklift/picker)
    created_on    TYPE datum,
    priority      TYPE /scwm/priority,
  END OF zewm_to_hdr,

  BEGIN OF zewm_to_item,
    tapos         TYPE /scwm/tapos,
    matnr         TYPE matnr,
    matnr_desc    TYPE maktx,
    quantity      TYPE char15,
    src_bin       TYPE lgpla,
    dest_bin      TYPE lgpla,
    src_bin_bc    TYPE char20,
    dest_bin_bc   TYPE char20,
    hu_id         TYPE venum,
  END OF zewm_to_item,
  zewm_to_items TYPE TABLE OF zewm_to_item.
```

## SD Invoice

### Interface

```abap
TYPES:
  BEGIN OF zsd_invoice_hdr,
    billing_doc   TYPE vbeln,
    billing_date  TYPE fkdat,
    sold_to_name  TYPE name1,
    street        TYPE stras_gp,
    city          TYPE ort01,
    vat_no        TYPE char20,
    payment_terms TYPE dzterm,
    due_date      TYPE datum,
    currency      TYPE waers,
    net_value     TYPE char20,     " Pre-formatted
    tax_amount    TYPE char20,
    gross_value   TYPE char20,
  END OF zsd_invoice_hdr,

  BEGIN OF zsd_invoice_item,
    item_no       TYPE posnr,
    matnr_desc    TYPE maktx,
    quantity      TYPE char15,
    uom           TYPE meins,
    unit_price    TYPE char15,
    net_value     TYPE char20,
    tax_rate      TYPE char8,
  END OF zsd_invoice_item.
```

## Dangerous Goods Declaration

### Interface

```abap
TYPES:
  BEGIN OF zdg_hdr,
    transport_doc TYPE char20,
    shipper       TYPE name1,
    consignee     TYPE name1,
    transport_mode TYPE char1,    " A=Air R=Road S=Sea
    un_class      TYPE char3,
  END OF zdg_hdr,

  BEGIN OF zdg_item,
    un_number     TYPE char6,     " "UN1234"
    proper_name   TYPE char100,
    haz_class     TYPE char4,     " "3" / "6.1" etc
    pack_group    TYPE char3,     " "I" "II" "III"
    quantity      TYPE char20,    " "100 L"
    packing_type  TYPE char30,
    flash_point   TYPE char10,
  END OF zdg_item,
  zdg_items TYPE TABLE OF zdg_item.
```

## Initialization Code Patterns (Global Definitions)

### Date Formatting

```abap
" Format date to local format
WRITE IS_HEADER-SHIP_DATE TO GS_DISPLAY-SHIP_DATE_TEXT DD/MM/YYYY.
```

### Quantity + UOM Pre-formatting

```abap
LOOP AT IT_ITEMS ASSIGNING FIELD-SYMBOL(<item>).
  DATA lv_qty_char TYPE char15.
  WRITE <item>-QUANTITY TO lv_qty_char LEFT-JUSTIFIED.
  CONDENSE lv_qty_char.
  <item>-QTY_TEXT = lv_qty_char && ' ' && <item>-UOM.
ENDLOOP.
```

### Currency Amount Formatting

```abap
DATA lv_amount_char TYPE char20.
WRITE IS_HEADER-NET_VALUE CURRENCY IS_HEADER-CURRENCY TO lv_amount_char.
CONDENSE lv_amount_char.
GS_DISPLAY-NET_VALUE = lv_amount_char && ' ' && IS_HEADER-CURRENCY.
```

### Running Total in Table Footer

```abap
" In Table → Events → Table Footer:
GV_TOTAL_QTY = 0.
LOOP AT IT_ITEMS ASSIGNING FIELD-SYMBOL(<i>).
  GV_TOTAL_QTY = GV_TOTAL_QTY + <i>-QUANTITY.
ENDLOOP.
WRITE GV_TOTAL_QTY TO GV_TOTAL_QTY_TEXT LEFT-JUSTIFIED.
CONDENSE GV_TOTAL_QTY_TEXT.
```
