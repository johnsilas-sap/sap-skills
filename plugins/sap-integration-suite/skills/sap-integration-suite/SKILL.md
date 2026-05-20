---
name: sap-integration-suite
description: |
  SAP Integration Suite skill, focused on Cloud Integration (CPI). Use when designing
  iFlows (integration flows), configuring adapters (OData, SOAP, HTTP, IDoc, JMS, SFTP,
  AS2, AMQP), writing Groovy/JavaScript scripts, building XSLT or message mapper
  transformations, configuring routing (Router, Multicast, Filter), handling errors
  (Exception Subprocess, dead letter), managing credentials and OAuth in the Security
  Material store, monitoring message processing logs, or deploying integration packages.
  Covers logistics patterns: IDoc-to-REST, OData-to-IDoc, EWM/TM event forwarding,
  carrier EDI (AS2/EDIFACT), and delivery confirmation flows.
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-20"
  sources:
    - "https://help.sap.com/docs/cloud-integration"
    - "https://help.sap.com/docs/integration-suite"
---

# SAP Integration Suite (Cloud Integration) Skill

## Related Skills

- **sap-api-management**: API Management is a separate Integration Suite component — proxies, policies, products
- **sap-idoc**: IDoc structure and ABAP side — complement to CPI IDoc adapter usage
- **sap-ewm**: EWM OData/IDoc services consumed or produced by iFlows
- **sap-btp**: BTP platform — subaccount, destinations, Cloud Connector

## Integration Suite Components

| Component | Purpose |
|---|---|
| **Cloud Integration (CPI)** | iFlow-based process integration — main dev component |
| **API Management** | API gateway — proxies, policies, products (see sap-api-management) |
| **Event Mesh** | Pub/sub messaging — AMQP/MQTT event streams |
| **Integration Advisor** | B2B/EDI message standard mapping (MA/MIG) |
| **Open Connectors** | Pre-built connectors to 200+ SaaS apps |
| **Trading Partner Management** | B2B partner profiles, AS2, agreements |

## iFlow Architecture

```
Sender System
  → Sender Channel (adapter: HTTP/IDoc/SFTP/JMS)
  → Start Event
  → [Processing Steps]
      Content Modifier / Script / Mapping / Router / Call
  → End Event
  → Receiver Channel (adapter: OData/SOAP/HTTP/IDoc/JMS)
  → Receiver System
```

## Creating an iFlow (Integration Suite UI)

```
Integration Suite → Design → Packages → Create Package
  Name:    EWM Logistics Integration
  Version: 1.0.0

Package → Add → Integration Flow
  Name:    Delivery Confirmation to ERP
  ID:      delivery-confirmation-to-erp

" Opens graphical iFlow editor
" Drag components from palette onto canvas
" Configure with Properties panel
```

## Common iFlow Patterns

### HTTP Trigger → OData Write (REST → SAP)

```
HTTP Sender → Content Modifier (set headers) → Request-Reply (OData adapter) → HTTP Response
```

### IDoc Inbound → Transform → OData

```
IDoc Sender → XSLT Mapping → Request-Reply (OData V2 adapter) → IDoc Ack
```

### Scheduled OData Poll → File / JMS

```
Timer Start → Request-Reply (OData GET) → Splitter → Router → SFTP / JMS Receiver
```

### AS2 EDI → EDIFACT Transform → SAP IDoc

```
AS2 Sender → EDI Splitter → EDIFACT-to-IDoc mapping (Integration Advisor) → IDoc Receiver
```

## Key Groovy Script Pattern

```groovy
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    def body = message.getBody(String)
    def headers = message.getHeaders()
    def props = message.getProperties()

    // Transform or validate
    def deliveryNo = props.get('DeliveryNo')

    // Set output
    message.setBody('{"deliveryNo":"' + deliveryNo + '","status":"Confirmed"}')
    message.setHeader('Content-Type', 'application/json')
    return message
}
```

## Key Transactions / Tools

| Tool | Purpose |
|---|---|
| Integration Suite UI (BTP) | Design, deploy, monitor iFlows |
| Web IDE / iFlow editor | Graphical iFlow canvas |
| Operations → Monitor | Message processing log, artifact status |
| Security Material | Credentials, OAuth, PGP keys, certificates |
| Integration Content | Package and iFlow versioning, deployment |
| Message Implementation Guide (MIG) | B2B message structure definition |
| Mapping Guideline (MAG) | B2B field mapping rules |
