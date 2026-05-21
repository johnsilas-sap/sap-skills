# RAP Behavior Implementation Reference

## Class Structure

```abap
" ADT generates skeleton from BDEF — two inner classes per entity:
" lhc_<Alias>  = local handler class (for managed: create/update/delete handled automatically)
" lsc_<Root>   = local saver class   (for unmanaged: implement save sequence here)

CLASS zbp_delivery DEFINITION PUBLIC ABSTRACT FINAL
  FOR BEHAVIOR OF zi_delivery.
ENDCLASS.

CLASS zbp_delivery IMPLEMENTATION.
ENDCLASS.
```

## Handler Class — Managed Provider

```abap
" Local handler in the global class's CCIMP include (ADT: Local Types tab)
CLASS lhc_delivery DEFINITION INHERITING FROM cl_abap_behavior_handler.
  PRIVATE SECTION.
    METHODS:
      " Actions
      post_goods_issue      FOR ACTION Delivery~PostGoodsIssue,
      confirm_delivery      FOR ACTION Delivery~ConfirmDelivery,
      cancel_delivery       FOR ACTION Delivery~CancelDelivery,

      " Validations
      validate_status       FOR VALIDATE ON SAVE Delivery~ValidateStatus,
      validate_ship_date    FOR VALIDATE ON SAVE Delivery~ValidateShipDate,

      " Determinations
      set_creation_defaults FOR DETERMINE ON MODIFY Delivery~SetCreationDefaults,
      recalc_total_weight   FOR DETERMINE ON MODIFY DeliveryItem~RecalcTotalWeight,

      " Feature control
      get_instance_features FOR INSTANCE FEATURES OF Delivery.
ENDCLASS.

CLASS lhc_delivery IMPLEMENTATION.

  " ─── Action: Post Goods Issue ─────────────────────────────────────────────
  METHOD post_goods_issue.
    " keys[] contains the instances the action was called on
    READ ENTITIES OF zi_delivery IN LOCAL MODE
      ENTITY Delivery
        FIELDS ( DeliveryNo Status ShipDate )
        WITH CORRESPONDING #( keys )
      RESULT DATA(lt_deliveries)
      FAILED failed.

    LOOP AT lt_deliveries ASSIGNING FIELD-SYMBOL(<dlv>).
      " Guard: already processed
      IF <dlv>-Status = 'C'.
        APPEND VALUE #( %tky = <dlv>-%tky ) TO failed-delivery.
        APPEND VALUE #(
          %tky        = <dlv>-%tky
          %msg        = new_message_with_text( severity = if_abap_behv_message=>severity-error
                                               text     = 'Delivery already completed' )
        ) TO reported-delivery.
        CONTINUE.
      ENDIF.

      " Call EWM GI posting
      CALL FUNCTION 'ZEWM_POST_GOODS_ISSUE'
        EXPORTING iv_delivery_no = <dlv>-DeliveryNo
        EXCEPTIONS OTHERS = 1.

      IF sy-subrc <> 0.
        APPEND VALUE #( %tky = <dlv>-%tky ) TO failed-delivery.
        APPEND VALUE #(
          %tky  = <dlv>-%tky
          %msg  = new_message( id = sy-msgid type = sy-msgty number = sy-msgno
                               v1 = sy-msgv1 v2 = sy-msgv2 v3 = sy-msgv3 v4 = sy-msgv4 )
        ) TO reported-delivery.
        CONTINUE.
      ENDIF.

      " Update status to completed
      MODIFY ENTITIES OF zi_delivery IN LOCAL MODE
        ENTITY Delivery
          UPDATE FIELDS ( Status )
          WITH VALUE #( ( %tky   = <dlv>-%tky
                          Status = 'C' ) ).
    ENDLOOP.

    " Return updated entities as action result
    READ ENTITIES OF zi_delivery IN LOCAL MODE
      ENTITY Delivery ALL FIELDS
      WITH CORRESPONDING #( keys )
      RESULT DATA(lt_result).

    result = VALUE #( FOR ls IN lt_result ( %tky = ls-%tky %param = ls ) ).
  ENDMETHOD.

  " ─── Validation ────────────────────────────────────────────────────────────
  METHOD validate_status.
    READ ENTITIES OF zi_delivery IN LOCAL MODE
      ENTITY Delivery
        FIELDS ( Status )
        WITH CORRESPONDING #( keys )
      RESULT DATA(lt_deliveries).

    LOOP AT lt_deliveries ASSIGNING FIELD-SYMBOL(<dlv>).
      IF <dlv>-Status NOT IN VALUE #( ( sign = 'I' option = 'EQ' low = 'O' )
                                       ( sign = 'I' option = 'EQ' low = 'P' )
                                       ( sign = 'I' option = 'EQ' low = 'C' ) ).
        APPEND VALUE #( %tky = <dlv>-%tky ) TO failed-delivery.
        APPEND VALUE #(
          %tky        = <dlv>-%tky
          %element-Status = if_abap_behv=>mk-on
          %msg        = new_message_with_text( severity = if_abap_behv_message=>severity-error
                                               text     = |Invalid status: { <dlv>-Status }| )
        ) TO reported-delivery.
      ENDIF.
    ENDLOOP.
  ENDMETHOD.

  " ─── Determination ─────────────────────────────────────────────────────────
  METHOD set_creation_defaults.
    READ ENTITIES OF zi_delivery IN LOCAL MODE
      ENTITY Delivery
        FIELDS ( Status CreatedBy CreatedAt )
        WITH CORRESPONDING #( keys )
      RESULT DATA(lt_deliveries).

    MODIFY ENTITIES OF zi_delivery IN LOCAL MODE
      ENTITY Delivery
        UPDATE FIELDS ( Status CreatedBy CreatedAt )
        WITH VALUE #(
          FOR <dlv> IN lt_deliveries
          WHERE ( Status IS INITIAL )
          ( %tky      = <dlv>-%tky
            Status    = 'O'
            CreatedBy = sy-uname
            CreatedAt = utclong_current( ) )
        ).
  ENDMETHOD.

  " ─── Feature Control (dynamic) ─────────────────────────────────────────────
  METHOD get_instance_features.
    READ ENTITIES OF zi_delivery IN LOCAL MODE
      ENTITY Delivery
        FIELDS ( Status )
        WITH CORRESPONDING #( keys )
      RESULT DATA(lt_deliveries)
      FAILED failed.

    result = VALUE #(
      FOR <dlv> IN lt_deliveries
      LET is_open = COND #( WHEN <dlv>-Status = 'O' THEN if_abap_behv=>fc-o-enabled
                                                     ELSE if_abap_behv=>fc-o-disabled )
      IN ( %tky                    = <dlv>-%tky
           %action-PostGoodsIssue  = is_open
           %action-CancelDelivery  = is_open )
    ).
  ENDMETHOD.

ENDCLASS.
```

