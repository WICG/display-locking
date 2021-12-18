## `renderpriority` attribute.

_Note that issues referenced in this document can refer to either the
`renderpriority` attribute or `updateRendering()` JavaScript API. The issues
discussed apply to both._

### TL;DR
The `renderpriority` attribute is an HTML attribute that informs the User
Agent to keep the element's rendering state updated with a specified priority.

(Note that the `renderpriority` name is a
[placeholder name](https://github.com/WICG/display-locking/issues/200))

### Motivation
The web includes a number of features and heuristics, such as
content-visibility, containment and others, that allow the User Agent to skip
rendering work for elements and contents of elements when they are not visible to the
user. However if that content is subsequently brought on-screen, or its rendering state is
queried via a DOM API, there may be a significant delay to render the content.

An example use case is optimizing the speed of single-page-application navigations. If
the application can predict a likely user action, then it can prerender the next
view offscreen via `content-visibility: hidden` plus `renderpriority`. This will make the
single-page application navigations faster for the user.

### Proposal: `renderpriority` element attribute
When present, this attribute informs the User Agent that it should keep the
element and its subtree updated according to the specified priority. The timing
and frequency of updates are kept in the User Agent's control to allow
flexibility in prioritizing this work.

The values that the attribute takes inform the User Agent of developer intent:
* `user-blocking` is the highest priority, and is meant to be used for updates
  that are blocking the user’s ability to interact with the page, such as
  rendering the core experience or responding to user input.

* `user-visible` is the second highest priority, and is meant to be used for
  updates that visible to the user but not necessarily blocking user actions,
  such as rendering secondary parts of the page. This is the default priority.

* `background` is the lowest priority which permits updates, and is meant to be
  used for updates that are not time-critical, such as background updates for
  speculative layout.

* `never` is the value that skips rendering updates. Note that this value only
  has an effect if the rendering of the page would already have been skipped --
  See Notes and Clarification below.

* `auto` is the default priority, which allows the User Agent select an
  approriate priotization for rendering work.

Note that the attribute values follow [the postTask API](https://wicg.github.io/scheduling-apis/#sec-task-priorities)
example.


### Interaction with `content-visibility`
Since the User Agent typically keeps rendering state of subtrees up-to-date,
this feature would be a no-op in a majority of cases. For instance, having
`renderpriority` on a visible, on screen, element would not have to do any
work since the rendering state of such an element is already kept up to date.
(Note that it is an [open question](https://github.com/WICG/display-locking/issues/202)
whether the behavior of visible elements with `renderpriority` should cause
asynchronous updates)

We're proposing this feature as an enhancement for the `content-visibility` CSS
property. For example, when `content-visibity: hidden` property is applied to
the element, it ensures that the element's subtree is not visible and that its
rendering state is not updated to save CPU time. Adding `renderpriority` on
such an element, however, would cause the User Agent to continually process its
rendering with a given priority.

### Examples

Consider the following example.

```html
<div id=container style="content-visibility: hidden">
  <!-- some complicated subtree here -->
  ...
</div>
```

Here the contents of `#container` are not visible, and its rendering state
is not necessarily up-to-date since the contents of the element are
[skipped](https://www.w3.org/TR/css-contain-2/#skips-its-contents)

This is typically done to avoid rendering work in this subtree. However, when
the developer decides that the contents should now be visible, they remove
the `content-visibility: hidden` style. This causes all of the rendering to be
updated in the subsequent frame. This work, in turn, can cause undue delay
(for example, some experimental data from Facebook indicates up to a
[250ms](https://web.dev/content-visibility/#hiding-content-with-content-visibility:-hidden)
delay due to this work in practice).

The solution is to add the `renderpriority` attribute:

```html
<div id=container renderpriority=background style="content-visibility: hidden">
  <!-- some complicated subtree here -->
  ...
</div>
```

Here, the developer has added `renderpriority=background`, which means that they
would like the User Agent to keep the rendering state of `#container`'s content
to be kept up to date with a low priority.

Assuming that the User Agent completes this work (i.e. it is not otherwise busy
doing more important rendering work), then when the developer removes the
`content-visibility` style, the contents are displayed without undue delay.
This is a consequence of the fact that the rendering state should already be
up-to-date, completed cooperatively with the rest of the required work. Note
that there may still be rendering work to be done, since the act of removing
`content-visibility: hidden` may cause layout changes that need to be updated
(e.g. containment may be turned off).

### Performance Potential

One of the benefits of this feature comes from the fact that the work can be
delayed until later. This is, of course, already possible via properties such as
`display: none` and `content-visibility: hidden`. This means, on its own, this
benefit is not sufficient to justify adding a new attribute. 

However, consider how a page with `display: none` would typically work: 
1. The user navigates to the page, which loads, parses, and renders all of the
   DOM. 
2. DOM that is styled with `display: none` does not generate layout boxes, which
   means that rendering work like layout and painting is skipped.
3. Sometime later, a user action, for example, causes the page to remove the
   `display: none` style from some large subtree.
4. At this point, the user agent renders the full subtree synchronously.

This process is similar for `content-visibility: hidden` subtrees.

The synchronous rendering could cause jank, since the amount of work is
dependent on the DOM and that work must be done synchronously. The key
observation here is that some time may pass between initial load and when the
page would like to display hidden content. This naturally leads to a proposal:
let's use that time to render the hidden content incrementally without causing
jank. That way, when the page ultimately displays the content, the rendering
cost is small.

We need to be careful not to render all of `content-visibility: hidden` content
though since we don't know whether the page intends to show it to the user at
all. It is also unclear whether some content is more important, meaning we
should do more rendering work in such subtrees even if we risk a small amount of
jank.

This leads us to the `renderpriority` attribute. It allows for the two behaviors
we want: take the opportunity to use idle time for incremental rendering, and
to let the page dictate the importance of each hidden content piece. The only
heuristic here is how much work the User Agent will do, which should be left up
to the implementation.

Note that one can draw an equivalence to adding `content-visibility: auto` to
items in a large list. The initial visible content is rendered, but the content
off-screen remains unrendered. As the user continuously scrolls, more and more
content enters the viewport and renders. This accomplishes two things: the
initial load is significantly faster, since we only render on-screen items, and
the progressive rendering of new content does not jank -- since it is rendered
one item at a time. We have observed [substatial savings in
performance](web.dev/content-visibility) in using this approach. The
renderpriority attribute should allow similar performance gains for general
content, the availability of which is not tied to the viewport intersection as
is the case with `content-visibility: auto`.

### Notes and Clarifications
* If the User Agent does not optimize the rendering of an element, by skipping
  work, then the attribute has no effect on such an element. Note, however, that
  the value can still affect subtree elements that _are_ optimized.
  Specifically, the value dictates the maximum priority that would be used on
  the subtree element. In the example below, `.is_optimized` will be processed
  with a `background` priority, since that is the maximum priority established
  by its parent, regardless of whether the parent is itself optimized.

```html
<div class=not_optimized renderpriority="background">
  <div class=is_optimized renderpriority="user-visible"></div>
</div>
```

* Note that the UA must be careful about selecting the `auto` priority that is
  lower than `user-blocking`, since the priority cannot be overriden to be
  higher -- it can only be lowered. This means that if the overall default
  `auto` value is equivalent to `user-visible`, for example, then setting a
  value of `user-blocking` on any element would have no effect. For this reason,
  we recommend that the default behavior of `auto` is equivalent to
  `user-blocking` unless the user agent has a good reason to enforce a lower
  priority.

* Setting the attribute on an element whose rendering state is not updated due
  to `display: none`, ancestor style that prevents update (e.g. ancestor
  `content-visibility: hidden`), or similar styles has no effect. In other
  words, this attribute would not force rendering state to be updated on
  elements whose rendering work is required to be skipped. ([more
  details](https://github.com/WICG/display-locking/issues/199)). Note that this
  does _not_ include elements that themselves have `content-visibility: hidden`
  style, since their rendering is updated and the attribute would cause the
  subtree to be udpated as well.

* Setting the attribute on an element that contains descendants with
  `content-visibility: hidden`, `display: none`, or similar styles would not
  cause the contents of such elements to be updated. This is a consequence of
  the fact that a fully updated parent element has all its rendering work
  completed without updating such descendants. ([more details](https://github.com/WICG/display-locking/issues/196)).
  If such an update is desired, a `renderpriority` attribute should be set on
  such elements. Note that recursive `renderpriority` settings may be considered
  in the future.
