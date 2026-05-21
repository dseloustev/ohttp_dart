# Phase 4 Plan: OhttpClient Layer Audit — Network Reliability, KeyConfig Lifecycle, Scheme Enforcement, and sendDirect() Bypass Risks

**Ticket:** AW-2865 — ohttp_dart investigation
**Phase:** 4 of 7 (`ohttp_client.dart` audit)
**Status:** PLAN_APPROVED
**Inputs:** `specs/.current/AW-2865/phase-4/prd.md`, `specs/.current/AW-2865/phase-4/research.md`, `specs/.current/AW-2865/phase-4/tasks.md`, ticket-wide `idea.md`, `vision.md`, prior phase research (`phase-1/research.md`, `phase-2/research.md`, `phase-3/research.md`)

---

## Phase Scope

This phase produces a **read-only audit deliverable** for `lib/src/ohttp_client.dart` (Layer 4 of the four-layer stack — the only layer a wallet integrator directly calls). The deliverable is a set of severity-classified, independently actionable draft Jira task entries — already captured in `phase-4/research.md` (Findings C-1..C-12 and the Draft Task Entries table).

No code in `lib/` or `test/` is modified in this phase. The plan below describes how the audit deliverable is structured, what artifacts are produced, where they live, and how they are reviewed for completeness before Phase 4 is closed.

**Explicitly in scope for Phase 4:**

- `lib/src/ohttp_client.dart` lines 1–177 — full file.
- `OhttpGatewayConfig` constructor and field validation surface.
- `OhttpClient.send()` orchestration: KeyConfig GET → BHTTP-serialize → encap → POST → decap → BHTTP-parse.
- `OhttpClient.sendDirect()` plaintext bypass path.
- `OhttpResponse` / `OhttpHeader` construction on both response paths.
- Exception propagation contract from `send()` (cross-references `package:http` and `package:cryptography` exception types reaching the wallet boundary).
- Cross-layer compounding with Phase 2 (BHTTP size caps) and Phase 3 (`SecretBoxAuthenticationError` propagation).

**Explicitly out of scope for Phase 4:**

- `lib/src/hpke.dart` (Phase 1), `lib/src/bhttp.dart` (Phase 2), `lib/src/ohttp.dart` (Phase 3) — only referenced where a client-layer finding compounds a lower-layer gap.
- `package:http` and `package:cryptography` internals — treated as primitives whose contracts are assumed correct; only their public exception surface is audited.
- Integration testing against a live OHTTP gateway (no such tests exist, none are added).
- Test files under `test/` — `ohttp_client.dart` has no dedicated test file at the audit commit.
- Any source edits, test additions, or `pubspec.yaml` changes.
- Designing the typed exception hierarchy itself — TASK-C7/C8 records the gap; the remediation design is deferred to the implementing phase.

---

## Components

Because this is a read-only audit, "components" means **deliverable artifacts** rather than runtime modules.

| Component | Path | Responsibility |
|---|---|---|
| Phase 4 PRD | `specs/.current/AW-2865/phase-4/prd.md` | Audit goals, ten investigation scenarios, success criteria, risks. Source of truth for what counts as "done". |
| Phase 4 Research | `specs/.current/AW-2865/phase-4/research.md` | Resolved questions, related modules table, current endpoints/contracts, twelve findings (C-1..C-12), cross-layer references, draft task entries. Canonical audit output. |
| Phase 4 Tasks | `specs/.current/AW-2865/phase-4/tasks.md` | Checklist driving the audit (tasks 4.1–4.12). Closed when all items are checked. |
| Phase 4 Plan (this file) | `specs/.current/AW-2865/phase-4/plan.md` | Defines deliverable structure, completeness gates, cross-layer references, and handoff to phase QA / Iteration 7 (task compilation). |
| Phase 4 QA | `specs/.current/AW-2865/phase-4/qa.md` (later) | Verification that every PRD success criterion is met by the research output. |
| Phase 4 Summary | `specs/.current/AW-2865/phase-4/summary.md` (later) | Concise close-out matching the structure used by Phases 1–3. |

