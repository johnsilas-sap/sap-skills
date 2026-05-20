# BAdI Implementation Reference (SE19)

## Creating an Implementation

```
SE19 → Create BAdI Implementation
  Enhancement spot:      /SCWM/ES_DLV_PROC         (optional — narrows selection)
  BAdI name:             /SCWM/EX_DLV_PROC_DLV
  Implementation name:   ZEWM_IM_DLV_PROC_CUSTOM
  Short text:            Custom delivery processing

  → System creates implementation class automatically
  → Class name: ZCL_IM_EWM_DLV_PROC_CUSTOM (or choose your own)
  → Interface /SCWM/IF_EX_DLV_PROC_DLV added to class automatically
```

## Implementation Class Template

```abap
CLASS zcl_im_ewm_dlv_proc_custom DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES /scwm/if_ex_dlv_proc_dlv.    " BAdI interface — added by SE19

  PRIVATE SECTION.
    " Add private helper methods here
ENDCLASS.

CLASS zcl_im_ewm_dlv_proc_custom IMPLEMENTATION.

  METHOD /scwm/if_ex_dlv_proc_dlv~check_delivery.
    " Parameters defined in BAdI interface:
    "   IMPORTING is_header TYPE /scwm/s_dlv_hdr
    "   EXPORTING et_messages TYPE /scwm/t_messages
    "             rv_reject   TYPE abap_bool

    " Example: reject delivery if weight exceeds threshold
    IF is_header-total_weight > 10000.
      rv_reject = abap_true.
      APPEND VALUE /scwm/s_message(
        msgty = 'E'
        msgid = 'ZEWM_MESSAGES'
        msgno = '001'
        msgv1 = is_header-delivery_no ) TO et_messages.
    ENDIF.
  ENDMETHOD.

  METHOD /scwm/if_ex_dlv_proc_dlv~set_delivery_data.
    " Called before delivery is saved
    " cs_result is CHANGING — modify to override delivery data

    " Example: set custom field based on carrier
    IF is_header-carrier_id = 'DHL'.
      cs_result-custom_route_code = 'DHL_EXPRESS'.
    ENDIF.
  ENDMETHOD.

ENDCLASS.
```

## Filter Values (SE19)

For filter-dependent BAdIs, set which filter values activate this implementation:

```
SE19 → Open implementation → Filter Values tab:
  Filter field: LGNUM
  Values:
    1710    " This implementation only runs for warehouse 1710
    1720    " Also runs for 1720

" Leave blank = run for all filter values (no restriction)
```

## Activating / Deactivating

```
SE19 → Implementation → Activate   (Ctrl+F3)
  " Implementation is now active — will be called at runtime

SE19 → Implementation → Deactivate
  " Temporarily switches off without deleting

" Active status visible in:
SE19 → Display → Status: Active / Inactive
```

## Runtime Activation Check (for multiple-use BAdIs)

```abap
" Check if any implementation is active before calling:
DATA lo_badi TYPE REF TO /scwm/if_ex_dlv_proc_dlv.

GET BADI lo_badi FILTERS lgnum = ls_header-lgnum.

IF lo_badi IS BOUND.
  CALL BADI lo_badi->check_delivery
    EXPORTING is_header   = ls_header
    IMPORTING et_messages = lt_messages
              rv_reject   = lv_reject.
ENDIF.
```

## Multiple Implementations Active (LOOP AT)

```abap
" Multiple-use BAdI: collect all matching implementations then call each
DATA lt_badi_impls TYPE TABLE OF REF TO /scwm/if_ex_dlv_proc_dlv.

GET BADI lt_badi_impls
  FILTERS lgnum = ls_header-lgnum.

LOOP AT lt_badi_impls INTO DATA(lo_impl).
  CALL BADI lo_impl->check_delivery
    EXPORTING is_header   = ls_header
    CHANGING  cs_result   = ls_result.
ENDLOOP.
```

## Classic BAdI Implementation (Legacy)

```abap
" Classic BAdI uses adapter class pattern

" 1. In SE19: create implementation for classic BAdI
"    System creates adapter class: ZCL_EX_MY_CLASSIC_BADI
"    Adapter inherits from: CL_EX_MY_CLASSIC_BADI (SAP base class)

CLASS zcl_ex_my_classic_badi DEFINITION
  PUBLIC
  INHERITING FROM cl_ex_my_classic_badi
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.
    METHODS my_method REDEFINITION.
ENDCLASS.

CLASS zcl_ex_my_classic_badi IMPLEMENTATION.
  METHOD my_method.
    " Implementation here
  ENDMETHOD.
ENDCLASS.

" 2. Calling code (in SAP standard or custom FM):
DATA lo_exit TYPE REF TO if_ex_my_classic_badi.

CALL METHOD cl_exithandler=>get_instance
  CHANGING instance = lo_exit.

IF lo_exit IS BOUND.
  CALL METHOD lo_exit->my_method
    EXPORTING iv_param = lv_value.
ENDIF.
```

## Finding Your Active Implementations

```
SE19 → Own Implementations
  → Lists all implementations in your system namespace (Z*/Y*)

SE19 → Filter by BAdI name
  → Shows all implementations for a specific BAdI

SE18 → BAdI → Check Where-Used
  → Shows all implementations for this BAdI across all namespaces
```

## Transport

BAdI implementations are Workbench objects — transport via SE09/SE10:

```
SE09 → Create transport request
  Object type: ENHO    (Enhancement Implementation)
  Object name: ZEWM_IM_DLV_PROC_CUSTOM

" The implementation class (SE24) is a separate transport object:
  Object type: CLAS
  Object name: ZCL_IM_EWM_DLV_PROC_CUSTOM
```

## Common Errors

### "Implementation class does not implement interface method"

```
Cause:  SAP added a new method to the BAdI interface after your impl was created
Fix:    SE24 → Open impl class → Add missing method → Activate
        Or: SE19 → Edit → Regenerate implementation class
```

### "BAdI not found" at GET BADI

```
Cause:  BAdI definition not active (business function not switched on)
Fix:    SFW5 → Check if required business function is active
        Or: SE18 → Check if BAdI definition exists in this system
```

### Implementation not called at runtime

```
Cause 1: Implementation is inactive → SE19 → Activate
Cause 2: Filter value does not match → SE19 → Filter Values → check values
Cause 3: Multiple-use=No and another implementation is already active
Fix:     SE19 → Deactivate other implementation first
```
