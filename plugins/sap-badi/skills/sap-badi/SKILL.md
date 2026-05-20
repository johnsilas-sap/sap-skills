---
name: sap-badi
description: |
  SAP BAdI (Business Add-In) development skill. Use when creating or implementing
  BAdIs in SE18/SE19, writing kernel BAdI implementations using GET BADI/CALL BADI,
  working with filter-dependent BAdIs, enhancement spots, classic BAdIs via
  CL_EXITHANDLER, or finding and implementing EWM/TM enhancement points. Covers
  BAdI definition structure, implementation class patterns, filter values, multiple
  implementations, activation, and the full EWM and TM BAdI catalogs for warehouse
  operations, RF mobile, delivery processing, carrier selection, and charge calculation.
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-20"
  sources:
    - "https://help.sap.com/docs/ABAP_PLATFORM_NEW/753088fc00704d0a80e7fbd6803c8adb"
    - "https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/7b24a64d9d0941bda1afa753263d9e39"
---

# SAP BAdI Development Skill

## Related Skills

- **sap-ewm**: EWM warehouse operations — BAdIs for picking, packing, RF, delivery
- **sap-tm**: TM freight management — BAdIs for carrier selection, charge calculation
- **sap-abap**: ABAP OO patterns used in BAdI implementation classes

## BAdI Generations

| Generation | Transactions | Call Pattern | Notes |
|---|---|---|---|
| Classic BAdI | SE18 / SE19 | `CL_EXITHANDLER=>GET_INSTANCE` | Single implementation, adapter class |
| Kernel (New) BAdI | SE18 / SE19 | `GET BADI` / `CALL BADI` | Multiple implementations, filter support |
| Enhancement Spot | SE18 / SE19 | `GET BADI` / `CALL BADI` | Groups related BAdIs into one spot |

## Kernel BAdI Pattern (Modern — Preferred)

### Finding the BAdI

```
SE18 → Enter BAdI name → Display
  Shows: Interface, Methods, Filter type (if any)
  Enhancement spot: /SCWM/ES_DELIVERY  (groups related BAdIs)

" Or search by topic:
SE18 → Where-Used List → Search by enhancement spot or package
```

### Creating an Implementation

```
SE19 → Create BAdI Implementation
  BAdI name:     /SCWM/EX_DLV_PROC_DLV
  Impl. name:    ZEWM_DLV_PROC_CUSTOM
  Description:   Custom delivery processing logic

  → Creates implementation class automatically:
    Class name:  ZCL_IM_EWM_DLV_PROC_CUSTOM
    Interface:   /SCWM/IF_EX_DLV_PROC_DLV (implemented automatically)

  → Activate implementation: SE19 → Activate
```

### Calling the BAdI (done by SAP — for reference only)

```abap
" SAP framework calls kernel BAdIs — you implement, not call
" For custom BAdIs you define yourself:

DATA lo_badi TYPE REF TO zif_my_badi.

GET BADI lo_badi
  FILTERS
    filter_field = lv_warehouse_no.    " If filter-dependent

CALL BADI lo_badi->method_name
  EXPORTING iv_param = lv_value
  IMPORTING ev_result = lv_result.
```

### Multiple Implementations (LOOP AT)

```abap
" When a BAdI allows multiple active implementations:
DATA lt_badis TYPE TABLE OF REF TO zif_my_badi.

GET BADI lt_badis
  FILTERS filter_field = lv_warehouse_no.

LOOP AT lt_badis INTO DATA(lo_badi).
  CALL BADI lo_badi->process
    EXPORTING is_data = ls_data
    CHANGING  cs_result = ls_result.
ENDLOOP.
```

## Classic BAdI Pattern (Legacy Systems)

```abap
" Classic BAdI — used in older SAP systems
DATA lo_badi TYPE REF TO if_ex_my_classic_badi.

CALL METHOD cl_exithandler=>get_instance
  CHANGING instance = lo_badi.

IF lo_badi IS BOUND.
  CALL METHOD lo_badi->my_method
    EXPORTING iv_param = lv_value.
ENDIF.
```

## Implementation Class Pattern

```abap
CLASS zcl_im_ewm_my_badi DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES /scwm/if_ex_my_badi.   " BAdI interface
  PRIVATE SECTION.
    " Private helper methods here
ENDCLASS.

CLASS zcl_im_ewm_my_badi IMPLEMENTATION.

  METHOD /scwm/if_ex_my_badi~my_method.
    " IS_HEADER, IT_ITEMS etc. are parameters defined in BAdI interface
    " Implement enhancement logic here
  ENDMETHOD.

ENDCLASS.
```

## Filter-Dependent BAdIs

Filters restrict which implementation runs based on runtime values (warehouse number, document type, plant, etc.).

```abap
" BAdI with filter — only the implementation matching the filter executes
GET BADI lo_badi
  FILTERS
    lgnum = ls_header-lgnum.        " Only impl. for this warehouse runs

CALL BADI lo_badi->process
  EXPORTING is_header = ls_header.
```

```
SE19 → Implementation → Filter Values tab:
  Filter: LGNUM
  Value:  1710        " This impl. only runs for warehouse 1710
```

## Key Transactions

| Transaction | Purpose |
|---|---|
| `SE18` | BAdI Builder — view definition, interface, methods, filter |
| `SE19` | BAdI Implementation — create/edit implementation |
| `SE24` | Class Builder — edit implementation class directly |
| `SXSB_SPOT` | Enhancement spot browser |
| `SPRO` | Customizing — some BAdIs activated via IMG |
| `SFW5` | Switch Framework — activate business functions that enable BAdIs |
