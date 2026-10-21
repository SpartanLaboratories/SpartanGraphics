# Strict explicit API and the Kotlin Gradle plugin's ABI validation across the G1 modules

- **Issue:** [SpartanLaboratories/SpartanGraphics#3](https://github.com/SpartanLaboratories/SpartanGraphics/issues/3), question 3 of the
  [research README](README.md).
- **Read:** 2026-10-20. Sources are pinned to Kotlin `v2.4.21` (commit `9180ce6a7a`, released 2026-10-08, the latest stable; `v2.5.0-Beta1`
  was also read), binary-compatibility-validator `0.18.2` (`534b4eb256`, 2026-09-02), Ktor `3.6.0`, Coil `3.6.3`, Arrow `2.2.3`,
  kotlinx.coroutines `1.11.0`, kotlinx-io `0.9.1`, Okio `parent-3.18.2`, `kotlin-web-site` `master` (`5d234526eb`, 2026-10-20),
  and GeneralTools branch `feat/issue-7-kotlin-multiplatform` (`5eef2dc`) with its research at `d424aaf`.
- **Method limit:** nothing was built or run. Every behaviour below comes from the plugin's source at `v2.4.21`, its own integration
  tests at that tag (which exercise the empty-module cases), YouTrack items, the documentation, and GeneralTools' recorded
  observations on Gradle 9.0.0. The convention-plugin code below has not been compiled; *What the implementation must verify*
  lists the runs that settle it.

## 1. Question

How do strict `explicitApi()` and the Kotlin Gradle plugin's (KGP's) ABI validation behave across the Kotlin Multiplatform module
`graphics-core` (`jvm()` and `js { nodejs() }`, no Kotlin source), the `kotlin("jvm")` + `java-library` module `graphics-desktop` (no
source) and the Kotlin-free `graphics-bom`, in one multi-module build on Kotlin 2.4.21 and Gradle 9.7.1 or 9.8.2? An answer must give:
strict explicit API and dump checks on both library modules from the first commit (a JVM dump for each, a klib dump for
`graphics-core`); the dumps committed while empty; `check`, and so `build`, failing on any public-API change the committed dumps do not
show, on every OS including Windows checkouts with `core.autocrlf`; no Kotlin and no dump for the BOM; everything under the
configuration cache; the convention-plugin code, the committed files with their initial content, and the developer commands.

## 2. Bottom line

- **One helper, called from both convention plugins, turns both guards on.** `abiValidation` and `explicitApi()` are members of the
  same interface chain (`KotlinTopLevelExtension` and `KotlinBaseExtension`) for KMP and for `kotlin("jvm")`, so a single
  `internal fun KotlinBaseExtension.guardPublicApi()` in `build-logic` (`@OptIn(ExperimentalAbiValidation::class)`, `explicitApi()`,
  `abiValidation { ... }`) is the whole wiring. The DSL is the 2.4 one: no `enabled`, `klib {}`, `legacyDump {}` or `variants`
  (all removed at ERROR level), and `exclude {}` / `include {}` in filters (not `excluded` / `included`, also ERROR). The DSL is
  still `@RequiresOptIn(ERROR)` and not stable (KT-71172, open).
- **Tasks: `checkKotlinAbi` and `updateKotlinAbi` per module; `check` depends on the former by default** for KMP and JVM once
  `abiValidation()` is called (`AbiValidationSetupAction`). No Kotlin plugin, no call, no tasks: the BOM and the root get none. The
  two tasks still fail together in one invocation on Gradle 9 (same files as output and input, no ordering declared in 2.4.21 or
  2.5.0-Beta1); one line, `checkTaskProvider.configure { mustRunAfter(updateTaskProvider) }`, orders them (the workaround the Kotlin team
  itself uses, KT-84365). Developers run `./gradlew updateKotlinAbi` and review the diff; CI's `./gradlew build` runs the check.
- **Committed files: three zero-byte files, no others.** `graphics-core/api/graphics-core.api`, `graphics-core/api/graphics-core.klib.api`
  and `graphics-desktop/api/graphics-desktop.api`. Names are `<project name>.api` and `<project name>.klib.api` in each module's own
  `api/` directory; the directory can be moved, the names cannot.
- **Empty modules are legal, and a committed file is required.** For the JVM side the dump task always writes its file, empty when there
  are no classes; for the klib side it writes an empty file when the js compilation produced no klib. `checkKotlinAbi` fails with
  "Expected file with ABI declarations ... does not exist" when the reference file is missing, empty or not, and passes against the empty
  file. KGP's own integration tests assert exactly these cases. There is no switch that turns klib validation off, and JetBrains' position
  is that an empty project is validated like any other (KT-78344, "As Designed").
- **Line endings cannot fail the check; the `.gitattributes` anchor can be wrong.** The comparison is `String.lines()` against
  `String.lines()`, so CRLF and LF are equal, but a missing or extra trailing newline is not. The plugin writes LF. GeneralTools' rule
  `api/*.api text eol=lf` is anchored to the repository root and matches nothing in module directories: use `**/api/*.api text eol=lf`.
  This corrects GeneralTools' comment that the build compares "byte-for-byte".
- **Strict explicit API applies to main and common-metadata compilations only; tests are exempt by design** (source comment in
  `AbstractKotlinCompileConfig`). The binary-compatibility-validator plugin (0.18.2) is in maintenance mode, is applied once at the root
  and configures all subprojects (cross-project configuration this build avoids), and lists different members from KGP's dumper; it is
  not used.
- **Nothing differs between Gradle 9.7.1 and 9.8.2** in what was found for these tasks; KGP's ABI sources and tests carry no Gradle-version
  logic.

## 3. Findings

Tiers: **consensus** = official source/documentation or several independent credible sources; **practice** = what mature libraries
visibly do; **opinion** = one source; **inference** = derived here from sources that do not state it.

### F1. The DSL in 2.4.21 is one extension for every project kind (consensus: source at `v2.4.21`)

- `abiValidation` is declared on `KotlinTopLevelExtension` as `val abiValidation: AbiValidationExtension`, `fun abiValidation(action:
  Action<AbiValidationExtension>)` and `fun abiValidation()`, all `@ExperimentalAbiValidation` and `@since 2.4.0`. `KotlinBaseExtension`
  extends `KotlinTopLevelExtension`; `KotlinProjectExtension` (the base of the multiplatform, JVM and Android extensions) implements it.
  "Calling this function causes ABI validation to be enabled and corresponding tasks to be created" ([KotlinTopLevelExtension.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/KotlinTopLevelExtension.kt),
  [KotlinProjectExtension.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/KotlinProjectExtension.kt)).
  `explicitApi()` is on `KotlinBaseExtension` too. There is therefore no separate "KMP form" and "JVM form".
- **Current, non-deprecated surface** of `AbiValidationExtension`: `filters` (`exclude {}` / `include {}` with `byNames` and
  `annotatedWith` sets), `referenceDumpDir` (default `<projectDir>/api`), `keepLocallyUnsupportedTargets` (default `true`),
  `binariesSource` (default `MAIN_COMPILATION`), `checkTaskProvider`, `updateTaskProvider`.
