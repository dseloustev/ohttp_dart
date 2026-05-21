# Task 18: Test OHTTP encap/decap round-trip

**Severity:** HIGH
**Vector:** Test suite
**Files:** `test/ohttp_test.dart`

**Evidence:** Cross-ref T-O4. `test/ohttp_test.dart` currently has 165 lines; it covers `OhttpKeyConfig.parse`/`validate` and the encapsulate structure plus a decapsulate short-circuit, but no end-to-end encap → decap round-trip with a synthetic gateway-side responder.

## Description

The current tests verify the encap output structure and the decap path's short-circuit logic in isolation. A true round-trip test — produce ciphertext via `ohttpEncapsulate`, run the receiver side (synthetic) to derive the response secret, encrypt a response with that secret, then verify `ohttpDecapsulate` recovers the original plaintext — is the missing piece. Without it, a regression that breaks the response-secret derivation (the plain-HKDF path noted in task 29) lands silently.

## Proposed change

Add a unit test that performs the full client-side encap, runs a small in-test helper to emulate the gateway-side response-channel derivation, encrypts a canned response, and asserts that `ohttpDecapsulate` returns the expected plaintext. The helper should use the same primitives the library exposes (`HpkeSender.hkdfExtract`/`hkdfExpand`) so the test doubles as documentation of the response-channel contract.

## Acceptance criteria

- New end-to-end round-trip test in `test/ohttp_test.dart`.
- Test exercises the full encap → response-secret derivation → response encryption → decap chain.
- Helper code is self-contained (no external gateway dependency).
- On success the decapsulated plaintext exactly equals the original plaintext.
- Test name references RFC 9458 §4.3 / §4.4.
