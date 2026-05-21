# CDS UI Annotations Reference

## Annotation Namespaces

```cds
using {
  sap.common,
  sap from '@sap/cds/common'
} from '@sap/cds/common';

// Annotations used: @UI.*, @Common.*, @Capabilities.*, @Communication.*
```

## @UI.LineItem — Table Columns

```cds
UI.LineItem: [
  // Standard field
  { $Type: 'UI.DataField', Value: DeliveryNo, Label: 'Delivery No.' },

  // With criticality color
  { $Type: 'UI.DataField', Value: Status, Criticality: StatusCriticality },

  // Link to another entity (navigation)
  { $Type: 'UI.DataFieldWithNavigationPath', Value: ShipToName, Target: to_ShipTo },

  // Action button (inline per row)
  { $Type: 'UI.DataFieldForAction', Action: 'DeliveryService.PostGI', Label: 'Post GI', Inline: true },

  // External link
  { $Type: 'UI.DataFieldWithUrl', Value: TrackingNo, Url: TrackingUrl, Label: 'Track' },

  // Importance (hides on smaller screens)
  { $Type: 'UI.DataField', Value: TotalWeight, ![@UI.Importance]: #Low }
]
```

## @UI.SelectionFields — Filter Bar

```cds
UI.SelectionFields: [ Status, ShipDate, CarrierName, WarehouseNo ]
// Shows these fields in the filter bar of List Report / ALP
```

## @UI.HeaderInfo — Page Title / Description

```cds
UI.HeaderInfo: {
  TypeName:       'Delivery',
  TypeNamePlural: 'Deliveries',
  Title:          { $Type: 'UI.DataField', Value: DeliveryNo },
  Description:    { $Type: 'UI.DataField', Value: ShipToName },
  ImageUrl:       ImageUrl    // optional — field containing URL
}
```

## @UI.FieldGroup — Groups of Fields

```cds
UI.FieldGroup#Shipping: {
  Label: 'Shipping Details',
  Data: [
    { $Type: 'UI.DataField', Value: ShipDate },
    { $Type: 'UI.DataField', Value: CarrierName },
    { $Type: 'UI.DataField', Value: TrackingNo },
    { $Type: 'UI.DataField', Value: TotalWeight },
    // Read-only label with static text:
    { $Type: 'UI.DataField', Value: WeightUOM, Label: 'UOM' }
  ]
}
```

## @UI.Facets — Object Page Sections

```cds
UI.Facets: [
  // Field group section
  {
    $Type:  'UI.ReferenceFacet',
    ID:     'ShippingSection',
    Label:  'Shipping',
    Target: '@UI.FieldGroup#Shipping'
  },

  // Table from navigation property
  {
    $Type:  'UI.ReferenceFacet',
    ID:     'ItemsSection',
    Label:  'Items',
    Target: 'to_Items/@UI.LineItem'
  },

  // Grouped sub-sections
  {
    $Type: 'UI.CollectionFacet',
    ID:    'AddressGroup',
    Label: 'Addresses',
    Facets: [
      { $Type: 'UI.ReferenceFacet', Target: '@UI.FieldGroup#ShipTo', Label: 'Ship To' },
      { $Type: 'UI.ReferenceFacet', Target: '@UI.FieldGroup#BillTo', Label: 'Bill To' }
    ]
  }
]
```

## @UI.Identification — Quick Info / Highlights

```cds
UI.Identification: [
  { $Type: 'UI.DataField', Value: Status },
  { $Type: 'UI.DataField', Value: Priority },
  { $Type: 'UI.DataFieldForAction', Action: 'DeliveryService.Confirm', Label: 'Confirm' }
]
// Shown in header area of Object Page
```

## @Common Annotations

