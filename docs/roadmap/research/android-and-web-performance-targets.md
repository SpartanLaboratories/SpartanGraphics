# Android and web performance targets (D13)

## 1. Question

What sprite counts at 60 fps does a well-batched 2D renderer sustain (a) on a mid-range Android phone with GLES 3.0 and (b) in a browser with WebGL2 on a mid-range laptop integrated GPU, as of 2025–2026, and which reference-device classes should the Level 4c gates name?

**Decision it informs:** D13, the Android and web performance targets and the reference-device profile for the gate. This builds on `render-performance-and-verification.md` (§2, §3.3, §5, §6 "Targets"); I do not repeat that file.

## 2. Bottom line

- **No published, named-device, mid-range Android or laptop-iGPU sprite benchmark from 2025–2026 exists that I could find.** I searched GitHub issues, engine forums, engine docs and vendor docs. What exists is old (2013–2018 bunnymark forum posts), unscientific (one 2019 MacBook table, one 2026 blog with 500 sprites), or unlabelled (Pixi v8 gives no hardware). So the Android and web figures can only be set by fill-rate and CPU arithmetic plus those anchors, and must be ratified by the first sample-app measurement. This confirms the provisional file's own caveat; it does not replace it with measured data.
- **Sprite count is the wrong unit; covered pixels per frame is the binding quantity on phones.** The provisional profile ("32–64 px sprites, at most 4x overdraw at device resolution, 10k sprites") is internally inconsistent: 10k sprites of 32 px already cover 10.2 Mpx, which is 4.9 screens at 1080p, and 64 px covers 41 Mpx (about 20 screens). Express every gate as sprites plus a covered-area ceiling in Mpx (my arithmetic, §6).
- **Android: 10k sprites looks feasible on paper, 5k is the only figure I would hard-gate before measuring.** A Mali-G52 MC2 has about 3.4 Gpix/s peak (my arithmetic from 2 px/clock/core, 2 cores, 850 MHz), and 10k sprites of 32 px at 60 fps needs about 0.61 Gpix/s, 18% of peak. CPU cost should be a few ms. But thermals, driver overhead and real efficiency are unmeasured. Recommend: 5k hard gate, 10k target promoted to a hard gate only if the first measurement leaves at least 30% headroom. Confidence: medium-low.
- **Web: raise the gate from "5k until measured" to 10k on Chrome with an Iris Xe-class laptop.** Fill rate is a non-issue (Iris Xe about 31 Gpix/s peak per a spec database, so 10k x 32 px is about 2%). The risk is CPU (JS or Wasm) and browser overhead. Anchors: Pixi v8 is about 0.15 microseconds per moving sprite (hardware unstated), Phaser 4 claims "tens of thousands of sprites comfortably", an unscientific 2019 table has Pixi at 60 fps with 8k boxes in Chrome. Confidence: medium. Firefox and Safari are report-only.
- **Name classes, not just GPUs, and pin the SKU.** The best-selling Android phones of 2025 sit in the Mali-G52 / G57 / G68 entry-to-mid tier, and one model (Galaxy A16 5G) ships with different SoCs by region. Gate device: Mali-G57 MC2 / G68 MP2 / Adreno 619 class at 1080p. Floor device: Mali-G52 MC2 at 720p (Galaxy A06). Web gate machine: Iris Xe laptop, Chrome stable; floor: UHD 620.
- **Two corrections to the earlier file.** (1) For the web backend do not orphan buffers (`bufferData(size)`); a Chrome engineer advises against it, and a plain `bufferSubData` into a sized buffer is the web idiom. (2) Android frame timing for a GL surface should come from `dumpsys SurfaceFlinger --timestats` (official, present-to-present histogram) plus in-app CPU time, because Perfetto FrameTimeline does not support SurfaceViews and the Macrobenchmark `FrameTimingMetric` doc is silent on GL.

## 3. Findings

### 3.1 What the evidence base actually is

