# Explainer: Prefetch Activation Beacon API

## Proponents

- Jiacheng Guo (gjc@google.com)

## Participate
- https://github.com/explainers-by-googlers/prefetch-activation-beacon/issues

## Table of Contents

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [User research](#user-research)
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
- [Security and Privacy Considerations](#security-and-privacy-considerations)
  - [The "Echo Back" Principle](#the-echo-back-principle)
  - [Static & Non-Modifiable](#static--non-modifiable)
  - [Origin Boundaries (Target-Origin Restriction)](#origin-boundaries-target-origin-restriction)
  - [Credentialless Beacons](#credentialless-beacons)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)

## Introduction

This proposal introduces the `on-prefetch-activation` HTTP response header, which allows a server to specify a telemetry endpoint for prefetched resources. This provides web developers with a reliable signal to accurately measure the precision and performance impact of their prefetch strategies. Because the activation beacon is dispatched directly by the browser's navigation stack when a prefetch is consumed, it is significantly more accurate than client-side scripts, which are susceptible to cache filtering, duplicate events, and aborted navigation.

## Goals

- **Reliable Activation Reporting**: Provide a robust, browser-level channel for servers to confirm exactly when a prefetched resource is presented to a user.
- **Declarative Integration**: Enable servers to specify telemetry reporting endpoints directly within the HTTP response headers of speculative loads, simplifying the implementation for developers.
- **Privacy-First Design**: Ensure that the telemetry mechanism does not leak additional client-side information or create new vectors for user fingerprinting.

## Non-goals

- **Legacy API Support**: This API will not trigger for `<link rel=prefetch>` or `<link rel=prerender>`, as these are being deprecated in favor of the Speculation Rules API.
- **Subresource Measurement**: This is not intended for subresource prefetches (e.g., images, scripts). "Prewarm" mechanisms or other specialized APIs are the preferred path for measuring subresource usage.
- **Dynamic Data Channel**: This is not a general-purpose channel for sending arbitrary client-side states. It is strictly limited to reporting the server-defined activation event.

## Use cases

Measuring prefetch activation is essential for web developers to:
- Evaluate the precision of their prefetch rules and optimize them for better performance and resource efficiency.
- Accurately measure the actual visits to prefetched pages to understand user behavior and feature impact.

## Proposed Solution

The core of this proposal is the `on-prefetch-activation` HTTP response header.

### Header Syntax

When responding to a prefetch request, a server can include the following header:

```http
on-prefetch-activation: <relative-endpoint-path>
```

The `<relative-endpoint-path>` value is the relative URL of the telemetry endpoint that the browser should notify upon activation. The reported URL is always the same-origin as the prefetched page.

### Browser Mechanism

The browser process consumes this header from the prefetch response and associates the provided URL with the prefetched document in its internal cache.

The activation beacon is triggered when the navigation utilizing the cached content is successfully committed in a non-prerender context. At this point, the browser sends a credentialless HTTP `HEAD` request to the specified endpoint.

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

One alternative is to continue relying on client-side JavaScript to report activation. For example, a site might inject a small script into a prefetched page that fires a `sendBeacon` request when the page becomes visible.

However, this approach was rejected due to several fundamental reliability issues:
- **Cache Ambiguity**: Telemetry often filters back/forward navigations to avoid double-counting. However, back/forward navigation to prefetched pages will be missed.
- **Duplicate Reporting**: Cache restores can re-run scripts, leading to duplicate activation signals and inaccurate attribution.
- **Race Conditions**: Scripts often fail to execute early enough to capture rapid page transitions.

## Security and Privacy Considerations

### The "Echo Back" Principle
The API is designed to avoid creating new tracking vectors. The browser conveys no new client-side information to the server; it simply "echoes back" the URL and parameters that the server itself provided in the initial response. This confirms the activation event without exposing dynamic client state.

### Static & Non-Modifiable
Because the activation URL is defined in the HTTP header, it is inaccessible to and cannot be modified by third-party JavaScript. This prevents malicious scripts from injecting tracking identifiers into the beacon URL.

### Origin Boundaries (Target-Origin Restriction)
To prevent the API from being used for cross-site tracking, the reporting endpoint must be same-origin with the prefetched page.
- If the browser initiatively prefetches `target.com`, the beacon can only be sent to an endpoint on `target.com`.
- If `referer.com` prefetches `target.com`, the beacon can only be sent to an endpoint on `target.com`.
- Reporting to the referrer `referer.com` or any third party `other.com` is prohibited.

This ensures that only the site being prefetched—which already knows the user is visiting—receives the signal.

### Credentialless Beacons
Activation beacons are sent without cookies, authentication headers, or other stored credentials. This ensures the beacon cannot be used to join user sessions across different security contexts.

## Stakeholder Feedback / Opposition
