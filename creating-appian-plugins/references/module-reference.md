# appian-plugin.xml module reference

Every module below lives in **one** `appian-plugin.xml` at the **root of the plug-in JAR**.
Multiple module types coexist in a single descriptor and a single JAR — there is no need to
split a function, a smart service, a servlet and a data type into separate plug-ins.

If the file is missing or misplaced, Appian ignores the JAR entirely and reports nothing.

## Envelope

```xml
<appian-plugin name="Barcode Utilities" key="com.yourco.barcode.utils">
  <plugin-info>
    <description>Human-readable description</description>
    <vendor name="Your Corporation" url="https://www.example.com"/>
    <version>1.8.0</version>
    <application-version min="26.3"/>
  </plugin-info>
  <!-- module elements here -->
</appian-plugin>
```

| Attribute / element | Rule |
|---|---|
| `key` | Globally unique across all plug-ins on the server. Use Java package naming. The `appian-system` namespace is **reserved**. |
| `name` | Documentation only. |
| `<version>` | Two or three digits (`X.X` / `X.X.X`). **Must increase on every release.** Custom data type changes do not take effect without a bump. |
| `<application-version min=>` | Minimum Appian version. Cannot be earlier than `18.4.0.0`. |

## Expression function

```xml
<function key="readBarCodeFromImage" class="com.yourco.barcode.utils.ReadBarCodeExpression"/>
```

- Attributes are `key` and `class` only.
- **There is no `<function-category>` element.** Categories are Java annotations —
  `@AppianScriptingFunctionsCategory`, `@Category("...")`, or `@HiddenCategory`.
- One class may export several `@Function` methods; register each with its own `<function>`.

## Smart service

```xml
<smart-service name="Generate QR Code" key="GenerateQRCode"
               class="com.yourco.barcode.utils.CreateQRCodeSmartService"/>
```

Display name and icons come from resources beside the class, keyed by the **smart service key**:

```
com/yourco/barcode/utils/GenerateQRCode.properties          -> name=Generate QR Code
com/yourco/barcode/utils/GenerateQRCode_en_US.properties    -> name=Generate QR Code
com/yourco/barcode/utils/GenerateQRCode/images/palette-icon.gif
com/yourco/barcode/utils/GenerateQRCode/images/canvas-icon.gif
```

## Enumeration (dropdown of allowed input values)

```xml
<enumeration key="img-format" type="3">
  <items>
    <item><label>png</label><detail>PNG Format</detail><value>png</value></item>
    <item><label>jpg</label><detail>JPEG Format</detail><value>jpg</value></item>
  </items>
</enumeration>
```

Referenced from a smart service input: `@Input(required = Required.ALWAYS, enumeration = "img-format")`.
`type="3"` is the text type.

## Custom data type

```xml
<datatype key="ProjectDataType" name="Example Project Data Type">
  <class>com.yourco.example.Project</class>
  <class>com.yourco.example.Status</class>
</datatype>
```

- Use `<class>` entries **or** `<package>` entries — reportedly they cannot be mixed.
- The Java class needs `@XmlType`. `@XmlRootElement` / `@XmlAccessorType` / `propOrder` are common but not required.
- **Use `javax.xml.bind.annotation`, not `jakarta.*`.** The jakarta namespace is not supported.
- The XML namespace must be a valid URI and must not be `appian`.
- Declare `<datatype>` before smart services or functions that consume the type.
- Changing a CDT requires a plug-in `<version>` bump or the change is ignored.

## Servlet

```xml
<servlet name="Example Servlet" key="exampleServlet" class="com.yourco.servlet.ExampleServlet">
  <description>An example servlet</description>
  <url-pattern>/example/*</url-pattern>
  <init-param>
    <param-name>foo</param-name>
    <param-value>bar</param-value>
  </init-param>
</servlet>
```

- Class extends `com.appiancorp.suiteapi.servlet.AppianServlet`, which extends
  `javax.servlet.http.HttpServlet` — **`javax`, not `jakarta`**.
- Multiple `<url-pattern>` elements allowed; `*` and `?` are wildcards.
- Reachable at `<context>/plugins/servlet/<pattern>`.
- Patterns under `/plugins/servlet/stateless/*` are treated as stateless and use HTTP Basic auth.
- `javax.servlet-api` is container-provided — keep it `compileOnly`.

