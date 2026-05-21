# Phase 1 Review — AW-2865 HPKE Layer Audit

**Subject:** Phase 1 deliverables for ticket AW-2865 (HPKE layer audit against RFC 9180).
**Reviewer:** Code review agent (read-only audit verification).
**Reviewed artifacts:**
- `specs/.current/AW-2865/phase-1/prd.md`
- `specs/.current/AW-2865/phase-1/plan.md`
- `specs/.current/AW-2865/phase-1/research.md`
- `specs/.current/AW-2865/phase-1/tasks.md`
- `specs/.current/AW-2865/vision.md` (reference)
- `lib/src/hpke.dart` (audited subject, read-only verification)

---

## Overall Verdict

**APPROVED.** Phase 1 meets all PRD success criteria. The research output is well-structured, every finding is grounded in concrete file:line references, severities are assigned consistently with the vision §2 model and the resolved-questions table, and the read-only constraint is honored.

No blocking findings. Two important observations and a handful of nice-to-have improvements are noted below for optional polish.

---

## Acceptance criteria verification

| Criterion | Status | Evidence |
|---|---|---|
| All RFC 9180 sections covered (§4, §4.1, §5.1, §5.2, §5.3) | PASS | `research.md` lines 42–190 contain a dedicated verdict block for each section with quoted RFC pseudo-code and a bolded "Verdict" line. No section is skipped. |
| Scrutiny-table rows from vision §4 (HPKE) addressed | PASS | Single-suite hard-coding addressed in `research.md` Limitations & Risks #1 (lines 260). Sequence-number overflow addressed by F-3, F-4, F-6. `testKeyPair` injection addressed by F-1. |
| Severity assigned to every finding | PASS | All 7 findings (F-1..F-7) carry exactly one of HIGH / IMPROVEMENT. Distribution matches user-reported counts (2 HIGH, 5 IMPROVEMENT, 0 BLOCKER). |
| No code changes (`git diff HEAD -- lib/ test/` empty) | PASS | Verified empty. Only modifications are to `specs/.current/.active_ticket` and `specs/.current/AW-2865/phase-1/tasks.md`; `prd.md`, `plan.md`, `research.md` are untracked spec files. No file under `lib/`, `test/`, `pubspec.yaml`, or `example/` is modified. |
| Each draft task independently actionable | PASS | TASK-1 through TASK-7 each carry a single concrete file:line target, one RFC citation, and concrete remediation directions. No task depends on another in-flight Phase 1 task. |

---

## Line-reference spot-check against `lib/src/hpke.dart`

I re-verified the key line references in the findings table against the current `hpke.dart` source (314 lines):

| Reference | Verified content | Status |
|---|---|---|
| `hpke.dart:47–48` — `testKeyPair` optional parameter | Lines 47–48 contain `Uint8List info, {` and `SimpleKeyPairData? testKeyPair,` — correct. | PASS |
| `hpke.dart:71–79` — ephemeral keypair generation with `testKeyPair` branch | Lines 71–79 contain the `_kemEncap` signature and the `if (testKeyPair != null)` branch — correct. | PASS |
| `hpke.dart:259–262` — `key`/`baseNonce`/`exporterSecret` fields | Lines 259–262 contain the four `final Uint8List` field declarations — correct. | PASS |
| `hpke.dart:275–277` — `_seq` overflow guard with `StateError` | Lines 275–277 contain `if (_seq >= (1 << 32))` and `throw StateError('HPKE message limit reached');` — correct. | PASS |
| `hpke.dart:281–284` — manual XOR of 4 bytes into nonce tail | Lines 281–284 contain the 4 byte-level XOR operations into `nonce[length - 1..4]` — correct. | PASS |
| `hpke.dart:225` / `hpke.dart:235` — public `hkdfExtract` / `hkdfExpand` | Confirmed both are `static Future<Uint8List>` with no `@internal` annotation — correct. | PASS |
| `hpke.dart:306–312` — `export` calls `_labeledExpand` with `'sec'` | Confirmed — correct. | PASS |

All findings' line ranges resolve to the claimed code. No drift between research and source.

---

## Findings (priority taxonomy)

### Blocking
None.

### Important
None that affect the audit deliverable itself. The findings in `research.md` are the actionable items; they will be filed as future Jira tickets per the workflow.

### Nice-to-have (audit-quality polish — not required for phase close)

1. **F-4 nonce-XOR description could clarify the semantic equivalence claim explicitly in the finding row, not only the section verdict.**
   The §5.2 verdict (research line 159) correctly notes that "for any `_seq < 2^32`, the result is identical to `I2OSP(_seq, 12) XOR base_nonce` since the leading 8 bytes of `I2OSP(_seq, 12)` are zero." The finding F-4 row (research line 251) describes the gap as a problem but does not repeat that the current behavior is RFC-equivalent within the guarded range. A reader who jumps directly to the findings table without reading the verdicts could misread F-4 as a correctness defect rather than a future-proofing concern. Consider adding a one-line clarifier in F-4: "for `_seq < 2^32` the truncated XOR is bitwise equivalent to the full 12-byte XOR".

