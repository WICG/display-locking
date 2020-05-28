## content-visibility: hidden-matchable.

Content visibility hidden matchable is an extension to the `content-visibility`
feature. It adds a new value to the possible set of properties:
* `content-visibility: hidden-matchable`: in this configuration, the content is
    hidden from the user (similar to `content-visibility: hidden`), but the
    content remains searchable via user-agent find-in-page algorithm.

Possible use case is Deep links and searchability into pages with hidden content
(example: mobile Wikipedia; scroll-to-text support for collapsed sections)


The property applies `contain: style layout size paint`. The element with this
property is invisible to rendering/ hit testing, but is visible to US algorithms
such as find-in-page. When the match is found, `beforematch` event will fire,
but the content will not automatically display. If it is the intent of the site
to show the matched content, then beforematch event handler has to make the
content visible.

### Examples

```html
<style>
.locked {
  content-visibility: hidden-matchable;
}
</style>

<div class=locked>
  ... some content goes here ...
</div>
```

Like with `content-visibility: hidden`, the rendering of the subtree is managed
by the developer.  However, it allows find-in-page to search for text within the
subtree and fire the activation signal if the active match is found. 

One intended use-case for this configuration is that the subtree is hidden and
"collapsed" (note the absense of `contain-intrinsic-size` which makes size
containment use empty size for intrinsic sizing). This is common when content is
paginated and the developer allows the user to expand certain sections with
button clicks. In the `content-visibility` case the developer may also listen to
the activation event and start rendering the subtree when the event targets the
element in the subtree. This means that find-in-page is able to expand an
otherwise collapsed section when it finds a match.

Another intended use-case is a developer-managed virtual scroller.
