# Filter-Dependent BAdI Reference

## What Filters Do

Filters let multiple implementations coexist — each active for a different set of values. Only the implementation whose filter value matches the runtime value is called.

```
BAdI: /SCWM/EX_PICK_HO   (filter: LGNUM)
  Implementation A — Filter: 1710 → runs only for Hamburg warehouse
  Implementation B — Filter: 1720 → runs only for Stuttgart warehouse
  Implementation C — Filter: * (all) → runs for all warehouses
```

## Setting Filter Values in SE19

```
SE19 → Open implementation → Filter Values tab

  Single value:
    LGNUM = 1710

  Multiple values (OR):
    LGNUM = 1710
    LGNUM = 1720

  Wildcard (all values):
    (leave empty = run for all filter values)

  Range (not supported in standard — use multiple values or empty)
```

## Calling a Filter-Dependent BAdI

```abap
" Single filter
DATA lo_badi TYPE REF TO /scwm/if_ex_pick_ho.

GET BADI lo_badi
  FILTERS lgnum = ls_task-lgnum.

IF lo_badi IS BOUND.
  CALL BADI lo_badi->set_picking_data
    EXPORTING is_task = ls_task
    CHANGING  cs_hu   = ls_hu.
ENDIF.
```

```abap
" Multiple filters
DATA lo_badi TYPE REF TO /scwm/if_ex_wt_proc.

GET BADI lo_badi
  FILTERS
    lgnum = ls_wt-lgnum
    bwlvs = ls_wt-bwlvs.     " Movement type — both must match

CALL BADI lo_badi->process_wt
  EXPORTING is_wt = ls_wt.
```

## Multiple Implementations with Same Filter

When multiple-use = ✓, multiple implementations can share the same filter value:

```
Implementation A — Filter: 1710 — checks weight limit
Implementation B — Filter: 1710 — sets custom label field
Implementation C — Filter: 1710 — writes audit log

" All three run for warehouse 1710 — order is not guaranteed
```

```abap
DATA lt_impls TYPE TABLE OF REF TO /scwm/if_ex_dlv_proc.
GET BADI lt_impls FILTERS lgnum = '1710'.

LOOP AT lt_impls INTO DATA(lo_impl).
  CALL BADI lo_impl->process
    EXPORTING is_header = ls_header
    CHANGING  cs_result = ls_result.
ENDLOOP.
```

## Common Filter Types in EWM

| Filter Field | Type | Filters By |
|---|---|---|
| `LGNUM` | `/SCWM/LGNUM` | Warehouse number |
| `BWLVS` | `/SCWM/BWLVS` | Warehouse movement type |
| `LGTYP` | `/SCWM/LGTYP` | Storage type |
| `RDOCCAT` | `/SCDL/REC_DOCCAT` | Document category (delivery, TO, etc.) |
| `PROCTY` | `/SCWM/PROCTY` | Process type |

## Common Filter Types in TM

| Filter Field | Type | Filters By |
|---|---|---|
| `FROTYP` | `/SCMTMS/FRO_TYPE` | Freight order type |
| `TRNMOD` | `/SCMTMS/TRANS_MODE` | Transportation mode |
| `CARRIER` | `/SCMTMS/CARRIER_ID` | Carrier ID |

## Filter-Dependent Implementation: Full Pattern

```abap
CLASS zcl_im_ewm_wt_proc_1710 DEFINITION
  PUBLIC FINAL CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES /scwm/if_ex_wt_proc.
ENDCLASS.

CLASS zcl_im_ewm_wt_proc_1710 IMPLEMENTATION.

  METHOD /scwm/if_ex_wt_proc~before_create_wt.
    " Only called for warehouse 1710 (filter set in SE19)
    " is_lgnum = '1710' guaranteed at runtime

    " Warehouse-specific logic here
    IF is_wt_head-bwlvs = 'Z001'.   " Custom movement type
      " Custom processing for Hamburg warehouse
    ENDIF.
  ENDMETHOD.

ENDCLASS.
```

```
SE19 → Implementation: ZEWM_IM_WT_PROC_1710
  Filter Values:
    LGNUM = 1710
  Status: Active
```

## No-Filter BAdI (Runs for All Values)

```abap
" BAdI without filter — always runs if active
DATA lo_badi TYPE REF TO /scwm/if_ex_delivery_note.

GET BADI lo_badi.   " No FILTERS clause

IF lo_badi IS BOUND.
  CALL BADI lo_badi->enrich_data
    CHANGING cs_form_data = ls_form_data.
ENDIF.
```

## Debugging Filter Execution

```abap
" Add breakpoint in GET BADI call to see resolved filter values
" Add breakpoint in implementation method to confirm it's being called

" Check active implementations at runtime:
DATA lt_impls TYPE TABLE OF REF TO /scwm/if_ex_wt_proc.
GET BADI lt_impls FILTERS lgnum = lv_lgnum.
DATA(lv_count) = lines( lt_impls ).
" lv_count = 0 means no active implementation matches the filter
```
