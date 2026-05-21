# Phase 5 Plan: Test Suite Coverage Gap Review

**Ticket:** AW-2865 — ohttp_dart investigation
**Phase:** 5 of 7 (test suite coverage audit)
**Status:** PLAN_APPROVED
**Inputs:** `specs/.current/AW-2865/phase-5/prd.md`, `specs/.current/AW-2865/phase-5/research.md`, `specs/.current/AW-2865/phase-5/tasks.md`, ticket-wide `idea.md`, `vision.md`, prior phase summaries in `specs/.current/AW-2865/phase-{1..4}/summary.md`

---

## Phase Scope

This phase produces a **read-only audit deliverable** covering the entire test suite of `ohttp_dart` — all three test files (`test/hpke_test.dart`, `test/bhttp_test.dart`, `test/ohttp_test.dart`). The deliverable is a set of severity-classified, independently actionable draft task entries — captured in `phase-5/research.md` (TASK-T1..TASK-T13 and the cross-phase confirmation table).

No code in `lib/`, `test/`, `pubspec.yaml`, or `example/` is modified in this phase. The plan below describes how the audit deliverable is structured, what artifacts are produced, where they live, and how they are reviewed for completeness before Phase 5 is closed.

**Explicitly in scope for Phase 5:**

- `test/hpke_test.dart` (296 lines) — RFC 9180 Appendix A.1 vector tests; `testKeyPair` injection hook; sequence-number overflow absence; export-edge-case absence.
- `test/bhttp_test.dart` (231 lines) — QUIC varint boundary coverage; truncated-input negative tests; 8-byte varint absence.
- `test/ohttp_test.dart` (164 lines) — KeyConfig parse tests; encapsulate structural check; decapsulate format-guard tests; gaps for auth-failure, round-trip, and multi-suite KeyConfig.
- Cross-phase confirmation of all named gaps from Phases 1–4 (30 items) — each confirmed present or absent by direct test-file inspection.
- New gaps identified (NG-1..NG-10) not named in prior phases.
- Fuzz/property-based test absence (confirmed, classified HIGH).
- Live-gateway integration test absence (confirmed, classified HIGH).

**Explicitly out of scope for Phase 5:**

- `lib/src/*.dart` — covered by Phases 1–4; only referenced to ground severity of a test gap against the implementation finding it would detect.
- `package:http` and `package:cryptography` internals.
- Adding or modifying any test.
- Prescribing a specific fuzz testing framework.
- Designing the test infrastructure for live-gateway integration (e.g., choosing a mock vs. real gateway endpoint).
- Phase 6 (privacy/documentation audit) and Phase 7 (task compilation into per-task Markdown files under `specs/.current/AW-2865/tasks/`).

---

## Components

Because this is a read-only audit, "components" means **deliverable artifacts** rather than runtime modules.

| Component | Path | Responsibility |
|---|---|---|
| Phase 5 PRD | `specs/.current/AW-2865/phase-5/prd.md` | Audit goals, four scenarios, success criteria, risks, resolved questions. Source of truth for what counts as "done". |
| Phase 5 Research | `specs/.current/AW-2865/phase-5/research.md` | Test file coverage summaries, varint boundary table, cross-phase confirmation table (30 items), new gaps (NG-1..NG-10), draft task entries (TASK-T1..TASK-T13). Canonical audit output. |
| Phase 5 Tasks | `specs/.current/AW-2865/phase-5/tasks.md` | Checklist driving the audit (tasks 5.1–5.11). Closed when all items are checked. |
| Phase 5 Plan (this file) | `specs/.current/AW-2865/phase-5/plan.md` | Defines deliverable structure, completeness gates, cross-layer references, and handoff to Phase 6 and Iteration 7. |
| Phase 5 QA | `specs/.current/AW-2865/phase-5/qa.md` (later) | Verification that every PRD success criterion is met by the research output. |
| Phase 5 Summary | `specs/.current/AW-2865/phase-5/summary.md` (later) | Concise close-out matching the structure used by Phases 1–4. |

