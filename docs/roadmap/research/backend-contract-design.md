# Backend contract design: abstraction level and post-1.0 evolution

## 1. Question

At what level of abstraction should SpartanGraphics' backend contracts sit (the split plan's high-level `Renderer`, a low-level GPU-resource layer, or a hybrid), and how can they evolve after 1.0 without breaking backend implementers?

## 2. Bottom line

- **Freeze a hybrid, pitched at "resources plus a recorded frame", not at "shapes" and not at "GL calls".**
  - Common code (graphics-core) owns everything shape-shaped: tessellation, culling, layer and z sorting, clip handling, outlines, text-to-quads, **and batching**.
  - It hands the backend a plain-data `RenderFrame`: an ordered list of batches. Each batch is a pipeline id, a texture binding, a vertex/index range in one bulk array, and a state block (clip rect, blend).
  - The backend implements about a dozen operations: create or delete a texture, create a pipeline from a descriptor, `submit(frame)`, a capability query and a device-lost signal.
  - Impeller (Flutter's renderer), Kool and KorGE all split it this way: a 2D framework above a backend-agnostic renderer API.
- **Reject the split plan's literal `Renderer` ("draw one frame of shapes and their outlines") as the frozen contract.**
  - Every new shape (decision 10 lists about nine) would become a new obligation on every backend, including third parties.
  - Batching, sorting and clip splitting would be re-implemented in each backend. That logic could not be tested without a GPU, and it is the logic G5 is meant to get right.
  - There would still be no route to custom shaders without a pipeline concept.
  - It also reduces to the hybrid anyway: `Renderer.render(frame)` with a lower-level `frame`. G4 can adopt that shape now at no cost.
- **Also reject a GL20-style fine-grained layer** (libGDX). It is shaped like OpenGL's global state machine and does not map onto WebGPU, Metal or Vulkan. The plan itself flags macOS OpenGL deprecation as a risk. It makes about 160 cross-boundary calls a frame the natural unit, and it makes a recording backend record noise. Godot, the other engine with a two-level design, cannot offer its low-level `RenderingDevice` on its GL ("Compatibility") renderer for the same reason.
- **Evolve by additive means only, and put the discipline into the contract's shape from day one.**
  - Interfaces whose new members always carry a default body, compiled as real JVM default methods (set the compiler flag explicitly; do not trust the default).
  - A keyed `Capability` set rather than boolean fields or data classes.
  - Optional sub-interfaces obtained through a typed lookup, returning null when absent.
  - Descriptor objects instead of long parameter lists.
  - Post-1.0 features land as new content in existing commands (more tessellation, new pipeline ids behind capabilities), not as new abstract methods.
- **Split the platform contracts into small independent SPIs** that evolve at different rates: `Surface` (with lifecycle events), `FrameDriver` plus `Clock`, `RenderBackend` (with textures as a sub-interface), `InputSource`, `TextMeasurer`, `AssetSource` and the audio backend. **GPU device loss is a separate channel from app and surface lifecycle**, signalled by an epoch counter the common layer reacts to by re-creating resources.
- **Reserve at 1.0 the slots that post-1.0 features need**, even if no 1.0 backend implements them: a render-target field on the pass (RTT, minimap, fog mask), a depth/stencil request on the surface config, a pipeline-state descriptor that includes depth, cull and vertex layout, and an `xyz` vertex position. Mark the unproven parts Experimental (section 7).

## 3. Findings

Each finding carries an evidence tier: **Consensus** (several independent credible sources or a spec), **Practice** (what mature libraries visibly do), **Opinion** (one source's view or my inference).

### 3.1 Where comparable libraries draw the line

| Library | What the backend implements | Where 2D drawing lives | Tier and source |
|---|---|---|---|
| libGDX | `Graphics` (exposes `GL20`/`GL30`/`GL31`/`GL32`), `Input`, `Audio`, `Files`, `Net`, `Application`. | Core (`SpriteBatch` etc.). The GL interfaces are thin: `GL20` has ~160 methods, `GL30 extends GL20`. | Practice. [App framework](https://libgdx.com/wiki/app/the-application-framework), [`Graphics.java`](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/Graphics.java), [`GL30.java`](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/GL30.java) |
| KorGE | Abstract class `AG`. `draw(...)` takes a framebuffer, vertex arrays, a program, blending, uniforms, texture units, stencil, colour mask, depth, **scissor**, cull face and instance count. It builds an `AGBatch` command object and calls `execute(command)`. | Common code (batching and sprites above `AG`). | Practice. [`AG.kt`](https://github.com/korlibs/korge/blob/main/korge-core/src/commonMain/kotlin/korlibs/graphics/AG.kt) |
| Kool | `RenderBackend` is ~15 members: `features`, `deviceCoordinates`, `renderFrame(frameData, ctx)`, `generateKslShader`, `createOffscreenPass2d`, `uploadTextureData`, `downloadTextureData`, `frameGpuTime`. Backends exist for GL, WebGPU and Vulkan. | Common scene and render-graph code builds the frame. The backend consumes frame data. | Practice. [`RenderBackend.kt`](https://github.com/kool-engine/kool/blob/main/kool-core/src/commonMain/kotlin/de/fabmax/kool/pipeline/backend/RenderBackend.kt), [`BackendFeatures.kt`](https://github.com/kool-engine/kool/blob/main/kool-core/src/commonMain/kotlin/de/fabmax/kool/pipeline/backend/BackendFeatures.kt) |
| sokol_gfx | Immutable-pipeline, pass-and-bindings API: buffers, images, samplers, shaders, pipelines, `sg_begin_pass`, `sg_apply_scissor_rect(x,y,w,h, origin_top_left)`. Runtime `sg_features` and `sg_limits`. A **dummy backend** (no-op stubs) exists specifically for command-line tests. | Not in the library. | Practice. [`sokol_gfx.h`](https://github.com/floooh/sokol/blob/master/sokol_gfx.h) |
| Impeller (Flutter) | A backend-agnostic renderer layer above a per-API backend directory. | Layers above: Entity ("a framework for building 2D renderers"), then Aiks (a Skia-style API the README calls "probably short-lived"), then display list. | Practice (secondary excerpt of the README). [Impeller README](https://flutter.googlesource.com/mirrors/engine/+show/refs/heads/zanderso-patch-6/impeller/README.md), [standalone guide](https://flutter.googlesource.com/mirrors/engine/+/refs/heads/main/impeller/docs/standalone_gles.md) |
| Godot | Two levels. `RenderingServer` is high-level. `RenderingDevice` is "much lower-level" and is **not available in headless mode or with the Compatibility rendering method**. | Server side. | Practice. [RenderingDevice docs](https://docs.godotengine.org/en/stable/classes/class_renderingdevice.html) |
| MonoGame | `GraphicsDevice` is a **concrete `partial class`** with per-platform partial files, not an SPI. Third-party backends are not a supported case. | Core. | Practice. [`GraphicsDevice.cs`](https://github.com/MonoGame/MonoGame/blob/develop/MonoGame.Framework/Graphics/GraphicsDevice.cs) |
| Compose / Skia | `Canvas` is a high-level interface (~35 members: `drawRect`, `drawPath`, `clipRect`, `saveLayer`, `enableZ`...). Android and Skia implement it **in-house**. | The GPU layer is internal to Skia. | Practice. [`Canvas.kt`](https://github.com/androidx/androidx/blob/androidx-main/compose/ui/ui-graphics/src/commonMain/kotlin/androidx/compose/ui/graphics/Canvas.kt) |

**What this shows**
- The high-level canvas shape (Compose, Skia) works for projects whose backends are all in-house. None of those projects promise that third parties can implement it. MonoGame does not even have an SPI.
- Every project that does want pluggable backends puts shape and batching logic **above** the contract. KorGE, Kool, sokol-based engines and Impeller all do this.
- The best-fit precedent is Impeller's layering. Even there, the high-level API layer (Aiks) is the part expected to churn, while the lower renderer/HAL is the durable one. That is the argument for freezing the lower layer and keeping shapes app-facing.
- Skia Graphite's public model (a `Context`, `Recorder`s that produce `Recording`s) also separates recording work from submitting it. This is a secondary excerpt only; I did not read the Graphite source.

### 3.2 Evidence on how contracts break

1. **Adding an abstract member breaks every implementer.**
   - libGDX 1.12.0 added `touchCancelled` to `InputProcessor` and listed it as a `[BREAKING CHANGE]` ([CHANGES](https://github.com/libgdx/libgdx/blob/master/CHANGES), [issue #6871](https://github.com/libgdx/libgdx/issues/6871)). The interface still has no `default` bodies.
   - kotlinx.coroutines marks `SharedFlow` and `StateFlow` with `@SubclassOptInRequired(ExperimentalForInheritanceCoroutinesApi)`. The marker's own text: "new methods may be added in the future, which would break the inheritance" ([Annotations.kt](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/Annotations.kt), [SharedFlow.kt](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/flow/SharedFlow.kt)). Tier: Practice.
2. **Long parameter lists are a freezing hazard.** Kotlin's own guidance: "Non-default and default arguments can both break compatibility", and `@JvmOverloads` does not preserve binary compatibility for Kotlin callers ([Kotlin API guidelines](https://kotlinlang.org/docs/api-guidelines-backward-compatibility.html)). KorGE's `AG.draw` has ~20 parameters. Tier: Consensus for the Kotlin rule, Opinion for the inference that descriptor objects are the cure.
3. **Interface default bodies need the right JVM mode.** Kotlin documents three `-jvm-default` modes. `enable` generates real default methods plus compatibility bridges. `no-compatibility` skips the bridges. `disable` generates only `DefaultImpls` ([Java interop](https://kotlinlang.org/docs/java-to-kotlin-interop.html)). That page does not state the default across compiler versions, so pin the flag explicitly in the convention plugin rather than relying on it. A real JVM default method lets an implementer compiled before the method existed keep working. This is my inference from how the JVM resolves interface methods. Tier: Opinion; verify with a two-version test (section 8, Gaps).
4. **`@SubclassOptInRequired` exists for exactly this split** ("stable to use, unstable to implement") and is "Since Kotlin 2.1" in the stdlib docs ([opt-in docs](https://kotlinlang.org/docs/opt-in-requirements.html), [API page](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/-subclass-opt-in-required/)). The requirement is not propagated to nested classes. It targets open or abstract classes and non-functional interfaces. Tier: Consensus (official docs).
5. **Capability and optional sub-interface pattern.**
   - libGDX: `GL30 extends GL20`, with `isGL30Available()` and `getGL30()` returning null when absent. Later `GL31` and `GL32` followed the same way.
   - Kool: a `BackendFeatures` data class (compute shaders, cube-map arrays, reversed depth, ...) plus a `DeviceCoordinates` object that surfaces NDC differences.
   - KorGE: `AGFeatures` with `isInstancedSupported`, `isFloatTextureSupported` etc., defaulted through a `parentFeatures` chain.
   - sokol: `sg_features` with `origin_top_left`, `compute`, `draw_base_instance` and others, and `sg_limits`.

   Tier: Practice. A flat list of booleans grows with every feature. In Kotlin a `data class` of N booleans is also binary-fragile, because the constructor and `copy` change when a field is added (consistent with the Kotlin guideline in point 2).
6. **Binary compatibility off the JVM is not settled in what I could verify.** The Kotlin binary-compatibility-validator describes its KLib support as "experimental and a subject to change", and it reports **any** difference in the dump as an error ([README](https://github.com/Kotlin/binary-compatibility-validator)). I found no source stating that "adding a defaulted member to an interface is klib-compatible". See Gaps.

### 3.3 What a recording backend needs (testability)

- sokol ships a dummy backend because "this is useful for writing tests that need to run on the command line" ([`sokol_gfx.h`](https://github.com/floooh/sokol/blob/master/sokol_gfx.h)). KorGE's `AG` routes through `execute(command)`, and it has an `AGToCommandChannel` that turns draws into data. Tier: Practice.
- **The consequence for SpartanGraphics (inference, Opinion):** a recording backend is only as useful as the logic that sits above the contract.
  - With a shapes-level contract (the split plan), a recorder captures the shape list. That is what graphics-core already tests. Batching, clip splitting and outline geometry are inside each real backend and so untestable without a GPU. The plan itself says G5 "can't be unit-tested without a GPU".
  - With a frame-level contract, a recorder captures the batches. Batching, sorting, clip-induced splits, draw-call counts and vertex counts become Level 2 and Level 4a assertions. It also gives the Level 4c gate a GPU-independent metric: draw calls and bytes per frame for N sprites.

### 3.4 Lifecycle and context loss

- **KorGE** models a lost GL context in the backend as `contextLost()` plus an incrementing `contextVersion` ([`AG.kt`](https://github.com/korlibs/korge/blob/main/korge-core/src/commonMain/kotlin/korlibs/graphics/AG.kt)). The same file carries a comment `@TODO: Unify beforeDoRender, startFrame --- dispose, finish & endFrame`. Six overlapping lifecycle hooks is the sprawl to avoid. Tier: Practice.
- **libGDX** recovers from Android context loss through "managed" textures that it re-uploads. Textures built straight from a `Pixmap` are not managed, so the app must reload them itself ([LibGDX textures wiki](https://github.com/mattdesl/lwjgl-basics/wiki/LibGDX-Textures); the forum threads from the search show black or garbled textures after resume). Tier: Practice. Lesson: the common layer must keep the recipe (asset name or CPU pixels) for every texture it creates.
- **MonoGame** declares `DeviceLost`, `ResourceCreated` and `ResourceDestroyed` events but its `GraphicsDevice.cs` carries "TODO Graphics Device events need implementing", and `GraphicsDeviceStatus` always returns `Normal` ([source](https://github.com/MonoGame/MonoGame/blob/develop/MonoGame.Framework/Graphics/GraphicsDevice.cs)). Lesson: a lifecycle event the contract declares but no backend raises is worse than none.
- **Browser visibility:** `visibilitychange` fires on tab switches, minimising, and (on mobile) app switches; `hidden` is "the last event that's reliably observable" and the page should stop UI updates ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event)). I could not confirm in this session that `requestAnimationFrame` is paused in hidden tabs.
- **Android `GLSurfaceView` context preservation and the WebGL `webglcontextlost` event:** I could not fetch the primary docs (session limit). They are listed in Gaps and are not relied on below as facts, only as design requirements to verify.

### 3.5 Interop cost per call

- A Kotlin Slack thread (JetBrains-side answer) says passing data to JS and "dynamic things, like interface method calls" tend to be slower on Kotlin/Wasm, and that arrays must be copied or wrapped to cross the boundary ([thread](https://slack-chats.kotlinlang.org/t/10259826/i-am-interested-in-using-kotlin-wasm-to-make-a-3d-game-i-man)). Tier: Opinion (one chat answer). Nobody I found published per-call numbers.
- Design implication (inference): make the unit of crossing "one bulk array plus a short command list per frame", not "one call per draw". That favours the frame-level contract (O(batches) crossings, one bulk copy of the vertex data) over the GL-call-level one (O(sprites x state calls) crossings). The brief asks me not to supply performance numbers, so I give none.

### 3.6 Clip rectangle and batching

- sokol requires `sg_apply_scissor_rect(x, y, w, h, origin_top_left)`, and notes that `sg_begin_pass()` resets viewport and scissor. KorGE's `draw` carries a `scissor` parameter on every call. Kool exposes `DeviceCoordinates` (NDC Y direction, depth range) because backends disagree. Tier: Practice.
- **Lessons:** (a) the contract must fix one clip convention (screen pixels, origin top-left, y-down) and make the backend convert, because the origin flip is the classic GL-versus-others trap; (b) a clip change **forces a batch break**, so splitting on clip belongs in common code, where it can be tested. G3's clip rectangle and G5's scissor fit this directly: G3 adds `clip` to the batch state, G5 just applies it.

## 4. How comparable projects do it (signatures worth copying)

- **KorGE (command object):** `AG.draw(frameBuffer, frameBufferInfo, vertexData, program, drawType, vertexCount, indices, ..., scissor, cullFace, instances)` builds `AGBatch(...)` and calls `execute(AGCommand)`. A data-shaped command is recordable, but the signature itself is the cautionary part (about 20 positional parameters).
- **Kool (frame-level):** `fun renderFrame(frameData: FrameData, ctx: KoolContext)`, `fun uploadTextureData(tex: Texture<T>)`, `val features: BackendFeatures`, `val deviceCoordinates: DeviceCoordinates`, `val frameGpuTime: Duration`. Frame-level, with a features object and GPU timing exposed, which maps well to the 4c gate.
- **libGDX (capability plus sub-interface):** `boolean isGL30Available(); GL30 getGL30();`
- **sokol (immutable pipeline objects, no-op backend):** `sg_make_pipeline(desc)`, `sg_apply_pipeline`, `sg_apply_bindings`, `sg_draw`, `sg_query_features()`.
- **Kotlin precedent for guarded implementation:** `@SubclassOptInRequired(UnstableApi::class) interface CoreLibraryApi`.

## 5. Pitfalls and regrets

1. **The flat feature struct.** Every new capability adds a field to a frozen type. Use a keyed `Capability` set, with the keys defined as an open class rather than an enum.
2. **Positional mega-signatures** (KorGE's `draw`, GL-style setters such as libGDX's `setGL20..setGL32` on `Graphics`). Use descriptor classes with defaults, built through a builder, and not `data class` in frozen API.
3. **Declared but unraised events** (MonoGame `DeviceLost`). Every event in the SPI needs a backend conformance test, or it does not ship.
4. **Treating app pause as context loss, or the reverse.**
   - Android pause, a browser tab hiding, and GPU device loss are three different events with different recoveries.
   - Merging them forces every backend to over-recover.
   - Splitting them without saying who owns the resources leads to libGDX's "unmanaged texture" bug class.
5. **A GL-shaped frozen contract when macOS and the plan's own risk list already point at Metal.** All three official backends will likely be GL-family (GL 3.3, GLES 3, WebGL 2), which validates the contract only against GL semantics. Review the pipeline descriptor against WebGPU/Metal's immutable pipeline state and explicit passes before 1.0. (Researcher 4 owns the web GPU API choice. If it is WebGPU, you get a non-GL check for free.)
6. **Shape-level contracts accumulate obligations.** Every new shape means every backend (and every third-party one) implements new drawing code.
7. **Two sources of truth for resources.** If the backend holds the only copy of a texture, a lost context cannot be repaired. The common layer must own the recipe.
8. **Abstract base classes as the evolution tool** re-create the fragile-base-class problem the user's library rules avoid. With real JVM default methods they are not needed for evolution. Keep them only to share logic.
9. **Enums and sealed hierarchies in the backend-facing data** (command kinds, capability keys). Adding a case breaks exhaustive `when` in backends. Use open interfaces or classes, and give an unknown kind a defined fallback.
10. **Overpromising non-JVM binary compatibility.** See Gaps.

## 6. Recommendation for this repo

### 6.1 Contract level: the "frame-level hybrid"

Keep the split plan's skeleton (`graphics-core` common, `graphics-desktop` backend) and change what `Renderer` means.

**Common code (graphics-core) owns:**
- Shapes (app-facing, closed, per decision 11) and the world and screen layers.
- Camera, culling, picking, sorting by layer and z.
- Outline geometry, clip handling, text-to-geometry, tessellation for every shape.
- **Batching**: it merges draws that share pipeline, texture and clip into `DrawBatch`es, writing vertices into one bulk array.

**The backend contract (`RenderBackend`, replacing the plan's `Renderer`) owns:**
- `capabilities`.
- A texture sub-interface (create from pixels, update region, delete).
- Pipeline creation from a descriptor.
- `submit(frame: RenderFrame): Result<...>`.
- A device-epoch signal.
- Optional `FrameStats`, which feeds the 4c gate with draw calls and bytes.

The sketch below is illustrative, not a design.

```kotlin
interface RenderBackend {
    val capabilities: Capabilities                   // keyed set; unknown keys read as "no"
    val textures: TextureStore                       // create / update / delete
    val deviceEpoch: Int                             // bumps when the GPU context is lost and replaced
    fun createPipeline(desc: PipelineDescriptor): Result<PipelineHandle>
    fun submit(frame: RenderFrame): Result<Unit>     // render thread only
    fun <E : Any> extension(key: ExtensionKey<E>): E? = null   // optional sub-interfaces
}
// RenderFrame = target/clear + ordered passes; each pass = target (null = surface),
// vertex array slice, then DrawBatch(pipeline, texture, indexRange, clip: PixelRect?, blend, ...)
```

**Why this level:**
- **New features land in common code.** Lines, circles, polygons, rounded rectangles, gradients, 9-slice, sprites, frame animation, tile maps, particles, glyph quads and outlines of every shape are all expressible as textured, vertex-coloured triangles through **one universal pipeline** that every backend must provide. Adding any of them is a minor release of graphics-core. No backend changes.
- **New pipelines are the only real new obligation,** and they are capability-gated with a fallback. SDF circles or analytic gradients are optional optimisations. The default is CPU tessellation into the universal pipeline.
- **Custom shaders** need a `ShaderProgram` plus `PipelineDescriptor` concept anyway. Having the descriptor in the contract from 1.0 (vertex layout, blend, depth, cull, topology) means the Experimental custom-shader seam fills in an existing slot, and a later 3D track also uses it.
- **The recording backend becomes powerful.** `graphics-test`'s recorder (draft proposal) just stores the `RenderFrame`. It also becomes the worked example that `@SupportedExtension` requires.

### 6.2 Platform SPI decomposition

Each is a separate small interface, so they can evolve at their own speed (input will change far more than the renderer, given decision 10's touch, IME, clipboard and cursors):

| SPI | Contents | Evolution note |
|---|---|---|
| `Surface` | Pixel size, density, window size separate from framebuffer size, close request, config (depth/stencil/MSAA/colour space requests), **lifecycle events** | Config as an extensible descriptor. Events as an open `SurfaceEvent` hierarchy, not a sealed one. |
| `FrameDriver` + `Clock` | Platform calls `onFrame(time)`. Only called while active. Clock is monotonic, and the contract says a hidden or paused gap is **not** a frame delta. | Policy (`maxFrameDelta`) is a parameter, per the user's rules. |
| `RenderBackend` | As above | Textures, targets and readback as sub-interfaces behind `extension(...)`. |
| `InputSource` | Pointer, keys, scroll now. Touch, gestures, IME, clipboard and cursor later | Each later capability is an opt-in sub-interface plus a capability key. |
| `TextMeasurer` | Measure a string at a size | A separate real-font `GlyphRasterizer` extension later (the same principle: the backend rasterises a glyph, common code owns atlas and layout). Researcher 8 owns the details. |
| `AssetSource` | Name to bytes | Whether it is `suspend` is the draft's open question. Researcher 11's lane. |
| Audio backend | No graphics dependency | Own contract, own version. |

A per-backend factory, for example `DesktopPlatform.create(config): GraphicsPlatform`, bundles the parts. A third party can then swap one part (a different `InputSource` on the stock renderer) without forking the rest.

### 6.3 Lifecycle crossing

- **Three channels, three owners:**
  1. *App lifecycle* (Paused, Resumed, LowMemory, FocusChanged, VisibilityChanged): emitted by `Surface` or `FrameDriver` and delivered to the game and to the common layer.
  2. *Surface lifecycle* (Created, Resized, Destroyed): from `Surface`.
  3. *GPU device loss*: `RenderBackend.deviceEpoch`, plus a listener, owned by the backend. Pause does **not** imply loss. A backend raises device loss only when its context really was replaced.
- **Recovery rule, stated in the contract:** after an epoch bump all backend handles are invalid. The common layer rebuilds pipelines from their descriptors and re-uploads textures from retained recipes (asset name or CPU pixels) lazily on the next frame. Render-target contents are **not** recoverable and the game must re-render them.
- **Threading rule in KDoc:** every backend call happens on the render thread (the GL thread on Android, the single JS thread on web, the main-thread constraint on macOS).
- **Conformance:** ship a published abstract conformance suite (Level 3) that any backend runs. It must assert that every declared event can fire.

### 6.4 Evolution toolkit (all additive)

1. Every new member has a default body (`Result.failure(Unsupported)`, a no-op, or "capability false"). Pin `-jvm-default` explicitly in the convention plugin so the bodies compile to real JVM default methods.
2. New operations ship as an **optional sub-interface** found through `extension(key)`, and new behaviour as a **capability key**. Capability keys are an open class, not an enum.
3. Frozen descriptors are final classes with a builder, never `data class`, and never long parameter lists. New fields are new builder properties with defaults.
4. The common layer never calls a feature its backend did not declare, and reports failure through `Result`. No silent fallbacks.
5. `RenderFrame` and command kinds are open types with a documented "unknown means skip and report" rule, not sealed.
6. The one-version BOM from the draft is good. Add a cheap guard: a backend factory declares the minimum core contract revision it was built for, and install fails early with a clear `Result` instead of an `AbstractMethodError`.
7. Run `abiValidation` on the JVM from G1. Treat non-JVM as source-compatible only (Gaps).

### 6.5 What Phase 1 should adopt now

- **G3:** define `DrawBatch`, `RenderFrame` and the batch state block (clip rectangle, blend, pipeline id, texture) in graphics-core, and have the draw model emit them. The G3 clip rectangle then becomes a field in this state, not a renderer special case. Reserve `xyz` in the vertex data and a 4x4 transform from the start (the app-facing side is researcher 6's lane).
- **G4:**
  - Write the SPIs in section 6.2 as above, with defaults and KDoc. Use `RenderBackend` in place of the plan's `Renderer`.
  - Include the capability set, the device-epoch signal, the pass-target slot and the surface config descriptor.
  - Guard them with `@SubclassOptInRequired(ExperimentalGraphicsApi)` during 0.x, as the draft proposes. 0.x releases may break them. At 1.0 swap the guard for `@SupportedExtension`, which is "implementing is supported", and keep the guard only on the still-Experimental sub-interfaces.
  - Ship `graphics-test` (recording backend, scripted input, manual clock, fixed surface) alongside, as the draft proposes. Without it, nothing tests the contract.
- **G5:**
  - Implement only `submit`, the texture store and the universal pipeline in the desktop module.
  - Add two-level tests: batching and clip splitting in Level 2 and 4a against the recorder, and a pixel-readback golden test as Level 3 or 4b on a machine with a GPU.
  - Have G5's renderer report `FrameStats` so the 4c gate does not depend on a GPU.

### 6.6 What stays Experimental at 1.0

- Custom shader and pipeline authoring (decision 10 already schedules it Experimental and last).
- Render-target creation (until a consumer, such as a minimap or fog mask, has built against it).
- Pixel readback.
- Depth and 3D vertex formats and multi-attachment passes (reserved, not promised).
- Instancing and any multithreaded recording.

Graduate each once a consumer builds against it, and record it in the CHANGELOG, per the user's rule.

### 6.7 What a later 3D track would need from the contract

Depth and stencil attachments, which the surface config must be able to request. A pipeline descriptor with depth test, cull mode, topology and arbitrary vertex layouts. 32-bit indices. 4x4 matrices and uniform buffers. Cube and depth texture formats. Multi-pass frames with offscreen targets, and a shader-program concept. Every one is already a slot in the hybrid above. None would require a new abstract method on a frozen interface.

## 7. Where I would reject the obvious option

The obvious option is the split plan's high-level `Renderer`, because it is simple and matches "what every platform renderer has to draw". It is the cheapest first backend and the most expensive third-party contract: the closed shape set is closed for apps but not closed in time, and decision 10 schedules nine more shapes, outline kinds, sprites, tile maps and particles after 1.0. Each would be a new obligation on every backend, and none of the batching that G5 is supposed to get right would be testable without a GPU.

## 8. Gaps

- **Klib (Kotlin/JS, Wasm, Native) binary compatibility.** I could not establish whether adding a defaulted member to a frozen interface is klib-compatible, nor how klib ABI behaves across compiler versions. The binary-compatibility-validator calls its klib support experimental. **Blocks a claim, not a design:** I recommend promising source compatibility off the JVM and JVM binary compatibility only. A spike with a core compiled at 0.1 and a backend compiled against 0.2 (and the reverse) on JS and Wasm would settle it.
- **JVM default-method claim (section 3.2, point 3)** rests on the Kotlin docs' mode table plus my reasoning about the JVM. It was not tested. The repo is on Kotlin 2.1.20, whose default mode I did not verify. Test it with a two-version fixture in G1's build-logic.
- **Android and web lifecycle primary sources were not fetched** (session limit). Unverified from this session: `GLSurfaceView`'s EGL-context behaviour on pause (`setPreserveEGLContextOnPause`) and when `onSurfaceCreated` fires again, the WebGL `webglcontextlost`/`webglcontextrestored` events and `preventDefault`, and whether `requestAnimationFrame` stops in hidden tabs. My three-channel split in 6.3 depends on these being distinct events. Verify before G4 freezes the event lists.
- **Per-call and per-copy cost on Kotlin/Wasm** is unquantified (one Slack answer only). Researcher 7 should measure; my "bulk array plus short command list" shape is the conservative choice either way.
- **Skia Graphite and bgfx internals** were only partly confirmed (secondary excerpts, and the bgfx docs pages did not load). I used them only as supporting examples. The conclusions do not rest on them.
- **Value classes in frozen SPI signatures:** I believe `@JvmInline value class` parameters mangle JVM method names and complicate binary compatibility. I did not source that. Treat as a question for the idiomatic-Kotlin researcher (3), and use plain classes for handles until it is answered.
- **Not covered:** the GPU API per platform, shader language strategy and batching numbers (other researchers), and the audio SPI design beyond "separate contract". The brief did not state whether `AssetSource` being `suspend` is decided. The draft proposes it and the baseline plan does not.
- **Firecrawl:** not used, as instructed. Primary sources for bgfx internals, Godot's `RenderingDeviceDriver` and Skia's Graphite docs are the ones I would fetch first if credits allow.
