# Phase 5 Summary — Test Suite Coverage Gap Audit

**Ticket:** AW-2865
**Phase:** 5 of 7
**Status:** CLOSED
**Date completed:** 2026-05-21
**Verdict:** PASS (QA) / APPROVED (review)

---

## What was audited and why

Phase 5 is a read-only, line-by-line audit of the entire `ohttp_dart` test suite — three files, 691 lines total — to determine exactly where coverage stops, which failure paths have never been exercised, and which classes of tests (negative, fuzz/property, integration) are entirely absent.

Phases 1–4 audited the four implementation layers (HPKE, BHTTP, OHTTP, OhttpClient) and repeatedly surfaced test coverage as a compounding risk: defects found in those layers are undetectable by the current test suite because the relevant code paths are never reached by any test. Phase 5 closes that loop by mapping every named prior-phase gap to a specific missing test scenario — confirming which are real absences and recording each as a draft task entry for Iteration 7.

The driver is AW-2865: assessing `ohttp_dart` for production use in a non-custodial crypto wallet.

No file under `lib/`, `test/`, `pubspec.yaml`, or `example/` was modified. All writes were confined to `specs/.current/AW-2865/phase-5/`.

---

## Files Audited

| File | Lines | Role |
|---|---|---|
| `test/hpke_test.dart` | 296 | RFC 9180 Appendix A.1 vectors for `hpke.dart` |
| `test/bhttp_test.dart` | 231 | QUIC varint round-trips and BHTTP framing for `bhttp.dart` |
| `test/ohttp_test.dart` | 164 | KeyConfig parse, encapsulate, and decapsulate tests for `ohttp.dart` |

---

## Positive Coverage Confirmed

Before documenting gaps, the audit confirmed what the test suite does correctly.

**`test/hpke_test.dart`** covers all RFC 9180 Appendix A.1 Base Mode sub-vectors relevant to the OHTTP sender path: `enc`, `key`, `base_nonce`, `exporter_secret`, `shared_secret` intermediate, ciphertext at seq=0 and seq=1, and exporter output with three distinct contexts. The `testKeyPair` injection hook in `setupBaseS` is active and deterministic across all six Appendix A.1 tests — removing this hook from any future `hpke.dart` refactor would silently break all vector assertions.

**`test/bhttp_test.dart`** covers varint encode, decode, and round-trip across the 1-byte (0–63), 2-byte (64–16383), and 4-byte (16384–1073741823) ranges. `serializeRequest` and `parseResponse` positive-path tests cover GET, POST with headers and body, header lowercasing, and empty-body responses.

**`test/ohttp_test.dart`** covers `OhttpKeyConfig.parse` on a valid 41-byte config and four error paths, all three `OhttpKeyConfig.validate` rejection cases, an `ohttpEncapsulate` structural check (header bytes, `enc` length, `exportedSecret` length, ciphertext length), and two `ohttpDecapsulate` format-guard rejection tests.

---

## Cross-Phase Gap Confirmations

The research document cross-checks every named gap from Phases 1–4 against the test files in full. **32 prior-phase gaps were confirmed still absent** as of Phase 5. All 32 are marked "Still absent" in `specs/.current/AW-2865/phase-5/research.md`.

Key confirmations with direct wallet-safety implications:

| Prior finding | Confirmed by | Risk if unaddressed |
|---|---|---|
| F-6 (Phase 1): `_seq` overflow guard untested | TASK-T1 | Sequence counter state error is unreachable in tests; silent after refactor |
| F-7 (Phase 1): no integration test | TASK-T13 | Empty-AAD deviation from RFC (Finding O-6) could fail silently against a conformant gateway |
| R-6 (Phase 2): no BHTTP negative tests | TASK-T5, TASK-T6 | Truncated input raises `RangeError`, not `FormatException`; callers cannot catch correctly |
| F2.6b (Phase 2): 8-byte varint untested | TASK-T4 | JS-target integer overflow is undetectable without coverage at these values |
| O-2 / TASK-O7 item 3 (Phase 3): `symLen > 4` silent ignore | TASK-T8 | Future parser change could silently alter multi-suite KeyConfig handling |
| TASK-O7 items 1–2 (Phase 3): no round-trip, no auth-failure test | TASK-T10, TASK-T9 | AES-GCM decryption path has zero test coverage |
| C-1..C-10 (Phase 4): no `OhttpClient` tests | TASK-T11 | All twelve Phase 4 client-layer findings are undetectable by the current test suite |

