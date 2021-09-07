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

The feature has been accepted by CSS Working Group, and is a part of the
[css-contain-2]() module.

The feature is implemented in Chromium M85.

#### Additional information

* [spec](https://www.w3.org/TR/css-contain-2/#content-visibility)
* [explainer](https://github.com/WICG/display-locking/blob/master/explainers/content-visibility.md)
* [TAG review](https://github.com/w3ctag/design-reviews/issues/306)
* [article](web.dev/content-visibility)

### `hidden=until-found` and searchable `details` element.

#### Summary

Leveraging content-visibility, we can also support searchable hidden content.
We are applying this automatically to the details elements, to make the contents
of the details element available to find-in-page.

We are also adding a new attribute `hidden=until-found` to allow developers to create
hidden, but searchable, content.

#### Status

Searchable `details` element is available in Chromium behind the
`--enable-blink-features=AutoExpandDetailsElement` flag.

* [spec PR](https://github.com/whatwg/html/pull/6466)

`hidden=until-found` is currently being implemented.

### `contain-intrinsic-size`

#### Summary

`contain-intrinsic-size` is a CSS property that allows the developer to specify
an intrinsic size to use when size-containment is specified. This enables the
developers to specify “placeholder size” on content which is meant to be sized
by intrinsic sizing, but has size containment applied to it.

#### Status

The feature is currently implemented and shipped in Chromium M83.

#### Additional information

* [spec](https://www.w3.org/TR/css-sizing-4/#intrinsic-size-override)
* [explainer](https://github.com/WICG/display-locking/blob/master/explainers/contain-intrinsic-size.md)
* [TAG review](https://github.com/w3ctag/design-reviews/issues/437)
* [privacy assessment](https://github.com/WICG/display-locking/blob/master/privacy-assessments/contain-intrinsic-size.md)


### `updateRendering()`

#### Summary

`updateRendering` is a JavaScript API that allows asynchronous rendering updates
within a DOM subtree. It also has a attribute implementation which lets the
user-agent make the decision of when to asynchronously update the annotated
element subtrees.

#### Status

This feature is in active development.

[discussion & open questions](https://github.com/WICG/display-locking/blob/master/explainers/update-rendering.md)

## Disclaimer

As the proposed features evolve, several competing API shapes might be
considered at the same time, the decisions on particular behaviors might not be
finalized, and some documentation may be out of date.

This document was last updated on May 28, 2020.
