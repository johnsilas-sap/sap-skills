# MDK Actions Reference

## Action Types

| Type | `_Type` value | Purpose |
|---|---|---|
| Navigation | `MDK.NavigationAction` | Navigate between pages |
| OData | `MDK.ODataAction` | CRUD operations on OData service |
| Rule | Invoke a `.js` rule directly (use rule path as action) | |
| Message | `MDK.MessageAction` | Show alert/toast to user |
| Confirmation | `MDK.ConfirmationDialogAction` | Yes/No prompt before proceeding |
| Barcode | `MDK.BarcodeServiceAction` | Scan barcode from device camera |
| Set State | `MDK.SetStateAction` | Store/retrieve key-value state |
| Application | `MDK.ApplicationAction` | Sync offline store, logout, etc. |

## Navigation Actions

```json
{
  "_Type": "MDK.NavigationAction",
  "_Name": "NavToDeliveryDetail",
  "PageToOpen": "/MyWarehouseApp/Pages/DeliveryDetail.page",
  "TransitionType": "Push"
}
```

```json
{
  "_Type": "MDK.NavigationAction",
  "_Name": "NavBack",
  "PageToOpen": "_prev",
  "TransitionType": "Pop"
}
```

```json
{
  "_Type": "MDK.NavigationAction",
  "_Name": "NavToModal",
  "PageToOpen": "/MyWarehouseApp/Pages/ScanResult.page",
  "TransitionType": "Modal"
}
```

## OData Create (POST)

```json
{
  "_Type": "MDK.ODataAction",
  "_Name": "PostGoodsReceipt",
  "Service": "/MyWarehouseApp/Services/EWM_SERVICE.service",
  "EntitySet": "GoodsReceiptSet",
  "RequestType": "POST",
  "Properties": {
    "DeliveryNo": "$(#Control:DeliveryNoField/#ControlData)",
    "ActualQty": "$(#Control:QuantityField/#ControlData)",
    "BatchNo": "$(#Control:BatchField/#ControlData)",
    "PostingDate": "$(#Control:PostingDateField/#ControlData)",
    "WarehouseNo": "1710"
  },
  "OnSuccess": "/MyWarehouseApp/Actions/GRSuccess.action",
  "OnFailure": "/MyWarehouseApp/Actions/ShowODataError.action"
}
```

## OData Update (PATCH/PUT)

```json
{
  "_Type": "MDK.ODataAction",
  "_Name": "ConfirmWarehouseTask",
  "Service": "/MyWarehouseApp/Services/EWM_SERVICE.service",
  "EntitySet": "WarehouseTaskSet",
  "RequestType": "PATCH",
  "ReadLink": "{@odata.readLink}",
  "Properties": {
    "Confirmed": true,
    "ActualQty": "$(#Control:ActualQtyField/#ControlData)",
    "DestBin": "$(#Control:DestBinField/#ControlData)"
  },
  "OnSuccess": "/MyWarehouseApp/Actions/TaskConfirmSuccess.action",
  "OnFailure": "/MyWarehouseApp/Actions/ShowODataError.action"
}
```

## OData Delete

```json
{
  "_Type": "MDK.ODataAction",
  "_Name": "DeleteDraft",
  "Service": "/MyWarehouseApp/Services/EWM_SERVICE.service",
  "EntitySet": "DraftSet",
  "RequestType": "DELETE",
  "ReadLink": "{@odata.readLink}"
}
```

## OData Read (Function Import)

```json
{
  "_Type": "MDK.ODataAction",
  "_Name": "GetBinStock",
  "Service": "/MyWarehouseApp/Services/EWM_SERVICE.service",
  "EntitySet": "GetBinStockSet",
  "RequestType": "GET",
  "QueryOptions": "$filter=BinId eq '$(#Control:BinField/#ControlData)'"
}
```

## Message Action (Toast / Alert)

