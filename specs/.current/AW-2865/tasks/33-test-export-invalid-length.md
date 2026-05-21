# Task 33: Test HPKE `export` with invalid length

**Severity:** IMPROVEMENT
**Vector:** Test suite
**Files:** `test/hpke_test.dart` (after line 241)

**Evidence:** Cross-ref T-H2. Line 241 is the last `});` of the `export with "TestContext"` test. The `HPKE RFC 9180` group closes at line 242. No existing test feeds a zero-length or oversized length parameter to `HpkeSenderContext.export`.

## Description

The HPKE `Export` interface accepts a length parameter; RFC 9180 §5.3 caps it at `2^32 * Nh` bytes. Zero-length is a defined edge case and oversized lengths must be rejected. Without explicit tests, both behaviours are unspecified by the test suite.

## Proposed change

Add unit tests covering: (a) `length == 0` (define expected behaviour — likely returns empty `Uint8List`), (b) `length == Nh` (the canonical case, already covered indirectly), (c) `length > 2^32 * Nh` (expect a typed exception). If the implementation does not currently enforce the cap, expand scope to include the validation.

## Acceptance criteria

- New tests placed after line 241 in `test/hpke_test.dart`.
- Cover at least the three cases enumerated above.
- Tests reference RFC 9180 §5.3.
- If the implementation lacks the cap, the task includes the validation fix.
