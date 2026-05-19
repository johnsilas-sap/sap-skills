# Partner Profiles and ALE Configuration Reference

## Partner Profiles (WE20)

Partner profiles define how IDocs are exchanged with each business partner.

### Partner Types

| Type | Code | Used For |
|---|---|---|
| Logical System | `LS` | SAP-to-SAP via ALE (ERP↔EWM, ERP↔TM) |
| Vendor | `LI` | EDI with suppliers (ORDERS, DESADV, INVOIC) |
| Customer | `KU` | EDI with customers (ORDERS, DELVRY, INVOIC) |
| Bank | `B` | Payment/banking IDocs |
| User | `US` | User-specific distribution |

### Inbound Parameters

```
WE20 → Partner type LS → Partner number EWM_100
  Inbound parameters → Add entry:
    Message type:    WMMBXY
    Process code:    WMMBXY_IN         (or custom Z process code)
    Trigger:         Trigger immediately  / Background
    Cancel processing after syntax error: ✓
    Processing by function module:  ✓
```

### Outbound Parameters

```
WE20 → Partner type LI → Partner number 0000001000 (Vendor)
  Outbound parameters → Add entry:
    Message type:    ORDERS
    Basic type:      ORDERS05          (or ZORDERS05 if extended)
    Extension:       (blank unless using extension type)
    Receiver port:   FILE_VENDOR       (or RFC port for SAP partner)
    Output mode:     4 = Transfer IDoc immediately
    IDoc type:       ORDERS05
    Pack. size:      0 = no packing
```

### Programmatically Read Partner Profile

```abap
SELECT *
  FROM edpp40                          " Partner profile outbound view
  WHERE rcvprt = 'LS'
    AND rcvprn = @lv_receiver
    AND mestyp = @lv_message_type
  INTO TABLE @DATA(lt_profiles).

" Or inbound:
SELECT *
  FROM edpi40                          " Partner profile inbound view
  WHERE sndprt = 'LS'
    AND sndprn = @lv_sender
    AND mestyp = @lv_message_type
  INTO TABLE @DATA(lt_inbound).
```

## ALE Distribution Model (BD64)

The distribution model defines which IDocs flow between which systems.

### Create Distribution Model

```
BD64 → Edit → Create model view
  Short name: MYMODEL
  Long name:  My ALE Distribution Model

  Add message type:
    Sender:    PRD_800        (ERP logical system)
    Receiver:  EWM_100        (EWM logical system)
    Message:   WMMBXY

  Add message type:
    Sender:    EWM_100
    Receiver:  PRD_800
    Message:   WMCATO         (TO confirmation back to ERP)
```

### Generate Partner Profiles from Model

After maintaining BD64, auto-generate WE20 entries:

```
BD64 → Environment → Generate partner profiles
  Select systems → Execute
  → Creates WE20 entries for all message types in the model
```

### Distribute Model to Target Systems

```
BD64 → Edit → Distribute model
  → Sends the model definition via ALE to partner systems
  → Partner systems generate their own WE20 entries
```

## Logical System Configuration (SALE)

```
SALE → Basic settings → Logical systems
  → Define logical system:
      Name:        PRD_800
      Description: ERP Production client 800

  → Assign logical system to client:
      Client 800 → PRD_800

SM59 → Create RFC destination:
  Name:           EWM_100
  Type:           3 (ABAP connection)
  Target host:    ewm-server.company.com
  System number:  00
  Logon:          technical user with IDoc authorization
```

## Common ALE Topologies

### Hub-and-Spoke (Central ERP)

```
          ERP (PRD_800)
         /      |       \
    EWM_100   TMS_100   SRM_200
```

### Point-to-Point (Direct)

```
ERP_800 ←────────→ EWM_100
                ←→ TMS_100
```

### Message Routing via Central Instance

```
External EDI ──→ SAP PI/PO/Integration Suite ──→ ERP
                                              ──→ EWM
                                              ──→ TM
```

## Filtering Distribution (BD64 Reduced Message)

Limit which IDocs are sent based on field values:

```
BD64 → Add filter:
  Message type:   MATMAS
  Filter object:  MTART (Material type)
  Value:          FERT, HALB        " Only finished/semi-finished
```

```abap
" Programmatic filter — in customer function during MATMAS outbound
CALL CUSTOMER-FUNCTION '001'
  TABLES
    object_tab = lt_objects.

" Filter out materials not relevant for receiver
DELETE lt_objects WHERE mtart NOT IN ('FERT', 'HALB').
```

## Serialization Groups

Control the order in which IDocs are processed (important for master data):

```
BD64 → Serialization:
  Add serialization group for MATMAS → DEBMAS
  Ensures material master arrives before customer master
```

## Checking ALE Configuration

```abap
" Verify logical system assignment
SELECT logsys FROM t000 WHERE mandt = @sy-mandt INTO @DATA(lv_logsys).

" Check distribution model
SELECT *
  FROM tbdls                           " Distribution model — message types
  WHERE logsys_snd = @lv_logsys
  INTO TABLE @DATA(lt_model_entries).

" Verify RFC destination
CALL FUNCTION 'RFC_PING'
  DESTINATION lv_rfc_dest
  EXCEPTIONS
    communication_failure = 1
    system_failure        = 2.
IF sy-subrc <> 0.
  " RFC destination unreachable
ENDIF.
```
