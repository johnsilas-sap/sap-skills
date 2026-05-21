# EWM and TM OOP Patterns Reference

## Typical EWM Class Hierarchy

```
ZIF_EWM_SERVICE               — common service interface (init, validate, execute)
  ZIF_EWM_DELIVERY_SERVICE    — delivery-specific operations
  ZIF_EWM_WHO_SERVICE         — warehouse order operations
  ZIF_EWM_HU_SERVICE          — handling unit operations

ZCL_EWM_DELIVERY_SERVICE      IMPLEMENTING ZIF_EWM_DELIVERY_SERVICE
ZCL_EWM_WHO_SERVICE            IMPLEMENTING ZIF_EWM_WHO_SERVICE

ZIF_EWM_REPOSITORY            — data access interface
  ZIF_EWM_DELIVERY_REPO       — delivery data
  ZIF_EWM_WHO_REPO            — warehouse order data

ZCL_EWM_DELIVERY_DB_REPO      IMPLEMENTING ZIF_EWM_DELIVERY_REPO
ZCL_EWM_DELIVERY_MOCK_REPO    IMPLEMENTING ZIF_EWM_DELIVERY_REPO  " test only
```

## EWM Delivery Service Class

```abap
INTERFACE zif_ewm_delivery_service.
  METHODS:
    get_open_deliveries
      IMPORTING iv_lgnum TYPE /scwm/lgnum
      RETURNING VALUE(rt_deliveries) TYPE /scwm/tt_dlv_header_str
      RAISING   zcx_ewm_error,
    post_goods_issue
      IMPORTING iv_lgnum      TYPE /scwm/lgnum
                iv_deliv_numb TYPE /scwm/de_lgnum
      RAISING   zcx_ewm_error,
    confirm_delivery
      IMPORTING is_confirm TYPE zewm_confirm_param
      RAISING   zcx_ewm_error.
ENDINTERFACE.

CLASS zcl_ewm_delivery_service DEFINITION PUBLIC FINAL CREATE PRIVATE.
  PUBLIC SECTION.
    INTERFACES zif_ewm_delivery_service.

    CLASS-METHODS get_instance
      IMPORTING io_repo TYPE REF TO zif_ewm_delivery_repo OPTIONAL
      RETURNING VALUE(ro_service) TYPE REF TO zif_ewm_delivery_service.
  PRIVATE SECTION.
    CLASS-DATA go_instance TYPE REF TO zcl_ewm_delivery_service.
    DATA mo_repo TYPE REF TO zif_ewm_delivery_repo.

    METHODS constructor IMPORTING io_repo TYPE REF TO zif_ewm_delivery_repo.
    METHODS validate_gi_preconditions
      IMPORTING is_delivery TYPE /scwm/s_dlv_header_str
      RAISING   zcx_ewm_error.
ENDCLASS.

CLASS zcl_ewm_delivery_service IMPLEMENTATION.

  METHOD get_instance.
    IF go_instance IS NOT BOUND.
      go_instance = NEW zcl_ewm_delivery_service(
        COALESCE( io_repo, CAST zif_ewm_delivery_repo( NEW zcl_ewm_delivery_db_repo( ) ) )
      ).
    ENDIF.
    ro_service = go_instance.
  ENDMETHOD.

  METHOD constructor. mo_repo = io_repo. ENDMETHOD.

  METHOD zif_ewm_delivery_service~get_open_deliveries.
    rt_deliveries = mo_repo->get_deliveries_by_status( iv_lgnum = iv_lgnum iv_status = 'A' ).
  ENDMETHOD.

  METHOD zif_ewm_delivery_service~post_goods_issue.
    DATA(ls_delivery) = mo_repo->get_delivery( iv_lgnum = iv_lgnum iv_deliv_numb = iv_deliv_numb ).
    validate_gi_preconditions( ls_delivery ).

    CALL FUNCTION '/SCWM/GI_POST'
      EXPORTING iv_lgnum    = iv_lgnum
                iv_deliv    = iv_deliv_numb
      EXCEPTIONS not_found  = 1  OTHERS = 2.

    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE zcx_ewm_error
        EXPORTING textid = zcx_ewm_error=>gc_msg-gi_post_failed
                  iv_deliv_numb = iv_deliv_numb.
    ENDIF.
  ENDMETHOD.

  METHOD validate_gi_preconditions.
    IF is_delivery-stat_ogi = 'C'.
      RAISE EXCEPTION TYPE zcx_ewm_error
        EXPORTING textid = zcx_ewm_error=>gc_msg-already_posted.
    ENDIF.
  ENDMETHOD.

ENDCLASS.
```

