# Phase 1: Read HPKE layer and compare with RFC 9180

**Goal:** Audit `lib/src/hpke.dart` against RFC 9180 to identify deviations, missing guards, and test-seam risks.

## Context

**Feature motivation (from idea):** Investigation of `ohttp_dart` for potential use in a non-custodial crypto wallet. The goal is not to implement improvements but to identify risks and form subsequent engineering tasks. Every finding must map to a specific file, line range, and RFC section.

**Severity classification (from vision §2):**
- `BLOCKER` — must fix before wallet use
- `HIGH` — should fix before wallet use
- `IMPROVEMENT` — desirable but not blocking

**Technical approach (from vision §4):** `lib/src/hpke.dart` is the bottom layer of a four-layer stack. It wraps `package:cryptography` primitives (X25519, Hmac.sha256, AesGcm.with128bits) into the labeled KDF, KEM Encap, key schedule, and a stateful `HpkeSenderContext`. The dependency graph is strictly acyclic — `hpke` is imported by `ohttp.dart` only.

**Primary investigation concerns (from vision §4 scrutiny table):**
- Single-suite hard-coding (DHKEM(X25519, HKDF-SHA256) + AES-128-GCM)
- Sequence-number counter not guarded against overflow with a typed exception
- `testKeyPair` injection leaves a test seam in the production code path

**Related classes/entities (from vision §5.3 — `HpkeSenderContext`):**

| Field | Type | Investigation concern |
|---|---|---|
| `key` | `Uint8List` (16 B) | AES-128 key; no zeroization after `seal()` |
| `baseNonce` | `Uint8List` (12 B) | Copy made per `_computeNonce()` call — correct |
| `exporterSecret` | `Uint8List` (32 B) | Retained for `export()`; no zeroization |
| `_seq` | `int` | Overflow guard throws `StateError`, not a typed catchable exception |

**Data flow relevant to this phase (from vision §4):**
```
ohttpEncapsulate()
  → HpkeSender.setupBaseS()     # DHKEM(X25519) Encap + HPKE key schedule
  → ctx.seal(aad=[], plaintext) # AES-128-GCM, seq-nonce XOR
  → ctx.export("message/bhttp response", 16)
```

## Tasks

- [ ] 1.1 Read the full file (use `ast-index outline lib/src/hpke.dart` first, then read targeted slices).
- [ ] 1.2 Verify that `LabeledExtract` and `LabeledExpand` match the labeled-KDF construction in RFC 9180 §4 (suite ID, label strings, lengths).
- [ ] 1.3 Verify that `KEM Encap` matches RFC 9180 §4.1 (DH step, `ExtractAndExpand`, `shared_secret` derivation).
- [ ] 1.4 Verify that `setupBaseS` key-schedule matches RFC 9180 §5.1 (`key_schedule_s`, `mode_base = 0`).
- [ ] 1.5 Verify nonce derivation in `HpkeSenderContext.seal` matches RFC 9180 §5.2 (XOR of `baseNonce` with counter as big-endian `Nn`-byte integer).
- [ ] 1.6 Verify `HpkeSenderContext.export` matches RFC 9180 §5.3 (`LabeledExpand(exporterSecret, "sec", context, L)`).
- [ ] 1.7 Note whether `_seq` overflow guard throws a typed exception that callers can catch cleanly.
- [ ] 1.8 Note the `testKeyPair` injection point: confirm it is reachable from production call sites (`setupBaseS` optional parameter).
- [ ] 1.9 Note absence of zeroization for `key`, `baseNonce`, `exporterSecret` after use.
- [ ] 1.10 Record each finding as a draft task entry (file, line range, RFC section, severity: BLOCKER / HIGH / IMPROVEMENT).

## Acceptance Criteria

**Test:** No code is changed. Verification is: all findings are supported by a specific line reference in `hpke.dart` and an RFC section number. Review your notes against the scrutiny table in `vision.md §4` before proceeding.

## Dependencies

- No prior phase dependency — this is the first iteration.

## Technical Details

**HPKE Base Mode (RFC 9180) key schedule overview:**

The labeled KDF construction uses a suite ID prefix (`HPKE\x00\x00` + KEM/KDF/AEAD IDs) to domain-separate all `LabeledExtract` and `LabeledExpand` calls. Each call embeds a human-readable label string and a context value.

The key schedule (`setupBaseS` / `key_schedule_s`) derives:
- `sharedSecret` from KEM Encap
- `keyScheduleContext = mode || ks_context`
- `secret = LabeledExtract(sharedSecret, "secret", psk)`
- `key = LabeledExpand(secret, "key", keyScheduleContext, Nk)`
- `baseNonce = LabeledExpand(secret, "base_nonce", keyScheduleContext, Nn)`
- `exporterSecret = LabeledExpand(secret, "exp", keyScheduleContext, Nh)`

Nonce derivation per seal (RFC 9180 §5.2):
- `nonce = baseNonce XOR I2OSP(_seq, Nn)` where `Nn = 12` for AES-128-GCM
- `_seq` must be incremented and must not wrap around (overflow → error)

**Fixed cipher suite IDs (from `hpke.dart` / `ohttp.dart`):**
- KEM ID: `0x0020` (DHKEM(X25519, HKDF-SHA256)), `Npk = 32`
- KDF ID: `0x0001` (HKDF-SHA256), `Nh = 32`
- AEAD ID: `0x0001` (AES-128-GCM), `Nk = 16`, `Nn = 12`

**Known deviation from vision §4:** Response decapsulation in `ohttp.dart` uses plain (unlabeled) `HKDF-Extract`/`HKDF-Expand` (separate code paths in `hpke.dart`), not `LabeledExtract`/`LabeledExpand`. This is intentional per RFC 9458 §4.4 and must not be unified.

## Implementation Notes

This is a read-only audit iteration. Do not edit any file in `lib/` or `test/`. Record all findings as draft task entries in this file or a scratch document for use in Iteration 7 (compile into per-task Markdown files).
