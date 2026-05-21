# Phase 1 Plan: HPKE Layer Audit Against RFC 9180

**Ticket:** AW-2865 — ohttp_dart investigation
**Phase:** 1 of N (HPKE layer)
**Status:** PLAN_APPROVED
**Inputs:** `phase-1/prd.md`, `phase-1/research.md`, `phase-1/tasks.md`, ticket-wide `idea.md`, `vision.md`

---

## Phase Scope

This phase produces a **read-only audit deliverable** for `lib/src/hpke.dart` (the HPKE Base Mode Sender, RFC 9180). The deliverable is a set of severity-classified, independently actionable draft Jira task entries — already captured in `phase-1/research.md` (sections "Findings Table" and "Draft Task Entries").

No code in `lib/` or `test/` is modified in this phase. The plan below describes how the audit deliverable is structured, what artifacts are produced, where they live, and how they are reviewed for completeness before Phase 1 is closed.

**Explicitly out of scope for Phase 1:**

- `lib/src/bhttp.dart`, `lib/src/ohttp.dart`, `lib/src/ohttp_client.dart` (later phases).
- Plain (unlabeled) `HpkeSender.hkdfExtract` / `HpkeSender.hkdfExpand` as consumed by `ohttp.dart` response decapsulation — reviewed in Phase 3.
- HPKE receiver-side (Decap, `key_schedule_r`) — not implemented in the package.
- Re-verifying `package:cryptography` primitives (X25519, HMAC-SHA-256, AES-128-GCM).
- Any source edits, test additions, or `pubspec.yaml` changes.

---

## Components

Because this is a read-only audit, "components" here means **deliverable artifacts** rather than runtime modules.

| Component | Path | Responsibility |
|---|---|---|
| Phase 1 PRD | `specs/.current/AW-2865/phase-1/prd.md` | Audit goals, user stories, success criteria. Source of truth for what counts as "done". |
| Phase 1 Research | `specs/.current/AW-2865/phase-1/research.md` | Section-by-section RFC verdicts, structured findings table, and per-finding draft Jira task entries. Canonical audit output. |
| Phase 1 Tasks | `specs/.current/AW-2865/phase-1/tasks.md` | Checklist driving the audit (tasks 1.1–1.10). Closed when all items are checked. |
| Phase 1 Plan (this file) | `specs/.current/AW-2865/phase-1/plan.md` | Defines deliverable structure, completeness gates, and handoff to phase QA / later phases. |
| Phase 1 QA | `specs/.current/AW-2865/phase-1/qa.md` (later) | Verification that every PRD success criterion is met by the research output. |

**Audited subject (read-only, UNCHANGED):**

| File | Lines | Role |
|---|---|---|
| `lib/src/hpke.dart` | 1–314 | HPKE Base Mode Sender. The subject of every Phase 1 finding. |
| `test/hpke_test.dart` | (referenced) | RFC 9180 A.1 vector tests. Referenced as supporting evidence; not modified. |
| `lib/src/ohttp.dart` | 113–149 (referenced only) | Consumer of `HpkeSender.setupBaseS` / `ctx.seal` / `ctx.export`. Referenced where a finding spans the HPKE/OHTTP boundary (`testKeyPair` threading, `exportedSecret` retention). |
| `lib/ohttp_dart.dart` | (referenced) | Public barrel re-export. Referenced for the "unlabeled HKDF on public API" finding. |

---

## Target Interfaces and Contracts

### Findings contract (research.md "Findings Table")

Every row in the findings table **must** carry exactly these columns:

1. **#** — stable finding ID (`F-1`, `F-2`, …) for cross-reference from later phases.
2. **Description** — concrete behavioral statement, not a generality. Includes both the observation and its consequence in the wallet threat model.
3. **File : Line Range** — absolute reference into the repository at the audit commit. Multiple references allowed when a finding spans files.
4. **RFC Section** — RFC 9180 section number (and, where relevant, RFC 9458 / RFC 5869). One section per row when possible; multiple allowed.
5. **Severity** — exactly one of `BLOCKER`, `HIGH`, `IMPROVEMENT`. No "TBD" or compound severities. Severities are assigned per the resolved-questions table in research §"Resolved Questions".

### Draft Jira task contract (research.md "Draft Task Entries")

Each finding produces exactly one draft task (`TASK-N`) with:

- **Title** — single-line, imperative, scoped (e.g., "Replace `testKeyPair` injection with a test-only factory").
- **Severity** — copied from the findings table.
- **File** — primary file:line targeted by the remediation.
- **RFC section** — same as findings table.
- **Description** — what is observed, why it is a risk, and one or more concrete remediation directions. The description must be sufficient for an unrelated engineer to scope the work without re-running the audit.

