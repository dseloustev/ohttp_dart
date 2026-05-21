# Phase 5: Review test suite coverage gaps

**Goal:** Read all three test files and identify negative-case, fuzz/property, and end-to-end test gaps relative to the implementation surface.

## Context

**Feature motivation (from idea):** Investigation of `ohttp_dart` for potential use in a non-custodial crypto wallet. The goal is not to implement improvements but to identify risks and form subsequent engineering tasks. Every finding must map to a specific test file, missing scenario, and severity.

**Severity classification (from vision §2):**
- `BLOCKER` — must fix before wallet use
- `HIGH` — should fix before wallet use
- `IMPROVEMENT` — desirable but not blocking

**Technical approach (from vision §3):** The test suite is read-only — all three files are `UNCHANGED` in this investigation. The audit reads them to identify coverage gaps, not to add or modify tests.

```
test/
├── bhttp_test.dart    # UNCHANGED — varint round-trip tests; review for missing negative/fuzz cases
├── hpke_test.dart     # UNCHANGED — RFC 9180 A.1 vector tests; review coverage gaps
└── ohttp_test.dart    # UNCHANGED — OHTTP encap/decap tests; review for missing failure paths
```

**Architecture context (from vision §4):** The test suite mirrors the four-layer stack:
- `hpke_test.dart` covers Layer 1 (`hpke.dart`) — the HPKE Base Mode sender
- `bhttp_test.dart` covers Layer 2 (`bhttp.dart`) — the BHTTP framing
- `ohttp_test.dart` covers Layer 3 (`ohttp.dart`) and some Layer 4 integration

Each test file uses RFC vector injection or handcrafted bytes; there are no integration tests against a live gateway and no fuzz/property-based tests.

**Known test-seam from CLAUDE.md:** `setupBaseS` in `hpke.dart` accepts an optional `testKeyPair` parameter to make RFC 9180 Appendix A.1 vectors reproducible. This hook is reachable from production call sites — keeping it is intentional, but it must remain functional after any `hpke.dart` refactor.

**Cross-phase context — findings that expose test gaps:**

Phase 1 (HPKE) identified:
- F-6: Sequence-number overflow is untested — `_seq` guard path not exercised.
- F-7: No integration test against a reference Go gateway to confirm wire-format interop (empty-AAD deviation could fail silently).

Phase 2 (BHTTP) identified:
- R-6: No negative-case tests for truncated body, oversized `symLen`, unknown framing indicator, or malformed header.

Phase 3 (OHTTP) identified:
- TASK-O7: Missing test groups — round-trip, auth failure (tampered ciphertext), multi-suite KeyConfig, truncated ciphertext.

Phase 4 (Client) confirmed:
- No test for timeout/retry/cancellation paths, no test for `sendDirect()` bypass warning absence, no test for `http://` scheme being accepted silently.

**Cross-cutting data model concerns from vision §5 that test gaps expose:**

| Concern | Severity | Test gap |
|---|---|---|
| `_seq` overflow guard throws `StateError` (not typed exception) | HIGH | No test exercises the overflow path |
| `RangeError` instead of `FormatException` on truncated BHTTP body | HIGH | No negative test for truncated input |
| `SecretBoxAuthenticationError` propagates unwrapped | HIGH | Auth-failure path not directly tested |
| Silent ignore of extra KDF+AEAD pairs in KeyConfig | HIGH | No `symLen > 4` test case |
| No size caps on response body / headers | HIGH | No oversized-input test |

## Tasks

### `test/hpke_test.dart`
- [x] 5.1 Read the file.
- [x] 5.2 Confirm it uses `testKeyPair` injection to reproduce RFC 9180 Appendix A.1 vectors — note which specific vectors are covered.
- [x] 5.3 Identify missing cases: sequence-number overflow, `export()` with wrong length, invalid public key input.

### `test/bhttp_test.dart`
- [x] 5.4 Read the file.
- [x] 5.5 Confirm varint round-trip tests cover 1/2/4/8-byte boundaries.
- [x] 5.6 Identify missing cases: truncated body, oversized `symLen`, unknown framing indicator, malformed header (no colon separator).

