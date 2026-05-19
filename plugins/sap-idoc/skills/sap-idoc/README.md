# SAP IDoc Development Skill

ABAP development skill for SAP IDoc (Intermediate Document) — inbound/outbound processing, custom types, ALE/EDI configuration, error handling, and cross-module integration.

## Skill Overview

- **IDoc Structure**: Control records (EDIDC), data records (EDIDD), status records (EDIDS)
- **Inbound Processing**: Standard FM signature, segment parsing, status posting, error handling
- **Outbound Processing**: Building control/data records, `MASTER_IDOC_DISTRIBUTE`, tRFC dispatch
- **Custom IDoc Types**: WE31 (segments) → WE30 (basic type) → WE82 → WE57 full workflow
- **Z-Segment Extensions**: Extending standard IDocs (ORDERS05, DELVRY07) with custom fields
- **Partner Profiles**: WE20 configuration for LS/LI/KU/LF partners, inbound/outbound params
- **ALE Distribution**: BD64 distribution model, BD87 reprocessing, logical system setup
- **Error Handling**: Status 51/56/68 query, BD87 reprocessing, `IDOC_STATUS_WRITE_TO_DATABASE`
- **Common Message Types**: ORDERS, INVOIC, DESADV, SHPMNT, WMMBXY, DELVRY, TPSSHT, IFTSTA
- **Cross-Module**: EWM IDocs, TM IDocs, MM/SD/FI standard types

## Auto-Trigger Keywords

### Core IDoc Concepts
- IDoc, intermediate document, ALE, EDI
- EDIDC, EDIDD, EDIDS, control record, data record, status record
- message type, basic type, IDoc type, segment type
- DOCNUM, MESTYP, IDOCTP, BASIC_TYPE, DIRECT, STATUS

### Inbound Processing
- inbound IDoc, process code, WE42, WE57
- IDOC_INPUT, inbound function module, inbound processor
- BDIDOCSTAT, idoc_status, status 51, status 53
- MASS_PROCESSING, INPUT_METHOD, IDOC_CONTRL, IDOC_DATA
- BD87, reprocess, failed IDoc, error IDoc

### Outbound Processing
- outbound IDoc, MASTER_IDOC_DISTRIBUTE, IDoc dispatch
- tRFC, SM58, output type, message control, NAST
- IDoc port, RFC destination, file port
- status 12, status 03, collect IDocs

### Custom IDoc Types
- WE31, segment type, IDoc segment
- WE30, basic IDoc type, IDoc structure
- WE82, message type assignment
- WE57, function module assignment
- Z-segment, custom segment, IDoc extension
- SDATA, segment data, 1000 bytes

### Partner Profiles
- WE20, partner profile, partner type
- LS, logical system, LI, vendor partner, KU, customer partner
- inbound parameter, outbound parameter, process code
- receiver port, output mode, transfer immediately

### ALE and Distribution
- ALE, Application Link Enabling
- BD64, distribution model
- SALE, logical system, RFC destination
- BD87, reprocess, mass reprocessing
- SM58, tRFC monitor, stuck IDocs

### Error Handling
- status 51, error in application, IDoc error
- status 56, IDoc with errors, status 68
- IDOC_STATUS_WRITE_TO_DATABASE, manual status update
- WE19, IDoc test tool, simulate inbound

### Common Message Types
- ORDERS, purchase order, sales order IDoc
- INVOIC, invoice IDoc, billing IDoc
- DESADV, advance ship notice, ASN IDoc
- DELVRY, delivery IDoc, DELVRY07
- SHPMNT, shipment IDoc, SHPMNT05
- WMMBXY, goods movement IDoc, warehouse IDoc
- WMCATO, transfer order confirmation
- TPSSHT, transportation planning IDoc
- IFTSTA, status message, tracking IDoc
- IFCSUM, carrier instruction IDoc
- MATMAS, material master IDoc
- DEBMAS, customer master IDoc
- CREMAS, vendor master IDoc
- HRMD_A, HR master data IDoc

## Directory Structure

```
sap-idoc/
├── SKILL.md                              # Main skill — structure, inbound/outbound patterns
├── README.md                             # This file
└── references/
    ├── idoc-structure.md                 # Control/data/status records, segment anatomy
    ├── inbound-processing.md             # FM signature, segment parsing, status, errors
    ├── outbound-processing.md            # Building IDocs, distributing, tRFC, ports
    ├── custom-idoc-types.md              # WE31/WE30/WE82/WE57, Z-extensions
    ├── partner-profiles-ale.md           # WE20, BD64, logical systems, distribution model
    ├── error-handling.md                 # BD87, status codes, reprocessing, monitoring
    └── common-message-types.md           # ORDERS, INVOIC, DESADV, WMMBXY, SHPMNT, DELVRY
```

## Source Documentation

- SAP IDoc Help: https://help.sap.com/docs/ABAP_PLATFORM_NEW/0b9668e854374d8fa3fc8ec327ff3693
- SAP ALE Guide: https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE

## Version

- **Skill Version**: 1.0.0
- **Last Updated**: 2026-05-19
- **Reference Files**: 7

---

## License

GPL-3.0 License — see LICENSE file in repository root.
