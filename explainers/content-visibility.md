## content-visibility

`content-visibility` is a CSS property designed to allow developers and browsers
to easily scale to large amount of content and control when rendering
[\[1\]](#foot-notes) work happens. More concretely, the goals is to avoid
rendering and layout measurement work for content not visible to the user.

The following use-cases motivate this work:

1. Fast display of large HTML documents (examples: HTML one-page spec; other long
   documents)
2. Scrollers with a large amount of content, without resorting to virtualization
   (examples: Facebook and Twitter feeds, CodeMirror documents)
3. Measuring layout for content not visible on screen
4. Optimizing single-page-app transition performance

## Summary

In the below, "invisible to rendering/hit testing" means not drawn or returned
from any hit-testing algorithms in the same sense as `visibility: hidden`, where
the content still conceptually has layout sizes, but the user cannot see or
interact with it.

Also, "visible to UA algorithms" means that find-in-page, link navigation, etc
can find the element.

`content-visibility: visible` - default state, subtree is rendered.

`content-visibility: auto` - avoid rendering cost when offscreen
* Use cases: (1), (3), (5)
* Applies `contain: style layout`, plus `contain: size` when invisible
* Invisible to rendering/hit testing, except when subtree intersects viewport
* Visible to UA algorithms

`content-visibility: hidden` - hide content, but preserve cached state and still
   support style/layout measurement APIs
* Use cases: (4), (5)
* Applies `contain: style layout size`
* Invisible to rendering/hit testing
* Not visible to UA algorithms

## Motivation & background

On the one hand, faster web page loads and interactions directly improve the
user experience of the web. On the other hand, web sites each year grow larger
and more complex than the last, in part because they support more and more use
cases, and contain more information, and the most common UI pattern for the
web is scrolling. This leads to pages with a lot of non-visible (offscreen or
hidden) DOM, and since the DOM presently renders atomically, it inherently
takes more and more time to render on the same machine.

For these reasons, web developers need ways to reduce loading and rendering time
of web apps that have a lot of non-visible DOM. Two common techniques are to mark
non-visible DOM as "invisible" [\[2\]](#foot-notes), or to use virtualization
[\[3\]](#foot-notes). Browser implementors also want to reduce loading and
rendering time of web apps. Common techniques to do so include adding caching of
rendering state [\[4\]](#foot-notes), and avoiding rendering work
[\[5\]](#foot-notes) for content that is not visible.

These techniques can work in many cases but have drawbacks and limitations:

a. [\[2\]](#foot-notes) and [\[3\]](#foot-notes) usually means that such content
   is not available to user-agent features, such as find-in-page functionality.
   Also, content that is merely placed offscreen may or may not have rendering
   cost (it depends on browser heuristics), which makes the technique
   unreliable.
  
b. Caching intermediate rendering state is [hard
   work](https://martinfowler.com/bliki/TwoHardThings.html), and often has
   performance limitations and cliffs that are not obvious to developers.
   Similarly, relying on the browser to avoid rendering for content that is clipped
   out or not visible is sometimes not reliable, as it's hard for the browser to
   efficiently detect what content is not visible and does not affect visible
   content in any way.

Previously adopted web APIs, in particular the
[contain](https://developer.mozilla.org/en-US/docs/Web/CSS/contain) and
[will-change](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change) CSS
properties, add ways to specify forms of rendering
[isolation](https://github.com/chrishtr/rendering/blob/master/isolation.md) or
isolation hints, with the intention of them being a mechanism for the web
developer to help the browser optimize rendering for the page.

While these forms of isolation help, they do not guarantee that isolated content
does not need to be rendered at all. Ideally there would be a way for the
developer to specify that specific parts of the DOM need not be rendered, and
pair that with a guarantee that when later rendered, it would not invalidate
more than a small amount of style, layout or paint in the rest of the document.

## Description of proposal

A new `content-visibility` CSS property is proposed. This property controls
whether DOM subtrees affected by the property are invisible to painting/hit
testing. This is the mechanism by which rendering work can be avoided. Some
values of `content-visibility` allow the user-agent to automatically manage
whether subtrees affected are rendered or not. Other values give the developer
complete control of subtree rendering. The possible values are the following:

* `content-visibility: visible`: this is the default state, in which this
  feature does not affect anything consequential.
* `content-visibility: auto`: this configuration allows the user-agent to
  automatically manage whether content is invisible to rendering/hit testing or not.
* `content-visibility: hidden`: this configuration gives the
  developer complete control of when the subtree is rendered. Neither the
  user-agent nor its features should need to process or render the subtree.

It is also worth noting that when the element is not rendered, then
`contain: layout style paint size;` is added to its style to ensure that the
subtree content does not affect elements outside of the subtree.
Furthermore, when the element is rendered in the `content-visibility: auto`
configuration (i.e. the user-agent decides to render the element), then
`contain: layout style paint;` applies to the element.

## Example usage

```html
<style>
.locked {
  content-visibility: auto;
  contain-intrinsic-size: 100px 200px;
}
</style>

<div class=locked>
  ... some content goes here ...
</div>
```

The `.locked` element's `content-visibility` configuration lets the user-agent
manage rendering the subtree of the element. Specifically when this element is
near the viewport, the user-agent will begin rendering the element. When the
element moves away from the viewport, it will stop being rendered.

Recall that when not rendered, the property also applies size containment to the
element. This means that when not rendered, the element will use the specified
`contain-intrinsic-size`, making the element layout as if it had a single block
child with 100px width and 200px height. This ensures that the element still
occupies space when not rendered. At the same time, it lets the element size to
its true contents when the subtree is rendered (since size containment no longer
applies), thus removing the concern that estimates like 100x200 are sometimes
inaccurate (which would otherwise result in displaying incorrect layout for
on-screen content).

One intended use-case for this configuration is to make it easy for developers to
avoid rendering work for off-screen content.

A second use-case is to support simple scroll virtualization.

```html
<style>
.locked {
  content-visibility: hidden;
  contain-intrinsic-size: 100px 200px;
}
</style>

<div class=locked>
  ... some content goes here ...
</div>
```

In this case, the rendering of the subtree is managed by the developer only.
This means that if script does not modify the value, the element's subtree will
remain unrendered, and it will use the `contain-intrinsic-size` input when
deciding how to size the element. However, the developer can still call methods such
as `getBoundingClientRect()` to query and measure layout for the invisible content.

One intended use-case for this mode are measuring layout geomery for content not displayed.

A second use-case is preserving rendering state for [single-page app](https://en.wikipedia.org/wiki/Single-page_application)
content that is not currently visible to the user, but may be displayed again
soon via user interaction.

## Alternatives considered

The `display: none` CSS property causes content subtrees not to render. However,
there is no mechanism for user-agent features to cause these subtrees to render.
Additionally, the cost of hiding and showing content cannot be eliminated since
`display: none` does not preserve the layout state of the subtree.

`visibility: hidden` causes subtrees to not paint, but they still need style and
layout, as the subtree takes up layout space and descendants may be `visibility:
visible`. (It's also possible for descendants to override visibility, creating
another complication.) Second, there is no mechanism for user-agent features to cause
subtrees to render. Note that with sufficient containment and intersection
observer, the functionality provided by `content-visibility` may be mimicked.
This relies on more browser heuristics to ensure contained invisible content is
cheap -- `content-visibility` is a stronger signal to the user-agent that work
should be skipped.

Similar to `visibility: hidden`, `contain: strict` allows the browser to
automatically detect subtrees that are definitely offscreen, and therefore that
don't need to be rendered. However, `contain: strict` is not flexible enough to
allow for responsive design layouts that grow elements to fit their content. To
work around this, content could be marked as `contain: strict` when offscreen
and then some other value when on-screen (this is similar to `content-visibility`).
Second, `contain: strict` may or may not result in rendering work, depending on
whether the browser detects the content is actually offscreen. Third, it does
not support user-agent features in cases when it is not actually rendered to the
user in the current application view.

<a name="foot-notes"></a>

[1]: Meaning, the [rendering
part](https://github.com/chrishtr/rendering/blob/master/rendering-event-loop.md)
of the browser event loop.

[2]: Examples: placing `display:none` CSS on DOM subtrees, or by placing content
far offscreen via tricks like `margin-left: -10000px`

[3]: In this context, virtualization means representing content outside of the
DOM, and inserting it into the DOM only when visible. This is most commonly used
for virtual or infinite scrollers.

[4]: Examples: caching the computed style of DOM elements, the output of text /
block layout, and display list output of paint.

[5]: Examples: detecting elements that are clipped out by ancestors, or not
visible in the viewport, and avoiding some or most rendering for such content.