## Saver Class — Unmanaged Provider

```abap
CLASS lsc_zi_delivery DEFINITION INHERITING FROM cl_abap_behavior_saver.
  PROTECTED SECTION.
    METHODS:
      finalize          REDEFINITION,
      check_before_save REDEFINITION,
      save              REDEFINITION,
      cleanup           REDEFINITION.
ENDCLASS.

CLASS lsc_zi_delivery IMPLEMENTATION.

  METHOD save.
    " called by RAP framework after all validations pass
    " use MODIFY from the passed change data
    DATA(lo_modification) = cl_abap_tx_manager=>get_modification( ).

    " Access create/update/delete data via %create/%update/%delete tables
    " These come from the framework — see cl_abap_behv_handler
    LOOP AT lo_modification->delivery-create ASSIGNING FIELD-SYMBOL(<create>).
      INSERT zewm_delivery FROM @( CORRESPONDING #( <create> ) ).
    ENDLOOP.

    LOOP AT lo_modification->delivery-update ASSIGNING FIELD-SYMBOL(<update>).
      UPDATE zewm_delivery FROM @( CORRESPONDING #( <update> ) ).
    ENDLOOP.

    LOOP AT lo_modification->delivery-delete ASSIGNING FIELD-SYMBOL(<delete>).
      DELETE zewm_delivery WHERE delivery_no = <delete>-DeliveryNo.
    ENDLOOP.
  ENDMETHOD.

  METHOD check_before_save.
    " Last chance validation before COMMIT WORK
  ENDMETHOD.

  METHOD finalize.
    " Set final field values (e.g., compute totals) before save
  ENDMETHOD.

  METHOD cleanup.
    " Called on ROLLBACK — release any locked resources
  ENDMETHOD.

ENDCLASS.
```

