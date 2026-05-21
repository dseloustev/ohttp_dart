# Introduce typed `OhttpException` hierarchy and wrap AEAD authentication errors

**Estimate:** 2d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

The library has no typed exception hierarchy. `OhttpClient.send()` at `lib/src/ohttp_client.dart:84-87, 120-122` throws bare `Exception('...')` for both KeyConfig GET and gateway POST non-200 responses, embedding the status code only inside the message string. Network errors (DNS failure, connection reset) surface as untyped exceptions. AEAD authentication failures bubble up as `SecretBoxAuthenticationError` from `package:cryptography` (via `aesGcm.decrypt(...)` at `lib/src/ohttp.dart:217-225`), which makes the transitive dependency part of the public API surface.

Two consequences for a wallet caller:

1. Meaningful error handling — distinguishing a 5xx from a 4xx, a timeout from a parse error, an AEAD authentication failure from a transport failure — is impossible without parsing exception messages or adding broad `catch (e)` blocks. This prevents sensible UI flows like "retry on transient failure, abort on misconfiguration."
2. Any future major-version change in `package:cryptography` becomes a breaking change for downstream consumers who currently catch `SecretBoxAuthenticationError` to surface tampered-response errors to the wallet UI.

## Technical Details

**1. Define the typed hierarchy**

Introduce a common base class `OhttpException` (extends `Exception`) and concrete subclasses:

- `OhttpHttpException(statusCode, body?)` — non-2xx response from the relay or gateway POST. Subdivide or expose enough information that callers can branch on `statusCode >= 400 && < 500` vs `>= 500`.
- `OhttpTimeoutException` — request timeout (coordinates with the timeout work in the network reliability task).
- `OhttpParseException(cause)` — malformed BHTTP/OHTTP/KeyConfig response.
- `OhttpKeyConfigException` — KeyConfig-specific parsing failures.
- `OhttpDecryptionException(cause)` — wraps the underlying `SecretBoxAuthenticationError` thrown by `aesGcm.decrypt(...)` at `lib/src/ohttp.dart:217-225`.
- `OhttpUnsupportedSuiteException` — unsupported KEM/KDF/AEAD (if the parallel multi-suite task is landing separately, slot its exception into this hierarchy under the same base class).

Re-export every type from `lib/ohttp_dart.dart`.

**2. Replace bare throws** (`lib/src/ohttp_client.dart:84-87, 120-122` and `lib/src/ohttp.dart:217-225`)

- Lines 84–87: `if (configResponse.statusCode != 200) { throw Exception('...'); }` → `throw OhttpHttpException(statusCode: configResponse.statusCode, ...)`.
- Lines 120–122: same pattern for the gateway POST.
- Lines 217–225 in `ohttp.dart`: catch `SecretBoxAuthenticationError` inside `ohttpDecapsulate` and re-throw `OhttpDecryptionException` wrapping the original cause without leaking the internal cause type name into the public contract.

Update doc comments on `send()`, `sendDirect()`, and `ohttpDecapsulate` to enumerate the exceptions they may throw.

**3. Tests** (`test/ohttp_test.dart`, `test/ohttp_client_test.dart`)

- Mock `http.Client` to return 4xx → assert `OhttpHttpException` with `statusCode` in the 4xx range.
- Mock `http.Client` to return 5xx → assert `OhttpHttpException` with `statusCode` in the 5xx range.
- Produce a valid OHTTP encap, derive the response secret, encrypt a response, flip one byte in the ciphertext, call `ohttpDecapsulate`, assert that an `OhttpDecryptionException` is thrown (not the underlying `SecretBoxAuthenticationError`).

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

- Library-owned `OhttpException` base class is exported from `lib/ohttp_dart.dart` with at least the subtypes listed above.
- All `throw Exception(...)` sites in `lib/src/ohttp_client.dart` are replaced with typed subclasses carrying the relevant context (status code, cause).
- `ohttpDecapsulate` no longer surfaces `SecretBoxAuthenticationError` directly to callers; it surfaces `OhttpDecryptionException` instead.
- Doc comments on `send()`, `sendDirect()`, and `ohttpDecapsulate` enumerate the exceptions they may throw.
- Unit tests cover: gateway 4xx → typed exception with `statusCode == 4xx`; gateway 5xx → typed exception with `statusCode == 5xx`; AEAD auth failure → `OhttpDecryptionException` (not the transitive cryptography type).

## Additional

- Tests: unit tests for each typed exception path enumerated above.
- Migration note: any downstream caller currently catching `Exception` from `OhttpClient.send` or `SecretBoxAuthenticationError` from decap must migrate to the new typed surface.