**Audited subject (read-only, UNCHANGED):**

| File | Lines | Role |
|---|---|---|
| `lib/src/ohttp_client.dart` | 1–177 | `OhttpClient`, `OhttpGatewayConfig`, `OhttpResponse`, `OhttpHeader`. The subject of every Phase 4 finding. |
| `lib/src/ohttp.dart` | 219 (referenced) | Throw site of `SecretBoxAuthenticationError` — confirmed to propagate through the client layer (C-11 cross-references Phase 3 O-9). |
| `lib/src/bhttp.dart` | 165–182 (referenced) | Parser without header/body caps — compounds C-9. |
| `package:http` 1.6.0 | (referenced) | `ClientException`, `_ClientSocketException`, `RequestAbortedException` — reach the wallet caller as documented in C-8. |
| `package:cryptography` | `mac.dart:52` (referenced) | `SecretBoxAuthenticationError implements Exception` — confirmed unwrapped propagation. |

---

## Target Interfaces and Contracts

### Findings contract (research.md "Limitations & Risks")

Every finding row carries exactly these fields:

1. **ID** — stable identifier (`C-1`, `C-2`, …) for cross-reference from later phases and the Iteration 7 task-compilation step.
2. **File : lines** — absolute reference into `lib/src/ohttp_client.dart` at the audit commit. Multiple references allowed where a finding spans constructor + call sites (e.g., C-1 cites lines 35–53, 81–82, 115–116).
3. **Concern category** — one of: Scheme enforcement, URL construction, Privacy/SSRF, Privacy/Documentation, Network reliability, KeyConfig lifecycle, Error handling, API contract, Parser robustness/DoS, API consistency, Privacy/Observability.
4. **Severity** — exactly one of `BLOCKER`, `HIGH`, `IMPROVEMENT`. No "TBD" or compound severities. Severities follow the vision §2 model with HIGH used as the default for wallet-blocking-but-not-immediate concerns.
5. **Description** — concrete behavioral statement plus the wallet-threat-model consequence.
6. **Draft task title** — single-line, imperative title for the Iteration 7 task-compilation step.

### Draft Jira task contract (research.md "Draft Task Entries")

Each finding produces exactly one draft task (`TASK-C1`..`TASK-C12`) with:

- **Task ID** — `TASK-C{N}` where `{N}` matches the finding ID for traceability.
- **Title** — single-line, imperative, scoped (e.g., "Enforce `https`-scheme on `OhttpGatewayConfig.gatewayBaseUrl`").
- **Severity** — copied from the findings entry.
- **Primary file : lines** — the remediation site.
- **Concern category** — copied from the findings entry.

Two contract exceptions are explicitly documented in the research:

- **TASK-C11** has no independent remediation work in `ohttp_client.dart`. It is a cross-layer confirmation of Phase 3 TASK-O5 (`ohttp.dart:219`). Closing TASK-O5 closes TASK-C11.
- **TASK-C7 and TASK-C8** overlap: a typed `OhttpClientException` hierarchy is the wrapping vehicle for foreign exceptions. They can be merged into a single Jira ticket at Iteration 7 — both rows are kept in the draft list so the merger is an explicit decision rather than a silent omission.

### Cross-layer reference contract

A finding qualifies as cross-layer when a defect at this layer compounds a defect at a lower layer. The research §"Cross-Layer References" table is the canonical source. Two compounding patterns are recorded:

1. **Phase 2 → Phase 4 amplification** — Phase 2 R-2 (no header/body cap in `parseResponse`) plus Phase 4 C-9 (no body cap before `ohttpDecapsulate`) = two-layer unbounded allocation. Recorded as a single HIGH at C-9 with an explicit Phase-2 reference.
2. **Phase 3 → Phase 4 confirmation** — Phase 3 O-9 (`SecretBoxAuthenticationError` unwrapped) confirmed to reach the wallet boundary as C-11; remediation site remains `ohttp.dart:219` (TASK-O5).

