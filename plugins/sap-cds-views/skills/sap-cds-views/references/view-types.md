# CDS View Types Reference

## Classic CDS View (define view)

```abap
" Legacy form — still works, but cannot be used as RAP BO root.
" Generates a DDIC database view (visible in SE11).

@AbapCatalog.sqlViewName: 'ZV_DELIVERY'   " REQUIRED — DDIC view name ≤16 chars
@AbapCatalog.compiler.compareFilter: true
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'Delivery Classic View'

define view ZI_Delivery_Classic
  as select from zewm_delivery
{
  key delivery_no   as DeliveryNo,
      status        as Status,
      ship_date     as ShipDate
}
```

## View Entity (define view entity)

```abap
" Modern form — no DDIC view generated.
" Used for: RAP child entities, reference/value help views, reporting.
" Can be used as data source in SELECT statements.

@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Delivery Item View Entity'

define view entity ZI_DeliveryItem
  as select from zewm_delivery_item as itm
  association to parent ZI_Delivery as _Delivery
    on $projection.DeliveryNo = _Delivery.DeliveryNo
{
  key itm.delivery_no  as DeliveryNo,
  key itm.item_no      as ItemNo,
      itm.material_no  as MaterialNo,
      itm.quantity     as Quantity,
      itm.unit         as Unit,
      _Delivery
}
```

## Root View Entity (define root view entity)

```abap
" Required for RAP BO root — owns composition tree.
" Cannot be a child of another entity.

@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'Delivery Root View Entity'

define root view entity ZI_Delivery
  as select from zewm_delivery as hdr
  composition [0..*] of ZI_DeliveryItem  as _Items
  association [0..1] to ZI_Carrier       as _Carrier
    on $projection.CarrierId = _Carrier.CarrierId
{
  key hdr.delivery_no         as DeliveryNo,
      hdr.status              as Status,
      hdr.ship_to_name        as ShipToName,

      @Semantics.businessDate.at: true
      hdr.ship_date           as ShipDate,
      hdr.carrier_id          as CarrierId,

      @Semantics.systemDateTime.localInstanceLastChangedAt: true
      hdr.local_last_changed_at as LocalLastChangedAt,

      _Items,
      _Carrier
}
```

## Projection View (as projection on)

```abap
" Consumption layer — thin projection over interface view.
" Carries UI annotations and service-specific restrictions.
" Must match the contract type of the interface view.

@EndUserText.label: 'Delivery Projection'

define root view entity ZC_Delivery
  provider contract transactional_query   " for Fiori / OData
  as projection on ZI_Delivery
{
  @UI.lineItem: [{ position: 10 }]
  @UI.selectionField: [{ position: 10 }]
  key DeliveryNo,

  @UI.lineItem: [{ position: 20, criticality: 'StatusCriticality' }]
  Status,

  @UI.lineItem: [{ position: 30 }]
  ShipToName,

  @UI.lineItem: [{ position: 40 }]
  @UI.selectionField: [{ position: 20 }]
  ShipDate,

  CarrierId,
  LocalLastChangedAt,

  " Virtual field (computed — not in DB):
  @UI.hidden: true
  cast( case Status when 'O' then 2 when 'P' then 1 when 'C' then 3 else 0 end
        as abap.int1 ) as StatusCriticality,

  " Redirect composition to projection child:
  _Items : redirected to composition child ZC_DeliveryItem,
  _Carrier
}
```

## Abstract Entity

```abap
" No DB table — used for action parameter types in RAP BDEFs.
" Cannot be selected from in ABAP.

define abstract entity ZA_PostGIParam
{
  PostingDate : abap.dats;
  ActualQty   : abap.quan(13,3);
  ActualUnit  : abap.unit(3);
  Note        : abap.char(255);
}
```

## Custom Entity

```abap
" Maps to a non-DB data source implemented in a class.
" Used for: external APIs, BAPIs, calculated results with no DB table.

@ObjectModel.query.implementedBy: 'ABAP:ZCL_DELIVERY_QUERY'

define custom entity ZI_DeliveryExternal
{
  key DeliveryNo  : abap.char(10);
      Status      : abap.char(1);
      ShipToName  : abap.char(80);
}

" Implementation class must implement IF_RAP_QUERY_PROVIDER:
CLASS zcl_delivery_query DEFINITION IMPLEMENTING if_rap_query_provider.
  PUBLIC SECTION.
    METHODS if_rap_query_provider~select REDEFINITION.
ENDCLASS.
CLASS zcl_delivery_query IMPLEMENTATION.
  METHOD if_rap_query_provider~select.
    " io_request->get_filter( ) — read $filter from OData request
    " io_response->set_total_number_of_records( lv_count )
    " io_response->set_data( lt_result )
  ENDMETHOD.
ENDCLASS.
```

## CDS View as SELECT Source in ABAP

```abap
" Classic view — can be used as standard DB table in SELECT:
SELECT * FROM zi_delivery_classic INTO TABLE @DATA(lt_deliveries).

" View entity — also usable in SELECT (ABAP 7.54+):
SELECT delivery_no, status FROM zi_delivery INTO TABLE @DATA(lt_d2).

" With association traversal (path expression):
SELECT delivery_no, _carrier-carrier_name
  FROM zi_delivery
  INTO TABLE @DATA(lt_with_carrier).

" In ABAP OO — use the CDS name, not the DDIC view name:
SELECT * FROM zi_delivery
  WHERE status = 'O'
  ORDER BY ship_date
  INTO TABLE @DATA(lt_open).
```

## Choosing View Type

```
Need RAP BO root?              → define root view entity
Need RAP child entity?         → define view entity  (with "association to parent")
Need UI/Fiori consumption?     → define root/view entity + "as projection on"
Need action parameter?         → define abstract entity
Need non-DB data source?       → define custom entity  (+ query class)
Need legacy/simple reporting?  → define view  (with @AbapCatalog.sqlViewName)
```
