# Phase 4 QA Report — OhttpClient Layer Audit

**Ticket:** AW-2865 — ohttp_dart investigation
**Phase:** 4 of 7 (`ohttp_client.dart` audit)
**QA date:** 2026-05-21
**Audited deliverables:**
- `specs/.current/AW-2865/phase-4/prd.md` (Status: PRD_READY)
- `specs/.current/AW-2865/phase-4/plan.md` (Status: PLAN_APPROVED)
- `specs/.current/AW-2865/phase-4/research.md` (Status: RESEARCH_COMPLETE)
- `specs/.current/AW-2865/phase-4/tasks.md` (12/12 tasks checked)
- `specs/.current/AW-2865/review.md` (Phase 4 section, Verdict: APPROVED)
- `lib/src/ohttp_client.dart` (177 lines, read-only subject)

---

## Phase Scope

Phase 4 is a read-only audit of `lib/src/ohttp_client.dart`. The deliverable is a completed
`tasks.md` with 12 findings (C-1..C-12), a `research.md` containing the full findings and draft
task entries, and confirmation that no source under `lib/` or `test/` was modified. This QA
report verifies the deliverable against the PRD success criteria, spot-checks findings against
the source, and issues a final pass/fail verdict.

---

## 1. Task Completion Check

All 12 tasks in `specs/.current/AW-2865/phase-4/tasks.md` are marked `[x]`.

| Task | Status | Finding |
|---|---|---|
| 4.1 Read full file | [x] | 177-line file confirmed; 4 top-level declarations enumerated |
| 4.2 https-scheme enforcement | [x] | C-1 (HIGH) |
| 4.3 Path string concatenation | [x] | C-2 (IMPROVEMENT) |
| 4.4 targetAuthority verbatim embedding | [x] | C-3 (HIGH) |
| 4.5 sendDirect() privacy warning | [x] | C-4 (HIGH) |
| 4.6 KeyConfig GET timeout and caching | [x] | C-5 (HIGH) |
| 4.7 Gateway POST timeout | [x] | C-6 (HIGH) |
| 4.8 Non-200 generic Exception | [x] | C-7 (HIGH) |
| 4.9 Network error untyped propagation | [x] | C-8 (HIGH) |
| 4.10 Response size caps | [x] | C-9 (HIGH) |
| 4.11 OhttpHeader.name case inconsistency | [x] | C-10 (IMPROVEMENT) |
| 4.12 Draft task entries | [x] | TASK-C1..TASK-C12 in research.md |

**Result: PASS — all 12 tasks checked.**

---

## 2. Source Spot-Checks

Three findings were verified by direct reading of `lib/src/ohttp_client.dart`.

### Spot-check 1 — C-5 "no timeout" at line 81

**Claim (tasks.md 4.6 / research.md C-5):** `_httpClient.get(Uri.parse(...))` at line 81 has no
`.timeout(Duration(...))` chained on the future, no retry loop, and no reference to any cached
`OhttpKeyConfig`.

**Source verification (lib/src/ohttp_client.dart:80-89):**

```
80  // 1. Get the gateway's KeyConfig
81  final configResponse = await _httpClient.get(
82    Uri.parse('${gateway.gatewayBaseUrl}${gateway.configPath}'),
83  );
84  if (configResponse.statusCode != 200) {
85    throw Exception(
86      'Failed to fetch KeyConfig: HTTP ${configResponse.statusCode}',
87    );
88  }
89  final config = OhttpKeyConfig.parse(configResponse.bodyBytes);
```

No `.timeout(...)`, no retry loop, no cached config reference. Instance fields at lines 60-61
are `_httpClient` and `gateway` only — no cache field. Every `send()` invocation unconditionally
re-fetches KeyConfig.

**Verdict: CONFIRMED.**

---

### Spot-check 2 — C-1 "no scheme enforcement" at lines 43-50

**Claim (tasks.md 4.2 / research.md C-1):** `OhttpGatewayConfig` constructor at lines 43-50
accepts `gatewayBaseUrl` as a bare `String` with no validation. `Uri.parse(...)` at lines 82 and
116 accepts `http://` silently.

