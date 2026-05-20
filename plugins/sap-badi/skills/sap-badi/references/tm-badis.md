# TM BAdI Catalog

## Enhancement Spot Overview

| Enhancement Spot | Area |
|---|---|
| `/SCMTMS/ES_FRO` | Freight order processing |
| `/SCMTMS/ES_CARR_SEL` | Carrier selection and tendering |
| `/SCMTMS/ES_CHRG` | Charge calculation |
| `/SCMTMS/ES_STLMT` | Settlement |
| `/SCMTMS/ES_PLNG` | Transportation planning |
| `/SCMTMS/ES_DG` | Dangerous goods |
| `/SAPTRX/ES_TT` | Track and trace |

## Freight Order BAdIs (`/SCMTMS/ES_FRO`)

### `/SCMTMS/EX_FRO_BOPF_CHK` — Freight Order Validation

```
Methods:
  CHECK_BEFORE_SAVE   Validate FO before DB write
                        IO_FRO      — freight order business object
                        IT_MODIFIED — changed fields
                        ET_MESSAGES — add error/warning messages here

Filter: FROTYP (freight order type)

Typical use:
  - Validate that all required dangerous goods data is complete
  - Check that carrier is approved for this lane
  - Enforce maximum weight/volume limits per FO type
```

### `/SCMTMS/EX_FRO_STATUS` — Freight Order Status Change

```
Methods:
  AFTER_STATUS_CHANGE  Called after FO status changes (planned → in execution → completed)
                         IV_OLD_STATUS — previous status
                         IV_NEW_STATUS — new status
                         IO_FRO        — freight order

Filter: FROTYP

Typical use:
  - Trigger track & trace event on FO departure
  - Notify external TMS on FO status change
  - Create EWM delivery when FO reaches "In Execution"
```

### `/SCMTMS/EX_FRO_ENRICH` — Freight Order Data Enrichment

```
Methods:
  ENRICH_FRO_DATA     Add custom fields to FO after creation/change
                        IO_FRO — freight order (modifiable via BOPF)

Filter: FROTYP

Typical use:
  - Populate custom fields from customer master
  - Set default carrier contract based on lane
  - Compute custom charges not in standard calculation
```

## Carrier Selection BAdIs (`/SCMTMS/ES_CARR_SEL`)

### `/SCMTMS/EX_CARR_SEL_FILTER` — Carrier Filtering

```
Methods:
  FILTER_CARRIERS     Remove ineligible carriers from selection list
                        CT_CARRIERS — (changing table) remove rows to exclude carriers
                        IS_FRO_DATA — freight order data for context

Filter: FROTYP, TRNMOD

Typical use:
  - Exclude carriers with expired insurance
  - Filter by hazmat certification for dangerous goods shipments
  - Apply lane-specific carrier blacklist from external system
```

### `/SCMTMS/EX_CARR_SEL_SCORE` — Carrier Scoring

```
Methods:
  SCORE_CARRIERS      Adjust carrier scores before ranking
                        CT_SCORED_CARRIERS — (changing) modify SCORE field
                        IS_FRO_DATA        — freight order context

Filter: FROTYP

Typical use:
  - Boost preferred carrier score based on past performance
  - Apply business rules (e.g., favor minority carriers on eligible lanes)
  - Penalize carriers with recent service failures
```

### `/SCMTMS/EX_TENDERING` — Carrier Tendering

```
Methods:
  BEFORE_TENDER       Before tendering request is sent to carrier
  AFTER_TENDER_REPLY  After carrier responds (accept/reject/counter-offer)
                        IS_TENDER_REPLY — carrier response data
                        EV_ACCEPT       — set true to auto-accept

Filter: FROTYP

Typical use:
  - Auto-accept if rate is within budget threshold
  - Send tender via EDI/API to carrier portal
  - Log all tender communications for compliance
```

## Charge Calculation BAdIs (`/SCMTMS/ES_CHRG`)

### `/SCMTMS/EX_CHRG_CALC` — Custom Charge Calculation

```
Methods:
  CALCULATE_CHARGES   Add or modify charges on the FO
                        IO_CHRG_DOC — charge document (modifiable)
                        IS_FRO_DATA — freight order data

Filter: FROTYP

Typical use:
  - Add fuel surcharge based on today's fuel index
  - Apply spot rate from external rate engine API
  - Calculate customs duties for cross-border shipments
```

### `/SCMTMS/EX_CHRG_VALIDATE` — Charge Validation

```
Methods:
  VALIDATE_CHARGES    Validate charges before posting
                        IT_CHARGES  — calculated charges
                        ET_MESSAGES — add warnings/errors

Typical use:
  - Warn if calculated freight > budget threshold
  - Block charges that exceed rate agreement limits
```

## Dangerous Goods BAdIs (`/SCMTMS/ES_DG`)

### `/SCMTMS/EX_DG_CHECK` — DG Compliance Check

```
Methods:
  CHECK_DG_COMPLIANCE Validate dangerous goods data on freight item
                        IS_ITEM     — freight order item
                        IS_DG_DATA  — DG classification data
                        ET_MESSAGES — add errors for compliance failures

Typical use:
  - Validate UN number against approved list for carrier
  - Check transport mode restrictions (e.g., Class 7 not air-eligible)
  - Verify emergency contact data is complete
```

## Track & Trace BAdIs (`/SAPTRX/ES_TT`)

### `/SAPTRX/EX_EM_APP_OBJ` — Event Management Application Object

```
Methods:
  GET_APP_OBJ_DATA    Provide TM document data to Event Management
  PROCESS_EVENTS      React to incoming track events from carrier

Filter: TRKOROBJ (tracked object type)

Typical use:
  - Map carrier tracking events to TM status updates
  - Forward events to customer portal
  - Trigger EWM inbound delivery on "Arrived at destination" event
```

## Settlement BAdIs (`/SCMTMS/ES_STLMT`)

### `/SCMTMS/EX_STLMT_ENRICH` — Settlement Enrichment

```
Methods:
  ENRICH_SETTLEMENT   Add custom data to settlement document
                        IO_STLMT — settlement business object

Typical use:
  - Add internal cost center allocation
  - Populate custom fields required by financial posting
  - Apply settlement exchange rate from external source
```

## Finding More TM BAdIs

```
SE18 → Enhancement Spot → /SCMTMS/*
  Lists all TM enhancement spots

SE84 → Enhancement → Business Add-Ins → Definitions
  Package: /SCMTMS/
  → Full list of all TM BAdI definitions

" For /SAPTRX/ track & trace:
SE18 → Enhancement Spot → /SAPTRX/*
```

## Implementation Pattern for TM BAdIs

```abap
CLASS zcl_im_tm_carr_filter DEFINITION
  PUBLIC FINAL CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES /scmtms/if_ex_carr_sel_filter.
ENDCLASS.

CLASS zcl_im_tm_carr_filter IMPLEMENTATION.

  METHOD /scmtms/if_ex_carr_sel_filter~filter_carriers.
    " Remove carriers without hazmat certification
    DATA(lv_is_dg) = is_fro_data-has_dangerous_goods.

    IF lv_is_dg = abap_true.
      DELETE ct_carriers
        WHERE hazmat_certified = abap_false.
    ENDIF.
  ENDMETHOD.

ENDCLASS.
```
