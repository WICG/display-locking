# Display locking

## Introduction

Display locking is a set of API changes that make it straightforward for developers and browsers to easily scale to large amounts of content and control when rendering [2] work happens. More concretely, the goals are:
* Avoid loading [1] and rendering work for content not visible to the user, without breaking user-agent features and supporting existing layout algorithms
* Support developer-controlled pre-loading, rendering and measurement of content without having to fully render it to the screen

The following use-cases motivate this work:
* Fast display of large HTML documents (examples: HTML one-page spec, other long documents)
* Deep links into pages with hidden content (example: mobile Wikipedia)
* Support scrollers with a large amount of content, without resorting to virtualization
* Pre-load and render non-visible content asynchronously to prepare it for future display (Example: improving latency to show content not currently displayed but predicted to be soon, but *without jank*. Think search as you type, tabbed UIs, hero element clicks.)
* Layout measurement (examples: responsive design or animation setup)

## Motivation & background

Web developers need ways to reduce loading and rendering time of web apps that have a lot of DOM. Two common techniques are to mark non-visible DOM as "invisible" [3], or to use virtualization [4]. Browser implementors also want to reduce loading and rendering time of web apps. Common techniques to do so include adding caching of rendering state [5], and avoiding rendering work [6] for content that is not visible.

These techniques can work in many cases but have some drawbacks and limitations. These include:
a. [3] and [4] usually means that such content is not available to [user-agent APIs](https://github.com/WICG/display-locking/blob/master/user-agent-apis.md). Also, content that is placed offscreen may or may not have rendering cost, which makes the heuristic unreliable.
b. Caching intermediate rendering state is [hard work](https://martinfowler.com/bliki/TwoHardThings.html), and often has performance limitations and cliffs that are not obvious to developers. Similarly, relying on the browser to avoid rendering for content that is clipped out or not visible is not reliable, as it's hard for the browser to efficiently detect what content is actually visible.

Previously adopted web APIs, in particular the [contain](https://developer.mozilla.org/en-US/docs/Web/CSS/contain) and [will-change](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change) CSS properties, add ways to specify forms of rendering [isolation](https://github.com/chrishtr/rendering/blob/master/isolation.md) or isolation hints, with the intention of them being a mechanism for the web developer to help the browser optimize rendering for the page.

While these forms of isolation help, they do not guarantee that isolated content does not need to be rendered at all. Ideally there would be a way for the developer to specify that specific parts of the DOM need not be rendered, and pair that with a guarantee that if they were rendered, it would not invalidate more than a small amount of style, layout or paint in the rest of the document.

## Summary

Three new features are proposed:

* A new `rendersubtree` attribute (early draft spec [here](https://chrishtr.github.io/html/output/#the-rendersubtree-attribute)). This controls whether DOM subtrees do or do not render, and is the mechanism by which rendering work can be avoided. User Agent features may modify this attribute, causing on-demand rendering, if desired. The developer may listen to this on-demand rendering via a MutationObserver and respond to it before rendering occurs. `rendersubtree`, when present, forces `style` and `layout` containment, plus `size` containment if invisible.

* A `content-size` attribute that specifies how much space a subtree should take up. This is intended to allocate a placeholder size for content marked as invisible by `rendersubtree` (since it already has `size` containment, as mentioned above).

* An `updateRendering` method on Element objects. This can be used to pre-render content within a subtree marked with `rendersubtree` as invisible to make it ready for display or measurement.

## Example usage

```
<div rendersubtree=invisible style="content-size: 200px 200px">...content...</div>
```
This div is not rendered, and there is no need for the browser to do any rendering lifecycle phases for the subtree of the div. Custom element upgrades and resources are not loaded. The div lays out as if it had a single 200px by 200px child, which serves as a placeholder in order to take up the approximate layout size of the div's subtree. This allows page layout to be approximately correct, and preserves layout overflow size for scrolling. The brrowser may *not* render the content, even with a User Agent feature.

```
<div rendersubtree=invisible-activatable  style="content-size: 200px 200px">...content</div>
```
Same as above, except that User Agent features may change the `rendersubtree` attribute to `visibile`.

```
<div rendersubtree=invisible-activatable  style="content-size: 200px 200px">...content</div>
```
This div renders, but still has `style` and `layout` containment. `content-size` has no effect because it doesn't have `size` containment.

```
<div id=target rendersubtree=invisible style="content-size: 200px 200px">...content...</div>
<script>
target.updateRendering().then(() => console.log(target.offsetTop)); // fast!
```
The div does not render to the screen, but the User Agent does work in the background to prepare to render it quickly in the future. This includes loading external resources referred to in the subtree, and custom element upgrades. It also may include running the rendering lifecycle steps for the subtree up to and including style, layout, paint and raster. When the returned promise resolves, reading layout or style-inducing properties on the div is expected to be fast.

## Alternatives considered

The `display:none` CSS property causes content subtrees not to render. However, there is no mechanism for User Agent features to cause these subtrees to render.

`visibility: hidden` causes subtrees to not paint, but they still need style and layout, as descendants may be `visibility: visible`. Second, there is no mechanism for User Agent features to cause subtrees to render.

`contain: strict` allows the browser to automatically detect subtrees that are definitely offscreen, and therefore that don't need to be rendered. However, `contain:strict` is not flexible enough to allow for responsive design layouts that grow elements to fit their content. Second, `contain:strict` may or may not result in rendering work, dependin on whether the browser detects the content is actually offscreen. Third, it does not support pre-rendering or User Agent features in cases when it is not actually rendered to the user in the current application view.

[1] Examples: fetching images off the network, custom element upgrade callbacks
[2] Meaning, the [rendering part](https://github.com/chrishtr/rendering/blob/master/rendering-event-loop.md) of the browser event loop.
[3] Examples: placing `display:none` CSS on DOM subtrees, or by placing content far offscreen via tricks like `margin-left: -10000px`
[4] In this context, virtualization means representing content outside of the DOM, and inserting it into the DOM only when visible. This is most commonly used for virtual or infinite scrollers.
[5] Examples: caching the computed style of DOM elements, the output of text / block layout, and display list output of paint.
[6] Examples: detecting elements that are clipped out by ancestors, or not visible in the viewport, and avoiding some or most rendering [lifecycle phases](https://github.com/WICG/display-locking/blob/master/lifecycle.md) for such content.
