# Display Locking

This is a short proposal for Display Locking. For more details, see the
[explainer](https://github.com/chrishtr/display-locking/blob/master/explainer.md).

### Problem

Websites frequently rely on dynamic content to present information. This means
that DOM is often manipulated using script to present rich and dynamic content.
There are cases when this can cause slow rendering phase updates, including
script to update views, style updates, layout, and paint. This, in turn, causes
jank or noticeable delay when presenting content, because rendering phase is
updated synchronously with user interactions and requestAnimationFrame script.

The following are a few of the common patterns that can cause jank:
- Resizing multi-pane UI with complex layout within each pane (e.g. IDEs)
- Adding widgets with complex DOM.
- Dynamically displaying content based on the current value of text input.
- Measuring layout of otherwise hidden DOM with intent of sizing containers.

Developers are aware of these issues and build systems to avoid adding complex
DOM at once in order to prevent jank. As an example, ReactJS is [adding the
ability](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html)
to do async operations to help with this problem.

### Proposal

This document proposes to augment user-agent APIs to help address the problem of
slow rendering phase updates. Specifically, we propose a concept of display
locking an element.

If an element is locked for display, it means that any DOM updates to its
subtree are not rendered immediately. Instead, when the lock is committed, the
user-agent processed rendering updates co-operatively, yielding periodically to
allow script or user interactions to happen.

The visual content of an element's subtree which is locked for display does not
change. Specifically, if the element already existed in the DOM, then the
content that was present at the time the lock was acquired remains visibile
until the lock is committed and the associated promise is resolved. Note that in
this case, the element itself still responds and updates its own visual state
(e.g. border); it is only the element's subtree that is locked for display.

If the element was locked before being inserted into the DOM, then it is
effectively inserted in a hidden state, which means the user agent does not
display any content for this element or its subtree until the work is completed.

### Example

```js
function updateDom() {
  let element = document.getElementById("container");

  // Acquire a lock, and modify the element's subtree in the callback.
  element.acquireDisplayLock((context) => {
    // Modify element's subtree. The DOM is modified immediately, same as
    // without display locking. However, rendering of the element is delayed.
    element.innerHTML = ...;
  });

  // In this example, after the callback runs the lock is automatically
  // committed. This means the user-agent starts to co-operatively process steps
  // required to render the modified subtree, including style, layout, and
  // painting. acquireDisplayLock() returns a promise which is resolved when
  // this work is completed.
}

```

In cases where DOM manipulations would cause jank, this instead allows the
user-agent to split up the work across several frames, yielding for script and
user interactions. When the updates are finished processing, the result is
displayed without jank.

Note that more advanced patterns are possible using the provided display lock
context in the callback. In particular, the context allows the following, which
are discussed in more details in the explainer linked below:

- Splitting up the script work across multiple frames using context.schedule()
- Postponing the co-operative work indefinitely using context.suspend()
- Subsequently resuming the postponed work using a handle object returned by
  context.suspend()
- Forcing the update to become synchronous to support "idle until urgent"
  pattern using a handle returned by context.getSynchronousHandle()

### Further reading

- [Detailed explainer](https://github.com/chrishtr/display-locking/blob/master/explainer.md)
- [Sample
  code](https://github.com/chrishtr/display-locking/blob/master/sample-code)
  (work in progress)
- [Layout transitions](http://tabatkins.github.io/specs/layout-transitions/) - a
  similar proposal from 2014, with motivating examples that are applicable to
  display locking.

### Revision log

- 2018-07-24: Initial version.
- 2018-10-17: Updated the explainer to reflect new api surface. Closes #24.
