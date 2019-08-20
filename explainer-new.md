Web developers need ways to reduce loading and rendering time of web apps that have a lot of DOM. Two common techniques are to mark non-visible DOM as invisible [1], or to use virtualization [2]. Browser implementors also want to reduce loading and rendering time of web apps. Common techniques to do so include adding caching of rendering state [3], and avoiding rendering work [4] for content that is not visible.

These techniques can work in many cases but have some drawbacks and limitations. These include:
a. [1] and [2] usually  means that such content is not available to [user-agent APIs](https://github.com/WICG/display-locking/blob/master/user-agent-apis.md). Also, content that is placed offscreen may or may not have rendering cost, which makes the heuristic unreliable.
b. Caching intermediate rendering state is [hard work](https://martinfowler.com/bliki/TwoHardThings.html), and often has performance limitations and cliffs that are not obvious to developers. Similarly, relying on the browser to avoid rendering for content that is clipped out or not visible is not reliable, as it's hard to efficiently detect what content is actually visible.

Previously adopted web APIs, in particular the [contain](https://developer.mozilla.org/en-US/docs/Web/CSS/contain) and [will-change](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change) CSS properties, add ways to specify forms of rendering isolation


[1] Examples: placing `display:none` CSS on DOM subtrees, or by placing content far offscreen.
[2] In this context, virtualization means representing content outside of the DOM, and inserting it into the DOM only when visible. This is most commonly used for virtual or infinite scrollers.
[3] Examples: caching the computed style of DOM elements, the output of text / block layout, and display list output of paint.
[4] Examples: detecting DOM that is clipped out by ancestor content, or not visible in the viewport, and avoiding some or most rendering [lifecycle phases](https://github.com/WICG/display-locking/blob/master/lifecycle.md) for them.
