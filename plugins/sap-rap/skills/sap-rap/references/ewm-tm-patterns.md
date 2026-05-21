# RAP Patterns for EWM and TM

## EWM Warehouse Task BO

```abap
" Interface CDS — Warehouse Task header
define root view entity ZI_WhoHeader
  as select from /scwm/ordim_c as who
  composition [1..*] of ZI_WhoStep as _Steps
  association [0..1] to ZI_WhoResource as _Resource
    on $projection.Rsrc = _Resource.Rsrc
   and $projection.Lgnum = _Resource.Lgnum
{
  key who.lgnum        as Lgnum,
  key who.who          as WhoId,
      who.refnum       as RefNum,
      who.who_type     as WhoType,
      who.status_a     as Status,
      who.rsrc         as Rsrc,
      who.priority     as Priority,
      who.local_last_changed_at as LocalLastChangedAt,
      _Steps,
      _Resource
}

" Child step entity
define view entity ZI_WhoStep
  as select from /scwm/ordim_c as stp
  association to parent ZI_WhoHeader as _Header
    on $projection.Lgnum = _Header.Lgnum
   and $projection.WhoId  = _Header.WhoId
{
  key stp.lgnum        as Lgnum,
  key stp.who          as WhoId,
  key stp.stenum       as StepNo,
      stp.matnr        as MaterialNo,
      stp.vsola        as QuantityPlanned,
      stp.vsolb        as QuantityUoM,
      stp.vltyp        as SourceBinType,
      stp.vlber        as SourceSection,
      stp.vlpla        as SourceBin,
      stp.nltypv       as DestBinType,
      stp.nlber        as DestSection,
      stp.nlpla        as DestBin,
      stp.huident      as HuIdent,
      stp.local_last_changed_at as LocalLastChangedAt,
      _Header
}
```

```abap
" BDEF — Warehouse Task BO
managed;
persistent table /scwm/ordim_c;
draft table zewm_who_d;
lock master total etag LocalLastChangedAt;

define behavior for ZI_WhoHeader alias WhoHeader
{
  " No create/delete — WHO created by EWM internally
  update;

  association _Steps { }   " read navigation only

  action ( features : instance ) ConfirmStep  result [1] $self;
  action ( features : instance ) CancelTask   result [1] $self;
  action AssignResource  parameter ZA_AssignResourceParam result [1] $self;

  validation ValidateConfirm on save { update; };
  determination SetStatusOnConfirm on modify { field Status; };

  field ( readonly ) Lgnum, WhoId, WhoType, RefNum;
}

define behavior for ZI_WhoStep alias WhoStep
{
  update;
  lock dependent by _Header;
  authorization dependent by _Header;

  field ( readonly ) Lgnum, WhoId, StepNo;
  field ( mandatory ) QuantityPlanned;
}
```

```abap
" Handler: Confirm step — calls EWM confirmation API
METHOD confirm_step.
  READ ENTITIES OF zi_who_header IN LOCAL MODE
    ENTITY WhoHeader FIELDS ( Lgnum WhoId Status )
    WITH CORRESPONDING #( keys )
  RESULT DATA(lt_whos) FAILED failed.

  LOOP AT lt_whos ASSIGNING FIELD-SYMBOL(<who>).
    IF <who>-Status = 'C'.
      APPEND VALUE #( %tky = <who>-%tky ) TO failed-whoheader.
      APPEND VALUE #(
        %tky = <who>-%tky
        %msg = new_message_with_text( severity = 'E' text = 'Task already confirmed' )
      ) TO reported-whoheader.
      CONTINUE.
    ENDIF.

    " Call EWM confirmation function
    CALL FUNCTION '/SCWM/WHO_CONF'
      EXPORTING
        iv_lgnum = <who>-Lgnum
        iv_who   = <who>-WhoId
      EXCEPTIONS
        not_found = 1
        OTHERS    = 2.

    IF sy-subrc <> 0.
      APPEND VALUE #( %tky = <who>-%tky ) TO failed-whoheader.
    ENDIF.
  ENDLOOP.

  READ ENTITIES OF zi_who_header IN LOCAL MODE
    ENTITY WhoHeader ALL FIELDS
    WITH CORRESPONDING #( keys )
  RESULT DATA(lt_result).
  result = VALUE #( FOR ls IN lt_result ( %tky = ls-%tky %param = ls ) ).
ENDMETHOD.
```

## EWM Delivery Processing BO

