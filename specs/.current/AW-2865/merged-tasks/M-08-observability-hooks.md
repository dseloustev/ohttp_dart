# Add structured observability hooks and normalize response header names

**Estimate:** 2d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

The library has no structured observability surface. There is one optional `onLog` string callback on `OhttpClient` that is used inconsistently and currently leaks sensitive path information: at `lib/src/ohttp_client.dart:77` it logs `gatewayBaseUrl + configPath`, and at line 113 it logs `gatewayBaseUrl + requestPath`. Wallet operators need actionable signals (KeyConfig fetch outcome, gateway POST status, decryption success/failure, accidental `sendDirect()` invocation) to diagnose production issues, but those signals must not include any cryptographic material, the inner request, the target authority, or the response body. The current ad-hoc string logging is both insufficient (no structure, no levels) and unsafe (mixes safe and unsafe fields into a single string).

Separately, response header names are not case-normalized at the OHTTP client boundary (`lib/src/ohttp_client.dart:139`): `e.key` comes directly from `streamedResponse.headers.entries`. HTTP/1.1 header names are case-insensitive (RFC 9110 §5.1), but different HTTP clients deliver them in different cases. Downstream consumers that compare header names with `==` or look up by exact string see inconsistent behaviour depending on which version of the gateway, the relay, or the transport stack is in play.

## Technical Details

**1. Typed observability interface**

Define a typed observer interface (e.g., `OhttpObserver`) with event methods such as:
- `onKeyConfigFetch({required int statusCode, required Duration elapsed})`
- `onGatewayPost({required int statusCode, required Duration elapsed})`
- `onSendDirectInvoked()` — for accidental fallback alerting (companion to the sendDirect privacy task).
- `onDecryptionFailure()` — emitted before/around the AEAD-failure exception path in `ohttpDecapsulate`.

Re-export from `lib/ohttp_dart.dart`.

Replace the existing string `onLog` callback in `OhttpClient`. Either remove it (breaking change, documented in migration note) or deprecate it alongside the new typed surface and document the migration path. The existing emission sites at lines 77 and 113 — which log full URLs containing both base and path — must be replaced by typed events that carry only the status code.

**2. Strict logging constraints**

Emission sites must obey these constraints (reproduced verbatim in the public doc comment of the observer interface so they cannot be lost in a refactor):

> Never log: (1) key material, (2) enc value, (3) inner request URL/path/method/headers/body, (4) targetAuthority, (5) response body/headers, (6) gatewayBaseUrl at DEBUG or lower in production.
>
> Safe to log: KeyConfig fetch success/failure (status code only), gateway POST status code (not body), sendDirect() invocation signal.

**3. Lowercase response header names** (`lib/src/ohttp_client.dart:139`)

At line 139 (`.map((e) => OhttpHeader(name: e.key, value: e.value))`), lowercase the header name before constructing `OhttpHeader`. Document the normalization in the `OhttpHeader` doc comment so downstream consumers know they can rely on it. Cite RFC 9110 §5.1.

**4. Tests**

- Drive a successful `OhttpClient.send` flow against a mock client and capture every emitted event. Assert that no event payload contains: gateway path, target authority, response body, key material strings, `enc` bytes.
- Drive a failing flow (4xx, 5xx, AEAD auth failure) and assert the same. Assert that the failure-specific events fire with the correct status code.
- Drive `sendDirect()` and assert that `onSendDirectInvoked()` fires.
- Feed mixed-case response header names through `OhttpClient.send` and assert the resulting `OhttpHeader.name` values are lowercase.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

- A typed observability interface (e.g., `OhttpObserver`) is defined and re-exported from `lib/ohttp_dart.dart`.
- All emission sites in `OhttpClient` produce structured events that obey the logging constraints above — no key material, no `enc`, no inner request details, no `targetAuthority`, no response body/headers, no full `gatewayBaseUrl + path` strings.
- The existing string `onLog` callback is either removed (breaking change, documented) or deprecated with a migration note.
- A `sendDirect()` invocation produces a distinct, documented event so wallet operators can alert on accidental fallback use.
- The observer interface's public doc comment reproduces the logging constraints verbatim.
- Response header names are lowercased at `lib/src/ohttp_client.dart:139` before constructing `OhttpHeader`.
- `OhttpHeader` doc comment documents that `name` is lowercase and cites RFC 9110 §5.1.
- Unit test verifies no emitted event contains any restricted field for a successful and a failing flow.
- Unit test verifies mixed-case response header names are normalized to lowercase.

## Additional

- Tests: observer-event capture tests; header-normalization tests.
- Migration note: removal/deprecation of `onLog` is breaking; document the new surface clearly.
