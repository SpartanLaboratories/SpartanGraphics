# Architecture: SpartanGraphics repository, modules, packages and artifacts

> **Superseded on 2026-10-20** by the roadmap, `docs/roadmap/vision-and-roadmap.md`. Its target
> architecture is in §2, and the decisions that settled this draft's proposals are in §5 (A to H).
> Issue #1's stage list and its scope-change comments carry the resulting Phase 1. What happened
> to each proposal:
>
> - **§4's kept decisions** (separate backend modules, one core module, audio in this repo, no
>   umbrella) stand.
> - **§5.1, one `assets` module:** rejected (decision B). `assets-core` stays, with its loaders in
>   the graphics backends. The `suspend` `AssetSource` was adopted on #1 (decision A).
> - **§5.2 `graphics-test`, §5.3 `graphics-bom`, §5.5 guarded backend contracts, §5.6 test levels
>   by package filter:** adopted (decisions C, D, E, F). `audio-test` was added for Phase 2.
> - **§5.4, one version for the repo:** kept as a roadmap assumption.
> - **§7, open question 1 (artifact prefix):** names kept (decision G).
> - **§7, open question 2 (basic types):** neither interfaces nor own final classes. The basic
>   types come from multiplatform GeneralTools (GeneralTools#7), and G2 waits for it (decision H).
> - **§7, open question 3 (PR sequence):** mapped onto G1–G7 through #1's scope-change comments.

## Header / Association

- **Covers:** [SpartanLaboratories/SpartanGraphics#1](https://github.com/SpartanLaboratories/SpartanGraphics/issues/1)
  (Phase 1: SpartanGraphics, tracking), stages G1–G7 of the split plan.
- **What this designs:** how the repository is built and published: its Gradle modules and their
  dependencies, the Kotlin package layout, the Maven artifacts, and how versioning, API stability
  and test levels map onto them. It doesn't design the graphics API itself (shapes, camera, input,
  outlines); the split plan covers that.
- **Status:** draft, for review. It proposes changes to the split plan's target architecture, and
  none of them is decided yet. No source, test or build file exists.
- **Baseline commit:** `main` @ `50b61e2` (the split plan on top of the initial commit).
- **Related docs:**
  - `docs/plans/spartan-libraries-split.md`: the split plan. This document reviews its "Target
    architecture" and "Build conventions" sections.
  - GameTools' `docs/plans/32-module-split/plan.md`: precedent for the `build-logic` convention
    plugins, the umbrella artifact and when to create a module.
  - GameTools' `com.spartanlabs.gaming.annotation` package: precedent for the stability-tier
    annotations.

---

## 1. Summary

The split plan's skeleton holds, and this design keeps it: a pure KMP core with a separate module
per platform backend, group `io.github.spartanlaboratories`, packages under
`com.spartanlabs.graphics`, flat module folders and a `build-logic` included build. It makes six
changes (§5). Most of them protect the setup where SpartanUI and MyGameClient both depend on this
repo.

## 2. Layout

```
SpartanGraphics/                  root project: no source, Dokka aggregation only (as in GameTools)
├─ build-logic/                   convention plugins: KMP library, JVM library, publishing
├─ gradle/libs.versions.toml
├─ gradle.properties              group + ONE version for every module
├─ graphics-bom/      BOM         keeps every artifact below on the same version
├─ graphics-core/     KMP         com.spartanlabs.graphics          API + backend interfaces
├─ graphics-desktop/  JVM         com.spartanlabs.graphics.desktop  LWJGL: GLFW, GL 3.3
├─ graphics-test/     KMP         com.spartanlabs.graphics.test     headless backend + fakes
├─ assets/            KMP         com.spartanlabs.assets            AssetSource + per-platform loaders
├─ audio-core/        KMP         com.spartanlabs.audio
└─ audio-desktop/     JVM         com.spartanlabs.audio.desktop     JLayer
```

Dependencies:

```
graphics-desktop, graphics-test ─api→ graphics-core ─api→ assets
audio-desktop ─api→ audio-core ─api→ assets
every module ─api(platform)→ graphics-bom
```

Each folder name, Gradle project name and artifactId match, and every project sets `group`.
Gradle's composite builds substitute an external dependency when it matches an included project's
`group:project.name`, so SpartanUI and MyGameClient can use `includeBuild("../SpartanGraphics")`
with no substitution rules.

## 3. Packages

### 3.1 graphics-core

```
com.spartanlabs.graphics       Color, Point, Size, Rect
  .camera                      Camera, world↔screen conversion
  .shape                       Rectangle, Text, OutlinedRectangle, Outline, layers, tags
  .picking
  .input                       Key, PointerButton, pointer/key/scroll events
  .platform                    what a backend implements: Surface, Renderer, FrameDriver,
                               Clock, InputSource, textures, TextMeasurer
  .annotation                  ExperimentalGraphicsApi, SupportedExtension (GameTools' pattern)
```

### 3.2 The other modules

- **graphics-desktop:** only its entry point and config are public. The `.gl` and `.glfw`
  subpackages are `internal`, and LWJGL is an `implementation` dependency, never `api`.
- **graphics-test, assets, audio-core, audio-desktop:** one package each, as listed in §2.

Test packages are covered in §5.6.

## 4. Kept from the split plan

### 4.1 Separate backend modules

The split plan's reason ("so an app pulls in only the backend it uses") doesn't hold: KMP's
per-platform variants already do that. The real reasons:

- A backend is a choice, not fixed by the platform: OpenGL now, maybe Metal or Vulkan on desktop
  or WebGPU on web later.
- The core stays free of LWJGL. If the backend lived in graphics-core's `jvmMain`, ui-core's
  `api(graphics-core)` would put LWJGL on every JVM consumer, headless tests included.
- It's the "interface plus default implementation" rule for systems and infrastructure, applied at
  module level.

### 4.2 One core module

Input, surface, frames and picking stay in graphics-core, in separate packages, so splitting one
out later is just a move. This follows GameTools' module-split plan (§1.3): create a module when
there's a clean cut and a consumer for it, not to fill out a diagram.

When SpartanUI needs platform services such as text input, clipboard or cursor shape, they go in
`.platform`, not in ui-* platform modules.

### 4.3 Audio in this repo, independent of graphics

Audio's artifact names don't say "graphics", so it can move to its own repo later without changing
coordinates.

### 4.4 No umbrella artifact

Unlike GameTools, there's no umbrella: it would have to pick a backend. `graphics-desktop` already
`api`s the core, so it's the only graphics dependency a desktop app needs.

## 5. Changes to the split plan

### 5.1 `assets-core` becomes `assets`, with the loaders inside it

- The split plan puts the classpath `AssetSource` in graphics-desktop, so an audio-only app would
  need the graphics backend just to load a file.
- `ClasspathAssetSource` goes in `assets`' `jvmMain`. Later, an AssetManager loader goes in
  `androidMain` and a fetch loader in `jsMain`. With no platform sibling module, the `-core` suffix
  no longer fits.
- The underlying rule: only a backend with a heavy or swappable dependency gets its own module.
  Platform glue with no dependencies and one natural implementation per platform goes in the KMP
  module's per-platform source sets.
- The contract is `suspend` from the start. A browser page can only load binary files
  asynchronously, so a blocking `AssetSource` would become a breaking change once web work starts.

### 5.2 New `graphics-test` module

It ships a recording `Renderer`, a scripted `InputSource`, a manual clock and a fixed-size
`Surface`.

- SpartanUI's layout and dispatch tests, and the client's ported `ViewportTest`, need exactly these
  fakes. Without this module each repo writes its own.
- It lets the whole frame pipeline be tested at Levels 3 and 4b with no GPU.
- It doubles as the worked example of implementing a backend.

### 5.3 New `graphics-bom`

Every module depends on it via `api(platform(...))`, so Gradle keeps the whole family on one
version. Without it, SpartanUI 0.2 can pull in graphics-core 0.2 while the client pins
graphics-desktop 0.1. The old backend then fails at runtime against the new interfaces
(`AbstractMethodError`).

### 5.4 One version for the whole repo

The version is set once in `gradle.properties`, not written into each module's `coordinates(...)`
the way GameTools does it. A backend only works with the core it was built against, so they always
release together.

### 5.5 Guarded backend interfaces

- The `.platform` interfaces carry `@SubclassOptInRequired(ExperimentalGraphicsApi::class)`. Apps
  call them freely, but anyone writing a backend has to opt in.
- The guard comes off once a second in-house backend (web or Android) has proven the interfaces.
  That is also the condition for 1.0.
- The KMP convention plugin turns on `explicitApi()` and the Kotlin Gradle plugin's
  `abiValidation()` (experimental), so every public-API change shows up as an API-dump diff in the
  PR.

### 5.6 Test packages and levels

- Tests go in `com.spartanlabs.testing.<level>.graphics.camera…`, mirroring the production
  package, as WebTools and GeneralTools do.
- Levels are selected by package filter, as in GameTools, not by JUnit `@Tag` as in WebTools.
  `commonTest` code can't use JUnit, so tags would only work for JVM-only tests.

## 6. Maven artifacts

Published at the first release, all under `io.github.spartanlaboratories`: `graphics-bom`,
`graphics-core`, `graphics-desktop`, `graphics-test`, `assets`, `audio-core` and `audio-desktop`.

Each KMP module also publishes one artifact per platform (`graphics-core-jvm`,
`graphics-core-wasm-js`, …). Gradle users depend on the base name. Maven-only users need the
`-jvm` artifact, which the README should say.

## 7. Open questions

1. **Artifact-name prefix.** `ui-core`, `audio-core` and `assets` are generic words, and jar
   filenames don't include the group. A `spartan-` prefix would fix that, but only before the first
   Maven Central release (G7), because published coordinates can't change. Recommendation: keep
   the split plan's names, which follow the `webtools-*` / `gametools-*` pattern.
2. **Basic types as interfaces.** This is outside this document's scope, but it shapes
   graphics-core's API. The split plan makes Color, Point, Size and Rect "contract interfaces with
   a default immutable implementation each".
   - An interface can't guarantee immutability, so a renderer could see a consumer's mutable
     `Point` change mid-frame.
   - It also rules out storing them as compact value classes, the way Compose does with `Offset`,
     `Size` and `Color`.

   Recommendation: final immutable types, with the client converting. No issue is filed yet.
3. **PR sequence.** How these changes map onto G1–G7 isn't worked out yet. For example,
   `graphics-test` has no stage.
