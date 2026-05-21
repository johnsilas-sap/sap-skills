# ABAP Classes and Interfaces Reference

## Class Definition Keywords

```abap
" Modifiers:
" PUBLIC        — visible in all programs
" LOCAL FRIENDS zcl_test_foo — test class can access private section
" FINAL         — cannot be subclassed
" ABSTRACT      — cannot be instantiated directly
" CREATE PUBLIC  — anyone can call NEW
" CREATE PROTECTED — only subclasses and FRIENDS can call NEW
" CREATE PRIVATE   — only class itself (Singleton pattern)

CLASS zcl_delivery_processor DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.
    " Interface implementation declaration
    INTERFACES zif_delivery_processor.

    " Alias — expose interface method directly (optional convenience)
    ALIASES process FOR zif_delivery_processor~process.

    " Static (class-level) members
    CLASS-METHODS:
      class_constructor,           " called once when class is first loaded
      get_instance RETURNING VALUE(ro_obj) TYPE REF TO zcl_delivery_processor.
    CLASS-DATA:
      gv_instance_count TYPE i READ-ONLY.

    " Instance members
    METHODS:
      constructor
        IMPORTING io_repo TYPE REF TO zif_delivery_repository OPTIONAL,
      get_status
        IMPORTING iv_delivery_no TYPE /scwm/de_lgnum
        RETURNING VALUE(rv_status) TYPE char1
        RAISING   zcx_delivery_error,
      set_warehouse
        IMPORTING iv_lgnum TYPE /scwm/lgnum
        RETURNING VALUE(ro_self) TYPE REF TO zcl_delivery_processor.  " fluent/chaining

  PROTECTED SECTION.
    " Accessible to subclasses
    DATA mv_lgnum TYPE /scwm/lgnum.
    METHODS validate_delivery
      IMPORTING is_delivery TYPE zewm_delivery
      RAISING   zcx_delivery_error.

  PRIVATE SECTION.
    " Only this class
    CLASS-DATA go_singleton TYPE REF TO zcl_delivery_processor.
    DATA mo_repo TYPE REF TO zif_delivery_repository.
    DATA mv_initialized TYPE abap_bool.

    METHODS build_filter
      IMPORTING iv_status TYPE char1
      RETURNING VALUE(ro_filter) TYPE REF TO cl_os_query_filter_factory.
ENDCLASS.
```

## Class Implementation

```abap
CLASS zcl_delivery_processor IMPLEMENTATION.

  " Class constructor — runs once, no IMPORTING
  METHOD class_constructor.
    gv_instance_count = 0.
  ENDMETHOD.

  " Instance constructor
  METHOD constructor.
    mo_repo = COALESCE( io_repo, CAST zif_delivery_repository(
      NEW zcl_delivery_db_repo( )
    ) ).
    ADD 1 TO gv_instance_count.
    mv_initialized = abap_true.
  ENDMETHOD.

  " Interface method — prefix with interface name~
  METHOD zif_delivery_processor~process.
    DATA(ls_delivery) = mo_repo->get_delivery( iv_delivery_no ).
    validate_delivery( ls_delivery ).
    " ... processing logic ...
    rv_new_status = 'C'.
  ENDMETHOD.

  " Fluent method (returns self reference)
  METHOD set_warehouse.
    mv_lgnum = iv_lgnum.
    ro_self = me.    " 'me' = current instance reference
  ENDMETHOD.

  METHOD validate_delivery.
    IF is_delivery-delivery_no IS INITIAL.
      RAISE EXCEPTION TYPE zcx_delivery_error
        EXPORTING iv_msg = 'Delivery number is empty'.
    ENDIF.
  ENDMETHOD.

ENDCLASS.
```

## Interface Definition

