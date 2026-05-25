# AW-2865: large follow-up tasks

This directory contains the **further-consolidated** follow-up tasks for AW-2865, scaled up from the 12 files under `../merged-tasks/` into larger 2–4 day units. Each large task groups two or three logically related merged tasks (paired implementation + tests, or paired implementation workstreams that share a configuration surface) so it can be picked up as a single Jira ticket — or a single multi-day work unit — without further synthesis.

Every large task is self-contained: file paths, line ranges, RFC sections, and acceptance criteria from the source merged tasks are inlined into the relevant section. No file in this directory cross-references another file by number for its content — only the landing-order section below names them as a sequence.

## Index

| # | Title | Source merged tasks | Est. | Severity |
|---|-------|---------------------|------|----------|
| [L-01](./L-01-privacy-and-input-validation.md) | Privacy & input-validation foundation (close `sendDirect` bypass, enforce HTTPS, validate `targetAuthority`) | M-01, M-02 | 3d | BLOCKER |
| [L-02](./L-02-exception-hierarchy-and-suite-negotiation.md) | Typed `OhttpException` hierarchy + KeyConfig multi-suite negotiation | M-05, M-04 | 3d | HIGH |
| [L-03](./L-03-hardened-transport-and-bhttp-parser.md) | Hardened transport: timeouts, KeyConfig caching, URL normalization + BHTTP parser bounds/caps | M-06, M-07 | 4d | HIGH |
| [L-04](./L-04-confidentiality-and-observability.md) | Confidentiality hygiene & structured observability (zeroize key material + observer hooks + header normalization) | M-03, M-08 | 3d | HIGH |
| [L-05](./L-05-negative-path-unit-test-coverage.md) | Negative-path unit-test coverage (HPKE + BHTTP + OHTTP) and interop-hazard comments | M-09, M-10, M-11 | 3d | HIGH |
| [L-06](./L-06-integration-and-fuzz-harness.md) | Gateway integration test + fuzz / property-based harness | M-12 | 2d | HIGH |

**Total: 6 large tasks, ~18 engineer-days** (from 12 merged tasks under `../merged-tasks/`, originally 34 individually-actionable findings under `../tasks/`).

## Suggested landing order

1. **L-01** must land first — it closes the only BLOCKER (the `sendDirect` privacy bypass) and the matching input-validation surface that every subsequent task assumes.
2. **L-02** lands second so the typed exception hierarchy is in place before any other task introduces new throw sites (L-03's timeouts and parser caps, L-04's decryption-failure observer event, L-06's integration assertions).
3. **L-03** and **L-04** can be done in parallel — L-03 touches `OhttpClient.send` orchestration and the BHTTP parser; L-04 touches `hpke.dart` zeroization, `ohttp.dart` decap-side cleanup, and the observability surface. No file-level conflicts.
4. **L-05** lands after L-03 (its BHTTP truncation tests assert the `FormatException`s introduced by L-03's parser hardening) and benefits from L-02 already having landed.
5. **L-06** lands last because it asserts behaviour established by L-01 (sendDirect surface), L-02 (typed exceptions), L-03 (timeouts, caching, parser caps), and L-04 (observer events).

Critical path with L-03 / L-04 parallelized: ~15 engineer-days.

## Relationship to `../merged-tasks/` and `../tasks/`

- `../tasks/` — original 34 individually-actionable findings; **source of truth** for individual file/line citations and severity.
- `../merged-tasks/` — packaging layer at 1–2 day granularity (12 files); useful when one of these large tasks needs to be split back into smaller engineering tickets.
- `large-tasks/` (this directory) — packaging layer at 2–4 day granularity (6 files); intended as the planning unit a single engineer picks up for a multi-day work block.

If a large task is dropped or split during planning, return to `../merged-tasks/` for the constituent merged tasks, or all the way back to `../tasks/` for unmerged findings.

## Out of scope

Same as the parent ticket (`../vision.md §Out of scope`):

- Implementing any of the identified improvements (this directory is planning output; implementation happens in the resulting Jira tickets).
- Adding new dependencies to `pubspec.yaml` (the integration / fuzz task in L-06 explicitly calls out the dev-dependency decision).
- Changing any source file under `lib/` or `test/` (that work happens under the resulting Jira tickets).
