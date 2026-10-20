# Idiomatic Kotlin for SpartanGraphics: what fits, what is cargo-culted

## 1. Question

Which Kotlin language constructs genuinely suit a multiplatform 2D graphics library with frozen backend contracts, and which would be cargo-culted? The library targets JVM, Android and web, with room for 3D, about 10k sprites per frame, and `Result`/KDoc/OOP+FP house rules.

## 2. Bottom line

- **Basic types (`Point`, `Size`, `Rect`, `Color`, later `Vec3`/`Matrix`): final, immutable, non-`data` classes with explicit `equals`/`hashCode`. Not interfaces, not Compose-style packed value classes.**
  - A value class's packed representation is public ABI. `Offset` exposes `packedValue: Long` in its constructor.
  - Value classes box in generics, nullables, interfaces and collections.
  - On Kotlin/JS a `Long` is an emulated object, so packing is slow there.
  - Only 2-float types fit a `Long`. `Rect`, `Vec3` and matrices cannot, so a value-class design splits into two tiers.
  - `data` classes freeze `copy` and `componentN`.
  - Interfaces cannot promise immutability and add virtual dispatch.
  - Hot paths get **scalar-first overloads** and renderer-internal buffers, so the object types never sit in the per-sprite loop.
- **One exception worth considering: open-ended id sets (`Key`, `PointerButton`) as `Int`-backed value classes with companion constants.** They are one field, `Int` is native on every target, and the set can grow without breaking `when`. Avoid `enum` for sets that will grow.
- **Hide mutable math.** Mutable twins (Kool's `MutableVec2f`) double the frozen surface. Immutable public types plus mutable scalar state inside shapes and the renderer is enough. Operators on immutable types are fine in app code; they allocate and must not appear in renderer inner loops.
- **Closed-but-not-sealed shapes.** Use a `public abstract class Shape internal constructor()` (no outside subclassing, not `sealed`), with dispatch to the renderer inside core. Then adding shapes after 1.0 is not a source break for third-party backends' exhaustive `when`. Use `sealed` only for hierarchies that really are final.
- **Evolving frozen backend contracts.**
  - Add members only with default bodies.
  - Compile with `-jvm-default=enable` (the default from language version 2.2), and never use `no-compatibility` or `@JvmDefaultWithoutCompatibility` on an existing contract.
  - Prefer new optional capability interfaces discovered with `as?`. They are additive on every target.
  - I found no documented guarantee for default-method additions on klib targets (JS, Wasm, Native). Back it with a CI old-implementor fixture.
- **Concurrency.**
  - `suspend` for asset loading.
  - Push callbacks for frames and input at the contract level.
  - `Flow` and `withFrameNanos`-style suspend adapters are extensions on top.
  - Never `runCatching` around suspend code without rethrowing `CancellationException`.
  - There is no `runBlocking` on JS or Wasm.
- **Failures.** Use `Result<T>` as your rules say, but make the failure types a small sealed exception hierarchy. The Kotlin KEEP itself says `Result` is not designed for domain-specific errors and recommends sealed classes where callers must tell failure kinds apart. GPU resources are `AutoCloseable` (common stdlib since 2.0), with idempotent `close()`.

## 3. Findings

### 3.1 Value classes

- **Boxing rules (established, language docs).** A value class is unboxed only when used as itself. It is boxed when used as a generic `T`, as an interface type, or as a nullable `Foo?`. Referential equality is prohibited. Functions that take value classes get hash-mangled names on the JVM. `@JvmInline` is needed on the JVM backend. A value class has a single property, is always final, and cannot extend classes. [Kotlin docs: inline classes](https://kotlinlang.org/docs/inline-classes.html)
- **Compose precedent (common practice, source code).** `Offset` is `value class Offset(val packedValue: Long)`.
  - Operators unpack, compute per lane and repack. `unaryMinus` skips the unpack (`xor` of the sign bits).
  - The KDoc says the constructor exists for "arrays or collections of primitives without boxing". That means users store the packed `Long` in a `LongArray` themselves.
  - `Unspecified` is a NaN-bit sentinel, and reading `x`/`y` on it throws. [Offset.kt](https://raw.githubusercontent.com/androidx/androidx/androidx-main/compose/ui/ui-geometry/src/commonMain/kotlin/androidx/compose/ui/geometry/Offset.kt)
  - A Kotlin Slack report found an array of `Offset` autoboxed, with the advice to keep a `LongArray`. [thread](https://slack-chats.kotlinlang.org/t/32847721/class-particles-val-positions-array-lt-androidx-compose-ui-g) (one source's anecdote)
- **`Long` on Kotlin/JS (common knowledge across several sources, none official).** `Long` is emulated by an object, unlike `Int`. A paper on JS 64-bit ints reports one to two orders of magnitude slowdown against `Int`. [Slack](https://slack-chats.kotlinlang.org/t/8192771/can-anyone-explain-why-long-math-is-extremely-slow-compared-), [ECOOP'18 paper](https://lively-kernel.org/lively4S2/testdir/ecoop18-paper22.pdf). KorGE's author notes the same: packing into a `Long` on JS "still requires allocation". [soywiz](https://soywiz.com/kotlin-allocation-free-vectorial-operations-using-custom-operators). A packed-`Long` design is therefore only safe if G1 picks `wasmJs`, and even then I could not verify the Wasm `Long`/`i64` mapping this session (see Gaps).
- **Kotlin/JS `Float` is not a 32-bit float (one source).** It is backed by a JS `Number`, and bit conversions can lose accuracy. [Slack](https://slack-chats.kotlinlang.org/t/2417895/hello-is-there-some-multiplatform-library-to-consistently-re), [bits docs](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/to-bits.html). Cross-target bit-exact camera/picking maths is therefore not promised. Level 4a tests need tolerances.
- **Multi-field value classes are not available (unresolved).** They are KEEP-0454 under the "Value Classes 2.0" effort. I could not confirm any shipped or preview status in 2.2, 2.3 or 2.4. The 2.4.0 notes mention only exporting `@JvmInline value class` to JS/TS. [KEEP notes](https://github.com/Kotlin/KEEP/blob/main/notes/value-classes.md) (not fetchable), [What's new 2.4](https://kotlinlang.org/docs/whatsnew24.html). A design that depends on them is a bet.
- **`data` classes in public API (established, Kotlin's own library-author guidance).** Adding a property changes the constructor and `copy` signatures, and reordering changes `componentN`. "In API design, this strength becomes a weakness." [API guidelines: backward compatibility](https://kotlinlang.org/docs/api-guidelines-backward-compatibility.html)
- **Interfaces for basic types (reasoned, plus the draft's own point).** The architecture draft notes an interface cannot guarantee immutability, so a renderer could see a consumer's `Point` change mid-frame. [architecture.md §7.2](../../plans/1-phase-1-spartangraphics/architecture.md)
  - Add the open equality contract (`Point` equal across implementations) and virtual calls in the hot path.
  - The split plan's real motive was adapting GeneralTools/GameTools types. Conversion functions in the client do that without any interface.

### 3.2 Mutable and immutable math in hot paths

- **Kool (common practice).** `open class Vec2f(open val x, open val y)`, with `MutableVec2f` as a subclass. Operators (`plus`, `times`) allocate. There are allocation-free `add(that, result)` and in-place `MutableVec2f.add`. `lerp(…, result = MutableVec2f())` allocates by default. [Vec2.kt](https://raw.githubusercontent.com/kool-engine/kool/main/kool-core/src/commonMain/kotlin/de/fabmax/kool/math/Vec2.kt)
- **KorGE/Korma (one author's account).** The author's earlier design was an immutable vector interface plus a mutable implementation. A KorGE user warns the mutable `Point` is bad for Kotlin/Native performance. The value-class plan aimed at "performance without mutable, error-prone APIs". The author also tried a scoped, pre-allocated pool behind operators and admits the lifetime rule is "not enforced at compile time, in the editor, or at runtime". [2018 post](https://soywiz.com/korma-1-0/), [value classes plan](https://soywiz.com/evolution-of-kotlin-and-korge-(value-classes)/), [pool trick](https://soywiz.com/kotlin-allocation-free-vectorial-operations-using-custom-operators). That pool trick is a cautionary example, not a pattern to copy.
- **libGDX and jMonkeyEngine.** libGDX documents pooling for spawned objects. jME offers `TempVars` and in-place `*Local` variants, with the warning that they overwrite values you still need. [libGDX memory management](https://libgdx.com/wiki/articles/memory-management), [jME TempVars](https://javadoc.jmonkeyengine.org/v3.6.0-beta1/com/jme3/util/TempVars.html)
- **This repo today.** `NdcConverter.offset`/`halfSize` return `Pair<Float, Float>` (`NdcConverter.kt:55-58, 107-108`). `Window.kt:616-631` destructures two of them per actor per frame, then issues one `glDrawElements` per actor. That is about two `Pair` objects plus four boxed `Float`s per sprite per frame, at the scale where the 10k target bites. The ported renderer should write straight into a vertex buffer from scalars.
- **ART and WasmGC allocation cost: not established.** A Slack remark says allocation in Compose draw paths is a Dalvik-era worry. That is anecdote, not measurement. Do not claim a number for ART or WasmGC.

### 3.3 Closed and extensible types

- **`@SubclassOptInRequired` (established, stdlib).** It applies to open/abstract classes and non-`fun` interfaces. Its stated purpose is to let authors add abstract members later without breaking outside implementers. Opt-in propagates via the marker, `@OptIn` (does not propagate) or `@SubclassOptInRequired` (propagates for subclassing only). [stdlib docs](https://kotlinlang.org/api/kotlin-stdlib/kotlin/-subclass-opt-in-required/index.html)
  - **Tension with decision 4.** After 1.0 the backend contracts carry Stable Core's semver promise, which a "may break implementers" marker contradicts. I recommend keeping a *warning-level* marker as a deliberate-implementer acknowledgement, not an experimental disclaimer. I recall kotlinx.coroutines doing this (`ExperimentalForInheritanceCoroutinesApi`) but could not re-verify it this session.
- **JVM default methods (established, Kotlin docs).**
  - `-jvm-default=enable` is the default since language version 2.2. It generates JVM default methods and `DefaultImpls` bridges, so it "maintains compatibility with older compiled Kotlin code".
  - `no-compatibility` omits `DefaultImpls` and "can break binary compatibility with older Kotlin code".
  - `@JvmDefaultWithoutCompatibility` on an existing class is binary-incompatible.
  - [Java interop docs](https://kotlinlang.org/docs/java-to-kotlin-interop.html), [option reference](https://kotlinlang.org/api/kotlin-gradle-plugin/kotlin-gradle-plugin-api/org.jetbrains.kotlin.gradle.dsl/-kotlin-jvm-compiler-options/jvm-default.html)
  - Adding a default-bodied method is the safe evolution under `enable`. This is also how Android compiles.
- **klib targets (JS, Wasm, Native): no authoritative statement found.**
  - The Kotlin ABI-validator design doc says klib compatibility guarantees were "not yet finalized" and klib validation is experimental. [KLibSupport.md](https://github.com/Kotlin/binary-compatibility-validator/blob/master/docs/design/KLibSupport.md)
  - Indirect evidence: fake overrides are not serialized in klibs and are rebuilt by the IR linker after deserialization. That suggests re-linking against a newer interface picks up a new default method. [KotlinAbiVersionBumpHistory](https://pigweed.googlesource.com/third_party/github/JetBrains/kotlin/+/b1f59e5f0d739fd4c19f9e413cc9c324657eafed/compiler/util-klib/KotlinAbiVersionBumpHistory.md). This is inference only.
  - Hence the CI fixture recommendation in §6.
- **ABI tooling.** The Kotlin Gradle plugin has built-in binary-compatibility validation from 2.2.0. Its DSL is still behind `ExperimentalAbiValidation`, and the 2.4.0 notes say it was streamlined with parts deprecated. [docs](https://kotlinlang.org/docs/gradle-binary-compatibility-validation.html), [2.4 notes](https://kotlinlang.org/docs/whatsnew24.html)
- **`inline` is ABI.** Public inline bodies are copied into callers. The `@PublishedApi` internals they call "must be treated the same as public declarations". [API guidelines](https://kotlinlang.org/docs/api-guidelines-backward-compatibility.html)
- **Sealed versus closed-not-sealed (reasoned from language rules, no source).**
  - A `sealed` hierarchy makes consumer `when` exhaustive, so a new subtype breaks every third-party backend that switches over it. Decision 10 schedules new shapes after 1.0.
  - `abstract class Shape internal constructor()` is closed to outside subclassing and not exhaustive, and the renderer contract receives draw commands, not a `when` over `Shape`.
  - Whether the contract takes shapes or lower-level draw commands is Q5's lane.
- **`expect`/`actual` versus interface (reasoned, matches the split plan).**
  - With one module per backend, platform services are interfaces with per-backend implementations. That keeps substitution, faking (`graphics-test`) and the "interface plus default" rule.
  - Reserve `expect`/`actual` for tiny internal glue inside a module (clock source, logger factory).
  - I recall `expect`/`actual` classes being flagged Beta by the compiler but did not re-verify.

### 3.4 Coroutines

- **`runBlocking` does not exist on JS/Wasm** (a maintainer: "impossible to implement on JS … inability to advance the current event loop"). A Ktor issue (2026) hits the same gap. A blocking `AssetSource` adapter can only be JVM-only. [Slack](https://slack-chats.kotlinlang.org/t/16176091/is-runblocking-going-to-be-supported-in-the-js-target), [issue #4239](https://redirect.github.com/Kotlin/kotlinx.coroutines/issues/4239) (common practice)
- **`CancellationException` swallowing (consensus across lint tools and write-ups).** `runCatching` catches `Throwable`, so cancellation becomes `Result.failure`. Fix with a helper that rethrows. [DeepSource KT-W1066](https://deepsource.com/directory/kotlin/issues/KT-W1066), [ProAndroidDev](https://proandroiddev.com/kotlin-coroutines-anti-patterns-in-production-5-subtle-traps-that-leak-memory-and-break-41bde8232b19). The monolith uses `runCatching` throughout (`Window.kt`, `TextureCache.kt:49`, `SoundPlayer.kt:35`). That is fine while blocking and wrong once `AssetSource` is `suspend`.
- **Frame clock precedent (common practice).** Compose's `MonotonicFrameClock.withFrameNanos` suspends until the next frame and passes a strictly increasing time. The Android implementation forwards the Choreographer time to all awaiters. `BroadcastFrameClock.sendFrame(timeNanos)` plus `onNewAwaiters` starts the host frame source only while someone waits. [MonotonicFrameClock](https://composables.com/docs/androidx.compose.runtime/runtime/interfaces/MonotonicFrameClock), [BroadcastFrameClock](https://composables.com/docs/androidx.compose.runtime/runtime/classes/BroadcastFrameClock)
  - That is the shape of a *suspend adapter over a push driver*, which fits the split plan's "platform calls `onFrame(time)`".
- **`SharedFlow` for input (reasoned from documented semantics).** `tryEmit` returns false when emit would suspend. Behaviour with no collectors and with `DROP_OLDEST` is per the constructor docs; I did not retrieve that page, and my summary of it is general knowledge. A lossy or conflated input flow can drop a key-up, leaving a stuck key. Delivery order relative to the frame also becomes dispatcher-dependent. [MutableSharedFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-mutable-shared-flow/)
- **Dispatchers on single-threaded targets.** One library reports using `Dispatchers.Default` where JVM code would use `Dispatchers.IO` on JS/Wasm. [klibs.io](https://klibs.io/project/codinux-gmbh/CoroutineExtensions). I did not verify `Dispatchers.Main` on Wasm.

### 3.5 Failures and lifetimes

- **`Result` (established, KEEP).** It is for capturing outcomes of arbitrary code. Errors are always `Throwable`. It is "not designed to represent domain-specific error conditions". Where callers must distinguish kinds, use sealed classes. Dropping a `Result` silently loses the exception (an open issue). [KEEP](https://github.com/Kotlin/KEEP/blob/master/proposals/stdlib/result.md)
  - Kotlin 2.3 adds an *experimental* unused-return-value checker (`@MustUseReturnValues`) that would catch dropped `Result`s. I did not confirm its 2.4 status. [What's new 2.3](https://kotlinlang.org/docs/whatsnew23.html)
  - Result-returning functions look like throwing functions to Java. [Kotlin forum](https://discuss.kotlinlang.org/t/kotlin-result-as-return-type-in-function-literal/20231) (one source)
- **`AutoCloseable` and `use` are common stdlib since 2.0**, with a common `AutoCloseable(closeAction)` factory. Implementers should make the close action idempotent. [AutoCloseable](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/-auto-closeable.html), [use](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/use.html)
- **Context parameters** are Stable only from Kotlin 2.4.0 (14 July 2026). Experimental behind `-Xcontext-parameters` in 2.2/2.3, and 2.4 adds explicit context arguments. [What's new 2.4](https://kotlinlang.org/docs/whatsnew24.html). Putting them in a frozen public signature would force consumers onto 2.4+.

### 3.6 Logging

- kotlin-logging is lazy-lambda based. On the JVM it needs an slf4j implementation supplied by the application. [README](https://github.com/oshai/kotlin-logging). A `kotlin-logging-wasm-js` artifact exists (listed at 8.0.4, [Sonatype](https://central.sonatype.com/artifact/io.github.oshai/kotlin-logging-wasm-js/)).
- I could not confirm what the Android or Wasm backends write to, or the multiplatform status in the current README.

## 4. How comparable projects do it

| Project | Basic types | Hot-path story |
|---|---|---|
| Compose (`androidx.compose.ui.geometry`) | `Offset`/`Size` are `value class …(packedValue: Long)`. `Rect`/`Matrix` are classes (from memory, unverified). | Value classes are single-field. Arrays of them are stored as `LongArray`. |
| Kool | `open class Vec2f`, `MutableVec2f` subclass | Allocating operators plus `add(that, result)` and in-place variants |
| KorGE/Korma | Mutable `Point` historically. Value-class plans (2021). | Unenforced scoped pool, rejected by its author as unsafe |
| libGDX | Mutable vectors and pools (from memory, not re-verified) | `Pool.Poolable` with `reset()` |
| This monolith | `Pair<Float,Float>` and `Camera` data class | Allocation per sprite per frame, one draw call per sprite |

## 5. Pitfalls and regrets

- Packed `Long` value classes expose the representation as ABI and need a `LongArray` workaround for storage. `Unspecified`-by-NaN sentinels make reads throw. The JS `Long` penalty hits any `js` target.
- Value classes mean mangled names, `@JvmInline`, and Java/JS export friction. JS/TS export of `@JvmInline value class` only arrived in 2.4.0.
- `data` classes in public API freeze `copy`/`componentN` (Kotlin's own guidance).
- Mutable twin types double the surface, and scoped pools hide unenforced lifetimes (soywiz).
- `runCatching` around `suspend` code swallows cancellation.
- Lossy flows for input lose key-ups. `runBlocking` is unavailable off-JVM.
- `no-compatibility` or `@JvmDefaultWithoutCompatibility` retrofitted onto a shipped interface breaks old implementers.
- `sealed` shapes plus scheduled post-1.0 shapes become a breaking change for every third-party backend.
- Float determinism is not guaranteed across targets (JS `Float`).

## 6. Recommendation for this repo

1. **Types (decide in G2, before anything is frozen).**
   - `Point`, `Size`, `Rect` and `Color` are `public class` (not `data`), immutable, with explicit `equals`/`hashCode`/`toString` and `Float` components.
   - Use `Float` for geometry. It is GPU-sized and matches Compose. Document that world offsets are subtracted from the camera origin *before* narrowing, to avoid far-from-origin jitter on large tile maps.
   - Add `Vec3`, `Vec4` and `Matrix` as separate classes later. Because nothing is packed, widening to 3D never changes `Point`.
   - Drop the split plan's interface requirement for these types. Adapters in the client convert GeneralTools/GameTools types.
   - `Color` as an `Int` ARGB value class is the one defensible packing (native `Int` everywhere), but it would freeze 8-bit channels. I would not take it at 1.0 unless wide-gamut/HDR is ruled out.
2. **Hot path.** Every shape setter and draw entry point that takes a basic type also has a scalar overload. The renderer writes interleaved floats into a reusable buffer and never creates `Pair` or `Point` per sprite. House rule: FP/immutable *around* the renderer, imperative buffer writes *inside* it.
3. **Operators.** Keep `plus`, `minus`, `times` on the immutable types for app code and tests. Document that they allocate. No scoped pools, no public mutable twins.
4. **Shapes.** `abstract class Shape internal constructor()`, not `sealed`, with renderer dispatch inside core. Ids (`Key`, `PointerButton`) are `Int`-backed value classes with companion constants.
5. **Backend contracts.** Pre-1.0, keep `@SubclassOptInRequired` as the draft proposes. At 1.0, swap it for a warning-level acknowledgement marker (rather than experimental), per the tension noted in §3.3.
   - Evolve with default-bodied members under `-jvm-default=enable`, or with new capability interfaces (`as?`).
   - Add a Level 3 fixture: an example backend compiled against the 1.0 contract, run against each later release on JVM, Android and the web target.
   - Pin `languageVersion`/`apiVersion` at the oldest Kotlin you want consumers on, rather than the compiler's latest (recollection of common practice, unverified).
6. **Concurrency.** `AssetSource.load` is `suspend` returning `Result<ByteArray>`. No blocking variant in common (`runBlocking` is JVM-only). Use a `catching` helper that rethrows `CancellationException`. Frames and input are push callbacks with synchronous delivery on the render thread. Ship `awaitFrame()` and `events(): Flow` as extensions or adapters only. Inject a `CoroutineDispatcher` for audio streaming and decoding (parameterised policy, default `Dispatchers.Default`).
7. **Failures and lifetimes.**
   - `Result<T>` as the rule says, with failures drawn from a small sealed exception hierarchy (`AssetNotFound`, `ShaderCompileFailed`, `UnsupportedFormat`). Programmer errors (draw after close) stay exceptions.
   - GPU handles are `AutoCloseable` with idempotent `close()`. Make `close()` safe after GL/EGL/WebGL context loss.
   - `use` fits short-lived scopes only. Textures live in a cache or asset-group owner.
8. **Language features.**
   - DSL builders for *definitions* (scenes, particles, animations) with `@DslMarker`, and `inline` entry points where a per-call lambda would otherwise allocate.
   - Extension functions for adapters and convenience over a small core.
   - No `reified`, `inline` or context parameters in frozen contracts, because their bodies or requirements become ABI.
   - Enable `explicitApi()` and Kotlin Gradle plugin ABI validation from G1.
9. **Logging.** kotlin-logging as planned. Confine it to one internal file so Android/Wasm output problems can be swapped behind a tiny facade. Log lifecycle events only, never per frame or per sprite.

## 7. Gaps

- **ART and WasmGC allocation cost.** I had no web access left (session limit) for Android ART docs or the V8 WasmGC post. I could not confirm that Kotlin/Wasm maps `Long` to a native `i64` (only a Slack hint that single-field value classes are inlined). The "allocation is fine" and "packed values are fast on Wasm" claims remain unmeasured.
- **Default-method additions on klib targets** (JS, Wasm, Native): no authoritative source found. Treat as unproven until the CI fixture in §6.5 passes.
- **Multi-field value class status** (KEEP-0454): unconfirmed. The KEEP page was not reachable.
- **Compose `Rect`/`Matrix` as classes**, **libGDX `Vector2` mutability**, the **kotlinx.coroutines `ExperimentalForInheritanceCoroutinesApi`** pattern, and the **`expect`/`actual` class Beta status** are from memory and not re-verified.
- **`SubclassOptInRequired` introduction version**: the stdlib pages disagree (1.8 on archived 1.9/2.0 pages, 2.1 on later pages). Stable status in the pinned Kotlin version is unverified.
- **`@MustUseReturnValues`** is documented only as experimental in 2.3.
- **`Result` return types in interfaces, Java/JS interop and name mangling** were not resolved.
- **kotlin-logging** per-platform backends (Android, Wasm console) and its current multiplatform status were not confirmed. The README text I retrieved is stale (shows 7.0.3).
- **G1's `js` versus `wasmJs` choice** is not mine to make, but it drives the `Long` argument above. If `js` is chosen, value-class packing is clearly wrong. If `wasmJs`, it is merely unproven.
- **Firecrawl:** not used, per the brief. The session's web limit, not the Firecrawl restriction, was the binding constraint.
