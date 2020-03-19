# beforematch event security and privacy self-review questionnaire

The `beforematch` event ([explainer](https://github.com/WICG/display-locking/blob/master/explainer-beforematch.md)) is fired on elements in the following cases, before the scrolling occurs:
* When the fragment is changed to scroll to an element by id, including on navigation to a url which includes a fragment.
  * I will refer to this case as "ElementFragment."
* When the [ScrollToTextFragment](https://github.com/WICG/ScrollToTextFragment) feature finds text and scrolls to it.
  * I will refer to this case as "ScrollToTextFragment."
* When the find-in-page (ctrl+f) feature is used to search for text.
  * I will refer to this case as "find-in-page."

## Questions to Consider

### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

This feature may expose hints as to what the user is searching for to webpages. I wrote more detailed considerations about this in the "Privacy Considerations" section at the bottom of this document.

This exposure is necessary in order to fulfill the use case of displaying content that was searched for, as described in [the beforematch explainer](https://github.com/WICG/display-locking/blob/master/explainer-beforematch.md).

### 2.2. Is this specification exposing the minimum amount of information necessary to power the feature?

Yes, this is exposing only the information required for the `beforematch` event to function as specified.

### 2.3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?

No new information stored on the device is exposed by this feature.

If the user enters this type of information into their device by using the ScrollToTextFragment or find-in-page features, then this feature may expose it as described in the "Privacy Considerations" section.

### 2.4. How does this specification deal with sensitive information?

The search queries present in the ScrollToTextFragment and find-in-page features may be sensitive information, and are considered in the "Privacy Considerations" section.

### 2.5. Does this specification introduce new state for an origin that persists across browsing sessions?

This feature does not store any new data in the browser.

### 2.6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?

No information from the underlying platform is exposed by this feature.

### 2.7. Does this specification allow an origin access to sensors on a user's device

No.

### 2.8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

This feature does not expose any information about an origin's state and does not enable sending or receving of any data to or from a network connection.

### 2.9. Does this specification enable new script execution/loading mechanisms?

No.

### 2.10. Does this specification allow an origin to access other devices?

No.

### 2.11. Does this specification allow an origin some measure of control over a user agent's native UI?

This feature allows the page to respond to find-in-page, which is part of the user agent's native UI. However, this feature does not allow control over find-in-page.

### 2.12. What temporary identifiers might this specification create or expose to the web?

None.

### 2.13. How does this specification distinguish between behavior in first-party and third-party contexts?

With regards to cross origin iframes, this behavior will occur in each beforematch case:
* In the ElementFragment case, the scrolling, fragments, and beforematch events are fully insulated from each other when they occur in different cross origin iframes.
* In the ScrollToTextFragment case, text inside of a cross origin iframe will not be searched for or scrolled to. Therefore, no beforematch event will be emitted.
* In the find-in-page case, content in cross origin iframes can be searched for and scrolled to. In this case, the cross origin iframe will get the beforematch event and scrolling will happen as normal, but the main frame won't know anything about it, which maintains the insulation between origins.

### 2.14. How does this specification work in the context of a user agent's Private Browsing or "incognito" mode?

This feature will work the same regardless of incognito mode being enabled or disabled.

### 2.15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?

I included a Privacy Considerations section further down this document. I did not include a Security Considerations section because I do not believe this feature exposes any security vulnerabilities.

### 2.16. Does this specification allow downgrading default security characteristics?

No.

### 2.17. What should this questionnaire have asked?

I have no suggestions to improve this questionnaire.

## Privacy Considerations

### 1. What privacy attacks have been considered?

The beforematch event could be used to guess what the user is searching for.
* In the ElementFragment case, there are no new vulnerabilities. Any information possibly exposed by beforematch is already exposed by `window.location.hash`, the `hashchange` event, and `document.getElementById`. The following code snippet's `elementFromFragment` is the same element that `beforematch` will fire on when an element is scrolled to on hash change:
```javascript
window.addEventListener('hashchange', () => {
  const elementFromFragment = document.getElementById(window.location.hash);
});
```
* In the ScrollToTextFragment case, the beforematch event will hint to what the user was searching for by firing on the block level element containing the text which was searched for. However, the :target pseudo selector which is already added by ScrollToTextFragment is also applied to the same element that beforematch would fire on, so the beforematch event will not expose anything new in this case.
* In the find-in-page case, the same information will be exposed as the ScrollToTextFragment case. However, if the user is incrementally searching by typing in a search character by character, it will result in repeated beforematch events being fired. It may be possible to reconstruct what the user is searching for in this case by having elements in the page which match every next possible character the user could search for, and then when a match occurs, changing the text of all of the elements to contain every next possible match, and so on. Here is an example page which conveys this idea. I couldn't actually get it to work properly, but with some more work I think it might:
```html
<style>
  .hidden {
    font-size: 3px;
  }
</style>
<script>
  onload = () => {
    const hiddenElementContainer = document.createElement('div');
    document.body.appendChild(hiddenElementContainer);

    const label = document.createElement('div');
    label.textContent = 'you are searching for:';
    document.body.appendChild(label);

    const resultElement = document.createElement('div');
    document.body.appendChild(resultElement);

    let result = '';

    const nextResultElements = [];

    const update = () => {
      resultElement.textContent = result;
      for (let i = 0; i < 128; i++) {
        const element = nextResultElements[i];
        element.textContent = result + String.fromCharCode(i);
      }
    };

    for (let i = 0; i < 128; i++) {
      const element = document.createElement('div');
      element.className = 'hidden';
      element.addEventListener('beforematch', () => {
        result = element.textContent;
        update();
      });
      hiddenElementContainer.appendChild(element);
      nextResultElements.push(element);
    }

    update();
  };
</script>
```

### 2. What privacy attacks have been deemed out of scope (and why)?

No attacks have been deemed out of scope.

However, find-in-page attacks may already be possible due to the fact that pages can already intercept ctrl+f and already have the power to spoof the functionality and appearance of the browser's native find-in-page. In fact, doing this may be even easier and more reliable than using beforematch to extract the user's search text.

For the ScrollToTextFragment case, you can already figure out what block-level element containing the search result was by looking at the element with the :target pseudo selector.

### 3. What privacy mitigations have been implemented?

No privacy mitigations have been implemented.

For the find-in-page privacy flaw, I could imagine a mitigation where we would choose not to fire beforematch repeatedly in this case. In the example page above, the scroll offset would not be change in between firings of the event, which we could use as a heuristic to combat this case. However, I'm not sure that it couldn't be circumvented or if it would cause issues when trying to use the feature legitimately.

### 4. What privacy mitigations have considered and not implemented (and why)?

No mitigations have been implemented.

Besides the possible mitigation I mentioned in the above section, no others have been considered.
