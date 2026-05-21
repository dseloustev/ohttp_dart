# Phase 3 Plan: OHTTP Layer Audit Against RFC 9458

**Ticket:** AW-2865 — ohttp_dart investigation
**Phase:** 3 of N (OHTTP encapsulation/decapsulation layer)
**Status:** PLAN_APPROVED
**Inputs:** `specs/.current/AW-2865/phase-3/prd.md`, `specs/.current/AW-2865/phase-3/research.md`, `specs/.current/AW-2865/phase-3/tasks.md`, ticket-wide `specs/.current/AW-2865/idea.md`, `specs/.current/AW-2865/vision.md`; prior `specs/.current/AW-2865/phase-1/plan.md`, `specs/.current/AW-2865/phase-1/research.md`, `specs/.current/AW-2865/phase-2/plan.md`, `specs/.current/AW-2865/phase-2/research.md` (referenced for deliverable shape and cross-layer findings).

---

## Phase Scope

This phase produces a **read-only audit deliverable** for `lib/src/ohttp.dart` (RFC 9458 OHTTP encapsulation/decapsulation core, 258 lines). The deliverable is a set of severity-classified, independently actionable draft task entries — already captured in `specs/.current/AW-2865/phase-3/research.md` under "Findings (O-1..O-11)", "Limitations & Risks", and "Draft Task Entries (TASK-O1..TASK-O7)".

The plan below describes how the audit deliverable is structured, what artifacts are produced, where they live, and how they are reviewed for completeness before Phase 3 is closed. No code in `lib/` or `test/` is modified.

**In scope:**

- `lib/src/ohttp.dart` — the audited subject: `OhttpKeyConfig.parse`, `OhttpKeyConfig.validate`, `ohttpEncapsulate`, `ohttpDecapsulate`, `_buildHpkeInfo`, `_buildRequestHeader`, `OhttpEncapsulateResult`.
- `test/ohttp_test.dart` — referenced as supporting evidence for the negative-case and round-trip test gap (read-only).
- RFC 9458 §4.1 (KeyConfig wire layout), §4.3 (request encapsulation, HPKE info string, AAD), §4.4 (response decapsulation, plain HKDF derivation, response nonce) as normative references.
- Cross-references to `lib/src/hpke.dart` (Phase 1 findings F-1, F-2, F-3 inherited), `lib/src/bhttp.dart` (Phase 2 findings R-3 inherited), `lib/src/ohttp_client.dart` (call sites for `ohttpEncapsulate`/`ohttpDecapsulate` — deferred to Phase 5), and `lib/ohttp_dart.dart` (re-export surface — deferred to Phase 6).

**Explicitly out of scope for Phase 3:**

- `lib/src/hpke.dart` (Phase 1 — complete).
- `lib/src/bhttp.dart` (Phase 2 — complete).
- `lib/src/ohttp_client.dart` orchestration, timeouts, retry, KeyConfig caching (Phase 5).
- Privacy / documentation / `package:cryptography` transitive dependency review (Phase 6).
- Multi-cipher-suite negotiation logic (the silent-drop finding O-2 documents the gap but the negotiation design is not produced here).
- Receiver-side / gateway-role OHTTP (sender-only library, per CLAUDE.md).
- Any source edits, test additions, or `pubspec.yaml` changes.

---

## Components

Because this is a read-only audit, "components" here means **deliverable artifacts** rather than runtime modules.

| Component | Path | Responsibility |
|---|---|---|
| Phase 3 PRD | `specs/.current/AW-2865/phase-3/prd.md` | Audit goals, user stories, scenarios, success criteria, resolved open questions. Source of truth for "done". |
| Phase 3 Research | `specs/.current/AW-2865/phase-3/research.md` | Findings O-1..O-11, limitations/risks table, cross-references to Phase 1 and Phase 2, draft task entries (TASK-O1..TASK-O7), new technical questions. Canonical audit output. |
| Phase 3 Tasks | `specs/.current/AW-2865/phase-3/tasks.md` | Checklist driving the audit (tasks 3.1–3.11). Closed when all items are checked. |
| Phase 3 Plan (this file) | `specs/.current/AW-2865/phase-3/plan.md` | Defines deliverable structure, completeness gates, handoff to Phase 3 QA and Phase 5. |
| Phase 3 QA | `specs/.current/AW-2865/phase-3/qa.md` (later) | Verifies every PRD success criterion is met by the research output. |

