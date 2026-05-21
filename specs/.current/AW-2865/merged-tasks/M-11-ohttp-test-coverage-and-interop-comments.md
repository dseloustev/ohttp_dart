# Expand OHTTP test coverage and document interop hazards

**Estimate:** 1d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

`test/ohttp_test.dart` is 165 lines today; it covers `OhttpKeyConfig.parse`/`validate` and the encapsulate structure plus a decap short-circuit, but no end-to-end encap → decap round-trip with a synthetic gateway-side responder, no AEAD-authentication-failure path, and no edge-case inputs (empty / oversized payloads). Each gap allows a regression to land silently in code paths that are central to the library's value proposition.

Alongside the tests, two existing comment blocks in `lib/src/ohttp.dart` (the empty-AAD note at line 148 and the plain-HKDF note at line 192) are too terse to dissuade a well-intentioned "fix" from a future maintainer. The current implementation intentionally deviates from one reading of RFC 9458 §4.3 (uses empty AAD instead of the OHTTP header) to match the Go reference implementation, and intentionally uses plain (unlabeled) HKDF in the response path per RFC 9458 §4.4 rather than the labeled variants used in the HPKE key schedule. Both are correct but surprising; both are protected by the new round-trip test added in this task and by the bundled vectors.

## Technical Details

**1. OHTTP encap → decap round-trip** (`test/ohttp_test.dart`)

The current tests verify the encap output structure and the decap path's short-circuit logic in isolation. Add a true round-trip test that:

- Produces ciphertext via `ohttpEncapsulate`.
- Runs a small in-test helper that emulates the gateway-side response-channel derivation (uses the same primitives the library exposes: `HpkeSender.hkdfExtract` / `hkdfExpand`).
- Encrypts a canned response with the derived secret.
- Calls `ohttpDecapsulate` and asserts the decapsulated plaintext exactly equals the original.

Test name references RFC 9458 §4.3 and §4.4.

**2. AEAD authentication failure** (`test/ohttp_test.dart`, after line 163)

Line 163 closes the second `ohttpDecapsulate` test. Lines 217–225 of `lib/src/ohttp.dart` decrypt the response with `aesGcm.decrypt(...)`, which throws `SecretBoxAuthenticationError` on authentication failure (or the wrapped `OhttpDecryptionException` once the typed-exception hierarchy task lands).

Add a test that produces a valid encap, derives the response secret as in step 1, encrypts a response, then flips one byte in the ciphertext before calling `ohttpDecapsulate`. Assert that decap throws — initially the underlying cryptography type, and after the typed-hierarchy work lands, the library-owned wrapper. Author the test so it pins the current behaviour and updates cleanly when the exception type changes. Test name references RFC 9458 §4.4 and AEAD authentication semantics.

**3. Edge-case inputs** (`test/ohttp_test.dart`)

Edge-case inputs — empty BHTTP payload, single-byte payload, payload sized at the boundary of the parser-hardening caps — surface subtle bugs in length-handling code that happy-path tests miss.

Add tests covering:
- `ohttpEncapsulate` with an empty `binaryRequest` — define expected behaviour (likely succeeds and produces a minimal ciphertext).
- `ohttpDecapsulate` with an empty response payload — expect a typed exception.
- Round-trip with a payload at exactly the configured response-size cap (succeed) and one byte above (expect rejection).

If the size caps from the parser-hardening task have not yet landed, the boundary tests pin current behaviour and are updated when the cap lands. Tests reference RFC 9458 §4.3 / §4.4 where applicable.

**4. Empty-AAD maintenance-hazard comment** (`lib/src/ohttp.dart:148-150`)

Current line 148: `// Seal with empty AAD (per reference Go implementation, not RFC header)`. Expand the comment block to:

1. Cite RFC 9458 §4.3 and explain the wording ambiguity that makes empty AAD vs header-as-AAD a live question.
2. Reference the Go reference implementation (URL or specific commit identifier) as the source of truth for the empty-AAD choice.
3. State that the bundled tests rely on empty AAD and that changing it without coordinated gateway changes is a wire-format break.
4. Cross-reference the plain-HKDF comment block (step 5 below) as the companion maintenance hazard in the same file.

A maintainer reading the comment must be able to decide whether a proposed change would break interop without further research.

**5. Plain-HKDF maintenance-hazard comment** (`lib/src/ohttp.dart:192-207`)

Lines 192–207 are the plain-HKDF block in `ohttpDecapsulate`. The existing comment at line 192 (`// prk = HKDF-Extract(salt, secret) — plain HKDF, not labeled`) hints at the issue but does not state the consequences. Expand the comment block to:

1. Cite RFC 9458 §4.4.
2. State the trap: replacing `HpkeSender.hkdfExtract` / `hkdfExpand` with the labeled variants used in the HPKE key schedule will produce different key material and break decryption against any real gateway — and the change will be silent because the labeled variant still produces a valid (just wrong) result.
3. State that this is intentional and required by the RFC.
4. Cross-reference the empty-AAD comment block (step 4 above) as the companion maintenance hazard in the same file.

A maintainer skimming the function cannot accidentally "harmonize" to labeled HKDF without first dismissing the warning.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

- New end-to-end round-trip test exercises the full encap → response-secret derivation → response encryption → decap chain in `test/ohttp_test.dart`; helper is self-contained; decapsulated plaintext exactly equals the original plaintext.
- New tampered-ciphertext test added after line 163; asserts decap throws an exception (precise type per the current state of the typed-exception hierarchy work).
- New edge-case tests cover empty `binaryRequest`, empty response payload, payload at cap, and payload above cap.
- Comment block at `lib/src/ohttp.dart:148-150` is expanded to at least the four points listed for the empty-AAD note; includes a URL or commit reference to the Go interop implementation.
- Comment block at `lib/src/ohttp.dart:192-207` is expanded to at least the four points listed for the plain-HKDF note.
- All new test names reference the relevant RFC section.

## Additional

- Tests: unit tests in `test/ohttp_test.dart`; the round-trip test helper doubles as documentation of the response-channel contract.
- The comment expansions are self-contained code-doc edits; no behavioural change.