- **Removed, each now throwing at `DeprecationLevel.ERROR`:** `enabled`; `legacyDump` (property and block; its members moved up a level);
  `variants`; `klib` (property and block: "Nested property 'enabled' was removed - ABI dumps always generated for klib-based targets";
  `keepUnsupportedTargets` became `keepLocallyUnsupportedTargets`); `legacyCheckTaskProvider`, `legacyUpdateTaskProvider`,
  `legacyDumpTaskProvider` (renamed `checkTaskProvider`, `updateTaskProvider`, `internalDumpTaskProvider`); in filters, `excluded` and
  `included` (renamed `exclude`, `include`). Classes `AbiValidationMultiplatformExtension`, `AbiValidationMultiplatformVariantSpec`,
  `AbiValidationKlibKindExtension`, `AbiValidationVariantSpec` and `AbiValidationLegacyDumpExtension` are kept only as ERROR-level
  stubs ([AbiValidationExtension.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/abi/AbiValidationExtension.kt),
  [AbiValidationMultiplatformExtension.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/abi/AbiValidationMultiplatformExtension.kt),
  [AbiFiltersSpec.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/abi/AbiFiltersSpec.kt)).
  This is the 2.4.0 change the compatibility guide records as KT-80685 ([Compatibility guide for Kotlin 2.4](https://kotlinlang.org/docs/compatibility-guide-24.html)).
  The klib-kind extension that GeneralTools found deprecated at ERROR level is among these: there is no `klib { enabled = false }` any more.
- **The documentation lags the code (consensus, two readings agree).** The docs page's filter example still uses `excluded {}` and
  `included {}` ([kotlin-web-site `docs/topics/gradle/gradle-binary-compatibility-validation.md`](https://github.com/JetBrains/kotlin-web-site/blob/5d234526eb/docs/topics/gradle/gradle-binary-compatibility-validation.md),
  last changed 2026-06-03), which `AbiFiltersSpec` at `v2.4.21` rejects. Follow the KDoc: `exclude {}` and `include {}` (`@since 2.3.20`).
- **Still experimental.** `ExperimentalAbiValidation` is `@RequiresOptIn(..., RequiresOptIn.Level.ERROR)`: "The ABI Validation DSL is
  experimental and may be changed or removed in any future release." Stabilisation (KT-71172) is open; its criteria include "Gradle DSL
  is stable and will not be actively changed in the coming releases". The DSL files at `v2.5.0-Beta1` differ from `v2.4.21` in KDoc wording
  only, and `ExperimentalAbiValidation`, `KotlinAbiCheckTaskImpl`, `KotlinAbiUpdateTask` and `AbiValidationAction` are byte-identical.
- **Comparable builds split by KGP version.** Ktor 3.6.0, kotlinx-io 0.9.1 and kotlinx.coroutines 1.11.0 still write
  `abiValidation { enabled = true }` and `tasks.check { dependsOn(tasks.checkLegacyAbi) }`, which Kotlin 2.4.21 rejects (DSL churn is a
  recorded pitfall in GeneralTools' research, F6). Coil 3.6.3 (KGP 2.4.10) and Arrow 2.2.3 (KGP 2.4.0) use the 2.4 form; their snippets
  are the usable precedent.

### F2. Tasks and wiring (consensus: source and KGP's tests at `v2.4.21`)

- **Task names.** `KotlinAbiCheckTaskImpl.NAME = "checkKotlinAbi"` (group `verification`), `KotlinAbiUpdateTask.NAME = "updateKotlinAbi"`
  (group `verification`), and a hidden `internalDumpKotlinAbi` (`group = null`, `@CacheableTask`) that writes the actual dumps to
  `build/kotlin/abi`. `checkLegacyAbi` and `updateLegacyAbi` remain as aliases that warn "Task ... is deprecated, use ... instead"; the
  comment in `Configs.kt` plans their removal. There are no per-variant tasks: the variants DSL is gone.
- **When tasks exist.** Registered at the first `abiValidation()` call (KT-83486, state Fixed): KGP's `testForDisabledAbiValidation`
  asserts `:checkKotlinAbi` is not registered and not in `check`'s graph without the call. There is no way to disable it again for a
  project that called it (KT-89357, open): the call belongs in the library conventions only, and any later non-API module (sample,
  benchmark) must not apply them.
- **`check` wiring.** `AbiValidationSetupAction` does `tasks.named("check") { checkTask -> checkTask.dependsOn(abiValidation.checkTaskProvider)
  }` when the extension is activated, for `kotlinJvmExtension`, `kotlinAndroidExtension` and `multiplatformExtension` alike, after the
  `ReadyForExecution` stage. The docs say the same. Test target: `./gradlew check --dry-run` lists `:graphics-core:checkKotlinAbi`.
- **Multi-module.** Per module and nothing at the root: "If your project has multiple modules where you want to check for binary
  compatibility, configure each module separately." `./gradlew checkKotlinAbi` at the root selects the same-named task in every
  project that has it; the root project and the BOM have none, which is fine because selection needs at least one match.
- **One invocation.** `checkKotlinAbi` declares `referenceDumps` as `@InputFiles` and `updateKotlinAbi` declares the same files as
  `@OutputFiles`, and no task sets `mustRunAfter`/`dependsOn` between them (a search of the ABI sources for `mustRunAfter` finds
  nothing). GeneralTools recorded that on Gradle 9.0.0 `./gradlew checkKotlinAbi updateKotlinAbi` fails both tasks ("Task ':checkKotlinAbi'
  uses this output of task ':updateKotlinAbi' without declaring an explicit or implicit dependency") while separate invocations work
  (`plan-kmp-build.md`, section 3 and S3 in `final-implementation.md`). The Kotlin team hit the same message on KT-84365 and wrote
  "as a workaround, we provide explicit `mustRunAfter`". Fixed there: the output files were made precise (so unrelated files in `api/`
  no longer collide); the check/update overlap itself remains in 2.4.21 and 2.5.0-Beta1.
- **Configuration cache (consensus on support; not run here).** The tasks store `project.name`, the project path and `rootDir` in
  fields at construction, take the build session as a `Property<BuildSessionService>`, and KGP's integration-test base defaults to
  `configurationCache = ENABLED` with problems set to `FAIL`; the ABI tests do not override it (they disable only Isolated Projects for
  js/wasm, `disableIsolatedProjectsBecauseOfJsAndWasmKT75899`). A `mustRunAfter` edge is configuration-cache neutral.
- **Isolated Projects.** KGP's js target does not support it yet (KT-75899, a duplicate of an open report). This qualifies the question-1
  statement that nothing blocks Isolated Projects later: the `js` target of `graphics-core` does, independently of this build's scripts.
- **What the tasks run.** The dump and check tasks execute through a build-tools-API classpath configuration that KGP resolves from the
  project's repositories (`AbiToolsTask.buildToolsClasspath`, `Configs.kt`), so the first run needs that artifact from Maven Central or a
  cache. The default `repositoriesMode` kept in question 1 allows the resolution (inference; not run).

### F3. Dump files: locations, names, headers (consensus: source; practice: Coil and Arrow)

- **Location and names.** `referenceDumpDir` defaults to `<projectDir>/api`; the JVM file is `<project.name>.api`, the klib file
  `<project.name>.klib.api` (`AbiValidationPaths`, `registerTasks` in `Configs.kt`). `project.name` is the name from `include(...)` in
  `settings.gradle.kts`: KGP's `updateKotlinAbi should create abi file with the name of the project, respecting settings file` test.
  Matching is case-sensitive on case-sensitive file systems. Here: `graphics-core`, `graphics-desktop`.
- **Movable, not renamable.** `referenceDumpDir` can point elsewhere (Coil 3.6.3 sets `layout.projectDirectory.dir("api")`
  explicitly, which is the default); `updateKotlinAbi` refuses a directory outside the build root ("'referenceDir' must be a subdirectory
  of the build root directory"). No property renames the files. A central tree is possible with one `referenceDumpDir` per module but
  buys nothing: BCV, kotlinx, Coil and Arrow all keep `api/` beside each module.
- **One JVM target means no sub-directory.** `processJvmKindTargets` takes a shortcut "if there is only one JVM target": the file is
  `api/graphics-core.api`. With a second JVM-kind target (an Android target in the Android phase) each JVM dump moves to
  `api/<targetName>/graphics-core.api`, which is why Coil's `coil-core/api` holds `android/`, `jvm/` and `coil-core.klib.api`. Plan
  that one-time move for the Android phase.
- **Both dumps are merged and sorted by the tool.** JVM dump: classes sorted by name, members by a fixed order, each class block
  ending `}` plus a blank line. Klib dump header (non-empty; GeneralTools' real file, targets `js` and `wasmJs`):
  `// Klib ABI Dump`, `// Targets: [...]`, `// Rendering settings:`, `// - Signature version: 2`, `// - Show manifest properties: true`,
  `// - Show declarations: true`, a blank line, `// Library unique name: <io.github.spartanlaboratories:GeneralTools>`. The unique name is
  `group:name`, so `group` and the project name are part of the file; the rendering settings are part of it too, so a plugin change
  of them shows as a dump diff (inference).
- **Not dumped:** test compilations (`binariesSource = MAIN_COMPILATION` takes the main compilation's output). The maven-publication
  mode exists, has an open empty-dump bug when no publications are configured (KT-85826), and is not needed.

### F4. What the plugin does with no source (consensus for each side; the both-empty KMP module is derived, not observed)

- **JVM side (`graphics-desktop`, and the `jvm()` target of `graphics-core`).** The dump task opens the output file unconditionally
  (`dumpFile.bufferedWriter().use { ... }`), then `printJvmDump` walks the class files; with none, `dump(to)` appends nothing. So
  `updateKotlinAbi` writes a **0-byte** `<project>.api`. KGP's tests at `v2.4.21`: "updateKotlinAbi should create empty abi file when there are
  no Kotlin sources" (`assertDumpsEqual("", ...)`), "checkKotlinAbi should fail, when there is no abi directory, even if there are no Kotlin
  sources", "checkKotlinAbi should succeed, when abi file is empty, but where are no kotlin source files", and for KMP "Test empty project with
  JVM target only" (check fails, update, check passes).
- **Klib side (`graphics-core`).** The klib dump is written when the project has a klib target or an unsupported target
  (`klibTargets.isNotEmpty() || unsupported.isNotEmpty()`), and a target contributes its klib only `if (klibDir.exists())`. With no
  `js` sources there is no klib, the merged dump has no targets, and `KlibAbiDumpFileMerger.dump` returns before writing even the header: a
  **0-byte** `<project>.klib.api`. KGP's tests: "updateKotlinAbi should not fail for empty project and should dump empty file" (also
  overwrites a non-empty reference with an empty one) and "checkKotlinAbi should fail for empty project without a dump file" with the
  message "Expected file with ABI declarations 'api/base-kotlin-multiplatform-library.klib.api' does not exist." GeneralTools observed the same
  0-byte klib file for `js` and `wasmJs` with no source on Gradle 9.0.0, beside a non-empty JVM dump.
- **Is a committed file required?** Yes, for every dump the plugin writes. `checkKotlinAbi` iterates the actual dumps and fails for any
  without a reference file; it does not need a reference file for a dump that is not written (a JVM-only module needs no `.klib.api`),
  and a stale extra reference file is ignored silently (inference from the loop: leftover references are never reported).
- **No off switch, and none is wanted.** The `klib` block that could disable klib dumps is gone (F1); JetBrains closed KT-78344 as "As
  Designed": "The absence of declarations in the project may be an error, otherwise the user must explicitly disable ABI validation for an
  empty project." The roadmap requires the guard from the first commit, so empty references are committed.
- **Nothing else fails on emptiness.** `compileKotlinJvm`, `compileKotlinJs` and `compileKotlin` are `NO-SOURCE`; the dump task's inputs
  are `@Optional`. The first `commonMain` declaration changes both `graphics-core` dumps (the klib dump becomes header plus
  declarations, and if the module holds only `internal` declarations it may become a header-only dump, which was not observed;
  see Known gaps).

### F5. Line endings and file comparison (consensus: source; corrects a sibling comment)

- `AbiToolsImpl.filesDiff` reads both files as text and compares `expectedText.lines()` with `actualText.lines()`; the source comment
  reads "We don't compare a full text because newlines on Windows & Linux/macOS are different". The same function serves JVM and klib
  dumps (`CompareAbiTextFilesOperationImpl`). Consequences:
  - CRLF versus LF never fails the check, whatever `core.autocrlf` is.
  - A trailing-newline difference does fail it: `"}\n\n".lines()` and `"}\n".lines()` differ, and so do `"".lines()` (one empty
    element) and `"\n".lines()` (two). Dumps are generated, never hand-edited, and no formatter or save-action may touch them. The EditorConfig
    specification exempts empty files from `insert_final_newline` ("Editors must not insert newlines in empty files when saving those
    files, even if `insert_final_newline = true`"), and a non-empty dump already ends with a newline, so a repository-wide
    `insert_final_newline = true` is harmless. `insert_final_newline = false` would strip a newline from non-empty dumps and must not
    apply to them.
- The plugin writes LF on every OS (`appendLine` appends `'\n'`), and `updateKotlinAbi` copies the file unchanged.
- The `.gitattributes` rule is hygiene, not correctness: it makes the index and every checkout LF so that `git diff` after
  `updateKotlinAbi` is empty without line-ending warnings under `core.autocrlf=true`, and so that a developer with `autocrlf=false` and a
  CRLF-writing editor cannot commit CRLF dumps. **Anchoring:** per the gitignore rules that `.gitattributes` patterns follow, "If there is
  a separator at the beginning or middle (or both) of the pattern, then the pattern is relative to the directory level of the particular
  `.gitignore` file itself", so GeneralTools' `api/*.api` would match only `/api/`. A leading `**/` "means match in all directories".
  GeneralTools' comment "Reference files compared byte-for-byte by the build (checkKotlinAbi) stay LF" misdescribes the comparison;
  its rule is right for its single root-level `api/`.
- Empty files are unaffected by `text eol=lf`; an empty blob is `e69de29bb2d1d6434b8b29ae775ad8c2e48c5391`.

### F6. Strict explicit API in both module kinds (consensus: source)

- `explicitApi()` sets `ExplicitApiMode.Strict` on the project extension. KGP passes it to a compilation only when it is a main
  compilation or a common (metadata) compilation: "Plugin explicitly does not configure 'explicitApi' mode for test sources compilation, as
  test sources are not published". Test source sets (`commonTest`, `jvmTest`, `jsTest`, `test`) are exempt in the KMP and the JVM module.
- It is pure configuration, so it holds with no source and costs nothing until the first declaration. GeneralTools measured the migration
  cost on 135 declarations; here there are none.
- It is independent of ABI validation: strict mode makes a declaration public on purpose, the dump records what is public. The dump
  cannot show `sealed` to open, opt-in gating (the marker class is dumped, its use is not), inline-function bodies or behaviour
  (GeneralTools' research, F5.2 and pitfall 4).

### F7. The alternative tool, binary-compatibility-validator 0.18.2 (consensus: its README and the sibling record)

- Its README: "This plugin is in maintenance mode." with new features "stopped in favor of adding them to the experimental binary
  compatibility validation in the Kotlin Gradle plugin". Klib support is "experimental and is a subject to change".
- It is applied once: "It is enough to apply the plugin only to the root project build file; all sub-projects will be configured
  automatically." That is cross-project configuration from the root, the pattern question 1 replaced with convention plugins to keep
  Isolated Projects possible; per-module opt-in (`ignoredProjects`) is a deny-list.
- It writes the same file names (`api/<project>.api`, `api/<project>.klib.api`), so a later switch either way costs one `apiDump`/
  `updateKotlinAbi` and a reviewed diff.
- **The two tools list different members.** For GeneralTools' 2.2.0 sources, BCV 0.18.2 lists `public synthetic fun <init> (DDILkotlin/jvm/internal/DefaultConstructorMarker;)V`
  of the sealed `TwoDoubles` and KGP 2.4.21 does not (it takes visibility from the primary constructor, `protected` in a sealed class)
  (`plan-kmp-build.md`, "The 2.2.0 API in the two dump tools"). A baseline from one tool cannot be checked by the other without a
  reviewed one-time diff. Here there is no baseline, so the choice is free.
- Practice: Okio `parent-3.18.2` (BCV 0.18.1) and kotlinx.serialization `1.12.0` still use BCV; Coil, Arrow and (older DSL) Ktor,
  kotlinx-io and kotlinx.coroutines use KGP's tool. The roadmap and sibling GeneralTools both use KGP's.

### F8. Excluding non-public markers later (consensus: KDoc; practice: Coil, Arrow)

- `filters { exclude { annotatedWith.add("<fqn>") ; byNames.add("<pattern>") } }` drops matching classes, top-level functions and
  properties, and members from both the JVM and the klib dump. The annotation must be `BINARY` or `RUNTIME` retention (a `SOURCE`
  annotation "will be ignored"). Name patterns take `**`, `*` and `?`, with `.` as the separator everywhere. `include {}` is the
  inverse and is not needed.
- Coil 3.6.3: `filters { exclude { annotatedWith.add("coil3.annotation.InternalCoilApi") } }` in the 2.4 DSL. Arrow 2.2.3 excludes its
  `Experimental...Api` markers the same way. BCV's equivalent was `nonPublicMarkers`.
- Shape for this repository: a typed list `NON_PUBLIC_API_MARKERS` in `SpartanBuild`, empty in G1, added to `exclude.annotatedWith` in
  `guardPublicApi()`. An empty set equals the default, so G1's dumps are unaffected. Markers meant to stay visible in the dump (the
  repository's experimental `@RequiresOptIn` tier and `@SupportedExtension`) must **not** be listed: excluding them would hide the
  changes the dumps exist to show. Excluding an internal-API marker hides changes to those declarations by design.
- A filter change is an input change to `internalDumpKotlinAbi` and re-dumps; the committed files change only if a declaration moves in
  or out.

### F9. Gradle 9.7.1 against 9.8.2 (inference from absence)

- KGP's ABI tasks and `AbiValidationAction` contain no Gradle-version branches; the Gradle 9 upgrade guide through 9.8.2 has no item on
  task input/output validation that changes the check/update overlap (its implicit-dependency item concerns tasks across builds).
  The documented KGP ceiling (9.7.0) and the wrapper choice are question 2's; nothing in this question moves it. The one-invocation
  failure was recorded on 9.0.0 and the source is unchanged, so it is expected on both candidates.

## 4. How comparable projects do it

| Project @ ref | KGP | Where ABI is configured | Dump files | `check` and CI |
| --- | --- | --- | --- | --- |
| [Coil 3.6.3](https://github.com/coil-kt/coil/tree/3.6.3) | 2.4.10 | root `build.gradle.kts` `configureAbiValidation()` for every module in `publicModules`: `referenceDumpDir.set(layout.projectDirectory.dir("api"))`, `filters { exclude { annotatedWith.add("coil3.annotation.InternalCoilApi") } }`, `binariesSource.set(BinariesSource.MAIN_COMPILATION)` inside `afterEvaluate` | `coil-core/api/coil-core.klib.api` plus `api/jvm/coil-core.api` and `api/android/` (several JVM-kind targets) | CI step `./gradlew checkKotlinAbi \|\| { ... run './gradlew updateKotlinAbi' and commit the new API files ...}`; `.gitattributes` `* text=auto eol=lf` |
| [Arrow 2.2.3](https://github.com/arrow-kt/arrow/tree/2.2.3) | 2.4.0 | `buildSrc` `arrow.kotlin.gradle.kts`: `fun KotlinProjectExtension.applyValidationIfNeeded()` calls `abiValidation { filters { exclude { annotatedWith.addAll(...) } } }` for modules not on an exclusion list | `arrow-libs/core/arrow-core/api/arrow-core.klib.api`, `api/jvm/`, `api/android/` | no `.gitattributes` |
| [Ktor 3.6.0](https://github.com/ktorio/ktor/tree/3.6.0) | 2.3.x | `ktorbuild.compatibility.gradle.kts`: `kotlinExtension.abiValidation { enabled = true }` | per module | pre-2.4 DSL; does not compile on 2.4.21 |
| [kotlinx.coroutines 1.11.0](https://github.com/Kotlin/kotlinx.coroutines/tree/1.11.0) | pre-2.4 | `buildSrc` `kotlin-multiplatform-conventions` and `kotlin-jvm-conventions`: `abiValidation { enabled = abiCheckEnabled }`, `tasks.check { dependsOn(tasks.checkLegacyAbi) }` | `kotlinx-coroutines-core/api/kotlinx-coroutines-core.api` and `.klib.api` | pre-2.4 DSL |
| [kotlinx-io 0.9.1](https://github.com/Kotlin/kotlinx-io/tree/0.9.1) | pre-2.4 | `kotlinx-io-compatibility.gradle.kts` convention plugin: `abiValidation { enabled = true }`, `tasks.check { dependsOn(tasks.checkLegacyAbi) }` | per module | pre-2.4 DSL |
| [Okio parent-3.18.2](https://github.com/square/okio/tree/parent-3.18.2) | n/a | root applies BCV 0.18.1 | `okio/api/okio.api` | BCV |
| GeneralTools `5eef2dc` | 2.4.21 | one module, `@OptIn(ExperimentalAbiValidation::class) abiValidation()` in `kotlin {}` | `api/GeneralTools.api`, `api/GeneralTools.klib.api` (0 bytes until the first common declaration) | `check`; rule `api/*.api text eol=lf` |

What survives across projects: dumps in each module's own `api/`, the plugin's default names, `check` as the gate, an
annotation-based exclusion of internal-API markers, and a CI hint telling the contributor to run the update task. The convention-plugin
form (Arrow, kotlinx-io, Ktor) is the form this build uses.

## 5. Pitfalls and regrets

- **Copying a pre-2.4 snippet.** `enabled = true`, `klib {}`, `legacyDump {}`, `variants`, `excluded {}` and `checkLegacyAbi` all fail or warn on 2.4.21;
  the docs page itself still shows `excluded {}` / `included {}`. Use the KDoc and Coil's file as the model.
- **`check` plus `update` in one invocation** fails on Gradle 9 unless ordered (F2). Without the `mustRunAfter` line, the
  documented developer workflow must keep them in separate invocations.
- **A missing reference file fails the build even for an empty module**, and deleting the empty klib file does the same. A zero-byte
  file is easy to lose to tooling that drops empty files (some archive and copy tools, "clean untracked" scripts); verify with
  `git ls-files -s` that all three blobs are `e69de29b...`.
- **Trailing-newline sensitivity** (F5). Editors are not the risk (EditorConfig exempts empty files; non-empty dumps already end in a newline);
  formatters and "strip trailing blank lines" actions are.
- **`.gitattributes` anchoring** (F5): `api/*.api` silently covers nothing under `graphics-core/`.
- **File names follow the project name; the klib header follows `group:name`.** Renaming a module or changing `group` rewrites its dumps,
  and in the klib case also the header line of a non-empty dump.
- **The JVM dump path changes with a second JVM-kind target** (F3). The Android phase moves `api/graphics-core.api` to a sub-directory.
- **No off switch** (KT-89357): a module that applies the library convention is validated. The BOM applies no Kotlin plugin and is
  therefore never validated; samples and benchmarks must not apply the library conventions.
- **Stale extra reference files are ignored** by the check (F4). A stray `graphics-desktop.klib.api` would never be flagged.
- **The dump is a signature set, not a contract.** It is blind to `sealed` to open, opt-in gating, inline bodies and behaviour
  (GeneralTools, F5.2); a clean diff proves less than it appears to. An excluded internal-API marker hides changes by design.
- **Experimental DSL.** Pin KGP exactly (question 2 does); expect to touch the helper at the next KGP upgrade. Keeping all DSL use in
  one helper bounds the exposure to one file.
- **Isolated Projects** is blocked by the js target (KT-75899), not by this wiring.

## 6. Recommendation for this repository

Apply both guards from one helper in `build-logic`, called by the KMP and JVM conventions, with the plugin's default dump layout (each
module's `api/`), three committed zero-byte dumps, and `check` as the gate. The reasoning: `abiValidation` and `explicitApi()` live on the
interface that both Kotlin extensions share, so one typed function covers both module kinds and keeps the one experimental opt-in in one
file; the plugin's own tests establish that an empty module needs, and accepts, empty committed references, so "dumps committed even
while empty" is the plugin's intended state and not a workaround; and nothing here needs BCV, a central dump tree or a root-level
aggregator. Rejected: BCV (maintenance mode, root-applied cross-project configuration, a different member set); switching validation off
until the first declaration (no switch exists, and JetBrains treats the empty case as an error to be made explicit); a central `api/`
tree (no benefit, every comparable build keeps `api/` beside the module).

### 6.1 `build-logic` code

`build-logic/src/main/kotlin/com/spartanlabs/graphics/build/SpartanBuild.kt` gains one constant (question 1 and 2 define the object):

```kotlin
internal object SpartanBuild {
    // ... JVM_LEVEL, JVM_TARGET, TEST_LEVELS (questions 1 and 2)

    /** Annotations (fully qualified) that mark public-but-not-API declarations. Empty in G1; see PublicApi.kt. */
    val NON_PUBLIC_API_MARKERS: List<String> = emptyList()
}
```

`build-logic/src/main/kotlin/com/spartanlabs/graphics/build/PublicApi.kt`:

```kotlin
package com.spartanlabs.graphics.build

import org.jetbrains.kotlin.gradle.dsl.KotlinBaseExtension
import org.jetbrains.kotlin.gradle.dsl.abi.ExperimentalAbiValidation

/**
 * Turns on the two public-API guards for a library module: strict explicit API (main and common compilations; tests are exempt) and
 * the Kotlin Gradle plugin's ABI validation, with reference dumps in `<module>/api/` and `check` depending on `checkKotlinAbi`.
 * Works for the multiplatform and the JVM extension alike: `abiValidation` and `explicitApi()` are members of [KotlinBaseExtension].
 */
@OptIn(ExperimentalAbiValidation::class)
internal fun KotlinBaseExtension.guardPublicApi() {
    explicitApi()
    abiValidation {
        // Default reference dir: <projectDir>/api, files <project.name>.api and (KMP with a klib target) <project.name>.klib.api.
        filters {
            exclude { annotatedWith.addAll(SpartanBuild.NON_PUBLIC_API_MARKERS) }
        }
        // The check reads the files the update writes; without an ordering, Gradle 9 rejects `updateKotlinAbi checkKotlinAbi` in one run.
        checkTaskProvider.configure { mustRunAfter(updateTaskProvider) }
    }
}
```

Both conventions call it inside their `kotlin { }` block, in place of the bare `explicitApi()` that questions 1 (6.9) and 2 (6.3) show:

```kotlin
// spartangraphics.kmp-library.gradle.kts            // spartangraphics.jvm-library.gradle.kts
kotlin {                                              kotlin {
    guardPublicApi()                                      guardPublicApi()
    jvmToolchain(SpartanBuild.JVM_LEVEL)                  jvmToolchain(SpartanBuild.JVM_LEVEL)
    // ... floor, targets, source sets                    // ... floor
}                                                     }
```

with `import com.spartanlabs.graphics.build.*` at the top of each script (already there for `SpartanBuild` and `catalog`). Nothing is added
to `graphics-bom`, to the root project, or to any module script. The root project and the BOM never apply a Kotlin plugin, so they
register no ABI task. The module scripts stay at `plugins { id("spartangraphics.kmp-library") }` plus dependencies.

### 6.2 Committed files and initial content

| Path | Initial content | Written by |
| --- | --- | --- |
| `graphics-core/api/graphics-core.api` | empty, 0 bytes | `:graphics-core:updateKotlinAbi` |
| `graphics-core/api/graphics-core.klib.api` | empty, 0 bytes | `:graphics-core:updateKotlinAbi` |
| `graphics-desktop/api/graphics-desktop.api` | empty, 0 bytes | `:graphics-desktop:updateKotlinAbi` |
| `graphics-bom/` | no `api/` directory | none |

Generate them with `./gradlew updateKotlinAbi` rather than by hand, so the plugin proves it writes exactly these names. Nothing in the
current `.gitignore` matches `api/` or `*.api`. The first `commonMain` declaration (G2) makes `graphics-core.api` and
`graphics-core.klib.api` non-empty: the commit that adds the declaration includes the regenerated dumps. Expected shape of the klib
header then (derived from GeneralTools' file, with the single `js` target): `// Klib ABI Dump`, `// Targets: [js]`, the three
`// Rendering settings` lines, a blank line, `// Library unique name: <io.github.spartanlaboratories:graphics-core>`.

### 6.3 `.gitattributes`

```gitattributes
# ABI reference dumps stay LF in the index and in every checkout. checkKotlinAbi compares line by line, so line endings cannot fail the
# check; this keeps `git diff` after updateKotlinAbi empty and the files identical on Windows, Linux and macOS.
# The leading **/ is required: a pattern with a slash in the middle is relative to this file, so `api/*.api` would match only /api/.
**/api/*.api text eol=lf
```

`*.klib.api` is covered because `*` matches `graphics-core.klib`. Merge this line into the repository's `.gitattributes` (question 7's
file list owns the rest of it).

### 6.4 Commands

| Who | Command | Purpose |
| --- | --- | --- |
| Developer, after an intended API change | `./gradlew updateKotlinAbi` (or `:graphics-core:updateKotlinAbi`) | rewrite the dumps; `git diff -- "*/api/*"` is the API change to review and commit with the code |
| Developer, before a commit | `./gradlew check` (or `build`) | tests plus `checkKotlinAbi` in each library module |
| Developer, API check alone | `./gradlew checkKotlinAbi` | fastest check |
| CI, `windows-latest` | `./gradlew build` | `check` includes `:graphics-core:checkKotlinAbi` and `:graphics-desktop:checkKotlinAbi` |
| CI, `ubuntu-latest`, `macos-latest` | compile tasks only | no ABI task needed; the js klib dump is host-independent, and the build has no native target |

On a failure the task prints `<<<ABI has changed>>>`, a unified diff, and "You can run ':graphics-core:updateKotlinAbi' task to create or
overwrite reference ABI declarations". A separate named CI step for `checkKotlinAbi` with a contributor hint (Coil's pattern) is optional;
the check already runs inside `build`. `checkLegacyAbi` and `updateLegacyAbi` are not used. With the `mustRunAfter` line,
`./gradlew updateKotlinAbi check` in one invocation is valid; without it the two must be separate invocations.

### 6.5 What the implementation must verify (the wiring was not executed here)

1. `./gradlew updateKotlinAbi` writes exactly the three files, each 0 bytes, and nothing under `graphics-bom/` or the root;
   `git ls-files -s` shows blob `e69de29bb2d1d6434b8b29ae775ad8c2e48c5391` for each.
2. `./gradlew check --dry-run` lists `:graphics-core:checkKotlinAbi` and `:graphics-desktop:checkKotlinAbi`, and no ABI task for
   `:graphics-bom` or the root.
3. Negative controls (do not commit): delete one file, `./gradlew check` fails with "Expected file with ABI declarations ... does not exist";
   add a public declaration in a scratch `commonMain`/`main` file, it fails with `<<<ABI has changed>>>`; add a declaration without a
   modifier, compilation fails under strict mode, while an unannotated declaration in `commonTest` compiles.
4. `./gradlew updateKotlinAbi checkKotlinAbi` in one invocation passes; remove the `mustRunAfter` line and confirm the Gradle
   validation failure (this proves the line does what the comment says).
5. `./gradlew check` twice with `--configuration-cache`: the second run reports "Reusing configuration cache" and `checkKotlinAbi` still runs.
6. Convert one committed dump to CRLF and run `checkKotlinAbi` (passes); append one newline (fails). This pins F5 on the pinned plugin version.
7. Whether the first `updateKotlinAbi` on a clean machine downloads Node.js or Yarn for the `js` compile (Known gaps).

## 7. Load-bearing claims

Quotes from Kotlin, Markdown or Java sources omit markup characters only; text from web pages is as extracted.

1. **`abiValidation` is a `KotlinTopLevelExtension` member, since 2.4.0, and calling it creates the tasks.** "Calling this function causes ABI
   validation to be enabled and corresponding tasks to be created." (with `@since 2.4.0`).
   [KotlinTopLevelExtension.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/KotlinTopLevelExtension.kt), `v2.4.21`.
2. **`KotlinBaseExtension` extends `KotlinTopLevelExtension`; `KotlinProjectExtension` implements `abiValidation`.** `interface KotlinBaseExtension :
   KotlinTopLevelExtension`; `abstract class KotlinProjectExtension ... : KotlinBaseExtension`; `override fun abiValidation() { abiValidationInternal.activate(compilerVersion) }`.
   [KotlinBaseExtension.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/KotlinBaseExtension.kt),
   [KotlinProjectExtension.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/KotlinProjectExtension.kt), `v2.4.21`.
3. **The DSL is experimental behind an ERROR-level opt-in.** "The ABI Validation DSL is experimental and may be changed or removed in any
   future release." with `RequiresOptIn.Level.ERROR`.
   [ExperimentalAbiValidation.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/abi/ExperimentalAbiValidation.kt), `v2.4.21`.
4. **`enabled`, `legacyDump`, `variants` and `klib` are removed at ERROR level.** "Property was removed, to enable ABI validation call function
   abiValidation(), abiValidation { ... } or read abiValidation property."; "A separate property 'legacyDump' was removed. Please place all its
   properties on a higher level."; "Variants DSL was removed and is no longer supported."; "Nested property 'enabled' was removed - ABI dumps
   always generated for klib-based targets." (each `level = DeprecationLevel.ERROR`).
   [AbiValidationExtension.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/abi/AbiValidationExtension.kt), `v2.4.21`.
5. **Filter blocks are `exclude`/`include`; `excluded`/`included` are ERROR.** "'excluded' property for filters is deprecated. Use 'exclude'
   instead." ; `exclude` and `include` are marked "@since 2.3.20".
   [AbiFiltersSpec.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/abi/AbiFiltersSpec.kt), `v2.4.21`.
6. **The 2.4.0 DSL change is documented as a removal.** "Kotlin 2.4.0 simplifies the ABI validation Gradle DSL and removes redundant configuration
   entries. To migrate, configure report settings directly in `abiValidation {}` instead of `abiValidation { legacyDump { ... } }`, remove
   `abiValidation { klib { enabled = ... } }`, and use `keepLocallyUnsupportedTargets` instead of `klib.keepUnsupportedTargets`."
   [Compatibility guide for Kotlin 2.4](https://kotlinlang.org/docs/compatibility-guide-24.html), read 2026-10-20.
7. **The documentation page still shows the removed filter names.** "Use the `filters {}` block to add exclusion and inclusion rules with
   `excluded {}` and `included {}` blocks respectively."
   [gradle-binary-compatibility-validation.md](https://github.com/JetBrains/kotlin-web-site/blob/5d234526eb/docs/topics/gradle/gradle-binary-compatibility-validation.md), `kotlin-web-site` master `5d234526eb` (2026-10-20).
8. **`check` depends on `checkKotlinAbi` for JVM and multiplatform projects once activated.** `if (!abiValidation.isActivated) return@KotlinProjectSetupCoroutine`
   then `tasks.named("check") { checkTask -> checkTask.dependsOn(abiValidation.checkTaskProvider) }`, with branches on `kotlinJvmExtensionOrNull`
   and `multiplatformExtensionOrNull`. Docs: "By default, when binary compatibility validation is enabled in your project and you run the `check` task,
   Gradle also runs the `checkKotlinAbi` task."
   [AbiValidationAction.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/abi/internal/AbiValidationAction.kt), `v2.4.21`;
   [docs source](https://github.com/JetBrains/kotlin-web-site/blob/5d234526eb/docs/topics/gradle/gradle-binary-compatibility-validation.md).
9. **Task names and aliases.** `const val NAME: String = "checkKotlinAbi"` (`KotlinAbiCheckTaskImpl`), `const val NAME = "updateKotlinAbi"` (`KotlinAbiUpdateTask`),
   `const val NAME = "internalDumpKotlinAbi"` (`KotlinAbiDumpTaskImpl`), and `tasks.register("checkLegacyAbi")` whose action logs "Task ${it.path} is deprecated, use $projectPath:$checkTaskName instead".
   [KotlinAbiCheckTaskImpl.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/tasks/abi/KotlinAbiCheckTaskImpl.kt),
   [KotlinAbiUpdateTask.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/tasks/abi/KotlinAbiUpdateTask.kt),
   [Configs.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/abi/internal/Configs.kt), `v2.4.21`.
10. **Default directory and file names.** `LEGACY_DEFAULT_REFERENCE_DUMP_DIR = "api"`, `LEGACY_JVM_DUMP_EXTENSION = ".api"`, `LEGACY_KLIB_DUMP_EXTENSION = ".klib.api"`;
    `referenceDumpDir.convention(project.layout.projectDirectory.dir(AbiValidationPaths.LEGACY_DEFAULT_REFERENCE_DUMP_DIR))`;
    `val klibFileName = "$projectName$LEGACY_KLIB_DUMP_EXTENSION"`; `val jvmDumpName = projectName + LEGACY_JVM_DUMP_EXTENSION`.
    [AbiValidationPaths.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/abi/internal/AbiValidationPaths.kt),
    [Configs.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/abi/internal/Configs.kt),
    [KotlinAbiDumpTaskImpl.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/tasks/abi/KotlinAbiDumpTaskImpl.kt), `v2.4.21`.
11. **A single JVM target gets no sub-directory; several do.** "// if there is only one JVM target then we will follow the shortcut"
    (then `addJvmInputs(..., true, ...)`); otherwise `addJvmInputs(..., target.targetName, false, ...)`, and `addJvmTarget(targetName, classfiles)`.
    [Multiplatform.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/abi/internal/Multiplatform.kt), `v2.4.21`.
12. **The check requires a reference for every dump written, and the update writes into the build root only.**
    "Expected file with ABI declarations '${referenceDump.relativeTo(rootDir)}' does not exist."; "You can run '$pathPrefix${KotlinAbiUpdateTask.NAME}' task to
    create or overwrite reference ABI declarations"; "'referenceDir' must be a subdirectory of the build root directory".
    [KotlinAbiCheckTaskImpl.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/tasks/abi/KotlinAbiCheckTaskImpl.kt),
    [KotlinAbiUpdateTask.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/tasks/abi/KotlinAbiUpdateTask.kt), `v2.4.21`.
13. **Comparison is line-wise.** "// We don't compare a full text because newlines on Windows & Linux/macOS are different"; `val expectedLines = expectedText.lines()`;
    `val actualLines = actualText.lines()`; `if (expectedLines == actualLines) { return null }`.
    [AbiToolsImpl.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/abi-validation/abi-tools/src/main/kotlin/org/jetbrains/kotlin/abi/tools/impl/AbiToolsImpl.kt), `v2.4.21`.
14. **The dump task writes the JVM file unconditionally and the klib file only when a klib target exists; a missing klib is skipped.**
    `dumpFile.bufferedWriter().use { writer ->`; `if (klibTargets.isNotEmpty() || unsupported.isNotEmpty()) {`; `if (klibDir.exists()) {`.
    [KotlinAbiDumpTaskImpl.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/tasks/abi/KotlinAbiDumpTaskImpl.kt), `v2.4.21`.
15. **A klib dump with no targets writes nothing; a JVM dump with no classes writes nothing; dump lines end in LF.**
    `if (targets.isEmpty()) { check(topLevelDeclaration.children.isEmpty()) { "Dump containing some declaration should have at least a single target" } return }`;
    `fun <T : Appendable> List<ClassBinarySignature>.dump(to: T): T { sortedBy { it.name }.forEach { ... appendLine("}\n") ...`.
    [KlibAbiDumpFileMerger.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/abi-validation/abi-tools/src/main/kotlin/org/jetbrains/kotlin/abi/tools/impl/klib/KlibAbiDumpFileMerger.kt),
    [KotlinSignaturesLoading.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/abi-validation/abi-tools/src/main/kotlin/org/jetbrains/kotlin/abi/tools/impl/jvm/KotlinSignaturesLoading.kt), `v2.4.21`.
16. **KGP's own tests cover the empty cases.** `@DisplayName("updateKotlinAbi should create empty abi file when there are no Kotlin sources")`;
    `@DisplayName("checkKotlinAbi should fail, when there is no abi directory, even if there are no Kotlin sources")`;
    `@DisplayName("checkKotlinAbi should succeed, when abi file is empty, but where are no kotlin source files")`;
    `@DisplayName("updateKotlinAbi should not fail for empty project and should dump empty file")`;
    `@DisplayName("checkKotlinAbi should fail for empty project without a dump file")`.
    [AbiValidationJvmIT.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-integration-tests/src/test/kotlin/org/jetbrains/kotlin/gradle/abi/AbiValidationJvmIT.kt),
    [AbiValidationKmpIT.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-integration-tests/src/test/kotlin/org/jetbrains/kotlin/gradle/abi/AbiValidationKmpIT.kt), `v2.4.21`.
17. **No tasks without the call.** `buildAndAssertAllTasks(notRegisteredTasks = listOf(":checkKotlinAbi"))` in `testForDisabledAbiValidation`.
    [AbiValidationTasksIT.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-integration-tests/src/test/kotlin/org/jetbrains/kotlin/gradle/abi/AbiValidationTasksIT.kt), `v2.4.21`.
18. **JetBrains' position on empty modules.** KT-78344 (State "As Designed"): "The absence of declarations in the project may be an error, otherwise the
    user must explicitly disable ABI validation for an empty project."
    [KT-78344](https://youtrack.jetbrains.com/issue/KT-78344), read 2026-10-20.
19. **No way to disable the validation once enabled.** KT-89357, title "ABI Validation: No way to disable abiValidation once it is enabled" (state Submitted, affected 2.4.20).
    [KT-89357](https://youtrack.jetbrains.com/issue/KT-89357), read 2026-10-20.
20. **The check/update overlap and the Kotlin team's own workaround.** KT-84365: "Task ... uses this output of task ':compiler:psi:psi-api:updateKotlinAbi' without declaring an
    explicit or implicit dependency" and "Currently, as a workaround, we provide explicit `mustRunAfter`, even though there is no such practical need."
    [KT-84365](https://youtrack.jetbrains.com/issue/KT-84365), read 2026-10-20. GeneralTools' recorded observation on Gradle 9.0.0:
    GeneralTools `docs/plans/7-kotlin-multiplatform/final-implementation.md`, S3 (sibling repository; stated as a repository fact, not an outside claim).
21. **Explicit API is strict via `explicitApi()` and exempts tests.** `override fun explicitApi() { explicitApi = ExplicitApiMode.Strict }`; "Plugin explicitly does not
    configure 'explicitApi' mode for test sources compilation, as test sources are not published" (the condition is `isMain || isCommonCompilation || isMainAndroidCompilation`).
    [KotlinProjectExtension.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/KotlinProjectExtension.kt),
    [AbstractKotlinCompileConfig.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/tasks/configuration/AbstractKotlinCompileConfig.kt), `v2.4.21`.
22. **KGP's tests run with the configuration cache on.** `val configurationCache: ConfigurationCacheValue = ConfigurationCacheValue.ENABLED,` and
    `val configurationCacheProblems: ConfigurationCacheProblems = ConfigurationCacheProblems.FAIL,`.
    [BuildOptions.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-integration-tests/src/test/kotlin/org/jetbrains/kotlin/gradle/testbase/BuildOptions.kt), `v2.4.21`.
23. **Each module is configured separately.** "If your project has multiple modules where you want to check for binary compatibility, configure each module separately."
    [docs source](https://github.com/JetBrains/kotlin-web-site/blob/5d234526eb/docs/topics/gradle/gradle-binary-compatibility-validation.md), `5d234526eb`.
24. **Exclusion by annotation needs BINARY or RUNTIME retention.** "The annotation **must** have a [Retention] of [BINARY][AnnotationRetention.BINARY] or
    [RUNTIME][AnnotationRetention.RUNTIME] type. Annotations with [SOURCE][AnnotationRetention.SOURCE] retention cannot be analyzed and will be ignored."
    [AbiFiltersSpec.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-api/src/common/kotlin/org/jetbrains/kotlin/gradle/dsl/abi/AbiFiltersSpec.kt), `v2.4.21`.
25. **Coil 3.6.3 (KGP 2.4.10) uses the 2.4 DSL with an internal-API marker exclusion and a CI step.** `referenceDumpDir.set(layout.projectDirectory.dir("api"))`;
    `exclude { annotatedWith.add("coil3.annotation.InternalCoilApi") }`; CI: `run: ./gradlew checkKotlinAbi || { exit_code=$?; printf "*** This PR changes Coil's public API. If this change is intentional, run './gradlew updateKotlinAbi' and commit the new API files. ***"; exit $exit_code; }`.
    [build.gradle.kts](https://github.com/coil-kt/coil/blob/3.6.3/build.gradle.kts), [ci.yml](https://github.com/coil-kt/coil/blob/3.6.3/.github/workflows/ci.yml), [libs.versions.toml](https://github.com/coil-kt/coil/blob/3.6.3/gradle/libs.versions.toml) (`kotlin = "2.4.10"`), tag `3.6.3`.
26. **BCV is in maintenance mode, root-applied, and its klib support is experimental.** "This plugin is in maintenance mode."; "It is enough to apply the plugin only to the root
    project build file; all sub-projects will be configured automatically."; "The KLib validation support is experimental and is a subject to change".
    [BCV README](https://github.com/Kotlin/binary-compatibility-validator/blob/0.18.2/README.md), tag `0.18.2` (2026-09-02).
27. **Stabilisation of KGP's tool is open.** KT-71172 "Stabilize migrated ABI Validation in Kotlin", state Submitted: "After stabilization, the separate plugin is officially
    deprecated. Also, all experimental annotations are removed from ABI validation DSL and API."
    [KT-71172](https://youtrack.jetbrains.com/issue/KT-71172), read 2026-10-20.
28. **Gitattributes patterns follow gitignore rules; `eol=lf` fixes the working-tree ending.** "The rules by which the pattern matches paths are the same as in `.gitignore` files";
    "If there is a separator at the beginning or middle (or both) of the pattern, then the pattern is relative to the directory level of the particular `.gitignore` file itself.";
    "A leading "`**`" followed by a slash means match in all directories."; "This setting uses the same line endings in the working directory as in the index when the file is checked out."
    [gitattributes](https://git-scm.com/docs/gitattributes), [gitignore](https://git-scm.com/docs/gitignore), read 2026-10-20.
29. **EditorConfig exempts empty files from the final-newline rule.** "Editors must not insert newlines in empty files when saving those files, even if `insert_final_newline = true`."
    [EditorConfig specification](https://spec.editorconfig.org/), read 2026-10-20.
30. **KGP's js target does not yet support Isolated Projects.** Test helper `disableIsolatedProjectsBecauseOfJsAndWasmKT75899` in `AbiValidationKmpIT`; KT-75899 title "Support Gradle Project Isolation in KGP JS & Wasm".
    [AbiValidationKmpIT.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin-integration-tests/src/test/kotlin/org/jetbrains/kotlin/gradle/abi/AbiValidationKmpIT.kt), `v2.4.21`; [KT-75899](https://youtrack.jetbrains.com/issue/KT-75899), read 2026-10-20.

## 8. Known gaps

- **No execution.** The helper, the `mustRunAfter` line and every behaviour of the both-empty build are established from source, from KGP's
  own tests (which cover the empty JVM project, the empty KMP project with a different target set, and the JVM-only KMP project) and from
  GeneralTools' single-module observation. A KMP module with both `jvm()` and `js` and no source at all, on Gradle 9.7.1 or 9.8.2, was not
  run by anyone found; 6.5 steps 1 to 4 settle it.
- **`mustRunAfter` is unproven here.** The Kotlin team uses it for the same message (KT-84365), and the Gradle message lists it as a
  remedy, but it was not run against `checkKotlinAbi`/`updateKotlinAbi` on Gradle 9.7.1 (step 4). If it does not work, the fallback is
  to drop the line and keep separate invocations in `CONTRIBUTING.md`.
- **Header-only klib dumps.** What the klib dump holds once the web klib exists but carries only `internal` declarations is not observed
  (KGP's tests show a header-only reference file as input, `EMPTY_DUMP_KLIB`, not as output); `graphics-core.klib.api` may go from 0 bytes to
  a header-only file at G2 even if no public declaration lands.
- **Node.js/Yarn on the first `updateKotlinAbi`.** Whether `compileKotlinJs` (a dependency of the dump task through the js main
  compilation's output) triggers the Node.js/Yarn download on a clean machine was not established. `windows-latest` downloads them for
  the tests anyway.
- **Gradle 9.8.2.** Nothing ABI-specific was found, but KGP's documented ceiling is 9.7.0 and its tests do not run 9.8; the claim in F9 is an
  inference from the absence of version logic.
- **Configuration cache.** Supported by the evidence in F2 (task design and KGP's CI defaults), not run in this build. The js target in
  the same build was not checked for configuration-cache problems of its own.
- **Stale-reference blindness** (F4) rests on reading the check loop, not on a test.
- **Rendering-settings churn.** That a KGP bump can rewrite committed dumps' header lines is an inference from the header recording the
  settings; no KGP release note on it was found.
- **Not examined:** the Android target's ABI wiring (`KotlinAndroidTarget` and the new Android KMP library target; KT-83410 is open for
  AGP 9 built-in Kotlin), BCV-versus-KGP dump differences for this repository's code (no baseline exists), SQLDelight and kotlinx.serialization/datetime
  setups beyond the BCV mention.
- **Contradicts or qualifies earlier material:** GeneralTools' `.gitattributes` comment ("compared byte-for-byte") and its root-anchored
  pattern (F5); the question-1 statement that nothing blocks Isolated Projects (the js target does, KT-75899); the bare `explicitApi()`
  lines in the question-1 and question-2 convention listings (replaced by `guardPublicApi()`). Nothing else in questions 1 and 2 or in
  GeneralTools' research is contradicted; the GeneralTools observations on the 0-byte klib file, the missing-file failure and the
  one-invocation failure are confirmed by source.

## 9. Considerations

None. Every point above has a recommendation that does not need a decision from the maintainer.