**Audited subject (read-only, UNCHANGED):**

| File | Line ranges of interest | Role |
|---|---|---|
| `lib/src/ohttp.dart` | 32–79 (`OhttpKeyConfig.parse`), 81–97 (`OhttpKeyConfig.validate`), 104–120 (`OhttpEncapsulateResult`), 131–167 (`ohttpEncapsulate`), 174–226 (`ohttpDecapsulate`), 232–245 (`_buildHpkeInfo`), 249–257 (`_buildRequestHeader`) | Subject of every Phase 3 finding. |
| `test/ohttp_test.dart` | Whole file | Referenced for negative-case coverage gap; underpins finding O-11. |
| `lib/src/hpke.dart` | `HpkeSender.setupBaseS`, `HpkeSender.hkdfExtract`, `HpkeSender.hkdfExpand`, `HpkeSenderContext` | Layer-1 collaborator; referenced for cross-layer findings (zeroization, `testKeyPair` injection). |
| `lib/ohttp_dart.dart` | Line 10 (re-export of `hpke.dart`) | Public API surface; underpins finding O-7 (maintenance hazard) and the `testKeyPair`/HKDF leak called out by Phase 1 F-1. |

---

## Target Interfaces and Contracts

The contracts below are inherited from Phase 1 and Phase 2 deliverable shape and applied to the OHTTP layer.

### Findings contract (`research.md` "Findings" + "Limitations & Risks")

Every finding entry (O-1..O-11) **must** carry:

1. **Finding ID** — `O-1`..`O-11`, monotonically assigned in `research.md`.
2. **File : Line Range** — repo-relative reference into `lib/src/ohttp.dart` (or `test/ohttp_test.dart` for O-11) at the audit commit. Multiple references allowed when a finding spans helpers (e.g., `parse` + `validate` for the exception-type inconsistency O-4).
3. **Code excerpt** — the smallest verbatim slice of `ohttp.dart` that captures the audited behavior, when the verdict is non-trivial.
4. **Verdict** — bold one-liner from the closed set: `COMPLIANT`, `intentional deviation: <reason>`, `guard absent: <description>`, `silent drop: <description>`, or `correct by inspection`.
5. **RFC reference** — RFC 9458 §4.1, §4.3, or §4.4; cross-cited Go reference implementation for the empty-AAD deviation (O-6); RFC 5869 (plain HKDF) and RFC 9180 (labeled HKDF) for O-7 contrast.
6. **Severity** — exactly one of `BLOCKER`, `HIGH`, `IMPROVEMENT`, `MAINTENANCE`, or `—` (positive verdict, no severity needed). Carried 1:1 into the "Limitations & Risks" table.
7. **Cross-phase reference** — if the finding compounds or instantiates a Phase 1 or Phase 2 pattern (zeroization, exception-type inconsistency, test-gap pattern, `RangeError` propagation), the prior phase's finding ID is cited explicitly.

The "Limitations & Risks" table aggregates findings into a single severity-sorted view; every row maps 1:1 to either a positive-verdict finding (positive verdicts are documented but produce no remediation task) or to one of TASK-O1..TASK-O7.

### Draft task contract (`research.md` "Draft Task Entries")

Each non-positive finding produces exactly one draft task (`TASK-O1`..`TASK-O7`) with:

