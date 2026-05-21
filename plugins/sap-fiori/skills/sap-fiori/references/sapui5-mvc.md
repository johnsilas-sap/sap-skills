# SAPUI5 MVC Reference

## App Structure

```
webapp/
  ├── manifest.json          — App descriptor (routing, data sources, models)
  ├── index.html             — Entry point (bootstraps SAPUI5)
  ├── Component.js           — App component, initializes models and router
  ├── view/
  │     ├── App.view.xml     — Shell/nav container
  │     ├── Main.view.xml    — Main page view
  │     └── Detail.view.xml
  ├── controller/
  │     ├── BaseController.js
  │     ├── Main.controller.js
  │     └── Detail.controller.js
  ├── model/
  │     └── models.js        — Model factory
  ├── formatter/
  │     └── formatter.js     — Value formatters
  └── i18n/
        └── i18n.properties
```

## XML View

```xml
<!-- view/Main.view.xml -->
<mvc:View
  controllerName="com.mycompany.ewm.deliveryapp.controller.Main"
  xmlns:mvc="sap.ui.core.mvc"
  xmlns="sap.m"
  xmlns:f="sap.ui.layout.form"
  displayBlock="true">

  <Page title="{i18n>deliveryListTitle}" showNavButton="false">
    <headerContent>
      <Button icon="sap-icon://refresh" press=".onRefresh"/>
    </headerContent>

    <content>
      <!-- Filter bar -->
      <SearchField
        width="100%"
        placeholder="{i18n>searchPlaceholder}"
        search=".onSearch"
        liveChange=".onSearch"/>

      <!-- Table -->
      <Table
        id="deliveryTable"
        items="{/DeliverySet}"
        growing="true"
        growingThreshold="20"
        mode="SingleSelectMaster"
        selectionChange=".onSelectionChange">

        <headerToolbar>
          <Toolbar>
            <Title text="{i18n>deliveries}"/>
            <ToolbarSpacer/>
            <Button text="{i18n>postGI}" press=".onPostGI" type="Emphasized"/>
          </Toolbar>
        </headerToolbar>

        <columns>
          <Column><Text text="Delivery No."/></Column>
          <Column><Text text="Ship To"/></Column>
          <Column><Text text="Status"/></Column>
          <Column><Text text="Ship Date"/></Column>
        </columns>

        <items>
          <ColumnListItem
            type="Navigation"
            press=".onDeliveryPress">
            <cells>
              <Text text="{DeliveryNo}"/>
              <Text text="{ShipToName}"/>
              <ObjectStatus
                text="{Status}"
                state="{path:'Status', formatter:'.formatter.statusState'}"/>
              <Text text="{path:'ShipDate', type:'sap.ui.model.type.Date',
                           formatOptions:{pattern:'dd.MM.yyyy'}}"/>
            </cells>
          </ColumnListItem>
        </items>
      </Table>
    </content>
  </Page>
</mvc:View>
```

## Controller

```javascript
// controller/Main.controller.js
sap.ui.define([
  "./BaseController",
  "sap/ui/model/Filter",
  "sap/ui/model/FilterOperator",
  "sap/m/MessageToast",
  "sap/m/MessageBox",
  "../formatter/formatter"
], function(BaseController, Filter, FilterOperator, MessageToast, MessageBox, formatter) {
  "use strict";

  return BaseController.extend("com.mycompany.ewm.deliveryapp.controller.Main", {

    formatter: formatter,

    onInit: function() {
      // Called once when view is created
      this._oModel = this.getOwnerComponent().getModel();
    },

    onDeliveryPress: function(oEvent) {
      var sDeliveryNo = oEvent.getSource().getBindingContext().getProperty("DeliveryNo");
      this.getRouter().navTo("detail", { deliveryNo: encodeURIComponent(sDeliveryNo) });
    },

    onSearch: function(oEvent) {
      var sQuery = oEvent.getParameter("newValue") || oEvent.getParameter("query");
      var oTable = this.byId("deliveryTable");
      var oBinding = oTable.getBinding("items");

      var aFilters = [];
      if (sQuery) {
        aFilters.push(new Filter({
          filters: [
            new Filter("DeliveryNo",  FilterOperator.Contains, sQuery),
            new Filter("ShipToName",  FilterOperator.Contains, sQuery)
          ],
          and: false   // OR between filters
        }));
      }
      oBinding.filter(aFilters);
    },

    onRefresh: function() {
      this.getModel().refresh(true);   // force reload from server
    },

    onPostGI: function() {
      var oContext = this.byId("deliveryTable").getSelectedItem()?.getBindingContext();
      if (!oContext) {
        MessageToast.show("Please select a delivery first.");
        return;
      }
      var sDeliveryNo = oContext.getProperty("DeliveryNo");

      MessageBox.confirm("Post Goods Issue for delivery " + sDeliveryNo + "?", {
        onClose: function(sAction) {
          if (sAction === MessageBox.Action.OK) {
            this._postGI(oContext);
          }
        }.bind(this)
      });
    },

    _postGI: function(oContext) {
      var oModel = this.getModel();
      oModel.callFunction("/PostGoodsIssue", {
        method: "POST",
        urlParameters: { DeliveryNo: oContext.getProperty("DeliveryNo") },
        success: function(oData) {
          MessageToast.show("Goods Issue posted successfully.");
          oModel.refresh();
        },
        error: function(oError) {
          MessageBox.error("Failed to post GI: " + oError.message);
        }
      });
    }
  });
});
```