**Audited subjects (read-only, UNCHANGED):**

| File | Lines | Role |
|---|---|---|
| `test/hpke_test.dart` | 296 | RFC 9180 Appendix A.1 vector suite for `hpke.dart`. |
| `test/bhttp_test.dart` | 231 | Varint + framing tests for `bhttp.dart`. |
| `test/ohttp_test.dart` | 164 | KeyConfig parse, encap, decap tests for `ohttp.dart` (with incidental Layer 4 coverage). |

---

## Target Interfaces and Contracts

### Draft task contract (research.md §"Draft Task Entries")

Each draft task entry (`TASK-T1`..`TASK-T13`) carries exactly these fields:

1. **Task ID** — `TASK-T{N}`, sequentially numbered.
2. **File** — the test file where the gap lives (e.g., `test/hpke_test.dart`).
3. **Missing scenario** — a concrete test scenario name, stated precisely enough that an engineer can write the test without re-reading the audit.
4. **Coverage stops at** — the line number in the existing test file where coverage ends, or "N/A — not tested at all" for entirely uncovered classes.
5. **Proposed group name** — the `group(...)` label for the new test.
6. **Severity** — exactly one of `BLOCKER` / `HIGH` / `IMPROVEMENT` (vision §2 model).
7. **Cross-phase reference** — the prior finding ID (e.g., `F-6`, `R-6`, `TASK-O7 item 2`) that this test gap confirms, or `New gap (NG-N)` for gaps first identified in Phase 5.

### Cross-phase confirmation contract

The cross-phase confirmation table in `research.md` covers all 30 named gaps from Phases 1–4. Each row is either:
- **Still absent** — no test exists; gap confirmed.
- **Covered** — an existing test exercises the path; gap is not real (none found in Phase 5).
- **Partial** — some boundary values covered, but not all; row documents the covered sub-set.

The contract: confirmed gaps produce draft task entries; partial gaps produce draft task entries that scope only the uncovered sub-cases (see TASK-O7 item 4 → TASK-T9).

### Eight investigation vectors coverage contract

All eight vectors from `idea.md` must appear across the 13 draft task entries:

| Vector | Task(s) |
|---|---|
| Cryptographic correctness | TASK-T1, TASK-T2, TASK-T3 |
| Interoperability | TASK-T10, TASK-T13 |
| Parser robustness | TASK-T4, TASK-T5, TASK-T6, TASK-T7, TASK-T8 |
| Privacy risks | TASK-T11 (sendDirect and scheme-enforcement paths) |
| Network reliability | TASK-T11 (timeout/retry simulation via MockClient) |
| KeyConfig management | TASK-T8, TASK-T10 |
| Test suite | TASK-T12 (fuzz/property coverage) |
| Documentation | TASK-T11 (sendDirect bypass — tested via behavior, not doc assertion) |

---

## Data Flow (audit pipeline)

```
[ tasks.md checklist (5.1–5.11) ]
                |
                v
[ Read each test file in full; record line counts and group structure ]
                |
                v
[ hpke_test.dart: map RFC 9180 A.1 sub-vectors to assertions;
  identify missing negative/edge cases ]
                |
                v
[ bhttp_test.dart: map varint boundary values to test coverage;
  identify truncated-input and unknown-framing gaps ]
                |
                v
[ ohttp_test.dart: map each test group to a code path in ohttp.dart;
  identify missing negative cases, round-trip, auth-failure ]
                |
                v
[ Cross-phase pass: confirm or refute each of the 30 named gaps
  from Phases 1–4 by direct test-file citation ]
                |
                v
[ New-gap pass: identify gaps not named in prior phases (NG-1..NG-10) ]
                |
                v
[ Each gap -> draft task entry (TASK-T1..TASK-T13) ]
                |
                v
[ Coverage check vs PRD success criteria and eight investigation vectors ]
                |
                v
[ Phase 5 QA + handoff to Phase 6 and Iteration 7 (task compilation) ]
```

