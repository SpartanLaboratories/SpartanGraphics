# Research: text rendering (question 8)

## 1. Question

How should SpartanGraphics render real text (TTF/OTF, runtime glyph atlas, kerning, wrapping, several sizes) on desktop JVM, Android and web? How should it leave room for later international text (CJK-scale glyph sets, shaping, right-to-left)? What must Phase 1's `Text` shape and text-measurement contract look like so real fonts do not break them?

## 2. Bottom line

- **Split the work: layout in common code, pixels per backend.**
  - `graphics-core` (common code) owns font-table reading (cmap, advances, `kern`/GPOS pairs), layout (wrapping, alignment, fallback, kerning), the glyph cache and atlas pages. Because it owns layout, text wraps and measures identically on every platform and can be tested without a GPU.
  - Each backend supplies only a narrow glyph rasteriser: a glyph id and a size go in, a coverage bitmap and its metrics come out.
  - Do not let each platform's text API (Android `Paint`, Canvas2D) decide layout. Wrapping would differ per platform, and the Phase 1 `TextMeasurer` contract would stop being deterministic.
- **Layout is zoom-independent and pixel-density-independent.** Measure and wrap in layer units using unhinted advances. Rasterise at `size * pixelScale`, where `pixelScale` is the DPI scale times the camera zoom for world text. Otherwise world-layer text reflows while the camera zooms, and the layout cannot be a pure function.
- **Two glyph modes behind one atlas.**
  - *Raster*, at an exact or bucketed pixel size, is the default and gives the sharpest small text.
  - *SDF*, opt-in, is for world-layer text that zooms. Generate it in common code by running a distance transform over a high-resolution bitmap supplied by the backend. It also gives outlines and shadows.
  - *MSDF/MTSDF* is produced offline by the Gradle plugin (msdf-atlas-gen) and loaded by a "baked font" provider that implements the same glyph-provider interface.
- **Phase 1 must ship three things so real fonts are additive.**
  - `Text` takes a `TextStyle` value object (even though v1 ignores most fields) and an opaque `Font` handle (default: the built-in bitmap font).
  - `TextMeasurer` measures `(text, style, maxWidth?)` synchronously and returns a result that separates line-box height from ink bounds, and carries ascent and baseline.
  - It returns a layout result rather than a bare width, so wrapping, carets and RTL do not need a second API.
  - The monolith already shows the cost of omitting these. It centres text with `stb_easy_font_width * TEXT_SCALE` and a magic `GLYPH_HEIGHT_PX = 7.0`, because the line box (about 12 px) is not the ink height.
- **Keep international text additive through a `TextShaper` seam.**
  - The interface takes text, font runs, direction and locale, and returns glyph runs (glyph id, cluster, advance, offset, direction).
  - The default is a simple LTR cmap-plus-kerning shaper, the same split Godot ships as `TextServerFallback` versus Advanced.
  - HarfBuzz-backed shapers come later and are already close on desktop: LWJGL has shipped `lwjgl-harfbuzz` and `lwjgl-freetype` since 3.3.2, and the project is on 3.3.4.
  - Ship this seam at 1.0 as **Experimental**, since no consumer will have built against it. Freeze the layout *result* shape (glyph runs with cluster mapping, in visual order, with per-run direction) at 1.0 as Stable Core, because carets and hit-testing depend on it.
- **Landing order.**
  1. Phase 1 contracts as above.
  2. Common-code font reading, layout and atlas, plus the desktop rasteriser (stb_truetype).
  3. Android rasteriser.
  4. Web rasteriser.
  5. 1.0 freeze of `Font`, `TextStyle`, `TextLayout` and `TextMeasurer`, with the glyph rasteriser contract following the other backend contracts.
  6. Gradle-plugin font baking.
  7. Shaping and international text (unscheduled).

## 3. Findings

Evidence tiers: **Consensus** = several independent credible sources or a spec; **Practice** = what mature libraries visibly do; **Opinion** = one source's view; **Judgement** = my synthesis, not sourced.

### 3.1 Rasterisation options per platform

