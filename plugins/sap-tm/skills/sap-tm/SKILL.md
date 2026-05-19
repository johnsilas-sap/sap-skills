---
name: sap-tm
description: |
  SAP Transportation Management (TM) development skill. Use when working with freight
  orders, transportation orders, freight bookings, carrier selection, charge calculation,
  transportation planning, dangerous goods, track and trace, settlement, output/forms,
  BADIs and enhancement spots (/SCMTMS/*), TM ABAP APIs, or TM IDoc integration
  (SHPMNT, TPSSHT). Covers SAP TM 9.x and embedded TM in S/4HANA.
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-19"
  tm_release: "SAP TM 9.4+ / S/4HANA TM 2020+"
  sources:
    - "https://help.sap.com/docs/SAP_TRANSPORTATION_MANAGEMENT"
    - "https://help.sap.com/docs/S4HANA_ON-PREMISE"
---

# SAP TM Development Skill

## Related Skills

- **sap-abap**: General ABAP syntax, internal tables, OO patterns, ABAP SQL
- **sap-ewm**: EWM ↔ TM integration — freight order triggers warehouse orders
- **sap-idoc**: TM IDoc types (SHPMNT, TPSSHT) and partner profiles
- **sap-btp-connectivity**: Exposing TM services to external carriers/partners via BTP
- **sap-api-style**: Documenting TM OData or REST APIs

## Document Model

TM organizes transportation in a hierarchy of documents:

```
Freight Unit (FU)         — smallest shippable unit (from sales/purchase order)
  └── Freight Order (FO)  — carrier assignment + stages for actual transport
        └── Stage         — individual leg of a multi-leg shipment
              └── Transportation Unit (TU) — vehicle/container carrying the freight
```

Additional documents:
- **Freight Booking (FB)** — ocean/air container booking with a carrier
- **Transportation Order (TO)** — internal transport (yard moves, cross-docking)
- **Freight Agreement (FA)** — contracted rates with a carrier
- **Settlement Document** — charges posted to FI/CO

## Key Tables

| Table | Description |
|---|---|
| `/SCMTMS/D_FRO_H` | Freight order header |
| `/SCMTMS/D_FRO_I` | Freight order items (freight units) |
| `/SCMTMS/D_STAGE` | Freight order stages (legs) |
| `/SCMTMS/D_TOR_H` | Transportation order header |
| `/SCMTMS/D_TOR_I` | Transportation order items |
| `/SCMTMS/D_FUB_H` | Freight unit header |
| `/SCMTMS/D_FUB_I` | Freight unit items |
| `/SCMTMS/D_CHARGE` | Charge calculation results |
| `/SCMTMS/D_SETTL` | Settlement documents |
| `/SCMTMS/D_TORQUE` | Transportation request queue |
| `/SAPTRX/APPL_OB` | Track and trace application objects |

## Freight Order API

### Read Freight Order

```abap
DATA: lo_fro    TYPE REF TO /scmtms/if_tor_bo_manager,
      lo_fo     TYPE REF TO /scmtms/if_tor_bo,
      ls_header TYPE /scmtms/s_tor_root.

lo_fro = /scmtms/cl_tor_bo_manager=>get_instance( ).

lo_fo = lo_fro->get_by_id(
  iv_tor_id  = lv_freight_order_id
  iv_torcat  = /scmtms/if_tor_const=>gc_torcat_fo ).  " Freight Order category

ls_header = lo_fo->get_root( ).
```

### Create Freight Order

```abap
DATA: ls_fo_data TYPE /scmtms/s_tor_root,
      lt_stages  TYPE /scmtms/t_stage,
      lt_items   TYPE /scmtms/t_tor_item.

ls_fo_data = VALUE /scmtms/s_tor_root(
  torcat     = /scmtms/if_tor_const=>gc_torcat_fo
  means_tra  = /scmtms/if_tor_const=>gc_means_tra_road
  carrier_id = lv_carrier_bp_id
  dep_date   = lv_departure_date
  arr_date   = lv_arrival_date ).

" Add stage (leg)
APPEND VALUE /scmtms/s_stage(
  src_loc_id = lv_source_location
  dst_loc_id = lv_dest_location
  dep_date   = lv_departure_date
  arr_date   = lv_arrival_date ) TO lt_stages.

CALL METHOD /scmtms/cl_tor_bo_factory=>create_tor
  EXPORTING
    is_root    = ls_fo_data
    it_stages  = lt_stages
    it_items   = lt_items
  IMPORTING
    ev_tor_id  = DATA(lv_new_fo_id)
    et_return  = DATA(lt_messages).
```

### Change Freight Order Status

```abap
" In process → Completed
CALL METHOD lo_fo->set_lifecycle_status
  EXPORTING
    iv_status  = /scmtms/if_tor_const=>gc_lifecycle_completed
  IMPORTING
    et_return  = DATA(lt_messages).

CALL METHOD lo_fro->save( ).
COMMIT WORK.
```

## Carrier Selection

### Automatic Carrier Selection

TM uses carrier selection profiles configured in Customizing. To trigger programmatically:

```abap
CALL FUNCTION '/SCMTMS/CARRIER_SEL_EXECUTE'
  EXPORTING
    iv_tor_id      = lv_freight_order_id
    iv_sel_profile = lv_carrier_sel_profile
  IMPORTING
    et_carrier     = DATA(lt_proposed_carriers)
    et_return      = DATA(lt_messages).
```

### Read Proposed Carriers

```abap
LOOP AT lt_proposed_carriers ASSIGNING FIELD-SYMBOL(<carrier>).
  " <carrier>-bp_id     = Business Partner ID
  " <carrier>-rank      = Ranking position
  " <carrier>-total_cost = Calculated cost
  " <carrier>-currency  = Cost currency
ENDLOOP.
```

### Assign Carrier to Freight Order

```abap
CALL FUNCTION '/SCMTMS/FRO_CARRIER_ASSIGN'
  EXPORTING
    iv_tor_id    = lv_freight_order_id
    iv_bp_id     = lv_selected_carrier_id
    iv_commit    = abap_true
  IMPORTING
    et_return    = DATA(lt_messages).
```

## Charge Calculation

### Trigger Charge Calculation

```abap
CALL FUNCTION '/SCMTMS/CHARGE_CALC_EXECUTE'
  EXPORTING
    iv_tor_id       = lv_freight_order_id
    iv_calc_profile = lv_calculation_profile
  IMPORTING
    et_charges      = DATA(lt_charges)
    et_return       = DATA(lt_messages).
```

### Read Calculated Charges

```abap
SELECT *
  FROM /scmtms/d_charge
  WHERE tor_id = @lv_freight_order_id
  INTO TABLE @DATA(lt_charges).

LOOP AT lt_charges ASSIGNING FIELD-SYMBOL(<charge>).
  " <charge>-charge_type = charge category (freight, surcharge, tax)
  " <charge>-calc_base   = calculation base (weight, volume, distance)
  " <charge>-amount      = charge amount
  " <charge>-currency    = currency key
ENDLOOP.
```

## Transportation Planning

### Create Freight Unit from Sales Order

```abap
CALL FUNCTION '/SCMTMS/FU_CREATE_FROM_SO'
  EXPORTING
    iv_vbeln    = lv_sales_order
    iv_posnr    = lv_item
  IMPORTING
    ev_fu_id    = DATA(lv_freight_unit_id)
    et_return   = DATA(lt_messages).
```

### Planning Cockpit (Programmatic)

```abap
" Trigger automatic planning for a set of freight units
DATA: lt_fu_ids TYPE /scmtms/t_tor_id_tab.
APPEND lv_freight_unit_id TO lt_fu_ids.

CALL FUNCTION '/SCMTMS/PLANNING_EXECUTE'
  EXPORTING
    it_fu_ids       = lt_fu_ids
    iv_plan_profile = lv_planning_profile
  IMPORTING
    et_fro_ids      = DATA(lt_created_fro_ids)
    et_return       = DATA(lt_messages).
```

## Key BADIs and Enhancement Spots

| BADI Interface | Enhancement Spot | When to Use |
|---|---|---|
| `/SCMTMS/IF_EX_FRO_CHANGE` | `/SCMTMS/ES_FRO_CHANGE` | React to freight order create/change/delete |
| `/SCMTMS/IF_EX_CHARGE_CALC` | `/SCMTMS/ES_CHARGE_CALC` | Override or add charges during calculation |
| `/SCMTMS/IF_EX_CARRIER_SEL` | `/SCMTMS/ES_CARRIER_SEL` | Filter or rank proposed carriers |
| `/SCMTMS/IF_EX_PLANNING` | `/SCMTMS/ES_PLANNING` | Influence automatic planning decisions |
| `/SCMTMS/IF_EX_FU_CREATION` | `/SCMTMS/ES_FU_CREATION` | Modify freight unit on creation |
| `/SCMTMS/IF_EX_SETTLEMENT` | `/SCMTMS/ES_SETTLEMENT` | Influence settlement/invoicing |
| `/SCMTMS/IF_EX_OUTPUT` | `/SCMTMS/ES_OUTPUT` | Custom output (forms, EDI) on TM events |
| `/SCMTMS/IF_EX_DG_CHECK` | `/SCMTMS/ES_DG_CHECK` | Custom dangerous goods validations |

### BADI Example: Custom Charge Addition

```abap
CLASS zcl_tm_custom_charge DEFINITION PUBLIC.
  PUBLIC SECTION.
    INTERFACES /scmtms/if_ex_charge_calc.
ENDCLASS.

CLASS zcl_tm_custom_charge IMPLEMENTATION.
  METHOD /scmtms/if_ex_charge_calc~enrich_charges.
    " Add a custom fuel surcharge based on distance
    LOOP AT ct_charges ASSIGNING FIELD-SYMBOL(<charge>).
      IF <charge>-charge_type = 'FREIGHT'.
        APPEND VALUE /scmtms/s_charge(
          charge_type = 'FUEL_SUR'
          amount      = <charge>-amount * '0.15'  " 15% fuel surcharge
          currency    = <charge>-currency
          calc_base   = 'FREIGHT' ) TO ct_charges.
      ENDIF.
    ENDLOOP.
  ENDMETHOD.
ENDCLASS.
```

### BADI Example: Carrier Ranking Override

```abap
METHOD /scmtms/if_ex_carrier_sel~rank_carriers.
  " Prefer preferred carrier from custom agreement table
  LOOP AT ct_carriers ASSIGNING FIELD-SYMBOL(<c>).
    SELECT SINGLE preferred FROM zcarrier_pref
      WHERE bp_id = @<c>-bp_id
      INTO @DATA(lv_pref).
    IF lv_pref = abap_true.
      <c>-rank = 1.
    ENDIF.
  ENDLOOP.
  SORT ct_carriers BY rank.
ENDMETHOD.
```

## Track and Trace

```abap
" Report a tracking event on a freight order
CALL FUNCTION '/SAPTRX/TRACK_EVENT_POST'
  EXPORTING
    iv_appl_table   = '/SCMTMS/D_FRO_H'
    iv_appl_key     = lv_freight_order_id
    iv_event_code   = 'DEPARTURE'          " Standard or custom event code
    iv_actual_date  = sy-datum
    iv_actual_time  = sy-uzeit
    iv_location_id  = lv_current_location
  IMPORTING
    et_return       = DATA(lt_messages).

" Read tracking history
SELECT *
  FROM /saptrx/appl_ob
  WHERE appl_table = '/SCMTMS/D_FRO_H'
    AND appl_key   = @lv_freight_order_id
  ORDER BY event_date, event_time
  INTO TABLE @DATA(lt_events).
```

## Settlement and Invoicing

```abap
" Trigger settlement for a completed freight order
CALL FUNCTION '/SCMTMS/SETTLEMENT_CREATE'
  EXPORTING
    iv_tor_id    = lv_freight_order_id
    iv_commit    = abap_true
  IMPORTING
    ev_settl_id  = DATA(lv_settlement_id)
    et_return    = DATA(lt_messages).

" Read settlement document
SELECT *
  FROM /scmtms/d_settl
  WHERE tor_id = @lv_freight_order_id
  INTO TABLE @DATA(lt_settlements).
```

## Key Transactions

| Transaction | Purpose |
|---|---|
| `/SCMTMS/MON_FO` | Freight order monitor — main operational view |
| `/SCMTMS/TOC` | Transportation order cockpit |
| `/SCMTMS/TP_LAND` | Land transportation planning board |
| `/SCMTMS/TP_SEA` | Ocean transportation planning |
| `/SCMTMS/TP_AIR` | Air transportation planning |
| `/SCMTMS/CARRIER` | Carrier master maintenance |
| `/SCMTMS/FA` | Freight agreement maintenance |
| `/SCMTMS/CHARGE` | Charge management |
| `/SCMTMS/SETTL` | Settlement cockpit |
| `WE02` | IDoc monitor (SHPMNT, TPSSHT) |
