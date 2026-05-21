# Task 05: Zeroize OHTTP `enc` and `exportedSecret` after response decap

**Severity:** HIGH
**Vector:** Privacy risks
**Files:** `lib/src/ohttp.dart:105-120, 196-207`

**Evidence:** Cross-refs TASK-O6, P6-4. `OhttpEncapsulateResult` (lines 105–120) stores `enc`, `encRequest`, and `exportedSecret` as `final Uint8List` fields. Lines 196–207 are the HKDF key/nonce derivation block inside `ohttpDecapsulate` that consumes `exportedSecret`. None of these buffers are zeroized after use.

## Description

`exportedSecret` is the response-channel root secret; `enc` is the ephemeral KEM share. Both are sensitive: leakage of `exportedSecret` recovers the response AEAD key and breaks confidentiality of the gateway reply (which may contain wallet balances or transaction history); leakage of `enc` alongside ciphertext breaks the response-channel unlinkability property. They are currently retained for the lifetime of the `OhttpEncapsulateResult` and beyond, which is longer than necessary. This task is a peer to task 04 — the two together cover the full set of sensitive in-process buffers.

## Proposed change

Add an explicit zeroization step that overwrites `enc` and `exportedSecret` once `ohttpDecapsulate` has derived the AEAD key/nonce and verified the ciphertext. The `OhttpEncapsulateResult` API should either expose a `dispose()`-style method or be reshaped so the lifetime of `exportedSecret` ends as soon as decap completes. Same Dart best-effort caveat as task 04 applies.

## Acceptance criteria

- After `ohttpDecapsulate` completes (success or failure), `enc` and `exportedSecret` buffers held by the call site are overwritten with zeros.
- The intermediate HKDF outputs (the derived AEAD key and nonce inside `ohttpDecapsulate`) are also zeroized after use.
- Doc comment notes the Dart best-effort limitation.
- Unit test asserts the zeroization behaviour on both happy-path and decryption-failure paths.
