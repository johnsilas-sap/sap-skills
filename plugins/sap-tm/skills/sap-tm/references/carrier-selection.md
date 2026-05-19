# TM Carrier Selection Reference

## Overview

Carrier selection in TM evaluates available carriers based on:
1. **Freight agreements** — contracted rates and service levels
2. **Carrier profiles** — capability, mode, lane coverage
3. **Ranking criteria** — cost, lead time, performance score
4. **Custom BADI logic** — business rules not covered by standard

## Carrier Master Data

### Business Partner Setup

Carriers are SAP Business Partners (BP) with transportation-specific roles:

```
BP transaction (BP) or /SCMTMS/CARRIER:
  - BP role: Carrier (assigned in BP master)
  - Transportation lane: defines which lanes carrier serves
  - Means of transport: road, sea, air, rail
  - Service level: standard, express, economy
```

### Read Carrier Master

```abap
SELECT bp_id, carrier_name, means_tra, service_level
  FROM /scmtms/d_carrier
  WHERE means_tra = @lv_mode
  INTO TABLE @DATA(lt_carriers).
```

## Freight Agreements

Freight agreements define contracted rates between shipper and carrier:

```
Freight Agreement (FA)
  └── Rate Table
        └── Rate Row (lane + validity + rate)
              └── Charge Components (freight, fuel, accessorial)
```

### Read Freight Agreement

```abap
SELECT fa_id, carrier_id, valid_from, valid_to, currency
  FROM /scmtms/d_fa_h
  WHERE carrier_id = @lv_carrier
    AND valid_from <= @sy-datum
    AND valid_to   >= @sy-datum
  INTO TABLE @DATA(lt_agreements).
```

## Executing Carrier Selection

### Standard API

```abap
CALL FUNCTION '/SCMTMS/CARRIER_SEL_EXECUTE'
  EXPORTING
    iv_tor_id        = lv_fo_id
    iv_sel_profile   = lv_carrier_sel_profile  " Configured in Customizing
    iv_mode          = 'AUTO'                  " AUTO / MANUAL / PROPOSE
  IMPORTING
    et_carrier       = DATA(lt_proposed)
    et_return        = DATA(lt_messages).

" Proposed carrier list
LOOP AT lt_proposed ASSIGNING FIELD-SYMBOL(<c>).
  " <c>-bp_id       = Carrier BP ID
  " <c>-rank        = Selection rank (1 = best)
  " <c>-total_cost  = Total estimated cost
  " <c>-currency    = Cost currency
  " <c>-transit_days = Transit time in days
  " <c>-fa_id       = Applicable freight agreement
ENDLOOP.
```

### Assign Selected Carrier

```abap
" Assign the top-ranked carrier (or user-selected)
CALL FUNCTION '/SCMTMS/FRO_CARRIER_ASSIGN'
  EXPORTING
    iv_tor_id    = lv_fo_id
    iv_bp_id     = lt_proposed[ 1 ]-bp_id
    iv_fa_id     = lt_proposed[ 1 ]-fa_id
    iv_commit    = abap_true
  IMPORTING
    et_return    = DATA(lt_messages).
```

## Tendering

Tendering sends the freight order to one or more carriers for acceptance/rejection:

```abap
" Create tender
CALL FUNCTION '/SCMTMS/TENDER_CREATE'
  EXPORTING
    iv_tor_id         = lv_fo_id
    it_carrier_ids    = lt_carrier_ids   " Carriers to tender to
    iv_deadline       = lv_response_by
    iv_commit         = abap_true
  IMPORTING
    ev_tender_id      = DATA(lv_tender_id)
    et_return         = DATA(lt_messages).

" Read tender responses
SELECT tender_id, carrier_id, response, offered_price, response_date
  FROM /scmtms/d_tender_r
  WHERE tender_id = @lv_tender_id
  INTO TABLE @DATA(lt_responses).
```

## BADI: Carrier Ranking Override

### Enhancement Spot: `/SCMTMS/ES_CARRIER_SEL`
### BADI Interface: `/SCMTMS/IF_EX_CARRIER_SEL`

```abap
CLASS zcl_tm_carrier_rank DEFINITION PUBLIC.
  PUBLIC SECTION.
    INTERFACES /scmtms/if_ex_carrier_sel.
ENDCLASS.

CLASS zcl_tm_carrier_rank IMPLEMENTATION.

  METHOD /scmtms/if_ex_carrier_sel~rank_carriers.
    " is_tor_root: freight order header data
    " ct_carriers: proposed carrier list — modify rank and cost

    " Example: boost preferred carrier for a specific lane
    SELECT SINGLE bp_id FROM zpreferred_carrier
      WHERE src_loc = @is_tor_root-src_loc_id
        AND dst_loc = @is_tor_root-dst_loc_id
      INTO @DATA(lv_preferred).

    LOOP AT ct_carriers ASSIGNING FIELD-SYMBOL(<c>).
      IF <c>-bp_id = lv_preferred.
        <c>-rank = 1.
      ELSEIF <c>-rank = 1.
        <c>-rank = 2.  " Demote whoever was rank 1
      ENDIF.
    ENDLOOP.
    SORT ct_carriers BY rank.
  ENDMETHOD.

  METHOD /scmtms/if_ex_carrier_sel~filter_carriers.
    " Remove carriers that fail custom business rules
    DELETE ct_carriers WHERE bp_id IN lt_blacklisted_carriers.
  ENDMETHOD.

ENDCLASS.
```

## BADI: Freight Order Change (Carrier-Related)

```abap
METHOD /scmtms/if_ex_fro_change~after_save.
  " React when a carrier is assigned/changed
  LOOP AT it_changed_fros ASSIGNING FIELD-SYMBOL(<fo>).
    IF <fo>-changed_fields-carrier_id = abap_true.
      " Carrier changed — trigger notification or downstream update
      CALL FUNCTION 'Z_NOTIFY_CARRIER_ASSIGNED'
        EXPORTING
          iv_fo_id      = <fo>-tor_id
          iv_carrier_id = <fo>-carrier_id.
    ENDIF.
  ENDLOOP.
ENDMETHOD.
```

## Carrier Performance Tracking

Track carrier KPIs for future selection decisions:

```abap
" Log on-time delivery performance
INSERT INTO zcarrier_perf FROM VALUE #(
  bp_id       = lv_carrier
  fo_id       = lv_fo_id
  planned_arr = ls_fo-arr_date
  actual_arr  = sy-datum
  on_time     = COND abap_bool(
                  WHEN sy-datum <= ls_fo-arr_date
                  THEN abap_true ELSE abap_false )
  udate       = sy-datum
  utime       = sy-uzeit ).
COMMIT WORK.

" Query on-time rate per carrier
SELECT bp_id,
       COUNT(*) AS total,
       SUM( CASE WHEN on_time = 'X' THEN 1 ELSE 0 END ) AS on_time_count
  FROM zcarrier_perf
  WHERE udate >= @lv_from
  GROUP BY bp_id
  INTO TABLE @DATA(lt_performance).
```
