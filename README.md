# Display Locking

This is a short proposal for Display Locking. For more details, see the
[explainer](https://github.com/chrishtr/display-locking/blob/master/explainer.md)

### Problem

Websites are frequently relying on dynamic content to present information. This
means that DOM is often manipulated using script to present rich and dynamic
content. There are cases when this cause cause slow rendering phase updates,
including style updates, layout, and paint. This, in turn, causes jank or
noticeable delay when presenting content, because rendering phase is updated
synchronously with user interactions and requestAnimationFrame script.

The following are a few of the common patterns that can cause jank:
- Resizing multi-pane UI with complex layout within each pane (e.g. IDEs)
- Adding widgets with complex DOM.
- Dynamically displaying content based on the current value of text input.
- Measuring layout of otherwise hidden DOM with intent of sizing containers.

Developers are aware of these issues and build systems to avoid adding complex
DOM at once in order to prevent jank. As an example, ReactJS is [adding
ability](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html)
to do async operations to help with this problem.

### Proposal

This document proposes to augment user-agent APIs to help address the problem of
slow rendering phase updates. Specifically, we propose a concept of display
locking an element.

If an element is locked for display, it means that any DOM
updates to its subtree are processed co-operatively, yielding periodically to
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

  // Modify element's subtree, which is processed co-operatively.
  element.innerHTML = ...;

  // Commit the lock. This returns a promise. When the user-agent is done
  // processing all of the rendering phase updates for the locked element,
  // this promise is resolved.
  lock.commit();
}

```

In cases where DOM manipulations would cause jank, this instead allows the
user-agent to split up the work across several frames, yielding for script and
user interactions. When the updates are finished processing, the result is
displayed without jank.

### Considerations

There are several important considerations when using display locking:
- Animations in a locked subtree will likely need to be frozen, since the
  underlying DOM can change and take several frames to process
- User input targeted for the locked element or its subtree will likely need to
  be ignored, since the visual representation of the content might be completely
  different at the time the lock is released.
- Layout inducing properties, (e.g. `offsetTop`) will likely not contain correct
  information for elements in a locked subtree.


### Further reading

- [Detailed explainer](https://github.com/chrishtr/display-locking/blob/master/explainer.md)
- [Sample
  code](https://github.com/chrishtr/display-locking/blob/master/sample-code)
  (work in progress)
- [Layout transitions](http://tabatkins.github.io/specs/layout-transitions/) - a
  similar proposal from 2014, with motivating examples that are applicable to
  display locking as well.





