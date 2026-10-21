# Better Responsive Image Knobs

## Authors:
- Noam Rosenthal

## Participate
- [whatwg/html#13077 — Define responsive image source selection normatively](https://github.com/whatwg/html/issues/13077)
- [whatwg/html#12248 — `sizes="auto"` and `object-fit` (`contain` / `cover`)](https://github.com/whatwg/html/issues/12248) ([PR #13053](https://github.com/whatwg/html/pull/13053))
- [whatwg/html#11428 — Account for pinch-zoom and CSS transforms in `sizes="auto"`](https://github.com/whatwg/html/issues/11428) ([PR #13046](https://github.com/whatwg/html/pull/13046))
- [whatwg/html#13069 — Let authors request a higher or lower pixel density from `srcset`](https://github.com/whatwg/html/issues/13069)
- [whatwg/html#12565 — Expose an image's true intrinsic pixel dimensions when `w` descriptors are used](https://github.com/whatwg/html/issues/12565)

---

## Introduction

HTML's `srcset` and `sizes` attributes were designed in 2014 around a simple 1D model:
1. Resolve a 1D layout width from `sizes`.
2. Divide each candidate's `w` descriptor by that width to get a candidate density (`w / sourceSize`).
3. Let the browser pick a candidate in an *"implementation-defined manner"*.
4. Overwrite `img.naturalWidth` and `img.naturalHeight` by dividing the bitmap's pixel dimensions by that density.

Over the past decade, web layouts have evolved to rely on `loading="lazy"`, `sizes="auto"`, 2D `object-fit: cover` / `contain` containers, pinch-zoomable viewports, and View Transitions. Fitting these modern patterns into the 2014 1D model has exposed five interlocking gaps. This explainer proposes a cohesive set of specification improvements and author knobs to address them.

---

## User-Facing Problems

### 1. Source selection is still "implementation-defined" ([#13077](https://github.com/whatwg/html/issues/13077))
In *"select an image source"*, the HTML Standard states:
> *"In an implementation-defined manner, choose one image source..."*

Historically this left room for bandwidth-saving experiments, most notably Chrome's geometric-mean density threshold and `Save-Data` clamping. However, Chrome removed those heuristics in December 2024 (Chrome 133). Today, **Blink, Gecko, and WebKit all implement the exact same selection rule**: sort candidates ascending by density and pick the first candidate with `density >= targetDensity` (or the largest if none).

Leaving selection "implementation-defined" makes it impossible to write normative Web Platform Tests (WPT) for `srcset` selection and makes it difficult to specify new responsive image features that depend on predictable candidate selection.

### 2. 1D `sizes` breaks on `object-fit: cover` and `contain` ([#12248](https://github.com/whatwg/html/issues/12248))
`sizes` only expresses a single horizontal length. When an `<img>` uses `object-fit: cover` or `object-fit: contain`, its rendered bitmap width depends on **both** container dimensions and the image's aspect ratio:
- **Under-fetching with `object-fit: cover`**: A 4:3 photo inside a `400px × 400px` square card with `object-fit: cover` is scaled to `533px × 400px` and cropped horizontally. Using `sizes="400px"` (or `sizes="auto"` reading content-box width) under-fetches resolution by **33%** (or **78%** for a 16:9 photo in a square card).
- **Over-fetching with `object-fit: contain`**: A 1:2 portrait image inside a `1000px × 200px` banner is rendered at `100px × 200px`. A 1D width of `1000px` over-fetches by **10×** in each dimension (**100×** in pixels).
- **The `concrete object size` feedback loop**: The HTML spec currently defines `sizes="auto"` using CSS *concrete object size*, which depends on the loaded image's intrinsic aspect ratio. If a placeholder has a 1:1 ratio in a `1000px × 200px` `object-fit: contain` box (concrete width `200px`), loading a 2:1 candidate changes the concrete width to `400px`, triggering a new fetch—which can oscillate in an **infinite download loop**.

### 3. Pinch-zoom and CSS transforms leave images blurry ([#11428](https://github.com/whatwg/html/issues/11428))
When a user pinch-zooms into a product photo, map, or comic strip—or when an image is scaled up via CSS `transform: scale(2)`—its CSS layout box does not change. Because `srcset` selection only considers layout width × static device pixel ratio, `sizes="auto"` never upgrades the image to a higher-resolution candidate, leaving zoomed images visibly pixelated even when high-res candidates exist in `srcset`.

### 4. No author knob for target pixel density ([#13069](https://github.com/whatwg/html/issues/13069))
Different images on the same page have different quality vs. bandwidth priorities:
- **High-priority detail**: Hero product photography, artwork, or text-heavy diagrams that need extra sharpness.
- **Bandwidth-constrained grids**: Decorative backgrounds, avatars, or 50-item thumbnail grids where `0.75x` device density saves 40%+ bandwidth with negligible visual impact.

Today, the only way to nudge selection is to **lie in `sizes`** (e.g., `sizes="50vw"` for a `100vw` image). Lying in `sizes`:
- Is impossible when using `sizes="auto"`.
- Corrupts `img.naturalWidth` / `img.naturalHeight` (which divide by the fake density).
- Conflates layout geometry with quality preferences.

### 5. `naturalWidth` and `naturalHeight` hide true pixel dimensions ([#12565](https://github.com/whatwg/html/issues/12565))
When an image uses `w` descriptors, `img.naturalWidth` returns the density-adjusted CSS size (`pixelWidth / (w / sourceSize)`), **not** the decoded bitmap's pixel width. With `sizes="auto"`, `naturalWidth` simply echoes the element's rendered layout width!

As a result, scripts cannot answer basic questions like:
- *"Can the user zoom into this lightbox image without exceeding 1:1 native pixels?"*
- *"What are the actual pixel dimensions of the loaded bitmap for a canvas crop tool or View Transition?"*

Workarounds like `await createImageBitmap(img)` are asynchronous and allocate a duplicate GPU texture just to read two integers.

---

## Proposed Approach

We propose five focused, composable improvements:

```
┌──────────────────┐   ┌────────────────────┐   ┌──────────────────────┐
│ 2D Layout Size   │   │ Visual Scale       │   │ Author Density Knob  │
│ (sizes / auto +  │ × │ (DPR × pinch-zoom  │ × │ (image-density /     │
│  object-fit + h) │   │  × CSS transforms) │   │  srcset-density)     │
└────────┬─────────┘   └─────────┬──────────┘   └──────────┬───────────┘
         └───────────────────────┼─────────────────────────┘
                                 ▼
              ┌─────────────────────────────────────┐
              │ Normative Source Selection (#13077) │
              │ First candidate >= targetWidth      │
              └──────────────────┬──────────────────┘
                                 ▼
              ┌─────────────────────────────────────┐
              │ True Intrinsic Dimensions (#12565)  │
              │ img.decodedWidth / decodedHeight    │
              └─────────────────────────────────────┘
```

### 1. Define Source Selection Normatively ([#13077](https://github.com/whatwg/html/issues/13077))
Replace *"In an implementation-defined manner, choose one image source"* with the algorithm all three browser engines already ship:
1. Compute `targetDensity = effectiveDevicePixelRatio * densityMultiplier`.
2. Sort candidates in ascending order of their resolved density (`w / sourceSize` or `x` descriptor).
3. Select the **first candidate** whose density is `>= targetDensity`, or the **last (largest) candidate** if all candidates are `< targetDensity`.

**Preserving UA flexibility:**
- **Bounded `targetDensity` adjustment**: User agents may scale `targetDensity` based on explicit user preferences, OS/browser data-saver modes, or privacy settings.
- **Cache preference**: If a higher-density candidate from the same `srcset` is already in the HTTP cache, the UA may select it instead of initiating a network fetch for a lower-density candidate.

### 2. 2D `sizes`, `object-fit`, and the `h` Descriptor ([#12248](https://github.com/whatwg/html/issues/12248), [PR #13053](https://github.com/whatwg/html/pull/13053))
Allow responsive image selection to account for 2D container geometry and `object-fit` without any post-load feedback loop:

#### A. Revive the `h` width/height descriptor pair in `srcset`
```html
<img loading="lazy" sizes="auto"
     style="width: 400px; height: 400px; object-fit: cover"
     srcset="card-400.jpg 400w 300h,
             card-800.jpg 800w 600h,
             card-1200.jpg 1200w 900h">
```
- When `h` is present alongside `w`, the candidate's pre-fetch aspect ratio is `r = w / h` (falling back to the `<img width height>` or CSS `aspect-ratio` if `h` is omitted).
- **No infinite loop**: If the markup provides a pre-fetch aspect ratio hint (`w`/`h`, `<img width height>`, or CSS `aspect-ratio`), that hint continues to govern source selection even after the image loads.

#### B. Make `sizes="auto"` 2D- and `object-fit`-aware
For `sizes="auto"`, resolve `(W, H)` from the element's CSS **content box** and compute the effective source width using the computed `object-fit` and candidate aspect ratio `r`:
- `fill` / `none`: `W`
- `cover`: `max(W, H * r)`
- `contain`: `min(W, H * r)`
- `scale-down`: `min(W, H * r, w)`

#### C. Optional 2D syntax in `sizes` for eager images
For above-the-fold images that cannot use `loading="lazy" sizes="auto"`, extend `<source-size-value>` to accept an optional height and fit keyword:
```html
<img sizes="(max-width: 600px) 100vw * 300px cover, 400px * 400px cover"
     srcset="hero-800.jpg 800w 450h, hero-1600.jpg 1600w 900h">
```

### 3. Visual Scale in `sizes="auto"` ([#11428](https://github.com/whatwg/html/issues/11428), [PR #13046](https://github.com/whatwg/html/pull/13046))
When computing the auto-sizes width of an `<img>`:
- Multiply the layout box size by the **visual scale factor** (`visualViewport.scale` from pinch-zoom × 2D scale factor of ancestor CSS transforms).
- **Upgrade-only & debounced**: Zooming in upgrades to a sharper candidate once the gesture settles; zooming back out never re-fetches a lower-resolution image already in memory.

### 4. Author Knob for Target Pixel Density ([#13069](https://github.com/whatwg/html/issues/13069))
Provide a dedicated declarative knob to scale or clamp the target pixel density without lying about layout geometry in `sizes`:

#### Option A: CSS Property (`image-density` / `image-resolution`)
```css
/* Save bandwidth on a dense thumbnail grid */
.thumbnail-grid img {
  image-density: 0.75x; /* scales target density by 0.75, or clamp(0.5dppx, 0.75x, 1.5dppx) */
}

/* Request extra sharpness for a hero diagram */
.hero-diagram {
  image-density: 1.5x;
}
```
#### Option B: HTML Attribute (`imagedensity` / `srcset-density`)
```html
<img loading="lazy" sizes="auto" imagedensity="0.75"
     srcset="thumb-200.jpg 200w, thumb-400.jpg 400w, thumb-800.jpg 800w">
```
*Note: An HTML attribute works with the speculative preload scanner on eager images, while a CSS property integrates naturally with media queries and `sizes="auto"`. Both map to the same `densityMultiplier` in the normative selection algorithm.*

### 5. Expose True Decoded Pixel Dimensions ([#12565](https://github.com/whatwg/html/issues/12565))
Add synchronous, read-only properties on `HTMLImageElement` returning the actual pixel dimensions of the current image resource before density scaling:

```webidl
partial interface HTMLImageElement {
  readonly attribute unsigned long decodedWidth;
  readonly attribute unsigned long decodedHeight;
};
```
*(Bikeshedding names: `decodedWidth` / `decodedHeight` vs. `intrinsicWidth` / `intrinsicHeight` vs. `bitmapWidth` / `bitmapHeight`.)*

```js
const img = document.querySelector("img");
// Suppose 1200x900 image is rendered in a 400px box with sizes="auto":
img.naturalWidth;  // 400 (density-corrected CSS width)
img.decodedWidth;  // 1200 (actual pixel width of the loaded resource!)
img.decodedHeight; // 900  (actual pixel height of the loaded resource!)
```
- Returns `0` if the image is not yet available or failed to load.
- Subject to the exact same timing and CORS/tainting rules as `naturalWidth` / `naturalHeight`.

---

## Alternatives Considered

### Keep source selection implementation-defined
We could leave selection implementation-defined and only add hints. However, without a normative baseline algorithm, authors cannot predict what `imagedensity="0.75"` or `400px * 400px cover` will select, and WPT tests cannot verify interoperability across engines.

### Fix `naturalWidth` directly instead of adding `decodedWidth`
Changing `naturalWidth` to return raw bitmap pixels when `w` descriptors are used would break existing pages that rely on `naturalWidth` matching the density-corrected intrinsic size (for example, scripts that compute `img.width = img.naturalWidth`). Adding `decodedWidth` / `decodedHeight` preserves 100% web compatibility.

### Use Client Hints (`Sec-CH-DPR` / `Sec-CH-Width`) instead of client-side knobs
Client Hints require server-side image negotiation, CDN `Vary` configuration, and extra round-trips or policy headers. Declarative client-side `srcset` knobs work on any static host or CDN without server logic.

### Rely on JS `ResizeObserver` + `visualViewport` listeners
Authors can manually rewrite `sizes` or `srcset` from JavaScript on resize and pinch-zoom, but doing so defeats declarative HTML, requires main-thread script execution, and still cannot read true pixel dimensions without `createImageBitmap()`.

---

## Accessibility, Internationalization, and Performance

- **Accessibility**: Pinch-zoom responsiveness (`#11428`) directly benefits low-vision users who rely on pinch-zoom or CSS magnification to inspect images and diagrams.
- **Internationalization**: None affected.
- **Performance**:
  - `object-fit: contain` awareness prevents up to **10× over-fetching** on letterboxed images.
  - `image-density` / `imagedensity` lets authors reduce image byte weight on thumbnail-heavy pages without sacrificing `sizes="auto"` ergonomics.
  - `decodedWidth` / `decodedHeight` eliminates wasteful `createImageBitmap()` allocations.

---

## Stakeholder Feedback / Signals

- **Normative selection ([#13077](https://github.com/whatwg/html/issues/13077))**: Positive discussion across Blink (`@noamr`, `@pdr`) and WebKit (`@annevk`) noting that all three engines now use the same algorithm following Chrome's removal of geometric-mean selection.
- **2D `sizes="auto"` & `object-fit` ([#12248](https://github.com/whatwg/html/issues/12248), [PR #13053](https://github.com/whatwg/html/pull/13053))**: Active collaboration between Gecko (`@zcorpan`) and Blink (`@noamr`, `@sakhapov`) to eliminate the `concrete object size` oscillation loop and support `object-fit: cover` / `contain`.
- **Pinch-zoom & transforms ([#11428](https://github.com/whatwg/html/issues/11428), [PR #13046](https://github.com/whatwg/html/pull/13046))**: Spec PR drafted by `@sakhapov` with review from `@zcorpan` and `@noamr`.

---

## Security and Privacy Self-Review Questionnaire

Per the [W3C Security and Privacy Self-Review Questionnaire](https://w3c.github.io/security-questionnaire/):

### 2.1 What information does this feature expose, and for what purposes?
- `decodedWidth` and `decodedHeight` expose the raw pixel width and height of the loaded image resource. Note that this information is **already exposed** to scripts today via `naturalWidth * (wDescriptor / currentLayoutSize)` or via `createImageBitmap(img)` (for CORS-same-origin images) and via rendering an un-descriptor'd `new Image()` pointing to `img.currentSrc`.
- Pinch-zoom responsiveness in `sizes="auto"` selects higher-resolution `srcset` candidates when the user zooms in, which is already observable via `window.visualViewport.scale`.

### 2.2 Do features in your specification expose the minimum amount of information necessary to enable their intended uses?
Yes. `decodedWidth` / `decodedHeight` expose only two integer dimensions when the image is in the *completely available* state, under the exact same cross-origin restrictions as `naturalWidth` / `naturalHeight`.

### 2.3 How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?
The feature does not handle or access personal information or PII.

### 2.4 How do the features in your specification deal with sensitive information?
The feature does not handle or access sensitive information.

### 2.5 Does data exposed by your specification carry related but distinct information that may not be obvious to users?
If a user has enabled an OS- or browser-level data-saver mode that lowers `targetDensity` (permitted by the normative selection exceptions in `#13077`), the selected `currentSrc` reflects that preference—just as `srcset` does today.

### 2.6 Do the features in your specification introduce new state for an origin that persists across browsing sessions?
No new persistent state is introduced. Selected images use the standard HTTP cache partitioned by network state key.

### 2.7 Do the features in your specification expose information about the underlying platform to origins?
No new platform information is exposed beyond `window.devicePixelRatio` and `window.visualViewport.scale`, which are already exposed to scripts.

### 2.8 Does this specification allow an origin to send data to the underlying platform?
No.

### 2.9 Do features in this specification allow an origin access to sensors on a user's device?
No.

### 2.10 What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.
- `decodedWidth` and `decodedHeight` expose the unscaled intrinsic pixel width and height of `img.currentSrc`. This is identical to loading `img.currentSrc` in an `<img src="...">` without `w` descriptors and reading `naturalWidth` / `naturalHeight`.

### 2.11 Do features in this specification enable new script execution/loading mechanisms?
No. It only governs image candidate selection in `<img>` / `<picture>` and exposes integer dimension getters.

### 2.12 Do features in this specification allow an origin to access other devices?
No.

### 2.13 Do features in this specification allow an origin some measure of control over a user agent's native UI?
No.

### 2.14 What temporary identifiers do the features in this specification create or expose to the web?
None.

### 2.15 How does this specification distinguish between behavior in first-party and third-party contexts?
Same-origin vs. cross-origin image resources follow existing HTML `<img>` security rules. Cross-origin images without CORS approval do not expose metadata before load rules permit, and `sizes="auto"` inside cross-origin `<iframe>`s uses the iframe's own viewport and transforms.

### 2.16 How do the features in this specification work in the context of a browser's Private Browsing or Incognito mode?
Identically to normal browsing, with the HTTP cache isolated to the ephemeral profile. UAs may optionally clamp `targetDensity` in Private Browsing if they already clamp `devicePixelRatio` for fingerprinting protection.

### 2.17 Does this specification have both "Security Considerations" and "Privacy Considerations" sections?
Yes, Summarized below:
- **Security Considerations**: Eliminating the `concrete object size` oscillation loop (`#12248`) fixes a potential resource-exhaustion / infinite-refetch bug when `sizes="auto"` is combined with `object-fit: contain` and mismatched candidate aspect ratios. `decodedWidth` / `decodedHeight` must follow the exact same cross-origin availability rules as `naturalWidth` / `naturalHeight`.
- **Privacy Considerations**: Normative source selection (`#13077`) explicitly preserves the user agent's right to adjust `targetDensity` for privacy/anti-fingerprinting modes or data-saver settings.

### 2.18 Do features in your specification enable origins to downgrade default security protections?
No.

### 2.19 What should this questionnaire have asked?
Nothing additional.
