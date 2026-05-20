# Adapters Reference

## HTTP Adapter (Sender — REST Inbound)

```
Sender → HTTP Adapter:
  Address:           /delivery/confirm        (relative URL — appended to iFlow endpoint)
  CSRF Protected:    true (for POST/PUT/DELETE from browser-based clients)
  Authorization:     Client Certificate / User Role / None
  Request Headers:   (forward all or specify list)
  Message Exchange Pattern: Request-Reply (synchronous) / One-Way (async)
```

Full iFlow URL:
`https://<tenant>.hana.ondemand.com/http/delivery/confirm`

## HTTP Adapter (Receiver — REST Outbound)

```
Receiver → HTTP Adapter:
  Address:           https://api.carrier.com/shipment/status
  Method:            POST / GET / PATCH / DELETE
  Authentication:    OAuth2 Client Credentials (select from Security Material)
  Request Headers:   Content-Type: application/json
  Timeout:           30000 ms
  Throw Exception on Failure: true
```

## OData V2 Adapter (SAP Backend)

### Receiver (Write to SAP)

```
Receiver → OData V2 Adapter:
  Address:           https://my-s4.ondemand.com
  XSRF Token:        Enabled     (required for POST/PATCH/DELETE to SAP OData V2)
  Path:              /sap/opu/odata/sap/ZEWM_DELIVERY_SRV
  Operation:         Create / Update / Delete / Query
  Entity:            GoodsReceiptSet
  Key Properties:    DeliveryNo='${header.DeliveryNo}'   (for Update/Delete)
  Custom Query Options: $format=json
  Authentication:    Basic (from Security Material) / OAuth2
```

### Sender (Poll SAP OData)

```
Sender → OData V2 Adapter:
  Address:           https://my-s4.ondemand.com
  Path:              /sap/opu/odata/sap/ZEWM_DELIVERY_SRV
  Operation:         Query
  Entity:            DeliverySet
  Query Options:     $filter=Status eq 'Open'&$orderby=ShipDate&$top=100&$format=json
  Authentication:    Basic
  Polling Interval:  (set on Timer Start Event instead)
```

## OData V4 Adapter

```
Receiver → OData V4 Adapter:
  Address:           https://my-s4.ondemand.com
  Path:              /sap/opu/odata4/sap/api_outbound_delivery_srv/srvd_a2x/sap/...
  Operation:         Create / Read / Update / Delete / Function / Action
  Entity Set:        A_OutbDeliveryHeader
  Authentication:    OAuth2 Client Credentials
```

## SOAP Adapter

```
Receiver → SOAP Adapter:
  Address:           https://my-system.com/ws/DeliveryService
  WSDL URL:          https://my-system.com/ws/DeliveryService?wsdl
  Service:           DeliveryService
  Endpoint:          DeliveryServicePort
  Operation:         ConfirmDelivery
  Authentication:    Basic / WS-Security
  WS-Security:       Username Token / X.509
```

## IDoc Adapter (Sender — Receive IDocs from SAP)

```
Sender → IDoc Adapter:
  Address:           /idoc/delivery       (relative path)
  IDoc Type:         DESADV01 / SHPCON01 / WMMBXY (etc.)
  " SAP sends IDocs to this CPI endpoint via RFC destination
  " RFC dest in SM59: HTTP type pointing to this iFlow URL
```

SAP ABAP side:
```abap
" IDoc partner profile (WE20) → port → HTTP connection to CPI endpoint
" Or: IDoc outbound processing with port type 'XML'
" Or: Configure in SM59 → HTTP destination → CPI iFlow URL
```

## IDoc Adapter (Receiver — Send IDocs to SAP)

```
Receiver → IDoc Adapter:
  Address:           https://my-s4.ondemand.com
  Path:              /sap/bc/srt/idoc    (or /sap/bc/idoc_xml/service)
  IDoc Content Type: IDoc / IDoc-Batch
  Authentication:    Basic
  Client:            100
  Sender Port:       CPI_INTEGRATION
  Receiver Port:     S4HANA_PROD
```

## JMS Adapter (Asynchronous Messaging)

```
Sender → JMS Adapter:
  Destination Name:  DeliveryConfirmationQueue
  Destination Type:  Queue / Topic
  Max. Retries:      3
  Retry Interval:    60 seconds
  Dead-Letter Queue: DeliveryDLQ
  Transaction:       Required

Receiver → JMS Adapter:
  Destination Name:  OutboundNotificationQueue
  Destination Type:  Queue
  Quality of Service: Exactly Once (via JMS transaction)
```

## SFTP Adapter (Sender — Poll Files)

```
Sender → SFTP Adapter:
  Host:              sftp.carrier.com
  Port:              22
  Directory:         /inbound/confirmations
  File Name:         *.csv / EDI*.txt
  Known Hosts File:  (from Security Material)
  Authentication:    Public Key (from Security Material)
  
  Post-Processing:
    Move to:         /processed/%date:yyyyMMdd%/
    OR: Delete File: true
  
  Polling Interval:  5 minutes   (set on Timer Start)
```

## SFTP Adapter (Receiver — Write Files)

```
Receiver → SFTP Adapter:
  Host:              sftp.partner.com
  Port:              22
  Directory:         /outbound/invoices
  File Name:         INV_${header.InvoiceNo}_${date:now:yyyyMMddHHmmss}.xml
  Duplicate Check:   true (avoid writing same file twice)
  Authentication:    User Credentials (from Security Material)
```

## AS2 Adapter (B2B / EDI)

```
Sender → AS2 Adapter:
  Address:           /as2/carrier/inbound
  Message ID Pattern: <timestamp>@mycompany.com
  Partner ID:        CARRIER_AS2_ID
  Own ID:            MYCOMPANY_AS2_ID
  Decrypt Message:   true (private key from Keystore)
  Verify Signature:  true (partner certificate from Keystore)
  MDN:               Synchronous
  MDN Signature:     Required

Receiver → AS2 Adapter:
  Host:              https://as2.carrier.com
  Partner AS2 ID:    CARRIER_AS2_ID
  Own AS2 ID:        MYCOMPANY_AS2_ID
  Sign Message:      true (private key alias: mycompany-signing)
  Encrypt Message:   true (partner public cert alias: carrier-enc)
  MDN:               Synchronous / Asynchronous
```

## AMQP Adapter (SAP Event Mesh)

```
Sender → AMQP Adapter (subscribe to Event Mesh):
  Host:              my-event-mesh.messaging.svc.cf.us10.hana.ondemand.com
  Port:              5671 (AMQP over TLS)
  Queue:             /queues/ewm/delivery-events
  Authentication:    OAuth2 (Event Mesh service key)

Receiver → AMQP Adapter (publish to Event Mesh):
  Host:              <same>
  Topic:             /topic/ewm/delivery/confirmed
  Quality of Service: At Least Once
```

## Mail Adapter (Receiver — Send Email)

```
Receiver → Mail Adapter:
  Host:              smtp.office365.com
  Port:              587
  Security:          STARTTLS
  Authentication:    Encrypted User/Password
  From:              integration@mycompany.com
  To:                ${property.RecipientEmail}
  Subject:           Delivery ${header.DeliveryNo} Confirmed
  Body:              <html><body>...</body></html>
  Body MIME Type:    text/html
  Attachment:        ${property.PDFContent}  (base64 or binary)
  Attachment Name:   Delivery_${header.DeliveryNo}.pdf
```
