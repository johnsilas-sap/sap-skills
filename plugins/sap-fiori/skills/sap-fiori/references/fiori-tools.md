# Fiori Tools Reference

## Fiori Tools Extension Pack

Install in BAS or VS Code:
- `@sap/generator-fiori` — Yeoman project generator
- `SAP Fiori tools - Application Generator`
- `SAP Fiori tools - Guided Development`
- `SAP Fiori tools - Service Modeler`
- `SAP Fiori tools - XML Annotation Language Server`

## Generating a New App (BAS)

```
BAS → View → Command Palette → Fiori: Open Application Generator

  Step 1: Template Selection
    Floorplan: List Report Page / Object Page / Worklist / ALP / Overview Page
    
  Step 2: Data Source
    Connect to an OData Service (system destination)
    Upload a Metadata Document (local .xml file — for mock)
    Use a Local CAP Project
    None (start empty)

  Step 3: Entity Selection (if OData connected)
    Main Entity:          A_OutbDeliveryHeader
    Navigation Entity:    to_DeliveryDocumentItem
    Automatically add table columns: ✓

  Step 4: Project Attributes
    Module Name:    ewm-delivery-app
    Application title: EWM Delivery Management
    Namespace:      com.mycompany.ewm
    Project folder: /home/user/projects/
    Add FLP config: ✓ (adds flp-config.json)
    Semantic Object: DeliveryManagement
    Action:          displayList

  → Generates complete runnable app
```

## CLI Generator (npx)

```bash
# Install yeoman + SAP generator
npm install -g yo @sap/generator-fiori

# Generate app interactively
yo @sap/fiori

# Generate programmatically (CI/CD)
yo @sap/fiori \
  --applicationtype SAP Fiori elements \
  --floorplan lrop \                     # lrop=List Report, op=Object Page, alp=ALP
  --datasource uri \
  --odata-uri https://my-s4.ondemand.com/sap/opu/odata4/... \
  --entity DeliverySet \
  --projectname ewm-delivery-app \
  --namespace com.mycompany.ewm
```

## Local Development and Preview

```bash
# Install dependencies
npm install

# Start local dev server with mock data
npm start
# Opens: http://localhost:8080/index.html

# Start with real backend (requires BTP destination or credentials)
npm run start-local    # connects to local system via proxy
npm run start-mock     # uses local service mock (no backend)
```

### fiori.config.json (proxy to real backend)

```json
{
  "middleware": [{
    "name": "fiori-tools-apimiddleware",
    "afterMiddleware": "compression",
    "configuration": {
      "backend": [{
        "path": "/sap",
        "url": "https://my-s4.ondemand.com",
        "client": "100",
        "authenticationType": "basic",
        "credentials": {
          "username": "myuser",
          "password": "mypass"
        }
      }]
    }
  }]
}
```

### ui5.yaml (UI5 tooling config)

```yaml
specVersion: "3.0"
metadata:
  name: ewm-delivery-app
type: application
framework:
  name: SAPUI5
  version: "1.132.0"
  libraries:
    - name: sap.m
    - name: sap.ui.core
    - name: sap.fe.templates
    - name: sap.ushell
server:
  customMiddleware:
    - name: fiori-tools-apimiddleware
      afterMiddleware: compression
      configuration:
        backend:
          - path: /sap
            url: https://my-s4.ondemand.com
```

## Annotation Modeler (VS Code / BAS)

```
BAS → Open any .cds or annotations.xml file
→ Right-click → Open with Guided Development
→ Annotation Modeler opens in panel

Guided Development:
  Search: "Add a UI.LineItem"
  → Provides step-by-step wizard
  → Generates annotation CDS/XML automatically
  → No manual annotation syntax needed
```

## Deployment to ABAP System

```bash
# Deploy to ABAP BSP repository
npm run deploy

# Or using fiori deploy CLI:
npx @sap/ux-ui5-tooling deploy \
  --url https://my-s4.ondemand.com \
  --client 100 \
  --username myuser \
  --password mypass \
  --name Z_EWM_DELIV_APP \
  --package ZEWM_FIORI \
  --transport DE1K900123

# .fiorirc.json (save config for repeated deploys):
{
  "endpoint": "https://my-s4.ondemand.com",
  "app": {
    "name": "Z_EWM_DELIV_APP",
    "package": "ZEWM_FIORI",
    "transport": "DE1K900123"
  }
}
```

## Deployment to BTP Cloud Foundry

```bash
# Build dist folder
npm run build

# Deploy using MTA (Multi-Target Application)
mbt build
cf deploy mta_archives/ewm-delivery-app_1.0.0.mtar

# OR: using fiori deploy for HTML5 Repository
npx fiori deploy --target abap  # ABAP
npx fiori deploy --target cf    # Cloud Foundry HTML5 repo
```

### mta.yaml for BTP Fiori App

```yaml
_schema-version: "3.1"
ID: ewm-delivery-app
version: 1.0.0
modules:
  - name: ewm-delivery-app
    type: html5
    path: .
    build-parameters:
      build-result: dist
      builder: custom
      commands:
        - npm install
        - npm run build
    parameters:
      disk-quota: 256M
      memory: 256M
resources:
  - name: ewm-delivery-app_html5_repo
    type: org.cloudfoundry.managed-service
    parameters:
      service: html5-apps-repo
      service-plan: app-host
```

## Mock Server (Offline Development)

```bash
# Generate mock data from metadata
npx @sap/ux-ui5-tooling mockserver generate \
  --metadata path/to/$metadata.xml \
  --output webapp/localService/mockdata/

# Mock data files: DeliverySet.json, ItemSet.json, etc.
```

```json
// webapp/localService/mockdata/DeliverySet.json
[
  { "DeliveryNo": "0180000001", "Status": "Open", "ShipToName": "ACME Corp", "ShipDate": "/Date(1748736000000)/" },
  { "DeliveryNo": "0180000002", "Status": "InProcess", "ShipToName": "Beta GmbH", "ShipDate": "/Date(1748822400000)/" }
]
```

## Build for Production

```bash
npm run build
# Output: dist/ folder ready to deploy
# Minified JS, merged CSS, optimized for production
```

## Fiori Tools Service Modeler

```
BAS/VS Code → Fiori: Show Service Modeler
  → Visual display of OData service entity model
  → Shows entity types, navigation properties, associations
  → Click entity → see all properties with types
  → Useful for understanding unfamiliar OData services before annotating
```