The contract: cross-layer findings cite the lower-layer finding ID by name and do not duplicate the remediation task. This keeps the Iteration 7 task list de-duplicated.

### Scrutiny-table coverage contract

Every row of `vision.md` §4 scrutiny table for `ohttp_client.dart` must have either a finding or an explicit "no issue found" note in the research. The vision scrutiny rows are:

- "No timeout/retry/cancellation on `http.Client.get()` or `http.Client.post()`" → C-5, C-6.
- "KeyConfig fetched fresh on every `send()` invocation (no caching, no TTL)" → C-5.
- "No downgrade detection when gateway rotates key or cipher suite" → recorded as "New Technical Question 3" in research (cache-invalidation strategy); deferred to TASK-C5 remediation design.
- "`sendDirect()` escape path co-located in the same config object as OHTTP paths" → C-4.

Vision §5.4 (`OhttpGatewayConfig`) and §5.5 (`OhttpResponse` / `OhttpHeader`) sub-rows map to C-1 (https), C-2 (Uri normalization), C-3 (targetAuthority), C-4 (directBaseUrl), C-9 (size caps), C-10 (lowercasing).

---

## Data Flow (audit pipeline)

```
[ tasks.md checklist (4.1–4.12) ]
                |
                v
[ Read ohttp_client.dart in full ]
   (file is 177 lines; below the ast-index outline threshold of 500
    but still routed through ast-index for symbol lookups per project rules)
                |
                v
[ Map each vision scrutiny row to a tasks.md item (4.2..4.11) ]
                |
                v
[ For each item, confirm presence/absence via direct line reference;
  cross-check exception types reaching send() boundary via package:http
  + package:cryptography source ]
                |
                v
[ research.md "Current Endpoints & Contracts" + "Limitations & Risks"
  populated with C-1..C-12 ]
                |
                v
[ Cross-layer pass: cite Phase 2 R-2/R-3/R-5 and Phase 3 O-4/O-9 IDs
  in the appropriate findings ]
                |
                v
[ Each finding -> draft task entry (TASK-C1..TASK-C12) ]
                |
                v
[ Coverage check vs PRD success criteria + vision §4 scrutiny rows ]
                |
                v
[ Phase 4 QA + handoff to Iteration 7 (task compilation) ]
```

The pipeline is **strictly read-only at every stage**. The only writes are to `specs/.current/AW-2865/phase-4/*.md`.

---

## Non-Functional Requirements

| NFR | Requirement | How it is met |
|---|---|---|
| **Read-only guarantee** | `git diff HEAD -- lib/ test/ pubspec.yaml example/` must be empty at phase end. | No write tool calls target those paths. Phase-4 QA verifies the diff. |
| **Traceability** | Every finding traces to (a) a file:line range in `ohttp_client.dart`, (b) a concern category, (c) a severity, (d) a draft task title. | Findings contract above; QA verifies no row is missing a field. |
| **Coverage** | Every vision §4 scrutiny row for `ohttp_client.dart` has a finding or an explicit "no issue found" / "deferred to remediation" note. | Scrutiny-table sweep is a named step in the pipeline; QA cross-checks against the §"Scrutiny-table coverage contract" mapping above. |
| **Independent actionability** | Each draft task can be picked up by a single engineer without depending on another in-flight Phase 4 task. | Verified during draft-task review. TASK-C7 + TASK-C8 are flagged as candidates for merge at Iteration 7; TASK-C11 is flagged as resolved-by-TASK-O5 (no independent work). |
| **Severity discipline** | Exactly one of `BLOCKER` / `HIGH` / `IMPROVEMENT` per task. No "TBD". | Resolved-questions notes in the PRD fix ambiguous cases (sendDirect = HIGH not BLOCKER; OhttpHeader casing = IMPROVEMENT). |
| **No leakage to other phases** | Findings about `hpke.dart`, `bhttp.dart`, `ohttp.dart` are deferred; cross-layer items cite the lower-layer finding ID and do not produce duplicate tasks. | Verified during draft-task review. C-9 cites Phase 2 R-2; C-8 cites Phase 2 R-3; C-11 cites Phase 3 O-9 and points to TASK-O5. |
| **Exception-type accuracy** | Every claimed exception type reaching `send()` callers is grounded in the actual source of `package:http` 1.6.0 and `package:cryptography` (cited in research Q2). | Research §"Resolved Questions" Q2 quotes file paths and line numbers from both libraries; QA spot-checks. |
| **Privacy-rule conformance** | Findings related to logging (C-12) must align with vision §7 forbidden-data list. | C-12 explicitly cites vision §7 constraint 6. |

