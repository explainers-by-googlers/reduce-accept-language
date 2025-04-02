# Security and Privacy questionnaire

### 1. What information does this feature expose, and for what purposes?

This feature exposes less information via the `Accept-Language` string than is currently exposed. Its intent is to reduce fingerprinting of existing information, not to expose new information. To mitigate the impact on certain use cases, we provide a language negotiation mechanism to determine the best language representation for the user. A server can potentially send an `Avail-Language` header to indicate its language preferences. If the language specified in the `Content-Language` header does not match the client's preferences, the client can select another language from the `Avail-Language` list that aligns with its `Accept-Language` preferences.

### 2. Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?
Yes.

### 3. Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?
No more than what Accept-Language already exposes (and our goal is to reduce that).

### 4. How do the features in your specification deal with sensitive information?
This feature does not deal with sensitive information.

### 5. Does data exposed by your specification carry related but distinct information that may not be obvious to users?
No.

### 6. Do the features in your specification introduce state that persists across browsing sessions?
For origins that opt-in the language negotiation mechanism, this specification will save the most preferred negotiated language in a cache owned by the user agent. This cache is an ordered map, keyed by origin, with each value being a single language. A site can clear the browser's cached language for its origin by clearing site data. The cached language will also be automatically cleared if there is no site activity for 30 days.

### 7. Do the features in your specification expose information about the underlying platform to origins?
No.

### 8. Does this specification allow an origin to send data to the underlying platform?
No.

### 9. Do features in this specification enable access to device sensors?
No.

### 10. Do features in this specification enable new script execution/loading mechanisms?
No.

### 11. Do features in this specification allow an origin to access other devices?
No.

### 12. Do features in this specification allow an origin some measure of control over a user agent’s native UI?
No.

### 13. What temporary identifiers do the features in this specification create or expose to the web?
None.

### 14. How does this specification distinguish between behavior in first-party and third-party contexts?
For origins that opt-in the language negotiation mechanism, to reduce performance risks, third-party contexts will not have a separate language negotiation process. Instead, they will inherit the first party's preferred language to maintain a consistent user experience.

### 15. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?
Chrome's Incognito mode already reduces Accept-Language; this change brings its behavior to regular mode.

### 16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?
No. The feature in the specification does not have any security or privacy impact beyond what Accept-Language today provide.

### 17. Do features in your specification enable origins to downgrade default security protections?
No

### 18. What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?
Nothing. The feature is only used during initial document processing when a navigation is triggered.

### 19. What happens when a document that uses your feature gets disconnected?
Nothing. The feature is only used during initial document processing when a navigation is triggered.

### 20. Does your spec define when and how new kinds of errors should be raised?
This feature does not produce new kinds of errors.

### 21. Does your feature allow sites to learn about the user’s use of assistive technology?
No.

### 22. What should this questionnaire have asked?
Nothing else that comes to mind.
