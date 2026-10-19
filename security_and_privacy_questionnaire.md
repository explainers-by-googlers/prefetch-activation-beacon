# [PUBLIC] Security and Privacy Questionnaire for Prefetch Activation Beacon API

## 1 What information does this feature expose, and for what purposes?

This feature exposes the fact that a specific prefetched main resource has been "activated" (consumed by a user navigation and presented to the user) to the origin server that served the resource. In traditional uncached web navigation, the origin server naturally learns of a page view via the main-document HTTP `GET` request; speculative prefetching decouples fetching from presentation, leaving the server unable to distinguish discarded speculative loads from actual visits at the HTTP layer.

By turning speculative loading into a 3-phase commit (Request, Response, and Activation Confirmation), this feature restores parity with non-speculative navigations and allows web developers to accurately measure the precision and performance impact of their prefetch strategies, overcoming the unreliability of client-side JavaScript reporting (due to broad extension filter rules on consolidated logging endpoints, cache restore ambiguity, lifecycle race conditions, and renderer crashes).

## 2 Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

Yes. The beacon simply sends an HTTP `HEAD` request to a relative URL specified by the server in the `on-prefetch-activation` response header of the prefetch request, carrying a post-modifier frozen snapshot (`C0`) of the exact same-origin cookies sent with the initial prefetch request. Because `C0` is captured from the final outbound `Cookie` header of the initial prefetch request after all browser cookie policies, network hooks, and extension modifications have been applied, the beacon conveys zero new client-side state or interim cookie changes; it only "echoes back" the URL, parameters, and prefetch cookie state that the server already received.

## 3 Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

No. The API does not collect or expose new personal information. The URL parameters are defined by the server, and any cookies included in the beacon are a post-modifier frozen snapshot (`C0`) of the exact same-origin cookies already sent to the server during the initial prefetch request (if the prefetch was credentialed; uncredentialed cross-site prefetches have an empty cookie snapshot). Because the server dynamically specifies the beacon URL per prefetch response and could otherwise encode a 1:1 session identifier in the query string, sending the frozen cookie snapshot exposes no additional identifying data beyond what the server already received during the prefetch and avoids forcing servers to build custom side-channel joining tables outside standardized browser cookie controls.

## 4 How do the features in your specification deal with sensitive information?

The API does not deal with sensitive information. It only reports the activation event.

## 5 Does data exposed by your specification carry related but distinct information that may not be obvious to users?

No. The beacon is a simple HEAD request to a server-defined URL carrying the frozen cookie snapshot from the initial prefetch request.

## 6 Do the features in your specification introduce state that persists across browsing sessions?

No. The activation metadata (including the frozen cookie snapshot) is associated with the browser's speculative loading context and is discarded when that context expires, when the user navigates away, or when the user clears cookies or site data. It does not introduce persistent state.

## 7 Do the features in your specification expose information about the underlying platform to origins?

No.

## 8 Does this specification allow an origin to send data to the underlying platform?

No. The communication is strictly between the browser and the web server.

## 9 Do features in this specification enable access to device sensors?

No.

## 10 Do features in this specification enable new script execution/loading mechanisms?

No. The beacon is handled directly by the browser's navigation stack upon navigation commit and does not boot up a Service Worker or execute any page scripts. It is designed to be non-interceptable and cannot be blocked or modified by client-side JavaScript, Service Workers, or browser extensions.

At the same time, user and extension agency are preserved:
- If an extension blocks access to the destination page itself, the initial prefetch request is blocked and no activation beacon is registered.
- Any extension or policy modifications to cookies on the initial prefetch request are honored because the frozen cookie snapshot (`C0`) is captured *after* all cookie modifiers have run on the initial prefetch request.
- Prefetching and its activation confirmation form an inseparable transactional unit: users (via browser preloading settings) and extensions (via `chrome.privacy.network.networkPredictionEnabled`) retain the ability to disable speculative preloading altogether, which disables both the speculative fetch and its activation signal.

## 11 Do features in this specification allow an origin to access other devices?

No.

## 12 Do features in this specification allow an origin some measure of control over a user agent’s native UI?

No.

## 13 What temporary identifiers do the features in this specification create or expose to the web?

None. The API itself does not create identifiers. Because the component initiating a prefetch (such as browser omnibox prediction or a referring page's speculation rules) often lacks the full server session state to mint a transaction ID at request time, the server can include a correlation identifier (like an event ID) in the `on-prefetch-activation` response header's relative URL. This identifier is generated by the server when serving the prefetch response, not by the browser, and is strictly scoped to a single same-origin activation event.

## 14 How does this specification distinguish between behavior in first-party and third-party contexts?

The reporting endpoint must be **same-origin** with the prefetched page (enforced by requiring a relative path in `on-prefetch-activation`). It is possible for site A to prefetch a page from site B and for site B to receive the activation confirmation on site B's origin. However, the prefetch activation beacon will never be sent to an origin different from the prefetched page.

Regarding credentials, the activation beacon sends a post-modifier frozen snapshot (`C0`) of the same-origin cookies from the initial prefetch request:
- **Credentialed (e.g., Same-Origin) Prefetches**: The beacon includes the frozen snapshot of same-origin cookies captured from the final outbound `Cookie` header of the initial prefetch request (after all browser cookie policies, network hooks, and extension modifications have been applied), ignoring any cookie mutations that occurred between prefetch and activation.
- **Uncredentialed Cross-Site Prefetches**: When a cross-site prefetch is dispatched without ambient credentials (in an isolated network context), the initial prefetch carries no existing cookies (`C0` is empty). Because the beacon uses this frozen snapshot rather than querying the live cookie store, the cookie snapshot is empty and the beacon never retroactively attaches ambient cookies.

If redirects occur:
- **Speculative Request Redirects**: Only the `on-prefetch-activation` header in the final response of the speculative navigation is honored. Any headers on intermediate redirects are discarded. The endpoint's same-origin status is validated against the final response URL.
- **Beacon Request Redirects**: If the activation beacon itself is redirected cross-origin, the request is blocked by the browser. It is only allowed to redirect to same-origin endpoints.

## 15 How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

Activation beacons only include a frozen snapshot of the same-origin cookies sent with the initial prefetch request within that same browsing context. In Private Browsing or Incognito mode, the beacon is bound to the ephemeral cookie store of that private session and never shares state or credentials with normal browsing sessions. If the private session ends or cookies/site data are cleared before activation, the prefetched response and its cookie snapshot are invalidated and discarded.

## 16 Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

Yes.

## 17 Do features in your specification enable origins to downgrade default security protections?

No.

## 18 What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

The trigger is tied to the speculative load context. The beacon does not fire for general navigations that happen to hit the disk cache long after the original speculative context has expired. It only fires when the browser's speculative loading logic explicitly consumes the response to satisfy a predicted navigation. The back/forward navigation and refreshes are filtered from reporting to avoid duplicate signals, unless the page is prefetched again in a new speculative context.

## 19 What happens when a document that uses your feature gets disconnected?

The activation trigger is valid for a single event and tied to the speculative context. If the document is disconnected, the context is lost and the beacon will not fire.

## 20 Does your spec define when and how new kinds of errors should be raised?

The spec focuses on the happy path of activation. If any error happens during the prefetch or the activation, the beacon will not be sent.

Additionally, the browser silently discards the beacon request if the beacon's own HTTP HEAD request is redirected to a cross-origin URL or if the redirect changes the HTTP method from HEAD. These events do not raise web-visible errors but will cause the activation telemetry to be lost.

## 21 Does your feature allow sites to learn about the user’s use of assistive technology?

No.
