# EWM Warehouse Structure Reference

## Organizational Hierarchy

```
Client (MANDT)
  └── Warehouse Number (LGNUM)         — top-level EWM organizational unit
        └── Storage Type (LGTYP)       — physical area type (rack, floor, HU, interim)
              └── Storage Section (LGBER) — subdivision within a type (fast-moving, slow)
                    └── Storage Bin (LGPLA) — physical location (aisle-row-column)
                          └── Quant      — stock record per product/batch/stock type
```

## Warehouse Number (LGNUM)

The warehouse number is the root of all EWM activity. Most `/SCWM/` APIs require it as a mandatory import.

- Configuration: `/SCWM/LGNUM` transaction or `/SCWM/V_LGNUM` view
- In embedded EWM (S/4HANA): determined from Plant + Storage Location via `/SCWM/LGNUM_DETERMINE`
- In decentralized EWM: matches the logical system mapping

```abap
" Determine warehouse number from MM org data
CALL FUNCTION '/SCWM/LGNUM_DETERMINE'
  EXPORTING
    iv_werks  = '1000'      " Plant
    iv_lgort  = '0001'      " MM storage location
  IMPORTING
    ev_lgnum  = DATA(lv_lgnum)
  EXCEPTIONS
    not_found = 1.
IF sy-subrc <> 0.
  " No EWM warehouse assigned to this plant/sloc
ENDIF.
```

## Storage Types (LGTYP)

Common standard storage type categories:

| Category | Typical LGTYP Range | Examples |
|---|---|---|
| Fixed bin storage | 0001–0099 | High-rack, shelving |
| Goods receipt zone | 0100–0109 | GR staging area |
| Goods issue zone | 0200–0209 | GI staging / dock |
| Packing station | 0300–0309 | Pack/ship workstation |
| HU interim | 9000–9999 | Interim storage for HUs |

Configuration: `/SCWM/LGTYP` or Customizing → EWM → Master Data → Define Storage Types.

## Quants (/SCWM/QUAN)

A quant is the smallest unit of stock management — one record per unique combination of bin + product + batch + stock type + owner.

### Key Quant Fields

| Field | Description |
|---|---|
| `LGNUM` | Warehouse number |
| `LGPLA` | Storage bin |
| `LGTYP` | Storage type |
| `MATNR` | Material/product number |
| `CHARG` | Batch number |
| `BESTQ` | Stock type (F=unrestricted, S=blocked, Q=quality) |
| `QUAN` | Current quantity |
| `MEINS` | Unit of measure |
| `OWNER` | Stock owner (for multi-client warehouses) |

### Query Quants

```abap
" All stock for a product in a warehouse
SELECT lgpla, lgtyp, charg, bestq, quan, meins
  FROM /scwm/quan
  WHERE lgnum = @lv_lgnum
    AND matnr = @lv_material
  INTO TABLE @DATA(lt_quants).

" Stock at a specific bin
SELECT *
  FROM /scwm/quan
  WHERE lgnum = @lv_lgnum
    AND lgpla = @lv_bin
  INTO TABLE @DATA(lt_bin_stock).

" Available (unrestricted) stock
SELECT SUM( quan ) AS total_qty, meins
  FROM /scwm/quan
  WHERE lgnum = @lv_lgnum
    AND matnr = @lv_material
    AND bestq = 'F'
  GROUP BY meins
  INTO TABLE @DATA(lt_available).
```

## Storage Bin Master (/SCWM/T302)

```abap
" Read bin details
SELECT SINGLE *
  FROM /scwm/t302
  WHERE lgnum = @lv_lgnum
    AND lgpla = @lv_bin
  INTO @DATA(ls_bin).

" Check if bin is locked
IF ls_bin-kzspb = abap_true.
  " Bin is locked for putaway/removal
ENDIF.
```

### Bin Lock Types

| Field | Meaning |
|---|---|
| `KZSPB` | General lock |
| `KZSPE` | Lock for putaway |
| `KZSPA` | Lock for removal |
| `KZRET` | Return lock |

## Handling Units (HU)

EWM uses Handling Units (pallets, cartons, etc.) for physical tracking.

```abap
" Read HU stock
CALL FUNCTION '/SCWM/HU_READ'
  EXPORTING
    iv_lgnum   = lv_lgnum
    iv_huident = lv_hu_number
  IMPORTING
    es_hu      = DATA(ls_hu)
    et_items   = DATA(lt_hu_items).

" Get all HUs in a bin
SELECT huident
  FROM /scwm/hu_st_vis           " HU stock visibility view
  WHERE lgnum = @lv_lgnum
    AND lgpla = @lv_bin
  INTO TABLE @DATA(lt_hus).
```

## Posting Changes (Stock Type Changes)

```abap
" Change stock type (e.g. Q→F: quality release)
CALL FUNCTION '/SCWM/POSTING_CHANGE'
  EXPORTING
    iv_lgnum    = lv_lgnum
    iv_matnr    = lv_material
    iv_bestq    = 'Q'            " From: quality
    iv_bestq_d  = 'F'            " To: unrestricted
    iv_quan     = lv_quantity
    iv_lgpla    = lv_bin
  IMPORTING
    et_bapiret  = DATA(lt_messages).
```
