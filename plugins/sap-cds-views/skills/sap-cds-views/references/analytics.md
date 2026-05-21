# CDS Analytical Views Reference

## Analytical View Layering

```
Query View   (@Analytics.query: true)          ← used by SAP Analytics Cloud / Fiori ALP
    ↓ selects from
Cube View    (@Analytics.dataCategory: #CUBE)  ← joins facts + dimensions, holds measures
    ↓ selects from
Fact View    (@Analytics.dataCategory: #FACT)  ← raw transactional data (close to DB tables)

Dimension Views  (@Analytics.dataCategory: #DIMENSION)  ← master data (material, carrier, location)
    ↑ referenced by cube via associations
```

## Fact View

```abap
@AccessControl.authorizationCheck: #CHECK
@Analytics.dataCategory: #FACT
@EndUserText.label: 'Delivery Fact View'

define view entity ZI_DeliveryFact
  as select from zewm_delivery as hdr
  join zewm_delivery_item as itm
    on hdr.delivery_no = itm.delivery_no
{
  key hdr.delivery_no   as DeliveryNo,
  key itm.item_no       as ItemNo,
      hdr.status        as Status,
      hdr.ship_date     as ShipDate,
      hdr.carrier_id    as CarrierId,
      hdr.lgnum         as Lgnum,
      itm.material_no   as MaterialNo,

      @Semantics.quantity.unitOfMeasure: 'Unit'
      itm.quantity      as Quantity,
      @Semantics.unitOfMeasure: true
      itm.unit          as Unit,

      @Semantics.amount.currencyCode: 'Currency'
      itm.net_value     as NetValue,
      @Semantics.currencyCode: true
      itm.currency      as Currency,

      @Semantics.systemDateTime.createdAt: true
      hdr.created_at    as CreatedAt
}
```

## Cube View

```abap
@AccessControl.authorizationCheck: #CHECK
@Analytics.dataCategory: #CUBE
@EndUserText.label: 'Delivery Analytics Cube'

define view entity ZI_DeliveryCube
  as select from ZI_DeliveryFact as fact

  " Dimension associations — master data attributes
  association [0..1] to ZI_CarrierDim    as _Carrier
    on fact.CarrierId = _Carrier.CarrierId
  association [0..1] to ZI_MaterialDim   as _Material
    on fact.MaterialNo = _Material.MaterialNo
  association [0..1] to ZI_WarehouseDim  as _Warehouse
    on fact.Lgnum = _Warehouse.Lgnum
  association [0..1] to ZI_CalendarDim   as _ShipDate
    on fact.ShipDate = _ShipDate.CalendarDate
{
  " ── Dimensions (non-aggregatable) ────────────────────────────────────
  @DefaultAggregation: #NOP
  key DeliveryNo,
  @DefaultAggregation: #NOP
  key ItemNo,

  @DefaultAggregation: #NOP
  Status,
  @DefaultAggregation: #NOP
  ShipDate,
  @DefaultAggregation: #NOP
  CarrierId,
  @DefaultAggregation: #NOP
  Lgnum,
  @DefaultAggregation: #NOP
  MaterialNo,
  @DefaultAggregation: #NOP
  Currency,
  @DefaultAggregation: #NOP
  Unit,

  " ── Measures (aggregatable) ─────────────────────────────────────────
  @DefaultAggregation: #SUM
  @Semantics.quantity.unitOfMeasure: 'Unit'
  Quantity,

  @DefaultAggregation: #SUM
  @Semantics.amount.currencyCode: 'Currency'
  NetValue,

  @DefaultAggregation: #COUNT_DISTINCT
  @AnalyticsDetails.exceptionAggregationSteps: [{
    exceptionAggregationBehavior: #COUNT,
    exceptionAggregationElements: ['DeliveryNo']
  }]
  cast( 1 as abap.int4 ) as DeliveryCount,

  " ── Dimension references ─────────────────────────────────────────────
  _Carrier,
  _Material,
  _Warehouse,
  _ShipDate
}
```

