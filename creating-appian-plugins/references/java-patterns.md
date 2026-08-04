# Java patterns for Appian plug-in modules

All patterns below are taken from a plug-in running in production on Appian 26.3.

## Smart service

```java
package com.yourco.barcode.utils;

import com.appiancorp.suiteapi.common.Name;
import com.appiancorp.suiteapi.content.ContentService;
import com.appiancorp.suiteapi.knowledge.DocumentDataType;
import com.appiancorp.suiteapi.knowledge.FolderDataType;
import com.appiancorp.suiteapi.process.exceptions.SmartServiceException;
import com.appiancorp.suiteapi.process.framework.AppianSmartService;
import com.appiancorp.suiteapi.process.framework.Input;
import com.appiancorp.suiteapi.process.framework.MessageContainer;
import com.appiancorp.suiteapi.process.framework.Required;
import com.appiancorp.suiteapi.process.palette.PaletteInfo;

@PaletteInfo(paletteCategory = "Appian Smart Services", palette = "Document Management")
public class ReadQRCodesFromPDFSmartService extends AppianSmartService {

  private final ContentService contentService;

  // Appian injects available services through the constructor.
  public ReadQRCodesFromPDFSmartService(ContentService contentService) {
    super();
    this.contentService = contentService;
  }

  private Long pdfDocument;
  private Long[] pageNumbers = new Long[0];
  private String errorText = "";
  private Boolean wasSuccessful = Boolean.FALSE;

  @Override
  public void run() throws SmartServiceException {
    // Instances are reused. Reset EVERY output, not just the success flag —
    // otherwise a run that fails early leaks the previous run's results.
    pageNumbers = new Long[0];
    errorText = "";
    wasSuccessful = Boolean.FALSE;
    try {
      // ... delegate to a plain, unit-testable class ...
      wasSuccessful = Boolean.TRUE;
    } catch (Exception e) {
      // Reporting through errorText rather than throwing keeps the process node
      // from failing. Throw SmartServiceException only if the node SHOULD fail.
      errorText = e.getMessage() == null ? e.toString() : e.getMessage();
    }
  }

  // Both are required by the framework even when empty.
  public void onSave(MessageContainer messages) { }
  public void validate(MessageContainer messages) { }

  // INPUTS are setters.
  @Input(required = Required.ALWAYS)
  @Name("pdfDocument")
  @DocumentDataType
  public void setPdfDocument(Long val) { this.pdfDocument = val; }

  @Input(required = Required.OPTIONAL, defaultValue = "200")
  @Name("dpi")
  public void setDpi(Long val) { /* defaultValue applies in the designer, not here —
                                   still null-guard for programmatic callers */ }

  // OUTPUTS are getters.
  @Name("pageNumbers")
  public Long[] getPageNumbers() { return pageNumbers; }

  @Name("wasSuccessful")
  public Boolean getWasSuccessful() { return wasSuccessful; }
}
```

Key points:

- `@DocumentDataType` / `@FolderDataType` mark a `Long` as an Appian Document / Folder id.
- `@Input(enumeration = "img-format")` binds an input to an `<enumeration>` in the descriptor.
- `@PaletteInfo` controls where the node appears in the process modeller palette.
- Appian number inputs map to `Long`, not `Integer`.

### Reading and writing documents with ContentService

This is the part most document-handling smart services need and the part that is hardest to guess.

```java
import com.appiancorp.suiteapi.content.ContentConstants;
import com.appiancorp.suiteapi.content.ContentService;
import com.appiancorp.suiteapi.content.ContentUploadOutputStream;
import com.appiancorp.suiteapi.knowledge.Document;

// READ an existing document
Document source = contentService.download(documentId, ContentConstants.VERSION_CURRENT, false)[0];
try (InputStream in = source.getInputStream()) {
  // ...
}

// CREATE a new document in a folder
Document doc = new Document();
doc.setName(baseName);        // NO extension here
doc.setExtension("zip");      // extension is a SEPARATE field
doc.setDescription("Generated archive");
doc.setParent(folderId);
doc.setSize(0);
doc.setId(null);
ContentUploadOutputStream out = contentService.uploadDocument(doc, ContentConstants.UNIQUE_NONE);
// ... write bytes to out ...
out.close();                  // close() COMMITS the version — it is not just cleanup
Long newDocumentId = out.getContentId();   // valid only after close()
```

Two traps worth stating outright:

