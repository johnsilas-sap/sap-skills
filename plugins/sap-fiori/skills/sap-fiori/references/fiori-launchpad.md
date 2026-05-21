# Fiori Launchpad (FLP) Reference

## FLP Architecture

```
FLP Shell (renderer)
  ├── Catalogs    → groups of tiles available to assign to roles
  ├── Groups      → visible tile groups on the launchpad homepage
  ├── Roles       → assign catalogs to business roles
  └── Tiles       → each tile = app launcher with intent navigation

Intent = Semantic Object + Action
  e.g.: DeliveryManagement#displayList
        WarehouseTask#confirm
```

## Transactions

| Transaction | Purpose |
|---|---|
| `/n/UI2/FLPD_CONF` | FLP Designer — create tiles, groups, catalogs |
| `/n/UI2/FLP` | Open FLP (test/launch apps) |
| `/n/UI2/FLPCM` | FLP Cache Manager (clear cache) |
| `/n/IWFND/MAINT_SERVICE` | Register OData service in Gateway |
| `SICF` | Activate ICF services for Fiori apps |
| `SE80` | ABAP repository — find BSP/UI5 apps |

## Creating a Tile (FLP Designer)

```
/n/UI2/FLPD_CONF → Fiori Configuration Cockpit

Step 1: Create Target Mapping
  Semantic Object:  DeliveryManagement
  Action:           displayList
  Application Type: SAPUI5
  Application ID:   com.mycompany.ewm.deliveryapp   (BSP app name or UI5 repository ID)
  Component:        (leave blank for Fiori Elements apps — auto-detected)
  URL:              /sap/bc/ui5_ui5/sap/z_ewm_deliv_app/ (BSP path)
  OR URL:           /sap/bc/ui2/nwbc/#Action-toMeArea  (for standard SAP apps)

Step 2: Create Tile
  Catalog:   Z_EWM_LOGISTICS_CATALOG
  Tile type: Static / Dynamic / Custom
  
  Static tile:
    Title:     Delivery List
    Subtitle:  Open deliveries
    Icon:      sap-icon://shipping-status
    Intent:    DeliveryManagement#displayList
  
  Dynamic tile (shows live count):
    Title:     Open Tasks
    Service URL: /sap/opu/odata/sap/ZEWM_TASKS_SRV/TaskCountSet?$filter=Status eq 'Open'
    Number Field: Count
    Unit:      Tasks

Step 3: Assign Catalog to Role
  Role:    Z_EWM_WAREHOUSE_MANAGER
  Catalog: Z_EWM_LOGISTICS_CATALOG

Step 4: Assign Role to User
  SU01 / PFCG → Assign role → user inherits tile catalog
```

## Target Mapping Parameters

```
Target Mapping → Default Values:
  Parameter Name: WarehouseNo
  Default Value:  1710
  " Passed as startup parameter to the app on launch

  Parameter Name: FilterStatus
  Default Value:  Open
```

```javascript
// App reads startup parameters:
var oComponentData = this.getOwnerComponent().getComponentData();
var oStartupParams = oComponentData && oComponentData.startupParameters;
var sWarehouseNo = oStartupParams && oStartupParams.WarehouseNo && oStartupParams.WarehouseNo[0];
```

## Intent-Based Navigation

### Navigate to Another App (Cross-App)

```javascript
// In controller — navigate to another Fiori app by intent
var oCrossAppNav = sap.ushell && sap.ushell.Container.getService("CrossApplicationNavigation");

oCrossAppNav.toExternal({
  target: {
    semanticObject: "WarehouseTask",
    action: "confirm"
  },
  params: {
    DeliveryNo: sDeliveryNo,
    WarehouseNo: "1710"
  }
});
```

### Navigate Back to Launchpad

```javascript
oCrossAppNav.toExternal({ target: { shellHash: "#" } });
// OR:
sap.ushell.Container.getService("CrossApplicationNavigation").toExternal({
  target: { shellHash: "#Shell-home" }
});
```

### Check if Target Exists (Before Navigating)

```javascript
oCrossAppNav.isIntentSupported([{
  semanticObject: "DeliveryManagement",
  action: "displayDetail"
}]).done(function(oResult) {
  if (oResult["#DeliveryManagement-displayDetail"].supported) {
    // safe to navigate
  }
});
```

## Registering a UI5 App (BSP) in ABAP

```
SE80 → Repository → Web Applications → New BSP Application
  Application Name:  Z_EWM_DELIV_APP
  Description:       EWM Delivery Management
  Default Page:      index.html

  Upload project:
    Right-click app → Upload/Download → Upload (zip of webapp folder)

SICF → /sap/bc/ui5_ui5/sap/z_ewm_deliv_app → Activate
/n/IWFND/MAINT_SERVICE → Add OData service if not registered
```

## Registering a UI5 App (Cloud via ABAP Repository)

```bash
# Using Fiori Tools / @sap/ux-ui5-tooling deploy
npx fiori deploy --config .fiorirc.json

# .fiorirc.json:
{
  "endpoint": "https://my-s4.ondemand.com",
  "credentials": { "sap-client": "100" },
  "app": {
    "name": "Z_EWM_DELIV_APP",
    "package": "ZEWM_FIORI",
    "transport": "DE1K900123"
  }
}
```

## FLP Configuration in BTP (Cloud Launchpad / Work Zone)

```
BTP Cockpit → Instances → SAP Build Work Zone
  → Site Manager → Create Site

  Site Settings:
    Site Name:   EWM Operations Portal
    Theme:       Horizon / Quartz

  Add Business Apps:
    → Import from content repository
    → Or: Add manually with URL + intent
  
  Pages:
    → Create page with sections
    → Add tiles/cards per section

  Roles:
    → Assign site roles to IAS groups
```

## Common SAP Standard Fiori App IDs (EWM/TM)

| App ID | Title | Semantic Object |
|---|---|---|
| `F0862` | Monitor Delivery Processing (EWM) | `DeliveryProcessing` |
| `F1344` | Create Outbound Delivery Order | `OutboundDelivery` |
| `F1797` | Manage Warehouse Tasks | `WarehouseTask` |
| `F3173` | Process Freight Orders | `FreightOrder` |
| `F3174` | Carrier Selection | `CarrierSelection` |
| `F0984` | Track Shipments | `Shipment` |

```
" Find app details:
/n/UI2/FLPD_CONF → App Finder → search by title
OR: SAP Fiori Apps Library: fioriappslibrary.hana.ondemand.com
```

## FLP Personalization

Users can personalize their launchpad — add/move tiles, create groups. Admins can lock layouts:

```
/n/UI2/FLPD_CONF → Configuration → Personalization Settings
  Allow users to add tiles: On / Off
  Allow users to create groups: On / Off
  Lock page layout: On  (prevents user personalization)
```
