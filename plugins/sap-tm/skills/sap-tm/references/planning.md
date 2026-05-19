# TM Transportation Planning Reference

## Planning Overview

Transportation planning in TM converts demand (freight units) into supply (freight orders):

```
Sales Order / Purchase Order (ERP)
  → Freight Unit (FU) — created automatically or manually
  → Planning Run — assigns FUs to freight orders
  → Freight Order (FO) — with carrier, route, stages
  → Execution — confirmation, goods issue
```

## Freight Unit Creation

### From Sales Order (ERP → TM)

```abap
" Triggered automatically via ALE/IDoc or called programmatically
CALL FUNCTION '/SCMTMS/FU_CREATE_FROM_SO'
  EXPORTING
    iv_vbeln     = lv_sales_order
    iv_posnr     = lv_item
    iv_commit    = abap_true
  IMPORTING
    ev_fu_id     = DATA(lv_fu_id)
    et_return    = DATA(lt_messages).
```

### Manual Freight Unit Creation

```abap
DATA: ls_fu TYPE /scmtms/s_tor_root.

ls_fu = VALUE /scmtms/s_tor_root(
  torcat      = /scmtms/if_tor_const=>gc_torcat_fu
  src_loc_id  = 'LOC_PLANT_1000'
  dst_loc_id  = 'LOC_CUSTOMER_500'
  req_date    = lv_required_delivery
  gross_weight = lv_weight
  volume       = lv_volume
  uom_weight   = 'KG'
  uom_volume   = 'M3' ).

CALL METHOD /scmtms/cl_tor_bo_factory=>create_tor
  EXPORTING
    is_root   = ls_fu
  IMPORTING
    ev_tor_id = DATA(lv_fu_id)
    et_return = DATA(lt_messages).
COMMIT WORK.
```

## Planning Execution

### Automatic Planning (API)

```abap
DATA: lt_fu_ids TYPE /scmtms/t_tor_id_tab.
APPEND lv_fu_id TO lt_fu_ids.

CALL FUNCTION '/SCMTMS/PLANNING_EXECUTE'
  EXPORTING
    it_fu_ids        = lt_fu_ids
    iv_plan_profile  = 'Z_ROAD_PLANNING'  " Customizing profile
    iv_mode          = 'AUTO'             " AUTO / INTERACTIVE
  IMPORTING
    et_fro_ids       = DATA(lt_created_fos)
    et_unplanned_fus = DATA(lt_unplanned)  " FUs that couldn't be planned
    et_return        = DATA(lt_messages).
```

### Planning Profiles

Planning profiles (Customizing → TM → Planning) control:
- Transportation mode selection (road/sea/air)
- Consolidation rules (consolidate by region, product, date)
- Carrier pre-selection
- Route determination

## Transportation Lane Determination

Lanes define valid routes between locations:

```abap
" Find valid lane from source to destination
SELECT lane_id, src_loc_id, dst_loc_id, means_tra,
       transit_days, carrier_id
  FROM /scmtms/d_lane
  WHERE src_loc_id = @lv_source
    AND dst_loc_id = @lv_destination
    AND means_tra  = @lv_mode
    AND valid_from <= @sy-datum
    AND valid_to   >= @sy-datum
  INTO TABLE @DATA(lt_lanes).
```

## Consolidation

TM consolidates multiple freight units into a single freight order when:
- Same source/destination
- Compatible DG classes
- Weight/volume within vehicle capacity
- Within the consolidation time window

### Check Consolidation Candidates

```abap
" Find FUs that can consolidate with a given FU
CALL FUNCTION '/SCMTMS/CONSOLIDATION_CANDIDATES'
  EXPORTING
    iv_fu_id         = lv_freight_unit_id
    iv_plan_profile  = lv_profile
  IMPORTING
    et_candidates    = DATA(lt_candidates)
    et_return        = DATA(lt_messages).
```

## BADI: Planning Enhancement

### Enhancement Spot: `/SCMTMS/ES_PLANNING`
### BADI Interface: `/SCMTMS/IF_EX_PLANNING`

```abap
CLASS zcl_tm_planning_enh DEFINITION PUBLIC.
  PUBLIC SECTION.
    INTERFACES /scmtms/if_ex_planning.
ENDCLASS.

CLASS zcl_tm_planning_enh IMPLEMENTATION.

  METHOD /scmtms/if_ex_planning~before_planning.
    " Runs before planning algorithm — filter or sort FUs
    " Example: prioritize express FUs
    SORT ct_fu_ids BY " custom sort on FU priority
      STABLE.
  ENDMETHOD.

  METHOD /scmtms/if_ex_planning~after_planning.
    " Runs after FOs are created — apply post-planning rules
    " Example: trigger capacity check on new FOs
    LOOP AT it_created_fros ASSIGNING FIELD-SYMBOL(<fo>).
      CALL FUNCTION 'Z_CHECK_VEHICLE_CAPACITY'
        EXPORTING iv_fo_id = <fo>-tor_id.
    ENDLOOP.
  ENDMETHOD.

  METHOD /scmtms/if_ex_planning~override_route.
    " Override the determined route for specific lanes
    IF is_fu-src_loc_id = 'LOC_HAZMAT_PLANT'.
      " Force specific safe route for hazmat origin
      cv_route_id = 'Z_HAZMAT_ROUTE_001'.
    ENDIF.
  ENDMETHOD.

ENDCLASS.
```

## BADI: Freight Unit Creation

### Enhancement Spot: `/SCMTMS/ES_FU_CREATION`
### BADI Interface: `/SCMTMS/IF_EX_FU_CREATION`

```abap
METHOD /scmtms/if_ex_fu_creation~enrich_fu.
  " Enrich freight unit with custom data from sales order
  " cs_fu_root: freight unit header — modify before save
  SELECT SINGLE zz_priority, zz_temperature_req
    FROM vbap
    WHERE vbeln = @cs_fu_root-ref_doc_id
      AND posnr = @cs_fu_root-ref_item_id
    INTO @DATA(ls_custom).

  cs_fu_root-zz_priority        = ls_custom-zz_priority.
  cs_fu_root-zz_temperature_req = ls_custom-zz_temperature_req.
ENDMETHOD.
```

## Vehicle Scheduling (VSR)

For road transport, TM can optimize vehicle routes using Vehicle Scheduling and Routing:

```abap
" Run VSR optimization
CALL FUNCTION '/SCMTMS/VSR_OPTIMIZE'
  EXPORTING
    it_fo_ids       = lt_freight_order_ids
    iv_vsr_profile  = lv_vsr_profile
  IMPORTING
    et_optimized    = DATA(lt_optimized_fos)
    et_return       = DATA(lt_messages).
```

## Planning Board Transactions

| Transaction | Purpose |
|---|---|
| `/SCMTMS/TP_LAND` | Land (road/rail) planning board |
| `/SCMTMS/TP_SEA` | Ocean planning board |
| `/SCMTMS/TP_AIR` | Air freight planning board |
| `/SCMTMS/TOC` | Transportation order cockpit |
| `/SCMTMS/MON_FU` | Freight unit monitor (unplanned FUs) |
