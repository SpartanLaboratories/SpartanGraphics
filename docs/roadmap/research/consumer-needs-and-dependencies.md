# Consumer needs and dependencies

## 1. Question

What do SpartanGraphics' known consumers (SpartanUI, the game client, and through the client, GameTools and GeneralTools) need from it, and by which roadmap phase, so that the roadmap's phase order and dependency notes match reality?

Phase labels used below follow the user's binding order. Version numbers are indicative only.

| Label | Content |
|---|---|
| **P1** | 0.1.0 (split-plan G1..G7, plus the G3 clip rectangle and G5 batching from issue #1) |
| **P2** | real fonts and text input (e.g. 0.2.0) |
| **P3** | Android backend |
| **P4** | web backend |
| **1.0** | stability release |
| **P5+** | the remaining scheduled features |

## 2. Bottom line

- **SpartanUI 0.1.0 is fully served by P1.** Clip rectangle (G3), pointer, scroll and key input (G4), measurement over the bitmap font (G4) and the rectangle outline (G3) cover U1..U6. Nothing SpartanUI plans needs P2 or later. Real fonts, text input, cursor shapes, 9-slice and rounded corners are SpartanUI's *next* releases, not blockers. No SpartanUI text field exists in U1..U6 or in "Not in this plan", so the roadmap's "text input first" is driven by the roadmap itself and the Android work, not by a known consumer.
- **The split plan's PR table has one wrong dependency.** U5 (scrolling) lists only U2, but scrolling needs wheel and drag routing from U3's dispatch and G4's scroll events. The roadmap should say U5 depends on U3 and G4.
- **The game client needs only P1 to be restored (C1..C4).** Everything past that is gated by GameTools' undated Phase 3, which is where terrain, vision, deltas and the event feed first reach the client. The client's tiled terrain, fog of war, event-driven combat sound, minimap and ability UI therefore land in P5+ without blocking anyone, *provided P1..1.0 freeze the right seams*. Those seams are listed in section 7.
- **GameTools' timing is ordered but undated.** Milestones 5.2.0, 5.3.0 and 6.0.0 have no due dates. 5.2.0 is complete (3 of 3 closed); 5.3.0 still has physics (#49) and vision (#50) open; 6.0.0 carries #124 and #132. Phase 3 (#96) has no milestone. The roadmap can say "needed by GameTools Phase N" but must not attach dates to it.
- **Nothing in GameTools, GeneralTools or WebTools plans runs the client on Android or the web.** GameTools is JVM-only (`jvmToolchain(23)`), its transport is raw UDP (no browser path) and no multiplatform or WebSocket/WebTransport issue exists. The Android and web backends therefore have no real consumer. The sample apps (decision 9) are the only proof, and any real-client use is a later, separately prerequisite-gated effort.
- **Three conflicts to put in front of the user:**
  - The split plan says libraries may use GeneralTools types after GeneralTools#7, but decision 11 says libraries never depend on GeneralTools. The client's adapters then do not go away.
  - Audio playback control (loop, stop, volume, handles) is scheduled after 1.0, yet decision 4 freezes the audio backend contract at 1.0, and the client's own sound folders already imply ambient loops and music.
  - The "about 10k sprites" target understates the client's draw count (see 3.6).

## 3. Findings

### 3.1 SpartanUI

Source for the U-stage content throughout: split plan `docs/plans/spartan-libraries-split.md:143-176` (ui-core), `:357-366` (U1..U6), `:381-390` (not in plan). Evidence tier is "codebase fact" unless stated.

- **Clipping (U2 `Group`).** Hard dependency on G3's clip rectangle ([SpartanGraphics#1 comment](https://github.com/SpartanLaboratories/SpartanGraphics/issues/1)). Latest phase: P1. Consequence not stated in the issue: nested groups (a scroll area inside a panel) need nested clips to *intersect*, so the clip contract should be a nested/intersecting clip, not a single replaceable rectangle.
- **Scrolling (U5).** Needs wheel events (G4 abstract input, split plan `:107-110`) and the clip. Latest phase: P1 for wheel and keyboard. Touch drag-scroll needs gestures, which arrive with P3. The U5 row `Depends on: U2` (`:365`) is incomplete. Dispatch (U3) routes the events, so the dependency is U3 and G4.
- **9-slice and rounded corners.** U4's `Panel`, `Button` and `StatBar` are specified from `Box` rectangles (`:154-158`), so they do not need either. Both are in the user's P5+ shapes list. Not a blocker; it is a cosmetic limit of SpartanUI until after 1.0. The shape set is closed (decision 11), so SpartanUI cannot add 9-slice itself. It could emulate it with nine textured rectangles, *only if* the textured rectangle can draw a sub-region of its texture (a source rectangle). The split plan's rectangle (`:100`) has none. Atlases, frame animation and glyph atlases need the same field. Recommendation: put the source rectangle in the shape before 1.0 freezes it.
- **Focus rings.** Outlines exist in P1 (`:105`, `:207-234`), and the plan names focus rings as a later outline user (`:232-234`). Focus and gamepad *navigation* is SpartanUI work, "Not in this plan" (`:387`). Rectangular rings need nothing new. Rings on rounded buttons need the P5+ rounded-rectangle outline shape.
- **Real fonts, measurement, wrapping.** P1's G4 contract includes text measurement over a fixed-size bitmap font (`:114-115`, `:353`), enough for centring and sizing in U4. Wrapping and several sizes arrive with real fonts in P2. Latest phase that does not block SpartanUI 0.1.0: P2. A measurement signature without a font or size parameter would be broken by P2, which is allowed in a 0.x batch but means SpartanUI co-releases.
- **Text input, IME, clipboard.** No U1..U6 element uses it, and SpartanUI has no `TextField` anywhere in its plan. The need appears only when SpartanUI adds one. Latest phase: SpartanUI's `TextField` release; P2 is the user's order. Android needs it designed in P3 at the latest (the soft keyboard).
- **Cursor shapes (hand, I-beam).** Cosmetic. Hand on buttons and I-beam on text fields are desktop and web only. Latest phase: with text input (P2), custom images later.
- **Hover on touch.** The split plan already says touch platforms "simply never fire" hover (`:166`), so SpartanUI needs no graphics feature. The requirement it does create is on G4's pointer event shape: touch must arrive as a primary pointer press, release and move, so SpartanUI's U3 dispatch works unchanged on Android. Decide the pointer event's source/id fields in P1 or accept a co-released breaking change at P3.
- **Screen density.** Outline thickness and text size are screen pixels (`:227`). On Android and high-DPI web, an unscaled 3 px outline or bitmap-size text is wrong. The split plan only separates window and framebuffer size for macOS (`:272`). A density factor on `Surface` is needed by P3 at the latest, and it is cheaper to add while 0.x is open.
- **Image and portrait textures** are looked up by name, so SpartanUI's `Image` needs the "unloaded or missing texture draws a fallback, never throws" behaviour that `TextureCache` has today (`GameGraphics/.../TextureCache.kt:36-52`: white fallback, logged once). That is also what async web loading needs (see 3.3).
- **Cadence consequence.** `ui-core` has `api(graphics-core)` (`:64-66`). Every breaking graphics 0.x forces a SpartanUI minor. Evidence tier: consequence of the plan's own dependency declaration.

### 3.2 The game client today

Sources: `GameGraphics/src/main/kotlin/com/spartanlabs/Main.kt`, `README.md`, `graphics/Window.kt`, `docs/plans/camera-follow-selection.md`.

- **Draws:** textured, tinted, optionally rotated rectangles with alpha (actors, the huge floor and graveyard backdrops, health bars as sub-objects, click markers, selection outline), through one camera with pan, zoom (`MIN_ZOOM = 0.1`, `Window.kt:785`) and edge panning (`EDGE_PAN_THRESHOLD_PX = 50`, `:791`). Text is STBEasyFont only.
- **Rendering semantics the client relies on:**
  - Draw order is the server's list order. The server adds the floor first "so clients draw them behind everything else" (`MyGameServer/src/main/kotlin/Main.kt:263-271`).
  - World y grows downward and angles are degrees, clockwise-positive (`GameGraphics/.../Window.kt:626`; `MyGameTools` plan 86 `:290`).
  - Sub-objects are drawn recursively with the parent (`Window.kt:646`).
  - Texture names are server-chosen strings, some with spaces (`"natures prophet.png"`), and unknown names fall back to a white texture.
- **UI:** panels, labels, a stat bar, a portrait, two buttons, a viewport as the back-most unhandled-input sink, all in fractions of the window.
- **Camera follow (`camera-follow-selection.md`).** Needs only camera centre-on and pan operations, which G2's `Camera` provides. C3 either rebuilds it on `Camera` or retires it. No graphics-library feature is missing.
- **Networking is out of the library's scope** and stays in the client until GameTools#132.

### 3.3 Client assets are named by the server, and the names are not safe or enumerable

Tier: codebase fact plus inference.

- The `texture` string arrives in every snapshot (`GameTools .../VisibleObject.kt:203-213`), so the client cannot preload "all textures" ahead of time. Preload groups work only for client-owned assets (UI, markers, sounds); server-named sprites need lazy, on-demand loading with a fallback, which is today's behaviour. The asset manager (scheduled in P5+) must keep that path.
- Names contain spaces and, for sounds, `/` (`"ui/micro_button-click.mp3"`, `Main.kt:37`). A web `AssetSource` over HTTP must percent-encode, and the name grammar should allow subfolders.
- A hostile or buggy server can send `"../../x"`. With a filesystem or HTTP `AssetSource`, names from the wire are untrusted input. Inference from the code, not a documented requirement: the `AssetSource` contract should state that a name is a relative, normalised key and implementations must reject traversal.
- Texture formats in use: PNG, JPEG and a static GIF (`textures/` folder). All three are decodable by Android and browser decoders.

### 3.4 Client audio

Source: `GameGraphics` working tree (uncommitted), `Main.kt`, `audio/SoundPlayer.kt`.

- **Use today:** one fire-and-forget call, `sounds.play("ui/micro_button-click.mp3")`, from right-click orders and the demo button (`Main.kt:137,149,160,170,178,297`). No loop, stop, volume or handle. Every play starts its own thread from a cached pool, so concurrency already exists implicitly.
- **Untracked sound folders (all MP3, 14 files):**
  - `ui/` (5 files, including the relocated `beep-07a.mp3`);
  - `combat/` (`air_swoosh`, `heavy_tink`, `metal_tink`);
  - `ambient/` (`background_wind`, `background_chirping_birds`, about 240 KB each);
  - `atmosphere/` (`fate_approaches`, 721 KB);
  - three loose files (`level_up`, `magic_reveal`, `transition`).
  Only `ui/micro_button-click.mp3` is referenced by code.
- **Reading of the folders (inference):** four categories (ui, combat, ambient, atmosphere/music), which map onto buses; `ambient` and `atmosphere` are the loop and streamed-music cases. Durations were not measured.
- **Event-driven sound is gated upstream.** `combat/` sounds want `AttackLanded` and `DamageDealt` events at a position. The only wire-side plan is the client event feed *draft* (`MyGameTools docs/plans/96-phase-3-authoritative-networking/client-event-feed-plan-draft.md`), "not to be planned" until the WebTools adoption (#124), reliability still open (§3 E), and the default projector depends on bugs #81 and #82. Until then the client can only play sounds from its own actions (as now) or infer them from state diffs. Positional audio therefore has no data source before GameTools Phase 3. Its listener point would be the camera centre.
- **Working-tree state:** `SoundResourcesTest` still asserts `/sounds/beep-07a.mp3`, which the working tree moved to `ui/`. The test fails until the C-phase port fixes it.
- **Audio contract timing (conflict).** Decision 4 freezes the audio backend at 1.0, but loop, stop, pause, volume and handles are listed as post-1.0 features. The frozen contract must already express handle-based playback (a returned playback handle with stop, pause, volume, loop) or those features become breaking changes. Tier: inference from the decisions plus the client's folder structure.

### 3.5 What GameTools' roadmap implies the client must render

Sources: `MyGameTools docs/framework-vision-and-roadmap.md` §3 and the issues cited.

| GameTools phase / issue | State | What reaches the client | Client rendering consequence |
|---|---|---|---|
| Phase 1: map (#46), zones (#47), spatial index (#48) | closed, milestone 5.2.0, unreleased (latest tag `v5.1.0`) | Nothing: "terrain reaches the client only in Phase 3" (plan 86 `:585`; `46-map-model/plan.md:546`) | None. The server can already hold a `TiledMap`. |
| Phase 1: physics (#49), vision (#50) | open, 5.3.0 | Nothing: "no change to `GameServer.broadcast`" (#50) | None. |
| Phase 2: combat (#71), `ExperienceSystem` (#78), WorldSystem graduation (#79), webtools-udp 2.0 (#124), `GameClient` (#132) | open, 6.0.0 | New wire format; `GameClient.latestState()`/`onState` still delivers decoded `STATE`. Respawn and death are existing snapshot changes. | Adapter input type is unchanged. `NetworkClient`, `ProtocolParsing` and `ClientCommands` are deleted (split plan `:377`). |
| Phase 3: codec (#97), stable-ID deltas (#98), action-map input (#99), interest filtering (#101), sessions (#102), event feed (draft) | open, **no milestone** | Terrain; filtered entity sets; deltas keyed by `EntityId`; an `EVENT` stream | The big step for the client; see rows below. |
| Phase 4: AI (#104..#109) | open | Nothing new. More moving entities. | Possible debug overlays (inference, not stated). |
| Phase 5: persistence (#110..#115) | open | Nothing new rendering-wise | Zone streaming of terrain is unaddressed (Gaps). |
| Phase 6: abilities (#117), items (#118) | open | `BuffSnapshot` already rides `GameObjectSnapshot` (`Buff.kt:55-70`). Ability and item state is undefined. | Section 3.7. |
| Phase 7: scale (#119..#122) | open | 10k entities at 10-20 Hz is the server medium target | Section 3.6. |

- **Terrain (#46).** `MapDefinition` is `widthTiles`, `heightTiles`, `tileSize`, a flat row-major `tiles` index list, a `terrainPalette` of `TerrainTypeSnapshot(walkable, movementCost, heightLevel, blocksVision)`, obstacle boxes and spawn points (`gametools-world .../MapDefinition.kt`). **There is no texture, colour or visual attribute at all.** The client must own a palette-index-to-tileset mapping. The tile-map layer's input should therefore be (grid dimensions, tile size, flat indices, a client-supplied index-to-region mapping). Obstacles are axis-aligned boxes (drawable as rectangles). Tile-layer culling and batching matter: today's terrain is two giant stretched sprites, and a real map multiplies that into tens of thousands of cells at the minimum zoom of 0.1.
- **Zones (#47).** Server-side interest and persistence scope. No rendering need, apart from optional debug lines and boxes.
- **Vision and fog of war (#50, #101).** What the client receives is, on the evidence, *only the filtered entity list* ("the authoritative broadcast sends each player only what they can currently see", roadmap decision 16). GameTools says it will not do "any fog-of-war state machine" (#50). Vision *maps*, sight radii and explored-area state are not on the wire in any plan. A client-side fog layer would have to build its own mask from visible own-team entities, which needs the sight radius, which `AliveSnapshot` does not carry. So the SpartanGraphics visibility mask must accept an *arbitrary consumer-built mask* (bitmap, cell grid or set of circles) and must not assume a GameTools format. Open question for GameTools, see Gaps.
- **Projectiles.** Ordinary `VisibleObject`s with `angle` and `turns` (existing `fireball.png`). P1 already covers them; many projectiles stress batching.
- **Status effects.** Buff name and remaining ticks are on the snapshot. Rendering icons above units needs sprites from an atlas plus text or bars. P5+.
- **Items and inventory.** UI work: grid containers and drag-and-drop in SpartanUI (layout containers are "Not in this plan", `:387`) plus atlas icons from graphics. P5+, gated by Phase 6.
- **Abilities.** Cooldown sweeps, cast bars (a `StatBar`), and targeting indicators for the roadmap's targeting types (self, unit, ground, skillshot, aura) need circles, lines or polygons and alpha. These are the P5+ shapes. Floating damage numbers need world-layer text with alpha, driven by `DamageDealt`, so they also wait on the event feed.
- **Minimap and fog.** Render-to-texture plus the visibility mask (P5+). Their data comes from the client's own entity store, so no GameTools prerequisite beyond Phase 3's filtered sets.

### 3.6 Scale: what "10k sprites" means for the client

- Tier: codebase fact plus arithmetic.
- Every `Alive` carries a `healthBar` sub-object that is drawn as its own quad (`Alive.kt:24,428-446`), and `VisibleObjectSnapshot.subObjects` flatten recursively. 10k alive entities are therefore about 20k quads, plus outlines. The 4c gate should count draw items (quads and glyph runs), or state that 10k means entities with one sub-object each.
- The GameTools medium target is 10k entities *server-wide*. Per-player filtering (#101) is what bounds the client's receive set, and it has no milestone. Until then the all-to-all broadcast sends everything. At the client's minimum zoom of 0.1 an RTS view can show most of a large map.
- The live client cannot prove the target. `MyGameServer` spawns 10 zombies plus 1 `Alive` per player (`MyGameServer/.../Main.kt:30,260`). The 4c gate needs a synthetic scene in the graphics repo or a stress-populated server, not the client.
- With deltas keyed by `EntityId` (#98), the client holds persistent per-entity state. Rebuilding 10k shape objects per frame creates garbage; the adapter would prefer updating retained shapes in place. The shape model should allow cheap in-place mutation. Tier: inference, for the render-performance researcher.
- After deltas, list order no longer carries draw order. The client needs an explicit layer or depth key per shape (and probably a sort key for top-down units). The reserved depth in the 2D API (decision 7) covers this. Inference.

### 3.7 Upstream changes

- **GeneralTools#7 (multiplatform; open, no milestone).** SpartanGraphics is not blocked: the libraries declare their own `Color`, `Point`, `Size`, `Rect` contracts (split plan `:86-88`). The client's adapters read GameTools *snapshot DTOs* (`ColorSnapshot(r,g,b,a)`, `PointSnapshot`, `DimensionsSnapshot`, which are GameTools types) and GeneralTools `Color`/`Square` in `Main.kt`. GameTools stays JVM-only unless it goes multiplatform, which no issue plans, so even a multiplatform GeneralTools removes at most the GeneralTools half of the adapters. It is also a major GeneralTools bump (plain-Maven consumers would need `GeneralTools-jvm`). **Conflict:** the split plan says libraries may use GeneralTools types directly afterwards (`:88`, `:186`), but decision 11 says never. Under decision 11 the adapters are permanent. GeneralTools#8 (read-only views of `Point` and `Square`) shows the same immutability pressure as the draft architecture's "final immutable types" proposal.
- **GameTools#132 (`GameClient`).** Decided 2026-10-08, ships with 6.0.0 in a new client-safe `gametools-protocol` module. No SpartanGraphics change. For the client: the `NetworkClient` row in split-plan Phase 3 "Later" (`:377`) becomes concrete, and the adapter input stays a decoded `List<DrawableSnapshot>` until Phase 3 replaces `STATE` with the codec and deltas. Isolating the adapter behind "an entity state source keyed by `EntityId`" lets deltas swap in. Each Phase 3 item updates `GameClient` in the same PR as the server half, so the client no longer mirrors protocol changes by hand.
- **GameTools#124 (webtools-udp 2.0.0).** Breaking wire change in 6.0.0; "merge after the last 5.x feature release". The client is pinned to GameTools 5.0.0 (`build.gradle.kts:38`), so its interim `NetworkClient` breaks the moment the server moves to 6.0.0. The split plan's risk list (`:403-406`) already names the three options; the key new fact is that 5.3.0 still has two open issues (#49, #50), so 6.0.0 is not imminent. No effect on SpartanGraphics.
- **Android or web client?** None of GameTools' documents, plans or issues mention Android, web, wasm or multiplatform (searched `docs/` and `docs/plans/`). `gametools.kotlin-library` sets `jvmToolchain(23)`. `webtools-udp` is raw UDP, and browsers expose no raw UDP socket; WebTransport and WebRTC data channels are the nearest options and are not UDP sockets ([Chrome: Using WebTransport](https://developer.chrome.com/docs/capabilities/web-apis/webtransport)). WebTools has no WebSocket or WebTransport issue. Android can send UDP but would have to consume JVM-23 bytecode (Gaps). So a real-client Android or web port needs, at minimum: a multiplatform GameTools client protocol (starting with GeneralTools#7, `gametools-protocol`), and for web a browser-capable transport in WebTools. All are unscheduled.

## 4. How comparable projects do it

Not researched; the question is about this ecosystem's own consumers. See sibling reports (1, 5, 11) for external designs.

## 5. Pitfalls and regrets

- **Building a feature for a consumer that does not exist.** Text input, IME, clipboard and cursor shapes have no consumer until SpartanUI adds a `TextField`. The user has ordered them first; the justification is the Android soft-keyboard design, not a consumer. Say so in the roadmap, so P2 is not described as "SpartanUI needs it".
- **Freezing a contract before the consumers have used it.** Decision 4 freezes pointer, text-measurement and audio contracts at 1.0. The consumers that exercise them are SpartanUI (pointer, measurement, clip) and the client (audio, textures). Neither will have used touch, fonts or loops before P3 unless the P1 shapes are chosen with them in mind (3.1, 3.4).
- **The only desktop proof is the live client, and it is a 10-entity demo.** Do not let "verified through C1" stand in for the 4c perf gate or for tile/mask/atlas behaviour.
- **Assuming vision data reaches the client.** GameTools' fog-of-war is a server filter; nothing guarantees a client-usable vision map. A visibility mask designed around one assumed format would be wrong.
- **Terrain with no visual data.** `TerrainType` has no texture or colour. A tile-map layer that expects the map to say how it looks will not fit GameTools' map model.
- **Loose coupling of version cadence.** `api(graphics-core)` means every breaking graphics 0.x also breaks SpartanUI. Batched phase-boundary 0.x releases help only if SpartanUI ships a matching minor at each boundary.

## 6. Dependency table

"Latest phase" is the last phase the feature can land without blocking the consumer; "Roadmap" is where the user's order currently puts it. A mismatch is marked **!**.

### 6.1 SpartanUI

| Consumer need | SpartanGraphics feature | Latest phase / roadmap | Upstream prerequisite |
|---|---|---|---|
| Group clipping | Clip rectangle with nested intersection (G3) | P1 / P1 | none |
| Scrolling (wheel) | Scroll events (G4) + clip | P1 / P1 | none; fix U5 to depend on U3 and G4 |
| Scrolling (touch drag) | Touch and gestures | P3 / P3 | Android backend |
| Panels, buttons | Rectangle, textured rectangle | P1 / P1 | none |
| 9-slice, rounded corners | 9-slice and rounded-rectangle shapes; texture source rectangle | cosmetic, none blocking / P5+ | source rectangle must be in the shape before 1.0 |
| Focus ring (rectangular) | Opt-in outline (G3) | P1 / P1 | focus navigation itself is SpartanUI-side |
| Focus ring on rounded controls | Rounded outline shape | SpartanUI's choice / P5+ | rounded rectangle |
| Label centring | Text measurement over the bitmap font (G4) | P1 / P1 | none |
| Real fonts, wrapping, sizes | Real fonts | P2 / P2 | measurement signature must take font and size |
| Text field input, IME, clipboard | Text input | at SpartanUI's `TextField`; roadmap P2 / P2 | no `TextField` in SpartanUI's plan |
| Hand and I-beam cursors | Cursor shapes | P2 / P2 | desktop and web only |
| Hover on touch | none (never fires); touch delivered as a primary pointer | P3 / P3 | pointer event carries source and id from P1 |
| Density-correct text and outlines | Surface density factor | P3 / not scheduled **!** | none |
| `Image` by name, missing texture | Fallback texture, never throws | P1 / P1 | none |

### 6.2 Game client

| Client need | SpartanGraphics feature | Latest phase / roadmap | Upstream prerequisite |
|---|---|---|---|
| Textured, tinted, rotated, alpha rectangles, y-down, submission-order draw | Shapes, world layer, camera (G2, G3, G5) | P1 / P1 | none |
| Large backdrops (6000 px) with culling | Camera visible region and culling (G2, G3) | P1 / P1 | none |
| Sub-object health bars | Adapter flattens sub-objects into shapes | P1 / P1 | none |
| Selection outline | Default outline (G3) | P1 / P1 | none |
| Picking, click markers, edge panning, camera follow | Picking, camera ops, abstract input | P1 / P1 | none |
| Inspector UI | SpartanUI 0.1.0 | P1 / P1 | U4, G7 |
| Single sound effects by name | `audio-desktop` (G6) | P1 / P1 | none |
| Ambient loops, music, per-category volume | Loop, stop, volume, mixing buses, streamed music | the client's own choice; user order P5+ **!** | audio contract must carry playback handles by 1.0 |
| Combat and event sounds at a position | Positional 2D audio (listener = camera centre) | P5+ / P5+ | GameTools Phase 3 event feed (no milestone), bugs #81 and #82 |
| Readable UI text, nameplates | Real fonts | P2 / P2 | none |
| Name entry, chat | Text input | no client plan / P2 | none |
| Tiled terrain | Tile-map layer (input: grid, tile size, flat indices, client palette-to-region map) | P5+ / P5+ | GameTools Phase 3 sends the map; the map carries no visuals |
| Fog of war, minimap | Visibility mask (consumer-built mask) + render-to-texture | P5+ / P5+ | Phase 3 interest filtering (#101); whether vision data or sight radii reach the client is undecided |
| Projectiles | Rotated sprites | P1 / P1 | none |
| Unit animation, status icons | Atlases, frame animation | P5+ / P5+ | snapshot has only an `intent` tag; animation choice is client policy; texture-name lookup should resolve to an atlas region |
| Ability cooldowns, targeting indicators | Circles, lines, polygons, alpha | P5+ / P5+ | GameTools Phase 6 (#117) |
| Floating combat text | World-layer text with alpha | P2 for text, P5+ polish / P2 | event feed for `DamageDealt` |
| Inventory grid, tooltips | SpartanUI layout containers; atlas icons | P5+ / P5+ | GameTools Phase 6 (#118); SpartanUI layout not in plan |
| 10k-entity scenes (about 20k quads) | Batched renderer (G5), in-place shape update, 4c gate | P1 / P1 | #101 bounds the receive set; client cannot prove the gate |
| Explicit draw order after deltas | Layer or depth key per shape | P1 (shape design) / P1 | #98 removes list-order draw order |
| Networking replacement | none | n/a | #124 + #132 (6.0.0) |
| Adapters for GeneralTools types | none; permanent under decision 11 | n/a | GeneralTools#7 helps only if GameTools goes multiplatform |
| Client on Android or web | Android and web backends | no consumer / P3, P4 | multiplatform GameTools protocol; a browser transport in WebTools; none planned |

## 7. Recommendation for this repo

1. **Keep P1 as scoped; add three contract details to it** because SpartanUI's U2..U3 and the client's C1 are built on them and the 1.0 freeze will inherit them:
   - nested, intersecting clips;
   - a pointer event that carries source and id (touch becomes a primary pointer);
   - an explicit layer or depth key plus a texture source rectangle on the textured rectangle.
2. **Describe P2 honestly in the roadmap.** Fonts serve SpartanUI and the client's readability. Text input, IME, clipboard and cursors have no consumer today; they are scheduled for the Android soft-keyboard design and SpartanUI's future `TextField`. Do not mark SpartanUI "blocked on P2".
3. **Correct the PR table.** U5 depends on U3 and G4. Record that `api(graphics-core)` means each breaking graphics 0.x boundary pairs with a SpartanUI minor.
4. **Pull the audio contract's playback handle forward.** Put handle-based playback (stop, pause, volume, loop) into the audio contract before 1.0, even if mixing, streaming and positional audio are implemented after. The client's four sound categories are the evidence. Put the decision to the user (the roadmap order cannot be both "contract frozen at 1.0" and "control features after 1.0" without it).
5. **Resolve the GeneralTools conflict explicitly.** State that under decision 11 the client's adapters are permanent, and amend the split plan's "go away when #7 lands" line. The libraries' basic types are graphics-owned for good.
6. **Mark the client's Phase 3-gated features as P5+ with the GameTools dependency** (tile terrain, fog and minimap, event-driven positional audio, ability and item UI) and write the data-shape assumptions into the feature designs: map = flat indices with a client-side visual mapping; visibility mask = consumer-built; events = client-supplied position.
7. **Treat Android and web as sample-app-proven only.** Record in the roadmap that no consumer exists for either, and list the prerequisites for a real client (section 3.7) as an unscheduled follow-on, not a dependency of any phase.
8. **Do not use the client as the 4c proof.** The gate needs a synthetic scene in this repo (about 20k quads, counting sub-objects).

## 8. Gaps

- **Audio durations** of the ambient and music files were not measured, so "needs streaming" is a reading of file sizes and names (about 240 KB and 721 KB), not a fact. The author's intent for the four sound folders is inferred; they are uncommitted.
- **What GameTools Phase 3 puts on the wire for vision** (sight radii, vision maps or only filtered entities) is not specified: #101 is a one-line issue. This decides the shape of the fog layer's input. It is a question for GameTools, not for SpartanGraphics. Blocks the visibility-mask API detail only, not 1.0 scope.
- **Terrain delivery format and zone streaming** are unspecified ("terrain reaches the client in Phase 3"). Large MMO-lite maps imply chunked delivery; nothing in the plans addresses it.
- **Ability and item snapshot shapes** (Phase 6) are undefined, so the UI and atlas needs in 3.5 are inferences from the roadmap prose.
- **Android consuming JVM-23 bytecode** was not checked (the Android toolchain's supported bytecode level is unverified here). Do not treat "Android can run GameTools" as established.
- **MP3 loop behaviour** (gap at loop point for encoded MP3) is general knowledge I did not verify against a source. It matters if ambient loops stay MP3 (the contract already lists WAV and OGG).
- **GameTools dates**: no milestone has a due date, and 5.2.0's three closed issues are unreleased (CHANGELOG `[Unreleased]`), so "needed by Phase N" cannot be converted to calendar time.
- **SpartanUI's own roadmap beyond U6** does not exist; "latest phase" for its future needs is therefore against the user's graphics order, not against a SpartanUI plan.
- **Not read in full:** the draft `docs/plans/1-phase-1-spartangraphics/architecture.md` (only grepped), `Window.kt` and `UiRenderer.kt` beyond the cited lines, and the GameTools #124 `plan.md`.
