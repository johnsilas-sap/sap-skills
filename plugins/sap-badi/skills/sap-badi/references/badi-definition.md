# BAdI Definition Reference (SE18)

## What SE18 Shows

```
SE18 → Enter BAdI Name → Display

  Definition tab:
    BAdI name:        /SCWM/EX_DLV_PROC_DLV
    Short text:       Delivery Processing - Delivery
    Enhancement spot: /SCWM/ES_DLV_PROC
    Multiple use:     ✓ (multiple implementations can be active)
    Filter-dependent: ✓ (filter: LGNUM)

  Interface tab:
    Interface name:   /SCWM/IF_EX_DLV_PROC_DLV
    Methods:          (list of all callable methods)

  Documentation tab:
    SAP documentation for the BAdI

  Where-Used tab:
    Where SAP calls this BAdI in standard code
```

## Enhancement Spots

Enhancement spots group related BAdIs. Browse all BAdIs in a spot:

```
SE18 → Enhancement Spot → /SCWM/ES_DLV_PROC
  Shows all BAdIs within the spot:
    /SCWM/EX_DLV_PROC_DLV       Delivery Processing - Delivery
    /SCWM/EX_DLV_PROC_ITEM      Delivery Processing - Items
    /SCWM/EX_DLV_PROC_PACK      Delivery Processing - Packing
    /SCWM/EX_DLV_PROC_CLOSE     Delivery Processing - Close
```

## Understanding the Interface

```
SE18 → Interface tab → double-click interface name → SE24

  Interface: /SCWM/IF_EX_DLV_PROC_DLV
    Methods:
      SET_DELIVERY_DATA
        Parameters:
          IS_HEADER   TYPE /SCWM/S_DLV_HDR    " Importing
          IT_ITEMS    TYPE /SCWM/T_DLV_ITEM    " Importing
          CS_RESULT   TYPE /SCWM/S_DLV_RESULT  " Changing — modify this

      CHECK_DELIVERY
        Parameters:
          IS_HEADER   TYPE /SCWM/S_DLV_HDR
          ET_MESSAGES TYPE /SCWM/T_MESSAGES    " Exporting — add error messages here
          RV_REJECT   TYPE ABAP_BOOL           " Return — set true to reject
```

## Reading Method Parameters

For each method, use SE24 to see the full parameter list:

```
SE24 → Open interface → Methods tab → Select method → Parameters button

  Or: In SE18 → Interface → right-click method → Parameters
```

## Finding BAdIs by Topic

### Search by Enhancement Spot

```
SE18 → Enhancement Spot search
  Package:    /SCWM/      (EWM package namespace)
  → Lists all EWM enhancement spots and their BAdIs
```

### Search by Where-Used (What code calls this BAdI)

```
SE18 → BAdI → Where-Used List
  Shows every place in SAP standard code where GET BADI / CALL BADI is executed
  → Gives you the exact calling context (FM, method, report)
```

### Search by Package in SE84

```
SE84 → Repository Information System
  → Enhancement → Business Add-Ins → Definitions
  Package: /SCWM/
  → Lists all BAdI definitions in EWM
```

## Multiple-Use Flag

| Setting | Behavior |
|---|---|
| Multiple use = ✓ | Multiple implementations can be active simultaneously; all execute |
| Multiple use = ☐ | Only one implementation active at a time |

Most modern BAdIs (kernel) are multiple-use. If you create a second implementation when multiple-use is off, you must deactivate the existing one first.

## Filter Type

```
SE18 → Definition → Filter type: LGNUM
  Data type: /SCWM/LGNUM (warehouse number)

" At runtime, only the implementation whose filter value matches
" the current LGNUM is called.

" Multiple filters possible:
  Filter 1: LGNUM  (warehouse)
  Filter 2: BWLVS  (movement type)
  " → Implementation runs only when BOTH match
```

## Creating a Custom BAdI Definition (rare — usually you implement SAP's)

```
SE18 → Create → BAdI Definition
  Name:             ZMY_BADI
  Enhancement spot: ZMY_ES    (create spot first if needed)
  Multiple use:     ✓
  Filter:           LGNUM (optional)

  Interface:
    Create new interface: ZIF_EX_MY_BADI
    Add methods:
      MY_METHOD
        IS_INPUT   TYPE ...   Importing
        CS_OUTPUT  TYPE ...   Changing

" Then in ABAP code, call your own BAdI:
DATA lo_badi TYPE REF TO zif_ex_my_badi.
GET BADI lo_badi FILTERS lgnum = lv_lgnum.
CALL BADI lo_badi->my_method
  EXPORTING is_input  = ls_input
  CHANGING  cs_output = ls_output.
```