## EWM Repository Pattern

```abap
INTERFACE zif_ewm_delivery_repo.
  METHODS:
    get_delivery
      IMPORTING iv_lgnum      TYPE /scwm/lgnum
                iv_deliv_numb TYPE /scwm/de_lgnum
      RETURNING VALUE(rs_delivery) TYPE /scwm/s_dlv_header_str
      RAISING   zcx_ewm_not_found,
    get_deliveries_by_status
      IMPORTING iv_lgnum  TYPE /scwm/lgnum
                iv_status TYPE char1
      RETURNING VALUE(rt_deliveries) TYPE /scwm/tt_dlv_header_str,
    save_delivery
      IMPORTING is_delivery TYPE /scwm/s_dlv_header_str.
ENDINTERFACE.

CLASS zcl_ewm_delivery_db_repo DEFINITION PUBLIC FINAL IMPLEMENTING zif_ewm_delivery_repo.
  PUBLIC SECTION.
    METHODS:
      zif_ewm_delivery_repo~get_delivery          REDEFINITION,
      zif_ewm_delivery_repo~get_deliveries_by_status REDEFINITION,
      zif_ewm_delivery_repo~save_delivery         REDEFINITION.
ENDCLASS.

CLASS zcl_ewm_delivery_db_repo IMPLEMENTATION.
  METHOD zif_ewm_delivery_repo~get_delivery.
    " Call standard EWM API to read delivery:
    CALL FUNCTION '/SCWM/DLV_MANAGEMENT_READ'
      EXPORTING iv_lgnum    = iv_lgnum
                iv_deliv_no = iv_deliv_numb
      IMPORTING es_delivery = rs_delivery
      EXCEPTIONS not_found  = 1.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE zcx_ewm_not_found
        EXPORTING iv_deliv_numb = iv_deliv_numb.
    ENDIF.
  ENDMETHOD.

  METHOD zif_ewm_delivery_repo~get_deliveries_by_status.
    SELECT * FROM /scwm/d_dlvd
      WHERE lgnum = @iv_lgnum AND stat_over = @iv_status
      INTO TABLE @rt_deliveries.
  ENDMETHOD.

  METHOD zif_ewm_delivery_repo~save_delivery.
    MODIFY /scwm/d_dlvd FROM @( CORRESPONDING #( is_delivery ) ).
  ENDMETHOD.
ENDCLASS.
```

## EWM Manager Pattern (Orchestrator)

```abap
" Manager coordinates multiple services for complex operations
CLASS zcl_ewm_inbound_manager DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS constructor
      IMPORTING io_delivery_svc TYPE REF TO zif_ewm_delivery_service OPTIONAL
                io_hu_svc       TYPE REF TO zif_ewm_hu_service        OPTIONAL
                io_who_svc      TYPE REF TO zif_ewm_who_service       OPTIONAL.
    METHODS process_goods_receipt
      IMPORTING is_gr_data TYPE zewm_gr_param
      RAISING   zcx_ewm_error.
  PRIVATE SECTION.
    DATA mo_delivery_svc TYPE REF TO zif_ewm_delivery_service.
    DATA mo_hu_svc       TYPE REF TO zif_ewm_hu_service.
    DATA mo_who_svc      TYPE REF TO zif_ewm_who_service.
ENDCLASS.

CLASS zcl_ewm_inbound_manager IMPLEMENTATION.
  METHOD constructor.
    mo_delivery_svc = COALESCE( io_delivery_svc,
      zcl_ewm_delivery_service=>get_instance( ) ).
    mo_hu_svc = COALESCE( io_hu_svc,
      zcl_ewm_hu_service=>get_instance( ) ).
    mo_who_svc = COALESCE( io_who_svc,
      zcl_ewm_who_service=>get_instance( ) ).
  ENDMETHOD.

  METHOD process_goods_receipt.
    " Orchestrate multi-step GR:
    " 1. Create inbound delivery
    DATA(lv_dlv_no) = mo_delivery_svc->create_inbound_delivery( is_gr_data ).
    " 2. Create HU if needed
    IF is_gr_data-pack_in_hu = abap_true.
      mo_hu_svc->create_hu_for_delivery( lv_dlv_no ).
    ENDIF.
    " 3. Create putaway warehouse order
    mo_who_svc->create_putaway_order( lv_dlv_no ).
    " 4. Post goods receipt
    mo_delivery_svc->post_goods_issue(
      iv_lgnum      = is_gr_data-lgnum
      iv_deliv_numb = lv_dlv_no
    ).
  ENDMETHOD.
ENDCLASS.
```

