# AW-2865: ohttp_dart investigation — follow-up tasks

## Executive summary

This directory is the final deliverable of investigation ticket AW-2865 (an evaluation of `ohttp_dart` for use in a non-custodial crypto wallet). The library implements the core OHTTP / HPKE / BHTTP wire formats correctly against RFC vectors, but it **must not be used in production until the 1 BLOCKER and 21 HIGH items below are resolved**. The BLOCKER is a privacy footgun in `sendDirect()` that silently sends plaintext to the relay when the optional `directBaseUrl` is omitted; the HIGH items collectively address missing input validation, missing typed errors, missing timeouts and caching, missing observability with safe logging constraints, missing zeroization of key material, and missing test coverage. The 12 IMPROVEMENT items are quality-of-life follow-ups that do not block production use but should be picked up alongside the BLOCKER/HIGH work where adjacent. All findings are deduplicated across Phases 1–6; each task below is independently actionable.

## Blockers for production use

- [Task 01 — OHTTP bypass via silent `effectiveDirectBaseUrl` fallback](./01-ohttp-bypass-send-direct.md) — `sendDirect()` silently issues plaintext HTTP to the relay when `directBaseUrl` is omitted; no encryption, no unlinkability, no runtime signal. Companion to task 32.

## All follow-up tasks

| # | Task title | Vector | Severity | File link |
|---|------------|--------|----------|-----------|
| 01 | OHTTP bypass via silent `effectiveDirectBaseUrl` fallback | Privacy risks | BLOCKER | [01](./01-ohttp-bypass-send-direct.md) |
| 02 | Enforce HTTPS scheme on `gatewayBaseUrl` | Privacy risks | HIGH | [02](./02-enforce-https-scheme.md) |
| 03 | Restrict and validate `targetAuthority` | Privacy risks | HIGH | [03](./03-restrict-target-authority.md) |
| 04 | Zeroize HPKE ephemeral key material in `HpkeSenderContext` | Privacy risks | HIGH | [04](./04-zeroize-hpke-key-material.md) |
| 05 | Zeroize OHTTP `enc` and `exportedSecret` after response decap | Privacy risks | HIGH | [05](./05-zeroize-ohttp-enc-exported-secret.md) |
| 06 | Fix silent drop of extra KDF+AEAD pairs in `OhttpKeyConfig.parse` | Cryptographic correctness | HIGH | [06](./06-fix-silent-drop-extra-kdf-aead-pairs.md) |
| 07 | Wrap `SecretBoxAuthenticationError` in a library-owned exception | Interoperability | HIGH | [07](./07-wrap-aead-auth-error.md) |
| 08 | Add timeouts to KeyConfig GET and gateway POST | Network reliability | HIGH | [08](./08-add-http-timeouts.md) |
| 09 | Implement KeyConfig caching with TTL | KeyConfig management | HIGH | [09](./09-implement-keyconfig-caching.md) |
| 10 | Add a typed error hierarchy for `OhttpClient` | Network reliability | HIGH | [10](./10-add-typed-error-hierarchy.md) |
| 11 | Convert BHTTP `RangeError` to `FormatException` via shared bounds helper | Parser robustness | HIGH | [11](./11-bhttp-bounds-check-rangeerror-to-formatexception.md) |
| 12 | Add size caps on BHTTP header section and response body | Parser robustness | HIGH | [12](./12-bhttp-size-cap-headers-and-body.md) |
| 13 | Add structured observability hooks (with strict logging constraints) | Documentation | HIGH | [13](./13-add-structured-observability-hooks.md) |
| 14 | Test HPKE sequence-number overflow in `HpkeSenderContext.seal` | Test suite | HIGH | [14](./14-test-hpke-sequence-number-overflow.md) |
| 15 | Test `setupBaseS` with invalid public key | Test suite | HIGH | [15](./15-test-setupbases-invalid-public-key.md) |
| 16 | Test QUIC varint encode/decode at the 8-byte boundary | Test suite | HIGH | [16](./16-test-varint-8byte-boundary.md) |
| 17 | Test BHTTP `parseResponse` with truncated body | Test suite | HIGH | [17](./17-test-bhttp-truncated-body.md) |
| 18 | Test OHTTP encap/decap round-trip | Test suite | HIGH | [18](./18-test-ohttp-encap-decap-round-trip.md) |
| 19 | Test OHTTP AEAD authentication failure surfaces correctly | Test suite | HIGH | [19](./19-test-ohttp-aead-auth-failure.md) |
| 20 | Test `OhttpKeyConfig.parse` with multiple KDF+AEAD pairs | Test suite | HIGH | [20](./20-test-ohttp-keyconfig-multi-suite.md) |
| 21 | Add integration test against a gateway stub | Test suite | HIGH | [21](./21-test-integration-gateway-stub.md) |
| 32 | Make `effectiveDirectBaseUrl` private or remove from public API | Privacy risks | HIGH | [32](./32-make-effective-direct-base-url-private.md) |
| 23 | Unify exception types for unsupported KEM in `parse()` and `validate()` | Cryptographic correctness | IMPROVEMENT | [23](./23-unify-unsupported-kem-exception.md) |
| 24 | Add size cap on `serializeRequest` inputs | Parser robustness | IMPROVEMENT | [24](./24-add-serialize-request-body-size-cap.md) |
| 25 | Normalize gateway URL path construction | Network reliability | IMPROVEMENT | [25](./25-normalize-gateway-url-paths.md) |
| 26 | Lowercase response header names | Interoperability | IMPROVEMENT | [26](./26-lowercase-response-header-names.md) |
| 27 | Validate BHTTP response status-code range | Parser robustness | IMPROVEMENT | [27](./27-validate-bhttp-status-code-range.md) |
| 28 | Expand empty-AAD comment at `ctx.seal` call site | Documentation | IMPROVEMENT | [28](./28-expand-empty-aad-comment.md) |
| 29 | Add plain-HKDF maintenance-hazard comments in `ohttpDecapsulate` | Documentation | IMPROVEMENT | [29](./29-add-plain-hkdf-maintenance-comments.md) |
| 30 | Add fuzz / property-based tests across the library | Test suite | IMPROVEMENT | [30](./30-add-fuzz-property-based-tests.md) |
| 31 | Improve example file documentation | Documentation | IMPROVEMENT | [31](./31-improve-example-file-documentation.md) |
| 33 | Test HPKE `export` with invalid length | Test suite | IMPROVEMENT | [33](./33-test-export-invalid-length.md) |
| 34 | Test BHTTP rejection of unknown framing indicator | Test suite | IMPROVEMENT | [34](./34-test-bhttp-unknown-framing-indicator.md) |
| 35 | Test OHTTP edge cases (empty / oversized inputs) | Test suite | IMPROVEMENT | [35](./35-test-ohttp-edge-cases.md) |

> Number 22 is intentionally absent — row 22 of the Phase 7 inventory was merged into task 13.

## Out of scope

Verbatim from `vision.md §Out of scope`:

- Implementing any of the identified improvements (this is an investigation ticket only).
- Adding new dependencies to `pubspec.yaml`.
- Changing any source file under `lib/` or `test/`.
- Designing new APIs, data models, or architecture layers.
- Writing fuzz tests, integration tests, or end-to-end tests (those are follow-up tasks).
- Evaluating alternative cryptographic libraries or cipher suites.

## Deliverable note

These `.md` files are the final deliverable of AW-2865. No Jira tickets are created automatically from this directory; engineers picking up follow-up work should reference these files directly, link to them from any subsequent ticket they open, and treat the `Severity`, `Vector`, `Files`, and `Acceptance criteria` fields as the source of truth for scoping the work.