## BaseController

```javascript
// controller/BaseController.js
sap.ui.define(["sap/ui/core/mvc/Controller", "sap/ui/core/routing/History"],
function(Controller, History) {
  "use strict";
  return Controller.extend("com.mycompany.ewm.deliveryapp.controller.BaseController", {

    getRouter:    function() { return this.getOwnerComponent().getRouter(); },
    getModel:     function(sName) { return this.getView().getModel(sName); },
    setModel:     function(oModel, sName) { return this.getView().setModel(oModel, sName); },
    getResourceBundle: function() {
      return this.getOwnerComponent().getModel("i18n").getResourceBundle();
    },

    onNavBack: function() {
      var oHistory = History.getInstance();
      var sPreviousHash = oHistory.getPreviousHash();
      if (sPreviousHash !== undefined) {
        window.history.go(-1);
      } else {
        this.getRouter().navTo("main", {}, true);
      }
    }
  });
});
```

## OData V2 Model Setup

```javascript
// Component.js
sap.ui.define(["sap/ui/core/UIComponent", "sap/ui/model/odata/v2/ODataModel"],
function(UIComponent, ODataModel) {
  return UIComponent.extend("com.mycompany.ewm.deliveryapp.Component", {
    metadata: { manifest: "json" },

    init: function() {
      UIComponent.prototype.init.apply(this, arguments);
      this.getRouter().initialize();
    }
  });
});
```

```json
// manifest.json — model configuration
"sap.ui5": {
  "models": {
    "": {
      "dataSource": "mainService",
      "settings": {
        "defaultBindingMode": "TwoWay",
        "useBatch": true,
        "defaultCountMode": "Inline",
        "refreshAfterChange": false
      }
    },
    "i18n": {
      "type": "sap.ui.model.resource.ResourceModel",
      "settings": { "bundleName": "com.mycompany.ewm.deliveryapp.i18n.i18n" }
    }
  }
}
```

## Formatter

```javascript
// formatter/formatter.js
sap.ui.define([], function() {
  "use strict";
  return {
    statusState: function(sStatus) {
      var mMap = {
        "Open":       "Information",
        "InProcess":  "Warning",
        "Completed":  "Success",
        "Blocked":    "Error"
      };
      return mMap[sStatus] || "None";
    },

    weightText: function(fWeight, sUOM) {
      if (!fWeight) return "";
      return parseFloat(fWeight).toFixed(2) + " " + (sUOM || "");
    }
  };
});
```

## Routing (manifest.json)

```json
"routing": {
  "config": {
    "routerClass": "sap.m.routing.Router",
    "viewType": "XML",
    "viewPath": "com.mycompany.ewm.deliveryapp.view",
    "controlId": "idAppControl",
    "controlAggregation": "pages",
    "bypassed": { "target": "notFound" }
  },
  "routes": [
    { "pattern": "",               "name": "main",   "target": "main" },
    { "pattern": "detail/{deliveryNo}", "name": "detail", "target": "detail" }
  ],
  "targets": {
    "main":     { "viewId": "main",     "viewName": "Main",     "viewLevel": 1 },
    "detail":   { "viewId": "detail",   "viewName": "Detail",   "viewLevel": 2 },
    "notFound": { "viewId": "notFound", "viewName": "NotFound", "viewLevel": 3 }
  }
}
```

## Fragment (Reusable Dialog)

```xml
<!-- view/fragment/PostGIDialog.fragment.xml -->
<core:FragmentDefinition xmlns="sap.m" xmlns:core="sap.ui.core">
  <Dialog id="postGIDialog" title="Post Goods Issue" type="Message">
    <content>
      <Input id="actualQtyInput" placeholder="Actual Quantity" type="Number"/>
    </content>
    <buttons>
      <Button text="Post"   type="Emphasized" press=".onConfirmPostGI"/>
      <Button text="Cancel"                   press=".onCancelDialog"/>
    </buttons>
  </Dialog>
</core:FragmentDefinition>
```

```javascript
// Load fragment in controller:
onShowDialog: function() {
  if (!this._oDialog) {
    this._oDialog = this.loadFragment({ name: "com.mycompany.ewm.deliveryapp.view.fragment.PostGIDialog" });
  }
  this._oDialog.then(function(oDialog) { oDialog.open(); });
}
```