---

## Risks

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-1 | Findings missed because `ohttp_client.dart` has changed since the vision was authored. | Low | Medium | Task 4.1 mandates a full file read at audit time; research §"Current Endpoints & Contracts" enumerates every call site by line number. |
| R-2 | Severity drift — `sendDirect()` inflated to BLOCKER because "no warning" feels critical. | Medium | Medium | PRD §"Open Questions" fixes `sendDirect()` at HIGH because activation requires deliberate misconfiguration (not the default `send()` path). |
| R-3 | Cross-layer findings accidentally produce duplicate tasks. | Medium | Low | Cross-layer reference contract above mandates citing the lower-layer finding ID; TASK-C11 explicitly documented as "resolved by TASK-O5". |
| R-4 | Draft task descriptions too vague to act on, requiring re-investigation in Iteration 7. | Low | Medium | Each finding entry in the research already includes the threat-model consequence and a proposed remediation direction. |
| R-5 | Exception-type claims drift if `package:http` is upgraded between audit and remediation. | Low | Medium | Research Q2 records the `package:http` version (1.6.0) at audit time; remediation tasks must re-verify against the pinned version in `pubspec.yaml`. |
| R-6 | The `HandshakeException` wrapping decision (New Technical Question 1) blocks TASK-C8 scoping. | Medium | Low | Documented as an open question in this plan; resolution is part of TASK-C8 design, not Phase 4 closure. |
| R-7 | KeyConfig cache-invalidation strategy (New Technical Question 3) is treated as a performance concern when it is also a correctness concern. | Low | Medium | Recorded in the plan §"Open Questions" so TASK-C5 remediation design includes a forced-refresh trigger on 4xx / decap `FormatException`. |
| R-8 | `effectiveDirectBaseUrl` footgun (New Technical Question 2) is overlooked because it is buried inside C-4. | Low | Low | Plan §"Open Questions" surfaces it as a discrete design question for TASK-C4. |
| R-9 | The `OhttpClient.dispose()` lifecycle gap (research New Technical Question 4) is silently dropped. | Low | Low | Plan §"Open Questions" records it as a candidate addendum to TASK-C12 or a new IMPROVEMENT task; QA decides at close-out. |

---

## Alternatives Considered

**A1. One Markdown file per finding (12 files).**
Rejected: matches the Phase 1/2/3 deliverable shape — one research document with a findings table and a draft-task table — keeping the audit reviewable in a single pass. Iteration 7 will fan out to per-task files when Jira tickets are produced.

**A2. Defer severity assignment to Iteration 7 triage.**
Rejected: PRD success criterion explicitly requires severity on every finding. The Phase 4 PRD §"Open Questions" already resolved the ambiguous cases (sendDirect, https placement, timeout default).

**A3. Merge TASK-C7 and TASK-C8 unconditionally.**
Rejected at this stage: both findings have distinct concern categories (Error handling vs. API contract) and distinct line citations. Merging is recorded as a candidate for Iteration 7 so the decision is explicit and reviewable, not silent.

