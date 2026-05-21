# QA Report — AW-2865 Phase 3: OHTTP Layer Audit Against RFC 9458

**Date:** 2026-05-21
**Phase:** 3
**Scope:** `lib/src/ohttp.dart` audit against RFC 9458. Read-only — no code in `lib/`, `test/`, `pubspec.yaml`, or `example/` was modified.
**Verdict: PASS**

---

## Phase Scope

Phase 3 targeted Layer 3 of the four-layer stack: `ohttp.dart` (257–258 lines). The QA plan verifies that:

1. All 11 tasks (3.1–3.11) in `tasks.md` are marked `[x]` complete.
2. Every finding O-1 through O-11 in `research.md` cites a specific `ohttp.dart:NNN` or `ohttp.dart:NNN–NNN` line range (or `test/ohttp_test.dart` for O-11).
3. All severity classifications are present and drawn from the `BLOCKER / HIGH / IMPROVEMENT / MAINTENANCE` taxonomy, with no blank or "TBD" entries.
4. The two PRD acceptance criteria requiring exact line citations are satisfied: the empty-AAD finding (O-6) cites `ohttp.dart:149` and RFC 9458 §4.3; the plain-HKDF finding (O-7) cites `ohttp.dart:192-207` and RFC 9458 §4.4.
5. Draft task entries TASK-O1 through TASK-O7 exist in `research.md` and `tasks.md` with file/line, RFC section, severity, and actionable remediation direction.
6. No source file under `lib/`, `test/`, `pubspec.yaml`, or `example/` was modified.
7. The `review.md` Phase 3 verdict is `APPROVED` with two Important items noted (I-1: O-7 severity inconsistency; I-2: MAINTENANCE tier absent from PRD severity model).
8. Required cross-phase references are present for O-4, O-10, and O-11.

This QA report does not re-verify RFC text or re-run code. It verifies the audit artifacts against the PRD acceptance criteria, the plan's Definition of Done, and the review sign-off in `review.md`.

---

## Check 1: All 11 tasks marked `[x]` in `tasks.md`

**Criterion:** Every checklist item 3.1–3.11 in `specs/.current/AW-2865/phase-3/tasks.md` must carry `[x]`.

| Task | Description | Status in tasks.md |
|---|---|---|
| 3.1 | Read full file; confirm `ast-index` outline, verify line ranges for all symbols | `[x]` |
| 3.2 | Trace `OhttpKeyConfig.parse` wire layout against RFC 9458 §4.1 | `[x]` |
| 3.3 | Confirm silent drop of extra KDF+AEAD pairs when `symLen > 4` | `[x]` |
| 3.4 | Confirm short-buffer `RangeError` vs `FormatException` behavior | `[x]` |
| 3.5 | Note `FormatException`/`UnsupportedError` inconsistency for unknown KEM | `[x]` |
| 3.6 | Verify HPKE info string construction against RFC 9458 §4.3 | `[x]` |
| 3.7 | Confirm empty AAD in `ctx.seal` and cross-reference RFC 9458 §4.3 | `[x]` |
| 3.8 | Verify plain (unlabeled) HKDF path in `ohttpDecapsulate` against RFC 9458 §4.4 | `[x]` |
| 3.9 | Confirm response AEAD uses empty AAD | `[x]` |
| 3.10 | Note `SecretBoxAuthenticationError` propagation and `package:cryptography` API leak | `[x]` |
| 3.11 | Record each finding as a draft task entry (file, line, RFC section, severity) | `[x]` |

All 11 checklist items are marked complete.

**Check 1: PASS**

---

## Check 2: Every finding (O-1 through O-11) cites an exact line range

**Criterion:** Each finding in `research.md` must cite `ohttp.dart:NNN` or `ohttp.dart:NNN–NNN` (or `test/ohttp_test.dart` for O-11). Positive-verdict findings must also carry a line range so the cited code is traceable.