The pipeline is **strictly read-only at every stage**. The only writes are to `specs/.current/AW-2865/phase-5/*.md`.

---

## Non-Functional Requirements

| NFR | Requirement | How it is met |
|---|---|---|
| **Read-only guarantee** | `git diff HEAD -- lib/ test/ pubspec.yaml example/` must be empty at phase end. | No write tool calls target those paths. Phase-5 QA verifies the diff. |
| **Traceability** | Every draft task entry traces to (a) a specific test file, (b) a line range where coverage stops, (c) a severity, (d) a cross-phase finding ID or new-gap ID. | Draft task contract above; QA verifies no field is missing. |
| **Coverage** | Every PRD success criterion and every vision §4 scrutiny row for test coverage is accounted for. | Eight-vector coverage table above; QA cross-checks. |
| **Independent actionability** | Each TASK-T entry can be picked up by a single engineer without depending on another in-flight Phase 5 task. | Verified during draft-task review. Tasks that share a test group are flagged as merge candidates (TASK-T5 and TASK-T6 share the `parseResponse` area but are kept separate because their scenario classes differ). |
| **Severity discipline** | Exactly one of `BLOCKER` / `HIGH` / `IMPROVEMENT` per task. No "TBD". | All 13 tasks carry resolved severities; PRD §"Constraints" prohibits "TBD" labels. |
| **No leakage** | Phase 5 does not re-audit implementation files; cross-layer references cite the lower-layer finding ID (e.g., F-6, R-6, O-9) and do not produce duplicate findings. | Confirmed: research.md §"Cross-Phase Gap Confirmation Table" is the only source of cross-phase assertions. |
| **No framework prescription** | Fuzz/property test tasks (TASK-T12) must not name a specific Dart fuzz testing framework or version. | PRD §"Constraints" constraint 6 enforced; research.md TASK-T12 entry is framework-agnostic. |
| **Severity alignment** | A test-gap severity must not be lower than the severity of the implementation finding it would detect. | Each HIGH or BLOCKER test-gap entry in the research cross-checks against the severity of the underlying implementation finding. |

---

## Risks

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-1 | A test gap named in a prior phase turns out to be covered by an existing test overlooked in that phase. | Low | Moderate — severity label would need downgrading. | Phase 5 reads each test file in full before asserting gaps; any already-covered case is excluded from TASK-T entries (none were found). |
| R-2 | The cross-phase table covers 30 items but a Phase 1–4 finding was inadvertently omitted. | Low | Low — one extra task created. | `research.md` §"Cross-Phase Gap Confirmation Table" lists the finding ID of every prior-phase task; QA spot-checks the count against phase-1..4 tasks.md. |
| R-3 | TASK-T5 and TASK-T6 are flagged as mergeable but are kept separate, inflating Iteration 7 work. | Low | Low — aggregation at Iteration 7. | Both entries carry distinct scenario classes; the decision to merge or not is deferred to Iteration 7 and recorded as an explicit note in the plan (not a silent omission). |
| R-4 | TASK-T12 (fuzz/property tests) is too broad to be independently actionable without a framework choice. | Moderate | Moderate — engineer cannot start without a framework decision. | TASK-T12 is structured as three sub-scenarios that are each testable with any property framework; framework choice is explicitly deferred to the implementing engineer. |
| R-5 | TASK-T13 (live-gateway integration) cannot be validated if no public OHTTP gateway is reachable. | Low | Moderate — task cannot be exercised end-to-end. | PRD §"Assumptions" assumption 6 classifies this as HIGH (not BLOCKER) and notes the mock-gateway fallback; TASK-T13 must specify both paths. |
| R-6 | A deliberate test omission (e.g., not testing `sendDirect()` because it is a bypass) is misidentified as a gap. | Low | Low — extra IMPROVEMENT task. | Ambiguous cases are marked IMPROVEMENT and the ambiguity noted in the task description. TASK-T11 is marked HIGH because the underlying findings (C-1..C-10) are HIGH. |

