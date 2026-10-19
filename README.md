# Explainer: Prefetch Activation Beacon API

## Proponents

- Jiacheng Guo (gjc@google.com)

## Participate
- https://github.com/explainers-by-googlers/prefetch-activation-beacon/issues

## Table of Contents

- [Introduction](#introduction)
  - [Restoring the HTTP Transaction Loop (3-Phase Commit)](#restoring-the-http-transaction-loop-3-phase-commit)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Use cases](#use-cases)
- [Proposed Solution](#proposed-solution)
  - [Header Syntax](#header-syntax)
  - [Browser Mechanism](#browser-mechanism)
  - [Non-Interceptable Execution](#non-interceptable-execution)
  - [Examples](#examples)
- [Detailed design discussion](#detailed-design-discussion)
  - [Activation Definition](#activation-definition)
  - [Post-Commit Dispatch](#post-commit-dispatch)
  - [Endpoint Lifetime](#endpoint-lifetime)
- [Considered alternatives](#considered-alternatives)
  - [Client-side JavaScript reporting](#client-side-javascript-reporting)
  - [Commit verb to the same prefetch URL](#commit-verb-to-the-same-prefetch-url)
  - [Extension interception and dedicated resource types](#extension-interception-and-dedicated-resource-types)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
  - [The "Echo Back" Principle](#the-echo-back-principle)
  - [Static & Non-Modifiable](#static--non-modifiable)
  - [Origin Boundaries (Target-Origin Restriction)](#origin-boundaries-target-origin-restriction)
  - [Same-Origin Cookie Snapshot](#same-origin-cookie-snapshot)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)

## Introduction

This proposal introduces the `on-prefetch-activation` HTTP response header, which allows a server to specify a same-origin telemetry endpoint for prefetched resources. This provides web developers with a reliable, browser-native signal to confirm when a speculatively fetched document is actually presented to the user, enabling accurate measurement of the precision and performance impact of their prefetch strategies. Because the activation beacon is dispatched directly by the browser's navigation stack when a prefetch is consumed, it is significantly more accurate than client-side scripts, which are susceptible to content-filter blocking on consolidated logging endpoints, cache restore ambiguity, duplicate events, and aborted navigations.

### Restoring the HTTP Transaction Loop (3-Phase Commit)

For decades, web traffic measurement has rested on a simple HTTP invariant: **if a user visits an uncached URL, the origin server receives and serves an incoming HTTP `GET` request for the main document.** As browsers expand speculative loading capabilities (such as the [Speculation Rules API](https://wicg.github.io/nav-speculation/speculation-rules.html), omnibox prefetching, and on-press prefetching), they unbundle **fetching content** from **presenting content**:
- **Speculative Fetch**: The browser requests the main document (`Sec-Purpose: prefetch`) in anticipation of a navigation.
- **Speculative Discard**: If the user does not navigate to the predicted URL, the prefetched document is silently discarded from the browser's prefetch cache.
- **Activation / Commit**: If the user proceeds with the predicted navigation, the browser satisfies the navigation directly from the local prefetch cache.

At the HTTP layer, the origin server has no native mechanism to distinguish between a discarded speculative prefetch and an activated page view, forcing sites to rely on fragile client-side JavaScript beacons just to count basic page visits.

Conceptually, HTTP with speculative preloading can be modeled as a **3-phase commit**: Request, Response, and verifying that the page is finally presented to the user. Combining `on-prefetch-activation` with prefetch turns speculative loading into a complete 3-phase commit—restoring parity with non-speculative navigations where the origin server always knows when an uncached main document is visited.

```
Traditional Navigation:
User Action ───> HTTP GET /page ───────────────> Server responds & logs Page View

Speculative Navigation (Status Quo - Broken HTTP Loop):
Speculation ───> HTTP GET /page (Sec-Purpose) ─> Server serves bytes (Uncertain State)
User Action ───> Satisfied from local cache   ──> Server learns nothing at HTTP layer
JS Telemetry ──> sendBeacon(/log) ─────────────> Blocked / Failed / Delayed

Speculative Navigation (With Activation Beacon):
Speculation ───> HTTP GET /page ───────────────> Server returns "on-prefetch-activation: /ack?id=123"
User Action ───> Committed to Frame Stack     ──> Browser fires HTTP HEAD /ack?id=123 (ACK)
```

## Goals

- **Reliable Activation Reporting**: Provide a robust, browser-level channel for servers to confirm when a prefetched resource is presented to a user, restoring HTTP-layer parity with non-speculative navigations.
- **Declarative Integration**: Enable servers to specify activation reporting endpoints directly within the HTTP response headers of speculative loads, simplifying implementation and correlation for developers.
- **Privacy-First Design**: Ensure that the activation mechanism exposes zero new client-side tracking information beyond what the server already received during the initial prefetch request.

## Non-goals

- **Legacy API Support**: This API will not trigger for `<link rel=prefetch>` or `<link rel=prerender>`, as these are being deprecated in favor of the Speculation Rules API.
- **Subresource Measurement**: This is not intended for subresource prefetches (e.g., images, scripts). "Prewarm" mechanisms or other specialized APIs are the preferred path for measuring subresource usage.
- **Dynamic Data Channel**: This is not a general-purpose channel for sending arbitrary client-side states. It is strictly limited to reporting the server-defined activation event.

## Use cases

Unlike general-purpose client-side analytics or ad-impression trackers—which measure arbitrary in-page behavioral events or third-party interactions that HTTP never natively exposed—the activation signal is strictly scoped to restoring the **1:1 first-party main-document HTTP `GET` signal** that the origin server would have received by definition in a non-prefetched navigation.

Accurately confirming prefetch activation is essential for web services across several operational and technical dimensions:
- **Infrastructure ROI & Capacity Planning**: Speculative preloading consumes significant server, database, and CDN resources. Engineering teams must measure the hit rate (precision) of their prefetch rules against the cost of discarded requests to justify and sustain investments in speculative loading.
- **Traffic Accounting & Business Operations**: Real user visits drive foundational operational metrics, including search query accounting, content engagement, and partner attribution. When prefetching masks whether a page was visited—or when client-side activation pings are dropped—operators observe artificial drops in reported traffic (such as an observed ~10% loss in search query accounting during on-press prefetch rollouts).
- **Optimizing Speculation Heuristics**: The Speculation Rules API provides fine-grained controls (such as `eagerness` levels and hover/viewport triggers). Developers need reliable ground-truth activation rates to tune whether their speculation rules are too aggressive or too conservative.
- **Search Relevance & User Intent Satisfaction**: Search engines and content platforms rely on accurate visit confirmations to evaluate result quality and query satisfaction.
- **Security, Fraud Detection, and Rate Limiting**: Distinguishing between actual human page views and discarded speculative requests is necessary for accurate rate limiting, bot detection, and anomaly-detection systems.

## Proposed Solution

The core of this proposal is the `on-prefetch-activation` HTTP response header.

### Header Syntax

When responding to a prefetch request, a server can include the following header:

```http
on-prefetch-activation: <relative-endpoint-path>
```

The `<relative-endpoint-path>` value is the relative URL (path and optional query string) of the telemetry endpoint that the browser should notify upon activation. Restricting the syntax to a relative path enforces that the reported endpoint is always **same-origin** with the prefetched page by construction, while allowing the server to dynamically generate and attach a correlation or transaction identifier when serving the prefetch response.

### Browser Mechanism

The browser process consumes this header from the prefetch response and associates the provided relative URL—along with a post-modifier frozen snapshot (`C0`) of the same-origin cookies sent with the initial prefetch request—with the prefetched document in its internal cache.

The activation beacon is triggered when the navigation utilizing the cached content is successfully committed in a non-prerender context. At this point, the browser sends an HTTP `HEAD` request to the specified endpoint carrying the frozen snapshot of same-origin cookies from the initial prefetch request.

### Non-Interceptable Execution

Because the activation beacon is handled directly by the browser's navigation stack as the final phase of the main-frame navigation commit, it does not pass through the origin's Service Worker, is not interceptable by client-side JavaScript, and cannot be blocked or modified by browser extensions.

Unless an extension intends to block access to the destination page altogether (which already blocks the initial prefetch request), content filters allow permitted top-level main-frame navigations (`type: "main_frame"`) to proceed so user-intended browsing works. Because the activation beacon is an atomic confirmation of that same permitted main-frame navigation commit, it is semantically part of the navigation transaction itself. This ensures:
- **High Reliability**: The signal cannot be blocked, delayed, or tampered with by faulty or malicious scripts, nor by broad extension filter rules targeting general-purpose JavaScript logging endpoints.
- **Zero Script Execution**: The dispatch of the beacon does not boot up a Service Worker or run any page scripts, keeping the activation path fast and lightweight.
- **Preserving User and Extension Opt-Out Controls**: Prefetching and its activation confirmation form an inseparable transactional unit. Users (via browser preloading settings) and extensions (via `chrome.privacy.network.networkPredictionEnabled`) retain the ability to disable speculative preloading altogether, which disables both the speculative fetch and its activation signal.

### Examples

#### Same-Origin Prefetch and Activation
The user navigates to `a.com/` and the loaded page prefetches `a.com/target`. The beacon will be sent when the user navigates to `a.com/target` afterwards.

#### Cross-Origin Prefetch and Activation
The user navigates to `a.com/` and the loaded page prefetches `b.com/target`. The beacon will be sent when the user navigates to `b.com/target` afterwards.

#### Devalidation on Navigating Away
The user navigates to `a.com/` and the loaded page prefetches `a.com/target`. Then the user navigates to `c.com`. The beacon will not be sent when the user navigates to `a.com/target` afterwards.

## Detailed design discussion

### Activation Definition

Activation is defined as the point at which a document loaded via a speculative trigger transitions to the active state in its browsing context. This typically occurs when a user initiates a navigation that the browser satisfies using a previously prefetched response.

### Post-Commit Dispatch

To ensure that the beacon accurately reflects a page being shown to the user, the browser dispatches the beacon after the navigation has been committed.

### Endpoint Lifetime

To maintain privacy and prevent "zombie" beacons from unrelated navigations, the following lifetime rules apply:
- **One-Time Use**: The activation trigger is valid for a single activation event. Once the beacon is fired, the trigger is devalidated for the page.
- **Referrer-Bound**: The activation trigger is tied to the speculative context of the triggering (referrer) document. If the user navigates away from the referrer page to an unrelated destination, the speculative load's activation metadata is discarded.
- **No General Cache Persistence**: The beacon does not fire for general navigations that happen to hit the disk cache long after the original speculative context has expired. It only fires when the browser's speculative loading logic explicitly consumes the response to satisfy a predicted navigation.

## Considered alternatives

### Client-side JavaScript reporting

One alternative is to continue relying on client-side JavaScript to report activation. For example, a site might inject a small script into a prefetched page that fires a `navigator.sendBeacon()` or `fetch({keepalive: true})` request when the page becomes visible (e.g., on `pageshow` or `prerenderingchange`).

However, this approach was rejected due to several fundamental reliability issues:
- **Cache Ambiguity and Duplicate Reporting**: Telemetry often filters back/forward navigations to avoid double-counting, which causes back/forward navigations to prefetched pages to be missed. Conversely, back/forward cache (BFCache) restores and standard HTTP disk cache hits (particularly with features like `No-Vary-Search`) can re-run scripts, leading to duplicate activation signals and corrupted attribution.
- **Lifecycle Race Conditions and Aborted Navigations**: If a user quickly navigates away, closes the tab, or the renderer process terminates unexpectedly, client scripts may fail to execute early enough or complete their outbound network requests.
- **Restricted or Delayed Scripting Environments**: In environments where JavaScript is disabled, constrained, or delayed by heavy main-thread execution, the activation signal is lost entirely despite the document being displayed to the user.
- **Blanket Extension Interception of JS Logging Endpoints**: Common content-filtering extensions maintain broad filter rules targeting consolidated client-side logging endpoints (such as `/gen_204`, `/telemetry`, or `/log`). When activation confirmation is piggybacked onto client-side scripts, these rules block the activation signal wholesale—for example, filtering out approximately 10% of activation beacons in Google Search's on-press prefetch even when the user actively viewed the main document.

### Commit verb to the same prefetch URL

Another alternative is to avoid specifying an endpoint in a response header and instead have the browser send a dedicated HTTP method (such as a `GET-COMMIT` verb) or request header back to the exact same URL that was prefetched when the navigation commits.

While conceptually similar, in practice the component initiating a prefetch (such as browser omnibox prediction, on-press speculation, or a referring page's speculation rules) often lacks the full server session state required to mint a unique transaction or event ID in the initial request URL. Allowing the server to specify a relative endpoint in the `on-prefetch-activation` response header (e.g., `on-prefetch-activation: /ack?id=123`) enables the server to generate the correlation token at the time it serves the prefetch response.

### Extension interception and dedicated resource types

We also evaluated two alternative models for how browser extensions (`chrome.webRequest` and `chrome.declarativeNetRequest`) interact with activation beacons:
- **Standard Extension Visibility (Interceptable)**: Treating the activation beacon as an ordinary network request exposed to extension filtering leaves the core reliability problem unresolved when filter lists maintain broad regular expressions matching first-party logging paths (e.g., `||example.com/log^`). If sites observe that activation beacons are blocked, they face strong operational incentives to implement fallback workarounds (such as `Set-Cookie` probes) or disable speculative preloading for those clients, degrading performance for extension users.
- **Dedicated Extension Resource Type (`speculation_beacon`)**: Tagging the activation beacon with a distinct resource type (e.g., `declarativeNetRequest.ResourceType.SPECULATION_BEACON` or `$speculation-beacon` in filter list syntax) would allow filter list maintainers to distinguish first-party main-document activation confirmations from third-party tracking pings. However, because filter lists frequently treat first-party beaconing and third-party tracking similarly, a dedicated resource type may still be blocked by default, resulting in the same reliability failure as standard extension visibility.

Dispatching the beacon directly via the navigation stack while preserving top-level user and extension preloading opt-outs (browser preloading settings and `chrome.privacy.network.networkPredictionEnabled`) treats the speculative fetch and its commit confirmation as an inseparable transactional unit.

## Security and Privacy Considerations

### The "Echo Back" Principle
The API is designed to avoid creating new tracking vectors. The browser conveys no new client-side information to the server; it simply "echoes back" the URL and parameters that the server itself provided in the initial response, along with a frozen snapshot of the exact same-origin cookies that the server already received during the initial prefetch request. This confirms the activation event without exposing dynamic client state or interim state changes.

### Static & Non-Modifiable
Because the activation URL is defined in the HTTP header, it is inaccessible to and cannot be modified by third-party JavaScript. This prevents malicious scripts from injecting tracking identifiers into the beacon URL.

### Origin Boundaries (Target-Origin Restriction)
To prevent the API from being used for cross-site tracking, the reporting endpoint must be same-origin with the prefetched page.
- If the browser initiatively prefetches `target.com`, the beacon can only be sent to an endpoint on `target.com`.
- If `referer.com` prefetches `target.com`, the beacon can only be sent to an endpoint on `target.com`.
- Reporting to the referrer `referer.com` or any third party `other.com` is prohibited.

This ensures that only the site being prefetched—which already knows the user is visiting—receives the signal.

#### Redirect Behavior

Security boundaries are strictly enforced during redirects in both the speculative navigation phase and the activation beacon request itself:

1. **Speculative Navigation Redirects**:
   - If the speculative request (prefetch or prerender) is redirected, only the `on-prefetch-activation` header specified in the **final response** is processed.
   - Any `on-prefetch-activation` headers encountered in intermediate redirect responses are **ignored and dropped**.
   - The same-origin validation for the activation beacon endpoint is checked against the **final response's URL**, not the initial request URL.

2. **Activation Beacon Request Redirects**:
   - The activation beacon request (sent as an HTTP `HEAD` request) **must only redirect to same-origin URLs** (same-origin with the original beacon URL).
   - If the beacon request is redirected to a cross-origin URL, the browser will **immediately block and discard the request**.
   - The HTTP method of the beacon request must remain `HEAD` during redirection. If a redirect attempts to change the request method (e.g., a `302` redirect that changes `HEAD` to `GET`), the request is blocked.

### Same-Origin Cookie Snapshot
Rather than sending a credentialless request or reading the browser's live cookie jar at the moment of activation, the activation beacon sends a **frozen snapshot of the exact same-origin cookies from the initial prefetch request**:

```
t0 (Prefetch):    Browser sends Prefetch with Cookie Snapshot C0 ──> Server receives C0
t1 (Interim):     Concurrent activity mutates live cookie jar to C1 (Ignored by beacon)
t2 (Activation):  Page activates; Browser sends Beacon with C0   ──> Server receives C0 (Zero new info)
```

This snapshot architecture enforces strict privacy invariants while supporting stateless server-side telemetry:
- **Zero New Tracking Information**: Because the origin server already received cookie snapshot `C0` when serving the initial prefetch request at `t0`, re-transmitting that identical snapshot in the activation beacon at `t2` reveals no new user-identifying data to the server. The server learns only that the previously prefetched response was committed to the active browsing context.
- **No Temporal State Joining**: Any cookie additions, modifications, or cross-tab state transitions that occur in the browser's live cookie jar between prefetch (`t0`) and activation (`t2`) are excluded from the beacon. The beacon cannot be used to bridge pre-prefetch tokens with post-prefetch identity changes.
- **Post-Modifier Snapshot Capture**: The cookie snapshot `C0` is captured from the final outbound `Cookie` header of the initial prefetch request after all browser cookie policies, network hooks, and extension modifications have been applied, ensuring zero skew with the initial prefetch and preventing any bypass of cookie controls.
- **Preserving Cross-Site Isolation and Cookie Deletion Controls**:
  - *Uncredentialed Cross-Site Prefetches*: When a cross-site prefetch is dispatched without ambient credentials (in an isolated network context), the initial prefetch carries no existing cookies (`C0` is empty). Because the beacon uses this snapshot rather than querying the live cookie store, it never retroactively attaches ambient cookies at the beacon layer.
  - *Respecting User Cookie Clearing*: If a user clears their cookies or site data between the prefetch and activation, the prefetched response and its cookie snapshot are invalidated and discarded.
- **Avoiding the "Pseudo-Credentialless" Anti-Pattern**: Because the server dynamically specifies the beacon URL and query parameters per prefetch response, a server that received cookies during the prefetch already has the capability to encode a 1:1 session or transaction identifier in the URL. Stripping cookies would only force servers needing session attribution to maintain custom side-channel joining tables outside of standardized browser cookie controls, whereas sending the frozen cookie snapshot allows stateless HTTP frontends and telemetry endpoints to process activation beacons using standard cookie middleware.

For a more detailed analysis, see the [Security and Privacy Questionnaire](security_and_privacy_questionnaire.md).

## Stakeholder Feedback / Opposition
