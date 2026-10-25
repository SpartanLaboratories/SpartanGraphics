# CI, the release workflow and the protection of `main`

- **Issue:** [SpartanLaboratories/SpartanGraphics#3](https://github.com/SpartanLaboratories/SpartanGraphics/issues/3), question 7 of the
  [research README](README.md).
- **Read:** 2026-10-20. Pinned sources: GitHub Docs `github/docs@3a444c04d005e5d91e29ee0658fd5c9bbeaecf24`, GitHub REST description
  `github/rest-api-description@755ee33b4a81d51f6f07a902cce26cbec3b822f2`, `gradle/actions` `v6.4.0`
  (`3f5f9adaf7d9fecd50b5935e54106014257a94e6`), `actions/checkout` `v7.0.1`, `actions/setup-java` `v6.1.0`, `actions/upload-artifact`
  `v7.0.2`, runner images `actions/runner-images@68bd77adbdf0017e6e3d622348abdb2763ca26f4` (Windows 2025 image `20261013.288.2`, Ubuntu 24.04
  image `20261012.334.1`, macOS 26 arm64 image `20261013.0424.1`), Kotlin Gradle plugin `v2.4.21`, Gradle `v9.8.2` sources, go-github `v93.0.0`,
  Okio `c0ac50617c4dd03efd13a78d882c4462a57bf543`, Coil `3.6.3`, MyGameTools `dc5214f`, GeneralTools `329590821b7bd16b7e68ce58c6c9f3d78f2606ca`
  (`.github/workflows/Build.yml`), and GeneralTools' `kmp-build-and-kotlin-compat.md` F13 at
  [`d424aaf`](https://github.com/SpartanLaboratories/GeneralTools/blob/d424aaf/docs/plans/7-kotlin-multiplatform/research/kmp-build-and-kotlin-compat.md).
- **Method limit:** nothing was run: no Gradle task, no workflow, and no `gh` call other than reads (`GET`). Every workflow line, every task name
  and every API payload below comes from documentation, source at the pinned refs, public read-only API output, or the two finished runs
  that the testing finding already used. What is untested is listed in *Known gaps*.

## 1. Question

How is the CI workflow, the release workflow and the protection of `main` set up for `SpartanLaboratories/SpartanGraphics` (public, GitHub
Free organization, one maintainer) so that every pull request into `main` is gated by `windows-latest` running the full build and every
non-GL test, `ubuntu-latest` and `macos-latest` compiling, the Dokka aggregate and the ABI checks, all on Temurin JDK 21, under exactly named
required checks; a pushed `v`-prefixed annotated tag produces a GitHub Release and publishes nothing; no registry credential exists in CI;
the organization's `CONTRIBUTING.md` baseline (pull requests only, merge commits only, "no registry credentials live in CI") governs; and the
settings steps name their checks and commands exactly and run only once the maintainer confirms them.

## 2. Bottom line

- **One workflow, `ci.yml`, with four plain jobs whose names are the required checks:** `Build (windows-latest)` (runs
  `./gradlew build :build-logic:check :dokkaGeneratePublicationHtml javadocJars --continue --stacktrace`), `Compile (ubuntu-latest)` and
  `Compile (macos-latest)` (each runs `./gradlew assemble testClasses jvmTestClasses jsTestClasses`, main and test sources of every module, no
  `check`, so no ABI check, no Dokka and no JS test run), and `build-logic (TestKit)` (the three nested suites from the testing finding).
  Triggers are `pull_request` (no branch or path filter), `push` to `main`, and `workflow_dispatch`; `concurrency` cancels superseded runs on
  pull requests only. Every job pins Temurin 21, the full-length-SHA-pinned actions, and `gradle/actions/setup-gradle` with
  `cache-provider: basic` (MIT; the default provider needs acceptance of Gradle's terms for v6).
- **`release.yml`** runs on a pushed `v*` tag: a Windows job checks that the tag is annotated and equals `v` plus `version=` in
  `gradle.properties`, then runs the same Gradle gate as CI; then a second job with `contents: write`, no checkout and no third-party action,
  runs `gh release create "$GITHUB_REF_NAME" --verify-tag --title … --generate-notes`. It uploads no asset, calls no `publish*` task and references
  no secret.
- **Protection of `main` is classic branch protection**, one idempotent `PUT .../branches/main/protection`: pull request required with 0
  approvals and stale-review dismissal, the four checks bound to the GitHub Actions app (id 15368), `strict: true`, force pushes and deletion
  off, linear history off (it would forbid merge commits), `enforce_admins: true`. It is applied after the checks have run once (the G1 pull
  request's own run), because the REST API fills an omitted `app_id` from a check that "recently" ran and the UI cannot select a check that never
  ran.
- **Repository settings are separate, individually confirmable commands:** merge-commit-only methods with branch deletion and PR-title merge
  subjects (the baseline), the `GITHUB_TOKEN` default lowered from today's `write` to `read`, and an optional `sha_pinning_required`.
- **Two things in earlier findings need a change:** the TestKit and `build-logic` `Test` tasks are `@CacheableTask`s whose inputs omit the build
  they test, and the repository sets `org.gradle.caching=true`, so a cached run can mask a violation (`outputs.upToDateWhen { false }` does not
  prevent a cache load); and the testing finding's sample CI job uses `actions/upload-artifact@v4` and no cache provider where this finding fixes
  `v7.0.2` and `cache-provider: basic`.

## 3. Findings

Tiers: **consensus** = GitHub or Gradle documentation, source or a standard at the pinned ref; **practice** = what mature repositories
visibly do (read from their files or from public API output); **opinion** = one source; **inference** = derived here from sources that
do not state it.

### F1. A required-check name is the job's name; matrix and reusable-workflow jobs add to it (consensus for plain jobs; practice for matrices)

- GitHub: "When defining status checks, the name format depends on the type of check: **Workflow**: The name format is `<job name>`.
  **Reusable workflow**: The name format is `<job name> / <reusable job name>`." and "Required status checks do not take workflow, matrix,
  or event trigger types into account"
  ([troubleshooting-rules.md](https://github.com/github/docs/blob/3a444c04d005e5d91e29ee0658fd5c9bbeaecf24/content/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/troubleshooting-rules.md)).
  The workflow's `name:` is not part of the check name.
- "make sure that job names are unique across all workflows. Using the same job name in multiple workflows can cause ambiguous status check
  results and block pull requests from being merged"
  ([about-protected-branches.md](https://github.com/github/docs/blob/3a444c04d005e5d91e29ee0658fd5c9bbeaecf24/content/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches.md)).
  The jobs of `ci.yml` and `release.yml` therefore have distinct names.
- A matrix job's check name appends the matrix values: Okio's required checks read `all-platforms (windows-latest)`, `all-platforms (ubuntu-latest)`,
  `jvm (17)` (a job id with a matrix and no `name:`), and Coil's ruleset requires `Instrumentation tests (26)` for a job named
  `Instrumentation tests` (public API output read 2026-10-20 from `GET /repos/square/okio/branches/main` and
  `GET /repos/coil-kt/coil/rules/branches/main`). What a `name:` that itself contains `${{ matrix.* }}` produces was not read from a live
  example, so the design uses plain jobs (one per OS) and no matrix; the two compile jobs will also diverge (Linux hosts the GL tests later).

### F2. Path-filtered or branch-filtered workflows leave required checks pending; skipped jobs pass; `workflow_dispatch` runs satisfy nothing (consensus)

- "A workflow is skipped by path filtering, branch filtering, or a commit message | Associated checks stay in a "Pending" state and block
  merging | Avoid requiring workflows that can be skipped." "A job is skipped by a conditional | The job reports "Success"." "A job depends on
  a failed job | The dependent job is skipped and may not block merging | Use `always()` with `needs` for required checks that depend on other
  jobs." "Successful check statuses are `success`, `skipped`, and `neutral`."
- "if a workflow is triggered by `workflow_dispatch` on a pull request's head branch, checks reported by its jobs do not appear in the pull
  request's checks section. Even if the checks pass for the head commit, they do not satisfy a required status check in a branch ruleset."
  Eligible events are `push`, `pull_request`, `pull_request_review`, `pull_request_target`, `deployment`, `deployment_status` (and
  `merge_group` for merge queues)
  ([troubleshooting-required-status-checks.md](https://github.com/github/docs/blob/3a444c04d005e5d91e29ee0658fd5c9bbeaecf24/content/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks.md)).
  `workflow_dispatch` is therefore only a manual convenience on `ci.yml`.
- Consequence: `ci.yml` has no `paths:` or `branches:` filter on `pull_request`, so the four jobs always report. The safe pattern when a
  job must be skipped for documentation-only changes is a job-level or step-level condition inside a workflow that always starts (section
  6.8), never a workflow-level filter.

### F3. Required checks: selection, the seven-day rule, binding to an app, and what the API does with a check that never ran (consensus on the documents; inference on the never-ran case)

- "A required status check must have completed successfully in the chosen repository during the past seven days." (same troubleshooting page.)
  "When you add a required status check, you can select an app that has recently set this check as the expected source of status updates. If the
  status is set by any other person or integration, merging won't be allowed." and "Any person or integration with write permissions to a
  repository can set the state of any status check in the repository" (about-protected-branches.md). Without the binding, anyone with write
  access can post a commit status named `Build (windows-latest)` and satisfy the rule.
- The GitHub Actions app is id **15368**: `GET /apps/github-actions` returns `{"id":15368,"name":"GitHub Actions","slug":"github-actions"}`, and
  every check run of Okio's latest commit lists `app.id` 15368.
- REST (`PUT /repos/{owner}/{repo}/branches/{branch}/protection`, `required_status_checks.checks[]`): `context` is "The name of the required
  check" and `app_id` is "The ID of the GitHub App that must provide this check. Omit this field to automatically select the GitHub App that has
  recently provided this check, or any app if it was not set by a GitHub App. Pass -1 to explicitly allow any app to set the status."
  ([OpenAPI description](https://github.com/github/rest-api-description/blob/755ee33b4a81d51f6f07a902cce26cbec3b822f2/descriptions/api.github.com/api.github.com.2022-11-28.json)).
  The `context` is a free-form string; nothing in the schema requires a prior run. So the UI limitation ("cannot be selected until it has run") is
  not an API limitation, but the app resolution depends on a prior run: with `app_id` omitted and no prior run, the documented result is "any
  app" (inference from that sentence; this is what MyGameTools' live protection shows: `"app_id": null` on all four checks). Whether an
  explicit `app_id: 15368` is accepted for a check that never ran was not established. The order of operations (6.7) sidesteps both: apply
  after the pull request's own run, with `app_id: 15368` explicit.
- The OpenAPI schema lists `contexts` (deprecated) as required beside `strict`; go-github `v93.0.0` documents the working rule:
  "(Deprecated. Note: only one of Contexts/Checks can be populated, but at least one must be populated)". The payload therefore sends `checks`
  only. The read-back `GET` in 6.4 is the confirmation.

### F4. Classic branch protection and repository rulesets coexist; both are in use by mature Kotlin libraries (consensus on the mechanisms; practice on the choice)

| Repository (default branch, read 2026-10-20) | Mechanism | Required checks (source) | Admins | PRs |
| --- | --- | --- | --- | --- |
| MyGameTools (`master`) | classic | four job names, `app_id: null`, `strict: true` | `enforce_admins: false` | 0 approvals, `dismiss_stale_reviews: true`, `required_conversation_resolution: true` |
| Okio (`main`) | classic | eight Actions matrix names, `app_id: null` | `enforcement_level: non_admins` | not read |
| Ktor (`main`) | classic | six checks from TeamCity (app id 2396224), not Actions | `non_admins` | not read |
| kotlinx-io (`master`) | classic | one aggregated check from TeamCity (app id 888181) | `non_admins` | not read |
| Coil (`main`) | **ruleset** `main` (id 24400516), `include: ["refs/heads/main"]`, no bypass actors | nine Actions checks with `integration_id: 15368`, `strict_required_status_checks_policy: false`, `do_not_enforce_on_create: true` | applies to all (empty bypass list) | 1 approval, `allowed_merge_methods: [squash, rebase]`, plus `non_fast_forward` and `deletion` rules |

- Docs: both are supported and "Rulesets and branch protection rules can both protect branches in a repository. They work alongside each
  other, and all applicable rules are enforced"; rulesets add layering, a status that can be switched without deleting, visibility to anyone
  with read access, and bypass lists with a `pull_request` mode ("`pull_request` means that an actor can only bypass rules on pull requests").
  `evaluate` mode "is only available with GitHub Enterprise" (REST description), and this organization is on the `free` plan
  (`GET /orgs/SpartanLaboratories`, `plan.name`), so a dry run is not available for either mechanism.
- Classic protection has every setting this repository needs, as one idempotent `PUT` that replaces the rule and can be read back with one
  `GET`; a ruleset is created with `POST` and updated by id. The organization's baseline names "branch protection" and MyGameTools uses it.
  Classic is therefore the recommended mechanism; the equivalent ruleset body is in 6.4.4.

### F5. Admins are exempt from classic protection unless told otherwise (consensus); the large libraries leave them exempt (practice)

- "By default, the restrictions of a branch protection rule don't apply to people with admin permissions to the repository or custom roles with
  the "bypass branch protections" permission. You can optionally apply the restrictions to administrators" (about-protected-branches.md).
  Okio, Ktor, kotlinx-io and MyGameTools leave admins exempt; each has several maintainers.
- In a one-maintainer repository the only person who pushes is the admin, so an exempt admin leaves "never receives direct commits" unenforced.
  `enforce_admins: true` makes the rule bind, and costs the "merge without waiting" override. The recovery paths are one `DELETE` and one
  `POST` on `.../protection/enforce_admins` (6.4.3). Presented as Consideration C1.

### F6. The baseline's merge strategy maps to four repository fields; the repository differs on three of them today (consensus on the API; facts read 2026-10-20)

- `CONTRIBUTING.md` (organization `.github`, "Merge strategy"): "On GitHub only “Create a merge commit” is enabled — “Squash” and “Rebase and
  merge” are turned off"; "Pull requests" step 3: the title "**must be a valid Conventional Commit** (it becomes the merge-commit subject)";
  step 6: "Merge, then delete the branch".
- REST (`PATCH /repos/{owner}/{repo}`): `allow_merge_commit`, `allow_squash_merge`, `allow_rebase_merge`, `delete_branch_on_merge`,
  `merge_commit_title` (`PR_TITLE` | `MERGE_MESSAGE`), `merge_commit_message` (`PR_BODY` | `PR_TITLE` | `BLANK`; `merge_commit_title` is
  "Required when using `merge_commit_message`").
- Today `SpartanLaboratories/SpartanGraphics` has `allow_merge_commit: true`, `allow_squash_merge: true`, `allow_rebase_merge: true`,
  `delete_branch_on_merge: false`, `merge_commit_title: MERGE_MESSAGE`, `merge_commit_message: PR_TITLE`, which is why `git log` shows
  "Merge pull request #10 from SpartanLaboratories/docs/roadmap". MyGameTools has `allow_squash_merge: false`, `allow_rebase_merge: false`,
  `delete_branch_on_merge: true`, `merge_commit_title: PR_TITLE`, `merge_commit_message: PR_BODY`, `allow_auto_merge: true`,
  `allow_update_branch: true`. GeneralTools and the organization's `.github` repository have the same defaults as SpartanGraphics today.
- With `strict: true` the "Update branch" control already appears for a pull request that is behind (the baseline's "Update with rebase");
  `allow_update_branch` only extends it to branches that are not required to be current, so it is not part of the commands.
- **Do not enable "Require linear history":** "Enforcing a linear commit history prevents collaborators from pushing merge commits ... any
  pull requests merged into the protected branch must use a squash merge or a rebase merge." It contradicts the baseline.

### F7. Action versions on 2026-10-20 (consensus: the release lists); GeneralTools' F13 holds, with one addition

| Action | Latest release (published) | Commit of the tag (`GET /repos/{o}/{r}/commits/{tag}`) |
| --- | --- | --- |
| `actions/checkout` | v7.0.1 (2026-07-20) | `3d3c42e5aac5ba805825da76410c181273ba90b1` |
| `actions/setup-java` | v6.1.0 (2026-10-14) | `b843ebf9b913146b38da2eaa7aeddf76074ae1ad` |
| `gradle/actions` (`setup-gradle`, `wrapper-validation`) | v6.4.0 (2026-09-28) | `3f5f9adaf7d9fecd50b5935e54106014257a94e6` |
| `actions/upload-artifact` | v7.0.2 (2026-10-07) | `cf430e030ddbb5b0abf93d22962f4752f3646cd9` |

- GeneralTools F13 (checkout v7.0.1, setup-java v6.1.0, `gradle/actions` v6.4.0, read 2026-10-15) is unchanged; `actions/upload-artifact` v7.0.2
  is the addition (v7 adds direct single-file uploads and ESM; v7.0.2 improves retry on HTTP 429). No release newer than these exists.
- Releases are listed out of date order (checkout v6.1.0 and v5.1.0 are dated after v7.0.1 because they are maintenance backports); "latest"
  is the one `GET /releases/latest` returns, and it equals the first row above for each action.
- `gradle/actions` tags are annotated, so `GET /git/ref/tags/v6.4.0` returns the tag object `b9bee63e…`, not the commit; `GET /commits/v6.4.0`
  (above) returns the commit that `uses:` needs. Okio's `build.yml` uses the same `3f5f9ada…` for `setup-gradle` and `wrapper-validation`.

### F8. Pinning: GitHub's guidance, the field, and the missing automation (consensus on the guidance; practice split)

- GitHub: "Pin actions to a full-length commit SHA ... This helps mitigate the risk of a bad actor adding a backdoor to the action's
  repository, as they would need to generate a SHA-1 collision" and "Pin actions to a tag only if you trust the creator ... there is risk to
  this approach even if you trust the author, because a tag can be moved or deleted"
  ([secure-use.md](https://github.com/github/docs/blob/3a444c04d005e5d91e29ee0658fd5c9bbeaecf24/content/actions/reference/security/secure-use.md)).
  The repository setting `sha_pinning_required` (REST `PUT /repos/{owner}/{repo}/actions/permissions`) enforces it; it is `false` here.
- Field: Okio pins every action by SHA with a version comment (`actions/checkout@3d3c42e… # v7.0.1`); Coil, GeneralTools and MyGameTools use
  major tags (`@v7`, `@v6`, `@v4`).
- The same page says that for repositories that adopt a lockfile "SHA pinning ... is no longer necessary" (`.github/workflows/actions.lock`,
  generated by `gh actions-lock`); the concept page marks it "in public preview and subject to change", and its how-to lists
  GitHub Enterprise Cloud as the product. It is not used.
- No `dependabot.yml` exists, so nothing bumps pins; a SHA pin is bumped by hand (6.9). Where it matters least: the only job with a write
  token (`github-release`) runs no third-party action at all (F15), and the jobs that run actions hold a read-only token and no secret.
  Presented as Consideration C2.

### F9. `setup-gradle` v6: providers, terms, caches, wrapper validation, scans (consensus; `gradle/actions` v6.4.0)

- **Caching provider.** v6.0.0: "By upgrading to v6, you accept the Terms of Use for the `gradle-actions-caching` component." That component is
  the default `cache-provider: enhanced` ("proprietary", "free forever" for public repositories under Gradle's terms); "`basic`: A fully
  open-source (MIT) caching implementation built on the standard GitHub Actions cache" which "Uses the same caching strategy as `actions/setup-java`
  with `cache: gradle`", and "The `gradle-actions-caching` component is used only when enhanced caching is enabled and is not loaded or used with
  basic caching or when caching is disabled" (DISTRIBUTION.md). GeneralTools already runs `cache-provider: basic` and records why ("the build
  gains nothing from" enhanced). Basic saves `~/.gradle/caches` and `~/.gradle/wrapper` under the key `setup-java-{RUNNER_OS}-{arch}-gradle-{hash}`
  (hash of `**/*.gradle*`, `**/gradle-wrapper.properties`, `gradle/*.versions.toml`, ...), has no restore keys, no
  `gradle-home-cache-includes` and no cleanup.
- **Who writes the cache.** "By default, The `setup-gradle` action will only write to the cache from Jobs on the default (`main`/`master`) branch.
  Jobs on other branches will read entries from the cache but will not write updated entries." Pull-request runs read what the last `push` to
  `main` stored, so `ci.yml` must run on `push` to `main`, and with basic caching a pull request that edits a build file misses the cache.
- **Configuration cache.** `org.gradle.configuration-cache=true` stays on. `setup-gradle` does not persist configuration-cache entries: the
  input `cache-encryption-key` says "Configuration-cache data will not be saved/restored without an encryption key being provided", v6.0.0
  removed "Existing, rudimentary, configuration-cache support", and the entry lives in the project's `.gradle/`, not in the Gradle user home.
  Every CI run therefore stores a fresh entry and never reuses one; that still fails the build on a configuration-cache problem
  (the reuse side is tested by the `ConfigurationCacheTest` of the testing finding). No encryption key and no secret is introduced.
- **Wrapper validation is built in.** "By default, this action will perform the same wrapper validation as is performed by the dedicated
  wrapper-validation action" (`validate-wrappers: true`); the separate `gradle/actions/wrapper-validation` is needed only where `setup-gradle`
  is not used. Known checksums are bundled, and `fetchUnknownChecksums` reads `https://services.gradle.org/versions/all` at run time for any
  version "not known", so a wrapper newer than the action (for example 9.8.2, published a day before this was read) still validates.
  `distributionSha256Sum` in `gradle-wrapper.properties` additionally pins the distribution (toolchain finding, 6.1).
- **Scans and reporting.** A Job Summary and `GITHUB_STEP_SUMMARY` report are on by default (`add-job-summary: always`) and cost nothing;
  v6.4.0 also annotates Gradle versions that are two majors old (warning) or one major / more than two minors old (notice), so 9.7.1 against
  the current 9.8.2 raises nothing. Publishing a Build Scan to `scans.gradle.com` needs the workflow to declare `build-scan-terms-of-use-url:
  'https://gradle.com/terms-of-service'` and `build-scan-terms-of-use-agree: 'yes'` (and a build that applies the Develocity plugin), which is
  the terms prompt this design avoids: no scan is published.
- `--stacktrace` is what every sibling passes; `--continue` lets one run report the ABI, test and Dokka failures together instead of the first.

### F10. Node.js and Yarn live under the Gradle user home, outside the default cache; whether compile jobs trigger them is not established (consensus on locations; unresolved on triggers)

- Locations and versions: GeneralTools F12/F13 (Node default 24.16.0 for `js`, Yarn classic, both under `<GRADLE_USER_HOME>`) and the testing finding F4
  (`BaseNodeJsRootExtension.installationDir = gradleHome.resolve("nodejs")`; `KotlinNodeJsIr.configureTestDependencies` makes the js **test**
  task depend on `nodeJsSetupTaskProvider` and `npmInstallTaskProvider`). Neither `nodejs` nor `yarn` is among the directories
  `setup-gradle` saves (`caches`, `notifications`, wrapper distributions), and the basic provider cannot add directories. A re-download of tens of
  megabytes per run is accepted (GeneralTools records caching them as its follow-up #18); a separate `actions/cache` for those two paths is possible
  and not part of this design.
- KGP `v2.4.21` `NpmResolverPluginApplier`: any task that is a `RequiresNpmDependenciesTask` with a non-empty `requiredNpmDependencies` (other than
  `KotlinJsTest`, handled through the test framework) gets `dependsOn(nodeJsRoot.npmInstallTaskProvider)`. Which js tasks are such tasks was not
  traced. GeneralTools' serial `./gradlew build` log (run 39507508729) shows `kotlinNodeJsSetup` after `compileKotlinJs` and before `jsMainClasses`,
  `jsJar` and `kotlinNpmInstall` before `assemble`, which neither proves nor rules out that `assemble` depends on them.
- Consequence: the Ubuntu and macOS jobs may still download Node and Yarn. That costs seconds and network, not correctness. The first CI
  run's log settles it (6.7 step 3): search for `kotlinNodeJsSetup` in the two compile jobs.

### F11. The compile task set (consensus on the task names; GeneralTools' run shows them executing on KGP 2.4.21)

- KGP `v2.4.21` registers a lifecycle task per compilation: `KotlinCreateLifecycleTasksSideEffect` ("Assembles outputs for compilation '…' of
  target '…'") for non-JVM compilations, and for a JVM compilation through the Java source set; the name is
  `lowerCamelCaseName(target.disambiguationClassifier, compilationName, "classes")` (`DefaultKotlinCompilationTaskNamesContainerFactory`;
  native compilations use `…klibrary`, irrelevant here). `KotlinJsIrSourceSetProcessor.doTargetSpecificProcessing` makes it `dependsOn(kotlinTask)`.
  GeneralTools' build log (KGP 2.4.21, `jvm` + `js` + `wasmJs`) lists `:jsMainClasses`, `:jsTestClasses`, `:jvmTestClasses`, `:compileTestKotlinJs`,
  `:compileTestKotlinJvm`.
- For `graphics-core`: `assemble` (jars, klib jar, metadata jar) plus `jvmTestClasses` and `jsTestClasses` (compile of test sources and
  processing of test resources). For `graphics-desktop` (`kotlin("jvm")` with `java-library`): `assemble` plus `testClasses`. Gradle's unqualified
  task names run in every project that has them, so the one command `./gradlew assemble testClasses jvmTestClasses jsTestClasses` covers all
  three modules; the BOM contributes its `assemble` only.
- `check` and `build` are excluded: they run `jsNodeTest`, whose dependencies download Node, Yarn and the npm packages even with no test
  (testing finding F4), and they run the ABI check and `verifyDokkaSourceSets` that the Windows job owns. The js klib ABI dump is
  host-independent (no native target), so a second operating system adds nothing (ABI finding, 6.4).
- The list must grow with targets (an iOS or Android target adds its own `…TestClasses`). A `compileAll` lifecycle task registered by the two
  library conventions would remove that edit from the workflow; it is a `build-logic` change and is noted for the build-logic owner (6.10), not
  assumed here.

### F12. Windows specifics (mixed tiers, each stated)

- **Shell.** The default shell on `windows-latest` is PowerShell; `./gradlew` works unquoted in it. Okio's `all-platforms` job runs
  `./gradlew build` and `./gradlew build -Dkwasm=false` on `windows-latest` with the default shell, and its check `all-platforms (windows-latest)`
  succeeds on `c0ac506` (practice). PowerShell splits an unquoted `-Dname=a.b` or `-PtestLevel=a,b` (a comma builds an array); CI passes neither.
  Scripts that need Unix tools use `shell: bash` (Git for Windows `bash` is on the image; `gh` 2.102.0 is installed on the Windows and Ubuntu images).
- **Executable bit.** A file added from Windows is committed as `100644`; `./gradlew` on Linux and macOS then fails with "Permission denied".
  GeneralTools had to fix it (commit `3295908`, "an executable wrapper"; `git ls-files -s gradlew` there prints `100755`). The fix is
  `git update-index --chmod=+x gradlew`, verified with `git ls-files -s gradlew` before the first push; `ci.yml` also checks it on Linux with a
  message that names the repair.
- **Line endings.** Two comparisons that CI relies on ignore them: the ABI check compares line by line (ABI finding, F5) and KGP's lock-file
  comparison is `contentEqualsIgnoringLineEndings(value.asFile, outputFile)` (`LockStoreTask.copy`, KGP `v2.4.21`), so a CRLF checkout cannot
  fail either. Whether the runner checks out CRLF is an inference (the Windows image installs Git for Windows silently without an auto-CRLF
  option, so the installer default `core.autocrlf=true` should apply); the repository's `.gitattributes` is what controls it. `gradlew` (a shell
  script) stays LF through `.gitattributes`; Windows runs `gradlew.bat`. The release job strips `\r` when it reads `version=` from
  `gradle.properties`.
- **Path length.** The Windows image sets `LongPathsEnabled` to 1 (`Configure-BaseImage.ps1`); the workspace is `D:\a\SpartanGraphics\SpartanGraphics`.
  The testing finding's TestKit workspace and Kotlin/JS `node_modules` are the long paths; unmeasured (Known gaps).

### F13. Hosted runners (consensus: GitHub Docs and runner-image READMEs)

- Labels: `ubuntu-latest` is Ubuntu 24.04, `windows-latest` Windows Server 2025, `macos-latest` macOS 26 arm64 (3 M1 cores, 7 GB RAM), per
  [supported-github-runners.md](https://github.com/github/docs/blob/3a444c04d005e5d91e29ee0658fd5c9bbeaecf24/data/reusables/actions/supported-github-runners.md).
  Linux and Windows have 4 CPUs, 16 GB RAM and 14 GB SSD. "Use of the standard GitHub-hosted runners is free and unlimited on public
  repositories." `-latest` "might not be the most recent version of the operating system"; the label moves when GitHub moves it, which can
  change a build without a commit (a reason the macOS job is a compile gate that fails loudly and is not a release input).
- Preinstalled JDKs: JDK 21.0.12 is installed on all three (the READMEs do not name the vendor), but the default is 17 on Ubuntu and Windows and 21 on macOS
  (`JAVA_HOME_17_X64` default; `JAVA_HOME_21_arm64` default). `actions/setup-java` with `distribution: temurin` and `java-version: "21"`
  makes 21 the active JDK on every runner, so the Gradle JVM and the toolchain agree.

### F14. Triggers and concurrency (practice; GitHub syntax)

- MyGameTools: `push` to the default branch, `pull_request` to it, `workflow_dispatch`, and `concurrency: group: ci-${{ github.ref }}`,
  `cancel-in-progress: ${{ github.event_name == 'pull_request' }}`. Coil: `group: ${{ github.head_ref || github.run_id }}` with the comment
  "Limit concurrency to 1 for PRs. 'main' concurrency isn't limited." Okio: `pull_request: {}` plus `push: branches: ['main']`.
- `push: branches: [main]` excludes tags, so a release tag does not start `ci.yml`. `pull_request` without a `branches:` filter also covers
  `hotfix/*` and `release/*` branches that target something other than `main`, and a filter would be one of the skip causes in F2.
- A push to `main` must not cancel the previous push's run: that run is the cache writer (F9).

### F15. The release workflow (consensus on the commands; practice from MyGameTools)

- `gh release create` flags: `--verify-tag` "Abort in case the git tag doesn't already exist in the remote repository", `--generate-notes`
  "Automatically generate title and notes for the release via GitHub Release Notes API", `--title` "Release title"; `GH_REPO` is "specify the
  GitHub repository in the `[HOST/]OWNER/REPO` format for commands that otherwise operate on a local repository" and `GH_TOKEN` authenticates
  ([gh manual](https://cli.github.com/manual/gh_release_create), [environment](https://cli.github.com/manual/gh_help_environment), `gh` 2.102.1
  is the latest release; 2.102.0 is on the images). With `GH_REPO` set the release job needs **no checkout**, so the job holding
  `contents: write` runs no repository code and no third-party action.
- MyGameTools' `release.yml` gives the shape: `on: push: tags: ["v*"]`, a `verify` job running `./gradlew build --stacktrace`, then a
  `github-release` job `needs: verify` with `gh release create "$GITHUB_REF_NAME" --title "$GITHUB_REF_NAME" --generate-notes --verify-tag`; it
  grants `contents: write` to the whole workflow. This design grants it to the release job only.
- **Annotated or not** is observable without a checkout: `GET /repos/{o}/{r}/git/ref/tags/{tag}` returns `object.type` `tag` for an
  annotated tag and `commit` for a lightweight one (read: `gradle/actions` `v6.4.0` returns `tag`, `actions/checkout` `v7.0.1` returns `commit`).
  `actions/checkout` is not used for this check: with a shallow checkout the tag may be fetched as a bare commit.
- **The tag must be passed through the environment, not interpolated.** A tag name may contain `$`, backticks, `;` and parentheses; writing
  `${{ github.ref_name }}` inside `run:` splices it into the script. The workflow reads `$GITHUB_REF_NAME`.
- **Which commit is verified.** CONTRIBUTING step 4 tags the merge commit of the release PR on the default branch, which already carried a
  green `push` run. The release job reruns the Windows gate (`build`, `:build-logic:check`, Dokka, `javadocJars`) on the tagged commit, plus a
  version guard (tag equals `v` + `version=`), and does not rerun the TestKit suites (minutes; they ran on the PR and on the push to `main`).
  Hotfix tags sit on `hotfix/<version>` branches, so the tag is not required to be reachable from `main`.
- Publishing nothing: no `publish*` task, no secret, no `id-token`, no asset upload, `permissions: contents: read` at workflow level, and the
  GitHub Release is the only artifact. Maven Central stays a by-hand step (CONTRIBUTING "Releasing" step 6).

### F16. Cached results can mask the build-logic checks (consensus on the facts; inference on the effect)

- Gradle `v9.8.2`: `@CacheableTask public abstract class Test extends AbstractTestTask` (`Test.java`). The repository sets
  `org.gradle.caching=true` (build-logic finding, 6.3), and `setup-gradle` saves `caches/build-cache-1` with the Gradle user home.
- The testing finding's `integrationTest`/`e2eTest`/`nonfunctionalTest` and the Level-1 `RepositoryContractTest` read files that are not
  task inputs (the repository root, passed as a system property). `outputs.upToDateWhen { false }` (testing finding, 6.4) makes the task
  not-up-to-date; it does not opt the task out of the build cache, and Gradle's own guidance is that "using `outputs.upToDateWhen` is
  discouraged and should be replaced by properly declaring the task's inputs". With the repository path identical across runner runs, a pull
  request that changes only `graphics-core/build.gradle.kts`, a dump or `.gitattributes` can produce the same cache key as the last `main` run
  and load the test result from the cache without running any nested build (inference; not run).
- Remedy in `build-logic` (flagged to its owner): `outputs.doNotCacheIf("drives builds outside its declared inputs") { true }` on the three
  heavy tasks, and either the declared repository files as inputs or the same `doNotCacheIf` on `:build-logic:test`. In CI, the TestKit job
  adds `--no-build-cache` as a second line of defence.

## 4. How comparable projects do it

| Project @ ref | Workflow shape | Pinning | Gradle setup | Protection |
| --- | --- | --- | --- | --- |
| Okio `c0ac506` (`.github/workflows/build.yml`) | one workflow; jobs `jvm` (matrix), `emulator`, `loom`, `all-platforms` (matrix `macos-14`, `ubuntu-latest`, `windows-latest`, `./gradlew build`); `pull_request: {}`, `workflow_dispatch`, `push` to `main`, `tags-ignore` | full SHAs with version comments (`checkout@3d3c42e… # v7.0.1`, `gradle/actions/wrapper-validation@3f5f9ada… # v6.4.0`) | separate `wrapper-validation` step, `GRADLE_OPTS` with `-Xmx2g -Dorg.gradle.daemon=false -Dkotlin.incremental=false` | classic, eight matrix-named checks, admins exempt |
| Coil `3.6.3` (`ci.yml`) | jobs `Checks`, `Unit tests`, `Unit tests (macOS)` (a separate job on `macos-26`), `Instrumentation tests`, `Build samples`; ABI check step prints "This PR changes Coil's public API. If this change is intentional, run './gradlew updateKotlinAbi' and commit the new API files." | major tags (`@v7`, `@v6`) | `wrapper-validation@v6` step, `setup-gradle@v6` | ruleset, `integration_id: 15368` on nine checks |
| MyGameTools `dc5214f` | five jobs with descriptive names; separate `release.yml` | major tags (`@v4`) | `setup-gradle@v4` | classic, `enforce_admins: false` |
| GeneralTools `3295908` | one `build` job on `ubuntu-latest`, `on: [push]` | major tags (`@v7`, `@v6`) | `cache-provider: basic` | none |
| Ktor `3.6.0`, kotlinx-io | builds run on TeamCity; Ktor's `.github/workflows` holds only automations and `gradle-wrapper-validation.yml` | n/a | n/a | classic, required checks from TeamCity apps |

Patterns that recur: one workflow with a job per concern; the macOS job separate from Linux; a cheap contributor hint on ABI failure (Coil;
optional here because the failing `checkKotlinAbi` task already names `updateKotlinAbi` in its message); `push` to the default branch so the
cache is written there; matrix names appearing verbatim in required checks.

## 5. Pitfalls and regrets

- **A renamed job deadlocks the pull request.** The old name stays "Expected — Waiting for status to be reported" and, with `enforce_admins:
  true`, cannot be bypassed; update the protection before or in the same step as a rename (6.4.3 gives the one-line `PATCH`).
- **A workflow-level `paths:` filter on a required workflow** blocks documentation-only pull requests (F2). Documentation pull requests are
  frequent here (every plan is committed), so the temptation is real; the in-job pattern in 6.8 is the safe one.
- **`workflow_dispatch` is not a way to "re-run for the check"** (F2); re-run the failed jobs of the pull-request run, or push.
- **`Require linear history`** forbids the baseline's merge commits (F6).
- **A check posted by anyone with write access** satisfies an unbound required check (F3); bind it to app 15368.
- **A cached Test task hides a violation** (F16).
- **`./gradlew` without the executable bit** fails on Linux and macOS only (F12); the bit is part of the first commit.
- **The `contexts` key** in the protection payload next to `checks` (F3).
- **An unbound `GITHUB_TOKEN` default.** The repository's default is `write` today (`GET .../actions/permissions/workflow`); MyGameTools has
  `read`. Every workflow here declares `permissions:` explicitly, so it does not depend on the default, but a future workflow that forgets does.
- **Tag-name injection** into `run:` (F15).
- **A first run with Node downloads on a flaky network.** Node, Yarn and npm packages are fetched from nodejs.org, GitHub releases and the npm
  registry on every clean runner (GeneralTools F13 pitfall 8); a red run from an outage is rerun, not "fixed" by a retry loop in the workflow.
- **`macos-latest` moves** (macOS 26 today) and has 7 GB RAM; the Gradle and Kotlin daemons run at default heap in G1 and have not been sized for
  the later renderer modules.
- **Gradle 9.8.x is outside KGP's "fully supported" line (toolchain finding C1)**; nothing in CI blocks moving to it: wrapper validation fetches
  unknown checksums (F9), and `setup-gradle` raises no annotation for a current Gradle.

## 6. Recommendation for this repository

Use one workflow `ci.yml` of four plain jobs whose names are the required checks, one `release.yml`, classic branch protection with
administrators included, and repository settings applied as separate confirmed commands. The reasoning in three sentences: required-check names
come from job names and nothing else, so plain jobs with fixed names are the safest binding and the two compile jobs will diverge later anyway;
the release job that holds a write token needs nothing but `gh`, which keeps third-party code away from it and makes the choice of how to pin
actions a small one; and in a one-maintainer repository only `enforce_admins: true` makes "the default branch never receives direct commits"
true. Rejected: a matrix for the compile jobs (name behaviour with expressions unproven, the jobs diverge), a single "CI passed" aggregate
job as the only required check (hides which guarantee failed and adds an `always()` subtlety, F2; revisit if the job count grows), workflow-level
path filters (F2), the proprietary default cache provider (F9), a ruleset (F4, equivalent body given), and a Gradle 9.0.0 job beside the TestKit
job (the testing finding's `CompositeConsumerFloorTest` covers it).

### 6.1 Jobs and the checks they define

| Job id | Required check (job `name`) | Runner | Runs | Opens no GL context |
| --- | --- | --- | --- | --- |
| `build` | `Build (windows-latest)` | `windows-latest` | `./gradlew build :build-logic:check :dokkaGeneratePublicationHtml javadocJars --continue --stacktrace` (all tests of every level on every target, `checkKotlinAbi` for both library modules through `check`, `verifyDokkaSourceSets`, the Dokka aggregate, the four javadoc jars, the build-logic unit tiers) | yes: G1 has no test that opens one |
| `compile-ubuntu` | `Compile (ubuntu-latest)` | `ubuntu-latest` | `./gradlew assemble testClasses jvmTestClasses jsTestClasses --stacktrace` | n/a |
| `compile-macos` | `Compile (macos-latest)` | `macos-latest` | the same command | n/a |
| `build-logic-testkit` | `build-logic (TestKit)` | `windows-latest` | `./gradlew :build-logic:integrationTest :build-logic:e2eTest :build-logic:nonfunctionalTest --continue --no-build-cache --stacktrace` | yes |

The release workflow's jobs are `Verify tagged build (windows-latest)` and `GitHub Release`; neither name equals a CI name (F1).
`build-logic (TestKit)` is a required check on every pull request (testing finding C1, option (a)); when the first run measures more than about
15 minutes, 6.8 gives the skip pattern.

### 6.2 `.github/workflows/ci.yml`

```yaml
name: CI

# Every job `name` below is a required status check of main's branch protection. Rename a job only together with that protection.
# Action pins are full commit SHAs; the comment names the release. Bump them by hand (CONTRIBUTING.md, "Bumping the pinned actions").

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions:
  contents: read

env:
  JAVA_VERSION: "21" # equals SpartanBuild.JVM_LEVEL in build-logic

jobs:
  build:
    name: Build (windows-latest)
    runs-on: windows-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: actions/setup-java@b843ebf9b913146b38da2eaa7aeddf76074ae1ad # v6.1.0
        with:
          distribution: temurin
          java-version: ${{ env.JAVA_VERSION }}
      - uses: gradle/actions/setup-gradle@3f5f9adaf7d9fecd50b5935e54106014257a94e6 # v6.4.0
        with:
          cache-provider: basic # MIT provider; the default one needs acceptance of Gradle's terms. Also validates the wrapper jar.
      - name: Build, check the API dumps, build the API docs
        run: ./gradlew build :build-logic:check :dokkaGeneratePublicationHtml javadocJars --continue --stacktrace
      - name: Upload the rendered API docs
        uses: actions/upload-artifact@cf430e030ddbb5b0abf93d22962f4752f3646cd9 # v7.0.2
        with:
          name: api-docs-html
          path: build/dokka/html
          retention-days: 14
      - name: Upload reports of a failed build
        if: failure()
        uses: actions/upload-artifact@cf430e030ddbb5b0abf93d22962f4752f3646cd9 # v7.0.2
        with:
          name: build-reports
          path: "**/build/reports/"
          if-no-files-found: ignore

  compile-ubuntu:
    name: Compile (ubuntu-latest)
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - name: Check that the wrapper script is executable
        run: test -x ./gradlew || { echo "::error file=gradlew::gradlew is not executable in git. Run 'git update-index --chmod=+x gradlew' and commit."; exit 1; }
      - uses: actions/setup-java@b843ebf9b913146b38da2eaa7aeddf76074ae1ad # v6.1.0
        with:
          distribution: temurin
          java-version: ${{ env.JAVA_VERSION }}
      - uses: gradle/actions/setup-gradle@3f5f9adaf7d9fecd50b5935e54106014257a94e6 # v6.4.0
        with:
          cache-provider: basic
      # Main and test sources of every module and target, plus the jars. Not `check`/`build`: those run the Node-backed js test task,
      # the ABI check and the Dokka guard, which the Windows job owns. Add each new target's `<target>TestClasses` here.
      - name: Compile
        run: ./gradlew assemble testClasses jvmTestClasses jsTestClasses --stacktrace

  compile-macos:
    name: Compile (macos-latest)
    runs-on: macos-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - name: Check that the wrapper script is executable
        run: test -x ./gradlew || { echo "::error file=gradlew::gradlew is not executable in git. Run 'git update-index --chmod=+x gradlew' and commit."; exit 1; }
      - uses: actions/setup-java@b843ebf9b913146b38da2eaa7aeddf76074ae1ad # v6.1.0
        with:
          distribution: temurin
          java-version: ${{ env.JAVA_VERSION }}
      - uses: gradle/actions/setup-gradle@3f5f9adaf7d9fecd50b5935e54106014257a94e6 # v6.4.0
        with:
          cache-provider: basic
      - name: Compile
        run: ./gradlew assemble testClasses jvmTestClasses jsTestClasses --stacktrace

  build-logic-testkit:
    name: build-logic (TestKit)
    runs-on: windows-latest
    timeout-minutes: 40
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: actions/setup-java@b843ebf9b913146b38da2eaa7aeddf76074ae1ad # v6.1.0
        with:
          distribution: temurin
          java-version: ${{ env.JAVA_VERSION }}
      - uses: gradle/actions/setup-gradle@3f5f9adaf7d9fecd50b5935e54106014257a94e6 # v6.4.0
        with:
          cache-provider: basic
      # The runner sets CI=true, which makes the nested builds share this Gradle user home (testing-build-logic finding, C2).
      # --no-build-cache: the nested suites drive builds that are not declared task inputs, so a cached result would prove nothing.
      - name: Nested build-logic suites
        run: ./gradlew :build-logic:integrationTest :build-logic:e2eTest :build-logic:nonfunctionalTest --continue --no-build-cache --stacktrace
      - name: Upload reports of a failed run
        if: failure()
        uses: actions/upload-artifact@cf430e030ddbb5b0abf93d22962f4752f3646cd9 # v7.0.2
        with:
          name: build-logic-testkit-reports
          path: build-logic/build/reports/tests/
          if-no-files-found: ignore
```

Notes on the file:

- The compile jobs run `setup-java` after the executable-bit check so a wrapper problem fails in seconds. The Windows jobs need no such check:
  they run `gradlew.bat` through `./gradlew` in PowerShell (F12).
- `JAVA_VERSION` is the one place the workflow names the JDK level; `SpartanBuild.JVM_LEVEL` is the build's own (toolchain finding, 6.3). They are
  kept equal by hand; a mismatch makes the toolchain provision a JDK through the foojay resolver instead of failing.
- No `fetch-depth`, no `persist-credentials` override: nothing in the build reads history or pushes.

### 6.3 `.github/workflows/release.yml`

```yaml
name: Release

# Creates the GitHub Release for a pushed v-prefixed annotated tag. It publishes NOTHING: Maven Central publishing is a manual
# step (CONTRIBUTING.md) and no registry credential exists in CI. This workflow has no secrets and uploads no assets.

on:
  push:
    tags: ["v*"]

concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false

permissions:
  contents: read

env:
  JAVA_VERSION: "21"

jobs:
  verify:
    name: Verify tagged build (windows-latest)
    runs-on: windows-latest
    timeout-minutes: 30
    steps:
      - name: Check that the tag is annotated
        shell: bash
        env:
          GH_TOKEN: ${{ github.token }}
          GH_REPO: ${{ github.repository }}
        run: |
          kind=$(gh api "repos/$GH_REPO/git/ref/tags/$GITHUB_REF_NAME" --jq .object.type)
          if [ "$kind" != "tag" ]; then
            echo "::error::$GITHUB_REF_NAME is a lightweight tag. Releases use annotated tags: git tag -a <tag> -m 'Release <version>'"
            exit 1
          fi
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - name: Check the tag against the project version
        shell: bash
        run: |
          version=$(grep -E '^version=' gradle.properties | head -n 1 | cut -d= -f2 | tr -d '\r')
          if [ "v$version" != "$GITHUB_REF_NAME" ]; then
            echo "::error::Tag $GITHUB_REF_NAME does not equal v$version (version= in gradle.properties)"
            exit 1
          fi
      - uses: actions/setup-java@b843ebf9b913146b38da2eaa7aeddf76074ae1ad # v6.1.0
        with:
          distribution: temurin
          java-version: ${{ env.JAVA_VERSION }}
      - uses: gradle/actions/setup-gradle@3f5f9adaf7d9fecd50b5935e54106014257a94e6 # v6.4.0
        with:
          cache-provider: basic
      - name: Build the tagged commit
        run: ./gradlew build :build-logic:check :dokkaGeneratePublicationHtml javadocJars --continue --stacktrace

  github-release:
    name: GitHub Release
    needs: verify
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: write # the only job with a write token; it runs no checkout and no third-party action
    env:
      GH_TOKEN: ${{ github.token }}
      GH_REPO: ${{ github.repository }}
    steps:
      - name: Create the GitHub Release
        run: gh release create "$GITHUB_REF_NAME" --verify-tag --title "$GITHUB_REF_NAME" --generate-notes
```

Behaviour: a lightweight tag, a tag that differs from `v<version>`, or a red build stops the run before any release exists; the tag then stays
and is deleted and re-created after the fix (`git push --delete origin <tag>`). The `v0.1.0` release at G7 needs the release branch to have set
`version=0.1.0` (CONTRIBUTING "Releasing" step 2), which this guard enforces.

### 6.4 Protection of `main`

#### 6.4.1 What the rule says

| Setting | Value | Why |
| --- | --- | --- |
| Require a pull request | on; `required_approving_review_count: 0`; `dismiss_stale_reviews: true`; no code-owner review | baseline: PRs even solo; "A second approving review becomes required once a second person is on the repository" (6.4.3 has the change) |
| Required status checks | `Build (windows-latest)`, `Compile (ubuntu-latest)`, `Compile (macos-latest)`, `build-logic (TestKit)`, each with `app_id: 15368` | F1, F3 |
| Require branches to be up to date (`strict`) | on | baseline step 5 |
| Conversation resolution | on | parity with MyGameTools |
| Administrators included (`enforce_admins`) | on | F5, C1 |
| Force pushes, deletion | off | baseline: never receives direct commits |
| Linear history, signed commits, push restrictions, branch locking | off | merge commits are the strategy (F6); the rest are not in the baseline |

#### 6.4.2 Payload and commands (apply only after the maintainer confirms; order in 6.7)

`protection-main.json` (a scratch file, not committed):

```json
{
  "required_status_checks": {
    "strict": true,
    "checks": [
      { "context": "Build (windows-latest)", "app_id": 15368 },
      { "context": "Compile (ubuntu-latest)", "app_id": 15368 },
      { "context": "Compile (macos-latest)", "app_id": 15368 },
      { "context": "build-logic (TestKit)", "app_id": 15368 }
    ]
  },
  "enforce_admins": true,
  "required_pull_request_reviews": {
    "required_approving_review_count": 0,
    "dismiss_stale_reviews": true,
    "require_code_owner_reviews": false,
    "require_last_push_approval": false
  },
  "restrictions": null,
  "required_linear_history": false,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "block_creations": false,
  "required_conversation_resolution": true,
  "lock_branch": false
}
```

```bash
gh api --method PUT repos/SpartanLaboratories/SpartanGraphics/branches/main/protection --input protection-main.json

# read back and compare with the table above
gh api repos/SpartanLaboratories/SpartanGraphics/branches/main/protection
```

`--input` avoids shell quoting on Windows PowerShell. `PUT` replaces the whole rule, so a repeat is safe. If the API rejects the body because
`contexts` is absent (the OpenAPI schema lists it as required, go-github says to omit it when `checks` is used), retry with `"contexts": []`
added to `required_status_checks`; if it rejects `app_id` for a check that never ran, run the CI first (6.7).

#### 6.4.3 Later changes, escape hatches and rollback

```bash
# a second maintainer joins: require one approval
gh api --method PATCH repos/SpartanLaboratories/SpartanGraphics/branches/main/protection/required_pull_request_reviews -F required_approving_review_count=1 -F dismiss_stale_reviews=true

# a job is renamed or added: replace the list of required checks (edit the JSON array, then)
gh api --method PATCH repos/SpartanLaboratories/SpartanGraphics/branches/main/protection/required_status_checks --input required-checks.json

# emergency: stop applying the rule to admins, merge, then restore it
gh api --method DELETE repos/SpartanLaboratories/SpartanGraphics/branches/main/protection/enforce_admins
gh api --method POST   repos/SpartanLaboratories/SpartanGraphics/branches/main/protection/enforce_admins

# remove the protection entirely
gh api --method DELETE repos/SpartanLaboratories/SpartanGraphics/branches/main/protection
```

#### 6.4.4 The equivalent ruleset (not recommended; payload derived from the REST schema and Coil's live ruleset, not applied by anyone)

```json
{
  "name": "main",
  "target": "branch",
  "enforcement": "active",
  "bypass_actors": [],
  "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] } },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "pull_request", "parameters": {
        "required_approving_review_count": 0, "dismiss_stale_reviews_on_push": true, "require_code_owner_review": false,
        "require_last_push_approval": false, "required_review_thread_resolution": true, "allowed_merge_methods": ["merge"] } },
    { "type": "required_status_checks", "parameters": {
        "strict_required_status_checks_policy": true, "do_not_enforce_on_create": false,
        "required_status_checks": [
          { "context": "Build (windows-latest)", "integration_id": 15368 },
          { "context": "Compile (ubuntu-latest)", "integration_id": 15368 },
          { "context": "Compile (macos-latest)", "integration_id": 15368 },
          { "context": "build-logic (TestKit)", "integration_id": 15368 } ] } }
  ]
}
```

`gh api --method POST repos/SpartanLaboratories/SpartanGraphics/rulesets --input ruleset-main.json`; later edits are
`PUT .../rulesets/<id>`. Its one advantage here is an admin bypass limited to pull requests (`bypass_actors` with `bypass_mode: "pull_request"`),
which is the weaker rule C1 argues against.

### 6.5 Repository settings (three independent commands; each confirmed or declined on its own)

Facts read 2026-10-20: `allow_merge_commit: true`, `allow_squash_merge: true`, `allow_rebase_merge: true`, `delete_branch_on_merge: false`,
`merge_commit_title: MERGE_MESSAGE`, `merge_commit_message: PR_TITLE`, `allow_auto_merge: false`, `default_workflow_permissions: write`,
`sha_pinning_required: false`.

**B. Merge methods and merge-commit text (the baseline's "Merge strategy" and "Pull requests")**

```bash
gh api --method PATCH repos/SpartanLaboratories/SpartanGraphics \
  -F allow_merge_commit=true -F allow_squash_merge=false -F allow_rebase_merge=false \
  -F delete_branch_on_merge=true \
  -f merge_commit_title=PR_TITLE -f merge_commit_message=PR_BODY
```

Effect: "Squash" and "Rebase and merge" disappear from pull requests; the head branch is deleted on merge; the merge-commit subject
defaults to the pull-request title (a Conventional Commit, per the baseline) and its body to the pull-request body. Declining the last two
fields leaves subjects as "Merge pull request #N from …". `allow_auto_merge` (MyGameTools: true) is left unchanged; add
`-F allow_auto_merge=true` only if wanted.

**C. Workflow token default (hardening; independent of the baseline)**

```bash
gh api --method PUT repos/SpartanLaboratories/SpartanGraphics/actions/permissions/workflow \
  -f default_workflow_permissions=read -F can_approve_pull_request_reviews=false
```

**D. Enforce SHA pinning (only with C2 option (a))**

```bash
gh api --method PUT repos/SpartanLaboratories/SpartanGraphics/actions/permissions -F enabled=true -f allowed_actions=all -F sha_pinning_required=true
```

### 6.6 Triggers, concurrency, permissions: the settled shape

`pull_request` with no filters, `push` to `main` only, `workflow_dispatch` for manual runs; `concurrency` group `ci-${{ github.ref }}` cancelling
only on pull requests; workflow-level `contents: read`; the release workflow adds `contents: write` to one job. No `merge_group` trigger (no
merge queue); add it with a queue. No `schedule`.

### 6.7 Order of operations

1. **Before the first push:** commit `gradlew` as `100755` (`git update-index --chmod=+x gradlew`; check `git ls-files -s gradlew`), and commit
   `.gitattributes` with `gradlew` as `text eol=lf`.
2. **Push the G1 branch and open the pull request.** `ci.yml` runs from the branch; the four jobs create the four check names. Compare the names
   in the pull request's checks list with the strings in 6.1 character for character.
3. **Read the first runs:** the two compile logs for `kotlinNodeJsSetup`/`kotlinNpmInstall` (F10), the job durations (record them in the
   as-built record; the TestKit estimate is 8 to 14 minutes on Linux-class I/O, up to twice that on Windows), the wrapper validation line in the
   `setup-gradle` summary, and `FROM-CACHE` lines for `:build-logic:*test` tasks (F16).
4. **Maintainer confirms B, C (and D if C2(a))**, and they are applied; B before the merge so the merge uses the baseline's commit text and
   deletes the branch.
5. **Merge the pull request** with a merge commit. `main` is not yet protected; the baseline's "every change through a PR" holds by convention
   for this one merge. The `push` run on `main` is the second green run and writes the Gradle cache.
6. **Maintainer confirms the protection; apply 6.4.2** within seven days of a green run (F3), then run the read-back `GET`.
7. **Prove the rule once:** open a trivial pull request: the merge box lists the four checks as Required and blocks until they pass; a direct
   `git push origin HEAD:main` of a new commit is rejected with `GH006: Protected branch update failed`.
8. **Optional rehearsal of the release path:** push an annotated tag `v0.0.0-rehearsal`; the `verify` job stops at the version guard (the tag
   differs from `v0.1.0-SNAPSHOT`), which exercises the tag-shape and version checks; delete the tag. It does not exercise `gh release create`,
   whose first real run is G7 (MyGameTools has run the same command).

### 6.8 Skipping the heavy job for some pull requests without leaving a required check pending

Not applied in G1 (every G1 pull request touches build files). If `build-logic (TestKit)` later needs to skip documentation-only pull requests,
keep the workflow unfiltered and decide inside it:

```yaml
  changes:
    name: Detect build-relevant changes
    runs-on: ubuntu-latest
    outputs:
      relevant: ${{ steps.diff.outputs.relevant }}
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with: { fetch-depth: 0 }
      - id: diff
        run: |
          base="${{ github.event.pull_request.base.sha || github.event.before }}"
          if git diff --name-only "$base" HEAD | grep -qvE '^(docs/|.*\.md$)'; then echo "relevant=true" >> "$GITHUB_OUTPUT"; else echo "relevant=false" >> "$GITHUB_OUTPUT"; fi
  build-logic-testkit:
    needs: changes
    if: needs.changes.outputs.relevant == 'true' # a job skipped by a condition reports "Success" (F2)
```

`Detect build-relevant changes` must itself never fail on a normal pull request, since a failed `needs` job skips the dependent job without a
blocking status (F2). Do not skip `Build (windows-latest)` this way: the ABI dumps and committed docs share paths with code reviews.

### 6.9 Bumping the pinned actions (goes into `CONTRIBUTING.md`)

```bash
gh api repos/actions/checkout/commits/v7.0.1 --jq .sha       # replace the SHA after `uses:` and the version in the comment
gh api repos/gradle/actions/commits/v6.4.0 --jq .sha         # the commit, not the annotated tag object
gh api repos/actions/checkout/releases/latest --jq .tag_name # what is current
```

Do not bump to a pre-release (`v6.4.0-rc.1` exists). Review the release notes; `gradle/actions` v6 majors have changed license terms
before (F9).

### 6.10 Relation to the other findings

- **Consistent with:** toolchain finding 6.1 (wrapper 9.7.1 with `distributionSha256Sum`, validated by `setup-gradle`); ABI finding 6.4 (Windows
  runs `build`; Ubuntu and macOS run compile tasks only); Dokka finding 6.5 (`build :dokkaGeneratePublicationHtml javadocJars` on Windows, nothing on
  the compile jobs); testing finding 6.6 (the `:build-logic:check` addition, the `build-logic (TestKit)` job on `windows-latest`, JDK 21 Temurin,
  its Considerations C1 (always required) and C2 (shared Gradle user home on CI)); GeneralTools F13 (action versions, `cache-provider: basic`,
  exec bit).
- **Corrections to carry into those documents:** (1) the testing finding's sample TestKit job uses `actions/upload-artifact@v4` and no cache
  provider; the job here supersedes it (`v7.0.2`, `cache-provider: basic`, `--no-build-cache`). (2) The testing finding's heavy `Test` tasks need
  `doNotCacheIf` in addition to `upToDateWhen { false }`, and `:build-logic:test` (which reads repository files) needs declared inputs or the
  same `doNotCacheIf` (F16). (3) The toolchain finding's permanent Gradle 9.0.0 job (its Consideration C3) is not a CI job; the testing
  finding replaced it, and this design has none. (4) Optional for the build-logic owner: a `compileAll` lifecycle task in the two library
  conventions would let the compile jobs name one task (F11).
- **Nothing contradicts** the Dokka, ABI, BOM or build-logic findings.

## 7. Considerations

**C1. Are administrators included in the protection of `main`?**
- *(a) Yes, `enforce_admins: true`* — the single maintainer is an admin, so the rule binds the only person who pushes; matches the baseline
  ("never receives direct commits — branch protection enforces this"). Costs the override: a pull request whose required check cannot pass
  (a renamed job, an outage) cannot be merged until the maintainer removes `enforce_admins` (one `DELETE`, 6.4.3) and restores it.
- *(b) No, as MyGameTools, Okio, Ktor and kotlinx-io* — the maintainer can merge past a failing check and can push to `main`; with a single
  maintainer the protection is then advisory.
Recommendation: **(a).** Switch to (b) only when a second maintainer exists and an override is wanted for emergencies.

**C2. How are actions pinned?**
- *(a) Full commit SHA with the version in a comment* (as 6.2 and 6.3), bumped by hand because no `dependabot.yml` exists; optionally enforced
  with `sha_pinning_required` (6.5 D). Follows GitHub's guidance and Okio; the cost is a manual bump of four actions in two files.
- *(b) Major tags* (`@v7`, `@v6`) as Coil, GeneralTools and MyGameTools — current patch releases arrive without edits; a moved or hijacked tag
  runs with the job's token (read-only everywhere except a job that runs no action).
Recommendation: **(a)**. The exposure of (b) is small here, which is why this is a call and not a rule; (a) keeps the pins reviewable and
costs one documented command (6.9) a few times a year.

**C3. Which repository settings are applied with the protection?** B (merge methods and PR-title merge subjects), C (token default `read`) and
D (`sha_pinning_required`) are independent of each other and of the protection (6.5). Recommendation: apply **B and C**; D only with C2(a).
Declining B leaves all three merge methods available, contrary to the baseline's "only Create a merge commit is enabled".

## 8. Known gaps

- **No execution.** No workflow ran, no Gradle task ran, no API was written. The first pull-request run is the measurement for names, timings and
  the Node question; step 3 of 6.7 lists what to read from it.
- **Node and Yarn on the compile jobs (F10).** Whether `assemble`, `jvmTestClasses` or `jsTestClasses` pulls `kotlinNodeJsSetup` or `kotlinNpmInstall`
  is not established; KGP's `NpmResolverPluginApplier` makes any `RequiresNpmDependenciesTask` with required npm dependencies depend on
  `kotlinNpmInstall`, and which js tasks qualify was not traced. If they do, the cost is network and seconds; the fallback task list is
  `compileKotlinJvm compileKotlinJs compileTestKotlinJvm compileTestKotlinJs :graphics-desktop:compileKotlin :graphics-desktop:compileTestKotlin`
  (which has the same uncertainty for `compileTestKotlinJs`).
- **The protection payload (F3).** That `checks` without `contexts` is accepted (go-github's comment says so; the OpenAPI schema lists
  `contexts` as required) and that an explicit `app_id` is accepted for a check that never ran were not tested; the retry notes in 6.4.2 are
  the fallback, and the order of operations avoids the second case.
- **Ruleset payload (6.4.4)** was composed from the REST schema and Coil's live ruleset; nothing was applied.
- **Matrix names with an expression in `name:`** (not used here) were not read from a live example.
- **Auto-CRLF on the Windows runner** is an inference from the image's Git installer options (F12); nothing here depends on it, since both
  comparisons that matter ignore line endings. Windows long-path behaviour of the TestKit workspace and `node_modules` is unmeasured.
- **Release on a non-snapshot version.** That `./gradlew build` with `version=0.1.0` succeeds without signing keys (vanniktech's
  `signAllPublications()` applies to publication tasks) was not run; the release rehearsal does not reach the build step. Run
  `./gradlew build -Pversion=0.1.0` once locally before G7.
- **Build cache masking (F16)** is derived from Gradle's task caching model and `Test`'s `@CacheableTask`; it was not observed. The first runs
  (6.7 step 3) show `FROM-CACHE` or not.
- **Timings.** Windows `Build` and macOS/Ubuntu `Compile` durations are unmeasured; the TestKit estimate is the testing finding's. `macos-latest`
  has 7 GB RAM and the build sets no `org.gradle.jvmargs`.
- **Workflow lockfiles (F8)** are in public preview and listed under GitHub Enterprise Cloud; their availability for a Free organization was not
  checked, and they are not used.
- **Build scans** need Gradle's terms of use and the Develocity plugin; nothing was evaluated beyond that boundary.
- **Gradle 9.8.2 as the wrapper (toolchain finding C1)** raises no CI obstacle found here (wrapper validation fetches unknown checksums; no
  `setup-gradle` annotation), but Dokka 2.2.0 prints a deprecation on 9.8 and the combination has not run.

## 9. Load-bearing claims

Quotes from Markdown/Liquid, AsciiDoc, YAML, JSON, Java, Kotlin and Go sources omit markup characters only. "Read" is 2026-10-20.

1. **A required-check name is the job name; workflow, matrix and trigger are not part of the rule.** "**Workflow**: The name format is `<job name>`.
   * **Reusable workflow**: The name format is `<job name> / <reusable job name>`." and "Required status checks do not take workflow, matrix, or
   event trigger types into account."
   [troubleshooting-rules.md](https://github.com/github/docs/blob/3a444c04d005e5d91e29ee0658fd5c9bbeaecf24/content/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/troubleshooting-rules.md), `github/docs@3a444c0`.
2. **Job names must be unique across workflows.** "make sure that job names are unique across all workflows. Using the same job name in multiple
   workflows can cause ambiguous status check results and block pull requests from being merged."
   [about-protected-branches.md](https://github.com/github/docs/blob/3a444c04d005e5d91e29ee0658fd5c9bbeaecf24/content/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches.md), `github/docs@3a444c0`.
3. **Skipped workflows leave checks pending; skipped jobs pass.** "| A workflow is skipped by path filtering, branch filtering, or a commit message |
   Associated checks stay in a "Pending" state and block merging | Avoid requiring workflows that can be skipped. |" and "| A job is skipped by a
   conditional | The job reports "Success" |" and "| A job depends on a failed job | The dependent job is skipped and may not block merging |".
   [troubleshooting-required-status-checks.md](https://github.com/github/docs/blob/3a444c04d005e5d91e29ee0658fd5c9bbeaecf24/content/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks.md), `github/docs@3a444c0`.
4. **`workflow_dispatch` checks do not satisfy a required check.** "if a workflow is triggered by `workflow_dispatch` on a pull request's head
   branch, checks reported by its jobs do not appear in the pull request's checks section. Even if the checks pass for the head commit, they do
   not satisfy a required status check in a branch ruleset." Same page.
5. **Successful states and the seven-day rule.** "Successful check statuses are `success`, `skipped`, and `neutral`." and "A required status check
   must have completed successfully in the chosen repository during the past seven days." Same page.
6. **Unbound checks can be set by anyone with write access; strict means up to date.** "Any person or integration with write permissions to a
   repository can set the state of any status check in the repository, but in some cases you may only want to accept a status check from a
   specific GitHub App." and, in the table, "**Strict** | The **Require branches to be up to date before merging** checkbox is checked. | The
   branch **must** be up to date with the base branch before merging." about-protected-branches.md, `github/docs@3a444c0`.
7. **Admins are exempt from classic protection unless included.** "By default, the restrictions of a branch protection rule don't apply to people
   with admin permissions to the repository or custom roles with the "bypass branch protections" permission. You can optionally apply the
   restrictions to administrators and roles with the "bypass branch protections" permission, too." about-protected-branches.md, `github/docs@3a444c0`.
8. **Linear history forbids merge commits.** "Enforcing a linear commit history prevents collaborators from pushing merge commits to the branch.
   This means that any pull requests merged into the protected branch must use a squash merge or a rebase merge." Same page.
9. **`app_id` semantics.** "The ID of the GitHub App that must provide this check. Omit this field to automatically select the GitHub App that
   has recently provided this check, or any app if it was not set by a GitHub App. Pass -1 to explicitly allow any app to set the status."
   [OpenAPI, `PUT /repos/{owner}/{repo}/branches/{branch}/protection`](https://github.com/github/rest-api-description/blob/755ee33b4a81d51f6f07a902cce26cbec3b822f2/descriptions/api.github.com/api.github.com.2022-11-28.json), `rest-api-description@755ee33`.
10. **The OpenAPI schema lists `contexts` as required; go-github says only one of `contexts`/`checks` may be populated.** Schema: `"required":
    ["strict", "contexts"]` under `required_status_checks`. go-github `RequiredStatusChecks`: "(Deprecated. Note: only one of Contexts/Checks can be
    populated, but at least one must be populated)." [repos.go](https://github.com/google/go-github/blob/v93.0.0/github/repos.go), go-github `v93.0.0`.
11. **GitHub Actions is app 15368.** `GET https://api.github.com/apps/github-actions` returned `{"id":15368,"name":"GitHub Actions","slug":"github-actions"}`
    (read 2026-10-20; Okio's check runs on `c0ac506` all report `app.id` 15368).
12. **Rulesets: `evaluate` is Enterprise-only; `pull_request` bypass mode.** "`evaluate` allows admins to test rules before enforcing them ...
    (`evaluate` is only available with GitHub Enterprise)." and "`pull_request` means that an actor can only bypass rules on pull requests."
    OpenAPI description, `rest-api-description@755ee33` (`repository-rule-enforcement`, `repository-ruleset-bypass-actor`). The organization's plan:
    `GET https://api.github.com/orgs/SpartanLaboratories` → `"plan":{"name":"free"}`.
13. **Merge settings fields.** `merge_commit_title`: "`PR_TITLE` - default to the pull request's title. `MERGE_MESSAGE` - default to the classic title for a
    merge message (e.g., Merge pull request #123 from branch-name)."; `merge_commit_message`: "`PR_BODY` - default to the pull request's body.";
    `delete_branch_on_merge`: "Either `true` to allow automatically deleting head branches when pull requests are merged". OpenAPI, `PATCH
    /repos/{owner}/{repo}`, `rest-api-description@755ee33`. Workflow token fields `default_workflow_permissions` and `can_approve_pull_request_reviews`
    (`actions-set-default-workflow-permissions`) and `sha_pinning_required` (`PUT /repos/{owner}/{repo}/actions/permissions`) are in the same description.
14. **The organization's baseline.** "On GitHub only “Create a merge commit” is enabled — “Squash” and “Rebase and merge” are turned off", "title
    **must be a valid Conventional Commit** (it becomes the merge-commit subject)", "A release workflow creates the GitHub Release from the tag.",
    "**Publishing to a package registry (e.g. Maven Central) is manual.** It is irreversible and stays a deliberate human action — no registry
    credentials live in CI." and "A second approving review becomes required once a second person is on the repository."
    [CONTRIBUTING.md](https://github.com/SpartanLaboratories/.github/blob/main/CONTRIBUTING.md), read 2026-10-20.
15. **Latest action releases.** `GET /repos/actions/checkout/releases/latest` → `v7.0.1` (2026-07-20); `actions/setup-java` → `v6.1.0` (2026-10-14);
    `gradle/actions` → `v6.4.0` (2026-09-28); `actions/upload-artifact` → `v7.0.2` (2026-10-07). Tag commits: `GET /repos/actions/checkout/commits/v7.0.1` →
    `3d3c42e5aac5ba805825da76410c181273ba90b1`; `setup-java v6.1.0` → `b843ebf9b913146b38da2eaa7aeddf76074ae1ad`; `gradle/actions v6.4.0` →
    `3f5f9adaf7d9fecd50b5935e54106014257a94e6`; `upload-artifact v7.0.2` → `cf430e030ddbb5b0abf93d22962f4752f3646cd9`
    ([checkout releases](https://github.com/actions/checkout/releases/tag/v7.0.1), [setup-java](https://github.com/actions/setup-java/releases/tag/v6.1.0),
    [gradle/actions](https://github.com/gradle/actions/releases/tag/v6.4.0), [upload-artifact](https://github.com/actions/upload-artifact/releases/tag/v7.0.2)).
16. **GitHub's pinning guidance.** "Pin references to actions to a commit SHA in your workflow files. This helps mitigate the risk of a bad actor
    adding a backdoor to the action's repository, as they would need to generate a SHA-1 collision for a valid Git object payload." and "Note that
    there is risk to this approach even if you trust the author, because a tag can be moved or deleted if a bad actor gains access to the
    repository storing the action." [secure-use.md](https://github.com/github/docs/blob/3a444c04d005e5d91e29ee0658fd5c9bbeaecf24/content/actions/reference/security/secure-use.md), `github/docs@3a444c0`.
17. **Workflow lockfiles are a preview.** "Workflow dependency lockfiles are in public preview and subject to change."
    [workflow-dependency-lockfiles.md](https://github.com/github/docs/blob/3a444c04d005e5d91e29ee0658fd5c9bbeaecf24/content/actions/concepts/security/workflow-dependency-lockfiles.md)
    (note text from `data/reusables/actions/workflow-lockfiles-preview.md`), `github/docs@3a444c0`.
18. **v6 of `setup-gradle` comes with terms; basic caching is MIT and avoids them.** "By upgrading to v6, you accept the [Terms of Use] for the
    `gradle-actions-caching` component." ([v6.0.0 release notes](https://github.com/gradle/actions/releases/tag/v6.0.0)); "If you do not wish to use
    proprietary code or accept the Gradle Technologies Terms of Use, you can opt-out of enhanced caching with a single line of configuration" and
    "The `gradle-actions-caching` component is used only when enhanced caching is enabled and is not loaded or used with basic caching or when
    caching is disabled." ([DISTRIBUTION.md](https://github.com/gradle/actions/blob/v6.4.0/DISTRIBUTION.md), `v6.4.0`).
19. **Basic provider facts.** "`basic`: A fully open-source (MIT) caching implementation built on the standard GitHub Actions cache (`@actions/cache`).
    Uses the same caching strategy as `actions/setup-java` with `cache: gradle`." and "Unlike the Enhanced Caching provider, Basic Caching does not use
    restore keys." and "`gradle-home-cache-includes` / `gradle-home-cache-excludes`): The cached paths are fixed."
    [setup-gradle.md](https://github.com/gradle/actions/blob/v6.4.0/docs/setup-gradle.md), `v6.4.0`.
20. **Cache writes only from the default branch.** "By default, The `setup-gradle` action will only write to the cache from Jobs on the default
    (`main`/`master`) branch. Jobs on other branches will read entries from the cache but will not write updated entries." Same file.
21. **Configuration-cache data needs an encryption key.** "Configuration-cache data will not be saved/restored without an encryption key being
    provided." [setup-gradle/action.yml](https://github.com/gradle/actions/blob/v6.4.0/setup-gradle/action.yml), `v6.4.0`; v6.0.0 notes: "Existing,
    rudimentary, configuration-cache support has been removed".
22. **Wrapper validation is part of `setup-gradle` and fetches unknown checksums.** "By default, this action will perform the same wrapper
    validation as is performed by the dedicated [wrapper-validation action]." (setup-gradle.md, `v6.4.0`); `const all = await
    httpGetJsonArray('https://services.gradle.org/versions/all')` and `(entry: any) => !knownChecksums.versions.has(entry.version)`
    ([checksums.ts](https://github.com/gradle/actions/blob/v6.4.0/sources/src/wrapper-validation/checksums.ts), `v6.4.0`).
23. **Build scans need terms acceptance.** "To publish to https://scans.gradle.com, you must specify in your workflow that you accept the [Gradle
    Terms of Use]." with `build-scan-terms-of-use-agree: 'yes'` (setup-gradle.md, `v6.4.0`).
24. **KGP lifecycle task names.** `compileAllTaskName = lowerCamelCaseName(target.disambiguationClassifier, compilationName, "classes")`
    ([KotlinCompilationTaskNamesContainerFactories.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/mpp/compilationImpl/factory/KotlinCompilationTaskNamesContainerFactories.kt));
    `project.registerTask<DefaultTask>(compilation.compileAllTaskName) {` with `description = "Assembles outputs for compilation '${compilation.name}'
    of target '${compilation.target.name}'"`
    ([KotlinCreateLifecycleTasksSideEffect.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/mpp/compilationImpl/KotlinCreateLifecycleTasksSideEffect.kt));
    `project.tasks.named(compilationInfo.compileAllTaskName).configure { it.dependsOn(kotlinTask) }`
    ([KotlinJsIrSourceSetProcessor.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/plugin/KotlinJsIrSourceSetProcessor.kt)), Kotlin `v2.4.21`.
25. **Which tasks depend on the npm install.** `NpmResolverPluginApplier.apply`: "// KotlinJsTest delegates npm dependencies to testFramework" `if
    (task !is KotlinJsTest) {` … `if (task.requiredNpmDependencies.isNotEmpty()) { task.dependsOn( nodeJsRoot.npmInstallTaskProvider,`
    ([NpmResolverPluginApplier.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/targets/web/npm/NpmResolverPluginApplier.kt), Kotlin `v2.4.21`);
    and the js test task: `test.dependsOn(project.nodeJsSetupTaskProvider)` (testing finding, claim 13, KotlinNodeJsIr.kt `v2.4.21`).
26. **The yarn lock comparison ignores line endings.** `lockFileMismatchReport.get() != LockFileMismatchReport.NONE &&
    !contentEqualsIgnoringLineEndings(value.asFile, outputFile)`
    ([LockStoreTask.kt](https://github.com/JetBrains/kotlin/blob/v2.4.21/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/targets/js/npm/LockStoreTask.kt), Kotlin `v2.4.21`);
    the ABI dump comparison: "We don't compare a full text because newlines on Windows & Linux/macOS are different" (ABI finding, claim 13).
27. **`Test` is cacheable; `upToDateWhen` is discouraged.** `@CacheableTask public abstract class Test extends AbstractTestTask implements
    JavaForkOptions, PatternFilterable`
    ([Test.java](https://github.com/gradle/gradle/blob/v9.8.2/platforms/jvm/testing-jvm/src/main/java/org/gradle/api/tasks/testing/Test.java), Gradle `v9.8.2`);
    "For the very same reason using `outputs.upToDateWhen` is discouraged and should be replaced by properly declaring the task's inputs."
    ([common_caching_problems.adoc](https://github.com/gradle/gradle/blob/v9.8.2/platforms/documentation/docs/src/docs/userguide/optimizing-builds/build-cache/common_caching_problems.adoc), `v9.8.2`).
28. **Runner labels, size and cost.** `windows-latest` → `Windows2025-Readme.md`; `ubuntu-latest` → `Ubuntu2404-Readme.md`; `macos-latest` →
    `macos-26-arm64-Readme.md` ("3 (M1)" CPUs, "7 GB"); "Use of the standard GitHub-hosted runners is free and unlimited on public repositories."
    [supported-github-runners.md](https://github.com/github/docs/blob/3a444c04d005e5d91e29ee0658fd5c9bbeaecf24/data/reusables/actions/supported-github-runners.md), `github/docs@3a444c0`.
29. **Preinstalled JDKs and the CLI.** Windows 2025 image `20261013.288.2`: "| 17.0.20+101 (default) | JAVA_HOME_17_X64 |", "| 21.0.12+101.0 |
    JAVA_HOME_21_X64 |"; Ubuntu 24.04 image `20261012.334.1`: "| 17.0.20+1 (default) | JAVA_HOME_17_X64 |", "| 21.0.12+1 | JAVA_HOME_21_X64 |"; macOS 26
    arm64 image `20261013.0424.1`: "| 21.0.12+101.0 (default) | JAVA_HOME_21_arm64 |"; "GitHub CLI 2.102.0" on Windows and Ubuntu.
    [Windows2025-Readme.md](https://github.com/actions/runner-images/blob/68bd77adbdf0017e6e3d622348abdb2763ca26f4/images/windows/Windows2025-Readme.md),
    [Ubuntu2404-Readme.md](https://github.com/actions/runner-images/blob/68bd77adbdf0017e6e3d622348abdb2763ca26f4/images/ubuntu/Ubuntu2404-Readme.md),
    [macos-26-arm64-Readme.md](https://github.com/actions/runner-images/blob/68bd77adbdf0017e6e3d622348abdb2763ca26f4/images/macos/macos-26-arm64-Readme.md).
30. **Windows long paths.** `Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem' -Name 'LongPathsEnabled' -Value 1`
    ([Configure-BaseImage.ps1](https://github.com/actions/runner-images/blob/68bd77adbdf0017e6e3d622348abdb2763ca26f4/images/windows/scripts/build/Configure-BaseImage.ps1)).
31. **`gh release create` flags and environment.** `--verify-tag`: "Abort in case the git tag doesn't already exist in the remote repository";
    `--generate-notes`: "Automatically generate title and notes for the release via GitHub Release Notes API"; `GH_REPO`: "specify the GitHub
    repository in the `[HOST/]OWNER/REPO` format for commands that otherwise operate on a local repository"
    ([gh_release_create](https://cli.github.com/manual/gh_release_create), [gh_help_environment](https://cli.github.com/manual/gh_help_environment), read 2026-10-20).
32. **Annotated tags are distinguishable through the refs API.** `GET /repos/gradle/actions/git/ref/tags/v6.4.0` → `"object":{"type":"tag",…}`;
    `GET /repos/actions/checkout/git/ref/tags/v7.0.1` → `"object":{"type":"commit",…}` (read 2026-10-20).
33. **Sibling and field practice read as files.** Okio `build.yml` at `c0ac506`: `uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1`,
    `runs-on: ${{ matrix.os }}` with `os: [ macos-14, ubuntu-latest, windows-latest ]`, run step `./gradlew build` and `./gradlew build -Dkwasm=false`
    on Windows ([build.yml](https://github.com/square/okio/blob/c0ac50617c4dd03efd13a78d882c4462a57bf543/.github/workflows/build.yml)); Coil `ci.yml`
    at `3.6.3` ("# Limit concurrency to 1 for PRs. 'main' concurrency isn't limited.", `uses: actions/checkout@v7`,
    [ci.yml](https://github.com/coil-kt/coil/blob/3.6.3/.github/workflows/ci.yml)); MyGameTools `.github/workflows/ci.yml` and `release.yml` at
    `dc5214f` (`concurrency: group: ci-${{ github.ref }}`, `cancel-in-progress: ${{ github.event_name == 'pull_request' }}`,
    `gh release create "$GITHUB_REF_NAME" --title "$GITHUB_REF_NAME" --generate-notes --verify-tag`); GeneralTools `Build.yml` at `3295908`
    (`cache-provider: basic`, `uses: actions/checkout@v7`). Live protection read 2026-10-20 via `GET /repos/{o}/{r}/branches/{default}` (Okio, Ktor,
    kotlinx-io: `"enforcement_level":"non_admins"`), `GET /repos/coil-kt/coil/rules/branches/main` (Coil: `"integration_id":15368`,
    `"strict_required_status_checks_policy":false`) and `GET /repos/SpartanLabsGaming/MyGameTools/branches/master/protection` (`"enforce_admins":{"enabled":false}`,
    `"required_approving_review_count":0`, `"dismiss_stale_reviews":true`).
