# Cheat-sheet for terminology and status.

As the feature is being developed, it can become hard to keep all of the
documentation, explainers, and READMEs up to date. This is a small cheat-sheet
that should be the first thing to be up-to-date with the current terminlogoy
and status. Note that this does not go into details on what each component
represents and why. Instead, this acts as an easy way to map up-to-date concepts
with possibly out-of-date explainers.

## Revision log

2019-10-10: Initial version.

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
  * `rendersubtree="invisible skip-visibility-activation"`: this refers to the
    fact that the UA can only activate the element on "user action" (see User
    Activation; Activation).
* **Activation / Activatability**: this refers to the notion that an element
  that is locked can be unlocked by the UA. Some examples of this are
  find-in-page, or scrolling. Different rendersubtree tokens can control this
  behavior.
* **User Activation**: this refers to activation caused by user or developer
  actions. Examples of this include find-in-page, scrollIntoView(), tab
  navigation. Notably, this does *not* include UA activation when the element is
  scrolled into view.
* **Visibilty Activation**: this refers to activation caused by the element
  entering the visible region of the page.
* **content-size**: this refers to a complementary CSS feature called content-size,
  which allows the developer to specify a placeholder size while size
  containment is present (which `rendersubtree=invisible` puts into place).

## Current status (Chromium)

* **Activation**: As of [this patch](https://chromium-review.googlesource.com/c/chromium/src/+/1853854),
  activation is the default behavior, which can be disabled with
  `skip-activation` or `skip-visibility-activation`

* **Containment**: The presence of the `rendersubtree` attribute forces
  `contain: layout style;` *in addition* to any other containment. If the
  `invisible` token is present, it also forces `contain: size` in addition to
  layout, style and any other containment already present.

* **Intent to Experiment** proposed and approved ([thread](https://groups.google.com/a/chromium.org/d/msg/blink-dev/-6Cp2osHn50/VZhPCrXHDAAJ)

* **`content-size`** is implemented, but is likely to change in both syntax and
  semantics pending spec revision.
