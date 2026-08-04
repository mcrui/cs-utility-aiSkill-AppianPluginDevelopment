# Changelog

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
