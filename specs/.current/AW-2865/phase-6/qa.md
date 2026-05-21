# QA Report — AW-2865 Phase 6
# Privacy Risk Review, Observability Gaps Audit, and Documentation Review

**Ticket:** AW-2865
**Phase:** 6
**Date:** 2026-05-21
**Auditor:** QA agent (read-only verification)
**Verdict:** PASS

---

## 1. Phase Scope

Phase 6 is a completed, read-only static-source audit covering three cross-cutting review streams:

1. Privacy risk review across `lib/src/ohttp_client.dart`, `lib/src/ohttp.dart`, and `lib/src/hpke.dart`.
2. Observability gaps catalog aligned with `vision.md §7` (five gaps, six logging constraints).
3. Documentation review of `example/ohttp_dart_example.dart` and `pubspec.yaml`.

No code was written or changed. The deliverable is `specs/.current/AW-2865/phase-6/tasks.md` with all 11 tasks checked off and 13 draft findings (P6-1 through P6-13) appended. This QA report verifies that every acceptance criterion and exit condition is met.

---

## 2. Task Completion Check

All 11 tasks are marked `[x]` in `specs/.current/AW-2865/phase-6/tasks.md`.

| Task | Description | Status |
|---|---|---|
| 6.1 | Confirm `sendDirect()` has no privacy-impact warning | DONE |
| 6.2 | Confirm no inner request / response logging in `lib/` | DONE |
| 6.3 | Confirm no logging of `enc`, `exportedSecret`, or HKDF outputs in `ohttp.dart` | DONE |
| 6.4 | Confirm no logging of ephemeral key material in `hpke.dart` | DONE |
| 6.5 | Read `example/ohttp_dart_example.dart` | DONE |
| 6.6 | Note example documentation gaps: gateway URL requirement, `sendDirect()` bypass warning, prerequisites | DONE |
| 6.7 | Read `pubspec.yaml` | DONE |
| 6.8 | Note `package:cryptography` version pinning | DONE |
| 6.9 | Note absence of FFI / native-code dependencies | DONE |
| 6.10 | Catalog five observability gaps (OG-1..OG-5) | DONE |
| 6.11 | Record six logging constraints (LC-1..LC-6) | DONE |

**Result: 11/11 tasks complete. PASS.**

---

## 3. Acceptance Criteria Verification

Per `specs/.current/AW-2865/phase-6/prd.md §Success / Metrics`:

### 3.1 Tasks 6.1–6.11 checked off with concrete findings

PASS — verified by direct inspection of `phase-6/tasks.md:58–78`. Every checkbox is `[x]`; the confirmation summary table in the findings block records the specific file:line evidence for each task.

### 3.2 At least one BLOCKER or HIGH finding tied to `sendDirect()` bypass (Scenario A)

PASS — exceeded.

- **P6-1 (BLOCKER):** `sendDirect()` at `lib/src/ohttp_client.dart:148–171` carries no `@Deprecated`, no `assert`, no privacy-impact warning. The `effectiveDirectBaseUrl` getter at line 52 silently resolves a null `directBaseUrl` to `gatewayBaseUrl`, making OHTTP bypass the default silent code path. Source-confirmed escalation from HIGH (Phase 4 C-4) to BLOCKER per PRD Resolved Question Q1.
- **P6-8 (HIGH):** Observability companion — `sendDirect()` emits no signal when the plaintext path is taken (no `onLog` parameter, no structured event).
- **P6-13 (IMPROVEMENT):** API-visibility companion — `effectiveDirectBaseUrl` is a public getter that exposes the silent-fallback surface.

### 3.3 At least one HIGH finding tied to key-material zeroization in `hpke.dart` (Scenario C, hpke scope)

PASS — P6-3 (HIGH): `HpkeSenderContext` at `lib/src/hpke.dart:258–270` retains `key` (16 B AES-128), `baseNonce` (12 B), and `exporterSecret` (32 B) as `final Uint8List` fields after `seal()` returns. No explicit zeroization; Dart GC does not guarantee prompt collection. Cross-ref: Phase 1 F-2.

### 3.4 At least one HIGH finding tied to key-material zeroization in `ohttp.dart` (Scenario C, ohttp scope)

PASS — P6-4 (HIGH): `OhttpEncapsulateResult.enc` (32 B) and `.exportedSecret` (16 B) at `lib/src/ohttp.dart:105–120` are not zeroed after `ohttpDecapsulate` completes. Local HKDF outputs `aeadKey` and `aeadNonce` at lines 196–207 also remain on the heap until GC. Cross-ref: Phase 3 TASK-O6.

