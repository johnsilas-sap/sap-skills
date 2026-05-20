# Monitoring and Deployment Reference

## Message Processing Log (MPL)

```
Integration Suite → Monitor → Monitor Message Processing
  Filters:
    Status:         Completed / Failed / Retry / Discarded / Processing
    iFlow:          Delivery-Confirmation-iFlow
    Time Range:     Last 1 hour / 24 hours / custom
    Correlation ID: (trace specific message through multiple iFlows)
  
  Message Detail:
    Properties:     Exchange properties at time of failure
    Headers:        Message headers
    Payload:        Request and response body (if logging enabled)
    Log Levels:     INFO / DEBUG (set per iFlow)
```

## Log Levels

```
iFlow Properties → Log Configuration:
  Log Level:  None / Error / Info / Debug (Trace)

" Debug: logs message body at each step — use only temporarily, data is stored 10 min
" Info:  logs step entry/exit — lightweight, recommended for prod
" None:  no logging (fastest, no visibility)
```

## Trace Mode (Step-by-Step Debugging)

```
Monitor → Monitor Message Processing → Select FAILED message → Trace
  " Shows payload at every step: before/after each Content Modifier, Script, Mapping
  " Automatically enabled for FAILED messages if Log Level = Trace

" Enable trace for a running iFlow:
Monitor → Manage Integration Content → Select iFlow → Set Log Level → Trace
  " Trace data retained for 10 minutes only
```

## Attaching Custom Log Messages

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import java.util.logging.Logger

def Message processData(Message message) {
    def log = Logger.getLogger('DeliveryScript')
    def deliveryNo = message.getProperty('DeliveryNo')
    
    log.info("Processing delivery: ${deliveryNo}")
    
    // Add structured log entry to MPL
    def mplMsg = message.getProperty('SAP_MessageProcessingLogID')
    message.setProperty('CustomLogEntry', "Delivery ${deliveryNo} processed successfully")
    
    return message
}
```

## iFlow Artifact Status

```
Monitor → Manage Integration Content
  Status:  Started / Stopped / Error / Starting / Stopping
  
  Actions:
    Start:    Deploy and start iFlow
    Stop:     Stop without undeploying (keeps last payload log)
    Undeploy: Remove from runtime
    Download: Download iFlow as .zip for backup or CI/CD
```

## Deployment Workflow

### Deploy from UI

```
1. Design → Open Package → Open iFlow
2. Save (Ctrl+S) — saves draft
3. Deploy → Deploy (Ctrl+Shift+D)
   → iFlow starts in Worker node
   → Monitor → Manage Integration Content → Status: Started

" Versioning:
4. Save as Version → enter version number + description
   " Creates snapshot; revert via Version History
```

### Deploy via API (CI/CD)

```bash
# CPI Management API (OData V2)
# Authenticate: OAuth2 client credentials for CPI API

# Upload integration package
curl -X POST \
  "https://<tenant>.it-cpi001.cfapps.eu10.hana.ondemand.com/api/v1/IntegrationPackages" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"Id":"EWMLogisticsIntegration","Name":"EWM Logistics Integration","Version":"1.0.0"}'

# Upload iFlow artifact
curl -X POST \
  "https://<tenant>/api/v1/IntegrationDesigntimeArtifacts" \
  -H "Authorization: Bearer ${TOKEN}" \
  -F "artifact=@delivery-confirmation.zip;type=application/zip" \
  -F 'artifactMetaData={"Id":"delivery-confirmation","Name":"Delivery Confirmation","PackageId":"EWMLogisticsIntegration","ArtifactContent":"..."}'

# Deploy (start runtime)
curl -X POST \
  "https://<tenant>/api/v1/DeployIntegrationDesigntimeArtifact?Id='delivery-confirmation'&Version='active'" \
  -H "Authorization: Bearer ${TOKEN}"

# Check deployment status
curl "https://<tenant>/api/v1/IntegrationRuntimeArtifacts('delivery-confirmation')" \
  -H "Authorization: Bearer ${TOKEN}"
```

## Transport Between Environments (Dev → QA → Prod)

### Option 1: CTS+ (SAP Transport Management)

```
Design → Package → Transport to CTS+
  → Creates transport request in SAP TMS
  → Ops team imports via CTS+ console to target system
```

### Option 2: Export/Import ZIP

```
Design → Package → Export → .zip file
  → Import to target tenant:
Monitor → Manage Integration Content → Import
  OR: API upload (see above)
```

### Option 3: SAP Cloud Transport Management Service

```
Design → Package → Transport
  → Pushes to Cloud TMS (BTP service)
  → TMS pipeline: DEV → QA → PROD
  → Approval gates per environment
```

## Externalized Parameters (Environment-Specific Config)

Allows the same iFlow to run in DEV/QA/PROD with different endpoint URLs:

```
iFlow Properties → Externalize:
  Parameter Name: BackendURL
  Default Value:  https://dev-s4.ondemand.com
  
" In adapter configuration:
  Address: {{BackendURL}}    " Uses externalized parameter

" Override per environment in Monitor → Manage Integration Content:
  iFlow → Configure → BackendURL → https://prod-s4.ondemand.com → Deploy
```

## Alert Configuration

```
Monitor → Manage Alerts → Create Alert Rule
  Name:           EWM-Delivery-iFlow-Alert
  Integration Flow: Delivery-Confirmation-iFlow
  Status Filter:    Failed
  Time Range:       1 hour
  Threshold:        3 occurrences
  Notification:
    Email:     ops@mycompany.com
    Webhook:   https://hooks.teams.microsoft.com/...   (Teams / Slack)
```

## JMS Queue Monitoring

```
Monitor → Manage Message Queues
  Shows:
    Queue Name:      DeliveryConfirmationQueue
    Messages:        15 pending
    DLQ Messages:    2 failed (in dead letter queue)
    
  Actions:
    Retry DLQ:    Resubmit failed messages from dead letter queue
    Delete:       Discard messages in DLQ
    Download:     Export message payloads for analysis
```

## Performance Tuning

```
iFlow Properties → Runtime Configuration:
  Number of Parallel Processes:  2   (worker threads for this iFlow)
  " Increase for high-throughput flows; decrease to avoid overwhelming backend

" Timer-based flows: increase interval if backend throttles
  Timer: every 1 min → every 5 min

" JMS-based flows: tune Max. Retries and Retry Interval
  Max Retries: 3, Interval: 60s → prevents rapid hammering on backend failure

" Message size: use streaming for large payloads
  Streaming: true (in adapter settings) for files > 10MB
```

## Tenant Limits (Common Constraints)

| Resource | Limit |
|---|---|
| Max message payload | 40 MB (standard), 100 MB (premium) |
| JMS queue size | 100 MB per queue |
| Max iFlows (started) | 200 per tenant (standard) |
| Message log retention | 30 days |
| Trace data retention | 10 minutes |
| Concurrent requests | Based on service plan |