```abap
INTERFACE zif_delivery_processor.
  " Constants in interface:
  CONSTANTS:
    gc_status_open      TYPE char1 VALUE 'O',
    gc_status_completed TYPE char1 VALUE 'C'.

  " Methods:
  METHODS:
    process
      IMPORTING iv_delivery_no TYPE /scwm/de_lgnum
      RETURNING VALUE(rv_new_status) TYPE char1
      RAISING   zcx_delivery_error,
    get_open_deliveries
      IMPORTING iv_lgnum TYPE /scwm/lgnum
      RETURNING VALUE(rt_deliveries) TYPE zewm_delivery_t,
    is_initialized
      RETURNING VALUE(rv_result) TYPE abap_bool.
ENDINTERFACE.
```

## Implementing Multiple Interfaces

```abap
CLASS zcl_delivery_service DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    INTERFACES:
      zif_delivery_processor,
      zif_loggable,
      zif_lifecycle.   " multiple interfaces — all methods must be implemented

    " Disambiguate same method name from two interfaces:
    ALIASES log       FOR zif_loggable~write_log.
    ALIASES initialize FOR zif_lifecycle~init.
ENDCLASS.

CLASS zcl_delivery_service IMPLEMENTATION.
  METHOD zif_delivery_processor~process.     " ... ENDMETHOD.
  METHOD zif_loggable~write_log.             " ... ENDMETHOD.
  METHOD zif_lifecycle~init.                 " ... ENDMETHOD.
  METHOD zif_lifecycle~shutdown.             " ... ENDMETHOD.
ENDCLASS.
```

## Object Creation and Reference Handling

```abap
" NEW operator (ABAP 7.40+)
DATA(lo_proc) = NEW zcl_delivery_processor( io_repo = lo_repo ).

" Reference via interface
DATA lo_proc TYPE REF TO zif_delivery_processor.
lo_proc = NEW zcl_delivery_processor( ).

" Check if bound / initial
IF lo_proc IS BOUND.          " reference is not null
IF lo_proc IS NOT BOUND.      " null reference
IF lo_proc IS INITIAL.        " same as IS NOT BOUND for object refs

" Clear reference
CLEAR lo_proc.

" CAST — downcast from interface/parent to specific class
DATA lo_specific TYPE REF TO zcl_delivery_processor.
lo_specific = CAST zcl_delivery_processor( lo_proc ).   " raises cx_sy_move_cast_error if wrong type

" IS — type check before cast
IF lo_proc IS INSTANCE OF zcl_delivery_processor.
  lo_specific = CAST #( lo_proc ).
ENDIF.

" Safe cast with TRY:
TRY.
    lo_specific = CAST zcl_delivery_processor( lo_proc ).
  CATCH cx_sy_move_cast_error.
    " lo_proc is not a zcl_delivery_processor
ENDTRY.
```

## Static Members and Singleton

```abap
CLASS zcl_config DEFINITION PUBLIC CREATE PRIVATE FINAL.
  PUBLIC SECTION.
    CLASS-METHODS get_instance
      RETURNING VALUE(ro_config) TYPE REF TO zcl_config.
    METHODS get_warehouse RETURNING VALUE(rv_lgnum) TYPE /scwm/lgnum.
  PRIVATE SECTION.
    CLASS-DATA go_instance TYPE REF TO zcl_config.
    DATA mv_lgnum TYPE /scwm/lgnum.
ENDCLASS.

CLASS zcl_config IMPLEMENTATION.
  METHOD get_instance.
    IF go_instance IS NOT BOUND.
      go_instance = NEW zcl_config( ).
    ENDIF.
    ro_config = go_instance.
  ENDMETHOD.

  METHOD get_warehouse.
    rv_lgnum = mv_lgnum.
  ENDMETHOD.
ENDCLASS.

" Usage:
DATA(lv_wh) = zcl_config=>get_instance( )->get_warehouse( ).  " method chaining
```

## READ-ONLY Attributes

```abap
CLASS zcl_delivery DEFINITION PUBLIC.
  PUBLIC SECTION.
    DATA mv_delivery_no TYPE /scwm/de_lgnum READ-ONLY.   " external read, internal write
    DATA mv_status      TYPE char1          READ-ONLY.
ENDCLASS.
" Externally: lo_dlv->mv_delivery_no  (read only)
" Internally: mv_delivery_no = '001'. (OK inside class)
```
