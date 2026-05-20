# SAP API Management Skill

Skill for SAP API Management (Integration Suite on BTP) — API proxy design, policy configuration, OAuth 2.0, quota/rate limiting, mediation, developer portal, and SAP backend connectivity.

## Skill Overview

- **API Proxy**: Base path, target endpoint, virtual host, Cloud Connector routing
- **Flow Structure**: PreFlow, conditional flows, PostFlow, flow conditions
- **Security Policies**: VerifyAPIKey, OAuthV2, BasicAuthentication, IP restriction
- **Traffic Policies**: Quota, SpikeArrest, ResponseCache
- **Mediation Policies**: AssignMessage, ExtractVariables, JSONToXML, ServiceCallout, JavaScript
- **KeyValue Maps**: Secure credential and config storage at proxy runtime
- **API Products**: Bundling proxies, access control, quota settings per product
- **Developer Portal**: App registration, subscriptions, API documentation publishing
- **SAP Backends**: EWM/TM OData on-premise via Cloud Connector, S/4HANA cloud
- **Analytics**: API usage, error rates, latency monitoring

## Auto-Trigger Keywords

### Core API Management
- SAP API Management, API Management, APIM
- API Portal, API Gateway, API proxy
- Integration Suite API, BTP API Management
- API policy, proxy flow, target endpoint

### Proxy and Flow
- API proxy, proxy endpoint, target endpoint
- PreFlow, PostFlow, conditional flow, flow condition
- base path, virtual host, proxy path suffix
- request flow, response flow, fault rule

### Security
- VerifyAPIKey, API key, API key verification
- OAuthV2, OAuth 2.0, Bearer token, access token
- client credentials flow, authorization code flow
- BasicAuthentication, basic auth encode
- IP allowlist, IP restriction policy
- mTLS, mutual TLS, client certificate

### Traffic Management
- Quota policy, rate limit, API quota
- SpikeArrest, spike arrest, burst limit
- ResponseCache, response cache, cache policy
- LookupCache, PopulateCache, InvalidateCache

### Mediation
- AssignMessage, assign message, set header
- ExtractVariables, extract variable, JSONPath
- JSONToXML, XMLToJSON, format conversion
- ServiceCallout, service callout, mid-flow call
- RaiseFault, raise fault, custom error
- JavaScript policy, Python policy, custom script

### KeyValue Maps
- KeyValueMap, KVM, key value map
- EncryptedKeyValueMap, secure storage
- Get KVM entry, set KVM entry

### Products and Portal
- API product, API bundle, product plan
- developer portal, Developer Hub, API catalog
- app registration, API subscription, developer app
- client ID, client secret, application key

### SAP Backend Connectivity
- OData API Management, EWM OData proxy
- Cloud Connector API, on-premise API
- S/4HANA API, destination service
- SAP API Business Hub, SAP Graph

### Analytics
- API analytics, API usage, API monitoring
- error rate, latency, throughput
- API dashboard, custom report
