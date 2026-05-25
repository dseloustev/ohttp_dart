# AW-2865: large follow-up tasks

This directory contains the **further-consolidated** follow-up tasks for AW-2865, packaged into multi-day units. The first task (L-01) is the HTTP-client-agnostic package restructure, derived from the design spec and implementation plan at `docs/superpowers/{specs,plans}/2026-05-25-http-client-agnostic-integration*.md`. The remaining six (L-02..L-07) are scaled up from the 12 files under `../merged-tasks/`; each groups logically related merged tasks (paired implementation + tests, or paired implementation workstreams that share a configuration surface) so it can be picked up as a single Jira ticket — or a single multi-day work unit — without further synthesis.

Every large task is self-contained: file paths, type names, RFC sections, and acceptance criteria from the source material are inlined into the relevant section. No file in this directory cross-references another file by number for its content — only the landing-order section below names them as a sequence.

## Index

| # | Title | Source | Est. | Severity |
|---|-------|--------|------|----------|
| [L-01](./L-01-http-client-agnostic-restructure.md) | HTTP-client-agnostic package restructure (core + opt-in `http` adapter) | spec + plan | 5d | BLOCKER |
| [L-02](./L-02-input-validation.md) | Input validation for `HttpClientTransport` and `OhttpRequestData` (HTTPS scheme + authority shape) | M-02 | 1.5d | HIGH |
| [L-03](./L-03-exception-hierarchy-and-suite-negotiation.md) | Typed `OhttpException` hierarchy + KeyConfig multi-suite negotiation | M-05, M-04 | 3d | HIGH |
| [L-04](./L-04-hardened-transport-and-bhttp-parser.md) | Hardened transport: HTTP timeouts, OHTTP-layer response cap + BHTTP parser bounds/caps | M-06 (partial), M-07 | 2.5d | HIGH |
| [L-05](./L-05-confidentiality-and-observability.md) | Confidentiality hygiene & structured observability (zeroize key material + typed observer + header normalization) | M-03, M-08 | 3d | HIGH |
| [L-06](./L-06-negative-path-unit-test-coverage.md) | Negative-path unit-test coverage (HPKE + BHTTP + OHTTP) and interop-hazard comments | M-09, M-10, M-11 | 3d | HIGH |
| [L-07](./L-07-integration-and-fuzz-harness.md) | Gateway integration test + fuzz / property-based harness | M-12 | 2d | HIGH |

**Total: 7 large tasks, ~20 engineer-days** (was 6 / 18 — the +5d restructure offsets ~3d of merged-task work that the restructure subsumes; see "Restructure impact" below).

## Suggested landing order

