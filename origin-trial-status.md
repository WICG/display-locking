## Origin trial status of `rendersubtree` and `content-size`

In Chrome 79-81, an [Origin Trial](https://www.chromium.org/blink/origin-trials) is being run to allow testing of these features in the wild.

This page tracks the current status of what parts of the APIs are implemented as part of the trial, and any caveats to be aware of. Spec links are included, but it's better to read the [explainer](https://github.com/WICG/display-locking/blob/master/README.md) to understand how to take advantage of these APIs.

In addition to participating via the origin trial, these APIs are available on Chrome Canary by turning on experimental web platform APIs in Chrome. You can do that by navigating to `chrome://flags/#enable-experimental-web-platform-features`.

Example code is available [here](https://github.com/WICG/display-locking/tree/master/sample-code)

### `content-size`

This property is fully implemented as specified [here](http://tabatkins.github.io/specs/css-content-size/).

#### `rendersubtree`

A draft (out of date) spec is [here](https://github.com/whatwg/html/pull/4862).
The implementation currently supports the `invisible`, `skip-viewport-activation` and `skip-activation` keywords. Activation happens by default, but can be turned on with the two activation keywords mentioned. Before activation shows content, a `beforeactivate` event is fired.

We also support an experimental `updateRendering()` method on any DOM element that has `rendersubtree` specified. This method  asynchronously performs the work needed to render the subtree in the future, but does not actually have any visual effect. This can be used to warm up a subtree to be rendered, or to get it ready for reading style- or layout-inducing properties such as `getComputedStyle` or `offsetTop`. The method returns a promise which resolves when the rendering work is completed.
