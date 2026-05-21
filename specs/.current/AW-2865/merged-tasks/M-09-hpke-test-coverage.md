# Expand HPKE test coverage (overflow, invalid pubkey, export bounds)

**Estimate:** 1d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

`test/hpke_test.dart` covers RFC 9180 Appendix A.1 vectors via the `testKeyPair` injection hook in `setupBaseS` and a small set of happy-path tests, but it leaves three meaningful negative paths unexercised. Each gap allows a regression to land silently and undermines the cryptographic guarantees the library is supposed to provide. While normal OHTTP flow does not reach any of these conditions (one-shot seal, gateway-supplied public key), the underlying machinery exists and could be exercised by future code paths or by malicious gateway-supplied inputs.

## Technical Details

**1. Sequence-number overflow in `HpkeSenderContext.seal`** (`test/hpke_test.dart`, after line 200)

`HpkeSenderContext.seal` computes the per-call nonce by XOR-ing `_seq` into `baseNonce`. RFC 9180 §5.2 caps the sequence number at `2^(8*Nn) - 1`; on overflow, the implementation must refuse to seal. AES-GCM nonce reuse after overflow is catastrophic. The existing `seal seq=0` and `seal seq=1` tests end around line 200; no overflow test exists.

Add a test that:
- Forces `_seq` to `2^(8*Nn) - 1`.
- Calls `seal` once successfully.
- Calls `seal` again and asserts the call fails with a library-owned exception (not a generic `Error`).

If the current implementation does not enforce the cap, scope expands to include the implementation fix in `hpke.dart`. Test name references RFC 9180 §5.2.

**2. `setupBaseS` with invalid public key** (`test/hpke_test.dart`)

`setupBaseS` accepts a recipient public key but no existing test feeds it a degenerate value. The X25519 specification (RFC 7748 §6.1) flags certain low-order points for which the shared secret is constant or zero. A wallet receiving an attacker-controlled KeyConfig could be presented with a public key chosen to produce a known shared secret.

Add tests that call `setupBaseS` with:
- The all-zero 32-byte public key.
- The canonical X25519 low-order points listed in RFC 7748 §6.1 / informational HPKE guidance.

Assert the expected behaviour (rejection at `setupBaseS`, or — if the X25519 layer surfaces the error — a defined library-owned exception). If the library does not currently reject these inputs, scope expands to include the validation in `hpke.dart` or in the `OhttpKeyConfig.parse`/`validate` flow. Test name references RFC 7748 §6.1.

**3. `HpkeSenderContext.export` with invalid length** (`test/hpke_test.dart`, after line 241)

Line 241 is the last `});` of the `export with "TestContext"` test. The `HPKE RFC 9180` group closes at line 242. RFC 9180 §5.3 caps the `Export` length parameter at `2^32 * Nh` bytes; zero-length is a defined edge case; oversized lengths must be rejected.

Add tests covering:
- `length == 0` — define expected behaviour (likely returns empty `Uint8List`).
- `length == Nh` — canonical case (already covered indirectly; explicit assertion).
- `length > 2^32 * Nh` — expect a library-owned exception.

If the implementation does not currently enforce the cap, scope expands to include the validation. Test names reference RFC 9180 §5.3.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

- Sequence-number overflow test added after line 200 in `test/hpke_test.dart`; exercises the boundary at `_seq == 2^(8*Nn) - 1` and the overflow case at `_seq == 2^(8*Nn)`; asserts a library-owned exception. If the current code lacks the cap, the implementation fix is included.
- Invalid-public-key tests added; cover at least the all-zero 32-byte public key plus the RFC 7748 §6.1 low-order points; assert the expected library-owned exception. If the implementation lacks validation, the validation fix is included.
- `export` invalid-length tests added after line 241; cover `length == 0`, `length == Nh`, and `length > 2^32 * Nh`; assert a library-owned exception on the over-cap case. If the implementation lacks the cap, the implementation fix is included.
- All new test names reference the relevant RFC section.

## Additional

- Tests: pure unit tests in `test/hpke_test.dart`.
- Some sub-items may expand to include implementation changes if the current code does not enforce the relevant constraint.
