# Typed `OhttpException` hierarchy and KeyConfig multi-suite negotiation

**Estimate:** 3d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

After the L-01 restructure, the package has one new exception type (`OhttpGatewayException`, defined in `lib/src/transport.dart`) and otherwise still surfaces error conditions through bare `Exception`, `FormatException`, `UnsupportedError`, and `SecretBoxAuthenticationError` from `package:cryptography`. This task lands a typed `OhttpException` hierarchy that subsumes the existing `OhttpGatewayException`, fixes the broken KeyConfig multi-suite parser in `ohttp.dart`, and unifies the two unrelated exception types that the current parser throws for the same conceptual condition:

1. **No typed exception hierarchy.** The `http`-adapter `HttpClientTransport.fetchKeyConfig` / `postToGateway` (in `lib/src/adapters/http/http_transport.dart`) throws `OhttpGatewayException` with a status code on non-200 responses — good — but every other error path is untyped:
   - Network errors (DNS failure, connection reset) surface as `package:http` / `dart:io` exceptions.
   - AEAD authentication failures bubble up as `SecretBoxAuthenticationError` from `package:cryptography` (via `aesGcm.decrypt(...)` inside `ohttpDecapsulate` in `lib/src/ohttp.dart`), making the transitive dependency part of the public contract.
   - BHTTP/OHTTP parse failures throw `FormatException` (good, but not under a library-owned base).
   - `OhttpGatewayException` (already in core) does not extend a shared library base, so consumers cannot write a single `on OhttpException` clause to cover all library-thrown errors.

2. **KeyConfig multi-suite negotiation is broken.** RFC 9458 §4.1 permits a gateway to advertise multiple KDF+AEAD pairs in the symmetric algorithms section (`symLen > 4`). `OhttpKeyConfig.parse()` reads `symLen`, performs a bounds check, then consumes exactly one 4-byte (`kdfId`, `aeadId`) pair and returns, silently dropping any trailing pairs. A gateway that advertises an unsupported suite first followed by the supported suite will cause the parser to lock onto the first pair and fail validation — or silently succeed on a suite the library does not implement.

3. **Companion mismatch in unsupported-suite errors.** `parse()` throws `FormatException('Unsupported KEM: ...')`, while `validate()` throws `UnsupportedError` for the same conceptual condition. Two unrelated exception hierarchies for the same error class force callers to write two `catch` blocks or fall back to broad `catch (e)`.

Doing these together avoids re-churning the exception surface: the multi-suite fix introduces new throw sites that should already use the typed hierarchy.

## Technical Details

### Part A — Typed exception hierarchy

**1. Define the base + subclasses**

In `lib/src/exceptions.dart` (new file) define:

- `abstract class OhttpException implements Exception` — common base, exposes `message`.
- `class OhttpHttpException extends OhttpException` — non-2xx response from the relay or gateway. Carries `statusCode` and an optional `body` excerpt. **This is the type `HttpClientTransport` throws** (replacing the existing `OhttpGatewayException` — see §2 below).
- `class OhttpTimeoutException extends OhttpException` — request timeout. Consumed by the timeout work in L-04.
- `class OhttpParseException extends OhttpException` — wraps BHTTP/OHTTP parse failures. Adapter for `FormatException` at library boundaries where appropriate; raw `FormatException` from `bhttp.dart` may still surface from `serializeRequest` / `parseResponse` (they document `FormatException` directly).
- `class OhttpKeyConfigException extends OhttpException` — KeyConfig-specific parsing failures (malformed `symLen`, unknown structure).
- `class OhttpDecryptionException extends OhttpException` — wraps the `SecretBoxAuthenticationError` thrown by `aesGcm.decrypt(...)` inside `ohttpDecapsulate`. Stores the original cause without leaking the cryptography-package type name into the public API.
- `class OhttpUnsupportedSuiteException extends OhttpException` — unsupported KEM/KDF/AEAD. Replaces `parse()`'s `FormatException('Unsupported KEM: ...')` and `validate()`'s `UnsupportedError`.
- `class OhttpConfigException extends OhttpException` — input validation failures from L-02 (HTTPS scheme, authority shape). If L-02 lands first with a placeholder type, fold it into this hierarchy here.

