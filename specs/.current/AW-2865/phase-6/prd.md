# AW-2865 Phase 6: Privacy Risk Review, Observability Gaps Audit, and Documentation Review

Status: PRD_READY

## Context / Idea

This is phase 6 of the AW-2865 investigation into `ohttp_dart` for potential use in a non-custodial crypto wallet. The overall ticket goal is to identify risks and form subsequent engineering tasks — no code changes are made within this investigation.

Phases 1–5 audited the four implementation layers (HPKE, BHTTP, OHTTP encap/decap, OhttpClient) and the test suite. Phase 6 is the final audit phase. It performs three cross-cutting reviews that are not layer-specific:

1. **Privacy risk review** — assess whether the package's current state preserves OHTTP's core unlinkability guarantee when used in a crypto wallet, and identify gaps that undermine it.
2. **Observability gaps audit** — catalog the five structured observability gaps identified in `vision.md §7` and establish the six logging constraints that any future logging must respect.
3. **Documentation review** — audit `example/ohttp_dart_example.dart` and `pubspec.yaml` for documented limitations, version pinning integrity, and absence of native dependencies.

Inherited findings from prior phases that are directly relevant to this phase:
- Phase 1 F-2: No zeroization of `HpkeSenderContext.key`, `baseNonce`, `exporterSecret` after use.
- Phase 3 TASK-O6: No zeroization of `enc` and `exportedSecret` in `OhttpEncapsulateResult`.
- Phase 4 Finding C-1: No `https`-scheme enforcement on `gatewayBaseUrl`.
- Phase 4 Finding C-4: `sendDirect()` has no doc comment, no assert, and no privacy warning.

---

## Goals

1. Confirm whether `sendDirect()` is documented with a privacy-impact warning anywhere in the source, and if not, classify this as a production-blocking privacy risk.
2. Confirm that no logging of inner request URL, method, headers, body, response body, or response headers exists anywhere in `lib/`.
3. Confirm that no logging of cryptographic material (`enc`, `exportedSecret`, HKDF-derived keys) exists in `lib/src/ohttp.dart` or `lib/src/hpke.dart`.
4. Assess the `example/ohttp_dart_example.dart` file for documented limitations: real-gateway requirement, `sendDirect()` privacy bypass notice, known prerequisites.
5. Assess `pubspec.yaml` for dependency version pinning integrity and absence of native-code dependencies.
6. Produce a complete catalog of the five observability gaps and six logging constraints (from `vision.md §7`) as draft task entries for use in Phase 7.
7. Produce at least one BLOCKER or HIGH privacy finding tied to the `sendDirect()` bypass and at least one tied to key-material zeroization absence — these are the minimum outputs required by the phase 6 acceptance criteria.

---

## User Stories

**US-1 — Wallet developer using `OhttpClient.send()`**
As a wallet developer integrating `ohttp_dart`, I need to know whether calling `OhttpClient.send()` reliably routes all requests through the OHTTP protocol, so that I can trust that the wallet IP address is never exposed to the target server.

Acceptance: The audit either confirms that `send()` always uses the OHTTP path, or surfaces a finding that documents how misconfiguration can route traffic outside of OHTTP.

**US-2 — Wallet developer using `OhttpClient.sendDirect()`**
As a wallet developer, I need any non-OHTTP code path to be clearly marked with a privacy warning, so that I cannot accidentally call it and silently expose wallet IP addresses to the target server.

Acceptance: The audit confirms whether `sendDirect()` carries a privacy-impact doc comment or assertion, and if not, raises a HIGH finding. Escalation to BLOCKER is contingent on task 6.1 confirming whether the `effectiveDirectBaseUrl` fallback to `gatewayBaseUrl` makes `sendDirect()` callable without any explicit `directBaseUrl` configuration.

**US-3 — Security auditor reviewing logging safety**
As a security auditor, I need confirmation that no key material, inner request content, or response body is ever written to any logging output in `lib/`, so that I can certify the package for use in a high-value-asset wallet.

Acceptance: The audit sweeps all four source files for `dart:developer`, `print`, `debugPrint`, `log`, `Logger`, and string interpolation that references sensitive identifiers, and produces a confirmed absence or a specific finding.

