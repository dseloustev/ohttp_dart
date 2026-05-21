# AW-2865 Phase 5: Test Suite Coverage Gap Review

Status: PRD_READY

## Context / Idea

This is Phase 5 of 7 in the AW-2865 investigation of `ohttp_dart` for potential use in a non-custodial crypto wallet. The overarching goal of the ticket is not to implement improvements but to identify risks and form subsequent engineering tasks with severity labels (`BLOCKER` / `HIGH` / `IMPROVEMENT`).

Phases 1–4 have audited the four implementation layers (HPKE, BHTTP, OHTTP, OhttpClient) and have produced 40+ draft Jira tasks. Those audits repeatedly surfaced test coverage as a compounding risk factor — uncovered code paths mean implementation defects cannot be confidently detected. Phase 5 closes the loop by systematically reviewing the existing test suite to determine exactly where coverage stops, what scenarios are absent, and what class of tests (negative, fuzz/property, integration) are entirely missing.

The test suite currently consists of exactly three files:

- `test/hpke_test.dart` — Layer 1 (HPKE Base Mode sender)
- `test/bhttp_test.dart` — Layer 2 (BHTTP framing + varint)
- `test/ohttp_test.dart` — Layer 3 (OHTTP encap/decap) with incidental Layer 4 coverage

No integration tests against a live OHTTP gateway exist. No fuzz or property-based tests exist.

This phase is read-only. No file under `lib/`, `test/`, `pubspec.yaml`, or `example/` is modified. All output is confined to `specs/.current/AW-2865/phase-5/`.

Inherited context: ticket-wide PRD at `specs/.current/AW-2865/prd.md` (if present); vision at `specs/.current/AW-2865/vision.md`; completed phase summaries in `specs/.current/AW-2865/phase-{1..4}/summary.md`.

---

## Goals

1. Read all three test files in full and record actual line counts and test group names.
2. Confirm which RFC 9180 Appendix A.1 vectors `hpke_test.dart` exercises via `testKeyPair` injection, scoped to Base Mode + one seal (the sub-test directly used by the OHTTP flow).
3. Confirm which QUIC varint boundary values `bhttp_test.dart` covers.
4. Identify every missing negative-case scenario across all three files and tie each gap to a specific line range where existing coverage ends.
5. Identify the complete absence of fuzz/property-based tests across all files.
6. Identify the complete absence of integration tests against a live gateway.
7. Confirm that test gaps already surfaced in Phases 1–4 (F-6, R-6, TASK-O7, Phase 4 client findings) are real by citing specific missing test scenarios.
8. Produce a numbered draft task entry for every identified gap, each carrying: test file, missing scenario name, line range where coverage stops, proposed test group name, and severity.
9. Ensure each draft task entry is independently actionable for an engineer in a future implementation ticket.

---

## User Stories

**US-1 — Security auditor reviewing gaps**
As a security auditor assessing `ohttp_dart` for production readiness, I need a complete list of untested failure paths so I can understand which defects could exist undetected in a deployed wallet.

**US-2 — Engineer implementing follow-up test tickets**
As an engineer picking up a future test-implementation task, I need each gap described precisely enough (file, missing scenario, line range, proposed group name) that I can write the test without re-reading the audit.

**US-3 — Wallet integration team**
As a wallet integration team reviewing blockers, I need the test gap severity to be correctly calibrated (BLOCKER / HIGH / IMPROVEMENT) so I can distinguish test absences that expose critical protocol correctness risks from those that are merely quality improvements.

**US-4 — Maintainer of `hpke_test.dart`**
As the maintainer of HPKE tests, I need confirmation that the `testKeyPair` injection hook in `setupBaseS` is still exercised by the current tests, so that any future `hpke.dart` refactor does not silently break deterministic vector coverage.

---

## Main Scenarios

### Scenario A — `test/hpke_test.dart` analysis

