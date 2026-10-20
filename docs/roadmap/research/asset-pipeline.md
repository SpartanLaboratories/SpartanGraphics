# Research: asset-pipeline

## 1. Question

What should SpartanGraphics' asset system look like in the final state across desktop JVM, Android and web? That covers the `AssetSource` contract, the asset manager (handles, caching, preload groups), development hot reload, and a build-time Gradle plugin (atlas packing, font baking, manifests, compressed textures). The answer must say what 1.0 freezes, where each piece lives, and in what order the pieces land.

## 2. Bottom line

- **Freeze a small, `suspend`, whole-asset `AssetSource` at 1.0.**
  - The shape is `suspend fun read(name: String): Result<ByteArray>`, plus a written name grammar and a typed, open failure hierarchy.
  - The browser has no synchronous file access, and every mature web-capable engine either goes async (Compose resources, KorIO, Bevy) or works around the lack (libGDX's GWT backend preloads everything at startup).
  - Everything beyond `read` (streaming, URI/locator, listing, change notification) is a separate capability interface, added without breaking implementers. Do not put any of it in the frozen core.
- **The asset manager is a post-1.0, additive layer, and it must not shape the 1.0 texture contract.**
  - Decision 4 does not list it among the frozen items.
  - Protect the future manager with three 1.0 rules.
    - Backends create textures from decoded pixels, not from names.
    - A texture's identity is an opaque backend token, never a GL id.
    - Every backend keeps the recipe (asset name plus decode options) needed to rebuild a texture after context loss or reload.
  - Android context loss and web `webglcontextlost` make the recipe a day-one requirement.
- **Manager shape: typed handles with an explicit state, lifetimes by scope, and a three-stage loader.**
  - Handles are Bevy-style: `AssetHandle<T>` carries `Loading`, `Ready` or `Failed`, and `load` returns immediately.
  - Lifetimes are scopes or groups (PixiJS-bundle or libGDX-reference-count style), not Rust-style drop-based refcounts, because Kotlin has no deterministic drop and GPU memory needs deterministic release.
  - Loaders are split into IO and decode off-thread, then commit on the context owner's thread. This is libGDX's `loadAsync`/`loadSync` split.
  - The manager lives in its own generic module with no graphics dependency, so audio can use it too.
- **Hot reload is an Experimental, desktop-only, development capability. It is not part of the 1.0 freeze.**
  - It is an optional `ChangeNotifying` capability on a filesystem `AssetSource` (built on `WatchService`), plus a manager subscription that swaps content in place under a stable handle.
  - Android and web hot reload stay out of scope.
- **Gradle plugin: a separate included build in this repo, with its own plugin id under the Maven group, published separately.**
  - Suggested id: `io.github.spartanlaboratories.assets`.
  - Land the tasks in this order: manifest (sizes and hashes), atlas packer, font baking, texture compression. All are `@CacheableTask`s that feed per-target resource and asset directories.
  - The contract between plugin and runtime is versioned file formats, not shared code.

## 3. Findings

### 3.1 Blocking versus `suspend`: the web forces `suspend`

- **Compose Multiplatform resources.** `suspend fun readBytes(path: String): ByteArray` on `Res`.
  - On web, `painterResource` first returns an empty painter, then the loaded one.
  - `Res.getUri(path)` exists precisely because some consumers want a path or URI, not bytes.
  - Only bundled files are resources; remote files need Ktor or similar.
  - Tier: common practice (JetBrains' official library).
  - Source: [Compose resources usage](https://kotlinlang.org/docs/multiplatform/compose-multiplatform-resources-usage.html).
- **KorIO VFS.** `VfsFile` exposes `suspend` functions (`readString`, `list`, `stat`, `exists`).
  - Implementations: `LocalVfs`, `MemoryVfs`, `ZipVfs`, `UrlVfs` (HTTP), `MergedVfs`, `JailVfs`.
  - Standard roots: `resourcesVfs`, `applicationDataVfs`, `cacheVfs`.
  - Tier: common practice, though KorGE's own page is marked "not reviewed yet".
  - Source: [KorIO VFS](https://docs.korge.org/io/vfs/).
- **Bevy.** `AssetReader` is async. Its `read_directory` is a required method.
  - The default `FileAssetReader` is not available on wasm or Android, so each platform registers its own reader through `AssetSource`.
  - Tier: common practice.
  - Source: [Bevy `AssetReader`](https://dev-docs.bevyengine.org/bevy/asset/io/trait.AssetReader.html).
- **libGDX HTML5 (GWT) backend.** It must preload every asset before the app starts.
  - libGDX APIs are unavailable until preloading finishes.
  - From 1.9.12 a filter can restrict what is preloaded. Filtered files must then be loaded only through `AssetManager`, "otherwise your game will freeze".
  - This is the cost of a synchronous file API on a platform without one. libGDX had to bolt on an asset-filter workaround later.
  - Tier: one project's documentation, but strongly corroborating.
  - Source: [libGDX HTML5 backend and GWT specifics](https://libgdx.com/wiki/html5-backend-and-gwt-specifics).
- **Conclusion (established consensus among the KMP and web-capable engines surveyed).** The core read must be `suspend`.
  - A blocking convenience on desktop is fine as a separate extension (`runBlocking`, JVM-only), but never as the contract.
  - This supports the draft's proposal and refutes the split plan's implicit blocking shape ("an `AssetSource` that loads from the classpath").

### 3.2 Bytes versus streams: bytes in the core, streams and locators as capabilities

- **Platform-natural primitives differ for streaming.**
  - Compose pairs `readBytes` with `getUri` for external players ([usage](https://kotlinlang.org/docs/multiplatform/compose-multiplatform-resources-usage.html)).
  - Android Play Asset Delivery's `fast-follow` and `on-demand` packs are "served as archives and expanded into internal storage". They cannot be assumed to be at a fixed location, and install-time packs are the ones readable via `AssetManager` ([PAD docs](https://developer.android.com/guide/playcore/asset-delivery)).
  - The browser path is `fetch` then a `Blob`, or a URL handed to an audio element.
  - Android `AssetManager.open(name, ACCESS_STREAMING)` yields an `InputStream`. `openFd` works only for uncompressed assets, which the build must be told not to compress (from general knowledge; the official page was truncated in my fetch).
  - So a single "stream" abstraction is not natural. Sequential-read, URI and fd capabilities each fit different audio backends.
- **No stable KMP stream type exists to freeze against.**
  - kotlinx-io is marked "Alpha... experimental... API is subject to change" at 0.9.1, with its filesystem support also experimental ([kotlinx-io](https://github.com/Kotlin/kotlinx-io)).
  - Okio's core is multiplatform, and Wasm support has existed since 3.5, but its early announcement called the Wasm support "not ready for production". The Wasm file-system module is WASI-oriented, and one user reported `okio-nodefilesystem` failing to resolve for `wasmJs` ([Okio multiplatform docs mirror](https://os.unplugged.com/werunplugged/unplugged-system/raw/branch/main/external/okio/docs/multiplatform.md), [Okio 3.5 has WASM](https://code.cash.app/okio-wasm), [okio#1463](https://github.com/square/okio/issues/1463)). Tier: single-source reports.
  - Putting a third-party stream type in a contract frozen at 1.0 is a risk with no payoff for 2D game assets.
- **Conclusion (design inference).**
  - The frozen core is `ByteArray`.
  - Streaming music and large files come later as capability interfaces (`StreamingAssetSource`, `LocatableAssetSource`).
  - Each capability is `@SupportedExtension` or Experimental until a backend has used it, and discovered by `is` checks or a `capabilities` query.

### 3.3 Name and path conventions

- **Bevy's name syntax.**
  - A `#` in a path is a label separator (sub-asset, e.g. an atlas region), so files containing `#` need `AssetPath`.
  - Multiple named sources exist (`source://path`).
  - Source: [`AssetServer` docs](https://docs.rs/bevy/latest/bevy/asset/struct.AssetServer.html). Tier: common practice.
  - The lesson is to reserve syntax early, because retrofitting a separator breaks existing names.
- **Compose resources.** Paths are relative to `composeResources` (`files/myDir/someFile.bin`). Typed accessors (`Res.drawable.example_image`) are generated for images, strings and fonts, but raw `files/` get no qualifiers.
  - Source: [setup](https://kotlinlang.org/docs/multiplatform/compose-multiplatform-resources-setup.html).
- **The monolith today.** `TextureCache` and `SoundPlayer` take bare names and prepend `textures/` and `sounds/` internally (`TextureCache.kt:14`, `SoundPlayer.kt:14`). Missing or undecodable textures fall back to a 1x1 white texture, logged once.

### 3.4 Asset managers: what each does and where it hurts

| System | Shape | Regret or trap |
|---|---|---|
| **libGDX `AssetManager`** | Async loaders and `update()` polling with a time budget. Reference counting with dependencies (a font keeps its texture alive). `unload`, `clear` and `dispose`. A `FileHandleResolver` abstracts the source. Loader split into `loadAsync` (worker) and `loadSync` (GL thread). | `update(ms)` is a hint, not a frame guarantee. A static instance on Android can hold dead GL resources (black textures). Managed textures are reloaded on resume, which takes time. Loader state lives in fields that must be reset per asset. |
| **Bevy `AssetServer`** | `load(path)` returns a handle at once. Loading the same path returns the existing handle. Strong handles are reference counted by `Assets<T>` and the asset is unloaded when the last one drops. `AssetId` is a weak, possibly dangling reference. `LoadState` is separate for the asset, its dependencies, and recursive dependencies. `load_folder` returns a `LoadedFolder`. | PR #10785 fixed a panic when an asset was removed manually while handles were alive. Issue #21878 notes that subassets have separate lifetimes. `#` is the label separator. |
| **Godot `ResourceLoader`** | `load_threaded_request(path, type_hint, use_sub_threads, cache_mode)`, then `load_threaded_get_status` with a progress array, then `load_threaded_get`. Cache modes: IGNORE, REUSE (default), REPLACE (refresh in place), plus `_DEEP` variants. `list_directory` order is nondeterministic. | `load_threaded_get` blocks if the load is not finished. `use_sub_threads` can slow the main thread. Five cache modes is a lot of policy surface. |
| **PixiJS `Assets`** | A manifest of named bundles. `loadBundle` takes progress and retry options. `backgroundLoadBundle` prefetches the next screen's assets (requires a manifest). Cached by URL or alias. `unload` frees memory. A texture preference list selects format and resolution. | Compressed-texture loaders must be imported before the first `load`. (Details of the 8.x guide were not reachable directly. The bundle API detail comes from third-party mirrors of the docs.) |
| **KorGE** | Resources through `resourcesVfs` and `suspend` reads. An atlas is a folder marked by an `.atlas` file and read with `readAtlas()`. Hot reload is JVM-only, run through a `runJvmAutoreload` task, and expects game code inside a `Scene`. | The docs are marked "not reviewed". The Gradle plugin's namespace has moved (`com.soywiz.korge` to `com.soywiz.korlibs.korge.plugins`, apparently on to `org.korge.engine`). The Plugin Portal listing for `com.soywiz.korge` is stale at 1.10.0.0 from 2020. |
| **Compose resources** | A generated `Res` class with typed accessors and `suspend readBytes`. Resources ship inside published Maven artifacts. `configureWebResources { resourcePathMapping }` redirects web paths to a CDN. | Raw files get no qualifiers. A library target needs AGP 8.8+ and `androidResources.enable = true`. The empty-first-frame behaviour on web. |

Sources:
- libGDX: [managing your assets](https://libgdx.com/wiki/managing-your-assets).
- Bevy: [AssetServer](https://docs.rs/bevy/latest/bevy/asset/struct.AssetServer.html), [PR 10785](https://github.com/bevyengine/bevy/pull/10785), [issue 21878](https://github.com/bevyengine/bevy/issues/21878).
- Godot: [background loading](https://docs.godotengine.org/en/stable/tutorials/io/background_loading.html), [ResourceLoader](https://docs.godotengine.org/en/stable/classes/class_resourceloader.html).
- PixiJS: [Assets guide](https://pixijs.com/8.x/guides/components/assets) (caching, unload and aliases confirmed there).
- KorGE: [hot reloading](https://docs.korge.org/hot-reloading/), [KorGE target plugin pages](https://plugins.gradle.org/plugin/com.soywiz.korge).
- Compose: [setup](https://kotlinlang.org/docs/multiplatform/compose-multiplatform-resources-setup.html).

Tier: common practice. All of these are mature, widely used systems.

Three cross-cutting observations hold across them:
- **Every system gives load a non-blocking entry point and a pollable or observable state** (`update()`, `LoadState`, `load_threaded_get_status`, progress callbacks).
- **Every system separates "reference exists" from "bytes loaded".** The handle comes first.
- **GPU work is always deferred to the context-owner thread** (libGDX `loadSync`; Bevy and Godot do the same internally).

### 3.5 Handles, ownership and GPU context loss

- **Android.**
  - Everything tied to the GL context is destroyed when it is lost.
  - `onSurfaceCreated` is called at start and again whenever the context is lost (I read this from the official page's described behaviour; the page excerpt itself was truncated).
  - Textures, shaders and buffers must be recreated from CPU-side data. Java-side `Bitmap`s survive but GL ids are invalid.
  - Source: [GLSurfaceView.Renderer](https://developer.android.com/reference/android/opengl/GLSurfaceView.Renderer).
- **Web.** `webglcontextlost` and `webglcontextrestored` follow the same model.
  - MDN lists the events and says context loss is one of the only two errors a well-formed page should see.
  - Source: [WebGL best practices](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API/WebGL_best_practices).
- **libGDX.** Managed textures are reloaded from their `FileHandle` on resume. This is the recipe model, and the black-texture bug reports show the cost of leaking raw GL state across the lifecycle ([wiki](https://libgdx.com/wiki/managing-your-assets), [Stack Exchange answer via search](https://backiee.wasmer.app/https_gamedev_stackexchange_com/a/169543)).
- **Conclusion (design inference).**
  - A `Texture` handed to app code must be a backend-issued token that survives context loss. Content is rebuilt from a retained recipe: asset name, decode options, or a decoded image kept in memory by policy.
  - Hot reload uses the same re-create path, so building context-loss rebuild first gives hot reload almost for free.

### 3.6 Decode location and the pixel contract

- **Web.** `createImageBitmap` accepts a `Blob` and has `premultiplyAlpha` (`none`, `premultiply`, default `default`), `colorSpaceConversion` and `imageOrientation` options. The `default` values are implementation-specific, and it is available in workers.
  - Source: [MDN createImageBitmap](https://developer.mozilla.org/en-US/docs/Web/API/Window/createImageBitmap).
  - Pin `premultiplyAlpha`, `colorSpaceConversion` and orientation explicitly, otherwise the same PNG decodes differently per platform.
- **Desktop.** The monolith decodes with `stbi_load_from_memory` after a global `stbi_set_flip_vertically_on_load(true)`. Decoding off the render thread would race that global flag. Prefer a thread-safe variant or flip at upload.
  - I did not verify the thread-local variant exists in LWJGL 3.3.4. See Gaps.
- **Android.** `BitmapFactory` decodes on any thread, and upload happens in the GL thread. I did not verify details beyond general knowledge.
- **WebGL upload advice.** Skip mipmaps for 2D textures never zoomed out, prefer RGBA8, and use `texStorage2D` plus `texSubImage2D` ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API/WebGL_best_practices)).
- **Conclusion (design inference).**
  - Define the decoded image (`DecodedImage`: width, height, RGBA8, straight alpha, top-left origin) as the unit passed between the platform decoder and the backend uploader.
  - That is the seam between stage 2 (decode) and stage 3 (commit).

### 3.7 Hot reload

- **Bevy.** Enabling the `file_watcher` feature watches sources and reloads the asset in place, so handles and entities need no changes ([docs](https://docs.rs/bevy/latest/bevy/asset/struct.AssetServer.html), [asset docs](https://docs.rs/bevy_asset/)). `reload` only works on an already-loaded asset. Its processor is "a build system for assets", and the watcher reprocesses changed sources ([processor docs](https://docs.rs/bevy_asset/latest/bevy_asset/processor/index.html)).
- **KorGE.** JVM-only, through a dedicated Gradle task, and scoped to scene code ([docs](https://docs.korge.org/hot-reloading/)).
- **JVM `WatchService` limits.**
  - Implementations may poll, and timing and ordering are implementation specific.
  - One edit can generate several `ENTRY_MODIFY` events.
  - `OVERFLOW` requires a rescan.
  - A modify event gives no guarantee the writer has finished.
  - Registration is per directory, and recursion is the caller's job (the last point is general knowledge, not on the page).
  - Source: [`WatchService` javadoc](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/file/WatchService.html).
- **Conclusion (design inference).**
  - A debounce of tens of milliseconds, a rescan-on-OVERFLOW path, and a retry for half-written files are needed.
  - Reloading is a per-asset-type decision. A texture swaps pixels under the same token. An atlas swaps its regions. A font rebuilds its glyph atlas. A sound swaps the buffer for new plays only.
  - Failure keeps the previous version and logs.

### 3.8 Build-time tooling

- **Atlas packing.**
  - libGDX `TexturePacker` is the reference: defaults `pot = true`, 1024 max page size, `paddingX/Y = 2`, `bleed = true`, `rotation = false`, and nearest filters ([wiki](https://libgdx.com/wiki/tools/texture-packer)). Output is page images plus a text `.atlas` descriptor.
  - The libGDX Gradle recipe is a custom task and re-packing is the user's job. There is no built-in skip-if-unchanged. A real Gradle task gets that from input/output tracking.
  - KorGE uses a folder plus an `.atlas` marker file ([docs](https://docs.korge.org/atlas)).
  - Padding and bleed defaults matter for filtering seams. Rotation requires the draw path to support it, so default it off.
- **Font baking.**
  - msdf-atlas-gen (MIT) is a CLI that takes TTF/OTF and writes an atlas image plus JSON/CSV layout, with kerning in the JSON ([README](https://github.com/Chlumsky/msdf-atlas-gen)).
  - It is native code built with vcpkg, so the plugin would need to ship or fetch per-OS binaries. This weakens the case for making baking a v1 plugin task.
  - The glyph-atlas format itself belongs to the text-rendering lane.
- **Manifests.**
  - A build-generated manifest is how libGDX's HTML5 preloader works around the lack of directory listing (inferred from the wiki).
  - PixiJS needs a manifest for `backgroundLoadBundle`.
  - Web servers cannot list directories, so `list`/`load_folder`-style features require a manifest on web. The Bevy `AssetReader` makes `read_directory` mandatory, and the wasm reader must therefore implement it some other way.
- **Compressed GPU textures.**
  - Basis Universal's KTX2 container transcodes at load time to ASTC 4x4, BC7, BC1-5, ETC1/ETC2 or uncompressed, depending on GPU support. UASTC is the high-quality mode and ETC1S the small one ([Basis Universal README](https://skia.googlesource.com/external/github.com/BinomialLLC/basis_universal/+/refs/tags/v2_0)).
  - WebGL exposes per-family extensions (`s3tc` on desktop, `etc`/`etc1` on mobile, `astc` on newer hardware), and Basis is called out as the one-file answer ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API/WebGL_best_practices)).
  - Play Asset Delivery's "texture compression format targeting" lets one bundle carry several formats and the store picks one ([PAD docs](https://developer.android.com/guide/playcore/asset-delivery)).
  - For 2D sprites at 10k on screen, compression is a memory and load-time optimisation, not a prerequisite. Schedule it last.

### 3.9 Gradle plugin mechanics

- **Publishing.**
  - The Plugin Portal's `com.gradle.plugin-publish` applies `java-gradle-plugin` and `maven-publish` and produces the plugin marker artifact the `plugins {}` DSL needs.
  - It requires `website`/`vcsUrl`, says the group and artifact "must reasonably represent the organization, person, and the plugin" (the example is `io.github.johndoe`), has a manual approval that "can take a few days", and requires `configurationCache` (and `isolatedProjects`) compatibility declarations. Once declared, a feature must stay declared.
  - Source: [Gradle: publishing plugins](https://docs.gradle.org/current/userguide/publishing_gradle_plugins.html).
  - Publishing only to Maven Central also works. The marker comes from `java-gradle-plugin`, and per a forum answer the Portal proxies Central, but the plugin then does not appear in the Portal's web search ([Gradle forum](https://discuss.gradle.org/t/how-to-publish-mirror-a-plugin-available-from-maven-central-to-the-gradle-plugin-portal/41526)). Tier: one source's opinion.
  - Whether the `com.vanniktech.maven.publish` plugin used for the libraries handles plugin-marker publications is unverified (see Gaps).
- **Development.**
  - Gradle documents `pluginManagement { includeBuild("my-plugin") }` as the recommended way to develop a plugin alongside its consumers, and warns that `--include-build` can resolve unreliably with `plugins {}`.
  - Included builds do not share repositories, plugin management or `gradle.properties` with the root build ([Gradle: composite builds](https://docs.gradle.org/current/userguide/composite_builds.html)).
- **Incremental tasks.**
  - Use `InputChanges`, with `@Incremental` or `@SkipWhenEmpty` on a file input held in a `DirectoryProperty` or `ConfigurableFileCollection` ([Gradle: custom tasks](https://docs.gradle.org/current/userguide/custom_tasks.html)). The page does not cover `@CacheableTask` or configuration-cache rules, so treat those as unverified here.
  - Outputs for `REMOVED` inputs must be deleted. A task may have only one incremental action.
  - A packer should be a plain `@CacheableTask` rather than an incremental one, because repacking an atlas depends on every input at once.
- **Per-target resource wiring.**
  - Compose resources do it with one task per concern: `GenerateResClassTask`, `GenerateResourceAccessorsTask`, `AssembleTargetResourcesTask`, `PrepareComposeResources`, and platform files for Android and iOS ([plugin source](https://github.com/JetBrains/compose-multiplatform/tree/master/gradle-plugins/compose/src/main/kotlin/org/jetbrains/compose/resources); purposes inferred from names). Resources are packed as Android assets since 1.7.0, and published inside Maven artifacts since 1.6.10.
  - AGP's variant API supports generated asset directories (`SingleArtifact.ASSETS`, `variant.sources.*.addGeneratedSourceDirectory()`) per the [android/gradle-recipes](https://github.com/android/gradle-recipes) listing. A Google issue records that the call overwrote an already-set output directory, and a forum report records duplicate-merge errors when a generated folder was registered through `assets.srcDirs` ([issue 232805583](https://issuetracker.google.com/issues/232805583)).

## 4. How comparable projects do it (summary)

- **Compose Multiplatform resources.** Generated typed accessors, `suspend readBytes`, `getUri`, resources shipped in published artifacts, a Gradle plugin that routes resources per target.
- **KorGE.** A VFS of `suspend` file objects (`resourcesVfs`, `UrlVfs`, `MergedVfs`), folder-based atlas generation, JVM-only hot reload.
- **libGDX.** A `FileHandleResolver` for the source, an `AssetManager` for async plus refcounted loading, GWT preloading as the web workaround, `TexturePacker` as the offline step.
- **Bevy.** Pluggable async `AssetReader` per `AssetSource`, `Handle`/`AssetId`, `LoadState` split across the asset and its dependencies, a file watcher for hot reload, an asset processor as a build system.
- **Godot.** Threaded `ResourceLoader` with explicit status polling and five cache modes.
- **PixiJS.** A manifest of bundles with progress and background prefetch.

## 5. Pitfalls and regrets

1. **A blocking source contract.** libGDX shows the price: whole-directory preloading on web, an asset filter added later, and games that freeze if a file is read outside the manager.
2. **Putting GL ids or other context-bound state in the app-visible texture type.** The Android black-texture reports show the failure.
3. **A static or global asset manager that outlives the GL context** (libGDX's own warning).
4. **Unspecified decode semantics.** `createImageBitmap`'s `default` premultiplication and colour-space conversion are implementation-specific. Pin them.
5. **Nonportable names.**
   - Windows is case-insensitive, while Linux, web servers and APK asset paths are case-sensitive, so a mismatched-case name works in desktop development and fails on web or Android.
   - A cheap debug-time check against a manifest catches this. (Design inference from general platform behaviour, not a sourced finding.)
   - Reserve `#` and a scheme-like prefix now (Bevy's label separator caused an escape-hatch API).
6. **Directory listing.** Browsers cannot list directories. Any manager feature that assumes listing (preload by folder) needs a manifest on web.
7. **Android delivery splits.** Play Asset Delivery fast-follow and on-demand packs are not at a fixed path. A single `AssetManager` source will not see them.
8. **Handle lifetime edge cases.** Bevy needed fixes for removal while handles were alive and has a known problem with sub-asset lifetimes, so decide up front whether a sub-asset (atlas region) holds its parent.
9. **Hot reload by raw `WatchService`.** Duplicate events, polling latency on some platforms, `OVERFLOW`, half-written files. Debounce and rescan.
10. **Plugin identity churn.** KorGE's Gradle plugin moved namespaces and left a stale Portal page. Pick the plugin id under the Maven group at first publish, because consumers' `plugins {}` blocks depend on it.
11. **Once declared, always declared.** The Plugin Portal requires a compatibility feature (configuration cache) to stay declared in later versions.
12. **Generated Android assets.** Registering the output through `assets.srcDirs` caused duplicate-merge errors in one report. Use the variant API.

## 6. Recommendation for this repo

### 6.1 What 1.0 freezes (asset side)

- **`AssetSource`, minimal and `suspend`.**
  ```kotlin
  interface AssetSource {
      /** Whole-asset read. Never blocks the caller's thread; safe to call from any coroutine. */
      suspend fun read(name: String): Result<ByteArray>
  }
  ```
  - Failure type: an open `AssetException(name, cause)` hierarchy with `AssetNotFoundException` as the expected one. Failures are values, not thrown, per the repo rules.
  - Name grammar (documented and tested): UTF-8, `/`-separated, relative, no leading `/`, no `..`, no `\`, case-sensitive, and `#`, `?` and `:` reserved for future sub-asset labels and source selection.
  - Directory prefixes such as `textures/` and `sounds/` stop being library policy and become a `PrefixAssetSource` the game client wraps, or its own name constants. The graphics API takes full names.
  - Ship in the same module: `MemoryAssetSource` (the fake for tests and the `graphics-test` backend), `PrefixAssetSource` and `CompositeAssetSource` (first match wins, so mods or dev overrides are a one-liner).
- **Texture contract rules** (this affects the Q5 backend-contract lane, so only the asset-relevant parts are stated).
  - A backend creates textures from `DecodedImage`.
  - A `Texture` is an opaque backend token with `release()`.
  - Backends keep a rebuild recipe.
  - Failure to load yields a failed `Result`. The white fallback is a policy of the layer above the source (today's `TextureCache`), not of `AssetSource`.
- **Capability interfaces reserved but not frozen.**
  - `StreamingAssetSource`, `LocatableAssetSource`, `ListingAssetSource`, `ChangeNotifying`.
  - All are separate interfaces discovered with an `is` check, so third-party sources written against the 1.0 `AssetSource` keep working.
  - Adding an abstract method to a frozen interface later would break every third-party implementer.

### 6.2 Modules

- **Keep the draft's single multiplatform `assets` module, with loaders in its platform source sets.** This departs from the split plan's `assets-core` plus loaders inside each graphics backend, and from decision 11's "one module per backend family" in letter. I recommend it anyway.
  - The split plan puts the classpath loader in `graphics-desktop`, which forces an audio-only app to depend on the graphics backend. Audio depends on no graphics module, so the loader has to live in a module both can use.
  - The loaders are dependency-free platform glue: classpath and `Path` on `jvmMain`, `AssetManager` on `androidMain`, `fetch` on `wasmJsMain` (or `jsMain`).
  - This needs a user decision, since decision 11 is binding and I am not allowed to override it.
  - If the user insists on strictly per-family modules, the fallback is a contract-only `assets-core` plus tiny `assets-desktop`, `assets-android` and `assets-web` artifacts. The cost is three more artifacts to release by hand.
- **Asset manager: its own module, `asset-manager`, a KMP module with no graphics dependency.**
  - Depends on `assets` and `kotlinx-coroutines`.
  - Generic over `AssetLoader<T>`. Texture loaders register from the graphics side and sound loaders from the audio side, so audio can use the manager without graphics.
  - Post-1.0 and additive.
- **Hot reload:** not a module.
  - The filesystem source (`DirectoryAssetSource` in `jvmMain` of `assets`) implements `ChangeNotifying` using `WatchService`.
  - The manager subscribes if the source offers changes.
  - Both parts are Experimental behind `@RequiresOptIn`, and the manager only subscribes in an explicit "dev" configuration, so release builds cannot enable it by accident.
- **Gradle plugin: a separate included build** in a top-level folder (`gradle-plugin/`), wired through `pluginManagement { includeBuild(...) }` for the sample apps.
  - It is not part of `build-logic`, which holds private convention plugins for building the libraries, and it is not a normal library module, because it runs on Gradle's embedded Kotlin and JDK.
  - Keep its group and project names aligned with its publication coordinates so composite substitution works without rules.

### 6.3 Manager shape (post-1.0)

- `AssetManager.load<T>(name, params): AssetHandle<T>`: returns immediately, deduplicates by `(name, params)`, and exposes `state: Loading | Ready(value) | Failed(error)` plus an `await()` suspend.
- Lifetime by scope: `manager.scope()` or `AssetGroup` owns the handles it loads, and closing the group releases them. Reference counts are internal, which gives libGDX's shared-dependency behaviour. A named group is also the PixiJS-style bundle. `preload(group)` returns a `Flow<Progress>`. Do not expose raw refcounts, since Kotlin has no deterministic drop.
- Three-stage loader: `read` (IO dispatcher) then `decode` (worker) then `commit` (on the context owner's thread). The `commit` stage is the only one allowed to touch GPU, audio-device or other context state.
- Dependencies (a font depends on its atlas pages) are loader-declared and reported separately from the root's own state, as in Bevy.
- A texture reload or context rebuild reuses the same token, so handle holders see no change.
- No polling `update()` (libGDX): `suspend` and `Flow` replace it.

### 6.4 Plugin scope and order

1. **`manifest`.** A JSON listing of names, sizes and content hashes, with `formatVersion`. Needed for web preload groups and listing, and for cache-busting. Cheap to build and the lowest-risk first task.
2. **`atlas`.** A pure-Kotlin max-rects packer (libGDX defaults: padding 2, bleed on, rotation off, power-of-two off by default for WebGL 2 and GLES3). Output is page PNGs plus a descriptor whose format SpartanGraphics owns and versions. It reads and writes through declared inputs and outputs and is a `@CacheableTask`.
3. **`fonts`.** Wraps a text-lane-defined baked format. If that lane picks MSDF, shelling out to msdf-atlas-gen means shipping per-OS binaries, which is the main cost. Treat it as optional and last among the "atlas-like" tasks.
4. **`compress`.** KTX2/Basis, only after profiling shows memory or download pressure on Android or web.

- **Wiring:** each output goes to the right per-target directory: JVM resources, `wasmJs` resources, and Android generated assets through the AGP variant API (`addGeneratedSourceDirectory`), not through `assets.srcDirs`.
- **Declarations:** declare `configurationCache` compatibility from the first release, because the Portal requires it and never lets you retract it.
- **Versioning and decoupling:** coupling between plugin and runtime is by file-format `formatVersion`, not by shared code. That lets the plugin ship on its own cadence. The draft's "one version for the whole repo" would force a plugin release whenever the runtime changes, so exempt the plugin and say so.
- **Publishing:** the plugin id is `io.github.spartanlaboratories.assets`, matching the Maven group (the Portal's own guidance is an `io.github.<owner>` style id).
  - Prefer the Plugin Portal via `com.gradle.plugin-publish`. It builds the marker artifact, enforces the compatibility declaration, and makes the plugin discoverable.
  - Central-only is a lower-cost fallback (consumers add `mavenCentral()` to `pluginManagement.repositories`) if the extra credential and approval wait are unwelcome. That is a single-source judgement, so the user should choose.

### 6.5 Landing order

| When | Asset-side deliverable |
|---|---|
| **0.1.0** (Phase 1) | `assets`: the `suspend` `read` contract, name grammar, `AssetException`, `ClasspathAssetSource`, `MemoryAssetSource`, `PrefixAssetSource`. Desktop `TextureCache` ported to read through it, with the white fallback as policy. Contract not yet frozen. |
| **Fonts and text input** | `DecodedImage` (the pixel contract). A `DirectoryAssetSource` for desktop development (needed for font files in dev and a prerequisite for hot reload). Reserve capability interfaces. |
| **Android backend** | `AndroidAssetSource` over `AssetManager`. The texture rebuild recipe proven by a real context loss in the sample app. Decide on Play Asset Delivery access (probably defer, but keep `LocatableAssetSource`). |
| **Web backend** | Fetch-based `WebAssetSource` with `resourcePathMapping`-style base URL configuration. `createImageBitmap` decode with pinned options. Context-restored rebuild proven in the sample app. |
| **1.0** | Freeze `AssetSource.read`, the name grammar, the error base types and the texture recipe rule. Graduate `AssetSource` to `@SupportedExtension` with the in-memory source and one platform source as the worked examples. |
| **After 1.0** | `asset-manager` (handles, groups, progress), then plugin `manifest`, then sprite atlas runtime plus plugin `atlas`, then hot reload (Experimental), then font baking, then `compress`. Capability interfaces graduate as backends use them (streaming with audio mixing and music). |

Why not the obvious alternative of building the manager first and freezing it at 1.0: the manager touches loaders, groups and handles, whose right shapes depend on text (font dependencies), audio (streaming) and atlas work, none of which will have shipped. The 1.0 contracts only need to leave the door open.

## 7. Gaps

- **Tooling limits.** My search session ran out partway through (the WebSearch quota was exhausted), so a few checks I intended did not happen. I used no Firecrawl.
- **Not researched:** free-tex-packer, Hiero/BMFont, KorGE's atlas generator internals, and the exact current Compose web resource caching and preload API. Also not researched: how LWJGL exposes a thread-local `stbi_set_flip_vertically_on_load` (I believe `stbi_set_flip_vertically_on_load_thread` exists, but did not verify), Android `BitmapFactory` threading and premultiplication, and Godot's threading guarantees beyond the page I read.
- **Not verified:**
  - Whether the `com.vanniktech.maven.publish` plugin used for the libraries can publish `java-gradle-plugin` marker artifacts. If it cannot, the Plugin Portal route is simpler.
  - The behaviour of `Result` as a `suspend` return type in a public, ABI-frozen interface. Value-class mangling and historical interop bugs with `Result` in suspend functions and interfaces may apply. Check against the Kotlin 2.1.20 compiler and the ABI validator before freezing. If a problem appears, the fallback is a library-owned sealed result type, but that departs from the repo's `Result` rule.
  - The exact Gradle rules for `@CacheableTask` and configuration-cache-safe tasks. The page I fetched covered incremental tasks only.
- **Taken from general knowledge** (the fetched pages were truncated or silent): `AssetManager.open` access modes and the `openFd` uncompressed-only limit, `WatchService` not being recursive, and Android compressing assets by default. Verify before relying on them in design docs.
- **Weak sources.** The PixiJS bundle and progress API detail comes from third-party mirrors, not the official page I could reach. The claim that the Plugin Portal proxies Central is a single forum answer. The KorGE namespace history comes from search-result snippets.
- **The compressed-texture support matrix** (which desktop, Android and browser targets support which formats, in practice) was not found. Only the Basis target list, WebGL extension names and Play's format-targeting feature are sourced.
- **Okio on `wasmJs`.** Unresolved and single-source (a resolution failure for the Node filesystem module). It matters only if a stream type is ever put in the contract.
- **Decisions for the user.**
  - The `assets` module naming and loader placement, which deviates from decision 11 in letter.
  - Plugin Portal versus Central-only publishing.
  - Whether the plugin is exempt from the one-repo-wide-version rule.
  - Whether Play Asset Delivery is in scope for the Android loader.
