# Routing and Error Handling Reference

## Router (Condition-Based Branching)

Sends the message down exactly one matching branch.

```
Step: Router
  Routing Conditions:
    Branch 1: "GoodsReceipt"
      Condition Type: Non-XML (Simple Expression)
      Condition:      ${header.DocType} = 'GR'
      Default:        false

    Branch 2: "GoodsIssue"
      Condition Type: Non-XML (Simple Expression)
      Condition:      ${header.DocType} = 'GI'
      Default:        false

    Branch 3: "Default"
      Default:        true    (catches unmatched messages)
```

### XPath Condition on XML Body

```
Condition Type: XPath
Expression:     //DocumentType = 'DELIVERY_CONFIRMATION'
Value Type:     nodeset/boolean
```

### Common Router Conditions

```
" Header value check:
${header.Content-Type} contains 'json'
${header.X-Action} = 'POST_GR'

" Property check:
${property.ValidationStatus} = 'VALID'
${property.ItemCount} > 0

" XPath on body:
//Status = 'Open'
//Priority[.='HIGH']

" Regular expression:
${header.DeliveryNo} regex 'DLV[0-9]{10}'
```

## Multicast (Fan-Out)

Sends the message to all branches in parallel (or sequential):

```
Step: Multicast
  Parallel Processing: true

  Branch 1: → JMS Receiver (audit log)
  Branch 2: → HTTP Receiver (carrier webhook)
  Branch 3: → Mail Receiver (email notification)

" All branches receive the same message simultaneously
" Use Sequential Multicast to process one at a time
```

## Splitter → Aggregator Pattern

Split a batch message, process each item, recombine:

```
[IDoc Batch Sender]
  ↓
[IDoc Splitter]          — splits IDOC_BATCH into individual IDocs
  ↓
[General Splitter]       — OR: split JSON array or XML repeating elements
  Expression: //item     (XPath) or ${bodyAs(String)} (custom)
  ↓
[Per-Item Processing]    — runs once per split message
  ↓
[Aggregator]
  Completion Condition: ${property.CamelAggregatedSize} >= ${property.TotalCount}
  Correlation Key:      ${header.BatchId}
  Strategy:             Collect (gather all into one)
```

### General Splitter for JSON Array

```
Step: General Splitter
  Expression Type:  XPath
  XPath:            /root/items/item

  " OR for JSON: convert to XML first, then split
  " Use Groovy to split JSON array and process with looping
```

## Filter

Discards messages that don't match a condition:

```
Step: Filter
  Condition Type:     XPath
  Expression:         //Status = 'Open'
  Filter Element:     (leave blank to filter entire message)
  
" Messages not matching are stopped (no error, just discarded)
" Messages matching pass through unchanged
```

## Exception Subprocess

Handles errors within the main flow without stopping the entire process:

```
iFlow layout:
  [Main Flow]
  [Exception Subprocess] — drawn as a separate pool below main flow
    Events:  Error Start Event → processing → Error End Event
             OR: Escalation End Event (re-raise for caller)
```

### Exception Subprocess Configuration

```
Exception Subprocess:
  Error Start Event:
    Error Code:   (leave blank = catch all errors)
                  OR specify: com.sap.esb.messaging.api.ESBMessagingException
  
  Steps inside:
    Content Modifier:
      Header: CamelExceptionCaught   Type: Header   (contains exception object)
    
    Script: ExtractErrorMessage
      Groovy: extract error details from exception
    
    Send to Dead Letter JMS Queue
    OR: HTTP Receiver (send alert to webhook)
    OR: Mail Receiver (send error email)
  
  Error End Event:
    Throw exception up (re-raise)
    OR: just end (suppress error, message marked failed but no re-throw)
```

### Groovy: Extract Exception Details

```groovy
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    def headers = message.getHeaders()
    
    // CamelExceptionCaught contains the exception
    def ex = headers.get('CamelExceptionCaught')
    
    def errorMsg = ex?.getMessage() ?: 'Unknown error'
    def errorType = ex?.getClass()?.getSimpleName() ?: 'Exception'
    
    def deliveryNo = message.getProperty('DeliveryNo') ?: 'UNKNOWN'
    
    message.setProperty('ErrorMessage', errorMsg)
    message.setProperty('ErrorType', errorType)
    message.setBody("""{"error":"${errorType}","message":"${errorMsg}","deliveryNo":"${deliveryNo}"}""")
    message.setHeader('Content-Type', 'application/json')
    
    return message
}
```

## Dead Letter Queue Pattern (JMS)

Store failed messages for manual retry or inspection:

```
Exception Subprocess:
  Error Start
    ↓
  Script: ExtractErrorDetails
    ↓
  Content Modifier:
    Header: JMSDeliveryMode  = 2 (persistent)
    Header: JMSExpiration    = 604800000  (7 days TTL)
    ↓
  JMS Receiver:
    Queue: DeliveryConfirmation-DLQ

" Ops team inspects DLQ via Integration Suite → Monitor → JMS
" Manual replay: resubmit from DLQ to main queue
```

## Retry Configuration (JMS Sender)

```
Sender → JMS Adapter:
  Max. Retries:        5
  Retry Interval:      60 seconds
  Exponential Backoff: true
  Dead-Letter Queue:   DeliveryDLQ   (after max retries exhausted)
```

## Idempotent Process Call (Duplicate Check)

Prevent processing the same message twice:

```
Step: Idempotent Process Call
  Message ID:      ${header.DeliveryNo}-${header.X-Request-Id}
  Skip Process Call on Duplicate: true
  Scope:           Integration Flow / Tenant
```

## Alert Rule (Monitoring)

```
Integration Suite → Monitor → Manage Alerts → Add Alert Rule
  Integration Flow:   Delivery-Confirmation-iFlow
  Condition:          Status = FAILED
  Period:             Last 1 hour
  Threshold:          3 failures
  Action:             Email / Webhook
```

## Retry from Monitor (Manual)

```
Integration Suite → Monitor → Monitor Message Processing
  Filter:   Status = FAILED, iFlow = Delivery-Confirmation
  Select failed message(s)
  Action:   Retry   (re-processes from the persisted checkpoint)
```