1. Read the file and record total line count and test group structure.
2. Verify which RFC 9180 Appendix A.1 sub-vectors are asserted (enc, ciphertext, exporter output values), focusing on Base Mode + one seal — the sub-test directly exercised by the OHTTP flow.
3. Check for presence or absence of tests for:
   - Sequence-number overflow (`_seq` wrap-around → `StateError`) — known gap from Phase 1 (F-6).
   - `export()` called with a length outside expected bounds.
   - Invalid / off-curve public key passed to `setupBaseS`.
   - `seal()` called more than once on the same `HpkeSenderContext` instance.
4. Record each absent test as a draft task entry.

### Scenario B — `test/bhttp_test.dart` analysis

1. Read the file and record total line count and test group structure.
2. Confirm varint round-trip coverage at 1/2/4/8-byte boundary values (63, 64, 16383, 16384, 2^30−1, 2^30, 2^62−1).
3. Check for presence or absence of tests for:
   - Truncated body (`sublist` → `RangeError`) — known gap from Phase 2 (R-6) and confirmed in Phase 4 (C-8).
   - Oversized `symLen` in a KeyConfig blob causing buffer overread.
   - Unknown framing indicator byte (anything other than `0x00` for request / `0x01` for response).
   - Malformed header (missing colon separator in header field).
   - Header field list with no terminating zero-length field.
   - Varint encoding of values ≥ 2^62 (should fail or be rejected).
4. Record each absent test as a draft task entry.

### Scenario C — `test/ohttp_test.dart` analysis

1. Read the file and record total line count and test group structure.
2. Map each existing test group to a code path it exercises in `ohttp.dart` and `ohttp_client.dart`.
3. Check for presence or absence of tests for:
   - Malformed KeyConfig: buffer too short to hold the mandatory fields.
   - Malformed KeyConfig: `symLen > 4` (extra KDF+AEAD pairs) — known gap from Phase 3 (O-4, vision §5.1).
   - KeyConfig with unknown KEM ID (not `0x0020`) — should produce `FormatException`.
   - KeyConfig with unknown KDF or AEAD ID — should produce `UnsupportedError`.
   - AEAD authentication failure from a tampered ciphertext — known gap from Phase 3 (TASK-O7) and Phase 4 (C-11).
   - Empty response body passed to `ohttpDecapsulate`.
   - Response body shorter than 16 bytes (response nonce cannot be extracted).
   - Round-trip: encapsulate then decapsulate with matching key material.
4. Confirm absence of live-gateway integration tests.
5. Confirm absence of fuzz/property-based tests across all three files.
6. Record each absent scenario as a draft task entry.

### Scenario D — Cross-phase confirmation

For each test gap already named in a prior phase summary:
- F-6 (Phase 1): sequence-number overflow test absence — confirm it is still absent.
- R-6 (Phase 2): truncated-body negative test absence — confirm it is still absent.
- TASK-O7 (Phase 3): four missing ohttp_test.dart test groups — confirm each is still absent.
- Phase 4 (C-11): auth-failure path tested only indirectly — confirm.
Record a cross-reference note for each confirmation.

---

## Success / Metrics

| Criterion | How measured |
|---|---|
| Every test file read in full | Confirmed by citing actual line count in research output |
| Every existing test group named | Test group names listed by file in research |
| Every gap tied to a line range | Draft task entry includes file + line range where coverage stops |
| Every gap from Phases 1–4 confirmed or refuted | Cross-phase confirmation table complete (Scenario D) |
| All 8 investigation vectors addressed for test suite | Table in tasks.md §"Eight investigation vectors" marked complete |
| No code changes | `git diff HEAD -- lib/ test/` is empty at close of phase |
| All draft task entries independently actionable | Each entry: file, scenario, line range, group name, severity |
| Severity assigned to every gap | Each draft task entry carries exactly one of BLOCKER / HIGH / IMPROVEMENT |

---

## Constraints and Assumptions

**Constraints:**

