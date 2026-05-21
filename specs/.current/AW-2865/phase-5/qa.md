# Phase 5 QA Report — Test Suite Coverage Gap Audit (AW-2865)

**Ticket:** AW-2865 — ohttp_dart investigation
**Phase:** 5 of 7 (test suite coverage audit)
**QA date:** 2026-05-21
**Audited artifacts:**
- `specs/.current/AW-2865/phase-5/prd.md` (Status: PRD_READY)
- `specs/.current/AW-2865/phase-5/plan.md` (Status: PLAN_APPROVED)
- `specs/.current/AW-2865/phase-5/research.md` (Status: RESEARCH_COMPLETE)
- `specs/.current/AW-2865/phase-5/tasks.md` (11 of 11 tasks checked)
- `specs/.current/AW-2865/review.md` (Phase 5 section present)

---

## Phase Scope

Phase 5 is a **read-only audit** of the three test files in `ohttp_dart`:
- `test/hpke_test.dart` (296 lines) — RFC 9180 Appendix A.1 vector suite for `hpke.dart`
- `test/bhttp_test.dart` (231 lines) — QUIC varint + framing tests for `bhttp.dart`
- `test/ohttp_test.dart` (164 lines) — KeyConfig parse, encapsulate, decapsulate tests for `ohttp.dart`

The deliverable is audit findings documented in `specs/.current/AW-2865/phase-5/research.md` and a task checklist in `specs/.current/AW-2865/phase-5/tasks.md`. No file under `lib/`, `test/`, `pubspec.yaml`, or `example/` was modified.

---

## PRD Status Check

| Artifact | Required status | Actual status | Pass? |
|---|---|---|---|
| `specs/.current/AW-2865/phase-5/prd.md` | PRD_READY | PRD_READY (line 3) | YES |
| `specs/.current/AW-2865/phase-5/plan.md` | PLAN_APPROVED | PLAN_APPROVED (line 5) | YES |
| `specs/.current/AW-2865/phase-5/research.md` | RESEARCH_COMPLETE | RESEARCH_COMPLETE (line 3) | YES |
| `specs/.current/AW-2865/review.md` | Phase 5 section must exist | Present at line 720 (Phase 5 Review section) | YES |

---

## Task Checklist Verification (tasks 5.1–5.11)

All 11 tasks in `specs/.current/AW-2865/phase-5/tasks.md` are checked `[x]`. Each is substantiated by findings in the same file.

| Task | Description | Checked? | Substantiated? |
|---|---|---|---|
| 5.1 | Read `test/hpke_test.dart` | YES | Research §5.1–5.3 cites actual line ranges (lines 82–295) |
| 5.2 | Confirm `testKeyPair` injection and RFC 9180 A.1 sub-vector coverage | YES | Research lists 7 covered vectors with exact line ranges |
| 5.3 | Identify missing HPKE test cases | YES | T-H1 (overflow), T-H2 (export edge), T-H3 (invalid pubkey) documented with line anchors |
| 5.4 | Read `test/bhttp_test.dart` | YES | Research §5.4–5.6 cites test group names and line ranges |
| 5.5 | Confirm varint 1/2/4/8-byte boundary coverage | YES | Boundary table with explicit YES/NO per value — 8-byte range confirmed absent |
| 5.6 | Identify missing BHTTP test cases | YES | T-B1 through T-B5 documented with line anchors |
| 5.7 | Read `test/ohttp_test.dart` | YES | Research §5.7–5.11 maps all 4 test groups to code paths |
| 5.8 | Identify missing OHTTP test cases | YES | T-O1 through T-O6 documented with line anchors |
| 5.9 | Note absence of live-gateway integration tests | YES | Confirmed in research and in CLAUDE.md cross-reference |
| 5.10 | Note absence of fuzz/property-based tests | YES | Confirmed absent across all three files |
| 5.11 | Record each gap as a draft task entry with file, scenario, line range, proposed group name, and severity | YES | 15 entries in `tasks.md` summary table; 13 canonical entries (TASK-T1..TASK-T13) in `research.md` |

---

## Positive Scenarios

### PS-1 — RFC 9180 Appendix A.1 vector coverage in `hpke_test.dart`

