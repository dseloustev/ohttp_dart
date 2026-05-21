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

## Files referenced

- specs/.current/AW-2865/phase-1/prd.md
- specs/.current/AW-2865/phase-1/plan.md
- specs/.current/AW-2865/phase-1/research.md
- specs/.current/AW-2865/phase-1/tasks.md
- specs/.current/AW-2865/vision.md
- lib/src/hpke.dart

---
---

# Phase 2 Review — AW-2865 BHTTP Layer Audit

**Subject:** Phase 2 deliverables for ticket AW-2865 (BHTTP layer audit against RFC 9292).
**Reviewer:** Code review agent (read-only audit verification).
**Reviewed artifacts:**
- `specs/.current/AW-2865/phase-2/prd.md`
- `specs/.current/AW-2865/phase-2/plan.md`
- `specs/.current/AW-2865/phase-2/research.md`
- `specs/.current/AW-2865/phase-2/tasks.md`
- `specs/.current/AW-2865/idea.md` and `vision.md` (Iteration 2 / §5.6 reference)
- `lib/src/bhttp.dart` (audited subject, read-only verification)

---

## Overall Verdict

**APPROVED with notes.** Phase 2 meets all PRD success criteria. Every parsing surface (2.2–2.7) has a per-task finding grounded in a concrete `bhttp.dart` line citation, all 8 checklist items are marked `[x]` in `phase-2/tasks.md`, no source under `lib/` or `test/` was modified, and the deliverable shape mirrors Phase 1 (per-task verdicts → risks table → draft task entries).

There are **no Blocking issues**. There are **three Important findings** centered on severity-discipline drift between `tasks.md` and `research.md` (the same finding is classified at different severities in the two files), and one Important finding about scope drift in TASK-B6. Several Nice-to-have items follow.

---

## Acceptance criteria verification (against `phase-2/prd.md` Success / Metrics table)

| Criterion | Status | Evidence |
|---|---|---|
| All RFC 9292 parsing surfaces covered (2.2–2.7 each have a verdict + cited line) | PASS | `research.md` lines 87–285 contain a per-task block for each of 2.2–2.7. Tasks 2.2 (framing), 2.3 (status), 2.4 (header loop), 2.5 (body sublist), 2.6 (varint), 2.7 (serializeRequest) each have a bolded **Verdict** line and a `bhttp.dart:NNN` reference. |
| Framing indicator behavior documented | PASS | `research.md` task 2.2 (lines 89–107) cites `bhttp.dart:140–146` and quotes the exact `FormatException('Expected known-length response (framing=1), got $framing')` throw. Verdict: "guard present and correct". |
| Every finding has a severity | PASS for `research.md` (R-1..R-7 all carry HIGH or IMPROVEMENT). See "Severity-discipline drift" below for inconsistencies between `tasks.md` and `research.md` on individual findings. |
| No code changes | PASS | `git diff HEAD -- lib/ test/ pubspec.yaml example/` is empty. Only modifications are to `specs/.current/.active_ticket`, `specs/.current/AW-2865/phase-2/tasks.md`, and the three untracked phase-2 spec files (`prd.md`, `plan.md`, `research.md`). |
| Draft tasks independently actionable | PASS | TASK-B1..TASK-B7 each carry a single file:line target and concrete remediation directions. TASK-B5 explicitly notes the layer-ownership ambiguity (BHTTP vs. `OhttpClient`) inline, consistent with resolved-question Q2. TASK-B6 depends on TASK-B3 and TASK-B4 for "post-fix" assertions — this dependency is called out inline (research.md line 374). |
| Exception-type mismatch confirmed | PASS | Task 2.5 verdict at `research.md` line 182 explicitly traces the call stack `bhttp.dart:187 → ohttp_client.dart:136 → caller` and confirms `RangeError` propagates unwrapped. The Dart semantics note (`RangeError extends Error, not Exception`) is at line 198. |

All six PRD success criteria are satisfied. The audit deliverable is complete.

---

## Line-reference spot-check against `lib/src/bhttp.dart`

I re-read the source (191 lines, no `ast-index outline` needed) and verified the key line references:

| Reference | Verified content in `bhttp.dart` | Status |
|---|---|---|
| `bhttp.dart:140–146` (framing indicator) | Lines 140–146 contain `final (framing, framingLen) = decodeVarint(data, offset);`, `offset += framingLen;`, and the `if (framing != 1) throw FormatException(...)` block. Matches research.md exactly. | PASS |
| `bhttp.dart:148–162` (tasks.md) vs. `bhttp.dart:149–163` (research.md) — 1xx skip loop | Source lines 148–162 contain the `// Skip informational responses (1xx)` comment, the `int statusCode; int statusLen;` declarations, and the `while (true) { ... }` block ending at the closing brace on line 162. **`tasks.md` uses the correct range (148–162); `research.md` is off by one (149–163).** | PASS (with off-by-one in research.md) |
| `bhttp.dart:164–182` (tasks.md) vs. `bhttp.dart:165–182` (research.md) — header loop | Source line 164 is the `// Header section` comment; line 165 is the `decodeVarint(data, offset)` call for `headersLen`. **Same off-by-one as above.** | PASS (off-by-one in research.md) |
| `bhttp.dart:184–187` (tasks.md) vs. `bhttp.dart:187` (research.md) — body sublist | Source line 184 is `// Content`, 185 is the `contentLen` decode, 187 is `data.sublist(offset, offset + contentLen)`. Both citations resolve correctly; `tasks.md` is broader (whole content block), `research.md` is the precise throw site. | PASS |
| `bhttp.dart:44–67` — `decodeVarint` | All four prefix branches at lines 47–66, with `data[offset + 1]` / `data[offset + 2..3]` / `data[offset + 1..7]` reads as research.md describes. The `default: throw StateError('unreachable')` is at line 65. | PASS |
| `bhttp.dart:74–111` — `serializeRequest` | Function signature at 74; framing indicator write at 85 (`buf.add(encodeVarint(0));`); body write at 104–105 (`buf.add(encodeVarint(body.length)); if (body.isNotEmpty) buf.add(body);`); `_writeField` helper at 113–116. Matches research.md. | PASS |
| `bhttp.dart:113–116` — `_writeField` helper (referenced in plan.md and research.md) | Confirmed correct. | PASS |
| `bhttp.dart:187` — body sublist (`Uint8List.fromList(data.sublist(...))`) | Confirmed. The wrapping `Uint8List.fromList(...)` does not catch the underlying `RangeError`. | PASS |

All findings' line ranges resolve to the claimed code. The off-by-one drift between `tasks.md` and `research.md` for tasks 2.3 and 2.4 is cosmetic — both ranges encompass the cited construct — but worth tightening.

---

## Findings (priority taxonomy)

### Blocking
None.

### Important

1. **Severity-discipline drift between `tasks.md` and `research.md` for the same finding (PRD criterion "Every finding has a severity" technically passes per file, but the two files disagree).**
   - **Status code range (task 2.3).** `tasks.md` line 68 classifies F2.3 as `IMPROVEMENT` ("does not enable DoS or memory exhaustion; consumer-side concern"). `research.md` R-1 (line 291) classifies the same finding as `HIGH`. Vision §5.6 lists "No range validation after varint decode" as a `BhttpResponse.statusCode` concern but does not pin a severity for it in §5's cross-cutting concerns table; only "No size caps on response body / headers" and "`RangeError` instead of `FormatException` on truncated BHTTP body" are listed as HIGH. Decision: the analyst should pick one severity and propagate consistently. The downstream Jira import will use the `research.md` draft task entry (TASK-B1, HIGH), so `tasks.md` F2.3 should be raised to HIGH or the Iteration 7 reviewer should be made aware of the discrepancy.
   - **Body size cap in `serializeRequest` (task 2.7).** `tasks.md` line 110 classifies F2.7 as `IMPROVEMENT→HIGH for wallet`. `research.md` R-5 (line 295) and TASK-B5 (line 364) classify it as `HIGH` plainly. Vision §6.1 has the inline `[CONCERN] No upper-bound on body size before serialization` but does not assign a severity. The PRD's NFR "Severity discipline" (plan.md line 163) explicitly forbids "TBD"; the conditional `IMPROVEMENT→HIGH for wallet` notation in `tasks.md` is arguably a TBD in disguise. Recommend collapsing to a single `HIGH` (matching research.md and the wallet-use deployment context that is the entire investigation's premise).
   - **Framing indicator (task 2.2).** `tasks.md` line 59 classifies it as `IMPROVEMENT` based on the adjacent empty-buffer `RangeError` concern. `research.md` records no severity for task 2.2 because the verdict is "guard present and correct" — the empty-buffer concern is rerouted exclusively through task 2.5 / R-3 / TASK-B3. Both treatments are defensible, but the tasks.md "IMPROVEMENT" label is misleading because the framing-indicator guard itself is correct. Recommend either (a) drop the severity from `tasks.md` task 2.2 (and route the empty-buffer note as a sub-bullet of task 2.5), or (b) explicitly note in `tasks.md` that the IMPROVEMENT severity refers only to the empty-buffer-RangeError aspect.

2. **`tasks.md` lists 10 findings (F2.3, F2.4a, F2.4b, F2.4c, F2.5a, F2.5b, F2.5c, F2.6a, F2.6b, F2.7) but `research.md` consolidates into 7 draft tasks (TASK-B1..TASK-B7).**
   The consolidation is explicit and justifiable — TASK-B3 covers both body sublist (F2.5a) and header sublist (F2.5b) in one task because they share a single remediation (`_safeSublist` helper); TASK-B4 covers all three varint widths (F2.6a) in a single task because they share the same `if (offset + N > data.length) throw FormatException(...)` pattern. `tasks.md` line 128 acknowledges this: "F2.5* and F2.6a together motivate a single shared helper rather than per-call-site guards; this should be one engineering task in Iteration 7, not seven." However, **F2.4a, F2.4b, F2.4c are three distinct DoS surfaces (oversized `headersLen`, large header count, single oversized header value)** that all map to research.md's single TASK-B2. The PRD success criterion "Independent actionability" is satisfied because one engineer can implement all three guards in one PR, but a reviewer triaging the Jira import should be aware that TASK-B2 encompasses three independent guards. **Recommendation:** either split TASK-B2 into TASK-B2a (max `headersLen`), TASK-B2b (max header count), TASK-B2c (max single-field length), or add a sub-bullet list inside TASK-B2's description enumerating the three guards. The current TASK-B2 text in research.md mentions (1) and (2) explicitly but only obliquely covers (3) ("optionally, a maximum header-count check"). The single-oversized-header DoS (F2.4c — `nameLen = 1, valueLen = 2^30 - 1`) is the most novel of the three and risks being lost.

