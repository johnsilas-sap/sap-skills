# CDS Annotations Reference

## @AbapCatalog (Classic Views Only)

```abap
@AbapCatalog.sqlViewName: 'ZV_DELIVERY'      " DDIC view name — ≤16 chars, required for 'define view'
@AbapCatalog.compiler.compareFilter: true    " optimize filter pushdown
@AbapCatalog.preserveKey: true               " keep key from base table even if not projected
@AbapCatalog.buffering.status: #ACTIVE       " table buffering (SINGLE / GENERIC / FULL)
@AbapCatalog.buffering.type: #FULL
```

## @AccessControl

```abap
" View-level:
@AccessControl.authorizationCheck: #CHECK             " enforce DCL (default)
@AccessControl.authorizationCheck: #NOT_REQUIRED      " no DCL — open read
@AccessControl.authorizationCheck: #PRIVILEGED_ONLY   " only internal/privileged callers
@AccessControl.authorizationCheck: #INHERITED          " inherit from base view

" Programmatic bypass (in ABAP — for technical reads where auth was already checked):
SELECT * FROM zi_delivery BYPASSING BUFFER INTO TABLE @lt_deliveries.
" Note: BYPASSING BUFFER also skips DCL — use only for system-level logic
```

## @EndUserText

```abap
@EndUserText.label: 'Delivery Interface View'      " display label in UI/search helps
@EndUserText.quickInfo: 'Shows all EWM deliveries including items and HU assignments'
```

## @Semantics — Field Meaning Declarations

```abap
" ── Amounts and quantities ────────────────────────────────────────────
@Semantics.amount.currencyCode: 'Currency'   " field 'Currency' holds the currency key
NetValue,

@Semantics.currencyCode: true                " this field IS the currency code
Currency,

@Semantics.quantity.unitOfMeasure: 'Unit'    " field 'Unit' holds UoM key
Quantity,

@Semantics.unitOfMeasure: true               " this field IS the unit of measure
Unit,

" ── Dates and timestamps ─────────────────────────────────────────────
@Semantics.businessDate.at: true             " business key date
ShipDate,

@Semantics.systemDate.createdAt: true        " record creation date
CreatedDate,

@Semantics.systemDateTime.createdAt: true    " record creation UTC timestamp (utclong)
CreatedAt,

@Semantics.systemDateTime.lastChangedAt: true          " last change (for ETag)
ChangedAt,

@Semantics.systemDateTime.localInstanceLastChangedAt: true  " local last change (RAP ETag)
LocalLastChangedAt,

" ── User fields ───────────────────────────────────────────────────────
@Semantics.user.createdBy: true
CreatedBy,

@Semantics.user.lastChangedBy: true
ChangedBy,

" ── UUID / key ────────────────────────────────────────────────────────
@Semantics.systemUUID: true
key TaskUUID,

" ── Text / language ───────────────────────────────────────────────────
@Semantics.language: true
Language,

@Semantics.text: true                        " this field is a text for another field
MaterialDescription,

" ── Document number ───────────────────────────────────────────────────
@Semantics.businessDocumentBasicData.documentNumber: true
DeliveryNo,
```

## @ObjectModel

```abap
" Transactional processing (RAP)
@ObjectModel.transactionalProcessingEnabled: true    " classic BOPF
@ObjectModel.transactionalProcessingEnabled: false   " read-only (reporting)

" Provider contract (for projection views)
provider contract transactional_query     " Fiori / OData UI
provider contract analytical_query        " embedded analytics / SAC

" Virtual element — computed field (not in DB)
@ObjectModel.virtualElement: true
@ObjectModel.virtualElementCalculatedBy: 'ABAP:ZCL_DELIVERY_VIRTUAL'
StatusText : abap.char(20);              " only in abstract/custom entity virtual fields

" Usage for virtual calculation in ABAP (if_sadl_exit_calc_element_read):
CLASS zcl_delivery_virtual DEFINITION IMPLEMENTING if_sadl_exit_calc_element_read.
  PUBLIC SECTION.
    METHODS if_sadl_exit_calc_element_read~calculate REDEFINITION.
ENDCLASS.

" Association semantics
@ObjectModel.association.type: [#TO_COMPOSITION_CHILD]
_Items,

@ObjectModel.association.type: [#TO_COMPOSITION_PARENT]
_Delivery,

" Readable by (authorization scope)
@ObjectModel.dataCategory: #VALUE_HELP        " used as value help source
@ObjectModel.dataCategory: #TEXT              " text table for another entity
```

