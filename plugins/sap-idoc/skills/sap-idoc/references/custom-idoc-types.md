# Custom IDoc Types Reference

## Full Creation Workflow

```
WE31  → Define segment types (fields + structure)
  ↓
WE30  → Define basic IDoc type (segment hierarchy)
  ↓
WE82  → Assign basic type to message type
  ↓
WE57  → Assign inbound FM to message type
  ↓
WE20  → Add partner profile for message type
  ↓
BD64  → Add to distribution model (if ALE)
```

## Step 1: Create Segment Type (WE31)

```
WE31 → Create
  Segment type: E1ZMYHDR       (convention: E1Z... for custom)
  Description:  My IDoc Header

  Fields:
    Field name    Type   Length  Description
    REF_DOCNO     CHAR   10      Reference document number
    DOC_DATE      DATS   8       Document date
    CURRENCY      CUKY   5       Currency
    AMOUNT        CURR   13      Amount (paired with CURRENCY)
    VENDOR_ID     CHAR   10      Vendor number
    ZZCUSTOM      CHAR   20      Custom field
```

Rules for segment fields:
- Field names must be uppercase
- No internal SAP types (TYPE REF TO, etc.) — only flat elementary types
- Currency/quantity fields need reference fields in the same segment
- Max segment size: 1000 bytes (SDATA length)

## Step 2: Create Basic IDoc Type (WE30)

```
WE30 → Create
  Basic type:   ZMYBASIC01
  Description:  My Custom IDoc Type

  Segment hierarchy:
    E1ZMYHDR    (Header)    Mandatory, 1 occurrence
      E1ZMYADR  (Address)   Optional,  1 occurrence
      E1ZMYITM  (Item)      Optional,  1-999 occurrences
        E1ZMYTXT (Item text) Optional, 1-10 occurrences
```

Occurrence settings:
- **Mandatory** (Min=1, Max=1): must appear exactly once
- **Optional** (Min=0, Max=1): 0 or 1 time
- **Repeating** (Min=0, Max=999999): multiple occurrences

## Step 3: Assign to Message Type (WE82)

```
WE82 → New entry
  Message type:  ZMYTYPE
  Basic type:    ZMYBASIC01
  Release:       (current SAP release, e.g. S4CORE 107)
  Direction:     Both (or Inbound/Outbound only)
```

If ZMYTYPE doesn't exist yet, create it first:
```
WE81 → Create message type
  Message type: ZMYTYPE
  Description:  My Custom IDoc Message
```

## Step 4: Assign Inbound FM (WE57)

```
WE57 → New entry
  Function module: Z_IDOC_INPUT_ZMYTYPE
  Basic type:      ZMYBASIC01
  Message type:    ZMYTYPE
  Direction:       2 (Inbound)
```

Then create process code in WE42:
```
WE42 → Create process code
  Process code:    ZMYTYPE_IN
  Description:     My Custom IDoc Inbound
  Function module: Z_IDOC_INPUT_ZMYTYPE
```

## Extending Standard IDoc Types (Z-Extension)

Use when you need extra fields on a standard IDoc (e.g. ORDERS05) without replacing it.

### Create Extension Segment

```
WE31 → Create segment
  E1ZORDEXT    Custom fields for ORDERS extension
  Fields: ZZ_PRIORITY, ZZ_SPECIAL_INSTR, ZZ_CUSTOMER_REF
```

### Create Extension Type (WE30)

```
WE30 → Choose "Extension" tab
  Basic type to extend: ORDERS05
  Extension name:       ZORDERS05

  Add E1ZORDEXT under E1EDK01 (header level)
  OR under E1EDP01 (item level)
```

### Read Extension Segment in Inbound FM

```abap
" Standard ORDERS inbound calls your BADI or you process in a custom FM
READ TABLE idoc_data ASSIGNING FIELD-SYMBOL(<ext>)
  WITH KEY segnam = 'E1ZORDEXT' docnum = <ctrl>-docnum.
IF sy-subrc = 0.
  DATA ls_ext TYPE e1zordext.
  MOVE <ext>-sdata TO ls_ext.
  " Use ls_ext-zz_priority, ls_ext-zz_special_instr, etc.
ENDIF.
```

### Update Partner Profile to Use Extension

```
WE20 → Find partner → Outbound parameters (or Inbound)
  Message type:  ORDERS
  Basic type:    ZORDERS05    ← Change from ORDERS05
```

## ABAP Structures for Custom Segments

After creating in WE31, generate ABAP structures:

```
WE31 → Display segment → Activate
  → Automatically creates matching ABAP structure (same name as segment)
  → E1ZMYHDR structure is available in ABAP Dictionary
```

Verify in SE11:
```
SE11 → Structure → E1ZMYHDR → Display
```

## Naming Conventions

| Object | Convention | Example |
|---|---|---|
| Segment type | `E1Z` + name | `E1ZMYHDR`, `E1ZMYITM` |
| Basic type | `Z` + name + version | `ZMYBASIC01` |
| Message type | `Z` + meaning | `ZMYTYPE`, `ZEWMCONF` |
| Inbound FM | `Z_IDOC_INPUT_` + type | `Z_IDOC_INPUT_ZMYTYPE` |
| Outbound FM | `Z_IDOC_OUTPUT_` + type | `Z_IDOC_OUTPUT_ZMYTYPE` |
| Process code | type + `_IN` / `_OUT` | `ZMYTYPE_IN` |

## Transport and Change Management

IDoc type objects are transportable:
- WE30/WE31 objects → transport via SE01/STMS
- WE82/WE57/WE42 entries → table entries in transport
- WE20 partner profiles → **NOT** transported automatically (environment-specific)

```
Partner profiles (WE20) must be maintained in each system manually
or via a report/LSMW after transport.
```
