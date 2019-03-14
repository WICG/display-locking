# Display Locking

This is a short proposal for Display Locking. For more details, see the
[explainer](https://github.com/WICG/display-locking/blob/master/explainer.md).

### Problem

Websites frequently rely on dynamic content to present information. This means
that DOM is often manipulated using script to present rich and dynamic content.
There are cases when this can cause slow rendering phase updates, including
script to update views, style updates, layout, and paint. This, in turn, causes
**jank or noticeable delay** when presenting content, because rendering phase is
updated **synchronously** with user interactions and requestAnimationFrame script.

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
slow rendering phase updates.
Specifically, we propose a concept of *display locking* an element.

If an element is display locked, it means that
**any DOM, style or layout updates to its subtree are not rendered immediately**.
Instead, when allowed, the user agent will process rendering updates *co-operatively*,
yielding periodically to allow script or user interactions to happen.
The web author can also commit/unlock the lock,
which triggers a synchronous update and render when necessary,
causing the element to get updated and rendered in the next frame.

The visual content of an element's subtree which is locked for display does not
change. Specifically, if the element already existed in the DOM, then the
content that was present at the time the lock was acquired remains.

If the element was locked before being inserted into the DOM, then it is
effectively inserted in a hidden state, which means the user agent does not
display any content for this element or its subtree until the lock gets committed.

The web author can also opt-in to allow display-locked elements to be **committed by the user agent in cases like focus navigation,
find-in-page match navigation**, etc.
so that they are properly rendered when needed in those cases.

### Example

```js
async function updateDom() {
  let element = document.createElement("div");

  // Construct the element
  element.appendChild(...);
  element.id = "...";

  // Acquire a lock, and allow it to be unlocked by the user agent when
  // necessary.
  await element.displayLock.acquire({ activatable: true });

  // Append the element to the DOM.
  document.body.appendChild(element);

  // Now even if the element is attached to the tree we can do DOM updates,
  // change style, etc. to it but it won't affect rendering.
  element.style = "...";
  element.appendChild(...);

  // Calling update() will allow rendering to go update the style & layout
  // values through co-operative updates that may last multiple frames.
  // Note that this will not cause the element to be painted/rendered.
  // After the calculations finish the promise it returns will resolve.
  let updatePromise = element.displayLock.update();
  updatePromise.then(() => {
    // Calling commit() will cause the element to be rendered on the next frame.
    // Note that we can call commit() without calling update() beforehand, but
    // that might cause a jank because we're doing the style, layout, paint in
    // one frame instead because we don't have the updated values.
    element.displayLock.commit();
  });
}

```

In cases where DOM rendering would cause jank, this instead allows the
user-agent to **split up the work across several frames**, yielding for script and
user interactions. When the updates are finished processing,
we can commit and the result is displayed without jank.

Note that display locking allows several patterns, which serve as motivating
examples:

- **Splitting up the DOM construction work across several frames**, while storing
  intermediate results in the DOM itself under a locked element. These
  intermediate results are not displayed until the lock is committed.
- **Postponing the rendering work indefinitely** by controlling when the lock is
  committed.
- Performing co-operative rendering work using update(), which can always be
  forced to be synchronous using a commit() call enabling an idle-until-urgent
  pattern.
- **Measuring layout without jank** or display by calling update() on a locked
  element, then reading off relevant layout values and either removing the
  element or committing it for display depending on the desired behavior.

### Further reading

- [Detailed explainer](https://github.com/WICG/display-locking/blob/master/explainer.md)
- [Sample code](https://github.com/WICG/display-locking/blob/master/sample-code)
  (work in progress)
- [Layout transitions](http://tabatkins.github.io/specs/layout-transitions/) - a
  similar proposal from 2014, with motivating examples that are applicable to
  display locking.

### Revision log

- 2018-07-24: Initial version.
- 2018-10-17: Updated the explainer to reflect new api surface. Closes #24.
- 2018-12-21: Updated the explainer to acquire/update/commit API.
- 2019-02-01: Updated to displayLock (previously getDisplayLock()), added updateAndCommit().
- 2019-03-14: Update readme with "activatable"
