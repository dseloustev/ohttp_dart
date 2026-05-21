# Phase 1 Summary — HPKE Layer Audit Against RFC 9180

**Ticket:** AW-2865
**Phase:** 1 of N
**Status:** CLOSED
**Date completed:** 2026-05-21
**Verdict:** PASS (QA) / APPROVED (review)

---

## What was audited and why

Phase 1 is a read-only, line-by-line audit of `lib/src/hpke.dart` (314 lines) against RFC 9180. This file is the bottom layer of the four-layer `ohttp_dart` stack and is the cryptographic foundation for everything above it. Its correctness is a prerequisite for trusting the OHTTP layer (`ohttp.dart`), the Binary HTTP layer (`bhttp.dart`), and the high-level client (`ohttp_client.dart`).

The driver for the investigation is AW-2865: assessing `ohttp_dart` for production use in a non-custodial crypto wallet. Phase 1 specifically answers: does the HPKE implementation match RFC 9180, and what risks does it introduce to the wallet threat model?

No file under `lib/`, `test/`, `pubspec.yaml`, or `example/` was modified. All writes were confined to `specs/.current/AW-2865/phase-1/`.

---

## RFC 9180 Section Verdicts

| RFC Section | Subject | Verdict |
|---|---|---|
| §4 — LabeledExtract | Suite ID split (KEM vs HPKE), label embedding, HMAC-SHA-256 invocation | Matches RFC |
| §4 — LabeledExpand | 2-byte big-endian length prefix, concatenation order | Matches RFC |
| §4 — hkdfExtract empty-salt handling | 32 zero-bytes fallback (RFC 5869 §2.2) | Matches RFC |
| §4 — hkdfExpand T(i) construction | Single-byte counter, correct truncation | Matches RFC |
| §4.1 — KEM Encap (DHKEM X25519) | DH step, enc serialization, kem_context byte order, ExtractAndExpand labels | Matches RFC |
| §5.1 — Key Schedule (Base Mode) | mode_base=0, psk_id_hash, info_hash, secret, key, base_nonce, exp derivations | Matches RFC |
| §5.2 — Nonce derivation and sequence counter | baseNonce XOR I2OSP(_seq, 12), post-increment sequencing | Matches RFC (within single-use operating mode; see F-4) |
| §5.3 — Export | LabeledExpand with HPKE suite ID, label "sec", exporterSecret as PRK | Matches RFC |

All eight sub-sections produced a positive verdict. No cryptographic defect was found. The existing `test/hpke_test.dart` suite passes RFC 9180 Appendix A.1 test vectors for SetupBaseS, seal (seq=0 and seq=1), and export (three context variants), corroborating the line-by-line verdicts.

---

## Findings Summary

Seven findings were produced. None are BLOCKER. Two are HIGH (must address before wallet use). Five are IMPROVEMENT (recommended backlog items).

| ID | Title | Severity | File : Line Range | RFC Section |
|---|---|---|---|---|
| F-1 | `testKeyPair` optional parameter in production `setupBaseS` signature allows any caller to bypass X25519 key generation | HIGH | `hpke.dart:47-48`, `hpke.dart:71-79`, `ohttp.dart:134-146` | RFC 9180 §4.1 |
| F-2 | No zeroization of `key`, `baseNonce`, `exporterSecret` after seal/export; key material remains on Dart heap | IMPROVEMENT | `hpke.dart:259-262`, `ohttp.dart:113-114` | RFC 9180 §5.1, §5.2 |
| F-3 | `_seq` overflow guard throws untyped `StateError`; callers cannot catch it selectively | IMPROVEMENT | `hpke.dart:275-277` | RFC 9180 §5.2 |
| F-4 | Nonce XOR covers only 4 bytes (32-bit `_seq`); overflow guard capped at `2^32` rather than RFC's `2^96 - 1` | IMPROVEMENT | `hpke.dart:275-285` | RFC 9180 §5.2 |
| F-5 | `hkdfExtract` and `hkdfExpand` are public static methods; risk of misuse in place of labeled variants | IMPROVEMENT | `hpke.dart:225-254`, `lib/ohttp_dart.dart` | RFC 9180 §4 |
| F-6 | No test covers `_seq` overflow boundary; silent regression possible on refactor | IMPROVEMENT | `test/hpke_test.dart`, `hpke.dart:274-277` | RFC 9180 §5.2 |
| F-7 | RFC 9180 A.1 vectors use non-empty AAD; actual OHTTP usage calls `seal` with empty AAD — no integration test confirms gateway interoperability | HIGH | `test/hpke_test.dart:104-119`, `ohttp.dart:149` | RFC 9458 §4.6.1, RFC 9180 §5.2 |

