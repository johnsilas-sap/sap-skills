# MDK Pages and Controls Reference

## Page Types

### ListReportPage — Filterable List

```json
{
  "_Type": "MDK.ListReportPage",
  "_Name": "DeliveryList",
  "Caption": "Open Deliveries",
  "SearchEnabled": true,
  "SearchPlaceholder": "Search delivery number...",
  "ObjectCell": {
    "_Type": "MDK.ObjectTableCell",
    "Title": "{DeliveryNo}",
    "Subhead": "{ShipToName}",
    "Footnote": "Ship Date: {ShipDate}",
    "StatusText": "{Status}",
    "StatusState": "/MyWarehouseApp/Rules/GetStatusState.js",
    "Accessories": [
      { "AccessoryType": "DisclosureIndicator" }
    ],
    "OnPress": "/MyWarehouseApp/Actions/NavToDeliveryDetail.action"
  },
  "Query": "$filter=Status eq 'Open'&$orderby=ShipDate asc&$expand=Items",
  "Service": "/MyWarehouseApp/Services/EWM_SERVICE.service",
  "EntitySet": "DeliverySet",
  "OnRefresh": "/MyWarehouseApp/Actions/SyncOfflineStore.action"
}
```

### SectionedTablePage — Grouped / Detail

```json
{
  "_Type": "MDK.SectionedTablePage",
  "_Name": "DeliveryDetail",
  "Caption": "{DeliveryNo}",
  "ActionBar": {
    "Items": [
      {
        "_Type": "MDK.BarButton",
        "Caption": "Post GI",
        "SystemItem": "Done",
        "OnPress": "/MyWarehouseApp/Actions/PostGoodsIssue.action"
      }
    ]
  },
  "Sections": [
    {
      "_Type": "MDK.SimplePropertySection",
      "Header": { "Caption": "Header" },
      "Items": [
        {
          "_Type": "MDK.SimplePropertyFormCell",
          "Caption": "Delivery",
          "Value": "{DeliveryNo}",
          "IsEditable": false
        },
        {
          "_Type": "MDK.SimplePropertyFormCell",
          "Caption": "Ship To",
          "Value": "{ShipToName}",
          "IsEditable": false
        },
        {
          "_Type": "MDK.SimplePropertyFormCell",
          "Caption": "Status",
          "Value": "{Status}",
          "IsEditable": false
        }
      ]
    },
    {
      "_Type": "MDK.ObjectTableSection",
      "Header": { "Caption": "Items" },
      "Service": "/MyWarehouseApp/Services/EWM_SERVICE.service",
      "EntitySet": "DeliveryItemSet",
      "Query": "$filter=DeliveryNo eq '{DeliveryNo}'",
      "ObjectCell": {
        "_Type": "MDK.ObjectTableCell",
        "Title": "{MaterialDesc}",
        "Subhead": "Material: {Material}",
        "Footnote": "Qty: {Quantity} {UOM}",
        "OnPress": "/MyWarehouseApp/Actions/NavToItemDetail.action"
      }
    }
  ]
}
```

### FormCellContainerPage — Data Entry

```json
{
  "_Type": "MDK.FormCellContainerPage",
  "_Name": "GoodsReceiptForm",
  "Caption": "Post Goods Receipt",
  "ActionBar": {
    "Items": [
      {
        "_Type": "MDK.BarButton",
        "Caption": "Post",
        "OnPress": "/MyWarehouseApp/Rules/ValidateAndPostGR.js"
      }
    ]
  },
  "FormCells": [
    {
      "_Type": "MDK.SimplePropertyFormCell",
      "Name": "DeliveryNoField",
      "Caption": "Delivery Number",
      "Value": "{DeliveryNo}",
      "IsEditable": false
    },
    {
      "_Type": "MDK.SimplePropertyFormCell",
      "Name": "QuantityField",
      "Caption": "Actual Quantity",
      "PlaceHolder": "Enter quantity",
      "KeyboardType": "DecimalPad",
      "IsEditable": true,
      "IsRequired": true,
      "Value": "{ExpectedQty}"
    },
    {
      "_Type": "MDK.SimplePropertyFormCell",
      "Name": "BatchField",
      "Caption": "Batch Number",
      "PlaceHolder": "Scan or enter batch",
      "IsEditable": true,
      "ActionIcon": "barcode",
      "OnActionPress": "/MyWarehouseApp/Actions/ScanBatch.action"
    },
    {
      "_Type": "MDK.ListPickerFormCell",
      "Name": "StorageTypePicker",
      "Caption": "Storage Type",
      "AllowMultipleSelection": false,
      "Items": "/MyWarehouseApp/Rules/GetStorageTypes.js"
    },
    {
      "_Type": "MDK.DateTimeFormCell",
      "Name": "PostingDateField",
      "Caption": "Posting Date",
      "Mode": "Date",
      "Value": "$(now)"
    }
  ]
}
```

## Controls Reference

### ObjectTableCell

```json
{
  "_Type": "MDK.ObjectTableCell",
  "Title": "{DeliveryNo}",
  "Subhead": "{ShipToName}",
  "Footnote": "{TotalWeight} KG",
  "Description": "{CarrierName}",
  "StatusText": "{Status}",
  "StatusState": "Positive",     " Positive / Negative / Critical / None
  "SubstatusText": "{Priority}",
  "Icons": ["/MyWarehouseApp/Images/hazmat.png"],
  "Accessories": [
    { "AccessoryType": "DisclosureIndicator" }
  ],
  "OnPress": "/MyWarehouseApp/Actions/NavToDetail.action"
}
```

### BarButton (ActionBar)

```json
{
  "_Type": "MDK.BarButton",
  "Caption": "Confirm",
  "SystemItem": "Done",          " Done / Cancel / Add / Edit / Save
  "OnPress": "/MyWarehouseApp/Actions/ConfirmTask.action",
  "Visible": "/MyWarehouseApp/Rules/IsConfirmable.js"
}
```

### SegmentedControl

```json
{
  "_Type": "MDK.SegmentedControl",
  "Name": "StatusFilter",
  "Items": [
    { "Title": "Open", "Value": "Open" },
    { "Title": "In Progress", "Value": "InProgress" },
    { "Title": "Done", "Value": "Done" }
  ],
  "SelectedIndex": 0,
  "OnValueChange": "/MyWarehouseApp/Rules/FilterByStatus.js"
}
```

## Binding Syntax

```json
"{PropertyName}"                    // Direct OData property
"{#Property:EntitySet/Field}"       // Explicit property reference
"$(#Control:FieldName/#ControlData)" // Read value from a control on same page
"$(now)"                            // Current date/time
"{{i18n>KEY}}"                      // i18n translation key
"/MyWarehouseApp/Rules/MyRule.js"   // Rule result used as value
```

## Status State Values

| Value | Color | Use For |
|---|---|---|
| `Positive` | Green | Completed, confirmed, OK |
| `Negative` | Red | Error, rejected, blocked |
| `Critical` | Orange | Warning, partial, at risk |
| `None` | Default | Neutral, informational |
| `Informative` | Blue | In progress, pending |

## Page OnLoad Event

```json
{
  "_Type": "MDK.SectionedTablePage",
  "OnLoaded": "/MyWarehouseApp/Rules/InitializePage.js"
}
```

```javascript
// Rules/InitializePage.js
export default function InitializePage(context) {
  // Set default values, load dependent data
  const pageProxy = context.getPageProxy();
  pageProxy.setCaption('Delivery: ' + context.binding.DeliveryNo);
}
```