### `test/ohttp_test.dart`
- [x] 5.7 Read the file.
- [x] 5.8 Identify missing cases: malformed KeyConfig (short buffer, extra KDF+AEAD pairs), AEAD auth failure (tampered ciphertext), empty response body, KeyConfig with unknown KEM/KDF/AEAD IDs.
- [x] 5.9 Note that there are no integration tests against a live gateway.
- [x] 5.10 Note absence of fuzz / property-based tests across all three files.
- [x] 5.11 Record each gap as a draft task entry with the specific test file, missing scenario, and severity.

## Acceptance Criteria

**Test:** No code is changed. Verification is: the gap list is grounded in specific test-file line counts and identified scenario names — no vague "needs more tests" entries.

## Dependencies

- Phase 4 complete (client layer audit)

## Technical Details

**Known RFC vector coverage in `hpke_test.dart` (from CLAUDE.md):**

`hpke_test.dart` validates against RFC 9180 Appendix A.1 vectors using the `testKeyPair` injection hook in `setupBaseS`. The hook makes the ephemeral key pair deterministic so the test can assert exact `enc`, `ciphertext`, and exporter output values. **The `testKeyPair` parameter must remain in `setupBaseS` when refactoring `hpke.dart`** — removing it silently breaks these vector tests by switching to random ephemerals.

**Expected varint coverage in `bhttp_test.dart` (from CLAUDE.md):**

`bhttp_test.dart` covers QUIC varint round-trips at the 1/2/4/8-byte boundaries. Specifically look for: values at exact boundary edges (63, 64, 16383, 16384, 2^30−1, 2^30, 2^62−1).

**Expected positive-path coverage in `ohttp_test.dart` (from CLAUDE.md / vision §4):**

Based on the code, `ohttp_test.dart` likely covers:
- `OhttpKeyConfig.parse` on a valid config blob
- `ohttpEncapsulate` on a valid BHTTP request
- `ohttpDecapsulate` on a valid response (or simulated gateway response)

Known confirmed gap from Phase 3 (TASK-O7): the auth-failure path (`ohttp.dart:219`) is not directly tested — `test/ohttp_test.dart:149-163` covers only short-response rejection.

**Eight investigation vectors from idea.md — test suite relevance:**

| Vector | Test file | Expected gap |
|---|---|---|
| Cryptographic correctness | `hpke_test.dart` | Overflow path, wrong-length export |
| Interoperability | all | No live-gateway integration test |
| Parser robustness | `bhttp_test.dart`, `ohttp_test.dart` | Truncated input, malformed KeyConfig |
| Privacy risks | none | No test that `sendDirect()` is not accidentally invoked |
| Network reliability | none | No timeout/retry test |
| KeyConfig management | `ohttp_test.dart` | No multi-suite or rotation test |
| Test suite | all | Fuzz/property tests absent across all files |
| Documentation | none | No test that inline comments match behavior |

**Instructions for recording draft task entries (from vision §2):**

Each gap noted must become a draft task entry with:
- Specific test file (e.g., `test/hpke_test.dart`)
- Missing scenario name (e.g., "sequence-number overflow → `StateError`")
- Line range in the existing test file where coverage stops
- Proposed new test group name
- Severity: `BLOCKER` / `HIGH` / `IMPROVEMENT`

Entries feed into Iteration 7 (compile into per-task Markdown files under `specs/.current/AW-2865/tasks/`).

## Implementation Notes

This is a read-only audit iteration. Do not edit any file in `lib/` or `test/`. Record all findings as draft task entries in this file for use in Iteration 7 (compile into per-task Markdown files).

Phases 1–4 are complete with findings in their respective `tasks.md` files. Cross-reference those findings when noting test gaps that confirm or extend lower-layer concerns (e.g., a missing overflow test in `hpke_test.dart` confirming the Phase 1 F-6 finding, or a missing auth-failure test in `ohttp_test.dart` confirming the Phase 3 TASK-O7 finding).

---

## Findings

