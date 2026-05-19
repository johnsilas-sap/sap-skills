# TM Charge Calculation Reference

## Charge Model

```
Freight Order
  └── Charge Calculation Run
        └── Charge Items
              ├── Base Freight Charge    (from freight agreement rate)
              ├── Fuel Surcharge         (% of base, from fuel index)
              ├── Accessorial Charges    (detention, liftgate, residential)
              └── Taxes                 (VAT, customs duties)
```

## Charge Calculation Profiles

Configured in Customizing → TM → Charge Management → Define Calculation Profiles.
Each profile specifies which charge types to calculate and in which order.

## Triggering Calculation

### Manual / Programmatic Trigger

```abap
CALL FUNCTION '/SCMTMS/CHARGE_CALC_EXECUTE'
  EXPORTING
    iv_tor_id       = lv_fo_id
    iv_calc_profile = lv_profile      " Customizing profile name
    iv_recalculate  = abap_true       " Force recalc even if charges exist
  IMPORTING
    et_charges      = DATA(lt_charges)
    et_return       = DATA(lt_messages).
```

### Automatic Calculation Trigger Points

Charge calculation is triggered automatically at:
- Carrier assignment
- Freight order completion
- Settlement initiation
- Manual user action in `/SCMTMS/MON_FO`

## Reading Charges

```abap
SELECT tor_id, charge_type, calc_base, rate, amount, currency, fa_id
  FROM /scmtms/d_charge
  WHERE tor_id = @lv_fo_id
  INTO TABLE @DATA(lt_charges).

" Summarize by charge type
SELECT charge_type, SUM( amount ) AS total, currency
  FROM /scmtms/d_charge
  WHERE tor_id = @lv_fo_id
  GROUP BY charge_type, currency
  INTO TABLE @DATA(lt_summary).
```

### Key Charge Fields

| Field | Description |
|---|---|
| `CHARGE_TYPE` | Category (FREIGHT, FUEL, DETENT, TAX, etc.) |
| `CALC_BASE` | Basis for calculation (WEIGHT, VOLUME, DISTANCE, SHIPMENT) |
| `RATE` | Rate per unit of calculation base |
| `AMOUNT` | Calculated charge amount |
| `CURRENCY` | Currency key |
| `FA_ID` | Freight agreement used |
| `RATE_ROW_ID` | Specific rate row from agreement |

## Freight Agreement Rate Structure

```
Freight Agreement
  └── Rate Table (by lane, mode, validity)
        └── Rate Row
              ├── Base Rate (per kg, per km, per shipment)
              ├── Min Charge
              ├── Max Charge
              └── Surcharge Rules
```

### Read Rate from Agreement

```abap
SELECT *
  FROM /scmtms/d_fa_rate
  WHERE fa_id      = @lv_fa_id
    AND src_loc_id = @lv_source
    AND dst_loc_id = @lv_destination
    AND valid_from <= @sy-datum
    AND valid_to   >= @sy-datum
  INTO TABLE @DATA(lt_rates).
```

## BADI: Custom Charge Enrichment

### Enhancement Spot: `/SCMTMS/ES_CHARGE_CALC`
### BADI Interface: `/SCMTMS/IF_EX_CHARGE_CALC`

```abap
CLASS zcl_tm_custom_charges DEFINITION PUBLIC.
  PUBLIC SECTION.
    INTERFACES /scmtms/if_ex_charge_calc.
ENDCLASS.

CLASS zcl_tm_custom_charges IMPLEMENTATION.

  METHOD /scmtms/if_ex_charge_calc~enrich_charges.
    " ct_charges: existing charge items — add/modify/delete
    " is_tor_root: freight order header for context

    " Example 1: Add fuel surcharge as % of base freight
    DATA(lv_base) = REDUCE decfloat34(
      INIT s = CONV decfloat34( 0 )
      FOR <c> IN ct_charges
        WHERE ( charge_type = 'FREIGHT' )
      NEXT s = s + <c>-amount ).

    IF lv_base > 0.
      APPEND VALUE /scmtms/s_charge(
        charge_type = 'FUEL_SUR'
        amount      = lv_base * '0.12'   " 12% fuel surcharge
        currency    = is_tor_root-currency
        calc_base   = 'FREIGHT' ) TO ct_charges.
    ENDIF.

    " Example 2: Apply residential delivery surcharge
    IF is_tor_root-dst_zone_type = 'RESIDENTIAL'.
      APPEND VALUE /scmtms/s_charge(
        charge_type = 'RESIDENTIAL'
        amount      = '45.00'
        currency    = is_tor_root-currency
        calc_base   = 'SHIPMENT' ) TO ct_charges.
    ENDIF.
  ENDMETHOD.

  METHOD /scmtms/if_ex_charge_calc~before_calc.
    " Called before standard calculation — set up context or defaults
  ENDMETHOD.

  METHOD /scmtms/if_ex_charge_calc~after_calc.
    " Called after standard calculation — post-processing, rounding
    LOOP AT ct_charges ASSIGNING FIELD-SYMBOL(<c>).
      " Round all amounts to 2 decimals
      <c>-amount = CONV decfloat34( round( val = <c>-amount dec = 2 ) ).
    ENDLOOP.
  ENDMETHOD.

ENDCLASS.
```

## Charge Correction and Adjustment

```abap
" Manual charge adjustment (e.g. credit/dispute resolution)
CALL FUNCTION '/SCMTMS/CHARGE_ADJUST'
  EXPORTING
    iv_tor_id      = lv_fo_id
    iv_charge_type = 'FREIGHT'
    iv_adjustment  = '-50.00'      " Negative = credit
    iv_currency    = 'USD'
    iv_reason      = 'DISPUTE'     " Reason code
    iv_commit      = abap_true
  IMPORTING
    et_return      = DATA(lt_messages).
```

## Settlement Flow

Once charges are finalized, settlement posts to FI/CO:

```abap
" Create settlement document from freight order
CALL FUNCTION '/SCMTMS/SETTLEMENT_CREATE'
  EXPORTING
    iv_tor_id    = lv_fo_id
    iv_commit    = abap_true
  IMPORTING
    ev_settl_id  = DATA(lv_settlement_id)
    et_return    = DATA(lt_messages).

" Read accrual entries posted to CO
SELECT *
  FROM /scmtms/d_settl
  WHERE tor_id = @lv_fo_id
  INTO TABLE @DATA(lt_settlements).
```

## Charge Types — Common Standard Values

| Charge Type | Description |
|---|---|
| `FREIGHT` | Base freight rate |
| `FUEL_SUR` | Fuel surcharge |
| `DETENT` | Detention/demurrage |
| `LIFT` | Liftgate fee |
| `INSIDE_DEL` | Inside delivery |
| `RESIDENTIAL` | Residential surcharge |
| `HAZMAT` | Dangerous goods fee |
| `OVERSIZE` | Oversize/overweight |
| `TAX` | Tax (VAT, customs) |
