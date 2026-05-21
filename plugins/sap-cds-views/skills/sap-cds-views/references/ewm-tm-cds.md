# EWM and TM CDS Patterns Reference

## EWM Standard CDS Views (VDM)

```
" Key SAP-delivered EWM CDS views (read-only — extend or project, don't modify):

/SCWM/I_Delivery              — Delivery header interface view
/SCWM/I_DeliveryItem          — Delivery item
/SCWM/I_WarehouseOrder        — Warehouse order (WHO) header
/SCWM/I_WarehouseTask         — Warehouse task (WHT) step
/SCWM/I_HandlingUnit          — Handling unit header
/SCWM/I_HandlingUnitItem      — HU item / packed material
/SCWM/I_StorageBin            — Storage bin master
/SCWM/I_Resource              — Warehouse resource (forklift/user)
/SCWM/I_Wave                  — Wave header
/SCWM/I_ProductionOrder       — Production order supply (EWM-PP)

" Consumption (projection) views — use for OData services:
/SCWM/C_Delivery
/SCWM/C_WarehouseOrder
/SCWM/C_HandlingUnit
```

## EWM Delivery CDS — Custom Extension Pattern

```abap
" Interface view — extends SAP base or custom table
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'EWM Delivery Interface'

define root view entity ZI_EwmDelivery
  as select from /scwm/d_dlvd as dlv
  join /scwm/d_dlvh as hdr
    on dlv.del_id = hdr.del_id

  composition [0..*] of ZI_EwmDeliveryItem as _Items
  composition [0..*] of ZI_EwmDeliveryHU   as _HandlingUnits

  association [0..1] to ZI_EwmWarehouse as _Warehouse
    on $projection.Lgnum = _Warehouse.Lgnum
  association [0..1] to ZI_EwmResource  as _Resource
    on $projection.Rsrc = _Resource.Rsrc
   and $projection.Lgnum = _Resource.Lgnum
{
  key dlv.lgnum          as Lgnum,
  key dlv.deliv_numb     as DeliveryNo,
      dlv.dlv_type       as DeliveryType,
      dlv.direction      as Direction,           " 'A' inbound / 'B' outbound
      dlv.stat_ogi       as GoodsIssueStatus,
      dlv.stat_gr        as GoodsReceiptStatus,
      dlv.stat_over      as OverallStatus,
      hdr.ship_to_name   as ShipToName,
      hdr.route          as Route,
      hdr.carrier_name   as CarrierName,

      @Semantics.businessDate.at: true
      hdr.req_dlv_date   as RequestedDeliveryDate,

      dlv.rsrc           as Rsrc,

      @Semantics.systemDateTime.localInstanceLastChangedAt: true
      dlv.local_lchg_tstmp as LocalLastChangedAt,

      _Items,
      _HandlingUnits,
      _Warehouse,
      _Resource
}

" Projection for Fiori:
define root view entity ZC_EwmDelivery
  provider contract transactional_query
  as projection on ZI_EwmDelivery
{
  @UI.headerInfo: {
    typeName: 'Delivery', typeNamePlural: 'Deliveries',
    title: { value: 'DeliveryNo' }, description: { value: 'ShipToName' }
  }

  @UI.lineItem: [{ position: 10 }]
  @UI.selectionField: [{ position: 10 }]
  key Lgnum,

  @UI.lineItem: [{ position: 20 }]
  key DeliveryNo,

  @UI.lineItem: [{ position: 30 }]
  ShipToName,

  @UI.lineItem: [{ position: 40, criticality: 'OverallStatusCriticality' }]
  @UI.selectionField: [{ position: 20 }]
  OverallStatus,

  @UI.lineItem: [{ position: 50 }]
  @UI.selectionField: [{ position: 30 }]
  RequestedDeliveryDate,

  CarrierName,
  GoodsIssueStatus,
  GoodsReceiptStatus,
  LocalLastChangedAt,

  @UI.hidden: true
  cast( case OverallStatus
    when 'A' then 3   " success
    when 'B' then 1   " warning
    when 'C' then 2   " error
    else 0
  end as abap.int1 ) as OverallStatusCriticality,

  _Items : redirected to composition child ZC_EwmDeliveryItem,
  _HandlingUnits,
  _Warehouse,
  _Resource
}
```

## EWM Warehouse Task CDS

