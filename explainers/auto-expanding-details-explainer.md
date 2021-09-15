Auto-expanding `<details>` explainer

# The problem

Today, users can’t search for content inside `<details>` elements when they are closed. If the user wants to search the page for content inside `<details>` elements with find-in-page, then they have to manually expand every `<details>` element in the page before they use find-in-page.

# The solution

The solution is to make the hidden contents of closed details elements searchable by find-in-page and have the browser  automatically them when trying to scroll to their hidden contents.

In addition to find-in-page, this feature should also work for Element fragments (navigating to a hash with an element id) and [ScrollToTextFragment](https://github.com/WICG/scroll-to-text-fragment/issues/173) in order to make content hidden inside `<details>` elements more findable and shareable.
  
I have already [added this feature to the HTML spec](https://github.com/whatwg/html/pull/6466).

# Privacy concerns

I wrote a security and privacy self review [here](/privacy-assessments/auto-expanding-details-privacy.md). TL;DR this feature does not reveal any new sensitive information to the page due to the existing `scroll` events and `:target` psuedo selector.