**Source verification (lib/src/ohttp_client.dart:35-53):**

```
35  class OhttpGatewayConfig {
36    final String gatewayBaseUrl;
37    ...
43    const OhttpGatewayConfig({
44      required this.gatewayBaseUrl,
45      required this.configPath,
46      required this.requestPath,
47      required this.targetAuthority,
48      this.targetScheme = 'https',
49      this.directBaseUrl,
50    });
```

No `assert`, no `Uri.parse(gatewayBaseUrl).scheme == 'https'` guard, no validation method. The
`const` constructor cannot call async or external validation. Lines 82 and 116 call
`Uri.parse('${gateway.gatewayBaseUrl}${gateway.configPath}')` and
`Uri.parse('${gateway.gatewayBaseUrl}${gateway.requestPath}')` respectively — both accept any
scheme.

**Verdict: CONFIRMED.**

---

### Spot-check 3 — C-9 "no size cap" at lines 129-136

**Claim (tasks.md 4.10 / research.md C-9):** `gatewayResponse.bodyBytes` is passed to
`ohttpDecapsulate` at lines 129-133 with no length check. The result `binaryResponse` is passed
to `bhttp.parseResponse` at line 136 with no length check.

**Source verification (lib/src/ohttp_client.dart:127-136):**

```
127  // 5. Decapsulate response (pure Dart)
128  onLog?.call('Decapsulating response...');
129  final binaryResponse = await ohttpDecapsulate(
130    encResult.enc,
131    encResult.exportedSecret,
132    gatewayResponse.bodyBytes,
133  );
134
135  // 6. Parse BHTTP response
136  final bhttpResp = bhttp.parseResponse(binaryResponse);
```

No `if (gatewayResponse.bodyBytes.length > maxLen)` guard before line 129. No length check on
`binaryResponse` before line 136. Both calls receive unbounded input from a potentially
adversarial gateway response.

**Verdict: CONFIRMED.** Cross-layer amplification with Phase 2 R-2 (`bhttp.parseResponse` also
has no header/body cap) is correctly characterized in research.md.

---

## 3. Read-Only Constraint Verification

The review.md Phase 4 section (Criterion 8) confirms:

> `git diff HEAD -- lib/ test/` produces no output. `git status --short lib/ test/` produces no
> output. The only working-tree modifications are inside `specs/.current/AW-2865/` (tasklist.md
> modification and the `phase-4/` directory). No source under `lib/`, `test/`, `pubspec.yaml`,
> or `example/` is changed.

The current git status shows `M specs/.current/AW-2865/tasklist.md` and
`?? specs/.current/AW-2865/phase-4/` — both confined to `specs/`. No `lib/` or `test/` entries.

**Result: PASS — read-only constraint honored.**

---

## 4. PRD Acceptance Criteria Verification

| PRD criterion | Status | Evidence |
|---|---|---|
| All ten investigation goals covered (tasks 4.2–4.11 each produce a verdict with a line ref) | PASS | All 10 tasks (4.2–4.11) have inline verdicts with at least one `ohttp_client.dart:NNN` citation. No PRD goal from the vision §4 scrutiny table is skipped. |
| Timeout and caching findings cite call-site lines | PASS | C-5 cites line 81 (`_httpClient.get`); C-6 cites line 115 (`_httpClient.post`). Both verified in spot-checks above. |
| `sendDirect()` finding documents privacy impact | PASS | C-4 (research.md) explicitly references RFC 9458 §1 and states "A wallet developer using `sendDirect()` as a connectivity fallback silently degrades the wallet's privacy guarantee with no runtime signal." Concrete remediation options (doc comment, `assert`, `@Deprecated`) listed. |
| Two-layer response-size finding cross-references Phase 2 | PASS | C-9 (research.md) cross-references "Phase 2 finding R-2" in the text and the §"Cross-Layer References" table maps the compounding explicitly. tasks.md 4.10 says "Two-layer amplification with Phase 2 finding R-2." |
| Severity assigned to every finding | PASS | C-1 through C-12 each carry exactly one of HIGH or IMPROVEMENT. No "TBD". Distribution: 9 HIGH (C-1, C-3..C-9, C-11), 3 IMPROVEMENT (C-2, C-10, C-12). Severities are internally consistent between tasks.md and research.md. |
| No code changes | PASS | See Section 3 above. |
| Draft tasks independently actionable | PASS | TASK-C1..TASK-C12 each carry a stable ID, single-line imperative title, severity, primary file:line remediation site, and concern category. TASK-C11 is documented as resolved by Phase 3 TASK-O5 (no independent work). TASK-C7 and TASK-C8 are flagged as merge candidates at Iteration 7. Both are deliberate explicit decisions, not silent omissions. |

