# EWM BADIs and Enhancement Spots Reference

## Finding BADIs

1. **SE18** — Browse enhancement spots: search `%/SCWM%` for all EWM spots
2. **ST05** — SQL trace during a process to find which BADIs fire
3. **Runtime Check**: Set breakpoint in `CL_BADI_MANAGER=>GET_IMPL` and run the RF transaction

## Core BADIs Catalog

### RF Business Logic

| BADI Interface | Enhancement Spot | Trigger |
|---|---|---|
| `/SCWM/IF_EX_RF_BLL_PROCESSOR` | `/SCWM/ES_RF_BLL_PROCESSOR` | Every RF screen transition |
| `/SCWM/IF_EX_RF_UI_DEFINITION` | `/SCWM/ES_RF_UI_DEFINITION` | RF screen layout/field definition |

#### /SCWM/IF_EX_RF_BLL_PROCESSOR~PROCESS_REQUEST

Called on every screen submit. Modify `cs_resp` to influence the next screen.

```abap
METHOD /scwm/if_ex_rf_bll_processor~process_request.
  " Validation example: reject wrong product on pick screen
  IF is_req-linfld-tcode = '/SCWM/RF03'
  AND is_req-linfld-matnr <> lv_expected_matnr.
    cs_resp-msgty = 'E'.
    cs_resp-msgid = 'ZEWM'.
    cs_resp-msgno = '001'.
    cs_resp-msgv1 = 'Wrong product scanned'.
  ENDIF.
ENDMETHOD.
```

---

### Warehouse Task / Order Creation

| BADI Interface | Enhancement Spot | Trigger |
|---|---|---|
| `/SCWM/IF_EX_TO_CREATE` | `/SCWM/ES_TO_CREATE` | Before/after WT creation |
| `/SCWM/IF_EX_WHO_CREATE` | `/SCWM/ES_WHO_CREATE` | Warehouse order creation |
| `/SCWM/IF_EX_WHO_PACKING` | `/SCWM/ES_WHO_PACKING` | Packing step during WO |
| `/SCWM/IF_EX_TO_CONFIRM` | `/SCWM/ES_TO_CONFIRM` | Before/after WT confirmation |
| `/SCWM/IF_EX_TO_CANCEL` | `/SCWM/ES_TO_CANCEL` | Before/after WT cancellation |

#### /SCWM/IF_EX_TO_CREATE~CHANGE_WT_FOR_CREATION

```abap
METHOD /scwm/if_ex_to_create~change_wt_for_creation.
  " Modify warehouse task data before it is saved
  " ct_ordim_o: table of WTs being created — modify in place
  LOOP AT ct_ordim_o ASSIGNING FIELD-SYMBOL(<wt>).
    IF <wt>-lgtyp = '0001' AND <wt>-vsolm > 100.
      " Split large quantities to separate bins
      " (custom logic)
    ENDIF.
  ENDLOOP.
ENDMETHOD.
```

---

### Bin / Storage Determination

| BADI Interface | Enhancement Spot | Trigger |
|---|---|---|
| `/SCWM/IF_EX_CORE_DETERMINATION` | `/SCWM/ES_CORE_DETERMINATION` | Putaway / pick bin determination |
| `/SCWM/IF_EX_STRAT_PUTAWAY` | `/SCWM/ES_STRAT_PUTAWAY` | Override putaway strategy |
| `/SCWM/IF_EX_STRAT_REMOVAL` | `/SCWM/ES_STRAT_REMOVAL` | Override removal strategy |

#### /SCWM/IF_EX_CORE_DETERMINATION~DETERMINE_PUTAWAY_BIN

```abap
METHOD /scwm/if_ex_core_determination~determine_putaway_bin.
  " Force a specific bin for a product family
  IF is_ltap_vb-matnr(3) = 'HAZ'.  " Hazardous material prefix
    cs_ltap_vb-lgpla = 'HAZ-ZONE-001'.
    cs_ltap_vb-lgtyp = 'HAZ'.
  ENDIF.
ENDMETHOD.
```

---

### Quant Changes

| BADI Interface | Enhancement Spot | Trigger |
|---|---|---|
| `/SCWM/IF_EX_QUAN_CHANGE` | `/SCWM/ES_QUAN_CHANGE` | Any stock change (GR, GI, transfer, PI) |