- **Title** — single-line, imperative, scoped (e.g., "Wrap `SecretBoxAuthenticationError` in a library-owned exception").
- **Severity** — copied from the findings table; no "TBD".
- **File / Line range** — primary `ohttp.dart` line range targeted by remediation; secondary files noted where remediation crosses module boundaries (none expected for Phase 3 — all remediation is in `ohttp.dart` per Constraint 8 of the PRD).
- **RFC section** — exactly one normative reference, or "Go reference implementation" for the empty-AAD deviation (O-6 / TASK-O3), or "RFC 5869 / RFC 9180 distinction" for the maintenance hazard (O-7 / TASK-O4).
- **Description** — explicit remediation direction (wrap, comment, zero-fill, unify exception type, add tests), with enough specificity that an engineer can scope the work without a second triage pass.
- **Cross-phase reference** — when the task is the OHTTP-layer instance of a multi-layer pattern, link to the equivalent Phase 1 / Phase 2 task; this is required for TASK-O2 (Phase 1 F-3, Phase 2 R-3), TASK-O6 (Phase 1 F-2), and TASK-O7 (Phase 1 F-6/F-7, Phase 2 R-6).

### Severity policy (inherited from `vision.md §2`)

- `BLOCKER` — must fix before wallet use. **None expected at the OHTTP layer per research.md (confirmed: no BLOCKER findings).**
- `HIGH` — should fix before wallet use. Four findings: O-2, O-9, O-10, O-11.
- `IMPROVEMENT` — desirable but not blocking. Two findings: O-4, O-6.
- `MAINTENANCE` — recommended backlog (a sub-category of IMPROVEMENT used for "do not silently break this in future refactors"). One finding: O-7.

Severity is bound by the PRD's "Resolved Questions"; reviewers do not re-litigate severity during QA — they verify it is present and matches the PRD-anchored value.

---

## Data Flows

Phase 3's "data flow" is the audit pipeline — how a section of `ohttp.dart` becomes a finding, then a draft task, then a row in `tasklist.md` (the Iteration 7 consolidation).

```
+----------------------------------------+
|  Task 3.1 — ast-index outline          |
|    lib/src/ohttp.dart                  |
+----------------------------------------+
                |
                v
+----------------------------------------+
|  Tasks 3.2–3.10                        |
|    Targeted Read slices of ohttp.dart  |
|    + RFC 9458 §4.1 / §4.3 / §4.4 cite  |
|    + Cross-phase reference check       |
+----------------------------------------+
                |
                v
+----------------------------------------+
|  Per-area Finding (O-N) recorded in    |
|  research.md "Findings" section        |
|    - File, line range                  |
|    - Verdict (positive/deviation/gap)  |
|    - RFC section                       |
|    - Severity                          |
|    - Cross-phase reference             |
+----------------------------------------+
                |
                v
+----------------------------------------+
|  Task 3.11 — emit Draft Task entry      |
|  in research.md "Draft Task Entries"   |
|  for every non-positive finding         |
|    TASK-O1..TASK-O7                    |
+----------------------------------------+
                |
                v
+----------------------------------------+
|  Phase 3 QA (phase-3/qa.md)             |
|  cross-checks PRD success criteria      |
|  against the deliverable                |
+----------------------------------------+
                |
                v
+----------------------------------------+
|  Iteration 7: each TASK-O entry         |
|  becomes a row in tasklist.md and a     |
|  follow-up Jira task (out of phase)    |
+----------------------------------------+
```

Coverage map — each PRD scenario maps to exactly one finding (or a positive verdict) and at most one draft task:

| PRD Scenario | Tasks (from `tasks.md`) | Finding | Draft task | Verdict / severity |
|---|---|---|---|---|
| 1 — KeyConfig wire-layout verification | 3.2 | O-1 | — | COMPLIANT |
| 1 (cont.) — silent drop of extra pairs | 3.3 | O-2 | TASK-O1 | HIGH |
| 2 — `RangeError` confirmation | 3.4 | O-3 | — | Parser guards are complete (positive); message-clarity sub-improvement folded into TASK-O7 (test) only |
| 3 — `FormatException` vs `UnsupportedError` | 3.5 | O-4 | TASK-O2 | IMPROVEMENT |
| 4 — HPKE info string construction | 3.6 | O-5 | — | COMPLIANT |
| 5 — Empty AAD in request seal | 3.7 | O-6 | TASK-O3 | IMPROVEMENT (intentional deviation, inline-comment remediation) |
| 6 — Plain HKDF in response decap | 3.8 | O-7 | TASK-O4 | MAINTENANCE (positive verdict; comment-only remediation against future unification) |
| 6 (cont.) — empty AAD on response | 3.9 | O-8 | — | COMPLIANT |
| 7 — `SecretBoxAuthenticationError` leak | 3.10 | O-9 | TASK-O5 | HIGH |
| 8 — Zeroization absence | 3.6/3.7 follow-on | O-10 | TASK-O6 | HIGH (cross-phase: Phase 1 F-2) |
| (additional, surfaced during 3.1) — test gap | 3.11 | O-11 | TASK-O7 | HIGH |

Every PRD success-criteria row in `phase-3/prd.md` (Success / Metrics) is satisfied by this map: every §4 subsection produces a verdict, the empty-AAD and plain-HKDF findings each cite exact `ohttp.dart` lines and RFC sections, parser gaps each have line references, every finding carries one severity, no code changes occur, every draft task is independently actionable, and cross-layer references are present for findings that compound prior phases.

---

## NFR (Non-Functional Requirements for the deliverable)

| Requirement | How it is enforced |
|---|---|
| **Determinism / reproducibility** | Every finding cites a specific line range in `lib/src/ohttp.dart` at the audit commit. Re-running the audit on the same commit must produce the same finding set. Line numbers are anchored to the SHA at branch `feature/AW-2865-investigation-ohttp_dart` (current HEAD `09bd89e` + the phase-3 working tree). |
| **Read-only** | Acceptance gate is `git diff HEAD -- lib/ test/` is empty when Phase 3 closes (PRD success criterion). QA re-checks this. |
| **RFC traceability** | Every finding cites at least one of RFC 9458 §4.1 / §4.3 / §4.4. The two positive verdicts that are not RFC-rooted (the maintenance hazard O-7 and the test-gap O-11) cite RFC 5869 / RFC 9180 distinction and the relevant test-driven RFC sections respectively. |
| **Severity present** | Findings table in `research.md` lists severity for every entry. QA verifies no entry is blank or "TBD". |
| **Cross-phase consistency** | Findings that recur from Phase 1 / Phase 2 cite the prior finding ID. Inverse direction (Phase 1 / Phase 2 findings being re-classified) is out of scope and not allowed. |
| **Path conventions** | All paths in `phase-3/*.md` are repo-relative (`lib/src/ohttp.dart`, not `/Users/...`), per `.claude/agents/docs/path-conventions.md` and CLAUDE.md "Project conventions". |
| **Auditor isolation from `package:cryptography`** | Findings explicitly call out where a `package:cryptography` type leaks into the public API surface (O-9) but do not re-verify any `package:cryptography` primitive. Constraint 5 of the PRD. |

---

## Risks

