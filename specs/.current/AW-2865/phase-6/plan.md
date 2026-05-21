# AW-2865 Phase 6 — Implementation Plan

**Ticket:** AW-2865
**Phase:** 6 — Privacy Risk Review, Observability Gaps Audit, and Documentation Review
**Status:** PLAN_APPROVED
**Date:** 2026-05-21
**Inputs:** `specs/.current/AW-2865/phase-6/prd.md`, `specs/.current/AW-2865/phase-6/research.md`, `specs/.current/AW-2865/phase-6/tasks.md`, `specs/.current/AW-2865/vision.md`
**ADR:** None required — no architectural trade-off is being made in this phase.

---

## 1. Phase Scope

Phase 6 is the final investigation-phase audit of the AW-2865 ticket. It is a **read-only static-source audit** producing draft task entries only — no code, test, or example file is modified. The phase covers three cross-cutting reviews not bound to a single implementation layer:

1. **Privacy risk review** — assess whether `ohttp_dart` preserves OHTTP's unlinkability guarantee for non-custodial wallet use, sweeping `lib/src/ohttp_client.dart`, `lib/src/ohttp.dart`, `lib/src/hpke.dart`.
2. **Observability gaps catalog** — confirm and structure the five observability gaps and six logging constraints introduced in `vision.md §7`.
3. **Documentation review** — audit `example/ohttp_dart_example.dart` and `pubspec.yaml` for documented limitations, version pinning integrity, and absence of native/FFI dependencies.

Tasks 6.1–6.11 in `phase-6/tasks.md` drive the work. The phase output feeds Phase 7 (cross-phase severity triage and consolidation), which compiles findings from Phases 1–6 into per-task Markdown files.

**Explicitly out of scope:**
- Any modification to files in `lib/`, `test/`, or `example/`.
- Adding logging, assertions, or documentation to source files.
- Re-running prior-phase findings; cross-references cite by ID only.
- Transitive dependency audit (`pubspec.yaml` review covers direct dependencies only).
- Live-gateway verification — all checks are static source review.

---

## 2. Components

This phase has no runtime components. The "components" are the audit streams and the source artifacts under review.

### 2.1 Audit streams

| Stream | Driver tasks | Source artifacts |
|---|---|---|
| Privacy risk review | 6.1, 6.2, 6.3, 6.4 | `lib/src/ohttp_client.dart`, `lib/src/ohttp.dart`, `lib/src/hpke.dart` |
| Observability gaps catalog | 6.10, 6.11 | All four files in `lib/src/` plus `vision.md §7` |
| Documentation review | 6.5, 6.6, 6.7, 6.8, 6.9 | `example/ohttp_dart_example.dart`, `pubspec.yaml` |

### 2.2 Output artifact

A single artifact: an updated `specs/.current/AW-2865/phase-6/tasks.md` containing 13 draft task entries (P6-1 through P6-13) plus the five-gap observability catalog and six-constraint logging matrix. Each task is recorded under the corresponding numbered checklist item (6.1–6.11) and as a consolidated draft entry block at the end of the file.

### 2.3 Severity matrix (input to Phase 7 triage)

| Task ID | Severity | Concern category |
|---|---|---|
| P6-1 | BLOCKER | Privacy — OHTTP bypass via silent `effectiveDirectBaseUrl` fallback |
| P6-2 | HIGH | Privacy — no `https` enforcement on outer channel |
| P6-3 | HIGH | Privacy — key material persistence (`hpke.dart` scope) |
| P6-4 | HIGH | Privacy — key material persistence (`ohttp.dart` scope) |
| P6-5 | HIGH | Observability — KeyConfig fetch (OG-1) |
| P6-6 | HIGH | Observability — gateway POST status (OG-2) |
| P6-7 | HIGH | Observability — AEAD authentication failure (OG-3) |
| P6-8 | HIGH | Observability — OHTTP bypass signal (OG-4) |
| P6-9 | IMPROVEMENT | Privacy — metadata in opt-in `onLog` |
| P6-10 | IMPROVEMENT | Observability — encap timing (OG-5) |
| P6-11 | IMPROVEMENT | Documentation — example file limitations |
| P6-12 | IMPROVEMENT | Dependency — version pinning (confirmed-clean) |
| P6-13 | IMPROVEMENT | Privacy — `effectiveDirectBaseUrl` API visibility |

P6-1 is escalated from HIGH to BLOCKER because Phase 6 research source-confirmed the silent fallback condition spelled out in PRD Constraint 10 and Resolved Question Q1.

---

## 3. API Contract — Draft Task Entry Schema

Every finding entered into `phase-6/tasks.md` follows this schema. Phase 7 ingests these entries verbatim.

