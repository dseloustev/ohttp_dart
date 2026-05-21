# Phase 3 Summary — OHTTP Layer Audit Against RFC 9458

**Ticket:** AW-2865
**Phase:** 3 of N (Iteration 3)
**Status:** CLOSED
**Date completed:** 2026-05-21
**QA verdict:** PASS — Release with Reservations
**Review verdict:** APPROVED (two non-blocking Important items)

---

## What was audited and why

Phase 3 is a read-only, line-by-line audit of `lib/src/ohttp.dart` (258 lines) against
RFC 9458. This file is Layer 3 of the four-layer `ohttp_dart` stack. It sits above
`hpke.dart` (Layer 1, Phase 1) and `bhttp.dart` (Layer 2, Phase 2), and below
`ohttp_client.dart` (Layer 4, deferred to Phase 5).

`ohttp.dart` is the OHTTP encapsulation and decapsulation core. It parses the gateway's
`KeyConfig` wire format, constructs the HPKE info string, seals the pre-serialized BHTTP
request with HPKE, exports the 16-byte response secret, and performs response
decapsulation using plain (unlabeled) HKDF. Every correctness or safety defect at this
layer directly affects the privacy guarantees of the OHTTP protocol for wallet users.

No file under `lib/`, `test/`, `pubspec.yaml`, or `example/` was modified. All writes are
confined to `specs/.current/AW-2865/phase-3/`.

---

## RFC 9458 Section Verdicts

| RFC Section | Subject | Verdict |
|---|---|---|
| §4.1 — KeyConfig wire layout | `key_id` (1 B), `kem_id` (2 B BE), `public_key` (32 B), `symLen` (2 B BE), first KDF+AEAD pair | COMPLIANT |
| §4.1 — Multi-suite advertisement | Behavior when `symLen > 4` (more than one KDF+AEAD pair in buffer) | GAP — silent drop (Finding O-2) |
| §4.1 — Short-buffer handling | `RangeError` vs `FormatException` on truncated `KeyConfig` blob | COMPLIANT — parser guards catch all truncation paths with `FormatException`; no `RangeError` escapes (Finding O-3) |
| §4.3 — HPKE info string | `utf8("message/bhttp request") || 0x00 || 7-byte header` | COMPLIANT |
| §4.3 — Request sealing AAD | AAD passed to `HPKE.Seal` | INTENTIONAL DEVIATION — empty AAD, not header-as-AAD; matches Go reference implementation (Finding O-6) |
| §4.4 — Response nonce extraction | `max(Nk, Nn) = 16` bytes for AES-128-GCM | COMPLIANT |
| §4.4 — Response key derivation | Plain (unlabeled) `HKDF-Extract(salt=enc||response_nonce, ikm=exportedSecret)` and `HKDF-Expand` | COMPLIANT (with maintenance hazard — Finding O-7) |
| §4.4 — Response AEAD AAD | AAD for AES-128-GCM response decryption | COMPLIANT — empty AAD per RFC |

All eight inspected paths produced a verdict. Four are positive (compliant), three
require follow-up work (O-2, O-6, O-7), and one is a positive finding that corrects a
risk stated in `vision.md §5.1` (O-3).

---

## Findings Summary

Eleven findings were recorded. Zero BLOCKERs. Four are HIGH (should fix before wallet
use). Two are IMPROVEMENT. One is MAINTENANCE. Four are positive verdicts.

| ID | Title | Severity | File : Lines | RFC Section |
|---|---|---|---|---|
| O-1 | Wire layout of `OhttpKeyConfig.parse` complies with RFC 9458 §4.1 | — (positive) | `ohttp.dart:32-79` | RFC 9458 §4.1 |
| O-2 | Silent drop of extra KDF+AEAD pairs when `symLen > 4` | HIGH | `ohttp.dart:60-71` | RFC 9458 §4.1 |
| O-3 | `RangeError` on malformed buffer NOT present; parser guards are complete | — (positive) | `ohttp.dart:33-71` | RFC 9458 §4.1 |
| O-4 | `FormatException` (`parse`) vs `UnsupportedError` (`validate`) for same unknown-KEM condition | IMPROVEMENT | `ohttp.dart:51`, `ohttp.dart:82-85` | RFC 9458 §4.1 |
| O-5 | HPKE info string construction matches RFC 9458 §4.3 | — (positive) | `ohttp.dart:232-245` | RFC 9458 §4.3 |
| O-6 | Empty AAD in request sealing: intentional deviation matching Go reference implementation | IMPROVEMENT | `ohttp.dart:149` | RFC 9458 §4.3 |
| O-7 | Plain HKDF in response decap is compliant; maintenance hazard if unified with labeled variants | MAINTENANCE | `ohttp.dart:192-207` | RFC 9458 §4.4 |
| O-8 | Response AEAD uses empty AAD: compliant with RFC 9458 §4.4 | — (positive) | `ohttp.dart:219-223` | RFC 9458 §4.4 |
| O-9 | `SecretBoxAuthenticationError` propagates unwrapped; `package:cryptography` leaks into public API | HIGH | `ohttp.dart:219` | RFC 9458 §4.4 |
| O-10 | `enc` and `exportedSecret` not zeroized in `OhttpEncapsulateResult`; AOT is higher-risk target | HIGH | `ohttp.dart:104-120`, `ohttp.dart:162-166` | RFC 9458 §4.3 |
| O-11 | Test suite missing round-trip, auth-failure, multi-suite, and truncated-ciphertext tests | HIGH | `test/ohttp_test.dart` | RFC 9458 §4.3, §4.4 |