1. **stb_truetype (in LWJGL 3.3.4 as `org.lwjgl.stb.STBTruetype`). Practice.**
   - Pros:
     - It parses TrueType and CFF-based OpenType and renders antialiased one-channel bitmaps.
     - It renders single-channel SDF bitmaps via `stbtt_GetCodepointSDF`.
     - It exposes pair kerning through `stbtt_GetGlyphKernAdvance`, with GPOS pair kerning since v1.19.
     - It is already a dependency, since STBEasyFont comes from the same family.
   - Cons:
     - It has no hinting and no shaping.
     - The "query whole kerning table" API reads the `kern` table only, not GPOS.
     - The header states the library "does no range checking of the offsets found in the file, meaning an attacker can use it to read arbitrary memory". That matters if fonts ever come from untrusted sources (user mods or downloads).
   - Its own docs say that because the output is antialiased but not hinted, subpixel positioning "is very import for quality".
   - Source: [stb_truetype.h](https://raw.githubusercontent.com/nothings/stb/master/stb_truetype.h) (v1.26 header, read directly).
2. **FreeType and HarfBuzz via LWJGL. Consensus on existence.**
   - The bindings were added in LWJGL 3.3.2 and refreshed in 3.4.x. The default HarfBuzz build is standalone, and FreeType interop is opt-in.
   - Sources: [LWJGL 3.3.2 notes](https://newreleases.io/project/github/LWJGL/lwjgl3/release/3.3.2), [3.4.2 notes](https://newreleases.io/project/github/LWJGL/lwjgl3/release/3.4.2).
   - FreeType gives better hinting, strokers for outlines, and colour glyphs. libGDX's `FreeTypeFontGenerator` exposes border, shadow and kerning options, which are FreeType features.
   - FreeType is desktop-only in practice for this repo. Android would need an NDK build, and web would need a Wasm build.
3. **Android. Practice, partly unverified.**
   - Android's `Paint`/`Canvas` stack shapes with HarfBuzz internally (one source, [Medium](https://medium.com/on-coding/androids-font-renderer-c368bbde87d9)).
   - It handles complex scripts and system-font fallback for free.
   - Glyph-level access (glyph ids) exists only on newer API levels (`android.graphics.text.TextRunShaper`). I could not confirm its API level; see Gaps.
   - Loading from in-memory bytes is awkward. `Typeface.Builder` takes a file, descriptor or asset, and I believe it has no `ByteBuffer` constructor. That is unconfirmed.
   - Game engines that need control use an NDK FreeType+HarfBuzz pipeline into a texture atlas (one reported example, same Medium search result).
4. **Web (browser). Consensus.**
   - Canvas2D and `OffscreenCanvas` `fillText` and `measureText` are widely available since March 2023, including in workers ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvasRenderingContext2D)).
   - A browser font backend that draws each glyph to a canvas, measures it with `actualBoundingBox*`, and packs it into an atlas is established practice. Ghostty does this. Sizing the canvas needs care because emoji have no bounding box.
   - The browser gives no glyph ids, so shaped *runs* can only be rasterised as whole strings. Per-glyph atlases are therefore only available through the simple cmap path or a Wasm shaper.
   - `Intl.Segmenter` (Baseline 2024) gives grapheme, word and sentence segmentation, **not** line-break opportunities ([MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Segmenter)). Web line breaking beyond whitespace needs our own code.
