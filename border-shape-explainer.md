# Free-form borders with `border-shape`

## User need
Non-rectangular shapes are already everywhere on the web. However, the current limitations of the web platform allow using them only for clipping, or for standalone shapes in the form of SVG.
Creating a simple speech bubble that paints its border and clips its internal contents requires both clip-path, SVG, and multiple elements.

## Existing Solutions
Currently, the toolkit for web authors to decorate elements in a non-rectangular way contains:
- SVG: this requires setting up a tree of elements separate from styling, and clip the element separately.
- `clip-path`: this clips the border, as well as effects like shadow/blur.
- `corner-shape`: this allows more expressive shapes than before, however it is still limited to box-like shapes.

## Proposed solution
The proposed solution is to allow authors to specify one or two basic shapes that define the border's contour.
The given shape(s) would be strokes, and if two are given, the area between them would be filled.
The area inside the internal shape would clip the content for `overflow: hidden` (or scroll/clip etc.)
This is documented in the [draft spec](https://drafts.csswg.org/css-borders-4/#border-shape).

## Alternatives considered

### Changing the semantics of `clip-path`
This doesn't seem right because clipping a shape can be useful regardless of border-shaping.

### Taking the colors and width from the `border-*` properties
This is still under consideration, but it's complicated because it's unclear which "side" maps to which parts of an arbitrary shape.
Using `fill` and `stroke` is more of a straightforward solution.

This is also why `corner-shape` was needed despite of `border-shape` being more expressive.
`border-shape` and `corner-shape` have a different set of constraints, expressions as well as ergonomics, that are not possible with the alternative.
While `border-shape` allows full expression with shapes, this kind of flexibility comes at a cost: some of the aforementioned traditional "border" features become infeasible.
With `corner-shape`, we can have superellipse-defined corners and the full expression of 4 borders and 4 corners (width, color, style),
while with the general purpose `border-shape` the solution remains a uniform stroke width and color, plus whatever style of stroking available (joins, dash-array, caps).


## Privacy & Security Questionnaire

01.  What information does this feature expose,
     and for what purposes?

It does not.
     
2.  Do features in your specification expose the minimum amount of information
     necessary to implement the intended functionality?

Yes

3.  Do the features in your specification expose personal information,
     personally-identifiable information (PII), or information derived from
     either?

No

4.  How do the features in your specification deal with sensitive information?

N/A
5.  Does data exposed by your specification carry related but distinct
     information that may not be obvious to users?

N/A

6.  Do the features in your specification introduce state
     that persists across browsing sessions?

No

7.  Do the features in your specification expose information about the
     underlying platform to origins?

No

8.  Does this specification allow an origin to send data to the underlying
     platform?

No

9.  Do features in this specification enable access to device sensors?

No

10.  Do features in this specification enable new script execution/loading
     mechanisms?

No

11.  Do features in this specification allow an origin to access other devices?

No

12.  Do features in this specification allow an origin some measure of control over
     a user agent's native UI?

No

13.  What temporary identifiers do the features in this specification create or
     expose to the web?

None

14.  How does this specification distinguish between behavior in first-party and
     third-party contexts?

N/A

15.  How do the features in this specification work in the context of a browser’s
     Private Browsing or Incognito mode?

N/A

20.  Does this specification have both "Security Considerations" and "Privacy
     Considerations" sections?

Yes

21.  Do features in your specification enable origins to downgrade default
     security protections?

No

22.  What happens when a document that uses your feature is kept alive in BFCache
     (instead of getting destroyed) after navigation, and potentially gets reused
     on future navigations back to the document?

No

23.  What happens when a document that uses your feature gets disconnected?

Nothing much

24.  Does your spec define when and how new kinds of errors should be raised?

N/A

25.  Does your feature allow sites to learn about the user's use of assistive technology?

No

26.  What should this questionnaire have asked?

Nothing in particular
