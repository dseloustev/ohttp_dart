# Phase 3: Read OHTTP layer and assess encap/decap correctness

**Goal:** Audit `lib/src/ohttp.dart` against RFC 9458 to identify AAD deviations, response-decap correctness, and KeyConfig parser gaps.

## Context

**Feature motivation (from idea):** Investigation of `ohttp_dart` for potential use in a non-custodial crypto wallet. The goal is not to implement improvements but to identify risks and form subsequent engineering tasks. Every finding must map to a specific file, line range, and RFC section.

**Severity classification (from vision §2):**
- `BLOCKER` — must fix before wallet use
- `HIGH` — should fix before wallet use
- `IMPROVEMENT` — desirable but not blocking

**Technical approach (from vision §4):** `lib/src/ohttp.dart` is Layer 3 of the four-layer stack. It parses the gateway's `KeyConfig`, runs HPKE encapsulation over a pre-serialized BHTTP request, and decapsulates the encrypted gateway response. It sits between `hpke.dart` (Layer 1) and `ohttp_client.dart` (Layer 4).

**Primary investigation concerns (from vision §4 scrutiny table):**
- Empty AAD deviation from RFC 9458 (intentional deviation matching Go reference implementation)
- Response decap uses plain HKDF (not labeled) — different code paths from HPKE `LabeledExtract`/`LabeledExpand`
- KeyConfig parser reads only the first KDF+AEAD pair silently when `symLen > 4`

**Related classes/entities (from vision §5.1 — `OhttpKeyConfig`):**

| Field | Type | Wire source | Investigation concern |
|---|---|---|---|
| `keyId` | `int` (uint8) | byte 0 | No validation against a pinned key-ID set; gateway can rotate freely |
| `kemId` | `int` (uint16 BE) | bytes 1–2 | Only `0x0020` accepted; `parse()` throws `FormatException`, `validate()` throws `UnsupportedError` — inconsistent |
| `publicKey` | `Uint8List` (32 B) | bytes 3–34 | No pinning; point-on-curve check delegated to `package:cryptography` |
| `kdfId` / `aeadId` | `int` (uint16 BE) | bytes 37–40 | Parser reads only the **first** KDF+AEAD pair when `symLen > 4`; extra pairs are silently ignored |

Parser gap: malformed blob with large `symLen` and short buffer throws `RangeError`, not `FormatException`.

**Related classes/entities (from vision §5.2 — `OhttpEncapsulateResult`):**

| Field | Type | Size | Investigation concern |
|---|---|---|---|
| `encRequest` | `Uint8List` | 7 + 32 + plaintext + 16 | No upper-bound guard on payload size |
| `enc` | `Uint8List` | 32 B | Ephemeral X25519 public key; no zeroization after use |
| `exportedSecret` | `Uint8List` | 16 B | Sensitive; no zeroization after use |

**Data flow relevant to this phase (from vision §4 / §6.1):**
```
[inner HTTP request]
  → ohttpEncapsulate()
      → OhttpKeyConfig.parse/validate
      → HpkeSender.setupBaseS()     # DHKEM(X25519) Encap + HPKE key schedule
      → ctx.seal(aad=[], plaintext) # AES-128-GCM, seq-nonce XOR
          [CONCERN] Empty AAD deviates from RFC 9458 §4.6.1 (header-as-AAD wording)
      → ctx.export("message/bhttp response", 16)
          [CONCERN] enc and exportedSecret never zeroed after this call returns
      → assemble: header(7) || enc(32) || ciphertext
  → ohttpDecapsulate()
      → responseNonce = bodyBytes[0..15]
      → plain HKDF-Extract(salt=enc||responseNonce, ikm=exportedSecret)
          [CONCERN] Plain HKDF labels differ from labeled variants — any unification is a protocol break
      → plain HKDF-Expand → key(16 B) + nonce(12 B)
      → AES-128-GCM decrypt(aad=[])
          → auth failure → SecretBoxAuthenticationError (package:cryptography type)
          [CONCERN] AEAD failure type leaks package:cryptography into public API surface
```

**Error paths (from vision §6.2 / §6.3):**
- `RangeError` from malformed KeyConfig body propagates unwrapped; caller cannot distinguish it from a logic bug.
- AEAD auth failure (`SecretBoxAuthenticationError`) propagates unwrapped — `package:cryptography` type leaks into the public API surface.

