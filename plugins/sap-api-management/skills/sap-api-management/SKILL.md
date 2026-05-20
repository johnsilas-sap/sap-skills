---
name: sap-api-management
description: |
  SAP API Management skill (part of SAP Integration Suite on BTP). Use when creating
  or configuring API proxies, applying policies (Verify API Key, OAuth 2.0, Quota,
  Spike Arrest, Assign Message, Extract Variables, Service Callout, Response Cache),
  designing API products, managing developer portal subscriptions, or exposing SAP
  S/4HANA, EWM, and TM OData services via a managed API gateway. Covers proxy
  configuration, policy XML syntax, pre/post flows, conditional flows, OAuth client
  credentials and authorization code flows, on-premise backend connectivity via
  Cloud Connector, and API analytics.
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-20"
  sources:
    - "https://help.sap.com/docs/sap-api-management"
    - "https://help.sap.com/docs/integration-suite"
---

# SAP API Management Skill

## Related Skills

- **sap-btp**: BTP platform — subaccount, service instances, Cloud Connector, destinations
- **sap-ewm**: EWM OData services exposed via API Management
- **sap-idoc**: IDoc-to-REST mediation patterns via API Management + Cloud Integration

## Architecture Overview

```
API Consumer (mobile app / partner / microservice)
  ↓ HTTPS request
API Proxy (SAP API Management)
  ↓ applies policies: auth, quota, transformation
Target Endpoint (SAP backend)
  ← OData V2/V4, REST, SOAP
SAP S/4HANA / EWM / TM  (on-premise via Cloud Connector or cloud)
```

API Management runs on BTP as part of **SAP Integration Suite**. The API Portal is the design-time; the API Gateway is the runtime.

## Core Artifacts

| Artifact | Purpose |
|---|---|
| **API Proxy** | Routes and applies policies to one backend API |
| **API Product** | Bundles one or more proxies; controls access |
| **Application** | Developer registration to consume an API product |
| **Policy** | XML rule applied in proxy flow (security, traffic, mediation) |
| **KeyValue Map (KVM)** | Secure runtime storage for secrets, config values |

## Creating an API Proxy (API Portal)

```
API Portal → APIs → Create
  API Type:       REST / OData / SOAP
  Name:           ewm-delivery-api
  Base Path:      /ewm/v1
  Target Endpoint: https://my-s4.company.com/sap/opu/odata/sap/ZEWM_DELIVERY_SRV
  
  " Or on-premise via Cloud Connector:
  Target Endpoint: http://my-ewm-system:8000/sap/opu/odata/sap/ZEWM_DELIVERY_SRV
  Virtual Host:   (Cloud Connector location ID)
```

## Proxy Flow Structure

```xml
<ProxyEndpoint name="default">
  <PreFlow>
    <Request>
      <Step><Name>VerifyAPIKey</Name></Step>
      <Step><Name>SpikeArrest</Name></Step>
    </Request>
    <Response/>
  </PreFlow>

  <Flows>
    <Flow name="GetDeliveries">
      <Condition>request.verb == "GET" AND proxy.pathsuffix MatchesPath "/DeliverySet*"</Condition>
      <Request>
        <Step><Name>CheckQuota</Name></Step>
      </Request>
      <Response>
        <Step><Name>ResponseCache</Name></Step>
      </Response>
    </Flow>
  </Flows>

  <PostFlow>
    <Response>
      <Step><Name>AddCORSHeaders</Name></Step>
    </Response>
  </PostFlow>
</ProxyEndpoint>
```

## Common Policies Summary

| Category | Policy | Purpose |
|---|---|---|
| Security | `VerifyAPIKey` | Validate API key in header/query |
| Security | `OAuthV2` | Verify Bearer token or generate tokens |
| Security | `BasicAuthentication` | Encode/decode Basic auth header |
| Traffic | `Quota` | Limit calls per developer per time period |
| Traffic | `SpikeArrest` | Smooth burst traffic (rate per second/minute) |
| Traffic | `ResponseCache` | Cache responses to reduce backend load |
| Mediation | `AssignMessage` | Set/remove headers, query params, body |
| Mediation | `ExtractVariables` | Parse values from request/response into variables |
| Mediation | `JSONToXML` / `XMLToJSON` | Convert payload format |
| Mediation | `ServiceCallout` | Call an external service mid-flow |
| Mediation | `JavaScript` | Custom logic in JavaScript |

## Key Transactions / Tools

| Tool | Purpose |
|---|---|
| API Portal (BTP) | Design and manage API proxies, products, KVMs |
| Developer Portal (BTP) | Publish APIs; developers subscribe via apps |
| API Business Hub Enterprise | SAP's hosted developer portal (alternative to self-hosted) |
| Integration Suite | Umbrella service hosting API Management on BTP |
