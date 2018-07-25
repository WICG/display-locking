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

The visual content of an element which is locked for display is fixed to
whatever content was present at the time the lock was acquired, and remains this
way until the lock is released and all associated promises are resolved.

### Example

```js
async function updateDom() {
  let element = document.getElementById("container");

  // Get and acquire a lock, causing the element to be locked.
  let lock = element.getDisplayLock();
  await lock.acquire();

  // Modify element's subtree. The DOM is modified immediately, same as without
  // display locking. However, rendering of the element is delayed.
  element.innerHTML = ...;

  // Commit the lock. This returns a promise. The user-agent starts to
  // co-operatively process steps required to render the modified subtree,
  // including style, layout, and painting. When the user-agent is done
  // with these steps, the promise is resolved.
  lock.commit();
}

```

In cases where DOM manipulations would cause jank, this instead allows the
user-agent to split up the work across several frames, yielding for script and
user interactions. When the updates are finished processing, the result is
displayed without jank.

### Further reading

- [Detailed explainer](https://github.com/chrishtr/display-locking/blob/master/explainer.md)
- [Sample
  code](https://github.com/chrishtr/display-locking/blob/master/sample-code)
  (work in progress)
- [Layout transitions](http://tabatkins.github.io/specs/layout-transitions/) - a
  similar proposal from 2014, with motivating examples that are applicable to
  display locking.

