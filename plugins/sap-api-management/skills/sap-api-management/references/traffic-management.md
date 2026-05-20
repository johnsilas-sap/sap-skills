# Traffic Management Policies Reference

## Spike Arrest — Burst Protection

Smooths bursts by enforcing a maximum rate per second or minute. Applied globally per proxy, not per developer.

```xml
<!-- policies/SpikeArrest.xml -->
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<SpikeArrest async="false" continueOnError="false" enabled="true" name="SpikeArrest">
  <Rate>30ps</Rate>    <!-- 30 requests per second max -->
  <!-- options: 30ps (per second), 500pm (per minute) -->
  <UseEffectiveCount>true</UseEffectiveCount>
</SpikeArrest>
```

When exceeded: fault `policies.ratelimit.SpikeArrestViolation` is raised → catch in FaultRule.

```xml
<!-- FaultRule for spike arrest in ProxyEndpoint -->
<FaultRule name="SpikeArrestViolation">
  <Condition>fault.name == "SpikeArrestViolation"</Condition>
  <Step><Name>Return429</Name></Step>
</FaultRule>
```

## Quota — Developer-Level Rate Limiting

Enforces call limits per developer app per time window. Works with API product quota settings.

```xml
<!-- policies/CheckQuota.xml -->
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Quota async="false" continueOnError="false" enabled="true" name="CheckQuota" type="calendar">
  <Allow count="1000" countRef="verifyapikey.VerifyAPIKey.apiproduct.developer.quota.limit"/>
  <Interval ref="verifyapikey.VerifyAPIKey.apiproduct.developer.quota.interval">1</Interval>
  <TimeUnit ref="verifyapikey.VerifyAPIKey.apiproduct.developer.quota.timeunit">month</TimeUnit>
  <Identifier ref="client_id"/>
  <Distributed>true</Distributed>
  <Synchronous>true</Synchronous>
  <StartTime>2026-01-01 00:00:00</StartTime>
</Quota>
```

`countRef`, `Interval`, and `TimeUnit` with `ref=` attributes read quota settings from the **API Product** definition, so you can configure limits per product without changing policy XML.

### Quota per API Product (API Portal)

```
API Portal → Products → My Product → Quota
  Quota limit:    1000
  Interval:       1
  Time unit:      Month
```

### Quota Variables Set After Evaluation

| Variable | Value |
|---|---|
| `ratelimit.CheckQuota.allowed.count` | Configured limit |
| `ratelimit.CheckQuota.used.count` | Calls made this period |
| `ratelimit.CheckQuota.available.count` | Remaining calls |
| `ratelimit.CheckQuota.expiry.time` | When quota resets (epoch ms) |

### Add Quota Headers to Response

```xml
<AssignMessage name="AddQuotaHeaders">
  <Set>
    <Headers>
      <Header name="X-RateLimit-Limit">{ratelimit.CheckQuota.allowed.count}</Header>
      <Header name="X-RateLimit-Remaining">{ratelimit.CheckQuota.available.count}</Header>
      <Header name="X-RateLimit-Reset">{ratelimit.CheckQuota.expiry.time}</Header>
    </Headers>
  </Set>
  <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
</AssignMessage>
```

## Response Cache

Caches backend responses to reduce load and improve latency:

```xml
<!-- policies/CacheDeliveries.xml -->
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ResponseCache async="false" continueOnError="false" enabled="true" name="CacheDeliveries">
  <CacheKey>
    <KeyFragment ref="request.uri" type="string"/>
    <KeyFragment ref="request.header.Accept" type="string"/>
  </CacheKey>
  <ExpirySettings>
    <TimeoutInSec ref="">300</TimeoutInSec>    <!-- 5 minute cache -->
  </ExpirySettings>
  <SkipCacheLookup>request.verb != "GET"</SkipCacheLookup>
  <SkipCachePopulation>response.status.code != 200</SkipCachePopulation>
</ResponseCache>
```

`ResponseCache` serves cached responses on the **Request** flow and populates the cache on the **Response** flow — attach to both:

```xml
<!-- In Flow Request step: serves cached response if available -->
<!-- In Flow Response step: populates cache with fresh response -->
<Step><Name>CacheDeliveries</Name></Step>
```

## Manual Cache (LookupCache / PopulateCache / InvalidateCache)

For more control — store arbitrary values, not full responses:

```xml
<!-- Store token in cache -->
<PopulateCache name="CacheToken">
  <CacheKey>
    <KeyFragment>sap_access_token</KeyFragment>
  </CacheKey>
  <Source>private.sap_token</Source>
  <ExpirySettings>
    <TimeoutInSec>3500</TimeoutInSec>     <!-- slightly less than token TTL -->
  </ExpirySettings>
</PopulateCache>

<!-- Retrieve cached token -->
<LookupCache name="GetCachedToken">
  <CacheKey>
    <KeyFragment>sap_access_token</KeyFragment>
  </CacheKey>
  <AssignTo>private.sap_token</AssignTo>
</LookupCache>

<!-- Invalidate when token expires -->
<InvalidateCache name="ClearToken">
  <CacheKey>
    <KeyFragment>sap_access_token</KeyFragment>
  </CacheKey>
  <PurgeChildEntries>true</PurgeChildEntries>
</InvalidateCache>
```

## Concurrency Limit

```xml
<ConcurrentRatelimit name="ConcurrencyLimit">
  <AllowConnections count="100"/>
  <Distributed>true</Distributed>
</ConcurrentRatelimit>
```

## Pattern: Token Cache for SAP Backend Auth

Fetch a SAP OAuth token once, cache it, reuse until expiry — avoids a token call on every request:

```xml
<!-- 1. Try to get token from cache -->
<Step>
  <Name>GetCachedSAPToken</Name>
</Step>

<!-- 2. Conditional: only fetch new token if cache miss -->
<Step>
  <Name>FetchSAPToken</Name>
  <Condition>private.sap_token == null</Condition>
</Step>

<!-- 3. Conditional: only cache if we just fetched -->
<Step>
  <Name>CacheSAPToken</Name>
  <Condition>private.sap_token != null AND lookupcache.GetCachedSAPToken.cachehit == false</Condition>
</Step>

<!-- 4. Add token to backend request -->
<Step>
  <Name>AddBearerTokenToTarget</Name>
</Step>
```
