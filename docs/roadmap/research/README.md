# Roadmap research: plan, summary and verification

- **For:** `docs/roadmap/vision-and-roadmap.md`, the target-state and phased roadmap for
  `SpartanLaboratories/SpartanGraphics`.
- **Written by:** the planner. The `<question-slug>.md` files beside this one belong to the
  researchers who wrote them and are never edited here.
- **Dates:** research plan and findings 2026-10-11; verification and summary 2026-10-19.
  Baseline: `main` @ `50b61e2`, issue #1 as updated on 2026-10-11.
- **Citations:** documents by section, code by symbol, never by line number.

This file has three parts: the research plan (written before any researcher started), the
verification results, and the summary that stage 3 designs from.

---

## Part 1: Research plan

### 1.1 Binding constraints

The user's interview answers, one line each. Every researcher brief carries them.

| ID | Constraint |
|----|------------|
| Q1a | The roadmap describes the final state, phased all the way, with 1.0 as a named milestone along the way. |
| Q1b | Versions follow 0.x; breaking changes are batched at phase boundaries into 0.2.0, 0.3.0 and so on; 1.0.0 is the deliberate stability release. |
| Q1c | 1.0 ships after the web backend. |
| Q1d | 1.0 freezes the app-facing API and the backend contracts. The contracts graduate to `@SupportedExtension` with a documented example backend. |
| Q2 | Scheduled: R1–R7, T1, I1, A1–A3, S1, S2. Listed but unscheduled ("Later"): R8, T2, I2, A4. Out: nothing. |
| Q3a | Official platforms: Windows, Android, web. macOS and Linux are "should work, untested" with compile-only CI, listed as Later. iOS is out of the roadmap. |
| Q3b | Order: real fonts and text input, then Android, then web, then 1.0, then the remaining features. |
| Q4a | 2D with room for 3D: the API reserves depth and 3D transforms from the start. The 3D track is "beyond this roadmap", unscheduled; every scheduled phase is 2D; the frozen 2D API leaves that room. |
| Q4b | About 10k sprites on screen at 60 fps on a mid-range desktop GPU. Android and web targets are set by research. Batched drawing comes early, enforced by a Level 4c performance gate. |
| Q5 | An unpublished sample app per platform in this repo proves the Android and web backends. Desktop is proven by the game client. |
| Q6a | Roadmap at `docs/roadmap/vision-and-roadmap.md`; research in `docs/roadmap/research/`; replaced versions in `docs/roadmap/superseded/`. |
| Q6b | Roadmap Phase 1 is G1–G7 (issue #1); later phases are 2..N, with a note that the split plan's Phases 2 and 3 belong to SpartanUI and the game client. |

Feature codes (Q2): R1 more shapes (SpartanUI's needs first); R2 more outline shapes; R3 sprites,
atlases and frame animation; R4 generic tile-map layers; R5 render-to-texture, several cameras or
viewports and a generic visibility mask; R6 particles (late); R7 custom shaders and
post-processing, Experimental, last; T1 real fonts; I1 touch and gestures, text input, IME and
clipboard, custom cursors; A1 playback control and WAV/OGG; A2 volume buses and streamed music
with crossfade; A3 2D positional audio (late); S1 asset manager; S2 hot reload and a build-time
Gradle plugin. Later: R8 parallax, skeletal animation, 2D lighting; T2 CJK, shaping, RTL;
I2 gamepad; A4 reverb, filters, ducking.

Assumptions the user saw and did not object to: the split plan is the baseline and G1–G7 is
summarised, not re-planned; the roadmap does not decide the architecture draft's proposals (they
go to Open decisions with recommendations); one shared version for all artifacts; audio stays in
this repo and does not depend on graphics; the shape set stays closed, with custom drawing only
through R7; research picks the rendering API; GeneralTools#7 is an open decision; breaking changes
are batched at phase boundaries.

Settled facts: issue #1 is the source of truth for Phase 1. G3 adds a screen-space clip rectangle
to graphics-core's draw model, and G5's renderer is batched and draws the clip with a scissor.

### 1.2 Reuse first

An interrupted planner run left ten complete findings files here, written on 2026-10-11 under
the previous researcher format (no "Load-bearing claims" section). All ten were current when this
plan was written (same day, same versions), and the 2026-10-19 verification re-checked their
claims against today's sources. Each is **reused** by path; none is researched again.

They cite the previous run's interview by number. The mapping to today's answers:

| Old reference | Today |
|---|---|
| decision 2 (0.x breaks allowed) | Q1b |
| decision 4 (1.0 freezes API and contracts) | Q1d |
| decision 7 (reserved depth) | Q4a |
| decision 9 (sample apps) | Q5 |
| decision 10 (feature list) | Q2 |
| decision 11 (closed shape set; one module per device family) | the closed-shape-set assumption and the split plan's module rule |
| decision 11, read as "libraries never depend on GeneralTools" | **superseded**: GeneralTools#7 is an open decision |

| File | Reused for | Known weaknesses (from its own Gaps) |
|---|---|---|
| [`gpu-api-per-platform.md`](gpu-api-per-platform.md) | D1, part of D2 and D3 | GLSL-prefix and GL-vs-WebGPU convention claims are spec recall; Wasm-vs-JS cost rests on one maintainer's README |
| [`backend-contract-design.md`](backend-contract-design.md) | D4, part of D5 | klib compatibility unsettled; Android and web lifecycle docs not fetched |
| [`idiomatic-kotlin.md`](idiomatic-kotlin.md) | D4, D6 types | klib default-method compatibility unproven; several items from memory |
| [`room-for-3d.md`](room-for-3d.md) | D6 | depth/stencil-at-context-creation from general knowledge |
| [`text-rendering.md`](text-rendering.md) | D9 | Android glyph APIs unverified; no pure-Kotlin rasteriser sizing |
| [`cross-platform-input.md`](cross-platform-input.md) | D10 | Android key and IME pages not parsed; GLFW key-token meaning unverified |
| [`cross-platform-audio.md`](cross-platform-audio.md) | D11 | Android codec table from memory; OpenAL mono-only rule unverified |
| [`asset-pipeline.md`](asset-pipeline.md) | D12 | vanniktech plugin-marker support unverified; some Android asset facts from memory |
| [`render-performance-and-verification.md`](render-performance-and-verification.md) | D13, D14 | Android and web targets are judgement on thin evidence; CI GL spike not run |
| [`consumer-needs-and-dependencies.md`](consumer-needs-and-dependencies.md) | D15, inputs to D11 and D16 | audio durations not measured; GameTools vision wire format unspecified |

The previous run's "best 2D graphics systems" researcher wrote nothing. Its need is folded into
the new questions N2 and N3 below as their "best systems for the purpose" lens, aimed at the
decisions those features force on the frozen contracts.

### 1.3 Decisions stage 3 must make, and the research behind each

Stage 3 produces the target architecture and the phase list. These are the decisions it makes
that the interview did not settle.

**Platform technology**

| ID | Decision | Options | Criteria | Evidence that settles it | Research |
|---|---|---|---|---|---|
| D1 | GPU API per backend, and shader strategy | GL family everywhere; WebGPU; Vulkan/Metal; a C abstraction | Q3a, Q4a, Q4b, Q1d; "research picks the rendering API" | stable bindings per platform; reach; 2D feature needs | reused: `gpu-api-per-platform.md` |
| D2 | Web compilation target, and G1's non-JVM compile-check target | Kotlin/JS; `wasmJs`; both via the shared `web` source set | Q3a, Q1c, Q1d, Q4b, Q5 | current stability; interop cost for WebGL and typed arrays; consumer reach; dependency support | **new: N1** (builds on `gpu-api-per-platform.md` §3.5) |
| D3 | Android build form and platform floors | minSdk choices; Android-KMP library plugin vs `com.android.library` | Q3a, Q5; libraries target JVM 21 | AGP and Kotlin docs; distribution data; API levels the features need | **new: N5** |

**Contracts that 1.0 freezes (Q1d)**

| ID | Decision | Options | Criteria | Evidence that settles it | Research |
|---|---|---|---|---|---|
| D4 | Backend contract level and post-1.0 evolution | shape-level `Renderer`; frame-level hybrid; GL-call level | Q1d, Q4a, Q2; additive evolution | what pluggable-backend libraries do; how contracts broke | reused: `backend-contract-design.md`, `idiomatic-kotlin.md` |
| D5 | Binary-compatibility promise off the JVM | JVM binary + source elsewhere; full binary everywhere | Q1d | Kotlin's klib ABI guarantees and tooling | **new: N1** (sub-question) |
| D6 | What 1.0 reserves for 3D, and what it must not | reserve at seams; reserve in types | Q4a, Q1d | 2D-to-3D library precedents | reused: `room-for-3d.md` |
| D7 | What R1/R2 shapes need from the frozen contract | universal pipeline suffices; extra pipelines or attributes; MSAA request | Q2 R1/R2, Q1d, Q4b; closed shape set | how mature 2D renderers draw and antialias vector shapes | **new: N2** |
| D8 | What R3–R6 need from the frozen contract | per-frame streaming only; retained buffers; render-target passes; stencil | Q2 R3–R6, Q1d, Q4b | how mature 2D engines do tile maps, render targets, masks, particles | **new: N3** |
| D9 | Text architecture and the Phase-1 text contract | layout in common code; platform text APIs | Q2 T1/T2, Q1d | engine practice; platform APIs | reused: `text-rendering.md` |
| D10 | Input growth, and where each part of I1 lands | queue vs `Flow`; physical vs logical keys; session-based text input | Q2 I1/I2, Q3b, Q1d | engine and platform practice | reused: `cross-platform-input.md` |
| D11 | Audio contract and backends, and A1–A3 timing against the 1.0 freeze | as listed in the file | Q2 A1–A4, Q1d, Q3b | platform audio APIs; libGDX/miniaudio/Godot practice | reused: `cross-platform-audio.md`, `consumer-needs-and-dependencies.md` §3.4 |
| D12 | Asset system: source contract, manager, hot reload, Gradle plugin | as listed in the file | Q2 S1/S2, Q1d | KMP and web asset systems; Gradle plugin practice | reused: `asset-pipeline.md` |

**Proof and performance**

| ID | Decision | Options | Criteria | Evidence that settles it | Research |
|---|---|---|---|---|---|
| D13 | Batching design, Level 4c gate, Android and web targets | as listed | Q4b | batcher designs; published throughput on mid-range devices | reused: `render-performance-and-verification.md`; **new: N4** for the Android and web numbers |
| D14 | Renderer verification without a GPU; role of the sample apps | software rasterisers; recording backend; manual only | Q4b, Q5, Q3a | CI practice of renderers | reused: `render-performance-and-verification.md` |

**Consumers and ordering**

| ID | Decision | Research |
|---|---|---|
| D15 | What SpartanUI and the game client need, by phase; which Phase-1 seams matter | reused: `consumer-needs-and-dependencies.md` |
| D16 | How the post-1.0 features group into phases | judgement at stage 3, from D7, D8, D11, D12 and D15 |
| D17 | Module layout of the final state, including where a shared GL renderer lives | judgement at stage 3, from D1, D4 and D12; the draft architecture's proposals stay Open decisions |

Not researched: GeneralTools#7 (an open decision by assumption; `consumer-needs-and-dependencies.md`
§3.7 informs the recommendation), versioning (Q1b), one shared version and audio's home
(assumptions).

