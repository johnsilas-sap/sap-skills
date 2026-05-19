# EWM RF Performance Monitoring Reference

## Overview

EWM's built-in performance framework captures timing at three layers:

| Layer | Metric | Captured By |
|---|---|---|
| ABAP backend | Backend processing time (`Bck. ms`) | SAP standard (Note 1690850) |
| ITS/network | Round-trip to ITS server (`ITS ms`) | SAP ICM logging (section 4.3.7 of guide) |
| Frontend | GUI response + render (`GUI ms`) | Custom JS monitor (EWM_RF_Perf project) |

## Prerequisites

Run `ZABAP_VERIFY_PREREQS` (SE38) to validate all prerequisites before setup:

| SAP Note | Delivers | Check |
|---|---|---|
| 1690850 | RFUI performance measurement; `/SCMB/PFM_RFUI` table | Table exists in DD02L |
| 1595305 | `ewmrf_store_gui` routine in `/SCMB/PFM2_GUI_STORE` | Program exists in TRDIR |
| ST/A-PI 01P | EWM_RF_Analysis tool in ST13 | ST13 → PERF_TOOL shows EWM_RF_Analysis |

## Viewing Data in ST13

```
ST13
  → Performance Tools
  → PERF_TOOL
  → EWM_RF_Analysis

Filter options:
  Warehouse Number: (your LGNUM)
  RF User: (SAP username on scanner)
  From Date / To Date

Tabs:
  Transactions  → backend times per RF screen (Bck. ms)
  Load GUI      → merges frontend data; adds GUI ms column
  ITS           → ITS server times (requires ICM access logging)
```

## Enabling Performance Logging

### Per Resource (Scanner)

1. `/SCWM/RSRC` → find the resource record
2. Check **Performance Logging** (`PFMLOG = X`)
3. User must log off ITSmobile and log back in

### Programmatically Enable Logging

```abap
UPDATE /scwm/rsrc
  SET pfmlog = abap_true
  WHERE lgnum = @lv_lgnum
    AND rsrc  = @lv_resource.
COMMIT WORK.
```

## Frontend Timing (EWM_RF_Perf Tool)

The EWM_RF_Perf project captures two frontend metrics not available in the standard tool:

| Metric | Formula | What It Measures |
|---|---|---|
| `GUI Response Time` | `T_page_start − T_submit` | Network transit + ITS rendering + ABAP (end-user wait) |
| `GUI Rendering Time` | `window.load − T_page_start` | Device browser rendering HTML |

### How It Works

1. JavaScript (`ewm_rf_perf_monitor.js`) is injected into every ITSmobile page via Velocity profile
2. On ENTER/button press, `T_submit` is stored in `localStorage`
3. On next page load, metrics are calculated and POSTed to `/sap/bc/ewmrf_perf`
4. ABAP handler `ZCL_EWM_RF_PERF_HANDLER` persists via `ewmrf_store_gui`
5. Data appears in ST13 → EWM_RF_Analysis after clicking **Load GUI**

### ICF Endpoint Setup

```
SICF → default_host/sap/bc/ewmrf_perf
Handler class: ZCL_EWM_RF_PERF_HANDLER
Logon: service user with EWM RF auth
```

Test the endpoint:
```bash
curl -X POST "http://YOUR_SAP_HOST:PORT/sap/bc/ewmrf_perf" \
  -d "gui_resp=1234&gui_render=456&ts=2026-01-15T08:22:31.000Z&device=SCANNER01"
# Expected: HTTP 204 No Content
```

### Troubleshooting Frontend Data

| Symptom | Check |
|---|---|
| GUI ms = 0 in ST13 | Set `DEBUG: true` in JS, check browser console in Velocity |
| ICF returns 404 | SICF — service not activated |
| ICF returns 500 | SM21 / SLG1 — likely wrong PERFORM program name in handler |
| Device maps to ITS service user | RSRC lookup failed — check device name vs `rsrc` field in `/SCWM/RSRC` |
| No data after curl test | Verify `ewmrf_store_gui` program name matches your system's post-Note version |

## Performance Data Tables

| Table | Content |
|---|---|
| `/SCMB/PFM_RFUI` | RF UI timing records (backend) — requires Note 1690850 |
| `/SCMB/PFM2_GUI` | Frontend GUI timing records — requires Note 1595305 |

### Query Raw Performance Data

```abap
" Read backend timing for a user/date
SELECT *
  FROM /scmb/pfm_rfui
  WHERE uname  = @lv_rf_user
    AND rfdate = @sy-datum
  ORDER BY rfdate, rftime
  INTO TABLE @DATA(lt_perf).

" Average backend time per transaction code
SELECT tcode, AVG( rftime_ms ) AS avg_ms, COUNT(*) AS cnt
  FROM /scmb/pfm_rfui
  WHERE lgnum  = @lv_lgnum
    AND rfdate BETWEEN @lv_from AND @lv_to
  GROUP BY tcode
  ORDER BY avg_ms DESCENDING
  INTO TABLE @DATA(lt_by_tcode).
```

## Data Retention

Performance data grows quickly. Schedule the standard cleanup job:

```
SE38 → /SCMB/PFM2_DELETE
  Max age: 3–5 days recommended
  Schedule as background job via SM36
```

Production checklist item: ensure this job runs daily or on a schedule matching your data volume.