1. **L-01 must land first** — it is the BLOCKER. Every other task targets file paths, type names (`OhttpTransport`, `OhttpSession`, `OhttpRequestData`, `OhttpResponseData`, `HttpClientTransport`, `OhttpHttpClient`, `KeyConfigCache`, `OhttpGatewayException`), and concerns that only exist after the restructure. The legacy `OhttpClient`, `OhttpResponse`, `OhttpHeader`, `OhttpGatewayConfig`, and `sendDirect()` are removed in this task; subsequent tasks assume they are gone.
2. **L-02** lands second so the construction-time input validation (HTTPS scheme on `HttpClientTransport`, authority shape on `OhttpRequestData`) is in place before any other task introduces a new construction surface.
3. **L-03** lands third so the typed `OhttpException` hierarchy is in place before any other task introduces new throw sites (L-04's timeouts and parser caps, L-05's decryption-failure observer event, L-07's integration assertions). The L-01 `OhttpGatewayException` is folded into the hierarchy as `OhttpHttpException` in this task.
4. **L-04** and **L-05** can be done in parallel — L-04 touches `HttpClientTransport` (timeouts), `OhttpSession.send` (response-cap check), and `bhttp.dart` (parser hardening); L-05 touches `hpke.dart` (zeroization), `ohttp.dart` (decap-side zeroization), and `OhttpSession.send` (observer wiring + header lowercasing). Modest coordination is needed for the shared `ohttp_session.dart` file.
5. **L-06** lands after L-04 (its BHTTP truncation tests assert the `FormatException`s introduced by L-04's parser hardening) and benefits from L-03 already having landed.
6. **L-07** lands last because it asserts behaviour established by L-01 (entire new public surface), L-03 (typed exceptions), L-04 (timeouts, caps), and L-05 (observer events).

Critical path with L-04 / L-05 parallelized: ~16 engineer-days (5 + 1.5 + 3 + max(2.5, 3) + 3 + 2 ≈ 17 with handoffs).

## Restructure impact (what changed when L-01 was added)

The L-01 restructure subsumes several concerns from the original L-01..L-06 task set. Authoritative source of truth for individual file/line citations is still `../tasks/` / `../merged-tasks/`, but several of those tasks are partially or fully **superseded** by the restructure:

- **M-01 `sendDirect` privacy bypass** — fully dropped. `sendDirect()` and `OhttpGatewayConfig.directBaseUrl` no longer exist in the new architecture. A consumer wanting plaintext HTTP just uses a raw `http.Client` directly; the library no longer exposes a method whose name suggests "OHTTP" but silently sends plaintext.
- **M-02 gateway config validation** — preserved; retargeted from `OhttpGatewayConfig` to `HttpClientTransport` + `OhttpRequestData` constructors.
- **M-06 timeouts / KeyConfig caching / URL normalization** — partial: the **caching** half is satisfied by L-01's `KeyConfigCache` (TTL + single-flight + manual invalidation + injectable clock + cache invalidation on `OhttpGatewayException`); the **URL normalization** half is eliminated by L-01's switch from string concatenation to `Uri keysUrl` / `Uri gatewayUrl` on `HttpClientTransport`; the **timeout** half survives in L-04, retargeted to `HttpClientTransport`.
- **M-08 observability hooks + header normalization** — preserved; the old `OhttpClient.onLog` string callback is deleted with `OhttpClient` itself, and the new observer surface (`OhttpObserver`) attaches to `OhttpSession.send`. Header lowercasing retargets from `ohttp_client.dart:139` to `OhttpSession.send`.

If a future planner needs to drop or split a large task, they should compare its scope against L-01 first to confirm the concern is not already covered.

## Relationship to `../merged-tasks/` and `../tasks/`

- `../tasks/` — original 34 individually-actionable findings; **source of truth** for individual file/line citations and severity.
- `../merged-tasks/` — packaging layer at 1–2 day granularity (12 files); useful when one of these large tasks needs to be split back into smaller engineering tickets. Note that several merged-tasks file/line citations are now stale after L-01 (see "Restructure impact" above).
- `large-tasks/` (this directory) — packaging layer at 1.5–5 day granularity (7 files); intended as the planning unit a single engineer picks up for a multi-day work block.

If a large task is dropped or split during planning, return to `../merged-tasks/` for the constituent merged tasks, or all the way back to `../tasks/` for unmerged findings — but cross-check with L-01 before assuming a finding still applies.

## Out of scope

Same as the parent ticket (`../vision.md §Out of scope`):

- Implementing any of the identified improvements (this directory is planning output; implementation happens in the resulting Jira tickets).
- Adding new dependencies to `pubspec.yaml` (L-07 explicitly calls out the `glados` dev-dependency decision; the integration test no longer needs `shelf` because `MockClient` is already available transitively via `http`).
- Changing any source file under `lib/` or `test/` (that work happens under the resulting Jira tickets).
- Supporting cipher suites other than the fixed `(0x0020, 0x0001, 0x0001)` triple end-to-end (L-03 only fixes multi-suite **parsing**; supporting additional suites is a separate effort).
- A Dio adapter (the L-01 architecture supports it without core changes, but the implementation is deferred).
