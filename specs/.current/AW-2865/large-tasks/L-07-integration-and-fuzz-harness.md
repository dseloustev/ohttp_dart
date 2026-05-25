# Gateway integration test and fuzz / property-based harness

**Estimate:** 2d
**Priority:** P2: High
**Component:** App
**Severity:** HIGH

## Task Description

The test suite has two structural coverage gaps that compound:

1. After L-01, the existing tests (`bhttp_test`, `hpke_test`, `ohttp_test`, plus the new `key_config_cache_test`, `ohttp_session_test`, `adapters/http_adapter_test`) exercise individual layers or pieces of orchestration in isolation. None of them drives the **full integration**: `OhttpHttpClient.send` (or the lower-level `OhttpSession.send` path) end-to-end against a stubbed gateway that responds to both the KeyConfig GET and the encapsulated POST. A regression that breaks the orchestration (URL composition, header propagation, content-type handling, KeyConfig caching, timeouts, observer wiring) would not be caught by the existing unit tests. For a library whose value proposition is correctness against an external party, the minimum coverage is an integration test against a stubbed gateway.
2. The existing tests cover RFC vectors and a small set of hand-curated negative cases, but no property-based or fuzz coverage of the varint codec, BHTTP parser, or KeyConfig parser. Pathological inputs constructed by an attacker are exactly the surface fuzz testing is designed to cover. Properties such as "varint encode then decode is identity" and "any input to `parseResponse` either parses successfully or throws `FormatException`" are not currently exercised at scale.

This task creates the directory and harness that future tests can extend.

## Technical Details

### Part A — Integration test against a gateway stub

**1. New `test/integration/` directory**

The `test/integration/` directory does not currently exist. Create it and add at least one test that wires `OhttpHttpClient` against a local in-process stub. The recommended approach (no new dev-dep) is:

**MockClient from `package:http/testing.dart`**. The package already depends on `http ^1.6.0`, which transitively provides `MockClient` — no new dependency needed. The integration test constructs an `OhttpHttpClient` wired through `HttpClientTransport` wired through a `MockClient` that:

- Responds to `keysUrl` with a synthetic 41-byte KeyConfig (key_id || X25519 KEM || 32-byte pubkey || symLen=4 || HKDF-SHA256 || AES-128-GCM).
- Responds to `gatewayUrl` POSTs by parsing the encapsulated request just enough to validate the wire format, then returning a pre-baked encrypted response that decap will successfully unwrap (or a deliberately-malformed one for failure cases).

This is the cheaper option. The full crypto round trip is not testable end-to-end here because this package has no HPKE receiver implementation — the stub cannot encrypt a valid OHTTP response without one. Two ways to handle it:

(a) Restrict the integration test to **orchestration assertions** that don't require a successful decap: the encapsulated request gets posted with the right `Content-Type`, the right path, the right body shape; gateway errors propagate as the right typed exceptions; the cache is invalidated on the right events; observer events fire with the right fields. Test wraps decap in `expectLater(..., throwsA(...))` and inspects what was sent.

