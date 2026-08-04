# SAIL component plug-ins — a different artifact

A component plug-in adds a custom UI component to Appian interfaces. It shares almost nothing
with a Java plug-in beyond the word "plug-in".

| | Java plug-in | Component plug-in |
|---|---|---|
| Artifact | `.jar` | **`.zip`** |
| Manifest | `appian-plugin.xml` | **`appian-component-plugin.xml`** |
| Root element | `<appian-plugin>` | **`<appian-component-plugin>`** |
| Contents | Java classes, `META-INF/lib` | HTML, JS, CSS — **no Java** |
| SDK | Appian Plug-in SDK | UI SDK (JS), current version 2.0.0 |

There is no `<component>` element in `appian-plugin.xml`. If you are trying to register a UI
component there, you are building the wrong artifact.

## Before building one

Appian's own guidance is blunt: **"Always use web content unless you have a good reason not to."**
The built-in web content component embeds a URL with no build, no review and no admin install.

Build a component plug-in only when you need **bi-directional interaction** — the component both
receives values from the interface and writes values back. If your design has no `input-output` or
`event` parameters, you probably want web content instead.

Listing on the AppMarket additionally requires passing an Appian review.

## Package structure

```
my-components_1.0.0.zip
├── appian-component-plugin.xml          ← manifest, at the ZIP ROOT
└── shipmentTrackerField/                ← one folder per component rule-name
    └── v1/                              ← one folder per MAJOR version
        ├── index.html                   ← the html-entry-point
        ├── main.js
        ├── styles.css
        ├── images/icon.svg
        └── shipmentTrackerField_en_US.properties
```

- The manifest and every rule-name folder must be at the ZIP root.
- Web content paths are relative to the **version folder** — it is unpacked into its own directory
  on deployment.
- An i18n bundle is required per version folder; a missing one fails the deploy.
- i18n bundles are **stripped at deploy time**. Never reference them from your JS.

## Manifest

```xml
<appian-component-plugin>
  <name>Orbital Freight Components</name>
  <component rule-name="shipmentTrackerField" version="1.0.0">
    <sdk-version>2.0.0</sdk-version>
    <html-entry-point>index.html</html-entry-point>
    <icon-file>images/icon.svg</icon-file>
    <supported-user-agents>
      <user-agent>chrome</user-agent>
      <user-agent>firefox</user-agent>
      <user-agent>edge</user-agent>
      <user-agent>safari</user-agent>
      <user-agent>mobile</user-agent>
    </supported-user-agents>
    <parameter>
      <name>shipmentId</name>
      <type>text</type>
      <category>input-only</category>
      <required>Enter a shipment id.</required>
    </parameter>
  </component>
</appian-component-plugin>
```

Gotchas:

- `<required>` takes **free text** — the validation message shown to the designer — not `true`/`false`.
- `<category>` is one of `input-only`, `input-output`, `event`.
- The `Field` name suffix is a convention, not enforced.

## JavaScript API

Load the SDK with the literal token as the `src`; Appian rewrites it at render time:

```html
<script src="APPIAN_JS_SDK_URI"></script>
```

```js
Appian.Component.onNewValue(function (allParameters) { /* interface -> component */ });
Appian.Component.saveValue('selectedEvent', value);   // component -> interface
Appian.Component.setValidations(['Something is wrong']);
```

## Using it in SAIL

Custom components are called by **bare rule-name**, with no `a!`, `fn!` or `rule!` prefix:

```
shipmentTrackerField(
  shipmentId: ri!shipmentId,
  selectedEventValue: ri!selected,
  selectedEventSaveInto: ri!selected
)
```

(Integrations, by contrast, use the `rule!` domain.)

## Unresolved

Appian's own docs are inconsistent about `event`-category parameters: the manifest reference shows
a single `event` parameter, while the walkthrough calls the same component with a
`…Value` / `…SaveInto` pair, which is the documented shape for `input-output`. If an `event`
parameter does not behave as expected, try the input-output pair shape.

This page was assembled from documentation only — unlike the Java plug-in material in this skill,
it was not validated against a working deployed component. Treat the manifest details as a strong
starting point rather than settled fact.