### 5.1–5.3 `test/hpke_test.dart` (296 lines)

**Vectors confirmed covered (task 5.2):**

The file uses `testKeyPair` injection (e.g., lines 85–94, 105–114, 126–133) in every test that needs deterministic output. The following RFC 9180 Appendix A.1 vectors are explicitly asserted:

| Vector | Test name | Lines |
|---|---|---|
| `key`, `base_nonce`, `exporter_secret` | "SetupBaseS produces correct key, base_nonce, exporter_secret" | 82–101 |
| Ciphertext for seq=0 | "seal produces correct ciphertext for seq=0" | 104–119 |
| Export with context `\x00` | "export produces correct value" | 121–136 |
| `shared_secret` intermediate | "shared_secret intermediate value matches RFC" | 138–175 |
| Ciphertext for seq=1 | "seal seq=1 produces correct ciphertext (nonce increment)" | 177–200 |
| Export with empty context | "export with empty context produces correct value" | 202–219 |
| Export with "TestContext" | "export with 'TestContext' produces correct value" | 222–241 |

Also present: RFC 5869 HKDF Test Case 1 (lines 244–258) and three HKDF utility tests (lines 261–295).

**Missing cases (task 5.3):**

- **T-H1 — Sequence-number overflow** (`test/hpke_test.dart`, after line 200): The `_seq` overflow guard in `hpke.dart` throws `StateError` when the 12-byte nonce space is exhausted, but no test exercises this path. Severity: **HIGH** (confirms Phase 1 F-6).
  - Missing scenario: call `ctx.seal()` until `_seq` wraps past the Nn-byte maximum, assert `StateError`.

- **T-H2 — `export()` with L = 0 or L > 255 × Nh** (`test/hpke_test.dart`, after line 241): The three export tests use lengths 32 or the RFC vector value; no test passes an invalid length (0 or a value exceeding the HKDF-Expand output limit). Severity: **IMPROVEMENT**.
  - Missing scenario: `ctx.export(context, 0)` should either succeed (empty output) or throw, but behavior is untested.

- **T-H3 — Invalid (all-zero) public key** (`test/hpke_test.dart`): No test passes an all-zero or otherwise invalid X25519 public key to `setupBaseS`. HKDF over the low-order point produces a weak shared secret; behavior is untested. Severity: **HIGH**.
  - Missing scenario: `HpkeSender.setupBaseS(Uint8List(32), info)` should produce a deterministic (though weak) result or throw, but neither outcome is verified.

---

### 5.4–5.6 `test/bhttp_test.dart` (231 lines)

**Varint boundary coverage (task 5.5):**

The encode tests cover the 1-byte range (0, 1, 63), 2-byte range (64, 16383), and 4-byte range (16384, 1073741823). The decode tests cover 1-byte, 2-byte, and 4-byte round-trips plus an offset-based decode. The roundtrip group (lines 61–68) explicitly exercises [0, 1, 63, 64, 100, 255, 16383, 16384, 100000].

**Gap**: The 8-byte varint range (values ≥ 2^30, encoded with prefix `0xC0`) is **not tested** at all — neither encoding nor decoding. The 8-byte boundary (2^30 = 1073741824 through 2^62−1) has zero coverage. Severity: **HIGH** (confirms Phase 2 gap).

**Missing cases (task 5.6):**

- **T-B1 — 8-byte varint encode/decode** (`test/bhttp_test.dart`, after line 68): No test for values ≥ 1073741824. Severity: **HIGH**.
  - Missing scenario: `encodeVarint(1073741824)` → `[0xC0, 0x00, 0x00, 0x00, 0x40, 0x00, 0x00, 0x00]`; `decodeVarint` round-trip.

- **T-B2 — Truncated body raises `RangeError` instead of `FormatException`** (`test/bhttp_test.dart`, after line 185): `parseResponse` calls `sublist` on the raw bytes without a bounds check; truncated input causes `RangeError`, not `FormatException`. No negative test covers this path. Severity: **HIGH** (confirms Phase 2 R-6).
  - Missing scenario: build a response buffer with a body length varint of 100 but only 5 actual bytes; call `parseResponse` and assert `FormatException` (or document that `RangeError` is acceptable, which it is not for library callers).

