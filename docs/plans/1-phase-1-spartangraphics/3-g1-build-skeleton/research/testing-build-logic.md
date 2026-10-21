# Testing the G1 build logic across the five levels, and what it costs in CI

- **Issue:** [SpartanLaboratories/SpartanGraphics#3](https://github.com/SpartanLaboratories/SpartanGraphics/issues/3), question 6 of the
  [research README](README.md).
- **Read:** 2026-10-20. Pinned sources: Gradle `v9.8.2` (docs, TestKit, plugin-use and composite-build sources and tests), Kotlin
  `v2.4.21` (Kotlin Gradle plugin sources), vanniktech `0.37.0`, `gradle/actions` `v6.4.0`, MyGameTools `dc5214f` (local clone and
  GitHub Actions run 38693939100), GeneralTools branch `feat/issue-7-kotlin-multiplatform` (GitHub Actions run 39507508729),
  Ktor `3.6.0`, Okio `parent-3.18.2`, Coil `3.6.3`, kotlinx-io `0.9.1`, workflow-kotlin `40d22c9eb443`.
- **Method limit:** nothing was built or run: no Gradle task, in this repository or any other. Every behaviour below comes from
  documentation, source and tests at the pinned tags, a local read of MyGameTools' `build-logic`, and the timestamps of two finished
  GitHub Actions runs. Timings for this repository's tests are estimates derived from those two runs and are labelled as such.

## 1. Question

How does a build with no Kotlin product source test its own build logic (the `build-logic` convention plugins
`spartangraphics.kmp-library`, `spartangraphics.jvm-library` and `spartangraphics.publishing`, the typed helpers in
`com.spartanlabs.graphics.build`, the root `plugins {}` hoist, the BOM and the Dokka guard) at the right levels of the five-level
hierarchy, where does each test live and run, and what does it cost in CI? An answer has to: put every test in
`com.spartanlabs.testing.<level>.graphics.build` with one class per file; check every behaviour the conventions promise (strict explicit
API, ABI checks and empty dumps, the `-PtestLevel` filter, BOM alignment and coverage, the Kotlin 2.2 floor and stdlib pins in published
metadata, the JVM 21 level and `TargetJvmVersion`, `-jvm-default=enable`, the Dokka source-set guard and its negative control, and the
Gradle 9.0.0 composite-consumer floor), automatically or by a named manual step; keep heavy tests off the modules' `check`; and keep CI
time proportionate to a skeleton.

## 2. Bottom line

- **Five mechanisms, by cost.** (1) The three real-build guards (`checkKotlinAbi`, `verifyDokkaSourceSets`, `verifyBomCoverage`) plus
  Gradle's own explicit-API compile are Level 1 gates inside every `check`: positive only, free. (2) Pure functions are Level 4a
  (`parseTestLevels`, `bomCoverageProblems`, the floor parser, the filter-pattern builder) and Gradle-object classes are Level 2
  (`ProjectBuilder`, `@TempDir`, MockK): JUnit 5 in `build-logic/src/test/kotlin`, seconds, part of `:build-logic:check`. (3) Level 3
  is TestKit against a **copy of the real build**, not against fixtures. (4) Level 4b is one TestKit test: a composite consumer on
  Gradle 9.0.0 and Kotlin Gradle plugin 2.1.20 that includes the copy. (5) Level 4c is the configuration-cache store-and-reuse pair. Level 5
  is a recorded manual checklist.
- **Do not use `withPluginClasspath()` for this build.** It puts every plugin of the injected classpath in one classloader, so the
  arrangement the root hoist exists to create is always true under it, and "remove the hoist" can never be reproduced. A copy of the
  real build, with `pluginManagement { includeBuild("build-logic") }` intact, loads plugins exactly as a developer's build does; it also
  needs no catalog duplication, so the tests run the files consumers include.
- **Keep the heavy tiers off the default tasks and name them.** `:build-logic:test` (levels 1, 2, 4a) rides on `:build-logic:check`;
  `integrationTest`, `e2eTest` and `nonfunctionalTest` are separate `Test` tasks that select their level by package, are never
  `UP-TO-DATE`, and are not wired into `check`. The root `build` does not run any of them (Gradle does not run an included build's tests
  from the root).
- **The js target costs nothing if the tests avoid `jsNodeTest`.** Node.js and Yarn are installed into `<Gradle user home>/nodejs` and, from
  the dependency declarations, are prerequisites of the js *test* task only; klib compilation, `checkKotlinAbi`, metadata generation and
  Dokka do not need them (test 10 asserts it on the first run). Nested runs name their tasks and never run `check`, `build`, `test` or
  `allTests` on `graphics-core`; `check --dry-run` is safe. The js test filter is checked by reading the configured `TestFilter` of every
  `AbstractTestTask`, not by executing Node.
- **CI: one parallel job, `build-logic (TestKit)`, on `windows-latest`; the unit tiers ride on the existing full-build job.** Estimated
  8 to 14 minutes cold for the job (3 to 5 of them for the Gradle 9.0.0 consumer), from two measured data points (below). The
  separate Gradle 9.0.0 workflow job proposed in the toolchain finding is replaced by `CompositeConsumerFloorTest`, which proves more.
  Nested builds on CI share the runner's Gradle user home so the existing `setup-gradle` cache covers them.
- **Ten checks stay manual** (section 6.7): two one-off experiments that become tests once run (the hoist removal, the 9.0.0
  `build-logic` assemble), the real-client composite copies, the js filter execution at G2, Isolated Projects, warnings at bumps, the
  maintainer's `core.autocrlf` check, and the two judgements only a person makes (rendered Dokka site, IDE import).

## 3. Findings

Tiers: **consensus** = official documentation, source or a Gradle/Kotlin test at the pinned tag; **practice** = what mature builds visibly
do; **opinion** = one source; **inference** = derived here from sources that do not state it.

### F1. Gradle's own guidance: functional-test build logic with TestKit, keep it separate and explicit (consensus; Gradle 9.8.2)

- "A mature build should include functional tests for its custom types", and "Because TestKit tests tend to be slower and more complex
  than unit tests, they are typically kept separate." Gradle's best-practice sample is a `build-logic` included build with a
  `functionalTest` suite ([best_practices_testing.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/best-practices/best_practices_testing.adoc)).
- The same page states the wiring consequence for this layout: "By default, tests within the included `build-logic` build are not
  executed when you run tests in the root project", and they run through `./gradlew :build-logic:check` or `:build-logic:functionalTest`.
  Gradle's own test confirms the path syntax for a `pluginManagement`-included build: `PluginBuildsIntegrationTest`, "can execute a task
  from included plugin build", declares `pluginManagement { includeBuild(...) }` and asserts `succeeds(":$pluginBuild:jar")`
  ([source](https://github.com/gradle/gradle/blob/v9.8.2/subprojects/composite-builds/src/integTest/groovy/org/gradle/integtests/composite/plugins/PluginBuildsIntegrationTest.groovy)).
  Running the build-logic project on its own with `-p build-logic` is the other form; both read its `settings.gradle.kts`.
- Gradle's sample adds the functional suite to `check` (`tasks.check { dependsOn(functionalTest) }`) because its fixtures are tiny. Real-build
  tests are not tiny (F10), so this repository keeps them off `check`, as MyGameTools does.
- The unit tooling is named in the plugin-testing chapter: "Unit tests | Any JVM-based test framework + `ProjectBuilder`"
  ([testing_gradle_plugins.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/plugin-development/testing_gradle_plugins.adoc)).
  `ProjectBuilder`'s Javadoc calls its product "dummy instances of Project"
  ([ProjectBuilder](https://github.com/gradle/gradle/blob/v9.8.2/platforms/extensibility/unit-test-fixtures/src/main/java/org/gradle/testfixtures/ProjectBuilder.java)).
  It builds a project without evaluating a `settings.gradle.kts`, so by inference it has no `libs` catalog, and the convention scripts,
  which read `catalog` at apply time, cannot be applied under it. (Inference; the Javadoc does not mention catalogs.)

### F2. `withPluginClasspath()` loads the injected plugins into one classloader (consensus on the code; inference on the consequence)

- `kotlin-dsl` applies `java-gradle-plugin` ("Applies the `java-gradle-plugin` plugin", `KotlinDslPlugin` KDoc), which "automatically adds
  the `gradleTestKit()` dependency to the `testImplementation` configuration and generates a plugin classpath manifest file consumed by a
  `GradleRunner` instance if found" ([java_gradle_plugin.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/core-plugins/java_gradle_plugin.adoc)).
  So a `kotlin-dsl` build-logic needs no extra TestKit wiring, and `withPluginClasspath()` works with no arguments.
- Mechanism. `DefaultInjectedClasspathPluginResolver` builds one registry over `parentScope.createChild("injected-plugin", null).local(classPath).lock()`
  and, for a requested plugin id, resolves the plugin from that registry and exports its classloader to the requesting project
  (`InjectedClasspathPluginResolution.accept` calls `visitor.visitClassLoader(plugin.asClass().getClassLoader())`;
  [source](https://github.com/gradle/gradle/blob/v9.8.2/platforms/extensibility/plugin-use/src/main/java/org/gradle/plugin/use/resolve/service/internal/DefaultInjectedClasspathPluginResolver.java)).
  Gradle's test `GradleRunnerPluginClasspathInjectionIntegrationTest` pins the observable result: "injected classes are loaded only once"
  (two child projects applying the plugin get the identical task class) and "plugin applied via injection can apply another plugin from
  its implementation classpath"
  ([source](https://github.com/gradle/gradle/blob/v9.8.2/platforms/extensibility/test-kit/src/integTest/groovy/org/gradle/testkit/runner/GradleRunnerPluginClasspathInjectionIntegrationTest.groovy)).
- Consequence for this build (inference from the two sources above; not run). The failure the root hoist prevents is Kotlin Gradle plugin
  classes living in per-module classloaders while Dokka lives in the root's. Under injection every plugin of the build-logic classpath
  (Kotlin, Dokka, vanniktech and the conventions) comes from the one "injected-plugin" loader, whether or not the root hoists it. The hoist
  is therefore invisible under `withPluginClasspath()`: the Dokka guard would pass with the hoist removed, and the negative control would
  report nothing.
- Further limits: "Automatic classpath injection only works when the plugin-under-test is applied in the test build using the
  `plugins {}` DSL", and it is "incompatible with plugins that use `compileOnly` to depend on another plugin's API" (this build's
  `build-logic` uses `implementation`, so the second limit does not bite)
  ([test_kit.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/plugin-development/test_kit.adoc)).
  A fixture project would also need its own `settings.gradle.kts` catalog import and a copy of the root block.

### F3. The TestKit environment: isolated home, cached distributions, version matrix (consensus; Gradle 9.8.2)

- The test build "does not inherit settings, caches, or environment customizations from the machine running the tests"; "Configuration
  from the default Gradle User Home (for example, `~/.gradle/gradle.properties`) is **not** used"; TestKit "uses dedicated Gradle daemon processes"
  (test_kit.adoc, section "Controlling the build environment").
- `GradleRunner.withTestKitDir`: the directory is TestKit's working storage and "If no explicit Gradle user home is specified via the build
  arguments (i.e. the `-g «dir»` option), this directory will also be used for the Gradle user home for the test build"; unspecified, it is
  created in a temporary directory ("When executed from a Gradle Test task, the Test task's temporary directory is used"), and it can be
  set with the system property `org.gradle.testkit.dir`
  ([GradleRunner.java](https://github.com/gradle/gradle/blob/v9.8.2/platforms/extensibility/test-kit/src/main/java/org/gradle/testkit/runner/GradleRunner.java)).
  `ToolingApiGradleExecutor.buildConnector` keeps the test daemons under the TestKit directory (`daemonBaseDir(new File(gradleUserHome, TEST_KIT_DAEMON_DIR_NAME))`,
  where `DefaultGradleRunner` passes the TestKit directory as `gradleUserHome`) even when `-g` is passed as a build argument, and takes
  distributions from the real user home (`useDistributionBaseDir(GradleUserHomeLookup.gradleUserHome())`).
- `withGradleVersion(v)` "will cause the Gradle runtime for the version specified to be downloaded over the Internet from Gradle's
  distribution servers", cached "beneath the Gradle User Home directory" (the real one: system property, then `GRADLE_USER_HOME`, then
  `~/.gradle`). The Tooling API range is wide: "the Tooling API 9.0.0 release is compatible with Gradle versions >= 4.0"
  ([tooling_api.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/integration/tooling_api.adoc)),
  so a 9.7.1 test JVM can drive a 9.0.0 build. The JDK of the nested build is not selectable ("The TestKit does not currently expose
  fine-grained control over all environmental aspects—for example, selecting a JDK"); a Gradle 9.0.0 daemon needs a JVM 17 to 24
  (the compatibility table lists Java 25 from Gradle 9.1.0;
  [compatibility.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/releases/compatibility.adoc)).
  CI's JDK 21 and the client's JDK 23 are inside the range.
- vanniktech's tests do not isolate cheaply either: each test passes `withTestKitDir(temp.resolve("test-kit-dir"))` (fresh home per test)
  unless a quick-test flag is set, and its fixtures set `kotlin.compiler.execution.strategy=in-process` and `org.gradle.vfs.watch=false`
  (`ProjectSpecRunner.run` and `writeGradleProperties` at `0.37.0`), which keeps Kotlin compile daemons from outliving the test JVM.

### F4. Kotlin/JS: where Node.js lands and what triggers it (consensus on the code; inference on the Node cost per task)

- `BaseNodeJsRootExtension` (Kotlin Gradle plugin `v2.4.21`) initialises `private val gradleHome = project.gradle.gradleUserHomeDir` and
  `var installationDir: File = gradleHome.resolve("nodejs")`; `NodeJsPluginApplier` makes that the convention of
  `NodeJsEnvSpec.installationDirectory`. Under TestKit, `gradleUserHomeDir` is the TestKit directory (or the `-g` directory), so Node
  downloads once per TestKit home.
- The setup is wired to the js **test** task: `KotlinNodeJsIr.configureTestDependencies` does `test.dependsOn(project.nodeJsSetupTaskProvider)`,
  then `test.dependsOn(nodeJsRoot.npmInstallTaskProvider, ...)`. Nothing in `KotlinJsIrTargetConfigurator` or `KotlinNodeJsIr` adds those
  dependencies to the klib compile, and the Kotlin Gradle plugin's `KotlinJsIrTarget` only applies the npm resolver plugins. (Inference
  from the dependency declarations: the klib compile, the klib ABI dump, metadata generation and Dokka do not run `kotlinNodeJsSetup`.) A
  NO-SOURCE test task still runs its dependencies, so `check` or `build` on `graphics-core` does download Node and Yarn on a cold home even
  with no tests (inference).
- Observed in a finished run: GeneralTools' `./gradlew build` on `ubuntu-latest` (run 39507508729, KMP `jvm` + `js` + `wasmJs`) finished
  `compileKotlinJs` at 22:22:38, started `kotlinNodeJsSetup` at 22:22:39 and `kotlinNpmInstall` at 22:22:43, after the package.json tasks
  (`jsPackageJson`, 22:22:29) that only write files; the wasm tooling setup took 25 seconds (22:22:45 to 22:23:10); the whole step ended
  `BUILD SUCCESSFUL in 1m 51s`, of which 45 seconds elapsed between `Starting a Gradle Daemon` and the first task. The order is consistent with
  the inference (the compile did not wait for Node), but parallel execution makes it no proof; the state of that run's Gradle home cache is
  unknown, so the setup durations are a floor, not a cold-download measurement.
- So `js` can stay in the copied build; only the task selection keeps Node out. The js `TestFilter` can be read without Node (F6).

### F5. Running and caching: which Gradle home the nested builds use (consensus for the facts; the CI recommendation is a judgement)

- `gradle/actions` `v6.4.0` `setup-gradle`: "By default, The `setup-gradle` action will only write to the cache from Jobs on the default
  (`main`/`master`) branch", and it saves and restores "any wrapper distributions" and "the `caches` and `notifications` directories from
  Gradle User Home"; other directories (it names `jdks`) need `gradle-home-cache-includes`; it warns against a second mechanism that caches
  the Gradle user home
  ([setup-gradle.md](https://github.com/gradle/actions/blob/v6.4.0/docs/setup-gradle.md)).
- TestKit's default home (F3) is outside that cache, so every CI run of an isolated-home suite re-downloads the Kotlin, Dokka and
  vanniktech plugin dependencies and the JDK/Node/Yarn extras. Passing `-g <real Gradle user home>` to the nested builds puts them under
  the cache the job already restores. The cost of sharing is the loss of isolation: the real `~/.gradle/gradle.properties` is read, which
  matters on a developer machine (a stray `version=` there overrides the repository's, build-logic finding F5) and not on a runner.

### F6. What a test can read without running the slow part (consensus; Gradle 9.8.2, Kotlin `v2.4.21`)

- `KotlinTest` (`abstract class KotlinTest ... : AbstractTestTask()`, KGP `v2.4.21`) is the base of `KotlinJsTest`, so `jsNodeTest` is an
  `AbstractTestTask` with Gradle's `filter`. `TestFilter.getIncludePatterns()` ("Returns the included test name patterns") and
  `isFailOnNoMatchingTests()` ("The default is true") are readable at configuration time. A probe that prints, for every
  `AbstractTestTask`, `path`, sorted `includePatterns` and `failOnNoMatchingTests` shows the filter reached the js test task without
  running Node (the execution of a package pattern on Mocha stays from-source in the reused research and is exercised from G2, section 6.7).
- Configured compiler options are readable the same way: `KotlinCompilationTask.compilerOptions` carries `languageVersion`, `apiVersion`,
  and for JVM `jvmTarget` and `jvmDefault`. With no source the compile tasks are `NO-SOURCE`, so their command-line arguments are never
  printed even under `--debug`; the configured options are the only observable. (The first half of the sentence is the plugin's API; the
  NO-SOURCE consequence is inference from the toolchain finding's note that the arguments "show only under `--debug`" with sources.)
- Published metadata is observable without publishing: `generateMetadataFileFor<Publication>Publication` and
  `generatePomFileFor<Publication>Publication` write `module.json` and the POM under the module's `build/` directory (toolchain finding
  6.5, kmp-bom finding 6.6).

### F7. Configuration cache: Gradle prescribes two runs (consensus; Gradle 9.8.2)

"A single run is not enough to verify Configuration Cache compatibility: a build can succeed under CC simply because the entry was stored,
while a silent CC incompatibility only surfaces on the second run"; the documented TestKit pattern runs `--configuration-cache myTask`
twice and asserts `result.output.contains("Reusing configuration cache.")`
([configuration_cache_debugging.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/optimizing-builds/configuration-cache/configuration_cache_debugging.adoc),
section "Test your Build Logic", and its sample `BuildLogicFunctionalTest.kt`).

### F8. Comparable projects mostly do not test their build logic; the exceptions run real builds (practice; mixed tiers)

Listed in section 4. The common practice of large KMP libraries is no build-logic tests at all; the organisation's own MyGameTools and the
vanniktech plugin are the two reads that test build logic with TestKit, and neither uses `withPluginClasspath()`.

### F9. The `kotlin-dsl` and SLF4J pitfall is about applying `kotlin-dsl` to a library (consensus on the record; scope clarified here)

MyGameTools' plan records the concern precisely: `kotlin-dsl` "puts `gradleApi()` (and its bundled slf4j provider) on `build-logic`'s
*own* compile classpath. The root build's existing warning is about that provider reaching a **library module's test runtime** and
silencing logback" (`docs/plans/32-module-split/plan.md`, section on `build-logic` and the `kotlin-dsl` slf4j concern); GeneralTools
records the same reason for not applying `kotlin-dsl` to its single module (`plan-kmp-build.md`, comment in the build script's `plugins`
block). In this repository `kotlin-dsl` lives only in `build-logic`, which no library module's classpath includes. The `build-logic` test
runtime has `gradleApi()`/`gradleTestKit()` and their SLF4J provider and no logback, so nothing conflicts; the rule that follows is not
to add logback or kotlin-logging to `build-logic`'s test dependencies.

### F10. MyGameTools' two TestKit styles, read from the code (practice; `dc5214f`)

- `UmbrellaArtifactAggregationTest` (package `gametools.testing.integration`) runs one nested build in `@BeforeAll` against the **real
  repository directory** (`GradleRunner.create().withProjectDir(repoRoot)`, `repoRoot` from the system property `gametools.repoRoot`) and
  asserts on the jars it wrote. That makes it a positive-only test that mutates the developer's `build/` directories, and it cannot
  remove a plugin from the root block.
- `UmbrellaVerifyTaskContractTest` tests the guard's negative direction in a `@TempDir` throwaway project that carries a **copy** of the
  production `check(...)` text, and a third test, `copiedContractStillMatchesProduction`, fails when the copy drifts from
  `gametools/build.gradle.kts`. That is the cost of testing a copy: a test that tests its own duplicate and a second test to police the
  duplicate. Mutating a copy of the real build removes the duplicate.
- Wiring (`build-logic/build.gradle.kts`): a `Test` task `integrationTest` filters `gametools.testing.integration.*` with
  `isFailOnNoMatchingTests = false`; `test` excludes that package; the comment reads "The TestKit classes spawn nested Gradle builds -
  too heavy for `./gradlew build`". Neither task declares the repository's files as inputs, so with unchanged test classes and classpath
  Gradle can report the task `UP-TO-DATE` after a change to the build it drives (inference from Gradle's input-based up-to-date model; not
  run). The design below sets `outputs.upToDateWhen { false }`.

### F11. CI data points and runners (practice; two finished runs, read 2026-10-20)

| Run | What | Wall time |
| --- | --- | --- |
| MyGameTools, run 38693939100 (2026-10-15), job `build-logic (TestKit)` on `ubuntu-latest`, JDK 23 | `./gradlew :build-logic:integrationTest`: compile `build-logic`, one nested build of two modules including two Dokka renders and the umbrella guards, two throwaway builds | job 3 m 22 s, the Gradle step 3 m 09 s |
| GeneralTools, run 39507508729 (2026-10-20), job `build` on `ubuntu-latest`, JDK 17, Gradle 9.0.0 | `./gradlew build` of one KMP module (`jvm`, `js`, `wasmJs`, 185 tests) with `cache-provider: basic` | job about 1 m 55 s, `BUILD SUCCESSFUL in 1m 51s` |

- Public repositories get the standard hosted runners free and unlimited with the same size on Linux and Windows (4 CPU, 16 GB RAM, 14 GB
  SSD; [GitHub docs](https://docs.github.com/en/actions/reference/runners/github-hosted-runners), page read 2026-10-20), so for this
  repository the cost of a slow job is wall-clock time and the 14 GB disk, not billing.
- No Windows-versus-Linux Gradle timing was found for current images. The only reports found are anecdotal: runner-images
  [#7320](https://github.com/actions/runner-images/issues/7320) (2023, Go test and `actions/checkout`, "15 times as long") and
  [#5166](https://github.com/actions/runner-images/issues/5166) (2022, "Windows Server 2022 builds are taking 4x longer than on Windows
  Server 2019"). Opinion, old, and not Gradle: they support only "Windows file-system-heavy work is slower", which is why the estimates in
  section 6.6 carry a Windows factor of up to 2.

## 4. How comparable projects do it

| Project @ ref | Tests its build logic? | How |
| --- | --- | --- |
| MyGameTools `dc5214f` (this organisation) | Yes, partly | `build-logic/src/test/kotlin/gametools/testing/integration/*`, JUnit 5 + TestKit; one test drives the real repo (`gametools.repoRoot`), two use `@TempDir` throwaway builds with copied guard text; tasks `integrationTest` (explicit) and `test` (excludes it); CI job `build-logic (TestKit)` runs `./gradlew :build-logic:integrationTest` |
| [vanniktech gradle-maven-publish-plugin 0.37.0](https://github.com/vanniktech/gradle-maven-publish-plugin/tree/0.37.0) | Yes, extensively | `plugin/src/integrationTest/kotlin/com/vanniktech/maven/publish/*PluginTest.kt` (`BasePluginTest`, `KotlinPluginTest`, `DokkaPluginTest`, ...); `ProjectSpec.run` writes a fixture project whose `pluginManagement` lists `mavenLocal()` first and applies the plugin **by id and version** (the `integrationTest` task `dependsOn(tasks.publishToMavenLocal, ...)`), runs `GradleRunner.create().withGradleVersion(options.gradleVersion.value)` with `--configuration-cache` and a per-test `withTestKitDir`; `maxRetries = 2` on CI; `tasks.check { dependsOn(integrationTest) }`; a `GradleVersion` matrix of minimum and latest |
| [Ktor 3.6.0](https://github.com/ktorio/ktor/tree/3.6.0/build-logic) | No | `build-logic/src` holds `main` only |
| [Okio parent-3.18.2](https://github.com/square/okio/tree/parent-3.18.2/build-support) | No | `build-support/src` holds `main` only |
| [Coil 3.6.3](https://github.com/coil-kt/coil/tree/3.6.3/buildSrc) | No | `buildSrc/src` holds `main` only |
| [kotlinx-io 0.9.1](https://github.com/Kotlin/kotlinx-io/tree/0.9.1/build-logic) | No | `build-logic/src` holds `main` only |
| [workflow-kotlin 40d22c9eb443](https://github.com/square/workflow-kotlin/tree/40d22c9eb443adb0b00bbf653e6fab7898ccf052/build-logic) | No | `build-logic/src` holds `main` only |
| Gradle's own sample `useTestKit-do` (v9.8.2) | Yes | `build-logic/src/functionalTest/java/org/example/MyPluginFunctionalTest.java`, a `JvmTestSuite`, `gradlePlugin { testSourceSets(...) }`, `tasks.check { dependsOn(functionalTest) }` |

Pattern: the libraries with the most elaborate conventions (Ktor, Okio, Coil, kotlinx-io, workflow-kotlin) ship none; they rely on the
build failing when a convention breaks. Where tests exist they apply the plugin through a real resolution path (`mavenLocal`, a real
repository, or an included build), not through an injected classpath, and they treat the nested runs as the expensive tier (separate task,
retries, a Gradle-version matrix). The one precedent in this organisation for a guard-of-the-guard is MyGameTools'
`UmbrellaVerifyTaskContractTest`, with the duplication cost in F10.

## 5. Pitfalls and regrets

- **`withPluginClasspath()` hides the hoist** (F2). Any test of the root block, the Dokka guard or the plugin-version rule built on it
  proves nothing.
- **Testing a copy of the guard instead of the guard** (F10): a drift test is required to keep it honest and it still cannot catch a
  production regression. Mutate a copy of the real build.
- **An `UP-TO-DATE` heavy task** (F10): a `Test` task whose inputs are its classes and classpath is skipped after a change to the build it
  drives. Heavy tasks must never be up to date.
- **Nested `check` or `build` on `graphics-core`** downloads Node and Yarn and, on the first run, the npm packages of the js runner (F4).
  Name the tasks. `--dry-run` prints the task graph without executing it, so "`check` runs `checkKotlinAbi` for both library modules and
  none for the BOM" is a free assertion.
- **A shared mutable workspace.** Mutation tests (delete a dump, add a declaration, drop the root hoist, add a module directory) corrupt
  later tests if one throws. Snapshot, mutate, run, restore in `finally`, and run the heavy tests of one workspace on one thread. The
  restore must include regenerated files (`updateKotlinAbi` rewrites the dumps).
- **Log-based assertions under the configuration cache.** The Dokka warning is a configuration-time message; a reused cache entry
  prints nothing (dokka finding F6). Assertions on configuration output run with `--no-configuration-cache`; the cache is tested separately.
- **`gradle.properties` is copied into the workspace.** It carries `org.gradle.configuration-cache=true`, `org.gradle.parallel=true` and
  `org.gradle.caching=true`. Nested runs add `--no-build-cache` (so outcomes are `SUCCESS`, not `FROM-CACHE`) and probes that read
  project state at execution time add `--no-configuration-cache`. Daemon heap comes from `org.gradle.jvmargs` in the same file; set it in
  the real file so TestKit's daemon matches a developer's (not measured here).
- **`-g <real home>` leaks `~/.gradle/gradle.properties`** into the nested builds (F5). Use it on CI only.
- **Orphaned Kotlin compile daemons.** Nested builds left on the default compiler strategy start Kotlin daemons that outlive the test
  JVM; vanniktech's fixtures use `kotlin.compiler.execution.strategy=in-process` (F3). The harness passes it.
- **Toolchain provisioning in an isolated home.** `jvmToolchain(21)` inside the nested build looks for JDK 21 among installed JDKs and, if
  none, provisions it through the foojay resolver into the TestKit home. On CI `JAVA_HOME` is JDK 21; on a developer machine with only
  JDK 23 the first nested run downloads a JDK 21 (inference; not run).
- **The Gradle 9.0.0 run needs a JVM of 24 or lower** (F3). The floor test skips with an assumption on a newer test JVM instead of
  failing.
- **Windows path length.** Kotlin, Dokka and the copied workspace create long paths; keep the workspace root short
  (`build/testkit/...` under a checkout path of about 40 characters is believed to fit, unmeasured).
- **A new network dependency in CI.** The suite needs Maven Central, the Gradle Plugin Portal and (for the Gradle 9.0.0 distribution)
  `services.gradle.org`. vanniktech retries twice on CI for that reason; this design does not retry (a retry hides a flaky wiring bug) and
  records it as a lever (section 6.6).
- **JUnit's `@TempDir` is the wrong home for the shared workspace** (it is deleted after the class, so the nested build-logic compile is
  repeated for every test class); a stable directory under `build/testkit` is reused.
- **`kotlin-dsl` language level.** `build-logic` main and test sources compile at Kotlin 2.2 whatever KGP the modules use
  (build-logic finding F1); test code uses no newer language feature.

## 6. Recommendation for this repository

Test the build logic in four mechanisms that match its four kinds of risk, and drive every nested build against a mutated **copy of the
real build** so that the root hoist, the catalog, the settings and the convention scripts under test are the files a consumer includes.
The reasoning in three sentences: pure and Gradle-object code is cheap to test in-process, so it takes the lowest levels and the
fast task; everything the conventions promise is a property of a *configured or published build*, which only a nested Gradle run can
observe, so Level 3 is TestKit; and the single most fragile promise (one plugin classloader, which the Dokka guard exists to protect) is
unobservable under `withPluginClasspath()`, so no fixture-with-injection approach is used. Rejected: fixtures with `withPluginClasspath()`
(F2); the MyGameTools "copied contract" style (F10); wiring the heavy suite into `check` (F1, F10); a separate Gradle 9.0.0 workflow job
(replaced by a test that also proves consumer resolution).

### 6.1 Levels, mechanisms and what each leaves uncovered

| Mechanism | Level | Runs | Covers | Leaves uncovered |
| --- | --- | --- | --- | --- |
| Real-build guards `checkKotlinAbi`, `verifyDokkaSourceSets`, `verifyBomCoverage`, strict explicit-API compile | 1 (local gating, inside every `check`) | every `./gradlew check` and CI `build` | the committed state is consistent *now* | that a guard bites (negative direction); published metadata; compiler options; filter semantics; consumers; Gradle 9.0.0; the cache |
| JUnit 5 on pure functions | 4a | `:build-logic:test`, seconds | `parseTestLevels`, `bomCoverageProblems`, floor parsing, filter patterns, the level-name list | anything needing a Gradle object |
| JUnit 5 + `ProjectBuilder` + `@TempDir` + MockK | 2 | `:build-logic:test`, seconds | task classes (`VerifyDokkaSourceSets`, `VerifyBomCoverage`), `configureTestLevels`, `Catalog.kt` error paths | convention *scripts* (need a catalog and settings), KGP-dependent helpers |
| Repository-file checks | 1 | `:build-logic:test`, milliseconds | committed dumps, `.gitattributes` anchor, `include(...)` vs directories, `group` | contents of the dumps (the check task owns that) |
| TestKit on a mutated copy of the real build, JVM 9.7.1 | 3 | `integrationTest` | negative controls, task graph, configured options, published metadata, the hoist negative control, filter | js test *execution*, anything needing two versions, IDE |
| TestKit composite consumer on Gradle 9.0.0, Kotlin 2.1.20 | 4b | `e2eTest` | the composite-consumer floor, substitution, stdlib 2.2.0, JDK-attribute failure, foreign `-PtestLevel` | the real client project, IDE import |
| TestKit store and reuse | 4c | `nonfunctionalTest` | configuration-cache compatibility of the real tasks and of the guards on a cache hit | Isolated Projects (blocked by the js target) |
| Recorded manual steps | 1 and 5 | by hand, once or at bumps | section 6.7 | n/a |

### 6.2 Test inventory

Source root `build-logic/src/test/kotlin/`; production package `com.spartanlabs.graphics.build`; every class in
`com.spartanlabs.testing.<level>.graphics.build`, one class per file, JUnit Jupiter `@Test` with backtick names. `[S]` = in-process, under
a few seconds; `[N]` = nested Gradle runs, with the invocation count.

**Shared support** (no test classes; outside every level filter): `com.spartanlabs.testing.support` holds `RealBuildWorkspace`,
`ConsumerFixture` and `ModuleMetadata` (section 6.3).

| # | Class (path under `com/spartanlabs/testing/`) | Level | Cost | Asserts |
| --- | --- | --- | --- | --- |
| 1 | `gating/graphics/build/RepositoryContractTest` | 1 | [S] | for each `*/build.gradle.kts` applying `spartangraphics.kmp-library`: `<dir>/api/<dir>.api` and `<dir>/api/<dir>.klib.api` exist; applying `spartangraphics.jvm-library`: `<dir>/api/<dir>.api` exists; the BOM has no `api/`; every `include(...)` in `settings.gradle.kts` names an existing directory and every directory with a `build.gradle.kts` is included (folder = project name = artifactId); `gradle.properties` holds `group=io.github.spartanlaboratories` and a `version=` line; `.gitattributes` holds the line `**/api/*.api text eol=lf` (the leading `**/` is the anchor the ABI finding requires) |
| 2 | `deterministic/graphics/build/TestLevelsTest` | 4a | [S] | `parseTestLevels(null/""/" ", known)` is `success([])`; `"component"` is `[component]`; `"gating, component"` keeps order and trims; duplicates drop; a blank entry between commas is ignored; `"Component"` and `"bogus"` are `failure` whose message names every unknown value and lists `known`; `SpartanBuild.TEST_LEVELS` equals `gating, component, integration, deterministic, e2e, nonfunctional, uat` in that order; `levelFilterPattern("component")` is `com.spartanlabs.testing.component.*` |
| 3 | `deterministic/graphics/build/BomCoverageProblemsTest` | 4a | [S] | consistent input gives `[]`; a `spartangraphics.kmp-library` directory missing from the KMP list, a `jvm-library` one missing from the JVM list, a listed module with no directory, a module listed under the wrong kind, and a script applying an unknown `spartangraphics.*-library` id each give exactly one problem whose text names the module and the repair; a script applying neither convention (the BOM itself) is ignored; ids match as whole tokens (`spartangraphics.kmp-library-x` is not `kmp-library`) |
| 4 | `deterministic/graphics/build/KotlinFloorTest` | 4a | [S] | the floor parser (`Result<KotlinVersion>`): `"2.2.0"` and `"2.2.21"` give `KOTLIN_2_2`; `"2.2"` gives `KOTLIN_2_2`; `"2"`, `""`, `"2.x.0"` and `"9.9.0"` are `failure` with a message naming the catalog key `kotlin-floor` |
| 5 | `component/graphics/build/ConfigureTestLevelsTest` | 2 | [S] | with a `ProjectBuilder` project, the `java` plugin and a registered `Test` task, `configureTestLevels(requested = provider { "component,gating" })` gives `filter.includePatterns` of exactly the two patterns and `isFailOnNoMatchingTests == false`; a task registered *after* the call is filtered too; a `null` provider leaves `includePatterns` empty and `isFailOnNoMatchingTests` at its default `true`; an unknown level throws `GradleException` naming the property |
| 6 | `component/graphics/build/VerifyDokkaSourceSetsTest` | 2 | [S] | `VerifyDokkaSourceSets.verify()` with empty `documentedSourceSets` throws, and the message contains the module path and `KotlinBasePlugin`; with `["commonMain", "jvmMain"]` it returns |
| 7 | `component/graphics/build/VerifyBomCoverageTest` | 2 | [S] | `VerifyBomCoverage.verify()` over module scripts written into a `@TempDir`: consistent passes; each problem class of test 3 throws `GradleException` containing all problems, one per line |
| 8 | `component/graphics/build/CatalogTest` | 2 | [S] | with a MockK `VersionCatalog`, `version("kotlin-floor")` returns the required version; a missing alias throws `GradleException` whose message names `gradle/libs.versions.toml`, the table (`[versions]` or `[libraries]`) and the alias |
| 9 | `integration/graphics/build/ExplicitApiTest` | 3 | [N] 3 | `graphics-desktop/src/main/kotlin/Probe.kt` holding `class Probe` makes `:graphics-desktop:compileKotlin` fail with an explicit-API visibility error; the same declaration in `src/test/kotlin` compiles (`:graphics-desktop:compileTestKotlin` succeeds); `graphics-core/src/commonMain/kotlin/Probe.kt` makes `:graphics-core:compileKotlinJvm` fail |
| 10 | `integration/graphics/build/AbiGuardTest` | 3 | [N] 5 | `check --dry-run` lists `:graphics-core:checkKotlinAbi` and `:graphics-desktop:checkKotlinAbi` and no `:graphics-bom:` ABI task; with both `graphics-desktop.api` and `graphics-core.klib.api` deleted, `checkKotlinAbi --continue` fails both tasks with "Expected file with ABI declarations"; `public class Probe` added to `graphics-desktop` makes `checkKotlinAbi` fail with `<<<ABI has changed>>>`, and `updateKotlinAbi checkKotlinAbi` **in one invocation** then passes and the dump contains `Probe` (proves the `mustRunAfter`); a dump rewritten with CRLF passes, one with an extra trailing newline fails; no executed task of any of these runs has a path ending in `kotlinNodeJsSetup` (cost guard for F4) |
| 11 | `integration/graphics/build/TestLevelFilterTest` | 3 | [N] 5 | `-PtestLevel=bogus` makes `:graphics-desktop:test` fail with a message listing the known levels; with probe tests `…/testing/component/probe/ProbeComponentTest.kt` and `…/testing/integration/probe/ProbeIntegrationTest.kt` in `graphics-desktop` and in `graphics-core` `commonTest`, unset runs both, `component` runs only the first (read from `build/test-results/test/TEST-*.xml` and `jvmTest`), `uat` succeeds with no test executed; the filter probe (section 6.3) lists `includePatterns` for `:graphics-desktop:test`, `:graphics-core:jvmTest` **and** `:graphics-core:jsNodeTest` with `isFailOnNoMatchingTests == false` |
| 12 | `integration/graphics/build/CompilerContractTest` | 3 | [N] 1 | the compiler-options probe reports, for `:graphics-desktop:compileKotlin`, `:graphics-core:compileKotlinJvm`, `:graphics-core:compileKotlinJs` and the common metadata compilation: `languageVersion` and `apiVersion` 2.2; for the two JVM tasks `jvmTarget` 21 and `jvmDefault` ENABLE; `kotlin.explicitApi` is Strict; the toolchain language version is 21 |
| 13 | `integration/graphics/build/PublishedMetadataTest` | 3 | [N] 2 | runs `generateMetadataFileFor*Publication` and `generatePomFileFor*Publication`, then parses the module metadata: `graphics-core` jvm variants (`jvmApiElements-published`, `jvmRuntimeElements-published`) have `org.gradle.jvm.version` 21 and `kotlin-stdlib` `requires 2.2.0`; the root metadata variants require `kotlin-stdlib` 2.2.0; the js variants require the catalog's `kotlin` version; `graphics-desktop` `apiElements` and `runtimeElements` have 21, `kotlin-stdlib` 2.2.0 and `graphics-core`; every library variant carries `graphics-bom` as `org.gradle.category: platform` with `endorseStrictVersions`; the BOM `.module` has exactly the constraints `graphics-core`, `graphics-core-jvm`, `graphics-core-js`, `graphics-desktop`, each `requires` the project version and none `strictly`; the `graphics-desktop` POM names `graphics-core-jvm` and imports the BOM (`<scope>import</scope>`); `:graphics-core:dependencies --configuration jsCompileClasspath` lists `graphics-bom` |
| 14 | `integration/graphics/build/DokkaSourceSetGuardTest` | 3 | [N] 4 | with `--no-configuration-cache --info`: `verifyDokkaSourceSets` passes in both modules, the output has no `Dokka could not load KotlinBasePlugin`, and it logs `commonMain`, `jsMain`, `jvmMain` and `main`; `:graphics-core:jvmDokkaJavadocJar --dry-run` lists `verifyDokkaSourceSets` before `dokkaGeneratePublicationHtml`; `:dokkaGeneratePublicationHtml --dry-run` lists `dokkaGenerateModuleHtml` for every directory that applies a library convention and nothing for `graphics-bom`; **negative control:** with the two Kotlin `apply false` aliases removed from the root `plugins {}` block, the warning appears and `verifyDokkaSourceSets` fails with its message |
| 15 | `integration/graphics/build/BomCoverageGuardTest` | 3 | [N] 2 | a directory `graphics-probe/build.gradle.kts` applying `spartangraphics.jvm-library` makes `:graphics-bom:verifyBomCoverage` fail naming `graphics-probe`; the target-completeness check: the `generatePomFileFor<Target>Publication` tasks in `:graphics-core:tasks --all`, other than the root publication, map to the `graphics-core-<target>` BOM entries one-to-one |
| 16 | `e2e/graphics/build/CompositeConsumerFloorTest` | 4b | [N] 4, own workspace | on Gradle **9.0.0**, a generated consumer (`kotlin("jvm") version "2.1.20"`, `mavenCentral()`, the foojay settings plugin as in the client, `includeBuild(<workspace>)`, `api("io.github.spartanlaboratories:graphics-desktop:<version>")`): `compileKotlin` succeeds, which compiles `build-logic` with embedded Kotlin 2.2.0 and applies the plugins; `dependencies --configuration compileClasspath` shows `graphics-desktop` and `graphics-core` as `project :` substitutions and `graphics-bom`; `dependencyInsight --dependency kotlin-stdlib` selects 2.2.0; with `java.targetCompatibility` 17 the build fails at resolution with a message naming Java 21; with `-PtestLevel=bogus` it fails with this build's message, with `-PtestLevel=component` it succeeds; skipped by assumption on a test JVM newer than 24 |
| 17 | `nonfunctional/graphics/build/ConfigurationCacheTest` | 4c | [N] 4 | `checkKotlinAbi verifyBomCoverage verifyDokkaSourceSets :graphics-core:generateMetadataFileForJvmPublication --configuration-cache` run twice: first output has "Configuration cache entry stored", the second has "Reusing configuration cache.", neither reports a problem, and the guard tasks execute on the reused entry; with the hoist removed (workspace mutation of test 14) the failing guard fails again on the second, reused run |

Level 5 has no automated class. Counts: 17 classes; 8 of them in-process, 9 nested.

### 6.3 Harness (typed helpers, shape only)

`com.spartanlabs.testing.support.RealBuildWorkspace` (about 80 lines):

```kotlin
internal class RealBuildWorkspace private constructor(val dir: File, private val testKitDir: File, private val gradleVersion: String?) {
    /** Runs Gradle in the copy. Common arguments: --no-build-cache --stacktrace -Pkotlin.compiler.execution.strategy=in-process
     *  -Dorg.gradle.vfs.watch=false; `-g <real home>` only when the CI property is set. */
    fun build(vararg args: String): BuildResult = runner(*args).build()
    fun buildAndFail(vararg args: String): BuildResult = runner(*args).buildAndFail()

    /** Snapshots every tracked text file, runs [block], restores the tree in `finally` (also undoes updateKotlinAbi). */
    fun <T> mutating(block: RealBuildWorkspace.() -> T): T

    fun write(relativePath: String, text: String)
    fun delete(relativePath: String)
    fun edit(relativePath: String, transform: (String) -> String)

    companion object {
        val shared: RealBuildWorkspace by lazy { copyOfRealBuild(gradleVersion = null) }      // JVM 9.7.1: tests 9 to 15 and 17
        fun fresh(gradleVersion: String): RealBuildWorkspace = copyOfRealBuild(gradleVersion) // test 16: its own copy, its own build-logic outputs
    }
}
```

- `copyOfRealBuild` copies `settings.gradle.kts`, `build.gradle.kts`, `gradle.properties`, `gradle/libs.versions.toml`, `build-logic/*.gradle.kts`,
  `build-logic/src/main/**`, `graphics-*/**` (excluding `build`, `.gradle`, `.kotlin`, `node_modules`), `.gitattributes`, and
  `kotlin-js-store/` from `System.getProperty("spartangraphics.repoRoot")` into `<spartangraphics.testkit.root>/workspace[-<version>]`. It does
  not copy `build-logic/src/test`, `docs`, `.git` or the wrapper (TestKit supplies the distribution).
- The shared workspace and the 9.0.0 workspace are separate because the same `build-logic/build` directory cannot be compiled by two Gradle
  versions without invalidating the other's outputs.
- **Probes** are Kotlin DSL snippets kept as test resources (`src/test/resources/probes/*.gradle.kts`) and appended to a module script inside
  `mutating { }`: the compiler-options probe (test 12) and the test-filter probe (test 11) each register a `printContract` task that prints
  `CONTRACT <key>=<value>` lines read from the Kotlin extension and `tasks.withType<AbstractTestTask>()`. They read project state at
  execution time and so run with `--no-configuration-cache`. Typed accessors (`kotlin`) are available in a module script that applies only
  a convention plugin because Kotlin DSL generates accessors for every extension registered after the `plugins {}` block (Gradle's
  documented behaviour; not run here).
- `ModuleMetadata` wraps parsing of `module.json` (`variants[].name`, `attributes`, `dependencies[]` with `version.requires`,
  `dependencyConstraints[]`, `endorseStrictVersions`) behind `variant(name)` accessors. `groovy.json.JsonSlurper` (on the `gradleApi()`
  classpath that `kotlin-dsl` provides) avoids a new dependency; `kotlinx-serialization-json` `Json.parseToJsonElement` is the idiomatic
  alternative and needs one catalog entry (Known gaps).
- `ConsumerFixture` writes the consumer of test 16 into a directory next to the 9.0.0 workspace, with the workspace path in
  `invariantSeparatorsPath` form and the project version read from the workspace's `gradle.properties`.

### 6.4 `build-logic/build.gradle.kts` (test wiring)

Extends build-logic finding 6.7; the `plugin(...)` helper and the four `implementation(plugin(...))` lines are unchanged.

```kotlin
import java.time.Duration

plugins { `kotlin-dsl` }                 // applies java-gradle-plugin: gradleTestKit() and the plugin-under-test metadata come with it

dependencies {
    // implementation(plugin(libs.plugins.kotlin.multiplatform)) ... as in build-logic-and-version-catalog.md 6.7
    testImplementation(platform(libs.junit.bom))
    testImplementation(libs.junit.jupiter)
    testImplementation(libs.mockk)
    testRuntimeOnly(libs.junit.platform.launcher)      // required explicitly on Gradle 9
    // no logback, no kotlin-logging: kotlin-dsl's gradleApi() brings its own SLF4J provider (F9)
}

val repoRoot = layout.projectDirectory.dir("..").asFile.absolutePath
val testKitRoot = layout.buildDirectory.dir("testkit")
val heavyLevels = mapOf("integrationTest" to "integration", "e2eTest" to "e2e", "nonfunctionalTest" to "nonfunctional")

tasks.withType<Test>().configureEach {
    useJUnitPlatform()
    systemProperty("spartangraphics.repoRoot", repoRoot)
}

// Levels 1, 2 and 4a: in-process, seconds, part of `check`.
tasks.test {
    filter {
        heavyLevels.values.forEach { excludeTestsMatching("com.spartanlabs.testing.$it.*") }
        isFailOnNoMatchingTests = false
    }
}

// Levels 3, 4b and 4c: nested Gradle builds. Named tasks; not part of `check`.
heavyLevels.forEach { (taskName, level) ->
    tasks.register<Test>(taskName) {
        group = "verification"
        description = "Level $level: TestKit runs against a copy of the real build (nested Gradle builds)."
        testClassesDirs = sourceSets.test.get().output.classesDirs
        classpath = sourceSets.test.get().runtimeClasspath
        filter {
            includeTestsMatching("com.spartanlabs.testing.$level.*")
            isFailOnNoMatchingTests = false
        }
        // The build under test is a set of files outside this task's declared inputs.
        outputs.upToDateWhen { false }
        maxParallelForks = 1                                   // the tiers share a mutable workspace
        maxHeapSize = "1g"
        timeout.set(Duration.ofMinutes(30))
        systemProperty("org.gradle.testkit.dir", testKitRoot.get().dir("gradle-home").asFile.absolutePath)
        systemProperty("spartangraphics.testkit.root", testKitRoot.get().asFile.absolutePath)
        // On CI, nested builds use the runner's Gradle user home so setup-gradle's cache applies (F5).
        systemProperty("spartangraphics.testkit.sharedGradleHome", providers.environmentVariable("CI").isPresent.toString())
    }
}
tasks.named("e2eTest") { mustRunAfter("integrationTest") }
tasks.named("nonfunctionalTest") { mustRunAfter("e2eTest") }
```

- `-PtestLevel` is not used for `build-logic`'s own tests. The library modules need one property that filters every target's test task
  inside `check`; `build-logic` needs a split between tiers that are cheap and tiers that spawn builds, and its build script cannot import
  its own compiled `parseTestLevels`. Level selection there is by task name, or by `--tests "com.spartanlabs.testing.<level>.*"`.
- Test dependencies come from the catalog aliases, so one `junit` version feeds the modules and `build-logic`. The catalog value is the
  5.x line this build uses for Jupiter; JUnit Jupiter 6.x would need only the catalog value to change.
- Commands. Pre-commit loop: `./gradlew :build-logic:test` (or `-p build-logic test`). Everything cheap: `./gradlew :build-logic:check`. The
  nested suites: `./gradlew :build-logic:integrationTest`, `:build-logic:e2eTest`, `:build-logic:nonfunctionalTest`. One class:
  `./gradlew :build-logic:integrationTest --tests "*AbiGuardTest"`.

### 6.5 What this design adds to the production code

Small seams, each with a use beyond the tests; all in `com.spartanlabs.graphics.build`:

- `Project.configureTestLevels(requested: Provider<String> = providers.gradleProperty("testLevel"))`: the provider is a parameter, so
  Level 2 can feed a value (a `ProjectBuilder` project cannot receive a `-P` property).
- `fun levelFilterPattern(level: String): String` (pure), used by `configureTestLevels` and tested at Level 4a.
- A pure floor parser returning `Result<KotlinVersion>` (the repository's rule for expected failure); `Catalog.kotlinFloor()` unwraps it at
  the single place a failure becomes a `GradleException`.
- `bomCoverageProblems(...)` and `VerifyDokkaSourceSets` exactly as already designed; `VerifyDokkaSourceSets` stays a public abstract class
  (a task type), `bomCoverageProblems` stays `internal` (the test source set sees `internal` members of `main`).

### 6.6 CI

**Windows full-build job** gets one added invocation in its Gradle step (no extra job):

```
./gradlew build :build-logic:check :dokkaGeneratePublicationHtml javadocJars --stacktrace
```

`:build-logic:check` runs test classes 1 to 8 (levels 1, 2, 4a). Estimated added time: 20 to 60 seconds (compile the test source set, run
the tests, spin up MockK).

**New job `build-logic (TestKit)`** on `windows-latest`, JDK 21 (Temurin), in parallel with the build job (it does not need it):

```yaml
  build-logic-testkit:
    name: build-logic (TestKit)
    runs-on: windows-latest
    timeout-minutes: 40
    steps:
      - uses: actions/checkout@v7            # versions as pinned by the CI finding (GeneralTools currently uses these)
      - uses: actions/setup-java@v6
        with: { distribution: temurin, java-version: "21" }
      - uses: gradle/actions/setup-gradle@v6
      - run: ./gradlew :build-logic:integrationTest :build-logic:e2eTest :build-logic:nonfunctionalTest --continue --stacktrace
      - if: failure()
        uses: actions/upload-artifact@v4     # reports of the three tasks
        with: { name: build-logic-testkit-reports, path: build-logic/build/reports/tests/ }
```

- Why Windows. The Windows job is the one that runs every non-GL test; the ABI dumps and their `core.autocrlf` behaviour, path length
  and file locking are Windows risks that a Linux-only run would not see. Why a separate job: the suite is minutes, the build job is
  minutes, and they share nothing but the cache; parallel jobs make the pull request wait the longer of the two.
- Required-check naming is the CI finding's to own; this job's name is `build-logic (TestKit)`, the same string MyGameTools uses.
- Cost estimate for this job, cold cache, from F11 (MyGameTools: 3 m 09 s for compile, one real two-module build with two Dokka renders
  and two throwaway builds; GeneralTools: 45 s of daemon start and configuration for one KMP module): compile `build-logic` and its tests
  1 to 2 minutes; the shared workspace's first nested run (build-logic compile again in the copy, KGP/Dokka/vanniktech resolution) 2 to 3
  minutes; the roughly 26 further nested invocations of tests 9 to 15 and 17 at 8 to 20 seconds each, 3 to 6 minutes; the Gradle 9.0.0
  consumer (distribution download, a full second `build-logic` compile on the other embedded Kotlin, KGP 2.1.20 and 2.4.21 resolution,
  four invocations) 3 to 5 minutes. About **8 to 14 minutes on Linux-class I/O, up to about twice that on Windows**, and well under
  half of that with a warm `setup-gradle` cache on a re-run of the same branch. These are estimates; the first CI run is the measurement
  and its timings go into the plan's as-built record.
- Trimming levers, in order of cost saved: batch assertions into fewer invocations with `--continue` (already used in tests 10 and 11);
  run test 16 only when the pull request touches `build-logic/**`, `gradle/libs.versions.toml` or `gradle/wrapper/**` (needs a job-internal
  path check so the required check still reports; see Consideration C1); retries for network flakes (vanniktech does two on CI; not in
  the baseline, because a retry also hides a flaky wiring bug).

### 6.7 Manual steps (named, recorded in a UAT record beside the plan)

Level 5 and one-off Level 1 experiments. M1 and M2 become automated, or are covered by tests above, once they have run; they are named
so the first run is deliberate.

| # | Step | Expectation | Why not automated |
| --- | --- | --- | --- |
| M1 | Before pinning test 14's negative control, run it once as the Dokka finding 6.6 step 2 describes (remove the two Kotlin `apply false` aliases) | the `Dokka could not load KotlinBasePlugin` warning appears and `verifyDokkaSourceSets` fails; if it does not, the hoist is not needed and the design (root block, guard, test 14) is revisited | it is the experiment that decides whether the test is valid |
| M2 | Run `gradle -p build-logic assemble` once with a 9.0.0 distribution on the maintainer's JDK before test 16 is trusted | build-logic compiles and its accessors generate | test 16 subsumes it afterwards |
| M3 | Copy `D:\Documents\Programming\GameGraphics`, add `includeBuild("../SpartanGraphics")` and a `graphics-desktop` dependency, build on its 9.0.0 wrapper (KGP 2.1.20, JDK 23, toolchain 23) | builds; `dependencyInsight` shows `project :graphics-bom` as the platform | the generated consumer of test 16 stands in for it; the real client has more plugins (serialization, `application`) |
| M4 | The same copy on the repository's wrapper line (9.7.x) | builds | one-off before the client's wrapper is bumped |
| M5 | `./gradlew build --isolated-projects` (diagnostic) | the only problems are the Kotlin Gradle plugin's js target (KT-75899) and nothing from `build-logic` or the scripts | the js target makes the run fail; it can only be read |
| M6 | `./gradlew build --warning-mode all` at each wrapper or plugin bump | no deprecation attributed to this repository's scripts | third-party deprecations (Dokka on Gradle 9.8) make an assertion noisy |
| M7 | When the first `commonTest` class lands (G2): `./gradlew :graphics-core:jsNodeTest -PtestLevel=component` then `-PtestLevel=integration` | component tests run on Node; the second run executes none and is green | needs Node and npm; the js execution of a package pattern is source-read only (reused research) |
| M8 | Fresh clone on the maintainer's Windows machine with `core.autocrlf=true`: `./gradlew updateKotlinAbi` | `git status` clean | the CI Windows runner covers the comparison (test 10) but not the maintainer's git configuration |
| M9 | Open `build/dokka/html/index.html` of the aggregate once | the "All modules" page and module names are as intended (empty table in G1) | rendering is judged by a person |
| M10 | Import the repository and the throwaway client copy into IntelliJ IDEA | `libs` accessors resolve in `build-logic`, no red imports in `build-logic/src/main` and `src/test`, the composite imports (KT-30285 and KTIJ-18903 are reports of IDE problems with composites) | IDE import is UAT |

### 6.8 Relation to the other findings

- **Toolchain finding, 6.6 and Consideration C3** (a permanent CI job running `gradle -p build-logic assemble` on a 9.0.0 distribution):
  replaced by test 16 and manual step M2. Test 16 proves strictly more (consumer resolution, the other Kotlin Gradle plugin in the same
  composite) and costs 3 to 5 minutes instead of about one, and it lives in the suite rather than in the workflow. Keep the one-minute job
  as well only if the longer suite is later moved off the pull-request path.
- **Dokka finding, 6.6 steps 2, 4 and 5**: automated as test 14 (negative control, dry-run wiring) and test 17 (cache hit still fails the
  guard). Its pitfall "`dokka(project(...))` is a hand-written list" is covered by the `--dry-run` assertion in test 14.
- **ABI finding, 6.5 steps 2, 3, 4, 5 and 6**: automated as tests 10 and 17; its step 7 (does a first `updateKotlinAbi` download Node)
  is answered by F4 from source (it should not) and checked by test 10's `kotlinNodeJsSetup` assertion. F4 also adds a fact that finding
  does not state: `check` on `graphics-core` does run Node and Yarn setup, through the js test task's dependencies, even with no tests.
- **BOM finding, 6.6 steps 1, 2, 3, 4 and 6**: automated as tests 13, 15, 16 and 17; its step 5 (alignment after a second version exists) is
  Gradle's documented, Gradle-tested behaviour and stays a G7-time check.
- **Build-logic finding, 6.10**: step 1 is test 14 plus M1; step 2 is test 17 plus M5; step 3 is test 16 plus M3 and M4.
- **Toolchain finding, F10 and Consideration C2** (JUnit 6.1.3): this design reads `junit` from the catalog and so follows whatever C2
  settles; nothing here needs 6.x.

## 7. Considerations

**C1. Does the Gradle 9.0.0 consumer test (test 16, 3 to 5 minutes) gate every pull request?**
- *(a) Always* — simplest; the floor is checked on every change, including Dependabot bumps of Kotlin, Dokka, vanniktech and the wrapper.
  Costs the minutes on every pull request.
- *(b) Only when build inputs change* (`build-logic/**`, `gradle/libs.versions.toml`, `gradle/wrapper/**`, root build files), using a
  job-internal path check so the job still reports success when skipped. Saves the minutes on documentation and source-only changes;
  adds a path list to maintain, and in G1 every pull request touches those paths anyway.
- *(c) Scheduled* weekly plus manual dispatch, not required. Cheapest; a floor regression is found a week late.
Recommendation: **(a)** for G1 and until the first CI run is measured; move to (b) if the suite exceeds roughly 15 minutes on
`windows-latest`.

**C2. Should the suite share the runner's Gradle user home (`-g`) on CI?**
- *(a) Yes, on CI only* (the design above) — the `setup-gradle` cache then covers the nested builds; loses TestKit's isolation on the
  runner, which has no user-level `gradle.properties` to leak.
- *(b) No* — hermetic, but every run re-downloads plugin dependencies, the JDK extras and Node; a minute or two more in each cache-cold
  run and more network exposure.
Recommendation: **(a)**, because the isolation it gives up protects against a developer's machine, not a runner. Local runs keep the
isolated home.

## 8. Known gaps

- **No execution.** No Gradle task, TestKit run or CI run of this design happened. The inventory, the helper and the wiring are unproven in
  detail; the timings are estimates built from two finished runs of other builds (F11) and carry a stated Windows factor of up to 2 that is
  supported only by anecdotal 2022 and 2023 reports. The first CI run is the measurement.
- **The hoist negative control is inferred.** That removing the root `apply false` Kotlin aliases reproduces MyGameTools' empty Dokka
  output in this exact arrangement (`pluginManagement` build-logic, catalog aliases, vanniktech hoisted) is a reading of Gradle's
  classloader behaviour, not an observation here; the MyGameTools measurement used a different include form (build-logic finding F3).
  Step M1 settles it before test 14 is pinned.
- **Why `withPluginClasspath()` hides the hoist is source-read.** F2 rests on `DefaultInjectedClasspathPluginResolver` and one Gradle test;
  no run confirmed that a fixture with the root block removed still passes the Dokka guard.
- **`ProjectBuilder` projects have no `libs` catalog** is inferred (Javadoc says "dummy instances"; it does not mention catalogs). The design
  does not depend on it: convention scripts are tested through TestKit either way.
- **Appended probes and accessors.** That a module script applying only a convention plugin can use the typed `kotlin` accessor in an
  appended probe is Kotlin DSL's documented behaviour, but it was not run; the fallback is `extensions.getByName("kotlin")` with
  reflection.
- **`groovy.json.JsonSlurper` on the `gradleApi()` test classpath** was not confirmed; if absent, add `kotlinx-serialization-json` to the
  catalog and the test dependencies.
- **`jsNodeTest` filter execution** (a trailing-`*` package pattern on the Mocha runner) is not run by any G1 test; test 11 checks that the
  filter reaches the task, and M7 runs it once real tests exist.
- **Node.js only through test tasks** is inferred from dependency declarations (F4); test 10's `kotlinNodeJsSetup` assertion would show
  otherwise on the first run.
- **Sharing the real Gradle user home with TestKit (`-g`)** is documented for the argument; its interaction with `setup-gradle`'s end-of-job
  cache cleanup and with two Gradle versions in one home was not examined. Nested JDK 21 provisioning into an isolated home on a machine
  without JDK 21 is likewise unexamined.
- **Daemon heap.** The workspace inherits the real `gradle.properties`, which currently sets no `org.gradle.jvmargs`; whether TestKit's
  default daemon heap suffices for the KMP configuration plus the Kotlin compile was not measured.
- **Windows hosted-runner timing for Gradle** has no current data; GeneralTools' and MyGameTools' public runs are both Linux.
- **Not examined:** Kotlin test frameworks other than JUnit for `build-logic`; Spock; Develocity test distribution; whether a plugin build
  included from a consumer's `pluginManagement` behaves differently when the consumer itself is a nested included build (build-logic
  finding gap; test 16 covers the include-as-library shape only).

## 9. Load-bearing claims

Quotes from AsciiDoc, Java, Kotlin and Groovy sources omit markup characters only. "Read" is 2026-10-20.

1. **Gradle recommends functional tests for build logic and keeps TestKit tests separate; included `build-logic` tests are not run from
   the root.** "A mature build should include functional tests for its custom types to ensure they behave as expected." and "Because
   TestKit tests tend to be slower and more complex than unit tests, they are typically kept separate." and "By default, tests within the
   included `build-logic` build are not executed when you run tests in the root project." [best_practices_testing.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/best-practices/best_practices_testing.adoc), Gradle v9.8.2.
2. **A task of a `pluginManagement`-included build can be run by its qualified path.** `PluginBuildsIntegrationTest`: `def "can execute a task
   from included plugin build"()` with `pluginManagement { includeBuild("$pluginBuild") }` and `succeeds(":$pluginBuild:jar")`.
   [source](https://github.com/gradle/gradle/blob/v9.8.2/subprojects/composite-builds/src/integTest/groovy/org/gradle/integtests/composite/plugins/PluginBuildsIntegrationTest.groovy), Gradle v9.8.2.
3. **Unit tests of plugins use `ProjectBuilder`; it creates dummy projects.** "Unit tests | Any JVM-based test framework + `ProjectBuilder`"
   ([testing_gradle_plugins.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/plugin-development/testing_gradle_plugins.adoc), v9.8.2);
   "Creates dummy instances of {@link org.gradle.api.Project} which you can use in testing custom task and plugin implementations."
   ([ProjectBuilder.java](https://github.com/gradle/gradle/blob/v9.8.2/platforms/extensibility/unit-test-fixtures/src/main/java/org/gradle/testfixtures/ProjectBuilder.java), v9.8.2).
4. **`kotlin-dsl` applies `java-gradle-plugin`, which adds TestKit and the plugin-under-test metadata.** "Applies the `java-gradle-plugin` plugin"
   ([KotlinDslPlugin.kt](https://github.com/gradle/gradle/blob/v9.8.2/platforms/core-configuration/kotlin-dsl-plugins/src/main/kotlin/org/gradle/kotlin/dsl/plugins/dsl/KotlinDslPlugin.kt), v9.8.2);
   "It automatically adds the `gradleTestKit()` dependency to the `testImplementation` configuration and generates a plugin classpath
   manifest file consumed by a `GradleRunner` instance if found."
   ([java_gradle_plugin.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/core-plugins/java_gradle_plugin.adoc), v9.8.2).
5. **Injected plugins come from one classloader scope.** `DefaultInjectedClasspathPluginResolver.newPluginRegistryOf`:
   `parentScope.createChild("injected-plugin", null)` `.local(classPath)` `.lock()`; `InjectedClasspathPluginResolution.accept`:
   `visitor.visitClassLoader(plugin.asClass().getClassLoader());`
   ([source](https://github.com/gradle/gradle/blob/v9.8.2/platforms/extensibility/plugin-use/src/main/java/org/gradle/plugin/use/resolve/service/internal/DefaultInjectedClasspathPluginResolver.java), v9.8.2);
   test `def "injected classes are loaded only once"()` asserting `project("child1").tasks.helloWorld1.getClass() == project("child2").tasks.helloWorld1.getClass()`
   ([source](https://github.com/gradle/gradle/blob/v9.8.2/platforms/extensibility/test-kit/src/integTest/groovy/org/gradle/testkit/runner/GradleRunnerPluginClasspathInjectionIntegrationTest.groovy), v9.8.2).
6. **Classpath injection applies to the `plugins {}` DSL only.** "Automatic classpath injection only works when the plugin-under-test is
   applied in the test build using the `plugins {}` DSL." [test_kit.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/plugin-development/test_kit.adoc), v9.8.2.
7. **TestKit isolates the build from the user's home and uses dedicated daemons.** "Configuration from the default Gradle User Home (for example,
   `~/.gradle/gradle.properties`) is **not** used." and "TestKit uses dedicated Gradle daemon processes for the builds it runs."
   [test_kit.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/plugin-development/test_kit.adoc), v9.8.2.
8. **`withTestKitDir` is also the build's Gradle user home unless `-g` is given; default location.** "If no explicit Gradle user home is
   specified via the build arguments (i.e. the {@code -g «dir»} option}), this directory will also be used for the Gradle user home for the
   test build." and "When executed from a Gradle Test task, the Test task's temporary directory is used". And `withGradleVersion`: "The
   download will be cached beneath the Gradle User Home directory". [GradleRunner.java](https://github.com/gradle/gradle/blob/v9.8.2/platforms/extensibility/test-kit/src/main/java/org/gradle/testkit/runner/GradleRunner.java), v9.8.2.
9. **TestKit daemons stay under the TestKit directory; distributions come from the real home.** `ToolingApiGradleExecutor.buildConnector`:
   `gradleConnector.useDistributionBaseDir(GradleUserHomeLookup.gradleUserHome());` `gradleConnector.useGradleUserHomeDir(gradleUserHome);`
   `gradleConnector.daemonBaseDir(new File(gradleUserHome, TEST_KIT_DAEMON_DIR_NAME));`
   [source](https://github.com/gradle/gradle/blob/v9.8.2/platforms/extensibility/test-kit/src/main/java/org/gradle/testkit/runner/internal/ToolingApiGradleExecutor.java), v9.8.2.
10. **The Tooling API reaches back to Gradle 4.0.** "For example, the Tooling API 9.0.0 release is compatible with Gradle versions >= 4.0."
    [tooling_api.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/integration/tooling_api.adoc), v9.8.2.
11. **Gradle 9.0.0 runs on Java 17 to 24, Java 25 from 9.1.0; TestKit cannot select the build JDK.** Table rows "| 24| 8.14 | 8.14 and
    after" and "| 25| 9.1.0 | 9.1.0 and after"
    ([compatibility.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/releases/compatibility.adoc), v9.8.2);
    "The TestKit does not currently expose fine-grained control over all environmental aspects—for example, selecting a JDK for the executed
    build." ([test_kit.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/plugin-development/test_kit.adoc)).
12. **Node.js is installed under the Gradle user home.** `BaseNodeJsRootExtension`: `private val gradleHome = project.gradle.gradleUserHomeDir.also {`
    and `var installationDir: File = gradleHome.resolve("nodejs")`.
    [source](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/targets/web/nodejs/BaseNodeJsRootExtension.kt), Kotlin v2.4.21.
13. **Node.js setup is a dependency of the js test task.** `KotlinNodeJsIr.configureTestDependencies(test: KotlinJsTest, binary: JsIrBinary)`:
    `test.dependsOn(project.nodeJsSetupTaskProvider)` and `test.dependsOn(`, `nodeJsRoot.npmInstallTaskProvider,`.
    [source](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/targets/js/ir/KotlinNodeJsIr.kt), Kotlin v2.4.21.
14. **The js test task is an `AbstractTestTask` with Gradle's `TestFilter`.** `KotlinTest`: `abstract class KotlinTest` ... `) : AbstractTestTask() {`
    and `filterExt.isFailOnNoMatchingTests = false`
    ([KotlinTest.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/tasks/KotlinTest.kt), Kotlin v2.4.21);
    `TestFilter`: "Returns the included test name patterns." and "Returns whether the task should fail if no matching tests where found.
    The default is true."
    ([TestFilter.java](https://github.com/gradle/gradle/blob/v9.8.2/platforms/software/testing-base/src/main/java/org/gradle/api/tasks/testing/TestFilter.java), v9.8.2).
15. **Gradle prescribes two runs to test configuration-cache compatibility.** "A single run is not enough to verify Configuration Cache
    compatibility: a build can succeed under CC simply because the entry was stored, while a silent CC incompatibility only surfaces on the
    second run when Gradle tries to load that entry back." and the sample's `require(result.output.contains("Reusing configuration cache."))`.
    [configuration_cache_debugging.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/optimizing-builds/configuration-cache/configuration_cache_debugging.adoc)
    and [BuildLogicFunctionalTest.kt](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/snippets/optimizing-builds/configuration-cache/testKit/kotlin/src/test/kotlin/org/example/BuildLogicFunctionalTest.kt), Gradle v9.8.2.
16. **`setup-gradle` caches `caches`/`notifications` and wrapper distributions, writes only from the default branch.** "By default, The
    `setup-gradle` action will only write to the cache from Jobs on the default (`main`/`master`) branch." and "As well as any wrapper
    distributions, the action will attempt to save and restore the `caches` and `notifications` directories from Gradle User Home."
    [setup-gradle.md](https://github.com/gradle/actions/blob/v6.4.0/docs/setup-gradle.md), `gradle/actions` v6.4.0.
17. **vanniktech's tests resolve the plugin from `mavenLocal`, run a Gradle-version matrix with `--configuration-cache`, run the Kotlin compiler
    in-process, and retry on CI.** `tasks.register("integrationTest", Test::class) { dependsOn(tasks.publishToMavenLocal, ...` and `maxRetries = 2`
    ([plugin/build.gradle.kts](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/plugin/build.gradle.kts), 0.37.0);
    `val arguments = mutableListOf(task, "--stacktrace", "--configuration-cache")`, `.withGradleVersion(options.gradleVersion.value)`,
    the generated settings `pluginManagement {` `repositories {` `mavenLocal()`, and `appendLine("kotlin.compiler.execution.strategy=in-process")`
    ([ProjectSpecRunner.kt](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/plugin/src/integrationTest/kotlin/com/vanniktech/maven/publish/util/ProjectSpecRunner.kt), 0.37.0).
18. **MyGameTools' wiring and its reason.** `tasks.register<Test>("integrationTest")` ... `includeTestsMatching("gametools.testing.integration.*")`
    and "The TestKit classes spawn nested Gradle builds - too heavy for `./gradlew build` (which runs `test` via `check`)."
    [build-logic/build.gradle.kts](https://github.com/SpartanLabsGaming/MyGameTools/blob/dc5214f/build-logic/build.gradle.kts), `dc5214f`
    (the file at `dc5214f`; read from the local clone, where the sentence wraps across two comment lines).
19. **MyGameTools' CI timing.** GitHub Actions run 38693939100 (2026-10-15), job `build-logic (TestKit)`: `started_at` `2026-10-15T06:41:38Z`,
    `completed_at` `2026-10-15T06:45:00Z`; step "Run ./gradlew :build-logic:integrationTest --stacktrace" `2026-10-15T06:41:49Z` to
    `2026-10-15T06:44:58Z`. [run](https://github.com/SpartanLabsGaming/MyGameTools/actions/runs/38693939100), `gh api repos/SpartanLabsGaming/MyGameTools/actions/runs/38693939100/jobs`.
20. **GeneralTools' CI timing and Node tasks.** Run 39507508729 (2026-10-20), job `build` on `ubuntu-latest`: log lines
    "Starting a Gradle Daemon (subsequent builds will be faster)" at 22:21:40, "> Task :compileKotlinJs" at 22:22:38, "> Task :kotlinNodeJsSetup" at
    22:22:39, "> Task :kotlinNpmInstall" at 22:22:43, "> Task :kotlinWasmToolingSetup" at 22:23:10 and "BUILD SUCCESSFUL in 1m 51s".
    [run](https://github.com/SpartanLaboratories/GeneralTools/actions/runs/39507508729), `gh run view 39507508729 --log`.
21. **Large KMP libraries ship no build-logic tests.** Contents API listings at the pinned refs return only `main` under `build-logic/src` (Ktor
    `3.6.0`), `build-support/src` (Okio `parent-3.18.2`), `buildSrc/src` (Coil `3.6.3`), `build-logic/src` (kotlinx-io `0.9.1`, workflow-kotlin
    `40d22c9eb443`); `repos/ktorio/ktor/contents/build-logic/src/test?ref=3.6.0` returns "Not Found". [Ktor 3.6.0](https://github.com/ktorio/ktor/tree/3.6.0/build-logic/src), read 2026-10-20.
22. **Embedded Kotlin of Gradle 9.0.0 is 2.2.0 (reused: the floor test's reason).** Table row "| 2.2.0  | 9.0.0 | 2.2" under "Embedded Kotlin
    version / Minimum Gradle version / Kotlin Language version". [compatibility.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/releases/compatibility.adoc), v9.8.2.
23. **The Dokka failure, and the root-hoist remedy, as established by the build-logic finding (reused).** Dokka's warning text: "The most
    common cause is a Gradle limitation: the plugins applied to subprojects should be consistent. … 1. Apply the Dokka and Kotlin plugins to the
    root project using the `plugins {}` DSL." ([KotlinAdapter.kt](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-runners/dokka-gradle-plugin/src/main/kotlin/adapters/KotlinAdapter.kt), Dokka v2.2.0);
    `gametools-core-5.0.0-javadoc.jar` is 261 bytes with 0 `.html` entries and `gametools-core-5.1.0-javadoc.jar` is 1,690,968 bytes with 415
    ([5.0.0](https://repo1.maven.org/maven2/io/github/spartanlabsgaming/gametools-core/5.0.0/gametools-core-5.0.0-javadoc.jar),
    [5.1.0](https://repo1.maven.org/maven2/io/github/spartanlabsgaming/gametools-core/5.1.0/gametools-core-5.1.0-javadoc.jar), listed 2026-10-20 in the build-logic finding).
