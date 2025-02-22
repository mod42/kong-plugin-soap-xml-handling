# Kong plugins: SOAP/XML Handling for Request and Response
This repository concerns Kong plugins developed in Lua and use the GNOME C libraries [libxml2](https://gitlab.gnome.org/GNOME/libxml2#libxml2) and [libxslt](https://gitlab.gnome.org/GNOME/libxslt#libxslt). Part of the functions are bound in the [XMLua/libxml2](https://clear-code.github.io/xmlua/) library.
Both GNOME C and XMLua/libxml2 libraries are already included in [kong/kong-gateway](https://hub.docker.com/r/kong/kong-gateway) Enterprise Edition Docker image, so you don't need to rebuild a Kong image.
These plugins don't apply to Kong OSS.

The plugins handle the SOAP/XML **Request** and/or the SOAP/XML **Response** in this order:

**soap-xml-request-handling** plugin to handle Request:

1) ```XSLT TRANSFORMATION - BEFORE XSD```: Transform the XML request with XSLT (XSLTransformation) before XSD Validation (step #2)
2) ```XSD VALIDATION```: Validate XML request with its XSD schema
3) ```XSLT TRANSFORMATION - AFTER XSD```: Transform the XML request with XSLT (XSLTransformation) after XSD Validation (step #2)
4) ```ROUTING BY XPATH```: change the Route of the request to a different hostname and path depending of XPath condition
5) ```EXTRACT OPERATION```: get the operation from SOAP 1.2 and add header SOAPAction for routing. Keeps the 1.1 SOAPAction if present.

**soap-xml-response-handling** plugin to handle Reponse:

5) ```XSLT TRANSFORMATION - BEFORE XSD```: Transform the XML response before step #6
6) ```XSD VALIDATION```: Validate the XML response with its XSD schema
7) ```XSLT TRANSFORMATION - AFTER XSD```:  Transform the XML response after step #6

Each handling is optional. In case of misconfiguration the Plugin sends to the consumer an HTTP 500 Internal Server Error ```<soap:Fault>``` (with the error detailed message)

![Alt text](/images/Pipeline-Kong-soap-xml-handling.png?raw=true "Kong - SOAP/XML execution pipeline")

![Alt text](/images/Kong-Manager.png?raw=true "Kong - Manager")


## How deploy SOAP/XML Handling plugins
1) Create and prepare a PostgreDB called ```kong-gateway-soap-xml-handling```.
[See documentation](https://docs.konghq.com/gateway/latest/install/docker/#prepare-the-database).

2) Provision a license of Kong Enterprise Edition and put the content in ```KONG_LICENSE_DATA``` environment variable. The following license is only an example. You must use the following format, but provide your own content.
```
 export KONG_LICENSE_DATA='{"license":{"payload":{"admin_seats":"1","customer":"Example Company, Inc","dataplanes":"1","license_creation_date":"2023-04-07","license_expiration_date":"2023-04-07","license_key":"00141000017ODj3AAG_a1V41000004wT0OEAU","product_subscription":"Konnect Enterprise","support_plan":"None"},"signature":"6985968131533a967fcc721244a979948b1066967f1e9cd65dbd8eeabe060fc32d894a2945f5e4a03c1cd2198c74e058ac63d28b045c2f1fcec95877bd790e1b","version":"1"}}'
```

3) Start the Kong Gateway
```
./start-kong.sh
```

## How configure and test ```calcWebService/Calc.asmx``` Service in Kong
1) Create a Kong Service named ```calcWebService``` with this URL: https://ecs.syr.edu/faculty/fawcett/Handouts/cse775/code/calcWebService/Calc.asmx.
This simple backend Web Service adds or subtracts 2 numbers.

2) Create a Route on the Service ```calcWebService``` with the ```path``` value ```/calcWebService```

