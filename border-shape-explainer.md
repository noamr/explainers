#  `border-shape`: Beyond Rectangular Borders

You know how much of the web goes beyond simple rectangles these days. Think of speech bubbles, custom buttons, or unique layout elements. While the web platform lets us create these with tools like `clip-path` or SVG, adding a border to them or clipping their content cleanly can be surprisingly tricky.

### The Challenge with Current Solutions

Right now, if you want a non-rectangular shape with a border and content that stays within those bounds, you often run into limitations:

* **SVG:** Powerful, but it means managing a separate set of elements from your regular HTML structure. If you just want a border, it can feel like overkill.
* **`clip-path`:** This is great for cutting out custom shapes, but it clips everything – including any shadows or visual effects you might want on your border. It doesn't actually *draw* a border.
  Achieving a `border-shape`-like effect with `clip-path` is quite tricky, as it requires nesting the inner clipping element in the outer one, and dealing with `box-shadow` separately. 
* **`corner-shape`:** This CSS property offers more advanced control over rounded corners, allowing for more organic or complex corner designs. However, it's still fundamentally about modifying a box's corners, not creating entirely freeform shapes.

Consider a simple speech bubble: making its border paint correctly and clipping its internal text usually requires a combination of `clip-path`, SVG, and multiple HTML elements. It's not ideal for something seemingly straightforward.

---

## The Solution: `border-shape`

The new `border-shape` CSS property aims to simplify this by letting you define the precise contour of an element's border. Imagine being able to draw your border with a custom shape, just like you would in a design program!

Here's how it works:

* **Define Your Contour:** You can specify one or two basic shapes that will serve as the "strokes" of your border.
* **Filled Area:** If you provide two shapes, the space between them will be filled in, creating a solid border.
* **Content Clipping:** The inner shape you define also controls how the element's content is clipped, much like `overflow: hidden` but following your custom shape.

You can dive deeper into the technical details in the [draft specification](https://drafts.csswg.org/css-borders-4/#border-shape).

---

### Why a New Property?

You might wonder why we need `border-shape` when we have `clip-path` or `corner-shape`.

* **`clip-path` vs. `border-shape`:** While `clip-path` is useful for general clipping, it doesn't give you a true border that can be styled independently. `border-shape` is specifically for painting the border itself.
* **`corner-shape` vs. `border-shape`:** `corner-shape` excels at creating advanced, box-like shapes with custom corners (like squricles and diagonals), allowing you to fine tune them using all the traditional border properties (width, color, style for each side). `border-shape`, on the other hand, offers full artistic freedom for any shape, but with a more limited approach to border styling - a uniform stroke width and color, along with standard stroking styles (like joins, dashes, and caps). Each serves a distinct purpose, offering different levels of flexibility and control.

---

### How it Renders (Behind the Scenes)

While this is still being discussed and may evolve, here's the current thinking on how `border-shape` would render:

* An element can have **zero, one, or two `border-shape`s**.
* If an element has no `border-shape`, it renders its traditional box borders as normal.
* If you specify one `border-shape`, it acts as both the inner and outer contour.
* When a `border-shape` is present, the regular CSS `border` property (e.g., `border-width`, `border-color`) is **not rendered**, but it still **influences layout**. This means you can use `border-width` to control spacing even if your border is a custom shape, and also use it to affect the inner shape, by using the `padding-box` keyword when specifying that shape.
* The two `border-shape`s essentially define an **"inner" and "outer" boundary**.
    * The **inner contour** determines how `overflow` content is clipped.
    * The **outer contour** determines the extent to which the border is filled. The area between the inner and outer contoured is filed using CSS `fill-*` properties similar to how SVG elements are filled).
    * Both contours are **stroked** using CSS `stroke` properties (again, like in SVG). The stroke width does not affect layout in any way.
    * The **outer contour** alsos determine how `box-shadow`, `backdrop-filter` and `outline`s are rendered. Those properties follow the outer-contour. `outline` acts as an additional stroke on that contour, using the outline's color, style and width.

---

This new `border-shape` property promises to make creating complex, non-rectangular designs on the web much more straightforward and efficient.
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
