# AW-2865 Phase 1: HPKE Layer Audit Against RFC 9180

Status: PRD_READY

## Context / Idea

Ticket AW-2865 is a read-only investigation of the `ohttp_dart` package to determine its production-readiness for use in a non-custodial crypto wallet. The investigation is organized into phases, each covering one layer of the four-layer stack (`hpke.dart` → `ohttp.dart` → `bhttp.dart` → `ohttp_client.dart`) or one cross-cutting concern.

Phase 1 targets the bottom layer: `lib/src/hpke.dart`. This file implements RFC 9180 HPKE Base Mode (Sender only) for the fixed cipher suite DHKEM(X25519, HKDF-SHA256) + AES-128-GCM. Its correctness is the cryptographic foundation everything else rests on.

**Inherited context (ticket-wide vision §2 severity model):**
- `BLOCKER` — must fix before wallet use
- `HIGH` — should fix before wallet use
- `IMPROVEMENT` — desirable but not blocking

**No code is changed in this phase.** Every output is a finding mapped to a future Jira task.

---

## Goals

1. Establish whether `LabeledExtract`, `LabeledExpand`, KEM Encap, `setupBaseS` key schedule, nonce derivation, and the `export` function match the corresponding RFC 9180 sections precisely.
2. Identify concrete deviations from RFC 9180 (wrong labels, wrong lengths, wrong byte ordering, missing domain separation) that would constitute cryptographic defects.
3. Identify structural risks: untyped exception on sequence-number overflow, test-seam parameter reachable from production, and absence of key-material zeroization.
4. Produce a set of draft follow-up Jira task entries, each with a file reference, line range, RFC section, and severity classification.

---

## User Stories

**US-1 — Security reviewer**
As a security reviewer preparing the package for wallet integration, I need a line-by-line audit of `hpke.dart` against RFC 9180 so I can determine whether the cryptographic implementation is correct and identify which gaps must be closed before production use.

**US-2 — Engineering lead**
As an engineering lead, I need each finding to be classified by severity (BLOCKER / HIGH / IMPROVEMENT) and mapped to a specific code location and RFC section, so I can prioritize follow-up tasks independently and assign them to engineers without further triage.

**US-3 — Wallet integrator**
As a developer integrating `ohttp_dart` into a non-custodial crypto wallet, I need to know whether the HPKE layer introduces risks to key material (e.g., residual secrets in memory, overflow conditions) that could compromise user funds or privacy before the package can be used in production.

---

## Main Scenarios

### Scenario 1 — Labeled KDF correctness verification (tasks 1.2)
Analyst reads `hpke.dart` and identifies the `LabeledExtract` and `LabeledExpand` implementations. Each is compared against RFC 9180 §4: suite ID construction (`HPKE` ASCII || `\x00\x00` || KEM ID || KDF ID || AEAD ID in big-endian), label string embedding, and HMAC-SHA-256 invocation. Any missing prefix, wrong label, or wrong concatenation order is recorded as a finding.

### Scenario 2 — KEM Encap correctness verification (task 1.3)
Analyst traces the DH step (X25519 of ephemeral private key with recipient public key), the `ExtractAndExpand` call, and the derivation of `shared_secret` against RFC 9180 §4.1. Verifies that `enc` (the ephemeral public key) is the correct 32-byte X25519 public key and that the KEM context is assembled in the right byte order.

### Scenario 3 — Key-schedule verification (task 1.4)
Analyst traces `setupBaseS` to confirm: `mode_base = 0`, correct `ks_context` construction, `secret = LabeledExtract(sharedSecret, "secret", psk)`, and the three `LabeledExpand` calls for `key`, `base_nonce`, and `exp` against RFC 9180 §5.1.

### Scenario 4 — Nonce derivation verification (task 1.5)
Analyst checks that `HpkeSenderContext.seal` computes `nonce = baseNonce XOR I2OSP(_seq, 12)` using big-endian byte order, that the copy-per-call pattern is present (no aliasing), and that `_seq` is incremented after each seal.

### Scenario 5 — Export function verification (task 1.6)
Analyst verifies `HpkeSenderContext.export` calls `LabeledExpand(exporterSecret, "sec", context, L)` with the correct suite-ID prefix as required by RFC 9180 §5.3.

### Scenario 6 — Structural risk recording (tasks 1.7–1.9)
Analyst notes:
- Whether the `_seq` overflow guard throws a typed, catchable exception or an untyped `StateError`.
- Whether `testKeyPair` is an optional named parameter in the production `setupBaseS` signature, making the test seam reachable from outside the test harness.
- Whether `key`, `baseNonce`, and `exporterSecret` are zeroed after `seal()` / `export()` complete.

