# Phase 2 Plan: BHTTP Layer Audit Against RFC 9292

**Ticket:** AW-2865 — ohttp_dart investigation
**Phase:** 2 of N (BHTTP layer)
**Status:** PLAN_APPROVED
**Inputs:** `phase-2/prd.md`, `phase-2/research.md`, `phase-2/tasks.md`, ticket-wide `idea.md`, `vision.md`; prior `phase-1/plan.md` and `phase-1/research.md` (referenced for deliverable shape and cross-layer findings)

---

## Phase Scope

This phase produces a **read-only audit deliverable** for `lib/src/bhttp.dart` (RFC 9292 Known-Length Binary HTTP framing, 191 lines). The deliverable is a set of severity-classified, independently actionable draft Jira task entries — already captured in `phase-2/research.md` under "Per-Task Findings (2.2–2.7)", "Limitations & Risks", and "Draft Task Entries (for Iteration 7)".

The plan below describes how the audit deliverable is structured, what artifacts are produced, where they live, and how they are reviewed for completeness before Phase 2 is closed. No code in `lib/` or `test/` is modified.

**In scope:**

- `lib/src/bhttp.dart` — the audited subject: `encodeVarint`, `decodeVarint`, `serializeRequest`, `parseResponse`, `BhttpResponse`, `BhttpHeader`.
- `test/bhttp_test.dart` — referenced as supporting evidence for the negative-case test gap (read-only).
- RFC 9292 (Known-Length framing only) and RFC 9000 §16 (QUIC variable-length integers) as normative references.
- Cross-references to `lib/src/ohttp_client.dart` (lines 97 and 136 — call sites of `serializeRequest` and `parseResponse`) and to Phase 1 findings F-1 through F-7.

**Explicitly out of scope for Phase 2:**

- RFC 9292 Indeterminate-Length framing (not implemented in `bhttp.dart`).
- `lib/src/hpke.dart` (Phase 1 — complete).
- `lib/src/ohttp.dart` (Phase 3).
- `lib/src/ohttp_client.dart` orchestration, timeouts, retry, KeyConfig caching (Phase 5).
- Privacy / documentation / `package:cryptography` transitive dependency review (Phase 6).
- Any source edits, test additions, or `pubspec.yaml` changes.

---

## Components

Because this is a read-only audit, "components" here means **deliverable artifacts** rather than runtime modules.

| Component | Path | Responsibility |
|---|---|---|
| Phase 2 PRD | `specs/.current/AW-2865/phase-2/prd.md` | Audit goals, user stories, scenarios, success criteria, resolved open questions. Source of truth for "done". |
| Phase 2 Research | `specs/.current/AW-2865/phase-2/research.md` | Per-task findings (2.2–2.7), limitations/risks table (R-1..R-7), cross-references to Phase 1, draft task entries (TASK-B1..TASK-B7), new technical questions. Canonical audit output. |
| Phase 2 Tasks | `specs/.current/AW-2865/phase-2/tasks.md` | Checklist driving the audit (tasks 2.1–2.8). Closed when all items are checked. |
| Phase 2 Plan (this file) | `specs/.current/AW-2865/phase-2/plan.md` | Defines deliverable structure, completeness gates, handoff to Phase 2 QA and Phase 3. |
| Phase 2 QA | `specs/.current/AW-2865/phase-2/qa.md` (later) | Verifies every PRD success criterion is met by the research output. |

**Audited subject (read-only, UNCHANGED):**

| File | Line ranges of interest | Role |
|---|---|---|
| `lib/src/bhttp.dart` | 15–40 (`encodeVarint`), 44–67 (`decodeVarint`), 74–111 (`serializeRequest`), 113–116 (`_writeField`), 136–190 (`parseResponse`) | Subject of every Phase 2 finding. |
| `test/bhttp_test.dart` | Whole file | Referenced for negative-case coverage gap (test-gap evidence). |
| `lib/src/ohttp_client.dart` | 97, 136 | Call sites of `bhttp.serializeRequest` / `bhttp.parseResponse`; referenced for call-stack propagation of `RangeError` / `FormatException`. |

---

## Target Interfaces and Contracts

The contracts below are inherited from Phase 1's deliverable shape and applied to the BHTTP layer.

