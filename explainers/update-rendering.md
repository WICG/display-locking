## `updateRendering()` and `keepUpdated`

### TL;DR
updateRendering() is an asynchronous JavaScript function on Element which
requests that the rendering for the element be updated in preparation for
display.

The declarative counterpart is a `keepUpdated` attribute which lets the
User Agent keep the element's rendering updated whenever possible.

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

### Proposal: `updateRendering()` JavaScript function
The proposal is to add a feature that developers can use to inform the User
Agent of their intent. Specifically, the feature would inform the User Agent
that an element and its subtree should udpate its rendering state.

This can be done using the following function:

Signature: `Promise Element.updateRendering(Priority?)`

The function takes a Priority as an argument, with the following proposed values,
following [the postTask API](https://wicg.github.io/scheduling-apis/#sec-task-priorities)
example.

* `user-blocking` is the highest priority, and is meant to be used for updates
  that are blocking the user’s ability to interact with the page, such as
  rendering the core experience or responding to user input.

* `user-visible` is the second highest priority, and is meant to be used for
  updates that visible to the user but not necessarily blocking user actions,
  such as rendering secondary parts of the page. This is the default priority.

* `background` is the lowest priority, and is meant to be used for updates that
  are not time-critical, such as background updates for speculative layout.

The function returns a promise which resolves when the contents of the element
are ready to be displayed to screen.  Note that repeated calls to
updateRendering() returns new promises, with the ultimate priority of the
request being the highest requested priority.

### Proposal: `keepUpdated` element attribute
The declarative version of `updateRendering()` is an attribute called `keepUpdated`.

When present, this attribute informs the User Agent that it should keep the
element and its subtree updated whenever possible. The timing and frequency of
updates are kept in the User Agent's control to allow flexibility in
prioritizing this work.

### Interaction with `content-visibility`
Since the User Agent typically keeps rendering state of subtrees up-to-date,
this feature would be a no-op in a majority of cases. For instance, calling
`updateRendering()` on a visible, on screen, element would return a resolved
promise since the rendering state of such elements is already kept up to date.

We're proposing this feature as an enhancement for the `content-visibility` CSS
property. For example, when `content-visibity: hidden` property is applied to
the element, it ensures that the element's subtree is not visible and that its
rendering state is not updated to save CPU time. Calling `updateRendering()` on
such an element, however, would cause it to process its rendering with a given
priority, resolving the promise when the updates are complete.

### Notes
For all intents and purposes, the promise returned by `updateRendering()` is
resolved when the rendering state of the element is fully updated. This means
it is updated to the same degree as it would if the User Agent was always
keeping this element's subtree updated.

If the User Agent does not optimize rendering of elements, then the
function can return a resolved promise for every call. Note that a successful
resolution of the promise does not provide strong guarantees of performance:
bringing the element on screen may partially or fully invalidate the work that
has been prepared.  The User Agent should prepare the element and its contents
as much as it deems is necessary for presentation

Also note that calling the function on an element whose rendering state is not
updated due to `display: none` or ancestor style that prevents update (e.g.
ancestor `content-visibility: hidden`) would also return a resolved promise
since the rendering state of the element and its subtree is indeed fully
updated, respecting its ancestor styles. ([more details](https://github.com/WICG/display-locking/issues/199))

Likewise, calling `updateRendering()` on an element that contains descendants
with `content-visibility: hidden` or `display: none` would return a promise
that resolves _without_ updating such descdants. This is, again, a consequence
of being fully updated: the element itself is as updated as it would be without
the `content-visibility` applying to itself. ([more details](https://github.com/WICG/display-locking/issues/196))