## Connected system and integration

```xml
<connected-system-template key="HelloWorldConnectedSystem"
                           name="HelloWorldConnectedSystem"
                           class="com.yourco.templates.HelloWorldConnectedSystemTemplate">
  <integration-template key="HelloWorldIntegrationTemplate"
                        name="HelloWorldIntegrationTemplate"
                        class="com.yourco.templates.HelloWorldIntegrationTemplate"/>
  <client-api key="HelloWorldClientApi"
              name="HelloWorldClientApi"
              class="com.yourco.templates.HelloWorldClientApi"/>
</connected-system-template>
```

| Element | Base class | Artifact |
|---|---|---|
| `connected-system-template` | `SimpleConnectedSystemTemplate` | connected-systems-**client** |
| `connected-system-template` (with Test Connection button) | `SimpleTestableConnectedSystemTemplate` | connected-systems-**client** |
| `integration-template` | `SimpleIntegrationTemplate` | connected-systems-**client** |
| `client-api` | `SimpleClientApi` | connected-systems-**client** |

### Which artifact holds what — verified by unzipping the published JARs

This trips people up, and guessing wrong means imports that never resolve:

| Artifact | Package | Contents |
|---|---|---|
| `connected-systems-core:1.10.0` | `com.appian.connectedsystems.templateframework.sdk.*` | The **interfaces** `ConnectedSystemTemplate`, `IntegrationTemplate`, plus `ExecutionContext`, `IntegrationResponse`, `IntegrationError`, `ClientApi`. No `Simple*` class anywhere. |
| `connected-systems-client:1.4.0` | `com.appian.connectedsystems.simplified.sdk.*` | All nine `Simple*` / `ConfigurableTemplate` / `SimpleConfiguration` classes — i.e. everything you actually extend. |

You need **both** on the compile classpath. Note the package prefixes differ (`templateframework` vs
`simplified`) — they are not interchangeable.

`connected-systems-client` contains **no HTTP client** despite the name; all nine of its classes are
the simplified template framework. Bring your own HTTP library and bundle it.

Both artifacts are compiled to bytecode major 52 (Java 8), unlike `appian-plug-in-sdk` which is major 61.

- `<integration-template>` and `<client-api>` are optional and nest **inside** the connected system.
- `@TemplateId` is required on the template classes.
- `@IntegrationTemplateType(READ | WRITE | READ_AND_WRITE)` declares the operation kind.
- `majorVersion` on `@TemplateId` defaults to `1`; omitting it is safe. Setting it on an *integration*
  template is reported to fail upload. Whether the same applies to a connected system template is unverified.

### Branding and i18n

```
src/main/resources/com/yourco/MyConnectedSystemTemplate_40px.png
src/main/resources/com/yourco/MyConnectedSystemTemplate_50px.png
src/main/resources/com/yourco/MyConnectedSystemTemplate_80px.png
src/main/resources/resources_en_US.properties   -> keys like MyConnectedSystemTemplate.name
```

## Deployment

- Drop the JAR in `APPIAN_INSTALL/_admin/plugins/`.
- Hot deploy polls at `conf.plugins.poll-interval` seconds; `0` disables it and requires a restart.
- Repeated deploy timeouts at startup will **fail application server startup** by design.
- Using `EncryptionService` requires per-plug-in authorisation in the Admin Console.
- Confirm success in Admin Console → plug-ins, or in the log:
  `AppianOsgiPlugin - Successfully installed Plug-in '<name>' version <x>`

## Verified vs reported

Directly confirmed against a working production plug-in and current Appian docs: the envelope,
`<function>`, `<smart-service>`, `<enumeration>`, `<datatype>`, `<servlet>`, `<connected-system-template>`,
the `META-INF/lib` contract, and deployment mechanics.

Reported by research but not confirmed here — verify before relying on them: the `<class>`/`<package>`
mixing restriction, the `majorVersion` upload failure, the stateless servlet path convention, and the
`jakarta` incompatibility. Each is plausible and consistent, but was not exercised against a live server.
