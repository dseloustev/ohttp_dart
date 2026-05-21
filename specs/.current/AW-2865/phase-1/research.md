# Phase 1 Research: HPKE Layer Audit Against RFC 9180
# AW-2865 — ohttp_dart Investigation

**Phase scope:** `lib/src/hpke.dart` only — RFC 9180 Base Mode Sender.
Response-decap helpers (`hkdfExtract` / `hkdfExpand` as consumed by `ohttp.dart`) are covered in Phase 3.
No code in `lib/` or `test/` was modified during this research.

---

## Phase Scope

This phase audits the bottom layer of the four-layer stack: `lib/src/hpke.dart` (314 lines).
The file implements DHKEM(X25519, HKDF-SHA256) + HKDF-SHA256 + AES-128-GCM (KEM `0x0020`, KDF `0x0001`, AEAD `0x0001`) in Base Mode, Sender only.

The audit covers:
- `LabeledExtract` / `LabeledExpand` vs RFC 9180 §4
- KEM Encap vs RFC 9180 §4.1
- `_keySchedule` (key schedule) vs RFC 9180 §5.1
- Nonce derivation in `HpkeSenderContext._computeNonce` / `seal` vs RFC 9180 §5.2
- `HpkeSenderContext.export` vs RFC 9180 §5.3
- Structural risks: `_seq` overflow guard, `testKeyPair` injection, key-material zeroization

Out of scope for this phase: `bhttp.dart`, `ohttp.dart` (except as consumer context), `ohttp_client.dart`, and the plain (unlabeled) `hkdfExtract` / `hkdfExpand` as used in response decapsulation.

---

## Resolved Questions

| # | Question | User Answer |
|---|---|---|
| 1 | `testKeyPair` severity: HIGH or BLOCKER? | **HIGH.** Rationale: requires deliberate caller action to activate; fix is straightforward (assert / test-only factory). Not an automatic threat, but must be remediated before wallet use. |
| 2 | Key-material zeroization severity? | **IMPROVEMENT.** Keys at risk are ephemeral OHTTP session keys, not wallet private keys. A memory dump reveals one request's inner HTTP content (traffic deanonymization), not user funds. Threat model does not elevate this. |
| 3 | Is `HpkeSenderContext` always single-use in the target wallet? | **Yes.** One `seal()` per context, always. `_seq` overflow guard severity is downgraded to **IMPROVEMENT**. |
| 4 | Output format? | Structured findings table (description, file:line range, RFC section, severity) as canonical output. |
| 5 | Additional context documents available? | No additional documents beyond the spec files already read. |
| 6 | Where do draft task entries live? | This `phase-1/research.md` is the canonical source for draft task entries. No separate artifact needed. |

---

## RFC 9180 Section Verdicts

### §4 — Labeled KDF construction

**LabeledExtract** (`hpke.dart` lines 187–200):

RFC 9180 §4 defines:
```
LabeledExtract(salt, label, ikm) =
    Extract(salt, "HPKE-v1" || I2OSP(suite_id, Ns) || label || ikm)
```

Implementation constructs `labeledIkm = "HPKE-v1" || suiteId || label || ikm` then calls `hkdfExtract(salt, labeledIkm)`.
The suite ID used is `_kemSuiteId` (KEM context: `"KEM" || 0x00 0x20`, 5 bytes) for KEM-scoped calls and `_hpkeSuiteId` (`"HPKE" || 0x00 0x20 || 0x00 0x01 || 0x00 0x01`, 10 bytes) for key-schedule calls. This matches the split required by RFC 9180 §4 (KEM operations use KEM suite ID; key schedule uses HPKE suite ID).

**Verdict: matches RFC.**

---

**LabeledExpand** (`hpke.dart` lines 204–220):

RFC 9180 §4 defines:
```
LabeledExpand(prk, label, info, L) =
    Expand(prk, I2OSP(L, 2) || "HPKE-v1" || I2OSP(suite_id, Ns) || label || info, L)
```

Implementation constructs `labeledInfo = I2OSP(L,2) || "HPKE-v1" || suiteId || label || info` then calls `hkdfExpand(prk, labeledInfo, length)`.
Big-endian two-byte length prefix is correct: `(length >> 8) & 0xFF, length & 0xFF`.

**Verdict: matches RFC.**

---

**hkdfExtract empty-salt handling** (`hpke.dart` line 226):

