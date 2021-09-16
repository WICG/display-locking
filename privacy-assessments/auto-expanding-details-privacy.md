# Auto-expanding `<details>` security and privacy self-review questionnaire

## Questions to Consider

### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

This feature will open details elements in response to find-in-page and ScrollToTextFragment. When a details element is opened, the page can observe it via the `toggle` event which is fired and the `open` attribute. It is necessary that we continue to fire this event and add the attribute when the details element is opened because the page may be relying on them to reconcile other state in the page or in script with the open state of the details element.

### 2.2. Is this specification exposing the minimum amount of information necessary to power the feature?

Yes.

### 2.3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?

If you consider find-in-page queries or ScrollToTextFragment queries personal information or PII, it is dealt with the same way "sensitive information" is in the next section.

No new information stored on the device is exposed by this feature.

### 2.4. How does this specification deal with sensitive information?

The search queries present in the ScrollToTextFragment and find-in-page features may be sensitive information, and are considered in the "Privacy Considerations" section.

### 2.5. Does this specification introduce new state for an origin that persists across browsing sessions?

This feature does not store any new data in the browser.

### 2.6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?

No information from the underlying platform is exposed by this feature.

### 2.7. Does this specification allow an origin access to sensors on a user's device

No.

### 2.8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

This feature does not expose any information about an origin's state and does not enable sending or receiving of any data to or from a network connection.

### 2.9. Does this specification enable new script execution/loading mechanisms?

No.

### 2.10. Does this specification allow an origin to access other devices?

No.

### 2.11. Does this specification allow an origin some measure of control over a user agent's native UI?

No.

### 2.12. What temporary identifiers might this specification create or expose to the web?

None.

### 2.13. How does this specification distinguish between behavior in first-party and third-party contexts?

ScrollToTextFragment doesn't work on cross-origin iframes, so that isn't a concern for this feature.

Find-in-page does work on cross origin iframes, but every time find-in-page scrolls to a match in an iframe it will continue to emit `scroll` events which are already revealing more information than this feature would.

### 2.14. How does this specification work in the context of a user agent's Private Browsing or "incognito" mode?

This feature will work the same regardless of incognito mode being enabled or disabled.

### 2.15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?

A Privacy Considerations section can be found further down in this document. There is no Security Considerations section because this feature does not expose any security vulnerabilities.

### 2.16. Does this specification allow downgrading default security characteristics?

No.

## Privacy Considerations

### 1. What privacy attacks have been considered?

The page could observe the `toggle` event or the addition of the `open` attribute when this feature automatically expands details elements in an attempt to find out what the user was searching for with find-in-page or ScrollToTextFragment. If the page listens to the relevant events, it may be able to determine whether the user clicked on the details element to expand it or if it was automatically opened by this feature.

### 2. What privacy attacks have been deemed out of scope (and why)?

The aforementioned privacy attack is out of scope because it is less powerful than existing vulnerabilities for find-in-page and ScrollToTextFragment, meaning that this feature has a zero information delta with find-in-page and ScrollToTextFragment.

For find-in-page, vulnerabilities already exist which use the `scroll` event which is synchronous unlike the `toggle` event this feature will trigger. Every time find-in-page scrolls to a result, the page fires a `scroll` event. If the page creates a tiny scrollable area with every next possible character the user could type, it can recreate what the user typed into find-in-page. An example page which uses this attack can be found [here](/resources/find-in-page/search-incremental.html). If we were to remove the `scroll` events from find-in-page then this feature could become another vector for this attack, but this is unlikely to happen because it would break find-in-page for web pages which rely on the `scroll` event. This attack could be addressed regardless of the particular vector by adding a delay to find-in-page's scrolling which allows the user to type in enough characters to prevent the page incrementally building the user's search query, which Safari already does today.

For ScrollToTextFragment, this feature exposes less than the existing `:target` pseudo selector added by ScrollToTextFragment.

### 3. What privacy mitigations have been implemented?

No privacy mitigations have been implemented.

### 4. What privacy mitigations have considered and not implemented (and why)?

Adding a delay to the frequency find-in-page queries are sent to the page would help mitigate the possibility of this feature being used to snoop on find-in-page. Due to the existing `scroll` events, this feature does not make the situation any worse and therefore should not be blocked on adding a delay to find-in-page. However, the find-in-page delay is planned to be implemented in chromium.
