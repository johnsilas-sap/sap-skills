# RAP Draft Handling Reference

## Draft Architecture

```
Active Table:  zewm_delivery        (committed data, visible to all)
Draft Table:   zewm_delivery_d      (work-in-progress, per user, locked)

Draft lifecycle:
  1. User opens app → READ active data
  2. User clicks Edit → system calls Draft Edit action → copies active→draft, locks entity
  3. User makes changes → saved to draft table only (auto-save on navigate)
  4. User clicks Save → Draft Activate → validations → write active table → delete draft
  5. User clicks Discard → Draft Discard → delete draft record, release lock
```

## Draft Table Definition

```abap
" SE11: copy zewm_delivery → add draft admin fields

" Draft table must have same key fields as active table PLUS:
" DRAFTENTITYOPERATIONTAB — type /bobf/conf_key     (operation)
" DRAFTUUID               — type sysuuid_x16        (unique draft key)
" DRAFTENTITYCREATEDBY    — type abp_creation_user
" DRAFTENTITYCREATEDONDATETIME — type abp_creation_tstmpl
" DRAFTENTITYLASTCHANGEDBY — type abp_locinst_lastchange_user
" DRAFTENTITYLASTCHANGEDONDATETIME — type abp_locinst_lastchange_tstmpl
" DRAFTADMINISACTIVEENTITY — type abp_admin_is_active  (X = active shadow)

" Minimal draft table:
TABLES: zewm_delivery_d.
" Type: Transparent Table
" Key fields:        delivery_no (from active table)
" Additional:        all other active table fields
" + draft admin fields listed above
```

## BDEF Draft Configuration

```abap
managed;
persistent table zewm_delivery;
draft table     zewm_delivery_d;
lock master total etag LocalLastChangedAt;

define behavior for ZI_Delivery alias Delivery
{
  draft action Edit;         " lock + copy active→draft
  draft action Activate;     " apply validations, commit draft→active
  draft action Discard;      " delete draft, release lock
  draft action Resume;       " return to existing draft
  draft determine action Prepare;   " validate before Activate (optional but recommended)

  create;
  update;
  delete;
  association _Items { create; with draft; }

  validation ValidateStatus  on save { create; update; }  " runs during Activate
  validation ValidateShipDate on save { create; update; }

  determination SetCreationDefaults on modify { create; }

  field ( readonly ) DeliveryNo;
  field ( mandatory ) ShipToName;
}

define behavior for ZI_DeliveryItem alias DeliveryItem
{
  update;
  delete;
  lock dependent by _Delivery;
  authorization dependent by _Delivery;
  " No separate draft actions on child — handled by root
}
```

## Projection BDEF for Draft

```abap
projection;
use draft;

define behavior for ZC_Delivery alias Delivery
{
  use create;
  use update;
  use delete;

  use draft action Edit;
  use draft action Activate;
  use draft action Discard;
  use draft action Resume;
  use draft determine action Prepare;

  use association _Items { create; with draft; }
}

define behavior for ZC_DeliveryItem alias DeliveryItem
{
  use update;
  use delete;
}
```

## Draft in CDS View Entity

```abap
" Both interface and projection views need draft-related fields:
define root view entity ZI_Delivery
  as select from zewm_delivery as hdr
" For draft: also select from draft admin view
  composition [0..*] of ZI_DeliveryItem as _Items
{
  key DeliveryNo,
      Status,
      ShipToName,
      ShipDate,
      LocalLastChangedAt,

      // Draft admin fields (system-managed):
      @UI.hidden: true
      DraftEntityCreatedBy,
      @UI.hidden: true
      DraftEntityLastChangedBy,
      @UI.hidden: true
      DraftEntityLastChangedDateTime,
      @UI.hidden: true
      IsActiveEntity,          " X = viewing active, ' ' = viewing draft
      @UI.hidden: true
      HasActiveEntity,         " X = draft has a corresponding active record
      @UI.hidden: true
      HasDraftEntity,          " X = active record has a pending draft

      _Items
}

" Draft admin fields come automatically when using 'draft table' in BDEF
" They are added to the view entity automatically by the RAP framework
```

## Draft UI Annotations (Fiori Elements)

```abap
" Projection view annotations for draft-aware Fiori Elements app:

" Header info showing draft state
@UI.headerInfo: {
  typeName: 'Delivery',
  typeNamePlural: 'Deliveries',
  title: { value: 'DeliveryNo' },
  description: { value: 'ShipToName' }
}

" Status badge on list report showing draft indicator
@UI.lineItem: [
  { position: 10, value: 'DeliveryNo' },
  { position: 20, value: 'Status', criticality: 'StatusCriticality' }
  " Draft column added automatically by Fiori Elements when draft enabled
]
```

## Draft Locking Behavior

```
Lock type: pessimistic (only one user edits at a time)

Lock acquired:    on Draft Edit action
Lock released:    on Draft Activate, Draft Discard, or session timeout

If another user tries Edit:
  → error: "Entity is locked by user <X> since <timestamp>"
  → UI shows lock owner info

Lock timeout:     configured in /n/iwfnd/v4_admin or system profile
  icm/server_port_0 → PROT=HTTPS → lock expiry = 300 seconds default
```

## Determine Action: Prepare

```abap
" Prepare = pre-validation before Activate (gives early feedback)
" Runs validations without committing — user sees errors while still editing

METHOD prepare.
  " Trigger validations manually (framework calls them anyway during Activate,
  " but Prepare lets UI show them earlier)
  MODIFY ENTITIES OF zi_delivery
    ENTITY Delivery
      EXECUTE ValidateStatus   FROM CORRESPONDING #( keys )
    FAILED   failed
    REPORTED reported.
ENDMETHOD.
```

## Draft-Aware OData V4 Requests

```
" Read active + draft (Fiori Elements does this automatically):
GET /sap/opu/odata4/.../Deliveries?$filter=IsActiveEntity eq true

" Start new draft (create draft for new entity):
POST /Deliveries
{ ShipToName: "ACME", ShipDate: "2026-06-01", IsActiveEntity: false }

" Edit existing (copy active → draft):
POST /Deliveries('0180000001')/ZI_Delivery.draftEdit
{ PreserveChanges: false }

" Auto-save change (PATCH to draft instance):
PATCH /Deliveries(DeliveryNo='0180000001',IsActiveEntity=false)
{ Status: "P" }

" Activate (save draft → active):
POST /Deliveries('0180000001')/ZI_Delivery.draftActivate

" Discard draft:
POST /Deliveries('0180000001')/ZI_Delivery.draftDiscard
```

## manifest.json for Draft-Enabled Fiori Elements App

```json
"targets": {
  "DeliveryList": {
    "type": "Component",
    "name": "sap.fe.templates.ListReport",
    "options": {
      "settings": {
        "entitySet": "Deliveries",
        "initialLoad": true,
        "navigation": {
          "Deliveries": {
            "detail": { "route": "DeliveryObjectPage" }
          }
        }
      }
    }
  },
  "DeliveryObjectPage": {
    "type": "Component",
    "name": "sap.fe.templates.ObjectPage",
    "options": {
      "settings": {
        "entitySet": "Deliveries",
        "editableHeaderContent": false,
        "navigation": {
          "to_Items": {
            "detail": { "route": "ItemObjectPage" }
          }
        }
      }
    }
  }
}
```
