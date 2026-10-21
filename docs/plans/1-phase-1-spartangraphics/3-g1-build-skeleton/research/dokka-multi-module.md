# Dokka 2 javadoc jars through vanniktech, and one aggregated site, for the G1 build skeleton

- **Issue:** [SpartanLaboratories/SpartanGraphics#3](https://github.com/SpartanLaboratories/SpartanGraphics/issues/3), question 4 of the
  [research README](README.md).
- **Read:** 2026-10-20. Pinned sources: Dokka `v2.2.0` (and `v2.3.0-Beta` where stated), vanniktech `0.37.0`, Gradle `v9.8.2` docs through the
  reused findings, Coil `3.6.3`, Ktor `3.6.0`, kotlinx-io `0.9.1`, Square workflow-kotlin `40d22c9eb443` (2026-08-19), MyGameTools `dc5214f`,
  JetBrains/kotlin `8d016a647d9a` (a `master` commit of 2026-10-09; see *Gaps*).
- **Method limit:** nothing was built or run: no Gradle task, in this repository or any other. Every behaviour below comes from the source of
  Dokka and vanniktech at the pinned tags (read from shallow clones in a scratch directory, since deleted), from their documentation, and from
  four measurements of published artifacts (Maven Central jar listings). The wiring in section 6 has not been executed; section 6.6 lists the
  runs the implementation must make, including a negative run that proves the guard bites.

## 1. Question

How does Dokka 2.2.0 build each published library module's javadoc jar through vanniktech 0.37.0, and one aggregated site at the root, for a
Kotlin Multiplatform module (`jvm()` and `js { nodejs() }`) and a `kotlin("jvm")` + `java-library` module that hold no Kotlin source yet,
while the `java-platform` BOM gets no Dokka and no javadoc jar? An answer must give: jar content from Dokka for each library module; one
root site that CI builds; no failure for a module without source; a build-time or CI-time catch for a module that renders nothing because
Dokka cannot load `KotlinBasePlugin`; the root-hoisted classloader arrangement from
[`build-logic-and-version-catalog.md`](build-logic-and-version-catalog.md) honoured; configuration-cache compatibility; the exact convention,
root and `gradle.properties` text; and the task names CI runs.

## 2. Bottom line

- **Wiring.** The publishing convention (`spartangraphics.publishing`) selects the vanniktech platform by `pluginManager.withPlugin(...)` and,
  for the two Kotlin plugin ids only, applies Dokka (through one typed helper in `build-logic`) and pins
  `KotlinMultiplatform(JavadocJar.Dokka("dokkaGeneratePublicationHtml"))` or `KotlinJvm(JavadocJar.Dokka("dokkaGeneratePublicationHtml"))`.
  For `java-platform` it pins `JavaPlatform()`, which publishes no javadoc jar (Sonatype exempts `pom` packaging). The BOM never sees Dokka.
  No fourth convention id is needed. HTML is the only usable format: Dokka's Javadoc format does not support Kotlin Multiplatform.
- **Only `dokkaGeneratePublicationHtml` works as the jar's input.** The lifecycle task `dokkaGenerate` has no outputs, and Coil 3.6.3, which
  passes it (`KotlinMultiplatform(Dokka("dokkaGenerate"))`), published 261-byte, manifest-only javadoc jars on Maven Central for
  `coil-core` and `coil-core-jvm`. vanniktech's own docs still show the Dokka v1 task name `dokkaHtml`, which does not exist in V2 mode.
- **Plugin mode.** Dokka 2.2.0 runs the V2 plugin by default (`V2EnabledWithHelpers`). `…pluginMode=V2Enabled` is harmless and tidy: it only
  drops the disabled V1 task and configuration shims. `…pluginMode.noWarn=true` is inert: it only silences a V1 message that V2 modes never
  print. Keep the first line, drop the second.
- **An empty module does not fail the build and produces an empty jar.** With Kotlin detected but no declarations, Dokka drops the empty module
  (`EmptyModulesFilterTransformer`) and exits generation gracefully ("Nothing to document"); the task succeeds with an empty output directory,
  each module's javadoc jar is manifest-only (the size Okio and Coil publish and Central accepts), and the root aggregate renders an "All modules"
  page with no rows.
- **A page count cannot catch the classloader failure in G1.** The failure (Dokka cannot load `KotlinBasePlugin`) and the legitimate empty
  module both yield zero pages. They differ in the configuration model: when the adapter works Dokka registers one source set per Kotlin source set
  even with no files; when it fails, none is left unsuppressed. The guard is a `verifyDokkaSourceSets` task per library module that fails when
  no Dokka source set is documented, wired into `check` and ahead of the module's Dokka tasks. It keeps working when real API lands. A log grep is
  unreliable under the configuration cache (the warning is a configuration-time message), and `failOnWarning` cannot see it.
- **Root and source links.** Root: `alias(libs.plugins.dokka)` applied, `dokka(project(":graphics-core"))` and
  `dokka(project(":graphics-desktop"))`; the aggregate is `:dokkaGeneratePublicationHtml`, output `build/dokka/html/`. One `sourceLink` per
  Dokka source set with `localDirectory` = repository root and `remoteUrl` = `…/blob/main` covers KMP and JVM source sets alike (Ktor and
  kotlinx-io do the same). Dokka 2.2.0 and the 2.3.0-Beta both print one Gradle 9.8 deprecation warning (`Configuration.setVisible`); Gradle 9.7.1
  prints none. Neither candidate wrapper changes the design.

## 3. Findings

Tiers: **consensus** = Dokka's or vanniktech's documentation or source at the pinned tag, or several independent sources; **practice** = what
mature builds visibly do; **opinion** = one source; **inference** = derived here from sources that do not state it.

### F1. Dokka 2.2.0 runs the V2 Gradle plugin by default; the two MyGameTools flags are not both needed (consensus; Dokka `v2.2.0`)

- `PluginFeaturesService.PluginMode.Default` is `V2EnabledWithHelpers`; the docs say "Starting from Dokka 2.1.0, DGP v2 is enabled by default."
  `V1Enabled` still exists in the 2.2.0 and 2.3.0-Beta sources and is the only mode that logs a message.
- The `noWarn` flag (`…pluginMode.noWarn`, also read as `…nowarn`) is consulted only in `logV1PluginMessage`. In the two V2 modes the code
  comments read "this is the default, don't log messages to avoid spamming users". So `noWarn=true` does nothing in V2.
- `V2EnabledWithHelpers` additionally calls `addV2MigrationHelpers`: it registers disabled V1 tasks (`dokkaHtml`, `dokkaHtmlPartial`,
  `dokkaHtmlMultiModule`, `dokkaGfm`, …) and deprecated V1 configurations, with `notCompatibleWithConfigurationCache(...)` on each V1 task.
  They never run unless named. `V2Enabled` omits them. MyGameTools' line
  `org.jetbrains.dokka.experimental.gradle.pluginMode=V2Enabled` was written for Dokka 2.0.0/2.1.0 and is still valid; its `noWarn` line was
  needed only while V1 could be selected.
- Dokka reads this flag by itself even while Gradle generates accessors for a precompiled script that applies Dokka in `plugins {}`
  (`configureParamsDuringAccessorsGeneration`). The design below applies Dokka programmatically, so that path is not exercised.

### F2. The javadoc jar through vanniktech 0.37.0 (consensus; vanniktech `0.37.0`)

- Auto-detection exists. `MavenPublishPlugin.apply` registers `afterEvaluate { baseExtension.configureBasedOnAppliedPlugins() }`, which "will
  no-op if it was already called". With no `configure(...)` call it picks, in this order for the javadoc jar: `org.jetbrains.dokka-javadoc`
  applied gives `JavadocJar.Dokka("dokkaGeneratePublicationJavadoc")`; `org.jetbrains.dokka` applied (and the `dokka` extension present, that is V2
  mode) gives `JavadocJar.Dokka("dokkaGeneratePublicationHtml")`; otherwise `Javadoc()` for non-KMP and `Empty()` for KMP. The platform comes from
  the applied plugins: KMP gives `KotlinMultiplatform`, `org.jetbrains.kotlin.jvm` gives `KotlinJvm`, `java-platform` gives `JavaPlatform`.
  So applying Dokka only to the Kotlin library modules would already give the right result; an explicit `configure(...)` pins it, so a
  stray `org.jetbrains.dokka-javadoc` (an unsupported combination for KMP) or a vanniktech change cannot switch the format silently.
- `JavadocJar.Dokka(taskName: String)` (also `Provider<String>` and `TaskProvider<*>` overloads) registers a `JavadocJar` task (a
  `org.gradle.jvm.tasks.Jar`) that does `dependsOn(dokkaTask)` and `from(dokkaTask)`; the Dokka task is resolved lazily with `tasks.named(...)`.
  `from(task)` copies the task's declared outputs.
  - JVM: one task, `dokkaJavadocJar`.
  - KMP: `withJavadocJar(javadocJar, project, multipleTasks = true)` runs for every Maven publication, so one jar task per publication, prefixed by
    the publication name: `kotlinMultiplatformDokkaJavadocJar`, `jvmDokkaJavadocJar`, `jsDokkaJavadocJar` (derived from `prefixedTaskName`).
    All three copy the same Dokka output. Measured: Square workflow-kotlin 1.31.0's `workflow-core`, `workflow-core-jvm` and `workflow-core-js`
    javadoc jars are byte-identical, 1,031,440 bytes, 250 `.html` entries each.
- **The task must be `dokkaGeneratePublicationHtml`.** Dokka's docs describe it as the task that "exposes the output directory as an
  `@OutputDirectory`" for "packaging them into a `javadoc.jar`". `dokkaGenerate` is a plain `DokkaBaseTask : DefaultTask()` with `dependsOn`
  and no outputs, so `from(dokkaGenerate)` copies nothing. Coil 3.6.3's convention passes `Dokka("dokkaGenerate")` and its published jars are
  261 bytes with only `META-INF/MANIFEST.MF` (measured, 2026-10-20). The cause is inferred from the two sources above; it was not run.
- vanniktech's documentation (`docs/what.md` at `0.37.0`) still tells readers to pass `dokkaHtml`, a Dokka v1 task name. Its own
  `DokkaPluginTest` passes `dokkaGeneratePublicationHtml`, but only for a `kotlin("jvm")` project; no KMP + Dokka fixture exists
  in the plugin's tests. KMP + Dokka evidence comes from the field (workflow-kotlin, below).
- `JavaPlatform` sets `javadocJar = JavadocJar.None()` and `sourcesJar = SourcesJar.None()`, and calls
  `from(project.components.getByName("javaPlatform"))`. `JavadocJar.None` is documented as "not compatible with Maven Central"; that applies to
  jar packaging. Sonatype: "Projects with packaging other than `pom` have to supply JAR files that contain Javadoc and sources." The BOM is
  `pom` packaging (question 5 owns its publication).

### F3. Dokka's Javadoc format cannot serve this build (consensus; Dokka `v2.2.0` docs)

"Dokka doesn't support the Javadoc format for multi-project builds or Kotlin Multiplatform projects", and the format is Alpha. The brief's
choice is therefore not open: HTML for both module kinds, and one task name for both. Dokka's HTML pages "need to be hosted on a web server in
order to render everything correctly", so the jar's content is meant for hosts such as javadoc.io, not for opening from disk.

### F4. What Dokka registers for a module, and why that discriminates the classloader failure (consensus on the code; the consequence is inference)

- `KotlinAdapter.applyTo` calls `project.plugins.withType<KotlinBasePlugin>()`. If the class cannot be loaded it catches
  `ClassNotFoundException`/`NoClassDefFoundError`, logs the warning quoted in question 1 (a Gradle `logger.warn` at configuration time) and
  returns without registering anything.
- When the class loads, the adapter registers one `DokkaSourceSetSpec` for every `KotlinSourceSet`, with no condition on whether the source
  directories exist: `sourceSetDetails.all { dokkaExtension.dokkaSourceSets.register(...) }`. `suppress` defaults to
  `!details.isPublishedSourceSet()`, that is true for the test source sets and false for the main ones. Expected non-suppressed sets in G1:
  KMP `commonMain`, `jvmMain`, `jsMain`; JVM `main`. (Dokka's own test `KotlinSourceSetGistTest` records that source roots are empty "because
  default source root does not exist on filesystem yet": an empty root set does not remove the source set.)
- `JavaAdapter` also registers `javaMain`/`javaTest` for a module with `java-library`, but suppresses them whenever a Kotlin or Android plugin
  *id* is applied (`PluginIds.kotlin.any { project.pluginManager.hasPlugin(it) }`, an id check that needs no class loading). So in the failure
  state a JVM module has registered source sets and none of them is documented.
- The `DokkaExtension.dokkaSourceSets` KDoc states the contract: "Dokka will not generate documentation unless there is at least one Dokka
  Source Set." A check on the set of non-suppressed Dokka source sets is therefore a check on the one precondition the failure removes.

### F5. An empty module: no failure, no pages, an empty jar (consensus on the code; not executed)

- Generation. With source sets present but no declarations, `translateSources` yields `DModule`s with no packages. `skipEmptyPackages` defaults to
  `true`, and `EmptyModulesFilterTransformer` (`modules.filter { it.children.isNotEmpty() }`, ordered after `emptyPackagesFilter`) then removes them.
  `mergeDocumentationModels` returns null (`DefaultDocumentableMerger` uses `reduceOrNull`), and `SingleModuleGeneration` runs
  `exitGenerationGracefully("Nothing to document")`. `DokkaGenerator` catches `GracefulGenerationExit` and logs `Exiting Generation: Nothing to
  document` at progress level, which the Gradle adapter maps to `info`. Dokka tests it: `AbortGracefullyOnMissingDocumentablesTest`,
  "Generation aborts Gracefully with no Documentables". Nothing is written; no exception; `failOnWarning` is not triggered (the exit is not a
  warning).
- Output directories exist. `DokkaGeneratorWorker.prepareOutputDir` deletes and re-creates the output directory (and each module's directory)
  before the generator runs, and `DokkaGenerateModuleTask` always writes `module-descriptor.json`. So a root aggregate that reads an empty module
  directory does not hit "missing module output directory".
- Aggregation. `DefaultTemplateProcessor.visit` walks an empty module directory (it calls `target.mkdirs()`) and counts a module as non-empty only
  if the directory holds a non-dot file; `MultimodulePageCreator` lists only `nonEmptyModules`. With both modules empty, the root publication
  renders an "All modules" page with an empty table. (Dokka issue #1687, "New MultiModule should omit empty modules", is the history.)
- Analysis. K2 analysis builds one source module per source set with `addSourceRoots(sourceSet.sourceRoots)`; the standalone session maps every
  source module to `psiRoots.filterIsInstance<PsiFile>()`, an empty list for no files, so `modulesWithFiles[sourceModule]` is present, not null,
  and the "No source files for a source module" `IllegalStateException` is not raised. This last step was read on JetBrains/kotlin `master`, not at
  the Kotlin build Dokka 2.2.0 embeds (`2.3.20-dev-7064`); see *Gaps*.
- Result in G1. `:graphics-core:dokkaGeneratePublicationHtml` and `:graphics-desktop:dokkaGeneratePublicationHtml` succeed with an empty
  `build/dokka/html`; every `*DokkaJavadocJar` is a manifest-only jar; the root's `build/dokka/html` holds `index.html`, `navigation.html` and
  Dokka's UI assets. Sonatype: "If you cannot provide sources or documentation, we allow uploading placeholder `.jar` files in order to pass
  validation"; Okio 3.18.2 and Coil 3.6.3 publish 261-byte javadoc jars that Central accepted (reused:
  [`kmp-maven-central-publishing.md`](https://github.com/SpartanLaboratories/GeneralTools/blob/d424aaf/docs/plans/7-kotlin-multiplatform/research/kmp-maven-central-publishing.md),
  Q "Central accepts what KMP produces", and the Coil listing above).
- **Consequence for the guard.** MyGameTools' measurement (question 1, F3) found `gametools-core` with 0 `.html` files in the failure state.
  The empty-module path also produces 0 `.html` files. The failure and "no source yet" are indistinguishable by output in G1; only the
  configuration model (F4) separates them.
- `js` analysis needs the Kotlin/JS stdlib klib on the source set's classpath. The adapter takes the compilation classpath
  (`compilation.compilationClasspath`), resolved from the repositories as for compilation. Node.js and Yarn are test/run machinery and are not
  referenced by the adapter. (Inference from source; `jsMain` is not run in this research.)

### F6. Detecting the silent failure: what each candidate costs (consensus on mechanics; the recommendation is a judgement)

| Check | Tells failure from "no source yet" in G1? | Works once API exists? | Cost / fault |
| --- | --- | --- | --- |
| Count `.html` files in the javadoc jar (MyGameTools `verifyUmbrellaJavadocJar`) | No: both give 0 | Yes | Needs the jar built; threshold must change at G2 |
| Dokka `failOnWarning` | No: the `KotlinBasePlugin` warning is a Gradle `logger.warn` in the plugin; `failOnWarning` counts only `DokkaLogger` warnings during generation (`SingleModuleGeneration.reportAfterRendering`) | n/a | None, but does not apply |
| CI grep of the build log for `Dokka could not load KotlinBasePlugin` | Yes, on a fresh configuration | Yes | The message is logged during configuration only. A configuration-cache hit (e.g. restored by `setup-gradle`) does not re-run configuration, so the grep passes with the failure still present. |
| Read `dokka-configuration.json` (`build/tmp/<task>/`) and check `sourceSets` | Yes | Yes | It is a debugging dump (`dokkaConfigurationJsonFile` is `@Internal`), a file format Dokka does not promise |
| **Task `verifyDokkaSourceSets`: non-suppressed `dokka.dokkaSourceSets` names, captured as a task input, fail on empty** | **Yes** | **Yes** | One small task class in `build-logic`; the read happens at configuration-cache store time (inferred safe, section 6.6 runs it) |
| The same check inline at configuration time (`afterEvaluate { check(...) }`) | Yes | Yes | Fails `clean`, IDE sync and every unrelated invocation; rejected |
| Assert on the module page (`index.html`) once API exists | n/a in G1 | Yes | Optional later hardening against analysis problems, with an ABI-dump condition to avoid false alarms for modules whose public API is empty |

### F7. Source links (practice; Dokka `v2.2.0`, Ktor `3.6.0`, kotlinx-io `0.9.1`)

- `DokkaSourceLinkSpec`: `localDirectory` ("path to the local source directory"), `remoteUrl` (String and Provider overloads), `remoteLineSuffix`
  (default `#L`). Dokka resolves a declaration's file relative to `localDirectory` and appends it to `remoteUrl`. The default `localDirectory`
  convention is the project directory.
- Ktor and kotlinx-io both set one block for all source sets with `localDirectory` = the repository root: `sourceLink { localDirectory = rootDir;
  remoteUrl("https://github.com/Kotlin/kotlinx-io/tree/master") … }`, and Ktor `remoteUrl("https://github.com/ktorio/ktor/blob/$version")`. With the
  root as `localDirectory` a file `graphics-core/src/jvmMain/kotlin/…/A.kt` becomes `…/blob/main/graphics-core/src/jvmMain/kotlin/…/A.kt#L10`, so no
  per-source-set directory, and no KMP/JVM distinction, is needed. A per-source-set `src/$name/kotlin` form would also work (Dokka source-set names
  equal the directory names in both module kinds) but multiplies configuration for nothing.
- Links to `main` move with the branch and rot in published jars; Ktor links the version tag instead. A tag-or-`main` switch is a two-line
  change, deferred (the tag does not exist when the jar is built at G7 until the tag is pushed).

### F8. Configuration cache and Gradle 9.7.1 / 9.8.2 (consensus on the claims; the combination is inference)

- Dokka documents support: "DGP supports Gradle build cache and configuration cache, improving build performance." 2.2.0's release notes include
  "Fix DGP reads all Gradle properties, which causes unnecessary CC invalidation" (#4467). vanniktech 0.34.0 added configuration-cache support;
  its integration tests run with `--configuration-cache` (reused: GeneralTools' `kmp-maven-central-publishing.md`).
- Dokka's declared minimum is Gradle 7.6; its own CI runs 7.6.3 and 8.14.3 (question 2). Coil 3.6.3, Ktor 3.6.0 and MyGameTools run Dokka 2.2.0
  on Gradle 9.7.1.
- Gradle 9.8: Dokka 2.2.0 calls `Configuration.isVisible = …` in `declarable()`/`consumable()`, which Gradle 9.8 reports as "deprecated ... scheduled to
  be removed in Gradle 11" (Kotlin/dokka#4600). PR #4601 (merged 2026-09-11, commit `cba8250`) removes the call on Gradle 9+. It is **not** in any release:
  `v2.3.0-Beta` (the only newer tag) diverges from that commit and still contains the call in `gradleUtils.kt`. It is one warning in Gradle's
  deprecation summary, not a failure (no `--warning-mode fail` is used). On 9.7.1 Gradle does not report it. This does not decide between the
  wrappers: either works, and 9.8.x's other cost is KGP's documented ceiling of 9.7.0 (question 2).
- Latest releases checked 2026-10-20 (Maven Central metadata): Dokka Gradle plugin 2.2.0 stable (2.3.0-Beta is a pre-release), vanniktech
  0.37.0.

### F9. Styling and what the output looks like with none (consensus; Dokka `v2.2.0`)

- Default: Dokka's standard HTML theme; footer `© <current year> Copyright` (`DokkaBaseConfiguration.defaultFooterMessage`); no logo; the module name
  is the project name (`graphics-core`), the version is the project version, and the root site's title is the root project name. Defaults that
  matter: `jdkVersion` 11 (the JDK link points at the Java 11 API, set it to 21), `externalDocumentationLinks` for the JDK and the Kotlin stdlib are
  enabled and are fetched over the network at generation time (`offlineMode` is the switch).
- Options and cost, per `pluginsConfiguration.html`: `footerMessage` and `homepageLink` are one line each; `customAssets` (a logo, for example
  `logo-icon.svg` as in Ktor) needs a file in the repository; `customStyleSheets` needs a CSS file that must be re-checked against each Dokka
  upgrade (the page markup is Dokka's, so this is a maintenance cost; stated here as an opinion). Because modules render their own pages with their own
  html configuration and the root's block styles only the aggregate pages, every option is needed in two places: the helper applied to each
  module and the root build script (the root cannot call `build-logic`'s internal helpers). The repository has no brand assets, so G1 ships
  no styling beyond what section 6 shows as an optional footer.

## 4. How comparable projects do it

| Project @ ref | Dokka and publishing wiring | Source link | Published javadoc jar |
| --- | --- | --- | --- |
| [Square workflow-kotlin @ `40d22c9eb443`](https://github.com/square/workflow-kotlin/blob/40d22c9eb443adb0b00bbf653e6fab7898ccf052/build-logic/src/main/java/com/squareup/workflow1/buildsrc/PublishingConventionPlugin.kt) (symbol `PublishingConventionPlugin.apply`) | `com.vanniktech.maven.publish.base`; `withPlugin("org.jetbrains.kotlin.multiplatform") { configure(KotlinMultiplatform(javadocJar = JavadocJar.Dokka(taskName = "dokkaGeneratePublicationHtml"))) }`, and `withPlugin("org.jetbrains.kotlin.jvm")` with `KotlinJvm(…Dokka…)`. The closest analogue: KMP with `jvm` and `js` targets, per-kind selection inside the publishing convention. | n/a read | KMP `workflow-core` 1.31.0 (2026-08-25): root, `-jvm`, `-js` jars identical, 1,031,440 bytes, 250 `.html` entries (`index.html`, `navigation.html`, `workflow-core/<package>/…`). |
| [Coil 3.6.3](https://github.com/coil-kt/coil/blob/3.6.3/buildSrc/src/main/kotlin/coil3/projects.kt) (symbols `multiplatformAndroidLibrary`, `setupDokka`) | `KotlinMultiplatform(Dokka("dokkaGenerate"))`; root applies Dokka, `dokka(project(":$module"))` for each public module, `outputDirectory` = `docs/api`; `failOnWarning = true`, `jdkVersion = 8`, many external links | n/a read | `coil-core` and `coil-core-jvm` 3.6.3: 261 bytes, manifest only. |
| [Ktor 3.6.0](https://github.com/ktorio/ktor/blob/3.6.0/build-logic/src/main/kotlin/ktorbuild.dokka.gradle.kts) | Dokka via `ktorbuild.dokka`, aggregated by a dedicated `ktor-dokka` project; Dokka runs only for stable versions; no Dokka in the publish convention (no javadoc content) | `localDirectory = rootDir`, `remoteUrl("https://github.com/ktorio/ktor/blob/$version")` | not read |
| [kotlinx-io 0.9.1](https://github.com/Kotlin/kotlinx-io/blob/0.9.1/build-logic/src/main/kotlin/kotlinx/io/conventions/kotlinx-io-dokka.gradle.kts) | Dokka per module for the site; the publish convention attaches an empty javadoc `Jar` (`configureEmptyJavadocArtifact`) | `localDirectory = rootDir`, `remoteUrl("https://github.com/kotlin/kotlinx-io/tree/master")` | empty by design |
| MyGameTools @ `dc5214f` (`gametools.published-library`, root `build.gradle.kts`) | `configure(JavaLibrary(javadocJar = JavadocJar.Dokka("dokkaGeneratePublicationHtml"), sourcesJar = true))` for `kotlin("jvm")`; root aggregates three modules; flags `pluginMode=V2Enabled` + `noWarn=true`; `verifyUmbrellaJavadocJar` counts HTML in the umbrella jar | per-module `sourceLink` to `blob/master/<module>/src/main/kotlin` | `gametools-core` 5.1.0: 1,690,968 bytes, 415 `.html` (question 1) |
| vanniktech 0.37.0 own tests (`DokkaPluginTest`) | `kotlinJvmProjectSpec` with `JavadocJar.Dokka("dokkaGeneratePublicationHtml")`, asserts `index.html` in the jar | n/a | JVM only; no KMP + Dokka fixture |

Pattern: libraries either keep Dokka as a site and publish empty javadoc jars (kotlinx-io, Okio, the JetBrains template), or wire
`dokkaGeneratePublicationHtml` into the jar (workflow-kotlin, MyGameTools). The latter is the form this build needs.

## 5. Pitfalls and regrets

- **Passing `dokkaGenerate` (or the v1 name `dokkaHtml` from vanniktech's docs) as the jar's task** gives an empty jar or a missing-task error. A green
  build and a Central-accepted release prove nothing, because Central accepts manifest-only jars. Coil 3.6.3's published jars show it.
- **Believing a page count tells the failure from an empty module.** In G1 both give zero. Question 1's suggestion to "assert a page count" cannot
  stand alone until real API exists.
- **Grepping the log for the Dokka warning under the configuration cache.** The warning is printed once, when the cache entry is stored. A restored
  entry (including one restored by CI caching) reruns no configuration and prints nothing. The question 1 check "prints no `Dokka could not load
  KotlinBasePlugin`" must be made on a fresh configuration (`--no-configuration-cache`).
- **Applying Dokka in the shared publishing convention unconditionally.** The BOM would receive Dokka, a pointless publication and Dokka
  configurations. The sketch in question 1, §6.9 does this (`plugins { id("com.vanniktech.maven.publish"); id("org.jetbrains.dokka") }`); the
  conditional form in section 6 replaces it.
- **A version on any Dokka or Kotlin plugin request outside the root `plugins {}` block** splits the classloader copy or fails (question 1, F3).
  `pluginManager.apply("org.jetbrains.dokka")` carries no version, as intended.
- **The typed helper depends on the single-copy assumption.** `extensions.getByType<DokkaExtension>()` compares classes loaded from `build-logic`'s
  classpath with the extension Dokka created. If the root hoist did not unify the loaders, `getByType` would fail with a class mismatch: a loud
  failure, not a silent one.
- **Dokka's external documentation links need the network** (JDK and Kotlin stdlib package lists) and `failOnWarning = true`, which Coil enables, turns any
  Dokka warning (broken KDoc link, unresolved reference) into a build failure. Left off in G1 (nothing to warn about); it is a G2 decision, once
  KDoc exists.
- **`dokka(project(...))` is a hand-written list.** A library module added to `settings.gradle.kts` and forgotten at the root silently drops out of
  the site. `CONTRIBUTING.md`'s "add a module" steps and a Level-3 check of the `include(...)` list against the root dependencies (question 6)
  are the guards; the root cannot enumerate sibling projects without cross-project configuration.
- **The jar holds an HTML site.** IDEs do not render it as Javadoc; the audience is javadoc.io and similar hosts. That is the cost of "Dokka from
  the first commit" for Kotlin libraries and is what Square's and MyGameTools' consumers get.
- **Dokka forks a JVM** (default `ProcessIsolation`, `maxHeapSize` 2g) and needs the settings repositories to reach its generator classpath
  (`mavenCentral()` suffices in MyGameTools). Memory on a hosted runner and first-run download time are the cost of the CI step.

## 6. Recommendation for this repository

Apply Dokka only to the Kotlin library modules, from the publishing convention, through one typed `build-logic` helper; pin the jar to
`dokkaGeneratePublicationHtml`; guard with a source-set check wired into `check` and ahead of Dokka's own tasks; aggregate at the root with
`dokka(project(...))`. A separate `spartangraphics.docs` precompiled script (the other shape the options allow) is not used because it would
add a fourth public convention id for about twenty lines that read just as well in a `.kt` helper beside `Floor.kt` and `PublicApi.kt`,
and typed accessors matter little for `extensions.configure<DokkaExtension>`. Auto-detection by vanniktech is not relied on, so that applying a second Dokka
plugin or upgrading vanniktech cannot change the jar format silently.

### 6.1 Relation to the other findings

- Refines [`build-logic-and-version-catalog.md`](build-logic-and-version-catalog.md) §6.9: `spartangraphics.publishing` no longer lists
  `id("org.jetbrains.dokka")` in its `plugins {}` block; it applies Dokka conditionally (6.3). §5 and F3's "fail on the warning text, or assert a page
  count" is replaced by the source-set guard (F6). §6.10 (1) must run with `--no-configuration-cache`.
- Consistent with [`toolchain-versions-and-pins.md`](toolchain-versions-and-pins.md) F2 and extends it: no released Dokka contains the `setVisible` fix.
- Consistent with [`abi-validation-across-modules.md`](abi-validation-across-modules.md): `guardPublicApi()` and the Dokka helper are separate
  calls; neither touches the BOM or the root.

### 6.2 `gradle.properties` (Dokka lines; group, version and runtime flags are in question 1, §6.3)

```properties
# Dokka 2.2.0 runs its V2 Gradle plugin by default (V2EnabledWithHelpers). V2Enabled also drops the disabled V1 task and configuration shims.
# No ...pluginMode.noWarn line: it only silences a V1 message that V2 modes never print.
org.jetbrains.dokka.experimental.gradle.pluginMode=V2Enabled
```

### 6.3 `build-logic`: the helper, the guard task, and the publishing convention

`build-logic/src/main/kotlin/com/spartanlabs/graphics/build/Docs.kt`

```kotlin
package com.spartanlabs.graphics.build

import com.vanniktech.maven.publish.tasks.JavadocJar as JavadocJarTask
import org.gradle.api.DefaultTask
import org.gradle.api.Project
import org.gradle.api.provider.ListProperty
import org.gradle.api.provider.Property
import org.gradle.api.tasks.Input
import org.gradle.api.tasks.TaskAction
import org.gradle.kotlin.dsl.getByType
import org.gradle.kotlin.dsl.register
import org.gradle.kotlin.dsl.withType
import org.gradle.work.DisableCachingByDefault
import org.jetbrains.dokka.gradle.DokkaExtension

/**
 * The Dokka task that holds a module's HTML site in a declared `@OutputDirectory`. It is the input of every javadoc jar. The lifecycle task
 * `dokkaGenerate` has no outputs and would give an empty jar.
 */
internal const val DOKKA_HTML_PUBLICATION = "dokkaGeneratePublicationHtml"

private const val SOURCE_ROOT_URL = "https://github.com/SpartanLaboratories/SpartanGraphics/blob/main"

/**
 * Fails when Dokka would document nothing because it registered no source set. Dokka itself only warns when it cannot load `KotlinBasePlugin`
 * (the Kotlin plugin lives in another classloader) and then renders an empty site; a module with no source yet also renders nothing, so only
 * the configured source sets tell the two apart.
 */
@DisableCachingByDefault(because = "Compares configured names only; there is nothing to cache.")
abstract class VerifyDokkaSourceSets : DefaultTask() {
    /** Names of the Dokka source sets that will be documented (not suppressed). */
    @get:Input
    abstract val documentedSourceSets: ListProperty<String>

    @get:Input
    abstract val modulePath: Property<String>

    @TaskAction
    fun verify() {
        val names = documentedSourceSets.get()
        check(names.isNotEmpty()) {
            "Dokka has no source set to document in ${modulePath.get()}, so its site and javadoc jar would be empty. This almost always means " +
                "Dokka could not load KotlinBasePlugin (see 'Dokka could not load KotlinBasePlugin' in the configuration output). Keep the " +
                "Kotlin, Dokka and vanniktech plugins in the root build.gradle.kts plugins {} block with their versions from the catalog, " +
                "and request them without a version everywhere else."
        }
        logger.info("Dokka documents {} in {}", names, modulePath.get())
    }
}

/** Applies Dokka to a Kotlin library module and configures it: JDK level, source links, the guard, and the `javadocJars` lifecycle task. */
internal fun Project.configureDokka() {
    pluginManager.apply("org.jetbrains.dokka") // no version: the root plugins {} block loaded the plugin

    val dokka = extensions.getByType<DokkaExtension>()
    dokka.dokkaSourceSets.configureEach {
        jdkVersion.set(SpartanBuild.JVM_LEVEL)
        sourceLink {
            // The repository root as the local directory: one block serves every source set of a KMP or JVM module.
            localDirectory.set(isolated.rootProject.projectDirectory)
            remoteUrl(SOURCE_ROOT_URL)
        }
    }

    val owner = path // the project path; inside the task block, `path` would be the task's
    val verify = tasks.register<VerifyDokkaSourceSets>("verifyDokkaSourceSets") {
        group = "verification"
        description = "Fails when Dokka registered no source set for this module (empty site and javadoc jar)."
        documentedSourceSets.set(providers.provider { dokka.dokkaSourceSets.filterNot { it.suppress.get() }.map { it.name }.sorted() })
        modulePath.set(owner)
    }
    tasks.named("check") { dependsOn(verify) }
    tasks.named("dokkaGenerateModuleHtml") { dependsOn(verify) } // the task the root aggregate consumes
    tasks.named(DOKKA_HTML_PUBLICATION) { dependsOn(verify) }    // the task every javadoc jar consumes

    tasks.register("javadocJars") {
        group = "documentation"
        description = "Builds every javadoc jar of this module from Dokka's HTML publication."
        dependsOn(tasks.withType<JavadocJarTask>())
    }
}
```

`build-logic/src/main/kotlin/spartangraphics.publishing.gradle.kts` (publishing block as in question 1, §6.9; the new part is below it):

```kotlin
import com.spartanlabs.graphics.build.DOKKA_HTML_PUBLICATION
import com.spartanlabs.graphics.build.configureDokka
import com.vanniktech.maven.publish.JavaPlatform
import com.vanniktech.maven.publish.JavadocJar
import com.vanniktech.maven.publish.KotlinJvm
import com.vanniktech.maven.publish.KotlinMultiplatform
import com.vanniktech.maven.publish.SourcesJar

plugins {
    id("com.vanniktech.maven.publish")   // no Dokka here: the BOM applies this convention too
}

mavenPublishing {
    publishToMavenCentral()
    signAllPublications()
    coordinates(group.toString(), name, version.toString())
    pom { /* licence, developers, scm */ }
}

// One platform per module kind, pinned explicitly. Dokka reaches only the two Kotlin kinds.
pluginManager.withPlugin("org.jetbrains.kotlin.multiplatform") {
    configureDokka()
    mavenPublishing {
        configure(KotlinMultiplatform(javadocJar = JavadocJar.Dokka(DOKKA_HTML_PUBLICATION), sourcesJar = SourcesJar.Sources()))
    }
}
pluginManager.withPlugin("org.jetbrains.kotlin.jvm") {
    configureDokka()
    mavenPublishing {
        configure(KotlinJvm(javadocJar = JavadocJar.Dokka(DOKKA_HTML_PUBLICATION), sourcesJar = SourcesJar.Sources()))
    }
}
pluginManager.withPlugin("java-platform") {
    mavenPublishing { configure(JavaPlatform()) }   // pom packaging: no javadoc jar, no sources jar (question 5 owns the BOM)
}
```

The library conventions stay as in questions 1 to 3: each applies its Kotlin plugin and `spartangraphics.publishing`; neither mentions Dokka.
`withPlugin` fires immediately for an already-applied plugin and later for one applied afterwards, so the order of `plugins {}` entries does not
matter. The calls happen before vanniktech's `afterEvaluate` hook, which then no-ops.

Task names this creates:

| Module | Dokka tasks | Javadoc jar tasks (vanniktech) | Guard |
| --- | --- | --- | --- |
| `graphics-core` (KMP) | `dokkaGenerateModuleHtml`, `dokkaGeneratePublicationHtml` | `kotlinMultiplatformDokkaJavadocJar`, `jvmDokkaJavadocJar`, `jsDokkaJavadocJar` | `verifyDokkaSourceSets` (expects `commonMain`, `jsMain`, `jvmMain`) |
| `graphics-desktop` (JVM) | same | `dokkaJavadocJar` | `verifyDokkaSourceSets` (expects `main`) |
| `graphics-bom` | none | none | none |

### 6.4 Root `build.gradle.kts` (the plugins block is question 1, §6.5)

```kotlin
plugins {
    alias(libs.plugins.kotlin.multiplatform) apply false
    alias(libs.plugins.kotlin.jvm) apply false
    alias(libs.plugins.vanniktech.publish) apply false
    alias(libs.plugins.dokka)                 // applied: the root is the aggregate
}

dokka { moduleName = "SpartanGraphics" }

dependencies {
    dokka(project(":graphics-core"))
    dokka(project(":graphics-desktop"))       // add every new library module here; never graphics-bom
}
```

The aggregate task is `:dokkaGeneratePublicationHtml`, its output `build/dokka/html/` (`<module>/…` per module, an "All modules" index at the top).
The root applies no Kotlin plugin, so Dokka's adapter registers nothing there and prints no warning; the hoisted `apply false` lines are what let the
modules' adapters load `KotlinBasePlugin`. Optional footer, one place in the root and the same text in `Docs.kt`:

```kotlin
// root build.gradle.kts, optional
dokka { pluginsConfiguration.html { footerMessage = "© 2026 Spartan Laboratories" } }
// Docs.kt, optional (untested here): dokka.pluginsConfiguration.withType<DokkaHtmlPluginParameters>().configureEach { footerMessage.set("…") }
```

### 6.5 What CI runs

- Windows full build job: `./gradlew build :dokkaGeneratePublicationHtml javadocJars`
  - `build` runs `check`, which runs `verifyDokkaSourceSets` in both library modules (and the ABI and test checks of questions 3 and 6);
  - `:dokkaGeneratePublicationHtml` builds the aggregate and, through the modules' `dokkaGenerateModuleHtml`, runs the guard again;
  - `javadocJars` (an unqualified name, so it runs in every project that has it, never in the root or the BOM) builds every jar from the same Dokka
    publication, which exercises the vanniktech wiring.
  Uploading `build/dokka/html` as a workflow artifact is optional; nothing is hosted.
- Ubuntu and macOS compile jobs need none of this.
- G1's jars are expected to be manifest-only (F5). Nothing in CI should assert a page count in G1.

### 6.6 What the implementation must run (the wiring was not executed)

1. Fresh configuration: `./gradlew :graphics-core:verifyDokkaSourceSets :graphics-desktop:verifyDokkaSourceSets --no-configuration-cache` passes with no
   `Dokka could not load KotlinBasePlugin` warning. Add `--info` once and record the source-set names (expect `commonMain`, `jsMain`, `jvmMain` and `main`).
2. Negative run proving the guard bites: temporarily remove the two `apply false` Kotlin aliases from the root `plugins {}` block. The warning must
   appear and `verifyDokkaSourceSets` must fail with its message. Restore the lines.
3. `./gradlew :dokkaGeneratePublicationHtml javadocJars` succeeds in G1 with no Kotlin source; record the aggregate's file list and the sizes of the four
   javadoc jars (manifest-only is the expected result of F5). This is the one run that confirms F5 end to end, including K2 analysis of empty
   `commonMain`/`jvmMain`/`jsMain` source sets.
4. Wiring without running Dokka: `./gradlew :graphics-core:jvmDokkaJavadocJar --dry-run` lists `:graphics-core:dokkaGeneratePublicationHtml`
   (and `verifyDokkaSourceSets` before it). A TestKit variant of this assertion is a cheap Level 3 check (question 6).
5. Configuration cache: `./gradlew build :dokkaGeneratePublicationHtml --configuration-cache` twice; the second run reports the reused entry. Re-run
   step 2 on a second invocation too: a cache hit must still fail the guard. If the provider-based input cannot be stored, replace the lambda by a
   task-time read of a `@Input` list that the helper fills in `afterEvaluate`.
6. After the first declaration lands (G2): confirm the jars gain `index.html` and `<module>/…` pages, and then add the page-level assertion of F6
   (condition it on a non-empty ABI dump).
7. On Gradle 9.8.2, if chosen: confirm the only Dokka deprecation is `Configuration.setVisible`.

## 7. Load-bearing claims

Quotes from Kotlin sources and Markdown omit nothing but markup characters. "Measured" claims quote the listing that was read.

1. **Dokka 2.2.0 uses the V2 plugin by default.** "Starting from Dokka 2.1.0, DGP v2 is enabled by default."
   [dokka-migration.md @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/docs/topics/dokka-migration.md); and in
   [PluginFeaturesService.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/internal/PluginFeaturesService.kt):
   "/** The default value, if [Params.pluginMode] is not set. */ val Default: PluginMode = V2EnabledWithHelpers".
2. **`noWarn` affects only the V1 message.** Same file: "private fun logV1PluginMessage() { if (primaryService && !pluginModeNoWarn) {" and, for V2,
   "V2EnabledWithHelpers -> { // this is the default, don't log messages to avoid spamming users".
3. **`V2EnabledWithHelpers` registers disabled V1 shims.**
   [v2MigrationUtils.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/internal/v2MigrationUtils.kt):
   "Create the same tasks as V1, but disable them and update the description." and `notCompatibleWithConfigurationCache("Dokka V1 tasks use deprecated Gradle features.`
4. **Dokka's Javadoc format does not support KMP.**
   [dokka-javadoc.md @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/docs/topics/formats/dokka-javadoc.md): "Dokka doesn't support the Javadoc format for
   multi-project builds or Kotlin Multiplatform projects."
5. **vanniktech 0.37.0 auto-detects Dokka V2 and picks the HTML task.**
   [MavenPublishBaseExtension.kt @ 0.37.0](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/plugin/src/main/kotlin/com/vanniktech/maven/publish/MavenPublishBaseExtension.kt),
   `defaultJavaDocOption`: "project.plugins.hasPlugin("org.jetbrains.dokka") -> { // only dokka v2 has an extension check(project.extensions.findByName("dokka") != null) {
   "Dokka in v2 mode is required when using Dokka" } JavadocJar.Dokka("dokkaGeneratePublicationHtml")"; and
   [MavenPublishPlugin.kt @ 0.37.0](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/plugin/src/main/kotlin/com/vanniktech/maven/publish/MavenPublishPlugin.kt):
   "// will no-op if it was already called baseExtension.configureBasedOnAppliedPlugins()".
6. **The platform classes and their defaults.**
   [Platform.kt @ 0.37.0](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/plugin/src/main/kotlin/com/vanniktech/maven/publish/Platform.kt):
   "public data class KotlinMultiplatform @JvmOverloads constructor( override val javadocJar: JavadocJar = JavadocJar.Empty(), override val sourcesJar: SourcesJar = SourcesJar.Sources(),",
   "public data class KotlinJvm @JvmOverloads constructor(", and for `JavaPlatform`: "override val javadocJar: JavadocJar = JavadocJar.None()".
7. **`JavadocJar.Dokka` takes the task name, and the jar copies that task's outputs, one jar per KMP publication.** Same file:
   "Creates a javadoc jar using Dokka's output. The argument is the name of the dokka task that should be used for that purpose.";
   "it.withJavadocJar(javadocJar, project, multipleTasks = true)" (in `KotlinMultiplatform.configure`); and
   [tasks/JavadocJar.kt @ 0.37.0](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/plugin/src/main/kotlin/com/vanniktech/maven/publish/tasks/JavadocJar.kt):
   "tasks.register(prefixedTaskName("dokkaJavadocJar", prefix), JavadocJar::class.java) { val dokkaTask = dokkaTaskWrapper.asProvider(project) it.dependsOn(dokkaTask) it.from(dokkaTask)".
8. **`dokkaGeneratePublicationHtml` has an output directory; `dokkaGenerate` does not.**
   [dokka-html.md @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/docs/topics/formats/dokka-html.md): "`dokkaGeneratePublicationHtml` to generate documentation only in HTML format.
   This task exposes the output directory as an `@OutputDirectory`. Use this task when you need to consume the generated files in other Gradle tasks, such as
   uploading them to a server, moving them into a GitHub Pages directory, or packaging them into a `javadoc.jar`."; and
   [DokkaBasePlugin.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/DokkaBasePlugin.kt):
   "target.tasks.register<DokkaBaseTask>(taskNames.generate) { description = "Generates Dokka publications for all formats" dependsOn(target.tasks.withType<DokkaGeneratePublicationTask>())";
   [DokkaBaseTask.kt](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/tasks/DokkaBaseTask.kt): "constructor() : DefaultTask() {".
9. **Coil 3.6.3 passes the lifecycle task and ships empty KMP javadoc jars.**
   [projects.kt @ 3.6.3](https://github.com/coil-kt/coil/blob/3.6.3/buildSrc/src/main/kotlin/coil3/projects.kt): "configure(KotlinMultiplatform(Dokka("dokkaGenerate")))". Measured 2026-10-20:
   `https://repo1.maven.org/maven2/io/coil-kt/coil3/coil-core/3.6.3/coil-core-3.6.3-javadoc.jar` is 261 bytes, listing
   "25 1980-02-01 00:00 META-INF/MANIFEST.MF … 2 files"; `coil-core-jvm-3.6.3-javadoc.jar` is identical.
10. **workflow-kotlin wires KMP and JVM to the HTML task and publishes populated jars.**
    [PublishingConventionPlugin.kt @ 40d22c9](https://github.com/square/workflow-kotlin/blob/40d22c9eb443adb0b00bbf653e6fab7898ccf052/build-logic/src/main/java/com/squareup/workflow1/buildsrc/PublishingConventionPlugin.kt):
    "target.pluginManager.withPlugin("org.jetbrains.kotlin.multiplatform") { basePluginExtension.configure( KotlinMultiplatform( javadocJar = JavadocJar.Dokka(taskName = "dokkaGeneratePublicationHtml") )".
    Measured 2026-10-20: `https://repo1.maven.org/maven2/com/squareup/workflow1/workflow-core{,-jvm,-js}/1.31.0/…-javadoc.jar` are each 1,031,440 bytes with 250 `.html` entries including
    "index.html", "navigation.html", "workflow-core/package-list".
11. **vanniktech's docs show a Dokka v1 task name.**
    [docs/what.md @ 0.37.0](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/docs/what.md): "`JavadocJar.Dokka("dokkaHtml")` when using Kotlin with Dokka, where `dokkaHtml` is the name of the Dokka task that should be used as input".
12. **Dokka registers a source set per Kotlin source set, and only warns when it cannot load the Kotlin plugin.**
    [KotlinAdapter.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/adapters/KotlinAdapter.kt):
    "sourceSetDetails.all details@{ dokkaExtension.dokkaSourceSets.register( projectPath = projectPath, details = this@details, ) }",
    "suppress.convention(!details.isPublishedSourceSet())", and "If so, DGP will be lenient and not cause errors, but it must display a prominent warning to help users find the problem."
13. **No Dokka source set means no documentation.**
    [DokkaExtension.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/DokkaExtension.kt): "Dokka will not generate documentation unless there is at least one Dokka Source Set."
14. **Java source sets are suppressed by plugin id when Kotlin is applied.**
    [JavaAdapter.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/adapters/JavaAdapter.kt):
    "suppress.convention(!src.isPublished() or isConflictingPluginPresent)" and "PluginIds.kotlin.any { project.pluginManager.hasPlugin(it) }".
15. **An empty module is dropped and generation exits gracefully.**
    [EmptyModulesFilterTransformer.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-subprojects/plugin-base/src/main/kotlin/org/jetbrains/dokka/base/transformers/documentables/EmptyModulesFilterTransformer.kt): "return modules.filter { it.children.isNotEmpty() }";
    [SingleModuleGeneration.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-subprojects/plugin-base/src/main/kotlin/org/jetbrains/dokka/base/generation/SingleModuleGeneration.kt): "?: exitGenerationGracefully("Nothing to document")";
    [DokkaGenerator.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-subprojects/core/src/main/kotlin/org/jetbrains/dokka/DokkaGenerator.kt): "} catch (exit: GracefulGenerationExit) { report("Exiting Generation: ${exit.reason}")";
    [AbortGracefullyOnMissingDocumentablesTest.kt](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-subprojects/plugin-base/src/test/kotlin/basic/AbortGracefullyOnMissingDocumentablesTest.kt): "Generation aborts Gracefully with no Documentables".
16. **The Gradle adapter logs the graceful exit at info level; `failOnWarning` counts only generator warnings.**
    [LoggerAdapter.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/internal/LoggerAdapter.kt): "PROGRESS, INFO -> logger.info("[$logTag] " + message.prependIndent().trimStart())";
    SingleModuleGeneration.kt (above): "if (context.configuration.failOnWarning && (context.logger.warningsCount > 0 || context.logger.errorsCount > 0)) {".
17. **Output directories exist and aggregation tolerates empty modules.**
    [DokkaGeneratorWorker.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/workers/DokkaGeneratorWorker.kt): "dokkaParameters.outputDir.deleteRecursively() dokkaParameters.outputDir.mkdirs()";
    [TemplateProcessor.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-subprojects/plugin-templating/src/main/kotlin/org/jetbrains/dokka/templates/TemplateProcessor.kt): "?.takeIf { files.firstOrNull { !it.startsWith(".") } != null }";
    [MultimodulePageCreator.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-subprojects/plugin-all-modules-page/src/main/kotlin/org/jetbrains/dokka/allModulesPage/MultimodulePageCreator.kt): "modules.filter { it.name in creationContext.nonEmptyModules }.sortedBy { it.name }".
18. **The Kotlin Analysis API maps an empty source module to an empty file list.**
    [StandaloneAnalysisAPISessionBuilderImpl.kt @ 8d016a6 (master)](https://github.com/JetBrains/kotlin/blob/8d016a647d9ae86e6c137b69da1409632d84520d/analysis/analysis-api-standalone/analysis-api-standalone-fir/src/org/jetbrains/kotlin/analysis/api/standalone/fir/StandaloneAnalysisAPISessionBuilderImpl.kt):
    "ktModule to ktModule.psiRoots.filterIsInstance<PsiFile>()"; and the consumer, Dokka's
    [DefaultSymbolToDocumentableTranslator.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-subprojects/analysis-kotlin-symbols/src/main/kotlin/org/jetbrains/dokka/analysis/kotlin/symbols/translators/DefaultSymbolToDocumentableTranslator.kt):
    "?: throw IllegalStateException("No source files for a source module".
19. **Sonatype's javadoc rule and placeholder allowance.** "Projects with packaging other than `pom` have to supply JAR files that contain Javadoc and sources." and
    "If you cannot provide sources or documentation, we allow uploading placeholder `.jar` files in order to pass validation."
    [Sonatype Central requirements](https://central.sonatype.org/publish/requirements/), read 2026-10-20.
20. **Source-link form in comparable builds.** Ktor 3.6.0 `ktorbuild.dokka.gradle.kts`: "localDirectory = rootDir remoteUrl("https://github.com/ktorio/ktor/blob/$version")";
    kotlinx-io 0.9.1 `kotlinx-io-dokka.gradle.kts`: "localDirectory = rootDir remoteUrl("https://github.com/kotlin/kotlinx-io/tree/master") remoteLineSuffix = "#L"".
    [Ktor](https://github.com/ktorio/ktor/blob/3.6.0/build-logic/src/main/kotlin/ktorbuild.dokka.gradle.kts),
    [kotlinx-io](https://github.com/Kotlin/kotlinx-io/blob/0.9.1/build-logic/src/main/kotlin/kotlinx/io/conventions/kotlinx-io-dokka.gradle.kts).
    Dokka's [DokkaSourceLinkSpec.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/engine/parameters/DokkaSourceLinkSpec.kt): "Default is `#L`."
21. **Dokka defaults that matter.** [DokkaBasePlugin.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/DokkaBasePlugin.kt):
    "jdkVersion.convention(11)", "maybeCreate("kotlinStdlib") { enabled.set(this@dss.enableKotlinStdLibDocumentationLink) url("https://kotlinlang.org/api/core/") }", and
    "localDirectory.convention(layout.projectDirectory)"; footer: [DokkaBaseConfiguration.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-subprojects/plugin-base/src/main/kotlin/org/jetbrains/dokka/base/DokkaBaseConfiguration.kt):
    "public val defaultFooterMessage: String = "© ${Year.now().value} Copyright"".
22. **Dokka's HTML needs a web server.** dokka-html.md @ v2.2.0: "HTML pages generated by this format need to be hosted on a web server in order to render everything correctly."
23. **Configuration-cache support and the 2.2.0 fix.** [dokka-gradle.md @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/docs/topics/runners/dokka-gradle.md): "DGP supports Gradle build cache and configuration cache, improving build performance.";
    [release v2.2.0, 2026-03-26](https://github.com/Kotlin/dokka/releases/tag/v2.2.0): "Fix DGP reads all Gradle properties, which causes unnecessary CC invalidation ([#4467]".
24. **The Gradle 9.8 deprecation and its unreleased fix.** [Kotlin/dokka#4600](https://github.com/Kotlin/dokka/issues/4600): "The Configuration.setVisible(boolean) method has been deprecated. This is scheduled to be removed in Gradle 11."
    (Dokka 2.2.0, Gradle `9.8.0-rc-1`); [PR #4601](https://github.com/Kotlin/dokka/pull/4601), title "Drop usages of `Configuration.isVisible` on Gradle 9+ as it's no-op there, as well as it's deprecated for removal",
    merged 2026-09-11, commit `cba8250107ca3d9eab4ff3a9047a9070e9be1d0f`. Measured 2026-10-20: GitHub's compare of that commit against tag `v2.3.0-Beta` reports status "diverged" (the tag does not contain it),
    and `v2.3.0-Beta`'s `gradleUtils.kt` still reads "canBeDeclared(true) isVisible = visible".
25. **Latest stable releases.** [Maven Central metadata](https://repo1.maven.org/maven2/org/jetbrains/dokka/dokka-gradle-plugin/maven-metadata.xml), read 2026-10-20: versions "2.2.0", "2.3.0-Beta" (`<latest>2.3.0-Beta</latest>`);
    [vanniktech](https://repo1.maven.org/maven2/com/vanniktech/gradle-maven-publish-plugin/maven-metadata.xml): "<release>0.37.0</release>".
26. **Precompiled-script users must give Dokka the Kotlin plugin.** dokka-gradle.md @ v2.2.0: "If you are using Dokka in a precompiled script plugin, add the Kotlin Gradle plugin as a dependency to ensure it works properly."
27. **Reused, not re-verified here:** the root `plugins {}` hoist as the remedy for the classloader failure and the MyGameTools 0-to-415 measurement (question 1, claims 13 and 14); Dokka's CI tested on Gradle 7.6.3 and 8.14.3 only
    and the MyGameTools, Ktor and Coil pairings on Gradle 9.7.1 (question 2, claims 9 and 31); vanniktech's configuration-cache testing and Okio's 261-byte javadoc jars accepted by Central
    ([GeneralTools `kmp-maven-central-publishing.md` @ d424aaf](https://github.com/SpartanLaboratories/GeneralTools/blob/d424aaf/docs/plans/7-kotlin-multiplatform/research/kmp-maven-central-publishing.md)).

## 8. Gaps

- **Nothing was executed.** F5 (an empty module generates without failing, with an empty output directory) is established from the generator's control flow, a Dokka
  unit test of the graceful exit, and the aggregation code; the end-to-end run on empty `commonMain`/`jvmMain`/`jsMain` and `main` source sets is step 3 of 6.6. If
  K2 analysis raises on a source module without files, Dokka would fail the build for an empty module; the fallback is to `suppress.set(true)` on source sets with
  no Kotlin file until the first declaration (which would also empty the guard's input, so the guard would have to read the *registered* names instead of the
  documented ones for that interval).
- **Kotlin Analysis API evidence is from `master`.** The `psiRoots` mapping (claim 18) was read at a JetBrains/kotlin commit of 2026-10-09; Dokka 2.2.0 embeds the
  Kotlin analysis `2.3.20-dev-7064`, and the file is absent at the Kotlin tags tried (`v2.3.20`, `v2.4.21`) under that path. The shape (`mapNotNull … toMap()` over
  source modules) is simple, but the pinned version was not read.
- **Why Coil's jars are empty** is inferred (lifecycle task without outputs); a classloader cause was not excluded by running its build. The measurement (261-byte
  jars) is solid; the cause is not.
- **Configuration-cache storage of the guard's provider input** is inferred from how Gradle stores task inputs; step 5 of 6.6 settles it. Isolated Projects
  compatibility of Dokka 2.2.0 was not checked (question 1 gap); the helper uses `isolated.rootProject.projectDirectory`, as MyGameTools does.
- **Root-hoist classloader mechanism** remains the inference recorded in question 1; the guard and the negative run (6.6 step 2) are its test from Dokka's side.
- **`js` analysis without Node** is inferred from the adapter's use of the compilation classpath; not run. The first CI run on a clean cache shows whether the `js`
  compilation classpath resolution needs anything beyond Maven Central (the build's settings repository).
- **Offline generation** (external documentation links fetched from `docs.oracle.com` and `kotlinlang.org`) was not examined for its failure mode; `offlineMode`
  exists on the publication and is not set.
- **Not decided here because there is nothing to decide in G1:** `failOnWarning` (left at its default `false`), `reportUndocumented`, `Module.md` includes, per-package
  options, and the page-level assertion for populated jars. Each belongs with the first public declarations (G2) and the release runbook (G7).
- **Source links to `main`** drift after release; a tag-based `remoteUrl` for non-snapshot versions is a small later change.
- **Styling** options are reported with cost; no theme is designed. The `DokkaHtmlPluginParameters` call in the optional `Docs.kt` footer line was not compiled.
- **Considerations:** none that block; the choices in section 6 are made on the evidence above.
