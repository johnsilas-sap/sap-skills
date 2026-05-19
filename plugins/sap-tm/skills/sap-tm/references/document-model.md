# TM Document Model Reference

## Document Hierarchy

```
Sales/Purchase Order (ERP)
  └── Freight Unit (FU)          — shippable unit, demand-side
        └── Freight Order (FO)   — carrier + route + stages, supply-side
              └── Stage          — individual transport leg
                    └── Transportation Unit (TU) — vehicle/container
```

Additional standalone documents:
- **Freight Booking (FB)** — pre-booked ocean/air slot with carrier
- **Transportation Order (TO)** — internal yard move or cross-dock
- **Freight Agreement (FA)** — contracted rate structure with a carrier

## Document Categories (TORCAT)

| Constant | Value | Document |
|---|---|---|
| `/SCMTMS/IF_TOR_CONST=>GC_TORCAT_FO` | `FO` | Freight Order |
| `/SCMTMS/IF_TOR_CONST=>GC_TORCAT_FB` | `FB` | Freight Booking |
| `/SCMTMS/IF_TOR_CONST=>GC_TORCAT_TO` | `TO` | Transportation Order |
| `/SCMTMS/IF_TOR_CONST=>GC_TORCAT_FU` | `FU` | Freight Unit |

## Freight Order Lifecycle

```
Created → Planned → In Execution → Completed → Settled
                ↓
           Cancelled
```

### Lifecycle Status Constants

| Status | Constant |
|---|---|
| Created | `/SCMTMS/IF_TOR_CONST=>GC_LIFECYCLE_CREATED` |
| In Planning | `/SCMTMS/IF_TOR_CONST=>GC_LIFECYCLE_IN_PLANNING` |
| In Execution | `/SCMTMS/IF_TOR_CONST=>GC_LIFECYCLE_IN_EXEC` |
| Completed | `/SCMTMS/IF_TOR_CONST=>GC_LIFECYCLE_COMPLETED` |
| Cancelled | `/SCMTMS/IF_TOR_CONST=>GC_LIFECYCLE_CANCELLED` |

## Reading Documents

### Read Freight Order by ID

```abap
DATA(lo_manager) = /scmtms/cl_tor_bo_manager=>get_instance( ).

DATA(lo_fo) = lo_manager->get_by_id(
  iv_tor_id  = lv_fo_id
  iv_torcat  = /scmtms/if_tor_const=>gc_torcat_fo ).

DATA(ls_root)   = lo_fo->get_root( ).
DATA(lt_stages) = lo_fo->get_stages( ).
DATA(lt_items)  = lo_fo->get_items( ).
```

### Read Freight Order Header Fields

```abap
" ls_root key fields:
" ls_root-tor_id       = Document ID
" ls_root-torcat       = Document category (FO/FB/TO)
" ls_root-carrier_id   = Assigned carrier (BP ID)
" ls_root-means_tra    = Means of transport (road/sea/air/rail)
" ls_root-dep_date     = Planned departure date
" ls_root-arr_date     = Planned arrival date
" ls_root-lifecycle    = Current lifecycle status
" ls_root-exec_status  = Execution status
" ls_root-src_loc_id   = Source location
" ls_root-dst_loc_id   = Destination location
" ls_root-gross_weight = Total gross weight
" ls_root-volume       = Total volume
" ls_root-currency     = Currency for charges
```

### Search Freight Orders

```abap
" By carrier and date range
SELECT tor_id, carrier_id, dep_date, arr_date, lifecycle
  FROM /scmtms/d_fro_h
  WHERE carrier_id  = @lv_carrier
    AND dep_date   >= @lv_from_date
    AND dep_date   <= @lv_to_date
    AND lifecycle  <> @( /scmtms/if_tor_const=>gc_lifecycle_cancelled )
  INTO TABLE @DATA(lt_fos).
```

### Read Stages

```abap
SELECT *
  FROM /scmtms/d_stage
  WHERE tor_id = @lv_fo_id
  ORDER BY stage_num
  INTO TABLE @DATA(lt_stages).

LOOP AT lt_stages ASSIGNING FIELD-SYMBOL(<stage>).
  " <stage>-stage_num    = Leg sequence number
  " <stage>-src_loc_id   = Departure location
  " <stage>-dst_loc_id   = Arrival location
  " <stage>-dep_date     = Planned departure
  " <stage>-arr_date     = Planned arrival
  " <stage>-carrier_id   = Stage carrier (may differ from FO carrier)
  " <stage>-means_tra    = Mode of transport for this leg
ENDLOOP.
```

## Creating Documents

### Create Freight Order (Full Example)

```abap
DATA: ls_root   TYPE /scmtms/s_tor_root,
      lt_stages TYPE /scmtms/t_stage,
      lt_items  TYPE /scmtms/t_tor_item.

ls_root = VALUE /scmtms/s_tor_root(
  torcat      = /scmtms/if_tor_const=>gc_torcat_fo
  means_tra   = /scmtms/if_tor_const=>gc_means_tra_road
  src_loc_id  = 'LOC_PLANT_1000'
  dst_loc_id  = 'LOC_CUSTOMER_500'
  dep_date    = '20260601'
  arr_date    = '20260603'
  carrier_id  = lv_carrier_bp ).

APPEND VALUE /scmtms/s_stage(
  stage_num  = 1
  src_loc_id = 'LOC_PLANT_1000'
  dst_loc_id = 'LOC_CUSTOMER_500'
  dep_date   = '20260601'
  arr_date   = '20260603' ) TO lt_stages.

" Add freight unit reference
APPEND VALUE /scmtms/s_tor_item(
  fu_id      = lv_freight_unit_id
  quantity   = lv_quantity
  uom        = lv_uom ) TO lt_items.

CALL METHOD /scmtms/cl_tor_bo_factory=>create_tor
  EXPORTING
    is_root   = ls_root
    it_stages = lt_stages
    it_items  = lt_items
  IMPORTING
    ev_tor_id = DATA(lv_new_fo_id)
    et_return = DATA(lt_messages).

COMMIT WORK.
```

## Means of Transport Constants

| Constant | Mode |
|---|---|
| `/SCMTMS/IF_TOR_CONST=>GC_MEANS_TRA_ROAD` | Road (truck) |
| `/SCMTMS/IF_TOR_CONST=>GC_MEANS_TRA_SEA` | Ocean |
| `/SCMTMS/IF_TOR_CONST=>GC_MEANS_TRA_AIR` | Air |
| `/SCMTMS/IF_TOR_CONST=>GC_MEANS_TRA_RAIL` | Rail |
| `/SCMTMS/IF_TOR_CONST=>GC_MEANS_TRA_INLAND_WATER` | Inland waterway |

## Document Relationships

```abap
" Find freight orders for a freight unit
SELECT tor_id
  FROM /scmtms/d_fro_i
  WHERE fu_id = @lv_freight_unit_id
  INTO TABLE @DATA(lt_fo_ids).

" Find freight units on a freight order
SELECT fu_id, quantity, uom
  FROM /scmtms/d_fro_i
  WHERE tor_id = @lv_fo_id
  INTO TABLE @DATA(lt_items).

" Navigate from sales order to freight unit
SELECT fu_id
  FROM /scmtms/d_fub_ref
  WHERE ref_doc_type = 'SO'
    AND ref_doc_id   = @lv_sales_order
  INTO TABLE @DATA(lt_fu_ids).
```
