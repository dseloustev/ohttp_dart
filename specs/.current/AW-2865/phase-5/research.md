# Phase 5 Research: Test Suite Coverage Gap Review

Status: RESEARCH_COMPLETE

---

## Phase Scope

This research document supports Phase 5 of the AW-2865 investigation of `ohttp_dart` for potential use in a non-custodial crypto wallet. The scope is read-only: no file under `lib/`, `test/`, `pubspec.yaml`, or `example/` is modified. All draft task entries feed into Iteration 7 (compile into per-task Markdown files under `specs/.current/AW-2865/tasks/`).

**Severity scale (vision §2):**
- `BLOCKER` — must fix before wallet use
- `HIGH` — should fix before wallet use
- `IMPROVEMENT` — desirable but not blocking

---

## Test File Coverage Summary

### `test/hpke_test.dart` — 296 lines

**Test groups:**

| Group | Tests | What it covers |
|---|---|---|
| `X25519 DH diagnostic` | 1 | RFC 7748 §6.1 DH test vector with `package:cryptography` |
| `HPKE RFC 9180 Appendix A.1 test vectors` | 6 | DHKEM(X25519) + HKDF-SHA256 + AES-128-GCM, Base Mode |
| `HKDF RFC 5869 test vectors` | 1 | Plain HKDF extract + expand (Test Case 1) |
| `HKDF utilities` | 3 | Empty-salt extract, expand length 16, expand length 32 |

**RFC 9180 Appendix A.1 sub-vectors covered:**

| Sub-vector | Assertion | Status |
|---|---|---|
| `enc` (ephemeral public key) | `ctx.enc == pkEm` | COVERED (line 97) |
| `key` (AEAD key) | `ctx.key == expectedKey` | COVERED (line 98) |
| `base_nonce` | `ctx.baseNonce == expectedBaseNonce` | COVERED (line 99) |
| `exporter_secret` | `ctx.exporterSecret == expectedExporterSecret` | COVERED (line 100) |
| `shared_secret` intermediate | manual DH + LabeledExtract + LabeledExpand | COVERED (lines 138–174) |
| Ciphertext seq=0 | `ctx.seal(aad, pt) == expectedCiphertext` | COVERED (lines 104–119) |
| Ciphertext seq=1 (nonce increment) | second `ctx.seal(aad1, pt)` | COVERED (lines 177–200) |
| Exporter with context `\x00` | `ctx.export(exportContext, 32)` | COVERED (lines 121–136) |
| Exporter with empty context | `ctx.export(Uint8List(0), 32)` | COVERED (lines 202–220) |
| Exporter with `"TestContext"` | `ctx.export(exportCtx, 32)` | COVERED (lines 222–241) |

`testKeyPair` injection hook is confirmed active across all six Appendix A.1 group tests (lines 85–88, 105–108, 122–125, 145–148, 185–188, 209–212). The hook correctly sets the ephemeral key pair so assertions on `enc` and ciphertext are deterministic.

**What is absent in `test/hpke_test.dart`:**

- Sequence-number overflow (known gap F-6 from Phase 1)
- `export()` called with a length of 0 or a length exceeding the HKDF-SHA256 hash output maximum (255 * 32 = 8160 bytes)
- Invalid / off-curve public key passed to `setupBaseS`
- `seal()` called twice on the same context to verify the sequence counter increments correctly beyond seq=1 (seq=2, seq=3, …)
- Behavior when `info` is empty
- Behavior when the AEAD tag is stripped (malformed ciphertext passed to any seal/open path — though the sender-only design means there is no `open` path; this gap is in `ohttp_test.dart`)

---

### `test/bhttp_test.dart` — 231 lines

**Test groups:**

| Group | Tests | What it covers |
|---|---|---|
| `varint encoding` | 3 | `encodeVarint` for 1-byte (0–63), 2-byte (64–16383), 4-byte (16384–1073741823) |
| `varint decoding` | 4 | `decodeVarint` for 1-, 2-, 4-byte boundary values; decode with offset |
| `varint roundtrip` | 1 | Encode then decode for values `[0, 1, 63, 64, 100, 255, 16383, 16384, 100000]` |
| `serializeRequest` | 3 | Simple GET, POST with headers and body, header lowercasing |
| `parseResponse` | 3 | Simple 200, with headers, rejects non-response framing |
| `serialize/parse roundtrip` | 3 | Non-empty GET request, 404 response, 204 empty body |

