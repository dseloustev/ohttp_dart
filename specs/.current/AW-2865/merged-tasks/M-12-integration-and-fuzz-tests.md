# Add gateway integration test and fuzz/property-based tests

**Estimate:** 2d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

The test suite has two structural coverage gaps that compound:

1. All three existing test files exercise individual layers in isolation. There is no end-to-end test of `OhttpClient.send()` against a stub that emulates a gateway's GET/POST surface. A regression that breaks the orchestration (URL composition, header propagation, content-type handling, KeyConfig caching, timeouts) would not be caught by the existing unit tests. For a library whose value proposition is correctness against an external party, the minimum coverage is an integration test against a stubbed gateway.
2. The existing tests cover RFC vectors and a small set of hand-curated negative cases, but no property-based or fuzz coverage of the varint codec, BHTTP parser, or KeyConfig parser. Pathological inputs constructed by an attacker are exactly the surface fuzz testing is designed to cover. Properties such as "varint encode then decode is identity" and "any input to `parseResponse` either parses successfully or throws `FormatException`" are not currently exercised at scale.

Both items create the directory and harness that future tests can extend.

## Technical Details

**1. Integration test against a gateway stub** (new `test/integration/` directory)

The `test/integration/` directory does not currently exist. Create it and add at least one test that wires `OhttpClient` against a local in-process stub. Two approaches are acceptable; pick one and document the choice in a `test/integration/README.md`:

(a) **Local `shelf` server**: stand up a `package:shelf` server inside the test that responds to the configured `configPath` with a synthetic KeyConfig and to `requestPath` with a synthetic OHTTP response. Exercises the full HTTP transport path. Requires adding `shelf` as a `dev_dependency`.

(b) **Recorded HTTP fixture replayed via a mock `http.Client`**: avoids any actual network listener but still exercises the orchestration in `send()`. Lower setup cost; less coverage of the actual transport.

Either approach must keep CI hermetic (no real network calls). The test exercises `OhttpClient.send()` end-to-end and asserts the orchestration path: KeyConfig discovery, BHTTP serialization, OHTTP encapsulation, gateway POST, decapsulation, BHTTP response parsing.

**2. Fuzz / property-based tests**

Introduce a property-based or fuzz layer. Two practical options:

(a) Use a Dart-compatible package such as `glados` (adds a `dev_dependency`).
(b) Hand-rolled randomized generators with a fixed CI seed.

Add at least three property tests:

- **Varint round-trip identity** over random values in `[0, 2^62)`. For each generated value, `decodeVarint(encodeVarint(v))` must equal `v` and the encoded form must match the expected length for the value's range.
- **`parseResponse(randomBytes)` never throws anything other than `FormatException`** (coordinates with the BHTTP parser hardening that converts `RangeError` to `FormatException`).
- **`OhttpKeyConfig.parse(randomBytes)` never throws anything other than the typed library exception(s)** (coordinates with the typed-exception hierarchy and the multi-suite parsing fix).

CI seed is fixed for reproducibility; the ability to vary the seed for periodic deep runs must be documented. No single test should exceed a few seconds per CI run; cap the iteration count accordingly.

Any new exception type surfaced by fuzz inputs must be triaged and either fixed or added to the documented exception surface.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

- `test/integration/` directory exists with at least one test file and a `README.md` documenting the chosen stub approach.
- The integration test exercises `OhttpClient.send()` end-to-end against the stubbed gateway, covering KeyConfig discovery → BHTTP serialize → OHTTP encap → gateway POST → decap → BHTTP parse.
- CI runs the integration test without external network access.
- At least three property/fuzz tests are added: varint round-trip identity, `parseResponse` random-byte invariant, `OhttpKeyConfig.parse` random-byte invariant.
- CI configuration uses a fixed seed for reproducibility; the seed-override mechanism for deeper periodic runs is documented.
- No test takes longer than a few seconds per CI run.
- Any new exception types surfaced by fuzz inputs are documented in the relevant doc comments or — if unexpected — fixed.

## Additional

- Tests: integration tests + property/fuzz tests, both runnable via `dart test` and gated to skip in environments without dev-dependency support.
- May require new `dev_dependencies` (`shelf` and/or `glados`) — coordinate with the team before adding to `pubspec.yaml`.