**US-4 — Developer reviewing the example file**
As a developer onboarding to `ohttp_dart`, I want the example file to document that a real OHTTP relay URL is required, that `sendDirect()` bypasses OHTTP, and what the prerequisites are, so that I cannot run the example against a placeholder URL and draw wrong conclusions.

Acceptance: The audit catalogs what the example documents and what it omits, classified by severity. Any real gateway URL found in source control is flagged as IMPROVEMENT only (not HIGH), per the investigation scope for this ticket.

**US-5 — Dependency manager reviewing supply-chain risk**
As a dependency manager, I need to confirm that `package:cryptography` is pinned to a specific caret constraint (not `any`), and that no transitive dependency introduces native code or FFI, so that a dependency update cannot silently change cipher behavior.

Acceptance: The audit confirms the exact version constraint on `package:cryptography` and checks for `dart:ffi` / `package:ffi` / platform-specific dependencies.

---

## Main Scenarios

### Scenario A — `sendDirect()` bypass (privacy)

**Trigger:** A wallet developer configures `OhttpGatewayConfig` and calls `sendDirect()` either intentionally (as a fallback) or by mistake.

**Current behavior:**
- `sendDirect()` performs a plain HTTP request to `directBaseUrl` (falls back to `gatewayBaseUrl` when `directBaseUrl` is null — an additional footgun from Phase 4 Finding C-4).
- There is no doc comment on `sendDirect()`, no `@Deprecated` annotation, no `assert`, and no runtime warning.
- The wallet's IP address is exposed directly to the target server, defeating OHTTP's core unlinkability guarantee.

**Risk classification:** HIGH. Escalation to BLOCKER is deferred until task 6.1 re-confirms that the `effectiveDirectBaseUrl` fallback to `gatewayBaseUrl` enables silent non-OHTTP routing. Until that confirmation is in writing in the task findings, this remains HIGH.

**Expected finding:** At minimum one HIGH finding; escalate to BLOCKER in the task entry if and when task 6.1 confirms the silent-fallback behavior.

---

### Scenario B — Plain-HTTP outer channel

**Trigger:** A wallet developer constructs `OhttpGatewayConfig(gatewayBaseUrl: 'http://relay.example.com', ...)`.

**Current behavior (confirmed in Phase 4 Finding C-1):**
- No scheme validation in the `OhttpGatewayConfig` constructor.
- Both the KeyConfig GET and the OHTTP POST are made over plain HTTP.
- The outer OHTTP ciphertext is transmitted in cleartext, allowing a network observer to correlate timing even though content is encrypted.
- RFC 9458 §1 requires TLS on the outer channel.

**Risk classification:** HIGH.

---

### Scenario C — Key material persistence in memory

**Trigger:** Any call to `OhttpClient.send()` completes (success or failure).

**Current behavior (confirmed in Phase 1 F-2 and Phase 3 TASK-O6):**
- `HpkeSenderContext.key` (16 B AES-128 key), `baseNonce` (12 B), and `exporterSecret` (32 B) remain in Dart heap memory after `seal()` returns.
- `OhttpEncapsulateResult.enc` (32 B ephemeral X25519 public key) and `exportedSecret` (16 B) are not zeroed after `ohttpDecapsulate` completes.
- Dart's garbage collector does not guarantee prompt collection; in a memory-dump or heap-analysis attack these values may be readable.

**Risk classification:** HIGH. Two separate tasks are required — one for `hpke.dart` (Phase 1 F-2 scope) and one for `ohttp.dart` (Phase 3 TASK-O6 scope) — because the fix scopes differ. These are not merged into a single BLOCKER task. Both carry HIGH severity in Phase 7 task entries.

---

### Scenario D — Inner request content in logs

**Trigger:** A developer adds diagnostic logging to `lib/` (or the package already contains it).

**Current behavior:** `vision.md §7` confirms no logging exists in the current source. Phase 6 confirms this is still true (or surfaces a finding if logging was silently added).

**Risk classification:** Any discovery of logging that references inner request URL, method, headers, body, `targetAuthority`, `enc`, or key material is an immediate BLOCKER.

---

### Scenario E — Dependency version drift

**Trigger:** A `dart pub upgrade` bumps `package:cryptography` past the pinned version.

