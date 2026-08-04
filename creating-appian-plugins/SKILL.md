---
name: creating-appian-plugins
description: Use when building, scaffolding, or setting up the build for an Appian plug-in — smart services, expression functions, connected systems, integrations, custom data types, servlets, or SAIL component plug-ins — including appian-plugin.xml descriptors, the Appian Plug-in SDK, Gradle setup, and META-INF/lib packaging.
---

# Creating Appian Plug-ins

Appian plug-ins are OSGi bundles. Most are a **JAR** with `appian-plugin.xml` at its root. SAIL UI components are a **different artifact entirely**. Getting the packaging contract wrong produces a JAR that builds cleanly, deploys silently, and does nothing.

The official Appian docs describe an **Eclipse click-through workflow**, not a command-line build. There is no authoritative Gradle recipe published. `references/build.gradle` in this skill is a working one — start from it rather than deriving your own.

## Step 1: Confirm the SDK exists before anything else

**The Maven Central search index does not list the Appian Plug-in SDK.** A groupId search returns only the two `connected-systems-*` artifacts. The SDK is nonetheless published — it is only visible at the direct repository path.

```bash
curl -s https://repo1.maven.org/maven2/com/appian/appian-plug-in-sdk/maven-metadata.xml
```

At the time of writing this returns exactly one version: **26.3**. There is no 25.x or 24.x coordinate. If you need an older SDK, get it from `<APPIAN_HOME>/_admin/sdk/appian-plug-in-sdk.jar` on a server install.

### Compile dependencies by module type

The SDK jar is smaller than it looks: it contains **zero** `javax.servlet` and **zero**
`javax.xml.bind` classes. Those are not optional extras — without them the module does not compile.
All are container-provided at runtime, so all are `compileOnly`.

| Building | Also needs |
|---|---|
| Smart service, expression function | `com.appian:appian-plug-in-sdk:26.3` |
| Servlet | + `javax.servlet:javax.servlet-api:3.1.0` |
| Custom data type | + `javax.xml.bind:jaxb-api:2.3.1` (removed from the JDK in Java 11) |
| Connected system / integration | `com.appian:connected-systems-core:1.10.0` **and** `com.appian:connected-systems-client:1.4.0` |

For connected systems you need **both** artifacts, and the split is counter-intuitive: `-core` holds
only interfaces (`com.appian.connectedsystems.templateframework.sdk.*`), while every `Simple*` base
class you actually extend lives in `-client` (`com.appian.connectedsystems.simplified.sdk.*`).
Despite its name `-client` contains no HTTP client — bring your own and bundle it.

**Never write a stub of an SDK class to make compilation proceed.** A stubbed `AppianServlet` or `AppianSmartService` compiles and then fails at class-load time on the server, because the real class is supplied by the container. If you cannot resolve the SDK, stop and say so — do not route around it.

## Step 2: Pick the artifact

| Building | Descriptor | Root element | Artifact |
|---|---|---|---|
| Smart service, function, servlet, data type, connected system, integration | `appian-plugin.xml` | `<appian-plugin>` | **`.jar`** |
| SAIL UI component | `appian-component-plugin.xml` | `<appian-component-plugin>` | **`.zip`** |

A SAIL component plug-in contains **no Java at all** — HTML, JS and CSS only, in `<rule-name>/v<major>/`. It is not a module you add to `appian-plugin.xml`; there is no `<component>` element there. See `references/component-plugins.md`.

Appian's own guidance: prefer the built-in **web content component** over writing a component plug-in unless you need bi-directional interaction with the interface.

One `appian-plugin.xml` holds **many module types together** — functions, enumerations, smart services, servlets and data types all coexist in a single descriptor and a single JAR. Do not split into multiple plug-ins out of caution.

## Step 3: The packaging contract

```
myplugin.jar
├── appian-plugin.xml          ← MUST be at the JAR root. Absent = silently ignored, no error.
├── com/yourco/...             ← your classes
├── com/yourco/....properties  ← i18n bundles, beside the classes
└── META-INF/lib/*.jar         ← every third-party runtime dependency, as nested JARs
```

- Third-party libraries go in `META-INF/lib/` as **whole JARs**. A flat shaded/fat JAR is not the Appian convention.
- The SDK itself must be `compileOnly` — bundling it duplicates every `com.appiancorp.*` class against the container's own and causes `LinkageError`.
- Bump `<version>` in `appian-plugin.xml` on every release. **Custom data type changes do not take effect without a version bump.**

### What to bundle and what not to

| Packages | Rule |
|---|---|
| `com.appian.*`, `com.appiancorp.*`, `javax.servlet.*`, JDBC drivers, `com.atlassian.*` | Container-provided — **do not bundle** |
| Guava, Apache Commons (io/lang/collections/logging/…), Log4j, SLF4J, DOM4J, JDOM, JSON libs | *Deprecated*-provided — **do bundle**, to pin your own version |
| Everything else (Jackson, Gson, PDFBox, ZXing, HttpClient…) | **Bundle** |

The middle row is the counter-intuitive one and is easy to get backwards. Those libraries are still on the container classpath, but relying on that means your plug-in silently changes behaviour when Appian upgrades its copy.

