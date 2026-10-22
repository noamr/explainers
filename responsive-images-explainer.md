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
- **User Agents are designed for interoperability, while websites are best suited to measure context-specific business trade-offs.** A browser cannot know whether a given `<img>` is a conversion-critical product photo (where blurriness hurts sales), an interactive hero that will scale up on click, or the 40th thumbnail in a scrollable feed (where saving 45% of bytes improves LCP with negligible quality loss).
- **Browser selection has converged and can be standardized.** Chrome retired its geometric-mean density rounding and `Save-Data` clamping experiments in late 2024, meaning WebKit, Gecko, and Blink now all use the same baseline selection rule (`first candidate >= targetDensity`). Chrome also retains a useful optimization to avoid downloading a lower-resolution source when a higher-resolution candidate is already loaded/cached—behavior that can be **standardized directly** rather than leaving the selection algorithm implementation-defined.

**The theme of this proposal:**
1. **Standardize the browser baseline (`#13077`)** — define source selection (including reusing already-loaded higher-resolution sources) normatively for interoperability.
2. **Give the knobs to the website (`#13069`, `#12248`, `#11428`, `#12565`)** — empower the site owner (who knows the content and can A/B test UX and business impact) to control target density, 2D `object-fit` sizing (`cover`, `contain`, `scale-down`, `none`), and true pixel inspection.

---

## User-Facing Problems: Blurry Images and Wasted Bandwidth

Because websites lack proper knobs—and because `sizes` conflates 1D layout width, candidate selection, and `naturalWidth`—users routinely suffer from either **blurred images** or **overused bandwidth**:

### 1. No way to tune quality vs. bandwidth per image ([#13069](https://github.com/whatwg/html/issues/13069))
When selecting from a `w`-descriptor `srcset`, the browser always targets `1.0 × devicePixelRatio` of the resolved `sizes` slot:
- **Wasted bandwidth on low-priority images**: A `3x` phone downloads `9×` the pixels of a `1x` display. For secondary thumbnails, avatars, or far-below-the-fold cards, `0.75x` density cuts pixel count by **44%** with no noticeable degradation, freeing bandwidth for critical LCP content.
- **Blurry high-priority or zoomable images**: Conversely, a product photo that users zoom into or a hero image that scales up during a transition may need `1.5x` density headroom to stay sharp.

Today, sites must **lie in markup** to nudge selection:
- **Lying in `sizes`** (e.g., writing `sizes="75vw"` for a `100vw` image) is **impossible with `sizes="auto"`** and corrupts `img.naturalWidth`.
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

Today, WebKit, Gecko, and Blink all implement the same baseline rule (`first candidate >= targetDensity`), and Chrome's optimization to keep an already-loaded higher-resolution candidate rather than re-fetching a smaller one can be standardized. Leaving the entire selection step "implementation-defined" blocks normative Web Platform Tests (WPT) and prevents authors from having a guaranteed baseline for new knobs.

