# ABAP OOP Design Patterns Reference

## Factory Pattern

```abap
" Centralizes object creation — swap implementation without changing callers.
" Enables test injection and environment-based variants.

INTERFACE zif_gi_poster.
  METHODS post IMPORTING iv_delivery TYPE /scwm/de_lgnum RAISING zcx_delivery_error.
ENDINTERFACE.

CLASS zcl_gi_poster_real DEFINITION PUBLIC FINAL IMPLEMENTING zif_gi_poster.
  PUBLIC SECTION. METHODS zif_gi_poster~post REDEFINITION.
ENDCLASS.
CLASS zcl_gi_poster_real IMPLEMENTATION.
  METHOD zif_gi_poster~post.
    CALL FUNCTION 'ZEWM_POST_GOODS_ISSUE' EXPORTING iv_delivery = iv_delivery.
  ENDMETHOD.
ENDCLASS.

" Factory class:
CLASS zcl_gi_poster_factory DEFINITION PUBLIC FINAL CREATE PRIVATE.
  PUBLIC SECTION.
    CLASS-METHODS:
      create RETURNING VALUE(ro_poster) TYPE REF TO zif_gi_poster,
      " Test injection point:
      set_mock IMPORTING io_mock TYPE REF TO zif_gi_poster.
  PRIVATE SECTION.
    CLASS-DATA go_mock TYPE REF TO zif_gi_poster.
ENDCLASS.

CLASS zcl_gi_poster_factory IMPLEMENTATION.
  METHOD create.
    ro_poster = COALESCE( go_mock, CAST zif_gi_poster( NEW zcl_gi_poster_real( ) ) ).
  ENDMETHOD.
  METHOD set_mock.
    go_mock = io_mock.
  ENDMETHOD.
ENDCLASS.

" Usage:
DATA(lo_poster) = zcl_gi_poster_factory=>create( ).
lo_poster->post( '0180000001' ).

" In tests:
zcl_gi_poster_factory=>set_mock( lo_mock_poster ).
```

## Singleton Pattern

```abap
" Only one instance ever exists — for configuration, caches, shared state.
CLASS zcl_warehouse_config DEFINITION PUBLIC FINAL CREATE PRIVATE.
  PUBLIC SECTION.
    CLASS-METHODS get_instance
      RETURNING VALUE(ro_config) TYPE REF TO zcl_warehouse_config.
    METHODS get_lgnum RETURNING VALUE(rv_lgnum) TYPE /scwm/lgnum.
    METHODS set_lgnum IMPORTING iv_lgnum TYPE /scwm/lgnum.
  PRIVATE SECTION.
    CLASS-DATA go_instance TYPE REF TO zcl_warehouse_config.
    DATA mv_lgnum TYPE /scwm/lgnum.
ENDCLASS.

CLASS zcl_warehouse_config IMPLEMENTATION.
  METHOD get_instance.
    IF go_instance IS NOT BOUND.
      go_instance = NEW zcl_warehouse_config( ).
    ENDIF.
    ro_config = go_instance.
  ENDMETHOD.
  METHOD get_lgnum. rv_lgnum = mv_lgnum. ENDMETHOD.
  METHOD set_lgnum. mv_lgnum = iv_lgnum. ENDMETHOD.
ENDCLASS.

" Usage (always same instance):
zcl_warehouse_config=>get_instance( )->set_lgnum( '1710' ).
DATA(lv_wh) = zcl_warehouse_config=>get_instance( )->get_lgnum( ).
```

## Strategy Pattern