| Finding | Verdict / Severity | Line Citation | RFC Section |
|---|---|---|---|
| O-1 | COMPLIANT (positive) | `ohttp.dart:32-79` | RFC 9458 §4.1 |
| O-2 | HIGH | `ohttp.dart:60-71` | RFC 9458 §4.1 |
| O-3 | COMPLIANT (positive — vision assumption corrected) | `ohttp.dart:33-71` | RFC 9458 §4.1 |
| O-4 | IMPROVEMENT | `ohttp.dart:51`, `ohttp.dart:82-85` | RFC 9458 §4.1 |
| O-5 | COMPLIANT (positive) | `ohttp.dart:232-245` | RFC 9458 §4.3 |
| O-6 | IMPROVEMENT | `ohttp.dart:149` | RFC 9458 §4.3 / §4.6.1 |
| O-7 | COMPLIANT + MAINTENANCE hazard | `ohttp.dart:192-207` | RFC 9458 §4.4 |
| O-8 | COMPLIANT (positive) | `ohttp.dart:219-223` | RFC 9458 §4.4 |
| O-9 | HIGH | `ohttp.dart:219` | RFC 9458 §4.4 |
| O-10 | HIGH | `ohttp.dart:104-120`, `ohttp.dart:162-166` | RFC 9458 §4.3 |
| O-11 | HIGH | `test/ohttp_test.dart` (whole file) | RFC 9458 §4.3, §4.4 |

All 11 findings carry explicit file:line citations. All non-positive findings are tied to at least one RFC 9458 section. O-7's maintenance hazard additionally cites RFC 5869 / RFC 9180 distinction. O-6 additionally cites the Go reference implementation.

**Check 2: PASS**

---

## Check 3: No blank or "TBD" severities in the Limitations & Risks table

**Criterion:** The `research.md` "Limitations & Risks" table must have a severity (or `—` for positive verdicts) in every row. No entry may be blank or "TBD."

| Finding | Severity | Populated? |
|---|---|---|
| O-1 | `—` (positive) | Yes |
| O-2 | HIGH | Yes |
| O-3 | `—` (positive) | Yes |
| O-4 | IMPROVEMENT | Yes |
| O-5 | `—` (positive) | Yes |
| O-6 | IMPROVEMENT | Yes |
| O-7 | MAINTENANCE | Yes |
| O-8 | `—` (positive) | Yes |
| O-9 | HIGH | Yes |
| O-10 | HIGH | Yes |
| O-11 | HIGH | Yes |

No blank or TBD entries found in the 11-row table.

**Check 3: PASS**

---

## Check 4: PRD acceptance criteria — empty-AAD and plain-HKDF findings fully cited

**Criterion (from `phase-3/prd.md` Success / Metrics):** "Both the empty-AAD finding and the plain-HKDF-vs-labeled-HKDF finding cite the exact line in `ohttp.dart` plus the RFC 9458 section that motivates the concern."

### 4a — Empty-AAD finding (O-6)

- **Exact `ohttp.dart` line cited:** `ohttp.dart:149` (`final ct = await ctx.seal(Uint8List(0), binaryRequest);`)
- **RFC 9458 section cited:** §4.3 / §4.6.1 (request encapsulation AAD prescription)
- **Go reference cited:** Yes — "intentional deviation matching Go reference implementation"
- **Remediation documented:** Inline comment expansion at `ohttp.dart:148-150`; no ADR required

Criterion satisfied. Finding text confirms the exact line, the RFC section, and the deviation rationale.

**Check 4a: PASS**

### 4b — Plain-HKDF finding (O-7)

- **Exact `ohttp.dart` line range cited:** `ohttp.dart:192-207`, with sub-line breakdown:
  - `:193` — `HpkeSender.hkdfExtract(salt, exportedSecret)` (plain RFC 5869 HKDF-Extract)
  - `:196-200` — `HpkeSender.hkdfExpand(prk, utf8.encode('key'), _nk)` (label "key", 16 B)
  - `:203-207` — `HpkeSender.hkdfExpand(prk, utf8.encode('nonce'), _nn)` (label "nonce", 12 B)