`test/hpke_test.dart` exercises all RFC 9180 Appendix A.1 Base Mode sub-vectors that are directly relevant to the OHTTP sender flow:
- `enc`, `key`, `base_nonce`, `exporter_secret` assertions confirmed at lines 97–100.
- Ciphertext for seq=0 confirmed at lines 104–119.
- Ciphertext for seq=1 (nonce-increment) confirmed at lines 177–200.
- Exporter with three distinct contexts (byte `\x00`, empty, `"TestContext"`) confirmed at lines 121–136, 202–220, 222–241.
- Shared-secret intermediate confirmed at lines 138–175.

The `testKeyPair` injection hook is active in all six vector tests (lines 85–88, 105–108, 122–125, 145–148, 185–188, 209–212). The hook guarantees deterministic assertion on `enc` and ciphertext. Its continued function is confirmed — it was not removed or bypassed by any Phase 5 action.

RFC 5869 HKDF Test Case 1 (lines 244–258) and three HKDF utility tests (lines 261–295) further verify the `hkdfExtract`/`hkdfExpand` primitives used by the OHTTP response decapsulation path.

**Verdict:** The covered portion of `hpke_test.dart` is correctly structured, uses the injection hook as intended, and produces accurate RFC vector assertions. No positive-path test is broken.

### PS-2 — QUIC varint boundary coverage in `bhttp_test.dart`

The following boundary values are positively confirmed covered:
- 1-byte range: 0, 1, 63 (encoding, decoding, roundtrip).
- 2-byte range: 64, 16383 (encoding, decoding, roundtrip).
- 4-byte range: 16384, 1073741823 (encoding, decoding, roundtrip).
- Mid-range values: 100, 255, 100000 (roundtrip group at line 62).

`serializeRequest` positive-path tests cover simple GET, POST with headers and body, and header lowercasing (3 tests). `parseResponse` positive-path tests cover simple 200, response with headers, and a 204 empty-body case. Three serialize/parse roundtrip tests close the loop.

**Verdict:** The covered positive-path portion of `bhttp_test.dart` is complete and correct for the expected happy-path scenarios.

### PS-3 — OHTTP `OhttpKeyConfig.parse` and structural `ohttpEncapsulate` coverage

`test/ohttp_test.dart` verifies:
- Valid 41-byte KeyConfig parses correctly (lines 9–26).
- Four error paths in `OhttpKeyConfig.parse` are exercised: too-short data (line 28), unsupported KEM (line 35), short symmetric section (line 50).
- All three `OhttpKeyConfig.validate` rejection cases tested: unsupported KEM (line 77), unsupported KDF, unsupported AEAD (lines 65–108).
- `ohttpEncapsulate` structural check asserts correct header bytes, `enc` length (32), `exportedSecret` length (16), and ciphertext length (lines 111–146).
- Two `ohttpDecapsulate` format-guard rejection paths are exercised: `encResponse` of 16 bytes (line 152) and 17 bytes (line 159).

**Verdict:** Existing positive-path and near-negative-path coverage in `ohttp_test.dart` is accurate and correctly implemented.

---

## Negative and Edge Cases

### NC-1 — `test/hpke_test.dart` missing cases (HIGH severity)

**T-H1 — Sequence-number overflow:** `_seq` overflow guard at `hpke.dart:275–277` throws `StateError`; no test calls `seal()` enough times to reach this path. The guard is exercised in production but never regression-tested. **CONFIRMED ABSENT** (coverage stops at line 200, seq=1 is the last seal call).

**T-H2 — `export()` with boundary lengths:** No test calls `ctx.export()` with length 0 or a length exceeding the 255 × 32 = 8160 byte HKDF-SHA256 maximum. Behavior under these inputs is undefined from a test perspective. **CONFIRMED ABSENT** (coverage stops at line 295, all export tests use length 32).

**T-H3 — All-zero or low-order public key in `setupBaseS`:** No test passes an invalid X25519 public key (e.g., `Uint8List(32)`). A wallet receiving a crafted low-order key from a compromised gateway could silently derive a weak shared secret. **CONFIRMED ABSENT.**

### NC-2 — `test/bhttp_test.dart` missing cases (HIGH + IMPROVEMENT)

**T-B1 — 8-byte varint encode/decode:** No test exercises values in the range 2^30 through 2^62−1. The 8-byte decode branch in `bhttp.dart:57–63` is entirely uncovered. The JS integer overflow bug (F2.6b from Phase 2) cannot be caught without tests at these values. **CONFIRMED ABSENT** (coverage stops at max 4-byte value 1073741823).