---

## Notable Finding — O-3 Corrects a vision.md Assumption

`vision.md §5.1` predicted that a malformed `KeyConfig` blob with a large `symLen` value
and a short buffer would propagate a `RangeError` rather than a `FormatException`. The
Phase 3 audit found this risk does NOT exist in the current code. The guard at
`ohttp.dart:63` tests both `symLen < 4` and `data.length < offset + symLen` in a single
condition before any list access occurs. Every truncation path that reaches the
symmetric-algorithms section throws `FormatException('Invalid symmetric algorithms
section')`.

Earlier truncation paths (buffer length below 7 bytes, buffer too short for the public
key, buffer too short for `symLen` field itself) are each caught by dedicated guards at
lines 33, 54, and 63, respectively. No `RangeError` escapes through the parser.

A minor improvement remains: the single error message at `ohttp.dart:64` conflates
"semantically invalid `symLen` (< 4)" with "truncated buffer", making the failure mode
indistinguishable to callers. This sub-improvement is documented in Finding O-3 but not
lifted into a separate draft task.

---

## Cross-Phase References

Phase 3 findings that compound patterns identified in Phase 1 or Phase 2:

| Phase 3 finding | Prior-phase findings | Pattern |
|---|---|---|
| O-4 (FormatException vs UnsupportedError inconsistency) | Phase 1 F-3 (HPKE exception type inconsistency); Phase 2 R-3 (BHTTP body-sublist RangeError) | No normalized exception hierarchy across the stack; catch-site handling is unreliable |
| O-10 (enc and exportedSecret not zeroized) | Phase 1 F-2 (HpkeSenderContext fields key, baseNonce, exporterSecret not zeroized) | Key-material zeroization absent at both the HPKE context level and the OHTTP result level; `exportedSecret` is the most sensitive field at the OHTTP layer |
| O-11 (test suite gaps) | Phase 1 F-6/F-7 (HPKE test gaps); Phase 2 R-6 (BHTTP negative-test gap) | Test-coverage gap repeats at every layer; Iteration 7 should address all layers in a coordinated effort |

Finding O-2 (silent multi-suite drop) is a new gap at Layer 3 with no direct Phase 1 or
Phase 2 counterpart.

The `testKeyPair` injection hook from Phase 1 F-1 surfaces again at the OHTTP layer:
`ohttpEncapsulate` (`ohttp.dart:134`) exposes the same optional parameter in its public
signature. This was not classified as a separate Phase 3 finding because it is a
consequence of Phase 1 F-1. Confirmation that `ohttp_client.dart` does not pass a
non-null `testKeyPair` in production is deferred to Phase 5.

---

## Draft Task Entries for Iteration 7 (TASK-O1 through TASK-O7)

All seven tasks target `lib/src/ohttp.dart` or `test/ohttp_test.dart` and are
independently actionable. Full remediation directions are in `research.md` "Draft Task
Entries".

| ID | Title | Severity | Primary file : lines |
|---|---|---|---|
| TASK-O1 | Fix silent drop of extra KDF+AEAD pairs in `OhttpKeyConfig.parse` | HIGH | `ohttp.dart:60-78` |
| TASK-O2 | Unify exception types for unsupported KEM in `parse()` and `validate()` | IMPROVEMENT | `ohttp.dart:51`, `ohttp.dart:82-85` |
| TASK-O3 | Expand inline comment at empty-AAD `ctx.seal` call site | IMPROVEMENT | `ohttp.dart:148-150` |
| TASK-O4 | Add maintenance-hazard comment at plain-HKDF call sites in `ohttpDecapsulate` | MAINTENANCE | `ohttp.dart:192-207` |
| TASK-O5 | Wrap `SecretBoxAuthenticationError` in a library-owned `OhttpAuthenticationException` | HIGH | `ohttp.dart:217-225` |
| TASK-O6 | Add best-effort zeroization for `OhttpEncapsulateResult.enc` and `exportedSecret` | HIGH | `ohttp.dart:104-120`, `ohttp.dart:162-166` |
| TASK-O7 | Add missing test cases for `ohttp_test.dart` (round-trip, auth-failure, multi-suite, truncated-ciphertext) | HIGH | `test/ohttp_test.dart` |

