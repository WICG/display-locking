## `renderPriority` attribute.

_Note that issues referenced in this document can refer to either the
`renderPriority` attribute or `updateRendering()` JavaScript API. The issues
discussed apply to both._

### TL;DR
The `renderPriority` attribute is an HTML attribute that informs the User
Agent to keep the element's rendering state updated with a specified priority.

(Note that the `renderPriority` name is a
[placeholder name](https://github.com/WICG/display-locking/issues/200))

### Motivation
The web includes a number of features and heuristics (such as
content-visibility, containment and others) that allow the User Agent to skip
rendering work for elements and contents of elements. This is done with the
intent to allow other content, animations and interactions to remain smooth and
get as much CPU time as possible. However, there are situations where this
neglects the site intent of showing currently skipped content shortly. In other
words, if the website intends to show an element whose contents are currently
skipped, then skipping work may cause jank when the contents are ultimately
presented.

### Proposal: `renderPriority` element attribute
When present, this attribute informs the User Agent that it should keep the
element and its subtree updated according to the specified priority. The timing
and frequency of updates are kept in the User Agent's control to allow
flexibility in prioritizing this work.

The values that the attribute takes inform the User Agent of developer intent:
* `userBlocking` is the highest priority, and is meant to be used for updates
  that are blocking the user’s ability to interact with the page, such as
  rendering the core experience or responding to user input.

* `userVisible` is the second highest priority, and is meant to be used for
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
`renderPriority` on a visible, on screen, element would not have to do any
work since the rendering state of such an element is already kept up to date.
(Note that it is an [open question](https://github.com/WICG/display-locking/issues/202)
whether the behavior of visible elements with `renderPriority` should cause
asynchronous updates)

We're proposing this feature as an enhancement for the `content-visibility` CSS
property. For example, when `content-visibity: hidden` property is applied to
the element, it ensures that the element's subtree is not visible and that its
rendering state is not updated to save CPU time. Adding `renderPriority` on
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

Here the contents of the `#container` are not visible, and its rendering state
is not necessarily up-to-date since the contents of the element are
[skipped](https://www.w3.org/TR/css-contain-2/#skips-its-contents)

This is typically done to avoid rendering work in this subtree. However, when
the developer decides that the contents should now be visible, they remove
`content-visibility: hidden` style. This causes all of the rendering to be
updated in the subsequent frame. This work, in turn, can cause undue delay
and visible jank on the page.

The solution is to add the `renderPriority` attribute:

```html
<div id=container renderPriority=background style="content-visibility: hidden">
  <!-- some complicated subtree here -->
  ...
</div>
```

Here, the developer has added renderPriority=background, which means that they
would like the User Agent to keep the rendering state of `#container`'s content
to be kept up to date with a low priority.

Assuming that the User Agent completes this work (i.e. it is not otherwise busy
doing more important rendering work), then when the developer removes
`content-visibility` style, the contents are displayed without delay or jank.
This is a consequence of the fact that the rendering state should already be
up-to-date, completed cooperatively with the rest of the required work.


### Notes and Clarifications
* If the User Agent does not optimize rendering of elements, by skipping work,
  then the attribute has no effect.

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
  If such an update is desired, a `renderPriority` attribute should be set on
  such elements. Note that recursive `renderPriority` settings may be considered
  in the future.