### 3.5 Five observability gaps (OG-1..OG-5) documented with file, severity, and draft task ID

PASS — complete catalog in `phase-6/tasks.md`:

| Gap ID | Missing event | File | Lines | Severity | Draft task |
|---|---|---|---|---|---|
| OG-1 | KeyConfig fetch succeeded / failed | `lib/src/ohttp_client.dart` | 81–89 | HIGH | P6-5 |
| OG-2 | Gateway POST status code | `lib/src/ohttp_client.dart` | 115–122 | HIGH | P6-6 |
| OG-3 | AEAD authentication failure on decap | `lib/src/ohttp.dart` | 218–224 | HIGH | P6-7 |
| OG-4 | `sendDirect()` invoked (OHTTP bypass) | `lib/src/ohttp_client.dart` | 148–171 | HIGH | P6-8 |
| OG-5 | OHTTP encapsulation completed (timing) | `lib/src/ohttp.dart` | 131–167 | IMPROVEMENT | P6-10 |

### 3.6 Six logging constraints (LC-1..LC-6) documented

PASS — complete matrix in `phase-6/tasks.md`:

| ID | Constraint |
|---|---|
| LC-1 | No key material of any kind (`HpkeSenderContext.key`, `exporterSecret`, any HKDF output) |
| LC-2 | No ephemeral `enc` value — leakage alongside ciphertext breaks forward secrecy |
| LC-3 | No inner request URL, path, method, headers, or body |
| LC-4 | No `targetAuthority` from `OhttpGatewayConfig` |
| LC-5 | No response body or response headers |
| LC-6 | No `gatewayBaseUrl` at DEBUG or lower in production builds |

### 3.7 `pubspec.yaml` dependency review complete

PASS — P6-12 (IMPROVEMENT, confirmed-clean): `cryptography: ^2.9.0` (tight caret), `http: ^1.6.0` (tight caret). No wildcard `any` constraint. No `dart:ffi`, `package:ffi`, or platform-specific packages in direct dependencies. No engineering task required; recorded to preserve the audit trail.

### 3.8 All 13 draft entries conform to the schema in `plan.md §3`

PASS — every P6-N entry carries Task ID, Severity, File (repo-relative), Lines, Concern category, and Description. Cross-refs are present on the four entries that compound prior-phase findings (P6-1, P6-2, P6-3, P6-4). No hybrid severity labels appear; every entry carries exactly one of BLOCKER / HIGH / IMPROVEMENT per `vision.md §2`.

### 3.9 No file in `lib/`, `test/`, or `example/` was modified

PASS — confirmed below in §5.

---

## 4. Plan Exit Conditions Verification

Per `specs/.current/AW-2865/phase-6/plan.md §9`:

| Exit condition | Status |
|---|---|
| 1. Tasks 6.1–6.11 checked off | PASS |
| 2. All 13 draft entries (P6-1..P6-13) recorded, conforming to §3 schema | PASS |
| 3. OG-1..OG-5 catalog and LC-1..LC-6 matrix recorded | PASS |
| 4. P6-1 recorded as BLOCKER with silent-fallback rationale citing `lib/src/ohttp_client.dart:52` | PASS — P6-1 entry names line 52 (`effectiveDirectBaseUrl`) and lines 148–171 (`sendDirect()` body) |
| 5. P6-3 (hpke scope) and P6-4 (ohttp scope) recorded as two separate HIGH entries | PASS — disjoint files (`hpke.dart` vs `ohttp.dart`), disjoint fix scopes; not merged |
| 6. P6-12 recorded as confirmed-clean IMPROVEMENT with `cryptography: ^2.9.0` and absence of FFI/native deps | PASS |
| 7. No file in `lib/`, `test/`, or `example/` was modified | PASS — see §5 |
| 8. Four Phase 7 carry-forward questions recorded | PASS — recorded in `phase-6/tasks.md §Phase 7 Carry-Forward Questions` matching `plan.md §8.1–§8.4` exactly |

**All 8 exit conditions satisfied.**

---

## 5. Read-Only Constraint Verification

The phase constraint requires zero modifications to files under `lib/`, `test/`, or `example/`. Verified by:

