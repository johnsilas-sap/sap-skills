# OData Backend for Fiori Reference

## Backend Options

| Option | Stack | Best For |
|---|---|---|
| SEGW (OData V2) | ABAP | Legacy S/4, ECC, EWM on-premise |
| CDS + ABAP (OData V4) | ABAP | S/4HANA 2022+, modern ABAP |
| CAP (OData V4) | Node.js/Java on BTP | Cloud-first, greenfield, BTP apps |
| S/4HANA VDM APIs | OData V2/V4 | Use existing SAP standard APIs |

## SEGW — OData V2 Service (ABAP)

```
SEGW → Create Project
  Name:        ZEWM_DELIVERY_SRV
  Namespace:   /EWM/
  Description: EWM Delivery Service for Fiori

Step 1: Import from ABAP Dictionary
  Right-click Entity Types → Create → From ABAP Structure
  Structure:   ZEWM_DELIVERY_HDR    (ABAP structure)
  Entity Type: Delivery
  Entity Set:  DeliverySet
  Key Fields:  DeliveryNo

Step 2: Create Navigation Property
  Entity Type: Delivery → Navigation Properties → Create
  Name:        to_Items
  Target:      DeliveryItem (1:N relationship)

Step 3: Implement DPC (Data Provider Class)
  Runtime Artifacts → Generate → Data Provider Extension Class: ZEWM_DELIVERY_SRV_DPC_EXT

Step 4: Activate and Publish
  SEGW → Generate → Runtime Objects
  /n/IWFND/MAINT_SERVICE → Add Service → Technical Service Name: ZEWM_DELIVERY_SRV

Step 5: Test
  /sap/opu/odata/sap/ZEWM_DELIVERY_SRV/$metadata
  /sap/opu/odata/sap/ZEWM_DELIVERY_SRV/DeliverySet
```

### SEGW DPC Extension (Data Provider Class)

```abap
CLASS zewm_delivery_srv_dpc_ext DEFINITION
  INHERITING FROM zewm_delivery_srv_dpc
  PUBLIC FINAL CREATE PUBLIC.

  PUBLIC SECTION.
    METHODS:
      deliveryset_get_entityset REDEFINITION,
      deliveryset_get_entity     REDEFINITION,
      goodsreceiptset_create_entity REDEFINITION.
ENDCLASS.

CLASS zewm_delivery_srv_dpc_ext IMPLEMENTATION.

  METHOD deliveryset_get_entityset.
    " Called for GET /DeliverySet?$filter=...
    DATA lt_deliveries TYPE TABLE OF zewm_delivery_hdr.
    
    " Read filter parameters
    DATA(lv_status) = VALUE #( it_filter_select_options[
      sign = 'I' option = 'EQ' low = '' " read Status filter
    ] OPTIONAL ).

    SELECT * FROM zewm_delivery_hdr
      INTO TABLE @lt_deliveries
      WHERE status = 'Open'
      ORDER BY ship_date ASCENDING.

    " Map to entity structure
    LOOP AT lt_deliveries ASSIGNING FIELD-SYMBOL(<dlv>).
      APPEND CORRESPONDING #( <dlv> ) TO et_entityset.
    ENDLOOP.
  ENDMETHOD.

  METHOD goodsreceiptset_create_entity.
    " Called for POST /GoodsReceiptSet
    DATA ls_gr TYPE zewm_goods_receipt.
    io_data_provider->read_entry_data( IMPORTING es_request_entity = ls_gr ).
    
    " Call EWM API to post GR
    CALL FUNCTION 'ZEWM_POST_GOODS_RECEIPT'
      EXPORTING is_gr_data = ls_gr
      EXCEPTIONS OTHERS = 1.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE /iwbep/cx_mgw_busi_exception
        MESSAGE id sy-msgid type sy-msgty number sy-msgno
                with sy-msgv1 sy-msgv2 sy-msgv3 sy-msgv4.
    ENDIF.

    er_entity = CORRESPONDING #( ls_gr ).
  ENDMETHOD.

ENDCLASS.
```

## CDS-Based OData V4 (S/4HANA 2022+)

```abap
" In ABAP CDS — @OData.publish: true generates V4 service automatically
@OData.publish: true
@ObjectModel.transactionalProcessingEnabled: true
define root view entity ZC_Delivery
  as select from ZI_Delivery
{
  key DeliveryNo,
      Status,
      ShipToName,
      ShipDate,
      TotalWeight,
      WeightUOM,

  // Navigation — composition (items owned by delivery)
  _Items
}
```

