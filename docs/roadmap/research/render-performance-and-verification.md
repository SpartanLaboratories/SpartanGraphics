# Render performance and verification without a GPU

## 1. Question

How should SpartanGraphics hit and prove its render target (about 10k sprites at 60 fps on a mid-range desktop GPU, with Android and web targets still to be set)? And how can its renderers be verified automatically when CI has no GPU?

## 2. Bottom line

- **Batch with a CPU-built vertex stream, flushed on state change, and keep draw order.**
  - Every mature 2D batcher does this: libGDX, MonoGame, KorGE, PixiJS. Bevy is the exception: it draws instanced quads, with one instanced draw per run of sprites sharing a texture.
  - Use 4 vertices per quad and a static index buffer built once.
  - Upload one orphaned or ring-buffered vertex buffer per flush. Do not rely on persistent mapping, which needs GL 4.4.
  - Sort only by layer. Batch consecutive runs of the same state. Atlases are what make runs long.
  - The monolith's cost is per-actor uniforms and per-actor draws. Both go away when per-sprite data moves into vertices.
- **10k sprites is a modest target for a batched renderer.**
  - PixiJS v8 reports 100k moving sprites at about 15 ms CPU and about 2 ms GPU (hardware unstated).
  - "10k sprites" is meaningless without a scene profile: sprite size, overdraw, atlas count, blend. Fill rate and overdraw, not sprite count, decide mobile results.
  - The gate must pin the scene profile.
- **The 4c gate has two halves.**
  - *Structural counters run in CI on any machine, GPU or not.* They are draw calls, flushes by reason, bytes uploaded, vertices, and allocated bytes per steady-state frame. All are deterministic.
  - *Timing is a reference-hardware release gate* (manual or self-hosted runner). It uses frame-time percentiles (p50, p95, p99, max), a warm-up, a seeded scene, and results committed as a baseline file.
  - Software rasterisers cannot give meaningful timing.
- **The split plan's "no CI runner can open a window" is too strong. Revise it.**
  - `windows-latest` has no usable GL driver. One developer reported a WGL error there and had to install Mesa.
  - An `ubuntu-latest` job with Xvfb and Mesa llvmpipe, which is OpenGL 4.5 conformant, can very likely run GL 3.3 core. It needs a spike to confirm on the current runner image.
  - Headless Chromium on SwiftShader covers WebGL2. A KVM-backed Android emulator covers GLES, with caveats below.
  - So the renderer *can* be verified automatically, as pixel-probe and tolerant-golden tests. It cannot be *timed*.
- **Sample apps double as tests.**
  - Each sample app runs one shared, seeded "reference scene". A launch parameter selects capture mode (render N frames, dump a PNG, exit) or benchmark mode (warm up, measure, dump JSON).
  - Goldens are generated on the same software rasteriser that CI uses, never on a developer GPU. Pin and record the rasteriser version, as Filament does.
- **Provisional per-platform targets** (all weak-evidence; ratify after the first sample-app measurement):

| Platform | Target |
|---|---|
| Desktop | 10k sprites, 1080p, p99 frame ≤ 16.7 ms, CPU frame cost ≤ ~3 ms |
| Web | 10k at 60 fps on a mid-range laptop integrated GPU, Chrome, WebGL2. Gate at 5k until measured. |
| Android | 5k at 60 fps on a mid-range GLES 3.0 phone (Mali-G52 / Adreno 610 class), ≤ 4× overdraw |

## 3. Findings

### 3.1 Batching techniques (GL 3.3, GLES 3.0, WebGL2)

