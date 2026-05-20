# SAP Backend API Connectivity Reference

## On-Premise SAP via Cloud Connector

```
Internet consumer
  → API Gateway (BTP API Management)
  → Cloud Connector (on-premise agent on customer network)
  → SAP EWM / S/4HANA (internal)
```

### Cloud Connector Setup

```
1. Install Cloud Connector on a VM inside the corporate network
   (download from SAP Tools: cloudconnector-installer-2.x.x.msi)

2. Logon to Cloud Connector Admin UI:
   https://localhost:8443/

3. Connect to BTP subaccount:
   Subaccount:     api.hana.ondemand.com (CF region endpoint)
   Subaccount ID:  <your BTP subaccount ID>

4. Add System Mapping:
   Virtual Host:     ewm-internal          (used in API proxy URL)
   Virtual Port:     8000
   Internal Host:    ewm-prod.corp.local   (real SAP host)
   Internal Port:    8000
   Protocol:         HTTP

5. Add Resources (paths to expose):
   URL Path:         /sap/opu/odata/sap/ZEWM_DELIVERY_SRV/
   Access Policy:    Path and all sub-paths
```

### BTP Destination for Cloud Connector

```
BTP Cockpit → Connectivity → Destinations → New
  Name:          EWM_BACKEND
  Type:          HTTP
  URL:           http://ewm-internal:8000
  Proxy Type:    OnPremise
  Authentication: BasicAuthentication
  User:          svc_api_user
  Password:      (encrypted)
  
  Additional Properties:
    sap-client:  100
    HTML5.DynamicDestination: true
```

### API Proxy TargetEndpoint for On-Premise

```xml
<TargetEndpoint name="default">
  <HTTPTargetConnection>
    <!-- Reference the BTP destination by name -->
    <URL>http://EWM_BACKEND/sap/opu/odata/sap/ZEWM_DELIVERY_SRV</URL>
    <!-- API Management resolves EWM_BACKEND via Cloud Connector -->
  </HTTPTargetConnection>
</TargetEndpoint>
```

## SAP S/4HANA Cloud (Public Cloud)

S/4HANA Public Cloud exposes APIs via SAP API Business Hub. Use OAuth 2.0 (SAML 2.0 Bearer or client credentials).

### Communication Arrangement Setup

```
S/4HANA → Communication Management → Communication Arrangements
  Communication Scenario:  SAP_COM_0207   (Delivery Management)
  Communication System:    MY_API_MGMT_SYSTEM
  Authentication:          OAuth 2.0

  → Generates:
    Token URL:       https://my-s4.authentication.eu10.hana.ondemand.com/oauth/token
    Client ID:       sb-1a2b3c4d-...
    Client Secret:   (copy and store securely)
    API URL:         https://my-s4.s4hana.ondemand.com/sap/opu/odata/sap/API_DELIVERY_SRV
```

### Proxy Flow: Fetch S/4 Token and Forward Request

```xml
<!-- 1. Get cached token or fetch new one -->
<Step><Name>GetCachedS4Token</Name></Step>
<Step>
  <Name>FetchS4Token</Name>
  <Condition>lookupcache.GetCachedS4Token.cachehit == false</Condition>
</Step>
<Step>
  <Name>ExtractS4Token</Name>
  <Condition>lookupcache.GetCachedS4Token.cachehit == false</Condition>
</Step>
<Step>
  <Name>CacheS4Token</Name>
  <Condition>lookupcache.GetCachedS4Token.cachehit == false</Condition>
</Step>

<!-- 2. Add Bearer token to backend request -->
<Step><Name>AddBearerToTarget</Name></Step>
```

```xml
<!-- FetchS4Token: ServiceCallout to S/4 token endpoint -->
<ServiceCallout name="FetchS4Token">
  <Request variable="s4TokenReq">
    <Set>
      <Verb>POST</Verb>
      <Headers>
        <Header name="Content-Type">application/x-www-form-urlencoded</Header>
        <Header name="Authorization">{private.s4_client_credentials_b64}</Header>
      </Headers>
      <Payload>grant_type=client_credentials</Payload>
    </Set>
  </Request>
  <Response>s4TokenResp</Response>
  <HTTPTargetConnection>
    <URL>https://my-s4.authentication.eu10.hana.ondemand.com/oauth/token</URL>
  </HTTPTargetConnection>
</ServiceCallout>

<!-- ExtractS4Token -->
<ExtractVariables name="ExtractS4Token">
  <Source>s4TokenResp</Source>
  <JSONPayload>
    <Variable name="token" type="string">
      <JSONPath>$.access_token</JSONPath>
    </Variable>
  </JSONPayload>
  <VariablePrefix>s4</VariablePrefix>
</ExtractVariables>

<!-- AddBearerToTarget -->
<AssignMessage name="AddBearerToTarget">
  <Set>
    <Headers>
      <Header name="Authorization">Bearer {s4.token}</Header>
      <Header name="sap-client">100</Header>
    </Headers>
  </Set>
  <AssignTo type="request"/>
</AssignMessage>
```

