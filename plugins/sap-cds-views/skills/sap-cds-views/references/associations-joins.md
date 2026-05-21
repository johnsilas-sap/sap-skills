# CDS Associations and Joins Reference

## Association vs Join

| Use | Association | Join |
|---|---|---|
| Navigable from OData / Fiori | Yes (client uses $expand) | No |
| Reduces DB columns by default | Yes (lazy) | No (always fetched) |
| Required for RAP composition | Yes | No |
| Best for optional related data | Yes | No |
| Best for always-needed columns | No | Yes (inner join) |
| Can be used in SELECT path expr | Yes | N/A |

## Association Types

```abap
define root view entity ZI_Delivery
  as select from zewm_delivery as hdr

  " ── To one (optional — 0..1) ─────────────────────────────────────────
  association [0..1] to ZI_Carrier as _Carrier
    on $projection.CarrierId = _Carrier.CarrierId

  " ── To one (mandatory — 1..1) ────────────────────────────────────────
  association [1..1] to ZI_WarehouseNo as _Warehouse
    on $projection.Lgnum = _Warehouse.Lgnum

  " ── To many (one-to-many) ─────────────────────────────────────────────
  association [0..*] to ZI_DeliveryItem as _Items
    on $projection.DeliveryNo = _Items.DeliveryNo

  " ── Composition (RAP: root OWNS child — cascade delete) ───────────────
  composition [0..*] of ZI_DeliveryItem as _ItemsOwned

  " ── To parent (child side of composition) ─────────────────────────────
  " Written in child view entity ZI_DeliveryItem:
  " association to parent ZI_Delivery as _Delivery
  "   on $projection.DeliveryNo = _Delivery.DeliveryNo
{
  key hdr.delivery_no as DeliveryNo,
      hdr.carrier_id  as CarrierId,
      hdr.lgnum       as Lgnum,

      " Expose associations — OData client can $expand these:
      _Carrier,
      _Warehouse,
      _Items,
      _ItemsOwned
}
```

## Association with Multiple Conditions

```abap
association [0..1] to ZI_StorageBin as _SourceBin
  on  $projection.Lgnum     = _SourceBin.Lgnum
  and $projection.SourceBinType = _SourceBin.LgTyp
  and $projection.SourceBin = _SourceBin.Lgpla
```

## Association with Filter (Filtered To)

```abap
" Narrow association to a specific subset:
association [0..*] to ZI_DeliveryItem as _OpenItems
  on  $projection.DeliveryNo = _OpenItems.DeliveryNo
  and _OpenItems.ItemStatus  = 'O'    " filter on target side
```

## Joins — Inner, Left Outer, Cross

```abap
define view entity ZI_DeliveryWithCarrier
  as select from zewm_delivery as hdr

  " ── Inner join — only rows with matching carrier ─────────────────────
  join zewm_carrier as car
    on hdr.carrier_id = car.carrier_id

  " ── Left outer join — all deliveries, carrier may be null ────────────
  left outer join zewm_route as rte
    on hdr.route_id = rte.route_id

  " ── Left outer to one — hint: at most one match per left row ─────────
  left outer to one join zewm_priority as pri
    on hdr.priority_code = pri.code
{
  key hdr.delivery_no   as DeliveryNo,
      hdr.status        as Status,
      car.carrier_name  as CarrierName,     " from inner join
      rte.route_desc    as RouteDesc,       " from left outer (may be null)
      pri.description   as PriorityText     " from left outer to one
}
```

## Path Expressions

```abap
" Access association field inline (generates JOIN in SQL):
define view entity ZI_DeliveryItem
  as select from zewm_delivery_item as itm
  association to parent ZI_Delivery as _Delivery
    on $projection.DeliveryNo = _Delivery.DeliveryNo
  association [0..1] to I_Material as _Material
    on $projection.MaterialNo = _Material.Material
{
  key itm.delivery_no  as DeliveryNo,
  key itm.item_no      as ItemNo,
      itm.material_no  as MaterialNo,

      " Path expression: navigate association → select field
      _Material.MaterialGroup as MaterialGroup,   " generates LEFT OUTER JOIN
      _Material.BaseUnit      as BaseUnit,

      _Delivery,   " expose for OData $expand
      _Material
}
```

## Redirected Associations (Projection Layer)

```abap
" Projection view must redirect compositions to projection children:
define root view entity ZC_Delivery
  as projection on ZI_Delivery
{
  key DeliveryNo,
      Status,
      CarrierId,

      " Redirect: instead of ZI_DeliveryItem, expose ZC_DeliveryItem:
      _Items    : redirected to composition child ZC_DeliveryItem,
      " Read-only association redirect:
      _Carrier  : redirected to ZC_Carrier
}

define view entity ZC_DeliveryItem
  as projection on ZI_DeliveryItem
{
  key DeliveryNo,
  key ItemNo,
      MaterialNo,
      Quantity,
      " Redirect back-association to parent projection:
      _Delivery : redirected to parent ZC_Delivery
}
```

## Union (Combining Multiple Sources)

```abap
" Combine rows from two tables:
define view entity ZI_AllWarehouseMovements
  as select from zewm_goods_movement as gm
  {
    key gm.movement_id   as MovementId,
        gm.lgnum         as Lgnum,
        gm.quantity      as Quantity,
        'GM'             as MovementType
  }
  union all select from zewm_stock_transfer as st
  {
    key st.transfer_id   as MovementId,
        st.lgnum         as Lgnum,
        st.quantity      as Quantity,
        'ST'             as MovementType
  }
```

## EWM-Specific Associations

```abap
" Delivery header → items → handling units → storage bins
define root view entity ZI_EwmDelivery
  as select from /scwm/d_dlvd as dlv
  composition [0..*] of ZI_EwmDeliveryItem as _Items
  association [0..1] to /scwm/d_lgnum      as _Warehouse
    on $projection.Lgnum = _Warehouse.Lgnum
  association [0..*] to ZI_EwmHu           as _HandlingUnits
    on $projection.DeliveryNo = _HandlingUnits.RefNo
{
  key dlv.deliv_numb  as DeliveryNo,
      dlv.lgnum       as Lgnum,
      dlv.dlv_type    as DeliveryType,
      dlv.stat_ogi    as GoodsIssueStatus,
      dlv.stat_gr     as GoodsReceiptStatus,
      _Items,
      _Warehouse,
      _HandlingUnits
}
```

## Association Cardinality and Performance

```
[1..1]  — inner join generated (guaranteed one match)
[0..1]  — left outer join generated (zero or one)
[0..*]  — no automatic join in SQL — used for OData $expand only
[1..*]  — no automatic join in SQL — use for document-item patterns

" Performance tip: avoid path expressions with [0..*] associations
" (they generate correlated subqueries).
" Instead: expose the association and let the client $expand.

" Path expressions on [0..1] are efficient — generate single LEFT OUTER JOIN.
```
