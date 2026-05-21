# Navigation and Routing Reference

## Router Configuration (manifest.json)

```json
"routing": {
  "config": {
    "routerClass": "sap.m.routing.Router",
    "viewType": "XML",
    "viewPath": "com.mycompany.ewm.deliveryapp.view",
    "controlId": "idAppControl",
    "controlAggregation": "pages",
    "bypassed": { "target": "notFound" },
    "async": true
  },
  "routes": [
    { "pattern": "",                            "name": "list",    "target": "list"   },
    { "pattern": "delivery/{deliveryNo}",       "name": "detail",  "target": "detail" },
    { "pattern": "delivery/{deliveryNo}/gi",    "name": "gi",      "target": "gi"     }
  ],
  "targets": {
    "list":     { "viewId": "list",     "viewName": "DeliveryList",   "viewLevel": 1 },
    "detail":   { "viewId": "detail",   "viewName": "DeliveryDetail", "viewLevel": 2 },
    "gi":       { "viewId": "gi",       "viewName": "GoodsIssue",     "viewLevel": 3 },
    "notFound": { "viewId": "notFound", "viewName": "NotFound",       "viewLevel": 1 }
  }
}
```

## Router Initialization

```javascript
// Component.js — initialize once
init: function() {
  UIComponent.prototype.init.apply(this, arguments);
  this.getRouter().initialize();   // attaches hash change listener
}
```

## Navigating With Parameters

```javascript
// Forward navigation — encode special characters in route params
onDeliveryPress: function(oEvent) {
  var sDeliveryNo = oEvent.getSource().getBindingContext().getProperty("DeliveryNo");
  this.getRouter().navTo("detail", {
    deliveryNo: encodeURIComponent(sDeliveryNo)   // handles "/" in keys
  });
},

// Navigate to GI form from detail
onOpenGI: function() {
  var sDeliveryNo = this.getView().getBindingContext().getProperty("DeliveryNo");
  this.getRouter().navTo("gi", {
    deliveryNo: encodeURIComponent(sDeliveryNo)
  });
},

// Replace current history entry (no back button to this step)
this.getRouter().navTo("list", {}, true);   // true = replace hash
```

## Receiving Route Parameters

```javascript
// controller/DeliveryDetail.controller.js
onInit: function() {
  var oRouter = this.getRouter();
  oRouter.getRoute("detail").attachPatternMatched(this._onRouteMatched, this);
},

_onRouteMatched: function(oEvent) {
  var sDeliveryNo = decodeURIComponent(oEvent.getParameter("arguments").deliveryNo);

  // Bind view to OData entity
  var sPath = "/DeliverySet('" + sDeliveryNo + "')";
  this.getView().bindElement({
    path: sPath,
    parameters: {
      expand: "to_Items"
    },
    events: {
      dataRequested: function() { this.getView().setBusy(true);  }.bind(this),
      dataReceived:  function() { this.getView().setBusy(false); }.bind(this)
    }
  });
},
```

## Back Navigation

```javascript
// controller/BaseController.js
onNavBack: function() {
  var oHistory = sap.ui.core.routing.History.getInstance();
  var sPreviousHash = oHistory.getPreviousHash();

  if (sPreviousHash !== undefined) {
    window.history.go(-1);
  } else {
    // No history — go to app root (e.g., user opened deep link directly)
    this.getRouter().navTo("list", {}, true);
  }
}
```

```xml
<!-- In view — wire nav button -->
<Page showNavButton="true" navButtonPress=".onNavBack" title="Delivery Detail">
```

## Router Events

```javascript
// Listen on any route match (e.g., for Shell title update)
this.getRouter().attachRouteMatched(function(oEvent) {
  var sRouteName = oEvent.getParameter("name");
  // update breadcrumb, shell title, etc.
}, this);

// Listen for bypassed route (not found)
this.getRouter().attachBypassed(function(oEvent) {
  var sHash = oEvent.getParameter("hash");
  sap.m.MessageToast.show("No route found for hash: " + sHash);
}, this);
```

## Cross-App Navigation (FLP)

```javascript
// Navigate to another Fiori app by semantic intent
var oCrossNav = sap.ushell.Container.getService("CrossApplicationNavigation");

// Basic cross-app navigate
oCrossNav.toExternal({
  target: {
    semanticObject: "WarehouseTask",
    action:         "confirm"
  },
  params: {
    DeliveryNo:  sDeliveryNo,
    WarehouseNo: "1710"
  }
});

// Navigate back to FLP home
oCrossNav.toExternal({ target: { shellHash: "#Shell-home" } });

// Check intent is supported before navigating
oCrossNav.isIntentSupported([{
  semanticObject: "FreightOrder",
  action: "display"
}]).done(function(oResult) {
  if (oResult["#FreightOrder-display"].supported) {
    oCrossNav.toExternal({ target: { semanticObject: "FreightOrder", action: "display" }, params: { FreightOrderId: sId } });
  } else {
    sap.m.MessageToast.show("App not available.");
  }
});
```

