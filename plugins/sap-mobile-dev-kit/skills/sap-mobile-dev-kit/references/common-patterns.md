# MDK Common App Patterns Reference

## List-Detail Navigation Pattern

```
DeliveryList.page (ListReportPage)
  ↓ tap row → NavToDeliveryDetail.action
DeliveryDetail.page (SectionedTablePage)
  ↓ tap "Post GI" → ConfirmPostGI.action (ConfirmationDialog)
  ↓ OK → PostGoodsIssue.action (OData POST)
  ↓ OnSuccess → NavBack.action
```

### DeliveryList.page

```json
{
  "_Type": "MDK.ListReportPage",
  "Caption": "Open Deliveries",
  "SearchEnabled": true,
  "EntitySet": "DeliverySet",
  "Service": "/MyWarehouseApp/Services/EWM_SERVICE.service",
  "Query": "/MyWarehouseApp/Rules/GetDeliveryFilter.js",
  "ObjectCell": {
    "_Type": "MDK.ObjectTableCell",
    "Title": "{DeliveryNo}",
    "Subhead": "{ShipToName}",
    "StatusText": "{Status}",
    "StatusState": "/MyWarehouseApp/Rules/GetStatusState.js",
    "OnPress": "/MyWarehouseApp/Actions/NavToDeliveryDetail.action"
  }
}
```

## Barcode Scan → Lookup Pattern

```
TaskList.page
  ↓ tap "Scan HU" button
ScanHUBarcode.action (BarcodeServiceAction)
  ↓ OnSuccess → LookupHU.js (rule: OData read by scanned ID)
  ↓ found → NavToHUDetail.action
  ↓ not found → ShowNotFound.action (MessageAction)
```

### ScanHUBarcode.action

```json
{
  "_Type": "MDK.BarcodeServiceAction",
  "_Name": "ScanHUBarcode",
  "BarcodeType": ["Code128", "QR", "PDF417"],
  "OnSuccess": "/MyWarehouseApp/Rules/LookupScannedHU.js",
  "OnFailure": "/MyWarehouseApp/Actions/ShowScanError.action"
}
```

### LookupScannedHU.js

```javascript
export default function LookupScannedHU(context) {
  const scanned = context.actionResults.ScanHUBarcode.data;

  return context.read(
    '/MyWarehouseApp/Services/EWM_SERVICE.service',
    'HandlingUnitSet',
    [],
    `$filter=HUId eq '${scanned}'&$expand=Items`
  ).then((result) => {
    if (result.length === 0) {
      return context.executeAction({
        '_Type': 'MDK.MessageAction',
        'Title': 'Not Found',
        'Message': `HU ${scanned} not found in open tasks`,
        'MessageType': 'Warning'
      });
    }
    // Binding for next page set to found entity
    return context.executeAction({
      '_Type': 'MDK.NavigationAction',
      'PageToOpen': '/MyWarehouseApp/Pages/HUDetail.page',
      'TransitionType': 'Push'
    });
  });
}
```

## Goods Receipt Confirmation Pattern

```
DeliveryDetail.page
  ↓ tap "Post GR" → NavToGRForm.action
GRForm.page (FormCellContainerPage)
  Fields: DeliveryNo (readonly), ActualQty, Batch, Bin, PostingDate
  BarButton: "Post" → ValidateAndPostGR.js
ValidateAndPostGR.js:
  Validates qty > 0
  If over-delivery → ConfirmOverDelivery dialog
  Calls PostGR.action (OData POST)
  OnSuccess: UploadOffline → NavToDeliveryList
```

### ValidateAndPostGR.js

```javascript
export default function ValidateAndPostGR(context) {
  const pageProxy = context.getPageProxy();
  const qty = parseFloat(pageProxy.getControl('ActualQtyField').getValue() || 0);
  const expectedQty = parseFloat(context.binding.ExpectedQty || 0);

  if (qty <= 0) {
    return context.executeAction({
      '_Type': 'MDK.MessageAction',
      'Title': 'Validation Error',
      'Message': 'Actual quantity must be greater than zero.',
      'MessageType': 'Error'
    });
  }

  if (qty > expectedQty * 1.1) {   // >110% = over-delivery
    return context.executeAction({
      '_Type': 'MDK.ConfirmationDialogAction',
      'Title': 'Over-Delivery',
      'Message': `Quantity ${qty} exceeds expected ${expectedQty}. Continue?`,
      'OKCaption': 'Post Anyway',
      'CancelCaption': 'Cancel',
      'OnOK': '/MyWarehouseApp/Actions/PostGR.action'
    });
  }

  return context.executeAction('/MyWarehouseApp/Actions/PostGR.action');
}
```

