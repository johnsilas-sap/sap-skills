# EWM RF Mobile Reference

## Architecture Overview

```
Velocity/Ivanti Device Browser
  → HTTP to SAP ITS (Internet Transaction Server)
  → ITSmobile renders HTML RF screens
  → /SCWM/CL_RF_BLL_PROCESSOR (Business Logic Layer)
  → EWM warehouse tasks, quants, resources
```

The RF UI layer runs entirely inside SAP ITS (not a standalone app). Screens are HTML forms
served to the device browser. All SAP EWM business logic runs server-side in ABAP.

## RF Logical Transactions

RF transactions are configured in `/SCWM/RFUI`. Each logical transaction maps to:
- A process type (putaway, pick, GR, GI, inventory, etc.)
- A series of screen steps
- BADI enhancement hooks between steps

### Standard RF Transaction Codes

| T-Code | Process |
|---|---|
| `/SCWM/RF01` | Goods receipt / unloading |
| `/SCWM/RF02` | Putaway |
| `/SCWM/RF03` | Pick (outbound) |
| `/SCWM/RF04` | Goods issue confirmation |
| `/SCWM/RF05` | Internal stock transfer |
| `/SCWM/RF06` | Replenishment |
| `/SCWM/RF10` | Physical inventory count |
| `/SCWM/RF11` | HU management |
| `/SCWM/RFLOGIN` | Resource login / device assignment |

## Resource Management

### Resource Login Flow

1. Scanner opens ITSmobile URL
2. User authenticates via SAP logon
3. `/SCWM/RFLOGIN` assigns the SAP user to a resource (device) in `/SCWM/RSRC`
4. All subsequent WO/WT assignments use the resource record

### /SCWM/RSRC Table Key Fields

| Field | Description |
|---|---|
| `LGNUM` | Warehouse number |
| `RSRC` | Resource name (usually matches device hostname) |
| `RSRC_TYPE` | Resource type (e.g. 0001 = Forklift, 0002 = Hand scanner) |
| `UNAME` | Currently logged-in SAP user |
| `LGPLA` | Current position (last scanned bin) |
| `PFMLOG` | Performance logging enabled flag |

### Query Resource → User Mapping

```abap
" Exact match
SELECT SINGLE uname
  FROM /scwm/rsrc
  WHERE lgnum = @lv_lgnum
    AND rsrc   = @lv_device_id
  INTO @DATA(lv_user).

" Partial match (device IDs may have suffixes in Velocity)
IF sy-subrc <> 0.
  SELECT SINGLE uname
    FROM /scwm/rsrc
    WHERE lgnum = @lv_lgnum
      AND rsrc LIKE @( lv_device_id && '%' )
    INTO @lv_user.
ENDIF.
```

### Enable Performance Logging on a Resource

1. Open `/SCWM/RSRC`
2. Find the resource record
3. Check **Performance Logging** (`PFMLOG`) checkbox
4. RF user must log off and log back in for the change to take effect

## RF BADI: /SCWM/EX_RF_BLL_PROCESSOR

This is the primary hook for custom RF screen logic. It fires on every screen transition.

### Enhancement Spot
`/SCWM/ES_RF_BLL_PROCESSOR`

### BADI Interface
`/SCWM/IF_EX_RF_BLL_PROCESSOR`

### Method: PROCESS_REQUEST

```abap
METHOD /scwm/if_ex_rf_bll_processor~process_request.
  " Fires on every RF screen transition
  " Parameters:
  "   is_req   — current request (scanned input, function key pressed, tcode)
  "   cs_resp  — response to render on screen (modify to change display)
  "   iv_lgnum — warehouse number

  CASE is_req-linfld-tcode.
    WHEN '/SCWM/RF03'.  " Pick transaction
      " Validate scanned quantity before confirming
      IF is_req-linfld-vbeln IS NOT INITIAL.
        " Custom validation logic here
        IF lv_invalid = abap_true.
          cs_resp-msgty  = 'E'.
          cs_resp-msgid  = 'ZEWM_RF'.
          cs_resp-msgno  = '001'.
          cs_resp-msgv1  = 'Custom validation failed'.
        ENDIF.
      ENDIF.
  ENDCASE.
ENDMETHOD.
```

### Common Use Cases

| Use Case | Approach |
|---|---|
| Add custom field validation | Check `is_req-linfld` values, set error in `cs_resp` |
| Force a specific bin | Override `cs_resp-lgpla` |
| Log RF activity | Write to custom table in `process_request` |
| Skip a screen step | Set `cs_resp-skip_screen = abap_true` |
| Redirect to different tcode | Set `cs_resp-tcode` |

## Velocity / Ivanti Integration

Velocity Management Server (VMS) sits between the device and SAP ITSmobile. It handles:
- Session management and device profiles
- HTML injection into ITSmobile pages (used by EWM RF Performance Monitor)
- Device group management and profile push

### Velocity Profile — Script Injection

The `2 velocity_profile_inject.xml` in EWM_RF_Perf injects JavaScript into every
ITSmobile page `<head>`. Key elements:

```xml
<injectHTML>
  <target>head</target>
  <position>append</position>
  <content><![CDATA[
    <script>window.__ewmrf_page_start = Date.now();</script>
    <script src="http://YOUR_HOST/ewm_rf_perf_monitor.js" async="false"></script>
  ]]></content>
</injectHTML>

<!-- Expose device name to JavaScript -->
<injectHTML>
  <target>head</target>
  <position>prepend</position>
  <content><![CDATA[
    <meta name="velocityDeviceId" content="${deviceName}" />
  ]]></content>
</injectHTML>
```

### Velocity Version Notes

| Version | Script Injection Method |
|---|---|
| Velocity 1.x / 2.x | Paste `<script>` tag in "Custom HTML" field of session profile |
| Velocity 2023+ (Ivanti branded) | XML profile import supported |

## Custom ICF Endpoints for RF Devices

RF devices can POST to custom SAP endpoints via SAP ICF (Internet Communication Framework).
Used by EWM RF Performance Monitor to receive timing data.

```abap
" Pattern: implement IF_HTTP_EXTENSION for a custom ICF service
CLASS zcl_my_rf_endpoint DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    INTERFACES if_http_extension.
ENDCLASS.

CLASS zcl_my_rf_endpoint IMPLEMENTATION.
  METHOD if_http_extension~handle_request.
    DATA(lv_method) = server->request->get_method( ).

    IF lv_method = 'OPTIONS'.  " CORS pre-flight
      server->response->set_header_field( name = 'Access-Control-Allow-Origin' value = '*' ).
      server->response->set_status( code = 204 reason = 'No Content' ).
      RETURN.
    ENDIF.

    DATA(lv_param) = server->request->get_form_field( 'my_param' ).
    " ... business logic ...
    server->response->set_status( code = 204 reason = 'No Content' ).
  ENDMETHOD.
ENDCLASS.
```

### Register ICF Service

1. `SICF` → Navigate to `default_host/sap/bc`
2. Right-click → **Create Service**
3. Set handler class to your `ZCL_*` implementation
4. Set logon data (service user)
5. Right-click → **Activate Service**

## Troubleshooting RF Issues

| Symptom | Check |
|---|---|
| Scanner can't reach SAP | SMICM → check ICM active, ports open |
| User logs in but no resource assigned | `/SCWM/RSRC` — verify rsrc name matches device hostname |
| WO not appearing on scanner | Queue assignment — `/SCWM/TQACT`, resource type match |
| BADI not firing | SE18 — verify implementation is active; check filter values |
| Performance data not in ST13 | SAP Notes 1690850/1595305; check ICF service; SLG1 for errors |
