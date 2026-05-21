# Fiori Elements Reference

## How Fiori Elements Works

Fiori Elements generates UI from OData metadata + CDS annotations. You don't write HTML/JS views — the framework renders them at runtime based on what the annotations say.

```
CDS Entity + @UI annotations
  ↓  (activates OData V4 service with $metadata)
Fiori Elements runtime (sap.fe.templates)
  ↓  (reads $metadata and renders)
Generated UI (List Report / Object Page / etc.)
```

## List Report Page

Filterable list with a table. Navigates to Object Page on row click.

### Annotations Needed

```cds
annotate DeliveryService.Deliveries with @(
  // Table columns
  UI.LineItem: [
    { $Type: 'UI.DataField', Value: DeliveryNo,  Label: 'Delivery' },
    { $Type: 'UI.DataField', Value: ShipToName,  Label: 'Ship To' },
    { $Type: 'UI.DataField', Value: Status,      Label: 'Status' },
    { $Type: 'UI.DataField', Value: ShipDate,    Label: 'Ship Date' },
    { $Type: 'UI.DataField', Value: TotalWeight, Label: 'Weight' },
    {
      $Type: 'UI.DataFieldForAction',
      Action: 'DeliveryService.PostGoodsIssue',
      Label: 'Post GI',
      Inline: true                               // Shows button in each row
    }
  ],

  // Filter bar fields
  UI.SelectionFields: [ Status, ShipDate, CarrierName ],

  // Page header
  UI.HeaderInfo: {
    TypeName:       'Delivery',
    TypeNamePlural: 'Deliveries',
    Title:          { Value: DeliveryNo }
  }
);
```

### manifest.json (List Report target)

```json
"DeliveryList": {
  "type": "Component",
  "id": "DeliveryList",
  "name": "sap.fe.templates.ListReport",
  "options": {
    "settings": {
      "entitySet": "Deliveries",
      "initialLoad": true,
      "variantManagement": "Page",
      "navigationRoute": "DeliveryDetail"
    }
  }
}
```

## Object Page

Detail view with header and sections. Used for record display and editing.

### Header Annotations

```cds
annotate DeliveryService.Deliveries with @(
  UI.HeaderInfo: {
    TypeName:       'Delivery',
    TypeNamePlural: 'Deliveries',
    Title:          { Value: DeliveryNo },
    Description:    { Value: ShipToName },
    ImageUrl:       '/images/delivery.png'        // Optional header image
  },

  // Key fields shown in header
  UI.HeaderFacets: [
    {
      $Type:  'UI.ReferenceFacet',
      Target: '@UI.FieldGroup#HeaderData',
      Label:  'Key Details'
    }
  ],

  UI.FieldGroup#HeaderData: {
    Data: [
      { Value: Status,      Label: 'Status' },
      { Value: ShipDate,    Label: 'Ship Date' },
      { Value: CarrierName, Label: 'Carrier' }
    ]
  }
);
```

### Sections (Facets)

```cds
annotate DeliveryService.Deliveries with @(
  UI.Facets: [
    // Simple field group section
    {
      $Type:  'UI.ReferenceFacet',
      ID:     'GeneralInfo',
      Label:  'General Information',
      Target: '@UI.FieldGroup#GeneralInfo'
    },
    // Table section (navigation property / items)
    {
      $Type:  'UI.ReferenceFacet',
      ID:     'Items',
      Label:  'Delivery Items',
      Target: 'to_Items/@UI.LineItem'
    },
    // Collection facet (groups sub-sections)
    {
      $Type: 'UI.CollectionFacet',
      ID:    'AddressSection',
      Label: 'Addresses',
      Facets: [
        { $Type: 'UI.ReferenceFacet', Target: '@UI.FieldGroup#ShipTo', Label: 'Ship To' },
        { $Type: 'UI.ReferenceFacet', Target: '@UI.FieldGroup#BillTo', Label: 'Bill To' }
      ]
    }
  ],

  UI.FieldGroup#GeneralInfo: {
    Data: [
      { Value: DeliveryNo },
      { Value: Status },
      { Value: TotalWeight },
      { Value: TotalVolume }
    ]
  },

  UI.FieldGroup#ShipTo: {
    Data: [
      { Value: ShipToName },
      { Value: ShipToStreet },
      { Value: ShipToCity },
      { Value: ShipToCountry }
    ]
  }
);
```

## Analytical List Page (ALP)

KPI tiles + interactive chart + filterable table — for operational dashboards.

```cds
annotate WarehouseTaskService.Tasks with @(
  // KPI tiles
  UI.KPI#OpenTasks: {
    DataPoint: {
      Value:         TaskCount,
      Title:         'Open Tasks',
      CriticalityCalculation: {
        ImprovementDirection: #Minimize,
        ToleranceRangeLowValue: 10,
        DeviationRangeLowValue: 20
      }
    },
    SelectionVariant: {
      SelectOptions: [{ PropertyName: Status, Ranges: [{ Low: 'Open', Option: #EQ }] }]
    }
  },

  // Chart
  UI.Chart: {
    ChartType: #Bar,
    Dimensions: [ StorageType ],
    Measures:   [ TaskCount ],
    Title:      'Tasks by Storage Type'
  },

  // Table (same as ListReport)
  UI.LineItem: [
    { Value: TaskNo },
    { Value: Material },
    { Value: Status },
    { Value: Priority }
  ]
);
```

## Worklist Page

Simple task list without a filter bar — for action-oriented apps.

```json
"TaskWorklist": {
  "type": "Component",
  "name": "sap.fe.templates.ListReport",
  "options": {
    "settings": {
      "entitySet": "WarehouseTasks",
      "variantManagement": "None",
      "initialLoad": true,
      "hideFilterBar": true       // Worklist — no filter bar
    }
  }
}
```

## Actions (Buttons) in Fiori Elements

### Bound Action (single record)

```cds
// In CDS service definition:
action PostGoodsIssue(DeliveryNo: String) returns String;

// Annotation:
{ $Type: 'UI.DataFieldForAction', Action: 'DeliveryService.PostGoodsIssue', Label: 'Post GI' }
```

### Unbound Action (list-level toolbar)

```cds
// Toolbar button — no record context needed
{
  $Type:    'UI.DataFieldForAction',
  Action:   'DeliveryService.BulkConfirm',
  Label:    'Confirm All',
  Inline:   false    // Shows in toolbar, not row
}
```

### Action with Dialog (Fiori Elements V4)

```cds
// Action with parameters prompts a dialog automatically
action ConfirmDelivery(
  DeliveryNo:  String,
  PostingDate: Date    @Common.Label: 'Posting Date'
) returns String;
```

## Criticality (Color Coding)

```cds
// Enum: 0=None, 1=Negative/Red, 2=Critical/Orange, 3=Positive/Green, 5=New/Purple
UI.LineItem: [
  {
    $Type:       'UI.DataField',
    Value:       Status,
    Criticality: StatusCriticality   // computed field returning 0-3
  }
]

// In CDS — add a virtual/calculated field:
entity Deliveries {
  ...
  virtual StatusCriticality : Integer;
}

// Set in ABAP backend or CAP handler:
// 1 = Blocked, 2 = At Risk, 3 = On Track
```

## Draft Handling (Edit Flow)

Fiori Elements V4 supports draft editing — partial saves without affecting production data.

```cds
service DeliveryService {
  @odata.draft.enabled
  entity Deliveries as projection on db.Deliveries;
}
```

manifest.json — enable edit button automatically:
```json
"options": {
  "settings": {
    "entitySet": "Deliveries",
    "editableHeaderSection": true
  }
}
```