## TM Freight Order Service

```abap
INTERFACE zif_tm_fro_service.
  METHODS:
    get_freight_orders
      IMPORTING iv_lifecycle_status TYPE char2 DEFAULT 'OP'
      RETURNING VALUE(rt_fros) TYPE /scmtms/t_tor_k
      RAISING   zcx_tm_error,
    approve_freight_order
      IMPORTING iv_fro_id TYPE /scmtms/tor_id
      RAISING   zcx_tm_error,
    assign_carrier
      IMPORTING iv_fro_id    TYPE /scmtms/tor_id
                iv_carrier   TYPE bu_partner
      RAISING   zcx_tm_error.
ENDINTERFACE.

CLASS zcl_tm_fro_service DEFINITION PUBLIC FINAL CREATE PRIVATE
  IMPLEMENTING zif_tm_fro_service.
  PUBLIC SECTION.
    CLASS-METHODS get_instance
      RETURNING VALUE(ro_svc) TYPE REF TO zif_tm_fro_service.
    METHODS:
      zif_tm_fro_service~get_freight_orders  REDEFINITION,
      zif_tm_fro_service~approve_freight_order REDEFINITION,
      zif_tm_fro_service~assign_carrier      REDEFINITION.
  PRIVATE SECTION.
    CLASS-DATA go_instance TYPE REF TO zcl_tm_fro_service.
ENDCLASS.

CLASS zcl_tm_fro_service IMPLEMENTATION.
  METHOD get_instance.
    IF go_instance IS NOT BOUND. go_instance = NEW #( ). ENDIF.
    ro_svc = go_instance.
  ENDMETHOD.

  METHOD zif_tm_fro_service~approve_freight_order.
    CALL FUNCTION '/SCMTMS/FRO_SET_STATUS'
      EXPORTING iv_fro_id = iv_fro_id  iv_status = 'APR'
      EXCEPTIONS OTHERS = 1.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE zcx_tm_error
        EXPORTING textid = zcx_tm_error=>gc_msg-approve_failed.
    ENDIF.
  ENDMETHOD.

  METHOD zif_tm_fro_service~get_freight_orders.
    SELECT node_id FROM /scmtms/d_tor
      WHERE lifecycle_status = @iv_lifecycle_status
      INTO TABLE @DATA(lt_ids).
    " ... load full FRO data ...
  ENDMETHOD.

  METHOD zif_tm_fro_service~assign_carrier.
    " Update FRO carrier via BOPF or /SCMTMS/FRO_CHANGE
  ENDMETHOD.
ENDCLASS.
```

## Exception Classes for EWM/TM

```abap
" EWM exception hierarchy:
" ZCX_EWM_ERROR (cx_static_check)
"   ZCX_EWM_NOT_FOUND
"   ZCX_EWM_LOCKED
"   ZCX_EWM_INVALID_STATE
"   ZCX_EWM_POST_FAILED

CLASS zcx_ewm_error DEFINITION PUBLIC
  INHERITING FROM cx_static_check CREATE PUBLIC.
  PUBLIC SECTION.
    INTERFACES if_t100_message.
    DATA mv_deliv_numb TYPE /scwm/de_lgnum.
    DATA mv_lgnum      TYPE /scwm/lgnum.
    CONSTANTS:
      BEGIN OF gc_msg,
        gi_post_failed TYPE scx_t100key
          VALUE VALUE #( msgid = 'ZEWM' msgno = '001' attr1 = 'MV_DELIV_NUMB' ),
        already_posted TYPE scx_t100key
          VALUE VALUE #( msgid = 'ZEWM' msgno = '002' attr1 = 'MV_DELIV_NUMB' ),
        not_found TYPE scx_t100key
          VALUE VALUE #( msgid = 'ZEWM' msgno = '003' attr1 = 'MV_DELIV_NUMB' ),
      END OF gc_msg.
    METHODS constructor
      IMPORTING
        iv_deliv_numb TYPE /scwm/de_lgnum OPTIONAL
        iv_lgnum      TYPE /scwm/lgnum    OPTIONAL
        textid        LIKE if_t100_message=>t100key OPTIONAL
        previous      LIKE previous.
ENDCLASS.

CLASS zcx_ewm_error IMPLEMENTATION.
  METHOD constructor.
    super->constructor( textid = textid  previous = previous ).
    mv_deliv_numb = iv_deliv_numb.
    mv_lgnum      = iv_lgnum.
    IF textid IS INITIAL.
      if_t100_message~t100key = gc_msg-not_found.
    ENDIF.
  ENDMETHOD.
ENDCLASS.
```
