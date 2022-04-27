# Element.isVisible explainer

The web features several ways that content can be hidden. Each of the ways may
have distinct characteristics and are used for different purposes:

* `visibility: hidden`: this hides the element visually but keeps its layout box and allows for the descendant to
  make itself visible. In other words, this property doesn't affect the subtree,
  with the exception of providing the `visibility` value to be inherited by
  descendant styles.
* `opacity: 0`: this hides the element and its subtree visually but keeps its layout box and allows for animations
  that fade content in or out.
* `display: none`: this hides the element and its subtree, it also destroys
  rendering state (removes its layout box) and makes the DOM subtree not take up any time while
  processing rendering.
* `content-visibility: hidden`: this hides the subtree of an element, without
  destroying the rendering state (i.e. its layout box is preserved). It also makes the DOM subtree not take up any
  time while processing rendering, but allows for the possibility for such
  rendering state to be forced (e.g. via calls to getBoundingClientRect). It
  also allows the element's subtree to be shown again quicker than `display:
  none` would allow, since `content-visibility: hidden` would preserve rendering
  state.

There are times when script wants to determine whether an element is visible to
the user, or would be visible if it was in the viewport. This can be used for a
variety of reasons, such as general state tracking of visibility.

This can be hard to compute correctly, since the script author needs to remember to
check all the necessary methods by which content can be hidden (including
accounting for the fact that new methods could be added). It also is hard to do
efficiently, since for example access into the subtree hidden by
`content-visibility: hidden` may cause rendering work to be updated.

Additionally, checking if an element is hidden by a closed shadow tree, such as
the case with `<details>` element is difficult, if not impossible

For this reason, we propose to add Element.isVisible function to compute the
values, so that script authors may use this as a correct and efficient way to
determine the necessary visibility. This also returns a correct value for
elements slotted inside closed shadow trees.

The spec draft for the function can be found
[here](https://drafts.csswg.org/cssom-view/#dom-element-isvisible). Note that
the options specified to isVisible call allow developers to customize the check
they would like to make.