## Reading Startup Parameters (FLP → App)

```javascript
// Component.js or first controller onInit
onInit: function() {
  var oComponentData = this.getOwnerComponent().getComponentData();
  var oStartupParams = oComponentData && oComponentData.startupParameters;

  // FLP target mapping passes: WarehouseNo=1710, FilterStatus=Open
  var sWarehouseNo  = oStartupParams && oStartupParams.WarehouseNo  && oStartupParams.WarehouseNo[0];
  var sFilterStatus = oStartupParams && oStartupParams.FilterStatus && oStartupParams.FilterStatus[0];

  if (sFilterStatus) {
    // Pre-apply filter on the list binding
    var oTable   = this.byId("deliveryTable");
    var oBinding = oTable.getBinding("items");
    oBinding.filter([ new Filter("Status", FilterOperator.EQ, sFilterStatus) ]);
  }
}
```

## Deep Linking

```
FLP deep link pattern:
  https://my-s4.ondemand.com/sap/bc/ui2/flp#DeliveryManagement-displayList

With parameters:
  #DeliveryManagement-displayDetail?DeliveryNo=0180000001

Requires:
  1. Target mapping for semantic object + action exists in FLP
  2. Route pattern matches parameter (e.g., "delivery/{deliveryNo}")
  3. App handles direct URL load (no prior history) — use navTo("list",{},true) fallback
```

## Navigation Patterns — Logistics Apps

### Delivery List → Detail → GI Confirmation

```
DeliveryList (pattern: "")
  → press row → navTo("detail", { deliveryNo })
  
DeliveryDetail (pattern: "delivery/{deliveryNo}")
  → bindElement to /DeliverySet('{deliveryNo}') with expand=to_Items
  → press "Post GI" → navTo("gi", { deliveryNo })
  → onNavBack → history.go(-1)

GoodsIssue (pattern: "delivery/{deliveryNo}/gi")
  → POST to OData function import
  → success → navTo("list", {}, true)   // clear history
  → onNavBack → history.go(-1)
```

### Handling Unit Scan → Task Confirm (MDK-Style in UI5)

```javascript
// Read scanned HU from route param
_onRouteMatched: function(oEvent) {
  var sHUId = decodeURIComponent(oEvent.getParameter("arguments").huId);
  // Read open tasks for HU
  var oModel = this.getModel();
  oModel.read("/WarehouseTaskSet", {
    filters: [ new Filter("HandlingUnitId", FilterOperator.EQ, sHUId) ],
    success: function(oData) {
      // bind first open task to form
      var sTaskPath = oModel.createKey("/WarehouseTaskSet", { TaskId: oData.results[0].TaskId });
      this.getView().bindElement(sTaskPath);
    }.bind(this)
  });
}
```

## Programmatic URL / Hash Reading

```javascript
// Read current hash (for sharing deep link)
var sHash = sap.ui.core.routing.HashChanger.getInstance().getHash();

// Parse intent from hash (inside FLP)
var oParser = sap.ushell.Container.getService("URLParsing");
var oIntent = oParser.parseShellHash(sHash);
// oIntent.semanticObject, oIntent.action, oIntent.params
```

## Routing in Fiori Elements (manifest.json)

```json
"sap.ui5": {
  "routing": {
    "routes": [
      {
        "pattern": ":?query:",
        "name": "DeliveryList",
        "target": "DeliveryList"
      },
      {
        "pattern": "Deliveries({key}):?query:",
        "name": "DeliveryObjectPage",
        "target": "DeliveryObjectPage"
      }
    ],
    "targets": {
      "DeliveryList": {
        "type": "Component",
        "id": "DeliveryList",
        "name": "sap.fe.templates.ListReport",
        "options": {
          "settings": {
            "entitySet": "Deliveries",
            "navigation": {
              "Deliveries": { "detail": { "route": "DeliveryObjectPage" } }
            }
          }
        }
      },
      "DeliveryObjectPage": {
        "type": "Component",
        "id": "DeliveryObjectPage",
        "name": "sap.fe.templates.ObjectPage",
        "options": {
          "settings": {
            "entitySet": "Deliveries",
            "navigation": {
              "to_Items": { "detail": { "route": "ItemObjectPage" } }
            }
          }
        }
      }
    }
  }
}
```
