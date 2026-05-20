# iFlow Design Reference

## iFlow Canvas Elements

```
Palette categories:
  Events:     Start, Timer, End, Error End, Escalation End
  Steps:      Content Modifier, Script, Filter, Router, Splitter, Aggregator
  Calls:      Request-Reply, Send, ProcessDirect
  Mapping:    Message Mapping, XSLT Mapping
  Security:   Encryptor, Decryptor, Signer, Verifier
  Persistence: Write Variables, Data Store Operations
  External:   Sender/Receiver (drag to connect adapters)
```

## Step Reference

| Step | Type | Use For |
|---|---|---|
| Content Modifier | Transformation | Set/remove headers, properties, body |
| Script (Groovy/JS) | Transformation | Custom logic, complex transformation |
| Message Mapping | Transformation | Graphical field mapping |
| XSLT Mapping | Transformation | XML-to-XML via XSLT stylesheet |
| Router | Routing | Conditional branching (XPath/header conditions) |
| Multicast | Routing | Send to multiple branches simultaneously |
| Splitter | Routing | Split one message into many |
| Aggregator | Routing | Combine split messages back into one |
| Filter | Routing | Keep or discard message based on condition |
| Request-Reply | Call | Call external system, wait for response |
| Send | Call | Fire-and-forget to receiver |
| ProcessDirect | Call | Call another iFlow within same tenant |
| Data Store Write/Get | Persistence | Store/retrieve message for async processing |
| Write Variables | Persistence | Store value to iFlow variable |

## Content Modifier

The most commonly used step — sets message headers, exchange properties, and body.

```
Step: Content Modifier
  
  Message Header tab:
    Action: Create/Overwrite
    Name:   Content-Type
    Type:   Constant
    Value:  application/json

    Name:   SAP-Client
    Type:   Constant
    Value:  100

    Name:   DeliveryNo
    Type:   XPath
    Value:  //DeliveryNo/text()
    Datatype: java.lang.String

  Exchange Property tab:
    Name:   WarehouseNo
    Type:   Header         (read from incoming header)
    Value:  X-Warehouse-No

  Message Body tab:
    Type:   Expression
    Body:   ${property.responsePayload}
```

## Timer-Based iFlow (Scheduled Poll)

```
Timer (Start Event)
  ├── Schedule: Every 5 minutes / cron expression
  │   Cron: 0 */5 * * * ?   (every 5 min)
  │   OR: On Date: specific datetime, run once
  └── → processing steps
```

```
Properties → Scheduler:
  Run Once:   2026-06-01T08:00:00Z
  Recurring:  Interval: 5 minutes
  Cron:       0 0 6 * * MON-FRI   (6am weekdays)
```

## Request-Reply Call

Calls an external system and brings the response back into the iFlow:

```
Request-Reply
  → Receiver Channel (OData / HTTP / SOAP)
  → Response available as message body for next steps
```

## ProcessDirect (iFlow-to-iFlow)

```
iFlow A: Delivery Inbound (main flow)
  → ProcessDirect → calls "Delivery Validation" sub-iFlow
                  ← returns validation result

iFlow B: Delivery Validation (local endpoint: "DeliveryValidation")
  ProcessDirect Receiver
  → validation logic
  → ProcessDirect Sender back to caller
```

## Data Store (Async Persistence)

```
Store message for retry or async processing:

Write Data Store:
  Data Store Name:  PendingDeliveries
  Entry ID:         ${header.DeliveryNo}
  Retention:        90 days
  Overwrite:        true (update if same entry ID exists)

Get Data Store (in another iFlow or later):
  Data Store Name:  PendingDeliveries
  Entry ID:         ${header.DeliveryNo}
  Delete on Completion: true
```

## iFlow Variables

```
Write Variables step:
  Variable Name:  TotalItems
  Type:           XPath
  Value:          count(//item)
  Data Type:      java.lang.Integer

" Access in subsequent steps:
  ${variable.TotalItems}
  ${property.TotalItems}   (if set as exchange property)
```

## Correlation Key for Aggregator

```
Splitter → [N parallel branches] → Aggregator

Aggregator:
  Completion Condition: Expression
    ${property.CamelAggregatedSize} >= ${property.TotalCount}
  Correlation Expression: ${header.DeliveryNo}
  Aggregation Algorithm: Combine (concat bodies)
  OR: Script (custom merge logic)
```

## iFlow Properties (Runtime)

Set at iFlow level, accessible via `${property.name}`:

```
iFlow → Properties → Externalize:
  Name:         BackendUrl
  Default:      https://my-s4.ondemand.com
  Description:  Target S/4HANA URL

" Externalized properties can be overridden per deployment
" without changing the iFlow code
```

## Common Expression Language (Simple Expressions)

```
${header.Content-Type}          — message header
${property.DeliveryNo}          — exchange property
${body}                         — full message body
${date:now:yyyy-MM-dd}          — current date formatted
${exchangeId}                   — unique message ID
${camelId}                      — Camel exchange identifier
${header.CamelFileName}         — SFTP file name
```

## Pattern: HTTP Inbound → Validate → OData Write

```
[HTTP Sender]
  ↓
[Content Modifier]  — extract DeliveryNo from body into property
  ↓
[Script: ValidatePayload]  — Groovy: check required fields
  ↓
[Router]
  ├── valid → [Request-Reply: OData POST to SAP]
  │             ↓
  │           [Content Modifier: build success response]
  │             ↓
  │           [HTTP Sender Reply]
  └── invalid → [Script: BuildErrorResponse]
                  ↓
                [HTTP Sender Reply with 400]
```

## Pattern: IDoc Inbound → Transform → REST Notification

```
[IDoc Sender]
  ↓
[XSLT Mapping]  — IDOC_DESADV → JSON
  ↓
[Script: EnrichWithTimestamp]
  ↓
[Request-Reply: HTTP POST to carrier webhook]
  ↓
[Content Modifier: log result]
  ↓
[IDoc Sender Reply: ACK]
```
