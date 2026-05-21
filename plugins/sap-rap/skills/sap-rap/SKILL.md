---
name: sap-rap
description: |
  RESTful ABAP Programming Model (RAP) for building OData V4 services and Fiori apps on S/4HANA and BTP ABAP Environment. Covers CDS view entity composition trees, behavior definitions (managed/unmanaged/abstract), behavior implementation classes (handler/saver methods), EML (Entity Manipulation Language), draft handling, actions and functions, service definitions and bindings, and ABAP Unit testing for RAP objects.

  Key patterns:
  - Root entity with composition to child (header/item model)
  - Managed provider with late numbering (UUID keys)
  - Unmanaged provider for legacy table access
  - Bound actions (approve/reject/post) with result
  - Draft-enabled editing for Fiori Elements apps
  - Feature control (static/dynamic field/action visibility)
  - Validations and determinations on save/field change
  - EML READ/MODIFY/COMMIT in test classes and logic
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-20"
  abap_version: "S/4HANA 2022+ / BTP ABAP 2405"
---

# SAP RAP (RESTful ABAP Programming Model)

## RAP Object Layers

```
CDS View Entities       → data model + projections
Behavior Definition     → what operations are allowed
Behavior Implementation → ABAP class executing operations
Service Definition      → which entities to expose
Service Binding         → protocol (OData V4) + endpoint URL
```

## Quick Start — Managed BO

```abap
" 1. Root CDS view entity
define root view entity ZI_Delivery
  as select from zewm_delivery
  composition [0..*] of ZI_DeliveryItem as _Items
{
  key delivery_no     as DeliveryNo,
      status          as Status,
      ship_to_name    as ShipToName,
      ship_date       as ShipDate,
      _Items
}

" 2. Behavior definition
managed implementation in class ZBP_Delivery unique;
persistent table zewm_delivery;
draft table zewm_delivery_d;
{
  create; update; delete;
  association _Items { create; }
  action PostGoodsIssue result [1] $self;
  validation ValidateStatus on save { create; update; }
  determination SetDefaults  on modify { create; }
}

" 3. Service definition
define service ZSD_Delivery {
  expose ZC_Delivery as Delivery;
  expose ZC_DeliveryItem as DeliveryItem;
}
```

## Key Transactions / ADT

| Tool | Purpose |
|---|---|
| ADT (Eclipse) | Primary RAP development IDE |
| `/n/iwfnd/v4_admin` | OData V4 service administration |
| `SACM` | Access control (DCL/PFCG) |
| `SE11` | Draft table definition |
| `SDSV` | Service binding activation log |
| ADT → Run As → OData V4 Client | Test service directly |