## Warehouse Task Confirmation Pattern

```
TaskList.page — lists open warehouse tasks for current user
  ↓ tap task → NavToTaskDetail.action
TaskDetail.page — shows src bin, dest bin, material, qty
  ↓ tap "Confirm" → ScanDestBin.action (barcode scan)
  ↓ OnSuccess → ConfirmTask.js
ConfirmTask.js:
  Reads actual qty from form
  Calls OData PATCH on WarehouseTaskSet
  OnSuccess: UploadOffline → back to TaskList
```

### TaskList page filter rule

```javascript
// Rules/GetTaskFilter.js
export default function GetTaskFilter(context) {
  const clientData = context.getClientData();
  const userId = clientData.userId.toUpperCase();
  return `$filter=AssignedUser eq '${userId}' and Confirmed eq false`
       + `&$orderby=Priority desc,CreatedAt asc`;
}
```

### ConfirmTask.js

```javascript
export default function ConfirmTask(context) {
  const pageProxy = context.getPageProxy();
  const actualQty = parseFloat(pageProxy.getControl('ActualQtyField').getValue());
  const destBin = context.actionResults.ScanDestBin
    ? context.actionResults.ScanDestBin.data
    : pageProxy.getControl('DestBinField').getValue();

  const readLink = context.binding['@odata.readLink'];

  return context.update(
    '/MyWarehouseApp/Services/EWM_SERVICE.service',
    'WarehouseTaskSet',
    readLink,
    { Confirmed: true, ActualQuantity: actualQty, DestinationBin: destBin }
  )
  .then(() => context.executeAction('/MyWarehouseApp/Actions/UploadAndRefresh.action'))
  .catch((err) => context.executeAction({
    '_Type': 'MDK.MessageAction',
    'Title': 'Confirmation Failed',
    'Message': err.message || 'Could not confirm task. Check connectivity.',
    'MessageType': 'Error'
  }));
}
```

## Inventory Count Pattern

```
CountList.page — list of inventory count documents
  ↓ select count doc → CountDetail.page
CountDetail.page — items to count
  ↓ tap item → CountEntry.page (FormCell)
CountEntry.page:
  Fields: Material (readonly), StorageBin (readonly), CountedQty (editable)
  BarButton: "Record Count" → SaveCount.action (OData PATCH)
  ↓ back to CountDetail → next item
```

## Offline-First Pattern (Field Work with No Connectivity)

```
App launch (online):
  SyncOfflineStore.action → downloads all open tasks, deliveries, bins

User works (offline):
  Reads: ListReport pages read from local store
  Writes: OData creates/updates go into upload queue

End of shift (online again):
  Manual "Sync" button → UploadOfflineChanges.action
  → uploads all queued changes to SAP
  → DownloadOfflineStore.action refreshes data
```

### Sync button in ActionBar

```json
{
  "_Type": "MDK.BarButton",
  "Caption": "Sync",
  "SystemItem": "Refresh",
  "OnPress": "/MyWarehouseApp/Actions/SyncOfflineStore.action"
}
```

## Status Badge / Color Coding Rule

```javascript
// Rules/GetStatusState.js — returns MDK status state string
export default function GetStatusState(context) {
  const status = context.binding.Status;
  const statusMap = {
    'Open':       'Informative',   // Blue
    'InProcess':  'Critical',      // Orange
    'Completed':  'Positive',      // Green
    'Blocked':    'Negative',      // Red
    'Cancelled':  'Negative'
  };
  return statusMap[status] || 'None';
}
```

## Pull-to-Refresh on List Page

```json
{
  "_Type": "MDK.ListReportPage",
  "OnRefresh": "/MyWarehouseApp/Actions/SyncOfflineStore.action"
}
```

## Dynamically Show/Hide BarButton

```javascript
// Rules/IsTaskConfirmable.js
export default function IsTaskConfirmable(context) {
  const status = context.binding.Status;
  const assignedUser = context.binding.AssignedUser;
  const currentUser = context.getClientData().userId.toUpperCase();

  return status === 'Open' && assignedUser === currentUser;
}
```

```json
{
  "_Type": "MDK.BarButton",
  "Caption": "Confirm",
  "OnPress": "/MyWarehouseApp/Actions/ConfirmTask.action",
  "Visible": "/MyWarehouseApp/Rules/IsTaskConfirmable.js"
}
```