### 1.4 New questions

Each researcher works through three lenses inside its question: the best systems for the purpose,
common practice and convention, and idiomatic Kotlin.

| ID | File | Question | Informs |
|---|---|---|---|
| N1 | [`web-target-kotlin-js-vs-wasm.md`](web-target-kotlin-js-vs-wasm.md) | As of October 2026, should the web backend and the Phase-1 compile check target Kotlin/JS, Kotlin/Wasm, or both; and what binary-compatibility guarantee can a frozen KMP contract give third-party implementers on klib targets? | D2, D5 |
| N2 | [`shape-rendering-and-antialiasing.md`](shape-rendering-and-antialiasing.md) | How should the R1/R2 shapes be drawn and antialiased on GL 3.3, GLES 3.0 and WebGL2, and what does the chosen technique require of the backend contract frozen at 1.0? | D7 |
| N3 | [`tilemaps-render-targets-and-masks.md`](tilemaps-render-targets-and-masks.md) | What do sprite atlases and animation, tile-map layers, render-to-texture with several cameras and a visibility mask, and particles require of the frozen backend contract (retained geometry, render targets, stencil, blend modes), and how do mature 2D engines build them? | D8 |
| N4 | [`android-and-web-performance-targets.md`](android-and-web-performance-targets.md) | What sprite counts at 60 fps does a well-batched 2D renderer sustain on a mid-range Android phone (GLES 3.0) and in WebGL2 on a mid-range laptop integrated GPU, so the Android and web Level 4c targets can be set? | D13 |
| N5 | [`android-build-and-platform-floors.md`](android-build-and-platform-floors.md) | How should a KMP library publish its Android artifacts in October 2026, and what minSdk and other platform floors should the Android backend adopt? | D3 |

### 1.5 Verification plan

- Each new findings file ends with its load-bearing claims.
- The reused files have no such section, so the planner extracts their load-bearing
  outside-world claims from their Findings.
- One verifier receives the combined list. Its verdicts are recorded in Part 2.
- A claim that fails leaves the design inputs, and any decision resting on it reopens.
- Claims about this repo, the monolith client (`SpartanLabsGaming/GameGraphics`) and the sibling
  repos are spot-checked by the planner against the code, not by the verifier. Those checks are
  recorded in Part 2 too.

---

## Part 2: Verification results

### 2.1 How the claim list was built

- **Reused files:** they have no "Load-bearing claims" section, so the planner extracted the
  outside-world claims their recommendations rest on from their Findings.
- **New files (N1–N5):** their §7 "Load-bearing claims" lists, taken whole.
- **Two tiers, one verifier.**
  - **Tier A** (about 125 claims): every claim a roadmap-level decision (D1–D17) rests on,
    checked fully, including currency.
  - **Tier B** (about 90 claims): every other listed claim. These are supporting precedents or
    details that only a later phase plan will use. They were checked for existence only.
- **Duplicates merged before sending:** G5 into N5-17, IN10 into N5-13, AU6 into N5-16, RP12
  into N4-9, K4 into N1-15, B9 into N1-14, K10 into N1-12, and the Wasm-speed half of G12 into
  N1-8.
- **Runs:** the first verifier (2026-10-11) died on an API usage limit before returning
  anything. The second run (2026-10-19) produced the verdicts in 2.2.

### 2.2 Verdicts

**Totals.** Tier A (124): 118 hold, 2 fail, 4 unclear. Tier B (91): all 91 hold. No claim went
unchecked.

**What the failures and unclear verdicts change:**