**Result: PASS — all PRD acceptance criteria met.**

---

## 5. Plan Definition of Done Verification

| DoD item | Status |
|---|---|
| 1. tasks.md items 4.1–4.12 checked | PASS — all 12 marked `[x]` |
| 2. research.md contains Resolved Questions Q1/Q2/Q3 with source citations | PASS — Q1 traces `SecretBoxAuthenticationError` call chain; Q2 quotes `package:http` 1.6.0 source paths and line numbers for `ClientException`, `_ClientSocketException`, `IOClient.send()` catch clauses |
| 2. research.md contains Related Modules tables with call-site line numbers | PASS — "Direct dependencies called by `ohttp_client.dart`" table covers all 8 call sites |
| 2. research.md contains Current Endpoints & Contracts (OhttpGatewayConfig, send(), sendDirect(), dispose()) | PASS — all four documented |
| 2. research.md contains 12 findings C-1..C-12 (each with file:lines, category, severity, description, draft task title) | PASS — all 12 findings have every required field |
| 2. research.md contains Cross-Layer References tables | PASS — Phase 2 and Phase 3 cross-reference tables present |
| 2. research.md contains Draft Task Entries table TASK-C1..TASK-C12 | PASS — table complete with documented exceptions for TASK-C11 and TASK-C7+C8 |
| 2. research.md contains New Technical Questions 1–5 | PASS — five deferred design questions recorded |
| 3. `git diff HEAD -- lib/ test/ pubspec.yaml example/` is empty | PASS |
| 4. Phase 4 QA report (this file) confirms each PRD success criterion is met | PASS (this report) |
| 5. No finding row has unset severity, file:line, or concern category | PASS |
| 6. Cross-layer findings explicitly annotate owning phase (C-9 → Phase 2 R-2; C-11 → Phase 3 O-9/TASK-O5) | PASS |
| 7. Every vision §4 scrutiny row for `ohttp_client.dart` has a finding or explicit deferral | PASS — plan.md §"Scrutiny-table coverage contract" maps all vision rows to C-1 through C-12 |

**Result: PASS — all 13 DoD items satisfied.**

---

## 6. Findings Summary

### Positive Scenarios (no issues found)

No positive (gap-free) verdicts are expected in Phase 4 — the phase explicitly targets known
concern areas from the vision §4 scrutiny table. All 12 findings are confirmed deficiencies or
cross-layer confirmations. Task 4.1 (file outline) produced no finding because it was a
completeness gate, not a deficiency check.

### Confirmed Findings