### Findings contract (`research.md` "Per-Task Findings" + "Limitations & Risks")

Every per-task finding entry **must** carry:

1. **Task ID** — `2.2`–`2.7` mapped to the audit checklist in `tasks.md`.
2. **File : Line Range** — absolute reference into `lib/src/bhttp.dart` at the audit commit. Multiple references allowed when a finding spans helpers (e.g., `decodeVarint` consumed inside `parseResponse`).
3. **Code excerpt** — the smallest verbatim slice of `bhttp.dart` that captures the audited behavior.
4. **Verdict** — bold one-liner: `guard present and correct`, `guard absent: <description>`, or `correct by inspection`.
5. **RFC reference** — RFC 9292 section number, RFC 9000 §16 for varint correctness, or explicit "DoS scenario (no RFC requirement)".
6. **Severity** — exactly one of `BLOCKER`, `HIGH`, `IMPROVEMENT`. Carried into the "Limitations & Risks" row (`R-N`) for the same finding.

The "Limitations & Risks" table aggregates findings into a single severity-sorted view; every row maps 1:1 to either a positive-verdict finding (no row needed) or a guard-absent finding that becomes a draft task.

### Draft Jira task contract (`research.md` "Draft Task Entries")

Each guard-absent finding produces exactly one draft task (`TASK-B1`..`TASK-B7`) with:

- **Title** — single-line, imperative, scoped (e.g., "Add buffer-length guards to `decodeVarint` for 2-, 4-, and 8-byte widths").
- **Severity** — copied from the findings table; no "TBD".
- **File** — primary `bhttp.dart` line range targeted by remediation; secondary files (e.g., `bhttp_test.dart`, `ohttp_client.dart`) noted where the appropriate enforcement layer is ambiguous.
- **RFC section / DoS scenario** — exactly one normative reference or DoS narrative.
- **Description** — observation + risk + one or more concrete remediation directions, sufficient for an unrelated engineer to scope the work without re-running the audit.
- **Upstream check note** — per resolved-question Q5 (vision), each task carries "Check the upstream `ohttp_dart` GitHub repository before implementing a local patch".

### Section-coverage contract (`research.md`)

Every checklist item from `tasks.md` (2.2–2.7) **must** correspond to a per-task finding section. Task 2.1 (full file outline read) is a process step, not a finding; it is satisfied by the line citations in subsequent sections. Task 2.8 (draft task compilation) is satisfied by the "Draft Task Entries" section.

### Cross-phase reference contract

Findings that compound a Phase 1 risk **must** explicitly cite the Phase 1 finding ID (`F-1`..`F-7`) in a "Cross-reference Phase 1" line. Currently in `research.md`:

- R-3 (RangeError on truncated body) ↔ Phase 1 F-3 (StateError on HPKE overflow) — same untyped-error anti-pattern.
- R-5 (no body size cap in `serializeRequest`) ↔ Phase 1 F-2 (no size cap on `encRequest`).
- R-6 (no negative tests) ↔ Phase 1 F-6 / F-7 (test gaps).

---

## Audit Approach / Data Flow (audit pipeline)

```
[ tasks.md checklist (2.1..2.8) ]
                |
                v
[ 2.1: ast-index outline lib/src/bhttp.dart + targeted Read slices ]
                |
                v
[ 2.2..2.7: per-task parsing-surface verdicts ]
        |
        +--> 2.2 framing indicator validation     (bhttp.dart:140-146)
        +--> 2.3 status code range validation     (bhttp.dart:149-163)
        +--> 2.4 header loop size guards          (bhttp.dart:165-182)
        +--> 2.5 body sublist truncation type     (bhttp.dart:187)
        +--> 2.6 decodeVarint correctness + trunc (bhttp.dart:44-67)
        +--> 2.7 serializeRequest body size cap   (bhttp.dart:74-111)
                |
                v
[ research.md "Per-Task Findings (2.2-2.7)" populated ]
                |
                v
[ Limitations & Risks table (R-1..R-7) aggregates negative verdicts ]
                |
                v
[ Cross-phase sweep: cite Phase 1 F-IDs where applicable ]
                |
                v
[ 2.8: each guard-absent finding -> draft Jira task TASK-B1..TASK-B7 ]
                |
                v
[ New technical questions captured (web/WASM int, 1xx flood, utf8 errors,
  trailer-section silent drop) ]
                |
                v
[ Completeness check vs PRD success criteria + vision scrutiny table ]
                |
                v
[ Phase 2 QA + handoff to Phase 3 (ohttp.dart) ]
```

