# Zeroize HPKE and OHTTP sensitive key material

**Estimate:** 1d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

A wallet running on a mobile or desktop host is a credible memory-dump target. Sensitive HPKE and OHTTP outputs currently linger in heap-allocated `Uint8List` fields long after the OHTTP request completes, which broadens the post-request attack surface unnecessarily. The OHTTP send flow is one-shot — there is no functional reason to retain these buffers — but Dart GC does not guarantee prompt collection, so an attacker who acquires a memory snapshot of the wallet process minutes after a request can recover the AEAD key, base nonce, exporter secret, KEM ephemeral share, and response-channel root secret.

This task covers the full set of sensitive in-process buffers across both `hpke.dart` and `ohttp.dart`.

## Technical Details

**1. HPKE sender context** (`lib/src/hpke.dart`, `HpkeSenderContext` fields at lines 258–263)

`HpkeSenderContext` retains `key` (16 B, AES-128), `baseNonce` (12 B), and `exporterSecret` (32 B) as `final Uint8List` fields. None are zeroized.

Provide an explicit zeroization step (method or `dispose`-style call) that overwrites each `Uint8List` field with zeros once the context is no longer needed. Callers in `ohttp.dart` invoke the step after `seal()` and `export()` complete.

**2. OHTTP encap result + intermediate HKDF outputs** (`lib/src/ohttp.dart:105-120, 192-207`)

`OhttpEncapsulateResult` (lines 105–120) stores `enc`, `encRequest`, and `exportedSecret` as `final Uint8List` fields. `exportedSecret` is the response-channel root secret; `enc` is the ephemeral KEM share. Leakage of `exportedSecret` recovers the response AEAD key and breaks confidentiality of the gateway reply (which may contain wallet balances or transaction history); leakage of `enc` alongside ciphertext breaks the response-channel unlinkability property.

Lines 192–207 are the HKDF key/nonce derivation block inside `ohttpDecapsulate` that consumes `exportedSecret`. The derived AEAD key and nonce intermediates are also sensitive.

Expose a `dispose()`-style method on `OhttpEncapsulateResult` (or reshape so the lifetime ends as soon as decap completes). After `ohttpDecapsulate` returns (success or failure), `enc` and `exportedSecret` buffers held by the call site, plus the derived AEAD key and nonce intermediates, are overwritten with zeros.

**3. Dart caveat**

AOT/JIT compilers may optimize away the write. The mitigation is best-effort and the doc comment must say so. Evaluate whether `package:cryptography` exposes a `SecretKey.destroy()`-style API and prefer it if available.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

- `HpkeSenderContext` exposes an explicit zeroization step (method or `dispose`-style call) that overwrites `key`, `baseNonce`, and `exporterSecret` with zeros.
- Callers in `ohttp.dart` invoke the zeroization step after the seal/export operations complete.
- `OhttpEncapsulateResult` exposes a zeroization API that wipes `enc` and `exportedSecret`.
- After `ohttpDecapsulate` completes (success or failure), `enc`, `exportedSecret`, and the derived AEAD key/nonce intermediates are overwritten with zeros.
- Doc comments on each zeroization API document the Dart best-effort limitation.
- Unit test verifies that after the zeroization call, each field's bytes are all zero (or that the field reference is replaced with a zeroed buffer), on both happy-path and decryption-failure paths.

## Additional

- Tests: unit tests asserting zeroization on each layer; cover both successful seal/decap and failure paths.
