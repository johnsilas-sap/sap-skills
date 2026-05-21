# RAP Data Model Reference

## CDS View Entity vs Classic CDS View

```abap
" RAP uses view ENTITIES — not define view (DDL source type matters)
define root view entity ZI_Delivery   " root of composition tree
define view entity ZI_DeliveryItem    " child — no "root"

" Classic CDS (not RAP-managed):
define view ZV_Delivery ...           " still usable for read-only reporting
```

## Composition Tree (Header/Item)

```abap
" Root entity — owns composition
define root view entity ZI_Delivery
  as select from zewm_delivery as hdr
  composition [0..*] of ZI_DeliveryItem as _Items
  association [0..1] to ZI_Carrier      as _Carrier  on $projection.CarrierId = _Carrier.CarrierId
{
  key hdr.delivery_no   as DeliveryNo,
      hdr.status        as Status,
      hdr.ship_to_name  as ShipToName,
      hdr.ship_date     as ShipDate,
      hdr.carrier_id    as CarrierId,
      hdr.created_by    as CreatedBy,
      hdr.created_at    as CreatedAt,
      hdr.changed_at    as ChangedAt,
      hdr.local_last_changed_at as LocalLastChangedAt,  " required for ETag/draft

      // Associations exposed — client can $expand
      _Items,
      _Carrier
}

" Child entity — back-association to root is required
define view entity ZI_DeliveryItem
  as select from zewm_delivery_item as itm
  association to parent ZI_Delivery as _Delivery on $projection.DeliveryNo = _Delivery.DeliveryNo
  association [0..1] to I_Material  as _Material  on $projection.MaterialNo = _Material.Material
{
  key itm.delivery_no   as DeliveryNo,
  key itm.item_no       as ItemNo,
      itm.material_no   as MaterialNo,
      itm.quantity      as Quantity,
      itm.unit          as Unit,
      itm.storage_bin   as StorageBin,
      itm.batch         as Batch,
      itm.changed_at    as ChangedAt,
      itm.local_last_changed_at as LocalLastChangedAt,

      _Delivery,
      _Material
}
```

## Projection View (Consumption Layer)

```abap
" Projection = what the service exposes — annotate UI here, not in interface view
define root view entity ZC_Delivery
  provider contract transactional_query
  as projection on ZI_Delivery
{
  @UI.facet: [{ id: 'Items', type: #LINEITEM_REFERENCE, targetElement: '_Items', label: 'Items' }]
  @UI.lineItem: [{ position: 10, label: 'Delivery No.' }]
  key DeliveryNo,

  @UI.lineItem: [{ position: 20, criticality: 'StatusCriticality' }]
  @UI.selectionField: [{ position: 10 }]
  Status,

  @UI.lineItem: [{ position: 30 }]
  ShipToName,

  @UI.lineItem: [{ position: 40 }]
  @UI.selectionField: [{ position: 20 }]
  ShipDate,

  CarrierId,
  CreatedBy,
  CreatedAt,
  LocalLastChangedAt,

  // Virtual field for status color
  @UI.hidden: true
  cast(case Status
    when 'O' then 2
    when 'P' then 1
    when 'C' then 3
    else 0 end as abap.int1) as StatusCriticality,

  // Redirect composition to projection child
  _Items : redirected to composition child ZC_DeliveryItem,
  _Carrier
}

define view entity ZC_DeliveryItem
  provider contract transactional_query
  as projection on ZI_DeliveryItem
{
  @UI.lineItem: [{ position: 10 }]
  key DeliveryNo,
  @UI.lineItem: [{ position: 20 }]
  key ItemNo,
  @UI.lineItem: [{ position: 30 }]
  MaterialNo,
  @UI.lineItem: [{ position: 40 }]
  Quantity,
  Unit,
  StorageBin,
  Batch,
  LocalLastChangedAt,

  _Delivery : redirected to parent ZC_Delivery,
  _Material
}
```

## UUID-Based Keys (Late Numbering)

```abap
" For managed provider with system-assigned keys
define root view entity ZI_WarehouseTask
  as select from zewm_wh_task
{
  key task_uuid       as TaskUUID,        " SYSUUID_X16 (raw16)
      delivery_no     as DeliveryNo,
      source_bin      as SourceBin,
      dest_bin        as DestBin,
      quantity        as Quantity,
      local_last_changed_at as LocalLastChangedAt
}

" Table definition for UUID key:
" task_uuid TYPE sysuuid_x16,  " key
" In BDEF: use numbering: managed (system assigns UUID on create)
```

## Abstract Entity (For Action Parameters)

```abap
" Used as parameter type for actions — no database table needed
define abstract entity ZA_PostGIParam
{
  PostingDate  : abap.dats;
  ActualQty    : abap.quan(13,3);
  ActualUnit   : abap.unit(3);
  Note         : abap.char(255);
}
```

## EWM-Specific Data Model Patterns

```abap
" Warehouse Task — header+step model
define root view entity ZI_WhoHeader   " Warehouse Order header
  composition [1..*] of ZI_WhoStep as _Steps
...

define view entity ZI_WhoStep
  association to parent ZI_WhoHeader as _Header
  on $projection.WhoId = _Header.WhoId
...

" Delivery — header, item, packaging hierarchy
define root view entity ZI_HuHeader   " Handling Unit
  composition [0..*] of ZI_HuItem as _Items
...
```

## Behavior-Relevant CDS Annotations

```abap
" ETag — optimistic locking, must be timestamp/counter
@Semantics.systemDateTime.localInstanceLastChangedAt: true
LocalLastChangedAt,

@Semantics.systemDateTime.lastChangedAt: true
ChangedAt,

@Semantics.systemDateTime.createdAt: true
CreatedAt,

@Semantics.user.createdBy: true
CreatedBy,

@Semantics.user.lastChangedBy: true
ChangedBy,

" UUID key
@Semantics.systemUUID: true
key TaskUUID,
```