**Varint boundary coverage:**

| Boundary value | Covered? |
|---|---|
| 0 | YES (encoding line 10, decoding line 34, roundtrip line 62) |
| 1 | YES (encoding line 10, roundtrip line 62) |
| 63 (max 1-byte) | YES (encoding line 12, decoding line 38, roundtrip line 62) |
| 64 (min 2-byte) | YES (encoding line 15–17, decoding line 41–43, roundtrip line 62) |
| 16383 (max 2-byte) | YES (encoding line 19–20, decoding line 45–46, roundtrip line 62) |
| 16384 (min 4-byte) | YES (encoding line 23–25, decoding line 49–51, roundtrip line 62) |
| 1073741823 (max 4-byte) | YES (encoding line 27–28) |
| 100000 (4-byte range) | YES (roundtrip line 62) |
| 2^30 (first 8-byte-requiring value: 1073741824) | NO — not covered |
| 2^62−1 (max 8-byte varint) | NO — not covered |
| 8-byte varint anywhere | NO — no 8-byte varint encode/decode test exists |

**What is absent in `test/bhttp_test.dart`:**

- 8-byte varint encoding and decoding (2^30 through 2^62−1) — the 8-byte case is implemented in `decodeVarint` (lines 57–63 of bhttp.dart) but has zero test coverage
- Truncated body test: build a response where `contentLen` exceeds remaining buffer, confirm `RangeError` is raised (known gap R-6 from Phase 2)
- Truncated header section: `headersLen` exceeds remaining buffer
- Truncated varint: buffer that claims a 4-byte varint but has only 2 bytes
- Unknown framing indicator: `parseResponse` with framing byte `0x02` or `0x03`
- `statusCode` out of HTTP range: varint of 0, 99, 600, or 65535 passed as status in an otherwise valid response
- Header with no terminating zero-length field (missing trailer sentinel)
- Oversized header section: `headersLen` claims 1 GiB — confirm either `RangeError` or `FormatException` (known gap F2.4a from Phase 2)
- Many tiny headers exhausting the count: no max-header-count guard exercised
- `serializeRequest` with body at or over a sane size cap — no size guard exercised

---

### `test/ohttp_test.dart` — 164 lines

**Test groups:**

| Group | Tests | What it covers |
|---|---|---|
| `OhttpKeyConfig.parse` | 4 | Valid 41-byte config; too-short data; unsupported KEM; short symmetric section |
| `OhttpKeyConfig.validate` | 4 | Accepted cipher suite; unsupported KEM; unsupported KDF; unsupported AEAD |
| `ohttpEncapsulate` | 1 | Structural check (header bytes, enc length, exportedSecret length, ciphertext length) |
| `ohttpDecapsulate` | 2 | Too-short response (32 bytes total); response with only nonce and no valid ciphertext (17 bytes) |

**Detailed mapping of existing `ohttpDecapsulate` tests:**

- `ohttpDecapsulate(Uint8List(32), Uint8List(16), Uint8List(16))` at line 153: `encResponse` is 16 bytes — less than `_responseNonceLen (16) + _tagLen (16) = 32` minimum; `FormatException` expected.
- `ohttpDecapsulate(Uint8List(32), Uint8List(16), Uint8List(17))` at line 158: `encResponse` is 17 bytes — nonce extractable (16 bytes) but ciphertext is only 1 byte (less than AES-GCM tag length 16); `FormatException` expected.

These tests confirm the format-guard paths in `ohttp.dart:211–213` but do not exercise the AEAD decryption path at all.

**What is absent in `test/ohttp_test.dart`:**

