# BAdI Testing and Debugging Reference

## Confirming Your Implementation is Called

### Breakpoint in Method

```abap
METHOD /scwm/if_ex_dlv_proc_dlv~check_delivery.
  BREAK-POINT.    " Hard breakpoint — stops in ABAP debugger
  " Or: set external breakpoint in SE24 line editor
```

### Runtime Check — Count Active Implementations

```abap
DATA lt_impls TYPE TABLE OF REF TO /scwm/if_ex_dlv_proc_dlv.

GET BADI lt_impls
  FILTERS lgnum = lv_lgnum.

DATA(lv_count) = lines( lt_impls ).
" lv_count = 0 → no active implementation for this filter value
" lv_count > 0 → implementation(s) found
```

## SE19 Test Function

```
SE19 → Open implementation → Test (Ctrl+F8)
  → Launches SE37-like test screen for the BAdI interface
  → Fill in parameter values manually
  → Execute → implementation method runs in foreground
  → Inspect return values

" Note: not all BAdIs support this (complex object params may fail)
```

## Checking Implementation Status

```
SE19 → Enter implementation name → Display
  Status: Active / Inactive (shown in header)
  Filter values visible in Filter Values tab

" Batch check — all your implementations:
SE19 → Own Implementations
  → See status column for all Z* implementations at once
```

## Checking That GET BADI Resolves

```abap
" Add temporary debug code:
DATA lo_badi TYPE REF TO /scwm/if_ex_dlv_proc_dlv.

GET BADI lo_badi FILTERS lgnum = '1710'.

IF lo_badi IS NOT BOUND.
  " No active implementation found
  " Check: SE19 → is impl active? filter value correct?
  MESSAGE 'BAdI not resolved for LGNUM 1710' TYPE 'I'.
ELSE.
  " Implementation found — call it
  CALL BADI lo_badi->check_delivery ...
ENDIF.
```

## ABAP Debugger — Watching BAdI Resolution

```
1. Set external breakpoint (Ctrl+Shift+F12 in ABAP editor) at GET BADI statement
2. Trigger business transaction
3. Debugger stops at GET BADI
4. F5 (step into) → jumps into the framework resolution
5. F8 (continue) → skips framework, next stop is in your implementation method
```

## Finding Where a BAdI is Called (SE18 Where-Used)

```
SE18 → BAdI: /SCWM/EX_DLV_PROC_DLV → Where-Used

  Shows: CALL BADI /SCWM/EX_DLV_PROC_DLV->CHECK_DELIVERY
         in: Method /SCWM/CL_WM_DLV_PROC=>VALIDATE_DELIVERY
             Line: 234

" This tells you exactly when in the process flow your BAdI runs
" → Navigate to that method to understand the calling context
```

## Switch Framework (SFW5)

Some BAdIs are only activated when a business function is switched on:

```
SFW5 → Business Functions
  Check if required function is active:
    EWM_INB_DLV_ENHANC → Inbound delivery enhancements
    EWM_PICKING_ENHANC → Picking enhancements

  If not active:
    Set to "Active" → Confirm → system restart may be required

" After switching on, the BAdI definition becomes visible in SE18
```

## Testing Sequence for New Implementation

```
1. SE18 → Verify BAdI exists and find exact method signatures
2. SE19 → Create implementation, set filter values
3. SE24 → Implement method with BREAK-POINT in first line
4. SE19 → Activate implementation
5. Trigger business transaction (e.g., save delivery in EWM)
6. Debugger stops in your method → verify parameters
7. Remove BREAK-POINT → add real logic
8. Test again without debugger
9. Create transport request (SE09)
```

## Common Issues and Fixes

### Method not called — filter mismatch

```
Symptom: Breakpoint in method never hit
Check:   SE19 → Filter Values → is LGNUM = '1710' (not '1710 ' with space)?
         The filter value must match exactly including leading zeros
Fix:     LGNUM is typically 4 chars: '1710' not '710'
```

### Wrong method signature

```
Symptom: Syntax error in implementation class
Cause:   SAP upgraded and changed BAdI interface method parameters
Fix:     SE24 → Delete method → Re-add via interface inheritance
         SE19 → Edit → Regenerate implementation (may reset method)
```

### Implementation active but not called

```
Symptom: SE19 shows Active, but method never executes
Cause 1: Business function not active (SFW5)
Cause 2: Calling code has guard condition that skips GET BADI
         (e.g., only called for certain document types)
Fix:     SE18 → Where-Used → navigate to calling code → check guard conditions
         Add your document type/movement type to the guard condition scope
```

### Multiple implementations conflict (CHANGING parameter)

```
Symptom: Two implementations both modify CS_RESULT — last one wins
Fix:     Design implementations to work additively, not overwrite each other
         Use CS_RESULT fields that are distinct per implementation
         Consider making one implementation call-and-check others
```

## Logging BAdI Calls (Production Debugging)

```abap
METHOD /scwm/if_ex_dlv_proc_dlv~check_delivery.
  " Temporary structured log — remove after debugging
  DATA ls_log TYPE bal_s_msg.
  ls_log = VALUE bal_s_msg(
    msgty = 'I'
    msgid = 'ZEWM_DEBUG'
    msgno = '001'
    msgv1 = is_header-delivery_no
    msgv2 = is_header-lgnum ).
  CALL FUNCTION 'BAL_LOG_MSG_ADD'
    EXPORTING i_log_handle = gv_log_handle
              i_s_msg      = ls_log.
ENDMETHOD.
```

## Transport Checklist

```
□ Implementation object (ENHO): ZEWM_IM_DLV_PROC_CUSTOM
□ Implementation class (CLAS): ZCL_IM_EWM_DLV_PROC_CUSTOM
□ Any custom message class (MSAG): ZEWM_MESSAGES
□ Any custom types/structures (DTEL, TABL, SHLP): as needed
□ Activate in target system after transport (SE19 → Activate)
□ Verify filter values survive transport (SE19 → Filter Values)
```
