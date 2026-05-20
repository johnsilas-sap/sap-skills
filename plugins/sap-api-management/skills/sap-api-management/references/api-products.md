# API Products, Developer Portal, and Analytics Reference

## API Products

An API Product bundles one or more proxies and controls access — quota, scopes, environments.

### Creating an API Product

```
API Portal → Products → Create
  Name:         EWM Logistics API - Standard
  Display Name: EWM Logistics API (Standard Tier)
  Description:  Read access to deliveries and warehouse tasks
  
  APIs:         ewm-delivery-api (all resources)
                ewm-task-api (/WarehouseTaskSet)
  
  Environments: prod
  
  Access:       Private (requires approval) or Public
  
  Quota:        1000 calls / 1 month / app
  Scopes:       delivery.read, task.read
```

### Multiple Tiers (Free / Standard / Premium)

```
Product: EWM Logistics - Free
  Quota: 100 calls / day
  Scopes: delivery.read (GET only)

Product: EWM Logistics - Standard
  Quota: 10000 calls / month
  Scopes: delivery.read, delivery.write, task.read

Product: EWM Logistics - Premium
  Quota: unlimited
  Scopes: delivery.read, delivery.write, task.read, task.write, gr.post
```

## Developer Portal

### Publishing an API

```
API Portal → APIs → Publish to Developer Portal
  " Makes the API visible with Swagger UI / OpenAPI docs
  " Developers can browse, test, and subscribe

" Required before publishing:
  - API must have an OpenAPI spec (auto-generated for OData)
  - At least one Product must include the API
```

### App Registration (Developer Side)

```
Developer Portal → My Apps → Create App
  App Name:     My Warehouse Integration
  Description:  Integration to EWM for goods receipt posting
  Products:     EWM Logistics API - Standard  (select and subscribe)

  → System generates:
    Client ID:     b3f5a8c2-...
    Client Secret: (shown once — store immediately)
    API Key:       zX9kP2mN...   (alternative to OAuth)
```

### Approval Flow

```
Product Access = Private:
  Developer subscribes → request goes to API Admin for approval
  API Portal → Apps → Pending Approvals → Approve / Reject

Product Access = Public (auto-approved):
  Developer subscribes → credentials issued immediately
```

## Developer Portal Customization

```
API Portal → Manage → Developer Portal Settings
  Logo:         Upload company logo
  Theme:        Custom CSS
  Custom pages: Add terms of use, getting started guide
  Domain:       developer.mycompany.com (custom domain via CNAME)
```

## Monitoring and Analytics

### API Portal Analytics Dashboard

```
API Portal → Analytics
  Traffic:      Total calls over time, per proxy, per product
  Errors:       Error rate by HTTP status code, by proxy
  Latency:      P50, P95, P99 response time
  Top APIs:     Most-called endpoints
  Developers:   Top consuming apps
  
Time range: Last hour / day / week / month / custom
Filters: Environment, Proxy, Product, Developer App
```

### Custom Reports

```
API Portal → Analytics → Custom Reports → Create
  Dimensions:   API Proxy, Developer App, Resource Path, Status Code
  Metrics:      Request Count, Error Count, Response Time (avg, max)
  Filter:       Status Code >= 400
  Export:       CSV / PDF
```

### Key Metrics to Monitor

| Metric | Warning Threshold | Alert Threshold |
|---|---|---|
| Error rate | > 1% | > 5% |
| P95 latency | > 2s | > 5s |
| Quota usage | > 80% | > 95% |
| 5xx errors | Any | Sustained for 5 min |

## Monetization (Optional)

```
API Portal → Monetize → Rate Plans
  Plan:          Pay-per-call
  Rate:          $0.001 per call above free tier
  
  Or: Flat rate subscription
  Rate:          $99/month for Standard tier
```

## API Product Lifecycle

```
Draft → Published → Deprecated → Retired

Draft:       Internal testing only; not visible in developer portal
Published:   Active; developers can subscribe
Deprecated:  Still works; new subscriptions blocked; developers warned
Retired:     Traffic blocked; returns 403 to all consumers
```

```
API Portal → Products → Product → Deprecate
  Set sunset date:  2026-12-31
  Notification:     Email all subscribed developers
```

## Testing APIs from Developer Portal

```
Developer Portal → API → Try It Out
  Authorize:    (enter API key or OAuth credentials)
  Resource:     GET /DeliverySet
  Parameters:   $filter=Status eq 'Open'
  Execute:      Sends live request through the API gateway
  Response:     JSON response from SAP backend
```

## Alerting

```
API Portal → Analytics → Alerts → Create Alert
  Name:        High Error Rate
  Condition:   Error Rate > 5% for 5 consecutive minutes
  Scope:       ewm-delivery-api proxy, prod environment
  Action:      Email admin@mycompany.com
               Webhook: https://hooks.mycompany.com/api-alert
```