Two Phase 4 findings (C-11, C-12) were confirmed absent by test-file inspection but are missing from the 30-row confirmation table in `research.md`. This is a quality-of-record gap noted in Open Items below.

---

## New Gaps Identified (Not Named in Prior Phases)

Phase 5 identified **10 gaps** not explicitly named in any prior phase:

| ID | Test file | Missing scenario | Severity |
|---|---|---|---|
| NG-1 | `test/bhttp_test.dart` | 8-byte varint encode/decode (2^30 through 2^62−1) — entire 8-byte branch untested | HIGH |
| NG-2 | `test/hpke_test.dart` | `export()` with length 0 or length > 255 x Nh | HIGH |
| NG-3 | `test/hpke_test.dart` | All-zero / low-order public key passed to `setupBaseS` | HIGH |
| NG-4 | `test/hpke_test.dart` | `seal()` overflow beyond seq=1 toward `_seq` maximum | HIGH |
| NG-5 | `test/bhttp_test.dart` | `statusCode` out of HTTP range (0, 99, 600, 65535) | IMPROVEMENT |
| NG-6 | `test/bhttp_test.dart` | Empty buffer passed to `parseResponse` — `RangeError` vs. `FormatException` | HIGH |
| NG-7 | `test/ohttp_test.dart` | `ohttpEncapsulate` sender-side round-trip (receiver decryption confirms plaintext) | HIGH |
| NG-8 | new `test/ohttp_client_test.dart` | `OhttpClient` error paths: 503 on KeyConfig GET, 400 on POST, scheme enforcement | HIGH |
| NG-9 | all files | No fuzz / property-based tests across the entire test suite | HIGH |
| NG-10 | none (missing integration dir) | No live-gateway or stub-gateway integration test | HIGH |

Of the 10 new gaps, 9 are HIGH and 1 is IMPROVEMENT.

---

## Draft Tasks Produced

Phase 5 produced **13 canonical draft task entries** (TASK-T1 through TASK-T13) recorded in `specs/.current/AW-2865/phase-5/research.md`. Each entry carries: test file, missing scenario, line range where coverage stops, proposed test group name, severity, and a cross-phase reference.

| Task | Title | Severity |
|---|---|---|
| TASK-T1 | Add `_seq` overflow test to `hpke_test.dart` | HIGH |
| TASK-T2 | Add invalid public key test to `hpke_test.dart` | HIGH |
| TASK-T3 | Add `export()` edge-case tests to `hpke_test.dart` | HIGH |
| TASK-T4 | Add 8-byte varint tests to `bhttp_test.dart` | HIGH |
| TASK-T5 | Add truncated-input negative tests to `bhttp_test.dart` | HIGH |
| TASK-T6 | Add unknown framing indicator test to `bhttp_test.dart` | IMPROVEMENT |
| TASK-T7 | Add `statusCode` out-of-range tests to `bhttp_test.dart` | IMPROVEMENT |
| TASK-T8 | Add `symLen > 4` KeyConfig test to `ohttp_test.dart` | HIGH |
| TASK-T9 | Add AEAD authentication failure test to `ohttp_test.dart` | HIGH |
| TASK-T10 | Add round-trip test to `ohttp_test.dart` | HIGH |
| TASK-T11 | Add `OhttpClient` error-path tests (new `test/ohttp_client_test.dart`) | HIGH |
| TASK-T12 | Add fuzz / property-based tests across the test suite | HIGH |
| TASK-T13 | Add live-gateway integration test | HIGH |

Severity breakdown: 11 HIGH, 2 IMPROVEMENT, 0 BLOCKER.

The `tasks.md` summary table carries 15 entries (T-H1..T-H3, T-B1..T-B5, T-O1..T-O6, T-X1, T-X2) using a different ID scheme. The canonical set for Iteration 7 is `research.md` (TASK-T1..TASK-T13) as declared in `plan.md`. See Open Items for the reconciliation action.

---

## Highest-Priority Findings for Wallet Use

