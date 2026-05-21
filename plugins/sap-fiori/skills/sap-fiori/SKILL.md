---
name: sap-fiori
description: |
  SAP Fiori development skill. Use when building Fiori Elements apps (List Report,
  Object Page, Worklist, Analytical List Page, Overview Page), writing CDS UI
  annotations (@UI.lineItem, @UI.fieldGroup, @UI.selectionField, @UI.identification,
  @UI.facets), developing custom SAPUI5 apps (XML views, controllers, OData models,
  routing), configuring Fiori Launchpad (tiles, target mappings, semantic objects),
  using Fiori Tools in BAS or VS Code, or building OData V4 CAP-backed Fiori apps.
  Covers logistics scenarios: delivery management, warehouse operations, transfer
  order UI, goods receipt apps, and EWM/TM Fiori extensions.
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-20"
  sources:
    - "https://help.sap.com/docs/SAP_FIORI_tools"
    - "https://ui5.sap.com"
    - "https://help.sap.com/docs/btp/sap-business-application-studio"
---

# SAP Fiori Development Skill

## Related Skills

- **sap-btp**: BTP platform — BAS, Cloud Foundry deployment, destinations
- **sap-api-management**: Expose OData services via API gateway for Fiori
- **sap-integration-suite**: Backend integration feeding Fiori data services

## Fiori App Types

| Type | Technology | Best For |
|---|---|---|
| Fiori Elements | OData + CDS annotations | Standard CRUD, reports, object pages |
| Custom SAPUI5 | SAPUI5 MVC + JS/XML | Complex UI, non-standard interactions |
| Freestyle UI5 | SAPUI5 with Fiori shell | Custom layouts, dashboards |
| CAP + Fiori Elements | CDS + OData V4 + annotations | BTP-hosted full-stack apps |

## Fiori Elements Page Types

| Page Type | Floorplan | Use For |
|---|---|---|
| `ListReport` | List Report + Object Page | Master-detail browse and edit |
| `ObjectPage` | Object Page | Detail view with sections/facets |
| `AnalyticalListPage` | ALP | KPI tiles + filterable chart + table |
| `OverviewPage` | Overview Page | Dashboard with multiple cards |
| `Worklist` | Worklist | Simple task list, no filter bar |

## Quick Start: Fiori Elements App (BAS)

```
BAS → File → New Project → Fiori Application
  Template: List Report Page
  Data source: Connect to an OData Service
  OData URL: https://my-s4.ondemand.com/sap/opu/odata4/sap/...
  Entity: A_OutbDeliveryHeader
  Navigation: A_OutbDeliveryItem

  " Generates complete app with manifest.json, views, and annotations
```

## CDS Entity for Fiori (Minimal)

```cds
// srv/delivery-service.cds
using { ZEWM_DELIVERY_SRV as external } from '../srv/external/ZEWM_DELIVERY_SRV.csn';

service DeliveryService @(path:'/delivery') {
  entity Deliveries as projection on external.DeliverySet {
    key DeliveryNo,
        Status,
        ShipToName,
        ShipDate,
        TotalWeight,
        CarrierName
  }
  annotate Deliveries with @(
    UI.LineItem: [
      { Value: DeliveryNo, Label: 'Delivery' },
      { Value: ShipToName, Label: 'Ship To' },
      { Value: Status,     Label: 'Status' },
      { Value: ShipDate,   Label: 'Ship Date' }
    ],
    UI.SelectionFields: [ Status, ShipDate ]
  );
}
```

## manifest.json (App Descriptor)

```json
{
  "_version": "1.65.0",
  "sap.app": {
    "id": "com.mycompany.ewm.deliverylist",
    "type": "application",
    "title": "Delivery List",
    "description": "EWM Delivery Management",
    "dataSources": {
      "mainService": {
        "uri": "/sap/opu/odata4/sap/ZEWM_DELIVERY_SRV/",
        "type": "OData",
        "settings": { "odataVersion": "4.0" }
      }
    }
  },
  "sap.ui5": {
    "routing": {
      "routes": [
        { "name": "DeliveryList",   "pattern": "",                    "target": "DeliveryList" },
        { "name": "DeliveryDetail", "pattern": "Deliveries({key})",   "target": "DeliveryDetail" }
      ],
      "targets": {
        "DeliveryList":   { "type": "Component", "id": "DeliveryList",   "name": "sap.fe.templates.ListReport" },
        "DeliveryDetail": { "type": "Component", "id": "DeliveryDetail", "name": "sap.fe.templates.ObjectPage" }
      }
    }
  }
}
```

## Key Transactions

| Transaction | Purpose |
|---|---|
| `/n/UI2/FLP` | Fiori Launchpad (ABAP) |
| `/n/UI2/FLPD_CONF` | FLP Designer (configure tiles) |
| `SE80` / `SE38` | ABAP backend service development |
| `SEGW` | OData V2 service builder (legacy) |
| BAS | Business Application Studio — Fiori development IDE |
| VS Code + Fiori Tools | Local development alternative |
| `SAP Fiori Tools` | Yeoman generator, annotation modeler, guided dev |
