# Research: what a 2D-only graphics API must reserve for a later 3D track

## 1. Question

What must SpartanGraphics' 2D API reserve from its first release so a later, unscheduled 3D track can be added without breaking the frozen 1.0 API, and what should it deliberately not reserve? The answer is based on libraries that grew from 2D to 3D or that serve both.

## 2. Bottom line

- **Reserve at the seams, not in the types.** Every precedent that kept 2D clean (libGDX, Godot, raylib, MonoGame) put the 2D/3D join at the *camera and renderer boundary* (a 4x4 view-projection matrix, a per-pass depth/stencil setting, a separate batch or renderer for 3D). Every precedent that put 3D *inside* the 2D types (Bevy's `Vec3` `Transform` for sprites, Unity's z-as-sort-key, JavaFX's `translateZ` on every `Node`) still hits surprises in 2D ordering, or carries 3D properties on 2D objects. So: `Point`, `Size`, `Rect` and every shape stay strictly 2D, with no z and no quaternion. 3D gets its own types in its own module later.
- **Phase 1 should make four cheap shape decisions now**, because each is hard to retrofit once G3/G4/G5 are built and frozen:
  1. The renderer contract consumes a **per-layer view-projection 4x4 matrix**, not camera fields (position/zoom). G5's batched renderer wants exactly one `mat4` uniform per layer anyway, so this costs the 2D user nothing and removes today's five by-name uniforms.
  2. A frame is an **ordered list of layers**, each with its own camera, clip and (later) target. The split plan's world and screen layers become the two default entries. This is needed for the scheduled multi-camera and render-to-texture work as well, so it is not a 3D-only cost.
  3. **Draw order is an explicit sort key**, separate from geometry. Do not use a `z` coordinate for layering. Bevy's design document and issue #4149 list the problems of z-as-layering: magic numbers, z-fighting, and parent z-scale reordering children.
  4. **Surface and render-target descriptors carry optional depth/stencil.** Stencil has a 2D consumer today (the scheduled visibility mask, and clipping). Window and context creation settings are the part that cannot be changed after the context exists, so they are the late-regret item.
- **Do not reserve:** z on `Point`/`Size`/`Rect`; a `Transform` with `Vec3`/`Quat` on shapes; rays in picking; a mesh/material/light shape in the closed shape set; a perspective-capable `Camera` that 2D users must configure; per-shape depth test. A 3D track is a *second renderable family* behind a second, opt-in backend capability, not a new `Shape` subtype. This matches libGDX (`SpriteBatch` vs `ModelBatch`, plus `DecalBatch` for sprites in 3D), Godot (canvas vs scenario) and raylib.
- **Rename the concrete camera to `Camera2D` now** (zero cost, nothing is implemented), and put a thin `Camera` interface above it that exposes only `viewport` and `viewProjection`. After 1.0 the unqualified name `Camera` cannot become the 3D-capable abstraction without a major bump. This is a judgement call; see section 6.
- **Validate before freezing.** The user's own tier rule says a seam is proven by a real consumer. Before 1.0, run a throwaway spike that records a depth-tested 3D layer through the test or headless backend using only the frozen contracts, to find reservations that are wrong. An unexercised reservation is the kind that breaks.

## 3. Findings

### F1. Libraries that grew 2D to 3D put the join in the camera and batch, not in the vector types
**Tier: common practice** (multiple mature libraries).

- **libGDX** has a general `Camera` whose state is `Vector3` `position`/`direction`/`up` and `Matrix4` `projection`/`view`/`combined`/`invProjectionView`. It also has `project`, `unproject` and `getPickRay`. `OrthographicCamera` is documented as the 2D one. `SpriteBatch` receives `camera.combined` via `setProjectionMatrix`. [Camera.java](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/Camera.java), [wiki](https://libgdx.com/wiki/graphics/2d/orthographic-camera)
- `SpriteBatch` itself is genuinely 2D. Its position attribute has 2 components, giving 5 floats per vertex and 20 floats per sprite. It applies `projection * transform` as one `u_projTrans` `mat4` per batch. [SpriteBatch.java](https://raw.githubusercontent.com/libgdx/libgdx/master/gdx/src/com/badlogic/gdx/graphics/g2d/SpriteBatch.java)
- A separate `DecalBatch` exists for "a Sprite that can be manipulated and rendered in 3D space". [libGDX decals](https://libgdx.com/wiki/graphics/3d/decals)
- libGDX never added z to `SpriteBatch`. 3D arrived as a parallel family (`ModelBatch`, `Decal`).
- **MonoGame/XNA** `SpriteBatch.Begin(..., DepthStencilState depthStencilState = null, ..., Matrix? transformMatrix = null)`. The matrix applies to the whole batch, and `Draw(..., float layerDepth)` exists per sprite. The vertex is `VertexPositionColorTexture` (3D position), and `layerDepth` is written into each vertex's `Position.Z`. [API](https://docs.monogame.net/api/Microsoft.Xna.Framework.Graphics.SpriteBatch.html), [SpriteBatchItem.cs](https://raw.githubusercontent.com/MonoGame/MonoGame/develop/MonoGame.Framework/Graphics/SpriteBatchItem.cs). This is the "reserve depth in the 2D batch" design. It works, but depth is a `[0,1]` ordering float that doubles as a depth-buffer value, a conflation of the two jobs.
- **raylib** keeps `Camera2D` (offset, target, rotation, zoom) and `Camera3D` as separate structs with `BeginMode2D`/`BeginMode3D`. [core_2d_camera.c](https://github.com/raysan5/raylib/blob/master/examples/core/core_2d_camera.c). The Camera3D fields were not verified (see Gaps).
- **Godot** splits 2D and 3D at the server level: `World2D` gives a `RenderingServer` *canvas*, `World3D` a *scenario*, and "the 3D APIs are different from the 2D ones". `Camera2D` extends `Node2D`, and `Transform2D` is a 2x3 affine matrix. [using_servers](https://docs.godotengine.org/en/stable/tutorials/performance/using_servers.html), [Camera2D](https://docs.godotengine.org/en/stable/classes/class_camera2d.html), [Transform2D](https://docs.godotengine.org/en/stable/classes/class_transform2d.html). Godot also has 2D and 3D physics servers that are "identical" in use but separate. Where Godot users needed 3D-ish effects inside 2D, the maintainers added a pseudo-3D option to the 2D engine rather than routing through 3D ("not happy with the amount of work"). [gamedev.net summary, secondary](https://gamedev.net/news/godot-to-simplify-3d-support-in-the-2d-engine-r815)

### F2. Putting 3D transforms inside the 2D object model is the precedent people regret
**Tier: common practice with one explicit design-document regret (Bevy), one engine behaviour (Unity).**

- **Bevy** historically used one `Transform` (Vec3 translation, Quat rotation) for 2D, 3D and UI, with sprite layering on `translation.z`. The 2D working group's document says: "Rotating an object in 2D should be easy. In Bevy, it's not." It lists the z-layering problems: it "encourages magic numbers", "leads to z-fighting bugs", and z-scale on 2D objects "has a subtle and surprising effect on the relative ordering of sprites". [Bevy 2D transforms](https://hackmd.io/@bevy/B17Lzrh6ee)
- Issue #4149 (filed Bevy 0.6.1, still open; labelled as needing a design doc in Oct 2025) shows a parent's `scale.z` changing a child's draw order. A member called it "a strong argument for dedicated `Transform2D` types" and "very surprising to beginners". [#4149](https://github.com/bevyengine/bevy/issues/4149)
- The proposal is a `Transform2d` (`Vec2` translation, 2D rotation, scale) plus *separate* sort components (ZIndex, Y-sort), while a shared `GlobalTransform` stays the render interface. In the proposal `Camera2d` requires `Transform2d` and `Camera3d` requires `Transform3d`. It is marked WIP and "a recommendation and proposal, not a decision", so whether it landed is unconfirmed. [2d_transforms](https://hackmd.io/@bevy/2d_transforms)
- **Unity** runs 2D on its 3D engine. Sprites sort by sorting layer, then order-in-layer, then render queue, then *distance to camera*. With an orthographic camera you "change the order by changing the Transform's z position", and a Transparency Sort Axis can be changed for top-down or isometric games. [Unity 2D sorting](https://docs.unity3d.com/Manual/2d-renderer-sorting.html). Unity thus has to expose camera-distance and sort-axis concepts to 2D users.
- **Godot** orders by an explicit `z_index` (clamped between `CANVAS_ITEM_Z_MIN`/`MAX`, optionally relative to the parent) plus optional `y_sort_enabled`, never by a geometric z. [CanvasItem](https://docs.godotengine.org/en/stable/classes/class_canvasitem.html)
- The convergent lesson is that an explicit order key is robust, and geometric z is where surprises come from.

### F3. The camera's 3D-agnostic common ground is "viewport plus view-projection matrix"
**Tier: common practice.**

- libGDX's `combined` matrix, MonoGame's `transformMatrix`, and Bevy's computed camera projection all feed 2D and 3D batches the same way. Compose's `Matrix` is a flat 16-element `FloatArray` with `rotateX/Y/Z`, `translate(x,y,z)` and `map(Offset)`, even for a 2D UI toolkit. [Compose Matrix](https://composables.com/jetpack-compose/androidx.compose.ui/ui-graphics/classes/Matrix). Flutter's `Canvas.transform(Float64List matrix4)` takes a 4x4 for 2D drawing. [Flutter](https://api.flutter.dev/flutter/dart-ui/Canvas/transform.html)
- The things a 3D camera has that a 2D one lacks (frustum, `getPickRay`, near/far) are *not* part of the common ground. 2D `visibleRegion: Rect` and `screenToWorld(Point): Point` do not generalise; they become a frustum and a ray. Cull and pick therefore belong to the layer kind, not to the camera interface.
- **Pitfall in sources:** documentation of the 4x4 layout is easy to get ambiguous. The search summary for Compose notes `Matrix.get` documented as row-major indexing while `Canvas.concat` says column-major, unresolved in what I could read (secondary reading, see Gaps). Fix layout, handedness and clip-space depth range in KDoc before freezing.

### F4. 2.5D on a 2D API is a layer/group property, and it is additive
**Tier: common practice (two sources).**

- Compose's `graphicsLayer` exposes `rotationX`, `rotationY` and `cameraDistance`, which belong to a layer, not to shape geometry. [graphicsLayer docs](https://composables.com/jetpack-compose/androidx.compose.ui/ui/modifiers/graphicsLayer). The Android `View` equivalent documents `cameraDistance` in "depth pixels", defaults depending on screen density (1280 on medium density, 1920 on high), and warns of "drawing artifacts" when a rotated view ends up behind the camera. [View.SetCameraDistance](https://learn.microsoft.com/dotnet/api/android.views.view.setcameradistance)
- PixiJS offers perspective effects via meshes. Per a third-party summary these "fake perspective at the UV level, with no depth buffer", and batching is reduced when depth or culling flags are set. [PixiJS mesh guide](https://pixijs.com/8.x/guides/components/scene-objects/mesh) (the "no depth buffer / batching disabled" detail comes from a community skill page, so treat it as weak).
- PixiJS says real 3D is being built "designed to work alongside the 2D renderer", reusing the 2D batching architecture, and has not shipped it yet. [PixiJS 8.16.0](https://pixijs.com/blog/8.16.0)
- Implication: a per-layer 3D transform (flip, tilt) can be added later to a layer without touching shapes, as long as the layer already feeds the renderer a 4x4.

### F5. JavaFX is the closest "reserved 3D in a 2D scene graph" precedent, with a capability flag
**Tier: one source (official docs); useful as a worked example, not as a consensus.**

- Every JavaFX `Node` carries `translateZ`, `rotationAxis`, `scaleZ` and `depthTest`, and `viewOrder` is its explicit "rendering and picking order within its parent". [Node](https://openjfx.io/javadoc/21/javafx.graphics/javafx/scene/Node.html)
- `SubScene` separates "different parts of a scene, each of which can be rendered with a different camera, depth buffer, or scene anti-aliasing", with "Mixing 2D and 3D content" as a use case, and the depth buffer and anti-aliasing flags are "conditional features" (`ConditionalFeature.SCENE3D`). [SubScene](https://openjfx.io/javadoc/21/javafx.graphics/javafx/scene/SubScene.html)
- Take: the *per-pass camera plus depth buffer plus capability flag* part is the reusable idea. Putting the 3D properties on every 2D node is the cost; that part does not need copying.

### F6. 3D inside a 2D-first Kotlin engine historically stayed a separate, experimental library
**Tier: one source, plus absence of evidence.**

- KorGE's 3D was a separate experimental module, "only an experimental preview", API "likely to evolve", "not optimized at all", no raycasting or physics APIs. A Kotlin-community thread says the maintainers had no 3D plans for KorGE 2.0. [KorGE-3D](https://soywiz.com/presenting-korge-3d/), [Slack thread](https://slack-chats.kotlinlang.org/t/505280/is-anyone-else-woking-on-the-ui-stuff-or-the-3d-stuff-i-woul). I did not find what core 2D changes it required.
- Kool (3D-first, Vulkan/WebGPU/OpenGL, JVM, Android, browser) ships an embedded Compose-like UI framework that "needs to run within the OpenGL context". [kool README](https://github.com/kool-engine/kool). I could not establish how it relates 2D UI to its 3D camera (see Gaps).
- LittleKt, a Kotlin multiplatform 2D framework on WebGPU, describes itself as 2D only. [LittleKt](https://github.com/littlektframework/littlekt)

## 4. How comparable projects do it

| Project | 2D point/vector | Transform for 2D object | Camera | Ordering | 3D joined how |
|---|---|---|---|---|---|
| libGDX | `Vector2`, separate `Vector3` | per-batch `Matrix4`; sprite fields in 2D | one `Camera` base (Vector3 + Matrix4) with `OrthographicCamera`/`PerspectiveCamera` | batch order | separate `ModelBatch`, `DecalBatch` |
| MonoGame/XNA | `Vector2` | optional batch `Matrix` | none (matrix) | `layerDepth` float, written to vertex z | same batch, depth state on `Begin` |
| Godot | `Vector2` / `Vector3` | `Transform2D` (2x3) | `Camera2D` / `Camera3D` | `z_index`, Y-sort | separate server worlds (canvas vs scenario) |
| Bevy | `Vec2` / `Vec3` | `Transform` with `Vec3` + `Quat` for everything; `Transform2d` proposed | `Camera2d` / `Camera3d` over a common `Camera` | z in translation; sort components proposed | one pipeline family; regrets documented |
| Unity | `Vector2` / `Vector3` | full 3D `Transform` | one `Camera` (ortho or perspective) | sorting layer, order, then camera distance | 2D on the 3D engine |
| raylib | `Vector2` / `Vector3` | n/a (immediate) | `Camera2D` / `Camera3D` | draw call order | modes: `BeginMode2D` / `BeginMode3D` |
| JavaFX | `Point2D` / `Point3D` | every `Node` has Z fields | `ParallelCamera` / `PerspectiveCamera` per `SubScene` | `viewOrder` | per-subscene depth buffer, `SCENE3D` flag |
| Compose | `Offset` (2D) | 4x4 `Matrix`; `graphicsLayer` rotationX/Y + `cameraDistance` | none | draw order | tilt as a layer property |

(JavaFX `Point2D`/`Point3D`, `ParallelCamera`/`PerspectiveCamera` and Unity `Vector2`/`Vector3` are from general knowledge, not fetched.)

## 5. Pitfalls and regrets

- **Geometric z as layering.** Documented regret (Bevy design doc, issue #4149). Unity needs a Transparency Sort Axis feature to patch it for top-down games.
- **One transform type for both.** Bevy's 2D transform is awkward ("rotating in 2D should be easy. In Bevy, it's not") and its Z behaviour surprised users. Its fix is two transform types sharing only the render-facing `GlobalTransform`.
- **A `Point` interface that later grows a `z`.** The split plan makes basic types contract *interfaces*. Adding a member to a published interface breaks every implementer (a default value only softens that). So `Point` must be 2D permanently; a later 3D type needs a distinct name (JavaFX uses `Point2D`/`Point3D`; libGDX uses `Vector2`/`Vector3`).
- **Wrong reservations are the expensive ones.** An unexercised depth/stencil descriptor or 3D matrix convention that turns out wrong is a post-1.0 break. Reserve only what has a 2D consumer now, and spike the rest.
- **Matrix convention drift.** Layout (row vs column major), handedness, y-up vs y-down, and clip-space depth range (OpenGL -1..1 versus 0..1 on Vulkan, D3D and WebGPU; stated from general knowledge, not fetched) must be fixed in one place. A matrix in a frozen backend contract with an unspecified convention is a trap for third-party backends.
- **Context-creation attributes.** Depth and stencil bits are chosen when the window or context is created (GLFW hints, Android EGL config, WebGL context attributes), so a surface config without them forces a contract change. This is stated from general knowledge. The primary sources failed to load, see Gaps.
- **Batching hostility.** Per the PixiJS community notes, setting depth or culling flags disabled batching in its mesh path. If depth is added later, a batcher whose sort key is a depth value (MonoGame's `layerDepth`) mixes two jobs: a key chosen for batching (texture/state) and a key chosen for ordering. Keep order key and depth separate.
- **Per-shape 3D data on 2D objects.** JavaFX pays this for convenience (`translateZ` on every `Node`). Under the user's "closed shape set, additive and opt-in" rules, this would surface in every 2D user's autocomplete forever.

## 6. Recommendation for this repo

### Reasoning in two sentences
Use the "second renderable family behind a second opt-in capability" pattern (libGDX, Godot, raylib), with the one shared currency being a view-projection matrix at the renderer boundary. Reject the "reserve z everywhere" pattern (Bevy, Unity, JavaFX, MonoGame's `layerDepth`) because its costs land on every 2D user and its benefits are unproven until a 3D consumer exists.

### Per-surface decisions

| Surface | Reserve now | Do not reserve | 2D cost today |
|---|---|---|---|
| **Point/vector types** (G2) | `Point`, `Size`, `Rect` stay 2D and immutable. Later 3D adds `Vec3`/`Point3` etc. in the 3D module under different names. | z or w on any existing type. | None. |
| **Per-shape transform** (G3) | Keep the plan's rectangle with rotation (about the screen axis) and alpha. | `Transform` with `Vec3`/`Quat`, per-shape `Matrix4`. | None. At 10k sprites a 2D affine is 6 floats (24 B) versus 16 floats (64 B) for a 4x4, i.e. 240 KB versus 640 KB per frame if uploaded per sprite. That is my arithmetic; it is small but pointless to pay when only rotation is exposed. A libGDX-style CPU-expanded vertex is 20 floats (800 KB at 10k) versus 24 (960 KB, +20%) with a z component (MonoGame style). Not measured. |
| **Camera** (G2) | Name the concrete class `Camera2D` (position, zoom, viewport size, world/screen conversion, visible region, as in the split plan). Add a thin `Camera` interface: `viewport: Size` and `viewProjection: Matrix4`. | Perspective or frustum in `Camera2D`; making `visibleRegion` generic. | A naming change only. |
| **Renderer boundary** (G4/G5) | Each layer is submitted with a `viewProjection` 4x4 plus clip rect. Define `Matrix4` once (16 floats, column-major, immutable, KDoc'd handedness and y direction). The batched G5 renderer uses one `mat4` uniform per layer. | Camera fields (position/zoom) crossing the contract. | Positive: replaces five by-name uniforms per draw. |
| **Layers and draw order** (G3/G4) | Frame = ordered list of layers, each with camera, clip and later a target. World and screen layers are the two defaults. Within a layer, an explicit order key (integer plus stable insertion tiebreak). Document that it is a painter's-order key and that "depth" is reserved for a real position along the camera axis. | Geometric z as the layer mechanism; a depth buffer in 2D layers. | The order key is wanted anyway (UI stacking). |
| **Shapes vs meshes** | The shape set stays closed. 3D adds a separate renderable family in a later `graphics-3d` module (like `ModelBatch`/`DecalBatch`). A 3D scene appears as a layer kind. | A `Mesh` or `Model` shape, materials or lights in graphics-core. | None. |
| **Picking** (G3) | Return a small value type, not a bare tag: `PickHit(tag, worldPoint)`, additive-friendly. Picking runs against a layer's camera and content. | Rays, distance, 3D hit tests. A 3D layer defines its own `pickRay` (libGDX `getPickRay`). | One extra wrapper type. |
| **Render targets and surface** (G4, later RTT) | Descriptor fields for `depth`/`stencil` default-off; surface config requests depth and stencil bits at creation. Stencil is justified now by the visibility mask and clipping. | Per-shape depth test; MSAA policy beyond a request field. | None while off. |
| **Backend contract** | Add a `RendererCapabilities` value (immutable, extended additively) so a backend can say whether it supports depth-tested layers. A future `SceneRenderer3D`-style interface is an optional extra a backend implements. Third-party 2D backends never need it. (JavaFX `ConditionalFeature.SCENE3D` is the precedent.) | Lights, materials, pipelines for 3D in the Phase 1 or 1.0 contract. | None. |

### Timing

**Settle in Phase 1 (these shape G2, G3, G4, G5; retrofitting after they exist means rewriting them):**
1. `Camera2D` plus the thin `Camera` interface, and the `viewProjection` boundary (G2, G4, G5).
2. Frame as an ordered list of layers with per-layer camera and clip (G3, G4). G3 already adds the clip rectangle per issue #1, so the layer is the natural carrier.
3. The order key, and keeping `Point`/`Size`/`Rect` and shapes strictly 2D (G2, G3).
4. `PickHit` instead of a bare tag (G3).

**Can wait until just before 1.0 (0.x may break):**
- Exact `Matrix4` API surface beyond what the renderer needs, `RendererCapabilities` fields, depth/stencil descriptor formats, layer-level 3D transform (`graphicsLayer`-style tilt), and the example backend's documentation of these. Decide them together with the web and Android backends, because those are the first real tests of what a third-party backend can implement.

**Never schedule in the 2D phases:** `graphics-3d`, mesh types, lights, rays, 3D shaders. They are a separate roadmap.

### Suggested scope change for issue #1
Add to G2: rename the concrete camera `Camera2D`, add the `Camera` interface and `Matrix4`. Add to G3: layers as an ordered list with a per-layer camera, clip and order key; `PickHit`. Add to G4/G5: the `Renderer` contract takes layer + `viewProjection`; the desktop renderer uses one `mat4` uniform per layer; surface config includes optional depth/stencil request. None of this adds a 3D feature. It re-shapes three stages that are not yet implemented.

### What I would reject
- The obvious option "put an optional `z` (defaulting to 0) on `Point`, `Rect` and shapes". It is cheap to write but makes z-as-layering the default habit, which is the documented Bevy/Unity regret. It also bakes a depth semantic (sort key vs real depth) into the frozen API before any 3D consumer exists.
- A full 3D-capable `Camera` hierarchy in Phase 1 (libGDX-style `Vector3` position and direction). It buys nothing for 2D users and freezes a shape no one has built against.

## 7. Gaps

- **Web sources that failed to load** (session limit hit): MDN's WebGL context attributes page (depth/stencil defaults, immutability after creation) and GLFW's window guide (depth/stencil hint defaults, must be set before window creation). My statement that depth and stencil are chosen at context creation is from general knowledge and is unverified here. Re-check before the surface-config design is final.
- **Camera3D fields in raylib** were not verified (the fetched page showed only `Camera2D`).
- **Kool's 2D-over-3D model and KorGE's core-2D changes for 3D** are not established; I found only high-level statements.
- **Bevy `Transform2d` shipped status** could not be confirmed. The 2D working-group document is a proposal; treat Bevy as "regret documented, fix in design".
- **JavaFX `Point2D`/`Point3D`, `ParallelCamera`/`PerspectiveCamera`, Unity `Vector2`/`Vector3`**, and the clip-space depth-range difference between OpenGL and Vulkan/D3D/WebGPU are stated from general knowledge, not fetched.
- **Cocos2d-x** (a 2D engine that added 3D to its node class) came up in search only through forum posts and a header mirror, so I excluded it as a precedent. It would be a useful additional example of a 2D node growing 3D.
- **The per-sprite cost numbers** in section 6 are arithmetic from float counts, not benchmarks. Whether a 4x4 per shape would hurt at 10k sprites on the JVM is not measured. Nothing in the recommendation depends on it, because shapes do not carry matrices.
- **Compose's matrix layout inconsistency** (row-major `get` versus column-major `concat`) came from search summaries and was not verified in source.
- **Brief gaps:** I did not read `architecture.md`'s `Renderer`/`Frame` design in detail, only grepped it. It contains no 3D reservations, and its recording `Renderer` in a `graphics-test` backend would be the natural place for the pre-1.0 spike. Whether `Matrix4` should live in `graphics-core` or a tiny shared math module is a module-layout question I left to the sibling researchers (questions 2 and 5).
- **Decision for the user:** is the `Camera2D` rename plus thin `Camera` interface acceptable? The alternative of keeping a concrete `Camera` and later adding `Camera3D` without a shared interface (Godot, raylib) works too, at the price that the unqualified name means "2D" forever.
