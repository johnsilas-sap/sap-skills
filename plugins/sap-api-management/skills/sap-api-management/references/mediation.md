# Mediation Policies Reference

## AssignMessage — Set Headers, Body, Query Params

The most-used mediation policy. Modifies request or response.

### Add/Override Headers

```xml
<!-- Add SAP basic auth header to backend request -->
<AssignMessage name="AddSAPAuthHeader">
  <Set>
    <Headers>
      <Header name="Authorization">{private.sap_basic_auth}</Header>
      <Header name="sap-client">100</Header>
    </Headers>
  </Set>
  <IgnoreUnresolvedVariables>false</IgnoreUnresolvedVariables>
  <AssignTo createNew="false" transport="http" type="request"/>
</AssignMessage>
```

### Remove Headers (strip internal headers from response)

```xml
<AssignMessage name="RemoveInternalHeaders">
  <Remove>
    <Headers>
      <Header name="SAP-Requestcontext"/>
      <Header name="sap-metadata-last-modified"/>
      <Header name="x-vcap-request-id"/>
    </Headers>
  </Remove>
  <AssignTo createNew="false" transport="http" type="response"/>
</AssignMessage>
```

### Add CORS Headers

```xml
<AssignMessage name="AddCORSHeaders">
  <Set>
    <Headers>
      <Header name="Access-Control-Allow-Origin">*</Header>
      <Header name="Access-Control-Allow-Methods">GET, POST, PATCH, DELETE, OPTIONS</Header>
      <Header name="Access-Control-Allow-Headers">Authorization, Content-Type, x-api-key</Header>
    </Headers>
  </Set>
  <AssignTo createNew="false" type="response"/>
</AssignMessage>
```

### Rewrite Query Parameters

```xml
<!-- Strip internal query params, add SAP-specific ones -->
<AssignMessage name="RewriteQueryParams">
  <Remove>
    <QueryParams>
      <QueryParam name="apikey"/>    <!-- Don't forward to backend -->
    </QueryParams>
  </Remove>
  <Set>
    <QueryParams>
      <QueryParam name="sap-language">EN</QueryParam>
    </QueryParams>
  </Set>
  <AssignTo type="request"/>
</AssignMessage>
```

### Build Custom Payload

```xml
<AssignMessage name="BuildErrorResponse">
  <Set>
    <StatusCode>400</StatusCode>
    <ReasonPhrase>Bad Request</ReasonPhrase>
    <Headers>
      <Header name="Content-Type">application/json</Header>
    </Headers>
    <Payload contentType="application/json">
      {
        "error": "invalid_request",
        "message": "Missing required field: DeliveryNo",
        "timestamp": "{system.time}"
      }
    </Payload>
  </Set>
  <AssignTo type="response"/>
</AssignMessage>
```

## ExtractVariables — Parse Request/Response

Extracts values from headers, query params, body (JSON/XML/form), and stores in flow variables.

### Extract from Header

```xml
<ExtractVariables name="ExtractClientId">
  <Header name="x-client-id">
    <Pattern ignoreCase="false">{client_id}</Pattern>
  </Header>
  <VariablePrefix>extracted</VariablePrefix>
</ExtractVariables>
<!-- sets: extracted.client_id -->
```

### Extract from Query Parameter

```xml
<ExtractVariables name="ExtractWarehouseNo">
  <QueryParam name="warehouse">
    <Pattern>{lgnum}</Pattern>
  </QueryParam>
  <VariablePrefix>req</VariablePrefix>
</ExtractVariables>
<!-- sets: req.lgnum -->
```

### Extract from JSON Body (JSONPath)

```xml
<ExtractVariables name="ExtractDeliveryNo">
  <JSONPayload>
    <Variable name="deliveryNo" type="string">
      <JSONPath>$.DeliveryNo</JSONPath>
    </Variable>
    <Variable name="quantity" type="integer">
      <JSONPath>$.Quantity</JSONPath>
    </Variable>
  </JSONPayload>
  <Source>request</Source>
  <VariablePrefix>body</VariablePrefix>
</ExtractVariables>
<!-- sets: body.deliveryNo, body.quantity -->
```

### Extract from XML Body (XPath)

