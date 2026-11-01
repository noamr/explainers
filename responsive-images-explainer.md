# Better Responsive Image Knobs

## Authors:
- Noam Rosenthal

## Participate
- [whatwg/html#13077 — Responsive images: should we define source selection normatively instead of leaving it implementation-defined](https://github.com/whatwg/html/issues/13077)
- [whatwg/html#13069 — Authors have no way to ask for a higher or lower pixel density than the browser's default when selecting from `srcset`](https://github.com/whatwg/html/issues/13069)
- [whatwg/html#12248 — `sizes="auto"` images shouldn't use concrete object width to select source](https://github.com/whatwg/html/issues/12248) ([PR #13053](https://github.com/whatwg/html/pull/13053))
- [whatwg/html#11428 — Responsive images with `srcset` unexpectedly upscale when using `object-fit: none` or `scale-down`](https://github.com/whatwg/html/issues/11428)
- [whatwg/html#12565 — No way to read an image's true intrinsic dimensions when `sizes="auto"` (or any `w`-descriptor `srcset`) is used](https://github.com/whatwg/html/issues/12565) ([PR #13046](https://github.com/whatwg/html/pull/13046))

---

## Introduction: Moving the Knobs from the Browser to the Website

Every responsive image on the web sits on a trade-off curve between **visual sharpness** and **bandwidth**:
- Pick a candidate that is too small (or upscale it beyond its native resolution), and the user sees a **blurry, pixelated image**.
- Pick a candidate that is too large, and the user pays in **wasted bandwidth, slower page loads, and higher memory use**.

When `srcset` and `sizes` were designed in 2014, the specification gave authors a single 1D width hint (`sizes`), derived both candidate density and `img.naturalWidth` from that 1D width, and left candidate selection *"implementation-defined"* so browsers could experiment with bandwidth-saving heuristics.

A decade of production experience has clarified where browser behavior should be standardized and where websites need direct control:
- **User Agents are designed for interoperability, while websites are best suited to measure context-specific business trade-offs.** A browser cannot know whether a given `<img>` is a conversion-critical product photo (where blurriness hurts sales), an interactive hero that will scale up on click, or a decorative backdrop / below-the-fold card (where capping density at `1.5x` on a `3x` phone cuts pixel count by **75%** with negligible visual loss).
- **Browser selection has converged and can be standardized.** Chrome retired its geometric-mean density rounding and `Save-Data` clamping experiments in late 2024, meaning WebKit, Gecko, and Blink now all use the same baseline selection rule (`first candidate >= targetDensity`, or the largest if none). Chrome also retains an optimization to avoid downloading a lower-resolution source when a higher-resolution candidate is already loaded/cached—behavior that can be **permitted via an explicit `MAY`** rather than leaving the entire selection algorithm implementation-defined.

**The theme of this proposal:**
1. **Standardize the browser baseline (`#13077`)** — define the end-to-end source selection algorithm (including reusing already-loaded/cached higher-resolution sources) normatively for interoperability.
2. **Give the knobs to the website (`#13069`, `#12248`, `#11428`, `#12565`)** — empower the site owner (who knows the content and can A/B test UX and business impact) to clamp target density via `mindensity` and `maxdensity`, handle 2D `object-fit` sizing (`cover`, `contain`, `scale-down`, `none`) with `h` descriptors, and inspect true intrinsic dimensions via `imageWidth`, `imageHeight`, and `imageAspectRatio`.

---

## User-Facing Problems: Blurry Images and Wasted Bandwidth

Because websites lack proper knobs—and because `sizes` conflates 1D layout width, candidate selection, and `naturalWidth`—users routinely suffer from either **blurred images** or **overused bandwidth**:

### 1. No way to bound target density per image ([#13069](https://github.com/whatwg/html/issues/13069))
When selecting from a `srcset`, the browser always targets `window.devicePixelRatio` (`1x`, `2x`, `3x`, `4x`):
- **Wasted bandwidth on high-DPR devices**: A `3x` phone downloads **9×** the pixels of a `1x` display (and **2.25×** the pixels of a `2x` display). For avatars, decorative backdrops, ambient banners, or below-the-fold feed photos, capping selection at `1.5x` or `2x` saves **56–75%** of pixels on `3x` screens with no perceptible loss—while keeping `1x` desktop screens at a crisp `1.0x`.
- **Blurry zoomable or scaled heroes on low-DPR devices**: Conversely, a product photo that users zoom into, a diagram with fine text, or a hero image that scales up (`transform: scale(2)`) during a View Transition needs at least `2x` density even on a `1x` desktop monitor, without forcing a `3x` phone to fetch `6x`.

Today, sites must **lie in markup** to nudge selection:
- **Lying in `sizes`** (e.g., writing `sizes="50vw"` for a `100vw` image) is **impossible with `sizes="auto"`**, degrades `1x` desktop monitors to `0.5x`, and corrupts `img.naturalWidth`.
- **Lying in `w` descriptors** (e.g., declaring `img.jpg?w=720 800w` so the browser treats a 720px file as 800w, as Airbnb currently has to do in production) pollutes HTML, prevents CDN cache sharing across components with different quality targets, and breaks features that rely on accurate `w`/`h` descriptors.

### 2. Blurry `object-fit: cover` cards & bloated `object-fit: contain` banners ([#12248](https://github.com/whatwg/html/issues/12248))
`sizes` only expresses a 1D width, ignoring container height and `object-fit`:
- **Blurry images with `object-fit: cover`**: Most listing grids (e.g., Airbnb cards) render wide photos inside fixed-ratio boxes using `object-fit: cover`. A 4:3 photo in a `400px × 400px` square card is drawn at `533px × 400px` (`1.33×` the box width). Using 1D `sizes="400px"` or `sizes="auto"` content-box width under-fetches resolution by **33%** (and by **78%** for a 16:9 photo), leaving images visibly soft.
- **Massive bandwidth waste with `object-fit: contain`**: A 1:1 image inside a wide `1000px × 100px` container with `object-fit: contain` renders at `100px × 100px`. A 1D content-box width of `1000px` fetches a `1000w` image—**10× too wide** (**100× too many pixels**).
- **Infinite download loops in the current spec**: The HTML spec currently defines `sizes="auto"` using CSS *concrete object size*, which depends on the loaded image's aspect ratio under `object-fit: contain`. Once a candidate loads, its aspect ratio changes the concrete object width, which triggers a new `srcset` selection—oscillating in an **infinite download loop** when candidates have different aspect ratios.

### 3. Unexpected blurry upscaling with `object-fit: none` and `scale-down` ([#11428](https://github.com/whatwg/html/issues/11428))
Authors use `object-fit: scale-down` or `object-fit: none` specifically to **prevent** a small image (such as a `200px` logo or icon) from upscaling and becoming blurry inside a larger container (e.g., `800px × 600px`).

However, when `srcset` with `w` descriptors is used, the browser computes `density = w / sourceSize` and sets `naturalWidth = rawPixelWidth / density = sourceSize`. For a `200w` image in an `800px` `sizes="auto"` box:
- `density = 200 / 800 = 0.25x`
- `naturalWidth = 200px / 0.25 = 800px`

Because CSS `object-fit: none` and `scale-down` size the image using its density-corrected `naturalWidth` (`800px`), **both `object-fit: none` and `scale-down` upscale the `200px` image to `800px`**, resulting in a blurry image and defeating the purpose of `scale-down` and `none`!

### 4. Unnecessary "implementation-defined" spec baseline ([#13077](https://github.com/whatwg/html/issues/13077))
The HTML Standard still states:
> *"In an implementation-defined manner, choose one image source..."*

Today, WebKit, Gecko, and Blink all implement the same baseline rule (`first candidate >= targetDensity`), and Chrome's optimization to keep an already-loaded higher-resolution candidate rather than re-fetching a smaller one can be standardized. Leaving the entire selection step "implementation-defined" blocks normative Web Platform Tests (WPT) and prevents authors from having a predictable baseline for new knobs.

### 5. Blindfolds on true pixel dimensions and aspect ratio ([#12565](https://github.com/whatwg/html/issues/12565), [PR #13046](https://github.com/whatwg/html/pull/13046))
Because `w` descriptors divide `naturalWidth` by `density` (and with [PR #13046](https://github.com/whatwg/html/pull/13046) `naturalWidth` always equals `sourceSize`), under `sizes="auto"`, `img.naturalWidth` echoes the element's CSS layout width (`img.clientWidth`), and `naturalWidth / naturalHeight` suffers from integer-rounding quantization (or returns `0 / 0` for `viewBox`-only SVGs).

As a result, websites cannot read the true pixel dimensions or exact intrinsic aspect ratio of the resource the browser selected:
- **RUM / performance telemetry** cannot measure whether pages are over-fetching (wasting bandwidth) or under-fetching (showing blurry images) in production.
- **Lightbox, crop, and zoom UIs** cannot check whether the currently loaded image has enough real pixels to zoom in without exceeding `1:1` native pixels, or read its exact unquantized aspect ratio without creating an offscreen `new Image()`.

---

## Proposed Solution

```
Browser Responsibility (Interop)          Website Responsibility (Business & UX Knobs)
────────────────────────────────          ────────────────────────────────────────────
• Explicit selection algorithm (#13077):  • Density Bounds (#13069):
  Deterministic candidate normalization,    Clamp target density with mindensity / maxdensity
  targetDensity clamping, ascending sort, • 2D Layout & object-fit (#12248):
  & cached higher-res reuse (MAY)           Declare w/h aspect ratio & cover/contain
• Respect native size bounds for          • True Resource Inspection (#12565):
  object-fit: scale-down / none (#11428)    imageWidth, imageHeight, imageAspectRatio
```

### 1. `mindensity` and `maxdensity` Attributes ([#13069](https://github.com/whatwg/html/issues/13069))

Introduce two content attributes—`mindensity` and `maxdensity`—on `<img>` and `<source>`, parsed using HTML's rules for parsing positive floating-point numbers (`> 0`):

```html
<!-- Cap high-DPR phones at 1.5x (saves 75% pixels vs 3x), while 1x desktop stays at 1.0x -->
<img loading="lazy" sizes="auto" maxdensity="1.5"
     srcset="card-400.jpg 400w, card-600.jpg 600w, card-800.jpg 800w, card-1200.jpg 1200w"
     alt="Listing preview">

<!-- Guarantee at least 2x density for a zoomable product hero (even on 1x displays) -->
<img loading="lazy" sizes="auto" mindensity="2"
     srcset="hero-600.jpg 600w, hero-1200.jpg 1200w, hero-1800.jpg 1800w"
     alt="Product detail">

<!-- Pin density to [1.5x, 2x] across all devices, or override per <picture> <source> -->
<picture>
  <source type="image/avif" mindensity="1.5" maxdensity="2.5"
          srcset="hero-600.avif 600w, hero-1200.avif 1200w">
  <img sizes="auto" mindensity="1.5" maxdensity="2"
       srcset="hero-600.jpg 600w, hero-1200.jpg 1200w" alt="Hero">
</picture>
```

```webidl
partial interface HTMLImageElement {
  [CEReactions, Reflect] attribute DOMString minDensity;
  [CEReactions, Reflect] attribute DOMString maxDensity;
};

partial interface HTMLSourceElement {
  [CEReactions, Reflect] attribute DOMString minDensity;
  [CEReactions, Reflect] attribute DOMString maxDensity;
};
```

**Key properties:**
- **Clamps `targetDensity` without lying in `sizes` or `w` descriptors:** `sizes` and `w`/`h` descriptors stay 100% honest, preserving `img.naturalWidth`, 2D aspect-ratio math, and shared CDN URLs across components.
- **Works with both `sizes="auto"` and the preload scanner:** Because `mindensity` and `maxdensity` are HTML attributes on `<img>` and `<source>`, eager images scanned before stylesheets load can immediately apply the density bounds.
- **Avoids extreme-DPR distortion:** Unlike a relative multiplier, `maxdensity="1.5"` reduces `3x` and `2x` fetches to `1.5x` *without* degrading `1x` displays to `0.5x`, and `mindensity="2"` upgrades `1x` displays to `2x` *without* forcing `3x` displays to fetch `6x`.
- **`<picture>` precedence:** When `<picture>` selects a `<source>` element, `mindensity` and `maxdensity` are read from that `<source>` if specified (and valid), falling back independently to the `<img>` element's `mindensity` / `maxdensity` if omitted on `<source>`.
- **Precedence when `mindensity > maxdensity`:** Following CSS `clamp(min, val, max)` (`max(min, min(val, max))`), `mindensity` wins if an author specifies `mindensity > maxdensity`.

---

### 2. 2D `sizes`, `h` Descriptors, and `object-fit` ([#12248](https://github.com/whatwg/html/issues/12248), [#11428](https://github.com/whatwg/html/issues/11428), [PR #13053](https://github.com/whatwg/html/pull/13053))

#### A. Revive `h` descriptors in `srcset` as a pre-fetch aspect-ratio hint
Allow an optional positive integer `<height-descriptor>` (`Nh`) alongside a width descriptor (`Mw`) in `srcset`:

```html
<img loading="lazy" sizes="auto"
     style="width: 400px; height: 400px; object-fit: cover"
     srcset="card-400.jpg 400w 300h,
             card-800.jpg 800w 600h,
             card-1200.jpg 1200w 900h"
     alt="">
```
- When all candidates share the same aspect ratio as the `<img>` (or `<source>`) `width` and `height` attributes, lazy `sizes="auto"` images do not even need `h` descriptors—each candidate falls back to `width / height`.
- When candidates have differing aspect ratios—or when `width`/`height` attributes are omitted—`w` + `h` provides each candidate's exact pre-fetch aspect ratio `r = w / h` directly to both layout and the speculative preload scanner.
- **Zero infinite loops:** Candidate selection uses *only* declared markup hints (`w`/`h` descriptors or `width`/`height` attributes), never the decoded bitmap's intrinsic ratio. Loading an image never mutates its candidate aspect ratio or triggers a re-selection oscillation loop.

#### B. Make `sizes="auto"` 2D- and `object-fit`-aware
Instead of using CSS *concrete object size* (which depends on the already-loaded image), `sizes="auto"` reads the `<img>` element's CSS **content box** `(boxWidth, boxHeight)` and computed `object-fit`, computing each candidate's effective rendered width from its declared ratio `r` (see Step 2 of the explicit algorithm below).

#### C. Extend `sizes` grammar for eager images & the preload scanner ([PR #13053](https://github.com/whatwg/html/pull/13053))
Allow `<source-size-value>` items in `sizes` to express a 2D box with `cover` or `contain`, with graceful 1D fallback in older browsers:

```html
<img srcset="s.jpg 50w 100h, l.jpg 100w 200h"
     sizes="(max-width: 600px) cover 100vw 300px, cover 400px 400px, 400px"
     alt="">
```
Grammar extension for `<source-size-value>`:
```
<source-size-value> = <length> | auto | <fit-modifier> <length>{2}
<fit-modifier>      = cover | contain
```
*(In legacy browsers, an unrecognized `<source-size-value>` fails parsing for that comma-separated item and falls through to the trailing 1D `400px` fallback.)*

#### D. Fix unexpected upscaling under `object-fit: scale-down` and `none` ([#11428](https://github.com/whatwg/html/issues/11428))
When an `<img>` has no explicit `width`/`height` dimension attributes and its computed `object-fit` is `none` or `scale-down`, clamp the density divisor used to compute CSS intrinsic size (`naturalWidth` / layout natural size) to `>= 1.0` (or `>= devicePixelRatio` when scaling to physical pixels is undesired, i.e., `naturalWidth = min(sourceSize, imageWidth)`), so a `200w` image inside an `800px` `object-fit: scale-down` container renders at `200px` rather than upscaling to `800px`.

---

### 3. `imageWidth`, `imageHeight`, and `imageAspectRatio` Getters ([#12565](https://github.com/whatwg/html/issues/12565))

Expose the true, unscaled intrinsic dimensions and aspect ratio of the loaded image resource on `HTMLImageElement`:

```webidl
partial interface HTMLImageElement {
  readonly attribute unsigned long imageWidth;
  readonly attribute unsigned long imageHeight;
  readonly attribute double imageAspectRatio;
};
```

#### Exact Getter Semantics

All three getters share the **exact same readiness timing and cross-origin rules** as `naturalWidth` and `naturalHeight` (they inspect `el`'s *current request* when its state is *available*, including cross-origin `no-cors` images, and do not leak anything prior to *available*):

1. **`imageWidth` getter (`unsigned long`):**
   - If `this`'s *current request*'s state is not *available*, or if the underlying image resource has no intrinsic width (e.g., an `<svg viewBox="0 0 16 9">` with no `width` attribute), return `0`.
   - Otherwise, return the **unscaled intrinsic width** of the *current request*'s image data in CSS/image pixels **before** dividing by the `srcset` candidate's pixel density (rounded to an unsigned integer if needed for non-integer SVG lengths).

2. **`imageHeight` getter (`unsigned long`):**
   - If `this`'s *current request*'s state is not *available*, or if the underlying image resource has no intrinsic height, return `0`.
   - Otherwise, return the **unscaled intrinsic height** of the *current request*'s image data **before** dividing by the `srcset` candidate's pixel density.

3. **`imageAspectRatio` getter (`double`):**
   - If `this`'s *current request*'s state is not *available*, return `0.0`.
   - If the underlying image resource has an **intrinsic aspect ratio** (as defined by [CSS Images § 2.2](https://drafts.csswg.org/css-images-3/#intrinsic-dimensions)):
     - For raster images with pixel dimensions `imageWidth > 0` and `imageHeight > 0`: return the IEEE 754 `double` value `imageWidth / imageHeight`.
     - For vector images (such as SVG): return the exact intrinsic aspect ratio (from `width`/`height` if both are definite non-percentage lengths, or from `viewBox` width / `viewBox` height when `width`/`height` are omitted or percentage-based).
   - Otherwise (the image has no intrinsic aspect ratio, e.g., an SVG with neither `width`/`height` nor `viewBox`, or a degenerate `0`-height image), return `0.0`.

```js
const img = document.querySelector("img");
// Suppose card-800.jpg (800×450, 16:9) was selected for a 300px box with sizes="auto":
img.naturalWidth;     // 300               (density-corrected to match sizes)
img.naturalHeight;    // 169               (rounded integer: 300 * 450 / 800 = 168.75 → 169)
img.naturalWidth / img.naturalHeight; // 1.7751479... (quantization error from integer rounding!)

img.imageWidth;       // 800               (true bitmap width in pixels)
img.imageHeight;      // 450               (true bitmap height in pixels)
img.imageAspectRatio; // 1.7777777777777777 (exact 16 / 9 ratio; also works for viewBox-only SVGs!)
```

**Why `imageAspectRatio` in addition to `imageWidth` and `imageHeight`?**
- **Vector images without pixel dimensions:** An `<svg viewBox="0 0 16 9">` has no intrinsic width or height (`imageWidth === 0`, `imageHeight === 0`), yet has a well-defined intrinsic aspect ratio (`16 / 9`).
- **No integer quantization error:** `naturalWidth` and `naturalHeight` are `unsigned long` integers scaled down to `sourceSize` (e.g., `100` and `56` for a `16:9` image in a `100px` slot), which distorts `naturalWidth / naturalHeight`. `imageAspectRatio` returns the exact `double` ratio.
- **Ergonomic RUM & lightbox math:** Avoids divide-by-zero boilerplate (`img.imageHeight ? img.imageWidth / img.imageHeight : 0`).

---

### 4. Explicit Normative Source Selection Algorithm ([#13077](https://github.com/whatwg/html/issues/13077), [#13069](https://github.com/whatwg/html/issues/13069), [#12248](https://github.com/whatwg/html/issues/12248))

Replace HTML's *"In an implementation-defined manner, choose one image source"* (and update *"create a source set"* / *"select an image source from a source set"*) with the following deterministic algorithm.

#### Inputs
- `el`: the `HTMLImageElement`.
- `sourceEl`: the matched `<source>` child of a parent `<picture>` element, or `null` if `el`'s own `srcset` / `src` is used.
- `rawCandidates`: the parsed list of image candidates from `sourceEl` (if non-null) or `el` (`srcset` + optional `src` fallback), in markup order. Each candidate `c` has:
  - `c.url`: candidate URL string
  - `c.w`: integer width descriptor (`> 0`), or `null`
  - `c.h`: integer height descriptor (`> 0`, valid only when `c.w` is non-null), or `null`
  - `c.x`: floating-point density descriptor (`> 0`, mutually exclusive with `c.w`), or `null`

---

#### Step 1 — Compute Each Candidate's Declared Aspect Ratio (`c.ratio`)

For each candidate `c` in `rawCandidates`:
1. If `c.w != null` and `c.h != null`, set `c.ratio = c.w / c.h`.
2. Otherwise:
   - Let `dimHost` be `sourceEl` if `sourceEl` is non-null and specifies both `width` and `height` content attributes; otherwise `el`.
   - Parse `dimHost`'s `width` and `height` content attributes as non-negative integers `attrW` and `attrH`.
   - If both succeed and `attrW > 0` and `attrH > 0`, set `c.ratio = attrW / attrH`.
   - Otherwise, set `c.ratio = null` (*unknown*).

> **Important:** `c.ratio` **MUST NOT** inspect the decoded dimensions of any previously loaded image on `el`. This guarantees that `sizes="auto"` selection is a pure function of layout box size and declared markup hints, preventing infinite download oscillation loops.

---

#### Step 2 — Resolve Effective Source Size (`c.sourceWidth`) per Candidate

Let `sizesAttr` be `sourceEl`'s `sizes` attribute if `sourceEl` is non-null and has a `sizes` attribute, otherwise `el`'s `sizes` attribute.

1. **If `el` allows auto-sizes** (`sizesAttr` starts with `auto`, or is omitted on a `loading="lazy"` `<img>` per HTML's auto-sizes rules) **and `el` is being rendered with layout information available**:
   1. Let `boxWidth >= 0` and `boxHeight >= 0` be the width and height in CSS pixels of `el`'s CSS **content box** (if `el` has no CSS layout box, fall back to `300px` × `150px` or evaluate trailing fallback items in `sizesAttr` per existing HTML rules).
   2. Let `fit` be the computed value of `object-fit` on `el`.
   3. For each candidate `c` in `rawCandidates`:
      - If `c.ratio == null`, or `boxHeight == 0`, or `fit` is `"fill"` or `"none"`:
        `c.sourceWidth = boxWidth`
      - Else if `fit` is `"cover"`:
        `c.sourceWidth = max(boxWidth, boxHeight * c.ratio)`
      - Else (`fit` is `"contain"` or `"scale-down"`):
        `c.sourceWidth = min(boxWidth, boxHeight * c.ratio)`

2. **Otherwise (explicit `sizes` evaluation, including eager preload scanning):**
   1. Parse `sizesAttr` as a comma-separated `<source-size-list>`. Find the **first** valid `<source-size>` whose `<media-condition>` evaluates to true (defaulting to `100vw` if `sizesAttr` is missing or no item matches):
      - **Case 2a — 1D `<length>` `W`:**
        Resolve `W >= 0` to CSS pixels. For every candidate `c`, set `c.sourceWidth = W`.
      - **Case 2b — 2D `<fit-modifier> <length> <length>` (`fitKeyword`, `W`, `H`):**
        Resolve `W >= 0` and `H >= 0` to CSS pixels. For each candidate `c`:
        - If `c.ratio == null` or `H == 0`, set `c.sourceWidth = W`.
        - Else if `fitKeyword` is `cover`, set `c.sourceWidth = max(W, H * c.ratio)`.
        - Else (`fitKeyword` is `contain`), set `c.sourceWidth = min(W, H * c.ratio)`.

---

#### Step 3 — Normalize Candidate Pixel Densities (`c.density`) and Deduplicate

Initialize an empty list `candidates`.

1. For each candidate `c` in `rawCandidates` (in markup order):
   1. Compute `c.density`:
      - If `c.w != null`:
        `c.density = c.w / max(c.sourceWidth, 1e-9)`
        *(Clamping zero-width boxes to `1e-9` preserves strict ordering by `c.w` ascending when `c.sourceWidth == 0`, so Step 5 deterministically picks the smallest `w` candidate rather than collapsing all densities to `Infinity`.)*
      - Else if `c.x != null`:
        `c.density = c.x`
      - Else (bare URL with no descriptor, or fallback `src`):
        `c.density = 1.0`
   2. **Deduplicate equal densities:** If `candidates` already contains an entry `e` with `e.density == c.density`, **discard `c`** (the first entry in markup order wins ties). Otherwise, append `c` to `candidates`.

2. If `candidates` is empty, return `null` (no source selected).

---

#### Step 4 — Compute `targetDensity`

1. **Base display density:** Let `baseDensity` be the CSS-to-physical pixel ratio of `el`'s output device (`window.devicePixelRatio`, incorporating page zoom).
2. **Read `minDensity`:**
   - Let `minAttr` be `sourceEl`'s `mindensity` attribute if `sourceEl` is non-null and has a `mindensity` attribute; otherwise `el`'s `mindensity` attribute.
   - Parse `minAttr` using the *rules for parsing floating-point number values*. Let `minDensity` be the parsed number if valid, finite, and `> 0`; otherwise `null`.
3. **Read `maxDensity`:**
   - Let `maxAttr` be `sourceEl`'s `maxdensity` attribute if `sourceEl` is non-null and has a `maxdensity` attribute; otherwise `el`'s `maxdensity` attribute.
   - Parse `maxAttr` using the *rules for parsing floating-point number values*. Let `maxDensity` be the parsed number if valid, finite, and `> 0`; otherwise `null`.
4. **Apply author density clamp:**
   - Let `targetDensity = baseDensity`.
   - If `maxDensity` is non-null: `targetDensity = min(targetDensity, maxDensity)`.
   - If `minDensity` is non-null: `targetDensity = max(targetDensity, minDensity)`.
5. **Apply user-preference / privacy adjustments (`MAY`):**
   - The user agent **MAY** adjust `targetDensity` to honor explicit **user** preferences (e.g., an OS or browser data-saver setting or user-configured high-DPI / reduced-resolution accessibility preference) or **privacy / anti-fingerprinting** protections (e.g., DPR capping in private browsing).
   - User agents **MUST NOT** apply content-agnostic candidate-midpoint rounding heuristics (such as geometric-mean rounding) when selecting among `candidates`.

---

#### Step 5 — Sort Ascending and Pick `preferredSource`

1. Sort `candidates` in **strictly ascending order** of `c.density` (`c_1.density < c_2.density < ... < c_n.density`).
2. Find the smallest index `i` (`1 <= i <= n`) such that `c_i.density >= targetDensity`.
3. - If such an index `i` exists, let `preferredSource = c_i` (the **first** candidate meeting or exceeding `targetDensity`).
   - Otherwise (all candidates have `c.density < targetDensity`), let `preferredSource = c_n` (the **last / highest-density** candidate).

---

#### Step 6 — Higher-Resolution Local Cache Reuse (`MAY`)

1. Let `higherCached` be the subset of entries `c` in `candidates` such that:
   - `c.density > preferredSource.density`, **and**
   - `c`'s image data is **already available** in `el`'s `Document`'s *list of available images*, or is synchronously usable from a local cache without initiating a network request.
2. If `higherCached` is non-empty, the user agent **MAY** set `selectedSource` to an entry in `higherCached` instead of `preferredSource` (avoiding a redundant network fetch for a lower-resolution file when a sharper candidate is already downloaded).
   > *Note:* Cache reuse is intentionally allowed even when `maxdensity` is set, because `maxdensity` exists to avoid **downloading** unnecessary bytes—if a higher-resolution candidate is already in local cache, reusing it costs zero network bytes while avoiding an extra HTTP request.
3. Otherwise, set `selectedSource = preferredSource`.
4. Return `(selectedSource, selectedSource.density)`.

---

#### Worked Example of the Selection Algorithm

Suppose a listing card renders inside a `300px × 300px` square CSS content box with `object-fit: cover` and `sizes="auto"`, on a **`3x` mobile phone** (`devicePixelRatio = 3.0`), with `maxdensity="1.5"`:

```html
<img loading="lazy" sizes="auto" maxdensity="1.5"
     style="width: 300px; height: 300px; object-fit: cover"
     srcset="card-400.jpg  400w 300h,
             card-600.jpg  600w 450h,
             card-800.jpg  800w 600h,
             card-1200.jpg 1200w 900h"
     alt="4:3 listing photo">
```

1. **Step 1 (Aspect ratio):** Every candidate has `c.ratio = 400 / 300 = 1.3333...` (`4:3`).
2. **Step 2 (2D `sourceWidth`):** With `object-fit: cover` in a `300 × 300` box:
   `c.sourceWidth = max(300, 300 * 4/3) = max(300, 400) = 400px`.
3. **Step 3 (Normalized densities):**
   - `card-400.jpg`: `400 / 400 = 1.0x`
   - `card-600.jpg`: `600 / 400 = 1.5x`
   - `card-800.jpg`: `800 / 400 = 2.0x`
   - `card-1200.jpg`: `1200 / 400 = 3.0x`
4. **Step 4 (`targetDensity`):** `baseDensity = 3.0`; clamped by `maxdensity="1.5"` → `targetDensity = min(3.0, 1.5) = 1.5`.
5. **Step 5 (`preferredSource`):** First candidate with `c.density >= 1.5` is **`card-600.jpg` (`1.5x`)** — sharp after the `4:3 → 1:1` crop (`600 × 450` vs `1200 × 900`, **saving 75% of pixels** compared to `3x`, while a `1x` desktop monitor has `targetDensity = min(1.0, 1.5) = 1.0` and picks `card-400.jpg` so it is never blurry after cropping!).
6. **Step 6 (Cache reuse):** If `card-1200.jpg` was already downloaded earlier (e.g., from visiting the listing detail page), the browser **MAY** reuse `card-1200.jpg` from memory/cache with zero network traffic; otherwise it fetches `card-600.jpg`.

---

## Alternatives Considered

### Relative multiplier (`densityscale="0.75"`) vs. `mindensity` / `maxdensity`
We initially explored a multiplicative `densityscale` attribute (`targetDensity = DPR * densityscale`). However, multiplying `DPR` behaves poorly at the ends of the device spectrum:
- Setting `densityscale="0.5"` to tame `3x` mobile downloads (`1.5x`) unintentionally degrades `1x` desktop monitors to **`0.5x`** (visibly blurry).
- Setting `densityscale="2"` so `1x` desktop users can zoom `2×` into a product image forces `3x` mobile phones to request **`6x`** (**36×** the pixels of `1x`).
`mindensity` and `maxdensity` express the author's actual quality floor and bandwidth ceiling directly across all DPRs—and an author who wants a narrow target on all devices can combine both (`mindensity="1.5" maxdensity="2"`).

### Semantic keywords (`resolutionpriority="low | high"`)
Because `srcset` candidates are discrete rungs chosen by each site's image CDN (e.g., `1.3×`, `1.5×`, or `2×` steps), a vague keyword with a UA-invented factor either fails to cross a rung boundary (doing nothing) or drops two rungs unexpectedly—the same predictability problem that led Blink to remove its internal bandwidth heuristics in 2024. Explicit numeric `mindensity` and `maxdensity` match numeric `x` descriptors in `srcset`.

### CSS-only density properties
While CSS properties could work for `loading="lazy"` images, **eager LCP images** are fetched by the HTML preload scanner before external stylesheets arrive. HTML attributes (`mindensity` / `maxdensity`) work for both eager and lazy images.

### Keep lying in `sizes` or `w` descriptors
Writing `sizes="50vw"` on a `100vw` image is impossible with `sizes="auto"`, degrades `1x` monitors, and corrupts `img.naturalWidth` ([PR #13046](https://github.com/whatwg/html/pull/13046)). Lying in `w` descriptors (`img.jpg?w=720 800w`) breaks CDN cache reuse and 2D `w`/`h` aspect ratios.

### Change `naturalWidth` directly or rely on `createImageBitmap(img)` instead of `imageWidth` / `imageHeight` / `imageAspectRatio`
- Changing `naturalWidth` to return raw file pixels for `w`-descriptor images would break web compatibility with pages relying on density-corrected `naturalWidth` ([PR #13046](https://github.com/whatwg/html/pull/13046)).
- `(await createImageBitmap(img)).width` is asynchronous, allocates a duplicate decoded bitmap in RAM, cannot express vector `viewBox` aspect ratios, and **throws `SecurityError` on cross-origin `no-cors` CDN images**—whereas `imageWidth`, `imageHeight`, and `imageAspectRatio` work synchronously on all *available* `<img>` elements just like `naturalWidth`.

### Server-side Client Hints (`Sec-CH-DPR` / `Sec-CH-Width`)
Client Hints require server-side negotiation, CDN `Vary` configuration, and extra policy headers. Declarative client-side `srcset` knobs work on any static host or CDN and compose directly with `sizes="auto"`.

---

## Accessibility, Internationalization, and Performance

- **Accessibility**: Fixing `object-fit: cover` under-fetching (`#12248`), `scale-down`/`none` upscaling (`#11428`), and enabling `mindensity` (`#13069`) ensures diagrams, text-in-images, and zoomable photos remain sharp and legible across `1x` and high-DPR displays.
- **Internationalization**: None affected.
- **Performance & Business Impact**:
  - Sites can cut high-DPR secondary/below-the-fold image pixels by **50–75%** using `maxdensity` (`#13069`) without degrading `1x` displays, and cut up to **90%** on letterboxed `object-fit: contain` images (`#12248`).
  - Sites can eliminate blurry `object-fit: cover` cards (`#12248`) and measure exact pixel efficiency in RUM via `imageWidth`, `imageHeight`, and `imageAspectRatio` (`#12565`).

---

## Stakeholder Feedback / Signals

- **Normative selection**: [whatwg/html#13077](https://github.com/whatwg/html/issues/13077)
- **2D `sizes="auto"`, `h` descriptors, & `object-fit`**: [whatwg/html#12248](https://github.com/whatwg/html/issues/12248), [whatwg/html#11428](https://github.com/whatwg/html/issues/11428), [whatwg/html#13053](https://github.com/whatwg/html/pull/13053)
- **Density bounds (`mindensity` / `maxdensity`)**: [whatwg/html#13069](https://github.com/whatwg/html/issues/13069)
- **True intrinsic dimensions (`imageWidth` / `imageHeight` / `imageAspectRatio`)**: [whatwg/html#12565](https://github.com/whatwg/html/issues/12565), [whatwg/html#13046](https://github.com/whatwg/html/pull/13046)

---

## Security and Privacy Self-Review Questionnaire

Per the [W3C Security and Privacy Self-Review Questionnaire](https://w3c.github.io/security-questionnaire/):

### 2.1 What information does this feature expose, and for what purposes?
`imageWidth`, `imageHeight`, and `imageAspectRatio` expose the unscaled intrinsic pixel dimensions and aspect ratio of the loaded image resource so sites can manage zoom/crop bounds and measure image sizing accuracy in RUM. This information is **already exposed** today by loading `img.currentSrc` in an `<img>` without `w` descriptors and reading `naturalWidth` / `naturalHeight`.

### 2.2 Do features in your specification expose the minimum amount of information necessary to enable their intended uses?
Yes. `imageWidth`, `imageHeight`, and `imageAspectRatio` expose only intrinsic dimensions and ratio when the image request is *available*, governed by the exact same readiness and cross-origin rules as `naturalWidth` / `naturalHeight`.

### 2.3 How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?
The feature does not handle or access personal information or PII.

### 2.4 How do the features in your specification deal with sensitive information?
The feature does not handle or access sensitive information.

### 2.5 Does data exposed by your specification carry related but distinct information that may not be obvious to users?
If a user has enabled an OS- or browser-level data-saver setting that lowers `targetDensity` (permitted by Step 4.5 of the selection algorithm), the selected `currentSrc` reflects that preference—identical to how `srcset` behaves today.

### 2.6 Do the features in your specification introduce new state for an origin that persists across browsing sessions?
No new persistent state is introduced.

### 2.7 Do the features in your specification expose information about the underlying platform to origins?
No new platform information is exposed beyond `window.devicePixelRatio`, which is already exposed to scripts.

### 2.8 Does this specification allow an origin to send data to the underlying platform?
No.

### 2.9 Do features in this specification allow an origin access to sensors on a user's device?
No.

### 2.10 What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.
`imageWidth`, `imageHeight`, and `imageAspectRatio` expose the unscaled intrinsic dimensions and aspect ratio of `img.currentSrc`. This is identical to loading `img.currentSrc` in an `<img src="...">` without `w` descriptors and inspecting its intrinsic size / layout ratio.

### 2.11 Do features in this specification enable new script execution/loading mechanisms?
No.

### 2.12 Do features in this specification allow an origin to access other devices?
No.

### 2.13 Do features in this specification allow an origin some measure of control over a user agent's native UI?
No.

### 2.14 What temporary identifiers do the features in this specification create or expose to the web?
None.

### 2.15 How does this specification distinguish between behavior in first-party and third-party contexts?
Same-origin vs. cross-origin image resources follow existing HTML `<img>` security rules.

### 2.16 How does the features in this specification work in the context of a browser's Private Browsing or Incognito mode?
Identically to normal browsing, with the HTTP cache isolated to the ephemeral profile. UAs may clamp `targetDensity` in Private Browsing (Step 4.5) if they clamp `devicePixelRatio` for fingerprinting protection.

### 2.17 Does this specification have both "Security Considerations" and "Privacy Considerations" sections?
Yes:
- **Security Considerations**: Eliminating the `concrete object size` oscillation loop (`#12248` / PR `#13053`) fixes an infinite-refetch resource-exhaustion bug when `sizes="auto"` is combined with `object-fit: contain`. `imageWidth`, `imageHeight`, and `imageAspectRatio` follow the exact same cross-origin *available* state rules as `naturalWidth` / `naturalHeight`.
- **Privacy Considerations**: Step 4.5 of the normative selection algorithm (`#13077`) explicitly preserves the user agent's ability to adjust `targetDensity` for user privacy/anti-fingerprinting or reduced-data settings *after* author `mindensity`/`maxdensity` clamping, ensuring user privacy settings always take precedence over page attributes.

### 2.18 Do features in your specification enable origins to downgrade default security protections?
No.

### 2.19 What should this questionnaire have asked?
Nothing additional.
