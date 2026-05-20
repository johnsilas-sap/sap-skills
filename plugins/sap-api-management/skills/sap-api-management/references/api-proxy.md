# API Proxy Reference

## What an API Proxy Does

An API proxy sits between consumers and your backend. It:
- Provides a stable public URL independent of the backend
- Applies policies (auth, rate limiting, transformation)
- Hides backend implementation details
- Enables versioning without breaking consumers

## Creating a Proxy in API Portal

### REST / OData Proxy

```
API Portal → APIs → Create → REST
  Name:            ewm-delivery-api
  Title:           EWM Delivery API
  Base Path:       /ewm/v1
  API Version:     v1
  Target Endpoint: https://my-s4.ondemand.com/sap/opu/odata/sap/ZEWM_DELIVERY_SRV

  " Or: import from OpenAPI spec (.yaml/.json)
  " Or: discover from SAP API Business Hub
```

### OData-Specific Proxy

```
API Portal → APIs → Create → OData
  Metadata URL:    https://my-s4.ondemand.com/sap/opu/odata/sap/ZEWM_DELIVERY_SRV/$metadata
  " Automatically discovers entity sets, generates API documentation
```

### On-Premise via Cloud Connector

```
API Portal → APIs → Create → REST
  Target Endpoint: http://ewm-internal-host:8000/sap/opu/odata/sap/ZEWM_DELIVERY_SRV
  Target Server:   (create a Target Server pointing to CC location)

" OR use a BTP Destination:
  Target Endpoint: (use destination name in AssignMessage policy or Cloud Connector binding)
```

## Proxy Bundle Structure (downloaded ZIP)

```
ewm-delivery-api/
  apiproxy/
    ewm-delivery-api.xml          — Proxy manifest
    proxies/
      default.xml                 — ProxyEndpoint definition
    targets/
      default.xml                 — TargetEndpoint definition
    policies/
      VerifyAPIKey.xml
      SpikeArrest.xml
      AssignAuthHeader.xml
    resources/
      jsc/
        ValidateRequest.js        — JavaScript resources
```

## ProxyEndpoint (proxies/default.xml)

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ProxyEndpoint name="default">

  <PreFlow name="PreFlow">
    <Request>
      <Step><Name>VerifyAPIKey</Name></Step>
      <Step><Name>SpikeArrest</Name></Step>
    </Request>
    <Response/>
  </PreFlow>

  <Flows>
    <!-- GET /DeliverySet -->
    <Flow name="ListDeliveries">
      <Condition>request.verb == "GET" AND proxy.pathsuffix MatchesPath "/DeliverySet"</Condition>
      <Request>
        <Step><Name>CheckReadQuota</Name></Step>
      </Request>
      <Response>
        <Step><Name>CacheDeliveries</Name></Step>
        <Step><Name>RemoveInternalHeaders</Name></Step>
      </Response>
    </Flow>

    <!-- POST /GoodsReceiptSet -->
    <Flow name="PostGoodsReceipt">
      <Condition>request.verb == "POST" AND proxy.pathsuffix MatchesPath "/GoodsReceiptSet"</Condition>
      <Request>
        <Step><Name>ValidateGRPayload</Name></Step>
        <Step><Name>AddSAPAuthHeader</Name></Step>
      </Request>
      <Response/>
    </Flow>
  </Flows>

  <PostFlow name="PostFlow">
    <Request/>
    <Response>
      <Step><Name>AddCORSHeaders</Name></Step>
    </Response>
  </PostFlow>

  <FaultRules>
    <FaultRule name="AuthFailure">
      <Condition>fault.name == "InvalidApiKey" OR fault.name == "InvalidAccessToken"</Condition>
      <Step><Name>Return401</Name></Step>
    </FaultRule>
    <FaultRule name="QuotaExceeded">
      <Condition>fault.name == "QuotaViolation"</Condition>
      <Step><Name>Return429</Name></Step>
    </FaultRule>
  </FaultRules>

  <DefaultFaultRule name="DefaultFaultRule">
    <Step><Name>Return500</Name></Step>
  </DefaultFaultRule>

  <HTTPProxyConnection>
    <BasePath>/ewm/v1</BasePath>
    <VirtualHost>secure</VirtualHost>
  </HTTPProxyConnection>

  <RouteRule name="default">
    <TargetEndpoint>default</TargetEndpoint>
  </RouteRule>

</ProxyEndpoint>
```

## TargetEndpoint (targets/default.xml)

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<TargetEndpoint name="default">

  <PreFlow name="PreFlow">
    <Request>
      <Step><Name>AddSAPAuthHeader</Name></Step>
    </Request>
    <Response/>
  </PreFlow>

  <HTTPTargetConnection>
    <URL>https://my-s4.ondemand.com/sap/opu/odata/sap/ZEWM_DELIVERY_SRV</URL>
    <SSLInfo>
      <Enabled>true</Enabled>
    </SSLInfo>
    <Properties>
      <Property name="connect.timeout.millis">10000</Property>
      <Property name="io.timeout.millis">60000</Property>
    </Properties>
  </HTTPTargetConnection>

</TargetEndpoint>
```

## Flow Conditions

```xml
<!-- Match HTTP method -->
<Condition>request.verb == "GET"</Condition>
<Condition>request.verb == "POST" OR request.verb == "PUT"</Condition>

<!-- Match path -->
<Condition>proxy.pathsuffix MatchesPath "/DeliverySet/**"</Condition>
<Condition>proxy.pathsuffix Matches "/DeliverySet*"</Condition>

<!-- Match header -->
<Condition>request.header.Content-Type == "application/json"</Condition>

<!-- Combine conditions -->
<Condition>request.verb == "GET" AND proxy.pathsuffix MatchesPath "/DeliverySet*"</Condition>

<!-- Check variable -->
<Condition>apikey.approved == "true"</Condition>
```

## Flow Variables Reference

| Variable | Value |
|---|---|
| `request.verb` | HTTP method: GET, POST, PATCH, DELETE |
| `proxy.pathsuffix` | Path after base path: `/DeliverySet('DLV001')` |
| `request.header.{name}` | Request header value |
| `request.queryparam.{name}` | Query parameter value |
| `response.status.code` | HTTP response status |
| `response.header.{name}` | Response header value |
| `fault.name` | Fault name when in FaultRule |
| `client.ip` | Consumer IP address |
| `system.time` | Current timestamp |
| `apikey.approved` | `"true"` if API key is valid |
| `developer.app.name` | Name of the registered app |

## API Versioning Strategy

```
Version in base path (recommended):
  /ewm/v1/DeliverySet
  /ewm/v2/DeliverySet

" Create separate proxy per version
" Route v1 → legacy backend
" Route v2 → new backend

" Or: version in header
  API-Version: 2024-01-01
  → Use ExtractVariables to read header, conditional flow to route
```

## Proxy Revision and Deployment

```
API Portal → API → Revisions → Create Revision
  " Revisions allow rollback without downtime

API Portal → API → Deploy → Select Environment
  Environment:  prod / dev / test
  Revision:     3   (deploy specific revision)

" Roll back:
API Portal → API → Deploy → Select older revision → Deploy
```
