# Smooth corners with `corner-shape`

## User need
Squircles, or other techniques of smoothing corners, are available as design featuers in multiple platforms for many years.
Some folks in the design community have been asking for this persistently, see https://www.figma.com/blog/desperately-seeking-squircles/ for example.
Being able to design beautiful websites, and also to match brand designs in print or in mobile apps.

## Existing Solutions
To design a web page with smooth corners today, the best solution is probably SVG, or `clip-path` without stroking.
The web is filled with SVG Squircle generators like [this one](https://observablehq.com/@daformat/draw-squircle-shapes-with-svg-javascript).
Though the existing squircle generators usually do a decent job, they are not easy to integrate into an existing design, and often require JS to be parametric.

## Proposed solution
The proposed solution is to treat this issue as "shaping a corner as a superellipse", that would work together with the existing `border-radius`.
A superellipse is a mathematical function, that can return any shape between a square (fully convex) and a notch (fully concave).
With `border-radius` controlling the size of the corner and `corner-shape` controlling its curvature, designers can have the full expression for shaping/smoothing corners,
as well as being able to animate between different shapes, by applying some interpolation to the superellipse's parameter.

For convenience, `corner-shape` will come with a few "stock" parameters - notch/scoop/bevel/round/squircle, with `round` being the default due to the existing behavior of `border-radius`.

This is documented in the [draft spec](https://drafts.csswg.org/css-borders-4/#corner-shaping), and an approximated demo (using canvas) can be seen [here](https://codepen.io/noamr/pen/pvzBYGZ?editors=1111).

## Alternatives considered

### Using the existing `shape` functions (`clip-path`, the upcoming `border-shape`)
Since basic shapes can provide modular means of expressing different shapes, why special-case corners?

One reason for that makes `corner-shape` compelling is its interaction with `border-radius`, and its quality as a progressive enhancement. If a browser does not support `corner-shape`, the usual rounded corners are used.
Since `border-radius` is alreay well established, augmenting it with shaping seems natural, and doesn't require going all the way to the world of basic shapes.

### Seperating `corner-size` from `border-radius`
A few [use cases](https://github.com/w3c/csswg-drafts/issues/10653#issuecomment-2628385487) came up for supporting a separate `corner-size` property, separate from `border-radius`. This would allow having "corners within corners",
e.g. where there is a big notch in the bottom-right corner, and the notch's corners themselves are rounded.

A few question arises with this:
* Should this be limited to corners? perhaps you'd want a notch from the bottom-center?
* These corners are likely going to be a bit bigger and more square. Perhaps they should interact with layout or grid from the start, like `shape-inside` of some form?
* Perhaps for these more complex shapes it's enough to use `shape()` or `polygon()`?
* Where do we draw the line?

In any case, adding some form of an inset-notch corner later on is not something that `corner-shape` needs to close the door on, and we can always find the name if and when we decide to tackle this use case.