**F1. Vertex-expanded quad batching is the dominant design. Tier: common practice.**
- libGDX [`SpriteBatch`](https://raw.githubusercontent.com/libgdx/libgdx/master/gdx/src/com/badlogic/gdx/graphics/g2d/SpriteBatch.java):
  - Default capacity is 1000 sprites; the cap is 8191 because indices are 16-bit (32767 / 4).
  - A sprite is 20 floats: 4 vertices of x, y, packed colour, u, v.
  - The index buffer is built once.
  - It flushes on texture change, on a full buffer, and on blend, shader or matrix change.
  - A flush calls `mesh.setVertices`, then draws.
- MonoGame [`SpriteBatcher`](https://raw.githubusercontent.com/MonoGame/MonoGame/develop/MonoGame.Framework/Graphics/SpriteBatcher.cs):
  - `MaxBatchSize = short.MaxValue / 6` (5461 sprites per pass); the initial batch is 256.
  - `SpriteSortMode.Texture`, `FrontToBack` and `BackToFront` sort; other modes keep submission order.
- KorGE [`BatchBuilder2D`](https://github.com/korlibs/korge/blob/main/korge-core/src/commonMain/kotlin/korlibs/korge/render/BatchBuilder2D.kt):
  - Default 8192 quads, maximum 16383 (16-bit index limit).
  - 24 bytes per vertex.
  - Up to 4 textures per batch via a per-vertex texture index (1 on 32-bit ARM Linux).
  - Flushes on a full buffer, on state change (stencil, colour mask, scissor), and at end of frame.
- mattdesl's [LWJGL sprite-batching notes](https://github.com/mattdesl/lwjgl-basics/wiki/Sprite-Batching):
  - Flush on `end()`/`flush()`, on texture change, and on capacity.
  - "Many textures lead to many draw calls", so atlas.
  - Pack the colour into one float to cut per-vertex size.

**F2. Bevy uses instancing, not vertex expansion. Tier: common practice, from the source.**
- Sprites are sorted by z into a transparent phase, then grouped by consecutive image. A non-sprite item breaks the batch, which preserves order.
- All sprites share one instance buffer. `SpriteInstance` holds transform, colour and UV, with an 80-byte stride and instance step mode. One 6-index quad is shared by every sprite.
- One `draw_indexed(0..6, 0, range)` is issued per batch.
- Source: [bevy_sprite_render](https://github.com/bevyengine/bevy/blob/main/crates/bevy_sprite_render/src/render/mod.rs).

**F3. Instancing versus vertex expansion is a wash for static data. Tier: one source's measurement, desktop, and old.**
- Cesium measured the two as about equal when static. Instancing won when data was animated, because the per-frame upload was about a quarter of the size.
- Source: [Cesium billboard instancing](https://cesium.com/blog/2015/10/28/billboard-instancing/).
- I found no 2D-sprite benchmark of instancing against vertex expansion on mobile GLES or WebGL2. This is a gap.
- Instancing is core in GL 3.3, ES 3.0 and WebGL2 (general knowledge, not separately sourced).

**F4. Buffer streaming: orphan or ring-buffer; persistent mapping is out. Tier: established consensus (Khronos wiki, via a search snippet only).**
- Orphaning (`glBufferData(NULL)`, `glMapBufferRange` with invalidate, or `glInvalidateBufferData`) is the preferred way to avoid stalls.
- Persistent mapping needs GL 4.4 or `ARB_buffer_storage`, with 2 to 3 buffers' worth of storage.
- Inference: GL 3.3, ES 3.0 and WebGL2 do not have it, so a cross-backend design cannot depend on it.
- Source: [Buffer Object Streaming](https://www.khronos.org/opengl/wiki/Buffer_Object_Streaming). My direct fetch of this page failed.

**F5. Draw order forbids sorting blended sprites by state. Tier: common practice.**
- Bevy and PixiJS batch only *consecutive* same-state items. Sorting is by depth or layer only.
- That makes atlas discipline, not state sorting, the lever for long batches.
- PixiJS also notes that devices with fewer than 16 texture units benefit more from a single atlas ([html5gamedevs](https://html5gamedevs.com/topic/45900-performance-issue-when-moving-from-485-to-524)).

**F6. Cost scales with state changes, not sprite count. Tier: common practice.**
- A scissor/clip change is a batch break in KorGE and libGDX. G3 and G5 add exactly this clip. A UI scene with many nested clipped groups can multiply draw calls.
- The gate scene set must include a clip-heavy UI scene and a text-heavy scene, not only a sprite swarm.

**F7. Retained or incremental builds are the next step beyond rebuilding every frame. Tier: one source.**
- PixiJS v8 reuses render instructions when the scene graph is unchanged, uploads all scene data at once, and skips the upload if nothing moved.
- Reported: 100k sprites static, 0.12 ms CPU against ~21 ms in v7; all moving, ~15 ms CPU.
- The post does not state the hardware. Source: [Pixi v8 beta](https://pixijs.com/blog/pixi-v8-beta).
- A later `ParticleContainer` post claims 200k sprites at 60 fps and 1M particles on an M3 MacBook Pro ([source](https://pixijs.com/blog/particlecontainer-v8)).
- Not needed for 10k, but the batcher's API should not preclude it.

### 3.2 Platform specifics

**F8. JVM (LWJGL). Tier: partly sourced, partly general knowledge.**
- Allocate one direct buffer once (`MemoryUtil.memAlloc` or `ByteBuffer.allocateDirect`, native order) and write vertices into it. Allocate nothing per sprite.
- The monolith allocates a `VisibleObjectSnapshot` per actor per frame upstream of the renderer. That is a likely GC source and the gate should measure it.
- Measure with `ThreadMXBean.getThreadAllocatedBytes`. That counter is deterministic and needs no GPU.

**F9. Web (Kotlin/Wasm or JS). Tier: one source's opinion plus anecdote.**
- WebGL's call boundary passes only primitives, so it is cheap. A Khronos list reply says WebGPU's object-heavy API will cost more per call ([Khronos list, 2020](https://www.khronos.org/webgl/public-mailing-list/public_webgl/2005/msg00018.php)).
- An anecdote puts a Wasm-to-JS call at about 5 ns, so a few thousand calls per frame are negligible ([ziggit](https://ziggit.dev/t/using-std-io-writer-for-efficient-canvas-batching-from-webassembly/11977)).
- A JetBrains developer says Kotlin/Wasm arrays cannot be passed to JS directly and must be copied or wrapped ([Kotlin Slack](https://slack-chats.kotlinlang.org/t/10259826/i-am-interested-in-using-kotlin-wasm-to-make-a-3d-game-i-man)).
- Design implication: build the vertex data in Kotlin and do one bulk copy per flush. Do not write vertices through an external `Float32Array` one element at a time.
- Whether `FloatArray` is zero-copy in Kotlin/JS (not Wasm) is general knowledge I did not verify. It bears on the Wasm-or-JS choice, which is someone else's question.

**F10. Android. Tier: official docs for pacing; the rest is general knowledge.**
- Use Swappy ([frame pacing](https://developer.android.com/games/sdk/frame-pacing)) for stable pacing.
- Keep GL calls per frame low; each is a JNI hop (general knowledge, not sourced).
- Avoid per-frame allocation under ART (not sourced).

### 3.3 Measuring

**F11. Percentiles, not average fps. Tier: established on Android (official).**
- Macrobenchmark's [`FrameTimingMetric`](https://developer.android.com/topic/performance/benchmarking/macrobenchmark-metrics) reports P50, P90, P95 and P99.
- It reports `frameOverrunMs` (API 31+), where positive values are dropped frames, and `frameCount`.
- It advises watching the P95 and P99 bounds.
- It targets UI frames; whether it works for a `GLSurfaceView`-style GL surface is unverified.

**F12. GPU timer queries are unreliable off-desktop. Tier: mailing-list and forum evidence, dated.**
- `EXT_disjoint_timer_query(_webgl2)` was disabled in Chrome 65 and Firefox 59 in 2018 over Spectre and GLitch concerns ([Khronos list](https://www.khronos.org/webgl/public-mailing-list/public_webgl/1803/msg00017.php)).
- A 2021 post reports Chrome re-enabling it with site isolation while Firefox did not ([Babylon.js forum](https://forum.babylonjs.com/t/diving-deeper-engineinstrumentation-info-not-up-to-date/25223)). Current status is unverified.
- Desktop GL 3.3 has `GL_TIME_ELAPSED` natively (general knowledge). On ES 3.0 it is only an extension.
- So GPU time is a desktop-only metric. Android and web gate on wall-clock frame-time percentiles plus CPU time.

**F13. VSync hides headroom. Tier: my inference; the Love2D bunnymark convention supports it.**
- The classic bunnymark finds "the maximum sprite count keeping 60 fps" ([love2d-bunnymark](https://github.com/rishavs/love2d-bunnymark)).
- A vsynced 60 fps pass/fail cannot show a 2x regression until it falls off the cliff.
- Add an uncapped mode (swap interval 0) and record ms per frame for headroom. Android cannot easily disable vsync, so use CPU time there.

**F14. Structural counters are the CI-portable performance signal. Tier: common practice.**
- libGDX ships `GLProfiler`, which counts draw calls and texture bindings ([overview](https://dev.to/skaterdad/how-to-use-the-libgdx-glprofiler-4dke)).
- `SpriteBatch` itself tracks `renderCalls` and `maxSpritesInBatch`.
- Equivalent counters in the renderer make "10k sprites on one atlas ≤ N draw calls" a deterministic test.

### 3.4 Verification without a GPU

**F15. Linux: Xvfb plus Mesa llvmpipe. Tier: common practice.**
- llvmpipe is [OpenGL 4.5 conformant](https://www.khronos.org/news/permalink/mesas-llvmpipe-is-opengl-4.5-conformant), so a GL 3.3 core context is feasible.
- A GLFW-forum reply recommends `GALLIUM_DRIVER=llvmpipe`, or an OSMesa context via `GLFW_OSMESA_CONTEXT_API`, for GPU-less runners ([GLFW forum](https://discourse.glfw.org/t/running-tests-in-github-actions/2143)). The same thread reports window-creating tests crashing the host process on a runner with no display or GPU.
- ModernGL's documented CI runs `ubuntu-latest` with `xvfb` ([ModernGL CI](https://moderngl.readthedocs.io/en/5.8.2/install/using-moderngl-in-ci.html)). It installs no Mesa package, which implies the image's Mesa suffices.
- I found no confirmed LWJGL plus GLFW test on a current runner. The spike must print `GL_RENDERER` and `GL_VERSION`.
- GLFW 3.4 has an always-built Null platform ([3.4 news](https://glfw.org/docs/3.4/news.html)). Whether it plus OSMesa is a usable headless route under LWJGL 3.3.4 is unverified.
- Set `LP_NUM_THREADS` to bound oversubscription. A Wine CI change reports that `0` forces a single thread and cuts memory greatly (from search results).

**F16. Windows runner: no GL. Tier: one report, plausible.**
- A developer found `windows-latest` has no GPU. WGL failed with "driver doesn't support OpenGL", and they fixed it with Mesa from MSYS2 plus `GLCONTEXT_WIN_LIBGL` ([ModernGL CI](https://moderngl.readthedocs.io/en/5.8.2/install/using-moderngl-in-ci.html)).
- Prebuilt Windows llvmpipe and SwiftShader binaries exist ([jakoch/rasterizers](https://gitblind.noratr.app/jakoch/rasterizers)).
- The GDI generic renderer is GL 1.1 and will not satisfy a 3.3 context.
- No GitHub source confirms the image's default driver. Printing `GL_VERSION` in a step settles it.

**F17. Web: headless Chromium plus SwiftShader. Tier: official docs plus a real project.**
- Flags: `--use-gl=angle --use-angle=swiftshader-webgl --enable-unsafe-swiftshader`. Automatic fallback is deprecated and the flag is an explicit opt-in ([Chromium docs](https://chromium.googlesource.com/chromium/src/+/da170c2267b2201954efe1c8fa5d2d89ab09f7bf/docs/gpu/swiftshader.md)).
- Do not pass `--disable-gpu`, which can disable WebGL entirely.
- Without the right setup, WebGL can quietly degrade and still produce a plausible screenshot. Assert the renderer string, not just that an image exists ([microlink](https://microlink.io/blog/webgl-without-a-gpu)).
- Filament builds `gltf_viewer` to Wasm and runs it in headless Chromium on SwiftShader, with the browser version pinned via Playwright ([renderdiff](https://google.github.io/filament/dup/test_ci_renderdiff.html)).
- WebGPU over SwiftShader Vulkan is "slow but adequate" for a non-blank check, from one blog.

**F18. Android emulator in CI. Tier: official changelog plus README; GPU-mode status needs confirmation.**
- GitHub announced hardware-accelerated Android virtualisation (KVM) on 2-vCPU hosted Linux runners on 2024-04-02, after a KVM-permission step ([changelog](https://github.blog/changelog/2024-04-02-github-actions-hardware-accelerated-android-virtualization-now-available/)).
- The [android-emulator-runner](https://github.com/ReactiveCircus/android-emulator-runner) README says it is free for public repositories. The sources disagree on runner sizes, so confirm eligibility.
- Translated Android docs say the `swiftshader_indirect` GPU mode was retired in emulator 36.4.9 in favour of `swiftshader` and `swangle` ([emulator acceleration](https://developer.android.com/studio/run/emulator-acceleration)). My fetch of the English page failed, so this is unconfirmed.
- Whether software GPU modes expose GLES 3.0 is unverified.
- The emulator is slow, so run it nightly or pre-release, not per commit.

**F19. Golden images need tolerance and a pinned rasteriser. Tier: established by Chromium, Filament and the Vulkan CI guidance.**
- Chromium's Skia Gold uses exact matching by default but allows multiple approved baselines per test. The reason: "visually indistinguishable but differing in a handful of pixels by a small RGB value" ([Chromium docs](https://chromium.googlesource.com/chromium/src/+show/800fb94de33f2f4ae52f7b56dbf1feae8a98fa0c/docs/gpu/gpu_pixel_testing_with_gold.md)).
- Filament keeps separate goldens per platform and backend, generated on the same runner kind that presubmit compares against. Different hosts produce different pixels from compiler, Mesa build or libm differences. Browser build, SwiftShader and render size must be fixed or the goldens regenerated ([renderdiff](https://google.github.io/filament/dup/test_ci_renderdiff.html)).
- The Vulkan CI guide warns software rasterisers use different floating-point rounding, so exact comparison fails ([Vulkan CI](https://docs.vulkan.org/tutorial/latest/ML_Inference/Desktop_Applications/04_ci_render_validation.html)).
- three.js uses Puppeteer screenshots of its examples compared to stored references ([forum](https://discourse.threejs.org/t/headless-rendering-functional-how-to-take-screenshot/31700)). I did not retrieve its tolerance.

## 4. How comparable projects do it

| Project | Batching | Verification (as found) |
|---|---|---|
| libGDX `SpriteBatch` | 1000 default and 8191 max sprites, 5 floats per vertex, static `short[]` indices, flush on texture, blend or full. `GLProfiler` counts calls. | Not researched |
| MonoGame `SpriteBatcher` | 256 initial, 5461 max per pass, sort modes Texture / FrontToBack / BackToFront / Deferred (no sort) | Not researched |
| KorGE `BatchBuilder2D` | 8192 default quads, 16383 max, up to 4 textures per batch, flush on scissor, stencil or colour-mask change | Not researched |
| PixiJS v8 | Retained instruction set, consecutive same-state batches, upload skipped when static, 100k-sprite figures above | Not researched |
| Bevy | Z-sorted phase, one instanced draw per same-image run, 80-byte instance | Not established |
| Filament | n/a | `renderdiff`: Mesa software rasterisers on desktop, headless Chromium on SwiftShader for web, goldens in a separate repo |
| Chromium | n/a | Skia Gold, exact with multiple baselines, retries capped at 3 |
| three.js | n/a | Puppeteer example screenshots against stored references |
| wgpu, Godot, Skia, Compose (Roborazzi, Paparazzi) | n/a | Not established; see Gaps |

## 5. Pitfalls and regrets

- **Counting sprites without a scene profile.** Overdraw and sprite size decide fill-rate cost.
  - 10k sprites of 64×64 px is about 41 Mpx per frame, roughly 20 screens of overdraw at 1080p (my arithmetic). That is a fill-rate test, not a batching test.
  - Fix the profile in the gate, for example 32 to 64 px sprites at 1080p.
- **16-bit index caps** (8191, 5461 and 16383 quads in the three batchers above). 10k sprites exceeds libGDX's and MonoGame's caps. Make capacity a parameter, and count the resulting flushes in the gate.
- **State changes silently multiplying flushes:** per-sprite blend or shader changes, scissor changes, and interleaving a separate text pipeline. The monolith has separate actor and `UiRenderer` pipelines; G5 merges them.
- **Mean fps and vsync-bound pass/fail** hide stutter and headroom. Report percentiles and uncapped ms.
- **llvmpipe timings mean nothing.** They measure a CPU rasteriser on a noisy shared runner. Never gate timing on it.
- **Exact-match goldens** break on a Mesa or browser update. Pin the versions, keep the tolerance honest, prefer pixel-aligned content, and regenerate with a script. Filament's experience.
- **Silent WebGL degradation** in headless Chromium: assert the renderer string.
- **`windows-latest` job crashes.** GLFW tests that open a window can kill the test host. Keep GL-opening tests out of the Windows job and tag them.
- **LWJGL natives are per-OS** (general knowledge). The Linux test runtime needs the Linux natives classifier, which today's compile-only Linux job does not.
- **Headless Godot under Xvfb is unusably slow on CPU-only CI** ([gdUnit4 thread](https://github.com/godot-gdunit-labs/gdUnit4Net/discussions/350)). Keep software-GL scenes small: tiny viewport, few frames.
- **Reading pixels from a WebGL canvas** after compositing returns blank unless read in the same frame or `preserveDrawingBuffer` is set (general knowledge, not sourced).

## 6. Recommendation for this repo

### Batching (G5, then Android and web)

- **Build the batch in common, platform-independent code. Only the GL (or other GPU API) calls are backend-specific.**
  - A pure `BatchBuilder` turns the ordered draw list into vertex runs. It has capacity, flush reasons, a texture slot and a clip rectangle, and writes into a platform vertex sink.
  - That makes the draw-call count, flush reasons, vertex bytes and allocation behaviour testable at Levels 2 and 4a with exact assertions and no GPU.
  - It is also where the shared, expensive-to-change decisions live. How this sits against the backend contract is question 5's concern.
- **Vertex-expanded quads, with the vertex format reserving z.**
  - Reasons: lines, polygons, 9-slice and gradients need arbitrary vertices anyway, so one general stream serves every shape. The clip and draw-order rules are simplest there.
  - Instancing remains a possible later optimisation *inside* a backend, decided by the gate. Bevy proves it works.
  - It must never leak into the public contract.
- **Draw order is the contract.**
  - Sort by layer and z only. Batch consecutive same-state runs.
  - Break on texture, blend, clip rectangle, shader and capacity.
  - Make capacity a constructor parameter, defaulting to about 8k quads.
- **Atlases first.** The asset pipeline's atlas packing is the largest batching lever. Reserve a per-vertex texture-slot attribute so multi-texture batching (KorGE-style, 4 slots) can be added later without a format change.
- **Uploads.**
  - One direct buffer allocated once on the JVM, and one `ByteArray` or `FloatArray` mirror on Wasm and JS with one bulk copy per flush.
  - Use a ring of 2 to 3 GPU buffers or orphan per flush. Never depend on persistent mapping.
  - Cache every uniform location at link time. The old five-lookup-per-actor pattern disappears because per-sprite data moves into vertices.
  - Per-frame uniforms only: the view-projection matrix and similar.
- **CPU culling:** a cheap AABB-versus-camera test in the draw-list builder. Do not build a spatial index in the renderer.

### Targets (provisional, ratify after first measurement)

The profile is: single atlas, alpha-blended, rotated and moving sprites, 32 to 64 px, ≤ 4× overdraw at device resolution.

| Platform | Target | Notes |
|---|---|---|
| Desktop | 10k at 60 fps, 1080p, p99 ≤ 16.7 ms, CPU render cost ≤ ~3 ms, draw calls ≤ ~10 | Headroom check: uncapped ≥ 3× for the same scene. |
| Web | 10k at 60 fps on a mid-range laptop iGPU, Chrome, WebGL2 | Gate at 5k until measured. |
| Android | 5k at 60 fps on a mid-range GLES 3.0 phone, ≤ 4× overdraw | Gate on p95 ≤ 16.7 ms. |

Evidence for the web and Android figures is thin: old Pixi bunnymark anecdotes ([iPad 4 held 15k bunnies](https://www.html5gamedevs.com/topic/5653-bunnymark-perf-comparisons/)) and the Pixi v8 figures. These are my judgment, to be replaced by measurement.

### The 4c gate

1. **CI-enforced structural counters** (every commit, any runner).
   - A deterministic scene set: sprite swarm, clip-heavy UI, text-heavy.
   - Assert a draw-call ceiling, flushes by reason, bytes uploaded, and steady-state allocated bytes per frame ≈ 0.
2. **Reference-hardware timing** (release gate, manual or a self-hosted GPU runner).
   - Warm up (discard the first N frames for shader compile, JIT and ART profile). Then measure a fixed frame count of a seeded scene stepped on a fixed timestep, independent of render time.
   - Record p50, p95, p99 and max of CPU frame time and wall frame time. On desktop add GPU time via `GL_TIME_ELAPSED`. Run an uncapped pass for headroom.
   - Commit the JSON result per release and fail on regression against it.
   - Platforms: Windows reference machine, one reference Android phone, one browser and laptop.
   - The hardware must be named; see Gaps.

### Verification strategy

This revises the "no CI runner can open a window" line in the split plan's testing section and Risks. Replace it with: "No CI runner has a GPU; the renderer is verified on software rasterisers, and timing is verified on reference hardware."

| Level | Technique | Where |
|---|---|---|
| 2 | `BatchBuilder`, draw-list translation, recording renderer | any |
| 3 | Real GL on llvmpipe: shader compile and link, FBO readback pixel probes (clip, blend, order, texture formats) | Linux job (Xvfb) |
| 4a | Scene to draw-call-count and vertex-bytes mapping | any |
| 4b | Sample app renders N frames and dumps a PNG compared to a tolerant golden | Linux Xvfb; headless Chromium plus SwiftShader (web); Android emulator, nightly |
| 4c | Counters in CI; timing on reference hardware | CI, plus release gate |
| 5 | Game client, reference-device runs, human review | by hand |

- Windows job: keep it as the full build and non-GL tests. Tag GL-opening tests so they do not run there. Do not install Mesa on Windows unless a need appears.
- Prefer a few analytic probes ("pixel (x,y) equals C") over many goldens. Use pixel-aligned, nearest-filtered content so output is near-exact across rasterisers.
- Compare with a per-channel tolerance plus a maximum differing-pixel fraction.
- Generate goldens only on the CI rasteriser. Hardware correctness is Level 5.
- The draft architecture's `graphics-test` module (recording renderer, scripted input, manual clock) tests the frame pipeline *above* the renderer. It does not verify the GL renderer. Software GL does that.

### Sample apps as tests

- One shared reference-scene definition lives in test fixtures, used by golden and benchmark runs on every platform.
- Each sample app takes launch parameters: scene, sprite count, frames, and mode (capture or benchmark).
  - Web: query string, then Playwright waits for a done flag.
  - Android: intent extras.
- The apps write a PNG or JSON and exit.
- Each platform's backend is proven by its sample app on the software rasteriser per commit (or nightly for Android), and on reference hardware at release.

### Rejected

- Instancing as the base design: no measured advantage for static data (F3), and it needs a general stream anyway.
- GPU timing as a CI gate: not available.
- Persistent mapped buffers: not in GL 3.3, ES 3.0 or WebGL2.

## 7. Gaps

- **Reference hardware is unspecified.** "Mid-range desktop GPU" needs a named card, phone and laptop. Only the user can supply them.
- **No Android or web sprite-count evidence beyond old anecdotes.** The provisional targets are my judgment.
- **Instancing versus vertex expansion on mobile and WebGL2** has no benchmark I could find.
- **No confirmed LWJGL plus GLFW context on a current `ubuntu-latest` image under Xvfb and llvmpipe.** Needs a spike that prints `GL_RENDERER` and `GL_VERSION`. Also unconfirmed: that the image ships Mesa llvmpipe, and what Linux natives the test runtime needs.
- **Windows runner's default GL driver is not documented by GitHub.** Same spike.
- **Android emulator software GPU mode.** The English docs fetch failed; the `swiftshader_indirect` retirement comes from translated pages. GLES 3.0 under `swiftshader` or `swangle` is unverified. The 2-vCPU KVM claim and free-for-public-repos status need confirmation.
- **Not established (searches thin or fetch failed):** how Bevy, Godot, wgpu and Skia themselves run rendering tests; one search snippet claimed wgpu runs lavapipe, WARP and llvmpipe in CI, but its README did not show a table. Roborazzi and Paparazzi: I believe they render through Android layoutlib or Robolectric and cannot exercise a GLES context. Unsourced, so treat them as not applicable until checked.
- **The Khronos streaming wiki fetch failed.** F4 rests on a search snippet of that page.
- **Whether `FrameTimingMetric` works for a GL surface**, and the current status of GPU timer queries in browsers, are unverified.
- **Kotlin/Wasm typed-array upload cost** has no measurement; the copy-per-flush design follows from one developer's statement.
- **Firecrawl was not used** (credits low), and the session hit its limit mid-research, so a few primary fetches (Android docs, GitHub changelog, Khronos wiki) were not completed.