---

## Alternatives Considered

**A1. One TASK-T file per gap produced here in Phase 5.**
Rejected: Phase 5 research.md is the correct artifact; per-task Markdown files are produced by Iteration 7 only. This keeps the audit pipeline consistent with Phases 1–4.

**A2. Merge all BHTTP truncated-input gaps into a single task entry.**
Rejected at this stage: TASK-T5 (truncated body/header/varint/empty-buffer) and TASK-T6 (unknown framing indicator) have different root causes (parser length guards vs. indicator validation) and different remediation sites in `bhttp.dart`. They are kept separate so Iteration 7 can make an informed merge decision.

**A3. Skip cross-phase confirmation for gaps already confirmed in prior phases.**
Rejected: PRD §"Scenario D" requires each named gap to be re-confirmed by Phase 5 test-file inspection. This prevents false-positives from slipping through if a test was added between phases. The confirmation table is the canonical cross-phase evidence record.

**A4. Classify missing live-gateway integration tests as IMPROVEMENT because public gateways may not be reachable.**
Rejected: PRD §"Resolved Questions" Q2 explicitly classifies this as HIGH. The empty-AAD deviation (Finding O-6, Phase 3) could fail silently against a conformant gateway with no existing test; this is a wire-format correctness risk, not a quality improvement.

**A5. Prescribe `dart test --coverage` or a specific fuzz library for TASK-T12.**
Rejected: PRD §"Constraints" constraint 6 prohibits framework prescription. The absence of a CI coverage tool is addressed by the manual full-file reading approach documented in the research.

No alternative reaches the bar of requiring a standalone ADR — see ADR section.

---

## Dependencies

**Prior phases (must be complete before Phase 5 can close):**

- **Phase 1 (`hpke.dart` audit) — COMPLETE.** Cross-referenced: F-6 (sequence-number overflow untested), F-7 (no integration test). Both confirmed in Phase 5 research.
- **Phase 2 (`bhttp.dart` audit) — COMPLETE.** Cross-referenced: R-6 (no negative-case tests for truncated body, unknown framing, malformed header), F2.3..F2.7 (status-code, header-cap, body-cap, varint-overflow findings). All confirmed in Phase 5 research.
- **Phase 3 (`ohttp.dart` audit) — COMPLETE.** Cross-referenced: TASK-O7 items 1–4 (round-trip, auth-failure, multi-suite KeyConfig, truncated ciphertext). All confirmed in Phase 5 research.
- **Phase 4 (`ohttp_client.dart` audit) — COMPLETE.** Cross-referenced: C-1..C-10 (no `OhttpClient` tests whatsoever). Confirmed in Phase 5 research.

**Concurrent/inherited inputs:**

- `specs/.current/AW-2865/idea.md` — ticket-wide motivation; eight investigation vectors.
- `specs/.current/AW-2865/vision.md` — severity model (§2), architecture (§4), logging constraints (§7).
- `specs/.current/AW-2865/phase-5/prd.md` — phase goals, four scenarios, success criteria, risks, resolved questions.
- `specs/.current/AW-2865/phase-5/research.md` — audit output (coverage summaries, 30-item cross-phase table, NG-1..NG-10, TASK-T1..TASK-T13).
- `specs/.current/AW-2865/phase-5/tasks.md` — checklist (5.1–5.11).
- `test/hpke_test.dart`, `test/bhttp_test.dart`, `test/ohttp_test.dart` — read-only subjects.
- CLAUDE.md — confirms three test files represent the entire test suite; confirms no live-gateway integration tests.

**Downstream consumers:**