```abap
define view entity ZI_EwmWho
  as select from /scwm/ordim_c as wht
  association to parent ZI_EwmDelivery as _Delivery
    on $projection.RefNum = _Delivery.DeliveryNo
   and $projection.Lgnum  = _Delivery.Lgnum
  association [0..1] to ZI_EwmResource as _Resource
    on $projection.Rsrc  = _Resource.Rsrc
   and $projection.Lgnum = _Resource.Lgnum
  association [0..1] to ZI_EwmStorageBin as _SourceBin
    on $projection.Lgnum    = _SourceBin.Lgnum
   and $projection.SourceBinType = _SourceBin.LgTyp
   and $projection.SourceBin = _SourceBin.StorageBin
{
  key wht.lgnum        as Lgnum,
  key wht.tanum        as TaskNo,
      wht.who          as WhoId,
      wht.refnum       as RefNum,
      wht.procty       as ProcessType,
      wht.flgto        as TaskType,      " 'PICK' / 'PUT' / 'MOVE'
      wht.matnr        as MaterialNo,
      wht.charg        as Batch,

      @Semantics.quantity.unitOfMeasure: 'PlannedUnit'
      wht.vsola        as PlannedQty,
      wht.vsolb        as PlannedUnit,

      @Semantics.quantity.unitOfMeasure: 'ActualUnit'
      wht.nista        as ActualQty,
      wht.nistb        as ActualUnit,

      wht.vltyp        as SourceBinType,
      wht.vlpla        as SourceBin,
      wht.nltypv       as DestBinType,
      wht.nlpla        as DestBin,
      wht.rsrc         as Rsrc,
      wht.priority     as Priority,
      wht.status_d     as Status,

      wht.local_lchg_tstmp as LocalLastChangedAt,

      _Resource,
      _SourceBin
}
```

## TM Freight Order CDS

```abap
define root view entity ZI_TmFreightOrder
  as select from /scmtms/d_tor as fro
  composition [0..*] of ZI_TmFreightOrderItem as _Items
  association [0..1] to ZI_TmCarrier as _Carrier
    on $projection.CarrierId = _Carrier.CarrierId
  association [0..1] to ZI_TmLocation as _DepLoc
    on $projection.DepartureLocationId = _DepLoc.LocationId
  association [0..1] to ZI_TmLocation as _ArrLoc
    on $projection.ArrivalLocationId = _ArrLoc.LocationId
{
  key fro.client             as Client,
  key fro.node_id            as FroId,
      fro.fro_category       as FroCategory,
      fro.lifecycle_status   as LifecycleStatus,
      fro.planning_status    as PlanningStatus,
      fro.execution_status   as ExecutionStatus,
      fro.carrier_id         as CarrierId,
      fro.dep_loc_id         as DepartureLocationId,
      fro.arr_loc_id         as ArrivalLocationId,
      fro.request_date_d     as RequestedDepartureDate,
      fro.request_date_a     as RequestedArrivalDate,

      @Semantics.amount.currencyCode: 'Currency'
      fro.charge_amount      as ChargeAmount,
      @Semantics.currencyCode: true
      fro.charge_currency    as Currency,

      @Semantics.systemDateTime.localInstanceLastChangedAt: true
      fro.local_lchg_tstmp   as LocalLastChangedAt,

      _Items,
      _Carrier,
      _DepLoc,
      _ArrLoc
}
```

## Value Help CDS Views (EWM)

```abap
" Storage bin value help:
@ObjectModel.dataCategory: #VALUE_HELP
@EndUserText.label: 'Storage Bin Value Help'

define view entity ZVH_EwmStorageBin
  as select from /scwm/lgpla as bin
{
  key bin.lgnum  as Lgnum,
  key bin.lgtyp  as LgTyp,
  key bin.lgpla  as StorageBin,
      bin.lgber  as Section
}

" Carrier value help:
@ObjectModel.dataCategory: #VALUE_HELP
@EndUserText.label: 'Carrier Value Help'

define view entity ZVH_Carrier
  as select from zewm_carrier
{
  key carrier_id   as CarrierId,
      carrier_name as CarrierName,
      country      as Country,
      transport_mode as TransportMode
}

" Usage in data model CDS field:
@Consumption.valueHelpDefinition: [{
  entity: { name: 'ZVH_EwmStorageBin', element: 'StorageBin' },
  additionalBinding: [{ localElement: 'Lgnum', element: 'Lgnum' }]
}]
DestBin,
```

## Analytical CDS for EWM (Task Performance)

```abap
@Analytics.dataCategory: #CUBE
@EndUserText.label: 'Warehouse Task Performance Cube'

define view entity ZI_WhtPerformanceCube
  as select from /scwm/ordim_c as wht
  association [0..1] to ZI_EwmResourceDim  as _Resource
    on wht.rsrc = _Resource.Rsrc and wht.lgnum = _Resource.Lgnum
  association [0..1] to ZI_EwmMaterialDim  as _Material
    on wht.matnr = _Material.MaterialNo
{
  key wht.lgnum     as Lgnum,
  key wht.tanum     as TaskNo,
      wht.procty    as ProcessType,
      wht.rsrc      as Rsrc,
      wht.matnr     as MaterialNo,
      wht.vltyp     as SourceBinType,
      wht.nltypv    as DestBinType,
      wht.status_d  as Status,

      @DefaultAggregation: #NOP
      wht.start_conf as StartTime,
      @DefaultAggregation: #NOP
      wht.end_conf   as EndTime,

      @DefaultAggregation: #SUM
      @Semantics.quantity.unitOfMeasure: 'PlannedUnit'
      wht.vsola      as PlannedQty,
      wht.vsolb      as PlannedUnit,

      @DefaultAggregation: #SUM
      cast( tstmp_seconds_between(
        wht.start_conf, wht.end_conf, 'NULL'
      ) as abap.int4 ) as ConfirmationSeconds,

      @DefaultAggregation: #COUNT_DISTINCT
      cast( 1 as abap.int4 ) as TaskCount,

      _Resource,
      _Material
}
```
