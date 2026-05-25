# Negative-path unit-test coverage (HPKE + BHTTP + OHTTP) and interop-hazard comments

**Estimate:** 3d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

The existing test suite covers RFC vectors and a small set of happy paths across `test/hpke_test.dart`, `test/bhttp_test.dart`, and `test/ohttp_test.dart`, but leaves a substantial negative-path surface unexercised. Each gap allows a regression to land silently in code paths central to the library's value proposition. This task expands the three unit-test files in one bundled PR's worth of work and tightens two interop-hazard comments in `lib/src/ohttp.dart` that protect intentional deviations from the obvious "RFC harmonization" reading.

Depends on the BHTTP parser hardening from L-03 (M-10's truncation tests assert `FormatException` rather than `RangeError`) and the typed exception hierarchy from L-02 (M-11's tampered-ciphertext test asserts `OhttpDecryptionException`).

## Technical Details

### Part A — HPKE coverage (`test/hpke_test.dart`)

**1. Sequence-number overflow in `HpkeSenderContext.seal`** (after line 200)

`HpkeSenderContext.seal` computes the per-call nonce by XOR-ing `_seq` into `baseNonce`. RFC 9180 §5.2 caps the sequence number at `2^(8*Nn) - 1`; on overflow, the implementation must refuse to seal. AES-GCM nonce reuse after overflow is catastrophic. The existing `seal seq=0` and `seal seq=1` tests end around line 200; no overflow test exists.

Add a test that:
- Forces `_seq` to `2^(8*Nn) - 1`.
- Calls `seal` once successfully.
- Calls `seal` again and asserts the call fails with a library-owned exception (not a generic `Error`).

If the current implementation does not enforce the cap, scope expands to include the implementation fix in `hpke.dart`. Test name references RFC 9180 §5.2.

**2. `setupBaseS` with invalid public key**

The X25519 specification (RFC 7748 §6.1) flags certain low-order points for which the shared secret is constant or zero. A wallet receiving an attacker-controlled KeyConfig could be presented with a public key chosen to produce a known shared secret.

Add tests that call `setupBaseS` with:
- The all-zero 32-byte public key.
- The canonical X25519 low-order points listed in RFC 7748 §6.1 / informational HPKE guidance.

Assert the expected behaviour (rejection at `setupBaseS`, or — if the X25519 layer surfaces the error — a defined library-owned exception). If the library does not currently reject these inputs, scope expands to include the validation in `hpke.dart` or in the `OhttpKeyConfig.parse`/`validate` flow. Test name references RFC 7748 §6.1.

**3. `HpkeSenderContext.export` with invalid length** (after line 241)

RFC 9180 §5.3 caps the `Export` length parameter at `2^32 * Nh` bytes; zero-length is a defined edge case; oversized lengths must be rejected.

Add tests covering:
- `length == 0` — define expected behaviour (likely returns empty `Uint8List`).
- `length == Nh` — canonical case (already covered indirectly; explicit assertion).
- `length > 2^32 * Nh` — expect a library-owned exception.

If the implementation does not currently enforce the cap, scope expands to include the validation. Test names reference RFC 9180 §5.3.

### Part B — BHTTP coverage (`test/bhttp_test.dart`)

**4. QUIC varint 8-byte boundary round-trip** (after line 68)

RFC 9000 §16 (referenced by RFC 9292 §3.5.4) defines varint encoding boundaries at 1, 2, 4, and 8 bytes; the existing tests cover the lower boundaries but the 8-byte boundary case is not exercised. A bug in the 8-byte encode/decode path would silently corrupt large response framing.

Add a test that round-trips three values through `encodeVarint` and `decodeVarint`:
- The largest 4-byte-encodable value.
- The smallest 8-byte-encodable value.
- The largest 8-byte-encodable value (`2^62 - 1` per RFC 9000 §16).

Assert both byte-level equality of the encoded form and exact equality after decode. Test name references RFC 9000 §16.

**5. `parseResponse` truncated body** (after line 185)

Lines 173, 178, and 187 of `lib/src/bhttp.dart` perform `data.sublist(...)` calls that throw `FormatException` once the parser is hardened in L-03.

Add unit tests covering truncation at each of the documented offsets:
- Header name length.
- Header value length.
- Content length.

Assert that the exception thrown is `FormatException` with a useful message. If L-03 has not yet landed, pin the current `RangeError` behaviour and update when L-03 lands.

**6. Unknown framing indicator**

RFC 9292 §3.3 defines framing indicators `0` (Known-Length Request), `1` (Known-Length Response), `2` (Indeterminate-Length Request), and `3` (Indeterminate-Length Response). The library supports only `0` and `1`.

Add unit tests covering:
- Indicator `2` (indeterminate-length request).
- Indicator `3` (indeterminate-length response).
- Indicator `4` (out-of-range valid varint).
- A large varint value.

Assert each produces a `FormatException` with a message that includes the offending indicator. Tests reference RFC 9292 §3.3.

### Part C — OHTTP coverage (`test/ohttp_test.dart`)

**7. OHTTP encap → decap round-trip**

The current tests verify the encap output structure and the decap path's short-circuit logic in isolation. Add a true round-trip test that:

- Produces ciphertext via `ohttpEncapsulate`.
- Runs a small in-test helper that emulates the gateway-side response-channel derivation (uses the same primitives the library exposes: `HpkeSender.hkdfExtract` / `hkdfExpand`).
- Encrypts a canned response with the derived secret.
- Calls `ohttpDecapsulate` and asserts the decapsulated plaintext exactly equals the original.