**Cross-cutting severity (from vision §5 — data model concerns):**

| Concern | Severity |
|---|---|
| Silent ignore of extra KDF+AEAD pairs in KeyConfig | HIGH |
| `enc` and `exportedSecret` never zeroized after use | HIGH |
| Inconsistent exception types in `OhttpKeyConfig` | IMPROVEMENT |

## Tasks

- [x] 3.1 Read the full file (use `ast-index outline lib/src/ohttp.dart` first, then read targeted slices). Confirmed outline (258 lines): `OhttpKeyConfig` class :17, `.parse` :32, `validate` :81, `OhttpEncapsulateResult` :105, `ohttpEncapsulate` :131, `ohttpDecapsulate` :174, `_buildHpkeInfo` :232, `_buildRequestHeader` :249. Line numbers match all citations in `specs/.current/AW-2865/phase-3/research.md`.
- [x] 3.2 Trace `OhttpKeyConfig.parse`: confirm the wire layout matches RFC 9458 §4.1 (key ID 1 B, KEM ID 2 B, public key `Npk` B, `symLen` 2 B, KDF+AEAD pairs). **Finding O-1 (POSITIVE) — `ohttp.dart:32-79`:** Parser reads `key_id` at `:40`, `kem_id` BE at `:41`, `public_key` (32 B for X25519) at `:57`, `symLen` BE at `:60`, and first `kdf_id`/`aead_id` pair at `:68`/`:70`. All four length guards (`:33`, `:54`, `:63`) are present and correctly ordered. Happy-path wire layout is **COMPLIANT** with RFC 9458 §4.1. Reference: `specs/.current/AW-2865/phase-3/research.md` Finding O-1.
- [x] 3.3 Confirm that when `symLen > 4` the parser reads only the first KDF+AEAD pair and silently ignores the rest — note this as a finding. **Finding O-2 (HIGH) — `ohttp.dart:60-71`:** RFC 9458 §4.1 allows multiple KDF+AEAD pairs (variable-length `sym_algorithms`); the parser at `:67-71` reads exactly the first 4-byte pair and returns without inspecting remaining pairs. No loop, no rejection, no diagnostic. Combined with `validate()` at `:81-97` enforcing the supported suite only on the (first) selected pair, this enables an implicit cipher-suite downgrade if a gateway ever advertises the preferred suite second. Buffer length is fully present (guard at `:63` confirms `data.length >= offset + symLen`); the bytes are simply never examined. Test gap: no `symLen > 4` case in `test/ohttp_test.dart`. Reference: `specs/.current/AW-2865/phase-3/research.md` Finding O-2.
- [x] 3.4 Confirm that a short buffer with large `symLen` raises `RangeError`, not `FormatException` — note the exact throw site. **Finding O-3 (POSITIVE — vision assumption corrected) — `ohttp.dart:33-71`:** Audit revealed the vision's premise is **incorrect for the current code**. The guard at `:63` (`symLen < 4 || data.length < offset + symLen`) catches both invalid-`symLen` and short-buffer conditions with `FormatException('Invalid symmetric algorithms section')` before any list access at `:68`/`:70`. The earlier guards at `:33-34` (min length 7) and `:54-56` (`pkLen + 2`) close all earlier truncation gaps. **No `RangeError` escapes through the parser.** Minor IMPROVEMENT: the single error message at `:64` conflates "semantically invalid `symLen`" with "truncated buffer"; splitting would aid diagnostics. Reference: `specs/.current/AW-2865/phase-3/research.md` Finding O-3.
- [x] 3.5 Note the inconsistency: `parse()` throws `FormatException` for unknown KEM, but `validate()` throws `UnsupportedError` for same condition. **Finding O-4 (IMPROVEMENT) — `ohttp.dart:51` vs `ohttp.dart:82-85`:** `parse()` at `:51` throws `FormatException('Unsupported KEM: 0x…')`; `validate()` at `:82-85` throws `UnsupportedError('Unsupported KEM: 0x… (expected X25519 0x0020)')`. Both express the same "this KEM is not supported" condition but force callers to catch two unrelated exception hierarchies. In normal flow `parse()` rejects first via its `switch`; the `validate()` path is reached only when an `OhttpKeyConfig` is constructed directly bypassing `parse()` (as in `test/ohttp_test.dart:79-85`). Cross-phase: same anti-pattern as Phase 1 F-3 and Phase 2 R-3. Reference: `specs/.current/AW-2865/phase-3/research.md` Finding O-4.
- [x] 3.6 Trace `ohttpEncapsulate`: verify HPKE info string construction (`"message/bhttp request" || 0x00 || header`) against RFC 9458 §4.3. **Finding O-5 (POSITIVE) — `ohttp.dart:232-245` (`_buildHpkeInfo`):** Constructs `utf8("message/bhttp request")` (22 B) || `0x00` || `key_id` (1 B) || `kem_id` (2 B BE) || `kdf_id` (2 B BE) || `aead_id` (2 B BE) = 30 B total. `_buildRequestHeader` at `:249-257` produces the matching 7-byte header used as the OHTTP request prefix. Both encode 16-bit IDs as big-endian. **COMPLIANT** with RFC 9458 §4.3. Reference: `specs/.current/AW-2865/phase-3/research.md` Finding O-5.
- [x] 3.7 Confirm that AAD passed to `ctx.seal` is empty (`[]`) and cross-reference with RFC 9458 §4.3 wording; note the intentional deviation (matches Go reference implementation). **Finding O-6 (IMPROVEMENT) — `ohttp.dart:149`:** `final ct = await ctx.seal(Uint8List(0), binaryRequest);` passes empty AAD. RFC 9458 §4.3 / §4.6.1 wording suggests header-as-AAD. The deviation is intentional and matches the Go reference implementation (documented inline at `:148` and in `CLAUDE.md`); the bundled test suite passes with empty AAD. Confidentiality/integrity of the plaintext are unaffected — the header is transmitted verbatim at `:160` anyway. Remediation: expand the inline comment; no protocol change required. Test gap: no integration test against a reference Go gateway confirms wire-format interop (same gap as Phase 1 F-7). Reference: `specs/.current/AW-2865/phase-3/research.md` Finding O-6.
- [x] 3.8 Trace `ohttpDecapsulate`: verify response HKDF usage — confirm it is plain (unlabeled) `HKDF-Extract(salt=enc||response_nonce, ikm=exportedSecret)` and `HKDF-Expand` for key/nonce derivation per RFC 9458 §4.4. **Finding O-7 (POSITIVE with MAINTENANCE hazard) — `ohttp.dart:192-207`:** `:193` calls `HpkeSender.hkdfExtract(salt, exportedSecret)` (plain RFC 5869); `:196-200` calls `HpkeSender.hkdfExpand(prk, utf8.encode('key'), _nk=16)`; `:203-207` calls `HpkeSender.hkdfExpand(prk, utf8.encode('nonce'), _nn=12)`. Salt at `:190` = `enc(32) || responseNonce(16)` = 48 B. Response nonce extracted at `:186-187` as `max(Nn, Nk) = 16` B. **COMPLIANT** with RFC 9458 §4.4. **Maintenance hazard:** `HpkeSender.hkdfExtract` / `hkdfExpand` are `static` and re-exported through `lib/ohttp_dart.dart:10`; substituting them with `LabeledExtract`/`LabeledExpand` would silently derive different key material with no compile-time signal. A warning comment at the three call sites is the recommended remediation. Reference: `specs/.current/AW-2865/phase-3/research.md` Finding O-7.
- [x] 3.9 Confirm response AEAD also uses empty AAD. **Finding O-8 (POSITIVE) — `ohttp.dart:219-223`:** `aesGcm.decrypt(secretBox, secretKey: SecretKeyData(aeadKey), aad: [])`. RFC 9458 §4.4 specifies empty AAD on response decryption; the implementation uses `aad: []` (equivalent to empty byte string). **COMPLIANT** with RFC 9458 §4.4. Reference: `specs/.current/AW-2865/phase-3/research.md` Finding O-8.
- [x] 3.10 Note that `SecretBoxAuthenticationError` (a `package:cryptography` type) propagates unwrapped to callers. **Finding O-9 (HIGH) — `ohttp.dart:219`:** On AEAD auth failure, `aesGcm.decrypt()` throws `SecretBoxAuthenticationError` from `package:cryptography/cryptography.dart`. The exception is not re-exported by `lib/ohttp_dart.dart`, so a caller wishing to catch it specifically must add a direct `package:cryptography` import — an unintended transitive dependency that makes `package:cryptography` part of the public API surface despite being an internal implementation detail. If the dependency is replaced or its exception hierarchy changes, all wallet-layer catch clauses break. Test gap: `test/ohttp_test.dart:149-163` covers only short-response rejection; the auth-failure path is untested. Remediation: define a library-owned `OhttpAuthenticationException` in `ohttp.dart` and wrap the `decrypt()` call. Reference: `specs/.current/AW-2865/phase-3/research.md` Finding O-9.
- [x] 3.11 Record each finding as a draft task entry (file, line range, RFC section, severity). Draft entries TASK-O1 through TASK-O7 recorded below and in `specs/.current/AW-2865/phase-3/research.md` § Draft Task Entries. Two additional positive verdicts (O-10 zeroization absence; O-11 test-coverage gaps) are captured in the research document and folded into TASK-O6 and TASK-O7 respectively.