Severity assignments were resolved before research began and are recorded in `research.md` "Resolved Questions". Key resolutions: F-1 (testKeyPair) is HIGH not BLOCKER because the attack requires deliberate caller action; F-2 (zeroization) is IMPROVEMENT because the exposed keys are ephemeral OHTTP session keys, not user wallet keys; F-3/F-4/F-6 are IMPROVEMENT because the wallet always uses single-use contexts, making overflow unreachable; F-7 is HIGH because an untested path risks gateway interoperability failures.

---

## Draft Jira Tasks Produced (TASK-1 through TASK-7)

All seven tasks are independently actionable. Full descriptions including remediation directions are in `research.md` "Draft Task Entries".

| Task | Title | Severity | Primary file target |
|---|---|---|---|
| TASK-1 | Replace `testKeyPair` injection with a test-only factory or assertion | HIGH | `hpke.dart:47-48`, `ohttp.dart:134-146` |
| TASK-2 | Add key-material zeroization for `HpkeSenderContext` fields | IMPROVEMENT | `hpke.dart:259-262`, `ohttp.dart:113-114` |
| TASK-3 | Replace `StateError` on `_seq` overflow with a typed exception | IMPROVEMENT | `hpke.dart:275-277` |
| TASK-4 | Extend nonce XOR to full 12 bytes and update overflow guard | IMPROVEMENT | `hpke.dart:275-285` |
| TASK-5 | Restrict unlabeled `hkdfExtract`/`hkdfExpand` from the public API surface | IMPROVEMENT | `hpke.dart:225-254`, `lib/ohttp_dart.dart` |
| TASK-6 | Add negative test for `_seq` overflow path | IMPROVEMENT | `test/hpke_test.dart` |
| TASK-7 | Add integration test: `seal` with empty AAD against OHTTP gateway or reference vector | HIGH | `test/hpke_test.dart`, `ohttp.dart:149` |

Note: TASK-3 and TASK-4 target overlapping code in `_computeNonce` (lines 275-285). If assigned concurrently, a merge conflict is expected; sequential scheduling or merging into one task is recommended.

---

## PRD Acceptance Criteria — Outcome

| Criterion | Result |
|---|---|
| AC-1: All RFC 9180 sections covered (§4, §4.1, §5.1, §5.2, §5.3) | PASS — 8 sub-section verdicts, none skipped |
| AC-2: Vision §4 scrutiny table fully addressed | PASS — single-suite hard-coding documented; seq overflow in F-3/F-4/F-6; testKeyPair in F-1 |
| AC-3: Severity assigned to every finding | PASS — 7 findings, all carry exactly one label; 2 HIGH, 5 IMPROVEMENT, 0 BLOCKER |
| AC-4: No code changes | PASS — `git diff HEAD -- lib/ test/` is empty |
| AC-5: Draft tasks independently actionable | PASS — 7 tasks, each with a single-file target and concrete remediation direction |

---

## Open Items Deferred to Later Phases

1. `lib/ohttp_dart.dart` re-export surface — Phase 3. Confirm whether `HpkeSender` (and thus `hkdfExtract`/`hkdfExpand`) is re-exported to external consumers.
2. `package:cryptography` version pinning — cross-cutting / supply-chain phase. Version range not reviewed for CVEs or breaking changes.
3. Web/WASM integer semantics — deferred unless the wallet targets Flutter Web or WASM. `dart2js` 53-bit integers may affect `_seq` arithmetic.
4. `export()` length bound (`L <= 255 * Nh = 8160`) — New Technical Question #4 in `research.md`. Not promoted to F-8 in Phase 1. Must be explicitly resolved (promote to TASK-8 or record Phase 3 deferral) before Jira import to avoid loss during triage.
5. F-7 scope placement — the review recommends moving F-7 to a "Cross-Phase References" sub-table or downgrading its Phase 1 severity to IMPROVEMENT with a Phase 3 pointer. Presentation choice; must be resolved before Jira import (manual check #1 in `qa.md`).

---

## Key Decisions Made

- No code was changed. All findings are deferred to future implementation phases as Jira tasks.
- Severity model from vision §2 applied uniformly. Severity resolutions are recorded in `research.md` "Resolved Questions".
- Single-use context confirmed by the engineering lead. The wallet always instantiates one `HpkeSenderContext` per request and calls `seal()` exactly once. This reduces F-3, F-4, and F-6 from potential HIGH to IMPROVEMENT.
- Empty AAD in `ohttp.dart` is intentional and matches the Go reference implementation. F-7 calls for a test, not a code change.

---

## Artifacts

- `specs/.current/AW-2865/phase-1/prd.md`
- `specs/.current/AW-2865/phase-1/plan.md`
- `specs/.current/AW-2865/phase-1/tasks.md`
- `specs/.current/AW-2865/phase-1/research.md`
- `specs/.current/AW-2865/phase-1/qa.md`
- `specs/.current/AW-2865/review.md`
