# TM Dangerous Goods Reference

## Overview

SAP TM integrates with the SAP Dangerous Goods Management (DGM) component to:
- Classify materials as DG
- Validate DG compatibility on freight orders
- Generate required DG documents (bill of lading, declaration)
- Apply DG-specific charges (HAZMAT surcharge)

## DG Regulations Supported

| Regulation | Mode | Authority |
|---|---|---|
| ADR | Road | UN (Europe) |
| RID | Rail | UN (Europe) |
| IMDG | Ocean | IMO |
| IATA DGR | Air | IATA |
| DOT | Road/Rail | US DOT |
| ICAO | Air | ICAO |

## Material DG Classification

Materials are classified in the EHS (Environment, Health & Safety) module:

```
Material Master → Dangerous Goods tab:
  - UN Number          (e.g. UN1090 = Acetone)
  - Hazard Class       (e.g. 3 = Flammable liquid)
  - Packing Group      (I = highest danger, II, III = lowest)
  - Proper Shipping Name
  - Emergency Action Code
  - DG Indicator per regulation (ADR, IMDG, IATA, DOT)
```

## DG Check on Freight Order

TM automatically triggers DG checks at:
- Freight unit creation (material classified as DG)
- Carrier assignment (carrier DG certification check)
- Document output (DG declaration generation)

### Programmatic DG Check

```abap
CALL FUNCTION '/SCMTMS/DG_CHECK_EXECUTE'
  EXPORTING
    iv_tor_id     = lv_fo_id
    iv_regulation = 'ADR'         " ADR / IMDG / IATA / DOT
  IMPORTING
    et_dg_items   = DATA(lt_dg_items)
    et_errors     = DATA(lt_dg_errors)
    et_return     = DATA(lt_messages).

" Check for DG violations
LOOP AT lt_dg_errors ASSIGNING FIELD-SYMBOL(<err>).
  " <err>-error_code    = DG check error code
  " <err>-matnr         = Material causing error
  " <err>-un_number     = UN number
  " <err>-description   = Human-readable error
ENDLOOP.
```

## DG Item Data

```abap
" Read DG items on a freight order
SELECT tor_id, matnr, un_number, haz_class, pack_group,
       net_weight, regulation, proper_ship_name
  FROM /scmtms/d_dg_item
  WHERE tor_id = @lv_fo_id
  INTO TABLE @DATA(lt_dg_items).
```

## Compatibility Check

TM checks whether DG materials on the same freight order can be co-loaded:

```abap
CALL FUNCTION '/SCMTMS/DG_COMPAT_CHECK'
  EXPORTING
    iv_tor_id     = lv_fo_id
    iv_regulation = 'IMDG'
  IMPORTING
    et_incompatible = DATA(lt_incompatible_pairs)
    et_return       = DATA(lt_messages).

" lt_incompatible_pairs lists pairs of UN numbers that cannot co-load
```

## BADI: Custom DG Validation

### Enhancement Spot: `/SCMTMS/ES_DG_CHECK`
### BADI Interface: `/SCMTMS/IF_EX_DG_CHECK`

```abap
CLASS zcl_tm_dg_custom DEFINITION PUBLIC.
  PUBLIC SECTION.
    INTERFACES /scmtms/if_ex_dg_check.
ENDCLASS.

CLASS zcl_tm_dg_custom IMPLEMENTATION.

  METHOD /scmtms/if_ex_dg_check~validate.
    " Add custom DG validation rules beyond standard SAP checks
    " is_tor_root: freight order header
    " it_dg_items: DG items on the shipment

    " Example: reject DG class 1 (explosives) on road mode
    IF is_tor_root-means_tra = /scmtms/if_tor_const=>gc_means_tra_road.
      LOOP AT it_dg_items ASSIGNING FIELD-SYMBOL(<dg>).
        IF <dg>-haz_class = '1'.
          APPEND VALUE /scmtms/s_dg_error(
            error_code  = 'Z_NO_EXPL_ROAD'
            matnr       = <dg>-matnr
            un_number   = <dg>-un_number
            description = 'Explosives (Class 1) not permitted by road under company policy'
            severity    = 'E' ) TO et_errors.
        ENDIF.
      ENDLOOP.
    ENDIF.
  ENDMETHOD.

ENDCLASS.
```

## DG Document Output

DG declarations are generated via PPF (Post Processing Framework):

```
PPF Action: ZDG_DECLARATION
  → Processing class: ZCL_TM_DG_DECLARATION
  → Form: Adobe Form or SmartForm
  → Output: Print / Email / EDI
```

### Trigger DG Declaration

```abap
" Trigger DG declaration PPF action
CALL FUNCTION 'CL_APF_EXEC_SIMPLE=>EXEC_IMMEDIATELY'
  EXPORTING
    iv_applic = /scmtms/if_tor_const=>gc_torcat_fo
    iv_docno  = lv_fo_id
    iv_actn   = 'ZDG_DECLARATION'
  IMPORTING
    et_return = DATA(lt_return).
```

## DG Charges

Apply HAZMAT surcharge automatically for DG shipments:

```abap
" In charge calculation BADI (/SCMTMS/IF_EX_CHARGE_CALC~ENRICH_CHARGES):
SELECT COUNT(*) FROM /scmtms/d_dg_item
  WHERE tor_id = @is_tor_root-tor_id
  INTO @DATA(lv_dg_count).

IF lv_dg_count > 0.
  APPEND VALUE /scmtms/s_charge(
    charge_type = 'HAZMAT'
    amount      = '75.00'
    currency    = is_tor_root-currency
    calc_base   = 'SHIPMENT' ) TO ct_charges.
ENDIF.
```

## Carrier DG Certification

Verify carrier is certified for the DG class before assignment:

```abap
" In carrier selection BADI (/SCMTMS/IF_EX_CARRIER_SEL~FILTER_CARRIERS):
DATA(lv_has_dg) = abap_false.
SELECT COUNT(*) FROM /scmtms/d_dg_item
  WHERE tor_id = @is_tor_root-tor_id
  INTO @DATA(lv_dg_count).
IF lv_dg_count > 0. lv_has_dg = abap_true. ENDIF.

IF lv_has_dg = abap_true.
  " Remove carriers without DG certification from proposed list
  DELETE ct_carriers WHERE bp_id NOT IN (
    SELECT bp_id FROM zcarrier_dg_cert
      WHERE certified = abap_true
        AND valid_to >= @sy-datum ).
ENDIF.
```
