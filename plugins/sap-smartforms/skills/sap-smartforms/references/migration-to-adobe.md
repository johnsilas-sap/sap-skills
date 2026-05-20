# Migration from SmartForms to Adobe Forms Reference

## Why Migrate

- SmartForms cannot produce interactive (fillable) PDFs
- Adobe Forms (ADS) produces true PDF/A — better archiving, digital signatures
- SAP's form strategy favors Adobe Forms for new development
- SmartForms rendering is ABAP-only OTF; Adobe Forms uses industry-standard XFA/PDF

## Concept Mapping

| SmartForms | Adobe Forms (SFP) |
|---|---|
| SMARTFORMS transaction | SFP transaction |
| SMARTSTYLES | Style in Adobe LiveCycle Designer |
| Form Interface (import params) | Form Interface → Import parameters |
| Global Definitions (data/types) | Form Interface → Global Data |
| Initialization code | Form Interface → Initialization |
| Subroutine FORM blocks | Helper methods called in init code |
| Pages | Body Pages + Master Pages |
| Windows | Subforms (static or flowed) |
| MAIN window | Flowed subform (expands/overflows) |
| Secondary window | Static subform (fixed position) |
| Table node (loop) | Repeating subform (flowed, repeat for each item) |
| Template node | Static positioned subform with fields |
| Text element | Text field bound to data node |
| Graphic node | Image field bound to XSTRING |
| Address node | Positioned text fields |
| Condition on element | JavaScript `this.presence` in Initialize event |
| Table event: footer | FormCalc `Sum()` or JS Calculate event |
| Alternative element | Two subforms with inverse conditions |
| Page break | Subform pagination → Start new page |
| `&FIELD&` text syntax | Field binding in Data View panel |
| `SFSY-PAGE` | `Page()` in FormCalc |
| `SFSY-FORMPAGES` | `Count(PageAll())` in FormCalc |

## Print Program Migration

### SmartForms Pattern → Adobe Forms Pattern

```abap
" SmartForms (before)
CALL FUNCTION 'SSF_FUNCTION_MODULE_NAME'
  EXPORTING  formname = 'ZEWM_DELIVERY_NOTE'
  IMPORTING  fm_name  = lv_fm_name.

CALL FUNCTION lv_fm_name
  EXPORTING
    control_parameters = ls_control
    output_options     = ls_compop
    is_header          = ls_header
    it_items           = lt_items.

" Adobe Forms (after)
CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
  EXPORTING i_name     = 'ZEWM_DELIVERY_NOTE'
  IMPORTING e_funcname = lv_fm_name.

CALL FUNCTION 'FP_JOB_OPEN'
  CHANGING ie_outputparams = ls_outputparams.

CALL FUNCTION lv_fm_name
  EXPORTING
    is_header = ls_header
    it_items  = lt_items.

CALL FUNCTION 'FP_JOB_CLOSE'.
```

### Control Options Migration

```abap
" SmartForms
ls_control = VALUE ssfctrlop( no_dialog = 'X' getotf = 'X' ).
ls_compop  = VALUE ssfcompop( tdprinter = 'LP01' tdcopies = 1 tdimmed = 'X' ).

" Adobe Forms equivalent
ls_outputparams = VALUE sfpoutputparams(
  nodialog = abap_true   " no_dialog = 'X'
  dest     = 'LP01'      " tdprinter
  copies   = 1           " tdcopies
  reqimmed = abap_true   " tdimmed
).
```

### PDF Retrieval Migration