### Scenario 7 — Draft task entry compilation (task 1.10)
For each finding from Scenarios 1–6, the analyst produces a draft task entry containing: finding description, affected file and line range, relevant RFC section (e.g., "RFC 9180 §4"), severity classification, and a proposed task title for Jira.

---

## Success / Metrics

| Criterion | Definition of done |
|---|---|
| All RFC 9180 sections covered | Tasks 1.2–1.6 each produce a positive ("matches RFC") or negative ("deviates: …") verdict with a cited RFC section and line reference in `hpke.dart`. No section from the labeled-KDF, KEM Encap, key-schedule, nonce-derivation, and export sub-sections is skipped. |
| Scrutiny table fully addressed | Every row in the vision §4 scrutiny table for `hpke.dart` (single-suite hard-coding, sequence-number overflow, `testKeyPair` injection) has a corresponding finding or explicit "no issue found" note. |
| Severity assigned to every finding | Every finding carries exactly one of `BLOCKER`, `HIGH`, or `IMPROVEMENT`. |
| No code changes | `git diff HEAD -- lib/ test/` is empty at the end of this phase. |
| Draft tasks independently actionable | Each draft task can be assigned to a single engineer without requiring another in-flight task from this list to be completed first. |

---

## Constraints and Assumptions

1. **Read-only.** No file in `lib/` or `test/` is modified during this phase.
2. **Fixed cipher suite.** Only the single suite DHKEM(X25519, HKDF-SHA256) + AES-128-GCM is in scope. Evaluating alternative suites is out of scope.
3. **Sender only.** RFC 9180 receiver (decapsulation) is not implemented and not audited in this phase.
4. **Response decap excluded.** `HpkeSender.hkdfExtract` / `hkdfExpand` (plain, unlabeled variants used for response decapsulation in `ohttp.dart`) are reviewed in Phase 3 (OHTTP layer), not here.
5. **`package:cryptography` primitives assumed correct.** The audit does not re-verify X25519, HMAC-SHA-256, or AES-128-GCM implementations from the external package. The audit verifies only how `hpke.dart` composes them.
6. **RFC 9180 is the normative reference.** RFC 9180 Appendix A.1 test vectors are the ground truth for correctness. The `hpke_test.dart` vector tests are an additional signal but are not authoritative.

---

## Risks

| Risk | Severity | Mitigation |
|---|---|---|
| A labeled-KDF deviation (wrong suite ID, wrong label) would silently produce incorrect ciphertext that a conformant receiver would reject — this would be a BLOCKER for interoperability | HIGH | Scenario 1 directly addresses this; any deviation is escalated to BLOCKER |
| The `testKeyPair` injection hook is a non-production code path active in production builds; a caller could supply a weak ephemeral key, breaking forward secrecy | HIGH | Scenario 6 captures this; remediation task would gate the hook behind an assertion or separate test-only API |
| Absence of zeroization means ephemeral AES key and exporter secret remain in heap memory after the OHTTP request completes; relevant to wallet threat model (memory dump of a compromised process) | HIGH | Scenario 6 captures this; remediation task is a separate IMPROVEMENT or HIGH task depending on wallet threat model |
| Untyped `StateError` on `_seq` overflow is uncatchable by type; callers cannot distinguish it from unrelated state errors | IMPROVEMENT | Scenario 6 captures this; remediation is a typed exception class |
| Audit findings may be incomplete if `hpke.dart` has grown since the vision was written | LOW | Tasks 1.1 (full outline read) and 1.10 (cross-check against scrutiny table) serve as completeness gates |

---

## Open Questions

1. Should the `testKeyPair` injection point be classified as `HIGH` or `BLOCKER` for a production wallet, given that it requires a deliberate caller action to activate? The severity affects prioritization of the remediation task.
2. Is key-material zeroization (`IMPROVEMENT` in the vision) considered `HIGH` for the specific wallet threat model (non-custodial, user keys in memory)? If the wallet stores or derives keys in the same process, the threat model may elevate this.
3. Are there any known interoperability test results against a real OHTTP gateway (e.g., the Go reference implementation) that would confirm or deny the RFC 9180 vector test coverage? If yes, those results should be referenced in the Phase 1 findings to avoid duplicating gateway-level testing in later phases.
4. Is there a requirement to support sequence numbers greater than 1 per `HpkeSenderContext` instance in the target wallet use case, or is the context always single-use? If always single-use, the overflow risk severity may be downgraded.
