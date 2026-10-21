# Build-logic, the version catalog and plugin loading for the G1 build skeleton

- **Issue:** [SpartanLaboratories/SpartanGraphics#3](https://github.com/SpartanLaboratories/SpartanGraphics/issues/3), question 1 of the
  [research README](README.md).
- **Read:** 2026-10-20. Gradle sources are pinned to tag `v9.8.2` (released 2026-10-19), Dokka to `v2.2.0`, vanniktech to `0.37.0`,
  Ktor to `3.6.0`, kotlinx-io to `0.9.1`, Okio to `parent-3.18.2`, Coil to `3.6.3`, Now in Android to `a49ed253d75e`, the JetBrains
  multiplatform-library-template to `226096493b`, MyGameTools to `dc5214f` (`v5.1.0-98`).
- **Method limit:** nothing was built or run. Every behaviour below is established from Gradle's documentation and source, from the
  source of the comparable builds, and from two measurements of published artifacts (Maven Central jars and class-file metadata). The
  wiring has not been executed in this repository; see *Known gaps* for the checks the implementation must run.

## 1. Question

How should `build-logic` and `gradle/libs.versions.toml` be structured, and how must the build load its plugins, so that (a) every
library module applies one convention plugin and declares little else, (b) the root's Dokka aggregate sees the same Kotlin plugin
classes as the modules, (c) a consumer's `includeBuild("../SpartanGraphics")` from the JVM client (Kotlin Gradle plugin 2.1.20,
Gradle 9.0.0 wrapper) or from a KMP library substitutes `graphics-core`, `graphics-desktop` and `graphics-bom` without substitution
rules, with `build-logic` nested in this build, and (d) the build works under the configuration cache, has no cross-project
configuration that would block Isolated Projects, and reads as a model for SpartanUI.

## 2. Bottom line

- **Precompiled script plugins, with the logic that needs testing or types in plain `.kt` files beside them.** Three public plugin ids,
  prefixed by the repository: `spartangraphics.kmp-library`, `spartangraphics.jvm-library`, `spartangraphics.publishing`. This is
  Gradle's documented convention-plugin form and the form of Ktor and kotlinx-io. Binary plugins (Okio, Now in Android) buy
  testability this build can get from `.kt` helpers, at the price of losing typed `kotlin {}` accessors.
- **One catalog, `gradle/libs.versions.toml`, imported into `build-logic` explicitly** (`from(files("../gradle/libs.versions.toml"))`
  in `build-logic/settings.gradle.kts`). Plugin artifacts reach `build-logic` through the catalog's `[plugins]` aliases, converted to
  marker coordinates by the helper Gradle's own documentation shows. Convention plugins read the catalog through
  `VersionCatalogsExtension`, wrapped once in a small `Catalog.kt`. The generated-accessor workaround (Ktor) and every native
  alternative are rejected: gradle/gradle#15383 is still open on 9.8.2, and Gradle's documentation still prescribes
  `VersionCatalogsExtension` for precompiled scripts.
