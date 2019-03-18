# Display Locking

### Introduction

As the quality and performance of the user-agents progresses, developers are
looking towards the web as a platform to deliver rich, visually appealing and
complex applications. In addition, these applications increasingly use
animations or designs which extend or go beyond simple scrolling and animations
of elements. Examples of this include IDE-like interfaces, infinite lists,
tabbed UIs and drawers. One of the reasons for this progression is that the web
can deliver content comparable to native applications and it allows this content
to be easily distributed to users.

However, as developers are implementing richer and more compelling applications,
they are hitting the limits of performance. Specifically, some patterns commonly
used to produce dynamic content to the user can **cause the performance of the
whole application to degrade temporarily**. For example, when a site modifies the
DOM in a way that causes expensive layout, the user experiences jank --
noticeable delay in visual updates. Note that because of the complexity of the
DOM, layouts in some areas of the page can frequently cause jank elsewhere on
the page. For instance, a script-driven animation will jank anywhere on the page
if the user agent is busy performing layout. The reason for this is that script
runs in the same event loop as layout, and browser rendering semantics require
previous layout and visual update to be done before running subsequent script.

Creating more complex applications might also mean having a **larger number of things in a web page**,
making the rendering costs of having all of the content to actually be part of the DOM
prohibitively expensive.
This drives some amount of web authors to *virtualization* instead,
where they actually **refrain from putting things in the DOM**,
keeping content in memory in some non-DOM data structure,
with only the visible portion converted into DOM nodes and inserted into the document,
recycling them in and out as needed.
Popular examples include https://m.twitter.com/ and [react-window](https://github.com/bvaughn/react-window).
However, this **causes problems for the web app** because a lot of things like find-in-page,
accessibility, indexability, focus navigation, anchor links, etc.
**depends on having the contents in the DOM**.

We propose a new concept, **display locking**, to assist developers with **alleviating
jank caused by DOM updates** and having some control on **when to pay rendering costs**,
while also **allowing things that depend on the updates to get the up-to-date values** when urgent.

Using display locking, the developer will be able to lock an element and its subtree,
preventing visual updates.
Then, the developer will be able construct the locked subtree’s DOM however they desire,
and insert it into the DOM without any rendering cost or jank. After insertion, the element
can be updated. These updates can be *co-operative* -- **interleaved with other
work such as running script or DOM updates outside of the locked subtree**. The
developer will then be able to commit the element's lock, causing the visual
updates of the modified subtree to appear.  In essence, display locking will
make it possible to **perform complicated DOM updates without causing the rest of
the page to jank** and **only pay rendering costs on things that really need it**.

In the rest of the document, we describe the display locking concept in detail.
We will first examine some motivating examples which we commonly observe in rich
web applications. We will discuss the intent of each of the examples, as well as
the areas that can potentially cause jank. In order to understand the display
locking proposal details, we will then describe common phases in user-agent
lifecycles. Specifically, we will talk about script, layout, paint, compositing,
and presentation to the screen as well as areas that display locking is geared
to improve. We will then move on to describe the API changes required to
implement display locking, with a description of each new functionality. With
that in place, we will revisit the motivating examples to see how display
locking can be applied in order to improve their performance. Finally, we will
discuss possible display locking implementations, along with possible corner
cases and limitations.

---
### Motivating Examples

Consider the following example.

```html
 <style>
   #container {
    width: 100px;
    height: 100px;
    contain: content;
  }
 </style>
 <div id="container">
   <div id="complicated_subtree" style="display: none">
   ...
   </div>
 </div>

 <script>
 function presentContent() {
   let subtree = document.getElementById("complicated_subtree");
   subtree.style.display = "block";
   window.requestAnimationFrame(onContentPresented);
 }

 window.onload = function() {
   window.requestAnimationFrame(presentContent);
 };
 </script>
```

Here the content in the `complicated_subtree` is either already loaded or is
constructed at some earlier time. However, the `div` has `display: none`,
meaning that it isn’t visible and does not take up any space in the page. Then,
some event, load in this example, triggers the content to be displayed by
flipping the display property to `block`. This causes the user-agent to process
the DOM, lay it out, paint it, composite it, and present it to the screen.
However, depending on the complexity of the DOM contained in the `div`, this
process can be slow and take several frames. During this work, the page janks
causing script, rAF animations, and user input to stall and in some cases even
prevent the page from scrolling.