### Section-verdict contract (research.md "RFC 9180 Section Verdicts")

Each audited RFC section (§4, §4.1, §5.1, §5.2, §5.3) carries:

- The relevant RFC pseudo-code block (quoted).
- The corresponding implementation location (`hpke.dart` line range).
- A line-by-line correctness statement.
- A **Verdict** line in bold: `matches RFC` or `deviates: <description>`. No section may be skipped.

---

## Data Flow (audit pipeline)

```
[ tasks.md checklist (1.1–1.10) ]
                |
                v
[ Read hpke.dart via ast-index outline + targeted Read slices ]
                |
                v
[ Section-by-section RFC comparison ]
                |       (§4, §4.1, §5.1, §5.2, §5.3)
                v
[ research.md "RFC 9180 Section Verdicts" populated ]
                |
                v
[ Scrutiny-table sweep (vision §4 row for hpke.dart) ]
                |       (single-suite hard-coding, seq overflow, testKeyPair)
                v
[ research.md "Findings Table" populated (F-1 .. F-N) ]
                |
                v
[ Each finding -> draft task entry ]
                |
                v
[ research.md "Draft Task Entries" (TASK-1 .. TASK-N) ]
                |
                v
[ Completeness check vs PRD success criteria + vision scrutiny table ]
                |
                v
[ Phase 1 QA + handoff to Phase 2 (BHTTP) ]
```

The pipeline is **strictly read-only at every stage**. The only writes are to `specs/.current/AW-2865/phase-1/*.md`.

---

## Non-Functional Requirements

| NFR | Requirement | How it is met |
|---|---|---|
| **Read-only guarantee** | `git diff HEAD -- lib/ test/ pubspec.yaml example/` must be empty at phase end. | No write tool calls target those paths. Phase-1 QA verifies the diff. |
| **Traceability** | Every finding traces to (a) a file:line range, (b) an RFC section, (c) a severity. | Findings-table contract above; QA verifies no row is missing a column. |
| **Coverage** | Every row of vision §4 scrutiny table for `hpke.dart` has either a finding or an explicit "no issue found" note. | Scrutiny-table sweep is a named step in the pipeline; QA cross-checks. |
| **Independent actionability** | Each draft task can be picked up by a single engineer without depending on another in-flight Phase 1 task. | Verified during draft-task review; merged where two findings are inseparable, with a note. |
| **Severity discipline** | Exactly one of `BLOCKER` / `HIGH` / `IMPROVEMENT` per task. No "TBD". | Resolved-questions table fixes ambiguous cases (testKeyPair = HIGH; zeroization = IMPROVEMENT; seq overflow = IMPROVEMENT). |
| **No leakage to other phases** | Findings about `bhttp.dart`, `ohttp.dart`, `ohttp_client.dart` are deferred and noted in "New Technical Questions", not added to Phase 1 task entries. | Verified during draft-task review. Cross-phase notes (e.g., F-7 references RFC 9458) are explicitly labeled as such. |
| **RFC citation accuracy** | Every cited RFC section must be the section that actually defines the behavior under audit. | Verdict-section pseudo-code is quoted from RFC 9180 directly; cross-check against the canonical RFC text. |

---

## Risks

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-1 | Findings missed because `hpke.dart` has changed since `vision.md` was authored. | Low | Medium | Task 1.1 mandates a full outline read of `hpke.dart` at audit time, not a reliance on vision notes. |
| R-2 | Severity drift between findings (e.g., one finding inflated to HIGH because it "feels" important). | Medium | Medium | Severity bar fixed by vision §2 + research "Resolved Questions" table. Reviewer cross-references both before sign-off. |
| R-3 | Cross-phase findings (HPKE/OHTTP boundary) accidentally close issues that Phase 3 should own. | Medium | Low | F-7 explicitly notes "covered as a Phase 3 finding but noted here as a HPKE-layer test gap"; this annotation pattern is used for all cross-phase items. |
| R-4 | Draft task descriptions are too vague to act on, requiring re-investigation later. | Low | Medium | Draft-task contract above mandates concrete remediation directions in every entry. |
| R-5 | Citation errors against RFC 9180 weaken the credibility of the deliverable. | Low | High | Each verdict section quotes the RFC pseudo-code verbatim and gives a line-by-line correspondence to the implementation. |
| R-6 | `package:cryptography` upgrade between audit and remediation invalidates findings. | Low | Low | Limitations & Risks section in research notes the trust boundary; pinning is a separate task (deferred to a later phase / cross-cutting concern). |

---

## Alternatives Considered

**A1. Inline findings inside each task file (one Markdown per task).**
Rejected: would create N small files for 7 findings, fragmenting review. Research-table-of-findings + draft-task list keeps the audit deliverable in one document while preserving 1:1 mapping to future Jira tasks. Resolved-questions table (research §"Resolved Questions" Q6) confirms this choice.