**F1. There is no controlled mid-range-Android sprite benchmark in public. Tier: absence, from several independent searches.**
- Searches of GitHub issues (bunnymark, bevymark, libgdx, godot, raylib, pixijs, phaser), the Defold, Godot, libGDX and Pixi forums, and general engine-comparison articles returned no 2024–2026 result naming a Mali-G52 / Adreno 610-class device with a sprite count at 60 fps.
- The single 2026 hit is a blog test: 500 animated sprites on a Snapdragon 720G, Cocos Creator 3.8 at 58 fps, Godot 4.4 at 50 fps, Unity 6 at 42 fps; the author says "These aren't controlled lab benchmarks; they're from an actual profiling session on a real project device." ([engineranked, 2026-03-27](https://engineranked.com/article/cocos-creator-review-2026/)). Tier: one source's opinion. It shows what scene-graph engines with real game logic do, not what a flat batcher does. It is a reminder that CPU overhead outside the renderer can dominate.

**F2. Old bunnymark anecdotes put even 2015 budget phones at 15k+ small sprites. Tier: anecdote, dated.**
- Moto G3 (Android 6.0.1) 17,800 bunnies and a Nexus 10 (Android 5.1.1) about 17,000, both with the threshold at 30 fps, in Pixi's bunnymark, April 2017 ([html5gamedevs thread](https://html5gamedevs.com/topic/29786-game-is-very-slow-in-nexus-10-device)). Pixi version and browser not stated.
- A bunnymark bunny is about 26 x 37 px ([neotolis-engine bunnymark README](https://github.com/d954mas/neotolis-engine), `examples/bunnymark/README.md`, "Test Conditions"), so 17.8k bunnies is about 17 Mpx per frame at 30 fps, about 0.5 Gpix/s (my arithmetic). That is a measured sustained throughput on a 2015 budget GPU, and a Mali-G52 MC2 has a much higher peak. It is weak support that 5k–10k small sprites at 60 fps is well inside a 2020-era mid-range GPU, not proof.
- Earlier anecdotes (iPad 4 at 15k Pixi bunnies, already in the prior file) are 2014–2017 and add nothing.

**F3. Browser-side anchors: Pixi, Phaser. Tier: vendor claims plus one unscientific table.**
- Pixi v8 launch bunnymark: 100k moving sprites, v7 about 50 ms CPU, v8 about 15 ms CPU; the page states no device, browser or resolution ([pixijs.com/blog/pixi-v8-launches](https://pixijs.com/blog/pixi-v8-launches); `v8.0.0` published 2024-03-05). That is about 0.5 and 0.15 microseconds per moving sprite, so about 5 ms and 1.5 ms for 10k (my arithmetic).
- The same table's "GPU" column (about 2 ms for 100k sprites) cannot be real fill-rate execution time: 100k bunnies of about 960 px is about 96 Mpx, which would be about 48 Gpix/s. Treat it as submission time, not GPU cost (my inference).
- Pixi ParticleContainer post: 200,000 sprites in a Container and 1,000,000 particles in a ParticleContainer at 60 fps on a "Macbook Pro M3"; no mobile statement ([pixijs.com/blog/particlecontainer-v8](https://pixijs.com/blog/particlecontainer-v8)).
- Phaser 4 (`v4.2.1`, 2026-07-09): "standard Phaser rendering handles tens of thousands of sprites comfortably", and "Quads now use index buffers, reducing vertex upload costs by a third" ([README](https://cdn.jsdelivr.net/npm/phaser@4.2.1/README.md)). The second quote independently supports 4 vertices plus a static index buffer over 6 vertices per quad.
- An unscientific 2019 MacBook Pro table has Pixi at 60 / 48 / 24 fps in Chrome / Firefox / Safari with 8k boxes ([canvas-engines-comparison](https://github.com/slaylines/canvas-engines-comparison)); the repo itself says "(highly unscientific)", gives no versions or chip. Useful only as evidence that browsers differ widely on identical code.

**F4. Fill rate and display resolution, not sprite count, are what the platform vendors warn about on phones. Tier: official docs, qualitative.**
- Android: "On less performant GPUs, available fill-rate—the speed at which the GPU can fill the frame buffer—can be low", and "Although low-end devices continue to improve in GPU performance, their displays remain at relatively low resolutions" ([Android Developers, GPU overdraw page](https://developer.android.com/topic/performance/rendering/overdraw)). No numbers.
- Arm gives the budgeting method: `fragCycleBudget = (No of GPU fragment cores) * (gpuFrequency in Hz) / ( FPS * numPixels)` ([Arm blog, 2017-07-03](https://developer.arm.com/community/arm-community-blogs/b/mobile-graphics-and-gaming-blog/posts/gpu-processing-budget-approach-to-game-development)). I use it in §6 (arithmetic is mine).

**F5. GPU class facts I could source. Tier: press and spec database, secondary.**
- Mali-G52: "clocked up to 850 MHz delivering 6.5 Gpix/s pixel/texturing throughput" ([CNX Software, 2018-03-06](https://www.cnx-software.com/2018/03/06/arm-introduces-mali-g52-mali-g31-gpus-mali-d51-display-processor-and-mali-v52-video-processor-for-mainstream-devices/)). The core count behind 6.5 is not stated, so I derive the MC2 figure from the per-core rate (2 px/clock/core, from Arm tables as relayed by search results, not quoted by me): 2 x 2 x 0.85 GHz = 3.4 Gpix/s. This is the largest source of uncertainty in the Android arithmetic.
- Iris Xe 96EU: 31.20 GPixel/s, boost 1300 MHz, per a GPU database ([askgeek](https://askgeek.io/fr/gpus/vs/Intel_Iris-Xe-Graphics-96EU-Raptor-Lake-vs-Intel_UHD-Graphics-Raptor-Lake-64-EU)). UHD 620 shows "Pipelines - 24" and boost 1150 MHz but no pixel rate ([askgeek](https://askgeek.io/en/gpus/Intel/UHD-Graphics-620)). I could not source Adreno 610 or AMD 680M/760M fill rates (TechPowerUp and Notebookcheck return 403 to the fetch tool; Wikipedia's Adreno table lost its headers).

### 3.2 What "mid-range" means in 2026

**F6. The volume Android phones are entry-to-mid class on Mali-G52 / G57 / G68. Tier: sales ranking plus device spec pages.**
- Counterpoint's 2025 ranking, as relayed: "The Galaxy A16 5G was Samsung's best-selling phone of the year, followed by the Galaxy A06 4G", and the S25 Ultra at ninth was "the only high-end Android phone in the list" ([SamMobile](https://www.sammobile.com/news/galaxy-s25-ultra-only-high-end-android-phone-worlds-top-ten-selling-phones/)). The Counterpoint page itself did not render for me.
- Galaxy A06: Helio G85, GPU "Mali-G52 MC2", display "720 x 1600 pixels" ([GSMArena](https://m.gsmarena.com/samsung_galaxy_a06-ampp-13265.php)).
- Galaxy A16 5G review unit: "Our review unit is running the Exynos 1330", Mali-G68 MP2, 1080 x 2340 ([GSMArena review](https://m.gsmarena.com/samsung_galaxy_a16_5g-review-2758p4.php)). One search summary says other regions ship a Dimensity 6300 (Mali-G57 MC2); I could not confirm that by quote.
- Consequences: (a) the realistic floor is Mali-G52 MC2 at 720p, not the flagship-adjacent parts; (b) the same marketing name maps to different GPUs, so a gate must name the SoC and resolution, not the phone line.
- Android's Media Performance Class is not usable as a GPU-class definition: the CDD pages I could read give media requirements only, and I did not find its §7.11 text. Not used.

**F7. On the web, Intel integrated GPUs are the largest single group of reporting GPUs. Tier: one survey, undated.**
- Web3DSurvey's WebGL2 vendor table lists "Google Inc. (Intel)" at 30%, the largest row ([web3dsurvey.com/webgl2](https://web3dsurvey.com/webgl2)). The page states neither sample size nor date, and mixes desktop and mobile. It supports naming an Intel iGPU as the web reference class, nothing more.

### 3.3 Per-sprite CPU cost: JS, Wasm, ART

**F8. Kotlin/JS: a `FloatArray` already is a `Float32Array`. Tier: official docs.**
- `FloatArray` maps to `Float32Array` and `IntArray` to `Int32Array`; `Long` is `BigInt` only under the experimental `-Xes-long-as-bigint` / `es2020` setting ([Kotlin docs, "Use JavaScript code in Kotlin" type table](https://kotlinlang.org/docs/js-to-kotlin-interop.html)). So the vertex array can be filled in Kotlin and handed to `bufferSubData` without a copy on Kotlin/JS. Keep `Long` out of the per-sprite path (my advice; the doc says only how it is represented).

**F9. Kotlin/Wasm: arrays do not cross the JS boundary directly. Tier: official docs.**
- "Arrays: Not supported in interop directly yet. You can use the new `JsArray` type instead", plus adapters such as `toInt32Array()` ([Kotlin docs, Wasm-JS interop](https://kotlinlang.org/docs/wasm-js-interop.html)). The fetched page does not document the cost or a `FloatArray` adapter, so per-flush copy cost is unmeasured. At 10k sprites x 20 floats = 200k elements per frame; if an adapter makes one JS call per element, a 5 ns call (anecdote in the prior file) gives about 1 ms, and 20 ns gives about 4 ms (my arithmetic).

**F10. ART/JVM: build in a primitive array, copy once. Tier: common practice (libGDX), plus official allocation guidance.**
- libGDX's `BufferUtils.copy(float[], Buffer, int, int)` delegates to a native `memcpy`; the class exists "to increase the speed of array/direct buffer and direct buffer/direct buffer transfers" (libgdx `master` at `c941e9bc`, `BufferUtils`). The batcher writes a `float[]` and does one bulk copy per flush.
- Android's own guidance (2009, Dalvik era): "the creation of short-lived objects in performance critical code paths is thus very costly" ([Android Developers Blog](https://android-developers.googleblog.com/2009/02/track-memory-allocations.html)). Since ART in Android 8.0, "Allocations are 70% faster than in Android 7.0" and "Pause times no longer scale with heap size", though read barriers add a small cost per field read ([source.android.com, ART improvements](https://source.android.com/docs/core/runtime/improvements)). Net: per-frame allocation is cheaper than in 2009 but still worth gating at about zero; the allocation counter stays in CI.
- I found no published per-sprite cost for a Kotlin vertex-building loop on ART. Gap.

**F11. WebGL buffer streaming: do not orphan on the web. Tier: browser-engineer advice in a list thread; undated in the fetch.**
- On `webgl-dev-list`, Ken Russell (Chrome): "WebGL's bufferData(target, size, usage) call is guaranteed to initialize the buffer to 0, so it can't be a no-op", and "I would recommend just keeping your code simple and not doing the bufferData(...) call attempting to abandon the buffer." A Firefox-side poster says `bufferData(null)` "causes us to upload a calloc'd buffer of that size" ([thread](https://groups.google.com/g/webgl-dev-list/c/vMNXSNRAg8M); the fetched page names that poster "Jeff Dash").
- A hobby engine reached the same conclusion in August 2026 and planned to remove orphaning from its web path ([neotolis-engine issue #325](https://github.com/d954mas/neotolis-engine/issues/325)); I do not know its A/B result. One project, one opinion.
- This overrides the prior file's F4 for the web backend only. Orphaning remains fine on desktop GL and GLES.

**F12. Instancing is not a safe web or iGPU optimisation. Tier: single bug report.**
- WebKit bug 239518, "ANGLE_instanced_arrays has poor performance, Intel": "Slow on integrated Intel, fast on dgpu AMD. Fast on OpenGL, slow on Metal." (2022-04-21, Safari) ([WebKit Bugzilla](https://bugs.webkit.org/show_bug.cgi?id=239518)). Supports keeping vertex expansion as the baseline.

### 3.4 How frame time is measured

**F13. Android, GL surface: use SurfaceFlinger timestats; FrameTimeline and Macrobenchmark do not clearly cover it. Tier: official docs, with an absence claim.**
- `dumpsys SurfaceFlinger --timestats` "provides the average FPS and a `presentToPresent` timing histogram for all layers that are being rendered", and Google says "frame stability is just as crucial as tracking the average frame rate. This is where you should measure the P90 and P99 frame rate metrics" ([Android Developers, Frame Rate](https://developer.android.com/games/optimize/framerate)). It gives a bucketed histogram, so percentiles are bucket-granular.
- Perfetto FrameTimeline: "FrameTimeline requires Android 12(S) or higher" and "SurfaceViews are currently not supported, but will be, in future" ([Perfetto docs](https://perfetto.dev/docs/data-sources/frametimeline)). A `GLSurfaceView` is a SurfaceView, so the FrameTimeline gap applies.
- Macrobenchmark `FrameTimingMetric` "captures precise timing information from frames produced by a benchmark journey, such as scrolling a list or a complex UI layout animation" and reports P50/P90/P95/P99; the page never mentions SurfaceView, GLSurfaceView or OpenGL ([doc](https://developer.android.com/topic/performance/benchmarking/macrobenchmark-metrics)). Treat it as unverified for GL.
- Swappy offers `SwappyGL_getStats` histograms in refresh counts (late, idle, latency frames) ([Swappy reference](https://developer.android.com/games/sdk/reference/frame-pacing/struct/swappy-stats)); I only saw it via search summary.
- Refresh rate must be pinned or recorded: "Many devices now support additional refresh rates such as 90Hz or 120Hz", and "there is no guarantee that your app will get the frame rate you request with `setFrameRate()`" ([Android Developers, frame rate API](https://developer.android.com/media/optimize/performance/frame-rate)). The A06 spec sources disagree on 60 vs 90 Hz.

**F14. Browser: rAF follows the display; `performance.now()` is coarse. Tier: MDN.**
- rAF "will generally match the display refresh rate. The most common refresh rate is 60hz ... though 75hz, 120hz, and 144hz are also widely used" ([MDN, requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame)). A 144 Hz laptop reports 6.9 ms frames; the gate must state its refresh and compare against 16.7 ms explicitly.
- `performance.now()` resolution is "5 microseconds" when cross-origin isolated and "100 microseconds" otherwise ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now)). Sub-100 microsecond phases cannot be timed individually; time the whole CPU frame, and serve the benchmark page cross-origin isolated.
- GPU timer queries in browsers: still unverified (see Gaps); do not depend on them.

## 4. How comparable projects do it

| Project | How it states or gates performance | Source |
|---|---|---|
| Bevy `many_sprites` stress test | Fixed 1920 x 1080, `PresentMode::AutoNoVsync`, `WinitSettings::continuous()`, 320 x 320 map of 64 px sprites; `--colored` deliberately breaks batches. Tracked on a dedicated "bevy-bench-01" box (Ryzen 9 9950X, RTX 3060). No mobile target found. | [`examples/stress_tests/many_sprites.rs`](https://github.com/bevyengine/bevy) at v0.20.0, `main`, `WindowPlugin`; [metrics.bevy.org](https://metrics.bevy.org) |
| Godot Benchmarks | Results JSON printed to stdout, saved via `--save-json`; a "Bunnymark" benchmark exists; hosted results run on a dedicated machine (detail from search only). No mobile gate found. | [godotengine/godot-benchmarks](https://github.com/godotengine/godot-benchmarks) |
| PixiJS, Phaser | Marketing numbers (100k, "tens of thousands"); no device list; no gate. | F3 |
| neotolis-engine (hobby, C and Wasm) | Bunnymark with a fixed spec (800 x 600, 26 x 37 px, premultiplied alpha, one atlas page, 20 B vertex); logs `fps cpu gpu draws bunnies`; "GPU time is `N/A` when `EXT_disjoint_timer_query_webgl2` is unavailable"; notes CPU numbers from an unpresented window are meaningless ("cpu_ms/frame_ms > ~0.5"). | [README](https://github.com/d954mas/neotolis-engine), [#322](https://github.com/d954mas/neotolis-engine/issues/322) |
| Android | Slow-session radar plus average, P90, P99 from timestats. | F13 |

Pattern: engines state a fixed scene and resolution, an uncapped or percentile metric, and a dedicated machine. None of the surveyed engines publishes a mobile sprite-count gate with named devices. SpartanGraphics would be setting a rarer kind of target, so the scene definition has to carry more weight.

## 5. Pitfalls and regrets

- **Counting sprites without area.** The profile's own numbers conflict (above). Fix the gate in Mpx covered.
- **Bunnymark is a small-sprite test.** About 960 px per bunny is light; a 64 px sprite is 4x heavier. Do not quote bunnymark counts against a 32–64 px profile.
- **Vendor "GPU ms" figures can be submission time**, not execution (Pixi's 2 ms for 100k sprites). Do not copy their method.
- **Window not presented gives false CPU numbers** (one developer's finding on Windows, issue #322): ensure the benchmark runs presented, in the foreground.
- **Refresh-rate drift.** 90/120 Hz Android panels and 144 Hz laptops move the rAF / present cadence. Record refresh and, on Android 11+, request 60 with `setFrameRate`, accepting that it is not guaranteed.
- **Thermal throttling is not measured by a 10-second run.** Mid-range phones without vapour chambers throttle; no source collected. Gate on a warmed-up, minute-long run and record SoC temperature or state.
- **Region-split SKUs.** A16 5G is Exynos 1330 / G68 MP2 in one review and reportedly Dimensity 6300 elsewhere. Pin the model number and SoC.
- **Web orphaning and instancing.** F11 and F12: both look attractive and both are web-hostile. Keep vertex expansion and plain sub-data upload.
- **Kotlin/Wasm element-wise interop and `Long`.** Per-element JS calls and `BigInt` conversions put 200k calls per frame on the hot path; build the whole batch in one array and cross once (F9).
- **Browsers differ 2.5x on identical code** (60 / 48 / 24 fps in the 2019 table). A single-browser gate is a Chrome gate; say so.

## 6. Recommendation for this repo

### The unit: sprites plus covered area

Define the scene profile as sprite count N and a covered-area ceiling A (sum of sprite pixel areas, in Mpx, at the rendered resolution). Single atlas, alpha-blended, rotating and moving, as before. My arithmetic:

| Scene | Covered Mpx | x 1080 x 2340 phone | x 720 x 1600 phone | x 1080p desktop |
|---|---|---|---|---|
| 5k x 32 px | 5.1 | 2.0 | 4.4 | 2.5 |
| 10k x 32 px | 10.2 | 4.1 | 8.9 | 4.9 |
| 5k x 48 px | 11.5 | 4.6 | 10.0 | 5.6 |
| 10k x 48 px | 23.0 | 9.1 | 20.0 | 11.1 |
| 10k x 64 px | 41.0 | 16.2 | 35.6 | 19.8 |

Fill budget (assumptions flagged): Mali-G52 MC2 peak about 3.4 Gpix/s means about 57 Mpx per frame at 60 fps; 10.2 Mpx is 18% of that and gives the Arm formula about 2.8 fragment-core cycles per pixel (5.5 at 5.1 Mpx). A one-sample, one-blend shader needs about 1 cycle at best. If real efficiency is 30–50% of peak (my assumption, unsourced), the usable budget is about 17–28 Mpx per frame, so 10.2 Mpx fits with little slack and 23 Mpx does not. Iris Xe at 31.2 Gpix/s peak has about 520 Mpx per frame; fill is irrelevant at these loads. Upload is 10k x 4 x 20 B = 800 KB per frame (48 MB/s), also irrelevant.

### Targets

| Platform | Gate | Scene | Metric | Confidence |
|---|---|---|---|---|
| Android, reference-mid | **5k sprites hard gate; 10k target** (promote to hard gate if the first run has at least 30% headroom, else keep 5k) | single atlas, 32 px sprites, A at most 10 Mpx for 10k and at most 5 Mpx for 5k, 60 s warm run | p95 present-to-present at most 16.7 ms, p99 reported (propose at most 33.3 ms), CPU render cost at most 4 ms | medium-low |
| Android, floor | 5k sprites, A at most 5 Mpx, 720 x 1600 | as above | p95 at most 16.7 ms, report only until measured | low |
| Web, reference | **10k sprites** (raise from "5k until measured"), 32 px, A at most 10 Mpx, 1080p canvas | single atlas, Chrome stable (ANGLE D3D11), cross-origin isolated page, 60 Hz pinned | p95 rAF delta at most 16.7 ms, CPU frame at most 5 ms | medium |
| Web, floor | 5k sprites on UHD 620, report only; Firefox and Safari report only | as above | same | low |
| Desktop | unchanged, but restate the profile in Mpx: 10k x 32 px is 10.2 Mpx (4.9x of 1080p), not "at most 4x" | | | the prior file's |

CPU frame budgets (4 ms Android, 5 ms web) are my proposals anchored only on Pixi's 1.5–5 ms per 10k (F3) and the 3 ms desktop figure in the prior file.

### Reference-device classes

- **Android reference-mid:** Mali-G57 MC2, Mali-G68 MP2 or Adreno 619-class, 1080 x 2340-class panel, Android 13+. Example: Samsung Galaxy A16 5G, Exynos 1330 / G68 MP2 variant (verified, F6). Alternate: any Dimensity 6300 / Mali-G57 MC2 device (pairing unverified).
- **Android floor:** Mali-G52 MC2, 720 x 1600. Example: Samsung Galaxy A06 (verified, F6). This is the entry end of the volume market, and a good device for catching CPU and fill problems that mid devices hide.
- **Web reference:** an Iris Xe laptop (Core i5-1135G7 / 1235U class) or Radeon 680M/760M laptop, 16 GB dual-channel, plugged in, Chrome stable. **Floor:** UHD 620 (8th-gen U-series). I did not source specific laptop SKUs beyond GPU names; the owner must pick hardware they can actually run.
- The earlier "Adreno 610" in the provisional text is a comparable floor class, but I could not source its fill rate, so I name the Mali floor instead and leave an Adreno device as an optional second floor.

### Measuring

- **Android:** `adb shell dumpsys SurfaceFlinger --timestats -clear -enable` before, `-dump` after the warm window; derive p95/p99 from the `presentToPresent` histogram; also log in-app CPU time per frame and the count of `eglSwapBuffers` stalls. Record refresh rate, `setFrameRate(60, ...)` request, battery/thermal state and device model plus SoC. Do not rely on Perfetto FrameTimeline or `FrameTimingMetric` for GL until a spike shows they see the layer (Gap).
- **Web:** collect rAF deltas and per-frame CPU time inside the page, serve with COOP/COEP for 5 microsecond timers, run presented in foreground, write JSON. Record browser version, refresh rate and `UNMASKED_RENDERER_WEBGL`.
- **Both:** the CI structural counters from the prior file stay the primary regression signal; these device runs only ratify the numbers per release.

### Design consequences for the batcher

- Web upload path: plain `bufferSubData` into a pre-sized buffer (or a small ring), no orphaning. Android and desktop may keep orphan or ring.
- Kotlin/JS: build into a `FloatArray` and upload it directly. Kotlin/Wasm: one bulk copy per flush; measure the adapter in the first spike before choosing Wasm for the web backend (this feeds the separate Wasm-versus-JS decision).
- Keep `Long` and boxed numerics out of the sprite loop on every target.

### Reasoning

The evidence cannot distinguish 5k from 10k on Android, but the fill, CPU and upload arithmetic all point to feasibility at 32 px and not at 48–64 px, so the stated area ceiling matters more than the count. Staging the hard gate (5k now, 10k on measured headroom) keeps the Level 4c gate honest without promising numbers that no published benchmark backs. The web gate can be higher because the machine has about 10x the fill rate and several independent JS anchors, but it is conditional on a Chrome-only, cross-origin-isolated setup. I reject "keep 5k for web": it is more conservative than any anchor I found and would make the gate meaningless on Iris Xe.

## 7. Load-bearing claims

1. **Pixi v8 bunnymark table, no hardware stated.** "100k sprites all moving | ~50ms | ~15ms" (V7 CPU / V8 CPU). Also: "CPU = time spent by the CPU rendering a single frame". The page names no device, browser or resolution. [pixijs.com/blog/pixi-v8-launches](https://pixijs.com/blog/pixi-v8-launches); `v8.0.0`, 2024-03-05.
2. **Pixi ParticleContainer.** "capable of rendering 100K+ without breaking a sweat"; test machine "Macbook Pro M3"; 200,000 sprites in a Container and 1,000,000 particles at 60 fps. [pixijs.com/blog/particlecontainer-v8](https://pixijs.com/blog/particlecontainer-v8) (quotes as extracted by the fetch tool).
3. **Phaser 4.** "Where standard Phaser rendering handles tens of thousands of sprites comfortably"; "Quads now use index buffers, reducing vertex upload costs by a third." [README `v4.2.1`](https://cdn.jsdelivr.net/npm/phaser@4.2.1/README.md), 2026-07-09 (as extracted).
4. **Moto G3 / Nexus 10 bunnymark.** "my smarphone Moto g3 3rd generation with Android 6.0.1" ... "with an astounding 17800 bunnies!"; Nexus 10: "it creates totally 17000 bunnies(Approx)" (30 fps threshold, as extracted). [html5gamedevs](https://html5gamedevs.com/topic/29786-game-is-very-slow-in-nexus-10-device), April 2017.
5. **Unscientific browser table.** The repo labels its comparison "(highly unscientific)"; MacBook Pro 2019, 8k boxes; Pixi 60 / 48 / 24 in Chrome / Firefox / Safari. [canvas-engines-comparison](https://github.com/slaylines/canvas-engines-comparison) README (as extracted; date not stated).
6. **500-sprite Snapdragon 720G test.** "I ran a test scene with 500 animated sprites on a mid-range Android device (Snapdragon 720G)." and "These aren't controlled lab benchmarks; they're from an actual profiling session on a real project device." [engineranked](https://engineranked.com/article/cocos-creator-review-2026/), 2026-03-27.
7. **Android timestats.** "This tool provides the average FPS and a `presentToPresent` timing histogram for all layers that are being rendered." and "frame stability is just as crucial as tracking the average frame rate. This is where you should measure the P90 and P99 frame rate metrics". [Android Developers, Frame Rate](https://developer.android.com/games/optimize/framerate) (retrieved 2026-10-11).
8. **FrameTimeline and SurfaceView.** "FrameTimeline requires Android 12(S) or higher" and "SurfaceViews are currently not supported, but will be, in future." [Perfetto docs](https://perfetto.dev/docs/data-sources/frametimeline) (retrieved 2026-10-11).
9. **Macrobenchmark scope.** `FrameTimingMetric` "captures precise timing information from frames produced by a benchmark journey, such as scrolling a list or a complex UI layout animation". The page does not mention SurfaceView or OpenGL. [Android Developers](https://developer.android.com/topic/performance/benchmarking/macrobenchmark-metrics) (retrieved 2026-10-11).
10. **Refresh rate.** "Many devices now support additional refresh rates such as 90Hz or 120Hz." and "there is no guarantee that your app will get the frame rate you request with `setFrameRate()`". [Android Developers, frame rate API](https://developer.android.com/media/optimize/performance/frame-rate) (retrieved 2026-10-11).
11. **Fill rate on low-end GPUs.** "On less performant GPUs, available fill-rate—the speed at which the GPU can fill the frame buffer—can be low." and "Although low-end devices continue to improve in GPU performance, their displays remain at relatively low resolutions." [Android Developers overdraw page](https://developer.android.com/topic/performance/rendering/overdraw) (retrieved 2026-10-11).
12. **Arm fragment budget.** "fragCycleBudget = (No of GPU fragment cores) * (gpuFrequency in Hz) / ( FPS * numPixels)". [Arm community blog](https://developer.arm.com/community/arm-community-blogs/b/mobile-graphics-and-gaming-blog/posts/gpu-processing-budget-approach-to-game-development), 2017-07-03.
13. **Mali-G52 throughput.** "Mali-G52 will be clocked up to 850 MHz delivering 6.5 Gpix/s pixel/texturing throughput, and be manufactured using a 16nm process technology." [CNX Software](https://www.cnx-software.com/2018/03/06/arm-introduces-mali-g52-mali-g31-gpus-mali-d51-display-processor-and-mali-v52-video-processor-for-mainstream-devices/), 2018-03-06.
14. **Iris Xe fill.** "Taux de remplissage de pixels" ... "31.20 GPixel/s" (Iris Xe 96EU) and "1300 MHz" boost. [askgeek](https://askgeek.io/fr/gpus/vs/Intel_Iris-Xe-Graphics-96EU-Raptor-Lake-vs-Intel_UHD-Graphics-Raptor-Lake-64-EU) (undated database).
15. **Best sellers and specs.** "The Galaxy A16 5G was Samsung's best-selling phone of the year, followed by the Galaxy A06 4G." ([SamMobile](https://www.sammobile.com/news/galaxy-s25-ultra-only-high-end-android-phone-worlds-top-ten-selling-phones/), relaying Counterpoint 2025). A06: "Mali-G52 MC2", "720 x 1600 pixels" ([GSMArena](https://m.gsmarena.com/samsung_galaxy_a06-ampp-13265.php)). A16 5G review: "Our review unit is running the Exynos 1330" with Mali-G68 MP2 and 1080 x 2340 ([GSMArena review](https://m.gsmarena.com/samsung_galaxy_a16_5g-review-2758p4.php)).
16. **Intel share of WebGL2 GPUs.** Web3DSurvey lists "Google Inc. (Intel)" at 30%. [web3dsurvey.com/webgl2](https://web3dsurvey.com/webgl2) (undated).
17. **Timer precision.** "Resolution in isolated contexts: 5 microseconds" and "Resolution in non-isolated contexts: 100 microseconds". [MDN, Performance.now()](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now) (retrieved 2026-10-11).
18. **rAF cadence.** "The frequency of calls to the callback function will generally match the display refresh rate." [MDN, requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) (retrieved 2026-10-11).
19. **Kotlin/JS arrays.** `FloatArray` maps to `Float32Array`; `Long` maps to `BigInt` only with `target = "es2020"` or `-Xes-long-as-bigint` (marked experimental). [Kotlin docs, Use JavaScript code in Kotlin, type table](https://kotlinlang.org/docs/js-to-kotlin-interop.html) (retrieved 2026-10-11, as extracted).
20. **Kotlin/Wasm arrays.** "Arrays: Not supported in interop directly yet. You can use the new `JsArray` type instead." [Kotlin docs, Wasm-JS interop](https://kotlinlang.org/docs/wasm-js-interop.html) (retrieved 2026-10-11).
21. **libGDX bulk copy.** `BufferUtils` exists "to increase the speed of array/direct buffer and direct buffer/direct buffer transfers"; `copy(float[] src, Buffer dst, int numFloats, int offset)` calls native `memcpy`. libgdx `master` @ `c941e9bc`, `gdx/src/com/badlogic/gdx/utils/BufferUtils.java`, symbol `BufferUtils.copy`; latest release `1.14.2` (2026-06-05).
22. **WebGL orphaning.** "WebGL's bufferData(target, size, usage) call is guaranteed to initialize the buffer to 0, so it can't be a no-op." and "I would recommend just keeping your code simple and not doing the bufferData(...) call attempting to abandon the buffer." (Ken Russell). [webgl-dev-list thread](https://groups.google.com/g/webgl-dev-list/c/vMNXSNRAg8M) (as extracted; undated).
23. **WebKit instancing bug.** "Slow on integrated Intel, fast on dgpu AMD. Fast on OpenGL, slow on Metal." [WebKit bug 239518](https://bugs.webkit.org/show_bug.cgi?id=239518), 2022-04-21.
24. **ART allocation.** "Allocations are 70% faster than in Android 7.0." and "Pause times no longer scale with heap size". [source.android.com ART improvements](https://source.android.com/docs/core/runtime/improvements), Android 8.0 page (retrieved 2026-10-11). Older guidance: "the creation of short-lived objects in performance critical code paths is thus very costly" ([Android Developers Blog, 2009-02](https://android-developers.googleblog.com/2009/02/track-memory-allocations.html)).
25. **Bevy benchmark practice.** `many_sprites.rs` comment: "Since this is also used as a benchmark, we want it to display performance data."; window `WindowResolution::new(1920, 1080)`, `PresentMode::AutoNoVsync`. Dedicated machine "bevy-bench-01" with "AMD Ryzen 9 9950X" and "NVIDIA GeForce RTX 3060". [bevy v0.20.0 `examples/stress_tests/many_sprites.rs`](https://github.com/bevyengine/bevy), [metrics.bevy.org](https://metrics.bevy.org).
26. **Presented-window guard.** "Benchmark guard worth adopting in perf tooling: a case where cpu_ms/frame_ms > ~0.5 was measured against an unpresented window and its render_ms is meaningless". [neotolis-engine #322](https://github.com/d954mas/neotolis-engine/issues/322), Aug 2026 (one developer, Windows/DWM).

Claims I stated but cannot back with a quote are in Gaps.

## 8. Gaps

- **No measured data for the central question.** No named-device Android or iGPU sprite benchmark exists in what I could reach. Every number in §6 is arithmetic plus weak anchors; the targets are provisional until the first sample-app run. This blocks nothing about the *structure* of D13 (unit, device classes, measurement method) but does block treating 10k as a committed Android number.
- **Mali pixel rates are second-hand.** The 2 pixels/clock/core figure comes from search-result relays of an Arm table and a Wikipedia row I could not parse; the PDF Arm table would not render (no `pdftoppm`). CNX's 6.5 Gpix/s does not match my MC2 figure and the core count behind it is unstated.
- **No Adreno 610/619, Mali-G57/G68 or AMD 680M/760M fill figures.** TechPowerUp and Notebookcheck returned 403; Wikipedia's Adreno table lost its column headers. The Adreno floor is therefore unsupported; I name a Mali floor instead.
- **Fill-efficiency assumption (30–50% of peak) is unsourced.** It drives the "17–28 Mpx" usable budget.
- **No per-sprite CPU cost on ART, Kotlin/JS or Kotlin/Wasm.** Only Pixi's JS scene-graph numbers (hardware unstated). The 4 ms and 5 ms CPU budgets are proposals.
- **Kotlin/Wasm bulk-copy cost unmeasured**; the docs page does not describe a `FloatArray` adapter or its cost.
- **Dimensity 6300 / Mali-G57 MC2 for Galaxy A16 5G** rests on a search summary; GSMArena's review unit is Exynos 1330. A second GSMArena page I fetched for "A16 5G" returned Unisoc T606 / Mali-G57 MP1, which contradicts both (possibly a different model page); I did not use it.
- **Counterpoint's own page did not render**; the ranking is via SamMobile. Rank positions beyond "A16 5G then A06" are not established.
- **Web3DSurvey figure is undated and mixes desktop and mobile.**
- **GPU timer query availability in Chrome, Firefox and Safari in 2026 is unverified**; the MDN page for `EXT_disjoint_timer_query_webgl2` returned 404. GPU time on web remains out of the gate.
- **Whether `FrameTimingMetric` or FrameTimeline sees a `GLSurfaceView` layer** is undocumented in what I read; needs a one-device spike. `dumpsys SurfaceFlinger --timestats` is the documented path.
- **Thermal throttling on mid-range phones**: no evidence gathered.
- **Firefox and Safari numbers** rest on the single unscientific 2019 table.
- **The Godot benchmark hosting detail** (dedicated machine, daily runs, three GPUs) came from a search summary; its contributor page returned 404 and the live site exceeded the fetch size limit. Not relied on.
- **Brief gaps:** I do not know which physical devices the owner has or will buy; the reference-device section names classes with examples, and the owner must choose hardware they can run. The prior file's desktop profile has the Mpx inconsistency noted above; resolving it is a call for whoever owns the desktop gate.
- **Firecrawl not used** (credits negative, so skipped); everything came from `WebSearch`, `WebFetch` and `gh`. Several fetches returned only a model-extracted summary rather than raw text; quotes marked "as extracted" were not checked against raw page text.
