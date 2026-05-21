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

- [ ] 3.1 Read the full file (use `ast-index outline lib/src/ohttp.dart` first, then read targeted slices).
- [ ] 3.2 Trace `OhttpKeyConfig.parse`: confirm the wire layout matches RFC 9458 §4.1 (key ID 1 B, KEM ID 2 B, public key `Npk` B, `symLen` 2 B, KDF+AEAD pairs).
- [ ] 3.3 Confirm that when `symLen > 4` the parser reads only the first KDF+AEAD pair and silently ignores the rest — note this as a finding.
- [ ] 3.4 Confirm that a short buffer with large `symLen` raises `RangeError`, not `FormatException` — note the exact throw site.
- [ ] 3.5 Note the inconsistency: `parse()` throws `FormatException` for unknown KEM, but `validate()` throws `UnsupportedError` for same condition.
- [ ] 3.6 Trace `ohttpEncapsulate`: verify HPKE info string construction (`"message/bhttp request" || 0x00 || header`) against RFC 9458 §4.3.
- [ ] 3.7 Confirm that AAD passed to `ctx.seal` is empty (`[]`) and cross-reference with RFC 9458 §4.3 wording; note the intentional deviation (matches Go reference implementation).
- [ ] 3.8 Trace `ohttpDecapsulate`: verify response HKDF usage — confirm it is plain (unlabeled) `HKDF-Extract(salt=enc||response_nonce, ikm=exportedSecret)` and `HKDF-Expand` for key/nonce derivation per RFC 9458 §4.4.
- [ ] 3.9 Confirm response AEAD also uses empty AAD.
- [ ] 3.10 Note that `SecretBoxAuthenticationError` (a `package:cryptography` type) propagates unwrapped to callers.
- [ ] 3.11 Record each finding as a draft task entry (file, line range, RFC section, severity).

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