```abap
" Business Object (BOPF-based for draft/lock handling)
" Or: RAP (RESTful ABAP Programming Model) for modern approach

" RAP behavior definition:
define behavior for ZC_Delivery alias Delivery
  implementation in class ZBP_Delivery unique
  persistent table ZEWM_DELIVERY
  draft table ZEWM_DELIVERY_D
  lock master total etag LocalLastChangedAt
{
  create; update; delete;
  
  action PostGoodsIssue result [1] $self;
  action ConfirmDelivery parameter ZA_ConfirmParam result [1] $self;
  
  association _Items { create; with draft; }
}
```

## CAP OData V4 (BTP / Cloud)

```javascript
// srv/delivery-service.js (Node.js CAP)
const cds = require('@sap/cds');

module.exports = cds.service.impl(async function() {

  const { Deliveries, Items } = this.entities;

  // Custom handler: before READ — add filter
  this.before('READ', Deliveries, req => {
    if (!req.query.SELECT.where) {
      req.query.where({ Status: 'Open' });
    }
  });

  // Custom action: Post Goods Issue
  this.on('PostGoodsIssue', async req => {
    const { DeliveryNo } = req.params[0];
    const delivery = await SELECT.one.from(Deliveries).where({ DeliveryNo });

    if (!delivery) req.error(404, `Delivery ${DeliveryNo} not found`);
    if (delivery.Status !== 'Open') req.error(400, 'Delivery is not open');

    await UPDATE(Deliveries).set({ Status: 'Completed' }).where({ DeliveryNo });
    return await SELECT.one.from(Deliveries).where({ DeliveryNo });
  });

});
```

## S/4HANA VDM APIs (Standard — No Custom Dev)

Common S/4HANA API Business Hub services ready for Fiori:

| Service | Path | Content |
|---|---|---|
| `API_OUTBOUND_DELIVERY_SRV` | `/sap/opu/odata/sap/API_OUTBOUND_DELIVERY_SRV/` | Outbound deliveries |
| `API_INBOUND_DELIVERY_SRV` | `/sap/opu/odata/sap/API_INBOUND_DELIVERY_SRV/` | Inbound / GR |
| `API_SALES_ORDER_SRV` | `/sap/opu/odata/sap/API_SALES_ORDER_SRV/` | Sales orders |
| `API_MATERIAL_STOCK_SRV` | `/sap/opu/odata/sap/API_MATERIAL_STOCK_SRV/` | Stock |
| `API_PURCHASEORDER_PROCESS_SRV` | `/sap/opu/odata/sap/API_PURCHASEORDER_PROCESS_SRV/` | POs |

```
" Activate in /n/IWFND/MAINT_SERVICE
" Or: Communication Arrangement (S/4HANA Cloud)
```

## OData Function Import (V2) / Bound Action (V4)

### V2 Function Import (SEGW)

```abap
" In SEGW → Function Imports → Create
Name:         PostGoodsIssue
HTTP Method:  POST
Return Type:  Boolean
Parameters:
  DeliveryNo  Edm.String  In
  PostingDate Edm.DateTime In

" DPC implementation:
METHOD postgoodsissue_func_imp.
  DATA ls_param TYPE zewm_gi_param.
  io_tech_request_context->get_keys( IMPORTING es_key_tab = ls_param ).
  " ... call EWM GI posting ...
  copy_data_to_ref( EXPORTING is_data = abap_true CHANGING cr_data = er_data ).
ENDMETHOD.
```

### V4 Bound Action (CDS/RAP)

```
" Automatically exposed as POST /Deliveries(DeliveryNo='0180000001')/PostGoodsIssue
```

```javascript
// Call in SAPUI5 controller (V4 model):
var oModel = this.getModel();
var oOperation = oModel.bindContext(
  "PostGoodsIssue(...)",
  this.getView().getBindingContext()
);
oOperation.execute().then(function() {
  sap.m.MessageToast.show("GI posted successfully");
}).catch(function(oError) {
  sap.m.MessageBox.error(oError.message);
});
```

## Registering OData Service in Gateway

```
/n/IWFND/MAINT_SERVICE → Add Service
  System Alias:     LOCAL (same system) or RFC alias for remote
  External Service Name: ZEWM_DELIVERY_SRV
  → Test Service → should return $metadata XML

" Then accessible at:
https://my-s4.ondemand.com/sap/opu/odata/sap/ZEWM_DELIVERY_SRV/
```
