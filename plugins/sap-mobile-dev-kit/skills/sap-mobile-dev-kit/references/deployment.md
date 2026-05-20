# MDK Deployment Reference

## Deployment Path

```
BAS / VS Code
  → MDK Build (generates native app bundle or QR code)
  → Deploy to SAP Mobile Services (BTP)
  → MDK Client (iOS/Android) downloads app via QR / app store
```

## SAP Mobile Services Setup (BTP Cockpit)

### 1. Create Mobile App Registration

```
BTP Cockpit → Mobile Services → Mobile Applications → Native/MDK
  → New Application
  App ID:      com.mycompany.ewm.warehouse    (must match Metadata.json)
  Name:        EWM Warehouse App
  Vendor:      My Company
  License:     (select appropriate)
```

### 2. Configure Destination (Backend Connectivity)

```
Mobile Services → App → Mobile Connectivity → New
  Name:        EWM_BACKEND                   (matches .service file Destination field)
  URL:         https://my-s4-system.company.com
  Proxy Type:  Internet (or On-Premise via Cloud Connector)
  Auth Type:   Basic / OAuth2 / Principal Propagation

" For on-premise SAP EWM via Cloud Connector:
  Proxy Type:    On-Premise
  Location ID:   MY_CLOUD_CONNECTOR_LOCATION
  URL:           http://ewm-host:8000             (internal address)
```

### 3. Enable Features

```
Mobile Services → App → Features → Enable:
  ✓ Mobile Development Kit                (required for MDK)
  ✓ Offline OData                         (if using offline mode)
  ✓ Push Notifications                    (if sending push)
  ✓ Mobile Connectivity                   (backend routing)
```

## Deploying from BAS

```
BAS → MDK App → Right-click project root
  → "MDK: Deploy"
    Target: SAP Mobile Services
    → Logs in to BTP using your BAS credentials
    → Uploads app bundle to Mobile Services
    → Generates QR code for client download
```

## Deploying from CLI (mdk tool)

```bash
# Install MDK CLI
npm install -g @sap/mdk-tools

# Login to BTP
mdk login --url https://myaccount.hana.ondemand.com \
          --client-id ... --client-secret ...

# Deploy
mdk deploy \
  --project ./MyWarehouseApp.mdkproject \
  --app-id com.mycompany.ewm.warehouse \
  --environment mobile-services-url

# Check deployment status
mdk app-list
```

## MDK Client Setup (iOS/Android)

```
1. Install SAP Mobile Services Client from App Store / Google Play
   (or distribute custom-branded client built with MDK SDK)

2. Open client → Scan QR code from Mobile Services cockpit
   OR enter:
     URL:       https://mobile-services-url/mobileservices/origin/hcpms
     User/Pass: BTP credentials

3. App downloads to device
4. Offline store initializes (first sync downloads data)
```

## Custom MDK Client (Branded App)

For distributing as a branded app (your company logo, app name):

```bash
# Prerequisites: Xcode (iOS) or Android Studio + NDK

# Get MDK SDK from SAP Mobile Services
# Mobile Services → Downloads → MDK Client SDK (iOS/Android)

# Build custom client
cd /path/to/mdkclient-sdk
./build-ios.sh \
  --app-id com.mycompany.ewm.warehouse \
  --app-name "EWM Warehouse" \
  --server-url https://mobile-services-url

# Output: .ipa (iOS) or .apk/.aab (Android)
# Distribute via MDM or enterprise app store
```

## Update Deployment (App Changes)

```
BAS → MDK: Deploy
  → Uploads new version to Mobile Services

On device:
  MDK Client detects new version → prompts user
  → User taps "Update" → app bundle downloads (OData store preserved)
  → App restarts with new UI without reinstall
```

## Push Notifications Setup

### Mobile Services Configuration

```
Mobile Services → App → Push Notification
  iOS:     Upload APNs certificate (.p12) or APNs Auth Key (.p8)
  Android: Enter Firebase Server Key (from Google Firebase Console)
```

### Sending Push from ABAP

```abap
" Send push to specific user via REST
DATA lo_http TYPE REF TO if_http_client.

CALL METHOD cl_http_client=>create_by_destination
  EXPORTING  destination = 'MOBILE_SERVICES_PUSH'
  IMPORTING  client      = lo_http.

lo_http->request->set_method( 'POST' ).
lo_http->request->set_header_field(
  name = 'Content-Type' value = 'application/json' ).

DATA(lv_body) = |{"alert":"New delivery { lv_dlv_no } ready","user":"{ lv_user }"}|.
lo_http->request->set_cdata( lv_body ).
lo_http->send( ).
lo_http->receive( ).
```

### Push Handler in MDK Rule

```javascript
// Triggered when push notification is tapped
export default function OnPushReceived(context) {
  const pushData = context.getPushData();
  const deliveryNo = pushData.deliveryNo;

  if (deliveryNo) {
    // Navigate directly to the referenced delivery
    return context.executeAction({
      '_Type': 'MDK.NavigationAction',
      'PageToOpen': '/MyWarehouseApp/Pages/DeliveryDetail.page',
      'TransitionType': 'Push'
    });
  }
}
```

## Troubleshooting Deployment

### "App ID mismatch"

```
Cause:  AppId in Metadata.json ≠ App ID registered in Mobile Services
Fix:    Match exactly: com.mycompany.ewm.warehouse (case-sensitive)
```

### "Destination not reachable"

```
Cause:  BTP destination not configured or Cloud Connector tunnel down
Fix:    BTP Cockpit → Connectivity → Destinations → Test connection
        Check Cloud Connector → Tunnel state
```

### OData service returns 401 in offline mode

```
Cause:  Auth token expired during offline period
Fix:    Mobile Services → App → Security → Increase token validity
        Or: implement token refresh in OnResume rule
```

### App update not showing on device

```
Cause:  Client caches previous version
Fix:    Force update: Mobile Services → App → Publish → Force Update
        Or: delete and reinstall app on device
```

## Environment Management (Dev / QA / Prod)

```bash
# Deploy to dev environment
mdk deploy --app-id com.mycompany.ewm.warehouse.dev \
           --environment dev-mobile-services-url

# Deploy to prod
mdk deploy --app-id com.mycompany.ewm.warehouse \
           --environment prod-mobile-services-url
```

Use separate Mobile Services instances or separate app registrations per environment. The destination in each environment points to the correct backend (DEV / QA / PROD).
