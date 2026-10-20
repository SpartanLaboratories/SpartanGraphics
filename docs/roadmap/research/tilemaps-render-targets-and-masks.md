# R3-R6 and the frozen backend contract: atlases and animation, tile maps, render targets and masks, particles

Researcher N3. Date: 2026-10-11. Tools: `gh api` (library source at pinned tags), `WebFetch`, `WebSearch`. Firecrawl not used. Citations: documents by section, code by symbol at a pinned tag, never by line number.

## 1. Question

What do the scheduled R3-R6 features (sprite atlases and frame animation; generic tile-map layers; render-to-texture with several cameras or viewports and a generic visibility mask; particles) require of the backend contract frozen at 1.0, and how do mature 2D engines build them?

**Decision it informs:** D8. Which slots the 1.0 backend contract reserves, whether each is Stable or Experimental at 1.0, and the order of R3-R6 after 1.0.

## 2. Bottom line

- **Almost nothing in R3-R6 needs a new backend primitive, provided the 1.0 contract has four things that G5 needs anyway.** These are (a) a blend state with separate colour and alpha factors and the presets Normal, Premultiplied, Additive and Multiply; (b) a per-pass viewport rectangle; (c) texture sampler state plus sub-region update; (d) 32-bit indices. Atlases and animation (R3), the first tile-map tier (R4) and CPU particles (R6) are then pure `graphics-core` work over the universal textured, vertex-coloured pipeline. Make (a)-(d) Stable and implemented at 1.0.
- **Tile maps: stream chunk-cached vertices first, add retained GPU geometry only if the 4c gate fails.** Re-expanding every visible tile every frame (libGDX `OrthogonalTiledMapRenderer`) is what does not survive 0.1x zoom: by my arithmetic about 50k tiles at 64 px is about 4.8 MB of vertex data per frame. Mature engines all cache tile geometry in chunks (Godot `rendering_quadrant_size` 16x16, `bevy_ecs_tilemap` 64x64, libGDX `SpriteCache`). A camera-independent chunk vertex cache, copied into the frame stream, needs no contract change. Retained GPU buffers then become a capability-gated optional extension, **Experimental at 1.0**, with exact streaming as the fallback. A shader-side data-texture layer (Phaser 4 `TilemapGPULayer`) is the only technique with zoom-independent cost, but it needs its own pipeline. Defer it behind R7.
- **The visibility mask needs no stencil and no new primitive.** Fog of war in mature RTS work is a low-resolution visibility grid built on the CPU, smoothed, and composited (Riot: 128x128 grid, blurred). In this repo that is a dynamic RGBA texture whose alpha is fog opacity, drawn as one linearly filtered quad with normal alpha blending. That uses only texture update, sampler state and blend. Stencil cannot do soft edges, and Phaser 4 and Godot have both moved their general masks to texture-alpha. Keep the stencil request field on the surface and render-target descriptors (default off, capability-reported). Nothing in R3-R6 depends on it.
- **Render targets are needed for caching and for a "real" second camera, not for the minimap or the fog.** Reserve them as an open `PassTarget` interface in the frame (the surface is one implementation), plus a per-pass viewport. Keep render-target creation, the handle type, the `Load` (non-clear) op and a `contentsLost` signal **Experimental at 1.0**, as `backend-contract-design.md` §6.6 already proposes. Persistent-accumulation targets (a fog that remembers explored cells) cannot survive Android EGL loss or `webglcontextlost`, so the explored-state grid must live in CPU memory and the target is only a cache.
- **Particles: CPU simulation emitting quads into the batch; reject GPU particles for the contract.** libGDX, Godot `CPUParticles2D` and Pixi `ParticleContainer` all run this way. Godot's GLES3 `GPUParticles2D` needs transform feedback, and Godot hard-disables it on Adreno 3xx for driver bugs. Transform feedback is a GL concept that would leak into a frame-level contract.
- **Order after 1.0: R3, then R4 tier 1, then R5, then R6, with R4 tier 2 (retained geometry) pulled in only when a measured 4c scene demands it, and R7 last.** R3, R4 tier 1 and R6 need no backend work. R5 is the one phase with new backend obligations, so it is the one that needs a new conformance suite. R6 could move ahead of R5 at no contract cost if the roadmap wants it.

## 3. Findings