**2. Subsume `OhttpGatewayException`**

The L-01 restructure defines `OhttpGatewayException` in `lib/src/transport.dart` with `statusCode` and `message`. Rename it to `OhttpHttpException` and move it into the hierarchy as a subclass of `OhttpException`. The transport interface contract in `OhttpTransport.postToGateway` updates accordingly — implementations MUST throw `OhttpHttpException` on non-2xx so `OhttpSession` can invalidate the cache.

Update `OhttpSession.send` catch clause: `on OhttpHttpException` (instead of `on OhttpGatewayException`). Verify with `ast-index usages "OhttpGatewayException"` that no stale reference remains.

**3. Retarget throw sites**

- `lib/src/adapters/http/http_transport.dart`: `OhttpGatewayException(...)` → `OhttpHttpException(...)` on both `fetchKeyConfig` (non-200 on KeyConfig GET) and `postToGateway` (non-200 on gateway POST).
- `lib/src/ohttp.dart` AEAD decrypt site inside `ohttpDecapsulate`: wrap `SecretBoxAuthenticationError` and re-throw as `OhttpDecryptionException` preserving the cause.
- `lib/src/ohttp.dart` `OhttpKeyConfig.parse` and `OhttpKeyConfig.validate` (unsupported KEM/KDF/AEAD paths): throw `OhttpUnsupportedSuiteException` from both — same type, regardless of which code path detects the unsupported value.
- `lib/src/ohttp.dart` `OhttpKeyConfig.parse` (malformed wire format): throw `OhttpKeyConfigException`.

**4. Export the hierarchy**

Add `export 'src/exceptions.dart';` to `lib/ohttp_dart.dart`. Remove the now-unused `OhttpGatewayException` export if it survived under the old name.

Update doc comments on `OhttpSession.send`, `OhttpTransport.postToGateway`, `OhttpTransport.fetchKeyConfig`, `ohttpDecapsulate`, `OhttpKeyConfig.parse`, and `OhttpKeyConfig.validate` to enumerate the exceptions each may throw.

### Part B — KeyConfig multi-suite parsing

**5. Rework `OhttpKeyConfig.parse` multi-pair handling** (`lib/src/ohttp.dart`)

Inspect every KDF+AEAD pair in the symmetric algorithms section. Pick one of two policies and document it in the function doc comment with a reference to RFC 9458 §4.1:

(a) **Iterate-and-select.** Walk all `(kdfId, aeadId)` pairs in `symLen / 4` order; select the first pair that matches the library's single supported suite (`kdfId == 0x0001`, `aeadId == 0x0001`). If none match, throw `OhttpUnsupportedSuiteException` naming the advertised pairs.

(b) **Strict-single-pair.** Require `symLen == 4` and exactly one supported pair. Throw `OhttpUnsupportedSuiteException` when `symLen > 4` is implied by the wire format.

Pick (a) unless there's a specific reason to refuse multi-pair configs — (a) is friendlier to gateways that advertise legacy suites alongside the supported one.

**6. Reject malformed `symLen`** (`lib/src/ohttp.dart`)

`symLen` not divisible by 4 indicates a malformed wire format. Reject with `OhttpKeyConfigException` (typed) carrying the offending length.

### Part C — Tests

**7. Hierarchy tests** (`test/adapters/http_adapter_test.dart`, `test/ohttp_test.dart`)

- `MockClient` returns 4xx for `keysUrl` → `HttpClientTransport.fetchKeyConfig` throws `OhttpHttpException` with `statusCode` in the 4xx range.
- `MockClient` returns 5xx for `gatewayUrl` → `HttpClientTransport.postToGateway` throws `OhttpHttpException` with `statusCode` in the 5xx range.
- The same exception is catchable as `on OhttpException`.
- Produce a valid OHTTP encap, derive the response secret in-test, encrypt a response, flip one byte in the ciphertext, call `ohttpDecapsulate`, assert `OhttpDecryptionException` is thrown (not the underlying `SecretBoxAuthenticationError`). Assert the cause field carries the original error.
- `OhttpSession.send` invalidates the cache when the transport throws `OhttpHttpException` (still works — the catch is now on `OhttpHttpException`, not `OhttpGatewayException`).

