# Display Locking

This is a short proposal for Display Locking. For more details, see the
[explainer](https://github.com/WICG/display-locking/blob/master/explainer.md).

### Problem

Websites frequently rely on dynamic content to present information. This means
that DOM is often manipulated using script to present rich and dynamic content.
There are cases when this can cause slow rendering phase updates, including
script to update views, style updates, layout, and paint. This, in turn, causes
**jank or noticeable delay** when presenting content, because the rendering phase is
updated **synchronously** with user interactions and requestAnimationFrame script.

Developers are aware of these issues and build systems to avoid adding complex
DOM at once in order to prevent jank. As an example, ReactJS is [adding the
ability](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html)
to do async operations to help with this problem.

Creating more complex applications might also mean having a **larger number of things on a web page**,
making the rendering costs of having all of the content to actually be part of the DOM prohibitively expensive.

This drives some number of web authors to *virtualization* instead,
where they actually **refrain from putting things in the DOM**,
keeping content in memory in some non-DOM data structure,
with only the visible portion converted into DOM nodes and inserted into the document,
recycling them in and out as needed.

However, this will **cause problems to the web app** because a lot of user-agent features such as find-in-page,
accessibility, indexability, focus navigation, anchor links, etc.
**depend on having the content be in the DOM**.

### Proposal

This document proposes to augment user-agent APIs to help address the problem of
**slow rendering phase updates**, and having some control on **when to pay rendering costs**.
Specifically, we propose a concept of *display locking* an element.

If an element is display locked, it means that
**any DOM, style or layout updates to its subtree are not rendered immediately**.
Instead, when allowed, the user agent will process rendering updates *co-operatively*,
yielding periodically to allow script or user interactions to happen.
The web author can also commit the lock,
which triggers a synchronous update and render when necessary,
causing the element to get updated and rendered in the next frame.

The web author can also opt-in to allow display-locked elements to be
**committed by the user agent in cases like focus navigation, find-in-page match navigation**, etc.
so that they are properly rendered when needed in those cases.

### Example

```js
async function updateDom() {
  let element = document.createElement("div");

  // Construct the element
  element.appendChild(...);
  element.id = "...";

  // Acquire a lock, and allow it to be committed by the user agent when
  // necessary when focused, navigated to in find-in-page, anchor links, etc.
  // We also state that we don't want the user agent to automatically commit
  // triggered by a timeout by setting it to Infinity.
  await element.displayLock.acquire({ activatable: true, timeout: Infinity });

  // Append the element to the DOM.
  document.body.appendChild(element);

  // Now even if the element is attached to the tree we can do DOM updates,
  // change style, etc. to it but it won't affect rendering.
  element.style = "...";
  element.appendChild(...);

  // Calling update() will allow rendering to go update the style & layout
  // values through co-operative updates that may last multiple frames,
  // starting from the next frame.
  // Note that this will not cause the element to be painted/rendered.
  // Also, only elements in the locked element's subtree will be updated
  // co-operatively. If there are other things in the document outside of
  // the locked subtree that needed rendering update, they will be updated
  // in one go at the next frame, as usual.
  let updatePromise = element.displayLock.update();

  // After the co-operative calculations finish the promise that update()
  // returned will resolve.
  updatePromise.then(() => {
    // Calling commit() will cause the element to be rendered on the next frame.
    // commit() returns a promise that resolves when rendering finishes.
    // Note that we can call commit() without calling update() beforehand, but
    // that might cause a jank because we're doing the style, layout, paint in
    // one frame instead because we don't have the updated values.
    // However, even if we call update() beforehand, there are cases like flex
    // layout where the layout of the locked element affects its siblings,
    // possibly causing this to be slow.
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
- Performing **co-operative rendering work** using update(), which can always be
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

### Notes
- Display Locking is available in Chrome Canary behind a Display Locking flag.
  See about:flags.