RFC 5869 §2.2: if salt is not provided, a string of `HashLen` zeros is used. `HashLen = 32` for SHA-256.
Implementation: `final effectiveSalt = salt.isEmpty ? Uint8List(32) : salt;` — 32 zero bytes when empty. Correct.

**Verdict: matches RFC.**

---

**hkdfExpand** (`hpke.dart` lines 235–254):

RFC 5869 §2.3: `T(i) = HMAC-SHA-256(PRK, T(i-1) || info || I2OSP(i, 1))`, `OKM = T(1) || ... || T(n)`.
Implementation: `input = [...t, ...info, i]` where `t` starts as empty (T(0)). Correct counter encoding (single byte `i`). Output is truncated to `length` bytes via `sublistView`.

**Verdict: matches RFC.**

---

### §4.1 — KEM Encap (DHKEM X25519)

RFC 9180 §4.1 (DHKEM):
```
Encap(pkR):
    skE, pkE = GenerateKeyPair()
    enc = SerializePublicKey(pkE)
    dh = DH(skE, pkR)
    kem_context = enc || pkR
    shared_secret = ExtractAndExpand(dh, kem_context)
    return shared_secret, enc
```

Implementation (`hpke.dart` lines 69–104):
- Lines 74–79: ephemeral keypair generated via `_x25519.newKeyPair()` or taken from `testKeyPair`. Correct.
- Lines 81–84: `enc` = ephemeral public key bytes (32 bytes for X25519). Correct.
- Lines 87–95: `dh = X25519.sharedSecretKey(skE, pkR)`. Correct.
- Line 98: `kem_context = enc || recipientPkBytes`. Correct byte order.
- Line 101: `ExtractAndExpand(dh, kemContext)` via `_extractAndExpand`. Correct.

`_extractAndExpand` (`hpke.dart` lines 106–123):
- LabeledExtract with KEM suite ID, label `"eae_prk"`, salt = empty, ikm = dh. Correct.
- LabeledExpand with KEM suite ID, label `"shared_secret"`, info = kemContext, L = `_nSecret` (32). Correct.

**Verdict: matches RFC.**

---

### §5.1 — Key Schedule (Base Mode)

RFC 9180 §5.1:
```
key_schedule_s(mode, shared_secret, info, psk="", psk_id=""):
    psk_id_hash = LabeledExtract("", "psk_id_hash", psk_id)
    info_hash = LabeledExtract("", "info_hash", info)
    ks_context = I2OSP(mode, 1) || psk_id_hash || info_hash
    secret = LabeledExtract(shared_secret, "secret", psk)
    key = LabeledExpand(secret, "key", ks_context, Nk)
    base_nonce = LabeledExpand(secret, "base_nonce", ks_context, Nn)
    exporter_secret = LabeledExpand(secret, "exp", ks_context, Nh)
    return Context_S(key, base_nonce, 0, exporter_secret)
```

Implementation (`hpke.dart` lines 127–181):
- Lines 131–137: `psk_id_hash = LabeledExtract(salt="", "psk_id_hash", ikm=Uint8List(0))`. Salt is `Uint8List(0)` which triggers the zero-filled salt path (RFC-correct). `ikm = Uint8List(0)` = empty psk_id in Base Mode. Correct.
- Lines 138–143: `info_hash = LabeledExtract(salt="", "info_hash", ikm=info)`. Correct.
- Line 146: `ks_context = [0x00, ...pskIdHash, ...infoHash]` = `I2OSP(mode_base=0, 1) || psk_id_hash || info_hash`. Correct.
- Lines 149–154: `secret = LabeledExtract(salt=sharedSecret, "secret", ikm=Uint8List(0))`. Salt is `sharedSecret`, ikm is empty psk in Base Mode. Correct (RFC §5.1 uses `sharedSecret` as the salt).
- Lines 156–162: `key = LabeledExpand(secret, "key", ks_context, Nk=16)`. Correct.
- Lines 164–170: `base_nonce = LabeledExpand(secret, "base_nonce", ks_context, Nn=12)`. Correct.
- Lines 172–178: `exporter_secret = LabeledExpand(secret, "exp", ks_context, Nh=32)`. Correct.

**Verdict: matches RFC.**

---

### §5.2 — Nonce Derivation and Sequence Counter

RFC 9180 §5.2:
```
ComputeNonce(base_nonce, seq):
    seq_bytes = I2OSP(seq, Nn)
    return xor(base_nonce, seq_bytes)
```
where `Nn = 12` for AES-128-GCM, and `seq` must not exceed `(1 << (8*Nn)) - 1`.