| ID | Risk | Likelihood | Severity | Mitigation |
|---|---|---|---|---|
| P3-PLAN-R1 | A finding's severity diverges between `research.md` and `prd.md` "Risks" table (`vision.md §5` lists the silent-drop and zeroization concerns as HIGH; the empty-AAD as IMPROVEMENT). | Low | MEDIUM | QA cross-checks the severity column of every finding row against the PRD "Risks" and "Resolved Questions" tables. Any divergence must be reconciled in `research.md` (the source of truth for the deliverable) before close. |
| P3-PLAN-R2 | Line numbers shift if `ohttp.dart` is touched mid-phase (e.g., for an unrelated formatter run). | Low | LOW | Read-only constraint prevents this. QA re-checks `git diff HEAD -- lib/ test/` empty. |
| P3-PLAN-R3 | The `RangeError`-vs-`FormatException` claim from `vision.md §5.1` was inherited from Phase 2 but turns out **not** to be present in `ohttp.dart` (parser guards are complete — see O-3). The deliverable must explicitly document this negative-by-confirmation result, not silently omit it. | Confirmed in research | INFORMATIONAL | O-3 is filed as a positive verdict with explicit narrative; QA validates that O-3 explains why the vision's risk did not materialize at this layer (guard at `ohttp.dart:63` catches both `symLen < 4` and `data.length < offset + symLen`). |
| P3-PLAN-R4 | A reviewer assumes the maintenance-hazard (O-7) is a "remediation-not-required" finding and drops TASK-O4. | Low | MEDIUM | The deliverable contract explicitly defines `MAINTENANCE` as a sub-category of IMPROVEMENT that DOES produce a task (a code-comment task). QA validates TASK-O4 exists. |
| P3-PLAN-R5 | The `testKeyPair` parameter on `ohttpEncapsulate` (`ohttp.dart:134`) is not independently filed as a Phase 3 finding because it carries forward from Phase 1 F-1. A future reviewer might miss this when scoping Layer-3 remediation. | Medium | LOW | Research's "New Technical Questions" entry #4 documents this explicitly; Phase 5 (`ohttp_client.dart` audit) is tasked with confirming whether the wallet production path ever passes a non-null `testKeyPair`. |
| P3-PLAN-R6 | A future Iteration-7 consolidation drops O-11 (test-gap) on the grounds that "tests can be added with the production fix." The deliverable must keep O-11 independently actionable. | Medium | MEDIUM | TASK-O7 enumerates four specific test scenarios (round-trip, auth-failure, multi-suite, truncated-ciphertext) so the work is independently scopable from production fixes. The cross-phase note (Phase 1 F-6/F-7, Phase 2 R-6) reinforces this as a recurring class of work, not a one-off chore. |

---

## Dependencies

**Must be complete before Phase 3 closes:**

1. **Phase 1 (HPKE) complete** — referenced for F-1 (`testKeyPair` injection), F-2 (HpkeSenderContext field zeroization absence), F-3 (HPKE exception-type inconsistency), F-6/F-7 (test-gap pattern). Status per `git log`: complete (`dab8cf7 Add Phase 1 Summary and Review for AW-2865 HPKE Layer Audit`).
2. **Phase 2 (BHTTP) complete** — referenced for R-3 (BHTTP exception-type inconsistency) and R-6 (BHTTP test-gap pattern). Status per `git log`: complete (`09bd89e Add Phase 2 research and summary for BHTTP layer audit against RFC 9292`).
3. **`phase-3/prd.md` approved** — Resolved Questions list is the binding source for severity assignments and remediation scope. Status: PRD_READY.
4. **`phase-3/research.md` complete with O-1..O-11 and TASK-O1..TASK-O7** — Status: COMPLETE (file header at line 6).
5. **`phase-3/tasks.md` checked through 3.11** — Status (per file): unchecked checkboxes — but the content has been produced in `research.md`. QA gate is to mark each box closed when `research.md` covers the corresponding scope. Done as a documentation-only step at phase close.

**Out-of-phase follow-up dependencies (do NOT block Phase 3 close):**

