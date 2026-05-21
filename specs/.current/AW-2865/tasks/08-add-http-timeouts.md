# Task 08: Add timeouts to KeyConfig GET and gateway POST

**Severity:** HIGH
**Vector:** Network reliability
**Files:** `lib/src/ohttp_client.dart:81-83, 115-119`

**Evidence:** Cross-ref C-6. Lines 81–83 contain `_httpClient.get(Uri.parse(...))` with no `.timeout(Duration(...))` and no retry policy. Lines 115–119 contain `_httpClient.post(...)` with the same shape. A non-responsive gateway can stall an `await` indefinitely.

## Description

A wallet that issues an OHTTP request without timeouts can hang forever if the relay is slow, unreachable, or under attack. For an interactive wallet UI, this surfaces as a frozen screen with no progress indication and no escape hatch — the user must kill the app. Defensive timeouts and a retry policy are standard hygiene for any HTTP client embedded in a user-facing wallet.

## Proposed change

Add configurable timeouts (with sensible defaults) on both HTTP calls in `OhttpClient.send()`. Allow the caller to override the defaults via `OhttpGatewayConfig` (or via a new `OhttpClientOptions` shape). On timeout, throw a typed library exception (see task 10 for the typed-error hierarchy). Decide separately on retry semantics — the proposed change should at minimum document the absence of retries and the caller's responsibility, even if no automatic retry is added.

## Acceptance criteria

- Both `_httpClient.get` and `_httpClient.post` calls in `OhttpClient.send` chain `.timeout(...)` with a configurable duration.
- Default timeout is documented in the `OhttpGatewayConfig` doc comment.
- Timeout produces a typed library exception (coordinates with task 10).
- Unit test simulates a slow gateway (mock HTTP client) and verifies the configured timeout fires.