**T-B2 — Truncated body raises `RangeError` instead of `FormatException`:** `parseResponse` calls `data.sublist(offset, offset + contentLen)` without a bounds check; a truncated buffer raises `RangeError`, not `FormatException`. No negative test documents this behavior. Callers wrapping `parseResponse` in `on FormatException` silently miss this exception class. **CONFIRMED ABSENT** (coverage stops at line 185, last `parseResponse` group test).

**T-B3 — Unknown framing indicator:** Only framing bytes `0x00` and `0x01` are tested. No test passes `0x02`, `0x03`, or `0xFF`. The framing guard in `bhttp.dart:140–146` would throw `FormatException`, but this path is unverified. Severity: IMPROVEMENT.

**T-B4 — Malformed header (zero-length name or truncated value):** The positive-path header tests build well-formed name/value pairs. No test covers a header name varint declaring more bytes than remain, or a missing value field. Severity: IMPROVEMENT.

**T-B5 — `serializeRequest` without body-size cap:** No test verifies behavior when a body would exhaust the `BytesBuilder` buffer. No size guard exists; behavior is undocumented. Severity: IMPROVEMENT.

### NC-3 — `test/ohttp_test.dart` missing cases (HIGH + IMPROVEMENT)

**T-O1 — `symLen > 4` silently drops extra KDF+AEAD pairs:** No test builds a KeyConfig with `sym_len = 8` (two pairs). The parser reads only the first pair and discards the rest without error. The current regression-gap means a future code change could silently change this behavior. **CONFIRMED ABSENT** (all four parse tests use `sym_len = 4`).

**T-O2 — `symLen` pointing past buffer end raises `RangeError`:** A `sym_len` field claiming more bytes than remain causes `RangeError`, not `FormatException`. No test covers this path. **CONFIRMED ABSENT.**

**T-O3 — AEAD authentication failure (tampered ciphertext):** Both `ohttpDecapsulate` tests pass `Uint8List(16)` and `Uint8List(17)` — these are rejected by the format guard before reaching AES-GCM decrypt. The actual `aesGcm.decryptList` call and its `SecretBoxAuthenticationError` throw path are entirely untested. This is a critical cryptographic contract path. **CONFIRMED ABSENT** (coverage stops at line 163).

**T-O4 — Full encap/decap positive round-trip:** There is no test that encapsulates a request, constructs a synthetic gateway response using the exported secret, calls `ohttpDecapsulate`, and asserts plaintext recovery. The positive path through `ohttpDecapsulate` AES-GCM decryption has zero test coverage. **CONFIRMED ABSENT.**

**T-O5 — `parse` vs. `validate` exception-type inconsistency coverage:** `parse()` throws `FormatException` for unsupported KEM (line 35); `validate()` throws `UnsupportedError` for the same condition (line 77). Both paths are individually tested, but no test documents that callers cannot catch both with a single clause. Severity: IMPROVEMENT.

**T-O6 — Empty inner payload through `ohttpEncapsulate`:** The one encapsulate test uses a 5-byte `binaryRequest`. No test uses `Uint8List(0)`. Severity: IMPROVEMENT.

### NC-4 — Absent test classes (HIGH)

**T-X1 — No live-gateway or stub-gateway integration test:** No test exercises a full OHTTP request/response cycle against any real or simulated gateway. The empty-AAD deviation (Finding O-6, Phase 3) could fail silently against a strictly RFC-compliant gateway with no existing test to detect it.

**T-X2 — No fuzz/property-based tests across any test file:** No test uses randomized or generator-based inputs. No corpus-based fuzzing exists. Parser attack surfaces (`OhttpKeyConfig.parse`, `bhttp.parseResponse`, `decodeVarint`) are validated only by manual case enumeration.

---

## Automated Tests Coverage

The current test suite contains only the three unit test files evaluated in this phase. No CI integration, no fuzz harness, and no property-based test framework is present.

| Test file | Automated? | Coverage level | Known gaps |
|---|---|---|---|
| `test/hpke_test.dart` | YES — `dart test` | RFC vectors for positive path; seq=0 and seq=1; three export contexts | Overflow path, invalid pubkey, export length edge cases |
| `test/bhttp_test.dart` | YES — `dart test` | Varint 1/2/4-byte; serialize/parse roundtrip; positive response paths | 8-byte varint entirely absent; no truncated-input negative tests |
| `test/ohttp_test.dart` | YES — `dart test` | KeyConfig parse (4 tests); validate (4 tests); encapsulate structural; decapsulate format-guards | No round-trip; no AEAD auth-failure test; no `OhttpClient` tests whatsoever |