- **The Dokka fix is expressed as the root `plugins {}` block, `alias(libs.plugins.…) apply false` for Kotlin (jvm and multiplatform)
  and vanniktech, plus `alias(libs.plugins.dokka)` applied.** The same four catalog aliases feed both the root block and
  `build-logic/build.gradle.kts`, so the root and `build-logic` can never load different versions. Dokka, vanniktech and Kotlin all
  register or rely on shared state (Dokka's `KotlinBasePlugin` lookup, build services in Dokka and vanniktech), so all must come from
  one classloader. A settings-level hoist cannot be used because catalog plugins are unavailable in settings. The same remedy
  (there with versions in settings rather than the catalog), at the same Dokka version, turned MyGameTools' published `gametools-core` javadoc jar from 0 HTML pages (5.0.0) into 415 (5.1.0).
- **`pluginManagement { includeBuild("build-logic") }`, not a top-level `includeBuild`.** It is Gradle's documented form, it makes
  `build-logic` a plugin build that offers no dependency substitutions (nothing leaks into a consumer's composite), and nested
  `build-logic` directories coexist under hierarchical build paths.
- **Shared values:** `group` and `version` in the root `gradle.properties` (Gradle assigns them to every project, root and BOM
  included, before any script runs, and an included build reads its own file); artifact-resolved versions, including the Kotlin
  floor `kotlin-floor = "2.2.0"` from which the language/API floor is derived, in the catalog; non-artifact policy (JVM level 21,
  test-level names) as typed constants in `build-logic`. Gradle's best-practice page says not to store non-library constants in a
  catalog.
- **The largest unverified composite risk:** a consumer's Gradle compiles `build-logic` with *its* embedded Kotlin. The client's
  wrapper (9.0.0, embedded Kotlin 2.2.0, and also vanniktech 0.37.0's exact minimum) is two minors behind Kotlin 2.4.21's
  `kotlin-stdlib`, whose class metadata is 2.4. Whether that breaks the nested `build-logic` compile on 9.0.0 is not established;
  it must be tested against the client's wrapper before G1 closes (Considerations, C1).

## 3. Findings

### F1. Convention-plugin form: precompiled script plugins, plus `.kt` helpers (common practice)

- Gradle's current best-practice guidance puts convention plugins in an included `build-logic` rather than `buildSrc`, and says
  "Unlike binary plugins, convention plugins allow accessing plugin extensions, tasks and configurations via static accessors in
  build scripts written in Kotlin." ([best_practices_structuring_builds.adoc, v9.8.2](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/best-practices/best_practices_structuring_builds.adoc))
- The plugin id of a precompiled script is its file name (plus package, if declared). A script without a `package` and named
  `spartangraphics.kmp-library.gradle.kts` is the id `spartangraphics.kmp-library`
  ([implementing_gradle_plugins_precompiled.adoc, v9.8.2](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/plugin-development/implementing_gradle_plugins_precompiled.adoc)).
- Ktor 3.6.0 mixes both: `ktorbuild.*.gradle.kts` scripts hold the DSL, and `.kt` files hold typed code. Ktor's `Accessors.kt` states
  why: "Gradle doesn't generate accessors for plugins defined in this module, and we can't use generated accessors from .kt files,
  so we have to declare them manually." So a helper `.kt` function takes the extension type as a parameter
  (`KotlinMultiplatformExtension`), and the script passes `kotlin`.
- `kotlin-dsl` compiles `build-logic` at Kotlin language and API version 2.2 on every Gradle 9.0 to 9.8
  (`KotlinDslCompilerPlugins` in v9.8.2: `apiVersion.set(KotlinVersion.KOTLIN_2_2)`). `build-logic` sources therefore use no newer
  language feature, whatever Kotlin Gradle plugin the modules use. `kotlin-dsl` also sets `JvmTargetValidationMode.WARNING`
  "as this plugin is frequently used without toolchains", so `build-logic` declares no toolchain (and a composite consumer needs no
  JDK 21 provisioning for it).

### F2. The catalog inside `build-logic` (consensus)

- **Import.** A plugin build does not see the main build's catalog. Gradle documents the import for `buildSrc`; it is identical for
  `build-logic`: `dependencyResolutionManagement { versionCatalogs { create("libs") { from(files("../gradle/libs.versions.toml")) } } }`
  in `build-logic/settings.gradle.kts` ([version_catalogs.adoc, v9.8.2](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/dependency-management/centralizing-dependencies/version_catalogs.adoc)).
  Kotlinx-io 0.9.1, Okio parent-3.18.2 and Now in Android all do exactly this.
- **In `build-logic/build.gradle.kts`** the typed accessors work (`libs.plugins.…`). Plugin marker coordinates come from a `[plugins]`
  alias through the helper Gradle's own sample defines:
  `fun DependencyHandlerScope.plugin(plugin: Provider<PluginDependency>) = plugin.map { "${it.pluginId}:${it.pluginId}.gradle.plugin:${it.version}" }`.
  The alternative, a second `[libraries]` entry for `org.jetbrains.kotlin:kotlin-gradle-plugin` (Ktor, kotlinx-io, Now in Android),
  duplicates every version reference; here the root block needs the `[plugins]` alias anyway.
- **Inside precompiled scripts** there are no generated `libs` accessors, and `plugins {}` in a precompiled script cannot read the
  catalog. Gradle 9.8.2's documentation: "However, the plugins block in the precompiled script plugin cannot access the version
  catalog", and for the body it prescribes `extensions.getByType(VersionCatalogsExtension::class.java).named("libs")` with
  `findLibrary(...)`/`findVersion(...)`. gradle/gradle#15383 ("Make generated type-safe version catalogs accessors accessible from
  precompiled script plugins") is **open**, last active 2026-06-22; its comments name three workarounds (the generated-class hack, the
  string API, and the third-party `typesafe-conventions-gradle-plugin`, 0.12.1 on 2026-10-18).
- **The generated-accessor hack** (Ktor: `implementation(files(libs.javaClass.superclass.protectionDomain.codeSource.location))` in
  `build-logic/build.gradle.kts`, then `rootProject.the<LibrariesForLibs>()`) works but depends on the internal generated class
  `org.gradle.accessors.dm.LibrariesForLibs` and gives no accessor inside `plugins {}`. Ktor has dozens of lookups; this build has
  about ten. The hack is not adopted. **Common practice** for small builds is the `VersionCatalogsExtension` wrapper
  (Now in Android `ProjectExtensions.kt`: `val Project.libs get(): VersionCatalog = extensions.getByType<VersionCatalogsExtension>().named("libs")`).
- **Nothing native in 9.x changes this.** 9.5 added typed accessors for precompiled *Settings* plugins, not catalog access.

### F3. Plugin classloading, the Dokka failure and where the versions live (consensus on the remedy; the mechanism is partly inferred)

**What fails.** Dokka's `KotlinAdapter` finds Kotlin through `KotlinBasePlugin`. When the class cannot be loaded from Dokka's own
classloader it does **not fail**; it logs a warning whose text (Dokka v2.2.0, `KotlinAdapter.logKgpClassNotFoundWarning`) is:

> The most common cause is a Gradle limitation: the plugins applied to subprojects should be consistent. … 1. Apply the Dokka and
> Kotlin plugins to the root project using the `plugins {}` DSL. (If the root project does not need the plugins, use 'apply false')
> 2. Remove the Dokka and Kotlin plugins versions in the subprojects.

and the module renders no pages. A green build therefore proves nothing: the build needs its own guard (question 4).

**Evidence that the remedy works, measured.** MyGameTools 5.0.0 loaded Dokka at the root (versioned through `pluginManagement`)
and Kotlin only in `build-logic`'s per-module scope. `https://repo1.maven.org/maven2/io/github/spartanlabsgaming/gametools-core/5.0.0/gametools-core-5.0.0-javadoc.jar`
is 261 bytes and holds 0 `.html` entries. Commit `235810e` ("apply Kotlin & Dokka from the root so module Dokka renders", an ancestor of
`v5.1.0`) added `id("org.jetbrains.kotlin.jvm") apply false` to the root `plugins {}`; `gametools-core-5.1.0-javadoc.jar` is
1,690,968 bytes and holds 415 `.html` entries (listed with `unzip -l` on 2026-10-20). Gradle 9.7.1, Dokka 2.2.0, Kotlin 2.2.0,
`build-logic` included at the top level of settings.

**Why it works, and how far that is documented.** Gradle documents the remedy, not the classloader hierarchy: "In the root/parent
`build.gradle(.kts)`, plugins are declared but not applied (via `apply false`). This approach, while optional, makes the plugins
available to be explicitly applied in specific subprojects" ([plugins_intermediate.adoc, v9.8.2](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/fundamentals/authoring-builds/plugins_intermediate.adoc)).
The Isolated Projects page adds that a plugin declared with `apply false` in the **settings** `plugins {}` block "exports the
classpath into the settings classloader scope — the parent of every project's scope". The hierarchy itself is the subject of
gradle/gradle#25616 ("Rework plugin classloader hierarchy", open; its comments cite exactly KGP/AGP and "plugins try to talk to each
other") and #35117 (documentation gap, open). The reading used here, consistent with all of the above: a plugin already present in
an ancestor project's scope is found there when a child's `plugins {}` or a precompiled script requests it, and its classes are
resolved parent-first, so every project, the root included, shares one copy. Whichever project introduces the plugin first, in a
*sibling* scope, splits the copy. This reading is inferred; it is not stated in Gradle's documentation.

**Shared state makes one copy mandatory, not cosmetic.** Dokka 2.2.0 (`PluginFeaturesService`) and vanniktech 0.37.0
(`MavenCentralBuildService`) both register build services (class files checked in the published jars). Gradle's own issue
gradle/gradle#17559 ("Build services can only be registered if the set of applied plugins is the same on all projects", open) is the
resulting hazard. Gradle's best-practice page makes the same point about `build-logic` classpaths ("Using different plugin
combinations can cause features like Build Services to break in difficult to diagnose ways"). So the root hoists **all three** plugin
families, not only Kotlin: MyGameTools hoisted Kotlin and Dokka but not vanniktech and worked only because every module gets the
identical `build-logic` classpath.

**Where the versions live when they come from a catalog.**

| Hoist point | Can read `libs`? | Verdict |
| --- | --- | --- |
| Settings `plugins { … apply false }` (Gradle's strongest hoist) | No: "You cannot use a plugin declared in a version catalog in your settings file or settings plugin." | Rejected: the versions would leave the catalog. |
| Settings `pluginManagement { plugins { id(…) version … } }` (MyGameTools) | No | Rejected for the same reason. |
| Root `build.gradle.kts` `plugins { alias(libs.plugins.x) apply false }` | Yes | **Chosen.** Gradle documents it; used by Now in Android, JetBrains' multiplatform-library-template and Gradle's own samples. |
| Root applies a plugin *from `build-logic`* so that `build-logic`'s whole runtime classpath enters the root scope (Ktor `ktorbuild.root`, kotlinx-io `id("kotlinx-io-dokka")`) | n/a | Works in production (Ktor 3.6.0 aggregates Dokka 2.2.0 this way) but hides the purpose of the line and needs a root-only convention plugin. Not chosen; it is the fallback if the chosen form misbehaves. |
| Root `buildscript { classpath(libs…) }` (Okio, Coil) | Yes | Legacy form; Gradle's table marks the `buildscript` block "Legacy". |

Now in Android states the rationale in the root script: "By listing all the plugins used throughout all subprojects in the root
project build script, it ensures that the build script classpath remains the same for all projects. This avoids potential problems
with mismatching versions of transitive plugin dependencies. A subproject that applies an unlisted plugin will have that plugin and
its dependencies _appended_ to the classpath, not replacing pre-existing dependencies."

**The version rule that keeps this safe.** A plugin requested *with a version* while it is already on a parent classpath fails
("The request for this plugin could not be satisfied because the plugin is already on the classpath with an unknown version",
gradle/gradle#20084, closed with the answer "This is a known limitation of the `plugins` block"). Therefore versions appear in
exactly one place per plugin, the root block (through the catalog alias); no module script and no convention plugin ever names
a version. This is Dokka's remedy step 2.

### F4. `pluginManagement { includeBuild(…) }` versus a top-level `includeBuild` (common practice; semantic difference verified in Gradle's tests)

- Gradle's convention-plugin sample is `pluginManagement { includeBuild("build-logic") }`; so are kotlinx-io and Now in Android.
  Top-level is used by Ktor, Okio and MyGameTools.
- A `pluginManagement` build is a *plugin build*: Gradle's test `PluginBuildConfigurationIntegrationTest` (v9.8.2) has
  `"included plugin build can not have dependency substitutions"` (`dependencySubstitution {}` inside it fails with "Could not find
  method dependencySubstitution()"), and `"included project plugin build can not clash with root subproject name"`. A top-level
  `includeBuild("build-logic")` is an ordinary library build: its root project is a candidate for dependency substitution under
  `unspecified:build-logic`, and `CompositeBuildDependencySubstitutions` keys substitutions by `group:name` only, raising "Module
  version … is not unique in composite" when two builds provide the same module and something requests it. Two builds in a composite
  that each include their own top-level `build-logic` provide the same module id. Nothing requests it today, but the plugin-build form
  removes the possibility.
- The only functional reason for the top-level form is a `build-logic` that must also be a library. This build has none.
- A settings plugin (the second reason to use `pluginManagement`) is not needed here. Gradle's guidance for settings plugins in a
  `build-logic` is to put them in a separate minimal build; the foojay resolver stays a plain `plugins {}` entry in
  `settings.gradle.kts`.

### F5. Where shared values live (common practice; the property mechanism is verified in Gradle's source)

- **`group` and `version` in the root `gradle.properties`.** Gradle's `ProjectPropertySettingBuildLoader` (v9.8.2) walks every project
  of the build after settings and calls `project.setVersion(...)` / `project.setGroup(...)` from the `version` and `group`
  properties. That reaches the root project and the BOM without an `allprojects {}` block or a plugin, and before any script reads
  `project.group`. It is what kotlinx-io 0.9.1 does (`group=org.jetbrains.kotlinx`, `version=0.10.0-SNAPSHOT`). Okio and Coil use
  `GROUP`/`VERSION_NAME` plus `allprojects {}`, which is cross-project configuration. WebTools' `allprojects { group = …; version = … }`
  is the same shape.
- **Catalog `[versions]`: everything resolved as an artifact**, including the Kotlin floor `kotlin-floor = "2.2.0"` (the stdlib pin).
  The language/API floor `2.2` is derived from that one string in `Catalog.kt`, so the stdlib pin and `languageVersion`/`apiVersion`
  cannot drift apart.
- **Typed constants in `build-logic`: policy that is not an artifact.** The JVM level (`21`) and the test-level names. Gradle's
  best-practice page: "Don’t use them to store shared strings or non-library constants". (kotlinx-io does read a `java` version from
  its catalog, so this is practice, not unanimity; Ktor keeps its JDK level as a constant.)
- A user-level `~/.gradle/gradle.properties` has higher precedence than the project's file; a stray `version=` there overrides the
  repository's. CI and a release branch can override with `-Pversion=…`.

### F6. What a consumer's `includeBuild("../SpartanGraphics")` needs (consensus from Gradle's documentation and tests)

- **Substitution without rules.** "Gradle will inspect the group and name for the projects in the included build and substitute
  project dependencies for any external dependency matching `${project.group}:${project.name}`." Setting `group` on every project and
  keeping folder = project name = artifactId satisfies it. Substitution keys on `group:name` only (no version), so any requested
  version of `io.github.spartanlaboratories:graphics-core` becomes the project, including the BOM's constraints. Per-target
  coordinates (`graphics-core-jvm`, `graphics-core-js`) are not project names and are never substituted; consumers must depend on the
  root coordinates (question 5 owns the BOM's treatment of them). Explicit rules are needed only for the cases Gradle lists
  (custom `archivesBaseName`, non-default configurations, publication coordinates that differ from `group:name`); none applies.
- **Root project name.** `rootProject.name = "SpartanGraphics"` only names the build (the root project itself publishes nothing). It
  must not equal a project name in the consumer ("An included build path cannot conflict with any existing project path in the root
  build"); `includeBuild("../SpartanGraphics") { name = "…" }` is the escape hatch.
- **Nested `build-logic`.** Gradle's prose says nested composites are flattened and build paths derive from the directory name; its
  tests say otherwise for paths. `CompositeBuildBuildPathAssignmentIntegrationTest` ("can have buildLogic build and include build with
  buildLogic build", "build paths are selected based on the directory hierarchy") asserts the paths `:includedBuild:buildLogic` and
  `:buildLogic` side by side. So SpartanUI's own `build-logic` and this repository's coexist. The test fixtures include the nested
  `buildLogic` at the top level, which exercises the same path assignment as a `pluginManagement` include. Another test in the
  suite, `"nested build can contribute to build script classpath"`, covers a plugin build that itself includes a library build.
  Not found in Gradle's tests: a `pluginManagement`-included build inside an included build, exactly as here (Known gaps).
- **Isolation.** "Included builds do not share any configuration with the root build or other included builds"; repositories, plugin
  management and version catalogs are per build, so `libs` of SpartanGraphics and of the consumer never meet. "`gradle.properties`
  files defined in the root build are not visible to included builds. Each included build must define its own properties": this
  repository's `group`/`version` come from its own file. Gradle runtime properties (`org.gradle.configuration-cache`,
  `org.gradle.parallel`, …) are read **only from the root build** of the invocation, so this repository's choices apply to its own
  invocations and not to a consumer's. `-P` properties from the command line propagate to included builds, so `-PtestLevel` passed
  in a consumer's invocation reaches this build's test tasks and an unknown level there would fail configuration.
- **Gradle version.** The included build runs on the consumer's Gradle. Applied plugin minimums that bind it: vanniktech 0.37.0 lists
  "JDK 17, Gradle 9.0.0, Android Gradle Plugin 8.13.0, Kotlin Gradle Plugin 2.2.0" as its minimum supported versions (the client's
  9.0.0 wrapper sits exactly on it); the Kotlin Gradle plugin 2.4.20–2.4.21 documents Gradle 7.6.3–9.7.0. (Dokka 2.2.0's minimum was
  not found; question 2 owns the wrapper choice.)
- **`build-logic` is compiled by the consumer's Gradle with the consumer's embedded Kotlin.** Embedded Kotlin per Gradle:
  9.0.0 → 2.2.0, 9.7.0 → 2.4.0, 9.8.0 → 2.4.10 (compatibility.adoc, v9.8.2). Class-file metadata read from the published jars with
  `javap`: `kotlin-gradle-plugin-api-2.4.21` (`KotlinBasePlugin`) `mv = 1.8`; Dokka Gradle plugin 2.2.0 `mv = 1.4`; vanniktech 0.37.0
  `mv = 2.0`; `kotlin-stdlib-2.4.21` (`Unit`) `mv = 2.4`. Plugin APIs are readable by any embedded compiler; the open question is whether
  `kotlin-stdlib` 2.4.x enters `build-logic`'s compile classpath (KGP's own module file lists no direct stdlib dependency; its
  transitive dependencies were not resolved) and a 2.2.0 compiler then rejects it. `embedded-kotlin` adds the embedded
  stdlib as `compileOnly` and Gradle warns when versions differ. See C1.
- **Two Kotlin Gradle plugins in one composite** (client 2.1.20, this build 2.4.21) load in separate build scopes. A JVM client
  consuming the KMP module resolves variants by attributes on the substituted project's outgoing configurations (`jvmApiElements`,
  `TargetJvmVersion = 21` from the reused findings); producer attributes the consumer does not request are not matched against.
  Community threads report that KMP→KMP composite builds build correctly while IDE import has problems (KT-30285, KTIJ-18903);
  that is one-source opinion and not verified here.
- **Toolchains.** `jvmToolchain(21)` in the modules needs a JDK 21 on the consumer's machine or a resolver in SpartanGraphics' own
  settings (foojay, as in every sibling build). Whether an included build's resolver is honoured when the consumer's settings
  declare none was not established.
- **A build with a `js { nodejs() }` target must not set `repositoriesMode` to `FAIL_ON_PROJECT_REPOS`/`PREFER_SETTINGS`** without
  checking: the Kotlin/JS plugin adds Node.js and Yarn distribution repositories to projects (Kotlin Slack threads; one-source opinion).
  Now in Android uses `FAIL_ON_PROJECT_REPOS` with no JS target. The default mode is kept.

### F7. Configuration cache and Isolated Projects (consensus)

- Convention plugins that read the catalog, `providers.gradleProperty("testLevel")` or `project.group`/`version` at apply time are
  configuration-cache inputs; nothing needs `Project` at execution time. Dokka aggregation through `dependencies { dokka(project(…)) }`
  is a resolvable-configuration dependency, not cross-project access.
- Isolated Projects is incubating since Gradle 9.7 (`org.gradle.isolated-projects=true`); "prefer refactoring cross-project
  configuration into convention plugins". `allprojects {}`/`subprojects {}` (WebTools, Coil, Okio) are the constructs it rejects; here
  they are replaced by convention plugins (applied explicitly per module) and `gradle.properties` (`group`/`version`). File
  references from a convention to the root go through `project.isolated.rootProject.projectDirectory` (MyGameTools'
  `gametools.published-library` does this for Dokka assets).
- Plugin-side compatibility of Kotlin Gradle plugin 2.4.21, Dokka 2.2.0 and vanniktech 0.37.0 with Isolated Projects was not checked;
  the design only guarantees that this build adds no violation of its own.

### F8. Plugin ids

- Sibling convention: MyGameTools `gametools.kotlin-library` / `gametools.published-library` (repository-based). Ktor `ktorbuild.*`,
  kotlinx-io `kotlinx-io-*`, Now in Android `nowinandroid.*`. A repository-specific prefix avoids two builds in one composite
  defining the same id. `spartangraphics.<role>` follows MyGameTools.
- Now in Android registers its convention ids in its catalog as version-less `[plugins]` entries and applies them with `alias(...)`
  (typo-safe ids at the cost of a second declaration per id). Not adopted: with precompiled scripts the file name is the id.

## 4. How comparable projects do it

| Project @ ref | `build-logic` included | Convention form | Catalog inside the plugin build | Plugin deps of the plugin build | Root plugin hoist | `group`/`version` |
| --- | --- | --- | --- | --- | --- | --- |
| MyGameTools @ `dc5214f` | top-level `includeBuild("build-logic")` | precompiled scripts | none; literal GAV strings | markers, literal versions | root `plugins { id("org.jetbrains.kotlin.jvm") apply false … }`, versions in settings `pluginManagement` | per-module `coordinates(...)` |
| GeneralTools @ `d424aaf` | none (single module) | none | none | n/a | n/a | in the build file |
| WebTools @ `main` (local) | none | `allprojects`/`subprojects` in the root | none | n/a | `kotlin("jvm") version … apply false` | `allprojects { group; version }` |
| [Ktor 3.6.0](https://github.com/ktorio/ktor/tree/3.6.0/build-logic) | top-level; settings plugins in `pluginManagement { includeBuild("build-settings-logic") }` | precompiled `ktorbuild.*` + `.kt` | `ktorsettings` settings plugin imports it | library coordinates (`libs.kotlin.gradlePlugin`, `libs.dokka.gradlePlugin`, `libs.mavenPublishing`) + `LibrariesForLibs` hack | root applies `ktorbuild.root`; the `ktor-dokka` project applies `ktorbuild.dokka` (no `apply false`) | `version = resolveVersion()` in `ktorbuild.base` (from a `VERSION` file) |
| [kotlinx-io 0.9.1](https://github.com/Kotlin/kotlinx-io/tree/0.9.1/build-logic) | `pluginManagement { includeBuild("build-logic") }` | precompiled `kotlinx-io-*` | `from(files("../gradle/libs.versions.toml"))` | library coordinates; `findVersion("java")` via `VersionCatalogsExtension` | root `plugins { id("kotlinx-io-publish") apply false; id("kotlinx-io-dokka") }` | `gradle.properties` |
| [Okio parent-3.18.2](https://github.com/square/okio/tree/parent-3.18.2/build-support) | top-level `includeBuild("build-support")` | binary plugin class `BuildSupport` | same `from(files(...))` | library coordinates | root `buildscript { classpath(libs…) }` | `allprojects` from `GROUP`/`VERSION_NAME` |
| [Coil 3.6.3](https://github.com/coil-kt/coil/tree/3.6.3) | `buildSrc` | binary | n/a | n/a | root `buildscript { classpath(libs…) }`; root applies Dokka (comment cites gradle#20084) | `allprojects { group; version }` |
| [Now in Android @ `a49ed253d75e`](https://github.com/android/nowinandroid/tree/a49ed253d75e) | `pluginManagement { includeBuild("build-logic") }` | binary, ids from catalog | `from(files(...))` | `compileOnly(libs.*.gradlePlugin)` | root `alias(libs.plugins.…) apply false` for every plugin, with the rationale quoted in F3 | module scripts |
| [multiplatform-library-template @ `226096493b`](https://github.com/Kotlin/multiplatform-library-template) | none | none | n/a | n/a | root `alias(libs.plugins.…) apply false` | in the module script |

SQLDelight and Arrow were not examined.

## 5. Pitfalls and regrets

- **Silent empty documentation.** Dokka warns and renders nothing; MyGameTools shipped 5.0.0 javadoc jars with no pages and
  discovered it a release later. The check belongs in the build (question 4/6): fail on the warning text
  `Dokka could not load KotlinBasePlugin`, or assert a page count.
- **A plugin with a version in a child script** (or in a convention plugin) when it is already on a parent classpath fails or
  splits the copy. Versions only in the root block.
- **Partial hoisting.** Hoisting Kotlin and Dokka but not vanniktech works only while every module sees one identical `build-logic`
  classpath; any module or future module that adds an unlisted plugin gets it "appended" (Now in Android's note). When AGP arrives in
  the Android phase it must be added to the root block with the others; AGP 9.4 also needs Gradle 9.6 or newer (reused findings).
- **`libs` where it does not exist.** Not in precompiled `plugins {}`, not in `settings.gradle.kts` plugins. The foojay resolver
  therefore keeps a literal version in `settings.gradle.kts`; it is the single version outside the catalog, and it is a settings
  plugin, not a dependency.
- **Top-level `includeBuild("build-logic")`** lets the plugin build register substitutions and can clash with a consumer project or
  build of the same name; `pluginManagement` avoids both. Never name a module `build-logic`.
- **`allprojects {}`/`subprojects {}`** (WebTools, Coil, Okio): the construct Isolated Projects rejects, and Gradle's documentation
  calls it improper for sharing build logic.
- **Accessors in `.kt` helpers.** Typed accessors exist only inside precompiled scripts (Ktor's comment above). Keep scripts thin and
  helpers typed by extension class.
- **Applying `kotlin("jvm")` to `build-logic` itself.** `kotlin-dsl` already applies Gradle's embedded Kotlin plugin and warns on a
  second version (`EmbeddedKotlinPlugin.warnOnDifferentKotlinVersion`). Do not apply KGP 2.4.21 to `build-logic`; it is only a
  dependency.
- **`kotlin-dsl` language level.** `build-logic` sources are Kotlin 2.2 regardless of the modules' Kotlin; do not assume 2.3/2.4
  language features there.
- **Runtime properties in `gradle.properties`** are ignored when this build is included; a CI or developer habit of relying on
  `org.gradle.configuration-cache=true` from this file does not carry into a consumer's invocation.
- **`-PtestLevel` is global in a composite invocation.** Parse it lazily and name the failure so a consumer's invocation that passes
  a level this build does not know fails with a clear message.
- **User-level `gradle.properties` overriding `version`.**

## 6. Recommendation for this repository

Use the precompiled-script layout below. The reasoning, in three sentences: Gradle's own documentation, Now in Android, the
JetBrains template and the MyGameTools measurement all agree that a root `plugins {} … apply false` block is what puts Kotlin,
Dokka and vanniktech in one classloader, and it is the one hoist point that can read the catalog. Feeding that block and
`build-logic` from the same four catalog aliases makes divergence impossible, and `pluginManagement { includeBuild }` keeps
`build-logic` a plugin-only build that a consumer's composite cannot trip over. Ktor's root-convention-plugin form and Ktor's
accessor hack are both rejected: the first obscures the purpose of the root block, the second leans on a generated internal class
for about ten lookups.

### 6.1 File layout

```
SpartanGraphics/
  settings.gradle.kts
  build.gradle.kts                      # root: hoisted plugins + Dokka aggregate, nothing else
  gradle.properties                     # group, version, runtime flags
  gradle/libs.versions.toml
  graphics-bom/build.gradle.kts         # question 5
  graphics-core/build.gradle.kts        # applies spartangraphics.kmp-library
  graphics-desktop/build.gradle.kts     # applies spartangraphics.jvm-library
  build-logic/
    settings.gradle.kts
    build.gradle.kts
    src/main/kotlin/
      spartangraphics.kmp-library.gradle.kts
      spartangraphics.jvm-library.gradle.kts
      spartangraphics.publishing.gradle.kts
      com/spartanlabs/graphics/build/
        Catalog.kt                      # the one stringly-typed seam
        SpartanBuild.kt                 # JVM level, test-level names
        TestLevels.kt                   # pure parser + Project.configureTestLevels()
    src/test/kotlin/com/spartanlabs/testing/<level>/...   # question 6
```

### 6.2 `settings.gradle.kts`

```kotlin
pluginManagement {
    // A plugin build: contributes plugins by id, offers no dependency substitutions to this build or to a consumer.
    includeBuild("build-logic")
}

plugins {
    // Settings plugins cannot read the version catalog, so this is the one version outside libs.versions.toml.
    id("org.gradle.toolchains.foojay-resolver-convention") version "1.0.0"
}

dependencyResolutionManagement {
    repositories { mavenCentral() }   // default repositoriesMode: the Kotlin/JS plugin adds project repositories
}

rootProject.name = "SpartanGraphics"

include("graphics-bom", "graphics-core", "graphics-desktop")
```

`gradle/libs.versions.toml` is imported as `libs` by convention; no declaration here. Folder = project name = artifactId.

### 6.3 `gradle.properties` (identity and runtime flags; Dokka flags belong to question 4)

```properties
# Project identity: Gradle assigns these to every project (root and BOM included) before any script runs.
group=io.github.spartanlaboratories
version=0.1.0-SNAPSHOT

org.gradle.configuration-cache=true
org.gradle.parallel=true
org.gradle.caching=true
kotlin.code.style=official
```

### 6.4 `gradle/libs.versions.toml` (only the entries question 1 depends on; question 2 fixes the values and adds the rest)

```toml
[versions]
kotlin = "2.4.21"          # the compiler/plugin
kotlin-floor = "2.2.0"     # consumer floor: stdlib pin; language/API version is derived from it
dokka = "2.2.0"
vanniktech-publish = "0.37.0"

[libraries]
kotlin-stdlib-floor = { module = "org.jetbrains.kotlin:kotlin-stdlib", version.ref = "kotlin-floor" }   # pin mechanism: question 2

[plugins]
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
kotlin-jvm           = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
dokka                = { id = "org.jetbrains.dokka", version.ref = "dokka" }
vanniktech-publish   = { id = "com.vanniktech.maven.publish", version.ref = "vanniktech-publish" }
```

### 6.5 Root `build.gradle.kts`

```kotlin
plugins {
    // Hoist every plugin the modules use into the ROOT classpath so Gradle shares ONE copy with every project. The conventions in
    // build-logic apply these by id with no version. Without this, Dokka cannot load KotlinBasePlugin (it logs a warning and
    // renders an empty site), and the build services of Dokka and vanniktech get registered from several classloaders.
    // Gradle #25616, #17559; Dokka's own warning prescribes this block. The two Kotlin ids are one jar; both are listed for
    // clarity. A new plugin used by any module (AGP in the Android phase) is added here, with its version in the catalog.
    alias(libs.plugins.kotlin.multiplatform) apply false
    alias(libs.plugins.kotlin.jvm) apply false
    alias(libs.plugins.vanniktech.publish) apply false
    alias(libs.plugins.dokka)
}

dokka { moduleName = "SpartanGraphics" }

dependencies {
    dokka(project(":graphics-core"))
    dokka(project(":graphics-desktop"))
}
```

### 6.6 `build-logic/settings.gradle.kts`

```kotlin
dependencyResolutionManagement {
    repositories {
        mavenCentral()
        gradlePluginPortal()
    }
    versionCatalogs {
        create("libs") { from(files("../gradle/libs.versions.toml")) }   // a plugin build does not inherit the catalog
    }
}

rootProject.name = "build-logic"
```

### 6.7 `build-logic/build.gradle.kts`

```kotlin
plugins { `kotlin-dsl` }   // supplies Gradle's embedded Kotlin; do not apply kotlin("jvm") here

dependencies {
    // Same catalog aliases as the root block: root and build-logic cannot disagree on a version.
    implementation(plugin(libs.plugins.kotlin.multiplatform))
    implementation(plugin(libs.plugins.kotlin.jvm))
    implementation(plugin(libs.plugins.dokka))
    implementation(plugin(libs.plugins.vanniktech.publish))
}

// Plugin-marker coordinates for a catalog [plugins] alias (the helper Gradle's catalog documentation shows).
fun DependencyHandlerScope.plugin(plugin: Provider<PluginDependency>): Provider<String> =
    plugin.map { "${it.pluginId}:${it.pluginId}.gradle.plugin:${it.version.requiredVersion}" }
```

No toolchain in `build-logic`: `kotlin-dsl` does not require one and a composite consumer then needs no JDK 21 for it.

### 6.8 `build-logic/src/main/kotlin/com/spartanlabs/graphics/build/`

```kotlin
// Catalog.kt -- the only place a catalog alias is a string.
package com.spartanlabs.graphics.build

import org.gradle.api.GradleException
import org.gradle.api.Project
import org.gradle.api.artifacts.MinimalExternalModuleDependency
import org.gradle.api.artifacts.VersionCatalog
import org.gradle.api.artifacts.VersionCatalogsExtension
import org.gradle.api.provider.Provider
import org.gradle.kotlin.dsl.getByType
import org.jetbrains.kotlin.gradle.dsl.KotlinVersion

internal val Project.catalog: VersionCatalog
    get() = extensions.getByType<VersionCatalogsExtension>().named("libs")

internal fun VersionCatalog.version(alias: String): String =
    findVersion(alias).orElseThrow { GradleException("gradle/libs.versions.toml has no [versions] entry '$alias'") }.requiredVersion

internal fun VersionCatalog.library(alias: String): Provider<MinimalExternalModuleDependency> =
    findLibrary(alias).orElseThrow { GradleException("gradle/libs.versions.toml has no [libraries] entry '$alias'") }

/** The language and API version of the consumer floor: `kotlin-floor` "2.2.0" -> 2.2, so the stdlib pin and the floor cannot drift. */
internal fun VersionCatalog.kotlinFloor(): KotlinVersion =
    KotlinVersion.fromVersion(version("kotlin-floor").substringBeforeLast('.'))
```

```kotlin
// SpartanBuild.kt -- policy that is not an artifact; typed, not in the catalog.
package com.spartanlabs.graphics.build

internal object SpartanBuild {
    const val JVM_LEVEL = 21
    val TEST_LEVELS = listOf("gating", "component", "integration", "deterministic", "e2e", "nonfunctional", "uat")
}
```

`TestLevels.kt` holds the pure `parseTestLevels(raw, known)` (GeneralTools' `parseTestLevels`, moved out of the build script so
Level 2 / 4a tests can reach it) and `fun Project.configureTestLevels()`, which reads `providers.gradleProperty("testLevel")` and
adds the package filter to every `AbstractTestTask`. Reused: `kmp-test-levels-and-names.md`.

### 6.9 Convention plugins (shape; the Kotlin/ABI/Dokka/BOM details are questions 2 to 5)

```kotlin
// spartangraphics.kmp-library.gradle.kts
import com.spartanlabs.graphics.build.*
import org.jetbrains.kotlin.gradle.dsl.JvmTarget

plugins {
    id("org.jetbrains.kotlin.multiplatform")      // no version: already in the root classpath
    id("spartangraphics.publishing")
}

kotlin {
    explicitApi()
    jvmToolchain(SpartanBuild.JVM_LEVEL)
    compilerOptions {
        languageVersion = catalog.kotlinFloor()
        apiVersion = catalog.kotlinFloor()
    }
    jvm { compilerOptions { jvmTarget = JvmTarget.fromTarget(SpartanBuild.JVM_LEVEL.toString()) } }
    js { nodejs() }
}

configureTestLevels()
```

```kotlin
// spartangraphics.jvm-library.gradle.kts
plugins {
    `java-library`                                // `api` configuration
    id("org.jetbrains.kotlin.jvm")
    id("spartangraphics.publishing")
}
// explicitApi, floor, toolchain, ABI, configureTestLevels(): same helpers as above.
```

```kotlin
// spartangraphics.publishing.gradle.kts
plugins {
    id("com.vanniktech.maven.publish")
    id("org.jetbrains.dokka")
}

mavenPublishing {
    publishToMavenCentral()
    signAllPublications()
    // artifactId = project name = folder, group/version from gradle.properties: the same coordinates a consumer's
    // composite derives, so substitution needs no rules.
    coordinates(group.toString(), name, version.toString())
    pom { /* licence, developers, scm: GeneralTools' block */ }
}
```

The publishing convention must also be applicable to `graphics-bom` (no Kotlin plugin, `java-platform`): it must not assume Kotlin and
configures the Dokka javadoc jar only for Kotlin projects (for example `plugins.withId("org.jetbrains.kotlin.multiplatform") { … }`).
Questions 4 and 5 own that wiring.

Module scripts then reduce to `plugins { id("spartangraphics.kmp-library") }`, the POM `name`/`description`, and dependencies
(`api(project(":graphics-core"))` in `graphics-desktop`).

### 6.10 What the implementation must verify (the wiring was not executed here)

1. `./gradlew :graphics-core:dokkaGeneratePublicationHtml --rerun-tasks` prints no `Dokka could not load KotlinBasePlugin`; run it
   both with and without the root `alias(libs.plugins.dokka)`, and with an extra throwaway module, to see the guard bite.
2. `./gradlew build --configuration-cache` stores and then reuses the entry; a diagnostic run with `--isolated-projects` reports
   nothing from this build's own scripts.
3. A composite smoke test: a throwaway copy of `GameGraphics` with `includeBuild("../SpartanGraphics")` and a dependency on
   `io.github.spartanlaboratories:graphics-desktop:0.1.0-SNAPSHOT`, **on the wrapper the client has today (9.0.0)**, then on the
   wrapper chosen for this repository.

## 7. Known gaps

- **No execution.** The recommended wiring (hoisting through catalog aliases, `pluginManagement` plus a nested `build-logic`) is
  established by documentation, Gradle's tests, sibling builds and one measured MyGameTools outcome, but the exact form was not run.
  MyGameTools used a top-level include and settings-level versions, not this combination. Section 6.10 lists the three checks.
- **Composite on Gradle 9.0.0.** Unknown whether `build-logic` compiles there with Kotlin Gradle plugin 2.4.21 on its classpath
  (embedded Kotlin 2.2.0, `kotlin-stdlib` 2.4.21 metadata `2.4`; KGP API classes `1.8`). Resolution of KGP's transitive stdlib was
  not performed. The same check also covers Dokka 2.2.0 and vanniktech 0.37.0 starting on 9.0.0.
- **No Gradle test was found** for a `pluginManagement` plugin build declared inside an included build, only for top-level nested
  builds and for nested plugin-build chains. The behaviour follows from the documented model and from one test's
  "nested build can contribute to build script classpath".
- **The classloader mechanism** (parent-first resolution from an ancestor project's scope) is inferred; Gradle documents the remedy
  and the settings-scope fact only.
- **Toolchain resolvers in included builds** (foojay in this build's settings versus the consumer's) were not established.
- **KMP in composite builds:** only community reports (KT-30285, KTIJ-18903; IDE import), none verified; variant selection between a
  Kotlin 2.1.20 JVM consumer and a 2.4.21 KMP producer is expected, not tested.
- **Dokka 2.2.0's minimum Gradle**, and Isolated Projects compatibility of KGP 2.4.21, Dokka 2.2.0 and vanniktech 0.37.0, were not
  checked.
- **Kotlin/JS repositories** versus `repositoriesMode`: a one-source report; the chosen default mode avoids the question but the root
  project's Dokka dependency resolution with Kotlin/JS-added project repositories should be seen working in CI.
- SQLDelight and Arrow `build-logic` were not read.
- **Reused findings:** nothing contradicts GeneralTools' `kmp-build-and-kotlin-compat.md`. Its F14 (no catalog for one module) stands
  for one module; the multi-module case here differs because of the shared root and `build-logic`.

## 8. Considerations

**C1. Minimum Gradle version for composite consumers.** The build runs on the consumer's Gradle, and `build-logic` is compiled by that
Gradle's embedded Kotlin. The client's wrapper is 9.0.0 (embedded Kotlin 2.2.0, vanniktech 0.37.0's exact minimum); Kotlin Gradle plugin
2.4.21 is documented up to Gradle 9.7.0.
- *(a)* State a consumer floor equal to the repository's wrapper line (9.7 or newer, embedded Kotlin 2.4) in `CONTRIBUTING.md`, and
  bump the client's wrapper when it first includes this build.
- *(b)* Keep 9.0.0 consumers working, adding `-Xskip-metadata-version-check` to `build-logic` or constraining its stdlib, if the
  smoke test of 6.10 (3) shows an error.
- *(c)* Test first, then choose (a) or (b).
Recommendation: (c), defaulting to (a) because (b) adds a compiler-flag workaround to a build meant to be a model. Unresolved until the
smoke test runs.

## 9. Load-bearing claims

Quotes from AsciiDoc or Java/Kotlin sources omit markup characters only.

1. **A `pluginManagement` include is a plugin build, and a plugin build cannot declare dependency substitutions.**
   Gradle test `PluginBuildConfigurationIntegrationTest`: `def "included plugin build can not have dependency substitutions"` …
   `failureCauseContains("Could not find method dependencySubstitution()")`.
   [Source](https://github.com/gradle/gradle/blob/v9.8.2/subprojects/composite-builds/src/integTest/groovy/org/gradle/integtests/composite/plugins/PluginBuildConfigurationIntegrationTest.groovy), Gradle v9.8.2.
2. **Default substitution key is `group:name`.** "Gradle will inspect the group and name for the projects in the included build and
   substitute project dependencies for any external dependency matching `${project.group}:${project.name}`."
   [composite_builds.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/structuring-builds/composite_builds.adoc), v9.8.2; rendered at https://docs.gradle.org/current/userguide/composite_builds.html.
3. **Substitution ignores versions and is ambiguous only when requested.** `CompositeBuildDependencySubstitutions`:
   `replacementMap.put(replacement.getLeft().getModule(), replacement.getRight())` and
   "Module version '%s' is not unique in composite: can be provided by %s."
   [Source](https://github.com/gradle/gradle/blob/v9.8.2/subprojects/composite-builds/src/main/java/org/gradle/composite/internal/CompositeBuildDependencySubstitutions.java), v9.8.2.
4. **Nested `build-logic` directories coexist; build paths follow the directory hierarchy.**
   `CompositeBuildBuildPathAssignmentIntegrationTest`, `def "can have buildLogic build and include build with buildLogic build"`:
   `assignedBuildPaths ==~ [ ':includedBuild', ':includedBuild:buildLogic', ':', ':buildLogic' ]`.
   [Source](https://github.com/gradle/gradle/blob/v9.8.2/subprojects/composite-builds/src/integTest/groovy/org/gradle/integtests/composite/CompositeBuildBuildPathAssignmentIntegrationTest.groovy), v9.8.2.
   The prose in composite_builds.adoc ("When a composite build includes another composite, Gradle flattens the structure.") does not match
   the paths; the test is taken as authoritative.
5. **Included builds share no configuration and read their own `gradle.properties`.** "Included builds do not share any
   configuration with the root build or other included builds." and "`gradle.properties` files defined in the root build are not
   visible to included builds. Each included build must define its own properties or properties must be passed via the CLI." and
   "Gradle runtime properties (such as `org.gradle.configuration-cache`) are a special case — they are only read from the *root build*
   and applied to the entire invocation."
   [composite_builds.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/structuring-builds/composite_builds.adoc), v9.8.2.
6. **`group` and `version` in `gradle.properties` are applied to every project at build load.**
   `ProjectPropertySettingBuildLoader`: `String versionName = "version"; Object versionValue = projectGradleProperties.findUnsafe(versionName); if (versionValue != null) { project.setVersion(versionValue);`
   (and the same for `"group"` with `project.setGroup(groupValue)`), applied recursively through `for (ProjectState childProject : project.getChildProjects()) { setProjectProperties(access, childProject); }`.
   [Source](https://github.com/gradle/gradle/blob/v9.8.2/subprojects/core/src/main/java/org/gradle/initialization/ProjectPropertySettingBuildLoader.java), v9.8.2.
7. **A plugin build does not inherit the catalog and must import it; precompiled `plugins {}` cannot read it; precompiled bodies use
   `VersionCatalogsExtension`.** "`buildSrc` does not automatically inherit the version catalog from the main project, so additional
   configuration is required." and "However, the plugins block in the precompiled script plugin cannot access the version catalog."
   and "In precompiled script plugins inside `buildSrc`, the version catalog can be accessed using `extensions.getByType(VersionCatalogsExtension)`".
   The sample's `settings.gradle.kts` is `create("libs") { from(files("../gradle/libs.versions.toml")) }`.
   [version_catalogs.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/dependency-management/centralizing-dependencies/version_catalogs.adoc) and
   [sample settings](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/snippets/reference/dependency-management/centralizing-dependencies/catalogs-toml-buildSrc/kotlin/buildSrc/settings.gradle.kts), v9.8.2.
8. **The marker-coordinate helper is Gradle's documented idiom.** Sample `buildSrc/build.gradle.kts`:
   `fun DependencyHandlerScope.plugin(plugin: Provider<PluginDependency>) = plugin.map { "${it.pluginId}:${it.pluginId}.gradle.plugin:${it.version}" }`.
   [Source](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/snippets/reference/dependency-management/centralizing-dependencies/catalogs-toml-buildSrc/kotlin/buildSrc/build.gradle.kts), v9.8.2.
9. **Catalog plugins cannot be used in settings.** "You cannot use a plugin declared in a version catalog in your settings file or
   settings plugin." [version_catalogs.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/dependency-management/centralizing-dependencies/version_catalogs.adoc), v9.8.2.
10. **gradle/gradle#15383 is open; there is no native accessor in precompiled project scripts.** Issue title "Make generated type-safe
    version catalogs accessors accessible from precompiled script plugins", state OPEN, last updated 2026-06-22.
    [Issue](https://github.com/gradle/gradle/issues/15383), read 2026-10-20.
11. **Root `plugins {}` with `apply false` is the documented way to share plugins with subprojects.** "In the root/parent
    `build.gradle(.kts)`, plugins are declared but not applied (via `apply false`). This approach, while optional, makes the plugins
    available to be explicitly applied in specific subprojects."
    [plugins_intermediate.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/fundamentals/authoring-builds/plugins_intermediate.adoc), v9.8.2.
12. **A settings-level `apply false` exports to the parent of every project's scope.** "Declare the plugin in the settings `plugins {}`
    block with `apply false`. This forces resolution at settings evaluation time and exports the classpath into the settings
    classloader scope — the parent of every project's scope — without applying the plugin there".
    [isolated_projects.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/optimizing-builds/isolated_projects.adoc), v9.8.2.
13. **Dokka only warns, and prescribes the root `plugins {}` remedy.** Dokka `KotlinAdapter.logKgpClassNotFoundWarning`: "The most
    common cause is a Gradle limitation: the plugins applied to subprojects should be consistent." … "1. Apply the Dokka and Kotlin
    plugins to the root project using the `plugins {}` DSL." … "2. Remove the Dokka and Kotlin plugins versions in the subprojects."
    [Source](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/adapters/KotlinAdapter.kt), Dokka v2.2.0.
14. **The root-hoist pattern fixed MyGameTools' empty javadoc.** Measured 2026-10-20 with `unzip -l`:
    `gametools-core-5.0.0-javadoc.jar` (261 bytes) contains 0 `.html` entries; `gametools-core-5.1.0-javadoc.jar` (1,690,968 bytes)
    contains 415. Commit `235810e` (ancestor of tag `v5.1.0`) is the root `plugins { id("org.jetbrains.kotlin.jvm") apply false … }` change.
    [5.0.0 jar](https://repo1.maven.org/maven2/io/github/spartanlabsgaming/gametools-core/5.0.0/gametools-core-5.0.0-javadoc.jar),
    [5.1.0 jar](https://repo1.maven.org/maven2/io/github/spartanlabsgaming/gametools-core/5.1.0/gametools-core-5.1.0-javadoc.jar).
15. **Now in Android's rationale for the root block.** "By listing all the plugins used throughout all subprojects in the root project
    build script, it ensures that the build script classpath remains the same for all projects. This avoids potential problems with
    mismatching versions of transitive plugin dependencies."
    [Source](https://github.com/android/nowinandroid/blob/a49ed253d75e61a2b6ab80a8da677b57437b08eb/build.gradle.kts), commit `a49ed253d75e` (2026-09-22).
16. **A plugin requested with a version while already on the classpath fails.** "This is a known limitation of the `plugins` block. A
    plugin declaration cannot use a version if the plugin is already on the classpath."
    [gradle/gradle#20084 comment](https://github.com/gradle/gradle/issues/20084#issuecomment-1060822638), read 2026-10-20.
17. **Dokka 2.2.0 and vanniktech 0.37.0 register build services; mismatched plugin sets break build services.** Class files
    `org/jetbrains/dokka/gradle/internal/PluginFeaturesService.class` in `dokka-gradle-plugin-2.2.0.jar` and
    `com/vanniktech/maven/publish/central/MavenCentralBuildService.class` in `gradle-maven-publish-plugin-0.37.0.jar` (Maven Central).
    Gradle's guidance: "Using different plugin combinations can cause features like Build Services to break in difficult to diagnose
    ways." [best_practices_structuring_builds.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/best-practices/best_practices_structuring_builds.adoc), v9.8.2; issue
    [#17559](https://github.com/gradle/gradle/issues/17559) (open, "Build services can only be registered if the set of applied plugins is the same on all projects").
18. **`build-logic` in an included build is Gradle's recommendation, with `pluginManagement { includeBuild }` in its sample.**
    "The preferred location for build logic is an included build (typically named `build-logic`), not in `buildSrc`." Sample
    `settings.gradle.kts`: `pluginManagement { includeBuild("build-logic") }`.
    [best_practices_structuring_builds.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/best-practices/best_practices_structuring_builds.adoc) and
    [sample](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/snippets/best-practices/useConventionPlugins-do/kotlin/settings.gradle.kts), v9.8.2.
19. **Catalogs are for libraries, not constants.** "Don’t use them to store shared strings or non-library constants".
    [best_practices_dependencies.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/best-practices/best_practices_dependencies.adoc), v9.8.2.
20. **`kotlin-dsl` compiles build logic at Kotlin 2.2 and tolerates toolchain-less use.** `KotlinDslCompilerPlugins`:
    `apiVersion.set(KotlinVersion.KOTLIN_2_2)`, `languageVersion.set(KotlinVersion.KOTLIN_2_2)` and
    "Set this back to a warning for now, as this plugin is frequently used without toolchains specifying a JVM target".
    [Source](https://github.com/gradle/gradle/blob/v9.8.2/platforms/core-configuration/kotlin-dsl-plugins/src/main/kotlin/org/gradle/kotlin/dsl/plugins/dsl/KotlinDslCompilerPlugins.kt), v9.8.2.
21. **Embedded Kotlin per Gradle release.** Table rows `| 2.2.0  | 9.0.0 | 2.2`, `| 2.4.0  | 9.7.0 | 2.2`, `| 2.4.10 | 9.8.0 | 2.2`
    under "Embedded Kotlin version / Minimum Gradle version / Kotlin Language version".
    [compatibility.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/releases/compatibility.adoc), v9.8.2.
22. **vanniktech 0.37.0 minimums.** "Minimum supported versions - JDK 17 - Gradle 9.0.0 - Android Gradle Plugin 8.13.0 - Kotlin Gradle
    Plugin 2.2.0".
    [CHANGELOG.md](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/CHANGELOG.md), tag 0.37.0 (2026-06-21).
23. **Kotlin Gradle plugin 2.4.20–2.4.21 Gradle range.** Table row `| 2.4.20–2.4.21 | 7.6.3–9.7.0 | 8.5.2–9.3.1 |`.
    [Kotlin docs, Configure a Gradle project](https://kotlinlang.org/docs/gradle-configure-project.html), read 2026-10-20. (Flagged for question 2; the recommendation here does not depend on it.)
24. **Ktor 3.6.0's hoist and accessor hack (comparable-project evidence).** `build-logic/build.gradle.kts`: "A hack to make version
    catalogs accessible from buildSrc sources" `implementation(files(libs.javaClass.superclass.protectionDomain.codeSource.location))`;
    `ktor-dokka/build.gradle.kts` applies `id("ktorbuild.dokka")`; `build.gradle.kts` applies `id("ktorbuild.root")`.
    [Ktor 3.6.0](https://github.com/ktorio/ktor/tree/3.6.0).
25. **kotlinx-io 0.9.1's form (comparable-project evidence).** `settings.gradle.kts`: `pluginManagement { includeBuild("build-logic") …`;
    `build-logic/settings.gradle.kts`: `create("libs") { from(files("../gradle/libs.versions.toml")) }`; `gradle.properties`:
    `group=org.jetbrains.kotlinx`, `version=0.10.0-SNAPSHOT`.
    [kotlinx-io 0.9.1](https://github.com/Kotlin/kotlinx-io/tree/0.9.1).
26. **Class-file metadata of the plugin jars.** `javap -v` on `KotlinBasePlugin.class` (`kotlin-gradle-plugin-api-2.4.21.jar`) shows
    `mv` `1`, `8`; on `kotlin/Unit.class` (`kotlin-stdlib-2.4.21.jar`) `mv` `2`, `4`; on `DokkaBasePlugin.class` (`dokka-gradle-plugin-2.2.0.jar`)
    `mv` `1`, `4`; on a `MavenPublishPlugin` class (`gradle-maven-publish-plugin-0.37.0.jar`) `mv` `2`, `0`.
    Jars from `https://repo1.maven.org/maven2/` (`org/jetbrains/kotlin/kotlin-gradle-plugin-api/2.4.21/`, `org/jetbrains/kotlin/kotlin-stdlib/2.4.21/`,
    `org/jetbrains/dokka/dokka-gradle-plugin/2.2.0/`, `com/vanniktech/gradle-maven-publish-plugin/0.37.0/`), read 2026-10-20. (Used only to bound the C1 risk.)