```json
{
  "_Type": "MDK.MessageAction",
  "_Name": "ShowGRSuccess",
  "Title": "Success",
  "Message": "Goods receipt posted for delivery {DeliveryNo}",
  "OKCaption": "OK",
  "OnOK": "/MyWarehouseApp/Actions/NavBack.action",
  "MessageType": "Success"
}
```

```json
{
  "_Type": "MDK.MessageAction",
  "_Name": "ShowError",
  "Title": "Error",
  "Message": "/MyWarehouseApp/Rules/GetErrorMessage.js",
  "OKCaption": "OK",
  "MessageType": "Error"
}
```

## Confirmation Dialog

```json
{
  "_Type": "MDK.ConfirmationDialogAction",
  "_Name": "ConfirmPostGI",
  "Title": "Post Goods Issue",
  "Message": "Are you sure you want to post GI for delivery {DeliveryNo}?",
  "OKCaption": "Post",
  "CancelCaption": "Cancel",
  "OnOK": "/MyWarehouseApp/Actions/PostGoodsIssue.action",
  "OnCancel": "/MyWarehouseApp/Actions/DoNothing.action"
}
```

## Barcode Scan Action

```json
{
  "_Type": "MDK.BarcodeServiceAction",
  "_Name": "ScanHUBarcode",
  "BarcodeType": ["Code128", "QR", "PDF417"],
  "OnSuccess": "/MyWarehouseApp/Rules/ProcessScannedHU.js",
  "OnFailure": "/MyWarehouseApp/Actions/ShowScanError.action"
}
```

```javascript
// Rules/ProcessScannedHU.js — called after successful scan
export default function ProcessScannedHU(context) {
  const scannedValue = context.actionResults.ScanHUBarcode.data;
  // scannedValue = "00012345678901234567" (scanned barcode string)

  // Set the scanned value into a form field
  const pageProxy = context.getPageProxy();
  pageProxy.getControl('HUIDField').setValue(scannedValue);

  // Navigate to lookup page
  return context.executeAction('/MyWarehouseApp/Actions/NavToHUDetail.action');
}
```

## Set State Action (Temporary Storage)

```json
{
  "_Type": "MDK.SetStateAction",
  "_Name": "SaveSelectedDelivery",
  "Operation": "Set",
  "Key": "SelectedDeliveryNo",
  "Value": "{DeliveryNo}"
}
```

```javascript
// Read state in a rule:
const deliveryNo = context.getAppState().SelectedDeliveryNo;
```

## Application Action (Offline Sync)

```json
{
  "_Type": "MDK.ApplicationAction",
  "_Name": "SyncOfflineStore",
  "ApplicationAction": "OfflineDataSync",
  "OnSuccess": "/MyWarehouseApp/Actions/ShowSyncSuccess.action",
  "OnFailure": "/MyWarehouseApp/Actions/ShowSyncError.action"
}
```

## Chaining Actions (OnSuccess / OnFailure)

```json
{
  "_Type": "MDK.ODataAction",
  "_Name": "CreateGR",
  "Service": "...",
  "EntitySet": "GoodsReceiptSet",
  "RequestType": "POST",
  "Properties": { "..." },
  "OnSuccess": "/MyWarehouseApp/Actions/UploadOffline.action"
}
```

```json
{
  "_Type": "MDK.ApplicationAction",
  "_Name": "UploadOffline",
  "ApplicationAction": "OfflineDataUpload",
  "OnSuccess": "/MyWarehouseApp/Actions/NavToDeliveryList.action",
  "OnFailure": "/MyWarehouseApp/Actions/ShowUploadError.action"
}
```

## OData Error Action — Display Error Message

```json
{
  "_Type": "MDK.MessageAction",
  "_Name": "ShowODataError",
  "Title": "Error",
  "Message": "/MyWarehouseApp/Rules/GetODataErrorMsg.js",
  "OKCaption": "OK",
  "MessageType": "Error"
}
```

```javascript
// Rules/GetODataErrorMsg.js
export default function GetODataErrorMsg(context) {
  const err = context.actionResults.currentAction.error;
  if (err && err.message) {
    return err.message;
  }
  return 'An unexpected error occurred. Please try again.';
}
```