The pipeline is **strictly read-only at every stage**. The only writes are to `specs/.current/AW-2865/phase-2/*.md`.

### Per-parsing-surface audit method

For each parsing surface (tasks 2.2 through 2.7), the analyst:

1. Locates the code path in `bhttp.dart` using `ast-index symbol` or `ast-index outline`.
2. Reads only the line range cited by `tasks.md` Technical Details; avoids bulk reads.
3. Quotes the relevant code excerpt verbatim in `research.md`.
4. Constructs a malformed-input mental model (truncated buffer, oversized varint, adversarial header count, out-of-range status code, oversized body).
5. Determines the exact exception type that propagates to the caller; traces the call stack to `OhttpClient.send()` for at least the body-truncation case.
6. Renders a bold **Verdict** line. Negative verdicts are tagged with severity and added to the risks table.
7. Notes any test-coverage gap by reading `test/bhttp_test.dart` for the same code path.

---

## Non-Functional Requirements

| NFR | Requirement | How it is met |
|---|---|---|
| **Read-only guarantee** | `git diff HEAD -- lib/ test/ pubspec.yaml example/` must be empty at phase end. | No write tool calls target those paths. Phase-2 QA verifies the diff. |
| **Traceability** | Every finding traces to (a) a `bhttp.dart` line range, (b) an RFC section or DoS scenario, (c) a severity. | Findings contract above; QA verifies no row is missing a column. |
| **Coverage of all parsing surfaces** | Every checklist item 2.2–2.7 has a positive or negative verdict; the vision §4 scrutiny row for `bhttp.dart` ("No length guards", "No negative-case tests") is fully addressed. | Pipeline includes a completeness step against `tasks.md` and `vision.md §4`. |
| **Independent actionability** | Each draft task (`TASK-B1`..`TASK-B7`) can be picked up by a single engineer without depending on another in-flight Phase 2 task. | Per-site task decomposition (per resolved Q4); enforcement-layer ambiguity is noted inline, not deferred. |
| **Severity discipline** | Exactly one of `BLOCKER` / `HIGH` / `IMPROVEMENT` per task. No "TBD". | Resolved-question Q1 fixes the framing-indicator severity at HIGH (with reviewer-may-escalate note); all other severities are assigned in `research.md` from concrete DoS / API impact. |
| **Cross-layer accuracy** | Findings that compound Phase 1 risks cite the exact Phase 1 finding ID. | Cross-reference contract above; currently R-3, R-5, R-6. |
| **RFC citation accuracy** | Every cited RFC section must be the section that actually defines the behavior under audit. | Each per-task section quotes the implementation and names the RFC clause governing it (e.g., RFC 9292 §3.2, §3.4, §4.1; RFC 9000 §16). |
| **No leakage to other phases** | Findings about `ohttp.dart` / `ohttp_client.dart` are noted as cross-references only; they do not become Phase 2 tasks. | Cross-reference rows are explicit; the only `ohttp_client.dart` mentions in tasks are when the body-size cap may belong at that layer (per resolved Q2). |

---

