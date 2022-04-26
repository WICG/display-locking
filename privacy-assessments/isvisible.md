# Element.isVisible Privacy and Security Self-Review.

* What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

This function exposes information that is already available to the page, and is
retrievable by other means. It is being exposed, because it is difficult to
efficiently and correctly compute this value. The function ss provided as a
useful convenience. 

* Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes.

* How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

The feature does not deal with such information.

* How do the features in your specification deal with sensitive information?

The feature does not deal with such information.

* Do the features in your specification introduce new state for an origin that persists across browsing sessions?

No.

* Do the features in your specification expose information about the underlying platform to origins?

No.

* Does this specification allow an origin to send data to the underlying platform?

No.

* Do features in this specification enable access to device sensors?

No.

* Do features in this specification enable new script execution/loading mechanisms?

No.

* Do features in this specification allow an origin to access other devices?

No.

* Do features in this specification allow an origin some measure of control over a user agent's native UI?

No.

* What temporary identifiers do the features in this specification create or expose to the web?

None.

* How does this specification distinguish between behavior in first-party and third-party contexts?

This feature does not make a distinction between contexts, it acts on DOM state
already available to the page.

* How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

This feature does not make a distinction between regular or private browsing
contexts.

* Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

No. I don't believe this is necessary, since this is function that determines
its result by functionality already available in other APIs

* Do features in your specification enable origins to downgrade default security protections?

No.

* How does your feature handle non-"fully active" documents?

This feature does not make a dinstinction between fully active or non-fully
active documents.

* What should this questionnaire have asked?

It asked all the necessary questions.