- `symLen > 4` (extra KDF+AEAD pairs in KeyConfig): no test with `sym_len = 8` — known gap from Phase 3 (O-2, TASK-O7 item 3)
- Round-trip test: `ohttpEncapsulate` → gateway-simulated `ohttpDecapsulate` with matching key material to confirm plaintext recovery — known gap from Phase 3 (TASK-O7 item 1)
- AEAD authentication failure: valid-length `encResponse` with a single bit-flipped in the ciphertext — known gap from Phase 3 (TASK-O7 item 2; O-9); confirmed absent because the only two `ohttpDecapsulate` tests pass `Uint8List(16)` or `Uint8List(17)` (both are rejected by the format guard before reaching the AES-GCM decrypt call)
- Truncated ciphertext: `encResponse` length exactly `_responseNonceLen + tagLen - 1` (31 bytes) — known gap from Phase 3 (TASK-O7 item 4)
- `OhttpKeyConfig.parse` with `symLen = 0`: a zero-length symmetric section (currently guarded at `symLen < 4` — should throw `FormatException`)
- No `OhttpClient` tests: no test for `send()` or `sendDirect()` at any layer; no test for KeyConfig GET failure, gateway POST failure, timeout, scheme enforcement, or response size cap
- No test that `targetAuthority` is embedded verbatim into the BHTTP request
- No test confirming that `SecretBoxAuthenticationError` (or a wrapped version) is raised on auth failure rather than a silent wrong-plaintext result

**Live-gateway integration tests:** Absent. Confirmed by CLAUDE.md ("There are no integration tests against a live gateway") and by inspection of the `test/` directory (three files only).

**Fuzz / property-based tests:** Absent across all three files. No test uses random or semi-random inputs, generator-based property checking, or corpus-based fuzzing.

---

## Cross-Phase Gap Confirmation Table

All named gaps from Phases 1–4 are listed below. Status is confirmed by reading the three test files in full.