## Common SAP OData Endpoints to Proxy

### EWM OData Services

| Service | Path | Description |
|---|---|---|
| `ZEWM_DELIVERY_SRV` | `/sap/opu/odata/sap/ZEWM_DELIVERY_SRV/` | Delivery CRUD |
| `ZEWM_TASK_SRV` | `/sap/opu/odata/sap/ZEWM_TASK_SRV/` | Warehouse tasks |
| `ZEWM_LABEL_SRV` | `/sap/opu/odata/sap/ZEWM_LABEL_SRV/` | Label data |
| `/SCWM/HU_SRV` | `/sap/opu/odata/sap//SCWM/HU_SRV/` | Handling units |

### S/4HANA Standard APIs (API Business Hub)

| API | CDS / Service | Description |
|---|---|---|
| Delivery API | `API_OUTBOUND_DELIVERY_SRV` | SD outbound delivery |
| Purchase Order | `API_PURCHASEORDER_PROCESS_SRV` | MM purchase orders |
| Sales Order | `API_SALES_ORDER_SRV` | SD sales orders |
| Inbound Delivery | `API_INBOUND_DELIVERY_SRV` | Goods receipt posting |

## Path Rewriting (Strip Base Path)

If your proxy base path is `/ewm/v1` but the backend expects `/sap/opu/odata/sap/ZEWM_DELIVERY_SRV`:

```xml
<!-- AssignMessage to rewrite path -->
<AssignMessage name="RewriteTargetPath">
  <Set>
    <Path>/sap/opu/odata/sap/ZEWM_DELIVERY_SRV{proxy.pathsuffix}</Path>
  </Set>
  <AssignTo type="request"/>
</AssignMessage>
```

Or configure in TargetEndpoint:

```xml
<HTTPTargetConnection>
  <URL>https://my-s4.ondemand.com/sap/opu/odata/sap/ZEWM_DELIVERY_SRV</URL>
  <!-- proxy.pathsuffix is automatically appended -->
</HTTPTargetConnection>
```

## CSRF Token Handling (SAP OData V2)

SAP OData V2 services require a CSRF token for write operations (POST/PATCH/DELETE):

```xml
<!-- Step 1: Fetch CSRF token with HEAD/GET -->
<ServiceCallout name="FetchCSRFToken">
  <Request variable="csrfReq">
    <Set>
      <Verb>GET</Verb>
      <Headers>
        <Header name="x-csrf-token">Fetch</Header>
        <Header name="Authorization">{private.sap_basic_auth}</Header>
      </Headers>
      <Path>/sap/opu/odata/sap/ZEWM_DELIVERY_SRV/</Path>
    </Set>
  </Request>
  <Response>csrfResp</Response>
  <HTTPTargetConnection>
    <URL>https://my-ewm.ondemand.com</URL>
  </HTTPTargetConnection>
</ServiceCallout>

<!-- Step 2: Extract token from response header -->
<ExtractVariables name="ExtractCSRFToken">
  <Source>csrfResp</Source>
  <Header name="x-csrf-token">
    <Pattern>{csrf_token}</Pattern>
  </Header>
</ExtractVariables>

<!-- Step 3: Add token to write request -->
<AssignMessage name="AddCSRFToken">
  <Set>
    <Headers>
      <Header name="x-csrf-token">{csrf_token}</Header>
    </Headers>
  </Set>
  <AssignTo type="request"/>
</AssignMessage>
```

Apply the CSRF flow only for write operations:

```xml
<Flow name="WriteOperations">
  <Condition>request.verb == "POST" OR request.verb == "PATCH" OR request.verb == "DELETE"</Condition>
  <Request>
    <Step><Name>FetchCSRFToken</Name></Step>
    <Step><Name>ExtractCSRFToken</Name></Step>
    <Step><Name>AddCSRFToken</Name></Step>
    <Step><Name>AddSAPAuthHeader</Name></Step>
  </Request>
</Flow>
```
