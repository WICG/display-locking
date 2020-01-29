# The `contain-contain-intrinsic-size` CSS property

`contain-intrinsic-size` represents the intrinsic layout sizing of a DOM
subtree, which is applied when `contain: size` is present. This feature extends
size containment by allowing the developer to specify an intrinsic size other
than 0x0, which is a common result of size containment. 

Intrinsic sizing is an input to layout of ancestors, and is an important input
to the layout constraint algorithms of the web. As one example, if an element
has [`width: max-content`](https://drafts.csswg.org/css-sizing-3/#valdef-width-max-content)
CSS on it, then it is sized to the [max-content inline size](https://drafts.csswg.org/css-sizing-3/#max-content-inline-size),
which is the maximum intrinsic width of elements in the subtree.  Currently, such
intrinsic sizes are determined implicitly from other factors [2], and the other
factors have additional undesirable layout side-effects.

Under size containment, the intrinsic size is determined without consideration
of element's children. This means that typically the intrinsic size will be 0x0,
with some exceptions such as `display: grid` which also considers its tracks
when determining the intrinsic size. 

The `contain-intrinsic-size` property extends size containment by allowing the
developer to specify the size directly. In that sense, one motivation for the
`contain-intrinsic-size` property could be
[explaining](https://extensiblewebmanifesto.org) these algorithms. One practical
application could be its use in testing their behavior.

## Use cases

(See also the [main
explainer](https://github.com/WICG/display-locking/blob/master/README.md) for
display-locking for more motivating examples.)

### Virtualization

Adding `contain: size layout; contain-intrinsic-size: XXpx YYpx` to blocks of DOM that
are known to be of offscreen or known-invisible  allows the web browser to avoid
rendering cost for this offscreen content, while minimizing impact of the layout
of on-screen content. This allows the web author to control
[virtualization](https://github.com/chrishtr/rendering/blob/master/virtualization.md)
of rendering costs, in order to improve web site performance.

### Async rendering lifecycle

Another motivating use-case for `contain-intrinsic-size` is to use it in cases when the
layout of a subtree is *not available*. The layout can be unavailable in
situations such as:
* The subtree is not yet fully loaded from the network, or rendered into DOM
* a custom element, framework or the User Agent has temporarily skipped rendering
  for the subtree as an optimization
* render-subtree property enforces size containment while not rendering the
  subtree.

Note that these three cases can be seen as the same use case, if the concept of
"rendering" is extended to include asynchronous, scheduled factors such as the
network, a DOM rendering framework [1], a [custom
element](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements),
dynamic replaced elements such as embedded SVG documents, images and videos. In
other words, the [rendering event
loop](https://github.com/chrishtr/rendering/blob/master/rendering-event-loop.md)
is only one source of potential asynchrony in an expanded concept of rendering.

In such cases, the developer may still wish for the element to participate in
the intrinsic sizing aspects of layout, by providing placeholder sizing, because
otherwise there may be an undesirable, jarring [shift of the
layout](https://web.dev/layout-instability-api) of the page as the content
renders.
