# TM IDoc Integration Reference

## Key TM IDoc Message Types

| Message Type | Basic Type | Direction | Purpose |
|---|---|---|---|
| `SHPMNT` | `SHPMNT05` | Inbound/Outbound | Shipment order — core TM transport document |
| `TPSSHT` | `TPSSHT01` | Inbound | Transportation planning — FU creation from ERP |
| `IFTSTA` | `IFTSTA02` | Inbound | Status/tracking update from carrier |
| `IFCSUM` | `IFCSUM02` | Outbound | Carrier instruction / freight order to carrier |
| `DESADV` | `DELVRY07` | Inbound | Advance ship notice triggering FU creation |
| `INVOIC` | `INVOIC02` | Inbound | Carrier invoice for settlement matching |

## ERP ↔ TM Integration (Decentralized TM)

```
ERP (SAP ECC / S4HANA)        IDocs              TM (Decentralized)
────────────────────────   ────────────────   ──────────────────────
Sales Order delivery due  ──→ TPSSHT       ──→ Freight Unit creation
Purchase Order delivery   ──→ TPSSHT       ──→ Freight Unit creation
Shipment confirmation     ←── SHPMNT       ←── Freight Order complete
Goods issue trigger       ←── WMCATO       ←── From EWM via TM
Carrier invoice           ──→ INVOIC       ──→ Settlement matching
Tracking event            ←── IFTSTA       ←── Carrier EDI feed
```

## SHPMNT — Shipment IDoc

### Outbound SHPMNT (TM → Carrier / ERP)

```abap
" Create SHPMNT outbound IDoc from completed freight order
CALL FUNCTION '/SCMTMS/IDOC_SHPMNT_CREATE'
  EXPORTING
    iv_tor_id    = lv_fo_id
    iv_mestyp    = 'SHPMNT'
    iv_rcvprt    = 'LS'
    iv_rcvprn    = lv_receiver_logical_system
    iv_commit    = abap_true
  IMPORTING
    et_bapiret   = DATA(lt_messages).
```

### Inbound SHPMNT Processing

```abap
" Standard process code: SHPMNT_IN
" Handler FM: /SCMTMS/IDOC_INPUT_SHPMNT

" Custom enhancement via BADI /SCMTMS/EX_IDOC_SHPMNT:
METHOD /scmtms/if_ex_idoc_shpmnt~enrich_fo_from_idoc.
  " Map custom IDoc fields to TM freight order
  " is_e1edt20: SHPMNT header segment
  " cs_tor_root: freight order root to modify
  cs_tor_root-zz_custom_ref = is_e1edt20-zz_field.
ENDMETHOD.
```

## TPSSHT — Transportation Planning IDoc

TPSSHT is sent from ERP to TM to create freight units from delivery demand.

### Segment Structure

```
E1TPSHH  — Planning header (delivery, date, plant)
  E1TPSHE — Header extension
  E1TPSHI — Item (material, quantity, weight, volume)
    E1TPSPE — Partner (ship-to, sold-to)
    E1TPSAD — Address
```

### Custom Inbound Processor

```abap
FUNCTION z_tm_idoc_input_tpssht.
  TABLES
    idoc_contrl  STRUCTURE edidc
    idoc_data    STRUCTURE edidd
    idoc_status  STRUCTURE bdidocstat.

  DATA: ls_fu_data TYPE /scmtms/s_tor_root,
        lt_items   TYPE /scmtms/t_tor_item.

  " Parse header segment
  LOOP AT idoc_data ASSIGNING FIELD-SYMBOL(<seg>)
    WHERE segnam = 'E1TPSHH'.

    DATA(ls_hdr) = CORRESPONDING /scmtms/s_tpssht_hdr( <seg>-sdata ).

    ls_fu_data = VALUE /scmtms/s_tor_root(
      torcat      = /scmtms/if_tor_const=>gc_torcat_fu
      src_loc_id  = ls_hdr-src_location
      dst_loc_id  = ls_hdr-dst_location
      req_date    = ls_hdr-required_date ).

    CALL METHOD /scmtms/cl_tor_bo_factory=>create_tor
      EXPORTING
        is_root   = ls_fu_data
        it_items  = lt_items
      IMPORTING
        ev_tor_id = DATA(lv_fu_id)
        et_return = DATA(lt_msg).

    IF line_exists( lt_msg[ type = 'E' ] ).
      idoc_status-status = '51'.  " Error
    ELSE.
      idoc_status-status = '53'.  " Posted
    ENDIF.
  ENDLOOP.
ENDFUNCTION.
```

## IFTSTA — Carrier Status/Tracking

```abap
" Standard inbound: /SAPTRX/IDOC_INPUT_IFTSTA
" Maps carrier status codes to TM event codes

" Custom mapping in BADI /SAPTRX/EX_IFTSTA_MAP:
METHOD /saptrx/if_ex_iftsta_map~map_event_code.
  " Map carrier-specific status codes to standard TM events
  CASE iv_carrier_status.
    WHEN '11'. rv_event_code = 'DEPARTURE'.
    WHEN '17'. rv_event_code = 'ARRIVAL'.
    WHEN '44'. rv_event_code = 'POD'.
    WHEN '96'. rv_event_code = 'EXCEPTION'.
    WHEN OTHERS. rv_event_code = 'IN_TRANSIT'.
  ENDCASE.
ENDMETHOD.
```

## IFCSUM — Freight Order to Carrier (EDI)

```abap
" Send freight order details to carrier via EDI
CALL FUNCTION '/SCMTMS/IFCSUM_CREATE'
  EXPORTING
    iv_tor_id    = lv_fo_id
    iv_carrier   = lv_carrier_bp
    iv_commit    = abap_true
  IMPORTING
    et_return    = DATA(lt_messages).
```

## Partner Profile Setup (WE20)

```
WE20 → Partner type LS (Logical System) or LI (Carrier BP)

Inbound:
  TPSSHT → Process code: TPSSHT_IN (or Z custom FM)
  SHPMNT → Process code: SHPMNT_IN
  IFTSTA → Process code: IFTSTA_IN (track event posting)
  INVOIC → Process code: INVOIC_IN (settlement matching)

Outbound:
  SHPMNT → Triggered on FO completion / PPF action
  IFCSUM → Triggered on carrier assignment / tendering
```

## IDoc Monitoring

| Transaction | Purpose |
|---|---|
| `WE02` | IDoc list — filter by message type or status |
| `WE05` | IDoc list with status filter |
| `WE19` | Test inbound IDoc processing |
| `BD87` | Reprocess failed inbound IDocs (status 51) |
| `WE20` | Partner profiles |

### Query Failed TM IDocs

```abap
SELECT docnum, mestyp, direct, status, credat
  FROM edidc
  WHERE mestyp IN ('SHPMNT', 'TPSSHT', 'IFTSTA', 'INVOIC')
    AND status IN ('51', '56', '68')
    AND credat >= @lv_from_date
  ORDER BY credat DESCENDING
  INTO TABLE @DATA(lt_failed).
```

## ALE Distribution Model (BD64)

For decentralized TM, configure distribution:

```
BD64 → Add message types:
  TPSSHT: ERP → TM   (demand transfer)
  SHPMNT: TM  → ERP  (confirmation)
  IFTSTA: Carrier → TM (via middleware)

SALE → Logical systems:
  Define ERP system (e.g. PRD_800)
  Define TM system  (e.g. TMS_100)
  Define carrier EDI system

SM59 → RFC destinations:
  ERP ↔ TM RFC connection (type 3 = ABAP)
```