3) Call the ```calcWebService``` through the Kong Gateway Route by using [httpie](https://httpie.io/) tool
```
http POST http://localhost:8000/calcWebService \
Content-Type:"text/xml; charset=utf-8" \
--raw "<?xml version=\"1.0\" encoding=\"utf-8\"?>
<soap:Envelope xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\" xmlns:xsd=\"http://www.w3.org/2001/XMLSchema\" xmlns:soap=\"http://schemas.xmlsoap.org/soap/envelope/\">
  <soap:Body>
    <Add xmlns=\"http://tempuri.org/\">
      <a>5</a>
      <b>7</b>
    </Add>
  </soap:Body>
</soap:Envelope>"
```

The expected result is ```12```:
```xml
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" ...>
  <soap:Body>
    <AddResponse xmlns="http://tempuri.org/">
      <AddResult>12</AddResult>
    </AddResponse>
  </soap:Body>
</soap:Envelope>
```

## How test XML Handling plugins with ```calcWebService/Calc.asmx```
### Example #1: Request | ```XSLT TRANSFORMATION - BEFORE XSD```: adding a Tag in XML request by using XSLT 

The plugin applies a XSLT Transformation on XML request **before** the XSD Validation.
In this example the XSLT **adds the value ```<b>8</b>```** that will not be present in the request.

Add ```soap-xml-request-handling``` plugin and configure the plugin with:
- ```XsltTransformBefore``` property with this XSLT definition:
```xml
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:output version="1.0" method="xml" encoding="utf-8" omit-xml-declaration="no"/>
  <xsl:strip-space elements="*"/>
  <xsl:template match="node()|@*">
    <xsl:copy>
      <xsl:apply-templates select="node()|@*"/>
    </xsl:copy>
  </xsl:template>   
  <xsl:template match="//*[local-name()='a']">
    <xsl:copy-of select="."/>
      <b>8</b>
  </xsl:template>
</xsl:stylesheet>
```
Use command defined at step #3, **remove ```<b>7</b>```**, the expected result is no longer ```12``` but ```13```:
```xml
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" ...>
  <soap:Body>
    <AddResponse xmlns="http://tempuri.org/">
      <AddResult>13</AddResult>
    </AddResponse>
  </soap:Body>
</soap:Envelope>
```
### Example #2: Request | ```XSD VALIDATION```: calling incorrectly ```calcWebService``` and detecting issue on the Request with XSD schema
Calling incorrectly ```calcWebService``` and detecting issue in the Request with XSD schema. 
We call incorrectly the Service by injecting a SOAP error; the plugin detects it, sends an error message to the Consumer and Kong doesn't call the SOAP backend API.

Open ```soap-xml-request-handling``` plugin and configure the plugin with:
- ```VerboseRequest``` enabled
- ```XsdApiSchema``` property with this value:
```xml
<xs:schema attributeFormDefault="unqualified" elementFormDefault="qualified" targetNamespace="http://tempuri.org/" xmlns:xs="http://www.w3.org/2001/XMLSchema">
  <xs:element name="Add" type="tem:AddType" xmlns:tem="http://tempuri.org/"/>
  <xs:complexType name="AddType">
    <xs:sequence>
      <xs:element type="xs:integer" name="a" minOccurs="1"/>
      <xs:element type="xs:integer" name="b" minOccurs="1"/>
    </xs:sequence>
  </xs:complexType>
  <xs:element name="Subtract" type="tem:SubtractType" xmlns:tem="http://tempuri.org/"/>
  <xs:complexType name="SubtractType">
    <xs:sequence>
      <xs:element type="xs:integer" name="a" minOccurs="1"/>
      <xs:element type="xs:integer" name="b" minOccurs="1"/>
    </xs:sequence>
  </xs:complexType>
</xs:schema>
```

Use command defined at step #3, **change** ```<soap:Envelope>``` by **```<soap:EnvelopeKong>```**  and **change** ```</soap:Envelope>``` by **```</soap:EnvelopeKong>```** => Kong says: 
```xml
HTTP/1.1 500 Internal Server Error
...
<faultstring>
Request - XSD validation failed: Error Node: EnvelopeKong, Error code: 1845, Line: 1, Message: Element '{http://schemas.xmlsoap.org/soap/envelope/}EnvelopeKong': No matching global declaration available for the validation root.
</faultstring>
```
Use command defined at step #3, **remove ```<a>5</a>```** => there is an error because the ```<a>``` tag has the ```minOccurs="1"``` XSD property and Kong says: 
```xml
HTTP/1.1 500 Internal Server Error
...
<faultstring>
Request - XSD validation failed: Error Node: Add, Error code: 1871, Line: 1, Message: Element '{http://tempuri.org/}Add': Missing child element(s). Expected is ( {http://tempuri.org/}a ).
</faultstring>
```
### Example #3: Request | ```XSLT TRANSFORMATION - AFTER XSD```:  renaming a Tag in XML request by using XSLT
The plugin applies a XSLT Transformation on XML request **after** the XSD Validation.
In this example we **change the Tag name from ```<Subtract>...</Subtract>```** (present in the request) **to ```<Add>...</Add>```**.

**Without XSLT**: Use command defined at step #3, rename the Tag ```<Add>...</Add>```, to ```<Subtract>...</Subtract>```, remove ```<b>7</b>```, so the new command is:
```
http POST http://localhost:8000/calcWebService \
Content-Type:"text/xml; charset=utf-8" \
--raw "<?xml version=\"1.0\" encoding=\"utf-8\"?>
<soap:Envelope xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\" xmlns:xsd=\"http://www.w3.org/2001/XMLSchema\" xmlns:soap=\"http://schemas.xmlsoap.org/soap/envelope/\">
  <soap:Body>
    <Subtract xmlns=\"http://tempuri.org/\">
      <a>5</a>
    </Subtract>
  </soap:Body>
</soap:Envelope>"
```

The expected result is ```-3```
```xml
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" ...>
  <soap:Body>
    <SubtractResponse xmlns:tempui="http://tempuri.org/">
      <SubtractResult>-3</SubtractResult>
    </SubtractResponse>
  </soap:Body>
</soap:Envelope>
```

Open ```soap-xml-request-handling``` plugin and configure the plugin with:
- ```XsltTransformAfter``` property with this XSLT definition:
```xml
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:output method="xml" version="1.0" encoding="utf-8" omit-xml-declaration="no" indent="yes"/>
  <xsl:strip-space elements="*"/>
  <xsl:template match="node()|@*">
    <xsl:copy>
      <xsl:apply-templates select="node()|@*"/>
    </xsl:copy>
  </xsl:template>   
  <xsl:template match="//*[local-name()='Subtract']">
    <Add xmlns="http://tempuri.org/"><xsl:apply-templates select="@*|node()" /></Add>
  </xsl:template>
</xsl:stylesheet>
```
**With XSLT**: Use command defined at Example #3, the expected result is ```13```:
```xml
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" ...>
  <soap:Body>
    <AddResponse xmlns="http://tempuri.org/">
      <AddResult>13</AddResult>
    </AddResponse>
  </soap:Body>
</soap:Envelope>
```
### Example #4: Request | ```ROUTING BY XPATH```: change the Route of the request to a different hostname and path depending of XPath condition
The plugin searches the XPath entry and compares it to a Condition value. If this is the right Condition value, the plugin changes the host and the path of the Route.

This example uses a new backend Web Service (https://websrv.cs.fsu.edu/~engelen/calc.wsdl) which provides the same capabilities as ```calcWebService``` Service (https://ecs.syr.edu) defined at step #1. 

Add a Kong ```Upstream``` named ```websrv.cs.fsu.edu``` and defines a ```target```with ```websrv.cs.fsu.edu:443``` value. 
Open ```soap-xml-request-handling``` plugin and configure the plugin with:
- ```RouteToPath``` property with the value ```/~engelen/calcserver.cgi```
- ```RouteToUpstream``` property with the value ```websrv.cs.fsu.edu```
- ```RouteXPath``` property with the value ```/soap:Envelope/soap:Body/*[local-name() = 'add']/*[local-name() = 'a']```
- ```RouteXPathCondition``` property with the value ```5```
- ```RouteXPathRegisterNs``` leave the default value; we can also register specific NameSpace with the syntax ```prefix,uri```
- ```XsltTransformAfter``` property with the following XSLT definition (the ```websrv.cs.fsu.edu``` introduces a new XML NameSpace so we have to change the XSLT transformation to make the proper call):
```xml
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:output method="xml" version="1.0" encoding="utf-8" omit-xml-declaration="no" indent="yes"/>
  <xsl:strip-space elements="*"/>
  <xsl:template match="node()|@*">
    <xsl:copy>
      <xsl:apply-templates select="node()|@*"/>
        </xsl:copy>
    </xsl:template>
    <xsl:template match="//*[local-name()='Subtract']">
      <urn:add xmlns:urn="urn:calc"><xsl:apply-templates select="@*|node()" /></urn:add>
  </xsl:template>
</xsl:stylesheet>
```
Use command defined at Example #3, the expected result is ```13```. The new Route (to ```websrv.cs.fsu.edu```) sends a slightly different response:
- SOAP tags are in capital letter: ```<SOAP-ENV:Envelope>``` instead of ```<soap:Envelope>```
- Namespace is injected: ```xmlns:ns="urn:calc"```
```xml
<?xml version="1.0" encoding="UTF-8"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/" ... xmlns:ns="urn:calc">
  <SOAP-ENV:Body SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
    <ns:addResponse>
      <result>13</result>
    </ns:addResponse>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```
### Example #5: Response | ```XSLT TRANSFORMATION - BEFORE XSD```: changing a Tag name in XML response by using XSLT
The plugin applies a XSLT Transformation on XML response **before** the XSD Validation.
In this example the XSLT **changes the Tag names**:
-  from ```<ns:addResponse>...</ns:addResponse>``` (present in the response) to **```<addResponse>...</addResponse>```**
-  from ```<result>...</result>``` (present in the response) to **```<KongResult>...</KongResult>```**

Add ```soap-xml-response-handling``` plugin and configure the plugin with:
- ```VerboseResponse``` enabled
- ```XsltTransformBefore``` property with this XSLT definition:
```xml
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:output method="xml" version="1.0" encoding="utf-8" omit-xml-declaration="no" indent="yes"/>
  <xsl:template match="@*|node()">
    <xsl:copy>
      <xsl:apply-templates select="@*|node()" />
    </xsl:copy>
  </xsl:template>
  <xsl:template match="//*[local-name()='addResponse']">
    <addResponse>
      <xsl:apply-templates select="@*|node()" />
    </addResponse>
  </xsl:template>
  <xsl:template match="//*[local-name()='result']">
    <KongResult><xsl:apply-templates select="@*|node()" /></KongResult>
  </xsl:template>
</xsl:stylesheet>
```
Use command defined at Example #3, the expected result is ```<KongResult>13</KongResult>```:
```xml
<?xml version="1.0" encoding="utf-8"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/" ... xmlns:ns="urn:calc">
  <SOAP-ENV:Body SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
    <addResponse>
      <KongResult>13</KongResult>
    </addResponse>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```
### Example #6: Response | ```XSD VALIDATION```: checking validity of XML response with its XSD schema
Open ```soap-xml-response-handling``` plugin and configure the plugin with:
- ```XsdApiSchema``` property with this value:
```xml
<xs:schema attributeFormDefault="unqualified" elementFormDefault="qualified" xmlns:xs="http://www.w3.org/2001/XMLSchema">
  <xs:element name="addResponse" type="addResponseType"/>
  <xs:complexType name="addResponseType">
    <xs:sequence>
      <xs:element type="xs:string" name="KongResult"/>
    </xs:sequence>
  </xs:complexType>
</xs:schema>
```
For testing purposes only: one can play with the XSD schema to raise error by temporarily replacing ```KongResult``` by ```KongResult2```

### Example #7: Response | ```XSLT TRANSFORMATION - AFTER XSD```:  transforming the SOAP response to a XML response
In this example the XSLT removes all <soap> tags and **converts the response from SOAP to XML**.

Open ```soap-xml-response-handling``` plugin and configure the plugin with:
- ```XsltTransformAfter``` property with this value:
```xml
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" exclude-result-prefixes="soapenv">
  <xsl:strip-space elements="*"/>
  <xsl:output method="xml" version="1.0" encoding="utf-8" omit-xml-declaration="no" indent="yes"/>
  <!-- remove all elements in the soapenv namespace -->
  <xsl:template match="soapenv:*">
    <xsl:apply-templates select="node()"/>
  </xsl:template>
  <!-- for the remaining elements (i.e. elements in the default namespace) ... -->
  <xsl:template match="*">
      <!-- ... create a new element with similar name in no-namespace -->
      <xsl:element name="{local-name()}">
          <xsl:apply-templates select="@*|node()"/>
      </xsl:element>
  </xsl:template>
  <!-- convert attributes to elements -->
  <xsl:template match="@*">
      <xsl:element name="{local-name()}">
          <xsl:value-of select="." />
      </xsl:element>
  </xsl:template>
</xsl:stylesheet>
```
Use command defined at Example #3, the expected result is:
```xml
<?xml version="1.0" encoding="utf-8"?>
<addResponse>
  <KongResult>13</KongResult>
</addResponse>
```

### Example #8: Response | Use a SOAP/XML WebService with a ```Content-Encondig: gzip```
With ```Content-Encondig: gzip``` the SOAP/XML Response body is zipped. So the ```soap-xml-response-handling``` has to unzip the SOAP/XML Response body, apply XSD and XSLT handling and re-zip the SOAP/XML Response body.

In this example the XSLT **changes the Tag names**:
-  from ```<m:NumberToWordsResult>...</m:NumberToWordsResult>``` (present in the response) to **```<KongResult>...</KongResult>```**

1) Create a Kong Service named ```dataAccess``` with this URL: https://www.dataaccess.com/webservicesserver/NumberConversion.wso. This simple backend Web Service converts a digit number to a number in full

2) Create a Route on the Service ```dataAccess``` with the ```path``` value ```/dataAccess```