| ID | Severity | Category | File : lines | Description |
|---|---|---|---|---|
| C-1 | HIGH | Scheme enforcement | `ohttp_client.dart:35-53, 82, 116` | No `https`-scheme enforcement on `OhttpGatewayConfig.gatewayBaseUrl`; plain HTTP accepted silently |
| C-2 | IMPROVEMENT | URL construction | `ohttp_client.dart:82, 116, 154` | String concatenation instead of `Uri.resolve`; double-slash and `..` traversal not normalized |
| C-3 | HIGH | Privacy / SSRF | `ohttp_client.dart:98-101` | `targetAuthority` embedded verbatim with no allow-list or scheme-stripping |
| C-4 | HIGH | Privacy / Documentation | `ohttp_client.dart:41, 52, 148-171` | `sendDirect()` bypasses OHTTP entirely; no doc comment, assert, or log warning; RFC 9458 §1 privacy guarantee silently broken |
| C-5 | HIGH | Network reliability / KeyConfig lifecycle | `ohttp_client.dart:81-89` | KeyConfig GET has no timeout, no retry, no TTL cache; two round trips per `send()` invocation |
| C-6 | HIGH | Network reliability | `ohttp_client.dart:115-122` | Gateway POST has no timeout, no retry, no cancellation hook |
| C-7 | HIGH | Error handling | `ohttp_client.dart:84-87, 120-122` | Non-200 responses throw generic untyped `Exception`; no 4xx/5xx distinction |
| C-8 | HIGH | Error handling / API contract | `ohttp_client.dart:69` | `send()` has no documented throws contract; exposes `ClientException`, `HandshakeException`, `SecretBoxAuthenticationError`, `RangeError` without any library-owned wrapper |
| C-9 | HIGH | Parser robustness / DoS | `ohttp_client.dart:129-133, 136` | No size cap on gateway response body before `ohttpDecapsulate` or `bhttp.parseResponse`; two-layer unbounded allocation (cross-ref Phase 2 R-2) |
| C-10 | IMPROVEMENT | API consistency | `ohttp_client.dart:139, 168` | `OhttpHeader.name` not lowercased in `send()` response path; lowercased in `sendDirect()` path; violates RFC 7230 §3.2 case-insensitivity |
| C-11 | HIGH | Error handling / API boundary | `ohttp_client.dart:129-133`, `ohttp.dart:219` | `SecretBoxAuthenticationError` propagates unwrapped to wallet caller (cross-layer confirmation of Phase 3 O-9; resolved by TASK-O5) |
| C-12 | IMPROVEMENT | Privacy / Observability | `ohttp_client.dart:77-78` | `onLog` unconditionally interpolates `gatewayBaseUrl` and `configPath`; violates `vision.md §7` constraint 6 |

### Negative and Edge Cases

All Phase 4 findings were derived from static code reading (no live gateway). The following
edge-case characterizations were established:

- **`effectiveDirectBaseUrl` null footgun (C-4):** When `directBaseUrl` is null, `sendDirect()`
  sends to `gatewayBaseUrl` (the OHTTP relay), not the intended target server. Deferred to
  TASK-C4 design.
- **KeyConfig rotation race (C-5):** Any TTL cache added for TASK-C5 must define a forced-refresh
  trigger on 4xx gateway response or `FormatException` from decapsulation, or key rotation will
  produce persistent failures. Recorded as New Technical Question 3.
- **`HandshakeException` wrapping scope (C-8):** `package:http` 1.6.0 does NOT wrap
  `dart:io.HandshakeException`. TLS failures propagate as `dart:io` types, requiring an explicit
  `dart:io` import in the wallet catch hierarchy. Deferred to TASK-C8 design.
- **`onLog` string evaluation (C-12):** Dart evaluates `?.call(string)` arguments even when the
  receiver is null. `onLog?.call(expensiveString)` still constructs `expensiveString`. Recorded as
  New Technical Question 5 (negligible at current call sites; guard needed for future additions).

### Automated Test Coverage

No tests exist for `ohttp_client.dart` at the audit commit. The `test/` directory contains no
`ohttp_client_test.dart` file. All Phase 4 findings are derived from static analysis only. The
draft task entries imply the following missing negative-test categories:

- Scheme misconfiguration (`http://` gateway URL with a mock client).
- Simulated gateway timeout (mock client that never resolves).
- Non-200 status codes (4xx and 5xx separately) for both KeyConfig GET and gateway POST.
- Oversized gateway response (body exceeding a `maxResponseBytes` threshold once added).
- `sendDirect()` invocation with privacy-impact assertions.

### Manual Checks Needed

1. **Live gateway smoke test:** Verify that the string-concatenation URL construction (C-2) does
   not produce double-slash failures against the actual OHTTP gateway deployment. Cannot be
   assessed statically.
2. **`sendDirect()` fallback usage audit:** Grep existing wallet code for any `sendDirect()`
   call sites to assess how widely the privacy-bypass path is used in practice before
   deprecating or adding a warning.

---