#### /SCWM/IF_EX_QUAN_CHANGE~QUAN_CHANGE_POST

```abap
METHOD /scwm/if_ex_quan_change~quan_change_post.
  " Fires after every quant change is committed
  " it_quans_new: new quant state; it_quans_old: prior state
  LOOP AT it_quans_new ASSIGNING FIELD-SYMBOL(<new>).
    " Log stock changes to custom audit table
    INSERT zewm_stock_log FROM VALUE #(
      lgnum    = <new>-lgnum
      lgpla    = <new>-lgpla
      matnr    = <new>-matnr
      quan_new = <new>-quan
      udate    = sy-datum
      utime    = sy-uzeit
      uname    = sy-uname ).
  ENDLOOP.
ENDMETHOD.
```

---

### Delivery Processing

| BADI Interface | Enhancement Spot | Trigger |
|---|---|---|
| `/SCWM/IF_EX_DELIVERY_PPF` | `/SCWM/ES_DELIVERY_PPF` | Post-processing of inbound/outbound |
| `/SCWM/IF_EX_DLV_MANAGEMENT` | `/SCWM/ES_DLV_MANAGEMENT` | Delivery creation/change |
| `/SCWM/IF_EX_GR_GI` | `/SCWM/ES_GR_GI` | Before/after goods receipt or goods issue |

#### /SCWM/IF_EX_GR_GI~BEFORE_GR

```abap
METHOD /scwm/if_ex_gr_gi~before_gr.
  " Validate delivery before GR posting
  " Check e.g. vendor certification, temperature logs
  IF ls_header-lifnr = 'UNCERTIFIED'.
    RAISE EXCEPTION TYPE /scwm/cx_core
      EXPORTING textid = /scwm/cx_core=>vendor_not_certified.
  ENDIF.
ENDMETHOD.
```

---

### Physical Inventory

| BADI Interface | Enhancement Spot | Trigger |
|---|---|---|
| `/SCWM/IF_EX_PHYINV` | `/SCWM/ES_PHYINV` | PI document creation, count, posting |
| `/SCWM/IF_EX_PHYINV_DIFF` | `/SCWM/ES_PHYINV_DIFF` | PI difference handling |

---

### Wave Management

| BADI Interface | Enhancement Spot | Trigger |
|---|---|---|
| `/SCWM/IF_EX_WAVE` | `/SCWM/ES_WAVE` | Wave creation and release |
| `/SCWM/IF_EX_WAVE_ASSIGN` | `/SCWM/ES_WAVE_ASSIGN` | Delivery-to-wave assignment |

---

## Creating a BADI Implementation

### Step-by-Step in SE18/SE19

1. `SE18` → Enter enhancement spot (e.g. `/SCWM/ES_RF_BLL_PROCESSOR`) → Display
2. Note the BADI definition name and interface
3. `SE19` → Create implementation → Enter a Z-name
4. Select the BADI definition
5. Create implementing class (SE24 or inline)
6. Implement all required methods
7. Activate class, then activate BADI implementation

### Filter-Dependent BADIs

Some EWM BADIs are filter-dependent — they only fire for a specific warehouse number:

```abap
" When creating the BADI implementation in SE19:
" Under "Filter Values" tab, enter:
"   LGNUM = 1000   (your warehouse number)
" This prevents the BADI from firing in other warehouses
```

### BADI Implementation Class Template

```abap
CLASS zcl_ewm_my_badi DEFINITION
  PUBLIC FINAL CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES /scwm/if_ex_rf_bll_processor.  " Replace with target BADI interface
ENDCLASS.

CLASS zcl_ewm_my_badi IMPLEMENTATION.

  METHOD /scwm/if_ex_rf_bll_processor~process_request.
    " Implementation here
  ENDMETHOD.

ENDCLASS.
```

## Common Pitfalls

| Problem | Solution |
|---|---|
| BADI fires but changes are ignored | Check if the method has `CHANGING` vs `EXPORTING` params — use the right one |
| BADI fires in wrong warehouse | Add LGNUM filter in SE19 implementation |
| Multiple implementations conflict | Check priority/sequence in SE19; only one active impl per filter set by default |
| Changes not persisted | Some BADIs run before COMMIT — ensure no ROLLBACK WORK between BADI and commit |
| BADI not found in SE18 | Try searching enhancement spot with `%EX%` pattern for the process area |