---

## Manual Checks Needed

The following gaps require manual verification or test authorship before Phase 5 findings can be fully regression-covered:

| ID | Manual check | Reason automation is insufficient |
|---|---|---|
| T-H1 | Verify `StateError` is raised when `_seq` overflows 12-byte nonce space | No test seam exposes `_seq` directly; requires either a large loop or an internal test hook |
| T-H3 | Verify `package:cryptography` rejects all-zero X25519 public key | Behavior is library-dependent; requires empirical test to document outcome |
| T-O3 | Verify `SecretBoxAuthenticationError` is thrown (not swallowed) on bit-flipped ciphertext | Requires constructing a valid-length but corrupted `encResponse` buffer — not automatable from current test infrastructure |
| T-O4 | Verify encap/decap round-trip produces correct plaintext | Requires receiver-side HPKE key generation and manual HKDF-derived response construction |
| T-X1 | Verify empty-AAD choice is interoperable with a conformant OHTTP gateway | Requires either a live gateway or a self-hosted stub |

---

## Cross-Phase Gap Confirmation

The research document's 30-row cross-phase confirmation table was verified against this QA run. All 30 rows are marked `Still absent`, with the single partial exception of `O-4/TASK-O2` (the `parse/validate` exception inconsistency, where both individual code paths are tested but no combined catch-clause contract test exists).

Key confirmations relevant to QA:

| Prior finding | Phase 5 confirmation | QA status |
|---|---|---|
| F-6 (Phase 1): `_seq` overflow untested | T-H1 — CONFIRMED ABSENT | PASS |
| F-7 (Phase 1): no integration test | T-X1 / TASK-T13 — CONFIRMED ABSENT | PASS |
| R-6 (Phase 2): no negative BHTTP tests | T-B1, T-B2 — CONFIRMED ABSENT | PASS |
| F2.6b (Phase 2): 8-byte varint overflow | T-B1 / NG-1 — CONFIRMED ABSENT | PASS |
| O-2 (Phase 3): `symLen > 4` silent ignore | T-O1 / TASK-T8 — CONFIRMED ABSENT | PASS |
| TASK-O7 items 1–4 (Phase 3) | T-O4, T-O3, T-O1, T-O3 partial — CONFIRMED ABSENT | PASS |
| O-9 / TASK-O5 (Phase 3): `SecretBoxAuthenticationError` unwrapped | T-O3 / TASK-T9 — CONFIRMED ABSENT | PASS |
| C-1..C-10 (Phase 4): no `OhttpClient` tests | T-X (via TASK-T11) — CONFIRMED ABSENT | PASS |

**Note on C-11/C-12:** The review section in `specs/.current/AW-2865/review.md` (Important I-3) correctly identifies that C-11 is not in the cross-phase confirmation table, though it is referenced as a cross-phase tag on TASK-T9. C-12 is also absent from the table. These are quality-of-record gaps in `research.md` that do not affect the QA verdict on the underlying findings — both C-11 and C-12 were confirmed absent by inspection of the test files.

---

## Draft Task Entries Verification

`research.md` contains 13 canonical draft task entries (TASK-T1..TASK-T13). Each was checked against the draft task contract in `plan.md §"Target Interfaces and Contracts"`.

| Field | Required | All 13 entries compliant? |
|---|---|---|
| Task ID | TASK-T{N} sequential | YES (TASK-T1..TASK-T13) |
| File | Specific test file path | YES |
| Missing scenario | Concrete, independently actionable | YES |
| Coverage stops at | Line number or "N/A — not tested at all" | YES |
| Proposed group name | `group(...)` label | YES |
| Severity | Exactly one of BLOCKER / HIGH / IMPROVEMENT | YES — 11 HIGH, 2 IMPROVEMENT, 0 BLOCKER |
| Cross-phase reference | Prior finding ID or NG-N | YES |

