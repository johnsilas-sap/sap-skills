# MDK Rules (JavaScript) Reference

## Rule Basics

Rules are JavaScript files in the `/Rules` folder. Every rule exports a default function that receives the MDK `context` object.

```javascript
// Rules/MyRule.js
export default function MyRule(context) {
  // return a value, or a Promise that resolves to a value
  return 'Hello World';
}
```

Rules can:
- Return a value used in page/control bindings
- Return a Promise for async operations
- Call `context.executeAction()` to trigger actions
- Read and write OData data
- Access page controls and their values

## Context API

### Reading Binding Data

```javascript
export default function GetStatusColor(context) {
  // context.binding = the current OData entity bound to the page/cell
  const status = context.binding.Status;

  switch (status) {
    case 'Open':       return 'Informative';
    case 'InProcess':  return 'Critical';
    case 'Completed':  return 'Positive';
    case 'Blocked':    return 'Negative';
    default:           return 'None';
  }
}
```

### Reading Control Values (on same page)

```javascript
export default function ValidateForm(context) {
  const pageProxy = context.getPageProxy();

  const qty = pageProxy.getControl('QuantityField').getValue();
  const batch = pageProxy.getControl('BatchField').getValue();
  const bin = pageProxy.getControl('DestBinField').getValue();

  if (!qty || qty <= 0) {
    return context.executeAction({
      '_Type': 'MDK.MessageAction',
      'Message': 'Quantity is required',
      'Title': 'Validation Error'
    });
  }

  return context.executeAction('/MyWarehouseApp/Actions/PostConfirmation.action');
}
```

### Reading App-Level Data

```javascript
export default function GetWarehouseFilter(context) {
  const appData = context.getAppData();
  const lgnum = appData.WarehouseNo || '1710';
  return `$filter=WarehouseNo eq '${lgnum}' and Status eq 'Open'`;
}
```

### Getting Client Data (Device Info)

```javascript
export default function GetDeviceInfo(context) {
  const clientData = context.getClientData();
  return {
    deviceType: clientData.deviceType,     // ios / android
    screenSize: clientData.screenSize,
    userId: clientData.userId
  };
}
```

## OData Read in a Rule

```javascript
// Rules/GetDeliveryItems.js
export default function GetDeliveryItems(context) {
  const deliveryNo = context.binding.DeliveryNo;

  const serviceUrl = '/MyWarehouseApp/Services/EWM_SERVICE.service';
  const entitySet = 'DeliveryItemSet';
  const query = `$filter=DeliveryNo eq '${deliveryNo}'`;

  return context.read(serviceUrl, entitySet, [], query)
    .then((result) => {
      // result is an ODataReadResult object
      return result.length;    // Return item count as binding value
    })
    .catch((err) => {
      console.error('Read failed:', err.message);
      return 0;
    });
}
```

## OData Create in a Rule

```javascript
// Rules/CreateTransferOrder.js
export default function CreateTransferOrder(context) {
  const pageProxy = context.getPageProxy();
  const deliveryNo = pageProxy.getControl('DeliveryNoField').getValue();
  const qty = parseFloat(pageProxy.getControl('QtyField').getValue());

  const serviceUrl = '/MyWarehouseApp/Services/EWM_SERVICE.service';
  const entitySet = 'TransferOrderSet';

  const payload = {
    DeliveryNo: deliveryNo,
    Quantity:   qty,
    Created:    new Date().toISOString()
  };

  return context.create(serviceUrl, entitySet, payload)
    .then(() => context.executeAction('/MyWarehouseApp/Actions/NavToTOList.action'))
    .catch((err) => context.executeAction({
      '_Type': 'MDK.MessageAction',
      'Title': 'Error',
      'Message': err.message || 'Failed to create transfer order'
    }));
}
```

## OData Update in a Rule