```abap
" Pluggable algorithm — select at runtime without IF/CASE chains.
INTERFACE zif_picking_strategy.
  METHODS pick
    IMPORTING it_bins TYPE zewm_bin_t
    RETURNING VALUE(rs_selected_bin) TYPE zewm_bin.
ENDINTERFACE.

CLASS zcl_fifo_strategy DEFINITION PUBLIC FINAL IMPLEMENTING zif_picking_strategy.
  PUBLIC SECTION. METHODS zif_picking_strategy~pick REDEFINITION.
ENDCLASS.
CLASS zcl_fifo_strategy IMPLEMENTATION.
  METHOD zif_picking_strategy~pick.
    " Return oldest bin (first in, first out)
    rs_selected_bin = it_bins[ 1 ].
  ENDMETHOD.
ENDCLASS.

CLASS zcl_nearest_bin_strategy DEFINITION PUBLIC FINAL IMPLEMENTING zif_picking_strategy.
  PUBLIC SECTION. METHODS zif_picking_strategy~pick REDEFINITION.
ENDCLASS.
CLASS zcl_nearest_bin_strategy IMPLEMENTATION.
  METHOD zif_picking_strategy~pick.
    " Return bin with shortest travel path
    rs_selected_bin = VALUE #( it_bins[ 1 ] OPTIONAL ).  " simplified
  ENDMETHOD.
ENDCLASS.

" Context class uses strategy:
CLASS zcl_picking_engine DEFINITION PUBLIC CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS constructor IMPORTING io_strategy TYPE REF TO zif_picking_strategy.
    METHODS execute_pick IMPORTING it_bins TYPE zewm_bin_t
                         RETURNING VALUE(rs_bin) TYPE zewm_bin.
  PRIVATE SECTION.
    DATA mo_strategy TYPE REF TO zif_picking_strategy.
ENDCLASS.

CLASS zcl_picking_engine IMPLEMENTATION.
  METHOD constructor. mo_strategy = io_strategy. ENDMETHOD.
  METHOD execute_pick.
    rs_bin = mo_strategy->pick( it_bins ).
  ENDMETHOD.
ENDCLASS.

" Caller selects strategy at runtime:
DATA(lv_strategy_type) = zcl_warehouse_config=>get_instance( )->get_picking_strategy( ).
DATA lo_strategy TYPE REF TO zif_picking_strategy.
lo_strategy = SWITCH #( lv_strategy_type
  WHEN 'FIFO'    THEN NEW zcl_fifo_strategy( )
  WHEN 'NEAREST' THEN NEW zcl_nearest_bin_strategy( )
  ELSE NEW zcl_fifo_strategy( )
).
DATA(lo_engine) = NEW zcl_picking_engine( lo_strategy ).
```

## Observer Pattern

```abap
" Decouple event source from subscribers.
INTERFACE zif_delivery_observer.
  METHODS on_delivery_posted
    IMPORTING iv_delivery_no TYPE /scwm/de_lgnum
              iv_new_status  TYPE char1.
ENDINTERFACE.

CLASS zcl_delivery_event_bus DEFINITION PUBLIC FINAL CREATE PRIVATE.
  PUBLIC SECTION.
    CLASS-METHODS get_instance RETURNING VALUE(ro_bus) TYPE REF TO zcl_delivery_event_bus.
    METHODS subscribe   IMPORTING io_observer TYPE REF TO zif_delivery_observer.
    METHODS unsubscribe IMPORTING io_observer TYPE REF TO zif_delivery_observer.
    METHODS publish
      IMPORTING iv_delivery_no TYPE /scwm/de_lgnum
                iv_new_status  TYPE char1.
  PRIVATE SECTION.
    CLASS-DATA go_instance TYPE REF TO zcl_delivery_event_bus.
    DATA mt_observers TYPE TABLE OF REF TO zif_delivery_observer.
ENDCLASS.

CLASS zcl_delivery_event_bus IMPLEMENTATION.
  METHOD get_instance.
    IF go_instance IS NOT BOUND. go_instance = NEW #( ). ENDIF.
    ro_bus = go_instance.
  ENDMETHOD.
  METHOD subscribe.
    APPEND io_observer TO mt_observers.
  ENDMETHOD.
  METHOD unsubscribe.
    DELETE mt_observers WHERE table_line = io_observer.
  ENDMETHOD.
  METHOD publish.
    LOOP AT mt_observers ASSIGNING FIELD-SYMBOL(<obs>).
      <obs>->on_delivery_posted( iv_delivery_no = iv_delivery_no iv_new_status = iv_new_status ).
    ENDLOOP.
  ENDMETHOD.
ENDCLASS.

" Subscriber:
CLASS zcl_gi_notification DEFINITION PUBLIC IMPLEMENTING zif_delivery_observer.
  PUBLIC SECTION. METHODS zif_delivery_observer~on_delivery_posted REDEFINITION.
ENDCLASS.
CLASS zcl_gi_notification IMPLEMENTATION.
  METHOD zif_delivery_observer~on_delivery_posted.
    " Send email, update dashboard, etc.
  ENDMETHOD.
ENDCLASS.

" Wire up:
zcl_delivery_event_bus=>get_instance( )->subscribe( NEW zcl_gi_notification( ) ).
" Later: zcl_delivery_event_bus=>get_instance( )->publish( '001' 'C' ).
```