### Draft task entries (from research §Draft Task Entries)

**TASK-O1 — Fix silent drop of extra KDF+AEAD pairs in `OhttpKeyConfig.parse`**
- File: `lib/src/ohttp.dart:60-78`
- RFC section: RFC 9458 §4.1
- Severity: HIGH
- Description: When `symLen > 4`, the parser reads only the first KDF+AEAD pair and silently returns. Either iterate all pairs and confirm only the supported suite is present, or throw `FormatException` when `symLen > 4` to reject multi-suite advertisements until multi-suite negotiation is implemented.

**TASK-O2 — Unify exception types for unsupported KEM in `parse()` and `validate()`**
- File: `lib/src/ohttp.dart:51`, `lib/src/ohttp.dart:82-85`
- RFC section: RFC 9458 §4.1
- Severity: IMPROVEMENT
- Description: `parse()` throws `FormatException` and `validate()` throws `UnsupportedError` for the same "unsupported KEM" condition. Consolidate to a single exception type (e.g., a library-owned `OhttpUnsupportedSuiteException`) so callers can write a single catch clause. Cross-phase: same anti-pattern as Phase 1 F-3 and Phase 2 R-3.

**TASK-O3 — Expand inline comment at empty-AAD `ctx.seal` call site**
- File: `lib/src/ohttp.dart:148-150`
- RFC section: RFC 9458 §4.3
- Severity: IMPROVEMENT
- Description: The existing comment at `:148` mentions "per reference Go implementation, not RFC header" but does not fully document the deviation. Expand to cite RFC 9458 §4.3 (header-as-AAD wording), explain the deliberate Go-interop choice, and note that the bundled test suite locks in this behavior.

