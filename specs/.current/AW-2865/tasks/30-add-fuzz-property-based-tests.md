# Task 30: Add fuzz / property-based tests across the library

**Severity:** IMPROVEMENT
**Vector:** Test suite
**Files:** all test files

**Evidence:** Cross-ref T-X2. The existing test suite covers RFC vectors and a small set of hand-curated negative cases. There is no property-based or fuzz coverage of the varint codec, BHTTP parser, or KeyConfig parser. Pathological inputs constructed by an attacker are exactly the surface fuzz testing is designed to cover.

## Description

Properties such as "varint encode then decode is identity for every value in `[0, 2^62)`," "any input to `parseResponse` either parses successfully or throws `FormatException`," and "any input to `OhttpKeyConfig.parse` either parses successfully or throws a typed library exception" are not currently exercised at scale. A small property-based / fuzz layer catches whole classes of bugs that hand-written tests miss.

## Proposed change

Introduce a property-based or fuzz layer (using a Dart-compatible package such as `glados`, or hand-rolled randomized generators). Add at least three property tests:

1. Varint round-trip identity over random values in `[0, 2^62)`.
2. `parseResponse(randomBytes)` never throws anything other than `FormatException` (coordinates with task 11).
3. `OhttpKeyConfig.parse(randomBytes)` never throws anything other than the typed library exception (coordinates with tasks 06 and 23).

The CI seed should be fixed for reproducibility, with the ability to vary the seed for periodic deep runs.

## Acceptance criteria

- At least three property/fuzz tests are added.
- The CI configuration uses a fixed seed for reproducibility.
- No test takes longer than a few seconds per CI run.
- Any new exception types surfaced by fuzz inputs are catalogued back into the relevant per-task files.