- **Name and extension are stored separately.** `getName()` returns `invoice`, not `invoice.pdf`.
  Rebuild the display filename yourself when you need it.
- **`close()` is the commit.** A helper that writes into the upload stream must **not** close it —
  otherwise ownership of the commit is ambiguous and `getContentId()` may be read too early. Pass the
  stream in, let the caller close it, and consider testing that your helper leaves it open.

`ContentService.setSizeOfDocumentVersion(Long)` is deprecated in SDK 26.3 and unnecessary — `close()`
already records the size. Older plug-ins call it; new ones should not.

### Testability

**SDK types cannot be instantiated in unit tests.** Anything placed in the smart service class is
effectively untestable. Keep the class as a thin adapter: read inputs, delegate to a plain class,
map results to outputs. Test the plain class.

```groovy
// build.gradle — tests need the SDK on the test compile classpath
testImplementation "com.appian:appian-plug-in-sdk:${appianSdkVersion}"
```

## Expression function

```java
package com.yourco.barcode.utils;

import com.appiancorp.suiteapi.content.ContentService;
import com.appiancorp.suiteapi.expression.annotations.AppianScriptingFunctionsCategory;
import com.appiancorp.suiteapi.expression.annotations.Function;
import com.appiancorp.suiteapi.expression.annotations.Parameter;
import com.appiancorp.suiteapi.knowledge.DocumentDataType;

@AppianScriptingFunctionsCategory
public class ReadBarCodeExpression {

  // Injected services come FIRST and are NOT annotated with @Parameter.
  // Everything after them is a visible function argument.
  @Function
  public String readBarCodeFromImage(ContentService cs,
                                     @Parameter @DocumentDataType Long img,
                                     @Parameter(required = false) String errorTxt) {
    return "...";
  }
}
```

Register in the descriptor: `<function key="readBarCodeFromImage" class="...ReadBarCodeExpression"/>`

### i18n keys

Property keys use the **lowercased** function name regardless of the Java method's camelCase:

```properties
function.readbarcodefromimage.description=Reads a bar code from an image.
function.readbarcodefromimage.param.img.description=Photo of a Bar Code or QR Code
function.readbarcodefromimage.param.errorTxt.description=Custom error text
```

Note the mismatch: the function name is lowercased but **parameter names keep their camelCase**.

Filename ambiguity: Appian docs say the bundle is named after the category class
(`ExampleCategory_en_US.properties`); the production plug-in this was drawn from names it after the
function key (`readBarCodeFromImage.properties` and `readBarCodeFromImage_en_US.properties`) and
works. If descriptions do not appear in the expression editor, try the other filename before
suspecting the keys.

## Servlet

```java
import com.appiancorp.suiteapi.servlet.AppianServlet;   // extends javax.servlet.http.HttpServlet

public class ExampleServlet extends AppianServlet {
  private String foo;

  // AppianServlet exposes only a no-arg init(). There is NO init(ServletConfig)
  // overload to override — read <init-param> values through getInitParameter().
  @Override
  public void init() {
    this.foo = getInitParameter("foo");
  }

  @Override
  protected void doGet(HttpServletRequest req, HttpServletResponse resp) { }
}
```

Use `javax.servlet.*`, never `jakarta.servlet.*`.

**`javax.servlet-api` is a required compile dependency**, not optional — the Appian SDK jar contains
zero `javax/servlet` entries, so a servlet module does not compile without it. Keep it `compileOnly`;
the container supplies it at runtime.

Testing caveat: merely *naming* a class that extends `AppianServlet` in a test drags `javax.servlet`
onto the test compile classpath, because javac resolves the whole supertype chain. Rather than adding
servlet-api to `testImplementation`, move the real logic into an SDK-free class and test that — the
same thin-adapter rule that applies to smart services.

## Custom data type

**Add the JAXB dependency or this will not compile.** `javax.xml.bind` was removed from the JDK in
Java 11 and the Appian SDK jar contains zero `javax/xml/bind` entries, so on the Java 17 toolchain the
annotation below looks like a JDK builtin but is not on the classpath:

```groovy
compileOnly 'javax.xml.bind:jaxb-api:2.3.1'
```

```java
import javax.xml.bind.annotation.XmlType;              // javax, NOT jakarta

@XmlType(namespace = "http://yourco.com/appian/types", propOrder = {"id", "name"})
public class Project {
  private Long id;
  private String name;
  public Long getId() { return id; }
  public void setId(Long id) { this.id = id; }
  public String getName() { return name; }
  public void setName(String name) { this.name = name; }
}
```

