# Common IDoc Message Types Reference

## Procurement (MM)

### ORDERS — Purchase Order

```
Message type:  ORDERS
Basic type:    ORDERS05
Direction:     Outbound (ERP → Vendor)

Key segments:
  E1EDK01  Header (document type, currency, payment terms)
  E1EDKA1  Partners (vendor, ship-to, bill-to)
  E1EDK14  Organizational data (purchasing org, company code)
  E1EDP01  Item (material, quantity, UOM, delivery date)
  E1EDP19  Item text
  E1EDPT1  Delivery schedule
  E1EDP26  Invoice detail (price, conditions)
```

```abap
" Read ORDERS inbound — header
DATA ls_e1edk01 TYPE e1edk01.
READ TABLE idoc_data ASSIGNING FIELD-SYMBOL(<s>)
  WITH KEY segnam = 'E1EDK01' docnum = <ctrl>-docnum.
MOVE <s>-sdata TO ls_e1edk01.
" ls_e1edk01-belnr = PO number
" ls_e1edk01-ntgew = Net weight
" ls_e1edk01-gewei = Weight UOM
" ls_e1edk01-curcy = Currency
```

### INVOIC — Vendor Invoice (Inbound)

```
Message type:  INVOIC
Basic type:    INVOIC02
Direction:     Inbound (Vendor → ERP) for logistics invoice verification

Key segments:
  E1EDK01  Invoice header (doc type, currency)
  E1EDKA1  Partners (vendor, payer)
  E1EDP01  Invoice item
  E1EDP26  Pricing (amount, tax)
  E1EDKT1  Header text
```

## Sales (SD)

### ORDERS (Inbound Sales Order)

```
Message type:  ORDERS
Basic type:    ORDERS05
Direction:     Inbound (Customer EDI → ERP)
Process code:  ORDE

Key segments: same structure as MM ORDERS
  E1EDKA1-PARVW = 'AG' (sold-to), 'WE' (ship-to), 'RE' (bill-to)
```

### DELVRY — Delivery

```
Message type:  DELVRY
Basic type:    DELVRY07
Direction:     Outbound (ERP → Customer or 3PL)

Key segments:
  E1EDL20  Delivery header (delivery number, ship date)
  E1EDKA1  Partners
  E1EDL24  Delivery item (material, quantity, batch)
  E1EDL44  Handling unit reference
```

## Warehouse (EWM)

### WMMBXY — Goods Movement

```
Message type:  WMMBXY
Basic type:    WMMBXY01
Direction:     Inbound (ERP → EWM) for stock synchronization

Key segments:
  E1MBXYH  Header (warehouse, movement type, posting date)
  E1MBXYI  Item (material, plant, quantity, batch, storage location)
```

```abap
" Parse WMMBXY header
DATA ls_hdr TYPE e1mbxyh.
READ TABLE idoc_data ASSIGNING FIELD-SYMBOL(<h>)
  WITH KEY segnam = 'E1MBXYH' docnum = <ctrl>-docnum.
MOVE <h>-sdata TO ls_hdr.
" ls_hdr-lgnum    = Warehouse number
" ls_hdr-bwlvs    = Movement type
" ls_hdr-budat    = Posting date
```

### WMCATO — Transfer Order Confirmation

```
Message type:  WMCATO
Basic type:    WMCATO01
Direction:     Outbound (EWM → ERP) confirming TO completion

Key segments:
  E1WMCAT  TO confirmation header
  E1WMCAI  Confirmed quantities per item
```

### DESADV — Advance Ship Notice (ASN)

```
Message type:  DESADV
Basic type:    DELVRY07
Direction:     Inbound (Vendor/Carrier → EWM) triggers inbound delivery

Key segments:
  E1EDL20  ASN header (delivery number, vendor ref)
  E1EDL24  Item (material, quantity, batch, HU reference)
  E1EDL44  Handling unit (pallet ID, packaging)
  E1EDKA1  Partners (vendor, ship-from, ship-to)
```

```abap
" DESADV inbound — standard process code: DESADV
" Handler: /SCWM/IDOC_INPUT_DESADV

" Read HU from DESADV
LOOP AT idoc_data ASSIGNING FIELD-SYMBOL(<hu>)
  WHERE segnam = 'E1EDL44' AND docnum = <ctrl>-docnum.
  DATA ls_hu TYPE e1edl44.
  MOVE <hu>-sdata TO ls_hu.
  " ls_hu-vpobjkey = HU number (pallet ID)
  " ls_hu-tarag    = Tare weight
ENDLOOP.
```

## Transportation (TM)

### SHPMNT — Shipment

```
Message type:  SHPMNT
Basic type:    SHPMNT05
Direction:     Inbound/Outbound

Key segments:
  E1EDT20  Shipment header (shipment type, carrier, dates)
  E1EDKA1  Partners (carrier, shipper, consignee)
  E1EDT21  Delivery reference
  E1EDT22  Stage (leg)
```

### TPSSHT — Transportation Planning

```
Message type:  TPSSHT
Basic type:    TPSSHT01
Direction:     Inbound (ERP → TM) for freight unit creation

Key segments:
  E1TPSHH  Planning header (delivery, date)
  E1TPSHI  Item (material, quantity, weight, volume)
  E1TPSPE  Partner
```

### IFTSTA — Status Message (Track & Trace)

```
Message type:  IFTSTA
Basic type:    IFTSTA02
Direction:     Inbound (Carrier EDI → TM)
               Reports departure, arrival, POD events

Key segments:
  E1IFTSHH  Header (shipment reference, carrier)
  E1IFTSRR  Status record (event code, date, time, location)
```

## Master Data (ALE)

### MATMAS — Material Master

```
Message type:  MATMAS
Basic type:    MATMAS05
Direction:     Outbound (Central ERP → Receiving systems)
Process code:  MATM (inbound)

Key segments:
  E1MARAM  Material header (material number, type, base UOM)
  E1MAKTM  Descriptions (language, description text)
  E1MARMM  Units of measure
  E1MBEWM  Valuation (standard price, moving average)
  E1MLAGM  Warehouse management (storage type indicator)
```

### DEBMAS — Customer Master

```
Message type:  DEBMAS
Basic type:    DEBMAS06
Process code:  DEBI (inbound)

Key segments:
  E1KNA1M  General customer data
  E1KNB1M  Company code data
  E1KNVVM  Sales area data
  E1KNVPM  Partner functions
```

### CREMAS — Vendor Master

```
Message type:  CREMAS
Basic type:    CREMAS05
Process code:  CRED (inbound)

Key segments:
  E1LFA1M  General vendor data
  E1LFB1M  Company code data
  E1LFBKM  Bank details
  E1LFM1M  Purchasing data
```

## Segment Field Reference Quick Lookup

### Partner Roles (E1EDKA1-PARVW)

| Value | Partner Role |
|---|---|
| `AG` | Sold-to party |
| `WE` | Ship-to party |
| `RE` | Bill-to party |
| `RG` | Payer |
| `LF` | Vendor |
| `SP` | Freight carrier |
| `LS` | Logical system |

### Movement Types (E1MBXYH-BWLVS — EWM)

| Value | Movement |
|---|---|
| `101` | Goods receipt from PO |
| `261` | Goods issue for production order |
| `311` | Transfer posting (plant to plant) |
| `551` | Scrapping |
| `561` | Initial stock entry |
| `601` | Goods issue for delivery |
