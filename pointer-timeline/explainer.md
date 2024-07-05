# Context
## Goal
Allow users to animate elements based on the position of the pointer, what is
sometimes referred to as "mouse parallax". Ideally we should provide a solution
for the common features (listed below) that has a coherent model and API with
the existing one for scroll-driven animations.

## Common features
There are common characteristics to pointer-driven animations:
- The timeline is linked to the position of the pointer relative to (usually),
  and contained within, either the target (animated) element, a container of 
  the target, or the entire viewport.
- Some effects are linked to a cartesian position of the pointer, others to
  the polar position.
- Timelines usually have a `[1, 1]` or `[-1, 1]` effect progress.
- Timelines are usually centered on the target's center, regardless of the
  element containing the timeline.
- Effects with delayed (transitioned) progress are common.
- Sometimes effects are linked to the velocity of the pointer.

## Solution
Add a new non-monotonic, pointer-based timeline, similar to the
scroll-based one. This timeline should be linked to the position
of the pointer over an element, or the entire viewport.
The progress of the timeline is linked to the position of the pointer from
the start edge to the end edge of the source element or the viewport.

## Prior art
[This previous proposal]((https://github.com/w3c/csswg-drafts/issues/6733)) by
[@bramus](https://github.com/bramus) with more elaborate details, that are also
relevant here, which relied on exposing a new pseudo-class for hovered element,
plus exposing new environment variables for the position of the pointer.

## JS implementations
Some libraries that allow this
effect: [Parallax.js](https://matthew.wagerfield.com/parallax/),
[Tilt.js](https://gijsroge.github.io/tilt.js/),
and [Atropos.js](https://atroposjs.com/).

## Live examples
TBD: some examples of sites using pointer-driven animations.

------------------------------
# Concept and terminology
## Timeline
A new timeline that's linked to the position of the pointer, relative to an
element/viewport - let's call it "source" - on a specific axis, either `x` or `y`.
Initially the timeline is defined by the source, starting at its start edge
and uniformly increasing to its end edge.

Like ViewTimeline, the PointerTimeline is linked to the un-transformed layout
box of the source (so that a timeline on the same element that's animated
with transforms doesn't change with the animation).

## Attachment range's centering (center shift)
It's common for pointer-driven animations to shift the center of
the attachment range to a specific point, so that common animations with an
effect progress of `[1, 1]` or `[-1, 1]` always reach `0` on that
specified point.
Usually that point is relative to the animated element - let's call it "target" -
rather than its source. Usually it's the target's center.

To achieve that, we also need to a way for authors to define that shift of the
timeline's center to a specified point, either on the source, or on the target.
The important thing to note here is that while the range is defined relative to
the source, the shift of the range's center may be defined relative to the target.

## Ranges
The timeline can then be expanded/contracted or stretched/squeezed using ranges.
These are also controlled in a similar fashion to ranges of ViewTimeline, but
with some adjustments. The available ranges are: `cover`, `contain`, `fill`,
and `fit` - building on top of known keywords of `object-fit` - though it
seems having `none` for a range feels awkward, so currently it's replaced
with `fit`.

All these ranges produce the same identical timeline if the range's center
is at the source's center, i.e. center _is not_ shifted.
However, if the range's center _is_ shifted, the ranges behave
differently and produce different timelines.

### Cover
This range acts similar to radial-gradient's `farthest-side` keyword.
The attachment range reaches either 0% or 100% at the farther edge of the
source, and then mirrored to the other side from range's center, so that
the attachment range is always covering the source.

### Contain
This range acts similar to radial-gradient's `closest-side` keyword.
The attachment range reaches either 0% or 100% at the closer edge of the
source, and then mirrored to the other side from range's center, so that
the attachment range is always contained within the source.

### Fill
This range acts similar to the `object-fit`'s `fill` keyword. The attachment
range reaches 0% at the start edge of the source, and 100% at the end edge,
so that it's stretched to fill the source from its center outwards.
In practice this is equal to automatically set `cover` to the farthest edge
and `contain` to the closest edge.

### Fit
This range acts similar to the `object-fit: none` keyword. The attachment
range reaches 0% at the start edge of the source, and 100% at the end edge,
and maintains this size even if its center is shifted, so that it's simply
displaced according to the center shift.

## Transitioned progress
It's also very common to see pointer-driven animations that have a "lerp" effect
or a time-based transition on the effect's progress, so that it slightly lags
behind the pointer position. This is usually done with a `transition` on the
animated properties or by an interpolation on every frame between the current
progress and the previous one.

This was suggested for scroll-driven animations in
[#7059](https://github.com/w3c/csswg-drafts/issues/7059), but was deferred to
level 2.
Since it's a common pattern for pointer-driven animations, it could be a good
opportunity to introduce it here.

## Velocity
Some effects are linked to the velocity of the pointer, rather than its position.
This is also common for scroll-driven animations, but was deferred to level 2.
Mouse events already expose the delta between previous and current position via
`movementX` and `movementY`, so it could be a chance to build on that and
introduce that as well.

## Polar Axes
Some effects are linked to the polar coordinates of the pointer, rather than its
cartesian ones. While it could be very useful to add a "distance" and an "angle"
axes to the proposal, they get very complex when trying to solve their progress
and ranges with the proposed model.
So it's better to defer them to further iterations, or to level 2 entirely.

------------------------------
# Proposal
## CSS
Add a new property `pointer-timeline` that takes a `dahsed-ident` as `name` and
a one of `x` or `y` as `axis`. 

For the anonymous timeline, a `pointer()` function that takes a source keyword
and an axis keyword should be added as value for `animation-timeline`.
Possible values for `source` are: `self` for same element, `nearest` for
nearest containing block , and `root` for viewport.

The `animation-range` should be extended to include the new range names:
`fill` and `fit`.

In order to allow the attachment range's center shift, a new property
`animation-range-center` should be added, that takes a `<length-percentage>`
value and an optional keyword `target`. Without the `target` keyword, the
value is relative to the source, otherwise it's relative to the target.
Inside the `animation-range` shorthand this value can either be introduced
following an `at`, or a `/`.

### Example:
```css
@keyframes move-x {
  from, to { translate: 50%; }
  50 { translate: 0; }
}

@keyframes move-y {
  from, to { translate: 0 50%; }
  50 { translate: 0 0; }
}

.container {
  pointer-timeline: --x x, --y y;
}

.figure {
  animation: move-x linear auto both, move-y linear auto both;
  animation-composition: replace, add;
  animation-timeline: --x, --y;

  /* alternatively with the anonymous timeline */
  animation-timeline: pointer(x nearest), pointer(y nearest);
  animation-range: cover at target 50%, cover at target 50%;
}
```

## Web Animations API
Expose a new interface `PointerTimeline`:

```webidl
enum PointerAxis {
  "block",
  "inline",
  "x",
  "y"
};

dictionary PointerTimelineOptions {
  Element? source;
  PointerAxis axis = "inline";
};

[Exposed=Window]
interface PointerTimeline : AnimationTimeline {
  constructor(optional PointerTimelineOptions options = {});
  readonly attribute Element? source;
  readonly attribute PointerAxis axis;
};
```

Add a new attribute `rangeCenter` to `Animation` of the following type:
```webidl
dictionary TimelineRangeCenter {
  CSSOMString? subject = "normal"; 
  CSSNumericValue offset;  
};

(TimelineRangeCenter or CSSNumericValue or CSSKeywordValue or DOMString) rangeCenter = "normal"
```

### Example:
```js
const source = document.querySelector('.container');
const target = document.querySelector('.figure');

const timelineX = new PointerTimeline({
  source,
  axis: 'x'
});
const timelineY = new PointerTimeline({
  source,
  axis: 'y'
});

const moveX = new KeyframeEffect(
  target,
  { translate: [0, '50%', 0] },
  { duration: 'auto', fill: 'both' }
);
const moveY = new KeyframeEffect(
  target,
  { translate: ['0 0', '0 50%', '0 0'] },
  { duration: 'auto', fill: 'both', composite: 'add' }
);

const animationX = new Animation(moveX, timelineX);
const animationY = new Animation(moveY, timelineY);

animationX.rangeCenter = { offset: CSS.percent(50), subject: 'target' };
animationY.rangeCenter = { offset: CSS.percent(50), subject: 'target' };

animationX.play();
animationY.play();
```