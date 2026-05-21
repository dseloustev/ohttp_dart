# Task 15: Test `setupBaseS` with invalid public key

**Severity:** HIGH
**Vector:** Test suite
**Files:** `test/hpke_test.dart`

**Evidence:** Cross-ref T-H3. `setupBaseS` accepts a recipient public key but no existing test feeds it a degenerate value such as the all-zero X25519 point. The X25519 specification (RFC 7748 §6.1) flags certain low-order points for which the shared secret is constant or zero.

## Description

A wallet receiving an attacker-controlled KeyConfig could be presented with a public key chosen to produce a known shared secret. Without a test, regressions that fail to detect such inputs go undetected. The HPKE Base Mode sender must either reject the public key at `setupBaseS` (preferred) or the X25519 layer must surface a clear error.

## Proposed change

Add unit tests that call `setupBaseS` with: (a) the all-zero 32-byte public key, (b) the canonical X25519 low-order points listed in RFC 7748 / informational HPKE guidance. Assert the expected behaviour (rejection or a defined error). If the library does not currently reject these inputs, scope expands to include the validation in `hpke.dart` or in the `OhttpKeyConfig.parse`/`validate` flow.

## Acceptance criteria

- New tests in `test/hpke_test.dart` cover at least the all-zero public key case.
- Tests assert the expected error type (library-owned exception, not a generic `Error`).
- If the implementation does not validate, the task includes the validation fix.
- Test name references RFC 7748 §6.1.