```
Task ID:           P6-N                       (stable, monotonic within phase)
Severity:          BLOCKER | HIGH | IMPROVEMENT
File:              <repo-relative path>
Lines:             <line range, e.g. 148-171, or comma-separated singletons>
Concern category:  <free text, examples in §2.3 above>
Description:       <plain-text finding; must cite specific source evidence>
Cross-refs:        <prior-phase finding IDs only, e.g. "Phase 4 C-4, Phase 1 F-2">
```

**Mandatory fields:** Task ID, Severity, File, Lines, Concern category, Description. Cross-refs are mandatory when a prior phase identified the same or compounding issue; otherwise omit the field.

**Path discipline:** Per `.claude/agents/docs/path-conventions.md`, all `File` values are repo-relative. No `/Users/...`, `/home/...`, `C:\...` paths in committed artifacts.

**Severity discipline:** Per `vision.md §2`, every finding carries exactly one of `BLOCKER`, `HIGH`, `IMPROVEMENT`. No hybrid labels ("HIGH-leaning", "BLOCKER candidate") in the final entry — Phase 6 has resolved all severities. The "BLOCKER candidate" framing used in PRD Scenario A is closed by P6-1 being entered directly as BLOCKER.

---

## 4. Data Flows

### 4.1 Audit-stream flow per task

```
Task (6.x)
  -> ast-index outline <file>          (skip if file < 500 lines and structure is known)
  -> Read <file> at targeted line range
  -> Grep for forbidden patterns (print/debugPrint/dart:developer/log\(/Logger)
      across lib/src/ (one sweep; results cached across tasks 6.2-6.4)
  -> Record finding(s) as draft task entries P6-N per the schema in §3
  -> Tick the task checkbox in phase-6/tasks.md
```

### 4.2 Cross-phase findings consolidation flow

```
Prior-phase findings (Phase 1 F-2, Phase 3 TASK-O6, Phase 4 C-1, Phase 4 C-4)
  -> Phase 6 source re-confirmation at original line ranges
  -> Promote to P6-N draft entries with prior-phase IDs in Cross-refs field
  -> No verbatim duplication of prior-phase descriptions
```

### 4.3 Phase 7 feed

```
13 draft entries (P6-1 ... P6-13)
  + 5-gap observability catalog (OG-1 ... OG-5)
  + 6-constraint logging matrix (LC-1 ... LC-6)
  -> phase-6/tasks.md (consolidated draft-entries block)
  -> Phase 7 reads phase-6/tasks.md and merges with Phases 1-5
     into per-task Markdown files under specs/.current/AW-2865/tasks/
```

---

## 5. Non-Functional Requirements

1. **Zero source mutation.** No file in `lib/`, `test/`, or `example/` is modified. The Phase 6 outputs are confined to `specs/.current/AW-2865/phase-6/`.
2. **Concrete citations.** Every finding cites a specific file and line range. Vague statements are rejected per `vision.md §2`.
3. **Single grep sweep.** Tasks 6.2–6.4 share one ripgrep pass over `lib/src/` for `print|debugPrint|dart:developer|log\(|Logger`. Results are cached and re-used across the three tasks; no per-task redundant sweeps.
4. **ast-index first.** Per `CLAUDE.md` and `.claude/rules/ast-index.md`, symbol search uses `ast-index` before `Grep`. The grep sweep above is the explicit exception (regex over string literals, not symbol names).
5. **Repo-relative paths in all artifacts.** Per `.claude/agents/docs/path-conventions.md`.
6. **Stable IDs.** Task IDs `P6-1` through `P6-13` are monotonic and immutable once written. Phase 7 may rename, but Phase 6 does not renumber.
7. **No new dependencies, no toolchain changes.** This phase uses only `ast-index`, `Grep`, `Read`, and editing of the Phase 6 tasks file.

---

## 6. Risks

| Risk | Mitigation |
|---|---|
| Vague finding text undermines Phase 7 actionability | Schema in §3 is mandatory; every entry validated against it before phase exits |
| Grep sweep misses a non-standard logging idiom (e.g. `developer.log` aliased) | Sweep regex covers `dart:developer`, `log\(`, and `Logger` literal; manual file scan during ast-index outline catches aliased imports |
| P6-1 severity is later disputed in Phase 7 because the silent fallback is interpreted as "documented behavior" | The escalation rationale in P6-1 description cites Resolved Question Q1 and PRD Constraint 10 explicitly; Phase 7 triage owns the final classification |
| Phase 1 F-2 and Phase 3 TASK-O6 get merged into a single task in Phase 7 despite Phase 6 separating them | Resolved Question Q2 mandates two tasks; P6-3 and P6-4 carry distinct file scopes and fix scopes that prevent mechanical merge |
| `pubspec.yaml` review is interpreted as a "positive finding without engineering task", confusing Phase 7 | P6-12 is explicitly recorded as confirmed-clean and tagged IMPROVEMENT to keep it in the audit trail without becoming an action item |
| `onLog` metadata leak (P6-9) is conflated with the absent-logging gaps (P6-5..P6-8) | P6-9 is filed as IMPROVEMENT (caller opt-in only) and is scoped distinctly from the structured-observability gaps |