- **T-B3 — Unknown framing indicator** (`test/bhttp_test.dart`): Only framing `0` (request) and `1` (response) are tested. No test passes framing byte `2` or `0xFF` to `parseResponse`. Severity: **IMPROVEMENT**.
  - Missing scenario: `parseResponse(Uint8List.fromList([0x02, ...]))` should throw `FormatException`.

- **T-B4 — Malformed header (no colon / zero-length name)** (`test/bhttp_test.dart`): The positive-path header test (lines 143–171) builds well-formed name/value pairs. No test covers a zero-length name, a name varint that declares more bytes than remain, or a value that is missing. Severity: **IMPROVEMENT**.

- **T-B5 — `serializeRequest` with no size cap** (`test/bhttp_test.dart`): No test verifies behavior when body size would cause the `BytesBuilder` to exhaust memory. Not directly testable in a unit test, but a documented boundary (max body < 2^62 bytes) is absent. Severity: **IMPROVEMENT**.

---

### 5.7–5.11 `test/ohttp_test.dart` (164 lines)

**Coverage summary (task 5.7):**

The file covers: `OhttpKeyConfig.parse` on valid input (lines 9–26), two error paths (`too-short data` at line 28, `unsupported KEM` at line 35, `short symmetric section` at line 50), all three `OhttpKeyConfig.validate` rejection cases (lines 65–108), `ohttpEncapsulate` structure check (lines 111–146), and two `ohttpDecapsulate` short-response rejection tests (lines 149–163).

**Missing cases (task 5.8):**

- **T-O1 — `symLen > 4` silently ignores extra KDF+AEAD pairs** (`test/ohttp_test.dart`, after line 62): A KeyConfig with `sym_len = 8` (two KDF+AEAD pairs) is never tested. The parser reads only the first pair and discards the rest without error — this silent truncation is untested. Severity: **HIGH** (confirms Phase 3 finding).
  - Missing scenario: build a config with `sym_len = 8`, two KDF+AEAD pairs where the second is the supported suite; assert parse succeeds and returns the first pair only (or document that the second pair is reachable by re-ordering).

- **T-O2 — `symLen` that implies bytes beyond buffer end** (`test/ohttp_test.dart`, after line 62): A `sym_len` field that points past the end of the buffer currently raises `RangeError`, not `FormatException`. No test covers this path. Severity: **HIGH** (confirms Phase 3 finding).
  - Missing scenario: `sym_len = 0xFF, 0xFF` with only 4 bytes remaining; assert `FormatException`.

- **T-O3 — AEAD authentication failure (tampered ciphertext)** (`test/ohttp_test.dart`, after line 163): `ohttpDecapsulate` calls `aesGcm.decryptList` which throws `SecretBoxAuthenticationError` on tamper; no test flips a ciphertext byte and asserts this exception (or a wrapping `OhttpDecapException`). Severity: **HIGH** (confirms Phase 3 TASK-O7).
  - Missing scenario: encapsulate a request, flip one byte in the ciphertext region, call `ohttpDecapsulate`, assert `SecretBoxAuthenticationError` (or typed wrapper).

- **T-O4 — Full encap/decap round-trip** (`test/ohttp_test.dart`): There is no test that encapsulates a request, simulates a gateway response via `ohttpDecapsulate`, and asserts the recovered plaintext matches. The positive path through `ohttpDecapsulate` is completely untested. Severity: **HIGH**.
  - Missing scenario: generate a fresh X25519 key pair, build a KeyConfig pointing to its public key, call `ohttpEncapsulate`, construct a synthetic HKDF-derived response (using the exported secret and a random nonce), call `ohttpDecapsulate`, assert plaintext recovery.