```javascript
// Rules/ConfirmWarehouseTask.js
export default function ConfirmWarehouseTask(context) {
  const readLink = context.binding['@odata.readLink'];
  const actualQty = context.getPageProxy().getControl('ActualQtyField').getValue();

  const serviceUrl = '/MyWarehouseApp/Services/EWM_SERVICE.service';

  return context.update(serviceUrl, 'WarehouseTaskSet', readLink, {
    Confirmed: true,
    ActualQuantity: parseFloat(actualQty)
  })
  .then(() => context.executeAction('/MyWarehouseApp/Actions/ShowConfirmSuccess.action'))
  .catch((err) => context.executeAction('/MyWarehouseApp/Actions/ShowError.action'));
}
```

## Navigation in a Rule

```javascript
// Rules/NavigateBasedOnType.js
export default function NavigateBasedOnType(context) {
  const docType = context.binding.DocumentType;

  let targetPage;
  if (docType === 'INBOUND') {
    targetPage = '/MyWarehouseApp/Pages/InboundDelivery.page';
  } else if (docType === 'OUTBOUND') {
    targetPage = '/MyWarehouseApp/Pages/OutboundDelivery.page';
  } else {
    targetPage = '/MyWarehouseApp/Pages/GenericDocument.page';
  }

  return context.executeAction({
    '_Type': 'MDK.NavigationAction',
    'PageToOpen': targetPage,
    'TransitionType': 'Push'
  });
}
```

## Setting Page Control Values Programmatically

```javascript
// Rules/PopulateFormFromScan.js
export default function PopulateFormFromScan(context) {
  const scanned = context.actionResults.ScanHUBarcode.data;
  // scanned = "HU00001234567"

  const pageProxy = context.getPageProxy();

  // Set control values
  pageProxy.getControl('HUIDField').setValue(scanned);
  pageProxy.getControl('StatusField').setValue('Scanned');

  // Enable a previously disabled button
  pageProxy.getControl('ConfirmButton').setVisible(true);

  return Promise.resolve();
}
```

## Using Action Results in Rules

```javascript
// Rule called OnSuccess of a barcode scan action
export default function AfterScan(context) {
  // Access results of the previous action
  const scanResult = context.actionResults.ScanHUBarcode;

  if (scanResult && scanResult.data) {
    const huId = scanResult.data;

    // OData lookup for the scanned HU
    return context.read(
      '/MyWarehouseApp/Services/EWM_SERVICE.service',
      'HandlingUnitSet',
      [],
      `$filter=HUId eq '${huId}'`
    ).then((result) => {
      if (result.length === 0) {
        return context.executeAction({
          '_Type': 'MDK.MessageAction',
          'Title': 'Not Found',
          'Message': `Handling unit ${huId} not found`
        });
      }
      // Navigate to HU detail with the found entity as binding
      return context.executeAction({
        '_Type': 'MDK.NavigationAction',
        'PageToOpen': '/MyWarehouseApp/Pages/HUDetail.page',
        'TransitionType': 'Push'
      });
    });
  }
}
```

## Dynamic OData Filter from Rule

```javascript
// Rules/GetTaskFilterQuery.js
export default function GetTaskFilterQuery(context) {
  const pageProxy = context.getPageProxy();
  const segCtrl = pageProxy.getControl('StatusFilter');
  const selectedStatus = segCtrl ? segCtrl.getValue() : 'Open';
  const lgnum = context.getAppData().WarehouseNo || '1710';

  return `$filter=WarehouseNo eq '${lgnum}' and Status eq '${selectedStatus}'`
       + `&$orderby=Priority desc,CreatedAt asc`
       + `&$top=100`;
}
```

## Error Handling Pattern

```javascript
export default function SafeOperation(context) {
  return someAsyncOperation(context)
    .then((result) => {
      return context.executeAction('/MyWarehouseApp/Actions/OnSuccess.action');
    })
    .catch((error) => {
      console.error('Operation failed:', JSON.stringify(error));
      return context.executeAction({
        '_Type': 'MDK.MessageAction',
        'Title': 'Error',
        'Message': error.message || 'An error occurred',
        'MessageType': 'Error'
      });
    });
}
```