## Template Method Pattern

```abap
" Define skeleton in abstract base — subclasses fill in the steps.
CLASS zcl_document_workflow DEFINITION PUBLIC ABSTRACT CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS run_workflow IMPORTING iv_doc_no TYPE char20.   " template method
  PROTECTED SECTION.
    " Steps — subclasses override as needed:
    METHODS validate   IMPORTING iv_doc_no TYPE char20.
    METHODS enrich     ABSTRACT IMPORTING iv_doc_no TYPE char20.
    METHODS post       ABSTRACT IMPORTING iv_doc_no TYPE char20.
    METHODS notify     IMPORTING iv_doc_no TYPE char20.     " default: no-op
ENDCLASS.

CLASS zcl_document_workflow IMPLEMENTATION.
  METHOD run_workflow.
    validate( iv_doc_no ).    " common
    enrich(   iv_doc_no ).    " abstract — subclass provides
    post(     iv_doc_no ).    " abstract
    notify(   iv_doc_no ).    " default: no-op; override to notify
  ENDMETHOD.
  METHOD validate.
    IF iv_doc_no IS INITIAL. RAISE EXCEPTION TYPE zcx_processing_error. ENDIF.
  ENDMETHOD.
  METHOD notify. " default: do nothing ENDMETHOD.
ENDCLASS.

" Subclass provides the steps:
CLASS zcl_delivery_workflow DEFINITION PUBLIC FINAL
  INHERITING FROM zcl_document_workflow CREATE PUBLIC.
  PROTECTED SECTION.
    METHODS enrich  REDEFINITION.
    METHODS post    REDEFINITION.
    METHODS notify  REDEFINITION.   " override default no-op
ENDCLASS.
CLASS zcl_delivery_workflow IMPLEMENTATION.
  METHOD enrich.  " ... load carrier data ...  ENDMETHOD.
  METHOD post.    " ... call EWM GI FM ...     ENDMETHOD.
  METHOD notify.  " ... send email ...          ENDMETHOD.
ENDCLASS.

" Usage: polymorphic call, specific steps execute:
DATA(lo_wf) TYPE REF TO zcl_document_workflow.
lo_wf = NEW zcl_delivery_workflow( ).
lo_wf->run_workflow( '0180000001' ).
```

## Repository Pattern

```abap
" Isolates DB access — keeps business logic free of SELECT statements.
INTERFACE zif_delivery_repository.
  METHODS get_delivery
    IMPORTING iv_delivery_no TYPE /scwm/de_lgnum
    RETURNING VALUE(rs_delivery) TYPE zewm_delivery
    RAISING   zcx_delivery_not_found.
  METHODS save_delivery IMPORTING is_delivery TYPE zewm_delivery.
  METHODS get_open_deliveries
    IMPORTING iv_lgnum TYPE /scwm/lgnum
    RETURNING VALUE(rt_deliveries) TYPE zewm_delivery_t.
ENDINTERFACE.

CLASS zcl_delivery_db_repo DEFINITION PUBLIC FINAL IMPLEMENTING zif_delivery_repository.
  PUBLIC SECTION.
    METHODS zif_delivery_repository~get_delivery      REDEFINITION.
    METHODS zif_delivery_repository~save_delivery     REDEFINITION.
    METHODS zif_delivery_repository~get_open_deliveries REDEFINITION.
ENDCLASS.

CLASS zcl_delivery_db_repo IMPLEMENTATION.
  METHOD zif_delivery_repository~get_delivery.
    SELECT SINGLE * FROM zewm_delivery
      WHERE delivery_no = @iv_delivery_no
      INTO @rs_delivery.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE zcx_delivery_not_found
        EXPORTING iv_delivery_no = iv_delivery_no.
    ENDIF.
  ENDMETHOD.

  METHOD zif_delivery_repository~save_delivery.
    UPDATE zewm_delivery FROM @is_delivery.
    IF sy-subrc <> 0.
      INSERT zewm_delivery FROM @is_delivery.
    ENDIF.
  ENDMETHOD.

  METHOD zif_delivery_repository~get_open_deliveries.
    SELECT * FROM zewm_delivery
      WHERE status = 'O' AND lgnum = @iv_lgnum
      INTO TABLE @rt_deliveries.
  ENDMETHOD.
ENDCLASS.
```