| Gap ID | Phase | Named gap | Still absent? | Evidence |
|---|---|---|---|---|
| F-6 | 1 | Sequence-number overflow (`_seq` wrap-around → `StateError`) is untested | YES | `hpke_test.dart` has no test that calls `seal()` enough times to overflow `_seq`; the overflow guard at `hpke.dart:_seq` is never exercised |
| F-7 | 1 | No integration test against a reference Go gateway to confirm wire-format interop (empty-AAD deviation could fail silently) | YES | No integration test directory; CLAUDE.md confirms absence |
| R-6 | 2 | No negative-case tests for truncated body, oversized `symLen`, unknown framing indicator, or malformed header | YES | `bhttp_test.dart` has no test for any of these four scenarios; existing `parseResponse` group has only three positive/near-negative tests (lines 125–186) |
| F2.3 | 2 | `statusCode` not range-checked (100–599); accepts 0, 999, 2^62−1 | YES | No `statusCode` out-of-range test in `bhttp_test.dart` |
| F2.4a | 2 | `headersLen` accepted without bound check vs. remaining buffer; no max-section-byte cap | YES | No oversized-`headersLen` test in `bhttp_test.dart` |
| F2.4b | 2 | No max-header-count cap; `headers` list grows unboundedly | YES | No high-header-count test |
| F2.4c | 2 | Single header name/value `utf8.decode(sublist(...))` allocates declared size verbatim | YES | No single-oversized-header test |
| F2.5a | 2 | `data.sublist(offset, offset + contentLen)` raises `RangeError` (not `FormatException`) on truncated body | YES | No truncated-body test in `bhttp_test.dart` |
| F2.5b | 2 | Header-loop `sublist` calls raise `RangeError` on truncation | YES | No truncated-header test |
| F2.5c | 2 | Empty input → `decodeVarint` on empty buffer raises `RangeError` before framing check | YES | No empty-buffer test in `bhttp_test.dart` |
| F2.6a | 2 | Varint decode lacks pre-index bound checks; `RangeError` instead of `FormatException` on short buffer | YES | No short-buffer varint test |
| F2.6b | 2 | 8-byte varint silently overflows on dart2js / dart2wasm | YES | 8-byte varint has zero test coverage in `bhttp_test.dart` |
| F2.7 | 2 | `serializeRequest` has no body-size or header-size guard | YES | No oversized-body test in `bhttp_test.dart` |
| O-2 / TASK-O1 | 3 | `symLen > 4`: extra KDF+AEAD pairs in KeyConfig silently ignored | YES | `ohttp_test.dart` has no `symLen = 8` test case (all four `OhttpKeyConfig.parse` tests use `sym_len = 4`, line 14 / 57 / etc.) |
| TASK-O7 item 1 | 3 | Round-trip: `ohttpEncapsulate` → `ohttpDecapsulate` with matching key material | YES | No round-trip test in `ohttp_test.dart`; `ohttpEncapsulate` group (line 111–147) only checks structure, never decrypts |
| TASK-O7 item 2 | 3 | Authentication failure: bit-flipped ciphertext → `OhttpAuthenticationException` | YES | Both `ohttpDecapsulate` tests (lines 149–163) are format-guard tests that never reach AES-GCM decrypt |
| TASK-O7 item 3 | 3 | Multi-suite KeyConfig: `symLen = 8`, supported first pair | YES | Absent (confirmed above) |
| TASK-O7 item 4 | 3 | Truncated ciphertext: `encResponse` between `_responseNonceLen + 1` and `_responseNonceLen + tagLen - 1` | YES | Partially covered: `Uint8List(17)` is 17 bytes (nonce=16 + 1 byte), which IS in this range and does throw `FormatException`; however, the gap remains for the exact boundary value `_responseNonceLen + tagLen - 1 = 31` bytes, and for any value between 17 and 31 |
| O-4 / TASK-O2 | 3 | `parse()` throws `FormatException` vs `validate()` throws `UnsupportedError` for unsupported KEM — inconsistent | PARTIAL | `ohttp_test.dart:28–33` tests `parse()` with unsupported KEM (`0x0010`) → `FormatException`; `ohttp_test.dart:77–85` tests `validate()` with unsupported KEM (`0x0010`) → `UnsupportedError`; both are covered, but no test asserts that callers can use a single catch clause — the API inconsistency is documented, not tested |
| O-9 / TASK-O5 | 3 | `SecretBoxAuthenticationError` propagates unwrapped; no test for auth-failure catch contract | YES | Confirmed absent — same as TASK-O7 item 2 |
| C-1 / TASK-C1 | 4 | `OhttpGatewayConfig.gatewayBaseUrl` accepts plain `http://` silently | YES | No `OhttpClient` or `OhttpGatewayConfig` tests in any test file |
| C-2 / TASK-C2 | 4 | String-concatenated paths (double-slash, path-traversal) not normalized | YES | No `OhttpGatewayConfig` URL construction test |
| C-3 / TASK-C3 | 4 | `targetAuthority` embedded verbatim in BHTTP request | YES | No test for authority embedding |
| C-4 / TASK-C4 | 4 | `sendDirect()` co-located in config, no bypass warning | YES | No `sendDirect()` test |
| C-5 / TASK-C5 | 4 | No KeyConfig caching; fresh GET on every `send()` | YES | No `OhttpClient.send()` test |
| C-6 / TASK-C6 | 4 | `http.Client.post()` has no timeout, retry, or cancellation | YES | No `OhttpClient` test |
| C-7 / TASK-C7 | 4 | Non-200 responses throw generic `Exception` | YES | No HTTP error-status test |
| C-8 / TASK-C8 | 4 | Network errors (DNS, connection reset) surface as untyped exceptions | YES | No network-error test |
| C-9 / TASK-C9 | 4 | No response body/headers size cap before `bhttp.parseResponse` | YES | No size-cap test at client layer |
| C-10 / TASK-C10 | 4 | `OhttpHeader.name` not lowercased on response side | YES | No header-case-normalization test for responses |
| C-11 / TASK-C11 | 4 | `SecretBoxAuthenticationError` propagates unwrapped to `OhttpClient.send()` caller (cross-layer confirmation; fix site is `ohttp.dart:219` per TASK-O5) | YES | No test exercises the AEAD auth-failure path through the client layer; confirmed by TASK-T9 gap |
| C-12 / TASK-C12 | 4 | `onLog` at `ohttp_client.dart:77` logs gateway base URL at every `send()` call — privacy/observability gap | YES | No test verifies or constrains the `onLog` callback output; no log-level guard test |

---

## New Gaps Identified (Not Named in Prior Phases)

The following gaps were identified by reading the test files in full and are not explicitly named in any prior phase tasks.md:

### NG-1 — 8-byte varint has zero test coverage

