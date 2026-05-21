# ABAP Exception Handling Reference

## Exception Class Hierarchy

```
CX_ROOT                       — all exceptions (never catch directly)
  ├── CX_STATIC_CHECK         — checked: caller MUST handle or declare
  ├── CX_DYNAMIC_CHECK        — unchecked: caller may handle or ignore
  └── CX_NO_CHECK             — always propagates (runtime errors)

" Custom exceptions:
ZCX_DELIVERY_ERROR            INHERITING FROM CX_STATIC_CHECK
ZCX_DELIVERY_NOT_FOUND        INHERITING FROM ZCX_DELIVERY_ERROR
ZCX_DELIVERY_LOCKED           INHERITING FROM ZCX_DELIVERY_ERROR
```

## Creating a Custom Exception Class (SE24 / ADT)

```abap
" New class → INHERITING FROM cx_static_check

CLASS zcx_delivery_error DEFINITION PUBLIC
  INHERITING FROM cx_static_check
  CREATE PUBLIC.

  PUBLIC SECTION.
    " Interface for T100 messages (recommended):
    INTERFACES if_t100_message.

    " Rich context attributes:
    DATA mv_delivery_no TYPE /scwm/de_lgnum.
    DATA mv_status      TYPE char1.
    DATA mv_operation   TYPE char20.

    " T100 message placeholders (v1-v4 shown in SY-MSG*):
    CONSTANTS:
      BEGIN OF gc_msg,
        delivery_not_found TYPE scx_t100key
          VALUE VALUE #( msgid = 'ZEWM_MSG' msgno = '001' attr1 = 'MV_DELIVERY_NO' ),
        invalid_status TYPE scx_t100key
          VALUE VALUE #( msgid = 'ZEWM_MSG' msgno = '002'
                         attr1 = 'MV_DELIVERY_NO' attr2 = 'MV_STATUS' ),
      END OF gc_msg.

    METHODS constructor
      IMPORTING
        iv_delivery_no TYPE /scwm/de_lgnum OPTIONAL
        iv_status      TYPE char1          OPTIONAL
        iv_operation   TYPE char20         OPTIONAL
        textid         LIKE if_t100_message=>t100key OPTIONAL
        previous       LIKE previous.
ENDCLASS.

CLASS zcx_delivery_error IMPLEMENTATION.
  METHOD constructor.
    CALL METHOD super->constructor
      EXPORTING
        textid   = textid
        previous = previous.
    mv_delivery_no = iv_delivery_no.
    mv_status      = iv_status.
    mv_operation   = iv_operation.

    " Default message if none provided:
    IF textid IS INITIAL.
      if_t100_message~t100key = gc_msg-delivery_not_found.
    ENDIF.
  ENDMETHOD.
ENDCLASS.
```

## Raising Exceptions

```abap
" Simple raise:
RAISE EXCEPTION TYPE zcx_delivery_error.

" With attributes:
RAISE EXCEPTION TYPE zcx_delivery_error
  EXPORTING
    iv_delivery_no = lv_delivery_no
    iv_status      = ls_delivery-status
    iv_operation   = 'POST_GI'.

" With specific T100 message key:
RAISE EXCEPTION TYPE zcx_delivery_error
  EXPORTING
    textid         = zcx_delivery_error=>gc_msg-invalid_status
    iv_delivery_no = lv_delivery_no
    iv_status      = ls_delivery-status.

" Re-raise caught exception as cause (exception chaining):
TRY.
    lo_repo->get_delivery( lv_delivery_no ).
  CATCH zcx_db_error INTO DATA(lo_db_ex).
    RAISE EXCEPTION TYPE zcx_delivery_error
      EXPORTING
        previous = lo_db_ex    " chain: inner cause preserved
        iv_delivery_no = lv_delivery_no.
ENDTRY.

" Resumable exception (caller can RESUME after catch):
RAISE RESUMABLE EXCEPTION TYPE zcx_delivery_warning
  EXPORTING iv_msg = 'Delivery is late — continue?'.
```

## TRY / CATCH / CLEANUP / ENDTRY

```abap
TRY.
    " ── Protected code ────────────────────────────────────────────────
    DATA(ls_delivery) = lo_repo->get_delivery( lv_no ).
    lo_processor->process( ls_delivery ).

  CATCH zcx_delivery_not_found INTO DATA(lo_not_found).
    " Most specific exception first:
    lo_logger->warning( lo_not_found->get_text( ) ).

  CATCH zcx_delivery_error INTO DATA(lo_dlv_ex).
    " Parent catches remaining ZCX_DELIVERY_ERROR subtypes:
    DATA(lv_msg) = lo_dlv_ex->get_text( ).
    lo_logger->error( lv_msg ).
    " Re-raise if caller must know:
    RAISE EXCEPTION lo_dlv_ex.

  CATCH cx_static_check INTO DATA(lo_gen_ex).
    " Catch-all for any other static check exception:
    lo_logger->error( lo_gen_ex->get_text( ) ).

  CLEANUP.
    " Always runs after an exception (even if re-raised):
    " Use for resource cleanup (file handles, locks):
    lo_lock->release( ).

ENDTRY.
```

## Resumable Exception

```abap
" In raising method:
RAISE RESUMABLE EXCEPTION TYPE zcx_delivery_warning
  EXPORTING iv_msg = 'Quantity exceeds planned — continue?'.

" In caller:
TRY.
    lo_processor->process_with_warnings( ls_delivery ).
  CATCH BEFORE UNWIND zcx_delivery_warning INTO DATA(lo_warn).
    " BEFORE UNWIND: stack not yet unwound — can RESUME
    IF lv_allow_overdelivery = abap_true.
      RESUME.   " continue execution after RAISE RESUMABLE
    ENDIF.
    " If no RESUME — exception propagates normally
ENDTRY.
```

## Exception Text Methods

```abap
" Get displayable text from exception:
DATA(lv_text) = lo_exception->get_text( ).

" Get structured message (for ABAP message display):
lo_exception->if_message~get_longtext( ).

" Use in MESSAGE statement context:
MESSAGE lo_exception->get_text( ) TYPE 'E'.

" Display in Fiori / OData error response:
RAISE EXCEPTION TYPE /iwbep/cx_mgw_busi_exception
  MESSAGE id     sy-msgid
          type   sy-msgty
          number sy-msgno
          with   sy-msgv1 sy-msgv2 sy-msgv3 sy-msgv4.
```

## Exception in Method Signature

```abap
" RAISING declares what can propagate to caller:
METHODS process_delivery
  IMPORTING iv_delivery_no TYPE /scwm/de_lgnum
  RAISING
    zcx_delivery_not_found    " caller must handle or re-declare
    zcx_delivery_error.

" cx_dynamic_check subtypes: no RAISING needed (unchecked)
" cx_no_check subtypes:      cannot be listed in RAISING
```

## Nested Exception with Cause Chain

```abap
" Access full cause chain:
DATA lo_ex TYPE REF TO cx_root.
lo_ex = lo_caught_exception.

WHILE lo_ex IS BOUND.
  lo_logger->error( lo_ex->get_text( ) ).
  lo_ex = lo_ex->previous.   " walk the chain
ENDWHILE.
```

## Application Log (BAL) for Exception Logging

```abap
" Log exception with context using BAL:
DATA(lo_log) = NEW cl_bali_log( ).
lo_log->add_item(
  item = cl_bali_exception_setter=>create(
    severity  = if_bali_constants=>c_severity_error
    exception = lo_exception
  )
).
cl_bali_log_db=>save_log( log = lo_log ).
```
