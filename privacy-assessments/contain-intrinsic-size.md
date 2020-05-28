### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

This feature exposes the intrinsic sizing control to web developers in order to
better control layout when size containment is present.

### 2.2. Is this specification exposing the minimum amount of information necessary to power the feature?

Yes

### 2.3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?

This feature does not expose or deal with PII or PII-derived data.

### 2.4. How does this specification deal with sensitive information?

This feature does not expose or deal with sensitive information.

### 2.5. Does this specification introduce new state for an origin that persists across browsing sessions?

No

### 2.6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?

None

### 2.7. Does this specification allow an origin access to sensors on a user’s device

No

### 2.8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

This specification does not expose any data. It only allows the UA's rendering
engine to consider new inputs from the origin. This is analogous to CSS `min-width`
and `min-height` features, which act as inputs without exposing any data.

### 2.9. Does this specification enable new script execution/loading mechanisms?

No

### 2.10. Does this specification allow an origin to access other devices?

No

### 2.11. Does this specification allow an origin some measure of control over a user agent’s native UI?

No

### 2.12. What temporary identifiers might this this specification create or expose to the web?

None

### 2.13. How does this specification distinguish between behavior in first-party and third-party contexts?

This specifically allows additional layout controls for the developer. It does
not distinguish first- and third-party context.

### 2.14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?

Private Browsing or "incognito" mode context do no affect this specification.

### 2.15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?

Yes

### 2.16. Does this specification allow downgrading default security characteristics?

No

### 2.17. What should this questionnaire have asked?

This questionnaire seems very thorough. Some questions are somewhat hard to
answer when considering a specification such as this one, but it was a good
exercise to think about the specification from all privacy and security angles.