```cds
// Field label (used everywhere)
DeliveryNo @Common.Label: 'Delivery Number';

// Value help (dropdown/search help)
Status @Common.ValueList: {
  CollectionPath: 'StatusCodes',
  Parameters: [
    { $Type: 'Common.ValueListParameterOut', LocalDataProperty: Status, ValueListProperty: 'Code' },
    { $Type: 'Common.ValueListParameterDisplayOnly', ValueListProperty: 'Description' }
  ]
};

// Text association (show text alongside code)
CarrierId @Common.Text: CarrierName @Common.TextArrangement: #TextOnly;

// Semantic key (used in Object Page URL)
@Common.SemanticKey: [DeliveryNo]

// Field is a unit for another field
TotalWeight @Measures.Unit: WeightUOM;

// Currency
NetValue @Measures.ISOCurrency: Currency;
```

## @Capabilities Annotations

```cds
// Entity-level capabilities
annotate DeliveryService.Deliveries with @(
  Capabilities.Insertable:  false,     // No create button
  Capabilities.Deletable:   false,     // No delete button
  Capabilities.Updatable:   true,      // Allow editing
  Capabilities.SearchRestrictions.Searchable: false   // No search box
);

// Navigation property restrictions
annotate DeliveryService.Deliveries with @(
  Capabilities.NavigationRestrictions.RestrictedProperties: [{
    NavigationProperty: to_Items,
    InsertRestrictions.Insertable: false,
    DeleteRestrictions.Deletable:  false
  }]
);
```

## @UI.Hidden and @UI.ReadOnly

```cds
// Hide field completely from UI
InternalField @UI.Hidden;

// Hide from list but show on object page
AuditField @UI.HiddenFilter: true;   // Hidden from filter bar

// Read-only in edit mode
PostingDate @Core.Immutable;         // Cannot be changed after create
SystemField  @Core.Computed;         // System-set, never editable
```

## @UI.Chart — For ALP / Charts

```cds
UI.Chart: {
  Title:      'Tasks by Storage Type',
  ChartType:  #Bar,
  Dimensions: [ StorageType ],
  DimensionAttributes: [{
    Dimension: StorageType,
    Role:      #Category
  }],
  Measures:   [ TaskCount ],
  MeasureAttributes: [{
    Measure:   TaskCount,
    Role:      #Axis1,
    DataPoint: '@UI.DataPoint#TaskCount'
  }]
},

UI.DataPoint#TaskCount: {
  Value:       TaskCount,
  Title:       'Open Tasks',
  Criticality: TaskCountCriticality
}
```

## @UI.PresentationVariant — Default Sort / Table Settings

```cds
UI.PresentationVariant: {
  SortOrder: [
    { Property: ShipDate, Descending: false },
    { Property: Priority, Descending: true }
  ],
  GroupBy: [ Status ],
  MaxItems: 20,
  Visualizations: ['@UI.LineItem']
}
```

## @UI.SelectionVariant — Default Filters

```cds
UI.SelectionVariant#OpenDeliveries: {
  Text: 'Open Deliveries',
  SelectOptions: [{
    PropertyName: Status,
    Ranges: [{ Low: 'Open', Option: #EQ, Sign: #I }]
  }]
}
```

## Annotation File Structure (.cds vs annotations.xml)

### CDS (recommended for CAP / new S/4 development)

```cds
// annotations/delivery-annotations.cds
using DeliveryService from '../srv/delivery-service';

annotate DeliveryService.Deliveries with @(...) { ... };
```

### XML Annotations (for ABAP OData V2 / SEGW)

```xml
<!-- webapp/localService/metadata.xml or ABAP annotations -->
<Annotations Target="ZEWM_DELIVERY_SRV.DeliverySet">
  <Annotation Term="UI.LineItem">
    <Collection>
      <Record Type="UI.DataField">
        <PropertyValue Property="Value" Path="DeliveryNo"/>
        <PropertyValue Property="Label" String="Delivery No."/>
      </Record>
    </Collection>
  </Annotation>
</Annotations>
```