**File:** `test/bhttp_test.dart` — coverage stops at 4-byte varint (line 29–30 for max 4-byte value 1073741823).
**Missing scenario:** `encodeVarint(1073741824)` (first value requiring 8 bytes) and `decodeVarint` of the corresponding encoded bytes; `encodeVarint(9007199254740991)` (max JS-safe-int); `decodeVarint` of a full 8-byte varint.
**Proposed group name:** `varint 8-byte encoding and decoding`
**Severity:** HIGH — the 8-byte branch in `decodeVarint` (bhttp.dart:57–63) is untested and the JS integer overflow bug (F2.6b) cannot be caught without tests at these values.

### NG-2 — `export()` with out-of-bounds length has no test

**File:** `test/hpke_test.dart` — existing export tests (lines 121–241) use lengths 32 only. The `HKDF utilities` group (lines 261–295) tests lengths 16 and 32 only.
**Missing scenario:** `ctx.export(context, 0)` (zero-length export); `ctx.export(context, 8161)` (above the HKDF-SHA256 maximum of 255 * 32 = 8160 bytes for the underlying `package:cryptography` HMAC-SHA256 expand).
**Proposed group name:** `HpkeSenderContext export edge cases`
**Severity:** HIGH — zero-length or oversized export lengths could trigger undefined behavior or a confusing internal exception in `package:cryptography`; the library does not guard against these inputs.

### NG-3 — Invalid / off-curve public key passed to `setupBaseS` has no test

**File:** `test/hpke_test.dart` — all six Appendix A.1 group tests use valid `pkRm` (line 61). No test passes a zero public key, an all-ones key, or a point not on Curve25519.
**Missing scenario:** `HpkeSender.setupBaseS(Uint8List(32), info)` with `pkRm = Uint8List(32)` (all zeros) — confirm that `package:cryptography` throws and that the exception type is sane.
**Proposed group name:** `setupBaseS invalid public key`
**Severity:** HIGH — a wallet receiving a zero or low-order public key from a compromised gateway could silently derive a predictable shared secret; the test would verify that `package:cryptography` rejects such inputs.

### NG-4 — `seal()` called on an already-used context has no test beyond seq=1

**File:** `test/hpke_test.dart` — the seq=1 test (lines 177–200) is the only multi-seal test; it calls `seal()` exactly twice. No test calls `seal()` a large number of times or confirms the `StateError` thrown on overflow.
**Missing scenario:** call `seal()` until `_seq` overflows the 12-byte nonce counter limit (2^96 − 1 operations); or, for a practical test, confirm `StateError` is thrown when `_seq` is manually set to `2^96 − 1` via a test seam (if exposed) and then `seal()` is called once more. Alternatively, a test that calls `seal()` a meaningful number of times (e.g., 1000) to confirm the counter increments without panicking.
**Proposed group name:** `HpkeSenderContext sequence number overflow`
**Severity:** HIGH — this is the F-6 gap confirmed here; the `StateError` path at `hpke.dart` is unreachable by the current test suite.

### NG-5 — `parseResponse` with `statusCode = 0` or `statusCode = 999` has no test

**File:** `test/bhttp_test.dart` — existing `parseResponse` tests use status codes 200 (line 129), 404 (line 206), and 204 (line 219). No test exercises an out-of-range status code.
**Missing scenario:** build a valid BHTTP response frame with `statusCode = 0` (below valid HTTP range) and `statusCode = 999` (above valid range 599); confirm that `parseResponse` either throws `FormatException` or returns the invalid code without a guard (documenting the current no-guard behavior).
**Proposed group name:** `parseResponse statusCode out-of-range`
**Severity:** IMPROVEMENT — per F2.3 (Phase 2) this is a silent acceptance of invalid data; no memory exhaustion, but downstream wallet code that branches on status ranges could misbehave.

### NG-6 — `parseResponse` with empty buffer has no test

**File:** `test/bhttp_test.dart` — no test passes `Uint8List(0)` to `parseResponse`.
**Missing scenario:** `parseResponse(Uint8List(0))` — confirm that a `RangeError` (not `FormatException`) is raised by `decodeVarint` on the empty buffer before the framing-indicator check runs.
**Proposed group name:** `parseResponse empty buffer`
**Severity:** HIGH — callers wrapping `parseResponse` in `on FormatException` will not catch the `RangeError`; the exception type mismatch is a documented correctness risk (F2.5c Phase 2) that has no regression test.