## 7. Risk Zone

| Risk | Severity | Status |
|---|---|---|
| No timeout on HTTP calls; stalled gateway hangs wallet indefinitely | HIGH | Confirmed (C-5, C-6); remediation deferred to TASK-C5, TASK-C6 |
| No `https`-scheme enforcement; misconfigured wallet transmits outer channel in plaintext | HIGH | Confirmed (C-1); remediation deferred to TASK-C1 |
| `sendDirect()` has no privacy warning; RFC 9458 §1 guarantee silently broken | HIGH | Confirmed (C-4); remediation deferred to TASK-C4 |
| No response size cap; malicious gateway can force unbounded memory allocation | HIGH | Confirmed (C-9); compounded by Phase 2 R-2; deferred to TASK-C9 |
| Untyped exception surface; wallet cannot implement differentiated retry logic | HIGH | Confirmed (C-7, C-8); deferred to TASK-C7, TASK-C8 |
| `targetAuthority` verbatim embedding; SSRF and privacy misdirection possible | HIGH | Confirmed (C-3); deferred to TASK-C3 |
| No KeyConfig caching; two round trips per request; intermittent config fetch failures | HIGH | Confirmed (C-5); deferred to TASK-C5 |
| `SecretBoxAuthenticationError` reaches wallet caller unwrapped | HIGH | Confirmed (C-11; cross-layer); resolved by Phase 3 TASK-O5 |
| String path concatenation; double-slash / path traversal not normalized | IMPROVEMENT | Confirmed (C-2); deferred to TASK-C2 |
| `OhttpHeader.name` case inconsistency; RFC 7230 §3.2 violated for `send()` responses | IMPROVEMENT | Confirmed (C-10); deferred to TASK-C10 |
| `gatewayBaseUrl` interpolated unconditionally in `onLog`; metadata leak if callback supplied | IMPROVEMENT | Confirmed (C-12); deferred to TASK-C12 |

**Highest risk:** The combination of no timeout (C-5, C-6), no HTTPS enforcement (C-1), and
`sendDirect()` with no privacy warning (C-4) represents the most wallet-visible reliability and
privacy risk cluster. None of these require deep refactoring — each is a targeted guard or
annotation addition.

**Cross-layer amplification:** C-9 (no size cap at client layer) combined with Phase 2 R-2 (no
cap in `bhttp.parseResponse`) is a two-layer unbounded-allocation vector. A single malicious
gateway response can exhaust memory in the wallet process with no guard at either layer.

---

## 8. Known Housekeeping Items (non-blocking)

The review.md Phase 4 section (Verdict: APPROVED) identified three Important items that do not
block phase close but should be resolved before Iteration 7:

1. **I-1 (review.md):** PRD §Risks table omits C-11 and C-12. Add rows or a note in PRD
   §Resolved Questions acknowledging these were surfaced in research rather than pre-classified
   in the PRD.
2. **I-2 (review.md):** research.md C-10 cites `sendDirect()` response header construction as
   "line 169" — actual line is 168. tasks.md correctly cites 168. Fix the drift in research.md
   before Iteration 7 Jira import.
3. **I-3 (review.md):** research.md §"Patterns Used" item 3 says "Pattern is sound for
   observability" — contradicts C-12 which classifies `onLog` URL logging as a vision §7
   violation. Soften the wording.

None of these housekeeping items affect the correctness of the 12 findings or the 12 draft task
entries.

---

## 9. Final Verdict

**PASS.**

All 12 tasks in `tasks.md` are `[x]`. Three spot-checked findings (C-5 at line 81, C-1 at lines
43-50, C-9 at lines 129-136) are confirmed by direct source reading and match the claims in
`tasks.md` and `research.md` exactly. No file under `lib/` or `test/` was modified. All seven
PRD success criteria are met. All 13 plan Definition of Done items are satisfied.

Phase 4 is ready to close. The 12 draft task entries (TASK-C1..TASK-C12) are independently
actionable and ready for Iteration 7 consolidation. Three housekeeping items (review.md I-1,
I-2, I-3) are recommended before Jira import but do not affect the audit's findings.