## Step 4: Compile target

**Java 17.** Not a guess — every class in `appian-plug-in-sdk-26.3.jar` is bytecode major version 61. Exceeding it throws `UnsupportedClassVersionError` at deploy or first invocation. (The two `connected-systems-*` artifacts are major 52 / Java 8, so they impose no additional ceiling.)

If `JAVA_HOME` is a newer JDK, do not fight it — declare a Gradle toolchain. This also catches
accidental use of post-17 APIs, which `sourceCompatibility` alone does not.

```groovy
// build.gradle
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
```

```groovy
// settings.gradle — REQUIRED, not optional
plugins { id 'org.gradle.toolchains.foojay-resolver-convention' version '1.0.0' }
rootProject.name = 'my-appian-plugin'
```

**Gradle cannot download a JDK without a toolchain repository.** Omit the foojay plugin and the build
works only where a JDK 17 already exists — it then fails on a colleague's machine or in CI with
`No matching toolchains found`, and hard-fails everywhere on Gradle 10. Because it succeeds locally,
this is invisible to whoever wrote it. Verify with `--warning-mode all`: if you see
"installed via auto-provisioning without toolchain repositories", the plugin is missing.

Expect ~100 `warning: unknown enum constant Data.K_CONTENT` messages on every compile. The SDK jar
references a class it does not ship. Harmless, unavoidable, not a broken dependency.

## Quick reference — descriptor modules

| Module | Element | Java contract |
|---|---|---|
| Expression function | `<function key= class=/>` | `@Function` method; class annotated with a category annotation |
| Smart service | `<smart-service name= key= class=/>` | extends `AppianSmartService` |
| Dropdown values | `<enumeration key= type=>` + `<items>` | referenced by `@Input(enumeration="key")` |
| Data type | `<datatype key= name=>` + `<class>`/`<package>` | `@XmlType`; **`javax.xml.bind`, not `jakarta.*`** |
| Servlet | `<servlet name= key= class=>` + `<url-pattern>` | extends `AppianServlet` (`javax.servlet`) |
| Connected system | `<connected-system-template>` nesting `<integration-template>` | extends `SimpleConnectedSystemTemplate` / `SimpleIntegrationTemplate` |

Full attribute detail, i18n rules and working code: `references/module-reference.md`, `references/java-patterns.md`.

There is **no `<function-category>` element.** Function categories come from Java annotations (`@Category`, `@AppianScriptingFunctionsCategory`).

## Common mistakes

| Mistake | Result |
|---|---|
| Stubbing an SDK class to get past a resolution failure | Compiles; `NoClassDefFoundError`/`LinkageError` on the server |
| `appian-plugin.xml` under `META-INF/` or in a package dir | Plug-in **silently ignored** — no error anywhere |
| SDK on `implementation` instead of `compileOnly` | SDK bundled into `META-INF/lib`, class conflicts |
| Loose `.class` files instead of nested JARs in `META-INF/lib` | Dependencies not on the bundle classpath |
| Shipping the whole `runtimeClasspath` unfiltered | Compile-only annotation jars (`error_prone_annotations`) bundled |
| Forgetting to bump `<version>` | CDT changes don't take effect; upgrade may be ignored |
| Putting business logic inside the smart service class | Untestable — SDK types cannot be instantiated in unit tests |
| Reusing output fields across `run()` invocations | Stale values leak; instances are reused — reset every output at the top of `run()` |

## Verifying without a server

You usually cannot deploy locally. Verify what you can mechanically. `references/build.gradle`
implements checks 1–3; add 4 and 5 yourself when the plug-in has those module types.

1. `appian-plugin.xml` is at the JAR root and its `<version>` matches the build.
2. No class in the JAR **or in any nested `META-INF/lib` JAR** exceeds bytecode major 61.
3. Neither SDK is present in `META-INF/lib`.
4. The descriptor is well-formed XML and every `class=` attribute names a class actually in the JAR.
5. For a component plug-in: manifest at the ZIP root, and every `rule-name`/`v<major>` folder holds
   its `html-entry-point` and its `_en_US.properties`.

When writing your own verifier tasks in Groovy, make each one assert it examined a non-zero number of
entries. A check that silently matches nothing reports OK and proves nothing — `Set<String>.contains(GString)`
is always false, which is an easy way to build a verifier that can never fail.

After deploying, confirm in the Admin Console plug-ins list, or tail the app server log for
`AppianOsgiPlugin - Successfully installed Plug-in`.

## Known ambiguity — ship both

Function i18n bundles: the docs name the file after the *category class*; working plug-ins in the
field name it after the *function key*. This could not be settled without a live server, so **ship
both filenames with identical contents** rather than guessing and re-deploying:

```
com/yourco/plugin/zipFileSize.properties           + _en_US variant
com/yourco/plugin/ZipFileSizeFunction.properties   + _en_US variant
```

It costs about a kilobyte and removes a coin flip. Both spellings agree that property keys use the
**lowercased** function name while parameter names keep camelCase:

```properties
function.zipfilesize.description=Returns the compressed size in bytes.
function.zipfilesize.param.documentId.description=The document to measure.
```