- **T-O5 — `KeyConfig.parse` inconsistency: `FormatException` vs. `UnsupportedError`** (`test/ohttp_test.dart`, lines 35–47 and 65–85): The parse-path test (line 35) asserts `FormatException` for unsupported KEM; the validate-path test (line 77) asserts `UnsupportedError` for the same condition. No test exercises the `parse → validate` combined path to document that callers may see either exception type. Severity: **IMPROVEMENT**.

- **T-O6 — Empty binary request body through encapsulate** (`test/ohttp_test.dart`, line 125): The encapsulate test uses a 5-byte `binaryRequest`. No test uses an empty (`Uint8List(0)`) inner payload. Severity: **IMPROVEMENT**.

**No live-gateway integration tests (task 5.9):**

Confirmed: there are no integration tests against a live or stub gateway. The `test/` directory contains exactly three unit test files. All OHTTP-layer positive-path tests use either in-process key generation or handcrafted byte buffers. A stubbed HTTP server (e.g., using `package:shelf`) could provide repeatable gateway simulation without a live endpoint. Severity: **HIGH** (missing coverage of full OHTTP wire format, including gateway POST response parsing).

**Absence of fuzz / property-based tests (task 5.10):**

Confirmed across all three files:
- No use of `package:test` `forAll` / hypothesis-style generators.
- No use of `dart fuzz` tooling.
- No randomized round-trip property tests (e.g., `∀ value ∈ [0, 2^62): encodeVarint(decodeVarint(encodeVarint(value))) == value`).
- No mutation-testing of ciphertext bytes to verify AEAD rejection.

Severity: **IMPROVEMENT** for the varint property gap; **HIGH** for the absence of tamper-detection fuzz tests given the cryptographic nature of the library.

**Draft task entries summary (task 5.11):**

| ID | Test file | Missing scenario | Proposed group | Severity |
|---|---|---|---|---|
| T-H1 | `test/hpke_test.dart:200` | Sequence-number overflow → `StateError` | `HpkeSenderContext.seal overflow` | HIGH |
| T-H2 | `test/hpke_test.dart:241` | `export()` with L=0 or L>255×Nh | `HpkeSenderContext.export edge cases` | IMPROVEMENT |
| T-H3 | `test/hpke_test.dart` | All-zero public key in `setupBaseS` | `setupBaseS invalid key` | HIGH |
| T-B1 | `test/bhttp_test.dart:68` | 8-byte varint encode/decode (≥2^30) | `varint 8-byte boundary` | HIGH |
| T-B2 | `test/bhttp_test.dart:185` | Truncated body → `RangeError` vs `FormatException` | `parseResponse negative cases` | HIGH |
| T-B3 | `test/bhttp_test.dart` | Unknown framing indicator → `FormatException` | `parseResponse negative cases` | IMPROVEMENT |
| T-B4 | `test/bhttp_test.dart` | Zero-length header name / truncated header value | `parseResponse negative cases` | IMPROVEMENT |
| T-O1 | `test/ohttp_test.dart:62` | `symLen > 4` silently drops extra KDF+AEAD pairs | `OhttpKeyConfig.parse extra pairs` | HIGH |
| T-O2 | `test/ohttp_test.dart:62` | `symLen` points past buffer end → `RangeError` | `OhttpKeyConfig.parse negative` | HIGH |
| T-O3 | `test/ohttp_test.dart:163` | AEAD auth failure on tampered ciphertext | `ohttpDecapsulate auth failure` | HIGH |
| T-O4 | `test/ohttp_test.dart` | Full encap/decap positive round-trip | `ohttpEncapsulate + ohttpDecapsulate roundtrip` | HIGH |
| T-O5 | `test/ohttp_test.dart:35,77` | `parse` vs `validate` exception-type inconsistency | `OhttpKeyConfig exception consistency` | IMPROVEMENT |
| T-O6 | `test/ohttp_test.dart:125` | Empty inner payload through `ohttpEncapsulate` | `ohttpEncapsulate edge cases` | IMPROVEMENT |
| T-X1 | all | No live-gateway / shelf-stub integration tests | `integration` (new file) | HIGH |
| T-X2 | all | No fuzz / property-based tests | `property tests` (new file) | IMPROVEMENT |