```abap
" Projection CDS with EWM-specific annotations
define root view entity ZC_EwmDelivery
  provider contract transactional_query
  as projection on ZI_EwmDelivery
{
  @UI.facet: [
    { id: 'GeneralSection', type: #COLLECTION, label: 'General' },
    { id: 'ShipTo',   type: #FIELDGROUP_REFERENCE, parentId: 'GeneralSection',
      targetQualifier: 'ShipTo',  label: 'Ship To' },
    { id: 'Shipping', type: #FIELDGROUP_REFERENCE, parentId: 'GeneralSection',
      targetQualifier: 'Shipping', label: 'Shipping' },
    { id: 'Items', type: #LINEITEM_REFERENCE, targetElement: '_Items', label: 'Items' }
  ]

  @UI.headerInfo: { typeName: 'Delivery', typeNamePlural: 'Deliveries',
                    title: { value: 'DeliveryNo' }, description: { value: 'ShipToName' } }
  key DeliveryNo,

  @UI.lineItem: [{ position: 10, criticality: 'StatusCriticality' }]
  @UI.fieldGroup: [{ qualifier: 'ShipTo', position: 10 }]
  Status,

  @UI.lineItem: [{ position: 20 }]
  @UI.fieldGroup: [{ qualifier: 'ShipTo', position: 20 }]
  ShipToName,

  @UI.lineItem: [{ position: 30 }]
  @UI.fieldGroup: [{ qualifier: 'Shipping', position: 10 }]
  @UI.selectionField: [{ position: 10 }]
  ShipDate,

  @UI.fieldGroup: [{ qualifier: 'Shipping', position: 20 }]
  CarrierId,

  @UI.lineItem: [{
    position: 90,
    type: #FOR_ACTION,
    dataAction: 'PostGoodsIssue',
    label: 'Post GI',
    inline: true
  }]
  StatusCriticality,

  LocalLastChangedAt,

  _Items : redirected to composition child ZC_EwmDeliveryItem
}
```

## TM Freight Order BO

```abap
" TM Freight Order — approval workflow
define root view entity ZI_TmFreightOrder
  as select from /scmtms/d_tor as fro
{
  key fro.client          as Client,
  key fro.node_id         as FroId,
      fro.fro_category    as FroCategory,
      fro.lifecycle_status as LifecycleStatus,
      fro.planning_status as PlanningStatus,
      fro.execution_status as ExecutionStatus,
      fro.shipper_id      as ShipperId,
      fro.carrier_id      as CarrierId,
      fro.dep_loc_id      as DepartureLocationId,
      fro.arr_loc_id      as ArrivalLocationId,
      fro.request_date_d  as RequestedDepartureDate,
      fro.request_date_a  as RequestedArrivalDate,
      fro.local_last_changed_at as LocalLastChangedAt
}

" BDEF with TM approval actions
managed;
persistent table /scmtms/d_tor;
lock master total etag LocalLastChangedAt;

define behavior for ZI_TmFreightOrder alias FreightOrder
{
  update;

  action ( features : instance ) SubmitForApproval  result [1] $self;
  action ( features : instance ) ApproveFreightOrder result [1] $self;
  action ( features : instance ) RejectFreightOrder  parameter ZA_RejectParam result [1] $self;
  action AssignCarrier           parameter ZA_CarrierParam  result [1] $self;

  validation ValidateCarrier  on save { update; };
  determination UpdatePlanningStatus on modify { field CarrierId; };

  field ( readonly ) FroId, FroCategory, ShipperId;
}

" Handler: Approve — calls TM status management
METHOD approve_freight_order.
  LOOP AT keys ASSIGNING FIELD-SYMBOL(<key>).
    CALL FUNCTION '/SCMTMS/FRO_SET_STATUS'
      EXPORTING
        iv_fro_id = <key>-FroId
        iv_status = 'APR'
      EXCEPTIONS OTHERS = 1.

    IF sy-subrc <> 0.
      APPEND VALUE #( %tky = <key>-%tky ) TO failed-freightorder.
    ENDIF.
  ENDLOOP.

  READ ENTITIES OF zi_tm_freight_order IN LOCAL MODE
    ENTITY FreightOrder ALL FIELDS
    WITH CORRESPONDING #( keys )
  RESULT DATA(lt_result).
  result = VALUE #( FOR ls IN lt_result ( %tky = ls-%tky %param = ls ) ).
ENDMETHOD.
```

## EWM Handling Unit BO (Packing)

```abap
" HU header → items composition
define root view entity ZI_HuHeader
  as select from /scwm/huhdr as hu
  composition [0..*] of ZI_HuItem as _Items
{
  key hu.lgnum     as Lgnum,
  key hu.huident   as HuIdent,
      hu.matnr     as PackagingMaterial,
      hu.vsolm     as GrossWeight,
      hu.mvsolm    as WeightUOM,
      hu.packst    as PackStatus,
      hu.local_last_changed_at as LocalLastChangedAt,
      _Items
}

" BDEF — allow packing items into HU
managed;
persistent table /scwm/huhdr;
lock master total etag LocalLastChangedAt;

define behavior for ZI_HuHeader alias HuHeader
{
  create;
  update;
  delete;

  association _Items { create; }

  action CloseHu     result [1] $self;
  action OpenHu      result [1] $self;
  action PrintHuLabel result [1] $self;

  validation ValidatePackedQty on save { create; update; };

  field ( readonly ) HuIdent, Lgnum;
}
```

## Service Definition for Multiple Logistics Entities

```abap
" Combined EWM logistics service
define service ZSD_EwmLogistics {
  expose ZC_EwmDelivery     as Delivery;
  expose ZC_EwmDeliveryItem as DeliveryItem;
  expose ZC_WhoHeader       as WarehouseOrder;
  expose ZC_WhoStep         as WarehouseTask;
  expose ZC_HuHeader        as HandlingUnit;
  expose ZC_HuItem          as HandlingUnitItem;

  // Value helps
  expose ZVH_StorageBin     as StorageBin;
  expose ZVH_Material       as Material;
  expose ZVH_Resource       as Resource;
}
```
