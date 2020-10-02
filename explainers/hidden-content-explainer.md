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
    target text. The event is fired at render timing on the nearest ancestor
    block-level element for these cases:
    * There is a new find-in-page (ctrl+f)
      [active match](https://html.spec.whatwg.org/multipage/interaction.html#fip-matches)
      which is located inside an element with `content-visibility:
      hidden-matchable` style.
    * There is a [scroll-to-text](https://github.com/WICG/ScrollToTextFragment)
      navigation (`example.com/#:~:text=foo`), where the target text is located
      inside a `content-visibility: hidden-matchable` element.

If the matching text spans multiple block-level elements, then the first
block-level element is used. Since the flat tree is used to determine the
element to fire the event on, shadow boundaries have no impact.

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
<style>
.collapsed {
  content-visibility: hidden-matchable;
}
.title-collapsed::before {
  content: '➡';
}
.title-open::before
  content: '⬇';
}
</style>

Please explore the following sections:
<div class=section>
  <h1 class="title-collapsed">Introduction</h1>
  <div class=collapsed>lorem ipsum ...</div>
</div>

<div class=section>
  <h1 class="title-collapsed">Thesis</h1>
  <div class=collapsed>dolor sit amet ...<div>
</div>

<div class=section>
  <h1 class="title-collapsed">Conclusion</h1>
  <div class=collapsed>consectetur adipiscing ...</div>
</div>

<script>
document.querySelectorAll('.section').forEach(section => {
  const title = section.querySelector('.title-collapsed');
  const hiddenContent = section.querySelector('.collapsed');
  section.addEventListener('beforematch', () => {
    hiddenContent.classList.remove('collapsed');
    title.classList.remove('title-collapsed');
    title.classList.add('title-open');
  });
});
</script>
```

![example1 1](images/example1-1.png)
![example1 2](images/example1-2.png)
![example1 3](images/example1-3.png)

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

2. If the page doesn't reveal the text in response to the beforematch event, the
   beforematch event will not be fired for the remainder of the lifetime of the
   document. By "reveal," I mean that the `content-visibility: hidden-matchable`
   property is removed and there are no other properties applied which would
   prevent find-in-page from finding the text normally, such as `display: none`.
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

The scrolling behavior exhibited by find-in-page and ScrollToTextFragment
after style and DOM changes are made by `beforematch` event
handlers is not well specified yet and is being discussed here: 
https://github.com/WICG/display-locking/issues/150

Here is a list of `beforematch` handler use cases that could affect scrolling:
1. **Removed from DOM**: The `beforematch` event handler removes the
target element from the DOM.
2. **Reparented in DOM**: The `beforematch` event handler removes the target
element from the DOM, and re-adds it further down the tree such that the
location to scroll to is different.
3. **display: none**: The `beforematch` event handler adds the style
`display: none` to the target element.
4. **visibility: hidden**: The `beforematch` event handler adds the style
`visibility: hidden` to the target element.
5. **content-visibility: hidden-matchable -> visible**: The target element has
the style `content-visibility: hidden-matchable` before the `beforematch` event
is fired on the target element, and the `beforematch` event handler changes the
style value from `hidden-matchable` to `visible`.


Here is the current scrolling behavior in Chromium for each of those cases in
ScrollToTextFragment, and ElementFragment, and find-in-page. As mentioned
before, this is still being discussed
[here](https://github.com/WICG/display-locking/issues/150) and is subject to
change.

#### ScrollToTextFragment
1. **Removed from DOM**: ScrollToTextFragment will not scroll to the removed
element. If there is a second match which is still in the DOM after
beforematch, it will be scrolled to but a second beforematch event will not be
fired.
2. **Reparented in DOM**: ScrollToTextFragment will scroll to the new
location of the target. If there is a second match which was not reparented,
ScrollToTextFragment will scroll to whichever comes first from the top of the
page after the beforematch event is fired.
3. **display: none**: ScrollToTextFragment will not scroll to the target
element. If there is a second match which was not modified, ScrollToTextFragment
will scroll to it.
4. **visibility: hidden**: ScrollToTextFragment will not scroll to the target
element. If there is a second match which was not modified, ScrollToTextFragment
will scroll to it.
5. **content-visibility: hidden-matchable -> visible**: ScrollToTextFragment
will scroll to the revealed text.

#### find-in-page
The find-in-page behavior has some issues in these cases. These will likely
be fixed after adding an async step to pick up layout changes made by the
beforematch event listener. Here is a bug to track this work:
https://bugs.chromium.org/p/chromium/issues/detail?id=1074121
1. **Removed from DOM**: find-in-page does not scroll to the target element. If
there is a second match in the DOM, find-in-page will find it and scroll to it
without needing additional user input.
2. **Reparented in DOM**: If the only match in the document is reparented to the
end of the document, find-in-page becomes stuck at "0/x" matches and can't
scroll to the new location, even if you keep typing more of the match out or
close and reopen find-in-page. This behavior is obviously not good, and
hopefully will be fixed by adding an async step to get updated layout
information. If there is a second match in the DOM which is not reparented,
the page will scroll but find-in-page will still get stuck.
3. **display: none**: find-in-page will scroll to the spot the element used to
take up. This should probably not scroll at all instead, and will hopefully
happen after adding an async step to get updated layout information. If there
is a second match, the page will scroll to the second match.
4. **visibility: hidden**: find-in-page will scroll to the spot the element used
to take up. This should probably not scroll at all instead, and hopefully won't
scroll after adding an async step.
5. **content-visibility: hidden-matchable -> visible**: find-in-page scrolls to
to the revealed text.

### Alternatives Considered
Given the purpose of displaying `content-visibility: hidden-matchable` text
when it is searched for, there are a number of alternatives we have considered.

#### Automatic Revealing
`content-visibility: hidden-matchable` text would automatically become visible
when searched for by adding an internal flag to the `hidden-matchable` element
saying that it has been revealed.
This used to be implemented as a prior iteration of this feature.
##### Pros
* Provides the desired behavior without the need for the web developer to do
  anything besides use the `hidden-matchable` value.
* Since there is no event causing script to run, the interaction and scrolling
  occurs entirely within the browser, which guarantees that we can scroll to
  the element without complications.
##### Cons
* Doesn't allow the developer to change other properties in conjunction with
  displaying hidden content. For example, in "Example 1: Expanding
  `hidden-matchable`," automatic revealing wouldn't be able to change the
  style of the `<h1>` title elements.
* Doesn't allow script to make the `hidden-matchable` content hidden again once
  it has been revealed since the flag saying that is has been revealed is
  internal to the browser. This doesn't allow for use cases where the
  `hidden-matchable` content looks similar to a details element with a button
  that runs script to reveal and collapse the content.
* Automatic revealing and adding internal hidden state to track revealed
  `hidden-matchable` sections gets complicated and confusing in the browser
  implementation.

#### Automatic Revealing without internal state
`content-visibility: hidden-matchable` text would automatically be changed
to `content-visibility: visible` by modifying `element.style` when text inside
it has been searched for.
##### Pros
* Same pros as "Automatic Revealing."
* Don't need to maintain internal state in the browser.
* If a developer knows how it works, they can change the style back to
  `content-visibility: hidden-matchable`.
##### Cons
* May require modifying `element.style` of multiple elements.
* If script later modifies `element.style`, then the matching text would become
  invisible again. In general, having the browser change DOM or style like this
  isn't a good idea because it would be likely to clash with how the page is
  maintaining the same state.

#### Automatic Revealing with activation event
This is like "Automatic Revealing," but with an added "activation" event emitted
when content is displayed to allow developers to change other state and styles
if needed.
##### Pros
* Same pros as "Automatic Revealing."
* Allows other script state and styles to be changed when content is displayed.
##### Cons
* Doesn't allow script to make the `hidden-matchable` content hidden again once
  it has been revealed since the flag saying that is has been revealed is
  internal to the browser. This doesn't allow for use cases where the
  `hidden-matchable` content looks similar to a details element with a button
  that runs script to reveal and collapse the content.
* Automatic revealing and adding internal hidden state to track revealed
  `hidden-matchable` sections gets complicated and confusing in the browser
  implementation.

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

#### `<details>`/`<summary>` auto-expanding
Instead of having a generic hidden-matchable property or a specialized
beforematch event, we could just slightly tweak the `<details>` element to make
collapsed content searchable. This addresses most of the use cases we have seen
for the beforematch event, and the details element already has a toggle event
which would act like the beforematch event. This could be implemented either by
adding an attribute to the details element saying that the collapsed content
should be searchable, or just by making it searchable as-is because find-in-page
is not specced.
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
##### Pros
* Requires less work and less stuff added to the web platform.
* Exposes less information about find-in-page to the page, which improves
  privacy.
* Lets the browser handle updating style and scrolling, so there would be no
  need to implement async scrolling to work around use cases with scrolling
  to expanded content.
##### Cons
* Doesn't handle every use case. Not every collapsed section of content has an
  associated persistent summary view, such as hidden parts of virtual scrollers.
* Rather than making a powerful primitive for the web platform like beforematch,
  this would provide a narrowly scoped complete feature to the web.

#### Only fire beforematch on a hidden-matchable element attribute
Instead of having `content-visibility: hidden-matchable` and a separate
`beforematch` event that fires whenever text is searched for, we could have
a `hidden-matchable` element attribute which would function like the
`content-visibility: hidden-matchable` css property. By having an element
attribute instead of a css property, we could have the browser reveal the
content and scroll on its own by removing the `hidden-matchable` attribute
instead of needing to have script to do so. We would also only fire
`beforematch` on the element with the `hidden-matchable` property as soon as it
is revealed, rather than firing the event on the nearest block-level element
every time any text is searched for.
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
* Exposes less information about find-in-page to the page, which improves
  privacy.
* Lets the browser handle updating style and scrolling, so there would be no
  need to implement async scrolling to work around use cases with scrolling
  to expanded content.
##### Cons
* Couples the `beforematch` event with `hidden-matchable`. Both of them would
  need to be shipped together, since `beforematch` would only fire on
  `hidden-matchable` elements.
* Doesn't allow for `beforematch` to be used for use cases outside of hidden
  content, but if it is determined that `beforematch` reveals too much
  information to the page then this would be necessary.
  Alternatively, we could still fire `beforematch` on everything we scroll to
  during find-in-page/ScrollToTextFragment and keep the `hidden-matchable` as
  an attribute idea which would still keep the Pros of not needing script for
  a big use case and letting the browser revealing and scrolling.