The `tasks.md` summary table (lines 231–249) carries 15 entries (T-H1..T-H3, T-B1..T-B5, T-O1..T-O6, T-X1, T-X2) using a different ID scheme than the canonical `research.md` entries. Five entries in `tasks.md` (T-B4, T-B5, T-O2, T-O5, T-O6) have no direct TASK-T counterpart in `research.md`. Two canonical TASK-T entries (TASK-T7 statusCode out-of-range; TASK-T11 OhttpClient error paths) are absent from the `tasks.md` summary table. This divergence is documented as Important I-2 in `review.md`. It is a quality-of-record issue: the canonical set for Iteration 7 is `research.md §"Draft Task Entries"` (TASK-T1..TASK-T13), as declared in `plan.md:12`. Iteration 7 should reconcile the two tables before compiling per-task Markdown files.

---

## Eight Investigation Vectors Coverage

Per `idea.md` and `plan.md §"Target Interfaces and Contracts"`:

| Vector | Addressed by | All gaps identified? |
|---|---|---|
| Cryptographic correctness | TASK-T1 (seq overflow), TASK-T2 (invalid pubkey), TASK-T3 (export edge) | YES |
| Interoperability | TASK-T10 (round-trip), TASK-T13 (live gateway) | YES |
| Parser robustness | TASK-T4, TASK-T5, TASK-T6, TASK-T7, TASK-T8 | YES |
| Privacy risks | TASK-T11 (`sendDirect()` + scheme-enforcement test paths) | YES |
| Network reliability | TASK-T11 (timeout/retry simulation via mock client) | YES |
| KeyConfig management | TASK-T8 (multi-suite), TASK-T10 (key material in round-trip) | YES |
| Test suite | TASK-T12 (fuzz/property coverage) | YES |
| Documentation | TASK-T11 (sendDirect bypass tested via behavior) | YES |

---

## Phase-Specific Risk Zone

| Risk | Assessment | QA finding |
|---|---|---|
| T-O3 and T-O4 together represent an untested AEAD contract | The authentication-failure path and the round-trip decryption path are both absent. Any regression in `ohttpDecapsulate` after a refactor would be undetectable. | HIGH risk — recommend T-O3 and T-O4 as the highest-priority tests to author in the follow-up implementation ticket. |
| 8-byte varint (T-B1) is a JS-target correctness risk | `bhttp.dart:57–63` implements 8-byte varint decode but it is entirely untested. On dart2js / dart2wasm, 64-bit integer overflow silently produces wrong results (F2.6b from Phase 2). | HIGH risk — cannot be detected without test coverage at these values. |
| No `OhttpClient` tests (TASK-T11) | Layers 1–3 have unit tests; Layer 4 has none. All 12 Phase 4 findings (C-1..C-12) are undetectable by the current test suite. | HIGH risk — any client layer regression is invisible. |
| `tasks.md`/`research.md` draft-task divergence | Five entries in `tasks.md` lack `research.md` TASK-T counterparts and could be dropped at Iteration 7. | MODERATE process risk — does not affect code correctness, but may cause findings to be missed at task compilation. Confirm Iteration 7 uses `research.md` as canonical. |
| C-11 and C-12 missing from cross-phase confirmation table | Present in TASK-T9 cross-phase notes but absent from the table. Review section I-3 flags this. | LOW risk to audit correctness — both were confirmed absent by test-file inspection. |

---

## Read-Only Constraint Verification

`git diff HEAD -- lib/ test/ pubspec.yaml example/` confirmed empty at Phase 5 close (per `review.md` lines 857–863). The only working-tree changes at close of phase were:
- `specs/.current/AW-2865/tasklist.md` — progress tick-off for Iteration 5.
- `specs/.current/AW-2865/phase-5/` — four new spec artifacts (prd.md, plan.md, research.md, tasks.md).

PRD constraint 1 and Definition of Done item 3 are satisfied.

---

## PRD Success Criteria Checklist

| # | Criterion | Status |
|---|---|---|
| 1 | Every test file read in full (cited with actual line count) | PASS — hpke_test.dart: 296 lines, bhttp_test.dart: 231 lines, ohttp_test.dart: 164 lines |
| 2 | Every existing test group named by file | PASS — research.md §"Test File Coverage Summary" enumerates all groups |
| 3 | Every gap tied to a specific line range where coverage stops | PASS — all 13 TASK-T entries and all 15 tasks.md entries carry line anchors |
| 4 | Every gap from Phases 1–4 confirmed or refuted | MOSTLY PASS — 30-row cross-phase table complete; C-11/C-12 confirmed absent in research but absent from the table (see I-3 in review.md) |
| 5 | All 8 investigation vectors addressed | PASS — see vector coverage table above |
| 6 | No code changes | PASS — git diff confirmed empty |
| 7 | All draft task entries independently actionable | PASS — each TASK-T carries all five required fields |
| 8 | Severity assigned to every gap | PASS — 11 HIGH, 2 IMPROVEMENT, 0 BLOCKER, 0 TBD |

