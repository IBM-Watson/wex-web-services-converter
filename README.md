# Using the Watson Explorer Engine Web Services Converter

[IBM Watson Explorer](http://www.ibm.com/smarterplanet/us/en/ibmwatson/explorer.html) combines content and data from many different systems throughout the enterprise and presents it to users in a single view, dramatically reducing the amount of time spent looking for information and increasing their ability to work smarter. Watson Explorer’s 360-degree information applications deliver data, analytics, and cognitive insights relevant to the user’s role, context and current activities. These applications can be enhanced using content from external sources, external visualization libraries (such as D3.js), and external APIs. Integrating with external services like those available in the [Watson Developer Cloud](http://www.ibm.com/smarterplanet/us/en/ibmwatson/developercloud/) provides opportunities for further enhancing Watson Explorer applications to include cognitive-based features.

With the Watson Explorer Engine Web Services Converter described here, the process for augmenting data with content and analysis from external web services during ingestion is generalized for use with nearly any web service.  While the Watson Explorer Engine Web Services Converter can be used to interact with a wide variety of web services, this documentation also walks through a specific example where we configure the Web Services Converter for use with the [Watson Developer Cloud Relationship Extraction service](http://www.ibm.com/smarterplanet/us/en/ibmwatson/developercloud/relationship-extraction.html).  We then provide you with some food for thought &mdash; What should you think about when deploying this kind of integration into a production environment?  What are some additional ideas for integration? 

By the end of this example you should understand what the Watson Explorer Engine Web Services Converter does, how it could benefit your organization, and how to integrate it with Watson Explorer Engine's ingestion to enrich data.

## Architecture

The Watson Explorer Engine Web Services Converter is a [converter](http://www.ibm.com/support/knowledgecenter/SS8NLW_10.0.0/com.ibm.swg.im.infosphere.dataexpl.engine.doc/c_vse_converting.html) like any other that processes text to be indexed at ingestion time.  When the Web Services Converter is configured properly, it will send administrator-defined name-value pairs as CGI parameters to a REST-based web service.  The response from the REST web service is then stored in its entirety in a new administrator-specified [`<content>`](http://www.ibm.com/support/knowledgecenter/SS8NLW_10.0.0/com.ibm.swg.im.infosphere.dataexpl.engine.schema.doc/r_schema-ref-element-content.html) element.  The Web Services Converter is designed to work in conjunction with **two** [custom converters](http://www.ibm.com/support/knowledgecenter/SS8NLW_10.0.0/com.ibm.swg.im.infosphere.dataexpl.engine.doc/c_vse_converting_custom.html) which handle:

 * The pre-processing of text to prepare the name-value pairs for consumption by the web service.  The output of the custom converter responsible for pre-processing should contain a `<content>` for each CGI parameter to be sent to the configured web service.  The name and value of the `<content>` will be sent as a CGI parameter name and value.
 * The post-processing of the web service response.  The complete web service response is stored in a `<content>`.  Under most circumstances, the web service response needs to be processed and transformed into Watson Explorer Engine XML (*VXML*) to be useful.  It is also likely that some `<content>` elements can be discarded at this time, like the `<content>` containing the original web service response and any `<content>` whose only purpose was to represent a CGI parameter for the web service call.

A detailed example is provided below which demonstrates Web Services Converter configuration as well as appropriate pre-processing and post-processing via custom converters.

 *Note*:  IBM recommends complementing this out of the box architecture with a caching proxy.   Routing web service calls through a caching proxy can speed up refreshes and recrawls in some situations, overcome some network failures, and may also allow you to reduce the total number of web service transactions.

## Example demonstrating enrichment of ingested data

This example shows how to configure the Watson Explorer Engine Web Services Converter for a specific web service, namely the Watson Developer Cloud Relationship Extraction service.  To complete this example, custom converters which do the pre-processing of text and post-processing of the web service response are also discussed.

### Prerequisites

 1. Watson Explorer Engine installed and running.  Basic familiarity with the Engine software is assumed.  You will find completion of the [Engine Metadata Tutorial](http://www.ibm.com/support/knowledgecenter/SS8NLW_10.0.0/com.ibm.swg.im.infosphere.dataexpl.engine.tut.md.doc/c_vse-mt-oview.html) helpful if you are not already familiar with Watson Explorer Engine.
 2. A running instance of the Watson Relationship Extraction service.  You can quickly and easily deploy [an example Bluemix application](https://github.com/Watson-Explorer/wex-wdc-integration-samples/tree/master/wex-re/BlueMix) that exposes the Relationship Extraction service by following the [Watson Explorer WDC Relationship Extraction integration example](https://github.com/Watson-Explorer/wex-wdc-integration-samples/blob/master/wex-re/watson-re-readme.md#configuring-and-deploying-the-bluemix-custom-watson-relationship-extraction-web-service).  You do not need to complete all parts of the example Relationship Extraction integration, deployment of the Bluemix application is sufficient.
    Alternatively, you may want to experiment with a different web service.  To do so you will need to be able to understand and modify the XSL of the example pre-processing and post-processing converters.

### Add the Web Services Converter to your Engine installation

The Web Services Converter is XML, so it can be added to your Engine installation easily via copy-and-paste.

 1. Navigate to the [Web Services Converter XML source](/engine/function.vse-converter-webservice.xml), select and copy all the XML there.
 2. Navigate to your Watson Explorer Engine administrative interface.  Select the "Configuration" menu.  Click the "+" next to the "XML" item in the left menu to add new XML.
 3. You will be prompted to provide an Element and Name.  Enter "function" and "vse-converter-webservice" respectively, without quotes.  Click the **Add** button at right.
 4. You will see minimal XML representing the new function.  Select the minimal XML and then paste the Web Services Converter XML which you copied in step 1.  Click the **OK** button at right.

There are alternative methods for adding XML to your Engine installation, including via the repository-supplements directory, via peer repository configuration, and via the repository-add API function, but explanation of those methods is beyond the scope of this example.

### Create a new collection with the Web Services Converter

 1. Navigate to your Watson Explorer Engine administrative interface.  Select the "Configuration" menu.  Click the "+" next to the "Search Collections" item in the left menu to create a new collection.
 2. For "Copy defaults from" select "example-metadata".  This allows us to start with a pre-configured small example collection.  Name the collection and click the **Add** button at right.
 3. Select your collection's "Configuration" and then the "Converting" tab.  Click the **Add a new converter** button.  Scroll down and select the "Web Services" converter and click **Add**.
 4. In the "Web Service endpoint URL" text box, supply the endpoint URL for your application which exposes the Relationship Extraction service.
 5. In the "Contents to send as name/value pairs" text area, enter "sid" and "text" on different lines without quotes.  Click the **OK** button at right.

### Create the custom converter which pre-processes text

REST-based web services take input in the form of CGI parameters.  CGI parameters are name-value pairs.  `<content>` elements also have a name and a value.  This similarity is leveraged by the Web Services converter.  In the configuration for the Web Services converter, `<content>` elements are identified by name, and those content names and related content values are sent as CGI parameter names and values to the web service.

The Relationship Extraction service expects two parameters: "sid" and "text".

 * The value of "sid" identifies the training set that the service uses.  Currently, values may be either "ie-en-news" or "ie-es-news".
 * The value of "text" is the complete text from which you want to extract relationships.

To prepare the data for use by the Web Services converter, we must create a custom converter that:

 1. Copies all input to the output.  This should preserve all XML input, including all XML elements and their attributes.
 2. Generates new `<content>` elements.  One new `<content>` should be created for each CGI parameter that must be sent to the web service.
 
 *Note*:  In some cases your `<document>` may already have a `<content>` whose name is the same as one of the web service CGI parameters.  If this `<content>` already accurately represents a name-value pair that must be sent to the web service, then no further action is necessary.  If, however, the `<content>` should not be sent but has a name which matches a CGI parameter name that must be sent, then this pre-processing converter must **_replace_** this `<content>` with a new `<content>` with a new name.  The value can be moved back into a `<content>` with the original name in the custom post-processing converter.
 
The following XSL can be dropped into a custom converter to accomplish this.  The following XSL:

 1. Copies all input to the output.
 2. Creates a new `<content>` named "sid" and gives it a value of "ie-en-news".
 3. Creates a new `<content>` named "text" and copies the value of the `<content>` named "snippet" into it.  The "snippet" `<content>` often contains the bulk of the unstructured text that will be indexed.

```xml
<xsl:template match="/">
  <xsl:apply-templates select="@*|node()" />
</xsl:template>

<xsl:template match="@*|node()">
  <xsl:copy>
    <xsl:apply-templates select="@*|node()" />
  </xsl:copy>
</xsl:template>

<xsl:template match="document">
  <xsl:copy>
    <xsl:apply-templates select="@*|node()" />

    <!-- that takes care of copying the contents, now add any new ones -->
    <content name="sid">ie-en-news</content>

  </xsl:copy>
</xsl:template>

<xsl:template match="content[@name='snippet']">
  <xsl:copy>
    <xsl:apply-templates select="@*|node()" />
  </xsl:copy>

  <!-- copy the snippet, but also add a content to send to the webservice -->
  <content name="text">
    <xsl:value-of select="./text()" />
  </content>

</xsl:template>
```

To create a custom converter with the above XSL, follow these steps:

 1. Click the **Add a new converter** button, select "Custom converter", and click **Add**.
 2. For "Type-In" select "application/vxml-unnormalized".
 3. For "Type-Out" select "application/vxml-unnormalized".
 4. Give your converter a name, for example "Pre-processor for web service"
 5. For "Action" select "xsl".
 6. Paste the above XSL into the text area.
 7. Click the **OK** button at right.
 8. Ensure that the new "Pre-processor for web service" converter appears in the converter list **prior** to the Web Services converter.

### Create the custom converter which post-processes the response

The Web Services converter places the response from the configured web service in a new `<content>`.  In most cases, you will want to process this response in some way, usually by extracting relevant data and placing it into one or more additional new `<content>` elements.  You may also wish to discard this response `<content>` as well as the `<content>`s that were created only for the purpose of representing the CGI parameters sent to the configured web service.

The following XSL can be dropped into a custom converter to accomplish this.  The following XSL:

 1. Copies all input to the output.
 2. Specially handles the `<content>` named "webservice-response"
   * Parses the string value into XML
   * Loops over interesting XML elements in the web service response
   * Creates new `<content>` elements based on the web service response
   * Discards the "webservice-response" `<content>`
 3. Discards the "sid" and "text" `<content>`s which were only needed temporarily and never intended to be indexed.

```xml
<xsl:template match="/">
  <xsl:apply-templates select="@*|node()" />
</xsl:template>

<xsl:template match="@*|node()">
  <xsl:copy>
    <xsl:apply-templates select="@*|node()" />
  </xsl:copy>
</xsl:template>

<xsl:template match="content[@name='webservice-response']">
  <xsl:variable name="sgts"> > </xsl:variable>

  <!-- the response from my webservice is XML, so it will be convenient to treat it as XML instead of text -->
  <xsl:variable name="response" select="viv:string-to-node(.)" />

  <!-- now turn the relevant data from the webservice response into indexable content elements -->
  <!-- first, entities -->
  <xsl:for-each select="$response/rep/doc/entities/entity">
    <content name="watson-relationship-extraction-entity" entity-type="{@type}" entity-score="{@score}" type="text" action="none" >
      <xsl:value-of select="./mentref[1]" />
    </content>
  </xsl:for-each>
  <!-- second, relationships -->
  <xsl:for-each select="$response/rep/doc/relations/relation">
    <xsl:variable name="relation-type" select="@type" />
    <xsl:for-each select="./relmentions/relmention">
      <content name="watson-relationship-extraction-relation-tree" type="text" action="none" >
        <xsl:value-of select="concat($relation-type, $sgts, ./rel_mention_arg[@argnum='1'], $sgts, ./rel_mention_arg[@argnum='2'])" />
      </content>
    </xsl:for-each>
  </xsl:for-each>
</xsl:template>

<!-- drop the contents that we created for the webservice request but don't want to index -->
<xsl:template match="content[@name='sid']" />

<xsl:template match="content[@name='text']" />
```

To create a custom converter with the above XSL, follow these steps:

 1. Click the **Add a new converter** button, select "Custom converter", and click **Add**.
 2. For "Type-In" select "application/vxml-unnormalized".
 3. For "Type-Out" select "application/vxml-unnormalized".
 4. Give your converter a name, for example "Post-processor for web service"
 5. For "Action" select "xsl".
 6. Paste the above XSL into the text area.
 7. Click the **OK** button at right.
 8. Ensure that the new "Post-processor for web service" converter appears in the converter list **after** the Web Services converter.  Drag the number to the left of the post-processing converter to move it down below the Web Services converter.

### Crawl the collection

Your collection is now configured to crawl the metadata tutorial documents and enrich those documents with analysis from the Watson Developer Cloud Relationship Extraction service.

Before starting any crawl however, it is always a good idea to navigate to the overview tab of your collection and click the **Test it** button.  Do that now and you should see all of the files in the metadata tutorial enqueued.  Click the **Test it** button across from any of the files and that file should be fetched and processed by the conversion framework.  You should see each of the three converters you added active in the conversion trace.

In particular, notice that the Web Services converter added data.  Click on the number representing the size of the output from the Web Services converter.  You will be taken to a new tab containing the VXML output of the Web Services converter.  Here you will find VXML that would normally be indexed during the metadata example crawl, as well as the "sid", "text", and "webservice-response" `<content>`s.  Confirm that the text in the "webservice-response" `<content>` contains what you would expect to see as a response from your web service.

If the conversion details revealed by **Test it** look good, navigate back to the overview tab of your collection and start a crawl.  When the crawl completes, perform a search in your collection, turn on debugging, and expose the XML for a result by clicking the [XML] button in the result's footer.  You will find all the `<content>` elements that normally appear in the metadata example crawl, as well as new `<content>` elements that represent the entities and relationships discovered by the Watson Developer Cloud Relationship Extraction service!

## Production and Deployment Considerations

This example is intended for demonstrative purposes only.  While you might be able to reuse the patterns and even parts of the code from these examples, there are several points that should be considered when developing a production-grade application.

 * **Security** - At the very least, you should ensure that the web service you are using can be accessed via an encrypted HTTPS connection if you might be crawling any data of a sensitive nature.  There are additional security options in the "Advanced HTTP Config" section of the Web Services converter configuration that will allow Engine to establish a secure channel, authenticate, etc.  See the tool tips on individual settings for more detail.
 * **Performance** - Adding the Web Services converter can significantly impact the collection's total crawl time.  Factors include the size of the web service request, the size of the web service response, the web service processing time, latency and bandwith of the networks connecting the WEX server and the web service endpoint, etc.
   As mentioned earlier, a caching proxy will likely increase crawl performance if there is any chance of duplicate web service calls during crawling or subsequent refreshes.
 * **Failures will happen** - All distributed systems are inherently unreliable and failures will inevitably occur when calling out to a web service. Carefully consider how failures should be handled at conversion time. Should the whole document fail? Is a partially indexed document without enriched metadata OK for your use cases?
 * **Data Preparation** - It is the responsibility of the caller to ensure that representative data is being sent to a web service. Data preparation strategies not demonstrated here may be required in some cases. For example, some Engine converters will produce HTML tags in the "snippet" `<content>`, such as PDFtoHTML or WordtoHTML. These tags provide hints to the indexer but in a snippet become encoded XML. Prepare your data carefully and ensure it is clean enough for your web service.
 * **Scalability** - Some web services will enforce limits on usage.  Such limits may include metrics like calls per day, data per call, number of simultaneous calls, etc.  Carefully consider the demand you are placing on the web service.  You may need to reduce the aggressiveness of the Engine crawl, reduce the number of converters that may be running simultaneously, or reduce the number of simultaneously active crawls in order to stay within the operational limits of the web service or within the limits of your license to use it.
 

# Licensing
All sample code contained within this project repository or any subdirectories is licensed according to the terms of the MIT license, which can be viewed in the file [license.txt](/license.txt).


# Open Source @ IBM
[Find more open source projects on the IBM Github Page](http://ibm.github.io/)
