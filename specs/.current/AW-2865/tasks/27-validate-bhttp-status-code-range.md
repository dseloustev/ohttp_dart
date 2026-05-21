# Task 27: Validate BHTTP response status-code range

**Severity:** IMPROVEMENT
**Vector:** Parser robustness
**Files:** `lib/src/bhttp.dart:148-162`

**Evidence:** Cross-ref F2.3. Lines 148–162 of `lib/src/bhttp.dart`: the response-parsing loop reads `statusCode` as a raw varint and only skips informational `1xx` responses (`100 <= x < 200`). No validation that the final `statusCode` falls within the canonical HTTP range of 100–599 (RFC 9110 §15).

## Description

A malformed or malicious gateway response could carry a status code of 999, 0, or `2^62 - 1` and `parseResponse` would happily return it. Downstream wallet code that switches on the status code may behave unpredictably with out-of-range values. A range check at the parser layer rejects nonsense early and produces a clear error.

## Proposed change

After reading the final (non-informational) status code, validate it falls in 100–599 inclusive. Raise `FormatException` (coordinate with task 11's helper) on out-of-range values. Document the constraint in the `parseResponse` doc comment.

## Acceptance criteria

- `parseResponse` rejects status codes outside 100–599 with `FormatException`.
- Doc comment cites RFC 9110 §15.
- Unit tests cover at least: 100 (accept after informational skip), 200 (accept), 599 (accept), 600 (reject), 99 (reject), `2^62 - 1` (reject).
