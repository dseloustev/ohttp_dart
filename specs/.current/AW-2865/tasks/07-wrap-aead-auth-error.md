# Task 07: Wrap `SecretBoxAuthenticationError` in a library-owned exception

**Severity:** HIGH
**Vector:** Interoperability
**Files:** `lib/src/ohttp.dart:217-225`

**Evidence:** Cross-refs TASK-O5, P6-7. Lines 217–225 instantiate `AesGcm.with128bits()` and call `aesGcm.decrypt(...)`. On AEAD authentication failure, `decrypt` throws `SecretBoxAuthenticationError` from `package:cryptography`. This type is not re-exported by `lib/ohttp_dart.dart`.

## Description

Callers who want to specifically catch authentication failures (for example, to surface a "tampered response" error to the wallet UI) must add a transitive `package:cryptography` import. That makes `package:cryptography` an unintended part of the public API surface: any future major-version change in `package:cryptography` becomes a breaking change for downstream consumers. The library should own its own exception hierarchy so that the transitive dependency stays an implementation detail.

## Proposed change

Introduce a library-owned exception (e.g., `OhttpDecryptionException`) that wraps the underlying `SecretBoxAuthenticationError`. Catch the transitive exception inside `ohttpDecapsulate` and re-throw the library-owned wrapper. The wrapper should expose enough context (e.g., "response AEAD authentication failed") without leaking the internal cause type name into the public contract. Re-export the new exception from `lib/ohttp_dart.dart`.

## Acceptance criteria

- `ohttpDecapsulate` no longer surfaces `SecretBoxAuthenticationError` directly to callers.
- A library-owned exception is exported from `lib/ohttp_dart.dart`.
- Doc comment for `ohttpDecapsulate` documents the new exception type.
- Unit test triggers an AEAD authentication failure and verifies the wrapper exception is thrown (not the underlying cryptography library type).
- See task 19 for the companion test-only finding.
