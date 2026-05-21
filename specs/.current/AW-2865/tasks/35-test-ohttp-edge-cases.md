# Task 35: Test OHTTP edge cases (empty / oversized inputs)

**Severity:** IMPROVEMENT
**Vector:** Test suite
**Files:** `test/ohttp_test.dart`

**Evidence:** Cross-refs T-O5, T-O6. The existing `test/ohttp_test.dart` (165 lines) covers happy paths and the decap short-circuit but does not exercise edge cases such as an empty inner request, an empty response, or a maximally-sized payload.

## Description

Edge-case inputs — empty BHTTP payload, single-byte payload, payload sized at the boundary of any future caps from task 12/24 — surface subtle bugs in length-handling code that happy-path tests miss. A small set of these tests pins behaviour at the boundaries.

## Proposed change

Add tests covering: (a) `ohttpEncapsulate` with an empty `binaryRequest` (define expected behaviour), (b) `ohttpDecapsulate` with an empty response payload (expect a typed exception), (c) round-trip with a payload at exactly the size cap from task 12 (succeed) and one byte above (expect rejection). If task 12 has not yet shipped, the boundary tests pin current behaviour and will be updated when the cap lands.

## Acceptance criteria

- New tests in `test/ohttp_test.dart` cover at least three edge cases.
- Each test asserts the expected outcome (success with empty payload, or typed exception).
- Tests cross-reference tasks 12 and 24 in comments.
- Tests reference RFC 9458 §4.3 / §4.4 where applicable.
