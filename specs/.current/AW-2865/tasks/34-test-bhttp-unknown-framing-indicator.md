# Task 34: Test BHTTP rejection of unknown framing indicator

**Severity:** IMPROVEMENT
**Vector:** Test suite
**Files:** `test/bhttp_test.dart`

**Evidence:** Cross-refs T-B3, T-B4. RFC 9292 §3.3 defines framing indicators `0` (Known-Length Request), `1` (Known-Length Response), `2` (Indeterminate-Length Request), and `3` (Indeterminate-Length Response). The library supports only `0` and `1`. The existing tests cover `rejects non-response framing` but do not exhaustively test all unknown / indeterminate-length indicators.

## Description

A gateway response that carries framing indicator `2`, `3`, or any out-of-range value should be rejected with a clear `FormatException`. The current test suite covers only the immediate-neighbour case. A broader set of negative tests pins the rejection of every non-supported indicator value.

## Proposed change

Add unit tests covering: framing indicator `2` (indeterminate-length request), `3` (indeterminate-length response), `4` (out-of-range valid varint), and a large varint value. Assert each produces a `FormatException` with a clear message naming the indicator value.

## Acceptance criteria

- New tests in `test/bhttp_test.dart` exhaustively cover unsupported framing indicator values.
- Each test asserts `FormatException` with a message that includes the offending indicator.
- Tests reference RFC 9292 §3.3.
