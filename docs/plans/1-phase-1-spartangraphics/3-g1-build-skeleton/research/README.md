# Research: G1 build skeleton

- **Issue:** [SpartanLaboratories/SpartanGraphics#3](https://github.com/SpartanLaboratories/SpartanGraphics/issues/3),
  "G1: Build skeleton", the first stage of Phase 1
  ([#1](https://github.com/SpartanLaboratories/SpartanGraphics/issues/1)).
- **Baseline:** `main` @ `765bf15`. The repository holds a one-line `README.md`, the Apache-2.0
  `LICENSE`, a `.gitignore`, and under `docs/` the roadmap, its research and the split plan. No
  source, test or build file exists.
- **Citations:** documents by section, code by symbol; outside repositories at a pinned tag or
  commit.

This file holds the research questions behind G1's design, then the verification of the
outside-world claims their answers rest on, then a topic-by-topic summary. Each question's
findings live in the file it names.

---

## What G1 builds

**Modules**

- `graphics-core`: Kotlin Multiplatform with the targets `jvm()` and `js { nodejs() }` and the
  default hierarchy template. There is no `browser()` and no `wasmJs`. It holds no Kotlin source
  in G1.
- `graphics-desktop`: a Kotlin/JVM library that takes `api(graphics-core)`. It holds no Kotlin
  source in G1 and no LWJGL dependency yet.
- `graphics-bom`: a BOM whose constraints keep every artifact this repository publishes on one
  version. Every library module takes `api(platform(graphics-bom))`. It is published with the
  rest at G7.
- Folder name, Gradle project name and artifactId agree, under the group
  `io.github.spartanlaboratories`, so that a consumer's `includeBuild("../SpartanGraphics")`
  substitutes the modules without substitution rules. Packages are `com.spartanlabs.*`.
- No LWJGL, GeneralTools, kotlinx.coroutines or `graphics-gl` enters the build in G1.

**Build**

- The Gradle wrapper 9.8.2, `-bin` distribution, with its `distributionSha256Sum`.
  - 9.8.1 is the first public release that patches three high-severity Gradle advisories covering
    9.0.0 to 9.8.0. Fixed 9.7.x and older-line releases exist only through Gradle's paid Security
    Subscription.
  - 9.8.2 is one minor past the Kotlin Gradle plugin 2.4.21's "fully supported" range, which
    permits newer Gradle.
- A version catalog, `gradle/libs.versions.toml`.
- A private `build-logic` included build with convention plugins for a KMP library module, a JVM
  library module and publishing.
- The Kotlin Gradle plugin 2.4.21, the latest stable release.
- The consumer floor: `languageVersion` and `apiVersion` 2.2, and a JVM and common `kotlin-stdlib`
  requirement of 2.2.0. A JVM consumer on Kotlin 2.1.20 or later can therefore compile against the
  libraries. This is the same floor as GeneralTools 3.x, whose types `graphics-core` exposes from
  G2. The web klibs follow the compiler, so a `js` consumer needs Kotlin 2.4.
- A JDK 21 toolchain and JVM 21 bytecode.
- Strict `explicitApi()` and the Kotlin Gradle plugin's ABI validation (JVM and klib dumps) on
  every library module from the first commit. The dumps are committed even while they are empty.
- One shared version, `0.1.0-SNAPSHOT`, kept in one place for every artifact. The release branch
  of G7 sets `0.1.0`.
- Logging through kotlin-logging, which logs through slf4j on the JVM.

**Tests**

- **Level 1 is a process, not a test package.**
  - It is the pre-commit run plus each plan's manual checks, with the outcomes recorded in the
    pull request's description.
  - Gating tests are temporary by definition ("active until the WIP is finished"), so there is no
    `testing.gating` folder.
- **Every persistent test** sits in `com.spartanlabs.testing.<level>.<production package>`, one
  level per package segment: `component`, `integration`, `deterministic`, `e2e`, `nonfunctional`,
  `uat`.
- **Level 5 (`uat`) is used selectively,** for systems a downstream developer using the library
  sees directly: public authoring APIs, installable systems and lookup `Result`s. Internal
  mechanics get no Level 5.
- **`-PtestLevel=<level>[,<level>]`** takes those six names and adds the package filter to every
  test task. With the property unset, `check` runs every level on every target.
- `kotlin.test` in common code. JUnit 5 and MockK appear only in JVM-only tests, and logback is
  the JVM test logging backend.

**Documentation and publishing**

- Dokka builds each published module's javadoc jar through the publishing convention. The root
  project aggregates one site, and CI builds it. There is no GitHub Pages site.
- The root build loads the Kotlin Gradle plugin where the root's Dokka can see it. MyGameTools
  shows the failure otherwise: when the Kotlin plugin lives only in per-module classloaders, Dokka
  cannot load `KotlinBasePlugin` and each module renders an empty site (MyGameTools root
  `build.gradle.kts`, the comment on its `plugins` block; `docs/plans/40-umbrella-empty-sources-jar/plan.md`
  §1.2 there).
- Publishing uses `com.vanniktech.maven.publish` to the Central Portal with signing, in the shape
  of GeneralTools' `mavenPublishing` block. A maintainer releases by hand at G7, and nothing
  publishes from CI.

**CI and the repository**

- GitHub Actions on JDK 21:
  - `windows-latest` runs the full build and every test that opens no GL context;
  - `ubuntu-latest` compiles;
  - `macos-latest` compiles.

  GL tests on Linux arrive with the renderer in a later stage.
- A release workflow creates the GitHub Release from a pushed `v`-prefixed tag and publishes
  nothing.
- `main` is protected, administrators included: changes arrive by pull request, and G1's CI checks
  must pass before a merge.
- Beside the protection, the repository settings are aligned:
  - merge commits only, with head branches deleted on merge;
  - merge-commit text taken from the pull request's title and body;
  - a read-only default workflow token.

  Applying the protection and these settings is a GitHub settings step, with the checks and
  settings named exactly. It is applied only once the maintainer confirms it.
- The workflows pin actions by major tag.
- Repository files:
  - `.gitignore`, whose current `*.jar` rule would leave the Gradle wrapper jar uncommitted;
  - `.gitattributes`;
  - `.editorconfig`;
  - `CONTRIBUTING.md`, which supplements the organisation's baseline with where the version lives,
    which artifacts share it, and the build, test and publish commands;
  - `CHANGELOG.md` with an `[Unreleased]` section;
  - a README per module, and the root README.
- The coding rules in `.aiassistant/rules/` (`CLAUDE.md` and `Manifesto.md`) are MyGameTools'
  rules, with three changes:
  - the version-nomenclature rule names the publishing group `io.github.spartanlaboratories`;
  - its duplicated section number is corrected;
  - the logging rule names kotlin-logging (slf4j on the JVM), and the testing rule names
    `kotlin.test` in common code, JUnit 5 and MockK only in JVM-only tests, the level packages
    and the package filter.

---

## Sources reused

These findings answer parts of G1's questions and are cited by path rather than researched again.
Each was written in October 2026 against the same Kotlin Gradle plugin (2.4.21) and vanniktech
plugin (0.37.0) that G1 uses, and nothing newer contradicts them.

**GeneralTools' issue #7 research**, on the `feat/issue-7-kotlin-multiplatform` branch at
`d424aaf` (`docs/plans/7-kotlin-multiplatform/research/` in
[SpartanLaboratories/GeneralTools](https://github.com/SpartanLaboratories/GeneralTools/tree/d424aaf/docs/plans/7-kotlin-multiplatform/research)),
whose verification re-opened every load-bearing claim on 2026-10-18 and 2026-10-19:

- [`kmp-build-and-kotlin-compat.md`](https://github.com/SpartanLaboratories/GeneralTools/blob/d424aaf/docs/plans/7-kotlin-multiplatform/research/kmp-build-and-kotlin-compat.md)
  answers the floor mechanics for one KMP module:
  - a compiler reads class metadata one minor version ahead of itself, and the metadata follows
    `languageVersion`;
  - the stdlib pin is split by target, because the JS compiler rejects an older stdlib klib;
  - `jvmToolchain` with an explicit `jvmTarget`, and the `TargetJvmVersion` attribute that a KMP
    `jvm` publication otherwise omits;
  - the `js { nodejs() }` test runtime and the committed `kotlin-js-store/yarn.lock`;
  - the GitHub Actions versions current on 2026-10-15.
- [`kmp-maven-central-publishing.md`](https://github.com/SpartanLaboratories/GeneralTools/blob/d424aaf/docs/plans/7-kotlin-multiplatform/research/kmp-maven-central-publishing.md)
  answers KMP publishing with vanniktech 0.37.0: the artifact names (`-jvm`, `-js`), the signing
  properties and the staged release through the Central Portal.
- [`kmp-test-levels-and-names.md`](https://github.com/SpartanLaboratories/GeneralTools/blob/d424aaf/docs/plans/7-kotlin-multiplatform/research/kmp-test-levels-and-names.md)
  answers test-level selection:
  - one `-PtestLevel` property adds the package filter to every `AbstractTestTask`, `jvmTest` and
    the JS test task alike;
  - `isFailOnNoMatchingTests = false` is needed on the JVM task;
  - per-level test tasks do not work across KMP targets.
- [`kotlin-logging-adoption.md`](https://github.com/SpartanLaboratories/GeneralTools/blob/d424aaf/docs/plans/7-kotlin-multiplatform/research/kotlin-logging-adoption.md)
  answers the logging dependency:
  - kotlin-logging 8.0.4 as an `implementation` dependency of `commonMain`;
  - `slf4j-api` supplied on the JVM;
  - explicit logger names;
  - the startup line that 8.x prints.
- [`idiomatic-kotlin-api-mechanics.md`](https://github.com/SpartanLaboratories/GeneralTools/blob/d424aaf/docs/plans/7-kotlin-multiplatform/research/idiomatic-kotlin-api-mechanics.md)
  answers `explicitApi()` and the Kotlin Gradle plugin's `abiValidation()` in a single KMP module,
  and records that the plugin's dumper and the binary-compatibility-validator list different
  members.
- That branch's `plan-kmp-build.md` and `final-implementation.md` record observed behaviour on
  Gradle 9.0.0:
  - a target with no source has a 0-byte klib dump, which `checkKotlinAbi` still needs committed;
  - `checkKotlinAbi` and `updateKotlinAbi` fail when run in one invocation;
  - the pinned compiler arguments (`-api-version 2.2 -language-version 2.2`) show only under
    `--debug`.

**This repository's roadmap research**, in `docs/roadmap/research/`:

- `web-target-kotlin-js-vs-wasm.md` §6, on the web compilation target, answers the non-JVM target: `js()` with the default
  hierarchy template, `nodejs()` for `kotlin.test` tests of common code, no `browser()`, and klib
  compatibility that follows the compiler.
- `android-build-and-platform-floors.md` §6 records that AGP 9.4 needs Gradle 9.6.0 or newer,
  which matters when the Android phase adds AGP to this build.
- `render-performance-and-verification.md` F15 and F16 record that `windows-latest` has no usable
  GL driver and that Linux runners can host GL tests on llvmpipe later.
- `backend-contract-design.md` and `idiomatic-kotlin.md` record that `-jvm-default=enable` is the
  default from Kotlin 2.2 and that `explicitApi()` and ABI validation belong in the build from G1.

---

## Questions

The questions are researched in this order, each researcher reading the findings that precede
it. Three lenses apply inside every question:

- the best systems for the purpose;
- common practice and the conventions of this repository's sibling builds;
- idiomatic Kotlin, here mostly the Gradle Kotlin DSL and Kotlin code in `build-logic`.

### 1. How should `build-logic` and the version catalog be structured, and how must the build load its plugins so that the root's Dokka aggregate and consumers' composite builds both work?

- **Options.**
  - Precompiled script plugins (`*.gradle.kts` under `build-logic/src/main/kotlin`), or binary
    plugins (classes registered through `gradlePlugin {}`).
  - How a convention plugin reads the catalog:
    - by name through `VersionCatalogsExtension`;
    - through the generated-accessor workaround on `build-logic`'s classpath;
    - through whatever Gradle 9 itself offers.
  - How plugin artifacts reach `build-logic`'s classpath (plugin-marker coordinates declared from
    the catalog), and how the root build puts the Kotlin, Dokka and vanniktech plugins in one
    classloader (root `plugins { ... apply false }`, as MyGameTools' root build does).
  - `includeBuild("build-logic")` inside `pluginManagement {}` or at the top level of
    `settings.gradle.kts`.
  - Convention plugin ids and their prefix.
  - Where the shared values live: group, version, the Kotlin floor and the JVM level, as catalog
    `[versions]`, `gradle.properties` entries or convention-plugin constants.
- **Requirements.**
  - Each library module applies one convention plugin and declares little else.
  - Every version comes from the catalog, and the catalog holds only what G1 uses.
  - One shared version lives in one place.
  - The root's Dokka aggregate renders every module's API, so the root sees the same Kotlin
    plugin classes as the modules.
  - `group` is set on every project, so that `includeBuild("../SpartanGraphics")` from a JVM app
    on Kotlin 2.1.20 (the game client) and from a KMP library (SpartanUI) substitutes the modules,
    with `build-logic` nested inside this build.
  - The configuration cache works, and nothing blocks Gradle's isolated projects later.
  - The build reads as a model for SpartanUI's own.
- **Evidence that settles it.**
  - Gradle's documentation on sharing build logic, version catalogs in included builds,
    composite and plugin builds, and the configuration cache.
  - Gradle issues on catalog accessors in precompiled script plugins.
  - The `build-logic` of mature KMP projects at pinned tags (for example Ktor, Coil, Okio,
    kotlinx-io, SQLDelight, Arrow, Now in Android).
  - Dokka's documentation and issues on the classloader failure.
  - MyGameTools' `build-logic`, root build and `docs/plans/40-umbrella-empty-sources-jar/plan.md`.
- **Findings:** [`build-logic-and-version-catalog.md`](build-logic-and-version-catalog.md).

### 2. Which versions go into the catalog, which Gradle 9.x wrapper is the newest inside every plugin's documented range, and how do the Kotlin 2.2 floor and the JVM 21 level apply to both the KMP module and the JVM-only module?

- **Options.**
  - The wrapper: Gradle 9.7.x or 9.8.x, against the documented ranges of the Kotlin Gradle plugin
    2.4.21, vanniktech 0.37.0, Dokka 2.2.0, the foojay toolchain resolver and any other plugin the
    build applies.
  - The stdlib pin on a `kotlin("jvm")` module: an explicit `kotlin-stdlib:2.2.0` dependency, or
    `coreLibrariesVersion`.
  - `jvmToolchain(21)` with `jvmTarget`, or `-Xjdk-release=21`.
  - Hand-written pins in the convention plugins, or the `com.gradleup.tapmoc` plugin with its
    dependency check.
  - JUnit Jupiter 5.x or 6.x beside `kotlin-test`'s JUnit integration; the current MockK,
    logback, `slf4j-api` and kotlin-logging releases.
- **Requirements.**
  - The Kotlin Gradle plugin is 2.4.21.
  - The floor is 2.2 for `languageVersion` and `apiVersion`, with a JVM and common stdlib
    requirement of 2.2.0. The floor must hold in what each published variant declares, not only
    in how the code compiles.
  - A JDK 21 toolchain and JVM 21 bytecode, and a resolution-time failure for a consumer on an
    older JVM.
  - The wrapper is the newest 9.x inside every applied plugin's documented range. Whether that
    wrapper also satisfies AGP 9.4's floor of 9.6.0, which the Android phase needs, is recorded.
  - Each library is at its latest stable release that keeps the floor.
- **Evidence that settles it.**
  - Gradle's compatibility matrix and release notes for 9.7 and 9.8.
  - The Kotlin documentation's Gradle range for 2.4.20–2.4.21.
  - vanniktech's and Dokka's changelogs and compatibility statements, and foojay's releases.
  - Maven Central metadata for each library.
  - `kotlin-test`'s documentation and source for its JUnit 5 integration.
  - [`kmp-build-and-kotlin-compat.md`](https://github.com/SpartanLaboratories/GeneralTools/blob/d424aaf/docs/plans/7-kotlin-multiplatform/research/kmp-build-and-kotlin-compat.md)
    for the KMP mechanics.
- **Findings:** [`toolchain-versions-and-pins.md`](toolchain-versions-and-pins.md).

### 3. How do strict `explicitApi()` and the Kotlin Gradle plugin's ABI validation behave across a KMP module, a `kotlin("jvm")` module and modules with no Kotlin source, in one multi-module build?

- **Options.**
  - The plugin's `abiValidation` DSL as it stands in 2.4.21, for KMP and for JVM projects, or the
    binary-compatibility-validator plugin.
  - The dump location: each module's own `api/` directory, or a central tree.
  - Applying `explicitApi()` and `abiValidation()` in the convention plugins, or per module.
  - A module with no source: empty dump files committed, or validation switched off until the
    first declaration.
  - Line endings for the dump files.
- **Requirements.**
  - Both library modules have strict explicit API and dump checks from the first commit: JVM
    dumps for both, a klib dump for `graphics-core`.
  - The dumps are committed even while empty.
  - `check`, and so `build`, fails on any public-API change that the committed dumps do not
    show, on every operating system, including Windows checkouts with `core.autocrlf`.
  - A pull request shows its public-API change as a diff of the committed dumps.
  - The BOM has no Kotlin code and no dump.
  - Everything works under the configuration cache.
- **Evidence that settles it.**
  - The Kotlin Gradle plugin's documentation and source at `v2.4.21` (the `abiValidation`
    extensions for KMP and JVM, task wiring, empty-input handling), and the release notes 2.2 to
    2.4.
  - Kotlin issues on ABI validation.
  - Libraries running the plugin's ABI validation in multi-module builds.
  - [`idiomatic-kotlin-api-mechanics.md`](https://github.com/SpartanLaboratories/GeneralTools/blob/d424aaf/docs/plans/7-kotlin-multiplatform/research/idiomatic-kotlin-api-mechanics.md)
    and GeneralTools' `plan-kmp-build.md` for the single-module behaviour.
- **Findings:** [`abi-validation-across-modules.md`](abi-validation-across-modules.md).

### 4. How does Dokka 2 build each module's javadoc jar through vanniktech and one aggregated site at the root, for a KMP module and a JVM module that hold no source yet?

- **Options.**
  - The javadoc jar's content: Dokka's HTML publication or its javadoc format, as vanniktech
    0.37.0 wires each through `JavadocJar.Dokka(...)`.
  - Whether Dokka 2.2.0 still needs the plugin-mode flags in `gradle.properties`.
  - Applying Dokka in the publishing convention, or in a documentation convention of its own.
  - Root aggregation through `dependencies { dokka(project(...)) }`, with each module's source
    links.
  - What Dokka does with empty source sets, and how a build catches a module that renders an
    empty site. MyGameTools guards its umbrella javadoc jar with a verification task
    (`verifyUmbrellaJavadocJar`).
  - Configuration-cache support.
- **Requirements.**
  - Each published library module's javadoc jar holds Dokka output.
  - The root aggregates every library module into one site, and CI builds it. There is no Pages
    site.
  - The BOM has no documentation of its own.
  - A module with no source does not fail the build.
  - The classloader arrangement from question 1 holds from Dokka's side.
  - Everything works under the configuration cache.
- **Evidence that settles it.**
  - Dokka 2.2.0's documentation, migration guide and source, and its issues on `KotlinBasePlugin`
    and classloaders.
  - vanniktech 0.37.0's documentation and source for Dokka javadoc jars.
  - KMP libraries that pair Dokka 2 with vanniktech, at pinned tags.
  - MyGameTools' root build, its `gametools.published-library` convention and its #40 plan.
- **Findings:** [`dokka-multi-module.md`](dokka-multi-module.md).

### 5. How is `graphics-bom` built and published, and how does each module depend on it, so that a consumer's graphics artifacts always resolve to one version, including the per-target artifacts of a KMP module?

- **Options.**
  - A `java-platform` project with its constraints written by hand, or generated from every
    published project.
  - Constraints on the root coordinates only (`graphics-core`), or also on each target
    publication (`graphics-core-jvm`, `graphics-core-js`).
  - `api(platform(project(":graphics-bom")))` in KMP `commonMain`, or in each target's source
    set.
  - Publishing through vanniktech's `JavaPlatform()`.
  - Keeping the BOM out of Dokka and ABI validation.
- **Requirements.**
  - Every published module depends on the BOM through `api(platform(...))` from its first commit.
  - The BOM constrains only this repository's artifacts.
  - Gradle consumers (a JVM app, a KMP library, a composite build) resolve aligned versions, and
    a mixed-version family cannot resolve silently. That mix is what fails at runtime with
    `AbstractMethodError`.
  - A Maven consumer can import the BOM.
  - The BOM carries the shared version, and it is published with the rest at G7.
  - No dependency cycle breaks configuration.
- **Evidence that settles it.**
  - Gradle's documentation on platforms and on aligning versions with a published platform.
  - The Kotlin documentation on platform dependencies in multiplatform source sets.
  - Published BOMs and Gradle module metadata of KMP libraries (for example kotlinx-coroutines,
    Ktor, Arrow, Koin, OkHttp and Okio), and Jackson's per-module platform dependency, decoded
    read-only.
  - vanniktech's documentation and source for `JavaPlatform`.
- **Findings:** [`kmp-bom-alignment.md`](kmp-bom-alignment.md).

### 6. How does a build with no product code test its own build logic at the right levels of the five-level hierarchy, and at what cost in CI?

The build logic to cover: the convention plugins' explicit API, ABI checks, test-level filter, BOM
alignment, Kotlin floor, JVM level and Dokka output.

- **Options.**
  - Gradle TestKit functional tests in `build-logic` (Level 3), against fixture projects or the
    real build.
  - Verification tasks in the real build, in the style of MyGameTools' umbrella-jar checks.
  - Unit tests of pure build-logic functions such as the `-PtestLevel` parser (Levels 2 and 4a).
  - A throwaway consumer project, or named manual Level 1 checks only.
  - How TestKit runs with an included `build-logic`, the version catalog and a KMP `js` target,
    whose Node.js download lands in TestKit's isolated Gradle home.
  - Where such tests run in CI.
- **Requirements.**
  - Tests follow the five-level hierarchy, each level in its own package.
  - G1 has no Kotlin product source.
  - Every behaviour the convention plugins promise is checked, either automatically or by a named
    manual step.
  - CI time stays proportionate to a skeleton.
  - Heavy tests stay off the modules' default `check` unless they are cheap. MyGameTools keeps
    its TestKit tests off `build`.
- **Evidence that settles it.**
  - Gradle's documentation on testing build logic with TestKit.
  - How mature builds test their conventions: MyGameTools' `build-logic` tests, Gradle's samples,
    the builds of Ktor, Coil and Square's libraries, and Gradle-plugin projects' functional tests
    such as vanniktech's own.
  - TestKit issues with KMP and included builds.
  - CI timing data where it exists.
- **Findings:** [`testing-build-logic.md`](testing-build-logic.md).

### 7. How should the CI workflow, the release workflow and the protection of `main` be set up, so that every pull request is gated by the three-OS build, the Dokka aggregate and the ABI checks, under exactly named required checks?

- **Options.**
  - One workflow with three jobs, or a matrix.
  - Triggers (`pull_request` and `push` to `main`, `workflow_dispatch`) and `concurrency`.
  - Where the Dokka aggregate builds.
  - What "compiles" covers on Ubuntu and macOS: main sources only, or test sources too.
  - Caching: `gradle/actions/setup-gradle`, and Node.js and Yarn in the Gradle user home.
  - Wrapper validation.
  - The release workflow's steps: verifying the tagged build, then `gh release create`.
  - Classic branch protection or a repository ruleset.
  - Required-check names for plain and matrix jobs, and "require branches to be up to date".
  - Whether the merge-method settings belong to the same step: the organisation's baseline
    allows merge commits only.
  - Applying all of it through `gh api`.
- **Requirements.**
  - `windows-latest` runs the full build and every non-GL test.
  - `ubuntu-latest` and `macos-latest` compile.
  - Every job runs on Temurin JDK 21.
  - CI builds the Dokka aggregate.
  - The release workflow creates the GitHub Release from a `v`-prefixed annotated tag and
    publishes nothing.
  - No registry credential lives in CI.
  - `main` accepts changes only through pull requests whose CI checks pass. The step names the
    exact checks and settings and is applied only once the maintainer confirms it.
  - The organisation's baseline governs branching, merge commits and releasing.
  - The repository is public.
- **Evidence that settles it.**
  - GitHub's documentation on Actions, required status checks and their names, rulesets and
    branch protection, and the REST API for each.
  - Release notes of `actions/checkout`, `actions/setup-java` and `gradle/actions`.
  - The runner images' documentation for `windows-latest`, `ubuntu-latest` and `macos-latest`.
  - MyGameTools' and GeneralTools' workflows.
  - The organisation's `CONTRIBUTING.md` in
    [SpartanLaboratories/.github](https://github.com/SpartanLaboratories/.github).
- **Findings:** [`ci-release-and-branch-protection.md`](ci-release-and-branch-protection.md).

---

## Verification results

Every numbered load-bearing claim of the seven findings files was re-opened at its source on
2026-10-20, at the pinned tag, commit or version.

- **Tier A, checked in full:** the claims a recommendation rests on, including currency.
  Measurements were taken again from the published artifacts: class-metadata scans, jar listings,
  distribution checksums and CI run timestamps.
- **Tier B, checked for existence:** the supporting precedents.

| Findings file | Tier A: hold / fail / unclear | Tier B: hold / fail / unclear |
| --- | --- | --- |
| [`build-logic-and-version-catalog.md`](build-logic-and-version-catalog.md) | 19 / 0 / 0 | 7 / 0 / 0 |
| [`toolchain-versions-and-pins.md`](toolchain-versions-and-pins.md) | 27 / 0 / 0 | 5 / 0 / 0 |
| [`abi-validation-across-modules.md`](abi-validation-across-modules.md) | 24 / 0 / 0 | 6 / 0 / 0 |
| [`dokka-multi-module.md`](dokka-multi-module.md) | 18 / 0 / 0 | 9 / 0 / 0 |
| [`kmp-bom-alignment.md`](kmp-bom-alignment.md) | 21 / 0 / 0 | 10 / 0 / 0 |
| [`testing-build-logic.md`](testing-build-logic.md) | 15 / 0 / 0 | 7 / 0 / 0 |
| [`ci-release-and-branch-protection.md`](ci-release-and-branch-protection.md) | 26 / 0 / 0 | 7 / 0 / 0 |
| **Total** | **150 / 0 / 0** | **51 / 0 / 0** |

**One figure is wrong, in a claim that holds.** `build-logic-and-version-catalog.md` claim 26
gives vanniktech 0.37.0's class metadata as 2.0. Every one of the 106 classes in
`gradle-maven-publish-plugin-0.37.0.jar` carries metadata 2.2.0, as `toolchain-versions-and-pins.md`
claim 16 states. The composite-consumer floor below is reasoned from 2.2.0, so nothing changes.

**One statement holds only for public releases.** `toolchain-versions-and-pins.md` F3 says "No
9.7.2 or 9.6.2 exists". That is true of public releases only. Each advisory's Patches table lists
two kinds of fix:

- OSS: 9.8.1 and 8.14.6;
- Gradle's paid Security Subscription: 9.7.2, 9.6.2, 9.5.2, 9.4.2, 9.3.2, 9.2.2, and 9.1.1 or 9.1.2
  depending on the advisory.

The advisories' patched-versions field names only the two OSS releases. The wrapper choice is
unchanged: this build uses public distributions, and 9.8.1 is the first public fix.

**Currency.** These states changed after the findings were written or are worth knowing; no
verdict changes.

- **Gradle.** Gradle 9.8.2 (2026-10-19) is the latest public release, and no public 9.7.2 exists.
  The three advisories GHSA-mvvg-497x-hmj8, GHSA-xwqc-3h47-hg64 and GHSA-j5m7-59rp-24f5 are
  unchanged: severity high, affecting "< 8.14.6, 9.0.0 to 9.8.0". The public fixes are 8.14.6 and
  9.8.1; the fixed releases of older 9.x lines are Security Subscription releases, as the
  correction above records.
- **Kotlin and the plugins.**
  - The Kotlin documentation's last compatibility row is still "2.4.20–2.4.21 | 7.6.3–9.7.0 |
    8.5.2–9.3.1".
  - Dokka 2.2.0, vanniktech 0.37.0 and the foojay resolver 1.0.0 are the latest releases.
  - Dokka's fix for the Gradle 9.8 `Configuration.setVisible` deprecation is in no release.
- **Kotlin issue states.**
  - KT-75899 (Isolated Projects for the Kotlin/JS target) is now a duplicate of KT-80311, which is
    In Progress for 2.5.20-Beta1. KGP 2.4.21's js target still blocks Isolated Projects.
  - KT-84365 (the check/update overlap) is Fixed in 2.4.0-Beta1 for its own case; the
    `mustRunAfter` ordering stays in the design because the overlap itself remains.
- **Libraries.** `slf4j-api`'s Maven metadata names 2.1.0-alpha1 as `<release>`; the latest stable
  release is 2.0.20.
- **AGP.** An Android Gradle plugin 9.5 preview exists. AGP 9.4's minimum is still Gradle 9.6.0.

**Partial support worth knowing:**

- The JVM dump file is written once per JVM target, so "written unconditionally" holds for any
  module with a JVM target.
- `BaseNodeJsRootExtension.installationDir` is deprecated at ERROR level. The live
  `NodeJsEnvSpec.installationDirectory` derives from the same value, so Node.js still lands under
  the Gradle user home.
- Gradle's wording for the unpublished-project deprecation does not name the exact release of the
  hard error.
- Two pieces of evidence rest on source read at a revision other than the one that runs here:
  - That KMP target publications resolve as children of the root publication is read from source,
    not observed at runtime.
  - Dokka's handling of an empty source module is read at a Kotlin `master` commit, not at the
    analysis build Dokka 2.2.0 embeds.
- Coil's empty javadoc jars are measured; their cause, a lifecycle task passed as the jar's input,
  is inferred.

**Claims about this repository and its siblings,** checked against the code and the GitHub API.
All hold.

- **This repository:**
  - `.gitignore` ignores `*.jar`, which would drop `gradle/wrapper/gradle-wrapper.jar`.
  - `main` has no branch protection and no ruleset.
  - All three merge methods are enabled, `delete_branch_on_merge` is off, and the repository is
    public.
- **The game client (`GameGraphics`):** Gradle 9.0.0 wrapper, `kotlin("jvm")` 2.1.20 and
  `jvmToolchain(23)`.
- **MyGameTools:**
  - `build-logic` is included at the top level of its settings.
  - Its root `plugins {}` holds the Kotlin plugins with `apply false`, and Dokka, with versions in
    settings `pluginManagement`.
  - Commit `235810e`, "apply Kotlin & Dokka from the root so module Dokka renders", is an ancestor
    of `v5.1.0`.
  - `gradle.properties` sets the Dokka `pluginMode=V2Enabled` and `noWarn` flags.
  - `gametools.published-library` builds its javadoc jar from
    `JavadocJar.Dokka("dokkaGeneratePublicationHtml")`.
  - `verifyUmbrellaJavadocJar` counts the HTML files in the umbrella's javadoc jar.
  - `build-logic`'s `integrationTest` task is kept off `test`.
  - `master` has classic branch protection: its four job names as contexts, `strict`, a pull
    request with 0 approvals, `enforce_admins: false`. Only merge commits are allowed, and branches
    are deleted on merge.
- **GeneralTools:**
  - No `build-logic` and no catalog.
  - `languageFloor = KOTLIN_2_2`, `stdlibFloor = "2.2.0"` and `jvmFloor = 17`, with the
    `TargetJvmVersion` attribute.
  - `js { nodejs() }`, `abiValidation()` and the `-PtestLevel` parser over
    `com.spartanlabs.testing.<level>.*`.
  - `.gitattributes` holds `api/*.api text eol=lf`, with a comment that the check compares
    byte-for-byte.
  - `Build.yml` uses `checkout@v7`, `setup-java@v6`, `setup-gradle@v6` with
    `cache-provider: basic`, and `on: [push]`.
- **WebTools:** shared configuration in root `allprojects {}`/`subprojects {}` blocks, and test
  levels selected by JUnit tags.
- **The organisation's `CONTRIBUTING.md`:**
  - only "Create a merge commit" is enabled;
  - releases go through a `release/<version>` branch and a `v`-prefixed annotated tag;
  - a release workflow creates the GitHub Release;
  - registry publishing is manual, with no registry credentials in CI;
  - a `-SNAPSHOT` version sits on the default branch;
  - each repository's `CONTRIBUTING.md` states where its version lives.

---

## Summary

Each topic gives what was found, the recommended approach with the evidence behind it, where
sources disagree and which view holds, and the options that do not fit.

### Build structure: `build-logic`, the catalog and plugin loading

- **Recommended.**
  - `settings.gradle.kts` includes `build-logic` inside `pluginManagement {}`. That makes it a
    plugin build, which offers no dependency substitutions, so it adds nothing to a consumer's
    composite, and nested `build-logic` builds coexist there.
  - The settings also hold `include("graphics-bom", "graphics-core", "graphics-desktop")`,
    `rootProject.name = "SpartanGraphics"` and the foojay toolchain resolver 1.0.0. The resolver's
    version is the one version outside the catalog, because settings plugins cannot read it.
  - Three precompiled script plugins: `spartangraphics.kmp-library`, `spartangraphics.jvm-library`
    and `spartangraphics.publishing`. Typed helpers sit beside them in package
    `com.spartanlabs.graphics.build`, one file per concern: `Catalog.kt`, `SpartanBuild.kt`,
    `TestLevels.kt`, `Floor.kt`, `PublicApi.kt`, `Docs.kt` and `VerifyBomCoverage.kt`.
  - `build-logic` imports `gradle/libs.versions.toml` explicitly:
    `from(files("../gradle/libs.versions.toml"))` in its own settings. Its `build.gradle.kts`
    turns the catalog's `[plugins]` aliases into plugin-marker dependencies through the helper
    Gradle's documentation shows. Convention scripts read the catalog through
    `VersionCatalogsExtension`, wrapped once in `Catalog.kt`.
  - The root `plugins {}` block holds Kotlin multiplatform, Kotlin JVM and vanniktech with
    `apply false`, and Dokka applied. The same four aliases feed `build-logic`, so the two cannot
    load different versions, and no module script or convention plugin ever names a plugin
    version. This puts Kotlin, Dokka and vanniktech in one classloader. Dokka needs that to see
    `KotlinBasePlugin`, and Dokka and vanniktech both register build services, which break when
    projects load different plugin sets.
  - Where shared values live:
    - `group=io.github.spartanlaboratories` and `version=0.1.0-SNAPSHOT` in the root
      `gradle.properties`. Gradle assigns them to every project, the BOM included, before any
      script runs.
    - The consumer floor as the catalog version `kotlin-floor = "2.2.0"`. The language and API
      version 2.2 are derived from that one string, so the stdlib pin and the language level
      cannot drift apart.
    - The JVM level (21) and the seven test-level names as typed constants in `SpartanBuild`,
      since a catalog holds libraries, not policy.
  - `gradle.properties` turns on the configuration cache, parallel execution and the build cache.
    These runtime flags apply only to this build's own invocations: a consumer's build reads its
    own.
  - Evidence: [`build-logic-and-version-catalog.md`](build-logic-and-version-catalog.md) §6, claims
    1–11, 13, 14, 16–18.
- **What composite consumers get.**
  - Gradle substitutes `group:name`, so folder name = project name = artifactId, with `group` set
    everywhere, needs no substitution rules.
  - Consumers depend on the root coordinates. Per-target coordinates such as `graphics-core-jvm`
    are never substituted (KT-56161, "As Designed").
  - An including build runs this build on its own Gradle, and its embedded Kotlin compiles
    `build-logic`. Every jar on `build-logic`'s compile classpath carries class metadata 2.2.0 or
    lower: KGP 2.4.21 and Dokka exclude `kotlin-stdlib` on every dependency edge.
  - The composite-consumer floor is therefore **Gradle 9.0.0**. It is vanniktech 0.37.0's minimum,
    enforced through `org.gradle.plugin.api-version`, and `CONTRIBUTING.md` states it.
  - The Level 4b test below proves the floor with a Kotlin 2.1.20 consumer on Gradle 9.0.0. Gradle
    9.0.0 runs on JDK 17 to 24, and JDK 21 is provisioned through this build's own foojay resolver.
  - Evidence: [`toolchain-versions-and-pins.md`](toolchain-versions-and-pins.md) F5, claims 7, 8,
    13–17; build-logic F6.
- **Isolated Projects** is not a requirement of this build. Nothing in its scripts would violate
  it, but KGP's js target does not support it yet (KT-80311, planned for 2.5.20).
- **Rejected.**
  - Binary convention plugins: they lose the typed `kotlin {}` accessors, and the `.kt` helpers
    give the same testability.
  - The generated-accessor workaround for the catalog: it depends on the internal class
    `LibrariesForLibs`, and gradle/gradle#15383 is still open.
  - A top-level `includeBuild("build-logic")`: it makes `build-logic` a library build that offers
    substitutions.
  - A settings-level hoist: settings plugins cannot read the catalog.
  - A root-applied convention plugin as the hoist (Ktor's shape): it hides the purpose of the root
    block. It stays the fallback.
  - `allprojects {}`/`subprojects {}`: cross-project configuration.

### Toolchain, versions and the consumer floor

- **Recommended catalog versions.** Each is the latest stable release that keeps the floor
  ([`toolchain-versions-and-pins.md`](toolchain-versions-and-pins.md) §6.2, claim 30):

  | Entry | Version |
  | --- | --- |
  | Kotlin Gradle plugin | 2.4.21 |
  | `kotlin-floor` | 2.2.0 |
  | Dokka | 2.2.0 |
  | `com.vanniktech.maven.publish` | 0.37.0 |
  | kotlin-logging | 8.0.4 |
  | `slf4j-api` | 2.0.20 |
  | `logback-classic` | 1.6.5 (JVM test backend) |
  | MockK | 1.14.11 |
  | JUnit (`junit-bom`, Jupiter, `junit-platform-launcher`) | 5.14.4 |

  - **JUnit.** 5.14.4 is the last release of the 5 line, the line `kotlin-test-junit5` 2.4.21 is
    built and tested against. JUnit 6.1.3 keeps every `Assertions` member and annotation that
    `kotlin-test-junit5` calls, so moving to it is one catalog line.
  - **The JUnit Platform launcher** is declared explicitly, which Gradle 9 requires.
  - **`kotlin-test`** follows the Kotlin plugin through `kotlin("test")` and has no catalog entry.
- **The wrapper is Gradle 9.8.2,** `-bin`, with
  `distributionSha256Sum=21bd5da2de49b642904077f38c0efdc7778e92b3ed0889392dca61b97e389607`, the
  value Gradle publishes.
  - **The advisories it closes.** Three high-severity Gradle advisories published 2026-10-07 affect
    9.0.0 to 9.8.0, and 9.8.1 is the first patched public release. Fixed 9.7.x releases exist only
    through Gradle's paid Security Subscription, which this build does not use.
    - GHSA-mvvg-497x-hmj8: the worker-to-daemon channel deserialises untrusted objects.
    - GHSA-xwqc-3h47-hg64: the client-to-daemon channel deserialises before authentication.
    - GHSA-j5m7-59rp-24f5: a repository that fails its TLS handshake is not disabled, so resolution
      can fall through to another repository.

    A compromised daemon runs with the privileges of the account running the build and can read
    the secrets on disk, which at G7 include the signing key and the Central Portal token.
  - **The compatibility cost.** 9.8.2 is one minor past KGP 2.4.21's "fully supported" range, which
    ends at 9.7.0. KGP permits newer Gradle "but ... you might encounter deprecation warnings or
    some new features might not work", and it checks only a minimum version.
    - Dokka 2.2.0 prints one deprecation warning on 9.8 (`Configuration.setVisible`, scheduled for
      removal in Gradle 11).
    - vanniktech's own development branch builds on Gradle 9.8.1 with KGP 2.4.21.
  - **The fallback** is Gradle 9.7.1 (`-bin`, checksum
    `acd53f1edaf02f1a8ff99879f8a34b302661a057d9b063ae9e35b552f804d20a`), the patch of KGP's last
    fully supported minor. It is a two-line change in `gradle-wrapper.properties`
    (`distributionUrl` and `distributionSha256Sum`) if the first run shows an incompatibility.
  - Evidence: `toolchain-versions-and-pins.md` F1–F3, claims 1–7, 11 and 12.
- **The floor, the same three settings in both module kinds:**
  - `languageVersion` and `apiVersion` 2.2 at the `kotlin {}` extension level;
  - an explicit `api(kotlin-stdlib:2.2.0)` from the catalog, in `commonMain` for `graphics-core`
    and in `api` for `graphics-desktop`;
  - `coreLibrariesVersion` left alone.

  The KMP module adds `jsMain` `api(kotlin-stdlib:$coreLibrariesVersion)`, because the JS compiler
  rejects a stdlib klib older than itself. A web consumer therefore needs Kotlin 2.4, since the klib
  ABI follows the compiler. Evidence: `toolchain-versions-and-pins.md` F6, claims 21 and 22;
  GeneralTools' `kmp-build-and-kotlin-compat.md` F1–F5.
- **JVM level.**
  - Both modules use `jvmToolchain(21)` with an explicit `jvmTarget` 21, so the JVM tests run on a
    real 21.
  - The KMP `jvm` target declares `TargetJvmVersion` 21 itself: a KMP publication declares none,
    and without it a JDK 17 consumer fails at runtime instead of at resolution.
  - `jvmDefault = JvmDefaultMode.ENABLE` is set explicitly. It is the default only because
    language version 2.2 switches it on, and stating it keeps the published default-method shape
    fixed across later floor bumps.
  - Evidence: F7, F8, claims 20, 23, 24.
- **Hand-written pins, about fifteen lines in one helper, not the `com.gradleup.tapmoc` plugin.**
  tapmoc 0.5.0 is seven days old, rewrote its dependency checking in that release, is built against
  KGP 2.3.0, and would be another plugin in the root block. Its `checkDependencies()` is worth a
  look once the first third-party `api` dependency arrives (G2).
- **`-Xjdk-release`** is not used: the toolchain both restricts the JDK API and runs the tests on
  JDK 21.
- **For the Android phase:** KGP 2.4.21 documents AGP only up to 9.3.1, while AGP 9.4 is current.

### Strict explicit API and ABI validation

- **Recommended.**
  - **One helper for both module kinds.** `internal fun KotlinBaseExtension.guardPublicApi()`,
    opted in to `ExperimentalAbiValidation`, calls `explicitApi()` and
    `abiValidation { filters { exclude { annotatedWith.addAll(NON_PUBLIC_API_MARKERS) } };
    checkTaskProvider.configure { mustRunAfter(updateTaskProvider) } }`. Both members live on the
    interface the multiplatform and JVM extensions share, so the single experimental opt-in sits in
    one file. The marker list is empty in G1.
  - **The 2.4 DSL only.** The removed `enabled`, `klib {}`, `legacyDump {}`, `variants` and
    `excluded`/`included` are not used; the documentation page still shows the removed filter names.
  - **Tasks and wiring.**
    - `check` depends on `checkKotlinAbi` once `abiValidation()` is called, in both module kinds.
    - Developers run `./gradlew updateKotlinAbi` and review the dump diff.
    - The `mustRunAfter` line lets `updateKotlinAbi checkKotlinAbi` run in one invocation; Gradle 9
      rejects the pair without it.
  - **Three committed zero-byte dumps:** `graphics-core/api/graphics-core.api`,
    `graphics-core/api/graphics-core.klib.api` and `graphics-desktop/api/graphics-desktop.api`.
    `updateKotlinAbi` writes them; `checkKotlinAbi` requires them and accepts them empty (KGP's own
    tests cover these cases), and JetBrains treats an empty module as validated like any other
    (KT-78344, "As Designed"). The BOM applies no Kotlin plugin and gets no ABI task.
  - **Line endings.** The check compares `lines()` with `lines()`, so CRLF cannot fail it, but a
    trailing-newline difference does.
  - **`.gitattributes`.** The rule is `**/api/*.api text eol=lf`. The leading `**/` is required,
    because a pattern with a slash in the middle is anchored to the repository root.
  - **Explicit API scope.** Strict mode covers main and common compilations; test source sets are
    exempt.
  - Evidence: [`abi-validation-across-modules.md`](abi-validation-across-modules.md) §6, claims
    1–6, 8–22.
- **Disagreement.** GeneralTools' `.gitattributes` comment says `checkKotlinAbi` compares
  byte-for-byte. The plugin's `AbiToolsImpl.filesDiff` compares line by line. The LF rule remains
  good hygiene: `git diff` after `updateKotlinAbi` stays clean on Windows.
- **Rejected.**
  - The binary-compatibility-validator plugin: it is in maintenance mode, applied once at the root
    to configure every subproject, and lists different members from KGP's dumper.
  - Switching validation off until the first declaration: no switch exists.
  - A central dump tree: it buys nothing.

### Dokka

- **Recommended.**
  - **Where Dokka applies.** The publishing convention selects the vanniktech platform by
    `pluginManager.withPlugin(...)`. For the two Kotlin plugin ids it applies Dokka through
    `Project.configureDokka()` in `Docs.kt` and pins the platform:
    - KMP: `KotlinMultiplatform(javadocJar = JavadocJar.Dokka("dokkaGeneratePublicationHtml"),
      sourcesJar = SourcesJar.Sources())`;
    - JVM: `KotlinJvm(...)` with the same arguments;
    - `java-platform`: `JavaPlatform()`, with no javadoc jar and no Dokka.

    Pinning keeps a second Dokka plugin or a vanniktech upgrade from changing the jar silently.
  - **HTML is the only usable format**, because Dokka's Javadoc format does not support Kotlin
    Multiplatform.
  - **`gradle.properties`:** `org.jetbrains.dokka.experimental.gradle.pluginMode=V2Enabled`, with
    no `noWarn` line. V2 is the default in 2.2.0, and `noWarn` only silences a V1 message.
  - **Root aggregate.** `:dokkaGeneratePublicationHtml` writes to `build/dokka/html/`, from
    `dokka(project(":graphics-core"))` and `dokka(project(":graphics-desktop"))`.
  - **Source settings.** `jdkVersion` 21, and source links with `localDirectory` set to the
    repository root and `remoteUrl` set to `…/blob/main`. One block serves every KMP and JVM source
    set.
  - **Styling.** Dokka's default HTML theme with a footer naming Spartan Laboratories. There is no
    logo or stylesheet, because the repository has no brand assets; a theme belongs with a Pages
    site.
  - Evidence: [`dokka-multi-module.md`](dokka-multi-module.md) §6, claims 1, 2, 4–8, 12–19, 23–25.
- **Empty modules.**
  - With Kotlin detected and no declarations, Dokka drops the empty module and exits with "Nothing
    to document"; the task succeeds.
  - Every javadoc jar is manifest-only, which Central accepts as a placeholder.
  - The aggregate renders an "All modules" page with no rows.
- **The guard against a silently empty site.**
  - In G1, the classloader failure and "no source yet" both render zero pages, so a page count
    cannot separate them. Neither can `failOnWarning`: it counts only generator warnings.
  - The configuration model does separate them. When Kotlin is visible, Dokka registers a
    documented source set per Kotlin source set, even an empty one; when it is not, none.
  - `verifyDokkaSourceSets`, a small typed task per library module, fails when no Dokka source set
    is documented. It is wired into `check` and ahead of the module's Dokka tasks, and it keeps
    working once real API lands.
  - A grep of the CI log for Dokka's warning is not used: the warning is printed only when a
    configuration-cache entry is stored.
- **Rejected.**
  - The lifecycle task `dokkaGenerate` as the jar's input: it has no outputs. Coil 3.6.3 passes it
    and ships 261-byte javadoc jars.
  - Applying Dokka unconditionally in the publishing convention: the BOM would receive it.
  - A fourth convention id for documentation.

### `graphics-bom`

- **Recommended.**
  - **The BOM project.** A `java-platform` project applying the publishing convention, which
    selects `JavaPlatform()`. Its constraint list is written by hand and grouped by module kind in
    `graphics-bom/build.gradle.kts`:
    - for each KMP module, `api(project(":graphics-core"))` plus string constraints for
      `graphics-core-jvm` and `graphics-core-js`;
    - for each JVM module, `api(project(":graphics-desktop"))`.

    A project constraint yields only the root coordinate, since KGP's target publications are
    children of the root. Maven consumers, and the POMs Gradle writes for JVM modules, name
    `graphics-core-jvm`. The published BOM therefore holds four entries, all at the shared version.
  - **Each module's dependency on the BOM:**
    - KMP: `commonMain` `api(project.dependencies.platform(project(":graphics-bom")))`, the form
      that replaces `KotlinDependencyHandler.platform()`, removed in Kotlin 2.4.0;
    - JVM: `api(platform(project(":graphics-bom")))`.

    This is Gradle's documented alignment shape, and Gradle's own documentation tests run it with
    publishing, so it has no cycle. The published modules record the BOM as
    `org.gradle.category: platform` with `endorseStrictVersions`, and the POMs import it.
  - **Mixed versions.** The highest requested BOM wins and its constraints apply. A consumer
    holding `graphics-core` 0.2.0 and `graphics-desktop` 0.1.0 resolves both at 0.2.0, which is
    the runtime mix the BOM exists to prevent.
  - **Completeness.** `verifyBomCoverage`, a typed task in `build-logic` on the BOM's `check`,
    scans `*/build.gradle.kts` for the two library convention ids and fails when the lists and the
    modules on disk disagree. It reads files only, so it involves no cross-project configuration.
    The Level 3 tests below also map the KMP target publications to the BOM's per-target entries.
  - Evidence: [`kmp-bom-alignment.md`](kmp-bom-alignment.md) §6, claims 1–5, 7–16, 19–21, 24, 25,
    28.
- **The least-evidenced step.** Kotlin's own libraries put their BOM in `jvmMain` only, and no
  published KMP library with a `js` target was found that depends on its own BOM from
  `commonMain`. Kotlin's usage-compatibility rules and its integration test support `commonMain`.
  The implementation checks it once, against `jsCompileClasspath` and the generated `.module`
  files; the fallback is the `jvmMain` form, which leaves `js` consumers unaligned.
- **Rejected.**
  - A generated BOM: every generator read needs another project's state, and OkHttp 5.5.0's
    generated BOM constrains tests and samples, one at version `unspecified`.
  - `enforcedPlatform` or `strictly` in a library: both propagate to every consumer.
  - `allowDependencies()`: it removes the fail-fast on a stray dependency.

### Testing the build logic

- **Recommended.**
  - **The test source set.** Every test lives in `build-logic/src/test/kotlin/` under
    `com.spartanlabs.testing.<level>.graphics.build`, one class per file, 17 classes in all.
  - **Level 1 is a process.**
    - **The pre-commit run** is `./gradlew check :build-logic:test -PtestLevel=component,deterministic`.
      Its `check` runs the real-build guards: `checkKotlinAbi`, `verifyDokkaSourceSets`,
      `verifyBomCoverage`, and the strict explicit-API compile.
    - **The manual checks** come with it, and their outcomes go in the pull request's description.
      G1's are:
      - the first run of the hoist negative control;
      - `build-logic` assembled on Gradle 9.0.0;
      - copies of the real client, built on its own wrapper and on this build's;
      - an Isolated Projects diagnostic;
      - a `--warning-mode all` run;
      - a `core.autocrlf` check on the maintainer's machine;
      - a look at the rendered site;
      - an IDE import.

      The `--warning-mode all` run repeats at every wrapper or plugin bump. The js filter execution
      runs when the first `commonTest` class lands in G2. Both are Level 1 checks of those later
      changes.
    - **`gating` is not a selectable `-PtestLevel` name.** It would select nothing, so
      `-PtestLevel=gating` fails with a message that names the six levels and points to the
      Level 1 definition in `CONTRIBUTING.md`.
  - **Level 4a:** pure functions. `parseTestLevels`, `bomCoverageProblems` and the floor parser,
    which returns `Result<KotlinVersion>`.
  - **Level 2:** the task classes, `configureTestLevels` and `Catalog.kt`, with `ProjectBuilder`,
    `@TempDir` and MockK.
  - **Level 3.**
    - **`RepositoryContractTest`** checks the committed dumps, the `.gitattributes` anchor, the
      `include(...)` list against the module directories, `group` and `version` in
      `gradle.properties`, and that no source set holds a `testing/gating` package.
      - It belongs at Level 3 because it reads the real repository's files, so it is no isolated
        component test.
      - What it checks is the contract between the committed files and the tools that consume
        them: Gradle's settings and ABI tasks, and git's attributes.
      - It runs in-process.
    - **Seven Gradle TestKit classes** run against a mutated **copy of the real build**: explicit
      API, the ABI guard, the test-level filter, the compiler contract, the published metadata, the
      Dokka guard with the negative control that removes the root hoist, and BOM coverage.
  - **Level 4b:** `CompositeConsumerFloorTest`, a generated Kotlin 2.1.20 consumer on Gradle 9.0.0
    that includes the copy.
  - **Level 4c:** `ConfigurationCacheTest`, a store-and-reuse pair. Gradle prescribes two runs.
  - **Level 5: none in G1.**
    - Level 5 covers what a downstream developer sees directly: public authoring APIs, installable
      systems and lookup `Result`s.
    - G1's public API is empty. What a consumer meets in G1 is packaging: the coordinates, the BOM
      and the composite include. Levels 3 and 4b decide its behaviour mechanically.
    - `uat` stays a selectable level for the first authoring API.
  - Evidence: [`testing-build-logic.md`](testing-build-logic.md) §6, claims 1, 2, 4–16.
- **No `withPluginClasspath()`.** TestKit's injected classpath loads every plugin from one
  classloader. The arrangement the root hoist creates would then always hold, and the negative
  control could never bite.
- **Task wiring.**
  - `:build-logic:test` (Levels 2 and 4a) runs on `:build-logic:check`.
  - `integrationTest` (Level 3, including the in-process `RepositoryContractTest`), `e2eTest` and
    `nonfunctionalTest` are separate `Test` tasks selected by package, kept off `check`, never up to
    date, and **never cached**.
  - `Test` is a `@CacheableTask`, and the repository turns the build cache on, so both the heavy
    tasks and `:build-logic:test` carry `outputs.doNotCacheIf(...)`. Otherwise a run could be
    loaded from the cache without testing the changed files. The nested runs pass
    `--no-build-cache` as well.
  - `build-logic`'s tests take no logback and no kotlin-logging. `kotlin-dsl`'s `gradleApi()`
    brings its own SLF4J provider, and the two would conflict.
- **Avoiding Node.js.** Nested runs name their tasks and never run `check` or the js test task, so
  Node.js and Yarn are not downloaded. The js test filter is read from the configured `TestFilter`
  instead of being executed.
- **Disagreement.** A permanent CI job that assembles `build-logic` on Gradle 9.0.0
  (`toolchain-versions-and-pins.md`, its Consideration C3) is replaced by the Level 4b test, which
  proves strictly more.

### CI, the release workflow and the protection of `main`

- **Recommended `ci.yml`.** Four plain jobs; each job's `name:` is the required-check name:

  | Job | Runner | Command |
  | --- | --- | --- |
  | `Build (windows-latest)` | `windows-latest` | `./gradlew build :build-logic:check :dokkaGeneratePublicationHtml javadocJars --continue --stacktrace` |
  | `Compile (ubuntu-latest)` | `ubuntu-latest` | `./gradlew assemble testClasses jvmTestClasses jsTestClasses` |
  | `Compile (macos-latest)` | `macos-latest` | the same compile command |
  | `build-logic (TestKit)` | `windows-latest` | the three nested suites, with `--no-build-cache` |

  - **What the jobs cover.**
    - The Windows build runs every non-GL test, the ABI checks through `check`, the Dokka
      aggregate and every javadoc jar.
    - The compile jobs compile main and test sources of every module, without `check`, Dokka or
      the ABI check. The klib dump is host-independent.
    - The TestKit job is required on every pull request. It is estimated at 8 to 14 minutes cold on
      Linux-class I/O and up to twice that on Windows. If the first measured runs exceed about 15
      minutes, a job-internal change check skips it for documentation-only pull requests
      (`ci-release-and-branch-protection.md` §6.8); a workflow-level path filter would leave the
      required check pending.
    - Plain jobs rather than a matrix keep the check names literal, and the Linux job will later
      differ, when it hosts the GL tests.
  - **Triggers.** `pull_request` with no branch or path filter, `push` to `main`, and
    `workflow_dispatch`. Concurrency cancels superseded pull-request runs only.
  - **Actions.**
    - Every job runs on Temurin JDK 21, with `gradle/actions/setup-gradle` and
      `cache-provider: basic`. The default provider requires accepting Gradle's terms for its
      proprietary component.
    - `setup-gradle` validates the wrapper itself, against Gradle's published checksums.
    - The current releases are `actions/checkout` v7.0.1, `actions/setup-java` v6.1.0,
      `gradle/actions` v6.4.0 and `actions/upload-artifact` v7.0.2. They are pinned by major tag
      (`@v7`, `@v6`, `@v6`, `@v7`), so patch and security releases arrive without edits.
      - The exposure a moved tag would bring is small. Every job that runs an action holds a
        read-only token and no secret, and the one job with a write token runs no action.
      - SHA pins would go stale with nothing to bump them.
      - No SHA-pinning requirement is set on the repository: with it, "all actions must be pinned
        to a full-length commit SHA to be used", including "actions authored by GitHub" (GitHub
        Docs, `data/reusables/actions/actions-use-policy-settings.md`).
  - **Gradle in CI.**
    - On CI the nested TestKit builds use the runner's Gradle user home, so the job's cache covers
      them. Locally they keep TestKit's isolated home, because the runner has no user-level
      `gradle.properties` to leak.
    - `setup-gradle` writes the cache only from `main`, and it keeps no configuration-cache entry
      without an encryption key. Every CI run stores a fresh entry, so a problem still fails the
      build; reuse is what the Level 4c test covers.
  - **Repository mechanics.**
    - `gradlew` is committed with mode `100755` and kept LF through `.gitattributes`.
    - PowerShell, the default shell on Windows, runs `./gradlew` as is.
  - Evidence: [`ci-release-and-branch-protection.md`](ci-release-and-branch-protection.md) §6,
    claims 1–6, 9–11, 15, 18–22, 24, 27–29.
- **Recommended `release.yml`.**
  - **Trigger:** a pushed `v*` tag.
  - **Verify job, on Windows.**
    - It checks through the refs API that the tag is annotated, without a checkout.
    - It checks that the tag equals `v` plus `version=` in `gradle.properties`.
    - It runs the same Gradle gate as the Windows CI job.
  - **Release job.** It holds `contents: write`, runs no checkout and no third-party action, and
    calls `gh release create "$GITHUB_REF_NAME" --verify-tag --title … --generate-notes` with
    `GH_REPO` set.
  - **It publishes nothing:** it references no secret and uploads no asset. The tag name is read
    from the environment, never interpolated into the script.
- **Recommended protection of `main`.**
  - **Mechanism.** Classic branch protection, applied with one idempotent
    `PUT repos/SpartanLaboratories/SpartanGraphics/branches/main/protection` and read back with a
    `GET`. Rulesets would work too; classic is the mechanism the organisation's baseline names and
    MyGameTools uses.
  - **The payload:**
    - a pull request is required, with 0 approvals and stale-review dismissal;
    - the four checks are bound to the GitHub Actions app (id 15368), so no one with write access
      can satisfy them by posting a commit status;
    - `strict: true`;
    - conversations must be resolved;
    - force pushes and deletion are blocked;
    - linear history is off, because it would forbid merge commits;
    - administrators are included (`enforce_admins: true`). With one maintainer, an exempt
      administrator would leave the rule advisory. A pull request stuck on a check that cannot
      pass waits until admin enforcement is removed and restored: one `DELETE` and one `POST` on
      `.../protection/enforce_admins`, given in `ci-release-and-branch-protection.md` §6.4.3.
  - **Order of operations:**
    1. Commit `gradlew` as `100755`.
    2. Open the G1 pull request; its run creates the four check names. Compare them with the
       payload, character for character.
    3. Read the first runs: the job durations, whether the compile jobs set up Node.js, and any
       `FROM-CACHE` on the `build-logic` test tasks.
    4. Once the maintainer confirms them, apply the repository settings in two commands:
       - `PATCH repos/SpartanLaboratories/SpartanGraphics` with `allow_squash_merge=false`,
         `allow_rebase_merge=false`, `delete_branch_on_merge=true`, `merge_commit_title=PR_TITLE`
         and `merge_commit_message=PR_BODY`. Merge commits stay enabled.
       - `PUT .../actions/permissions/workflow` with `default_workflow_permissions=read`.

       Applying them before the merge gives the G1 merge the baseline's commit text and deletes its
       branch.
    5. Merge.
    6. Within seven days of a green run, apply the protection once the maintainer confirms it.
    7. Prove it: a trivial pull request shows the four checks as required, and a direct push is
       rejected.

    The exact payload, rollback and later-change commands are in
    `ci-release-and-branch-protection.md` §6.4 and §6.7.

### Where sources disagree

| Point | Views | What holds, and why |
| --- | --- | --- |
| The wrapper line | 9.7.1 sits inside KGP 2.4.21's fully supported range; 9.8.1 and later fix three high-severity advisories that cover 9.0.0 to 9.8.0 | 9.8.2: KGP permits newer Gradle, and the cost on the evidence is one deprecation warning, against advisories on the machine that holds the release secrets; 9.7.1 is the two-line fallback |
| JUnit line | 6.1.3 as the latest stable (toolchain findings); 5.x as named by the testing requirement (testing findings) | 5.14.4: the line named by the requirement and the one `kotlin-test-junit5` is tested against; 6.x is a one-line move |
| Guarding the Gradle 9.0.0 floor | a permanent CI job compiling `build-logic` on 9.0.0 (toolchain C3); a TestKit consumer test (testing findings) | The Level 4b test: it also proves substitution, the stdlib pin and the JVM-attribute failure |
| Where Dokka is applied | unconditionally in the publishing convention (build-logic §6.9); only for the Kotlin kinds (Dokka §6.3) | Only for the Kotlin kinds: the BOM must not receive Dokka |
| Detecting an empty site | a page count or a log grep (build-logic §5); a source-set check (Dokka F6) | The source-set check: in G1 both failure and "no source" give zero pages, and the warning is not re-printed on a configuration-cache hit |
| Keeping heavy tests honest | `outputs.upToDateWhen { false }` (testing §6.4); also `doNotCacheIf` (CI F16) | Both: `Test` is cacheable and the build cache is on |
| Isolated Projects | nothing blocks it later (build-logic F7); KGP's js target blocks it (ABI F2) | The js target blocks it (KT-80311); this build adds no violation of its own |
| ABI dump comparison | byte-for-byte (GeneralTools' `.gitattributes` comment); line by line (ABI F5) | Line by line, read in `AbiToolsImpl.filesDiff` at `v2.4.21` |
| Pinning actions | full SHAs (CI findings, GitHub's guidance, Okio); major tags (Coil, MyGameTools, GeneralTools) | Major tags: the actions are GitHub's and Gradle's own, every action-running job holds a read-only token and no secret, and SHA pins would go stale with nothing to bump them |

### Known gaps

Nothing in this research was built or run; the first local runs and the first pull-request run
are the measurements. The items they settle:

- **The hoist's negative control.** Does removing the root `apply false` Kotlin aliases reproduce
  Dokka's empty output with a `pluginManagement` `build-logic`? It is read from Gradle's
  classloader behaviour; the measured MyGameTools case used a top-level include. Manual step M1
  settles it before the Level 3 test is pinned.
- **The BOM reaching `js` consumers.** Whether the `commonMain` platform dependency reaches the
  `js` resolution and the published `-js` module is checked once in implementation, with the
  `jvmMain` form as the fallback.
- **Dokka's analysis of empty `commonMain`, `jvmMain` and `jsMain` source sets,** at the Kotlin
  analysis build Dokka 2.2.0 embeds.
- **Node.js on the compile jobs.** Whether `assemble` or the test-class tasks pull Node.js and Yarn
  costs seconds, not correctness.
- **The runtime half of the Gradle 9.0.0 floor.** KGP 2.4.21 applying under Gradle 9.0.0 through the
  nested `build-logic` is proven by the Level 4b test and manual steps M2 and M3.
- **`mustRunAfter`** ordering `updateKotlinAbi` before `checkKotlinAbi` in one invocation.
- **The protection payload.** Whether `checks` without the deprecated `contexts` is accepted, and an
  explicit `app_id` for a check that has not yet run. The order of operations avoids the second
  case, and the findings give the fallback body.
- **Timings,** above all the TestKit job on `windows-latest`.
- **Gradle 9.8.2 with KGP 2.4.21, Dokka 2.2.0 and vanniktech 0.37.0** has run only in vanniktech's
  own build. This build's first run is the check, and Gradle 9.7.1 is the fallback.
- **A release-version build without signing keys.** `./gradlew build -Pversion=0.1.0` is run once
  locally before G7.
