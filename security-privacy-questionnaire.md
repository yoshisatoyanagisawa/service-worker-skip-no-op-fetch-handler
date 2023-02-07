# Answers to the Self-Review Questionnaire: Security and Privacy

https://www.w3.org/TR/security-privacy-questionnaire/

### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

No additional information should be exposed by the proposal.

### 2.2. Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes.  However, the proposal does not change how the browser handles information.

### 2.3. How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

The proposal does not change how the browser handles PII.

### 2.4. How do the features in your specification deal with sensitive information?

n/a

### 2.5. Do the features in your specification introduce new state for an origin that persists across browsing sessions?

Yes.  The proposal asks a user agent to remember if the service worker fetch handler is no-op or not.
It can be stored to the local storage to be used in the next navigation.

### 2.6. Do the features in your specification expose information about the underlying platform to origins?

No.

### 2.7. Does this specification allow an origin to send data to the underlying platform?

No.  It asks the browser to remember the service worker fetch handler is no-op or not, though.

### 2.8. Do features in this specification enable access to device sensors?

No.

### 2.9. Do features in this specification enable new script execution/loading mechanisms?

No.

### 2.10. Do features in this specification allow an origin to access other devices?

No.

### 2.11. Do features in this specification allow an origin some measure of control over a user agent’s native UI?

No.

### 2.12. What temporary identifiers do the features in this specification create or expose to the web?

n/a

### 2.13. How does this specification distinguish between behavior in first-party and third-party contexts?

m/a

### 2.14. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

It should not bring any difference with and without the proposal.

### 2.15. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

No.  However, the original Service Worker specification considers it in
[6. Security Considerations](https://www.w3.org/TR/service-workers/#security-considerations).
The proposal does not change any security requirements and privacy requirements explained there.

### 2.16. Do features in your specification enable origins to downgrade default security protections?

No.

### 2.17. How does your feature handle non-"fully active" documents?

n/a

The proposal should not change the behavior of non-”fully active” documents.

### 2.18. What should this questionnaire have asked?

n/a

No additional questions are needed.