- **Phase 5 (`ohttp_client.dart` audit)** will:
  - Confirm whether `ohttp_client.dart` ever passes a non-null `testKeyPair` (research §New Technical Questions #4).
  - Confirm whether `ohttp_client.dart` wraps or surfaces the inconsistent exception types from `OhttpKeyConfig.parse` / `validate` (research §New Technical Questions #2).
  - Assess the lifetime contract of `OhttpEncapsulateResult` (research §New Technical Questions #3).
- **Phase 6 (privacy / API surface)** will:
  - Re-examine the `lib/ohttp_dart.dart` re-export of `hpke.dart` that exposes unlabeled `hkdfExtract`/`hkdfExpand` to all callers, and decide whether the public API surface should be narrowed.
- **Iteration 7 (consolidation)** will:
  - Convert each `TASK-O*` draft into a Jira ticket and a row in `specs/.current/AW-2865/tasklist.md`.

---

## Alternatives Considered (for ADR — not produced)

The PRD's Resolved Questions section closes the major architectural trade-offs already, so no companion `phase-3/adr.md` is created. The alternatives below are recorded here for traceability; if a future revision reopens any of them, an ADR can be lifted directly from this section.

### A1 — `SecretBoxAuthenticationError` wrapping layer

- **Chosen:** Wrap in `ohttp.dart` (Resolved Question #3).
- **Alternative considered:** Wrap in `ohttp_client.dart` (closer to the wallet caller) or in a shared `lib/src/exceptions.dart`.
- **Trade-off:** Wrapping in `ohttp.dart` keeps the fix co-located with the throw site and avoids a new shared file; the cost is that future layers re-implementing decapsulation (e.g., a hypothetical receiver-side) must repeat the wrap. Acceptable because the package is sender-only by design.

### A2 — `OhttpEncapsulateResult` lifetime / dispose API

- **Chosen:** Best-effort zero-fill before nulling (Resolved Question #4, TASK-O6).
- **Alternative considered:** Adopt a `dispose()`-style contract on `OhttpEncapsulateResult` and enforce single-use via an internal flag.
- **Trade-off:** A `dispose()` API would be more explicit but adds API surface and a stateful object; given the Dart VM's no-guarantee on zero-writes, the marginal cryptographic gain over best-effort zeroization does not justify the API cost in a single-use type. Research §New Technical Questions #3 leaves the door open for Phase 5 to revisit if `ohttp_client.dart` reveals retention bugs.

### A3 — Multi-suite negotiation in `OhttpKeyConfig.parse`

- **Chosen:** Reject `symLen > 4` (or loop and reject all but the supported suite) via TASK-O1.
- **Alternative considered:** Implement full multi-suite negotiation now (loop, pick the preferred supported pair, fall back).
- **Trade-off:** Full negotiation requires adding suite-selection state and is out of scope for a single-fixed-cipher-suite library (CLAUDE.md). Rejecting `symLen > 4` is the minimal correct behavior pending an explicit decision to add suites.

### A4 — Empty-AAD documentation scope

- **Chosen:** Inline comment at `ohttp.dart:149` (Resolved Question #2, TASK-O3).
- **Alternative considered:** Dedicated ADR documenting the Go-interop justification and the rejected RFC reading.
- **Trade-off:** An ADR would over-document a small, well-understood deviation already noted in `CLAUDE.md`. The inline comment is the appropriate weight.

---

## Open Questions

All PRD-level open questions are resolved (per `phase-3/prd.md` Resolved Questions §1–§4 and `phase-3/research.md` §Resolved Questions §1–§6).

Three deferred technical questions are documented in `phase-3/research.md` "New Technical Questions" §2, §3, §4. They are **explicitly out of scope for Phase 3** (each is assigned to Phase 5 or noted as cross-phase informational) and do not block this plan's close.

---

## Acceptance / Close Criteria for Phase 3

Phase 3 closes when **all** of the following are true:

1. `specs/.current/AW-2865/phase-3/research.md` contains O-1..O-11 and TASK-O1..TASK-O7 with the contract above (status header: COMPLETE).
2. `specs/.current/AW-2865/phase-3/tasks.md` items 3.1–3.11 are all checked.
3. `git diff HEAD -- lib/ test/` is empty (read-only constraint).
4. `specs/.current/AW-2865/phase-3/qa.md` is produced and confirms every PRD success-metric row is satisfied.
5. Every finding row in the "Limitations & Risks" table of `research.md` carries a severity, a file:line, and an RFC section (or explicit non-RFC justification for O-7 maintenance and O-11 test-gap).
6. Cross-phase references are present for O-3 (parser-guard family), O-4 (exception inconsistency), O-10 (zeroization), and O-11 (test-gap) — these are the four findings that compound prior-phase patterns.

The plan is then handed to the QA agent for `phase-3/qa.md` generation.