3) Call the ```dataAccess``` through the Kong Gateway Route by using [httpie](https://httpie.io/) tool
```
http 'http://localhost:8000/dataAccess' \
Content-Type:'text/xml; charset=utf-8' \
Accept-Encoding:'gzip' \
--raw '<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <NumberToWords xmlns="http://www.dataaccess.com/webservicesserver/">
      <ubiNum>500</ubiNum>
    </NumberToWords>
  </soap:Body>
</soap:Envelope>'
```

The expected result is zipped with ```Content-Encoding: gzip``` header and we get ```<m:NumberToWordsResult>five hundred </m:NumberToWordsResult>```
```xml
...
Connection: keep-alive
Content-Encoding: gzip
Content-Length: 213
...
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <m:NumberToWordsResponse xmlns:m="http://www.dataaccess.com/webservicesserver/">
      <m:NumberToWordsResult>five hundred </m:NumberToWordsResult>
    </m:NumberToWordsResponse>
  </soap:Body>
</soap:Envelope>
```
4) Add ```soap-xml-response-handling``` plugin and and configure the plugin with:
- ```xsltTransformBefore``` property with this value:
```xml
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:output method="xml" version="1.0" encoding="utf-8" omit-xml-declaration="no" indent="yes"/>
  <xsl:template match="@*|node()">
    <xsl:copy>
      <xsl:apply-templates select="@*|node()" />
    </xsl:copy>
  </xsl:template>
   <xsl:template match="//*[local-name()='NumberToWordsResult']">
    <KongResult>
      <xsl:apply-templates select="@*|node()" />
    </KongResult>
  </xsl:template>
</xsl:stylesheet>
```
Use command defined at step #3, the expected result is zipped with ```Content-Encoding: gzip``` header and we get ```<KongResult>five hundred </KongResult>```
```xml
Connection: keep-alive
Content-Encoding: gzip
Content-Length: 185
...
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <m:NumberToWordsResponse xmlns:m="http://www.dataaccess.com/webservicesserver/">
      <KongResult>five hundred </KongResult>
    </m:NumberToWordsResponse>
  </soap:Body>
</soap:Envelope>
```
### Example #9: Request | Use a SOAP 1.2 WebService
1) Create a Kong Service
2) Add the ```soap-xml-request-handling``` plugin and and configure the plugin with:
- ```RouteXPath``` property with this ```//soap:Body/*[1]```
- ```extractOperation``` ticked

## Changelog
- v1.0.0:
  - Initial Release
- v1.0.1:
  - Improve the behavior of SOAP/XML Handling plugins in conjunction with the Kong System plugins: Rate Limiting, Auth (OIDC, basic-auth, etc.)
  - Reformat the JSON Error messages (of Kong System plugins) to a SOAP/XML `<soap:Fault>` error
- v1.0.2:
  - Add the capacity to provide `wsdl` content to `xsdApiSchema`. The raw `<xs:schema>` is still valid
- v1.0.3:
  - When `VerboseRequest` or  `VerboseResponse` are disabled, the plugins no longer send the detailed error to the logs
- v1.0.4: Improve the log error management by initializing it in the `init_worker` phase