## @Analytics

```abap
" Mark view as analytical:
@Analytics.dataCategory: #CUBE          " fact view (measures + dimensions)
@Analytics.dataCategory: #DIMENSION     " dimension view (attributes)
@Analytics.dataCategory: #FACT          " raw fact (used inside cube)

" Aggregation behavior:
@DefaultAggregation: #SUM
TotalWeight,

@DefaultAggregation: #MAX
MaxShipDate,

@DefaultAggregation: #MIN
MinShipDate,

@DefaultAggregation: #COUNT_DISTINCT
UniqueCarriers,

@DefaultAggregation: #NOP               " no aggregation (dimension key)
DeliveryNo,

" Measure annotation (for Fiori ALP / embedded analytics):
@Measures.unitOfMeasure: 'WeightUOM'
TotalWeight,

@Measures.iSOCurrency: 'Currency'
NetValue,

" Hidden from analytical query (internal use only):
@AnalyticsDetails.hidden: true
InternalField,
```

## @UI Annotations (Projection Views)

```abap
" ── Line item table columns ────────────────────────────────────────────
@UI.lineItem: [{
  position: 10,
  label: 'Delivery No.',
  importance: #HIGH
}]
DeliveryNo,

@UI.lineItem: [{
  position: 20,
  criticality: 'StatusCriticality',
  criticalityRepresentation: #WITH_ICON
}]
Status,

" ── Filter bar fields ─────────────────────────────────────────────────
@UI.selectionField: [{ position: 10 }]
Status,

@UI.selectionField: [{ position: 20 }]
ShipDate,

" ── Object Page header ────────────────────────────────────────────────
@UI.headerInfo: {
  typeName: 'Delivery',
  typeNamePlural: 'Deliveries',
  title: { type: #STANDARD, value: 'DeliveryNo' },
  description: { type: #STANDARD, value: 'ShipToName' }
}

" ── Field groups (form sections) ──────────────────────────────────────
@UI.fieldGroup: [{ qualifier: 'Shipping', position: 10, label: 'Ship Date' }]
ShipDate,

@UI.fieldGroup: [{ qualifier: 'Shipping', position: 20 }]
CarrierId,

" ── Facets (Object Page sections) ─────────────────────────────────────
@UI.facet: [
  {
    id: 'ShippingSection',
    type: #FIELDGROUP_REFERENCE,
    targetQualifier: 'Shipping',
    label: 'Shipping Details'
  },
  {
    id: 'ItemsSection',
    type: #LINEITEM_REFERENCE,
    targetElement: '_Items',
    label: 'Delivery Items'
  }
]

" ── Hidden from UI ────────────────────────────────────────────────────
@UI.hidden: true
StatusCriticality,

" ── Action buttons ────────────────────────────────────────────────────
@UI.lineItem: [{
  type: #FOR_ACTION,
  dataAction: 'PostGoodsIssue',
  label: 'Post GI',
  position: 90,
  inline: true
}]
ActionDummy : abap.char(1);   " placeholder field for action annotation
```

## @Common — Value Helps and Text

```abap
" Field label override
@Common.label: 'Delivery Number'
DeliveryNo,

" Text for a code field (show text next to code)
@Common.text: 'CarrierName'
@Common.textArrangement: #TEXT_ONLY     " show text only (not code)
CarrierId,

" Value help definition
@Consumption.valueHelpDefinition: [{
  entity: { name: 'ZVH_Carrier', element: 'CarrierId' }
}]
CarrierId,

" Semantic key (for Object Page URL)
@Common.semanticKey: ['DeliveryNo']

" Field is a unit for another field
@Common.unitInFinal: 'Unit'
Quantity,
```

## @Consumption

```abap
" Filter value help (inline filter bar dropdown)
@Consumption.filter.multipleSelections: true
@Consumption.filter.defaultValue: 'O'
Status,

" Sort order (default)
@Consumption.sortOrder: [{ element: 'ShipDate', direction: #ASC }]

" Derivation (auto-fill related fields)
@Consumption.derivation: {
  lookupEntity: 'ZI_Carrier',
  resultElement: 'CarrierName'
}
CarrierId,
```
