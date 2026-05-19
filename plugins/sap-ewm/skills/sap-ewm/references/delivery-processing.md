# EWM Delivery Processing Reference

## Delivery Model

EWM uses the `/SCDL/` (Supply Chain Delivery Layer) API for all delivery operations.
Deliveries are process-oriented documents separate from MM/SD delivery objects.

```
Inbound:  Supplier → ASN (DESADV IDoc) → EWM Inbound Delivery → GR → Stock
Outbound: Sales Order → EWM Outbound Delivery → Pick → GI → Goods Issue
```

## Key Tables

| Table | Content |
|---|---|
| `/SCDL/DB_PROCH_O` | Delivery header (all types) |
| `/SCDL/DB_PROCI_O` | Inbound delivery items |
| `/SCDL/DB_PROCO_O` | Outbound delivery items |
| `/SCDL/DB_PROCD_O` | Delivery schedule lines |
| `/SCDL/DB_PROCREF_O` | Delivery document references (links to WO/WT) |

## Document Categories

| Constant | Value | Meaning |
|---|---|---|
| `/SCWM/IF_DL_C=>SC_RDOCCAT_ID` | `IDL` | Inbound delivery |
| `/SCWM/IF_DL_C=>SC_RDOCCAT_OD` | `ODL` | Outbound delivery |
| `/SCWM/IF_DL_C=>SC_RDOCCAT_WT` | `WHT` | Warehouse task |
| `/SCWM/IF_DL_C=>SC_RDOCCAT_WO` | `WHO` | Warehouse order |

## Inbound Delivery Processing

### Read Inbound Delivery

```abap
DATA: ls_header TYPE /scdl/ds_dochead_con,
      lt_items  TYPE /scdl/tt_docitem_con.

CALL METHOD /scdl/cl_dm_delivery=>get_delivery_by_docno
  EXPORTING
    iv_docno   = lv_delivery_number
    iv_rdoccat = /scwm/if_dl_c=>sc_rdoccat_id
  IMPORTING
    es_header  = ls_header
    et_items   = lt_items.
```

### Search Deliveries

```abap
DATA: lt_sel   TYPE /scdl/tt_db_proch_o_sel,
      lt_heads TYPE /scdl/tt_db_proch_o.

" Search by vendor and date
APPEND VALUE /scdl/s_db_proch_o_sel(
  sign   = 'I'
  option = 'EQ'
  low    = lv_vendor ) TO lt_sel.

CALL METHOD /scdl/cl_dm_delivery=>get_list
  EXPORTING
    it_sel_hdr = lt_sel
    iv_rdoccat = /scwm/if_dl_c=>sc_rdoccat_id
  IMPORTING
    et_headers = lt_heads.
```

### Post Goods Receipt

```abap
CALL FUNCTION '/SCWM/GR_POST'
  EXPORTING
    iv_lgnum    = lv_lgnum
    iv_docno    = lv_inbound_delivery
    iv_commit   = abap_true
  IMPORTING
    et_bapiret  = DATA(lt_messages).

" Check for errors
LOOP AT lt_messages ASSIGNING FIELD-SYMBOL(<msg>) WHERE type = 'E' OR type = 'A'.
  " Handle error
ENDLOOP.
```

### Unplanned GR (without prior delivery)

```abap
CALL FUNCTION '/SCWM/ADGI'
  EXPORTING
    iv_lgnum    = lv_lgnum
    iv_matnr    = lv_material
    iv_werks    = lv_plant
    iv_quan     = lv_quantity
    iv_meins    = lv_uom
    iv_lgpla    = lv_bin
  IMPORTING
    et_bapiret  = DATA(lt_messages).
```

## Outbound Delivery Processing

### Read Outbound Delivery

```abap
CALL METHOD /scdl/cl_dm_delivery=>get_delivery_by_docno
  EXPORTING
    iv_docno   = lv_delivery_number
    iv_rdoccat = /scwm/if_dl_c=>sc_rdoccat_od
  IMPORTING
    es_header  = DATA(ls_header)
    et_items   = DATA(lt_items).
```

### Post Goods Issue

```abap
CALL FUNCTION '/SCWM/GI_POST'
  EXPORTING
    iv_lgnum    = lv_lgnum
    iv_docno    = lv_outbound_delivery
    iv_commit   = abap_true
  IMPORTING
    et_bapiret  = DATA(lt_messages).
```

### Check Pick Completeness

```abap
" All warehouse tasks confirmed = delivery ready for GI
SELECT COUNT(*) FROM /scwm/ordim_o
  WHERE lgnum  = @lv_lgnum
    AND rdocno = @lv_outbound_delivery
    AND tostat <> @( /scwm/if_ordim_c=>sc_tostat_conf )
  INTO @DATA(lv_open_count).

IF lv_open_count = 0.
  " All tasks confirmed — safe to GI
ENDIF.
```

## Delivery Status Fields

| Field | Values | Meaning |
|---|---|---|
| `WBSTA` | blank / A / B / C | Warehouse processing status: not started / partial / complete |
| `GRSTA` | blank / A | Goods receipt status |
| `GISTA` | blank / A | Goods issue status |

## Packing (Outbound)

```abap
" Create HU for packing
CALL FUNCTION '/SCWM/HU_CREATE'
  EXPORTING
    iv_lgnum     = lv_lgnum
    iv_hutyp     = lv_hu_type     " Pallet, carton, etc.
  IMPORTING
    ev_huident   = DATA(lv_hu_number)
    et_bapiret   = DATA(lt_messages).

" Pack delivery items into HU
CALL FUNCTION '/SCWM/HU_PACK'
  EXPORTING
    iv_lgnum     = lv_lgnum
    iv_huident   = lv_hu_number
    iv_docno     = lv_outbound_delivery
    iv_itemno    = lv_item
    iv_quan      = lv_pack_quantity
  IMPORTING
    et_bapiret   = DATA(lt_messages).
```

## Delivery BADI Hooks

| BADI | When It Fires | Common Use |
|---|---|---|
| `/SCWM/IF_EX_GR_GI` | Before/after GR or GI posting | Pre-GR validation, post-GI notification |
| `/SCWM/IF_EX_DLV_MANAGEMENT` | Delivery create/change/delete | Custom field defaulting, cross-checks |
| `/SCWM/IF_EX_DELIVERY_PPF` | Post-processing (PPF actions) | Trigger output, label printing, EDI |

## Label and Output (PPF)

EWM uses the Post Processing Framework (PPF) for delivery output — shipping labels, ASN outputs, etc.

```abap
" Trigger a PPF action on a delivery (e.g. print label)
CALL FUNCTION 'CL_APF_EXEC_SIMPLE=>EXEC_IMMEDIATELY'
  EXPORTING
    iv_applic  = /scwm/if_dl_c=>sc_rdoccat_od
    iv_docno   = lv_delivery_number
    iv_actn    = 'ZSHIP_LABEL'   " Your PPF action
  IMPORTING
    et_return  = DATA(lt_return).
```

For SmartForms and Adobe Forms triggered via PPF, the PPF action calls a processing class
that instantiates the form — see `sap-adobe-forms` and `sap-smartforms` skills for form
development patterns.