**Current behavior (to be confirmed in Phase 6):**
- If `package:cryptography` is constrained with a tight caret (`^x.y.z`), major API or cipher behavior changes are blocked.
- If constrained with `any`, any version is accepted, including ones with changed cipher semantics.

**Risk classification:** HIGH if a wildcard constraint is found; IMPROVEMENT if a tight caret is confirmed.

---

### Scenario F — Example file misleads a developer

**Trigger:** A developer clones the repo and runs `example/ohttp_dart_example.dart` without reading source.

**Current behavior (to be confirmed in Phase 6):**
- The CLAUDE.md note says the example "needs a real gateway URL — edit before running."
- It is unclear whether: (a) a placeholder URL is hardcoded and fails silently or loudly, (b) `sendDirect()` is called in the example without a privacy notice, (c) prerequisites are stated.

**Risk classification:** IMPROVEMENT if the example merely lacks comments or contains a real (non-sensitive) gateway URL. HIGH only if hardcoded credentials, private tokens, or non-public private URLs are found in source control.

---

### Scenario G — Encapsulation timing observability gap

**Trigger:** A developer or operator needs to profile or troubleshoot OHTTP encapsulation performance in a wallet integration.

**Current behavior:** No structured timing instrumentation exists around `ohttpEncapsulate` or `ohttpDecapsulate`.

**Risk classification:** IMPROVEMENT. This classification is authoritative for Phase 7 task prioritization per `vision.md §7` and is not upgraded absent a concrete wallet performance requirement.

---

## Success / Metrics

The phase is complete when all of the following are met:

1. All eleven tasks in `specs/.current/AW-2865/phase-6/tasks.md` (6.1–6.11) are checked off with concrete findings.
2. At least one BLOCKER or HIGH finding is tied to the `sendDirect()` bypass (Scenario A).
3. At least one BLOCKER or HIGH finding is tied to absence of key-material zeroization in `hpke.dart` (Scenario C, hpke scope).
4. At least one BLOCKER or HIGH finding is tied to absence of key-material zeroization in `ohttp.dart` (Scenario C, ohttp scope).
5. The complete catalog of five observability gaps (task 6.10) is documented with file, severity, and draft task ID.
6. The six logging constraints (task 6.11) are documented as a reference for Phase 7 task formulation.
7. `pubspec.yaml` dependency review is complete: `package:cryptography` constraint noted, absence of FFI/native deps confirmed.
8. All findings are recorded as draft task entries (stable ID, severity, file:lines, concern category) ready for Phase 7 compilation.
9. No file in `lib/`, `test/`, or `example/` is modified.

---

## Constraints and Assumptions