---

## 7. Dependencies

### 7.1 Prior phases (must be complete before this phase starts)

- **Phase 1** — HPKE audit complete with Finding F-2 (no key zeroization in `HpkeSenderContext`). Status: complete.
- **Phase 3** — OHTTP audit complete with Finding TASK-O6 (no zeroization of `OhttpEncapsulateResult` fields). Status: complete.
- **Phase 4** — Client audit complete with Findings C-1 (no https enforcement) and C-4 (`sendDirect()` no warning). Status: complete.
- **Phase 5** — Test suite coverage gaps review. Status: complete.

### 7.2 Source artifacts (must exist and be readable)

- `lib/src/ohttp_client.dart`, `lib/src/ohttp.dart`, `lib/src/hpke.dart`, `lib/src/bhttp.dart`.
- `example/ohttp_dart_example.dart`.
- `pubspec.yaml`.

### 7.3 Reference documents

- `specs/.current/AW-2865/vision.md` — severity classification (§2) and observability gaps/constraints (§7).
- `specs/.current/AW-2865/phase-6/prd.md` — scope, user stories, scenarios, constraints, resolved questions.
- `specs/.current/AW-2865/phase-6/research.md` — source-confirmed findings backing each P6-N entry.

### 7.4 Tooling

- `ast-index` (per `.claude/rules/ast-index.md`) for symbol resolution.
- `Grep` for the single forbidden-pattern sweep in §4.1.
- `Read` for line-range-targeted file inspection.

### 7.5 Downstream consumers

- **Phase 7** — cross-phase severity triage. Consumes the 13 P6-N entries plus OG-1..OG-5 and LC-1..LC-6 from `phase-6/tasks.md` and reconciles them with Phases 1–5 findings.

---

## 8. Open Questions

No open questions block Phase 6 execution. The four technical questions below surfaced during Phase 6 research (`research.md §8`) and are carried forward to Phase 7 scoping — they are not gating items for Phase 6 closure.

1. **`onLog` production guidance (carry-forward).** Is there a plan to document that `onLog: print` in the example must not be adapted as `onLog: structuredLogger.debug` in production? Affects whether P6-9 needs a dedicated API-documentation task in Phase 7 or is absorbed into P6-11.

2. **`sendDirect()` retention rationale (carry-forward).** Was `sendDirect()` designed as a testing utility (per the "for comparison" doc comment), or as a production fallback? If it is a testing utility, the Phase 7 fix scope for P6-1 might be deprecation or removal rather than adding a warning. This affects the fix-shape of the BLOCKER task.

3. **`OhttpEncapsulateResult` ownership (carry-forward).** After `OhttpClient.send()` calls `ohttpDecapsulate`, the `OhttpEncapsulateResult` object is no longer referenced. Should P6-4's zeroization fix live in a new `dispose()` method on `OhttpEncapsulateResult`, or should `OhttpClient.send()` explicitly zero the fields before returning? The decision introduces (or avoids) a new disposal pattern at layer 3.

4. **AEAD failure audit trail (carry-forward).** OG-3 / P6-7 notes that `SecretBoxAuthenticationError` propagates without an audit trail. In wallet-integration context, should AEAD failures trigger a rate-limited security alert or just a structured log event? Affects whether the Phase 7 task is "add structured log" or "add typed exception + alert hook".

---

## 9. Phase Exit Conditions

Phase 6 closes when **all** of the following are true:

1. Tasks 6.1 through 6.11 are checked off in `phase-6/tasks.md`.
2. All 13 draft entries (P6-1 through P6-13) are recorded under the consolidated draft-entries block in `phase-6/tasks.md`, conforming to the schema in §3.
3. The five-gap observability catalog (OG-1 through OG-5) and the six-constraint logging matrix (LC-1 through LC-6) are recorded in `phase-6/tasks.md`.
4. P6-1 is recorded as BLOCKER with the silent-fallback rationale citing `lib/src/ohttp_client.dart:52` and the `effectiveDirectBaseUrl` getter.
5. P6-3 (hpke scope) and P6-4 (ohttp scope) are recorded as two separate HIGH entries — not merged.
6. P6-12 is recorded as confirmed-clean IMPROVEMENT documenting the `cryptography: ^2.9.0` tight caret and absence of FFI/native deps.
7. No file in `lib/`, `test/`, or `example/` was modified.
8. The four technical questions in §8 are recorded as Phase 7 carry-forward items.