TASK-O7 test scenario #2 (authentication failure) depends on the `OhttpAuthenticationException`
type introduced by TASK-O5. These tasks can be assigned independently; scenario #2 of
TASK-O7 should be implemented after TASK-O5 is complete.

---

## Pre-Iteration-7 Action Items

Two non-blocking Important items from the review must be resolved before Iteration 7
compiles draft entries into Jira tickets:

**I-1 — O-7 severity inconsistency.** The PRD Risks table classifies the plain-HKDF
maintenance hazard as HIGH; `research.md`, `tasks.md`, and `plan.md` classify it as
MAINTENANCE (IMPROVEMENT). Pick one label and apply consistently. The MAINTENANCE
(sub-IMPROVEMENT) position is more defensible: the underlying code is compliant and the
risk is a future-refactor regression, not a current defect.

**I-2 — MAINTENANCE tier not in vision §2 severity model.** The four-tier scheme is
introduced in `plan.md` but the PRD inherits only the three-tier model from `vision.md
§2`. Either fold MAINTENANCE rows into IMPROVEMENT with a parenthetical "(maintenance
hazard)" qualifier, or add a one-line note to PRD §Resolved Questions explicitly admitting
the fourth tier.

Two additional loose ends from the QA report (not blocking Iteration 7, but should be
resolved before Jira import):

- The O-3 sub-improvement (splitting the ambiguous `'Invalid symmetric algorithms
  section'` error message at `ohttp.dart:64`) is documented but has no draft task. Add
  TASK-O8 or explicitly record it as declined.
- Add a "Phase 5 must verify" note in `tasklist.md` Phase 5 row for the `testKeyPair`
  parameter at `ohttp.dart:134`, so it is not lost at Iteration 7 compilation.

---

## QA and Review Outcome

**QA (`phase-3/qa.md`):** PASS — Release with Reservations. All 8 QA checks pass. No
BLOCKER was identified. Four HIGH-severity items (O-2, O-9, O-10, O-11) should be
remediated before wallet production use. Two IMPROVEMENT items (O-4, O-6) and one
MAINTENANCE item (O-7) are recommended backlog work.

**Review (`review.md` Phase 3 section):** APPROVED with two Important items (I-1 and
I-2). All 11 tasks checked, all 11 findings filed with exact line citations and RFC
section references, all 7 draft tasks independently actionable, read-only constraint
confirmed satisfied by both the review and git status. Five Nice-to-have items (N-1
through N-5) noted for optional polish.

---

## Key Decisions Made

- No code was changed. All findings are deferred to future implementation phases.
- The `RangeError`-on-short-buffer risk from `vision.md §5.1` was re-investigated and
  confirmed absent from the current code (O-3). The parser guards are sufficient.
- Empty AAD in `ctx.seal` (`ohttp.dart:149`) is confirmed intentional and documented as
  an IMPROVEMENT (inline-comment expansion only, no protocol change).
- Plain HKDF in response decapsulation is confirmed compliant with RFC 9458 §4.4 and must
  not be unified with the labeled `LabeledExtract`/`LabeledExpand` variants from
  `hpke.dart`. This distinction is recorded as MAINTENANCE (TASK-O4).
- `SecretBoxAuthenticationError` wrapping target is `ohttp.dart` (Resolved Question #3 in
  `prd.md`); the new `OhttpAuthenticationException` type is defined and thrown in `ohttp.dart`.
- Zeroization severity is HIGH (O-10), extending Phase 1 F-2 to the OHTTP layer. Dart VM
  does not guarantee zero-writes prevent GC-root or JIT-optimized memory access;
  best-effort zeroization (explicit zero-fill before nulling references) is the actionable
  remediation, particularly for AOT-compiled deployments.

---

## Artifacts

- `specs/.current/AW-2865/phase-3/prd.md`
- `specs/.current/AW-2865/phase-3/plan.md`
- `specs/.current/AW-2865/phase-3/research.md`
- `specs/.current/AW-2865/phase-3/tasks.md`
- `specs/.current/AW-2865/phase-3/qa.md`
- `specs/.current/AW-2865/review.md` (Phase 3 section, lines 341–532)
