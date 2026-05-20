---
name: sap-mobile-dev-kit
description: |
  SAP Mobile Development Kit (MDK) skill. Use when building or modifying MDK mobile
  apps in Business Application Studio or VS Code, defining pages (ListReport, Sectioned,
  FormCell, ObjectTable), controls, OData bindings, actions (navigation, OData CRUD,
  barcode scan, rules), JavaScript rules, offline OData store configuration, push
  notifications, or deploying to SAP Mobile Services on BTP. Covers logistics use
  cases: EWM warehouse operations, inventory checks, goods receipt, transfer order
  confirmation, and field service apps connecting to SAP S/4HANA or EWM OData APIs.
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-20"
  sources:
    - "https://help.sap.com/docs/mobile-development-kit-client"
    - "https://help.sap.com/docs/SAP_MOBILE_SERVICES"
---

# SAP Mobile Development Kit (MDK) Skill

## Related Skills

- **sap-ewm**: EWM OData APIs consumed by MDK apps — warehouse tasks, delivery, HU
- **sap-btp**: BTP services — Mobile Services cockpit, destination service, Cloud Foundry
- **sap-abap**: OData service development (SEGW / CDS annotations) for MDK backends

## Architecture Overview

```
MDK App (metadata JSON + JavaScript rules)
  ↓ deployed via MDK tooling
SAP Mobile Services (BTP)
  ↓ proxies OData requests
SAP S/4HANA / EWM / TM backend
  ↓ OData V2/V4 services
```

MDK apps are **metadata-driven** — pages, controls, and navigation are defined in JSON files, not code. Business logic lives in JavaScript rule files.

## Project Structure

```
MyApp.mdkproject/
  ├── Application.app           — App entry point, service URLs, offline config
  ├── Metadata.json             — App metadata (name, version, BAS config)
  ├── Pages/
  │     ├── Main.page           — Entry page JSON
  │     ├── DeliveryList.page
  │     └── DeliveryDetail.page
  ├── Actions/
  │     ├── NavToDetail.action  — Navigation action JSON
  │     ├── CreateGR.action     — OData create action JSON
  │     └── ScanBarcode.action  — Barcode action JSON
  ├── Rules/
  │     ├── GetDeliveries.js    — JavaScript rule
  │     └── ValidateQty.js
  ├── i18n/
  │     └── i18n.properties     — Translated strings
  └── Images/
        └── logo.png
```

## Application.app — Service Registration

```json
{
  "Application": {
    "Name": "EWM Warehouse App",
    "Version": "1.0.0",
    "MainPage": "/MDKApp/Pages/Main.page",
    "Services": [
      {
        "ServiceName": "EWM_SERVICE",
        "ServiceUrl": "/sap/opu/odata/sap/ZEW_DELIVERY_SRV/",
        "EntitySet": "DeliverySet",
        "Destination": "EWM_BACKEND"
      }
    ],
    "OfflineEnabled": true
  }
}
```

## Page Types

| Page Type | JSON `_Type` | Use For |
|---|---|---|
| List Report | `MDK.ListReportPage` | Filterable list with search |
| Sectioned Table | `MDK.SectionedTablePage` | Grouped list, detail sections |
| Form Cell | `MDK.FormCellContainerPage` | Data entry / edit form |
| Object Table | `MDK.ObjectTablePage` | Table with multiple columns |

## Basic Page — Delivery List

```json
{
  "_Type": "MDK.ListReportPage",
  "_Name": "DeliveryList",
  "Caption": "Deliveries",
  "SearchEnabled": true,
  "Query": "$filter=Status eq 'Open'&$orderby=ShipDate asc",
  "ObjectCell": {
    "_Type": "MDK.ObjectTableCell",
    "Title": "{DeliveryNo}",
    "Subhead": "{ShipToName}",
    "StatusText": "{Status}",
    "Footnote": "{ShipDate}",
    "OnPress": "/MDKApp/Actions/NavToDeliveryDetail.action"
  },
  "Service": "/MDKApp/Services/EWM_SERVICE.service",
  "EntitySet": "DeliverySet"
}
```

## Actions — Navigation

```json
{
  "_Type": "MDK.NavigationAction",
  "_Name": "NavToDeliveryDetail",
  "PageToOpen": "/MDKApp/Pages/DeliveryDetail.page",
  "TransitionType": "Push"
}
```

## Actions — OData Create

```json
{
  "_Type": "MDK.ODataAction",
  "_Name": "CreateGoodsReceipt",
  "Service": "/MDKApp/Services/EWM_SERVICE.service",
  "EntitySet": "GoodsReceiptSet",
  "RequestType": "POST",
  "Properties": {
    "DeliveryNo": "{#Property:DeliveryNo}",
    "ActualQty": "$(#Control:QuantityField/#ControlData)",
    "PostingDate": "$(now)"
  },
  "OnSuccess": "/MDKApp/Actions/NavBack.action",
  "OnFailure": "/MDKApp/Actions/ShowError.action"
}
```

## Rules — JavaScript

```javascript
// Rules/GetOpenDeliveries.js
export default function GetOpenDeliveries(context) {
  // Return OData query filter string
  const warehouseNo = context.getAppData().WarehouseNo || '1710';
  return `$filter=WarehouseNo eq '${warehouseNo}' and Status eq 'Open'`;
}
```

```javascript
// Rules/ValidateAndSubmit.js
export default function ValidateAndSubmit(context) {
  const qty = context.getValue('QuantityField');
  const expectedQty = context.binding.ExpectedQty;

  if (!qty || qty <= 0) {
    return context.executeAction({
      '_Type': 'MDK.MessageAction',
      'Message': 'Quantity must be greater than 0',
      'Title': 'Validation Error'
    });
  }

  if (qty > expectedQty) {
    // Prompt user to confirm over-delivery
    return context.executeAction('/MDKApp/Actions/ConfirmOverDelivery.action')
      .then(() => context.executeAction('/MDKApp/Actions/PostGR.action'));
  }

  return context.executeAction('/MDKApp/Actions/PostGR.action');
}
```

## Offline Store Configuration

```json
{
  "OfflineODataProvider": {
    "ServiceName": "EWM_SERVICE",
    "OfflineEnabled": true,
    "RequestObjects": [
      {
        "ServiceName": "EWM_SERVICE",
        "EntitySet": "DeliverySet",
        "ReadLinks": [],
        "Query": "$filter=Status eq 'Open'"
      },
      {
        "ServiceName": "EWM_SERVICE",
        "EntitySet": "WarehouseTaskSet",
        "Query": "$filter=Confirmed eq false"
      }
    ]
  }
}
```

## Key MDK Transactions / Tools

| Tool | Purpose |
|---|---|
| Business Application Studio (BAS) | MDK project editor + deployment |
| VS Code + MDK extension | Alternative editor |
| SAP Mobile Services cockpit (BTP) | App registration, push config, user management |
| MDK Client (iOS/Android) | Native app shell that runs MDK apps |
| `mdk` CLI | Build and deploy from command line |