- **RFC 9458 section cited:** §4.4 (response key derivation)
- **RFC 5869 / RFC 9180 distinction documented:** Yes — `hkdfExtract`/`hkdfExpand` are plain RFC 5869; `LabeledExtract`/`LabeledExpand` prepend the RFC 9180 suite ID; the two are not interchangeable
- **Maintenance hazard documented:** Yes — substitution would silently derive different key material with no compile-time signal

Criterion satisfied. Finding provides full sub-line breakdown, RFC section, and the labeled-vs-unlabeled distinction.

**Check 4b: PASS**

**Check 4 overall: PASS**

---

## Check 5: Draft task entries TASK-O1 through TASK-O7 are complete

**Criterion (from `phase-3/plan.md` §Draft task contract):** Each non-positive finding must produce exactly one draft task with: title, severity, file/line, RFC section (or explicit non-RFC justification), and actionable remediation direction.

| Task | Severity | File : Line | RFC Section | Remediation | Independent? |
|---|---|---|---|---|---|
| TASK-O1 | HIGH | `ohttp.dart:60-78` | RFC 9458 §4.1 | Loop all pairs or reject `symLen > 4` | Yes |
| TASK-O2 | IMPROVEMENT | `ohttp.dart:51`, `ohttp.dart:82-85` | RFC 9458 §4.1 | Consolidate to one exception type (e.g., `OhttpUnsupportedSuiteException`) | Yes |
| TASK-O3 | IMPROVEMENT | `ohttp.dart:148-150` | RFC 9458 §4.3 | Expand inline comment to cite RFC §4.3 AAD wording and Go reference | Yes |
| TASK-O4 | MAINTENANCE | `ohttp.dart:192-207` | RFC 9458 §4.4 | Add warning comment at `:193`, `:196`, `:203` against labeled-HKDF substitution | Yes |
| TASK-O5 | HIGH | `ohttp.dart:217-225` | RFC 9458 §4.4 | Wrap `aesGcm.decrypt()` in try/catch; rethrow as `OhttpAuthenticationException` | Yes |
| TASK-O6 | HIGH | `ohttp.dart:104-120`, `ohttp.dart:162-166` | RFC 9458 §4.3 | Explicit zero-fill of `enc` and `exportedSecret` before reference release | Yes |
| TASK-O7 | HIGH | `test/ohttp_test.dart` | RFC 9458 §4.3, §4.4 | Add 4 test groups: round-trip, auth-failure, multi-suite, truncated-ciphertext | Yes |