**TASK-O4 — Add maintenance-hazard comment at plain-HKDF call sites in `ohttpDecapsulate`**
- File: `lib/src/ohttp.dart:192-207`
- RFC section: RFC 9458 §4.4
- Severity: MAINTENANCE (IMPROVEMENT)
- Description: Add a code comment at `:193`, `:196`, and `:203` warning that these are plain (unlabeled) RFC 5869 HKDF calls per RFC 9458 §4.4 and must NOT be replaced with `LabeledExtract`/`LabeledExpand` from `hpke.dart`. Substitution would silently derive different key material and break response decryption with no compile-time signal.

**TASK-O5 — Wrap `SecretBoxAuthenticationError` in a library-owned exception**
- File: `lib/src/ohttp.dart:217-225`
- RFC section: RFC 9458 §4.4
- Severity: HIGH
- Description: Wrap the `aesGcm.decrypt()` call in a try/catch for `SecretBoxAuthenticationError` and rethrow as a new library-owned `OhttpAuthenticationException` defined in `ohttp.dart`. Prevents `package:cryptography` from leaking into the public API surface and gives callers a stable catch target.

**TASK-O6 — Add best-effort zeroization for `OhttpEncapsulateResult.enc` and `exportedSecret`**
- File: `lib/src/ohttp.dart:104-120`, `lib/src/ohttp.dart:162-166`
- RFC section: RFC 9458 §4.3
- Severity: HIGH
- Description: After `ohttpDecapsulate` consumes `enc` and `exportedSecret`, explicitly zero-fill both `Uint8List` backing buffers before the reference is released. Best-effort (Dart VM does not guarantee GC-root clearing) but reduces exposure in AOT-compiled deployments, which are the primary wallet target and the higher-risk scenario for memory-dump attacks. Cross-phase: extends Phase 1 F-2 to the OHTTP layer.

