# SAP Integration Suite (Cloud Integration/CPI) Skill

Skill for SAP Integration Suite Cloud Integration — iFlow design, adapter configuration, message transformation, Groovy scripting, routing, error handling, security material, monitoring, and logistics integration patterns.

## Skill Overview

- **iFlow Design**: Sender/receiver channels, processing steps, start/end events, exception subprocesses
- **Adapters**: HTTP, OData V2/V4, SOAP, IDoc, JMS, SFTP, AS2, AMQP, Mail, JDBC
- **Transformations**: XSLT, Message Mapper, Content Modifier, Groovy script, JavaScript
- **Routing**: Router (condition-based), Multicast, Filter, Splitter, Aggregator
- **Error Handling**: Exception Subprocess, dead-letter JMS, error end event, alerting
- **Security**: Credential store, OAuth2 client credentials, PGP encryption, certificates, keystore
- **Monitoring**: Message processing log, artifact status, alerting, trace mode
- **Deployment**: Package versioning, transport, CTS+, environment promotion
- **Logistics Patterns**: IDoc↔REST, OData poll, EDI/AS2, EWM/TM event forwarding, carrier integration

## Auto-Trigger Keywords

### Core CPI / Integration Suite
- Cloud Integration, CPI, SAP CPI, iFlow
- Integration Suite, SAP Integration Suite
- integration flow, iFlow design, iFlow editor
- integration package, design time, runtime

### iFlow Components
- sender channel, receiver channel, adapter
- start event, end event, timer start
- Content Modifier, Script step, Mapping step
- Request-Reply, Send step, Call adapter
- Exception Subprocess, error end event

### Adapters
- HTTP adapter, OData adapter, SOAP adapter
- IDoc adapter, JMS adapter, SFTP adapter
- AS2 adapter, AMQP adapter, Mail adapter
- JDBC adapter, SuccessFactors adapter, Ariba adapter
- XI adapter, RFC adapter, Open Connectors

### Transformation
- XSLT mapping, XSLT transformation
- Message Mapper, graphical mapping
- Content Modifier, set header, set property
- Groovy script, JavaScript script, CPI script
- EDI to XML, EDIFACT, ANSI X12, IDOC mapping

### Routing
- Router, condition-based routing, XPath routing
- Multicast, parallel multicast, sequential multicast
- Splitter, general splitter, IDoc splitter, PKCS splitter
- Aggregator, filter, join

### Error Handling
- Exception Subprocess, error handling iFlow
- dead letter queue, JMS dead letter
- escalation end event, error end event
- retry, exponential backoff, max retry
- alert, monitoring alert

### Security
- Security Material, credential store
- User Credentials, OAuth2, client credentials
- Secure Parameter, PGP, keystore
- certificate, X.509, known hosts SFTP

### Monitoring
- Message Processing Log, MPL
- message status, FAILED, COMPLETED, RETRY
- iFlow monitoring, artifact status
- trace mode, message trace
- Operations cockpit, message dashboard

### B2B / EDI
- AS2, AS4, EDIFACT, ANSI X12
- EDI splitter, EDI validator
- Integration Advisor, MIG, MAG
- Trading Partner Management, partner profile
- B2B integration, EDI integration

### Logistics Patterns
- IDoc to REST, REST to IDoc
- OData poll, scheduled poll
- delivery confirmation flow, GR posting flow
- EWM integration, TM integration
- carrier EDI, freight invoice EDI
- event mesh, SAP Event Mesh
