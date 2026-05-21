# Task 02: Enforce HTTPS scheme on `gatewayBaseUrl`

**Severity:** HIGH
**Vector:** Privacy risks
**Files:** `lib/src/ohttp_client.dart:43-51, 52, 82, 116`

**Evidence:** Cross-refs C-1, P6-2. RFC 9458 §1 requires TLS on the outer channel between client and relay/gateway. `OhttpGatewayConfig` constructor (lines 43–50) accepts `gatewayBaseUrl` as a plain `String` with no `startsWith('https://')` check. The same lack of validation applies to `directBaseUrl`. Line 82 builds the KeyConfig GET URI and line 116 builds the gateway POST URI from these unvalidated strings; line 52 is the public getter that exposes the fallback.

## Description

The library silently accepts `http://` URLs for both `gatewayBaseUrl` and `directBaseUrl`. The outer OHTTP ciphertext is then transmitted in cleartext, which gives a passive on-path observer the ability to perform timing correlation against the gateway and undermines the unlinkability property that OHTTP exists to provide. A wallet caller using a typo'd configuration or a misconfigured staging URL has no signal that the deployment is broken until a network observer is already capturing traffic.

## Proposed change

Validate the URL scheme at `OhttpGatewayConfig` construction time. The constructor must reject any base URL that is not `https://` (with a clearly documented escape hatch for tests, if needed, that does not affect production builds). The validation must cover both `gatewayBaseUrl` and `directBaseUrl`.

## Acceptance criteria

- Constructing `OhttpGatewayConfig` with `http://` (or any non-`https://`) `gatewayBaseUrl` throws a library-owned exception at construction time.
- The same validation rejects `http://` `directBaseUrl` when supplied.
- Doc comment for `OhttpGatewayConfig` cites RFC 9458 §1 and explains the rationale.
- Unit test covers: valid `https://`, rejected `http://`, rejected `ftp://`, rejected empty string.
