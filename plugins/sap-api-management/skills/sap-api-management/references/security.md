# Security Policies Reference

## Verify API Key

Validates that the consumer has provided a registered API key.

```xml
<!-- policies/VerifyAPIKey.xml -->
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<VerifyAPIKey async="false" continueOnError="false" enabled="true" name="VerifyAPIKey">
  <APIKey ref="request.header.x-api-key"/>
  <!-- alternatives:
  <APIKey ref="request.queryparam.apikey"/>
  <APIKey ref="request.header.Authorization"/>
  -->
</VerifyAPIKey>
```

After a successful `VerifyAPIKey`, these variables are populated:

| Variable | Value |
|---|---|
| `apikey.approved` | `"true"` |
| `developer.app.name` | Name of the registered app |
| `developer.email` | Developer's email |
| `apiproduct.name` | API product the key belongs to |

## OAuth 2.0 — Verify Token

Validates a Bearer token on every request:

```xml
<!-- policies/VerifyOAuthToken.xml -->
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<OAuthV2 async="false" continueOnError="false" enabled="true" name="VerifyOAuthToken">
  <Operation>VerifyAccessToken</Operation>
  <!-- Token read from Authorization: Bearer <token> header by default -->
</OAuthV2>
```

After successful verify:

| Variable | Value |
|---|---|
| `oauthv2.VerifyOAuthToken.client_id` | Client ID |
| `oauthv2.VerifyOAuthToken.scope` | Space-separated granted scopes |
| `oauthv2.VerifyOAuthToken.expires_in` | Token TTL remaining |

## OAuth 2.0 — Generate Token (Client Credentials)

Expose a `/token` endpoint on your proxy that issues tokens:

```xml
<!-- policies/GenerateToken.xml -->
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<OAuthV2 async="false" continueOnError="false" enabled="true" name="GenerateToken">
  <Operation>GenerateAccessToken</Operation>
  <ExpiresIn>3600000</ExpiresIn>           <!-- 1 hour in ms -->
  <SupportedGrantTypes>
    <GrantType>client_credentials</GrantType>
  </SupportedGrantTypes>
  <GenerateResponse enabled="true"/>
</OAuthV2>
```

```
Token request:
POST /oauth/token
Authorization: Basic base64(client_id:client_secret)
Content-Type: application/x-www-form-urlencoded
Body: grant_type=client_credentials&scope=delivery.read

Response:
{
  "access_token": "eyJhbGci...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

## OAuth 2.0 — Authorization Code Flow

```xml
<!-- Generate Authorization Code -->
<OAuthV2 name="GenerateAuthCode">
  <Operation>GenerateAuthorizationCode</Operation>
  <ExpiresIn>600000</ExpiresIn>
  <GenerateResponse enabled="true"/>
</OAuthV2>

<!-- Exchange Code for Token -->
<OAuthV2 name="ExchangeCodeForToken">
  <Operation>GenerateAccessToken</Operation>
  <ExpiresIn>3600000</ExpiresIn>
  <SupportedGrantTypes>
    <GrantType>authorization_code</GrantType>
  </SupportedGrantTypes>
  <GenerateResponse enabled="true"/>
</OAuthV2>
```

## OAuth Scope Check

```xml
<!-- Verify that the token has the required scope -->
<OAuthV2 name="VerifyScope">
  <Operation>VerifyAccessToken</Operation>
  <Scope>delivery.write</Scope>    <!-- Token must include this scope -->
</OAuthV2>
```

## Basic Authentication (Encode)

Used to convert consumer credentials into Basic auth for the SAP backend:

```xml
<!-- policies/EncodeBasicAuth.xml -->
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<BasicAuthentication async="false" continueOnError="false" enabled="true" name="EncodeBasicAuth">
  <Operation>Encode</Operation>
  <IgnoreUnresolvedVariables>false</IgnoreUnresolvedVariables>
  <User ref="private.sap_user"/>
  <Password ref="private.sap_password"/>
  <AssignTo createNew="false">request.header.Authorization</AssignTo>
</BasicAuthentication>
```

`private.sap_user` and `private.sap_password` come from a KeyValue Map (see mediation reference).

## Basic Authentication (Decode — extract incoming credentials)

```xml
<BasicAuthentication name="DecodeIncomingBasic">
  <Operation>Decode</Operation>
  <IgnoreUnresolvedVariables>false</IgnoreUnresolvedVariables>
  <User ref="extracted.username"/>
  <Password ref="extracted.password"/>
  <Source>request.header.Authorization</Source>
</BasicAuthentication>
```

## IP Restriction

```xml
<!-- policies/IPAllowlist.xml -->
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<AccessControl async="false" continueOnError="false" enabled="true" name="IPAllowlist">
  <IPRules noRuleMatchAction="DENY">
    <MatchRule action="ALLOW">
      <SourceAddress mask="32">203.0.113.50</SourceAddress>   <!-- specific IP -->
    </MatchRule>
    <MatchRule action="ALLOW">
      <SourceAddress mask="24">10.0.1.0</SourceAddress>        <!-- /24 subnet -->
    </MatchRule>
  </IPRules>
</AccessControl>
```

## JSON Threat Protection

Prevents malicious JSON payloads (deeply nested, oversized):

```xml
<JSONThreatProtection name="JSONThreat">
  <ArrayElementCount>100</ArrayElementCount>
  <ContainerDepth>10</ContainerDepth>
  <ObjectEntryCount>50</ObjectEntryCount>
  <ObjectEntryNameLength>64</ObjectEntryNameLength>
  <StringValueLength>500</StringValueLength>
  <Source>request</Source>
</JSONThreatProtection>
```

## XML Threat Protection

```xml
<XMLThreatProtection name="XMLThreat">
  <NameLimits>
    <Element>64</Element>
    <Attribute>64</Attribute>
  </NameLimits>
  <StructureLimits>
    <NodeDepth>10</NodeDepth>
    <AttributeCountPerElement>5</AttributeCountPerElement>
    <ChildCount>100</ChildCount>
  </StructureLimits>
  <ValueLimits>
    <Text>500</Text>
    <Attribute>500</Attribute>
  </ValueLimits>
</XMLThreatProtection>
```

## RaiseFault — Custom Error Responses

```xml
<!-- Return 401 for auth failures -->
<RaiseFault name="Return401">
  <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
  <FaultResponse>
    <Set>
      <StatusCode>401</StatusCode>
      <ReasonPhrase>Unauthorized</ReasonPhrase>
      <Headers>
        <Header name="Content-Type">application/json</Header>
        <Header name="WWW-Authenticate">Bearer realm="ewm-api"</Header>
      </Headers>
      <Payload contentType="application/json">
        {"error":"invalid_token","error_description":"Access token is invalid or expired"}
      </Payload>
    </Set>
  </FaultResponse>
</RaiseFault>

<!-- Return 429 for rate limit exceeded -->
<RaiseFault name="Return429">
  <FaultResponse>
    <Set>
      <StatusCode>429</StatusCode>
      <ReasonPhrase>Too Many Requests</ReasonPhrase>
      <Payload contentType="application/json">
        {"error":"quota_exceeded","message":"Rate limit exceeded. Retry after {ratelimit.CheckQuota.expiry.time}"}
      </Payload>
    </Set>
  </FaultResponse>
</RaiseFault>
```
