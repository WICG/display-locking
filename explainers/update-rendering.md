## `updateRendering()`

### TL;DR
updateRendering() is an asynchronous JavaScript function on Element which
requests that the rendering for the element be updated in preparation for
display.

### Motivation
The web includes a number of features and heuristics (such as
content-visibility, containment and others) that allow the user-agent to skip
rendering work for elements and contents of elements. This is done with the
intent to allow other content, animations and interactions to remain smooth and
get as much CPU time as possible. However, there are situations where this
neglects the site intent of showing currently skipped content shortly. In other
words, if the website intends to show an element whose contents are currently
skipped, then skipping work may cause jank when the contents are ultimately
presented.

### Proposal
The proposal is to add the following function, which lets the user-agent know
that the element’s contents should be updated in preparation for display:

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

### Notes
If the User Agent does not optimize rendering of elements, then the
function can return a resolved promise for every call. Note that a successful
resolution of the promise does not provide strong guarantees of performance:
bringing the element on screen may partially or fully invalidate the work that
has been prepared.  The User Agent should prepare the element and its contents
as much as it deems is necessary for presentation