**A4. Skip cross-layer findings (C-9, C-11) because they "belong to" Phases 2 and 3.**
Rejected: the wallet-visible consequence only manifests at the client boundary. Cross-layer findings are recorded here with explicit references to the owning phase's task; this is the only way to capture two-layer amplification (C-9) and to confirm propagation reaches the wallet (C-11).

**A5. Audit `package:http` internals for the exception-type list.**
Rejected as a deep audit, accepted as a quoted-source step (research Q2). The audit cites file paths and line numbers from `package:http` 1.6.0 source without claiming responsibility for the library's correctness. This is consistent with the vision §2 principle "ground every finding in concrete evidence".

No alternative reaches the bar of requiring a standalone ADR — see ADR section.

---

## Dependencies

**Prior phases (must be complete before Phase 4 can close):**

- **Phase 1 (`hpke.dart` audit) — COMPLETE.** Relevant cross-layer findings: F-2 (no zeroization of `key`, `baseNonce`, `exporterSecret`) is referenced obliquely by the OHTTP-client privacy posture but does not produce a new Phase 4 finding; F-5 (`testKeyPair` reachable from production) is confirmed not triggered at `ohttp_client.dart:108` (research §"Patterns Used" item 4).
- **Phase 2 (`bhttp.dart` audit) — COMPLETE.** Cross-referenced findings: R-2 (no header/body cap in `parseResponse`) compounds C-9; R-3 (`RangeError` on truncated body) propagates through C-8; R-5 (no body cap in `serializeRequest`) partially compounds C-9.
- **Phase 3 (`ohttp.dart` audit) — COMPLETE.** Cross-referenced findings: O-4 (inconsistent exception types) contributes to C-8; O-9 (`SecretBoxAuthenticationError` unwrapped) is confirmed at the wallet boundary as C-11, with TASK-O5 remaining the canonical fix site.

**Concurrent/inherited inputs:**

- `specs/.current/AW-2865/idea.md` — ticket-wide motivation.
- `specs/.current/AW-2865/vision.md` — severity model (§2), architecture (§4), data model (§5.4, §5.5), workflows (§6.1, §6.4), logging constraints (§7).
- `specs/.current/AW-2865/phase-4/prd.md` — phase goals, ten scenarios, risks.
- `specs/.current/AW-2865/phase-4/research.md` — audit output (findings C-1..C-12, draft tasks TASK-C1..TASK-C12).
- `specs/.current/AW-2865/phase-4/tasks.md` — checklist (4.1–4.12).
- `lib/src/ohttp_client.dart` — read-only subject (177 lines).
- `lib/src/ohttp.dart`, `lib/src/bhttp.dart` — read-only references for cross-boundary findings.
- `package:http` 1.6.0, `package:cryptography` — read-only references for exception-type claims.
- RFC 9458 §1 (privacy goals), RFC 7230 §3.2 (header case-insensitivity).

**Downstream consumers:**

- **Phase 5/6/7** (if scoped — tests, documentation, threat model) consume Phase 4 findings for coverage planning. Each Phase 4 finding implies one or more missing negative-test cases (network timeout simulation, scheme misconfiguration, oversized response).
- **Iteration 7 (task compilation)** consumes the §"Draft Task Entries" table directly. The Iteration 7 step is the only step that produces per-task Markdown files for Jira import.
- **AW-2857 (blocked-by relationship)** — the consuming Jira ticket. Phase 4 deliverables feed into the AW-2857 production-readiness checklist for the wallet integration.

---

## Open Questions (Phase 4)

The PRD's Open Questions were resolved before research began (PRD §"Open Questions"). New questions surfaced during research (research §"New Technical Questions") are deferred and recorded below for downstream visibility — none block Phase 4 closure:

