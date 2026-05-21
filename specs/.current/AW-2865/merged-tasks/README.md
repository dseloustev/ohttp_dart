# AW-2865: merged follow-up tasks

This directory contains the **merged** follow-up tasks for AW-2865, consolidated from the 34 individually-actionable findings under `../tasks/`. Each merged task is sized for 1–2 days of work and groups logically related sub-items (typically one implementation change plus its companion tests, or one test file's worth of negative-path coverage) so it can be picked up as a single Jira ticket without further synthesis.

Every merged task is self-contained: the file paths, line ranges, RFC sections, and acceptance criteria from the source items are inlined into the relevant section. No file in this directory cross-references another file by number.

## Index

| # | Title | Source tasks | Est. | Severity |
|---|-------|--------------|------|----------|
| [M-01](./M-01-harden-send-direct.md) | Eliminate `sendDirect` privacy bypass and update example | 01, 31, 32 | 2d | BLOCKER |
| [M-02](./M-02-validate-gateway-config.md) | Validate `gatewayBaseUrl` scheme and `targetAuthority` | 02, 03 | 1d | HIGH |
| [M-03](./M-03-zeroize-key-material.md) | Zeroize HPKE and OHTTP sensitive key material | 04, 05 | 1d | HIGH |
| [M-04](./M-04-keyconfig-multi-suite.md) | Fix KeyConfig multi-suite negotiation and unify unsupported-suite exception | 06, 20, 23 | 1d | HIGH |
| [M-05](./M-05-typed-exception-hierarchy.md) | Introduce typed `OhttpException` hierarchy and wrap AEAD authentication errors | 07, 10 | 2d | HIGH |
| [M-06](./M-06-timeouts-caching-url-paths.md) | Add HTTP timeouts, KeyConfig caching, and normalize URL paths | 08, 09, 25 | 2d | HIGH |
| [M-07](./M-07-harden-bhttp-parser.md) | Harden BHTTP parser (bounds, size caps, status code, request cap) | 11, 12, 24, 27 | 2d | HIGH |
| [M-08](./M-08-observability-hooks.md) | Add structured observability hooks and normalize response header names | 13, 26 | 2d | HIGH |
| [M-09](./M-09-hpke-test-coverage.md) | Expand HPKE test coverage (overflow, invalid pubkey, export bounds) | 14, 15, 33 | 1d | HIGH |
| [M-10](./M-10-bhttp-test-coverage.md) | Expand BHTTP test coverage (varint, truncation, framing) | 16, 17, 34 | 1d | HIGH |
| [M-11](./M-11-ohttp-test-coverage-and-interop-comments.md) | Expand OHTTP test coverage and document interop hazards | 18, 19, 28, 29, 35 | 1d | HIGH |
| [M-12](./M-12-integration-and-fuzz-tests.md) | Add gateway integration test and fuzz/property-based tests | 21, 30 | 2d | HIGH |

**Total: 12 merged tasks** (from 34 source items in `../tasks/`).

## Relationship to `../tasks/`

The per-task files in `../tasks/` remain the source of truth for individual findings (file/line citations, severity, vector). The files here are a packaging layer for engineering planning — each can be copy-pasted into a Jira ticket with minimal editing.

If a merged task is dropped or split during planning, return to `../tasks/` for the unmerged version of each constituent finding.

## Suggested landing order

1. **M-01** must land first — it closes the only BLOCKER (the `sendDirect` privacy bypass).
2. **M-05** (typed exception hierarchy) should land before tasks that introduce new throw sites (**M-04**, **M-06**, **M-07**, **M-12**) so the new throws use the typed surface from day one.
3. **M-07** (BHTTP parser hardening) should land before **M-10** (BHTTP test coverage), since several of M-10's tests assert `FormatException` that only arrives after M-07.
4. The remaining tasks are largely independent and can be parallelized.

## Out of scope

Same as the parent ticket (`../vision.md §Out of scope`):

- Implementing any of the identified improvements (this directory is planning output; implementation happens in the resulting Jira tickets).
- Adding new dependencies to `pubspec.yaml` (the integration / fuzz task in M-12 explicitly calls out the dev-dependency decision).
- Changing any source file under `lib/` or `test/` (that work happens under the resulting Jira tickets).