1. This is a read-only audit. No file under `lib/`, `test/`, `pubspec.yaml`, or `example/` may be modified.
2. All findings are draft task entries — no tests are added in this phase.
3. Severity labels must follow the vision §2 definitions: `BLOCKER` (must fix before wallet use), `HIGH` (should fix before wallet use), `IMPROVEMENT` (desirable but not blocking).
4. Each draft task must be scoped so a single engineer can pick it up without depending on another in-flight task from this phase's output.
5. If two gap findings are inseparable, they must be merged into one task with an explanation of why.
6. Fuzz test tasks are framework-agnostic — the choice of testing framework is left to the implementing engineer and must not be prescribed in the draft task entries.
7. No coverage tooling is available in CI; coverage gaps must be inferred by reading the test files in full, not from line coverage reports.

**Assumptions:**

1. Phases 1–4 summaries are complete and their findings are authoritative. This phase confirms or extends them; it does not re-audit the implementation files.
2. The `testKeyPair` injection hook in `setupBaseS` is intentional and must remain functional — this is confirmed by CLAUDE.md.
3. The three test files named in `vision.md §3` represent the entirety of the current test suite — there are no additional test files.
4. The absence of an integration test directory against a live gateway is confirmed by CLAUDE.md ("There are no integration tests against a live gateway").
5. Fuzz/property-based tests are absent — confirmed by the tasks.md context; this phase confirms and documents the gap, not discovers it fresh.
6. Public OHTTP test gateways may be available for integration testing; integration test gap tasks are therefore classified as `HIGH` (not `IMPROVEMENT`), reflecting that a real gateway endpoint can be used to validate wire-format correctness without requiring a self-hosted setup.
7. RFC 9180 Appendix A.1 vector coverage is scoped to Base Mode + one seal only — the sub-test directly exercised by the OHTTP encapsulation flow. Full sub-test coverage (export-only, multiple seals) is out of scope for this phase's gap analysis.

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| A gap named in a prior phase turns out to be covered by an existing test not noticed in that phase | Low | Moderate — severity label would need downgrading | This phase reads each test file in full before asserting gaps; any already-covered case is excluded from draft tasks |
| The phase produces too many fine-grained tasks that overlap at Iteration 7 compilation | Moderate | Low — only aggregation effort | Tasks that share a test group can be merged at Iteration 7; flag candidates in research |
| A test gap is marked HIGH but the underlying implementation defect is BLOCKER | Low | High — severity understatement | Each test gap severity is cross-checked against the severity of the implementation finding it would detect |
| `testKeyPair` hook is found to be unreachable from the current test suite | Low | High — vectors may not be reproducible | If found, flag as a separate BLOCKER finding and do not rely on vector assertions as proof of correctness |
| Misidentifying a deliberate test omission as a gap | Low | Low — extra task created | Mark ambiguous cases as IMPROVEMENT; note the ambiguity in the task description |
| No public OHTTP gateway is reachable at the time integration tests are written | Low | Moderate — integration test tasks cannot be validated end-to-end | Tasks should specify both a live-gateway path and a mock-gateway fallback so work is not blocked |

---

## Resolved Questions

1. **Fuzz test tooling.** Framework-agnostic — the choice of Dart fuzz testing framework (e.g., `package:test` fuzz APIs, an external tool) is left to the implementing engineer. Draft task entries must not prescribe a specific framework.

2. **Integration test environment.** No internal gateway is available yet; however, public OHTTP test gateways may exist. Integration test gap tasks are classified as `HIGH` (not `IMPROVEMENT`) because a public gateway may be usable for validation without a self-hosted setup.

3. **Test coverage tooling in CI.** No `dart test --coverage` in CI. Coverage gaps are inferred by reading the three test files in full; no specific uncovered line numbers from a coverage report are available.

4. **Scope of HPKE vector coverage.** Base Mode + one seal only — the sub-test directly exercised by the OHTTP flow. Full RFC 9180 Appendix A.1 coverage (export-only, multiple-seal sub-tests) is not expected in this phase's gap analysis.

5. **Property-based test threshold.** Left to the implementing engineer; draft task entries must not specify a minimum iteration count.