## Dimension View

```abap
@Analytics.dataCategory: #DIMENSION
@EndUserText.label: 'Carrier Dimension'

define view entity ZI_CarrierDim
  as select from zewm_carrier as car
{
  @EndUserText.label: 'Carrier ID'
  key car.carrier_id   as CarrierId,

  @EndUserText.label: 'Carrier Name'
      car.carrier_name as CarrierName,

  @EndUserText.label: 'Country'
      car.country      as Country,

  @EndUserText.label: 'Transport Mode'
      car.transport_mode as TransportMode,

  @EndUserText.label: 'Rating'
  @DefaultAggregation: #NOP
      car.rating       as CarrierRating
}
```

## Query View

```abap
@AccessControl.authorizationCheck: #CHECK
@Analytics.query: true
@EndUserText.label: 'Delivery Performance Query'

define view entity ZQ_DeliveryPerformance
  as select from ZI_DeliveryCube
{
  " Select subset of dimensions and measures for this report:
  @EndUserText.label: 'Status'
  Status,

  @EndUserText.label: 'Ship Date'
  ShipDate,

  @EndUserText.label: 'Carrier'
  CarrierId,

  @EndUserText.label: 'Warehouse'
  Lgnum,

  @EndUserText.label: 'Total Quantity'
  Quantity,

  @EndUserText.label: 'Net Value'
  NetValue,

  @EndUserText.label: 'Deliveries'
  DeliveryCount,

  " Expose dimension attributes via path (adds label from dimension view):
  _Carrier.CarrierName    as CarrierName,
  _Carrier.Country        as CarrierCountry,
  _Material.MaterialGroup as MaterialGroup,
  _Warehouse.WarehouseName as WarehouseName
}
```

## KPI Annotation (@UI.KPI)

```abap
" On projection view — for Fiori ALP KPI header:
@UI.KPI: [{
  qualifier: 'OpenDeliveries',
  label:     'Open Deliveries',
  dataPoint: {
    value:       'DeliveryCount',
    title:       'Open Deliveries',
    criticality: 'KPICriticality'
  },
  selectionVariantQualifier: 'OpenOnly'
}]

@UI.selectionVariant: [{
  qualifier: 'OpenOnly',
  text:      'Open Deliveries',
  filterDefaultValues: [{ property: 'Status', value: 'O' }]
}]
```

## Embedded Analytics in Fiori (ALP)

```json
// manifest.json for Analytical List Page:
"targets": {
  "DeliveryALP": {
    "type": "Component",
    "name": "sap.fe.templates.AnalyticalListPage",
    "options": {
      "settings": {
        "entitySet": "ZQ_DeliveryPerformance",
        "defaultTemplateAnnotationPath": "com.sap.vocabularies.UI.v1.PresentationVariant",
        "enableDraftSupport": false,
        "chartPresentationQualifier": "",
        "tableType": "GridTable",
        "multipleViews": {
          "mode": "single"
        }
      }
    }
  }
}
```

## @UI.PresentationVariant for Analytics

```abap
@UI.presentationVariant: [{
  sortOrder: [{ property: 'ShipDate', descending: true }],
  groupBy:   ['Status'],
  total:     ['NetValue', 'Quantity'],
  visualizations: ['@UI.LineItem', '@UI.Chart']
}]

@UI.chart: [{
  title:      'Deliveries by Status',
  chartType:  #BAR,
  dimensions: ['Status'],
  measures:   ['DeliveryCount'],
  dimensionAttributes: [{
    dimension: 'Status',
    role:      #CATEGORY
  }],
  measureAttributes: [{
    measure:   'DeliveryCount',
    role:      #AXIS_1,
    dataPoint: '@UI.DataPoint#DeliveryCount'
  }]
}]

@UI.dataPoint: [{
  qualifier:   'DeliveryCount',
  value:       'DeliveryCount',
  title:       'Open Deliveries',
  criticality: 'KPICriticality'
}]
```