## EML — Entity Manipulation Language

```abap
" EML is used to call RAP operations from ABAP (tests, determinations, etc.)

" READ single
READ ENTITIES OF zi_delivery
  ENTITY Delivery
    FIELDS ( DeliveryNo Status ShipToName )
    WITH VALUE #( ( DeliveryNo = '0180000001' ) )
  RESULT DATA(lt_result)
  FAILED DATA(ls_failed)
  REPORTED DATA(ls_reported).

" READ with expand
READ ENTITIES OF zi_delivery
  ENTITY Delivery BY \_Items
    ALL FIELDS
    WITH CORRESPONDING #( lt_deliveries )
  RESULT DATA(lt_items).

" MODIFY — create
MODIFY ENTITIES OF zi_delivery
  ENTITY Delivery
    CREATE FIELDS ( ShipToName ShipDate )
    WITH VALUE #( ( %cid      = 'cid1'
                    ShipToName = 'ACME Corp'
                    ShipDate   = '20260601' ) )
  MAPPED   DATA(ls_mapped)
  FAILED   DATA(ls_failed2)
  REPORTED DATA(ls_reported2).

" %cid is used to reference newly created entity in same EML block
" MODIFY ENTITIES ... ENTITY DeliveryItem CREATE ... WITH VALUE #( ( %cid_ref = 'cid1' ... ) )

" MODIFY — update
MODIFY ENTITIES OF zi_delivery
  ENTITY Delivery
    UPDATE FIELDS ( Status )
    WITH VALUE #( ( DeliveryNo = '0180000001'
                    Status     = 'P' ) ).

" MODIFY — delete
MODIFY ENTITIES OF zi_delivery
  ENTITY Delivery
    DELETE FROM VALUE #( ( DeliveryNo = '0180000001' ) ).

" Execute action via EML
MODIFY ENTITIES OF zi_delivery
  ENTITY Delivery
    EXECUTE PostGoodsIssue
    FROM VALUE #( ( DeliveryNo = '0180000001' ) )
  RESULT DATA(lt_action_result)
  FAILED DATA(ls_failed3).

" COMMIT (required in unmanaged + tests)
COMMIT ENTITIES
  RESPONSE OF zi_delivery
  FAILED   DATA(ls_commit_failed)
  REPORTED DATA(ls_commit_reported).
```

## Reporting Messages

```abap
" Structured message appended to reported table
APPEND VALUE #(
  %tky        = <dlv>-%tky
  %element-Status = if_abap_behv=>mk-on   " highlights field in UI
  %msg        = new_message_with_text(
                  severity = if_abap_behv_message=>severity-error
                  text     = 'Status is invalid' )
) TO reported-delivery.

" Using T100 message (recommended for translatable texts)
APPEND VALUE #(
  %tky = <dlv>-%tky
  %msg = new_message( id     = 'ZEWM_MSG'
                      number = '001'
                      type   = 'E'
                      v1     = <dlv>-DeliveryNo )
) TO reported-delivery.
```
