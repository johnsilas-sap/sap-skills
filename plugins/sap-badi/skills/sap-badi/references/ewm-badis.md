# EWM BAdI Catalog

## Enhancement Spot Overview

| Enhancement Spot | Area |
|---|---|
| `/SCWM/ES_DLV_PROC` | Delivery processing |
| `/SCWM/ES_WHO` | Warehouse order creation/confirmation |
| `/SCWM/ES_WHT` | Warehouse task creation/confirmation |
| `/SCWM/ES_PICK_HO` | Pick handling unit |
| `/SCWM/ES_PACK` | Packing |
| `/SCWM/ES_RSRC` | Resource management |
| `/SCWM/ES_RF` | RF mobile processing |
| `/SCWM/ES_PP_CTRL` | Physical inventory |
| `/SCWM/ES_WAVE` | Wave management |
| `/SCWM/ES_QINSP` | Quality inspection |

## Delivery Processing BAdIs (`/SCWM/ES_DLV_PROC`)

### `/SCWM/EX_DLV_PROC_DLV` — Delivery Header Processing

```
Methods:
  CHECK_DELIVERY      Validate delivery before save — add errors to ET_MESSAGES, set RV_REJECT
  SET_DELIVERY_DATA   Modify delivery header data before posting (CS_HEADER CHANGING)
  CLOSE_DELIVERY      Called when delivery is goods-issued / closed

Filter: LGNUM

Typical use:
  - Validate custom fields on delivery header
  - Set carrier-specific fields
  - Trigger external notifications on GI
```

### `/SCWM/EX_DLV_PROC_ITEM` — Delivery Item Processing

```
Methods:
  CHECK_ITEM          Validate individual delivery item
  SET_ITEM_DATA       Modify item data before posting (CS_ITEM CHANGING)

Filter: LGNUM

Typical use:
  - Cross-check item quantities against ERP data
  - Populate custom item fields (Z-fields in /SCWM/S_DLV_ITEM_EXT)
```

### `/SCWM/EX_DLV_PROC_PACK` — Packing BAdI

```
Methods:
  CHECK_PACK          Validate packing action before saving
  AFTER_PACK          Run custom logic after packing completes

Filter: LGNUM

Typical use:
  - Enforce packing rules (max weight, mixed-item restrictions)
  - Trigger label printing after packing
  - Write audit log
```

## Warehouse Order BAdIs (`/SCWM/ES_WHO`)

### `/SCWM/EX_WHO_CREATION` — Warehouse Order Creation

```
Methods:
  DEF_WHO_ATTR        Set WO attributes before creation (priority, resource type)
  CHECK_WHO           Validate WO before creation — can reject

Filter: LGNUM

Typical use:
  - Override default priority based on customer tier
  - Restrict WO to specific resource types
  - Split WOs based on weight/volume limits
```

### `/SCWM/EX_WHO_CONF` — Warehouse Order Confirmation

```
Methods:
  BEFORE_CONFIRMATION  Run logic before WO is confirmed (can block)
  AFTER_CONFIRMATION   Run logic after WO confirmation (audit, external notification)

Filter: LGNUM

Typical use:
  - Validate all tasks are completed before confirming WO
  - Trigger downstream processes (e.g., update custom status table)
  - Post confirmation event to external system
```

## Warehouse Task BAdIs (`/SCWM/ES_WHT`)

### `/SCWM/EX_WHT_CREATION` — Warehouse Task Creation

```
Methods:
  DEF_WHT_ATTR        Modify WT attributes before creation
                        IS_LTAP_VB (importing) — proposed WT data
                        CS_LTAP    (changing)  — modify to override
  BEFORE_CREATE_WHT   Last chance to change/reject WT before DB write

Filter: LGNUM, BWLVS

Typical use:
  - Override source/destination bin based on custom rules
  - Set custom movement type for special materials
  - Assign WT to specific resource
```

### `/SCWM/EX_WHT_CONF` — Warehouse Task Confirmation

```
Methods:
  BEFORE_CONF_WHT     Run before WT confirmation — can block with message
  AFTER_CONF_WHT      Run after WT confirmation
                        IS_LTAP (importing) — confirmed WT data
  
Filter: LGNUM, BWLVS

Typical use:
  - Validate actual qty vs. expected qty at confirmation
  - Trigger custom goods movement in ERP on confirmation
  - Update third-party WMS integration
```

## Picking BAdIs (`/SCWM/ES_PICK_HO`)

### `/SCWM/EX_PICK_HO` — Pick Handling Unit

```
Methods:
  GET_PICK_HU         Determine which HU to pick from
                        CT_AVAIL_HU (table) — list of available HUs
                        CS_SELECTED_HU (changing) — set selected HU
  AFTER_PICK_CONF     After pick confirmation of an HU

Filter: LGNUM

Typical use:
  - FEFO (First Expiry First Out) picking beyond standard SAP logic
  - HU selection based on external quality system data
  - Multi-criteria picking (weight, batch, location priority)
```

## RF Mobile BAdIs (`/SCWM/ES_RF`)

### `/SCWM/EX_RF_BIN_ACCESS` — RF Bin Access Control

```
Methods:
  CHECK_BIN_ACCESS    Allow/deny user access to a specific bin
                        IS_RSRC — resource (user/forklift)
                        IS_BIN  — target storage bin
                        RV_DENY — set to true to block

Filter: LGNUM

Typical use:
  - Restrict bins by shift (e.g., hazmat bins only accessible in day shift)
  - Block bins under maintenance
  - Enforce pick-to-light zone assignments
```

### `/SCWM/EX_RF_PROCESS` — RF Process Step Enhancement

```
Methods:
  BEFORE_STEP         Before processing an RF step — can redirect or add fields
  AFTER_STEP          After processing — trigger downstream actions

Filter: LGNUM

Typical use:
  - Add custom validation at each RF step
  - Log step timing for performance analysis
  - Inject additional confirmation prompts
```

## Resource Management BAdIs (`/SCWM/ES_RSRC`)

### `/SCWM/EX_RSRC_DETERM` — Resource Determination

```
Methods:
  DETERMINE_RSRC      Override default resource assignment for a WO/WT
                        IS_WHO — warehouse order data
                        CT_RSRC — list of candidate resources
                        CV_RSRC — (changing) final resource to assign

Filter: LGNUM

Typical use:
  - Assign WO to forklift based on battery level (IoT integration)
  - Balance load across resources
  - Restrict hazmat tasks to certified operators
```

## Wave Management BAdIs (`/SCWM/ES_WAVE`)

### `/SCWM/EX_WAVE_CREATION` — Wave Creation

```
Methods:
  SELECT_WAVE_ORDERS  Add/remove delivery items from wave
  SET_WAVE_ATTR       Modify wave attributes before creation

Filter: LGNUM

Typical use:
  - Include only items for specific carriers in a wave
  - Set wave priority based on ship date urgency
  - Exclude items with open quality holds
```

## Finding More EWM BAdIs

```
SE18 → Enhancement Spot → /SCWM/*
  Lists all EWM enhancement spots

SE84 → Enhancement → Business Add-Ins → Definitions
  Package: /SCWM/
  → Full list of all EWM BAdI definitions

" For /SCDL/ delivery framework BAdIs:
SE18 → Enhancement Spot → /SCDL/ES_*
```
