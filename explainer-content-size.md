# The `content-size` CSS property

This is an explainer for why the `content-size` property is needed. A draft of the spec for this property is [here](http://tabatkins.github.io/specs/css-content-size/). The corresponding spec issue is [here](https://github.com/w3c/csswg-drafts/issues/4229).

`content-size` represents the intrinsic layout sizing of a DOM subtree. Intrinsic sizing is an input to layout of ancestors, and is an important input to the layout constraint algorithms of the web. As one example, if an element has [`width: max-content`](https://drafts.csswg.org/css-sizing-3/#valdef-width-max-content) CSS on it, then it is sized to the [max-content inline size](https://drafts.csswg.org/css-sizing-3/#max-content-inline-size), which is the maximum intrinsic width of elements in the subtree. 
Currently, such intrinsic sizes are determined implicitly from other factors [2], and the other factors have additional undesirable layout side-effects.

This proposal exposes these intrinsic sizes explicitly. (In that sense, one motivation for the `content-size` property could be [explaining](https://extensiblewebmanifesto.org) these algorithms. One practical application could be its use in testing their behavior.

`contain:size` has an effect similar to `content-size`, in that it detaches constraints on the layout outside of the element from subtree elements; in place of a specified size for the subtree content's intrinsic sizing, it uses a intrinsic width and height of 0. In the [spec](https://drafts.csswg.org/css-contain/#containment-size), this is due to the sentence that says, "when calculating the size of the containing box, it must be treated as having no contents". `content-size` can also be thought of as extending `contain:size` to change to passing a non-zero intrinsic width or height.

## Use cases

(See also the [main explainer](https://github.com/WICG/display-locking/blob/master/README.md) for display-locking for more motivating examples.)

### Virtualization

Adding `contain: size layout; content-size: XXpx YYpx` to blocks of DOM that are known to be of offscreen or known-invisible  allows the web browser to avoid rendering cost for this offscreen content, while minimizing impact of the layout of on-screen content. This allows the web author to control [virtualization](https://github.com/chrishtr/rendering/blob/master/virtualization.md) of rendering costs, in order to improve web site performance.

### Async rendering lifecycle

Another motivating use-case for `content-size` is to use it in cases when the layout of a subtree is *not available*. The layout can be unavailable in situations such as:
* The subtree is not yet fully loaded from the network, or rendered into DOM by a custom element or framework
* The User Agent has temporarily skipped rendering for the subtree as an optimization

Note that these two cases can be seen as the same use case, if the concept of "rendering" is extended to include asynchronous, scheduled factors such as the network, a DOM rendering framework [1], a [custom element](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements), dynamic replaced elements such as embedded SVG documents, images and videos. In other words, the [rendering event loop](https://github.com/chrishtr/rendering/blob/master/rendering-event-loop.md) is only one source of potential asynchrony in an expanded concept of rendering.

In such cases, the developer may still wish for the element to participate in the intrinsic sizing aspects of layout, by providing placeholder sizing, because otherwise there may be an undesirable, jarring [shift of the layout](https://web.dev/layout-instability-api) of the page as the content renders.


## Working examples of `content-size`

[Here](https://wicg.github.io/display-locking/sample-code/contain-size-block-flow-examples.html) is a page demonstrating some simple block-flow layout use-cases.

[Here](https://wicg.github.io/display-locking/sample-code/contain-size-flexbox-examples.html) is a page demonstrating a basic flexbox use-case.

## Alternatives considered

Elements can already be sized by `width`, `min-width`, and various other sizing CSS properties. However, these properties fall short, because they all affect not just intrinsic sizing, but other layout inputs as well. The examples section gives several examples of this.


[1] For example, a div with `width: 200px` has a `200px` intrinsic width that is communicated upwards in layout algorithms in order to provide sizing constraints to ancestors. Another example is that an image with a `width=200px` attribute will have a `200px` intrinsic width; if it instead has a `src` attribute pointing at an image resource that has a `200px` image resolution width, the image will also have a `200px` intrinsic width once the image is loaded.

[2] Examples: React, Angular, Vue, ...
