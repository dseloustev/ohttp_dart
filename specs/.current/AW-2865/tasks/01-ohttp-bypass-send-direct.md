# Task 01: OHTTP bypass via silent `effectiveDirectBaseUrl` fallback

**Severity:** BLOCKER
**Vector:** Privacy risks
**Files:** `lib/src/ohttp_client.dart:52, 147, 148-171`

**Evidence:** Cross-refs P6-1, C-4. `sendDirect()` carries a single doc line ("Send a direct HTTP request (for comparison)."), no `@Deprecated`, no `assert`, and no privacy-impact warning. `effectiveDirectBaseUrl` at line 52 silently resolves a null `directBaseUrl` to `gatewayBaseUrl` (the OHTTP relay). Verified against branch `feature/AW-2865-investigation-ohttp_dart`: `sendDirect()` body spans lines 148–171; the doc comment is line 147.

## Description

A developer who constructs `OhttpGatewayConfig` with only the required parameters (omitting the optional `directBaseUrl`) and then calls `sendDirect()` issues plaintext HTTP directly to the relay host — no encryption, no unlinkability, no runtime signal that the OHTTP flow was bypassed. The fallback default at line 52 (`directBaseUrl ?? gatewayBaseUrl`) is the silent mechanism that converts a "comparison" helper into a privacy-breaking footgun. For a non-custodial wallet, this is the difference between an unlinkable RPC call and a plaintext leak of wallet activity to the relay operator. This task is the companion BLOCKER to task 32, which removes the public surface of the same fallback.

## Proposed change

Eliminate the silent fallback path. `sendDirect()` must either be removed from the production library or require an explicit, non-null `directBaseUrl` distinct from `gatewayBaseUrl`. At minimum, the public API must communicate the privacy impact in three independent ways: a `@Deprecated` annotation, an assertion that prevents the relay host from being used as the direct target, and prominent doc-comment text describing the consequences of calling it. See companion task 32 for the API-surface change that prevents discovery of the fallback through the public getter.

## Acceptance criteria

- Calling `sendDirect()` without an explicit `directBaseUrl` that differs from `gatewayBaseUrl` fails fast (assertion or thrown library-owned exception), never silently sending plaintext to the relay.
- The `sendDirect()` doc comment names the privacy impact (no encryption, no unlinkability) and references the OHTTP guarantees that are bypassed.
- A `@Deprecated` annotation (or equivalent compile-time signal) is present on the method to discourage production use.
- Unit test exercises the silent-fallback configuration and asserts that the call is rejected.
- The companion change in task 32 is referenced from the task description so engineers see both as a pair.