**A2. Defer severity assignment to a separate triage phase.**
Rejected: PRD success criterion explicitly requires severity on every finding. Open Questions in the PRD were resolved by the user before research started (see research "Resolved Questions" Q1–Q3).

**A3. Treat all `IMPROVEMENT` findings as a single combined task.**
Rejected: violates the "independently actionable" principle from vision §2.5. Each `IMPROVEMENT` is scoped to a single concern (zeroization, typed exception, nonce XOR width, public-API surface) and can be picked up independently.

**A4. Skip the section-verdict block, jump straight to findings.**
Rejected: PRD success criterion "All RFC 9180 sections covered" requires a positive or negative verdict per section. The verdict block is the evidence that the analyst actually read each section, not just the lines that flagged a finding.

No alternatives reach the bar of requiring a separate ADR document — see ADR section below.

---

## Dependencies

**Prior phases:** None. This is Phase 1.

**Concurrent/inherited inputs:**

- `specs/.current/AW-2865/idea.md` — ticket-wide motivation and acceptance criteria.
- `specs/.current/AW-2865/vision.md` — severity model (§2), architecture (§4), data model (§5), scrutiny table (§4).
- `specs/.current/AW-2865/phase-1/prd.md` — phase goals, scenarios, success criteria.
- `specs/.current/AW-2865/phase-1/research.md` — audit output.
- `specs/.current/AW-2865/phase-1/tasks.md` — checklist driving the audit.
- `lib/src/hpke.dart` — read-only subject.
- `test/hpke_test.dart` — read-only supporting evidence (RFC 9180 A.1 vector coverage).
- `lib/src/ohttp.dart`, `lib/ohttp_dart.dart` — read-only references for cross-boundary findings.
- RFC 9180, RFC 9458 (cited), RFC 5869 (cited via HKDF references).

**Downstream consumers (later phases):**

- Phase 2 (`bhttp.dart` audit) consumes nothing from Phase 1 directly but follows the same deliverable structure.
- Phase 3 (`ohttp.dart` audit) will revisit cross-boundary findings F-1 (testKeyPair threading), F-2 (`OhttpEncapsulateResult.exportedSecret` zeroization), F-5 (public API surface), and F-7 (empty-AAD integration test). The Phase 1 research file is the authoritative reference for those rows; Phase 3 must cite the `F-N` IDs to keep the audit traceable.
- The eventual "compile draft entries into Jira tickets" task (cross-phase, not yet scheduled) consumes the "Draft Task Entries" section of every phase's research file.

---

## Open Questions (Phase 1)

The PRD's four Open Questions were resolved before research began (see research `Resolved Questions` table). New questions surfaced during the audit are recorded in research §"New Technical Questions" and deferred:

1. `lib/ohttp_dart.dart` re-export surface composition — Phase 3.
2. `package:cryptography` version pinning and CVE review — cross-cutting / supply-chain phase.
3. Web/WASM target and `dart2js` integer semantics for `_seq` — out of scope unless wallet target adds web.
4. `HpkeSenderContext.export` `length` parameter upper bound (`L <= 255 * Nh`) — candidate Phase 1 IMPROVEMENT addendum or Phase 3 finding (currently filed under "New Technical Questions" pending a decision on whether to add F-8).

None of these block closing Phase 1 — they are recorded for downstream phases.

---

## Definition of Done (Phase 1)

Phase 1 closes when **all** of the following hold:

1. `tasks.md` items 1.1–1.10 are checked.
2. `research.md` contains:
   - RFC verdicts for §4, §4.1, §5.1, §5.2, §5.3 (no skipped sections).
   - A findings table with at least one row per scrutiny-table concern (single-suite, seq overflow, testKeyPair) or an explicit "no issue found" note.
   - One draft task entry per findings row, conforming to the draft-task contract above.
3. `git diff HEAD -- lib/ test/ pubspec.yaml example/` is empty.
4. Phase 1 QA report (`phase-1/qa.md`) confirms each PRD success criterion is met.
5. No finding row has an unset severity.
6. Cross-boundary findings are explicitly annotated with the phase that owns the remediation.

---

## ADR

No standalone ADR is produced for Phase 1. The alternatives considered (deliverable format, severity timing, finding aggregation, verdict structure) are routine audit-shape decisions and are documented inline in the "Alternatives Considered" section above. No architectural trade-off in the package itself is decided in this phase — the phase produces findings only, not design choices.

If a Phase 1 finding later motivates an architectural change (e.g., introducing a typed exception hierarchy, or splitting the HPKE Sender into a labeled-only public API plus a package-private unlabeled HKDF), that change will be planned and ADR'd in the implementing phase, not here.