### 5. Blindfolds on true pixel dimensions ([#12565](https://github.com/whatwg/html/issues/12565), [PR #13046](https://github.com/whatwg/html/pull/13046))
Because `w` descriptors divide `naturalWidth` by `density` (and with [PR #13046](https://github.com/whatwg/html/pull/13046) `naturalWidth` always equals `sourceSize`), under `sizes="auto"`, `img.naturalWidth` echoes the element's CSS layout width (`img.clientWidth`).

As a result, websites cannot read the true pixel dimensions of the bitmap the browser selected:
- **RUM / performance telemetry** cannot measure whether pages are over-fetching (wasting bandwidth) or under-fetching (showing blurry images) in production.
- **Lightbox and zoom UIs** cannot check whether the currently loaded image has enough real pixels to allow zooming in without exceeding `1:1` native pixels, forcing an offscreen `new Image()` workaround.

---

## Proposed Approach: Standard Baseline + Website Knobs

```
Browser Responsibility (Interop)          Website Responsibility (Business & UX Knobs)
────────────────────────────────          ────────────────────────────────────────────
• Normative selection (#13077):           • Target Density Knob (#13069):
  Pick first candidate >= target            Scale density up/down per image (densityscale)
  & reuse loaded higher-res source        • 2D Layout & object-fit (#12248):
• Respect native size bounds for            Declare w/h aspect ratio & cover/contain
  object-fit: scale-down / none (#11428)  • True Pixel Inspection (#12565):
                                            Read resourceWidth / resourceHeight for RUM & zoom
```

### 1. Define Source Selection Normatively ([#13077](https://github.com/whatwg/html/issues/13077))
Replace *"In an implementation-defined manner, choose one image source"* with a normative algorithm:
1. Let `density` be the `devicePixelRatio` of the document's viewport (including page zoom), multiplied by the element's author density scale (`#13069`). *The user agent may adjust `density` to reflect explicit user preferences (such as a reduced-data mode or density cap) or for privacy reasons.*
2. Sort `sourceSet` in ascending order of pixel density (using a stable sort so document order breaks ties).
3. Let `selectedSource` be the **first entry** in `sourceSet` whose pixel density is `>= density`, or the **last entry** if no such entry exists.
4. **Standardize higher-resolution reuse**: The user agent may instead select an entry with a pixel density greater than `selectedSource`'s if that entry's image data is already loaded or available in a local cache.

### 2. Give Websites a Target Density Knob ([#13069](https://github.com/whatwg/html/issues/13069))
Let authors explicitly tune the sharpness-vs-bandwidth trade-off per image without lying in `sizes` or `w` descriptors:

```html
<!-- Below-the-fold thumbnail grid: save ~44% pixels with sizes="auto" -->
<img loading="lazy" sizes="auto" densityscale="0.75"
     srcset="thumb-400.jpg 400w, thumb-800.jpg 800w, thumb-1200.jpg 1200w">

<!-- Conversion-critical or zoomable product hero: ask for 1.5x density headroom -->
<img loading="lazy" sizes="auto" densityscale="1.5"
     srcset="hero-600.jpg 600w, hero-900.jpg 900w, hero-1200.jpg 1200w">
```
*(Can be exposed as an HTML attribute `densityscale`—and/or coarse `resolutionpriority="low|high|auto"`—or a CSS property so it works both with the preload scanner and with `sizes="auto"`.)*

- Multiplies target density (`targetDensity = DPR * densityscale`) after `sizes` is resolved.
- Keeps `sizes` and `w` descriptors 100% honest, preserving CDN cache reuse and `naturalWidth`.

### 3. 2D `sizes`, `h` Descriptors, and `object-fit` ([#12248](https://github.com/whatwg/html/issues/12248), [#11428](https://github.com/whatwg/html/issues/11428), [PR #13053](https://github.com/whatwg/html/pull/13053))

#### A. Revive `h` descriptors in `srcset` as a pre-fetch aspect-ratio hint
```html
<img loading="lazy" sizes="auto"
     style="width: 400px; height: 400px; object-fit: cover"
     srcset="card-400.jpg 400w 300h,
             card-800.jpg 800w 600h,
             card-1200.jpg 1200w 900h">
```
- Each candidate's `source aspect ratio` is `r = w / h` (falling back to the element's `width` and `height` attributes).
- **Zero infinite loops**: `h` only affects candidate selection, and the pre-fetch hint remains authoritative after load—the downloaded image's intrinsic ratio never feeds back into source selection.

#### B. Make `sizes="auto"` 2D- and `object-fit`-aware
Resolve `(width, height)` from the CSS **content box** and compute the candidate's **source concrete width** using `object-fit` and `r = w / h`:
- `cover`: `max(width, height * r)` — prevents blurry cropped cards.
- `contain` / `scale-down`: `min(width, height * r)` — prevents 10× over-fetching on letterboxed boxes.
- `fill` / `none`: `width`

#### C. Extend `sizes` grammar for eager images ([PR #13053](https://github.com/whatwg/html/pull/13053))
```html
<img srcset="photo-400.jpg 400w 300h, photo-800.jpg 800w 600h"
     sizes="(max-width: 600px) 100vw * 300px cover, 400px * 400px cover">
```

#### D. Fix unexpected upscaling under `object-fit: scale-down` and `none` ([#11428](https://github.com/whatwg/html/issues/11428))
When `object-fit: scale-down` or `object-fit: none` is used on a `w`-descriptor image without explicit `width`/`height` attributes, prevent sub-`1x` (or sub-DPR) candidate densities from inflating `naturalWidth` up to the container's `sizes` width—so a `200w` image inside an `800px` `object-fit: scale-down` container stays crisp at its native size rather than upscaling to `800px`.

### 4. Expose True Intrinsic Resource Dimensions ([#12565](https://github.com/whatwg/html/issues/12565))
Expose the uncorrected pixel dimensions of the loaded image resource (and/or its selected pixel density) on `HTMLImageElement`:

```webidl
partial interface HTMLImageElement {
  readonly attribute unsigned long resourceWidth;
  readonly attribute unsigned long resourceHeight;
  readonly attribute double currentPixelDensity;
};
```
*(Bikeshedding names: `resourceWidth` / `resourceHeight` vs. `intrinsicWidth` / `intrinsicHeight` vs. `decodedWidth` / `decodedHeight`.)*

```js
const img = document.querySelector("img");
// 800x600 image (800w) rendered in a 400px box with sizes="auto":
img.naturalWidth;   // 400 (density-corrected CSS width)
img.resourceWidth;  // 800 (true pixel width of the loaded file!)
img.resourceHeight; // 600 (true pixel height of the loaded file!)
```
- Lets sites measure real-world image over-fetching and under-fetching in RUM telemetry.
- Lets lightbox and zoom components check available pixel headroom synchronously without creating an offscreen `new Image()`.

---

## Alternatives Considered

### Keep selection entirely implementation-defined
We could leave selection implementation-defined and rely solely on UA heuristics. However, browsers lack per-image business context to know which images can tolerate lower density and which require extra sharpness, while useful UA optimizations like reusing an already-loaded higher-resolution image can be standardized for all engines.

### Keep lying in `sizes` or `w` descriptors to control density
Authors currently write `sizes="50vw"` on a `100vw` image (which breaks `sizes="auto"` and `naturalWidth`) or lie in `w` descriptors (`img.jpg?w=720 800w`, which breaks CDN cache reuse and 2D `w`/`h` aspect ratios). A dedicated `densityscale` attribute avoids both problems.

### Change `naturalWidth` directly instead of adding `resourceWidth`
Changing `naturalWidth` to return raw file pixels for `w`-descriptor images would break existing web pages that rely on `naturalWidth` reflecting the density-corrected CSS size (including [PR #13046](https://github.com/whatwg/html/pull/13046)). Adding `resourceWidth` / `resourceHeight` preserves full backward compatibility.

### Server-side Client Hints (`Sec-CH-DPR` / `Sec-CH-Width`)
Client Hints require server-side negotiation, CDN `Vary` configuration, and extra policy headers. Declarative client-side `srcset` knobs work on any static host or CDN and compose directly with `sizes="auto"`.

---

## Accessibility, Internationalization, and Performance

- **Accessibility**: Fixing `object-fit: cover` under-fetching (`#12248`), `scale-down`/`none` upscaling (`#11428`), and enabling `densityscale` (`#13069`) ensures diagrams, text-in-images, and zoomable photos remain sharp and legible.
- **Internationalization**: None affected.
- **Performance & Business Impact**:
  - Sites can cut secondary/below-the-fold image bandwidth by **40–50%** using `densityscale` (`#13069`) and up to **90%** on letterboxed `object-fit: contain` images (`#12248`).
  - Sites can eliminate blurry `object-fit: cover` cards (`#12248`) and measure their exact pixel efficiency in RUM via `resourceWidth` (`#12565`).

---

## Stakeholder Feedback / Signals

- **Normative selection ([#13077](https://github.com/whatwg/html/issues/13077))**: Positive discussion across Blink (`@noamr`, `@pdr`) and WebKit (`@annevk`) noting that all three engines now use the same baseline rule.
- **2D `sizes="auto"`, `h` descriptors, & `object-fit` ([#12248](https://github.com/whatwg/html/issues/12248), [#11428](https://github.com/whatwg/html/issues/11428), [PR #13053](https://github.com/whatwg/html/pull/13053))**: Active collaboration between Gecko (`@zcorpan`), Blink (`@noamr`), and web developers (`@austinh`, Airbnb).
- **Density control ([#13069](https://github.com/whatwg/html/issues/13069))**: Strong production motivation and real-world data shared by Airbnb (`@austinh`) and Cloudinary (`@eeeps`).

---

## Security and Privacy Self-Review Questionnaire

Per the [W3C Security and Privacy Self-Review Questionnaire](https://w3c.github.io/security-questionnaire/):

### 2.1 What information does this feature expose, and for what purposes?
`resourceWidth` and `resourceHeight` expose the raw pixel width and height of the loaded image resource so sites can manage zoom bounds and measure image sizing accuracy. This information is **already exposed** today by loading `img.currentSrc` in an `<img>` without `w` descriptors and reading `naturalWidth` / `naturalHeight`, or via `createImageBitmap(img)`.

### 2.2 Do features in your specification expose the minimum amount of information necessary to enable their intended uses?
Yes. `resourceWidth` / `resourceHeight` expose only integer pixel dimensions when the image is *completely available*, governed by the exact same cross-origin rules as `naturalWidth` / `naturalHeight`.

### 2.3 How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?
The feature does not handle or access personal information or PII.

### 2.4 How do the features in your specification deal with sensitive information?
The feature does not handle or access sensitive information.

### 2.5 Does data exposed by your specification carry related but distinct information that may not be obvious to users?
If a user has enabled an OS- or browser-level data-saver setting that lowers `targetDensity` (permitted by `#13077`), the selected `currentSrc` reflects that preference—identical to how `srcset` behaves today.

### 2.6 Do the features in your specification introduce new state for an origin that persists across browsing sessions?
No new persistent state is introduced.

### 2.7 Do the features in your specification expose information about the underlying platform to origins?
No new platform information is exposed beyond `window.devicePixelRatio`, which is already exposed to scripts.

### 2.8 Does this specification allow an origin to send data to the underlying platform?
No.

### 2.9 Do features in this specification allow an origin access to sensors on a user's device?
No.

### 2.10 What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.
`resourceWidth` and `resourceHeight` expose the unscaled intrinsic pixel width and height of `img.currentSrc`. This is identical to loading `img.currentSrc` in an `<img src="...">` without `w` descriptors and reading `naturalWidth` / `naturalHeight`.

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

### 2.16 How do the features in this specification work in the context of a browser's Private Browsing or Incognito mode?
Identically to normal browsing, with the HTTP cache isolated to the ephemeral profile. UAs may clamp `targetDensity` in Private Browsing if they clamp `devicePixelRatio` for fingerprinting protection.

### 2.17 Does this specification have both "Security Considerations" and "Privacy Considerations" sections?
Yes:
- **Security Considerations**: Eliminating the `concrete object size` oscillation loop (`#12248` / PR `#13053`) fixes a potential infinite-refetch resource-exhaustion bug when `sizes="auto"` is combined with `object-fit: contain`. `resourceWidth` / `resourceHeight` follow the same cross-origin availability rules as `naturalWidth` / `naturalHeight`.
- **Privacy Considerations**: Normative source selection (`#13077`) preserves the user agent's ability to adjust `targetDensity` for explicit user privacy/anti-fingerprinting or data-saver settings.

### 2.18 Do features in your specification enable origins to downgrade default security protections?
No.

### 2.19 What should this questionnaire have asked?
Nothing additional.
