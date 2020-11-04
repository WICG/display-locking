# beforematch event

_These changes will likely go into the
[find-in-page](https://html.spec.whatwg.org/multipage/interaction.html#find-in-page)
section of the HTML spec_

When a new _active match_ is set, either by advancing through the match list or
due to a new find-in-page request, a `beforematch` event is fired on a node
identified by the _active match_'s [range
start](https://dom.spec.whatwg.org/#concept-range-start). This process follows
the following algorithm:

1. Identify the candidate match which will become the new _active match_. The
   _active match_'s DOM range must not be
   [collapsed](https://dom.spec.whatwg.org/#range-collapsed). 

2. At the next rendering opportunity, specifically step 12 "run the animation
   frame callbacks" of
   [update-the-rendering](https://html.spec.whatwg.org/multipage/webappapis.html#update-the-rendering):

      2.1. If the DOM range representing the _active match_ has been collapsed,
        start the search over again starting at this collapsed range.

      2.2. Let "_matchable ancestor_" be the nearest flat-tree ancestor element of
        the beginning of the DOM range representing the _active match_ which has
        the content-visibility: hidden-matchable property.

      2.3. If the beforematch event has been disabled or there is no _matchable
        ancestor_, run
        [scroll into view](https://drafts.csswg.org/cssom-view/#scroll-an-element-into-view)
        on the _active match_ and end the algorithm.

      2.4. Fire the beforematch event on the _matchable ancestor_.

      2.5. Signal a need for another run of update-the-rendering.

3. At the next rendering opportunity (the _next_ Step 12 of
   [update-the-rendering](https://html.spec.whatwg.org/#update-the-rendering)):

      3.1. Disable the beforematch event for the remaining lifetime of the
        document and start the search over again starting at the end of the
        _active match_'s DOM range if any of the following conditions are true:
        - The DOM range representing the _active match_ has been collapsed.
        - The _active match_ has the `content-visibility: hidden-matchable` or
          `content-visibility: hidden` in any ancestors.
        - The _active match_ has the `display: none` property in any ancestors.
        - The _active match_'s `visibility` property is not `visible` in any
          ancestors.

      3.2. Run
        [scroll into view](https://drafts.csswg.org/cssom-view/#scroll-an-element-into-view)
        on the _active match_.

# content-visibility: hidden-matchable

_These changes will likely go into the
[css-contain-2](https://www.w3.org/TR/css-contain-2/#content-visibility)
module_.

'hidden-matchable': 

  This value behaves very similarly to 'hidden'.
  That is, the element [skips its contents](https://www.w3.org/TR/css-contain-2/#skips-its-contents).

  The skipped contents must not be accessible to user-agent features such as
  tab-order navigation, nor be selectable or focusable.

  However, unlike 'hidden', the skipped contents must be accessible to the
  find-in-page algorithm in order to allow the `beforematch` event to fire.
  (TODO: reference the find-in-page beforematch spec)
