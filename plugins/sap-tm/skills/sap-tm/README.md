# SAP TM Development Skill

ABAP development skill for SAP Transportation Management — SAP TM 9.x and embedded TM in S/4HANA.

## Skill Overview

Covers end-to-end TM development including:

- **Document Model**: Freight units, freight orders, freight bookings, transportation orders, stages
- **Carrier Selection**: Automatic carrier selection, ranking BADIs, carrier assignment APIs
- **Charge Calculation**: Rate agreements, surcharges, custom charge BADIs, `/SCMTMS/D_CHARGE`
- **Transportation Planning**: Freight unit creation, planning profiles, planning cockpit APIs
- **Dangerous Goods**: DG classification, compatibility checks, DG BADI
- **Track and Trace**: Event posting, `/SAPTRX/` APIs, tracking history
- **Settlement**: Freight invoice, settlement document creation, FI/CO posting
- **Output / Forms**: PPF-driven forms (SmartForms, Adobe), EDI output, carrier notification
- **BADIs & Enhancement Spots**: FO change, charge calc, carrier selection, planning, settlement
- **IDoc Integration**: SHPMNT, TPSSHT message types, partner profiles, error handling
- **EWM Integration**: Freight order → warehouse order handoff, goods issue trigger

## Auto-Trigger Keywords

### Core TM Concepts
- TM, Transportation Management, SAP TM
- freight order, FO, freight unit, FU
- freight booking, FB, transportation order, TO
- /SCMTMS/D_FRO_H, /SCMTMS/D_FRO_I, /SCMTMS/D_STAGE
- /SCMTMS/D_TOR_H, /SCMTMS/D_FUB_H
- stage, leg, means of transport, carrier

### Carrier Selection
- carrier selection, carrier assignment, carrier ranking
- /SCMTMS/CARRIER_SEL_EXECUTE, /SCMTMS/FRO_CARRIER_ASSIGN
- carrier profile, preferred carrier, freight agreement
- /SCMTMS/IF_EX_CARRIER_SEL, carrier BADI

### Charge Calculation
- charge calculation, freight charge, surcharge, rate
- /SCMTMS/CHARGE_CALC_EXECUTE, /SCMTMS/D_CHARGE
- freight agreement, tariff, rate table
- /SCMTMS/IF_EX_CHARGE_CALC, charge BADI
- fuel surcharge, accessorial, detention

### Transportation Planning
- transportation planning, freight unit, planning board
- /SCMTMS/TP_LAND, /SCMTMS/TP_SEA, /SCMTMS/TP_AIR
- planning profile, optimization, consolidation
- /SCMTMS/PLANNING_EXECUTE, /SCMTMS/FU_CREATE_FROM_SO

### Dangerous Goods
- dangerous goods, DG, hazmat, ADR, IMDG, IATA
- DG classification, UN number, packing group
- /SCMTMS/IF_EX_DG_CHECK, DG BADI, DG check

### Track and Trace
- track and trace, tracking event, shipment tracking
- /SAPTRX/TRACK_EVENT_POST, /SAPTRX/APPL_OB
- departure event, arrival event, exception event
- geolocation, GPS, carrier update

### Settlement
- settlement, freight invoice, carrier invoice
- /SCMTMS/SETTLEMENT_CREATE, /SCMTMS/D_SETTL
- accruals, FI posting, CO posting
- /SCMTMS/IF_EX_SETTLEMENT, settlement BADI

### BADIs and Enhancements
- /SCMTMS/IF_EX_FRO_CHANGE, /SCMTMS/ES_FRO_CHANGE
- /SCMTMS/IF_EX_CHARGE_CALC, /SCMTMS/ES_CHARGE_CALC
- /SCMTMS/IF_EX_CARRIER_SEL, /SCMTMS/ES_CARRIER_SEL
- /SCMTMS/IF_EX_PLANNING, /SCMTMS/ES_PLANNING
- /SCMTMS/IF_EX_FU_CREATION, /SCMTMS/IF_EX_SETTLEMENT
- /SCMTMS/IF_EX_OUTPUT, /SCMTMS/IF_EX_DG_CHECK

### IDoc Integration
- SHPMNT, TPSSHT, TM IDoc, shipment IDoc
- /SCMTMS/IDOC_INPUT_SHPMNT, DESADV, TM interface
- partner profile, WE20, inbound shipment

### TM Transactions
- /SCMTMS/MON_FO, freight order monitor
- /SCMTMS/TOC, transportation order cockpit
- /SCMTMS/FA, freight agreement
- /SCMTMS/CHARGE, /SCMTMS/SETTL, /SCMTMS/CARRIER

### EWM Integration
- TM EWM integration, freight order warehouse order
- /SCWM/GI_POST trigger, goods issue from TM
- delivery handoff, staging area, dock appointment

## Directory Structure

```
sap-tm/
├── SKILL.md                              # Main skill — document model, APIs, BADIs
├── README.md                             # This file
└── references/
    ├── document-model.md                 # FU, FO, FB, TO lifecycle and relationships
    ├── carrier-selection.md              # Carrier selection, ranking, agreement APIs
    ├── charge-calculation.md             # Rate lookup, surcharges, charge BADI
    ├── planning.md                       # Transportation planning, optimization
    ├── dangerous-goods.md                # DG classification, checks, BADI
    ├── track-and-trace.md                # Event posting, /SAPTRX/, carrier updates
    └── idoc-integration.md               # SHPMNT/TPSSHT, partner profiles, ERP sync
```

## Source Documentation

- SAP TM Help: https://help.sap.com/docs/SAP_TRANSPORTATION_MANAGEMENT
- S/4HANA TM: https://help.sap.com/docs/S4HANA_ON-PREMISE

## Version

- **Skill Version**: 1.0.0
- **Last Updated**: 2026-05-19
- **TM Release**: SAP TM 9.4+ / S/4HANA TM 2020+
- **Reference Files**: 7

---

## License

GPL-3.0 License — see LICENSE file in repository root.
