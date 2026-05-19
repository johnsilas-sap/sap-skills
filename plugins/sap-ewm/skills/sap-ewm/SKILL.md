---
name: sap-ewm
description: |
  SAP Extended Warehouse Management (EWM) development skill. Use when working with
  warehouse structures, transfer orders (TO/WT), warehouse tasks, RF mobile transactions
  (ITSmobile), resource management, wave management, queue management, physical inventory,
  delivery processing, posting changes, BADIs and enhancement spots for EWM, ABAP APIs
  (/SCWM/*, /SCDL/*), performance monitoring (ST13 PERF_TOOL), or EWM IDoc integration.
  Covers decentralized and embedded EWM on SAP S/4HANA and SAP SCM.
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-19"
  ewm_release: "SAP EWM 9.4+ / S/4HANA EWM 2020+"
  sources:
    - "https://help.sap.com/docs/SAP_EXTENDED_WAREHOUSE_MANAGEMENT"
    - "https://help.sap.com/docs/S4HANA_ON-PREMISE"
---

# SAP EWM Development Skill

## Related Skills

- **sap-abap**: Use for general ABAP syntax, internal tables, OO patterns, and ABAP SQL
- **sap-abap-cds**: Use for CDS views backing EWM Fiori apps or analytical reporting
- **sap-btp-connectivity**: Use when exposing EWM services to BTP or external systems via APIs
- **sap-idoc**: Use for EWM IDoc message types (WMMBXY, SHPMNT, DESADV) and partner profiles
- **sap-api-style**: Use when documenting EWM OData or REST APIs

## Warehouse Structure

EWM organizes physical storage in a strict hierarchy:

```
Warehouse Number (LGNUM)
  └── Storage Type (LGTYP)       e.g. 0001 = High-rack, 0100 = Goods Receipt Zone
        └── Storage Section (LGBER)
              └── Storage Bin (LGPLA)
                    └── Quant (/SCWM/QUAN)  — actual stock quantity per product/batch
```

### Key Tables

| Table | Description |
|---|---|
| `/SCWM/QUAN` | Quants — stock at bin level (product, batch, qty, stock type) |
| `/SCWM/WHO` | Warehouse orders — groups of warehouse tasks assigned to a resource |
| `/SCWM/WHT` | Warehouse tasks — individual movement instructions |
| `/SCWM/RSRC` | Resources — RF devices and their logged-in users |
| `/SCWM/T300` | Storage type configuration |
| `/SCWM/T301` | Storage section configuration |
| `/SCWM/T302` | Storage bin master |
| `/SCDL/DB_PROCI_O` | Inbound delivery items |
| `/SCDL/DB_PROCO_O` | Outbound delivery items |
| `/SCWM/ORDIM_C` | Confirmed warehouse tasks (history) |
| `/SCWM/ORDIM_O` | Open warehouse tasks |

## Transfer Orders and Warehouse Tasks

EWM uses **Warehouse Tasks (WT)** grouped into **Warehouse Orders (WO)**.

### Create Warehouse Task via ABAP API

```abap
DATA: lt_request  TYPE /scwm/tt_to_request_hdr,
      lt_created  TYPE /scwm/tt_ordim_o,
      lt_messages TYPE /scwm/tt_bapiret2.

APPEND VALUE /scwm/s_to_request_hdr(
  rdoccat  = /scwm/if_dl_c=>sc_rdoccat_wt  " Warehouse task
  lgnum    = lv_lgnum
  bseg_src = VALUE #(
    lgpla  = lv_source_bin
    lgtyp  = lv_source_type
    matnr  = lv_material
    quan   = lv_quantity
    meins  = lv_uom )
  bseg_dst = VALUE #(
    lgpla  = lv_dest_bin
    lgtyp  = lv_dest_type )
) TO lt_request.

CALL FUNCTION '/SCWM/TO_CREATE_TD'
  EXPORTING
    iv_lgnum     = lv_lgnum
    iv_commit    = abap_true
  CHANGING
    ct_request   = lt_request
  IMPORTING
    et_ordim_o   = lt_created
    et_bapiret   = lt_messages.
```

### Confirm Warehouse Task

```abap
CALL FUNCTION '/SCWM/TO_CONFIRM_TD'
  EXPORTING
    iv_lgnum     = lv_lgnum
    iv_tanum     = lv_task_number
    iv_commit    = abap_true
  IMPORTING
    et_bapiret   = lt_messages.
```

### Query Open Warehouse Tasks

```abap
SELECT * FROM /scwm/ordim_o
  WHERE lgnum = @lv_lgnum
    AND tostat = @( /scwm/if_ordim_c=>sc_tostat_open )
    AND rsrc   = @lv_resource
  INTO TABLE @DATA(lt_open_tasks).
```

## RF Mobile (ITSmobile)

EWM RF transactions run through SAP ITSmobile (ITS = Internet Transaction Server). The UI is rendered as HTML on the device browser and driven by **RF Logical Transaction** configurations.

### RF Architecture

```
Scanner (Velocity/Ivanti browser)
  → HTTP GET/POST to SAP ITS (/sap/bc/gui/sap/its/webgui)
  → ITSmobile renders HTML for the RF screen
  → /SCWM/CL_RF_BLL_PROCESSOR handles business logic
  → Data stored in EWM tables (/SCWM/RSRC, /SCWM/WHO, /SCWM/WHT)
```

### Key RF Transactions

| Transaction | Description |
|---|---|
| `/SCWM/RFUI` | RF logical transaction configuration |
| `/SCWM/RF01` | RF goods receipt |
| `/SCWM/RF02` | RF putaway |
| `/SCWM/RF03` | RF pick |
| `/SCWM/RF04` | RF goods issue |
| `/SCWM/RF05` | RF stock transfer |
| `/SCWM/RF10` | RF physical inventory |
| `/SCWM/RFLOGIN` | RF user login / resource assignment |

### Resource Assignment (Login)

When an RF user logs in, EWM links the SAP user to a physical device (resource) in `/SCWM/RSRC`:

```abap
" Look up which user is logged into a resource/device
SELECT SINGLE uname
  FROM /scwm/rsrc
  WHERE lgnum = @lv_lgnum
    AND rsrc   = @lv_resource_name
  INTO @DATA(lv_rf_user).
```

### RF BADI — Enhancing RF Screen Logic

```abap
" BADI: /SCWM/EX_RF_BLL_PROCESSOR
" Enhancement spot: /SCWM/ES_RF_BLL_PROCESSOR

METHOD /scwm/if_ex_rf_bll_processor~process_request.
  " Called for every RF screen transition
  " is_req: current request data (scanned values, function key)
  " cs_resp: modify response before it renders on screen
  CASE is_req-linfld-tcode.
    WHEN '/SCWM/RF03'.  " Pick transaction
      " Custom logic: validate product before confirming pick
  ENDCASE.
ENDMETHOD.
```

## Key BADIs and Enhancement Spots

| BADI Interface | Enhancement Spot | When to Use |
|---|---|---|
| `/SCWM/IF_EX_RF_BLL_PROCESSOR` | `/SCWM/ES_RF_BLL_PROCESSOR` | Modify RF screen logic, add validations |
| `/SCWM/IF_EX_TO_CREATE` | `/SCWM/ES_CORE_DETERMINATION` | Influence WT/WO creation (bin determination, qty split) |
| `/SCWM/IF_EX_WHO_PACKING` | `/SCWM/ES_WHO_PACKING` | Packing logic during WO processing |
| `/SCWM/IF_EX_QUAN_CHANGE` | `/SCWM/ES_QUAN_CHANGE` | React to stock quantity changes |
| `/SCWM/IF_EX_CORE_DETERMINATION` | `/SCWM/ES_CORE_DETERMINATION` | Override putaway/pick bin determination |
| `/SCWM/IF_EX_DELIVERY_PPF` | `/SCWM/ES_DELIVERY_PPF` | Post-processing of deliveries |
| `/SCWM/IF_EX_PHYINV` | `/SCWM/ES_PHYINV` | Physical inventory count enhancements |

### BADI Implementation Pattern

```abap
" 1. In SE18: find enhancement spot /SCWM/ES_CORE_DETERMINATION
" 2. Create BADI implementation for /SCWM/EX_CORE_DETERMINATION

CLASS zcl_ewm_bin_override DEFINITION PUBLIC.
  PUBLIC SECTION.
    INTERFACES /scwm/if_ex_core_determination.
ENDCLASS.

CLASS zcl_ewm_bin_override IMPLEMENTATION.
  METHOD /scwm/if_ex_core_determination~determine_putaway_bin.
    " Override destination bin for a specific storage type
    IF is_ltap_vb-lgtyp = '0001' AND is_ltap_vb-matnr = 'SPECIAL'.
      cs_ltap_vb-lgpla = 'SPECIAL-BIN-001'.
    ENDIF.
  ENDMETHOD.
ENDCLASS.
```

## Performance Monitoring (ST13 PERF_TOOL)

EWM has a built-in RF performance analysis tool. See also: `EWM_RF_Perf` project for frontend timing capture.

### Viewing Performance Data

1. Transaction `ST13` → Performance Tools → `PERF_TOOL`
2. Select `EWM_RF_Analysis`
3. Enter warehouse number, RF user, date range
4. **Transactions** tab: shows backend times (`Bck. ms`)
5. **Load GUI** button: merges frontend data (`GUI ms`) if the JS monitor is deployed

### Performance Tables

| Table | Content |
|---|---|
| `/SCMB/PFM_RFUI` | RF UI performance records (delivered by SAP Note 1690850) |
| `/SCMB/PFM2_GUI_STORE` | Program delivering `ewmrf_store_gui` routine (SAP Note 1595305) |

### Key SAP Notes for RF Performance

| Note | Purpose |
|---|---|
| 1690850 | RFUI performance measurement base — activates backend timing |
| 1595305 | Frontend data upload interface — delivers `ewmrf_store_gui` |

## Delivery Processing

### Inbound Delivery API

```abap
" Read inbound delivery
DATA: ls_dlv_header TYPE /scdl/ds_dochead_con,
      lt_dlv_items  TYPE /scdl/tt_docitem_con.

CALL METHOD /scdl/cl_dm_delivery=>get_delivery_by_docno
  EXPORTING
    iv_docno   = lv_delivery_number
    iv_rdoccat = /scwm/if_dl_c=>sc_rdoccat_id  " Inbound
  IMPORTING
    es_header  = ls_dlv_header
    et_items   = lt_dlv_items.
```

### Goods Receipt Posting

```abap
CALL FUNCTION '/SCWM/GR_POST'
  EXPORTING
    iv_lgnum    = lv_lgnum
    iv_docno    = lv_inbound_delivery
  IMPORTING
    et_bapiret  = lt_messages.
```

### Goods Issue Posting

```abap
CALL FUNCTION '/SCWM/GI_POST'
  EXPORTING
    iv_lgnum    = lv_lgnum
    iv_docno    = lv_outbound_delivery
  IMPORTING
    et_bapiret  = lt_messages.
```

## Wave Management

Waves group outbound deliveries for coordinated picking.

```abap
" Create wave
CALL FUNCTION '/SCWM/WAVE_CREATE'
  EXPORTING
    iv_lgnum      = lv_lgnum
    it_docno      = lt_delivery_numbers
  IMPORTING
    ev_wave_id    = lv_wave_id
    et_bapiret    = lt_messages.

" Release wave (triggers WT creation)
CALL FUNCTION '/SCWM/WAVE_RELEASE'
  EXPORTING
    iv_lgnum      = lv_lgnum
    iv_wave_id    = lv_wave_id
  IMPORTING
    et_bapiret    = lt_messages.
```

## Physical Inventory

```abap
" Create physical inventory document
CALL FUNCTION '/SCWM/PI_DOC_CREATE'
  EXPORTING
    iv_lgnum    = lv_lgnum
    it_lgpla    = lt_bins_to_count
  IMPORTING
    et_pi_docs  = lt_pi_documents
    et_bapiret  = lt_messages.

" Post count result
CALL FUNCTION '/SCWM/PI_COUNT_POST'
  EXPORTING
    iv_lgnum    = lv_lgnum
    iv_pi_docno = lv_pi_document
    it_count    = lt_count_results
  IMPORTING
    et_bapiret  = lt_messages.
```

## Queue Management

Queues route warehouse orders to specific resources or resource groups.

```abap
" Assign warehouse order to queue
CALL FUNCTION '/SCWM/WHO_QUEUE_ASSIGN'
  EXPORTING
    iv_lgnum    = lv_lgnum
    iv_who      = lv_warehouse_order
    iv_queue    = lv_queue_name
  IMPORTING
    et_bapiret  = lt_messages.

" Read queue assignments
SELECT * FROM /scwm/tqact
  WHERE lgnum = @lv_lgnum
    AND queue = @lv_queue
  INTO TABLE @DATA(lt_queue_assignments).
```

## Common Patterns

### Error Handling with BAPIRET2

```abap
DATA lt_messages TYPE /scwm/tt_bapiret2.

" After any /SCWM/ function module call:
LOOP AT lt_messages ASSIGNING FIELD-SYMBOL(<msg>)
  WHERE type = 'E' OR type = 'A'.
  " Log or raise exception
  MESSAGE ID <msg>-id TYPE <msg>-type NUMBER <msg>-number
    WITH <msg>-message_v1 <msg>-message_v2
         <msg>-message_v3 <msg>-message_v4.
ENDLOOP.
```

### Warehouse Number from Plant/Storage Location

```abap
" Determine EWM warehouse number from MM storage location
CALL FUNCTION '/SCWM/LGNUM_DETERMINE'
  EXPORTING
    iv_werks   = lv_plant
    iv_lgort   = lv_storage_location
  IMPORTING
    ev_lgnum   = lv_lgnum
  EXCEPTIONS
    not_found  = 1.
```

## Key Transactions

| Transaction | Purpose |
|---|---|
| `/SCWM/MON` | EWM Warehouse Monitor — main operational cockpit |
| `/SCWM/LGNUM` | Warehouse number configuration |
| `/SCWM/PRDI` | Inbound delivery processing |
| `/SCWM/PRDO` | Outbound delivery processing |
| `/SCWM/TO01` | Create transfer order manually |
| `/SCWM/TO09` | Confirm transfer order |
| `/SCWM/RFUI` | RF logical transaction configuration |
| `/SCWM/RSRC` | Resource (scanner) management |
| `ST13` | Performance analysis (PERF_TOOL → EWM_RF_Analysis) |
| `SICF` | ICF service management (for custom HTTP handlers) |
| `SLG1` | Application log viewer |
