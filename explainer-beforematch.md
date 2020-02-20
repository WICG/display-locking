# The `beforematch` event

### Summary (TL;DR)

The `beforematch` event allows developers to display content to the user in
response to to find-in-page or fragment navigation actions. It is fired just
before these actions, and at render timing. The applicable actions are the
following:

* There is a new find-in-page active match, in which case the event is fired
  with the matched element as the target.
* There is a fragment-link navigation, in which case the event is fired with
  the matched fragment as the target.
* There is a scroll-to-text navigation, in which case the event is fired with
  the matched element as the target.

Note that the 'matched element' in this document refer to one of the following:
* the element that contains the text which was matched by find-in-page or
  scroll-to-text navigation.
* the element which was selected for the fragment link navigation.

The use case for this event is to allow developers to help users find hidden
content on the page. Hidden content includes collapsed sections via clipping,
`visibility: hidden` subtrees, and [`subtree-visibility:
hidden-matchable`](https://github.com/WICG/display-locking) subtrees.

### Motivation

With the evolution of the web, there are always new and interesting ways that
developers choose to organize the information on their pages. Some of these
approaches (e.g. the common case of text scrolling), lend themselves naturally to
user-agent features like find-in-page. This is not an accident, since
find-in-page was designed with common use-cases in mind.

However, other approaches like collapsed sections of text do not work well with
user-agent features since the page does not get any indication that the user
initiated a find-in-page request, fragment navigation, or scroll-to-text
navigation.

The `beforematch` event is a step in the direction that allows developers to
leverage information that the user-agent already has to make these search and
navigation experiences great. Specifically, with [hidden but
matchable](#footnotes) content, it will be possible to process text for
find-in-page match in sections that are not visible. In turn, the `beforematch`
event will be fired on hidden (a.k.a. collapsed) sections, allowing the
developer to unhide the section. The net effect is that the user is able to use
find-in-page to find content in a collapsed section -- something that is not
currently possible.

Even without hidden but matchable features, `beforematch` is useful signal to the
page which allows custom styling of the matched element, which is now only
possible with approximations from scroll positions and intersection
observations.

### Use Cases

#### Example 1
```html
<style>
.collapsed {
  subtree-visibility: hidden-matchable;
}
</style>

Please explore the following sections:
<h1>Introduction</h1> <div class=collapsed>lorem ipsum ...</div>
<h1>Thesis</h1>       <div class=collapsed>dolor sit amet ...<div>
<h1>Conclusion</h1>   <div class=collapsed>consectetur adipiscing ...</div>

<script>
function expand(e) {
  e.target.classList.remove("collapsed");
}

document.querySelectorAll(".collapsed").forEach(item => {
  item.addEventListener("beforematch", expand);
});
</script>
```

In this example, most of the content of the page is hidden in collapsed section.
It uses the upcoming `subtree-visibility` CSS property to hide the
content while letting it be searchable by find-in-page. When a match is found,
and `beforematch` event is fired, we expand the section by removing the
collapsed class.

Note that the net effect of this is that the user is able to find matches in
collapsed sections which are automatically expanded. This is possible due to
both hidden-but-matchable content and the `beforematch` event.

The same effect occurs when doing a fragment link navigation to any element
within the collapsed section. Likewise, scroll-to-text navigation would expand
relevant sections. For instance, navigating to
`example.com/page.html#:~:text=lorem` would expand the introduction section by
observing the `beforematch` event for that section as a result of the
scroll-to-text fragment navigation match.

Also note that the adoption of the `beforematch` in these use-cases should be
straight-forward, since the typical page that provides content in collapsed
sections already has an event handler to expand and show the section. With the
`beforematch` event, we can reuse the same handler to expand the section.

#### Example 2

```html
<style>
.section {
  transition: background 2s;
}
.highlighted {
  background: cornsilk;
  transition: background 0s;
}
</style>

<div class=section>Lorem ipsum dolor sit amet</div>
<div class=section>consectetur adipiscing elit</div>
<div class=section>Sed augue lacus</div>

<script>
function highlight(e) {
  e.target.classList.add("highlighted");
  requestAnimationFrame(
    () => requestAnimationFrame(
      () => e.target.classList.remove("highlighted")));
}

document.querySelectorAll(".section").forEach(item => {
  item.addEventListener("beforematch", highlight);
});
</script>
```

In this example, a highlighted entry gains a corksilk background color, which
fades over approximately two seconds. It's a subtle effect that grabs the user's
attention but is not too intrusive. 

We use a `beforematch` event to highlight the section. Note that the highlighted
element is the target element of the `beforematch` event, but we can also modify
any related style or DOM based on the target's location.

The effect of this example is that the section is briefly highlighted in
cornsilk on an active find-in-page match. This effect happens in addition to the
user-agent highlight the found match.

### Privacy Concerns

Note that this event exposes more information to the page that would otherwise
be available. In particular, the page can know which section of text was found
using find-in-page, fragment navigation, and scroll-to-text navigation.

The developer may also be able to find out some information about what the user
typed into a find-in-page dialog based on which elements receive an event.
Likewise, for scroll-to-text navigation, the developer may be able to learn more
about the content of the fragment search terms based on which section of the
page receives an event.

We believe that the benefit of providing this information to the site outweighs
the potential risks. Moreover, the lower granularity information can already be
approximated from scroll position and intersection observerations on the page.

The privacy aspect of the event should be discussed in more detail.

### Footnotes

**Hidden but matchable** content refers to an idea that although some content
may be hidden from the user, it may still be useful to have that content be
searchable.  One such approach is [`subtree-visibility:
hidden-matchable`](https://github.com/WICG/display-locking), but there may be
other proposed approaches for similar functionality.
