# Changelog

## 1.1.0 — 2026-08-27

### Added

- **Kiro steering file** at `.kiro/steering/creating-appian-plugins.md`. Uses `manual` inclusion
  so the guidance is available via `#` in chat when needed. References the same
  `creating-appian-plugins/` content via `#[[file:...]]` includes.
- Kiro installation section in `README.md` with three options: work in this repo directly, copy
  `.kiro/` to another project, or use inline `#[[file:...]]` references.
- Steering inclusion mode reference table in `README.md` (always / fileMatch / manual).

### Changed

- `README.md` header updated from "Claude skill" to "AI skill" — the content is AI-agnostic and
  now works with both Claude Code (as a skill) and Kiro (as a steering file).
- Layout section in `README.md` now shows the `.kiro/steering/` directory.

## 1.0.1 — 2026-08-27

Improvements derived from reviewing three production plug-ins: `cs-plugin-userPasswordTools`,
`ps-plugin-barcodeAndQRcode`, and `ps-plugin-EncryptionFunctions`.

### Corrected

- **`<function-category>` element exists** — the skill previously stated "There is no
  `<function-category>` element." This was wrong. The element is used in production with an
  `<i18n-bundle>` child for category label i18n. Categories can come from either the descriptor
  element or Java annotations; both are valid. Fixed in `SKILL.md`, `module-reference.md`.

### Added — java-patterns.md

- **Additional smart service annotations**: `@UserDataType`, `@PasswordDataType`, `@Order` (input
  display order in the designer), `@Unattended` (system-only smart services).
- **SmartServiceException.Builder** pattern with `.userMessage()` referencing i18n property keys —
  the alternative to catch-and-report when the node should fail with a user-facing message.
- **Smart service signature freezing**: documented `IncompatibleSmartServiceRegistrationException`
  and the reflection-based signature test pattern to prevent accidental parameter changes after
  first deploy.
- **Injectable services table**: `ContentService`, `SecureCredentialsStore`, `UserService`,
  `TypeService`, `ServiceContext` — with constructor injection (smart services) and method parameter
  injection (expression functions) examples.
- **Custom function category annotation**: `@Category` meta-annotation pattern for reusable
  categories across multiple function classes.
- **ContentService.create() + getOutputStream()**: alternative to `uploadDocument()` for streaming
  transforms (encryption, compression).
- **Testability patterns**: the stripped SDK problem, the `libs/` unstripped SDK workaround for
  tests, interface seams for library boundaries, and OSGi static initializer classloader traps.

### Added — module-reference.md

- **`<function-category>` element** with `<i18n-bundle>` child and relationship to `@Category`.
- **`<i18n-bundle>` child** on `<function>` and `<smart-service>` elements.
- **Smart service i18n key reference**: `input.<Name>.displayName`, `input.<Name>.comment`,
  `output.<Name>.displayName`, `output.<Name>.comment`, `error.<key>` patterns.

### Added — SKILL.md

- Signature freezing row in the Common Mistakes table.
- Function category row in the Quick Reference table.

### Added — build.gradle

- `verifySmartServiceBundles` task (commented out by default) — asserts every `<smart-service>` key
  has both `.properties` and `_en_US.properties` beside its class.

### Findings from the production plug-ins

- `@Order` annotation on smart service classes controls input display order in the Appian designer —
  without it, order is undefined and may shuffle between deploys.
- Appian permanently pins smart service parameter count to `<package>.<key>` at first deploy. A
  version bump does not lift this. The `userPasswordTools` plug-in carries a signature-freezing test
  that catches this at build time.
- The published SDK's `Required` enum has a `<clinit>` that references a constructor not in the JAR.
  Reading any `@Input` annotation in tests throws `NoSuchMethodError`. The signature test works
  around this by deriving parameters from JavaBean method shape instead.
- OSGi classloader isolation means `ImageIO.scanForPlugins()` must run with the plug-in's own
  classloader as TCCL, or bundled image format plugins are invisible. The barcode plug-in handles
  this in a static initializer with broad exception catching.
- The EncryptionFunctions plug-in keeps an older unstripped SDK in `libs/` for test execution — a
  pragmatic workaround for the stripped-SDK testing problem.

## 1.0.0 — 2026-08-04

Initial release of the `creating-appian-plugins` skill.

### Added

- `SKILL.md` — artifact decision table (JAR vs component ZIP), SDK acquisition, compile dependencies
  per module type, packaging contract, bundle/don't-bundle rules, Java 17 toolchain setup, common
  mistakes, and server-free verification checklist.
- `references/build.gradle` — working Gradle build, verified end to end against Gradle 9.6.1 with
  `JAVA_HOME` on JDK 25. Three guard tasks: bytecode ceiling (including inside nested
  `META-INF/lib` JARs), descriptor placement/version/SDK-leak, and descriptor `class=` resolution.
- `references/module-reference.md` — every `appian-plugin.xml` module element with attributes,
  plus deployment mechanics.
- `references/java-patterns.md` — working code for smart services, functions, servlets, custom data
  types and connected systems, including `ContentService` document read/write.
- `references/component-plugins.md` — the SAIL component ZIP artifact.

### Findings that shaped the content

Derived from baseline runs by agents working without the skill:

- `com.appian:appian-plug-in-sdk` **does not appear in the Maven Central search index** but resolves
  at the direct repository path. Exactly one version is published: `26.3`. An agent that searched by
  groupId concluded the SDK was unobtainable and stubbed `AppianServlet` — undeployable.
- Apache Commons / Guava / Log4j / SLF4J are *deprecated*-provided: they should be **bundled** to pin
  a version, which is the opposite of the intuitive reading.
- There is no `<function-category>` element; function categories are Java annotations.
- The SDK jar contains **zero** `javax.servlet` and **zero** `javax.xml.bind` entries, so servlets and
  custom data types each need an extra `compileOnly` dependency to compile at all.

### Corrected during review

- `SimpleConnectedSystemTemplate` / `SimpleIntegrationTemplate` are in **`connected-systems-client`**,
  not `connected-systems-core`. `-core` holds only interfaces. `-client` contains no HTTP client
  despite its name. Verified by unzipping both artifacts.
- The Gradle toolchain does **not** auto-download a JDK without the
  `org.gradle.toolchains.foojay-resolver-convention` plugin in `settings.gradle`. Omitting it produces
  a build that works only where a JDK 17 already exists and hard-fails on Gradle 10.
- `SimpleIntegrationTemplate` methods take `integrationConfiguration` **first**,
  `connectedSystemConfiguration` second — reversing them compiles and silently reads the wrong object.
- Appian stores a document's name and extension in **separate fields**, and `close()` on
  `ContentUploadOutputStream` is what commits the version.