1. The `review.md` Phase 6 section (lines 921–922 and 934) confirms: `git diff HEAD -- lib/ test/ example/ pubspec.yaml` returned zero lines at review time.
2. The git status at the start of this session confirms the branch `feature/AW-2865-investigation-ohttp_dart` is clean (no uncommitted changes).
3. All five source files audited (`lib/src/ohttp_client.dart`, `lib/src/ohttp.dart`, `lib/src/hpke.dart`, `example/ohttp_dart_example.dart`, `pubspec.yaml`) are present and unmodified.
4. The only artifact produced during Phase 6 is the findings block appended to `specs/.current/AW-2865/phase-6/tasks.md`.

**Result: PASS — zero source mutations confirmed.**

---

## 6. Findings Block Consistency Check

### 6.1 Draft entries count

The tasks file contains exactly 13 draft entries (P6-1 through P6-13), matching the plan §2.2 and §2.3 specification.

### 6.2 Severity distribution

| Severity | Count | Entries |
|---|---|---|
| BLOCKER | 1 | P6-1 |
| HIGH | 7 | P6-2, P6-3, P6-4, P6-5, P6-6, P6-7, P6-8 |
| IMPROVEMENT | 5 | P6-9, P6-10, P6-11, P6-12, P6-13 |

Severity distribution is consistent with `plan.md §2.3` (the plan's severity matrix) and with vision §2 (three-tier model: BLOCKER / HIGH / IMPROVEMENT). No hybrid labels, no MAINTENANCE tier, no TBD.

### 6.3 Cross-references to prior phases

All cross-references are accurate and cite IDs that exist in prior-phase deliverables:

| Entry | Cross-ref cited | Prior-phase origin |
|---|---|---|
| P6-1 | Phase 4 C-4 | `sendDirect()` no warning, `phase-4/research.md` |
| P6-2 | Phase 4 C-1 | No `https`-scheme enforcement, `phase-4/research.md` |
| P6-3 | Phase 1 F-2 | `HpkeSenderContext` key zeroization absence, `phase-1/tasks.md` |
| P6-4 | Phase 3 TASK-O6 | `OhttpEncapsulateResult` zeroization absence, `phase-3/tasks.md` |

No prior-phase description is reproduced verbatim (per `plan.md §4.2`). All paths in the findings block are repo-relative; no absolute filesystem paths appear.

### 6.4 P6-1 BLOCKER escalation rationale

The escalation from HIGH (Phase 4 C-4) to BLOCKER satisfies the conditions in PRD Resolved Question Q1 and `plan.md §2.3` note: Phase 6 source-confirmed the silent fallback. The `effectiveDirectBaseUrl` getter at `lib/src/ohttp_client.dart:52` (`directBaseUrl ?? gatewayBaseUrl`) is the specific evidence; the research.md §3.1 and §5.1 entries document this confirmation at source level before the escalation was recorded. The escalation is justified in the entry body and is not pre-emptive.

### 6.5 P6-3 / P6-4 separation

P6-3 targets `lib/src/hpke.dart:258–270` (class `HpkeSenderContext`, fix scope: the HPKE layer). P6-4 targets `lib/src/ohttp.dart:105–120, 196–207` (class `OhttpEncapsulateResult` and local HKDF variables, fix scope: the OHTTP layer). The two entries have disjoint file scopes and distinct fix patterns, satisfying PRD Resolved Question Q2 (do not merge).

### 6.6 Observability catalog completeness

The five OG entries cover every gap enumerated in `vision.md §7`. Severities match vision §7 exactly (OG-1..OG-4 are HIGH; OG-5 is IMPROVEMENT). Each entry carries a file and line range grounded in the actual source.

### 6.7 Logging constraints completeness

The six LC entries enumerate every constraint from `vision.md §7`. Each row identifies the specific identifiers affected. The matrix is self-consistent: LC-1 and LC-2 cover key material and enc; LC-3 through LC-5 cover inner request and response content; LC-6 covers gateway metadata in production logs.

### 6.8 Path discipline

All `File:` fields in the 13 draft entries use repo-relative paths (`lib/src/ohttp_client.dart`, `lib/src/ohttp.dart`, `lib/src/hpke.dart`, `example/ohttp_dart_example.dart`, `pubspec.yaml`). No absolute paths are present in the artifact. Compliant with `.claude/agents/docs/path-conventions.md`.

---

## 7. Positive Scenarios Verified

### Scenario A — `sendDirect()` bypass (privacy)

CONFIRMED via task 6.1. Source evidence: doc comment "Send a direct HTTP request (for comparison)." at line 147; no `@Deprecated`, no `assert`, no warning. `effectiveDirectBaseUrl` at line 52 confirms the silent fallback. Finding P6-1 (BLOCKER) and P6-8 (HIGH) both raised.

### Scenario B — Plain-HTTP outer channel

CONFIRMED via prior-phase cross-reference (Phase 4 C-1, re-confirmed in Phase 6 research). Finding P6-2 (HIGH) raised. `OhttpGatewayConfig` constructor at `lib/src/ohttp_client.dart:43–51` performs no scheme validation.

### Scenario C — Key material persistence in memory

CONFIRMED in two separate findings: P6-3 (HIGH, `hpke.dart` scope) and P6-4 (HIGH, `ohttp.dart` scope). Both cite concrete Uint8List field declarations with no zeroization method present.

### Scenario D — Inner request content in logs

CONFIRMED ABSENT. Grep across `lib/src/` for `print(`, `debugPrint`, `dart:developer`, `log(`, `Logger` returned zero matches. Tasks 6.2, 6.3, 6.4 all closed as confirmed-absent. No finding raised; absence is the desired outcome.

### Scenario E — Dependency version drift

CONFIRMED CLEAN. P6-12 (IMPROVEMENT) documents `cryptography: ^2.9.0` (tight caret) and `http: ^1.6.0` (tight caret). No wildcard constraint. No engineering action required.

### Scenario F — Example file misleads a developer

CONFIRMED as documentation gap only. P6-11 (IMPROVEMENT) catalogs three omissions: (a) no comment requiring a real OHTTP relay URL, (b) no `sendDirect()` mention or bypass warning, (c) no prerequisites block. Placeholder URLs are obviously non-sensitive (`'https://your-gateway.example.com'`). No credentials or private URLs found; severity correctly stays at IMPROVEMENT.

### Scenario G — Encapsulation timing observability gap

CONFIRMED as IMPROVEMENT. P6-10 records the absence of timing instrumentation around `ohttpEncapsulate` at `lib/src/ohttp.dart:131–167`. OG-5 carries IMPROVEMENT severity per `vision.md §7`. No upgrade attempted.

---

## 8. Negative and Edge Cases

### Edge: `effectiveDirectBaseUrl` with only required parameters

Confirmed footgun: when `directBaseUrl` is not set (the common case), `effectiveDirectBaseUrl` resolves to `gatewayBaseUrl`, causing `sendDirect()` to silently issue a plaintext POST to the OHTTP relay. This edge is source-confirmed and documented in P6-1 (BLOCKER). No additional edge case was missed.

### Edge: `onLog` wired to a production structured logger

The `onLog` callback is opt-in and not invoked unless the caller provides it. If a caller wires it to a production logger, lines 77 and 113 of `lib/src/ohttp_client.dart` will interpolate `gatewayBaseUrl`, violating LC-6. This edge is documented in P6-9 (IMPROVEMENT). The risk is caller-controlled; no hard-coded logger exists in `lib/`.

### Edge: `SecretBoxAuthenticationError` from `package:cryptography`

Propagates unwrapped from `ohttpDecapsulate` at `lib/src/ohttp.dart:218–224`. A caller cannot distinguish an AEAD auth failure from other runtime exceptions without catching the `package:cryptography` internal type, leaking implementation-internal types into the public API surface. Documented in P6-7 (HIGH) and OG-3.

### Edge: Example `targetAuthority` equals `gatewayBaseUrl` host

In `example/ohttp_dart_example.dart`, `targetAuthority` is set to `'your-gateway.example.com'`, the same host as `gatewayBaseUrl`. In real OHTTP usage, the target server should differ from the relay. This is an example correctness gap, not a security finding, and is documented in P6-11 (IMPROVEMENT).

---

## 9. Risk Zones

The following items represent the highest-risk findings and must be addressed before wallet production use.

### BLOCKER — P6-1: `sendDirect()` OHTTP bypass via silent `effectiveDirectBaseUrl` fallback

**File:** `lib/src/ohttp_client.dart:52, 148–171`

This is the only BLOCKER. A wallet developer who constructs `OhttpGatewayConfig` with default parameters and calls `sendDirect()` will route plaintext traffic to the OHTTP relay with no runtime warning. OHTTP's unlinkability guarantee is completely defeated on this code path. Fix scope: add `@Deprecated` annotation, `assert`, or a strongly-worded doc comment; separately, consider removing `sendDirect()` from the public API (P6-13 addresses API visibility as a companion IMPROVEMENT).

### HIGH — P6-2: No `https`-scheme enforcement

**File:** `lib/src/ohttp_client.dart:43–51`

Plain HTTP `gatewayBaseUrl` silently accepted. Outer OHTTP ciphertext transmitted in cleartext. RFC 9458 §1 requires TLS on the outer channel.

### HIGH — P6-3, P6-4: Key material persistence in Dart heap

**Files:** `lib/src/hpke.dart:258–270`, `lib/src/ohttp.dart:105–120, 196–207`

`HpkeSenderContext` and `OhttpEncapsulateResult` fields are retained in memory after use. Dart GC does not guarantee prompt collection. High-value wallet processes are targets for memory-dump attacks. These are two separate HIGH tasks with different fix scopes.

### HIGH — P6-5, P6-6, P6-7, P6-8: Four observability gaps

All four map to existing code paths in `lib/src/ohttp_client.dart` and `lib/src/ohttp.dart`. The specific risk: a deployed wallet has no structured way to distinguish gateway misconfiguration from network failure (OG-1/OG-2), detect ciphertext tampering (OG-3), or detect an OHTTP bypass event (OG-4).

---

## 10. Automated vs. Manual Tests

### No automated tests apply to this phase

Phase 6 is a read-only static-source audit. All checks are manual source inspection and grep sweeps. There are no runnable test scenarios because:
- The logging absence check (tasks 6.2–6.4) is a grep pattern: `print(|debugPrint|dart:developer|Logger|log(` across `lib/src/` — confirmed zero matches.
- The `pubspec.yaml` check is a text read, not a runtime test.
- The example file check is a read and a structural review.
- The `sendDirect()` warning absence is a source inspection — a test cannot confirm the absence of a doc comment.

### Manual verification checklist

The following checks were performed manually during the audit and are recorded in the confirmation summary in `phase-6/tasks.md`:

| Check | Verified by | Result |
|---|---|---|
| `sendDirect()` doc comment read in full | Direct source read `lib/src/ohttp_client.dart:147` | No privacy warning present |
| Grep for logging in `lib/src/` | Single ripgrep sweep | Zero matches |
| `example/` read in full (38 lines) | Direct source read | No credentials; documented gaps noted |
| `pubspec.yaml` read in full (18 lines) | Direct source read | Tight caret constraints confirmed |
| `vision.md §7` gap/constraint enumeration | Cross-reference with source | All five gaps and six constraints catalog-complete |

---

## 11. Phase 7 Carry-Forward Questions

Four open questions are recorded in `phase-6/tasks.md` and are not gating items for Phase 6 closure. They are listed here for completeness and to confirm they are present in the deliverable:

1. `onLog` production guidance: does P6-9 need a dedicated API-documentation task in Phase 7, or is it absorbed into P6-11?
2. `sendDirect()` retention rationale: is it a testing utility (per "for comparison" doc) or a production fallback? Affects whether the Phase 7 fix for P6-1 is "add warning" vs "deprecate/remove."
3. `OhttpEncapsulateResult` ownership for P6-4 fix: `dispose()` method on the result object, or explicit zeroing in `OhttpClient.send()` before return?
4. AEAD failure audit trail (P6-7): structured log event only, or rate-limited security alert?

These questions are not defects in the Phase 6 deliverable. Phase 7 cross-phase triage owns the resolution.

---

## 12. Final Verdict

**PASS**

Phase 6 satisfies every acceptance criterion in `specs/.current/AW-2865/phase-6/prd.md §Success / Metrics` and every exit condition in `specs/.current/AW-2865/phase-6/plan.md §9`. The deliverable is internally consistent, fully grounded in specific source evidence, and produces a schema-conformant set of 13 draft task entries (P6-1..P6-13) ready for Phase 7 cross-phase severity triage.

**Summary by stream:**

| Stream | Outcome |
|---|---|
| Privacy risk review | 1 BLOCKER (P6-1 `sendDirect()` bypass), 3 HIGH (P6-2 outer-channel HTTP, P6-3/P6-4 key-material persistence) |
| Observability gaps catalog | 4 HIGH observability gaps (OG-1..OG-4), 1 IMPROVEMENT (OG-5); 6 logging constraints (LC-1..LC-6) complete |
| Documentation review | 1 IMPROVEMENT (P6-11 example gaps), 1 IMPROVEMENT confirmed-clean (P6-12 dependency pinning) |
| Logging sweep | CLEAN — zero logging calls in `lib/src/` confirmed |
| Read-only constraint | SATISFIED — no file in `lib/`, `test/`, or `example/` was modified |

The Phase 6 investigation is ready to feed Phase 7 (cross-phase severity triage and consolidated task list).
