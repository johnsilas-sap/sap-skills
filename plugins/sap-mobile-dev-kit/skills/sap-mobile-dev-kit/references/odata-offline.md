# OData and Offline Store Reference

## Online vs. Offline Mode

| Mode | Data source | Write behavior |
|---|---|---|
| Online | Direct OData calls to backend | POST/PATCH sent immediately |
| Offline | Local SQLite store (downloaded) | Changes queued → upload on sync |

MDK offline uses SAP's OData Offline Store — part of SAP Mobile Services SDK.

## Offline Store Configuration (Application.app)

```json
{
  "Application": {
    "OfflineEnabled": true,
    "Services": [
      {
        "ServiceName": "EWM_SERVICE",
        "ServiceUrl": "/sap/opu/odata/sap/ZEWM_DELIVERY_SRV/",
        "Destination": "EWM_BACKEND",
        "ODataVersion": "V2",
        "Offline": true
      }
    ],
    "OfflineODataProvider": {
      "DefiningRequests": [
        {
          "Name": "OpenDeliveries",
          "ServiceName": "EWM_SERVICE",
          "EntitySet": "DeliverySet",
          "Query": "$filter=Status eq 'Open'&$expand=Items"
        },
        {
          "Name": "OpenTasks",
          "ServiceName": "EWM_SERVICE",
          "EntitySet": "WarehouseTaskSet",
          "Query": "$filter=Confirmed eq false and WarehouseNo eq '1710'"
        },
        {
          "Name": "StorageBins",
          "ServiceName": "EWM_SERVICE",
          "EntitySet": "StorageBinSet",
          "Query": "$select=BinId,BinDesc,StorageType,AvailableQty"
        }
      ]
    }
  }
}
```

## Sync Actions

### Download (Backend → Device)

```json
{
  "_Type": "MDK.ApplicationAction",
  "_Name": "DownloadOfflineStore",
  "ApplicationAction": "OfflineDataDownload",
  "OnSuccess": "/MyWarehouseApp/Actions/ShowDownloadSuccess.action",
  "OnFailure": "/MyWarehouseApp/Actions/ShowDownloadError.action"
}
```

### Upload (Device → Backend)

```json
{
  "_Type": "MDK.ApplicationAction",
  "_Name": "UploadOfflineChanges",
  "ApplicationAction": "OfflineDataUpload",
  "OnSuccess": "/MyWarehouseApp/Actions/DownloadOfflineStore.action",
  "OnFailure": "/MyWarehouseApp/Actions/ShowUploadError.action"
}
```

### Full Sync (Upload then Download)

```json
{
  "_Type": "MDK.ApplicationAction",
  "_Name": "SyncOfflineStore",
  "ApplicationAction": "OfflineDataSync",
  "OnSuccess": "/MyWarehouseApp/Actions/ShowSyncSuccess.action",
  "OnFailure": "/MyWarehouseApp/Actions/ShowSyncError.action"
}
```

## Reading Offline Data in Rules

```javascript
// context.read() reads from the local offline store (no network needed)
export default function GetOpenTasks(context) {
  return context.read(
    '/MyWarehouseApp/Services/EWM_SERVICE.service',
    'WarehouseTaskSet',
    [],
    "$filter=Confirmed eq false&$orderby=Priority desc"
  ).then((result) => {
    // result = array of entity objects from local SQLite store
    return result;
  });
}
```

## Writing Offline Data (Queued Until Upload)

```javascript
// context.create() in offline mode queues the POST
export default function CreateOfflineGR(context) {
  const payload = {
    DeliveryNo: context.binding.DeliveryNo,
    ActualQty: parseFloat(context.getPageProxy().getControl('QtyField').getValue()),
    PostingDate: new Date().toISOString().split('T')[0]
  };

  return context.create(
    '/MyWarehouseApp/Services/EWM_SERVICE.service',
    'GoodsReceiptSet',
    payload
  ).then(() => {
    // Change is now in the local queue — user sees it immediately
    // The actual POST to SAP happens when UploadOfflineChanges runs
    return context.executeAction('/MyWarehouseApp/Actions/ShowQueuedMessage.action');
  });
}
```

## OData Service Definition (.service file)

```json
{
  "_Type": "MDK.ODataService",
  "_Name": "EWM_SERVICE",
  "ServiceUrl": "/sap/opu/odata/sap/ZEWM_DELIVERY_SRV/",
  "Destination": "EWM_BACKEND",
  "ODataVersion": "V2",
  "Offline": true,
  "CustomHeaders": {
    "sap-client": "100"
  }
}
```

## OData Version Differences

| Feature | V2 | V4 |
|---|---|---|
| Entity key | `DeliverySet('DLV001')` | `DeliverySet(DeliveryNo='DLV001')` |
| ReadLink | `@odata.id` (V4) vs key predicate (V2) | |
| Batch | `$batch` both versions | V4 uses JSON batch |
| Annotations | `edmx:Annotation` | Rich annotation support |
| Lambda | Not supported | `$filter=Items/any(i: i/Qty gt 0)` |

Most SAP EWM/TM OData services are V2. Check `$metadata` URL to confirm.

## Handling Offline Errors (Upload Conflicts)

```javascript
// Rules/HandleUploadError.js
export default function HandleUploadError(context) {
  const errorResult = context.actionResults.UploadOfflineChanges;

  if (errorResult && errorResult.error) {
    const httpStatus = errorResult.error.responseStatusCode;

    if (httpStatus === 409) {
      // Conflict — data changed on server since last download
      return context.executeAction({
        '_Type': 'MDK.ConfirmationDialogAction',
        'Title': 'Sync Conflict',
        'Message': 'Data was modified on the server. Download latest and retry?',
        'OKCaption': 'Download',
        'OnOK': '/MyWarehouseApp/Actions/DownloadOfflineStore.action'
      });
    }

    return context.executeAction({
      '_Type': 'MDK.MessageAction',
      'Title': 'Upload Failed',
      'Message': `Error ${httpStatus}: ${errorResult.error.message}`
    });
  }
}
```

## OData Expand (Related Entities)

```json
{
  "Query": "$expand=Items,Address&$filter=Status eq 'Open'"
}
```

```javascript
// Accessing expanded navigation property in a rule
export default function GetItemCount(context) {
  const items = context.binding.Items;   // Expanded navigation property
  return items ? items.length : 0;
}
```

## OData Function Imports

```javascript
// Calling a function import (action in OData V4, function in V2)
export default function PostGoodsIssue(context) {
  const deliveryNo = context.binding.DeliveryNo;

  return context.callFunction(
    '/MyWarehouseApp/Services/EWM_SERVICE.service',
    'PostGoodsIssue',                     // Function import name
    { DeliveryNo: deliveryNo },            // Parameters
    'POST'                                 // HTTP method (GET for functions, POST for actions)
  ).then((result) => {
    return context.executeAction('/MyWarehouseApp/Actions/NavBack.action');
  });
}
```