`propOrder` is optional but stops field order shuffling between builds. Namespace must be a valid
URI and must not be `appian`. Every CDT change needs a plug-in `<version>` bump to take effect.

## Connected system and integration

These base classes are **abstract**; an empty body will not compile. The signatures below were
recovered from the published sources JAR.

Imports come from two different artifacts and two different package prefixes:

```java
// connected-systems-client  ->  com.appian.connectedsystems.simplified.sdk.*
import com.appian.connectedsystems.simplified.sdk.SimpleConnectedSystemTemplate;
import com.appian.connectedsystems.simplified.sdk.SimpleIntegrationTemplate;
import com.appian.connectedsystems.simplified.sdk.configuration.SimpleConfiguration;
import com.appian.connectedsystems.simplified.sdk.connectiontesting.SimpleTestableConnectedSystemTemplate;
// connected-systems-core    ->  com.appian.connectedsystems.templateframework.sdk.*
import com.appian.connectedsystems.templateframework.sdk.ExecutionContext;
import com.appian.connectedsystems.templateframework.sdk.IntegrationResponse;
import com.appian.connectedsystems.templateframework.sdk.configuration.PropertyPath;
import com.appian.connectedsystems.templateframework.sdk.connectiontesting.TestConnectionResult;
```

```java
@TemplateId(name = "com.yourco.orbital.OrbitalConnectedSystem")
public class OrbitalConnectedSystemTemplate extends SimpleTestableConnectedSystemTemplate {

  // Declares the fields the designer fills in on the connected system.
  @Override
  protected SimpleConfiguration getConfiguration(SimpleConfiguration configuration,
                                                 ExecutionContext executionContext) {
    return configuration.setProperties(
        textProperty("baseUrl").label("Base URL").isRequired(true).build(),
        encryptedTextProperty("apiKey").label("API Key").isRequired(true).build());
  }

  // Only present because we extend SimpleTestableConnectedSystemTemplate.
  // Without that base class there is no Test Connection button in the designer.
  @Override
  protected TestConnectionResult testConnection(SimpleConfiguration connectedSystemConfiguration,
                                                ExecutionContext executionContext) {
    return TestConnectionResult.success();
  }
}
```

```java
@TemplateId(name = "com.yourco.orbital.GetShipment")
@IntegrationTemplateType(IntegrationTemplateRequestPolicy.READ)
public class GetShipmentIntegrationTemplate extends SimpleIntegrationTemplate {

  // NOTE THE PARAMETER ORDER: integration config FIRST, connected system SECOND.
  // Reversing them compiles cleanly and then reads every value off the wrong
  // object at runtime — baseUrl comes back null.
  @Override
  protected SimpleConfiguration getConfiguration(SimpleConfiguration integrationConfiguration,
                                                 SimpleConfiguration connectedSystemConfiguration,
                                                 PropertyPath updatedProperty,
                                                 ExecutionContext executionContext) {
    return integrationConfiguration.setProperties(
        // isExpressionable(true) is what lets a designer pass a rule input here.
        textProperty("shipmentId").label("Shipment Id").isExpressionable(true).isRequired(true).build());
  }

  @Override
  protected IntegrationResponse execute(SimpleConfiguration integrationConfiguration,
                                        SimpleConfiguration connectedSystemConfiguration,
                                        ExecutionContext executionContext) {
    String baseUrl = connectedSystemConfiguration.getValue("baseUrl");
    String shipmentId = integrationConfiguration.getValue("shipmentId");
    Map<String, Object> result = new HashMap<>();
    // ... call the API ...
    return IntegrationResponse.forSuccess(result).build();
  }
}
```

Property builders (`textProperty`, `encryptedTextProperty`, `booleanProperty`, `dropdownProperty`)
come from `ConfigurableTemplate`, the shared superclass. Failure paths use
`IntegrationError.builder()` fed into `IntegrationResponse.forError(...)`.

`@TemplateId` is required. `majorVersion` defaults to `1` — leave it unset on integration templates.

## Logging

The SDK ships the Log4j 1.2 API bridge, so existing plug-ins commonly use:

```java
private static final Logger LOG = Logger.getLogger(MyClass.class);   // org.apache.log4j
```

Declare `org.apache.logging.log4j:log4j-1.2-api` as `compileOnly` if you follow that pattern.
