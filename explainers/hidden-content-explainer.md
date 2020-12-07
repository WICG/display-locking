# Finding information in hidden sections.

## content-visibility: hidden-matchable and the `beforematch` event.

### Summary (TL;DR)

This is an explainer for two closely related features:

1. Content visibility hidden matchable is an extension to the `content-visibility`
    feature. It adds a new value to the possible set of properties:
    * `content-visibility: hidden-matchable`: in this configuration, the content is
        hidden from the user (similar to `content-visibility: hidden`), but the
        content remains searchable via user-agent find-in-page algorithms.

2. The `beforematch` event allows developers to display `hidden-matchable`
    content to the user in response to searches which scroll the page to some
    target text. The event is fired on the nearest `content-visibility:
    hidden-matchable` ancestor at render timing for these cases:
    * There is a new find-in-page
      [active match](https://html.spec.whatwg.org/multipage/interaction.html#fip-matches)
      which is located inside an element with `content-visibility:
      hidden-matchable` style.
    * There is a [scroll-to-text](https://github.com/WICG/ScrollToTextFragment)
      navigation (`example.com/#:~:text=foo`), where the target text is located
      inside a `content-visibility: hidden-matchable` element.

If the matching text spans multiple `content-visibility: hidden-matchable`
ancestors, the beforematch event will be fired on the first one. Since the flat
tree is used to determine the element to fire the event on, shadow boundaries
have no impact.

Note that the proposal for beforematch is still in review and is subject to
change.

### Motivation

With the evolution of the web, there are always new and interesting ways that
developers choose to organize the information on their pages. Some of these
approaches (e.g. the common case of text scrolling), lend themselves naturally to
user-agent features like find-in-page. This is not an accident, since
find-in-page was designed with common use-cases in mind.

However, other approaches like collapsed sections of text do not work well with
user-agent features since the page does not get any indication that the user
initiated a find-in-page request, or scroll-to-text navigation.

The content-visibility: hidden-matchable in concert with the `beforematch` event
is a step in the direction that allows developers to leverage information that
the user-agent already has to make these search and navigation experiences
great. Specifically, it makes it possible to process text for find-in-page match
in sections that are not visible. In turn, the `beforematch` event will be fired
on hidden (a.k.a. collapsed) sections, allowing the developer to unhide the
section. The net effect is that the user is able to use find-in-page or link
navigation to find content in collapsed sections -- something that is not
currently possible.

### Primary Use Case: collapsed searchable sections
```html
<!DOCTYPE html>
<meta charset="utf-8">

<style>
.title {
  cursor: pointer;
}
.title::before {
  content: '⬇️ ';
}
.collapsed > .title::before {
  content: '➡️ ';
}

.details {
  margin-left: 20px;
}
.collapsed > .details {
  content-visibility: hidden-matchable;
}
</style>

Please explore the following sections:
<div class="section collapsed">
  <h2 class=title>Introduction</h1>
  <div class=details>lorem ipsum ...</div>
</div>

<div class="section collapsed">
  <h2 class=title>Thesis</h1>
  <div class=details>dolor sit amet ...</div>
</div>

<div class="section collapsed">
  <h2 class=title>Conclusion</h1>
  <div class=details>consectetur adipiscing ...</div>
</div>

<script>
document.querySelectorAll('.section').forEach(section => {
  section.onbeforematch = () => {
    section.classList.remove('collapsed');
  };
  section.querySelector('.title').onclick = () => {
    section.classList.toggle('collapsed');
  };
});
</script>
```

![beforematch](images/beforematch.gif)

As you can see in the above gif, the flow of the use case is as follows:
1. The page loads and all subsections are hidden with only the headings visible,
similar to a `<details>` element.
2. User searches for hidden text, such as "lorem ipsum".
3. The beforematch event is fired on the element containing "lorem ipsum."
4. User may click on the section to collapse it again.

In this example, most of the content of the page is hidden in collapsed sections.
It uses the `content-visibility: hidden-matchable` CSS property to hide the
content while letting it be searchable by find-in-page. When a match is found,
and `beforematch` event is fired, we expand the section by removing the
collapsed class.

Note that the net effect of this is that the user is able to find matches in
collapsed sections which are automatically expanded. This is possible due to
both hidden-matchable content and the `beforematch` event.

The same effect occurs when scroll-to-text navigation targets text in a hidden
section. For instance, navigating to
`example.com/page.html#:~:text=lorem` would expand the introduction section by
observing the `beforematch` event for that section as a result of the
scroll-to-text fragment navigation match.

One possible real-world application of this is the collapsed sections on mobile
wikipedia pages. find-in-page and scroll-to-text currently can't find text inside
of the collapsed sections, but with beforematch and content-visibility they
could be searchable and automatically expanded.

Also note that developer adoption of the `beforematch` event in these use-cases should be
straight-forward, since the typical page that provides content in collapsed
sections already has an event handler to expand and show the section. With the
`beforematch` event, we can reuse the same handler to expand the section.

### Privacy Concerns

The beforematch event could expose more information to the page than is
currently exposed because it is fired on an element that contains the text the
user is searching for.

Note that it is already possible to snoop on find-in-page by creating a
scrollable area containing every next character the user could type into
find-in-page, listening to scroll events to see which caracter the user typed
in, then prepending the new character to all of the next possible search terms.
If the scrollable area is 1px tall or otherwise very small or hard to see, then
the user may not be able to tell it is happening. This is demonstrated in
[search-incremental.html](/resources/find-in-page/search-incremental.html).

In order to mitigate the additional information that beforematch could expose to
the page, we have added two constraints to beforematch:

1. The beforematch event will only be fired on content in the DOM subtree of an
   element with `content-visibility: hidden-matchable`. This removes the
   information delta of find-in-page on normal pages without hidden find-in-page
   snooping. Without this constraint, a page could listen to beforematch events
   and get some idea of what you're searching for within the text already in the
   page. Although a page could already try to do this by listening to scroll
   events and guessing which element was scrolled to, the beforematch event
   without this restriction is a stronger and more accurate signal.

2. If the page fails to reveal the active match when the browser tries to scroll
   to the active match after firing the beforematch event, the beforematch event
   will be disabled for the remaining lifetime of the document. By "reveal," I
   mean that the active match fulfills all of these requirements:
   - The DOM range of the target match is not
     [collapsed](https://dom.spec.whatwg.org/#range-collapsed), meaning that the
     target match was not removed from the DOM.
   - Neither the match nor any of its ancestors have `content-visibility:
     hidden` or `content-visibility: hidden-matchable`.
   - Neither the match nor the any of its ancestors have `display: none`.
   - The `visibility` CSS property is `visibility: visible`.
   This mitigation will prevent the page from building out a string of what the
   user is searching for without having to reveal that string to the user
   visually. Although this is already possible as shown in 
   [search-incremental.html](/resources/find-in-page/search-incremental.html),
   firing beforematch on hidden content that stays hidden would be harder for
   the user to notice.

As for the ScrollToTextFragment case, the page can already guess with reasonable
certainty based on the scroll offset which text is already highlighted. In
addition, ScrollToTextFragment can only occur once at the beginning of a
document's load, so we don't have to worry about the page building out a search
term since there will only be one possible beforematch event.

### Responses to DOM and style changes in the `beforematch` event handler

The beforematch event handler, as well as any other script that runs during the
async steps before we actually scroll to the target match, affects the outcome
of the scroll. As mentioned in the privacy section, if the DOM range
representing the active match is collapsed or it has a style which makes it
invisible, the scroll will be canceled. Otherwise, the target match will be
scrolled into view.

### Alternatives Considered
Given the purpose of displaying `content-visibility: hidden-matchable` text
when it is searched for, there are a number of alternatives we have considered.

#### Automatic Revealing
`content-visibility: hidden-matchable` text would automatically become visible
when searched for by adding an internal flag to the `hidden-matchable` element
saying that it has been revealed.
This used to be implemented in Blink as a prior iteration of this feature.
##### Pros
* The browser reveals the content and scrolls to it without the need for any
  script.
* Since there is no event causing script to run, the interaction and scrolling
  occurs entirely within the browser, which guarantees that we can scroll to
  the element without complications.
* Less privacy concerns since the browser doesn't explicitly signal new
  information to the page when a match is found or revealed.
##### Cons
* Doesn't allow the page to change other state in conjunction with displaying
  hidden content. For example, the html example earlier in this explainer uses
  the beforematch event to change the arrow in the clickable title section which
  expand and collapses the section to show whether or not the section is
  expanded or collapsed. This is a very common pattern, and without the page
  being notified about the reveal, it isn't possible.
* Doesn't make it feasible for script to toggle the expanded/collapsed state
  since script can't see the internal flag representing the expanded/collapsed
  state.
* Automatic revealing and adding internal hidden state to track revealed
  `hidden-matchable` sections gets complicated and confusing in the browser
  implementation.

#### Automatic Revealing with `element.style`
`content-visibility: hidden-matchable` text would automatically be changed
to `content-visibility: visible` by modifying `element.style` when text inside
it has been searched for.
##### Pros
* The browser reveals the content and scrolls to it without the need for any
  script.
* Since there is no event causing script to run, the interaction and scrolling
  occurs entirely within the browser, which guarantees that we can scroll to
  the element without complications.
* Don't need to maintain internal state in the browser.
* If a developer knows how it works, they can change the style back to
  `content-visibility: hidden-matchable`.
##### Cons
* May require modifying `element.style` of multiple elements.
* If script later modifies `element.style`, then the matching text would become
  invisible again. In general, having the browser change DOM or style like this
  isn't a good idea because it would be likely to clash with how the page is
  maintaining the same state and isn't very intuitive to the developer.
* Doesn't make it easy for the page to change other state in conjunction with
  the reveal.
* Requires privacy mitigations since the reveal/match is observable by the page.

#### Automatic Revealing with activation event
This is like "Automatic Revealing," but with an added "activation" event emitted
when content is revealed to allow the page to change other state and styles if
needed.
##### Pros
* The browser reveals the content and scrolls to it without the need for any
  script.
* Allows script to modify state and style when content is revealed.
##### Cons
* Doesn't make it feasible for script to toggle the expanded/collapsed state
  since script can't see the internal flag representing the expanded/collapsed
  state.
* Automatic revealing and adding internal hidden state to track revealed
  `hidden-matchable` sections gets complicated and confusing in the browser
  implementation.
* Requires privacy mitigations since the reveal/match is observable by the page.

#### CSS Pseudo Selector
A pseudo selector, such as `:target`, would be applied to the element
containing the matching text when it is searched for. This pseudo selector
could be applied to the entire ancestor chain.
##### Pros
* Allows content to become visible when searched for with only CSS.
* Allows other styles to be changed when the content is displayed.
##### Cons
* If CSS with a pseudo selector is used to make text visible, then when
  find-in-page is closed or the search text changes, the pseudo selector would
  be removed and then any selector which is displaying the text based on that
  pseudo selector would not apply, causing the expanded section to unexpectedly
  collapse.
* There is no way in CSS to say "change my style if a descendant has a pseudo
  class on it." For this reason, developers would be unable to change styles
  outside of the particular element that has the matching text, which would
  make the functionality of "Example 1: Expanding `hidden-matchable`" not
  possible. Although this could be mitigated for some cases by applying the
  pseudo selector to the entire ancestor chain, it can be complicated or
  impossible to provide the right selector which can modify a style on an
  unrelated element.
* Requires privacy mitigations since the reveal/match is observable by the page.
* Harder to add privacy mitigations. Making the presence of the persistent
  pseudo selector based on the presence of `content-visibility:
  hidden-matchable` in the ancestor chain is more complicated than simply not
  firing the beforematch event in some situations.
* Not elegant or not even possible for script to listen for the reveal and
  change other state in the page.

#### `<details>`/`<summary>` auto-expanding
Instead of having a generic hidden-matchable property or a specialized
beforematch event, we could just slightly tweak the `<details>` element to make
collapsed content searchable. This addresses most of the use cases we have seen
for the beforematch event, and the details element already has a toggle event
which would act like the beforematch event. This could be implemented either by
adding an attribute to the details element saying that the collapsed content
should be searchable, or just by making it searchable as-is because the
find-in-page spec does not specify this behavior.
```html
<details searchable id=mydetails>
  <summary>Persistent summary content</summary>
  Hidden searchable details content
</details>
<script>
  mydetails.addEventListener('toggle', () => {
    // change other state, fancy animation, etc...
  });
</script>
```
We may still add this feature to the details element separately from the
beforematch event.
##### Pros
* Requires less to be added to the web platform.
* Exposes less information about find-in-page to the page, which improves
  privacy.
* No script needed to get the basic revealing behavior.
##### Cons
* Doesn't handle every use case. Not every collapsed section of content has an
  associated persistent summary view, such as hidden parts of virtual scrollers.
* Rather than making a powerful primitive for the web platform like beforematch,
  this would provide a narrowly scoped complete feature to the web.
* May require some small privacy mitigations.

#### Make hidden-matchable an element attribute instead of a CSS property
Instead of having `content-visibility: hidden-matchable` and a separate
`beforematch` event that fires whenever text is searched for, we could have
a `hidden-matchable` element attribute which would function like the
`content-visibility: hidden-matchable` css property. By having an element
attribute instead of a css property, we could have the browser reveal the
content by removing the script observable attribute as well as fire the
beforematch event.
```html
<div hidden-matchable id=mydiv>
  hidden searchable content
</div>
<script>
  mydiv.addEventListener('beforematch', () => {
    // change other state, fancy animation, etc...
  });
</script>
```
##### Pros
* Simple use cases won't require any script to reveal content.
* Lets the browser handle updating style and scrolling, so there would be no
  need to implement async scrolling to work around use cases with scrolling
  to expanded content.
##### Cons
* It makes more sense for visibility to be controlled by CSS than an element
  attribute given that the `display`, `visibility`, and `content-visibility`
  CSS properties all control visibility.
* Requires privacy mitigations since the reveal/match is observable by the page.
* It's harder to apply an attribute to a lot of content at once than it is to
  apply a CSS property to a lot of content at once.
* If you have a custom element which wants to apply hidden-matchable to its
  light DOM children, you can easily do so with a CSS selector which doesn't
  actually modify the state of the light DOM children. With this attribute, you
  would have to modify the light DOM children.