| ID | Verdict | Effect on the design |
|---|---|---|
| AS5 | fails | The cited MDN page only links the `webglcontextlost`/`webglcontextrestored` events; it gives no handling advice. The claim leaves the design inputs. The requirement that web GPU resources are rebuilt after context loss stands on N3-C26 (the WebGL spec's "Context Restored Event"), which holds. |
| AS9 | fails at the cited page | The Gradle user guide does not say it. The Plugin Portal's own page does ([plugins.gradle.org/docs/publish-plugin](https://plugins.gradle.org/docs/publish-plugin)): "All properties are mandatory", a "manual review process", and the `io.github` prefix for GitHub users. The plugin-publishing Open decision cites that page instead. |
| N5-4 | unclear | The AGP roadmap confirms AGP 10.0 (late 2026) removes the `android.newDsl` and `android.builtInKotlin` opt-outs. It does not say plain `com.android.library` stays supported; that is inference. The Android phase confirms it, and the fallback is a single-target KMP module on the Android-KMP plugin. |
| AU7 | unclear | Only "WAV, AIFF and AU are the standard Java Sound formats" is on the cited page. That is enough for "Java Sound alone does not play MP3 or Ogg", and the desktop decoder choice is unaffected. |
| RP6 | unclear | The Khronos wiki would not open; search summaries agree with the claim. Nothing depends on it: the batcher uses orphaning or a small ring and never persistent mapping. |
| RP13 | unclear | The 2018 thread would not open. Current browser support for GPU timer queries is only secondary-sourced. The 4c gate does not use browser GPU timers, so nothing changes. |

Partial support worth knowing, though the verdict is `holds`:
- **G15:** `GL30 extends GL20` is in libGDX's source, not on the cited wiki page.
- **G16:** "deprecated since 10.14" is not on the macOS 26 page.
- **B7:** the quoted wording is a paraphrase of "even adding default arguments can break
  compatibility".
- **CN1:** that browsers have no raw UDP socket is implied ("WebTransport is not a UDP Socket
  API"), not stated outright.
- **AS2:** `FileAssetReader` being absent on wasm and Android comes from `bevy_asset`
  `io/source.rs`, not from the cited page.
- **B8:** the answer to its question is that `@SubclassOptInRequired` arrived in 1.8 as
  Experimental and has been Stable since Kotlin 2.1.

#### Tier A (checked fully, including currency)

| ID | Claim | Feeds | Verdict | Verifier's finding (2026-10-19) |
|---|---|---|---|---|
| N1-1 | Kotlin/Wasm Beta, Kotlin/JS Stable | D2 | holds | Both stability pages; Wasm Beta since 2.2.20 |
| N1-3 | Roadmap: "Promote Kotlin/Wasm to Stable" (KT-88663), no version | D2 | holds | "In focus now"; modified Aug 2026, next Feb 2027 |
| N1-4 | Wasm browser floor Chrome 119, Firefox 120, Safari 18.2 | D2 | holds | `wasm-configuration` page |
| N1-5 | Shared `web` source set since 2.2.20, aimed at library authors | D2 | holds | What's new 2.2.20 |
| N1-6 | `FloatArray.toFloat32Array()` is a zero-copy `unsafeCast` on JS | D2 | holds | kotlinx-browser @ `9a2f01f` |
| N1-7 | The same call copies element by element on `wasmJs` | D2 | holds | kotlinx-browser @ `9a2f01f` |
| N1-8 | Kool: Wasm slower than JS; JS recommended | D2 | holds | Kool README, verbatim |
| N1-10 | Compose web combines JS and Wasm distributions for fallback | D2 | holds | What's new Compose 1.9.0, verbatim |
| N1-12 | kotlin-logging 8.0.4 and coroutines 1.11.0 publish `js` and `wasm` variants | D2 | holds | Gradle module metadata |
| N1-13 | Klib guarantees "not yet finalized" | D5 | holds | BCV `KLibSupport.md` |
| N1-14 | BCV klib validation is experimental | D5 | holds | BCV README; BCV 0.18.2 is current |
| N1-15 | KGP ABI validation since 2.2.0, experimental, DSL reshaped in 2.4.0 | D5 | holds | KGP docs and What's new 2.4 |
| N1-16 | Non-JVM consumers need the library's language version or later | D5 | holds | Kotlin API guidelines, verbatim |
| N1-17 | Partial linkage always on from 2.4.0, log level `SILENT` | D5 | holds | What's new 2.4, verbatim |
| N1-18 | Old klib class inherits a newly added default member | D5 | holds | Kotlin v2.4.20 test `severalInheritedImplementations` |
| N1-19 | Adding an abstract member breaks on Native, JS, Wasm (KT-85018) | D5 | holds | v2.4.20 `addUnimplementedAbstractMethod/project.info` |
| G1 | WebGL2 about 97% global support | D1 | holds | caniuse 97.13% |
| G2 | Browser WebGPU not yet universal | D1 | holds | web.dev, gpuweb wiki (also: Chrome on Linux for Intel Gen12+ and NVIDIA on Wayland) |
| G3 | LWJGL has no WebGPU binding; issue #614 open | D1 | holds | Open, updated 2026-09-18 |
| G4 | wgpu4k 0.1.x Beta, JDK 25+, Android experimental | D1 | holds | wgpu4k README |
| N5-17 | androidx.webgpu `1.0.0-alpha06`, minSdk 26 | D1, D3 | holds | AndroidX release page |
| G6 | GLES 3.0+ on about 99.5% of handhelds; about 7.4% no Vulkan | D1 | holds | Dashboard, data to 2025-11-24 |
| N5-12 | GLES 3.0 from API 18 | D1, D3 | holds | Android OpenGL overview, verbatim |
| G8 | GameActivity is C/C++-oriented; Swappy is native-only | D1, D3 | holds | Both Android pages |
| G9 | EGL context loss deletes GL resources; recreate in `onSurfaceCreated` | D4, D12 | holds | Same Javadoc, read from AOSP `android-14.0.0_r1` |
| N3-C27 | `GLSurfaceView` default config has no stencil; context loss deletes resources | D8, D4 | holds | AOSP `android-14.0.0_r1` |
| N3-C26 | WebGL `stencil = false` by default; resources invalid and extensions lost after restore | D8, D4 | holds | WebGL 1.0 spec, raw check |
| N3-C28 | GLFW defaults: 24 depth bits, 8 stencil bits | D8 | holds | GLFW window guide |
| G12 | Kool has no OpenGL on macOS; its desktop WebGPU needs Java 22 | D1 | holds | Kool README |
| G15 | libGDX `GL30 extends GL20`, implemented per backend | D17 | holds | In source; wiki page says less |
| G16 | macOS 26 still ships OpenGL; no removal announced | D1 | holds | macOS 26 notes: "OpenGL still remains in the SDK" |
| G17 | WebGL2 is derived from OpenGL ES 3.0 | D1 | holds | WebGL 2.0 spec, Introduction |
| N5-1 | The Android-KMP plugin is the supported path; legacy KMP APIs removed in AGP 10 | D3 | holds | Android KMP plugin page, verbatim |
| N5-2 | KMP is incompatible with `com.android.library` from AGP 9 | D3 | holds | Kotlin AGP 9 migration page, verbatim |
| N5-4 | AGP 10 removes the opt-outs; plain `com.android.library` stays | D3 | **unclear** | First half sourced; second half not stated |
| N5-5 | AGP 9.4.0 needs Gradle 9.6.0 and JDK 17; max API 37 | D3 | holds | AGP release notes (updated 2026-10-16) |
| N5-7 | KGP accepts a `jvm` variant for an `androidJvm` consumer | D3 | holds | KGP v2.4.0 `KotlinPlatformType.CompatibilityRule` |
| N5-8 | vanniktech needs no special config for the Android-KMP plugin | D3 | holds | 0.37.0 docs; 0.35.0 notes |
| N5-10 | AndroidX default minSdk 24, aimed at 99% of Play check-ins | D3 | holds | Jetpack versions page, verbatim |
| N5-11 | API 24+ 96.6%, 26+ 96.1% (Statcounter, April 2026) | D3 | holds | apilevels.com |
| N5-13 | `PointerIcon.create` added in API 24 | D3, D10 | holds | Search extraction of the reference page |
| N5-16 | Audio focus: `AudioFocusRequest` from API 26; fade-out from 12; top-app rule at 15 | D3, D11 | holds | Audio-focus page |
| N5-18 | Google's Java table stops at Java 17 (Android 14) | D3 | holds | JDKs in Android builds |
| N5-20 | A JDK 21 build leaks JDK 21 APIs; `-Xjdk-release` fixes it | D3 | holds | Jake Wharton, 2024-03-13 |
| K3 | `-jvm-default=enable` is the default from 2.2 and keeps old compiled code working | D4, D5 | holds | What's new 2.2; Java interop page |
| B8 | `@SubclassOptInRequired` gates only subclassing | D4 | holds | 1.8 Experimental; Stable since 2.1 |
| B6 | kotlinx.coroutines guards `SharedFlow`/`StateFlow` inheritance | D4 | holds | `Annotations.kt` |
| B7 | Even adding default arguments can break compatibility | D4 | holds | API guidelines (claim wording paraphrased) |
| K5 | Data classes in public API are a weakness | D4 | holds | API guidelines, verbatim |
| B5 | libGDX 1.12.0 `touchCancelled` was a breaking change | D4 | holds | libGDX CHANGES |
| B1 | Kool's `RenderBackend` is frame-level | D4 | holds | `RenderBackend.kt` members |
| B4 | Godot `RenderingDevice` absent in headless and Compatibility modes | D4 | holds | Godot class docs, verbatim |
| K1 | Value classes box as generics, interfaces, nullables; names mangled | D6 | holds | Kotlin inline-classes page |
| K7 | `AutoCloseable` is common stdlib since 2.0 | D4 | holds | Stdlib reference |
| K9 | `runBlocking` does not exist on JS or Wasm/JS | D12 | holds | Declared only in the `concurrent` source set |
| R3D5 | Bevy's 2D-transform document lists z-layering problems | D6 | holds | Bevy design document |
| R3D6 | Bevy #4149 still open | D6 | holds | Open, updated 2025-10-14 |
| R3D1 | libGDX `SpriteBatch`: 2D position, 20 floats per sprite, one `mat4` per batch | D6, D13 | holds | `SpriteBatch` source |
| R3D3 | MonoGame writes `layerDepth` into vertex z | D6 | holds | MonoGame source |
| R3D4 | Godot separates 2D and 3D servers; `Transform2D` is 2x3 | D6 | holds | Godot docs |
| N2-15 | GLES guarantees only width-1 lines | D7 | holds | ES refpage, verbatim |
| N2-23 | WebGL `antialias` leaves technique and quality to the implementation | D7 | holds | WebGL spec @ `714857a` |
| N2-25 | Multisample textures are not in GLES 3.0 | D7 | holds | ES refpage version table |
| N2-9 | Godot recommends geometry antialiasing | D7 | holds | Godot docs 4.4, verbatim |
| N2-10 | Godot 2D MSAA absent from the Compatibility renderer; edges only | D7 | holds | Same page |
| N2-4 | Dear ImGui's fringe uses zero-alpha outer vertices | D7 | holds | ImGui v1.92.9b |
| N2-5 | Dear ImGui samples a white texel for shapes | D7 | holds | ImGui v1.92.9b |
| N2-3 | NanoVG scales tessellation tolerance and fringe by pixel ratio | D7 | holds | NanoVG @ `ce3bf74` |
| N3-C3 | Godot tile quadrants (default 16) | D8 | holds | Godot 4.7.2-stable |
| N3-C4 | bevy_ecs_tilemap 64x64 chunks | D8 | holds | v0.15.0 `map.rs` |
| N3-C1 | libGDX re-batches visible tiles each frame | D8 | holds | libGDX 1.14.2 |
| N3-C9 | Phaser 4 GPU tile layer: fixed cost per pixel; 4096x4096 limit | D8 | holds | Phaser v4.2.1 |
| N3-C10 | 32-bit indices are core in WebGL2 | D8 | holds | MDN, verbatim |
| N3-C18 | libGDX framebuffers are recreated after context loss | D8 | holds | libGDX 1.14.2 |
| N3-C23 | PixiJS asks for a stencil buffer and warns without one | D8 | holds | PixiJS v8.22.0 |
| N3-C24 | Phaser 4 dropped WebGL geometry masks for a texture-alpha mask | D8 | holds | Phaser v4.2.1 |
| N3-C29 | Riot's fog of war: a 128x128 CPU grid | D8 | holds | Riot article (undated) |
| N3-C31 | Godot: CPU particles can beat GPU particles on low-end devices | D8 | holds | Godot docs |
| N3-C33 | Godot GLES3 GPU particles use transform feedback; disabled on Adreno 3xx | D8 | holds | Godot 4.7.2-stable |
| N3-C37 | Minimum texture units: 8 (WebGL1), 16 (WebGL2) | D8 | holds | WebGL2Fundamentals |
| TX1 | stb_truetype capabilities and its range-checking warning | D9 | holds | `stb_truetype.h` |
| TX2 | LWJGL 3.3.2 added FreeType and HarfBuzz | D9 | holds | LWJGL 3.3.2 notes |
| TX3 | `OffscreenCanvas` 2D widely available since March 2023, in workers | D9 | holds | MDN |
| TX6 | MSDF costs and small-size softness | D9 | holds | Godot font docs |
| IN6 | LWJGL 3.3.4 has experimental GLFW IME support | D10 | holds | LWJGL 3.3.4 notes |
| IN5 | Upstream GLFW IME PR #2130 still open (milestone 3.6) | D10 | holds | GitHub; 3.5 news has no IME |
| IN7 | EditContext is Chromium-only; hidden elements hurt accessibility | D10 | holds | MDN; W3C explainer |
| IN8 | Async clipboard needs a secure context and user activation | D10 | holds | MDN |
| IN1 | Pointer Events model: id, type, primary, pressure, button 0, cancel | D10 | holds | Pointer Events Level 3 |
| IN4 | `KeyboardEvent.code` is layout-independent | D10 | holds | MDN, verbatim |
| AU8 | LWJGL has OpenAL and stb modules, no miniaudio | D11 | holds | `lwjgl3` modules |
| AU7 | Java Sound standard formats; MP3 and Ogg need providers | D11 | **unclear** | Standard-format half sourced; MP3/Ogg half not on the page |
| AU5 | libGDX Android: `SoundPool` and `MediaPlayer`; 1 MB limit; MP3 loop issues | D11 | holds | libGDX source and docs |
| AU1 | Web Audio contexts start suspended without a user gesture | D11 | holds | MDN best practices |
| AU2 | Web Audio state includes `interrupted` | D11 | holds | Web Audio spec |
| AU3 | Safari 18.4 added Ogg Vorbis and Opus | D11 | holds | Safari 18.4 notes |
| AU4 | OpenSL ES not recommended for new designs | D11 | holds | NDK audio guide, verbatim |
| AU9 | OpenAL Soft is LGPL 2 or later | D11 | holds | OpenAL Soft README |
| AU12 | JLayer is LGPL | D11 | holds | Search confirms LGPL; the exact version (2.1+ or 3.0) is unsettled |
| AS3 | libGDX HTML5 freezes on assets not loaded through `AssetManager` | D12 | holds | libGDX docs, verbatim |
| AS1 | Compose resources expose `suspend readBytes` and `getUri` | D12 | holds | Compose resources docs |
| AS4 | kotlinx-io is experimental (0.9.x) | D12 | holds | README; 0.9.1 is latest |
| AS5 | MDN advises handling WebGL context loss | D4, D12 | **fails** | The page only links the events |
| AS9 | Plugin Portal publishing requirements | D12 | **fails** at the cited page | True at plugins.gradle.org/docs/publish-plugin |
| RP1 | libGDX `SpriteBatch` default 1000, max 8191, flushes on texture or full buffer | D13 | holds | `SpriteBatch` source |
| RP6 | Orphaning avoids stalls; persistent mapping needs GL 4.4 | D13 | **unclear** | Khronos wiki would not open |
| N4-22 | Chrome engineer: do not orphan WebGL buffers | D13 | holds | webgl-dev-list, verbatim |
| N4-19 | Kotlin/JS `FloatArray` is a `Float32Array`; `Long` as `BigInt` experimental | D13, D2 | holds | Kotlin JS interop page |
| N4-20 | Kotlin/Wasm arrays do not cross interop directly | D13, D2 | holds | Kotlin Wasm interop page, verbatim |
| N4-1 | PixiJS v8: 100k moving sprites about 15 ms CPU; no hardware named | D13 | holds | PixiJS blog |
| N4-3 | Phaser 4: "tens of thousands of sprites comfortably" | D13 | holds | Phaser README 4.2.1, verbatim |
| N4-4 | 2017 bunnymark: Moto G3 about 17,800 bunnies at 30 fps | D13 | holds | html5gamedevs thread |
| N4-13 | Mali-G52 up to 850 MHz, 6.5 Gpix/s | D13 | holds | CNX Software, 2018 |
| N4-14 | Iris Xe 96EU 31.2 Gpix/s | D13 | holds | askgeek |
| N4-15 | Galaxy A16 5G and A06 are the top sellers; their GPUs and panels | D13 | holds | SamMobile, GSMArena |
| N4-7 | SurfaceFlinger `--timestats` gives a present-to-present histogram | D13 | holds | Android frame-rate page |
| N4-8 | Perfetto FrameTimeline does not support SurfaceViews | D13 | holds | Perfetto docs, verbatim |
| N4-9 | Macrobenchmark `FrameTimingMetric` never mentions GL surfaces | D13 | holds | Macrobenchmark page |
| RP13 | WebGL GPU timer queries disabled in 2018 | D13 | **unclear** | Primary source would not open; current status secondary only |
| RP7 | llvmpipe is OpenGL 4.5 conformant | D14 | holds | Khronos news title, Mesa 20.2 |
| RP8 | OpenGL in CI: xvfb on Linux; Windows needs Mesa | D14 | holds | ModernGL 5.8.2 CI guide |
| RP9 | SwiftShader WebGL needs explicit flags; auto fallback deprecated | D14 | holds | Chromium `swiftshader.md` |
| RP11 | Hardware-accelerated Android emulation on hosted Linux runners since 2024-04-02 | D14 | holds | GitHub changelog |
| CN1 | Browsers have no raw UDP socket; WebTransport is nearest | boundary | holds | Chrome WebTransport page (absence implied) |

#### Tier B (existence check)

All 91 hold.
- **From the reused files:** G7, B2, B3, B11, K2, K6 (the KEEP moved to
  `proposals/stdlib/KEEP-0127-result.md`), R3D7, TX4, TX5, IN2, IN3, IN9, IN11, AS2, AS6, AS7,
  AS10, RP4, RP5.
- **N1** (`web-target-kotlin-js-vs-wasm.md` §7): N1-9, N1-11, N1-20.
- **N2** (`shape-rendering-and-antialiasing.md` §7): items 1, 2, 6, 7, 8, 11, 12, 13, 14, 16,
  17, 18, 19, 20, 21, 22, 24, 26, 27, 28, 29, 30, 31.
- **N3** (`tilemaps-render-targets-and-masks.md` §7): items C2, C5, C6, C7, C8, C11, C12, C13,
  C14, C15, C16, C17, C19, C20, C21, C22, C25, C30, C32, C34, C35, C36.
- **N4** (`android-and-web-performance-targets.md` §7): items 2, 5, 6, 10, 11, 12, 16, 17, 18,
  21, 23, 24, 25, 26.
- **N5** (`android-build-and-platform-floors.md` §7): items 3, 6, 9, 14, 15, 19, 21, 22, 23, 24.

### 2.3 Codebase spot-checks (planner)

Claims about this repo, the monolith client (`SpartanLabsGaming/GameGraphics`, the future
`MyGameClient`) and the sibling repos are not the verifier's job. The planner checked each one
the research leans on, against the code. All hold.

| Claim | Where it is used | Checked against | Result |
|---|---|---|---|
| The monolith draws one `glDrawElements` per actor and looks up five uniforms by name per draw | D13, issue #1 batching | `Window.drawActor` | holds |
| Shaders hard-code `#version 330 core` | D1 shader strategy | shader sources in `Window` and `UiRenderer` | holds |
| `NdcConverter` returns `Pair<Float, Float>` per call | idiomatic-kotlin hot-path note | `NdcConverter` | holds |
| Text uses STBEasyFont with `TEXT_SCALE = 2.0` and a magic `GLYPH_HEIGHT_PX` | D9 | `UiRenderer` | holds |
| `TextureCache` sets the global `stbi_set_flip_vertically_on_load(true)` and falls back to a white texture, logged once | D12 | `TextureCache` | holds |
| `SoundPlayer` creates a JLayer `Player` per `play` on a cached thread pool | D11 | `SoundPlayer` | holds |
| `KeyAction.key` is a raw GLFW key code; repeats are not reported | D10 | `KeyAction` | holds |
| Minimum zoom 0.1; edge-pan threshold 50 px | D8, D15 | `Window` companion constants | holds |
| Monolith build: Kotlin 2.1.20, LWJGL 3.3.4, `jvmToolchain(23)`, JLayer 1.0.1.4, gametools 5.0.0 | D1, D11, D15 | `build.gradle.kts` | holds |
| Untracked sound folders: `ui/` (5 files), `combat/` (3), `ambient/` (2), `atmosphere/` (1), plus three loose files | D11 | working tree | holds |
| Every `Alive` carries a `healthBar` that is a `VisibleObject` sub-object | D13 (10k sprites) | GameTools `Alive` | holds |
| `MapDefinition` has grid size, tile size, flat indices and a terrain palette, with no visual attribute | D8 | GameTools `MapDefinition`, `TerrainTypeSnapshot` | holds |
| GameTools builds on `jvmToolchain(23)` | D15 | `gametools.kotlin-library` convention plugin | holds |
| GameTools milestones 5.2.0 (0 open, 3 closed), 5.3.0 (2 open, 3 closed), 6.0.0 (4 open); none has a due date | D15 | GitHub milestones API | holds |
| GameTools has `ExperimentalGameToolsApi` and `SupportedExtension` annotations, the precedent for this repo's tiers | D4 | `com.spartanlabs.gaming.annotation` | holds |
| The demo server spawns 10 zombies and 1 `Alive` per player, and adds the floor first so clients draw it behind everything | D13, D15 | MyGameServer `Main.kt` | holds |
| SpartanUI holds only a README, a LICENSE and the split plan; no text-field element is planned | D10, D15 | SpartanUI repository tree | holds |
| The split plan's U5 (scrolling) lists only U2 as its dependency | D15 | split plan, "Phase 2: SpartanUI" | holds |

---

## Part 3: Summary

Status: written 2026-10-19, after verification. 118 of the 124 decision-bearing claims hold.
The two failures (AS5, AS9) left the design inputs without reopening any decision, and the four
unclear claims are used below only with their weakness stated. No follow-up research round was
needed. Each decision below names the findings files behind it. Where a recommendation rests on
an outside-world claim, the claim ID points into 2.2.

### 3.1 Decisions, recommendations and evidence

**D1. GPU API per backend, and shaders.**
- **Recommendation:** use the OpenGL family through 1.0 and for every scheduled phase:
  - desktop: OpenGL 3.3 core through LWJGL;
  - Android: OpenGL ES 3.0 through `GLSurfaceView`;
  - web: WebGL2, which is based on ES 3.0.
- **One renderer:** write the batched renderer once, against a thin internal GL seam (the
  libGDX `GL20`/`GL30` model). LWJGL, `android.opengl.GLES30` and `WebGL2RenderingContext` each
  implement the seam.
- **Shaders:** author the built-in shaders once, in a GLSL ES 3.00 subset with a
  per-platform `#version` header, from G5 on.
- **Later backends:** WebGPU, Vulkan or Metal arrive later, as additional backend modules, for
  the 3D track or if Apple removes OpenGL. They never replace the GL backends.
- **Evidence:** `gpu-api-per-platform.md` §3 and §6. Claims G1–G4, G6, G8, G9, G12, G15–G17,
  N5-12, N5-17.
- **Rejected:**
  - WebGPU only: Linux and Android browser gaps, and alpha or Beta bindings on JVM and Android.
  - Vulkan everywhere: one renderer per API, and gaps on Android devices.
  - bgfx or sokol: native glue on two platforms that breaks a pure-Kotlin web target.
- **Remaining risk:** GLES 3.0 and WebGL2 have no compute. That is fine for 2D, and it is a cap
  only for the unscheduled 3D track.

**D2. Web compilation target.**
- **Recommendation:** Kotlin/JS (IR) first, for `graphics-web`, `audio-web` and the web asset
  source.
  - G1's non-JVM compile-check target is `js`, with the default hierarchy template.
  - Web code lives in `webMain`.
  - The web phase adds `wasmJs` as a compile-only tripwire.
  - `wasmJs` is published once Kotlin/Wasm is Stable and `wasmJs` passes the web Level 4c gate.
    That is an additive variant, so it can land before or after 1.0.
  - The roughly 40 WebGL2 entry points are hand-written `internal external` declarations.
  - The vertex upload sits behind one `expect`/`actual` seam.
- **Evidence:** `web-target-kotlin-js-vs-wasm.md` §6. Claims N1-1 to N1-12 and N4-19/N4-20.
- **Rejected:**
  - `wasmJs` only: Beta, a narrower browser floor, and a per-element copy on the hottest path.
  - Both targets from the start: it doubles the gate matrix on Beta tooling for consumers who
    do not exist yet.
  - JS with no tripwire: `webMain` assumptions would go untested until 1.0 had frozen them.

**D3. Android build form and floors.**
- **Module form:**
  - `graphics-android` and `audio-android` are plain `com.android.library` modules: AGP 9
    built-in Kotlin, the new DSL, and a single-variant AAR published through vanniktech.
  - A KMP module with Android source sets (the `assets` module, if the draft's proposal is
    accepted) uses `com.android.kotlin.multiplatform.library`.
  - Pure-common modules have no Android target. Android consumers resolve their `jvm` variant.
  - Android-consumed `jvm` variants compile with `-Xjdk-release`.
  - Only the Android modules need the Android SDK to build.
- **Floors:**
  - minSdk 24 is recommended (Open decision).
  - compileSdk is the newest the pinned AGP supports.
  - Libraries set no targetSdk.
- **Evidence:** `android-build-and-platform-floors.md` §6. Claims N5-1, N5-2, N5-4 to N5-8,
  N5-10 to N5-13 and N5-18.
- **Unproven:** whether Android tooling accepts JVM 21 bytecode. The Android sample app's
  release build is the gate. If it fails, lower only the Android-consumed modules to 17 (Open
  decision, Phase 3).
- **Unclear (N5-4):** that plain `com.android.library` stays supported under AGP 10 is
  inference; the AGP roadmap only confirms that the opt-out flags go. The Android phase confirms
  it, and the fallback is a single-target KMP module on the Android-KMP plugin.
- **Rejected:** `androidTarget()` with `com.android.library`, which is incompatible from AGP 9
  and removed in AGP 10.

**D4. Backend contract.** The frame-level hybrid from `backend-contract-design.md` §6, with
the refinements from N2 and N3.
- **Common code owns** shapes, tessellation, culling, layer and order sorting, clipping,
  outlines, text layout and glyph quads, and batching. It emits a `RenderFrame`:
  - an ordered list of passes;
  - each pass has a target (the surface by default), a viewport and a clear;
  - each batch names a pipeline, a texture binding, an index range in a named vertex buffer, a
    clip rectangle and a blend state.
- **The backend implements:**
  - an open, keyed capability set;
  - a texture store: create from decoded pixels, sub-region update, sampler state, delete;
  - pipeline creation from descriptors that name their vertex layout and blend state;
  - `submit(frame)`;
  - a device-loss epoch;
  - optional sub-interfaces found by typed lookup.
- **Platform SPIs stay small and separate:**
  - `Surface`: pixel size, density, window versus framebuffer size, lifecycle, and a config
    with depth, stencil and MSAA requests;
  - `FrameDriver` with `Clock`;
  - `RenderBackend`;
  - `InputSource`, with text-input, clipboard and cursor services as capabilities;
  - `GlyphRasterizer`;
  - `AssetSource`;
  - the audio backend.
- **Lifecycle:** three channels: app lifecycle, surface lifecycle and GPU device loss. Common
  code keeps a rebuild recipe for every GPU resource. Render-target contents are never
  recoverable after loss.
- **Stable at 1.0:**
  - several vertex layouts per frame (N2);
  - per-layer device-pixel scale available to common code (N2);
  - blend state with separable colour and alpha factors, plus the presets Normal,
    Premultiplied, Additive and Multiply (N3);
  - a pass viewport (N3);
  - sampler state and sub-region texture update (N3);
  - 16- and 32-bit indices (N3);
  - a per-vertex texture-slot field (`render-performance-and-verification.md`, N3);
  - an open `PassTarget`, with only the surface implemented (N3);
  - depth and stencil request fields, default off (N3, `room-for-3d.md`).
- **Experimental at 1.0:**
  - render-target creation, the `Load` op and the `contentsLost` signal;
  - retained geometry;
  - readback;
  - the MSAA request;
  - stencil state in pipelines;
  - every pipeline kind besides the universal one;
  - custom shaders (R7).
- **Not reserved:** transform feedback, compute, GPU particles.
- **Evolution rules:**
  - every new member has a default body, compiled with `-jvm-default=enable` (K3), and on klib
    targets too (N1-18);
  - never add an abstract member to a contract (N1-19, B5);
  - optional sub-interfaces and open capability keys;
  - final descriptor classes with builders, never `data class` or long parameter lists
    (B7, K5);
  - open (not sealed) frame-command and event hierarchies;
  - each backend declares the contract revision it was built for.
- **Tier marking:** in 0.x the SPIs carry `@SubclassOptInRequired(ExperimentalGraphicsApi)`
  (B6, B8). At 1.0 the graduated contracts carry `@SupportedExtension`, as Q1d requires, and
  the guard stays only on Experimental sub-interfaces.
- **Evidence:** `backend-contract-design.md` §6; `shape-rendering-and-antialiasing.md` §6;
  `tilemaps-render-targets-and-masks.md` §6.1; `idiomatic-kotlin.md` §3.3. Claims B1, B4–B8,
  K3, K5, K7, N1-18, N1-19.
- **Rejected:**
  - the split plan's shape-level `Renderer`: every new shape becomes an obligation on every
    backend, and batching cannot be tested without a GPU;
  - a GL-call-level contract: it is GL-shaped, and Godot's `RenderingDevice` is unavailable on
    its own GL renderer (B4);
  - `idiomatic-kotlin.md`'s warning-level marker at 1.0: Q1d settles the graduation.

**D5. Binary-compatibility promise.**
- **Recommendation:** promise JVM and Android binary compatibility within a major.
  - JS (and Wasm, once added) are promised source compatibility within a major, with
    additive-only evolution.
  - That is enforced by klib ABI dumps from `abiValidation()` (from G1) and an old-implementor
    relink fixture.
  - `languageVersion` and `apiVersion` are pinned low.
  - Klib binary compatibility follows the Kotlin toolchain, which has not finalized its
    guarantee.
- **Evidence:** `web-target-kotlin-js-vs-wasm.md` §6 (D5). Claims N1-13 to N1-19.
- **Status:** the wording is an Open decision. The engineering discipline is the same under
  every wording.

**D6. Room for 3D.** Settled by H-A (3.5): at the seams.
- **Recommendation ("at the seams", adopted):**
  - a `Matrix4` type, column-major, with its conventions documented;
  - a per-layer view-projection at the renderer boundary;
  - a thin `Camera` interface (viewport and view-projection) above a concrete `Camera2D`;
  - frames as ordered layers, each with its own camera, clip and viewport (and later a target);
  - an explicit integer order key with a stable insertion tiebreak;
  - depth and stencil request fields;
  - pipeline descriptors that name their vertex layout, so a 3D family declares its own `xyz`
    layout;
  - a `PickHit` result.
- **Strictly 2D:** `Point`, `Size`, `Rect` and every shape. A 3D track is a separate renderable
  family in its own module, behind an opt-in backend capability.
- **Before 1.0:** a throwaway spike records a depth-tested 3D layer through the frozen
  contracts, to find any reservation that is wrong.
- **Evidence:** `room-for-3d.md` §6. Claims R3D1, R3D3–R3D6.
- **Rejected:** z on the basic types or shapes, and per-shape 3D transforms. Bevy documents the
  regret, and Bevy issue #4149 is still open (R3D5, R3D6).

**D7. Shapes (R1, R2).**
- **Technique:** CPU tessellation in common code, at the layer's device-pixel scale, with a
  feathered antialiasing fringe in vertex alpha, all through the universal pipeline. Shapes
  sample a white texel, so they batch with sprites.
- **Strokes:** one stroker serves lines and every outline kind, including outlines that rotate
  with their shape and elliptical outlines.
- **Polygons:** ear clipping for simple concave polygons.
- **Gradients:** vertex colours. Linear gradients are exact; radial gradients are exact on
  circles and ellipses.
- **9-slice:** nine quads on a shared vertex grid, using the texture source rectangle.
- **Later options:** MSAA is an optional Experimental request. SDF pipelines are added only if
  the Level 4c gate demands them.
- **Landing order:**
  1. 9-slice and linear gradients;
  2. the tessellator, with rounded-rectangle fill and outline;
  3. lines, circles and polygons;
  4. the R2 outlines, from the stroker;
  5. radial gradients.
- **Evidence:** `shape-rendering-and-antialiasing.md` §6. Claims N2-3 to N2-5, N2-9, N2-10,
  N2-15, N2-23, N2-25.
- **Rejected:** MSAA as the baseline, SDF primitives first, stencil-then-cover.

**D8. Atlases, tile maps, render targets, masks, particles (R3–R6).**
- **R3:** atlases and animation are common code. A runtime packer covers lazily loaded,
  server-named textures.
- **R4 tier 1:** a camera-independent CPU chunk cache, copied into the frame stream. No contract
  change.
- **R4 tier 2:** retained GPU geometry, as an Experimental extension, only if a measured 4c tile
  scene fails.
- **R5:**
  - The visibility mask is a CPU-built low-resolution texture, composited by blending. It needs
    no stencil.
  - A minimap is a second layer with its own viewport, or an optional render target.
  - Render-target creation is Experimental at 1.0. R5 is the one post-1.0 phase that adds
    backend obligations, so it ships a render-target conformance suite.
- **R6:** CPU particles, emitted into the batch.
- **Order:** R3, then R4 tier 1, then R5, then R6, then R7. R4 tier 2 comes in only on a
  measured need.
- **Evidence:** `tilemaps-render-targets-and-masks.md` §6. Claims N3-C1, C3, C4, C9, C10, C18,
  C23, C24, C26–C29, C31, C33, C37.
- **Rejected:**
  - stencil masks: binary, and off by default on Android and in WebGL (N3-C26, N3-C27);
  - GPU particles: transform feedback is GL-only and fragile on devices (N3-C33);
  - a data-texture tile layer before R7.

**D9. Text.**
- **Layout in common code:**
  - font-table reading;
  - a simple left-to-right shaper with kerning;
  - greedy wrapping, with a replaceable line breaker;
  - a glyph atlas with LRU pages.
- **Rasterising per backend:** stb_truetype on desktop, `Paint`/`Canvas` on Android,
  `OffscreenCanvas` on web.
- **Glyph modes:** raster by default; SDF opt-in for world-layer text that zooms. MSDF baking
  comes later, through the Gradle plugin.
- **At 1.0:**
  - `TextShaper` is Experimental, with HarfBuzz to follow for T2;
  - `Font`, `TextStyle`, `TextLayout` and `TextMeasurer` are frozen.
- **Phase-1 contract recommendation:** `Text` takes a `TextStyle` and an opaque `Font`, and
  `TextMeasurer` returns a layout result that separates the line box from the ink bounds.
- **Evidence:** `text-rendering.md` §6. Claims TX1–TX3, TX6.
- **Android note:** loading a font from bytes is API 29+; on API 24–28 the backend goes through
  a temporary file (`android-build-and-platform-floors.md` F6).

**D10. Input.**
- **Pointer:** a pointer model shaped like W3C Pointer Events from 0.1.0: id, kind, a phase
  that includes Cancel, primary flag, pressure, modifiers.
- **Keys:** physical keys named by W3C `KeyboardEvent.code` strings, never GLFW integers, with
  a repeat flag.
- **Delivery:** events queue up and the game drains them once per frame on the frame thread.
  The event hierarchy is open. An `InputTracker` in common code synthesises releases on focus
  loss.
- **Text input:** a state-mirroring `TextInputSession`.
  - Phase 2: Experimental, on desktop, through LWJGL's experimental IME support with a
    char-only fallback.
  - Phases 3 and 4: proven by Android's `InputConnection` and the web's hidden text area.
  - 1.0: frozen.
- **Clipboard:** asynchronous and `Result`-returning.
- **Cursors:** a cursor service per backend (desktop in Phase 2, Android `PointerIcon` in
  Phase 3, CSS in Phase 4).
- **Gestures:** a pure-Kotlin recogniser, Experimental, arriving with Android.
- **Gamepad (I2):** later, as new events plus a capability.
- **Evidence:** `cross-platform-input.md` §6. Claims IN1, IN4–IN8, N5-13.

**D11. Audio.** Timing settled by H-B (3.5): A1 and the A2 model in Phase 2 on OpenAL; Android
and web audio in Phases 3 and 4; A3 after 1.0.
- **App-facing contract:** `AudioEngine`, `Clip`, `StreamSource`, a generation-checked `Voice`
  handle, an `AudioBus` tree and a `Listener`.
- **Policy in common code:** bus gains, voice limits, fades and crossfade, ducking rules, and
  positional gain and pan.
- **Backend:** receives per-voice gain, pan, rate and loop. Lifecycle state and the web autoplay
  unlock belong in the contract.
- **Backends:**
  - desktop: the JLayer port at 0.1.0, then OpenAL Soft through LWJGL with `stb_vorbis`, keeping
    JLayer for MP3 decoding;
  - Android: `SoundPool` and `MediaPlayer`, with audio focus;
  - web: the Web Audio graph.
- **Asset needs:** audio needs streaming and locator capabilities on `AssetSource`.
- **Licences:** OpenAL Soft and JLayer are LGPL, which the README and NOTICE must state.
- **Evidence:** `cross-platform-audio.md` §6; `consumer-needs-and-dependencies.md` §3.4. Claims
  AU1–AU5, AU8, AU9, AU12, N5-16. AU7 is only half-sourced (Java Sound's standard formats are
  WAV, AIFF and AU), which is enough for the desktop decoder choice. JLayer's exact LGPL version
  (2.1+ or 3.0) is unsettled (AU12), which the licence check before publishing must settle.

**D12. Assets.**
- **Contract:** `AssetSource` is a minimal `suspend fun read(name): Result<ByteArray>` with:
  - a written name grammar that reserves `#`, `?` and `:`;
  - an open failure hierarchy;
  - memory, prefix and composite sources.
- **Capabilities:** streaming, locator, listing and change notification are separate interfaces.
- **Textures:** created from decoded pixels, held as opaque tokens, with a rebuild recipe kept.
- **S1:** the asset manager is post-1.0, in its own module with no graphics dependency.
- **Hot reload:** Experimental, desktop only.
- **S2, the Gradle plugin:** a separate included build. Its tasks land in this order: manifest,
  atlas, fonts, compression.
- **Evidence:** `asset-pipeline.md` §6. Claims AS1, AS3, AS4, K9, G9, and N3-C26 for web
  context loss. AS5 failed and left the inputs. AS9 holds only at the Plugin Portal's own page.
- **Open:** the module form (the draft's `assets` versus the split plan's `assets-core`) and the
  plugin's publishing route.

**D13. Batching, the Level 4c gate, and the targets.**
- **Batching:** vertex-expanded quads, built in common code. A batch flushes on texture, blend,
  clip, shader or capacity change; capacity is a parameter; draw order is preserved; atlases
  keep batches long.
- **Uploads:** desktop and Android may orphan or ring their buffers. Web uses `bufferSubData`
  into a pre-sized buffer or a small ring (N4-22).
- **The gate has two halves:**
  - CI structural counters on the recording backend: draw calls, flushes by reason, bytes
    uploaded, and steady-state allocation of about zero.
  - Release timing on named reference hardware: p95 and p99, after a warm-up, on a seeded
    scene, plus an uncapped pass.
- **Unit:** state every target as sprites plus covered area in megapixels (N4).
- **Targets:**
  - Desktop: 10k textured quads of about 32 px, about 10 Mpx covered at 1080p (H-D, 3.5).
  - Android: 5k sprites as the hard gate and 10k as the target, promoted once a first
    measurement shows at least 30% headroom. Reference class: Mali-G57/G68 or Adreno 619 at
    1080p. Floor: Mali-G52 at 720p, report-only.
  - Web: 10k sprites on an Iris Xe-class laptop in Chrome stable. Firefox and Safari are
    report-only.
- **Android timing:** `dumpsys SurfaceFlinger --timestats` (N4-7 to N4-9).
- **Evidence:** `render-performance-and-verification.md` §6;
  `android-and-web-performance-targets.md` §6. Claims RP1, N4-1, N4-3, N4-4, N4-13 to N4-15,
  N4-22. RP6 (persistent mapping needs GL 4.4) and RP13 (browser GPU timers) are unclear; the
  design depends on neither.
- **Confidence:** medium-low on Android and medium on web. No named-device 2025–2026 benchmark
  exists, so the first sample-app measurement ratifies both.

**D14. Verification without a GPU.** Settled by H-C (3.5): Linux runners may host the official
platforms' tests.
- **Level 2 and 4a:** the recording backend.
- **Level 3 and 4b:** software rasterisers: llvmpipe under Xvfb for the GL renderer, headless
  Chromium on SwiftShader for web, and the Android emulator nightly.
- **Goldens:** generated on the CI rasteriser, compared with a tolerance.
- **Sample apps:** each runs a shared, seeded reference scene in capture or benchmark mode.
- **Timing:** only on reference hardware.
- **Evidence:** `render-performance-and-verification.md` §3.4 and §6. Claims RP7–RP9, RP11.

**D15. Consumers.**
- SpartanUI 0.1.0 is fully served by Phase 1.
- **Phase-1 seams to get right:**
  - nested, intersecting clips;
  - pointer events that carry an id and a kind;
  - an explicit order key, plus a texture source rectangle;
  - a density factor on `Surface`, by Phase 3 at the latest.
- **Split plan correction:** U5 depends on U3 and G4, not only U2.
- **Cadence:** because of `api(graphics-core)`, every breaking graphics 0.x boundary pairs with
  a SpartanUI minor.
- **GameTools-gated client features** (tiled terrain, fog and minimap, event-driven positional
  audio, ability and item UI) are post-1.0. Their data-shape assumptions:
  - map visuals are supplied by the client;
  - the visibility mask is built by the consumer;
  - events supply positions.
- **Android and web** have no consumer beyond the sample apps. A real client port needs a
  multiplatform GameTools protocol and a browser transport in WebTools (CN1). Both are
  unscheduled.
- **4c gate:** the client cannot prove it; it is a 10-zombie demo.
- **Phase 2 text input** has no consumer in today's plans. SpartanUI plans no text field
  (verified). The user's order stands.
- **Evidence:** `consumer-needs-and-dependencies.md` §6 and §7; 2.3.

**D16 and D17.** Post-1.0 phase grouping and the final module layout are stage-3 judgement.
- **Phase-grouping inputs:**
  - the landing orders in D7 and D8;
  - the audio answer (H-B);
  - the asset-pipeline order;
  - SpartanUI's needs first.
- **Module-layout inputs:**
  - where the shared GL renderer lives (D1);
  - the per-device-family rule;
  - the unpublished sample apps;
  - the plugin's included build;
  - the draft architecture's proposals, which stay Open decisions.

### 3.2 Contradictions resolved

| Contradiction | Resolution |
|---|---|
| `backend-contract-design.md`, `render-performance-and-verification.md` and N2 reserve `z` in the universal vertex layout; `room-for-3d.md` says reserve at the seams, not in the types | N2's own finding settles it: pipeline descriptors name their vertex layout and a frame carries several layouts, so a 3D family declares its own `xyz` layout. The universal 2D layout stays `xy`, which saves about 20% of vertex bytes. H-A adopted the seams reading, so this holds. |
| Orphaning buffers (render-performance F4, Khronos) versus not orphaning on web (N4, a Chrome engineer) | Backend-specific: desktop and Android may orphan or ring; web uses `bufferSubData`. Not a contract matter. |
| Android frame timing through Macrobenchmark (render-performance) versus SurfaceFlinger timestats (N4) | Timestats plus in-app CPU time. `FrameTimingMetric` only after a spike shows it sees a GL surface. |
| Scene profile "32–64 px, at most 4x overdraw" (render-performance) is internally inconsistent (N4) | Gates state sprites plus covered area. The desktop profile was settled by H-D (3.5): 10k quads of about 32 px. |
| Android and web targets: web gated at 5k (render-performance) versus 10k (N4) | N4's figures, which are newer, name device classes and state their confidence. |
| Audio control after 1.0 (Q3b) versus a frozen audio contract at 1.0 (Q1d); the audio research wants the engine before Android | Not resolvable by research; resolved by H-B (3.5): A1 and the A2 model move into Phase 2. |
| `@SubclassOptInRequired` at 1.0: graduate (`backend-contract-design.md`, the draft) versus keep a warning-level marker (`idiomatic-kotlin.md`) | Q1d is binding: graduated contracts carry `@SupportedExtension`. |
| Stencil "has a 2D consumer today" (`room-for-3d.md`) versus no R3–R6 feature needs stencil (N3) | N3 holds: the mask is texture-based. Keep the request field (Stable, default off) and stencil pipeline state (Experimental), motivated by a future rounded-corner clip in SpartanUI (N2). |
| `consumer-needs-and-dependencies.md` treats "libraries never depend on GeneralTools" as decided | Superseded: by the user's assumption, GeneralTools#7 is an open decision. |
| `asset-pipeline.md` exempts the Gradle plugin from the one-version rule | The user's assumption (one shared version for all artifacts) holds. The cost is republishing an unchanged plugin at each release, which is cheap with manual releases; the versioned file formats still decouple plugin and runtime. |

### 3.3 Decisions for the user (the stage-2 halt)

Research changed the picture in four places. Each is either a reading of an interview answer
that research forces, or a new decision.

- **H-A. Q4a: where the room for 3D is reserved.**
  - *Seams* (recommended): the reservations listed in D6, with shapes and basic types strictly
    2D. They land in G2–G5 through an issue #1 scope change, as the clip did.
  - *Types:* shapes carry a depth coordinate and a 3D-capable transform from the start. That
    is the Bevy and Unity model, whose regrets are documented, and it costs about 20% more
    vertex bytes.
- **H-B. Q1d versus Q3b for audio.** The 1.0 freeze includes the audio backend contract, but
  A1–A3 come after 1.0.
  - *(a)* Pull A1 and the A2 model (voices, buses, streaming, OpenAL) forward, before Android,
    so 1.0 freezes an exercised contract (recommended).
  - *(b)* Freeze a minimal audio contract and add control later through optional
    sub-interfaces.
  - *(c)* Leave audio out of the 1.0 freeze.
- **H-C. Q3a: what "compile-only CI" covers.** May Linux runners host:
  - the Android-emulator and headless-browser tests of the official Android and web backends;
  - llvmpipe pixel tests of the shared GL renderer, while Linux desktop stays "should work,
    untested"?

  Recommended: both. Hosted Linux runners are where GitHub documents hardware-accelerated
  Android emulation (RP11). Without them, Android rendering verification falls back to manual
  runs or unaccelerated emulators.
- **H-D. Q4b: what "10k sprites" counts, and at what size.**
  - *Quads* (recommended): 10k textured quads of about 32 px, about 10 Mpx covered at 1080p.
  - *Game entities as the client draws them:* each `Alive` adds a health-bar quad, so about 20k
    quads.

  At 64 px the gate becomes a fill-rate test (about 41 Mpx).

The user answered on 2026-10-19. The answers are in 3.5.

### 3.4 Open decisions carried to the roadmap (they do not block stage 3)

- **The draft architecture's proposals**, kept open by the user's assumption:
  - the `assets` module with its loaders inside;
  - `graphics-test`;
  - `graphics-bom`;
  - the guarded SPIs;
  - test packages by level;
  - the artifact-name prefix;
  - basic types as final immutable classes rather than interfaces.
- **Phase-1 contract shapes to raise on issue #1** before G2 starts:
  - the frame-level `RenderBackend`;
  - layers with an order key;
  - nested clips;
  - the texture source rectangle;
  - Pointer-Events-shaped input with `code`-named keys;
  - the `TextStyle`/`Font`/`TextMeasurer` shape;
  - a `suspend` `AssetSource`;
  - a `Voice` handle at G6;
  - density on `Surface`.
- **GeneralTools#7:** whether the libraries ever take GeneralTools types.
- **D5:** the wording of the 1.0 compatibility promise.
- **D3:** minSdk, and the bytecode level of Android-consumed modules.
- **The Gradle plugin's publishing route:** Plugin Portal or Central only.
- **Reference hardware** for the timing half of the 4c gate.
- **When to publish `wasmJs`:** before or after 1.0, once Kotlin/Wasm is Stable.

*(2026-10-20:)* The roadmap's section 5 now records these decisions as they are settled. Two
answers there depart from this research's recommendation:
- the basic types are multiplatform GeneralTools 3.x's, used directly, rather than final classes
  owned by the library (decisions H and N);
- `assets-core` keeps the asset contracts, with the platform loaders in the graphics backends,
  rather than one `assets` module (decision B).

The roadmap records the risks of the GeneralTools types.

### 3.5 Answers at the stage-2 halt (binding)

The user's answers of 2026-10-19, one line each. They bind stage 3 exactly as Part 1.1 does.

| ID | Constraint |
|----|------------|
| H-A | Room for 3D is reserved at the renderer and camera boundary: a 4x4 matrix with a per-layer view-projection; a thin `Camera` interface over `Camera2D`; frames as ordered layers, each with its own camera, clip and viewport; an integer order key instead of z; depth and stencil request fields; pipeline descriptors that name their vertex layout; `PickHit`. Shapes, `Point`, `Size` and `Rect` stay strictly 2D, and a pre-1.0 spike drives a depth-tested 3D layer through the frozen contracts. |
| H-A (process) | The scope-change comment on issue #1 for G2–G5 is not approved for posting. Its proposed text goes to the user in the planner's report. Until it is approved, #1's stage list stays as it is. |
| H-B | A1 and the A2 model (buses, streamed music, crossfade) move ahead of Android, on an OpenAL desktop backend, as an independent audio track in Phase 2. `audio-android` and `audio-web` are written once, in Phases 3 and 4. A3 stays after 1.0. |
| H-C | Linux runners may host the official platforms' tests: the Android emulator, headless Chromium, and llvmpipe pixel tests of the shared GL renderer. The Linux desktop backend itself (windowing, input, packaging) stays "should work, untested". |
| H-D | The desktop gate is 10k textured quads of about 32 px: single atlas, alpha-blended, moving and rotating, about 10 Mpx covered at 1080p. Every gate is stated as sprites plus covered area. |
| Targets | The Android and web targets in D13 stand as research-set targets, to be confirmed by the first sample-app measurement. |

Q3b's order is unchanged.