(b) Add a minimal **in-test HPKE receiver helper** to encrypt a canned response with the same primitives the library exposes. This is more work but gives a true round-trip integration test. The same helper already exists in `test/ohttp_test.dart` (test 7 from L-06's OHTTP round-trip work) — share it.

Document the choice in `test/integration/README.md`. Option (b) on top of L-06 is the recommended scope: the helper exists already.

Either approach keeps CI hermetic (no real network calls). The test exercises `OhttpHttpClient.send` end-to-end and asserts the orchestration path: KeyConfig discovery → BHTTP serialization → OHTTP encapsulation → gateway POST → decapsulation → BHTTP response parsing.

**Cross-task assertions the integration test should cover:**
- L-03 typed exception types fire on the matching failure paths: gateway 4xx → `OhttpHttpException`(4xx) from `HttpClientTransport`; AEAD failure → `OhttpDecryptionException` from `OhttpSession.send`.
- L-04 timeouts: `MockClient` with a delayed response above the configured `gatewayTimeout` → `OhttpTimeoutException`.
- L-04 response-cap check: stub returns a body above the cap → `OhttpParseException`.
- L-01 + L-05 caching: first `send()` triggers `onKeyConfigFetch`; second `send()` within TTL fires `onKeyConfigCacheHit` and `MockClient` only saw one GET to `keysUrl`.
- L-05 observer events fire with the documented fields on every flow (success and each failure type).

### Part B — Fuzz / property-based tests

**2. Property-based or fuzz layer**

Two practical options:

(a) Use a Dart-compatible package such as `glados` (adds a `dev_dependency`).
(b) Hand-rolled randomized generators with a fixed CI seed.

Add at least three property tests:

- **Varint round-trip identity** over random values in `[0, 2^62)`. For each generated value, `decodeVarint(encodeVarint(v))` must equal `v` and the encoded form must match the expected length for the value's range.
- **`parseResponse(randomBytes)` never throws anything other than `FormatException`** (coordinates with the BHTTP parser hardening from L-04 that converts `RangeError` to `FormatException`).
- **`OhttpKeyConfig.parse(randomBytes)` never throws anything other than the typed library exception(s)** — specifically `OhttpKeyConfigException` or `OhttpUnsupportedSuiteException` (from L-03). Random bytes that happen to parse successfully are fine.

CI seed is fixed for reproducibility; the ability to vary the seed for periodic deep runs must be documented. No single test should exceed a few seconds per CI run; cap the iteration count accordingly.

Any new exception type surfaced by fuzz inputs must be triaged and either fixed or added to the documented exception surface.

| **Platform** | All (default) |
|---|---|
| **URLs** | https://github.com/AdguardTeam/ohttp_dart |
| **Figma** | |
| **Notion** | |

## Acceptance Criteria

- `test/integration/` directory exists with at least one test file and a `README.md` documenting the chosen stub approach (`MockClient`-based, in-test receiver helper or not).
- The integration test exercises `OhttpHttpClient.send` (or `OhttpSession.send`) end-to-end against the `MockClient`-backed stubbed gateway, covering KeyConfig discovery → BHTTP serialize → OHTTP encap → gateway POST → decap → BHTTP parse.
- Integration test asserts:
  - L-03 typed exception types fire on the expected failure paths (gateway 4xx/5xx, AEAD failure).
  - L-04 timeouts fire with `OhttpTimeoutException` on configured-timeout breaches.
  - L-04 response-cap rejection with `OhttpParseException` on oversized stub responses.
  - L-01 `KeyConfigCache` short-circuits within TTL (observable as zero GETs to `keysUrl` on the second `send()`).
  - L-05 `OhttpObserver` events fire with the documented fields on success and failure flows.
- CI runs the integration test without external network access.
- At least three property/fuzz tests are added: varint round-trip identity, `parseResponse` random-byte invariant (only throws `FormatException`), `OhttpKeyConfig.parse` random-byte invariant (only throws typed library exceptions from L-03).
- CI configuration uses a fixed seed for reproducibility; the seed-override mechanism for deeper periodic runs is documented.
- No test takes longer than a few seconds per CI run.
- Any new exception types surfaced by fuzz inputs are documented in the relevant doc comments or — if unexpected — fixed.

## Additional

- Tests: integration tests + property/fuzz tests, both runnable via `dart test`.
- `MockClient` is already available transitively via the existing `http: ^1.6.0` dependency — **no new dev-dep needed** for the integration test (this is a change from the original L-06 which weighed `shelf` as an option). `glados` remains an optional dev-dep for property tests if the team prefers it over hand-rolled generators — coordinate before adding to `pubspec.yaml`.
- This task should land last because it asserts behaviour established by L-01 (the entire new public surface), L-03 (typed exceptions), L-04 (timeouts, caps), and L-05 (observer events).
- Source merged tasks (in `../merged-tasks/`): M-12 (integration + fuzz). The integration-test target changes from `OhttpClient` to `OhttpHttpClient` (or `OhttpSession.send`); the stub-approach recommendation shifts from "`shelf` server" to "`MockClient` + reuse the in-test HPKE receiver helper from L-06."
