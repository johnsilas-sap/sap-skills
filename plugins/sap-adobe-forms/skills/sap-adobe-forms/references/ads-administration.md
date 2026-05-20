# ADS Administration Reference

## ADS Architecture

Adobe Document Services (ADS) is a Java application that renders XFA forms into PDF. It runs on SAP NetWeaver AS Java and communicates with ABAP via RFC/HTTP.

```
ABAP Print Program
  → FP_JOB_OPEN (initializes ADS session via SM59 RFC destination)
  → Form FM called → ABAP sends XFA template + data to ADS over HTTP
  → ADS Java renders PDF
  → PDF returned to ABAP (or sent to spool/printer)
  → FP_JOB_CLOSE
```

## Key Transactions

| Transaction | Purpose |
|---|---|
| `SFPADM` | ADS admin: test connection, check version, config |
| `SFPDEV` | Developer desktop — connect Adobe LiveCycle to SAP locally |
| `SFPVAR` | Form variants — locale-specific layouts |
| `SICF` | Activate ICF service `/sap/bc/fp` |
| `SM59` | RFC destination to ADS Java server |
| `SFP` | Form Builder — create/edit forms |
| `SP01` | Spool monitor — view generated print jobs |
| `SPAD` | Printer definitions (spool destinations) |

## Testing the ADS Connection

### Via SFPADM

```
SFPADM → Connection Test
  Shows: ADS version, host, port, response time
  If error: check SM59 destination first
```

### Via ABAP

```abap
CALL FUNCTION 'FP_CHECK_DESTINATION'
  EXCEPTIONS
    no_destination    = 1
    destination_error = 2.
IF sy-subrc <> 0.
  " ADS unreachable — check steps below
ENDIF.
```

## SM59 — ADS RFC Destination

```
SM59 → Create/Display RFC Destination
  Name:          ADS                     " Standard name expected by FP_*
  Type:          G (HTTP connection to external server)
  Target host:   <ADS Java server hostname>
  Service No.:   50000                   " Or as configured
  Path prefix:   /AdobeDocumentServices/Config

  Logon & Security tab:
    Logon method: Basic Authentication
    User:         ADS_AGENT              " Technical user on Java side
    Password:     <configured in J2EE>
```

```abap
" Test SM59 destination programmatically
DATA lo_http TYPE REF TO if_http_client.
CALL METHOD cl_http_client=>create_by_destination
  EXPORTING  destination = 'ADS'
  IMPORTING  client      = lo_http
  EXCEPTIONS OTHERS = 1.
lo_http->send( ).
lo_http->receive( ).
DATA(lv_code) = lo_http->response->get_status( ).
" Expected: 200
```

## SICF — ICF Service Activation

```
SICF → Navigate to: /default_host/sap/bc/fp
  Right-click → Activate Service

  Also activate:
    /default_host/sap/bc/fp_submit   (for interactive form POST submit)
    /default_host/sap/bc/fpads       (ADS communication endpoint)
```

## Common Errors and Fixes

### Error: "No destination ADS defined"

```
Cause:  SM59 destination "ADS" does not exist
Fix:    Create SM59 destination (see above)
        Or run SFPADM → Configure → Create Standard Destinations
```

### Error: "ADS connection failed" / HTTP 401

```
Cause:  Wrong credentials in SM59 Logon tab
Fix:    Verify ADS_AGENT user and password on Java side
        SFPADM → Connection → Reset ADS Password
```

### Error: "ADS not reachable" / Network timeout

```
Cause:  Firewall or wrong host/port in SM59
Fix:    Confirm Java AS host and port (default 50000)
        Test network: ping / telnet from ABAP host
        Check ICM proxy settings (SMICM)
```

### Error: "Form not found" / function module not generated

```
Cause:  Form not activated in SFP, or FM not generated
Fix:    SFP → Open form → Activate (Ctrl+F3)
        FP_FUNCTION_MODULE_NAME will then resolve
```

### Error: "FP_JOB_OPEN cancel = 1"

```
Cause:  Print dialog appeared and was cancelled
Fix:    Set ls_outputparams-nodialog = abap_true
```

### Error: "FP_JOB_OPEN system_error = 3"

```
Cause:  ADS Java crash or out of memory on Java side
Fix:    Check Java AS logs (log viewer in NWA)
        Restart ADS application if needed
        Check ADS memory in SFPADM → Performance
```

### Error: Barcode not rendering / blank in PDF

```
Cause:  Value contains non-printable characters or is too long
Fix:    Condense and strip special chars in ABAP before passing
        Check Code128 character set — avoid non-ASCII
        Verify binding path in layout matches data node name exactly
```

### Error: Arabic/Hebrew text rendering incorrectly (RTL)

```
Cause:  ADS requires licensed font packs for RTL languages
Fix:    Install SAP-provided Arabic/Hebrew font packs on ADS
        Check SFPADM → Fonts
```

## ADS Performance Tuning

```
SFPADM → Performance
  Thread pool size:  8–16 (depending on server CPU)
  Queue size:        50

" For batch printing (100+ forms):
" Open one FP_JOB and call FM in loop — one ADS session per job
" Each FP_JOB_OPEN/CLOSE creates a new ADS session (expensive)
```

```abap
" Efficient batch: one job, loop inside
CALL FUNCTION 'FP_JOB_OPEN' CHANGING ie_outputparams = ls_outputparams.

CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
  EXPORTING i_name = 'ZEWM_WH_LABEL' IMPORTING e_funcname = lv_fm.

LOOP AT lt_labels ASSIGNING FIELD-SYMBOL(<label>).
  CALL FUNCTION lv_fm EXPORTING is_label = <label>.
ENDLOOP.

CALL FUNCTION 'FP_JOB_CLOSE'.
```

## Developer Desktop Setup (SFPDEV)

```
SFPDEV → Configure
  Connects local Adobe LiveCycle Designer to SAP
  Allows designing forms with live SAP data context

  Prerequisites:
    Adobe LiveCycle Designer installed locally
    SAP GUI installed (same machine)
    SAP logon to development system
```

## ADS Version Check

```abap
DATA lv_version TYPE string.

CALL FUNCTION 'FP_GET_ADS_VERSION'
  IMPORTING ev_version = lv_version
  EXCEPTIONS no_destination = 1 destination_error = 2.

" lv_version example: "6.80.1.234 (ADS Java Build 234)"
```

## Checking Active ICF Services

```abap
" List all active FP services
DATA lt_services TYPE TABLE OF icfservice.
CALL FUNCTION 'ICF_TREE_GET_SERVICES'
  EXPORTING sap_node = '/sap/bc/fp'
  TABLES    services = lt_services
  EXCEPTIONS node_not_found = 1.

LOOP AT lt_services WHERE icf_activ = 'X'.
  " Active FP service found
ENDLOOP.
```