All 7 draft tasks are present in both `research.md` and `tasks.md`. Each has a title, severity, file:line, RFC section, and a concrete remediation direction. Every task is independently scoped (no task depends on another in-flight Phase 3 task to complete first; TASK-O7 test scenario #2 references the future `OhttpAuthenticationException` from TASK-O5 but correctly scopes it as a "future" dependency pointing to the post-TASK-O5 state, which is acceptable for a test task).

Cross-phase references are present where required:
- TASK-O2 cites Phase 1 F-3 and Phase 2 R-3 (exception anti-pattern)
- TASK-O6 cites Phase 1 F-2 (zeroization absence in `HpkeSenderContext`)
- TASK-O7 cites Phase 1 F-6/F-7 and Phase 2 R-6 (test-gap pattern)

**Check 5: PASS**

---

## Check 6: Read-only constraint — no modifications to `lib/` or `test/`

**Criterion (from `phase-3/prd.md` Constraints §1 and Success / Metrics "No code changes"):** `git diff HEAD -- lib/ test/` must be empty at the end of Phase 3.

Verification: The `review.md` Phase 3 section states at line 507: "git diff HEAD -- lib/ test/ produces no output. The constraint is honored. Status output shows only the untracked specs/.current/AW-2865/phase-3/ directory and the existing review.md being extended. No source under lib/, test/, pubspec.yaml, or example/ is modified."

Git status at the start of this session confirms only `specs/.current/AW-2865/phase-3/` is untracked. No file under `lib/` or `test/` appears in the modification list.

**Check 6: PASS**

---

## Check 7: Review verdict is APPROVED, two Important items noted

**Criterion:** `review.md` Phase 3 section must carry an APPROVED verdict; any blocking issues would require remediation before this QA can pass.

The `review.md` Phase 3 section (lines 341–532) states:

> "APPROVED with two Important items. Phase 3 meets all PRD success criteria. ... No blocking findings."

Both Important items are non-blocking and explicitly deferred to pre-Iteration-7 reconciliation:

**I-1 — Severity inconsistency on O-7 between PRD Risks (HIGH) and research/tasks/plan (MAINTENANCE/IMPROVEMENT).**
- PRD §Risks table row for the plain-HKDF maintenance hazard (prd.md:122) classifies it as HIGH.
- research.md Limitations & Risks table O-7 row and TASK-O4 classify it as MAINTENANCE (IMPROVEMENT).
- plan.md §Severity policy (plan.md:92) defines MAINTENANCE as a sub-category of IMPROVEMENT.
- The review recommends reconciling to MAINTENANCE (IMPROVEMENT) with a PRD edit before Iteration 7.
- **QA impact:** The inconsistency is in the documentation only; the underlying finding is a POSITIVE verdict (code is compliant). The MAINTENANCE tier is the more defensible position and is correctly documented across research, tasks, and plan. This QA report records the inconsistency as a known open item from the review but does not escalate it to a blocker.

**I-2 — MAINTENANCE severity tier is introduced in plan.md but absent from vision §2 and the PRD's inherited severity model.**
- The plan explicitly defines and documents the fourth tier.
- **QA impact:** Same as I-1 — documentation gap only, no correctness or completeness risk.

Both items are flagged here for the Iteration 7 reviewer but do not affect phase close.

**Check 7: PASS (with two noted review items)**

---

## Check 8: Required cross-phase references are present

**Criterion (from `phase-3/plan.md` §Acceptance / Close Criteria item 6):** Cross-phase references must be present for O-4 (exception inconsistency), O-10 (zeroization), and O-11 (test-gap). O-3 is the parser-guard reassessment which implicitly references the vision assumption rather than a prior-phase finding.

| Finding | Required cross-phase reference | Present? | Cited IDs |
|---|---|---|---|
| O-4 (FormatException vs UnsupportedError) | Phase 1 F-3, Phase 2 R-3 | Yes | research.md explicitly cites "same inconsistent exception-type anti-pattern as Phase 1 F-3 and Phase 2 R-3" |
| O-10 (zeroization absence) | Phase 1 F-2 | Yes | research.md explicitly states "extends the Phase 1 zeroization finding (F-2) to the OHTTP layer" |
| O-11 (test-suite gaps) | Phase 1 F-6/F-7, Phase 2 R-6 | Yes | research.md explicitly cites "same test-gap pattern recurs at every layer. Phase 1 documented it as F-6 and F-7; Phase 2 documented it as R-6." |
| O-2 (silent multi-suite drop) | None required — new Layer 3 finding | Yes (explicit disclaimer) | research.md states "independent instance of the silent-ignore pattern documented at the BHTTP layer in Phase 2. No direct Phase 2 finding covers this; it is a new gap at Layer 3." |

All required cross-phase references are present and cite the correct prior-phase finding IDs.

**Check 8: PASS**

---

## Positive Scenarios Verified

The following scenarios from `phase-3/prd.md` produced positive verdicts confirmed in `research.md`:

| PRD Scenario | Finding | Verdict | Key evidence |
|---|---|---|---|
| Scenario 1 (KeyConfig wire-layout) | O-1 | COMPLIANT | Parser reads all six fields from correct byte offsets; four length guards present and correctly ordered |
| Scenario 4 (HPKE info string) | O-5 | COMPLIANT | `_buildHpkeInfo` at `ohttp.dart:232-245` produces `utf8("message/bhttp request") \|\| 0x00 \|\| 7-byte header` = 30 B; matches RFC 9458 §4.3 exactly |
| Scenario 6 (plain HKDF compliant) | O-7 | COMPLIANT | `HpkeSender.hkdfExtract`/`hkdfExpand` calls at `ohttp.dart:193`, `196-200`, `203-207` are plain RFC 5869; salt = enc(32 B) \|\| responseNonce(16 B) matches §4.4; nonce 16 B = max(Nk=16, Nn=12) matches §4.4 |
| Scenario 6 (response empty AAD) | O-8 | COMPLIANT | `aad: []` at `ohttp.dart:219-223`; RFC 9458 §4.4 specifies empty AAD on response decryption |
| Scenario 2 (RangeError not present) | O-3 | Vision assumption corrected | Guard at `ohttp.dart:63` catches both `symLen < 4` and `data.length < offset + symLen` before any list access; no `RangeError` escapes |

---

## Negative Scenarios and Edge Cases Verified

The following scenarios produced non-positive findings:

| PRD Scenario | Finding | Severity | Confirmed behavior |
|---|---|---|---|
| Scenario 1 (multi-suite drop) | O-2 | HIGH | `ohttp.dart:67-71` reads exactly the first KDF+AEAD pair and returns; `symLen > 4` bytes are present in buffer (guard at `:63` confirms it) but are never examined; no loop, no rejection |
| Scenario 3 (exception type inconsistency) | O-4 | IMPROVEMENT | `parse()` `:51` throws `FormatException`; `validate()` `:82-85` throws `UnsupportedError`; both express "unsupported KEM" but force two-type catch clause |
| Scenario 5 (empty AAD intentional deviation) | O-6 | IMPROVEMENT | `ctx.seal(Uint8List(0), binaryRequest)` at `:149`; RFC §4.3 wording suggests header-as-AAD; deviation is intentional and documented in CLAUDE.md; no confidentiality/integrity loss |
| Scenario 6 (maintenance hazard) | O-7 | MAINTENANCE | `hkdfExtract`/`hkdfExpand` are public static methods re-exported at `lib/ohttp_dart.dart:10`; future substitution with `LabeledExtract`/`LabeledExpand` would silently break decryption |
| Scenario 7 (SecretBoxAuthenticationError) | O-9 | HIGH | `aesGcm.decrypt()` at `:219` throws `SecretBoxAuthenticationError` (type from `package:cryptography`); not re-exported by `lib/ohttp_dart.dart`; callers must add direct `package:cryptography` import to catch specifically |
| Scenario 8 (zeroization absence) | O-10 | HIGH | `OhttpEncapsulateResult.enc` (32 B) and `exportedSecret` (16 B) are `final Uint8List` fields with no zero-fill before GC eligibility; `exportedSecret` is the symmetric key for response decryption; AOT is the higher-risk deployment target |
| Scenario 9 (test coverage) | O-11 | HIGH | `test/ohttp_test.dart` missing: round-trip test, authentication-failure test, multi-suite KeyConfig test, truncated-ciphertext test |

---

## Automated Tests Coverage

**Current state (read-only audit — no tests were added or changed):**

`test/ohttp_test.dart` provides:
- `OhttpKeyConfig.parse`: happy path, too-short data, unsupported KEM, short symmetric section (4 cases)
- `OhttpKeyConfig.validate`: supported suite, unsupported KEM/KDF/AEAD (4 cases)
- `ohttpEncapsulate`: one structural test (byte count check)
- `ohttpDecapsulate`: two short-response rejection tests (`FormatException` path)

**Gaps documented (TASK-O7, HIGH):**
1. Round-trip encrypt-then-decrypt test confirming plaintext recovery
2. Authentication-failure test — valid-length response with corrupted ciphertext, confirming the future `OhttpAuthenticationException` throw path
3. Multi-suite KeyConfig test (`symLen = 8`, supported first pair) documenting the silent-drop behavior as a regression guard for TASK-O1
4. Truncated-ciphertext test confirming `FormatException` from `ohttp.dart:212-213`

---

## Manual Checks Needed

All checks in this phase are manual (read-only audit). The following verifications are recommended before Iteration 7 compiles task files:

1. **I-1 reconciliation:** Update PRD `phase-3/prd.md:122` Risks table to read "IMPROVEMENT (maintenance hazard)" (matching research.md O-7 severity) or update research.md to read "HIGH" (matching the PRD Risks table). Propagate to plan.md §Severity policy and tasks.md TASK-O4 consistently.
2. **I-2 reconciliation:** Either fold MAINTENANCE into IMPROVEMENT with a parenthetical qualifier throughout Phase 3 artifacts, or add a one-line note to PRD §Resolved Questions explicitly admitting the fourth tier.
3. **N-3 loose end:** The O-3 sub-improvement (splitting the `'Invalid symmetric algorithms section'` error message at `ohttp.dart:64` into two distinct messages) is documented in research.md but has no corresponding draft task. Either add TASK-O8 ("Split ambiguous error message at `ohttp.dart:63-64`") or explicitly record it as "declined."
4. **Phase 5 forward-reference:** The `testKeyPair` parameter on `ohttpEncapsulate` (`ohttp.dart:134`) is deferred to Phase 5 via research §New Technical Questions #4. A note in `specs/.current/AW-2865/tasklist.md` Phase 5 row would prevent this from falling through the cracks at Iteration 7.

---

## Phase-Specific Risk Zone

| Risk | Severity | Source | Status |
|---|---|---|---|
| Gateway advertising preferred suite second → client silently uses wrong suite (cipher-suite downgrade) | HIGH | O-2 | Open — TASK-O1 filed |
| `SecretBoxAuthenticationError` from `package:cryptography` leaks into public API surface; wallets must import internal package to catch | HIGH | O-9 | Open — TASK-O5 filed |
| `exportedSecret` (response AEAD key) persists in heap until GC; AOT memory-dump risk | HIGH | O-10 | Open — TASK-O6 filed |
| No round-trip or authentication-failure test; correctness of decap path untested | HIGH | O-11 | Open — TASK-O7 filed |
| Exception type inconsistency (`FormatException` vs `UnsupportedError`) for unknown-KEM condition | IMPROVEMENT | O-4 | Open — TASK-O2 filed |
| Empty-AAD deviation from RFC §4.3 not fully documented inline | IMPROVEMENT | O-6 | Open — TASK-O3 filed |
| Plain-HKDF at `ohttp.dart:192-207` could be silently broken by future labeled-HKDF unification | MAINTENANCE | O-7 | Open — TASK-O4 filed |
| Severity inconsistency: PRD Risks classifies O-7 hazard as HIGH; research/plan/tasks classify it MAINTENANCE | Documentation | review I-1 | Open — pre-Iteration-7 action item |
| MAINTENANCE tier absent from PRD severity model | Documentation | review I-2 | Open — pre-Iteration-7 action item |

No BLOCKER findings were identified at the OHTTP layer. The implementation matches RFC 9458 §4.1, §4.3, and §4.4 on all four inspected positive-verdict paths (O-1, O-5, O-7 core verdict, O-8).

---

## Final Verdict

**PASS — RELEASE WITH RESERVATIONS**

Phase 3 is complete. All 11 tasks are checked, all 11 findings are filed with exact line citations and RFC section references, all 7 draft tasks are independently actionable, and the read-only constraint is confirmed satisfied by both the review agent and git status evidence.

The phase is ready to close. Iteration 7 may import TASK-O1 through TASK-O7 into Jira immediately.

**Two pre-Iteration-7 documentation actions are recommended (not blocking):**
- Reconcile the O-7 severity between PRD Risks (HIGH) and research/tasks/plan (MAINTENANCE) — pick one label and apply consistently.
- Either fold the MAINTENANCE tier into IMPROVEMENT with a parenthetical qualifier, or add a one-line PRD note admitting the fourth tier.

**Four HIGH-severity items** (O-2, O-9, O-10, O-11) should be remediated before wallet production use.
**Two IMPROVEMENT items** (O-4, O-6) and **one MAINTENANCE item** (O-7) are recommended backlog work.
No BLOCKER was identified.
