# Message Transformation Reference

## Content Modifier

Set message headers, exchange properties, and body.

```
Tab: Message Header
  Name:    Content-Type       Type: Constant    Value: application/json
  Name:    Authorization      Type: Constant    Value: Bearer ${property.Token}
  Name:    X-DeliveryNo       Type: XPath       Value: //DeliveryNo/text()

Tab: Exchange Property
  Name:    WarehouseNo        Type: Header      Value: X-Warehouse-No
  Name:    Timestamp          Type: Expression  Value: ${date:now:yyyy-MM-dd'T'HH:mm:ss'Z'}
  Name:    TotalItems         Type: XPath       Value: count(//item)

Tab: Message Body
  Type:    Expression
  Body:    {"deliveryNo":"${property.DeliveryNo}","status":"Confirmed","ts":"${property.Timestamp}"}

  " Or: XPath to extract subset of XML
  Type:    XPath
  Value:   //DeliveryHeader
```

## Groovy Script

The most powerful transformation step — full Java/Groovy access to message, headers, properties.

### Script Skeleton

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import java.util.HashMap

def Message processData(Message message) {
    // Read body
    def bodyStr = message.getBody(String.class)
    
    // Read headers and properties
    def headers = message.getHeaders()
    def props   = message.getProperties()
    
    // Access specific values
    def deliveryNo = headers.get('X-DeliveryNo')
    def warehouseNo = props.get('WarehouseNo')
    
    // Transform
    def result = transformPayload(bodyStr, deliveryNo, warehouseNo)
    
    // Set output
    message.setBody(result)
    message.setHeader('Content-Type', 'application/json')
    message.setProperty('ProcessedAt', new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'"))
    
    return message
}

def String transformPayload(String body, String deliveryNo, String lgnum) {
    // Your transformation logic
    return """{"delivery":"${deliveryNo}","warehouse":"${lgnum}"}"""
}
```

### Working with JSON in Groovy

```groovy
import groovy.json.JsonSlurper
import groovy.json.JsonOutput
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    def body = message.getBody(String.class)
    def json = new JsonSlurper().parseText(body)
    
    // Access fields
    def deliveryNo = json.DeliveryNo
    def items      = json.Items   // List
    
    // Build output
    def output = [
        deliveryNo: deliveryNo,
        itemCount:  items?.size() ?: 0,
        items:      items?.collect { item ->
            [
                itemNo: item.ItemNo,
                material: item.Material,
                qty: item.Quantity as Double
            ]
        }
    ]
    
    message.setBody(JsonOutput.toJson(output))
    return message
}
```

### Working with XML in Groovy

```groovy
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    def body = message.getBody(String.class)
    def xml  = new XmlSlurper().parseText(body)
    
    // XPath-style navigation
    def deliveryNo = xml.IDOC.E1EDL20.VBELN.text()
    def items = xml.IDOC.E1EDL24

    def result = new StringBuilder()
    result << '{"deliveryNo":"' << deliveryNo << '","items":['
    
    items.eachWithIndex { item, idx ->
        if (idx > 0) result << ','
        result << '{"itemNo":"' << item.POSNR.text() << '"'
        result << ',"material":"' << item.MATNR.text() << '"}'
    }
    result << ']}'
    
    message.setBody(result.toString())
    return message
}
```

### HTTP Call from Groovy (ServiceCallout alternative)

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import groovy.json.JsonSlurper

def Message processData(Message message) {
    def deliveryNo = message.getProperty('DeliveryNo')
    
    // Call external REST API
    def url = new URL("https://api.carrier.com/track/${deliveryNo}")
    def conn = url.openConnection()
    conn.setRequestProperty('Authorization', "Bearer ${message.getProperty('CarrierToken')}")
    conn.setRequestProperty('Accept', 'application/json')
    
    def response = new JsonSlurper().parse(conn.inputStream)
    message.setProperty('TrackingStatus', response.status)
    
    return message
}
```

## XSLT Mapping

Use for XML-to-XML transformations — upload an `.xsl` file as an iFlow resource.

### XSLT File (IDoc DESADV → JSON-style XML)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="2.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:output method="xml" indent="yes"/>

  <xsl:template match="/">
    <DeliveryConfirmation>
      <DeliveryNo><xsl:value-of select="//IDOC/E1EDL20/VBELN"/></DeliveryNo>
      <ShipDate><xsl:value-of select="//IDOC/E1EDL20/WADAT_IST"/></ShipDate>
      <Items>
        <xsl:for-each select="//IDOC/E1EDL24">
          <Item>
            <ItemNo><xsl:value-of select="POSNR"/></ItemNo>
            <Material><xsl:value-of select="MATNR"/></Material>
            <Quantity><xsl:value-of select="LFIMG"/></Quantity>
            <UOM><xsl:value-of select="VRKME"/></UOM>
          </Item>
        </xsl:for-each>
      </Items>
    </DeliveryConfirmation>
  </xsl:template>
</xsl:stylesheet>
```

### XSLT Step Configuration

```
Step: XSLT Mapping
  XSLT File:   Resources → idoc-to-delivery.xsl    (uploaded to iFlow resources)
  Input:       Body
  Output:      Body
```

## Message Mapper (Graphical Mapping)

```
Step: Message Mapping
  Source Message: (define source XSD/WSDL structure)
  Target Message: (define target XSD/WSDL structure)
  
  " Drag fields to connect source → target
  " Add functions between connections:
    Constant, Concat, IfElse, Split, Date/Time, Custom Function

  Source field:  //IDOC/E1EDL20/VBELN
  Function:      removeLeadingZeros (custom)
  Target field:  //Delivery/DeliveryNo
```

### Custom Function in Message Mapper

```groovy
// In iFlow Resources → custom functions → MyFunctions.groovy
public class MyFunctions {
    public static String removeLeadingZeros(String value) {
        if (value == null) return ""
        return value.replaceAll("^0+", "")
    }
    
    public static String formatDate(String sapDate) {
        // SAP date: YYYYMMDD → ISO: YYYY-MM-DD
        if (!sapDate || sapDate.length() != 8) return ""
        return "${sapDate[0..3]}-${sapDate[4..5]}-${sapDate[6..7]}"
    }
}
```

## JSON-to-XML and XML-to-JSON (Built-in Steps)

```
Step: JSON to XML Converter
  JSON Prefix:   (optional namespace prefix)
  Root Element:  DeliveryPayload
  
Step: XML to JSON Converter
  JSON Output Key:  (root key name)
  Use Namespaces:   false
  Suppress JSON Root Element: false
```

## EDI Splitter (for AS2/EDIFACT Inbound)

```
Step: EDI Splitter
  Source Encoding:  UTF-8
  EDI Type:         UN/EDIFACT / ANSI X12
  Create Acknowledgement:  true (CONTRL/997)
  
" Output: individual EDI messages (one per transaction set)
" Each goes to next processing step independently
```

## EDI to XML Converter

```
Step: EDI to XML Converter
  EDI Type:     UN/EDIFACT
  Version:      D96A
  Message Type: DESADV   (dispatch advice)
  
" Converts EDIFACT text to XML representation
" Use with Message Mapping or XSLT for further transformation
```