5. **Wasm HarfBuzz and FreeType. Practice.**
   - harfbuzzjs exists. A 2019 issue reports builds around 128 KB compressed after feature cuts ([harfbuzz#1633](https://github.com/harfbuzz/harfbuzz/issues/1633)); the figure is old.
   - I found no FreeType-Wasm evidence.
6. **Kotlin libraries. Opinion, weak.**
   - [harfbuzz-kmp](https://klibs.io/package/com.mohamedrejeb.harfbuzz/harfbuzz-core) (Apache-2.0) is a KMP binding over HarfBuzz for Android, JVM, iOS and Wasm, but not JS. It is at 0.6.0 with about 8 stars, so it is a data point, not something to depend on.
   - KorGE has `TtfFont`/`BitmapFont` in its KorIM module, supported on all targets ([KorGE docs](https://docs.korge.org/imaging/fonts/)). The docs describe TTF text as rasterised once into a bitmap through `fillText`. Whether KorIM's reader is pure Kotlin and how complete its kerning is I could not verify.

### 3.2 Bitmap versus distance-field atlases

- **SDF. Consensus.**
  - Valve's 2007 SIGGRAPH technique stores a distance field from a high-resolution image in a low-resolution channel. A later MSDF paper identifies sharp corners as the problematic feature of the original technique.
  - Source: [Valve paper](https://cdn.akamai.steamstatic.com/apps/valve/2007/SIGGRAPH2007_AlphaTestedMagnification.pdf). The corner claim is secondhand: it came from search-result summaries of the MSDF paper ([Sloup et al.](https://dcgi.fel.cvut.cz/wp-content/wpallimport-dist/publications/pdf/publications-2018-sloup-cgf-msdf-paper.pdf)).
  - libGDX's guide is the practical reference: it needs linear filtering, mipmaps for minification, a tunable `smoothing` uniform, and only works for monochrome glyphs. Outlines and drop shadows are shader effects on the same distance field ([libGDX](https://libgdx.com/wiki/graphics/2d/fonts/distance-field-fonts)).
- **MSDF/MTSDF. Practice.**
  - msdf-atlas-gen (MIT) offers hardmask, softmask, SDF, PSDF, MSDF and MTSDF.
  - MSDF keeps sharp corners and supports "hard" effects (mitred outlines). SDF and MTSDF support "soft" effects (glow, rounded outlines, simple shadows). MTSDF carries both.
  - JSON output includes kerning pairs, metrics and the `distanceRange`, which the shader must use.
  - It also has a `DynamicAtlas` class, but the README warns that the bundled immediate generator may start threads per use.
  - Sources: [msdf-atlas-gen](https://github.com/Chlumsky/msdf-atlas-gen), [msdfgen](https://github.com/Chlumsky/msdfgen) (the shader takes the median of the RGB channels and the channels must be read as linear; the screen-pixel range must stay at least 1, and below 2 antialiasing may fail).
- **MSDF trade-offs (Godot's docs). Practice.**
  - Benefits: text stays crisp at any size, and there is less first-use stutter at large sizes.
  - Costs: a higher baseline render cost that can matter on low-end mobile, less clarity at small sizes because there is no hinting, no LCD antialiasing, and self-intersecting outlines render wrongly.
  - For outlines, Godot says to set the MSDF pixel range to at least twice the outline size.
  - Source: [Godot fonts](https://docs.godotengine.org/en/stable/tutorials/ui/gui_using_fonts.html).
- **Raster atlases at sizes. Practice.**
  - libGDX generates one `BitmapFont` per size and suggests regenerating on resize ([gdx-freetype](https://libgdx.com/wiki/extensions/gdx-freetype)).
  - Bevy keeps one atlas per font size per face and bins subpixel offsets to bound rasterisation ([FontAtlasSet](https://dev-docs.bevyengine.org/bevy/text/struct.FontAtlasSet.html)).
  - Atlases per size make animated size changes costly and jittery (Glyphy discussion, in the Medium/GNOME search results).
- **For SpartanGraphics. Judgement.**
  - Screen-layer UI text sits at 1:1 pixels and needs the sharpest small text, so raster wins.
  - World-layer text zooms with the camera, so SDF fits there, accepting softer small sizes.
  - Raster at a bucketed size, drawn with a slight scale, is the cheap alternative for world text when zoom is mostly static.
  - Expose the choice per `TextStyle`; do not hard-wire it.

### 3.3 Atlas management

- **Runtime dynamic atlas.**
  - Practice: the cache key includes the face, glyph id, size bucket, mode and subpixel bin (Bevy). Packing is shelf or skyline: raylib's SDF example uses the skyline packer, and freetype-gl is cited for skyline packing.
  - When an atlas page is full, a simple strategy is to allocate a larger or additional page and rebuild.
  - CJK-scale sets (about 20k glyphs) make pre-baking everything impractical, so multi-page, LRU-evicted runtime atlases are the only workable scheme. libGDX states CJK "may work, but their large glyph counts are a problem".
- **Page size.** Keep pages at or below 2048 and add pages rather than growing. This is my judgement; I did not source the GLES3/WebGL2 minimum texture size.
- **Build-time baking.**
  - BMFont/Hiero are the established offline path for libGDX ([libGDX bitmap fonts](https://libgdx.com/wiki/graphics/2d/fonts/bitmap-fonts)).
  - msdf-atlas-gen is the established MSDF baker, and its JSON is a ready interchange format with kerning and metrics.
  - A bake requires no rasteriser at runtime, which helps the web backend.
  - It is a build-time external tool, so a native binary there is acceptable.
- **Judgement.** Make runtime and baked fonts two implementations of one glyph-provider interface, so baking is a pure optimisation and the Gradle plugin lands without any API change.

### 3.4 Layout

- **Kerning. Consensus on both sources existing.** Fonts carry kerning in the legacy `kern` table or in OpenType GPOS, and many modern fonts have GPOS only. stb_truetype's GPOS support is "basic" (pair kerning). A reader of GPOS pair adjustment (formats 1 and 2) in common code covers the Latin case. Contextual positioning and mark attachment belong to a real shaper.
- **Line breaking.**
  - Greedy wrapping on measured advances, with break opportunities at whitespace and hyphens, is enough for v1 real fonts.
  - Godot's `TextServer` exposes breaks as flags (word, grapheme, adaptive, edge-space trimming) and relies on ICU data (about 4 MB) for unspaced languages ([i18n docs](https://docs.godotengine.org/en/stable/tutorials/i18n/internationalizing_games.html), [TextServer](https://docs.godotengine.org/en/stable/classes/class_textserver.html)).
  - Make the break-opportunity finder a small replaceable interface. UAX #14 can come later; the JVM and Android have `BreakIterator`, and web has no line-break API.
- **Measurement.** Measurement is layout without rasterisation, so the Phase 1 contract should be a facade over the same layout code the renderer uses.
- **Fallback.** Godot and cosmic-text both model a *list* of faces with per-glyph fallback. cosmic-text's roadmap includes per-character fallback and reuses Chromium/Firefox fallback lists ([cosmic-text](https://github.com/pop-os/cosmic-text)). Compose Multiplatform for Web downloads Noto subsets on demand when characters are missing ([release notes](https://kotlinlang.org/docs/multiplatform/whats-new-compose-112.html)). A `Font` should therefore be able to name an ordered fallback chain.
- **Shaping engines.**
  - HarfBuzz is the de-facto engine: cosmic-text uses HarfRust, and Godot's Advanced server uses HarfBuzz plus ICU. Godot ships a Fallback server with no bidi or complex-text support, which is "faster", and it is not in official builds.
  - ICU4X has C/C++/JS/TS/Dart bindings and a C API for bidi and segmenters ([icu_capi](https://doc.servo.org/icu_capi/index.html)). I found no released Kotlin binding.
  - Godot's shaped-text buffers hold runs, per-run direction and bidi overrides, and support caret and hit-test queries.
- **Judgement.** The layout result must be shaped like that: glyph runs in visual order, each with direction and a cluster-to-text-index map.

### 3.5 Phase 1 evidence from this repo

- The monolith's `UiRenderer` builds STBEasyFont quads on the CPU every frame and centres text using `stb_easy_font_width(text) * TEXT_SCALE` and `GLYPH_HEIGHT_PX = 7.0`, because `stb_easy_font_height` reports a roughly 12 px line box that sits the text off-centre.
- `TEXT_SCALE = 2.0` is a constant. Text has no size parameter.
- The architecture draft already lists `TextMeasurer` under `.platform`.

## 4. How comparable projects do it

| Project | Approach | Source |
|---|---|---|
| **libGDX** | `FreeTypeFontGenerator.generateFont(parameter)` builds one `BitmapFont` per size, `kerning = true` by default, border and shadow in the generator, not supported on HTML5 without a community port, no RTL. Distance-field fonts are a separate Hiero path. | [gdx-freetype](https://libgdx.com/wiki/extensions/gdx-freetype) |
| **KorGE** | `BitmapFont`, `TtfFont` rasterised to a `NativeImage`, device fonts auto-packed into an atlas; no distance-field support found in the docs I read. | [KorGE fonts](https://docs.korge.org/imaging/fonts/) |
| **Godot 4** | `TextServer` abstract (Advanced with HarfBuzz/ICU, Fallback without); dynamic fonts with hinting and subpixel options; optional MSDF; fallback lists. | links in 3.2 and 3.4 |
| **Bevy** | cosmic-text for shaping and layout, with swash rasterisation into a per-size, per-subpixel-bin `FontAtlasSet`. | [bevy_text](https://docs.rs/bevy_text) |
| **raylib** | `LoadFontData(..., FONT_SDF, ...)` from stb_truetype, skyline-packed atlas, bilinear filter and a dedicated SDF fragment shader. | [text_font_sdf.c](https://github.com/raysan5/raylib/blob/master/examples/text/text_font_sdf.c) |
| **Compose Multiplatform** | Skia's paragraph layout on all targets, system fonts on desktop and Android, on-demand Noto download on web. Too heavy to adopt here, and it conflicts with the closed-shape model. | [release notes](https://kotlinlang.org/docs/multiplatform/whats-new-compose-112.html) |
| **Kool** | README lists "MSDF Font support for text rendering in arbitrary font sizes" (msdf-atlas-gen) on JVM, JS/Wasm and Android. I could not find how it loads or manages the atlas. | [kool](https://github.com/kool-engine/kool) |

## 5. Pitfalls and regrets

- **Per-platform layout.** Letting `Paint` or Canvas2D measure text means each platform wraps differently and headless tests cannot assert layout. The mitigation is to keep layout metrics in one engine.
- **Layout coupled to hinted pixel advances.** If advances change with zoom or DPI, world text reflows during zooming. Use unhinted advances for layout and subpixel placement for drawing.
- **Magic constants in alignment.** The monolith's `GLYPH_HEIGHT_PX = 7.0` shows what happens when the contract exposes only a width and one height. Return ascent, descent, line gap and ink bounds separately.
- **SDF at small sizes.** Godot calls out softer small text and no hinting. Do not make SDF the default for UI.
- **MSDF generation is not free at runtime.** msdf-atlas-gen warns about threads in its immediate generator. Treat MSDF as a build-time artefact.
- **Self-intersecting glyph outlines** render wrongly under MSDF (Godot docs). Variable or poorly authored fonts can trip this.
- **Wide fonts and large atlases.** libGDX documents large fonts exceeding page size, especially under GWT. Multi-page atlases avoid it.
- **Untrusted fonts and stb_truetype.** No bounds checks in parsing (see 3.1).
- **Bundled default font licensing.** Replacing STBEasyFont with a real default font means shipping a font file. Pick a font whose licence permits redistribution under Apache-2.0 packaging. I have not chosen one.
- **Fixed-size bitmap fonts in a world layer.** STBEasyFont geometry scaled by the camera looks blocky. This is expected for the built-in font and is a reason to treat it as a debug/default font only.

## 6. Recommendation for this repo

### 6.1 Shape

```
graphics-core (commonMain)
  Font                  opaque handle: ordered face list (fallback chain); default = built-in bitmap font
  FontFace              parsed metrics: cmap, advances, ascent/descent/lineGap, kern/GPOS pairs
  TextStyle             font, size (layer units), colour, alignment, maxWidth?, wrap, lineHeight, outline?, glyphMode
  TextLayout            lines -> glyph runs (glyph id, cluster, x/y, advance, direction); line boxes; ink bounds
  TextMeasurer          measure(text, style, maxWidth?): TextLayout  -- sync, pure, deterministic
  TextShaper            Experimental seam: shape(text, runs, direction, locale) -> glyph runs
                        (default: SimpleTextShaper -- cmap + advance + kerning, LTR)
  LineBreaker           break opportunities (default: whitespace and hyphen)
  GlyphProvider         glyph(face, glyphId, pixelSize, mode) -> atlas rect + metrics
     RuntimeGlyphProvider   (atlas pages + eviction; owns a GlyphRasterizer)
     BakedGlyphProvider     (loads msdf-atlas-gen JSON + page images; later)
  GlyphRasterizer       backend contract: coverage bitmap + bearings for a glyph at a pixel size
```

Names are placeholders; the shapes are the point.

- **Rasterisers by backend:**
  - Desktop: stb_truetype (raster and `stbtt_GetCodepointSDF`). FreeType through LWJGL is the upgrade path for hinting and colour glyphs.
  - Android: `Paint`/`Canvas` into an `ALPHA_8` bitmap, or stb_truetype via NDK if fonts-from-bytes proves awkward.
  - Web: `OffscreenCanvas` `fillText` into a glyph canvas.
  - Common code's SDF generator runs a distance transform on a backend-supplied high-resolution bitmap, so SDF needs no extra backend work.
- **Glyph rasteriser contract level.** It is glyph-level (glyph id plus size in, bitmap plus bearings out). A backend that can only rasterise whole runs (a browser doing native shaping) declares that as an optional capability, the way Godot exposes `has_feature`. This keeps the shaping path additive without a second contract.

### 6.2 Phase 1 (0.1.0) must contain

- `Text(text, style = TextStyle())` with a `TextStyle` object. Fields v1 ignores are still present in the API only if v1 honours or rejects them explicitly. Do not accept `maxWidth` and then silently ignore it.
- An opaque `Font` handle with a `Font.Default`.
- `TextMeasurer` returning a layout object with width, line-box height, ascent, descent, baseline and ink bounds. It must be a pure function of its inputs, independent of GL state and of pixel scale.
- Alignment done *by the library* using that result, so the magic ink-height constant goes away.
- Font loading, when added, returns `Result` (an expected failure). Measurement never throws. Measuring with an unavailable font falls back to the default font.

Breaking changes at 0.2.0 are allowed (decision 2), so the aim is to protect SpartanUI 0.1.0's label and button layout code, not to freeze anything.

### 6.3 Landing order

1. **Phase 1:** the contracts above.
2. **First real-fonts release (0.2.0):**
   - font table reading and `SimpleTextShaper`;
   - greedy wrapping;
   - a runtime atlas with LRU pages;
   - the desktop rasteriser;
   - raster mode with outlines and shadows (drawn as extra quads);
   - SDF mode.
3. **Android rasteriser**, then **web rasteriser**, each proved by its sample app.
4. **Pre-1.0:** freeze `Font`, `TextStyle`, `TextLayout`, `TextMeasurer`, and the glyph rasteriser as a backend contract (becoming `@SupportedExtension` with an example backend, per decision 4). Ship `TextShaper` as Experimental.
5. **Post-1.0:** Gradle-plugin font baking (`BakedGlyphProvider`, MSDF), then, unscheduled, shaping engines (LWJGL HarfBuzz on desktop first), bidi and UAX #14.

**Rejected option.** Delegating layout and drawing to the platform text APIs on Android and web (the Compose model). It is the smallest amount of work per platform, but it makes `TextMeasurer` platform-dependent and non-deterministic, and it cannot be tested headless. It also puts text outside the closed shape set's atlas and batching, which the 10k-sprite target assumes.

## 7. Gaps

- **Pure-Kotlin rasteriser.** I could not verify KorIM's `TtfFont` internals (pure Kotlin or not, kerning and GPOS coverage). The WebSearch quota ran out before I could look. A pure-Kotlin rasteriser in common code is the main alternative to per-backend rasterisers (identical output everywhere and GPU-free deterministic tests). I did not size it.
- **Android specifics.** I did not confirm the API level of `TextRunShaper`, or whether `Typeface.Builder` can load from a `ByteBuffer`. Both come from model recall, not a source.
- **Kool.** The README mentions MSDF only; I could not find its atlas handling.
- **Bevy world-text zoom.** I found nothing on whether Bevy re-rasterises world text on zoom or scales a fixed bitmap.
- **Compose.** I did not find how Compose's desktop and Android targets handle fallback fonts.
- **FreeType on Wasm and Android NDK.** No sourced evidence on size or build cost.
- **Texture size limits.** The 2048 page-size advice is unsourced.
- **Quality and cost figures.** I measured nothing: SDF versus raster quality at small sizes, per-glyph rasterisation cost on web or Android, and atlas churn under zoom. A spike is needed before the SDF-versus-bucketed-raster default for world text is fixed.
- **Web target flavour.** Which of Kotlin/JS or Kotlin/Wasm the web backend uses was not given in my brief, and it changes the Wasm-shaper option. harfbuzz-kmp lists Wasm but not JS.
- **Default font.** No font was chosen or licence-checked.
- **Firecrawl.** Not used (credit limit), so some pages (Valve PDF, Godot TextServer's second half) were read only partially or via secondary summaries.