1. **Read-only audit.** No file in `lib/`, `test/`, or `example/` is modified. This phase produces only draft task entries in `specs/.current/AW-2865/phase-6/tasks.md`.
2. **No code changes, no new dependencies.** Adding logging, assertions, or documentation to source files is out of scope for this investigation ticket.
3. **Severity classification must follow vision §2.** Every finding carries exactly one of: `BLOCKER`, `HIGH`, `IMPROVEMENT`.
4. **Findings must cite specific file and line range.** Vague statements ("crypto could be weak") are not acceptable per vision §2.
5. **Cross-phase continuity.** Findings that compound prior-phase risks (e.g., Phase 1 F-2 + Phase 3 TASK-O6 → combined key-zeroization concern) are explicitly cross-referenced in the draft task entry, but are kept as two separate task entries with different fix scopes.
6. **No live gateway required.** All checks in this phase are static source review; no network access is needed.
7. **`package:cryptography` is the sole cryptographic primitive source.** Its version constraint must be confirmed; no alternative crypto library is evaluated in this phase.
8. **Dart garbage collection does not guarantee zeroization.** The risk of key material persistence in memory is a design-level finding, not solvable by relying on GC behavior.
9. **Example URL policy.** A real (but non-sensitive) gateway URL in the example file is classified as IMPROVEMENT for this investigation context. Only credentials, private tokens, or non-public URLs would escalate to HIGH or BLOCKER.
10. **`sendDirect()` severity floor.** The finding for `sendDirect()` enters Phase 7 as HIGH. Escalation to BLOCKER requires explicit confirmation from task 6.1 that the `effectiveDirectBaseUrl` fallback makes the bypass silently reachable without developer intent.

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| `sendDirect()` is callable without any explicit non-OHTTP intent due to `effectiveDirectBaseUrl` fallback | High (confirmed in Phase 4 C-4) | HIGH (BLOCKER candidate pending task 6.1 confirmation) | Enter as HIGH; escalate to BLOCKER in draft task if task 6.1 confirms silent fallback |
| Plain-HTTP `gatewayBaseUrl` accepted silently | High (confirmed in Phase 4 C-1) | HIGH — outer channel unprotected | Draft task for `https`-scheme enforcement in gateway config constructor |
| Key material survives in Dart heap after use — `hpke.dart` scope (Phase 1 F-2) | High (confirmed) | HIGH — high-value target; memory-dump exposure | Separate draft task for explicit byte-overwrite of `HpkeSenderContext` fields |
| Key material survives in Dart heap after use — `ohttp.dart` scope (Phase 3 TASK-O6) | High (confirmed) | HIGH — high-value target; memory-dump exposure | Separate draft task for explicit byte-overwrite of `OhttpEncapsulateResult` fields |
| `example/` file contains hardcoded credentials or private URLs | Low | BLOCKER if present — source-control exposure | Review example file in tasks 6.5–6.6; escalate if sensitive material found; treat real-but-public URL as IMPROVEMENT |
| `package:cryptography` constrained with `any` | Low (unlikely but unconfirmed) | HIGH — silent cipher behavior change on upgrade | Confirm exact constraint in tasks 6.7–6.9; escalate if wildcard found |
| Logging silently added to `lib/` since `vision.md` was written | Very low | BLOCKER if present — inner request content in logs | Sweep all four source files for print/log calls in tasks 6.2–6.4 |
| Phase 6 findings are too vague to produce independently actionable Phase 7 tasks | Medium | Delays follow-up task creation | Enforce stable-ID + file:lines + severity format for every draft entry |

---

## Resolved Questions

The following questions were open in the draft and have been resolved by the user prior to PRD finalization.

**Q1 — `sendDirect()` escalation to BLOCKER**
Resolution: Severity stays HIGH entering Phase 7. Task 6.1 must re-confirm the `effectiveDirectBaseUrl` fallback behavior. If task 6.1 confirms that `sendDirect()` is silently callable without explicit `directBaseUrl` configuration, the draft task entry is escalated to BLOCKER at that point. No pre-emptive escalation before confirmation.

**Q2 — Zeroization task merging (Phase 1 F-2 vs Phase 3 TASK-O6)**
Resolution: Keep as two separate HIGH tasks. The fix scope differs: Phase 1 F-2 targets `HpkeSenderContext` fields in `lib/src/hpke.dart`; Phase 3 TASK-O6 targets `OhttpEncapsulateResult` fields in `lib/src/ohttp.dart`. Phase 7 assigns them distinct task IDs.

**Q3 — Example file gateway URL policy**
Resolution: Acceptable for investigation context. If a real (non-sensitive) gateway URL is present in the example, it is flagged as IMPROVEMENT only, not HIGH. Only credentials, private tokens, or non-public private URLs would escalate beyond IMPROVEMENT.

**Q4 — `pubspec.yaml` constraint format**
Resolution: No pre-determination — task 6.7–6.8 confirms the actual constraint. Outcome: HIGH if wildcard, IMPROVEMENT if tight caret. No change to task scope.

**Q5 — Encapsulation timing severity**
Resolution: Classified as IMPROVEMENT per `vision.md §7`. This classification is authoritative for Phase 7 prioritization and is not upgraded unless a concrete wallet performance requirement is stated.

---

## Phase 7 Scope Note

Phase 7 is not a mechanical compilation of Phase 6 findings. It performs a **cross-phase severity triage** across all phases (1–6): reviewing every draft task entry from all six investigation phases, reconciling duplicate or overlapping findings, confirming severity classifications in light of the full picture, and producing a consolidated, prioritized task list ready for engineering execution. This includes re-evaluating any "BLOCKER candidate" items (such as the `sendDirect()` fallback finding) that required cross-phase confirmation before a final severity could be assigned.

---

## Open Questions

None. All open questions from the draft have been resolved. See "Resolved Questions" above.