1. **`HandshakeException` wrapping scope (research Q1).** Does TASK-C8 wrap `dart:io.HandshakeException` inside the new `OhttpClientException`, or does the wallet's `http.Client` wrapper handle it? Determines whether the fix lives in `ohttp_client.dart` or in a custom `http.Client` subclass. Deferred to TASK-C8 design.
2. **`effectiveDirectBaseUrl` null fallback (research Q2).** When `directBaseUrl` is null, `sendDirect()` sends to `gatewayBaseUrl` (the OHTTP relay), not the target. Should `sendDirect()` throw `StateError` on null, or is this intentional in some deployments? Deferred to TASK-C4 design.
3. **KeyConfig cache invalidation (research Q3).** Any TTL cache (TASK-C5) must define a forced-refresh trigger on key rotation (4xx response, decap `FormatException`). Invalidation strategy is a correctness concern, not just performance. Deferred to TASK-C5 design.
4. **`OhttpClient.dispose()` lifecycle (research Q4).** Abandoning `OhttpClient` without `dispose()` leaks the `IOClient` connection pool. May need a class-doc note; candidate addendum to TASK-C12 or a new IMPROVEMENT task. Decision deferred to Phase 4 QA.
5. **`onLog` argument evaluation (research Q5).** `onLog?.call(expensiveString)` still evaluates the string argument when `onLog` is null. Negligible at current call sites; recorded as an implementation note for TASK-C12.

None of these block closing Phase 4 — they are recorded for downstream phases and remediation design.

---

## Definition of Done (Phase 4)

Phase 4 closes when **all** of the following hold:

1. `tasks.md` items 4.1–4.12 are checked.
2. `research.md` contains:
   - "Resolved Questions" Q1, Q2, Q3 with source citations from `package:http` and `package:cryptography`.
   - "Related Modules / Services" tables listing every direct call site in `ohttp_client.dart` with line numbers.
   - "Current Endpoints & Contracts" enumerating `OhttpGatewayConfig`, `OhttpClient.send()` step-by-step, `sendDirect()`, and `dispose()`.
   - Twelve findings (C-1..C-12) — each with file:lines, concern category, severity, description, and a draft task title.
   - "Cross-Layer References" tables for Phase 2 and Phase 3 cross-references.
   - "Draft Task Entries" table with TASK-C1..TASK-C12 mapped 1:1 to the findings (with the documented exceptions: TASK-C11 resolved-by-TASK-O5, TASK-C7+TASK-C8 mergeable at Iteration 7).
   - "New Technical Questions" recording deferred items 1–5.
3. `git diff HEAD -- lib/ test/ pubspec.yaml example/` is empty.
4. Phase 4 QA report (`phase-4/qa.md`) confirms each PRD success criterion is met.
5. No finding row has an unset severity, an unset file:line, or an unset concern category.
6. Cross-layer findings explicitly annotate the phase that owns the remediation (C-9 → Phase 2 R-2; C-11 → Phase 3 O-9 / TASK-O5).
7. Every vision §4 scrutiny row for `ohttp_client.dart` is accounted for (finding or explicit deferral note).

---

## ADR

No standalone ADR is produced for Phase 4. The alternatives considered (deliverable format, severity timing, finding aggregation, cross-layer reference policy, exception-source audit depth) are routine audit-shape decisions and are documented inline in §"Alternatives Considered" above. No architectural trade-off in the package itself is decided in this phase — the phase produces findings only, not design choices.

The architectural decisions implied by the findings (introducing `OhttpClientException`, adding a KeyConfig TTL cache, enforcing `https` at construction time, adding `maxResponseBytes`, wrapping foreign exceptions) are explicitly deferred to the implementing phases / Iteration 7 task compilation. If any of those decisions later requires an ADR (for example, choosing between exception-wrapping at the `ohttp.dart` layer versus the `ohttp_client.dart` layer for cryptographic failures), the ADR will be authored in the implementing phase, not here.
