# Display locking

<div style="background: #ffffcc; border: 1px solid #ccc; padding: 20px;">
Note that this feature should not be confused with the
<a href="https://github.com/w3c/screen-wake-lock/">Screen Wake Lock API</a>.
</div>

## Introduction

Display Locking is an umbrella term of related features designed primarily to
allow developers to increase performance of their sites. In addition, some of
the features in the display locking project enable site behaviors that were not
previously easy to accomplish (e.g. searchability in collapsed sections).

This document provides an overview of the features under the Display Locking
projects with links that provide additional information.

## Features

### `content-visibility`

#### Summary

`content-visibility` is a CSS property designed to allow developers and browsers
to easily scale to large amount of content and control when rendering work
happens.

#### Status

The spec draft for the feature is in a mature state.

The feature is currently implemented in Chromium M85 behind the
`--enable-blink-features=CSSContentVisibility` flag.

#### Additional information

* [spec draft](https://drafts.csswg.org/css-contain-2/#content-visibility)
* [explainer](https://github.com/WICG/display-locking/blob/master/explainers/content-visibility.md)
* [TAG review](https://github.com/w3ctag/design-reviews/issues/306)

### `content-visibility: hidden-matchable`

#### Summary

`content-visibility: hidden-matchable` is an additional value for the
`content-visibility` property that allows user-agents to search for find-in-page
matches inside an otherwise hidden element. This feature is meant to be combined
with the `beforematch` event to allow searchability into collapsed sections
which expand if there is a match found.

#### Status

This feature depends on both the successful adoption of `content-visibility`
feature as well as the `beforematch` event. The explainer outlines the target
use-cases, but neither the spec draft nor the TAG review are available at this
time.

This is currently implemented in Chromium M85 behind the
`--enable-blink-features=CSSContentVisibilityHiddenMatchable` flag.

#### Additional information

* spec draft — not yet available.
* [explainer](https://github.com/WICG/display-locking/blob/master/explainers/content-visibility-hidden-matchable.md)
* TAG review — not yet available.

### `contain-intrinsic-size`

#### Summary

`containt-intrinsic-size` is a CSS property that allows the developer to specify
an intrinsic size to use when size-containment is specified. This enables the
developers to specify "placeholder size" on content which is meant to be sized
by intrinsic sizing, but has size containment applied to it.

#### Status

The spec draft for the feature is in a mature state.

The feature is currently implemented and shipped in Chromium M83.

#### Additional information

* [spec draft](https://drafts.csswg.org/css-sizing-4/#intrinsic-size-override)
* [explainer](https://github.com/WICG/display-locking/blob/master/explainers/contain-intrinsic-size.md)
* [TAG review](https://github.com/w3ctag/design-reviews/issues/437)
* [privacy assessment](https://github.com/WICG/display-locking/blob/master/privacy-assessments/contain-intrinsic-size.md)

### `beforematch`

#### Summary

`beforematch` is an event which is fired under one of the following conditions:

* There has been a find-in-page request with the matching text found inside the
    target element.
* There has been a scroll-to-text navigation with the text being found inside
    target element.
* There has been a fragment link navigation and the target element was the
    fragment target.

This allows pages to react to users searching for content on their sites. This
is particularly useful in combination with `content-visibility: hidden-matchable`
as it allows searchability inside otherwise hidden content.

#### Status

The exact requirements and the design of this feature is still being discussed.

The feature is under development in Chromium M85 and is available behind the
`--enable-blink-features=BeforeMatchEvent` flag.

#### Additional information

* spec draft — not yet available
* [explainer](https://github.com/WICG/display-locking/blob/master/explainers/beforematch.md)
* [TAG review](https://github.com/w3ctag/design-reviews/issues/511)
* [privacy assessment](https://github.com/WICG/display-locking/blob/master/privacy-assessments/beforematch.md)

## Disclaimer

As the proposed features evolve, several competing API shapes might be
considered at the same time, the decisions on particular behaviors might not be
finalized, and some documentation may be out of date.

This document was last updated on May 28, 2020.