Evidence tiers: **Consensus** (several independent credible sources or a spec), **Practice** (what mature libraries visibly do), **Opinion** (one source's view, or my inference, labelled). Numbers prefixed "C" refer to section 7.

### 3.1 Tile maps

**F1. Re-batching visible tiles every frame is the simple baseline, and it is O(visible tiles) per frame. Tier: Practice.**
libGDX 1.14.2 `OrthogonalTiledMapRenderer.renderTileLayer` derives a column and row range from `viewBounds`, then writes one 20-float quad per tile and calls `batch.draw(region.getTexture(), vertices, 0, NUM_VERTICES)` per tile (C1). Nothing is retained. KorGE 6.0.0 `TileMap` does the same re-expansion per visible rect, but keeps the vertex arrays per texture and rebuilds only when the virtual rect, `map.contentVersion` or `animationVersion` changes. Its own source carries `// @TODO: Use instanced rendering to support much more tiles at once` (C2).

**F2. Mature engines cache tile geometry in fixed-size chunks. Tier: Practice (four independent engines).**
- Godot 4.7.2 `TileMapLayer.rendering_quadrant_size`, default 16: "A quadrant is a group of tiles to be drawn together on a single canvas item, for optimization purposes." The default groups 16x16 = 256 tiles. Y-sorted layers group by Y instead (C3).
- `bevy_ecs_tilemap` v0.15.0: `CHUNK_SIZE_2D = [64, 64]`. Each chunk is its own mesh, `RenderChunk2d` carries `dirty_mesh` and a per-chunk frustum test, and the docs say larger chunks suit maps that change rarely and smaller chunks suit maps that change often (C4).
- libGDX `OrthoCachedTiledMapRenderer`/`SpriteCache`: "Renders ortho tiles by caching geometry on the GPU", cached for the view plus an over-cache margin, rebuilt when the view reaches the edge of the cache. It warns of "poor performance when tiles are often changed dynamically". `SpriteCache` stores geometry "in video memory" so it "does not have to be sent to the GPU each time it is drawn" (C5).
- `@pixi/tilemap` v5.0.2 keeps an interleaved points buffer per `Tilemap` and re-uploads vertices only "if layer was changed" (C6).

The trade-off is explicit in the sources: chunk size trades culling granularity and rebuild cost against draw-call count.

**F3. A cached renderer that only covers the view is a memory bound, not an optimisation of the whole map. Tier: Practice.**
libGDX's cached renderer caches only `cacheBounds` (view plus over-cache) and rebuilds on edge contact, because retaining an entire large map's geometry costs memory (C5). Arithmetic, mine: at 20 floats (80 B) per tile, a 1000x1000 map is 80 MB of vertices. Chunks must therefore be built lazily for visible chunks and evicted.

**F4. Animated tiles: three techniques, each with a cost. Tier: Practice.**
- *Rebuild on tick* (KorGE): `animationVersion++` when a tile's frame duration elapses, which invalidates the vertex arrays (C2).
- *GPU animation from a time uniform* (`@pixi/tilemap`, `bevy_ecs_tilemap`): per-vertex `aAnim`/`aAnimDivisor` plus a `u_anim_frame` uniform (pixi), or `fract(tilemap_data.time * animation_speed) * frames` (bevy). This keeps the buffer static but needs a custom shader, so it is not the universal pipeline (C6, C4).
- *A global static clock* (libGDX `AnimatedTiledMapTile`: `private static long lastTiledMapRenderTime`, set by `updateAnimationBaseTime()`). The cached renderer has no reference to animated tiles at all (searched; zero occurrences), so by inference it snapshots the frame current at cache time (C7).

**F5. Tile seams are a known defect class with three mitigations. Tier: Practice.**
- libGDX cached renderer insets UVs by half a texel (`adjustX = 0.5f / texture.getWidth()`) (C8).
- `@pixi/tilemap`'s fragment shader clamps the sampled coordinate to the tile's frame (`clamp(vTextureCoord, vFrame.xy, vFrame.zw)`) (C6).
- Phaser 4: "A regular TilemapLayer cannot render smooth borders like this, creating sharp seams between tiles", while its GPU layer renders no seams with linear filtering (C9).
- Build-time padding and bleed in the atlas packer is the third (`asset-pipeline.md` §3.8).

**F6. A shader-side data-texture layer gives zoom-independent cost, with strict limits. Tier: one source (Phaser 4.2.1), shipped.**
`TilemapGPULayer` draws the whole layer as one quad. The shader reads a tile-data texture (each texel: flip bits, animation bit, 28-bit tile index) and an animation-data texture. Limits: single tileset and texture image, at most 4096x4096 tiles, orthographic only, WebGL only. "It has a fixed cost per pixel on screen, whether there is anything in that tile or not. In general, it suffers no performance loss when many tiles are visible." Editing means regenerating the data texture (C9). This is the only evidence I found that directly addresses "a large map at low zoom", and it cannot be expressed with the universal pipeline.

**F7. Index width limits chunk and layer size. Tier: Practice and spec.**
KorGE's batcher caps at 16383 quads on 16-bit indices (`render-performance-and-verification.md` §3.1). `@pixi/tilemap` documents "a limitation on 16k tiles per one tilemap" lifted by `use32bitIndex` (C6). `UNSIGNED_INT` indices are available by default in WebGL2 (MDN, C10), and are core in GL 3.3. A 64x64 chunk is 16384 vertices, which fits 16-bit indices, but only just for a 128x128 chunk.

### 3.2 Sprite atlases and animation

**F8. Atlas regions are looked up by name, and a name can map to an ordered frame list. Tier: Practice.**
libGDX `TextureAtlas`: `findRegions(String name)` returns the regions sharing a name ordered by `index`; `AtlasRegion` carries `index`, trim offsets (`offsetX`, `originalWidth`) and `rotate`/`degrees` (C11). The build-time packer defaults come from `asset-pipeline.md` §3.8 (padding 2, bleed on, rotation off). Rotation requires draw-path support, so it is off by default there.

**F9. Animation is a frame list plus timing plus a play mode, and sampling is a pure function of elapsed time. Tier: Practice.**
- libGDX `Animation<T>`: uniform `frameDuration`, `enum PlayMode { NORMAL, REVERSED, LOOP, LOOP_REVERSED, LOOP_PINGPONG, LOOP_RANDOM }`, `getKeyFrame(stateTime)` where `stateTime` is held by the caller (C12).
- Godot 4.7.2 `SpriteFrames`: named animations, per-animation fps, and a **relative per-frame duration** (`add_frame(anim, texture, duration=1.0, at_position=-1)`), with `absolute_duration = relative_duration / (animation_fps * abs(playing_speed))` (C13).
- libGDX's tile animation has per-frame intervals (`animationIntervals[]`), so per-frame durations are the more general form (C7).

**F10. Runtime atlasing exists for assets not known at build time. Tier: Practice.**
libGDX `PixmapPacker` "supports incremental inserts and updates of TextureAtlases generated with this class" (C14). This matters here because the client's server-chosen texture names arrive in snapshots and load lazily (`consumer-needs-and-dependencies.md` §3.3). Build-time packing cannot cover names it does not know. Without runtime packing, each unpacked texture is a batch break. It needs only texture sub-region update from the backend.

### 3.3 Render targets, viewports and cameras

**F11. Render-to-texture is universal, and mature engines hide one of two policies for freshness. Tier: Practice.**
- Godot 4.7.2 `SubViewport`: `render_target_clear_mode` (ALWAYS, NEVER, ONCE) and `render_target_update_mode` (DISABLED, ONCE, WHEN_VISIBLE, WHEN_PARENT_VISIBLE, ALWAYS), exposed to the app through `ViewportTexture` (C15). The engine owns the schedule.
- Bevy v0.20.0 `Camera`: `viewport: Option<Viewport>` (a physical rectangle inside the target), `order: isize` ("Cameras with a higher order are rendered later"), and `RenderTarget { Window, Image, TextureView, None }` (C16). Viewport and target are separate fields.
- Kool v0.19.0 `RenderBackend.createOffscreenPass2d(parentPass: OffscreenPass2d): OffscreenPass2dImpl`, and `DeviceCoordinates.ndcYDirection` (C17).
- libGDX `FrameBuffer(Pixmap.Format format, int width, int height, boolean hasDepth, boolean hasStencil)`: "FrameBuffers are managed. In case of an OpenGL context loss ... the framebuffer will be automatically recreated." Recreated means a fresh empty target, not restored contents (C18).

Because the recorded-frame contract in `backend-contract-design.md` §6.1 makes passes explicit, a render target's update schedule is simply whether its pass is in this frame's `RenderFrame`. No Godot-style update-mode enum is needed. That is an inference from the contract's shape, not an engine practice.

**F12. Render targets as a cache for static content, with a size cap. Tier: Practice.**
Pixi v8 `cacheAsTexture` renders a container to a texture so later frames draw one quad. Its docs warn that "If your container is too large (e.g., over 4096x4096 pixels), the texture may fail to cache, depending on GPU limitations", and that cached items "are rendered at their actual size, ignoring transforms like scaling" (C19). So a render-target cache of a large map needs a `maxRenderTargetSize` capability and a tiled scheme.

**F13. Y orientation of render targets is a standing trap and mature engines make the backend resolve it. Tier: Practice.**
- Pixi v8 `RenderTargetSystem.isFrontFaceInverted`: "WebGL's non-root FBOs carry an inherent Y-flip vs the root (the classic render-texture flip)", with `inverted = flipY XOR (isWebGL && !isRoot)` (C20).
- The libGDX wiki: "since GL assumes lower-left origin and TextureRegion assumes upper-left ... we need to flip the Y texcoord in order for it to render correctly" (C21).
- Kool lifts it into the backend contract as `DeviceCoordinates.ndcYDirection` (C17). This matches the clip-rectangle lesson already in `backend-contract-design.md` §3.6.

**F14. Premultiplied alpha is the correct storage for a render target holding translucent content. Tier: Practice (secondary tutorials).**
The mattdesl LWJGL/libGDX FBO tutorial: drawing translucent sprites into an FBO and again onto the screen blends twice, and the fix is "Enable pre-multipled alpha blending, `GL_ONE, GL_ONE_MINUS_SRC_ALPHA`" with opacity applied as `setColor(alpha, alpha, alpha, alpha)` (C21). A Khronos forum thread recommends `glBlendFuncSeparate` for writing into the buffer (search snippet only; not fetched). Hence the blend state needs separable colour and alpha factors. libGDX `ParticleEmitter` itself has a `premultipliedAlpha` flag next to `additive` (C22).

### 3.4 Masks and the visibility mask

**F15. Engines split masks into a hard stencil clip and a soft texture-alpha mask, and the soft one is where general masking is going. Tier: Practice (three engines).**
- Pixi v8.22.0 has both. `StencilMask`: "It is more performant than using a `Graphics` mask, but requires WebGL support." Its `StencilMaskPipe` breaks the batch, draws the mask with blend `none`, then draws the children, then re-draws the mask on pop. `AlphaMask` multiplies by a mask texture's red or alpha channel and "non-sprite masks are rendered to a texture automatically" (C23).
- Phaser 4.2.1 has dropped stencil masks from WebGL. `GeometryMask`: "GeometryMask is only supported in the Canvas Renderer. If you want to use geometry to mask objects in WebGL, see FilterList#addMask." The replacement `Filters.Mask` "multiplies the color and alpha of the input by the alpha of the mask in the corresponding texel" and renders a GameObject mask to a `DynamicTexture` (C24).
- Godot 4.7.2 `CanvasItem.clip_children`: "The mask is based on this node's alpha channel: Opaque pixels are kept, transparent pixels are discarded, and semi-transparent pixels are blended in according to their opacity." Clipping nodes cannot be nested (C25).

A stencil clip is binary, so it cannot make a soft fog edge.

**F16. Stencil availability is per-platform and off by default in two of the three official backends. Tier: Consensus (spec and platform source).**
- WebGL: `boolean stencil = false;` in `WebGLContextAttributes`; "If the value is false, no stencil buffer is available." (C26)
- Android: "By default GLSurfaceView chooses a EGLConfig that has an RGB_888 pixel format, with at least a 16-bit depth buffer and no stencil." A stencil needs `setEGLConfigChooser(r, g, b, a, depth, stencil)` before `setRenderer` (C27).
- GLFW: `GLFW_STENCIL_BITS` defaults to 8, so desktop has one without asking (C28).
- Pixi demonstrates the failure mode: it creates its own WebGL context with `stencil: true`, and warns "Provided WebGL context does not have a stencil buffer, masks may not render correctly" when handed one without (C23).

This confirms `room-for-3d.md` §3's reliance on the surface config: the request must be explicit and reported back, and no 1.0 feature should silently require it.

**F17. Fog of war in shipped RTS work is a coarse CPU-computed grid, smoothed, then composited. Tier: Practice (one detailed primary source; others secondary).**
Riot Games (League of Legends): "the entire map was covered by a 128x128 grid"; it was Gaussian-blurred before being rendered into the scene; blurring the same 128x128 image more "made it too blurry too quickly", so they upscaled to 512x512 with a pixel-to-4x4-block scheme and then blurred, and found that a 256x256 grid "would have incurred an unacceptable performance hit in the form of a quadrupled CPU cost" (C29). A search snippet adds that Stratagus exposes a bilinear-upscale toggle for its fog texture and that a gamedev.net thread suggests one texel per tile (not fetched). The GPU-side part is therefore only "sample a small texture with linear filtering".

**F18. A sampled mask texture composites with blend factors alone. Tier: Opinion (my inference from F14-F17).**
- *Fog overlay:* draw one black quad whose texture alpha is fog opacity, with normal alpha blend. Use `(ONE, ONE_MINUS_SRC_ALPHA)` if the texture is premultiplied.
- *Multiplicative mask of a rendered layer:* draw the mask quad into the render target with factors `(ZERO, SRC_ALPHA)`. This scales destination colour and alpha by mask alpha, as Phaser 4's `Mask` filter and Pixi's `AlphaMask` do in a shader.
- *Accumulating reveals:* `MAX` blend equation is core in WebGL2 (`gl.MAX`; MDN, C30). It is not core in WebGL1 and not in the universal pipeline's required blend set.

### 3.5 Particles

**F19. Everyone mature simulates 2D particles on the CPU into the sprite batch, and keeps GPU particles as an option. Tier: Practice.**
- libGDX `ParticleEffect.update(delta)` and `draw(Batch)`; `ParticleEmitter` holds a fixed `Particle[]` sized by `setMaxParticleCount`, and switches blend per emitter: `(SRC_ALPHA, ONE_MINUS_SRC_ALPHA)`, additive `(SRC_ALPHA, ONE)`, or premultiplied `(ONE, ONE_MINUS_SRC_ALPHA)`, restoring the batch default afterwards if `cleansUpBlendFunction` (C22).
- Godot 4.7.2: `GPUParticles2D` is the default recommendation, but `CPUParticles2D` is "a CPU-driven option with near-feature parity" with "lower performance when using large amounts of particles", and "may perform better on low-end systems or in GPU-bottlenecked situations". The `CPUParticles2D` class doc says GPU particles "may not run on older devices" (C31).
- Pixi v8 `ParticleContainer`: a restricted container of lightweight `Particle`s. "Dynamic properties are uploaded to the GPU every frame." "Static properties are uploaded only when `update()` is called." It supports no `addChild`. The pipe sets `container.texture ||= children[0].texture`, so by inference all particles in one container share one texture (C32).

**F20. GPU particles on GLES 3.0 use transform feedback and are fragile on real devices. Tier: Practice (Godot source).**
Godot 4.7.2 `drivers/gles3/storage/particles_storage.cpp` calls `glBeginTransformFeedback(GL_POINTS)`. `GLES3::Config` sets `disable_particles_workaround` for renderer strings starting `Adreno (TM) 3`, and `ParticlesStorage::particles_set_emitting` fails with "Due to driver bugs, GPUParticles are not supported on Adreno 3XX devices. Please use CPUParticles instead." A PowerVR path disables the transform-feedback shader cache (C33). Transform feedback is available in WebGL2 (MDN, C34), but it is a GL feature. I did not find a sourced statement about WebGPU (Gaps).

### 3.6 Blend modes and multi-texture batching

**F21. Engines expose a small preset set plus an escape hatch. Tier: Practice.**
- Godot 4.7.2 `CanvasItemMaterial.BlendMode`: MIX, ADD, SUB, MUL, PREMULT_ALPHA (C35).
- Pixi v8.22.0 `BLEND_MODES`: `normal`, `add`, `multiply`, `screen`, `erase`, `min`, `max`, `none`, plus non-premultiplied variants `normal-npm`, `add-npm` (C36).
- libGDX: `Batch.setBlendFunction(src, dst)` and a separate-alpha form (C22).
- Every one of these changes forces a batch break, so draw order, not sorting by blend, decides batch length (`render-performance-and-verification.md` §3.1 F5).

**F22. Multi-texture batching is bounded by sampler counts. Tier: Consensus (secondary, WebGL fundamentals) plus Practice.**
Minimum fragment texture units are 8 in WebGL1 and 16 in WebGL2 (C37), and `@pixi/tilemap` says the WebGL1 minimum of 8 is why it composes layers (C6). KorGE uses up to 4 textures per batch through a per-vertex index (`render-performance-and-verification.md` §3.1 F1). Relevant here because lazily loaded, unpacked client textures (F10) otherwise break batches.

### 3.7 Context loss and what each feature stores

**F23. After loss, textures and buffers are gone and extensions must be re-acquired. Tier: Consensus (spec and platform source).**
WebGL spec, context-lost handling: "resources such as textures and buffers that were created before the context was restored are no longer valid" and "Previously enabled extensions are not restored" (C26). Android: "when the EGL context is lost, all OpenGL resources associated with that context will be automatically deleted" (C27). Consequences by feature:

| Feature state | Recoverable by | Notes |
|---|---|---|
| Atlas texture | re-decode / re-pack from the retained recipe | `asset-pipeline.md` §3.5 |
| Runtime-packed atlas page | the packer's CPU pixmap | libGDX `PixmapPacker` keeps a `Pixmap` per page (C14) |
| Chunk vertex cache (CPU) | not affected | lives in common memory |
| Retained GPU chunk buffer | rebuild from the CPU chunk cache or tile data | cost lands on the first frames after loss |
| Render-target *cache* (baked terrain) | re-render | deterministic from tile data |
| Render-target *accumulator* (fog that remembers) | **not recoverable** | keep the explored grid in CPU memory |
| Visibility texture | re-upload from the CPU grid | trivially recoverable |

## 4. How comparable projects do it

| Project, version | Tile maps | Render targets and masks | Particles | Animation | Source |
|---|---|---|---|---|---|
| libGDX 1.14.2 | `OrthogonalTiledMapRenderer` (re-batch visible) and `OrthoCachedTiledMapRenderer` over `SpriteCache` (cache view + over-cache) | `FrameBuffer(Pixmap.Format, int, int, boolean hasDepth, boolean hasStencil)`, managed | `ParticleEffect`, CPU, per-emitter blend | `Animation<T>(float frameDuration, Array<? extends T>, PlayMode)`; tile animation `AnimatedTiledMapTile` | [renderer](https://github.com/libgdx/libgdx/blob/1.14.2/gdx/src/com/badlogic/gdx/maps/tiled/renderers/OrthogonalTiledMapRenderer.java), [cached](https://github.com/libgdx/libgdx/blob/1.14.2/gdx/src/com/badlogic/gdx/maps/tiled/renderers/OrthoCachedTiledMapRenderer.java), [FrameBuffer](https://github.com/libgdx/libgdx/blob/1.14.2/gdx/src/com/badlogic/gdx/graphics/glutils/FrameBuffer.java) |
| Godot 4.7.2 | `TileMapLayer`, `rendering_quadrant_size` 16 | `SubViewport` + `ViewportTexture`, clear and update modes; `clip_children` alpha mask | `GPUParticles2D` default, `CPUParticles2D` fallback | `SpriteFrames` relative durations x fps | [TileMapLayer.xml](https://github.com/godotengine/godot/blob/4.7.2-stable/doc/classes/TileMapLayer.xml), [SubViewport.xml](https://github.com/godotengine/godot/blob/4.7.2-stable/doc/classes/SubViewport.xml) |
| PixiJS 8.22.0, `@pixi/tilemap` 5.0.2 | `Tilemap`/`CompositeTilemap`, interleaved buffer, GPU animation uniform, UV clamp | `RenderTarget` with Y-flip rule; `cacheAsTexture`; `StencilMask`, `AlphaMask` | `ParticleContainer` with dynamic/static property flags | tile `animX`/`animCountX`/`animDivisor` | [pixijs](https://github.com/pixijs/pixijs/tree/v8.22.0/src/rendering/mask), [tilemap](https://github.com/pixijs/tilemap/blob/v5.0.2/src/gl_tilemap.ts) |
| Phaser 4.2.1 | `TilemapLayer` (CPU) and `TilemapGPULayer` (data texture, one quad) | `Filters.Mask` texture-alpha multiply; `GeometryMask` Canvas-only | not examined | tile animation data texture | [TilemapGPULayer.js](https://github.com/phaserjs/phaser/blob/v4.2.1/src/tilemaps/TilemapGPULayer.js), [Mask.js](https://github.com/phaserjs/phaser/blob/v4.2.1/src/filters/Mask.js) |
| Bevy 0.20.0, `bevy_ecs_tilemap` 0.15.0 | 64x64 chunk meshes, dirty re-upload, per-chunk frustum cull | `Camera { viewport: Option<Viewport>, order, target: RenderTarget }` | not examined | GPU animation by time uniform | [camera.rs](https://github.com/bevyengine/bevy/blob/v0.20.0/crates/bevy_camera/src/camera.rs), [map.rs](https://github.com/StarArawn/bevy_ecs_tilemap/blob/v0.15.0/src/map.rs) |
| KorGE 6.0.0 | `TileMap`: per-texture vertex arrays, version-gated rebuild | `ctx.renderToBitmap(bmp, hasStencil = true, useTexture = ...)` in `View.unsafeRenderToBitmapSync` | not present in this tag | per-frame `duration` in tileset infos | [TileMap.kt](https://github.com/korlibs/korge/blob/v6.0.0/korge/src/korlibs/korge/view/tiles/TileMap.kt) |
| Kool 0.19.0 | not examined | `createOffscreenPass2d` on the backend interface | not examined | not examined | [RenderBackend.kt](https://github.com/kool-engine/kool/blob/v0.19.0/kool-core/src/commonMain/kotlin/de/fabmax/kool/pipeline/backend/RenderBackend.kt) |

Real signatures worth copying:
- Kool: `fun createOffscreenPass2d(parentPass: OffscreenPass2d): OffscreenPass2dImpl`; `class DeviceCoordinates(val ndcYDirection: NdcYDirection, val ndcDepthRange: DepthRange)`.
- Bevy: `pub viewport: Option<Viewport>` separate from `pub enum RenderTarget { Window(WindowRef), Image(ImageRenderTarget), TextureView(ManualTextureViewHandle), None { .. } }`.
- Pixi: `bind({ target, clear, clearColor, frame, mipLevel, layer, flipY })`, an options object, as the pass-begin call.
- libGDX: `Animation<T>.getKeyFrame(float stateTime)` is pure in `stateTime`.

## 5. Pitfalls and regrets

1. **Per-frame tile re-expansion at low zoom.** The cost is proportional to visible tiles, and a camera that moves invalidates any version-gated cache (KorGE rebuilds on every virtual-rect change). Arithmetic, mine: a 1920x1080 view at 0.1x covers 19,200 x 10,800 world px; at 64 px tiles that is about 50,600 tiles, at 32 px about 202,000. At 24 B per vertex (the KorGE batcher format) a vertex-expanded tile is 96 B, so 50k tiles is about 4.8 MB and 202k about 19 MB per frame. The client's `tileSize` is not given in the brief, so this is an illustration, not a measurement.
2. **Caching the whole map.** 80 MB of vertices for a 1000x1000 map at 80 B per tile (F3). Chunks must be lazy and evictable.
3. **A global animation clock.** libGDX `AnimatedTiledMapTile` keeps `lastTiledMapRenderTime` in a static, which makes animation untestable and shared across maps. Time must be an explicit parameter.
4. **Cached renderers silently freezing animated tiles** (inference, F4).
5. **Tile seams** at fractional zoom and under linear filtering (F5). Mipmaps on an atlas also bleed neighbouring regions. That second point is general knowledge, not sourced here (Gaps).
6. **Default stencil is absent on Android and in WebGL** (F16). A mask feature built on stencil fails there silently.
7. **Persistent render-target state.** Godot's `CLEAR_MODE_NEVER` makes an accumulating target easy to build, and it is exactly the state EGL loss and `webglcontextlost` destroy (F23).
8. **Render-target Y flip and double blending** (F13, F14). Each shows up as a visibly wrong first implementation.
9. **Phaser 3's per-mask framebuffers.** Before 3.60, "every unique Bitmap Mask would create two full renderer sized framebuffers" (C24). Mask render targets must be pooled and sized to the masked region, not the surface.
10. **Fog resolution.** Blurring a 128x128 grid harder to hide blockiness "made it too blurry too quickly" (C29). Upscale before smoothing, or sample with linear filtering from a grid that is fine enough.
11. **GPU particles as a platform-specific code path.** Godot disables them on a whole GPU family and caches differently on PowerVR (F20). A capability-gated fallback that is a different simulation is double the maintenance.
12. **`ParticleContainer`-style restricted containers** (no child API) are fast because they refuse features. Do not expose particles as general scene nodes.

## 6. Recommendation for this repo

Frame of reference: the frame-level hybrid from `backend-contract-design.md` §6.1 (`RenderFrame` of ordered passes, `DrawBatch(pipeline, texture, indexRange, clip, blend)`), GL family everywhere (`gpu-api-per-platform.md`), 10k sprites at 60 fps.

### 6.1 The slots, with tier at 1.0

| Slot in the 1.0 contract | At 1.0 | Tier |
|---|---|---|
| **Blend state**: separate colour and alpha factors, equation default `ADD`. Presets `Normal` (straight), `Premultiplied`, `Additive`, `Multiply`. Other equations (`MIN`/`MAX`/subtract) behind a capability key. | Implemented by all three backends and by the recorder; conformance test per preset | Stable |
| **Pass viewport** (pixel rectangle, default full target), distinct from the clip rectangle | Implemented | Stable |
| **Texture store**: sub-region update; sampler state (filter, wrap) in the descriptor; clamp-to-edge default | Implemented (G5 needs them) | Stable |
| **Index format**: 16-bit and 32-bit | Implemented; 32-bit is core on all three GL levels | Stable |
| **Per-vertex texture slot** and `maxTextureSlots` capability | Format reserved (already in `render-performance-and-verification.md` §6); 1 slot required, more optional | Stable (format), optional (count) |
| **`PassTarget`** as an open interface in the pass; the surface is its default value; a backend skips and reports an unknown target | Shape fixed, only the surface implemented | Stable (shape) |
| **Render-target descriptor and handle** (size, format, `depth`, `stencil` flags, doubles as a sampleable texture); `renderTargets` and `maxRenderTargetSize` capabilities; `contentsLost` signal; pass `Load` op; pass `Clear` op | Reserved; capability may be false at 1.0 | **Experimental** (`Clear` Stable) |
| **Retained geometry**: an optional extension (`extension(RetainedGeometry)`) plus capability, and a batch geometry source defaulting to "frame stream", unknown source skip-and-report | Reserved; no 1.0 backend need implement it | **Experimental** |
| **Depth and stencil requests** on surface and render-target descriptors | Field exists, default off, granted-or-not reported by a capability | Stable (field); no feature depends on it |
| **Readback** | Reserved | Experimental (as in §6.6 of that file) |
| **Transform feedback, compute, GPU particle pipelines** | Not reserved | not in the contract |

Contract-wide rules that make the Experimental rows safe:
- The common layer never emits a retained-geometry source or a non-surface target unless the capability is declared (`backend-contract-design.md` §6.4 rule 4).
- Orientation: every texture, including a render target, is top-left origin with y-down, and the backend flips internally (F13).
- Colour storage in a render target is premultiplied, and the blend presets are defined so that a transparent-cleared target composited with `Premultiplied` equals direct drawing (F14).
- After an epoch bump, render-target contents are invalid and the owner is told via `contentsLost`. The contract states that the library never persists game-meaningful state in a target (F23).

### 6.2 Per feature

**R3 sprites, atlases, frame animation. Technique: common code over the universal pipeline. Backend work: none beyond the Stable rows above.**
- `Atlas` is an immutable value: name to `Region` (UV rectangle, trim offsets), with `frames(name)` returning index-ordered regions (libGDX `findRegions`). Lookup is fallible; use `Result` with a documented placeholder-region fallback, matching `asset-pipeline.md`'s white-texture policy.
- `Animation` is an immutable definition: `List<Frame(region, duration)>` plus a `PlayMode`, sampled by a pure `frameAt(elapsed: Duration)`. Per-frame durations, not libGDX's uniform one (F9). Time is always a parameter (pitfall 3). `PlayMode` is a small strategy interface with `Once`, `Loop`, `PingPong`, `Reversed` supplied, not a closed enum, per the library-design rules.
- Add an optional runtime packer (`PixmapPacker` analogue) for lazily loaded, server-named textures (F10). It is the only R3 part that touches the backend: sub-region update, already Stable. It is also the lever that keeps the client's batch count down without waiting for multi-texture batching.
- Reject: baking animation state into the backend or the shape model. It is a pure function of time.

**R4 tile-map layers. Technique: tiered. Tier 1 is common code only, and it ships first.**
- *Tier 1.* The map is a `TileLayer` over `(widthTiles, heightTiles, tileSize, indices, indexToRegion)`; the index-to-region mapping is consumer-supplied (the game's `MapDefinition` has no visual attribute). The layer builds fixed-size chunk vertex arrays (start at 64x64 tiles, a constructor parameter, per Godot 16 and Bevy 64), camera-independent in world space, lazily for visible chunks and evicted by a bounded cache. Per frame it culls chunks against the camera and copies each visible chunk's cached vertices into the frame stream. This costs a memcpy per visible chunk instead of per-tile expansion, and needs no contract change. A dirty-chunk flag rebuilds a chunk on edit. Animated tiles rebuild their chunk on frame tick (KorGE's approach, with an explicit clock), or live as ordinary sprites in a dynamic overlay if they are few.
- *Tier 2, only if the 4c tile scene at the minimum zoom fails budget on a target platform.* Retained geometry: the same chunks as backend buffers, drawn by reference, with the epoch rule rebuilding them from the CPU chunk cache after loss. Exact streaming remains the fallback when the capability is false. **Experimental at 1.0**, graduate when the RTS client has run it.
- *Low-zoom overview.* At 0.1x a 64 px tile is 6.4 px. A consumer-fed overview texture (one texel per tile, a CPU-built bitmap) drawn as one quad would remove the tile cost entirely below a zoom threshold. It also serves the minimap terrain and uses only texture update and sampler state. This is my judgement, not an engine practice. Include it as an option to prototype in the 4c scene, not as a committed design.
- *Seams.* Atlas padding and bleed at pack time; clamp-to-edge; half-texel UV inset available as a `Region` option (F5).
- Reject for R4: the data-texture layer (F6). It needs a per-backend custom pipeline and integer or float data textures, three times, to solve a problem that tier 1 plus tier 2 may already solve. Revisit after R7 gives the repo a supported way to add a pipeline.
- Reject: libGDX's "cache only view plus margin, rebuild at the edge" as the primary scheme. Rebuild hitches at the edge are what chunk caches avoid.

**R5 render targets, several cameras or viewports, visibility mask. Technique: viewports and a texture-based mask first; render targets only where a cache or a real second camera is needed.**
- *Several cameras or viewports.* Each layer in the frame already has a camera and a clip (`room-for-3d.md` §6). Add the pass viewport rectangle. A minimap or inset is a second layer with its own camera and viewport drawn into the surface, which costs a second draw of the content it shows. Bevy keeps viewport and target as separate fields for this reason (F11).
- *Minimap.* The default recipe needs no render target: a screen-layer rectangle showing a consumer-baked terrain texture plus entity dots drawn as shapes. A render target is used when the consumer wants the minimap to be a true camera view, or to cache terrain, and it is included in a frame only when the consumer decides to refresh it.
- *Visibility mask.* A `VisibilityMask` helper in `graphics-core` owns a small RGBA texture whose alpha is fog opacity. The consumer fills a grid (cells, circles or arbitrary data) and the library handles upload, linear upscaling and the overlay draw. Optional smoothing (separable box or Gaussian on the CPU) belongs to the consumer-facing helper, not the backend. The visibility rules, and the explored-state grid, stay in the client as Q2 requires. This is the technique that survives context loss.
- *Render-target uses that remain.* Baked-terrain cache (recoverable), mask-multiply of a rendered layer (blend factors `(ZERO, SRC_ALPHA)`, F18), and per-layer offscreen composition. All are re-renderable.
- *Stencil.* Do not build the mask on it. The 1.0 descriptors carry the request field; the desktop backend gets 8 bits from GLFW by default, Android and web need an explicit request (F16). A consumer wanting a hard-edged non-rectangular clip is a later, separate seam.
- *Capability gating.* `renderTargets` false means the common layer falls back to direct drawing: the cache is skipped and the second camera draws straight into the surface with a viewport. `maxRenderTargetSize` bounds cache tiles (F12).
- R5 is the only phase that adds backend obligations after 1.0. It must ship a render-target conformance suite (clear, draw, sample, orientation, premultiplied composite, loss) for all three backends and the recorder.

**R6 particles. Technique: CPU simulation, quads into the batch. Backend work: none.**
- An immutable `ParticleEffect` definition built with a `@DslMarker` DSL (emitters, rates, lifetimes, curves, blend preset, region or animation) and a mutable `ParticleSystem` runtime with a pre-sized pool per emitter. The runtime is plain CPU state, so it is not `AutoCloseable`: nothing GPU-side is owned.
- One texture (an atlas page) per emitter. This matches Pixi's single-texture `ParticleContainer` and keeps batches long (F19, F22). Additive and premultiplied emitters use the Stable blend presets and each preset change breaks a batch. Group emitters by blend in submission order to bound breaks. Draw order stays layer then submission.
- Particle counts feed the 4c gate: define the scene as "N live particles counted against the 10k budget".
- Reject GPU particles (F20): transform feedback is a GL feature the frame-level contract is deliberately free of; Godot's experience shows the device-fragility cost; and a CPU path is required anyway as the fallback.
- Later optimisation, only if the gate demands it: a Pixi-style split into static and per-frame attributes over retained geometry. It sits behind the Experimental retained-geometry extension and is not scheduled.

### 6.3 Landing order after 1.0

1. **R3** (atlases, animation, runtime packer). No backend change. It also pairs with the sprite-atlas step in `asset-pipeline.md` §6.5.
2. **R4 tier 1** (chunk cache, culling, animated tiles). No backend change. Include the minimum-zoom tile scene in the 4c gate from this phase.
3. **R5** (viewports, render targets, visibility mask). The first and only phase with new backend obligations; graduates render targets from Experimental once the minimap or cache has been built against by the client.
4. **R6** (CPU particles). Independent of R5; the roadmap can swap it ahead of R5 at no contract cost.
5. **R4 tier 2** (retained geometry): when, and only when, a measured 4c scene needs it.
6. **R7** (custom shaders), last as the user decided; the data-texture tile layer becomes a candidate afterwards.

### 6.4 Why this and not the obvious option

The obvious option is to answer D8 with "reserve retained buffers, render targets and stencil, all Stable". I would reject it. Retained buffers are a pure optimisation with an exact streaming fallback, so promising them at Stable freezes a surface nobody has measured a need for. The render-target and mask uses I could source can be met by a CPU-built texture plus blend factors, or are re-renderable caches. Stencil has no R3-R6 consumer. What must be Stable at 1.0 is the small set (blend state, pass viewport, texture sampler state and sub-update, index width, an open `PassTarget`) that every one of those features needs and that is awkward to add once third-party backends exist.

## 7. Load-bearing claims

Each entry: the claim, a verbatim quote, source, version or date. Quotes taken from raw source or documentation are marked "raw". Quotes returned by `WebFetch` (a summarising model) are marked "fetch" and should be spot-checked.

- **C1.** libGDX re-batches visible tiles per frame. Quote (raw): `final int col1 = Math.max(0, (int)((viewBounds.x - layerOffsetX) / layerTileWidth));` and `batch.draw(region.getTexture(), vertices, 0, NUM_VERTICES);`. Source: `OrthogonalTiledMapRenderer.renderTileLayer`, [libgdx 1.14.2](https://github.com/libgdx/libgdx/blob/1.14.2/gdx/src/com/badlogic/gdx/maps/tiled/renderers/OrthogonalTiledMapRenderer.java).
- **C2.** KorGE re-expands with version-gated caching and flags instancing as a TODO. Quote (raw): `// @TODO: Use instanced rendering to support much more tiles at once` and `if (!dirtyVertices && cachedContentVersion == map.contentVersion && cachedAnimationVersion == animationVersion) return`. Source: `TileMap.computeVertexIfRequired`, [korge v6.0.0](https://github.com/korlibs/korge/blob/v6.0.0/korge/src/korlibs/korge/view/tiles/TileMap.kt).
- **C3.** Godot quadrants. Quote (raw): "A quadrant is a group of tiles to be drawn together on a single canvas item, for optimization purposes." and "The quadrant size does not apply on a Y-sorted [TileMapLayer], as tiles are grouped by Y position instead in that case." Source: `TileMapLayer.rendering_quadrant_size`, [Godot 4.7.2-stable](https://github.com/godotengine/godot/blob/4.7.2-stable/doc/classes/TileMapLayer.xml).
- **C4.** bevy_ecs_tilemap chunks. Quote (raw): "Behind the scenes the tiles are split into chunks that each have their own mesh which is sent to the GPU in an optimal way." (README); `pub const CHUNK_SIZE_2D: UVec2 = UVec2::from_array([64, 64]);` and "Larger chunk sizes are better for tilemaps which change infrequently." (`map.rs`); `var current_animation_frame = fract(tilemap_data.time * animation_speed) * frames;` (`tilemap_vertex.wgsl`). Source: [v0.15.0](https://github.com/StarArawn/bevy_ecs_tilemap/tree/v0.15.0).
- **C5.** libGDX cached renderer and SpriteCache. Quote (raw): "Renders ortho tiles by caching geometry on the GPU. How much is cached is controlled by setOverCache(float). When the view reaches the edge of the cached tiles, the cache is rebuilt at the new view position." and "This class may have poor performance when tiles are often changed dynamically, since the cache must be rebuilt after each change." and (SpriteCache) "This information is stored in video memory and does not have to be sent to the GPU each time it is drawn." Source: class docs of `OrthoCachedTiledMapRenderer` and `SpriteCache`, [libgdx 1.14.2](https://github.com/libgdx/libgdx/tree/1.14.2/gdx/src/com/badlogic/gdx).
- **C6.** `@pixi/tilemap`. Quote (raw): "There's also a limitation on 16k tiles per one tilemap. If you want to lift it, please use PixiJS v5.1.0 and following setting `settings.use32bitIndex = true;`" (README); `vec2 textureCoord = clamp(vTextureCoord, vFrame.xy, vFrame.zw);` and `uniform vec2 u_anim_frame;` (`gl_tilemap.ts`); "The minimum supported value is 8, as defined by the WebGL 1 specification." (`Tilemap` class doc); "// if layer was changed, re-upload vertices" (`Tilemap.ts`). Source: [pixijs/tilemap v5.0.2](https://github.com/pixijs/tilemap/tree/v5.0.2).
- **C7.** libGDX tile animation. Quote (raw): `private static long lastTiledMapRenderTime = 0;`, `private int[] animationIntervals;`, `public static void updateAnimationBaseTime () { lastTiledMapRenderTime = TimeUtils.millis() - initialTimeOffset; }`, and the 0 occurrences of "Animated" in `OrthoCachedTiledMapRenderer`. Source: `AnimatedTiledMapTile`, [libgdx 1.14.2](https://github.com/libgdx/libgdx/blob/1.14.2/gdx/src/com/badlogic/gdx/maps/tiled/tiles/AnimatedTiledMapTile.java). The "frame frozen at cache time" reading is my inference.
- **C8.** Half-texel inset. Quote (raw): `final float adjustX = 0.5f / texture.getWidth();` and `final float u1 = region.getU() + adjustX;`. Source: `OrthoCachedTiledMapRenderer.renderTileLayer`, libgdx 1.14.2.
- **C9.** Phaser 4 GPU tilemap layer. Quote (raw): "It has a fixed cost per pixel on screen, whether there is anything in that tile or not. In general, it suffers no performance loss when many tiles are visible." and "A regular TilemapLayer cannot render smooth borders like this, creating sharp seams between tiles." and "Maximum tilemap size of 4096x4096 tiles." Source: `TilemapGPULayer` class doc, [phaser v4.2.1](https://github.com/phaserjs/phaser/blob/v4.2.1/src/tilemaps/TilemapGPULayer.js).
- **C10.** 32-bit indices in WebGL2. Quote (fetch): "In WebGL2, the functionality of this extension is available on the WebGL2 context by default." Source: [MDN OES_element_index_uint](https://developer.mozilla.org/en-US/docs/Web/API/OES_element_index_uint), fetched 2026-10-11. Core in GL 3.3 and GLES 3.0 is my general knowledge, unsourced (Gaps).
- **C11.** libGDX atlas. Quote (raw): `public Array<AtlasRegion> findRegions (String name)` and fields `public int index = -1;`, `public float offsetX;`, `public int originalWidth;`, `public boolean rotate;` Source: `TextureAtlas`, libgdx 1.14.2.
- **C12.** libGDX `Animation`. Quote (raw): `public enum PlayMode { NORMAL, REVERSED, LOOP, LOOP_REVERSED, LOOP_PINGPONG, LOOP_RANDOM, }` and "`@param frameDuration the time between frames in seconds.`" Source: `Animation<T>`, libgdx 1.14.2.
- **C13.** Godot `SpriteFrames`. Quote (raw): "[param duration] specifies the relative duration" and "absolute_duration = relative_duration / (animation_fps * abs(playing_speed))". Source: [SpriteFrames.xml, 4.7.2-stable](https://github.com/godotengine/godot/blob/4.7.2-stable/doc/classes/SpriteFrames.xml).
- **C14.** libGDX `PixmapPacker`. Quote (raw): "The packer supports incremental inserts and updates of TextureAtlases generated with this class." Source: [`PixmapPacker`, libgdx 1.14.2](https://github.com/libgdx/libgdx/blob/1.14.2/gdx/src/com/badlogic/gdx/graphics/g2d/PixmapPacker.java). That it keeps a CPU `Pixmap` per page is my reading of its API, not quoted.
- **C15.** Godot `SubViewport`. Quote (raw): "Update the render target once, then switch to [constant UPDATE_DISABLED]." and "Never clear the render target." Source: [SubViewport.xml, 4.7.2-stable](https://github.com/godotengine/godot/blob/4.7.2-stable/doc/classes/SubViewport.xml); `ViewportTexture` "provides the content of a [Viewport] as a dynamic [Texture2D]".
- **C16.** Bevy camera. Quote (raw): `pub viewport: Option<Viewport>,` and "Cameras with a higher order are rendered later, and thus on top of lower order cameras." and `pub enum RenderTarget { Window(WindowRef), Image(ImageRenderTarget), TextureView(ManualTextureViewHandle), None { ... } }`. Source: [bevy_camera `camera.rs`, v0.20.0](https://github.com/bevyengine/bevy/blob/v0.20.0/crates/bevy_camera/src/camera.rs).
- **C17.** Kool. Quote (raw): `fun createOffscreenPass2d(parentPass: OffscreenPass2d): OffscreenPass2dImpl` and `class DeviceCoordinates(val ndcYDirection: NdcYDirection, val ndcDepthRange: DepthRange)`. Source: `RenderBackend.kt`, [kool v0.19.0](https://github.com/kool-engine/kool/blob/v0.19.0/kool-core/src/commonMain/kotlin/de/fabmax/kool/pipeline/backend/RenderBackend.kt).
- **C18.** libGDX `FrameBuffer`. Quote (raw): "FrameBuffers are managed. In case of an OpenGL context loss, which only happens on Android when a user switches to another application or receives an incoming call, the framebuffer will be automatically recreated." Source: [`FrameBuffer`, libgdx 1.14.2](https://github.com/libgdx/libgdx/blob/1.14.2/gdx/src/com/badlogic/gdx/graphics/glutils/FrameBuffer.java). "Recreated, not restored" is my reading.
- **C19.** Pixi `cacheAsTexture`. Quote (raw): "If your container is too large (e.g., over 4096x4096 pixels), the texture may fail to cache, depending on GPU limitations." and "Cached items are rendered at their actual size, ignoring transforms like scaling." Source: [Cache As Texture guide, pixijs.com repo](https://github.com/pixijs/pixijs.com/blob/main/docs/guides/components/scene-objects/container/cache-as-texture.md), read 2026-10-11 (main branch, not tag-pinned).
- **C20.** Pixi Y-flip. Quote (raw): "WebGL's non-root FBOs carry an inherent Y-flip vs the root (the classic render-texture flip), so the same requested `flipY` lands with the opposite winding depending on `isRoot`." Source: `RenderTargetSystem.isFrontFaceInverted`, [pixijs v8.22.0](https://github.com/pixijs/pixijs/blob/v8.22.0/src/rendering/renderers/shared/renderTarget/RenderTargetSystem.ts).
- **C21.** libGDX FBO tutorial. Quote (fetch): "Enable "pre-multipled alpha" blending, `GL_ONE, GL_ONE_MINUS_SRC_ALPHA`." and "since GL assumes lower-left origin and TextureRegion assumes upper-left" and "this second blend is causing the result to look more transparent than it should." Source: [mattdesl lwjgl-basics wiki, FrameBufferObjects](https://github.com/mattdesl/lwjgl-basics/wiki/FrameBufferObjects), fetched 2026-10-11 (undated wiki; libGDX-era).
- **C22.** libGDX `ParticleEmitter`. Quote (raw): `batch.setBlendFunction(GL20.GL_SRC_ALPHA, GL20.GL_ONE);` (additive), `batch.setBlendFunction(GL20.GL_ONE, GL20.GL_ONE_MINUS_SRC_ALPHA);` (premultiplied), `private boolean additive = true; private boolean premultipliedAlpha = false;`. Source: [`ParticleEmitter.draw`, libgdx 1.14.2](https://github.com/libgdx/libgdx/blob/1.14.2/gdx/src/com/badlogic/gdx/graphics/g2d/ParticleEmitter.java).
- **C23.** Pixi masks and stencil. Quote (raw): "It is more performant than using a `Graphics` mask, but requires WebGL support." (`StencilMask` doc); `renderer.renderPipes.batch.break(instructionSet);` (`StencilMaskPipe.push`); `stencil: true,` and `warn('Provided WebGL context does not have a stencil buffer, masks may not render correctly');` (`GlContextSystem`); "The mask can be inverted, and non-sprite masks are rendered to a texture automatically." (`AlphaMask` doc). Source: [pixijs v8.22.0](https://github.com/pixijs/pixijs/tree/v8.22.0/src/rendering).
- **C24.** Phaser masks. Quote (raw): "GeometryMask is only supported in the Canvas Renderer. If you want to use geometry to mask objects in WebGL, see {@link Phaser.GameObjects.Components.FilterList#addMask}." (`GeometryMask`, v4.2.1); "It multiplies the color and alpha of the input by the alpha of the mask in the corresponding texel." (`Filters.Mask`, v4.2.1); "Previously, every unique Bitmap Mask would create two full renderer sized framebuffers and two corresponding WebGL textures for them." (3.60 changelog, `changelog/v3/3.60/Masks.md`). Source: [phaser v4.2.1](https://github.com/phaserjs/phaser/tree/v4.2.1).
- **C25.** Godot `clip_children`. Quote (raw): "The mask is based on this node's alpha channel: Opaque pixels are kept, transparent pixels are discarded, and semi-transparent pixels are blended in according to their opacity." and "Clipping nodes cannot be nested or placed within a [CanvasGroup]." Source: [CanvasItem.xml, 4.7.2-stable](https://github.com/godotengine/godot/blob/4.7.2-stable/doc/classes/CanvasItem.xml).
- **C26.** WebGL spec. Quote (fetch): `boolean stencil = false;` and "If the value is false, no stencil buffer is available." and "resources such as textures and buffers that were created before the context was restored are no longer valid" and "Previously enabled extensions are not restored". Source: [WebGL 1.0 spec, WebGLContextAttributes and context lost event](https://registry.khronos.org/webgl/specs/latest/1.0/), fetched 2026-10-11 (living spec). The second pair was returned by the fetch tool as a short excerpt; spot-check against the spec's "context lost" section.
- **C27.** Android `GLSurfaceView`. Quote (raw): "By default GLSurfaceView chooses a EGLConfig that has an RGB_888 pixel format, with at least a 16-bit depth buffer and no stencil." and "Note that when the EGL context is lost, all OpenGL resources associated with that context will be automatically deleted." Source: `GLSurfaceView` and `GLSurfaceView.Renderer.onSurfaceCreated` class docs, [AOSP `android-14.0.0_r1`](https://github.com/aosp-mirror/platform_frameworks_base/blob/android-14.0.0_r1/opengl/java/android/opengl/GLSurfaceView.java).
- **C28.** GLFW. Quote (fetch): `GLFW_STENCIL_BITS` "default: 8" and `GLFW_DEPTH_BITS` "default: 24". Source: [GLFW window guide, framebuffer hints](https://www.glfw.org/docs/latest/window_guide.html), fetched 2026-10-11 (docs "latest").
- **C29.** Riot fog. Quote (fetch): "the entire map was covered by a 128x128 grid." and a 256x256 resolution "would have incurred an unacceptable performance hit in the form of a quadrupled CPU cost" and blurring more "made it too blurry too quickly". Source: [Riot Games, "A Story of Fog and War"](https://www.riotgames.com/en/news/story-fog-and-war), fetched 2026-10-11 (article date not stated in the fetch).
- **C30.** `MAX` blend equation. Quote (fetch): "When using a WebGL 2 context, the following values are available additionally: `gl.MIN` ... `gl.MAX`". Source: [MDN `blendEquation`](https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/blendEquation), fetched 2026-10-11.
- **C31.** Godot particles. Quote (raw): "See also [GPUParticles2D], which provides the same functionality with hardware acceleration, but may not run on older devices." (`CPUParticles2D` class doc, 4.7.2-stable). Quote (fetch): "CPUParticles2D is a CPU-driven option with near-feature parity with GPUParticles2D" with "lower performance when using large amounts of particles", and "CPUParticles2D may perform better on low-end systems or in GPU-bottlenecked situations." Source: [Particle systems (2D)](https://docs.godotengine.org/en/stable/tutorials/2d/particle_systems_2d.html), fetched 2026-10-11 (page marked work in progress for 4.7).
- **C32.** Pixi `ParticleContainer`. Quote (fetch, docs): "Dynamic properties are uploaded to the GPU every frame." and "Static properties are uploaded only when `update()` is called." Quote (raw): `container.texture ||= children[0].texture;`. Source: [Particle Container guide](https://pixijs.com/8.x/guides/components/scene-objects/particle-container), and `ParticleContainerPipe`, [pixijs v8.22.0](https://github.com/pixijs/pixijs/blob/v8.22.0/src/scene/particle-container/shared/ParticleContainerPipe.ts). "Single texture per container" is my inference from the second.
- **C33.** Godot GLES3 GPU particles. Quote (raw): `glBeginTransformFeedback(GL_POINTS);` (`particles_storage.cpp`); `if (rendering_device_name.left(13) == "Adreno (TM) 3") { disable_particles_workaround = true;` (`config.cpp`); `ERR_FAIL_COND_MSG(GLES3::Config::get_singleton()->disable_particles_workaround, "Due to driver bugs, GPUParticles are not supported on Adreno 3XX devices. Please use CPUParticles instead.");` Source: `drivers/gles3/storage/`, [Godot 4.7.2-stable](https://github.com/godotengine/godot/tree/4.7.2-stable/drivers/gles3/storage).
- **C34.** Transform feedback in WebGL2. Quote (fetch): "The `WebGL2RenderingContext.transformFeedbackVaryings()` method of the WebGL 2 API specifies values to record in `WebGLTransformFeedback` buffers." Source: [MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebGL2RenderingContext/transformFeedbackVaryings), fetched 2026-10-11.
- **C35.** Godot blend modes. Quote (raw): `BLEND_MODE_MIX` "Colors are assumed to be independent of the alpha (opacity) value."; `BLEND_MODE_PREMULT_ALPHA` "Colors are assumed to be premultiplied by the alpha (opacity) value." Source: [CanvasItemMaterial.xml, 4.7.2-stable](https://github.com/godotengine/godot/blob/4.7.2-stable/doc/classes/CanvasItemMaterial.xml).
- **C36.** Pixi blend modes. Quote (raw): `export type BLEND_MODES = 'inherit' | 'normal' | 'add' | 'multiply' | 'screen' ... | 'erase' ... | 'normal-npm' ... | 'add-npm' ... | 'none' ... | 'min' | 'max';` Source: `src/rendering/renderers/shared/state/const.ts`, [pixijs v8.22.0](https://github.com/pixijs/pixijs/blob/v8.22.0/src/rendering/renderers/shared/state/const.ts).
- **C37.** Texture units. Quote (fetch): the article's table row "min texture units a fragment shader can reference" gives 8 for WebGL1 and 16 for WebGL2. Source: [WebGL2Fundamentals, cross-platform issues](https://webgl2fundamentals.org/webgl/lessons/webgl-cross-platform-issues.html), fetched 2026-10-11 (secondary source).

## 8. Gaps

- **No benchmark of any tile-map technique, anywhere.** Every cost statement in pitfall 1 and in F3 is my arithmetic from stated assumptions. The client's real `tileSize`, maximum map size, and the actual visible-tile count at 0.1x are not in the brief. Whether tier 1 meets the budget on a mid-range Android phone or a laptop iGPU is unknown, which is why tier 2 is gated on a measured scene and not scheduled (N4's lane for the platform numbers).
- **Chunk-cache copy cost is not measured.** "A memcpy per visible chunk is cheap" is reasoning, not evidence. It is the central bet of R4 tier 1.
- **I found no source on Godot's or Bevy's internal batching of retained tile data**, and I did not read Godot's canvas renderer. "Godot retains quadrant draw commands" is implied by the class doc, not verified in the renderer.
- **WebGPU and transform feedback.** I believe WebGPU has no transform feedback (compute replaces it), but I did not source that. It only strengthens the rejection of GPU particles; the rejection already stands on Godot's Adreno experience and on the contract being GL-agnostic.
- **Feedback loops** (sampling a render target while drawing to it) are an error in GL/WebGL. I did not source it. The conformance suite should test that the contract forbids it.
- **Render-target format guarantees.** I did not read the GLES 3.0 or WebGL2 specs for which formats are colour-renderable without extensions. I assume RGBA8 is universal and float formats need `EXT_color_buffer_float`, but the WebGL2Fundamentals page I fetched did not mention that extension. The contract should require RGBA8 only.
- **32-bit indices in core GL 3.3 and GLES 3.0** are general knowledge. Only the WebGL2 side is sourced (C10).
- **Atlas mipmap bleeding** (pitfall 5) is general knowledge, not sourced here. The pack-time padding defaults come from `asset-pipeline.md`.
- **Phaser 4's reason for dropping stencil masks** is not stated in what I read. I report the fact (GeometryMask Canvas-only; texture-alpha `Mask` filter), not a rationale.
- **Particles in Phaser, Bevy and KorGE, and tile maps, masks and particles in Kool, were not established.** KorGE v6.0.0 has no particle code in the repo tree. Kool contributes only the backend signatures (C17).
- **Pixi's masking guide URL returned 404.** The mask findings come from v8.22.0 source, not the guide. Pixi's `cacheAsTexture` and the Pixi particle guide are read from `main` or the live site, not tag-pinned.
- **`WebFetch` summarises.** Quotes marked "fetch" in section 7 passed through a model and may be paraphrased at the edges. C26, C28, C29 and C31 carry the most weight and should be spot-checked.
- **A Khronos forum thread on `glBlendFuncSeparate` for FBO writes (F14)** appeared only as a search snippet and was not fetched. F14 rests on the mattdesl tutorial (C21), which is secondary and undated.
- **Visibility inputs.** The brief's finding that GameTools puts no vision map on the wire, and that `AliveSnapshot` carries no sight radius, means the consumer must produce the visibility grid from its own data. Whether it can is a question for the client, not for this library, and I did not check the client.
- **Idiomatic Kotlin.** Constructs recommended (open interfaces, `@DslMarker` builders, `AutoCloseable` only for GPU-owning types, `Result`) rest on `idiomatic-kotlin.md`; I added no new Kotlin sources. The claim that `PlayMode` should be a strategy interface rather than an enum is a design judgement from the user's library-design rules, not an outside practice (libGDX uses an enum).