Implementation (`hpke.dart` lines 274–286):
- Line 275: overflow guard fires when `_seq >= (1 << 32)`. This is tighter than the RFC maximum of `2^96 - 1` for 12-byte nonces, but correct in practice because the XOR only covers 4 bytes (32 bits) of `_seq` in the last 4 bytes of the nonce. For any `_seq < 2^32`, the result is identical to `I2OSP(_seq, 12) XOR base_nonce` since the leading 8 bytes of `I2OSP(_seq, 12)` are zero.
- Lines 281–284: manual big-endian XOR into `nonce[8..11]` using 4 byte-level operations. This is correct for 32-bit `_seq`.
- Line 280: `_seq` incremented after reading. Correct (post-increment, pre-XOR using saved `s`).

**Nonce deviation note:** The overflow guard uses `(1 << 32)` instead of the RFC maximum `(1 << 96) - 1`. This means the context is rejected after 2^32 `seal()` calls rather than 2^96. In practice, with single-use contexts (see Resolved Questions #3), this is irrelevant. If multi-use contexts are ever introduced, the guard prevents valid nonce space from being used. This is a conservative choice, not a security defect.

**Verdict: matches RFC for the fixed-single-use operating mode. The guard limit is more conservative than RFC requires; not a defect.**

---

### §5.3 — Export

RFC 9180 §5.3:
```
def ExportSecret(exporter_context, L):
    return LabeledExpand(exporter_secret, "sec", exporter_context, L)
```

Implementation (`hpke.dart` lines 306–312):
```dart
HpkeSender._labeledExpand(
  HpkeSender._hpkeSuiteId,
  exporterSecret,
  utf8.encode('sec'),
  exporterContext,
  length,
)
```

Uses HPKE suite ID, label `"sec"`, `exporterSecret` as PRK, `exporterContext` as info, `length` as L. Matches RFC exactly.

**Verdict: matches RFC.**

---

## Related Modules / Services

| File | Role | Relationship to hpke.dart |
|---|---|---|
| `lib/src/hpke.dart` | HPKE Base Mode Sender | **Subject of this audit** |
| `lib/src/ohttp.dart` | RFC 9458 OHTTP encap/decap | Calls `HpkeSender.setupBaseS`, `ctx.seal`, `ctx.export`, `hkdfExtract`, `hkdfExpand` |
| `lib/src/bhttp.dart` | RFC 9292 framing | No dependency on hpke.dart |
| `lib/src/ohttp_client.dart` | High-level client | Calls `ohttpEncapsulate`/`ohttpDecapsulate` in ohttp.dart; transitive consumer |
| `test/hpke_test.dart` | RFC 9180 A.1 vector tests | Uses `testKeyPair` injection hook in all HPKE tests |
| `package:cryptography` | Crypto primitives | X25519, Hmac.sha256, AesGcm.with128bits — assumed correct; not re-audited |

---

## Current Endpoints & Contracts

`hpke.dart` exposes two public types:

**`HpkeSender` (static methods only):**
- `static Future<HpkeSenderContext> setupBaseS(Uint8List recipientPublicKey, Uint8List info, {SimpleKeyPairData? testKeyPair})` — lines 45–65
- `static Future<Uint8List> hkdfExtract(Uint8List salt, Uint8List ikm)` — lines 225–232 (public; used by `ohttp.dart` for response decap)
- `static Future<Uint8List> hkdfExpand(Uint8List prk, Uint8List info, int length)` — lines 235–254 (public; used by `ohttp.dart` for response decap)

**`HpkeSenderContext` (instance, private constructor):**
- `final Uint8List enc` — ephemeral public key
- `final Uint8List key` — AES-128 session key
- `final Uint8List baseNonce` — 12-byte base nonce
- `final Uint8List exporterSecret` — 32-byte exporter secret
- `Future<Uint8List> seal(Uint8List aad, Uint8List plaintext)` — lines 291–303
- `Future<Uint8List> export(Uint8List exporterContext, int length)` — lines 306–312

The private constructor `HpkeSenderContext._({...})` (line 265) prevents external instantiation, which is correct. However, `enc`, `key`, `baseNonce`, and `exporterSecret` are `final` public fields, exposing raw key material to any caller that holds a context reference.

---

## Patterns Used

| Pattern | Location | Notes |
|---|---|---|
| Static method class (no instances of `HpkeSender`) | `HpkeSender` class | All operations are async static methods |
| Named private constructor | `HpkeSenderContext._({...})` | Prevents external instantiation; correct pattern |
| Test-seam injection via optional named parameter | `setupBaseS(..., {SimpleKeyPairData? testKeyPair})` | Allows RFC vector reproducibility but leaks into production API surface |
| Manual big-endian byte encoding | `_computeNonce` lines 281–284 | Avoids external dependency; correct but verbose |
| `ByteBuilder` accumulation | `hkdfExpand` line 243 | Standard Dart pattern for building byte arrays |
| `Uint8List.sublistView` | `hkdfExpand` line 253 | Avoids copy for output truncation |
| Explicit `Uint8List.fromList([...spread])` | Throughout | Creates owned copies; correct but allocates frequently |

---

## Findings Table

Each row corresponds to a draft Jira task entry.

| # | Description | File : Line Range | RFC Section | Severity |
|---|---|---|---|---|
| F-1 | `testKeyPair` optional parameter in production `setupBaseS` signature allows any caller to supply a fixed (weak or known) ephemeral key, bypassing forward secrecy without any compiler or runtime warning. The `ohttp.dart` `ohttpEncapsulate` also threads this parameter through, widening the attack surface to any caller of the OHTTP layer. | `hpke.dart:47–48`, `hpke.dart:71–79`, `ohttp.dart:134–146` | RFC 9180 §4.1 (key generation requirement) | **HIGH** |
| F-2 | `HpkeSenderContext` fields `key` (AES-128 session key, 16 B), `baseNonce` (12 B), and `exporterSecret` (32 B) are `final` `Uint8List` fields that survive on the Dart heap after `seal()` and `export()` complete. No zeroization mechanism exists. Similarly, `OhttpEncapsulateResult.exportedSecret` (16 B) in `ohttp.dart:113` is retained until GC. A memory dump of a running wallet process reveals one request's encrypted inner HTTP content (traffic deanonymization risk). | `hpke.dart:259–262`, `ohttp.dart:113–114` | RFC 9180 §5.1, §5.2 (key-material handling) | **IMPROVEMENT** |
| F-3 | `_computeNonce` overflow guard (`hpke.dart:275`) throws `StateError('HPKE message limit reached')`. `StateError` is a core Dart type thrown by many unrelated APIs (e.g., iterators, streams). A caller catching `StateError` to handle flow-control conditions elsewhere will silently swallow the HPKE overflow, or conversely, a generic `try/catch StateError` for HPKE will inadvertently catch unrelated errors. A typed `HpkeSequenceOverflowError extends StateError` or a bespoke exception class would allow selective catching. In the target wallet (single-use contexts), this overflow is unreachable in practice, so severity is low. | `hpke.dart:275–277` | RFC 9180 §5.2 | **IMPROVEMENT** |
| F-4 | `_computeNonce` XORs only 4 bytes (32 bits) of `_seq` into the last 4 bytes of the nonce (lines 281–284). The overflow guard fires at `_seq >= 2^32`. This means the allowed nonce sequence space is `[0, 2^32 - 1]` rather than the RFC-permissible `[0, 2^96 - 1]`. For the current single-use context this is functionally irrelevant. If multi-use contexts are ever introduced (e.g., to support OHTTP streaming), the implementation would reject valid operations well before the RFC limit. The fix requires extending the XOR to all 8 high bytes and updating the overflow guard to `(1 << 96) - 1`. | `hpke.dart:275–285` | RFC 9180 §5.2 | **IMPROVEMENT** |
| F-5 | `hkdfExtract` and `hkdfExpand` are `static` public methods on `HpkeSender` (lines 225, 235). They are used internally by the labeled-KDF functions and externally by `ohttp.dart` for response decapsulation. Exposing raw unlabeled HKDF as part of the public API surface (re-exported through `lib/ohttp_dart.dart`) creates a risk that callers or future maintainers use them in place of the labeled variants, silently breaking domain separation. They should be package-private or clearly documented as internal. | `hpke.dart:225–254`, `lib/ohttp_dart.dart` (re-export surface) | RFC 9180 §4 (domain separation) | **IMPROVEMENT** |
| F-6 | No test covers the negative path where `_seq` reaches the overflow limit. The vector tests in `hpke_test.dart` cover `seq=0` and `seq=1` only; there is no test asserting that `seal()` on a context at `_seq = (1 << 32)` throws. If the overflow path is ever refactored, silent regression is possible. | `test/hpke_test.dart` (no overflow test), `hpke.dart:274–277` | RFC 9180 §5.2 | **IMPROVEMENT** |
| F-7 | RFC 9180 Appendix A.1 vector tests exist for `seq=0` and `seq=1` (`hpke_test.dart:104–199`). However, the vector test for `seal` uses the RFC 9180 AAD (`"Count-0"` / `"Count-1"`), while `ohttp.dart:149` calls `ctx.seal(Uint8List(0), binaryRequest)` with **empty AAD**. There is no test that exercises `seal` with empty AAD and verifies interoperability with a real OHTTP gateway. The RFC 9180 vectors confirm the HPKE primitives are correct; they do not confirm the OHTTP-level integration (empty-AAD choice). | `test/hpke_test.dart:104–119`, `ohttp.dart:149` | RFC 9458 §4.6.1 (OHTTP context), RFC 9180 §5.2 | **HIGH** (gap in integration test coverage; covered as a Phase 3 finding but noted here as a HPKE-layer test gap) |

---

## Limitations & Risks

1. **Single-suite constraint.** The implementation is hard-coded to DHKEM(X25519, HKDF-SHA256) + AES-128-GCM. Adding any other suite requires changes to `_kemSuiteId`, `_hpkeSuiteId`, the constants `_nk`, `_nn`, `_nh`, `_nSecret`, and the `OhttpKeyConfig` parser's `pkLen` branch in `ohttp.dart`. This is a design decision, not a defect, but it must be documented clearly before wallet integration so that a gateway advertising a different suite does not silently degrade security.

2. **`package:cryptography` trust boundary.** The audit assumes X25519, HMAC-SHA-256, and AES-128-GCM implementations in `package:cryptography` are correct. Any vulnerability in that package would bypass all HPKE-layer correctness. The `pubspec.yaml` dependency version should be pinned and reviewed separately (out of scope for Phase 1).

3. **Dart's `int` is 63-bit on native VM, 53-bit on web.** `_seq` is typed as `int`. The overflow guard `(1 << 32)` evaluates correctly on both platforms. However, if the package is ever run on a 32-bit JavaScript runtime (e.g., in a browser via dart2js with limited integer range), the behavior of `_seq` arithmetic should be verified.

4. **No receiver (decap) implementation.** The file is Sender-only. Any future addition of a receiver side would require careful integration of the key-schedule receiver path (`key_schedule_r`) and associated mode checks.

5. **Vector tests are the primary correctness evidence.** All RFC 9180 Appendix A.1 vectors for `SetupBaseS`, `seal` (seq=0 and seq=1), and `export` (three context variants) pass. This is strong evidence of correctness but does not substitute for an independent code review of the implementation against the RFC text (which this research provides).

---

## New Technical Questions

These questions surfaced during the audit and are not answered by the PRD or user responses above. They are recorded for follow-up in later phases or a separate triage.

1. **`lib/ohttp_dart.dart` re-export surface:** Does the public barrel re-export `HpkeSender` (and thus `hkdfExtract`/`hkdfExpand`) to external consumers? If so, the unlabeled HKDF functions are part of the package's public API. This should be audited in Phase 3 (ohttp.dart layer) when the full public surface is reviewed.

2. **`package:cryptography` version pinning:** The `pubspec.yaml` dependency on `package:cryptography` uses a version range. What is the current range, and has it been reviewed for CVEs or breaking changes? A supply-chain risk question relevant to all layers.

3. **Web/WASM target:** Is `ohttp_dart` intended to run in a Flutter Web or WASM context? If so, `dart2js` integer semantics for `_seq` and `(1 << 32)` should be verified. The `package:cryptography` implementations may also differ (browser subtle crypto vs. pure Dart).

4. **`export()` length validation:** `HpkeSenderContext.export` takes an arbitrary `length` parameter with no upper bound. RFC 9180 §5.3 states `L <= 255 * Nh`. For `Nh = 32`, the maximum is `L = 8160`. Values larger than this would require `hkdfExpand` to produce more than 255 blocks, which violates RFC 5869 §2.3. No guard exists. Is this a practical risk in the OHTTP context (where `export` is called with `length = 16`)? Probably not, but the absence of a guard means incorrect use goes undetected.

---

## Draft Task Entries (for Jira)

The following entries are derived directly from the findings table above. Each is independently actionable.

---

**TASK-1: Replace `testKeyPair` injection with a test-only factory or assertion**
- Severity: HIGH
- File: `hpke.dart:47–48`, `hpke.dart:71–79`, `ohttp.dart:134–146`
- RFC section: RFC 9180 §4.1
- Description: The `testKeyPair` optional parameter in `HpkeSender.setupBaseS` (and threaded through `ohttpEncapsulate`) allows any caller to supply a deterministic ephemeral key, bypassing X25519 key generation. Production builds cannot distinguish a valid call from a test call. Remediation: remove the parameter from the public signature and introduce a separate package-private or `@visibleForTesting` factory, or add an `assert(testKeyPair == null, ...)` in non-test builds.

---

**TASK-2: Add key-material zeroization for `HpkeSenderContext` fields**
- Severity: IMPROVEMENT
- File: `hpke.dart:259–262`, `ohttp.dart:113–114`
- RFC section: RFC 9180 §5.1, §5.2
- Description: `key`, `baseNonce`, `exporterSecret` (and `OhttpEncapsulateResult.exportedSecret`) are plain `Uint8List` fields retained on the Dart heap after the seal/export operations complete. Add a `dispose()` method that fills these arrays with zeros, and call it from `OhttpClient` after `ohttpDecapsulate` returns. Document in the API that callers must call `dispose()` to minimize key residency time. Note: Dart's GC provides no timing guarantee even after dispose; this is a best-effort mitigation for the stated threat model.

---

**TASK-3: Replace `StateError` on `_seq` overflow with a typed exception**
- Severity: IMPROVEMENT
- File: `hpke.dart:275–277`
- RFC section: RFC 9180 §5.2
- Description: `_computeNonce` throws `StateError('HPKE message limit reached')`. Introduce a typed `HpkeSequenceOverflowException` that extends `StateError` so callers can catch it selectively. Add a negative test that asserts this exception is thrown when `_seq` reaches the limit.

---

**TASK-4: Extend nonce XOR to full 12 bytes and update overflow guard**
- Severity: IMPROVEMENT
- File: `hpke.dart:275–285`
- RFC section: RFC 9180 §5.2
- Description: `_computeNonce` XORs only 32 bits of `_seq` into the nonce. The overflow guard is `_seq >= 2^32`, capping the sequence space far below the RFC maximum of `2^96 - 1`. For single-use contexts this is harmless, but it becomes a correctness issue if multi-use contexts are introduced. Extend the XOR loop to cover all 12 bytes of the nonce (8 zero-bytes plus the 4 sequence bytes), and update the guard to match the RFC limit or keep the 32-bit cap with an explicit comment documenting the intentional restriction.

---

**TASK-5: Restrict unlabeled `hkdfExtract` / `hkdfExpand` from the public API surface**
- Severity: IMPROVEMENT
- File: `hpke.dart:225–254`, `lib/ohttp_dart.dart`
- RFC section: RFC 9180 §4 (domain separation)
- Description: `HpkeSender.hkdfExtract` and `hkdfExpand` are public static methods visible to any consumer of the package. They are needed by `ohttp.dart` for response decapsulation (plain HKDF, per RFC 9458 §4.4). Move them to a package-private helper (e.g., in a separate `_hkdf.dart` or give them a `@internal` annotation via `package:meta`) so external callers cannot accidentally use them in place of the labeled variants.

---

**TASK-6: Add negative test for `_seq` overflow path**
- Severity: IMPROVEMENT
- File: `test/hpke_test.dart`, `hpke.dart:274–277`
- RFC section: RFC 9180 §5.2
- Description: No test verifies that `seal()` throws (with the correct exception type) when `_seq` reaches the overflow limit. Add a test that uses a white-box helper or reflection to pre-set `_seq` to `(1 << 32) - 1`, calls `seal()` once successfully, then calls `seal()` again and asserts the typed exception is thrown.

---

**TASK-7: Add integration test: `seal` with empty AAD against OHTTP gateway or reference vector**
- Severity: HIGH
- File: `test/hpke_test.dart`, `ohttp.dart:149`
- RFC section: RFC 9458 §4.6.1
- Description: The RFC 9180 A.1 vector tests confirm HPKE primitives are correct using non-empty AAD (`"Count-0"`). However, `ohttp.dart` calls `ctx.seal(Uint8List(0), ...)` with empty AAD. There is no test that confirms a full OHTTP encapsulate/decapsulate round-trip interoperates with a real gateway (e.g., the Go reference implementation). Add an integration test or a round-trip unit test using a simulated gateway that decapsulates with empty AAD, covering both the encapsulation and decapsulation paths.

---

*Research completed: 2026-05-21. Phase 1 only. No files in `lib/` or `test/` were modified.*
