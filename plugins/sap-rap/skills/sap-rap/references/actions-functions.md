# RAP Actions and Functions Reference

## Action vs Function

| Concept | Modifies Data | HTTP Method | BDEF keyword |
|---|---|---|---|
| Action | Yes | POST | `action` |
| Function | No (read-only) | GET | `function` |
| Factory Action | Yes (creates new instance) | POST | `action` (factory) |
| Static Action | Not instance-bound | POST | `static action` |

## Bound Action (instance-specific)

```abap
" BDEF:
action PostGoodsIssue result [1] $self;
" result [1] $self = returns the modified entity itself
" result [0..*] $self = returns 0 or more instances

" With parameter entity:
action ConfirmDelivery parameter ZA_ConfirmParam result [1] $self;

" With feature control (shown/hidden per instance state):
action ( features : instance ) CancelDelivery result [1] $self;
```

```abap
" Implementation:
METHOD post_goods_issue.
  " keys[] = instances selected by user
  READ ENTITIES OF zi_delivery IN LOCAL MODE
    ENTITY Delivery FIELDS ( DeliveryNo Status )
    WITH CORRESPONDING #( keys )
  RESULT DATA(lt_deliveries) FAILED failed.

  LOOP AT lt_deliveries ASSIGNING FIELD-SYMBOL(<dlv>).
    IF <dlv>-Status <> 'O'.
      APPEND VALUE #(
        %tky = <dlv>-%tky
        %msg = new_message_with_text( severity = 'E' text = 'Only open deliveries can be posted' )
      ) TO reported-delivery.
      APPEND VALUE #( %tky = <dlv>-%tky ) TO failed-delivery.
      CONTINUE.
    ENDIF.

    " Call backend logic
    CALL FUNCTION 'ZEWM_POST_GOODS_ISSUE'
      EXPORTING iv_delivery = <dlv>-DeliveryNo
      EXCEPTIONS OTHERS = 1.

    IF sy-subrc <> 0.
      APPEND VALUE #( %tky = <dlv>-%tky ) TO failed-delivery.
      CONTINUE.
    ENDIF.

    MODIFY ENTITIES OF zi_delivery IN LOCAL MODE
      ENTITY Delivery UPDATE FIELDS ( Status )
      WITH VALUE #( ( %tky = <dlv>-%tky  Status = 'C' ) ).
  ENDLOOP.

  " Return updated entities as action result
  READ ENTITIES OF zi_delivery IN LOCAL MODE
    ENTITY Delivery ALL FIELDS
    WITH CORRESPONDING #( keys )
  RESULT DATA(lt_result).
  result = VALUE #( FOR ls IN lt_result ( %tky = ls-%tky %param = ls ) ).
ENDMETHOD.
```

## Action with Parameter

```abap
" Abstract entity as parameter:
define abstract entity ZA_ConfirmParam
{
  PostingDate : abap.dats;
  ActualQty   : abap.quan(13,3);
  ActualUnit  : abap.unit(3);
}
```

```abap
" BDEF:
action ConfirmDelivery parameter ZA_ConfirmParam result [1] $self;

" Implementation — read parameter from keys table:
METHOD confirm_delivery.
  LOOP AT keys ASSIGNING FIELD-SYMBOL(<key>).
    DATA(ls_param)  = <key>-%param.   " ZA_ConfirmParam instance
    DATA(lv_date)   = ls_param-PostingDate.
    DATA(lv_qty)    = ls_param-ActualQty.
    DATA(lv_unit)   = ls_param-ActualUnit.

    " ... confirm delivery with provided data ...
  ENDLOOP.
ENDMETHOD.
```

## Static Action (Not Instance-Bound)

```abap
" BDEF — no instance required to call:
static action CreateFromPO parameter ZA_CreateFromPOParam result [1] $self;

" Called via: POST /Deliveries/ZI_Delivery.CreateFromPO
" Implementation:
METHOD create_from_po.
  DATA(ls_param) = keys[ 1 ]-%param.
  " ... create delivery from PO data ...
  " Append to mapped-delivery for new instance reference
  APPEND VALUE #( %cid = ls_param-%cid  DeliveryNo = lv_new_no ) TO mapped-delivery.
ENDMETHOD.
```

## Factory Action (Creates Child Instance)

```abap
" BDEF on root — factory action creates and returns a child:
action CreateReturnDelivery result [1] $self;   " creates a return delivery referencing original

" Or: factory action on item to create related item:
" factory action CopyItem result [1] DeliveryItem;
```

## Unbound Function (No Keys)

