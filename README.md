# Display locking

## Introduction

Display locking is a set of API changes that make it straightforward for developers and browsers to easily scale to large amounts of content and control when rendering [\[1\]](#foot-notes) work happens. More concretely, the goals are:
* Avoid loading [\[2\]](#foot-notes) and rendering work for content not visible to the user, without breaking [user-agent features](https://github.com/WICG/display-locking/blob/master/user-agent-features.md) or any layout algorithms (e.g. responsive design, flexbox, grid)
* Support developer-controlled pre-loading, pre-rendering and measurement of content without having to fully render it to the screen

The following use-cases motivate this work:
* Fast display of large HTML documents (examples: HTML one-page spec, other long documents)
* Deep links into pages with hidden content (example: mobile Wikipedia)
* Scrollers with a large amount of content, without resorting to virtualization (examples: twitter feed, codemirror documents)
* Pre-loading and rendering of non-visible content asynchronously to prepare it for future display (Example: improving latency to show content not currently displayed but predicted to be soon, but *without jank*. Think search as you type, tabbed UIs, hero element clicks.)
* Layout measurement (examples: responsive design or animation setup)

## Motivation & background

Faster web page loads and interactions directly improve the user experience of the web. On the other hand, web sites each year are larger and more complex than the last, in part because they support more and more use cases, and contain more information. This leads to pages with a lot of DOM, and since the DOM presently renders atomically, it inherently takes more and more time to render on the same machine.

For these reasons, web developers need ways to reduce loading and rendering time of web apps that have a lot of DOM. Two common techniques are to mark non-visible DOM as "invisible" [\[3\]](#foot-notes), or to use virtualization [\[4\]](#foot-notes). Browser implementors also want to reduce loading and rendering time of web apps. Common techniques to do so include adding caching of rendering state [\[5\]](#foot-notes), and avoiding rendering work [\[6\]](#foot-notes) for content that is not visible.

These techniques can work in many cases but have some drawbacks and limitations. These include:
a. [\[3\]](#foot-notes) and [\[4\]](#foot-notes) usually means that such content is not available to user-agent features. Also, content that is merely placed offscreen may or may not have rendering cost (it depends on browser heuristics), which makes the technique unreliable.
b. Caching intermediate rendering state is [hard work](https://martinfowler.com/bliki/TwoHardThings.html), and often has performance limitations and cliffs that are not obvious to developers. Similarly, relying on the browser to avoid rendering for content that is clipped out or not visible is sometimes not reliable, as it's hard for the browser to efficiently detect what content is actually visible.

Previously adopted web APIs, in particular the [contain](https://developer.mozilla.org/en-US/docs/Web/CSS/contain) and [will-change](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change) CSS properties, add ways to specify forms of rendering [isolation](https://github.com/chrishtr/rendering/blob/master/isolation.md) or isolation hints, with the intention of them being a mechanism for the web developer to help the browser optimize rendering for the page.

While these forms of isolation help, they do not guarantee that isolated content does not need to be rendered at all. Ideally there would be a way for the developer to specify that specific parts of the DOM need not be rendered, and pair that with a guarantee that when later rendered, it would not invalidate more than a small amount of style, layout or paint in the rest of the document.

## Summary

Three new features are proposed:

* A new `rendersubtree` attribute (early draft spec [here](https://chrishtr.github.io/html/output/#the-rendersubtree-attribute)). This controls whether DOM subtrees do or do not render, and is the mechanism by which rendering work can be avoided. User-agent features may modify this attribute, causing on-demand rendering, if desired. The developer may listen to this on-demand rendering via a MutationObserver and respond to it before rendering occurs. `rendersubtree`, when present, forces `style` and `layout` containment, plus `size` containment if invisible. This ensures minimal invalidation of the rest of the document when rendering occurs.

* A `content-size` attribute (early draft spec [here](http://tabatkins.github.io/specs/css-content-size/) that specifies how much space a subtree should take up. This is intended to allocate a placeholder size for content marked as invisible by `rendersubtree` (since it already has `size` containment, as mentioned above).

* An `updateRendering` method on Element objects. This can be used to pre-render content within a subtree marked with `rendersubtree` as invisible to make it ready for display or measurement.

## Example usage

```html
<div id=target rendersubtree="invisible skip-activation" style="content-size: 200px 200px">...content...</div>
<script>
target.setAttribute('rendersubtree', ''); // makes #target render
</script>
```
This div's subtree is not rendered (but the div itself is; this allows the div to show fallback or "loading..." affordances), and there is no need for the browser to do any rendering lifecycle phases for the subtree of the div. The div lays out as if it had a single 200px by 200px child, which serves as a placeholder in order to take up the approximate layout size of the div's subtree. This allows page layout to be approximately correct, and preserves layout overflow size for scrolling. The browser may *not* render the content, even via a user-agent feature.

```html
<div rendersubtree="invisible skip-viewport-activation"  style="content-size: 200px 200px">...content</div>
```
Same as above, except that user-agent features may change the `rendersubtree` attribute to the empty string, causing the div's subtree to get rendered. More on this on ["Element activation by the user agent"](https://github.com/rakina/display-locking#element-activation-by-the-user-agent)

```html
<div id=target rendersubtree="invisible" style="content-size: 200px 200px">...content...</div>
<script>
target.setAttribute('rendersubtree', ''); // makes #target render
</script>
```
Same as above, except that user-agent will also activate when content enters vthe visible viewport.

```html
<div id=target rendersubtree="invisible holdupgrades holdloads" style="content-size: 200px 200px">...content...</div>
```
Same as just having the `invisible` value, but custom element upgrades are not performed and resources are not loaded. Note that these values can also be used without the `invisible` value also.

```html
<div rendersubtree style="content-size: 200px 200px">...content</div>
```
This div and its subtree render, but still has `style` and `layout` containment. `content-size` has no effect because it doesn't have `size` containment. This existence of `rendersubtree` is valuable beause when the div later becomes invisible, invalidations of rendering state are minimized.

```html
<div id=target rendersubtree="invisible" style="content-size: 200px 200px">...content...</div>
<script>
target.updateRendering().then(() => console.log(target.firstElementChild.offsetTop)); // fast!
</script>
```
The div's subtree does not render to the screen, but when updateRendering is called, the browser does work in the background to prepare to render it quickly in the future. This includes loading external resources referred to in the subtree and custom element upgrades. It also may include running the rendering lifecycle steps for the subtree up to and including style, layout, paint and raster. When the returned promise resolves, reading layout or style-inducing properties on the subtree is expected to be fast. Changing the `rendersubtree` value to `visible` should also be fast.

## Element activation by the user agent

When an element is not rendered because it's a part of a not-rendered subtree caused by `rendersubtree`, there are some actions in the page that might require the element (and its ancestors) to be rendered to work properly. If all of the element's ancestors' `rendersubtree` attributes contain `activatable` (or not set), then the user agent can *activate* the element. This will change all of the non-null `rendersubtree` value of all of its ancestors to the empty string - causing the element and its ancestors to get rendered.

```html
<div id="focusable" rendersubtree="invisible activatable" tabindex=0>Focus me!</div>
<script>
 focusable.focus(); // Will cause the element to render, and will change the rendersubtree value to "" (empty string)
</script>
```

Note that if any of the element's ancestor is not activatable (the `rendersubtree` is not null but doesn't contain `activatable`) then the element is not activatable.

Actions that will trigger activation to an element and all of its ancestors, are listed below:

- `focus()` is called on the element
- `scrollIntoView()` is called on the element
- Tab order navigation lands on the element
- Anchor link navigation navigates to the element
- Find-in-page active match (just the main match, not all find-in-page matches) navigation goes to the element
- Selections includes part of the element
- Accessibility focus, press, selections, etc.

Actions which cause the element to intersect the visibile viewport will also cause activation if neither `skip-viewport-activation` nor `skip-activation` is specified.

## Alternatives considered

The `display:none` CSS property causes content subtrees not to render. However, there is no mechanism for User Agent features to cause these subtrees to render.

`visibility: hidden` causes subtrees to not paint, but they still need style and layout, as the subtree takes up layout space and descendants may be `visibility: visible`. Second, there is no mechanism for user-agent features to cause subtrees to render.

`contain: strict` allows the browser to automatically detect subtrees that are definitely offscreen, and therefore that don't need to be rendered. However, `contain:strict` is not flexible enough to allow for responsive design layouts that grow elements to fit their content. (To work around this, content could be marked as `contain:strict` when offscreen and then some other value when on-screen (this is similar to `rendersubtree`).) Second, `contain:strict` may or may not result in rendering work, depending on whether the browser detects the content is actually offscreen. Third, it does not support pre-rendering or user-agent features in cases when it is not actually rendered to the user in the current application view.

<a name="foot-notes"></a>

[1]: Meaning, the [rendering part](https://github.com/chrishtr/rendering/blob/master/rendering-event-loop.md) of the browser event loop.

[2]: Examples: fetching images off the network, custom element upgrade callbacks

[3]: Examples: placing `display:none` CSS on DOM subtrees, or by placing content far offscreen via tricks like `margin-left: -10000px`

[4]: In this context, virtualization means representing content outside of the DOM, and inserting it into the DOM only when visible. This is most commonly used for virtual or infinite scrollers.

[5]: Examples: caching the computed style of DOM elements, the output of text / block layout, and display list output of paint.

[6]: Examples: detecting elements that are clipped out by ancestors, or not visible in the viewport, and avoiding some or most rendering [lifecycle phases](https://github.com/WICG/display-locking/blob/master/lifecycle.md) for such content.
