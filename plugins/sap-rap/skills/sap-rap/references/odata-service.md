# RAP OData Service Reference

## Service Definition

```abap
" ADT: New → Other → Service Definition
" File: ZSD_Delivery.srvd

define service ZSD_Delivery {
  expose ZC_Delivery      as Delivery;
  expose ZC_DeliveryItem  as DeliveryItem;
  expose ZC_Carrier       as Carrier;          " read-only reference entity

  // Expose value help entity (for Status dropdown):
  expose ZVH_DeliveryStatus as DeliveryStatus;
}
```

## Service Binding

```abap
" ADT: New → Service Binding
" Name: ZSB_DELIVERY_O4    (convention: O4 = OData V4, O2 = OData V2)
" Binding type: OData V4 - UI (for Fiori Elements)
"               OData V4 - Web API (for API consumption)
"               OData V2 - UI (for backwards-compatible Fiori)

" Service Binding links to Service Definition and assigns URL path
" After creation: click 'Activate' button in ADT
" Published URL: /sap/opu/odata4/<namespace>/<service>/<version>/
"   e.g.: /sap/opu/odata4/sap/zsd_delivery/srvd_a2x/sap/zsd_delivery/0001/

" Test in ADT: click 'Preview' → opens Fiori preview in browser
```

## Typical Naming Convention

```
Interface CDS:        ZI_Delivery, ZI_DeliveryItem
Projection CDS:       ZC_Delivery, ZC_DeliveryItem
Behavior Def:         ZI_Delivery (interface), ZC_Delivery (projection)
Behavior Impl class:  ZBP_Delivery
Service Definition:   ZSD_Delivery
Service Binding:      ZSB_Delivery_O4 (OData V4 UI)
                      ZSB_Delivery_O4_API (OData V4 Web API)
Draft Table:          ZEWM_DELIVERY_D
```

## OData V4 URL Patterns (RAP)

```
Base:   /sap/opu/odata4/sap/zsd_delivery/srvd/sap/zsd_delivery/0001/

Read collection:
  GET .../Deliveries

Read with filter + expand:
  GET .../Deliveries?$filter=Status eq 'O'&$expand=to_Items&$select=DeliveryNo,Status,ShipDate

Read single:
  GET .../Deliveries('0180000001')

Read navigation:
  GET .../Deliveries('0180000001')/to_Items

Create:
  POST .../Deliveries
  { "ShipToName": "ACME", "ShipDate": "2026-06-01" }

Update:
  PATCH .../Deliveries('0180000001')
  { "Status": "P" }

Delete:
  DELETE .../Deliveries('0180000001')

Bound action:
  POST .../Deliveries('0180000001')/ZI_Delivery.PostGoodsIssue

Unbound/static action:
  POST .../Deliveries/ZI_Delivery.CreateFromPO
  { "PONumber": "4500000123" }

Function import:
  GET .../Deliveries/ZI_Delivery.GetOpenDeliveryCount()

$batch:
  POST .../$batch    (multiple operations in one round-trip)
```

## Authorization (PFCG + DCL)

```abap
" DCL (Data Control Language) — restrict READ access at CDS level
" File: ZI_Delivery.dcl

@EndUserText.label: 'Delivery access control'
@MappingRole: true
define role ZI_Delivery {
  grant select on ZI_Delivery
    where ( WarehouseNo ) = aspect pfcg_auth( ZEWM_WH, LGNUM, ACTVT = '03' );
}

" This restricts GET /Deliveries to records where WarehouseNo is
" in the user's ZEWM_WH authorization role assignment.

" For write operations, implement authority-check in handler class:
METHOD create_delivery.
  AUTHORITY-CHECK OBJECT 'ZEWM_DLV' ID 'ACTVT' FIELD '01'.   " 01 = create
  IF sy-subrc <> 0.
    APPEND VALUE #(
      %cid = <create>-%cid
      %msg = new_message_with_text( severity = 'E' text = 'Not authorized to create deliveries' )
    ) TO reported-delivery.
    APPEND VALUE #( %cid = <create>-%cid ) TO failed-delivery.
    RETURN.
  ENDIF.
ENDMETHOD.
```

## Service Binding Activation and Testing

```
In ADT:
  1. Open Service Binding (ZSB_Delivery_O4)
  2. Click 'Activate'
  3. Click entity name in list → 'Preview' button → Fiori Elements app opens

  ADT OData Client (F12 perspective):
  → New → OData Request
  → Service: https://my-s4.ondemand.com/sap/opu/odata4/.../
  → Authenticate → browse $metadata

" Register/find in ABAP system:
/n/iwfnd/v4_admin → Service Administration → list published V4 services
```

## $metadata Document Structure

```xml
<!-- Returned by GET .../$metadata -->
<Schema Namespace="ZSD_DELIVERY_0001" Alias="SAP">
  <EntityType Name="Delivery">
    <Key><PropertyRef Name="DeliveryNo"/></Key>
    <Property Name="DeliveryNo" Type="Edm.String" MaxLength="10"/>
    <Property Name="Status"     Type="Edm.String" MaxLength="1"/>
    <Property Name="ShipDate"   Type="Edm.Date"/>
    <NavigationProperty Name="to_Items" Type="Collection(SAP.DeliveryItem)"
      ContainsTarget="true"/>
  </EntityType>

  <Action Name="PostGoodsIssue" IsBound="true">
    <Parameter Name="in" Type="Collection(SAP.Delivery)" Nullable="false"/>
    <ReturnType Type="Collection(SAP.Delivery)"/>
  </Action>

  <EntitySet Name="Deliveries" EntityType="SAP.Delivery">
    <Annotation Term="Org.OData.Capabilities.V1.DeleteRestrictions">
      <Record><PropertyValue Property="Deletable" Bool="false"/></Record>
    </Annotation>
  </EntitySet>
</Schema>
```

## Multi-Inline Create (Deep Insert)

```
" Create header + items in single POST (managed provider handles nesting):
POST .../Deliveries
{
  "ShipToName": "ACME Corp",
  "ShipDate":   "2026-06-01",
  "to_Items": [
    { "MaterialNo": "MAT001", "Quantity": 10, "Unit": "PC" },
    { "MaterialNo": "MAT002", "Quantity":  5, "Unit": "KG" }
  ]
}

" Response: 201 Created with DeliveryNo assigned (late numbering)
```

## Value Help Integration

```abap
" Annotation in projection CDS:
@Consumption.valueHelpDefinition: [{
  entity: {
    name:    'ZVH_DeliveryStatus',
    element: 'StatusCode'
  }
}]
Status,

" Value help entity:
define view entity ZVH_DeliveryStatus
  as select from zewm_status_codes
{
  key status_code  as StatusCode,
      description  as Description
}
```

## Communication Arrangement (BTP ABAP Environment)

```
" On BTP ABAP (not on-premise), OData V4 services are exposed via:
BTP Cockpit → Instances → ABAP System → Communication Arrangements
  → New Communication Arrangement
  → Communication Scenario: (auto-generated from service binding)
  → Communication User: technical user (COMMUSER_*)
  → Inbound URL:  https://<tenant>.abap.us10.hana.ondemand.com/sap/opu/odata4/...

" No /n/iwfnd/v4_admin on BTP ABAP — service published at activation time
```