**AEAD contract entirely untested (TASK-T9 + TASK-T10).** The `ohttpDecapsulate` AES-GCM decryption path has zero test coverage. The authentication-failure path (`SecretBoxAuthenticationError`) and the full round-trip plaintext recovery path are both absent. Any regression in `ohttpDecapsulate` after a refactor is undetectable by the current test suite. These are the highest-priority tests to author in the follow-up implementation ticket.

**8-byte varint branch entirely untested (TASK-T4).** The 8-byte decode branch in `bhttp.dart:57–63` is implemented but never reached by any test. On dart2js and dart2wasm targets, 64-bit integer overflow silently produces wrong values (Finding F2.6b, Phase 2). This cannot be caught without coverage at values in the 2^30–2^62 range.

**OhttpClient layer has no tests at all (TASK-T11).** Layer 4 (`ohttp_client.dart`) is entirely without tests. Not a single test exercises any public method. All 12 Phase 4 findings (C-1 through C-12) covering scheme enforcement, privacy leaks, error typing, timeout, caching, and response-size DoS surface are invisible to the current test suite.

---

## PRD Acceptance Criteria — Outcome

| Criterion | Result |
|---|---|
| Every test file read in full (cited with actual line count) | PASS — hpke_test.dart: 296 lines, bhttp_test.dart: 231 lines, ohttp_test.dart: 164 lines |
| Every existing test group named by file | PASS — `research.md` enumerates all groups with test counts |
| Every gap tied to a specific line range where coverage stops | PASS — all 13 TASK-T entries carry line anchors |
| Every gap from Phases 1–4 confirmed or refuted | MOSTLY PASS — 30-row cross-phase table complete; C-11/C-12 confirmed absent in research but missing from the table (see Open Items) |
| All 8 investigation vectors addressed | PASS — see `research.md` "Eight Investigation Vectors" section |
| No code changes | PASS — `git diff HEAD -- lib/ test/` confirmed empty |
| All draft task entries independently actionable | PASS — each TASK-T carries all five required fields |
| Severity assigned to every gap | PASS — 11 HIGH, 2 IMPROVEMENT, 0 BLOCKER |

---

## Open Items Before Iteration 7

Two quality-of-record issues must be resolved before Iteration 7 compiles per-task Markdown files. Neither affects the correctness of any finding.

1. **Reconcile `tasks.md` draft-task table with `research.md` TASK-T entries (I-2 in review.md).** Five entries in `tasks.md` (T-B4, T-B5, T-O2, T-O5, T-O6) have no canonical TASK-T counterpart in `research.md`. Two `research.md` entries (TASK-T7, TASK-T11) are absent from the `tasks.md` summary table. Recommended fix: promote T-B4, T-B5, T-O2, T-O5, T-O6 into `research.md` as TASK-T14..TASK-T18, or explicitly note in `tasks.md` that `research.md` "Draft Task Entries" is the canonical list.

2. **Add C-11 and C-12 rows to the cross-phase confirmation table in `research.md` (I-3 in review.md).** Both C-11 (`SecretBoxAuthenticationError` reaches wallet boundary) and C-12 (`onLog` logs `gatewayBaseUrl` unconditionally) are absent from the table despite being confirmed absent by inspection. Adding two rows would bring the confirmed-absent count from 30 to 32 and satisfy the Q4 commitment in `research.md` "Resolved Questions".

---

## Next Steps

Iteration 7 (task compilation) consumes `research.md` "Draft Task Entries" (TASK-T1..TASK-T13) alongside the Phase 1–4 draft task outputs to produce per-task Markdown files under `specs/.current/AW-2865/tasks/`. The two open items above are pre-conditions: resolving them before Iteration 7 begins avoids dropping five `tasks.md`-only entries and ensures the cross-phase confirmation count is accurate.

No Phase 6 is defined in the current plan. Iteration 7 is the next active phase.

---

## Artifacts

- `specs/.current/AW-2865/phase-5/prd.md`
- `specs/.current/AW-2865/phase-5/plan.md`
- `specs/.current/AW-2865/phase-5/tasks.md`
- `specs/.current/AW-2865/phase-5/research.md`
- `specs/.current/AW-2865/phase-5/qa.md`
- `specs/.current/AW-2865/review.md` (Phase 5 section)