```xml
<ExtractVariables name="ExtractFromSOAP">
  <XMLPayload>
    <Namespaces>
      <Namespace prefix="ns">urn:sap-com:document:sap:soap:functions:mc-style</Namespace>
    </Namespaces>
    <Variable name="poNumber" type="string">
      <XPath>/ns:CreatePO/PONumber</XPath>
    </Variable>
  </XMLPayload>
  <Source>request</Source>
</ExtractVariables>
```

## JSONToXML and XMLToJSON

```xml
<!-- Convert JSON request to XML for SOAP backend -->
<JSONToXML name="RequestJSONToXML">
  <Options>
    <NullValue>NULL</NullValue>
    <NamespaceSeparator>:</NamespaceSeparator>
  </Options>
  <Source>request</Source>
  <OutputVariable>request</OutputVariable>
</JSONToXML>

<!-- Convert XML response from backend to JSON -->
<XMLToJSON name="ResponseXMLToJSON">
  <Options>
    <NullValue>NULL</NullValue>
    <NamespaceSeparator>:</NamespaceSeparator>
    <RecognizeBoolean>true</RecognizeBoolean>
    <RecognizeNumber>true</RecognizeNumber>
  </Options>
  <Source>response</Source>
  <OutputVariable>response</OutputVariable>
</XMLToJSON>
```

## ServiceCallout — Call External Service Mid-Flow

Call another service and use its response within the flow (e.g., fetch a token, validate data):

```xml
<ServiceCallout name="FetchSAPToken">
  <Request variable="tokenRequest">
    <Set>
      <Verb>POST</Verb>
      <Path>/oauth/token</Path>
      <Headers>
        <Header name="Content-Type">application/x-www-form-urlencoded</Header>
        <Header name="Authorization">{private.sap_client_credentials_basic}</Header>
      </Headers>
      <Payload>grant_type=client_credentials</Payload>
    </Set>
  </Request>
  <Response>tokenResponse</Response>
  <HTTPTargetConnection>
    <URL>https://my-s4.authentication.eu10.hana.ondemand.com</URL>
  </HTTPTargetConnection>
</ServiceCallout>

<!-- Then extract the token from tokenResponse -->
<ExtractVariables name="ExtractSAPToken">
  <Source>tokenResponse</Source>
  <JSONPayload>
    <Variable name="sap_token" type="string">
      <JSONPath>$.access_token</JSONPath>
    </Variable>
  </JSONPayload>
</ExtractVariables>
```

## JavaScript Policy — Custom Logic

```xml
<Javascript name="ValidateDeliveryPayload" timeLimit="200">
  <ResourceURL>jsc://ValidateDelivery.js</ResourceURL>
</Javascript>
```

```javascript
// resources/jsc/ValidateDelivery.js
var body = JSON.parse(context.getVariable('request.content') || '{}');

if (!body.DeliveryNo) {
  context.setVariable('custom.error', 'DeliveryNo is required');
  context.setVariable('custom.hasError', true);
  return;
}

if (body.Quantity && body.Quantity <= 0) {
  context.setVariable('custom.error', 'Quantity must be positive');
  context.setVariable('custom.hasError', true);
  return;
}

context.setVariable('custom.hasError', false);
context.setVariable('validated.deliveryNo', body.DeliveryNo);
```

```xml
<!-- Use the variable set by JS in a condition -->
<Step>
  <Name>Return400</Name>
  <Condition>custom.hasError == true</Condition>
</Step>
```

## KeyValue Map — Secure Config/Secrets

Store credentials that policies read at runtime. Values are encrypted in the API Management store.

```
API Portal → Environments → Key Value Maps → Create
  Name:        sap-backend-credentials
  Encrypted:   ✓
  Entries:
    sap_user:     svc_api_user
    sap_password: (encrypted value)
    sap_client:   100
```

```xml
<!-- Read KVM in a policy using KeyValueMapOperations -->
<KeyValueMapOperations name="GetSAPCredentials" mapIdentifier="sap-backend-credentials">
  <Scope>environment</Scope>
  <Get assignTo="private.sap_user">
    <Key><Parameter>sap_user</Parameter></Key>
  </Get>
  <Get assignTo="private.sap_password">
    <Key><Parameter>sap_password</Parameter></Key>
  </Get>
</KeyValueMapOperations>
```

`private.*` prefix prevents the variable from leaking into response data or logs.