### NG-7 — `ohttpEncapsulate` round-trip with simulated gateway decryption

**File:** `test/ohttp_test.dart` — the `ohttpEncapsulate` group (lines 111–147) verifies output structure but never decrypts the ciphertext.
**Missing scenario:** generate a real X25519 key pair (receiver); call `ohttpEncapsulate` with the receiver's public key; then manually perform the HPKE receiver-side Decap + AES-GCM open to recover the plaintext and confirm it equals the input. This does not require `ohttpDecapsulate` (which is response-side); it is a sender-only round-trip using the raw `HpkeSender` and `package:cryptography` AES-GCM primitives.
**Proposed group name:** `ohttpEncapsulate sender round-trip`
**Severity:** HIGH — without this test, it is impossible to confirm that the encapsulated ciphertext produced by the library is actually readable by a conformant receiver, including the specific info-string and empty-AAD choices.

### NG-8 — No test for `OhttpClient.send()` error paths at all

**File:** `test/ohttp_test.dart` (or a new `ohttp_client_test.dart`) — there is no test for any method on `OhttpClient`.
**Missing scenario:** use `MockClient` (from `package:http/testing.dart`) to simulate: (a) KeyConfig GET returning HTTP 503 → confirm typed or untyped exception; (b) gateway POST returning HTTP 400 → confirm typed or untyped exception; (c) `http://` scheme gatewayBaseUrl → confirm no scheme rejection; (d) `sendDirect()` with a mock direct URL → confirm response is parsed correctly.
**Proposed group name:** `OhttpClient error paths`
**Severity:** HIGH — the client layer (Layer 4) is entirely without tests; all four Findings C-1 through C-10 from Phase 4 are undetectable by the current test suite.

### NG-9 — No fuzz / property-based tests across any test file

**File:** all three test files — confirmed absent.
**Missing scenario class:** property tests for `encodeVarint` / `decodeVarint` round-trip over the full unsigned 62-bit value space; property tests for `serializeRequest` / `parseResponse` round-trip over generated request parameters; fuzz tests for `OhttpKeyConfig.parse` over arbitrary byte sequences.
**Proposed group name:** (framework-agnostic; group names TBD by implementing engineer)
**Severity:** HIGH — without property/fuzz coverage, adversarial inputs to `decodeVarint`, `parseResponse`, and `OhttpKeyConfig.parse` are identified only by manual case enumeration. The three HIGH BHTTP parser findings (F2.4a, F2.5a, F2.6a) are all reachable by property tests.

### NG-10 — No live-gateway integration test

**File:** none — no integration test directory exists.
**Missing scenario:** send a real OHTTP request to a public or self-hosted OHTTP gateway (e.g., the Cloudflare research gateway or a local Go reference implementation), confirm the response is decoded correctly, and confirm the empty-AAD choice produces a successful round-trip.
**Proposed group name:** `OhttpClient live gateway integration`
**Severity:** HIGH — per PRD §Constraints assumption 6, public OHTTP test gateways may be available. The empty-AAD deviation (Finding O-6, Phase 3) could fail silently against a strictly RFC-compliant gateway with no existing test to detect it.

---

## Resolved Questions

**Q1 (HPKE vector scope):** Stay strictly within the "Base Mode + one seal" boundary set in the PRD — do not expand to other A.1 sub-vectors.
- Resolution applied: The coverage table above documents all sub-vectors exercised within Base Mode (including export tests that are part of the same Appendix A.1 section); gaps are scoped to the OHTTP-relevant sender path. The gap analysis for `export()` edge cases (NG-2) applies to the `HpkeSenderContext.export` method itself, which is part of the Base Mode sender, not an out-of-scope receiver sub-test.

**Q2 (Integration test severity):** Use PRD classification as-is — missing live-gateway integration tests are HIGH.
- Resolution applied: NG-10 is classified HIGH.

**Q3 (Property-based test framework):** Keep entries completely framework-agnostic; no informational notes about candidate frameworks.
- Resolution applied: NG-9 and all draft task entries for fuzz/property tests contain no framework names or version references.

