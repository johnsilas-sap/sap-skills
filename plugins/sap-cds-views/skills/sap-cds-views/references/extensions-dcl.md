# CDS Extensions and Access Control Reference

## EXTEND VIEW (Classic CDS Extension)

```abap
" Extend a CDS view without modifying the original — for customer enhancements.
" Only works with 'define view' (classic), not view entities.

@AbapCatalog.sqlViewAppendName: 'ZV_DLV_EXT'   " name for the append view in DDIC
extend view ZI_Delivery_Classic with ZX_Delivery_Extension
{
  " Add fields from the same base table:
  hdr.dangerous_goods_flag as DangerousGoodsFlag,
  hdr.customs_status       as CustomsStatus,

  " Add fields from an additional join:
  hdr.incoterms_code       as IncotermsCode
}

" Restrictions:
" - Cannot add associations in EXTEND VIEW
" - Cannot add keys
" - Only classic 'define view' can be extended (not view entities)
" - Activation of extension may fail if base view is inactive
```

## Metadata Extension (@Metadata.layer)

```abap
" For view entities — add/override annotations without touching the CDS source.
" Used by partners and customers to add UI annotations on SAP standard views.

@Metadata.layer: #CUSTOMER        " layer: CORE < PARTNER < CUSTOMER < LOCALIZATION
@EndUserText.label: 'Delivery UI Extension'

annotate view ZC_Delivery with
{
  " Add UI annotation to existing field (overwrites for this layer):
  @UI.lineItem: [{ position: 5, label: 'Priority' }]
  @UI.selectionField: [{ position: 5 }]
  DangerousGoodsFlag;

  " Extend facets:
  @UI.facet: [{
    id: 'CustomSection',
    type: #FIELDGROUP_REFERENCE,
    targetQualifier: 'Customs',
    label: 'Customs Info',
    position: 50
  }]
  DeliveryNo;

  @UI.fieldGroup: [{ qualifier: 'Customs', position: 10, label: 'Incoterms' }]
  IncotermsCode;
}

" Metadata layers (lowest to highest priority):
" #CORE        — SAP standard
" #PARTNER     — ISV / SI partner
" #CUSTOMER    — Customer (Z namespace)
" #LOCALIZATION — Country-specific
```

## DCL — Data Control Language (Access Control)

```abap
" DCL file restricts which rows a user can read from a CDS view.
" File name = CDS view name (same object, separate source type).
" Assigned automatically based on @AccessControl.authorizationCheck: #CHECK

@EndUserText.label: 'Delivery Access Control'
@MappingRole: true

define role ZI_Delivery
{
  grant select on ZI_Delivery
  where ( Lgnum ) = aspect pfcg_auth( ZEWM_WH, LGNUM, ACTVT = '03' );
}

" This restricts GET /Deliveries to rows where:
" - Current user has PFCG role with auth object ZEWM_WH
" - LGNUM field matches the user's warehouse authorization
" - ACTVT = '03' = display

" Multiple conditions (AND):
define role ZI_WhoStep
{
  grant select on ZI_WhoStep
  where ( Lgnum )   = aspect pfcg_auth( ZEWM_WH, LGNUM, ACTVT = '03' )
    and ( LgTyp  )  = aspect pfcg_auth( ZEWM_LT, LGTYP, ACTVT = '03' );
}

" Inherit parent DCL (for child view — uses root's authorization):
define role ZI_DeliveryItem
{
  grant select on ZI_DeliveryItem
  where inheriting conditions from entity ZI_Delivery;
}
```

## DCL — No Restriction (Open Access)

```abap
" For value helps, reference data, non-sensitive views:
@AccessControl.authorizationCheck: #NOT_REQUIRED

define view entity ZVH_Status { ... }
" No DCL file needed — all users can read
```

## DCL — Aspect Patterns

```abap
" Filter by auth object field value:
where ( CarrierId ) = aspect pfcg_auth( Z_CARRIER, CARRIER, ACTVT = '03' );

" Filter by literal (restrict to current client — usually automatic):
where ( Client ) = $session.client;

" Filter by session user:
where ( CreatedBy ) = $session.user;   " only see own records

" Unrestricted (overrides field restriction — use for privileged roles):
where 1 = 1;   " no restriction

" Combine parent entity restriction with own:
define role ZI_DeliveryItem
{
  grant select on ZI_DeliveryItem
  where ( Lgnum ) = aspect pfcg_auth( ZEWM_WH, LGNUM, ACTVT = '03' );
}
```

## DCL Transaction Configuration

```
SACM → Assign Role (assign DCL role to CDS view)
  CDS View:    ZI_DELIVERY
  DCL Role:    ZI_DELIVERY
  Active:      Yes

SE16 → SACM_CDS_MAPPING — shows all DCL assignments

" Test DCL effect:
ADT → Data Preview (F8) on ZI_Delivery → filtered by current user's authorizations
" Enable auth trace: SU53 / ST01 to see which auth objects checked
```

## Metadata Extension — Full Example (EWM Custom Fields)

```abap
" Scenario: SAP delivers ZC_EwmDelivery — we add customer-specific fields
" without modifying SAP's source CDS.

" Step 1: Extend the interface view (ZI_EwmDelivery) via EXTEND VIEW
"         OR add fields via BADI if view entity
" Step 2: Add UI annotations via Metadata Extension

@Metadata.layer: #CUSTOMER
@EndUserText.label: 'EWM Delivery Customer Extension'

annotate view ZC_EwmDelivery with
{
  " Add label to existing field:
  @EndUserText.label: 'Dangerous Goods'
  @UI.lineItem: [{ position: 85, label: 'DG', importance: #LOW }]
  @UI.selectionField: [{ position: 50 }]
  DangerousGoodsFlag;

  " Add field group:
  @UI.fieldGroup: [{ qualifier: 'ZCustomer', position: 10 }]
  CustomerRef;

  @UI.fieldGroup: [{ qualifier: 'ZCustomer', position: 20 }]
  CustomerPONo;

  " Add section to Object Page:
  @UI.facet: [{
    id:              'CustomerSection',
    type:            #FIELDGROUP_REFERENCE,
    targetQualifier: 'ZCustomer',
    label:           'Customer Fields',
    position:        100
  }]
  DeliveryNo;
}
```

## CDS View Extension via BAdI (S/4HANA Cloud)

```abap
" In S/4HANA Cloud, EXTEND VIEW is restricted.
" Use Business Add-Ins (RAP BAdIs or CDS BAdIs) to add fields.

" Check if a CDS BAdI spot exists:
" SE18 → search for enhancement spot related to the CDS view name
" ADT → Enhancement Spot Explorer → search view name

" Example: add field via CDS BAdI implementation in projection view:
define root view entity ZC_EwmDelivery
  as projection on ZI_EwmDelivery
  provider contract transactional_query
{
  " ... standard fields ...

  " Field added by BAdI extension (appears after BAdI activation):
  @ObjectModel.virtualElement: true
  @ObjectModel.virtualElementCalculatedBy: 'ABAP:ZCL_EWM_DLV_EXTENSION'
  cast( '' as abap.char(20) ) as CustomerRef
}
```