---

## Definition of Done Verification (plan.md)

| DoD item | Satisfied? | Notes |
|---|---|---|
| 1. `tasks.md` items 5.1–5.11 all checked | YES | All 11 checked |
| 2a. `research.md` contains test file coverage summaries with actual line counts | YES | |
| 2b. `research.md` contains test group tables with test counts | YES | |
| 2c. `research.md` contains RFC 9180 A.1 sub-vector coverage table with line references | YES | |
| 2d. `research.md` contains varint boundary coverage table with explicit YES/NO per value | YES | |
| 2e. `research.md` contains 30-row cross-phase confirmation table | YES — 30 rows present; C-11/C-12 absent from table (quality-of-record gap) | |
| 2f. `research.md` contains new gaps section (NG-1..NG-10) | YES | 10 new gaps documented |
| 2g. `research.md` contains draft task entries (TASK-T1..TASK-T13) | YES | 13 entries |
| 3. `git diff HEAD -- lib/ test/ pubspec.yaml example/` empty | YES | |
| 4. Phase 5 QA report (`phase-5/qa.md`) confirms each PRD success criterion | YES — this file | |
| 5. No draft task entry has unset severity, file reference, or cross-phase reference | YES | |
| 6. All 8 investigation vectors appear across TASK-T1..TASK-T13 | YES | |
| 7. At least one HIGH draft task entry | YES — 11 entries are HIGH | |

---

## Open Items Before Iteration 7

Two quality-of-record items from `review.md` require resolution before Iteration 7 compiles the per-task Markdown files. Neither affects the correctness of the Phase 5 audit findings.

**O-1 (Important I-2 in review.md) — Reconcile `tasks.md` draft-task table with `research.md` TASK-T entries.**
Five entries in `tasks.md` (T-B4 zero-length header name, T-B5 serializeRequest size cap, T-O2 `symLen` past buffer end, T-O5 parse/validate exception inconsistency, T-O6 empty inner payload) have no canonical TASK-T counterpart in `research.md`. Two canonical TASK-T entries (TASK-T7 statusCode out-of-range, TASK-T11 OhttpClient error paths) are absent from the `tasks.md` summary table. Recommended fix: promote T-B4, T-B5, T-O2, T-O5, T-O6 into `research.md` as TASK-T14..TASK-T18 at Iteration 7 compilation, or explicitly note in `tasks.md` that `research.md §"Draft Task Entries"` is the canonical list.

**O-2 (Important I-3 in review.md) — Add C-11 and C-12 rows to the cross-phase confirmation table in `research.md`.**
Both C-11 (`SecretBoxAuthenticationError` reaches wallet boundary — cross-layer confirmation of Phase 3 O-9) and C-12 (`onLog` logs `gatewayBaseUrl` unconditionally) are absent from the table despite being confirmed absent by test-file inspection. C-11 is referenced as a cross-phase note in TASK-T9. Adding two rows to the table would bring the count from 30 to 32 and satisfy the Resolved Question Q4 commitment that the table covers "all named gaps from Phases 1–4".

---

## Final Verdict

**Release with reservations.**

Phase 5 is substantively complete and sound. All 11 tasks are checked and grounded in specific test-file line citations. The cross-phase confirmation table covers the 30 committed gaps. The 13 canonical draft task entries (TASK-T1..TASK-T13) are independently actionable and correctly severity-classified. The read-only constraint is honored. The PRD success criteria are met.

The reservations are quality-of-record only:

1. **Five `tasks.md` draft entries (T-B4, T-B5, T-O2, T-O5, T-O6) lack `research.md` TASK-T counterparts** — they risk being dropped at Iteration 7 unless explicitly promoted or the canonical source is clearly identified.
2. **C-11 and C-12 are absent from the cross-phase confirmation table** in `research.md` despite being confirmed absent by inspection.

Neither reservation affects the engineering lead's ability to act on the findings. The 13 canonical TASK-T entries and the underlying gap analysis are correct as stated. Phase 5 is approved to feed Iteration 7 task compilation with the two editorial fixes above noted as pre-conditions.
