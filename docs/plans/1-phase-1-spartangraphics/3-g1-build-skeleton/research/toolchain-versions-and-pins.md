# Toolchain versions, the Gradle wrapper and the floor pins for the G1 build skeleton

- **Issue:** [SpartanLaboratories/SpartanGraphics#3](https://github.com/SpartanLaboratories/SpartanGraphics/issues/3), question 2 of the
  [research README](README.md).
- **Read:** 2026-10-20. Pinned sources: Kotlin `v2.4.21`, Gradle `v9.0.0` / `v9.7.1` / `v9.8.2`, Dokka `v2.2.0`, vanniktech `0.37.0`,
  foojay `foojay-toolchains-plugin-1.0.0`, tapmoc `v0.5.0`, Ktor `3.6.0`, Coil `3.6.3`, MyGameTools `dc5214f`, GeneralTools branch
  `feat/issue-7-kotlin-multiplatform` @ `ebb2528`.
- **Method limit:** nothing was built or run and no Gradle task was executed. Every behaviour below comes from documentation, source at
  pinned tags, Maven Central metadata (`maven-metadata.xml`, `.pom`, `.module`), Gradle's published distribution archives, and two
  measurements of published artifacts (class-file metadata of plugin jars; the jar listings of two Gradle distributions). The wiring
  has not been executed in this repository; *Known gaps* lists the checks the implementation must run.

## 1. Question

Which versions does `gradle/libs.versions.toml` hold; which Gradle 9.x wrapper is the newest inside the documented range of every
applied plugin (the Kotlin Gradle plugin 2.4.21, Dokka, `com.vanniktech.maven.publish`, the foojay toolchain resolver, and Gradle's
`kotlin-dsl`); and how do the consumer floor (`languageVersion`/`apiVersion` 2.2, a JVM and common `kotlin-stdlib` requirement of
2.2.0) and the JVM 21 level apply to both the Kotlin Multiplatform module (`graphics-core`) and the `kotlin("jvm")` module
(`graphics-desktop`)? An answer must keep the floor in what each published variant declares (Gradle module metadata and POM), not
only in how code compiles; must give a consumer on an older JVM a resolution-time failure; and must state the Gradle version a
composite consumer needs, since a consumer's Gradle compiles this build's `build-logic` with its own embedded Kotlin.

## 2. Bottom line

- **Wrapper: Gradle 9.7.1, `-bin` distribution.** Kotlin's own documentation puts the maximum fully supported Gradle for KGP
  2.4.20–2.4.21 at 9.7.0, and 9.7.1 is the patch that replaces 9.7.0 (it fixes six 9.7.0 regressions, including one in Kotlin DSL
  scripts). KGP has no upper-bound check at all, vanniktech 0.37.0 and the Dokka 2.2.0 and foojay 1.0.0 plugins state only
  minimums or "tested up to" lists, and Ktor 3.6.0 and Coil 3.6.3 run exactly Gradle 9.7.1 with vanniktech 0.37.0 and Dokka 2.2.0.
  9.7.1 satisfies AGP 9.4's minimum of Gradle 9.6.0. `distributionSha256Sum` is
  `acd53f1edaf02f1a8ff99879f8a34b302661a057d9b063ae9e35b552f804d20a` (published by Gradle; recomputed from the downloaded zip).
  **Flag (Consideration C1):** three high-severity Gradle advisories published 2026-10-07 affect "9.0.0 to 9.8.0" and are patched
  only in 9.8.1 and later; the 9.7.x line has no fixed release.
- **Composite-consumer floor: Gradle 9.0.0.** vanniktech 0.37.0's own minimum is 9.0.0 and its `apiElements` variant declares
  `org.gradle.plugin.api-version = 9.0.0`. The feared failure (Kotlin 2.4 class metadata reaching the embedded Kotlin 2.2.0
  compiler of Gradle 9.0.0) does not arise on the evidence: every dependency edge of KGP 2.4.21 and Dokka 2.2.0 excludes
  `kotlin-stdlib`, vanniktech's `apiElements` has no dependencies, and the highest class-metadata version in all eleven jars on
  `build-logic`'s compile classpath is 2.2.0. `kotlin-dsl` sets no metadata-skip flag, and needs none. The cheapest way to make it
  certain is `gradle -p build-logic assemble` under a 9.0.0 distribution, plus one composite smoke test (section 6.6).
- **The floor is the same three settings in both module types:** `languageVersion = apiVersion = 2.2` at the `kotlin {}` extension
  level; an explicit `api(kotlin-stdlib:2.2.0)` taken from the catalog (in `commonMain` of the KMP module, in `api` of the JVM
  module); `coreLibrariesVersion` left alone. The KMP-only addition is `api(kotlin-stdlib:<KGP version>)` for the `js` target.
  For the JVM module an explicit `api` pin beats `coreLibrariesVersion = 2.2.0`: it is the same mechanism as the KMP module, it
  keeps `kotlin-test` at the compiler's version, and it publishes the same `requires 2.2.0`.
- **JVM level:** `jvmToolchain(21)` plus an explicit `jvmTarget = JVM_21` in both modules. A KMP `jvm` publication declares no
  `org.gradle.jvm.version` by itself, so the KMP module sets `TargetJvmVersion = 21` explicitly. `kotlin("jvm")` + `java-library`
  infers it from `JavaCompile`'s target, which the toolchain sets to 21. `-jvm-default=enable` is the default in 2.4.21 only because
  the 2.2 language version turns the `JvmDefaultEnableByDefault` feature on; set it explicitly (`JvmDefaultMode.ENABLE`) so a later
  floor bump cannot silently drop `DefaultImpls`.
- **Hand-written pins, not `com.gradleup.tapmoc`.** The pins are about fifteen lines in a shared helper that both convention plugins
  call; having several modules does not multiply them. tapmoc 0.5.0 is seven days old, rewrote its dependency-checking machinery in
  that release, is built against KGP 2.3.0, and would add a fifth plugin to the hoisted root classpath. Revisit its
  `checkDependencies()` when the first third-party `api` dependency arrives (G2).
- **Catalog libraries:** JUnit Jupiter 6.1.3 (through `junit-bom`, with an explicit `junit-platform-launcher`), MockK 1.14.11,
  logback-classic 1.6.5, `slf4j-api` 2.0.20, kotlin-logging 8.0.4. JUnit 6.1.3 keeps every `Assertions` method and annotation that
  `kotlin-test-junit5` 2.4.21 uses (checked by signature), and JUnit 5.14.4 is the one-line fallback (C2).

## 3. Findings

Tiers: **consensus** = official documentation or specification, or several independent credible sources; **practice** = what mature
libraries visibly do; **opinion** = one source; **inference** = derived here from sources that do not state it.

### F1. The documented Gradle range of every applied plugin (consensus for the facts; inference for the patch-release reading)

| Plugin (version) | Documented minimum Gradle | Documented upper statement | Enforced in code? |
| --- | --- | --- | --- |
| Kotlin Gradle plugin 2.4.21 (and 2.4.20) | 7.6.3 | "maximum fully supported version is 9.7.0"; newer Gradle usable "but ... you might encounter deprecation warnings or some new features might not work" ([Kotlin docs](https://kotlinlang.org/docs/gradle-configure-project.html), 2026-10-20) | Minimum only. `GradleCompatibilityCheck` (KGP `v2.4.21`) is fatal below 7.6.3 and a deprecation warning below 8.14.4; there is no check above any version. |
| `com.vanniktech.maven.publish` 0.37.0 | 9.0.0 | "Compatibility tested up to: Gradle 9.6.0, Gradle 9.7.0-milestone-1" ([CHANGELOG at 0.37.0](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/CHANGELOG.md)) | No Gradle version check in plugin sources (`gh search code` for `GradleVersion` finds test code only). Its `apiElements` variant carries `org.gradle.plugin.api-version = 9.0.0`, which makes Gradle 8.x fail to resolve it. |
| Dokka Gradle plugin 2.2.0 | 7.6 ("7.6 or higher", [Dokka docs](https://kotlinlang.org/docs/dokka-gradle.html)) | none; its CI tests Gradle 7.6.3 and 8.14.3 only (`TestedVersions` at `v2.2.0`) | none |
| foojay resolver convention 1.0.0 (2025-05-19, still the latest) | 7.6 ("Requires Gradle 7.6 or later") | none | none |
| Gradle `kotlin-dsl` (embedded Kotlin) | the consumer's own Gradle | tied to the Gradle release: 9.0.0 embeds Kotlin 2.2.0, 9.7.x embeds 2.4.0, 9.8.x embeds 2.4.10 | n/a |

- KGP 2.4.20's release notes headline "Gradle: Support for Gradle 9.7.0" and say "Kotlin 2.4.20 is fully compatible with Gradle 7.6.3
  through 9.7.0. You can also use Gradle versions up to the latest Gradle release." So the ceiling is "the newest Gradle that KGP's
  integration suite ran on", which is a test statement, not a prohibition. The suite pins literal strings (`TestVersions.Gradle.G_9_7 =
  "9.7.0"`, `MAX_SUPPORTED = G_9_7`, and the same list in the integration-test `build.gradle.kts`).
- **What "up to 9.7.0" means for 9.7.1 (inference).** Read literally, 9.7.1 sorts above 9.7.0. Three facts say it belongs to the range:
  Gradle defines a patch release as one "to replace the final release due to critical bug fixes or regressions" ("Gradle 5.2.1
  replaces the Gradle 5.2 release"); KGP checks nothing above its minimum; and KGP 2.4.21 itself treats 8.14.4 as the point below
  which 8.14 is deprecated while the docs for 2.2.x list the ceiling as "8.14", so JetBrains already counts the patches of a documented
  minor as inside it. The strictly literal alternative, 9.7.0, carries the six regressions fixed in 9.7.1 and gives nothing back.
- **vanniktech's list is a snapshot, not a ceiling.** `GradleVersionProvider` in its integration tests runs "minimum supported" plus the
  latest stable, RC, beta and alpha at release time (`TestOptions.kt` at `0.37.0`), which is why a 9.7.0 milestone appears. Its
  unreleased `main` already builds on Gradle 9.8.1 and KGP 2.4.21 (`gradle/libs.versions.toml` on `main`, commit "Update Gradle to
  v9.8.1 (#1457)", 2026-10-07).
- **Nothing in this table lowers the wrapper below 9.7.x.** The binding ceiling is KGP's 9.7.0; the binding floor for the wrapper is AGP
  9.4's 9.6.0 (F4).

### F2. Gradle 9.7.1 against 9.7.0, 9.8.x and the siblings that run it (consensus on the releases; practice on the pairing)

- **9.7.1** (2026-08-19, "the first patch release for Gradle 9.7.0", "We recommend upgrading to Gradle 9.7.1") fixes six 9.7.0
  regressions: the bundled antlr leaking onto the kapt classpath, `Option` annotation arguments changing order in
  `build.gradle.kts`, `BaseExecSpec` stream conformance, the "Click to see difference" test format, a broken `Transformer`, and
  `ant.taskdef` classpath resolution ([9.7.1 release notes](https://docs.gradle.org/9.7.1/release-notes.html)). It embeds Kotlin
  2.4.0 (its `lib/` holds `kotlin-compiler-embeddable-2.4.0.jar` and `kotlin-stdlib-2.4.0.jar`, listed from the downloaded zip).
- **Siblings on Gradle 9.7.1:** Ktor 3.6.0 (`gradle-wrapper.properties`; `dokka = "2.2.0"`, `mavenPublishing = "0.37.0"`, `kotlin =
  "2.3.21"`, AGP 9.4.0), Coil 3.6.3 (Gradle 9.7.1, KGP 2.4.10, Dokka 2.2.0, vanniktech 0.37.0), and MyGameTools `dc5214f` (Gradle 9.7.1,
  Dokka 2.2.0, foojay 1.0.0, vanniktech 0.36.0, which rendered 415 Dokka pages in `v5.1.0`). So Gradle 9.7.1 with Dokka 2.2.0,
  vanniktech 0.36/0.37 and foojay 1.0.0 is in production; KGP 2.4.21 on 9.7.1 is the documented pairing.
- **9.8.x** is outside KGP's "fully supported" label, and Dokka 2.2.0 already trips a 9.8 deprecation: `Configuration.setVisible(boolean)`
  "has been deprecated. This is scheduled to be removed in Gradle 11" ([Kotlin/dokka#4600](https://github.com/Kotlin/dokka/issues/4600),
  fixed on `master` by PR #4601 on 2026-09-11, after the 2.2.0 release). Gradle 9.8 also deprecates `Signature` as a configuration
  artifact and signing with a classifier (9.x upgrade guide, section "Upgrading from 9.7.0 and earlier"); whether vanniktech 0.37.0
  triggers those was not measured.
- **Checksums** (all from `services.gradle.org/distributions/*-bin.zip.sha256`): 9.7.1 `acd53f1e…d20a` (verified by hashing the
  151,433,392-byte download), 9.7.0 `84fbba45c7f4c64abc77460e1c00f541e9f960e3c7ed2538f1ede19eacd873ae`, 9.8.2
  `21bd5da2de49b642904077f38c0efdc7778e92b3ed0889392dca61b97e389607`, 9.0.0
  `8fad3d78296ca518113f3d29016617c7f9367dc005f932bd9d93bf45ba46072b` (also verified by hashing). The 9.7.1 wrapper jar's published
  checksum is `7a9ce74cff467ca1bf60a4fcd9f05185acceda4d0f382434d393e17864262c5d` (for wrapper validation in CI).

### F3. The 9.7.x line is inside the range of three unpatched high-severity advisories (consensus: Gradle's published advisories)

- GHSA-mvvg-497x-hmj8 (unauthenticated worker-to-daemon channel deserializes untrusted Java objects), GHSA-xwqc-3h47-hg64
  (deserialization before authentication on the client-daemon channel) and GHSA-j5m7-59rp-24f5 (a repository that fails an SSL
  handshake is not disabled, so resolution can fall through to another repository), all published 2026-10-07, all severity high.
  Affected: "< 8.14.6, 9.0.0 to 9.8.0"; patched: 8.14.6 and 9.8.1. Gradle's 9.8.2 notes repeat that 9.8.1 "addresses three high rated
  vulnerabilities". No 9.7.2 or 9.6.2 exists.
- The two daemon advisories need an attacker able to place bytes on a loopback socket of the user's daemon; the third needs a network
  position. Code execution in the daemon "runs with the privileges of the user running the build and exposes in-memory and on-disk
  secrets", which at G7 includes the Central Portal token and the signing key. This is a trade against the documented-range rule, so
  it is stated as Consideration C1 rather than decided here.

### F4. AGP 9.4 and the Android phase (consensus)

- AGP 9.4.0 lists "Gradle 9.6.0" as minimum and default ([AGP 9.4 release notes](https://developer.android.com/build/releases/gradle-plugin)),
  so 9.7.1 (and 9.8.x) satisfies it.
- New fact for that phase: KGP 2.4.20–2.4.21 documents AGP "8.5.2–9.3.1" (Kotlin docs table), and KGP's own `AgpCompatibilityMatrix`
  tops out at `AGP_93`. AGP 9.4.x is therefore one minor beyond what KGP 2.4.21 states as fully supported. Ktor 3.6.0 already runs AGP
  9.4.0 with vanniktech 0.37.0, so the pairing exists in practice.

### F5. What a composite consumer's Gradle needs (consensus on the mechanism; the compile classpath is established by metadata, not by a run)

**Mechanism.** A consumer's `includeBuild("../SpartanGraphics")` compiles `build-logic` with the consumer's Gradle and that Gradle's
embedded Kotlin: 9.0.0 → Kotlin 2.2.0, 9.7.x → 2.4.0, 9.8.x → 2.4.10 (the compatibility table at `v9.8.2`; the 9.0.0 and 9.7.1
distributions were opened and hold exactly `kotlin-compiler-embeddable-2.2.0.jar` and `-2.4.0.jar`). The embedded compiler reads class
metadata up to "max(X+1, Y)" for compiler X and language version Y (`MetadataVersion.lastSupportedVersionWithThisLanguageVersion` at
`v2.2.0`), so Gradle 9.0.0 reads metadata ≤ 2.3.0. `kotlin-stdlib` 2.4.21 has metadata 2.4.0 and would be rejected.

**What reaches `build-logic`'s compile classpath.** Resolving the `java-api` variants of the four plugins for Gradle 9.0.0 (a small
resolver script over the `.module`/`.pom` files, following `excludes`) gives eleven artifacts: `kotlin-gradle-plugin`,
`kotlin-gradle-plugin-api`, `fus-statistics-gradle-plugin`, `kotlin-gradle-plugin-annotations`, `kotlin-native-utils`,
`kotlin-tooling-core`, `kotlin-build-tools-api`, `kotlin-util-io`, `dokka-gradle-plugin`, `dokka-core`, and
`gradle-maven-publish-plugin`. **No `kotlin-stdlib` is reached.** KGP's `.module` lists `kotlin-stdlib` under `excludes` on every
dependency of every one of its variants (zero dependencies without the exclusion), Dokka's variants exclude it likewise, and
vanniktech's `apiElements` has no dependencies at all (its runtime-only `central-portal` brings okhttp 5.1.0, retrofit and moshi,
which are not on the compile classpath). The class-file `kotlin.Metadata` `mv` values of those jars, measured by scanning every
class (2026-10-20):

| Jar | `mv` values (class counts) |
| --- | --- |
| `kotlin-gradle-plugin-2.4.21-gradle813` (selected on 9.0.0; `-gradle96` is selected on 9.6+ and is the same) | 1.7.1 (365), 1.8.0 (6,948), **2.2.0 (2,472)** |
| `kotlin-gradle-plugin-api`, `-annotations`, `-native-utils`, `-tooling-core`, `-build-tools-api`, `-util-io`, `fus-statistics-gradle-plugin` | 1.8.0 only |
| `dokka-gradle-plugin-2.2.0` / `dokka-core-2.2.0` | 1.4.3 and 1.8.0 / 1.4.3 |
| `gradle-maven-publish-plugin-0.37.0` | **2.2.0 (106)** |
| `kotlin-stdlib-2.4.21` (not on the compile classpath) | 2.4.0 (951) |

The maximum is 2.2.0, which the Kotlin 2.2.0 compiler reads natively, and the stdlib that would carry 2.4.0 is excluded. Gradle's
`kotlin-dsl` configures the compile with `-java-parameters -Xjvm-default=all -Xjsr305=strict -Xjspecify-annotations=strict
-Xsam-conversions=class -Xuse-fir-lt=false` on 9.0.0 (`-jvm-default=enable` at `v9.8.2`) and language and API version 2.2; there is no
`-Xskip-metadata-version-check`, and the analysis shows none is needed. The same constraint explains vanniktech's own setup: its
classes are 2.2.0 because its minimum Gradle embeds Kotlin 2.2.0.

**Conclusion.** The floor is **Gradle 9.0.0**: vanniktech 0.37.0's minimum, enforced mechanically by its `apiElements`
`org.gradle.plugin.api-version = 9.0.0`. KGP's own minimum (7.6.3), Dokka's (7.6) and foojay's (7.6) are lower; KGP's deprecation
threshold (8.14.4) is also lower. The consumer's JVM must run Gradle 9 (JDK 17 to 27 per the compatibility matrix), and the 21
toolchain must be present or provisionable: Gradle's `JavaToolchainResolverRegistryInternal` is registered in `BuildServices`
(build-scoped), so an included build's own settings, with the foojay resolver in `settings.gradle.kts`, drive provisioning for its
projects (inference from service scope; not run).

**Still open after this analysis** (not removed by it): the runtime half (KGP 2.4.21 applying under Gradle 9.0.0 through nested
precompiled scripts and accessor generation), and the Kotlin compiler daemon when a 2.1.20 consumer and this 2.4.21 build share a JVM.
Both are covered by the smoke test in 6.6.

**Premise that must stay true.** A later bump of KGP, Dokka or vanniktech that puts a class with metadata ≥ 2.4 on the compile
classpath, or a `kotlin-stdlib` without the exclusion, would break the 9.0.0 promise silently. The permanent check in 6.6 guards it.

### F6. The stdlib pin: the KMP mechanics carry over; the `kotlin("jvm")` module takes an explicit `api` pin (consensus on mechanism; practice on effect)

- **KMP, unchanged by a multi-module build or by an empty module.** The reused findings
  ([`kmp-build-and-kotlin-compat.md`](https://github.com/SpartanLaboratories/GeneralTools/blob/d424aaf/docs/plans/7-kotlin-multiplatform/research/kmp-build-and-kotlin-compat.md)
  F1, F2, F5) stand: the pin is a dependency declared on a source-set configuration, so it is published whether or not the module
  has Kotlin source, and it does not depend on any other project. GeneralTools' own `build.gradle.kts` (branch
  `feat/issue-7-kotlin-multiplatform` @ `ebb2528`) is the working shape: `commonMain` `api(kotlin-stdlib:2.2.0)`, `jsMain`
  `api(kotlin-stdlib:$coreLibrariesVersion)`, `coreLibrariesVersion` untouched. In `stdlibDependencyManagement.kt` (KGP `v2.4.21`)
  `addStdlibDependency` skips a source set that already holds a stdlib dependency
  (`isStdlibAddedByUser`), and for KMP only adds the default to source sets with no `dependsOn`, which is why the web source set
  needs its own explicit `api` of the compiler-matching stdlib. With only `js` declared, use `jsMain` (or select by target platform
  type, as tapmoc does), not `webMain`, whose creation with a single web target was not established.
- **JVM module.** KGP adds the default stdlib as `api` of `main` (and `implementation` of `test`) at `coreLibrariesVersion`, skipping a
  source set where the user declared one. Two ways to publish `requires 2.2.0`:
  - *Explicit `api(kotlin-stdlib:2.2.0)`.* `main` publishes 2.2.0 and `test` keeps the default (2.4.21) for `kotlin-test`, so tests
    compile and run on the compiler's own library. Same mechanism and same catalog entry as the KMP module.
  - *`kotlin.coreLibrariesVersion = "2.2.0"`.* The property's KDoc: "The version of the core Kotlin libraries that are added to the
    Kotlin compile classpath, unless there is already a dependency added to this project", covering `kotlin-stdlib`, `kotlin-test`,
    `kotlin-dom-api-compat`, `kotlin-reflect`. It would move `kotlin-test` to 2.2.0 as well, differs from the KMP convention, and
    cannot be used in the KMP module (the web compilers reject a stdlib older than themselves, reused F5).
  - Evidence the explicit form is already the practice: MyGameTools' `gametools-core-5.1.0.module` (a `kotlin("jvm")` library) lists
    `org.jetbrains.kotlin:kotlin-stdlib` `requires 2.2.0` in both `apiElements` and `runtimeElements`.
- **What the tests do and do not prove.** Gradle takes the highest requested version, so `kotlin-test` 2.4.21 (and MockK's 2.2.21, and
  kotlin-logging's 2.1.21) put the test runtime on stdlib 2.4.21 even when `main` declares 2.2.0. The unit tests therefore never run
  on the floor stdlib. The floor is proven only by the declared metadata plus a consumer compile (reused pitfall 2).
- A `graphics-desktop` that takes `api(project(":graphics-core"))` still needs its own pin: without it KGP's default would publish
  `kotlin-stdlib:2.4.21` in the module's own dependency list.

### F7. The JVM level (consensus on mechanism; practice on choice)

- **Toolchain plus explicit target, both modules.** `jvmToolchain(21)` sets the `jvmTarget` convention, passes the toolchain's
  `-jdk-home` (so the compile classpath is JDK 21's class library) and runs tests on a real 21 (reused F9, KGP docs quotes). The
  explicit `jvmTarget = JVM_21` is redundant but makes the output level independent of the toolchain mechanism. `-Xjdk-release=21`
  (Jake Wharton's argument; tapmoc's `java(21)`) gives the same API restriction without requiring a JDK 21 on the machine and would
  let a consumer on JDK 23 compile this build without provisioning, but tests then run on the Gradle JVM (23 locally) rather than
  21, and there is still no typed DSL for it (KT-49746, Backlog). It stays the fallback.
- **What each module declares by itself.**
  - *`kotlin("jvm")` + `java-library`:* `org.gradle.jvm.version` comes from `JavaCompile`'s target, which is the `release` option if
    set, else `java.targetCompatibility` if set, else the toolchain compiler's language version
    (`JavaBasePlugin.configureCompileDefaults` at `v9.7.1`). KGP's KDoc says the same: "The Gradle Java compilation task
    `targetCompatibility` controls the value of the `org.gradle.jvm.version` attribute". With `jvmToolchain(21)` the published value is
    expected to be 21; MyGameTools' `gametools-core-5.1.0.module` shows 23 for its toolchain 23. Inference from source; confirm in the
    generated `.module` (6.5).
  - *KMP `jvm()`:* nothing. KGP `v2.4.21` never sets `TargetJvmVersion`, and Ktor 3.6.0, Coil 3.6.3, kotlinx-io, Okio and
    kotlinx-coroutines publish none; kotlinx-datetime and GeneralTools set it explicitly in `jvm { attributes { attribute(
    TargetJvmVersion.TARGET_JVM_VERSION_ATTRIBUTE, n) } }`. Set it to 21.
- **Resolution-time failure.** Gradle's attribute rule is "lower is compatible with higher, prefers highest compatible"
  (`variant_attributes.adoc`), so a consumer asking for 17 cannot select a variant that declares 21 and the build fails at
  dependency resolution instead of with `UnsupportedClassVersionError`. Android consumers are subject to the same rule (Android
  phase).

### F8. `-jvm-default` (consensus: compiler source)

- In 2.4.21 the Gradle DSL property `jvmDefault` has "Default value: null"; the effective mode is
  `if (supportsFeature(LanguageFeature.JvmDefaultEnableByDefault)) JvmDefaultMode.ENABLE else JvmDefaultMode.DISABLE`
  (`JvmDefaultMode.kt`), and `JvmDefaultEnableByDefault(KOTLIN_2_2, "KT-71768")` (`LanguageVersionSettings.kt`). So `enable`
  is the default **when the language version is ≥ 2.2**, which the floor guarantees. `enable` emits default methods plus `DefaultImpls`
  for compatibility with code compiled in `disable` mode, so a Kotlin 2.1.20 consumer (the game client) that implements a contract
  interface still compiles.
- Setting `jvmDefault = JvmDefaultMode.ENABLE` in the JVM compiler options costs one line, makes the published binary shape part of
  the build rather than of the compiler default, and protects against a later language version making `no-compatibility` the default
  when the floor is raised.

### F9. Hand-written pins against `com.gradleup.tapmoc` (practice; tapmoc facts from its source and release notes)

- tapmoc 0.5.0 (2026-10-13, the latest) sets `languageVersion`/`apiVersion` at the extension level, adds `api(kotlin-stdlib:<floor>)`
  for the `jvm`/`androidJvm`/`common` compilations and `api(kotlin-stdlib:<KGP version>)` for the others, and `java(n)` sets
  `-Xjdk-release` plus `jvmTarget` (`KgpImpl.kt`, `TapmocExtensionImpl.kt` at `v0.5.0`). That is exactly what the hand-written shape in
  6.3 does; `java()` would replace the toolchain-based JVM level and is not wanted. Okio uses it; Ktor hand-writes (`apiVersion =
  KOTLIN_2_2` in `ktorbuild.kmp`).
- Added value: `checkDependencies()`, which fails `check` if an `api` dependency exposes newer Kotlin metadata, a newer stdlib, or newer
  class files. That is the automated version of "the floor holds in what is published", but G1's only `api` inputs are this
  repository's own BOM and `graphics-core`, so there is nothing to catch before G2.
- Risks: 0.x; the 0.5.0 notes say the dependency-detection machinery changed ("acts on 'known' outgoing configuration names") and
  the package was renamed; its build uses KGP 2.3.0 (`kgp-latest` in `gradle/libs.versions.toml`), not 2.4.21; it is another plugin
  that must be hoisted into the root classloader (question 1) and kept at one version in `build-logic`. The number of pins does not
  grow with the number of modules, because both convention plugins call one helper.

### F10. Test and logging libraries (consensus on versions; the JUnit 6 pairing is API-level evidence)

Latest stable on Maven Central, 2026-10-20 (`maven-metadata.xml` `<release>`): `org.junit:junit-bom` and
`junit-jupiter`/`junit-platform-launcher` **6.1.3** (2026-08-07; 5.x ends at 5.14.4), `io.mockk:mockk` **1.14.11** (2026-05-29),
`ch.qos.logback:logback-classic` **1.6.5** (2026-09-30), `org.slf4j:slf4j-api` **2.0.20** (2026-09-22; 2.1.0 exists only as
alpha), `io.github.oshai:kotlin-logging` **8.0.4** (2026-05-27, unchanged since the reused research).

- **What `kotlin-test-junit5` 2.4.21 depends on.** Its `.pom`/`.module`: `kotlin-test` 2.4.21 and `junit-jupiter-api` 5.10.1 (api),
  `junit-jupiter-engine` 5.10.1 and `junit-platform-launcher` 1.10.1 (runtime). It is compiled against `junit-jupiter-api:5.0.0`
  (`compileOnly`) and uses only `Assertions.assertEquals/assertNotEquals/assertSame/assertNotSame/assertNotNull/assertNull/fail`
  and the annotations `Test`, `Disabled`, `BeforeEach`, `AfterEach` (`libraries/kotlin.test/junit5` at `v2.4.21`). I listed the
  public members of `junit-jupiter-api-6.1.3.jar` with `javap`: every one of those signatures is present (`assertEquals(Object,
  Object, String)`, `fail(String)`, `fail(String, Throwable)`, and so on). Kotlin's own build tests this module against Jupiter
  5.12.2 (`gradle/libs.versions.toml` at `v2.4.21`), not 6.
- **JUnit 6.** "Scope: Java 17 and Kotlin 2.2 baseline" ([JUnit 6.0.0 notes](https://docs.junit.org/6.0.0/release-notes/)), single version
  number across Platform, Jupiter and Vintage, which is why the BOM aligns `kotlin-test-junit5`'s 5.10.1/1.10.1 requests up to 6.1.3.
  Gradle 9.7.1 has JUnit 6 integration tests (`JUnit6JUnitIntegrationTest.groovy`, absent at 9.0.0). MockK's maintainers state
  "Current MockK versions work with projects that use JUnit 6" ([mockk/mockk#1446](https://github.com/mockk/mockk/issues/1446)).
  Peers: Ktor 3.6.0 runs JUnit 5.14.4 with MockK 1.14.11, logback 1.6.3 and slf4j 2.0.19.
- **Launcher on Gradle 9.** Gradle 8.2's upgrade guide: automatic loading of test framework dependencies from the distribution "is
  deprecated and will be removed in Gradle 9.0.0", with an explicit `testRuntimeOnly("org.junit.platform:junit-platform-launcher")`
  for JUnit 5 and later. So declare it, without a version, under the BOM.
- **Pairing of logback and slf4j.** logback-classic 1.6.5 is built for Java 11 and manages `slf4j-api` 2.0.19; `slf4j-api` 2.0.20 is
  the latest stable and Gradle picks the highest. kotlin-logging 8.0.4's `-jvm` artifact depends only on `kotlin-stdlib` 2.1.21 and
  declares no slf4j dependency (reused finding, re-read from the `.pom`), so the JVM source set supplies `slf4j-api`. As an
  `implementation` dependency it reaches consumers' runtime classpath without entering their compile classpath.
- **Floor safety.** MockK 1.14.11 declares `kotlin-stdlib` 2.2.21 (runtime); kotlin-logging 2.1.21. Both are runtime-only for
  consumers or test-only here. Jupiter, logback and slf4j are Java libraries.

### F11. `kotlin-dsl` imposes only what is already in question 1 (consensus)

Language and API version 2.2 for `build-logic` sources on every Gradle 9.0–9.8, Java 8 bytecode by default ("By default, the code will
target Java 8"), a warn-only JVM-target validation, no toolchain needed, and "Each Gradle release is meant to be used with a specific
version of the `kotlin-dsl` plugin" (so never version it). Gradle 9.8 changed its Kotlin compiler argument from `-Xjvm-default=all` to
`-jvm-default=enable`; irrelevant to this build. No further limit on the wrapper.

## 4. How comparable projects do it

| Project @ ref | Gradle | KGP | Dokka / vanniktech | Tests and logging | Floor and JVM pins |
| --- | --- | --- | --- | --- | --- |
| Ktor `3.6.0` | 9.7.1 | 2.3.21 | 2.2.0 / 0.37.0 (AGP 9.4.0) | JUnit 5.14.4, MockK 1.14.11, logback 1.6.3, slf4j 2.0.19 | hand-written: `apiVersion = KOTLIN_2_2`, `languageVersion = KOTLIN_2_3` at the extension; `jvmToolchain`; no `TargetJvmVersion` |
| Coil `3.6.3` | 9.7.1 | 2.4.10 | 2.2.0 / 0.37.0 | n/a read | n/a read |
| Okio `parent-3.18.2` | 9.6.1 | 2.2.21 | n/a read | n/a read | tapmoc `configureKotlinCompatibility`; JVM target 8 explicit |
| kotlinx-coroutines `1.11.0`; kotlinx-io `0.9.1` | 8.14.4; 8.9 | n/a read | n/a read | n/a read | Kotlin-team libraries trail on Gradle 8 |
| kotlinx-datetime `v0.8.0` | n/a | 2.1.20 | n/a | n/a | explicit `TargetJvmVersion = 8` in `jvm {}` |
| MyGameTools `dc5214f` | 9.7.1 | 2.2.0 | 2.2.0 / 0.36.0, foojay 1.0.0 | n/a | `kotlin("jvm")`, `kotlin-stdlib` 2.2.0 `requires` in the published `.module`; JVM 23 published as `org.gradle.jvm.version` |
| GeneralTools branch `feat/issue-7-kotlin-multiplatform` @ `ebb2528` | 9.0.0 | 2.4.21 | n/a / 0.37.0 | `kotlin("test")`, logback 1.5.13, slf4j 2.0.16 | the KMP shape in F6/F7 with `jvmFloor = 17` |
| vanniktech `main` (unreleased) | 9.8.1 | 2.4.21 | n/a | n/a | n/a |
| Kotlin `v2.4.21` (its own `kotlin.test` tests) | n/a | n/a | n/a | JUnit Jupiter 5.12.2 | n/a |

Pattern: libraries run a recent KGP and a recent Gradle patch, hand-write or tapmoc the floor, keep the web stdlib at the compiler's
version, and keep JUnit on the 5.14 line today (Ktor); a 6.x line is not yet visible in the comparable builds read.

## 5. Pitfalls and regrets

- **"Documented range" is a test statement.** Treating 9.7.0 as a hard ceiling would leave the build on a release with six known
  regressions; treating it as no ceiling would let a Gradle bump change behaviour under a KGP release that has not been run on it.
  The wrapper and KGP move together: when KGP's table gains 9.8, the wrapper follows.
- **The 9.7.x line is unpatched** against the October advisories (F3). A reader who sees "latest Gradle 9.8.2" and "wrapper 9.7.1" in
  one document will ask why; the Consideration records it.
- **Dokka 2.2.0 was never run by its authors on Gradle 9** (CI: 7.6.3 and 8.14.3). Evidence it works is the siblings (Ktor, Coil,
  MyGameTools). On 9.8 it prints a deprecation warning; do not turn deprecation warnings into errors.
- **`coreLibrariesVersion` lowered globally** breaks `compileKotlinJs` (reused F5). **A `webMain`-only web pin** may attach to nothing
  when only `js` is declared.
- **`jvmTest` does not prove the 2.2.0 floor** (F6). A consumer compile against the published artifacts does.
- **No `TargetJvmVersion` on KMP `jvm`** leaves JDK 17 consumers with `UnsupportedClassVersionError` at runtime instead of a
  resolution failure; the `kotlin("jvm")` module gets it for free only if the toolchain actually resolved to 21.
- **kotlin-test-junit5's own JUnit 5.10.1/1.10.1 requests** stay in the graph unless a BOM aligns them; without the BOM Gradle
  still picks the highest requested, but then the engine, API and launcher can come from different lines.
- **The 2.2 pin expires.** Kotlin 2.5.0-Beta1 deprecates language version 2.2 (reused F1); plan the floor bump together with the
  consumers' own upgrades. The `jvmDefault` line is part of that bump's review.
- **tapmoc's package rename in 0.5.0** (`tapmoc.*` → `com.gradleup.tapmoc.*`) shows the API is still moving.
- **KGP's AGP ceiling is 9.3.1** while AGP 9.4.x is current; the Android phase must decide between them (F4).
- **`kotlin-dsl` and logback in `build-logic` tests:** GeneralTools records that `kotlin-dsl`'s `gradleApi()` bundles an SLF4J provider
  that conflicts with logback on a test classpath; question 6 owns any test source set in `build-logic`.

## 6. Recommendation for this repository

Pin Gradle 9.7.1; keep every other version at the latest stable release that keeps the floor; hand-write the floor pins in one helper
shared by both convention plugins; state Gradle 9.0.0 as the composite-consumer floor and prove it. The reasoning in three sentences:
the wrapper must track the newest Gradle the Kotlin Gradle plugin has been run on, and 9.7.1 is that line's maintained patch and the
pairing two large KMP libraries already run; the floor is a property of what is published, so it is declared by the same three
settings in both modules and checked in the generated metadata; and the 9.0.0 promise is cheap to keep because the plugin jars on
`build-logic`'s compile classpath carry no metadata newer than 2.2.0.

### 6.1 `gradle/wrapper/gradle-wrapper.properties`

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionSha256Sum=acd53f1edaf02f1a8ff99879f8a34b302661a057d9b063ae9e35b552f804d20a
distributionUrl=https\://services.gradle.org/distributions/gradle-9.7.1-bin.zip
networkTimeout=10000
retries=0
retryBackOffMs=500
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

The key set is Gradle's own `gradle/wrapper/gradle-wrapper.properties` at tag `v9.7.1` plus the checksum line (the wrapper task writes
`distributionSha256Sum` when given `--gradle-distribution-sha256-sum`). Generate the jar and scripts once with
`gradle wrapper --gradle-version 9.7.1 --distribution-type bin --gradle-distribution-sha256-sum acd53f1e…d20a` (run twice so the jar
is from 9.7.1); the wrapper jar's published SHA-256 is `7a9ce74c…2c5d` for the CI wrapper validation of question 7. Use `-bin`, not
`-all`.

### 6.2 `gradle/libs.versions.toml`

```toml
[versions]
kotlin             = "2.4.21"   # compiler and Kotlin Gradle plugin
kotlin-floor       = "2.2.0"    # consumer floor: stdlib pin; language and API version 2.2 are derived from it
dokka              = "2.2.0"
vanniktech-publish = "0.37.0"
kotlin-logging     = "8.0.4"
slf4j              = "2.0.20"
logback            = "1.6.5"
junit              = "6.1.3"    # junit-bom: Jupiter and Platform share one version from 6.0
mockk              = "1.14.11"

[libraries]
kotlin-stdlib-floor     = { module = "org.jetbrains.kotlin:kotlin-stdlib", version.ref = "kotlin-floor" }
kotlin-logging          = { module = "io.github.oshai:kotlin-logging", version.ref = "kotlin-logging" }
slf4j-api               = { module = "org.slf4j:slf4j-api", version.ref = "slf4j" }
logback-classic         = { module = "ch.qos.logback:logback-classic", version.ref = "logback" }
junit-bom               = { module = "org.junit:junit-bom", version.ref = "junit" }
junit-jupiter           = { module = "org.junit.jupiter:junit-jupiter" }                  # version from junit-bom
junit-platform-launcher = { module = "org.junit.platform:junit-platform-launcher" }       # version from junit-bom
mockk                   = { module = "io.mockk:mockk", version.ref = "mockk" }

[plugins]
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
kotlin-jvm           = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
dokka                = { id = "org.jetbrains.dokka", version.ref = "dokka" }
vanniktech-publish   = { id = "com.vanniktech.maven.publish", version.ref = "vanniktech-publish" }
```

Not in the catalog: `kotlin-test` (the version follows the Kotlin Gradle plugin through `kotlin("test")`), the web stdlib (read from
`coreLibrariesVersion`), and the foojay resolver, a settings plugin that cannot read the catalog: `id("org.gradle.toolchains.foojay-
resolver-convention") version "1.0.0"` stays literal in `settings.gradle.kts` (question 1). The BOM project needs no entry. This
extends question 1's sketch (6.4 there) with the five libraries the conventions wire.

### 6.3 The pins, as typed build-logic code

`build-logic/src/main/kotlin/com/spartanlabs/graphics/build/SpartanBuild.kt` (extends question 1's constants):

```kotlin
package com.spartanlabs.graphics.build

import org.jetbrains.kotlin.gradle.dsl.JvmTarget

internal object SpartanBuild {
    const val JVM_LEVEL = 21
    val JVM_TARGET: JvmTarget = JvmTarget.JVM_21
    val TEST_LEVELS = listOf("gating", "component", "integration", "deterministic", "e2e", "nonfunctional", "uat")
}
```

`Floor.kt` in the same package (one helper both conventions call; `.set` rather than `=` keeps it independent of the assignment
plugin's reach into `.kt` files):

```kotlin
package com.spartanlabs.graphics.build

import org.jetbrains.kotlin.gradle.dsl.JvmDefaultMode
import org.jetbrains.kotlin.gradle.dsl.KotlinCommonCompilerOptions
import org.jetbrains.kotlin.gradle.dsl.KotlinJvmCompilerOptions
import org.jetbrains.kotlin.gradle.dsl.KotlinVersion

/** The consumer floor: metadata written and library API used both follow [floor] (2.2). */
internal fun KotlinCommonCompilerOptions.applyFloor(floor: KotlinVersion) {
    languageVersion.set(floor)
    apiVersion.set(floor)
}

/** JVM bytecode level plus the published default-method shape; `ENABLE` also emits `DefaultImpls` for Kotlin 2.1.x consumers. */
internal fun KotlinJvmCompilerOptions.applyJvmShape() {
    jvmTarget.set(SpartanBuild.JVM_TARGET)
    jvmDefault.set(JvmDefaultMode.ENABLE)
}
```

`spartangraphics.kmp-library.gradle.kts` (the floor and JVM part; the rest as in question 1, 6.9):

```kotlin
import com.spartanlabs.graphics.build.*
import org.gradle.api.attributes.java.TargetJvmVersion

kotlin {
    explicitApi()
    jvmToolchain(SpartanBuild.JVM_LEVEL)
    compilerOptions { applyFloor(catalog.kotlinFloor()) }          // extension level: all targets and shared source sets

    jvm {
        compilerOptions { applyJvmShape() }
        // A KMP jvm publication declares no org.gradle.jvm.version. Declare it, so a JDK 17 consumer fails at resolution.
        attributes { attribute(TargetJvmVersion.TARGET_JVM_VERSION_ATTRIBUTE, SpartanBuild.JVM_LEVEL) }
    }
    js { nodejs() }

    sourceSets {
        named("commonMain") {
            dependencies {
                api(catalog.library("kotlin-stdlib-floor"))      // published as `requires 2.2.0` in the jvm and metadata variants
                implementation(catalog.library("kotlin-logging"))
            }
        }
        // The web compilers reject a stdlib klib older than themselves; coreLibrariesVersion is NOT lowered.
        named("jsMain") { dependencies { api("org.jetbrains.kotlin:kotlin-stdlib:$coreLibrariesVersion") } }
        named("jvmMain") { dependencies { implementation(catalog.library("slf4j-api")) } }   // kotlin-logging-jvm brings no slf4j
        named("commonTest") { dependencies { implementation(kotlin("test")) } }
        named("jvmTest") {
            dependencies {
                implementation(project.dependencies.platform(catalog.library("junit-bom")))   // lifts kotlin-test-junit5's 5.10.1 requests
                implementation(catalog.library("junit-jupiter"))
                runtimeOnly(catalog.library("junit-platform-launcher"))                       // required explicitly on Gradle 9
                implementation(catalog.library("mockk"))
                runtimeOnly(catalog.library("logback-classic"))
            }
        }
    }
}

tasks.withType<Test>().configureEach { useJUnitPlatform() }        // kotlin-test then resolves to kotlin-test-junit5
```

`spartangraphics.jvm-library.gradle.kts` (the floor and JVM part):

```kotlin
import com.spartanlabs.graphics.build.*

plugins {
    `java-library`                                                   // `api` configuration; infers org.gradle.jvm.version
    id("org.jetbrains.kotlin.jvm")
    id("spartangraphics.publishing")
}

kotlin {
    explicitApi()
    jvmToolchain(SpartanBuild.JVM_LEVEL)                             // also sets the Java toolchain, which sets TargetJvmVersion
    compilerOptions {
        applyFloor(catalog.kotlinFloor())
        applyJvmShape()                                              // KotlinJvmProjectExtension.compilerOptions is KotlinJvmCompilerOptions
    }
}

dependencies {
    api(catalog.library("kotlin-stdlib-floor"))                      // coreLibrariesVersion stays at the compiler's version
    implementation(catalog.library("kotlin-logging"))
    implementation(catalog.library("slf4j-api"))
    testImplementation(kotlin("test"))
    testImplementation(platform(catalog.library("junit-bom")))
    testImplementation(catalog.library("junit-jupiter"))
    testRuntimeOnly(catalog.library("junit-platform-launcher"))
    testImplementation(catalog.library("mockk"))
    testRuntimeOnly(catalog.library("logback-classic"))
}

tasks.withType<Test>().configureEach { useJUnitPlatform() }
```

These snippets were not executed. `catalog.library(...)` and `catalog.kotlinFloor()` are question 1's `Catalog.kt`. `api(platform(graphics-bom))`
(question 5) and `configureTestLevels()` (question 1) are omitted here.

### 6.4 What stays out

No `kotlin.coreLibrariesVersion` assignment, no `-Xjdk-release`, no `tapmoc`, no `-Xskip-metadata-version-check` in `build-logic`,
no `kotlin("jvm")` or Kotlin Gradle plugin version applied to `build-logic`.

### 6.5 What the implementation checks in the generated metadata (Level 1, then question 6)

Run `generateMetadataFileFor…Publication` (or read each module's `build/publications/*/module.json`) and assert:

1. In `graphics-core`: `jvmApiElements-published` and `jvmRuntimeElements-published` declare `org.gradle.jvm.version: 21` and
   `kotlin-stdlib` `requires 2.2.0`; the root metadata variants require `kotlin-stdlib` 2.2.0; the `js` variants require the KGP
   version (2.4.21) and may also list 2.2.0.
2. In `graphics-desktop`: `apiElements` and `runtimeElements` declare `org.gradle.jvm.version: 21` and `kotlin-stdlib` `requires 2.2.0`
   (and `graphics-core`).
3. `--debug` output for a compile task shows `-language-version 2.2 -api-version 2.2` and `-jvm-default=enable`.
4. A `kotlin("jvm")` 2.1.20 consumer on JDK 21 compiles against the substituted modules; a consumer on a JDK 17 toolchain fails at
   resolution with a "compatible with Java 21" message.

### 6.6 The composite-consumer floor and how to make it certain

State in `CONTRIBUTING.md`: *Consumers include this build with Gradle 9.0.0 or newer (vanniktech 0.37.0's minimum), running on JDK 17
or newer; a JDK 21 toolchain is provisioned through this build's foojay resolver. The repository builds on Gradle 9.7.1.* Make it
true with two checks:

- **Cheap and permanent:** one CI job (or a script run at each plugin bump) that runs `gradle -p build-logic assemble` with a Gradle
  9.0.0 distribution (for example `gradle/actions/setup-gradle` with `gradle-version: 9.0.0`). It compiles the precompiled script plugins
  with the embedded Kotlin 2.2.0 and generates their accessors, which applies the plugins, so it exercises both the metadata and the
  plugin-application halves. It is the only step that fails if a plugin bump breaks the 9.0.0 floor.
- **One-off, before G1 closes (question 1, 6.10 step 3):** a throwaway copy of `GameGraphics` (Gradle 9.0.0 wrapper, KGP 2.1.20,
  JDK 23) with `includeBuild("../SpartanGraphics")`, depending on `io.github.spartanlaboratories:graphics-desktop`. If it fails with an
  incompatible-metadata error, fall back to stating the wrapper line (9.7+) as the floor, since the compile then needs an embedded
  Kotlin ≥ 2.4, and record that in `CONTRIBUTING.md`; do not add `-Xskip-metadata-version-check` to the model build.

Findings of question 1 this answer changes: its open item C1 (see F5 above) now has a static answer in favour of 9.0.0, and its
note that "KGP's own module file lists no direct stdlib dependency" is completed (every dependency edge excludes it). Nothing in
the reused GeneralTools findings is contradicted; the slf4j (2.0.16 → 2.0.20), logback (1.5.13 → 1.6.5) and JVM level (17 → 21)
values differ because they are newer or because this repository's floor is higher.

## Considerations

**C1. Gradle 9.7.1 sits inside the range of three unpatched high-severity advisories; 9.8.1 and later are patched.**
The wrapper rule (newest Gradle inside every plugin's documented range) yields 9.7.x, and the advisories (F3) affect "9.0.0 to 9.8.0".
- *(a) 9.7.1* — inside KGP's documented range and the pairing Ktor and Coil run; exposes the maintainer's machine to the daemon
  advisories until the wrapper moves, and no fixed 9.7.x release exists.
- *(b) 9.8.2* — patched; outside KGP's "fully supported" label (KGP says newer Gradle is usable with possible deprecation warnings;
  Dokka 2.2.0 warns on `setVisible`), no release-build evidence was read for Dokka 2.2.0 with vanniktech 0.37.0 on 9.8, and
  9.8.2 is one day old.
- *(c) 9.7.0* — the literal ceiling; strictly worse than (a).
Recommendation: **(a) for G1**, and re-decide when KGP's compatibility table first lists a 9.8 or later Gradle, or before the by-hand
release at G7 (the first time a signing key and a Central Portal token are in the maintainer's environment), whichever comes first.
If the maintainer prefers the patched line now, the change is the three lines of `gradle-wrapper.properties` (URL
`gradle-9.8.2-bin.zip`, checksum `21bd5da2…9607`) and nothing else in this document changes.

**C2. JUnit Jupiter 6.1.3 or 5.14.4.**
- *(a) 6.1.3* — latest stable; Java 17 and Kotlin 2.2 baselines match this build; API-compatible with everything `kotlin-test-junit5`
  2.4.21 calls; Gradle 9.7.1 and MockK document JUnit 6 support. Not exercised by Kotlin's own tests (5.12.2) or by the peers read
  (Ktor: 5.14.4).
- *(b) 5.14.4* — the line `kotlin-test-junit5` is built and tested against and the peers use; the same artifact ids and the same
  BOM, so moving is one catalog line (`junit = "5.14.4"`).
Recommendation: **(a)**, and fall back to (b) if the first `jvmTest` run in G2 shows any `kotlin.test` assertion or launcher
incompatibility. G1 has no Kotlin source, so no test exercises this pairing before G2.

**C3. Permanent Gradle 9.0.0 job.** Keeping a CI job on 9.0.0 costs about a minute per run and ties the floor to the lowest Gradle any
applied plugin supports (today vanniktech's). *(a)* keep it (recommended: it is the only guard on the floor, and a failure points at
the plugin bump that caused it); *(b)* run it by hand at each plugin bump; *(c)* state the wrapper line as the floor and drop the job.
Recommendation: (a). It sits with question 7's CI design.

## Known gaps

- **No execution.** Nothing here was built or run. The static analysis in F5 is complete for the compile classpath; the runtime half
  of the 9.0.0 floor (KGP 2.4.21 under Gradle 9.0.0 through the nested `build-logic`, and a Kotlin compiler daemon shared by a 2.1.20
  consumer and this 2.4.21 build) is unverified until 6.6 runs.
- **JUnit 6.1.3 with `kotlin-test-junit5` 2.4.21** is established by signatures and by two third-party statements, not by a test run.
  MockK's JUnit 5 extension class (`MockKExtension`) with JUnit 6 is untested by me; only plain `mockk()` use is expected in G1/G2.
- **Dokka 2.2.0 and vanniktech 0.37.0 deprecation profile on Gradle 9.7.1 and 9.8.x** under `--warning-mode all` was not measured. The
  Dokka `setVisible` warning is documented; vanniktech's use of the 9.8-deprecated signing APIs is unknown.
- **`org.gradle.jvm.version` of the `kotlin("jvm")` module** is derived from `JavaBasePlugin` source and the toolchain; no generated
  `.module` was read. Check 6.5 settles it.
- **A module with no source:** the stdlib pin and `TargetJvmVersion` are configuration-level and independent of source (F6), but
  what the empty `js` klib and empty jars look like in the publication was not established (questions 4 and 5).
- **Toolchain provisioning inside an included build** follows from the build-scoped `JavaToolchainResolverRegistryInternal` service; no
  test or documentation on included builds was found.
- **`webMain` with a single web target** (creation under the default hierarchy template) was not established; the recommendation avoids
  it by using `jsMain`.
- **tapmoc 0.5.0 against KGP 2.4.21** was not exercised; the recommendation does not depend on it.
- **Gradle 9.8.2 is one day old;** no field reports were searched beyond the issue tracker queries for Dokka, vanniktech and KGP
  ("Gradle 9.8"), which returned only the Dokka `setVisible` report.
- **AGP 9.4.x against KGP 2.4.21** is outside KGP's documented AGP range (F4); the Android phase has to settle it.
- The wrapper properties key set (`retries`, `retryBackOffMs`) is copied from Gradle's own file at `v9.7.1`; the wrapper task was not run.

## Load-bearing claims

Quotes from AsciiDoc, Kotlin or JSON sources omit markup characters only; page text is as extracted from the rendered page.

1. **KGP 2.4.21's documented Gradle maximum is 9.7.0 and newer Gradle is allowed.** "Similarly, the maximum fully supported version is
   9.7.0." and "You can also use Gradle and AGP versions up to the latest releases, but if you do, keep in mind that you might
   encounter deprecation warnings or some new features might not work." The same page's table row reads "2.4.20–2.4.21 7.6.3–9.7.0
   8.5.2–9.3.1". [Kotlin docs, Configure a Gradle project](https://kotlinlang.org/docs/gradle-configure-project.html), read 2026-10-20.
2. **Kotlin 2.4.20 added Gradle 9.7.0 support and allows newer Gradle.** "Kotlin 2.4.20 is fully compatible with Gradle 7.6.3 through
   9.7.0. You can also use Gradle versions up to the latest Gradle release. However, be aware that doing so may result in deprecation
   warnings, and some new Gradle features might not work." [What's new in Kotlin 2.4.20](https://kotlinlang.org/docs/whatsnew2420.html),
   read 2026-10-20.
3. **KGP checks only a minimum.** `GradleCompatibilityCheck`: `internal const val minSupportedGradleVersionString = "7.6.3"`,
   `internal const val nextMinimumSupportedGradleVersionString = "8.14.4"`, `if (currentVersion < minSupportedGradleVersion) {` …
   `} else if (currentVersion < nextMinimumSupportedGradleVersion) {` … `DeprecatedGradleVersionWarning`. No comparison above.
   [GradleCompatibilityCheck.kt @ v2.4.21](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/internal/diagnostics/GradleCompatibilityCheck.kt).
4. **KGP's integration tests top out at the literal 9.7.0.** `TestVersions.Gradle`: `const val G_9_7 = "9.7.0"` … `const val MIN_SUPPORTED = MINIMALLY_SUPPORTED_GRADLE_VERSION`
   `const val MAX_SUPPORTED = G_9_7`; AGP: `const val MAX_SUPPORTED = AGP_93 // Update once the Gradle MAX_SUPPORTED version is bumped`.
   [TestVersions.kt @ v2.4.21](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-integration-tests/src/test/kotlin/org/jetbrains/kotlin/gradle/testbase/TestVersions.kt).
5. **A Gradle patch release replaces the final release.** "The Gradle team may create additional patch releases to replace the final
   release due to critical bug fixes or regressions. For instance, Gradle 5.2.1 replaces the Gradle 5.2 release."
   [feature_lifecycle.adoc @ v9.8.2](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/releases/feature_lifecycle.adoc).
6. **Gradle 9.7.1 is the recommended patch of 9.7.0.** "Gradle 9.7.1 is the first patch release for Gradle 9.7.0." … "We recommend upgrading
   to Gradle 9.7.1." (fixes include "Gradle 9.7.0 expects Option annotation arguments in a different order than before in
   build.gradle.kts"). [9.7.1 release notes](https://docs.gradle.org/9.7.1/release-notes.html), read 2026-10-20.
7. **vanniktech 0.37.0's minimums and tested-up-to list.** "#### Minimum supported versions - JDK 17 - Gradle 9.0.0 - Android Gradle
   Plugin 8.13.0 - Kotlin Gradle Plugin 2.2.0" and "#### Compatibility tested up to - JDK 26 - Gradle 9.6.0 - Gradle 9.7.0-milestone-1 -
   Android Gradle Plugin 9.2.1 - Android Gradle Plugin 9.3.0-rc01 - Android Gradle Plugin 9.4.0-alpha01 - Kotlin Gradle Plugin 2.4.0".
   [CHANGELOG.md @ 0.37.0](https://github.com/vanniktech/gradle-maven-publish-plugin/blob/0.37.0/CHANGELOG.md).
8. **vanniktech 0.37.0's API variant requires plugin API version 9.0.0 and has no dependencies.** `.module`, variant `apiElements`:
   `"org.gradle.plugin.api-version": "9.0.0"`, `"org.gradle.usage": "java-api"`, no `dependencies` array; its `runtimeElements`
   depends on `com.vanniktech:central-portal` 0.37.0.
   [gradle-maven-publish-plugin-0.37.0.module](https://repo1.maven.org/maven2/com/vanniktech/gradle-maven-publish-plugin/0.37.0/gradle-maven-publish-plugin-0.37.0.module), read 2026-10-20.
9. **Dokka's documented minimum Gradle and its test matrix.** "Supported versions Ensure that your project meets the minimum version
   requirements: Tool Version Gradle 7.6 or higher". [Dokka Gradle docs](https://kotlinlang.org/docs/dokka-gradle.html), page showing 2.2.0,
   read 2026-10-20. `TestedVersions`: `val LATEST = BuildVersions("8.14.3", "2.3.0")` and `gradleVersions = listOf("7.6.3")`.
   [TestedVersions.kt @ v2.2.0](https://github.com/Kotlin/dokka/blob/v2.2.0/dokka-integration-tests/gradle/src/main/kotlin/org/jetbrains/dokka/it/gradle/TestedVersions.kt).
10. **foojay resolver: latest 1.0.0 and its minimum.** "Requires Gradle **7.6 or later** to work." [README @ foojay-toolchains-plugin-1.0.0](https://github.com/gradle/foojay-toolchains/blob/foojay-toolchains-plugin-1.0.0/README.md);
    Plugin Portal metadata `<release>1.0.0</release>`, `<lastUpdated>20250519164405</lastUpdated>`
    ([maven-metadata.xml](https://plugins.gradle.org/m2/org/gradle/toolchains/foojay-resolver-convention/org.gradle.toolchains.foojay-resolver-convention.gradle.plugin/maven-metadata.xml)), read 2026-10-20.
11. **Gradle 9.0.0 through 9.8.0 are affected by three high advisories, patched in 9.8.1.** GHSA-mvvg-497x-hmj8 "Unauthenticated
    worker-to-daemon channel deserializes untrusted Java objects", GHSA-xwqc-3h47-hg64 "Deserialization of untrusted Java objects
    before authentication in the client - daemon communication", GHSA-j5m7-59rp-24f5 "Failure to disable repositories failing to
    establish an SSL connection can expose builds to malicious artifacts"; vulnerable range "< 8.14.6, 9.0.0 to 9.8.0", patched
    "8.14.6, 9.8.1" (REST API `repos/gradle/gradle/security-advisories`, published 2026-10-07).
    [GHSA-mvvg-497x-hmj8](https://github.com/gradle/gradle/security/advisories/GHSA-mvvg-497x-hmj8). The 9.8.2 notes: "This release addresses
    three high rated vulnerabilities" ([9.8.2 release notes](https://docs.gradle.org/9.8.2/release-notes.html)).
12. **Dokka 2.2.0 triggers a Gradle 9.8 deprecation.** "The Configuration.setVisible(boolean) method has been deprecated. This is scheduled to
    be removed in Gradle 11." [Kotlin/dokka#4600](https://github.com/Kotlin/dokka/issues/4600); fix PR "Drop usages of `Configuration.isVisible`
    on Gradle 9+ as it's no-op there, as well as it's deprecated for removal" merged to `master` 2026-09-11
    ([#4601](https://github.com/Kotlin/dokka/pull/4601)); latest Dokka release 2.2.0 (2026-03-26), 2.3.0-Beta (2026-09-15) is a pre-release.
13. **Embedded Kotlin per Gradle release.** Table rows `| 2.2.0  | 9.0.0 | 2.2`, `| 2.4.0  | 9.7.0 | 2.2`, `| 2.4.10 | 9.8.0 | 2.2`.
    [compatibility.adoc @ v9.8.2](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/releases/compatibility.adoc).
    Confirmed by listing the downloaded `gradle-9.0.0-bin.zip` (`lib/kotlin-compiler-embeddable-2.2.0.jar`, `lib/kotlin-stdlib-2.2.0.jar`)
    and `gradle-9.7.1-bin.zip` (`-2.4.0`), 2026-10-20.
14. **A compiler reads metadata up to max(X+1, Y).** `MetadataVersion.lastSupportedVersionWithThisLanguageVersion`: "Compiler of
    deployVersion X (INSTANCE) with LV Y (metadataVersionFromLanguageVersion) can read metadata with version <= max(X+1, Y)".
    [MetadataVersion.kt @ v2.2.0](https://github.com/JetBrains/kotlin/blob/v2.2.0/core/metadata/src/org/jetbrains/kotlin/metadata/deserialization/MetadataVersion.kt).
15. **KGP 2.4.21 excludes `kotlin-stdlib` on every dependency edge.** `kotlin-gradle-plugin-2.4.21.module`: each variant's dependencies
    (for example `gradle813ApiElements`: `kotlin-gradle-plugins-bom`, `kotlin-gradle-plugin-api`, `fus-statistics-gradle-plugin`)
    carry `"excludes": [ { "group": "org.jetbrains.kotlin", "module": "kotlin-reflect" }, …, { "module": "kotlin-stdlib" }, … ]`;
    the same holds for `kotlin-gradle-plugin-api-2.4.21.module` and `dokka-gradle-plugin-2.2.0.module`. Scan result: zero KGP
    dependencies without the stdlib exclusion.
    [kotlin-gradle-plugin-2.4.21.module](https://repo1.maven.org/maven2/org/jetbrains/kotlin/kotlin-gradle-plugin/2.4.21/kotlin-gradle-plugin-2.4.21.module), read 2026-10-20.
16. **Class-file metadata of the plugin jars** (measurement, 2026-10-20, by scanning every `.class` for `kotlin.Metadata` `mv`): maximum 2.2.0
    in `kotlin-gradle-plugin-2.4.21-gradle813.jar` and `gradle-maven-publish-plugin-0.37.0.jar`; 1.8.0 in the KGP API-side jars; 1.4.3
    and 1.8.0 in the Dokka jars; 2.4.0 in `kotlin-stdlib-2.4.21.jar`. Jars from `https://repo1.maven.org/maven2/` under
    `org/jetbrains/kotlin/…/2.4.21/`, `org/jetbrains/dokka/…/2.2.0/`, `com/vanniktech/gradle-maven-publish-plugin/0.37.0/`.
17. **`kotlin-dsl` compiles build logic at language and API 2.2 with no metadata-skip flag.** `KotlinDslCompilerPlugins`:
    `apiVersion.set(KotlinVersion.KOTLIN_2_2)`, `languageVersion.set(KotlinVersion.KOTLIN_2_2)`,
    `freeCompilerArgs.addAll(KotlinDslPluginSupport.kotlinCompilerArgs)`; `kotlinCompilerArgs` at `v9.0.0`: `"-java-parameters"`,
    `"-Xjvm-default=all"`, `"-Xjsr305=strict"`, `"-Xjspecify-annotations=strict"`, `"-Xsam-conversions=class"`, `"-Xuse-fir-lt=false"`.
    [KotlinDslCompilerPlugins.kt @ v9.0.0](https://github.com/gradle/gradle/blob/v9.0.0/platforms/core-configuration/kotlin-dsl-plugins/src/main/kotlin/org/gradle/kotlin/dsl/plugins/dsl/KotlinDslCompilerPlugins.kt),
    [KotlinDslPluginSupport.kt @ v9.0.0](https://github.com/gradle/gradle/blob/v9.0.0/platforms/core-configuration/kotlin-dsl/src/main/kotlin/org/gradle/kotlin/dsl/provider/KotlinDslPluginSupport.kt).
18. **`kotlin-dsl`'s version coupling and Java target.** "Each Gradle release is meant to be used with a specific version of the `kotlin-dsl`
    plugin." and "The Kotlin DSL Plugin leverages Java Toolchains. By default, the code will target Java 8."
    [kotlin_dsl.adoc @ v9.7.1](https://github.com/gradle/gradle/blob/v9.7.1/platforms/documentation/docs/src/docs/userguide/reference/dsl-apis/kotlin_dsl.adoc).
19. **AGP 9.4's minimum Gradle.** "The maximum API level that Android Gradle plugin 9.4 supports is API level 37. Here is other
    compatibility info: Minimum version Default version Notes Gradle 9.6.0 9.6.0".
    [AGP 9.4 release notes](https://developer.android.com/build/releases/gradle-plugin), read 2026-10-20.
20. **`jvmDefault` default follows the language version.** `val LanguageVersionSettings.jvmDefaultMode … ?: if
    (supportsFeature(LanguageFeature.JvmDefaultEnableByDefault)) JvmDefaultMode.ENABLE else JvmDefaultMode.DISABLE`
    ([JvmDefaultMode.kt @ v2.4.21](https://github.com/JetBrains/kotlin/blob/v2.4.21/compiler/config.jvm/src/org/jetbrains/kotlin/config/JvmDefaultMode.kt));
    `JvmDefaultEnableByDefault(KOTLIN_2_2, "KT-71768"),`
    ([LanguageVersionSettings.kt @ v2.4.21](https://github.com/JetBrains/kotlin/blob/v2.4.21/core/language.version-settings/src/org/jetbrains/kotlin/config/LanguageVersionSettings.kt));
    DSL KDoc: "`-jvm-default=enable` … This is the default behavior since language version 2.2."
    ([KotlinJvmCompilerOptions.kt @ v2.4.21](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/KotlinJvmCompilerOptions.kt)).
21. **The default stdlib is skipped where the user declares one.** `isStdlibAddedByUser(configurations, stdlibModules - KOTLIN_STDLIB_COMMON_MODULE_NAME, kotlinSourceSet)`
    → `return@withDependencies`; and "Since 1.9.20 in MPP projects, we should add stdlib only for common dependencies except standalone
    compilations". [stdlibDependencyManagement.kt @ v2.4.21](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/internal/stdlibDependencyManagement.kt).
22. **`coreLibrariesVersion` covers more than the stdlib.** "The version of the core Kotlin libraries that are added to the Kotlin compile
    classpath, unless there is already a dependency added to this project. The core Kotlin libraries are: - 'kotlin-stdlib' - 'kotlin-test'
    - 'kotlin-dom-api-compat' - 'kotlin-reflect'". [kotlinTopLevelExtensionConfig.kt @ v2.4.21](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/kotlinTopLevelExtensionConfig.kt).
23. **The Java compilation target decides `org.gradle.jvm.version`.** "The Gradle Java compilation task
    [org.gradle.api.tasks.compile.JavaCompile.targetCompatibility] controls the value of the `org.gradle.jvm.version` attribute which itself
    controls the produced artifact's minimum supported JVM version via Gradle Module Metadata."
    [KotlinTaskConfigs.kt @ v2.4.21](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/tasks/KotlinTaskConfigs.kt).
    `JavaBasePlugin.configureCompileDefaults`: `if (javaCompile.getOptions().getRelease().isPresent()) { return
    JavaVersion.toVersion(javaCompile.getOptions().getRelease().get()); }` … `if (rawConvention != null) { return rawConvention; }` …
    `return JavaVersion.toVersion(javaCompile.getJavaCompiler().get().getMetadata().getLanguageVersion().toString());`
    [JavaBasePlugin.java @ v9.7.1](https://github.com/gradle/gradle/blob/v9.7.1/platforms/jvm/plugins-java-base/src/main/java/org/gradle/api/plugins/JavaBasePlugin.java).
24. **`TargetJvmVersion` semantics.** "Indicates the JVM version compatibility. … Defaults to the JVM version used by Gradle, lower is
    compatible with higher, prefers highest compatible."
    [variant_attributes.adoc @ v9.7.1](https://github.com/gradle/gradle/blob/v9.7.1/platforms/documentation/docs/src/docs/userguide/reference/dependency-management/controlling-dependency-resolution/variant_attributes.adoc).
25. **tapmoc's pins and checks.** `KgpImpl.kotlinCompatibility`: `api("org.jetbrains.kotlin:kotlin-stdlib:${version}")` for
    `KotlinPlatformType.jvm, androidJvm, common`, else `kotlin-stdlib:${kgpVersion}` with the comment "Non-JVM targets do not support
    compatibility flags and require the latest version of kotlin-stdlib"; `javaCompatibility`: `freeCompilerArgs.add("-Xjdk-release=${version}")`.
    [KgpImpl.kt @ v0.5.0](https://github.com/GradleUp/tapmoc/blob/v0.5.0/tapmoc-gradle-plugin/src/kgp/kotlin/com/gradleup/tapmoc/internal/KgpImpl.kt).
    Requirements: "Gradle 8.3+ … For Kotlin: Kotlin Gradle Plugin 1.9.0+"
    ([index.mdx @ v0.5.0](https://github.com/GradleUp/tapmoc/blob/v0.5.0/docs/src/content/docs/index.mdx)); 0.5.0 "changes the internal machinery
    that tried to auto-detect the api/runtime outgoing configuration" and the package rename ([release v0.5.0](https://github.com/GradleUp/tapmoc/releases/tag/v0.5.0), 2026-10-13);
    `kgp-latest = "2.3.0"` ([gradle/libs.versions.toml @ v0.5.0](https://github.com/GradleUp/tapmoc/blob/v0.5.0/gradle/libs.versions.toml)).
26. **`kotlin-test-junit5` 2.4.21's dependencies and use of JUnit.** `.pom`: `junit-jupiter-api` `5.10.1` (compile), `junit-jupiter-engine`
    `5.10.1` and `junit-platform-launcher` `1.10.1` (runtime); `libraries/kotlin.test/build.gradle.kts`: `compileOnly("org.junit.jupiter:junit-jupiter-api:5.0.0")`;
    `JUnitSupport.kt`: `Assertions.assertEquals(expected, actual, message)`, `Assertions.fail<Any?>(message, cause)`, and `Annotations.kt`:
    `public actual typealias Test = org.junit.jupiter.api.Test`, `Disabled`, `BeforeEach`, `AfterEach`.
    [kotlin-test-junit5-2.4.21.pom](https://repo1.maven.org/maven2/org/jetbrains/kotlin/kotlin-test-junit5/2.4.21/kotlin-test-junit5-2.4.21.pom),
    [JUnitSupport.kt @ v2.4.21](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/kotlin.test/junit5/src/main/kotlin/JUnitSupport.kt).
27. **JUnit 6's baseline and version scheme.** "6.0.0 Date of Release: September 30, 2025 Scope: Java 17 and Kotlin 2.2 baseline Single version
    number for Platform, Jupiter, and Vintage". [JUnit 6.0.0 release notes](https://docs.junit.org/6.0.0/release-notes/), read 2026-10-20.
    `javap` of `junit-jupiter-api-6.1.3.jar` lists `public static <V> V fail(java.lang.String, java.lang.Throwable)` and
    `assertEquals(java.lang.Object, java.lang.Object, java.lang.String)` ([jar](https://repo1.maven.org/maven2/org/junit/jupiter/junit-jupiter-api/6.1.3/junit-jupiter-api-6.1.3.jar)).
28. **MockK works with JUnit 6.** "Current MockK versions work with projects that use JUnit 6." [mockk/mockk#1446](https://github.com/mockk/mockk/issues/1446),
    open, 2026-01-17. Gradle's JUnit 6 integration tests exist at 9.7.1:
    `platforms/jvm/testing-jvm/src/integTest/groovy/org/gradle/testing/junit/junit6/jupiter/JUnit6JUnitIntegrationTest.groovy` ([v9.7.1](https://github.com/gradle/gradle/tree/v9.7.1/platforms/jvm/testing-jvm/src/integTest/groovy/org/gradle/testing/junit/junit6/jupiter)).
29. **Gradle 9 requires the JUnit Platform launcher on the test runtime classpath.** "In some cases, Gradle will load JVM test framework
    dependencies from the Gradle distribution to execute tests. … this behavior is deprecated and will be removed in Gradle 9.0.0." and "If
    using JUnit 5, an explicit `runtimeOnly` dependency on `junit-platform-launcher` is required".
    [upgrading_version_8.adoc @ v9.8.2](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/releases/upgrading/upgrading_version_8.adoc).
30. **Latest stable releases on Maven Central (2026-10-20).** `junit-bom` `<release>6.1.3</release>`; `mockk` `<release>1.14.11</release>`
    (`lastUpdated` 20260529130608); `logback-classic` `<release>1.6.5</release>` (20260930194727) with `jdk.version` 11 and `slf4j.version` 2.0.19 in
    `logback-parent-1.6.5.pom`; `slf4j-api` versions `…2.0.19, 2.0.20, 2.1.0-alpha0, 2.1.0-alpha1`; `kotlin-logging` `<release>8.0.4</release>`;
    `kotlin-gradle-plugin` latest stable 2.4.21 (2.5.0-Beta1 is the only newer). `https://repo1.maven.org/maven2/<group path>/<artifact>/maven-metadata.xml`.
31. **Peers' pairings.** Ktor `3.6.0` `gradle/libs.versions.toml`: `kotlin = "2.3.21"`, `dokka = "2.2.0"`, `junit = "5.14.4"`, `slf4j = "2.0.19"`,
    `logback = "1.6.3"`, `mockk = "1.14.11"`, `mavenPublishing = "0.37.0"`, `android-gradlePlugin = "9.4.0"`; `gradle-wrapper.properties`
    `gradle-9.7.1-bin.zip` ([Ktor 3.6.0](https://github.com/ktorio/ktor/tree/3.6.0)). Coil `3.6.3`: `kotlin = "2.4.10"`, `dokka = "2.2.0"`,
    `gradlePlugin-mavenPublish = "com.vanniktech:gradle-maven-publish-plugin:0.37.0"`, Gradle 9.7.1 ([Coil 3.6.3](https://github.com/coil-kt/coil/tree/3.6.3)).
32. **The reused stdlib and `TargetJvmVersion` mechanics.** "On the JVM, consumers can use any compiler version from the previous language version
    onwards" ([API guidelines](https://kotlinlang.org/docs/api-guidelines-backward-compatibility.html)); the KMP `jvm` publication declares no
    `org.gradle.jvm.version` unless set, and web compilations reject an older stdlib klib: see
    [`kmp-build-and-kotlin-compat.md`](https://github.com/SpartanLaboratories/GeneralTools/blob/d424aaf/docs/plans/7-kotlin-multiplatform/research/kmp-build-and-kotlin-compat.md)
    F1, F5 and F10 at `d424aaf` (verified there 2026-10-18/19). Re-checked here: the `v2.4.21` search for `TargetJvmVersion` in the Kotlin
    Gradle plugin sources returns only KDoc in `KotlinTaskConfigs.kt` and build conventions for the plugin's own jars.