**Q4 (Cross-phase gap scope):** Check ALL named gaps from phases 1–4 — read all prior phase tasks.md files and pick up any additional named gaps beyond the four explicitly listed in the PRD.
- Resolution applied: The cross-phase confirmation table above covers all 32 named gaps found across phases 1–4 (including C-11 and C-12 from Phase 4).

**Q5 (Draft task format):** research.md only — keep phase-5/tasks.md untouched; all findings go in research.md.
- Resolution applied: No changes were made to `phase-5/tasks.md`. All draft task entries are in this document.

**Q6:** No additional constraints.

---

## Draft Task Entries

Each entry is independently actionable for an engineer in a future implementation ticket.

### TASK-T1 — Add `_seq` overflow test to `hpke_test.dart`

**File:** `test/hpke_test.dart`
**Missing scenario:** sequence-number overflow → `StateError` (or typed equivalent)
**Coverage stops at:** line 200 (seq=1 seal test — last seal call in the file)
**Proposed group name:** `HpkeSenderContext sequence number overflow`
**Severity:** HIGH
**Cross-phase:** Confirms F-6 (Phase 1)

### TASK-T2 — Add invalid public key test to `hpke_test.dart`

**File:** `test/hpke_test.dart`
**Missing scenario:** `setupBaseS` with zero or low-order public key → exception from `package:cryptography`
**Coverage stops at:** line 242 (last `setupBaseS` call uses valid `pkRm`)
**Proposed group name:** `setupBaseS invalid public key`
**Severity:** HIGH
**Cross-phase:** New gap (NG-3)

### TASK-T3 — Add `export()` edge-case tests to `hpke_test.dart`

**File:** `test/hpke_test.dart`
**Missing scenario:** `export()` with length 0; `export()` with length exceeding HKDF-SHA256 output limit
**Coverage stops at:** line 295 (last `hkdfExpand` test uses length 32)
**Proposed group name:** `HpkeSenderContext export edge cases`
**Severity:** HIGH
**Cross-phase:** New gap (NG-2)

### TASK-T4 — Add 8-byte varint tests to `bhttp_test.dart`

**File:** `test/bhttp_test.dart`
**Missing scenario:** `encodeVarint` and `decodeVarint` for values in the 8-byte range (2^30 through 2^62−1); confirm JS-target overflow behavior
**Coverage stops at:** line 30 (max 4-byte value 1073741823 — last covered varint boundary)
**Proposed group name:** `varint 8-byte encoding and decoding`
**Severity:** HIGH
**Cross-phase:** Confirms F2.6b (Phase 2); new gap NG-1

### TASK-T5 — Add truncated-input negative tests to `bhttp_test.dart`

**File:** `test/bhttp_test.dart`
**Missing scenario:** truncated body (`contentLen > remaining bytes`); truncated header section (`headersLen > remaining bytes`); truncated varint (4-byte varint with only 2 bytes in buffer); empty buffer passed to `parseResponse`
**Coverage stops at:** line 231 (last test in the file)
**Proposed group name:** `parseResponse truncated input`
**Severity:** HIGH
**Cross-phase:** Confirms R-6 (Phase 2), F2.5a, F2.5b, F2.5c, F2.6a, NG-6

### TASK-T6 — Add unknown framing indicator test to `bhttp_test.dart`

**File:** `test/bhttp_test.dart`
**Missing scenario:** `parseResponse` with framing byte `0x02` (indeterminate-length request) and `0x03` (indeterminate-length response) — confirm `FormatException`; `parseResponse` with framing byte `0x00` (request indicator) — confirm `FormatException`
**Coverage stops at:** line 186 (the existing "rejects non-response framing" test covers only `0x00`; `0x02` and `0x03` are not covered)
**Proposed group name:** `parseResponse unknown framing indicator`
**Severity:** IMPROVEMENT
**Cross-phase:** Extends R-6 (Phase 2)

### TASK-T7 — Add `statusCode` out-of-range tests to `bhttp_test.dart`

**File:** `test/bhttp_test.dart`
**Missing scenario:** response frame with `statusCode = 0`; `statusCode = 99`; `statusCode = 600`; `statusCode = 65535` — confirm current no-guard behavior (or a `FormatException` if a guard is added)
**Coverage stops at:** line 231 (last test uses statusCode 204)
**Proposed group name:** `parseResponse statusCode out of range`
**Severity:** IMPROVEMENT
**Cross-phase:** Confirms F2.3 (Phase 2), NG-5

