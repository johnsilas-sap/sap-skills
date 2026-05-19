# TM Track and Trace Reference

## Architecture

SAP TM uses the `/SAPTRX/` (Track and Trace) framework for event management:

```
External Event Source (carrier EDI, GPS, mobile app)
  → Event posting (/SAPTRX/TRACK_EVENT_POST)
  → Application object (/SAPTRX/APPL_OB)
  → Expected vs actual event comparison
  → Exception detection and alerting
```

## Event Categories

| Category | Description |
|---|---|
| `DEPARTURE` | Vehicle/shipment departs a location |
| `ARRIVAL` | Vehicle/shipment arrives at a location |
| `LOADING` | Freight loaded onto vehicle |
| `UNLOADING` | Freight unloaded at destination |
| `DELAY` | Delay notification from carrier |
| `EXCEPTION` | Damage, loss, wrong delivery |
| `POD` | Proof of delivery |
| `CUSTOMS_CLEAR` | Customs clearance completed |

## Posting Tracking Events

### Standard Event Post

```abap
CALL FUNCTION '/SAPTRX/TRACK_EVENT_POST'
  EXPORTING
    iv_appl_table    = '/SCMTMS/D_FRO_H'   " Document type
    iv_appl_key      = lv_freight_order_id
    iv_event_code    = 'DEPARTURE'
    iv_actual_date   = sy-datum
    iv_actual_time   = sy-uzeit
    iv_location_id   = lv_departure_location
    iv_location_desc = lv_location_name     " Free text fallback
    iv_remarks       = lv_driver_notes
  IMPORTING
    et_return        = DATA(lt_messages).
COMMIT WORK.
```

### Post Event with GPS Coordinates

```abap
CALL FUNCTION '/SAPTRX/TRACK_EVENT_POST'
  EXPORTING
    iv_appl_table  = '/SCMTMS/D_FRO_H'
    iv_appl_key    = lv_fo_id
    iv_event_code  = 'IN_TRANSIT'
    iv_actual_date = sy-datum
    iv_actual_time = sy-uzeit
    iv_latitude    = lv_lat             " Decimal degrees e.g. 41.8781
    iv_longitude   = lv_lon             " Decimal degrees e.g. -87.6298
    iv_altitude    = lv_alt             " Meters above sea level
  IMPORTING
    et_return      = DATA(lt_messages).
COMMIT WORK.
```

### Proof of Delivery (POD)

```abap
CALL FUNCTION '/SAPTRX/TRACK_EVENT_POST'
  EXPORTING
    iv_appl_table   = '/SCMTMS/D_FRO_H'
    iv_appl_key     = lv_fo_id
    iv_event_code   = 'POD'
    iv_actual_date  = lv_delivery_date
    iv_actual_time  = lv_delivery_time
    iv_location_id  = lv_dst_location
    iv_receiver     = lv_receiver_name  " Who signed for the delivery
    iv_remarks      = lv_delivery_notes
  IMPORTING
    et_return       = DATA(lt_messages).
COMMIT WORK.
```

## Reading Tracking History

```abap
" Full event history for a freight order
SELECT event_code, actual_date, actual_time, location_id,
       latitude, longitude, remarks, created_by
  FROM /saptrx/appl_ob
  WHERE appl_table = '/SCMTMS/D_FRO_H'
    AND appl_key   = @lv_fo_id
  ORDER BY actual_date, actual_time
  INTO TABLE @DATA(lt_events).

" Last known position
SELECT SINGLE *
  FROM /saptrx/appl_ob
  WHERE appl_table = '/SCMTMS/D_FRO_H'
    AND appl_key   = @lv_fo_id
    AND latitude   IS NOT NULL
  ORDER BY actual_date DESCENDING, actual_time DESCENDING
  INTO @DATA(ls_last_position).
```

## Expected vs. Actual Event Comparison

TM compares actual events against planned milestones:

```abap
" Read expected events (from freight order stages)
SELECT stage_num, src_loc_id, dst_loc_id, dep_date, arr_date
  FROM /scmtms/d_stage
  WHERE tor_id = @lv_fo_id
  INTO TABLE @DATA(lt_planned).

" Compare: find stages with no actual ARRIVAL event
LOOP AT lt_planned ASSIGNING FIELD-SYMBOL(<stage>).
  SELECT COUNT(*) FROM /saptrx/appl_ob
    WHERE appl_table  = '/SCMTMS/D_FRO_H'
      AND appl_key    = @lv_fo_id
      AND event_code  = 'ARRIVAL'
      AND location_id = @<stage>-dst_loc_id
    INTO @DATA(lv_count).
  IF lv_count = 0 AND sy-datum > <stage>-arr_date.
    " Late arrival — no event posted past planned date
  ENDIF.
ENDLOOP.
```

## Carrier EDI Event Integration

Carriers send tracking updates via EDI (IFTSTA — Status message):

```
Carrier EDI System
  → IFTSTA IDoc / API call
  → /SAPTRX/IDOC_INPUT_IFTSTA (standard inbound processor)
  → /SAPTRX/TRACK_EVENT_POST (internal)
  → Event stored in /SAPTRX/APPL_OB
```

### Custom EDI Event Receiver (Function Module)

```abap
FUNCTION z_tm_carrier_event_receive.
  " Called by carrier API or custom IDoc processor
  IMPORTING
    value(iv_fo_id)      TYPE /scmtms/de_tor_id
    value(iv_event_code) TYPE /saptrx/de_event_code
    value(iv_event_date) TYPE d
    value(iv_event_time) TYPE t
    value(iv_location)   TYPE /saptrx/de_location_id
    value(iv_remarks)    TYPE string.

  CALL FUNCTION '/SAPTRX/TRACK_EVENT_POST'
    EXPORTING
      iv_appl_table  = '/SCMTMS/D_FRO_H'
      iv_appl_key    = iv_fo_id
      iv_event_code  = iv_event_code
      iv_actual_date = iv_event_date
      iv_actual_time = iv_event_time
      iv_location_id = iv_location
      iv_remarks     = iv_remarks
    IMPORTING
      et_return      = DATA(lt_msg).

  IF line_exists( lt_msg[ type = 'E' ] ).
    RAISE event_post_failed.
  ENDIF.

  COMMIT WORK.
ENDFUNCTION.
```

## Alert and Exception Management

```abap
" Create exception alert for delayed freight order
CALL FUNCTION '/SCMTMS/EXCEPTION_CREATE'
  EXPORTING
    iv_tor_id      = lv_fo_id
    iv_exc_code    = 'LATE_ARRIVAL'
    iv_planned_arr = ls_stage-arr_date
    iv_actual_arr  = sy-datum
    iv_remarks     = 'Freight order arrived 2 days late'
  IMPORTING
    ev_exc_id      = DATA(lv_exception_id)
    et_return      = DATA(lt_messages).

" Read open exceptions
SELECT tor_id, exc_code, planned_arr, actual_arr, status
  FROM /scmtms/d_exception
  WHERE status   = 'OPEN'
    AND exc_code = 'LATE_ARRIVAL'
  INTO TABLE @DATA(lt_open_exceptions).
```

## Monitoring Transactions

| Transaction | Purpose |
|---|---|
| `/SCMTMS/MON_FO` | Freight order monitor — includes tracking status column |
| `/SAPTRX/MON` | Track and trace event monitor |
| `/SAPTRX/EVTLOG` | Event log viewer per document |
