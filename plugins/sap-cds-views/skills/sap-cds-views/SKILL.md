---
name: sap-cds-views
description: |
  Core Data Services (CDS) views for SAP ABAP development on S/4HANA and BTP. Covers classic define view vs modern view entities, associations (to one/to many/to parent/composition), joins, CDS built-in functions and expressions, key annotations (@AbapCatalog, @AccessControl, @Semantics, @ObjectModel, @Analytics, @UI, @EndUserText), CDS hierarchy for tree structures, analytical cube/dimension/query views, metadata extensions, and EWM/TM-specific CDS patterns.

  Key patterns:
  - Root/child view entity composition trees (RAP data model)
  - Association vs join selection guide
  - Path expression $projection / $self / _Association.Field
  - @Semantics for amounts, quantities, dates, and user fields
  - @ObjectModel for virtual elements and transactional processing
  - Analytical views: cube → dimension → query layering
  - CDS hierarchy (parent-child for TreeTable)
  - EXTEND VIEW and metadata extensions (@Metadata.layer)
  - DCL access control (PFCG / aspect pfcg_auth)
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-20"
  abap_version: "S/4HANA 2022+ / ABAP 7.58+"
---

# SAP CDS Views

## View Type Quick Reference

| Type | Keyword | Use |
|---|---|---|
| Classic view | `define view` | Reporting, legacy, non-RAP |
| Interface view entity | `define view entity` | RAP child, non-root |
| Root view entity | `define root view entity` | RAP BO root |
| Projection view | `define root/view entity ... as projection on` | Consumption layer |
| Abstract entity | `define abstract entity` | Action parameter types |
| Custom entity | `define custom entity` | Non-DB data sources |
| Hierarchy | `define hierarchy` | Tree structures |

## Minimal CDS View Entity

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Delivery Interface View'
define view entity ZI_Delivery
  as select from zewm_delivery as hdr
{
  key hdr.delivery_no   as DeliveryNo,
      hdr.status        as Status,
      hdr.ship_to_name  as ShipToName,
      @Semantics.businessDate.at: true
      hdr.ship_date     as ShipDate
}
```

## Key Transactions / Tools

| Tool | Purpose |
|---|---|
| ADT (Eclipse) | Primary CDS development IDE — syntax check, activation, where-used |
| SE11 | View generated database view (DDIC view) |
| SACM | Assign DCL (access control) to CDS |
| `/n/iwfnd/v4_admin` | OData V4 service metadata (shows CDS annotations) |
| ST05 | SQL trace — see generated SQL from CDS |
| ADT → Data Preview | F8 — run SELECT on CDS view, see results |