**8. Multi-suite tests** (`test/ohttp_test.dart`)

Construct synthetic KeyConfig byte buffers in test fixtures (documented byte arrays so the wire format is explicit). Cover:

- Single supported pair (current happy path).
- Two pairs where the supported pair is at index 0 — `parse` returns successfully with the supported pair selected.
- Two pairs where the supported pair is at index 1 — `parse` returns successfully (this currently fails).
- Two or three pairs where no supported suite is present — `OhttpUnsupportedSuiteException` thrown, naming the advertised pairs.
- Malformed `symLen` not divisible by 4 — `OhttpKeyConfigException`.
- Unknown KEM via `parse()` — `OhttpUnsupportedSuiteException`.
- Unknown KEM/KDF/AEAD via `validate()` — `OhttpUnsupportedSuiteException` (same type as the `parse()` path).

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

**Hierarchy:**
- `lib/src/exceptions.dart` defines `OhttpException` plus at least the subclasses listed in §1. All are re-exported from `lib/ohttp_dart.dart`.
- The L-01 `OhttpGatewayException` is renamed to `OhttpHttpException` and brought under `OhttpException`. `OhttpTransport.postToGateway` doc comment is updated to reflect the new contract.
- `HttpClientTransport.fetchKeyConfig` and `HttpClientTransport.postToGateway` throw `OhttpHttpException` (carrying `statusCode`) on non-200.
- `ohttpDecapsulate` no longer surfaces `SecretBoxAuthenticationError`; it surfaces `OhttpDecryptionException` wrapping the original cause.
- `OhttpKeyConfig.parse` and `OhttpKeyConfig.validate` both throw `OhttpUnsupportedSuiteException` for the unsupported-suite condition.
- `OhttpKeyConfig.parse` throws `OhttpKeyConfigException` for malformed `symLen`.
- Doc comments on `OhttpSession.send`, `OhttpTransport.fetchKeyConfig` / `postToGateway`, `ohttpDecapsulate`, `OhttpKeyConfig.parse` / `validate` enumerate the exceptions they may throw.

**Multi-suite:**
- `OhttpKeyConfig.parse` no longer ignores trailing bytes when `symLen > 4`; it iterates all `(kdfId, aeadId)` pairs and selects a supported one, or throws `OhttpUnsupportedSuiteException` per the documented policy.
- Doc comment on `parse()` cites RFC 9458 §4.1 and documents the chosen negotiation policy.
- Malformed `symLen` (not divisible by 4) is rejected with `OhttpKeyConfigException`.

**Tests:**
- Hierarchy: 4xx → `OhttpHttpException`(4xx); 5xx → `OhttpHttpException`(5xx); both catchable as `on OhttpException`; AEAD auth failure → `OhttpDecryptionException`; `OhttpSession.send` cache invalidation still triggers on `OhttpHttpException`.
- Multi-suite: single pair, supported at index 0, supported at index 1, no supported pair (typed exception), malformed `symLen` (typed exception), unsupported KEM via both `parse()` and `validate()`.

## Additional

- Tests: unit tests for each typed exception path and each multi-suite wire-format case enumerated above.
- Depends on L-01 (the `OhttpTransport`, `OhttpGatewayException`, `OhttpSession`, `HttpClientTransport` types must exist before they can be rehomed under the new hierarchy).
- Migration: external consumers catching the L-01 `OhttpGatewayException` must update to `OhttpHttpException`. Since this restructure has no released consumers yet (the package's only `publish_to` is `none`), the rename is a free move.
- This task should land second (after L-01), so subsequent tasks (L-04, L-05, L-07) emit their new throws onto the typed surface from day one.
- Source merged tasks (in `../merged-tasks/`): M-05 (typed exception hierarchy), M-04 (KeyConfig multi-suite + unsupported-suite unification). The throw-site locations from M-05 (originally `ohttp_client.dart:84-87, 120-122`) are retargeted to `HttpClientTransport` as documented above.
