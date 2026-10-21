# `graphics-bom`: building, publishing and depending on a BOM for a Kotlin Multiplatform plus JVM family

- **Issue:** [SpartanLaboratories/SpartanGraphics#3](https://github.com/SpartanLaboratories/SpartanGraphics/issues/3), question 5 of the
  [research README](README.md).
- **Read:** 2026-10-20. Pinned sources: Gradle docs and source at `v9.8.2` and `v9.7.1`; Kotlin Gradle plugin (KGP) at `v2.4.21`;
  vanniktech `0.37.0`; kotlinx.coroutines `1.11.0`; kotlinx.serialization `v1.9.0`; Ktor `3.6.0`; Arrow `2.2.3`; Koin `4.2.2`;
  Okio `parent-3.18.2`; OkHttp `parent-5.5.0`; Jackson `2.22.4`. Published `.pom` and `.module` files were downloaded from
  `repo1.maven.org` and decoded read-only.
- **Method limit:** nothing was built or run, and no Gradle task was executed. Every behaviour below comes from Gradle's and
  Kotlin's documentation and source, Kotlin's own integration tests, YouTrack and GitHub issues, and the published metadata of the
  libraries named. Where a statement is an inference from those, it says so, and *Known gaps* lists the runs that settle it.

## 1. Question

How is `graphics-bom` built and published, and how does each library module depend on it, so that a consumer's graphics artifacts
always resolve to one version, including the per-target artifacts of the Kotlin Multiplatform module `graphics-core` (publications
`graphics-core`, `graphics-core-jvm`, `graphics-core-js`) and the `kotlin("jvm")` + `java-library` module `graphics-desktop`? An
answer must give: every library module depending on the BOM through `api(platform(...))` from its first commit; a BOM that constrains
only this repository's artifacts and carries the shared version; aligned versions for a Gradle JVM app, a Kotlin Multiplatform
library with a `js` target and a composite-build consumer; a BOM a Maven consumer can import to get versions for the artifacts it
names; no dependency cycle that breaks configuration; configuration-cache compatibility; the exact `graphics-bom/build.gradle.kts`,
the convention-plugin lines for both module kinds, and the expected published BOM content; and a cheap way to keep the constraint
list complete without cross-project configuration.

## 2. Bottom line

- **`java-platform` with a hand-written constraint list, published through vanniktech's `JavaPlatform()`, is the shape.** It is Gradle's
  documented "align a multi-module family" pattern (platform project with `constraints { api(project(":core")) … }`, each module
  with `api(platform(project(":platform")))`), and Gradle runs that exact sample (`publishAllPublicationsToMavenRepository`) in its
  documentation tests, so it creates no cycle. Arrow's `arrow-stack` and Koin's `koin-bom` are hand lists; every *generated*
  BOM read (coroutines, serialization, Ktor, Okio, OkHttp) reads `rootProject.subprojects` or a tag registry, which is
  cross-project configuration. OkHttp 5.5.0's generated BOM shows the failure of generating: it constrains tests and samples, one at
  version `unspecified`.
- **A `constraints { api(project(":graphics-core")) }` entry yields the root coordinate only.** Gradle's publication resolver takes
  the top-level publications, and the Kotlin plugin's target publications are children of the root component. The BOM therefore
  lists the per-target coordinates (`graphics-core-jvm`, `graphics-core-js`) as explicit string constraints. Gradle consumers
  resolve the root and follow `available-at` to the target module at the same version, so they need only the root entry. Maven
  consumers, and the POMs Gradle writes for JVM modules and Kotlin target modules (which name `graphics-core-jvm`, not the root),
  need the `-jvm` entry; `-js` is listed for completeness and for Gradle consumers that name it. The BOM holds four entries:
  `graphics-core`, `graphics-core-jvm`, `graphics-core-js`, `graphics-desktop`.
- **Each module depends on the BOM through `api(platform(...))`, set once in each convention plugin.** For `graphics-core` it is
  `commonMain.dependencies { api(project.dependencies.platform(project(":graphics-bom"))) }` (KGP 2.4.0 removed the
  `KotlinDependencyHandler.platform()` shortcut, and `project.dependencies.platform` is the form its deprecation names); for
  `graphics-desktop` it is `dependencies { api(platform(project(":graphics-bom"))) }`. Kotlin's own integration test publishes a
  `commonMain` platform dependency as a `type pom` / `scope import` entry in the target POM, and the published `.module` of every
  module records it as `org.gradle.category: platform` with `endorseStrictVersions: true`.
- **Alignment on a mixed request is "highest BOM wins, then its constraints apply".** With `graphics-core` 0.2.0 requested through
  SpartanUI and `graphics-desktop` 0.1.0 pinned, Gradle selects `graphics-bom` 0.2.0 (highest of the requested versions of the BOM
  module), whose constraint raises `graphics-desktop` to 0.2.0. Without the BOM, desktop 0.1.0 stays and runs against core 0.2.0,
  which is the kotlinx.coroutines failure that motivated that project's BOM (`NoSuchMethodError`, issue #2865). Use `platform`, never
  `enforcedPlatform`, in a library: Gradle's documentation says `enforcedPlatform` is for applications, because the forced versions
  propagate to every consumer.
- **Completeness is kept by a build-time check, not by generation.** `graphics-bom/build.gradle.kts` states the modules by kind
  (two lists), and a small typed task in `build-logic`, `verifyBomCoverage`, hooked to the BOM's `check`, scans
  `*/build.gradle.kts` for the `spartangraphics.kmp-library` and `spartangraphics.jvm-library` ids and fails when a module on disk is
  missing from, or extra in, the lists. It reads files only (no `Project` of another module), so it is configuration-cache and
  Isolated-Projects clean. A later Level 3 check (question 6) can compare the generated POM with the publication task names.
- **The platform dependency in `commonMain` is the least-evidenced step.** Kotlin's two own libraries that depend on their BOM
  (coroutines, serialization) do it from `jvmMain` only and publish a BOM with no `-js` entry; no rationale was found. The evidence
  for `commonMain` is Kotlin's source (a `kotlin-api` consumer accepts a `java-api` platform variant), Koin's documented KMP setup
  (BOM in the multiplatform module's dependencies, JS and Wasm among the supported targets), and Kotlin's integration test; no published KMP library with a `js` target that
  depends on its own BOM from `commonMain` was found. The implementation runs one check (section 6.6) before G1 closes; the fallback
  is moving the one line to `jvmMain`.

## 3. Findings

Tiers: **consensus** = official documentation, specification or source, or several independent credible sources; **practice** =
what mature libraries visibly do; **opinion** = one source; **inference** = derived here from sources that do not state it.

### F1. The documented shape, and why it has no cycle (consensus; Gradle 9.7.1 and 9.8.2)

- Gradle's "How to Align Dependency Versions" page, *Option 3: Using the Java Plugin*, describes this repository's problem: a
  consumer that declares `core` 1.0 and `lib` 1.1 gets the mix "leading to version misalignment", and the fix is "a platform module
  that enforces constraints". The platform declares `constraints { api(project(":core")); api(project(":lib")); … }`; "Each module
  should declare a dependency on the platform": `api(platform(project(":platform")))`. The sample is executed in Gradle's
  documentation tests (`align-dependency-versions.sample.conf`: `args: publishAllPublicationsToMavenRepository`), so the
  module-to-platform and platform-to-module edges, resolution and publication all work together
  ([how_to_align_dependency_versions.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/how-to/dependency-management/how_to_align_dependency_versions.adoc),
  [sample build](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/snippets/how-to/dependency-management/dependency-management/dependency-management/managingTransitiveDependencies-dependencyAlignmentWithPlatform/kotlin/core/build.gradle.kts),
  [sample test conf](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/snippets/how-to/dependency-management/dependency-management/dependency-management/managingTransitiveDependencies-dependencyAlignmentWithPlatform/tests/align-dependency-versions.sample.conf)).
- **Why it is not a cycle.** Constraints add no dependency: "constraints will only apply if such a component is added to the
  dependency graph" ([java_platform_plugin.adoc, v9.8.2](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/core-plugins/java_platform_plugin.adoc)).
  A constraint on the module that is itself resolving refers to the root component and changes nothing. The platform has no
  artifacts, so no task of `:graphics-core` depends on a task of the BOM and none of the BOM on a module's tasks. A project constraint
  is a `ProjectDependency`, created without reading the other project's state, so it adds no cross-project configuration.
- **Not a cycle that matters, found elsewhere.** kotlinx.coroutines' PR #2952 recorded a real cycle: "`kotlin-compiler-embeddable`
  depends on `kotlinx-coroutines-core`, and after dependency alignment, in order to build `kotlinx-coroutines-core` of our
  `x-SNAPSHOT` version we have to first resolve `kotlin-compiler-embeddable` dependency to the very same version". That is the build's
  own compiler classpath depending on the library being built, which does not exist here (no graphics artifact is a dependency of the
  Kotlin compiler, Dokka or vanniktech).
- **Published form.** "The project notation will become a classical `group:name:version` notation in the published metadata", and a
  published platform "will generate a BOM file for the platform, with a `<dependencyManagement>` block where its `<dependencies>`
  correspond to the constraints defined in the platform module" (same page). `java-platform` applies `BasePlugin` and
  `JvmEcosystemPlugin` (`JavaPlatformPlugin.apply`, v9.8.2), so the BOM project has `assemble`, `check` and `build`.
- **No `allowDependencies()`.** "By default, in order to avoid the common mistake of adding a dependency in a platform instead of a
  constraint, Gradle will fail if you try to do so." Constraints-only needs nothing; leaving the default makes a stray dependency fail
  fast. Koin's `koin-bom` calls `javaPlatform { allowDependencies() }` although its list holds only constraints.
- **Unpublished-project deprecation.** Gradle's 9 upgrade guide: publishing a project that depends on a project without a publication
  "will be forbidden and will cause the build to fail" from Gradle 10. Every module depends on the BOM project, so the BOM must
  always have a publication; `JavaPlatform()` creates it at apply time (F7).

### F2. A project constraint on a KMP project produces the root coordinate only (consensus on the mechanism; practice confirms)

- `DefaultProjectDependencyPublicationResolver` (Gradle v9.8.2) turns a project into published coordinates from its publications:
  "For all published components, find those that are not children of other components. These are the top-level publications." The
  Kotlin plugin creates the `kotlinMultiplatform` root publication from the root software component, and the target publications
  (`jvm`, `js`) from target components that are children of it (`createRootPublication` / `createTargetPublications` in KGP
  `Publishing.kt`, v2.4.21). The single top-level publication is the root, `graphics-core`.
- Three libraries show the consequence in their published BOMs. Koin's `koin-bom` 4.2.2 uses `api(project(":core:koin-core"))` for the
  Kotlin Multiplatform `koin-core`, and the published BOM lists `koin-core` but not `koin-core-js`; its author added
  `api("io.insert-koin:koin-core-jvm:$version") //Check later KMP Bom` by hand. Okio's `bom.kt` iterates `KotlinMultiplatformExtension`
  targets and builds `"$group:$name-<target>:$version"` strings itself ("Collect all the root project's multiplatform targets and add
  them to the BOM."). kotlinx.coroutines iterates `publishing.publications` and adds each non-klib publication's `artifactId`.
- A KMP `jvm` or `js` coordinate is a string constraint, spelled `"$group:graphics-core-jvm:$version"`, with the target suffix from
  the default target names (`jvm()`, `js()`): the artifactId is the project name plus `-<target>`, and a custom `js("browser")`
  would publish `-browser` (GeneralTools `kmp-maven-central-publishing.md` at `d424aaf`, section on artifact names).

### F3. Which of the coordinates a consumer actually needs (consensus on the metadata; inference on Gradle's graph behaviour)

- **Gradle metadata, root.** The root `.module` of a KMP library holds variants that redirect: `available-at` names the target module
  at the same version, and per the specification "`dependencies` … must not be present when `available-at` is present." Okio 3.18.2's
  root `.module`: `jvmApiElements-published` has `available-at` `okio-jvm` version `3.18.2`; `jsApiElements-published` has `okio-js`
  `3.18.2`. A Gradle consumer that depends on `graphics-core` selects a version of the *root* (the BOM's constraint applies to that
  module), and the redirect then names the target at that same version. So the root constraint is what governs the family for a Gradle
  consumer. (Inference: that Gradle's conflict resolution treats the redirect target as an ordinary module at the root's version.
  The specification fixes the version in the redirect; no Gradle test of "constraint on the root, target resolved through
  `available-at`" was found. The three `GradleModuleMetadataAvailableAtIntegrationTest` cases read cover transitivity, excludes
  and artifact notation, not version conflict.)
- **POMs name the target.** For a plain JVM module that depends on a KMP project, Gradle writes the *root* in the `.module` and the
  *target* in the `.pom`. kotlinx-coroutines-reactive 1.11.0: its `.module` lists `kotlinx-coroutines-core` and its `.pom` lists
  `kotlinx-coroutines-core-jvm` (scope `compile`). A Kotlin target's own POM does the same: KGP's `PublishingIT.testPomRewriter`
  asserts `-jvm` artifactIds for project dependencies ("pom rewriting"). So `graphics-desktop-0.1.0.pom` will name
  `graphics-core-jvm`, and a Maven consumer of `graphics-desktop` resolves `graphics-core-jvm` transitively. For GeneralTools, the same
  finding states that Maven-only consumers must depend on the `-jvm` artifact directly.
- **Maven applies only the consumer's own import.** The BOM's documented use is that "Other projects that wish to use the library
  should import this POM into the dependencyManagement section of their POM" ([Maven introduction to the dependency mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html),
  read 2026-10-20). A library's own `<dependencyManagement>` import (which Gradle writes into every module POM from
  `api(platform(...))`) is for building that library; the Maven consumer must import `graphics-bom` itself. Managed entries then need
  to cover every artifactId the consumer's graph names, which is why `graphics-core-jvm` is listed. (That Maven ignores a
  dependency's own `<dependencyManagement>` is general Maven behaviour; no sentence of Maven documentation quoted here states it.)
- **Composite builds substitute project names only.** Per-target coordinates are not project names. KT-56161 ("As Designed"):
  "Just including the other multiplatform build will only work when a consumer build references the dependency using the same module
  as the producer build (the mpp dependency, without the -platform classifier)." A BOM entry for `graphics-core-jvm` is inert in a
  composite build unless a consumer names that coordinate, in which case it fails to resolve; consumers use the root coordinates.
- **Conclusion:** keep three entries for the KMP module (root via `project()`, `-jvm` and `-js` as strings) and one for the JVM
  module. The `-js` entry costs nothing, is symmetrical with Ktor's and Okio's BOMs, and serves a Gradle consumer that names it
  directly. Arrow's `arrow-stack` 2.2.3 shows the opposite risk: `arrow-core-js` is on Central at 2.2.3, and the hand-written BOM
  lists `-jvm` and root only, so a consumer that names a `-js` artifact gets no managed version from it.

### F4. The platform dependency from a KMP module (consensus on the DSL; practice and inference on the targets)

- **DSL.** `KotlinDependencyHandler` (KGP v2.4.21) is `interface KotlinDependencyHandler : HasProject`, with `api(dependencyNotation: Any)`
  and a `project(path: String)` member, and no `platform`. In 2.3.20 `platform(notation)` and its `Action` overload were
  `@Deprecated(…, replaceWith = ReplaceWith("project.dependencies.platform(notation)"), level = DeprecationLevel.ERROR)`; Kotlin 2.4's
  compatibility guide lists `KotlinDependencyHandler.platform()` and `enforcedPlatform()` among "deprecated Kotlin Gradle plugin APIs"
  removed in 2.4.0. KT-58759's rationale: "we decided to deprecate and later remove these delegating functions in favor of the
  explicit use of `project.dependencies.XYZ()`". The alternative, `dependencies { "commonMainApi"(platform(project(":graphics-bom"))) }`,
  is the second workaround the Kotlin team gave in KT-40489; it names a configuration by string and gains nothing.
- **Kotlin's documentation** has no page on BOM or platform dependencies in multiplatform source sets (searched
  `kotlin-web-site` `docs/topics` for `platform(`, `BOM`, `bill of materials`: only the Gradle-JVM `kotlin-bom` section and the Maven page).
  The authority is KGP's integration test: `PublishingIT.testPomRewriter` builds a KMP project with
  `sourceSets.commonMain.dependencies { api(project.dependencies.platform("org.springframework.boot:spring-boot-dependencies:3.4.0")) … }`
  and asserts that the `-jvm` POM's `dependencyManagement` holds `spring-boot-dependencies` `3.4.0` with `scope = "import"` and
  `type = "pom"`.
