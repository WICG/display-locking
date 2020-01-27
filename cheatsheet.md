# Cheat-sheet for terminology and status.

As the feature is being developed, it can become hard to keep all of the
documentation, explainers, and READMEs up to date. This is a small cheat-sheet
that should be the first thing to be up-to-date with the current terminlogoy
and status. Note that this does not go into details on what each component
represents and why. Instead, this acts as an easy way to map up-to-date concepts
with possibly out-of-date explainers.

## Terminology

* **Display Locking**: this is the feature name. It refers to the fact that an
  element can be "locked from being displayed".
* **Locked element**: the element's subtree is not being rendered or hit-tested.
* **Unlocked element**: the element's subtree is being rendered and hit-tested.
  Note that this state is independent of whether render-subtree: invisible
  property applies to the element. See below for more information.
* **render-subtree**: this is an alternative feature name. It refers to the CSS
  property `render-subtree` which can take several values in order to lock an
  element.
  * `render-subtree: invisible`: this refers to the fact that this
    element's locked state is being managed by the user-agent.  Specifically,
    the user-agent will unlock elements as they approach the viewport and lock
    them as they move away from the viewport. Locked content is discoverable by
    user-agent algorithms which may bring the element into view, causing it to
    be unlocked. This configuration is in the initial proposal.
  * `render-subtree: invisible skip-activation`: this refers to the
    fact that the subtree is locked and the user-agent will not manage the
    state. The content is not discoverable by any user-agent algorithms.
    This configuration is in the initial proposal.
  * `render-subtree: invisible skip-viewport-activation`: this refers to the
    fact that the subtree is locked and the user-agent will not manage the
    state. The content is, however, discoverable by user-agent algorithms, which
    may fire events with targets in the locked subtree.
    This configuration is tentative and is not a part of the initial proposal.
    Specifically, the details of the event have not been discussed in detail
    yet.
  * Note that all of the above configurations enforce `contain: style layout;`.
    Additionally, if the element is locked it also enforces `contain: size;`.
    This containment is additive and takes effect on top of any other
    containment specified.
* **contain-intrinsic-size**: this refers to a complementary CSS feature called
  contain-intrinsic-size, which allows the developer to specify a placeholder size
  when size containment is present, as it would be when the subtree is locked.

## User-agent algorithms

This is a list of algorithms that can bring an element into view or otherwise
cause the user-agent to unlock the elements. Note that unless explicitly stated
otherwise, the language below uses 'unlock' as the action since that is the
action is performed on a `render-subtree: invisible` element. However, it should
be understood that for `render-subtree: invisible skip-viewport-activation` the
action is instead to fire an event while keeping the element locked. Similarly,
for `render-subtree: invisible skip-activation`, the action is similar to that
of a `display: none` subtree.

### Viewport intersection
* When a locked element enters a region near the viewport (50% margin on the
  viewport rect), it will be unlocked.

### Selection
* When content (text, image, etc) in locked subtree gets selected, all of the
  content's locked ancestors will be unlocked. Note that this only applies to
  `render-subtree: invisible` configuration; other configurations pay no special
  attention to selection.

### Sequential/tab-order focus navigation
* When an element is focused by sequential focus navigation (forward or backward),
  the locked ancestors of the focused element will be unlocked. Note that this
  only applies to `render-subtree: invisible` configuration. other configurations
  pay no special attention to sequential focus navigation.

### Find-in-page
* Find-in-page will find text even in locked subtrees if the configuration
  allows for it. The active or main match will cause all of its locked ancestors
  to be unlocked. Tentative design decision: for `render-subtree: invisible
  skip-viewport-activation` cases, the find-in-page algorithm, upon finding an
  active match in a locked subtree, will issue a signal and
  yield until script had a chance to react to the signal. At this time, the
  algorithm will resume and only consider the active match as valid if the
  subtree is now unlocked. Otherwise, it will proceed with looking for other
  matches outside of the current locked root.

### Accessibility
* Subtrees in the `render-subtree: invisible` configuration are included in the
  accessibility tree. Subtrees in other configurations are omitted. Note that
  there may be other ways of bringing the content into view from accessibility
  technology.  These algorithms should, for the most part, behave consistently
  with non-accessibility ways of bringing content into view.

### Anchor link navigation:
* When fragment link (ie url.html#elementid or url.html#:~:text=foo) navigation
  results in a navigation to an element in a locked subtree, its locked
  ancestors will be unlocked. Tentative design: Similar situations as in
  find-in-page cae should be considered. In particular, it may be prudent to
  yield for script to be able to handle an event before proceeding with a
  scroll.

#### focus():
* When `focus()` is called on an element and the element can be focused, its
  locked ancestors will be unlocked.

### scrollIntoView()
* When `scrollIntoView()` is called on an element, the locked ancestors of the
  element will be unlocked.

## Current status (Chromium)

* Initial proposal configurations are implemented, enabled by
  CSSIntrinsicSize runtime flag.

* contain-intrinsic-size is not implemented; it is recommended to use
  min-height/min-width in the interim.