### TASK-T8 — Add `symLen > 4` KeyConfig test to `ohttp_test.dart`

**File:** `test/ohttp_test.dart`
**Missing scenario:** `OhttpKeyConfig.parse` with `symLen = 8` and a supported first KDF+AEAD pair followed by a second arbitrary pair — confirm parse succeeds and documents that the second pair is silently dropped (regression guard for TASK-O1)
**Coverage stops at:** line 63 (last `OhttpKeyConfig.parse` test)
**Proposed group name:** `OhttpKeyConfig.parse multi-suite KeyConfig`
**Severity:** HIGH
**Cross-phase:** Confirms O-2 / TASK-O7 item 3 (Phase 3)

### TASK-T9 — Add AEAD authentication failure test to `ohttp_test.dart`

**File:** `test/ohttp_test.dart`
**Missing scenario:** valid-length `encResponse` with a single bit flipped in the ciphertext portion — confirm `SecretBoxAuthenticationError` (or `OhttpAuthenticationException` after TASK-O5 is implemented) is thrown; confirm it is not swallowed silently
**Coverage stops at:** line 163 (last `ohttpDecapsulate` test never reaches AES-GCM decrypt)
**Proposed group name:** `ohttpDecapsulate AEAD authentication failure`
**Severity:** HIGH
**Cross-phase:** Confirms O-9 / TASK-O7 item 2 (Phase 3), C-11 (Phase 4)

### TASK-T10 — Add round-trip test to `ohttp_test.dart`

**File:** `test/ohttp_test.dart`
**Missing scenario:** generate a real X25519 receiver key pair; call `ohttpEncapsulate`; use `package:cryptography` AES-GCM open (receiver side) to confirm the ciphertext decrypts to the original plaintext; also call `ohttpDecapsulate` with a simulated gateway response encrypted under the exported secret to confirm full decapsulation
**Coverage stops at:** line 147 (last `ohttpEncapsulate` test only checks structure; never decrypts)
**Proposed group name:** `ohttpEncapsulate / ohttpDecapsulate round-trip`
**Severity:** HIGH
**Cross-phase:** Confirms TASK-O7 item 1 (Phase 3), NG-7

### TASK-T11 — Add `OhttpClient` error-path tests

**File:** new `test/ohttp_client_test.dart` (or appended to `test/ohttp_test.dart`)
**Missing scenario:** KeyConfig GET returning HTTP 503 → exception; gateway POST returning HTTP 400 → exception; `http://` scheme gatewayBaseUrl → no scheme rejection; `sendDirect()` → correct response parsing with mock HTTP client
**Coverage stops at:** N/A — `OhttpClient` is entirely untested
**Proposed group name:** `OhttpClient error paths`
**Severity:** HIGH
**Cross-phase:** Confirms C-1 through C-10 (Phase 4), NG-8

### TASK-T12 — Add fuzz / property-based tests across the test suite

**File:** `test/bhttp_test.dart`, `test/ohttp_test.dart` (or a new `test/fuzz/` directory)
**Missing scenario class:** property tests for `encodeVarint` / `decodeVarint` round-trip; property tests for `serializeRequest` / `parseResponse` round-trip; fuzz tests for `OhttpKeyConfig.parse` over arbitrary byte sequences
**Coverage stops at:** N/A — no property or fuzz tests exist anywhere in the test suite
**Proposed group name:** framework-agnostic; group names left to the implementing engineer
**Severity:** HIGH
**Cross-phase:** Confirms NG-9

### TASK-T13 — Add live-gateway integration test

**File:** new `test/integration/ohttp_gateway_test.dart`
**Missing scenario:** send a real OHTTP request to a public or self-hosted gateway; confirm the response is decoded correctly; confirm empty-AAD choice produces a successful round-trip against a conformant gateway
**Coverage stops at:** N/A — no integration test directory exists
**Proposed group name:** `OhttpClient live gateway integration`
**Severity:** HIGH
**Cross-phase:** Confirms F-7 (Phase 1), NG-10