Let’s look at an [actual example of
this](https://drive.google.com/file/d/1Qip6D4Allotua8S6xSXzOhNolnvYPYjt/view?usp=sharing
"spinner jank"). Here, the toggle button changes the display property of the
`complicated_subtree` between `block` and `none`. We have also added a rAF
animation that records the length between the last ten requestAnimationFrame
callbacks. Note that when we switch the display property to `block`, the page
janks and the animation stops since requestAnimationFrame callbacks are not
running. After everything is laid out and is ready to be presented, the
requestAnimationFrame continues and we observe a noticeable increase in one
frame’s delay (indicated as the red slice). This delay is roughly 500ms, which
is enough to notice the animation stall.

<div style="text-align: center">
  <img src="resources/spinner_jank.jpg" alt="spinner jank">
</div>
&nbsp;

To reiterate, this effect occurs due to the user-agent laying out content that
we want to be presented. **Even though the spinner animation is not a part of
the subtree** (it’s an iframe in this example), it still janks because **DOM
updates are atomic.**


#### Common patterns

There are other common patterns that exhibit similar behavior, and as a result
suffer from similar drawbacks.

- Resizing multi-pane UI with complex layout within each pane (IDEs do this).
- Many widgets, for example YouTube: this site goes to great lengths to avoid
  long layouts, by incrementalizing their Polymer DOM updates. This is perhaps
  made trickier because Polymer uses a decentralized, widget-based update
  system.
- Latency-sensitive type/scroll with complex layout change. For example, display
  of search results as-you-type without janking the text input box.
- Expand or contract of an item with an infinite list (accordion view)
- Measuring layout, with intent of sizing containers without actually displaying
  the contents.

In general, **large-scale updates** to application state in a web app can cause
updates which **induce large document lifecycle updates**, including style, layout,
compositing, and paint. In turn, these can cause jank on the page, due to
lifecycle updates being synchronous with script and user interactions.

In a large enough web app, even having small updates like window resizing or a style tweak might be expensive,
due to the **large amount of things getting affected**,
even though we might **not necessarily need every part of the web app to be up-to-date all the time**. 

In the rest of this document, we discuss display locking and how it can help
with situations like the ones mentioned above.

---
### Background: User-agent's Update Phases

When processing DOM changes, the user-agent typically goes through several
stages, which we will call *update phases*. In order to understand how display
locking proposal is going to work, we will briefly discuss the major update
phases.

