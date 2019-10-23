# Cheat-sheet for terminology and status.

As the feature is being developed, it can become hard to keep all of the
documentation, explainers, and READMEs up to date. This is a small cheat-sheet
that should be the first thing to be up-to-date with the current terminlogoy
and status. Note that this does not go into details on what each component
represents and why. Instead, this acts as an easy way to map up-to-date concepts
with possibly out-of-date explainers.

## Revision log

<table>
<th style="width: 110px">date</th><th>description</th>
<tr>
  <td>2019-10-10</td>
  <td>Initial version.</td>
</tr>
<tr>
  <td>2019-10-11</td>
  <td>s/skip-visibility-activation/skip-viewport-activation/</td>
</tr>
<tr>
  <td>2019-10-23</td>
  <td>add a section on activation, add selection &
      tab-navigation to viewport activation, rename content-size to
      intrinsic-size</td>
</tr>
</table>

## Terminology

* **Display Locking**: this is the feature name. It refers to the fact that an
  element can be "locked from being displayed".
* **rendersubtree**: this is an alternative version of the feature name. It
  refers to the element attribute `rendersubtree` which can be set to various
  values in order to lock an element.
  * `rendersubtree="invisible"`: this refers to the fact that this
    element is currently locked (i.e. invisible)
  * `rendersubtree="invisible skip-activation"`: this refers to the
    fact that the UA cannot activate this element (see Activation)
  * `rendersubtree="invisible skip-viewport-activation"`: this refers to the
    fact that the UA can only activate the element on "viewport action" (see User
    Activation; Activation).
* **Activation / Activatability**: this refers to the notion that an element
  that is locked can be unlocked by the UA. Some examples of this are
  find-in-page, or scrolling. Different rendersubtree tokens can control this
  behavior.
* **User Activation**: this refers to activation caused by user or developer
  actions. Examples of this include find-in-page, scrollIntoView().
  Notably, this does *not* include UA activation caused by Viewport Activation
  described below.
* **Viewport Activation**: this refers to activation caused by the element
  entering the visible region of the page (i.e. intersection with the viewport).
  It also include user selection and tab navigation. In other words, if viewport
  activation is skipped, then the element will not activate on user selection,
  tab navigation, or viewport intersection.
* **intrinsic-size**: this refers to a complementary CSS feature called
  intrinsic-size, which allows the developer to specify a placeholder size when
  rendersubtree makes the subtree invisible.

## Activation algorithms

This is a list of algorithms that cause the element to activate. When activated,
a `beforeactivate` signal is fired on the element and the `rendersubtree`
property is set to `""`. Note that `skip-activation` implies
`skip-viewport-activation` when disabling a particular algorithm.

* find-in-page: when find-in-page finds an element and makes it an active match
  (current match), the locked ancestors of the element are activated. (disabled
  by `skip-activation`)
* viewport intersection: when a locked element enters the viewport, it is
  activated (disabled by `skip-viewport-activation`)
* scrollIntoView(): when scrollIntoView() is called on an element, the locked
  ancestors of the element are activated (disabled by `skip-activation`)
* accessibility: screen readers and other accessibility features activate
  elements (disabled by `skip-activation`)
* anchor link navigation: when fragment links (ie url.html#elementid) navigates
  to an element, its locked ancestors are activated (disabled by
  `skip-activation`)
* tab order navigation: when an element is focused by tab navigation (forward or
  backward), the locked ancestors of the focused element are activated (disabled
  by `skip-viewport-activation`)
* selection: when an element is selected, it is activated so its children can be
  selected as well (disabled by `skip-viewport-activation`)
* focus(): when focus() is called on an element, its locked ancestors are
  activated (disabled by `skip-activation`)
## Current status (Chromium)

* **Activation**: As of [this patch](https://chromium-review.googlesource.com/c/chromium/src/+/1853854),
  activation is the default behavior, which can be disabled with
  `skip-activation` or `skip-viewport-activation`. A `beforeactivate` event will be fired before showing the new content on-screen.

* **Containment**: The presence of the `rendersubtree` attribute forces
  `contain: layout style;` *in addition* to any other containment. If the
  `invisible` token is present, it also forces `contain: size` in addition to
  layout, style and any other containment already present.

* **Intent to Experiment** proposed and approved ([thread](https://groups.google.com/a/chromium.org/d/msg/blink-dev/-6Cp2osHn50/VZhPCrXHDAAJ))

* **`intrinsic-size`** is in process of being implemented [patch](https://chromium-review.googlesource.com/c/chromium/src/+/1867043)