- **Published where.** A `commonMain` `api` dependency reaches the root `metadataApiElements` variant and each target module:
  Okio 3.18.2's root `metadataApiElements` lists `kotlin-stdlib`, and the target modules list their own. The platform dependency
  follows the same configurations (inference from the mechanism and the test above; a published KMP library with a `js` target and a
  `commonMain` platform dependency was not found, F9).
- **`js` (klib) resolution.** KGP's usage rules make a `java-api` variant acceptable to a `kotlin-api` consumer
  (`KotlinUsagesCompatibility.execute`: `consumerValue?.name == KOTLIN_API && producerValue?.name == JAVA_API -> compatible()`), a
  `java-runtime` variant acceptable to `kotlin-api` and `kotlin-runtime` consumers, and `kotlin-metadata` consumers accept the
  `java-*` usages (`KotlinMetadataCompatibility`). The published BOM variants carry exactly `org.gradle.category: platform` and
  `org.gradle.usage: java-api` or `java-runtime` and no other attribute (decoded: okio-bom 3.18.2, arrow-stack 2.2.3, koin-bom
  4.2.2), so a `js` or metadata resolution can select them. Koin's documented Kotlin Multiplatform setup puts
  `implementation(project.dependencies.platform(libs.koin.bom))` in the multiplatform module's `Dependencies` block for a project whose supported targets
  include JS and Wasm. KT-65577 (closed *Incomplete*) is a user project with `jvm()` and `js { nodejs() }` and
  `implementation(project.dependencies.platform("io.ktor:ktor-bom:…"))` in `commonMain`; its complaint is about POM contents, not about
  resolution failing.