```abap
" BDEF:
static function GetOpenDeliveryCount result [1] ZA_CountResult;

" Called via: GET /Deliveries/ZI_Delivery.GetOpenDeliveryCount()
" Implementation:
METHOD get_open_delivery_count.
  SELECT COUNT(*) FROM zewm_delivery WHERE status = 'O' INTO @DATA(lv_count).
  result = VALUE #( ( %param-Count = lv_count ) ).
ENDMETHOD.
```

## Bound Function (Instance Read)

```abap
" BDEF:
function GetDeliveryStatus result [1] ZA_StatusResult;

" Called via: GET /Deliveries('0180000001')/ZI_Delivery.GetDeliveryStatus()
METHOD get_delivery_status.
  READ ENTITIES OF zi_delivery IN LOCAL MODE
    ENTITY Delivery FIELDS ( Status ShipDate )
    WITH CORRESPONDING #( keys )
  RESULT DATA(lt_deliveries).

  result = VALUE #(
    FOR <dlv> IN lt_deliveries
    ( %tky    = <dlv>-%tky
      %param  = VALUE #( Status   = <dlv>-Status
                         ShipDate = <dlv>-ShipDate ) )
  ).
ENDMETHOD.
```

## Calling Actions from Fiori / OData V4

```javascript
// Fiori Elements apps generate action buttons automatically from BDEF.
// For custom SAPUI5 controllers (OData V4 model):

// Bound action — needs binding context of entity instance
var oOperation = oModel.bindContext(
  "ZI_Delivery.PostGoodsIssue(...)",
  this.getView().getBindingContext()
);
oOperation.execute().then(function() {
  sap.m.MessageToast.show("Goods Issue posted.");
}).catch(function(oError) {
  sap.m.MessageBox.error(oError.message);
});

// Action with parameter
var oOperation = oModel.bindContext(
  "ZI_Delivery.ConfirmDelivery(...)",
  oContext
);
oOperation.getBoundContext().setProperty("PostingDate", "20260601");
oOperation.getBoundContext().setProperty("ActualQty",   10);
oOperation.execute();

// Static / unbound action (no context)
var oOperation = oModel.bindContext("/ZI_Delivery.CreateFromPO(...)");
oOperation.execute();
```

## EML Action Call (from ABAP)

```abap
" Call bound action via EML
MODIFY ENTITIES OF zi_delivery
  ENTITY Delivery
    EXECUTE PostGoodsIssue
    FROM VALUE #( ( %key-DeliveryNo = '0180000001' ) )
  RESULT DATA(lt_result)
  FAILED DATA(ls_failed)
  REPORTED DATA(ls_reported).

" Call action with parameter via EML
MODIFY ENTITIES OF zi_delivery
  ENTITY Delivery
    EXECUTE ConfirmDelivery
    FROM VALUE #( ( %key-DeliveryNo = '0180000001'
                    %param-PostingDate = '20260601'
                    %param-ActualQty   = '10'
                    %param-ActualUnit  = 'PC' ) )
  RESULT DATA(lt_confirm_result).

COMMIT ENTITIES.
```

## Feature Control Implementation

```abap
" Static feature control in BDEF (always enabled/disabled):
field ( readonly ) DeliveryNo;           " never editable
field ( mandatory ) ShipToName;          " always required
action ( features : global ) PostGoodsIssue;   " checked in get_global_features

" Dynamic feature control:
action ( features : instance ) CancelDelivery;

METHOD get_instance_features.
  READ ENTITIES OF zi_delivery IN LOCAL MODE
    ENTITY Delivery FIELDS ( Status )
    WITH CORRESPONDING #( keys )
  RESULT DATA(lt_deliveries) FAILED failed.

  result = VALUE #(
    FOR <dlv> IN lt_deliveries
    LET can_cancel  = COND #( WHEN <dlv>-Status = 'O'
                               THEN if_abap_behv=>fc-o-enabled
                               ELSE if_abap_behv=>fc-o-disabled )
        is_readonly = COND #( WHEN <dlv>-Status = 'C'
                               THEN if_abap_behv=>fc-f-read_only
                               ELSE if_abap_behv=>fc-f-unrestricted )
    IN ( %tky                   = <dlv>-%tky
         %action-CancelDelivery = can_cancel
         %field-ShipToName      = is_readonly )
  ).
ENDMETHOD.

" Global feature control (authorization-based):
METHOD get_global_features.
  AUTHORITY-CHECK OBJECT 'ZEWM_GI' ID 'ACTVT' FIELD '01'.
  result-%action-PostGoodsIssue =
    COND #( WHEN sy-subrc = 0
             THEN if_abap_behv=>fc-o-enabled
             ELSE if_abap_behv=>fc-o-disabled ).
ENDMETHOD.
```
