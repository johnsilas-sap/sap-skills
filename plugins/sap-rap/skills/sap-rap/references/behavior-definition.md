# RAP Behavior Definition (BDEF) Reference

## BDEF Syntax Overview

```abap
" ADT: New → Behavior Definition → Source = view entity name
[managed | unmanaged | abstract | projection]
  implementation in class ZBP_<Name> unique
  [persistent table <dbtab>]
  [draft table <draft_dbtab>]
  [lock master | lock dependent by _Parent]
  [total etag LocalLastChangedAt]
  [authorization master ( global ) | authorization dependent by _Parent]
;

define behavior for ZI_Delivery alias Delivery
{
  ...operations...
}

define behavior for ZI_DeliveryItem alias DeliveryItem
{
  ...
}
```

## Managed Provider

```abap
" System handles CRUD persistence automatically
managed
  implementation in class ZBP_Delivery unique;

persistent table zewm_delivery;
draft table     zewm_delivery_d;
lock master total etag LocalLastChangedAt;
authorization master ( global );
etag master LocalLastChangedAt;

define behavior for ZI_Delivery alias Delivery
{
  create;
  update;
  delete;

  // Late numbering — system assigns key after create
  // numbering: managed;    " for UUID keys only

  // Field control
  field ( readonly )  DeliveryNo;        " set by system
  field ( mandatory ) ShipToName, ShipDate;

  // Associations
  association _Items { create; with draft; }

  // Actions
  action PostGoodsIssue   result [1] $self;
  action ConfirmDelivery  parameter ZA_ConfirmParam result [1] $self;
  action ( features : instance ) CancelDelivery result [1] $self;

  // Validations — run on save
  validation ValidateStatus  on save { create; update; }
  validation ValidateShipDate on save { create; update; }

  // Determinations — run on field change or save
  determination SetCreationDefaults on modify { create; }
  determination RecalcTotalWeight   on modify { field Quantity; }  " on item change

  // Side effects (tell UI to refresh other data when field changes)
  side effects {
    field Status affects action PostGoodsIssue;
    field CarrierId affects field _Carrier.*;
  }

  // Mapping: CDS alias → DB column name
  mapping for zewm_delivery corresponding
  {
    DeliveryNo   = delivery_no;
    Status       = status;
    ShipToName   = ship_to_name;
    ShipDate     = ship_date;
    CarrierId    = carrier_id;
    CreatedBy    = created_by;
    CreatedAt    = created_at;
    ChangedAt    = changed_at;
    LocalLastChangedAt = local_last_changed_at;
  }
}

define behavior for ZI_DeliveryItem alias DeliveryItem
{
  update;
  delete;

  field ( readonly ) DeliveryNo, ItemNo;
  field ( mandatory ) MaterialNo, Quantity, Unit;

  // Inherit lock from root
  lock dependent by _Delivery;
  authorization dependent by _Delivery;

  mapping for zewm_delivery_item corresponding
  {
    DeliveryNo = delivery_no;
    ItemNo     = item_no;
    MaterialNo = material_no;
    Quantity   = quantity;
    Unit       = unit;
    StorageBin = storage_bin;
    Batch      = batch;
    LocalLastChangedAt = local_last_changed_at;
  }
}
```

## Unmanaged Provider

```abap
" For legacy tables / complex logic — you handle all CRUD in ABAP
unmanaged
  implementation in class ZBP_Delivery_UM unique;

define behavior for ZI_Delivery alias Delivery
{
  create;
  update;
  delete;

  association _Items { create; }

  action PostGoodsIssue result [1] $self;

  validation ValidateStatus on save { create; update; }

  // No persistent table / mapping — you write to DB yourself in save handler
}
```

## Projection Behavior Definition

```abap
" Thin layer over interface BDEF — exposes subset to service
projection;
use draft;

define behavior for ZC_Delivery alias Delivery
{
  use create;
  use update;
  use delete;
  use association _Items { create; with draft; }
  use action PostGoodsIssue;

  // Can restrict: omit 'use delete' to prevent delete from this service
}

define behavior for ZC_DeliveryItem alias DeliveryItem
{
  use update;
  use delete;
}
```

## Draft Handling in BDEF

```abap
" Enable draft on root entity — requires draft table
managed;
persistent table zewm_delivery;
draft table     zewm_delivery_d;   " SE11 → copy structure from zewm_delivery + draft fields
lock master total etag LocalLastChangedAt;

define behavior for ZI_Delivery alias Delivery
{
  draft action Edit;        " lock + copy active→draft
  draft action Activate;    " save draft→active (triggers validations)
  draft action Discard;     " delete draft record
  draft action Resume;      " reopen existing draft
  draft determine action Prepare;  " pre-validation before activate

  // All regular operations apply to draft instance too
  create; update; delete;
  association _Items { create; with draft; }
}
```

## Authorization Object in BDEF

```abap
" Global authorization — one check for the entire BO
authorization master ( global );

" Instance authorization — per-entity check
authorization master ( instance );

" Dependent — child inherits from root
authorization dependent by _Delivery;
```

## Abstract Behavior (for Business Events)

```abap
" Used for event publishing — no persistence
abstract;

define behavior for ZI_DeliveryEvent alias DeliveryEvent
{
  event DeliveryPosted parameter ZA_DeliveryPostedEvt;
}
```
