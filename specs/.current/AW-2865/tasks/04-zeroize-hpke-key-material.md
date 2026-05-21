# Task 04: Zeroize HPKE ephemeral key material in `HpkeSenderContext`

**Severity:** HIGH
**Vector:** Privacy risks
**Files:** `lib/src/hpke.dart` (HpkeSenderContext fields, lines 258-263)

**Evidence:** Cross-refs Phase 1 F-2, P6-3. `HpkeSenderContext` retains `key` (16 B AES-128), `baseNonce` (12 B), and `exporterSecret` (32 B) as `final Uint8List` fields. Dart GC does not guarantee prompt collection; in a wallet binary subject to memory inspection these buffers may persist long after the OHTTP request completes.

## Description

A wallet running on a mobile or desktop host is a credible memory-dump target. Sensitive HPKE outputs that linger in heap-allocated `Uint8List` fields broaden the post-request attack surface unnecessarily — the OHTTP send flow is one-shot, so there is no functional reason to keep these buffers alive after `seal()` returns. Without a zeroization step, an attacker who acquires a memory snapshot of the wallet process minutes after a request can recover the AEAD key, base nonce, and exporter secret.

## Proposed change

Provide a documented best-effort zeroization step on `HpkeSenderContext` and on the higher-level `OhttpEncapsulateResult.exportedSecret` flow (see task 05). The mechanism should overwrite each `Uint8List` field with zeros once the context is no longer needed. The task file should also note Dart-specific caveats: AOT/JIT compilers may optimize away the write, so the mitigation is best-effort and should be paired with a doc comment that explains the limitation. Evaluate whether `package:cryptography` exposes a stronger `SecretKey.destroy()`-style API and prefer it if available.

## Acceptance criteria

- `HpkeSenderContext` exposes an explicit zeroization step (method or `dispose`-style call) that overwrites `key`, `baseNonce`, and `exporterSecret` with zeros.
- Callers in `ohttp.dart` invoke the zeroization step after the seal/export operations complete.
- Doc comment documents that zeroization is best-effort on Dart, with rationale.
- Unit test verifies that after the zeroization call, each field's bytes are all zero (or that the field reference is replaced with a zeroed buffer).