## Risks

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-1 | Findings missed because `bhttp.dart` has grown or diverged since `vision.md` was authored. | Low | Medium | Task 2.1 mandates a full outline read at audit time; the PRD risks table calls this out (LOW). |
| R-2 | Severity drift between findings (one finding inflated/deflated). | Medium | Medium | Severity bar is fixed by vision §2 and resolved-question Q1 (framing indicator severity). Reviewer cross-references both before sign-off. |
| R-3 | Body-size guard ownership decision deferred indefinitely (BHTTP vs. `OhttpClient`). | Medium | Low | Resolved Q2 keeps the question open in the draft task entry; the follow-up engineer picks the layer at implementation time. No Phase 2 block. |
| R-4 | Wallet-team body-size threshold never supplied, making the cap task hard to remediate. | Medium | Medium | Resolved Q3 records the absence-of-cap as HIGH regardless of threshold; the threshold is a remediation input, not a finding gate. |
| R-5 | Cross-phase findings (e.g., `RangeError` propagation) accidentally close issues that Phase 5 (`ohttp_client.dart`) should own. | Medium | Low | Each cross-phase finding cites the Phase 1 ID and notes the call-stack path through `OhttpClient.send()`; the actual error-wrapping decision is deferred to Phase 5 if it spans layers. |
| R-6 | New technical questions (web/WASM int semantics, 1xx flood, utf8 errors, trailer-section drop) get lost. | Medium | Low | Captured in `research.md §"New Technical Questions"`; routed to Phase 5 or Phase 6 as appropriate during handoff. |
| R-7 | Upstream `ohttp_dart` GitHub repo may already have fixes for some findings, causing duplicate work later. | Medium | Low | Each draft task carries an explicit "Check upstream before patching" note (resolved Q5). |
| R-8 | RFC 9292 §3.4 trailer section is silently dropped in `parseResponse`; intentional or oversight is unclear. | Low | Low | Captured as New Technical Question #4; not promoted to a finding until intent is confirmed. |

---

## Alternatives Considered

**A1. Inline findings inside each task file (one Markdown per task).**
Rejected: same reasoning as Phase 1 (A1). A single research file with a findings section + per-site draft task entries preserves 1:1 mapping to future Jira tasks while keeping review centralized.

**A2. Consolidate the "RangeError vs. FormatException" findings into a single cross-cutting task.**
Rejected by resolved-question Q4: per-site tasks (TASK-B3 for body sublist, TASK-B4 for `decodeVarint`) preserve independent assignability. A consolidated task would couple two engineers' work for no architectural benefit.

**A3. Defer body-size-guard ownership (BHTTP vs. `OhttpClient`) to a separate ADR.**
Rejected by resolved-question Q2: the draft task records the ambiguity inline; the implementing engineer picks the layer based on the wallet team's threshold and call-site pattern. No Phase 2 architectural decision is needed.

**A4. Promote the framing-indicator finding to BLOCKER on cross-framing exploit suspicion.**
Rejected by resolved-question Q1: the guard is present and correct (verdict F-B1 is positive), so the question is moot at the code level. The reviewer-may-escalate note is preserved for Iteration 7 in case a higher-level exploit is identified.

**A5. Skip the per-task verdict block, jump straight to the risks table.**
Rejected: PRD success criterion "All RFC 9292 parsing surfaces covered" requires a positive or negative verdict per surface (2.2–2.7). The verdict block is the evidence that the analyst actually read each surface, not just the lines that flagged a risk.

No alternatives reach the bar of requiring a separate ADR document — see ADR section below.

---

## Dependencies

**Prior phases:**

- **Phase 1 (HPKE layer audit) — complete.** Phase 2 references Phase 1 findings F-2, F-3, F-6, F-7 by ID for cross-layer concerns. Phase 1's deliverable structure (findings table, draft task entries, RFC verdicts) is inherited.

**Concurrent/inherited inputs:**

- `specs/.current/AW-2865/idea.md` — ticket-wide motivation and acceptance criteria.
- `specs/.current/AW-2865/vision.md` — severity model (§2), architecture (§4), data model §5.6 (`BhttpResponse` concerns), §5 cross-cutting concerns ("RangeError instead of FormatException on truncated BHTTP body" = HIGH; "No size caps on response body / headers" = HIGH).
- `specs/.current/AW-2865/phase-2/prd.md` — phase goals, scenarios, success criteria, resolved open questions Q1–Q5.
- `specs/.current/AW-2865/phase-2/research.md` — audit output (canonical findings + draft tasks).
- `specs/.current/AW-2865/phase-2/tasks.md` — checklist driving the audit (2.1–2.8).
- `specs/.current/AW-2865/phase-1/research.md` — source of Phase 1 finding IDs cited by R-3, R-5, R-6.
- `lib/src/bhttp.dart` — read-only subject.
- `test/bhttp_test.dart` — read-only supporting evidence for test-gap claims (R-6, R-7).
- `lib/src/ohttp_client.dart` lines 97, 136 — read-only call-site reference for error-propagation path.
- RFC 9292 (Known-Length framing); RFC 9000 §16 (QUIC varints).

