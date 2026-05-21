# Task 21: Add integration test against a gateway stub

**Severity:** HIGH
**Vector:** Test suite
**Files:** new `test/integration/` directory

**Evidence:** Cross-ref T-X1. The `test/integration/` directory does not currently exist; this task proposes creating it. There are no end-to-end tests against either a real or stubbed gateway in the repository today.

## Description

All three existing test files exercise individual layers in isolation. There is no end-to-end test of `OhttpClient.send()` against a stub that emulates a gateway's GET/POST surface. A regression that breaks the orchestration (URL composition, header propagation, content-type handling, KeyConfig caching once task 09 lands, timeouts once task 08 lands) would not be caught by the existing unit tests. An integration test is the minimum coverage for a library whose central value proposition is correctness against an external party.

## Proposed change

Create the `test/integration/` directory and add at least one test that wires `OhttpClient` against a local in-process stub. Two approaches are acceptable to keep CI hermetic:

1. Stand up a local server using `package:shelf` inside the test that responds to the configured `configPath` (with a synthetic KeyConfig) and `requestPath` (with a synthetic OHTTP response), exercising the full HTTP transport path.
2. Use a recorded HTTP fixture replayed via a mock `http.Client`, avoiding any actual network listener but still exercising the orchestration in `send()`.

Either approach must keep CI hermetic (no real network calls). Pick one and document the choice in the test directory's `README.md`.

## Acceptance criteria

- `test/integration/` directory exists with at least one test file.
- Test exercises `OhttpClient.send()` end-to-end against a stubbed gateway.
- Stub approach is documented (shelf-based local server or recorded fixture).
- CI runs the integration test without external network access.
- Test name references the OHTTP end-to-end flow.
