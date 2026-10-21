# Better Responsive Image Knobs

## Authors:
- Noam Rosenthal

## Participate
- [whatwg/html#13077 — Define responsive image source selection normatively](https://github.com/whatwg/html/issues/13077)
- [whatwg/html#13069 — Let authors request a higher or lower pixel density from `srcset`](https://github.com/whatwg/html/issues/13069)
- [whatwg/html#12248 — `sizes="auto"` and `object-fit` (`contain` / `cover`)](https://github.com/whatwg/html/issues/12248) ([PR #13053](https://github.com/whatwg/html/pull/13053))
- [whatwg/html#11428 — Account for pinch-zoom and CSS transforms in `sizes="auto"`](https://github.com/whatwg/html/issues/11428) ([PR #13046](https://github.com/whatwg/html/pull/13046))
- [whatwg/html#12565 — Expose an image's true intrinsic pixel dimensions when `w` descriptors are used](https://github.com/whatwg/html/issues/12565)

---

## Introduction: Moving the Knobs from the Browser to the Website

Every responsive image on the web sits on a trade-off curve between **visual sharpness** and **bandwidth**:
- Pick a candidate that is too small, and the user sees a **blurry, pixelated image**.
- Pick a candidate that is too large, and the user pays in **wasted bandwidth, slower page loads, and higher memory use**.

When `srcset` and `sizes` were designed in 2014, the specification assumed that **User Agents** would manage this trade-off automatically. The spec gave authors a minimal 1D width hint (`sizes`) and left candidate selection *"implementation-defined"* so browsers could experiment with global bandwidth-saving heuristics.

A decade of production experience has shown that this responsibility was placed on the wrong party:
- **User Agents are built for interoperability, not context-specific business trade-offs.** A browser has no way of knowing whether a given `<img>` is a conversion-critical product photo (where blurriness hurts sales), a medical or technical diagram (where legibility is essential), or the 40th thumbnail in a scrollable feed (where saving 50% of bytes improves LCP with zero noticeable quality loss).
- **Global UA heuristics backfired.** When Chrome tried to save bandwidth globally using geometric-mean density thresholds, site owners complained about blurry images and worked around the browser by lying in `sizes` (e.g., `sizes="200vw"`). Chrome ultimately removed those heuristics in late 2024, and today all three major engines run the exact same deterministic selection rule.

**The theme of this proposal is simple:**
1. **Make the browser predictable (`#13077`)** — lock in a normative, interoperable baseline for `srcset` selection.
2. **Give the knobs to the website (`#13069`, `#12248`, `#11428`, `#12565`)** — empower the site owner (who knows the content and can A/B test business and UX impact) to control pixel density, 2D `object-fit` sizing, zoom upgrading, and bitmap inspection.

---

## User-Facing Problems: Blurry Images and Wasted Bandwidth

Because websites lack proper knobs—and because `sizes` is restricted to a 1D horizontal width—users routinely suffer from either **blurred images** or **overused bandwidth**:

### 1. No way to tune quality vs. bandwidth per image ([#13069](https://github.com/whatwg/html/issues/13069))
On a modern `3x` mobile screen, a browser blindly requests `3x` bitmaps for every image on the page:
- **Wasted bandwidth on low-priority images**: A grid of 50 thumbnails, avatars, or decorative cards rarely benefits from full `3x` density. Serving `1.5x` or `2x` (`~0.7x` of default density) cuts byte weight by **40–55%** without visible degradation, directly improving LCP and data usage.
- **Blurry high-priority images**: Conversely, an e-commerce hero image, fine art print, or text-heavy diagram on a `1x` or `1.5x` display may need a higher-density source to look crisp.

Today, the only workaround is to **lie about layout width in `sizes`** (e.g., writing `sizes="50vw"` for a `100vw` image). Lying in `sizes`:
- Is **impossible** when using `loading="lazy" sizes="auto"`.
- Corrupts `img.naturalWidth` and `img.naturalHeight` (which divide the bitmap size by the fake density).
- Mixes layout geometry with quality policy.

### 2. Blurry `object-fit: cover` cards & bloated `object-fit: contain` banners ([#12248](https://github.com/whatwg/html/issues/12248))
`sizes` only describes 1D width, ignoring container height and `object-fit`:
- **Blurry images with `object-fit: cover`**: A 4:3 photo inside a `400px × 400px` square card (`object-fit: cover`) is scaled to `533px × 400px` and cropped horizontally. A 1D `sizes="400px"` (or `sizes="auto"` reading content-box width) under-fetches resolution by **33%**—and by **78%** for a 16:9 image in a square card—leaving cards visibly soft.
- **Massive bandwidth waste with `object-fit: contain`**: A 1:2 portrait image inside a wide `1000px × 200px` container renders at `100px × 200px`. A 1D width of `1000px` fetches an image **10× too wide** (**100× too many pixels**).
- **Infinite download loops in the current spec**: The HTML spec currently defines `sizes="auto"` using CSS *concrete object size*, which depends on the loaded image's aspect ratio. If candidates in `srcset` have slightly different aspect ratios under `object-fit: contain`, loading a candidate changes the concrete width, triggering a re-fetch in an **infinite download loop**.

### 3. Blurry images when users pinch-zoom or scale ([#11428](https://github.com/whatwg/html/issues/11428))
When a user pinch-zooms into a product photo, map, or comic strip—or when a site scales an image with CSS `transform: scale(2)`—the CSS layout width does not change. Even when high-resolution candidates are right there in `srcset`, `sizes="auto"` never upgrades the image, leaving the user staring at a blurry bitmap.

### 4. Unpredictable "implementation-defined" spec baseline ([#13077](https://github.com/whatwg/html/issues/13077))
The HTML Standard still says:
> *"In an implementation-defined manner, choose one image source..."*

Even though Blink, Gecko, and WebKit now all implement the exact same rule (sort ascending by density and pick the first candidate `>= targetDensity`), leaving the spec "implementation-defined" prevents sites from relying on deterministic behavior when tuning their `srcset` breakpoints and density knobs, and blocks normative Web Platform Tests (WPT).

### 5. Blindfolds on script-driven zoom and inspection ([#12565](https://github.com/whatwg/html/issues/12565))
When a site uses `w` descriptors, `img.naturalWidth` and `img.naturalHeight` do **not** return the loaded bitmap's pixel dimensions—they return the density-adjusted CSS size (`pixelWidth / density`). With `sizes="auto"`, `naturalWidth` simply echoes the element's CSS layout width!

Because the site cannot read the true pixel dimensions of the bitmap the browser selected:
- Lightbox and product-zoom components cannot know how far the user can zoom before exceeding `1:1` native pixels (causing accidental blurriness).
- Performance monitoring scripts (RUM) cannot measure whether pages are over-fetching or under-fetching image pixels relative to rendered size.
- Workarounds like `await createImageBitmap(img)` are asynchronous and allocate a duplicate GPU texture just to read two integers.

---

## Proposed Approach: Interoperable Baseline + Author Knobs

```
Browser Responsibility (Interop)          Website Responsibility (Business & UX Knobs)
────────────────────────────────          ────────────────────────────────────────────
• Normative selection (#13077):           • Target Density Knob (#13069):
  Pick first candidate >= target            Scale density up/down per image role
• Safe 2D & zoom math without             • 2D Layout & object-fit (#12248):
  infinite loops                            Declare w/h aspect ratio & cover/contain
                                          • Visual Scale / Zoom (#11428):
                                            Auto-upgrade on pinch-zoom & transforms
                                          • True Pixel Inspection (#12565):
                                            Read decodedWidth / decodedHeight for RUM & zoom
```

### 1. Lock in Normative Source Selection ([#13077](https://github.com/whatwg/html/issues/13077))
Replace *"In an implementation-defined manner, choose one image source"* with the deterministic algorithm all three engines already ship:
1. Compute `targetDensity = effectiveDevicePixelRatio * authorDensityMultiplier`.
2. Sort candidates in ascending order of resolved density (`w / sourceSize` or `x` descriptor).
3. Select the **first candidate** with `density >= targetDensity`, or the **largest candidate** if none reaches `targetDensity`.

**Scoped UA exceptions:**
- **User & OS settings**: User agents may adjust `targetDensity` for explicit user preferences (e.g., browser/OS Data Saver mode or privacy/anti-fingerprinting modes), rather than opaque, unannounced heuristics.
- **Cache upgrade**: If a higher-resolution candidate is already in the HTTP cache, the UA may use it instead of making a new network request for a smaller one.

### 2. Give Websites a Target Density Knob ([#13069](https://github.com/whatwg/html/issues/13069))
Let authors explicitly tune the sharpness-vs-bandwidth trade-off per image without lying in `sizes`:

#### Option A: CSS Property (`image-density` / `image-resolution`)
```css
/* Cut bandwidth by ~44% on a 50-item thumbnail feed */
.feed-thumbnail {
  image-density: 0.75x;
}

/* Ensure crisp detail on conversion-critical product heroes */
.product-hero {
  image-density: 1.5x;
}
```
#### Option B: HTML Attribute (`imagedensity` / `srcset-density`)
```html
<img loading="lazy" sizes="auto" imagedensity="0.75"
     srcset="thumb-200.jpg 200w, thumb-400.jpg 400w, thumb-800.jpg 800w">
```
- Works seamlessly with `sizes="auto"`.
- Preserves accurate `naturalWidth` / `naturalHeight` layout geometry.
- Lets sites A/B test the exact density multiplier that maximizes user engagement and conversion.

### 3. 2D `sizes`, `object-fit`, and the `h` Descriptor ([#12248](https://github.com/whatwg/html/issues/12248), [PR #13053](https://github.com/whatwg/html/pull/13053))
Eliminate both blurry `cover` images and bloated `contain` fetches:

#### A. Revive `h` descriptors in `srcset` for pre-fetch aspect ratio
```html
<img loading="lazy" sizes="auto"
     style="width: 400px; height: 400px; object-fit: cover"
     srcset="card-400.jpg 400w 300h,
             card-800.jpg 800w 600h,
             card-1200.jpg 1200w 900h">
```
- When `h` is specified alongside `w`, the candidate's aspect ratio is `r = w / h` (falling back to `<img width height>` or CSS `aspect-ratio`).
- **Zero feedback loop**: Pre-fetch aspect ratio hints remain authoritative after load, preventing the `concrete object size` infinite download loop.

#### B. Make `sizes="auto"` 2D- and `object-fit`-aware
Resolve `(W, H)` from the CSS **content box** and compute the effective source width from `object-fit`:
- `cover`: `max(W, H * r)` — fetches enough horizontal pixels so the cropped card isn't blurry.
- `contain`: `min(W, H * r)` — avoids downloading 10× extra pixels for letterboxed images.
- `scale-down`: `min(W, H * r, w)`
- `fill` / `none`: `W`

#### C. Optional 2D `sizes` syntax for eager images
```html
<img sizes="(max-width: 600px) 100vw * 300px cover, 400px * 400px cover"
     srcset="hero-800.jpg 800w 450h, hero-1600.jpg 1600w 900h">
```

### 4. Upgrade `sizes="auto"` on Pinch-Zoom & CSS Transforms ([#11428](https://github.com/whatwg/html/issues/11428), [PR #13046](https://github.com/whatwg/html/pull/13046))
- Multiply the `sizes="auto"` box size by the **visual scale factor** (`visualViewport.scale` from pinch-zoom × ancestor 2D CSS transform scale).
- **Upgrade-only & debounced**: Sites can serve a lightweight initial image for fast page load, knowing the browser will automatically upgrade to a high-res `srcset` candidate if the user zooms in to inspect detail.

### 5. Expose True Decoded Pixel Dimensions ([#12565](https://github.com/whatwg/html/issues/12565))
Give sites synchronous visibility into the actual pixel dimensions of the selected bitmap:

```webidl
partial interface HTMLImageElement {
  readonly attribute unsigned long decodedWidth;
  readonly attribute unsigned long decodedHeight;
};
```

```js
const img = document.querySelector("img");
// 1200x900 bitmap rendered in a 400px box with sizes="auto":
img.naturalWidth;  // 400  (CSS layout width)
img.decodedWidth;  // 1200 (true bitmap pixel width!)
img.decodedHeight; // 900  (true bitmap pixel height!)
```
- Enables RUM telemetry to measure real-world over-fetching (wasted bytes) and under-fetching (blurry images) in production.
- Lets custom zoom/lightbox UIs cap zoom at `1:1` native bitmap pixels without allocating a GPU texture via `createImageBitmap(img)`.

---

## Alternatives Considered

### Keep relying on User Agent heuristics
We could leave selection implementation-defined and hope browsers invent smarter bandwidth-saving heuristics. History showed this does not work: browsers lack per-image product context, causing global heuristics to blur important images while still over-fetching unimportant ones.

### Keep lying in `sizes` to control density
Authors currently write `sizes="50vw"` on a `100vw` image to save bandwidth, or `sizes="200vw"` to force sharpness. This is incompatible with `sizes="auto"`, breaks `naturalWidth`, and becomes unmanageable when combined with 2D `object-fit`.

### Fix `naturalWidth` directly instead of adding `decodedWidth`
Changing `naturalWidth` to return raw bitmap pixels when `w` descriptors are present would break existing scripts that expect `naturalWidth` to reflect density-corrected CSS pixels. Adding `decodedWidth` / `decodedHeight` is 100% web-compatible.

### Server-side Client Hints (`Sec-CH-DPR` / `Sec-CH-Width`)
Client Hints move adaptation to the server, requiring custom image CDNs, `Vary` header management, and extra policy configuration. Declarative `srcset` knobs work everywhere—including static hosts—and compose directly with `sizes="auto"`.

---

## Accessibility, Internationalization, and Performance

- **Accessibility**: Pinch-zoom upgrading (`#11428`) ensures low-vision users who zoom into diagrams or photos receive sharp, legible imagery without penalizing initial page load for everyone else.
- **Internationalization**: None affected.
- **Performance & Business Impact**:
  - Sites can cut thumbnail/feed image bandwidth by **40–50%** using the density knob (`#13069`) and up to **90%** on letterboxed `object-fit: contain` images (`#12248`).
  - Sites can eliminate blurry `object-fit: cover` cards (`#12248`) and measure their exact pixel efficiency in RUM via `decodedWidth` (`#12565`).

---

## Stakeholder Feedback / Signals

- **Normative selection ([#13077](https://github.com/whatwg/html/issues/13077))**: Positive alignment across Blink (`@noamr`, `@pdr`) and WebKit (`@annevk`) now that all three engines implement the same rule.
- **2D `sizes="auto"` & `object-fit` ([#12248](https://github.com/whatwg/html/issues/12248), [PR #13053](https://github.com/whatwg/html/pull/13053))**: Active collaboration between Gecko (`@zcorpan`) and Blink (`@noamr`, `@sakhapov`).
- **Pinch-zoom & transforms ([#11428](https://github.com/whatwg/html/issues/11428), [PR #13046](https://github.com/whatwg/html/pull/13046))**: Spec PR drafted by `@sakhapov` with review from `@zcorpan` and `@noamr`.

---

## Security and Privacy Self-Review Questionnaire

Per the [W3C Security and Privacy Self-Review Questionnaire](https://w3c.github.io/security-questionnaire/):

### 2.1 What information does this feature expose, and for what purposes?
- `decodedWidth` and `decodedHeight` expose the raw pixel width and height of the loaded image resource so sites can manage zoom bounds and measure image sizing accuracy. This information is **already exposed** today via `naturalWidth * (wDescriptor / currentLayoutSize)`, `createImageBitmap(img)`, or loading `img.currentSrc` in an `<img>` without `w` descriptors.
- Pinch-zoom upgrading in `sizes="auto"` selects higher-resolution `srcset` candidates when the user zooms in, matching `window.visualViewport.scale`.

### 2.2 Do features in your specification expose the minimum amount of information necessary to enable their intended uses?
Yes. `decodedWidth` / `decodedHeight` expose only two integer dimensions when the image is *completely available*, governed by the exact same cross-origin rules as `naturalWidth` / `naturalHeight`.

### 2.3 How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?
The feature does not handle or access personal information or PII.

### 2.4 How do the features in your specification deal with sensitive information?
The feature does not handle or access sensitive information.

### 2.5 Does data exposed by your specification carry related but distinct information that may not be obvious to users?
If a user has enabled an OS- or browser-level data-saver setting that lowers `targetDensity` (permitted by `#13077`), the selected `currentSrc` reflects that preference—identical to how `srcset` behaves today.

### 2.6 Do the features in your specification introduce new state for an origin that persists across browsing sessions?
No new persistent state is introduced.

### 2.7 Do the features in your specification expose information about the underlying platform to origins?
No new platform information is exposed beyond `window.devicePixelRatio` and `window.visualViewport.scale`, which are already exposed to scripts.

### 2.8 Does this specification allow an origin to send data to the underlying platform?
No.

### 2.9 Do features in this specification allow an origin access to sensors on a user's device?
No.

### 2.10 What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.
`decodedWidth` and `decodedHeight` expose the unscaled intrinsic pixel width and height of `img.currentSrc`. This is identical to loading `img.currentSrc` in an `<img src="...">` without `w` descriptors and reading `naturalWidth` / `naturalHeight`.

### 2.11 Do features in this specification enable new script execution/loading mechanisms?
No.

### 2.12 Do features in this specification allow an origin to access other devices?
No.

### 2.13 Do features in this specification allow an origin some measure of control over a user agent's native UI?
No.

### 2.14 What temporary identifiers do the features in this specification create or expose to the web?
None.

### 2.15 How does this specification distinguish between behavior in first-party and third-party contexts?
Same-origin vs. cross-origin image resources follow existing HTML `<img>` security rules. `sizes="auto"` inside cross-origin `<iframe>`s uses the iframe's own viewport and transforms.

### 2.16 How do the features in this specification work in the context of a browser's Private Browsing or Incognito mode?
Identically to normal browsing, with the HTTP cache isolated to the ephemeral profile. UAs may clamp `targetDensity` in Private Browsing if they clamp `devicePixelRatio` for fingerprinting protection.

### 2.17 Does this specification have both "Security Considerations" and "Privacy Considerations" sections?
Yes:
- **Security Considerations**: Eliminating the `concrete object size` oscillation loop (`#12248`) fixes a potential infinite-refetch resource-exhaustion bug when `sizes="auto"` is combined with `object-fit: contain`. `decodedWidth` / `decodedHeight` follow the same cross-origin availability rules as `naturalWidth` / `naturalHeight`.
- **Privacy Considerations**: Normative source selection (`#13077`) preserves the user agent's ability to adjust `targetDensity` for explicit user privacy/anti-fingerprinting or data-saver settings.

### 2.18 Do features in your specification enable origins to downgrade default security protections?
No.

### 2.19 What should this questionnaire have asked?
Nothing additional.