**TASK-O7 — Add missing test cases for `ohttp_test.dart`**
- File: `test/ohttp_test.dart`
- RFC section: RFC 9458 §4.3, §4.4
- Severity: HIGH
- Description: Add four test groups:
  1. **Round-trip:** `ohttpEncapsulate` followed by a gateway-simulated `ohttpDecapsulate` using the same exported secret — confirm plaintext recovery.
  2. **Authentication failure:** valid-length `encResponse` with a bit-flipped byte in the ciphertext — confirm the future `OhttpAuthenticationException` (from TASK-O5) is thrown.
  3. **Multi-suite KeyConfig:** `symLen = 8` with supported first pair — confirm parse succeeds and documents that the second pair is silently dropped (current behavior; regression guard for TASK-O1).
  4. **Truncated ciphertext:** `encResponse` length between `_responseNonceLen + 1` and `_responseNonceLen + tagLen - 1` — confirm `FormatException` from `ohttp.dart:212-213`.
  Cross-phase: same test-gap pattern as Phase 1 F-6/F-7 and Phase 2 R-6.

## Acceptance Criteria

**Test:** No code is changed. Verification is: the empty-AAD finding and the plain-HKDF-vs-labeled-HKDF finding each cite the exact line in `ohttp.dart` plus the RFC 9458 section that motivates the concern.

## Dependencies

- Phase 2 complete (BHTTP layer audit)

## Technical Details

**RFC 9458 OHTTP encapsulation overview (§4.3):**

The client constructs the HPKE info string as:
```
info = "message/bhttp request" || 0x00 || header
```
where `header` is the 7-byte OHTTP request header (`key_id || kem_id || kdf_id || aead_id`).

The encapsulated request is assembled as:
```
encRequest = header(7 B) || enc(32 B) || ct
```
where `ct = HPKE.Seal(aad=[], plaintext=binaryRequest)`.

**RFC 9458 response decapsulation overview (§4.4):**

Response key derivation uses **plain (unlabeled) HKDF**, not the labeled variants from RFC 9180. The derivation is:
```
response_nonce = random(max(Nn, Nk)) = 16 B for AES-128-GCM
salt = enc || response_nonce
prk = HKDF-Extract(salt, ikm=exportedSecret)
key = HKDF-Expand(prk, label="key", L=Nk=16)
nonce = HKDF-Expand(prk, label="nonce", L=Nn=12)
```
This is distinct from `LabeledExtract`/`LabeledExpand` in `hpke.dart` — they must NOT be unified.

**Wire layout of `OhttpKeyConfig` (RFC 9458 §4.1):**
```
key_config {
  key_id          1 B   (uint8)
  kem_id          2 B   (uint16 BE)
  public_key      Npk B (32 B for X25519)
  sym_algorithms  2 B   (uint16 LE — total byte count of KDF+AEAD pairs)
  [kdf_id (2 B) + aead_id (2 B)] × N pairs
}
```
`symLen == 4` means exactly one KDF+AEAD pair. `symLen > 4` means multiple suites; the current parser reads only the first.

**Fixed cipher suite IDs (from `hpke.dart` / `ohttp.dart`):**
- KEM ID: `0x0020` (DHKEM(X25519, HKDF-SHA256)), `Npk = 32`
- KDF ID: `0x0001` (HKDF-SHA256)
- AEAD ID: `0x0001` (AES-128-GCM), `Nk = 16`, `Nn = 12`

**Known deviations (from CLAUDE.md):**
- **Empty AAD** in request sealing (`ohttp.dart:149`) is intentional — the bundled tests and Go interop reference both expect empty AAD. Do not flag as a defect; document as an intentional deviation from some RFC 9458 readings.
- **Response decap uses plain HKDF**, not the labeled `LabeledExtract`/`LabeledExpand` from HPKE — separate code paths (`HpkeSender.hkdfExtract` / `hkdfExpand` in `hpke.dart` vs. the labeled variants used internally during `setupBaseS`).

## Implementation Notes

This is a read-only audit iteration. Do not edit any file in `lib/` or `test/`. Record all findings as draft task entries in this file for use in Iteration 7 (compile into per-task Markdown files).

Phases 1 (HPKE) and 2 (BHTTP) completed with findings in their respective `tasks.md` files; reference those findings when noting cross-layer concerns (e.g., OHTTP AAD handling that compounds HPKE risks, or KeyConfig parser errors that compound BHTTP error-type concerns).
