WORK IN PROGRESS.

Display Locking is an imperative, simple-to-adopt API that allows web developer to help the browser improve performance, without sacrificing the declarative ergonomics of HTML and DOM.

* Display Locking is a form of progressive performance enhancement. Pages can still work if the browser doesn't support Display Locking, and and browsers with Display Locking can start simple and incrementally implement more powerful optimizations behind the scenes. A first-cut implementation of Display Locking should not be very hard for today's browsers to implement. In this sense, Display Locking is an imperative form of [CSS containment](https://developer.mozilla.org/en-US/docs/Web/CSS/contain).

* Display Locking is a primitive that unblocks new [async DOM](https://github.com/chrishtr/async-dom/blob/master/why-async-dom.md) rendering architectures and approaches. Some of these are discussed in the sections below, from the developer and browser implementor point of view.

* Display Locking is an [extensibility primitive](https://www.w3.org/community/nextweb/2013/06/11/the-extensible-web-manifesto/). It allows developers to provide their own performance isolation points, optimize rendering for UI patterns other than scrolling, and improve reliability across browsers.

*** Developer perspective

From the developer point of view, today there is not a lot that can be done to optimize when rendering happens, and how much it costs, other than following some best practices, and reducing the size of the DOM and CSS. [CSS containment](https://developer.mozilla.org/en-US/docs/Web/CSS/contain) exists, but may or may not result in better performance. Furthermore, there is no way to incrementally modify DOM that is connected to the document in a yielding manner, without visual artifacts.


From the developer side, this includes approaches such as [DOM Changelists](https://github.com/whatwg/dom/issues/270) or [Worker DOM](https://github.com/ampproject/worker-dom). 
