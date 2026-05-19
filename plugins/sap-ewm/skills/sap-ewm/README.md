# SAP EWM Development Skill

ABAP development skill for SAP Extended Warehouse Management (EWM) — decentralized and embedded on S/4HANA.

## Skill Overview

Covers end-to-end EWM development including:

- **Warehouse Structure**: Warehouse numbers, storage types, sections, bins, quants
- **Transfer Orders / Warehouse Tasks**: Create, confirm, cancel via `/SCWM/TO_CREATE_TD`, `/SCWM/TO_CONFIRM_TD`
- **Warehouse Orders**: Grouping, queue assignment, resource routing
- **RF Mobile (ITSmobile)**: RF transaction config, screen logic, resource login, Velocity/Ivanti integration
- **BADIs & Enhancement Spots**: Bin determination, RF screen hooks, quant changes, delivery PPF
- **Delivery Processing**: Inbound/outbound via `/SCDL/` API, goods receipt, goods issue
- **Wave Management**: Create and release waves for coordinated picking
- **Physical Inventory**: Document creation, count posting, differences
- **Queue Management**: Resource group routing, queue assignment
- **Performance Monitoring**: ST13 PERF_TOOL, EWM_RF_Analysis, frontend timing (SAP Notes 1690850, 1595305)
- **IDoc Integration**: WMMBXY, SHPMNT, DESADV message types

## Auto-Trigger Keywords

### Core EWM Concepts
- EWM, Extended Warehouse Management, warehouse management
- warehouse number, LGNUM, storage type, LGTYP
- storage section, LGBER, storage bin, LGPLA
- quant, /SCWM/QUAN, stock in bin
- warehouse task, WT, warehouse order, WO
- /SCWM/WHO, /SCWM/WHT, /SCWM/ORDIM_O, /SCWM/ORDIM_C

### Transfer Orders
- transfer order, TO, WT creation, task creation
- /SCWM/TO_CREATE_TD, /SCWM/TO_CONFIRM_TD
- putaway, pick, replenishment, stock transfer
- source bin, destination bin, movement type

### RF Mobile
- RF mobile, ITSmobile, RF transaction, barcode scanner
- Velocity, Ivanti, Slipstream, handheld scanner
- /SCWM/RFUI, RF logical transaction, RF screen
- /SCWM/CL_RF_BLL_PROCESSOR, RF business logic
- resource, /SCWM/RSRC, RF login, RF user

### BADIs and Enhancements
- BADI, enhancement spot, EWM enhancement
- /SCWM/EX_RF_BLL_PROCESSOR, /SCWM/EX_TO_CREATE
- /SCWM/EX_CORE_DETERMINATION, /SCWM/EX_QUAN_CHANGE
- bin determination override, putaway strategy
- /SCWM/ES_RF_BLL_PROCESSOR, /SCWM/ES_CORE_DETERMINATION

### Delivery Processing
- inbound delivery, outbound delivery, EWM delivery
- /SCDL/DB_PROCI_O, /SCDL/DB_PROCO_O
- goods receipt, GR posting, /SCWM/GR_POST
- goods issue, GI posting, /SCWM/GI_POST
- /SCWM/PRDI, /SCWM/PRDO, delivery item

### Wave and Queue Management
- wave, wave management, /SCWM/WAVE_CREATE, /SCWM/WAVE_RELEASE
- queue, queue assignment, /SCWM/TQACT
- resource group, work center, /SCWM/WHO_QUEUE_ASSIGN

### Physical Inventory
- physical inventory, cycle count, bin count
- /SCWM/PI_DOC_CREATE, /SCWM/PI_COUNT_POST
- inventory difference, recount

### Performance and Monitoring
- EWM performance, RF performance, GUI response time
- ST13, PERF_TOOL, EWM_RF_Analysis, Load GUI
- SAP Note 1690850, SAP Note 1595305
- ewmrf_store_gui, /SCMB/PFM_RFUI
- ICF handler, /sap/bc/ewmrf_perf

### EWM Transactions
- /SCWM/MON, warehouse monitor
- /SCWM/LGNUM, /SCWM/PRDI, /SCWM/PRDO
- /SCWM/TO01, /SCWM/TO09, /SCWM/RSRC

### IDoc Integration
- WMMBXY, SHPMNT, DESADV, EWM IDoc
- warehouse IDoc, EWM interface, goods movement IDoc

## Directory Structure

```
sap-ewm/
├── SKILL.md                              # Main skill — quick reference and code patterns
├── README.md                             # This file
└── references/
    ├── warehouse-structure.md            # Org structure, tables, quant management
    ├── transfer-orders.md                # WT/WO lifecycle, APIs, confirmation
    ├── rf-mobile.md                      # ITSmobile, Velocity, screen config, BADIs
    ├── badi-enhancements.md              # All EWM BADIs with signatures and examples
    ├── delivery-processing.md            # Inbound/outbound delivery, GR/GI APIs
    ├── performance-monitoring.md         # ST13, PERF_TOOL, frontend timing tool
    └── idoc-integration.md              # EWM IDoc types, partner profiles, error handling
```

## Source Documentation

- SAP EWM Help: https://help.sap.com/docs/SAP_EXTENDED_WAREHOUSE_MANAGEMENT
- S/4HANA EWM: https://help.sap.com/docs/S4HANA_ON-PREMISE
- SAP Note 1690850 (RFUI performance)
- SAP Note 1595305 (frontend upload interface)

## Version

- **Skill Version**: 1.0.0
- **Last Updated**: 2026-05-19
- **EWM Release**: EWM 9.4+ / S/4HANA EWM 2020+
- **Reference Files**: 7

---

## License

GPL-3.0 License — see LICENSE file in repository root.