- **Kotlin's own libraries differ.** kotlinx.coroutines 1.11.0 (`buildSrc` `bom-conventions.gradle.kts`) and kotlinx.serialization
  v1.9.0 (same file name) apply `api(project.dependencies.platform(project(bomProjectPath)))` to `jvmMain` for KMP projects and
  `"api"(platform(project(…)))` for JVM ones. Published, only `-jvm` modules carry the BOM: `kotlinx-coroutines-core-jvm` has it in
  both `jvmApiElements-published` and `jvmRuntimeElements-published`, `kotlinx-coroutines-core-js` and the root have none. No
  commit message or PR text states why; PR #2952 and issue #2865 concern the JVM failure only. Ktor, Okio, Arrow, Koin, Coil and
  Wire modules were checked and none depends on its own BOM (their BOMs serve consumers only).

### F5. What the published modules record (consensus: decoded published metadata)

- `kotlinx-coroutines-core-jvm-1.11.0.module`, in `jvmApiElements-published` and `jvmRuntimeElements-published`:
  `"module": "kotlinx-coroutines-bom"`, `"version": {"requires": "1.11.0"}`, `"attributes": {"org.gradle.category": "platform"}`,
  `"endorseStrictVersions": true`. kotlinx-serialization-core-jvm 1.9.0 is the same. jackson-databind 2.22.4's `.module` has
  `jackson-bom` `{"requires": "2.22.4"}` with `org.gradle.category: platform` and `endorseStrictVersions` in `apiElements` and
  `runtimeElements` (Jackson's POM carries no import; it inherits its BOM through parent POMs).
- `kotlinx-coroutines-core-jvm-1.11.0.pom` and `kotlinx-coroutines-reactive-1.11.0.pom` carry
  `<dependencyManagement>` with `kotlinx-coroutines-bom` `<type>pom</type><scope>import</scope>`.
- `platform()` enables endorsement by default ("Enables the `endorseStrictVersions` behavior by default, enforcing strict versions
  defined in the platform"). With a constraints-only BOM that uses `requires` (as every decoded BOM does), endorsement has nothing to
  endorse. The BOM must never use `strictly`: it would propagate to every consumer through `endorseStrictVersions`.

### F6. Alignment on a mixed request, and `enforcedPlatform` (consensus: Gradle documentation)

- "Gradle will consider *all* requested versions, wherever they appear in the dependency graph. By default, it will select the
  *highest* one out of these versions" ([dependency_constraints_conflicts.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/dependency-management/basics/dependency_constraints_conflicts.adoc)).
  `graphics-bom` is a module like any other, and a platform is "a special kind of software component" (java_platform_plugin.adoc).
- Worked case, with the published metadata of this design: `graphics-desktop` 0.1.0 depends on `graphics-core` 0.1.0 and
  `graphics-bom` 0.1.0; `graphics-core` 0.2.0 (via SpartanUI) depends on `graphics-bom` 0.2.0. The BOM module has two requests, so
  0.2.0 is selected and only its constraints apply. They state `graphics-desktop` `requires 0.2.0`, and the consumer's `graphics-desktop`
  0.1.0 request loses to the constraint; desktop 0.2.0 is resolved. The mix that failed at runtime in kotlinx.coroutines (core 1.5.1
  with jdk8 1.4.3, issue #2865: "To prevent such situation, both modules should depend on `kotlinx-coroutines-bom` of the corresponding
  version") cannot resolve silently. A `strictly` pin by the consumer on the older version makes resolution fail loudly instead.
- **Limits.** Maven does not align (the consumer must import the BOM, F3). A consumer that uses `resolutionStrategy.force` or
  `dependencySubstitution` can override alignment, and a graph containing only pre-BOM versions has nothing to align.
- **`enforcedPlatform`.** Gradle v9.8.2: "`enforcedPlatform` should generally only be used in applications, not in libraries or other
  components consumed by others"; "Because `enforcedPlatform` converts all BOM constraints to `strictly`, these strict versions
  propagate transitively to every consumer of your project." (v9.7.1: "When using `enforcedPlatform`, exercise caution if your
  software component is intended for consumption by others.") The modules use `platform`. An application (the game client) may use
  `enforcedPlatform` on the BOM; nothing in this repository needs it.
- **Disambiguation of the BOM's variants.** The consumable `enforcedApiElements` and `enforcedRuntimeElements` the plugin creates are
  not published: every decoded BOM `.module` has exactly two variants, `apiElements` and `runtimeElements` (`enforcedPlatform` on a
  published BOM is derived by the consumer).

### F7. vanniktech `JavaPlatform` (consensus: source at 0.37.0; practice on Central)

- `public class JavaPlatform : Platform()` with `javadocJar = JavadocJar.None()` and `sourcesJar = SourcesJar.None()`; its
  `configure(project)` does `check(project.plugins.hasPlugin("java-platform"))` and
  `project.gradlePublishing.publications.create(PUBLICATION_NAME, MavenPublication::class.java) { it.from(project.components.getByName("javaPlatform")) }`.
  `MavenPublishBaseExtension.configureBasedOnAppliedPlugins` picks it when `java-platform` is applied; its docs say
  `mavenPublishing { configure(JavaPlatform()) }` ([Platform.kt](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/plugin/src/main/kotlin/com/vanniktech/maven/publish/Platform.kt),
  [MavenPublishBaseExtension.kt](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/plugin/src/main/kotlin/com/vanniktech/maven/publish/MavenPublishBaseExtension.kt),
  [docs/what.md](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/docs/what.md)). With the publishing convention's
  `pluginManager.withPlugin("java-platform") { mavenPublishing { configure(JavaPlatform()) } }` (question 4, section 6.3), the
  `check` passes because the callback runs after the plugin is applied.
- The publication is `pom` packaging with a `.module`, and the sources and Javadoc jars are absent. Sonatype: "Projects with packaging
  other than `pom` have to supply JAR files that contain Javadoc and sources." Precedent on Central through vanniktech: `okio-bom`
  3.18.2 (built with `com.vanniktech.maven.publish.base` and `from(project.components.getByName("javaPlatform"))`, vanniktech 0.37.0 in
  Okio's catalog) publishes `.module`, `.pom`, and `.asc`/`.md5`/`.sha1` for both, and nothing else; `arrow-stack` 2.2.3 (vanniktech
  0.36.0, `configureBasedOnAppliedPlugins()`, `signAllPublications()`, `publishToMavenCentral(automaticRelease = true)`) publishes the
  same files.
- `signAllPublications()` signs the BOM's `.pom` and `.module` (the `.asc` files above). `coordinates(group, name, version)` needs no
  KMP special case for a BOM.

### F8. How comparable BOMs produce the list (practice) and what generation costs

| BOM @ read ref | Production | Cross-project? | Completeness evidence |
| --- | --- | --- | --- |
| kotlinx-coroutines-bom, tag `1.11.0` | loop over `rootProject.subprojects`, `evaluationDependsOn(it.path)`, `publishing.publications.all`; skips `*-kotlinMultiplatform`, `*-metadata` and any publication with a `klib` artifact | yes (`rootProject`, `evaluationDependsOn`) | complete by construction for JVM and root; native and `-js` skipped on purpose |
| kotlinx-serialization-bom, tag `v1.9.0` | same script | yes | same |
| ktor-bom, tag `3.6.0` | `projectsWithTag(ProjectTag.Library)`, publications minus `-metadata` / `-kotlinMultiplatform`, `dependencyConstraints.addAllLater` | yes | 883 entries, every target |
| okio-bom, tag `parent-3.18.2` | `collectBomConstraints()` in `build-support`: `rootProject.subprojects`, `plugins.withId("com.vanniktech.maven.publish.base")`, every KMP target | yes | 45 entries, every target |
| okhttp-bom, tag `parent-5.5.0` | `project.rootProject.subprojects.forEach { api(subproject) }` plus two manual strings | yes | over-inclusive: tests and samples (see pitfalls) |
| arrow-stack, tag `2.2.3` | hand list of strings (`api("io.arrow-kt:arrow-core:$version")`) | no | 45 entries; `-jvm` yes, `-js` absent although published |
| koin-bom, tag `4.2.2` | hand list of `api(project(":…"))`, two manual `-jvm` strings | no | 26 entries; `-js` absent |
| osipxd/encrypted-datastore (small published library) | hand list of `projects.*` accessors, plus third-party `androidx.datastore` | no | not examined further |

Every generator needs another project's `plugins` or `publishing` state, which is what Isolated Projects forbids and what this
repository's build avoids.

## 4. How comparable projects do it

Published BOMs, decoded from `repo1.maven.org` on 2026-10-20 (entries are `dependencyManagement` rows in the `.pom`, equal to the
`dependencyConstraints` of the `.module` where one exists):

| BOM @ version | Files published | Entries | Root | `-jvm` | `-js` | Native / wasm | Non-library artifacts | Do the modules depend on the BOM? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `org.jetbrains.kotlinx:kotlinx-coroutines-bom:1.11.0` | `.pom` only (Gradle metadata deliberately disabled in the BOM's build script) | 17 | yes (`core`, `test`) | yes (`core-jvm`, `test-jvm`) | no | no | no | JVM modules and `jvmMain` of KMP modules: yes |
| `org.jetbrains.kotlinx:kotlinx-serialization-bom:1.9.0` | `.pom` only | not counted | yes | yes | no | no | no | same |
| `io.ktor:ktor-bom:3.6.0` | `.pom`, `.module` | 883 | yes | 113 | 74 | 505 native, 73 wasm-js | no | no (`ktor-http`, `-jvm`, `-js` checked) |
| `com.squareup.okio:okio-bom:3.18.2` | `.pom`, `.module`, signatures | 45 | yes | 2 | 3 | 32 native, 4 wasm | no | no |
| `io.arrow-kt:arrow-stack:2.2.3` | `.pom`, `.module`, signatures | 45 | yes | 19 | 0 | 0 | no | no |
| `io.insert-koin:koin-bom:4.2.2` | `.pom`, `.module` | 26 | yes | 2 | 0 | 0 | no | no |
| `com.squareup.okhttp3:okhttp-bom:5.5.0` | `.pom`, `.module` | 34 | yes | 1 (`okhttp-jvm`) | n/a | n/a | yes: `android-test`, `container-tests`, `okhttp-hpacktests`, `okhttp-osgi-tests`, `okhttp-testing-support`, samples | no |
| `com.fasterxml.jackson:jackson-bom:2.22.4` (per-module platform dependency) | n/a | ~70 `artifactId`s | n/a | n/a | n/a | n/a | n/a | `jackson-databind` `.module`: yes, with `endorseStrictVersions` |

Representative code, cited by symbol at the pinned tag:

- Gradle `v9.8.2`, sample `managingTransitiveDependencies-dependencyAlignmentWithPlatform`: `platform/build.gradle.kts` holds
  `plugins { `java-platform` }` and `dependencies { constraints { api(project(":core")); api(project(":lib")); api(project(":utils")) } }`;
  `core/build.gradle.kts` holds `dependencies { api(platform(project(":platform"))) … }`.
- kotlinx.coroutines `1.11.0`, `buildSrc/src/main/kotlin/bom-conventions.gradle.kts`: for KMP projects
  `kotlinExtension.sourceSets.getByName("jvmMain").dependencies { api(project.dependencies.platform(project(":kotlinx-coroutines-bom"))) }`,
  otherwise `dependencies { "api"(platform(project(":kotlinx-coroutines-bom"))) }`.
  `kotlinx-coroutines-bom/build.gradle.kts`: `mavenBom` publication `from(components["javaPlatform"])`,
  and the comment "Skip platform artifacts (like *-linuxx64, *-macosx64) … It leads to inconsistent bom when publishing from
  different platforms (e.g. on linux it will include only linuxx64 artifacts and no macosx64)". Those are host-dependent Kotlin/Native
  targets; `jvm` and `js` publish from any host, so the reason does not apply to this repository.
- Okio `parent-3.18.2`, `okio-bom/build.gradle.kts`: `plugins { id("com.vanniktech.maven.publish.base"); id("java-platform") }`,
  `collectBomConstraints()`, `publications.create("maven", MavenPublication::class) { from(project.components.getByName("javaPlatform")) }`.
- Koin `4.2.2`, `projects/bom/koin-bom/build.gradle.kts`: `` `java-platform` ``, `javaPlatform { allowDependencies() }`,
  `constraints { api(project(":core:koin-core")); api("io.insert-koin:koin-core-jvm:$version") //Check later KMP Bom … }`.
- Arrow `2.2.3`, `arrow-libs/stack/build.gradle.kts`: `` `java-platform` `` plus the vanniktech plugin, `mavenPublishing {
  configureBasedOnAppliedPlugins(); publishToMavenCentral(automaticRelease = true); signAllPublications() }`, and one
  `api("io.arrow-kt:…:$version")` line per artifact.
- Koin's KMP setup page: `implementation(project.dependencies.platform(libs.koin.bom))` in the multiplatform module's `Dependencies` block (the page does not show the enclosing source set).

`webtools-*` and `gametools-*` in this organisation publish no BOM; MyGameTools' `docs/plans/32-module-split/plan.md` section 2.5
deferred one.

## 5. Pitfalls and regrets

- **Generating from "every subproject" over-includes silently.** OkHttp 5.5.0's BOM, from `rootProject.subprojects.forEach { api(subproject) }`,
  constrains `android-test`, `container-tests`, `okhttp-hpacktests`, `okhttp-osgi-tests`, `guide`, `crawler`, `slack` and others, and
  one entry reads `<groupId>okhttp-parent</groupId><artifactId>samples</artifactId><version>unspecified</version>`. Harmless to
  Gradle consumers (constraints for modules not in the graph are inert) and wrong as a published statement.
- **Hand lists drift silently the other way.** Arrow publishes `arrow-core-js` and leaves it out of `arrow-stack`; Koin names
  `-jvm` by hand with a "Check later KMP Bom" comment. A missing entry produces no error anywhere.
- **`enforcedPlatform` in a library** propagates `strictly` to every consumer (F6).
- **`strictly` in the BOM** would propagate through `endorseStrictVersions`, which `platform()` turns on by default.
- **Per-target coordinates in composite builds** are never substituted (KT-56161); a BOM entry for them is inert there. Document the
  root coordinates as the only supported ones for Gradle consumers, and the `-jvm` coordinate for Maven.
- **Unversioned dependencies in a published KMP module** may lose their version in the `-jvm` POM when the version comes from a BOM
  (KT-65577, closed *Incomplete*, not reproduced here, one-source). SpartanUI should give its published `graphics-*` dependencies an
  explicit version or a project dependency, and keep the BOM as an alignment device.
- **`KotlinDependencyHandler.platform()` and `enforcedPlatform()` are gone in Kotlin 2.4.** Older snippets (Kotlin forum, Slack,
  KT-40489's first example) fail to compile on the 2.4.21 plugin; use `project.dependencies.platform(...)`.
- **A platform project needs a publication** once any module depends on it (F1, the Gradle 10 deprecation); the BOM never goes
  unpublished while modules reference it.
- **Dokka and the BOM.** Question 1's section 6.9 sketch of the publishing convention applies Dokka in its `plugins {}` block, which
  the BOM would also receive. Question 4 section 6.3 moves Dokka into `withPlugin` blocks for the two Kotlin kinds; that form is the
  one that keeps the BOM clean (this finding assumes it).
- **Native BOM entries depend on the publishing host** (the coroutines comment above). Not an issue for `jvm` and `js`; it becomes one
  if a Kotlin/Native target is added.

## 6. Recommendation for this repository

Use a `java-platform` BOM with a hand-written constraint list stated by module kind, published through the publishing convention's
`JavaPlatform()` selection, and put one `api(platform(...))` line in each library convention plugin. Generate nothing: every
generator in the comparable projects needs another project's state, which the build does not allow, and the hand list is four lines
today; guard it with a file-scanning check so a forgotten module fails `check` instead of passing silently. Reject
`enforcedPlatform` and `strictly` in the BOM (they leak to consumers), a `jvmMain`-only platform dependency (it leaves `js`
consumers unaligned, the one scenario the BOM exists for besides JVM), and `allowDependencies()` (it removes a safety net for no use).

### 6.1 `graphics-bom/build.gradle.kts`

```kotlin
import com.spartanlabs.graphics.build.VerifyBomCoverage

plugins {
    `java-platform`                       // constraints only: no allowDependencies(), so a stray dependency fails fast
    id("spartangraphics.publishing")      // applies com.vanniktech.maven.publish; selects JavaPlatform() for java-platform
}

// Every artifact this repository publishes, by kind. Adding a module to settings.gradle.kts means adding it here;
// verifyBomCoverage (below) fails `check` when this file and the modules on disk disagree.
val kmpModules = listOf("graphics-core")       // publications: <name> (root), <name>-jvm, <name>-js
val jvmModules = listOf("graphics-desktop")    // publication:  <name>

val bomGroup = group.toString()
val bomVersion = version.toString()            // the one shared version, from gradle.properties

dependencies {
    constraints {
        kmpModules.forEach { module ->
            api(project(":$module"))                       // root coordinate: what Gradle consumers resolve; version = project version
            api("$bomGroup:$module-jvm:$bomVersion")        // named by Maven consumers and by the POMs Gradle writes for JVM modules
            api("$bomGroup:$module-js:$bomVersion")         // named by Gradle consumers that skip the root
        }
        jvmModules.forEach { module -> api(project(":$module")) }
    }
}

mavenPublishing {
    pom {
        name.set("SpartanGraphics BOM")
        description.set("Bill of materials that keeps every SpartanGraphics artifact on one version.")
    }
}

val verifyBomCoverage = tasks.register<VerifyBomCoverage>("verifyBomCoverage") {
    group = "verification"
    description = "Fails when a library module on disk is missing from, or absent from disk but listed in, the BOM."
    moduleScripts.from(fileTree(isolated.rootProject.projectDirectory) { include("*/build.gradle.kts") })
    kotlinMultiplatformModules.set(kmpModules)
    kotlinJvmModules.set(jvmModules)
}
tasks.named("check") { dependsOn(verifyBomCoverage) }
```

Notes. `description`/`name` come from the module script, as in question 1 section 6.9. The BOM applies no Kotlin plugin, so it
gets no explicit-API setting, no ABI task (question 3) and no Dokka task (question 4); the root's `dokka(project(...))` list never
names it. `project(":$module")` inside `constraints {}` is `DependencyConstraintHandlerScope.project`, the same call as Gradle's
sample. The string form is used for `-jvm` and `-js` because a project notation cannot name a target publication (F2).
`bomGroup`/`bomVersion` are the BOM project's own `group` and `version`, which `gradle.properties` assigns to every project
(question 1, F5), so they equal every module's.

### 6.2 `build-logic`: the completeness task (sketch, not compiled)

```kotlin
// build-logic/src/main/kotlin/com/spartanlabs/graphics/build/VerifyBomCoverage.kt
package com.spartanlabs.graphics.build

abstract class VerifyBomCoverage : DefaultTask() {
    /** Each module's build script: the only file inputs, so the task is configuration-cache and cache safe. */
    @get:InputFiles @get:PathSensitive(PathSensitivity.RELATIVE)
    abstract val moduleScripts: ConfigurableFileCollection
    @get:Input abstract val kotlinMultiplatformModules: SetProperty<String>
    @get:Input abstract val kotlinJvmModules: SetProperty<String>

    @TaskAction
    fun verify() {
        val scripts = moduleScripts.files.associate { it.parentFile.name to it.readText() }
        val problems = bomCoverageProblems(scripts, kotlinMultiplatformModules.get(), kotlinJvmModules.get())
        if (problems.isNotEmpty()) throw GradleException(problems.joinToString("\n"))
    }
}

/** Pure, so Levels 2 and 4a can test it with strings. `scripts`: module directory name -> build script text. */
internal fun bomCoverageProblems(scripts: Map<String, String>, kmp: Set<String>, jvm: Set<String>): List<String> { /* ... */ }
```

`bomCoverageProblems` classifies a script by the convention id it applies (`spartangraphics.kmp-library` or
`spartangraphics.jvm-library`), reports (a) a module on disk with a library convention but absent from the matching list, (b) a listed
module with no directory or the other kind, and (c) a library module whose script applies a `spartangraphics.*-library` id it does
not know, with the message "teach VerifyBomCoverage the new module kind and add its publications to the BOM" (the Android phase
adds such a kind). It reads files through the task's inputs, never another `Project`, and `isolated.rootProject.projectDirectory` is
the Isolated Projects API question 1 already uses.

### 6.3 Convention plugins

Kotlin Multiplatform (`spartangraphics.kmp-library.gradle.kts`, in the existing `named("commonMain")` block of question 2, section 6.3):

```kotlin
kotlin {
    sourceSets {
        named("commonMain") {
            dependencies {
                // Published in the root metadata variant and in every target module: one version for the whole family.
                api(project.dependencies.platform(project(":graphics-bom")))
                // …the kotlin-stdlib floor and kotlin-logging lines from question 2
            }
        }
    }
}
```

JVM (`spartangraphics.jvm-library.gradle.kts`, `java-library` already in its `plugins {}` block for the `api` accessor):

```kotlin
dependencies {
    api(platform(project(":graphics-bom")))
    // …the kotlin-stdlib floor line from question 2
}
```

Keep `":graphics-bom"` in the `SpartanBuild` constants of `build-logic`. Neither convention gives `graphics-bom` any plugin, and
`graphics-bom` applies neither convention, so there is no self-edge. `graphics-desktop` keeps `api(project(":graphics-core"))` in
its own script; it receives the BOM through the convention.

### 6.4 Expected published content (at release `0.1.0`; decoded shape of okio-bom 3.18.2 and arrow-stack 2.2.3, same toolchain family)

`graphics-bom-0.1.0.pom` (packaging `pom`; Gradle adds the `published-with-gradle-metadata` marker comment):

```xml
<groupId>io.github.spartanlaboratories</groupId>
<artifactId>graphics-bom</artifactId>
<version>0.1.0</version>
<packaging>pom</packaging>
<name>SpartanGraphics BOM</name>  <!-- plus description, url, licenses, developers, scm from the publishing convention's pom block -->
<dependencyManagement>
  <dependencies>
    <dependency><groupId>io.github.spartanlaboratories</groupId><artifactId>graphics-core</artifactId><version>0.1.0</version></dependency>
    <dependency><groupId>io.github.spartanlaboratories</groupId><artifactId>graphics-core-jvm</artifactId><version>0.1.0</version></dependency>
    <dependency><groupId>io.github.spartanlaboratories</groupId><artifactId>graphics-core-js</artifactId><version>0.1.0</version></dependency>
    <dependency><groupId>io.github.spartanlaboratories</groupId><artifactId>graphics-desktop</artifactId><version>0.1.0</version></dependency>
  </dependencies>
</dependencyManagement>
```

`graphics-bom-0.1.0.module` (`formatVersion` 1.1; two variants only, the second identical except `org.gradle.usage: java-runtime`):

```json
{
  "formatVersion": "1.1",
  "component": { "group": "io.github.spartanlaboratories", "module": "graphics-bom", "version": "0.1.0",
                 "attributes": { "org.gradle.status": "release" } },
  "createdBy": { "gradle": { "version": "9.7.1" } },
  "variants": [
    { "name": "apiElements",
      "attributes": { "org.gradle.category": "platform", "org.gradle.usage": "java-api" },
      "dependencyConstraints": [
        { "group": "io.github.spartanlaboratories", "module": "graphics-core",     "version": { "requires": "0.1.0" } },
        { "group": "io.github.spartanlaboratories", "module": "graphics-core-jvm", "version": { "requires": "0.1.0" } },
        { "group": "io.github.spartanlaboratories", "module": "graphics-core-js",  "version": { "requires": "0.1.0" } },
        { "group": "io.github.spartanlaboratories", "module": "graphics-desktop",  "version": { "requires": "0.1.0" } }
      ] },
    { "name": "runtimeElements", "attributes": { "org.gradle.category": "platform", "org.gradle.usage": "java-runtime" }, "dependencyConstraints": [ /* same four */ ] }
  ]
}
```

Order within the lists is not significant (Okio's is not alphabetical). Files on Central: `.pom`, `.module`, and a `.asc`, `.md5`, `.sha1` for each; no jar, sources or Javadoc.

Each library module records the BOM. `graphics-desktop-0.1.0.module` (both variants) holds the dependencies
`graphics-core` `{"requires": "0.1.0"}`, `kotlin-stdlib` `{"requires": "2.2.0"}`, and
`{"module": "graphics-bom", "version": {"requires": "0.1.0"}, "attributes": {"org.gradle.category": "platform"}, "endorseStrictVersions": true}`;
its `.pom` names `graphics-core-jvm` (F3) and imports the BOM (`<type>pom</type><scope>import</scope>`). `graphics-core-jvm` and
`graphics-core-js` carry the BOM dependency in their own `.module` (it comes from `commonMain`), and the root `.module` carries it in
`metadataApiElements`.

### 6.5 What each consumer does

- **Gradle JVM app (composite or Central):** depends on `io.github.spartanlaboratories:graphics-desktop` (root coordinates). In a
  composite, `includeBuild("../SpartanGraphics")` substitutes `graphics-desktop`, `graphics-core` and `graphics-bom` by `group:name`
  (question 1, F6). `./gradlew dependencyInsight --dependency graphics-core` shows the BOM as the origin of the selected version when
  a request differs.
- **KMP library (SpartanUI, `jvm` + `js`):** `api("io.github.spartanlaboratories:graphics-core:<version>")` in `commonMain` with an
  explicit version (KT-65577), or a project/composite dependency; the `js` and metadata resolutions reach the BOM through the root
  and `graphics-core-js` metadata (F4, subject to the check in 6.6).
- **Maven:** import the BOM, then name artifacts without versions:

  ```xml
  <dependencyManagement><dependencies><dependency>
    <groupId>io.github.spartanlaboratories</groupId><artifactId>graphics-bom</artifactId>
    <version>0.1.0</version><type>pom</type><scope>import</scope>
  </dependency></dependencies></dependencyManagement>
  <!-- dependencies: graphics-desktop, and graphics-core-jvm when the code uses core directly; neither needs a <version> -->
  ```

### 6.6 What the implementation checks (the wiring was not executed here)

1. **Cycle and configuration cache:** `./gradlew build --configuration-cache` twice (store, then reuse); `./gradlew :graphics-core:dependencies
   --configuration jvmCompileClasspath` and `…jsCompileClasspath` show `graphics-bom` with the constraint edges and no failure.
2. **Platform reaches the `js` resolution (the one open step):** `./gradlew :graphics-core:dependencies --configuration jsCompileClasspath`
   lists `project :graphics-bom`; then `generateMetadataFileForKotlinMultiplatformPublication`,
   `generateMetadataFileForJsPublication` and `generateMetadataFileForJvmPublication` produce `.module` files that each contain
   `"module": "graphics-bom"` with `org.gradle.category: platform`. If the `js` classpath resolution fails to select the platform
   variant, move the one line to `jvmMain` (the coroutines/serialization shape) and record that `js` consumers are unaligned.
3. **Published BOM:** `generatePomFileForMavenPublication` and `generateMetadataFileForMavenPublication` of `:graphics-bom` match 6.4;
   `generatePomFileFor*Publication` of `:graphics-desktop` names `graphics-core-jvm`.
4. **Composite:** the throwaway client from question 1, section 6.10 (3), resolves `graphics-desktop` with `dependencyInsight`
   showing `project :graphics-bom` as the platform and `project :graphics-core` selected by constraint.
5. **Alignment (after a second version exists):** a consumer with `graphics-core:0.2.0` and `graphics-desktop:0.1.0` resolves desktop to
   0.2.0.
6. **Level 3 completeness (question 6):** `./gradlew :graphics-core:tasks --all` lists a `generatePomFileFor<Target>Publication` per
   target; assert that the BOM's `graphics-core-<target>` entries equal them. This covers the case the file scan cannot: a target
   added to the `kmp-library` convention (for example `wasmJs`, which publishes `-wasm-js`) without a BOM entry.

The "add a module" step for `CONTRIBUTING.md`: create the folder; add `include(...)` to `settings.gradle.kts`; apply one library
convention in the module script; add the module to `kmpModules` or `jvmModules` in `graphics-bom/build.gradle.kts`; add
`dokka(project(":<module>"))` in the root build; commit the empty ABI dumps. `verifyBomCoverage` enforces the BOM step.

## 7. Load-bearing claims

Quotes omit markup characters only (AsciiDoc links, backticks inside code). "Read" is 2026-10-20.

1. **Gradle's documented alignment shape: platform with project constraints, each module `api(platform(project(...)))`.** "Each module
   should declare a **dependency on the platform**:" and the sample's comments "// The platform declares constraints on all components
   that require alignment" (`api(project(":core"))`, `api(project(":lib"))`, `api(project(":utils"))`) and "// Each project has a
   dependency on the platform" (`api(platform(project(":platform")))`).
   [how_to_align_dependency_versions.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/how-to/dependency-management/how_to_align_dependency_versions.adoc)
   and its [sample build](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/snippets/how-to/dependency-management/dependency-management/dependency-management/managingTransitiveDependencies-dependencyAlignmentWithPlatform/kotlin/platform/build.gradle.kts), Gradle v9.8.2.
2. **That sample is run by Gradle's tests with publishing.** `align-dependency-versions.sample.conf`: "executable: gradle / args:
   publishAllPublicationsToMavenRepository".
   [sample test conf](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/snippets/how-to/dependency-management/dependency-management/dependency-management/managingTransitiveDependencies-dependencyAlignmentWithPlatform/tests/align-dependency-versions.sample.conf), Gradle v9.8.2.
3. **Constraints add no dependency.** "In general, this is what you would like to do: constraints will only apply if such a component is
   added to the dependency graph, either directly or transitively."
   [java_platform_plugin.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/core-plugins/java_platform_plugin.adoc), Gradle v9.8.2
   (same text at v9.7.1 in [`reference/platforms/jvm/java_platform_plugin.adoc`](https://github.com/gradle/gradle/blob/v9.7.1/platforms/documentation/docs/src/docs/userguide/reference/platforms/jvm/java_platform_plugin.adoc)).
4. **Project constraints are published as `group:name:version`; a BOM is generated from them.** "The project notation will become a
   classical `group:name:version` notation in the published metadata." and "This will generate a BOM file for the platform, with a
   `<dependencyManagement>` block where its `<dependencies>` correspond to the constraints defined in the platform module." Same page, v9.8.2 and v9.7.1.
5. **A platform cannot carry dependencies unless enabled.** "By default, in order to avoid the common mistake of adding a dependency in a
   platform instead of a constraint, Gradle will fail if you try to do so." Same page, v9.8.2 and v9.7.1.
6. **`java-platform` creates `check`/`build`.** `JavaPlatformPlugin.apply`: `project.getPluginManager().apply(BasePlugin.class);`.
   [JavaPlatformPlugin.java](https://github.com/gradle/gradle/blob/v9.8.2/platforms/jvm/java-platform/src/main/java/org/gradle/api/plugins/JavaPlatformPlugin.java), Gradle v9.8.2.
7. **Highest requested version wins.** "Gradle will consider _all_ requested versions, wherever they appear in the dependency graph. By
   default, it will select the _highest_ one out of these versions."
   [dependency_constraints_conflicts.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/dependency-management/basics/dependency_constraints_conflicts.adoc), Gradle v9.8.2.
8. **`enforcedPlatform` is for applications.** "`enforcedPlatform` should generally only be used in applications, not in libraries or other
   components consumed by others." and "Because `enforcedPlatform` converts all BOM constraints to `strictly`, these strict versions
   propagate transitively to every consumer of your project."
   [platforms.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/reference/dependency-management/centralizing-dependencies/platforms.adoc), Gradle v9.8.2.
   At v9.7.1: "When using `enforcedPlatform`, exercise caution if your software component is intended for consumption by others." and
   "This declaration is transitive and affects the dependency graph of your consumers."
   ([v9.7.1](https://github.com/gradle/gradle/blob/v9.7.1/platforms/documentation/docs/src/docs/userguide/reference/dependency-management/centralizing-dependencies/platforms.adoc)).
9. **`platform()` endorses strict versions by default.** "Enables the `endorseStrictVersions` behavior by default, enforcing strict versions
   defined in the platform." platforms.adoc, Gradle v9.8.2 (v9.7.1: "Enables the `endorseStrictVersions` behavior by default,
   enforcing strict versions defined in the platform.").
10. **A project dependency resolves to the top-level publications' coordinates; the Kotlin plugin's target publications are children of
    the root.** `DefaultProjectDependencyPublicationResolver.createCoordinateResolver`: "For all published components, find those that
    are not children of other components. These are the top-level publications."
    [source](https://github.com/gradle/gradle/blob/v9.8.2/platforms/software/dependency-management/src/main/java/org/gradle/api/internal/artifacts/ivyservice/projectmodule/DefaultProjectDependencyPublicationResolver.java), Gradle v9.8.2.
    KGP `Publishing.kt`: "The root publication that references the platform specific publications as its variants" (`createRootPublication`).
    [source](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/mpp/publishing/Publishing.kt), Kotlin v2.4.21.
11. **Publishing a project that depends on an unpublished project becomes an error in Gradle 10.** "Publishing a project that depends on another
    project without a publication will be forbidden and will cause the build to fail."
    [upgrading_version_9.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/releases/upgrading/upgrading_version_9.adoc), section "Deprecation of publishing dependencies on unpublished projects", Gradle v9.8.2.
12. **Root variants of a KMP module redirect and carry no dependencies.** "`dependencies`: optional. When missing the variant is assumed to
    have no dependencies. Must not be present when `available-at` is present." and "`available-at`: optional. Information about where
    the metadata and files of this variant are available."
    [Gradle module metadata specification](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/design/gradle-module-metadata-latest-specification.md), Gradle v9.8.2.
    Published instance: okio 3.18.2 root `.module`, `jvmApiElements-published`: `"available-at": {"url": "../../okio-jvm/3.18.2/okio-jvm-3.18.2.module", … "module": "okio-jvm", "version": "3.18.2"}`.
    [okio-3.18.2.module](https://repo1.maven.org/maven2/com/squareup/okio/okio/3.18.2/okio-3.18.2.module).
13. **`KotlinDependencyHandler` has no `platform` in 2.4; the replacement is `project.dependencies.platform`.** Kotlin 2.4 compatibility
    guide: "Kotlin 2.4.0 removes the following deprecated Kotlin Gradle plugin APIs:" … "`KotlinDependencyHandler.platform()`".
    [compatibility-guide-24.md](https://github.com/JetBrains/kotlin-web-site/blob/master/docs/topics/compatibility-guides/compatibility-guide-24.md), read 2026-10-20.
    KGP v2.3.20 `KotlinDependencyHandler.platform`: `@Deprecated("Scheduled for removal in Kotlin 2.3. Check KT-58759", replaceWith =
    ReplaceWith("project.dependencies.platform(notation)"), level = DeprecationLevel.ERROR)`.
    [source](https://github.com/JetBrains/kotlin/blob/v2.3.20/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/KotlinDependencyHandler.kt).
    KT-58759: "Therefore we decided to deprecate and later remove these delegating functions in favor of the explicit use of
    `project.dependencies.XYZ()`" ([KT-58759](https://youtrack.jetbrains.com/issue/KT-58759)).
    KGP v2.4.21: `interface KotlinDependencyHandler : HasProject` and `HasProject`: `val project: Project`
    ([KotlinDependencyHandler.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/KotlinDependencyHandler.kt),
    [HasProject.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/HasProject.kt)).
14. **Kotlin's own integration test publishes a `commonMain` platform dependency as a Maven `import`.** `PublishingIT.testPomRewriter`:
    `sourceSets.commonMain.dependencies { … api(project.dependencies.platform("org.springframework.boot:spring-boot-dependencies:3.4.0")) …`
    and the expected `dependencyManagementConstraints` entry `MavenModule(groupId = "org.springframework.boot", artifactId =
    "spring-boot-dependencies", version = "3.4.0", scope = "import", type = "pom")`; the same test expects project dependencies in the
    `-jvm` POM as `local-jvm`, `included-jvm`.
    [PublishingIT.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-integration-tests/src/test/kotlin/org/jetbrains/kotlin/gradle/PublishingIT.kt), Kotlin v2.4.21.
15. **A `kotlin-api` / `kotlin-metadata` consumer accepts the platform's `java-api` variant.** `KotlinUsagesCompatibility.execute`:
    "consumerValue?.name == KOTLIN_API && producerValue?.name == JAVA_API -> compatible()" and `KotlinMetadataCompatibility`: "ensure that
    a consumer that requests 'kotlin-metadata' can also consumer 'kotlin-api' artifacts or the 'java-*' ones".
    [KotlinUsages.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/mpp/KotlinUsages.kt), Kotlin v2.4.21.
16. **Published BOM variants carry only `category: platform` and `usage`.** okio-bom 3.18.2 `.module`:
    `"name": "apiElements", "attributes": { "org.gradle.category": "platform", "org.gradle.usage": "java-api" }` and the
    `dependencyConstraints` entries `{ "group": "com.squareup.okio", "module": "okio-js", "version": { "requires": "3.18.2" } }`,
    `okio-jvm`, `okio`.
    [okio-bom-3.18.2.module](https://repo1.maven.org/maven2/com/squareup/okio/okio-bom/3.18.2/okio-bom-3.18.2.module); same shape in
    [arrow-stack-2.2.3.module](https://repo1.maven.org/maven2/io/arrow-kt/arrow-stack/2.2.3/arrow-stack-2.2.3.module) and
    [koin-bom-4.2.2.module](https://repo1.maven.org/maven2/io/insert-koin/koin-bom/4.2.2/koin-bom-4.2.2.module).
17. **Koin documents the BOM in shared KMP dependencies for a project with JS and Wasm targets.** Koin "Kotlin Multiplatform Setup", section
    "Dependencies": `implementation(project.dependencies.platform(libs.koin.bom))`; the page's "Supported targets" section lists JS and Wasm
    (`js`, `wasmJs`) among the targets.
    [insert-koin.io KMP setup](https://insert-koin.io/docs/reference/koin-core/kmp-setup), read 2026-10-20.
18. **Kotlin's own libraries apply the BOM from `jvmMain` and skip klib publications.** kotlinx.coroutines 1.11.0
    `bom-conventions.gradle.kts`: `kotlinExtension.sourceSets.getByName("jvmMain").dependencies { api(project.dependencies.platform(project(":kotlinx-coroutines-bom"))) }`;
    `kotlinx-coroutines-bom/build.gradle.kts`: "if (artifacts.any { it.extension == "klib" }) return@all".
    [bom-conventions](https://github.com/Kotlin/kotlinx.coroutines/blob/1.11.0/buildSrc/src/main/kotlin/bom-conventions.gradle.kts),
    [bom build](https://github.com/Kotlin/kotlinx.coroutines/blob/1.11.0/kotlinx-coroutines-bom/build.gradle.kts), tag 1.11.0.
    kotlinx.serialization: `buildSrc/src/main/kotlin/bom-conventions.gradle.kts` at `v1.9.0` has the same `jvmMain` line.
    [source](https://github.com/Kotlin/kotlinx.serialization/blob/v1.9.0/buildSrc/src/main/kotlin/bom-conventions.gradle.kts).
19. **The published module records the BOM as a platform with endorsement; the POM imports it.** `kotlinx-coroutines-core-jvm-1.11.0.module`:
    `"module": "kotlinx-coroutines-bom", "version": { "requires": "1.11.0" }, "attributes": { "org.gradle.category": "platform" },
    "endorseStrictVersions": true`; `kotlinx-coroutines-core-jvm-1.11.0.pom`: `<artifactId>kotlinx-coroutines-bom</artifactId> …
    <type>pom</type> <scope>import</scope>`.
    [module](https://repo1.maven.org/maven2/org/jetbrains/kotlinx/kotlinx-coroutines-core-jvm/1.11.0/kotlinx-coroutines-core-jvm-1.11.0.module),
    [pom](https://repo1.maven.org/maven2/org/jetbrains/kotlinx/kotlinx-coroutines-core-jvm/1.11.0/kotlinx-coroutines-core-jvm-1.11.0.pom), read 2026-10-20.
20. **A JVM module's `.module` names the KMP root; its `.pom` names `-jvm`.** `kotlinx-coroutines-reactive-1.11.0.module`: `"module":
    "kotlinx-coroutines-core", "version": { "requires": "1.11.0" }`; `kotlinx-coroutines-reactive-1.11.0.pom`: `<artifactId>kotlinx-coroutines-core-jvm</artifactId>
    <version>1.11.0</version> <scope>compile</scope>`.
    [module](https://repo1.maven.org/maven2/org/jetbrains/kotlinx/kotlinx-coroutines-reactive/1.11.0/kotlinx-coroutines-reactive-1.11.0.module),
    [pom](https://repo1.maven.org/maven2/org/jetbrains/kotlinx/kotlinx-coroutines-reactive/1.11.0/kotlinx-coroutines-reactive-1.11.0.pom), read 2026-10-20.
21. **A project constraint on a KMP module yields the root only; comparable BOMs add targets by hand or by iterating targets.** Koin
    `projects/bom/koin-bom/build.gradle.kts` at 4.2.2: `api(project(":core:koin-core"))` and `api("io.insert-koin:koin-core-jvm:$version") //Check later KMP Bom`;
    Okio `build-support/src/main/kotlin/bom.kt` at `parent-3.18.2`: "Collect all the root project's multiplatform targets and add them to
    the BOM."
    [Koin](https://github.com/InsertKoinIO/koin/blob/4.2.2/projects/bom/koin-bom/build.gradle.kts),
    [Okio](https://github.com/square/okio/blob/parent-3.18.2/build-support/src/main/kotlin/bom.kt).
22. **Generating from all subprojects over-includes (OkHttp).** `okhttp-bom/build.gradle.kts` at `parent-5.5.0`:
    `project.rootProject.subprojects.forEach { subproject -> if (subproject.name != "okhttp-bom") { api(subproject) } }`;
    `okhttp-bom-5.5.0.pom`: `<groupId>okhttp-parent</groupId> <artifactId>samples</artifactId> <version>unspecified</version>`.
    [build script](https://github.com/square/okhttp/blob/parent-5.5.0/okhttp-bom/build.gradle.kts),
    [pom](https://repo1.maven.org/maven2/com/squareup/okhttp3/okhttp-bom/5.5.0/okhttp-bom-5.5.0.pom), read 2026-10-20.
23. **A hand list omits a published target (Arrow).** `arrow-stack-2.2.3.module` has 45 constraints, none ending `-js`; the directory
    `https://repo1.maven.org/maven2/io/arrow-kt/arrow-core-js/2.2.3/` exists with a `.module`.
    [arrow-stack](https://repo1.maven.org/maven2/io/arrow-kt/arrow-stack/2.2.3/), [arrow-core-js](https://repo1.maven.org/maven2/io/arrow-kt/arrow-core-js/2.2.3/), read 2026-10-20;
    build script [arrow-libs/stack/build.gradle.kts](https://github.com/arrow-kt/arrow/blob/2.2.3/arrow-libs/stack/build.gradle.kts).
24. **vanniktech `JavaPlatform` source at 0.37.0.** "public class JavaPlatform : Platform() {" "override val javadocJar: JavadocJar =
    JavadocJar.None()" "override val sourcesJar: SourcesJar = SourcesJar.None()" "check(project.plugins.hasPlugin("java-platform"))"
    "it.from(project.components.getByName("javaPlatform"))".
    [Platform.kt](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/plugin/src/main/kotlin/com/vanniktech/maven/publish/Platform.kt), vanniktech 0.37.0.
    Docs: "For projects using the `java-platform` plugin." / `configure(JavaPlatform())`
    ([docs/what.md](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/docs/what.md)).
25. **Central accepts pom-packaged BOMs without Javadoc and sources jars; precedents exist through vanniktech.** Sonatype: "Projects with
    packaging other than `pom` have to supply JAR files that contain Javadoc and sources."
    [Central requirements](https://central.sonatype.org/publish/requirements/), read 2026-10-20. The `okio-bom` 3.18.2 directory lists only
    `okio-bom-3.18.2.module`, `.module.asc`, `.module.md5`, `.module.sha1`, `.pom`, `.pom.asc`, `.pom.md5`, `.pom.sha1`
    ([listing](https://repo1.maven.org/maven2/com/squareup/okio/okio-bom/3.18.2/)), and Okio's catalog pins
    `vanniktech-publish-plugin = { module = "com.vanniktech:gradle-maven-publish-plugin", version = "0.37.0" }`
    ([libs.versions.toml](https://github.com/square/okio/blob/parent-3.18.2/gradle/libs.versions.toml)).
26. **Maven consumers import the BOM themselves.** "Other projects that wish to use the library should import this POM into the
    dependencyManagement section of their POM." and the `import` scope "is only supported on a dependency of type `pom` in the
    `<dependencyManagement>` section".
    [Maven dependency mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html), read 2026-10-20.
27. **Why a family BOM exists: the runtime failure.** kotlinx.coroutines issue #2865: "To prevent such situation, both modules should depend
    on `kotlinx-coroutines-bom` of the corresponding version".
    [issue #2865](https://github.com/Kotlin/kotlinx.coroutines/issues/2865), read 2026-10-20.
28. **Per-target coordinates are not substituted by composite builds.** KT-56161, state *As Designed*: "Just including the other
    multiplatform build will only work when a consumer build references the dependency using the same module as the producer build (the
    mpp dependency, without the -platform classifier)."
    [KT-56161](https://youtrack.jetbrains.com/issue/KT-56161), read 2026-10-20.
29. **Unversioned BOM-managed dependencies in a published KMP module (opinion, one report).** KT-65577: "if the dependency is included to
    the `commonMain` source-set without declaring it with a version (IE: the version comes from a `BOM`) the generated POM does not
    include the platform specific artifact, neither the version"; State *Incomplete*.
    [KT-65577](https://youtrack.jetbrains.com/issue/KT-65577), read 2026-10-20.
30. **Kotlin's workaround for BOMs in source sets.** KT-40489 comment by h0tk3y: "Create the dependency using the Gradle's
    `project.dependencies` and add it with the Kotlin source sets DSL".
    [KT-40489](https://youtrack.jetbrains.com/issue/KT-40489), read 2026-10-20.
31. **Jackson records its BOM per module.** `jackson-databind-2.22.4.module`: `"module": "jackson-bom", "version": { "requires": "2.22.4" },
    "attributes": { "org.gradle.category": "platform" }, "endorseStrictVersions": true` in `apiElements` and `runtimeElements`.
    [module](https://repo1.maven.org/maven2/com/fasterxml/jackson/core/jackson-databind/2.22.4/jackson-databind-2.22.4.module), read 2026-10-20.

## 8. Known gaps

- **Nothing was executed.** No Gradle build or task ran. The following statements are inferences from documentation, source and
  published metadata and are settled by the checks in section 6.6: (a) the platform dependency declared in `commonMain` appears in
  the root `metadataApiElements` variant and in `graphics-core-js`'s `.module` (mechanism and Kotlin's POM test support it; no
  published KMP library with a `js` target and a `commonMain` platform dependency was found); (b) a `js` compile classpath selects the
  project BOM's `java-api` variant (compatibility rules and Koin's documentation support it); (c) a Gradle constraint on the root
  `graphics-core` governs the target module reached through `available-at` (the specification fixes the redirect's version; no
  Gradle test of the conflict case was found); (d) the configuration cache and Isolated Projects accept the constraint and platform
  edges (they are plain project dependencies, but no run was made).
- **Why Kotlin's own libraries use `jvmMain` only** was not found: PR #2952 and issue #2865 discuss only the JVM failure. If check
  6.6 (2) fails, the fallback is a one-line move to `jvmMain`, which leaves `js` consumers unaligned (the target modules would carry
  no BOM dependency), and that outcome would be recorded.
- **A platform project in a composite build:** no Gradle integration test of a substituted `java-platform` project was found among the
  composite-build tests (searched by name and by `java-platform`); substitution is by `group:name` for any project (question 1, F6). The
  composite smoke test in 6.6 (4) is the evidence.
- **Maven's treatment of a dependency's own `<dependencyManagement>`** (not applied to the consumer) is stated from general Maven
  behaviour; the quoted Maven page supports the import-yourself instruction, not the negative.
- **Central Portal acceptance of this BOM with vanniktech 0.37.0 on Gradle 9.7.1/9.8.2, and how vanniktech groups several projects into
  deployments at G7,** were not tested: precedents are `okio-bom` 3.18.2 (0.37.0) and `arrow-stack` 2.2.3 (0.36.0).
- **Gradle 9.7.1 versus 9.8.2:** nothing in the BOM mechanics differs in the documentation read at both tags (the `enforcedPlatform` warning
  is worded more strongly at 9.8.2). Gradle 9.8's deprecation of signing with a classifier (question 2, F2) may touch the BOM's signing; not measured.
- **Jackson's BOM** was read only for the per-module dependency; Ktor's module dependencies were checked on three modules
  (`ktor-http`, `-jvm`, `-js`), Okio, Arrow, Koin, Coil and Wire on root, `-jvm` and `-js` of one module each.
- **Typed project accessors** (`projects.graphicsCore`) were not evaluated; the string form is Gradle's own sample.
- **New targets and kinds:** a `wasmJs` target adds `-wasm-js`; an Android or web module kind adds a new convention id and BOM list;
  both are covered by the check's failure message and the Level 3 comparison in 6.6 (6), neither of which exists yet.
- **Nothing in earlier findings is contradicted.** Question 1's section 6.9 sketch applies Dokka in the publishing convention's
  `plugins {}` block, which question 4 section 6.3 replaces for the BOM's sake (section 5 above); question 1 F6's statement that
  per-target coordinates are never substituted is confirmed by KT-56161.