**Note to the reader**: these phases are covered in greater detail in the
[rendering event
loop](https://github.com/chrishtr/rendering/blob/master/rendering-event-loop.md).
Here, we briefly go over the main update phases that happen in a typical
user-agent.

#### Script
During the script update phase, the user-agent executes script requested by the
page. At this time, the script on the page is free to mutate the DOM in any way.
It can do things like update element style, position elements based on a custom
animation, add and remove elements, and many other things. Synchronous
javascript functions that are invoked here will finish to completion. This means
that if a function runs for a long time, other update phases cannot proceed and
the page janks. Script, however, cannot be interrupted by the user-agent.
Because of this, developers should always be mindful of long running script.

After the script phase finishes, the DOM structure and style for the next visual
update is finalized.

#### Layout and paint
During the layout and paint phase, the user-agent goes through the DOM and
updates any properties that need to be updated. Specifically, it can apply new
style to elements, position and size the elements based on style, as well as
generate draw commands needed to visually display the content to the screen.
Note that it is wasteful to do these steps on every element, so the user-agent
typically maintains a set of elements that have changed, or could have been
changed by updates to style or elements which may have happened during script
execution. This is typically done using some sort of dirty bits, or a list of
dirty elements.

For the elements that do need updates, the user-agent processes them in some
defined order. This work finishes to completion. This means that depending on
the complexity of the DOM, and the number of changes required, this process can
take a long time. This, again, causes jank since other update phases do not
happen while layout and paint is executing.

The output of the layout and paint phase is a sequence of draw commands which,
when executed, will draw the current visual update.

#### Compositing
In order to avoid doing repeated work, modern user-agents also employ
compositing. Compositing is a process of splitting up draw commands into
separate layers, each with its own backing, in order to avoid re-rasterizing
content in other layers.

As an example, if a div is animating and moving across the screen using a css
animation, then it might make sense to separate the div and its draw commands
into a separate layer. This would mean that the content of the div can be
rasterized once, and the animation would simply move the backing across the
screen. This is typically significantly faster than invalidating content,
discarding the previously rasterized content, and rasterizing new content in a
new location.

Compositing is typically determined by the user-agent based on heuristics. It
isn’t perfect, but it can help by reducing the amount of time spent rasterizing
content. On top of that, the developer can provide hints to the user-agent
indicating that an element, and its subtree, are likely to move together. For
example, “will-change: transform” is one such property that the user-agent may
use to promote the affected element to a separate layer.

The output of the compositing update phase is a set of layers with either draw
commands, or rasterized textures.

#### Presentation
The final update phase is presentation to the screen. By this time, the
user-agent has generated draw commands, or possibly even rasterized those into
separate layers, and arranged the layers in the final locations. With GPU
acceleration, this update phase issues GPU commands to the driver to display the
layers and its contents.

At the end of the presentation phase, the content is finally visible to the
user.

---
### Proposal

The display locking proposal is intended to **improve the script, layout, paint,
as well as parts of the compositing update phases**. In particular, it aims to add
javascript APIs to allow the developer to lock an element for display.

With the proposed APIs,
**web authors can control when not to pay the updating costs for a locked element**,
and also **request the user-agent to do the updates in a non-janky way**.

### Example code

In this example, we want to add a lot of items to a list in a non-janky way.

```html
<ul id="itemsList">
 <div>Item #1</div>
 <div>Item #2</div>
 <div>Item #3</div>
</ul>

<script>
// Remaining items in the list are in |remainingItems|.
// We want to add the remaining list items asynchronously.
remainingItems.forEach(item => {
  requestIdleCallback((deadline) => {
    let itemEl = createElementForItem(item);
    itemEl.displayLock.acquire({ timeout: Infinity, activatable: true });
    itemsList.appendChild(itemEl);
    // We can do expensive operations to the items without worrying of the rendering costs.
    // After we finished all the operations we can trigger a co-operative update & commit.
    // We might do this in a fancier way by not actually committing the element,
    // leaving it locked and detecting when to commit by using IntersectionObservers etc,
    // but that's out of the scope of this example :)
    doExpensiveOperationsToEl(itemEl).then(element.updateAndCommit);
  });
});

// Getting style/layout values within a locked subtree :
// - If we access child.offsetTop directly, it will cause a forced synchronous update
//   (see [what-forces-layout](https://gist.github.com/paulirish/5d52fb081b3570c81e3a)).
// - We can call update() on the locked element to trigger a co-operative update and get
//   the value after the calculations finished instead.
function getOffsetTopForLockedElementChild(child, lockedElement) {
  return new Promise((resolve) => {
    lockedElement.displayLock.update().then(() => { resolve(child.offsetTop) };
  });
}
</script>
```

### Proposed APIs Details

#### Element.displayLock

With display locking, each of the `Element` objects has a new attribute,
`displayLock` which returns a `DisplayLockContext` representing the display
lock. The rest of the display locking functionality happens by interacting with this
object. The `DisplayLockContext` is bound to the element from which it was
retrieved, meaning that operations on the object will affect that element.

#### DisplayLockContext.acquire(options)

Acquire performs the following steps:

* It finishes all of the update phases for the current state, if needed.
* (in lock-after-append) It saves the output of the layout and paint phase (the
  painted draw commands) and stashes them to be used when the locked element
  needs to be painted. Note that in the case of an element that is outside of
  the DOM, this painted output is empty.
* It marks the source element as locked for display, preventing normal update
  phases to recurse into it.
* It returns a promise which, when resolved, indicates that the above work has
  been completed.

Note that the acquire call takes options, consisting of the following key-value
pairs:
* (optional) `timeout`: `timeoutInMilliseconds`
  * This is an optional timeout value for the duration of the lock. When the
    lock is acquired, and commit is not called within `timeoutInMilliseconds`
    milliseconds, the lock will be automatically committed. This is a safety
    valve for protecting pages from inadvertently leaving an element locked
    indefinitely due to uncaught exceptions or other bugs in the code. For a
    more sophisticated use cases, the value of Infinity will effectively disable
    the timeout.
* (optional) `activatable`: `true|false`
  * Signifies whether the user-agent can automatically commit the lock
    when the browser needs to *activate* the locked element or its descendants.
    See [element activation section](#element-activation) for more details.
  * Not setting this, or setting this to `false`,
    means this element and its descendants will be ignored in cases like
    find-in-page, anchor link navigation, focus navigation, etc.

#### DisplayLockContext.locked

This is a boolean value that will be `true`
if the `Element` associated with this `DisplayLockContext` is in the locked state,
and `false` if not.

#### The locked state

#### What happens

When its lock is acquired, if the element already existed in the DOM,
the visual content that was present at the time the lock was acquired is cleared,
leaving only the space it takes in layout,
similar to an element with `visibility: hidden;`.

If the element was locked before being inserted into the DOM,
then it is effectively inserted in an state similar to `display: none;`,

##### Modifications to the locked subtree

Changes to the DOM, style, layout, etc of the *locked element itself*
will be applied immediately if the element is locked after it is attached to the DOM,
but will be ignored if it is locked before it is attached to the DOM.
See [lock-after-append](#lock-after-append-locked-subtree) vs [lock-before-append](#lock-before-append-locked-element--subtree).

Changes to the *descendants of a locked element* updates the DOM in such a way that script can inspect it immediately,
but no updates wil be painted until the element is unlocked,
through `commit()` or the user-agent activating the element.

Style and layout updates will not be inspectable immediately without first trigerring an update,
either through calling `update()`
or trigerring a *forced update* by calling style and layout inducing methods/properties
(see [what-forces-layout](https://gist.github.com/paulirish/5d52fb081b3570c81e3a)),
which will force style or layout synchronously in order to return correct values,
but not actually painting it.

#### DisplayLockContext.update()

This operation causes the lock to allow co-operative updates on the element and
its subtree in preparation for display or measuring layout. It performs the
following steps:

* The updates to rendering stages are processed as long as they do not cause
  undue delay to the rest of the update phases
* If an undue delay is likely to be caused, the work already completed is
  processed and the update phase yields to other update phases for unlocked
  content.
* Future update phases for the locked element are gated on the fact that the
  previous update phases for this locked element have completed.
* As before, when the element is painted, the changes are not visible to the
  user. Note that `update()` is free to skip doing any paint work for the
  element, since the visual state will not be displayed.
* When all of the above phases finish, the promise returned by
  `update()` is resolved, giving the script opportunity to measure layout, do
  other work, or signal to other subsystems that this element and its subtree
  are "prepared".

#### DisplayLockContext.commit()

This operation causes the lock to be released and visual state of the element
and its subtree to become visible to the user. It performs the following steps:

* It allows the element and its subtree to be updated for all the rendering
  phases.
* It paints the element and ensures that the process of displaying to the user
  is started. (e.g. it sends draw commands to the compositor, or to the GPU).
* It resolves the promise returned by `commit()`, indicating that the above work
  was finished.

Note that it is OK to call `commit()` following an earlier `update()` which has
not yet resolved. This causes work that is still needed to become synchronous,
enabling the idle-until-urgent pattern.


#### DisplayLockContext.updateAndCommit()

This operation combines the effects of an `update()` and a `commit()` calls:
* It causes the element to be co-operatively updated.
* When the update is finished, the element is committed resulting in visual
  updates to appear on screen.


#### Element activation

When an element is locked,
there are some actions in the page that might require the element to get unlocked and be rendered to work properly.

If the browser needs to *activate* a locked element
or an element that has one or more locked ancestors,
in order to make the activated element be visible,
it will commit all locked elements in the inclusive ancestor chain of the element to be activated,
unless at least one of them is locked without the `activatable` flag set to `true`,
in which case the element can't be activated.

*Activating an element* is defined as one of the following actions:
 - `focus()` is called on the element
 - `scrollIntoView()` is called on the element
 - Tab order navigation lands on the element
 - Anchor link navigation navigates to the element
 - Find-in-page active match navigation goes to the element

For all the elements that we commit as part of element activation,
we will send a `beforeactivate` event to it just before committing.

#### beforeactivate event

| property  | value  |
|---|---|
| bubbles  | true  |
| composed | false |
| target  | previously-locked ancestor |
| activatedElement  | The element that needs activation  |


Consider this tree structure:
```html
<div id="first">
 <div id="second"> <!-- locked, activatable -->
  <div id="third">
    blah
    <div id="fourth">  <!-- locked, activatable -->
     bleh
    </div>
  </div>
 </div>
<div>
```

If we need to activate `#fourth` div,
then two `beforeactivate` events will be fired.
One targeted at `#fourth` and one targeted at `#second`.
The `activatedElement` in both of them is the `#fourth` div.

If instead we need to activate `#third` div,
only one `beforeactivate` event will be fired,
at `#second` with `#third` in the `activatedElement` field.

If `#second` is actually locked with `activatable` set to false,
we can't activate anything `#second` or anything inside its subtree.
For example if `#fourth` needs activation,
even if we activate `#fourth`,
it will not be visible because `#second` is still locked.

---
### Restrictions

In consideration of display locking, we have also discussed when it would and
would not be appropriate to allow an element to be locked. One main
consideration for this is containment. That is, it seems to make sense to
require that the element that is going to be locked provides containment for
style and layout. This can be achieved with the `contain: style layout;` CSS
property.

This allows us to better reason about the expected behavior of the page. Specifically,
* Layout of the locked subtree will not affect the layout of other elements
  outside of the locked element. This ensures that we can process as little or
  as much of the subtree’s layout without visually affecting the rest of the
  page.
* Finally, style of the locked subtree will not affect style of elements outside
  of the locked element. Similarly to layout containment, this ensures that we
  can process any number of elements on the locked subtree without visually
  changing the content on the rest of the page.

---
### Implementation description

Note to the reader: this section describes the sample prototype implementation.
The design choices here are not meant to represent final design decisions, but
act as a proof of concept implementation in Blink. Feel free to skip this
section.

There are three key components to implementing the display locking API.

First, in lock-after-append, we need to be able to snapshot the current visual
representation of the locked element, so that future updates to the page have
content to populate in place of the locked element. Since the user-agents vary
greatly in the way they generate commands for drawing content to screen, we will
omit a detailed discussion of this. It suffices to say that any kind of
double-buffering approach would work. That is, the user-agent could generate the
commands needed to render the element and its subtree at the time the lock is
acquired. Then, for the duration of the lock it uses these commands to render
the content, while accumulating new draw commands in a separate buffer to be
used when the lock is released.

Second, we need to implement the co-operative update phases.  That is, we need a
way to prevent the update phases in the locked subtree from blocking other
phases from occurring for other parts of the tree. The co-operative updates part
of the API can be implemented in several ways. Here, we describe one possible
implementation, which is the basis for our prototype implementation. We call
this approach budgeted update phases.

##### Budgeted update phases

In the budgeted update phases approach, we introduce a new concept to the
user-agent, called an update budget. This budget dictates how much of a locked
subtree should be processed. As an example, consider the following the following
pseudo code for laying out an element:

```cpp
void LayoutElement::UpdateLayout() {
  UpdateOwnLayout();
  ClearDirtyBitForSelfLayout();

  for (auto* child : GetChildren()) {
    if (child->NeedsLayoutForSelfOrChildren())
      child->UpdateLayout();
  }
  ClearDirtyBitForChildrenLayout();
}
```

Here the code goes through the layout tree walk. At each step, the element
updates its own layout, then iterates over all of its children. If a child, or a
sub-element of the child needs a layout, then it recursively descends into the
call. Also note that every time an element completes its layout, it clears the
dirty bit for itself and after processing its children, it also clears the dirty
bit that indicates its children need processing.

With budgeted update phases, the recursive call gets a context which tracks the
current budget. Furthermore, it checks the current budget to see if it expired.
If so, it aborts doing any future work. When it comes to dirty bits, if the
element processed its own layout, it clears its dirty bit. Also, if all of the
children and, by extension, their subtrees successfully complete layout, it also
clears the children dirty bit.

```cpp
void LayoutElement::UpdateLayout(DisplayLockContext& context) {
  auto scoped_visitor = context.VisitObject(this);
  if (context.BudgetExpired()) {
    context.DidSkipObject();
    return;
  }
  UpdateOwnLayout();
  ClearDirtyBitForSelfLayout();
  context.DidProcessObject();

  for (auto* child : GetChildren()) {
    if (child->NeedsLayoutForSelfOrChildren())
      child->UpdateLayout(context);
  }
  if (!context.HadSkippedDescendants())
    ClearDirtyBitForChildrenLayout();
}
```

Here, the budget is generated if context is visited with a locked element. That
is, if `this` specified in `VisitObject` is locked, then the context generates a
budget which will be used for all subtree calculations. In cases that the
context is not working in a locked subtree, `BudgetExpired()` always returns
false, meaning that work can proceed.

By adding similar code to all update phases, we achieve co-operative processing
of locked subtrees. Note that if a budget expired on during one of the phases,
such as layout, then we won’t process the subtree at all for the future phases,
such as paint. The reason for this is that the budget that we generate is stored
on a display lock which is associated with an element. Hence, if we have an
expired budget at an earlier phase, then it is guaranteed to be expired at a
later phase. As the code processes the DOM tree, it essentially aborts doing
work on a locked subtree if an earlier phase has not finished its work on that
subtree. All other parts of the subtree, like the rAF spinner in our earlier
example, are still processed as normal.

For completeness, the budget can consist of several things:

* Time budget: this tracks the time that has elapsed since work started and
  expires after a predefined period.
* Element count budget: this tracks the number of elements that have been
  processed, and expires after a fixed number was completed.
* A combination of the two: this tracks the number of elements that have been
  processed, and after a fixed number is complete, it checks if the time has
  expired. If it did, the budget expires. Otherwise, it continues to process the
  next batch of elements.

##### Other approaches

Budgeted update phases is only one of the ways to achieve co-operative work.
Other approaches are possible and may yield different behavior depending on
implementation:

* Threaded update phase: locked subtrees can be processed on a separate thread.
* Fixed phase updates: locked subtrees could be processed one update phase at a
  time. For example, layout finishes first and skips the other phases. On the
  next frame, paint finishes, and skips the remainder. Etc.

It’s possible that other approaches could have better behavior than budgeted
update phases, but any approach is feasible as long as it implements
non-janking phase updates.

Another consideration is what to display when the element is locked for display.
As described in the implementation details, one possibility is to snapshot the
current visual representation and use that throughout the lock's lifetime. Other
possbilities are listed below:
* Do not show any content until the processing is done.
* Allow the developer to specify the content to display
  * Allow the developer to specify HTML to display instead of the content while
    the real content is being processed
  * Allow the developer to specify a Bitmap to display when the element is
    locked, possibly but not necessarily containing the result of the content at
    the time the lock was acquired.

---
### Examples revisited

Let's revisit the motivating examples, modified with display locking:

```html
 <style>
#container {
    width: 100px;
    height: 100px;
    contain: content;
  }
 </style>
 <div id="container">
   <div id="complicated_subtree" style="display: none">
   ...
   </div>
 </div>

 <script>
 async function presentContent() {
   let lock = document.getElementById("container").displayLock;
   await lock.acquire({ timeout: Infinity, activatable: true });
   document.getElementById("complicated_subtree").style.display = "block";
   lock.update().then(() => { lock.commit().then(onContentPresented); });
 }

 window.onload = presentContent;
 </script>
```

Similar to the original example, on load we present the content in the
`complicated_subtree`. However, we first acquire a lock on the item and then
modify the contents. This causes the user-agent to lock the container's subtree
for visual updates. Then, we modify the DOM by setting the `complicated_subtree`
display property to `block`. Then we call `update()` and when that finishes,
`commit()`. This sequence of commands causes the user-agent to co-operatively
update the phases without introducing an undue delay for the rest of the
updates. In other words, the remainder of the page remains interactive and
animating. When the updates eventually complete, we commit and when that promise
resolves and `onContentPresented` is invoked.

Also, because we specify the `activatable` option as `true` when acquiring,
the contents of `complicated_subtree` will be considered in focus navigation,
anchor link navigation, find-in-page, etc. even when the content is not
presented yet.

As before, let’s see a [real example, this time using a prototype of display
locking](https://drive.google.com/file/d/1r1aBi4P1_DMCZNXlpzW5jAibCEdT38YB/view?usp=sharing
"spinner without jank"). When toggling display to be `block`, there is a
significantly smaller jank (yellow slice). This happens due to the prototype
implementation being incomplete (the delay is approximately 40ms).

<div style="text-align: center">
  <img src="resources/spinner_without_jank_1.jpg" alt="spinner without jank">
</div>
&nbsp;

However, other than the small jank the animation keep going, while the
user-agent lays out the content co-operatively. Note that even though the
animation is going ahead, the content is not yet presented. We can detect this
from JavaScript, by observing that the `commit()` promise has not
yet resolved.

<div style="text-align: center">
  <img src="resources/spinner_without_jank_2.jpg" alt="spinner without jank">
</div>
&nbsp;

After the user-agent completes the update phases and commit, the content is
presented without jank.

<div style="text-align: center">
  <img src="resources/spinner_without_jank_3.jpg" alt="spinner without jank">
</div>
&nbsp;

---
### Locked subtree vs locked element + subtree

Display locking supports two modes of operation, distinguished by when the lock
is acquired

#### Lock-after-append. (locked subtree)

When the element's lock is acquired while the element is already a part of the
DOM, then the draw commands associated with that element's subtree are stashed
and used while the lock is acquired. Note that in this mode, changes to the
element itself (e.g. border, size) are updated synchronously. In other words,
the element itself is not locked for display, only its subtree.

The visual content of the locked subtree is cleared,
leaving only the space occupied by the locked element,
similar to having `visibility: hidden` on the locked element.

This mode is appropriate to use when the element's subtree needs to be updated
without jank.

#### Lock-before-append (locked element + subtree)

When the element's lock is acquired before the element is inserted into the DOM,
then the element itself as well as its subtree are considered locked. This means
that when the element is inserted into the DOM it will not visually change the
output of the page.

In this mode, changes to either the element itself or to the subtree will not be
reflected on screen until the lock is committed and the corresponding promise
resolves.

This mode is appropriate to use when a new element or a widget is inserted into
the page and it should present asynchonously when ready without causing jank.

---
### Dealing with user input

One of the difficult aspects of locking an element for display in
lock-after-append mode is deciding what to do with user input (e.g. mouse
clicks) that are targeted at the element which is locked for display.

##### Queuing up input and replaying

One option is to queue up the user input replaying it once the element is
unlocked.

Immediately, there is a problem with this approach: conceptually, user
input is targeted at what the user is currently seeing on screen. However, when
the display is locked for view, it means that whatever is currently visible does
not necessarily reflect the DOM that currently comprises the element. To put it
differently, script could have already mutated the DOM, removed interactive
elements, added more interactive elements, etc, which means that the user input
may be targeting an unexpected (from the user's perspective) element.

Based on this argument, it seems that replaying the user input would often cause
unexpected interactions. The only case where it might be reasonable to replay
user input is if the locked subtree was not and _will_ not be modified before it
is unlocked. However, detecting whether script intends to modify the locked
subtree's DOM is impossible.

##### Ignoring input

Another alternative, which seems more prudent, is to ignore the user input when
the target is a locked subtree. This prevents unexpected interactions.

However, this comes at a cost of potential user frustration when attempting to
interact with a locked subtree. From the user's perspective, the page would
appear to be frozen: buttons don't change state, text fields don't gain focus,
etc. An option to mitigate the appearance of a frozen page is to display an
affordance, such as graying out the locked element, or displaying a spinner, if
the user attempts to interact with it.

This option does not seem to be ideal, since it's hard to agree on what type of
affordance makes sense to display in any particular instance. That being said,
it appears to be a better choice when compared to simply ignoring user input.

---
### Other edge cases

There are a number of edge cases that need to be considered when working with
display locking. This section briefly lists a few of them, but the list is far
from complete.

* Subtree elements are moved in and out of the locked subtree
    * In this situation, we may have an element that is taken out of a locked
      subtree and moved elsewhere in the DOM. We think this case can be handled
      naturally, since any element added in an unlocked subtree will be
      processed by the next frame. Similarly to elements moved into locked
      subtrees, they will be processed co-operatively regardless of their
      origin.
* Locked element is moved around on the page.
    * In this situation we simply move the whole locked element around the DOM.
      This case should also be handled naturally, since the subtree will be
      processed co-operatively. Containment restriction also alleviates concerns
      about other elements changing as a result.
* Locked element in a locked subtree.
    * In this situation we have a locked element, which itself is located
      within a locked subtree. There’s a question of how to budget this
      correctly. It is possible that we just use a single budget for all of the
      subtree, even if some elements are themselves locked. It is also possible
      that we only process innermost locked subtree first. There are likely some
      other options here, but overall it doesn’t seem to be a blocking issue.
* Multiple locked elements.
    * Similar to above, this case simply has multiple locked elements on the
      page. Conceptually we should use a single budget for all of the locked
      elements on the page, to prevent jank. It is also possible that each
      subtree gets an independent budget to ensure forward progress on all
      elements, but at the cost of jank induced by the total budget exceeding
      the length of a frame.
* Querying layout inducing properties in a locked subtree.
    * This is the most worrying case. We may query things such as size and
      offset from an element in a locked subtree. Since the layout isn’t
      guaranteed to be finished for the elements queried, it is unclear what is
      the best course of action here. One possibility is to force layout for the
      queried element and, possibly, its ancestors and descendants, depending of
      what is required to get the queried information. Another possibility is to
      simply disallow such queries into the locked subtree, failing with an
      invalid number returned.
* Find-in-page and tab order.
    * It is unclear whether the content put into a locked subtree should be
      discoverable by find-in-page or tab order navigation. It is possible that
      this decision will be left up to the script author by adding an additional
      option to `acquire()`, which would dictate whether the content modified is
      searchable (e.g. { searchable: true }).

Display locking will certainly have edge cases that need to be considered, some
of which we have listed here. The general feeling is that the edge cases are
tractable and, given the benefits of the feature, should not block progress on
its implementation.

---
### Discussion

Let’s briefly touch on two aspects of web standards that are important for new
features: ergonomics and interoperability.

Ergonomics is a concern for easy of usability of the feature. Display locking
leverages a notion of a lock, which is common in threaded programming. In
typical threaded programming, a lock restricts access to data for one thread:
the thread that owns the lock. Display locking extends the notion by restricting
access to visual representation of the locked subtree. In other words, the
visual representation of the content is only accessible by the co-operative
updates. It is not accessible by a system that draws the content to the screen.
Hence, with the addition of three functions and leveraging known concepts,
display locking is a simple and straightforward API that alleviates jank due to
user-agent update phases. As demonstrated by the example above, the burden of
code complexity is also minimal.

Interoperability is another important aspect of new features. This is a concern
for cross browser support of new proposed features. Since display locking is
largely a performance API, it can be stubbed out for the most part by only
preventing the browser from displaying new content. The layout and other update
phases can still be non-co-operative. The co-operative updates can be
incrementally implemented in user-agents. As for the web developers, the feature
detection of `getDisplayLock` is also simple. If it is present, the developer
can wait on the lock acquisition and commit. Otherwise, the developer simply
updates the DOM in the same way as before. This makes this feature easy to adopt
and, in case of problems, deprecate.
