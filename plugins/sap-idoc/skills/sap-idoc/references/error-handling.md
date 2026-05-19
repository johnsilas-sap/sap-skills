# IDoc Error Handling Reference

## Error Status Codes

### Inbound Errors

| Status | Meaning | Next Step |
|---|---|---|
| 51 | Error in application (FM set this) | Fix root cause, reprocess via BD87 |
| 56 | IDoc with errors added | Fix data, reprocess |
| 61 | Processing despite syntax error | Investigate FM logic |
| 63 | Error passing IDoc to application | Check process code config |
| 65 | Error in ALE service | Check ALE config |
| 68 | Error — no further processing | Manual intervention required |
| 70 | IDoc added to queued inbound processing | Normal — awaiting queue |

### Outbound Errors

| Status | Meaning | Next Step |
|---|---|---|
| 02 | Error passing data to port | Check port config (WE21) |
| 04 | Error in control info of EDI subsystem | Check EDI layer |
| 25 | Processing despite warning | Review warning message |
| 26 | Error during syntax check | Fix IDoc data structure |
| 29 | Error in ALE service | Check ALE config |

## Querying Error IDocs

```abap
" All inbound errors for a message type in last 7 days
SELECT docnum, mestyp, direct, status, credat, cretim, sndprn
  FROM edidc
  WHERE mestyp = @lv_mestyp
    AND direct = '2'                   " Inbound
    AND status IN ('51', '56', '68')
    AND credat >= @( sy-datum - 7 )
  ORDER BY credat DESCENDING, cretim DESCENDING
  INTO TABLE @DATA(lt_errors).

" Read status history for a specific IDoc
SELECT status, logdat, logtim, msgty, msgid, msgno, msgv1
  FROM edids
  WHERE docnum = @lv_docnum
  ORDER BY logdat, logtim
  INTO TABLE @DATA(lt_history).

" Get the error message text
LOOP AT lt_history ASSIGNING FIELD-SYMBOL(<s>) WHERE msgty = 'E'.
  MESSAGE ID <s>-msgid TYPE 'I' NUMBER <s>-msgno
    WITH <s>-msgv1 <s>-msgv2 <s>-msgv3 <s>-msgv4
    INTO DATA(lv_msg_text).
ENDLOOP.
```

## Reprocessing via BD87

### Manual Reprocessing (BD87)

```
BD87 → Select IDocs:
  Status:       51          (or 56, 64)
  Message type: DESADV      (filter to specific type)
  Date range:   last 7 days

  → Select all → Execute → Reprocess
```

### Programmatic Reprocessing

```abap
" Reprocess a list of IDoc numbers
DATA: lt_ctrl TYPE TABLE OF edidc,
      lt_data TYPE TABLE OF edidd.

" Read IDoc data
CALL FUNCTION 'IDOC_READ_COMPLETELY'
  EXPORTING
    document_number = lv_docnum
  IMPORTING
    idoc_control    = DATA(ls_ctrl)
  TABLES
    idoc_data       = lt_data.

APPEND ls_ctrl TO lt_ctrl.

" Reprocess
CALL FUNCTION 'IDOC_INBOUND_ASYNCHRONOUS'
  TABLES
    idoc_control_rec_40 = lt_ctrl
    idoc_data_rec_40    = lt_data.

COMMIT WORK.
```

### Mass Reprocessing Program (RBDMANI2)

```
SE38 → RBDMANI2 (Inbound IDoc reprocessing)
  Status:       51
  Message type: WMMBXY
  Execute → Reprocesses all matching IDocs

Schedule as background job after fixing root cause.
```

## Updating IDoc Status Manually

```abap
" Mark IDoc as successfully processed after manual correction
CALL FUNCTION 'IDOC_STATUS_WRITE_TO_DATABASE'
  EXPORTING
    idoc_number = lv_docnum
  TABLES
    idoc_status = VALUE #( (
      docnum = lv_docnum
      status = '53'
      msgty  = 'S'
      msgid  = 'B1'
      msgno  = '011'
      msgv1  = 'Manually corrected' ) ).
COMMIT WORK.

" Lock IDoc from further processing (status 68)
CALL FUNCTION 'IDOC_STATUS_WRITE_TO_DATABASE'
  EXPORTING
    idoc_number = lv_docnum
  TABLES
    idoc_status = VALUE #( (
      docnum = lv_docnum
      status = '68'
      msgty  = 'E'
      msgv1  = 'Cancelled — duplicate document' ) ).
COMMIT WORK.
```

## Editing IDoc Data (WE19)

When data in the IDoc itself is wrong (not an application error):

```
WE19 → Enter IDoc number
     → "Test with existing data" → copy IDoc
     → Edit segment field values directly
     → Change control record if needed (receiver, message type)
     → Execute → triggers inbound FM with corrected data
```

## Monitoring and Alerting

### Custom Error Notification Report

```abap
REPORT z_idoc_error_monitor.

" Run as scheduled job — alert on new errors
DATA lv_threshold TYPE i VALUE 10.

SELECT COUNT(*) FROM edidc
  WHERE status = '51'
    AND credat = @sy-datum
  INTO @DATA(lv_error_count).

IF lv_error_count > lv_threshold.
  " Send alert email
  CALL FUNCTION 'SO_NEW_DOCUMENT_SEND_API1'
    EXPORTING
      document_data = VALUE sofdocchgi1(
        obj_name = 'IDoc Errors'
        obj_descr = |{ lv_error_count } IDoc errors today| )
    TABLES
      object_content = VALUE #(
        ( line = |{ lv_error_count } IDocs in status 51 on { sy-datum }| ) )
      receivers = VALUE #(
        ( receiver = 'idoc-team@company.com' rec_type = 'U' ) ).
ENDIF.
```

### IDoc Active Monitor (WE06)

```
WE06 → Real-time IDoc status overview
  → Color-coded: green (ok), yellow (warning), red (error)
  → Refresh rate configurable
  → Drill down to WE02 for detail
```

## Common Root Causes and Fixes

| Symptom | Root Cause | Fix |
|---|---|---|
| Status 51 — "material not found" | Master data missing on receiver | Create material master, reprocess |
| Status 51 — "partner profile not found" | Missing WE20 entry | Create partner profile for sender |
| Status 51 — "no process code" | WE57/WE42 not configured | Add FM assignment and process code |
| Status 02 — "error passing to port" | RFC destination down | Fix SM59, reprocess |
| Status 26 — syntax error | Z-extension segment missing in WE30 | Add segment to IDoc type, reprocess |
| Stuck at status 30 | RSEOUT00 not scheduled | Schedule outbound dispatch job |
| Stuck at status 64 | RBDAPP01 not scheduled | Schedule inbound processing job |
| Stuck in SM58 | tRFC destination unavailable | Fix target system / RFC, re-execute from SM58 |