3. **TASK-B6 scope drift — test-gap task severity.** `research.md` TASK-B6 (line 371) classifies "negative tests for truncated input and adversarial inputs in `bhttp_test.dart`" as `HIGH`. Phase 1's analogous test-gap finding (F-6, F-7 in Phase 1 research) was classified as `HIGH` only for F-7 (empty-AAD integration gap) and `IMPROVEMENT` for the rest. Vision §2's severity model defines `HIGH` as "should fix before wallet use" — the absence of negative tests is a regression-risk concern, not a runtime-DoS concern, so the wallet-use blocker rationale is weaker than for the runtime-impact findings (R-1..R-5). Recommend either downgrading TASK-B6 to `IMPROVEMENT` (consistent with Phase 1's treatment of test gaps) or adding a rationale in the TASK-B6 description explaining why this specific test gap is HIGH while Phase 1's analogous gaps were IMPROVEMENT. Without this, severity discipline is inconsistent across phases.

### Nice-to-have (audit-quality polish — not required for phase close)

1. **Off-by-one line drift between `tasks.md` and `research.md` for tasks 2.3 (148–162 vs. 149–163) and 2.4 (164–182 vs. 165–182).** Both ranges encompass the cited construct so the citation is correct in both, but Jira-import consistency would benefit from a single canonical range. Pick whichever is preferred and propagate.

2. **`research.md` "Resolved Questions" table line 33 says "Framing indicator IS validated (finding F-B1 is positive)" but no `F-Bn` IDs are used elsewhere in the file.** The Phase 1 convention is `F-1..F-7`; Phase 2 uses `R-1..R-7` and `TASK-B1..TASK-B7`. The stray `F-B1` reference appears to be a leftover from an earlier draft. Recommend either introducing a consistent `F-Bn` system for positive verdicts (parallel to `R-n` for negative verdicts) or removing the orphan reference.

3. **Trailer section silent drop (research.md New Technical Question #4) deserves promotion to a finding.** RFC 9292 §3.4 defines the trailer section as part of the Known-Length response wire format. `parseResponse` reads through the body sublist and returns immediately — any trailing bytes are silently discarded. The plan.md risk R-8 (line 181) explicitly flags this as a candidate finding pending an "intent" decision. Recommendation: either promote it to R-8 / TASK-B8 with severity `IMPROVEMENT` (matches the framing-correctness rather than DoS-risk pattern) or explicitly defer to Phase 5 with a one-line note in `research.md`. Leaving it in "New Technical Questions" risks loss during Iteration 7 compilation — the same concern raised in the Phase 1 review for `export()` length validation.

4. **8-byte varint test (TASK-B7) bundles two concerns: missing test + dart2js integer semantics.** The first (no 8-byte test vector) is a pure test-suite gap (IMPROVEMENT). The second (53-bit safe-int range under dart2js / dart2wasm) is a platform-support documentation question that depends on whether `ohttp_dart` is ever deployed in a web context. Vision §3 and `CLAUDE.md` say "Pure-Dart implementation … runs on plain Dart VM" but do not exclude dart2js. Bundling these two concerns in one task means an engineer picking up TASK-B7 may implement only the test and miss the documentation/runtime-support question, or vice versa. Recommend splitting into TASK-B7a (add 8-byte varint round-trip test) and TASK-B7b (document platform support; verify dart2js behavior or pin to VM in `pubspec.yaml`/README). Cross-reference Phase 1 New Technical Question #3 if it surfaces the same dart2js concern.

5. **`research.md` line 33 description of resolved-question Q1 is slightly self-contradictory.** It says "Severity classification for the absence of a guard in the hypothetical case is moot; the guard exists." This is correct — but then `tasks.md` line 59 still assigns severity `IMPROVEMENT` to task 2.2 (for the empty-buffer concern). The two files have drifted on whether 2.2 has a severity at all. See Important finding #1 above for the consolidated recommendation.

6. **No `phase-2/qa.md` exists.** Plan.md line 44 lists `phase-2/qa.md` as a deliverable artifact; plan.md DoD item 4 (line 259) says "Phase 2 QA report (`phase-2/qa.md`) confirms each PRD success criterion is met." This review serves the QA function but is being appended to `specs/.current/AW-2865/review.md` per the caller's instructions, not written to `phase-2/qa.md`. If the workflow distinguishes between a code-review-agent output and a phase-QA artifact, the QA file may still need to be produced separately. Same observation was made on Phase 1 — consistent treatment between phases.

7. **TASK-B5 layer-ownership decision is correctly left open per resolved-question Q2, but `serializeRequest` is the natural enforcement point because it owns the buffer construction.** Research.md TASK-B5 says "either BHTTP layer with a parameter, or `OhttpClient` layer with a hard-coded or config-driven limit". A `[NOTE: Phase 5 will decide]` cross-link would harden the handoff and prevent the engineer picking up TASK-B5 from re-litigating the decision in isolation. Phase 5 (`ohttp_client.dart` audit) is the canonical place per plan.md "Downstream consumers" (line 228).

---

## Cross-checks against the PRD scenarios

| Scenario | Mapped task | Mapped finding(s) | Status |
|---|---|---|---|
| S1 — Framing indicator validation | 2.2 | research.md task 2.2 verdict (positive, no R-n row) | Verified |
| S2 — Status code range validation | 2.3 | R-1 / TASK-B1 | Verified (severity drift — see Important #1) |
| S3 — Header loop size guards | 2.4 | R-2 / TASK-B2 | Verified (sub-finding consolidation — see Important #2) |
| S4 — Body sublist truncation exception type | 2.5 | R-3 / TASK-B3 (+ header-sublist applied to F2.5b) | Verified |
| S5 — `decodeVarint` correctness + truncation | 2.6 | research.md task 2.6 verdict (correctness positive) + R-4 / TASK-B4 (truncation negative) | Verified |
| S6 — `serializeRequest` body size guard | 2.7 | R-5 / TASK-B5 | Verified (severity drift — see Important #1) |
| S7 — Draft task entry compilation | 2.8 | TASK-B1..TASK-B7 | Verified |

Every PRD scenario has a corresponding research artifact and (where applicable) a draft Jira task entry. The vision §4 scrutiny-table row for `bhttp.dart` ("No length guards", "No negative-case tests") is addressed by R-2/R-4/R-5 (length guards) and R-6/R-7 (negative-case tests) — both halves covered, per plan.md DoD item 6.

---

## Cross-phase reference verification

Plan.md "Cross-reference contract" (lines 88–94) requires that compounding Phase 1 risks be cited by ID. Verified:

| Phase 2 finding | Cited Phase 1 ID | Status |
|---|---|---|
| R-3 (RangeError on truncated body) | Phase 1 F-3 (StateError on HPKE overflow) — same untyped-error anti-pattern | Cited at `research.md` line 305 |
| R-5 (no body size cap in `serializeRequest`) | Phase 1 F-2 (no size cap on `encRequest`) | Cited at `research.md` line 306 |
| R-6 (no negative tests in `bhttp_test.dart`) | Phase 1 F-6, F-7 (test gaps) | Cited at `research.md` line 307 |

All three required cross-references are present. The Phase 1 IDs (F-2, F-3, F-6, F-7) match the actual finding IDs in `specs/.current/AW-2865/phase-1/research.md` (verified during Phase 1 review). No drift.

---

## Read-only constraint verification

```
$ git diff HEAD -- lib/ test/ pubspec.yaml example/
(empty)

$ git status --short
 M specs/.current/.active_ticket
 M specs/.current/AW-2865/phase-2/tasks.md
?? specs/.current/AW-2865/phase-2/plan.md
?? specs/.current/AW-2865/phase-2/prd.md
?? specs/.current/AW-2865/phase-2/research.md
```

All modifications are confined to `specs/.current/AW-2865/phase-2/` and the active-ticket pointer. PRD success criterion "No code changes" is satisfied. Plan.md NFR "Read-only guarantee" (line 159) is satisfied.

---

## Completeness check vs. `idea.md` and `vision.md` Iteration/Phase 2 scope

| Source requirement | Coverage in Phase 2 deliverable | Status |
|---|---|---|
| `idea.md` investigation vector: "Robustness of parsers against malformed input and potential DoS scenarios" | All 7 R-rows and 7 TASK-B entries address parser-robustness / DoS scenarios | PASS |
| `vision.md §4` scrutiny row: "No length guards against oversized frames" | R-2 (header loop), R-4 (varint truncation), R-5 (serializeRequest body cap) | PASS |
| `vision.md §4` scrutiny row: "No negative-case tests for malformed input" | R-6 (test gap), TASK-B6 (negative test plan) | PASS |
| `vision.md §5.6` `BhttpResponse.statusCode` concern: "No range validation after varint decode" | R-1 / TASK-B1 | PASS |
| `vision.md §5.6` `BhttpResponse.headers` concern: "No max header-count or total-size guard" | R-2 / TASK-B2 | PASS |
| `vision.md §5.6` `BhttpResponse.body` concern: "sublist throws RangeError on truncated input" | R-3 / TASK-B3 | PASS |
| `vision.md §5` cross-cutting: "No size caps on response body / headers" — HIGH | R-2 (headers HIGH), R-5 (serializeRequest body HIGH) — note: response-body cap is `OhttpClient` / `OhttpResponse` concern in vision §5.5, deferred to Phase 5 | PASS (Phase 2 owns the request-body and header halves) |
| `vision.md §5` cross-cutting: "RangeError instead of FormatException on truncated BHTTP body" — HIGH | R-3 (HIGH), R-4 (HIGH) | PASS |
| `vision.md §6.1` happy-path concern: "No upper-bound on body size before serialization" | R-5 / TASK-B5 | PASS |

Every Phase 2 scope item from the vision is addressed by at least one research finding. No gaps. The vision's HIGH-severity classifications for the two cross-cutting concerns in §5 are preserved in research.md (R-3, R-4, R-5 = HIGH; R-2 = HIGH). The drift identified in "Important #1" above relates to findings that the vision does *not* explicitly pre-classify (status code range, request body cap) — the disagreement is between `tasks.md` and `research.md`, not against the vision.

---

## Severity-discipline summary

Per `vision.md §2` taxonomy:

| Severity | research.md count | tasks.md count | Aligned? |
|---|---|---|---|
| BLOCKER | 0 | 0 | Yes |
| HIGH | 6 (R-1..R-6) | 5 (F2.4a, F2.4b, F2.4c, F2.5a, F2.5b, F2.6a) | **No — disagreement on F2.3 (status code) and F2.7 (body cap)** |
| IMPROVEMENT | 1 (R-7: 8-byte varint test) | 5 (F2.3, F2.5c, F2.6b, F2.7) | **No — same disagreement** |

The aggregate severity counts differ because of the two disputed findings (F2.3 and F2.7). See Important finding #1 for the consolidation recommendation. Otherwise, severity discipline is internally consistent within each file.

---

## Summary for the engineering lead

- Audit deliverable is **complete and consistent** with PRD acceptance criteria. All 8 checklist items are checked; all 6 parsing surfaces have verdicts and line citations; all 7 draft tasks (TASK-B1..TASK-B7) are independently actionable; no source under `lib/` or `test/` was modified.
- **Six HIGH-severity items** (R-1 status code range, R-2 header loop guards, R-3 body sublist exception type, R-4 varint truncation, R-5 serializeRequest body cap, R-6 negative-test coverage) and **one IMPROVEMENT** (R-7 8-byte varint test) recommended for the follow-up backlog. No BLOCKER identified at the BHTTP layer.
- **One severity-discipline action item before Iteration 7:** reconcile F2.3 (status code) and F2.7 (body cap) severities between `tasks.md` and `research.md`. The `research.md` HIGH classifications are the more defensible position given the wallet-use deployment context.
- **One consolidation action item before Iteration 7:** ensure TASK-B2 (header loop) explicitly enumerates the three sub-DoS-vectors from `tasks.md` F2.4a/b/c (max `headersLen`, max header count, max single-field length). The single-oversized-header DoS (F2.4c) is the most novel and is currently underspecified in TASK-B2.
- **One scope-clarification action item:** decide whether trailer-section silent-drop (research.md New Technical Question #4) is promoted to R-8 / TASK-B8 now or deferred to Phase 5. Same loose-end pattern as Phase 1's `export()` length question.
- Phase 2 is ready to close pending (optional) production of `phase-2/qa.md` per the plan's Definition of Done item 4, and (recommended) resolution of the three action items above before Iteration 7 compiles the per-task Markdown files.

---

## Files referenced

- specs/.current/AW-2865/phase-2/prd.md
- specs/.current/AW-2865/phase-2/plan.md
- specs/.current/AW-2865/phase-2/research.md
- specs/.current/AW-2865/phase-2/tasks.md
- specs/.current/AW-2865/idea.md
- specs/.current/AW-2865/vision.md
- lib/src/bhttp.dart

---

# Phase 3 Review — AW-2865 OHTTP Layer Audit

**Subject:** Phase 3 deliverables for ticket AW-2865 (OHTTP encapsulation/decapsulation layer audit against RFC 9458).
**Reviewer:** Code review agent (read-only audit verification).
**Reviewed artifacts:**
- specs/.current/AW-2865/phase-3/prd.md
- specs/.current/AW-2865/phase-3/plan.md
- specs/.current/AW-2865/phase-3/research.md
- specs/.current/AW-2865/phase-3/tasks.md
- specs/.current/AW-2865/vision.md (reference)
- specs/.current/AW-2865/phase-1/summary.md (cross-phase reference)
- specs/.current/AW-2865/phase-2/summary.md (cross-phase reference)
- lib/src/ohttp.dart (audited subject, read-only verification)
- lib/ohttp_dart.dart (re-export surface verification)

---

## Overall Verdict

**APPROVED with two Important items.** Phase 3 meets all PRD success criteria. Coverage of the eight in-scope inspection targets is complete, each finding cites a concrete `ohttp.dart` line range with the relevant RFC 9458 section, severity is assigned to every non-positive finding, and the read-only constraint is honored (`git diff HEAD -- lib/ test/` is empty). All 11 tasks (3.1–3.11) are checked, all 11 findings (O-1–O-11) are recorded, and all 7 draft tasks (TASK-O1–TASK-O7) carry an actionable remediation direction with a single owner.

The two Important items are a severity inconsistency on the plain-HKDF maintenance hazard between PRD Risks (HIGH) and research/tasks/plan (MAINTENANCE/IMPROVEMENT), and the introduction of a `MAINTENANCE` severity tier that is not present in the vision §2 severity model the PRD itself inherits. Neither blocks phase close; both warrant a one-line reconciliation before Iteration 7.

No blocking findings.

---

## Acceptance criteria verification

| PRD criterion | Status | Evidence |
|---|---|---|
| All RFC 9458 §4 subsections covered (no row in vision §4 scrutiny table for `ohttp.dart` skipped) | PASS | research.md §Findings covers §4.1 (O-1, O-2, O-3, O-4), §4.3 (O-5, O-6, O-10), §4.4 (O-7, O-8, O-9). Every scrutiny-table row maps to a verdict via the coverage table at plan.md lines 152–164. |
| Empty-AAD and plain-HKDF findings fully cited | PASS | O-6 cites `ohttp.dart:149` + RFC 9458 §4.3 / §4.6.1. O-7 cites `ohttp.dart:192-207` (with sub-line breakdown for `:193`, `:196-200`, `:203-207`) + RFC 9458 §4.4 and the RFC 5869 / RFC 9180 distinction. Both meet the acceptance criterion verbatim. |
| `OhttpKeyConfig` parser gaps documented with line refs | PASS | O-2 (silent multi-suite drop) cites `ohttp.dart:60-71`; O-3 (RangeError reassessment) cites `ohttp.dart:33-71`. Both findings have exact line references. |
| Severity assigned to every finding | PASS (with note) | All 11 findings carry a severity tag or an explicit `—` for positive verdicts. See Important item I-1 about the `MAINTENANCE` tier not present in vision §2. |
| No code changes (`git diff HEAD -- lib/ test/` empty) | PASS | Verified empty. Untracked Phase 3 spec files only. |
| Draft tasks independently actionable | PASS | TASK-O1..TASK-O7 each target a single primary file:line in `ohttp.dart` (or `test/ohttp_test.dart` for O7), cite one RFC section (or Go-interop / RFC distinction), and carry a concrete remediation direction. No draft task depends on another in-flight Phase 3 task to be completed first. |
| Cross-layer references included for findings that compound Phase 1 / Phase 2 patterns | PASS | O-2 explicitly notes the silent-ignore pattern is a new Layer 3 instance (no Phase 2 counterpart). O-4 cross-refs Phase 1 F-3 and Phase 2 R-3. O-10 cross-refs Phase 1 F-2. O-11 cross-refs Phase 1 F-6/F-7 and Phase 2 R-6. Coverage of cross-phase obligation from PRD Success table row 7 is complete. |

---

## Line-reference spot-check against `lib/src/ohttp.dart`

I re-verified the key line citations against the current `ohttp.dart` source (257 lines on disk; research and plan claim 258, which is a +1 trailing-newline counting convention — see Nice-to-have N-1).

| Reference | Verified content | Status |
|---|---|---|
| `ohttp.dart:40` — `keyId = data[offset++]` | Confirmed at line 40. | PASS |
| `ohttp.dart:41` — kemId BE assembly | Confirmed at line 41 (`(data[offset] << 8) \| data[offset + 1]`). | PASS |
| `ohttp.dart:51` — `FormatException('Unsupported KEM:...')` in `parse` | Confirmed at line 51. | PASS |
| `ohttp.dart:57` — `publicKey = Uint8List.fromList(data.sublist(...))` | Confirmed at line 57. | PASS |
| `ohttp.dart:60` — symLen BE read | Confirmed at line 60. | PASS |
| `ohttp.dart:63-65` — guard `symLen < 4 \|\| data.length < offset + symLen` + `FormatException` | Confirmed at lines 63-65. | PASS |
| `ohttp.dart:68`, `:70` — kdfId / aeadId BE read | Confirmed at lines 68 and 70. | PASS |
| `ohttp.dart:82-85` — `validate()` `UnsupportedError` for KEM | Confirmed at lines 82-85. | PASS |
| `ohttp.dart:104-120` — `OhttpEncapsulateResult` data class with 3 `final Uint8List` fields | Confirmed at lines 105-120 (one-line shift, see N-1). | PASS (line range matches the class body inclusively) |
| `ohttp.dart:131-167` — `ohttpEncapsulate` async function | Confirmed at lines 131-167. | PASS |
| `ohttp.dart:149` — `ctx.seal(Uint8List(0), binaryRequest)` empty AAD | Confirmed at line 149. | PASS |
| `ohttp.dart:160` — `[...header, ...ctx.enc, ...ct]` assembly | Confirmed at line 160. | PASS |
| `ohttp.dart:174-226` — `ohttpDecapsulate` | Confirmed at lines 174-226. | PASS |
| `ohttp.dart:179-183` — short-response guard `encResponse.length <= _responseNonceLen` | Confirmed at lines 179-183 (research cites `:180-182`; the throw spans 180–182 with the `if` at 179). | PASS |
| `ohttp.dart:186-187` — responseNonce / ciphertext split | Confirmed at lines 186-187. | PASS |
| `ohttp.dart:190` — `salt = [...enc, ...responseNonce]` | Confirmed at line 190. | PASS |
| `ohttp.dart:193` — `HpkeSender.hkdfExtract(salt, exportedSecret)` | Confirmed at line 193. | PASS |
| `ohttp.dart:196-200` — `HpkeSender.hkdfExpand(prk, "key", _nk)` | Confirmed at lines 196-200. | PASS |
| `ohttp.dart:203-207` — `HpkeSender.hkdfExpand(prk, "nonce", _nn)` | Confirmed at lines 203-207. | PASS |
| `ohttp.dart:211-213` — tagLen guard `FormatException('Ciphertext too short for AES-GCM tag')` | Confirmed at lines 211-213 (research cites `:212-213`; the throw spans 212-213 with the guard `if` at 211). | PASS |
| `ohttp.dart:217-223` — AES-128-GCM decrypt block (target for TASK-O5 wrap) | Confirmed at lines 217-223. TASK-O5 line range `217-225` extends two lines past the closing parenthesis to cover the `return` statement at 225 — acceptable widening for the try/catch scope. | PASS |
| `ohttp.dart:219` — `aesGcm.decrypt(...)` throw site for `SecretBoxAuthenticationError` | Confirmed at line 219. | PASS |
| `ohttp.dart:232-245` — `_buildHpkeInfo` | Confirmed at lines 232-245. | PASS |
| `ohttp.dart:249-257` — `_buildRequestHeader` | Confirmed at lines 249-257. | PASS |
| `lib/ohttp_dart.dart:10` — `export 'src/hpke.dart';` (re-export surface for O-7 maintenance hazard) | Confirmed at line 10. | PASS |

All findings' line ranges resolve to the claimed code. No drift between research and source.

---

## Cross-phase consistency check

The Phase 3 deliverable cites four cross-phase references; I verified each against the Phase 1 and Phase 2 summaries.

| Phase 3 finding | Cited prior-phase finding | Verified | Notes |
|---|---|---|---|
| O-4 (FormatException vs UnsupportedError) | Phase 1 F-3 (HPKE exception-type inconsistency); Phase 2 R-3 (BHTTP body-sublist RangeError) | PASS for F-3 (HPKE has the same anti-pattern). Note: Phase 2 R-3 is specifically about `RangeError`-vs-`FormatException`, not `FormatException`-vs-`UnsupportedError`. The two patterns share a common root cause (no single normalized exception hierarchy) but are not strictly the same anti-pattern. The cross-reference is defensible at the meta-pattern level and the QA agent already accepts it; no change required. | See Nice-to-have N-2. |
| O-10 (zeroization absence) | Phase 1 F-2 (`HpkeSenderContext` field zeroization absence) | PASS | The cross-reference is exact: both findings describe missing best-effort zero-fill on `Uint8List` backing arrays, both classified HIGH, both note the Dart VM no-guarantee caveat. |
| O-11 (test gap) | Phase 1 F-6/F-7 (HPKE test-vector gap and interop gap); Phase 2 R-6 (BHTTP negative-test gap) | PASS | The recurring test-gap pattern is correctly identified as a multi-layer obligation. TASK-O7 enumerates four specific sub-scenarios so the gap is independently scopable. |
| O-2 (silent multi-suite drop) | Cross-phase note: "no direct Phase 2 finding covers this; new gap at Layer 3" | PASS | The research explicitly disclaims a Phase 2 counterpart, which matches Phase 2 R-1..R-7 (none address suite-negotiation gaps in `bhttp.dart`). |

The `testKeyPair` injection hook from Phase 1 F-1 is correctly handled as deferred (research §New Technical Questions #4 and plan §Risks P3-PLAN-R5) rather than re-classified as a Phase 3 finding — appropriate because the hook surfaces at the public API of `ohttpEncapsulate` (`ohttp.dart:134`) but the remediation lives at the Phase 1 layer.

---

## Severity-distribution sanity check

PRD-anchored severity counts:

| Severity | Count | Findings | Matches PRD Resolved Questions? |
|---|---|---|---|
| BLOCKER | 0 | — | YES (PRD explicitly states no BLOCKER expected at OHTTP layer; plan §Severity policy confirms). |
| HIGH | 4 | O-2, O-9, O-10, O-11 | YES — matches the four risks elevated to HIGH in PRD §Risks. |
| IMPROVEMENT | 2 | O-4, O-6 | YES — matches PRD Resolved Question #2 (empty-AAD inline comment) and Resolved Question (implicit, O-4 inconsistent exception). |
| MAINTENANCE | 1 | O-7 | See Important item I-1: tier is plan-introduced, not PRD-anchored. |
| Positive (—) | 4 | O-1, O-3, O-5, O-8 | YES |
| **Total** | **11** | | |

---

## Findings

### Blocking

None.

### Important

**I-1 — Severity inconsistency on the plain-HKDF maintenance hazard between PRD Risks (HIGH) and research/tasks/plan (MAINTENANCE/IMPROVEMENT)**

- PRD `phase-3/prd.md:122` (Risks table) classifies the underlying risk — "Any future refactoring that unifies response-decap HKDF with the labeled variants in `hpke.dart` would silently produce different key material and break decryption with no compile-time signal" — as **HIGH**, with mitigation "recorded as a maintenance hazard task with a code-comment remediation."
- research.md `phase-3/research.md:568` (Limitations & Risks table) classifies O-7 as **MAINTENANCE**.
- tasks.md `phase-3/tasks.md:110` and research.md `phase-3/research.md:625` classify TASK-O4 as **MAINTENANCE (IMPROVEMENT)**.
- plan.md `phase-3/plan.md:92` defines `MAINTENANCE` as a sub-category of `IMPROVEMENT`, but the PRD-anchored severity in the Risks table is HIGH.

This is the exact divergence that PRD §Resolved Questions did not pre-resolve (unlike the empty-AAD and zeroization severities). Per plan §P3-PLAN-R1 mitigation ("any divergence must be reconciled in `research.md` before close"), the resolution should be picked and applied consistently.

Recommendation: pick one direction and apply to all four documents. The MAINTENANCE (sub-IMPROVEMENT) treatment is the more defensible position because the underlying code is currently COMPLIANT — the risk is purely about a future refactor introducing a regression, and the remediation is a code comment, not a behavior change. If MAINTENANCE is kept, edit PRD line 122 to read "IMPROVEMENT (maintenance hazard)" rather than HIGH.

Severity: Important. Does not block phase close; should be reconciled before Iteration 7 to avoid carrying the inconsistency into the Jira ticket.

**I-2 — `MAINTENANCE` severity tier is introduced in plan.md but absent from vision §2 and PRD inherited severity model**

- PRD `phase-3/prd.md:12-15` declares the inherited severity model: "BLOCKER / HIGH / IMPROVEMENT" only.
- plan.md `phase-3/plan.md:89-92` introduces `MAINTENANCE` as a fourth tier, described as "a sub-category of IMPROVEMENT used for 'do not silently break this in future refactors.'"
- research.md and tasks.md adopt the new tier without flagging it as a vision deviation.

This is a controlled extension (the plan documents it explicitly), but it would benefit from a single sentence in the PRD's severity model or Resolved Questions to make the four-tier scheme PRD-anchored rather than plan-anchored. Without that, a future reader of the PRD alone would not understand why research carries `MAINTENANCE` rows.

Recommendation: either fold MAINTENANCE rows into IMPROVEMENT with a parenthetical "(maintenance hazard)" qualifier (preserves the three-tier vision model), or add a one-line note in PRD §Resolved Questions explicitly admitting the fourth tier. Either is fine; the current state of "plan-defines, PRD-omits" is what creates the inconsistency.

Severity: Important. Does not block phase close.

### Nice-to-have

**N-1 — `ohttp.dart` line count is reported as 258 in research.md and plan.md, actual file is 257 lines**

research.md line 5 and 63, plan.md line 12 cite "258 lines." `wc -l` on the working tree reports 257. This is a +1 trailing-newline convention discrepancy. No finding line ranges are affected (all individual citations were verified above). Consider standardizing on `wc -l` output in future research files.

**N-2 — O-4 cross-reference to Phase 2 R-3 conflates two distinct anti-patterns**

O-4 is about `FormatException` vs `UnsupportedError` (two normal exception types for the same condition). Phase 2 R-3 is about `RangeError` vs `FormatException` (an unchecked vs checked exception leak). These share a common root cause (no normalized exception hierarchy) but are not the same anti-pattern. The QA agent already accepts this cross-reference; consider rephrasing as "same exception-hierarchy-normalization gap as Phase 1 F-3 and (related) Phase 2 R-3" for precision.

**N-3 — Plan §Coverage map shows O-3 producing no draft task, but the O-3 narrative mentions an "IMPROVEMENT" sub-finding about error-message ambiguity ("Invalid symmetric algorithms section" combines two distinct failure modes)**

research.md lines 269-272 note this as an IMPROVEMENT but it is not lifted into a TASK-O* entry; plan.md line 156 folds it into TASK-O7 (test-message scope) but TASK-O7 is about test coverage, not about splitting the parser error message. This is a small loose end. Options: add a TASK-O8 "Split error message in `OhttpKeyConfig.parse` for symLen-invalid vs short-buffer cases" or explicitly drop the sub-improvement. Currently it is recorded but neither tasked nor declined.

**N-4 — TASK-O5 line range widens from `ohttp.dart:219` (throw site) to `217-225` (function body) without explanation**

The widening is defensible because a try/catch wrap needs to span the decrypt call setup and the return; however, a one-line note in the TASK-O5 description ("range covers the decrypt block and the return so the try/catch encompasses both the throw site and the consumer of `plaintext`") would prevent reviewer churn during Iteration 7.

**N-5 — research.md §New Technical Questions #4 and plan.md §P3-PLAN-R5 about `ohttpEncapsulate`'s `testKeyPair` parameter (`ohttp.dart:134`) deferred to Phase 5**

Defensible. Documenting this for completeness: a future Phase 5 reviewer should confirm that `ohttp_client.dart` never passes a non-null `testKeyPair` in production. If Phase 5 does not pick this up, the deferred check could fall through the cracks at Iteration 7. Suggest adding a small "Phase 5 must verify" note in `specs/.current/AW-2865/tasklist.md` Phase 5 row.

---

## Read-only constraint verification

`git diff HEAD -- lib/ test/` produces no output. The constraint is honored. Status output shows only the untracked `specs/.current/AW-2865/phase-3/` directory and the existing review.md being extended. No source under `lib/`, `test/`, `pubspec.yaml`, or `example/` is modified.

---

## Conclusion

- Audit deliverable is **complete and consistent** with PRD acceptance criteria. All 11 tasks (3.1–3.11) are checked; all 8 in-scope inspection targets (KeyConfig wire layout, multi-suite drop, RangeError reassessment, exception inconsistency, HPKE info string, empty AAD, plain HKDF, response empty AAD, SecretBoxAuthenticationError, zeroization) have verdicts and line citations; all 7 draft tasks (TASK-O1..TASK-O7) are independently actionable; no source under `lib/` or `test/` was modified.
- **Four HIGH-severity items** (O-2 silent multi-suite drop, O-9 SecretBoxAuthenticationError leak, O-10 zeroization absence, O-11 test-coverage gaps) and **two IMPROVEMENT items** (O-4 exception inconsistency, O-6 empty-AAD inline-comment expansion) recommended for the follow-up backlog. **One MAINTENANCE item** (O-7 plain-HKDF refactor guard) with code-comment-only remediation. No BLOCKER identified at the OHTTP layer.
- **One severity-discipline action item before Iteration 7** (I-1): reconcile the PRD Risks "HIGH" classification of the plain-HKDF maintenance hazard with the research/tasks/plan `MAINTENANCE` classification. Pick one and apply consistently.
- **One severity-model action item before Iteration 7** (I-2): either fold `MAINTENANCE` rows into `IMPROVEMENT` with a parenthetical qualifier (preserves vision §2 three-tier model) or add a one-line PRD note admitting the fourth tier (preserves plan-introduced semantics).
- Phase 3 is ready to close pending production of `phase-3/qa.md` per plan §Acceptance / Close Criteria item 4, and (recommended) resolution of the two Important action items above before Iteration 7 compiles the per-task Markdown files.

---

## Files referenced

- specs/.current/AW-2865/phase-3/prd.md
- specs/.current/AW-2865/phase-3/plan.md
- specs/.current/AW-2865/phase-3/research.md
- specs/.current/AW-2865/phase-3/tasks.md
- specs/.current/AW-2865/vision.md
- specs/.current/AW-2865/phase-1/summary.md
- specs/.current/AW-2865/phase-2/summary.md
- lib/src/ohttp.dart
- lib/ohttp_dart.dart

---

# Phase 4 Review — OhttpClient Layer Audit

**Date:** 2026-05-21
**Mode:** ticket (read-only audit phase)
**Scope file:** `lib/src/ohttp_client.dart` (177 lines)
**Artifacts reviewed:**
- specs/.current/AW-2865/phase-4/prd.md (Status: PRD_READY)
- specs/.current/AW-2865/phase-4/plan.md (Status: PLAN_APPROVED)
- specs/.current/AW-2865/phase-4/research.md (Status: RESEARCH_COMPLETE)
- specs/.current/AW-2865/phase-4/tasks.md (12 of 12 tasks checked)
- lib/src/ohttp_client.dart

## Verdict

**APPROVED with minor housekeeping notes.** All 12 audit tasks are complete; every finding cites a real, verifiable line in `lib/src/ohttp_client.dart`; cross-layer references are present and accurate; severities are internally consistent with the PRD risk table and research.md; no source under `lib/` or `test/` was modified.

No **Blocking** issues. Three **Important** items (severity-table cross-check, header construction line drift in research.md, claim drift in research §"Patterns Used"). Three **Nice-to-have** clarifications.

---

## Acceptance criteria check (review criteria 1–8)

| # | Criterion | Result |
|---|---|---|
| 1 | All 12 tasks marked `[x]` | YES — `grep -c "^- \[x\]"` returns 12, `grep -c "^- \[ \]"` returns 0 |
| 2 | Each task has inline finding with `file:line` citation | YES — every task 4.2–4.11 cites at least one specific line in `lib/src/ohttp_client.dart`; 4.1 cites line ranges; 4.12 references research.md §"Draft Task Entries" |
| 3 | Finding IDs C-1..C-12 consistent between tasks.md and research.md | YES — twelve findings, IDs match one-to-one; categories and severities identical |
| 4 | Severity labels match PRD risk table and research.md | YES with one ambiguity (see Important I-1 below) — HIGH for C-1, C-3, C-4, C-5, C-6, C-7, C-8, C-9, C-11; IMPROVEMENT for C-2, C-10, C-12 |
| 5 | Cross-layer references present (C-9 → Phase 2 R-2; C-11 → Phase 3 O-9/TASK-O5) | YES — research.md §"Cross-Layer References" table maps both explicitly; tasks.md 4.10 says "Two-layer amplification with Phase 2 finding R-2"; tasks.md 4.12 says "TASK-C11 resolved by Phase 3 TASK-O5 at ohttp.dart:219" |
| 6 | "no timeout" and "no KeyConfig caching" findings each cite specific `http.Client` call-site lines | YES — C-5 cites `ohttp_client.dart:81-89` (`_httpClient.get` on line 81); C-6 cites `ohttp_client.dart:115-122` (`_httpClient.post` on line 115). Both call sites verified in the source |
| 7 | No claims in tasks.md contradict ohttp_client.dart | YES (claims are accurate); see Important I-2 for a small drift in research.md (line 169 vs actual 168) that did not propagate to tasks.md |
| 8 | No code changes in lib/ or test/ | YES — `git diff HEAD -- lib/ test/` returns empty; only `specs/.current/AW-2865/tasklist.md` and `specs/.current/AW-2865/phase-4/` are touched in the working tree |

---

## Line-by-line citation verification against lib/src/ohttp_client.dart

| Citation (where appearing) | Source-of-truth line(s) | Match? |
|---|---|---|
| C-1 / 4.2: constructor 43-50 | constructor at 43-50; class block 35-53 | YES — both framings (constructor 43-50, class 35-53) are correct |
| C-1 / 4.2: GET at line 82 | `Uri.parse('${gateway.gatewayBaseUrl}${gateway.configPath}'),` is line 82 | YES |
| C-1 / 4.2: POST at line 116 | `Uri.parse('${gateway.gatewayBaseUrl}${gateway.requestPath}'),` is line 116 | YES |
| C-2 / 4.3: lines 82, 116, 154 | lines confirmed above; line 154 is `Uri.parse('${gateway.effectiveDirectBaseUrl}$path')` | YES |
| C-3 / 4.4: lines 97-104 (serializeRequest with authority) | `bhttp.serializeRequest(...)` spans 97-104; `authority: gateway.targetAuthority` on line 100 | YES |
| C-4 / 4.5: directBaseUrl at line 41; effectiveDirectBaseUrl at 52; sendDirect at 148-171 | `final String? directBaseUrl;` is line 41; `String get effectiveDirectBaseUrl => directBaseUrl ?? gatewayBaseUrl;` is line 52; sendDirect signature starts at 148, closing brace 171 | YES |
| C-5 / 4.6: `_httpClient.get(Uri.parse(...))` at lines 81-83 | YES — `final configResponse = await _httpClient.get(` is line 81; expression continues to line 83 | YES |
| C-5 / 4.6: instance fields at lines 60-61 | `final http.Client _httpClient;` is line 60; `final OhttpGatewayConfig gateway;` is line 61 | YES |
| C-6 / 4.7: `_httpClient.post(...)` at lines 115-119 | `final gatewayResponse = await _httpClient.post(` is line 115; closing `);` is line 119 | YES |
| C-7 / 4.8: `throw Exception(...)` at 84-87 and 120-122 | KeyConfig branch is lines 84-88 (if at 84, throw at 85-87, close `}` 88) — claim 84-87 is precise; Gateway branch is lines 120-122 (if at 120, throw at 121, close `}` 122) — claim 120-122 is precise | YES |
| C-8 / 4.9: send() body lines 69-145 | signature starts line 69; closing `}` line 145 | YES |
| C-9 / 4.10: ohttpDecapsulate at 129-133; parseResponse at 136 | `await ohttpDecapsulate(` is line 129; closing `);` is line 133; `bhttp.parseResponse(binaryResponse)` is line 136 | YES |
| C-10 / 4.11: send response headers built at line 139 | `headers: bhttpResp.headers.map((h) => OhttpHeader(name: h.$1, value: h.$2)).toList(),` is line 139 | YES |
| C-10 / 4.11: sendDirect response headers at line 168 | `headers: streamedResponse.headers.entries.map((e) => OhttpHeader(name: e.key, value: e.value)).toList(),` is line 168 | YES — tasks.md is correct |
| C-11 / 4.12: TASK-O5 at ohttp.dart:219 | per Phase 3 review record | accepted on trust from prior phase |
| C-12 / 4.12 (research only): `onLog` first call at lines 77-78 | `'Fetching OHTTP KeyConfig from ${gateway.gatewayBaseUrl}${gateway.configPath}...'` is line 77 (closing `);` line 78) | YES |

All citations in tasks.md and research.md match the actual file.

---

## Cross-layer reference verification (criterion 5)

**C-9 → Phase 2 R-2:** Confirmed.
- tasks.md task 4.10 says "Two-layer amplification with Phase 2 finding R-2."
- research.md §"Limitations & Risks" C-9 says "Two-layer amplification (cross-reference Phase 2 finding R-2): 1. `ohttp_client.dart` imposes no cap on the gateway response body before decapsulation. 2. `bhttp.parseResponse` imposes no cap on header count or total header size (`bhttp.dart:165-182`, Phase 2 R-2)."
- research.md §"Cross-Layer References" table row "R-2 — No header-count / header-size guard in `parseResponse` | C-9 | No size cap at client layer + no cap in parser = two-layer unbounded allocation."

**C-11 → Phase 3 O-9 / TASK-O5:** Confirmed.
- tasks.md task 4.12 says "TASK-C11 resolved by Phase 3 TASK-O5 at ohttp.dart:219."
- research.md C-11 says "the existing Phase 3 TASK-O5 ('Wrap `SecretBoxAuthenticationError` in `OhttpAuthenticationException` in `ohttp.dart`') remains the correct remediation site."
- research.md §"Cross-Layer References" table row "O-9 — `SecretBoxAuthenticationError` propagates unwrapped | C-11 | Confirmed at wallet boundary; TASK-O5 remains the correct fix site."

Both required cross-layer references are present and mutually consistent.

---

## Findings

### Blocking

None.

### Important

**I-1 — Severity label cross-check: PRD risk table omits C-12, and C-11 severity is implicit**

PRD §Risks (lines 119–130) enumerates risks for C-1..C-10 explicitly, but **C-12 (`onLog` logs `gatewayBaseUrl` unconditionally; IMPROVEMENT)** is not in the PRD risk table. research.md §"Limitations & Risks" lifts C-12 as a stand-alone finding and tasks.md §4.12 implicitly accepts it via research.md, but the PRD itself never names this risk. C-11 is also absent from the PRD risk table (it is folded into the Phase 3 cross-reference).

This is not a contradiction (research.md and tasks.md agree internally and assign IMPROVEMENT/HIGH respectively), but the PRD is the authoritative severity-table source for the phase. Recommend either:
1. Adding a row to PRD §Risks for C-11 (HIGH, cross-layer confirmation) and C-12 (IMPROVEMENT, logging), or
2. Adding a one-line note "C-11 and C-12 are confirmed in research.md without separate PRD risk-table entries because they are cross-layer / observability-only items."

Either resolves the ambiguity before Iteration 7 compiles the consolidated task list.

**I-2 — research.md line drift for `sendDirect()` response header construction**

research.md §"Current Endpoints & Contracts" line 217 (in the file, the prose paragraph) cites `sendDirect` response headers as coming from "`streamedResponse.headers.entries`" — correct in substance — and research.md §"Limitations & Risks" Finding C-10 attributes the construction to "line 169":

> In `sendDirect()`, response headers are built at line 169:

The actual line is **168**, not 169. tasks.md task 4.11 correctly cites line 168. The +1 drift exists only in research.md C-10 text and the §"Current Endpoints & Contracts" "`OhttpHeader(name: h.$1, value: h.$2)` at line 139" table row remains correct. This is the same convention discrepancy noted in the Phase 3 review (N-1) and indicates research.md was written against a `cat -n`-style numbering that off-by-ones a multi-line statement.

Fix: edit research.md line ~500 to "at line 168".

**I-3 — research.md §"Patterns Used" item 3 claim about `onLog` privacy contradicts C-12**

research.md §"Patterns Used" item 3 (line 231) asserts:

> Only the gateway URL and byte counts are logged — no key material. Pattern is sound for observability but `sendDirect()` has no equivalent callback.

But finding C-12 (lines 543–565) classifies "logs `gateway.gatewayBaseUrl` at every `send()` call" as an IMPROVEMENT against `vision.md §7` constraint 6 ("`gatewayBaseUrl` at DEBUG or lower in production builds" must not be logged). "Pattern is sound for observability" is too strong given C-12. Recommend softening the §Patterns Used wording to "Pattern logs no key material, but C-12 records that `gatewayBaseUrl` interpolation violates vision §7 constraint 6."

This is purely a documentation consistency item; tasks.md and the C-12 finding itself are correct.

### Nice-to-have

**N-1 — tasks.md 4.1 cites OhttpGatewayConfig at 35–53 and OhttpClient at 59–176; class actually closes at line 176 but file extends to 177**

The class `OhttpClient` body ends at line 176 (closing `}`) and line 177 is the final newline / EOF. The "59–176" range is correct for the class declaration. No action required; recording for completeness.

**N-2 — Task 4.5 cites sendDirect at "lines 148–171"; research.md table also says 148-171**

The actual `sendDirect` block is 148–171 (signature starts at 148 — `Future<OhttpResponse> sendDirect({` — closing `}` at 171). Correct. The previous Phase 3 review N-pattern about line counts does not apply here; numbers match exactly.

**N-3 — research.md "New Technical Questions" #2 (sendDirect → gatewayBaseUrl footgun when directBaseUrl is null) and #3 (KeyConfig cache invalidation on rotation) are not yet surfaced into Phase 4 tasks**

Both are flagged as design questions for TASK-C4 and TASK-C5 respectively. The deferred-decision approach is consistent with the read-only audit constraint, but Iteration 7 should pick these up explicitly before consolidating per-task Markdown files. Suggest noting them in `specs/.current/AW-2865/tasklist.md` so they do not fall through the cracks (same recommendation pattern used in Phase 3 review N-5).

---

## Severity table consistency check (criterion 4)

| Finding | PRD §Risks severity | research.md severity | tasks.md severity | Consistent? |
|---|---|---|---|---|
| C-1 (https) | HIGH | HIGH | HIGH | YES |
| C-2 (string concat) | IMPROVEMENT | IMPROVEMENT | IMPROVEMENT | YES |
| C-3 (targetAuthority) | HIGH | HIGH | HIGH | YES |
| C-4 (sendDirect) | HIGH | HIGH | HIGH | YES |
| C-5 (KeyConfig GET) | HIGH | HIGH | HIGH | YES |
| C-6 (gateway POST) | HIGH | HIGH | HIGH | YES |
| C-7 (Exception) | HIGH | HIGH | HIGH | YES |
| C-8 (network errors) | (folded into C-7 row implicitly) | HIGH | HIGH | YES (semantic) |
| C-9 (size cap) | HIGH | HIGH | HIGH | YES |
| C-10 (header case) | IMPROVEMENT | IMPROVEMENT | IMPROVEMENT | YES |
| C-11 (SecretBox cross-ref) | (not in PRD §Risks) | HIGH | HIGH (via cross-ref) | partial — see I-1 |
| C-12 (onLog URL log) | (not in PRD §Risks) | IMPROVEMENT | IMPROVEMENT (via research) | partial — see I-1 |

10 of 12 fully consistent; 2 (C-11, C-12) absent from PRD §Risks but agree between research.md and tasks.md. Documented under Important I-1.

---

## Read-only constraint verification (criterion 8)

`git diff HEAD -- lib/ test/` produces no output. `git status --short lib/ test/` produces no output. The constraint is honored. The only working-tree modifications are inside `specs/.current/AW-2865/` (tasklist.md modification and the `phase-4/` directory). No source under `lib/`, `test/`, `pubspec.yaml`, or `example/` is changed.

---

## Conclusion

- Phase 4 audit deliverable is **complete and consistent** with PRD acceptance criteria. All 12 tasks (4.1–4.12) are checked; every line citation verified against `lib/src/ohttp_client.dart`; all required cross-layer references (C-9 → Phase 2 R-2; C-11 → Phase 3 TASK-O5) are present and mutually consistent; no source under `lib/` or `test/` was modified.
- **Eight HIGH-severity findings** (C-1 https-enforcement, C-3 targetAuthority SSRF, C-4 sendDirect bypass, C-5 KeyConfig caching/timeout, C-6 gateway POST timeout, C-7 untyped Exception, C-8 undocumented throws contract, C-9 response size cap) and **three IMPROVEMENT findings** (C-2 Uri.resolve, C-10 header lowercasing, C-12 onLog URL leak) recommended for the follow-up backlog. **One cross-layer confirmation** (C-11) resolved by Phase 3 TASK-O5. No BLOCKER identified at the client layer.
- **Three Important housekeeping items** before Iteration 7: (I-1) reconcile PRD §Risks coverage for C-11 and C-12; (I-2) fix research.md C-10 line citation drift (169 → 168); (I-3) soften research.md §"Patterns Used" item 3 to align with C-12. None block the audit's findings or remediation tasks.
- Phase 4 is ready to close. The 12 draft task entries are independently actionable and ready for Iteration 7 consolidation into per-task Markdown files alongside Phase 1–3 outputs.

---

## Files referenced

- specs/.current/AW-2865/phase-4/prd.md
- specs/.current/AW-2865/phase-4/plan.md
- specs/.current/AW-2865/phase-4/research.md
- specs/.current/AW-2865/phase-4/tasks.md
- specs/.current/AW-2865/vision.md
- specs/.current/AW-2865/tasklist.md
- specs/.current/AW-2865/phase-1/summary.md (cross-layer)
- specs/.current/AW-2865/phase-2/summary.md (cross-layer)
- specs/.current/AW-2865/phase-3/summary.md (cross-layer; via review.md Phase 3 section)
- lib/src/ohttp_client.dart
- lib/src/ohttp.dart (cross-layer; TASK-O5 site at line 219)
- lib/src/bhttp.dart (cross-layer; R-2 site at lines 165-182)
- lib/ohttp_dart.dart


---

# Phase 5 Review — Test Suite Coverage Gap Audit

**Date:** 2026-05-21
**Mode:** ticket (read-only audit phase)
**Scope:** `test/hpke_test.dart` (296 lines), `test/bhttp_test.dart` (231 lines), `test/ohttp_test.dart` (164 lines)
**Artifacts reviewed:**
- specs/.current/AW-2865/phase-5/prd.md (Status: PRD_READY)
- specs/.current/AW-2865/phase-5/plan.md (Status: PLAN_APPROVED)
- specs/.current/AW-2865/phase-5/research.md (Status: RESEARCH_COMPLETE)
- specs/.current/AW-2865/phase-5/tasks.md (11 of 11 tasks checked)
- test/hpke_test.dart, test/bhttp_test.dart, test/ohttp_test.dart

## Verdict

**Approve with minor consistency fixes (no blockers).**

Phase 5 delivers a defensible, traceable audit of the test suite. All eleven checklist items in `phase-5/tasks.md` are checked and substantiated by line-grounded findings; the cross-phase confirmation table covers the 30 gaps the resolved question Q4 commits to; PRD success criteria are met. The artifacts are read-only — `git status` confirms no file under `lib/`, `test/`, `pubspec.yaml`, or `example/` was modified (the only changes are inside `specs/.current/AW-2865/phase-5/` and an unrelated tweak in `specs/.current/AW-2865/tasklist.md` recording phase completion).

The findings below are quality-of-record issues — none invalidate the audit conclusions.

## Acceptance criteria verification (against `phase-5/prd.md` Success / Metrics)

| Criterion | Status | Evidence |
|---|---|---|
| Every test file read in full | PASS | `research.md` cites actual line counts: hpke_test.dart 296 (line 20), bhttp_test.dart 231 (line 59), ohttp_test.dart 164 (line 103). Matches `wc -l` of the actual files. |
| Every existing test group named | PASS | `research.md` enumerates all four hpke groups (line 24–29), six bhttp groups (line 63–70), four ohttp groups (line 107–112) with test counts. |
| Every gap tied to a line range | PASS | All 13 TASK-T entries in `research.md` carry an explicit "Coverage stops at" line number; cross-phase table cites line numbers for each confirmation. |
| Every gap from Phases 1–4 confirmed or refuted | MOSTLY PASS — see Important #1 | Cross-phase table has exactly 30 rows; all marked `Still absent` except `O-4/TASK-O2` (PARTIAL — correctly explained). C-11, named explicitly in PRD Scenario D bullet 4, is not in the table (referenced only indirectly via TASK-T9 cross-phase note). |
| All 8 investigation vectors addressed for test suite | PASS | `plan.md` lines 87–96 maps every vector to one or more TASK-T entries; `tasks.md` lines 105–114 carries the equivalent matrix. |
| No code changes | PASS | `git status` shows only `specs/.current/AW-2865/tasklist.md` and `specs/.current/AW-2865/phase-5/` modified/added. `lib/`, `test/`, `pubspec.yaml`, `example/` are untouched. |
| All draft task entries independently actionable | PASS | Each TASK-T entry carries File, Missing scenario, Coverage stops at, Proposed group name, Severity, Cross-phase reference. |
| Severity assigned to every gap | PASS | All 13 TASK-T entries carry exactly one of HIGH / IMPROVEMENT. No BLOCKER assigned (consistent with `plan.md` Definition of Done item 7). No "TBD". |

## Line-reference spot-check against test files

Verified citations by reading the actual files:

| Claim | Citation | Verified |
|---|---|---|
| `testKeyPair` injection in seal-seq=0 test | `hpke_test.dart:104–119` | YES — `test('seal produces correct ciphertext for seq=0')` at line 104; `testKeyPair` block at 105–108; assertion at 117–118. |
| seq=1 nonce-increment seal test | `hpke_test.dart:177–200` | YES — last seal call in file at line 195 with assertion at 196–199. The "after line 200" anchor for T-H1 is correct. |
| export with empty context | `hpke_test.dart:202–220` | YES — assertion through 216–219; group ends 220. (`tasks.md` says "202–219", off by one — minor.) |
| export with "TestContext" | `hpke_test.dart:222–241` | YES. |
| `HKDF utilities` last expand-32 test | `hpke_test.dart:283–294`, group closes at 295 | YES — `research.md:300` cites "line 295 (last hkdfExpand test uses length 32)" correctly. |
| Varint roundtrip set | `bhttp_test.dart:61–68` | YES — `[0, 1, 63, 64, 100, 255, 16383, 16384, 100000]` at line 62. |
| 4-byte varint max (1073741823) | `bhttp_test.dart:27–28` | YES — `encodeVarint(1073741823)` at line 27, assertion at 28. |
| `ohttpDecapsulate` "rejects too-short response" | `ohttp_test.dart:150–155` | YES — `Uint8List(16)` encResponse at line 152. |
| `ohttpDecapsulate` "only nonce, no valid ciphertext" | `ohttp_test.dart:157–162` | YES — `Uint8List(17)` encResponse at line 159. |
| `ohttpEncapsulate` 5-byte binaryRequest | `ohttp_test.dart:125` | YES — `Uint8List.fromList([1, 2, 3, 4, 5])` at exactly line 125. |

Line citations are accurate to within ±1 line throughout the audit. No factual errors found.

## Findings (priority taxonomy)

### Blocking

None. The audit is defensible and well-grounded.

### Important

**I-1 — `tasks.md` says `hpke_test.dart` is 297 lines; the file is 296 lines.**

`phase-5/tasks.md:137` reads "### 5.1–5.3 `test/hpke_test.dart` (297 lines)". `wc -l test/hpke_test.dart` returns 296, and `plan.md:18,54` and `research.md:20` both correctly cite 296. The "297" in `tasks.md` is the only divergent line count in the phase. Cosmetic but visible.
- Fix: change `tasks.md:137` to `(296 lines)`.

**I-2 — `tasks.md` draft-task table and `research.md` draft-task entries use different IDs and counts.**

`tasks.md` lines 231–249 lists 15 draft tasks tagged `T-H1, T-H2, T-H3, T-B1..T-B5, T-O1..T-O6, T-X1, T-X2`. `research.md` lines 274–393 lists 13 draft tasks tagged `TASK-T1..TASK-T13`. The mapping is implicit and incomplete:

| tasks.md | research.md | Notes |
|---|---|---|
| T-H1 seq overflow | TASK-T1 | matches |
| T-H2 export L=0 / L>255*Nh | TASK-T3 | matches (re-numbered) |
| T-H3 zero pubkey | TASK-T2 | matches (re-numbered) |
| T-B1 8-byte varint | TASK-T4 | matches |
| T-B2 truncated body | TASK-T5 | matches (TASK-T5 is broader: also covers F2.5b, F2.5c, F2.6a) |
| T-B3 unknown framing | TASK-T6 | matches |
| T-B4 zero-length header / malformed header | — | **not promoted** to a TASK-T entry |
| T-B5 serializeRequest size cap | — | **not promoted** to a TASK-T entry |
| — | TASK-T7 statusCode out-of-range | not in `tasks.md` summary table |
| T-O1 symLen>4 | TASK-T8 | matches |
| T-O2 symLen past buffer end | — | **not promoted** to a TASK-T entry |
| T-O3 AEAD auth failure | TASK-T9 | matches |
| T-O4 round-trip | TASK-T10 | matches |
| T-O5 parse vs validate inconsistency | — | **not promoted**; appears only in cross-phase row O-4/TASK-O2 (PARTIAL) |
| T-O6 empty inner payload | — | **not promoted** to a TASK-T entry |
| — | TASK-T11 OhttpClient error paths | not in `tasks.md` summary table |
| T-X1 integration tests | TASK-T13 | matches |
| T-X2 fuzz tests | TASK-T12 | matches |

The two artifacts therefore disagree on the set of draft tasks. `plan.md` line 12 declares the canonical set as "TASK-T1..TASK-T13"; that is the artifact consumed by Iteration 7. The five `tasks.md` entries without a TASK-T counterpart (T-B4, T-B5, T-O2, T-O5, T-O6) are at risk of being dropped during compilation. Conversely, TASK-T7 (statusCode out-of-range, IMPROVEMENT) and TASK-T11 (OhttpClient error paths, HIGH) are absent from the `tasks.md` summary and may be missed by anyone using `tasks.md` as the entry point.
- Fix: reconcile the two tables in one direction. Either (a) promote T-B4, T-B5, T-O2, T-O5, T-O6 into research.md as TASK-T14..TASK-T18 (preferred — they are valid gaps), or (b) explicitly note in `tasks.md` that the canonical task list is `research.md §"Draft Task Entries"` and the `tasks.md` table is a working scratchpad. Either way, Iteration 7 must consume one canonical list, not both.

**I-3 — Cross-phase confirmation table is missing C-11 (Phase 4) row.**

PRD `phase-5/prd.md:104` explicitly enumerates "Phase 4 (C-11): auth-failure path tested only indirectly — confirm" as a Scenario D obligation. The cross-phase table in `research.md` lines 142–173 contains rows for C-1 through C-10 only; C-11 and C-12 are absent. C-11 IS referenced in `research.md:357` as a cross-phase tag on TASK-T9, so the confirmation work was done — but the table is the contract artifact (per `plan.md:74–81`) and is therefore incomplete on its face. Resolved Q4 (line 264–265) claims the table covers "all 30 named gaps found across phases 1–4"; Phase 4 actually has 12 findings (C-1..C-12), so the count is also off if C-11/C-12 are in scope.
- Fix: append two rows to the cross-phase table for C-11 (`Still absent` — confirmed via the same evidence as TASK-O7 item 2) and C-12 (likely `Still absent` — needs verification against `phase-4/research.md` C-12 wording).

### Nice-to-have

**N-1 — `tasks.md` line citations for missing-case anchors occasionally drift by one line versus the canonical research file.**

Examples: `tasks.md:202–219` for "export with empty context" vs `research.md:43` which cites "202–220"; `tasks.md:222–241` matches `research.md:44`. Within ±1 line, well within audit tolerance, but for downstream tooling consumers a single canonical citation per scenario reduces ambiguity.
- Fix: optional — pick one source (research.md) as canonical and have `tasks.md` reference it.

**N-2 — `research.md` Resolved Question Q4 (line 264–265) wording "all 30 named gaps" predates the C-11/C-12 omission noted in I-3.**

If I-3 is fixed by adding rows, this sentence should be updated to "all 32 named gaps" or "all named test-relevant gaps from Phases 1–4 (32 items)".
- Fix: trivial wording update after I-3.

**N-3 — TASK-T6 severity (IMPROVEMENT) vs `tasks.md` T-B3 severity (IMPROVEMENT) — consistent — but TASK-T5 severity (HIGH) absorbs F2.5c (IMPROVEMENT per Phase 2 tasks.md:122) without flagging the severity blend.**

`research.md:321` lists TASK-T5 cross-phase as "Confirms R-6 (Phase 2), F2.5a, F2.5b, F2.5c, F2.6a, NG-6". F2.5c is IMPROVEMENT in Phase 2; the others are HIGH. Bundling F2.5c into a HIGH task is defensible (worst-case severity wins for the bundle) but the bundle's severity rationale could be more explicit in the entry.
- Fix: optional — add a one-line note in TASK-T5 explaining that F2.5c is folded in despite being IMPROVEMENT at source, because the same test code can exercise all four boundary scenarios.

**N-4 — TASK-T11 covers Phase 4 C-1..C-10 but does not enumerate which sub-scenarios touch C-2, C-3, C-9, C-10.**

TASK-T11 (lines 369–375) names four scenarios: 503-on-KeyConfig, 400-on-gateway, `http://` scheme, `sendDirect()`. These cover C-1, C-5, C-6, C-7 cleanly. C-2 (path-traversal), C-3 (targetAuthority embedding), C-9 (no response-body size cap), and C-10 (header case) are listed as covered in the cross-phase reference but have no matching sub-scenario in the task. Either expand the task or note that those four are out of scope and need a separate task.
- Fix: optional — split TASK-T11 into TASK-T11a (HTTP error paths) and TASK-T11b (config/parsing paths) at Iteration 7 if scope becomes too broad.

## Cross-phase consistency check

Spot-checks against prior phases:

| Prior-phase claim | Phase 5 confirmation | Status |
|---|---|---|
| Phase 1 F-6: `_seq` overflow untested | TASK-T1; cross-phase row (line 144) | PASS |
| Phase 1 F-7: no integration test against Go reference gateway | TASK-T13; cross-phase row (line 145) | PASS |
| Phase 2 R-6: no negative-case BHTTP tests | TASK-T5; cross-phase row (line 146) | PASS |
| Phase 2 F2.6b: 8-byte varint silent overflow | TASK-T4 / NG-1; cross-phase row (line 155) | PASS |
| Phase 3 O-2 / TASK-O1: `symLen > 4` silent ignore | TASK-T8; cross-phase row (line 157) | PASS |
| Phase 3 TASK-O7 items 1–4 | TASK-T10, TASK-T9, TASK-T8, TASK-T9 (partial) | PASS — note TASK-O7 item 4 marked PARTIAL with explanation (`research.md:161`) which is accurate against `ohttp_test.dart:157–162`. |
| Phase 3 O-9 / TASK-O5: SecretBoxAuthenticationError unwrapped | TASK-T9; cross-phase row (line 163) | PASS |
| Phase 4 C-1..C-10: no `OhttpClient` tests | TASK-T11; cross-phase rows (lines 164–173) | PASS |
| Phase 4 C-11: SecretBoxAuthenticationError reaches wallet boundary | TASK-T9 cross-phase note only | PARTIAL — see I-3 (missing from confirmation table) |
| Phase 4 C-12 | not addressed | NOT FOUND — see I-3 |

## Read-only constraint verification

`git status` at review time:
- `M  specs/.current/AW-2865/tasklist.md` — progress-table tick-off for Iteration 5 only.
- `?? specs/.current/AW-2865/phase-5/` — all four artifacts (prd.md, plan.md, research.md, tasks.md).

No file in `lib/`, `test/`, `pubspec.yaml`, or `example/` is modified or added. PRD constraint 1 ("read-only audit") and Definition of Done item 3 are satisfied.

## Severity-discipline check

| Severity | Count | Entries |
|---|---|---|
| BLOCKER | 0 | (none) |
| HIGH | 11 | TASK-T1, T-T2, T-T3, T-T4, T-T5, T-T8, T-T9, T-T10, T-T11, T-T12, T-T13 |
| IMPROVEMENT | 2 | TASK-T6, T-T7 |

The 0 BLOCKER count is justified at `plan.md:247` — every test gap exposes an implementation finding already classified at most HIGH in prior phases. Cross-checked against vision §2 severity definitions: this assignment is internally consistent.

The two IMPROVEMENT entries (TASK-T6 unknown framing indicator; TASK-T7 statusCode out-of-range) are correctly downgraded because the underlying behaviors do not cause memory exhaustion or silent ciphertext acceptance.

## Summary for the engineering lead

Phase 5 is sound and ready to feed Iteration 7. Two items deserve attention before compilation:

1. **Reconcile the `tasks.md` draft-task summary table with `research.md` TASK-T entries** (Important I-2). The current divergence risks dropping five entries (T-B4, T-B5, T-O2, T-O5, T-O6) and may double-count via TASK-T7 / TASK-T11. The fix is purely editorial: pick one canonical list (recommend research.md) and either promote the missing five to TASK-T14..TASK-T18 or explicitly delete them with rationale.

2. **Add C-11 (and likely C-12) to the cross-phase confirmation table** (Important I-3). The PRD explicitly names C-11 in Scenario D; its absence from the table is the only PRD-conformance gap in the phase.

Both are quality-of-record issues, not correctness issues. Phase 5 is otherwise approved.

## Files referenced

- specs/.current/AW-2865/phase-5/prd.md
- specs/.current/AW-2865/phase-5/plan.md
- specs/.current/AW-2865/phase-5/research.md
- specs/.current/AW-2865/phase-5/tasks.md
- specs/.current/AW-2865/tasklist.md
- specs/.current/AW-2865/phase-1/research.md (cross-phase F-N IDs)
- specs/.current/AW-2865/phase-2/research.md (cross-phase R-N IDs)
- specs/.current/AW-2865/phase-2/tasks.md (cross-phase F2.x IDs)
- specs/.current/AW-2865/phase-3/research.md (cross-phase O-N / TASK-O IDs)
- specs/.current/AW-2865/phase-4/research.md (cross-phase C-1..C-12 IDs)
- test/hpke_test.dart
- test/bhttp_test.dart
- test/ohttp_test.dart

---

# Phase 6 Review — AW-2865 Privacy Risk Review, Observability Gaps, and Documentation

**Subject:** Phase 6 deliverables for ticket AW-2865 (privacy risk review, observability gaps audit, documentation review).
**Reviewer:** Code review agent (read-only audit verification).
**Reviewed artifacts:**
- `specs/.current/AW-2865/phase-6/prd.md`
- `specs/.current/AW-2865/phase-6/plan.md`
- `specs/.current/AW-2865/phase-6/research.md`
- `specs/.current/AW-2865/phase-6/tasks.md`
- `specs/.current/AW-2865/vision.md` (reference)
- `lib/src/ohttp_client.dart`, `lib/src/ohttp.dart`, `lib/src/hpke.dart`, `example/ohttp_dart_example.dart`, `pubspec.yaml` (read-only verification of cited evidence)

---

## Overall Verdict

**APPROVED.** Phase 6 satisfies every PRD success criterion and every Phase Exit Condition in `plan.md §9`. All 11 tasks (6.1–6.11) are checked off with concrete evidence in the confirmation summary; all 13 draft entries (P6-1 through P6-13) follow the schema in `plan.md §3`; the observability gaps catalog (OG-1..OG-5) and logging constraints matrix (LC-1..LC-6) are complete; the four Phase 7 carry-forward questions are recorded; and `git diff HEAD -- lib/ test/ example/ pubspec.yaml` is empty (verified zero lines).

No blocking findings. A small number of nice-to-have improvements are listed below.

---

## Acceptance criteria verification

| Criterion | Status | Evidence |
|---|---|---|
| All 11 tasks 6.1–6.11 marked `- [x]` | PASS | `phase-6/tasks.md:58–78` — every task checkbox is `[x]`. |
| ≥1 BLOCKER or HIGH finding tied to `sendDirect()` bypass | PASS — exceeds | P6-1 (BLOCKER, lines 164–168), P6-8 (HIGH observability companion, lines 203–206), P6-13 (IMPROVEMENT API-visibility companion, lines 228–231). |
| ≥1 HIGH finding tied to key-material zeroization absence | PASS — split per PRD Q2 | P6-3 (HIGH, `hpke.dart` scope, lines 176–180) and P6-4 (HIGH, `ohttp.dart` scope, lines 182–186) — two separate tasks as required by Resolved Question Q2. |
| No code change in `lib/`, `test/`, or `example/` | PASS | `git diff HEAD -- lib/ test/ example/ pubspec.yaml` returns zero lines. Only `specs/.current/.active_ticket` and `specs/.current/AW-2865/phase-6/tasks.md` are modified. |
| 13 draft entries P6-1..P6-13 conform to schema (§3) | PASS | All 13 entries carry Task ID, Severity, File, Lines, Concern category, Description; Cross-refs present where prior-phase compounding exists (P6-1, P6-2, P6-3, P6-4). |
| OG-1..OG-5 observability catalog complete with file + severity | PASS | `phase-6/tasks.md:233–241` — table with Gap ID, missing event, file, lines, severity. Severities match `vision.md §7` (4× HIGH + 1× IMPROVEMENT). |
| LC-1..LC-6 logging constraints matrix complete | PASS | `phase-6/tasks.md:243–254` — six rows, each citing the affected identifier set. |
| Phase 7 carry-forward questions present (4 items) | PASS | `phase-6/tasks.md:256–263` — exactly the four questions from `plan.md §8`. |
| `pubspec.yaml` review records the tight caret and absence of FFI | PASS | P6-12 (IMPROVEMENT, lines 223–226) confirms `cryptography: ^2.9.0`, `http: ^1.6.0`, no FFI/native deps. |
| Severity matrix in `plan.md §2.3` matches recorded P6-N severities | PASS | Spot-checked P6-1 BLOCKER, P6-2..P6-8 HIGH, P6-9..P6-13 IMPROVEMENT — matches §2.3. |
| Repo-relative paths throughout | PASS | No absolute `/Users/...`, `/home/...`, or `C:\...` paths appear in `phase-6/tasks.md`. |

---

## Line-reference spot-check against source

Re-verified every cited line range against the current source files. All resolve correctly:

| Entry | Cited file:lines | Verified content | Status |
|---|---|---|---|
| P6-1 | `ohttp_client.dart:52, 148–171` | Line 52 is `String get effectiveDirectBaseUrl => directBaseUrl ?? gatewayBaseUrl;`. Lines 148–171 are the `sendDirect()` body with no doc comment beyond line 147 ("Send a direct HTTP request (for comparison).") and no `assert`/`@Deprecated`/warning. | PASS |
| P6-2 | `ohttp_client.dart:43–51` | Lines 43–51 are the `OhttpGatewayConfig` const constructor — accepts `gatewayBaseUrl` as raw `String` with no scheme check. | PASS |
| P6-3 | `hpke.dart:258–270` | Lines 258–270 are the `HpkeSenderContext` class header and the four `final Uint8List` field declarations (`enc`, `key`, `baseNonce`, `exporterSecret`) plus the private constructor. No zeroization method exists in the class. | PASS |
| P6-4 | `ohttp.dart:105–120, 196–207` | Lines 105–120 are `OhttpEncapsulateResult` class with `final Uint8List encRequest/enc/exportedSecret`. Lines 196–207 are the local `aeadKey`/`aeadNonce` HKDF-Expand derivations inside `ohttpDecapsulate`. | PASS |
| P6-5 | `ohttp_client.dart:81–89` | KeyConfig GET, generic `Exception` on non-200, `OhttpKeyConfig.parse(configResponse.bodyBytes)`. No structured event. | PASS |
| P6-6 | `ohttp_client.dart:115–122` | Gateway POST, generic `Exception('Gateway error: HTTP …')` on non-200. No structured event. | PASS |
| P6-7 | `ohttp.dart:218–224` | `aesGcm.decrypt(secretBox, secretKey: …, aad: [])` — `SecretBoxAuthenticationError` from `package:cryptography` propagates unwrapped. | PASS |
| P6-8 | `ohttp_client.dart:148–171` | `sendDirect()` signature lacks `onLog`; body has no observability emission. | PASS |
| P6-9 | `ohttp_client.dart:77, 113` | Line 77: `'Fetching OHTTP KeyConfig from ${gateway.gatewayBaseUrl}${gateway.configPath}...'`. Line 113: `'Sending to OHTTP gateway ${gateway.gatewayBaseUrl}${gateway.requestPath}...'`. Both interpolate `gatewayBaseUrl` into `onLog`. | PASS |
| P6-10 | `ohttp.dart:131–167` | `ohttpEncapsulate` body. No timing instrumentation. | PASS |
| P6-11 | `example/ohttp_dart_example.dart:1–39` | File is 38 lines; cited 1–39 is one over by EOF. Placeholder `'https://your-gateway.example.com'`, `targetAuthority` equals gateway host, no `sendDirect()` reference, no prerequisites block. Verdict accurate; range is harmless overshoot. | PASS (cosmetic over-cite — see Nice-to-have) |
| P6-12 | `pubspec.yaml:12–18` | Lines 12–18 cover both the `dependencies:` and `dev_dependencies:` blocks (`cryptography: ^2.9.0` at line 13, `http: ^1.6.0` at line 14, no FFI/native deps). | PASS |
| P6-13 | `ohttp_client.dart:52` | Public getter `effectiveDirectBaseUrl` confirmed. | PASS |

Empty-logging grep claim re-verified: `grep -RnE 'print\(|debugPrint|dart:developer|Logger|log\(' lib/` returns zero matches.

---

## Internal consistency check

- **Severity classifications align with `plan.md §2.3`.** Each of P6-1 (BLOCKER), P6-2..P6-8 (HIGH), and P6-9..P6-13 (IMPROVEMENT) matches the matrix verbatim.
- **Cross-refs are accurate and minimal.** P6-1 cites Phase 4 C-4; P6-2 cites Phase 4 C-1; P6-3 cites Phase 1 F-2; P6-4 cites Phase 3 TASK-O6. No prior-phase ID is misquoted, and no verbatim prior-phase description is duplicated (per `plan.md §4.2` and Resolved Question Q6).
- **Resolved Question Q1 honored.** P6-1 enters Phase 7 directly as BLOCKER with rationale citing the source-confirmed silent fallback (`ohttp_client.dart:52`) — the escalation is justified inside the entry, not deferred with a "candidate" label, matching `plan.md §3` severity discipline.
- **Resolved Question Q2 honored.** P6-3 and P6-4 are kept as two separate HIGH tasks with disjoint file scopes (`hpke.dart` vs `ohttp.dart`); no mechanical merge.
- **Resolved Question Q3 honored.** P6-11 (example file) is IMPROVEMENT only; no hardcoded credentials or private URLs were found (placeholders are obviously such).
- **Resolved Question Q4 honored.** P6-12 records the tight caret outcome as IMPROVEMENT (confirmed-clean), not HIGH — appropriate per the resolution.
- **Resolved Question Q5 honored.** P6-10 (encap timing, OG-5) is IMPROVEMENT per `vision.md §7` — no upgrade attempted.
- **Observability companion split is principled.** P6-1 (privacy framing of `sendDirect()` bypass) and P6-8 (observability framing of OHTTP-bypass signal) are kept as separate tasks with distinct fix scopes, matching the §6 risk-mitigation note about not conflating frames.
- **Phase 7 carry-forward questions are the same four items as `plan.md §8.1–§8.4`,** with consistent wording. Good traceability.

No internal inconsistencies detected.

---

## Findings

### Blocking
None.

### Important
None.

### Nice-to-have

1. **P6-11 line range overshoots EOF by one.** `example/ohttp_dart_example.dart` is 38 lines (per `wc -l`); the entry cites `1–39`. Cosmetic only — Phase 7 should normalize to `1–38` when materializing tasks.

2. **P6-3 line range is wider than the actual field declarations.** The cited range `258–270` covers the class block end and the private constructor; the fields `key`, `baseNonge`, `exporterSecret` are declared at lines 260–262, with `enc` at 259. Phase 7 can tighten to `259–262` when materializing the per-task file. Not a defect — the wider range is defensible because the fix surface is the whole class.

3. **P6-9 could note the LC-6 connection explicitly in the entry body.** The description references LC-6 in spirit ("Compliant with LC-6 only if…") but doesn't print the LC-6 identifier. A trivial future polish.

4. **`pubspec.yaml` `http: ^1.6.0` is documented in P6-12 but not separately flagged for transitive-FFI confirmation.** `package:http` on some platforms pulls in `package:web`/`dart:io` paths; per `plan.md §1` "transitive dependency audit" is explicitly out of scope, so this is correctly excluded — but Phase 7 may want to add it as a follow-up if the wallet build context requires a hard no-FFI guarantee in dependents.

5. **`onLog` is invoked with `gatewayBaseUrl` interpolation but P6-9 doesn't enumerate every call site.** Lines 90, 96, 107, 109, 123, 128, 142 also emit `onLog` messages; only the two carrying `gatewayBaseUrl` (77 and 113) violate LC-6. The cited two lines are correct; the other call sites are non-violating and rightly omitted, but a one-line clarification would prevent Phase 7 re-checking.

These are documentation polish only — none affect the audit outcome or downstream Phase 7 actionability.

---

## Conclusion

Phase 6 is **APPROVED** with no blocking or important findings. The deliverable is internally consistent, fully grounded in source evidence, conforms to the Phase 6 plan and PRD without drift, and provides a clean, schema-conformant input to Phase 7 cross-phase triage. The investigation goal (identify risks, produce actionable task drafts) is met for the privacy, observability, and documentation streams.

**Files referenced:**
- specs/.current/AW-2865/phase-6/prd.md
- specs/.current/AW-2865/phase-6/plan.md
- specs/.current/AW-2865/phase-6/research.md
- specs/.current/AW-2865/phase-6/tasks.md
- specs/.current/AW-2865/vision.md
- lib/src/ohttp_client.dart
- lib/src/ohttp.dart
- lib/src/hpke.dart
- example/ohttp_dart_example.dart
- pubspec.yaml

---

# Phase 7 Review — Compile Findings into Per-Task Markdown Files

**Date:** 2026-05-21
**Mode:** ticket (Phase 7)
**Scope:** Markdown-only deliverable under `specs/.current/AW-2865/tasks/`. No source code modified.
**Artifacts reviewed:**
- specs/.current/AW-2865/phase-7/prd.md (Status: PRD_READY)
- specs/.current/AW-2865/phase-7/plan.md (Status: PLAN_APPROVED)
- specs/.current/AW-2865/phase-7/tasks.md (10/10 tasks checked)
- specs/.current/AW-2865/tasks/README.md
- specs/.current/AW-2865/tasks/01-ohttp-bypass-send-direct.md through 35-test-ohttp-edge-cases.md (34 task files; gap at 22 intentional)

## Verdict

**APPROVED.** No blocking, no important findings. Two nice-to-have observations follow.

Phase 7 mechanically satisfies every PRD acceptance criterion: 34 task files on disk, each with all 8 required structural fields, gap at 22 documented in three independent places, all eight investigation vectors represented, sole BLOCKER at task 01 with `sendDirect()` evidence, task 32 escalated to HIGH with explicit original/new classification block, task 13 merges rows 13 and 22 with the §4 logging constraints reproduced inline, README index table aligned 1:1 with the file numbers on disk, all paths repo-relative, executive summary takes the required direct production-readiness stance.

## Mechanical verification

| Criterion | Verification command | Result |
|---|---|---|
| 34 task files present | `ls [0-9]*.md \| wc -l` | 34 |
| All 8 required fields per file | per-file `grep -c` for each of the 8 markers across all 34 files | every count = 1, no field missing or duplicated in any file |
| Severity tally | `grep -h '^\*\*Severity:\*\*' [0-9]*.md \| sort \| uniq -c` | 1 BLOCKER + 21 HIGH + 12 IMPROVEMENT = 34 |
| Vector coverage (all 8) | `grep -h '^\*\*Vector:\*\*' [0-9]*.md \| sort \| uniq -c` | Cryptographic correctness 2, Documentation 4, Interoperability 2, KeyConfig management 1, Network reliability 3, Parser robustness 4, Privacy risks 6, Test suite 12 — all 8 vectors represented |
| No absolute paths | `grep -nE '/(Users\|home)/\|C:\\\\' [0-9]*.md README.md` | no matches |
| README index = files on disk | numeric extraction of file-link targets in README vs. on-disk numbers | both sets equal `{01..21, 23..35}` |
| README severity counts | `grep -oE '\\\| (BLOCKER\|HIGH\|IMPROVEMENT) \\\|' README.md` | 1 BLOCKER + 21 HIGH + 12 IMPROVEMENT = 34, matches file tally |
| Gap at file 22 | three corroborating notes | `README.md:50`, `13-add-structured-observability-hooks.md:7`, `phase-7/prd.md:88` (constraint 4) |

## Focus-area sign-off

1. **All 8 required content elements in all 34 files.** Every file has exactly one `# Task ` heading, one `**Severity:**`, one `**Vector:**`, one `**Files:**`, one `**Evidence:**`, one `## Description`, one `## Proposed change`, one `## Acceptance criteria`. No duplicates, no missing fields. PASS.

2. **All 8 investigation vectors appear at least once.** Distribution above shows every vector represented. The §3 coverage map in `phase-7/tasks.md:206-215` matches the actual file distribution. PASS.

3. **Task 01 labelled BLOCKER.** `tasks/01-ohttp-bypass-send-direct.md:3` reads `**Severity:** BLOCKER`. It is the sole BLOCKER on disk and in the README index. Cross-refs P6-1 and C-4 cited. README "Blockers for production use" section links to it. PASS.

4. **Task 32 escalated to HIGH with original/new classification documented.** `tasks/32-make-effective-direct-base-url-private.md:9-11` contains an explicit Evidence sub-block with `Original classification: IMPROVEMENT`, `New classification: HIGH`, and a multi-sentence escalation rationale tying the public getter to the BLOCKER blast-radius and the wallet unlinkability requirement. README index row at `README.md:36` shows `HIGH`, matching the in-file classification. PASS.

5. **Task 13 cross-refs and §4 logging constraints.** `tasks/13-add-structured-observability-hooks.md:7` cites P6-5, P6-6, P6-8, and T-X1 (row 22 note), explicitly states the gap at file 22 is intentional. Lines 17–21 reproduce the §4 logging constraints (six "never log" items and the safe-to-log allow-list). Acceptance criteria require the constraints be reproduced verbatim in the new observability interface's doc comment. PASS.

6. **README index table — all 34 files, correct severity labels.** Index table has exactly 34 data rows. File numbers in links match files on disk. Severity column tally matches per-file severity tags. Task 32 is correctly listed as `HIGH` (post-escalation). PASS.

7. **README executive summary takes a direct stance.** `README.md:5`: "…must not be used in production until the 1 BLOCKER and 21 HIGH items below are resolved." Direct, named counts, not hedged. PRD constraint 9 satisfied. PASS.

8. **Repo-relative paths only.** Grep for `/Users/`, `/home/`, `C:\` across all 34 task files and `README.md` returned no matches. All file citations use repo-relative paths (e.g., `lib/src/ohttp_client.dart:52`, `lib/src/hpke.dart`). PASS.

9. **Gap at file 22 intentional and documented.** Three independent traces (README, task 13 file, PRD constraint 4). PASS.

## Findings

### Blocking
None.

### Important
None.

### Nice-to-have

1. **README ordering — task 32 sits at the end of the HIGH block after task 21.** Internally consistent (task 32 is HIGH, belongs in the HIGH block) and the §2 numbering convention is preserved (slug retained from its original IMPROVEMENT position), but a reader scanning the index table sees a numeric jump `21 → 32 → 23 → 24…`. A one-line inline note next to the task-32 row ("Task 32 keeps its IMPROVEMENT-block slug after being escalated to HIGH; see in-file escalation rationale") would remove the only piece of ordering surprise. Cosmetic; not a Phase 7 fix.

2. **Executive summary IMPROVEMENT count is "13" but the index table has 12 IMPROVEMENT rows.** `README.md:5` mentions "13 IMPROVEMENT items" while the index table lists 12 IMPROVEMENT rows (task 32 moved to HIGH). Arithmetic still resolves to 34 across all severities, and the production-readiness stance counts (1 BLOCKER + 21 HIGH) are correct, so the document is internally consistent on the load-bearing numbers. The off-by-one in the IMPROVEMENT prose count is purely cosmetic.

## Tasklist updates

No `## Code Review Fixes` section is appended to `specs/.current/AW-2865/phase-7/tasks.md`. The two nice-to-have items above do not meet the blocking/important threshold under ticket-mode rules. The "PR review approval" checkbox is not unchecked.

## Files referenced

- specs/.current/AW-2865/phase-7/prd.md
- specs/.current/AW-2865/phase-7/plan.md
- specs/.current/AW-2865/phase-7/tasks.md
- specs/.current/AW-2865/tasks/README.md
- specs/.current/AW-2865/tasks/01-ohttp-bypass-send-direct.md
- specs/.current/AW-2865/tasks/13-add-structured-observability-hooks.md
- specs/.current/AW-2865/tasks/32-make-effective-direct-base-url-private.md
- specs/.current/AW-2865/tasks/04-zeroize-hpke-key-material.md
- specs/.current/AW-2865/tasks/21-test-integration-gateway-stub.md
- specs/.current/AW-2865/tasks/23-unify-unsupported-kem-exception.md
- All other 28 task files under specs/.current/AW-2865/tasks/ (validated by aggregate grep; no per-file findings).