2. **New Technical Question #4 (`export()` length validation) is left in limbo.**
   Research lines 282 raise the `L <= 255 * Nh` RFC 5869 §2.3 bound issue but do not promote it to either F-8 or explicitly defer it to Phase 3. The plan §"Open Questions" item 4 (plan lines 195) acknowledges it as "candidate Phase 1 IMPROVEMENT addendum or Phase 3 finding (currently filed under 'New Technical Questions' pending a decision on whether to add F-8)." This decision is left open. Recommendation: either promote it now to F-8 / TASK-8 (IMPROVEMENT) so it has a stable ID for downstream reference, or explicitly defer it to Phase 3 with a note that "export length validation" is a Phase 3 item. Leaving it in "New Technical Questions" risks it being lost during Jira import.

3. **F-7 severity rationale could be tightened.**
   F-7 is labeled HIGH with the parenthetical "covered as a Phase 3 finding but noted here as a HPKE-layer test gap." The Important-priority severity is justified for the OHTTP-level empty-AAD integration gap, but the finding sits in a Phase 1 deliverable that explicitly excludes the OHTTP layer (PRD constraint #4, research line 23). The plan §Dependencies (line 183) correctly flags F-7 as cross-boundary. Consider one of: (a) move F-7 into a "Cross-Phase References" sub-table separate from the Phase 1 findings table, so it does not count toward Phase 1's actionable Jira items; (b) downgrade Phase 1's view of it to IMPROVEMENT with a pointer to Phase 3 where the HIGH classification will land. This is a presentation choice, not a correctness issue.

4. **Plan §Definition of Done item 4 references a `phase-1/qa.md` file that does not yet exist.**
   The plan (line 211) says "Phase 1 QA report (`phase-1/qa.md`) confirms each PRD success criterion is met." This review serves a similar function but is being written to `specs/.current/AW-2865/review.md` per the caller's instructions, not to `phase-1/qa.md`. If the workflow distinguishes between a code-review-agent output and a phase-QA artifact, the QA file may still need to be produced separately to satisfy DoD #4. Flagging for the operator's awareness — not a blocker for the review itself.

---

## Cross-checks against the PRD scenarios

| Scenario | Mapped task | Mapped finding(s) | Status |
|---|---|---|---|
| S1 — Labeled KDF correctness | 1.2 | §4 verdict (matches RFC) | Verified |
| S2 — KEM Encap correctness | 1.3 | §4.1 verdict (matches RFC) | Verified |
| S3 — Key schedule | 1.4 | §5.1 verdict (matches RFC) | Verified |
| S4 — Nonce derivation | 1.5 | §5.2 verdict + F-4 | Verified |
| S5 — Export | 1.6 | §5.3 verdict (matches RFC) | Verified |
| S6 — Structural risks (seq overflow / testKeyPair / zeroization) | 1.7, 1.8, 1.9 | F-1, F-2, F-3 | Verified |
| S7 — Draft task compilation | 1.10 | TASK-1..TASK-7 | Verified |

Every PRD scenario has a corresponding research artifact and (where applicable) a draft Jira task entry.

---

## Read-only constraint verification

```
$ git diff HEAD -- lib/ test/ pubspec.yaml example/
(empty)

$ git status --short
 M specs/.current/.active_ticket
 M specs/.current/AW-2865/phase-1/tasks.md
?? specs/.current/AW-2865/phase-1/plan.md
?? specs/.current/AW-2865/phase-1/prd.md
?? specs/.current/AW-2865/phase-1/research.md
```

All modifications are confined to `specs/.current/AW-2865/phase-1/` and the active-ticket pointer. PRD success criterion "No code changes" is satisfied.

---

## Severity-discipline check

Per resolved-questions table (research lines 30–36):

| Concern | Resolved severity | Applied in findings | Consistent? |
|---|---|---|---|
| `testKeyPair` injection | HIGH | F-1: HIGH | Yes |
| Key-material zeroization | IMPROVEMENT | F-2: IMPROVEMENT | Yes |
| `_seq` overflow (single-use context) | IMPROVEMENT | F-3, F-4, F-6: IMPROVEMENT | Yes |

No severity drift. The "Resolved Questions" table successfully fixes the PRD's open questions before they could cause inconsistency.

---

## Summary for the engineering lead

- Audit deliverable is **complete and consistent** with the PRD acceptance criteria.
- All seven findings carry the data required for Jira import (title, severity, file:line, RFC section, description, remediation direction).
- Two HIGH-severity items (F-1 `testKeyPair` injection and F-7 empty-AAD integration gap) and five IMPROVEMENT items (F-2..F-6) are recommended for the follow-up backlog.
- No BLOCKER was identified in the HPKE layer. The implementation matches RFC 9180 for the fixed cipher suite within the single-use sender context constraint.
- Phase 1 is ready to close pending (optional) production of `phase-1/qa.md` per the plan's Definition of Done, and (optional) resolution of New Technical Question #4 (`export()` length bound) as either F-8 or an explicit Phase 3 deferral.

---

## Files referenced (absolute paths)

- /Users/comrade77/Documents/Performix/Projects/ohttp_dart/specs/.current/AW-2865/phase-1/prd.md
- /Users/comrade77/Documents/Performix/Projects/ohttp_dart/specs/.current/AW-2865/phase-1/plan.md
- /Users/comrade77/Documents/Performix/Projects/ohttp_dart/specs/.current/AW-2865/phase-1/research.md
- /Users/comrade77/Documents/Performix/Projects/ohttp_dart/specs/.current/AW-2865/phase-1/tasks.md
- /Users/comrade77/Documents/Performix/Projects/ohttp_dart/specs/.current/AW-2865/vision.md
- /Users/comrade77/Documents/Performix/Projects/ohttp_dart/lib/src/hpke.dart