```abap
" SmartForms OTF → PDF
ls_control-getotf = 'X'.
CALL FUNCTION lv_fm_name ... IMPORTING otf_data = lt_otf.
CALL FUNCTION 'CONVERT_OTF' EXPORTING format = 'PDF' TABLES otf = lt_otf lines = lt_lines.
CALL FUNCTION 'SCMS_TLINE_TO_XSTRING' IMPORTING buffer = lv_pdf_xstr.

" Adobe Forms PDF retrieval (simpler)
ls_outputparams-getpdf = abap_true.
CALL FUNCTION 'FP_JOB_OPEN' CHANGING ie_outputparams = ls_outputparams.
CALL FUNCTION lv_fm_name EXPORTING is_header = ls_header it_items = lt_items.
CALL FUNCTION 'FP_JOB_CLOSE' IMPORTING e_result = ls_result.
lv_pdf_xstr = ls_result-pdf.   " PDF directly as XSTRING
```

## Interface Migration

SmartForms and Adobe Forms interfaces are nearly identical in structure — both define import parameters and global data. The same ABAP types and parameters can be reused.

```abap
" SmartForms Interface → Import:
IS_HEADER   TYPE ZEWM_DELIV_HDR_S
IT_ITEMS    TYPE ZEWM_DELIV_ITEM_TAB

" Adobe SFP Interface → Import:
IS_HEADER   TYPE ZEWM_DELIV_HDR_S   " Same types — no change needed
IT_ITEMS    TYPE ZEWM_DELIV_ITEM_TAB
```

## Layout Migration Steps

1. **Create new Adobe form** in SFP with the same interface parameters as the SmartForms form
2. **Open Adobe LiveCycle Designer** — recreate pages and windows as body pages and subforms
3. **Recreate MAIN window** as a flowed subform that overflows across body pages
4. **Recreate secondary windows** (header, footer) in master pages
5. **Recreate table** as repeating subform bound to `IT_ITEMS[*]`
6. **Migrate text elements** — bind fields using Data View panel instead of `&FIELD&` syntax
7. **Migrate conditions** — use JavaScript Initialize event `this.presence = "hidden/visible"`
8. **Migrate table totals** — use FormCalc `Sum(IT_ITEMS[*].QUANTITY)` in Calculate event

## Migration Checklist

```
□ Form interface parameters copied (types are reusable)
□ Global data definitions copied to SFP Global Data
□ Initialization code copied/adapted (same ABAP syntax)
□ Subroutines refactored into init code or helper FMs
□ Pages recreated as body pages + master pages
□ MAIN window recreated as flowed subform
□ Secondary windows recreated in master page
□ Table recreated as repeating subform with correct binding
□ Conditions migrated to JS Initialize events
□ Page numbering migrated to FormCalc
□ Graphics re-uploaded (SE78 → import into layout as image field)
□ Barcodes recreated as barcode fields with correct type
□ Print program updated (SSF → FP function modules)
□ Output parameter struct changed (SSFCTRLOP → SFPOUTPUTPARAMS)
□ PDF retrieval updated (CONVERT_OTF chain → FP_JOB_CLOSE result)
□ Tested in ADS connection (SFPADM)
□ Tested in spool (SP01)
```

## Running Both Forms in Parallel (Cutover Strategy)

```abap
" Feature flag approach — print from SmartForms or Adobe based on config
DATA(lv_use_adobe) = get_config( 'USE_ADOBE_FORMS' ).

IF lv_use_adobe = 'X'.
  " Adobe Forms path
  CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
    EXPORTING i_name = 'ZEWM_DELIVERY_NOTE_ADS' IMPORTING e_funcname = lv_fm.
  CALL FUNCTION 'FP_JOB_OPEN' CHANGING ie_outputparams = ls_fp_out.
  CALL FUNCTION lv_fm EXPORTING is_header = ls_header it_items = lt_items.
  CALL FUNCTION 'FP_JOB_CLOSE'.
ELSE.
  " SmartForms path (legacy)
  CALL FUNCTION 'SSF_FUNCTION_MODULE_NAME'
    EXPORTING formname = 'ZEWM_DELIVERY_NOTE' IMPORTING fm_name = lv_fm.
  CALL FUNCTION lv_fm
    EXPORTING control_parameters = ls_ssf_ctrl output_options = ls_compop
              is_header = ls_header it_items = lt_items.
ENDIF.
```