Test name references RFC 9458 §4.3 and §4.4.

**8. AEAD authentication failure** (after line 163)

Lines 217–225 of `lib/src/ohttp.dart` decrypt the response with `aesGcm.decrypt(...)`. After L-02 lands, decapsulation surfaces `OhttpDecryptionException` on auth failure.

Add a test that produces a valid encap, derives the response secret as in test 7, encrypts a response, then flips one byte in the ciphertext before calling `ohttpDecapsulate`. Assert that decap throws `OhttpDecryptionException` (or the underlying cryptography type if L-02 has not yet landed; pin and update). Test name references RFC 9458 §4.4 and AEAD authentication semantics.

**9. Edge-case inputs**

Add tests covering:
- `ohttpEncapsulate` with an empty `binaryRequest` — define expected behaviour (likely succeeds and produces a minimal ciphertext).
- `ohttpDecapsulate` with an empty response payload — expect a typed exception.
- Round-trip with a payload at exactly the configured response-size cap (succeed) and one byte above (expect rejection from L-03's cap).

If L-03's size caps have not landed, pin current behaviour and update when the cap lands. Tests reference RFC 9458 §4.3 / §4.4 where applicable.

### Part D — Interop-hazard comments in `lib/src/ohttp.dart`

**10. Empty-AAD maintenance-hazard comment** (`lib/src/ohttp.dart:148-150`)

Current line 148: `// Seal with empty AAD (per reference Go implementation, not RFC header)`. Expand the comment block to:

1. Cite RFC 9458 §4.3 and explain the wording ambiguity that makes empty AAD vs header-as-AAD a live question.
2. Reference the Go reference implementation (URL or specific commit identifier) as the source of truth for the empty-AAD choice.
3. State that the bundled tests rely on empty AAD and that changing it without coordinated gateway changes is a wire-format break.
4. Cross-reference the plain-HKDF comment block (step 11 below) as the companion maintenance hazard in the same file.

A maintainer reading the comment must be able to decide whether a proposed change would break interop without further research.

**11. Plain-HKDF maintenance-hazard comment** (`lib/src/ohttp.dart:192-207`)

Lines 192–207 are the plain-HKDF block in `ohttpDecapsulate`. The existing comment at line 192 (`// prk = HKDF-Extract(salt, secret) — plain HKDF, not labeled`) hints at the issue but does not state the consequences. Expand to:

1. Cite RFC 9458 §4.4.
2. State the trap: replacing `HpkeSender.hkdfExtract` / `hkdfExpand` with the labeled variants used in the HPKE key schedule will produce different key material and break decryption against any real gateway — and the change will be silent because the labeled variant still produces a valid (just wrong) result.
3. State that this is intentional and required by the RFC.
4. Cross-reference the empty-AAD comment block (step 10 above) as the companion maintenance hazard in the same file.

A maintainer skimming the function cannot accidentally "harmonize" to labeled HKDF without first dismissing the warning.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

**HPKE:**
- Sequence-number overflow test added after line 200 in `test/hpke_test.dart`; exercises the boundary at `_seq == 2^(8*Nn) - 1` and the overflow case; asserts a library-owned exception. If the current code lacks the cap, the implementation fix is included.
- Invalid-public-key tests added; cover at least the all-zero 32-byte public key plus the RFC 7748 §6.1 low-order points; assert the expected library-owned exception. If the implementation lacks validation, the validation fix is included.
- `export` invalid-length tests added after line 241; cover `length == 0`, `length == Nh`, and `length > 2^32 * Nh`; assert a library-owned exception on the over-cap case. If the implementation lacks the cap, the implementation fix is included.

**BHTTP:**
- Varint 8-byte boundary round-trip test added after line 68; covers the three boundary values; asserts byte-level encoding equality and decoded-value equality; references RFC 9000 §16.
- Truncated-body tests added after line 185; cover header-name, header-value, and content truncation positions; assert `FormatException` with a useful message (assumes L-03 has landed).
- Unknown-framing-indicator tests cover indicators `2`, `3`, `4`, and a large varint value; each asserts `FormatException` with a message naming the offending indicator; references RFC 9292 §3.3.

**OHTTP:**
- New end-to-end round-trip test exercises the full encap → response-secret derivation → response encryption → decap chain in `test/ohttp_test.dart`; helper is self-contained; decapsulated plaintext exactly equals the original.
- New tampered-ciphertext test added after line 163; asserts decap throws `OhttpDecryptionException` (or underlying type if L-02 has not yet landed).
- New edge-case tests cover empty `binaryRequest`, empty response payload, payload at cap, and payload above cap.

**Comments:**
- Comment block at `lib/src/ohttp.dart:148-150` is expanded to at least the four points listed for the empty-AAD note; includes a URL or commit reference to the Go interop implementation.
- Comment block at `lib/src/ohttp.dart:192-207` is expanded to at least the four points listed for the plain-HKDF note.

**Naming:**
- All new test names reference the relevant RFC section.

## Additional

- Tests: pure unit tests in the three existing test files; the OHTTP round-trip helper doubles as documentation of the response-channel contract.
- The comment expansions are self-contained code-doc edits; no behavioural change.
- Depends on L-03 (parser hardening) and L-02 (typed exception hierarchy). If either has not landed, pin current behaviour and update when the upstream task lands.
- Source merged tasks (in `../merged-tasks/`): M-09 (HPKE coverage), M-10 (BHTTP coverage), M-11 (OHTTP coverage + interop comments).