- **Phase 6 (privacy/documentation audit)** — consumes Phase 5 findings for observability gap confirmation. Several Phase 5 test gaps (TASK-T11: no `sendDirect()` test; TASK-T13: no live-gateway test) compound Phase 6 privacy concerns.
- **Phase 7 / Iteration 7 (task compilation)** — consumes the `research.md §"Draft Task Entries"` table directly to produce per-task Markdown files under `specs/.current/AW-2865/tasks/`. TASK-T entries feed into that compilation alongside TASK-H, TASK-B, TASK-O, and TASK-C entries from prior phases.
- **AW-2857 (blocked-by relationship)** — the consuming Jira ticket. Phase 5 deliverables contribute to the AW-2857 production-readiness checklist for the wallet integration.

---

## Open Questions (Phase 5)

All PRD open questions were resolved before Phase 5 research began (see `prd.md §"Resolved Questions"`). No new blocking questions surfaced during research. The following non-blocking notes are recorded for downstream visibility:

1. **TASK-T5 / TASK-T6 merge decision.** These two entries both target the `parseResponse` area of `bhttp_test.dart` but address different root causes. Iteration 7 may merge them into a single task; if so, the merged task must preserve both scenario sets.
2. **TASK-T11 file placement.** The entry specifies "new `test/ohttp_client_test.dart` or appended to `test/ohttp_test.dart`". The implementing engineer decides; both placements are acceptable per CLAUDE.md conventions (no `test/` structure constraint documented).
3. **TASK-T12 iteration count.** The fuzz/property task deliberately omits a minimum property-test iteration count. If a CI time budget requires a cap, the implementing engineer must add it; the audit does not prescribe it.
4. **`ohttpDecapsulate` partial coverage (TASK-O7 item 4).** The 17-byte `encResponse` test covers one value in the 17–31-byte range. The gap covers the remaining values (18–31 bytes and exactly 31 bytes). Iteration 7 should decide whether this is a distinct task or a sub-case of TASK-T9.

None of these block closing Phase 5.

---

## Definition of Done (Phase 5)

Phase 5 closes when **all** of the following hold:

1. `tasks.md` items 5.1–5.11 are checked.
2. `research.md` contains:
   - Test file coverage summaries for all three files with actual line counts.
   - Test group tables enumerating every `group(...)` and its test count.
   - RFC 9180 Appendix A.1 sub-vector coverage table (with line references).
   - Varint boundary coverage table (with explicit YES/NO per boundary value).
   - Cross-phase gap confirmation table with 30 rows, each marked `Still absent`, `Covered`, or `Partial`.
   - New gaps section (NG-1..NG-10) for gaps not named in prior phases.
   - Draft task entries (TASK-T1..TASK-T13) — each with file, missing scenario, coverage stop line, proposed group name, severity, and cross-phase reference.
3. `git diff HEAD -- lib/ test/ pubspec.yaml example/` is empty.
4. Phase 5 QA report (`phase-5/qa.md`) confirms each PRD success criterion is met.
5. No draft task entry has an unset severity, an unset file reference, or an unset cross-phase reference.
6. All eight investigation vectors from `idea.md` appear across the 13 draft task entries.
7. At least one draft task entry is labelled `HIGH` (no `BLOCKER` was assigned because all identified test gaps expose implementation defects already classified at most HIGH in prior phases; any test gap that would detect a newly discovered BLOCKER-severity defect would be re-classified accordingly).

---

## ADR

No standalone ADR is produced for Phase 5. The alternatives considered (deliverable format, merge vs. keep-separate for TASK-T5/T6, framework-agnosticism for TASK-T12, severity assignment for live-gateway integration tests) are routine audit-shape decisions and are documented inline in §"Alternatives Considered" above. No architectural trade-off in the package itself is decided in this phase — the phase produces draft task entries only, not design choices.

The architectural decisions implied by the test gaps (e.g., whether fuzz testing uses `package:test` fuzz APIs or an external corpus-based tool; whether live-gateway tests use a mock server or a real public endpoint) are explicitly deferred to the implementing phases. If any of those decisions later requires an ADR, it will be authored in the implementing phase, not here.