**Downstream consumers (later phases):**

- **Phase 3** (`ohttp.dart` audit) consumes Phase 2's `RangeError` propagation finding (R-3) because the same anti-pattern recurs in `OhttpKeyConfig.parse` (vision §5.1 notes "malformed blob with large `symLen` and short buffer throws `RangeError`, not `FormatException`"). Phase 3 must cite Phase 2's TASK-B3 / TASK-B4 to avoid duplicating remediation language.
- **Phase 5** (`ohttp_client.dart` audit) consumes Phase 2's body-size-guard ambiguity (TASK-B5 resolved Q2): the canonical enforcement layer is decided there if not at the BHTTP layer.
- **Phase 6** (privacy and documentation) consumes Phase 2's new technical questions about web/WASM target and `dart2js` integer semantics for the 8-byte varint case.
- **Iteration 7** (compile draft entries into Jira tickets) consumes the "Draft Task Entries" section of `research.md` directly.

---

## Open Questions (Phase 2)

The PRD's five open questions were resolved before research began (see PRD §"Resolved Open Questions" Q1–Q5 and research §"Resolved Questions" Q1–Q6). New questions surfaced during the audit are recorded in `research.md §"New Technical Questions"` and deferred:

1. **`decodeVarint` integer overflow on 8-byte varints under dart2js / WASM.** Dart native VM has 63-bit signed `int`; dart2js has 53-bit double `int`. Values above `2^53` truncate silently. Routed to Phase 6 (privacy/documentation) or treated as a documentation-only constraint if the wallet target excludes web.
2. **`parseResponse` informational-response loop termination (1xx flood).** `while (true)` loop with no max-iteration cap; adversarial gateway could send unbounded 1xx responses. Candidate Phase 2 IMPROVEMENT addendum or Phase 5 finding pending wallet-team input on expected 1xx frequency.
3. **`utf8.decode` `FormatException` indistinguishable from framing `FormatException` at the `OhttpClient.send()` API boundary.** Error-message-parsing is fragile; routed to Phase 5 (client-layer typed-exception hierarchy design).
4. **Trailer section silently discarded by `parseResponse` (RFC 9292 §3.4).** Unclear whether intentional; the offset is not verified against `data.length` after the body sublist. Candidate Phase 2 IMPROVEMENT addendum or RFC-conformance follow-up — pending decision on whether to add finding R-8 / TASK-B8.

None of these block closing Phase 2 — they are recorded for downstream phases.

---

## Definition of Done (Phase 2)

Phase 2 closes when **all** of the following hold:

1. `tasks.md` items 2.1–2.8 are checked.
2. `research.md` contains:
   - A per-task findings section for each of 2.2–2.7 with a verdict line and a `bhttp.dart` line citation.
   - A "Limitations & Risks" table aggregating negative verdicts with severity per row.
   - One draft task entry per guard-absent finding (`TASK-B1`..`TASK-B7`), conforming to the draft-task contract above.
   - Cross-references to Phase 1 finding IDs for any compounding risk.
   - A "New Technical Questions" section capturing the four deferred questions above.
3. `git diff HEAD -- lib/ test/ pubspec.yaml example/` is empty.
4. Phase 2 QA report (`phase-2/qa.md`) confirms each PRD success criterion is met.
5. No risks-table row has an unset severity.
6. The vision §4 scrutiny-table row for `bhttp.dart` ("No length guards", "No negative-case tests") is addressed by at least one finding each.

---

## ADR

No standalone ADR is produced for Phase 2. The alternatives considered (deliverable format, severity timing, per-site vs. consolidated tasks, body-size-guard ownership) are routine audit-shape and remediation-scoping decisions, documented inline in the "Alternatives Considered" section above. No architectural trade-off in the package itself is decided in this phase — the phase produces findings only, not design choices.

If a Phase 2 finding later motivates an architectural change (e.g., introducing a `BhttpException` typed hierarchy that wraps `RangeError` from `Uint8List.sublist`, or moving the body-size cap to `OhttpClient` with a config field), that change will be planned and ADR'd in the implementing phase, not here.
