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
  that is locked can be unlocked by the UA. This also includes sending the
  `rendersubtreeactivation` event. Some examples of this are find-in-page,
  focus navigation, or scrolling. Different rendersubtree tokens can control this
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
a `rendersubtreeactivation` signal is fired on the element and the `rendersubtree`
property is set to `""`. Note that `skip-activation` implies
`skip-viewport-activation` when disabling a particular algorithm.

### Viewport intersection
* When a locked element enters the viewport or nearing the viewport (50% margin), it will be activated.
* Can be disabled with `skip-viewport-activation` or `skip-activation`.

### Selection
* When text in locked subtree gets selected, all of the text's locked ancestors will be activated.
* Can be disabled with `skip-viewport-activation` or `skip-activation`.
 
### Sequential/tab-order focus navigation
* When an element is focused by sequential focus navigation (forward or backward),
the locked ancestors of the focused element will be activated
* Can disabled with `skip-viewport-activation` or `skip-activation`.

### Find-in-page
* Find-in-page will find text even if they are in a locked subtree (but will skip locked subtrees with `skip-activation`), counting them in the total match.
* Find-in-page won't activate *all* text that match - it will only activate one main/currently-selected match (see below).
* When find-in-page navigates to an active match (currently-highlighted/selected match) because it's the first match or through find-next/find-prev navigation, the locked ancestors of the active match text will be activated.
* Can be disabled with `skip-activation`.

### Accessibility
* Nodes in locked subtree (except those with `skip-activation`) are included in the AX tree but are marked as offscreen and don't have layout values, and thus are exposed to assistive technologies.
* AX selections, focus, scrolling/navigation to nodes in locked subtrees will activate the locked ancestors.
* Can be disabled with `skip-activation`.
* There's still an active discussion on this, please see https://github.com/WICG/display-locking/issues/102 for more.

### Anchor link navigation:
* When fragment link (ie url.html#elementid) navigation results in a navigation to an element in a locked subtree, its locked ancestors will be activated.
* Can be disabled with `skip-activation`.

#### focus():
* When `focus()` is called on an element and the element can be focused, its locked ancestors will be activated.
* Can disabled with `skip-activation`.
  
### scrollIntoView()
* When `scrollIntoView()` is called on an element, the locked ancestors of the element will be activated.
* Can be disabled with `skip-activation`.

## Current status (Chromium)

* **Activation**: Activation is the default behavior, which can be disabled with
  `skip-activation` or `skip-viewport-activation`. A `rendersubtreeactivation` event will
  be fired at animation frame timing.

* **Viewport Activation**: When considering activation due to viewport
  intersection, the code considers a 50% viewport margin on the implicit root.
  This means that if the element is clipped by the viewport, and it's within 50%
  of the viewport width/height  away from the closest edge, it will be activated.

* **Containment**: The presence of the `rendersubtree` attribute forces
  `contain: layout style;` *in addition* to any other containment. If the
  `invisible` token is present, it also forces `contain: size` in addition to
  layout, style and any other containment already present.

* **Intent to Experiment** proposed and approved ([thread](https://groups.google.com/a/chromium.org/d/msg/blink-dev/-6Cp2osHn50/VZhPCrXHDAAJ))

* **`intrinsic-size`** is implemented, and is available behind the
  CSSIntrinsicSize flag.
